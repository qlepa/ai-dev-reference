# Next.js — Technical & AI-Readiness Audit Reference
> Attach this file to audit an existing Next.js project across two dimensions:
> 1. **Technical audit** — architecture, performance, security, SEO, testing
> 2. **AI-readiness audit** — what instructions, skills, and agents to leave so AI agents work reliably in this codebase going forward
>
> **How to use:** Paste into Claude with the project open. Ask:
> *"Audit this Next.js project against every section. For each item: current state, gap, and recommended fix or instruction to add."*
> At the end, ask Claude to produce `nextjs-audit-report.md` with traffic-light status and a prioritised action list.
>
> **Audit strategy — always start here:**
> Before reading any source file, read these in order:
> 1. `package.json` (root + frontend) — establishes the exact stack: Next.js version, router type, UI library, test framework, deploy adapter. Everything else is interpreted through this lens.
> 2. `tsconfig.json` — reveals type safety posture (`strict`, `noUncheckedIndexedAccess`, `extends`).
> 3. `next.config.ts` — shows image config, security headers, redirects, transpile packages.
> 4. `CLAUDE.md` / `frontend/CLAUDE.md` — shows what AI agents already know and what's missing.
> 5. Root `app/layout.tsx` — reveals font loading strategy, provider placement, metadata.
> Only then read individual pages and components. Starting with source files wastes tokens on details before you understand the structure.

---

## Part 1 — Technical Audit

---

## 1. Project Setup

### 1.1 Router & Version

- [ ] **Which router is in use?** — App Router (`app/`) and Pages Router (`pages/`) have fundamentally different mental models. Mixing them is a common source of AI mistakes.
- [ ] **Next.js version is documented** in CLAUDE.md and `package.json` — behaviour changed significantly between 13, 14, and 15 (PPR, Server Actions stability, caching defaults)
- [ ] **`next.config.ts` is typed** — use `import type { NextConfig }` for autocomplete and type safety
- [ ] **`.nvmrc` or `engines` field** in `package.json` pin the Node.js version

### 1.2 TypeScript

- [ ] **`strict: true`** in `tsconfig.json`
- [ ] **`noUncheckedIndexedAccess: true`** — catches `arr[0]` being undefined at runtime
- [ ] **Path alias `@/`** configured in `tsconfig.json` `paths` — AI agents use it consistently when you document it
- [ ] **`tsc --noEmit` runs in CI** — Next.js build does not always catch all type errors

### 1.3 Environment Variables

- [ ] **`.env.example` committed** — lists every variable with placeholder value and one-line comment
- [ ] **`.env.local` is gitignored** (Next.js default — verify it is not accidentally tracked)
- [ ] **Type-safe env vars** — validate with Zod at startup so the app crashes early on misconfiguration rather than silently at runtime:
  ```typescript
  // src/lib/env.ts
  import { z } from 'zod'
  const schema = z.object({
    DATABASE_URL: z.string().url(),
    NEXT_PUBLIC_APP_URL: z.string().url(),
  })
  export const env = schema.parse(process.env)
  ```
- [ ] **Rule documented in CLAUDE.md:** `NEXT_PUBLIC_` = browser-safe public config only; no prefix = server-only secrets (never log, never return in responses)

---

## 2. Architecture & Component Design

### 2.1 Server / Client Component Boundary

This is the most common area where AI agents make mistakes in App Router projects.

- [ ] **Default is Server Component** — `'use client'` is only added when a component directly uses browser APIs, event handlers, or hooks
- [ ] **`'use client'` is pushed as deep as possible** — not placed on layouts, not on page wrappers that only pass data
- [ ] **Interactive islands pattern** — large Server Component tree with small leaf-level Client Components for interactivity
- [ ] **Data never fetched in Client Components** — data comes from Server Component props, not from `useEffect` + `fetch`

```
✅ Correct boundary:
ServerPage (async, fetches data)
  └── ServerList (renders data, no interactivity)
        └── ClientFavoriteButton ('use client' — handles click)

❌ Wrong boundary:
'use client'  ← added to page because one child needs a click handler
ClientPage (now everything is client-rendered, data fetched in useEffect)
```

**AI instruction to add:**
> Never add 'use client' to page.tsx, layout.tsx, or any component whose only reason would be to pass the directive to a child. Create a new leaf component with 'use client' instead.

### 2.2 File & Folder Convention

