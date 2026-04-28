# Astro — AI Development Reference
> Attach this file when working on an Astro project. Covers the Islands architecture, .astro file structure, content collections, integrations, and patterns that AI tools frequently get wrong. If no base file (`fresh.md`, `new.md`, or `legacy.md`) is attached alongside this one, run a standalone audit: treat each section's guidelines as a checklist, assess the current state (✅ pass / ⚠️ partial / ❌ missing), and save a report as `astro-audit-YYYY-MM-DD.md`.

---

## 1. Core Mental Model: Islands Architecture

Astro ships **zero JavaScript by default**. Every `.astro` component renders to static HTML at build time. JavaScript is added back only where explicitly needed — these are called "islands."

```
Static HTML page (no JS)
├── Header (static HTML)
├── Hero (static HTML)
├── InteractiveCounter (island — React/Vue/Svelte with JS)  ← explicit opt-in
└── Footer (static HTML)
```

**AI common mistake:** Adding `client:*` directives to components that don't need interactivity. Always ask: "Does this component need JavaScript at all?"

---

## 2. .astro File Anatomy

```astro
---
// Component Script (server-side, runs at build time or request time)
// This block is TypeScript — it has full access to Node.js APIs
import Layout from '../layouts/Layout.astro'
import { getCollection } from 'astro:content'

// Props — like function arguments for this component
interface Props {
  title: string
  author?: string
}
const { title, author = 'Anonymous' } = Astro.props

// Fetch data — runs at build/request time, never in the browser
const posts = await getCollection('blog')
---

<!-- Template — JSX-like but compiles to HTML, not React -->
<Layout title={title}>
  <h1>{title}</h1>
  {author && <p>By {author}</p>}
  
  <!-- Loop with map -->
  <ul>
    {posts.map(post => (
      <li><a href={`/blog/${post.slug}`}>{post.data.title}</a></li>
    ))}
  </ul>
</Layout>

<style>
  /* Scoped CSS — applies only to this component */
  h1 { color: navy; }
</style>
```

