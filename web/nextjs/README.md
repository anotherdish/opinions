# Next.js App Router — Claude Code Plugin

A Claude Code **plugin** of skills for **creating and maintaining a quintessential
Next.js application** with the App Router and React Server/Client Components. These
encode the *ideal and dogmatic* way to build modern Next.js apps.

> Target: **Next.js 16+** (App Router, Turbopack default, React 19.2). Where the
> framework moves fast — especially **caching** — verify specifics against the
> installed version using the Context7 MCP (`/vercel/next.js`).

## Install as a plugin

```sh
/plugin marketplace add <owner/repo>          # the repo hosting this plugin
/plugin install nextjs-app-router@<marketplace-name>
```

Or browse interactively with `/plugin`. Once enabled, the skills below activate
automatically when a task matches their description.

## The skills

| Skill | Use it when |
|---|---|
| [`nextjs-app-architecture`](./skills/nextjs-app-architecture/SKILL.md) | Scaffolding an app, adding routes, organizing folders, file conventions (layouts/pages/loading/error/route groups/dynamic/parallel/intercepting), Route Handlers, `proxy.ts`, config. |
| [`nextjs-server-client-components`](./skills/nextjs-server-client-components/SKILL.md) | Writing React components, deciding Server vs Client, drawing the `'use client'` boundary, composition, context providers, `server-only`/`client-only`. |
| [`nextjs-data-fetching-and-caching`](./skills/nextjs-data-fetching-and-caching/SKILL.md) | Reading data, static vs dynamic, the Next 16 caching model (`'use cache'`, `cacheLife`, `cacheTag`, Cache Components/PPR), Suspense streaming, ISR, revalidation. |
| [`nextjs-server-actions-and-forms`](./skills/nextjs-server-actions-and-forms/SKILL.md) | Writing data: Server Actions, forms, `useActionState`/`useFormStatus`, validation, auth, optimistic UI, progressive enhancement. |
| [`nextjs-state-management`](./skills/nextjs-state-management/SKILL.md) | Deciding where state lives — server vs URL vs form vs local vs global — and the tool for each (Server Components, `searchParams`/nuqs, Context, Zustand/Jotai, TanStack Query/SWR). |
| [`nextjs-auth-and-authorization`](./skills/nextjs-auth-and-authorization/SKILL.md) | Login/sessions, the Data Access Layer pattern, defense-in-depth (`proxy.ts` is not a security boundary), RBAC, DTOs, and picking Auth.js/Clerk/WorkOS. |
| [`nextjs-testing`](./skills/nextjs-testing/SKILL.md) | The unit-vs-E2E split: Vitest + RTL for client components/Server Action logic/utils, Playwright for E2E and async Server Components, MSW for mocking. |
| [`nextjs-rendering-and-performance`](./skills/nextjs-rendering-and-performance/SKILL.md) | Loading/error states, streaming, PPR, Metadata API & SEO, `next/image`/`next/font`/`next/script`, reducing client JS, Core Web Vitals. |

They cross-reference each other; a real task usually touches several (e.g. a CRUD
feature = architecture + server/client + actions + caching + rendering).

## The overarching dogma

1. **App Router + TypeScript, always.** No new `pages/`, no new `.jsx`.
2. **Server-first.** Server Components are the default; `'use client'` is an opt-in
   for interactivity, pushed to leaf islands.
3. **Fetch on the server, mutate with Server Actions.** Don't build API routes for
   your own UI; reserve Route Handlers for external/raw-HTTP consumers.
4. **Caching is explicit (Next 16).** Everything is dynamic by default; opt into
   caching with `'use cache'` + `cacheLife` + `cacheTag`, and revalidate after writes.
5. **Stream, don't block.** Use `<Suspense>`/`loading.tsx`; request-time data lives
   inside Suspense boundaries (PPR).
6. **Every Server Action is a public endpoint.** Authenticate, authorize, and
   validate inside each one.
7. **Use the platform primitives** — Metadata API, `next/image`, `next/font`,
   `next/link` — instead of hand-rolling head tags, `<img>`, fonts, or navigation.
8. **Colocate, keep pages thin.** Route-specific code next to the route; shared data
   access and logic in `lib/`.
9. **Defense-in-depth for auth.** Verify in the Data Access Layer, not just
   `proxy.ts`; check identity *and* resource ownership in every action/handler.
10. **Test the split right.** Vitest + RTL for client components and Server Action
    logic; Playwright for E2E and async Server Components.

## Notable Next.js 16 changes these skills assume

- **Dynamic by default**; implicit caching removed. `'use cache'` / Cache Components
  / PPR via `cacheComponents: true`.
- `revalidateTag(tag, profile)` now needs a `cacheLife` profile; new `updateTag()`
  (read-your-writes) and `refresh()` (uncached data) Server-Action APIs.
- `middleware.ts` → **`proxy.ts`** (Node.js runtime).
- **Turbopack** is the default bundler; **React Compiler** support is stable.
- `params`, `searchParams`, `cookies()`, `headers()`, `draftMode()` are **async**.
- Requirements: **Node 20.9+**, **TypeScript 5.1+**. `next lint` removed (use ESLint/Biome).
