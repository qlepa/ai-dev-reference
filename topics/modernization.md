# Modernization — Legacy Code Upgrade Strategies
> Reference for planning and executing safe modernization of legacy codebases using AI and automated tools. Attach when working on a legacy project or when planning a significant refactor.

---

## 1. The Core Rule: Safety Before Speed

**Never modernize blindly.** The order always is:

```
Understand → Document → Safety net → Modernize
```

Skipping any step makes regression risk unacceptably high. AI-assisted modernization is fast — which makes skipping steps even more dangerous than manual refactoring.

---

## 2. Two Modernization Approaches

### Approach A: AI-Assisted Rewrite

The AI understands the intent of old code and rewrites it using modern patterns.

**Best for:**
- Logic that is well-understood but poorly expressed
- Code with good characterization tests (the tests define expected behavior)
- UI components: old class components → modern functional components
- Old callback patterns → async/await
- Framework-specific migrations (React Router v5 → v6, Vue 2 → Vue 3)

**Risk:** The AI may silently change behavior that was undocumented or intentional. Mitigations:
- Write characterization tests first (see testing.md Section 5)
- Review AI output line by line on first pass
- Keep the old code in git — revert is your safety net

**Prompt template for AI rewrite:**
```
I need to modernize [file path] from [old pattern] to [new pattern].

Context:
- [What this code does]
- [Why we are modernizing it]
- [Any behavior that must be preserved exactly]

Constraints:
- Do NOT change business logic — only the pattern/syntax
- Keep the same function signatures unless I explicitly say otherwise
- If you are unsure whether something is an intentional pattern or a bug, ask before changing it

Proceed file by file. Wait for my review after each file before moving to the next.
```

### Approach B: Codemods (Automated AST Transforms)

Codemods are programs that parse and transform code at the AST (Abstract Syntax Tree) level — they are deterministic, not probabilistic.

**Best for:**
- Mechanical transformations: renaming, restructuring, syntax changes
- Large-scale changes (hundreds of files) where AI would be too slow or inconsistent
- TypeScript migration from JavaScript
- Framework major version upgrades with known transformation patterns

**Tools:**
| Tool | Language | Use case |
|------|----------|----------|
| `jscodeshift` | JS/TS | Custom AST transforms, community codemods |
| `ts-migrate` | JS → TS | Bulk migration to TypeScript |
| `@next/codemod` | TypeScript | Next.js version upgrades |
| `react-codemod` | TypeScript | React API migrations |
| `OpenRewrite` | Java/Kotlin | JVM ecosystem modernization |

