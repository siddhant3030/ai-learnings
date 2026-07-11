# LangChain Core Concepts (1.x era, mid-2026)

A concepts guide for engineers building a production chat-with-data feature in Python. Covers what LangChain actually is today, the concepts that matter, what is legacy, and when to skip the framework entirely.

---

## TL;DR

- **LangChain 1.x is two things**: (1) a **standard interface over model providers** (chat models, messages, tools, structured output) and (2) an **agent runtime** — `create_agent` — built on top of LangGraph.
- The 0.x-era identity — "a library of chains" — is dead. `LLMChain`, `AgentExecutor`, `ConversationBufferMemory`, retrieval chains, etc. now live in `langchain-classic`, on life support until ~Dec 2026. Do not start new code on them.
- The concepts you actually need: `init_chat_model`, messages + content blocks, `@tool`, `with_structured_output` / `response_format`, `create_agent`, **middleware** (the 1.x extension mechanism: summarization, PII, HITL, dynamic prompts/models, retries), streaming via `stream_mode`, and checkpointers for conversation memory.
- LCEL/Runnables survive as the *interface* (`.invoke/.stream/.batch` everywhere, `|` for trivial composition) but are no longer how you build applications. Orchestration = LangGraph; agent loop = `create_agent`.
- For a chat-with-data product: LangChain buys you provider portability, tool/schema plumbing, middleware (PII redaction, HITL on SQL execution, summarization), and LangSmith tracing. It does **not** buy you your pipeline design — that's LangGraph (see the companion LangGraph doc) or your own code.
- Honest caveat: the abstraction tax is real. If you are single-provider and your workflow is a fixed pipeline, a thin wrapper over the provider SDK is a legitimate — often better — choice. LangChain earns its keep when you need multi-provider, agent loops with persistence/HITL, or the middleware/tracing ecosystem.

---

## Mental model: what LangChain IS in 2026

The 1.0 release (Oct 2025) was effectively a re-founding. The framework's own framing is **"Agent = Model + Tools + Prompt + Middleware"**, with `create_agent` as a *minimal, highly configurable harness* around the tool-calling loop. Everything else was either standardized (models, messages, tools) or evicted (`langchain-classic`).

Package layout:

| Package | Role |
|---|---|
| `langchain-core` | Base abstractions: `Runnable`, messages, `BaseChatModel`, `BaseTool`, prompts, output parsers. Few dependencies, stable. |
| `langchain` | The product surface: `create_agent`, middleware, `init_chat_model`, curated re-exports. Depends on `langgraph`. |
| `langchain-openai`, `langchain-anthropic`, `langchain-google-genai`, `langchain-aws`, … | One package per provider, each implementing the standard chat-model interface. Installable as extras: `pip install "langchain[anthropic]"`. |
| `langgraph` | The low-level orchestration runtime (graphs, state, checkpointers, interrupts). `create_agent` compiles to a LangGraph graph. |
| `langchain-community` | Long tail of third-party integrations (loaders, vector stores, utilities). Community-maintained, variable quality. |
| `langchain-classic` | Everything deprecated: `LLMChain`, `AgentExecutor`, old memory classes, legacy retrievers, `hub`. Security fixes only; retirement targeted ~Dec 2026. |

Requires Python 3.10+. The mental model that keeps you sane:

> **LangChain = the model/tool/message abstraction layer + an agent runtime on LangGraph. LangGraph = the orchestration substrate. langchain-classic = the museum.**

---

## 1. Chat models & the provider abstraction

**What it is.** A single interface (`BaseChatModel`) over every provider. `init_chat_model` gives you a model from a string; provider classes (`ChatAnthropic`, `ChatOpenAI`, …) give you the same thing explicitly. All of them expose `invoke()`, `stream()`, `batch()`, `bind_tools()`, `with_structured_output()` and return `AIMessage`s.

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("anthropic:claude-sonnet-4-6", temperature=0, max_tokens=2000)
resp = model.invoke("Why do parrots talk?")          # -> AIMessage
resp.text, resp.usage_metadata                        # text + token counts

