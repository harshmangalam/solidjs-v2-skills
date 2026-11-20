# SolidJS v2 data fetching recipes

All APIs from `"solid-js"` unless noted. Source:
https://docs.solidjs.com/guides/data-fetching-patterns

## Search as you type

```tsx
const [query, setQuery] = createSignal("");

const results = createMemo(() => {
  const text = query().trim();
  if (!text) return [];          // sync empty result: no spinner
  return api.search(text);
});

<input
  value={latest(query)}          // input reflects typing immediately
  onInput={debounce((e) => setQuery(e.currentTarget.value), 150)}
/>
<Loading fallback={<Spinner />}>
  <ul class={{ stale: isPending(results) }}>
    <For each={results()}>{(r) => <li>{r.title}</li>}</For>
  </ul>
</Loading>
```

- No request counter or `AbortController` is needed for correctness. Stale responses are dropped.
- Debounce in the event handler, not with an effect. (`debounce` here is your own helper or
  a utility library; it isn't exported by `solid-js`.)
- Without `latest(query)` on the input, a controlled input lags while the search is held.

## Several requests for one page (parallel)

Create them in the same scope so they start together, and give each region its own boundary:

```tsx
const product = createMemo(() => api.product(props.id));
const reviews = createMemo(() => api.reviews(props.id));
const related = createMemo(() => api.related(props.id));

<Loading fallback={<ProductSkeleton />}><ProductInfo product={product()} /></Loading>
<Loading fallback={<ReviewsSkeleton />}><Reviews items={reviews()} /></Loading>
<Loading fallback={<RelatedSkeleton />}><Related items={related()} /></Loading>
```

- Use `<Reveal order="sequential">` or `order="together"` to coordinate when regions appear.
- On a prop change, all three hold until ready (consistent page). Add `on={props.id}` to a
  boundary to let it show its skeleton instead.

## Dependent requests

Only wait when the data needs it:

```tsx
const product = createMemo(() => api.product(props.id));
const brand = createMemo(() => api.brand(product().brandId)); // real dependency
```

If the second request only needs something you already have (e.g. `props.id`), derive it from that
input directly so both start at once. Dev attribution reports chains of 3+ sequential
requests as `ASYNC_WATERFALL`. To fix it, pass the input you already have, or join on the server.

## Pagination

```tsx
const [page, setPage] = createSignal(1);
const [orders] = createStore(async () => api.orders({ page: page() }), [] as Order[]);

<Loading fallback={<TableSkeleton />}>
  <table class={{ stale: isPending(() => orders.length) }}>...</table>
</Loading>
<Pager current={latest(page)} onChange={setPage} />
```

To show a skeleton per page instead of dimming the old page, use `<Loading on={page()}>`.

## Infinite scroll

An async memo receives its previous value:

```tsx
const [page, setPage] = createSignal(1);
const items = createMemo(async (previous: Order[] = []) => {
  const p = page();                     // read before await
  const next = await api.orders({ page: p });
  return [...previous, ...next];
});
```

To reset on a filter change, remount the list component keyed on the filter so the memo
starts again with no previous value.

## Share one request across components

Create the source once in a common ancestor and pass it through context:

```tsx
const CartContext = createContext<Accessor<Cart>>();

function CartProvider(props) {
  const cart = createMemo(() => api.cart());
  return <CartContext value={cart}>{props.children}</CartContext>;
}

// consumer
const cart = useContext(CartContext)!;
<Loading fallback={<Spinner />}>{cart().items.length}</Loading>
```

For caching and deduping across routes, use Solid Router's `query`
(https://docs.solidjs.com/routing/solid-router/data).

## Keep data fresh (polling)

```tsx
const stats = createMemo(() => api.dashboardStats());

onSettled(() => {
  const id = setInterval(() => refresh(stats), 30_000);
  return () => clearInterval(id);
});
```

A plain `refresh` is quiet (`isPending` stays false). To show pending, do it in an action
with `affects(stats)` before `refresh(stats)`.

## Router data

With Solid Router, load route data with `createMemo(() => getUser(params.id))`, where
`getUser` is wrapped in the router's `query` for caching, preload, and revalidation.
Navigation already runs as a held update, so the old route stays visible until the
new route's data is ready. Use `<Loading on={params.id}>` when a param change should show
a skeleton. See https://docs.solidjs.com/routing/solid-router/data.

## Server functions

`"use server"` functions return promises, so pass them straight to `createMemo`. On the
server, mark errors that are safe to show to the client with `markSafeError`. Unmarked
errors are replaced with a generic message in production.

## Common problems

| Symptom | Fix |
|---|---|
| Typing in the search box lags | Bind the input to `latest(query)` |
| Old product stays visible after navigating | Add `on={props.id}` to its `Loading` |
| Two components make the same request | Share one memo via context, or use router `query` |
| `refresh` shows nothing loading | `affects(source)` before `refresh` inside an action |
| Click does nothing visible | Add `isPending` to the affected content (`SILENT_HOLD`) |
| Computation never updates for some input | That input is read after an `await`. Move the read to the top. |
| `Loading on` fallback never appears | Data is also read outside the boundary (`LOADING_ON_OUTSIDE_HOLD`) |
| Unneeded request chain | Derive the second request from the original input |
