# LangGraph Core Concepts — The Definitive Guide

**Audience:** data-platform engineers building a production chat-with-data feature in Python.
**Version context:** LangGraph 1.x (1.0 GA'd October 2025; 1.2.x is current as of mid-2026). The core graph API has been stable since ~0.2 and 1.0 explicitly froze it — everything below is the durable API surface, with version notes where things moved.

---

## TL;DR

- LangGraph is a **low-level orchestration runtime** for stateful, long-running LLM applications. It is not "LangChain v2" and does not require LangChain abstractions — nodes are plain Python functions.
- The unit of design is a **StateGraph**: a typed shared state + nodes (functions) + edges (routing, possibly conditional). You choose exactly where the LLM gets control and where it doesn't.
- The killer features for production are not the graph itself but what the runtime gives you *around* the graph: **checkpointing/persistence** (resume, fault tolerance, time travel), **`interrupt()` human-in-the-loop**, **first-class streaming**, **Send-API fan-out**, and **durable execution**.
- Everything hard hinges on the **checkpointer**. If you compile without one, you lose memory, HITL, resume, and time travel. For production: `PostgresSaver`, not `MemorySaver`.
- In 1.x, `langgraph.prebuilt.create_react_agent` is **deprecated** in favor of `langchain.agents.create_agent` (which is itself built on LangGraph, adds middleware). The graph primitives are unchanged.
- For a chat-with-data pipeline (mostly deterministic stages: intent → schema retrieval → SQL gen → validate → execute → summarize), LangGraph earns its keep through checkpointing, HITL query approval, streaming, and bounded self-correction loops — **not** through autonomous agent behavior. If your flow is a straight line with no persistence or interrupts, plain Python is genuinely fine.

---

## 1. The mental model: graphs vs chains vs autonomous agents

There is a spectrum of "how much control does the LLM have over control flow":

| Style | Control flow decided by | Example |
|---|---|---|
| **Chain / pipeline** | You, fully static | prompt → LLM → parse → done |
| **Workflow / graph** | You, with branches/loops the LLM can *influence* at defined decision points | SQL agent: generate → validate → (retry \| execute) → summarize |
| **Autonomous agent** | The LLM, in a loop (ReAct: think → call tool → observe → repeat) | "here are 20 tools, figure it out" |

LangGraph's bet: production systems live in the middle. Pure chains can't express retries, loops, or human approval. Pure autonomous agents are unreliable, unbounded in cost/latency, and hard to debug. A **graph** lets you make most transitions deterministic and confine LLM discretion to specific conditional edges — which is exactly the "fixed pipeline with bounded LLM decision points" shape a chat-with-data system wants.

**Why "agent runtime" and not "framework"?** LangGraph deliberately ships almost no prompts, no chains, no retrieval logic. What it provides is an execution engine (internally called **Pregel**, after Google's graph-processing model): it runs your nodes in discrete **super-steps** — all nodes with pending inputs run (in parallel where possible), their state writes are merged via reducers, a checkpoint is written, repeat until no node has pending input. That message-passing model is what makes parallelism, checkpointing, interrupts, and replay fall out naturally instead of being bolted on.

**Pitfall:** don't conflate LangGraph with LangChain. You can (and many teams do) use LangGraph with raw provider SDKs (`anthropic`, `openai`) inside nodes. You only need `langchain-core` message types if you want `add_messages`/`MessagesState` and the `messages` streaming mode.

---

## 2. StateGraph: state, reducers, nodes, edges

The core API. Everything else builds on this.

### 2.1 State schemas

State is a single shared, typed structure that flows through the graph. Define it as a `TypedDict` (most common), a Pydantic `BaseModel` (adds runtime validation on inputs to the graph), or a dataclass.

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class ChatState(TypedDict):
    messages: Annotated[list, add_messages]   # reducer: append/merge by ID
    user_question: str
    sql: str
    query_result: list[dict]
    retry_count: int
    error: str | None
```

Notes:
- **TypedDict** is zero-overhead; **Pydantic** validates input coming *into* the graph but nodes still exchange plain dicts internally — don't rely on validation between nodes.
- You can declare separate `input_schema=` / `output_schema=` on the `StateGraph` constructor to keep internal working keys (e.g. `retry_count`) out of your public API, and "private" state channels shared only between specific nodes.
- **Warning:** private/internal channels are *not* redacted from streaming output — filter with `output_keys` or in your API layer if state holds sensitive intermediate data (relevant if you carry raw query results or PII in state).

### 2.2 Reducers

Each state key has a **reducer** that defines how a node's returned update merges into the existing value. Default: overwrite. Annotate to change it:

```python
import operator
from typing import Annotated

class State(TypedDict):
    sql: str                                     # default: last write wins
    candidate_queries: Annotated[list[str], operator.add]  # append
    messages: Annotated[list, add_messages]      # smart message merge

# custom reducer
def merge_dicts(left: dict, right: dict) -> dict:
    return {**left, **right}

class S2(TypedDict):
    table_metadata: Annotated[dict, merge_dicts]
```

`add_messages` is the important prebuilt one: it appends new messages, but **updates in place** when a message with the same ID arrives (which enables editing history, and is how streaming/token accumulation works), and deserializes dicts into LangChain message objects. `MessagesState` is a prebuilt TypedDict with just that key — subclass it and add your own fields.

**Why reducers matter:** when two nodes run in parallel in the same super-step and both write the same key, the reducer is what makes that well-defined (e.g. `operator.add` collects both). With the default overwrite reducer, concurrent writes to the same key raise an `InvalidUpdateError`. If you fan out (Send API, parallel branches), every key written by parallel nodes **must** have an accumulating reducer.

### 2.3 Nodes

Nodes are plain functions (sync or async): take state, return a **partial** state update (only the keys you changed).

```python
def generate_sql(state: ChatState) -> dict:
    resp = llm.invoke(build_sql_prompt(state["user_question"]))
    return {"sql": resp.content}          # partial update, merged via reducers

builder = StateGraph(ChatState)
builder.add_node("generate_sql", generate_sql)
```

Nodes can optionally accept a second `config: RunnableConfig` arg (thread ID, tracing metadata) and, in 1.x, a `runtime` parameter exposing a typed **context schema** — the sanctioned way to inject per-request dependencies (DB connections, user/org ID, model choice) without stuffing them into state:

```python
from dataclasses import dataclass
from langgraph.runtime import Runtime

@dataclass
class Ctx:
    org_id: str
    warehouse_conn: object

def execute_query(state: ChatState, runtime: Runtime[Ctx]) -> dict:
    rows = runtime.context.warehouse_conn.run(state["sql"])
    return {"query_result": rows}

builder = StateGraph(ChatState, context_schema=Ctx)
# invoke: graph.invoke(inputs, config, context=Ctx(org_id=..., warehouse_conn=...))
```

**Pitfalls:**
- Return `{}`-style partial updates, not the full state. Returning the whole state works but fights reducers (you'll double-append lists).
- Nodes should be **idempotent-ish**: on resume after failure or interrupt, the node that was in flight re-runs *from the top*. Side effects (INSERTs, emails, non-idempotent API calls) before the failure point will happen again. Use upserts, or wrap side effects in `@task` (see §9).

### 2.4 Edges, conditional edges, START/END

```python
builder.add_edge(START, "generate_sql")           # entry point
builder.add_edge("generate_sql", "validate_sql")  # unconditional

def route_after_validation(state: ChatState) -> str:
    if state["error"] is None:
        return "execute"
    if state["retry_count"] >= 2:
        return "give_up"          # bounded self-correction
    return "generate_sql"         # loop back with error feedback

builder.add_conditional_edges(
    "validate_sql",
    route_after_validation,
    {"execute": "execute_query", "generate_sql": "generate_sql", "give_up": END},
)
builder.add_edge("execute_query", "summarize")
builder.add_edge("summarize", END)

graph = builder.compile()   # compiling is mandatory
result = graph.invoke({"user_question": "monthly active users by region?", "retry_count": 0})
```

- `START` / `END` are virtual nodes. Multiple edges *out* of a node = parallel fan-out (both run next super-step). Conditional edges return the name(s) of next node(s); the mapping dict is optional but makes the graph drawable.
- **Loops are just edges pointing backward.** The `recursion_limit` (default 25 super-steps, set via config) is your guard against infinite loops — it raises `GraphRecursionError`. For chat-with-data, cap SQL-repair loops explicitly with a counter in state rather than relying on recursion_limit. There's also a `RemainingSteps` managed value you can put in state to wind down gracefully before the limit hits.
- `add_node(..., defer=True)` defers a node until all upstream branches complete — useful for a "join" node after parallel fan-out.

**Pitfall:** routing functions must be cheap and pure — they run on every super-step transition and are not checkpointed as separate steps. Do the LLM call in a node, put its decision in state, and route on state.

---

## 3. Command: state update + goto in one step

Normally nodes update state and edges route. `Command` lets a node do both — useful when the routing decision and the update come from the same computation (e.g. an LLM call that both produces content and decides what's next), and essential for multi-agent handoffs.

```python
from typing import Literal
from langgraph.types import Command

def triage(state: ChatState) -> Command[Literal["generate_sql", "clarify"]]:
    decision = classify(state["user_question"])
    if decision.ambiguous:
        return Command(update={"clarification": decision.question}, goto="clarify")
    return Command(update={"intent": decision.intent}, goto="generate_sql")
```

- The `Command[Literal[...]]` return annotation is how LangGraph knows the possible targets (needed for compilation/visualization). No `add_conditional_edges` required for these transitions.
- `Command(goto=..., graph=Command.PARENT)` routes to a node in the **parent** graph — the primitive behind agent handoffs from inside subgraphs (see §8).
- `Command(resume=...)` is a different use of the same class: resuming interrupts (§6).
- You can also `goto=Send(...)` for dynamic fan-out from a node.

**When to use:** handoffs, and nodes whose one LLM call yields both content and routing. **Otherwise prefer plain edges** — they keep routing visible in the graph topology instead of buried in node bodies.

---

## 4. Send API: map-reduce fan-out

Normal edges send the *whole shared state* to a *statically known* set of next nodes. `Send` lets a conditional edge dispatch **N dynamic copies** of a node, each with its **own private state** — map-reduce.

```python
from langgraph.types import Send

class OverallState(TypedDict):
    tables: list[str]
    profiles: Annotated[list[dict], operator.add]   # reduce: accumulate

def fan_out(state: OverallState):
    return [Send("profile_table", {"table": t}) for t in state["tables"]]

def profile_table(payload: dict) -> dict:          # receives the Send payload, not OverallState
    return {"profiles": [profile(payload["table"])]}

builder.add_conditional_edges("plan", fan_out)      # no mapping dict — returns Sends
builder.add_edge("profile_table", "synthesize")     # implicit join: synthesize runs after all complete
```

All the `Send`-spawned instances run in parallel in one super-step; their writes merge through reducers. This is how you'd parallelize per-table schema profiling, multi-candidate SQL generation (generate 3 candidates, pick best), or per-chart data fetches.

**Pitfalls:**
- The reduce keys **must** have accumulating reducers or parallel writes collide.
- Payload state for the mapped node is separate from graph state — don't expect the mapped node to see the full state unless you pass what it needs.
- Fan-out multiplies token cost and hits rate limits fast; bound N.

---

## 5. Subgraphs

A compiled graph can be added as a node of another graph. Two integration modes:

1. **Shared state keys** — add the compiled subgraph directly as a node; overlapping keys flow in/out automatically:

```python
subgraph = sub_builder.compile()
builder.add_node("sql_pipeline", subgraph)
```

2. **Different schemas** — wrap it in a function node that translates state in/out:

```python
def call_sql_pipeline(state: ParentState) -> dict:
    out = subgraph.invoke({"question": state["user_question"]})
    return {"sql": out["final_sql"]}
```

Checkpointing propagates: the parent's checkpointer checkpoints subgraph state too (as nested namespaces), so interrupts inside subgraphs resume correctly. Stream from subgraphs with `stream(..., subgraphs=True)`.

**When useful:** team boundaries (one team owns the SQL-generation subgraph, another the visualization subgraph), reuse across products, and hierarchical multi-agent systems. **Pitfall:** deep nesting makes debugging and state-namespace reasoning hard — two levels is usually plenty; several 1.2.x patch releases were specifically subgraph-checkpoint bugfixes, so keep subgraph structures simple and pin versions.

---

## 6. Human-in-the-loop: `interrupt()` and `Command(resume=...)`

The pattern that justifies LangGraph for approval-gated workflows. `interrupt(payload)` pauses the graph *indefinitely* — state is checkpointed, the process can die, the pod can restart — and a later invocation with `Command(resume=value)` on the **same thread** continues, with `value` becoming the return of `interrupt()`.

```python
from langgraph.types import interrupt, Command

def approve_query(state: ChatState) -> dict:
    decision = interrupt({                      # graph pauses here
        "question": "Run this SQL against the warehouse?",
        "sql": state["sql"],
        "estimated_cost": state.get("cost_estimate"),
    })
    if not decision["approved"]:
        return {"error": "rejected by user"}
    return {"approved": True}

graph = builder.compile(checkpointer=checkpointer)   # REQUIRED for interrupts
config = {"configurable": {"thread_id": "conv-42"}}

result = graph.invoke({"user_question": "..."}, config)
# result contains "__interrupt__" with the payload → surface to the UI

graph.invoke(Command(resume={"approved": True}), config)   # continues from the pause
```

Approval-flow variants: approve/reject (route on the resume value), **edit state** (human corrects the generated SQL: resume with the edited text, or use `graph.update_state()` then resume), **tool-call review** (interrupt inside/around ToolNode before executing), and **input collection** (clarifying questions mid-flow). There are also static breakpoints — `compile(interrupt_before=["execute_query"])` — but dynamic `interrupt()` is the recommended, more expressive API.

**Pitfalls (the big one):**
- **On resume, the interrupted node re-runs from its top**, not from the interrupt line. Any code before `interrupt()` executes again. Put side effects *after* the interrupt, or in `@task`-wrapped functions (task results are checkpointed and replayed, not re-executed).
- Multiple `interrupt()` calls in one node must stay in a **consistent order** across resumes (resume values are matched up by order/ID) — don't call `interrupt()` conditionally on something that changes between runs.
- Payloads must be JSON-serializable.
- No checkpointer, no interrupts. And resuming requires the exact `thread_id`.
- If parallel nodes interrupt simultaneously, resume with a map of interrupt ID → value.

For chat-with-data: this is your "review generated SQL before it touches the warehouse" gate, and your detect-then-clarify loop — both are literally the canonical use cases.

---

## 7. Persistence: checkpointers, threads, time travel

**The checkpointer is the heart of production LangGraph.** At every super-step, the runtime serializes state to a checkpoint under a `thread_id`. That one mechanism buys you:

1. **Short-term memory** — invoke the same thread again and the conversation state is already there (no manual history management).
2. **Fault tolerance / resume** — process dies mid-run → re-invoke with `invoke(None, config)` and it continues from the last checkpoint.
3. **Human-in-the-loop** — interrupts are just checkpoints you resume later.
4. **Time travel** — fetch any historical checkpoint, fork it, replay from it.

```python
# dev
from langgraph.checkpoint.memory import InMemorySaver     # aka MemorySaver; RAM only
graph = builder.compile(checkpointer=InMemorySaver())

# production: pip install langgraph-checkpoint-postgres
from langgraph.checkpoint.postgres import PostgresSaver   # + AsyncPostgresSaver
with PostgresSaver.from_conn_string("postgresql://...") as cp:
    cp.setup()                                            # creates tables — run once
    graph = builder.compile(checkpointer=cp)
    config = {"configurable": {"thread_id": "user123-conv7"}}
    graph.invoke({"messages": [{"role": "user", "content": "Hi, I'm Bob"}]}, config)
    graph.invoke({"messages": [{"role": "user", "content": "What's my name?"}]}, config)  # remembers
```

(`langgraph-checkpoint-sqlite` → `SqliteSaver` for local/single-node; Redis and MongoDB savers exist as community/partner packages.)

**Threads** are just IDs — map them to your conversation/session model (`{org_id}-{conversation_id}` works well; keep under 255 chars for Postgres).

**Time travel:**

```python
state = graph.get_state(config)                      # current StateSnapshot
history = list(graph.get_state_history(config))      # all checkpoints, newest first

# replay/fork from a past checkpoint:
past = history[3].config                             # contains checkpoint_id
graph.invoke(None, past)                             # re-run from there (forks the thread)

# manual state surgery (e.g., human edits the SQL):
graph.update_state(config, {"sql": "SELECT ... FIXED ..."})
```

**Pitfalls:**
- `InMemorySaver` in production is the classic footgun — everything vanishes on restart, and it grows unboundedly.
- Checkpoints are written every super-step; large state (e.g. full query result sets) makes checkpointing slow and your Postgres fat. **Keep bulky data out of state** — store result sets elsewhere and keep references/summaries in state. (1.2 added a beta `DeltaChannel` that stores per-step deltas instead of re-serializing full values, aimed at exactly this.)
- State must be serializable by the built-in serializer (JSON-ish + LangChain objects). Custom objects need custom serde.
- Checkpoint tables need lifecycle management (TTL/cleanup jobs) — LangGraph OSS doesn't garbage-collect old threads for you (the Platform does).

---

## 8. Memory: short-term (thread) vs long-term (Store)

Two distinct axes, two distinct APIs:

| | Short-term | Long-term |
|---|---|---|
| Scope | one thread / conversation | cross-thread, cross-session |
| Mechanism | checkpointer + state (`messages`) | **Store API** (`BaseStore`) |
| Shape | full graph state snapshots | namespaced key-value docs, optionally semantically indexed |
| Use | conversation history, in-flight pipeline state | user preferences, learned facts, cached schema descriptions, feedback |

```python
from langgraph.store.memory import InMemoryStore
# production: PostgresStore from langgraph-checkpoint-postgres

store = InMemoryStore(index={"embed": embed_fn, "dims": 1536})  # optional semantic index
graph = builder.compile(checkpointer=cp, store=store)

from langgraph.config import get_store

def personalize(state: ChatState, config) -> dict:
    store = get_store()
    user_id = config["configurable"]["user_id"]
    ns = ("preferences", user_id)
    prefs = store.search(ns, query=state["user_question"], limit=3)  # semantic
    store.put(ns, "last_metric", {"metric": state.get("intent")})    # write
    return {"user_prefs": [p.value for p in prefs]}
```

Namespaces are tuples (like folder paths) — `("org_123", "user_456", "memories")`. `put/get/search/delete` are the whole interface.

For chat-with-data, the Store is a natural home for: per-user chart/metric preferences, corrections the user made to generated SQL ("we call it `gmv`, not `revenue`"), and org-level semantic-layer annotations — retrieved semantically per question and injected into the prompt.

**Pitfall:** the Store is a KV store with optional vector search, not a database — don't use it as your system of record, and design namespace hygiene (org/user isolation) up front for multi-tenant safety.

---

## 9. Durable execution, retries, caching

### Durability modes

Since 0.6 (formalized in 1.0), `invoke`/`stream` take a `durability` argument controlling *when* checkpoints are persisted:

- `"exit"` — checkpoint only when the run ends (fastest; no mid-run resume).
- `"async"` — persist in the background while the next step runs (**default** when a checkpointer is present; small window where a crash loses the latest step).
- `"sync"` — persist before starting the next step (strongest guarantee; highest latency).

```python
graph.invoke(inputs, config, durability="sync")   # e.g. before executing warehouse writes
```

Rule of thumb: `"async"` for chat; `"sync"` for anything with expensive/irreversible side effects.

### Determinism and `@task`

Durable execution replays from checkpoints — the code path up to the resume point re-executes. Non-deterministic or side-effecting work should be wrapped in **tasks**, whose results are saved in the checkpoint and *replayed, not recomputed*:

```python
from langgraph.func import task

@task
def run_warehouse_query(sql: str) -> list[dict]:
    return warehouse.execute(sql)          # runs once; replays from checkpoint on resume

def execute_node(state: ChatState) -> dict:
    fut = run_warehouse_query(state["sql"])
    return {"query_result": fut.result()}
```

### Retries and node lifecycle

```python
from langgraph.types import RetryPolicy

builder.add_node(
    "execute_query", execute_query,
    retry_policy=RetryPolicy(max_attempts=3, retry_on=(TransientDBError,)),
)
```

Exponential-backoff parameters (`initial_interval`, `backoff_factor`, `max_interval`) are configurable; by default a sane set of transient errors is retried. **1.2 additions:** per-node **timeouts** (wall-clock or idle — raises `NodeTimeoutError`, clears the attempt's writes, hands off to the retry policy) and node-level **error handlers** that run a recovery function after retries are exhausted. Use these instead of hand-rolled `try/except`-plus-sleep in nodes.

### Node caching

```python
from langgraph.types import CachePolicy
from langgraph.cache.memory import InMemoryCache

builder.add_node("profile_schema", profile_schema, cache_policy=CachePolicy(ttl=300))
graph = builder.compile(cache=InMemoryCache(), checkpointer=cp)
```

Cache key defaults to a hash of the node's input; great for expensive deterministic steps (schema introspection, embedding lookups). **Pitfall:** don't cache nodes whose output depends on things outside their input (time, external data freshness) without a TTL.

---

## 10. Streaming

Chat UX lives or dies on this. `stream()` / `astream()` take `stream_mode`:

| Mode | Emits | Use for |
|---|---|---|
| `"values"` | full state after each super-step | debugging, simple UIs |
| `"updates"` | per-node deltas (`{node_name: update}`) | progress indicators ("validating SQL…") |
| `"messages"` | LLM tokens + metadata, as generated **inside nodes** | token-by-token chat streaming |
| `"custom"` | whatever you emit via `get_stream_writer()` | row counts, query progress, chart specs |
| `"checkpoints"` / `"tasks"` / `"debug"` | checkpoint events / task start-finish / both with metadata | tracing, replay tooling |

```python
# token streaming from whichever node is calling the LLM
for msg_chunk, metadata in graph.stream(inputs, config, stream_mode="messages"):
    if metadata["langgraph_node"] == "summarize":       # filter by node
        print(msg_chunk.content, end="", flush=True)

# custom events from inside a node
from langgraph.config import get_stream_writer

def execute_query(state):
    writer = get_stream_writer()
    writer({"stage": "executing", "detail": "running against warehouse"})
    rows = run(state["sql"])
    writer({"stage": "done", "rows": len(rows)})
    return {"query_result": rows}

# multiple modes → (mode, chunk) tuples
for mode, chunk in graph.stream(inputs, config, stream_mode=["updates", "custom"]):
    ...
```

`stream_mode="messages"` works even when the LLM call is buried in a node or subgraph — the runtime taps callbacks; no per-node plumbing. Version note: 1.2 introduced a revised, versioned streaming surface (an enveloped v2 event shape and a content-block-centric "v3" message stream); the classic `stream_mode` API above remains supported — check the streaming docs for the current recommended event shape before building your WS protocol.

**Pitfalls:** in a sync `stream_mode="custom"` context, `get_stream_writer()` relies on contextvars — spawning raw threads inside nodes breaks it (Python < 3.11 async has similar caveats). And remember streamed `updates` include *all* state keys written, including "private" ones — filter server-side before forwarding to browsers.

---

## 11. The Functional API: `@entrypoint` / `@task`

An alternative to declaring graphs: write ordinary Python control flow and get the same runtime features (checkpointing, HITL, streaming, durability).

```python
from langgraph.func import entrypoint, task
from langgraph.types import interrupt

@task
def generate_sql(question: str) -> str:
    return llm.invoke(...).content

@task
def execute(sql: str) -> list[dict]:
    return warehouse.run(sql)

@entrypoint(checkpointer=checkpointer)
def chat_with_data(question: str) -> dict:
    sql = generate_sql(question).result()
    approved = interrupt({"sql": sql})           # HITL works here too
    if not approved:
        return {"status": "rejected"}
    rows = execute(sql).result()
    return {"status": "ok", "rows": rows}

chat_with_data.invoke("MAU by region?", {"configurable": {"thread_id": "t1"}})
```

Differences from the Graph API: standard `if`/`for` instead of edges; no declared state/reducers (state is function-scoped; `previous` + `entrypoint.final(value=..., save=...)` handle cross-invocation memory); checkpoints save **task results** rather than per-super-step state; no graph visualization; on resume the whole entrypoint replays with completed task results loaded from the checkpoint.

**When to choose which:** the Functional API shines for mostly-linear flows with occasional interrupts, or migrating existing imperative code. The Graph API wins when you want explicit topology (drawable, statically analyzable, easier for a team to reason about), parallel fan-out, and multi-agent structures. Determinism discipline matters more in the functional style: *all* side effects and randomness must live inside `@task`s or replay breaks.

---

## 12. Multi-agent patterns

You likely don't need multi-agent for chat-with-data v1 — but the shapes are worth knowing:

- **Supervisor (agents-as-tools / subagents):** a central agent (or router node) calls specialized agents; all routing returns to the center. Best for parallelizable, clearly-partitioned domains. The `langgraph-supervisor` package prebuilds this; hand-rolled it's just a node with conditional edges (or `Command(goto=...)`) to agent subgraphs.
- **Swarm / handoffs:** agents transfer control directly to each other via handoff tools that emit `Command(goto="other_agent", graph=Command.PARENT)`; the "active agent" is remembered in state. `langgraph-swarm` packages this. Best for multi-turn conversations where different specialists own different phases.
- **Hierarchical:** supervisors of supervisors, each level a subgraph. LinkedIn's Hiring Assistant is the flagship production example.
- **Custom workflow (usually right for you):** deterministic pipeline nodes, with a *single* agentic node where discretion is needed.

```python
# minimal supervisor sketch
def supervisor(state: MessagesState) -> Command[Literal["sql_agent", "viz_agent", END]]:
    decision = llm.invoke([SUPERVISOR_PROMPT, *state["messages"]])
    return Command(goto=decision.next_agent)

builder.add_node("supervisor", supervisor)
builder.add_node("sql_agent", sql_agent_graph)     # compiled subgraphs
builder.add_node("viz_agent", viz_agent_graph)
builder.add_edge("sql_agent", "supervisor")        # report back
builder.add_edge("viz_agent", "supervisor")
```

**Pitfalls:** multi-agent multiplies token cost and latency (each hop re-reads context); shared-`messages` state across agents grows fast — give agents private message channels and summarize at boundaries. LangChain's own docs stress "not every complex task requires multi-agent." Start single-graph.

---

## 13. Prebuilt components in 1.x

- **`create_react_agent` is deprecated** (still works, emits warnings). Its successor is **`langchain.agents.create_agent`** — same tool-calling loop, built on LangGraph, extensible via **middleware** (hooks before/after model and tool calls: summarization, HITL, PII redaction, guardrails). If you want an off-the-shelf tool-calling agent as a *node* in your graph, use `create_agent` and add the compiled agent as a subgraph node.

```python
from langchain.agents import create_agent

agent = create_agent(model="anthropic:claude-sonnet-4-5", tools=[run_sql, get_schema],
                     system_prompt="You are a careful analytics assistant...")
builder.add_node("analyst", agent)
```

- **`ToolNode`** — executes the tool calls found in the last AI message, in parallel, returning `ToolMessage`s; handles errors configurably. Pair with `tools_condition` for the classic agent loop. Still the standard building block when hand-rolling agent loops in graph form (importable from the LangChain 1.x agents package; the `langgraph.prebuilt` home is deprecated).
- `MessagesState`, `add_messages`, `InjectedState`/`InjectedStore` (inject graph state or the Store into tool signatures without exposing them to the LLM) remain current.

**Pitfall:** don't build your whole product *inside* `create_agent`'s loop if your flow is really a pipeline — wrap deterministic stages as graph nodes and reserve the agent loop for the genuinely open-ended part (or skip it entirely).

---

## 14. LangGraph Platform → LangSmith Deployment, and Studio (brief)

The commercial layer (renamed October 2025: **LangGraph Platform → LangSmith Deployment**, **LangGraph Studio → LangSmith Studio**):

- **LangGraph Server:** wraps your compiled graph in an opinionated API — threads/runs endpoints, streaming transport, background runs, cron, webhooks, built-in Postgres persistence + Redis task queue, double-texting handling (interrupt/rollback/enqueue when a user sends a second message mid-run).
- **Deployment options:** Cloud SaaS, BYOC (your VPC), Self-Hosted (Lite is free up to 1M node executions; Enterprise for full self-hosting). `langgraph dev` runs a local server for development.
- **Studio:** visual debugger — see the graph, step through runs, edit state at a checkpoint and re-run, test interrupts. Genuinely useful in development even if you never buy the platform.

The OSS library is MIT and fully usable without any of this — you self-host by running your graph inside your own FastAPI/worker processes with a Postgres checkpointer, which is a perfectly common production setup. What the platform buys you is the boring-but-real infra (queues, retries of *runs*, thread APIs, checkpoint lifecycle) you'd otherwise build. Decide based on how much of §7–§10 you want to own.

---

## 15. Honest assessment: when LangGraph is overkill, criticisms, alternatives

### Skip it (or defer it) when
- Your flow is a **straight line**: prompt → LLM → parse → respond. A function is better. Widely-cited war story: a twelve-node LangGraph pipeline replaced by ~180 lines of FastAPI + direct tool calling — 3× faster and debuggable by anyone.
- You have **no need for persistence, interrupts, or streaming-of-intermediate-steps** — those are the features that pay the complexity tax; without them you're buying ceremony.
- Strict-latency, short-lived, stateless calls (the super-step + checkpoint machinery adds overhead you don't need).

### Common criticisms (fair ones)
- **Learning curve / conceptual overhead:** reducers, super-steps, channels, checkpoints — non-trivial for what sometimes amounts to "call the LLM twice."
- **LangChain-ecosystem gravity:** docs and prebuilts nudge you into LangChain messages/models; debugging through layered abstractions when something misbehaves can be painful (less so in LangGraph core than old LangChain, but the criticism carries over by association).
- **Checkpointing costs:** naive designs serialize big state every step; you must actively keep state lean.
- **API churn at the edges:** the core froze at 1.0, but the periphery moves (prebuilt deprecations, streaming API versions, platform renames). Pin versions; re-read release notes on upgrade.

### Alternatives, one line each
- **Plain Python + provider SDK:** best power-to-weight for simple flows; you'll hand-roll persistence/HITL/streaming when you need them — which is exactly when LangGraph starts paying.
- **Temporal (or DBOS/Prefect):** *the* answer for heavy-duty durable execution and long-running jobs; general-purpose (not LLM-aware — no token streaming, message reducers, or HITL primitives), heavier ops. Some teams run LangGraph-style logic inside Temporal workflows.
- **PydanticAI:** excellent typed, single-agent request/response DX; integrates with Temporal for durability. Rule of thumb from the comparison literature: if you can describe your agent in two sentences, PydanticAI; if you need a flowchart, LangGraph.
- **OpenAI Agents SDK:** lightweight agents + handoffs + guardrails, great inside the OpenAI ecosystem; thinner on graph-shaped control flow, checkpoint-grade persistence, and provider neutrality.

For a chat-with-data pipeline with HITL SQL approval, resumable threads, bounded repair loops, and rich streaming — LangGraph's feature set maps almost one-to-one onto the requirements, which is why it's a defensible default *here* even though it's overkill for simpler products.

---

## 16. Who runs LangGraph in production, and for what

- **Uber (Developer Platform):** network of specialized agents for large-scale code migrations and unit-test generation; thousands of daily code fixes, ~21,000 developer-hours saved. Also the lineage of QueryGPT-style internal tooling patterns.
- **LinkedIn:** Hiring Assistant — hierarchical agent system on LangGraph for candidate sourcing/matching/messaging.
- **Klarna:** customer-support AI assistant for 85M users; ~80% reduction in resolution time (LangGraph + LangSmith).
- **Elastic:** orchestrating security AI agents for real-time threat detection in the Elastic AI Assistant.
- **Replit:** Agent that builds software from scratch — multi-agent with heavy human-in-the-loop visibility (every file edit / package install surfaced), an explicit showcase of the interrupt/streaming machinery.
- Others frequently cited by LangChain: AppFolio, GitLab (Duo), Komodo Health, Cisco, Vodafone.

Pattern worth noticing: these are all **long-running, stateful, HITL-heavy** systems — none of them are "one prompt in, one answer out."

---

## 17. Sources

**Official docs & releases**
- LangGraph docs hub: https://docs.langchain.com/oss/python/langgraph/overview
- Graph API (state, reducers, edges, Send, Command, caching): https://docs.langchain.com/oss/python/langgraph/graph-api
- Persistence (checkpointers, threads, time travel, stores): https://docs.langchain.com/oss/python/langgraph/persistence
- Durable execution: https://docs.langchain.com/oss/python/langgraph/durable-execution
- Interrupts / HITL: https://docs.langchain.com/oss/python/langgraph/interrupts
- Streaming: https://docs.langchain.com/oss/python/langgraph/streaming
- Functional API: https://docs.langchain.com/oss/python/langgraph/functional-api
- Multi-agent patterns: https://docs.langchain.com/oss/python/langchain/multi-agent
- LangGraph v1 release notes: https://docs.langchain.com/oss/python/releases/langgraph-v1
- Changelog (1.2 features): https://docs.langchain.com/oss/python/releases/changelog · https://github.com/langchain-ai/langgraph/releases
- LangChain & LangGraph 1.0 announcement: https://blog.langchain.com/langchain-langgraph-1dot0/
- LangGraph 1.0 GA: https://changelog.langchain.com/announcements/langgraph-1-0-is-now-generally-available
- Platform → LangSmith Deployment renaming: https://changelog.langchain.com/announcements/product-naming-changes-langsmith-deployment-and-langsmith-studio
- LangSmith Deployment: https://www.langchain.com/langsmith/deployment

**Production usage & ecosystem**
- "Is LangGraph used in production?": https://blog.langchain.com/is-langgraph-used-in-production/
- Built with LangGraph (Uber, LinkedIn, Klarna, Elastic, Replit case studies): https://www.langchain.com/built-with-langgraph
- Uber LangGraph developer-tools case study (ZenML LLMOps DB): https://www.zenml.io/llmops-database/building-ai-developer-tools-using-langgraph-for-large-scale-software-development
- create_react_agent deprecation discussion: https://github.com/langchain-ai/langgraph/issues/6404

**Critical perspectives & comparisons**
- Framework-overkill analysis: https://dev.to/theprodsde/most-ai-agent-frameworks-are-overkill-heres-how-to-choose-the-right-one-in-30-seconds-37k3
- PydanticAI vs LangGraph: https://www.zenml.io/blog/pydantic-ai-vs-langgraph · https://www.speakeasy.com/blog/ai-agent-framework-comparison/
- Scaling LangGraph (parallelization/subgraph trade-offs): https://aipractitioner.substack.com/p/scaling-langgraph-agents-parallelization
