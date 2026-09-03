# Code Structure: Composables, Features & Layers

Paths assume the Nuxt 4 default `srcDir: 'app/'`.

This file answers one question: **where does this file go, and when does the folder shape change?** Read it before creating a layer, before adding a `features/` folder, and before proposing any restructure of `composables/`.

---

## 1. Flat is the default. Defend it.

A flat `app/composables/` folder is the correct shape until it demonstrably hurts. Having read this file is not evidence that it hurts.

**Never propose a restructure the user did not ask for.** A folder reorganisation is a diff across dozens of files that changes no behaviour, invalidates every open branch, and buys nothing a rename could not. Raise it only when the user reports the pain, or when a Tier 2 audit finds a folder past the thresholds in section 4.

Layers are never the answer to "too many files". They answer "a second audience appeared".

---

## 2. `components/` and `composables/` are not symmetrical

This asymmetry is the most common source of wrong advice. They behave differently and get opposite defaults.

| | `app/components/` | `app/composables/` |
| :--- | :--- | :--- |
| Default scan depth | **Recursive** | **Top level only** |
| Config needed to nest | None | `imports.dirs` (section 5) |
| Does nesting namespace it? | **Yes** — `auth/LoginForm.vue` → `<AuthLoginForm>` | **No** — still a global `useAuth()` |
| Cost of one subfolder | Zero | Config, plus silent-breakage risk |
| Default | **Domain folders from day one** | **Flat until section 4 fires** |

Components: follow `components.md`. Nesting is free and the path prefix genuinely disambiguates the tag.

Composables: **auto-import is a single flat global namespace.** `layers/admin/app/composables/useUser.ts` and `layers/client/app/composables/useUser.ts` both register `useUser()`; the higher-priority layer silently wins with no error. Moving a composable into a folder changes the file tree and nothing else — not the call site, not the collision risk.

The namespace lives in the **name**, never in the path.

---

## 3. Is it a composable at all?

Answer this before deciding where it goes. A bloated `composables/` folder is usually 30–40% files that were never composables.

| Test | Destination | Auto-imported |
| :--- | :--- | :--- |
| Uses reactivity, lifecycle, or Nuxt context (`ref`, `watch`, `useState`, `useFetch`, `useRoute`) | `app/composables/` | Vue app only |
| Pure function — same input, same output, no Vue, no context | `app/utils/` | Vue app only |
| Needed by both Nitro and the Vue app (types, validators, pure formatters) | `shared/` | Both contexts |
| Server-only (env access, Prisma, secrets) | `server/utils/` | Server only |

`app/utils/` auto-imports **identically** to `app/composables/` — same mechanism, same top-level-only depth. Moving a pure function between them is a `git mv` that changes **zero call sites**.

Because they share one namespace, a util and a composable can collide with each other. The `use` prefix is reserved for composables: `requiredEnv`, not `useRequiredEnv` (see `smells.md`).

---

## 4. Fixing a bloated flat folder

Cheapest first. Each step must be exhausted before the next is proposed — and steps 1–3 usually mean step 4 never fires.

### Step 1 — Move out what was never a composable

Apply the section 3 table. Pure `git mv`, no call-site changes, no config. Do this first, always: it is the only risk-free step, and it shrinks the problem before you measure it.

### Step 2 — Rename for a domain prefix

The highest-leverage step, and it needs no folders and no config. File explorers sort alphabetically, so the prefix **is** the grouping:

```
# Unreadable — alphabetical order scatters every domain
useAuth.ts  useCart.ts  useLogin.ts  usePayment.ts  useSession.ts

# Groups itself, no folders, no config, and fixes the call site too
useAuthLogin.ts  useAuthSession.ts  useCartItems.ts  useCartTotal.ts
```

Unlike a folder, the prefix also survives to the call site and pre-empts collisions. Cover `utils/` in the same pass — it shares the namespace.

### Step 3 — Merge over-split files

Auto-import works per **export**, not per file. Three files exporting one small function each, never used apart, should be one file exporting three things. Call sites do not change.

```ts
// app/composables/useAuth.ts
export function useAuthSession() { /* ... */ }
export function useAuthLogin() { /* ... */ }
```

### Step 4 — Only now, subfolders

**Trigger:** after steps 1–3, the folder still does not fit one screen in the file explorer — roughly 15–20 files. Below that, subfolders add config for no gain.

Requires the `imports.dirs` config in section 5, in the same commit.

---

## 5. `imports.dirs` — the exact config, and the footgun