- [ ] **Feature-based structure** inside `app/` for complex apps (not type-based):
  ```
  app/
    (marketing)/        # Route group — shares layout, no URL segment
      page.tsx
      layout.tsx
    (app)/              # Authenticated app area
      dashboard/
        page.tsx
        _components/    # Private components (underscore = not a route)
          DashboardChart.tsx
    api/
      users/route.ts
  ```
- [ ] **`_components/` or `_lib/` prefix** for private folders inside `app/` — prevents accidental routing
- [ ] **Route groups `(name)/`** used to share layouts without creating URL segments
- [ ] **Shared components outside `app/`** (e.g., `src/components/`) — not mixed with route files

### 2.3 Layouts & Templates

- [ ] **Root `layout.tsx`** sets `<html lang>`, global fonts, global providers
- [ ] **Nested layouts** used for shared chrome within sections (sidebar, breadcrumbs)
- [ ] **`template.tsx` vs `layout.tsx`**: template re-mounts on navigation (resets state), layout persists — documented choice
- [ ] **Providers are server-compatible** — Context that requires `'use client'` is wrapped in a single leaf provider, not the root layout

> **Critical trap:** Wrapping the root layout with a `'use client'` provider (e.g., `<AuthProvider>`, `<ThemeProvider>`) forces **the entire app tree** into client-rendering — every page, every component. RSC benefits disappear. Instead, identify exactly which components need the context and wrap only those, or split the provider into a thin client shell around a server-fetched value.

### 2.5 Dead Code & Parallel Implementations

- [ ] **No two implementations of the same feature** coexist in the codebase — e.g., `v1/` and `v2/` of the same wizard, or `/create-item` and `/add-item` routes doing the same thing. AI agents cannot determine which version is canonical and will edit the wrong one or create a third.
- [ ] **Deprecated code is deleted**, not commented out or left in a sibling folder
- [ ] **Canonical route for each feature is documented** in `CLAUDE.md` — e.g., "The listing wizard lives at `/dodaj-ogloszenie`. Do NOT edit files under `create-listing/` (deprecated)."
- [ ] **`<Link>` from `next/link` used for all internal navigation** — plain `<a href>` causes full page reloads and bypasses client-side routing, prefetching, and scroll restoration

### 2.4 Middleware

- [ ] **Middleware only runs on matched routes** — explicit `matcher` config prevents it running on `_next/static`, images, etc.
- [ ] **Middleware is thin** — auth check, redirect, header injection only. No heavy computation.
- [ ] **Auth check in middleware** — redirects unauthenticated users before the page renders (no flash of protected content)
  ```typescript
  // middleware.ts
  export const config = { matcher: ['/dashboard/:path*', '/api/:path*'] }
  ```

---

## 3. Data Fetching & Caching

### 3.1 Server Component Data Fetching

- [ ] **Async Server Components** fetch directly — no `useEffect`, no React Query for server data
- [ ] **`fetch` uses Next.js extended API** — `cache: 'no-store'` for SSR, `next: { revalidate: N }` for ISR, `next: { tags: [] }` for on-demand revalidation
- [ ] **Parallel fetching** with `Promise.all` — not sequential `await`:
  ```typescript
  // ✅ Parallel — total time = slowest fetch
  const [user, posts] = await Promise.all([getUser(id), getPosts(id)])

  // ❌ Sequential — total time = sum of all fetches
  const user = await getUser(id)
  const posts = await getPosts(id)
  ```
- [ ] **`generateStaticParams`** used for dynamic routes that can be statically generated (product pages, blog posts)
- [ ] **`React.cache()` wraps data-fetching functions** called from both `generateMetadata` and the page component — without it, Next.js makes two separate network round-trips per request:
  ```typescript
  // src/lib/api.ts
  import { cache } from 'react'

  // ✅ Deduplicated — same args within one request = one network call
  export const getProduct = cache(async (id: string) => {
    return fetchFromAPI(`/products/${id}`)
  })

  // Both generateMetadata and the page call getProduct(id) — only one fetch happens
  ```
- [ ] **`revalidate` is NOT set on pages that read `searchParams`** — pages that depend on `searchParams` are always dynamic and rendered per-request regardless of `revalidate`. Setting it creates a false impression of caching. Use `unstable_cache` or `React.cache()` on individual fetch calls instead.

### 3.2 Client-Side Data (When Needed)

- [ ] **React Query / SWR** used for client-side data — not raw `useEffect + fetch`
- [ ] **Initial data passed from Server Component** to React Query via `initialData` — no double-fetch on hydration:
  ```typescript
  // Server Component
  const initialPosts = await getPosts()
  return <PostList initialData={initialPosts} />

  // Client Component
  const { data } = useQuery({ queryKey: ['posts'], queryFn: getPosts, initialData })
  ```

