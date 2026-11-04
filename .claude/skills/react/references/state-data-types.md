# State, Data & Types

Examples use TanStack Query v5 + Zod; translate to the project's libs if different.

## 1. HTTP adapter (one per app, in `shared/api`)

All network goes through it: abort + timeout + parse + typed errors in one place.

```ts
export class HttpError extends Error {
  constructor(readonly status: number, readonly correlationId: string | null, message: string) {
    super(message);
    this.name = "HttpError";
  }
}

type Init = Omit<RequestInit, "signal"> & { signal?: AbortSignal; timeoutMs?: number };

export async function request<T>(url: string, parse: (body: unknown) => T, init: Init = {}): Promise<T> {
  const { timeoutMs = 15_000, signal, ...rest } = init;
  const signals = signal ? [signal, AbortSignal.timeout(timeoutMs)] : [AbortSignal.timeout(timeoutMs)];
  const res = await fetch(url, { credentials: "include", ...rest, signal: AbortSignal.any(signals) });
  const correlationId = res.headers.get("x-correlation-id");
  if (!res.ok) throw new HttpError(res.status, correlationId, `${rest.method ?? "GET"} ${url} -> ${res.status}`);
  const body: unknown = res.status === 204 ? undefined : await res.json();
  return parse(body); // Zod: (b) => Schema.parse(b) — a contract break fails loudly here, not deep in UI
}
```

- 401 → one central handler (refresh once, else logout). 403 → permission-denied UI, not a generic error.
- Retries: only idempotent requests, only on network/5xx/408/429 (honor `Retry-After`). TanStack retries every error 3× by default, which delays 403/404 UI by seconds: set `new QueryClient({ defaultOptions: { queries: { retry: (n, e) => n < 2 && !(e instanceof HttpError && e.status < 500 && e.status !== 408 && e.status !== 429) } } })`.
- `AbortSignal.any` needs Chrome 116 / Safari 17.4; for older targets combine signals with a manual `AbortController`.
- Very large payloads: parse at the adapter once; never re-parse in components.

## 2. Schemas are the source of types

```ts
export const TenantId = z.string().uuid().brand<"TenantId">();
export const OrderId = z.string().uuid().brand<"OrderId">();
export const Order = z.object({ id: OrderId, totalCents: z.number().int(), status: z.enum(["new", "shipped"]) });
export type Order = z.infer<typeof Order>;
```

Without a schema lib, brand with `type Brand<T, B> = T & { readonly __brand: B }` and create values only through a parsing constructor (`toOrderId(raw: string): OrderId`). Parse URL params and storage with the same schemas, falling back to defaults on failure instead of crashing.

## 3. Query key factory, scoped

Every dimension the response varies by goes in the key: tenant, user/role when responses differ, locale, filters. A missing dimension serves one tenant's cached data to another.

```ts
type Scope = { tenantId: TenantId; locale: string };
export const orderKeys = {
  all: (s: Scope) => ["orders", s.tenantId, s.locale] as const,
  list: (s: Scope, f: OrderFilters) => [...orderKeys.all(s), "list", f] as const,
  detail: (s: Scope, id: OrderId) => [...orderKeys.all(s), "detail", id] as const,
};
export const orderListQuery = (s: Scope, f: OrderFilters) =>
  queryOptions({
    queryKey: orderKeys.list(s, f),
    queryFn: ({ signal }) => request(`/api/orders?${toSearch(f)}`, (b) => OrderPage.parse(b), { signal }),
    // keep old rows while paging/filtering, but never across tenants (plain keepPreviousData would show tenant A's rows while B loads)
    placeholderData: (prev, prevQuery) => (prevQuery?.queryKey[1] === s.tenantId ? prev : undefined),
    staleTime: 30_000,
  });
```

Reuse the same `queryOptions` for `useQuery`, route-loader `ensureQueryData`, and hover `prefetchQuery`.

## 4. Rendering async state (5 states, no flicker)

```tsx
const q = useQuery(orderListQuery(scope, filters));
if (q.data === undefined) {
  if (q.isError) return isForbidden(q.error) ? <NoAccess /> : <ErrorState onRetry={() => void q.refetch()} />;
  return <TableSkeleton />;
}
if (q.data.items.length === 0) return <EmptyState />;
return <OrderTable rows={q.data.items} dimmed={q.isPlaceholderData} />; // refetch errors → non-blocking banner/toast
```

Hand-rolled async state (wizards, uploads, websockets) uses a union + reducer:

```ts
type Upload =
  | { status: "idle" }
  | { status: "uploading"; progress: number; abort: () => void }
  | { status: "done"; fileId: FileId }
  | { status: "failed"; error: string; retryable: boolean };
// switch (s.status) { ...; default: { const _never: never = s; return _never; } }
```

## 5. Optimistic mutation with rollback

```ts
const qc = useQueryClient();
const m = useMutation({
  mutationFn: (id: OrderId) => request(`/api/orders/${id}/ship`, (b) => Order.parse(b), { method: "POST" }),
  onMutate: async (id) => {
    const key = orderKeys.list(scope, filters);
    await qc.cancelQueries({ queryKey: key }); // in-flight refetch would overwrite the optimistic value
    const prev = qc.getQueryData<OrderPage>(key);
    qc.setQueryData<OrderPage>(key, (p) => p && { ...p, items: p.items.map((o) => (o.id === id ? { ...o, status: "shipped" } : o)) });
    return { prev, key };
  },
  onError: (_e, _id, ctx) => { if (ctx) qc.setQueryData(ctx.key, ctx.prev); /* + toast */ },
  onSettled: () => qc.invalidateQueries({ queryKey: orderKeys.all(scope) }),
});
// <button disabled={m.isPending && m.variables === order.id}>Ship</button>
```

Use optimistic updates only when success rate is high and the change is reversible; for money/irreversible actions, wait for the server.

## 6. Placement mistakes to catch in review

| Smell | Fix |
|---|---|
| `useEffect(() => setRows(data), [data])` | Read from the query; derive in render |
| `useEffect` computing filtered/sorted list into state | `const visible = sort(filter(rows))` in render (memo only if measured slow) |
| Effect resetting state when an `id` prop changes | `<Editor key={id} />` |
| Effect reacting to a click via a flag | Put the logic in the event handler |
| Filters/page in `useState` | URL search params, schema-parsed, `replace` + debounce for typing. Free-text search that may hold PII (names, emails) stays in local state |
| Whole-store subscription `useStore()` | Atomic selector `useStore((s) => s.field)`; multiple fields → shallow compare |
| One context with value + setters | Split state and dispatch contexts (Compiler off: memoize the value object) |

## 7. Forms

- One schema for client validation; reuse server schema when shared. Report **all** field errors at once (`safeParse` → `error.issues` mapped by `path`), not first-error-only.
- Map server 422 field errors back onto fields; unknown errors → form-level message.
- Submit guard: disabled while pending + idempotency key header on create (`crypto.randomUUID()` per form session) so retries/double clicks never create duplicates.
- Guard unsaved changes on navigation (router blocker + `beforeunload`), removed on successful submit.
