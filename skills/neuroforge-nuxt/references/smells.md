# Codebase Smells, Audits & Quality Guards

---

## 1. Dormant Files Smell

- Codebases frequently accumulate orphaned components, unused composables, dead utility functions, and outdated config files that serve no purpose.
- **Rule**: During initial NeuroForge analysis (`01-project-analysis.md` / `05-potential-spaghetti-risks.md`), actively audit for dormant files and point them out to the user with recommendations for removal.

---

## 2. Zero Legacy Patches in Development

- **No Quick Hacks / Legacy Patches**: When building in development with zero live production users, do NOT apply quick legacy patches, Band-Aids, or temporary workarounds to satisfy flawed backend contracts or bad component structures.
- **Rule**: Always recommend clean, scalable, airtight solutions first. If a backend change is needed, inform the user clearly so they can update the backend API rather than polluting the frontend codebase.

---

## 3. Strict Environment Variable Auditing

- **No Hardcoded Env Secrets**: Hardcoded API keys, URLs, or secrets in source code are critical security and architectural red flags.
- **Rely Less on Fallbacks**: Avoid hiding missing env variables behind silent inline fallback strings (e.g. `process.env.API_URL || 'http://localhost:3000'`).
- **Rule**: If mandatory environment variables are missing, throw a descriptive error during runtime/startup to force proper environment configuration:

```ts
// server/utils/env.ts
export function useRequiredEnv(key: string): string {
  const val = process.env[key] || useRuntimeConfig()[key];
  if (!val) {
    throw new Error(`[CRITICAL ENV MISSING] Mandatory environment variable '${key}' is not set.`);
  }
  return val;
}
```

---

## 4. Summary of Code Smells to Flag in NeuroForge Analysis

1. **Dormant Files**: Unused files cluttering directories.
2. **Hardcoded Secrets & Silenced Env Vars**: Inline secrets or silent fallbacks for missing env keys.
3. **Legacy Patch Smells**: Temporary frontend hacks to bypass backend defects.
4. **False Fallback Smells**: Frontend state defaults hiding backend errors.
5. **Generic Error Message Smells**: Hardcoded strings like `"Something went wrong"` instead of rendering backend API responses.
6. **God Component Smells**: Components exceeding 200 lines or mixing page routing logic with visual presentation.
