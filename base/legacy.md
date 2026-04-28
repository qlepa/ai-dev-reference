# Legacy Project — Onboarding & Modernization
> You are onboarding to a legacy or long-running codebase and preparing it for safe, AI-assisted modernization. **Do not change production logic yet.** Your goal in this session is to understand the system and produce a clear map for future work.
>
> **Do not create or edit project files (CLAUDE.md, test files, config files) during this session.** Flag them in the Prioritized Action List. The only files you may create are analysis/report outputs (ARCHITECTURE.md, dead-code-report, modernization-plan, the onboarding report). After presenting the report, wait for the user to explicitly ask you to implement specific items.

---

## Companion Files

Before starting: read the **Tech stack** field in Context. Cross-reference it against this list of known companion files:

| Technology | File to attach |
|------------|----------------|
| Next.js | `nextjs.md` + `react.md` |
| React (standalone) | `react.md` |
| TypeScript | `typescript.md` |
| Tailwind CSS | `tailwind.md` |
| Vitest | `vitest.md` |
| Playwright | `playwright.md` |
| Supabase | `supabase.md` |
| Astro | `astro.md` |
| MCP | `mcp.md` |
| Next.js security hardening | `nextjs-security.md` |

**Topics files** — scan **Known problem areas** and **What triggered this** fields:

| If context mentions… | Suggest |
|----------------------|---------|
| *(always for legacy)* | `modernization.md` ✱ |
| test / no tests / coverage / regression | `testing.md` |
| CI / CD / deploy / pipeline / GitHub Actions | `cicd.md` |
| AI instructions / CLAUDE.md / rules | `ai-instructions.md` |
| prompt / prompting | `prompting.md` |
| context / long session / token | `context.md` |
| model / cost / expensive / budget | `model-selection.md` |
| AI-ready / readiness / checklist | `ai-project-checklist.md` |

✱ Recommended for all legacy projects — ask if not attached.

For each matched file: check if it is present in this conversation. If a matching file is missing, ask the user before proceeding:
> "Based on your context, `[filename]` looks relevant. Should I wait for it before starting, or continue without it?"

**Important:** Files must be attached to this conversation — copying them to the same folder is not enough.

---

## Context *(fill this in before handing to AI)*

```
Project name        : 
Short description   : 
Tech stack          : 
Approximate age     : 
Known problem areas : (things you already know are broken, messy, or risky)
What triggered this : (e.g., new feature needed, team handover, compliance audit)
Primary AI tool     : [ ] Claude Code  [ ] Cursor  [ ] Copilot  [ ] Other: ___
Repo location       : 
```

---

## Phase 1: Orientation

**Goal:** Build a mental model of the system before touching anything.

### 1.1 Repository Overview

- [ ] List all top-level folders and their apparent purpose
- [ ] Identify the entry point(s) of the application (main files, server startup, route definitions)
- [ ] Identify all configuration files (`.env*`, `config/`, `settings.*`, CI/CD files)
- [ ] Identify the package manager and dependency manifest (`package.json`, `requirements.txt`, `go.mod`, `Gemfile`, etc.)
- [ ] Check if any existing documentation exists: `README`, `CONTRIBUTING`, `docs/`, `wiki/`, `.ai/`

### 1.2 Git History Analysis

Run these commands and include the output in your report:

**Most frequently changed files (last 90 days):**
```bash
git log --since="90 days ago" --pretty=format:"" --name-only --no-merges | \
  grep '.' | sort | uniq -c | sort -nr | head -15 | \
  awk '{count=$1; $1=""; sub(/^[ \t]+/, ""); print $0 ": " count " changes"}'
```

**Most frequently changed modules/folders:**
```bash
git log --since="90 days ago" --pretty=format:"" --name-only --no-merges | \
  grep '.' | awk -F/ -v OFS=/ 'NF>1{$NF="";print $0}NF<=1{print "."}' | \
  sed 's|/*$||' | sort | uniq -c | sort -nr | head -10 | \
  awk '{count=$1; $1=""; sub(/^[ \t]+/, ""); print $0 ": " count " changes"}'
```

**Top contributors (last 90 days):**
```bash
git log --since="90 days ago" --pretty=format:"%an" --no-merges | \
  sort | uniq -c | sort -nr | head -5
```

**Most recent 10 commits (to understand current direction):**
```bash
git --no-pager log --stat -n 10
```

Interpret the results:
- Files changed most = hot spots (highest risk for AI edits, highest value for tests)
- Folders changed most = active development areas
- Contributors = domain knowledge holders

### 1.3 Generate ARCHITECTURE.md

