---
name: nextjs-data-fetching-and-caching
description: Fetch and cache data in the Next.js App Router — async Server Component fetching, request memoization, parallel vs sequential fetching, the Next 16 caching model ('use cache', cacheLife, cacheTag, Cache Components/PPR), Suspense streaming, ISR/generateStaticParams, and revalidation (revalidateTag, updateTag, refresh). Use when reading data, deciding static vs dynamic, controlling caching, fixing stale/over-fetching, or invalidating caches after mutations. Targets Next.js 16.
---

# Data Fetching & Caching

How to read data and control its freshness. **Next.js 16 changed the model: code
is dynamic by default and caching is explicit and opt-in.** If you carry over
Next 14/15 caching assumptions you will be wrong — read the model section first.

> Always confirm caching specifics against the installed version via the Context7
> MCP (`/vercel/next.js`). Caching is the fastest-moving part of Next.js.

## Dogma (the non-negotiables)

1. **Read data in `async` Server Components.** No `useEffect` + client `fetch`, no
   API route in between. Keep the fetch in a `lib/` function (testable, reusable).
2. **Dynamic by default; cache explicitly.** Next 16 caches nothing on its own — opt
   in with `'use cache'` + `cacheLife` + `cacheTag`. Don't carry over 14/15 defaults.
3. **Tag everything you cache.** A cache entry without a `cacheTag` can't be
   precisely invalidated after a write.
4. **Request-time data goes inside `<Suspense>`.** `cookies()`/`headers()`/
   `searchParams`/uncached `fetch` outside a boundary forces the whole route dynamic.
5. **Parallelize independent reads.** `Promise.all` (or independent Suspense
   boundaries) — never a sequential `await` waterfall.
6. **Personalized data is never plain `'use cache'`.** Use `'use cache: private'` or
   no cache, or it leaks across users.
7. **Revalidate after every write** with the API matching the freshness you need
   (`updateTag` for read-your-writes, `revalidateTag(tag, profile)` for SWR).

## Fetch data in Server Components

The default and best place to read data is an `async` Server Component. No
`useEffect`, no client `fetch`, no API route in between.

```tsx
// app/blog/page.tsx
import { getPosts } from '@/lib/posts'

export default async function BlogPage() {
  const posts = await getPosts() // runs on the server only
  return (
    <ul>
      {posts.map((p) => (
        <li key={p.id}>{p.title}</li>
      ))}
    </ul>
  )
}
```

Keep the actual fetching in `lib/` functions (testable, reusable) and call them
from the component. You may use `fetch`, an ORM (Prisma/Drizzle), or any SDK.

### Request memoization (automatic, per-request)

Within a single render pass, identical `fetch()` calls (same URL + options) are
**deduplicated automatically** — call the same data-loader in the layout and the
page without double-fetching. For non-`fetch` data sources (ORM/SDK), wrap the
loader in React's `cache()` to get the same per-request dedupe:

```ts
// lib/posts.ts
import { cache } from 'react'
import 'server-only'

export const getPost = cache(async (id: string) => {
  return db.post.findUnique({ where: { id } })
})
```

### Parallel vs sequential — avoid waterfalls

Kick off independent requests together; only `await` sequentially when one depends
on another.

```tsx
// ✅ parallel — start both, then await
const postsPromise = getPosts()
const categoriesPromise = getCategories()
const [posts, categories] = await Promise.all([postsPromise, categoriesPromise])

// ❌ waterfall — categories waits for posts for no reason
const posts = await getPosts()
const categories = await getCategories()
```

For independent slices that can render at different times, prefer **streaming with
`<Suspense>`** over `Promise.all` so fast parts paint first (see below).

## The Next.js 16 caching model — read this

- **Everything is dynamic (request-time) by default.** A page/route with no caching
  directives renders fresh on every request, like classic server rendering.
- **Caching is explicit and opt-in** via the **`'use cache'`** directive plus
  `cacheLife()` (how long) and `cacheTag()` (how to invalidate).
- **Cache Components + PPR**: enabling `cacheComponents: true` lets one route mix a
  **prerendered static shell** (everything cacheable) with **dynamic content
  streamed at request time** (anything inside `<Suspense>` that reads request data).
  This is Partial Prerendering, now the supported programming model.

