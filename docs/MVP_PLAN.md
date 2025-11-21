# ZX Client-Side Rendering MVP - Implementation Plan

## Overview

This document provides a detailed, actionable implementation plan for adding client-side rendering to the zx web framework. Each task includes specific file locations, code changes, and success criteria.

## Phase 1: Basic Static Rendering (Weeks 1-2)

### Goal
Render a simple static component from WASM with no interactivity.

### 1.1 Modify Transpiler for Placeholder Generation

**File**: `/home/xentropy/src/zx/src/cli/transpile.zig`

**Task**: Modify client component handling to generate placeholders instead of skipping.

**Current Code** (lines 79-81, 633-635):
```zig
if (std.mem.eql(u8, first_line, "'use client'")) {
    log.info("Skipping client-side file: {s}", .{path});
    return;
}
```

**Required Changes**:
```zig
// Line 79-81 - In transpile function
if (std.mem.eql(u8, first_line, "'use client'")) {
    log.info("Generating placeholder for client component: {s}", .{path});
    try generateClientPlaceholder(ctx.allocator, path, outdir);
    try addToClientManifest(ctx.allocator, path);
    return;
}

// Line 633-635 - In transpileSingleFile function
if (std.mem.eql(u8, first_line, "'use client'")) {
    log.info("Generating placeholder for client component: {s}", .{source_path});
    try generateClientPlaceholder(allocator, source_path, output_path);
    try addToClientManifest(allocator, source_path);
    return;
}
```

**New Functions to Add**:
```zig
// Add after line 700 in transpile.zig
fn generateClientPlaceholder(
    allocator: std.mem.Allocator,
    source_path: []const u8,
    output_path: []const u8,
) !void {
    const component_name = extractComponentName(source_path);
    const component_id = try generateComponentId(allocator, source_path);

    const placeholder_code = try std.fmt.allocPrint(allocator,
        \\// AUTO-GENERATED: Server placeholder for client component
        \\const std = @import("std");
        \\const zx = @import("zx");
        \\
        \\pub fn {s}(ctx: zx.PageContext) zx.Component {{
        \\    var _zx = zx.initWithAllocator(ctx.arena);
        \\    return _zx.zx(.div, .{{
        \\        .attributes = &.{{
        \\            .{{ .name = "id", .value = "{s}" }},
        \\            .{{ .name = "data-wasm-component", .value = "{s}" }},
        \\            .{{ .name = "data-wasm-mount", .value = "" }},
        \\        }},
        \\        .children = &.{{ _zx.txt("Loading...") }},
        \\    }});
        \\}}
        ,
        .{ component_name, component_id, component_name }
    );
    defer allocator.free(placeholder_code);

    try std.fs.cwd().writeFile(.{
        .sub_path = output_path,
        .data = placeholder_code,
    });
}

fn addToClientManifest(
    allocator: std.mem.Allocator,
    source_path: []const u8,
) !void {
    const manifest_path = ".zx/client_components.json";
    // Implementation to append to JSON manifest
}
```

**Success Criteria**:
- [ ] Transpiler generates `.zig` files for client components
- [ ] Generated files contain placeholder divs with data attributes
- [ ] Manifest file created at `.zx/client_components.json`

### 1.2 Create Client Build Command

**New File**: `/home/xentropy/src/zx/src/cli/build_client.zig`

**Implementation**:
```zig
const std = @import("std");
const zli = @import("zli");
const log = std.log.scoped(.build_client);

pub fn register(
    writer: *std.io.Writer,
    reader: *std.io.Reader,
    allocator: std.mem.Allocator,
) !*zli.Command {
    const cmd = try zli.Command.init(writer, reader, allocator, .{
        .name = "build-client",
        .description = "Build client-side WASM components",
    }, buildClient);

    return cmd;
}

fn buildClient(ctx: zli.CommandContext) !void {
    // 1. Read manifest
    const manifest_path = ".zx/client_components.json";
    const manifest = try std.fs.cwd().readFileAlloc(
        ctx.allocator,
        manifest_path,
        std.math.maxInt(usize),
    );
    defer ctx.allocator.free(manifest);

    // 2. Parse JSON
    const components = try parseManifest(ctx.allocator, manifest);
    defer components.deinit();

    // 3. Generate client entry point
    try generateClientMain(ctx.allocator, components);

    // 4. Build WASM
    try buildWasm(ctx.allocator);

    log.info("Client build complete: .zx/assets/app.wasm", .{});
}
```

**Update**: `/home/xentropy/src/zx/src/cli.zig` (main CLI entry)
```zig
// Add import
const build_client = @import("cli/build_client.zig");

// Register command (in appropriate location)
try app.addCommand(try build_client.register(writer, reader, allocator));
```

