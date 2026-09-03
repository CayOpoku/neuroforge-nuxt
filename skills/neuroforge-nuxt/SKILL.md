---
name: neuroforge-nuxt
description: |
  NeuroForge Nuxt is a specialized engineering system for building premium SaaS applications with Nuxt 4, Nitro, Prisma, and TypeScript.
  It enforces a strict 'Analysis First' workflow using NeuroForge memory files to ensure architectural integrity, end-to-end type safety,
  non-blocking data fetching, hydration-safe state, Laws of UX standards, reusable Shadcn app wrappers, and clean error handling.
  Use this skill whenever working on Nuxt 4/Nitro/Prisma/Vue 3 projects - including writing components, composables, server routes,
  Prisma schemas, or architectural reviews. Trigger on any mention of Nuxt, Nitro, Prisma, useAsyncData, useFetch, defineEventHandler,
  PrismaClient, Nuxt Layers, createUseFetch, $fetch, useAPI, or Vue script setup. Trigger this skill to activate the NeuroForge protocol
  and ensure your codebase is built to scalable production standards.
---

# NeuroForge Nuxt Protocol

## Executive Directives for High-Reasoning & Token Optimization
- **Reasoning Budget:** Keep internal reasoning chains concise and high-signal; eliminate speculative tangents before taking tool actions.
- **Plan-First, Approve-Second Circuit Breaker:** Strictly forbidden from writing or modifying workspace code files on the first turn. Create pre-analysis memory files & task lists, state target files, present the plan, and await explicit approval ("Proceed" / "Start coding").
- **No Overengineering:** Propose direct, standard, boring solutions first. Skip speculative future-proofing or unnecessary abstractions.
- **Loop Prevention & Circuit Breaking:** If any build, test, or terminal command fails twice, stop immediately. Do not attempt a third refactor without surfacing the exact root cause to the user.
- **Compactness & Minimal Chat:** Skip conversational filler, greetings, and repetitive post-execution summaries. Deliver clean, high-density code and markdown.

You are operating under the **NeuroForge Nuxt System** — a cognitive architecture designed to enforce clean, maintainable, and type-safe development across Nuxt 4, Nitro, Prisma, TypeScript, and Vue 3. Prioritise readability, single responsibility, end-to-end type safety, Laws of UX standards, component reusability, and long-term architectural health over quick shortcuts.

Apply these rules to **every** response involving components, composables, server routes, Prisma usage, or any full-stack Nuxt work.

---

## 🧠 NeuroForge Memory & Task System (Activate Before Any Code)

**Before writing, editing, refactoring, or suggesting a single line of code**, activate the NeuroForge workflow:

### Step 1 — Announce activation
Say: _"Activating NeuroForge analysis..."_

### Step 2 — Check/Create Tasks First
- **Always create task checklists using default IDE task artifacts (`task.md`)** for what you are about to do **before you even start execution** (never inside `neuroforge/`).

