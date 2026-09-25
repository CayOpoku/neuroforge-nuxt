# SSR, Diagnostics, Logging & Debugging Protocol

Use this reference to diagnose runtime issues, design logs, and address hydration mismatches.

---

## 1. Console Log Debugging Workflow

When investigating bugs, unexpected behaviors, or data flow issues:
1. **Lean on `console.log` Diagnostics**: Instead of burning context tokens guessing solutions or making speculative changes, insert targeted `console.log` statements to observe actual runtime values and execution paths.
2. **Clean Up Log Statements**: Once the root cause is identified and resolved, remove temporary debugging `console.log` calls to maintain clean code.

### Why frontend bugs cost more — and the fix

A Nitro bug has ground truth you can reach alone: throw, log, typecheck, read the terminal. **A frontend bug's ground truth is in the developer's browser, on their screen.** You cannot see it. The failure mode is substituting the only thing you *can* do alone — reading more files, guessing wider — for the one observation that would settle it.

So make the developer the instrument. They are sitting in front of the running app:

1. **Write the log, don't hunt for the answer.** One or two labelled `console.log`s at the exact point where the value should be right.
2. **Tell them precisely what to do** — which page, which click, which tab. *"Load `/dashboard`, click Save, paste what `DEBUG_SAVE:` prints."*
3. **Stop and wait.** Do not read more files while waiting. Do not ship a speculative fix "in the meantime".
4. **Read the output, then decide.** Real values beat five more file reads every time.

**Before a second fix attempt, you must have an observation** — a log, an error, something they saw. Two fixes for one symptom with no new evidence between them means stop guessing and instrument (`SKILL.md` → Diagnose mode).

Ask for what only they can see: the exact error text, what the network tab shows, whether the value is wrong or missing, whether it ever worked, what changed since it did.

---

## 2. Explicit Confirmation for Diagnostics & Linting Commands

- **Primary Lint Command**: Use `npm run lint` as the primary linting and static audit command. Check if a `"lint"` script exists in `package.json`; if missing, create one (e.g. `"lint": "eslint ."` or `"lint": "nuxi typecheck"`).
- **`npx nuxi typecheck`**: Use `npx nuxi typecheck` when specific type safety verification is needed, but default to `npm run lint`.
- **Inform User Before Running Checks**: Always notify and request permission from the user before executing `npm run lint` or `npx nuxi typecheck`.
- **Zero `any` Policy**: Never use `any` as a type escape hatch. Use `unknown` and narrow it — see `type-safety.md`.

### Reading a hydration warning

Vue reports the *symptom node*, not the cause. Work backwards:
1. Note the mismatched element in the console warning.
2. Find what feeds it — a prop, a `computed`, a store value.
3. Ask what could differ between server and client for that value: time, randomness, `window`, `localStorage`, a locale-dependent format, or a value the server never had.
4. Fix the source with the table in section 5. Never silence the symptom by wrapping the node in `<ClientOnly>` — that trades a warning for a blank flash and a lost SSR payload.

---

## 3. SSR / Client Context Rules

Before fixing any runtime error or using a browser API, verify the execution environment:
* `import.meta.server` — Nitro/SSR context (no `window`, `document`, or `localStorage`).
* `import.meta.client` — Browser execution only.

Always prefix your logs so they can be easily filtered in terminal logs:
```ts
const prefix = import.meta.server ? "[SERVER]" : "[CLIENT]";
console.log(`${prefix} User State:`, user.value);
```

---

## 4. Structured Logging Rules (Agent-Optimised)

Always use labelled, structured logs for instant searchability and grepping:
```ts
// ❌ BAD - Bare output, hard to scan
console.log(data)

// ✅ GOOD - Instant grepping and identification
console.log('DEBUG_DATA_FETCH:', { payload: data, timestamp: Date.now() })

// ✅ GOOD - For listing tabular rows
console.table(items)

// ✅ GOOD - For deeply nested server objects
console.log(JSON.stringify(serverData, null, 2))

// ✅ GOOD - Performance tracing
console.time('fetch-orders')
const orders = await db.order.findMany(...)
console.timeEnd('fetch-orders')
```

---

## 5. Hydration Mismatch Protocol

Never ignore Vue hydration warnings. Mismatches degrade SEO, disable interactivity, and force full re-renders.

