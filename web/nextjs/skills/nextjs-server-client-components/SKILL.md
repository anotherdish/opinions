---
name: nextjs-server-client-components
description: Decide and compose React Server Components and Client Components in the Next.js App Router — when to use 'use client', where to draw the client boundary, passing props/children across it, context providers, server-only/client-only guards, and the islands pattern. Use whenever writing React components in a Next.js app, adding interactivity, hitting "useState/onClick is not allowed in a Server Component" errors, or deciding what runs where. Targets Next.js 16 + React 19.2.
---

# Server & Client Components

The single most important mental shift in the App Router: **you are building a
server app with islands of client interactivity, not a client app that calls the
server.** Get the boundary right and everything else follows.

## The mental model

- **Server Components are the default.** Every component in `app/` is a Server
  Component unless the file (or an ancestor in the import chain) is marked
  `'use client'`. Server Components run only on the server, ship **zero JS** to the
  browser, and can be `async`.
- **Client Components** are marked with `'use client'` at the top of the file. They
  run on the server (for the initial HTML) **and** hydrate in the browser, so their
  code and dependencies ship to the client.
- `'use client'` marks an **entry point / boundary**, not "this one component."
  Every module imported by a Client Component becomes part of the client bundle.

## Decision table — what goes where

| Need | Component |
|---|---|
| Fetch data (DB, API, SDK) | **Server** (`async` component) |
| Use secrets / env / server-only libs | **Server** |
| Render large/heavy deps (markdown, syntax highlight, date libs) | **Server** (keeps them off the client) |
| `useState`, `useReducer`, `useEffect`, `useContext` | **Client** |
| Event handlers (`onClick`, `onChange`, `onSubmit`) | **Client** |
| Browser APIs (`window`, `localStorage`, `navigator`, `IntersectionObserver`) | **Client** |
| Hooks from libraries that use state/effects | **Client** |
| Animations, focus management, drag-and-drop | **Client** |
| Class components | **Client** (RSC are function-only) |

Default to Server. Add `'use client'` **only when you hit a concrete need above.**

## Dogma

1. **Push the client boundary as far down (leaf-ward) as possible.** Keep pages and
   layouts as Server Components; isolate interactivity in small client islands.
2. **A high `'use client'` opts the whole subtree out of RSC benefits.** Marking a
   layout or page `'use client'` turns everything it imports into client code.
   Don't.
3. **Server can render Client, but Client cannot import Server.** Data flows down as
   props. To put server-rendered UI "inside" a client component, pass it as
   `children`/slots, not via import.
4. **Props crossing the boundary must be serializable.** No functions (except
   Server Actions), no class instances, no `Date`→`Date` guarantees beyond what RSC
   serializes. Pass plain data; pass Server Actions for callbacks.
5. **Don't make a component a Client Component just to fetch data.** Fetch on the
   server and pass data down.

## The islands pattern (the core idiom)

A Server Component fetches and lays out; a tiny Client Component handles the one
interactive bit.

```tsx
// app/products/[id]/page.tsx  — Server Component (default)
import { getProduct } from '@/lib/products'
import { AddToCart } from './_components/add-to-cart' // client island

export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const product = await getProduct(id) // server-only data access

  return (
    <article>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      {/* serializable props cross into the client island */}
      <AddToCart productId={product.id} price={product.price} />
    </article>
  )
}
```

```tsx
// app/products/[id]/_components/add-to-cart.tsx — Client island
'use client'
import { useState } from 'react'

export function AddToCart({ productId, price }: { productId: string; price: number }) {
  const [qty, setQty] = useState(1)
  return (
    <div>
      <button onClick={() => setQty((q) => Math.max(1, q - 1))}>−</button>
      <span>{qty}</span>
      <button onClick={() => setQty((q) => q + 1)}>+</button>
      <button onClick={() => addToCart(productId, qty)}>Add · ${price * qty}</button>
    </div>
  )
}
```

The page stays a Server Component (no JS for the heading/description); only the
small interactive button ships to the browser.

## Composition: passing Server UI into Client components

Client Components can't `import` Server Components, but they **can render them as
`children`**. This keeps server-rendered content out of the client bundle even when
it's visually nested inside a client wrapper.

