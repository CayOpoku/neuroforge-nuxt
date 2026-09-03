# Nuxt ⇄ Strapi: Pages, Dynamic Zones & Preview

The frontend half of `strapi-backend.md`. Read before building a page against Strapi, adding a block, wiring preview, or debugging a blank/404 page. Assumes `data-fetching.md` for the fetching rules this file builds on.

---

## 1. Mode gate — resolve this before creating a single file

Two architectures. They are not a progression, and mixing them is how a marketing site ends up with three ways to render a hero.

| | **Mode S — single types** | **Mode C — collection + dynamic zone** |
| :--- | :--- | :--- |
| Strapi shape | One single type per page (`homepage`, `about-page`, `contact`) | One `pages` collection, each entry a `dynamic_zone` of components |
| Nuxt shape | One `.vue` per page, fixed layout | `pages/[...slug].vue` resolving blocks at runtime |
| Editors can | Edit copy and images in a fixed layout | Compose new pages from blocks without a deploy |
| Cost | Near zero architecture | Registry, block boundaries, populate strategy, lint rules |
| Correct when | The page set is small, fixed, and designed per-page | Editors add pages, or layouts repeat across pages |

**Mode S is the default.** A six-page brochure site of single types needs no registry, no `defineBlock`, no `import.meta.glob` — a page component and one typed fetch each. Building §3–§6 for it is the overengineering this skill exists to prevent.

**Escalate to Mode C when the answer to "can an editor publish a new page without a developer?" must be yes**, or when the same section appears on several pages with different content. That is a real requirement, and once it exists the block architecture is the cheapest correct answer.

Record the mode in `neuroforge/00-project-overview.md` alongside Offline Mode. **Per-route exceptions are normal**: `/blog/[slug].vue` rendering an `article` collection with a fixed layout is Mode S inside a Mode C project — a collection type does not oblige you to a dynamic zone.

---

## 2. Mode S — single-type pages, kept boring

```vue
<!-- app/pages/about.vue -->
<script setup lang="ts">
const { findOne } = useStrapi()

const { data: page } = await useAsyncData('about-page', () =>
  findOne<AboutPage>('about-page', undefined, { populate: { hero: { populate: '*' } } }),
)

useStrapiSeo(() => page.value?.data?.seo)
</script>
```

- One key per page, a literal — nothing reactive, so nothing to compose.
- The populate map lives **next to the page that needs it**, not in a shared `POPULATION_MAPS` object. There is no drift to prevent when there is one consumer.
- Type from the generated Strapi types (`strapi-backend.md` §7), never a hand-written mirror.
- No registry, no `__component` switch, no `defineAsyncComponent`. Sections are ordinary imported components with typed props.

Everything below this line is Mode C.

---

## 3. Mode C — the decentralised block layout

**One block = one directory.** Adding a block touches exactly one folder; the page, the registry and the populate map update themselves.

```
app/
├── components/
│   ├── blocks/                      # Strapi-driven layout blocks
│   │   ├── hero/
│   │   │   └── typing-collage/
│   │   │       ├── index.vue        # layout only
│   │   │       └── index.config.ts  # Strapi UID + populate
│   │   └── content/
│   │       └── feature-grid/
│   │           ├── index.vue
│   │           └── index.config.ts
│   ├── app/                         # Shadcn wrappers (components.md §3)
│   └── ui/                          # Shadcn primitives — CLI-generated; never edit or hand-write
├── pages/
│   └── [...slug].vue                # orchestrator, resolves blocks
├── composables/
│   ├── useStrapiPage.ts
│   └── useGlobal.ts
├── utils/
│   ├── block-registry.ts            # glob resolver: components + populate
│   └── strapi-schema.ts             # locale map, shared populate fragments
└── types/
    ├── blocks.types.ts
    ├── strapi-page.types.ts
    ├── strapi-query.types.ts
    └── global.types.ts
```

### Import boundaries

| From | May import |
| :--- | :--- |
| `blocks/<cat>/<name>/*.vue` | `utils/`, `types/`, `components/ui/`, `components/app/`, its own subfiles |
| `blocks/<cat>/<name>/index.config.ts` | `utils/block-registry`, `utils/strapi-schema`, `types/` — **nothing else** |
| `components/ui/*` | external packages only |

**A block never imports another block.** Shared markup is extracted to `components/app/` or to the block's own subfolder — a block importing a sibling recreates the coupling the registry exists to remove.

**A config file never imports a component or a page.** It is loaded eagerly at startup for every block in the project; an import chain from a config into a `.vue` file pulls the entire block tree into the initial bundle and kills the code splitting in §4.

