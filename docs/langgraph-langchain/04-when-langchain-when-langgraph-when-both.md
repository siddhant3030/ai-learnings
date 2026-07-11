# When LangChain, When LangGraph, When Both, When Neither

A decision guide for a Python data-platform team building a production chat-with-data feature (NL-to-SQL + analytics agent). Written against the LangChain/LangGraph 1.x era (both hit 1.0 in October 2025; state of the ecosystem as of mid-2026).

---

## TL;DR

- **"LangChain vs LangGraph" is a false dichotomy in 1.x.** They are layers of one stack, not competitors. `langchain.agents.create_agent` *is* a LangGraph graph under the hood. The real question is: **"do I stay at the high-level agent API + middleware, or do I drop down to the graph API?"**
- **LangChain alone** is enough when your feature is "a model calling tools in a loop" plus cross-cutting concerns (HITL approval, summarization, PII redaction) that middleware covers.
- **Drop to LangGraph** when the *shape of the loop itself* is wrong for you: deterministic multi-step pipelines, custom state beyond a message list, fan-out/map-reduce, subgraphs and multi-agent topologies, bounded retry loops with explicit routing, durable long-running workflows.
- **Both together** is the normal production pattern, not an exception: a `StateGraph` pipeline whose nodes call LangChain chat models/tools, or a `create_agent` embedded as a node/subgraph inside a bigger graph.
- **Neither** is the honest answer for single-LLM-call features and simple RAG — provider SDK + plain Python is easier to debug and cheaper to maintain, and a significant slice of the practitioner community has migrated exactly that way.
- **For chat-with-data specifically:** build the core as a **fixed LangGraph `StateGraph` pipeline** (intent → schema retrieval → SQL generation → validation → execution → bounded self-correction → chart/summary), with LangChain chat models and structured output *inside* nodes. Do not build it as a free-running ReAct agent.

---

## 1. The layering, precisely

In the 1.x era, LangChain Inc. ships a three-tier stack, and their own framing is explicit: *"LangGraph is the orchestration runtime; LangChain is the agent framework"* and *"LangChain agents are built on LangGraph, so you're not locked in. Start with high-level APIs and seamlessly drop down to LangGraph when you need more control."*

```
┌────────────────────────────────────────────────────────────┐
│  Prebuilt patterns / harnesses                             │
│  deepagents (create_deep_agent), langgraph-supervisor, …   │
│  = create_agent + a bundled middleware stack               │
├────────────────────────────────────────────────────────────┤
│  langchain (1.x)  —  the high-level agent framework        │
│  create_agent(): standardized tool-calling loop            │
│  middleware=[]: hooks into the loop (HITL, summarization,  │
│  PII redaction, retries, guardrails)                       │
│  response_format=: structured output                       │
├────────────────────────────────────────────────────────────┤
│  langgraph (1.x)  —  the low-level orchestration runtime   │
│  StateGraph, nodes/edges, custom state schemas, Send API   │
│  (fan-out), subgraphs, interrupt()/Command (HITL),         │
│  checkpointers (durable execution, threads), streaming     │
├────────────────────────────────────────────────────────────┤
│  langchain-core (1.x)  —  the foundation                   │
│  provider-agnostic chat models (init_chat_model /          │
│  ChatOpenAI / ChatAnthropic / …), messages & content       │
│  blocks, @tool, runnables, output parsers                  │
└────────────────────────────────────────────────────────────┘
```

Key consequences of this layering:

1. **`create_agent` returns a compiled LangGraph graph.** It accepts a `thread_id`, works with checkpointers, streams like any graph, and can be added as a node in a larger `StateGraph`. There is no migration cliff between the layers — dropping down is *"the intended path, not a workaround."*
2. **`langchain-core` is useful to both layers and to neither-framework code.** You can use `init_chat_model` and `@tool` in a plain Python script with no agent loop at all. Model abstraction does not require buying the orchestration story.
3. **Middleware is the 1.x extension primitive for the high-level layer.** Instead of subclassing or monkey-patching the agent loop, you attach composable hooks (before/after model call, after tool execution, context management). Official docs frame it as: each middleware handles one concern and composes freely.
4. **The old "LangChain = chains, LCEL everywhere" identity is gone.** 1.x LangChain is deliberately narrower: model/tool abstractions + `create_agent` + middleware. Orchestration lives in LangGraph. If your mental model of LangChain is 0.x `SequentialChain`/`AgentExecutor` sprawl, discard it — that sprawl moved to `langchain-classic` and is not the recommended surface.

So the decision is not "which framework" — it's **"which floor of the same building do I work on."**

---

## 2. The decision framework