**Success Criteria**:
- [ ] `zx build-client` command available
- [ ] Command reads manifest successfully
- [ ] Generates client entry point at `.zx/client/main.zig`

### 1.3 Implement Client Transpilation

**File**: `/home/xentropy/src/zx/src/cli/build_client.zig` (continued)

**Add Function**:
```zig
fn transpileForClient(
    allocator: std.mem.Allocator,
    source_path: []const u8,
    output_dir: []const u8,
) !void {
    // Read original .zx file
    const source = try std.fs.cwd().readFileAlloc(
        allocator,
        source_path,
        std.math.maxInt(usize),
    );
    defer allocator.free(source);

    // Skip 'use client' line
    const content_start = std.mem.indexOfScalar(u8, source, '\n') orelse 0;
    const jsx_content = source[content_start + 1..];

    // Parse JSX using existing parser
    const source_z = try allocator.dupeZ(u8, jsx_content);
    defer allocator.free(source_z);

    var result = try zx.Ast.parse(allocator, source_z);
    defer result.deinit(allocator);

    // Wrap with WASM exports
    const client_code = try std.fmt.allocPrint(allocator,
        \\const std = @import("std");
        \\const zx = @import("zx");
        \\
        \\var render_arena = std.heap.ArenaAllocator.init(std.heap.wasm_allocator);
        \\export var render_buffer: [8192]u8 = undefined;
        \\
        \\{s}
        \\
        \\export fn render() usize {{
        \\    _ = render_arena.reset(.retain_capacity);
        \\    const allocator = render_arena.allocator();
        \\    const ctx = zx.PageContext{{ .arena = allocator }};
        \\
        \\    const component = Component(ctx);
        \\    const html = component.toHtml(allocator) catch return 0;
        \\
        \\    const len = @min(html.len, render_buffer.len);
        \\    @memcpy(render_buffer[0..len], html[0..len]);
        \\    return len;
        \\}}
        ,
        .{result.zig_source}
    );
    defer allocator.free(client_code);

    // Write to output
    const output_path = try std.fmt.allocPrint(
        allocator,
        "{s}/{s}.zig",
        .{ output_dir, getComponentName(source_path) }
    );
    defer allocator.free(output_path);

    try std.fs.cwd().writeFile(.{
        .sub_path = output_path,
        .data = client_code,
    });
}
```

**Success Criteria**:
- [ ] Client components transpiled with WASM exports
- [ ] Files created in `.zx/client/` directory
- [ ] Export functions: `render()`, `render_buffer`

### 1.4 Generate Client Entry Point

**File**: `.zx/client/main.zig` (auto-generated)

**Template**:
```zig
const std = @import("std");

// Import all client components
const HomePage = @import("home.zig");
const CounterComponent = @import("counter.zig");

// Component registry
const ComponentType = enum(u8) {
    home = 0,
    counter = 1,
};

var current_component: ComponentType = .home;

export fn setComponent(id: u8) void {
    current_component = @enumFromInt(id);
}

export fn render() usize {
    return switch (current_component) {
        .home => HomePage.render(),
        .counter => CounterComponent.render(),
    };
}
```

**Success Criteria**:
- [ ] Entry point imports all client components
- [ ] Component switching mechanism works
- [ ] Exports required functions

### 1.5 Create Build Configuration

**New File**: `.zx/client/build.zig`

**Content**:
```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .freestanding,
    });

    const wasm = b.addExecutable(.{
        .name = "app",
        .root_module = b.createModule(.{
            .root_source_file = b.path("main.zig"),
            .target = target,
            .optimize = .ReleaseSmall,
            .single_threaded = true,
        }),
    });

    // Add zx module
    const zx = b.createModule(.{
        .root_source_file = b.path("../../src/zx/zx.zig"),
    });
    wasm.root_module.addImport("zx", zx);

    wasm.entry = .disabled;
    wasm.export_memory = true;
    wasm.rdynamic = true;

    const install = b.addInstallFileWithDir(
        wasm.getEmittedBin(),
        .{ .custom = "../assets" },
        "app.wasm",
    );

    b.getInstallStep().dependOn(&install.step);
}
```

**Build Command Integration**:
```zig
// In build_client.zig
fn buildWasm(allocator: std.mem.Allocator) !void {
    const result = try std.process.Child.run(.{
        .allocator = allocator,
        .argv = &.{ "zig", "build", "-Dbuild-file=build.zig" },
        .cwd = ".zx/client",
    });
    defer allocator.free(result.stdout);
    defer allocator.free(result.stderr);

    if (result.term.Exited != 0) {
        log.err("WASM build failed: {s}", .{result.stderr});
        return error.BuildFailed;
    }
}
```

