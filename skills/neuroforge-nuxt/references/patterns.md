# Copy-Paste Patterns (Nuxt 4)

All paths assume the Nuxt 4 default `srcDir: 'app/'`. Verify against the project's `nuxt.config.ts`; match the project if it uses a flat Nuxt 3 layout.

---

## 1. Custom API client with `createUseFetch`

The preferred pattern for authenticated external API calls. Source: https://nuxt.com/docs/4.x/guide/recipes/custom-usefetch

```ts
// app/composables/useAPI.ts
export const useAPI = createUseFetch({
  // Do NOT call useRuntimeConfig() here — this object is evaluated at module scope,
  // outside a Nuxt context. Read config inside the hooks, which run per request.
  baseURL: '/api',

  onRequest({ options }) {
    const { session } = useUserSession()
    if (session.value?.token) {
      options.headers.set('Authorization', `Bearer ${session.value.token}`)
    }
  },

  async onResponseError({ response }) {
    if (response.status === 401) {
      await navigateTo('/login')
    }
  },
})

// Usage — typed, auth handled, no boilerplate:
// const { data: profile } = await useAPI<Profile>('/me')
// const { data: orders } = await useAPI<Order[]>('/orders')
```

If the base URL genuinely must come from runtime config, use the `$fetch` plugin in section 2 instead — a plugin runs inside a Nuxt context, so `useRuntimeConfig()` is safe there.

---

## 2. Custom `$fetch` plugin (lower-level control, runtime config safe)

```ts
// app/plugins/api.ts
export default defineNuxtPlugin((nuxtApp) => {
  const config = useRuntimeConfig()
  const { session } = useUserSession()

  const api = $fetch.create({
    baseURL: config.public.apiBase,
    onRequest({ options }) {
      if (session.value?.token) {
        options.headers.set('Authorization', `Bearer ${session.value.token}`)
      }
    },
    async onResponseError({ response }) {
      if (response.status === 401) {
        await nuxtApp.runWithContext(() => navigateTo('/login'))
      }
    },
  })

  return { provide: { api } }
})

// Usage — always wrap with useAsyncData to avoid an SSR double-fetch:
// const { $api } = useNuxtApp()
// const { data } = await useAsyncData('users', () => $api<User[]>('/users'))
```

---

## 3. Prisma singleton (`server/utils/db.ts`)

```ts
import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient }

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
  })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db
```

The guard exists so HMR does not open a new connection pool on every reload.

---

## 4. Typed Nitro route skeleton

```ts
// server/api/widgets/[id].get.ts
import { z } from 'zod'
import { db } from '~~/server/utils/db'
import type { Widget } from '@prisma/client'

const paramsSchema = z.object({ id: z.string().uuid() })

type WidgetResponse = Pick<Widget, 'id' | 'name' | 'createdAt'>

export default defineEventHandler(async (event): Promise<{ data: WidgetResponse }> => {
  const user = requireUser(event)
  const { id } = await getValidatedRouterParams(event, paramsSchema.parse)

  const widget = await db.widget.findFirst({
    where: { id, userId: user.id },              // scope by session, never by a client id
    select: { id: true, name: true, createdAt: true },
  })

  if (!widget) throw createError({ statusCode: 404, statusMessage: 'Widget not found' })

  return { data: widget }
})
```

Validate → authorise → query with a precise `select` → typed response. A handler past ~40 lines has business logic that belongs in `server/utils/`.

---

## 5. Paginated list endpoint

```ts
// server/api/orders/index.get.ts
import { z } from 'zod'

const querySchema = z.object({
  cursor: z.string().uuid().optional(),
  limit: z.coerce.number().int().min(1).max(100).default(25),
})

export default defineEventHandler(async (event) => {
  const user = requireUser(event)
  const { cursor, limit } = await getValidatedQuery(event, querySchema.parse)

  const rows = await db.order.findMany({
    where: { userId: user.id },
    select: { id: true, status: true, total: true, createdAt: true },
    orderBy: { createdAt: 'desc' },
    take: limit + 1,                                    // one extra row signals a next page
    ...(cursor && { cursor: { id: cursor }, skip: 1 }),
  })

  const hasMore = rows.length > limit
  const data = hasMore ? rows.slice(0, limit) : rows

  return { data, nextCursor: hasMore ? data.at(-1)?.id ?? null : null }
})
```

Never ship an unbounded `findMany` behind a table.

---

## 6. Composable contract

```ts
// app/composables/useFetchUserOrders.ts
import type { MaybeRefOrGetter } from 'vue'
import type { OrderSummary } from '~~/shared/types/order.types'

export function useFetchUserOrders(userId: MaybeRefOrGetter<string>) {
  const { data: orders, status, error, refresh } = useAsyncData(
    // Reactive key — refetches automatically when userId changes.
    // No `watch` option needed; the key already carries the dependency.
    () => `orders-${toValue(userId)}`,
    () => $fetch<OrderSummary[]>(`/api/users/${toValue(userId)}/orders`),
    { lazy: true, default: () => [] },
  )

  return {
    orders: readonly(orders),
    status,
    error: readonly(error),
    refresh,
  }
}
```

Pure helpers go **outside** the exported function. Register cleanup in `onUnmounted` if the composable owns listeners, timers, or subscriptions.

---

## 7. Hydration-safe patterns

```ts
// Wrong — localStorage crashes on the server
const theme = localStorage.getItem('theme') || 'light'
// Right — SSR-safe cookie
const theme = useCookie('theme', { default: () => 'light' })

// Wrong — random/time values differ between server and client
const rand = Math.random()
// Right — generated once on the server, serialised to the client
const rand = useState('rand', () => Math.random())

// Wrong — browser lib in setup()
SomeBrowserLib.init()
// Right — deferred to client mount. This is one of the few legitimate uses of onMounted.
onMounted(async () => {
  const { default: SomeBrowserLib } = await import('browser-only-lib')
  SomeBrowserLib.init()
})

// Wrong — manual listener plumbing for a browser API
onMounted(() => window.addEventListener('resize', onResize))
onUnmounted(() => window.removeEventListener('resize', onResize))
// Right — SSR-safe and self-disposing
const { width } = useWindowSize()
```

For anything reactive that is not a browser-only library, check `reactivity.md` before writing `onMounted` or `watch`.

---

## 8. Hybrid rendering route rules

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/': { prerender: true },              // built at deploy time
    '/products/**': { swr: 3600 },         // cached, revalidated in the background
    '/blog': { isr: 3600 },                // regenerated on demand after an hour
    '/app/**': { ssr: false },             // private dashboard — client-only SPA
  },
})
```

---

## 9. Nuxt layer structure

```
layers/
  shared/
    app/composables/
    app/components/
    shared/types/
  auth/
    app/composables/
    app/components/
    app/middleware/
    server/api/
    server/utils/
  products/
    app/components/
    server/api/
    server/utils/
nuxt.config.ts   <- root orchestrator only; never feature logic
```

Reach for layers when a domain has its own routes, server code, and components and is genuinely separable. Two folders and a shared button do not need a layer — YAGNI applies to architecture too.