# provider-explicit form — identical interface
from langchain_anthropic import ChatAnthropic
model = ChatAnthropic(model="claude-sonnet-4-6", temperature=0)
```

Standard params (`temperature`, `max_tokens`, `timeout`, `max_retries`) are normalized across providers; provider-specific params pass through as kwargs.

**Why the abstraction matters vs raw SDKs.**
- Swap providers (or run per-tenant/per-tier model routing) without rewriting call sites — for a chat-with-data feature, this means you can A/B a cheap model for SQL generation vs an expensive one for planning with a config change.
- One tool-calling and structured-output code path instead of per-provider translation (OpenAI `tools` vs Anthropic `tool_use` blocks differ meaningfully at the raw-SDK level).
- Everything downstream (agents, middleware, LangSmith tracing) plugs into this interface.

**Pitfalls.**
- The abstraction is lowest-common-denominator-ish: provider-exclusive features (prompt caching specifics, server-side tools, some reasoning controls) need provider-specific kwargs anyway — so "portability" is partial in practice.
- `init_chat_model("gpt-…")` infers the provider from the name; be explicit (`"openai:…"`) in production to avoid surprises.
- Default `max_retries` is generous (6); tune it or your p99 latency will be shaped by silent retry storms.
- Version skew: keep `langchain-core` and provider packages in lockstep; mismatches are a classic source of cryptic errors.

---

## 2. Messages, content blocks, multimodal

**What it is.** The conversation data model. Four types:

- `SystemMessage` — behavior/context instructions.
- `HumanMessage` — user input (text or multimodal).
- `AIMessage` — model output: text, `tool_calls`, `usage_metadata`, provider metadata.
- `ToolMessage` — result of one tool execution, tied back via `tool_call_id`.

```python
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage, ToolMessage

msgs = [
    SystemMessage("You are a data analyst for the org's warehouse."),
    HumanMessage("How many donors did we add last month?"),
]
ai = model.invoke(msgs)             # AIMessage; may contain ai.tool_calls
# dict shorthand also accepted everywhere:
model.invoke([{"role": "user", "content": "hi"}])
```

**Content blocks (new in 1.x).** `message.content` remains provider-shaped, but `message.content_blocks` lazily normalizes it into standard typed blocks: `{"type": "text", ...}`, `{"type": "reasoning", ...}`, `{"type": "image", "url"/"base64": ...}`, plus audio/video/file and tool-call blocks. This is how you read reasoning traces and multimodal output uniformly across providers.

```python
human = HumanMessage(content_blocks=[
    {"type": "text", "text": "What does this dashboard show?"},
    {"type": "image", "url": "https://example.com/chart.png"},
])
for block in ai.content_blocks:
    if block["type"] == "reasoning": ...
```

**When useful.** Always — this is the wire format for everything else. `usage_metadata` (`input_tokens`/`output_tokens`/`total_tokens`) is your per-call cost hook.

**Pitfalls.**
- Don't parse `message.content` by hand for anything non-trivial; use `.content_blocks` (or `.text`) — raw content shape varies by provider (string vs list of provider blocks).
- Every `tool_call` in an `AIMessage` **must** get a matching `ToolMessage` before the next model call, or providers reject the request. `create_agent` handles this; hand-rolled loops often get it wrong.
- Message history is state you own: unbounded histories are the classic silent cost/latency leak (see middleware/summarization below).

---

## 3. Tools

**What it is.** Python functions exposed to the model. `@tool` derives name/description/schema from the function name, docstring, and **type hints** (hints are required — they *are* the schema).

```python
from langchain.tools import tool
from pydantic import BaseModel, Field

@tool
def run_sql(query: str, limit: int = 100) -> str:
    """Execute a read-only SQL query against the analytics warehouse."""
    return warehouse.execute_readonly(query, limit)

# explicit schema when you need descriptions/constraints per-arg
class SqlInput(BaseModel):
    query: str = Field(description="A single read-only SELECT statement")
    limit: int = Field(default=100, le=1000)

@tool(args_schema=SqlInput)
def run_sql2(query: str, limit: int = 100) -> str:
    """Execute a read-only SQL query."""
    ...
```

**Execution.** You rarely execute tools yourself. Inside `create_agent` (and LangGraph), a **ToolNode** executes the model's `tool_calls` — in parallel where possible — and appends `ToolMessage`s. Manually: `ai.tool_calls` is a list of `{"name", "args", "id"}`; you call the tool and build the `ToolMessage`.

**Runtime access.** Tools can take a `ToolRuntime` parameter (invisible to the model) exposing agent state, per-run context, the long-term store, and a stream writer; a tool can even update agent state by returning a `Command`.

```python
from langchain.tools import tool, ToolRuntime

