# Offline Data, Dexie & PWA Persistence

Read before installing Dexie, writing an IndexedDB schema, or adding an offline fallback to any query. Assumes `data-fetching.md` — Pinia Colada owns the network cache; this file only covers what survives a reload with no network.

---

## 1. Mode gate — resolve this before writing any data layer

**Do not infer the mode.** A `dexie` entry in `package.json` is compatible with all three modes and proves nothing.

Check `neuroforge/00-project-overview.md` for a recorded **Offline Mode**. If it is recorded, follow it and do not ask again. If it is not, ask once — one question, three options — and stop for the answer.

This is the one question this skill asks at any tier: it is an architecture decision the user owns, it is asked exactly once per project, and getting it wrong means rewriting the data layer. It does not license any other clarifying question — the tier's normal no-ask posture still applies to everything else.

| Mode | Source of truth | Reads offline | Writes offline | Default patterns |
| :--- | :--- | :--- | :--- | :--- |
| **Online-only** | Server | ✗ | ✗ | `data-fetching.md` only. **Do not install Dexie.** |
| **Hybrid** | Server | ✓ (last known) | ✗ (queued or blocked) | §4 for server data, §5 for drafts/settings |
| **Local-first** | Dexie | ✓ | ✓ (outbox → sync) | §5 everywhere; the network is a background sync job |

Then record the answer in `00-project-overview.md` (append, never overwrite — see `project-memory.md`):

```markdown
## Offline Mode
**Hybrid** — decided 2026-09-03. Server owns catalogue + orders; Dexie caches
reads for offline viewing. Drafts and user settings are local-first (§5).
```

**Choosing when the user asks you to pick:** most SaaS dashboards are **online-only** — Colada's cache already covers a flaky connection, and Dexie is a second source of truth to keep consistent forever. Escalate to **hybrid** only when the app must render real data with the network fully down. Reserve **local-first** for apps whose primary job happens offline (field capture, note-taking, POS) — it is the only mode that forces you to own conflict resolution.

Per-table overrides are normal and expected. "Hybrid" does not mean every table is hybrid; drafts and local settings are local-first in every mode above online-only.

---

## 2. Guardrails

- **Single source of truth per table.** A table is owned by Colada (server) or by Dexie (local) — never both. Never mirror a Dexie row into a `ref`/`reactive` and keep the two in sync by hand; consume `useQuery` or a live query directly.
- **Validate at the boundary, not everywhere.** Every payload crossing *into* persistence is parsed (§6). Reads from your own tables are trusted unless a schema version changed.
- **Degrade, never fabricate.** An offline fallback returns the last known row and says so in the UI. It never returns an empty object, a zero, or a placeholder shaped like success — same rule as `default: () => []` in `data-fetching.md`.
- **No conflict-resolution engine.** Last-write-wins on `updatedAt`, plus an outbox, until the user asks for more. CRDTs and merge loops are out of scope by default.

---

## 3. SSR: Dexie is client-only

IndexedDB and `navigator` do not exist in Nitro. **A Dexie call that runs during SSR crashes the render** — the most common way this layer breaks in Nuxt (see `debugging.md`).

Keep every Dexie access behind an `import.meta.client` guard, in one place:

```ts
// app/utils/localCache.ts
import type { Table } from 'dexie'

export async function readLocal<T>(table: Table<T>, id: string): Promise<T | undefined> {
  if (!import.meta.client) return undefined
  return table.get(id)
}

export async function writeLocal<T>(table: Table<T>, row: T): Promise<void> {
  if (!import.meta.client) return
  await table.put(row)
}
```

Server render → network only. Client → cache-aware. One guard, not fifty.

Also: **never branch on `navigator.onLine` for correctness.** It reports whether an interface is up, not whether your API is reachable — a captive portal and a dead backend both report `true`. Use `useOnline()` (VueUse, see `reactivity.md`) for *UI state* — a banner, a disabled button — and let a failed request be the authority on reachability.

---

## 4. Pattern A — hybrid query (network-first, durable fallback)

For server-owned data that must still render with the network down.

```ts
// app/queries/products.ts
import { z } from 'zod'
import { db } from '~/database'
import { readLocal, writeLocal } from '~/utils/localCache'

export const ProductSchema = z.object({
  id: z.string(),
  name: z.string(),
  price: z.number(),
  updatedAt: z.string(),
})
export type Product = z.infer<typeof ProductSchema>

export const productByIdOptions = defineQueryOptions((id: string) => ({
  key: ['products', id],
  query: async (): Promise<Product> => {
    try {
      const raw = await $fetch(`/api/products/${id}`)
      const product = ProductSchema.parse(raw)      // parse on write, once
      await writeLocal(db.products, product)
      return product
    }
    catch (networkError) {
      const cached = await readLocal(db.products, id)
      if (cached) return cached                     // trusted: we parsed it on the way in (§6)
      throw networkError                            // no data at all — surface it
    }
  },
}))
```

In the component: `const { data, error, status } = useQuery(productByIdOptions(id))`.

**What changed from the naive version, and why:**