Enable it:

```ts
// next.config.ts
const nextConfig = { cacheComponents: true }
export default nextConfig
```

> The old implicit defaults (`fetch` cached unless `no-store`, `export const
> dynamic`, `experimental.ppr`, `unstable_noStore`) are superseded. `fetch`'s
> `cache`/`next.revalidate` options still exist, but `'use cache'` is the primary
> mechanism in 16.

## `'use cache'` — caching pages, components, and functions

Put `'use cache'` at the top of a function/component/route to cache its output.
Next's compiler derives the cache key from the closure's serializable inputs
(args, props). Pair it with `cacheLife` and `cacheTag`.

```tsx
import { cacheLife, cacheTag } from 'next/cache'

// Cache a whole page
export default async function Page() {
  'use cache'
  cacheLife('hours')          // freshness profile
  cacheTag('users')           // invalidation handle

  const users = await db.query('SELECT * FROM users')
  return <UserList users={users} />
}
```

```tsx
// Cache a single component (shared by everyone, revalidated hourly)
async function BlogPosts() {
  'use cache'
  cacheLife('hours')
  cacheTag('posts')
  const posts = await (await fetch('https://api.vercel.app/blog')).json()
  return <PostList posts={posts} />
}
```

### `cacheLife` profiles

Use a built-in profile or an inline object. Built-ins include `'seconds'`,
`'minutes'`, `'hours'`, `'days'`, `'weeks'`, `'max'`. `'max'` is recommended for
long-lived content because it enables stale-while-revalidate (serve cached
immediately, refresh in the background).

```ts
cacheLife('max')                              // long-lived, SWR
cacheLife('minutes')
cacheLife({ stale: 60, revalidate: 300, expire: 3600 }) // custom, seconds
```

Define reusable custom profiles in `next.config.ts` under `cacheLife`.

### `'use cache: private'` and `'use cache: remote'`

- **`'use cache: private'`** — per-user caching; may read `cookies()`/`headers()`
  inside. Use for personalized-but-cacheable content (recommendations, dashboards).
- **`'use cache: remote'`** — explicitly use the shared/remote cache layer.

```tsx
async function getRecommendations(productId: string) {
  'use cache: private'
  cacheTag(`recommendations-${productId}`)
  cacheLife({ stale: 60 })
  const sessionId = (await cookies()).get('session-id')?.value ?? 'guest'
  return getPersonalized(productId, sessionId)
}
```

## Mixing static, cached, and dynamic on one page (the PPR pattern)

This is the quintessential Next 16 page: static shell, cached sections in the
shell, and per-request dynamic content streamed via `<Suspense>`.

```tsx
import { Suspense } from 'react'
import { cookies } from 'next/headers'
import { cacheLife, cacheTag } from 'next/cache'

export default function BlogPage() {
  return (
    <>
      <header><h1>Our Blog</h1></header>           {/* static — prerendered */}

      <BlogPosts />                                 {/* cached — in the static shell */}

      <Suspense fallback={<p>Loading…</p>}>
        <UserPreferences />                          {/* dynamic — streamed at request time */}
      </Suspense>
    </>
  )
}

async function BlogPosts() {
  'use cache'
  cacheLife('hours')
  cacheTag('posts')
  const posts = await (await fetch('https://api.vercel.app/blog')).json()
  return <PostList posts={posts} />
}

async function UserPreferences() {
  const theme = (await cookies()).get('theme')?.value ?? 'light' // reads request data → dynamic
  return <aside>Theme: {theme}</aside>
}
```

Rule: anything that reads **request-time data** (`cookies()`, `headers()`,
`searchParams`, uncached `fetch`) must be **inside a `<Suspense>` boundary** (or in
a `loading.tsx`-wrapped segment), or the page can't prerender a shell.

## Streaming with Suspense

`<Suspense>` lets the shell render instantly while slow data streams in. Two ways:

1. **Segment-level**: add `loading.tsx` to a route — Next wraps the page in Suspense
   automatically (instant route transition).