Create `ARCHITECTURE.md` in the repo root (or `.ai/architecture.md`) containing:

```markdown
# Architecture — [Project Name]

## Overview
[2–3 paragraphs: what the system does, who uses it, scale/load if known]

## Tech Stack
[List: language/version, framework/version, database, key libraries]

## Folder Structure
[Top-level map with one-line description per folder]

## Entry Points
[How the app starts, main files, key routes or API surface]

## Data Flow
[How a typical request flows through the system: client → ... → database]

## Key Components
[Top 5–10 most important files/modules with brief descriptions]

## Hot Spots (High-change areas)
[List from git analysis — these are highest risk for changes]

## Known Problem Areas
[From context provided above + any issues observed during analysis]
```

---

## Phase 2: Safety Net

**Goal:** Establish guardrails before any code changes. Changes without tests in a legacy codebase are dangerous.

### 2.1 Test Inventory

- [ ] Find all existing tests:
  - Look for: `test/`, `tests/`, `spec/`, `__tests__/`, `*.test.*`, `*.spec.*`
  - Run the test suite: record pass/fail count and any framework errors
- [ ] Assess coverage (if coverage tooling exists):
  - Run: `npm run test:coverage` / `pytest --cov` / `go test -cover ./...`
  - Identify the 3 most critical paths with zero coverage
- [ ] If NO tests exist at all: note this prominently — it is the highest risk factor

### 2.2 Identify Critical Paths (Must Protect)

List the 3–5 most critical user flows or system functions that MUST NOT break during modernization. These are candidates for characterization tests.

Typical critical paths:
- Authentication / session management
- Core data mutation (create, update, delete on primary entities)
- Payment or billing logic
- Public API endpoints
- Data export or report generation

### 2.3 Characterization Tests (if tests are sparse/absent)

For each critical path identified above, add it to the Prioritized Action List with:
- Which path needs a characterization test
- What behavior it should capture (current behavior, not ideal)
- Suggested test name and location (`tests/characterization/`)

Do not write the tests during this session — flag them and wait for the user to ask.

> Characterization tests document what code *does*, not what it *should* do. They are a safety net, not a specification.

---

## Phase 3: AI Instruction Files (Legacy-Specific)

**Goal:** CLAUDE.md for a legacy project must document not just conventions, but also *what NOT to touch* and *why existing patterns exist*.

### 3.1 Assess CLAUDE.md

Check if `CLAUDE.md` exists in the repo root.
- If it exists: review it and note any gaps in the report (missing legacy-specific sections, outdated info).
- If it is missing: flag as ❌ in the Prioritized Action List. Do not create it now — wait for the user to ask.

For legacy projects, CLAUDE.md must include these sections in addition to the standard content:

```markdown
# CLAUDE.md — [Project Name]

## Project Overview
[Brief description]

## Tech Stack
[List with versions]

## Architecture Summary
[Link to ARCHITECTURE.md or paste key sections]

## Working in This Codebase

### Do NOT modify without explicit instruction:
- [List files/folders that are off-limits — e.g., payment processing, auth modules]
- [Generated files, vendored code, migration files]

### Legacy patterns to preserve (intentional, do not "fix"):
- [e.g., "Double-write pattern in orders/ is intentional for data sync, do not refactor"]
- [e.g., "Global state in config.js is legacy — do not modularize without a plan"]

### Known footguns:
- [e.g., "userId in this context is an int, not a UUID — mixing them causes silent data loss"]
- [e.g., "The cache is write-through — updating the DB directly bypasses cache invalidation"]

### Modernization rules:
- Always create a branch before changes
- Commit after every working stage
- Run the characterization tests after every change
- Do not refactor and add features in the same commit

## Conventions
[Observed naming, file structure, import style]

## Running the Project
[How to start locally, required env vars, seed data]

## Running Tests
[Test command(s), how to run specific tests]
```

---

## Phase 4: Problem Mapping

**Goal:** Know what's broken before deciding what to fix. Never modernize blindly.

### 4.1 Dependency Audit

- [ ] List all direct dependencies with their versions
- [ ] Run vulnerability scan:
  - JS: `npm audit`
  - Python: `pip-audit`
  - Ruby: `bundle audit`
  - Go: `govulncheck ./...`
- [ ] Flag: HIGH/CRITICAL vulnerabilities, abandoned packages, packages 2+ major versions behind
- [ ] Do NOT automatically upgrade — record findings for human decision

### 4.2 Dead Code Detection

- [ ] Search for unused exports / unreferenced files:
  - JS/TS: `npx ts-prune` or `npx unimported`
  - Python: `vulture`
  - Or ask AI to identify files with no imports pointing to them
