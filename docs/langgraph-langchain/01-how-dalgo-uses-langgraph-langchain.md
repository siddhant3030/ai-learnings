# How Dalgo Uses LangGraph & LangChain

*An engineer's map of the chat-with-data implementation in `DDP_backend/ddpui/core/chat_with_data/`. All file references are relative to `DDP_backend/ddpui/` unless noted. Analyzed against the code as of July 2026.*

---

## TL;DR

Dalgo's chat-with-data feature is built on **LangChain 1.x's prebuilt agent loop** (`create_agent`), not on a hand-assembled `StateGraph`. The graph topology is deliberately never touched — one model node bound to nine tools, looping until the model stops calling tools. Every customization goes through the three sanctioned extension points: **middleware** (dynamic per-org system prompt, history trimming, a deterministic SQL-retry kill switch, context editing), **runtime context** (`RunContext` dataclass injected into tools via `ToolRuntime` so the model never sees org identifiers or credentials), and **the tool registry** (adding a capability = adding a module; the graph never changes).

Persistence is LangGraph's **`AsyncPostgresSaver`** checkpointer in the app database (one shared psycopg3 async pool per process); one chat session = one `thread_id`. Streaming uses `astream(stream_mode=["messages", "updates"])`, translated by a transport-independent runner into a typed WebSocket event protocol. Around the agent sit four **bare `ChatAnthropic` utility calls that live outside the graph entirely**: a Haiku router (intent/complexity), a pre-execution SQL reflection check (complex lane only), a post-execution result validator, and a session-title generator — all fail-open by design.

