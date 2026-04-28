# AI Instruction Files — Setup Guide for All Tools
> Reference for creating and maintaining AI instruction files across Claude Code, Cursor, Copilot, and async agents. Attach this file when setting up a new project or auditing existing AI configuration.

---

## 1. Why Instruction Files Matter

Without instruction files, every AI session starts cold — no knowledge of your project, conventions, or constraints. You repeat yourself in every conversation, and the AI makes the same wrong assumptions every time.

Instruction files are persistent memory. Write them once, get consistent behavior across all sessions.

**Rule:** Update instruction files whenever conventions change. An outdated CLAUDE.md is worse than none — it actively misleads.

---

## 2. CLAUDE.md (Claude Code)

**Location:** repo root (`CLAUDE.md`) or nested subfolder for sub-projects

**Loaded automatically** at the start of every Claude Code session in that directory.

**Target size: Under 200 lines.** The Claude Code system prompt already uses ~50 instruction slots. LLMs reliably follow roughly 150–200 instructions total. Anything beyond that is silently ignored — it adds cost without benefit.

**Filter test for every line:** "Would removing this cause the agent to make a mistake on any task?" If not, cut it or move it to a skill or separate doc.

### What CLAUDE.md must NOT contain
- Code style rules — delegate to ESLint/Prettier. Never use instruction slots for what a linter enforces deterministically.
- Standard language conventions the AI already knows ("use async/await", "write clean code")
- File-by-file descriptions of the codebase — the agent reads code directly
- Frequently-changing information (version numbers, team names, sprint goals)
- Detailed API documentation — link to docs instead
- **Code snippets as examples** — use `file:line` references to live code instead: "follow the pattern in `src/services/UserService.ts:45-60`". Snippets go stale silently; references always point to the current version and cost zero tokens in CLAUDE.md.

### Minimum viable CLAUDE.md

```markdown
# CLAUDE.md — [Project Name]

## Project Overview
[One paragraph: what it does, who uses it, what problem it solves]

## Tech Stack
- Language: [e.g., TypeScript 5.4]
- Framework: [e.g., Next.js 14 App Router]
- Database: [e.g., PostgreSQL via Supabase]
- Testing: [e.g., Vitest + Playwright]
- Styling: [e.g., Tailwind CSS 3]

## Folder Structure
```
src/
  app/          # Next.js App Router pages and layouts
  components/   # Shared React components
  lib/          # Business logic, utilities, service wrappers
  types/        # Shared TypeScript types and DTOs
```

## Conventions
- Naming: kebab-case for files, PascalCase for components, camelCase for functions
- File size limit: 300 lines — split if larger
- Each file has one responsibility — no catch-all utils files
- No `any` type in TypeScript — use `unknown` and narrow
- Imports: absolute paths from `src/` using `@/` alias

## Commit Format
feat: add user authentication
fix: prevent null dereference in session handler
chore: update dependencies
docs: add API endpoint documentation
refactor: extract payment logic to service
test: add unit tests for UserService

## Commands
- Dev server: `npm run dev`
- Tests (unit): `npm run test`
- Tests (e2e): `npm run test:e2e`
- Lint: `npm run lint`
- Type check: `npm run typecheck`
- Format: `npm run format`

## NEVER
- Modify migration files (create new ones only)
- Expose secrets or API keys in responses or logs
- [Add project-specific hard prohibitions here]

## Out of scope — do not implement
- [Features or areas explicitly outside current scope]

## Do NOT modify without explicit instruction
- [List any off-limits files — e.g., migration files, generated code, payment logic]

## Definition of Done
- New endpoint: handler + test with happy path + error case
- New service: handler + service + tests + env.example updated
- Bug fix: failing test that reproduces the bug + fix that makes it pass
- [Customize per project]

## When compacting, preserve
- List of files modified this session
- Names of any failing tests
- Current branch name
- Any domain rule that was violated and corrected

## Workflow
- Plan → implement in 3-step chunks → commit after each working stage
- Run tests after every change (`npm test`)
- Use /compact in long sessions, /clear between unrelated tasks
- If confused: stop, create status.md, start new session
```

### Three-Tier Memory Model

Structure your instruction files across three tiers to avoid overloading hot context:

```
TIER 1 — Hot Memory (always loaded, every session)
  Root CLAUDE.md: <200 lines
  → NEVER rules, Definition of Done, compaction instructions,
    naming conventions, @imports to other docs

TIER 2 — Warm Memory (loaded per domain)
  .claude/agents/ — specialized subagents
  Per-directory CLAUDE.md — loaded when agent enters that directory

TIER 3 — Cold Memory (loaded on demand)
  .claude/commands/ — workflow playbooks, loaded via /project:name
  docs/ — architecture, product specs, API docs
  Loaded via explicit @import or when agent opens a file in that folder
```

**Anti-pattern:** Putting everything in Tier 1. A session about a bug in the auth module shouldn't also load the entire search pipeline architecture doc.

### CLAUDE.md for subfolders

Large repos or monorepos can have multiple CLAUDE.md files:
- `CLAUDE.md` — root: project overview, global conventions, NEVER rules
- `src/backend/CLAUDE.md` — backend-specific rules, DB patterns, handler conventions
- `src/frontend/CLAUDE.md` — UI conventions, component patterns, what must not appear in UI

Claude Code reads all CLAUDE.md files in the path from root to the current working file.

Each subdirectory CLAUDE.md should cover: role of this module (one sentence), any deviations from root conventions, non-obvious architectural decisions, what NOT to do here.

### CLAUDE.local.md (per-developer overrides)

Create `CLAUDE.local.md` in the repo root for personal preferences that shouldn't affect the team:
```markdown
# My personal overrides
- Always show me the full file diff before editing
- Ask before running any terminal command
```

Add to `.gitignore`: `CLAUDE.local.md`

---

## 3. Cursor Rules (.cursor/rules/)

**Location:** `.cursor/rules/*.mdc` (Cursor 0.45+) or `.cursorrules` (legacy, single file)

**Loaded:** Per file type or globally, depending on frontmatter config.

**Size limit:** Under 500 lines per file. Split into multiple files by domain.

### Recommended split for .mdc files

```
.cursor/rules/
  shared.mdc      # Project overview, naming, git workflow (always loaded)
  frontend.mdc    # Component patterns, styling, UI conventions
  backend.mdc     # API routes, business logic, database access
  testing.mdc     # Test conventions, what to test, what not to test
```

### Frontmatter options for .mdc files

```yaml
---
description: Frontend development rules for React components
globs: ["src/components/**/*.tsx", "src/app/**/*.tsx"]
alwaysApply: false
---
```

- `alwaysApply: true` — loaded in every chat/edit context (use for core project rules)
- `alwaysApply: false` + `globs` — loaded only when matching files are in context (use for domain rules)

### .cursorrules (legacy format)

Plain markdown, no frontmatter. Loaded globally for all sessions in the repo. Use `.cursor/rules/` instead for new projects — it gives you per-domain control.

### .cursorignore

Prevents files from being read into Cursor context:
```
.env*
*.env
dist/
.next/
node_modules/
*.lock
```

Equivalent files for other tools:
- Claude Code: `.clineignore` (same format)
- General AI tools: `.aiignore`

Always exclude: secrets, build artifacts, generated files, lock files, large binary assets.

### Generating rules automatically

**10xRules.ai** — generates `.cursor/rules/*.mdc` files from your project description. Useful starting point; review and prune before using.

**cursor.directory** — community-contributed rule sets by technology. Good reference for standard patterns.

**Rule: always review generated rules** — auto-generated rules often include generic advice the model already knows. Keep only project-specific constraints.

---

## 4. GitHub Copilot (.github/copilot-instructions.md)

**Location:** `.github/copilot-instructions.md`

**Loaded:** Automatically in VS Code with the Copilot extension.

**Format:** Plain markdown. No special syntax.

Content mirrors CLAUDE.md — copy the key sections (stack, conventions, patterns, what to avoid). Copilot has a smaller effective instruction budget than Claude Code, so keep it concise: under 200 lines.

---

## 5. AGENTS.md (Async Agents)

**Location:** repo root (`AGENTS.md`)

**Use when:** OpenAI Codex, Jules (GitHub), Devin, or other async/autonomous agents will work on this repo.

**Purpose:** Documents what agents are and aren't allowed to do autonomously, what requires human approval, and how to hand off work.

