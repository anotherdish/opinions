---
name: nextjs-auth-and-authorization
description: Add authentication and authorization to a Next.js App Router app the secure way — sessions (encrypted cookies), the Data Access Layer (DAL) pattern, defense-in-depth (proxy.ts is not a security boundary), verifying sessions in Server Components/Actions/Route Handlers, role-based access control, DTOs to avoid leaking data, and choosing a library (Auth.js v5, Clerk, WorkOS). Use when adding login/signup, protecting routes/data, checking permissions, handling sessions/cookies, or fixing auth bypass/IDOR issues. Targets Next.js 16.
---

# Auth & Authorization

Authentication = *who are you*. Authorization = *what may you do*. In the App Router
the cardinal rule is **defense-in-depth: verify at the data layer, not just the
edge.** A pretty redirect in `proxy.ts` is UX, not security.

> Targets **Next.js 16+**. Auth library APIs (Auth.js v5, Clerk, WorkOS) and the
> `proxy.ts` migration move fast — verify against the installed versions; use the
> Context7 MCP (`/vercel/next.js`) for Next itself.

## Dogma

1. **`proxy.ts` (middleware) is NOT a security boundary.** It's an optimistic gate
   for redirects/UX. It can be bypassed and runs before some logic. **Never rely on
   it for authorization.**
2. **Verify in the Data Access Layer (DAL).** Every function that reads/writes
   sensitive data re-checks the session. This is the real boundary.
3. **Every Server Action and Route Handler is a public endpoint.** Each one
   independently authenticates *and* authorizes — no exceptions.
4. **AuthN then AuthZ, and check the specific resource.** Don't just confirm "logged
   in"; confirm this user may act on *this* record (prevents IDOR).
5. **Return DTOs, not raw rows.** Only expose fields the current user is allowed to
   see; never spread a DB record straight to the client.
6. **Secrets and session logic are `server-only`.** Mark those modules so they can't
   leak into a client bundle.
7. **Don't roll your own crypto/session for production.** Use a vetted library; the
   from-scratch pattern below is for understanding and small/custom cases.

## Architecture: the layers

```
Request
  │
  ├─ proxy.ts ............ optimistic redirect (no token → /login). UX only.
  │
  ├─ Server Component .... await verifySession() before rendering protected UI
  │
  ├─ Server Action ....... authN + authZ + validate, every action
  │
  ├─ Route Handler ....... authN + authZ, every handler
  │
  └─ Data Access Layer ... THE boundary: re-verify session + ownership on every
                           sensitive read/write, return DTOs
```

## The Data Access Layer (DAL) — the core pattern

Centralize session verification and data access in a `server-only` module. Memoize
the session check with React `cache()` so it runs once per request even if called
from many components.

```ts
// app/lib/dal.ts
import 'server-only'
import { cache } from 'react'
import { cookies } from 'next/headers'
import { redirect } from 'next/navigation'
import { decrypt } from '@/app/lib/session'

export const verifySession = cache(async () => {
  const cookie = (await cookies()).get('session')?.value // cookies() is async (16)
  const session = await decrypt(cookie)
  if (!session?.userId) redirect('/login')
  return { isAuth: true, userId: session.userId, role: session.role }
})

// Data getter: re-verifies, then returns a DTO (not the raw row)
export const getCurrentUser = cache(async () => {
  const { userId } = await verifySession()
  const user = await db.user.findUnique({ where: { id: userId } })
  if (!user) return null
  return { id: user.id, name: user.name, email: user.email } // DTO — no passwordHash, etc.
})
```

Use it everywhere sensitive:

```tsx
// Server Component — gate the page
import { verifySession, getCurrentUser } from '@/app/lib/dal'

export default async function DashboardPage() {
  await verifySession() // redirects if not authed
  const user = await getCurrentUser()
  return <Dashboard user={user} />
}
```

## Authorization in Server Actions (authN + authZ + ownership)

```ts
'use server'
import { auth } from '@/lib/auth' // or verifySession from the DAL
import { db } from '@/lib/db'

export async function deletePost(postId: string) {
  const session = await auth()
  if (!session?.user) throw new Error('Unauthorized')           // authN

  const post = await db.post.findUnique({ where: { id: postId } })
  if (post?.authorId !== session.user.id) throw new Error('Forbidden') // authZ on THIS resource (anti-IDOR)

  await db.post.delete({ where: { id: postId } })
}
```

