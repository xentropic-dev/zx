# CSR Implementation Alternatives & Architecture Brainstorm

## Rendering Communication Patterns

### 1. Shared Memory Buffer (Simplest)
**Status**: Implemented in `zx-wasm-renderer`

```
┌─────────────┐         ┌──────────────┐
│   WASM      │         │  JavaScript  │
│             │         │              │
│  render() ──┼────────▶│              │
│  writes to  │  shared │  reads from  │
│  buffer[]   │  memory │  buffer[]    │
│             │◀────────┼  updates DOM │
└─────────────┘         └──────────────┘
```

**Protocol**:
```zig
export var buffer: [N]u8 = undefined;

export fn render() usize {
    // Write HTML to buffer, return length
}
```

```js
const len = render();
const html = readString(memory, buffer.value, len);
element.innerHTML = html;
```

**Variants**:
- **Fixed size**: Fast, simple, limited
- **With length prefix**: Better for variable content
- **Ring buffer**: Multiple renders without blocking
- **Null-terminated**: C-style strings

### 2. Dynamic Allocation + Pointers

```
┌─────────────┐         ┌──────────────┐
│   WASM      │         │  JavaScript  │
│             │         │              │
│  render() ──┼────────▶│  ptr, len    │
│  alloc()    │  return │  read(ptr,   │
│  returns    │  ptr +  │       len)   │
│  ptr, len   │  len    │  free(ptr)   │
│  ◀──────────┼─────────┼              │
│  free()     │         │              │
└─────────────┘         └──────────────┘
```

**Protocol**:
```zig
export fn render(len_out: *usize) [*]const u8 {
    const html = allocPrint(...);
    len_out.* = html.len;
    return html.ptr;
}

export fn free(ptr: [*]u8, len: usize) void {
    allocator.free(ptr[0..len]);
}
```

```js
const lenPtr = new Uint32Array(memory.buffer, lenAddr, 1);
const ptr = render(lenAddr);
const len = lenPtr[0];
const html = readString(memory, ptr, len);
free(ptr, len);
```

**Pros**: Unlimited size
**Cons**: Manual memory management, easy to leak

### 3. Message Queue

```
┌─────────────┐         ┌──────────────┐
│   WASM      │         │  JavaScript  │
│             │         │              │
│  render() ──┼────┐    │              │
│  pushMsg()  │    │    │  pollMsg()   │
│  ┌────────┐ │    ▼    │  ┌─────────┐ │
│  │ Queue  │ │◀────────┼──│  Queue  │ │
│  └────────┘ │  shared │  └─────────┘ │
│             │  buffer │              │
└─────────────┘         └──────────────┘
```

**Protocol**:
```zig
const Message = struct {
    type: MessageType,
    len: u32,
    data: [MAX_SIZE]u8,
};

export var message_queue: [16]Message = undefined;
var queue_head: u32 = 0;
var queue_tail: u32 = 0;

export fn pushMessage(msg: Message) void { ... }
export fn popMessage() ?Message { ... }
```

**Pros**:
- Batching multiple updates
- Async-friendly
- Can queue events both ways

**Cons**:
- More complex
- Fixed queue size
- Need synchronization

### 4. Virtual DOM Operations

Instead of HTML strings, send DOM operations:

```
┌─────────────┐         ┌──────────────┐
│   WASM      │         │  JavaScript  │
│             │         │              │
│  render()   │  ops    │  applyOps()  │
│  diff()  ───┼────────▶│  element.    │
│  encode()   │  buffer │  appendChild │
│             │         │  element.    │
│             │         │  textContent │
└─────────────┘         └──────────────┘
```

**Operations**:
```zig
const OpCode = enum(u8) {
    create_element,    // tag_name
    create_text,       // text
    set_attribute,     // key, value
    set_text,          // text
    append_child,      // parent_id, child_id
    remove_child,      // parent_id, child_id
    replace_child,     // parent_id, old_id, new_id
};

const Operation = struct {
    code: OpCode,
    node_id: u32,
    data: [64]u8,  // variable data
};
```

**Example encoding**:
```
CREATE_ELEMENT, id=1, tag="div"
CREATE_TEXT, id=2, text="Hello"
SET_ATTR, id=1, key="class", value="container"
APPEND_CHILD, parent=1, child=2
```

**Pros**:
- Minimal DOM updates
- More efficient
- Structured data

