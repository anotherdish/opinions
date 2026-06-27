---
name: nextjs-rendering-and-performance
description: Make a Next.js App Router app fast and polished — streaming with Suspense and loading.tsx, error/not-found boundaries, Partial Prerendering, the Metadata API and SEO (generateMetadata, sitemap, robots, OG images), next/image, next/font, next/script, bundle/JS reduction, React Compiler, and Core Web Vitals. Use when adding loading/error states, improving performance or LCP/CLS, optimizing images/fonts, setting page metadata, or doing SEO. Targets Next.js 16 + React 19.2.
---

# Rendering & Performance

How to ship a fast, polished Next.js app: stream UI progressively, handle errors
and missing pages, optimize assets, and own SEO/metadata. The App Router gives most
of this for free if you use the conventions.

## Streaming & loading states

Stream the page shell instantly and let slow parts fill in. Two mechanisms:

### `loading.tsx` (segment-level, automatic)

Add a `loading.tsx` to any route segment; Next wraps the segment's `page` in
`<Suspense>` automatically, so navigation shows the fallback instantly while data
loads.

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <DashboardSkeleton />
}
```

### `<Suspense>` (component-level, granular)

Wrap individual async components to stream them independently — the rest of the page
renders immediately. This is the building block of fast pages and of PPR.

```tsx
import { Suspense } from 'react'

export default function Page() {
  return (
    <>
      <Header />                                  {/* instant */}
      <Suspense fallback={<ChartSkeleton />}>
        <RevenueChart />                          {/* slow, streams in */}
      </Suspense>
      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed />                          {/* slow, streams in parallel */}
      </Suspense>
    </>
  )
}
```

**Dogma:** show skeletons that match final layout to avoid layout shift (CLS); give
every independently-slow region its own boundary; don't `await` a slow fetch above a
boundary if it would block the whole shell.

### Partial Prerendering (PPR)

With `cacheComponents: true`, Next prerenders a **static shell** (static + cached
content) and **streams dynamic content** (anything reading request data inside a
`<Suspense>`) at request time — one route, best of both. See
`nextjs-data-fetching-and-caching` for the full pattern; the rendering rule is:
**request-time data must live inside a Suspense boundary.**

## Error & not-found boundaries

```tsx
// app/dashboard/error.tsx — segment error boundary (must be a Client Component)
'use client'
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div role="alert">
      <p>Something went wrong.</p>
      <button onClick={reset}>Try again</button>
    </div>
  )
}
```

```tsx
// app/blog/[slug]/not-found.tsx — rendered by notFound()
export default function NotFound() {
  return <p>Post not found.</p>
}
```

```tsx
// trigger it from a page when data is missing
import { notFound } from 'next/navigation'
const post = await getPost(slug)
if (!post) notFound()
```

- `error.tsx` catches errors in its segment's tree and below; it must be a Client
  Component and exposes `reset()` to retry.
- `global-error.tsx` is the last resort at the root (renders its own `<html>`/`<body>`).
- Use `notFound()` for missing resources; `not-found.tsx` renders the UI. Expected
  validation errors should be **returned** from Server Actions, not thrown (see
  actions skill).

## Metadata & SEO

Use the **Metadata API** — never hand-write `<head>` tags. Static or dynamic.

```tsx
// Static (layout or page)
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: { default: 'Acme', template: '%s · Acme' },
  description: 'The quintessential app.',
  openGraph: { type: 'website', siteName: 'Acme' },
}
```

```tsx
// Dynamic, per-route — runs on the server, can fetch
import type { Metadata } from 'next'

export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>
}): Promise<Metadata> {
  const { slug } = await params
  const post = await getPost(slug)
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { title: post.title, images: [post.cover] },
    alternates: { canonical: `/blog/${slug}` },
  }
}
```

File-based metadata conventions (just add the file under `app/`):

- `sitemap.ts` → `sitemap.xml` (export a function returning route entries).
- `robots.ts` → `robots.txt`.
- `opengraph-image.tsx` / `twitter-image.tsx` → generated OG images via
  `next/og`'s `ImageResponse`.
- `favicon.ico`, `icon.png`, `apple-icon.png` → app icons.
- `manifest.ts` → PWA manifest.

For structured data, render a `<script type="application/ld+json">` with JSON-LD in
the server component.

> Note (16): metadata image routes receive **async `params`**, and `id` from
> `generateImageMetadata` is a `Promise<string>`.

## Images — `next/image`

Always use `next/image` for raster images: automatic resizing, modern formats,
lazy-loading, and CLS prevention.

```tsx
import Image from 'next/image'