### Flowchart

```
Is the feature one (or a couple of) LLM calls with fixed
inputs/outputs — classify, extract, summarize, simple RAG?
        │
        ├─ YES ──► NEITHER. Provider SDK (or litellm) + plain
        │          Python + Pydantic. Add instructor if you
        │          want structured output sugar.
        │
        └─ NO
            │
Do you need multi-provider model abstraction, a tool-calling
loop, or structured output — but the control flow is just
"model picks tools until done"?
        │
        ├─ YES ──► LANGCHAIN. create_agent + middleware.
        │          Stay here as long as middleware hooks are
        │          enough customization.
        │
        └─ NO / OUTGROWN IT
            │
Do you need any of:
  • deterministic stage order (pipeline, not agent-decided)
  • state beyond a message list (typed fields, counters, artifacts)
  • bounded loops with explicit routing (retry ≤ N, then escalate)
  • fan-out / map-reduce over items (Send API)
  • multiple agents / subgraphs with different roles
  • pause-and-resume HITL at arbitrary points (interrupt())
  • durable, long-running, resumable executions
        │
        ├─ YES ──► LANGGRAPH StateGraph. Use LangChain models/
        │          tools inside nodes (= BOTH). Embed
        │          create_agent as a node where a sub-task is
        │          genuinely open-ended.
        │
        └─ YES, but the team is anti-framework / the surface
           is tiny ──► NEITHER, honestly: a Python function
           pipeline + provider SDK + your own retry loop and
           state dataclass. Valid; you give up checkpointing,
           streaming plumbing, and interrupt/resume for free.
```

### One-line heuristics

| Signal | Reach for |
|---|---|
| "One prompt in, one answer out" | Neither |
| "Model should decide which tools to call and when" | LangChain `create_agent` |
| "I need approval / redaction / summarization *around* the loop" | LangChain middleware |
| "I need to intercept state *mid*-execution or change the loop's shape" | LangGraph |
| "The stages are known in advance; only content varies" | LangGraph `StateGraph` (fixed pipeline) |
| "Retry at most twice, then ask the user" | LangGraph conditional edges + a counter in state |
| "Run this per-table / per-question in parallel and merge" | LangGraph `Send` |
| "An open-ended researcher inside a fixed pipeline" | Both: `create_agent` as a node |
| "It must survive a pod restart mid-run / resume next week" | LangGraph checkpointer |

---

## 3. Where LangChain alone is enough

Use plain `langchain` (+ `langchain-core`) when the default loop shape fits and your customization is cross-cutting rather than structural.

**a) Model abstraction and portability.** `init_chat_model("anthropic:claude-...")` vs `"openai:gpt-..."` behind one interface, standardized content blocks, one place to swap providers or A/B models. For a data platform serving multiple orgs with different model preferences, this alone can justify `langchain-core` — even if you use nothing else from the stack.

**b) Tools and structured output.** `@tool` with typed args and docstring-derived schemas; `response_format=` / `with_structured_output(PydanticModel)` for validated outputs. These are genuinely good primitives and usable without any agent.

**c) Simple tool-calling agents.** *"A model calling tools in a loop until a given task is complete"* — the official definition. If your feature really is that (a support agent with a handful of tools, an internal ops assistant), `create_agent` gives you the loop, persistence via `thread_id` + checkpointer, and streaming, in ~10 lines.

**d) Cross-cutting concerns via middleware.** HITL approval on specific tool calls, conversation summarization when context grows, PII redaction, model fallback/retry, dynamic prompt/tool selection — attach them without touching loop internals:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware, SummarizationMiddleware