### 3.3 Server Actions

- [ ] **Server Actions used for mutations** — not Route Handlers called from `fetch` in Client Components
- [ ] **Input validated with Zod** inside every Server Action before touching the database
- [ ] **`revalidatePath` / `revalidateTag`** called after mutations to invalidate cache
- [ ] **Error handling pattern** — Server Actions return typed results, not throw:
  ```typescript
  'use server'
  type ActionResult = { success: true; data: User } | { success: false; error: string }

  export async function updateUser(input: UpdateUserInput): Promise<ActionResult> {
    const parsed = schema.safeParse(input)
    if (!parsed.success) return { success: false, error: 'Invalid input' }
    // ...
  }
  ```
- [ ] **Server Actions are not used as GET endpoints** — they are mutations only

### 3.4 Route Handlers

- [ ] **Route Handlers used for**: webhooks, file downloads, third-party integrations (OAuth callbacks), streaming responses
- [ ] **Route Handlers NOT used for** what Server Actions cover (form mutations, authenticated writes)
- [ ] **Auth checked in every protected Route Handler** — middleware handles redirects, but Route Handlers still validate the session independently
- [ ] **Response types are explicit** — `NextResponse.json()` with correct `status` codes

---

## 4. Performance

### 4.1 Images

- [ ] **`next/image` used for all images** — automatic WebP/AVIF, lazy loading, CLS prevention via `width`/`height` or `fill`
- [ ] **`priority` set on above-the-fold images** (hero, LCP candidate) — prevents lazy loading the most important image
- [ ] **`sizes` attribute set** for responsive images — tells browser which size to download:
  ```tsx
  <Image src={hero} alt="Hero" fill sizes="(max-width: 768px) 100vw, 50vw" priority />
  ```
- [ ] **External image domains whitelisted** in `next.config.ts` `remotePatterns` (not the deprecated `domains`)
- [ ] **No `<img>` tags** — ESLint rule `@next/next/no-img-element` enforces this

### 4.2 Fonts

- [ ] **`next/font`** used for all fonts — eliminates FOUT, self-hosts, zero CLS from font loading
  ```typescript
  import { Inter } from 'next/font/google'
  const inter = Inter({ subsets: ['latin'], display: 'swap' })
  ```
- [ ] **Font loaded in root `layout.tsx`** once — not re-imported in individual components
- [ ] **`display: 'swap'`** set to prevent invisible text during font load

### 4.3 Bundle Size

- [ ] **Dynamic imports for heavy components** — charts, editors, map libraries:
  ```typescript
  const Chart = dynamic(() => import('@/components/Chart'), { ssr: false })
  ```
- [ ] **`ssr: false`** only when the component genuinely cannot run on the server
- [ ] **Heavy UI libraries (MUI, Ant Design) are dynamically imported** — these can be 300KB+ gzipped. If only one route uses MUI, it must NOT be statically imported from a shared module or it will be in the main bundle:
  ```typescript
  // ❌ Static import — MUI lands in every visitor's bundle
  import { Dialog } from '@mui/material'

  // ✅ Dynamic — MUI only loads for users who reach this route
  const CreateModal = dynamic(() => import('./CreateModal'), { ssr: false })
  ```
- [ ] **Multi-step wizard components lazy-load steps beyond Step 0** — importing all steps statically forces the user to download the full wizard bundle even if they abandon on Step 1:
  ```typescript
  import Step1 from './steps/Step1';  // Load immediately
  const Step2 = dynamic(() => import('./steps/Step2'));  // Load on demand
  const Step3 = dynamic(() => import('./steps/Step3'));
  ```
- [ ] **Bundle analyzer** configured — `@next/bundle-analyzer` run periodically to spot unexpected size increases:
  ```bash
  ANALYZE=true npm run build
  ```
- [ ] **Tree-shakeable imports** — `import { Button } from '@/components/ui'` not `import * as UI from '@/components/ui'`
- [ ] **No barrel files that re-export everything** from large libraries — breaks tree-shaking

### 4.4 Core Web Vitals

- [ ] **LCP candidate identified** and `priority` set on its image
- [ ] **No layout shift** from images (explicit dimensions), fonts (`next/font`), or ads/iframes (explicit containers)
- [ ] **`<Suspense>` boundaries** placed around slow data fetches — enables streaming, unblocks fast parts of the page
  ```tsx
  <Suspense fallback={<Skeleton />}>
    <SlowDataComponent />  {/* streams in when ready */}
  </Suspense>
  ```
