# Vue Component Casing, Folder Structures & Organization Rules

Use this reference whenever building, modifying, or refactoring Vue components in Nuxt 4.

---

## 1. Single File Component (SFC) Tag Order

Every Vue component file MUST follow a strict parent tag hierarchy. The `<template>` tag must always go first to prioritize layout readability, followed by TypeScript script setup, and ending with styling:

```html
<!-- 1. Template tag first -->
<template>
  <div class="card-wrapper">
    <h2 class="title">{{ title }}</h2>
    <!-- Standard tags & custom tags in kebab-case -->
    <base-button @click="submit">
      Submit Entry
    </base-button>
    
    <!-- Exception: Shadcn UI tags remain PascalCase -->
    <Dialog>
      <DialogContent>Confirm submission?</DialogContent>
    </Dialog>
  </div>
</template>

<!-- 2. Script setup second -->
<script setup lang="ts">
defineProps<{ title: string }>()
const emit = defineEmits<{ (e: 'submit'): void }>()
const submit = () => emit('submit')
</script>

<!-- 3. Style tag third (vanilla CSS scoped) -->
<style scoped>
.card-wrapper {
  padding: 1.5rem;
  background-color: var(--color-slate-900);
  border-radius: 0.5rem;
}
.title {
  font-size: 1.25rem;
  font-weight: bold;
}
</style>
```

---

## 2. Naming & Casing Conventions

### File Naming
*   All Vue Single File Component (`.vue`) file names MUST be written in **`kebab-case`** (e.g. `user-card.vue`, `order-status-badge.vue`).
*   Never use `PascalCase` or `camelCase` for Vue component files.

### Component Tags
*   All standard HTML tags and custom component tags inside a `<template>` MUST be written in **`kebab-case`** (e.g., `<user-card />`, `<base-button />`).
*   **Exception**: **Shadcn UI components** (e.g. `<Button />`, `<Dialog />`, `<FormItem />`, `<FormLabel />`) are permitted to remain in **`PascalCase`** to stay consistent with standard Shadcn UI conventions.

---

## 3. Nuxt Component Folder Structure & Auto-Imports

Nuxt auto-imports components by resolving their nested directories, concatenating directory names to form the component name. 

### Avoid Redundant Filenames
Never repeat the directory name inside the filename. 

*   **❌ BAD Structure**:
    *   `components/example/example-form.vue`
    *   *Resulting Auto-Import tag*: `<example-example-form />` or custom override. (Redundant and cluttered).
*   **✅ GOOD Structure**:
    *   `components/example/form.vue`
    *   *Resulting Auto-Import tag*: `<example-form />` (Clean, intuitive, and leverages Nuxt's auto-import perfectly).

### Shadcn UI Custom Folder Rule
*   Custom developed components **must never** be placed in the `components/ui/` folder. 
*   Keep `components/ui/` strictly reserved for un-customized Shadcn UI primitives. Put custom components in feature-sliced directories.

---

## 4. Prefix Folder Grouping Strategy

When multiple files would share a starting prefix name, they should be grouped into a directory named after that prefix rather than kept flat.

*   **❌ BAD (Flat Prefix)**:
    ```
    components/
      form-otp.vue
      form-input.vue
      form-select.vue
    ```
*   **✅ GOOD (Prefix Folder Grouped)**:
    ```
    components/
      form/
        otp.vue     → auto-imported as <form-otp />
        input.vue   → auto-imported as <form-input />
        select.vue  → auto-imported as <form-select />
    ```
    *This organizes your directories visually while generating the exact same clean component tags in templates.*
