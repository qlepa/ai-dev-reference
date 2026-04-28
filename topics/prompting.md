# Prompting — Effective AI Communication
> Reference for writing prompts that produce accurate, useful, non-sycophantic AI output. Attach this file when you want the AI to help you craft prompts, evaluate its own reasoning, or improve existing instructions.

---

## 1. The Five Elements of a Prompt (Priority Order)

When constructing any prompt, these elements stack — each one added sharpens the output:

| Priority | Element | What it does | Example |
|----------|----------|--------------|---------|
| 1 | **Command** | The core instruction | "Refactor this function to use async/await" |
| 2 | **Context** | Why and what surrounds it | "This runs in a Node.js 18 serverless function with a 10s timeout" |
| 3 | **Format** | How output should look | "Return only the updated function, no explanation" |
| 4 | **Role** | Persona that shapes style | "You are a senior backend engineer doing a code review" |
| 5 | **Examples** | Show, don't just tell | "Like this: `async function foo() { ... }`" |

**Rules:**
- Command is always required. Everything else is optional but additive.
- Role comes last in priority — it shapes tone, not correctness.
- Examples are the most powerful element when output format is hard to describe in words.
- Shorter prompts with all 5 elements beat long prompts with only 1-2.
- **One canonical example beats a list of edge cases.** A well-chosen representative example teaches pattern recognition. A list of edge cases adds noise without improving the core behavior. If you must cover edge cases, show them as variations of the canonical example, not as separate items.

---

## 2. When to Vibe Code vs When to Spec

Not all code deserves the same rigor. The right approach depends on the risk of getting it wrong.

| Layer / Area | Approach | Reason |
|---|---|---|
| UI layout, animations, copy | Vibe | Fast iteration, low consequence, easy to reverse |
| New feature exploration / prototyping | Vibe | Goal is to discover shape, not ship |
| CRUD endpoints, boilerplate | Vibe with review | Pattern-driven, quickly verifiable |
| Business logic, calculations | Spec | Wrong logic costs real money or trust |
| Auth flows, permissions | Spec | Security mistakes are hard to detect and costly to fix |
| State machines, event-driven flows | Spec | Correctness is non-obvious, hard to test exhaustively |
| Database schema changes | Spec | Migrations are irreversible in production |

**The decision rule:** If getting it wrong is easy to detect and easy to reverse, vibe. If getting it wrong could go unnoticed or is hard to undo, spec first.

**Hybrid workflow for features:**
1. Vibe code a rough implementation to validate the idea feels right
2. Once shape is clear, write a `spec.md` capturing requirements and design decisions
3. Start a fresh session and implement against the spec
4. The rough prototype is thrown away — its value was insight, not code

**Never vibe code:** auth logic, payment processing, data migrations, permission checks, anything that could result in data loss or security vulnerabilities.

---

## 3. Language Choice

**Use English for prompts** unless you have a specific reason not to:
- English requires 33–40% fewer tokens than Polish or other European languages for equivalent content
- Fewer tokens = faster responses + lower cost + more room in the context window for your actual code
- On flat-rate plans (Copilot, Windsurf): language choice does not affect cost — use Polish freely
- On usage-based plans (Cursor pay-as-you-go, direct API): Polish costs ~50% more tokens — use English
- Exception: if you need the model to reason about Polish text (e.g., UI copy, error messages), write that content in Polish but keep surrounding instructions in English

---

## 4. Meta-Prompting: Use AI to Improve Your Prompts

When you are unsure how to phrase a complex task, use this template to generate a better prompt:

```
I need to write a prompt that will instruct an AI to [task description].

The AI should:
- [key requirement 1]
- [key requirement 2]
- [constraint 1]

Please generate 3 versions of this prompt, each with a different approach (direct command / role-based / example-driven). Then recommend which version is best for this use case and why.
```

**When to use meta-prompting:**
- You are not getting the output quality you expect
- The task is complex or has many constraints
- You will reuse this prompt many times (worth optimizing once)

---

## 5. The Socratic Method: Ask Before Acting

Before starting a complex or ambiguous task, instruct the AI to gather information first:

```
Before you start, ask me 5–10 clarifying questions that would help you complete this task better. 
Wait for my answers before doing any implementation.
```

**When this matters most:**
- Generating a PRD, architecture plan, or CLAUDE.md from scratch
- Designing a new feature with unclear scope
- Debugging a problem where the AI doesn't have full context
- Any task where a wrong assumption wastes significant effort

**Why it works:** AI models are trained to infer and complete — they will guess missing information. The Socratic method forces the gap into the open before it becomes a buried assumption in generated code.

---

## 6. Anti-Sycophancy Techniques

