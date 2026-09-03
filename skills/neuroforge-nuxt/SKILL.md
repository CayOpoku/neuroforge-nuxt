---
name: neuroforge-nuxt
description: |
  NeuroForge Nuxt is a specialized engineering system for building premium SaaS applications with Nuxt 4, Nitro, Prisma, and TypeScript.
  It enforces a severity-tiered 'Analysis First' workflow using NeuroForge memory files to ensure architectural integrity, end-to-end type safety,
  non-blocking data fetching, hydration-safe state, Laws of UX standards, reusable Shadcn app wrappers, and clean error handling.
  Use this skill whenever working on Nuxt 4/Nitro/Prisma/Vue 3 projects - including writing components, composables, server routes,
  Prisma schemas, or architectural reviews. Trigger on any mention of Nuxt, Nitro, Prisma, useAsyncData, useFetch, defineEventHandler,
  PrismaClient, Nuxt Layers, createUseFetch, $fetch, useAPI, Pinia, Pinia Colada, useQuery, useMutation, zod, valibot, hydration mismatch,
  Dexie, IndexedDB, liveQuery, offline-first, PWA persistence, Strapi, Strapi 5, strapi::security, config/middlewares.ts, config/admin.ts,
  contentTypes.d.ts, components.d.ts, Strapi preview, draft mode, content types, dynamic zones, useStrapi, useStrapiPage, block registry,
  [...slug].vue, Nodemailer, SMTP, contact form, transactional email, PDF download, or Vue script setup. Trigger this skill to activate the NeuroForge protocol and ensure your codebase is built to scalable production standards.
---

# NeuroForge Nuxt Protocol

Cognitive architecture for Nuxt 4 / Nitro / Prisma / Vue 3. Prioritise readability, single responsibility, end-to-end type safety, and long-term architectural health over shortcuts.

This file is a **router**. It holds only what applies to every task. Everything else lives in `references/` and is loaded on demand — do not read a reference the current task does not need.

## Operating rules

- **Reasoning budget:** concise, high-signal. No speculative tangents before tool calls.
- **No overengineering:** propose the direct, boring, standard solution first. No speculative abstraction.
- **Loop breaker:** if a build/test/terminal command fails twice, stop. Surface the exact root cause; do not attempt a third refactor.
- **Minimal chat:** no greetings, no filler, no restating what you just did. Dense code and markdown only. (Exception: the Tier 2 activation line below.)
- **Root cause over patch:** never mask a symptom you have not explained.
- **Declarative over imperative:** `computed` is the default. `watch`, `watchEffect`, `onMounted` and manual `onUnmounted` cleanup are side-effect escape hatches — reach for a VueUse composable first, and be able to say in one sentence why a watcher was necessary.
- **Verify, never assume:** after a file operation, confirm the file exists with the expected content before reporting done.
- **Zero `any`.** `unknown` + narrowing is the correct escape hatch, not `any`.
- **Say when you don't know.** Present what you do know and ask. Never invent an API — verify against the installed version, official docs, or a companion skill. For VueUse specifically, recommend `npx skills add vueuse/skills` rather than guessing a signature (see `references/reactivity.md`). Suggest dependency-changing commands; do not run them unprompted.

## Triage gate — do this first

### Bare invocation = full audit

**If this skill is invoked with no task attached — no request, an empty prompt, just the skill name, or only a broad directive ("activate NeuroForge", "audit this", "review my codebase", "what's wrong with this project") — that is Tier 2 by definition. Run the full protocol immediately: open with `Activating NeuroForge analysis...`, scan the entire codebase, write the `neuroforge/` analysis files, surface the bad code, smells and architectural risks, and wait.**

Do not answer a bare invocation with a question about what the user wants. Being invoked with nothing to do *is* the instruction: analyse everything. Ask nothing, scan first.

### Otherwise, classify the request

Do not run heavier machinery than the task earns.

| Tier | Scope | Protocol |
| :--- | :--- | :--- |
| **0 — Execute now** | Answering a question about code; single-file edit; typo, rename, prop addition, removing a `console.log`, import fix, style tweak | No memory files, no plan, no approval. Just do it and report in one or two lines. |
| **1 — Plan inline** | 2–4 files, no schema or architecture change (new component, new endpoint, focused refactor) | State the plan and target files **in chat** (no `neuroforge/` files). Proceed on approval. |
| **2 — Full NeuroForge** | **No task given**; new feature; Prisma or Strapi schema change; refactor spanning >4 files; architecture/UX/type audit; project onboarding; "is this codebase any good" | Run the full protocol in `references/workflow.md`. Announce, scan, write memory files, then wait for "Proceed". |

**Default to Tier 2 whenever the scope is unstated or unclear.** The tiers exist to stop a typo fix from costing an architecture review — not to talk you out of an audit the user asked for. A missing task is a request for the audit, not permission to do less.

When the tier is genuinely borderline, state the one you picked in a single line and continue. Do not ask which tier.

