---
name: nextjs-state-management
description: Decide where state lives in a Next.js App Router app — server state vs URL state vs form state vs local UI state vs global client state, and which tool fits each (Server Components, searchParams/nuqs, useActionState, useState/useReducer, Context, Zustand/Jotai, TanStack Query/SWR). Use when adding state, reaching for a global store or Context, syncing state to the URL, fetching on the client, or asking "where should this state go / do I need Redux/Zustand?". Targets Next.js 16 + React 19.2.
---

# State Management

In a server-first app, the first question is never "which state library?" — it's
**"does this need to be client state at all?"** Most "global state" in classic SPAs
was really *server data cached on the client*. In the App Router the server owns
that data, so the amount of client state you need collapses dramatically.

## Dogma

1. **Server state is not client state.** Data owned by the backend belongs in
   Server Components (and the Next cache), not in a client store. Don't mirror the
   database into Redux/Zustand.
2. **The URL is state.** Anything that should be shareable, bookmarkable, or survive
   reload — filters, tabs, pagination, search, sort, selected item — belongs in
   `searchParams`, not `useState`.
3. **Reach for the smallest tool that fits.** Walk the ladder below top-to-bottom;
   stop at the first that works. A global store is the *last* resort, not the default.
4. **No global stores for server data.** Use Server Components or, for client-fetched
   server data, TanStack Query / SWR — never a hand-rolled Zustand cache.
5. **Never create a store at module scope on the server.** Module-level stores are
   shared across all requests/users. Instantiate per request via a client provider.

## The state taxonomy → tool ladder

| Kind of state | Examples | Use |
|---|---|---|
| **Server state** | DB rows, API data, the authed user | **Server Components** (`async` + fetch); revalidate via cache tags. Client-fetched? **TanStack Query / SWR**. |
| **URL state** | filters, search, sort, page, active tab, selected id | **`searchParams`** (read in Server Components) + `useRouter`/`useSearchParams` or **`nuqs`** to update. |
| **Form state** | input values, validation errors, pending | Uncontrolled `<form>` + **`useActionState`** / `useFormStatus`; `useState` only for live-controlled fields. |
| **Local UI state** | modal open, dropdown, hover, accordion, current step | **`useState` / `useReducer`** in the nearest Client Component. |
| **Shared UI state** | theme, locale, toast queue, sidebar collapsed | **React Context** (thin client provider, placed as deep as possible). |
| **Global client state** | complex cross-tree client app state (rare) | **Zustand / Jotai** (last resort), instantiated **per request**. |

Default to the top. The lower you go, the more justification you need.

## Server state — the default, no library

```tsx
// Owned by the backend → read it on the server, pass it down. No store needed.
export default async function Page() {
  const orders = await getOrders() // server cache handles freshness
  return <OrderTable orders={orders} />
}
```

Mutations update server state through **Server Actions** + revalidation
(`updateTag`/`revalidateTag`), which re-renders the affected Server Components. See
`nextjs-server-actions-and-forms` and `nextjs-data-fetching-and-caching`. You almost
never need a client store for this.

## URL state — make it shareable

Read in a Server Component from `searchParams`; the page re-renders on URL change.

```tsx
// app/products/page.tsx
export default async function Products({
  searchParams,
}: {
  searchParams: Promise<{ category?: string; sort?: string; page?: string }>
}) {
  const { category, sort, page = '1' } = await searchParams
  const products = await getProducts({ category, sort, page: Number(page) })
  return <ProductGrid products={products} />
}
```

Update from a Client Component — plain Next, or `nuqs` for type-safe ergonomics:

```tsx
'use client'
import { useRouter, useSearchParams, usePathname } from 'next/navigation'

export function SortSelect() {
  const router = useRouter()
  const pathname = usePathname()
  const params = useSearchParams()
  function setSort(value: string) {
    const next = new URLSearchParams(params)
    next.set('sort', value)
    router.push(`${pathname}?${next}`)
  }
  return <select onChange={(e) => setSort(e.target.value)}>{/* ... */}</select>
}
```

```tsx
// With nuqs — useState-like API bound to the URL, typed and debounceable
'use client'
import { useQueryState, parseAsInteger } from 'nuqs'

export function Search() {
  const [q, setQ] = useQueryState('q', { shallow: false }) // shallow:false → re-runs server
  const [page, setPage] = useQueryState('page', parseAsInteger.withDefault(1))
  return <input value={q ?? ''} onChange={(e) => setQ(e.target.value || null)} />
}
```

`nuqs` is the App-Router-era pick for URL state: typed parsers, defaults, batching,
debouncing, and it triggers server re-render with `shallow: false`. Wrap the app in
its `<NuqsAdapter>` once.

