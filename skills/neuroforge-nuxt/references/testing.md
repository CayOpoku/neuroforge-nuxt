# Testing

Load when writing or fixing tests. Test the things that break; do not chase a coverage number.

---

## 1. Setup

```bash
pnpm add -D @nuxt/test-utils vitest @vue/test-utils happy-dom
```

```ts
// vitest.config.ts
import { defineVitestConfig } from '@nuxt/test-utils/config'

export default defineVitestConfig({
  test: { environment: 'nuxt' },
})
```

Add the scripts to `package.json`: `test` runs `vitest run`, `test:watch` runs `vitest`.

---

## 2. What is worth testing

| Priority | Target | Why |
| :--- | :--- | :--- |
| High | `server/utils/` business logic, pure helpers | No mocking needed, catches real bugs |
| High | Validation schemas — valid, invalid, and edge input | The contract everything else depends on |
| High | Auth guards: `requireUser`, `requireRole`, tenant scoping | A silent failure here is a breach |
| Medium | Composables with branching logic | Fast with `mountSuspended` |
| Medium | Nitro handlers end-to-end | Covers validation, handler, and error shape together |
| Low | Presentational components | Slow, brittle, low yield |
| Never | Shadcn primitives, framework behaviour | Not your code |

If a bug reaches production, a regression test for it is mandatory. That is the highest-value test you will ever write.

---

## 3. Patterns

```ts
// unit — pure logic, no Nuxt context needed
import { describe, expect, it } from 'vitest'
import { getErrorMessage } from '~/utils/error'

describe('getErrorMessage', () => {
  it('reads a FastAPI detail array', () => {
    expect(getErrorMessage({ data: { detail: [{ msg: 'Field required', loc: ['body', 'email'] }] } }))
      .toBe('Field required')
  })

  it('falls back to a transport message when there is no payload', () => {
    expect(getErrorMessage(new TypeError('Failed to fetch')))
      .toBe('Could not reach the server. Check your connection and try again.')
  })
})
```

```ts
// component — needs the Nuxt context
import { mountSuspended } from '@nuxt/test-utils/runtime'

it('renders the error state instead of a fallback value', async () => {
  const wrapper = await mountSuspended(OrderStatus, {
    props: { status: 'error', error: new Error('nope') },
  })
  expect(wrapper.text()).not.toContain('Pending')
})
```

```ts
// endpoint — a real request through Nitro
import { setup, $fetch } from '@nuxt/test-utils/e2e'

await setup({ server: true })

it('rejects an invalid body with 400', async () => {
  await expect($fetch('/api/orders', { method: 'POST', body: { email: 'nope' } }))
    .rejects.toMatchObject({ statusCode: 400 })
})
```

---

## 4. Rules

- One behaviour per test. The test name states the behaviour, not the function name.
- Independent and repeatable: no shared mutable state, no reliance on execution order, no real network calls.
- Assert the **contract**, not the implementation. A test that breaks on every refactor is a liability.
- Test the failure paths — invalid input, 401, 404, empty list, network error. Those are the ones that ship broken.
- Never weaken an assertion to make a test pass. Fix the code, or delete the test and say why.
