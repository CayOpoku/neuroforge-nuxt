# NeuroForge Workflow, Protocol & Clean Code Guidelines

---

## 1. NeuroForge Memory System (Activate Before Any Code)

Before writing, editing, refactoring, or suggesting a single line of code, activate the NeuroForge workflow:

### Step 1 — Announce activation
Say: _"Activating NeuroForge analysis..."_

### Step 2 — Create `neuroforge/` folder and ensure `.gitignore` protection
- Create the `neuroforge/` folder if it doesn't already exist.
- **CRITICAL**: Check for a `.gitignore` file. If missing, create one. Always ensure `neuroforge/` (and its `.md` files) is added to `.gitignore` to prevent analysis/memory files from being committed to git. **This is mandatory and does NOT require asking for permission.**
- **UI Reviews folder**: Always create a subfolder `neuroforge/ui-reviews/` for HTML/MD mockup review files to keep visual drafts organized.
- **Scripts folder**: Put all database seed, test, or automation scripts inside `scripts/`, `test-scripts/`, or `seed-scripts/` at the root/layer rather than cluttering the root directory.
- Create multiple targeted `.md` files inside `neuroforge/`. Never use one big file. Each acts as a micro-agent with narrow responsibility:
  - `01-project-analysis.md` — codebase scan, folder structure, existing patterns
  - `02-architecture-decisions.md` — Layers, routing, rendering strategy
  - `03-composable-strategy.md` — what composables exist, what to split/merge
  - `04-prisma-type-safety.md` — schema review, select/include gaps, type sharing
  - `05-potential-spaghetti-risks.md` — god components, duplicated logic, prop drilling

Add more files as needed per task. Names and count should match task complexity.

### Step 3 — Populate files with deep analysis only (no executable code)
Each file must be dense, high signal-to-noise. Use:
- Clear sections and bullet points
- Illustrative snippets only (for illustration of what was found — not executable)
- Decision rationale (WHY, not just what)
- DRY / KISS / SOLID / YAGNI lens applied explicitly

**Instruction to obey verbatim**: _"Do not write or edit any code. Focus exclusively on scanning the entire system/context to create the NeuroForge md files for the user to review."_

### Step 4 — Present files and wait
Present the NeuroForge files clearly. **Do not generate any code until the user explicitly says "Proceed" or "Start coding".**

---

## 2. Strict NeuroForge Rules (Non-Negotiable)

1. **Never delete any file** — even in fast mode. If a file needs replacing, create a versioned new one (e.g. `03-v2-composable-strategy.md`) and leave the old intact, then tell the user what you did. Always ask before any destructive file action. **Never leave the user out of the loop when it comes to deleting files.**
2. **Never touch protected files without asking first** — this includes `.env`, `.env.*`, `.git/*`, `prisma/migrations/*`, `package-lock.json`, `pnpm-lock.yaml`, and any auth/secret config files. **Pause, explain what you want to change and why, then wait for explicit approval before proceeding.** (Exception: Automatically managing `.gitignore` for the `neuroforge/` folder as specified in Step 2).
3. **Never skip straight to code.** NeuroForge analysis always comes first.
4. **Mandatory Consent for Implementation**: You must NEVER write, modify, or suggest actual implementation code until the user has explicitly authorized you to do so (e.g., by saying "Proceed" or "Start coding"). There is NO "sneaking" in code changes. If the user hasn't authorized it, you don't write it.
5. **Verify Every Action**: Never hallucinate that a file has been updated or created. After performing a file operation, you MUST verify that the file exists and contains expected content before concluding.
6. **Use NeuroForge as single source of truth** — merges technical decisions with SaaS business requirements.
7. **Design for pause/resume** — NeuroForge files are checkpoints. Summarise current state before pausing.
8. **When issues arise, summarise concisely** (root cause + impact + proposed fix) before adding to NeuroForge. Never dump raw errors.
9. **If you don't know the solution, say so.** Pause, present what you do know in a NeuroForge file, and ask the user if they have any insight based on their review of the code. Never hallucinate a fix.
10. **Verify Every Action**: Re-verify file state using read/status tools before concluding tasks.

---

## 3. The 12 Factors of Agent Context Engineering & Token Guardrails

### High-Reasoning & Token Optimization Guardrails
1. **Reasoning Budget**: Keep internal reasoning chains concise and high-signal; avoid speculative tangents.
2. **Plan-First, Approve-Second Circuit Breaker**: Strictly forbidden from writing workspace code files on the first turn. Create pre-analysis memory files & task lists, state target files, present the plan, and await explicit approval ("Proceed" / "Start coding").
3. **No Overengineering**: Propose direct, standard, boring solutions first. Do not add future-proofing or speculative abstractions.
4. **Loop Prevention**: If any terminal command, build, or test fails twice, stop immediately. Do not attempt a third refactor without asking and presenting the root cause to the user.
5. **Compactness & Minimal Chat**: Skip conversational greetings, summaries, or post-execution explanations. Deliver clean code and markdown.