Enforce with `eslint-plugin-boundaries` — the rule table above is documentation until the linter agrees with it, and a boundary nobody checks is a boundary that is already broken.

### Naming

| Thing | Convention | Example |
| :--- | :--- | :--- |
| Block directory | `kebab-case` under a category | `hero/typing-collage/` |
| Block component | `index.vue` | `index.vue` |
| Block config | `index.config.ts` | `index.config.ts` |
| Custom component outside `ui/` | `kebab-case` | `featured-posts.vue` |
| Composable | camelCase, `use` prefix | `useStrapiPage.ts` |
| Types file | `kebab-case.types.ts` | `blocks.types.ts` |
| Utility module | `kebab-case` | `block-registry.ts` |

Category folders mirror Strapi's component categories (`hero.typing-collage` → `blocks/hero/typing-collage/`). Keeping the two aligned means the UID in an error message tells you the folder without a lookup.

---

## 4. The registry — one glob, two maps

The registry's job is not just to find components. **It must also produce the populate map**, because a hand-maintained `POPULATION_MAPS.page` is the one file everyone forgets — and a missing populate entry is invisible: the block renders, with empty fields, on production.

```ts
// app/utils/block-registry.ts
import type { Component } from 'vue'

export interface BlockConfig {
  /** Strapi component UID, exactly as it appears in `__component` */
  component: string
  /** Populate tree for this block's own fields. Omit for a block with no relations. */
  populate?: Record<string, unknown>
}

export function defineBlock(config: BlockConfig): BlockConfig {
  return config
}

const configs = import.meta.glob<{ default: BlockConfig }>(
  '../components/blocks/*/*/index.config.ts',
  { eager: true },
)
const components = import.meta.glob<{ default: Component }>(
  '../components/blocks/*/*/index.vue',
)

export const blockComponentMap: Record<string, Component> = {}
const blockPopulate: Record<string, unknown> = {}

for (const [path, mod] of Object.entries(configs)) {
  const { component: uid, populate } = mod.default
  const loader = components[path.replace('index.config.ts', 'index.vue')]

  if (!loader) {
    console.warn(`[blocks] ${uid} has a config but no index.vue — skipped`)
    continue
  }

  blockComponentMap[uid] = defineAsyncComponent(loader)
  blockPopulate[uid] = populate ? { populate } : true
}

/** Strapi v5 dynamic-zone population: one `on` entry per registered block. */
export const dynamicZonePopulate = { on: blockPopulate }
```

```ts
// app/components/blocks/hero/typing-collage/index.config.ts
import { defineBlock } from '~/utils/block-registry'

export default defineBlock({
  component: 'hero.typing-collage',
  populate: {
    images: { populate: '*' },
    cta: true,
  },
})
```

- **Configs eager, components lazy.** Configs are tiny and needed before the first fetch; components are code-split so a page ships only the blocks it renders. This is exactly why a config may not import its own component.
- **`import.meta.glob` needs a literal path.** No variables, no aliases inside the pattern — it is compile-time, and a dynamic pattern silently matches nothing.
- **The UID lives in the config, not in a filename convention.** Deriving `hero.typing-collage` from the folder path works right up to the first Strapi component that does not match its folder, and then fails at runtime with an empty page.
- **Verify each UID against `components.d.ts`** (`strapi-backend.md` §7) before shipping the config. A typo here is not a type error.

Then the page populate is composed, never typed by hand:

```ts
// app/utils/strapi-schema.ts
import { dynamicZonePopulate } from './block-registry'

export const POPULATION_MAPS = {
  page: {
    dynamic_zone: dynamicZonePopulate,
    seo: { populate: '*' },
  },
} as const
```

Adding a block is now: create the folder, write `index.vue` and `index.config.ts`. Nothing else in the app changes.

---

## 5. `[...slug].vue` — an orchestrator, and nothing else

The catch-all page resolves blocks, handles the four states, and returns a **real** 404. It holds no layout, no fetching logic, no error plumbing.

```vue
<template>
  <div>
    <component
      :is="blockComponentMap[block.__component]"
      v-for="block in dynamicZone"
      :key="block.id"
      :data="block"
    />
  </div>
</template>

<script setup lang="ts">
import { blockComponentMap } from '~/utils/block-registry'

const route = useRoute()
const slug = computed(() => {
  const param = route.params.slug
  return Array.isArray(param) ? param.join('/') : param || ''
})

const { dynamicZone, pageContent, error } = await useStrapiPage(slug)

// A missing page is a 404, not a component. `fatal` renders error.vue with the real status.
if (!pageContent.value) {
  throw createError({
    statusCode: 404,
    statusMessage: 'Page not found',
    fatal: true,
  })
}
if (error.value) {
  throw createError({ statusCode: 502, statusMessage: 'CMS unavailable', fatal: true })
}
</script>
```

