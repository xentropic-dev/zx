# Quick Start: CSR Implementation

This is a practical guide to get started with Phase 1 implementation.

## Prerequisites

- Zig 0.13.0 or later
- Modern web browser with WASM support
- HTTP server for testing (Python, Node, or built-in zx server)

## Phase 1: Hello World from WASM

### Step 1: Create Test Component

Create `test/client/hello.zx`:

```zig
'use client'

pub fn Component(ctx: zx.PageContext) zx.Component {
    return (
        <div>
            <h1>Hello from WASM!</h1>
            <p>This is rendered client-side using Zig compiled to WebAssembly.</p>
        </div>
    );
}

const zx = @import("zx");
```

### Step 2: Modify Transpiler (Pass 1: Generate Placeholder)

Edit `src/cli/transpile.zig`:

**Current code (lines 76-82)**:
```zig
if (std.mem.eql(u8, first_line, "'use client'")) {
    log.info("Skipping client-side file: {s}", .{path});
    return;  // ← Remove this!
}
```

**Change to**:
```zig
const is_client_side = std.mem.eql(u8, first_line, "'use client'");
if (is_client_side) {
    log.info("Generating placeholder for client component: {s}", .{path});
    try handleClientComponent(allocator, path, output_path);
    return; // Continue to next file after generating placeholder
}
```

**Add new function**:
```zig
fn handleClientComponent(
    allocator: std.mem.Allocator,
    source_path: []const u8,
    output_path: []const u8,
) !void {
    // Generate unique ID from path
    const component_id = try generateComponentId(source_path);

    // Generate placeholder server component
    const placeholder = try std.fmt.allocPrint(allocator,
        \\// AUTO-GENERATED: Server placeholder for 'use client' component
        \\const zx = @import("zx");
        \\
        \\pub fn Component(ctx: zx.PageContext) zx.Component {{
        \\    var _zx = zx.initWithAllocator(ctx.arena);
        \\    return _zx.zx(.div, .{{
        \\        .attributes = &.{{
        \\            .{{ .name = "id", .value = "{s}" }},
        \\            .{{ .name = "data-wasm-component", .value = "Component" }},
        \\            .{{ .name = "data-wasm-mount", .value = "" }},
        \\        }},
        \\        .children = &.{{ _zx.txt("Loading...") }},
        \\    }});
        \\}}
        \\
        , .{component_id});
    defer allocator.free(placeholder);

    // Write placeholder to output
    try std.fs.cwd().writeFile(.{ .sub_path = output_path, .data = placeholder });

    // Add to manifest for Pass 2
    try addToClientManifest(allocator, source_path, component_id);
}
```

### Step 3: Implement Pass 2 - Client Build Command

Create a new command to build client components:

**New file**: `src/cli/build_client.zig`

```zig
const std = @import("std");

pub fn buildClient(allocator: std.mem.Allocator) !void {
    // Read manifest
    const manifest = try std.fs.cwd().readFileAlloc(
        allocator,
        ".zx/client_components.json",
        std.math.maxInt(usize),
    );
    defer allocator.free(manifest);

    const components = try std.json.parseFromSlice(
        []ComponentEntry,
        allocator,
        manifest,
        .{},
    );
    defer components.deinit();

    // Transpile each component for WASM
    for (components.value) |entry| {
        try transpileForWasm(allocator, entry.source_path, entry.id);
    }

    // Generate client main
    try generateClientMain(allocator, components.value);

    // Build WASM
    try buildWasm(allocator);
}

const ComponentEntry = struct {
    id: []const u8,
    name: []const u8,
    source_path: []const u8,
};
```

### Step 4: Generate Client Main

Create `.zx/client_main.zig`:

```zig
const std = @import("std");

// Allocator for rendering
var render_arena = std.heap.ArenaAllocator.init(std.heap.wasm_allocator);

// Shared buffer for output
export var render_buffer: [8192]u8 = undefined;

// Import your component (adjust path as needed)
const hello = @import("pages/hello.zig");

export fn render() usize {
    // Reset arena for fresh allocations
    _ = render_arena.reset(.retain_capacity);
    const allocator = render_arena.allocator();

    // Create page context
    const ctx = zx.PageContext{ .arena = allocator };

    // Render component (returns zx.Component)
    const component = hello.Component(ctx);
    const html = component.toHtml(allocator) catch |err| {
        std.debug.print("Render error: {}\n", .{err});
        return 0;
    };

    // Copy to shared buffer
    const len = @min(html.len, render_buffer.len);
    @memcpy(render_buffer[0..len], html[0..len]);

    return len;
}

// Cleanup on exit (optional)
export fn cleanup() void {
    render_arena.deinit();
}
```

### Step 5: Create Client Build Script

