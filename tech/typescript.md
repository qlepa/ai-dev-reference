# TypeScript — AI Development Reference
> Attach this file when working on a TypeScript project. Covers strict configuration, type safety patterns, and conventions that produce reliable AI-generated code.

---

## 1. Strict Mode Configuration

**Always enable strict mode.** Without it, TypeScript is a linter with autocomplete, not a type system.

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,              // Enables all strict checks below
    "noUncheckedIndexedAccess": true,  // array[0] is T | undefined, not T
    "exactOptionalPropertyTypes": true, // {a?: string} ≠ {a: string | undefined}
    "noImplicitReturns": true,   // Function must return on all paths
    "noFallthroughCasesInSwitch": true
  }
}
```

`"strict": true` enables: `strictNullChecks`, `strictFunctionTypes`, `strictBindCallApply`, `strictPropertyInitialization`, `noImplicitAny`, `noImplicitThis`, `alwaysStrict`.

**Add to CLAUDE.md:** "This project uses TypeScript strict mode. Never use `any`. Use `unknown` and narrow with type guards."

---

## 2. The No-Any Rule

`any` disables the type system. It is contagious — one `any` propagates through the code.

```typescript
// ❌ Never
const data: any = fetchData()
data.user.name  // No error — also no safety

// ✓ Always — use unknown and narrow
const data: unknown = fetchData()
if (isUser(data)) {
  data.name  // TypeScript now knows this is safe
}

// Type guard
function isUser(value: unknown): value is User {
  return typeof value === 'object' && value !== null && 'name' in value
}
```

**Acceptable use of `any`:** Generated code (`.d.ts` files), third-party library types that haven't been updated yet (use `unknown` in your own code and cast only at the boundary).

---

## 3. Data Models in One Place

Define shared types once. Never duplicate.

```typescript
// src/types/index.ts — single source of truth
export interface User {
  id: string
  email: string
  name: string
  createdAt: Date
}

export interface ApiResponse<T> {
  data: T
  error: string | null
  timestamp: string
}

// DTOs (Data Transfer Objects) — what crosses the API boundary
export interface CreateUserDto {
  email: string
  name: string
}

export interface UpdateUserDto {
  name?: string
}
```

```typescript
// src/lib/user.ts — imports from types, never redefines
import type { User, CreateUserDto } from '@/types'

export async function createUser(dto: CreateUserDto): Promise<User> {
  // ...
}
```

**AI instruction:** When creating new types, check `src/types/` first. Never define the same interface in two places. If a type is used in more than one file, move it to `src/types/`.

---

## 4. Narrowing Patterns

TypeScript is only as good as your narrowing. These patterns handle common cases:

```typescript
// Discriminated unions — the most powerful narrowing tool
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string }

function handle(result: Result<User>) {
  if (result.success) {
    console.log(result.data.name)  // TypeScript knows data exists
  } else {
    console.log(result.error)  // TypeScript knows error exists
  }
}

// Type predicates
function isError(value: unknown): value is Error {
  return value instanceof Error
}

// Exhaustive check — TypeScript will error if you miss a case
function assertNever(value: never): never {
  throw new Error(`Unhandled value: ${value}`)
}

type Shape = 'circle' | 'square' | 'triangle'
function describeShape(shape: Shape) {
  switch (shape) {
    case 'circle': return 'round'
    case 'square': return 'four sides'
    case 'triangle': return 'three sides'
    default: return assertNever(shape)  // Error if a new shape is added to the union
  }
}
```

---

## 5. Generic Patterns

Use generics to write reusable typed functions:

```typescript
// Generic repository pattern
interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T | null>
  create(data: Omit<T, 'id'>): Promise<T>
  update(id: string, data: Partial<Omit<T, 'id'>>): Promise<T>
  delete(id: string): Promise<void>
}

// Generic API response wrapper
async function apiFetch<T>(url: string): Promise<Result<T>> {
  try {
    const response = await fetch(url)
    if (!response.ok) return { success: false, error: response.statusText }
    const data: T = await response.json()
    return { success: true, data }
  } catch (error) {
    return { success: false, error: String(error) }
  }
}

// Usage — T is inferred
const result = await apiFetch<User>('/api/users/123')
```

---

## 6. Utility Types

Built-in TypeScript utility types — use these instead of duplicating types:

```typescript
// Partial — makes all properties optional
type UpdateUser = Partial<User>

// Required — makes all properties required
type RequiredUser = Required<Partial<User>>

// Pick — keep only specified properties
type UserPreview = Pick<User, 'id' | 'name'>

// Omit — exclude specified properties
type CreateUserDto = Omit<User, 'id' | 'createdAt'>

// Record — typed object with specific key/value types
type UserMap = Record<string, User>

// ReturnType — extract return type of a function
type CreateResult = ReturnType<typeof createUser>

// Parameters — extract parameter types
type CreateParams = Parameters<typeof createUser>[0]
```

---

## 7. Type-Safe Environment Variables

```typescript
// src/env.ts — validate at startup, not at runtime
import { z } from 'zod'

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  PORT: z.coerce.number().default(3000),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
})

export const env = envSchema.parse(process.env)
// Now env.DATABASE_URL is string (not string | undefined)
// If any required variable is missing, the app crashes at startup with a clear error
```

---

## 8. Type Checking in CI

```json
// package.json
{
  "scripts": {
    "typecheck": "tsc --noEmit"
  }
}
```

```yaml
# In GitHub Actions
- name: Type check
  run: npm run typecheck
```

`--noEmit` runs the type checker without generating JavaScript files. Fast, pure validation.

**Rule:** Type checking must pass before any PR merges. A PR with type errors that "works" is a future bug.

---

## 9. TypeScript AI Instructions for CLAUDE.md

```markdown
## TypeScript Conventions

- strict mode: true in tsconfig.json (noUncheckedIndexedAccess also enabled)
- No any type — use unknown and narrow with type guards
- Shared types live in src/types/index.ts — check there before defining new types
- DTOs (API input/output shapes) defined in src/types/dtos.ts
- Use discriminated unions for success/error result types
- Use utility types (Partial, Omit, Pick) instead of duplicating type definitions
- Type check command: npm run typecheck (runs tsc --noEmit)
- Type errors block PR merges via CI
```
