# Data Fetching, Caching & State Ownership

The single highest-leverage area in a Nuxt SaaS app. Read before writing any `useAsyncData`, `useFetch`, `useQuery`, or store.

---

## 1. Pick the right tool

| Situation | Use |
| :--- | :--- |
| Page-level data the route needs to render | `useAsyncData` / `useFetch` |
| Data shared across components, refetched on focus, invalidated after writes | **Pinia Colada** `useQuery` |
| Any write (create/update/delete) with cache invalidation | **Pinia Colada** `useMutation` |
| Small global client state (sidebar open, theme, wizard step) | `useState` |
| Global state with real logic and actions (auth session, cart rules) | Pinia store |
| Imperative one-off call inside an event handler | `$fetch` |

**`useState` vs Pinia:** if it is a value, use `useState`. If it is a value *plus behaviour that several components must share*, use a Pinia store. Do not create a store to hold one boolean.

**Never call `$fetch` at the top level of `setup()`** — it runs twice (server and client). Wrap it in `useAsyncData`, or use `useQuery`.

---

## 2. `useAsyncData` — the parts that matter

```ts
const {
  data, status, error, refresh, clear,
} = await useAsyncData(
  // reactive key: MaybeRefOrGetter<string>. Refetches automatically when it changes —
  // no `watch` option needed when the key already carries the dependency.
  () => `orders-${toValue(userId)}`,
  () => $fetch<OrderSummary[]>(`/api/users/${toValue(userId)}/orders`),
  {
    lazy: true,          // do not block navigation — render the skeleton immediately
    server: true,        // keep SSR for SEO-relevant data; false for private dashboards
    default: () => [],   // shape-correct empty value, NOT a fake success value
    transform: (rows) => rows.map(toRow),   // shrink the payload before it is serialised
    pick: ['id', 'status'],                 // or drop fields entirely
    dedupe: 'defer',     // 'cancel' aborts the in-flight call; 'defer' reuses it
    getCachedData: (key, nuxtApp) => nuxtApp.payload.data[key] ?? nuxtApp.static.data[key],
  },
)
```

**Non-blocking is the default posture.** `lazy: true` plus a `status`-driven template beats a blocked navigation on every screen that is not SEO-critical:

```vue
<template>
  <app-skeleton v-if="status === 'pending'" />
  <app-alert v-else-if="status === 'error'" :message="getErrorMessage(error)" />
  <app-empty v-else-if="!data?.length" />
  <orders-table v-else :rows="data" />
</template>
```

Four states, always. `status` is `'idle' | 'pending' | 'success' | 'error'` — branch on it, never on `!data` alone, which cannot tell "loading" from "empty" from "failed".

**Key rules:** keys must be stable and unique. Two components using the same key share one request and one cache entry — that is the deduplication mechanism, not a bug. A key that accidentally collides across pages will serve stale data.

**`getCachedData`** is what stops a refetch on every client-side navigation back to a page. Add it to any list that does not change per-visit.

---

## 3. Pinia Colada — the caching layer

Nuxt's built-in fetching has no shared cache invalidation, no stale-while-revalidate, and no mutation story. Pinia Colada (from the Pinia author) adds them. Use it once the app has **writes that must invalidate reads across components** — that is the threshold, not "it looks nicer".

### Setup

```bash
npm add @pinia/colada
npm dlx nuxi module add @pinia/colada-nuxt
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@pinia/colada-nuxt'],
})
```

```ts
// colada.options.ts  (repo root — passed to the PiniaColada plugin)
import type { PiniaColadaOptions } from '@pinia/colada'

export default {
  queryOptions: {
    staleTime: 30_000,
    gcTime: 5 * 60_000,
  },
} satisfies PiniaColadaOptions
```

The module auto-imports `useQuery`, `useMutation`, `useInfiniteQuery`, `useQueryState`, `useQueryCache`, `useMutationCache`, `defineQuery`, `defineQueryOptions`, `defineMutation`.

**It does not require `await` for SSR** — it uses `onServerPrefetch` internally. Only `await refresh()` if you deliberately want to block client navigation.

### Queries

```ts
const { data, error, status, asyncStatus, isLoading, refresh, refetch } = useQuery({
  key: ['orders', 'list'],
  query: () => $fetch<OrderSummary[]>('/api/orders'),
  staleTime: 30_000,
})
```

- `status` — `'pending' | 'success' | 'error'`: is there data yet?
- `asyncStatus` — `'idle' | 'loading'`: is a request in flight right now?

Those are two different questions. A background revalidation of already-rendered data is `status: 'success'` + `asyncStatus: 'loading'` — show a subtle indicator, not a full skeleton. Getting this wrong is the most common Colada mistake.

### Keys are hierarchical

Keys are arrays of anything JSON-serialisable, matched by **prefix**:

```ts
['orders']                  // invalidating this...
['orders', 'list']          // ...invalidates these too
['orders', orderId]
['orders', { id, type }]
```

Structure keys broadest → narrowest so one invalidation can sweep a whole domain.

### Parametrised queries

```ts
// app/queries/orders.ts
export const orderByIdOptions = defineQueryOptions((id: string) => ({
  key: ['orders', id],
  query: () => $fetch<Order>(`/api/orders/${id}`),
}))

// in a component
const route = useRoute()
const { data: order } = useQuery(orderByIdOptions(route.params.id as string))
```

Define query options **once, in `app/queries/`**, and import them. Duplicating a key literal in two components is how cache bugs are born.

### Mutations + invalidation

```ts
const queryCache = useQueryCache()

const { mutate: createOrder, asyncStatus } = useMutation({
  mutation: (payload: CreateOrderInput) =>
    $fetch<Order>('/api/orders', { method: 'POST', body: payload }),

  onSuccess() {
    // prefix match — clears every ['orders', ...] entry
    queryCache.invalidateQueries({ key: ['orders'] })
    toast.success('Order created')
  },
  onError(error) {
    toast.error(getErrorMessage(error))   // backend-owned message, see backend-errors.md
  },
})
```

`mutate` is fire-and-forget (errors land in `onError`); `mutateAsync` returns a promise and **will reject** — if you use it, you own the `try/catch`.

Disable the submit button on `asyncStatus === 'loading'`, never on `status`.

### Do not mix layers for the same resource

One resource, one fetching mechanism. A list read through `useQuery` and written through a raw `$fetch` will not invalidate, and you will ship a stale table. If you adopt Colada for a domain, adopt it for that domain's writes too.

---

## 4. Pinia store shape

```ts
// app/stores/useAuthStore.ts
export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)
  const isAuthenticated = computed(() => !!user.value)

  async function login(credentials: LoginCredentials) {
    user.value = await $fetch<User>('/api/auth/login', { method: 'POST', body: credentials })
  }

  function logout() {
    user.value = null
  }

  return { user: readonly(user), isAuthenticated, login, logout }
})
```

Setup syntax, `readonly` on exposed state, mutations only through actions. **Server-cached data does not belong in a Pinia store** — that is what `useAsyncData` and `useQuery` are for. Stores hold client state and behaviour.
