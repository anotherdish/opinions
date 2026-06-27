# Payload + Next.js Website — Claude Code Plugin

A portable, opinionated Claude Code **plugin** of **skills** for building content-driven websites on **Payload v4** running natively in **Next.js (App Router)**. Distilled from the Payload v4 docs and the official `templates/website` starter.

## What's inside

```
.claude-plugin/plugin.json                 # plugin manifest (payload-website)
skills/
├── payload-conventions/SKILL.md           # the opinionated conventions (stack, structure, patterns, security)
├── scaffold-payload-collection/SKILL.md   # add a Collection or Global
├── add-payload-block/SKILL.md             # add a layout-builder block end-to-end
├── payload-frontend-fetch/SKILL.md        # fetch CMS data in Server Components (draft/preview/cache/depth)
└── payload-migrate-deploy/SKILL.md        # Postgres migration workflow + deploy checklist
```

Everything ships as a **skill**, so it's fully plugin-native — nothing to copy into your project.

- **`payload-conventions`** is the broad skill carrying the conventions (the former `CLAUDE.md`). Its `description` is written to match any work in a Payload + Next.js codebase, so Claude loads it whenever you touch config, collections, blocks, access control, or frontend fetching.
- The other four skills are the heavier, step-by-step workflows and load on demand when a task matches their `description`.

## The opinionated stack

Postgres (`@payloadcms/db-postgres`, Drizzle) · Lexical rich text · Vercel Blob storage · Tailwind v4 + shadcn-style primitives · plugins: SEO / redirects / search / nested-docs / form-builder · `@payloadcms/live-preview-react` · drafts + draft-preview + autosave live preview. Next.js 16 / React 19 / Node ≥24.15 / pnpm / TypeScript 6.

## Install as a plugin

Add the marketplace this plugin lives in, then install it:

```sh
/plugin marketplace add <owner/repo>      # the repo hosting this plugin
/plugin install payload-website@<marketplace-name>
```

Or browse interactively with `/plugin`. Installed skills become available across every project where the plugin is enabled.

## Verify the skills are picked up

Run `/help` or check that the skill names appear in Claude Code's available skills. They activate automatically when a request matches (e.g. "add a testimonials collection" → `scaffold-payload-collection`).

## Notes

- Built against Payload **v4** conventions. If you're on v3, the `(payload)`/`(frontend)` split and most patterns still apply, but check `strictDraftTypes`, storage-under-`storage`-key, and block `interfaceName` behavior.
- Keep all `@payloadcms/*` packages on the same version.
- Regenerate types (`pnpm generate:types`) and create a Postgres migration after any schema change — the skills reinforce this.