Observability is a **hand-rolled `BaseCallbackHandler` feeding a self-hosted Langfuse** (v2 SDK, because the dbt stack pins `protobuf<5` which blocks Langfuse v3, and the v2 SDK's bundled LangChain handler predates langchain 1.x). No LangSmith, no interrupts, no subgraphs, no Send API, no Store — most of those absences are justified for a single-agent product; a couple (interrupt-based confirmation, checkpoint cleanup, structured output) are worth revisiting.

**Version pins** (`DDP_backend/pyproject.toml:274-277`): `langgraph==1.2.7`, `langchain==1.3.11`, `langchain-anthropic==1.4.8`, `langgraph-checkpoint-postgres==3.1.0`, plus `sqlglot==30.12.0` for the SQL guard and `psycopg[binary,pool]>=3.3.4` for the checkpointer. Notably, **`langfuse` is not pinned anywhere** — it's a lazy optional import; tracing is silently off if it isn't installed.

There are **zero langchain/langgraph imports anywhere else in DDP_backend** — the framework's blast radius is fully contained inside `core/chat_with_data/` plus its two transports (`websockets/chat_with_data_consumer.py`, `api/chat_with_data_api.py`) and the REPL management command.

---

## Architecture Overview

### The cast

| Layer | File(s) | Role |
|---|---|---|
| Transports | `websockets/chat_with_data_consumer.py`, `api/chat_with_data_api.py`, `management/commands/chat_with_data_repl.py` | WS chat turns; REST session CRUD + history replay; terminal REPL |
| Turn runner | `core/chat_with_data/runner.py` | Runs one turn, yields typed events, writes the audit row |
| Agent | `core/chat_with_data/agent.py` | Compiles `create_agent` with tools + middleware + checkpointer |
| Middleware | `core/chat_with_data/middleware.py` | Dynamic prompt, history trim, SQL-retry limiter, context editing |
| Context | `core/chat_with_data/state.py`, `context.py`, `prompts.py` | `RunContext` dataclass; the only ORM-touching resolver; per-org system prompt |
| Tools | `core/chat_with_data/tools/` (registry + 5 modules, 9 tools) | schema discovery, column profiling, guarded SQL, charts, dashboards |
| Guards | `core/chat_with_data/guards/sql_guard.py` | sqlglot AST validation: SELECT-only, schema allowlist, LIMIT clamp |
| Sidecar LLM calls | `router.py`, `reflection.py`, `validator.py`, `titles.py` | Plain `ChatAnthropic` calls outside the graph, all fail-open |
| Persistence | `core/chat_with_data/checkpointer.py`, `history.py` | Shared `AsyncPostgresSaver`; checkpoint → UI-bubble replay |
| Observability | `core/chat_with_data/observability.py` | Custom callback handler → self-hosted Langfuse |

### The flow of one turn

```
 WebSocket message  {"action": "send_message", "message": "..."}
        │
        ▼
 ChatWithDataConsumer (websockets/chat_with_data_consumer.py)
   auth (cookie JWT) → RBAC perm → feature flag → llm_optin → rate limit
   → Redis turn lock → build_run_context(orguser)   [only ORM touch]
        │
        ▼
 run_turn() (runner.py) ── transport-independent event generator
        │
        ├─ STAGE 1: route_question()  [router.py — bare Haiku call, fail-open]
        │     intent: small_talk ──────────────► casual_reply(), then
        │     intent: needs_clarification        agent.aupdate_state() writes the
        │        (first turn only) ────────────► exchange into the thread; audit; END
        │     intent: data_question ─ continue (context.complexity tagged)
        │
        ▼
 agent.astream({"messages":[("user", q)]},
               config={thread_id, recursion_limit=25, callbacks=[langfuse]},
               context=RunContext, stream_mode=["messages","updates"])
        │
        │   ┌──────────── the create_agent loop (topology never modified) ─────────┐
        │   │                                                                      │
        │   │   before_model middleware (in order):                                │
        │   │     sql_retry_limiter  ── ≥3 failed SQL? append apology, jump_to=end │
        │   │     org_system_prompt  ── @dynamic_prompt from RunContext            │
        │   │     trim_history       ── trim_messages → llm_input_messages         │
        │   │     ContextEditingMiddleware(ClearToolUsesEdit)                      │
        │   │                    │                                                 │
        │   │                    ▼                                                 │
        │   │            ChatAnthropic (claude-sonnet-5, max_tokens=4096)          │
        │   │                    │                                                 │
        │   │         tool_calls? ──no──► END (final AIMessage)                    │
        │   │              │yes                                                    │
        │   │              ▼                                                       │
        │   │           ToolNode ── ToolRuntime[RunContext] injected               │
        │   │   list_schemas / list_tables / get_table_details / profile_column   │
        │   │   execute_sql ──► sql_guard (sqlglot AST) ──► reflection (complex   │
        │   │                    lane only, inside the tool) ──► warehouse        │
        │   │   create_chart / list_dashboards / create_dashboard /               │
        │   │   add_charts_to_dashboard   (Dalgo metadata only; RBAC via context) │
        │   │              │                                                       │
        │   │              └────────── loop back to model ────────────────────────┘
        │   └──────────────────────────────────────────────────────────────────────┘
        │
        │   events out, as the stream runs:
        │     "messages" mode → AIMessageChunk → {"type":"token", ...}
        │     "updates"  mode → AIMessage.tool_calls → {"type":"tool_start", ...}
        │                       ToolMessage.artifact → {"type":"tool_end", ...}
        ▼
 {"type":"message_complete", message, result_table, charts, usage}
        │
        ├─ STAGE 5b: validate_turn()  [validator.py — off critical path]
        │     → {"type":"validation", verdict, assumptions, caveat}
        │     → Langfuse score("result_validation")
        │
        └─ finally: ChatWithDataTurnAudit row (always), Langfuse trace.finish()
             consumer: generate_session_title() on first exchange
```

Persistence runs underneath the whole loop: every state transition is checkpointed to Postgres under the session's `thread_id`, so conversation memory across turns is entirely LangGraph's — the app never serializes messages itself.

---

## Concept-by-Concept Usage Map

### 1. `create_agent` — the prebuilt agent loop

**Where:** `agent.py:47-58`.
**How:** `create_agent(model=ChatAnthropic(...), tools=get_tools(), middleware=[...], context_schema=RunContext, checkpointer=...)`. This is langchain 1.x's successor to `langgraph.prebuilt.create_react_agent`, and it's the *only* graph in the codebase.
**Why:** The module docstring is explicit: "one model node bound to the tool registry, one ToolNode, loop until the model stops calling tools. All customization is middleware + context; the topology is never modified." This is a deliberate architectural bet — accept the prebuilt loop's shape, get the framework's streaming/checkpointing/tool-execution machinery for free, and channel all product logic into middleware, context, and tools. It keeps the LangGraph surface area tiny and upgrade-friendly.

The compiled agent is rebuilt per WebSocket turn (`chat_with_data_consumer.py:123-124`) — cheap, since compilation is in-memory; only the checkpointer pool is process-global.

### 2. Middleware — the sanctioned customization layer

All four middleware live in `middleware.py` and are registered in order at `agent.py:50-55` (order matters: the limiter must run first because it can end the run).

**`@dynamic_prompt` — per-org system prompt** (`middleware.py:46-49`). Rebuilds the system prompt on *every model call* from `request.runtime.context` via `build_system_prompt()` (`prompts.py:14-64`). Why: the prompt embeds the org's SQL dialect, schema allowlist, and row cap — none of which can be baked in at agent-build time, and all of which must be current even mid-conversation.

**`@before_model(can_jump_to=["end"])` — deterministic SQL-retry kill switch** (`middleware.py:52-73`). Counts `ToolMessage`s named `execute_sql` whose content starts with `"Query failed:"` / `"SQL rejected:"` since the last `HumanMessage`; at 3 failures (`MAX_SQL_ATTEMPTS`, `prompts.py:11`) it appends a canned apology `AIMessage` and returns `{"jump_to": "end"}`. Why: the system prompt *asks* the model to stop after 3 attempts, but prompts are suggestions — this middleware makes the limit deterministic. This is the one place the code uses LangGraph's conditional-jump capability, and it's a nice example of using middleware instead of adding a graph node.

**`@before_model` — history trimming** (`middleware.py:76-94`). Uses langchain-core's `trim_messages(token_counter=count_tokens_approximately, max_tokens=60_000, start_on="human", include_system=True)` and returns the result under the **`llm_input_messages`** key. Why the key matters: it trims *only what the model sees on this call* — the full conversation stays in the checkpoint, so the UI can render complete history and later turns still have everything.

**`ContextEditingMiddleware(ClearToolUsesEdit(trigger=40_000, keep=5))`** (`middleware.py:97-106`). Framework-provided middleware that drops bulky old tool results (query result tables are the big offender) from the request once total context passes 40k tokens, always keeping the 5 most recent. Why: query results dominate token growth in a data-chat; this is the Anthropic "context editing" pattern via langchain's built-in rather than custom code.

### 3. Runtime context — `context_schema=RunContext` + `ToolRuntime`

**Where:** `state.py:12-32` (the dataclass), `agent.py:56` (`context_schema=RunContext`), `runner.py:122-126` (`context=context` passed to `astream`), every tool signature (e.g. `tools/sql_tools.py:23` — `def execute_sql(sql: str, runtime: ToolRuntime[RunContext])`).
**How:** `RunContext` carries `org_id`, `org_slug`, `dialect`, `allowed_schemas`, `max_result_rows`, `query_timeout_s`, a live warehouse client, `orguser_id`, pre-resolved RBAC booleans (`can_create_charts`, `can_create_dashboards`), and two per-turn fields the runner sets from router output (`question`, `complexity`). Tools receive it through the `ToolRuntime` injected parameter — LangChain excludes that parameter from the tool schema the model sees.
**Why:** This is the security spine of the feature. The context is resolved server-side in exactly one place (`context.py:46-78`, `build_run_context`) — the only code in the agent that touches the ORM or Secrets Manager — and *the model can never see or supply any of these values*. A prompt-injected "query org 42's schema" is structurally impossible: tools take no org parameter. RBAC is likewise resolved at context-build time so tools never query permission tables.

### 4. Tools — `@tool`, `content_and_artifact`, and the registry

**Where:** `tools/registry.py` (the extension point), `tools/schema_tools.py` (`list_schemas`, `list_tables`, `get_table_details`), `tools/profile_tools.py` (`profile_column`), `tools/sql_tools.py` (`execute_sql`), `tools/chart_tools.py` (`create_chart`), `tools/dashboard_tools.py` (`list_dashboards`, `create_dashboard`, `add_charts_to_dashboard`).

**The registry** (`tools/registry.py:25-35`): `@register_tool` adds each `BaseTool` to a module-level dict at import time; `get_tools()` lazily imports the tool modules and returns the values. Adding a capability = one new module + one line in `_TOOL_MODULES`. The docstring calls this out as the spec's extension point: "the agent graph never changes."

**`@tool(response_format="content_and_artifact")`** (`tools/sql_tools.py:22`, `tools/chart_tools.py:37`, `tools/dashboard_tools.py:154,184`): these tools return `(content, artifact)` tuples. The **content** is a compact, token-frugal text rendering for the LLM (pipe-separated rows, cells truncated to 120 chars, `tools/common.py:60-78`); the **artifact** is the full structured payload (`{sql, status, row_count, columns, rows}` for SQL; `{type, chart_id, title, url_path}` for charts/dashboards) carried on the `ToolMessage.artifact` field **without ever entering the model's context**. Why this matters: it's the mechanism that lets the UI render a real result table and clickable chart chips while the model sees only a summary — the single most leveraged LangChain feature in the codebase. The runner (`runner.py:149-187`) and the history replayer (`history.py:25-46`) both read artifacts off `ToolMessage`s.

**Docstrings as prompt engineering:** every tool docstring is written *at the model* ("ALWAYS use this before filtering on a text column…", "If this returns an error, read it carefully, fix the SQL… and try again") — LangChain turns docstrings into the tool descriptions the model sees.

**Errors as feedback, not exceptions:** `execute_sql` never raises to the graph. Guard rejections return `"SQL rejected: …"`, warehouse failures return `"Query failed: <first line, 500 chars>"` — both as tool content the model reads and self-corrects on (`tools/sql_tools.py:36-59`). This deliberately opts out of LangGraph's tool-error handling in favor of the model-as-error-handler pattern; the middleware retry limiter is the deterministic backstop. Note the layering inside `execute_sql`: sqlglot AST guard (every query) → LLM reflection gate (complex-lane only, `tools/sql_tools.py:42-49`) → execution with `statement_timeout` (Postgres only; `tools/sql_tools.py:75-88`).

**Sync tools on purpose:** tools use the sync ORM and sync warehouse clients; comments (`tools/chart_tools.py:22-24`, `tools/dashboard_tools.py:52`) note that LangGraph executes sync tools in a worker thread, so blocking there is safe from the ASGI event loop's perspective.

### 5. Checkpointing — `AsyncPostgresSaver` as the conversation memory

**Where:** `checkpointer.py` (whole file), wired at `chat_with_data_consumer.py:123-124`, thread selection at `runner.py:105-107` (`config={"configurable": {"thread_id": str(session.thread_id)}}`).
**How:** One process-wide `AsyncPostgresSaver` over a shared psycopg3 `AsyncConnectionPool` (min 1 / max 4, autocommit, `dict_row`), created lazily inside the running event loop behind an `asyncio.Lock` — explicitly never per-message. It lives in the app's own Postgres database, alongside but separate from Django's psycopg2 connections. Tables are created once by `manage.py chat_with_data_setup` → `setup_tables()` (`checkpointer.py:66-70`), which uses `AsyncPostgresSaver.from_conn_string` on its own connection so it can run from a management command's event loop.
**Why:** The checkpointer *is* the memory system. The app stores no message JSON of its own — `ChatWithDataSession` holds only a `thread_id` pointer plus title metadata, and multi-turn context, UI history replay, and the router's history tail all come from checkpoints. The pool is small because a connection is held only during state reads/writes, not for the duration of a turn.

### 6. Streaming — `astream` with dual `stream_mode`

**Where:** `runner.py:122-194`.
**How:** `agent.astream(..., stream_mode=["messages", "updates"])`, iterating `(mode, chunk)` pairs. `"messages"` mode yields `AIMessageChunk`s → token events (after `extract_text()` strips thinking blocks — see §8). `"updates"` mode yields node-level state deltas, from which the runner derives: `tool_start` events (from `AIMessage.tool_calls`, including the SQL text so the UI can show the query as it runs, plus plain-language labels like "Looking at your tables…" from `TOOL_LABELS`, `runner.py:40-51`), `tool_end` events with success/error status (from `ToolMessage.artifact`), the final answer text, accumulated `usage_metadata` token counts, executed SQL for the audit row, and created-chart chips.
**Why:** Tokens alone aren't enough for the UX (non-technical users need activity labels while tools run), and updates alone can't stream prose. The docs (`docs/docs/features/chat-with-data-dev.md:102-103`) note a third mode they *wanted*: `"custom"` stream mode for tool-progress events, blocked because the repo floor is Python 3.10 and custom-mode-from-async-tools needs 3.11.

### 7. Direct state access — `aget_state` / `aupdate_state` / `aget_tuple`

Three places bypass the run loop and talk to graph state directly:

- **`agent.aget_state(config)`** (`runner.py:301-325`, `_thread_tail`): reads the thread's messages to build a compact 6-line "User:/Assistant:" tail for the router, so follow-ups saying "this"/"that" aren't misrouted as ambiguous. Fails to `[]` (treated as first turn).
- **`agent.aupdate_state(config, {"messages": [HumanMessage, AIMessage]})`** (`runner.py:271-277`, `_short_circuit_turn`): when the router diverts a turn (small talk / clarification), the exchange is manually appended to the checkpointed thread so the *next* turn's agent still sees it. This is the "write to history without running the graph" pattern — the diverted turn cost zero agent tokens but stays part of the conversation, and the turn is still audited with `tools_called=[]` so routing effectiveness stays measurable.
- **`saver.aget_tuple(config)`** (`history.py:65-72`): the REST history endpoint (`api/chat_with_data_api.py:76-87`) reads the latest checkpoint **straight off the saver**, no compiled graph needed, then reaches into `checkpoint["channel_values"]["messages"]` and collapses the raw message list into UI bubbles (`map_messages`, `history.py:15-62` — SQL/chart artifacts attach to the next assistant bubble; tool chatter is hidden; thinking-only AIMessages are skipped).

### 8. Messages and content blocks

**Where:** message-type dispatch throughout `runner.py`, `history.py`, `middleware.py`; `content.py` (whole file).
**How:** Standard `HumanMessage` / `AIMessage` / `ToolMessage` / `AIMessageChunk` handling, plus one Dalgo-specific wrinkle: `extract_text()` (`content.py:13-26`) reduces block-list content to its text blocks only. Claude models with thinking enabled (default on claude-sonnet-5) return content as `[{"type":"thinking", ..., "signature": ...}, {"type":"text", ...}]`; the signature must stay in the stored message (replays require it) but must never reach the user. Every user-facing or router-facing read of `message.content` in the codebase goes through this helper.
**Why:** Without it, thinking blocks and signatures would leak into the chat UI, router prompts, and Langfuse traces.

### 9. Bare `ChatAnthropic` — the four sidecar calls outside the graph

**Where:** `router.py:95-137` (`route_question`, Haiku, 300 max tokens), `router.py:84-92` (`casual_reply`), `reflection.py:49-67` (`check_sql`, *sync on purpose* — it runs inside the `execute_sql` tool, which LangGraph executes in a worker thread), `validator.py:87-123` (`validate_turn`, async, post-`message_complete`), `titles.py:34-45` (`generate_session_title`).
**How:** Plain `model.ainvoke(prompt_string)` / `model.invoke(...)` with hand-rolled JSON parsing (strip code fences, `json.loads`, validate enums, fall back on any failure). No chains, no `with_structured_output`, no output parsers.
**Why:** These are cheap single-shot classifier/judge calls where a graph would be overhead. All four are **fail-open**: the router falls back to `data_question/simple` (it may only ever divert obviously-non-data turns, never block a real question), reflection falls back to "no issue", the validator returns `None`, titles keep the default. The trade-off of skipping `with_structured_output` is discussed under Risks.

### 10. Callbacks — custom `BaseCallbackHandler` for Langfuse

**Where:** `observability.py:64-187`, attached per-turn at `runner.py:109-110` via `config["callbacks"] = [trace_handler]`.
**How:** `LangfuseTurnHandler(BaseCallbackHandler)` with `raise_error = False` maps LangChain callbacks onto a flat Langfuse trace: `on_chat_model_start`/`on_llm_end` → generations (with token usage from `usage_metadata`), `on_tool_start`/`on_tool_end` → spans, everything clipped to 4k chars and wrapped in try/except so a tracing failure can never break a turn. `trace.score()` records the validator's verdict as an eval score (`runner.py:213-218`); `finish()` stamps final output and status. Traces are tagged `org_slug`/`dialect`, keyed by opaque ids (never emails), and carry `request_uuid` to join with the `ChatWithDataTurnAudit` row.
**Why hand-rolled:** the dbt 1.8 stack pins `protobuf==4.25.3` (`pyproject.toml:187`), which rules out the Langfuse v3 SDK (OpenTelemetry needs protobuf 5), and the v2 SDK's bundled LangChain handler imports pre-1.x langchain modules. So Dalgo uses the v2 SDK's low-level client behind ~120 lines of its own handler. Self-hosted Langfuse is a privacy requirement: traces contain prompts *and query results*.

### 11. `recursion_limit` — the runaway-loop backstop

**Where:** `agent.py:25` (`RECURSION_LIMIT = 25`), applied at `runner.py:107`. Caps model⇄tool loop steps per turn. The retry-limiter middleware usually ends things much earlier; this is the hard framework-level ceiling.

---

## What We're NOT Using — and Whether It Matters

**Raw `StateGraph` / custom nodes.** Deliberately avoided; the prebuilt loop plus middleware covers everything so far. *Doesn't matter today* — and it's the main reason the LangGraph coupling is so shallow. It becomes a constraint only if the flow stops being "one agent with tools" (e.g. a separate planner node, or parallel per-table exploration).

**`with_structured_output` / structured output.** The four sidecar calls parse JSON by hand with code-fence stripping. Matters *somewhat*: `ChatAnthropic.with_structured_output` (or native structured outputs) would delete the brittle parsing and the enum re-validation in `router.py:114-134`, `reflection.py:58-64`, `validator.py:108-120`. The fail-open design caps the blast radius (a parse failure = a fail-open route, not an error), but every silent fail-open is also a silently skipped safety check. Low-effort, real win.

**`interrupt()` / human-in-the-loop.** The dashboard flow needs user confirmation ("add to an existing dashboard or create a new one?"), and it's implemented *conversationally*: the system prompt orders the agent to call `list_dashboards`, ask, and act only next turn (`prompts.py:50-52`, `tools/dashboard_tools.py:139-141,162-164`). LangGraph's `interrupt()` would make this a structural guarantee instead of a prompt-obedience hope — a model that ignores the instruction *can* create a dashboard without asking; the only hard checks are the RBAC booleans. Matters *moderately today, more with every write-capable tool added*. Since turns already run over a WebSocket with a checkpointer, the plumbing for interrupts is largely in place.

**Subgraphs / Send API / multi-agent.** Not needed for one agent with nine tools. Send-style parallelism could someday speed up multi-table discovery, but it's correctly absent now.

**Durable execution / retry policies / node caching.** A turn that dies mid-run (deploy, crash) just fails; the user retries. Fine for chat — turns are seconds-long and mostly read-only (chart creation is the exception and would re-create on retry). Not worth the complexity yet.

**Time travel / checkpoint history.** Only the *latest* checkpoint is ever read (`history.py:68`). The full checkpoint lineage LangGraph stores is unused — which also means it's pure storage cost (see Risks).

**Store (cross-thread memory).** No long-term memory across sessions — each thread is an island. Matters *eventually*: "remember that 'the program' means X for this org" is a natural ask, and LangGraph's Store is the intended home. Not a v1 gap.

**LangSmith.** Deliberately replaced by self-hosted Langfuse for data sovereignty (NGO warehouse data appears in traces and must not leave the deployment). The right call; the cost was the hand-rolled handler (§10).

**`"custom"` stream mode.** Wanted but blocked by the Python 3.10 floor (`pyproject.toml:6`, documented in `chat-with-data-dev.md:102-103`). Tool progress currently comes only from `updates` mode, so a long-running query shows a static label. Matters *a little*; fixed by raising the floor to 3.11+.

**LangChain chains / retrievers / RAG.** No semantic-layer retrieval — schema discovery is live tool calls against the warehouse catalog every turn. That's a product-architecture choice (freshness over token cost), not a framework gap, but worth knowing: nothing is cached between turns except what sits in message history.

---

## Risks and Smells

1. **`langfuse` is imported but not declared.** `observability.py:43` lazily imports it; `pyproject.toml` has no langfuse entry at all. A deployment that forgets to install it gets tracing silently disabled (one log line at init). Either pin `langfuse<3` as an optional extra or make the absence louder.

2. **Langfuse v2 SDK is a dead end.** It's legacy; the code depends on "plain HTTP, still accepted by current Langfuse servers" (`observability.py:5-7`). The real root cause is dbt 1.8's `protobuf<5` pin — when dbt is upgraded, the ~120-line custom handler and the v2 client should both be revisited. Until then this is a known, documented, but ticking dependency.

3. **`history.py` reaches into checkpoint internals.** `checkpoint_tuple.checkpoint.get("channel_values", {}).get("messages", [])` (`history.py:71`) depends on LangGraph's internal checkpoint dict layout rather than a public state API (the runner's `_thread_tail` does it properly via `agent.aget_state`). A `langgraph-checkpoint` upgrade could break history replay silently. Cheap fix: use `aget_state` here too, or add a contract test.

