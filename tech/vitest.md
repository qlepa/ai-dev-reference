# Vitest — AI Development Reference
> Attach when configuring Vitest or writing unit/integration tests with AI assistance. The most important rule is in Section 1.

---

## 1. CRITICAL: Use `vitest run` Not `vitest`

`vitest` (default) runs in **watch mode** — it watches for file changes and never exits. AI agents run in non-interactive terminals: they hang forever waiting for a prompt that never comes.

**Always configure package.json so `npm test` uses `vitest run`:**

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui"
  }
}
```

**Add to CLAUDE.md:**
> Run `npm test` to execute tests. Never run `vitest` directly — it enters watch mode and hangs. Use `npm run test:watch` only for manual interactive development.

---

## 2. Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import path from 'path'

export default defineConfig({
  test: {
    // Test environment
    environment: 'node',     // Default: Node.js (for server-side code)
    // environment: 'jsdom',  // For browser APIs (React components, DOM manipulation)
    // environment: 'happy-dom', // Faster alternative to jsdom

    // Globals — avoid needing to import describe/it/expect in every file
    globals: true,

    // Setup files run before each test file
    setupFiles: ['./tests/setup.ts'],

    // Coverage configuration
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'json'],
      exclude: [
        'node_modules/',
        'dist/',
        '*.config.*',
        'src/types/**',    // Type-only files
        '**/*.d.ts',
      ],
      thresholds: {
        statements: 80,
        branches: 75,
        functions: 80,
        lines: 80,
      }
    },

    // Timeouts
    testTimeout: 10000,     // 10s per test (increase for DB tests)
    hookTimeout: 10000,
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src')
    }
  }
})
```

### Setup file

```typescript
// tests/setup.ts
import { beforeAll, afterAll, afterEach } from 'vitest'

// Example: reset database between tests
beforeAll(async () => {
  // Connect to test database
})

afterAll(async () => {
  // Disconnect
})

afterEach(async () => {
  // Clean up state between tests
})
```

---

## 3. Test Structure

```typescript
// src/lib/auth.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { validateToken, createToken } from '@/lib/auth'

describe('auth', () => {
  describe('validateToken', () => {
    it('returns payload for valid token', () => {
      const token = createToken({ userId: 'u1' })
      const result = validateToken(token)
      expect(result.userId).toBe('u1')
    })

    it('throws AuthError for expired token', () => {
      expect(() => validateToken('expired-token')).toThrow('Token expired')
    })

    it('throws AuthError for malformed token', () => {
      expect(() => validateToken('')).toThrow()
    })
  })
})
```

**Test naming convention:**
- `describe` — the module or function being tested
- `it` / `test` — one specific behavior, starting with "returns", "throws", "calls", "updates"
- Be specific: `'throws AuthError for expired token'` not `'handles errors'`

---

## 4. Mocking

```typescript
import { vi, describe, it, expect, beforeEach } from 'vitest'

// Mock a module
vi.mock('@/lib/email', () => ({
  sendEmail: vi.fn().mockResolvedValue({ messageId: 'mock-id' }),
}))

// Mock a specific function
import { sendEmail } from '@/lib/email'
const mockSendEmail = vi.mocked(sendEmail)

describe('userService', () => {
  beforeEach(() => {
    vi.clearAllMocks()  // Reset call counts and return values between tests
  })

  it('sends welcome email on user creation', async () => {
    await createUser({ email: 'new@test.com', name: 'Alice' })
    
    expect(mockSendEmail).toHaveBeenCalledOnce()
    expect(mockSendEmail).toHaveBeenCalledWith({
      to: 'new@test.com',
      template: 'welcome',
    })
  })
})
```

### Spy on methods (without replacing)

```typescript
const consoleSpy = vi.spyOn(console, 'error').mockImplementation(() => {})

// After test
consoleSpy.mockRestore()
```

### Mock timers

```typescript
beforeEach(() => {
  vi.useFakeTimers()
})

afterEach(() => {
  vi.useRealTimers()
})

it('retries after delay', async () => {
  const promise = retryWithDelay(fn, { delay: 1000, attempts: 3 })
  
  await vi.advanceTimersByTimeAsync(1000)  // Fast-forward 1 second
  await vi.advanceTimersByTimeAsync(1000)
  
  await promise
  expect(fn).toHaveBeenCalledTimes(2)
})
```

---

## 5. Testing React Components

```typescript
// Install: @testing-library/react @testing-library/user-event
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { LoginForm } from '@/components/LoginForm'

// vitest.config.ts must have environment: 'jsdom'

describe('LoginForm', () => {
  it('submits with email and password', async () => {
    const onSubmit = vi.fn()
    const user = userEvent.setup()
    
    render(<LoginForm onSubmit={onSubmit} />)
    
    await user.type(screen.getByLabelText('Email'), 'user@test.com')
    await user.type(screen.getByLabelText('Password'), 'password123')
    await user.click(screen.getByRole('button', { name: 'Log in' }))
    
    expect(onSubmit).toHaveBeenCalledWith({
      email: 'user@test.com',
      password: 'password123',
    })
  })

  it('shows error for empty email', async () => {
    const user = userEvent.setup()
    render(<LoginForm onSubmit={vi.fn()} />)
    
    await user.click(screen.getByRole('button', { name: 'Log in' }))
    
    expect(screen.getByText('Email is required')).toBeInTheDocument()
  })
})
```

**Selector priority for Testing Library:**
1. `getByRole` — most accessible, most resilient
2. `getByLabelText` — for form inputs
3. `getByText` — for buttons and readable content
4. `getByTestId` — last resort (add `data-testid` to component)

---

## 6. Integration Tests with a Real Database

```typescript
// tests/integration/user.test.ts
import { describe, it, expect, beforeAll, afterAll, afterEach } from 'vitest'
import { db } from '@/lib/db'
import { createUser, getUserById } from '@/lib/user'

describe('user repository', () => {
  // Use a real test database — not mocks
  beforeAll(async () => {
    await db.connect()
  })

  afterAll(async () => {
    await db.disconnect()
  })

  afterEach(async () => {
    await db.user.deleteMany()  // Clean slate between tests
  })

  it('creates and retrieves a user', async () => {
    const created = await createUser({ email: 'test@test.com', name: 'Test' })
    const retrieved = await getUserById(created.id)
    
    expect(retrieved).toMatchObject({ email: 'test@test.com', name: 'Test' })
  })
})
```

**Why real DB in integration tests:** Mocked databases miss migration errors, constraint violations, and query bugs. Test against the real schema.

---

## 7. Snapshot Testing

Use sparingly — snapshots break on any UI change and create a false sense of coverage:

```typescript
// Good use: stable data transformations
it('formats user data correctly', () => {
  const result = formatUserForApi(rawUser)
  expect(result).toMatchSnapshot()
})

// Bad use: component markup that changes often
// Instead: test behavior, not structure
```

---

## 8. Coverage Report

```bash
npm run test:coverage
```

Generates:
- Terminal: text summary per file
- `coverage/index.html`: visual report showing covered/uncovered lines

**Coverage thresholds in CI:**
```typescript
// vitest.config.ts
coverage: {
  thresholds: { statements: 80, branches: 75 }
}
```

If coverage drops below the threshold, the CI run fails.