### The 12 Factors of Agent Context Engineering
Apply in every thought process:
1. **Natural language → tool calls**: Break every task into "reason → decide → document → act" phases.
2. **Own your prompts**: Self-refine internal reasoning for clarity and predictability before proceeding.
3. **Own your context window**: Keep NeuroForge dense and relevant. Remove noise. Prioritise high-signal insights.
4. **Tools are structured outputs**: Think `{ goal, constraints, options, chosen_path, rationale }`.
5. **Unify execution and business state**: NeuroForge tracks technical and SaaS product decisions together.
6. **Launch / Pause / Resume**: NeuroForge files are clean pause checkpoints.
7. **Contact humans via files**: Use NeuroForge to surface decisions, trade-offs, and questions for review.
8. **Own your control flow**: Explicit steps only: Analyse → Document → Present → Wait → Execute → Review.
9. **Compact errors**: Root cause + impact + fix. Never raw stack dumps.
10. **Small focused agents**: One md file per concern. Narrow responsibility per file.
11. **Meet users where they are**: Adapt to tone and urgency while maintaining engineering standards.
12. **Stateless reducer**: Core reasoning is stateless. NeuroForge carries all accumulated state and history.

---

## 4. Clean Code Principles & Engineering Standards

Code is clean if it can be understood easily by everyone on the team. Clean code can be read and enhanced by a developer other than its original author. With understandability comes readability, changeability, extensibility, and maintainability.

