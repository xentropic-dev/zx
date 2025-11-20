# Memory Management in WASM Client Components

## Overview

Memory management is critical for WASM applications. Unlike JavaScript with automatic GC, Zig requires explicit memory management. This document explores patterns, pitfalls, and best practices.

## The Challenge

### WASM Memory Model
- Linear memory (flat byte array)
- Shared between WASM and JS
- No automatic garbage collection
- Fixed or growable
- Must manage allocations carefully

### Common Pitfalls
1. **Memory leaks** - Allocating without freeing
2. **Use-after-free** - Accessing freed memory
3. **Double free** - Freeing same memory twice
4. **Buffer overflow** - Writing past buffer end
5. **Fragmentation** - Many small allocs/frees

## Allocator Strategies

### 1. Global WASM Allocator (Current zx-wasm-renderer)

```zig
const allocator = std.heap.wasm_allocator;

export fn render() void {
    const html = std.fmt.allocPrint(allocator,
        "<div>Count: {d}</div>",
        .{count}
    ) catch return;

    // Copy to buffer
    @memcpy(buffer[0..html.len], html);

    // ⚠️ MUST free!
    allocator.free(html);
}
```

**Pros**:
- Simple, built-in
- Grows as needed
- General purpose

**Cons**:
- Can fragment
- Manual free required
- No leak detection

### 2. Arena Allocator (Recommended for Rendering)

```zig
var arena = std.heap.ArenaAllocator.init(std.heap.wasm_allocator);
const arena_allocator = arena.allocator();

export fn render() void {
    defer arena.deinit(); // Free everything at once!

    const html = std.fmt.allocPrint(arena_allocator,
        "<div>Count: {d}</div>",
        .{count}
    ) catch return;

    // Copy to buffer
    @memcpy(buffer[0..html.len], html);

    // No manual free needed - arena.deinit() handles it
}
```

**Pros**:
- ✅ No manual frees
- ✅ Fast allocation (bump allocator)
- ✅ Perfect for per-render allocations
- ✅ Automatic cleanup

**Cons**:
- Can't free individual allocations
- Holds memory until deinit
- Not good for long-lived data

**Best for**: Temporary render allocations

### 3. Fixed Buffer Allocator

```zig
var buffer: [4096]u8 = undefined;
var fba = std.heap.FixedBufferAllocator.init(&buffer);
const allocator = fba.allocator();

export fn render() void {
    fba.reset(); // Clear all allocations

    const html = std.fmt.allocPrint(allocator,
        "<div>Count: {d}</div>",
        .{count}
    ) catch return;

    // html points into buffer, no copy needed!
    // Update DOM directly from buffer
}
```

**Pros**:
- ✅ Zero system allocations
- ✅ Predictable performance
- ✅ Fast reset
- ✅ No fragmentation

**Cons**:
- ❌ Fixed size
- ❌ Returns error if full
- ❌ Not suitable for dynamic sizes

**Best for**: Known maximum sizes, hot paths

### 4. Pool Allocator

```zig
const Node = struct {
    tag: []const u8,
    children: []Node,
    // ...
};

var pool = std.heap.MemoryPool(Node).init(std.heap.wasm_allocator);

export fn createNode() *Node {
    return pool.create() catch unreachable;
}

export fn destroyNode(node: *Node) void {
    pool.destroy(node);
}
```

**Pros**:
- ✅ Fast alloc/free
- ✅ No fragmentation
- ✅ Good for same-sized objects
- ✅ Reuses memory

**Cons**:
- ❌ Only for one type
- ❌ Must track and free manually
- ❌ More complex

**Best for**: Virtual DOM nodes, component instances

### 5. General Purpose Allocator (Debug Only)

```zig
var gpa = std.heap.GeneralPurposeAllocator(.{
    .enable_memory_limit = true,
}){};
const allocator = gpa.allocator();

export fn render() void {
    // ... allocations ...
}

export fn checkLeaks() bool {
    return gpa.deinit() == .leak; // Detects leaks!
}
```