Create `.zx/build_client.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // WASM target
    const target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .freestanding,
    });

    // Build WASM binary
    const wasm = b.addExecutable(.{
        .name = "app",
        .root_module = b.createModule(.{
            .root_source_file = b.path("client_main.zig"),
            .target = target,
            .optimize = .ReleaseSmall,
            .single_threaded = true,
        }),
    });

    wasm.entry = .disabled;
    wasm.export_memory = true;
    wasm.rdynamic = true;

    // Install to assets/
    const install_wasm = b.addInstallFileWithDir(
        wasm.getEmittedBin(),
        .{ .custom = "assets" },
        "app.wasm",
    );

    b.getInstallStep().dependOn(&install_wasm.step);
}
```

### Step 6: Build WASM (Run Pass 2)

```bash
cd .zx
zig build -Dbuild-file=build_client.zig
```

This should produce `.zx/assets/app.wasm`.

### Step 7: Create JavaScript Runtime

Create `.zx/assets/wasm_runtime.js`:

```javascript
/**
 * WASM Runtime for ZX Client Components
 */
export class WasmRuntime {
    constructor() {
        this.instance = null;
        this.memory = null;
        this.exports = null;
    }

    /**
     * Initialize WASM module
     * @param {string} wasmPath - Path to .wasm file
     */
    async init(wasmPath) {
        console.log(`Loading WASM from ${wasmPath}...`);

        const response = await fetch(wasmPath);
        if (!response.ok) {
            throw new Error(`Failed to fetch WASM: ${response.status}`);
        }

        const buffer = await response.arrayBuffer();
        console.log(`WASM size: ${(buffer.byteLength / 1024).toFixed(2)} KB`);

        const result = await WebAssembly.instantiate(buffer, {
            // Import object (empty for now)
            env: {}
        });

        this.instance = result.instance;
        this.exports = result.instance.exports;
        this.memory = this.exports.memory;

        console.log('WASM loaded successfully');
        return this;
    }

    /**
     * Call render and update DOM
     * @param {string} rootId - ID of element to render into
     */
    render(rootId = 'root') {
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

        const root = document.getElementById(rootId);
        if (!root) {
            throw new Error(`Element #${rootId} not found`);
        }

        root.innerHTML = html;
    }

    /**
     * Read string from WASM memory
     * @param {number} ptr - Pointer to start of string
     * @param {number} length - Length of string in bytes
     * @returns {string}
     */
    readString(ptr, length) {
        const memory = new Uint8Array(this.memory.buffer);
        const bytes = memory.slice(ptr, ptr + length);
        return new TextDecoder('utf-8').decode(bytes);
    }

    /**
     * Write string to WASM memory
     * @param {number} ptr - Pointer to write to
     * @param {string} str - String to write
     * @returns {number} - Bytes written
     */
    writeString(ptr, str) {
        const bytes = new TextEncoder().encode(str);
        const memory = new Uint8Array(this.memory.buffer);
        memory.set(bytes, ptr);
        return bytes.length;
    }

    /**
     * Cleanup WASM resources
     */
    cleanup() {
        if (this.exports.cleanup) {
            this.exports.cleanup();
        }
    }
}

// Auto-initialize if running as module
if (typeof window !== 'undefined') {
    window.WasmRuntime = WasmRuntime;
}
```

### Step 8: Create HTML Page

Create `.zx/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ZX WASM Test</title>
    <style>
        body {
            font-family: system-ui, -apple-system, sans-serif;
            max-width: 800px;
            margin: 2rem auto;
            padding: 0 1rem;
        }
        #root {
            border: 2px solid #ccc;
            border-radius: 8px;
            padding: 1rem;
            min-height: 100px;
        }
        .loading {
            color: #666;
            font-style: italic;
        }
    </style>
</head>
<body>
    <h1>ZX Client-Side Rendering Test</h1>

    <div id="root" class="loading">
        Loading WASM...
    </div>

    <script type="module">
        import { WasmRuntime } from './assets/wasm_runtime.js';

        async function main() {
            try {
                // Initialize runtime
                const runtime = new WasmRuntime();
                await runtime.init('./assets/app.wasm');

                // Initial render
                runtime.render('root');

                // Store globally for debugging
                window.runtime = runtime;

                console.log('✓ WASM initialized and rendered');
            } catch (error) {
                console.error('Failed to initialize WASM:', error);
                document.getElementById('root').innerHTML = `
                    <div style="color: red;">
                        <strong>Error loading WASM:</strong>
                        <pre>${error.message}</pre>
                    </div>
                `;
            }
        }

        main();
    </script>
</body>
</html>
```

### Step 9: Serve and Test

Start a local server:

```bash
# Python 3
cd .zx
python3 -m http.server 8000

# Or Node.js
npx serve .zx

# Or use zx dev server (if available)
zx dev
```

Open `http://localhost:8000` in your browser.