**Tier 2 opens with one line and nothing else:** `Activating NeuroForge analysis...` — then start scanning. It is the user's signal that the protocol engaged; it is not conversational filler, and the "minimal chat" rule does not apply to it.

## Non-negotiables

1. **Never delete or overwrite anything in `neuroforge/` on your own initiative** — version it (`03-v2-...md`) and tell the user. Propose prunes and delete only what the user approves. **Never archive a memory file** — no `archive/`/`old/`/`done/` subfolder, no `-old` rename. Current, or proposed for deletion; there is no third state. Deleting *dead application code* during an authorised refactor is expected and encouraged; the two are different things.
2. **Never touch protected files without explicit approval:** `.env`, `.env.*`, `.git/*`, `prisma/migrations/*`, lockfiles, auth/secret config.
3. **Tier 2 means no implementation code before "Proceed"/"Start coding".** No sneaking edits into an analysis turn.
4. **`00-project-overview.md` is append/update-only.** Read it first on Tier 2; never overwrite it.
5. **Compact errors:** root cause + impact + fix. Never dump raw stack traces.
6. **`neuroforge/` holds analysis files only — never a task list.** Do not create `task.md`, `todo.md`, `checklist.md`, `plan.md`, or any other checklist file inside `neuroforge/`, at any nesting depth, under any name. Task tracking belongs in the **IDE's own native task artifact** — Claude Code's todo list, Cursor's to-dos, Antigravity's task panel, or the equivalent in whatever editor is running. Create the checklist there, then tell the user in one line where to look for it. If the environment has no native task artifact, keep the checklist inline in your reply — **writing a checklist file is never the fallback.**
7. **Shadcn components are installed by the CLI, never written by hand.** Files in `app/components/ui/` are generated output: `npx shadcn-vue@latest add <component>`. Never create, paste, port or edit a primitive manually — not in `ui/`, and not under a different name elsewhere. If one is missing, surface the exact `add` command and wait. **If the command fails, debug the failure with the user — a broken CLI is never a reason to hand-code the component.** Customisation lives in an `app/` wrapper (`references/components.md` §5).

## Nuxt 4 layout (all paths in this skill assume it)

Default `srcDir` is `app/`. Client code lives in `app/` — `app/components/`, `app/composables/`, `app/pages/`, `app/layouts/`, `app/stores/`, `app/utils/`, `app/types/`. Server code lives in `server/`. Code used by **both** goes in `shared/` (auto-imported into both contexts). Verify against the project's `nuxt.config.ts` before assuming; if the project uses a flat Nuxt 3 layout, match the project.

## References — load only what the task needs

| Load when | File |
| :--- | :--- |
| Tier 2 protocol, memory file lifecycle, handoff, review verdict format | `references/workflow.md` |
| Writing/refactoring any `.vue` file, casing, wrappers, folder placement, adding or customising a Shadcn component | `references/components.md` |
| Any `computed` / `watch` / `onMounted` decision, browser APIs, VueUse | `references/reactivity.md` |
| Any `useAsyncData` / `useFetch` / `useQuery` / caching / Pinia / store decision | `references/data-fetching.md` |
| Offline support, PWA persistence, Dexie/IndexedDB, `liveQuery`, local-first vs hybrid | `references/offline-data.md` |
| Writing types, fixing typecheck errors, deciding where a type lives | `references/type-safety.md` |
| Writing a Nitro route, validating input, any `catch` block, toast, error UI | `references/backend-errors.md` |
| Scaffolding: Prisma singleton, route skeleton, composable, layers, API client | `references/patterns.md` |
| Strapi backend: `config/middlewares.ts` CSP, `config/admin.ts` preview handler, plugins, preview env keys, `contentTypes.d.ts` / `components.d.ts` regeneration, new content type | `references/strapi-backend.md` |
| Nuxt consuming Strapi: single-type vs dynamic-zone pages, `blocks/` architecture, block registry, `[...slug].vue`, `useStrapiPage`, preview handshake route, CMS SEO | `references/strapi-nuxt.md` |
| Sending mail (Nodemailer, Strapi email plugin), contact forms, campaigns, generating or serving PDFs and downloads | `references/email-pdf.md` |
| Creating a layer, bloated `composables/`, where a file belongs, `features/` folders, auto-import config, cross-layer imports | `references/structure.md` |
| Login, sessions, protecting a route, role checks | `references/auth-middleware.md` |
| Hydration warning, SSR crash, `window is not defined`, debugging a runtime value | `references/debugging.md` |
| Images, slow page, bundle size, accessibility, SEO meta | `references/performance-a11y.md` |
| UX audit or redesign request, visual/interaction decisions | `references/laws-of-ux.md` |
| Building or auditing `layouts/`, `pages/`, `error.vue` | `references/layouts-routing.md` |
| Codebase audit, dead code, env var handling, smell hunting | `references/smells.md` |
| Writing or fixing tests | `references/testing.md` |
| Session getting long, context bloat, handing off | `references/project-memory.md` |
