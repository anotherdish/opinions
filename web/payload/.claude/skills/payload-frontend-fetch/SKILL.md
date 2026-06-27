---
name: payload-frontend-fetch
description: Fetch Payload data inside Next.js Server Components the right way — Local API via getPayload, cache()-wrapped queries, draftMode/preview handling, generateStaticParams + generateMetadata, correct depth/select, and rendering Lexical rich text. Use when building or editing a frontend page/route that reads CMS data, a sitemap, or a preview route.
---

# Frontend Data Fetching (Next.js App Router + Payload Local API)

Fetch with the **Local API** directly in Server Components — no HTTP round-trip, no REST/GraphQL layer for your own pages. This is Payload's core performance advantage.

## The canonical detail page

```tsx
// src/app/(frontend)/posts/[slug]/page.tsx
import type { Metadata } from 'next'
import { draftMode } from 'next/headers'
import { notFound } from 'next/navigation'
import { getPayload } from 'payload'
import configPromise from '@payload-config'
import { cache } from 'react'

import { generateMeta } from '@/utilities/generateMeta'
import { RenderBlocks } from '@/blocks/RenderBlocks'
import { LivePreviewListener } from '@/components/LivePreviewListener'

type Args = { params: Promise<{ slug?: string }> }

export default async function Post({ params: paramsPromise }: Args) {
  const { isEnabled: draft } = await draftMode()
  const { slug = '' } = await paramsPromise
  const post = await queryPostBySlug({ slug: decodeURIComponent(slug) })

  if (!post) return notFound()

  return (
    <article className="pt-16 pb-24">
      {draft && <LivePreviewListener />}
      <h1>{post.title}</h1>
      <RenderBlocks blocks={post.layout} />
    </article>
  )
}

export async function generateMetadata({ params: paramsPromise }: Args): Promise<Metadata> {
  const { slug = '' } = await paramsPromise
  const post = await queryPostBySlug({ slug: decodeURIComponent(slug) })
  return generateMeta({ doc: post })
}

// cache() => the page + generateMetadata share ONE db hit per request
const queryPostBySlug = cache(async ({ slug }: { slug: string }) => {
  const { isEnabled: draft } = await draftMode()
  const payload = await getPayload({ config: configPromise })

  const result = await payload.find({
    collection: 'posts',
    limit: 1,
    pagination: false,
    draft,                 // return latest version (draft) when in draft mode
    overrideAccess: draft, // only bypass access control inside authenticated preview
    where: { slug: { equals: slug } },
  })

  return result.docs?.[0] || null
})
```

## Static params (build-time paths)

```tsx
export async function generateStaticParams() {
  const payload = await getPayload({ config: configPromise })
  const posts = await payload.find({
    collection: 'posts',
    draft: false,
    limit: 1000,
    pagination: false,
    overrideAccess: false,
    select: { slug: true }, // fetch ONLY slugs — do not pull whole docs
  })
  return posts.docs.map(({ slug }) => ({ slug }))
}
```

## Draft preview route

```ts
// src/app/(frontend)/next/preview/route.ts
import type { PayloadRequest } from 'payload'
import { getPayload } from 'payload'
import { draftMode } from 'next/headers'
import { redirect } from 'next/navigation'
import configPromise from '@payload-config'

export async function GET(req: Request & { cookies: { get: (n: string) => { value: string } } }) {
  const payload = await getPayload({ config: configPromise })
  const { searchParams } = new URL(req.url)
  const path = searchParams.get('path')
  const previewSecret = searchParams.get('previewSecret')

  if (previewSecret !== process.env.PREVIEW_SECRET) return new Response('Forbidden', { status: 403 })
  if (!path?.startsWith('/')) return new Response('Bad path', { status: 400 })

  const { user } = await payload.auth({ req: req as unknown as PayloadRequest, headers: req.headers })
  const draft = await draftMode()
  if (!user) { draft.disable(); return new Response('Forbidden', { status: 403 }) }

  draft.enable()
  redirect(path)
}
```

The matching `admin.preview` / `livePreview.url` on the collection points here via `generatePreviewPath` (builds `/next/preview?path=...&previewSecret=...`).

## Live preview listener (RSC)

```tsx
'use client'
import { RefreshRouteOnSave as PayloadLivePreview } from '@payloadcms/live-preview-react'
import { useRouter } from 'next/navigation'

export const LivePreviewListener = () => {
  const router = useRouter()
  return <PayloadLivePreview refresh={() => router.refresh()} serverURL={process.env.NEXT_PUBLIC_SERVER_URL!} />
}
```

## Rendering Lexical rich text

```tsx
import { RichText as RichTextConverter, type JSXConvertersFunction } from '@payloadcms/richtext-lexical/react'
import { LinkJSXConverter } from '@payloadcms/richtext-lexical/react'
import type { DefaultNodeTypes, SerializedLinkNode } from '@payloadcms/richtext-lexical'

const internalDocToHref = ({ linkNode }: { linkNode: SerializedLinkNode }) => {
  const { relationTo, value } = linkNode.fields.doc!
  const slug = typeof value === 'object' ? value.slug : value
  return relationTo === 'posts' ? `/posts/${slug}` : `/${slug}`
}

const converters: JSXConvertersFunction<DefaultNodeTypes> = ({ defaultConverters }) => ({
  ...defaultConverters,
  ...LinkJSXConverter({ internalDocToHref }),
  blocks: { /* key by block slug: myBlock: ({ node }) => <MyBlock {...node.fields} /> */ },
})

export default function RichText({ data }) {
  return <RichTextConverter converters={converters} data={data} />
}
```

## Rules & gotchas

- **`overrideAccess`**: pass `false` (and `user` when relevant) for normal page queries so unpublished/private content can't leak. Only set it to `draft` (true) inside the authenticated preview path, where `read` access would otherwise hide drafts.
- **Always `cache()`** any query reused by both the component and `generateMetadata` — otherwise you double the DB hits.
- **Depth**: rich text with uploads/links and relationship fields need enough `depth` to populate, or the JSX converter throws "fully populated data" errors. Prefer `defaultPopulate` on the referenced collection over cranking global depth.
- **`select`** in `generateStaticParams` / sitemaps to pull only the fields you need (slugs) — never fetch whole documents to build a URL list.
- Provide `internalDocToHref` to `LinkJSXConverter` or internal links error in the console.
- Custom Lexical blocks need a converter keyed by the block slug; they are not rendered by default.
- `draftMode()`, `params`, and `searchParams` are async in Next 16 — always `await` them.
- Never expose `PAYLOAD_SECRET` or `PREVIEW_SECRET` to the client. Public values use the `NEXT_PUBLIC_` prefix.
