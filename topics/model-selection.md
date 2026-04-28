# Model Selection — Choosing the Right AI for the Task
> Reference for selecting AI models and managing costs in development workflows. Attach when setting up a project's AI tooling or when optimizing an existing workflow.

---

## 1. The Two Modes of Work

The most useful split is not by model family but by the nature of the task:

| Mode | What it means | Best model tier |
|------|---------------|-----------------|
| **Execution** | The output is well-defined. You could verify it with a test, a diff, or your own eyes. | Fast, cheaper models — Claude Sonnet, GPT-5-Codex, Grok-Code-Fast |
| **Reasoning** | The output requires judgment. You cannot easily verify if the answer is correct without thinking hard yourself. | Capable, slower models — Claude Opus, Gemini 2.5 Pro, GPT-5 High/Medium |

**The practical split:** ~85% of daily development tasks are Execution. Reserve Reasoning models for the 15% where the cost of a wrong answer is high and hard to detect.

---

## 2. Decision Guide

### Execution mode — use the faster, cheaper model:
- Writing or refactoring code to a clear specification
- Writing tests for existing functions
- Fixing bugs with a known root cause
- Generating boilerplate, CRUD endpoints, documentation
- Running agentic tasks with well-defined steps

### Reasoning mode — use the more capable model:
- Designing a new system or choosing between architectures
- Evaluating long-term trade-offs (e.g., monolith vs microservices)
- Analyzing security implications across a full flow
- Debugging a multi-system problem with unknown root cause
- Writing a PRD or technical spec from a rough idea
- Anti-sycophancy review of consequential decisions (see prompting.md)

**Signal:** If you could write a test that verifies the output is correct, it's Execution. If the "right answer" requires human judgment to evaluate, it's Reasoning.

---

## 3. Claude Code: Switching Models

In Claude Code:
- Default session model is configurable in settings
- Override per session: `/model` command → select model
- Fast mode (`/fast`): enables faster output for the current session (same model, different configuration)

**Recommended setup:**
- Day-to-day coding: Claude Sonnet 4.6 (speed + cost balance)
- Architecture sessions / important decisions: Claude Opus 4.6

---

## 4. Language Choice: English vs Other Languages

**Always write prompts in English unless there is a specific reason not to.**

Why it matters for cost and quality:
- For many technical tasks, English prompts are shorter and easier for models to follow consistently
- Lower token usage usually means faster responses, lower cost, and more context budget for actual code
- Most coding examples in public datasets are in English, so outputs are often more stable in English-first prompts

**When to use non-English:**
- The output itself is in that language (UI copy, error messages for users)
- You are reasoning about non-English text (e.g., analyzing Polish business rules)
- In these cases: write instructions in English, include the non-English content as data

**Example:**
```
English (instruction): Translate the following error messages to be more user-friendly. 
Keep them in Polish.

Polish (data to process):
- "Błąd 404: Nie znaleziono zasobu"
- "Błąd 500: Wewnętrzny błąd serwera"
```

**Exception — long-context retrieval:** For needle-in-haystack tasks (finding one specific piece of information in a very long document), the language the document is in actually performs better. If your source documents are in Polish, asking in Polish may give better retrieval. But for generation tasks, English wins.

---

## 5. Cost Management

### The main cost drivers
1. **Context size** — more tokens in = higher cost. Manage with `/compact` and focused prompts.
2. **Model tier** — Reasoning-mode models cost 5–10x more per token than Execution-mode models.
3. **Agentic loops** — agents make many calls. A runaway agent can be expensive.

### Practical limits to set
- Set spending limits in your AI provider dashboard (Anthropic Console, OpenAI, etc.)
- Set a monthly budget alert — you want to know before the bill arrives
- Review Claude Code usage: `/context` shows token usage for the current session

### Cost-efficiency patterns
- Prefer precise prompts over exploratory back-and-forth (each exchange costs tokens)
- Use `/compact` before switching to a new sub-task — resets accumulated conversation cost
- Reference files by path (`see src/auth.ts`) instead of pasting large code blocks
- Batch related small changes into one prompt rather than many small prompts
- Use Execution-mode models for 90% of tasks; switch to Reasoning-mode only when necessary

### When Cursor is cheaper than Claude Code
- For autocomplete-heavy work (writing repetitive code), Cursor's inline completion may be more cost-effective
- For one-off architectural discussions, Claude Code (direct API) may be cheaper per session
- Evaluate based on your actual usage pattern — both have flat subscription options

---

## 6. Model Selection by Task Type

