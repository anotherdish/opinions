# Payload + Next.js Website — Claude Kit

A portable, opinionated set of Claude Code **rules** and **skills** for building content-driven websites on **Payload v4** running natively in **Next.js (App Router)**. Distilled from the Payload v4 docs and the official `templates/website` starter.

## What's inside

```
CLAUDE.md                                  # always-on project rules (stack, structure, patterns, security)
.claude/skills/
├── scaffold-payload-collection/SKILL.md   # add a Collection or Global
├── add-payload-block/SKILL.md             # add a layout-builder block end-to-end
├── payload-frontend-fetch/SKILL.md        # fetch CMS data in Server Components (draft/preview/cache/depth)
└── payload-migrate-deploy/SKILL.md        # Postgres migration workflow + deploy checklist
```

- **`CLAUDE.md`** is loaded into every session automatically — concise, always-on conventions.
- **Skills** load on demand when a task matches their `description` — the heavier, step-by-step workflows.

## The opinionated stack

Postgres (`@payloadcms/db-postgres`, Drizzle) · Lexical rich text · Vercel Blob storage · Tailwind v4 + shadcn-style primitives · plugins: SEO / redirects / search / nested-docs / form-builder · `@payloadcms/live-preview-react` · drafts + draft-preview + autosave live preview. Next.js 16 / React 19 / Node ≥24.15 / pnpm / TypeScript 6.

## Install into a website project

Copy the contents into the root of your Payload+Next.js repo:

```sh
cp CLAUDE.md /path/to/your-site/CLAUDE.md
cp -r .claude /path/to/your-site/.claude   # merge if .claude already exists
```

If you already have a `CLAUDE.md`, append the relevant sections rather than overwriting.

## Verify the skills are picked up

In the project, run `/help` or check that the skill names appear in Claude Code's available skills. They activate automatically when a request matches (e.g. "add a testimonials collection" → `scaffold-payload-collection`).

## Notes

- Built against Payload **v4** conventions. If you're on v3, the `(payload)`/`(frontend)` split and most patterns still apply, but check `strictDraftTypes`, storage-under-`storage`-key, and block `interfaceName` behavior.
- Keep all `@payloadcms/*` packages on the same version.
- Regenerate types (`pnpm generate:types`) and create a Postgres migration after any schema change — both skills and `CLAUDE.md` reinforce this.