- [ ] **`loading.tsx`** files placed at the right level — not too high (shows loading state for everything) not too low (no streaming benefit)

### 4.5 Static Generation

- [ ] **SSG used where content is not per-user** — blog posts, marketing pages, product catalog
- [ ] **ISR (`revalidate`)** used for content that changes occasionally — not `no-store` for everything
- [ ] **PPR (Partial Prerendering, Next.js 15)** evaluated for pages with mixed static shell + dynamic content
- [ ] **`loading.tsx` exists for every route that fetches data** — enables streaming and shows a skeleton instead of a blank page during slow fetches
- [ ] **`error.tsx` exists for every route that can fail** — without it, a thrown error in a Server Component shows the Next.js raw error page in production
- [ ] **Sitemap includes individual content pages** (e.g., `/listing/[id]`, `/blog/[slug]`), not just category/filter pages — content pages are the most indexable, and excluding them from the sitemap forces search engines to discover them via crawling only

---

## 5. Security

### 5.1 Environment Variables

- [ ] **Zero secrets in `NEXT_PUBLIC_*` variables** — these are embedded in the JS bundle, visible to all users
- [ ] **Secrets never logged** — `console.log(process.env)` in Server Components or Route Handlers exposes secrets to log aggregators
- [ ] **Secrets never returned in API responses** — strip sensitive fields explicitly before returning

### 5.2 Input Validation

- [ ] **Every Route Handler validates input with Zod** — never trust `request.json()` directly
- [ ] **Every Server Action validates input with Zod** — `'use server'` does not mean "safe from malicious input"
- [ ] **SQL/NoSQL injection prevention** — use parameterized queries, never string interpolation

### 5.3 Authentication & Authorization

- [ ] **Session validated in middleware** for protected routes — not only in the page component
- [ ] **Authorization checked in every Route Handler and Server Action** — middleware handles redirects, but individual handlers must re-validate identity
- [ ] **Never trust client-supplied user IDs** — always get the user from the session, not from request body/query params
- [ ] **Auth tokens stored in `HttpOnly` cookies, never in `localStorage`** — `localStorage` is readable by any JavaScript on the page (XSS, supply chain attacks, third-party scripts). `HttpOnly` cookies are invisible to JavaScript:
  ```typescript
  // ✅ Server-side: token never touches the browser JS environment
  // app/api/auth/login/route.ts
  response.cookies.set('accessToken', token, {
    httpOnly: true,   // JS cannot read this
    secure: true,
    sameSite: 'lax',
    maxAge: 3600,
    path: '/',
  })

  // ❌ Client-side (vulnerable)
  localStorage.setItem('accessToken', token)
  document.cookie = `accessToken=${token}` // also JS-readable without httpOnly
  ```
- [ ] **Token expiry is checked on app load** — read the `exp` claim from the JWT; if expired, call logout or trigger a refresh before the first API call fails silently with a 401
- [ ] **Refresh logic exists** — if access tokens expire (common: 1h), there must be a code path that uses `refreshToken` to obtain a new access token without requiring the user to log in again
- [ ] **Cookie set by server, read by server** — the dashboard layout (or middleware) reads the `HttpOnly` cookie server-side; client components never need to read the raw token, only derived UI state (email, isLoggedIn)

### 5.4 Security Headers

- [ ] **Security headers set in `next.config.ts`**:
  ```typescript
  const headers = [
    { key: 'X-Frame-Options', value: 'DENY' },
    { key: 'X-Content-Type-Options', value: 'nosniff' },
    { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
    { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  ]
  ```
- [ ] **CSP configured** for apps that handle sensitive data
- [ ] **CSRF protection for Server Actions** — Next.js 14+ includes built-in CSRF checks for Server Actions (same-origin only); verify this is not bypassed

### 5.5 Dependency Safety

- [ ] **`npm audit` / `pnpm audit` runs in CI** — catches known vulnerabilities
- [ ] **`@next/next` ESLint plugin enabled** — catches Next.js-specific anti-patterns at lint time

---

## 6. SEO

- [ ] **`export const metadata`** used in every `page.tsx` — at minimum: `title`, `description`
- [ ] **Dynamic metadata** via `generateMetadata` for dynamic routes:
  ```typescript
  export async function generateMetadata({ params }: Props): Promise<Metadata> {
    const product = await getProduct(params.id)
    return { title: product.name, description: product.tagline, openGraph: { images: [product.imageUrl] } }
  }
  ```
