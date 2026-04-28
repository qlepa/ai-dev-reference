# Playwright — AI Development Reference
> Attach when configuring Playwright or writing E2E tests with AI assistance. Covers setup, Page Object Model, test isolation, and CI integration. If no base file (`fresh.md`, `new.md`, or `legacy.md`) is attached alongside this one, run a standalone audit: treat each section's guidelines as a checklist, assess the current state (✅ pass / ⚠️ partial / ❌ missing), and save a report as `playwright-audit-YYYY-MM-DD.md`.

---

## 1. Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,      // Fail if .only in CI
  retries: process.env.CI ? 2 : 0,   // Retry flaky tests in CI
  workers: process.env.CI ? 1 : undefined,
  
  reporter: [
    ['html'],
    ['line'],  // Progress in terminal
  ],

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',          // Record trace on failure
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    // Add more browsers as needed:
    // { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    // { name: 'mobile', use: { ...devices['Pixel 5'] } },
  ],

  // Start the app automatically before tests
  webServer: {
    command: 'npm run dev:test',       // Start app pointed at test DB
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120 * 1000,
  },

  globalSetup: './tests/e2e/global-setup.ts',
  globalTeardown: './tests/e2e/global-teardown.ts',
})
```

---

## 2. Dedicated Test Database (Required)

**Never run E2E tests against your development or production database.**

```bash
# .env.test (not committed — add to .gitignore)
DATABASE_URL=postgres://localhost:5432/myapp_test
```

```typescript
// package.json scripts
{
  "scripts": {
    "dev": "NODE_ENV=development next dev",
    "dev:test": "NODE_ENV=test next dev",    // Used by Playwright webServer
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"   // Visual debugger
  }
}
```

---

## 3. Global Setup and Teardown

```typescript
// tests/e2e/global-setup.ts
import { chromium } from '@playwright/test'

export default async function globalSetup() {
  // Seed test database with known state
  await seedTestDatabase()
  
  // Optional: pre-authenticate and save session
  const browser = await chromium.launch()
  const page = await browser.newPage()
  await page.goto('http://localhost:3000/login')
  await page.fill('[data-testid="email"]', 'admin@test.com')
  await page.fill('[data-testid="password"]', 'testpassword')
  await page.click('[data-testid="login-button"]')
  await page.context().storageState({ path: 'tests/e2e/.auth/admin.json' })
  await browser.close()
}
```

```typescript
// tests/e2e/global-teardown.ts
export default async function globalTeardown() {
  await truncateTestDatabase()
  // Clean up any files created during tests
}
```

---

## 4. Page Object Model (Required Pattern)

Never write raw selectors directly in test files. All selectors and interactions live in Page Objects.

```typescript
// tests/e2e/pages/LoginPage.ts
import { type Page, type Locator, expect } from '@playwright/test'

export class LoginPage {
  private readonly emailInput: Locator
  private readonly passwordInput: Locator
  private readonly loginButton: Locator
  private readonly errorMessage: Locator

  constructor(private page: Page) {
    this.emailInput = page.getByTestId('email-input')
    this.passwordInput = page.getByTestId('password-input')
    this.loginButton = page.getByTestId('login-button')
    this.errorMessage = page.getByTestId('error-message')
  }

  async goto() {
    await this.page.goto('/login')
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email)
    await this.passwordInput.fill(password)
    await this.loginButton.click()
  }

  async expectError(message: string) {
    await expect(this.errorMessage).toBeVisible()
    await expect(this.errorMessage).toHaveText(message)
  }

  async expectRedirectTo(path: string) {
    await expect(this.page).toHaveURL(path)
  }
}
```

```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test'
import { LoginPage } from './pages/LoginPage'
import { DashboardPage } from './pages/DashboardPage'

