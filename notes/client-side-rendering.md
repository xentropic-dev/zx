# Client-Side Rendering (CSR) Implementation Notes

## Overview

Implementing CSR in zx to enable interactive, client-side components using WebAssembly. This allows Zig code to run in the browser and update the DOM dynamically.

## Current Architecture

### Transpiler Status
- Located in `src/cli/transpile.zig`
- Already has basic 'use client' detection (lines 76-82, 630-636)
- Currently **skips** transpilation of 'use client' files
- Transpiled files go to `.zx/` directory by default

### Existing CSR Handling
The transpiler currently:
1. Checks first line for `'use client'` directive
2. Skips transpilation if found
3. Logs: "Skipping client-side file: {path}"

**Key insight**: Files are being skipped, not transpiled differently!

## 'use client' Directive

### Current Approach
- Using `'use client'` at top of file (Next.js convention)
- Simple, familiar to web developers
- Easy to detect during transpilation

### Design Concern
**Goes against Zig's "no hidden control flow" paradigm**

In Zig, behavior should be explicit. `'use client'` is a string literal that magically changes how the entire file is compiled - this is hidden control flow!

### Alternative Approaches (Future Consideration)
1. **Explicit build.zig configuration**
   ```zig
   const client_module = b.addModule(.{
       .target = wasm_target,
       // ...
   });
   ```

2. **File naming convention**
   - `*.client.zx` → client-side
   - `*.server.zx` → server-side
   - More explicit, follows Zig philosophy

3. **Separate directories**
   - `pages/client/` vs `pages/server/`
   - Clear separation

**Decision**: Stick with 'use client' for now (simplicity, familiarity), revisit later.

## Build Pipeline Design

### Reference: zx-wasm-renderer
Located at `../zx-wasm-renderer/`, provides working example of WASM rendering.

#### WASM Build Configuration
```zig
// build.zig
const target = b.resolveTargetQuery(.{
    .cpu_arch = .wasm32,
    .os_tag = .freestanding,
});

const wasm = b.addExecutable(.{
    .name = "zx_wasm_renderer",
    .root_module = b.createModule(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = .ReleaseSmall,  // Critical for file size!
        .single_threaded = true,
    }),
});

wasm.entry = .disabled;
wasm.export_memory = true;
wasm.rdynamic = true;
```

Key flags:
- `entry = .disabled` - No main() needed
- `export_memory = true` - JS can read WASM memory
- `rdynamic = true` - Export functions dynamically
- `optimize = .ReleaseSmall` - Minimize file size

#### Memory Sharing Pattern
```zig
// Zig side (WASM)
export var buffer: [1024]u8 = undefined;

export fn render() void {
    const msg = std.fmt.allocPrint(allocator, "<div>...</div>", .{}) catch return;
    @memcpy(buffer[0..msg.len], msg);
    allocator.free(msg);
}
```

```javascript
// JS side
const { memory, buffer, render } = wasmInstance.exports;

function updateDom() {
    const memoryArray = new Uint8Array(memory.buffer);
    let length = 0;
    while (memoryArray[buffer.value + length] !== 0 && length < 1024) {
        length++;
    }
    const bytes = memoryArray.slice(buffer.value, buffer.value + length);
    const html = new TextDecoder().decode(bytes);
    document.getElementById("root").innerHTML = html;
}

render();
updateDom();
```

### Proposed Pipeline

#### Phase 1: Single WASM Binary
```
Source (.zx files with 'use client')
    ↓
Transpile → Zig code
    ↓
Compile → Single app.wasm
    ↓
Deploy → Served with HTML/JS
```

**Pros**:
- Simple to implement
- One HTTP request
- Easier debugging

**Cons**:
- Large initial download
- Can't code-split
- Everything loads upfront

#### Phase 2+: Multiple WASM Files (Future)
```
pages/home.zx → home.wasm
pages/about.zx → about.wasm
components/button.zx → button.wasm
```

**Pros**:
- Lazy loading
- Faster initial load
- Better caching

**Cons**:
- More complex build
- Multiple HTTP requests
- Shared dependencies tricky

**Decision**: Start with single WASM, optimize later.

## Rendering Approaches

