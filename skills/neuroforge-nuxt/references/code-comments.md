# Code Comments

Comments are load-bearing or they are noise. This file sets the bar for every comment written in this codebase.

Adapted for Nuxt/Vue from the `code-comments` skill (skills.sh). Where the two differ, this file wins — it is tuned for a convention-driven Nuxt 4 repo where the path already says what a file is.

---

## 1. The bar

**Default: no comment.** Write the code so the comment is unnecessary — a better name, an early return, an extracted composable, a named constant. Only when the *why* genuinely cannot live in the code does a comment earn its place.

**The why test:** if the comment restates *what* the line does, delete it. If it explains *why this line and not the obvious one*, keep it.

```ts
// ✗ Restates the code
// Fetch the user's orders
const { data } = await useAsyncData('orders', () => $fetch('/api/orders'))

// ✓ Explains the non-obvious choice
// Key carries the page so back-navigation refetches instead of serving page 1's cache.
const { data } = await useAsyncData(() => `orders-${page.value}`, fetchOrders, { watch: [page] })
```

---

## 2. Size and shape

- **One line.** Two only for a real trade-off with a real cost. Never a paragraph, never a banner, never an essay above a 10-line function.
- **Sentence case, plain language, no ceremony.** No `/** */` block where `//` says it.
- **Above the line it explains**, never trailing off the end of a long line.
- **No file-header blocks.** `app/composables/useOrderTotals.ts` does not need a header announcing that it holds order-total logic. A single line is allowed only where the purpose is genuinely not inferable from the path — a block registry, a webhook handler bound to an external contract, a file whose shape is dictated by a third party.
- **No section banners** (`// ===== STATE =====`, `// --- handlers ---`). A file that needs signposting needs splitting — `workflow.md` §5.

---

## 3. Never reference the `neuroforge/` folder — or this session

`neuroforge/` is local analysis memory. It is not part of every developer's workflow, it may never be committed, and a teammate cloning the repo will not have it. A comment pointing into it is a dead link the day it is written.

```ts
// ✗ Dead reference — the reader does not have this file
// See neuroforge/03-orders-architecture.md for why we denormalise here.

// ✓ Points at something that travels with the code
// Totals are denormalised on Order — the dashboard aggregates 10k+ rows per tenant (#412).
```

**Comments may only reference things that ship with the repo:** a file path in the repo, a symbol name, an issue/PR id, a migration name, an official docs URL.

The split is: the *reasoning* lives in the analysis file, the *one-line why* lives in the code and must stand alone without it. Never copy paragraphs out of `neuroforge/` into a source file.

**Equally banned — narrating the session or your own work:**

```ts
// ✗ Added by the assistant
// ✗ Updated to fix the bug reported earlier
// ✗ As requested, switched this to a computed
// ✗ Step 3 of the refactor
// ✗ v2 — new version of the handler above
```

Git records who changed what and when. A comment that describes the *edit* rather than the *code* is stale the moment the next edit lands.

---

## 4. What actually earns a comment here

| Situation | Example |
| :--- | :--- |
| A watcher where `computed` would be expected | `// watch, not computed: the editor is uncontrolled and must not re-render per keystroke.` |
| A client-only guard | `// Chart lib reads window at import time — SSR crashes without this guard.` |
| A non-obvious `useAsyncData` option | `// server: false — this endpoint needs the browser session cookie, which SSR has no access to.` |
| A `transform` that drops data | `// Drops body: the list renders titles only, and the payload is ~2MB with it.` |
| A deliberate Prisma narrowing | `// select, not include — passwordHash must never leave this function.` |
| A wrapper deviating from the primitive | `// 44px target instead of the primitive's 36px: primary touch action (Fitts).` |
| A workaround, with its expiry | `// Workaround for nuxt#28414 double-hydration. Remove once we are on 4.2.` |
| A guarantee that justifies a non-null assertion | `// Route middleware guarantees a session here; a null user is a bug, not a state.` |
| `<style scoped>` at all | `// Keyframes — utilities cannot express this.` (`components.md` §1) |

Everything on this list is one sentence that a reader could not have recovered from the code.

---

## 5. What never earns one

1. Restating the line below it.
2. Commented-out code — git has it, delete it (`workflow.md` §5).
3. Values a constant already names: `// 5 minutes` above `STALE_AFTER_MS`. Reference the constant by name instead; unit translations (`1_048_576 // 1MB`) are fine.
4. Types TypeScript already states: `@param userId - the user id` on `userId: string`.
5. Obvious props, obvious getters, obvious one-line helpers.
6. Vague intent: `// TODO: refactor later`, `// clean this up`, `// might need this`.
7. Anything from §3 — `neuroforge/` paths, session narration, changelog lines.

---

## 6. JSDoc

Only on **exported** utilities, composables and server helpers whose contract is not obvious from the signature. One sentence on what it guarantees, plus anything a caller would get wrong. Never a `@param` table that repeats the types.

```ts
/** Returns the backend's message verbatim. Never falsy — do not `||` a fallback onto it (`backend-errors.md` §4). */
export function getErrorMessage(error: unknown): string
```

```ts
/** Scoped to the session tenant — callers must not pass a client-supplied tenantId. */
export async function listOrders(event: H3Event): Promise<Order[]>
```

---

## 7. Templates

Keep `<!-- -->` rare. A template that needs headings to be navigable is a component that needs splitting (`components.md` §3). The legitimate uses are a non-obvious structural constraint or an accessibility decision:

```vue
<!-- aria-live on the wrapper, not the row: screen readers miss per-row updates. -->
<div aria-live="polite">
```

---

## 8. TODO / HACK / FIXME

Actionable and traceable, or not written at all. Owner or issue id, and the condition that retires it.

```ts
// TODO(#412): move totals into the aggregate query once the reporting view lands.
// HACK: Safari 17 flexbox gap bug — drop when we stop supporting 17.x.
// FIXME(#488): rapid toggling races; in-flight mutation is not cancelled.
```

No issue and no owner means it is not a TODO, it is litter.

---

## 9. Maintenance

- **A stale comment is a bug.** When you edit a line, re-read the comment above it — update it or delete it in the same edit.
- **Delete comments with the code they describe.** An orphaned comment above a rewritten block is worse than nothing; it is confidently wrong.
- **Match the file.** If the surrounding code comments sparsely, comment sparsely (`workflow.md` §5, consistency).

---

## 10. Review checklist

Flag in an audit:

1. Header blocks or banners restating what the path already says.
2. Any comment naming `neuroforge/`, an analysis file, or the conversation that produced the code.
3. Comments describing the edit rather than the code (`// updated`, `// new`, `// as requested`).
4. Commented-out code.
5. JSDoc restating types, or `@param` tables on self-describing signatures.
6. TODOs with no owner and no issue.
7. Comments contradicting the code beneath them.
