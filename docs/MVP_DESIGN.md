# ZX Client-Side Rendering MVP - Technical Design

## Architecture Overview

### System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Build Time                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Source Files (.zx)                                               │
│         ↓                                                          │
│   ┌─────────────────┐                                             │
│   │  Pass 1: Server │                                             │
│   │   Transpilation │                                             │
│   └────────┬────────┘                                             │
│            ↓                                                       │
│   ┌─────────────────────────────────┐                            │
│   │  'use client' Detection         │                            │
│   │      ↓               ↓          │                            │
│   │  Placeholder     Regular        │                            │
│   │  Component      Component       │                            │
│   │      ↓               ↓          │                            │
│   │  .zig file +    .zig file       │                            │
│   │  Manifest                       │                            │
│   └─────────────────────────────────┘                            │
│            ↓                                                       │
│   ┌─────────────────┐                                             │
│   │  Pass 2: Client │                                             │
│   │     Build       │                                             │
│   └────────┬────────┘                                             │
│            ↓                                                       │
│   ┌─────────────────────────────────┐                            │
│   │  Read Manifest                  │                            │
│   │  Re-transpile for WASM          │                            │
│   │  Generate Entry Point           │                            │
│   │  Compile to app.wasm            │                            │
│   └─────────────────────────────────┘                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                          Run Time                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Browser Request                                                  │
│         ↓                                                          │
│   Server Renders HTML                                              │
│   (with placeholders)                                              │
│         ↓                                                          │
│   ┌──────────────────────────────────────┐                       │
│   │  <div id="zx-abc123"                  │                       │
│   │       data-wasm-component="Counter"   │                       │
│   │       data-wasm-mount="">             │                       │
│   │    Loading...                         │                       │
│   │  </div>                              │                       │
│   └──────────────────────────────────────┘                       │
│         ↓                                                          │
│   JavaScript Runtime                                               │
│   Loads app.wasm                                                  │
│         ↓                                                          │
│   WASM Hydrates                                                   │
│   Components                                                      │
│         ↓                                                          │
│   Interactive UI                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Browser    │────▶│  JavaScript  │────▶│    WASM      │
│              │     │   Runtime    │     │   Module     │
│              │     │              │     │              │
│  User Click  │     │  Event       │     │  Handler     │
│              │     │  Dispatch    │     │  Function    │
│              │     │              │     │              │
│              │◀────│  DOM         │◀────│  Render      │
│  UI Update   │     │  Update      │     │  Output      │
└──────────────┘     └──────────────┘     └──────────────┘
```

## Data Structures

### Client Components Manifest

**File**: `.zx/client_components.json`

```json
{
  "version": "1.0",
  "components": [
    {
      "id": "zx-a1b2c3d4e5f6",
      "name": "HomePage",
      "source_path": "site/pages/page.zx",
      "output_path": ".zx/client/home_page.zig",
      "exports": ["render", "init"],
      "handlers": ["increment", "decrement"]
    },
    {
      "id": "zx-b2c3d4e5f6g7",
      "name": "Counter",
      "source_path": "site/components/counter.zx",
      "output_path": ".zx/client/counter.zig",
      "exports": ["render", "init", "increment", "decrement", "reset"],
      "handlers": ["increment", "decrement", "reset"]
    }
  ],
  "metadata": {
    "generated_at": "2025-11-20T12:00:00Z",
    "transpiler_version": "0.1.0",
    "total_components": 2
  }
}
```

### Component State Structure

```zig
pub const ComponentState = struct {
    // Component identifier
    id: []const u8,

    // Render arena for temporary allocations
    render_arena: std.heap.ArenaAllocator,

    // State variables (component-specific)
    state: anytype,

    // Dirty flag for re-rendering
    needs_render: bool,

    // Event handler map
    handlers: std.StringHashMap(HandlerFn),

    // Props from server (if any)
    props: ?[]const u8,
};

