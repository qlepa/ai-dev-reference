# CI/CD — GitHub Actions for AI-Assisted Projects
> Reference for setting up CI/CD pipelines that protect code quality and enable safe AI-assisted deployment. Attach when configuring GitHub Actions or auditing an existing pipeline.

---

## 1. Core Concepts

### Hierarchy
```
Workflow (.github/workflows/*.yml)
  └── Job (runs on a runner, e.g., ubuntu-latest)
        └── Step (single command or action)
              └── Action (reusable step from marketplace)
```

### Triggers
```yaml
on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]
```

One workflow file can respond to multiple events. The pattern: run fast checks on every PR, run full deploy only on push to main.

---

## 2. PR Pipeline (Quality Gate)

Run on every pull request. Goal: catch problems before they reach main.

```yaml
# .github/workflows/pr.yml
name: PR Quality Check

on:
  pull_request:
    branches: [main]

jobs:
  quality-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run typecheck

      - name: Lint
        run: npm run lint

      - name: Unit tests
        run: npm test
```

**Rules for the PR pipeline:**
- Must run in under 5 minutes — developers wait for this
- All jobs should fail fast: if type check fails, skip tests (save runner minutes)
- Use `npm ci` not `npm install` — reproducible, faster, respects lockfile
- Cache dependencies with `cache: 'npm'`

### Parallel jobs for speed

Run independent checks simultaneously:

```yaml
jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run typecheck

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm test
```

Three parallel jobs: total time = slowest job (not sum of all).

---

## 3. Main Branch Pipeline (Deploy)

Run on push to main. Goal: deploy only after all quality gates pass.

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  quality-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run typecheck
      - run: npm run lint
      - run: npm test

  e2e-tests:
    needs: quality-checks      # Wait for quality checks to pass first
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}

  deploy:
    needs: [quality-checks, e2e-tests]   # Deploy only after both pass
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: # your deploy command
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

**Key patterns:**
- `needs: quality-checks` — job dependency, creates a sequential pipeline
- E2E tests run only on main push (too slow for every PR, but required before deploy)
- Deploy only runs if both quality-checks and e2e-tests succeed

---

## 4. Secrets Management

**Never hardcode secrets in workflow files.** Use GitHub Secrets.

### Setting up secrets
1. GitHub repo → Settings → Secrets and variables → Actions
2. Add: `DATABASE_URL`, `DEPLOY_TOKEN`, `API_KEY`, etc.

### Accessing in workflow
```yaml
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_KEY: ${{ secrets.API_KEY }}
```

**Available at:** job level, step level, or individual `run` command level.

### Secret categories to define
- **Deployment:** hosting provider tokens (Vercel, Railway, Fly.io, etc.)
- **Database:** production DB URL, test DB URL (separate!)
- **External APIs:** third-party service keys
- **Notifications:** Slack webhook, email SMTP

### Environment-specific secrets

For staging vs production:
1. GitHub repo → Settings → Environments → Create environment
2. Add environment-specific secrets
3. Reference in workflow:

```yaml
deploy-staging:
  environment: staging
  steps:
    - run: deploy
      env:
        API_URL: ${{ secrets.API_URL }}  # staging value
```

---

## 5. Common Workflow Patterns

### Database migrations on deploy

```yaml
deploy:
  needs: [quality-checks, e2e-tests]
  steps:
    - name: Run migrations
      run: npm run db:migrate
      env:
        DATABASE_URL: ${{ secrets.DATABASE_URL }}
    - name: Deploy application
      run: npm run deploy
```

**Rule:** Run migrations before deploying the new application code. Never after.

### Caching for faster runs

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'          # Caches node_modules based on package-lock.json hash
```

For Playwright browsers (large download):
```yaml
- name: Cache Playwright browsers
  uses: actions/cache@v4
  with:
    path: ~/.cache/ms-playwright
    key: playwright-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
```

### Branch protection rules

In GitHub repo → Settings → Branches → Add rule for `main`:
- ✓ Require pull request before merging
- ✓ Require status checks to pass (add: `quality-checks`, `test`)
- ✓ Require branches to be up to date before merging
- ✓ Do not allow bypassing the above settings

This prevents anyone (including yourself) from pushing broken code to main.

---

## 6. Feature Flags — Separating Deployment from Release

**Deployment** = technical act of putting code on the server.
**Release** = business decision to expose a feature to users.

These must be independent. Conflating them forces long-lived branches (merge hell) and prevents safe continuous deployment.

### Minimal feature flag module

```typescript
// src/lib/feature-flags.ts
const ENV = process.env.PUBLIC_ENV_NAME ?? null;  // null → everything off

const flags: Record<string, Record<string, boolean>> = {
  local:       { newCheckout: true,  aiSearch: true  },
  integration: { newCheckout: true,  aiSearch: false },
  prod:        { newCheckout: false, aiSearch: false },
};

export function isEnabled(flag: string): boolean {
  if (!ENV || !flags[ENV]) return false;
  return flags[ENV][flag] ?? false;
}
```

```
// .env.example
PUBLIC_ENV_NAME=local     # local | integration | prod
```

**Key rule:** Default to `false`, not `local`. A misconfigured environment should give users nothing, not everything.

**Usage in components:**
```tsx
{isEnabled('newCheckout') && <NewCheckoutFlow />}
```

**When you need this:** Any team larger than one person, any feature that takes more than one day, any feature that needs a coordinated marketing launch.

**When you don't need this:** Small solo projects where you can deploy and release simultaneously without coordination.

### Async agents and feature flags

Async agents (Devin, Jules, Codex) should work on branches gated behind flags. This lets you merge their PRs continuously without exposing unfinished work.

---

## 7. Composite Actions — Reusable AI Steps

Composite Actions extract repeated GHA steps into an independent repo, then inject them into any workflow. Ideal for AI-powered steps you want to share across projects.

**Structure of a composite action repo:**
```
my-ai-action/
├── action.yml          # declares: name, inputs, steps
└── dist/
    └── index.js        # bundled with rolldown/esbuild (includes all deps)