**Success Criteria**:
- [ ] WASM builds without errors
- [ ] Output at `.zx/assets/app.wasm`
- [ ] Binary size < 50KB

### 1.6 Create JavaScript Runtime

**New File**: `.zx/assets/wasm_runtime.js`

**Content**:
```javascript
export class WasmRuntime {
    constructor() {
        this.instance = null;
        this.memory = null;
        this.exports = null;
    }

    async init(wasmPath) {
        console.log(`Loading WASM from ${wasmPath}...`);

        const response = await fetch(wasmPath);
        if (!response.ok) {
            throw new Error(`Failed to fetch WASM: ${response.status}`);
        }

        const buffer = await response.arrayBuffer();
        console.log(`WASM size: ${(buffer.byteLength / 1024).toFixed(2)} KB`);

        const result = await WebAssembly.instantiate(buffer, {
            env: {}
        });

        this.instance = result.instance;
        this.exports = result.instance.exports;
        this.memory = this.exports.memory;

        console.log('WASM loaded successfully');
        return this;
    }

    render(elementId = 'root') {
        if (!this.exports.render) {
            throw new Error('WASM module does not export render()');
        }

        const length = this.exports.render();
        if (length === 0) {
            console.warn('Render returned 0 bytes');
            return;
        }

        const html = this.readString(this.exports.render_buffer.value, length);
        console.log(`Rendered ${length} bytes of HTML`);

        const element = document.getElementById(elementId);
        if (!element) {
            throw new Error(`Element #${elementId} not found`);
        }

        element.innerHTML = html;
    }

    readString(ptr, length) {
        const memory = new Uint8Array(this.memory.buffer);
        const bytes = memory.slice(ptr, ptr + length);
        return new TextDecoder('utf-8').decode(bytes);
    }

    async hydrate() {
        // Find all components with data-wasm-mount
        const components = document.querySelectorAll('[data-wasm-mount]');

        for (const component of components) {
            const componentName = component.dataset.wasmComponent;
            const componentId = component.id;

            console.log(`Hydrating ${componentName} at #${componentId}`);

            // Set current component (if multiple)
            if (this.exports.setComponent) {
                // Map component name to enum value
                const componentMap = { 'HomePage': 0, 'Counter': 1 };
                this.exports.setComponent(componentMap[componentName] || 0);
            }

            // Render into element
            this.render(componentId);
        }
    }
}

// Auto-initialize
if (typeof window !== 'undefined') {
    window.WasmRuntime = WasmRuntime;
}
```

**Success Criteria**:
- [ ] Runtime loads WASM successfully
- [ ] Hydration finds placeholder divs
- [ ] Components render into placeholders

### 1.7 HTML Integration

**Template Update**: Server HTML generation

**Add to transpiler output**:
```html
<script type="module">
    import { WasmRuntime } from '/assets/wasm_runtime.js';

    async function initClient() {
        const runtime = new WasmRuntime();
        await runtime.init('/assets/app.wasm');
        await runtime.hydrate();
    }

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', initClient);
    } else {
        initClient();
    }
</script>
```

**Success Criteria**:
- [ ] Script tags included in HTML
- [ ] WASM loads on page load
- [ ] Components hydrate automatically

### 1.8 Create Test Component

**File**: `/home/xentropy/src/zx/test/client/hello.zx`

**Content**:
```zig
'use client'

pub fn Component(ctx: zx.PageContext) zx.Component {
    return (
        <div @allocator={ctx.arena}>
            <h1>Hello from WASM!</h1>
            <p>This component is rendered client-side using WebAssembly.</p>
            <time>{["Current render: ":s]}{[std.time.timestamp():d]}</time>
        </div>
    );
}

const std = @import("std");
const zx = @import("zx");
```

**Success Criteria**:
- [ ] Component transpiles without errors
- [ ] Placeholder generated on server
- [ ] WASM renders correct HTML
- [ ] Displays in browser

### 1.9 Testing Strategy

**Test Script**: `/home/xentropy/src/zx/test/test_csr.sh`

```bash
#!/bin/bash
set -e

echo "Testing CSR Implementation..."

# Clean previous build
rm -rf .zx

# Transpile test component
echo "1. Transpiling..."
zx transpile test/client

# Build client
echo "2. Building client..."
zx build-client

# Check outputs
echo "3. Verifying outputs..."
[ -f ".zx/client_components.json" ] || exit 1
[ -f ".zx/assets/app.wasm" ] || exit 1
[ -f ".zx/assets/wasm_runtime.js" ] || exit 1

