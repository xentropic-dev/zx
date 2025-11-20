# Transpiler Mechanics: SSR to CSR Coordination

## Overview

The transpiler needs to handle two different compilation paths:
1. **Server-side (SSR)**: Generate placeholder HTML + hydration scripts
2. **Client-side (CSR)**: Generate WASM-compatible Zig code

This document explores how to coordinate between these two worlds.

## The Problem

When a `.zx` file has `'use client'`:

**Server needs to**:
- Render initial HTML (placeholder or SSR content)
- Add a container element with unique ID
- Inject JavaScript to load WASM
- Pass any initial props/state to WASM

**Client (WASM) needs to**:
- Find the container element
- Render into that container
- Take over event handling
- Maintain or merge server-rendered state

## Key Terminology & Concepts

### 1. **Hydration**
The process of attaching client-side behavior to server-rendered HTML.

**Research terms**:
- "React hydration"
- "SSR hydration"
- "hydration mismatch"
- "progressive hydration"
- "selective hydration"

**Types**:
- **Full hydration**: Client re-renders everything and replaces server HTML
- **Partial hydration**: Client attaches to existing HTML without re-render
- **Progressive hydration**: Hydrate components as they become visible
- **Resumable hydration**: Serialize state on server, resume on client (Qwik approach)

### 2. **Islands Architecture**
Only specific components are interactive; rest is static HTML.

**Research terms**:
- "islands architecture"
- "Astro islands"
- "partial hydration"
- "component islands"

**Pattern**:
```html
<div>
  <h1>Static content</h1>
  <!-- Island: interactive component -->
  <div data-island="counter" data-component-id="123">
    <div>Count: 0</div>
  </div>
  <p>More static content</p>
</div>
```

### 3. **Server Components vs Client Components**
Separation between server-only and client-interactive code.

**Research terms**:
- "React server components"
- "Next.js server components"
- "server vs client components"

### 4. **Code Splitting / Lazy Loading**
Load JavaScript/WASM only for components that need it.

**Research terms**:
- "code splitting"
- "lazy loading components"
- "dynamic imports"
- "bundle splitting"

### 5. **Serialization / Dehydration**
Converting server state to JSON for client consumption.

**Research terms**:
- "state serialization"
- "dehydration/rehydration"
- "server-to-client state transfer"

## Transpiler Strategies

### Strategy 1: Full Client-Side Rendering (Simplest)

Server renders an empty div, client does everything.

#### Server-Side Transpilation

**Input** (`component.zx`):
```zig
'use client'

pub const Component = struct {
    pub fn render() ![]const u8 {
        return "<div>Hello from WASM!</div>";
    }
};
```

**Server Output** (`component.zig`):
```zig
// Generated server-side version
const std = @import("std");

pub const Component = struct {
    pub fn render(allocator: std.mem.Allocator) ![]const u8 {
        // Server just renders a placeholder
        return try std.fmt.allocPrint(allocator,
            \\<div id="component-{s}" data-client-component="Component" data-wasm-mount>
            \\  <div class="loading">Loading...</div>
            \\</div>
            ,
            .{generateComponentId()}
        );
    }
};

fn generateComponentId() []const u8 {
    // Generate unique ID for this component instance
    return "abc123"; // In reality: UUID or hash
}
```

**Client Output** (`.zx/client/component.zig`):
```zig
// Generated client-side version
const std = @import("std");

export var render_buffer: [4096]u8 = undefined;

pub const Component = struct {
    pub fn render() ![]const u8 {
        return "<div>Hello from WASM!</div>";
    }
};

export fn render() usize {
    const html = Component.render() catch return 0;
    const len = @min(html.len, render_buffer.len);
    @memcpy(render_buffer[0..len], html[0..len]);
    return len;
}
```

#### HTML Output

```html
<!DOCTYPE html>
<html>
<head>
    <script type="module" src="/assets/wasm_runtime.js"></script>
</head>
<body>
    <!-- Server-rendered placeholder -->
    <div id="component-abc123" data-client-component="Component" data-wasm-mount>
        <div class="loading">Loading...</div>
    </div>

    <!-- Hydration script -->
    <script type="module">
        import { WasmRuntime } from '/assets/wasm_runtime.js';

        const runtime = await WasmRuntime.init('/assets/app.wasm');

        // Find all client component mounts
        document.querySelectorAll('[data-wasm-mount]').forEach(el => {
            const componentName = el.dataset.clientComponent;
            runtime.renderInto(el.id, componentName);
        });
    </script>
</body>
</html>
```