pub const HandlerFn = *const fn () void;
```

### Memory Layout

```
WASM Linear Memory (1MB initial, 16MB max)
┌─────────────────────────────────────────┐
│  Static Data (0x0000 - 0x1000)         │
│    - Global variables                   │
│    - String constants                   │
├─────────────────────────────────────────┤
│  Stack (0x1000 - 0x10000)              │
│    - Function calls                     │
│    - Local variables                    │
├─────────────────────────────────────────┤
│  Heap (0x10000 - 0x100000)             │
│    - Dynamic allocations                │
│    - Arena allocators                   │
├─────────────────────────────────────────┤
│  Render Buffer (0x100000 - 0x102000)   │
│    - 8KB fixed buffer                   │
│    - HTML output                        │
├─────────────────────────────────────────┤
│  Component States (0x102000+)           │
│    - Per-component data                 │
│    - Event handler registry             │
└─────────────────────────────────────────┘
```

## API Specifications

### WASM Exports

```zig
// Initialize module with optional props
export fn init(props_ptr: [*]const u8, props_len: usize) void {
    // Parse props JSON
    // Initialize component states
    // Set up event handlers
}

// Render component to HTML
export fn render() usize {
    // Reset render arena
    // Call component render function
    // Write HTML to buffer
    // Return byte count
}

// Get pointer to render buffer
export fn getRenderBuffer() [*]const u8 {
    return &render_buffer[0];
}

// Component-specific exports (generated)
export fn increment() void { /* ... */ }
export fn decrement() void { /* ... */ }
export fn setValue(ptr: [*]const u8, len: usize) void { /* ... */ }

// Memory management
export fn cleanup() void {
    // Free all allocations
    // Reset state
}

// Debug/introspection
export fn getMemoryUsage() usize {
    return current_memory_usage;
}

export fn getComponentCount() u32 {
    return registered_components.count();
}
```

### JavaScript Runtime API

```typescript
interface WasmRuntime {
    // Core initialization
    init(wasmPath: string): Promise<WasmRuntime>;

    // Component management
    hydrate(): Promise<void>;
    hydrateComponent(elementId: string, componentName: string): void;

    // Rendering
    render(elementId?: string): void;
    renderComponent(componentId: string): string;

    // Event handling
    dispatchEvent(componentId: string, eventName: string, data?: any): void;
    registerHandler(name: string, handler: Function): void;

    // State management
    setState(componentId: string, key: string, value: any): void;
    getState(componentId: string, key: string): any;

    // Utilities
    readString(ptr: number, length: number): string;
    writeString(ptr: number, value: string): number;
    readBuffer(): Uint8Array;

    // Lifecycle
    cleanup(): void;
    destroy(): void;
}
```

### Transpiler API Extensions

```zig
// New functions for client component handling
pub fn detectClientDirective(source: []const u8) bool {
    const first_line = getFirstLine(source);
    return std.mem.eql(u8, first_line, "'use client'");
}

pub fn generatePlaceholder(
    allocator: std.mem.Allocator,
    component_name: []const u8,
    component_id: []const u8,
) ![]const u8 {
    // Generate server-side placeholder component
}

pub fn transpileForWasm(
    allocator: std.mem.Allocator,
    source: []const u8,
    options: TranspileOptions,
) !TranspileResult {
    // Transpile with WASM-specific transformations
}

pub const TranspileOptions = struct {
    target: enum { server, wasm },
    optimize: bool = false,
    source_map: bool = false,
    component_name: []const u8,
    component_id: []const u8,
};

pub const TranspileResult = struct {
    zig_source: []const u8,
    exports: []const []const u8,
    handlers: []const []const u8,
    dependencies: []const []const u8,
};
```

## Build Process Details

### Pass 1: Server Transpilation

1. **Input Processing**
   ```
   site/pages/counter.zx
   └── Read file
   └── Check for 'use client'
   └── Decision point
   ```

2. **Placeholder Generation** (if client component)
   ```zig
   pub fn generatePlaceholder(component: ComponentMetadata) ![]const u8 {
       return try std.fmt.allocPrint(allocator,
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
           \\        .children = &.{{
           \\            _zx.txt("Loading..."),
           \\        }},
           \\    }});
           \\}}
           ,
           .{ component.name, component.id, component.name }
       );
   }
   ```

3. **Manifest Update**
   ```zig
   fn updateManifest(component: ComponentMetadata) !void {
       var manifest = try readManifest() orelse Manifest.init();
       try manifest.components.append(component);
       try writeManifest(manifest);
   }
   ```

### Pass 2: Client Build

1. **Manifest Reading**
   ```zig
   const manifest = try std.json.parseFromSlice(
       Manifest,
       allocator,
       manifest_json,
       .{ .ignore_unknown_fields = true }
   );
   ```

2. **Component Re-transpilation**
   ```zig
   for (manifest.components) |component| {
       const source = try readFile(component.source_path);
       const client_code = try transpileForClient(source);
       try writeFile(component.output_path, client_code);
   }
   ```

3. **Entry Point Generation**
   ```zig
   fn generateMainZig(components: []ComponentMetadata) ![]const u8 {
       var imports = std.ArrayList(u8).init(allocator);
       var registry = std.ArrayList(u8).init(allocator);

       for (components) |comp| {
           try imports.writer().print(
               "const {s} = @import(\"{s}\");\n",
               .{ comp.name, comp.output_path }
           );

           try registry.writer().print(
               "    .{s} => {s}.render(),\n",
               .{ comp.name, comp.name }
           );
       }

       // Generate complete main.zig
   }
   ```

4. **WASM Compilation**
   ```bash
   zig build-exe \
       -target wasm32-freestanding \
       -O ReleaseSmall \
       -fno-entry \
       -rdynamic \
       --export-memory \
       .zx/client/main.zig
   ```

## Hydration Mechanism

### Server HTML Output

```html
<!DOCTYPE html>
<html>
<head>
    <title>ZX App</title>
    <script type="module" src="/assets/wasm_runtime.js"></script>
