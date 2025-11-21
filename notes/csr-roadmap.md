# CSR Implementation Roadmap

## Project Goal
Enable client-side rendering in zx using WebAssembly, allowing Zig code to run in the browser and create interactive components.

## Current Status
- ✅ Transpiler detects `'use client'` directive
- ✅ Reference implementation exists (`zx-wasm-renderer`)
- ❌ Files with 'use client' are **skipped**, not transpiled
- ❌ No client build pipeline
- ❌ No WASM compilation step
- ❌ No JS runtime/loader

## Phase 1: Basic Static Rendering

**Goal**: Render a simple component from WASM once, no interactivity.

**Timeline**: 1-2 weeks

### Tasks

#### 1.1 Transpiler Changes (Pass 1: Server Build)
**File**: `src/cli/transpile.zig`

Current behavior (lines 76-82, 630-636):
```zig
if (std.mem.eql(u8, first_line, "'use client'")) {
    log.info("Skipping client-side file: {s}", .{source_path});
    return;  // ← Change this!
}
```

**Changes needed - Pass 1**:
- [ ] Remove the `return` statement
- [ ] Instead, call `handleClientComponent()` function
- [ ] Generate **placeholder component** with:
  - Unique component ID (hash of path + name)
  - Div with `data-wasm-component`, `data-wasm-mount` attributes
  - "Loading..." text
- [ ] Write manifest entry to `.zx/client_components.json`:
  ```json
  {
    "id": "component-xyz",
    "name": "ComponentName",
    "source_path": "pages/component.zx"
  }
  ```
