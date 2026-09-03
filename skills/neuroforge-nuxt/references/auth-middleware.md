# Auth, Middleware & Route Protection

---

## 1. The two middleware layers

| Layer | Runs | Purpose |
| :--- | :--- | :--- |
| `app/middleware/*.ts` (route middleware) | Server on first load, client on navigation | UX-level gating — redirect an unauthenticated visitor to `/login` |
| `server/middleware/*.ts` (Nitro middleware) | Every server request, before the route handler | Populate `event.context`, attach the session, set headers |

**Route middleware is not security.** It runs in the browser on client-side navigation and can be bypassed. Every protected `server/api/` route must re-check authorisation itself. Treat route middleware as "don't show the user a broken page", and the server check as the actual gate.

---

## 2. Route middleware

```ts
// app/middleware/auth.ts
export default defineNuxtRouteMiddleware((to) => {
  const { loggedIn } = useUserSession()
  if (!loggedIn.value) {
    return navigateTo(`/login?redirect=${encodeURIComponent(to.fullPath)}`)
  }
})
```

```ts
// app/pages/dashboard/index.vue
definePageMeta({
  layout: 'dashboard',
  middleware: 'auth',
})
```

- Always `return navigateTo(...)` — a bare call does not halt the navigation.
- Preserve the intended destination in a `redirect` param so login returns the user where they were going.
- Name middleware for the rule it enforces (`auth`, `admin`, `onboarded`), not the page it guards.
- Use `middleware: 'auth'` on pages rather than a global middleware when only part of the app is gated — a global one runs on the login page too.

---

## 3. Server-side authorisation (the real gate)

```ts
// server/middleware/session.ts — attaches the session to every request
export default defineEventHandler(async (event) => {
  const session = await getUserSession(event)
  event.context.user = session?.user ?? null
})
```

```ts
// server/utils/auth.ts
import type { H3Event } from 'h3'
import type { SessionUser } from '~~/shared/types/auth.types'

export function requireUser(event: H3Event): SessionUser {
  const user = event.context.user as SessionUser | null
  if (!user) {
    throw createError({ statusCode: 401, statusMessage: 'Authentication required' })
  }
  return user
}

export function requireRole(event: H3Event, role: SessionUser['role']): SessionUser {
  const user = requireUser(event)
  if (user.role !== role) {
    throw createError({ statusCode: 403, statusMessage: 'Insufficient permissions' })
  }
  return user
}
```

```ts
// server/api/orders/index.get.ts
export default defineEventHandler(async (event) => {
  const user = requireUser(event)
  // scope every query to the caller — never trust an id from the client
  return db.order.findMany({ where: { userId: user.id } })
})
```

**Scope by the session, not by a client-supplied id.** `/api/orders?userId=123` where `userId` comes from the query string is an IDOR, not a feature. This is the single most common auth bug in a SaaS dashboard.

---

## 4. Multi-tenancy

If the app is multi-tenant, every Prisma query on a tenant-owned model carries the tenant id from the session:

```ts
const { id: userId, organizationId } = requireUser(event)

const invoice = await db.invoice.findFirst({
  where: { id: invoiceId, organizationId },   // tenant scope is part of the WHERE, always
})
if (!invoice) throw createError({ statusCode: 404, statusMessage: 'Invoice not found' })
```

Return **404, not 403**, for a resource in another tenant — a 403 confirms the record exists.

Flag any tenant-scoped query missing its `organizationId` filter as a critical finding during review.

---

## 5. Session handling rules

- Sessions live in httpOnly cookies. Never persist a token in `localStorage` — it is XSS-readable and breaks SSR.
- `useUserSession()` (nuxt-auth-utils) or the project's equivalent is the single client entry point. Do not read the session cookie manually in components.
- On 401 from the API client, clear the session and redirect once — see the `onResponseError` handler in `patterns.md`.
- Never log tokens, password hashes, or full session objects. Log the user id.
- Auth-related env vars are mandatory: fail loudly at startup if missing (see `smells.md`).