</head>
<body>
    <!-- Server-rendered content -->
    <main>
        <h1>Welcome to ZX</h1>

        <!-- Client component placeholder -->
        <div id="zx-a1b2c3d4e5f6"
             data-wasm-component="Counter"
             data-wasm-mount=""
             data-props='{"initial":0}'>
            Loading...
        </div>

        <p>Server-side content continues...</p>
    </main>

    <!-- Hydration script -->
    <script type="module">
        import { WasmRuntime } from '/assets/wasm_runtime.js';

        const runtime = new WasmRuntime();
        await runtime.init('/assets/app.wasm');
        await runtime.hydrate();
    </script>
</body>
</html>
```

### Hydration Process

1. **Component Discovery**
   ```javascript
   async hydrate() {
       const components = document.querySelectorAll('[data-wasm-mount]');

       for (const element of components) {
           const componentName = element.dataset.wasmComponent;
           const componentId = element.id;
           const props = element.dataset.props;

           await this.hydrateComponent(componentId, componentName, props);
       }
   }
   ```

2. **Component Mounting**
   ```javascript
   async hydrateComponent(elementId, componentName, propsJson) {
       // Parse props if provided
       const props = propsJson ? JSON.parse(propsJson) : {};

       // Initialize component in WASM
       if (this.exports.initComponent) {
           const propsStr = JSON.stringify(props);
           const ptr = this.writeString(propsStr);
           this.exports.initComponent(componentName, ptr, propsStr.length);
       }

       // Render component
       const html = this.renderComponent(componentName);

       // Replace placeholder
       const element = document.getElementById(elementId);
       element.innerHTML = html;

       // Wire event handlers
       this.attachEventHandlers(element);
   }
   ```

3. **Event Handler Attachment**
   ```javascript
   attachEventHandlers(root) {
       // Find all elements with event attributes
       const elements = root.querySelectorAll('[onclick], [onchange], [oninput]');

       for (const element of elements) {
           for (const attr of element.attributes) {
               if (attr.name.startsWith('on')) {
                   const eventType = attr.name.substring(2);
                   const handlerName = attr.value.replace('()', '');

                   element.addEventListener(eventType, (e) => {
                       this.handleEvent(handlerName, e);
                   });

                   // Remove inline handler
                   element.removeAttribute(attr.name);
               }
           }
       }
   }
   ```

## Error Handling Strategy

### Build-Time Errors

```zig
pub const TranspileError = error{
    InvalidSyntax,
    MissingComponent,
    CircularDependency,
    UnsupportedFeature,
    ManifestCorrupted,
};

fn handleTranspileError(err: TranspileError, context: ErrorContext) void {
    const message = switch (err) {
        .InvalidSyntax => "Invalid JSX syntax at line {d}, column {d}",
        .MissingComponent => "Component '{s}' not found",
        .CircularDependency => "Circular dependency detected: {s}",
        .UnsupportedFeature => "Feature '{s}' not supported in client components",
        .ManifestCorrupted => "Client manifest corrupted, please rebuild",
    };

    std.debug.print("Error: " ++ message ++ "\n", context.args);
    std.process.exit(1);
}
```

### Runtime Errors

```javascript
class ErrorBoundary {
    constructor(runtime) {
        this.runtime = runtime;
        this.errorHandlers = new Map();
    }

    wrap(componentId, fn) {
        return (...args) => {
            try {
                return fn.apply(this, args);
            } catch (error) {
                this.handleError(componentId, error);
            }
        };
    }

