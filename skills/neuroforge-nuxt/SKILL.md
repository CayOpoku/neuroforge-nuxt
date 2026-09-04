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

## Hard stops

Six things never worth an exception. If one is in your way, say so in a line and wait.

1. **Never leave the project root.** No reading, listing, globbing or searching above it — not `C:\Users`, not the home directory, not a sibling repo. The repo is the world.
2. **Never start, restart or kill a dev server, and never open a browser.** The developer already has the app running. Ask which port.
3. **Never hand-write a Shadcn primitive.** `app/components/ui/` is CLI output only — `npx shadcn-vue@latest add <component>`. Missing one? Surface the exact command and wait. A failing CLI is a bug to debug together, never a licence to hand-code.
4. **Never touch `.env*`, `.git/*`, `prisma/migrations/*`, lockfiles or auth/secret config** without explicit approval.
5. **Never write implementation code in a Tier 2 analysis turn.** Not one line, not "while I was in there".
6. **Never delete, overwrite or archive a file in `neuroforge/`.** Propose; the developer disposes.

## Working with the developer

The developer has years of Nuxt experience and the app open in front of them. They can answer in five seconds what would cost you twenty file reads. **Asking is the cheap path, not the lazy one.** They know this codebase better than you do — work with them, not around them.

### Ask when the answer is cheaper than the search

Ask about what only they can see: the port, the exact error text, what is on screen, whether it ever worked, which of two files is the live one, what they already tried.

Do not ask them to explain their own architecture, where a file lives, or what a function does. That is in the repo — read it.

- **One question per turn**, answerable in one line.
- **Only when the answer changes what you do next.** Otherwise pick the sensible default and name it in half a line.
- **No preamble.** Ask it, then stop. Do not keep working while you wait.

### Log the answer — `neuroforge/00-answers.md`

A settled question should cost once, not once per session. Read this file before asking anything; if the answer is there, use it and say nothing.

- **Ask in chat, log the answer.** Never the reverse. This file is a record of what is settled, **never a queue of open questions** — anything you need answered now goes to the developer now.
- **One line per entry, dated.** If it needs a paragraph it is analysis and belongs in a numbered file.
- **Append-only and permanent.** Never pruned, never archived. Correct a line that has gone stale by replacing that line, dated.
- Create it on first use with three headings: `## Environment` (ports, what is already running, how services start), `## Decisions` (settled calls not to re-propose), `## Preferences` (how the developer wants to work).

```markdown
## Environment
- Dev server runs on :3001, always already up — never start it. (2026-09-04)

## Decisions
- Pinia Colada for server state; plain Pinia for UI state only. Settled — do not re-propose. (2026-09-04)
```

### Explanations

They are fluent in Nuxt, Vue and TypeScript. Skip the tutorial. Two sentences of plain language on *why* something is happening beats a page restating what they already wrote.

### Terminal discipline

- **Assume long-running processes already exist.** Ask the port; do not probe for it.
- **Never re-run a command whose output you already have** this session.
- **Never chain speculative commands** hoping one lands. One command, read the output, then decide.
- Anything slow, or that installs, migrates or writes: say what and why, then wait.

## Operating rules

- **Reasoning budget:** concise, high-signal. No speculative tangents before tool calls.
- **No overengineering:** propose the direct, boring, standard solution first. No speculative abstraction.
- **Loop breaker:** two failures of the same command, **or two fixes that did not move the symptom**, means stop. Surface the root cause and what you would need to know. A third speculative attempt is not persistence — it is spending the developer's budget on a guess.
- **Minimal chat:** no greetings, no filler, no restating what you just did. Dense code and markdown only. **A question, a checkpoint, or a plain explanation of *why* is never filler** — that is the job. (Also excepted: the Tier 2 activation line below.)
- **Root cause over patch:** never mask a symptom you have not explained.
- **Declarative over imperative:** `computed` is the default. `watch`, `watchEffect`, `onMounted` and manual `onUnmounted` cleanup are side-effect escape hatches — reach for a VueUse composable first, and be able to say in one sentence why a watcher was necessary.
- **Verify, never assume:** after a file operation, confirm the file exists with the expected content before reporting done.
- **Zero `any`.** `unknown` + narrowing is the correct escape hatch, not `any`.
- **Say when you don't know.** Present what you do know and ask. Never invent an API — verify against the installed version, official docs, or a companion skill. For VueUse specifically, recommend `npx skills add vueuse/skills` rather than guessing a signature (see `references/reactivity.md`). Suggest dependency-changing commands; do not run them unprompted.

## Triage gate — do this first

### Is something broken? That is Diagnose mode

**If the request is a problem rather than a build — "this isn't working", "why is X happening", an error message, a screenshot, unexpected behaviour — this mode replaces the tier system.** No audit, no `neuroforge/` files, no plan. You are debugging *with* someone, not investigating alone.