You should see:
```
Hello from WASM!
This is rendered client-side using Zig compiled to WebAssembly.
```

### Step 10: Verify

Open browser DevTools and check:

1. **Console**: Should see logs:
   ```
   Loading WASM from ./assets/app.wasm...
   WASM size: XX.XX KB
   WASM loaded successfully
   Rendered XXX bytes of HTML
   ✓ WASM initialized and rendered
   ```

2. **Network**: Should see `app.wasm` loaded successfully

3. **Elements**: Inspect the `#root` div - should contain rendered HTML

4. **Performance**: Initial load should be < 100ms

## Phase 2: Add Interactivity

### Step 1: Add State to Component

Modify `test/client/hello.zx`:

```zig
'use client'

var count: u32 = 0;

pub fn Component(ctx: zx.PageContext) zx.Component {
    return (
        <div>
            <h1>Counter: {[count:d]}</h1>
            <button onclick="increment()">Increment</button>
            <button onclick="decrement()">Decrement</button>
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

### Step 2: Wire Up Events in JS

Update `wasm_runtime.js`:

```javascript
async init(wasmPath) {
    // ... existing init code ...

    // Expose WASM functions to window
    this.exposeToWindow();

    return this;
}

/**
 * Expose WASM functions to window for onclick handlers
 */
exposeToWindow() {
    const self = this;

    // Helper to call WASM and re-render
    const createHandler = (fnName) => {
        return function() {
            if (self.exports[fnName]) {
                self.exports[fnName]();
                self.render();
            } else {
                console.error(`WASM function ${fnName} not found`);
            }
        };
    };

    // Auto-expose common functions
    const functions = ['increment', 'decrement', 'reset', 'handleClick'];
    for (const fn of functions) {
        if (self.exports[fn]) {
            window[fn] = createHandler(fn);
        }
    }
}
```

### Step 3: Rebuild and Test

```bash
cd .zx
zig build -Dbuild-file=build_client.zig
```

Refresh browser - buttons should work!

## Debugging Tips

### Check WASM Exports

In browser console:
```javascript
console.log(window.runtime.exports);
// Should show: render, increment, decrement, reset, memory, render_buffer
```

### Inspect Memory

```javascript
const mem = new Uint8Array(window.runtime.memory.buffer);
console.log(mem.slice(0, 100)); // First 100 bytes
```

### Monitor Render Performance

```javascript
performance.mark('render-start');
window.runtime.render();
performance.mark('render-end');
performance.measure('render', 'render-start', 'render-end');
console.log(performance.getEntriesByName('render')[0].duration);
```

### Check WASM Size

```bash
ls -lh .zx/assets/app.wasm
wasm-opt --version # If available
wasm-opt -Oz .zx/assets/app.wasm -o .zx/assets/app.opt.wasm
```

## Common Issues

### Issue: "render is not a function"

**Cause**: Function not exported from WASM

**Fix**: Make sure function is marked `export` in Zig. Also ensure the component is properly transpiled from JSX:
```zig
export fn render() usize { // Note the 'export' keyword
    // This calls the transpiled Component function
    const component = hello.Component(ctx);
    const html = component.toHtml(allocator) catch return 0;
    // ...
}
```

### Issue: "Cannot read property 'buffer' of undefined"

**Cause**: Memory not exported

**Fix**: Check build.zig:
```zig
wasm.export_memory = true;
```

### Issue: Blank screen / no output

**Cause**: Render returning 0 or error

**Debug**:
1. Check browser console for errors
2. Add logging to Zig code:
   ```zig
   export fn render() usize {
       std.debug.print("Render called!\n", .{});
       // ...
   }
   ```
3. Verify buffer size is sufficient

### Issue: WASM too large

**Cause**: Debug symbols, unused code

**Fix**:
1. Use `.ReleaseSmall` optimization
2. Strip debug info
3. Remove unused imports
4. Use `wasm-opt` tool

### Issue: onclick not working

**Cause**: Function not exposed to window

**Debug**:
1. Check if function exists: `console.log(typeof window.increment)`
2. Verify function exported from WASM
3. Check console for JS errors

## Next Steps

Once Phase 1 is working:

1. [ ] Add more components (todo list, form, etc)
2. [ ] Implement component routing
3. [ ] Add error boundaries
4. [ ] Performance profiling
5. [ ] Move to Phase 2 (useState hooks)

See [csr-roadmap.md](./csr-roadmap.md) for full Phase 2 and 3 plans.

## Resources

- [Zig WASM Docs](https://ziglang.org/documentation/master/#WebAssembly)
- [MDN WebAssembly](https://developer.mozilla.org/en-US/docs/WebAssembly)
- [WASM Reference Manual](https://github.com/WebAssembly/design)
- [zx-wasm-renderer](../zx-wasm-renderer) - Working reference implementation