2. **Component-level**: wrap any async component in `<Suspense fallback={...}>` for
   granular control and parallel streaming of independent slices.

```tsx
<Suspense fallback={<RevenueSkeleton />}>
  <RevenueChart />        {/* slow */}
</Suspense>
<Suspense fallback={<FeedSkeleton />}>
  <ActivityFeed />        {/* slow, streams independently */}
</Suspense>
```

## Static generation: ISR & generateStaticParams

For content that's the same for everyone and changes occasionally, prerender paths
and revalidate on a schedule.

```tsx
// app/blog/[id]/page.tsx
export const revalidate = 60 // re-generate at most once per 60s (ISR)

export async function generateStaticParams() {
  const posts = await (await fetch('https://api.vercel.app/blog')).json()
  return posts.map((p: { id: number }) => ({ id: String(p.id) }))
}

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const post = await (await fetch(`https://api.vercel.app/blog/${id}`)).json()
  return <Article post={post} />
}
```

Paths returned by `generateStaticParams` are prerendered; unlisted paths are
generated on-demand and then cached.

## Revalidation — invalidate after mutations

Call these from **Server Actions** or **Route Handlers** (see actions skill for the
full mutation flow). Choose by the semantics you need:

| API | Use when | Semantics |
|---|---|---|
| `updateTag(tag)` | In a Server Action; user must see their own write immediately | **Read-your-writes**: expire + read fresh in the same request |
| `revalidateTag(tag, profile)` | Background/eventually-consistent content | **SWR**: serve stale now, refresh in background. **16 requires a `cacheLife` profile as the 2nd arg** (e.g. `'max'`) |
| `revalidatePath(path)` | Invalidate everything under a route path | Path-based purge |
| `refresh()` | In a Server Action; refresh **uncached** dynamic data shown elsewhere | Doesn't touch the cache |
| `router.refresh()` | Client-side; re-fetch RSC for current route | Clears client cache only, **not** server cache |

```ts
'use server'
import { updateTag, revalidateTag } from 'next/cache'

export async function createPost(formData: FormData) {
  await db.post.create({ data: { title: String(formData.get('title')) } })
  updateTag('posts')                  // author sees the new post right away
  // revalidateTag('posts', 'max')    // alternative: SWR for non-critical freshness
}
```

> ⚠️ Single-arg `revalidateTag('posts')` is **deprecated in 16**. Add the profile
> (`revalidateTag('posts', 'max')`) for SWR, or use `updateTag('posts')` for
> read-your-writes.

## Choosing a strategy (cheat sheet)

- Same for all users, rarely changes → `'use cache'` + `cacheLife('max'/'days')` + `cacheTag`.
- Same for all, changes on a schedule → ISR `export const revalidate` or `cacheLife('hours')`.
- Per-user but cacheable → `'use cache: private'`.
- Truly per-request (auth, personalization, live) → no cache; wrap in `<Suspense>`.
- Independent slow sections → stream each with its own `<Suspense>`.

## Anti-patterns

- ❌ Assuming Next 16 caches by default — it doesn't; opt in with `'use cache'`.
- ❌ Fetching in a Client Component via `useEffect` when a Server Component can read it.
- ❌ Sequential `await`s that create request waterfalls (use `Promise.all`/Suspense).
- ❌ Reading `cookies()`/`headers()`/`searchParams` outside a Suspense boundary on a
  page you want to prerender (forces the whole route dynamic).
- ❌ `revalidateTag('x')` with one argument in 16 (deprecated).
- ❌ Using `router.refresh()` and expecting it to clear server-side cache (it won't).
- ❌ Caching personalized data with plain `'use cache'` (leaks across users — use
  `'use cache: private'`).

## Cross-references

- Where pages/route handlers live → `nextjs-app-architecture`
- Server vs Client (why fetch on the server) → `nextjs-server-client-components`
- Mutations that trigger revalidation → `nextjs-server-actions-and-forms`
- Server state vs client state, client-fetched data (TanStack Query/SWR) → `nextjs-state-management`
- Suspense, `loading.tsx`, PPR rendering → `nextjs-rendering-and-performance`
