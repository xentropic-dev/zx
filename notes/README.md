# ZX Development Notes

This directory contains design documents, brainstorming notes, and implementation plans for the zx web framework.

## Contents

### [quick-start.md](./quick-start.md)
**START HERE!** Practical guide to implementing Phase 1 with step-by-step instructions.

**Topics covered**:
- Prerequisites and setup
- Creating test component
- Modifying transpiler
- Building WASM binary
- JavaScript runtime setup
- HTML integration
- Testing and debugging
- Common issues and solutions
- Phase 2 preview (adding interactivity)

### [transpiler-mechanics.md](./transpiler-mechanics.md)
**CRITICAL!** Deep dive into how the transpiler coordinates SSR and CSR.

**Topics covered**:
- The SSR ↔ CSR coordination problem (server placeholders + client hydration)
- Key terminology: hydration, islands architecture, server components, serialization
- 3 transpiler strategies (full CSR, SSR with takeover, islands)
- Dual output approach (server version generates placeholder, client version has full logic)
- Script injection and props passing via JSON
- Detailed pseudocode for transpiler implementation
- Data flow diagrams from build to hydration
- Essential research resources (React hydration, Next.js, Leptos, Qwik)
- Research terms: "hydration", "islands architecture", "progressive hydration", "SSR to CSR coordination"

### [client-side-rendering.md](./client-side-rendering.md)
Comprehensive overview of implementing client-side rendering (CSR) in zx using WebAssembly.

**Topics covered**:
- Current architecture analysis
- 'use client' directive design
- Build pipeline design (referencing `zx-wasm-renderer`)
- Rendering approaches (shared memory, dynamic allocation, virtual DOM, etc.)
- Implementation phases (1: static render, 2: stateful, 3: hooks)
- Design considerations and tradeoffs
- Open questions and future work

### [csr-alternatives.md](./csr-alternatives.md)
Deep dive into alternative approaches and architectural patterns for CSR implementation.

**Topics covered**:
- 6+ rendering communication patterns (shared memory, pointers, message queue, virtual DOM, etc.)
- 5+ state management approaches (global, component-local, hooks, signals, store/reducer)
- 4+ event handling patterns (string onclick, delegation, event IDs, synthetic events)
- 3+ build pipeline architectures
- 4+ hydration strategies (CSR only, SSR+takeover, islands, streaming)
- Performance optimizations
- Tooling & DX considerations
- Comparisons with React, Svelte, Solid.js, HTMX, Leptos

### [csr-roadmap.md](./csr-roadmap.md)
Detailed implementation roadmap with concrete tasks and timelines.

**Topics covered**:
- Phase 1 tasks: Basic static rendering (1-2 weeks)
  - Transpiler changes
  - Client build system
  - JavaScript runtime
  - HTML integration
- Phase 2 tasks: Interactive components (2-3 weeks)
  - Mutable state
  - Event handling
  - Component structure
- Phase 3 tasks: useState hook (3-4 weeks)
  - Hook context
  - Component registration
  - Automatic re-rendering
- Future phases: Optimization, advanced features, DX, SSR, production
- Technical decisions log
- Risks & mitigation strategies
- Success metrics

### [memory-management.md](./memory-management.md)
Deep dive into memory management patterns for WASM client components.

**Topics covered**:
- WASM memory model and common pitfalls
- 5+ allocator strategies (global, arena, fixed buffer, pool, GPA)
- Recommended hybrid approach (arena for renders, WASM allocator for state)
- Memory sharing patterns (fixed buffer, growable, multiple buffers, ring buffer)
- Common patterns (string building, component tree, object pool)
- Debugging techniques (leak detection, allocation tracking)
- Best practices and anti-patterns
- Memory budgets and performance targets

## Quick Reference

### Current Status
- ✅ Transpiler has 'use client' detection
- ✅ Reference WASM renderer exists (`../zx-wasm-renderer`)
- ❌ Client files are currently **skipped**, not transpiled
- ❌ No WASM build pipeline yet
- ❌ No JS runtime yet