    handleError(componentId, error) {
        console.error(`Component ${componentId} error:`, error);

        // Try to recover
        const element = document.getElementById(componentId);
        if (element) {
            element.innerHTML = `
                <div class="error-boundary">
                    <h3>Component Error</h3>
                    <p>${error.message}</p>
                    <button onclick="location.reload()">Reload Page</button>
                </div>
            `;
        }

        // Report to monitoring
        if (window.errorReporter) {
            window.errorReporter.report(error, { componentId });
        }
    }
}
```

### WASM Panic Handling

```zig
pub fn panic(msg: []const u8, error_return_trace: ?*std.builtin.StackTrace, ret_addr: ?usize) noreturn {
    // Log to console if available
    if (@hasDecl(js, "console_error")) {
        js.console_error(msg.ptr, msg.len);
    }

    // Set error state
    error_state = .{
        .message = msg,
        .trace = error_return_trace,
        .address = ret_addr,
    };

    // Return error code instead of aborting
    @trap();
}

export fn getLastError() ?[]const u8 {
    return if (error_state) |state| state.message else null;
}
```

## Performance Considerations

### Binary Size Optimization

```zig
// Build configuration for minimal size
pub fn buildWasm(b: *std.Build) void {
    const wasm = b.addExecutable(.{
        .name = "app",
        .root_module = .{
            .root_source_file = .{ .path = "main.zig" },
            .target = wasm_target,
            .optimize = .ReleaseSmall,  // Optimize for size
            .strip = true,              // Strip debug symbols
            .single_threaded = true,    // No threading overhead
        },
    });

    // Disable features we don't need
    wasm.disable_stack_probing = true;
    wasm.omit_frame_pointer = true;

    // Link-time optimization
    wasm.want_lto = true;
}
```

### Memory Management

```zig
// Arena allocator for render operations
const RenderArena = struct {
    arena: std.heap.ArenaAllocator,
    high_water_mark: usize = 0,

    pub fn init() RenderArena {
        return .{
            .arena = std.heap.ArenaAllocator.init(std.heap.wasm_allocator),
        };
    }

    pub fn reset(self: *RenderArena) void {
        // Track maximum usage
        const current = self.arena.queryCapacity();
        if (current > self.high_water_mark) {
            self.high_water_mark = current;
        }

        // Reset but retain capacity
        _ = self.arena.reset(.retain_capacity);
    }

    pub fn allocator(self: *RenderArena) std.mem.Allocator {
        return self.arena.allocator();
    }
};
```

### Render Optimization

```javascript
// Batch DOM updates
class RenderScheduler {
    constructor(runtime) {
        this.runtime = runtime;
        this.pendingUpdates = new Set();
        this.frameId = null;
    }

    schedule(componentId) {
        this.pendingUpdates.add(componentId);

        if (!this.frameId) {
            this.frameId = requestAnimationFrame(() => {
                this.flush();
            });
        }
    }

    flush() {
        const updates = Array.from(this.pendingUpdates);
        this.pendingUpdates.clear();
        this.frameId = null;

        // Batch render all components
        performance.mark('render-start');

        for (const componentId of updates) {
            this.runtime.renderComponent(componentId);
        }

        performance.mark('render-end');
        performance.measure('render', 'render-start', 'render-end');
    }
}
```

### Caching Strategy

```javascript
// WASM module caching
class WasmCache {
    static async load(url) {
        const cacheKey = `wasm-${url}`;

        // Try cache first
        if ('caches' in window) {
            const cache = await caches.open('wasm-cache-v1');
            const cached = await cache.match(url);

            if (cached) {
                console.log('Loading WASM from cache');
                return await cached.arrayBuffer();
            }
        }

        // Fetch and cache
        const response = await fetch(url);
        const buffer = await response.arrayBuffer();

        if ('caches' in window) {
            const cache = await caches.open('wasm-cache-v1');
            await cache.put(url, new Response(buffer, {
                headers: {
                    'Content-Type': 'application/wasm',
                    'Cache-Control': 'public, max-age=31536000'
                }
            }));
        }

        return buffer;
    }
}
```

## Security Considerations

### Input Sanitization

```zig
fn sanitizeHtml(allocator: std.mem.Allocator, input: []const u8) ![]const u8 {
    var result = std.ArrayList(u8).init(allocator);

    for (input) |char| {
        switch (char) {
            '<' => try result.appendSlice("&lt;"),
            '>' => try result.appendSlice("&gt;"),
            '&' => try result.appendSlice("&amp;"),
            '"' => try result.appendSlice("&quot;"),
            '\'' => try result.appendSlice("&#x27;"),
            '/' => try result.appendSlice("&#x2F;"),
            else => try result.append(char),
        }
    }

    return result.toOwnedSlice();
}
```

### Content Security Policy

```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self' 'wasm-unsafe-eval';
               style-src 'self' 'unsafe-inline';">
