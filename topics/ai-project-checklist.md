# AI-Native Project Checklist
> A prompt-ready checklist for auditing or bootstrapping any codebase for effective, safe, optimal, and cost-efficient AI agent development.
>
> **How to use:**
> - **Audit mode:** Paste this into Claude with your codebase open. Ask: *"Audit this repository against each item in the checklist. For each item: current state, gap, and recommended fix."*
> - **Bootstrap mode:** Paste this at project start. Ask: *"Set up this project according to every item marked CRITICAL. Skip OPTIONAL items and ask before implementing RECOMMENDED items."*
>
> Items are tagged: 🔴 CRITICAL · 🟡 RECOMMENDED · 🟢 OPTIONAL

---

## 1. Instruction Files — What the Agent Reads

### 1.1 Root CLAUDE.md (or equivalent: AGENTS.md, .cursorrules, Copilot instructions)

- 🔴 **Exists and is committed to git** — agents have no project context without it
- 🔴 **Under 200 lines** — LLMs reliably follow ~150–200 instructions total; the Claude Code system prompt already uses ~50 slots. Longer files cause rules to be silently ignored.
- 🔴 **Contains only universally applicable rules** — every line must answer: *"Would removing this cause the agent to make a mistake on any task?"* If not, cut it or move it to a skill/doc.
- 🔴 **Has a "NEVER" section** — hard prohibitions stated explicitly (never delete migrations, never expose internal package names in UI, never create .env.local, etc.)
- 🔴 **Has a Definition of Done per task type** — what must be true before the agent considers a task complete (e.g., "new endpoint: handler test with happy path + error case; new service: handler + service + repo + test + .env.example")
- 🔴 **Has compaction preservation instructions** — tells the agent what to remember when context compresses: *"When compacting, always preserve: modified file list, failing test names, current branch, and any violated domain rule"*
- 🟡 **Distinguishes domain code from generic/infrastructure code** — domain code (business rules, prompts, state machines) has different change rules than generic scaffolding
- 🟡 **Uses `@path/to/file` imports** for supplementary docs instead of inlining everything
- 🟡 **Has a "how to write issues for the overnight/autonomous worker"** section if async agents are used
- 🟢 **Has a `CLAUDE.local.md`** (gitignored) for per-developer overrides

### What CLAUDE.md must NOT contain
- 🔴 **No code style rules** — use a linter. "Never send an LLM to do a linter's job." Every style rule in CLAUDE.md that a linter could enforce is a wasted instruction slot.
- 🔴 **No standard language conventions** the agent already knows (e.g., "use async/await", "write clean code")
- 🔴 **No file-by-file descriptions of the codebase** — the agent reads code; it doesn't need a summary of what files exist
- 🟡 **No frequently-changing information** (version numbers, team member names, current sprint goals)
- 🟡 **No detailed API documentation** — link to docs instead

### 1.2 Per-directory CLAUDE.md files (monorepos)

- 🟡 **Each major subsystem has its own CLAUDE.md** — placed in the subdirectory, automatically loaded when agent works in that area
- 🟡 **Service/package CLAUDE.md covers:** role (one sentence), database engine and schema, handler pattern if it deviates from root convention, non-obvious architectural decisions, what NOT to do here, what is NOT IMPLEMENTED yet
- 🟢 **Frontend directory has its own CLAUDE.md** with binding UX rules (layout decisions, component patterns, what must never appear in UI)

### 1.3 Supporting documentation

- 🟡 **Architecture docs exist** and are `@imported` from CLAUDE.md — not inlined
- 🟡 **Product/domain model doc** (PRODUCT.md or equivalent) — defines ubiquitous language, entity lifecycle, business rules; `@imported`, not inlined
- 🟡 **NOT IMPLEMENTED features are marked in code** with inline comments pointing to the relevant doc, not only in architecture docs:
  ```typescript
  // [NOT IMPLEMENTED] district swap — see docs/architecture/search-pipeline.md#phase-5
  ```
- 🟡 **`audyt-*.md` or decision logs** referenced from the relevant CLAUDE.md so agents discover them

---

## 2. Automated Quality Gates — Deterministic Enforcement

### 2.1 Linting & Formatting

