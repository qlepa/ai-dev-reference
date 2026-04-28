# New Project — AI-Ready Bootstrap
> You are setting up a brand-new project for AI-assisted development. Work through every section top to bottom. For each item: check if it exists, create it if it doesn't, and report what you did. Do **not** skip sections — if something doesn't apply, say why.

---

## Context *(fill this in before handing to AI)*

```
Project name     : 
Short description: 
Tech stack       : 
Primary AI tool  : [ ] Claude Code  [ ] Cursor  [ ] Copilot  [ ] Other: ___
Repo location    : 
```

---

## 1. Project Documentation

**Goal:** Every AI session starts cold. These files are the persistent memory that gives AI context without you repeating yourself.

- [ ] `README.md` — exists and contains: project purpose, tech stack, how to run locally, env vars needed
- [ ] `.ai/prd.md` — Product Requirements Document: problem being solved, MVP scope (what's IN, what's OUT), success criteria, user stories
- [ ] `.ai/tech-stack.md` — chosen stack with rationale, known constraints, links to official docs
- [ ] `.ai/` folder is gitignored from editors but committed to the repo (it's documentation, not secrets)

**PRD structure for AI-assisted projects:**

```markdown
## Problem
[Specific user pain, market gap, quantified impact]

## MVP Scope
### IN scope
- [Core feature 1 with acceptance criteria]
- [Core feature 2 with acceptance criteria]

### OUT of scope (explicitly excluded)
- [Advanced feature deferred to v2]
- [Nice-to-have deferred to v2]

## Success Metrics
- [Measurable outcome — e.g., "75% of users complete onboarding in under 5 min"]

## Technical Constraints
- Stack: [choices]
- Performance: [requirements]
- Security/compliance: [requirements]
```

**Prompt to generate PRD if missing** (use XML tags to separate sections — prevents AI from mixing instructions with content):

```
Ask me 5–10 clarifying questions, then generate a structured PRD.

<output_format>
- Problem statement (2–3 sentences)
- MVP feature list: IN scope / OUT of scope (explicit)
- Success criteria (measurable, not "users will be happy")
- 3–5 user stories
- Technical constraints
</output_format>

Wait for my answers before writing anything.
```

**Stack selection criteria for AI-driven development:**
- Prefer **popular, well-documented stacks** (React, Next.js, TypeScript, Python) — models have millions of training examples
- Prefer **stable, typed APIs** — TypeScript over JavaScript; typed Python over duck-typed
- Avoid **niche or bleeding-edge frameworks** — post-cutoff APIs cause hallucinations
- Avoid **custom/proprietary tooling** — no examples in model training data

---

## 2. AI Instruction Files

**Goal:** AI tools read these files at the start of every session. They define how to work in this repo — conventions, what to avoid, project-specific patterns.

- [ ] `CLAUDE.md` (root) — for Claude Code. Must contain:
  - One-paragraph project overview
  - Tech stack with key library versions
  - Folder structure overview (which folder does what)
  - Coding conventions (naming, file size limits, import style)
  - Conventional commits format used in this repo
  - **NEVER section** — hard prohibitions stated explicitly ("Never delete migration files", "Never commit .env")
  - **Definition of Done per task type** — e.g., "new endpoint: handler + test (happy path + error case); bug fix: failing test that reproduces the bug + fix"
  - **Compaction preservation** — "When compacting, always preserve: list of files modified, failing test names, current branch name, any domain rule that was violated"
  - Any known footguns or non-obvious patterns
- [ ] `.cursorrules` or `.cursor/rules/*.mdc` — for Cursor. Mirror CLAUDE.md content, adapted to Cursor format. Recommended split:
  - `shared.mdc` — project overview, naming, git workflow
  - `frontend.mdc` — UI conventions (if applicable)
  - `backend.mdc` — API/business logic conventions (if applicable)
  - `testing.mdc` — test conventions
- [ ] `.github/copilot-instructions.md` — for GitHub Copilot (optional, same content as CLAUDE.md)
- [ ] `AGENTS.md` (root) — only if async agents (OpenAI Codex, Jules, Devin) will be used. Documents allowed actions, off-limits areas, approval workflows.

**Rules for AI instruction files:**
- `CLAUDE.md`: **under 200 lines** — Claude Code system prompt already uses ~50 instruction slots; models reliably follow ~150–200 instructions total. Lines beyond that are silently ignored.
- `.cursor/rules/*.mdc` files: under 500 lines per file. Split by domain (shared / frontend / backend / testing).
- Write rules in imperative English ("Use X", "Never Y", "Always Z")
- Update these files whenever project conventions change
- Optional helpers: any rules generator can be used as a starting point; treat generated rules as drafts and keep final rules repository-owned.

---

## 2.5 Claude Code Setup *(skip if not using Claude Code)*

**Goal:** Hooks, skills, and subagents turn Claude Code from a chat tool into an automated development assistant with guaranteed feedback loops.

- [ ] `.claude/settings.json` created with a typecheck hook — catches type errors immediately after every file edit, not 10 edits later:
  ```json
  {
    "hooks": {
      "PostToolUse": [
        {
          "matcher": "Edit|Write",
          "hooks": [{ "type": "command", "command": "npm run typecheck 2>&1 | tail -5" }]
        }
      ]
    }
  }
  ```
- [ ] Hook blocks writes to protected files if relevant (migrations, generated files):
  ```json
  { "matcher": "Write", "hooks": [{ "type": "command", "command": "bash -c 'echo $CLAUDE_FILE_PATH | grep -q migrations && echo BLOCKED: edit migrations manually && exit 1 || exit 0'" }] }
  ```
- [ ] `.claude/agents/` directory created — place specialized subagent files here (e.g., `security-reviewer.md`, `test-writer.md`)
- [ ] `.claude/skills/` directory created — place reusable workflow playbooks here (e.g., `fix-github-issue.md`, `create-service.md`)
- [ ] `CLAUDE.local.md` added to `.gitignore` — for per-developer personal overrides that should not affect teammates

---

## 3. Code Quality Tooling

**Goal:** Linters give AI an automatic feedback loop — it can see errors and self-correct without your intervention.

- [ ] Linter configured (ESLint / Pylint / RuboCop / golangci-lint / or equivalent for your stack)
  - Config file committed to repo
  - Runs with a single command (e.g., `npm run lint`)
  - Zero errors on a clean repo (fix existing issues first)
- [ ] Formatter configured (Prettier / Black / gofmt / or equivalent)
  - Config file committed to repo
  - Format-on-save enabled in project config (`.vscode/settings.json` or equivalent)
- [ ] Type checker passes with zero errors (TypeScript `tsc --noEmit`, mypy, etc.)
- [ ] Pre-commit hook or CI step runs lint + format check automatically
  - Tool: husky + lint-staged (JS), pre-commit (Python), lefthook (polyglot), or equivalent
  - Hook must run before every commit, not just in CI

**Document in CLAUDE.md:** the exact commands to run lint, format, and type-check.

---

## 4. Type Safety

**Goal:** Explicit types are a contract the AI can read. Dynamic/untyped code forces AI to guess — and it guesses wrong.

- [ ] Strict mode enabled:
  - TypeScript: `"strict": true` in `tsconfig.json`
  - Python: mypy with `strict = true` or typed annotations throughout
  - Other: enable strictest available setting for your language/linter
- [ ] No `any` / `Object` / untyped generics in source files (only in generated code or `*.d.ts`)
- [ ] Shared data models and DTOs defined in one place (not duplicated across files)
- [ ] API contracts and database schemas have matching types in code

---

## 5. Secrets & Security

**Goal:** Prevent secrets from ever touching git. One leaked key can compromise the whole project.

- [ ] `.env` is in `.gitignore` (verify with `git check-ignore -v .env`)
- [ ] `.env.example` is committed — lists every required variable with a placeholder value and a comment explaining what it is
- [ ] `.cursorignore` / `.claudeignore` excludes `.env*` from AI context (AI tools must not read your secrets)
- [ ] No hardcoded API keys, passwords, or tokens anywhere in tracked files
  - Scan: `git grep -rn "sk-\|api_key\|password\|secret"` — review any hits
- [ ] If using LLM APIs: spending limits set in the provider's dashboard

---

## 6. Git Setup

**Goal:** Readable git history is additional context for AI. Descriptive commits help AI understand why code changed, not just what changed.

- [ ] Conventional commits format adopted:
  ```
  feat: add user authentication
  fix: prevent race condition in session handler
  chore: update dependencies
  docs: add API endpoint documentation
  refactor: extract payment logic to service
  test: add unit tests for UserService
  ```
- [ ] Branch naming convention documented in CLAUDE.md (e.g., `feat/auth`, `fix/login-error`)
- [ ] `.gitignore` is appropriate for the stack (use gitignore.io or GitHub's templates as base)
- [ ] First commit contains: project scaffold + linter config + CLAUDE.md + `.env.example`

---

## 7. Project Structure & Module Design

**Goal:** AI works better with small, focused files. A file that does one thing is self-documenting.

- [ ] Single Responsibility Principle enforced at file/module level:
  - Each file has one clear purpose
  - File name reveals what it does (`user-auth.service.ts`, not `utils.ts`)
  - No file exceeds 300–400 lines (if it does, split it)
- [ ] Folder structure follows a clear convention (feature-based, layer-based, or framework default)
- [ ] Folder structure documented in CLAUDE.md with a one-line description of each top-level folder
- [ ] No circular dependencies between modules
- [ ] Framework's own scaffolding/CLI used for project init — **never generate the entire project from AI from scratch** (official CLI tools produce correct config; AI often gets build tool nuances wrong)

---

## 8. Testing Scaffold

**Goal:** Tests are living documentation. They tell AI what the code is supposed to do, and give it a feedback loop when making changes.

- [ ] Unit test framework configured (Vitest / Jest / pytest / RSpec / Go test / or equivalent)
  - Runs with a single command (`npm test`, `pytest`, etc.)
  - **Configured for non-interactive execution** — no watch mode: `"test": "vitest run"` not `"vitest"`. Watch mode hangs AI agents in non-interactive terminals.
  - At least one example test file exists and passes
- [ ] E2E framework configured if this is a web app (Playwright / Cypress / or equivalent)
  - At least one smoke test exists
- [ ] Test conventions documented in CLAUDE.md or `testing.mdc`:
  - Where tests live (`__tests__/`, `*.test.ts`, `spec/`)
  - What to test (business logic, services, API endpoints)
  - What NOT to test (third-party libs, simple presentational components)
- [ ] Tests run in CI (or a plan exists for it)

---

## 9. Workflow Documentation

**Goal:** When you hand a task to AI mid-project, it needs to know the process, not just the code.

- [ ] CLAUDE.md includes the **3×3 implementation workflow**:
  > For every feature: plan 3 steps → implement → review → next 3 steps → repeat. Commit after each working stage.
- [ ] CLAUDE.md includes context management instructions:
  > Use `/clear` between unrelated tasks. Use `/compact` in long sessions. Start a new conversation if context exceeds 50 messages.
- [ ] CLAUDE.md includes the recovery procedure:
  > If the AI gets confused: stop, create a `status.md` with current state, start a new conversation with that file as context.

---

## Deliverables

At the end of this setup, produce `setup-report-YYYY-MM-DD.md` (use today's date) in the `.ai/` or `ai-audit/` folder — whichever exists in this project. If a previous report already exists, do not overwrite it — create a new file with today's date.

```markdown
# Setup Report — [Project Name] — YYYY-MM-DD

## Created
- List every file created, with a one-line description

## Already existed / skipped
- List any item skipped, with reason

## Issues found
- List any problems that need manual attention

## Next steps
- Top 3 recommended actions before starting development
```