agent = create_agent(
    model="anthropic:claude-sonnet-4-5",
    tools=[query_warehouse, list_tables],
    system_prompt=SYSTEM,
    middleware=[
        HumanInTheLoopMiddleware(interrupt_on={"query_warehouse": True}),
        SummarizationMiddleware(max_tokens_before_summary=4000),
    ],
    response_format=AnswerWithSQL,  # structured output
)
```

**Stay at this layer until** you need to intercept state mid-execution, enforce stage order programmatically, add conditional retry routing, or coordinate multiple agents. The community consensus (and LangChain's own docs) put the boundary there — those needs are the signal to drop down, not to fight `create_agent` with prompt engineering.

---

## 4. Where you need to drop to LangGraph

Drop to `StateGraph` when the *loop's shape* is the problem. Concretely:

**a) Deterministic multi-step pipelines.** When stages are known in advance (validate → generate → check → execute → summarize) and you want the *program*, not the model, to decide order. The official SQL-agent tutorial makes this contrast explicitly: the prebuilt agent *"relies on the system prompt to constrain its behavior,"* while the custom graph *"enforces control through dedicated nodes and explicit routing — enabling programmatic validation and deterministic workflows impossible with prompt-based constraints alone."* For anything where a wrong step has real cost (running SQL against a customer warehouse), you want dedicated nodes, not vibes.

**b) Custom state beyond messages.** `create_agent`'s state is essentially a message list. A real pipeline carries typed state: the parsed intent, retrieved schema snippets, the candidate SQL, validation errors, a `retry_count`, the result dataframe handle, the chart spec. In LangGraph that is a first-class `TypedDict`/Pydantic state schema with reducers; in `create_agent` you'd be smuggling it through message content.

**c) Bounded loops and explicit routing.** Conditional edges routing on state (`if state["retry_count"] >= 2: escalate`) give you self-correction with a hard ceiling. A ReAct agent's "loop until done" has no such structural guarantee.

**d) Fan-out / map-reduce.** The `Send` API dispatches parallel branches with per-branch state and merges results via reducers — e.g., profile 10 candidate tables concurrently, or decompose a question into sub-questions answered in parallel.

**e) Subgraphs and multi-agent.** Supervisor/worker topologies, specialist agents composed as subgraphs whose state flows in and out of the parent. This is squarely graph-API territory (and where `langgraph-supervisor`-style prebuilts live).

**f) Fine-grained HITL.** `interrupt()` pauses execution *at an arbitrary point inside a node*, persists via the checkpointer (indefinitely — hours or days), and resumes with a `Command` carrying the human's approval/edit. Middleware HITL covers "approve this tool call"; `interrupt()` covers "pause here, show the user the SQL and estimated cost, let them edit it, resume."

**g) Durable, long-running workflows.** Checkpointers persist every step; executions survive crashes and redeploys and resume where they left off. LangChain's own selection guidance names exactly this: choose LangGraph for *"complex workflows mixing deterministic and agentic components, long-running business processes, sensitive workflows requiring oversight, and fine-grained latency and cost control."*

---

## 5. Where both shine together

This is the default production pattern in 2026, not a hybrid hack: **LangGraph owns control flow; LangChain owns model/tool plumbing inside nodes.**

**Pattern 1 — LangChain models/tools inside StateGraph nodes** (the official SQL tutorial's own structure):

```python
from langgraph.graph import StateGraph, START, END
from langchain.chat_models import init_chat_model

model = init_chat_model("anthropic:claude-sonnet-4-5")

class PipelineState(TypedDict):
    question: str
    schema_context: str
    sql: str
    validation_error: str | None
    retry_count: int
    result: list[dict] | None

def generate_sql(state: PipelineState) -> dict:
    resp = model.with_structured_output(SQLDraft).invoke(
        sql_prompt(state["question"], state["schema_context"],
                   prior_error=state["validation_error"])
    )
    return {"sql": resp.sql}

builder = StateGraph(PipelineState)
builder.add_node("generate_sql", generate_sql)
builder.add_node("validate_sql", validate_sql)      # deterministic, no LLM
builder.add_node("execute", execute_sql)
builder.add_conditional_edges("validate_sql", route_on_validation,
    {"ok": "execute", "retry": "generate_sql", "give_up": "clarify"})
```

**Pattern 2 — `create_agent` as a node/subgraph in a bigger graph.** Because `create_agent` returns a compiled graph, you can embed an open-ended agent inside a deterministic pipeline — e.g., a schema-exploration agent that may take several tool hops, inside an otherwise fixed flow:

```python
explorer = create_agent(model=model, tools=[list_tables, get_columns, sample_rows],
                        system_prompt=EXPLORER_PROMPT)

def explore_schema(state: PipelineState) -> dict:
    out = explorer.invoke({"messages": [("user", exploration_task(state["question"]))]})
    return {"schema_context": out["messages"][-1].content}

