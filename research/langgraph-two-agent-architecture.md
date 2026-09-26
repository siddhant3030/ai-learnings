# How Chat with Data uses LangGraph: the two-agent architecture

Audience: a Dalgo engineer who has never used LangGraph. All paths are relative to
`DDP_backend/` unless noted. The feature lives under `ddpui/core/ai/`.

## 1. Why LangGraph at all

Chat with Data is a WebSocket chat where an NGO user like Priya asks
"how many surveys in Maharashtra?" and an agent discovers tables, writes SQL,
asks for approval, runs the query, and answers. LangGraph gives us five things
we would otherwise hand-build:

| What we get | Where it shows up |
|---|---|
| Durable conversations — every message checkpointed in Postgres | `agent/checkpointer.py` |
| Interrupts — the graph pauses mid-run for human approval and resumes later, even after a server restart | `agent/hitl.py`, consumer `resume_approval` |
| Subgraph composition — two prebuilt agents mounted as nodes inside one parent pipeline | `chat/turn_graph.py:165,170` |
| Streaming — tokens, tool activity, and stage boundaries as one event stream | `chat/turn_runner.py:184` |
| One persistence story — no message table; the checkpoint IS the conversation | `agent/checkpointer.py` docstring |

## 2. The LangGraph concepts we actually use

### 2.1 StateGraph + TypedDict state
A `StateGraph` is a graph whose nodes read and write a shared state dict. The
state's shape is a `TypedDict`. Ours is `TurnState` (`chat/turn_graph.py:81-89`):
`messages`, `question`, `route`, `has_history`, `validation`. Built with
`StateGraph(TurnState, context_schema=RunContext)` (`turn_graph.py:160`).

### 2.2 Nodes
A node is a function that takes state and returns a partial state update.
Example: `route_node` (`turn_graph.py:108-121`) calls a cheap router LLM on
Priya's question and returns `{"route": {...}, "has_history": ...}`. Nodes are
thin adapters; the LLM "brains" live in `llm_calls/` and are injected as
functions so tests can patch them (`turn_graph.py` module docstring).

### 2.3 Conditional edges
`add_conditional_edges(node, decision_fn, destinations)` picks the next node at
runtime from the state. Two in our graph:
- `route_decision` after `route_node` (`turn_graph.py:148-158`): small talk →
  `casual_reply_node`; first-turn ambiguity → `clarify_node`; platform help →
  `guide_agent`; else `retrieve_context_node` → `sql_agent`.
- `after_sql_agent` (`turn_graph.py:184-187`): handed off → `guide_agent`,
  else `validate_node`.

### 2.4 The add_messages reducer and channel sharing
`messages: Annotated[list[AnyMessage], add_messages]` (`turn_graph.py:85`)
declares a reducer: node updates are *merged* into the list (append by default;
a message with an existing id *replaces* that message). Because the parent state
and the prebuilt agents' state both have a channel named `messages`, mounting an
agent as a node shares the full conversation between parent and subgraph — no
copying (`turn_graph.py:82-83`).

### 2.5 Compiled prebuilt agents mounted as subgraph nodes
`create_agent(model, tools, middleware, context_schema, checkpointer)` returns a
compiled graph implementing the standard loop: model → tools → model, until the
model stops calling tools (`agent/chat_data_agent.py:209-215`). We never edit
that topology. The compiled agent is passed straight into
`graph.add_node("sql_agent", agent)` (`turn_graph.py:165`) — a compiled graph is
a valid node.

### 2.6 Checkpointer + thread_id
`AsyncPostgresSaver` over a shared psycopg3 pool (`agent/checkpointer.py:38-53`;
tables created by `manage.py chat_with_data_setup`). Each run passes
`config={"configurable": {"thread_id": str(session.thread_id)}}`
(`chat/turn_runner.py:142-145`) — the thread_id keys the conversation. Only the
**parent** graph is compiled with the checkpointer; subgraphs inherit it
(`turn_graph.py:104-105`, compile at `:192`). The runner reuses
`agent.checkpointer` so there is one saver, one thread namespace
(`turn_runner.py:117,129`).