- [ ] **`metadataBase`** set in root layout — required for absolute OpenGraph URLs
- [ ] **`robots.txt`** generated via `app/robots.ts` — not a static file (easier to conditionally block staging)
- [ ] **`sitemap.xml`** generated via `app/sitemap.ts` — programmatic, stays current
- [ ] **Structured data (JSON-LD)** added for key page types (product, article, FAQ):
  ```tsx
  <script type="application/ld+json">{JSON.stringify(jsonLd)}</script>
  ```
- [ ] **`<link rel="canonical">`** set for pages that could be reached via multiple URLs

---

## 7. Testing

### 7.1 Unit & Component Tests (Vitest + React Testing Library)

- [ ] **Vitest configured** — `"test": "vitest run"` in `package.json` (not `vitest` — watch mode hangs CI)
- [ ] **React Testing Library** for component tests — test behaviour, not implementation
- [ ] **Server Components testable** — pure async functions, test them like async functions:
  ```typescript
  it('renders user name', async () => {
    const component = await UserCard({ userId: '1' })
    // render the returned JSX
  })
  ```
- [ ] **Client Components tested with RTL** — mock dependencies, test user interactions
- [ ] **MSW (Mock Service Worker)** for mocking Route Handler calls in component tests

### 7.2 E2E Tests (Playwright)

- [ ] **Playwright configured** with `baseURL` pointing to local dev server
- [ ] **Page Object Model** used — not inline selectors in test files
- [ ] **`data-testid`** on interactive elements — placed in the component, not in the test
- [ ] **Auth handled via storage state** — log in once, save state, reuse across tests:
  ```typescript
  // playwright.config.ts
  use: { storageState: 'playwright/.auth/user.json' }
  ```
- [ ] **Test DB** separated from dev DB — dedicated `TEST_DATABASE_URL`
- [ ] **Tests run in CI** against a real build (`next build && next start`), not `next dev`

---

## Part 2 — AI Agent Readiness Audit

---

## 8. What AI Agents Get Wrong in Next.js (Common Failure Modes)

Before setting up instructions, understand what will go wrong without them:

| Failure | Root cause | Fix via |
|---------|-----------|---------|
| Adds `'use client'` to `page.tsx` | Doesn't understand Server/Client boundary | CLAUDE.md rule |
| Fetches data in `useEffect` | Falls back to React SPA patterns | CLAUDE.md rule |
| Puts secrets in `NEXT_PUBLIC_*` | Doesn't know the naming convention | CLAUDE.md rule + Zod schema pointer |
| Creates `pages/api/` routes in App Router project | Confuses router versions | CLAUDE.md declares router version |
| Doesn't call `revalidatePath` after mutations | Doesn't know about ISR/cache | CLAUDE.md rule |
| Skips Zod validation in Server Actions | Treats `'use server'` as trusted input | CLAUDE.md rule + example |
| Breaks image CLS | Doesn't know to set `width`/`height` on `next/image` | CLAUDE.md rule |
| Imports large library without dynamic import | Doesn't check bundle impact | Skill: create-component |
| Creates a new Route Handler for a mutation | Doesn't know to use Server Actions | CLAUDE.md rule |
| Makes parallel routes sequential | Writes `await a; await b` instead of `Promise.all` | CLAUDE.md rule |
| Stores auth tokens in `localStorage` | Falls back to SPA session pattern; doesn't know about `HttpOnly` cookies | CLAUDE.md security rules |
| Adds `revalidate = N` to a page that reads `searchParams` | Doesn't know that dynamic pages bypass ISR | CLAUDE.md rule; remove misleading directive |
| Calls same data-fetch in `generateMetadata` and page — two round-trips | Doesn't know `React.cache()` deduplicates within a request | CLAUDE.md rule; wrap fetch with `cache()` |
| Edits the deprecated version of a feature | Two implementations exist; agent picks the wrong one | CLAUDE.md: document canonical path; delete deprecated code |
| Uses `<a href>` for internal navigation | Copies HTML instinct; doesn't know `<Link>` | CLAUDE.md rule; `nextjs-reviewer` catches it |
| Wraps root layout with `'use client'` provider | Thinks the provider is needed globally | CLAUDE.md: push providers to lowest necessary scope |
| Skips `loading.tsx` / `error.tsx` for new pages | Doesn't know the file-convention for streaming and error boundaries | Skill: create-page includes both |
| Sitemap omits individual content pages (only categories) | Copies minimal sitemap pattern | CLAUDE.md rule; sitemap audit check |