builder.add_node("explore_schema", explore_schema)   # agent inside a pipeline
```

**Pattern 3 — middleware on the embedded agent.** The embedded agent keeps its own middleware stack (tool-call approval, summarization) while the parent graph keeps pipeline-level control (retry ceilings, `interrupt()` before warehouse execution). Two levels of governance, cleanly separated.

**Pattern 4 — shared runtime services.** Because both layers run on the LangGraph runtime, one checkpointer/thread model, one streaming interface, and one LangSmith trace tree cover the whole system — the pipeline and any embedded agents.

Rule of thumb: **graph for the skeleton, agent for genuinely open-ended organs, langchain-core primitives for every model touchpoint.**

---

## 6. When to use neither

The honest cases — and the criticism behind them.

**a) Single-LLM-call features.** Classification, extraction, summarization, rewriting. A provider SDK call with a Pydantic schema (or `instructor`, or the provider's native structured output) is fewer dependencies and a stack trace you can read. A graph with one node is ceremony.

**b) Simple RAG.** Embed → retrieve → stuff → answer is ~50 lines of plain Python against a vector store client. The framework version isn't fewer lines, and it's harder to debug.

**c) Teams that value debuggability above all.** The practitioner criticism is real and worth stating plainly: LangChain 0.x earned a reputation for abstraction sprawl ("5 layers of abstraction to change a minute detail"), 15–40-frame stack traces through framework internals, and repeated breaking changes across 0.1/0.2/0.3. A visible cohort of teams migrated to raw SDKs, OpenAI Agents SDK, Claude Agent SDK, or Pydantic-AI, and the "just use the API" position is a permanent fixture of HN/Reddit threads. The steelman: provider APIs converged (tool calling, structured output, streaming are now consistent), so the abstraction that saved you three weeks in 2022 saves you days now, while its maintenance cost didn't shrink.

**d) The fair counterweight.** Most of that criticism targets 0.x-era LangChain chains, not the 1.x stack — and notably, **LangGraph mostly escapes it**: even framework-skeptical practitioners tend to concede that checkpointing, `interrupt()`/resume, streaming, and the Send API are real infrastructure you'd otherwise hand-roll (poorly) with Celery/Temporal + custom state tables. If you build the "neither" version of a durable, resumable, HITL pipeline, you are rebuilding LangGraph's hard parts by hand. That can still be the right call for a tiny surface or a hard no-new-frameworks policy — but make it consciously, knowing what you're re-implementing.

**Decision hygiene:** default to *neither* for every new feature; earn your way up the stack only when the feature exhibits the signals in section 2.

---

## 7. Applied to chat-with-data (NL-to-SQL + analytics agent)

The core recommendation: **a fixed `StateGraph` pipeline, not a free-running agent.** Wrong SQL against a customer warehouse is a correctness and trust problem; you want programmatic validation gates and a bounded self-correction loop, which is exactly the deterministic-nodes-over-prompt-constraints argument from LangChain's own SQL tutorial. Naive unbounded retry demonstrably hurts; structured, bounded retry with error context helps.

### Stage-by-stage mapping

| Stage | Tool | Why |
|---|---|---|
| Intent classification / guardrail | **Neither** (plain LLM call w/ structured output) or a cheap `langchain-core` model call as the graph's entry node | One call, fixed schema out (`in_scope`, `needs_clarification`, `intent`). No loop needed. |
| Schema retrieval / semantic-layer lookup | **Plain Python node** (deterministic) — embedding search or catalog lookup; optionally a **`create_agent` explorer subgraph** for cold-start/ambiguous questions | Retrieval is deterministic; exploration ("which of these 400 tables matter?") is genuinely open-ended → agent-in-a-node (Pattern 2). |
| SQL generation | **LangChain model inside a node**: `with_structured_output(SQLDraft)`; prompt receives schema context + prior validation error from state | Model plumbing, not control flow. Structured output beats regexing SQL out of prose. |
| Validation (syntax, dialect, allowed tables, row limits, cost guard, PII columns) | **Plain Python node** — sqlglot parse, allowlist checks, `EXPLAIN` | Zero LLM. This gate is the reason the pipeline exists; never delegate it to the model. |
| Execution | **Plain Python node** behind the gate; add **LangGraph `interrupt()`** before execution for expensive/destructive queries | HITL here needs pause-persist-resume with the actual SQL editable — graph-API `interrupt()`, not middleware. |
| Self-correction loop | **LangGraph conditional edges + `retry_count` in state**: validation/execution error → back to `generate_sql` with the error in state; ceiling of 2–3 → clarification path | The canonical bounded-loop case. Custom state (error, counter) + explicit routing = graph API, full stop. |
| Chart generation | **Plain Python** (rule-based spec from result shape) with an optional **single LLM call** for chart-type choice/labels | Mostly deterministic; don't agent-ify it. |
| Summarization / answer composition | **LangChain model call in a node**, streaming tokens to the client via the graph's streaming interface | Benefits from the shared runtime: one stream carries node progress + tokens. |
| Conversation memory / follow-ups | **LangGraph checkpointer + `thread_id`** | Follow-up questions ("now break that down by month") need prior state (last SQL, last result schema) — custom state again. |
| Multi-question / dashboard requests | **LangGraph `Send`** fan-out per sub-question, merge via reducers | Map-reduce over sub-questions. |

### Shape of the graph

```
START → classify_intent ─┬─ off_topic ────────────► respond_and_END
                         └─ in_scope
                              ▼
                    retrieve_schema  (plain python; optional
                              ▼       create_agent explorer subgraph)
                       generate_sql  (langchain model, structured output)
                              ▼
                       validate_sql  (plain python: sqlglot, allowlists, EXPLAIN)
                        │         │
                     ok ▼         ▼ error & retry_count < N
                  [interrupt()?]  └────────► generate_sql   (bounded loop)
                        ▼             error & retry_count ≥ N
                     execute      ────────► ask_clarification → END
                        │    │
                     ok ▼    ▼ runtime error (loop back, same ceiling)
                  build_chart (python)
                        ▼
                    summarize (langchain model, streamed)
                        ▼
                       END        [checkpointer on the whole graph → threads,
                                   resume, time-travel debugging]
