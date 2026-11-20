---
name: solidjs-data-fetching
description: Fetch, load, refetch, and mutate async data in SolidJS v2 using async createMemo/createStore, <Loading>, <Errored>, isPending, latest, refresh, affects, createDeferred, action, and createOptimistic/createOptimisticStore. Use whenever writing or reviewing SolidJS v2 code that loads data from an API or server, shows loading/error states, refetches, polls, paginates, searches as you type, or saves data (mutations, optimistic UI).
---

# SolidJS v2 data fetching

In SolidJS v2, async data is part of the reactive graph. A computation that returns a
promise **is** the data source. You do not write loading flags, abort controllers, or
stale-response guards. Solid tracks, holds, and discards these for you.

Reference: https://docs.solidjs.com/migration/data-fetching-from-solid-1 and
https://docs.solidjs.com/concepts/async-reactivity

All APIs below are imported from `"solid-js"`.

## Core rules

1. **Fetch with an async computation, not an effect.**
   `createMemo(() => fetchX(input()))` for a value, `createStore(async () => ..., seed)` for
   collections/objects. The accessor is typed as the resolved value (`T`), not
   `Promise<T>` or `T | undefined`.
2. **Read every reactive input before the first `await`.** Only synchronous reads are
   tracked. Reads after an `await` don't re-run the computation, and in dev they error.
3. **Throw on failure.** Don't return `{ success: false }` or an empty fallback value. A thrown
   error/rejected promise goes to the nearest `<Errored>` boundary.
4. **Wrap readers in `<Loading>`** for the first load, and in `<Errored>` for failures.
   `Loading` does not catch errors; `Errored` does not show loading UI.
5. **Refetches keep the current content.** After the first value, an input change *holds*
   the update until new data is ready, and other changes in the same update wait with it, so
   the page never shows mixed old/new data. Make the wait visible with `isPending`.
6. **Decide the refetch behavior per fetch** (table below). Don't leave a held update with no
   visible feedback; that looks like a dead click (dev warns `SILENT_HOLD`).
7. **Reload with `refresh(source)`**, not a "reload counter" signal.
8. **Save with `action(function* ...)`** and `yield` each promise. Show instant results with
   `createOptimistic` / `createOptimisticStore`. See [references/mutations.md](references/mutations.md).

## Load one thing

```tsx
import { createMemo, Loading, Errored } from "solid-js";

function UserPage(props: { id: string }) {
  const user = createMemo(() => fetchUser(props.id)); // Accessor<User>

  return (
    <Errored fallback={(err, reset) => (
      <div role="alert">
        {String(err())} <button onClick={reset}>Retry</button>
      </div>
    )}>
      <Loading fallback={<UserSkeleton />}>
        <h1>{user().name}</h1>
      </Loading>
    </Errored>
  );
}

async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`Failed to load user ${id}`); // throw, don't return a flag
  return res.json();
}
```

Lists and objects: use a store. The response is reconciled into the store by `id`.

```tsx
const [orders] = createStore(async () => api.orders(customerId()), [] as Order[]);
```

## Decide per fetch: what should the user see while it reloads?

| Want | Do |
|---|---|
| Keep old content until new data is ready (default; data must agree with the rest of the page) | Nothing extra. Add `isPending` to dim it. |
| Control (input, tab, pager) updates now, content waits | `latest(signal)` on the control + `isPending` on the content |
| Show a skeleton when the *subject* changes (e.g. other user's profile) | `<Loading on={key()} fallback={...}>` around that content |
| Independent widget (feed, suggestions) shouldn't make the page wait | Create it with `createDeferred` instead of `createMemo` + `isPending` |

Usually the first two go together. Test: if showing the previous subject's data with a
pending indicator for a few seconds is acceptable, `isPending`/`latest` is fine; if it would
be misleading (one user's name over another's data), use `on`.

### `isPending(fn)` shows that a reload is in flight

```tsx
<ul class={{ stale: isPending(results) }}>
  <For each={results()}>{(r) => <li>{r.title}</li>}</For>
</ul>
```

- Works on the source, a derived memo, a prop many levels down, or the signal whose write
  started the update. No prop-threading of loading flags.
- `false` on first load (the `Loading` fallback handles that).
- A bare `refresh(x)` does **not** flip `isPending`. Call `affects(x)` first if it should.
- An errored expression is not pending and `isPending` doesn't rethrow, so also read the
  value inside an `Errored` boundary.
- For stores, probe a property: `isPending(() => orders.length)`.

### `latest(fn)` puts the newest value on the control

```tsx
<input value={latest(query)} onInput={(e) => setQuery(e.currentTarget.value)} />
```

Use it on **inputs/controls only**, never on displayed data. Reading data through `latest`
can show new results while the rest of the page is still old.

### `<Loading on={value}>` shows the fallback again when the subject changes

```tsx
<Loading on={params.id} fallback={<UserSkeleton />}>
  <UserProfile user={user()} />
</Loading>
```

- Pass a **value**, not an accessor: `on={accountId()}`, not `on={accountId}` (dev warns
  `LOADING_ON_ACCESSOR`). Several keys: `on={[accountId(), tab()]}`.
- A same-subject `refresh` keeps content in place. Only a key change shows the fallback.
- If the same data is also read *outside* this boundary, the update waits for it anyway
  and the fallback never shows (`LOADING_ON_OUTSIDE_HOLD`). Move all readers inside.
- With `on`, the rest of the page stops waiting for this panel.

### `createDeferred` creates data the page doesn't wait for

```tsx
const feed = createDeferred(() => fetchFeed(store.period));
// <section class={{ stale: isPending(feed) }}>...
```

After its first load it never holds the page's updates. It may lag behind but never gets
ahead. Its first load still uses the nearest `Loading` unless `loadingValue` is set. Use
`createMemo` for totals and selected-item details that must match the page.

## Boundaries

- **Create async values high** (requests start early), **put `<Loading>` low** (small
  fallback regions). Nesting doesn't cause waterfalls: requests follow data dependencies.
- Wrap the data slot, not layout chrome. A boundary around the nav/header blanks the
  whole screen on reload.
- Passing an async accessor as a prop doesn't wait. Only the expression that *reads* it
  waits. Derived memos inherit the waiting.
- `Loading` children stay alive behind the fallback; they aren't recreated.
- `<Errored fallback={(err, reset) => ...}>`: `err` is an accessor (`err()`), and `reset()`
  re-runs the failing sources. Errors also clear on their own when an input changes or a
  refresh succeeds. Put an `Errored` around each region that should fail independently.
- `Errored` doesn't catch rejections from promises started in event handlers (e.g. an
  action call). Handle those in the action or with `.catch()` at the call site.