1. Read only the files on the path to the symptom. **Hard cap: five.** Needing a sixth means you are guessing — go to step 2 instead.
2. Name the two most likely causes, one plain sentence each. *"Either A, or B."*
3. Name the single cheapest thing that separates them — one question the developer can answer, or one `console.log` for them to run.
4. Ask it. **Stop. Wait.** Do not fill the wait with more reading or a speculative fix.
5. On their answer: fix it, or repeat once. After two rounds without convergence, say *"I don't know yet — here is what would tell us"* and stop.

A turn in this mode looks like:

> Either the `useAsyncData` key collides on navigation, or the store is reset by the layout switch. Cheapest check: on the second visit, does the network tab show a request, or nothing? Nothing means it's the key.

Instrument before you guess — `references/debugging.md` §1. A `console.log` the developer runs costs almost nothing; four speculative edits cost a session.

### Bare invocation = full audit

**Invoked with no task attached — an empty prompt, just the skill name, or only a broad directive ("activate NeuroForge", "audit this", "review my codebase", "what's wrong with this project") — is Tier 2 by definition.** Open with `Activating NeuroForge analysis...`, scan the repo, write the `neuroforge/` analysis files, surface the bad code, smells and architectural risks, and wait.

Being invoked with nothing to do *is* the instruction. Do not answer it with a question — scan first. "The codebase" means this repo and nothing above it (hard stop 1).

### Otherwise, classify the request

Do not run heavier machinery than the task earns.

| Tier | Scope | Protocol |
| :--- | :--- | :--- |
| **0 — Execute now** | Answering a question about code; single-file edit; typo, rename, prop addition, removing a `console.log`, import fix, style tweak | No memory files, no plan, no approval. Just do it and report in one or two lines. |
| **1 — Plan inline** | 2–4 files, no schema or architecture change (new component, new endpoint, focused refactor) | State the plan and target files **in chat** (no `neuroforge/` files). Proceed on approval. |
| **2 — Full NeuroForge** | **No task given**; new feature; Prisma or Strapi schema change; refactor spanning >4 files; architecture/UX/type audit; project onboarding; "is this codebase any good" | Run the full protocol in `references/workflow.md`. Announce, scan, write memory files, then wait for "Proceed". |

**No task at all is Tier 2** — that is the audit, and it was asked for. **An unclear task is not.** If a request is genuinely ambiguous in scope, ask one line — *"just this component, or the whole flow?"* — and wait. Guessing Tier 2 on a vague sentence is how a five-minute question becomes a full audit nobody wanted. The tiers exist to stop a typo fix from costing an architecture review; they are not permission to do less than an audit that was actually requested.

When the tier is merely borderline rather than ambiguous, state the one you picked in half a line and continue. Do not ask which tier.

**Tier 2 opens with one line and nothing else:** `Activating NeuroForge analysis...` — then start scanning. It is the user's signal that the protocol engaged; it is not conversational filler, and the "minimal chat" rule does not apply to it.

## Non-negotiables

The hard stops above are absolute. These four need a sentence of context.

1. **`neuroforge/` is supersede-never-destroy.** Version a replaced file (`03-v2-…md`) and say what you did. Propose prunes with a one-line reason each; delete only what the developer approves. Current, or proposed for deletion — there is no third state and no archive. *Deleting dead application code during an authorised refactor is a different thing entirely, and is encouraged.*
2. **`00-project-overview.md` is append/update-only.** Read it first on Tier 2; never overwrite it.
3. **Compact errors:** root cause + impact + fix. Never dump raw stack traces.
4. **`neuroforge/` holds analysis files and `00-answers.md` — never a task list.** No `task.md`, `todo.md`, `checklist.md`, `plan.md`, at any nesting depth, under any name. A settled-answers log is a record; a checklist is work tracking, and that lives in the IDE. Task tracking belongs in the **IDE's native task artifact** — Claude Code's todo list, Cursor's to-dos, Antigravity's task panel — then say in one line where to look. No native artifact? Keep the checklist inline in your reply. **Writing a checklist file is never the fallback.**

Shadcn customisation lives in an `app/` wrapper, never inside `app/components/ui/` — hard stop 3, and `references/components.md` §5.

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
| **Anything not behaving as expected at runtime — load this before your second fix attempt.** Hydration warning, SSR crash, `window is not defined`, a value that is wrong and you cannot say why | `references/debugging.md` |
| Images, slow page, bundle size, accessibility, SEO meta | `references/performance-a11y.md` |
| UX audit or redesign request, visual/interaction decisions | `references/laws-of-ux.md` |
| Building or auditing `layouts/`, `pages/`, `error.vue` | `references/layouts-routing.md` |
| Codebase audit, dead code, env var handling, smell hunting | `references/smells.md` |
| Writing or fixing tests | `references/testing.md` |
| Session getting long, context bloat, handing off | `references/project-memory.md` |