- 🔴 **ESLint (or equivalent) is configured** at root with rules for: no-explicit-any, consistent imports, no unused vars
- 🔴 **Prettier (or equivalent) is configured** — formatting is never a human or agent decision
- 🔴 **Both run as part of CI** — PRs cannot merge with lint or format violations
- 🟡 **Lint and format run as pre-commit hook** (Husky + lint-staged or equivalent) — agents get instant feedback, not just at CI

### 2.2 Type Checking

- 🔴 **TypeScript strict mode** is enabled (`strict: true`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitReturns`) — this is the single most effective guardrail for AI-generated code
- 🔴 **`tsc --noEmit` runs in CI** for every package and service — not just the shared packages
- 🔴 **Every package/service has a `"typecheck"` script** in `package.json` so CI can run `pnpm --recursive run typecheck`
- 🟡 **Frontend build runs in CI** — catches Next.js/Vite specific type and build errors that tsc alone misses

### 2.3 Testing

- 🔴 **Tests exist and run in CI** — if CI doesn't run tests, agents cannot verify their own work
- 🔴 **Test patterns are documented** (which layer to test, what to mock, what not to mock) — without this, agents write tests that test the wrong thing or mock too aggressively
- 🟡 **Coverage thresholds are configured** — prevents agents from reducing coverage silently
- 🔴 **Test command runs in non-interactive mode** — `"test": "vitest run"` not `"vitest"`. Watch mode hangs AI agents waiting for input that never comes; the agent loops forever instead of completing.
- 🟡 **DB/integration tests are separated** from unit tests (`*.db-test.ts` vs `*.test.ts`) — fast CI runs unit tests; slower integration tests run on demand
- 🟢 **Test data factories (TDF) exist** for all domain types — agents can create test data without building objects from scratch (reduces test noise, encodes domain invariants)

### 2.4 CI Pipeline Completeness

- 🔴 **CI builds ALL artifacts** (all services, frontend, all packages) — not just shared packages
- 🔴 **CI runs typecheck on ALL packages** — a TS error in a service that CI doesn't build is invisible
- 🟡 **CI enforces commit message format** (commitlint) if the team uses conventional commits
- 🟢 **Branch protection rules** require CI green before merge

---

## 3. Claude Code Specific Features

### 3.1 Hooks — Deterministic Action Triggers

> Hooks run shell commands automatically at specific points in the agent's workflow.
> Unlike CLAUDE.md instructions (advisory), hooks **guarantee** the action happens.

- 🔴 **PostToolUse hook runs typecheck** after every file edit — agents discover type errors immediately, not after 10 more edits
- 🟡 **PostToolUse hook runs lint --fix** after every file edit — keeps style consistent without agent effort
- 🟡 **Hook blocks writes to critical files** (e.g., migrations folder, generated files) — prevents agent from editing files it should not touch
- 🟡 **Hook runs relevant tests** after edits to a service's files
- 🟢 **Hook sends notification** (Slack, webhook) when overnight/autonomous agent completes a task

```json
// .claude/settings.json (example structure)
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "pnpm typecheck 2>&1 | tail -5" }]
      }
    ]
  }
}
```

### 3.2 Skills — On-Demand Playbooks

> Skills are `.claude/skills/{name}/SKILL.md` files loaded on demand.
> Use them for repeatable workflows that shouldn't bloat every conversation.

- 🟡 **`fix-github-issue` skill** — workflow: read issue → explore codebase → implement → write tests → verify → create PR
- 🟡 **`create-service` skill** — scaffolding playbook for new services (files to create, content template, checklist)
- 🟡 **`add-tests` skill** — project-specific test writing guide (which patterns to follow, what to mock, how to use TDF)
- 🟡 **`overnight-worker` skill** — instructions specific to autonomous agent: boundaries of autonomy, what requires human input, how to handle ambiguity
- 🟢 **`deploy` skill** — deployment checklist and commands
- 🟢 **`debug-production` skill** — how to read logs, traces, dashboards for this specific stack

### 3.3 Subagents — Isolated Context Windows

> Subagents run in their own context window. Exploration by a subagent doesn't contaminate the main conversation.

- 🟡 **`security-reviewer` subagent** — reviews code for injection, auth flaws, secrets in code, insecure data handling
- 🟡 **`test-writer` subagent** — writes tests in isolation, avoiding the "I just wrote this code" bias
- 🟡 **`domain-expert` subagent** — deep knowledge of business rules, used when agent needs to verify a decision against domain constraints
- 🟢 **`code-reviewer` subagent** — fresh-context review of a PR before it's submitted

```markdown
// .claude/agents/security-reviewer.md
---
name: security-reviewer
description: Reviews code changes for security vulnerabilities
tools: Read, Grep, Glob
model: sonnet
---
Review the specified files for: SQL/command injection, XSS, exposed secrets,
broken auth, insecure deserialization. Reference OWASP Top 10.
Provide file:line references and suggested fixes. Do not modify files.
```

### 3.4 Claude Code in CI/CD

> Claude Code is not limited to interactive sessions — it can run as a GitHub Actions step.

- 🟢 **Claude Code Action** (`anthropics/claude-code-action`) configured for automated AI-powered PR review — the same CLAUDE.md the developer uses locally is the instruction file the CI agent reads
- 🟢 **Composite Actions** wrap AI steps for reuse across workflows — e.g., a single `ai-review` composite used in both PR and nightly pipelines
- 🟢 **Scheduled AI tasks** (changelog generation, dead link scanning, JSDoc sync) defined as cron-triggered workflows

```yaml
# .github/workflows/pr-review.yml (example)
- uses: anthropics/claude-code-action@beta
  with:
    prompt: "Review this PR against CLAUDE.md conventions. Report violations. Do not approve or merge."
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### 3.5 CLAUDE.md Architecture (Three-Tier)