**Pros**:
- ✅ Leak detection
- ✅ Safety checks
- ✅ Use-after-free detection
- ✅ Double-free detection

**Cons**:
- ❌ Slow
- ❌ Large binary
- ❌ Not for production

**Best for**: Development, testing, debugging

## Recommended Pattern: Hybrid Approach

### For Rendering (Temporary Allocations)

```zig
// Global arena for render allocations
var render_arena = std.heap.ArenaAllocator.init(std.heap.wasm_allocator);

export fn render() usize {
    // Reset arena at start of each render
    _ = render_arena.reset(.retain_capacity);

    const allocator = render_arena.allocator();

    // All render allocations use arena
    const html = component.render(allocator) catch return 0;

    // Copy to shared buffer
    const len = @min(html.len, render_buffer.len);
    @memcpy(render_buffer[0..len], html[0..len]);

    // No manual frees needed!
    // Arena automatically cleaned up on next render
    return len;
}
```

### For State (Long-Lived Allocations)

```zig
// Global allocator for state
const state_allocator = std.heap.wasm_allocator;

// Long-lived state
var username: []u8 = undefined;

export fn setUsername(ptr: [*]const u8, len: usize) void {
    // Free old username
    if (username.len > 0) {
        state_allocator.free(username);
    }

    // Allocate new username
    username = state_allocator.alloc(u8, len) catch return;
    @memcpy(username, ptr[0..len]);
}

export fn cleanup() void {
    state_allocator.free(username);
}
```

### Complete Example

```zig
const std = @import("std");

// Allocators
var render_arena = std.heap.ArenaAllocator.init(std.heap.wasm_allocator);
const state_allocator = std.heap.wasm_allocator;

// Shared buffer
export var render_buffer: [8192]u8 = undefined;

// State (long-lived)
var counter: u32 = 0;
var items: std.ArrayList([]const u8) = std.ArrayList([]const u8).init(state_allocator);

// Component
const Component = struct {
    pub fn render(allocator: std.mem.Allocator, count: u32) ![]const u8 {
        var html = std.ArrayList(u8).init(allocator);
        const writer = html.writer();

        try writer.print("<div>Counter: {d}</div>", .{count});

        return html.toOwnedSlice();
    }
};

export fn increment() void {
    counter += 1;
}

export fn addItem(ptr: [*]const u8, len: usize) void {
    // Allocate permanent copy
    const item = state_allocator.alloc(u8, len) catch return;
    @memcpy(item, ptr[0..len]);
    items.append(item) catch return;
}

export fn render() usize {
    // Reset render arena
    _ = render_arena.reset(.retain_capacity);

    // Use arena for temporary render allocations
    const allocator = render_arena.allocator();

    // Render component
    const html = Component.render(allocator, counter) catch return 0;

    // Copy to shared buffer
    const len = @min(html.len, render_buffer.len);
    @memcpy(render_buffer[0..len], html[0..len]);

    return len;
}

export fn cleanup() void {
    // Cleanup state
    for (items.items) |item| {
        state_allocator.free(item);
    }
    items.deinit();
    render_arena.deinit();
}
```

## Memory Sharing Patterns

### Pattern 1: Fixed Export Buffer (Simplest)

```zig
export var buffer: [4096]u8 = undefined;

export fn render() usize {
    // Write directly to export buffer
    const len = std.fmt.bufPrint(&buffer,
        "<div>Hello</div>",
        .{}
    ) catch return 0;

    return len;
}
```

```js
const len = exports.render();
const memory = new Uint8Array(exports.memory.buffer);
const html = decoder.decode(memory.slice(
    exports.buffer.value,
    exports.buffer.value + len
));
```

**Pros**: Zero allocations
**Cons**: Fixed size limit

### Pattern 2: Growable Buffer