```

### What NOT to do

- **Don't** build this as one `create_agent` with `execute_sql` as a tool and hope the system prompt enforces "always validate first." That is prompt-constrained behavior where you need program-constrained behavior.
- **Don't** hand-roll durability. The checkpointer buys threads, resume-after-crash, and `interrupt()` HITL for free; the DIY equivalent is weeks of state-table plumbing.
- **Don't** pull in the whole 0.x-style toolbox. Dependencies you need: `langgraph`, `langchain` (for `create_agent` + middleware if you embed an agent), `langchain-core` + one provider package. Skip `langchain-community` unless a specific integration earns it.
- **Do** keep validation, execution, and chart-spec nodes framework-free plain Python — they're your test surface and your safety surface, and they should be unit-testable without any LLM or framework import.

---

## 8. Sources

**Official (docs.langchain.com / langchain.com):**
- [Agents — LangChain 1.x docs](https://docs.langchain.com/oss/python/langchain/agents) — `create_agent` as a "highly configurable harness" on the LangGraph runtime; middleware; when to drop down.
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) — "low-level orchestration framework and runtime"; explicit guidance to start with LangChain agents and use LangGraph for orchestration control.
- [Build a custom SQL agent — LangGraph tutorial](https://docs.langchain.com/oss/python/langgraph/sql-agent) — the canonical chat-with-data reference: dedicated nodes, explicit routing, `interrupt()` before query execution, contrast with prompt-constrained prebuilt agents.
- [LangChain & LangGraph 1.0 announcement](https://www.langchain.com/blog/langchain-langgraph-1dot0) — the three-tier layering and the "start high-level, drop down seamlessly" framing.
- [Agent middleware blog post](https://www.langchain.com/blog/agent-middleware) — middleware as the customization primitive for `create_agent`.
- [Subagents / multi-agent docs](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents-personal-assistant) and [langgraph-supervisor reference](https://reference.langchain.com/python/langgraph-supervisor) — agents as nodes/subgraphs, supervisor pattern.
- [deepagents](https://github.com/langchain-ai/deepagents) — prebuilt harness layer on top of `create_agent`.

**Independent comparisons and practitioner commentary:**
- [Is LangChain worth it in 2026? A practitioner's guide](https://blog.agentailor.com/posts/is-langchain-worth-it-2026) — `create_agent` until you need mid-execution state interception; dropping to StateGraph as the intended path.
- [LangChain 1.0 vs LangGraph 1.0 (ClickIT)](https://www.clickittech.com/ai/langchain-1-0-vs-langgraph-1-0/) and [Spheron decision guide](https://www.spheron.network/blog/langgraph-vs-langchain/) — production pattern of using both layers together.
- [LangChain vs LangGraph vs Deep Agents (BSWEN)](https://docs.bswen.com/blog/2026-06-16-langchain-vs-langgraph-vs-deep-agents/) — three-layer selection guidance.
- [Why we no longer use LangChain for building our AI agents — HN thread](https://news.ycombinator.com/item?id=40739982) and [the earlier abstractions critique](https://news.ycombinator.com/item?id=36648272) — the core "just use the API" criticism.
- [The LangChain exit: rewriting to raw SDKs in 2026 (Ravoid)](https://ravoid.com/blog/langchain-exit-raw-sdk-migration-2026) and [Why developers are quitting LangChain (AIM)](https://analyticsindiamag.com/ai-features/why-developers-are-quitting-langchain/) — the migration-to-SDKs wave and its reasoning.
- [Building a text-to-SQL assistant with LangGraph (Medium)](https://medium.com/@akhildevang/building-a-text-to-sql-assistant-with-langgraph-from-prompt-to-production-2dd2ec0ac37f) — independent production write-up of the StateGraph pipeline shape.