```tsx
// _components/tabs.tsx  — Client (holds interactive state)
'use client'
import { useState } from 'react'

export function Tabs({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(true)
  return (
    <div>
      <button onClick={() => setOpen((o) => !o)}>Toggle</button>
      {open && children}  {/* children were rendered on the server */}
    </div>
  )
}
```

```tsx
// app/page.tsx — Server Component composes them
import { Tabs } from '@/components/tabs'
import { ServerHeavyReport } from '@/components/server-heavy-report' // stays server

export default function Page() {
  return (
    <Tabs>
      <ServerHeavyReport /> {/* passed as children — not imported by Tabs */}
    </Tabs>
  )
}
```

Rule of thumb: **the component that owns the interactive state is the client
boundary; expensive/server content goes through it as a prop, not an import.**

## Context providers

React Context only works in Client Components. Create a thin client provider that
takes `children`, then render it in a (server) layout. Place providers **as deep as
possible**, not blanket-wrapping the whole tree.

```tsx
// components/theme-provider.tsx
'use client'
import { createContext, useContext, useState } from 'react'

const ThemeContext = createContext<'light' | 'dark'>('light')
export const useTheme = () => useContext(ThemeContext)

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme] = useState<'light' | 'dark'>('light')
  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>
}
```

```tsx
// app/layout.tsx — Server layout renders the client provider around children
import { ThemeProvider } from '@/components/theme-provider'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <ThemeProvider>{children}</ThemeProvider>
      </body>
    </html>
  )
}
```

## Guards: `server-only` and `client-only`

Make boundary violations fail at build time instead of leaking secrets.

```ts
// lib/db.ts — must never end up in a client bundle
import 'server-only' // throws if imported into a Client Component
export const db = createClient(process.env.DATABASE_URL!)
```

```ts
import 'client-only' // throws if imported on the server
```

Install with `npm i server-only client-only`. Put `server-only` at the top of any
module that touches secrets, the DB, or server APIs.

## Environment variables across the boundary

- Server Components/Actions can read **any** env var.
- Client Components can only read vars prefixed **`NEXT_PUBLIC_`** (inlined at build,
  so never put secrets there).

## Interleaving & "where does this run?"

- A Server Component can render Client Components and other Server Components.
- A Client Component can render Client Components and **slotted** Server Components
  (via children/props) — but anything it `import`s becomes client code.
- Client Components still render once on the server to produce initial HTML — so
  guard browser-only code: `useEffect`, or `if (typeof window !== 'undefined')`.

## React 19.2 niceties (App Router uses React Canary)

- `useActionState`, `useFormStatus`, `useOptimistic` for forms (see actions skill).
- `useEffectEvent` to keep non-reactive logic out of effect deps.
- View Transitions and `<Activity>` for animated/hidden-but-stateful UI.
- **React Compiler** (`reactCompiler: true`) auto-memoizes — once enabled, stop
  hand-writing `useMemo`/`useCallback`/`memo` except for measured hotspots.

## Anti-patterns

- ❌ `'use client'` at the top of a page/layout to "make hooks work" — it declares
  the whole subtree client. Extract a leaf island instead.
- ❌ Importing a Server Component into a Client Component (use `children`/props).
- ❌ Passing functions (non-action callbacks), class instances, or other
  non-serializable values across the boundary.
- ❌ Fetching data in a Client Component (`useEffect` + `fetch`) when a Server
  Component could fetch it directly.
- ❌ Putting one big `<Providers>` client wrapper at the root that pulls half the app
  into the client bundle — scope providers narrowly.
- ❌ Reading `process.env.SECRET` in a Client Component (undefined or, if
  `NEXT_PUBLIC_`, leaked).

## Cross-references

- Where files/segments live → `nextjs-app-architecture`
- Fetching in Server Components, caching, Suspense streaming → `nextjs-data-fetching-and-caching`
- Interactive forms, `useActionState`/`useOptimistic` → `nextjs-server-actions-and-forms`
- Where state should live (server/URL/local/global), Context vs stores → `nextjs-state-management`
- Splitting work with `<Suspense>` and `loading.tsx` → `nextjs-rendering-and-performance`