### 1. Shared Memory Buffer (Current zx-wasm-renderer)

**How it works**:
- WASM exports fixed-size buffer
- Zig writes HTML string to buffer
- JS reads buffer, updates DOM

**Pros**:
- Simple, proven approach
- Zero-copy for small renders
- Direct memory access

**Cons**:
- Fixed buffer size (what if HTML > buffer?)
- Manual memory management
- String-based (no structured data)

**Implementation**:
```zig
export var render_buffer: [4096]u8 = undefined;

export fn render() usize {
    const html = component.render(allocator) catch return 0;
    defer allocator.free(html);

    const len = @min(html.len, render_buffer.len);
    @memcpy(render_buffer[0..len], html[0..len]);
    return len;
}
```

### 2. Dynamic Memory Allocation

**How it works**:
- WASM allocates memory dynamically
- Returns pointer + length to JS
- JS reads from that location
- Needs explicit free() call

**Pros**:
- No size limit
- More flexible
- Can return arbitrary data

**Cons**:
- Manual memory management
- JS must call free()
- More complex protocol

**Implementation**:
```zig
export fn render() [*]const u8 {
    const html = component.render(allocator) catch return "";
    // ⚠️ Memory leak - who frees this?
    return html.ptr;
}

export fn renderLen() usize {
    return last_render_len;
}

export fn freeRender(ptr: [*]u8, len: usize) void {
    allocator.free(ptr[0..len]);
}
```

### 3. Virtual DOM / Diffing

**How it works**:
- WASM outputs operations, not HTML
- JS applies DOM operations
- Only update what changed

**Pros**:
- Minimal DOM updates (fast!)
- Structured data
- More React-like

**Cons**:
- Complex implementation
- Requires serialization
- Larger WASM binary

**Implementation concept**:
```zig
const DomOp = union(enum) {
    create: struct { tag: []const u8, id: u32 },
    update_text: struct { id: u32, text: []const u8 },
    set_attr: struct { id: u32, key: []const u8, value: []const u8 },
    remove: u32,
};

export fn render() void {
    const old_vdom = current_vdom;
    const new_vdom = component.render();
    const ops = diff(old_vdom, new_vdom);

    // Write ops to buffer for JS to consume
    serializeOps(ops, &op_buffer);
}
```

### 4. Web Components / Custom Elements

**How it works**:
- Define custom HTML elements
- WASM manages lifecycle
- Browser handles rendering

**Pros**:
- Standards-based
- Encapsulation
- Shadow DOM support

**Cons**:
- More API surface
- Browser compatibility
- Learning curve

### 5. Canvas/WebGL Rendering

**How it works**:
- Skip DOM entirely
- WASM draws to canvas
- Full control over rendering

**Pros**:
- Maximum performance
- Pixel-perfect control
- No DOM overhead

**Cons**:
- Not accessible
- No native controls
- More complex

**Use case**: Games, visualizations, not general UI.

## Implementation Phases

### Phase 1: Basic CSR (One-Time Render)

**Goal**: Render a static component from WASM once.

**Tasks**:
1. ✅ Detect 'use client' in transpiler
2. ⬜ Create client build pipeline
   - Separate build.zig for client code
   - Target wasm32-freestanding
   - Export render function
3. ⬜ Transpile to WASM-compatible Zig
   - No server-side APIs
   - Use wasm_allocator
   - Export render function
4. ⬜ Create JS loader
   - Fetch and instantiate WASM
   - Call render()
   - Update DOM
5. ⬜ Test with simple component

**Success criteria**:
- `<div>Hello from WASM!</div>` renders in browser
- No errors in console
- WASM file < 50KB

### Phase 2: Stateful Rendering

**Goal**: Mutate state and re-render.

**Tasks**:
1. ⬜ Add mutable state in WASM
2. ⬜ Export mutation functions
3. ⬜ Wire up event handlers (onclick, etc)
4. ⬜ Call render() after mutation
5. ⬜ Update DOM with new HTML

**Example**:
```zig
var count: u32 = 0;

export fn increment() void {
    count += 1;
}

export fn render() void {
    const html = std.fmt.allocPrint(
        allocator,
        "<div>Count: {d}</div>",
        .{count}
    ) catch return;
    // ... copy to buffer
}
```