**Cons**:
- Complex diffing algorithm
- Larger WASM binary
- More debugging complexity

### 5. JSON-RPC Style

```
┌─────────────┐         ┌──────────────┐
│   WASM      │         │  JavaScript  │
│             │  JSON   │              │
│  render() ──┼────────▶│  JSON.parse()│
│  toJSON()   │  string │  interpret() │
│             │         │  execute()   │
└─────────────┘         └──────────────┘
```

**Protocol**:
```json
{
  "type": "render",
  "component": "Counter",
  "tree": {
    "tag": "div",
    "attrs": { "class": "counter" },
    "children": [
      { "tag": "text", "content": "Count: 5" },
      { "tag": "button", "attrs": { "onclick": "increment" } }
    ]
  }
}
```

**Pros**:
- Human-readable
- Easy debugging
- Flexible structure

**Cons**:
- Parsing overhead
- Verbose (large payload)
- Need JSON serializer in Zig

### 6. Direct DOM Manipulation via JS Imports

WASM calls JS functions directly:

```
┌─────────────┐         ┌──────────────┐
│   WASM      │         │  JavaScript  │
│             │  calls  │              │
│  render()   │────────▶│  createElement
│  dom.create │────────▶│  setAttribute
│  dom.append │────────▶│  appendChild
│             │         │              │
└─────────────┘         └──────────────┘
```

**Protocol**:
```zig
extern fn js_createElement(tag_ptr: [*]const u8, tag_len: usize) u32;
extern fn js_setAttribute(node: u32, key_ptr: [*]const u8, key_len: usize,
                          val_ptr: [*]const u8, val_len: usize) void;
extern fn js_appendChild(parent: u32, child: u32) void;

pub fn render() void {
    const div = js_createElement("div", 3);
    js_setAttribute(div, "class", 5, "container", 9);
    // ...
}
```

```js
const imports = {
    env: {
        js_createElement: (tagPtr, tagLen) => {
            const tag = readString(memory, tagPtr, tagLen);
            const el = document.createElement(tag);
            return registerElement(el);
        },
        js_setAttribute: (node, keyPtr, keyLen, valPtr, valLen) => {
            const el = getElement(node);
            const key = readString(memory, keyPtr, keyLen);
            const val = readString(memory, valPtr, valLen);
            el.setAttribute(key, val);
        },
        // ...
    }
};
```

**Pros**:
- Direct control
- No intermediate format
- Full DOM API access

**Cons**:
- Many WASM ↔ JS calls (slow!)
- Complex API surface
- Hard to optimize

## State Management Approaches

### 1. Global Mutable State (Phase 2)

**Simplest approach**:
```zig
var count: u32 = 0;
var username: []const u8 = "";
var items: ArrayList(Item) = undefined;

export fn increment() void {
    count += 1;
    render();
}

export fn setUsername(ptr: [*]const u8, len: usize) void {
    username = ptr[0..len];
    render();
}
```

**Pros**: Simple, fast
**Cons**: No encapsulation, hard to track changes, couples state to render

### 2. Component-Local State

```zig
const Counter = struct {
    count: u32 = 0,

    pub fn increment(self: *Counter) void {
        self.count += 1;
        self.render();
    }

    pub fn render(self: Counter) []const u8 {
        return std.fmt.allocPrint(allocator,
            "<div>Count: {d}</div>",
            .{self.count}
        );
    }
};
```

**Challenge**: How to identify component instances from JS events?

### 3. React-Style Hooks

```zig
const HookContext = struct {
    state_index: usize = 0,
    states: ArrayList(StateValue),
    component_id: u32,

    pub fn useState(self: *HookContext, T: type, init: T) *T {
        if (self.state_index >= self.states.items.len) {
            // First render - create state
            try self.states.append(StateValue{ .value = init });
        }
        const state = &self.states.items[self.state_index];
        self.state_index += 1;
        return &state.value;
    }
};

const Counter = struct {
    pub fn Component(ctx: *HookContext) ![]const u8 {
        const count = ctx.useState(u32, 0);

        return try std.fmt.allocPrint(allocator,
            \\<div>
            \\  <span>{d}</span>
            \\  <button onclick="setState({d}, 0, {d})">+</button>
            \\</div>
            ,
            .{count.*, ctx.component_id, count.* + 1}
        );
    }
};
```

