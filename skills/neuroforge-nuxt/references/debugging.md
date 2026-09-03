# SSR, Diagnostics, Logging & Debugging Protocol

Use this reference to diagnose runtime issues, design logs, and address hydration mismatches.

---

## 1. Console Log Debugging Workflow

When investigating bugs, unexpected behaviors, or data flow issues:
1. **Lean on `console.log` Diagnostics**: Instead of burning context tokens guessing solutions or making speculative changes, insert targeted `console.log` statements to observe actual runtime values and execution paths.
2. **Clean Up Log Statements**: Once the root cause is identified and resolved, remove temporary debugging `console.log` calls to maintain clean code.

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