> Based on "Codified Context Infrastructure" research (arxiv 2602.20478):

```
TIER 1 — Hot Memory (always loaded)
  Root CLAUDE.md: <200 lines
  → naming conventions, NEVER rules, Definition of Done,
    compaction instructions, @imports

TIER 2 — Warm Memory (loaded per domain)
  .claude/agents/ — domain expert subagents
  Per-directory CLAUDE.md files — loaded when agent enters that directory

TIER 3 — Cold Memory (loaded on demand)
  .claude/skills/ — workflow playbooks
  docs/ — architecture, product, UX specs
  Loaded via @import in prompt or automatically by skills
```

**Anti-pattern:** Everything in Tier 1. This means every session about a bug in auth-service also loads the entire search pipeline architecture. Costs tokens, degrades instruction adherence.

---

## 4. Codebase Structure for Agent Navigation

### 4.1 Consistency

- 🔴 **Consistent file naming per layer** — agents navigate by pattern. If 8 services use `handler.ts` and 1 uses `handler.http.ts`, the agent wastes tokens figuring out which convention applies.
- 🔴 **Consistent directory structure** — deviations must be documented in the subdirectory's CLAUDE.md with the reason
- 🟡 **Barrel exports (`index.ts`)** in every package — agents can import from `@scope/package` without reading the entire source tree
- 🟡 **Every module has a clear single responsibility** — agents that edit multi-responsibility files often introduce regressions in the unrelated responsibility

### 4.2 Scaffolding Generator

- 🟢 **Code generation tool** (plop, hygen, or custom script) for new services/modules — eliminates convention drift when agents create new files
- 🟢 **Generated files include a CLAUDE.md template** — ensures new services start with documentation

### 4.3 Domain Boundaries

- 🔴 **Domain code is clearly separated from generic/infrastructure code** — and CLAUDE.md states the distinction explicitly. Agents freely refactor infrastructure; domain requires product approval.
- 🟡 **Domain entities have a single source of truth** — one file per entity type, not spread across services
- 🟡 **Event contracts are typed and centralized** — agents can verify event producers and consumers against a shared schema

### 4.4 "NOT IMPLEMENTED" Markers

- 🟡 **Planned but unimplemented features are marked in code** — not only in docs. Agents reading code see what's intentionally missing vs what was forgotten:
  ```typescript
  // [NOT IMPLEMENTED] Returns null until batch job is implemented — see docs/quality-score.md
  intrinsic_quality_score: null,
  ```

---

## 5. Security & Safety Guardrails

### 5.1 Secrets Management