AI models are trained on human feedback and learn to agree with users. This creates a systematic bias toward telling you what you want to hear. Use these techniques to counteract it.

### 5.1 Devil's Advocate
Force the AI to argue against its own recommendation:
```
You just proposed [solution]. Now argue as strongly as possible for why this is the WRONG approach. 
What are the strongest objections to it? What could go badly?
```

### 5.2 Pre-Mortem
Project the plan into a failure scenario before committing:
```
Imagine it is 6 months from now. We implemented this plan and it failed badly. 
What went wrong? List the 5 most likely causes of failure.
```

### 5.3 Alternative Comparison
Never accept the first option without seeing alternatives:
```
Give me 3 distinct approaches to this problem. For each: pros, cons, risks, and when you would choose it.
Do NOT recommend just one — present all three neutrally so I can decide.
```

### 5.4 Perspective Shift
Ask the AI to reason from a specific stakeholder's viewpoint:
```
Evaluate this solution from the perspective of [security engineer / user with slow internet / 
junior developer maintaining this in 2 years / someone who has never seen this codebase].
What concerns or blockers would they raise?
```

### 5.5 Unknown Unknowns
Explicitly ask for what the AI might be hiding:
```
What important considerations, risks, or factors am I NOT asking about that I should be?
What would you tell me if I wasn't here to approve or reject your ideas?
```

**When to use anti-sycophancy:**
- Architecture decisions with long-term consequences
- Before deleting or refactoring large amounts of code
- When the AI seems unusually confident or enthusiastic
- When evaluating a plan you really want to work (confirmation bias check)

---

## 7. Conversation Recovery

When a long conversation goes off track — wrong direction, accumulated confusion, or drifting context — use this reset procedure:

### Step 1: Create a status snapshot
Ask the AI to produce this file before starting a new conversation:
```
Create a file called status.md with:
1. What we were trying to accomplish
2. What has been completed (list files changed and what was done)
3. Where we got stuck or went wrong
4. What the next concrete step should be
5. Any decisions made that should carry forward
```

### Step 2: Start a fresh conversation
Open a new session and begin with:
```
Read status.md. This is a handover from a previous session. 
Continue from "Next step" without re-doing what's already done.
```

**Signs that recovery is needed:**
- The AI starts repeating itself or going in circles
- The AI contradicts something it said 20 messages ago
- Output quality degrades over a long session
- You've been debugging the same issue for 5+ messages without progress

**Rule of Three:** If you have corrected the AI on the same point three times in one conversation, stop and reset. The context is poisoned.

---

## 8. The 4-Phase Development Workflow

Every non-trivial task should follow four distinct phases. Mixing them is the most common reason AI sessions go off track.

```
Explore → Plan → Code → Commit
```

### Phase 1: Explore (Plan Mode — read only)

Activate Plan Mode (Shift+Tab twice in Claude Code) before touching anything. The AI reads files, maps dependencies, and understands scope without making changes.

```
Read the codebase and tell me:
1. Which files would be affected if I changed the auth middleware?
2. What does the current session handling look like?
3. What tests exist for this area?

Do NOT make any changes yet.
```

**Why it matters:** Most AI mistakes happen because the AI started implementing before it understood the problem. Explore phase prevents solving the wrong thing.

### Phase 2: Plan (human approval required)

After exploration, get a written plan before any code is written. Review it yourself.

```
Based on your exploration, write an implementation plan:
1. Files to create (with purpose)
2. Files to modify (with exact changes)
3. Order of changes (what depends on what)
4. How we will verify it works

Do NOT write any code yet. Wait for my approval.
```

**Skip planning only if** you could write the entire diff in one sentence yourself. For anything else, plan first.

### Phase 3: Code (small, committed chunks)

Implement in stages. Commit after each working stage — not at the end.

```
Implement step 1 from the plan only. Run tests after. Stop and report.
```

Each commit is a checkpoint: if the next step breaks something, you can roll back to the last working state in one command.

### Phase 4: Commit

```
feat: add session expiry validation to auth middleware

- validates token expiry on every request
- returns 401 with descriptive message on expiry
- unit tests cover: valid token, expired token, missing token
```

**The commit message is part of the workflow** — it's future context for both humans and AI. A descriptive commit is cheaper than re-explaining what was done in the next session.

---

## 9. Visualization Techniques

When you need a diagram, schema, or visual representation, request it in a format the AI can generate as text — no external tools required.

| Format | When to use | Example prompt |
|--------|-------------|----------------|
| **ASCII art** | Quick layout sketch, folder trees, flow arrows | "Show the data flow as ASCII art" |
| **Mermaid** | Architecture diagrams, flowcharts, ERDs, sequence diagrams | "Generate a Mermaid flowchart of the auth flow" |
| **SVG** | Icons, simple graphics, custom visuals as code | "Create an SVG icon for a loading spinner" |
| **LaTeX** | Mathematical formulas, algorithms | "Write this formula in LaTeX" |
| **Markdown tables** | Data comparison, trade-off matrices | "Compare these approaches in a table" |

