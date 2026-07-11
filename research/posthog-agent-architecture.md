# PostHog Agent Architecture: How a 35k-Star Product Company Builds AI Agents

> Read directly from [PostHog/posthog](https://github.com/PostHog/posthog) @ master, July 2026.
> Code paths cited throughout. Companion analysis: how these patterns apply to Dalgo's
> chat-with-data TurnGraph (see `dalgo-core/features/chat-with-data/architecture/approach-2.md`).

---

## Table of Contents

1. [The System, Top to Bottom](#1-the-system-top-to-bottom)
2. [The Agents: Vertical Specialization](#2-the-agents-vertical-specialization)
3. [Inside the Chat Agent's Graph](#3-inside-the-chat-agents-graph)
4. [The Tool Layer: Three Surfaces](#4-the-tool-layer-three-surfaces)
5. [The Model Layer: One Wrapper for Every Call](#5-the-model-layer-one-wrapper-for-every-call)
6. [State: Checkpointing and Compaction](#6-state-checkpointing-and-compaction)
7. [Evals](#7-evals)
8. [How They Use LangChain vs LangGraph](#8-how-they-use-langchain-vs-langgraph)
9. [Their 8 Learnings from 1 Year of Agents](#9-their-8-learnings-from-1-year-of-agents)
10. [AI in Their Development Workflow](#10-ai-in-their-development-workflow)
11. [What Dalgo Should Take from This](#11-what-dalgo-should-take-from-this)

---

## 1. The System, Top to Bottom

Everything lives in `ee/hogai/` (agent platform) plus `posthog/temporal/ai_observability/`
(background agents). Six layers:

```
ENTRY SURFACES     in-app chat · MCP server (Claude Code/Cursor) · REST API · Temporal cron
       │           (route by JOB, not by task decomposition)
       ▼
AGENTS             chat_agent · research_agent · background agents · summarizers
       │           (all built on the same base classes)
       ▼
CORE PLATFORM      AgentLoopGraph / BaseAssistantGraph (LangGraph StateGraph wrapper)
ee/hogai/core/     runner + stream processor · agent modes · plan mode · title generator
       │           (capability enters through registries, never graph surgery)
       ▼
TOOL LAYER         MaxTool (in-app) · MCPTool (external) · generated MCP tools (codegen)
       │
       ▼
MODEL LAYER        MaxChatOpenAI / MaxChatAnthropic — every LLM call goes through here
ee/hogai/llm.py    context injection · billing metering · flex-tier routing
       │
       ▼
STATE & OBS        DjangoCheckpointer (Postgres) · compaction sweep · Braintrust evals
```

---

## 2. The Agents: Vertical Specialization

Their learning #3 says "single loops beat subagents" — yet they run multiple agents. No
contradiction: the rule forbids splitting **one task** across subagents (context fragments at
every hand-off). Multiple agents exist because they own **different jobs**:

| Agent | Location | Trigger | Budget | Context |
|---|---|---|---|---|
| Chat agent ("Max" / PostHog AI) | `ee/hogai/chat_agent/` | user message | seconds, streaming | conversation + project taxonomy |
| Research agent | `ee/hogai/research_agent/` | user request (deep mode) | minutes, autonomous | accumulated findings |
| Trace/eval clustering labelers, eval-report writers | `posthog/temporal/ai_observability/` | Temporal cron | hours, batch | thousands of traces, no conversation |
| Summarizers | `ee/hogai/session_summaries/`, `llm_traces_summaries/` | event/API | one shot | single session or trace |

**The rule that falls out:** specialize *vertically by job* (own trigger, own latency budget,
own context, own state schema and tools) — never *horizontally within a task*. Variation within
the chat product is handled by **modes** (`switch_mode` tool, `core/agent_modes/`), not by more
agents.

Subagents aren't banned outright: `subagent_executor` exists for **context isolation** — when a
sub-task would flood the parent's context with material that compresses well (read 100 traces,
return one paragraph). That's outsourcing a bulky read, not decomposing reasoning.

---

## 3. Inside the Chat Agent's Graph

Simplified from `ee/hogai/chat_agent/graph.py`. The graph handles plumbing; the intelligence is
one loop where the model picks tools until it decides it's done.

```
START
  ├─► title_generator            (parallel branch — names the session)
  ├─► memory onboarding subgraph (first run only, interrupt-driven Q&A)
  ├─► slash-command handler      (bypasses the loop entirely)
  ▼
┌───────────────── THE AGENT LOOP ─────────────────┐
│  root (model node)  ⇄  root_tools (MaxTool exec) │   ← merged into the parent
│  loops until the model answers                   │     graph, NOT a subgraph
└──────────────────────────────────────────────────┘
  │ (may hand off to a specialized pipeline, control returns)
  ├─► insights pipelines: trends/funnels/retention/SQL
  │      query_planner → schema_generator → query_executor
  ├─► taxonomy / RAG (agentic lookup of events & properties)
  ▼
memory collector (post-turn) ─► END
```

Notable details:

- **Graphs are assembled fluently** — `AssistantGraph(team, user).add_root().add_memory_onboarding()…`
  — so different deployments compose different topologies from the same nodes.
- **A warning in their code**, directly relevant to Dalgo's TurnGraph G2 slice:
  > "Subgraphs incorrectly merge messages, so please don't use them here."
  They merge the loop's nodes into the parent graph instead of mounting a compiled subgraph.
  Dalgo mounts the compiled agent AS a subgraph sharing the `messages` channel — it passes our
  runner tests, but worth a regression test asserting messages aren't duplicated across a turn.
- **Node-path tracking**: every node knows its position in nested graphs (used for tracing).

---

## 4. The Tool Layer: Three Surfaces

One capability, defined once, exposed on up to three surfaces.

### 4.1 `MaxTool` (`ee/hogai/tool.py`) — the in-app agent's tools

Subclass of LangChain `BaseTool` with heavy machinery:

- **Dual-channel returns** — `response_format = "content_and_artifact"`, always. Tools return
  `(content, artifact)`: content is a compact string for the model; artifact becomes a
  `ui_payload` the frontend renders as real UI (insight, table, form). The model never re-echoes
  tabular data → fewer tokens, no transcription errors, smaller checkpoints.
- **HITL approval on the tool base, not the graph** — override `is_dangerous_operation()` and
  `format_dangerous_operation_preview()` (async, can query the DB for a rich preview). Triggers
  a LangGraph `interrupt()` with a generic `ApprovalRequest` payload; state checkpoints, so
  approval survives disconnects for days. Every future mutating tool inherits approval for free.
  One generic frontend component renders all approval requests.
- **Client-side execution handoff** — `ClientToolCallRequest`: a tool can pause and hand
  execution to the browser; the frontend runs a registered handler and resumes the graph.
- **Tools advertise their own relevance** — `context_prompt_template` injects context into the
  root prompt (e.g. "the current filters the user is seeing are: {current_filters}") to steer
  *when* the model reaches for the tool.
- **Governance**: per-tool `billable` flag, two-level RBAC (resource + object), typed errors
  (`MaxToolRetryableError` vs fatal) so the loop knows whether to self-correct.

The tool census reads like a coding agent transplanted into analytics: `todo_write` (their
learning #4), `subagent_executor`, `switch_mode`, `task`, `manage_memories`, plus domain tools
(`execute_sql`, `create_insight`, `upsert_dashboard`, `read_taxonomy`, replay tools) — and
`call_mcp_server`: the agent is an MCP **client** too.

### 4.2 `MCPTool` (`ee/hogai/mcp_tool.py`) — the same capabilities for external agents

Deliberately thin parallel interface: no LangChain state, no artifacts. Validated Pydantic args
in, string out; team/user bound at construction from API auth. Tools self-register into a
singleton registry via decorator, each declaring required **API scopes**.

### 4.3 Generated MCP tools — codegen from the REST API

Django serializer → OpenAPI → generated TypeScript tool handlers with Zod schemas. Products opt
endpoints in via `products/<name>/mcp/tools.yaml`; regenerate with `hogli build:openapi`.

**The consequence:** API documentation quality now IS agent tool quality. Serializer `help_text`
flows into the Zod `.describe()` the agent reads — "missing descriptions = agents guessing at
parameters." Their design rule: MCP tools are **atomic capabilities** (list/get/create/update/
delete), not workflows — composition is the agent's job.

---

## 5. The Model Layer: One Wrapper for Every Call

`ee/hogai/llm.py`: `MaxChatOpenAI` / `MaxChatAnthropic` subclass the LangChain provider clients.
Every LLM call in the platform goes through them:

- **Auto-injected context** appended to every system message: project + org name, user identity,
  project timezone and current datetime, plus domain rules (URL formats, person-on-events query
  semantics) via Mustache conditionals. No prompt anywhere has to remember who the user is.
- **Billing metering** — Prometheus counter for generations where billing was skipped
  (e.g. impersonation); pairs with the per-tool `billable` flag.
- **Infra policy in one place** — routing eligible models to OpenAI's flex processing tier,
  proxy bypass for the Anthropic client.

Prompt conventions are codified in `ee/hogai/PROMPTING_GUIDE.md`: flat XML sections
(`<agent_info>`, `<instructions>`, `<constraints>`, `<examples>`), Mustache templating for
dynamic content. Conventions, not vibes.

---

## 6. State: Checkpointing and Compaction

`ee/hogai/django_checkpoint/` — a custom LangGraph checkpointer persisting agent state into
their own Postgres via the Django ORM (one `global_checkpointer`), so conversations resume
server-side across requests. Two problems, solved separately:

1. **The prompt problem** (what the model sees): handled at prompt-assembly time
   (history trimming, old-tool-result clearing).
2. **The storage problem** (what Postgres keeps): LangGraph writes a checkpoint per super-step —
   the full lineage. Even with perfect prompt trimming, storage grows unboundedly.
   Their fix: `compaction.py`, an offline sweep:
   - Conversation eligible after **7 idle days** (measured: ~99.6% of resumes happen within a week).
   - Collapse the thread to **its tip checkpoint only**; delete older checkpoints, superseded
     blobs, stale writes. Works because every channel persists a *complete* value per version —
     documented invariant, with a warning that adopting a LangGraph `DeltaChannel` silently breaks it.
   - Safety gates: only `IDLE` conversations, **never pending-approval threads** (compaction
     nulls the tip's parent, which would drop the pending interrupt).
   - Gradual rollout via a team-id ceiling constant.

---

## 7. Evals

`ee/hogai/eval/` — their README leads with: evals verify prompt performance, spot regressions,
compare models. Platform: **Braintrust** (proprietary SaaS; the SDKs, autoevals scorers, and
proxy are open source). They chose it over LangSmith despite building on LangChain.
`braintrust-langchain` hooks LangChain callbacks so every call in a LangGraph run is traced
automatically.

- **CI evals** (`eval/ci/`): `pytest ee/hogai/eval/ci` runs the *real compiled assistant graph*
  against curated cases; results upload as Braintrust experiments.
- **Offline evals** (`eval/offline/`): dataset-driven. Datasets curated **in PostHog's own
  AI-evals product** (dogfooding), executed via Braintrust `EvalCase`/`Score`.
- **Custom scorers** (`eval/scorers/`): e.g. `SQLSyntaxCorrectness` and `SQLSemanticsCorrectness`
  (LLM-as-judge: does the generated SQL answer the same question as the golden SQL).
- **The loop**: interesting production turns → dataset items → scored experiment per change.

They also ship a warehouse connector (`products/warehouse_sources/.../braintrust/`) pulling
Braintrust data INTO PostHog for analysis.

---

## 8. How They Use LangChain vs LangGraph

From `pyproject.toml`: `langchain` ~0.3, `langgraph` pinned `0.4.10` (old — reads like frozen
legacy infra), plus raw `openai` and `anthropic` SDKs.

| Library | Role | How it's contained |
|---|---|---|
| LangChain | Provider clients, message types, prompt templates. NOT chains/agents. | Everything goes through their `MaxChat*` subclasses |
| LangGraph | State machine + persistence protocol (StateGraph, interrupts, checkpointing) | Wrapped in `BaseAssistantGraph`; own checkpointer, compaction, streaming dispatcher, node base class |

Each library used for its narrowest durable value; all business logic behind their own
abstractions so the framework could in principle be swapped underneath. Their public
retrospective calls LangGraph-style graph orchestration a mistake for the interactive agent —
what survived is *graph for plumbing, loop for thinking*.

---

## 9. Their 8 Learnings from 1 Year of Agents

Source: [8 learnings from 1 year of agents](https://posthog.com/blog/8-learnings-from-1-year-of-agents-posthog-ai)
(see also: [Why we killed our AI product assistant](https://posthog.com/blog/why-we-killed-our-ai-product-assistant))

1. **Model improvements are disruptive** — capability jumps invalidate architecture; scaffolding
   for planning/decomposing/retrying keeps becoming obsolete.
2. **Agents beat workflows** — a simple self-correcting loop outperforms graph orchestration for
   free-form work.
3. **Single loops beat subagents** — unified context beats fragmenting a task across specialists.
4. **To-do tools provide structure** — a simple `todo_write` tool keeps long tasks on track.
5. **Broader context improves performance** — front-load knowledge about product and user
   (the pattern behind Dalgo's M5 table cards).
6. **Transparency builds trust** — show reasoning steps, tool calls, and failures.
7. **Avoid framework lock-in** — direct LLM calls adapt; abstraction layers rot as models improve.
8. **Evals need real-world validation** — production traces beat synthetic test sets.

---

## 10. AI in Their Development Workflow

The repo root doubles as a manual for running an AI-assisted engineering org:

- **`AI_POLICY.md`** — "You own what you submit." Prove it works end-to-end (screenshots for
  frontend, test strategy for backend); disclose AI usage; no AI-generated issues or unsolicited
  AI PR reviews; two strikes → block. Explicitly pro-AI, anti-slop.
- **`.claude/`** — 8 custom subagents (code-reviewer, systematic-debugger, test-writer,
  prompt-engineer…), 11 rules files for footgun areas, SessionStart hooks bootstrapping the dev env.
- **`.agents/skills/`** — ~50 agent skills (`optimizing-clickhouse-and-hogql-queries`,
  `fixing-flaky-tests`, `implementing-mcp-tools`, `writing-skills`…): institutional knowledge
  encoded for agents, shared across harnesses (Claude Code, Cursor, Zed, Pi, their own).
- **PR template** with a mandatory "🤖 Agent context" section: autonomy declared
  ("Human-driven (agent-assisted)" vs "Fully autonomous"), skills invoked listed, no claiming
  untested work, draft by default, never self-merge, human DRI assigned.
- **Agents as devex users** — `hogli devex:feedback` lets agents file tooling complaints,
  tagged as agent-sent.
- **AI in CI**: Greptile PR review (reads `AGENTS.md` as context), Inkeep agent auto-updating docs.

---

## 11. What Dalgo Should Take from This

Prioritized against the chat-with-data TurnGraph (approach-2). Validated already: shallow graph
around one loop, fail-open helpers, diagram-as-test, audit table, visible tool chips.

1. **Build the eval loop now** (biggest gap). Pytest harness over the real TurnGraph; scorers for
   SQL syntax + semantics + answer-contract compliance; golden sets promoted from
   `ChatWithDataTurnAudit` rows. Langfuse datasets/experiments ≈ their Braintrust loop without a
   new vendor. Gate M5 with it: E10 measures efficiency, not accuracy — cards could make the
   agent faster AND worse.
2. **Fix history growth at the checkpointer, not just middleware.** Land M0 P1 (trim no-op), then
   a compaction sweep collapsing idle threads to their tip (same tip-is-complete invariant our
   history-replay endpoint already relies on). Skip pending-approval threads.
3. **HITL approval on the tool base class, not graph nodes.** `is_dangerous_operation()` +
   preview + generic `ApprovalRequest` interrupt. Policy travels with the capability; rule 6 stays
   intact. Keep pre-`interrupt()` code side-effect-free (LangGraph re-runs the tool on resume).
4. **Split tool returns into content vs artifact.** `execute_sql`: capped preview + row count for
   the model; full result as artifact for `ResultTable`. Caution: `validate_node` reads results
   from messages — adjust it or it gets lobotomized. Pin with a test.
5. **One LLM wrapper for all five brains** (`DalgoChat`): org/dialect/timezone context + usage
   counters, like `MaxChatOpenAI`.
6. **Re-examine the Phase-3 decomposer** against learnings #2/#3: null hypothesis is
   `todo_write` + richer context in the single loop. Let evals judge.
7. **Cheap wins**: tool `context_prompt_template` when the registry grows; a short
   PROMPTING_GUIDE; card prose quality = prompt engineering (their "missing help_text = agents
   guessing"); AGENTS.md + agent-context PR section, proportionate to team size.
8. **Don't copy their scale of abstraction** (registries, codegen pipelines, generic graph
   bases) — earns its keep at ~70 products, not at ours. Thin-nodes-brains-in-`calls/` is the
   better discipline for now.
9. **Subgraph caution**: their code comment says compiled subgraphs "incorrectly merge messages";
   we mount the agent as a subgraph. Add a regression test asserting no message duplication
   across a parent-node + subgraph turn.

**The vertical-agent litmus test** for any proposed agent #3: does it have its own trigger,
budget, and context? (The M5 enrichment agent does — scheduled, slow-and-thorough, warehouse
metadata.) If instead "the chat agent could delegate part of a question to it," that's horizontal
decomposition — the thing learning #3 and rule 6 exist to stop.

---

*Key code references: `ee/hogai/tool.py` (MaxTool), `ee/hogai/mcp_tool.py`, `ee/hogai/llm.py`,
`ee/hogai/core/base.py`, `ee/hogai/chat_agent/graph.py`, `ee/hogai/django_checkpoint/compaction.py`,
`ee/hogai/eval/README.md`, `ee/hogai/PROMPTING_GUIDE.md`, `AI_POLICY.md`,
`.github/pull_request_template.md`.*
