---
name: react
description: >
  Architects, builds, refactors, and reviews enterprise-grade React + TypeScript web applications for production:
  feature/module boundaries, server-cache vs client state, strict types with runtime parsing at trust boundaries,
  async waterfalls, bundle size and re-renders, memory and data leaks, client-side security (XSS, CSP, tokens,
  open redirect, multi-tenant cache), accessibility, and seam-based tests (RTL, Playwright). Use whenever writing,
  refactoring, or reviewing non-trivial React/TS code (feature folders, data fetching, forms, tables, dashboards,
  auth flows, perf or leak bugs), even if the user never says "enterprise".
---

# Enterprise React + TypeScript

Production React that stays fast, correct, and leak-free under real traffic and long sessions. Rules below are the non-obvious ones; apply normal React/TS competence for everything else.

## Step 0: detect, then conform

Before writing code, read `package.json`, `tsconfig*.json`, and lint config. Note: React version, React Compiler on/off, router (and whether it has loaders), server-cache lib, schema lib, styling/token system, test runner. Follow what exists. Only when nothing exists, default to TanStack Query + Zod + RTL/MSW + Playwright.

## Hard rules

**Boundaries**

1. Import a feature/package only through its root entry files. Never reach into its subfolders. Features never import each other: compose them in `app/`/routes, or communicate via URL, props, or shared events. Dependency flow: `shared → features → app`.
2. No `export *` mega-barrels. Small entry files per context (`index.ts`, `server.ts`). Import big icon/UI libs by direct path.
3. Add an interface/seam only when ≥2 real implementations exist (e.g., HTTP + in-memory fake) or at an I/O edge. Merge pass-through modules (deletion test: if removing it just moves code, it was shallow).

**Types and data** 4. `strict` + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes`. `as` only with a comment proving the invariant. 5. Everything crossing a trust boundary is `unknown` until parsed: responses, `FormData`, URL params, storage, `postMessage`, env, LLM output. Derive types from the schema; don't hand-write a duplicate interface. 6. Brand IDs and money that can be confused (`UserId`, `TenantId`, `Cents`). 7. Async state is a discriminated union (`status`), never parallel booleans. Exhaustive `switch` with a `never` check.

**State and UI** 8. Place state at the lowest fitting rung: local → lifted (≤3 levels of prop drilling) → URL (filters, page, tab, anything shareable) → server cache (all server data) → global store (rare, atomic selectors only). Never copy server-cache data into `useState`/store via an effect. 9. Values derivable from props/state are computed during render, not synced with `useEffect`. Effects are for external systems only. 10. Every async region handles loading, empty, error (+retry), success, and permission-denied. Skeleton only when there is no data yet: background refetch keeps current UI. Mutations disable their trigger while pending. 11. Components ≤200 lines; split data orchestration (container) from presentation. Prefer composition/slots over >6 config props. Styling uses design tokens only: no raw hex, no off-scale spacing.

**Performance** 12. Independent requests start in parallel (route loader / prefetch / `Promise.all`), never in sequential `await`s. Every request gets an `AbortSignal` and a timeout. 13. Lazy-load routes and heavy libs (charts, editors, grids, PDF/3D). Virtualize lists that can exceed 100 rows. Images/video declare dimensions. 14. Memoize only with a before/after measurement; no measurable win ⇒ revert. With React Compiler on, don't hand-memoize at all.

**Security and telemetry** 15. No session/refresh tokens in `localStorage`/`sessionStorage`. No raw-HTML sinks without a sanitizer. Redirect targets checked against an allowlist. Every cache key includes tenant/user scope; purge caches on logout and tenant switch. 16. Never swallow errors. Error boundaries per route/feature report structured events (`correlationId`, `entryPoint`). No PII in logs, URLs, analytics, or metric labels.

## Leak guard (check on every change)

- **Memory**: every effect that subscribes, listens, observes, times, or opens a socket returns cleanup; in-flight requests abort on unmount/param change; module-level `Map`/arrays are bounded; long-lived stores don't hold DOM nodes or large closures.
- **Data**: no cross-tenant cache hits, no tokens in JS-readable storage, no secrets in public env vars (`VITE_*`, `NEXT_PUBLIC_*`), no PII in URLs or logs.

## Playbooks (read only the one the task needs)

| Task                                                                                           | Read                                                                               |
| ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Folder layout, module boundaries, cross-feature communication, lint enforcement, ADRs          | `references/architecture-boundaries.md` (+ `assets/dependency-cruiser.config.cjs`) |
| Server cache, query keys, optimistic updates, HTTP adapter, schemas, branded types, forms      | `references/state-data-types.md`                                                   |
| Slow page, waterfalls, bundle, re-renders, long lists, memory leak, prod crashes, chunk errors | `references/performance-reliability.md`                                            |
| Auth/tokens, XSS, CSP, redirects, multi-tenant, PII, logging/telemetry                         | `references/security-observability.md`                                             |
| Writing tests, a11y of dialogs/forms/routing/async updates                                     | `references/testing-a11y.md`                                                       |

For a review, read the playbook for each area the diff touches.

## Done gate

- `tsc --noEmit` 0 errors; lint and boundary check (dependency-cruiser) 0 violations.
- Budgets: LCP ≤2.5s, INP ≤200ms, CLS ≤0.1, initial JS <200KB gzip.
- axe: 0 serious/critical. Keyboard path works for the changed flow.
- Tests drive the public seam (role/label queries); only network, time, and storage are mocked.
- Leak guard passes. Report any rule you deliberately broke and why.