@tool
def get_schema(table: str, runtime: ToolRuntime) -> str:
    """Get column schema for a warehouse table."""
    org_id = runtime.context.org_id            # per-run injected context — not model-visible
    return catalog.describe(org_id, table)
```

**Error handling.** Default behavior: a raised exception becomes an error `ToolMessage` so the model can retry/correct. For control, wrap it in middleware:

```python
from langchain.agents.middleware import wrap_tool_call

@wrap_tool_call
def handle_tool_errors(request, handler):
    try:
        return handler(request)
    except QuerySyntaxError as e:
        return ToolMessage(content=f"SQL error: {e}. Fix the query and retry.",
                           tool_call_id=request.tool_call["id"])
```

**When useful.** This is the core of chat-with-data: `run_sql`, `get_schema`, `search_semantic_layer`, `render_chart` are all tools. The schema-from-type-hints plumbing plus ToolNode execution is genuinely valuable vs hand-writing JSON schemas per provider.

**Pitfalls.**
- Tool descriptions are prompt engineering. Vague docstrings → wrong tool selection. Treat them as carefully as your system prompt.
- Return **strings/serializable content sized for a context window**. A tool that returns 50k rows will blow up cost and quality — truncate/aggregate in the tool (your result-set governance layer belongs here, including result-set PII checks).
- Too many tools degrades selection accuracy; use `LLMToolSelectorMiddleware` or split into subagents beyond ~15-20 tools.
- Don't put secrets/tenancy in tool args (model-visible and model-controlled); inject via `ToolRuntime.context`.

---

## 4. Structured output

**What it is.** Getting validated, typed objects instead of prose. Two levels:

**On a model** — `with_structured_output`:

```python
from pydantic import BaseModel

class SqlPlan(BaseModel):
    tables: list[str]
    sql: str
    explanation: str

structured = model.with_structured_output(SqlPlan)
plan = structured.invoke("Monthly active users by region")   # -> SqlPlan instance
```

**On an agent** — `response_format` in `create_agent`. The final answer (after any tool use) is coerced to your schema and surfaced at `result["structured_response"]`. Strategies:
- `ProviderStrategy` — uses native structured-output APIs (OpenAI, Anthropic, Gemini, Grok); pass `strict=True` where supported.
- `ToolStrategy` — universal fallback via tool calling, with retry-on-validation-error control: `handle_errors=True | "msg" | ExceptionType | callable | False`.
- Pass the schema class directly and LangChain picks the best strategy for the model.

Accepted schemas: Pydantic models (returns instances), dataclasses, TypedDict, raw JSON Schema (return dicts); `ToolStrategy` also accepts Unions.

**When useful.** Everywhere in chat-with-data: query plans, clarification decisions, chart specs, eval verdicts. Prefer structured output over parsing text — this replaced most of the old output-parser machinery.

**Pitfalls.**
- Validation failure handling differs by strategy — decide explicitly (`handle_errors`) rather than relying on defaults; naive infinite retries burn tokens.
- Very complex/deeply nested schemas degrade generation quality on all providers; flatten where possible.
- `with_structured_output` can't be combined with free-form tool use on the same call — that's what agent-level `response_format` is for.
- Prompted (non-tool, non-native) structured output from 0.x was **removed** in 1.x.

---

## 5. `create_agent` — the 1.x agent API

**What it is.** The successor to both the legacy `AgentExecutor` (0.x, string-parsing ReAct loops) and `langgraph.prebuilt.create_react_agent` (which it replaced and re-homed under `langchain.agents`). It builds the canonical tool-calling loop — model → tool calls → ToolNode → model → … until no tool calls remain — and **returns a compiled LangGraph graph**, so you get persistence, streaming, interrupts, and time-travel for free, and can embed it as a node in a bigger LangGraph.

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="anthropic:claude-sonnet-4-6",         # string or model instance
    tools=[run_sql, get_schema],
    system_prompt="You are a careful analytics assistant. Always inspect schema before writing SQL.",
    response_format=SqlAnswer,                    # optional structured final answer
    middleware=[...],                             # see next section
    checkpointer=InMemorySaver(),                 # Postgres-backed in prod
)

config = {"configurable": {"thread_id": "user-42-session-7"}}
result = agent.invoke({"messages": [{"role": "user", "content": "Top 5 programs by enrollment?"}]},
                      config)
result["messages"]              # full trace: AI msgs, tool calls, tool results
result["structured_response"]   # validated SqlAnswer, if response_format set
```

