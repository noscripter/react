# React Repository Interview Questions

The questions below highlight implementation details that are essential for working on this codebase and that distinguish React from other UI libraries.

## 1. How does the Fiber architecture coordinate rendering work across roots?

- Why it matters: Understanding Fiber's incremental work loop is foundational for debugging updates and performance tuning.
- Core concepts to cover: Fiber nodes as a linked tree, begin/complete phases, effect list commits, double buffering across `current` and `workInProgress`.
- Source pointers: `packages/react-reconciler/src/ReactFiber.js`, `packages/react-reconciler/src/ReactFiberWorkLoop.js`.

## 2. What problem do lane-based priorities solve and how do transitions fit into the scheduling model?

- Why it matters: React's concurrency story is driven by granular priorities that allow urgent updates to preempt rendering work.
- Core concepts to cover: Lane bitmasks, update queues tagging lanes, transition vs sync lanes, relationship to `scheduler` package callbacks.
- Source pointers: `packages/react-reconciler/src/ReactFiberLane.js`, `packages/scheduler/src/Scheduler.js`, `packages/react/src/ReactStartTransition.js`.

## 3. How does Suspense coordinate rendering, data fetching, and hydration across clients and servers?

- Why it matters: Suspense is the backbone for streaming and selective hydration that differentiates React's concurrent rendering model.
- Core concepts to cover: Boundary placement, fallback trees, deferred hydration in `hydrateRoot`, retry lanes, interactions with `ReactCache`.
- Source pointers: `packages/react-reconciler/src/ReactFiberSuspenseComponent.js`, `packages/react-dom-bindings/src/client/ReactDOMHydration.js`, `packages/react/src/ReactCache.js`.

## 4. What is the React Server Components pipeline (Flight) and how does it integrate with different bundlers?

- Why it matters: Server-driven component trees and selective client bundling are a key differentiator in React's architecture today.
- Core concepts to cover: Server rendering to a serialized payload, module references, client resumability, bundler-specific bindings (Webpack, ESM, Turbopack).
- Source pointers: `packages/react-server/src/ReactServerStreamConfig.js`, `packages/react-server-dom-webpack`, `packages/react-server-dom-esm`, `packages/react-client`.

## 5. Which guarantees does the React Compiler provide and how does it enforce the Rules of React?

- Why it matters: The compiler removes the need for manual memoization and catches hook misuse at build time, impacting code authoring patterns.
- Core concepts to cover: Babel entrypoint, High-level IR (HIR), reactive scopes, generated memoization wrappers, ESLint integration for rule violations.
- Source pointers: `compiler/packages/babel-plugin-react-compiler`, `compiler/docs/DESIGN_GOALS.md`, `compiler/packages/eslint-plugin-react-compiler`.

## 6. How do custom renderers leverage `react-reconciler` and what does a Host Config need to implement?

- Why it matters: Extending React beyond the DOM requires understanding the renderer contract and lifecycle methods.
- Core concepts to cover: Host config methods (creation, mutation, persistence), mutation vs persistent mode, update queues, effect tagging.
- Source pointers: `packages/react-reconciler/README.md`, `packages/react-reconciler/src/ReactFiberHostConfig.js`, `packages/react-art/src/ReactFiberConfigART.js`.

## 7. In what way do shared feature flags and internals coordinate experimental builds across packages?

- Why it matters: Feature gating controls runtime behavior, build targets, and experimental channels without duplicating logic per package.
- Core concepts to cover: `shared/ReactFeatureFlags.js` forks, build-time selection via `__DEV__` and environment variables, shared internals surface.
- Source pointers: `packages/shared/ReactFeatureFlags.js`, `packages/shared/forks`, `packages/shared/ReactSharedInternals.js`.

## 8. How does React DOM bridge the renderer host config to the browser environment?

- Why it matters: DOM-specific bindings layer tree mutations, events, hydration, and resources atop the generic reconciler.
- Core concepts to cover: `react-dom-bindings` separation from public `react-dom`, event system (`EventPluginRegistry`), hydration entrypoints, resource scheduling.
- Source pointers: `packages/react-dom/src/client/ReactDOM.js`, `packages/react-dom-bindings/src/client/ReactDOMHostConfig.js`, `packages/react-dom-bindings/src/events`.

## 9. What instrumentation hooks power React DevTools and timeline profiling?

- Why it matters: Instrumentation exposes Fiber lifecycle data to debugging tools and influences performance analysis workflows.
- Core concepts to cover: DevTools hook injection, timeline labels from lanes, profiling flags, bridge between runtime and devtools UI.
- Source pointers: `packages/react-devtools-shared/src/hooks`, `packages/react-reconciler/src/ReactFiberDevToolsHook.js`, `packages/react-devtools-timeline`.

## 10. How does `react-refresh` support instant feedback without losing component state?

- Why it matters: Fast Refresh is integral to the developer experience and requires cooperation between the runtime and bundlers.
- Core concepts to cover: Module registration, signature comparison, boundary invalidation, interaction with the reconciler during refresh.
- Source pointers: `packages/react-refresh/src/ReactFreshRuntime.js`, `packages/react-refresh/src/ReactFreshBabelPlugin.js`, `packages/shared/ReactRefreshRuntime.js`.