---

## 9. CLAUDE.md Snippet for Next.js Projects

Add this to the project's root `CLAUDE.md` (or `frontend/CLAUDE.md` in monorepos). Keep it under 80 lines — it is hot memory, loaded every session.

```markdown
## Next.js Conventions

### Project setup
- Router: App Router (`app/` directory). Next.js version: [version].
- TypeScript strict mode. Path alias: `@/` → `src/`.
- Env vars: `NEXT_PUBLIC_` = browser-safe public config only. No prefix = server-only secret.
  Never log secrets. Never return secrets in API responses. See `src/lib/env.ts` for validated env.

### Server vs Client Components
- Default is Server Component. Add `'use client'` ONLY when the component directly uses:
  browser APIs, event handlers, or hooks (useState, useEffect, useRef, useContext).
- Never add `'use client'` to page.tsx, layout.tsx, or any wrapper that doesn't itself need it.
  Create a leaf Client Component instead and keep the parent as Server Component.
- Never fetch data in Client Components. Pass data as props from Server Components,
  or use React Query with `initialData` from the server.

### Data fetching
- Mutations: use Server Actions (`'use server'`). Validate every input with Zod.
  Call `revalidatePath` or `revalidateTag` after any mutation.
- Queries: fetch in async Server Components. Use `Promise.all` for parallel fetches.
- Route Handlers: only for webhooks, file downloads, OAuth callbacks, streaming.
  NOT for mutations that Server Actions can handle.

### Performance rules
- Images: always `next/image`. Set `priority` on LCP-candidate images.
  Set `sizes` on responsive images. Never use `<img>` directly.
- Fonts: always `next/font`. Loaded once in root layout.tsx.
- Heavy components (charts, maps, editors): dynamic import with `next/dynamic`.

### Security rules
- Validate all input (Zod) in every Server Action and Route Handler.
  `'use server'` does not make input safe.
- Auth: check session in middleware for redirects AND re-validate in each handler independently.
  Never trust client-supplied user IDs — always read from session.
- NEVER store auth tokens in localStorage or sessionStorage — use HttpOnly cookies set
  by a server route. Client code reads only derived UI state (email, isLoggedIn), never raw tokens.
- Check token `exp` claim on app load. Implement refresh logic if access tokens expire.

### Testing
- Unit/component: `pnpm test` → `vitest run`. Tests in `*.test.tsx` next to components.
- E2E: `pnpm test:e2e` → Playwright. Page Object Model. `data-testid` in components, not tests.
- Add `data-testid` to every interactive element when creating or editing components.

### File structure
- Routes: `app/[segment]/page.tsx`
- Private files in app/: prefix with `_` (e.g., `_components/`, `_lib/`)
- Route groups: `app/(group-name)/` — shared layout, no URL segment
- Shared components: `src/components/`
- Business logic / service layer: `src/lib/`

### NEVER
- Add 'use client' to layout.tsx or page.tsx wrappers
- Put secrets in NEXT_PUBLIC_ variables
- Fetch data in useEffect when a Server Component can do it
- Use sequential await for independent fetches (use Promise.all)
- Skip Zod validation in Server Actions or Route Handlers
- Use <img> instead of next/image
- Create pages/api/ routes in an App Router project
- Store auth tokens in localStorage — always use HttpOnly cookies from a server route
- Add `revalidate = N` to pages that read `searchParams` — it has no effect (page is dynamic)
- Use `<a href>` for internal links — always use `<Link>` from next/link
- Create a page without a corresponding `loading.tsx` and `error.tsx`
- Wrap the root layout with a 'use client' provider if a narrower scope suffices
- Leave two parallel implementations of the same feature — delete the old one and document the canonical path in CLAUDE.md
```

---

## 10. Skills to Create (`.claude/skills/`)

Skills are playbooks loaded on demand. They prevent the AI from reinventing conventions each time.

### `.claude/skills/create-page.md`

```markdown
# Skill: Create a New Page

When asked to create a new page at [route]:

1. Create `app/[route]/page.tsx` — async Server Component, fetch data here
2. Create `app/[route]/loading.tsx` — Suspense fallback for the data fetch
3. Create `app/[route]/error.tsx` — Error boundary (must be 'use client')
4. Add metadata export: `export const metadata: Metadata = { title: '...', description: '...' }`
5. If the page has interactive elements, create `_components/` folder with Client Components
6. Add `data-testid` to all interactive elements
7. After creating: run `pnpm typecheck` and `pnpm test`

Do NOT:
- Fetch data in useEffect
- Add 'use client' to page.tsx unless the entire page is interactive (rare)
- Skip loading.tsx — data fetches should always have a loading state
```

