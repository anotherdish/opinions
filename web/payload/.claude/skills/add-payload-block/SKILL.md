---
name: add-payload-block
description: Add a layout-builder Block to a Payload + Next.js website end-to-end — the block config, its React render component, registration on the blocks field, and the RenderBlocks switch. Use when asked to add a content block (CTA, hero, gallery, media, quote, accordion, etc.) to the page/post builder, or "a new block".
---

# Add a Layout-Builder Block

A block is two paired files plus two registration points. Always do all four — a config without a renderer (or vice versa) is a half-finished block.

```
src/blocks/<BlockName>/
├── config.ts        # Payload Block config (schema + admin)
└── Component.tsx    # React Server Component that renders it on the frontend
```

## Steps

1. **`src/blocks/<BlockName>/config.ts`** — define the `Block`. Set `interfaceName` to a stable PascalCase name so the generated type is importable.
2. **`src/blocks/<BlockName>/Component.tsx`** — render component, props typed from `@/payload-types`.
3. **Register on the field** — add the block to the `blocks: [...]` array of the layout field (usually in `collections/Pages` / `collections/Posts`).
4. **Register the renderer** — add a `case` to `src/blocks/RenderBlocks.tsx`, keyed by the block `slug`.
5. `pnpm generate:types` → import the generated interface into the component.
6. (Postgres) blocks add tables → create a migration (**payload-migrate-deploy** skill).

## config.ts

```ts
import type { Block } from 'payload'
import {
  FixedToolbarFeature, InlineToolbarFeature, HeadingFeature, lexicalEditor,
} from '@payloadcms/richtext-lexical'
import { linkGroup } from '@/fields/linkGroup'

export const CallToAction: Block = {
  slug: 'cta', // stored as blockType on each row
  interfaceName: 'CallToActionBlock', // -> import { CallToActionBlock } from '@/payload-types'
  labels: { singular: 'Call to Action', plural: 'Calls to Action' },
  fields: [
    {
      name: 'richText',
      type: 'richText',
      label: false,
      editor: lexicalEditor({
        features: ({ rootFeatures }) => [
          ...rootFeatures,
          HeadingFeature({ enabledHeadingSizes: ['h1', 'h2', 'h3', 'h4'] }),
          FixedToolbarFeature(),
          InlineToolbarFeature(),
        ],
      }),
    },
    linkGroup({ appearances: ['default', 'outline'], overrides: { maxRows: 2 } }),
  ],
}
```

## Component.tsx

```tsx
import React from 'react'
import type { CallToActionBlock as CTABlockProps } from '@/payload-types'
import RichText from '@/components/RichText'
import { CMSLink } from '@/components/Link'

export const CallToActionBlock: React.FC<CTABlockProps> = ({ links, richText }) => {
  return (
    <div className="container">
      {richText && <RichText data={richText} enableGutter={false} />}
      {(links || []).map(({ link }, i) => (
        <CMSLink key={i} size="lg" {...link} />
      ))}
    </div>
  )
}
```

## Register on the layout field

```ts
import { CallToAction } from '@/blocks/CallToAction/config'

{
  name: 'layout',
  type: 'blocks',
  required: true,
  admin: { initCollapsed: true },
  blocks: [CallToAction, /* Content, MediaBlock, ... */],
}
```

## RenderBlocks switch

```tsx
import React, { Fragment } from 'react'
import type { Page } from '@/payload-types'
import { CallToActionBlock } from '@/blocks/CallToAction/Component'

const blockComponents = {
  cta: CallToActionBlock,
  // contentBlock: ContentBlock, mediaBlock: MediaBlock, ...
}

export const RenderBlocks: React.FC<{ blocks: Page['layout'] }> = ({ blocks }) => {
  if (!Array.isArray(blocks) || blocks.length === 0) return null
  return (
    <Fragment>
      {blocks.map((block, i) => {
        const Block = blockComponents[block.blockType as keyof typeof blockComponents]
        if (!Block) return null
        return (
          <div className="my-16" key={i}>
            <Block {...(block as any)} disableInnerContainer />
          </div>
        )
      })}
    </Fragment>
  )
}
```

## Rules & gotchas

- The `blockComponents` map key MUST equal the block `slug` (`block.blockType`), not the interface name.
- Always set `interfaceName` — without it the generated type name is derived from the slug and can collide / hash-suffix when two blocks share a slug.
- Component is a **Server Component** by default. Only add `'use client'` if it needs interactivity (state, effects, event handlers).
- If a block must render inside the **Lexical** editor too, register it via `BlocksFeature({ blocks: [...] })` and supply a converter keyed by slug in your `<RichText>` converters (see **payload-frontend-fetch**).
- Reuse blocks across collections by importing the same `config.ts`. For large configs, define the block once at the top-level `blocks: []` in `payload.config.ts` and reference it by slug string — but note slug-referenced blocks can't be overridden per-collection and their access control runs without collection data.
- Block-internal rich text needs sufficient query `depth` to populate uploads/links on the frontend.
- Regenerate types and create a migration before finishing.
