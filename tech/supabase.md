# Supabase — AI Development Reference
> Attach when working on a project that uses Supabase for database, auth, or storage. Covers local development setup, RLS, migrations, and type generation.

---

## 1. Local Development Setup

**Always develop against a local Supabase instance, not production.**

```bash
# Install Supabase CLI
npm install supabase --save-dev

# Initialize Supabase in your project (creates supabase/ folder)
npx supabase init

# Start local Supabase (requires Docker)
npx supabase start

# Output includes:
# API URL:     http://127.0.0.1:54321
# DB URL:      postgresql://postgres:postgres@127.0.0.1:54322/postgres
# Studio URL:  http://127.0.0.1:54323
# Anon key:    [key]
# Service key: [key]
```

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=[anon key from supabase start output]
SUPABASE_SERVICE_ROLE_KEY=[service key — server-only, never expose to browser]
DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:54322/postgres
```

**Add to `.gitignore`:** `supabase/.branches/`, `supabase/.temp/`

---

## 2. Client Setup

```typescript
// src/lib/supabase/client.ts — Browser client (uses anon key)
import { createBrowserClient } from '@supabase/ssr'
import type { Database } from '@/types/database'

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

```typescript
// src/lib/supabase/server.ts — Server client (uses anon key + cookie auth)
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
import type { Database } from '@/types/database'

export function createClient() {
  const cookieStore = cookies()
  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return cookieStore.getAll() },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          )
        },
      },
    }
  )
}
```

```typescript
// src/lib/supabase/admin.ts — Admin client (uses service role key — server ONLY)
import { createClient } from '@supabase/supabase-js'
import type { Database } from '@/types/database'

// NEVER use this on the client side — service role bypasses RLS
export const supabaseAdmin = createClient<Database>(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
)
```

---

## 3. Database Migrations

**All schema changes go through migrations — never edit the database directly in production.**

```bash
# Create a new migration
npx supabase migration new create_users_table

# This creates: supabase/migrations/[timestamp]_create_users_table.sql
```

```sql
-- supabase/migrations/20240101000000_create_users_table.sql
create table public.profiles (
  id uuid references auth.users on delete cascade primary key,
  email text not null unique,
  name text not null,
  created_at timestamptz default now() not null
);

-- Enable RLS immediately (before adding any data)
alter table public.profiles enable row level security;
```

```bash
# Apply migrations to local DB
npx supabase db reset        # Reset + apply all migrations (development)
npx supabase db push         # Apply pending migrations to linked project (production)

# Check migration status
npx supabase migration list
```

**Rule:** Never modify existing migration files. If you need to change a column, create a new migration.

---

## 4. Row Level Security (RLS)

**RLS is your primary security layer.** When RLS is enabled on a table, all queries (including from your application) are filtered by policies. No policy = no access (even for authenticated users).

```sql
-- Enable RLS on a table
alter table public.profiles enable row level security;

-- Allow users to read their own profile
create policy "Users can read own profile"
  on public.profiles
  for select
  using (auth.uid() = id);

-- Allow users to update their own profile
create policy "Users can update own profile"
  on public.profiles
  for update
  using (auth.uid() = id)
  with check (auth.uid() = id);

-- Allow authenticated users to insert their own profile
create policy "Users can insert own profile"
  on public.profiles
  for insert
  with check (auth.uid() = id);

-- Admin can do anything (using a custom claim or role check)
create policy "Admins have full access"
  on public.profiles
  using (auth.jwt() ->> 'role' = 'admin');
```

**Common pattern: public read, authenticated write**
```sql
create policy "Public can read posts"
  on public.posts for select using (true);

create policy "Authors can create posts"
  on public.posts for insert with check (auth.uid() = author_id);

create policy "Authors can update own posts"
  on public.posts for update
  using (auth.uid() = author_id)
  with check (auth.uid() = author_id);
```

**Testing RLS policies:**
```sql
-- In Supabase Studio SQL editor: simulate a specific user
set local role authenticated;
set local "request.jwt.claims" to '{"sub": "user-uuid-here"}';
select * from public.profiles;  -- Should only return the user's own profile
```

---

## 5. Authentication Patterns

### Server-side auth check (Next.js)
```typescript
// app/dashboard/page.tsx
import { createClient } from '@/lib/supabase/server'
import { redirect } from 'next/navigation'

export default async function Dashboard() {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  
  if (!user) {
    redirect('/login')
  }
  
  return <div>Welcome, {user.email}</div>
}
```

### Middleware for route protection
```typescript
// middleware.ts
import { createServerClient } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll() },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options)
          )
        },
      },
    }
  )

  const { data: { user } } = await supabase.auth.getUser()

  if (!user && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return response
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/protected/:path*'],
}
```

---

## 6. Type Generation

Generate TypeScript types from your database schema:

```bash
# Generate types from local Supabase instance
npx supabase gen types typescript --local > src/types/database.ts

# Or from a linked remote project
npx supabase gen types typescript --project-id [your-project-id] > src/types/database.ts
```

```json
// package.json — add as a script to run after migrations
{
  "scripts": {
    "db:types": "supabase gen types typescript --local > src/types/database.ts"
  }
}
```

**Run `npm run db:types` after every migration.** The generated types ensure queries are type-safe.

```typescript
// Usage with generated types
import type { Database } from '@/types/database'

type Profile = Database['public']['Tables']['profiles']['Row']
type NewProfile = Database['public']['Tables']['profiles']['Insert']
type ProfileUpdate = Database['public']['Tables']['profiles']['Update']
```

---

## 7. Common Query Patterns

```typescript
// Select with filtering
const { data, error } = await supabase
  .from('posts')
  .select('id, title, author:profiles(name)')
  .eq('published', true)
  .order('created_at', { ascending: false })
  .limit(10)

// Insert
const { data, error } = await supabase
  .from('posts')
  .insert({ title: 'Hello', author_id: user.id })
  .select()
  .single()

// Update
const { error } = await supabase
  .from('posts')
  .update({ title: 'Updated' })
  .eq('id', postId)
  .eq('author_id', user.id)  // Always scope to the current user

// Delete
const { error } = await supabase
  .from('posts')
  .delete()
  .eq('id', postId)

// Error handling
if (error) throw new Error(`Database error: ${error.message}`)
```

---

## 8. Supabase AI Instructions for CLAUDE.md

```markdown
## Supabase Conventions

### Clients
- Browser operations: use createClient() from @/lib/supabase/client.ts (anon key)
- Server operations: use createClient() from @/lib/supabase/server.ts (anon key + cookies)
- Admin operations: use supabaseAdmin from @/lib/supabase/admin.ts (service role — server only, never client)

### Database
- All schema changes: create a migration with `npx supabase migration new [name]`
- Never modify existing migration files — create new ones
- Run `npm run db:types` after every migration to update TypeScript types
- Types live in src/types/database.ts (auto-generated — do not edit manually)

### Security
- Every new table must have RLS enabled immediately (in the same migration)
- Every RLS policy must scope to auth.uid() for user data
- Never use service role key in client components or expose in NEXT_PUBLIC_*
- Always use .eq('user_id', user.id) in queries as a defense-in-depth measure

### Local development
- Local Supabase: `npx supabase start`
- Local Studio: http://127.0.0.1:54323
- Apply migrations: `npx supabase db reset`
```