**How AI helps with codemods:**
- Generate the codemod script itself: "Write a jscodeshift codemod that replaces all `require()` with `import`"
- Review codemod output: "Here are 10 sample outputs of the codemod — are they correct?"
- Handle edge cases the codemod misses (AI finishes what the codemod couldn't)

### The Hybrid Approach (Recommended for Large Migrations)

1. **Codemod handles 80%** — run automated transforms on all files
2. **AI handles remaining 20%** — fix edge cases, add types, resolve ambiguities
3. **Human reviews the diff** — spot-check, approve, merge

**Real-world benchmark (Slack's migration):** This hybrid approach achieved ~80% automation success rate on a large JS → TS migration. The remaining 20% required human + AI judgment for complex cases.

---

## 3. Classification Framework

For each modernization item, classify it before starting:

| Category | Description | Who does it |
|----------|-------------|-------------|
| **AI-rewrite** | Well-understood logic, testable, clear inputs/outputs | AI with supervision |
| **Codemod** | Mechanical transformation, syntax/structure only | Automated tool + AI verification |
| **Manual-only** | Security-critical, complex business logic, high risk, unclear behavior | Human, AI advises only |

**Manual-only triggers:** If any of these are true, it's Manual-only:
- Touches payment, auth, or data deletion logic
- Has zero tests and behavior is unclear from code alone
- The team member who wrote it is unavailable to clarify
- Failure would cause data loss or security breach

---

## 4. Migration Guides as AI Context

When migrating between framework versions, attach official migration guides directly to AI context:

```
I am migrating from [Framework A vX] to [Framework B vY].

Here is the official migration guide: [paste relevant sections]

Now look at [file path] and rewrite it following the migration guide. 
Flag any patterns the guide doesn't cover — don't guess, ask me.
```

**Why this works:** Migration guides encode the exact transformation rules. When the AI has this context, it makes fewer errors and asks better questions about edge cases.

**Sources for migration guides:**
- Official framework docs (always prefer these)
- GitHub release notes and CHANGELOG files
- `@[framework]/codemod` packages often include a migration guide in their README

---

## 5. Domain-Driven Design (DDD) with AI

DDD provides a framework for understanding complex legacy systems before changing them. Use AI as a thinking partner, not an authority.

### Event Storming with AI

Event Storming maps business processes using domain events. Run AI-assisted Event Storming when:
- You are onboarding to a large legacy codebase
- You need to break a monolith into services
- The business logic is embedded in code with no documentation

**Prompt to start:**
```
I am going to describe a business domain to you. As I describe it, help me identify:
1. Domain Events (things that happened — past tense): "OrderPlaced", "PaymentFailed"
2. Commands (actions that trigger events): "PlaceOrder", "ProcessPayment"
3. Aggregates (entities that own state): "Order", "Customer", "Payment"
4. Bounded Contexts (areas of the system with their own language): "Ordering", "Billing", "Fulfillment"

Ask me clarifying questions as we go. Do not invent business rules — only reflect back what I describe.

Here is what I know about the system: [describe the domain]
```

### Bounded Contexts → Code Structure

Once you have identified Bounded Contexts, they map to code structure:

```
src/
  ordering/      ← Ordering context
    domain/      ← Entities, value objects, domain events
    application/ ← Use cases, command handlers
    infra/       ← Repository implementations, API adapters
  billing/       ← Billing context (separate team language)
  fulfillment/   ← Fulfillment context
```

Each context has its own:
- Ubiquitous language (the "Order" in Ordering context is different from the "Order" in Billing)
- Data models (no shared database tables between contexts — they sync via events)
- API surface (context communicates through well-defined interfaces)

### Subdomain Classification

| Subdomain | Definition | Modernization priority |
|-----------|------------|----------------------|
| **Core** | Your competitive advantage, what makes the business unique | Highest — invest here |
| **Supporting** | Needed for core to function, but not differentiating | Medium — modernize as needed |
| **Generic** | Solved problems (auth, billing, email) | Low — use off-the-shelf solutions |

**Practical application:** If auth is a Generic subdomain, don't build it — use Supabase, Auth0, or Clerk. Spend modernization effort on Core subdomain code instead.

### AI Risks in DDD Work

- **Hallucinated business rules:** AI may confidently describe business rules that don't exist. Always verify against actual stakeholders or existing code.
- **Academic solutions:** AI is trained on DDD textbooks and may propose patterns that are theoretically correct but overly complex for your scale. Push back with "what is the simplest version of this that solves the problem?"
- **False ubiquity:** AI may assume the same term means the same thing across all contexts. Make the Bounded Context explicit in every prompt.

---

## 6. Safe Modernization Checklist

Before starting any modernization work:

- [ ] Characterization tests exist for the code being changed (write them if not)
- [ ] The item is classified: AI-rewrite / Codemod / Manual-only
- [ ] A git branch exists for this work (never modernize directly on main)
- [ ] The team knows this area of code is being touched (no parallel changes)
- [ ] Rollback plan exists (usually: revert the branch, or feature flag)
- [ ] Success criteria defined: what does "done" look like?

After each modernization step:
- [ ] Characterization tests still pass
- [ ] Linter and type checker pass
- [ ] The behavior is manually verified for the most critical path
- [ ] Changes are committed (atomic commit per logical change)
