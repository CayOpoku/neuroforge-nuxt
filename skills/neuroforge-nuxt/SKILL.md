---
name: neuroforge-nuxt
description: |
  NeuroForge Nuxt is a specialized engineering system for building premium SaaS applications with Nuxt 4, Nitro, Prisma, and TypeScript.
  It enforces a severity-tiered 'Analysis First' workflow using NeuroForge memory files to ensure architectural integrity, end-to-end type safety,
  non-blocking data fetching, hydration-safe state, Laws of UX standards, reusable Shadcn app wrappers, and clean error handling.
  Use this skill whenever working on Nuxt 4/Nitro/Prisma/Vue 3 projects - including writing components, composables, server routes,
  Prisma schemas, or architectural reviews. Trigger on any mention of Nuxt, Nitro, Prisma, useAsyncData, useFetch, defineEventHandler,
  PrismaClient, Nuxt Layers, createUseFetch, $fetch, useAPI, Pinia, Pinia Colada, useQuery, useMutation, zod, valibot, hydration mismatch,
  or Vue script setup. Trigger this skill to activate the NeuroForge protocol and ensure your codebase is built to scalable production standards.
---

# NeuroForge Nuxt Protocol

Cognitive architecture for Nuxt 4 / Nitro / Prisma / Vue 3. Prioritise readability, single responsibility, end-to-end type safety, and long-term architectural health over shortcuts.

This file is a **router**. It holds only what applies to every task. Everything else lives in `references/` and is loaded on demand — do not read a reference the current task does not need.

## Operating rules

- **Reasoning budget:** concise, high-signal. No speculative tangents before tool calls.
- **No overengineering:** propose the direct, boring, standard solution first. No speculative abstraction.
- **Loop breaker:** if a build/test/terminal command fails twice, stop. Surface the exact root cause; do not attempt a third refactor.
- **Minimal chat:** no greetings, no filler, no restating what you just did. Dense code and markdown only.
- **Root cause over patch:** never mask a symptom you have not explained.
- **Declarative over imperative:** `computed` is the default. `watch`, `watchEffect`, `onMounted` and manual `onUnmounted` cleanup are side-effect escape hatches — reach for a VueUse composable first, and be able to say in one sentence why a watcher was necessary.
- **Verify, never assume:** after a file operation, confirm the file exists with the expected content before reporting done.
- **Zero `any`.** `unknown` + narrowing is the correct escape hatch, not `any`.
- **Say when you don't know.** Present what you do know and ask. Never invent an API — verify against the installed version, official docs, or a companion skill. For VueUse specifically, recommend `npx skills add vueuse/skills` rather than guessing a signature (see `references/reactivity.md`). Suggest dependency-changing commands; do not run them unprompted.

## Triage gate — do this first

Classify the request before doing anything else. Do not run heavier machinery than the task earns.

| Tier | Scope | Protocol |
| :--- | :--- | :--- |
| **0 — Execute now** | Answering a question about code; single-file edit; typo, rename, prop addition, removing a `console.log`, import fix, style tweak | No memory files, no plan, no approval. Just do it and report in one or two lines. |
| **1 — Plan inline** | 2–4 files, no schema or architecture change (new component, new endpoint, focused refactor) | State the plan and target files **in chat** (no `neuroforge/` files). Proceed on approval. |
| **2 — Full NeuroForge** | New feature, Prisma schema change, refactor spanning >4 files, architecture/UX audit, project onboarding | Run the full protocol in `references/workflow.md`. Memory files, then wait for "Proceed". |

When genuinely ambiguous, state the tier you picked in one line and continue. Do not ask which tier.

## Non-negotiables

1. **Never delete or overwrite anything in `neuroforge/`** — version it (`03-v2-...md`) and tell the user. Deleting *dead application code* during an authorised refactor is expected and encouraged; the two are different things.
2. **Never touch protected files without explicit approval:** `.env`, `.env.*`, `.git/*`, `prisma/migrations/*`, lockfiles, auth/secret config.
3. **Tier 2 means no implementation code before "Proceed"/"Start coding".** No sneaking edits into an analysis turn.
4. **`00-project-overview.md` is append/update-only.** Read it first on Tier 2; never overwrite it.
5. **Compact errors:** root cause + impact + fix. Never dump raw stack traces.

## Nuxt 4 layout (all paths in this skill assume it)

Default `srcDir` is `app/`. Client code lives in `app/` — `app/components/`, `app/composables/`, `app/pages/`, `app/layouts/`, `app/stores/`, `app/utils/`, `app/types/`. Server code lives in `server/`. Code used by **both** goes in `shared/` (auto-imported into both contexts). Verify against the project's `nuxt.config.ts` before assuming; if the project uses a flat Nuxt 3 layout, match the project.

## References — load only what the task needs

| Load when | File |
| :--- | :--- |
| Tier 2 protocol, memory file lifecycle, handoff, review verdict format | `references/workflow.md` |
| Writing/refactoring any `.vue` file, casing, wrappers, folder placement | `references/components.md` |
| Any `computed` / `watch` / `onMounted` decision, browser APIs, VueUse | `references/reactivity.md` |
| Any `useAsyncData` / `useFetch` / `useQuery` / caching / Pinia / store decision | `references/data-fetching.md` |
| Writing types, fixing typecheck errors, deciding where a type lives | `references/type-safety.md` |
| Writing a Nitro route, validating input, any `catch` block, toast, error UI | `references/backend-errors.md` |
| Scaffolding: Prisma singleton, route skeleton, composable, layers, API client | `references/patterns.md` |
| Login, sessions, protecting a route, role checks | `references/auth-middleware.md` |
| Hydration warning, SSR crash, `window is not defined`, debugging a runtime value | `references/debugging.md` |
| Images, slow page, bundle size, accessibility, SEO meta | `references/performance-a11y.md` |
| UX audit or redesign request, visual/interaction decisions | `references/laws-of-ux.md` |
| Building or auditing `layouts/`, `pages/`, `error.vue` | `references/layouts-routing.md` |
| Codebase audit, dead code, env var handling, smell hunting | `references/smells.md` |
| Writing or fixing tests | `references/testing.md` |
| Session getting long, context bloat, handing off | `references/project-memory.md` |
