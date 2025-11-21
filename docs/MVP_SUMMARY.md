# ZX Client-Side Rendering MVP - Executive Summary

## What We're Building

We're implementing client-side rendering (CSR) for the zx web framework, enabling Zig components to run in the browser via WebAssembly. This allows developers to write interactive web applications entirely in Zig, using familiar JSX syntax that compiles to efficient WASM binaries. The system uses a two-pass transpilation architecture where 'use client' components are first converted to server-side placeholders, then separately compiled to WebAssembly modules that hydrate the placeholders at runtime.

## Why This Matters

### Problems Solved

1. **Language Fragmentation**: Currently, web developers must use JavaScript/TypeScript for client interactions even when building server applications in Zig. This creates context switching, duplicate logic, and maintenance burden.

2. **Type Safety Gap**: Moving between Zig's compile-time safety and JavaScript's runtime dynamism introduces bugs and reduces confidence in code correctness.

3. **Performance Limitations**: JavaScript's interpreted nature and garbage collection can't match the performance potential of compiled Zig/WASM for compute-intensive operations.

4. **Tooling Complexity**: Managing separate build pipelines, type definitions, and deployment artifacts for server (Zig) and client (JS) code adds unnecessary complexity.

### Value Proposition

- **Unified Language**: Write entire web applications in Zig, from database queries to DOM manipulation
- **Compile-Time Safety**: Catch errors at build time, not in production
- **Near-Native Performance**: WASM execution approaches native speed for client-side logic
- **Familiar Patterns**: JSX syntax and React-like component model lower the learning curve
- **Progressive Enhancement**: Start with server rendering, add interactivity incrementally

## How It Works: Two-Pass Architecture

### Pass 1: Server Transpilation (Current Process)
When the transpiler encounters a `'use client'` directive:
1. **Generate Placeholder Component**: Creates a server-side Zig component that renders a div with hydration markers
2. **Register in Manifest**: Adds component metadata to `.zx/client_components.json`
3. **Continue Transpilation**: Processes remaining files normally

**Current Issue**: The transpiler currently skips client files entirely (lines 79-81, 633-635 in `/home/xentropy/src/zx/src/cli/transpile.zig`). This needs modification.

### Pass 2: Client Build (New Process)
A new build command reads the manifest and:
1. **Re-transpile for WASM**: Processes each client component's JSX for the `wasm32-freestanding` target
2. **Generate Entry Point**: Creates `.zx/client/main.zig` with all client components
3. **Compile to WASM**: Produces `.zx/assets/app.wasm` binary
4. **Create Runtime**: Generates JavaScript hydration code

### Runtime Hydration
At page load:
1. Browser renders server HTML immediately (good SEO, fast perceived load)
2. JavaScript runtime loads WASM binary
3. Runtime finds placeholder divs via data attributes
4. WASM components mount and take over interactivity

## Key Design Decisions

### 1. 'use client' Directive (Temporary)
- **Choice**: Adopt Next.js convention for familiarity
- **Trade-off**: Goes against Zig's explicit control flow philosophy
- **Future**: Consider file naming (`.client.zx`) or build config approach

### 2. Two-Pass Transpilation
- **Choice**: Separate server and client transpilation phases
- **Rationale**: Clean separation of concerns, reuses existing JSX parser
- **Alternative Considered**: Single-pass with conditional compilation (too complex)

### 3. Shared Memory Buffer
- **Choice**: Fixed-size buffer for HTML exchange between WASM and JS
- **Rationale**: Simple, proven pattern from `zx-wasm-renderer` reference
- **Limitation**: 8KB default size (configurable)

### 4. Single WASM Binary (Phase 1-3)
- **Choice**: Bundle all client components into one `app.wasm`
- **Rationale**: Simpler build, fewer HTTP requests, easier debugging
- **Future**: Code splitting per route/component for larger apps

## MVP Scope