```markdown
# AGENTS.md — [Project Name]

## Scope
Agents working in this repo may:
- Read all files except .env* and credentials
- Create new files in src/, tests/
- Modify existing files in src/, tests/
- Run: npm test, npm run lint, npm run typecheck

Agents may NOT without human approval:
- Modify database migration files
- Change CI/CD pipeline configuration
- Push to main or create pull requests
- Install new dependencies
- Modify .env.example or any secret-adjacent config

## Approval Workflow
1. Agent opens a draft PR with changes
2. Human reviews diff before merging
3. Human runs E2E tests manually after merge

## Handover Format
When done, agent must create .ai/agent-status.md:
- What was completed
- What was skipped and why
- Open questions requiring human input
```

---

## 6. Claude Code: Advanced Configuration

### Permissions (.claude/settings.json)

Controls what Claude Code can read/run/write without asking:

```json
{
  "permissions": {
    "allow": [
      "Read(src/**)",
      "Write(src/**)",
      "Bash(npm run *)"
    ],
    "ask": [
      "Write(*.config.*)",
      "Bash(git push *)"
    ],
    "deny": [
      "Read(.env*)",
      "Bash(rm -rf *)"
    ]
  }
}
```

**Key principle:** Deny read access to `.env*`. AI tools should never read your actual secrets — they work from `.env.example`.

### Custom Commands (.claude/commands/)

Project-level slash commands for repetitive workflows:

**`.claude/commands/review.md`:**
```markdown
Review the staged changes for this PR:
1. Check for logic errors and edge cases
2. Verify no secrets or credentials are included
3. Confirm tests cover the changed paths
4. Check for TypeScript errors
5. Report: ready to merge / needs changes (list specific issues)
```

Usage in Claude Code: `/project:review`

**Global commands** (available across all projects): `~/.claude/commands/`

### Hooks (.claude/settings.json)

Run shell commands automatically on Claude Code events:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "npm run lint --silent" }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "echo 'Session ended' >> .ai/session-log.txt" }
        ]
      }
    ]
  }
}
```

Hook events: `PreToolUse`, `PostToolUse`, `Stop`

### Subagents (.claude/agents/)

Create specialized agents for recurring tasks:

**`.claude/agents/reviewer.md`:**
```markdown
---
name: Code Reviewer
description: Reviews code changes for bugs, security issues, and convention violations
model: claude-opus-4-6
tools: Read, Glob, Grep
---

You are a senior code reviewer for this project. When invoked:
1. Read the files specified
2. Check for: logic errors, security vulnerabilities, convention violations, missing tests
3. Report findings as a structured list — do not make any changes
```

Usage: `Use the Code Reviewer agent to check src/auth/`

---

## 7. Other Tool Formats

| Tool | File | Notes |
|------|------|-------|
| JetBrains AI Assistant | `.junie/guidelines.md` | Similar to CLAUDE.md, plain markdown |
| Windsurf | `.windsurf/rules` | Similar to .cursorrules |
| Aider | `.aider.conf.yml` | YAML config + conventions in comments |
| Continue.dev | `.continue/config.json` | JSON with model, context providers, and rules |

**Principle for all tools:** The content is the same — project overview, stack, conventions, what not to touch. Only the file location and format differ.

---

## 8. Keeping Instruction Files in Sync

When you update conventions (new linter rule, new folder structure, renamed pattern):
1. Update CLAUDE.md immediately
2. Update any active `.cursor/rules/*.mdc` files
3. Commit the instruction file changes together with the code change that caused them

**Never let instruction files lag behind the codebase.** Stale instructions produce inconsistent AI behavior that is hard to debug — the AI follows the rules you wrote, not the rules you meant.

### CLAUDE.md as a Living Document: The PR Feedback Loop

The best CLAUDE.md files are not written once — they are iterated over weeks based on real agent behavior.

**The feedback loop:**
1. Agent submits a PR
2. A human leaves a code review comment ("you should have used X", "don't do Y here")
3. That comment is a signal: the agent didn't know this rule
4. Add it to CLAUDE.md before the next session

Every review comment that corrects an agent's behavior is evidence of missing context. Treat the PR review queue as a CLAUDE.md backlog.

**What this means in practice:**
- Don't try to write a perfect CLAUDE.md upfront — you can't predict what the agent will get wrong
- After the first week of AI-assisted development, CLAUDE.md should have 5–10 new rules from real mistakes
- Schedule 10 minutes per sprint to review agent PR comments and extract rules

This is the compounding effect of AI instruction files: each iteration makes the next session cheaper and more accurate.
