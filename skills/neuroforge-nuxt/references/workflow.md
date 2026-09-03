# NeuroForge Tier 2 Protocol & Review Standards

Load this only for **Tier 2** work (new feature, schema change, refactor >4 files, architecture/UX audit, project onboarding). Tier 0 and Tier 1 tasks must not create memory files.

---

## 1. The Tier 2 memory cycle

### Step 1 — Prepare the memory layer
- Create `neuroforge/` if absent.
- Ensure `neuroforge/` is in `.gitignore`. If `.gitignore` is missing, create it. **Say in one line that you did this** — it modifies a tracked file, so it is announced, not silent.
- `neuroforge/ui-reviews/` holds standalone HTML/MD mockups.
- Automation, seed and one-off scripts go in `scripts/` or `test-scripts/`, never the repo root.
- Task checklists go in the IDE's own task artifact (`task.md`), **not** inside `neuroforge/`.

### Step 2 — Read before writing
Read `neuroforge/00-project-overview.md` first if it exists. It is the master record of architecture, models, env, and long-term decisions. **Update it incrementally; never overwrite it.** If it does not exist and this is an onboarding task, create it.

### Step 3 — Write focused analysis files
Create only the files the task actually needs — **match the count to the complexity, do not produce all five by reflex**:

- `01-project-analysis.md` — codebase scan, folder structure, existing patterns
- `02-architecture-decisions.md` — layers, routing, rendering strategy
- `03-composable-strategy.md` — what exists, what to split or merge
- `04-prisma-type-safety.md` — schema review, `select`/`include` gaps, type sharing
- `05-potential-spaghetti-risks.md` — god components, duplicated logic, prop drilling

Each file must be dense and high signal: sections, bullets, illustrative (non-executable) snippets, and the **rationale** — why, not just what. Apply a DRY / KISS / YAGNI / SRP lens explicitly.

**Obey verbatim during this step:** _"Do not write or edit any code. Focus exclusively on scanning the system to create the NeuroForge md files for the user to review."_

### Step 4 — Present and wait
Present the files. Write no implementation code until the user says "Proceed" or "Start coding".

---

## 2. Non-negotiable rules

1. **Never delete or silently overwrite a `neuroforge/` file.** Supersede it with a version (`03-v2-composable-strategy.md`) and say what you did. Ask before pruning finished analysis files.
   *This protects the memory layer only.* Deleting dead application code, stale files, or temporary `console.log` lines during an authorised refactor is expected — that is the Boy Scout rule, not a destructive action.
2. **Protected files need explicit approval:** `.env`, `.env.*`, `.git/*`, `prisma/migrations/*`, `package-lock.json`, `pnpm-lock.yaml`, auth/secret config. Explain what and why, then wait.
3. **No implementation before consent** on Tier 2.
4. **Verify every action.** Re-read or stat a file after writing it. Never report a change you did not confirm.
5. **Single source of truth.** `neuroforge/` merges technical decisions with SaaS product requirements.
6. **Design for pause/resume.** The files are checkpoints — summarise current state before pausing.
7. **Compact errors only:** root cause + impact + proposed fix.
8. **If you don't know, say so.** Present what you do know, ask what the user has seen. Never invent a fix or an API.

---

## 3. Context engineering guardrails

1. **Own the context window.** Keep memory files dense; strip noise. Never re-read a file already in context.
2. **Tools are structured outputs.** Think `{ goal, constraints, options, chosen_path, rationale }`.
3. **Explicit control flow:** Analyse → Document → Present → Wait → Execute → Verify.
4. **One concern per file.** Narrow responsibility per markdown file and per code module.
5. **Contact humans through files.** Surface trade-offs and open questions in `neuroforge/`, not in long chat messages.
6. **Stateless reasoning.** `neuroforge/` carries the accumulated state, not the transcript.

---

## 4. Engineering standards that are actually enforced here

Generic clean-code theory is assumed knowledge. These are the deltas this codebase holds you to:

- **SRP per file.** A component over ~200 lines, or one mixing route orchestration with presentation, gets split.
- **Composable granularity.** `useFetchCart` / `useAddToCart` / `useRemoveFromCart` — never one monolithic `useCart`.
- **DRY at the wrapper level.** Repeated Shadcn primitive blocks become an `app-*` wrapper (see `components.md`).
- **YAGNI.** No config flag, abstraction layer, or generic helper for a case that does not exist yet.
- **Flat logic.** Early returns and guard clauses over nested `if/else`. Object maps over long `switch` chains.
- **Named constants.** No magic numbers — `const ONE_DAY_IN_MS = 86_400_000`.
- **Positive conditionals.** `if (isAvailable)` over `if (!isUnavailable)`.
- **Config at the edges.** Runtime config, feature flags and constants live at the top level, never buried in a component.
- **Comments explain *why*.** No commented-out code, no restating the line below.
- **Consistency beats preference.** Match the surrounding file's existing idiom even if you would write it differently.

### Smells to name explicitly in a review
**Rigidity** (one change cascades) · **Fragility** (a change breaks something unrelated) · **Immobility** (can't reuse due to coupling) · **Needless complexity** · **Needless repetition** · **Opacity**.

---

## 5. Code review style

Flag findings as: `Senior-level issue: <problem> → <exact fix>`

Always show before/after, and explain **why** with reference to the official Nuxt 4 pattern. Close with an honest verdict:

```
## Code Quality Verdict

**Rating: X / 10** — [one-line summary]

[2-3 sentences of specific feedback on what is genuinely strong.]

**Strengths:** [bullets]
**Fix first:** [real, ranked improvements only — omit this section if there are none]
```

A rating is only useful if it can be low. Do not inflate.