**Mermaid is the best default** — GitHub, Notion, and most dev tools render it natively.

```
Example prompt:
Generate a Mermaid sequence diagram showing:
1. User submits login form
2. Frontend sends POST /auth/login  
3. API validates credentials against DB
4. API returns JWT + refresh token
5. Frontend stores access token in memory, refresh token in HttpOnly cookie
```

**When to request ASCII vs Mermaid:** ASCII for quick whiteboard-style sketches where rendering doesn't matter. Mermaid when you'll share the output in a GitHub PR, Notion doc, or README.

---

## 10. Prompt Patterns for Common Tasks

### Generating documentation
```
Read [file path]. Write a concise docstring/comment for each exported function.
Rules: one sentence per function, describe what it returns, note any side effects.
Do not modify any logic. Only add documentation.
```

### Code review
```
Review [file path] as a senior engineer. Focus on:
1. Logic errors or edge cases that could cause bugs
2. Security issues (injection, auth bypass, exposed secrets)
3. Performance problems that matter at scale
4. Readability issues that will slow down the next developer

Format: bullet list per category. Skip categories with no issues.
Do NOT suggest stylistic rewrites unless they affect correctness.
```

### Debugging
```
I have a bug: [describe exact symptom and when it occurs].

Here is the relevant code: [paste or reference file].
Here is the error or unexpected output: [paste].

Do NOT suggest random changes. Instead:
1. State your hypothesis about the root cause
2. Explain what evidence supports it
3. Propose ONE specific fix
4. Tell me what to verify after applying the fix
```

### Planning before implementing
```
Before writing any code, create an implementation plan for [feature].

Format:
1. Files to create (with purpose)
2. Files to modify (with what changes)
3. External dependencies needed
4. Risk: what could go wrong?
5. Test: how will we verify it works?

Wait for my approval before writing any code.
```

---

## 11. Structuring Complex Prompts with XML Tags

When a prompt contains multiple distinct sections — code, requirements, context, examples — use XML-style tags to separate them. LLMs are trained on structured formats and reliably distinguish tagged sections from untagged prose.

**Why this matters:** Without structure, the AI can conflate your instructions with example code, mix context with requirements, or apply constraints from one section to another. Tags make the boundaries explicit.

### Basic pattern

```
Analyze the code below and suggest improvements based on the requirements.

<requirements>
- Must handle null values without throwing
- Must run in under 50ms for inputs up to 10,000 items
- Cannot use external libraries
</requirements>

<code>
function processItems(items) {
  return items.map(i => i.value * 2)
}
</code>

<context>
This runs in a hot path in a payment processing service.
Called approximately 500 times per second.
</context>
```

### Tags for common scenarios

| Scenario | Useful tags |
|----------|-------------|
| Feature implementation | `<prd>`, `<tech_stack>`, `<existing_code>`, `<requirements>` |
| Bug fix | `<bug_description>`, `<error_log>`, `<relevant_code>`, `<expected_behavior>` |
| Code review | `<code>`, `<context>`, `<focus_areas>` |
| Migration | `<old_code>`, `<new_target>`, `<constraints>` |
| PRD generation | `<project_description>`, `<scope>`, `<constraints>`, `<output_format>` |

### PRD generation with XML structure

```
Generate a Product Requirements Document for this project.

<project_description>
[What the project does, who uses it, what problem it solves]
</project_description>

<scope>
In scope: [list]
Out of scope: [list]
</scope>

<constraints>
Tech stack: [list]
Non-negotiables: [security, compliance, performance requirements]
</constraints>

<output_format>
- Problem statement (2–3 sentences)
- MVP feature list (IN / OUT)
- Success criteria (measurable)
- 3–5 user stories
</output_format>

Ask me clarifying questions before writing anything.
```

### CLAUDE.md / instruction file generation with XML

When generating AI instruction files, provide context in tagged sections so the AI distinguishes what to read from what to write:

```
Generate a CLAUDE.md for this project.

<project_facts>
Name: [name]
Stack: [stack]
What it does: [description]
</project_facts>

<existing_conventions>
[Paste any existing README, folder structure, or conventions]
</existing_conventions>

<rules>
- Under 200 lines
- No style rules (linter handles those)
- Must include: NEVER section, Definition of Done, compaction instructions
</rules>
```

**Rule:** Use XML tags when a prompt has 3+ distinct sections. For simple one-liner commands, plain English is faster and equally effective.