### `.claude/skills/create-api-route.md`

```markdown
# Skill: Create a Route Handler

When asked to create an API endpoint at [path]:

1. Create `app/api/[path]/route.ts`
2. Export named functions for each HTTP method: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`
3. Validate request input with Zod schema (define schema at top of file)
4. Check authentication via session — never trust client-supplied userId
5. Return `NextResponse.json(data, { status: N })` with correct status codes
6. Handle errors with try/catch, return structured error: `{ error: 'message' }`

Use Server Actions instead if:
- This is a form submission or mutation triggered from a React component
- There is no need for a public HTTP endpoint (webhooks, third-party integrations)

After creating: run `pnpm typecheck`
```

### `.claude/skills/create-server-action.md`

```markdown
# Skill: Create a Server Action

1. Add `'use server'` at the top of the function (or file for action files)
2. Define Zod schema for input
3. Validate with `schema.safeParse(input)` — return typed error on failure
4. Check authentication — get user from session, never from input
5. Perform the mutation
6. Call `revalidatePath('/affected-path')` or `revalidateTag('tag')` to invalidate cache
7. Return a typed result object: `{ success: true, data } | { success: false, error: string }`

Never:
- Trust unvalidated input
- Return the full database object (strip sensitive fields)
- Throw errors — return them as typed results
```

---

## 11. Subagents to Create (`.claude/agents/`)

### `.claude/agents/nextjs-reviewer.md`

```markdown
---
name: nextjs-reviewer
description: Reviews Next.js code for Server/Client boundary violations, performance issues, security gaps, and convention violations
tools: Read, Glob, Grep
model: sonnet
---

You are a Next.js expert reviewer. When asked to review a file or PR:

**Check in this order:**

1. Server/Client boundary
   - Is 'use client' placed as deep as possible?
   - Is data fetched in Server Components, not useEffect?
   - Are large libraries dynamically imported?

2. Security
   - Is Zod validation present in every Server Action and Route Handler?
   - Are secrets accessed only on the server (no NEXT_PUBLIC_ for secrets)?
   - Is auth checked in the handler independently of middleware?

3. Performance
   - Are independent fetches parallel (Promise.all) not sequential?
   - Is next/image used instead of <img>? Is priority set on LCP candidates?
   - Is next/font used instead of CSS @import for fonts?

4. Correctness
   - Is revalidatePath/revalidateTag called after mutations?
   - Are metadata exports present on new pages?
   - Are data-testid attributes on interactive elements?

Report findings as: file:line — issue — suggested fix
Do not make any changes. Report only.
```

---

## 12. Hooks for Next.js Projects (`.claude/settings.json`)

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "comment": "Typecheck after every file edit",
            "command": "cd frontend/nextjs && pnpm typecheck 2>&1 | tail -10"
          }
        ]
      },
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "comment": "Block writes to next.config.ts without confirmation",
            "command": "bash -c 'echo $CLAUDE_FILE_PATH | grep -q next.config && echo \"WARNING: Modifying next.config.ts — ensure this was explicitly requested\" >&2 || true'"
          }
        ]
      }
    ]
  }
}
```

---

## 13. `.cursor/rules/frontend.mdc` Template

For teams using Cursor alongside Claude Code:

```markdown
---
description: Next.js App Router rules for frontend development
globs: ["frontend/**/*.tsx", "frontend/**/*.ts", "src/app/**/*.tsx", "src/components/**/*.tsx"]
alwaysApply: false
---

## Next.js Rules

### Server vs Client Components
- Default: Server Component. Add 'use client' only at the leaf that needs it.
- Never fetch in useEffect. Use async Server Components or React Query with initialData.

### Data mutations
- Use Server Actions for all form submissions and mutations from React components.
- Validate input with Zod. Call revalidatePath after every mutation.

### Performance
- Images: next/image only. Set priority on hero images. Set sizes for responsive images.
- Heavy libraries (charts, maps): next/dynamic with ssr: false.

### Security
- Validate all input with Zod in Server Actions and Route Handlers.
- Never put secrets in NEXT_PUBLIC_ variables.
- Always check auth independently in each handler — middleware redirects are not enough.

### Conventions
- New pages: page.tsx + loading.tsx + error.tsx
- Interactive elements: add data-testid
- Parallel fetches: Promise.all, never sequential await
```

