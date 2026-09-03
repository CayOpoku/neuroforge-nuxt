# Performance, Accessibility & SEO

---

## 1. Images

Never ship a raw `<img>` for content imagery. Use `@nuxt/image`:

```vue
<nuxt-img
  src="/hero.jpg"
  alt="Team reviewing the analytics dashboard"
  width="1200" height="630"
  sizes="sm:100vw md:50vw lg:1200px"
  format="webp"
  loading="lazy"
  placeholder
/>
```

- **Always set `width`/`height`** (or an aspect-ratio class) — missing dimensions cause layout shift, which is a Core Web Vitals penalty and visible jank.
- `loading="lazy"` on everything below the fold; **omit it and preload the LCP image** — lazy-loading the hero delays the largest paint.
- `alt` describes the content's purpose. Decorative images get `alt=""`, never a missing attribute.

---

## 2. Lazy hydration

Ship less JavaScript on first paint. Nuxt hydration strategies require the `Lazy` prefix:

```vue
<lazy-analytics-chart hydrate-on-visible />
<lazy-comment-thread hydrate-on-idle />
<lazy-share-menu hydrate-on-interaction="click" />
<lazy-cookie-banner :hydrate-after="3000" />
<lazy-heavy-widget :hydrate-when="isPremium" />
<lazy-static-footer hydrate-never />
```

Strategies: `hydrate-on-visible`, `hydrate-on-idle`, `hydrate-on-interaction`, `hydrate-after`, `hydrate-when`, `hydrate-never`.

Apply to anything below the fold, behind a tab, or inside a modal. A dashboard that hydrates six charts on load is doing five charts' worth of unnecessary work.

`<ClientOnly>` is **not** a performance tool — it disables SSR for that subtree, costing SEO and adding a paint delay. Use it only for genuinely browser-only widgets, and always supply a `#fallback` to hold the layout.

---

## 3. Rendering strategy

Set it per route in `nuxt.config.ts` — SSR-everything is rarely right for a SaaS app:

```ts
routeRules: {
  '/': { prerender: true },
  '/pricing': { prerender: true },
  '/blog/**': { isr: 3600 },
  '/app/**': { ssr: false },
  '/api/**': { cors: true },
}
```

Marketing pages prerender. Dashboards behind auth have no SEO value — turning SSR off there removes a whole class of hydration bugs and cuts server cost.

---

## 4. Payload discipline

- `transform` / `pick` in `useAsyncData` before the data is serialised into the HTML payload. Fetching 80 columns to render 4 ships all 80 to the browser.
- Precise Prisma `select` — the server-side half of the same rule.
- **Paginate every list.** Cursor pagination for infinite scroll, offset for numbered pages:

```ts
const orders = await db.order.findMany({
  where: { userId },
  select: { id: true, status: true, total: true, createdAt: true },
  orderBy: { createdAt: 'desc' },
  take: limit + 1,
  ...(cursor && { cursor: { id: cursor }, skip: 1 }),
})
```

Taking `limit + 1` rows is how you know whether a next page exists without a second count query.

- Watch for N+1: a `findMany` followed by a per-row lookup. Use `include`, or one `where: { id: { in: ids } }` query.
- Check the bundle before shipping a new dependency: `npx nuxi analyze`. A date library or icon set imported wholesale is the usual culprit.

---

## 5. Accessibility

Non-negotiable for a product people pay for, and cheap while writing the component rather than after.

- **Semantic elements first.** A `<div @click>` is not a button: no keyboard, no focus, no screen-reader role. Use `<button>`, `<a>`, `<nav>`, `<main>`, `<dialog>`.
- **Every interactive element must be keyboard-operable** — Tab to reach, Enter/Space to activate, Escape to dismiss.
- **Never remove focus outlines.** Restyle them: `focus-visible:ring-2 focus-visible:ring-offset-2`.
- **Focus management in overlays:** focus moves into a dialog on open, is trapped while open, returns to the trigger on close. Shadcn primitives handle this — one more reason not to hand-roll a modal.
- **Label every input.** A `<label for>` or `aria-label`. Placeholder text is not a label.
- **Announce async state.** Errors and toasts need `role="alert"`; live regions need `aria-live="polite"`. A silent failure is invisible to a screen reader user.
- **Contrast:** 4.5:1 for body text, 3:1 for large text and UI boundaries. Grey-on-grey secondary text is the usual failure.
- **Never encode meaning in colour alone** — a red border needs an accompanying message.
- **Respect `prefers-reduced-motion`** for transform and parallax animation.
- Icon-only buttons need an `aria-label`.

---

## 6. SEO

For every public page:

```ts
useSeoMeta({
  title: 'Pricing — Acme',
  description: 'Simple per-seat pricing. No setup fees.',
  ogTitle: 'Pricing — Acme',
  ogDescription: 'Simple per-seat pricing. No setup fees.',
  ogImage: 'https://acme.com/og/pricing.png',
  twitterCard: 'summary_large_image',
})
```

- One `<h1>` per page, headings in order — never skip a level for styling.
- Set `<html lang>` via `useHead({ htmlAttrs: { lang: 'en' } })` in the default layout.
- Pages behind auth need nothing beyond a title. Do not spend effort there.