**Why blocking (`await`, not `lazy: true`) here specifically.** `data-fetching.md` makes non-blocking the default posture; a public, indexable catch-all route is the documented exception. The status code must be decided before the response headers are sent, and a lazy fetch has already committed `200 OK` by the time the data arrives.

**Rendering an in-page 404 component is a soft 404** — the crawler gets `200` and indexes an error screen. If the design calls for a branded not-found page, put it in `error.vue` (`layouts-routing.md`) and let `createError` route to it. Same rule as `backend-errors.md` §5: never return a success shape for a failure.

**Unknown `__component`.** A block in the payload with no entry in the registry means the deploy is behind the CMS. Do not render a red debug box in production. Dev-only:

```vue
<component :is="blockComponentMap[block.__component]" v-if="blockComponentMap[block.__component]" :data="block" />
<app-block-missing v-else-if="import.meta.dev" :uid="block.__component" />
```

In production the unknown block renders nothing — a missing section beats a broken-looking page, and the gap is caught by monitoring, not by the visitor.

**Never write error payloads into a cookie.** A `useCookie` holding the raw error object and the raw CMS response ships internal detail to the browser and then re-uploads it on every subsequent request until it expires — including to third-party asset domains on the same site. If a debugging channel is genuinely wanted, gate it on `import.meta.dev` and use `useState`, which dies with the page.

---

## 6. Types — one discriminated union, zero `any`

```ts
// app/types/blocks.types.ts
export interface StrapiImage {
  url: string
  alternativeText?: string
  width?: number
  height?: number
}

interface BlockBase {
  id: number
  __component: string
}

export interface HeroTypingCollage extends BlockBase {
  __component: 'hero.typing-collage'
  headline: string
  images?: StrapiImage[]
}

export interface ContentFeatureGrid extends BlockBase {
  __component: 'content.feature-grid'
  title?: string
  features?: { id: number, label: string }[]
}

/** Every block the app can render. Add the interface here when you add the folder. */
export type StrapiBlock = HeroTypingCollage | ContentFeatureGrid
```

Each block component then types its own prop off the union — the discriminator does the narrowing for free:

```vue
<script setup lang="ts">
import type { HeroTypingCollage } from '~/types/blocks.types'

defineProps<{ data: HeroTypingCollage }>()
</script>
```

**`export type CMSData = any` is not a type, it is an opt-out**, and one `any` at the block boundary propagates into every template that touches `data`. If a payload genuinely cannot be described yet, `unknown` plus a narrowing function is the escape hatch (`type-safety.md` §1) — and an `eslint-disable` for `no-explicit-any` in a shared types file is a finding, not a workaround.

`strapi-page.types.ts` (page + SEO shape), `strapi-query.types.ts` (`StrapiStatus`, `StrapiLocale`, filter operators) and `global.types.ts` split by concern; blocks stay in `blocks.types.ts` because that file grows with every folder in §3.

**`seo?: StrapiSeo | StrapiSeo[]`** is worth killing at the boundary rather than living with. Normalise once in the composable and let the rest of the app see a single object — a union that every consumer must re-narrow is the schema's ambiguity leaking into fifty call sites.

---

## 7. `useStrapiPage` — the corrected shape

```ts
// app/composables/useStrapiPage.ts
import type { MaybeRefOrGetter } from 'vue'
import type { StrapiPage } from '~/types/strapi-page.types'
import { POPULATION_MAPS, mapToStrapiLocale } from '~/utils/strapi-schema'

export function useStrapiPage(slugInput: MaybeRefOrGetter<string>) {
  const { find } = useStrapi()
  const { locale } = useI18n()
  const { enabled: previewEnabled } = usePreviewMode()

  const strapiLocale = computed(() => mapToStrapiLocale(locale.value))

  const slug = computed(() => {
    const raw = toValue(slugInput)
    return !raw || raw === '/' || raw === 'home' ? 'homepage' : raw
  })

  const status = computed(() => (previewEnabled.value ? 'draft' : 'published'))

  const { data, status: fetchStatus, error, refresh } = useAsyncData(
    // Getter key: carries every dependency, so no `watch` option is needed.
    () => `page-${slug.value}-${strapiLocale.value}-${status.value}`,
    () => find<StrapiPage>('pages', {
      filters: { slug: { $eq: slug.value } },
      status: status.value,
      locale: strapiLocale.value,
      populate: POPULATION_MAPS.page,
    }),
  )

  const pageContent = computed(() => data.value?.data?.[0] ?? null)
  const dynamicZone = computed(() => pageContent.value?.dynamic_zone ?? [])
  const seo = computed(() => {
    const raw = pageContent.value?.seo
    return (Array.isArray(raw) ? raw[0] : raw) ?? null   // normalised once, here
  })

  return { pageContent, dynamicZone, seo, status: fetchStatus, error, refresh, slug }
}
```

