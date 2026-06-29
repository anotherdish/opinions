---
name: nextjs-testing
description: Test a Next.js App Router app — the unit-vs-E2E split, Vitest + React Testing Library for client components/Server Actions/Zod/utils, Playwright for E2E (auth flows, forms, streaming, and async Server Components which unit runners can't render), mocking with MSW, and what each layer should and shouldn't cover. Use when setting up testing, writing tests, deciding Vitest vs Playwright, testing Server Components/Actions, or fixing flaky/ill-fitting tests. Targets Next.js 16.
---

# Testing

The defining constraint of App Router testing: **async Server Components are not
reliably renderable in unit runners (Vitest/Jest) yet.** That single fact drives the
whole strategy — unit-test what runs as plain functions or client components, and
push everything server-rendered, cookie-, router-, or streaming-dependent into E2E.

> Targets **Next.js 16+**. Tooling (Vitest, Playwright, the `@next/playwright`
> helper) and async-RSC support move fast — verify against the installed versions;
> use the Context7 MCP (`/vercel/next.js`) for Next itself.

## Dogma

1. **Two tools, clear split.** **Vitest** (or Jest) + **React Testing Library** for
   units; **Playwright** for E2E. Don't try to make one do the other's job.
2. **Async Server Components → E2E.** Don't fight unit runners to render them; test
   them through Playwright against a real server.
3. **Test behavior, not implementation.** Query by role/text/label like a user
   (RTL); assert on what the user sees, not internal state or class names.
4. **Server Actions are just async functions** — unit-test their logic directly
   (validation, authz branches, return values) with the DB/auth mocked; test the
   full form→action→revalidate→UI loop in E2E.
5. **Don't mock what you're testing.** Mock the *network boundary* (MSW) and external
   services; keep your own code real.
6. **Few E2E, many units.** E2E is slow and flaky-prone — reserve it for critical
   flows; cover logic breadth in fast unit tests.

## The split — what goes where

| Test with **Vitest + RTL** (no real browser) | Test with **Playwright** (real browser/server) |
|---|---|
| Client Components (render, interactions) | Async **Server Components** (rendered output) |
| Server Action **logic** (as plain functions) | Full form submit → Server Action → revalidated UI |
| Zod schemas / validation | Auth/login flows, sessions, cookies |
| Utilities, hooks, formatters | `proxy.ts`/routing, redirects, protected routes |
| Pure presentational components | Streaming/Suspense, PPR, instant navigation |

## Vitest + React Testing Library

```bash
npm i -D vitest @vitejs/plugin-react jsdom \
  @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: { environment: 'jsdom', setupFiles: ['./vitest.setup.ts'], globals: true },
})
```

```ts
// vitest.setup.ts
import '@testing-library/jest-dom/vitest'
```

### Client component test

```tsx
// add-to-cart.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { AddToCart } from './add-to-cart'

test('increments quantity', async () => {
  render(<AddToCart productId="1" price={10} />)
  await userEvent.click(screen.getByRole('button', { name: '+' }))
  expect(screen.getByText('2')).toBeInTheDocument()
})
```

### Server Action logic (test as a function, mock the boundary)

```ts
// posts.test.ts
import { vi, test, expect, beforeEach } from 'vitest'
import { createPost } from '@/app/actions/posts'

vi.mock('@/lib/auth', () => ({ auth: vi.fn() }))
vi.mock('@/lib/db', () => ({ db: { post: { create: vi.fn() } } }))
vi.mock('next/cache', () => ({ updateTag: vi.fn(), revalidateTag: vi.fn() }))

import { auth } from '@/lib/auth'

beforeEach(() => vi.clearAllMocks())

test('rejects when unauthenticated', async () => {
  vi.mocked(auth).mockResolvedValue(null)
  const fd = new FormData()
  fd.set('title', 'Hi'); fd.set('content', 'There')
  const result = await createPost({ message: '' }, fd)
  expect(result.message).toMatch(/signed in/i)
})

test('returns field errors on invalid input', async () => {
  vi.mocked(auth).mockResolvedValue({ user: { id: 'u1' } } as any)
  const result = await createPost({ message: '' }, new FormData())
  expect(result.errors?.title).toBeTruthy()
})
```

> Async Server Components: Next's docs note unit runners don't support rendering
> them well. Extract testable logic into plain functions (DAL/lib), unit-test that,
> and verify the rendered component in Playwright.

## Playwright (E2E)

```bash
npm init playwright@latest
```

```ts
// playwright.config.ts (key bits)
import { defineConfig } from '@playwright/test'
export default defineConfig({
  testDir: './e2e',
  use: { baseURL: 'http://localhost:3000' },
  webServer: {
    command: 'npm run build && npm run start', // test the production build
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
})
```

### Flow test (Server Component output + interaction)

```ts
// e2e/blog.spec.ts
import { test, expect } from '@playwright/test'

test('renders posts and creates one', async ({ page }) => {
  await page.goto('/blog')
  await expect(page.getByRole('heading', { name: 'Our Blog' })).toBeVisible()

  await page.getByLabel('Title').fill('Hello')
  await page.getByLabel('Content').fill('World')
  await page.getByRole('button', { name: 'Publish' }).click()

  await expect(page.getByText('Hello')).toBeVisible() // server re-rendered after revalidate
})
```

### Streaming / instant navigation (Next 16 helper)

```ts
import { test, expect } from '@playwright/test'
import { instant } from '@next/playwright'

test('product title appears instantly', async ({ page }) => {
  await page.goto('/store/shoes')
  await instant(page, async () => {
    await page.click('a[href="/store/hats"]')
    await expect(page.locator('h1')).toContainText('Baseball Cap') // static shell
  })
  await expect(page.locator('text=in stock')).toBeVisible() // dynamic content streams in
})
```

### Auth flows

Test login/logout and protected routes end-to-end (cookies, `proxy.ts`, redirects).
Speed up suites by signing in once and reusing **storage state**:

```ts
// e2e/auth.setup.ts → save context.storageState({ path: 'e2e/.auth/user.json' })
// then in config: use: { storageState: 'e2e/.auth/user.json' }
```

## Mocking the network — MSW

Mock the network boundary, not your modules, so the same handlers serve unit and E2E.

```ts
// test/mocks/handlers.ts
import { http, HttpResponse } from 'msw'
export const handlers = [
  http.get('https://api.vercel.app/blog', () =>
    HttpResponse.json([{ id: 1, title: 'Mocked', author: 'T', date: '2026-01-01' }]),
  ),
]
```

## What to test (priorities)

- **Server Action logic**: every authz branch, validation failure, success path.
- **DAL/permissions**: `verifySession`, ownership checks, RBAC (`can(...)`).
- **Client interactions**: forms, optimistic UI, disabled/pending states.
- **Critical E2E journeys**: signup→login, the core create/edit/delete flow, payment.
- **Edge/regression**: each fixed bug gets a test.

## Anti-patterns

- ❌ Forcing Vitest/Jest to render async Server Components (unsupported → flaky).
- ❌ E2E-testing everything (slow, brittle) instead of unit-testing logic.
- ❌ Mocking your own functions so heavily the test asserts nothing real.
- ❌ Querying by CSS class / test-id when role/label/text would do (RTL guidance).
- ❌ Running E2E only against `next dev` (test the production `build`+`start`).
- ❌ Logging in via UI in every E2E test instead of reusing storage state.
- ❌ No test for auth/authz branches — the highest-risk code path.

## Cross-references

- Server vs Client (what's unit-testable) → `nextjs-server-client-components`
- Action logic, validation, return shapes → `nextjs-server-actions-and-forms`
- DAL/session/authz to test → `nextjs-auth-and-authorization`
- Streaming/PPR/instant navigation behavior → `nextjs-rendering-and-performance`
