# Fresh Project — AI-Readiness Audit
> You are auditing an existing, recently-started project to assess its readiness for AI-assisted development. For each item: check the current state, assign a status (✅ pass / ⚠️ partial / ❌ missing), and record the gap with a recommended fix.
>
> **Do not create, edit, or delete any files during the audit.** Your only output is the audit report. After presenting the report, wait for the user to explicitly ask you to implement specific items before making any changes.

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

**Topics files** (attach when relevant):

| Topic | File to attach |
|-------|----------------|
| Test strategy | `testing.md` |
| CI/CD pipeline | `cicd.md` |
| Prompting patterns | `prompting.md` |
| Context management | `context.md` |
| AI instruction files | `ai-instructions.md` |
| Codebase modernization | `modernization.md` |
| Model selection | `model-selection.md` |
| AI-readiness checklist | `ai-project-checklist.md` |

For each matched file: check if it is present in this conversation. If a matching file is missing, ask the user before proceeding:
> "I see your stack includes [technology]. Should I wait for `[filename]` before starting, or continue without it?"

**Important:** Files must be attached to this conversation — copying them to the same folder is not enough.

---

## Context *(fill this in before handing to AI)*

```
Project name     : 
Short description: 
Tech stack       : 
Approximate age  : 
Primary AI tool  : [ ] Claude Code  [ ] Cursor  [ ] Copilot  [ ] Other: ___
Repo location    : 
Known gaps       : (anything you already know is missing)
```

---

## 1. AI Instruction Files

**Why this matters:** Without these files, AI starts every session with zero project context. You'll repeat yourself in every conversation.

- [ ] `CLAUDE.md` exists and is accurate
  - Contains: project overview, stack, folder structure, naming conventions, areas to avoid
  - If missing or outdated: flag as ❌ and add to the Prioritized Action List
- [ ] `.cursorrules` or `.cursor/rules/*.mdc` exists (if Cursor is used)
  - Suggested split: `shared.mdc`, `frontend.mdc`, `backend.mdc`, `testing.mdc`
  - If missing: flag as ❌ and add to the Prioritized Action List
- [ ] `.github/copilot-instructions.md` exists (if Copilot is used)
- [ ] All instruction files are under 500 lines each

**If files are missing:** Flag as ❌ and add to the Prioritized Action List with a suggested content outline:
- One-paragraph project description
- Tech stack list
- Folder map with one-line descriptions
- Observed naming and coding conventions
- Any patterns you notice that should be preserved

Do not generate these files during the audit — wait for the user to ask.

---

## 2. Project Comprehension

**Why this matters:** AI should understand the project before making changes. Generate this understanding once and store it permanently.

- [ ] Read `README.md` — does it accurately describe the project? Note discrepancies in the report.
- [ ] Identify entry points (main files, route definitions, API handlers)
- [ ] Map top-level folder structure and understand what each folder does
- [ ] Identify the 3–5 most important files (high-change rate or core business logic)
- [ ] Does `CLAUDE.md` contain an **Architecture Summary** section?
  - If not: flag as ❌ and add to the Prioritized Action List
- [ ] Check if `.ai/` folder exists with planning documents (prd.md, tech-stack.md, etc.)
  - If not: flag as ❌ and add to the Prioritized Action List

---

## 3. Code Quality Tooling

**Why this matters:** Linters give AI an automatic feedback loop — it generates code, sees errors, self-corrects. Without linters, AI silently produces broken code.