Key facts:
- State is a `messages` list (plus your custom state via middleware/`state_schema`); input/output are state dicts, not strings.
- **Memory = checkpointer + `thread_id`.** Same `thread_id` ⇒ conversation resumes with full state. This is what replaced 0.x memory classes.
- `context` (typed, via `context_schema`) is the 1.x way to pass per-run data (user id, org id, permissions) to tools and middleware — replacing the `config["configurable"]` grab-bag.
- Migration deltas from `create_react_agent`: import path moved to `langchain.agents`; `prompt` → `system_prompt`; pre/post-model hooks → middleware; custom state must be `TypedDict` (not Pydantic); pre-bound models (`model.bind_tools(...)` passed in) no longer supported — give it the bare model and the tools list.

**When useful.** Whenever the model should *decide* which tools to call and iterate (agentic RAG, exploratory SQL agents). For a fixed pipeline (classify → generate SQL → validate → execute → summarize), build a LangGraph graph instead — `create_agent` is a loop, not a pipeline.

**Pitfalls.**
- No `checkpointer` ⇒ no memory and no HITL interrupts. In production use a durable checkpointer (`PostgresSaver`), not `InMemorySaver`.
- The loop is open-ended by default — bound it with `ModelCallLimitMiddleware` / `ToolCallLimitMiddleware` or a recursion limit, or a confused model will spin.
- Don't stuff giant schema dumps into `system_prompt`; use tools or dynamic prompts (middleware) to load context on demand.

---

## 6. Middleware — the 1.x extension system

**What it is.** The single mechanism for customizing the agent loop; it replaced ad-hoc hooks, callbacks-for-control, and most "wrap the chain" patterns. Middleware attach at defined points:

- Node-style: `before_agent`, `before_model`, `after_model`, `after_agent` — read/update state, optionally `jump_to` (`"end"`, `"model"`, `"tools"`).
- Wrap-style: `wrap_model_call`, `wrap_tool_call` — intercept the call, mutate the request (`request.override(model=..., system_message=...)`), retry, or short-circuit.

Write them as decorated functions or `AgentMiddleware` subclasses:

```python
from langchain.agents.middleware import before_model, wrap_model_call, AgentMiddleware

@before_model
def log_turn(state, runtime):
    log.info("model call", n_messages=len(state["messages"]))
    return None                                   # or a state update dict

@wrap_model_call
def route_model(request, handler):
    # dynamic model selection: escalate long/hard conversations
    if len(request.messages) > 12:
        return handler(request.override(model=big_model))
    return handler(request)
```

**Built-ins worth knowing** (all passed via `middleware=[...]`):

| Middleware | What it does |
|---|---|
| `SummarizationMiddleware(model=…, trigger=("tokens", 4000), keep=("messages", 20))` | Auto-compresses old history — the production answer to context growth. |
| `HumanInTheLoopMiddleware(interrupt_on={"run_sql": {...}})` | Pauses the graph (LangGraph interrupt) for approve/edit/reject on chosen tools. Requires a checkpointer. Ideal for gating destructive or expensive SQL. |
| `PIIMiddleware("email", strategy="redact", apply_to_input=…, apply_to_output=…, apply_to_tool_results=…)` | Detect/redact/mask/block emails, credit cards, IPs, URLs, etc. Note `apply_to_tool_results` — covers **result-set** PII, not just prompts. |
| `ModelFallbackMiddleware(primary, *fallbacks)` | Provider outage / error fallback chain. |
| `ModelRetryMiddleware`, `ToolRetryMiddleware` | Retries with backoff/jitter, filterable by error type. |
| `ModelCallLimitMiddleware`, `ToolCallLimitMiddleware` | Hard budgets per run/thread with graceful exit. |
| `LLMToolSelectorMiddleware` | Pre-filters large tool sets with a small model. |
| `ContextEditingMiddleware` | Clears stale tool outputs from context (Anthropic-style context editing). |
| `TodoListMiddleware`, `SubAgentMiddleware`, `FilesystemMiddleware` | Deep-agents patterns: planning, delegation, file memory. |

