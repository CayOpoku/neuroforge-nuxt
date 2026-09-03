# Type Safety, Type Placement & Per-File-Type Rules

---

## 1. `any` vs `unknown`

- **`any` is banned.** It disables checking and silently propagates.
- **`unknown` is the correct tool** for values whose shape you do not control — caught errors, third-party payloads, `JSON.parse` output. Narrow it before use:

```ts
function handle(error: unknown) {
  if (error instanceof Error) return error.message
  if (typeof error === 'object' && error !== null && 'message' in error) {
    return String((error as { message: unknown }).message)
  }
  return 'Unknown error'
}
```

- Never "fix" a typecheck error by widening to `any` or adding a blind `as`. Find why the type is wrong. A cast is a claim you must be able to justify in one sentence.
- `readBody<T>()` and `getQuery<T>()` are **casts, not validation**. Typed does not mean safe — see the schema validation rules in `backend-errors.md`.

---

## 2. Type placement (non-negotiable)

- **≤ 5 lines:** may live inline in the file that uses it.
- **> 5 lines:** MUST move to a dedicated `.types.ts` file. Never inline in a Vue SFC, composable, or server route.
- **Where it lives depends on who uses it:**
  - client only → `app/types/`
  - server only → `server/types/`
  - **both client and server → `shared/types/`** (Nuxt 4 auto-imports `shared/` into both contexts; this is the correct home for API request/response contracts)
- **Never create a new types file when a relevant one exists.** Extend it.
- Always import explicitly: `import type { Order } from '~~/shared/types/order.types'`
- Re-export Prisma-generated types rather than hand-rewriting them:

```ts
// shared/types/order.types.ts
import type { Order, OrderStatus } from '@prisma/client'

export type OrderSummary = Pick<Order, 'id' | 'status' | 'total' | 'createdAt'>
export type { OrderStatus }
```

Derive from the source of truth (`Pick`, `Omit`, `Prisma.OrderGetPayload<...>`) so the schema stays the single definition.

---

## 3. Type fix workflow

1. Run the check, capture the full output.
2. Group errors by root cause, not by file.
3. For each group: fix the cause. A missing `select` field, a wrong generic, or an unvalidated boundary — not a lazy annotation.
4. If the root cause is unclear, **pause and ask**.
5. Create or extend `.types.ts` files as needed.
6. Re-run to confirm zero errors. Report the before/after count.

---

## 4. Rules by file type

### Composables (`app/composables/`)
- Name MUST start with `use` — `useUserOrders.ts`, never `orders.ts`. A helper that is not a composable does not get a `use` prefix; it goes in `app/utils/`.
- Accept `MaybeRefOrGetter<T>`, unwrap with `toValue()` internally.
- Return `readonly` refs, destructuring-friendly.
- Small and focused — one job per composable.
- Pure helpers live **outside** the composable function (avoids re-creation per call and closure retention).
- Prefer a VueUse composable over hand-rolled listeners and timers — they self-dispose, so there is no `onUnmounted` to forget. If you do register something manually, clean it up in `onUnmounted`. See `reactivity.md`.

### Components (`app/components/`, `<script setup lang="ts">`)
- Strict `defineProps<{ ... }>()` and `defineEmits<{ ... }>()` — type-only, never runtime object syntax.
- UI plus minimal orchestration only. Complex logic goes to a composable.
- Derive with `computed`, not with a watcher writing to a ref (`reactivity.md`).
- Never call browser APIs in `setup()`. Prefer a VueUse composable, `useCookie`, or `useState`; fall back to `onMounted()` for browser-only libraries.
- Casing, folder placement and wrapper rules: `components.md`.

### Nitro routes (`server/api/`)
- `defineEventHandler` → **validate with a schema** → business logic in `server/utils/` → Prisma → typed response.
- Annotate the return type explicitly so the client infers it correctly.
- Thin routes. A handler over ~40 lines has business logic that belongs in `server/utils/`.
- Never leak raw Prisma errors to the client — wrap in `createError`.

### Prisma
- Singleton via `globalForPrisma` in `server/utils/db.ts` (see `patterns.md`).
- Precise `select` / `include` — never over-fetch columns.
- Paginate every list endpoint. An unbounded `findMany` is a production incident waiting to happen.
- Schema-first; `prisma generate` in `postinstall`.
- Share generated types through `shared/types/`.
