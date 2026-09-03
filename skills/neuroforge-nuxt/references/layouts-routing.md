# Layouts, Routing & Page Architecture

---

## 1. Pre-Building Essential SaaS Layouts

Layouts are a **primary architectural necessity**, not a secondary decision. When setting up or auditing a SaaS application, ensure foundational layouts exist in `layouts/`:

1. **`layouts/default.vue`**: Standard application shell with global navbar/footer.
2. **`layouts/auth.vue`**: Clean, focused layout for Login, Register, Password Reset, and Verification pages.
3. **`layouts/dashboard.vue`**: Responsive app dashboard shell with collapsible sidebar, user header, and breadcrumbs.

---

## 2. Page & Layout Separation Rules

- **Pages (`pages/`)**: Pages must act solely as thin route entry points and data fetch orchestrators. They assign layouts via `definePageMeta({ layout: 'dashboard' })`.
- **Layouts (`layouts/`)**: Layouts manage persistent structural UI elements (sidebars, navbars, shell grid containers).
- **Components (`components/`)**: Visual elements, widgets, and features should be placed inside `components/` using domain-driven folders (e.g. `components/app/`, `components/content/`, `components/datatable/`), **never tight-coupled to specific pages**.