---

## 14. Quick Audit Checklist (Yes/No)

Copy and fill in for a fast assessment. Ask Claude: *"Answer Yes/No for each item based on the current codebase."*

**Setup**
- [ ] Router version documented in CLAUDE.md?
- [ ] TypeScript strict mode enabled?
- [ ] `tsc --noEmit` in CI?
- [ ] `.env.example` committed with all variables?
- [ ] Type-safe env validation (Zod or equivalent)?

**Architecture**
- [ ] `'use client'` only on leaf components?
- [ ] No data fetching in `useEffect` when Server Component can do it?
- [ ] Feature-based folder structure (not type-based)?
- [ ] Route groups used to share layouts without URL segments?
- [ ] Middleware has explicit `matcher` config?
- [ ] Root layout NOT wrapped in a `'use client'` provider that forces the whole tree client?
- [ ] No parallel/deprecated implementations of the same feature coexisting?
- [ ] All internal navigation uses `<Link>` (not plain `<a href>`)?

**Data Fetching**
- [ ] Independent fetches use `Promise.all`?
- [ ] Mutations use Server Actions (not Route Handlers)?
- [ ] Every Server Action validates input with Zod?
- [ ] `revalidatePath`/`revalidateTag` called after mutations?
- [ ] React Query (if used) receives `initialData` from Server Component?
- [ ] Functions called from both `generateMetadata` and the page wrapped with `React.cache()`?
- [ ] All shared data-fetching functions wrapped with `React.cache()` (so two Server Components on the same page calling `getListings()` produce one HTTP call, not two)?
- [ ] No `revalidate = N` on pages that read `searchParams` (it is silently ignored)?
- [ ] No sequential dependent waterfall where the API could be redesigned to return composite data in one call?

**Performance**
- [ ] `next/image` for all images with `priority` on LCP candidates?
- [ ] `next/font` for all fonts (no `<link>` to Google Fonts)?
- [ ] Heavy UI libraries (MUI, Ant Design) loaded with `dynamic()` only on routes that need them?
- [ ] Multi-step wizard components lazy-load steps beyond Step 0?
- [ ] Heavy libraries use dynamic imports?
- [ ] `<Suspense>` boundaries around slow fetches?
- [ ] Bundle analyzer run recently?
- [ ] `loading.tsx` present for every route that fetches data?
- [ ] `error.tsx` present for every route that can throw?

**Security**
- [ ] Zero secrets in `NEXT_PUBLIC_*` variables?
- [ ] All Route Handlers validate input with Zod?
- [ ] Auth re-validated in handlers, not only in middleware?
- [ ] Security headers in `next.config.ts` (X-Frame-Options, X-Content-Type-Options, etc.)?
- [ ] Auth tokens in `HttpOnly` cookies (not localStorage / sessionStorage)?
- [ ] Token expiry checked on app load; refresh logic exists?

**SEO**
- [ ] `metadata` export in every `page.tsx`?
- [ ] `generateMetadata` in dynamic routes?
- [ ] `metadataBase` set in root layout (required for absolute OG image URLs)?
- [ ] `sitemap.ts` includes individual content pages (not only category/filter pages)?
- [ ] `robots.ts` present?

**Testing**
- [ ] Vitest configured with `vitest run` (not watch)?
- [ ] Playwright E2E tests exist?
- [ ] `data-testid` on interactive elements?
- [ ] Tests run against `next build && next start` in CI (not `next dev`)?

**AI Readiness**
- [ ] `frontend/CLAUDE.md` (or equivalent) with Next.js-specific rules?
- [ ] Server/Client boundary rule explicitly stated?
- [ ] Auth token storage rule in CLAUDE.md (no localStorage)?
- [ ] Canonical routes for each major feature documented in CLAUDE.md?
- [ ] `.claude/skills/create-page.md` exists?
- [ ] `nextjs-reviewer` agent exists?
- [ ] Typecheck hook in `.claude/settings.json`?
- [ ] `.cursor/rules/frontend.mdc` exists?

---

## 15. Scoring

Count Yes answers (max 54):

| Score | Assessment |
|-------|-----------|
| 47–54 | AI-ready Next.js project — agents will work correctly and consistently |
| 36–46 | Good foundation — address security and boundary gaps before scaling AI |
| 20–35 | Significant gaps — agents will make common mistakes; add CLAUDE.md rules first |
| <20 | High risk — add type safety and instruction files before using AI agents |
