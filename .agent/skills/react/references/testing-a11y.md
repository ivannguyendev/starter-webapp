# Seam Testing & Accessibility

## 1. Test at the public seam

Render the feature's entry component the way the app does (real providers, real hooks, real child components). Mock only the edges of the system: **network** (MSW), **time** (fake timers / `setSystemTime`), **storage**, **randomness**. Never mock child components, your own hooks, internal modules, or React state: those tests break on refactor and pass on bugs.

```tsx
function renderWithProviders(ui: ReactElement) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false }, mutations: { retry: false } } });
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
} // fresh client per test: a shared client leaks cache between tests → order-dependent flakes

// MSW setup: server.listen({ onUnhandledRequest: "error" }) so an unmocked call fails the test loudly
```

Queries: `getByRole(name)` > `getByLabelText` > `getByText` > `getByTestId` (last resort, for non-semantic canvases). Async: `findBy*`; `waitFor` contains assertions only, no side effects.

## 2. What every async feature tests

| Behaviour | How |
|---|---|
| Loading → success | default handler; `await findByRole("row", { name: /ord-1/ })` |
| Empty | handler returns `[]`; assert empty-state text + primary action |
| Error + retry | handler 500 once then 200; click Retry; rows appear |
| Permission denied | handler 403; assert no-access message, no retry button |
| Optimistic rollback | mutation handler 500; click action; value flips, then reverts; error announced |
| Double submit | click twice fast; handler receives exactly 1 request |
| URL state | render at `?status=shipped&page=2`; assert filter control state + request params |

## 3. Assertion hygiene

- No tautologies: don't recompute the production formula in `expect`. Use literal expected values from a worked example (`"$1,234.50"`).
- No large snapshots; snapshot only small, stable, serialized output.
- Assert outcomes users/callers see (DOM, request sent, URL), not internal call counts.
- Builders for test data, parsed through the real schema so fixtures can't drift: `buildOrder({ status: "shipped" })`.
- Work in vertical slices: one failing behaviour test → minimal code to pass → next behaviour → refactor while green.

## 4. E2E (Playwright): critical journeys only

Login/session expiry, the primary money/transaction path, tenant/permission boundaries (user of tenant A cannot see tenant B data after switch), and one smoke per top route. Use `getByRole`, web-first assertions (`await expect(locator).toBeVisible()`), `storageState` for auth, no fixed sleeps. Run `@axe-core/playwright` on key pages with zero serious/critical violations as a gate.

## 5. Accessibility beyond the basics

- **Route change**: SPA navigation is silent to screen readers. After navigation, move focus to the page `<h1>` (`tabIndex={-1}`) and update `document.title`.
- **Dialogs**: native `<dialog>` + `showModal()` or a tested headless library. Requirements: initial focus inside, `Escape` closes, background inert, focus returns to the trigger on close.
- **Composite widgets** (combobox, menu, tabs, tree, grid): follow ARIA Authoring Practices keyboard models; use a tested headless library instead of hand-rolling.
- **Async announcements**: a polite live region (`role="status"`) mounted *before* content changes announces "24 results", "Saved". Errors that block the user use `role="alert"`. Toasts never hold the only copy of critical info.
- **Form errors**: `aria-invalid` + `aria-describedby` pointing at the error text; on failed submit, focus the first invalid field (or an error summary linking to fields).
- **Loading**: `aria-busy="true"` on the updating region; skeletons `aria-hidden`; spinners have an accessible name.
- **Virtualized tables**: set `aria-rowcount` (total) and `aria-rowindex` per rendered row so assistive tech knows the real size.
- Motion respects `prefers-reduced-motion`; state is never conveyed by color alone.
- Automated checks (axe) catch roughly a third of issues: do one manual keyboard-only pass of every changed flow.