- **The cache is read in `catch`, not up front.** Reading IndexedDB before every successful online fetch is a wasted round-trip on the hot path.
- **The `navigator.onLine` pre-check is gone.** The `catch` branch already covers offline, DNS failure, 502s and captive portals — one code path instead of two that can disagree.
- **Cached rows are returned, not re-parsed.** They were parsed on the way in.
- **`defineQueryOptions` in `app/queries/`**, per `data-fetching.md` — the key literal exists exactly once.

Tell the user the data is stale. `useOnline()` plus the row's `updatedAt` is enough: *"Offline — showing data from 14:02."*

If the installed Colada version exposes an `enabled` option, skipping this query on the server is cleaner than leaning on the guard alone. **Verify the option exists in the installed version before using it** — do not assume it from this file.

---

## 5. Pattern B — local-first table + outbox

For rows the client creates and owns: drafts, settings, queued actions, and every table in local-first mode.

```ts
// app/database/index.ts
import Dexie, { type Table } from 'dexie'

export interface Task { id: string, title: string, createdAt: string, updatedAt: string }
export interface OutboxEntry {
  id?: number
  op: 'create' | 'update' | 'delete'
  table: string
  payload: unknown
  createdAt: string
  attempts: number
}

class AppDB extends Dexie {
  tasks!: Table<Task, string>
  products!: Table<Product, string>    // the hybrid cache table from §4
  outbox!: Table<OutboxEntry, number>

  constructor() {
    super('app-db')
    this.version(1).stores({
      tasks: 'id, createdAt',          // first field = primary key, the rest are indexes
      products: 'id, updatedAt',
      outbox: '++id, createdAt',
    })
  }
}

export const db = new AppDB()
```

Bind the table straight to the UI with a live query — it re-emits on every write, so there is no manual refetch and no store mirroring:

```ts
// app/composables/useLocalTasks.ts
import { liveQuery } from 'dexie'
import { db, type Task } from '~/database'

export function useLocalTasks() {
  const tasks = ref<Task[]>([])

  onMounted(() => {                                  // client-only by construction (§3)
    const sub = liveQuery(() => db.tasks.orderBy('createdAt').reverse().toArray())
      .subscribe({ next: rows => (tasks.value = rows) })
    onScopeDispose(() => sub.unsubscribe())
  })

  return { tasks: readonly(tasks) }
}
```

`useObservable` from `@vueuse/rxjs` does the same in one line, but it pulls in `rxjs` and needs a cast plus an `initialValue` for Dexie's observable. Use it only where the project already depends on rxjs; the eight lines above add no dependency.

**Writes go to the table and the outbox in one transaction**, so a crash cannot leave a local row that will never sync:

```ts
export async function createTask(input: Pick<Task, 'title'>) {
  const now = new Date().toISOString()
  const task: Task = { id: crypto.randomUUID(), title: input.title, createdAt: now, updatedAt: now }

  await db.transaction('rw', db.tasks, db.outbox, async () => {
    await db.tasks.add(task)
    await db.outbox.add({ op: 'create', table: 'tasks', payload: task, createdAt: now, attempts: 0 })
  })
}
```

Flush the outbox when connectivity returns: oldest first, delete each entry only after the server confirms it, and cap `attempts` so one poison entry cannot loop forever. Server wins on `updatedAt` conflicts unless the user has specified otherwise.

---

## 6. Zod placement — parse on write, `safeParse` on version change

Parsing every row on every read is a tax on data you already validated.

| Boundary | Rule |
| :--- | :--- |
| Network → app | `.parse()` — the payload is untrusted. Failure is a real error. |
| App → Dexie | Already parsed above; do not parse twice. |
| Dexie → app, same schema version | Trust it. No parse. |
| Dexie → app, after a schema version bump | `.safeParse()` per row, **drop** invalid rows, never throw the whole query. |

```ts
const rows = await db.products.toArray()
const valid = rows.flatMap((row) => {
  const parsed = ProductSchema.safeParse(row)
  return parsed.success ? [parsed.data] : []
})
```

Bump `db.version(n)` whenever a stored shape changes, and add the Dexie upgrade function in the same commit as the Zod schema change. A schema edit without a version bump ships silent data corruption to everyone who already holds the old DB.

---

## 7. Smells

| Smell | Why it is wrong |
| :--- | :--- |
| Dexie call with no `import.meta.client` guard | Crashes SSR — see `debugging.md` |
| `if (navigator.onLine)` deciding whether to fetch | Reports the interface, not your API |
| Same table read via `useQuery`, written via raw `db.table.put` | Two sources of truth; the cache goes stale silently |
| `Schema.parse()` on every Dexie read | Re-validating data you wrote and already parsed |
| Dexie schema edited without a `version()` bump | Silent corruption for existing users |
| Server data overwriting a local row with no `updatedAt` comparison | Destroys an unsynced local edit |
| Offline fallback returning `{}` or `0` | Fabricated success — the UI cannot tell it from real data |
| Dexie installed in an online-only app | A second source of truth to maintain forever, for no gain |
