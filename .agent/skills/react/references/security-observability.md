# Client Security, Privacy & Observability

## 1. Auth tokens

- Preferred for enterprise: **BFF** (backend-for-frontend) holds tokens; browser only has an `httpOnly; Secure; SameSite=Lax|Strict` session cookie. No token ever reaches JS.
- If the SPA must hold an access token: keep it in a module variable inside the HTTP adapter (not React state, not a persisted store, not storage). Refresh token stays in an `httpOnly` cookie scoped to the refresh path.
- Single-flight refresh: concurrent 401s share one refresh promise; on refresh failure → purge (section 4) and redirect to login.
- Cookie auth needs CSRF defence on state-changing requests: `SameSite` + CSRF token header (or server-verified custom header).
- OAuth/OIDC: Authorization Code + PKCE with `state`; never implicit flow.

## 2. XSS sinks

Raw-HTML and code sinks: `dangerouslySetInnerHTML`, `innerHTML`/`outerHTML`/`insertAdjacentHTML`, `document.write`, `eval`, `new Function`, string `setTimeout`, markdown renderers with raw HTML enabled. Only feed them sanitized output (`DOMPurify.sanitize(html, { USE_PROFILES: { html: true } })`), sanitized as close to the sink as possible.

User-controlled URLs (`href`, `src`, `window.open`, `location.assign`) must pass a protocol allowlist:

```ts
export function safeHref(raw: string): string | undefined {
  try {
    const u = new URL(raw, window.location.origin);
    return ["http:", "https:", "mailto:"].includes(u.protocol) ? u.href : undefined;
  } catch {
    return undefined;
  }
}
```

## 3. Open redirect (`?next=`, `returnTo`, deep links)

```ts
const ALLOWED_ORIGINS = new Set([window.location.origin]);
export function safeRedirect(next: string | null, fallback = "/"): string {
  if (!next) return fallback;
  try {
    const u = new URL(next, window.location.origin); // normalizes //evil.com and /\evil.com
    if (!ALLOWED_ORIGINS.has(u.origin)) return fallback;
    const path = `${u.pathname}${u.search}${u.hash}`;
    return path.startsWith("//") ? fallback : path; // "/.//evil.com" is same-origin but yields "//evil.com"
  } catch {
    return fallback;
  }
}
```

`startsWith("/")` checks are bypassable (`//evil.com`, `/\evil.com`); compare parsed origins, then reject a returned path starting with `//`.

## 4. Multi-tenant isolation & purge

- Cache keys carry tenant (and user/role/locale when responses differ). See query key factory in `state-data-types.md`.
- On **logout, tenant switch, or user change**: cancel in-flight queries → `queryClient.clear()` → call each store's `reset()` → delete persisted caches (IndexedDB persister, per-user storage keys) → hard navigate. Partial purges leak the previous user's data into the next session on shared machines.
- Cross-tab: broadcast logout on `BroadcastChannel("auth")`; every tab runs the same purge.
- Persisted query caches: buster includes user+tenant; never persist sensitive queries.

## 5. Other boundaries

- `postMessage`: check `event.origin` against an exact allowlist, parse `event.data` with a schema, send with an explicit `targetOrigin` (never `"*"`).
- Env: anything prefixed `VITE_`, `NEXT_PUBLIC_`, `REACT_APP_` is shipped to every browser. Only public config there; parse env with a schema at startup.
- Source maps: upload to the error tracker; don't serve them publicly.
- Third-party scripts can read the whole DOM: minimize, pin with SRI, load after consent.
- CSP (deliver via header; start with `Content-Security-Policy-Report-Only`):
  `default-src 'self'; script-src 'self' 'nonce-{n}' 'strict-dynamic'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'; require-trusted-types-for 'script'`.
  With Trusted Types enforced, string HTML sinks throw: sanitize with `DOMPurify.sanitize(html, { RETURN_TRUSTED_TYPE: true })` or a named policy.
- `Referrer-Policy: strict-origin-when-cross-origin`, so path/query never leak to third parties.

## 6. PII

- Never in URLs (logged by servers, proxies, analytics, Referer). Use opaque IDs or request bodies.
- Analytics events use an allowlist of properties; session replay masks all inputs by default.
- Error reporting: scrub in `beforeSend` (emails, phones, tokens, `Authorization` headers, form values).

## 7. Observability

Design events from on-call questions: *which release, which route, which API call, which tenant cohort, how many users?*

```ts
type ClientEvent = {
  level: "error" | "warn" | "info";
  event: string;            // stable name: "order.ship.failed"
  entryPoint: string;       // route pattern or widget: "/orders/:id", "OrdersTable"
  release: string;
  correlationId?: string;   // from HttpError / response header
  errorName?: string;
  message: string;          // no PII
};
```

- Generate `x-request-id` (`crypto.randomUUID()`) per request, read the server's correlation ID back, and show it in error UI ("Reference: 7f3a…") so support can trace one failure end to end.
- `catch` either recovers into a visible UI state or reports and rethrows. `console.log(e)` alone is swallowing.
- Metric labels come from bounded sets only (route pattern, status class, release, feature). `userId`, email, raw URL, and error message go in log/trace attributes, never metric labels.
- Errors 100% sampled, traces sampled; filter known noise (extension errors, `ResizeObserver loop` warnings) before alerting.