# Check WASM size
SIZE=$(stat -c%s ".zx/assets/app.wasm")
echo "WASM size: $SIZE bytes"
[ $SIZE -lt 51200 ] || echo "WARNING: WASM exceeds 50KB"

# Start dev server
echo "4. Starting server..."
zx dev --port 8080 &
SERVER_PID=$!

sleep 2

# Test with curl
echo "5. Testing endpoints..."
curl -s http://localhost:8080/ | grep "data-wasm-mount" || exit 1

# Cleanup
kill $SERVER_PID

echo "✓ All tests passed!"
```

**Success Criteria**:
- [ ] All build steps complete
- [ ] WASM size within limits
- [ ] HTML contains hydration markers
- [ ] No runtime errors

## Phase 2: Interactive Components (Weeks 3-5)

### 2.1 Add State Management

**Modify**: Client component template

**Add to transpiled output**:
```zig
// Component state
var count: u32 = 0;
var username: []const u8 = "";

// State mutation functions
export fn increment() void {
    count += 1;
}

export fn setUsername(ptr: [*]const u8, len: usize) void {
    const new_username = ptr[0..len];
    username = new_username;
}
```

### 2.2 Event Handler Registration

**Modify**: JSX transpilation in `/home/xentropy/src/zx/src/zx/Transpiler_prototype.zig`

**Detect and register event handlers**:
```zig
// When processing attributes
if (std.mem.startsWith(u8, attr.name, "on")) {
    // Register as WASM export
    const handler_name = attr.value;
    try registerEventHandler(handler_name);
}
```

### 2.3 JavaScript Event Wiring

**Update**: `wasm_runtime.js`

```javascript
exposeEventHandlers() {
    const self = this;

    // Find all exported functions starting with 'on' or specific names
    const handlers = ['increment', 'decrement', 'reset', 'handleClick'];

    for (const handler of handlers) {
        if (self.exports[handler]) {
            window[handler] = () => {
                self.exports[handler]();
                self.render(); // Re-render after state change
            };
        }
    }
}
```

### 2.4 Re-rendering System

**Add to runtime**:
```javascript
class RenderQueue {
    constructor(runtime) {
        this.runtime = runtime;
        this.pending = false;
    }

    schedule() {
        if (this.pending) return;
        this.pending = true;

        requestAnimationFrame(() => {
            this.runtime.render();
            this.pending = false;
        });
    }
}
```

### 2.5 Test Interactive Component

**File**: `/home/xentropy/src/zx/test/client/counter.zx`

```zig
'use client'

var count: u32 = 0;

pub fn Component(ctx: zx.PageContext) zx.Component {
    return (
        <div @allocator={ctx.arena}>
            <h1>Counter: {[count:d]}</h1>
            <button onclick="increment()">+</button>
            <button onclick="decrement()">-</button>
            <button onclick="reset()">Reset</button>
        </div>
    );
}

export fn increment() void {
    count += 1;
}

export fn decrement() void {
    if (count > 0) count -= 1;
}

export fn reset() void {
    count = 0;
}

const zx = @import("zx");
```

## Phase 3: useState Hook (Weeks 6-9)

### 3.1 Hook Context System

**New File**: `/home/xentropy/src/zx/src/zx/hooks.zig`

```zig
pub const HookContext = struct {
    state_index: usize = 0,
    states: std.ArrayList(StateValue),
    component_id: u32,
    dirty: bool = false,

    pub fn useState(self: *HookContext, comptime T: type, initial: T) *T {
        // Implementation
    }
};
```

### 3.2 Component Registration

**Track component instances**:
```zig
const ComponentRegistry = struct {
    components: std.AutoHashMap(u32, *HookContext),

    pub fn register(self: *ComponentRegistry, id: u32) !*HookContext {
        // Create and track hook context
    }
};
```

### 3.3 Automatic Re-rendering

**Implement setState with auto-render**:
```zig
pub fn setState(self: *HookContext, index: usize, value: anytype) void {
    self.states.items[index] = value;
    self.dirty = true;

    // Trigger re-render
    scheduleRender(self.component_id);
}
```

## Testing & Validation

### Unit Tests

**File**: `/home/xentropy/src/zx/test/transpiler_test.zig`

```zig
test "transpiler generates placeholder for client component" {
    // Test placeholder generation
}

test "manifest correctly tracks client components" {
    // Test manifest creation
}
```

### Integration Tests

**File**: `/home/xentropy/src/zx/test/integration_test.js`

```javascript
import { WasmRuntime } from '../.zx/assets/wasm_runtime.js';
import { test } from 'node:test';
import assert from 'node:assert';