### 2.7 Interrupts and Command(resume=...) — human-in-the-loop
`HumanInTheLoopMiddleware` intercepts gated tool calls and raises an interrupt:
the graph checkpoints and stops, surfacing `__interrupt__` in the stream
(`turn_runner.py:193-210`). Example: the SQL agent wants `execute_sql`; Priya
sees an approval card. Her click becomes
`build_resume_payload(...)` → `Command(resume={"decisions": [...]})`
(`agent/hitl.py:101-113`, `turn_runner.py:177-181`) and the graph continues from
the checkpoint. Two pause kinds ride one mechanism (`hitl.py:1-15`): *approval*
(approve/reject `execute_sql` and the guide's creation tools) and *question*
(`ask_user` is never executed — the user's typed reply becomes the tool result
via a "respond" decision). Paused turns are durable: graph state in Postgres,
the pending card in Redis for 24h (`websockets/chat_with_data_consumer.py:60-64,317-330`).

### 2.8 Middleware hooks
Middleware is the sanctioned way to customize the prebuilt loop
(`agent/middleware.py:1-7`). Hooks we use:

| Hook | Ours | What it does |
|---|---|---|
| `@before_model(can_jump_to=["end"])` | `sql_retry_limiter` (`middleware.py:53-74`) | After 5 failed `execute_sql` calls, appends an apology AIMessage and returns `{"jump_to": "end"}` — a deterministic loop exit |
| `@before_model` returning `llm_input_messages` | `trim_history` (`middleware.py:77-95`) | Caps the model request at 60k tokens. `llm_input_messages` affects only this model call; the checkpoint keeps the full history |
| `@before_model` returning `messages` | `repair_foreign_tool_errors` (`middleware.py:137-151`) | Returns replacement ToolMessages carrying the *original ids*, so `add_messages` swaps them **durably in the checkpoint** — a one-time repair |
| `@dynamic_prompt` | `org_system_prompt` (`chat_data_agent.py:179-182`) | Rebuilds the system prompt per model call from `runtime.context` (dialect, schema allowlist, org memory) |
| `ContextEditingMiddleware` | `clear_old_tool_results` (`middleware.py:154-163`) | Drops bulky old tool outputs from the request past 40k tokens, keeping the 5 newest |
| `PIIMiddleware` (several) | `agent/pii.py:175-193` | Masks PII in user input and tool results. Rewrites the *state*, so redacted values are what get checkpointed and traced (`pii.py:26-29`) |
| `HumanInTheLoopMiddleware` (after_model) | `agent/hitl.py:59-73` | The interrupts above |

Key distinction: `llm_input_messages` = this-request-only view; a `messages`
update = durable state change via the reducer.

### 2.9 Runtime / context_schema + ToolRuntime — dependency injection
`RunContext` (`agent/run_context.py`) is a dataclass carrying org_id, dialect,
schema allowlist, the warehouse client, and resolved RBAC flags. It is declared
as `context_schema` on both the parent graph and the agents, and passed at
invoke time: `graph.astream(..., context=context)` (`turn_runner.py:184-188`).
Nodes read it via `runtime: Runtime[RunContext]` (`turn_graph.py:108`). Tools
declare a `runtime: ToolRuntime[RunContext]` parameter — the model never sees
it; LangGraph injects it. Example: `execute_sql(sql, runtime)` reads
`runtime.context.warehouse` and `allowed_schemas` (`tools/sql_tools.py:23-35`).
`build_run_context()` (`agent/context_builder.py:47`) is the ONLY place that
touches the ORM or credentials — tools never do, and never trust an
LLM-supplied org id.

### 2.10 Streaming modes and namespaces
The runner streams with
`graph.astream(input, stream_mode=["messages", "updates"], subgraphs=True)`
(`turn_runner.py:184-190`), yielding `(namespace, mode, chunk)` triples:
- `mode == "messages"`: token chunks. Only chunks whose
  `meta["langgraph_node"] == "model"` reach the user — router/validator LLM
  calls never leak (`turn_runner.py:212-223`).
- `namespace` non-empty: the chunk came from inside a subgraph (sql_agent or
  guide_agent) — used for `tool_start`/`tool_end` events (`turn_runner.py:225-275`).
- namespace empty, `mode == "updates"`: parent stage boundaries —
  `message_complete` when an agent node finishes, `validation` after
  `validate_node` (`turn_runner.py:277-306`).

### 2.11 recursion_limit
LangGraph aborts a run after N graph steps. Ours is 160
(`chat_data_agent.py:31-40`), passed in config (`turn_runner.py:144`). Every
middleware hook is its own graph node, so one model⇄tool cycle costs ~12 steps
with our stack (7 before_model hooks incl. 5 PII rules + model + 3 after_model
+ tools). 160 ≈ ~13 tool calls of headroom; the real runaway guard is
`sql_retry_limiter`. Adding middleware? Re-check
`test_realistic_discovery_turn_fits_in_the_recursion_limit`.

### 2.12 return_direct tools as loop exits
A tool with `return_direct=True` ends the agent loop right after it runs — no
final model call. `handoff_to_platform_guide` uses this
(`tools/clarify_tools.py:44`) so the SQL agent's run ends on the tool result.
Tools live in a registry (`tools/registry.py`); each agent gets a named subset
via `get_tools(names=...)`, and a typo'd name fails at build time
(`registry.py:36-49`).

## 3. The two-agent architecture

```
START → route_node ──┬─ small_talk           → casual_reply_node → END
                     ├─ needs_clarification* → clarify_node      → END
                     ├─ platform_help        → guide_agent       → END
                     └─ data_question        → retrieve_context_node
                          (*first turn only)          ↓
                                              sql_agent (subgraph)
                                                      ↓
                                     turn_handed_off? ─ yes → guide_agent → END
                                                      ↓ no
                                              validate_node → END
```
(`chat/turn_graph.py:1-27` and `:174-190`)

**The router is a cheap LLM call, not an agent.** `route_node` calls
`route_question` from `llm_calls/router.py` once — no tools, no loop
(`turn_graph.py:108-121`, injected at `turn_runner.py:126`). It classifies
intent and complexity; `route_decision` turns that into an edge.

**Tool split — least privilege.** Priya asking "how many surveys in
Maharashtra?" routes to `sql_agent`; "make that a KPI" routes to `guide_agent`.

| SQL agent (`SQL_AGENT_TOOLS`, `chat_data_agent.py:61-69`) | Platform guide (`GUIDE_AGENT_TOOLS`, `platform_guide_agent.py:34-51`) |
|---|---|
| list_schemas, list_tables, get_table_details, profile_column | same discovery tools (for real column names during creation) |
| execute_sql (approval-gated) | create_chart/dashboard/metric/kpi/report, add_charts_to_dashboard (all approval-gated), list_* inventory, get_dalgo_help |
| ask_user, handoff_to_platform_guide | ask_user |

The guide has **no** `execute_sql` or `profile_column`
(`platform_guide_agent.py:8-10`). The split limits blast radius, but it created
the cross-agent poisoning problem below — an agent hallucinating the *other*
agent's tool earns a "not a valid tool" error that the shared history then
shows to the agent that really has it.

**Mid-turn handoff — a conditional edge, NOT `Command.PARENT`.** When Priya
says "go ahead" to a chart offer and the router still sends it to the SQL
agent, the SQL agent calls `handoff_to_platform_guide` with a one-line summary
(prompt rule 7, `chat_data_agent.py:142-147`). Because the tool is
`return_direct=True`, the agent loop exits on the ToolMessage. Back in the
parent, `after_sql_agent` checks `turn_handed_off(state["messages"])` — does
the current turn segment contain a `handoff_to_platform_guide` ToolMessage
(`turn_graph.py:44-50`) — and routes to `guide_agent`, which reads the
"(Handing off to the platform guide: ...)" note in the shared history and
proceeds (guide prompt rule 8, `platform_guide_agent.py:99-104`). The runner
suppresses the sql_agent's `message_complete` when handed off
(`turn_runner.py:287-290`).

**Shared full message history.** Benefit: the guide sees everything the SQL
agent discovered — table names, column values, what Priya agreed to — so a
handoff needs no re-asking. Cost: cross-agent poisoning. When the SQL agent
hallucinated `create_metric`, the "Error: create_metric is not a valid tool"
ToolMessage stayed in the thread, and the guide agent — which really has
create_metric — read it as proof its own tool was broken ("I don't have write
access") (`middleware.py:103-115`). Three mitigations:
1. `repair_foreign_tool_errors` middleware — each agent durably rewrites
   invalid-tool errors that name a tool it *owns*; errors about tools it does
   NOT own stay intact, because those are true and push the erring agent to
   hand off (`middleware.py:103-151`).
2. Prompt rule 9 in both agents — "that error happened to the other assistant,
   not to you" (`chat_data_agent.py:152-155`, `platform_guide_agent.py:105-109`).
3. The router's last-responder line — `last_responder_line` appends
   "(The last answer above was written by the PLATFORM GUIDE / DATA
   ASSISTANT...)" to the router's history so short follow-ups ("yes", "make it
   monthly") route to the right agent (`turn_graph.py:53-78,111-115`).

## 4. Design rules we follow

- **Never modify prebuilt agent topology.** All customization is middleware +
  context; each AI feature is one module assembling `create_agent`
  (`chat_data_agent.py:1-8`, `middleware.py:5-6`).
- **Tools never touch the ORM.** `build_run_context()` resolves warehouse,
  schemas, and RBAC server-side; tools read only `runtime.context`
  (`context_builder.py:1-7`, `run_context.py:1-6`).
- **The checkpointer is the single source of truth for messages.** There is no
  chat-message table; the UI renders history from the checkpoint, and audit
  rows (`ChatWithDataTurnAudit`) store metadata, not transcripts
  (`turn_runner.py:329-343`).
- **Errors are written for model self-correction.** `execute_sql` returns
  warehouse errors as tool text ("Query failed: ...") instead of raising, so
  the model reads and fixes its SQL (`sql_tools.py:52-58`).
- **State repairs replace messages by id.** `repair_foreign_tool_errors`
  returns `model_copy` replacements carrying original ids; `add_messages`
  swaps them in the checkpoint once, not per call (`middleware.py:137-151`).
- **Never trust the client.** Model ids are validated against an allowlist
  (`chat_data_agent.py:94-99`); the consumer re-derives everything from the
  authenticated OrgUser.

## 5. Gotchas we hit

- **Anthropic prefill rejection.** If a handed-off thread ended on an assistant
  message, the guide agent's first Anthropic call would be rejected as
  "prefill". `handoff_to_platform_guide` is `return_direct=True` precisely so
  the turn ends on a ToolMessage (user-role for the API)
  (`clarify_tools.py:39-42`).
- **Recursion limit counts middleware.** Each hook is a graph node; adding one
  PII rule adds one step per model call. 160 was sized from a realistic messy
  discovery turn, guarded by a test (`chat_data_agent.py:31-39`).
- **Invalid-tool error poisoning.** Shared history turned one agent's
  hallucinated tool call into the other agent's false belief that its own tool
  was broken. Fixed durably in state, not per-request (`middleware.py:103-151`).
- **Langfuse tag updates REPLACE the list.** `finish()` re-sends the base tags
  plus `agent:*`, else adding a tag would wipe org/env/model tags; resumed
  turns never touch tags to preserve the original run's
  (`core/ai/tracing.py:176-179,351-373`). Also: a `GraphInterrupt` arrives as a
  chain *error* callback — it's a healthy pause and must not hit error
  dashboards (`tracing.py:233-236`).
- **MAX_SQL_ATTEMPTS budget vs discovery cost.** Raised 3 → 5: real warehouses
  with case-sensitive Airbyte tables burn 2-3 attempts on discovery before the
  query that works. The prompt states the same number so the model stops
  gracefully before the limiter forces it (`middleware.py:20-23`,
  `chat_data_agent.py:132-134`).

## 6. Where to look

All under `DDP_backend/ddpui/` unless noted.

| File | What's in it |
|---|---|
| `core/ai/chat/turn_graph.py` | TurnState, the parent StateGraph, router node, handoff edge |
| `core/ai/chat/turn_runner.py` | astream loop, event translation to WS protocol, resume, audit row |
| `core/ai/agent/chat_data_agent.py` | SQL agent: prompt, tools, middleware stack, RECURSION_LIMIT |
| `core/ai/agent/platform_guide_agent.py` | Guide agent: prompt, creation tools |
| `core/ai/agent/middleware.py` | sql_retry_limiter, trim_history, repair_foreign_tool_errors, context editing |
| `core/ai/agent/run_context.py` / `context_builder.py` | RunContext type; the only ORM-touching resolver |
| `core/ai/agent/hitl.py` | approval/question interrupts, resume payload builder |
| `core/ai/agent/checkpointer.py` | AsyncPostgresSaver + shared pool |
| `core/ai/agent/pii.py` | default + org PII rules over PIIMiddleware |
| `core/ai/tools/registry.py` | tool registration and per-agent subsetting |
| `core/ai/tools/clarify_tools.py` | ask_user, handoff_to_platform_guide |
| `core/ai/tools/sql_tools.py` | execute_sql tool body, ToolRuntime injection example |
| `core/ai/guards/sql_guard.py` | the actual AST guard: sqlglot parse tree, single-SELECT-only, forbidden-node walk, schema allowlist, LIMIT clamp |
| `core/ai/evals/` | golden-set runner: executed gold-SQL result comparison gates, Langfuse dataset runs |
| `core/ai/tracing.py` | Langfuse handler: trace per turn, spans per stage/tool |
| `websockets/chat_with_data_consumer.py` | WS auth, turn lock, pending-card Redis, resume_approval |