```

### WASM Sandbox

```javascript
// Isolate WASM execution
class SecureWasmRuntime extends WasmRuntime {
    constructor() {
        super();
        this.sandbox = this.createSandbox();
    }

    createSandbox() {
        return {
            // Limit imports
            env: {
                memory: new WebAssembly.Memory({
                    initial: 16,  // 1MB
                    maximum: 256, // 16MB max
                    shared: false
                })
            },

            // No direct DOM access
            js: {
                console_log: (ptr, len) => {
                    const msg = this.readString(ptr, len);
                    console.log('[WASM]', msg);
                },
                // No eval, no dynamic code execution
            }
        };
    }
}
```

## Testing Approach

### Unit Tests

```zig
// Test transpiler
test "generates correct placeholder component" {
    const source = "'use client'\npub fn Component() {}";
    const result = try generatePlaceholder(testing.allocator, source);

    try testing.expect(std.mem.indexOf(u8, result, "data-wasm-component") != null);
    try testing.expect(std.mem.indexOf(u8, result, "Loading...") != null);
}

// Test manifest generation
test "creates valid manifest entry" {
    const component = ComponentMetadata{
        .name = "TestComponent",
        .path = "test.zx",
        .id = "zx-test123",
    };

    const json = try std.json.stringify(component, .{}, testing.allocator);
    try testing.expect(std.mem.indexOf(u8, json, "\"id\":\"zx-test123\"") != null);
}
```

### Integration Tests

```javascript
// Test runtime hydration
describe('WASM Runtime', () => {
    let runtime;

    beforeEach(async () => {
        runtime = new WasmRuntime();
        await runtime.init('./test.wasm');
    });

    test('hydrates components correctly', async () => {
        document.body.innerHTML = `
            <div id="test" data-wasm-component="Test" data-wasm-mount></div>
        `;

        await runtime.hydrate();

        const element = document.getElementById('test');
        expect(element.innerHTML).not.toBe('Loading...');
        expect(element.innerHTML).toContain('Test Component');
    });

    test('handles events', async () => {
        const mockHandler = jest.fn();
        runtime.exports.testHandler = mockHandler;

        await runtime.handleEvent('testHandler', {});

        expect(mockHandler).toHaveBeenCalled();
    });
});
```

## Migration Path

### For Existing zx Applications

1. **Phase 1: Preparation**
   ```bash
   # Update zx to latest version
   zig fetch --save zx@latest

   # Audit existing components
   find site -name "*.zx" -exec grep -l "onclick\|state" {} \;
   ```

2. **Phase 2: Migration**
   ```zig
   // Before: Server-only component
   pub fn Counter(ctx: zx.PageContext) zx.Component {
       // Static counter
   }

   // After: Client-side component
   'use client'

   var count: u32 = 0;

   pub fn Counter(ctx: zx.PageContext) zx.Component {
       // Interactive counter
   }
   ```

3. **Phase 3: Validation**
   ```bash
   # Build and test
   zx transpile site
   zx build-client
   zx test
   ```

## Future Enhancements

### Phase 4: Advanced Features
- Virtual DOM diffing
- Code splitting
- Lazy loading
- Streaming SSR

### Phase 5: Developer Experience
- Hot module replacement
- Chrome DevTools extension
- VS Code integration
- Component playground

### Phase 6: Ecosystem
- Component library
- State management library
- Router package
- Form validation

## Conclusion

This technical design provides a comprehensive blueprint for implementing client-side rendering in zx. The two-pass architecture elegantly separates server and client concerns while reusing existing transpiler infrastructure. The design prioritizes simplicity and performance while maintaining flexibility for future enhancements.

Key strengths of this approach:
- Leverages existing JSX parser and transpiler
- Clean separation between server and client code
- Predictable memory management with arena allocators
- Progressive enhancement friendly
- Clear migration path for existing applications

Success depends on careful implementation of each component, thorough testing, and iterative refinement based on real-world usage.