**Pros**:
- ✅ Simple transpiler logic
- ✅ Clear separation
- ✅ Easy to debug

**Cons**:
- ❌ Flash of loading state
- ❌ Bad for SEO (empty content)
- ❌ Slower perceived load

### Strategy 2: SSR with Client Takeover (Better UX)

Server renders full HTML, client hydrates and takes over.

#### Server-Side Transpilation

**Server Output**:
```zig
pub const Component = struct {
    count: u32 = 0,

    pub fn render(self: Component, allocator: std.mem.Allocator) ![]const u8 {
        const component_id = generateComponentId();

        return try std.fmt.allocPrint(allocator,
            \\<div id="{s}" data-client-component="Counter" data-wasm-mount>
            \\  <h1>Counter: {d}</h1>
            \\  <button onclick="increment()">Increment</button>
            \\  <script type="application/json" data-component-props>
            \\    {{"count": {d}}}
            \\  </script>
            \\</div>
            ,
            .{component_id, self.count, self.count}
        );
    }
};
```

**Explanation**:
- Server renders full HTML (not just placeholder)
- Embeds initial state in `<script type="application/json">`
- Client can read props and hydrate with same state

#### Client-Side Transpilation

**Client Output**:
```zig
const std = @import("std");

var count: u32 = 0;

pub const Component = struct {
    pub fn render() ![]const u8 {
        return try std.fmt.allocPrint(allocator,
            \\<div>
            \\  <h1>Counter: {d}</h1>
            \\  <button onclick="increment()">Increment</button>
            \\</div>
            ,
            .{count}
        );
    }
};

// Initialize from server-provided props
export fn init(count_initial: u32) void {
    count = count_initial;
}

export fn increment() void {
    count += 1;
}

export fn render() usize {
    const html = Component.render() catch return 0;
    // ... copy to buffer
}
```

#### JavaScript Runtime Enhancement

```javascript
class WasmRuntime {
    async hydrate(elementId, componentName) {
        const element = document.getElementById(elementId);
        if (!element) return;

        // Read initial props from embedded script
        const propsScript = element.querySelector('[data-component-props]');
        let props = {};
        if (propsScript) {
            props = JSON.parse(propsScript.textContent);
        }

        // Initialize WASM component with props
        if (this.exports.init) {
            // Convert props to WASM types
            if (props.count !== undefined) {
                this.exports.init(props.count);
            }
        }

        // Render (will replace server HTML)
        this.exports.render();
        const html = this.readBuffer();
        element.innerHTML = html;

        // Wire up event handlers
        this.attachEventHandlers(element);
    }
}
```

**Pros**:
- ✅ Good initial render (server HTML visible immediately)
- ✅ SEO friendly
- ✅ No flash of loading
- ✅ State continuity

**Cons**:
- ❌ More complex transpiler
- ❌ Potential hydration mismatches
- ❌ Need serialization logic

### Strategy 3: Islands Architecture (Most Flexible)

Only specific components are interactive islands in a sea of static HTML.

#### Page Structure

```html
<html>
<body>
    <header>
        <h1>My App</h1>
        <!-- Static, no WASM needed -->
    </header>

    <main>
        <p>Some static content...</p>

        <!-- Island 1: Interactive counter -->
        <div id="island-counter" data-island="Counter" data-props='{"count":0}'>
            <div>Count: 0</div>
            <button>Increment</button>
        </div>

        <p>More static content...</p>

        <!-- Island 2: Interactive form -->
        <div id="island-form" data-island="ContactForm">
            <form>...</form>
        </div>
    </main>

    <footer>
        <!-- Static, no WASM needed -->
    </footer>

    <!-- Only load WASM for islands -->
    <script type="module">
        import { WasmRuntime } from '/assets/wasm_runtime.js';

        const runtime = await WasmRuntime.init('/assets/app.wasm');

        // Hydrate each island independently
        document.querySelectorAll('[data-island]').forEach(async (island) => {
            const componentName = island.dataset.island;
            const props = JSON.parse(island.dataset.props || '{}');

            await runtime.hydrateIsland(island.id, componentName, props);
        });
    </script>
</body>
</html>
```