What changed, and why each one matters:

- **The key is a getter, not a template literal.** `` `page-${slug.value}` `` is evaluated **once**, at setup. Every slug then shares one cache entry, so a client-side navigation serves the previous page's blocks from the payload until the refetch lands — the classic "wrong page flashes" bug. The getter form (`data-fetching.md` §2) refetches on change and gives each slug its own entry; the `watch` option becomes redundant.
- **`isLoading` (`pending`) is gone in favour of `status`.** Four states, not a boolean — `!data` cannot distinguish loading from empty from failed.
- **The `$or` on `slug` / `documentId` is gone.** Matching a slug against a `documentId` means a page is reachable at two URLs with no canonical between them, and it doubles the query for a lookup nobody performs by hand. If preview-by-id is needed, give it its own composable and `noindex` it.
- **The `as unknown as Record<string, unknown>` casts are gone.** If the SDK's filter typing genuinely does not accept your shape, put **one** typed wrapper in `utils/strapi-schema.ts` and cast there once — a cast repeated at every call site is a pattern, and the pattern is the bug (`type-safety.md`).
- **No global-loading `watch`.** Driving a global spinner from a page fetch is a side effect running per page instance; Nuxt's own `useLoadingIndicator` / `<NuxtLoadingIndicator>` already tracks navigation, and `status` covers in-page skeletons.
- **SEO moved out** (§8). A composable that fetches, normalises, drives a global spinner and writes meta tags has four reasons to change.

### Preview status: do not trust the browser

The security-relevant change: **the secret never reaches the client, and the client never decides `status`.**

```ts
// ✗ leaks the secret to every visitor and lets anyone request drafts
const { public: { strapiPreviewSecret } } = useRuntimeConfig()
const status = computed(() => route.query.secret === strapiPreviewSecret ? 'draft' : 'published')
```

Anything under `runtimeConfig.public` is embedded in the client bundle. Publishing `PREVIEW_SECRET` there means a visitor can read it from the page source, append it to any URL, and read unpublished content — and it defeats the whole point of the `.env` value being a shared secret between Strapi and Nuxt (`strapi-backend.md` §6).

The secret belongs in `runtimeConfig` (private), compared **once**, server-side, in the handshake route below. After that the page only asks "is preview mode on?".

---

## 8. `/api/preview` — the Nitro handshake

This is the route `config/admin.ts` points at. It is the only place the secret is compared.

```ts
// server/api/preview.get.ts
import { z } from 'zod'

const QuerySchema = z.object({
  slug: z.string().startsWith('/'),
  secret: z.string(),
  status: z.enum(['draft', 'published']).default('draft'),
  locale: z.string().optional(),
})

export default defineEventHandler(async (event) => {
  const { slug, secret, status, locale } = await getValidatedQuery(event, QuerySchema.parse)
  const config = useRuntimeConfig(event)

  if (!config.previewSecret || secret !== config.previewSecret) {
    throw createError({ statusCode: 401, statusMessage: 'Invalid preview secret' })
  }

  setCookie(event, 'preview_mode', status, {
    httpOnly: false,     // the composable reads it; it carries no secret, only a mode
    sameSite: 'none',    // the page is rendered inside the Strapi admin iframe
    secure: true,        // required by SameSite=None — see the dev note below
    path: '/',
    maxAge: 60 * 60,
  })

  const target = locale ? `/${locale}${slug}` : slug
  return sendRedirect(event, target, 307)
})
```

