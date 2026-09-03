# Codebase Smells, Audits & Quality Guards

---

## 1. Dormant files

Codebases accumulate orphaned components, unused composables, dead utilities and stale config. During a Tier 2 analysis (`01-project-analysis.md` / `05-potential-spaghetti-risks.md`), audit for them and list them with a recommendation.

Deleting dead application code during an authorised refactor is expected — that is the Boy Scout rule. The never-delete rule protects `neuroforge/` memory files, not source. Say what you removed and why.

---

## 2. No legacy patches in development

With zero live users, a temporary frontend workaround for a flawed backend contract is pure debt with no upside. Do not write one.

Recommend the clean fix. If the correct change is a backend change, say so plainly and specifically — which endpoint, which field, what shape it should return — so the user can act on it rather than absorbing the defect into the frontend.

---

## 3. Environment variables

- **No hardcoded secrets.** API keys, tokens, or environment-specific URLs in source are a critical finding, not a nit.
- **No silent fallbacks for mandatory values.** `process.env.API_URL || 'http://localhost:3000'` turns a misconfigured production deploy into a silent failure that points at localhost.
- **Fail loudly at startup** for anything mandatory:

```ts
// server/utils/env.ts
export function requiredEnv(key: string): string {
  const value = process.env[key]
  if (!value) {
    throw new Error(`[CONFIG] Mandatory environment variable '${key}' is not set.`)
  }
  return value
}
```

Named `requiredEnv`, not `useRequiredEnv` — the `use` prefix is reserved for composables, and this is a server utility.

Note it reads `process.env` directly and does **not** touch `useRuntimeConfig()`. Runtime config is nested (`{ public: { ... } }`), so a flat `useRuntimeConfig()[key]` lookup silently misses every nested key and returns `undefined` — a fallback that never fires. Where runtime config is the right source, read the typed path explicitly:

```ts
const config = useRuntimeConfig(event)
if (!config.stripeSecretKey) {
  throw createError({ statusCode: 500, statusMessage: 'Payment provider is not configured' })
}
```

Optional values with a sensible default are fine — the rule targets values whose absence breaks the app.

---

## 4. Smells to flag in an audit

1. **Dormant files** — unused components, composables, utilities.
2. **Hardcoded secrets or silenced env vars** — inline keys, or fallbacks masking missing config.
3. **Legacy patches** — frontend hacks compensating for backend defects.
4. **False fallbacks** — UI defaults hiding a failed request (`backend-errors.md`).
5. **Generic error strings** — `"Something went wrong"` where a backend message exists.
6. **God components** — over ~200 lines, or mixing routing with presentation.
7. **Unbounded queries** — a `findMany` with no `take`, or a table with no pagination.
8. **Unvalidated boundaries** — `readBody<T>()` with no schema parse.
9. **Client-trusted ids** — a query scoped by a request parameter instead of the session.
10. **`any` escapes** — including implicit ones from an untyped `catch` or `JSON.parse`.
11. **Duplicated Shadcn blocks** — the same primitive markup pasted across pages instead of an `app-*` wrapper.
12. **Orphaned state** — a Pinia store holding server data that a query layer should own.
13. **Imperative reactivity** — a `watch` writing to a ref that should be a `computed`, a `watch` calling `refresh()`, data fetched in `onMounted`, or hand-rolled listener/timer cleanup where VueUse would self-dispose (`reactivity.md`).
14. **Misfiled composables** — `composables/` past ~15 files with no domain prefix in the names, or holding pure functions that belong in `utils/` or `shared/` (`structure.md`).
15. **Sibling layer coupling** — `admin` reaching into `client` or vice versa, via a `#layers/` alias, a relative path out of the layer root, or an auto-imported symbol the layer does not define. Shared code moves down to `base`, never sideways (`structure.md`).
