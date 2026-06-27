---
name: nextjs-server-actions-and-forms
description: Mutate data in the Next.js App Router with Server Actions and forms — defining 'use server' actions, wiring forms with useActionState/useFormStatus, validating input (zod), authentication/authorization checks, revalidating caches after writes (updateTag/revalidateTag), redirects, optimistic UI with useOptimistic, and progressive enhancement. Use whenever handling form submissions, button-triggered mutations, create/update/delete flows, or "how do I save data" questions. Targets Next.js 16 + React 19.2.
---

# Server Actions & Forms

How to **write** data. Server Actions are async functions that run on the server and
are callable from client/server components and `<form action>` — they replace
hand-rolled API routes for mutations. This is the canonical mutation path in the
App Router.

## Dogma

1. **Mutations are Server Actions, not API routes.** Don't build `app/api/*` +
   `fetch` for your own UI's writes.
2. **Every action is a trust boundary.** A Server Action is a public HTTP endpoint.
   **Always** re-check authentication and authorization inside it, and **always**
   validate input. Never trust the caller.
3. **Validate input with a schema** (zod or similar) at the top of every action.
4. **Revalidate after a successful write** so the UI reflects the new state
   (`updateTag`/`revalidateTag`/`revalidatePath`).
5. **Prefer the native `<form action={...}>`** so forms work before/without JS
   (progressive enhancement); layer `useActionState`/`useOptimistic` on top.
6. **Return errors, throw for the unexpected.** Return typed validation/business
   errors as state; throw only for truly exceptional cases (caught by `error.tsx`).

## Defining a Server Action

Mark a function or file with `'use server'`. Either inline in a Server Component, or
(preferred for reuse/testing) in a dedicated module.

```ts
// app/actions/posts.ts
'use server'

import { z } from 'zod'
import { revalidateTag, updateTag } from 'next/cache'
import { redirect } from 'next/navigation'
import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

const CreatePost = z.object({
  title: z.string().min(1).max(120),
  content: z.string().min(1),
})

export type FormState = { message: string; errors?: Record<string, string[]> }

export async function createPost(
  _prev: FormState,
  formData: FormData,
): Promise<FormState> {
  // 1) AuthN
  const session = await auth()
  if (!session?.user) return { message: 'You must be signed in.' }

  // 2) Validate
  const parsed = CreatePost.safeParse(Object.fromEntries(formData))
  if (!parsed.success) {
    return {
      message: 'Please fix the errors below.',
      errors: parsed.error.flatten().fieldErrors,
    }
  }

  // 3) Mutate
  await db.post.create({ data: { ...parsed.data, authorId: session.user.id } })

  // 4) Revalidate so readers see the change
  updateTag('posts') // read-your-writes; author sees it immediately

  // 5) Redirect (must be OUTSIDE try/catch — it throws internally)
  redirect('/blog')
}
```

## Wiring a form (the standard pattern)

`useActionState` connects an action to a form, exposes its return value as `state`,
and gives a `pending` flag. `useFormStatus` reads pending status for a nested submit
button without prop-drilling.

```tsx
// app/blog/_components/post-form.tsx
'use client'
import { useActionState } from 'react'
import { useFormStatus } from 'react-dom'
import { createPost, type FormState } from '@/app/actions/posts'

const initial: FormState = { message: '' }

function SubmitButton() {
  const { pending } = useFormStatus()
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Publishing…' : 'Publish'}
    </button>
  )
}

export function PostForm() {
  const [state, formAction] = useActionState(createPost, initial)
  return (
    <form action={formAction}>
      <label htmlFor="title">Title</label>
      <input id="title" name="title" required />
      {state.errors?.title && <p role="alert">{state.errors.title[0]}</p>}

      <label htmlFor="content">Content</label>
      <textarea id="content" name="content" required />
      {state.errors?.content && <p role="alert">{state.errors.content[0]}</p>}

      <p aria-live="polite">{state.message}</p>
      <SubmitButton />
    </form>
  )
}
```

Notes:
- The action signature for `useActionState` is `(prevState, formData) => newState`.
- This **works without JavaScript**: the `<form action>` submits to the server, and
  enhances to client-side handling once hydrated.
- `name` attributes on inputs become `FormData` keys — no controlled state needed
  for plain forms.

## Actions outside forms (buttons / programmatic)

Bind arguments and call from an event handler or `formAction`. Use `useTransition`
to track pending state.