- 🔴 **No secrets in code** — `.env.example` has placeholder values only, real values in secret manager
- 🔴 **`.env` is gitignored** — `.env.example` is committed
- 🔴 **No `.env.local` or non-standard env file patterns** — documented in CLAUDE.md so agents don't create them
- 🟡 **`.cursorignore` (or `.claudeignore`) excludes `.env*`, `node_modules/`, `dist/`, `*.lock`** — prevents secrets and large artifacts from being silently read into IDE AI context. A missing ignore file means your real API keys can enter the LLM context window.
- 🟡 **Hook blocks commits containing common secret patterns** (API keys, connection strings with passwords)

### 5.2 Dangerous File Protection

- 🟡 **Hooks block agent writes to migration files** — migrations must be human-reviewed; agents can create them but not overwrite existing ones
- 🟡 **Hooks block agent writes to CI/CD workflows** unless explicitly permitted
- 🟡 **Hooks block agent writes to CDK/Terraform/infrastructure files** without explicit confirmation
- 🟢 **Sandbox mode configured** for autonomous/overnight agents — restricts filesystem and network access

### 5.3 Scope Boundaries in CLAUDE.md

- 🔴 **"Out of scope for this project" section** — agents should not implement features outside the defined scope even if asked
- 🟡 **"Never touch these files" list** — generated files, vendored code, migration files, security configs
- 🟡 **"Always ask before" list** — actions that require human confirmation (schema changes, new external dependencies, changes to auth flow)

---

## 6. Cost Optimization

### 6.1 Context Window Management

- 🔴 **CLAUDE.md is lean** — every unnecessary line costs tokens on every single agent invocation
- 🔴 **Practical context window limit is respected** — divide the model's stated context limit by 2–4 for reliable output quality. A 200k-token model produces consistent results up to ~50–100k tokens. Beyond that, quality degrades silently: the AI continues generating but starts ignoring constraints and hallucinating details.
- 🟡 **Long docs are cold memory** (accessed on demand) not hot memory (loaded every session)
- 🟡 **Subagents used for exploration** — research that reads many files happens in a separate context and reports back a summary, not polluting the main conversation
- 🟢 **`/btw` used for quick questions** — doesn't enter conversation history

### 6.2 LLM Model Selection

- 🔴 **Model selection is a documented decision** — not random. Use cheaper models (haiku) for mechanical tasks (tagging, scanning, classification); expensive models (opus/sonnet) only for reasoning-heavy tasks (polished descriptions, complex debugging)
- 🔴 **If using LLM in application code, model selection is in domain layer** — not hardcoded in infrastructure (hexagonal architecture: LlmPort with model as domain type)
- 🟡 **Cost per operation is tracked** — e.g., `$0.02/search query` budget enforced; alarms when exceeded
- 🟢 **Model selection for application LLM calls is validated with systematic evaluation** — use Promptfoo (`promptfoo eval`) with real project prompts and stack-specific assertions (Vitest, ESLint) before committing to a model. Cache by `{provider, config, prompt, vars}` — re-runs are nearly free.

### 6.3 Prompt Engineering as Domain Code

- 🔴 **Prompts are treated as domain code, not infrastructure** — they encode business knowledge and are not "refactored" without product approval
- 🔴 **Prompt changes are versioned** — a `tagging_version` field or equivalent so reprocessing is possible when prompts change
- 🟡 **Prompts live in `*.prompts.ts` files** co-located with the service that uses them — not scattered across infrastructure files

---

## 7. Workflow Patterns for AI Teams

### 7.1 Verification-First Development

- 🔴 **Every agent task has explicit success criteria** — not "implement feature X" but "implement feature X; run the test suite; it must pass; take a screenshot of the UI result"
- 🔴 **Agents run tests after every change** — either by instruction or by hook
- 🟡 **Failing tests are written before fixes** — "write a failing test that reproduces the bug, then fix it so the test passes"

### 7.2 Writer / Reviewer Pattern

- 🟡 **Code review happens in a separate agent session** — fresh context avoids the "I just wrote this, it looks fine" bias
- 🟡 **Overnight/autonomous agent submits PRs** but does not merge — a human or reviewer agent approves
- 🟢 **Security review agent runs on every PR** — separate context, specialized prompt, read-only tools

### 7.3 Autonomous Agent Boundaries