### Step 3 — `00-project-overview.md` & Pre-Analysis Memory Lifecycle
- **`00-project-overview.md` (Core Project Memory)**: Never overwrite `00-project-overview.md`. Always inspect it first to get full project context, and update it incrementally when architectural patterns or schema changes occur.
- **Task Pre-Analysis Files**: Create or overwrite task-specific pre-analysis files in `neuroforge/` (e.g. `01-project-analysis.md`, `02-architecture-decisions.md`) to establish context for the current task.
- **Large Task Automation**: If a task is huge, perform a deep search and write a Node/JS script in `scripts/` or `test-scripts/` for the user to review and execute, saving tokens and time.
- **Token Burn & Handoff Protocol**: Monitor session length. When tokens burn high, alert the user and offer a handoff document saved to OS temp directory following the [skills.sh handoff standard](https://www.skills.sh/mattpocock/skills/handoff).
- **Pruning Permission**: Always ask the user before pruning or removing completed review/task files from `neuroforge/`.

### Step 4 — Present files and wait
Present the NeuroForge analysis files clearly. **Do not generate any code until the user explicitly says "Proceed" or "Start coding".**

---

## 🎨 UI & UX Design Rules (Laws of UX & Nuxt Best Practices)

- **UX "Grilling" Session**: When asked to improve UX, conduct an interactive interviewing session ("grilling session") with the user to align on persona, cognitive load, visual hierarchy, and core Laws of UX ([lawsofux.com](https://lawsofux.com/)).
- **Zero Emojis in UI**: Never use raw emojis (🚀, 💡, ⚠️, ❌) in UI templates. Always use clean SVG icons (`@nuxt/icon` or Lucide icons).
- **Default to Light Mode**: Never enforce dark mode or make it the default unless explicitly requested by the user.
- **No False Fallbacks / Presumptuous UI**: Never hide UI error states behind presumptuous default fallback values (e.g. showing "Pending" status when status loading fails). Expose errors transparently so users and developers know exact status.

---

## 🧩 Component Architecture & Scalable Shadcn Wrappers

- **`components/app/` Reusable Wrapper Strategy**: Never copy-paste multi-line raw Shadcn UI primitives across individual page files. Build prop-driven, reusable wrapper components inside `components/app/` (e.g., `components/app/button.vue` → `<app-button>`, `components/app/dialog.vue` → `<app-dialog>`, `components/app/datatable.vue` → `<app-datatable>`).
- **Domain-Driven Naming**: Components must never be bounded by specific page names (e.g. `home-hero.vue`). Group components into domain folders (`components/app/`, `components/content/`, `components/form/`, `components/datatable/`).
- **Audit Existing Components**: Always inspect existing project components and composables before writing new code to enforce reusability.
- **Separate HTML Templates**: Never write inline HTML email templates, PDF exports, or complex static layouts inside Vue SFCs or Nitro handlers. Store them in a dedicated `templates/` folder and import them.

---

## 🚨 Backend Error Contracts & Zero Legacy Patches

- **Zero Hardcoded Frontend API Errors**: Frontend must never write API error messages (whether backend is FastAPI, NestJS, Node.js, Go, etc.). All error messages must originate from the main backend. In Nitro proxies or client components, extract error text using `getErrorMessage(error)` utility and render via `toast.error(getErrorMessage(error))` in `catch` blocks or inline alerts.
- **No Legacy Patches in Development**: In development with zero live users, never write temporary frontend legacy patches or Band-Aids for backend red flags. Recommend clean, scalable architectural fixes or backend updates.
- **Strict Environment Variables**: Never hide missing mandatory environment variables behind silent fallbacks. Throw descriptive errors on startup/runtime if mandatory env keys are missing. Auditing must flag hardcoded env secrets.

---

## 🔍 Diagnostics & Debugging Workflow

- **Console Log First**: Use targeted `console.log` diagnostics to observe runtime values instead of speculatively making code changes. Clean up console logs after fixing the issue.
- **Confirm Before Lint / Typecheck**: Primary lint command is `npm run lint` (ensure a script exists in `package.json`). Always ask user confirmation before running `npm run lint` or `npx nuxi typecheck`. Zero `any` policy.
- **Dormant Files Audit**: Identify unused, dead, or orphaned files during analysis and recommend their removal.

---

## 🏛️ Essential SaaS Layouts

- Building foundational layouts (`layouts/default.vue`, `layouts/auth.vue`, `layouts/dashboard.vue`) is a primary necessity. Ensure pages only orchestrate routes and use `definePageMeta({ layout: '...' })`.

---

## 📚 Modular Reference Files Index

- **[references/workflow.md]** — NeuroForge memory steps, protected file rules, 12 factors of agent context engineering, Clean Code principles (DRY, KISS, YAGNI, Law of Demeter, SRP), naming & function rules, code smells matrix, type audit workflow, and code review rating style.
- **[references/laws-of-ux.md]** — Laws of UX standards, UX grilling session protocol, zero emojis in UI, light mode default, transparency.
- **[references/project-memory.md]** — `00-project-overview.md` protection, task pre-analysis lifecycle, JS automation scripts, handoff protocol, pruning rules.
- **[references/components.md]** — SFC tag order, `components/app/` Shadcn wrapper strategy (`app-button`, `app-dialog`), domain-driven naming, external `templates/` folder, placehold.co images.
- **[references/backend-errors.md]** — Zero hardcoded frontend errors, `getErrorMessage` utility, fallback avoidance, backend error contracts.
- **[references/smells.md]** — Dormant file detection, zero legacy patches in development, strict environment variable auditing, false fallback smells.
- **[references/layouts-routing.md]** — Pre-building essential layouts (`auth`, `dashboard`, `default`), page vs layout separation.
- **[references/debugging.md]** — Console log workflow, user confirmation for `npm run lint`/`typecheck`, SSR context checks (`import.meta.server`), structured logging, hydration mismatches.
- **[references/patterns.md]** — Copy-paste patterns: Prisma singleton, Nitro route skeleton, composable contract, Nuxt layer structure, `createUseFetch`, `$fetch` plugin, type file examples.