- [ ] Linter configured?
  - Check for config files: `.eslintrc*`, `pylintrc`, `.golangci.yml`, etc.
  - Run linter: record the error count (don't fix all — just know the state)
  - **Status: ✅ / ⚠️ (configured but errors exist) / ❌ (missing)**
- [ ] Formatter configured?
  - Check for: `.prettierrc`, `pyproject.toml [tool.black]`, `.editorconfig`, etc.
  - **Status: ✅ / ⚠️ / ❌**
- [ ] Type checker clean?
  - Run: `tsc --noEmit` (TS), `mypy src/` (Python), etc.
  - Report error count. Do not fix all — flag the top 3 worst files.
  - **Status: ✅ / ⚠️ (N errors) / ❌ (not configured)**
- [ ] If linter/formatter is missing: flag as ❌ and add a recommended config to the Prioritized Action List
- [ ] Are lint/format/typecheck commands documented in CLAUDE.md? Flag as ❌ if not.

---

## 4. Type Safety

**Why this matters:** Explicit types are contracts AI can read reliably. Untyped code makes AI guess — and it guesses wrong under pressure.

- [ ] Is the project using a typed language or has type annotations?
- [ ] Is strict mode enabled? (TypeScript `"strict": true`, mypy strict, etc.)
- [ ] Identify top 3 files with the most untyped / `any` usage — flag for future attention
- [ ] Are shared data models (API responses, DB schemas, DTOs) defined in one canonical place?
  - If not: note which types are duplicated or implicit

---

## 5. Secrets & Security

**Why this matters:** One committed secret can compromise the project. AI tools also must not read your `.env` file.

- [ ] `.env` is in `.gitignore`
  - Verify: `git check-ignore -v .env` — should output a match
- [ ] `.env.example` exists and lists every required variable
  - If missing: flag as ❌ and add to the Prioritized Action List
- [ ] Scan tracked files for hardcoded secrets:
  ```bash
  git grep -rn "sk-\|api_key\|password\s*=\|secret\s*=\|token\s*=" -- ':!*.example' ':!*.md'
  ```
  Report any hits. Do not remove — flag for human review.
- [ ] `.cursorignore` / `.claudeignore` exists and excludes `.env*`
  - If missing: flag as ❌ and add to the Prioritized Action List
- [ ] Check git history for accidentally committed secrets:
  ```bash
  git log --all --full-history -- "*.env" "**/.env"
  ```
  Report any hits.

---

## 6. Git Hygiene

**Why this matters:** Descriptive commit history is extra context for AI. It reveals *why* code changed, which helps AI make consistent decisions.

- [ ] Are recent commits following a consistent format?
  - Check last 20 commits: `git log --oneline -20`
  - Good: `feat: add user auth`, `fix: prevent null ref in session`
  - Bad: `fix`, `update`, `wip`, `asdf`
- [ ] Conventional commits adopted? (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`)
  - If not: flag as ❌ and add to the Prioritized Action List
- [ ] `.gitignore` appropriate for the stack?
  - Check for common omissions (build artifacts, IDE files, OS files)
  - Reference: gitignore.io for stack-specific templates

---

## 7. Dependencies

**Why this matters:** Outdated or vulnerable dependencies create noise and potential security issues that distract AI and create tech debt.

- [ ] Run the package manager's audit command:
  - JS/TS: `npm audit` or `yarn audit`
  - Python: `pip-audit` or `safety check`
  - Ruby: `bundle audit`
  - Go: `govulncheck ./...`
- [ ] List packages with HIGH or CRITICAL vulnerabilities
- [ ] Identify packages that are significantly outdated (major versions behind)
- [ ] Flag any abandoned packages (no releases in 2+ years, archived repo)
- [ ] Do NOT automatically upgrade — report findings for human decision

---

## 8. Module Structure

**Why this matters:** Large files with mixed responsibilities are hard for AI to reason about and easy for it to break.

- [ ] Find files over 300 lines:
  ```bash
  find . -name "*.ts" -o -name "*.py" -o -name "*.js" | xargs wc -l | sort -rn | head -20
  ```
  (adjust extensions for your stack)
- [ ] For each large file: does it have a single clear responsibility? Flag files that mix multiple concerns.
- [ ] Are there files named `utils`, `helpers`, `common`, or `misc` that have grown into catch-all dumping grounds? Flag them.
- [ ] Are imports clean? (No circular deps, no reaching deep into another module's internals)

---

## 9. Testing State

**Why this matters:** Tests are the feedback loop AI uses during agentic tasks. Without tests, AI edits blindly.

- [ ] Does a test framework exist?
  - Look for: `jest.config.*`, `vitest.config.*`, `pytest.ini`, `spec/` folder, etc.
- [ ] Run tests: `npm test` / `pytest` / equivalent
  - Report: number of tests, pass/fail, coverage if available
- [ ] Identify the 3 most critical paths that have NO test coverage
  - Priority: auth flows, payment/billing logic, data mutation endpoints
- [ ] Is there a single command to run all tests?
  - If not: flag as ❌ and add to the Prioritized Action List

---

## Deliverables

The audit report is the **only file you create** during the audit. Do not create CLAUDE.md, .env.example, config files, or any other project files — those go on the Prioritized Action List for the user to request separately.

Produce `audit-report-YYYY-MM-DD.md` (use today's date) in the `.ai/` or `ai-audit/` folder — whichever exists in this project. If neither exists, save to the project root. If a previous report already exists, do not overwrite it — create a new file with today's date.

```markdown
# AI-Readiness Audit — [Project Name] — YYYY-MM-DD

## Summary
[2–3 sentence overall assessment]

## Status by Area

| Area                    | Status | Notes |
|-------------------------|--------|-------|
| AI instruction files    | ✅/⚠️/❌ | |
| Project comprehension   | ✅/⚠️/❌ | |
| Code quality tooling    | ✅/⚠️/❌ | |
| Type safety             | ✅/⚠️/❌ | |
| Secrets & security      | ✅/⚠️/❌ | |
| Git hygiene             | ✅/⚠️/❌ | |
| Dependencies            | ✅/⚠️/❌ | |
| Module structure        | ✅/⚠️/❌ | |
| Testing                 | ✅/⚠️/❌ | |

## Prioritized Action List
1. [Most critical — blocks AI work]
2. [Important — should fix soon]
3. [Nice to have — low urgency]

## Issues Requiring Human Decision
- [anything that needs manual review, e.g., exposed secrets, vulnerable deps]

## Nice to Have
- [ ] `.ai/prd.md` — product goals, user personas, success metrics (helps AI understand *why* features exist)
- [ ] `.ai/tech-stack.md` — technology choices with rationale (e.g., "we use X instead of Y because…")
- [ ] `.ai/decisions/` — Architecture Decision Records for non-obvious choices
- [ ] `CONTRIBUTING.md` — branch strategy, PR conventions, review process
- [ ] `.ai/glossary.md` — domain-specific terms AI should know (e.g., "a 'session' here means X, not a user session")
```