```zig
export var buffer_ptr: [*]u8 = undefined;
export var buffer_len: usize = 0;
var buffer_capacity: usize = 0;

fn ensureCapacity(needed: usize) !void {
    if (needed <= buffer_capacity) return;

    const new_capacity = needed * 2;
    const new_buffer = try allocator.realloc(
        if (buffer_capacity > 0) buffer_ptr[0..buffer_capacity] else &[_]u8{},
        new_capacity
    );

    buffer_ptr = new_buffer.ptr;
    buffer_capacity = new_capacity;
}

export fn render() !void {
    const html = try Component.render(allocator);
    defer allocator.free(html);

    try ensureCapacity(html.len);
    @memcpy(buffer_ptr[0..html.len], html);
    buffer_len = html.len;
}
```

```js
exports.render();
const ptr = exports.buffer_ptr.value;
const len = exports.buffer_len.value;
const memory = new Uint8Array(exports.memory.buffer);
const html = decoder.decode(memory.slice(ptr, ptr + len));
```

**Pros**: Handles any size
**Cons**: More complex, need realloc

### Pattern 3: Multiple Buffers

```zig
export var html_buffer: [8192]u8 = undefined;
export var style_buffer: [2048]u8 = undefined;
export var script_buffer: [2048]u8 = undefined;

export fn render() struct { html_len: usize, style_len: usize, script_len: usize } {
    // Write different parts to different buffers
    // ...
}
```

**Pros**: Organized, separate concerns
**Cons**: Multiple fixed sizes

### Pattern 4: Ring Buffer (Advanced)

```zig
const RingBuffer = struct {
    data: [8192]u8 = undefined,
    write_pos: usize = 0,
    read_pos: usize = 0,

    pub fn write(self: *RingBuffer, bytes: []const u8) !void {
        // Circular write logic
    }

    pub fn read(self: *RingBuffer, out: []u8) usize {
        // Circular read logic
    }
};

export var ring_buffer = RingBuffer{};
```

**Pros**: Continuous writing, no blocking
**Cons**: Complex, can overflow

## Common Patterns

### Pattern: String Building

```zig
fn buildHtml(allocator: std.mem.Allocator) ![]const u8 {
    var html = std.ArrayList(u8).init(allocator);
    errdefer html.deinit(); // Cleanup on error

    const writer = html.writer();
    try writer.writeAll("<div>");
    try writer.print("Count: {d}", .{counter});
    try writer.writeAll("</div>");

    return html.toOwnedSlice(); // Caller owns memory
}
```

### Pattern: Component Tree

```zig
const VNode = union(enum) {
    element: struct {
        tag: []const u8,
        children: []VNode,
    },
    text: []const u8,
};

fn renderTree(allocator: std.mem.Allocator, node: VNode) ![]const u8 {
    var result = std.ArrayList(u8).init(allocator);
    errdefer result.deinit();

    switch (node) {
        .element => |el| {
            try result.writer().print("<{s}>", .{el.tag});
            for (el.children) |child| {
                const child_html = try renderTree(allocator, child);
                defer allocator.free(child_html);
                try result.appendSlice(child_html);
            }
            try result.writer().print("</{s}>", .{el.tag});
        },
        .text => |t| try result.appendSlice(t),
    }

    return result.toOwnedSlice();
}
```

### Pattern: Object Pool

```zig
const ComponentPool = struct {
    const MAX = 100;

    components: [MAX]Component = undefined,
    free_list: [MAX]bool = [_]bool{true} ** MAX,

    pub fn alloc(self: *ComponentPool) ?*Component {
        for (self.free_list, 0..) |is_free, i| {
            if (is_free) {
                self.free_list[i] = false;
                return &self.components[i];
            }
        }
        return null;
    }

    pub fn free(self: *ComponentPool, component: *Component) void {
        const index = (@intFromPtr(component) - @intFromPtr(&self.components[0])) / @sizeOf(Component);
        self.free_list[index] = true;
    }
};
```

## Debugging Memory Issues

### Enable Leak Detection (Development)

```zig
test "memory leak detection" {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer {
        const leaked = gpa.deinit();
        if (leaked == .leak) {
            std.debug.print("Memory leak detected!\n", .{});
        }
    }

    const allocator = gpa.allocator();

    // Your code here
    const memory = try allocator.alloc(u8, 100);
    // Oops, forgot to free!
    // allocator.free(memory);
}
```

