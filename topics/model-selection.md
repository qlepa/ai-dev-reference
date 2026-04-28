# Model Selection — Choosing the Right AI for the Task
> Reference for selecting AI models and managing costs in development workflows. Attach when setting up a project's AI tooling or when optimizing an existing workflow.

---

## 1. The Two Categories of Models

Not all AI models are equal. The key distinction for development work:

| Category | Best for | Examples (as of mid-2025) |
|----------|----------|--------------------------|
| **Coder models** | Code generation, autocomplete, refactoring, test writing | Claude Sonnet 4.6, GPT-5-Codex, Grok-Code-Fast |
| **Architect models** | System design, complex reasoning, architecture decisions, tradeoff analysis | Gemini 2.5 Pro, GPT-5 High/Medium, Claude Opus 4.6 |

**Rule:** Use Coder models for the majority of work (faster, cheaper). Escalate to Architect models only for decisions that require deep reasoning.

---

## 2. Decision Matrix

### Use a Coder model when:
- Writing or refactoring code to a clear specification
- Writing tests for existing functions
- Fixing bugs with a known root cause
- Generating boilerplate from a pattern
- Running agentic tasks with clear steps
- Code review of specific files

### Use an Architect model when:
- Designing a new system or choosing between architectures
- Evaluating long-term trade-offs (e.g., monolith vs microservices)
- Analyzing security implications of a design
- Debugging a complex, multi-system problem with unclear root cause
- Creating a PRD or technical spec from scratch
- Anti-sycophancy review of important decisions (see prompting.md)

**Cost signal:** If a task can be specified clearly enough that the output is verifiable, use a Coder model. If the task requires judgment that you cannot easily verify, use an Architect model.

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
- English requires 33–40% fewer tokens than Polish, French, German, Spanish for equivalent content
- Fewer tokens = faster responses + lower cost + more context budget for actual code
- Most models are trained primarily on English — quality degrades measurably in non-English prompts for technical tasks

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
2. **Model tier** — Architect models cost 5–10x more per token than Coder models.
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
- Use the Coder model for 90% of tasks; switch to Architect only when necessary

### When Cursor is cheaper than Claude Code
- For autocomplete-heavy work (writing repetitive code), Cursor's inline completion may be more cost-effective
- For one-off architectural discussions, Claude Code (direct API) may be cheaper per session
- Evaluate based on your actual usage pattern — both have flat subscription options

---

## 6. Model Selection by Task Type

| Task | Recommended model tier | Notes |
|------|----------------------|-------|
| Write a React component | Coder | Specify behavior clearly |
| Write unit tests for a function | Coder | Provide the function + test cases |
| Fix a failing test | Coder | Paste error + relevant code |
| Refactor a 200-line file | Coder | Provide the file + desired structure |
| Design a new API | Architect | Open-ended, requires judgment |
| Choose between SQL and NoSQL | Architect | Trade-off analysis |
| Security review of auth flow | Architect | Risk assessment requires reasoning |
| Debug a production incident | Architect | Unknown root cause, systemic reasoning |
| Generate CRUD endpoints | Coder | Template-driven, verifiable output |
| Evaluate a modernization plan | Architect | Strategic decision |

---

## 7. Multi-Model Workflows

For complex tasks, use models in sequence:

**Design → Implement pattern:**
1. Architect model: "Design the data model and API surface for [feature]" → get a spec
2. Coder model: "Implement this spec" (paste the spec) → get code
3. Coder model: "Write tests for this implementation" → get tests

**Review → Fix pattern:**
1. Architect model: "Review this PR for security and architecture issues" → get findings
2. Coder model: "Fix these specific issues: [paste findings]" → get fixes

**This separation is powerful:** Architect models are expensive for generation but excellent for analysis. Coder models are cheap and fast for generation when given clear specs.

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
- **SWE-Lancer finding:** Even the best models solve only ~20% of real Upwork software tasks autonomously — benchmarks are far more optimistic than practice
- **Better signal:** OpenRouter model rankings, which reflect actual user queries across millions of real tasks, not curated test sets

**Practical approach:**
1. Use OpenRouter rankings as a discovery tool for new models
2. Run a "vibe check" — test the model on YOUR actual task type for 30 minutes
3. If high stakes, run `promptfoo` evals (see Section 9)

**Buying pattern to avoid:** Switching models every week based on news/benchmarks. Pick one strong Coder and one strong Architect, use them consistently, update only when a new model clearly outperforms on your specific tasks.

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
