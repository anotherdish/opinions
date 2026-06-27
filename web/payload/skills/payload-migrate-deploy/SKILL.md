---
name: payload-migrate-deploy
description: Run the Payload Postgres migration workflow and the production deploy checklist. Use when changing the database schema (new collection/field/block), creating or running migrations, setting up the CI build script, or preparing a Payload + Next.js site for production deployment.
---

# Migrations & Deployment (Postgres)

Payload uses Drizzle. In **development** the schema is auto-synced via `push`; in **production** you ship explicit, committed SQL migrations.

## Mental model

- **Local dev = sandbox.** `push` mode syncs your DB to the config automatically as you edit. Do not run migrations against the local push DB — mixing the two triggers warnings and drift.
- **Every schema change needs a committed migration** before it reaches a non-dev environment, or reads/writes error because the DB shape lags the config.
- Migrations run **in CI before the build**, in creation order, inside transactions.

## Required package.json scripts

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "ci": "payload migrate && pnpm build",
    "payload": "cross-env PAYLOAD_CONFIG_PATH=src/payload.config.ts payload",
    "generate:types": "cross-env PAYLOAD_CONFIG_PATH=src/payload.config.ts payload generate:types"
  }
}
```

Set the deploy platform's **build command** to `pnpm ci` (or `pnpm run ci`).

## Workflow: making a schema change

1. Edit the config (add collection / field / block) in dev with `push` on. Verify it works in the admin.
2. `pnpm generate:types`.
3. `pnpm payload migrate:create <descriptive-name>`.
4. **Read the generated SQL** in `src/migrations/` — confirm it does what you expect (no unintended drops/renames). Postgres column renames may appear as drop+add; data-preserving renames must be hand-edited.
5. Commit the migration file(s) together with the config change.

## Commands

| Command | Purpose |
| --- | --- |
| `pnpm payload migrate:create [name]` | Generate a migration from config diff |
| `pnpm payload migrate` | Apply pending migrations (this is what CI runs) |
| `pnpm payload migrate:status` | Show applied vs pending |
| `pnpm payload migrate:down` | Roll back the last batch |
| `pnpm payload migrate:fresh` | Drop everything and re-run all (dev only) |

- `--skip-empty` / `--force-accept-warning` are useful for generating migrations in CI scripts.
- Never edit a migration that's already been applied anywhere — write a new one forward.

## MongoDB note (if the project ever uses Mongo)

Mongo is schemaless, so migrations are only needed for **data reshaping** (transforming existing documents), not for schema changes. Run them in CI or locally against the prod connection string as needed.

## Long-running servers / containers

If build-time CI can't reach the DB, run migrations at startup instead:

```ts
db: postgresAdapter({ pool: {...}, prodMigrations: migrations }) // import { migrations } from './migrations'
```

Avoid this on serverless (Vercel) — it slows cold starts. Prefer the CI `payload migrate` approach there.

## Production deploy checklist

**Env vars set on the platform:**
- `PAYLOAD_SECRET` — long, random, secret.
- `DATABASE_URI` — production Postgres connection string (reachable from CI for `payload migrate`).
- `PREVIEW_SECRET` — for the draft-preview route.
- `NEXT_PUBLIC_SERVER_URL` — public site URL (used by live preview + metadata).
- `BLOB_READ_WRITE_TOKEN` (Vercel Blob) or the relevant storage adapter credentials.

**Security:**
- Re-check every collection's access control. Default is "must be logged in"; if you allow public registration or public reads, confirm `read`/`create`/`update`/`delete` are scoped correctly.
- Confirm drafts are gated (`read: authenticatedOrPublished`) — `draft: true` does not hide them on its own.
- Enable secure cookies on auth collections (you have SSL in prod).
- Keep Payload's anti-abuse defaults (login lockout, max `depth`, GraphQL complexity limits).

**Build / runtime:**
- Build command runs migrations first (`payload migrate && next build`). A failed migration must fail the deploy.
- File storage points at a **persistent** store (Vercel Blob / S3 / GCS), never an ephemeral filesystem — uploads on ephemeral hosts vanish on restart.
- On Vercel, server uploads cap at 4.5MB; enable `clientUploads: true` on the storage adapter for larger files.
- Test a real production build locally (`next build`) before deploying — client/server bundling errors don't surface in `pnpm dev`.
- For Docker, set `output: 'standalone'` in `next.config.js` and use the multi-stage Node 24-alpine build.

## Gotchas

- Environment-specific config (a plugin enabled only in prod) means a migration generated in dev can miss prod-only entities. Generate with the prod-equivalent env, or hand-adjust the migration.
- `schedulePublish` requires a running Jobs Queue in production or scheduled publish/unpublish never executes.