The same applies to Route Handlers — verify inside `GET`/`POST`, never assume
`proxy.ts` ran.

## Role-based access control (RBAC)

```tsx
import { verifySession } from '@/app/lib/dal'
import { redirect } from 'next/navigation'

export default async function AdminArea() {
  const session = await verifySession()
  if (session.role !== 'admin') redirect('/') // or render a 403 / notFound()
  return <AdminDashboard />
}
```

Keep role/permission checks in the DAL or a small `lib/permissions.ts` (`can(user,
action, resource)`), and call it from components *and* actions — never trust the UI
to hide a forbidden control.

## Sessions

Two models — pick one:

- **Stateless (encrypted JWT cookie):** session payload encrypted into an
  `httpOnly`, `secure`, `sameSite` cookie. Fast, no DB read; revocation is harder.
- **Database sessions:** a session id cookie maps to a server-side record. Easy
  revocation; one DB lookup per request.

Cookie hygiene (whichever model): `httpOnly`, `secure` (prod), `sameSite: 'lax'`,
sensible `maxAge`/`expires`, and a clear path. Read/write via `await cookies()`
(async in 16) inside Server Components, Actions, or Route Handlers.

```ts
// app/lib/session.ts (sketch — prefer a library for real crypto)
import 'server-only'
import { cookies } from 'next/headers'

export async function createSession(userId: string) {
  const token = await encrypt({ userId, expires: Date.now() + 7 * 864e5 })
  ;(await cookies()).set('session', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    path: '/',
    maxAge: 7 * 24 * 60 * 60,
  })
}
```

## proxy.ts as an optimistic gate (UX only)

Fine for fast redirects, but **always** back it with DAL checks.

```ts
// proxy.ts
import { NextRequest, NextResponse } from 'next/server'

export default function proxy(req: NextRequest) {
  const hasSession = Boolean(req.cookies.get('session')?.value) // existence only — don't trust contents here
  const isProtected = req.nextUrl.pathname.startsWith('/dashboard')
  if (isProtected && !hasSession) return NextResponse.redirect(new URL('/login', req.url))
  return NextResponse.next()
}

export const config = { matcher: ['/dashboard/:path*'] }
```

## Choosing a library (recommended over DIY)

- **Auth.js v5 (NextAuth):** open-source, OAuth + credentials, App-Router native,
  `auth()` helper. Note 16: `middleware.ts` → `proxy.ts` (named/default `proxy`
  export). Great default for self-hosted.
- **Clerk:** managed, batteries-included UI, orgs/MFA, fastest to ship.
- **WorkOS / AuthKit:** enterprise SSO/SAML/SCIM, App-Router helpers.
- **Lucia-style / DIY:** maximum control; you own sessions and security. Only if you
  have the expertise.

Whatever you choose, **the DAL/defense-in-depth rules above still apply** — libraries
give you sessions, not authorization.

## Anti-patterns

- ❌ Treating `proxy.ts`/middleware as your authorization layer (bypassable).
- ❌ Checking auth only in the page/layout and assuming child data is safe.
- ❌ A Server Action/Route Handler with no internal auth check ("the button is hidden").
- ❌ Checking "is logged in" but not "owns this resource" → IDOR.
- ❌ Returning raw DB rows (leaks `passwordHash`, internal flags, other users' data).
- ❌ Session/crypto code without `server-only` (risk of client leak).
- ❌ Non-`httpOnly` session cookies, or secrets in `NEXT_PUBLIC_*`.
- ❌ Rolling your own auth crypto for production instead of a vetted library.

## Cross-references

- `proxy.ts`, route structure, Route Handlers → `nextjs-app-architecture`
- `server-only`, client boundary → `nextjs-server-client-components`
- DAL data access, caching, revalidation → `nextjs-data-fetching-and-caching`
- Validating + authorizing mutations → `nextjs-server-actions-and-forms`
- Testing auth flows (Playwright) → `nextjs-testing`
