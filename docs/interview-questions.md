# React Repository Interview Questions

The questions below highlight implementation details that are essential for working on this codebase and that distinguish React from other UI libraries.

## 1. How does the Fiber architecture coordinate rendering work across roots?

- Why it matters: Understanding Fiber's incremental work loop is foundational for debugging updates and performance tuning.
- Core concepts to cover: Fiber nodes as a linked tree, begin/complete phases, effect list commits, double buffering across `current` and `workInProgress`.
- Source pointers: `packages/react-reconciler/src/ReactFiber.js`, `packages/react-reconciler/src/ReactFiberWorkLoop.js`.

### Answer

React represents each rendered element with a `FiberNode` that links to its parent (`return`), first child (`child`), and siblings, letting the reconciler traverse and schedule work like a linked list of work units (`packages/react-reconciler/src/ReactFiber.js:140`). When React prepares an update it reuses a “work-in-progress” tree through double buffering—each `Fiber` has an `alternate` pointer to its other version. `createWorkInProgress` either allocates that twin lazily or resets it for reuse, ensuring only two copies of the tree exist even across many renders:

```js
// packages/react-reconciler/src/ReactFiber.js:325
export function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {
  let workInProgress = current.alternate;
  if (workInProgress === null) {
    workInProgress = createFiber(current.tag, pendingProps, current.key, current.mode);
    workInProgress.alternate = current;
    current.alternate = workInProgress;
  } else {
    workInProgress.pendingProps = pendingProps;
    workInProgress.flags = NoFlags;
    workInProgress.subtreeFlags = NoFlags;
  }
  workInProgress.flags = current.flags & StaticMask;
  return workInProgress;
}
```

Rendering proceeds unit by unit. `performUnitOfWork` calls `beginWork` to reconcile children, pushes the next child onto the work stack, and when a branch finishes it unwinds through `completeUnitOfWork` to build the effect list that the commit phase later flushes (`packages/react-reconciler/src/ReactFiberWorkLoop.js:2990`). During `beginWork` React may yield between units when the scheduler says there’s higher priority work, which is how multiple roots and lane priorities co-exist without starving urgent updates:

```js
// packages/react-reconciler/src/ReactFiberWorkLoop.js:2990
function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate;
  let next = beginWork(current, unitOfWork, entangledRenderLanes);
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

Once the render phase finishes building the effect list, the commit phase runs synchronously against the host environment (DOM, native) to apply mutations and run lifecycle effects. Because each root has its own fiber tree and pending lanes, multiple roots can be rendered independently in the same event loop while still sharing scheduling heuristics.

## 2. What problem do lane-based priorities solve and how do transitions fit into the scheduling model?

- Why it matters: React's concurrency story is driven by granular priorities that allow urgent updates to preempt rendering work.
- Core concepts to cover: Lane bitmasks, update queues tagging lanes, transition vs sync lanes, relationship to `scheduler` package callbacks.
- Source pointers: `packages/react-reconciler/src/ReactFiberLane.js`, `packages/scheduler/src/Scheduler.js`, `packages/react/src/ReactStartTransition.js`.

### Answer

Lanes encode update priority as individual bits on the fiber tree so React can cheaply merge, mask, and compare priorities when scheduling work. Urgent updates (like input) occupy low-order bits, while transitions and retries live in higher ranges; combining lane masks lets React know which pending update should run next. The `getHighestPriorityLane` helper isolates the least significant bit set—effectively picking the most urgent work—without allocating extra data structures (`packages/react-reconciler/src/ReactFiberLane.js:751`):

```js
// packages/react-reconciler/src/ReactFiberLane.js:751
export function getHighestPriorityLane(lanes: Lanes): Lane {
  return lanes & -lanes;
}
```

As updates flow through `markRootUpdated` and the render work loop, React continuously intersects and merges lanes to figure out which roots to render and which work to defer. Transitions, started via `startTransition`, push a new `Transition` object onto `ReactSharedInternals.T`, allowing React to tag all state updates during the scope with the appropriate transition lane (`packages/react/src/ReactStartTransition.js:32`). When the scope completes, React can keep the transition alive if it returns a promise and mark participating fibers for tracing in dev builds:

```js
// packages/react/src/ReactStartTransition.js:45
export function startTransition(scope: () => void, options?: StartTransitionOptions): void {
  const prevTransition = ReactSharedInternals.T;
  const currentTransition: Transition = ({}: any);
  ReactSharedInternals.T = currentTransition;
  try {
    const returnValue = scope();
    const onStartTransitionFinish = ReactSharedInternals.S;
    if (onStartTransitionFinish !== null) {
      onStartTransitionFinish(currentTransition, returnValue);
    }
  } finally {
    ReactSharedInternals.T = prevTransition;
  }
}
```

During scheduling, lanes determine which updates can interrupt others—discrete lanes can preempt rendering transitions, while transition lanes cooperate with the `scheduler` package to yield control via `shouldYield`, enabling smooth rendering even under heavy workloads.

## 3. How does Suspense coordinate rendering, data fetching, and hydration across clients and servers?

- Why it matters: Suspense is the backbone for streaming and selective hydration that differentiates React's concurrent rendering model.
- Core concepts to cover: Boundary placement, fallback trees, deferred hydration in `hydrateRoot`, retry lanes, interactions with `ReactCache`.
- Source pointers: `packages/react-reconciler/src/ReactFiberSuspenseComponent.js`, `packages/react-dom-bindings/src/client/ReactDOMHydration.js`, `packages/react/src/ReactCache.js`.

### Answer

Every Suspense boundary stores a `SuspenseState` describing whether it is dehydrated from the server or currently showing a fallback. Dehydrated boundaries keep a reference to the server-rendered `SuspenseInstance`, a retry lane, and any hydration errors so the reconciler can schedule retries when data arrives (`packages/react-reconciler/src/ReactFiberSuspenseComponent.js:31`):

```js
// packages/react-reconciler/src/ReactFiberSuspenseComponent.js:31
export type SuspenseState = {
  dehydrated: null | SuspenseInstance,
  treeContext: null | TreeContext,
  retryLane: Lane,
  hydrationErrors: Array<CapturedValue<mixed>> | null,
};
```

On the client, `hydrateRoot` constructs a hydration-aware root that tracks callbacks, identifiers, and transition listeners before wiring up event replay and hydration heuristics (`packages/react-dom/src/client/ReactDOMRoot.js:274`). Hydration reuses the server DOM where possible, deferring to the boundary’s retry lane when it needs to suspend:

```js
// packages/react-dom/src/client/ReactDOMRoot.js:274
export function hydrateRoot(container, initialChildren, options?): RootType {
  const hydrationCallbacks = options != null ? options : null;
  const root = createHydrationContainer(
    initialChildren,
    null,
    container,
    ConcurrentRoot,
    hydrationCallbacks,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
    identifierPrefix,
    onUncaughtError,
    onCaughtError,
    onRecoverableError,
    onDefaultTransitionIndicator,
    transitionCallbacks,
    formState,
  );
  markContainerAsRoot(root.current, container);
  listenToAllSupportedEvents(container);
  return new ReactDOMHydrationRoot(root);
}
```

Suspense relies on `ReactCache` to memoize async resources. Components wrapped in cached data readers will throw promises when data is pending; the cache machinery records the promise and retries once it resolves, so boundaries resume without redundant fetches (`packages/react/src/ReactCacheImpl.js:43`). Together, the dehydrated state, hydration root coordination, and cache-backed data fetching allow the same Suspense primitives to power server streaming, client hydration, and concurrent data loading without bespoke APIs per environment.

## 4. What is the React Server Components pipeline (Flight) and how does it integrate with different bundlers?

- Why it matters: Server-driven component trees and selective client bundling are a key differentiator in React's architecture today.
- Core concepts to cover: Server rendering to a serialized payload, module references, client resumability, bundler-specific bindings (Webpack, ESM, Turbopack).
- Source pointers: `packages/react-server/src/ReactServerStreamConfig.js`, `packages/react-server-dom-webpack`, `packages/react-server-dom-esm`, `packages/react-client`.

### Answer

React Server Components render to a Flight stream on the server, emitting JSON-like chunks that describe component payloads and module references. The runtime expects the renderer host config (`packages/react-server/src/ReactServerStreamConfig.js`) to be shimmed per environment so that the same reconciler logic can encode JSX elements into the transport format.

On the build side, bundler plugins (e.g. the Webpack plugin) discover “client references”—modules that must ship to the browser because they contain interactivity. The plugin walks configured directories, registers async dependencies, and writes manifests that map server module IDs to client chunks (`packages/react-server-dom-webpack/src/ReactFlightWebpackPlugin.js:66`):

```js
// packages/react-server-dom-webpack/src/ReactFlightWebpackPlugin.js:66
export default class ReactFlightWebpackPlugin {
  clientReferences: $ReadOnlyArray<ClientReferencePath>;
  constructor(options: Options) {
    if (!options || typeof options.isServer !== 'boolean') {
      throw new Error(PLUGIN_NAME + ': You must specify the isServer option as a boolean.');
    }
    this.clientReferences = Array.isArray(options.clientReferences)
      ? options.clientReferences
      : [{directory: '.', recursive: true, include: /\.(js|ts|jsx|tsx)$/}];
    this.clientManifestFilename = options.clientManifestFilename || 'react-client-manifest.json';
  }
  apply(compiler: any) {
    compiler.hooks.beforeCompile.tapAsync(PLUGIN_NAME, ({contextModuleFactory}, callback) => {
      // scan and mark client references...
    });
  }
}
```

Client runtimes such as `react-client` know how to consume the manifest, look up client references during streaming, and lazily hydrate only the modules referenced by the server payload. Different bundler integrations (ESM, Webpack, Turbopack) provide their own loaders and manifests but emit the same protocol, letting the unified React Flight bridge hand off server-rendered component trees to the client without duplicate bundles.

## 5. Which guarantees does the React Compiler provide and how does it enforce the Rules of React?

- Why it matters: The compiler removes the need for manual memoization and catches hook misuse at build time, impacting code authoring patterns.
- Core concepts to cover: Babel entrypoint, High-level IR (HIR), reactive scopes, generated memoization wrappers, ESLint integration for rule violations.
- Source pointers: `compiler/packages/babel-plugin-react-compiler`, `compiler/docs/DESIGN_GOALS.md`, `compiler/packages/eslint-plugin-react-compiler`.

### Answer

React Compiler’s goal is to make idiomatic components fast by default: it analyzes each component or hook, isolates “reactive scopes” (the stateful subgraphs that must re-run on specific state changes), and emits memoized wrappers automatically. The design doc emphasizes predictable startup cost, removing the need for manual `useMemo`/`useCallback`, and rejecting code that violates the Rules of React (`compiler/docs/DESIGN_GOALS.md:7`).

The Babel plugin entry point inspects every program, parses compiler options, and invokes `compileProgram`, which performs lowering to the high-level intermediate representation (HIR), validation, SSA conversion, scope construction, and codegen before swapping the AST in place (`compiler/packages/babel-plugin-react-compiler/src/Babel/BabelPlugin.ts:24`):

```ts
// compiler/packages/babel-plugin-react-compiler/src/Babel/BabelPlugin.ts:24
export default function BabelPluginReactCompiler(_babel: typeof BabelCore): BabelCore.PluginObj {
  return {
    name: 'react-forget',
    visitor: {
      Program: {
        enter(prog, pass): void {
          const opts = parsePluginOptions(pass.opts);
          const result = compileProgram(prog, {
            opts,
            filename: pass.filename ?? null,
            comments: pass.file.ast.comments ?? [],
            code: pass.file.code,
          });
          validateNoUntransformedReferences(prog, pass.filename ?? null, opts.logger, opts.environment, result);
        },
      },
    },
  };
}
```

If the compiler detects violations—like calling hooks conditionally—it throws `CompilerError` diagnostics that surface via Babel, ESLint, or the dedicated lint plugin. When compilation succeeds, the emitted code wraps each reactive scope in stable memo functions so only inputs that actually change trigger recomputation, satisfying the guarantees outlined in the design goals without requiring the developer to annotate components manually.

## 6. How do custom renderers leverage `react-reconciler` and what does a Host Config need to implement?

- Why it matters: Extending React beyond the DOM requires understanding the renderer contract and lifecycle methods.
- Core concepts to cover: Host config methods (creation, mutation, persistence), mutation vs persistent mode, update queues, effect tagging.
- Source pointers: `packages/react-reconciler/README.md`, `packages/react-reconciler/src/ReactFiberHostConfig.js`, `packages/react-art/src/ReactFiberConfigART.js`.

### Answer

`react-reconciler` exposes a generic reconciler that consumes a “host config”: an object describing how to create instances, apply updates, track contexts, and commit effects for a specific platform. When you create a renderer (`Reconciler(hostConfig)`), React calls back into your host config for every lifecycle step, so you decide whether updates mutate existing nodes or replace entire subtrees (persistent mode).

The React ART renderer shows a concrete implementation. It wires up event subscriptions, translates React props into ART drawing commands, and implements host operations such as `applyNodeProps` to mutate the underlying drawing primitives (`packages/react-art/src/ReactFiberConfigART.js:70`):

```js
// packages/react-art/src/ReactFiberConfigART.js:70
function applyNodeProps(instance, props, prevProps = {}) {
  const scaleX = getScaleX(props);
  const scaleY = getScaleY(props);

  pooledTransform
    .transformTo(1, 0, 0, 1, 0, 0)
    .move(props.x || 0, props.y || 0)
    .rotate(props.rotation || 0, props.originX, props.originY)
    .scale(scaleX, scaleY, props.originX, props.originY);

  if (props.transform != null) {
    pooledTransform.transform(props.transform);
  }

  if (instance.xx !== pooledTransform.xx || instance.yx !== pooledTransform.yx) {
    instance.transformTo(pooledTransform);
  }
}
```

Beyond mutation hooks, the host config exports scheduler integration (`getCurrentUpdatePriority`), hydration support, and optional features (scopes, resources). Choosing between mutation and persistent mode happens by setting `supportsMutation` or `supportsPersistence`. Once the host config is in place, you call `MyRenderer.updateContainer(element, container)` just like React DOM does; the reconciler handles diffing, effect tagging, and scheduling while delegating all platform-specific work to your implementations.

## 7. In what way do shared feature flags and internals coordinate experimental builds across packages?

- Why it matters: Feature gating controls runtime behavior, build targets, and experimental channels without duplicating logic per package.
- Core concepts to cover: `shared/ReactFeatureFlags.js` forks, build-time selection via `__DEV__` and environment variables, shared internals surface.
- Source pointers: `packages/shared/ReactFeatureFlags.js`, `packages/shared/forks`, `packages/shared/ReactSharedInternals.js`.

### Answer

React centralizes configuration in `shared/ReactFeatureFlags.js`, exporting booleans that each package can tree-shake or fork. Build scripts swap in different versions of this file (e.g. experimental, www, OSS) so the same source toggles features per release channel. The file groups flags by lifecycle (killswitches, ongoing experiments) and exports explicit booleans the reconciler consults before enabling new behavior (`packages/shared/ReactFeatureFlags.js:25`):

```js
// packages/shared/ReactFeatureFlags.js:25
export const enableHydrationLaneScheduling: boolean = true;
export const enableSuspenseCallback: boolean = false;
export const enableYieldingBeforePassive: boolean = false;
export const enableGestureTransition = __EXPERIMENTAL__;
export const enableDefaultTransitionIndicator = __EXPERIMENTAL__;
```

Shared internals supplement the flags. `ReactSharedInternals` provides runtime pointers (`ReactSharedInternals.H` for the current hooks dispatcher, `.T` for transitions, `.S` for transition callbacks) that host packages and tooling share without exposing them publicly (`packages/shared/ReactSharedInternals.js:12`). By swapping the feature-flag module during builds and keeping internals behind this gateway, all renderers and developer tooling see the same configuration knobs, preventing divergent behavior between React DOM, React Native, the compiler, and devtools.

## 8. How does React DOM bridge the renderer host config to the browser environment?

- Why it matters: DOM-specific bindings layer tree mutations, events, hydration, and resources atop the generic reconciler.
- Core concepts to cover: `react-dom-bindings` separation from public `react-dom`, event system (`EventPluginRegistry`), hydration entrypoints, resource scheduling.
- Source pointers: `packages/react-dom/src/client/ReactDOM.js`, `packages/react-dom-bindings/src/client/ReactDOMHostConfig.js`, `packages/react-dom-bindings/src/events`.

### Answer

The published `react-dom` entry points (`ReactDOMClient`, `ReactDOMRoot`) are thin veneers that expose `createRoot`/`hydrateRoot` while delegating to the shared reconciler. Before exporting, they inject renderer metadata into devtools so inspection and profiling work in the browser (`packages/react-dom/src/client/ReactDOMClient.js:8`):

```js
// packages/react-dom/src/client/ReactDOMClient.js:8
import {createRoot, hydrateRoot} from './ReactDOMRoot';
import {injectIntoDevTools, findHostInstance} from 'react-reconciler/src/ReactFiberReconciler';
import Internals from 'shared/ReactDOMSharedInternals';
Internals.findDOMNode = findHostInstance;
export {ReactVersion as version, createRoot, hydrateRoot};
injectIntoDevTools();
```

All DOM-specific knowledge lives in `react-dom-bindings`. The host config (`ReactFiberConfigDOM`) implements how to create elements, set properties, hydrate nodes, and manage resources such as preloading scripts. The event system wires into the reconciler by translating DOM events into prioritized React events; it sets the correct update priority before dispatching so that input handlers run in the discrete lane while continuous events use a lower priority (`packages/react-dom-bindings/src/events/ReactDOMEventListener.js:60`):

```js
// packages/react-dom-bindings/src/events/ReactDOMEventListener.js:60
function dispatchDiscreteEvent(domEventName, eventSystemFlags, container, nativeEvent) {
  const prevTransition = ReactSharedInternals.T;
  ReactSharedInternals.T = null;
  const previousPriority = getCurrentUpdatePriority();
  try {
    setCurrentUpdatePriority(DiscreteEventPriority);
    dispatchEvent(domEventName, eventSystemFlags, container, nativeEvent);
  } finally {
    setCurrentUpdatePriority(previousPriority);
    ReactSharedInternals.T = prevTransition;
  }
}
```

By splitting public APIs from bindings, React DOM can share reconciler logic with other renderers while still handling browser-specific concerns like hydration event replay, namespace-aware element creation, Trusted Types integration, and default transition indicators.

## 9. What instrumentation hooks power React DevTools and timeline profiling?

- Why it matters: Instrumentation exposes Fiber lifecycle data to debugging tools and influences performance analysis workflows.
- Core concepts to cover: DevTools hook injection, timeline labels from lanes, profiling flags, bridge between runtime and devtools UI.
- Source pointers: `packages/react-devtools-shared/src/hooks`, `packages/react-reconciler/src/ReactFiberDevToolsHook.js`, `packages/react-devtools-timeline`.

### Answer

React detects the global DevTools hook (`__REACT_DEVTOOLS_GLOBAL_HOOK__`) and injects metadata so DevTools can subscribe to fiber lifecycle events. The reconciler module exposes `injectInternals`, `onScheduleRoot`, and `onCommitRoot`; the host renderer calls these as roots are scheduled and committed. The injection guards against missing or outdated DevTools and handles profiling flags when enabled (`packages/react-reconciler/src/ReactFiberDevToolsHook.js:50`):

```js
// packages/react-reconciler/src/ReactFiberDevToolsHook.js:50
export function injectInternals(internals: Object): boolean {
  if (typeof __REACT_DEVTOOLS_GLOBAL_HOOK__ === 'undefined') {
    return false;
  }
  const hook = __REACT_DEVTOOLS_GLOBAL_HOOK__;
  if (!hook.supportsFiber) {
    if (__DEV__) {
      console.error('The installed version of React DevTools is too old...');
    }
    return true;
  }
  rendererID = hook.inject(internals);
  injectedHook = hook;
  return hook.checkDCE;
}
```

When a commit completes, React reports whether the tree threw, which lanes were rendered, and profiled durations. DevTools uses this stream to populate the timeline (mapping lanes to labels, frame durations, suspense suspensions) and to power features like component render reason inspection. Additional instrumentation files in `packages/react-devtools-timeline` consume the same data to render Flamechart and React Profiler views, keeping diagnostic tooling in sync with the runtime.

## 10. How does `react-refresh` support instant feedback without losing component state?

- Why it matters: Fast Refresh is integral to the developer experience and requires cooperation between the runtime and bundlers.
- Core concepts to cover: Module registration, signature comparison, boundary invalidation, interaction with the reconciler during refresh.
- Source pointers: `packages/react-refresh/src/ReactFreshRuntime.js`, `packages/react-refresh/src/ReactFreshBabelPlugin.js`, `packages/shared/ReactRefreshRuntime.js`.

### Answer

React Refresh tracks “component families”—groups of exports that belong to the same logical component across hot updates. The Babel plugin injects registration calls per module, feeding the runtime maps keyed by family IDs. When a module updates, the runtime compares component signatures (including nested hooks) to decide whether it can preserve state or must remount. The runtime maintains weak maps of updated families and mounted roots so it can re-render only the affected boundaries (`packages/react-refresh/src/ReactFreshRuntime.js:44`):

```js
// packages/react-refresh/src/ReactFreshRuntime.js:44
const allFamiliesByID: Map<string, Family> = new Map();
const allFamiliesByType: WeakMap<any, Family> = new PossiblyWeakMap();
const allSignaturesByType: WeakMap<any, Signature> = new PossiblyWeakMap();
const updatedFamiliesByType: WeakMap<any, Family> = new PossiblyWeakMap();
let pendingUpdates: Array<[Family, any]> = [];
const helpersByRendererID: Map<number, RendererHelpers> = new Map();
const mountedRoots: Set<FiberRoot> = new Set();
const failedRoots: Set<FiberRoot> = new Set();
```

When bundlers trigger `performReactRefresh`, the runtime walks `pendingUpdates`, notifies each renderer helper (React DOM, React Native), and the reconciler rerenders affected roots while attempting to reuse state unless hook order or signatures changed. If a component fails to reconcile, it falls back to remounting only that subtree. This tight integration between injected module metadata, the runtime bookkeeping, and reconciler helpers gives developers instant feedback while preserving component state across edits whenever it’s safe.