## Local & shared UI state

```tsx
'use client'
import { useState } from 'react'
export function Disclosure({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false)
  return <>{/* toggle + conditional children */}</>
}
```

For state shared across a subtree (theme, toasts), use **Context** via a thin client
provider placed as deep as possible — see the provider pattern in
`nextjs-server-client-components`. Context is for low-frequency, broadly-read values;
it re-renders all consumers, so don't put rapidly-changing state in it.

## Client-fetched server state — TanStack Query / SWR

Prefer Server Components. Use a client query library only when data must be fetched
*in the browser* with rich client behavior: polling/real-time, infinite scroll,
dependent on client-only state, window-focus refetch, or heavy optimistic flows.

```tsx
'use client'
import { useQuery } from '@tanstack/react-query'

export function LivePrice({ symbol }: { symbol: string }) {
  const { data } = useQuery({
    queryKey: ['price', symbol],
    queryFn: () => fetch(`/api/price/${symbol}`).then((r) => r.json()),
    refetchInterval: 5000,
  })
  return <span>{data?.price}</span>
}
```

You can hydrate from the server: fetch in a Server Component, `dehydrate` into a
`<HydrationBoundary>`, then `useQuery` on the client with no loading flash. Treat
this as **server-state** tooling — don't put pure UI state in the query cache.

## Global client store — last resort (Zustand)

Only when you have genuinely complex, cross-tree *client* state (e.g. a canvas
editor, a multi-step wizard with shared widgets). The critical App Router rule:
**never create the store at module scope** — on the server that single instance is
shared across every request and user. Create it **per request** inside a client
provider, expose it via Context.

```tsx
// store.ts
import { createStore } from 'zustand'
export type EditorState = { tool: string; setTool: (t: string) => void }
export const createEditorStore = () =>
  createStore<EditorState>((set) => ({ tool: 'select', setTool: (tool) => set({ tool }) }))
```

```tsx
// editor-store-provider.tsx
'use client'
import { createContext, useContext, useRef } from 'react'
import { useStore } from 'zustand'
import { createEditorStore, type EditorState } from './store'

const Ctx = createContext<ReturnType<typeof createEditorStore> | null>(null)

export function EditorStoreProvider({ children }: { children: React.ReactNode }) {
  const ref = useRef<ReturnType<typeof createEditorStore>>()
  if (!ref.current) ref.current = createEditorStore() // one per render tree / request
  return <Ctx.Provider value={ref.current}>{children}</Ctx.Provider>
}

export function useEditorStore<T>(selector: (s: EditorState) => T): T {
  const store = useContext(Ctx)
  if (!store) throw new Error('Missing EditorStoreProvider')
  return useStore(store, selector)
}
```

Jotai/Redux follow the same per-request-provider rule. **Don't add Redux to a new
App Router app** unless you're migrating one or have a proven need — Server
Components + URL state + Context cover the vast majority.

## Decision checklist

Ask, in order:
1. Is it backend data? → Server Component (or TanStack Query if client-fetched). Stop.
2. Should it survive reload / be shareable? → URL (`searchParams` / `nuqs`). Stop.
3. Is it form input? → `<form>` + `useActionState`. Stop.
4. Is it local to one component/subtree? → `useState`/`useReducer`. Stop.
5. Shared by a subtree, low-frequency? → Context. Stop.
6. Genuinely complex global client state? → Zustand/Jotai, per-request provider.

## Anti-patterns

- ❌ Putting server data in Redux/Zustand and syncing it by hand (re-implements the
  Next cache; causes staleness). Let the server own it.
- ❌ `useState` for filters/tabs/pagination that should be in the URL (lost on
  reload, not shareable).
- ❌ Creating a Zustand/Redux store at module scope (shared across all server
  requests/users — a data-leak bug).
- ❌ A giant root `<Providers>` that pulls the whole app into the client bundle —
  scope providers narrowly and deep.
- ❌ Reaching for a global store before trying server state + URL + Context.
- ❌ Fetching server data in `useEffect` when a Server Component (or TanStack Query)
  is the right tool.
- ❌ Putting high-frequency state (mouse position, text input every keystroke) in
  Context — it re-renders every consumer.

## Cross-references

- Provider pattern, client boundary, serializable props → `nextjs-server-client-components`
- Server data, caching, revalidation → `nextjs-data-fetching-and-caching`
- Form state (`useActionState`, `useOptimistic`) → `nextjs-server-actions-and-forms`
- Reading `searchParams`, route structure → `nextjs-app-architecture`