Dynamic **prompts** are just `wrap_model_call` overriding the system message per call (e.g., inject the schema slice relevant to the user's question).

**When useful.** This is where most of your chat-with-data production hardening lives: PII redaction, HITL on query execution, summarization, cost caps, model routing — as composable, ordered units (`before_*` run first→last, `wrap_*` nest, `after_*` run last→first) instead of a hand-woven loop.

**Pitfalls.**
- Ordering matters and is easy to get wrong (e.g., PII redaction should wrap *inside* logging or your logs leak). Test the composed stack, not units in isolation.
- Middleware is `create_agent`-specific. If you build a raw LangGraph pipeline instead, you re-implement these behaviors as nodes — factor them so they're reusable.
- Regex-based `PIIMiddleware` built-ins cover common patterns only; names/addresses need a custom `detector` (or your own NER pass). Don't treat the built-in as a compliance guarantee.

---

## 7. Runnables & LCEL — what's left

**What it is.** `Runnable` is the universal interface in `langchain-core`: everything (models, tools, prompts, parsers, graphs, agents) exposes `invoke / stream / batch` (+ async `ainvoke/astream/abatch`). LCEL is the `|` composition syntax over Runnables.

**Still relevant:**
- The interface itself — it's why `agent.stream(...)` and `model.invoke(...)` feel the same, and how LangSmith traces everything uniformly.
- Trivial linear composition: `prompt | model | parser` for a stateless one-shot transform is still fine and officially supported.
- `RunnableConfig` / configuration plumbing under the hood.

**Legacy:** LCEL as an *application architecture* — long pipe chains, `RunnableBranch`, `RunnableParallel`-heavy graphs, routing via lambdas. Official guidance since 1.0: anything with branching, state, retries, or more than a couple of steps belongs in **LangGraph** (explicit nodes/edges/state), and agent loops belong in `create_agent`. Debugging a 10-stage pipe chain was one of the framework's biggest pain points; don't recreate it.

**Pitfall.** Much of the tutorial content on the internet (2023–2024) teaches LCEL-everything and `AgentExecutor`. Date-check anything you copy.

---

## 8. Streaming

**What it is.** Two layers:

**Model-level** — `model.stream()` yields `AIMessageChunk`s:

```python
for chunk in model.stream("Explain this metric"):
    print(chunk.text, end="", flush=True)
```

**Agent/graph-level** — `agent.stream(..., stream_mode=...)` (it's a LangGraph graph):

- `stream_mode="updates"` — one event per node step (model finished, tools finished). Good for progress UI ("running query…").
- `stream_mode="messages"` — `(token_chunk, metadata)` tuples: true token streaming from any model call inside the graph.
- `stream_mode="custom"` — arbitrary payloads emitted from inside tools/nodes via the stream writer (e.g., "scanned 3 tables", row-count progress).
- Pass a list (`stream_mode=["updates", "messages"]`) to multiplex; `astream_events` remains for fine-grained event streams across nested runnables.

```python
async for token, meta in agent.astream({"messages": [msg]}, config, stream_mode="messages"):
    if meta.get("langgraph_node") == "model":
        ws.send(token.text)
```

**When useful.** Non-negotiable for chat UX. `messages` for the answer tokens + `updates`/`custom` for tool progress is the standard chat-with-data pattern (maps directly onto a WebSocket protocol).

**Pitfalls.**
- Tool-call arguments stream as partial JSON fragments in `messages` mode; use `updates` to get parsed, complete tool calls rather than reassembling fragments.
- Filter by node/tags — raw streams include *every* model call (summarization middleware, tool-selector calls), and you don't want internal chatter in the user's chat bubble.
- Use async (`astream`) end-to-end in a server; mixing sync streaming into an async web stack causes event-loop blocking.

---

## 9. Prompt templates & output parsers — current relevance

**Prompt templates** (`ChatPromptTemplate`) still exist in `langchain-core` and are fine for parameterized, versionable prompts:

```python
from langchain_core.prompts import ChatPromptTemplate
prompt = ChatPromptTemplate([("system", "You answer questions about {schema_name}."),
                             ("user", "{question}")])
chain = prompt | model      # legitimate small LCEL use
```

But in the agent world their role shrank: `create_agent` takes a `system_prompt` string, and *dynamic* prompting is middleware (`wrap_model_call` overriding the system message). Many teams just use f-strings; that's fine too — templates earn their keep mainly with LangSmith prompt versioning.

**Output parsers** (`StrOutputParser`, `PydanticOutputParser`, "fix the JSON" retry parsers) are **mostly legacy**. Native structured output (`with_structured_output` / `response_format`) replaced them. `StrOutputParser` survives as a trivial `AIMessage → str` adapter in small chains. If you're reaching for `OutputFixingParser`, you're on the old path — use structured output.

---

## 10. Retrieval / RAG components — brief, and when NOT to use them

**What exists.** Document loaders → text splitters (`langchain-text-splitters`, e.g. `RecursiveCharacterTextSplitter`) → embeddings → vector stores (`langchain-postgres` for PGVector, `langchain-chroma`, etc.) → retrievers (a `Runnable` interface: query → documents).

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter
chunks = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150).split_documents(docs)
vs = PGVector.from_documents(chunks, embeddings, connection=PG_URL)
retriever = vs.as_retriever(search_kwargs={"k": 5})
```

Current official guidance favors **agentic RAG**: expose retrieval as a *tool* and let the agent decide when to search — rather than the 0.x `RetrievalQA`-style hardwired retrieve-then-generate chain (those chains are in `langchain-classic` now).

**When NOT to use them — important for chat-with-data:**
- **Structured/warehouse data is not a RAG problem.** Don't embed rows and similarity-search them; questions like "how many donors last month" need SQL over governed tables, not vector search. Vector retrieval in a chat-with-data product is only for *metadata*: table/column descriptions, semantic-layer docs, verified example queries.
- Small corpora (a few hundred docs of schema descriptions) often don't need a vector store at all — put them in the prompt or use keyword search; embeddings add infra and failure modes.
- The loaders/splitters are commodity code; if you outgrow them, replacing them is trivial — don't contort your pipeline around their interfaces.

---

## 11. Integrations ecosystem & `langchain-community`

The ecosystem is the moat: ~1000+ integrations. Structure matters for supply-chain hygiene:

- **First-party provider packages** (`langchain-openai`, `langchain-anthropic`, `langchain-postgres`, …): maintained with the framework, versioned properly. Prefer these.
- **`langchain-community`**: the long tail — hundreds of loaders, stores, and utilities of highly variable quality and maintenance. Fine for prototyping; audit before production (pin versions, check upstream activity, watch transitive deps).
- Standalone partner packages increasingly graduate out of community (`langchain-chroma`, `langchain-elasticsearch`, …).

Rule of thumb: production dependencies should be `langchain-core`, `langchain`, `langgraph`, your provider package(s), and a *small, deliberate* set of integration packages. If you find yourself importing from `langchain_community` in a hot path, check whether a dedicated package exists.

---

## 12. LangSmith (tracing & evals) — high level

LangSmith is LangChain's hosted observability/eval platform (proprietary, separate from the MIT-licensed OSS). Because everything is a `Runnable`, tracing is nearly free:

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=...
```

Every agent run becomes a nested trace: each model call (full prompts/completions, token counts, latency), each tool call with args/results, middleware steps. For a SQL agent this is the difference between "it gave a wrong answer" and "it picked the wrong table because the schema tool returned a truncated listing."

Beyond tracing: **datasets + evaluators** (run your agent over curated question→expected-answer sets, score with LLM-as-judge or custom Python — this is where a chat-with-data eval scorecard lives), **prompt versioning/hub**, and **production monitoring** (cost, latency, error dashboards; sample traces into datasets).

Pitfalls: traces contain full prompts and tool results — configure PII scrubbing/self-hosting per your data-governance needs; it's a paid product with per-trace pricing at scale; alternatives (Langfuse — OSS, Phoenix, OTel-based setups) also speak LangChain's instrumentation.

---

## What's legacy — do not build on these

| Legacy (in `langchain-classic`, EOL ~Dec 2026) | 1.x replacement |
|---|---|
| `AgentExecutor`, `initialize_agent`, ReAct-via-string-parsing | `create_agent` (tool-calling loop on LangGraph) |
| `langgraph.prebuilt.create_react_agent` | `create_agent` from `langchain.agents` (same lineage, moved + renamed params) |
| `LLMChain`, `ConversationChain`, `RetrievalQA`, `ConversationalRetrievalChain`, `load_summarize_chain` | Direct model calls, small LCEL pipes, LangGraph graphs, agentic RAG |
| `ConversationBufferMemory`, `ConversationSummaryMemory`, etc. | Checkpointer + `thread_id`; `SummarizationMiddleware`; LangGraph store for long-term memory |
| Output-fixing/retry parsers, prompted structured output | `with_structured_output`, `response_format` (Tool/Provider strategies) |
| LCEL as app architecture (`RunnableBranch`, mega-pipes) | LangGraph for orchestration; LCEL only for trivial linear composition |
| Callbacks for control flow | Middleware (callbacks remain for observability plumbing) |
| `config["configurable"]` for app data | Typed `context` / `context_schema` |

Red flags in code review or tutorials: `from langchain.chains import …`, `from langchain.memory import …`, `AgentExecutor`, `initialize_agent`, `RetrievalQA` — all 0.x patterns.

---

## Criticisms — read before committing

Be honest about the trade:

1. **Abstraction tax.** Extra layers between you and the API mean more indirection when debugging ("what prompt was actually sent?"). Mitigated in 1.x (middleware is explicit; LangSmith shows exact payloads) but not gone. Hidden context accumulation (naive full-history memory) has produced well-documented 2–3x cost overruns for teams that didn't manage state deliberately.
2. **Churn.** The 0.x era rewrote its core abstraction roughly annually (chains → LCEL → LangGraph-everything → `create_agent`/middleware). 1.0 is a stability commitment, and 1.x has held so far — but the ecosystem's tutorial corpus is a minefield of stale patterns, and `langchain-classic` retires ~Dec 2026, forcing migrations on laggards.
3. **The gap it filled has narrowed.** In 2023, providers had no tool calling or structured output; LangChain solved real problems. In 2026, provider SDKs (and agent SDKs — OpenAI Agents SDK, Claude Agent SDK) are excellent, and provider APIs have converged. A meaningful cohort of production teams has migrated to thin wrappers over raw SDKs and reports simpler debugging and less maintenance. If you're **single-provider with a fixed workflow**, a direct SDK + your own ~200-line loop is a defensible, often better choice.
4. **Dependency surface.** Even trimmed, you're pulling `langchain-core` + `langchain` + `langgraph` + providers; audit and pin.
5. **Where it still clearly wins:** multi-provider routing/fallback, the agent runtime (durable execution, checkpointing, HITL interrupts — genuinely hard to hand-roll well), the middleware library (summarization, PII, limits), and first-class tracing/evals. For a chat-with-data product needing HITL on SQL execution, PII layers, per-tenant model routing, and eval infrastructure, that bundle is worth the tax — *provided* you use LangGraph for the pipeline and keep LangChain to the model/tool layer.

Decision heuristic: **use the provider SDK when the workflow is fixed and single-provider; use LangChain/LangGraph when you need the runtime (persistence, interrupts, streaming infra) and the middleware/observability ecosystem.** Avoid the middle path of using LangChain but fighting its abstractions.

---

## Sources

- [LangChain 1.x overview (official docs)](https://docs.langchain.com/oss/python/langchain/overview)
- [Agents / `create_agent`](https://docs.langchain.com/oss/python/langchain/agents)
- [Models & `init_chat_model`](https://docs.langchain.com/oss/python/langchain/models)
- [Messages & content blocks](https://docs.langchain.com/oss/python/langchain/messages)
- [Tools & ToolRuntime](https://docs.langchain.com/oss/python/langchain/tools)
- [Structured output](https://docs.langchain.com/oss/python/langchain/structured-output)
- [Middleware (built-in)](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [Middleware (custom)](https://docs.langchain.com/oss/python/langchain/middleware/custom)
- [Streaming](https://docs.langchain.com/oss/python/langchain/streaming)
- [Short-term memory](https://docs.langchain.com/oss/python/langchain/short-term-memory)
- [Retrieval / RAG](https://docs.langchain.com/oss/python/langchain/retrieval)
- [LangChain v1 migration guide](https://docs.langchain.com/oss/python/migrate/langchain-v1)
- [`langchain-classic` reference](https://reference.langchain.com/python/langchain-classic)
- [Community discussion: is LangChain too complex for simple RAG (2025)](https://github.com/orgs/community/discussions/182015)
- [Designveloper: an honest look at LangChain criticism](https://www.designveloper.com/blog/is-langchain-bad/)
