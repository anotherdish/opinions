---
name: nextjs-conventions
description: The always-on baseline for building any Next.js App Router app — the stack, the server-first mental model, and the overarching dogma (App Router + TS, fetch on the server / mutate with Server Actions, explicit caching, stream don't block, every action is a public endpoint, defense-in-depth auth). Use whenever working in a Next.js project — starting, planning, reviewing, or any task that touches routing, components, data, mutations, state, auth, rendering, or tests. Routes to the focused sibling skills. Targets Next.js 16 + React 19.2.
---

# Next.js App Router — Conventions

The baseline that holds for **every** task in a Next.js App Router app. Load this
first; the focused skills below carry the depth. The one mental shift everything
else follows from: **you are building a server app with islands of client
interactivity, not a client app that calls the server.**

> Targets **Next.js 16+** (App Router, Turbopack default, React 19.2). The framework
> moves fast — especially **caching** and the Next 16 API renames. Verify specifics
> against the installed version with the Context7 MCP (`/vercel/next.js`) rather than
> trusting this snapshot.

## The stack

App Router · TypeScript (`strict`) · React Server Components · Server Actions for
writes · the Next 16 explicit caching model (`'use cache'`) · Tailwind · ESLint flat
config / Biome · Turbopack. Scaffold with `npx create-next-app@latest` and don't
fight the defaults.

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

## Next.js 16 changes to assume (don't generate the old way)

- **Dynamic by default**; implicit caching removed. Opt in via `'use cache'` /
  Cache Components / PPR with `cacheComponents: true`.
- `revalidateTag(tag, profile)` now needs a `cacheLife` profile (e.g. `'max'`); new
  `updateTag()` (read-your-writes, Server-Action-only) and `refresh()` (uncached
  data) APIs.
- `middleware.ts` → **`proxy.ts`** (Node.js runtime; no edge).
- **Turbopack** is the default bundler; **React Compiler** support is stable.
- `params`, `searchParams`, `cookies()`, `headers()`, `draftMode()` are **async** —
  always `await` them.
- Requirements: **Node 20.9+**, **TypeScript 5.1+**. `next lint` removed (use
  ESLint/Biome directly); `images.domains` → `remotePatterns`.

## Which skill to reach for

| Task | Skill |
|---|---|
| Scaffold/structure, routing files, Route Handlers, `proxy.ts`, config | `nextjs-app-architecture` |
| Server vs Client, `'use client'` boundary, composition, providers | `nextjs-server-client-components` |
| Reading data, `'use cache'`/`cacheLife`/`cacheTag`, PPR, revalidation | `nextjs-data-fetching-and-caching` |
| Mutations: Server Actions, forms, validation, optimistic UI | `nextjs-server-actions-and-forms` |
| Where state lives (server/URL/form/local/global) | `nextjs-state-management` |
| Login/sessions, the DAL, RBAC, picking an auth library | `nextjs-auth-and-authorization` |
| Loading/error states, streaming, metadata/SEO, images/fonts, CWV | `nextjs-rendering-and-performance` |
| Vitest + RTL vs Playwright, what to test where | `nextjs-testing` |

A real feature usually touches several (a CRUD flow = architecture + server/client +
actions + caching + rendering). Start here, then pull the specific skill.

## Anti-patterns (the cross-cutting ones)

- ❌ Reaching for `'use client'`, an API route, or a global store by reflex — each is
  an opt-in with a justification, not the default.
- ❌ Assuming Next 16 caches by default — it doesn't; opt in explicitly.
- ❌ Trusting `proxy.ts` for authorization, or trusting the client at all.
- ❌ Carrying over Next 14/15 caching/`middleware` assumptions into a 16 app.
