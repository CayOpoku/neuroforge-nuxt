# Laws of UX & User Interface Standards

Reference: [Laws of UX](https://lawsofux.com/) & Official Nuxt Guidelines.

---

## 1. UX Improvement & Interactive "Grilling" Session Protocol

When requested to improve, audit, or redesign UX:
1. **Initiate an interactive interview session** with the user (a "grilling session") to challenge assumptions, clarify target personas, evaluate visual hierarchy, and align on design decisions.
2. **Reference core Laws of UX principles**:
   - **Fitts's Law**: Touch/click targets must be appropriately sized and placed near action hotspots.
   - **Hick's Law**: Reduce cognitive load by minimizing choices in critical user paths.
   - **Jakob's Law**: Users expect your app to work like other familiar apps. Avoid reinventing standard patterns without clear reason.
   - **Miller's Law**: Organize information into logical chunks (5-9 items per group).
   - **Peak-End Rule**: Ensure key action completions (success states, error recovery) feel rewarding and explicit.
   - **Postel's Law**: Be liberal in what you accept, conservative in what you send (flexible inputs, forgiving form design).
   - **Doherty Threshold**: Keep feedback snappy (<400ms) with visible skeletons or progress indicators.

---

## 2. Mandatory Visual & UI Design Rules

- **Zero Emojis in Production UI**: Never use raw emojis (e.g. 🚀, 💡, ⚠️, ❌) in UI code or component templates. Always use clean SVG icons (e.g. Lucide Icons / `@nuxt/icon`) for visual elements.
- **Default to Light Mode**: Never force dark mode or make dark mode the default unless the user explicitly requests it.
- **No False Fallbacks / Presumptuous UI**:
  - Never hide UI states behind arbitrary default fallback values that mask what is happening.
  - If a status fails to load, do NOT display a placeholder status like "Pending" or "Active". Show the explicit error state so the user knows an issue occurred.
  - Never make decisions for the user through silent fallbacks.

---

## 3. Transparency & Error Visibility

- Frontend UI must **never leave the user guessing what went wrong**.
- Always display exact, human-readable error feedback provided by the backend API.
- If the backend returns a structured error payload (e.g. validation issues, missing parameters), render those details clearly in the UI.