### Key Files in Main Project
- `src/cli/transpile.zig` - Main transpiler (lines 76-82, 630-636 handle 'use client')
- `build.zig` - Main build configuration

### Reference Project
- `../zx-wasm-renderer/` - Working WASM renderer example
  - `build.zig` - WASM build configuration
  - `src/main.zig` - WASM exports (render, buffer)
  - `app/src/app.ts` - JS runtime (instantiate, read memory, update DOM)
  - `app/src/index.html` - HTML integration

### Next Immediate Steps
1. Modify `src/cli/transpile.zig` to transpile (not skip) 'use client' files
2. Create client build system (new file: `src/cli/transpile/client_build.zig`)
3. Generate client main entry point (`.zx/client_main.zig`)
4. Create JavaScript runtime (`.zx/assets/wasm_runtime.js`)
5. Test with simple hello world component

## Philosophy & Design Principles

### Explicit Over Implicit
Zig values explicit control flow. Using `'use client'` goes against this - it's a magic string that changes compilation behavior. This is acceptable as a **temporary** solution for familiarity and simplicity, but should be reconsidered later (file naming conventions, explicit build config, etc.).

### Start Simple, Optimize Later
- Phase 1: Single WASM binary, shared memory buffer, full re-renders
- Phase 2+: Code splitting, virtual DOM, fine-grained updates

### Performance Budget
- WASM binary: < 100KB (50KB ideal)
- First render: < 100ms
- Event response: < 16ms (60fps)
- Re-render: < 16ms

### Developer Experience Matters
- Clear error messages
- Good documentation
- Familiar patterns (React-like where sensible)
- Easy debugging
- Fast iteration

## Architectural Decisions

| Decision | Status | Reasoning | Alternatives |
|----------|--------|-----------|--------------|
| Use 'use client' directive | **Temporary** | Familiar, simple | File naming, build config |
| Single WASM binary | **Phase 1-3** | Simple, fewer requests | Multiple per route/component |
| Shared memory buffer | **Phase 1-2** | Proven, fast, simple | Dynamic alloc, virtual DOM |
| String-based HTML | **Phase 1-2** | Simple, debuggable | Virtual DOM operations |
| ReleaseSmall optimization | **Permanent** | Minimize binary size | ReleaseSpeed, ReleaseFast |

## Resources

### Zig WASM
- [Zig WebAssembly docs](https://ziglang.org/documentation/master/#WebAssembly)
- [WASM allocator](https://ziglang.org/documentation/master/std/#A;std:heap.wasm_allocator)

### WebAssembly
- [MDN WebAssembly](https://developer.mozilla.org/en-US/docs/WebAssembly)
- [WebAssembly spec](https://webassembly.github.io/spec/)

### Similar Projects
- [Leptos](https://github.com/leptos-rs/leptos) - Rust + WASM framework (most similar!)
- [Yew](https://yew.rs/) - Rust + WASM (React-like)
- [Percy](https://github.com/chinedufn/percy) - Rust virtual DOM

### Inspiration
- [React](https://react.dev/) - Hooks, component model
- [Solid.js](https://www.solidjs.com/) - Fine-grained reactivity
- [Svelte](https://svelte.dev/) - Compile-time reactivity
- [HTMX](https://htmx.org/) - Hypermedia approach

## Questions & Discussions

Have ideas or questions? Document them here or in the relevant note file.

### Open Questions
1. How to handle routing in CSR?
2. How to share code between server and client?
3. What's the hydration strategy?
4. How to handle assets (CSS, images)?
5. Hot reload strategy?
6. Error handling across WASM boundary?
7. Testing approach?
8. Build time optimization?

See [client-side-rendering.md](./client-side-rendering.md) for detailed discussion of these questions.

## Contributing

When adding new notes:
1. Create a new `.md` file with a descriptive name
2. Update this README with a link and description
3. Use clear headings and structure
4. Include code examples where helpful
5. Link to relevant files in the codebase

## Status Legend

- ✅ Done / Working
- ⬜ Todo / Not started
- 🚧 In progress
- ❌ Not working / Blocked
- ⚠️ Needs attention / Decision needed
