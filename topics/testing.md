# Testing — AI-Assisted Development Strategy
> Reference for setting up and using a testing strategy designed to work with AI agents. Attach when configuring a test suite, writing tests with AI help, or auditing test coverage.

---

## 1. Why Tests Are Critical for AI-Assisted Work

Without tests, AI edits blindly. With tests:
- The AI gets automatic feedback — it can see when something breaks
- You can verify AI-generated code without reading every line
- Agentic tasks (multi-step, autonomous) become safe to run
- Refactoring becomes low-risk even in unfamiliar code

**Rule:** Never let AI make changes to code with no test coverage. If coverage is zero, write characterization tests first (see Section 5).

---

## 2. The Testing Pipeline

```
Test Plan → Unit Tests → Integration Tests → E2E Tests
```

Each layer has a different purpose:

| Layer | Tool (JS/TS) | What it tests | When to run |
|-------|-------------|---------------|-------------|
| Unit | Vitest / Jest | Individual functions, pure logic | Every commit |
| Integration | Vitest / Jest | Module interactions, DB queries | Every commit |
| E2E | Playwright / Cypress | Full user flows in a browser | On PR / before deploy |

**Do not skip the Test Plan.** A Test Plan written before implementation catches design problems before code does.

---

## 3. Test Plan: Write Before Code

Before implementing any feature, generate a Test Plan:

```
I am building [feature description].

Create a Test Plan with:
1. Unit test cases — list every function/logic branch to test
2. Integration test cases — list component interactions to verify
3. E2E test cases — list critical user flows to automate
4. Edge cases — inputs/states that are likely to break things
5. What NOT to test — third-party library internals, simple presentational components

Format: nested bullet list. No code yet.
```

Review and approve the Test Plan before writing code. The plan becomes the contract — implementation is done when all tests pass.

---

## 4. Unit Tests with Vitest

### Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import path from 'path'

export default defineConfig({
  test: {
    environment: 'node',        // or 'jsdom' for browser APIs
    globals: true,              // no need to import describe/it/expect
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      exclude: ['node_modules/', 'dist/', '*.config.*']
    }
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') }
  }
})
```

### CRITICAL: Use `vitest run` not `vitest` for AI agents

`vitest` (default) runs in watch mode — it waits for file changes forever. AI agents run in non-interactive terminals and will hang.

**Always configure package.json scripts:**
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

The AI uses `npm test` → runs `vitest run` → exits after tests complete. ✓

### Test file conventions

```
src/
  lib/
    auth.ts
    auth.test.ts       ← co-located with source
  components/
    Button.tsx
    Button.test.tsx
tests/
  integration/
    auth.integration.test.ts
```

### Example unit test structure

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { validateToken } from '@/lib/auth'

describe('validateToken', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('returns user id for valid token', () => {
    const result = validateToken('valid-jwt-token')
    expect(result.userId).toBe('user-123')
  })

  it('throws for expired token', () => {
    expect(() => validateToken('expired-token')).toThrow('Token expired')
  })

  it('throws for malformed token', () => {
    expect(() => validateToken('')).toThrow('Invalid token')
  })
})
```

### What to test vs what to skip

**Test:**
- Business logic and computation functions
- Data transformation and validation
- Error handling and edge cases
- Service layer functions (with mocked dependencies)
- Custom hooks with meaningful state logic