### Key differences from JSX
- `class` not `className` (it's HTML, not React)
- `---` fences separate script from template (not a comment)
- No `return` statement in the template
- `{expression}` works but `{statement}` does not (use `.map()` for loops)
- Attributes use `set:html={htmlString}` for raw HTML (equivalent of `dangerouslySetInnerHTML`)

---

## 3. Client Directives (Islands)

Add interactivity to any UI framework component:

| Directive | When JS loads | Use case |
|-----------|--------------|----------|
| `client:load` | Immediately on page load | Critical interactive UI (nav, forms) |
| `client:idle` | When browser is idle | Non-critical UI (chat widget) |
| `client:visible` | When element enters viewport | Below-fold content (comments, maps) |
| `client:media="(max-width: 768px)"` | When media query matches | Mobile-only interactivity |
| `client:only="react"` | Immediately, no SSR | Components that must not SSR (browser-only APIs) |

```astro
---
import ReactCounter from '../components/Counter.tsx'
import VueModal from '../components/Modal.vue'
---

<!-- Hydrate immediately -->
<ReactCounter client:load />

<!-- Hydrate when visible -->
<VueModal client:visible />

<!-- Static — no JS, renders HTML only -->
<ReactCounter />
```

**Rule:** Default to no directive (static). Add `client:load` only when the component needs user interaction. Use `client:visible` for anything below the fold.

---

## 4. Rendering Modes

Astro supports three rendering modes, configured per route:

```typescript
// astro.config.mjs
import { defineConfig } from 'astro/config'

export default defineConfig({
  output: 'static',   // Default: fully static, all pages pre-rendered
  // output: 'server', // All pages server-rendered (SSR)
  // output: 'hybrid', // Mix: static by default, opt-in to SSR per route
})
```

### Per-route rendering (hybrid mode)
```astro
---
// This route is server-rendered (not pre-built)
export const prerender = false

// Access request-time data
const session = Astro.cookies.get('session')
---
```

```astro
---
// This route is statically pre-rendered (in hybrid/server mode)
export const prerender = true
---
```

### When to use each mode
- `static` (default) — blogs, marketing sites, docs
- `server` — user dashboards, personalized content, real-time data
- `hybrid` — mostly static with a few dynamic routes (e.g., static blog + server-rendered dashboard)

---

## 5. Content Collections

Type-safe content management for markdown, MDX, and JSON files.

```
src/content/
  blog/
    first-post.md
    second-post.mdx
  authors/
    alice.json
  config.ts       # Schema definitions
```

### Schema definition
```typescript
// src/content/config.ts
import { defineCollection, z } from 'astro:content'

const blog = defineCollection({
  type: 'content',  // markdown/MDX
  schema: z.object({
    title: z.string(),
    pubDate: z.date(),
    author: z.string(),
    tags: z.array(z.string()).default([]),
    draft: z.boolean().default(false),
  }),
})

const authors = defineCollection({
  type: 'data',  // JSON/YAML
  schema: z.object({
    name: z.string(),
    bio: z.string(),
  }),
})

export const collections = { blog, authors }
```

### Querying collections
```typescript
import { getCollection, getEntry } from 'astro:content'

// All posts, filter drafts
const posts = await getCollection('blog', ({ data }) => !data.draft)

// Sort by date
const sorted = posts.sort((a, b) => b.data.pubDate.valueOf() - a.data.pubDate.valueOf())

// Single post by slug
const post = await getEntry('blog', 'first-post')
const { Content } = await post.render()
```

```astro
---
const { Content } = await post.render()
---
<article>
  <h1>{post.data.title}</h1>
  <Content />  <!-- Renders the markdown/MDX content -->
</article>
```

---

## 6. Integrations

Add capabilities via integrations (not all are needed — install only what you use):

```typescript
// astro.config.mjs
import { defineConfig } from 'astro/config'
import react from '@astrojs/react'
import tailwind from '@astrojs/tailwind'
import mdx from '@astrojs/mdx'
import sitemap from '@astrojs/sitemap'

export default defineConfig({
  integrations: [
    react(),      // Enable React components
    tailwind(),   // Enable Tailwind CSS
    mdx(),        // Enable MDX in content collections
    sitemap(),    // Auto-generate sitemap.xml
  ],
  site: 'https://example.com',  // Required for sitemap
})
```

**Common integrations:**
- `@astrojs/react` / `@astrojs/vue` / `@astrojs/svelte` — UI framework support
- `@astrojs/tailwind` — Tailwind CSS
- `@astrojs/mdx` — MDX support in content
- `@astrojs/sitemap` — automatic sitemap generation
- `@astrojs/image` — image optimization
- `@astrojs/db` — Astro's built-in SQLite DB (for small projects)

---

## 7. Layouts

```astro
---
// src/layouts/Layout.astro
interface Props {
  title: string
  description?: string
}
const { title, description = 'Default description' } = Astro.props
---

<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width" />
    <title>{title}</title>
    {description && <meta name="description" content={description} />}
  </head>
  <body>
    <header><!-- navigation --></header>
    <main>
      <slot />  <!-- Page content goes here -->
    </main>
    <footer><!-- footer --></footer>
  </body>
</html>
```

Multiple named slots:
```astro
<!-- In layout -->
<slot name="sidebar" />
<slot />  <!-- default slot -->

<!-- In page -->
<Fragment slot="sidebar">
  <nav>...</nav>
</Fragment>
<p>Main content in default slot</p>
```

---

## 8. Routing

Astro uses file-based routing under `src/pages/`:

```
src/pages/
  index.astro          → /
  about.astro          → /about
  blog/
    index.astro        → /blog
    [slug].astro       → /blog/:slug (dynamic)
    [...path].astro    → /blog/:path* (catch-all)
  api/
    users.ts           → /api/users (API endpoint)
```

### Dynamic routes (static mode)
```astro
---
// src/pages/blog/[slug].astro
import { getCollection } from 'astro:content'

export async function getStaticPaths() {
  const posts = await getCollection('blog')
  return posts.map(post => ({
    params: { slug: post.slug },
    props: { post },
  }))
}

const { post } = Astro.props
const { Content } = await post.render()
---

<article>
  <h1>{post.data.title}</h1>
  <Content />
</article>
```

`getStaticPaths()` is required for dynamic routes in static mode — it tells Astro which pages to pre-build.

---

## 9. Astro AI Instructions for CLAUDE.md

Add this section to CLAUDE.md when working on an Astro project:

```markdown
## Astro Conventions

### Version and output mode
Astro version: [version]. Output mode: [static / server / hybrid].

### Islands principle
- Default: no JavaScript. Static HTML only.
- Add client:load only for components that need user interaction
- Prefer client:visible for below-fold interactive content
- Never add client:* to purely presentational components

### File structure
- src/pages/ — file-based routes
- src/layouts/ — page layout wrappers
- src/components/ — reusable .astro and framework components
- src/content/ — content collections (blog posts, data files)
- src/content/config.ts — collection schemas (always define schema here)
- public/ — static assets served as-is

### .astro syntax rules
- Use class not className
- --- fences are TypeScript, not markdown
- No return statement in templates
- Use .map() for loops, not for...of
- set:html for raw HTML rendering

### Content collections
- Always define schema in src/content/config.ts using Zod
- Use getCollection() with a filter function to exclude drafts
- Use getEntry() for single entries by slug

### Do NOT
- Add 'use client' to .astro files (that's React syntax — use client:load directive)
- Fetch data inside client islands (fetch in the ---script section and pass as props)
- Use document or window in .astro scripts (they run at build time, not in browser)
```