| Task | Mode | Notes |
|------|------|-------|
| Write a React component | Execution | Specify behavior clearly |
| Write unit tests for a function | Execution | Provide the function + test cases |
| Fix a failing test | Execution | Paste error + relevant code |
| Refactor a 200-line file | Execution | Provide the file + desired structure |
| Generate CRUD endpoints | Execution | Template-driven, verifiable output |
| Design a new API | Reasoning | Open-ended, requires judgment |
| Choose between SQL and NoSQL | Reasoning | Trade-off analysis |
| Security review of auth flow | Reasoning | Risk assessment requires reasoning |
| Debug a production incident | Reasoning | Unknown root cause, systemic reasoning |
| Evaluate a modernization plan | Reasoning | Strategic decision |

---

## 7. Multi-Model Workflows

For complex tasks, use models in sequence:

**Design → Implement pattern:**
1. Reasoning model: "Design the data model and API surface for [feature]" → get a spec
2. Execution model: "Implement this spec" (paste the spec) → get code
3. Execution model: "Write tests for this implementation" → get tests

**Review → Fix pattern:**
1. Reasoning model: "Review this PR for security and architecture issues" → get findings
2. Execution model: "Fix these specific issues: [paste findings]" → get fixes

**This separation is powerful:** Reasoning models are expensive for generation but excellent for analysis. Execution models are cheap and fast for generation when given clear specs.

**Cross-model validation pattern:**

For decisions with high stakes, run the same problem through models from different providers. Different models have different training data and different blind spots — one model's confident wrong answer is often another model's obvious concern.

```
Workflow:
1. Claude writes the implementation
2. Gemini reviews it (paste code + "review this for correctness and security issues")
3. Bring findings back to Claude to fix

Or reversed:
1. Gemini designs the architecture
2. Claude critiques it ("what are the weaknesses in this approach?")
3. Iterate until both models' concerns are addressed
```

**When cross-model validation is worth the effort:**
- Auth flows and security-sensitive code
- Novel architecture decisions (no established pattern to follow)
- Code that will be hard to change later (database schema, public API surface)
- When one model seems unusually confident about a non-obvious decision

**When it is not worth it:** routine feature implementation, standard CRUD, anything with clear tests that verify correctness.

---

## 8. Benchmarks and Reality

**Treat benchmarks as starting points, not verdicts.**

- Vendors optimize for specific benchmarks — scores can be gamed
- Academic benchmarks don't represent real-world messiness
- Public benchmark results are useful, but they overestimate day-to-day autonomy on messy real projects
- Better signal: your own acceptance tests on representative tasks plus real usage data over a few weeks

**Practical approach:**
1. Use public rankings as a shortlist, not as a final decision
2. Run a short trial on your real task mix (same prompts, same success criteria)
3. For high-stakes workflows, run `promptfoo` evals (see Section 9)

**Buying pattern to avoid:** Switching models every week based on news/benchmarks. Pick one strong Execution-mode model and one strong Reasoning-mode model, use them consistently, update only when a new model clearly outperforms on your specific tasks.

---

## 9. Model Evaluation with Promptfoo

When you need to rigorously compare models for a specific use case (not benchmark scores):

```bash
npm install -g promptfoo
promptfoo init          # creates promptfooconfig.yaml
promptfoo eval          # runs tests against all configured models
```

**Basic config:**
```yaml
providers:
  - id: anthropic:messages:claude-sonnet-4-6
  - id: openai:gpt-5
  - id: google:gemini-2.5-pro

prompts:
  - "Generate a TypeScript component for: {{spec}}"

tests:
  - vars:
      spec: "a button with loading state"
    assert:
      - type: javascript
        value: |
          // Run eslint, vitest, or tsc on the output
          const pass = output.includes('useState');
          return { pass, score: pass ? 1 : 0 };
```

**Key advantages:**
- Tests run in parallel across all providers
- Results are cached — unchanged prompts don't re-cost tokens
- Assertions can be: text equals, `contains`, `is-json`, LLM-as-a-Judge, or custom JS/Python scripts
- Can run `tsc`, `vitest`, `eslint` inside assertions to verify generated code actually compiles/passes

**When promptfoo is worth it:**
- You're building AI integrations (CI/CD pipelines, agents) where prompt quality matters at scale
- You want to validate that a new model handles your stack correctly before switching
- You need to benchmark cost vs quality tradeoffs across providers

**When it's overkill:** Day-to-day feature development. Use promptfoo for infrastructure-level AI decisions, not routine coding tasks.
