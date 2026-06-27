---
name: nextjs-app-architecture
description: Scaffold and organize a Next.js App Router application — project structure, file-system routing conventions (layouts, pages, loading, error, route groups, dynamic/parallel/intercepting routes), Route Handlers, proxy.ts, and config. Use when creating a new Next.js app, adding routes, organizing folders, building APIs, or deciding where files go. Targets Next.js 16 (App Router).
---

# Next.js App Architecture

The skeleton of a quintessential Next.js app: how the file system becomes the
router, where code lives, and the conventions that keep a project scalable.

> Targets **Next.js 16+** (App Router, Turbopack default, React 19.2). Notes mark
> where 15 differs. Verify specifics against the installed version with the
> Context7 MCP (`/vercel/next.js`) — the framework moves fast.

## Dogma (the non-negotiables)

1. **App Router only.** Never scaffold `pages/`. The App Router is the present and
   future; `pages/` is legacy. Do not mix them in new work.
2. **TypeScript always.** `.tsx`/`.ts`, `strict: true`. No new `.jsx`.
3. **Colocation first.** A route owns its components, tests, and styles. Files
   only become routes via the reserved names below — everything else is safe to
   put next to the route that uses it.
4. **The `app/` directory is for routing, not for everything.** Shared, reusable
   logic lives outside route segments (`lib/`, `components/`, `hooks/`).
5. **Thin pages, fat lib.** Pages and layouts orchestrate; data access and business
   logic live in `lib/` so they are testable and reusable.
6. **`params` and `searchParams` are async** (Next 15+/16). Always `await` them.

## Scaffolding a new app

```bash
npx create-next-app@latest my-app   # App Router, TS, Tailwind, ESLint, Turbopack
```

The default template is the right baseline: App Router, TypeScript-first, Tailwind,
ESLint flat config, Turbopack. Do not fight it.

## Recommended project structure

Next.js is unopinionated about organization. The opinionated, scalable convention:

```
my-app/
├── app/                      # ROUTING ONLY — segments, layouts, pages
│   ├── layout.tsx            # root layout (required; has <html>/<body>)
│   ├── page.tsx              # /
│   ├── globals.css
│   ├── (marketing)/          # route group — organizes, no URL segment
│   │   ├── layout.tsx
│   │   ├── page.tsx          # still "/"... (groups don't add to path)
│   │   └── about/page.tsx    # /about
│   ├── (app)/                # authenticated app shell
│   │   ├── layout.tsx
│   │   └── dashboard/
│   │       ├── page.tsx      # /dashboard
│   │       ├── loading.tsx
│   │       ├── error.tsx
│   │       └── _components/  # private folder — colocated, never routable
│   ├── blog/
│   │   └── [slug]/page.tsx   # /blog/:slug  (dynamic)
│   └── api/
│       └── webhooks/route.ts # Route Handler (only when you need raw HTTP)
├── components/               # shared, cross-route UI
│   └── ui/                   # design-system primitives (button, input, ...)
├── lib/                      # data access, business logic, clients, utils
│   ├── db.ts
│   ├── auth.ts
│   └── posts.ts              # e.g. getPosts(), createPost()
├── hooks/                    # shared client hooks (use-*)
├── proxy.ts                  # request interception (formerly middleware.ts)
├── next.config.ts
└── tsconfig.json             # set "@/*" path alias
```

Two organization strategies, both valid — pick per project and stay consistent:

- **Colocation** (default): everything a route needs lives in/next to its segment,
  hidden from routing via `_private` folders. Best for route-specific code.
- **Feature folders**: group everything for a feature (`features/billing/...`) and
  import into thin routes. Best for logic shared across many routes.

A hybrid — colocate route-specific UI, extract shared logic to `lib/` and
`features/` — is the sweet spot for growing apps.

## File-system routing — the reserved names

Inside `app/`, only these filenames have special meaning. Everything else is inert.

| File | Role |
|---|---|
| `layout.tsx` | Shared UI wrapping a segment + children; **preserves state across navigation**, does not re-render on child nav. Root `layout.tsx` is required and renders `<html>`/`<body>`. |
| `page.tsx` | The unique UI of a route; makes the segment publicly routable. |
| `loading.tsx` | Instant loading UI; auto-wraps the segment in `<Suspense>`. |
| `error.tsx` | Error boundary for the segment (must be `'use client'`). |
| `global-error.tsx` | Root-level error boundary; must render its own `<html>`/`<body>`. |
| `not-found.tsx` | UI for `notFound()` and unmatched URLs. |
| `template.tsx` | Like layout but **re-mounts** on navigation (fresh state per nav). |
| `route.ts` | Route Handler (HTTP endpoint). Cannot coexist with `page.tsx` in the same segment. |
| `default.tsx` | Fallback for unmatched parallel-route slots (**required in 16** for every slot). |