### IN Scope (Phase 1 - 2 weeks)
- ✅ Modify transpiler to generate placeholder components
- ✅ Client component manifest system (`.zx/client_components.json`)
- ✅ Pass 2 transpilation for WASM target
- ✅ Basic JavaScript runtime for hydration
- ✅ Shared memory buffer for HTML exchange
- ✅ Simple test components (hello world, counter)
- ✅ Development server integration

### IN Scope (Phase 2 - 3 weeks)
- ✅ Mutable component state
- ✅ Event handling (onclick, onchange, etc.)
- ✅ Re-rendering after state changes
- ✅ Multiple component instances
- ✅ Props passing from server to client

### OUT of Scope (Future Phases)
- ❌ Virtual DOM diffing (full re-renders for now)
- ❌ useState/useEffect hooks (Phase 3)
- ❌ Code splitting/lazy loading
- ❌ Server-side rendering hydration
- ❌ Hot module reload
- ❌ Production optimizations
- ❌ TypeScript definitions
- ❌ Component libraries

## Success Metrics

### Technical Metrics
- **WASM Binary Size**: < 50KB for hello world, < 100KB with basic framework
- **Initial Render Time**: < 100ms from WASM load to DOM update
- **Re-render Performance**: < 16ms for state changes (60 FPS)
- **Memory Usage**: < 1MB for typical single-page app
- **Browser Support**: Chrome, Firefox, Safari (latest 2 versions)

### Developer Experience Metrics
- **Build Time**: < 1 second for small apps
- **Learning Curve**: Developers familiar with React productive in < 1 day
- **Error Messages**: Clear, actionable transpilation and runtime errors
- **Documentation Coverage**: 100% of public APIs documented

### Project Health Metrics
- **Test Coverage**: > 80% for transpiler, > 90% for runtime
- **Zero-Day Bugs**: < 5 critical issues in first week after release
- **Community Adoption**: 10+ example projects within first month

## Risk Mitigation

### High Risk: WASM Binary Size
- **Mitigation**: Use `.ReleaseSmall` optimization, strip symbols, monitor in CI
- **Fallback**: Implement code splitting earlier if size exceeds 150KB

### Medium Risk: Hydration Mismatches
- **Mitigation**: Start with full client-side rendering, add SSR hydration later
- **Fallback**: Clear documentation on limitations, error recovery

### Medium Risk: Browser Compatibility
- **Mitigation**: Test on all major browsers in CI, use feature detection
- **Fallback**: Provide progressive enhancement guidance

## Next Steps

1. **Immediate** (This Week):
   - Modify transpiler to generate placeholders instead of skipping
   - Create client build command structure
   - Set up WASM compilation pipeline

2. **Short Term** (Next 2 Weeks):
   - Implement JavaScript runtime
   - Create test components
   - Write developer documentation

3. **Medium Term** (Next Month):
   - Add event handling system
   - Implement state management
   - Create example applications

## Team & Resources

### Required Expertise
- **Zig Systems Programming**: Transpiler modifications, WASM compilation
- **JavaScript/TypeScript**: Runtime implementation, DOM manipulation
- **Web Platform**: Browser APIs, performance optimization
- **Technical Writing**: Documentation, tutorials

### Development Timeline
- **Phase 1**: 2 weeks (Basic rendering)
- **Phase 2**: 3 weeks (Interactivity)
- **Phase 3**: 4 weeks (Hooks system)
- **Total MVP**: 9 weeks to production-ready

### Dependencies
- Zig 0.13.0+ compiler
- Modern browsers with WASM support
- Existing zx framework infrastructure

## Conclusion

This MVP delivers a pragmatic path to client-side rendering in zx, balancing implementation simplicity with real-world usability. The two-pass architecture leverages existing transpiler infrastructure while maintaining clean separation between server and client concerns. By starting with basic rendering and progressively adding features, we can validate the approach early and iterate based on user feedback.

The design prioritizes developer experience through familiar patterns (JSX, 'use client') while staying true to Zig's performance and safety goals. Success will be measured not just by technical metrics but by community adoption and developer satisfaction.