# React — AI Development Reference
> Attach when working on a React project. Covers component patterns, state management, performance, and conventions that produce reliable AI-generated components.

---

## 1. Component Design Rules

### Single Responsibility — 300 line limit
Each component does one thing. If a component exceeds 300 lines, it likely has multiple responsibilities and should be split.

```typescript
// ❌ Too many responsibilities
function UserPage() {
  // Auth logic
  // Data fetching
  // Form state
  // Validation
  // Rendering (300+ lines)
}

// ✓ Decomposed
function UserPage() {
  return <UserProfileForm userId={userId} />
}

function UserProfileForm({ userId }: { userId: string }) {
  const { user } = useUser(userId)        // Data fetching hook
  const form = useUserForm(user)          // Form state hook
  return <form>...</form>
}
```

### File naming and co-location

```
src/components/
  UserCard/
    UserCard.tsx          # Component
    UserCard.test.tsx     # Tests
    UserCard.stories.tsx  # Storybook (optional)
    index.ts              # Re-export: export { UserCard } from './UserCard'
  ui/
    Button.tsx
    Input.tsx
    Modal.tsx
```

**Name files after what they export.** `UserCard.tsx` exports `UserCard`. Not `Card.tsx`, not `index.tsx`.

---

## 2. Props Patterns

```typescript
// Always type props explicitly
interface ButtonProps {
  children: React.ReactNode
  variant?: 'primary' | 'secondary' | 'danger'
  size?: 'sm' | 'md' | 'lg'
  disabled?: boolean
  onClick?: () => void
}

// Use interface not type for props (extends is cleaner than &)
interface LinkButtonProps extends ButtonProps {
  href: string
}
```

### Spreading HTML attributes

```typescript
// Allow passing through standard HTML attributes
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string
  error?: string
}

function Input({ label, error, ...inputProps }: InputProps) {
  return (
    <div>
      <label>{label}</label>
      <input {...inputProps} />
      {error && <span className="error">{error}</span>}
    </div>
  )
}
```

---

## 3. State Management

### Local state: `useState`

Use for state that belongs to one component and its children:
```typescript
const [isOpen, setIsOpen] = useState(false)
const [count, setCount] = useState(0)
```

### Derived state: compute, don't store

```typescript
// ❌ Redundant state — will get out of sync
const [items, setItems] = useState<Item[]>([])
const [filteredItems, setFilteredItems] = useState<Item[]>([])

// ✓ Derived — always consistent
const [items, setItems] = useState<Item[]>([])
const [filter, setFilter] = useState('')
const filteredItems = items.filter(item => item.name.includes(filter))
```

### Cross-component state: Context

```typescript
// Contexts are for state that many components need — not for everything
const UserContext = createContext<{ user: User | null; logout: () => void } | null>(null)

function useUser() {
  const context = useContext(UserContext)
  if (!context) throw new Error('useUser must be used inside UserProvider')
  return context
}
```

### Server state: React Query / SWR

For data from an API, use a data-fetching library rather than `useEffect + useState`:

```typescript
// ✓ React Query — handles loading, error, caching, refetching
import { useQuery, useMutation } from '@tanstack/react-query'

function useUser(id: string) {
  return useQuery({
    queryKey: ['user', id],
    queryFn: () => fetchUser(id),
  })
}

function useUpdateUser() {
  return useMutation({
    mutationFn: (data: UpdateUserDto) => updateUser(data),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['user'] }),
  })
}

// ❌ Manual fetch with useEffect — reinvents the wheel, has race conditions
function UserProfile({ id }: { id: string }) {
  const [user, setUser] = useState(null)
  const [loading, setLoading] = useState(true)
  
  useEffect(() => {
    fetchUser(id).then(setUser).finally(() => setLoading(false))
  }, [id])
}
```

---

## 4. Custom Hooks

Extract reusable logic into custom hooks (`use` prefix required):

```typescript
// hooks/useLocalStorage.ts
function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key)
      return item ? JSON.parse(item) : initial
    } catch {
      return initial
    }
  })

  const setStoredValue = (value: T) => {
    setValue(value)
    window.localStorage.setItem(key, JSON.stringify(value))
  }

  return [value, setStoredValue] as const
}
```

