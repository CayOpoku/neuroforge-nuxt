# Server Contracts: Input Validation & Error Handling

Validation and errors are one subject. An error message is only trustworthy if the boundary that produced it actually checked its input.

---

## 1. Validate every boundary with a schema

`readBody<T>()` and `getQuery<T>()` are **casts, not validation**. They tell TypeScript what you hope arrived; they check nothing at runtime. Every route that accepts input parses it with a schema first.

```ts
// shared/schemas/order.schema.ts — shared so the form and the route agree
import { z } from 'zod'

export const createOrderSchema = z.object({
  email: z.string().email(),
  quantity: z.number().int().positive().max(100),
  note: z.string().max(500).optional(),
})

export type CreateOrderInput = z.infer<typeof createOrderSchema>
```

```ts
// server/api/orders/index.post.ts
import { createOrderSchema } from '~~/shared/schemas/order.schema'

export default defineEventHandler(async (event) => {
  const user = requireUser(event)
  const body = await readValidatedBody(event, createOrderSchema.parse)
  // body is now genuinely CreateOrderInput
  return createOrder({ ...body, userId: user.id })
})
```

- `readValidatedBody(event, schema.parse)` and `getValidatedQuery(event, schema.parse)` are built into H3 — use them rather than parsing by hand.
- Put schemas in `shared/schemas/` so the client form and the server route validate against **one definition**. `z.infer` then replaces the hand-written type entirely.
- A thrown `ZodError` surfaces as a 400. Shape it once, globally, rather than per route:

```ts
// server/utils/validation.ts
import { ZodError } from 'zod'

export function toValidationError(error: unknown) {
  if (error instanceof ZodError) {
    return createError({
      statusCode: 400,
      statusMessage: 'Validation failed',
      data: {
        fields: error.issues.map((i) => ({ path: i.path.join('.'), message: i.message })),
      },
    })
  }
  return error
}
```

- Never trust an id from the client to scope a query. See `auth-middleware.md`.
- valibot is a drop-in alternative if bundle size matters — the rule is a schema at the boundary, not a specific library.

---

## 2. Zero hardcoded API error messages

- **The frontend never invents an API error string.** No `"An unexpected error occurred"`, `"Failed to fetch data"`, or `"Operation failed"` written into a Vue SFC, composable, or form handler.
- **All user-facing API errors originate from the primary backend** — whether that backend is FastAPI, NestJS, Express, Go, or Nitro itself.
- **Nitro proxies forward the upstream payload intact.** A BFF layer must not replace a real backend error with a synthetic generic one.
- **Audit rule:** during analysis, flag every hardcoded string inside a `catch` block or `$fetch` error handler.

### The one exception: transport-level failures

If the request never reached the backend, there is no backend message to render, and `String(error)` would show the user `TypeError: Failed to fetch`. These four cases are legitimately frontend-owned and belong in `getErrorMessage`, nowhere else:

| Case | Message |
| :--- | :--- |
| Network failure / offline | "Could not reach the server. Check your connection and try again." |
| Timeout / aborted | "The request timed out. Please try again." |
| 5xx with an empty body | "Something went wrong on our end. Please try again shortly." |
| Genuinely unparseable payload | "An unexpected error occurred." |

Everything else must come from the response body.

---

## 3. The `getErrorMessage` utility

Lives in `app/utils/error.ts` (or `shared/utils/error.ts` if the server needs it too). Parses the common backend shapes, and is typed with `unknown` — the skill's zero-`any` rule applies to its own utilities.

```ts
// app/utils/error.ts

interface BackendErrorPayload {
  message?: string | string[]
  detail?: string | Array<{ msg?: string; loc?: string[] }>
  error?: string
  statusCode?: number
  statusMessage?: string
}

const TRANSPORT_FALLBACK = 'Could not reach the server. Check your connection and try again.'
const TIMEOUT_FALLBACK = 'The request timed out. Please try again.'
const SERVER_FALLBACK = 'Something went wrong on our end. Please try again shortly.'
const UNKNOWN_FALLBACK = 'An unexpected error occurred.'

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null
}

function readPayload(error: Record<string, unknown>): BackendErrorPayload {
  if (isRecord(error.data)) return error.data as BackendErrorPayload
  if (isRecord(error.response) && isRecord((error.response as Record<string, unknown>)._data)) {
    return (error.response as Record<string, unknown>)._data as BackendErrorPayload
  }
  return error as BackendErrorPayload
}

export function getErrorMessage(error: unknown): string {
  if (!error) return UNKNOWN_FALLBACK
  if (typeof error === 'string') return error

  // Transport-level: the request never produced a backend payload.
  if (error instanceof DOMException && error.name === 'AbortError') return TIMEOUT_FALLBACK
  if (error instanceof TypeError) return TRANSPORT_FALLBACK

  if (!isRecord(error)) return UNKNOWN_FALLBACK

  const payload = readPayload(error)

  // FastAPI: { detail: '...' } or { detail: [{ msg, loc }] }
  if (typeof payload.detail === 'string') return payload.detail
  if (Array.isArray(payload.detail)) {
    const messages = payload.detail.map((d) => d.msg).filter(Boolean)
    if (messages.length) return messages.join(', ')
  }

  // NestJS / Express: { message: '...' } or { message: ['...'] }
  if (typeof payload.message === 'string') return payload.message
  if (Array.isArray(payload.message) && payload.message.length) return payload.message.join(', ')

  // H3 / Nitro
  if (typeof payload.statusMessage === 'string') return payload.statusMessage
  if (typeof payload.error === 'string') return payload.error

  // 5xx that returned nothing useful
  const status = typeof payload.statusCode === 'number' ? payload.statusCode : undefined
  if (status && status >= 500) return SERVER_FALLBACK

  if (typeof error.message === 'string' && error.message) return error.message

  return UNKNOWN_FALLBACK
}
```

Field-level validation errors (the `data.fields` array from section 1) are rendered next to their inputs, not flattened into a toast. `getErrorMessage` is for the summary line.

---

## 4. Rendering errors

```ts
try {
  await $fetch('/api/users/profile', { method: 'POST', body: formData.value })
  toast.success('Profile updated')
} catch (error) {
  toast.error(getErrorMessage(error))
}
```

With Pinia Colada, the same call belongs in `onError` — see `data-fetching.md`.

---

## 5. No false fallbacks

- **The smell:** defaulting UI state when a call fails — showing `status = 'Pending'` or `role = 'User'` because the fetch errored. This hides a system fault and shows the user a value that is not true.
- **The rule:** render the failure. An `<app-alert>` or a toast carrying the backend message, so both the user and the developer see exactly what happened.
- `default: () => []` in `useAsyncData` is fine — an empty array is a *shape*, not a fabricated value. `default: () => ({ status: 'Pending' })` is not.
- Distinguish the four states in every data-bound view: loading, empty, error, success. Collapsing "error" into "empty" is the same bug wearing a different hat.