test.describe('Authentication', () => {
  test('user can log in with valid credentials', async ({ page }) => {
    const loginPage = new LoginPage(page)
    
    await loginPage.goto()
    await loginPage.login('user@test.com', 'password123')
    await loginPage.expectRedirectTo('/dashboard')
  })

  test('shows error for wrong password', async ({ page }) => {
    const loginPage = new LoginPage(page)
    
    await loginPage.goto()
    await loginPage.login('user@test.com', 'wrongpassword')
    await loginPage.expectError('Invalid email or password')
  })
})
```

**Benefits of POM:**
- Selector changes require update in one place only
- Tests read like user stories — easy for AI to extend
- Locators are typed — TypeScript catches errors

---

## 5. data-testid: Place Inside Components

**Rule: `data-testid` attributes belong inside the component definition, not in parent wrappers.**

```tsx
// ✓ CORRECT — testid is part of the component
export function SubmitButton({ children }: { children: React.ReactNode }) {
  return (
    <button data-testid="submit-button" type="submit">
      {children}
    </button>
  )
}

// ✗ WRONG — testid added by the parent, not the component
function Form() {
  return (
    <div data-testid="submit-button">
      <SubmitButton>Submit</SubmitButton>
    </div>
  )
}
```

Why: The component owns its own testability contract. When the component moves to a different parent, the test still works.

---

## 6. Selector Priority

Use in this order — most resilient first:

```typescript
// 1. Role (accessible, stable, preferred)
page.getByRole('button', { name: 'Submit' })
page.getByRole('textbox', { name: 'Email' })
page.getByRole('heading', { name: 'Dashboard' })

// 2. Label (for form inputs)
page.getByLabel('Email address')

// 3. Test ID (explicit, stable, acceptable)
page.getByTestId('login-button')

// 4. Text (fragile with i18n, last resort)
page.getByText('Submit')

// ✗ Never use CSS selectors in tests
page.locator('.submit-btn')          // breaks on refactor
page.locator('#form > button:first-child')  // implementation detail
```

---

## 7. Test Isolation

Each test should be fully independent — no test should rely on state from a previous test.

```typescript
test.describe('User management', () => {
  // Run before each test in this describe block
  test.beforeEach(async ({ page }) => {
    // Reset to known state
    await resetUserTable()
    await seedTestUser({ email: 'test@test.com' })
    
    // Navigate to starting point
    await page.goto('/admin/users')
  })

  test('displays user list', async ({ page }) => {
    await expect(page.getByTestId('user-row')).toHaveCount(1)
  })

  test('deletes a user', async ({ page }) => {
    await page.getByTestId('delete-button').click()
    await expect(page.getByTestId('user-row')).toHaveCount(0)
  })
  // Both tests start from exactly the same state — they can run in any order
})
```

---

## 8. Authentication in Tests

### Option A: Reuse saved auth state (fast)

```typescript
// playwright.config.ts
projects: [
  {
    name: 'authenticated',
    use: {
      storageState: 'tests/e2e/.auth/user.json',  // Pre-saved login session
    },
    testMatch: /.*\.auth\.spec\.ts/,
  },
  {
    name: 'public',
    testMatch: /.*\.public\.spec\.ts/,
  },
]
```

### Option B: Login fixture (flexible)

```typescript
// tests/e2e/fixtures.ts
import { test as base } from '@playwright/test'
import { LoginPage } from './pages/LoginPage'

type Fixtures = { authenticatedPage: Page }

export const test = base.extend<Fixtures>({
  authenticatedPage: async ({ page }, use) => {
    const loginPage = new LoginPage(page)
    await loginPage.goto()
    await loginPage.login('test@test.com', 'testpassword')
    await use(page)
  },
})
```

---

## 9. Running Tests

```bash
# Run all tests (headless)
npm run test:e2e

# Run with visual UI (debug mode)
npm run test:e2e:ui

# Run specific file
npx playwright test tests/e2e/auth.spec.ts

# Run with headed browser (watch it run)
npx playwright test --headed

# Debug a specific test
npx playwright test --debug tests/e2e/auth.spec.ts

# Show last HTML report
npx playwright show-report
```

---

## 10. CI Integration

```yaml
# .github/workflows/e2e.yml (or in your main deploy workflow)
- name: Install Playwright browsers
  run: npx playwright install --with-deps chromium
  # --with-deps installs system dependencies (OS-level, needed on fresh runners)

- name: Run E2E tests
  run: npm run test:e2e
  env:
    DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
    CI: true

- name: Upload test results
  uses: actions/upload-artifact@v4
  if: failure()    # Only upload on failure — save storage
  with:
    name: playwright-report
    path: playwright-report/
    retention-days: 7
```
