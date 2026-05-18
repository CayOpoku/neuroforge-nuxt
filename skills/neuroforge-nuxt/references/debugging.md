# SSR, Client Casing, Hydration & Diagnostics Protocol

Use this reference to diagnose runtime issues, design logs, and address hydration mismatches.

---

## 1. SSR / Client Context Rules
Before fixing any runtime error or using a browser API, verify the execution environment:
*   `import.meta.server` — Nitro/SSR context (no `window`, `document`, or `localStorage`).
*   `import.meta.client` — Browser execution only.

Always prefix your logs so they can be easily filtered in terminal logs:
```ts
const prefix = import.meta.server ? "[SERVER]" : "[CLIENT]";
console.log(`${prefix} User State:`, user.value);
```

---

## 2. Structured Logging Rules (Agent-Optimised)
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

## 3. Hydration Mismatch Protocol
Never ignore Vue hydration warnings. Mismatches degrade SEO, disable interactivity, and force full re-renders.

| Problem | Wrong | Right |
| :--- | :--- | :--- |
| **Browser-only API** | `localStorage.getItem('theme')` | `useCookie('theme', { default: () => 'light' })` |
| **Inconsistent state** | `Math.random()` in template | `useState('key', () => Math.random())` |
| **Client-only condition** | `v-if="window?.innerWidth > 768"` | CSS media queries or `<ClientOnly>` |
| **Time-based content** | `new Date().getHours()` in setup | `<NuxtTime>` component or `onMounted` + `<ClientOnly>` |
| **Browser-only 3rd party lib** | Init in `setup()` | Init in `onMounted()` |

---

## 4. Error Handling & Boundaries

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