test('WASM runtime loads and renders', async () => {
    const runtime = new WasmRuntime();
    await runtime.init('.zx/assets/app.wasm');

    // Mock DOM
    global.document = {
        getElementById: () => ({ innerHTML: '' })
    };

    runtime.render();
    assert.ok(true, 'Render completed without error');
});
```

### E2E Tests

**Using Playwright**:
```javascript
test('Counter increments on click', async ({ page }) => {
    await page.goto('http://localhost:8080/counter');

    const counter = page.locator('h1');
    await expect(counter).toContainText('Counter: 0');

    await page.click('button:text("+")');
    await expect(counter).toContainText('Counter: 1');
});
```

## Performance Benchmarks

### Metrics to Track

1. **Build Performance**:
   - Transpilation time per component
   - WASM compilation time
   - Total build time

2. **Runtime Performance**:
   - WASM load time
   - Initial render time
   - Re-render time
   - Memory usage

3. **Binary Size**:
   - Per-component overhead
   - Framework size
   - Compression ratio

### Benchmark Script

```bash
#!/bin/bash
# benchmark.sh

echo "Running CSR Benchmarks..."

# Build benchmark
TIME_START=$(date +%s%N)
zx build-client
TIME_END=$(date +%s%N)
BUILD_TIME=$((($TIME_END - $TIME_START) / 1000000))
echo "Build time: ${BUILD_TIME}ms"

# Size benchmark
WASM_SIZE=$(stat -c%s .zx/assets/app.wasm)
echo "WASM size: ${WASM_SIZE} bytes"

# Runtime benchmark (using Deno)
deno run --allow-read benchmark_runtime.js
```

## Deployment Checklist

### Pre-deployment
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Performance targets met
- [ ] Security review complete

### Deployment Steps
1. [ ] Tag release version
2. [ ] Build production artifacts
3. [ ] Update CDN assets
4. [ ] Deploy documentation
5. [ ] Announce to community

### Post-deployment
- [ ] Monitor error rates
- [ ] Gather user feedback
- [ ] Track adoption metrics
- [ ] Plan next iteration

## Success Criteria Summary

### Phase 1 Complete When:
- ✅ Transpiler generates placeholders for 'use client' components
- ✅ Client build command successfully creates WASM
- ✅ JavaScript runtime hydrates components
- ✅ Test component renders "Hello from WASM!"
- ✅ WASM binary < 50KB
- ✅ No console errors

### Phase 2 Complete When:
- ✅ State changes trigger re-renders
- ✅ Event handlers work (onclick, etc.)
- ✅ Counter component fully functional
- ✅ Multiple component instances supported
- ✅ Re-render performance < 16ms

### Phase 3 Complete When:
- ✅ useState hook API implemented
- ✅ Multiple state variables work independently
- ✅ Automatic re-rendering on state change
- ✅ Component registry tracks instances
- ✅ No memory leaks detected

## Timeline

### Week 1
- Days 1-2: Transpiler modifications
- Days 3-4: Client build command
- Day 5: Testing & debugging

### Week 2
- Days 1-2: WASM compilation pipeline
- Days 3-4: JavaScript runtime
- Day 5: Integration testing

### Weeks 3-4
- Event handling system
- State management
- Re-rendering optimization

### Weeks 5-6
- Hook context implementation
- Component registry
- useState API

### Weeks 7-8
- Performance optimization
- Documentation
- Example applications

### Week 9
- Final testing
- Release preparation
- Community announcement

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| WASM size exceeds target | High | Medium | Monitor in CI, optimize aggressively |
| Browser compatibility issues | Medium | High | Test all browsers, provide polyfills |
| Performance regression | Medium | High | Benchmark every commit |
| Hydration mismatches | Medium | Medium | Clear error messages, recovery mode |
| Memory leaks | Low | High | Use arena allocators, test thoroughly |

## Next Actions

1. **Immediate** (Today):
   - Create feature branch: `git checkout -b csr-implementation`
   - Set up test directory structure
   - Begin transpiler modifications

2. **This Week**:
   - Complete Phase 1.1-1.3
   - Create first test component
   - Get basic WASM building

3. **Next Week**:
   - Complete Phase 1.4-1.9
   - Full integration test
   - Begin Phase 2 planning

## Conclusion

This implementation plan provides a clear, actionable path to CSR in zx. Each phase builds on the previous, with concrete success criteria and testing strategies. The timeline is aggressive but achievable, with risk mitigation built in. Success depends on disciplined execution and early validation of core assumptions.