**Success criteria**:
- Button click increments counter
- DOM updates automatically
- No memory leaks

### Phase 3: useState Hook (Automatic Re-rendering)

**Goal**: React-style hooks that trigger re-renders.

**Concept**:
```zig
const Counter = struct {
    pub fn Component(self: *Counter) ![]const u8 {
        const count = useState(u32, "count", 0);

        return try std.fmt.allocPrint(
            allocator,
            \\<div>
            \\  Count: {d}
            \\  <button onclick="setState('count', {d})">Increment</button>
            \\</div>
            ,
            .{count.*, count.* + 1}
        );
    }
};
```

**Challenges**:
- Track which state changed
- Invalidate component
- Batch re-renders
- Hooks must be called in order (like React)

**Tasks**:
1. ⬜ Design state management API
2. ⬜ Track component state
3. ⬜ Detect state changes
4. ⬜ Auto-trigger render
5. ⬜ Efficient re-rendering (only affected components)

**Success criteria**:
- Declarative state management
- Automatic re-renders
- Multiple independent state variables
- Minimal re-renders (don't render whole tree)

## Design Decisions & Tradeoffs

### Why WASM?
- Run Zig in browser
- Near-native performance
- Type safety all the way down
- No JavaScript compilation step

### Why Not Just JS?
- Keep codebase in Zig
- Leverage Zig's type system
- Better performance for heavy computation
- Unified language for client + server

### Single WASM vs Multiple
**Single** (Phase 1):
- Simpler to implement
- Good for small apps
- One cache entry

**Multiple** (Future):
- Better for large apps
- Code splitting
- Lazy loading

### String-based vs Virtual DOM
**String-based** (Phase 1-2):
- Simple to implement
- Easy to debug (inspect HTML)
- Works with existing tools

**Virtual DOM** (Future):
- More efficient updates
- Finer-grained control
- More React-like

### Memory Management
**Fixed Buffer** (Phase 1):
- Simple
- Predictable
- Fast

**Dynamic Allocation** (Future):
- Flexible
- Handles large renders
- Needs discipline

## Open Questions

1. **How to handle routing?**
   - Client-side router in WASM?
   - History API integration?
   - Server-side hydration?

2. **How to share code between server and client?**
   - Some code can't run in WASM (file I/O, etc)
   - Need conditional compilation?
   - Separate module paths?

3. **Hydration strategy?**
   - Server renders initial HTML
   - Client "hydrates" with WASM
   - How to avoid flash of unstyled content?

4. **Asset handling?**
   - CSS in WASM?
   - Images?
   - Fonts?

5. **Developer experience?**
   - Hot reload?
   - Source maps for WASM?
   - Debugging tools?

6. **Error handling?**
   - WASM panics → JS errors?
   - Show errors in development?
   - Production error boundaries?

7. **Testing?**
   - Unit tests for components?
   - Integration tests with DOM?
   - Headless browser?

8. **Build time?**
   - Compiling WASM is slow
   - Incremental builds?
   - Caching strategies?

## Next Steps

1. **Create proof of concept**
   - Single .zx file with 'use client'
   - Transpile to Zig
   - Compile to WASM
   - Render in browser

2. **Extend transpiler**
   - Add WASM build path
   - Generate export functions
   - Handle memory buffer setup

3. **Create runtime library**
   - Client-side rendering utilities
   - State management primitives
   - Event handling

4. **Document patterns**
   - How to structure client components
   - Best practices
   - Performance tips

5. **Optimize build**
   - Minimize WASM size
   - Compression (brotli/gzip)
   - Caching strategy

## References

- `../zx-wasm-renderer` - Working WASM renderer example
- `src/cli/transpile.zig` - Current transpiler implementation
- Zig WASM target docs: https://ziglang.org/documentation/master/#WebAssembly
- WebAssembly MDN: https://developer.mozilla.org/en-US/docs/WebAssembly

## Appendix: Code Size Comparison

Need to benchmark:
- Empty WASM module: ~100 bytes
- Hello world: ~500 bytes
- With allocator: ~10KB
- With fmt: ~20KB
- Full app: ???

Goal: Keep < 100KB for initial load.