```tsx
'use client'
import { useTransition } from 'react'
import { deletePost } from '@/app/actions/posts'

export function DeleteButton({ id }: { id: string }) {
  const [pending, startTransition] = useTransition()
  return (
    <button
      disabled={pending}
      onClick={() => startTransition(async () => { await deletePost(id) })}
    >
      {pending ? 'Deleting…' : 'Delete'}
    </button>
  )
}
```

Or bind args directly: `<form action={deletePost.bind(null, id)}>`.

## Security — the rules that matter

Server Actions are reachable by anyone who can hit your server. Treat each one like
a public POST endpoint:

```ts
'use server'
export async function deletePost(postId: string) {
  const session = await auth()
  if (!session?.user) throw new Error('Unauthorized')          // authN

  const post = await db.post.findUnique({ where: { id: postId } })
  if (post?.authorId !== session.user.id) throw new Error('Forbidden') // authZ — prevents IDOR

  await db.post.delete({ where: { id: postId } })
  revalidateTag('posts', 'max')
}
```

- **AuthN then AuthZ, every action.** Verify the session, then verify the user may
  act on *this specific resource* (prevents Insecure Direct Object Reference).
- **Validate and coerce all input** — never trust `formData`/args shapes.
- **Don't return secrets or internal errors** to the client.
- Don't rely on `proxy.ts` for action authorization — enforce it in the action.

## Revalidation after writes

Pick based on the freshness semantics you want (full table in the caching skill):

- `updateTag('posts')` — user must see their own change now (read-your-writes).
- `revalidateTag('posts', 'max')` — fine to be eventually consistent (SWR; **2nd
  arg required in 16**).
- `revalidatePath('/blog')` — invalidate a whole route subtree.
- `refresh()` — refresh **uncached** dynamic data shown elsewhere on the page,
  without touching the cache.

## Redirects

```ts
import { redirect } from 'next/navigation'
// ...
redirect('/blog') // throws a special signal — keep OUTSIDE try/catch
```

`redirect()` works inside Server Actions and Server Components. Inside a `try/catch`
it would be swallowed — call it after the work succeeds.

## Optimistic UI with `useOptimistic`

Show the result instantly, reconcile when the action resolves.

```tsx
'use client'
import { useOptimistic } from 'react'
import { addTodo } from '@/app/actions/todos'

export function Todos({ todos }: { todos: Todo[] }) {
  const [optimistic, addOptimistic] = useOptimistic(
    todos,
    (state, newTitle: string) => [...state, { id: 'temp', title: newTitle, pending: true }],
  )
  return (
    <>
      <ul>{optimistic.map((t) => <li key={t.id} aria-busy={t.pending}>{t.title}</li>)}</ul>
      <form
        action={async (formData) => {
          const title = String(formData.get('title'))
          addOptimistic(title)        // instant
          await addTodo(title)        // server reconciles + revalidates
        }}
      >
        <input name="title" />
        <button type="submit">Add</button>
      </form>
    </>
  )
}
```

## Progressive enhancement

- Plain `<form action={serverAction}>` submits and works with JS disabled / before
  hydration.
- Keep critical inputs as uncontrolled `name`d fields; reserve client state for
  enhancement (live validation, optimistic updates).
- `useFormStatus` / `pending` give feedback only once hydrated — the form still
  functions without it.

## Anti-patterns

- ❌ Building Route Handlers + client `fetch` for your own UI's mutations.
- ❌ Skipping authN/authZ inside an action ("the UI already hides the button").
- ❌ Trusting `formData`/args without schema validation.
- ❌ `redirect()` inside `try/catch` (swallows the redirect).
- ❌ Forgetting to revalidate, leaving stale UI after a write.
- ❌ Fully controlled forms with `useState` for everything (breaks progressive
  enhancement; usually unnecessary).
- ❌ `revalidateTag('x')` single-arg in 16 (deprecated — add a profile or use
  `updateTag`).

## Cross-references

- Where action modules / Route Handlers live → `nextjs-app-architecture`
- Client boundary for forms, serializable props → `nextjs-server-client-components`
- Which revalidation API to choose, caching model → `nextjs-data-fetching-and-caching`
- Form state vs URL/local/global state, where state lives → `nextjs-state-management`
- Sessions, the DAL, RBAC, defense-in-depth → `nextjs-auth-and-authorization`
- Unit-testing action logic, E2E form flows → `nextjs-testing`
- `error.tsx` for thrown errors, pending UX, streaming → `nextjs-rendering-and-performance`