- [ ] Continue to next file (don't skip!)

**New transpiler output**:
```zig
// Generated from component.zx with 'use client' (transpiled JSX)
const std = @import("std");
const zx = @import("zx");

var render_arena = std.heap.ArenaAllocator.init(std.heap.wasm_allocator);
export var render_buffer: [4096]u8 = undefined;

// Transpiled from JSX: <div>Hello from WASM!</div>
pub fn Component(ctx: zx.PageContext) zx.Component {
    var _zx = zx.initWithAllocator(ctx.arena);
    return _zx.zx(
        .div,
        .{ .children = &.{ _zx.txt("Hello from WASM!") } },
    );
}

export fn render() usize {
    _ = render_arena.reset(.retain_capacity);
    const allocator = render_arena.allocator();
    const ctx = zx.PageContext{ .arena = allocator };

    const component = Component(ctx);
    const html = component.toHtml(allocator) catch return 0;

    const len = @min(html.len, render_buffer.len);
    @memcpy(render_buffer[0..len], html[0..len]);
    return len;
}
```

#### 1.2 Client Build System (Pass 2: Client Build)
**New file**: `src/cli/build_client.zig` or new command `zx build-client`

- [ ] Read `.zx/client_components.json` manifest
- [ ] For each component entry:
  - [ ] Re-read original `.zx` source file
  - [ ] Parse and transpile JSX → Zig (same as server, but for WASM)
  - [ ] Generate WASM-compatible code:
    - Use `wasm_allocator` or arena
    - Export `render()` function
    - Export event handlers (increment, etc.)
- [ ] Create client entry point (`.zx/client/main.zig`)
- [ ] Build WASM binary:
  - Target: `wasm32-freestanding`
  - Optimization: `.ReleaseSmall`
  - Export memory: `true`
  - Output: `.zx/assets/app.wasm`

**Build script template**:
```zig
pub fn buildClient(b: *std.Build, client_files: [][]const u8) !void {
    const wasm_target = b.resolveTargetQuery(.{
        .cpu_arch = .wasm32,
        .os_tag = .freestanding,
    });

    const wasm = b.addExecutable(.{
        .name = "app",
        .root_module = b.createModule(.{
            .root_source_file = b.path(".zx/client_main.zig"),
            .target = wasm_target,
            .optimize = .ReleaseSmall,
            .single_threaded = true,
        }),
    });

    wasm.entry = .disabled;
    wasm.export_memory = true;
    wasm.rdynamic = true;

    const install = b.addInstallFileWithDir(
        wasm.getEmittedBin(),
        .{ .custom = ".zx/assets" },
        "app.wasm",
    );

    b.getInstallStep().dependOn(&install.step);
}
```

#### 1.3 Client Main Entry Point
**New file**: `.zx/client_main.zig` (auto-generated)

- [ ] Import all client components
- [ ] Set up shared render buffer
- [ ] Export render function
- [ ] Route to correct component based on page

**Template**:
```zig
const std = @import("std");
const allocator = std.heap.wasm_allocator;

// Auto-imported based on transpiled files
const HomePage = @import("pages/home.zig");
const AboutPage = @import("pages/about.zig");
// ...

export var render_buffer: [4096]u8 = undefined;
var current_component: ComponentType = .home;

const ComponentType = enum {
    home,
    about,
    // ...
};

export fn setComponent(component: u8) void {
    current_component = @enumFromInt(component);
}

export fn render() usize {
    const html = switch (current_component) {
        .home => HomePage.Component.render(),
        .about => AboutPage.Component.render(),
    } catch return 0;

    defer allocator.free(html);

    const len = @min(html.len, render_buffer.len);
    @memcpy(render_buffer[0..len], html[0..len]);
    return len;
}
```

#### 1.4 JavaScript Runtime
**New file**: `.zx/assets/wasm_runtime.js`

- [ ] Fetch and instantiate WASM
- [ ] Read render buffer
- [ ] Update DOM
- [ ] Export initialization function

**Runtime template**:
```javascript
class WasmRuntime {
    constructor() {
        this.instance = null;
        this.memory = null;
        this.exports = null;
    }

    async init(wasmPath) {
        const response = await fetch(wasmPath);
        const buffer = await response.arrayBuffer();
        const result = await WebAssembly.instantiate(buffer, {});

        this.instance = result.instance;
        this.exports = result.instance.exports;
        this.memory = this.exports.memory;

        return this;
    }

    render() {
        const length = this.exports.render();
        const html = this.readString(this.exports.render_buffer.value, length);

        const root = document.getElementById('root');
        if (root) {
            root.innerHTML = html;
        }
    }

    readString(ptr, length) {
        const memory = new Uint8Array(this.memory.buffer);
        const bytes = memory.slice(ptr, ptr + length);
        return new TextDecoder().decode(bytes);
    }
}

// Auto-initialize
window.wasmRuntime = new WasmRuntime();
```

#### 1.5 HTML Integration
**Update**: Generated HTML files

- [ ] Add `<div id="root"></div>` for WASM mounting
- [ ] Include `wasm_runtime.js`
- [ ] Call `init()` and `render()`

**HTML template**:
```html
<!DOCTYPE html>
<html>
<head>
    <title>ZX App</title>
</head>
<body>
    <div id="root">Loading...</div>
    <script type="module">
        import { WasmRuntime } from '/assets/wasm_runtime.js';

        const runtime = await WasmRuntime.init('/assets/app.wasm');
        runtime.render();
    </script>
</body>
</html>
```

#### 1.6 Testing
- [ ] Create test component: `test/client/hello.zx`
- [ ] Transpile and build
- [ ] Serve with development server
- [ ] Verify rendering in browser
- [ ] Check WASM binary size

**Success Criteria**:
- ✓ `'use client'` files transpile successfully
- ✓ WASM binary builds without errors
- ✓ Binary size < 50KB
- ✓ Component renders in browser
- ✓ No console errors
- ✓ HTML content matches expected

---

## Phase 2: Interactive Components

**Goal**: Add state and event handling.

**Timeline**: 2-3 weeks

### Tasks

#### 2.1 Mutable State
- [ ] Add global state variables
- [ ] Export mutation functions
- [ ] Re-render after mutations

**Example**:
```zig
var count: u32 = 0;

export fn increment() void {
    count += 1;
}

export fn decrement() void {
    if (count > 0) count -= 1;
}

export fn render() usize {
    const html = std.fmt.allocPrint(allocator,
        \\<div>
        \\  <h1>Counter: {d}</h1>
        \\  <button onclick="increment()">+</button>
        \\  <button onclick="decrement()">-</button>
        \\</div>
        ,
        .{count}
    ) catch return 0;
    // ... copy to buffer
}
```

#### 2.2 Event Handling
- [ ] Wire up onclick handlers
- [ ] Call WASM functions from JS
- [ ] Re-render after events

**JS Runtime update**:
```javascript
async init(wasmPath) {
    // ... existing code ...

    // Expose WASM functions globally
    window.increment = () => {
        this.exports.increment();
        this.render();
    };

    window.decrement = () => {
        this.exports.decrement();
        this.render();
    };
}
```

#### 2.3 Event Types
- [ ] onclick
- [ ] onchange
- [ ] onsubmit
- [ ] oninput
- [ ] onkeydown/onkeyup

#### 2.4 Component Structure
- [ ] Define component interface
- [ ] Separate state from rendering
- [ ] Component lifecycle

**Pattern**:
```zig
'use client'

const Counter = struct {
    count: u32 = 0,

    pub fn init() Counter {
        return .{};
    }

    pub fn increment(self: *Counter) void {
        self.count += 1;
    }
};

var counter = Counter.init();

pub fn Component(ctx: zx.PageContext) zx.Component {
    return (
        <div>Count: {[counter.count:d]}</div>
    );
}

export fn increment() void {
    counter.increment();
}

export fn render() usize {
    _ = render_arena.reset(.retain_capacity);
    const allocator = render_arena.allocator();
    const ctx = zx.PageContext{ .arena = allocator };
    const component = Component(ctx);
    const html = component.toHtml(allocator) catch return 0;
    const len = @min(html.len, render_buffer.len);
    @memcpy(render_buffer[0..len], html[0..len]);
    return len;
}

const zx = @import("zx");
var render_arena = std.heap.ArenaAllocator.init(std.heap.wasm_allocator);
export var render_buffer: [4096]u8 = undefined;
```

#### 2.5 Testing
- [ ] Counter component
- [ ] Todo list component
- [ ] Form input component
- [ ] Performance testing (event handling speed)

**Success Criteria**:
- ✓ Buttons respond to clicks
- ✓ State updates correctly
- ✓ DOM re-renders after state change
- ✓ No memory leaks
- ✓ Smooth interactions (< 16ms render)

---

## Phase 3: useState Hook

**Goal**: Automatic re-rendering with declarative state.

**Timeline**: 3-4 weeks

### Tasks

#### 3.1 Hook Context
- [ ] Design hook API
- [ ] Track component state
- [ ] State indexing
- [ ] Re-render invalidation

**API Design**:
```zig
const HookContext = struct {
    state_index: usize,
    states: std.ArrayList(StateValue),
    dirty: bool,

    pub fn useState(self: *HookContext, comptime T: type, initial: T) *T {
        if (self.state_index >= self.states.items.len) {
            // First render
            self.states.append(StateValue.init(T, initial)) catch unreachable;
        }

        const state = &self.states.items[self.state_index];
        self.state_index += 1;
        return state.as(T);
    }

    pub fn setState(self: *HookContext, index: usize, comptime T: type, value: T) void {
        self.states.items[index].set(T, value);
        self.dirty = true;
    }
};
```

#### 3.2 Component Registration
- [ ] Assign unique IDs to components
- [ ] Track component instances
- [ ] Map events to components

#### 3.3 Automatic Re-rendering
- [ ] Detect state changes
- [ ] Mark component dirty
- [ ] Batch re-renders
- [ ] Efficient updates (only dirty components)

#### 3.4 Hook Rules
- [ ] Hooks must be called in order
- [ ] Same number of hooks each render
- [ ] Only in component functions
- [ ] Validation/errors for violations

#### 3.5 Multiple State Variables
```zig
const Component = struct {
    pub fn render(ctx: *HookContext) ![]const u8 {
        const count = ctx.useState(u32, 0);
        const name = ctx.useState([]const u8, "World");
        const enabled = ctx.useState(bool, true);

        return try std.fmt.allocPrint(allocator,
            \\<div>
            \\  <p>Count: {d}</p>
            \\  <p>Name: {s}</p>
            \\  <p>Enabled: {}</p>
            \\</div>
            ,
            .{count.*, name.*, enabled.*}
        );
    }
};
```

#### 3.6 JS Bridge
- [ ] `setState(componentId, stateIndex, value)` function
- [ ] Serialize values to WASM
- [ ] Trigger re-render

#### 3.7 Testing
- [ ] Multiple state variables
- [ ] Independent updates
- [ ] Complex component tree
- [ ] Performance benchmarks

**Success Criteria**:
- ✓ useState works like React
- ✓ Automatic re-rendering
- ✓ Multiple independent state
- ✓ No unnecessary re-renders
- ✓ Developer-friendly API

---

## Future Phases (Phase 4+)

### Phase 4: Optimization
- Virtual DOM diffing
- Lazy loading components
- Code splitting
- WASM streaming compilation
- Service worker caching

### Phase 5: Advanced Features
- useEffect hook (side effects)
- useRef hook (DOM refs)
- useContext hook (shared state)
- Custom hooks
- Suspense for async

### Phase 6: Developer Experience
- Hot module reload
- Source maps
- DevTools integration
- Error boundaries
- Logging/debugging

### Phase 7: Server Integration
- SSR (server-side rendering)
- Hydration
- Streaming SSR
- Islands architecture
- Server components

### Phase 8: Production Ready
- Build optimizations
- Bundle size analysis
- Performance monitoring
- Error tracking
- Documentation

---

## Technical Decisions Log

### Decision 1: Use 'use client' Directive
**Date**: 2025-11-20
**Status**: Temporary

**Reasoning**:
- Familiar to Next.js users
- Easy to implement
- Low barrier to entry

**Concerns**:
- Goes against Zig's explicit control flow
- Hidden compilation behavior
- Could be confusing

**Future**: Consider file naming (*.client.zx) or explicit build config

### Decision 2: Single WASM Binary (Phase 1-3)
**Date**: 2025-11-20
**Status**: Accepted

**Reasoning**:
- Simpler to implement
- Fewer HTTP requests
- Easier debugging
- Good for small apps

**Tradeoffs**:
- Large initial download for big apps
- No lazy loading
- Cache invalidation (one binary changes, all reload)

**Future**: Split into multiple WASM files per route/component

### Decision 3: Shared Memory Buffer
**Date**: 2025-11-20
**Status**: Accepted for Phase 1-2

**Reasoning**:
- Proven pattern (zx-wasm-renderer)
- Simple implementation
- Fast for small renders
- Zero-copy

**Tradeoffs**:
- Fixed buffer size
- Won't scale to large renders
- String-based (no structured data)

**Future**: Consider virtual DOM operations or dynamic allocation

### Decision 4: String-Based HTML Rendering
**Date**: 2025-11-20
**Status**: Accepted for Phase 1-2

**Reasoning**:
- Simple to implement
- Easy to debug
- Works with existing tools
- Familiar pattern

**Tradeoffs**:
- Full re-render (no diffing)
- Slower for large DOMs
- No fine-grained updates

**Future**: Add virtual DOM diffing in Phase 4

---

## Dependencies

### External
- Zig compiler (0.13.0+)
- Web browser with WASM support
- HTTP server for development

### Internal
- `zx` transpiler
- `zx` build system
- `zx` runtime library

---

## Risks & Mitigation

### Risk 1: WASM Binary Size
**Impact**: Slow page loads, poor mobile experience
**Probability**: High
**Mitigation**:
- Use `.ReleaseSmall` optimization
- Strip debug symbols
- Compress with brotli
- Monitor size in CI
- Set size budget (< 100KB)

### Risk 2: Performance
**Impact**: Slow interactions, janky UI
**Probability**: Medium
**Mitigation**:
- Profile early and often
- Benchmark against native JS
- Optimize hot paths
- Use virtual DOM diffing
- Batch DOM updates

### Risk 3: Debugging Difficulty
**Impact**: Hard to develop, fix bugs
**Probability**: Medium
**Mitigation**:
- Good error messages
- Source maps
- Logging utilities
- DevTools integration
- Documentation

### Risk 4: Memory Leaks
**Impact**: Browser crashes, poor UX
**Probability**: Medium
**Mitigation**:
- Use arena allocators
- Clear state between renders
- Memory profiling tools
- Automated leak detection tests

### Risk 5: Browser Compatibility
**Impact**: Doesn't work on some browsers
**Probability**: Low
**Mitigation**:
- Test on major browsers
- Polyfills if needed
- Feature detection
- Graceful degradation

---

## Resources

### Documentation
- [ ] Architecture overview
- [ ] Component API reference
- [ ] Hook usage guide
- [ ] Event handling patterns
- [ ] Performance tips
- [ ] Migration guide

### Examples
- [ ] Hello World
- [ ] Counter
- [ ] Todo List
- [ ] Form validation
- [ ] Data fetching
- [ ] Complex app

### Tools
- [ ] CLI for scaffolding
- [ ] Build analyzer
- [ ] Performance profiler
- [ ] Bundle size checker

---

## Success Metrics

### Phase 1
- Binary size: < 50KB
- First render: < 100ms
- Zero errors
- Works in Chrome, Firefox, Safari

### Phase 2
- Event response: < 16ms
- Re-render time: < 16ms
- Memory stable (no leaks)
- 10+ event types supported

### Phase 3
- API matches React hooks 80%
- Developer satisfaction: 8/10
- Performance = Phase 2
- Documentation complete

---

## Notes

- Keep it simple in Phase 1
- Optimize in later phases
- Get feedback early
- Iterate on API design
- Document everything
- Test on real projects