**Skip:**
- Third-party library functions (trust the library's own tests)
- Simple presentational components (no logic)
- Getters/setters with no logic
- Generated code (type definitions, schema types)

---

## 5. Characterization Tests (Legacy Code Safety Net)

When working with legacy code that has no tests, write characterization tests BEFORE making any changes.

**Characterization tests document what code *does*, not what it *should* do.**

```typescript
// tests/characterization/order-processing.char.test.ts
describe('OrderProcessor — characterization tests', () => {
  // These tests capture current behavior as of [date].
  // They are regression guards — do NOT change them unless 
  // you have intentionally changed the behavior they cover.

  it('applies 10% discount for orders over 100', () => {
    const result = processOrder({ total: 150, userId: 'u1' })
    // Current behavior: rounds down, not up
    expect(result.discountedTotal).toBe(135)
  })

  it('sends email even when payment fails', () => {
    // Bug or feature? Unknown. Capturing current behavior.
    const spy = vi.spyOn(emailService, 'send')
    processOrder({ total: 50, paymentMethod: 'invalid' })
    expect(spy).toHaveBeenCalled()
  })
})
```

**Rules for characterization tests:**
- Name the file `*.char.test.ts` or put in `tests/characterization/`
- Write tests that pass against the code AS IS — even if the behavior seems wrong
- Add a comment explaining what "correct" behavior might be, but don't change the test
- Run these after every AI change — if any fail, the AI changed behavior it shouldn't have

---

## 6. E2E Tests with Playwright

### Setup principles

**Dedicated test database:** Never run E2E tests against your development or production database.

```typescript
// playwright.config.ts
export default defineConfig({
  testDir: './tests/e2e',
  use: {
    baseURL: 'http://localhost:3000',
    // Set test-specific env
  },
  webServer: {
    command: 'npm run dev:test',   // starts app pointing at test DB
    url: 'http://localhost:3000',
    reuseExistingServer: false
  },
  globalSetup: './tests/e2e/setup.ts',
  globalTeardown: './tests/e2e/teardown.ts'
})
```

**Teardown after every E2E session.** Reset the test database to a known state:

```typescript
// tests/e2e/teardown.ts
export default async function teardown() {
  await resetTestDatabase()
  // or: await truncateAllTables()
}
```

### data-testid: place inside components, not in parents

```tsx
// ✓ CORRECT — testid on the element itself
export function LoginButton() {
  return <button data-testid="login-button">Log in</button>
}

// ✗ WRONG — testid on a wrapper in a parent component
// <div data-testid="login-button"><LoginButton /></div>
```

Placing `data-testid` inside the component keeps the test contract with the component, not with whoever renders it.

### Page Object Model (POM)

Never write raw selectors in test files. Use Page Objects:

```typescript
// tests/e2e/pages/LoginPage.ts
export class LoginPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/login')
  }

  async login(email: string, password: string) {
    await this.page.getByTestId('email-input').fill(email)
    await this.page.getByTestId('password-input').fill(password)
    await this.page.getByTestId('login-button').click()
  }

  async expectError(message: string) {
    await expect(this.page.getByTestId('error-message')).toHaveText(message)
  }
}

// tests/e2e/auth.spec.ts
test('successful login', async ({ page }) => {
  const loginPage = new LoginPage(page)
  await loginPage.goto()
  await loginPage.login('user@test.com', 'password123')
  await expect(page).toHaveURL('/dashboard')
})
```

**Benefits of POM:**
- Selector changes require update in one place
- Tests read like user stories
- Easy for AI to generate new tests from the pattern

### Selector priority (Playwright best practices)

Prefer in this order:
1. `getByRole` (semantic, accessible): `page.getByRole('button', { name: 'Submit' })`
2. `getByTestId` (explicit, stable): `page.getByTestId('submit-button')`
3. `getByText` (fragile with i18n): use only for unique, stable text
4. CSS selectors: last resort, avoid

---

## 7. Test Conventions to Document in CLAUDE.md

Add this section to CLAUDE.md:

```markdown
## Testing Conventions

### Running tests
- Unit: `npm test` (uses `vitest run` — exits after completion)
- E2E: `npm run test:e2e`
- Coverage: `npm run test:coverage`

### File locations
- Unit tests: co-located with source (`src/**/*.test.ts`)
- Integration tests: `tests/integration/`
- E2E tests: `tests/e2e/`
- Characterization tests: `tests/characterization/`

### Rules
- Use `vitest run` not `vitest` (watch mode hangs in CI and agents)
- All E2E tests use Page Object Model — see `tests/e2e/pages/`
- Place `data-testid` attributes inside the component, never in parent wrappers
- E2E tests run against a dedicated test database — never dev or prod
- Write characterization tests before modifying any untested legacy code
```

---

## 8. Video-Based Test Generation (Legacy & Unknown Codebases)

When you inherit a codebase with no tests and no documentation, generating tests from code is hard — you don't know what the intended behavior is. An alternative: record a user session and let a multimodal model extract test scenarios from the video.

### Optional approach: video-to-test-plan tools (example: @10xdevspl/test-planner)

```bash
# Requires: GEMINI_API_KEY in .env or environment
npx @10xdevspl/test-planner --video=user-session.mov --outDir=./e2e

# Optimize video size before sending (reduces token cost):
ffmpeg -i session.mov -r 15 -c:v libx264 -crf 23 session_15fps.mov
npx @10xdevspl/test-planner --video=session_15fps.mov --outDir=./e2e --optimize --fps=15
```

**What this class of tools does (not vendor-specific):**
1. Sends video frames to Gemini 2.5 Flash (1M context window supports up to ~60min of video)
2. Model identifies distinct user flows, UI elements, and interaction sequences
3. Outputs a `test-plan.md` (business test plan) + agent instructions for Playwright test implementation
4. You then run an AI agent (or implement manually) to create real test files from the plan

**Token cost estimate:** 60s @ 24fps ≈ 40k tokens. Optimizing to 15fps cuts cost by ~40%.

**Tips for better results:**
- Use a large cursor and enable "show click indicators" in your screen recorder
- Record one scenario per video for higher-quality plans
- Use test data, never production data

**When video-based generation helps:**
- Legacy code with zero tests — you don't know what "correct" looks like
- Rapid regression test creation before a refactor
- Onboarding to an unfamiliar system — the video teaches both you and the AI

If you prefer no dependency on a specific external tool, keep the same workflow:
1. Record a short user session video
2. Ask your model of choice to produce a `test-plan.md` from that video
3. Implement tests from the plan using Playwright/Cypress/Vitest
4. Review selectors/assertions manually before committing

**Alternative: Playwright Test Generator** (simpler but no AI analysis)
```bash
npx playwright codegen http://localhost:3000
```
Records your browser interactions and generates Playwright selectors in real time. Good for learning the selector API; requires manual cleanup of generated code.

---

## 9. Testing in CI

Every PR should automatically run:
```yaml
- name: Run unit tests
  run: npm test

- name: Run type check
  run: npm run typecheck

- name: Run linter
  run: npm run lint
```

E2E tests on PRs are optional (slow, costly) but required before merging to main:
```yaml
- name: Run E2E tests
  run: npm run test:e2e
  if: github.ref == 'refs/heads/main'
```
