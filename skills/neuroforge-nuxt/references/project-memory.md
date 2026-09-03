# Project Memory & Session Management

Applies to **Tier 2** work. Tier 0 and Tier 1 tasks create no memory files — see the triage gate in `SKILL.md`.

Handoff specification: [skills.sh handoff standard](https://www.skills.sh/mattpocock/skills/handoff)

---

## 1. `00-project-overview.md` — the core record

- **Never overwrite it.** It is the master record of architecture, database models, environment configuration, and long-term decisions.
- **Read it first** on any Tier 2 task, before scanning the codebase — it usually answers questions a scan would cost thousands of tokens to re-derive.
- **Update it incrementally** whenever a new model, route group, layer, or architectural pattern is introduced. Append; do not rewrite history.
- If it does not exist and the task is onboarding, create it.

## 2. Task pre-analysis files

- Task-specific files (`01-project-analysis.md`, `02-architecture-decisions.md`, …) may be created or overwritten freely — they are scoped to the current task.
- Create only the ones the task needs. Producing five files for a two-file change is the exact token waste this system exists to prevent.
- Task checklists go in the IDE's own task artifact (`task.md`), **not** in `neuroforge/`.

---

## 3. Large tasks: write a script instead of reading

For a massive refactor, a multi-file scan, or a bulk migration, do not read thousands of lines into context.

Write a Node script into `scripts/` or `test-scripts/` that performs the scan or transformation, present it for review, and let the user run it. The script is cheaper than the context, reviewable before it runs, and repeatable.

---

## 4. Token burn and handoff

Watch session length. When context is getting heavy, say so and offer a handoff document rather than degrading quietly.

A handoff document:

1. **Saves to the OS temp directory** (`%TEMP%` on Windows, `/tmp` on POSIX) — never inside the repo.
2. **Includes a "Suggested Skills" section** naming the exact skills a fresh agent should invoke — `neuroforge-nuxt` plus any companion skill this work depends on (e.g. `vueuse/skills` if the session leaned on VueUse), so the next agent does not re-derive which tools it needs.
3. **References, never duplicates.** Point at files, commits, and `neuroforge/` paths instead of pasting their content.
4. **Redacts everything sensitive** — secrets, keys, tokens, PII.
5. **States what the next session is for**, incorporating whatever the user says the next focus is.

---

## 5. Pruning

`neuroforge/` accumulates finished plans, old UI reviews, and superseded audits.

**Always ask before pruning.** Never delete a memory file silently. When a file is superseded, version it (`03-v2-composable-strategy.md`) and leave the original in place until the user says otherwise.
