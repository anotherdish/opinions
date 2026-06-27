---
name: scaffold-payload-collection
description: Scaffold a new Payload Collection or Global for a Next.js website — wires access control, slug, SEO, drafts/versions, live preview, and revalidation following the official website-template conventions. Use when asked to add a content type (pages, posts, authors, categories, media), a singleton (header, footer, settings), or "a new collection/global".
---

# Scaffold a Payload Collection or Global

Produce a config that matches the project's established conventions. **Read existing collections in `src/collections/` first** and mirror their patterns (imports, access funcs, hook names) rather than inventing new ones.

## Decide first

1. **Collection vs Global?** Many records → Collection. Exactly one (header, footer, site settings) → Global.
2. **Public-facing content?** (rendered on the site, has its own URL) → needs `slug`, drafts, preview, SEO, revalidation.
   **Internal/admin-only?** (e.g. users) → minimal: access + fields, no drafts/preview.
3. **Authenticated?** A users-style collection adds `auth: true`.
4. **Uploads?** Media-style collection adds `upload: {...}` and is wired to the storage adapter in `payload.config.ts`.

## Steps

1. Create `src/collections/<Name>/index.ts` (or `src/globals/<Name>/index.ts`).
2. Write the config (templates below). Use the shared access functions from `src/access/`.
3. Register it in `src/payload.config.ts` (`collections: [...]` or `globals: [...]`).
4. For public content: add `revalidate` hooks in `src/collections/<Name>/hooks/` and `populatePublishedAt` on `beforeChange`.
5. Run `pnpm generate:types`.
6. (Postgres) create a migration — see the **payload-migrate-deploy** skill.
7. `pnpm lint`.

## Template: public content collection (pages/posts)

```ts
import type { CollectionConfig } from 'payload'

import { authenticated } from '@/access/authenticated'
import { authenticatedOrPublished } from '@/access/authenticatedOrPublished'
import { slugField } from '@/fields/slug'
import { generatePreviewPath } from '@/utilities/generatePreviewPath'
import { populatePublishedAt } from '@/hooks/populatePublishedAt'
import { revalidatePost, revalidateDelete } from './hooks/revalidatePost'
import {
  MetaDescriptionField, MetaImageField, MetaTitleField,
  OverviewField, PreviewField,
} from '@payloadcms/plugin-seo/fields'

export const Posts: CollectionConfig = {
  slug: 'posts',
  access: {
    create: authenticated,
    delete: authenticated,
    read: authenticatedOrPublished, // hides drafts from the public
    update: authenticated,
  },
  defaultPopulate: { title: true, slug: true }, // what gets pulled when this is referenced
  admin: {
    useAsTitle: 'title',
    defaultColumns: ['title', 'slug', 'updatedAt'],
    livePreview: {
      url: ({ data, req }) =>
        generatePreviewPath({ slug: data?.slug, collection: 'posts', req }),
    },
    preview: (data, { req }) =>
      generatePreviewPath({ slug: data?.slug as string, collection: 'posts', req }),
  },
  fields: [
    { name: 'title', type: 'text', required: true },
    {
      type: 'tabs',
      tabs: [
        { label: 'Content', fields: [/* richText / blocks / relationships */] },
        {
          name: 'meta',
          label: 'SEO',
          fields: [
            OverviewField({ titlePath: 'meta.title', descriptionPath: 'meta.description', imagePath: 'meta.image' }),
            MetaTitleField({ hasGenerateFn: true }),
            MetaImageField({ relationTo: 'media' }),
            MetaDescriptionField({}),
            PreviewField({ hasGenerateFn: true, titlePath: 'meta.title', descriptionPath: 'meta.description' }),
          ],
        },
      ],
    },
    { name: 'publishedAt', type: 'date', admin: { position: 'sidebar' } },
    ...slugField(),
  ],
  hooks: {
    afterChange: [revalidatePost],
    beforeChange: [populatePublishedAt],
    afterDelete: [revalidateDelete],
  },
  versions: {
    maxPerDoc: 50,
    drafts: { autosave: { interval: 100 }, schedulePublish: true },
  },
}
```

## Template: global (header/footer/settings)

```ts
import type { GlobalConfig } from 'payload'
import { authenticated } from '@/access/authenticated'
import { revalidateHeader } from './hooks/revalidateHeader'

export const Header: GlobalConfig = {
  slug: 'header',
  access: { read: () => true, update: authenticated },
  fields: [
    { name: 'navItems', type: 'array', maxRows: 6, fields: [/* link group */] },
  ],
  hooks: { afterChange: [revalidateHeader] },
}
```

## Revalidation hook pattern

```ts
import type { CollectionAfterChangeHook, CollectionAfterDeleteHook } from 'payload'
import { revalidatePath, revalidateTag } from 'next/cache'

export const revalidatePost: CollectionAfterChangeHook = ({ doc, previousDoc, req: { payload, context } }) => {
  if (context.disableRevalidate) return doc
  if (doc._status === 'published') {
    const path = `/posts/${doc.slug}`
    payload.logger.info({ msg: `Revalidating post at path: ${path}` })
    revalidatePath(path)
    revalidateTag('posts-sitemap')
  }
  // revalidate the old path if the post was unpublished
  if (previousDoc?._status === 'published' && doc._status !== 'published') {
    revalidatePath(`/posts/${previousDoc.slug}`)
  }
  return doc
}
```

## Rules & gotchas

- Reuse `src/access/*` — do not write inline access closures unless the logic is genuinely collection-specific.
- `read: authenticatedOrPublished` is what actually hides drafts. Enabling `drafts` alone does NOT secure them.
- Put each collection in its own folder; keep hooks in a `hooks/` subfolder.
- If the collection is referenced by relationship fields elsewhere, set `defaultPopulate` to the minimal fields the frontend needs (keeps payloads small).
- `useAsTitle` must point at a real top-level text field (not a relationship/join — those show only the ID).
- After ANY field change, regenerate types and (Postgres) create a migration before considering the task done.
- A `slugField()` factory living in `src/fields/slug` (returns `[slug, slugLock]`) is the convention; if the project instead imports `slugField` from `'payload'`, follow whichever the existing collections use.