### Folder conventions

| Pattern | Meaning |
|---|---|
| `folder/` | A route segment → URL path. |
| `[id]/` | Dynamic segment → `params.id`. |
| `[...slug]/` | Catch-all → `params.slug: string[]`. |
| `[[...slug]]/` | Optional catch-all (also matches the parent). |
| `(group)/` | Route group — organizes/shares layouts, **no URL segment**. |
| `_folder/` | Private folder — opted out of routing entirely. |
| `@slot/` | Named slot for parallel routes. |
| `(.)folder` | Intercepting route (same level / `(..)` parent / `(...)` root). |

## Pages, layouts, and async params

```tsx
// app/blog/[slug]/page.tsx
export default async function Page({
  params,
  searchParams,
}: {
  params: Promise<{ slug: string }>
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>
}) {
  const { slug } = await params          // ⚠️ Next 15+/16: params is a Promise
  const { sort } = await searchParams
  const post = await getPost(slug)       // data access lives in lib/
  return <Article post={post} />
}
```

```tsx
// app/layout.tsx — root layout (required)
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

**Layouts don't receive `params`/`searchParams` the same way.** Layouts get
`params` (async) but **not** `searchParams` — reading search params requires a
page or a client component, because layouts don't re-render on query changes.

## Route Handlers (`route.ts`) — use sparingly

In the App Router you rarely need API routes. Prefer **Server Components** for
reads and **Server Actions** for mutations. Reach for a Route Handler only when an
external/non-React consumer needs raw HTTP: webhooks, OAuth callbacks, public REST,
cron endpoints, streaming/SSE, file downloads.

```ts
// app/api/posts/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const query = request.nextUrl.searchParams.get('q')
  const posts = await getPosts(query)
  return NextResponse.json(posts)
}

export async function POST(request: NextRequest) {
  const body = await request.json()
  // validate, authorize, then act
  return NextResponse.json({ ok: true }, { status: 201 })
}
```

- One handler file exports functions named for HTTP verbs (`GET`, `POST`, ...).
- A segment is **either** a `page.tsx` **or** a `route.ts`, never both.
- Route Handlers are **dynamic by default** in 16; add `'use cache'`/`cacheLife`
  inside if you want caching (see the data-fetching skill).

## proxy.ts (formerly middleware.ts)

Next 16 renames `middleware.ts` → **`proxy.ts`** and runs it on the **Node.js
runtime**. Rename the file and the exported function (`proxy`); logic is unchanged.
`middleware.ts` still works for Edge cases but is deprecated.

```ts
// proxy.ts
import { NextRequest, NextResponse } from 'next/server'

export default function proxy(request: NextRequest) {
  const token = request.cookies.get('session')?.value
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  return NextResponse.next()
}

export const config = { matcher: ['/dashboard/:path*'] }
```

Keep proxy logic thin (it runs on every matched request): redirects, header
rewrites, coarse auth gates. **Do real authorization in the data layer / Server
Actions** — proxy is a convenience gate, not a security boundary.

## TypeScript & config conventions

- `tsconfig.json`: `"strict": true`, path alias `"@/*": ["./*"]` (or `./src/*`).
- `next.config.ts` (TypeScript config is supported):

```ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  cacheComponents: true, // opt into Cache Components / PPR (see caching skill)
  // reactCompiler: true, // automatic memoization (needs babel-plugin-react-compiler)
  images: { remotePatterns: [{ protocol: 'https', hostname: 'images.example.com' }] },
}

export default nextConfig
```

- **Removed in 16** — don't generate these: `next lint` (use ESLint/Biome
  directly), `serverRuntimeConfig`/`publicRuntimeConfig` (use env vars),
  `experimental.ppr` / `export const experimental_ppr` (now `cacheComponents`),
  `images.domains` (use `remotePatterns`), AMP.
- Requirements: **Node 20.9+**, **TypeScript 5.1+**.

## Anti-patterns

- ❌ Creating `pages/` in a new App Router app, or mixing routers needlessly.
- ❌ Dumping shared components/utilities inside `app/` route segments.
- ❌ Building `app/api/*` Route Handlers your own React app calls — use Server
  Components (read) and Server Actions (write) instead.
- ❌ Forgetting `await params`/`await searchParams` (and `await cookies()`/`headers()`).
- ❌ Treating `proxy.ts` as your authorization layer.
- ❌ Missing `default.tsx` for a parallel-route slot (build fails in 16).

## Cross-references

- Server vs Client component decisions → `nextjs-server-client-components`
- Reading data in pages, `use cache`, revalidation → `nextjs-data-fetching-and-caching`
- Mutations from forms/buttons → `nextjs-server-actions-and-forms`
- `loading.tsx`, streaming, metadata, images → `nextjs-rendering-and-performance`