// Static import → width/height/blur inferred automatically
import hero from '@/public/hero.jpg'
<Image src={hero} alt="Hero" priority placeholder="blur" />

// Remote → set sizing + allowlist the host in next.config
<Image src="https://cdn.example.com/p.jpg" alt="Product" width={800} height={600} />
```

- Add `priority` to the LCP image (usually above the fold); never lazy-load it.
- Provide `width`/`height` (or `fill` + a sized parent) to prevent layout shift.
- Use `sizes` for responsive images so the browser picks the right source.
- Allowlist remote hosts via `images.remotePatterns` (not the deprecated
  `images.domains`).
- 16 default changes: `images.qualities` defaults to `[75]`; local-IP optimization
  is blocked by default; local `src` with query strings needs `images.localPatterns`.

## Fonts — `next/font`

Self-host fonts at build time — no layout shift, no external request, no privacy
leak.

```tsx
// app/layout.tsx
import { Inter } from 'next/font/local' // or 'next/font/google'
import { Inter as Google_Inter } from 'next/font/google'

const inter = Google_Inter({ subsets: ['latin'], display: 'swap', variable: '--font-inter' })

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.variable}>
      <body>{children}</body>
    </html>
  )
}
```

## Scripts — `next/script`

Load third-party JS with a strategy instead of raw `<script>`:

```tsx
import Script from 'next/script'
<Script src="https://example.com/analytics.js" strategy="afterInteractive" />
```

`beforeInteractive` (critical, rare), `afterInteractive` (default, analytics),
`lazyOnload` (low priority, chat widgets).

## Reducing client JavaScript

The biggest lever in the App Router. In priority order:

1. **Keep components on the server.** Default to Server Components; shrink the
   `'use client'` surface to leaf islands (see server/client skill). Server
   Components ship zero JS.
2. **Push the client boundary leaf-ward** so heavy deps stay server-side.
3. **`next/dynamic`** for client components that are heavy and below-the-fold or
   conditionally shown: `dynamic(() => import('./Heavy'), { ssr: false })`.
4. **React Compiler** (`reactCompiler: true`) auto-memoizes — then stop hand-writing
   `useMemo`/`useCallback`/`memo` except at measured hotspots.
5. Import only what you use (named imports from large libs; avoid barrel files that
   defeat tree-shaking).

## Navigation & prefetching

- Use `<Link>` (not `<a>`) for internal navigation — it prefetches in-viewport
  routes and does client-side transitions.
- Next 16's router does **layout deduplication** (shared layouts fetched once) and
  **incremental prefetching** (only missing parts, cancels off-screen) — no code
  changes needed; just use `<Link>`.
- Use `useRouter().prefetch()` for programmatic prefetch; `router.refresh()` to
  re-fetch the current route's RSC.

## Core Web Vitals checklist

- **LCP**: `priority` on the hero image; stream non-critical content; cache/prerender
  the shell.
- **CLS**: always size images/embeds; `next/font` for fonts; skeletons that match
  final layout.
- **INP**: minimize client JS; keep handlers light; lean on Server Components.
- Measure with `next build` output, Lighthouse, and the `useReportWebVitals` hook.

## Anti-patterns

- ❌ Hand-writing `<head>`/`<title>` tags instead of the Metadata API.
- ❌ Plain `<img>` / raw `@font-face` / raw `<script>` instead of `next/image` /
  `next/font` / `next/script`.
- ❌ Lazy-loading or omitting `priority` on the LCP image.
- ❌ Images/embeds without dimensions → layout shift.
- ❌ One giant Suspense boundary around the whole page (no streaming benefit) — or
  none at all (blocks on the slowest fetch).
- ❌ `error.tsx` without `'use client'` (won't work).
- ❌ Premature `useMemo`/`useCallback` everywhere instead of enabling React Compiler.

## Cross-references

- Reserved files (`loading`, `error`, `not-found`), config → `nextjs-app-architecture`
- Shrinking the client bundle via the boundary → `nextjs-server-client-components`
- PPR, `'use cache'`, what makes a route dynamic → `nextjs-data-fetching-and-caching`
- Returning vs throwing errors from mutations → `nextjs-server-actions-and-forms`