**JS side**:
```js
window.setState = (componentId, stateIndex, newValue) => {
    updateComponentState(componentId, stateIndex, newValue);
    rerenderComponent(componentId);
};
```

**Pros**: Declarative, React-like, multiple states
**Cons**: Complex, needs state tracking, order-dependent

### 4. Signals/Observables

```zig
const Signal = struct {
    value: T,
    subscribers: ArrayList(fn() void),

    pub fn set(self: *Signal, new_value: T) void {
        if (self.value != new_value) {
            self.value = new_value;
            for (self.subscribers.items) |callback| {
                callback();
            }
        }
    }

    pub fn get(self: *Signal) T {
        return self.value;
    }
};

var count = Signal(u32).init(0);

pub fn increment() void {
    count.set(count.get() + 1);
    // Auto re-renders subscribers
}
```

**Pros**: Fine-grained reactivity, efficient updates
**Cons**: More complex, need subscription management

### 5. Store/Reducer Pattern (Redux-like)

```zig
const Action = union(enum) {
    increment,
    decrement,
    set_value: u32,
};

const State = struct {
    count: u32 = 0,
};

fn reducer(state: State, action: Action) State {
    return switch (action) {
        .increment => .{ .count = state.count + 1 },
        .decrement => .{ .count = state.count - 1 },
        .set_value => |v| .{ .count = v },
    };
}

var store: Store(State, Action) = undefined;

export fn dispatch(action_type: u8, payload: u32) void {
    const action: Action = decodeAction(action_type, payload);
    store.dispatch(action);
}
```

**Pros**: Predictable, time-travel debugging, centralized
**Cons**: Boilerplate, overkill for simple apps

## Event Handling Patterns

### 1. String-Based onclick (Current)

```zig
const html =
    \\<button onclick="increment()">Click</button>
;
```

```js
window.increment = () => {
    wasmExports.increment();
    rerender();
};
```

**Pros**: Simple, works with innerHTML
**Cons**: Global namespace pollution, string-based

### 2. Event Delegation

```zig
const html =
    \\<button data-action="increment">Click</button>
;
```

```js
document.addEventListener('click', (e) => {
    const action = e.target.dataset.action;
    if (action) {
        wasmExports.handleEvent(action);
        rerender();
    }
});
```

**Pros**: One event listener, no globals
**Cons**: Need data attributes, limited to clicks

### 3. Event IDs with Payload

```zig
var event_handlers = ArrayList(EventHandler).init(allocator);

pub fn registerHandler(handler: EventHandler) u32 {
    const id = event_handlers.items.len;
    event_handlers.append(handler);
    return id;
}

const html = std.fmt.allocPrint(allocator,
    \\<button data-handler="{d}">Click</button>
    ,
    .{registerHandler(onIncrement)}
);
```

```js
document.addEventListener('click', (e) => {
    const handlerId = e.target.dataset.handler;
    if (handlerId) {
        wasmExports.callHandler(parseInt(handlerId));
        rerender();
    }
});
```

**Pros**: Type-safe, supports closures
**Cons**: Need handler registry, cleanup needed

### 4. Synthetic Events

```zig
export fn handleSyntheticEvent(
    event_type: EventType,
    target_id: u32,
    payload_ptr: [*]const u8,
    payload_len: usize
) void {
    const payload = payload_ptr[0..payload_len];
    // Deserialize and handle event
}
```

```js
element.addEventListener('click', (e) => {
    const payload = JSON.stringify({
        clientX: e.clientX,
        clientY: e.clientY,
        // ...
    });
    const bytes = encoder.encode(payload);
    // Write to WASM memory
    wasmExports.handleSyntheticEvent(EVENT_CLICK, elementId, ptr, len);
});
```

**Pros**: Full event data, type-safe
**Cons**: Serialization overhead, complex

## Build Pipeline Architectures

### Option A: Separate WASM Project

```
zx/
├── src/           # Server code
├── client/        # Client WASM project
│   ├── build.zig
│   └── src/
└── build.zig      # Main server build
```

**Pros**: Clean separation, independent builds
**Cons**: Two build files, harder to share code

### Option B: Unified Build with Multiple Targets

```zig
// build.zig
const server = b.addExecutable(.{
    .name = "zx-server",
    .target = native,
});

const client = b.addExecutable(.{
    .name = "zx-client",
    .target = wasm_target,
});

// Shared module
const shared = b.createModule(.{
    .root_source_file = "src/shared.zig",
});
server.root_module.addImport("shared", shared);
client.root_module.addImport("shared", shared);
```