### Track Allocations

```zig
var total_allocated: usize = 0;
var total_freed: usize = 0;

const TrackedAllocator = struct {
    child_allocator: std.mem.Allocator,

    pub fn allocator(self: *TrackedAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = alloc,
                .resize = resize,
                .free = free,
            },
        };
    }

    fn alloc(ctx: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *TrackedAllocator = @ptrCast(@alignCast(ctx));
        const result = self.child_allocator.rawAlloc(len, ptr_align, ret_addr);
        if (result) |ptr| {
            total_allocated += len;
            std.debug.print("Alloc: {} bytes (total: {})\n", .{len, total_allocated});
        }
        return result;
    }

    fn free(ctx: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        const self: *TrackedAllocator = @ptrCast(@alignCast(ctx));
        total_freed += buf.len;
        std.debug.print("Free: {} bytes (total freed: {})\n", .{buf.len, total_freed});
        self.child_allocator.rawFree(buf, buf_align, ret_addr);
    }

    // ... resize implementation ...
};
```

### Memory Usage Export

```zig
export fn getMemoryStats() struct { allocated: usize, freed: usize, current: usize } {
    return .{
        .allocated = total_allocated,
        .freed = total_freed,
        .current = total_allocated - total_freed,
    };
}
```

```js
const stats = exports.getMemoryStats();
console.log(`Memory: ${stats.current} bytes in use`);
```

## Best Practices

### ✅ Do

1. **Use arena allocators for temporary render allocations**
   ```zig
   var arena = std.heap.ArenaAllocator.init(base_allocator);
   defer arena.deinit();
   ```

2. **Reset arenas between renders**
   ```zig
   _ = arena.reset(.retain_capacity);
   ```

3. **Use `defer` for cleanup**
   ```zig
   const data = try allocator.alloc(u8, 100);
   defer allocator.free(data);
   ```

4. **Use `errdefer` for error cleanup**
   ```zig
   var list = std.ArrayList(u8).init(allocator);
   errdefer list.deinit();
   ```

5. **Document ownership**
   ```zig
   /// Caller owns returned memory
   pub fn render(allocator: std.mem.Allocator) ![]const u8 {
       // ...
   }
   ```

6. **Profile and measure**
   - Track allocations in development
   - Monitor WASM memory growth
   - Test with large datasets

### ❌ Don't

1. **Don't leak allocations**
   ```zig
   // BAD
   const data = try allocator.alloc(u8, 100);
   // Forgot to free!
   ```

2. **Don't use global state allocator for temporary data**
   ```zig
   // BAD - will fragment
   const html = try std.heap.wasm_allocator.alloc(u8, 1000);
   defer std.heap.wasm_allocator.free(html);
   ```

3. **Don't allocate in hot loops**
   ```zig
   // BAD
   for (items) |item| {
       const temp = try allocator.alloc(u8, 100);
       defer allocator.free(temp);
       // ...
   }
   ```

4. **Don't ignore allocation failures**
   ```zig
   // BAD
   const data = allocator.alloc(u8, 100) catch unreachable;
   ```

5. **Don't mix allocators**
   ```zig
   // BAD
   const data = try allocator1.alloc(u8, 100);
   allocator2.free(data); // Wrong allocator!
   ```

## Summary

### Recommended Strategy

1. **Arena allocator** for all render allocations
   - Reset at start of render
   - Fast, no manual frees
   - Automatic cleanup

2. **WASM allocator** for long-lived state
   - Careful manual management
   - Document ownership
   - Use `defer`

3. **Fixed buffers** for known sizes
   - Export buffers
   - Hot paths
   - Predictable performance

4. **General purpose allocator** (debug builds only)
   - Leak detection
   - Safety checks
   - Not for production

### Memory Budget

- Initial WASM memory: 1MB
- Max per-render allocation: 10KB
- Total state: < 1MB
- Memory growth: < 100KB/minute

Monitor these metrics and optimize hot paths!
