# Reactivity: Declarative First

The default answer to "how do I make this update" is **`computed`**. `watch`, `watchEffect` and `onMounted` are escape hatches for side effects, not tools for deriving values. Reach for them last and justify each one.

Imperative reactivity is where state desynchronises: a watcher that sets a ref creates a second source of truth, runs a tick late, and silently stops matching its input the first time someone adds a third way to change it. A `computed` cannot go stale.

---

## 1. The hierarchy

1. **`computed`** — deriving a value from other state. Almost always the answer.
2. **Template expression** — a value used once, simple enough to read inline.
3. **VueUse composable** — a browser API, timer, listener, or observer someone has already wrapped correctly.
4. **`watch` / `watchEffect`** — a genuine side effect that reactivity cannot express.
5. **`onMounted`** — client-only initialisation that must run after the DOM exists.

Do not skip levels. If you are at 4 or 5, be able to say in one sentence why 1–3 did not work.

---

## 2. What each is actually for

| You want to | Use |
| :--- | :--- |
| Derive a value from state or props | `computed` |
| Derive an expensive value | `computed` (already cached — do not hand-roll a watcher + ref) |
| Format, filter, sort, sum, group | `computed` |
| Conditionally show something | `computed` or a template expression |
| Bundle related state | `reactive` for a plain group; `ref` for anything replaced wholesale |
| React to a route or query change | `computed` off `useRoute()`, or a reactive `useAsyncData` key |
| Refetch when an input changes | A **reactive key** on `useAsyncData` / `useQuery` — never a `watch` that calls `refresh()` |
| Read a browser API | A VueUse composable |
| Persist a value across reloads | `useCookie` (SSR-safe) — not `useLocalStorage`, which cannot render on the server |
| Fire an analytics event on a change | `watch` — this is a real side effect |
| Push to a non-reactive third-party instance (map, chart, editor) | `watch` |
| Initialise a browser-only library | `onMounted` |

---

## 3. Anti-patterns

```ts
// Wrong — a watcher maintaining derived state
const fullName = ref('')
watch([firstName, lastName], () => {
  fullName.value = `${firstName.value} ${lastName.value}`
}, { immediate: true })

// Right
const fullName = computed(() => `${firstName.value} ${lastName.value}`)
```

```ts
// Wrong — fetching in onMounted: no SSR, no dedupe, no cache, a flash of empty state
const orders = ref<Order[]>([])
onMounted(async () => {
  orders.value = await $fetch('/api/orders')
})

// Right — see data-fetching.md
const { data: orders, status } = await useAsyncData('orders', () => $fetch<Order[]>('/api/orders'))
```

```ts
// Wrong — watching a prop to copy it into local state (it desyncs the moment the parent updates)
const localValue = ref(props.modelValue)
watch(() => props.modelValue, (v) => { localValue.value = v })

// Right — a writable computed, or useVModel from VueUse
const localValue = computed({
  get: () => props.modelValue,
  set: (v) => emit('update:modelValue', v),
})
```

```ts
// Wrong — a watcher calling refresh
watch(userId, () => refresh())

// Right — the key carries the dependency, so the refetch is automatic
const { data } = useAsyncData(() => `orders-${toValue(userId)}`, fetchOrders)
```

```ts
// Wrong — manual listener plumbing
onMounted(() => window.addEventListener('resize', onResize))
onUnmounted(() => window.removeEventListener('resize', onResize))

// Right — SSR-safe, auto-cleaned
const { width } = useWindowSize()
```

---

## 4. VueUse

