# SolidJS v2 mutations: actions and optimistic UI

All APIs from `"solid-js"`. Sources:
https://docs.solidjs.com/concepts/mutations,
https://docs.solidjs.com/reference/solid-js/lifecycle-actions/action

## `action`

```ts
action(genFn: (...args) => Generator | AsyncGenerator): (...args) => Promise<R>
```

- Each call runs the generator as **one transaction**. Writes are held and commit together
  when the action finishes.
- **`yield` each promise.** `yield` waits and re-enters the transaction. A plain `await`
  leaves the transaction, so writes after it commit immediately and the optimistic overlay is lost.
  If you must `await`, put a bare `yield` before the next write.
- The call returns a promise of the generator's return value. It rejects if the generator
  throws.
- Call `refresh(x)` **in the generator** after the request. Don't `await refresh(x)` inside
  an async helper that the action yields; that can hang.
- Don't call `flush()` inside an action (`FLUSH_IN_ACTION`).
- Failures **do not** reach `<Errored>`. Catch inside the generator, or `.catch()` at the
  call site and render the error yourself.
- Actions aren't cancelled when the component unmounts. Guard steps or pass an
  `AbortSignal` if that matters.

## Mutate, then refetch

```tsx
const reviews = createMemo(() => api.reviews(productId()));

const addReview = action(function* (productId: string, text: string) {
  yield api.addReview(productId, text);
  refresh(reviews);
});
```

## Optimistic list: `createOptimisticStore`

Use the derived form so the store is backed by server data:

```tsx
const [items, setItems] = createOptimisticStore<CartItem[]>(() => api.listCart(), []);

const addItem = action(function* (item: CartItem) {
  setItems((list) => { list.push({ ...item, pending: true }); }); // optimistic prediction
  yield api.addToCart(item);
  refresh(items);                                                  // reconcile to server
});

const removeItem = action(function* (id: string) {
  setItems((list) => list.filter((i) => i.id !== id));
  yield api.removeFromCart(id);
  refresh(items);
});
```

- Synchronous writes before the `yield` are the prediction and show immediately.
- When the action settles (success **or** failure), the overlay is discarded. You don't need
  rollback code. `refresh(items)` brings in the server's truth, and if it matches the
  prediction nothing visibly changes.
- `{ pending: true }` in the predicted row gives a per-row "saving" state. The refreshed
  server row doesn't have the flag.
- Writes made outside an action are not tentative.
- Rows reconcile by `key` (default `"id"`). Set the `key` option if identity is elsewhere.
- A plain `createStore` keeps a failed write. Use `createOptimisticStore` for tentative
  writes.

## Optimistic single value: `createOptimistic`

```tsx
const [name, setName] = createOptimistic("Ada");

const rename = action(function* (next: string) {
  setName(next);           // shows now
  yield api.rename(next);  // settle commits or reverts
});
```

Derived form: `createOptimistic(() => serverName())` computes the authoritative value. The
optimistic overlay reverts after each action. To keep a value, write it to the authoritative
source inside the action.

## Pending without predicting: `affects`

```ts
affects(target: Accessor | Store): void
affects(store, key): void
```

This marks a source or store property as pending while the action is in flight. It doesn't change
the value and doesn't refetch, so pair it with `refresh`:

```tsx
const setQuantity = action(function* (id: string, qty: number) {
  const item = items.find((i) => i.id === id);
  if (item) affects(item, "quantity");   // isPending(() => item.quantity) → true
  yield api.setQuantity(id, qty);
  refresh(items);
});
```

## Wait for a live source to confirm: `until`

```tsx
yield until(() => messages.some((m) => m.clientId === clientId), { timeout: 10_000 });
```

This keeps the action (and its overlay) open until confirmed data satisfies the predicate. A
timeout or abort rejects and drops the overlay.

## Handling action errors

```tsx
const [error, setError] = createSignal<string>();

<button onClick={() => addItem(item).catch((e) => setError(String(e)))}>Add</button>
<Show when={error()}>{(msg) => <p role="alert">{msg()}</p>}</Show>
```

Or catch inside the generator for local retry state, and refresh in `finally`:

```tsx
const setGiftWrap = action(function* (id: string, on: boolean) {
  try {
    yield api.setGiftWrap(id, on);
  } catch (e) {
    setRowError(id, String(e));
  } finally {
    refresh(items);
  }
});
```

## Common problems

| Symptom | Fix |
|---|---|
| Row snaps back after saving | Add `refresh(source)` after the `yield` |
| Value flashes or never appears | A write happened after `await`, outside the transaction. Use `yield`. |
| Tentative write stays after failure | Use `createOptimisticStore`/`createOptimistic`, not a plain store/signal |
| Failed action doesn't show `Errored` | Expected. Use `.catch()` at the call site or catch in the generator. |
| Save keeps going after navigation | Actions aren't cancelled. Guard steps or pass an `AbortSignal`. |
| `refresh` inside a helper hangs | Call `refresh` directly in the generator |