```

```yaml
# action.yml
name: AI Code Reviewer
description: Reviews a PR diff with Gemini Flash

inputs:
  GOOGLE_API_KEY:
    description: "Google AI Studio key"
    required: true

runs:
  using: composite
  steps:
    - name: Run reviewer
      run: node ${{ github.action_path }}/dist/index.js
      shell: bash
      env:
        GOOGLE_API_KEY: ${{ inputs.GOOGLE_API_KEY }}
```

**Consumer workflow:**
```yaml
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with: { node-version: '22' }
  - uses: your-org/my-ai-action@main
    with:
      GOOGLE_API_KEY: ${{ secrets.GOOGLE_API_KEY }}
```

**Bundle your dependencies before committing** — the consumer repo won't run `npm install` for your action:
```bash
npx rolldown src/index.js --file dist/index.js
git add dist/index.js && git commit -m "chore: bundle action"
```

---

## 7b. GITHUB_TOKEN vs Personal Access Token

| Feature | `GITHUB_TOKEN` | Personal Access Token (PAT) |
|---------|---------------|----------------------------|
| Created | Automatically per-workflow-run | Manually by a user |
| Scope | This repo only | Any repo/org you choose |
| Expiry | Workflow duration | 30 / 90 days (you set) |
| Cross-repo actions | No | Yes |
| Use case | Standard CI steps, PR comments | Creating PRs in other repos, managing org-level resources |

```yaml
# Use GITHUB_TOKEN (built-in, no setup)
- uses: actions/github-script@v7
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    script: |
      github.rest.issues.create({ ... })

# Use PAT (for cross-repo or elevated permissions)
- uses: actions/github-script@v7
  with:
    github-token: ${{ secrets.MY_PAT }}
```

**Enable write access for GITHUB_TOKEN** (required for PR creation, file commits):
Repo → Settings → Actions → General → Workflow permissions → Read and write permissions.

---

## 8. Claude Code CLI in CI/CD (Headless Mode)

Claude Code can run non-interactively in pipelines, pre-commit hooks, and scripts using the `-p` flag.

### Basic headless invocation

```bash
# Run a one-shot prompt and exit
claude -p "Review the staged changes for security issues. Report findings only, do not edit files."

# Structured JSON output for programmatic parsing
claude -p "Analyze src/auth.ts and return issues as JSON" --output-format stream-json
```

### Critical flags for CI/CD

```bash
claude -p "Fix the failing tests" \
  --max-turns 10 \                  # Hard limit on agent iterations — prevents runaway cost
  --permission-mode auto \           # Auto-approve safe actions (read, non-destructive writes)
  --output-format stream-json        # Machine-readable output for parsing
```

**`--max-turns`** is the most important cost-control flag. Without it, an agent stuck in a loop will keep retrying until it hits your account limit. Set it to 2–3x what you expect the task to need.

### GitHub Actions cost controls

```yaml
# .github/workflows/ai-review.yml
jobs:
  ai-review:
    runs-on: ubuntu-latest
    timeout-minutes: 10            # Workflow-level timeout — kills hung agents
    concurrency:
      group: ai-review-${{ github.ref }}
      cancel-in-progress: true     # Cancel previous run if new PR push arrives

    steps:
      - uses: anthropics/claude-code-action@beta
        with:
          claude_args: "--max-turns 5"   # Per-invocation turn limit
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### When to use headless vs Claude Code Action

| Use case | Approach |
|----------|----------|
| Automated PR code review | `claude-code-action` — has GitHub integration built in |
| Pre-commit hook | `claude -p "..."` with `--max-turns 3` |
| Changelog generation (cron) | `claude -p "..." --output-format stream-json` piped to a script |
| JSDoc sync (nightly) | `claude -p "..." --max-turns 20 --permission-mode auto` |
| One-shot analysis script | `claude -p "..."` — simplest form |

**Rule:** Always set `--max-turns` and a `timeout-minutes` in any automated context. An agent with no limits and a task it cannot complete will run until you hit your API spending cap.

---

## 9. Pipeline Design Principles

| Principle | Why |
|-----------|-----|
| Fast feedback first | Fail cheap (lint, types) before running expensive tests |
| Parallel where possible | Independent jobs run simultaneously — cuts total time |
| Separate PR and deploy pipelines | PRs need speed; deploys need completeness |
| Secrets only from GitHub Secrets | Never in workflow YAML, never in repo files |
| Pin action versions | `actions/checkout@v4` not `@latest` — prevents supply chain attacks |
| Test against a real test DB | Mocked DBs miss migration and constraint errors |

---

## 10. CI/CD Section in CLAUDE.md

Add this to CLAUDE.md:

```markdown
## CI/CD

### Pipelines
- PR checks: `.github/workflows/pr.yml` — typecheck + lint + unit tests (parallel, ~2 min)
- Deploy: `.github/workflows/deploy.yml` — quality checks → E2E → deploy (sequential)

### Secrets
All secrets stored in GitHub repo Settings → Secrets. Required:
- DATABASE_URL — production database
- TEST_DATABASE_URL — dedicated E2E test database
- DEPLOY_TOKEN — deployment provider token

### Rules for AI
- Do NOT modify workflow files without explicit instruction
- Never add secrets to workflow YAML — use ${{ secrets.NAME }} references only
- When adding a new npm script, ensure it exits (no watch mode) or it will hang CI
```
