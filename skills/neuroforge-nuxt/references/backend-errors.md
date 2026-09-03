# Backend Error Contracts & Multi-Backend Error Handling Protocol

---

## 1. Universal Zero-Hardcoded API Error Principle

- **Frontend Must Never Create API Error Messages**: Never hardcode generic or speculative error strings (e.g. `"An unexpected error occurred"`, `"Failed to fetch data"`, `"Operation failed"`) inside Vue SFCs, composables, or form handlers.
- **Backend Origin Mandatory**: All user-facing error messages **must originate from the primary backend API** — regardless of whether the backend service is **FastAPI, NestJS, Express/Node.js, Go, Python, or Ruby**.
- **Nitro Proxy Transparency**: Even when Nitro server routes act as proxies or BFF layers, Nitro must forward the underlying main backend error payload intact rather than replacing it with synthetic generic errors.
- **Code Audit Rule**: During code analysis, sniff out any hardcoded error strings in `catch` blocks or `$fetch` error handling, and advise the developer to remove them in favor of backend-driven error extraction.

---

## 2. Universal `getErrorMessage` Utility

Every project must maintain a flexible `getErrorMessage` utility in `app/utils/error.ts` (or `shared/utils/error.ts`) capable of parsing error responses across diverse backend frameworks (FastAPI detail arrays/strings, NestJS message arrays/strings, standard HTTP status text):

```ts
// app/utils/error.ts

export interface BackendErrorResponse {
  message?: string | string[];
  detail?: string | Array<{ msg: string; loc: string[] }>;
  error?: string;
  statusCode?: number;
  statusMessage?: string;
  data?: any;
}

export function getErrorMessage(error: unknown): string {
  if (!error) return "An unknown error occurred.";

  if (typeof error === "object" && error !== null) {
    const err = error as Record<string, any>;

    // 1. Nuxt / Nitro / $fetch wrapped response data
    const responseData: BackendErrorResponse = err.data || err.response?._data || err;

    // 2. FastAPI error format ({ detail: "..." } or { detail: [{ msg: "..." }] })
    if (responseData.detail) {
      if (typeof responseData.detail === "string") return responseData.detail;
      if (Array.isArray(responseData.detail) && responseData.detail[0]?.msg) {
        return responseData.detail.map((d) => d.msg).join(", ");
      }
    }

    // 3. NestJS / Express error format ({ message: "..." } or { message: ["..."] })
    if (responseData.message) {
      if (typeof responseData.message === "string") return responseData.message;
      if (Array.isArray(responseData.message)) return responseData.message.join(", ");
    }

    // 4. Standard H3 / Nitro statusMessage or top-level message
    if (responseData.statusMessage) return responseData.statusMessage;
    if (err.message && typeof err.message === "string") return err.message;
  }

  return typeof error === "string" ? error : String(error);
}
```

---

## 3. Toast Notification Integration (`catch` blocks)

Always pass extracted backend messages directly into Shadcn / Vue toast notifications inside component action handlers:

```ts
// Example: Form submission / Action handler in Vue script setup
try {
  await $fetch('/api/users/profile', {
    method: 'POST',
    body: formData.value,
  });
  toast.success("Profile updated successfully!");
} catch (error) {
  // Extract exact backend error and display in Toast notification
  toast.error(getErrorMessage(error));
}
```

---

## 4. Fallback Smells & Error Transparency

- **Frontend Fallback Smell**: Defaulting UI state when an API call fails (e.g. setting `status = 'Pending'` or `role = 'User'` upon fetch failure) takes decision power away from the user and disguises system issues.
- **Rule**: Expose API failure states explicitly. Render the backend error payload directly using `<app-alert>` or Shadcn toast so both users and developers have full visibility into the exact error reason.
