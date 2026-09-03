# Simple Developer IDE Workflow

This workflow ensures your AI assistant generates clean, type-safe code rather than "sloppy" solutions. It follows a simple cycle: **Analyze ➔ Mockup ➔ Code ➔ Verify**.

---

## The 5-Step Process

### 1. Setup & Trigger

- **Start the Conversation:** Tell the AI your tech stack (e.g., Nuxt 4, Nitro, Prisma, TS).
- **Prepare the Memory Folder:** Create a folder for memory logs (e.g., `memory/` or `project-brain/`) in the root directory. Add this folder to your `.gitignore` so temporary draft files are not committed to Git.
- **Optional Shortcut:** Install this pre-built skill for that by running:
  ```bash
  npx skills add https://github.com/cayopoku/neuroforge-nuxt --skill neuroforge-nuxt
  ```

---

### 2. Analysis First (No Coding Yet)

- **Codebase Onboarding:** Let the AI create `.md` files to document and build a better understanding of the project's folder structure, boundaries, and overall design patterns.
- **Trello & Ticket Planning:** When a new ticket or task is assigned (e.g., from Trello), have the AI create a planning `.md` file in your memory folder detailing exactly how it will go about solving it (steps, dependencies, database schema updates).
- **Plan Before Code:** Before writing any actual code, the AI must draft these design plans inside your memory folder.
- **Consent Check:** The AI must present the plan and **wait for you to say "Proceed" or "Start coding"** before modifying any project files.

---

### 3. UI Reviews (For Design Changes)

- **Create Standalone Mockups:** If building user interfaces, create standalone HTML files in a folder like `memory/ui-reviews/` (e.g., `features.html`).
- **Verify in Browser:** Open the file directly in a web browser to review CSS styles, layout structures, and responsiveness before converting it into Vue components.

---

### 4. Implementation & Quality Checks

Once authorized, build out the code adhering to these clean standards:

- **Types:** Keep types longer than 5 lines in separate `.types.ts` files. Avoid using `any` or `unknown`.
- **Pre-PR Check:** Run `npx nuxi typecheck` before opening a Pull Request.

---

### 5. Code Quality Rating

- After completing the feature, you can ask the AI to provide a final review and score the implementation from **1 to 10** with clear notes on what is strong and what could be improved.
