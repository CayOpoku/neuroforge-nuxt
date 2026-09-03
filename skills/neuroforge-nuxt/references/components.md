# Vue Component Casing, Folder Structures & Reusability Rules

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
    <app-button @click="submit">
      Submit Entry
    </app-button>
    
    <!-- Exception: Raw Shadcn UI primitives remain PascalCase when inside wrappers -->
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

## 2. Scalable Shadcn UI Wrapper Strategy (`components/app/`)

### Write Once, Use Everywhere
Instead of copying and pasting dozens of lines of raw Shadcn UI primitives into individual page files (making customization, styling consistency, and updates difficult):

- **Build Reusable Wrappers**: Create custom application wrapper components inside `components/app/` (e.g. `components/app/button.vue`, `components/app/dialog.vue`, `components/app/input.vue`, `components/app/datatable.vue`).
- **Auto-Import Names**: Placing them in `components/app/` automatically generates clean, conflict-free auto-import tags:
  - `<app-button>`
  - `<app-dialog>`
  - `<app-input>`
  - `<app-datatable>`
- **Pass Props & Slots**: Expose flexible props, variants, sizes, and slots on the `app-*` wrapper while hiding repetitive internal Shadcn markup.
- **Project Component Check**: Before writing any new UI element in a page, always inspect `components/app/` or existing component folders to re-use established project components.

---

## 3. Component Naming & Directory Organization

### Domain-Driven Naming (Not Bounded by Pages)
Never name or structure components after specific pages or routes (e.g. `home-hero.vue`, `about-card.vue`). Instead, organize components by domain/feature capabilities:

```
components/
  app/           → Reusable app wrappers (<app-button />, <app-dialog />, <app-datatable />)
  ui/            → Raw Shadcn UI primitives ONLY (never put custom components here)
  content/       → Content display widgets (<content-card />, <content-list />)
  form/          → Form input components (<form-otp />, <form-select />)
  datatable/     → Table cells and filters (<datatable-filter />, <datatable-pagination />)
```

---

## 4. External Templates Directory (`templates/`)

Never embed raw HTML email templates, complex static data structures, or PDF download layouts directly inside Vue component files or Nitro handlers:

- **Create `templates/` Folder**: Store HTML/mustache templates in `templates/email/`, `templates/pdf/`, or `templates/export/`.
- **Import into Vue / Nitro**: Load and compile template files using utility functions or file readers.

---

## 5. Image Placeholder Standards

Whenever adding image placeholders to component templates, mockups, or UI layouts:
- Use standard placehold.co images with responsive CSS classes:
  ```html
  <img src="https://placehold.co/1500x1500" alt="Placeholder" class="w-full h-full object-cover" />
  ```
- Adjust dimensions in the URL as needed (e.g. `https://placehold.co/600x400`, `https://placehold.co/150x150`) matching the container aspect ratio.
- Never use broken local relative paths or missing asset references for UI placeholders.