### Rendering the first-load branch yourself

Prefer a boundary. If it doesn't fit, give a placeholder with `loadingValue`:

```tsx
const orders = createMemo<Order[] | undefined>(() => fetchOrders(filters()), {
  loadingValue: undefined,
});

<Show when={orders()} fallback={<Spinner />}>
  {(loaded) => <OrderTable orders={loaded()} />}
</Show>
```

Stores: `createStore(async () => ..., seed, { seedLoadingValue: true })` makes the seed the
placeholder.

## Reloading

```tsx
const user = createMemo(() => fetchUser(id()));

<button onClick={() => refresh(user)}>Reload</button>
const fresh = await refresh(user); // optional: resolves with the settled value
```

- Targets: an accessor from `createMemo`/`createDeferred`, or a store from `createStore(fn)`
  / `createProjection`. A plain `createSignal` accessor is a no-op.
- Fire-and-forget is safe; an ignored failed refetch doesn't raise an unhandled rejection.
- To show pending during a manual reload, mark it inside an action:
  `action(function* () { affects(user); refresh(user); })`.

## Cancellation and cleanup

Stale responses are discarded automatically; there's no need for request counters. The old request
still runs to completion. To actually cancel it, register cleanup **before the first await**:

```tsx
const results = createMemo(async () => {
  const q = query();                       // read inputs first
  const controller = new AbortController();
  onCleanup(() => controller.abort());     // before the first await
  const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
    signal: controller.signal,
  });
  if (!res.ok) throw new Error("Search failed");
  return res.json();
});
```

A run that is never shown is cleaned up when a newer run replaces it. The run on screen is
cleaned up when its replacement appears.

## Anti-patterns (rewrite these)

| Don't | Do |
|---|---|
| `createEffect` that calls `fetch` then sets a signal/store | `createMemo(() => fetch...)` / `createStore(async () => ..., seed)` |
| `isLoading` signal + `<Show when={!isLoading()}>` | `<Loading>` for first load, `isPending` for reloads |
| `AbortController`/request-id checks to drop stale responses | Nothing (automatic). Use `onCleanup` only to cancel network work. |
| `reloadCount` signal read by the fetch | `refresh(source)` |
| `catch` → `return []` / `{ success: false }` | Let it throw. Wrap the readers in `<Errored>`. |
| Reading a signal after `await` | Read all inputs at the top of the function |
| `on={id}` (accessor) | `on={id()}` |
| `latest(data)` on displayed content | `latest` on controls only. Use `isPending` for content. |
| Debouncing by writing a signal from an effect | Debounce the event handler: `onInput={debounce(handler, 150)}` |
| Same request created in two components | Create the memo once in a common ancestor and share via context, or use Solid Router `query` |

## More

- Recipes (search as you type, pagination, infinite scroll, dependent and parallel
  requests, polling, shared requests, router): [references/patterns.md](references/patterns.md)
- Mutations, actions, optimistic UI: [references/mutations.md](references/mutations.md)

## Checklist

- [ ] Every fetch is an async `createMemo`/`createStore`/`createDeferred`, not an effect.
- [ ] Inputs are read before the first `await`.
- [ ] Failures throw, and an `Errored` boundary covers each region.
- [ ] `Loading` wraps the readers (and only the data slot).
- [ ] Each fetch has a reload decision: default + `isPending`, `latest` on control, `on`, or `createDeferred`.
- [ ] `isPending` gives feedback wherever a click would otherwise look dead.
- [ ] Every `on` receives a value, not an accessor.
- [ ] No loading flags, abort-for-staleness guards, or reload counters remain.
- [ ] Reloads use `refresh(source)`, with `affects` where pending should show.