- **`getValidatedQuery` with a schema**, per `backend-errors.md` §1 — the slug is attacker-controllable and is about to become a redirect target. `startsWith('/')` is what stops `?slug=https://evil.example` becoming an open redirect from your domain.
- **The cookie holds a mode, never the secret.** It says "this browser passed the check", which is why it is safe to let the client read it.
- **`SameSite=None; Secure` is mandatory for the iframe.** The admin panel is a different origin, so a `Lax` cookie is simply not sent on the framed request and preview appears not to work. Over plain `http://localhost`, browsers reject `Secure` cookies — either run the frontend over HTTPS in dev, or accept that preview is verified against the deployed environment (`strapi-backend.md` §3 has the matching CSP concession).
- **Pair it with an exit route** (`server/api/preview-exit.get.ts`, `deleteCookie` + redirect) and a small "Exit preview" affordance, or the editor's browser keeps showing drafts on the public site for an hour and reports it as a bug.
- Then the composable side, once:

```ts
// app/app.vue (or a plugin)
usePreviewMode({ shouldEnable: () => Boolean(useCookie('preview_mode').value) })
```

Verify `usePreviewMode`'s options against the installed Nuxt version before relying on the signature.

**Draft content and tokens.** Reading drafts requires a Strapi token with draft permission. Never give that token to the browser: a public read token in `runtimeConfig.public` that can see drafts publishes every unfinished page. Route draft reads through a Nitro proxy (`server/api/cms/[...].ts`) that checks the `preview_mode` cookie and attaches the privileged token server-side; published reads may go direct.

---

## 9. `useGlobal` and SEO

`useGlobal` fetches the site-wide single type (name, favicon, `defaultSeo`) with a locale-keyed `useAsyncData` key. Because the key is stable, calling it from several components costs one request — that is the deduplication working, not a leak. Same getter-key rule as §7.

Extract the meta into its own composable so pages and blocks share one resolution order:

```ts
// app/composables/useStrapiSeo.ts
export function useStrapiSeo(seoInput: MaybeRefOrGetter<StrapiSeo | null | undefined>) {
  const { defaultSeo } = useGlobal()
  const { public: { siteUrl, cmsUrl } } = useRuntimeConfig()
  const route = useRoute()

  const seo = computed(() => toValue(seoInput))
  const title = computed(() => seo.value?.metaTitle ?? defaultSeo.value?.metaTitle)
  const image = computed(() => absoluteMediaUrl(seo.value?.metaImage?.url ?? defaultSeo.value?.metaImage?.url, cmsUrl))

  useSeoMeta({
    title,
    description: () => seo.value?.metaDescription ?? defaultSeo.value?.metaDescription,
    ogTitle: title,
    ogImage: image,
    ogUrl: () => `${siteUrl}${route.path}`,
    ogType: 'website',
    twitterCard: 'summary_large_image',
    twitterImage: image,
  })
}
```

- **No project domain hardcoded in a composable.** `|| 'https://cms.example.com'` inside a fallback chain is the silent-fallback anti-pattern from `smells.md` §3 — it turns a missing `NUXT_PUBLIC_SITE_URL` in production into absolute URLs pointing at whatever the last project was called. Read runtime config; fail loudly if it is absent.
- **Site defaults belong in `nuxt.config`'s `app.head` or `useSeoMeta` in `app.vue`**, not repeated as `?? 'Trinity Software Center'` on eight properties.
- **`absoluteMediaUrl` is one util** (`app/utils/`) handling Strapi's relative media paths: return the URL unchanged if it is already absolute, otherwise prefix the CMS origin, and normalise the slash. Every `og:image`, `<nuxt-img>` and download link uses it — this logic inlined per component is the second-most duplicated snippet in a Strapi/Nuxt app after the populate map.

---

## 10. Smells to flag on the Nuxt side

1. **Preview secret in `runtimeConfig.public`**, or a client-side comparison of it — critical, it exposes unpublished content.
2. **A Strapi token with draft or write scope reachable from the browser.**
3. **A template-literal `useAsyncData` key** carrying reactive values — one cache entry for every slug.
4. **A soft 404** — an in-page not-found component instead of `createError({ statusCode: 404 })`.
5. **A hand-maintained populate map** alongside a block registry that could derive it (§4).
6. **`any` / `CMSData` at the block boundary**, or an `eslint-disable` in a shared types file.
7. **A block importing another block**, or a config importing a component.
8. **Repeated `as unknown as Record<string, unknown>`** at SDK call sites instead of one typed wrapper.
9. **Hardcoded domains, site names or CMS URLs** in composables and components.
10. **`populate: '*'` on a page query** — it is one level deep, so it silently misses nested block relations while looking like it populates everything, and it over-fetches what it does reach.
11. **Error payloads written to cookies or global state in production.**
12. **A registry entry with no matching Strapi component UID** (or vice versa) — verify both against `components.d.ts`.