| Problem | Wrong | Right |
| :--- | :--- | :--- |
| **Browser-only API** | `localStorage.getItem('theme')` | `useCookie('theme', { default: () => 'light' })` |
| **Inconsistent state** | `Math.random()` in template | `useState('key', () => Math.random())` |
| **Client-only condition** | `v-if="window?.innerWidth > 768"` | CSS media queries or `<ClientOnly>` |
| **Time-based content** | `new Date().getHours()` in setup | `<NuxtTime>` component or `onMounted` + `<ClientOnly>` |
| **Browser-only 3rd party lib** | Init in `setup()` | Init in `onMounted()` |

---

## 6. Error Handling & Boundaries

### API Routes (Nitro)
```ts
// ✅ Always use createError — never return raw strings
throw createError({
  statusCode: 400,
  statusMessage: "Validation failed",
  data: { field: "email", reason: "Invalid format" },
});

// Fatal errors (trigger error.vue)
throw createError({
  statusCode: 500,
  statusMessage: "DB unavailable",
  fatal: true,
});
```

### UI-Level Failures (NuxtErrorBoundary)
```html
<NuxtErrorBoundary>
  <MyRiskyComponent />
  <template #error="{ error, clearError }">
    <p>{{ error.message }}</p>
    <button @click="clearError({ redirect: '/' })">Go home</button>
  </template>
</NuxtErrorBoundary>
```

---

## 7. Works locally, broken in production

The dev server never serves stale bundles, never lacks an env var you set in `.env`, and never runs the build CI produced. So a production-only failure is usually one of three things: the deployed build, the host environment, or the visitor's cache. Read the signature before reading code.

| Signature | Meaning | Cheapest separator |
| :--- | :--- | :--- |
| Submitting a form reloads the page and the URL gains `?field=…`; no request in the Network tab | **The page never hydrated.** `@submit.prevent` is JavaScript, so without it the browser does a native GET submit. Only inputs with a `name` appear in the query, which is often just the honeypot. | Console tab → the **first** red error. Everything after it is fallout. |
| `SyntaxError: The requested module './A.js' does not provide an export named 'x'` | **Stale chunk mix:** the browser combined a cached chunk from an earlier deploy with one from the current deploy. | Same page in **Incognito**. It works there → browser cache. It fails there too → server/CDN cache or a broken build. |
| A third-party script "was preloaded … but not used within a few seconds" | Almost always **collateral**: the app crashed before its trigger (`onNuxtReady`, consent) fired. | Fix the first error; this warning goes with it. |
| New env var has no effect | Host env is read **at process start** (`smells.md` §3). | Restart the app and check again. |
| New code has no effect | The deploy didn't run, or didn't go green. | CI run status for the merge commit, before any code reading. |

**Restart ≠ redeploy.** A restart relaunches the **same build** with a fresh environment. A deploy uploads a **new build**, then restarts. An env change needs a restart. A code or `nuxt.config.ts` change needs a deploy. Say which one applies in one line; developers routinely suspect the wrong one.

### Stale chunks after a deploy

`/_nuxt/*` is served `immutable` with a year's `max-age`, which is correct, because the filename is supposed to be the content hash. Observed on Nuxt 4.5.2 / Vite 8.2.1 / Rolldown 1.2.4: a chunk kept its filename across deploys while its exports changed, so returning visitors crashed and first-time visitors didn't. The root cause in the bundler isn't traced yet. Don't claim one.

**Why Nuxt doesn't self-heal:** its chunk-error reload fires when a chunk **fails to load**. Here the chunk loads fine and then fails to link, so nothing triggers a reload.

**Confirm on the server, not by theory:** fetch both chunks named in the error directly (`curl -sS https://site/_nuxt/A.js`). Check that the importer's `import{x as …}from"./B.js"` matches B's final `export{…}`. They match on the server → the stale copy lives in the browser.

**Fix for every visitor** (after confirming): a fresh asset URL per deploy, so no cached copy can ever be combined with new code.

```ts
// nuxt.config.ts
app: {
  // new asset URL per deploy, so a cached chunk can never meet a newer one
  buildAssetsDir: process.env.GITHUB_SHA ? `/_nuxt/${process.env.GITHUB_SHA.slice(0, 8)}/` : '/_nuxt/',
},
```

- `GITHUB_SHA` is set automatically in every GitHub Actions step. On another CI, use its commit variable. Local builds keep the default.
- **Prerequisite: HTML must not be cached.** Check with `curl -sSI https://site/some-page` and look for `cache-control`. If pages are cached, stale HTML will point at asset folders the deploy has deleted.
- Cost: returning visitors re-download scripts after each deploy. That's negligible for a marketing site. Mention it anyway.
- For the developer's own browser right now: DevTools → Application → **Clear site data**.
