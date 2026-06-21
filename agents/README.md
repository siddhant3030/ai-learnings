# Building Production Agents for Dalgo

How to design, build, secure, measure, and improve a runtime AI agent inside the Dalgo product — grounded in the actual stack (`DDP_backend`, `webapp_v2`, Prefect/Airbyte/dbt), the existing Dalgo MCP surface, the NGO-user constraints, and the prior research in this repo.

**Guiding principle (the team's):** intelligence belongs at **build-time**; the **runtime** agent stays simple, deterministic, bounded, cheap, and narrow. Complex agentic loops in the hot path mean unpredictable cost, latency, and results for non-technical NGO users.

| # | Guide | Answers |
|---|-------|---------|
| 1 | [Use Cases](01-use-cases.md) | What a Dalgo agent can do, prioritized for NGO users; what NOT to build |
| 2 | [Production Architecture](02-production-architecture.md) | How to build it — where it lives, model/tool layers, build-vs-buy |
| 3 | [Guardrails & Safety](03-guardrails-and-safety.md) | Org isolation, PII, prompt-injection-from-warehouse, write safety |
| 4 | [Evals & Quality](04-evals-and-quality.md) | Measuring a data agent — eval sets, grading, evals-as-discovery |
| 5 | [Agent Lifecycle](05-agent-lifecycle.md) | Operating & improving the agent over time; Crawl/Walk/Run |

## The build path, in one screen

1. **Build first:** the **pipeline failure explainer** ("why did my sync fail?") — read-only, bounded, cheap, relieves the support team. *Not* data Q&A first — it's highest-impact but highest-risk (PII, accuracy, injection); ship it once the guardrail muscle is proven. (→ guide 1)
2. **Where it lives:** `DDP_backend` as `ddpui/core/agent/`, in-process, streaming over the existing websockets. **"One core, two surfaces"** — the runtime agent and the Dalgo MCP server both wrap the same `ddpui/core/*functions.py`. **Direct functions at runtime, no MCP** (Dalgo owns both sides; MCP is pure indirection + token tax). (→ guide 2)
3. **Build vs buy:** a ~150-line loop on the Anthropic SDK directly. No LangGraph/CrewAI. Model tier: **Haiku 4.5** default for the bounded runtime agent, **Sonnet 4.6** when reasoning depth is needed (Opus 4.8 is for dev-time, not the runtime hot path). (→ guide 2)
4. **First guardrail:** wrap every agent tool in the existing org-scoped, `@has_permission`-gated service call with **org pinned from the token** — the agent inherits the human's exact cage, no tool ever takes an `org_id`. Remove all `delete` tools from the autonomous path. (→ guide 3)
5. **First eval set:** 20–50 cases against a **fixed two-org test-warehouse fixture**; grade on **execution accuracy** (run the generated SQL, match the returned *number*, not the SQL string). Calibrate any LLM judge against ~50 human labels before trusting it. (→ guide 4)
6. **Lifecycle:** Observe → Evaluate → Improve → Ship (eval-gated, canary to one org) → repeat. Highest-ROI lever is **tool descriptions + context, not model upgrades**. (→ guide 5)

## The risk that defines this whole effort

**Warehouse rows are untrusted input to the model.** Airbyte ingests free-text from NGO source systems (forms, sheets, KoboToolbox), so a beneficiary "name" field can itself be a prompt-injection payload. The agent reads it with no privilege boundary — the lethal-trifecta leg sourced from your own data. "Read-only" does **not** make an agent safe. (→ guide 3)

## Notes & things to verify before building

- An existing `LIMIT_ROWS_TO_SEND_TO_LLM=500` constant and an external batch llm-service (Celery) were found in `DDP_backend` — there's already an LLM path; confirm what it does before adding the runtime agent.
- Confirm the Dalgo MCP server wraps `ddpui/core/*functions.py` rather than reimplementing logic. If it reimplements, collapsing onto a shared core is the prerequisite for "one core, two surfaces."
- Model IDs/pricing in guide 2 were left as model *classes*; current models are Opus 4.8 / Sonnet 4.6 / Haiku 4.5 (+ Fable 5) — confirm exact IDs and pricing against current Anthropic docs.

## Builds on the prior research here

`research/agent-design.md`, `research/evals.md`, `research/ai-security-guardrails.md`, `research/memory-systems.md`, `research/ai-observability.md`, `research/context-engineering.md`, the `mcp/` folder, and `sentry/` (PII scrubbing). This folder applies all of it to one concrete product: a Dalgo runtime agent.