**Pros**: One build file, easy code sharing
**Cons**: Conditional compilation needed, complexity

### Option C: Build Script Generator

Transpiler generates client build.zig:

```
zx transpile app/
→ .zx/pages/home.zig
→ .zx/pages/about.zig
→ .zx/build.zig  (auto-generated)
```

Auto-generated build.zig:
```zig
// Auto-generated - do not edit
const client = b.addExecutable(.{
    .name = "app",
    .root_module = b.createModule(.{
        .root_source_file = b.path("main.zig"),
        .target = wasm_target,
        .imports = &.{
            .{ .name = "pages", .module = pages_module },
        },
    }),
});
```

**Pros**: Flexible, adapts to project structure
**Cons**: Build complexity, generated code

## Hydration Strategies

### 1. No Server Rendering (CSR Only)

```html
<div id="root"></div>
<script>
    // WASM renders everything
    initWasm().then(render);
</script>
```

**Pros**: Simple, no hydration needed
**Cons**: Blank screen while loading, bad SEO

### 2. Server Renders, Client Takes Over

```html
<div id="root">
    <!-- Server-rendered HTML -->
    <div>Count: 0</div>
</div>
<script>
    // WASM attaches to existing DOM
    initWasm().then(hydrate);
</script>
```

**Challenge**: How to match server HTML with WASM render?

### 3. Islands Architecture

Only hydrate interactive components:

```html
<div>
    <h1>Static content</h1>
    <div data-island="counter">
        <!-- Only this is hydrated -->
    </div>
    <p>More static content</p>
</div>
```

**Pros**: Fast, minimal JS, best of both worlds
**Cons**: Need to identify islands, more complex

### 4. Streaming SSR + Progressive Hydration

```
1. Server starts rendering
2. Send HTML chunks as ready
3. Client hydrates top-down
4. Interactive as soon as hydrated
```

**Pros**: Fast first paint, progressive enhancement
**Cons**: Very complex, need streaming protocol

## Performance Optimizations

### WASM Binary Size
- ✓ Use `.ReleaseSmall` optimization
- ✓ Strip debug symbols
- ⬜ Link-time optimization (LTO)
- ⬜ Remove unused code (treeshaking)
- ⬜ Compress (brotli/gzip)
- ⬜ Split into modules (dynamic import)

### Runtime Performance
- ⬜ Virtual DOM diffing (only update changed)
- ⬜ Batch DOM updates
- ⬜ requestAnimationFrame for renders
- ⬜ Web Workers for heavy computation
- ⬜ Memoization of render results
- ⬜ Lazy component loading

### Memory Management
- ⬜ Object pooling (reuse allocations)
- ⬜ Arena allocators per render
- ⬜ Generational GC (if needed)
- ⬜ Memory profiling tools

### Network
- ⬜ Cache WASM binary aggressively
- ⬜ HTTP/2 server push
- ⬜ Service worker for offline
- ⬜ CDN distribution

## Tooling & DX

### Development
- ⬜ Hot module reload
- ⬜ Source maps for WASM
- ⬜ Browser DevTools integration
- ⬜ Component inspector
- ⬜ State debugger

### Testing
- ⬜ Unit tests (zig test)
- ⬜ Component tests (jsdom)
- ⬜ E2E tests (Playwright)
- ⬜ Visual regression tests

### Debugging
- ⬜ Error boundaries
- ⬜ Logging/tracing
- ⬜ Performance profiling
- ⬜ Memory leak detection

## Comparison: Other Frameworks

### React
- Virtual DOM diffing
- Hooks for state
- Component lifecycle
- JSX syntax

**What we can learn**: Hooks API, lifecycle

### Svelte
- Compile-time reactivity
- No virtual DOM
- Minimal runtime

**What we can learn**: Compile-time optimizations

### Solid.js
- Fine-grained reactivity
- Signals
- Fast updates

**What we can learn**: Signals pattern

### HTMX
- Hypermedia-driven
- Server-side logic
- Minimal JS

**What we can learn**: Progressive enhancement

### Leptos (Rust + WASM)
- Similar to our approach!
- Server functions
- Hydration

**What we can learn**: WASM patterns, API design
