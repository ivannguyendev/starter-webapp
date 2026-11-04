# Performance, Memory Leaks & Production Reliability

## 1. Optimization loop (no guessing)

1. **Measure** on a production build, CPU throttled 4×: Web Vitals (RUM p75 if available), React Profiler, bundle analyzer.
2. **Identify** the single largest cost. 3. **Fix** one thing. 4. **Verify** under identical conditions.
5. **Neutral is a revert**: gain within noise, or tests break ⇒ revert; complexity without evidence is debt.
6. **Guard**: bundle budget in CI (`size-limit` or bundler budget), Lighthouse CI assertions, RUM vitals labelled by route *pattern* (never raw URL).

## 2. Waterfalls (highest-impact fix)

| Waterfall | Fix |
|---|---|
| Parent fetches, then child mounts and fetches | Both depend only on route params → prefetch both in the route loader in parallel |
| `await a(); await b();` with no data dependency | Start both, then await: `const [x, y] = await Promise.all([a(), b()])` |
| Partial dependency (`c` needs `b`; `a` is independent) | Chain the dependent part, run in parallel: `const [x, y] = await Promise.all([a(), getB().then(c)])` (a dangling un-awaited promise becomes an unhandled rejection if the other path throws) |
| `lazy()` component that fetches on mount | Prefetch its data alongside the chunk import (route loader or on intent) |
| Navigation feels slow | Prefetch on hover/focus/viewport for likely next routes |

## 3. Bundle

- Analyze first (`vite-bundle-visualizer`, `source-map-explorer`, `@next/bundle-analyzer`).
- Import icons/utilities by path (`lodash-es/debounce`, per-icon imports); check `sideEffects` in package.json of internal packages.
- Heavy and rarely-used code loads on interaction: `const { exportXlsx } = await import("./export-xlsx")` inside the click handler.
- Charts/editors/grids/maps/PDF behind `lazy()` + `<Suspense>` with a same-size fallback.

## 4. Re-renders and INP

- **React Compiler on**: no manual `useMemo`/`useCallback`/`memo`. Keep components pure (no mutation during render, no reading refs in render) or the compiler bails out; run its ESLint rules.
- **Compiler off**: memoize only what Profiler proves slow; stable references matter only when passed to a memoized child or used as effect deps.
- Pass expensive static subtrees as `children` so state changes in the wrapper don't re-render them.
- Typing-driven filtering: `useDeferredValue(query)`; non-urgent updates: `startTransition`.
- Any task >50ms hurts INP: chunk loops with `await scheduler.yield()` (fallback `setTimeout(0)`), or move compute to a Web Worker.

## 5. Lists, layout, media

- >100 potential rows → virtualize (TanStack Virtual / react-window). Long non-interactive sections → `content-visibility: auto` + `contain-intrinsic-size`.
- Keys are stable entity IDs, never array index, for any list that reorders, filters, or inserts.
- CLS: media has `width`/`height` or `aspect-ratio`; skeletons match final size; fonts use `font-display` + metric-matched fallback.

## 6. Memory leaks

### Cleanup matrix (every effect that acquires, releases)

| Acquired | Released by |
|---|---|
| `addEventListener` | pass `{ signal: controller.signal }`, cleanup `controller.abort()` |
| `setInterval` / `setTimeout` / `requestAnimationFrame` | `clearInterval` / `clearTimeout` / `cancelAnimationFrame` |
| Resize/Intersection/Mutation observer | `.disconnect()` |
| WebSocket / EventSource / BroadcastChannel | `.close()` |
| Store/emitter subscription | call the returned unsubscribe (prefer `useSyncExternalStore`) |
| `fetch` in an effect | `AbortController.abort()` (also prevents stale responses overwriting newer ones) |
| Third-party instance (chart, map, editor) | `.destroy()` / `.dispose()` |
| `URL.createObjectURL` | `URL.revokeObjectURL` |

StrictMode double-invokes effects in dev to expose missing cleanup. If something breaks only under StrictMode, fix the cleanup; never disable StrictMode.

### Leaks outside effects

- Module-level `Map`/array caches keyed by ID grow forever → cap size (LRU) or use `WeakMap` keyed by object.
- Long-lived singletons (event bus, websocket manager, analytics queue) holding component callbacks → handlers must unregister on unmount; buffers must be bounded.
- DOM nodes or refs stored in global stores/closures → detached DOM trees.
- Server cache: never `gcTime: Infinity` on parameterized queries; infinite queries set `maxPages`.
- Timers started at module import time (polling services) run forever → start/stop from a component or app lifecycle.

### Finding a leak (3-snapshot method)

1. Production build, DevTools → Memory. Warm up the flow once.
2. Snapshot A → perform the suspect action N times (open/close dialog, navigate away/back) → collect garbage → Snapshot B → repeat N times → Snapshot C.
3. In C, view "Objects allocated between A and B": anything still alive that scales with N is leaked. Filter `Detached` for detached DOM; follow the retainer chain to the owning closure/listener.
4. For long-lived screens (dashboards open all day), add a regression check: automate the loop (memlab or Playwright + CDP heap usage) and assert heap growth stays under a threshold.

## 7. Production reliability

- **Error boundaries** at three levels: app root (last resort), each route (with reset on location change), each independent dashboard widget (one widget failing must not blank the page). Boundaries miss event-handler and async errors: report those from mutation `onError` and global `error` / `unhandledrejection` listeners.
- **Chunk load failure after deploy** (old hashed chunks removed while a tab is open): keep previous deploy's assets on the CDN for days, and reload once on failure:

```ts
const KEY = "chunk-reload-at";
// ComponentType<any> mirrors React.lazy's own signature
export function lazyWithReload<T extends ComponentType<any>>(load: () => Promise<{ default: T }>) {
  return lazy(() =>
    load().catch((err: unknown) => {
      const last = Number(sessionStorage.getItem(KEY) ?? 0);
      if (Date.now() - last > 60_000) {
        sessionStorage.setItem(KEY, String(Date.now()));
        window.location.reload();
        return new Promise<never>(() => {}); // keep fallback visible until reload
      }
      throw err; // already reloaded recently → let the error boundary show it
    }),
  );
}
```
Vite also emits `vite:preloadError` on `window` for the same case.
- **Version skew**: long-open tabs compare a build ID (header or `/version.json`) and prompt to refresh rather than calling a changed API.
- Risky features behind a flag with a kill switch. Error rate tracked per release tag; roll back on regression.
