# Architecture & Module Boundaries

## Layout

```
src/
  app/                 # composition only: routes, layouts, providers, error boundaries
  features/<name>/     # deep module: big behaviour behind a small API
    index.ts           # public API (components, hooks, types) — the only import target
    server.ts          # optional: loaders/prefetch for route files (keeps client bundle clean)
    api/ model/ ui/    # private internals; free to change without touching callers
  shared/<pkg>/        # ui primitives, http client, lib; no business knowledge
    index.ts
```

Name features after domain nouns from the business (`invoicing`, `shipments`), not after tech (`hooks`, `modals`). If a term is ambiguous across teams, add it to `GLOSSARY.md` once and use it verbatim in code.

## Enforce, don't hope

Copy `assets/dependency-cruiser.config.cjs` to the repo root as `.dependency-cruiser.cjs`, adjust the three roots, run `npx depcruise src --config .dependency-cruiser.cjs` in CI. Also map TS path aliases to entry files only, one per entry kind (`"@features/*": ["src/features/*/index.ts"]`, `"@features/*/server": ["src/features/*/server.ts"]`), so deep imports fail to resolve in the editor.

## Cross-feature needs (features never import each other)

| Need | Pattern |
|---|---|
| Page shows A and B together | Route file in `app/` composes `<A.Panel/>` and `<B.Panel/>` |
| B needs an action from A | `app/` passes a callback or render slot: `<B.Detail actions={<A.ApproveButton id={id}/>}/>` |
| B reacts to A's data change | Both invalidate/read the same server-cache key via a key factory in `shared/api`, or A emits a typed event on a shared bus |
| Shared selection/filter | URL search params, read by each feature |
| Same type used by both | The type belongs to the API contract → `shared/api` schema; or the concept is really one feature (merge) |

Repeated need for the same cross-feature link is a signal the boundary is wrong: merge or extract a new feature.

## Depth checks

- **Deep module**: interface much smaller than implementation. A feature exporting 20 symbols is shallow; export the few things callers actually compose.
- **Deletion test**: imagine deleting a module. If complexity vanishes, it was a pass-through; inline it. If complexity reappears in N callers, it earns its place.
- **Seam rule**: introduce an interface/adapter only with ≥2 real implementations (production HTTP + in-memory fake for tests/storybook) or at an I/O edge (network, storage, time, analytics). One implementation behind an interface is indirection, not abstraction.
- **Design it twice**: for a module whose interface will be widely imported, sketch two radically different interfaces (e.g., hook-based vs component-based) and choose by caller simplicity, not implementation ease.

## Entry files

- Export explicitly by name. No `export * from './ui'` chains: they defeat tree-shaking in some bundlers, pull side-effectful modules into every importer, and hide circular imports.
- Split entry files by runtime when a feature has server-only code (`server.ts`) or heavy optional UI (`charts.ts` imported lazily).
- Never re-export third-party libs through a feature; import them where used.

## When to write an ADR

Write `docs/adr/NNNN-title.md` only if all three hold: hard to reverse, surprising without context, a real trade-off was made (e.g., choosing a global store, splitting into a monorepo package, picking token strategy). Keep it to Context / Decision / Consequences, ≤1 page.

## Monorepo packages

Promote `shared/<pkg>` to a workspace package only when a second app consumes it. Then: `exports` field lists entry files explicitly, `sideEffects: false` (or list CSS files), and the package has its own `tsc --noEmit` and tests.