If using scheduled/overnight agents:
- 🔴 **Maximum scope per run is defined** — e.g., "one issue per run", "only files in services/*, never infra/"
- 🔴 **Ambiguous issues are skipped with a comment** — not guessed at
- 🔴 **PRs are created, never merged** by the autonomous agent
- 🟡 **Issues have a structured template** — title (verb: scope), context (relevant files), acceptance criteria (what must be true), boundaries (what not to touch)
- 🟡 **Autonomous agent has its own skill** (overnight-worker/SKILL.md) with: how to handle ambiguity, how deep to go without asking, how to write the PR description

### 7.4 Session Hygiene

- 🟡 **`/clear` between unrelated tasks** — long sessions with mixed context produce worse results
- 🟡 **CLAUDE.md has compaction instructions** — ensures critical context survives history compression
- 🟢 **Sessions are named** (`/rename`) for complex multi-session tasks

---

## 8. Observability for AI-Generated Code

- 🟡 **Distributed tracing is configured** — agents can debug async flows (SQS, event-driven) without reading every log line manually
- 🟡 **Structured logging** — agents can grep logs effectively; JSON > plain text
- 🟡 **Errors have enough context** — stack trace + request ID + relevant input values
- 🟢 **AI operation costs are tracked** — LLM calls per operation, latency, token usage visible in traces

---

## 9. Quick Audit Checklist (Yes/No)

Copy this section and answer Yes/No for a fast assessment:

**Instructions**
- [ ] Root CLAUDE.md exists and is committed?
- [ ] Root CLAUDE.md is under 200 lines?
- [ ] CLAUDE.md has explicit NEVER rules?
- [ ] CLAUDE.md has Definition of Done per task type?
- [ ] CLAUDE.md has compaction preservation instructions?
- [ ] No code style rules in CLAUDE.md (delegated to linter)?
- [ ] Per-subdirectory CLAUDE.md for major subsystems?

**Automated Gates**
- [ ] ESLint configured?
- [ ] Prettier configured?
- [ ] TypeScript strict mode?
- [ ] `tsc --noEmit` runs in CI for ALL packages?
- [ ] All services/packages build in CI?
- [ ] Tests run in CI?
- [ ] Test command is non-interactive (no watch mode)?
- [ ] Pre-commit hooks auto-installed (not manual)?

**Claude Code Features**
- [ ] `.claude/settings.json` with hooks?
- [ ] Typecheck hook runs after file edits?
- [ ] `.claude/skills/` directory with workflow playbooks?
- [ ] `.claude/agents/` directory with specialized subagents?
- [ ] `CLAUDE.local.md` in `.gitignore`?

**Codebase Structure**
- [ ] Consistent file naming across all services?
- [ ] Barrel exports in all packages?
- [ ] Domain code clearly separated from generic/infra?
- [ ] Event contracts typed and centralized?
- [ ] NOT IMPLEMENTED features marked in code (not only in docs)?

**Security**
- [ ] No secrets in code or `.env.example`?
- [ ] `.env` is gitignored?
- [ ] `.cursorignore` / `.claudeignore` excludes `.env*` and build artifacts?
- [ ] Hooks block writes to migrations/infra files?
- [ ] "Out of scope" defined in CLAUDE.md?

**Cost**
- [ ] Model selection documented per use case?
- [ ] Prompts treated as domain code (not freely refactorable)?

**Workflow**
- [ ] Agents have explicit success criteria per task?
- [ ] Writer/reviewer pattern for complex changes?
- [ ] Autonomous agent has defined scope boundaries?
- [ ] Autonomous agent creates PRs but never merges?

---

## 10. Scoring

Count your Yes answers:

| Score | Assessment |
|-------|-----------|
| 32–36 | AI-native ready — agents will be effective, safe, and cost-efficient |
| 24–31 | Good foundation — address CRITICAL gaps before scaling AI development |
| 14–23 | Significant gaps — AI will produce inconsistent results; fix quality gates first |
| <14 | High risk — AI development will amplify existing problems; bootstrap from scratch |

---

*Sources: [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices) · [Codified Context Infrastructure](https://arxiv.org/abs/2602.20478) · [Context Engineering for Coding Agents](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html) · [Writing a good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md)*
