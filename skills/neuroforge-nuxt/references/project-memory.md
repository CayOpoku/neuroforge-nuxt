# Project Memory & Session Management Protocol

Specification reference: [skills.sh handoff standard](https://www.skills.sh/mattpocock/skills/handoff)

---

## 1. Project Overview & Project Analysis Lifecycle

### `00-project-overview.md` (Core Project Memory)
- **Do NOT overwrite `00-project-overview.md`**. It serves as the master source of truth for the entire codebase architecture, database models, environment settings, and long-term decisions.
- **Always inspect `00-project-overview.md`** before starting work, and **update it incrementally** whenever new architectural components, routes, models, or patterns are introduced.

- **Always create task checklists/plans using default IDE task artifacts (`task.md`) before starting any implementation work** (do not put task artifacts inside `neuroforge/`).

---

## 2. Large Tasks & JS Automation Scripts

- For huge tasks (e.g., massive refactoring, multi-file code scans, complex schema migrations):
  - Do NOT burn context window tokens by manually inspecting or outputting thousands of lines in chat.
  - Perform a deep search and **write a Node/JS script** inside a `scripts/` or `test-scripts/` directory.
  - Present the script to the user for review so they can run it locally, saving time and tokens.

---

## 3. Token Burn Warning & Handoff Protocol

### Token & Session Bloat Warnings
- Monitor conversation length and context bloat.
- When context window size becomes large or token usage increases significantly, **alert the user** and offer to generate a clean handoff document.

### Handoff Document Specification (`https://www.skills.sh/mattpocock/skills/handoff`)
When creating a handoff document:
1. **Save Path**: Save the handoff file directly to the user's OS temporary directory (e.g. `%TEMP%` on Windows or `/tmp` on POSIX) — **never inside the workspace repo**.
2. **Suggested Skills Section**: Must include a "Suggested Skills" section listing the exact skill names a fresh agent should call.
3. **No Duplication**: Reference existing artifacts, commits, plans, issues, and diffs by relative path or URL rather than duplicating raw content.
4. **Redact Sensitive Info**: Strip all secrets, API keys, passwords, and PII.
5. **Tailor to Next Focus**: Incorporate any user input describing what the next session will focus on.

---

## 4. Pruning Bloated `neuroforge/` Files

- The `neuroforge/` folder can become bloated over time with finished task plans, old UI reviews, and historical audits.
- **Always ask the user for permission** before pruning or removing completed task files from `neuroforge/`.
- Never delete files silently without user consent.