4. **No checkpoint lifecycle management.** `delete_session` soft-deletes the Django row (`chat_with_data_service.py:75-77`) but never calls `checkpointer.adelete_thread()` — orphaned conversation data (including query results embedded in ToolMessages) lives in the checkpoint tables forever, and every turn's full checkpoint lineage accrues. For a privacy-conscious product this is both a storage and a data-retention smell.

5. **Exact pins on a fast-moving 1.x line.** `langgraph==1.2.7` / `langchain==1.3.11` exact-pinning is good for reproducibility, but langchain 1.x middleware APIs (`@dynamic_prompt`, `before_model`'s `can_jump_to`/`llm_input_messages` contract, `ContextEditingMiddleware`) are young; upgrades need the agent-loop tests (scripted fake model, per the dev guide) run deliberately. No deprecated APIs are in use today — `create_agent` is the current replacement for `create_react_agent`.

6. **The retry limiter couples to tool-output prefixes.** `count_failed_sql_attempts` string-matches `("Query failed:", "SQL rejected:")` (`middleware.py:27-43`) against `execute_sql` content. Anyone rewording those messages in `tools/sql_tools.py` silently disables the kill switch. The artifact already carries `status` ∈ {rejected, error, success} — counting from that structured field (or a small state counter) would be sturdier.

7. **Checkpointer pool is loop-bound and process-global.** Documented (`checkpointer.py:40-41`), but any code calling `get_checkpointer()` from a different event loop (management commands, task workers) gets undefined behavior. `setup_tables()` correctly sidesteps this with its own connection; the constraint is fine as long as everyone remembers it.

8. **Duplicated final-answer extraction.** The runner reconstructs the final message, usage, and artifacts from streamed `updates` (`runner.py:138-194`) instead of reading the run's final state once. It works, but it's the kind of stream-parsing code that breaks subtly when message shapes change (the earlier thinking-blocks surprise that `content.py` exists to handle is the cautionary tale). A post-run `aget_state` cross-check would be cheap insurance.

9. **Identifier interpolation trusts the allowlist.** `tools/common.py:26-40` interpolates schema names into catalog SQL after allowlist validation, and `profile_column` quotes identifiers manually. Safe *because* schemas come from the server-derived allowlist and tables from the live catalog — but the pattern must not be copy-pasted into a tool that accepts freer input. (The main `execute_sql` path is properly AST-guarded by sqlglot.)

10. **Fail-open everywhere is a double-edged sword.** Router, reflection, validator, titles, tracing, audit writes — every auxiliary path swallows exceptions. Correct for availability, but degraded runs (no reflection, no validation, no traces) look identical to healthy ones except in logs. A counter/metric on fail-open events would make silent degradation visible.

11. **Per-turn mutation of `RunContext`.** `question` and `complexity` are stamped onto the context by the runner (`runner.py:76-77`) so the reflection gate inside `execute_sql` can see them. It works because a fresh context is built per turn, but it blurs "immutable run configuration" with "per-turn state" — the kind of thing LangGraph state (or tool-call middleware) is actually for.

---

## Open Questions

- **Should the dashboard confirm step become an `interrupt()`?** Prompt-enforced confirmation is the softest guarantee in the write path. As artifact-creating tools multiply (exports? scheduled reports?), a structural confirm gate seems inevitable — is the WS protocol ready to carry a resume action?
- **When does the Python 3.10 floor move?** It blocks `"custom"` stream mode (tool progress UX) and is the stated reason for at least one design compromise. What else in DDP_backend holds it back?
- **What's the checkpoint retention policy?** Nothing prunes checkpoint lineage or deletes threads for soft-deleted sessions. Is there a compliance requirement (e.g. `llm_optin` revocation) that should trigger `adelete_thread`?
- **Reflection placement:** `check_sql` runs synchronously *inside* the tool, adding Haiku latency to every complex-lane query. Was a tool-call-middleware placement considered — which would also remove the per-turn `RunContext` mutation (smell #11)?
- **Eval loop:** the Langfuse `result_validation` score and the `ChatWithDataTurnAudit` table are the raw material for the eval scorecard the design docs call for — is anything consuming them yet, or is the loop still open?
- **Upgrade drill:** who owns re-running the scripted-fake-model agent tests against a langchain/langgraph bump, and is there a canary for the `history.py` checkpoint-internals dependency (smell #3)?