**Rules for custom hooks:**
- Start with `use`
- One concern per hook (don't pack auth + data + UI state into one hook)
- Return arrays for [value, setter] pairs (like useState), objects for multiple named values

---

## 5. Performance: When to Optimize

**Default: don't optimize.** React is fast. Premature optimization creates complexity without benefit.

**Optimize when you can measure a problem**, then use:

```typescript
// memo — skip re-render if props haven't changed
// Use when: expensive render + parent re-renders often with same props
const ExpensiveList = React.memo(function ExpensiveList({ items }: { items: Item[] }) {
  return <ul>{items.map(item => <li key={item.id}>{item.name}</li>)}</ul>
})

// useMemo — memoize expensive computation
// Use when: computation is measurably slow (>16ms)
const sortedItems = useMemo(
  () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  [items]
)

// useCallback — stable function reference
// Use when: function is passed to a memoized child component
const handleClick = useCallback((id: string) => {
  onSelect(id)
}, [onSelect])
```

**Never use `memo`, `useMemo`, `useCallback` by default.** They add overhead too. Profile first.

---

## 6. Error Boundaries

```typescript
'use client'  // Required in Next.js App Router
import { Component, type ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback: ReactNode
}

interface State {
  hasError: boolean
}

class ErrorBoundary extends Component<Props, State> {
  state = { hasError: false }

  static getDerivedStateFromError() {
    return { hasError: true }
  }

  componentDidCatch(error: Error) {
    console.error('Uncaught error:', error)
    // reportToErrorTracking(error)
  }

  render() {
    if (this.state.hasError) return this.props.fallback
    return this.props.children
  }
}

// Usage
<ErrorBoundary fallback={<div>Something went wrong</div>}>
  <UserProfile />
</ErrorBoundary>
```

---

## 7. Accessibility Basics

```typescript
// ✓ Semantic HTML first
<button onClick={handleSubmit}>Submit</button>  // Not <div onClick={...}>

// ✓ Label all form inputs
<label htmlFor="email">Email</label>
<input id="email" type="email" />

// ✓ Images need alt text
<img src={avatar} alt={`${user.name}'s avatar`} />
<img src={decorativeImage} alt="" />  // Empty alt for decorative images

// ✓ ARIA when semantic HTML isn't enough
<div role="dialog" aria-labelledby="dialog-title" aria-modal="true">
  <h2 id="dialog-title">Confirm deletion</h2>
</div>
```

---

## 8. React AI Instructions for CLAUDE.md

```markdown
## React Conventions

### Component rules
- Single responsibility: one component = one concern
- 300 line limit — split if larger
- File named after default export (UserCard.tsx exports UserCard)
- Props typed with interface, not type

### State rules
- Compute derived state from source of truth — do not duplicate to state
- Server/API data: use React Query (or SWR), not useEffect + useState
- Cross-component state: Context API or Zustand (not prop drilling >2 levels)
- Local UI state (open/closed, hover, etc.): useState

### Performance
- Default: no optimization
- Add memo/useMemo/useCallback only when profiler shows a problem

### Hooks
- Custom hooks start with 'use'
- One concern per hook — do not combine unrelated logic
```

---

## Part 2 — AI Agent Readiness

---

## 9. What AI Agents Get Wrong in React (Common Failure Modes)

| Failure | Root cause | Fix via |
|---------|-----------|---------|
| Fetches data in `useEffect + useState` | Falls back to pre-React-Query patterns | CLAUDE.md rule |
| Stores derived state (duplicates source of truth) | Doesn't recognise derivability | CLAUDE.md rule + example |
| Adds `memo` everywhere "for performance" | Cargo-cults memoization | CLAUDE.md: "only after profiler" |
| Puts all state in Context | Doesn't know Zustand; over-engineers | CLAUDE.md: state decision tree |
| Puts `useEffect` for every side effect | Doesn't know derived state, event handlers | CLAUDE.md rule |
| Creates 500-line components | No size constraint given | CLAUDE.md: 300-line limit |
| Combines unrelated logic in one hook | No single-responsibility rule for hooks | CLAUDE.md: one concern per hook |
| Prop-drills 4+ levels instead of lifting | Doesn't suggest Context or Zustand | CLAUDE.md: prop drill threshold |
| Skips `key` on list items or uses index | Doesn't explain why this matters | CLAUDE.md rule |
| Tests implementation, not behaviour | Writes enzyme-style tests with RTL | Skill: create-component |
| Stores auth tokens in `localStorage` | Falls back to SPA session pattern | CLAUDE.md: auth rules; tokens in HttpOnly cookies only |
| Uses `<a href>` for internal navigation | Copies HTML instinct | CLAUDE.md rule; use router `<Link>` |
| Adds fields to a monolithic form state object indefinitely | No grouping convention; AI adds to end of flat object | CLAUDE.md: form state grouping rule; split by step/domain |
| Grows a `FormState` with duplicate fields (v1 + v2 fields coexist) | Two wizard versions share one type | CLAUDE.md: canonical types; delete v1 types when deprecating |
| Imports a heavy UI library (MUI, Ant) without `dynamic()` | Doesn't know about bundle splitting | CLAUDE.md: heavy libraries use dynamic import |

---

## 10. CLAUDE.md Snippet for React Projects

Replaces the minimal snippet from §8. Keep under 60 lines — hot memory.

```markdown
## React Conventions

### Component design
- One component = one concern. 300-line limit — split if larger.
- File named after default export: `UserCard.tsx` exports `UserCard`.
- Props typed with `interface`, not `type`. Extend HTML attributes with
  `extends React.ButtonHTMLAttributes<HTMLButtonElement>` when wrapping native elements.
- `data-testid` on every interactive element — placed in the component, not in the test.

### State — decision tree (in order)
1. Derivable from existing state? → compute it, don't store it
2. Only this component needs it? → `useState`
3. Several nearby components? → lift to closest common ancestor
4. Deeply nested or many consumers? → Context or Zustand
5. Server / API data? → React Query. Never `useEffect + useState` for fetching.

### useEffect — when it is allowed
ONLY for: subscribing to external systems (WebSocket, ResizeObserver, timers).
NOT for: data fetching, computing derived values, syncing state to state.
If you want to run something "on mount", question whether it belongs in useEffect at all.

### Performance
- Default: no memoization. React is fast without it.
- `memo`: only when profiler shows this component re-renders with unchanged props.
- `useMemo`: only when profiler shows computation >16ms.
- `useCallback`: only when passing a function to a memoized child.

### Hooks
- Custom hooks: `use` prefix, one concern per hook.
- Return `[value, setter]` (tuple) for single pairs; object for multiple named values.

### Keys in lists
- Always use a stable unique ID as `key`. Never use array index.

### NEVER
- `useEffect` for data fetching — use React Query
- Store derived state — compute it
- Add memo/useMemo/useCallback without a profiler measurement
- Prop-drill more than 2 levels — use Context or Zustand
- Write tests that assert on state variables or implementation — test user behaviour
- Store auth tokens in localStorage — use HttpOnly cookies set by a server route
- Use `<a href>` for internal navigation — use the router's `<Link>` component
- Import heavy UI libraries (MUI, Ant Design) without `dynamic()` — they will be in the main bundle for every page
- Add new fields to a monolithic form state beyond 20-30 fields — split into domain groups (e.g., `locationForm`, `detailsForm`)
```

---

## 11. Skills to Create (`.claude/skills/`)

### `.claude/skills/create-component.md`

```markdown
# Skill: Create a React Component

When asked to create a component named [Name]:

1. Create `src/components/[Name]/[Name].tsx`
   - Props interface: `interface [Name]Props { ... }`
   - Extend relevant HTML attributes if wrapping a native element
   - Add `data-testid` to every interactive or meaningful element
2. Create `src/components/[Name]/index.ts`
   - `export { [Name] } from './[Name]'`
3. Create `src/components/[Name]/[Name].test.tsx`
   - Test user-visible behaviour, not implementation
   - Use `getByRole`, `getByLabelText` — avoid `getByTestId` unless no semantic selector exists
   - One `it` per behaviour: "shows error when...", "calls onSubmit when..."

Checklist before finishing:
- [ ] Props interface exported
- [ ] No useState for derivable values
- [ ] No useEffect for data fetching
- [ ] data-testid on interactive elements
- [ ] Test file covers: happy path + one error/edge case
- [ ] Run `pnpm typecheck` and `pnpm test`
```

### `.claude/skills/create-hook.md`

```markdown
# Skill: Create a Custom Hook

When asked to create a hook named use[Name]:

1. Create `src/hooks/use[Name].ts`
2. One concern only — if it needs two unrelated things, split into two hooks
3. Return signature:
   - Single value + setter → tuple: `return [value, setValue] as const`
   - Multiple named values → object: `return { data, isLoading, error, refetch }`
4. If the hook fetches data: use React Query (`useQuery`/`useMutation`), not useState + useEffect
5. Export named, not default: `export function use[Name]() { ... }`

Checklist:
- [ ] Starts with `use`
- [ ] Single responsibility
- [ ] No raw fetch in useEffect
- [ ] TypeScript return type explicit
- [ ] Test file: `src/hooks/use[Name].test.ts` (use `renderHook` from RTL)
```

---

## 12. Subagent to Create (`.claude/agents/`)

### `.claude/agents/react-reviewer.md`

```markdown
---
name: react-reviewer
description: Reviews React components for state misuse, missing memoization rationale, hook violations, accessibility issues, and test quality
tools: Read, Glob, Grep
model: sonnet
---

You are a React expert reviewer. When asked to review files or a PR:

**Check in this order:**

1. State correctness
   - Any derived state stored in useState? (should be computed)
   - Any useEffect used for data fetching? (should be React Query)
   - Prop drilling more than 2 levels? (should be Context or Zustand)

2. Component design
   - Any component over 300 lines? (should be split)
   - Any component with multiple unrelated responsibilities?
   - Props typed with interface and HTML attributes extended where applicable?

3. Performance claims
   - Any memo/useMemo/useCallback without a comment explaining the measured problem?
   - Any key={index} in lists?

4. Hooks
   - Any custom hook with multiple unrelated concerns?
   - Any hook that fetches with useEffect + useState instead of React Query?

5. Testability
   - data-testid on interactive elements?
   - Tests assert on user behaviour, not internal state?

6. Security & Storage
   - Any auth tokens stored in localStorage or sessionStorage? (should be HttpOnly cookies)
   - Any use of `document.cookie` from client code to store sensitive values?

7. Bundle & Imports
   - Any heavy UI library (MUI, Ant Design, Recharts) imported without `dynamic()` and loaded on pages that don't need it?
   - Any monolithic form state object exceeding ~30 fields? (should be split by domain)

Report: file:line — issue — suggested fix. Do not modify any files.
```

---

## 13. Quick Audit Checklist (Yes/No)

**Component design**
- [ ] All components under 300 lines?
- [ ] Each component has one clear responsibility?
- [ ] Props typed with `interface`?
- [ ] `data-testid` on interactive elements?

**State management**
- [ ] No derived state stored in `useState`?
- [ ] No `useEffect + useState` for data fetching (React Query used instead)?
- [ ] `useEffect` only for external system subscriptions?
- [ ] Prop drilling limited to ≤2 levels?
- [ ] No `key={index}` in lists?

**Performance**
- [ ] No `memo`/`useMemo`/`useCallback` without a measured reason?
- [ ] `React.lazy` / `dynamic()` used for heavy components?

**Hooks**
- [ ] Custom hooks start with `use`?
- [ ] Each hook has one responsibility?
- [ ] Hooks return tuples for single pairs, objects for multiple values?

**Testing**
- [ ] Tests use `getByRole` / `getByLabelText` over `getByTestId` where possible?
- [ ] Tests assert on user-visible behaviour, not internal state?
- [ ] Each component has at least a happy-path test?

**Security**
- [ ] No auth tokens in `localStorage` / `sessionStorage` (use HttpOnly cookies)?
- [ ] No `document.cookie` writes with sensitive values from client components?

**Bundle**
- [ ] Heavy UI libraries (MUI, Ant, Recharts) loaded with `dynamic()` / loaded only on pages that need them?
- [ ] Monolithic form/wizard state split into domain-scoped groups when >30 fields?
- [ ] No `FormState` with duplicate fields from deprecated and current implementations?

**Navigation**
- [ ] `<Link>` (or router hook) used for all internal navigation — no plain `<a href>`?

**AI Readiness**
- [ ] CLAUDE.md has state decision tree?
- [ ] CLAUDE.md has 300-line limit rule?
- [ ] CLAUDE.md has `useEffect` allowed-uses rule?
- [ ] CLAUDE.md has auth token storage rule (no localStorage)?
- [ ] CLAUDE.md names the canonical form implementation when multiple versions exist?
- [ ] `.claude/skills/create-component.md` exists?
- [ ] `react-reviewer` agent exists?

**Score: 27 Yes = solid React + AI-ready codebase**