- [ ] Search for large commented-out code blocks:
  ```bash
  git grep -n "^[[:space:]]*//" -- "*.ts" | wc -l   # TS example
  ```
- [ ] Search for `TODO`, `FIXME`, `HACK`, `XXX` markers:
  ```bash
  git grep -rn "TODO\|FIXME\|HACK\|XXX"
  ```
  Count and categorize — don't fix, just map.
- [ ] Create `dead-code-report-YYYY-MM-DD.md` (use today's date) in `.ai/` or `ai-audit/`:
  - Unused files list
  - Top 10 largest commented blocks
  - TODO/FIXME count by category

### 4.3 Security Baseline

- [ ] Check for secrets in tracked files:
  ```bash
  git grep -rn "sk-\|api_key\s*=\|password\s*=\|secret\s*=\|token\s*=" -- ':!*.example' ':!*.md' ':!*.lock'
  ```
- [ ] Check git history for accidentally committed secrets:
  ```bash
  git log --all --full-history --oneline -- "**/.env" "*.env"
  ```
- [ ] Check authentication patterns:
  - Is auth middleware applied consistently to protected routes?
  - Is there input validation at API boundaries (not just client-side)?
  - Are SQL queries parameterized (no string interpolation into queries)?
- [ ] Report findings — do NOT auto-fix security issues (they require human review)

### 4.4 CI/CD State

- [ ] Does a CI/CD pipeline exist? (`.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`, etc.)
- [ ] Does it run tests? Does it enforce linting?
- [ ] Is there an automated deployment process or is deployment manual?
- [ ] Document the current state — do NOT change the pipeline yet

---

## Phase 5: Modernization Roadmap

**Goal:** Turn the findings from phases 1–4 into a prioritized, safe action plan.

### 5.1 Classify Each Issue

For every significant finding, classify it as:

| Category | Description | Approach |
|----------|-------------|----------|
| **AI-rewrite** | Well-understood logic, good test coverage or can get it, clear inputs/outputs | AI can do this with supervision |
| **Codemod** | Mechanical transformation (rename, restructure, upgrade syntax) | Automated tool + AI verification |
| **Manual-only** | Security-critical, complex business logic, high risk, unclear behavior | Human-only, AI advises only |

### 5.2 Create modernization-plan.md

Create `modernization-plan-YYYY-MM-DD.md` (use today's date) in `.ai/` or `ai-audit/`:

```markdown
# Modernization Plan — [Project Name] — [Date]

## Immediate (before any feature work):
1. [Item — Category: AI-rewrite/Codemod/Manual — Risk: High/Med/Low]
2. ...

## Short-term (next sprint/iteration):
1. ...

## Long-term (backlog):
1. ...

## Do NOT touch (at least not yet):
- [Area] — Reason: [why it's off-limits for now]

## First 3 safe changes recommended:
1. [Specific, small, reversible change with clear benefit]
2. ...
3. ...
```

---

## Deliverables

At the end of this session, produce `legacy-onboarding-report-YYYY-MM-DD.md` (use today's date) in the `.ai/` or `ai-audit/` folder — whichever exists in this project. If a previous report already exists, do not overwrite it — create a new file with today's date.

```markdown
# Legacy Onboarding Report — [Project Name] — YYYY-MM-DD

## Architecture Summary
[Condensed version — 1 paragraph + key files list]

## Hot Spots (Risk Map)
| File/Module | Change Rate | Risk Level | Notes |
|-------------|-------------|------------|-------|
| ...         | X changes   | High/Med/Low | |

## Test Coverage State
- Total tests: N
- Critical paths covered: N/M
- Critical paths needing characterization tests: [list]

## Security Findings
[Brief list — severity, location, nature]

## Recommended First 3 Safe Changes
1. [Specific action, why safe, expected benefit]
2. ...
3. ...

## What NOT to Touch (and why)
- [Module] — [Reason]

## Prioritized Action List
1. [Most critical — e.g., create CLAUDE.md, write characterization tests for X]
2. ...
3. ...

## Open Questions for the Team
- [Things that need human knowledge to resolve]

## Nice to Have
- [ ] `.ai/prd.md` — original or current product requirements with business context
- [ ] `.ai/tech-stack.md` — technology decisions and rationale (especially valuable for legacy tech choices that may seem odd without context)
- [ ] `.ai/decisions/` or `adr/` — Architecture Decision Records explaining historical choices ("why is auth done this way?")
- [ ] `.ai/runbook.md` — deployment steps, known failure modes, on-call procedures
- [ ] `.ai/glossary.md` — domain terminology and business concepts (critical in legacy systems with company-specific jargon)
```