### Core Principles
- **DRY (Don't Repeat Yourself)**: Avoid writing the same logic in multiple places by extracting shared code into reusable functions or composables.
- **KISS (Keep It Simple, Stupid)**: Choose the simplest solution possible instead of overcomplicating or overengineering a feature. Simpler is always better. Reduce complexity as much as possible.
- **YAGNI (You Ain't Gonna Need It)**: Write only the code required to solve your current problem, skipping speculative features for the future.
- **Single Responsibility Principle (SRP)**: Ensure each class, file, component, or function focuses on doing one specific job.
- **Meaningful Naming**: Give variables, functions, and classes clear names that reveal their intent so you need fewer comments.
- **Reduce Nesting**: Keep your logic flat by using early returns and guard clauses instead of deeply indented `if-else` blocks.
- **Boy Scout Rule**: Always leave the code cleaner than you found it by making small, continuous improvements.
- **Always Find Root Cause**: Always look for the root cause of a problem instead of patching symptoms.

### Design Rules & Architecture
- **Keep Configurable Data at High Levels**: Keep constants, runtime configurations, and feature flags at top-level files or environment setup.
- **Prefer Polymorphism to `if/else` or `switch/case`**: Use strategies, object maps, or polymorphic objects rather than sprawling conditional chains.
- **Separate Multi-Threading / Async Code**: Isolate asynchronous worker logic and concurrency handlers from standard synchronous business logic.
- **Prevent Over-Configurability**: Avoid adding flags for options nobody will ever change.
- **Use Dependency Injection**: Inject services and dependencies rather than tightly coupling instantiations inside classes or composables.
- **Follow Law of Demeter**: A module or class should know only its direct dependencies (`a.getB()` instead of `a.getB().getC().doSomething()`).

### Understandability & Type Safety
- **Never Use `any`**: Always code with strict, explicit type definitions (`interface`, `type`, generics).
- **Be Consistent**: If you do something a certain way, do all similar things in the exact same way across the codebase.
- **Use Explanatory Variables**: Break complex boolean calculations or transformations into well-named intermediate variables.
- **Encapsulate Boundary Conditions**: Put the processing for boundary conditions in one dedicated place.
- **Prefer Dedicated Value Objects to Primitives**: Wrap complex primitive domain items in typed value objects/interfaces.
- **Avoid Logical Dependency**: Don't write methods whose correct operation secretly relies on state set by another method in the same class.
- **Avoid Negative Conditionals**: Prefer `if (isAvailable)` over `if (!isUnavailable)`.

### Naming Rules
- **Descriptive & Unambiguous**: Choose names that clearly describe purpose and content.
- **Make Meaningful Distinctions**: Avoid arbitrary variations like `userData1` vs `userData2`.
- **Use Pronounceable & Searchable Names**: Easy to speak aloud and quick to grep in an IDE.
- **Replace Magic Numbers with Named Constants**: Replace numbers like `86400000` with `const ONE_DAY_IN_MS = 86400000`.
- **Avoid Encodings**: Don't append Hungarian notation prefixes or redundant type prefixes.

### Functions & Methods Rules
- **Small**: Keep functions short and focused (< 20-30 lines).
- **Do One Thing**: Perform a single logical action cleanly.
- **Prefer Fewer Arguments**: Ideal arguments count is 0 to 2. Wrap 3+ arguments in a typed options object.
- **No Side Effects**: Do not secretly mutate global state or passed arguments.
- **Don't Use Flag Arguments**: Split boolean flag methods into independent methods (`doSomethingForAdmin` vs `doSomethingForGuest`).

### Code Comments Rules
- **Explain Yourself in Code**: Code should be self-documenting through clear structure and naming.
- **Don't Be Redundant or Obvious**: Avoid noise comments like `// increment i`.
- **No Closing Brace Comments**: Keep methods small enough that block ends are obvious.
- **Don't Comment Out Code**: Delete dead code immediately; source control tracks history.
- **Use Comments for Intent & Clarification**: Explain *why* unusual decisions were made or warn of non-obvious consequences.

### Source Code Structure
- **Separate Concepts Vertically**: Use blank lines to separate logical code blocks.
- **Vertical Density**: Related code lines should be tightly grouped vertically.
- **Declare Variables Close to Usage**: Instantiate variables right before consumption.
- **Dependent & Similar Functions Close**: Place caller functions directly above or near called functions.
- **Short Lines & Proper Indentation**: Keep lines < 100-120 characters without breaking indentation.

### Objects & Data Structures
- **Hide Internal Structure**: Expose abstract interfaces rather than internal implementation details.
- **Prefer Data Structures or Pure Objects**: Keep data structures distinct from object behavior.
- **Base Class Knowledge**: Base classes should know nothing about their derived subclasses.

### Testing Standards
- **One Assert Per Test**: Keep test assertions focused on a single concept per test case.
- **Readable, Fast, Independent, Repeatable**: Execute rapidly and document system behavior.

### Code Smells Matrix
- **Rigidity**: Small changes cause a cascade of subsequent changes across multiple files.
- **Fragility**: Single changes break code in unrelated places.
- **Immobility**: Code cannot be reused in other parts of the system due to high coupling.
- **Needless Complexity**: Overengineered abstractions for simple requirements.
- **Needless Repetition**: Duplicate blocks or patterns.
- **Opacity**: Code is cryptic and difficult to comprehend.

---

## 5. Type Safety Auditing & Placement Rules

### Type File Placement Rules (Non-Negotiable)
- If a type or interface is **5 lines or fewer**, it may live inline in the file that uses it.
- If a type or interface is **more than 5 lines**, it MUST go in a dedicated `.types.ts` file — never inline in a Vue SFC, composable, or server route.
- Create an `app/types/` or `shared/types/` folder at the appropriate layer if it doesn't exist.
- **Never create a new types file if a relevant one already exists** — update the existing file and import from it.
- Always import types explicitly: `import type { MyType } from '~/types/my-feature.types'`

### Type Fix Workflow
1. Run type/lint check -> capture output.
2. Categorize errors in a NeuroForge file.
3. For each error: analyze root cause — don't just add lazy type annotations.
4. If root cause is unclear, **pause and ask the user**.
5. Create or update `.types.ts` files as needed.
6. Re-run type check to confirm resolution.

---

## 6. Rules by File Type

### Composables
- Name MUST start with `use` — e.g. `useUserOrders.ts`, never `orders.ts`
- Accept `MaybeRefOrGetter` → use `toValue()` internally
- Return destructuring-friendly `readonly` refs
- Small and focused — `useFetchCart`, `useAddToCart`, `useRemoveFromCart`; not a monolithic `useCart`
- Move pure helpers OUTSIDE composable function (prevents closure memory leaks)
- Clean up: `onUnmounted` for listeners, timers, subscriptions

### Components (`<script setup lang="ts">`)
- Strict UI Casing, Structure & Tag Rules documented in [references/components.md].
- Strict `defineProps<{ ... }>()` and `defineEmits<{ ... }>()`
- UI + minimal orchestration only; complex logic goes to composables
- Never use browser APIs in `setup()` — use `onMounted()`, `useCookie`, or `useState`

### Nitro Server Routes
- `defineEventHandler` → validate → business logic (utils) → Prisma → typed response
- `readBody<MyType>()` and `getQuery<MyType>()` always typed
- Thin routes — extract logic to `server/utils/`
- Never expose raw Prisma errors to client

### Prisma
- Singleton in `server/utils/db.ts` or `server/database/client.ts` via `globalForPrisma`
- Precise `select`/`include` — never fetch more columns than needed
- Prisma-generated types everywhere; re-export from `types/` for client sharing
- Schema-first; `prisma generate` in `postinstall`
- Wrap errors — return typed, user-friendly responses

---

## 7. Code Review Style & Quality Verdict

When **reviewing**:
- Flag as: `🚩 Senior-level issue: <problem> → <exact fix>`
- Always show before/after code
- Explain WHY — reference official Nuxt 4 patterns

### Clean Codebase Verdict
Close reviews with an honest Code Quality Rating:

```
## ✅ Code Quality Verdict

**Rating: X / 10** — [one-line summary e.g. "Production-ready SaaS standard"]

[2–3 sentences of genuine, specific feedback on what's strong.]

**Strengths spotted:** [bullet list]
**Suggestions (if any):** [real improvements only]
```