#### Transpiler Island Detection

```zig
// In transpiler: detect if component should be an island

fn shouldBeIsland(source: []const u8) bool {
    // Check for 'use client' directive
    if (std.mem.startsWith(u8, source, "'use client'")) return true;

    // Or detect interactivity (event handlers, state)
    if (std.mem.indexOf(u8, source, "onclick=") != null) return true;
    if (std.mem.indexOf(u8, source, "useState") != null) return true;

    return false;
}

fn transpileAsIsland(source: []const u8, component_name: []const u8) ![]const u8 {
    // Server version wraps in island container
    return try std.fmt.allocPrint(allocator,
        \\pub const {s} = struct {{
        \\    pub fn render(props: Props) ![]const u8 {{
        \\        const island_id = generateId();
        \\        const props_json = try serializeProps(props);
        \\
        \\        return try std.fmt.allocPrint(allocator,
        \\            \\<div id="{{s}}" data-island="{s}" data-props='{{s}}'>
        \\            \\  {{s}}
        \\            \\</div>
        \\            ,
        \\            .{{island_id, props_json, renderInitialContent(props)}}
        \\        );
        \\    }}
        \\}};
        ,
        .{component_name}
    );
}
```

**Pros**:
- ✅ Minimal WASM (only for interactive parts)
- ✅ Fast initial load
- ✅ SEO friendly (mostly static)
- ✅ Progressive enhancement

**Cons**:
- ❌ Complex to implement
- ❌ Need to detect interactivity
- ❌ Communication between islands tricky

## Transpiler Implementation Plan

### Phase 1: Dual Output

Transpiler generates TWO versions of each `'use client'` file:

```
app/pages/counter.zx
  ↓ transpile
  ├─→ .zx/pages/counter.zig       (Server version - placeholder)
  └─→ .zx/client/counter.zig      (Client version - full logic)
```

### Phase 2: Server Version Template

```zig
// Template for server-side output
const std = @import("std");

pub const {COMPONENT_NAME} = struct {
    // Extract any props from original component
    {PROPS_STRUCT}

    pub fn render(self: {COMPONENT_NAME}, allocator: std.mem.Allocator) ![]const u8 {
        const component_id = "{GENERATED_ID}";
        const props_json = try serializeProps(self, allocator);

        return try std.fmt.allocPrint(allocator,
            \\<div id="{{s}}" data-client-component="{COMPONENT_NAME}" data-wasm-mount>
            \\  {INITIAL_HTML_OR_PLACEHOLDER}
            \\  <script type="application/json" data-component-props>{{s}}</script>
            \\</div>
            ,
            .{component_id, props_json}
        );
    }
};

fn serializeProps(props: anytype, allocator: std.mem.Allocator) ![]const u8 {
    // Serialize props to JSON
    return try std.json.stringifyAlloc(allocator, props, .{});
}
```

### Phase 3: Client Version Template

```zig
// Template for client-side output
const std = @import("std");

// Component state (extracted from original)
{STATE_VARIABLES}

// Original component logic (preserved)
{ORIGINAL_COMPONENT_CODE}

// Export functions for WASM
export fn init(props_ptr: [*]const u8, props_len: usize) void {
    // Deserialize props and initialize state
    const props_json = props_ptr[0..props_len];
    // Parse JSON and set state variables
}

export fn render() usize {
    const html = {COMPONENT_NAME}.render() catch return 0;
    // Copy to render_buffer
}

// Export any event handlers
{EXPORTED_EVENT_HANDLERS}
```

### Phase 4: Script Injection

Transpiler adds to HTML:

```zig
fn injectHydrationScript(allocator: std.mem.Allocator, components: []ComponentInfo) ![]const u8 {
    return try std.fmt.allocPrint(allocator,
        \\<script type="module">
        \\  import {{ WasmRuntime }} from '/assets/wasm_runtime.js';
        \\
        \\  async function hydrate() {{
        \\    const runtime = await WasmRuntime.init('/assets/app.wasm');
        \\
        \\    // Hydrate each component
        \\    {HYDRATION_CALLS}
        \\  }}
        \\
        \\  if (document.readyState === 'loading') {{
        \\    document.addEventListener('DOMContentLoaded', hydrate);
        \\  }} else {{
        \\    hydrate();
        \\  }}
        \\</script>
        ,
        .{}
    );
}
```

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Build Time                            │
└─────────────────────────────────────────────────────────────┘

    counter.zx ('use client' at top)
           ↓
    ┌──────────────────┐
    │   Transpiler     │
    └──────────────────┘
           ↓
      ┌────┴────┐
      ↓         ↓
  Server     Client
  Version    Version
      ↓         ↓
  counter.  counter.
  zig       zig
  (SSR)     (WASM)
      ↓         ↓
    Compile   Compile
      ↓         ↓
   Binary    app.wasm

┌─────────────────────────────────────────────────────────────┐
│                        Request Time                          │
└─────────────────────────────────────────────────────────────┘

    User requests page
           ↓
    Server executes (counter.zig SSR version)
           ↓
    Generate HTML with:
      - Placeholder div with ID
      - Initial state in <script>
      - Hydration script tag
           ↓
    Send HTML to browser
           ↓
    Browser renders HTML (visible immediately!)
           ↓
    Browser downloads app.wasm
           ↓
    Hydration script runs:
      1. Init WASM with props from <script>
      2. Call render()
      3. Replace placeholder with WASM output
      4. Attach event handlers
           ↓
    Component now interactive!
```

## Detailed Transpiler Pseudocode

```zig
fn transpileClientComponent(
    source: []const u8,
    output_dir: []const u8,
    component_name: []const u8,
) !void {
    // 1. Parse the source
    var ast = try parseZxSource(source);
    defer ast.deinit();

    // 2. Extract metadata
    const has_use_client = detectUseClient(source);
    if (!has_use_client) return; // Not a client component

    const props = try extractProps(&ast);
    const state_vars = try extractStateVars(&ast);
    const event_handlers = try extractEventHandlers(&ast);

    // 3. Generate server version (SSR)
    const server_code = try generateServerVersion(.{
        .component_name = component_name,
        .props = props,
        .initial_render = try generateInitialRender(&ast),
    });

    const server_path = try std.fs.path.join(allocator, &.{
        output_dir,
        "pages",
        component_name ++ ".zig"
    });
    try writeFile(server_path, server_code);

    // 4. Generate client version (WASM)
    const client_code = try generateClientVersion(.{
        .component_name = component_name,
        .original_ast = &ast,
        .state_vars = state_vars,
        .event_handlers = event_handlers,
    });

    const client_path = try std.fs.path.join(allocator, &.{
        output_dir,
        "client",
        component_name ++ ".zig"
    });
    try writeFile(client_path, client_code);

    // 5. Register component for WASM build
    try registerClientComponent(component_name, client_path);
}

fn generateServerVersion(config: ServerConfig) ![]const u8 {
    return try std.fmt.allocPrint(allocator,
        \\const std = @import("std");
        \\
        \\pub const {s} = struct {{
        \\    {s} // Props struct
        \\
        \\    pub fn render(self: {s}, allocator: std.mem.Allocator) ![]const u8 {{
        \\        const id = generateComponentId();
        \\        const props_json = try std.json.stringifyAlloc(allocator, self, .{{}});
        \\
        \\        return try std.fmt.allocPrint(allocator,
        \\            \\<div id="{{s}}" data-client="{s}" data-wasm-mount>
        \\            \\  {s}
        \\            \\  <script type="application/json" data-props>{{s}}</script>
        \\            \\</div>
        \\            ,
        \\            .{{id, props_json}}
        \\        );
        \\    }}
        \\}};
        \\
        \\fn generateComponentId() []const u8 {{
        \\    // Generate unique ID
        \\    return "component-" ++ std.fmt.comptimePrint("{{}}", .{{std.time.milliTimestamp()}});
        \\}}
        ,
        .{
            config.component_name,
            config.props,
            config.component_name,
            config.component_name,
            config.initial_render,
        }
    );
}

