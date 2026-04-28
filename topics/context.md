# Context Management — Keeping AI Effective in Long Sessions
> Reference for managing AI context windows efficiently. Attach this file when working on long-running sessions, multi-step implementations, or when AI output quality has started to degrade.

---

## 1. Why Context Management Matters

Every AI session has a context window — a limit on how much text (code + conversation + files) it can hold at once. As the window fills:
- Earlier messages get compressed or dropped
- The AI "forgets" decisions made early in the session
- Output quality degrades (contradictions, repeated questions, wrong assumptions)
- Responses slow down and cost more

Managing context is not optional on serious development work — it is the difference between a productive 2-hour session and a confused 30-minute one.

---

## 2. Claude Code: Context Commands

### `/compact`
**What it does:** Summarizes the current conversation into a compact form, freeing context window space while preserving key decisions and state.

**When to use:**
- You have been working for 30+ messages and notice slowdowns
- You are about to start a new sub-task within the same session
- Before switching from planning to implementation
- When the AI starts asking questions it already answered earlier

**How to use:**
```
/compact
```
Claude will summarize what has been accomplished. Review the summary — if something important was lost, state it explicitly in your next message.

### Compaction Preservation Instructions

Tell Claude what to remember during compaction by adding this section to your CLAUDE.md:

```markdown
## When compacting, always preserve
- List of files modified this session
- Names of any currently failing tests
- Current branch name and what it's for
- Any domain rule that was violated and corrected
- Key decision made this session with the reasoning
```

Without this, compaction may preserve the general direction but lose the specific decisions that led here — causing Claude to re-ask or re-decide things already settled.

### `/clear`
**What it does:** Completely resets the conversation. No memory of previous messages.

**When to use:**
- Switching to a completely unrelated task
- After a session that went badly (wrong direction, bad code generated)
- Starting fresh after you have committed working code

**Important:** `/clear` loses all context. Always commit or save your work before clearing.

### `/context`
**What it does:** Shows current token usage — how full the context window is.

**When to use:** Check before starting a complex task to know how much runway you have.

---

## 3. Plan Mode (Claude Code)

**What it does:** Switches Claude into a read-only planning mode. Claude can read files and explore the codebase but cannot make any changes until you approve the plan.

**How to activate:** Press `Shift+Tab` twice (or use the mode selector in the UI).

**When to use Plan Mode:**
- Before any significant refactor — understand scope before touching code
- When you are unsure of the blast radius of a change
- When you want to review the AI's approach before it starts writing
- On legacy codebases where mistakes are hard to reverse

**Workflow:**
1. Activate Plan Mode
2. Describe what you want
3. Claude explores and writes a plan
4. You review and approve (or ask for changes)
5. Exit Plan Mode → Claude implements

---

## 4. Context Rot and Drift

**Context rot:** The gradual degradation of AI output quality as the context window fills with stale or contradictory information.

**Signs of context rot:**
- AI repeats questions you already answered
- AI contradicts decisions made earlier in the session
- Output quality is noticeably worse than at the start
- AI hallucinates details about your codebase

**Context drift:** The AI's working model of your codebase or task drifts away from reality as the session continues.

**Signs of context drift:**
- AI references a function that no longer exists (or was renamed)
- AI generates code inconsistent with the actual file structure
- AI forgets a constraint you stated at the start

**Prevention:**
- Use `/compact` proactively (before rot sets in, not after)
- Restate key constraints at the start of each new sub-task
- Keep CLAUDE.md updated — it is re-read at the start of each session
- Reference files explicitly (`see src/auth/session.ts`) rather than relying on AI memory

---

## 5. The Rule of Three

**If you have corrected the AI on the same point three times in one conversation, stop and reset.**

The context is poisoned. The AI has learned a wrong pattern and will keep reproducing it. No amount of correction within the same session will reliably fix it.

**Recovery procedure:**
1. Ask the AI to produce `status.md` (what's done, what's next, key decisions)
2. Commit any working code to git
3. Start a new session
4. Open with: "Read status.md. Continue from the next step."

---

## 6. Token Budget Strategy

### High-token tasks (use sparingly within a session)
- Reading large files (>500 lines) — read only the relevant section
- Generating large amounts of code in one message
- Asking for exhaustive analysis of the whole codebase

### Low-token tactics (prefer these)
- Reference files by path instead of pasting content: "See `src/utils/auth.ts`"
- Ask for targeted changes: "Update only the `validateToken` function"
- Use `/compact` before switching tasks
- Break large features into commits — each commit = potential context reset point

### Tool output compression

Raw outputs from tools consume far more tokens than their informational value warrants. Before a large output enters conversation history, compress it:

| Raw output | Compressed equivalent |
|-----------|----------------------|
| Full `git diff` of 500 lines | "Changed files: auth.ts (middleware rewrite), session.ts (added expiry logic)" |
| Full test run output | "17 passed, 2 failed: `should reject expired tokens` (line 45), `should validate scope` (line 89)" |
| Full API response (JSON, 2000 tokens) | Extract only the fields relevant to the task |
| `npm audit` full report | "3 high vulnerabilities: lodash (prototype pollution), axios (SSRF), moment (ReDoS)" |

**How to trigger compression:** Ask the AI explicitly — "Summarize the test output, don't paste it raw" or add to CLAUDE.md: "When running tests, report only failures with file:line references, not the full output."

**Why it matters:** A single uncompressed `git diff` on a medium-sized PR can consume 5–10% of your context budget on information the AI already has (it just wrote the code).

### Session hygiene
- Start sessions with the specific task, not open-ended exploration
- End sessions by committing what works — clean break for the next session
- Keep CLAUDE.md, `.ai/context.md`, and other reference files up to date — they are cheaper than re-explaining everything in the conversation

---

## 7. Multi-Session Work: Handover Pattern

For work that spans multiple sessions, maintain a handover file:

**`.ai/status.md`** — updated at the end of each session:
```markdown
# Session Status

## Goal
[What we are building / fixing]

## Done
- [x] Created user authentication module (src/auth/)
- [x] Added JWT validation middleware
- [ ] Write tests for auth module
- [ ] Connect auth to protected routes

## Current state
[One paragraph: what exists, what works, what doesn't]

## Next session: start here
[Specific next action, file to open, command to run]

## Key decisions
- Used RS256 instead of HS256 for JWT (reason: distributed verification)
- User table uses UUID not auto-increment (reason: multi-region safe)
```

**Starting a new session:**
```
Read .ai/status.md. This is the current project state.
Also read CLAUDE.md for project conventions.
We are continuing from: [copy the "Next session: start here" line].
```

---

## 8. Context Management for Cursor

Cursor uses a different context model — each file you open or reference in chat is added to context.

**Best practices:**
- Open only the files relevant to the current task
- Use `@file` references instead of pasting code
- Use Cursor's "include file" feature intentionally — every file you add costs context
- Split large tasks across separate Cursor chat windows (each starts fresh)
- Keep `.cursorrules` / `.cursor/rules/*.mdc` under 500 lines — they load every session

**Note on `.cursorignore`:** Add `.env*`, `*.lock`, `dist/`, `.next/` to `.cursorignore`. These files waste context budget when accidentally included and may expose secrets.