Nuxt scans only the **top level** of `app/composables/`. To scan nested folders, all three entries are required:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  imports: {
    dirs: [
      '~/composables',
      '~/composables/*/index.{ts,js,mjs,mts}',
      '~/composables/**',
    ],
  },
})
```

`'~/composables/**'` **alone is not enough** — it misses the `*/index.ts` case.

**The footgun:** unprefixed paths resolve against `srcDir`, which is `app/` in Nuxt 4. Writing `'app/composables/**'` — the reflex — resolves to `app/app/composables/**` and silently imports nothing. Use the `~/` alias and the problem disappears.

**Migration safety.** Moving a composable into a subfolder before this config lands does not fail the build. It throws `useThing is not defined` at runtime, and under SSR that usually reads as a hydration bug. Therefore:

- The config edit ships in the **same commit** as the first file move.
- Restart the dev server — Nuxt caches the scan in `.nuxt/imports.d.ts`.

---

## 6. `features/` inside a single app

When one app has grown distinct functional domains but has no second audience, group by feature *before* reaching for layers:

```
app/
  features/
    inventory/
      components/
      composables/     -> useInventoryStock.ts
    billing/
      components/
      composables/     -> useBillingInvoices.ts
  composables/         -> genuinely cross-feature only
  components/
```

Registering it takes **two** config blocks, not one. Composables alone is the common mistake — it leaves every component in `features/` unimported and unexplained:

```ts
export default defineNuxtConfig({
  imports: {
    dirs: ['~/features/*/composables', '~/features/*/composables/**'],
  },
  components: [
    { path: '~/features/inventory/components', prefix: 'Inventory' },
    { path: '~/features/billing/components', prefix: 'Billing' },
    '~/components',
  ],
})
```

`imports.dirs` accepts globs. **`components[].path` is not documented as glob-aware** — the official examples are all concrete paths. List feature directories explicitly, or build the array in the config. If you try a glob there, verify it resolves before relying on it.

Domain-prefix names still apply inside `features/`. The folder does not namespace the composable (section 2).

---

## 7. Layers: when, and when to object

**Default when creating a layer: macro-layers by audience** — `base`, `auth`, `client`, `admin`.

That axis is the right one because those boundaries are **enforceable**: they line up with the auth boundary, the route boundary, and the rendering strategy (`admin` and `client` are usually `ssr: false` SPAs while marketing prerenders — see `layouts-routing.md`). A `billing` layer has none of those; nothing in the framework holds it to its edges, so it degenerates.

Use `base` as the foundation layer name, not `shared` — Nuxt 4 already has a top-level `shared/` directory with different semantics (section 3), and two meanings of "shared" in one project is a coin flip every time someone reads a path.

**Object, naming which case applies, when:**

1. **No distinct user classes** (marketing site, blog, single-audience tool) — layers add config and buy nothing. Use section 6.
2. **Under ~15 composables, solo developer, no admin surface yet** — sections 4 and 6. Layers are reversible; add them when the second audience actually exists.
3. **The proposed layer is a feature, not an audience** (`billing`, `inventory`, `notifications`) — that is a `features/` folder inside a macro layer. This objection fires most often.
4. **The feature spans two audiences** (billing exists in both client and admin) — it belongs in `base`, not copy-pasted into both. Duplicating across macro layers is the failure that makes teams abandon layers.
5. **It is really a design system for other repositories** — a publishable layer or package is a different axis and is legitimate. Do not fold it into the audience split.

Inside a macro layer, partition by feature exactly as in section 6, configured in **that layer's own** `nuxt.config.ts`.

---

## 8. Layer priority is alphabetical, and it will surprise you

Layers auto-registered from `~~/layers/` are sorted **alphabetically, Z beats A**. Project files always win; layers listed in `extends` rank above unlisted ones.

With the default macro names, the effective priority is:

```
client  >  base  >  auth  >  admin
```

`base` outranks the two layers meant to override it. **The default naming ships broken unless you fix the order.** Two fixes:

```
layers/
  1.base/
  2.auth/
  3.client/
  4.admin/
```

```ts
// nuxt.config.ts — explicit, and documents the dependency direction in one place
export default defineNuxtConfig({
  extends: [
    '~~/layers/admin',   // highest priority
    '~~/layers/client',
    '~~/layers/auth',
    '~~/layers/base',    // lowest — the foundation
  ],
})
```

Prefer the explicit `extends` array. It is the only place a reader can see the whole graph at once.

---

## 9. Layers never import from each other

**A layer may only reach downward, into `base` or another foundation layer. Never sideways, never upward.**

| Tier | Layers | May import from |
| :--- | :--- | :--- |
| Audience | `client`, `admin` | Foundation only |
| Foundation | `auth` | `base` only |
| Foundation | `base` | Nothing |

- `admin` never reaches into `client`, and `client` never reaches into `admin`. Siblings are invisible to each other by design.
- `base` never reaches into `auth`, `client` or `admin`. An upward import is a circular `extends` and it breaks the build.
- When two audience layers need the same thing, **move it down into `base`**. Never import sideways to avoid a move.

### The rule is invisible to a grep for `import`

Once both layers are registered, their composables and components share one global auto-import namespace. `useClientBilling()` is callable from inside `layers/admin/` **with no import statement at all** — the violation leaves no textual trace.

So enforcement is three things, not one:

1. **Grep the explicit forms** — a `#layers/<sibling>` alias, or any relative path climbing out of a layer root (`../../client/`), is an unambiguous violation.
2. **Check definitions before use.** Inside a layer, before calling an auto-imported symbol you did not define there, confirm which layer defines it. If it is a sibling, stop and move the definition to `base`.
3. **Domain-prefix every layer's exports** — `useAdminInvoices`, `useClientBilling`, `useBaseToast`. This is what makes an out-of-layer symbol *visibly* wrong at the call site instead of silently fine. The naming rule in section 4 is load-bearing for architecture here, not just for DX.

Flag any sibling-to-sibling dependency as an architectural finding in a Tier 2 audit, with the move-to-`base` fix named.

---

## 10. Restructures are Tier 2

Steps 1–3 of section 4 across a real folder touch far more than four files, so a restructure is **Tier 2**: announce, scan, write the memory files, wait for "Proceed". Step 1 alone, scoped to two or three files, is Tier 1.

Report the move as a mechanical diff — files moved, files renamed, config changed, behaviour unchanged — so the user can review it as one shape rather than a hundred edits.