fn generateClientVersion(config: ClientConfig) ![]const u8 {
    var code = std.ArrayList(u8).init(allocator);
    const writer = code.writer();

    // Imports
    try writer.writeAll("const std = @import(\"std\");\n\n");

    // State variables
    try writer.writeAll("// Component state\n");
    for (config.state_vars) |state_var| {
        try writer.print("var {s}: {s} = {s};\n", .{
            state_var.name,
            state_var.type_name,
            state_var.initial_value,
        });
    }

    // Original component code
    try writer.writeAll("\n// Component logic\n");
    try writer.writeAll(config.original_ast.toZig());

    // WASM exports
    try writer.writeAll("\n// WASM interface\n");
    try writer.writeAll(
        \\export var render_buffer: [8192]u8 = undefined;
        \\
        \\export fn init(props_ptr: [*]const u8, props_len: usize) void {
        \\    const props_json = props_ptr[0..props_len];
        \\    // TODO: Parse JSON and initialize state
        \\}
        \\
        \\export fn render() usize {
        \\    const html = Component.render() catch return 0;
        \\    const len = @min(html.len, render_buffer.len);
        \\    @memcpy(render_buffer[0..len], html[0..len]);
        \\    return len;
        \\}
        \\
    );

    // Export event handlers
    for (config.event_handlers) |handler| {
        try writer.print(
            \\export fn {s}() void {{
            \\    {s}
            \\}}
            \\
            ,
            .{handler.name, handler.body}
        );
    }

    return code.toOwnedSlice();
}
```

## Research Resources

### Essential Reading

1. **React Hydration**
   - https://react.dev/reference/react-dom/client/hydrateRoot
   - Understanding hydration mismatches
   - Progressive hydration patterns

2. **Islands Architecture**
   - https://jasonformat.com/islands-architecture/
   - Astro documentation: https://docs.astro.build/en/concepts/islands/
   - Partial hydration benefits

3. **Next.js App Router**
   - https://nextjs.org/docs/app/building-your-application/rendering/server-components
   - Server vs Client Components
   - `'use client'` directive

4. **Qwik Resumability**
   - https://qwik.builder.io/docs/concepts/resumable/
   - Alternative to hydration
   - Serializing execution context

5. **Leptos (Rust + WASM)**
   - https://github.com/leptos-rs/leptos
   - Most similar to what we're building
   - SSR + hydration patterns

6. **WASM Bindgen**
   - https://rustwasm.github.io/wasm-bindgen/
   - JS ↔ WASM communication
   - Memory sharing patterns

### Specific Techniques

**Hydration Mismatch Prevention**:
```javascript
// Suppress hydration warnings during development
if (process.env.NODE_ENV === 'development') {
    console.warn('Hydration mismatch detected:', element);
}
```

**Lazy Component Loading**:
```javascript
// Load WASM only when component enters viewport
const observer = new IntersectionObserver(async (entries) => {
    for (const entry of entries) {
        if (entry.isIntersecting) {
            await hydrateComponent(entry.target);
            observer.unobserve(entry.target);
        }
    }
});

document.querySelectorAll('[data-lazy-hydrate]').forEach(el => {
    observer.observe(el);
});
```

**State Serialization**:
```zig
pub fn serializeState(state: anytype, allocator: std.mem.Allocator) ![]const u8 {
    return try std.json.stringifyAlloc(allocator, state, .{
        .whitespace = .minified,
        .emit_null_optional_fields = false,
    });
}
```

## Open Questions

1. **How to handle component composition?**
   - Parent server component, child client component?
   - Nested client components?
   - Props passing?

2. **How to detect what needs to be interactive?**
   - Explicit `'use client'` only?
   - Analyze code for event handlers?
   - Type system annotations?

3. **How to handle shared code?**
   - Some functions can't run in WASM (file I/O)
   - Need conditional compilation?
   - Separate modules?

4. **Performance: when to hydrate?**
   - Immediately on page load?
   - When component enters viewport?
   - On first interaction (hover/click)?

5. **How to minimize hydration mismatches?**
   - Hash server output and compare?
   - Accept mismatches and re-render?
   - Strict matching?

## Next Steps

1. [ ] Implement dual transpiler output (server + client versions)
2. [ ] Create component ID generation system
3. [ ] Build props serialization/deserialization
4. [ ] Implement hydration script injection
5. [ ] Test with simple counter component
6. [ ] Handle hydration mismatches gracefully
7. [ ] Add error boundaries
8. [ ] Performance profiling

See `quick-start.md` for immediate implementation, and `csr-roadmap.md` for full timeline.
