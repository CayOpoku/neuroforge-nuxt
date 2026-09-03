# Layouts, Routing & Page Architecture

Paths assume the Nuxt 4 default `srcDir: 'app/'`.

---

## 1. Essential SaaS layouts

Layouts are a primary architectural decision, not an afterthought. A SaaS app needs these in `app/layouts/` before feature work starts:

1. **`default.vue`** — marketing/public shell: navbar, footer.
2. **`auth.vue`** — focused, distraction-free shell for login, register, reset, verify.
3. **`dashboard.vue`** — app shell: collapsible sidebar, user header, breadcrumbs.

Plus `app/error.vue` at the root (not in `layouts/`) — it catches fatal errors and 404s:

```vue
<script setup lang="ts">
const props = defineProps<{ error: NuxtError }>()
const isNotFound = computed(() => props.error.statusCode === 404)
</script>
```

Show the real status and message. An error page that says "Something went wrong" for every failure is the false-fallback smell at page scale.

---

## 2. Separation of concerns

- **Pages (`app/pages/`)** are thin route entry points. They set the layout, read route params, orchestrate a fetch, and compose components. A page holding presentation markup or business logic is doing two jobs.
- **Layouts (`app/layouts/`)** own persistent structural chrome — sidebars, navbars, shell grids. They persist across navigation, so state in a layout survives a page change (useful for a sidebar toggle, dangerous for anything page-specific).
- **Components (`app/components/`)** hold the visuals, in domain folders, never bound to a page name.

```vue
<script setup lang="ts">
definePageMeta({
  layout: 'dashboard',
  middleware: 'auth',
})
</script>
```

---

## 3. Routing rules

- Route params are untrusted input. Validate before use — a `[id].vue` that passes `route.params.id` straight into a fetch will happily request `/api/orders/undefined`.
- Protect routes with named middleware (`auth-middleware.md`), not with a `v-if` in the template.
- Set the rendering strategy per route group in `routeRules` (`performance-a11y.md`). Marketing prerenders; a dashboard behind auth usually does not need SSR at all.
- Group related routes in a folder with a shared layout rather than repeating `definePageMeta` in every file.
- Use `<NuxtLink>` for internal navigation — `<a href>` triggers a full page reload and discards the SPA state.