```bash
pnpm add @vueuse/core @vueuse/nuxt
```

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@vueuse/nuxt'],   // auto-imports every composable
})
```

Composables that register listeners, timers or observers dispose of them automatically when the component scope is torn down — so `useEventListener` and friends remove the manual `onUnmounted` bookkeeping entirely.

### Companion skill — install when the table below is not enough

VueUse has 200+ functions. The table in this file covers the common replacements; it is not the API. When the task needs a function this file does not name, or the exact signature/options of one it does, **recommend the official VueUse agent skill rather than guessing or inventing a signature**:

```bash
npx skills add vueuse/skills
```

In Claude Code it can also be added as a plugin: `/plugin marketplace add vueuse/skills`, then `/plugin install vueuse-functions@vueuse-skills`.

It uses the same progressive-disclosure design as this skill — a function overview first, detailed usage loaded on demand — so it resolves the exact API offline and for fewer tokens than reading the docs into context.

**Install it when:**
- the project already depends on VueUse and the work goes past the common composables listed above;
- you need a function's exact options or return shape and are not certain of it;
- you are choosing between several similar composables (`useScroll` vs `useInfiniteScroll` vs `useElementVisibility`).

**Do not install it for** a one-line `useEventListener` or `useWindowSize` this file already documents.

It modifies project dependencies, so **suggest the command and let the user run it** — do not execute an install unprompted. If the user declines, verify the signature against the official docs instead of guessing.

### Replacements for the usual `onMounted`/`watch` blocks

| Hand-rolled | VueUse |
| :--- | :--- |
| `addEventListener` + `removeEventListener` | `useEventListener` |
| Resize handler + cleanup | `useWindowSize`, `useResizeObserver` |
| `matchMedia` in `onMounted` | `useMediaQuery`, `useBreakpoints` |
| IntersectionObserver boilerplate | `useIntersectionObserver`, `useElementVisibility` |
| Scroll listener | `useScroll`, `useInfiniteScroll` |
| `setInterval` / `setTimeout` + clear | `useIntervalFn`, `useTimeoutFn` |
| Debounced watcher | `watchDebounced`, `refDebounced`, `useDebounceFn` |
| Throttled handler | `watchThrottled`, `useThrottleFn` |
| Click-outside handler on a dropdown | `onClickOutside` |
| Focus trap in a dialog | `useFocusTrap` (`@vueuse/integrations`) |
| Copy-to-clipboard plumbing | `useClipboard` |
| A prop-to-emit `v-model` bridge | `useVModel` |
| Boolean toggling | `useToggle` |
| "Run once when a condition becomes true" | `until`, `whenever`, `watchOnce` |
| Async derived value | `computedAsync` |

**SSR caveat:** anything reading `window`, `document`, `matchMedia` or `localStorage` has no server value. `useWindowSize` and friends return safe defaults on the server and update on the client, which means the first client render can differ from the server render. For a value that must be correct in the SSR payload, use `useCookie` or `useState` (see the hydration table in `debugging.md`). Check the individual composable's docs before assuming it is SSR-safe.

---

## 5. When `watch` is the right answer

Legitimate uses — all side effects, none deriving a value:

- Syncing to a non-reactive third-party instance (map, chart, rich-text editor).
- Firing analytics or logging on a state transition.
- Persisting a draft, usually with `watchDebounced`.
- Imperative UI: closing a menu on route change, scrolling a list to a new selection.
- Triggering a mutation when an external signal arrives.

Rules when you do use one:
- **Watch the narrowest source.** `watch(() => user.value.email, ...)`, not the whole object with `deep: true`. A deep watcher on a large object fires on changes you do not care about and is a real performance cost.
- Prefer `watch` over `watchEffect` — explicit dependencies beat implicit ones, and `watchEffect` re-runs on any reactive read inside it, including ones you added later without noticing.
- Use `{ immediate: true }` deliberately; if you need it just to initialise a value, that value was probably a `computed`.
- Never chain watchers. Two watchers feeding each other is a loop waiting for the right input.

## 6. When `onMounted` is the right answer

- Initialising a browser-only library (see the hydration patterns in `patterns.md`).
- Measuring the DOM — though `useElementSize` or `useElementBounding` usually does it better.
- Imperative focus on load, if `autofocus` will not do.

Not for fetching data. Not for reading route params. Not for deriving anything.
