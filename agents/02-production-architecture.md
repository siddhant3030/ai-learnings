# Production Architecture for Runtime AI Agents in Dalgo

> Staff-engineering design for the **runtime** agent surface — the assistant NGO users
> actually hit inside the Dalgo webapp. Grounded in the real `DDP_backend` codebase, the
> Dalgo MCP server, and the team's research notes (`agent-design.md`,
> `context-engineering.md`, `mcp/05-building-by-category.md`).
>
> **Core principle (from the team):** dev-time agents can be deep and multi-layered;
> **runtime agents must be simple, deterministic, bounded, cheap-model, narrow-tool.**
> Complex agentic loops at runtime cause unpredictable cost, latency, and results. Every
> decision below is made to honor that.

---

## 0. What the codebase already tells us (grounding)

Before designing anything, here is what `DDP_backend` already has — this shapes the whole
recommendation:

| Capability found | Where | Implication for the runtime agent |
|---|---|---|
| **An external LLM microservice** | `ddpui/core/llm_service.py` — `LLM_SERVICE_API_URL`, `LLM_SERVICE_API_KEY`, HTTP `file/upload` + `file/query`, `poll_llm_service_task()` | Already exists for **async/batch** LLM work (log + long-text summarization, file-search). It **blocks-and-polls** via Celery. Keep it for batch; do **not** route interactive chat through it. |
| **Celery + Redis** | `requirements.txt` (`celery==5.2.7`, `redis==5.0.4`), `ddpui/celeryworkers/tasks.py` (`summarize_logs`, `summarize_warehouse_results`) | Async/long-running work already runs on Celery. The agent reuses this for any tool that triggers a pipeline/dbt run. |
| **Django Channels + Redis websockets** | `requirements.txt` (`channels==4.1.0`, `channels-redis==4.2.0`), `ddpui/settings.py` `CHANNEL_LAYERS`, `ddpui/websockets/airbyte_consumer.py` | **Streaming to the UI is already a solved problem here.** `airbyte_consumer.py` is the template for an agent token-stream consumer. |
| **LLM persistence models** | `ddpui/models/llm.py` — `LlmSession`, `AssistantPrompt`, `UserPrompt`, `LlmSessionStatus`, `LlmAssistantType` | Conversation/session state has a home already. Extend, don't reinvent. |
| **Org-scoped auth on every endpoint** | `ddpui/auth.py` `has_permission([...])`, `request.orguser.org` | Multi-tenancy + RBAC is enforced server-side at the API boundary. The agent rides on exactly this. |
| **A "core functions" layer** | `ddpui/core/*functions.py` (`warehousefunctions.py`, `pipelinefunctions.py`, `transformfunctions.py`, `orgfunctions.py`, …) and `ddpui/services/*` | This is the real business-logic layer. **Both** the existing Ninja APIs and the Dalgo MCP server should be thin wrappers over it. So is the agent. (See §3 — "one core, two surfaces.") |
| **A Dalgo MCP server exposing ~80 tools** | Separate process (location not confirmed from this repo — see assumption below) | The **external** agent surface (Claude Desktop, third-party clients). The runtime agent is a **different** surface and should not go through it. |

> **Stated assumption (could not confirm from this repo):** the Dalgo MCP server's tools
> wrap the same `ddpui/core/*functions.py` / `ddpui/services/*` business logic that the
> Ninja APIs use. The "one core, two surfaces" recommendation in §3 depends on this; if
> the MCP server currently reimplements logic, the first refactor is to collapse it onto
> the shared core.

The headline: **Dalgo is not greenfield for AI.** It already separates *batch* LLM work
(external microservice + Celery poll) from *interactive* work (which doesn't exist yet).
The runtime agent is the interactive surface, and it belongs in a different place than the
batch microservice for concrete, defensible reasons (§1).

---

## 1. Where the agent lives

### Recommendation: a new module **inside `DDP_backend`**, not a new microservice

Create `ddpui/core/agent/` (a package, optionally a thin Django app `ddpui/agent/` if it
needs its own models/migrations/routes). The runtime agent runs **in-process with Django**.

**Why in-Django and not the existing external llm-service:**

1. **It must call internal functions directly.** The agent's tools are thin wrappers over
   `ddpui/core/*functions.py`. An external service would have to re-expose all of that
   over HTTP — pure indirection for code Dalgo owns on both ends (§3, and the
   "plain functions when your own code is the consumer" lesson).
2. **It must reuse `has_permission` + `request.orguser.org`.** Tenancy and RBAC are
   enforced at the Django boundary. Running the loop in-process means the agent inherits
   the *already-authorized* request context — it cannot act outside the caller's org or
   permissions (§3 tool scoping). An external service would have to re-implement and
   re-trust auth.
3. **Streaming on slow connections.** The team requires perceived speed on slow internet.
   Dalgo already streams over Channels/Redis (`airbyte_consumer.py`). The agent loop can
   push Claude's streamed tokens straight onto that websocket. The existing llm-service is
   **HTTP request → Celery → poll** (`poll_llm_service_task` literally `time.sleep`s) —
   that pattern is the *opposite* of streaming and would fight the requirement.

**Why not a brand-new microservice:** a small team does not want a second deployable,
second auth surface, second on-call. The "simplest thing that works" principle (Anthropic:
*"find the simplest solution possible"*) says: a module in the monolith until scale forces
otherwise. The agent loop is I/O-bound (waiting on Claude + tools), so it does **not** need
process isolation for CPU reasons.

**Division of labor (keep both):**

| Surface | Lives | Pattern | Use for |
|---|---|---|---|
| **Batch LLM** (existing) | external `llm-service` | HTTP submit → Celery → **poll** | Log/text summarization, file-search over large docs. Not user-interactive. |
| **Runtime agent** (new) | `ddpui/core/agent/` in Django | thin tool-loop → **stream** over Channels | Interactive in-product assistant. Bounded, cheap, narrow. |

### Request flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ webapp_v2 (Next.js)                                                            │
│  Chat UI  ──HTTP POST /api/agent/chat (start)──┐   ◄─── WebSocket token stream │
└────────────────────────────────────────────────┼──────────────▲───────────────┘
                                                  │              │
                          ┌───────────────────────▼──────────────┴───────────────┐
                          │ Django (DDP_backend)                                  │
                          │                                                       │
                          │  ddpui/api/agent_api.py   @has_permission([...])      │
                          │     request.orguser.org  ── authorizes + scopes ──┐   │
                          │                                                   │   │
                          │  ddpui/core/agent/loop.py  (the thin agent loop)  │   │
                          │   ┌───────────────────────────────────────────┐   │   │
                          │   │ while not done and budget_ok:             │   │   │
                          │   │   1. build context (§5)                   │   │   │
                          │   │   2. call Claude API (stream)  ───────────┼───┼───┼──► Claude API
                          │   │   3. on tool_use → dispatch (§3)          │   │   │    (tool use,
                          │   │   4. enforce budget/timeout/cost (§4)     │   │   │     streaming)
                          │   │   5. stream tokens → Channels consumer    │   │   │
                          │   └───────────────────────────────────────────┘   │   │
                          │            │ tool calls (in-process Python)        │   │
                          │            ▼                                       │   │
                          │  ddpui/core/agent/tools.py  ──► ddpui/core/*functions.py
                          │                                  ddpui/services/*     │
                          │       (the SAME core the Ninja APIs + MCP wrap)    │   │
                          │            │                                       │   │
                          │            ├──► read tools  → warehouse / Prefect / dbt
                          │            └──► write tools → Celery task (async, §7)│  │
                          │                                                   │   │
                          │  ddpui/models/llm.py  LlmSession  ◄── persist (§6)─┘   │
                          │  Redis  ◄── per-org cost counter, circuit breaker (§4)│
                          └───────────────────────────────────────────────────────┘
```

Two channels by design: the **POST** kicks off the turn and returns a `session_id` + the
websocket group name; **tokens stream back over the websocket** (reusing the
`airbyte_consumer.py` pattern) so the user sees output within a second on slow links even
while tools are still running.

---

## 2. The model layer (Claude API, tool use)

> **Accuracy caveat:** model IDs and per-token prices below should be **verified against
> the `claude-api` skill / current Anthropic pricing page at implementation time**. This
> doc was written with a Jan-2026 knowledge cutoff; Anthropic refreshes the lineup and
> prices, and the runtime agent's whole economics depend on picking the *current* cheapest
> capable tier. Treat the tier *strategy* below as stable and the exact IDs as
> "confirm before coding."

### Tiering strategy

The runtime agent is a **bounded, narrow-tool, mostly-workflow** surface. Per the team
principle and the research (`agent-design.md`: *"most production 'agents' are actually
workflows… cheaper to debug"*; *"more capable model ≠ better agent"*), the default model
is the **cheapest fast tier in the Claude family** (the Haiku-class / small-fast tier),
not the flagship.

| Tier | Model class | Use at runtime for | Rationale |
|---|---|---|---|
| **Default (runtime)** | Cheapest fast Claude (Haiku-class) | All standard interactive turns: answer questions about pipelines/sources/dashboards, explain an error, look up a schema, summarize a run | Bounded tasks with a narrow fixed toolset don't need a flagship. Cheap + fast = predictable cost and low latency on slow connections. Tool-use accuracy is high enough when the toolset is ≤10 (`context-engineering.md`). |
| **Escalation (rare, gated)** | Mid Claude (Sonnet-class) | A *single* harder step the cheap model demonstrably failed (e.g. ambiguous multi-table reasoning), behind a **router** decision, not an open-ended "use the smart model" | Justified only when measured failure on the cheap tier warrants it. Routing pattern, not default. |
| **Never at runtime** | Flagship (Opus-class), extended thinking | — | Flagship + extended thinking belongs to **dev-time** agents (planning, code gen). At runtime it blows the cost cap and adds latency. Extended-thinking budgets are also non-linear: past a point, more thinking *hurts* (`agent-design.md`). |

**Routing, not autonomy:** model choice is a **code decision** (a cheap classifier or a
rule on the request surface), never something the model escalates itself into. Keep it a
Routing workflow (Anthropic's taxonomy), the second-simplest pattern.

### Streaming for perceived speed

- Use the Anthropic SDK's **streaming** API (`stream=True`) for every turn.
- Pipe `content_block_delta` text events straight onto the Channels websocket group as
  they arrive. The user sees the first tokens in ~1s even though the full turn (with tool
  calls) may take longer — critical on the slow internet / old-device profile.
- **Prompt caching** the static prefix (system prompt + tool schemas + org-static context)
  cuts both cost and time-to-first-token. Structure context so the cacheable prefix is
  stable (§5) — `context-engineering.md` KV-cache rules.

---

## 3. The tool layer — MCP indirection vs direct functions

### The decision: the runtime agent calls **internal Django functions directly. No MCP.**

This is the crux. Dalgo **owns both sides** — the agent and the platform. The research is
unambiguous here (`mcp/05-building-by-category.md`, §8 "When NOT to build an MCP"):

> **"Prefer a plain function / direct API call when your own code is the consumer. MCP's
> value is the protocol — a standard surface for external agents. If you're orchestrating
> your own LLM calls, just call the function; MCP adds a network hop, a server to run, and
> serialization overhead for no benefit."**

And the MCP Tax (`context-engineering.md`): loading ~80 Dalgo MCP tool definitions into
every turn would consume tens of thousands of tokens of context **before the user speaks**,
and tool-selection accuracy collapses past ~10–50 tools (Opus 4 measured 49% with large
toolsets). The runtime agent must be cheap and bounded — it cannot afford the MCP tax.

### "One core, two surfaces"

The right mental model: there is **one** business-logic core and **two** agent-facing
surfaces over it.

```
                       ┌──────────────────────────────────────┐
                       │  ddpui/core/*functions.py             │   ◄── ONE implementation
                       │  ddpui/services/*                     │       (warehouse, pipelines,
                       │  (warehouse, Prefect, Airbyte, dbt…)  │        dbt, dashboards…)
                       └───────────────┬──────────────────────┘
            ┌──────────────────────────┼──────────────────────────┐
            ▼                          ▼                          ▼
   ┌────────────────┐        ┌──────────────────┐       ┌──────────────────────┐
   │ Ninja REST API │        │ Dalgo MCP server │       │ Runtime agent (NEW)  │
   │ (webapp_v2)    │        │ EXTERNAL surface │       │ INTERNAL surface     │
   │ humans/UI      │        │ Claude Desktop,  │       │ in-product assistant │
   │                │        │ 3rd-party agents │       │ direct fn calls      │
   └────────────────┘        └──────────────────┘       └──────────────────────┘
```

- **Keep the Dalgo MCP server** — it earns its keep as the **external** surface (the case
  MCP was designed for: *"the same integration surface is needed by many… agent clients"*).
- **Build the runtime agent as direct Python calls** into the same core. Its tool functions
  are ~10 thin Python wrappers in `ddpui/core/agent/tools.py`, each calling a
  `*functions.py` function and returning **high-signal, token-trimmed** results (CSV/TSV
  for tabular, not raw JSON dumps — Datadog/Block lesson).

This avoids duplicating logic *and* avoids paying the MCP network/context cost for an
in-process consumer. If the MCP server today reimplements logic instead of wrapping the
core, refactoring it onto the shared core is the prerequisite (the stated assumption in §0).

### Tool scoping per org — never trust the model

Multi-tenancy is enforced **in the tool dispatch layer, server-side, from the request
context — never from a model-supplied argument.**

```python
# ddpui/core/agent/tools.py  (sketch)
def tool_list_pipelines(ctx: AgentContext, **model_args):
    # org comes from the AUTHORIZED request, NOT from model_args
    org = ctx.orguser.org                      # set by has_permission at the boundary
    if not ctx.orguser.has_perm("can_view_pipelines"):
        return tool_error("not permitted")     # deterministic refusal
    return trim(pipelinefunctions.list_for_org(org))   # org-scoped query
```

Rules:
- The model **never** supplies `org_id`. It is injected from `request.orguser.org`.
- Each tool re-checks the relevant `has_permission` slug — the same RBAC the REST API uses.
- **Write tools are a distinct, gated set** (see §4/§7), separated from read tools, with
  confirmation. Treat every warehouse value the agent reads as a potential indirect
  prompt-injection vector (`agent-design.md`: OWASP #1 LLM risk) — it can change what the
  model *says*, but server-side scoping bounds what it can *do*.

---

## 4. Determinism & bounding

This is where the team principle becomes code. The runtime loop is **not** an open-ended
ReAct agent — it is a **bounded loop with hard stops enforced in Python**, not in the
prompt. (`agent-design.md` anti-pattern: *"prompt-as-architecture"* — control flow belongs
in code, not the system prompt.)

| Control | Default | Enforcement point (in code, not prompt) |
|---|---|---|
| **Tool-call budget / turn** | ≤ 5 tool calls | A counter in `loop.py`; on exceed → stop, return best answer + "couldn't fully resolve" |
| **Loop iterations** | ≤ 6 | Hard `for`-bound on the agentic loop. No `while True`. |
| **Max output tokens** | small cap per turn | `max_tokens` on the Anthropic call |
| **Wall-clock timeout** | ~30–45 s/turn | `asyncio.wait_for` around the loop; on timeout → graceful message |
| **Fixed tool set per surface** | ~10 tools, frozen | Tools are registered per agent surface at boot, not chosen dynamically. Sub-30 tools → 3x better selection (`agent-design.md`). |
| **No open-ended loops** | — | The loop terminates on: final text, budget hit, timeout, or tool error threshold. |
| **Circuit breaker** | N consecutive tool failures → abort | Counter in `loop.py`; the **$47k runaway-loop lesson** (`agent-design.md`: *"naive agents retry indefinitely"*). After N failures → structured failure, escalate to human, **never** silent retry. |
| **Per-request cost cap** | hard ₹/$ ceiling | Sum streamed `usage` tokens × price; abort mid-turn if exceeded. |
| **Per-org cost cap** | daily/monthly ceiling | **Redis counter keyed by org** (Redis already deployed). On exceed → degrade to a non-LLM fallback (§8) and notify. Protects the ~₹2L/yr/NGO budget. |

The $47k story is the load-bearing one: a runaway loop with no circuit breaker, no
tool-budget, and no cost cap is exactly the failure mode the team principle exists to
prevent. All three controls live in the loop code so a bad prompt or a flaky tool **cannot**
produce unbounded spend.

---

## 5. Context engineering for the agent

Goal: **minimum viable context** — cheap tokens (budget + slow connections) and high
accuracy (avoid context rot). From `context-engineering.md`: *"context is a finite,
depreciating resource… most agent failures are context failures."*

### What to inject, in cache-friendly order

```
┌─ STATIC PREFIX (prompt-cached, stable across turns) ───────────────────────┐
│ 1. System prompt — Dalgo domain vocab as COMPACT definitions, not essays.   │
│    Altitude: enough to guide a non-technical-NGO-facing assistant; few-shot  │
│    examples > long rule lists (`context-engineering.md`).                    │
│ 2. Fixed tool schemas (~10, trimmed descriptions + input_examples).          │
│ 3. Org-static context: org name, user role/permissions, plan/feature flags.  │
├─ DYNAMIC SUFFIX (changes per turn, placed last for recency) ────────────────┤
│ 4. Compacted conversation summary (older turns).                             │
│ 5. Last N raw turns verbatim.                                                │
│ 6. Just-in-time retrieved data (only what THIS turn needs).                  │
│ 7. Current user message.                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Keep it token-cheap

- **Just-in-time, never pre-load schema.** Do **not** stuff the org's full warehouse schema
  into context. Give the agent `list_schemas → list_tables → get_table_columns` tools and
  let it fetch only the tables this question touches (Claude-Code model, called out
  explicitly for Dalgo in `context-engineering.md`).
- **Product docs via retrieval, not bulk.** Dalgo docs are searchable (the MCP exposes
  `search_docs`); the in-Django agent calls the same docs search **function** and injects
  only top 2–3 reranked chunks — not the whole doc set.
- **Trim tool results.** CSV/TSV for tables, cap rows + bytes, "N more rows omitted" marker.
- **Compact, don't accumulate.** When history grows, summarize older turns but **preserve
  error traces** (high-signal — `context-engineering.md` Factor 9). For a bounded runtime
  agent, sessions are short, so simple last-N-turns + a one-line running summary is enough;
  no vector memory needed for working context.
- **No chain-of-thought bloat** at runtime — it adds tokens that push instructions into the
  low-attention middle and worsens context rot.

Context assembly is **deterministic business logic** — unit-test it (budget respected,
ordering correct, org-static fields present, trimming drops lowest-relevance first).

---

## 6. State & memory

Keep runtime memory **minimal and deterministic**. Two stores, both already in the stack.

| Need | Store | What | Isolation |
|---|---|---|---|
| **Conversation/session state** (durable) | **Postgres via `LlmSession`** (`ddpui/models/llm.py`) | Turn history, tool calls made, final responses, token/cost usage, feedback. Add a `session_type = "runtime_agent"` (extend the existing `LlmAssistantType` enum). | `org` + `orguser` FKs already on the model → per-org isolation for free. |
| **Hot working state** (ephemeral) | **Redis** (already deployed) | In-flight turn buffer, per-org cost counters, circuit-breaker counters, websocket group routing. TTL'd. | Key namespaced by org id. |

- **Persist:** messages, tool-call log, usage/cost, latency, feedback (`LlmSession` already
  has a `feedback` field). This doubles as the **audit trail** and the **eval dataset**.
- **Do NOT build a vector long-term memory for runtime.** `agent-design.md` /
  `context-engineering.md`: memory systems are "unsolved," and *"vector stores are NOT
  appropriate for critical facts, permissions, or financial data — use structured storage
  with deterministic retrieval."* NGO data accuracy is non-negotiable. Any cross-session
  "memory" (e.g. preferred chart type) should be **structured user preferences in Postgres**
  (Dalgo already has `UserPreferences`), retrieved deterministically — not embeddings.
- **Per-org isolation** is enforced the same way as the rest of Dalgo: every query filters
  on `request.orguser.org`. The model never names an org.

---

## 7. Async & long-running work

Some agent intents trigger genuinely long operations — **run dbt**, **trigger a pipeline**,
**sync a source**. These take minutes; the request must not block, and the agent loop's
30–45s budget must not be held hostage.

### Pattern: write tools are **fire-and-confirm**, executed on Celery, watched via the stream

```
User: "run my daily pipeline"
  │
  ▼ agent (cheap model) selects write tool  trigger_pipeline(pipeline_id)
  │
  ▼ tool layer:  has_permission("can_run_pipeline")? ──► CONFIRM gate (§4, irreversible)
  │              on confirm → enqueue existing Celery task (reuse pipeline trigger fn)
  │              return immediately: { "status": "started", "flow_run_id": ... }
  │
  ▼ agent streams: "Started your pipeline. I'll update you as it runs."  (turn ENDS — non-blocking)
  │
  ▼ Prefect runs the flow (orchestration already exists)
  ▼ status flows back via the SAME mechanism Dalgo already uses:
       - Prefect/Celery updates LlmSession.flow_run_id (field already on the model!)
       - a Channels message pushes "pipeline succeeded/failed" to the user's websocket group
```

Key points:
- **Reuse what exists.** Long work already runs on **Prefect** (orchestration) and
  **Celery** (`summarize_logs`, `run_dbt_commands` in `tasks.py`). The agent's write tools
  enqueue the *same* tasks — it does not invent a new execution path. `LlmSession` already
  carries `flow_run_id`, `task_id`, `airbyte_job_id` for exactly this correlation.
- **Don't poll inside the loop.** The existing batch `poll_llm_service_task()` blocks with
  `time.sleep` — fine for a Celery worker, **forbidden** in an interactive turn. The agent
  turn ends after enqueue; completion arrives later as a **push** over Channels (or, if the
  client prefers, a status endpoint it can poll).
- **Webhooks/events, not blocking waits.** Prefect→Celery→Channels is the chain. The agent
  is stateless between the "started" turn and the "finished" notification.

This keeps the interactive loop bounded (§4) while long work runs on the infra built for it.

---

## 8. Deployment & ops

### Fits the existing deploy

- The agent is a **module in `DDP_backend`** → ships with the existing Django/Gunicorn +
  Celery + Channels(ASGI) deployment. **No new deployable, no new on-call.** This is the
  whole point for a small team.
- Config via the existing `.env` pattern (`ANTHROPIC_API_KEY`, model IDs, budget caps) —
  same as the existing `LLM_SERVICE_API_*` vars.
- The streaming path needs the ASGI/Channels worker that already serves `airbyte_consumer`.

### Scaling for a small team

- The loop is **I/O-bound** (waiting on Claude + tools), so it scales with async workers,
  not CPU. Channels + Redis already handle the websocket fan-out.
- Cost is the real scaling axis, and §4's **per-org Redis cost caps** make spend bounded
  and predictable per NGO — the thing the budget constraint demands.

### Fallback / graceful degradation (model down or uncertain)

| Failure | Degradation |
|---|---|
| **Claude API down / timing out** | Catch at the loop boundary → return a deterministic message ("assistant unavailable, here's the relevant page/link") and, where possible, route the user to the **existing non-AI UI** for the same task. The product must work without the agent. |
| **Per-org cost cap hit** | Disable the agent for that org until reset; show a clear notice; non-AI features unaffected. |
| **Low model confidence / ambiguous** | Confidence-based escalation (`agent-design.md`): ask one clarifying question, or hand to a human/support — never guess on an irreversible action. |
| **Tool failure threshold (circuit breaker)** | Structured failure message + log; do not retry blindly. |
| **Write action** | **Pre-execution approval gate is mandatory** for anything irreversible (run/trigger/delete) — non-negotiable for non-technical NGO users (`agent-design.md`: *"a 30-second human review prevents irreversible mistakes"*). |

### Observability + evals

- Trace every turn: tool calls (in/out), token + cost per step, latency, success/failure —
  all persisted on `LlmSession`.
- **Build the eval set from real `LlmSession` rows.** The research warns: 89% of teams have
  observability, only 52% have evals — and *"only 12% of agent pilots reach production"*
  mostly for lack of evals. For NGO users, the gold-standard metric is **human-judged
  accuracy + understandability**; assemble 20–50 real cases and run them before any prompt
  or tool change ships.

---

## 9. Build vs buy

### Recommendation: **roll a thin loop on the Anthropic SDK directly.** No agent framework.

| Option | Verdict | Why |
|---|---|---|
| **Anthropic SDK + thin Django loop** | ✅ **Build this** | ~150 lines: the bounded loop, tool dispatch into `*functions.py`, streaming to Channels. Maximum debuggability, zero black box, full control of the determinism/cost controls (§4) that are the entire point of the runtime principle. Matches Anthropic's own guidance: *"reduce abstraction layers as you move to production."* |
| **Pydantic AI** | ⚠️ **One justified concession** | Use Pydantic **only** for typed tool input/output validation (it already underpins Django Ninja's schemas here — zero new mental model). Optionally its thin agent wrapper. It validates every model response against a typed schema before use — valuable when tool args drive real platform actions. Do not adopt it as a heavy orchestration layer. |
| **LangGraph** | ❌ **Reject for runtime** | Graph state / time-travel / checkpointing solve a problem the runtime agent **deliberately does not have** — it's bounded, short, and must be dead simple. The abstraction fights debuggability and the determinism requirement, and adds a heavy dependency for a small team. (Reasonable for a *dev-time* agent later; not here.) |
| **CrewAI / AutoGen (multi-agent)** | ❌ **Reject** | Multi-agent at runtime is the opposite of the team principle. `agent-design.md` / Cognition: *"don't build multi-agents"* — fragmented context, 17.2x error amplification, ~15x token cost. The runtime agent is single-threaded by design. |

**Why thin-loop wins for Dalgo specifically:** the runtime surface is mostly a **Routing +
bounded-ReAct workflow**, not an autonomous agent. The hard parts (cost caps, circuit
breaker, org scoping, streaming, Celery hand-off, `LlmSession` persistence) are all
**Dalgo-specific integration points** that no framework gives you for free — and that a
framework's abstractions make *harder* to inspect and bound. The simplest thing that works
is a loop you can read top to bottom.

```python
# ddpui/core/agent/loop.py — the entire runtime agent, conceptually
def run_turn(ctx, user_message, stream):
    messages = build_context(ctx, user_message)           # §5
    for step in range(MAX_STEPS):                          # §4: hard bound, no while True
        check_org_cost_cap(ctx.org)                        # §4: Redis counter
        resp = anthropic.stream(model=CHEAP, max_tokens=CAP, tools=FIXED_TOOLS, messages=messages)
        for delta in resp: stream.push(delta)              # §2: stream to Channels
        if resp.stop_reason != "tool_use": return          # done
        for call in resp.tool_calls[:TOOL_BUDGET]:         # §4: budget
            result = dispatch(ctx, call)                   # §3: org-scoped, RBAC-checked
            messages.append(result)
        if circuit_breaker_tripped(ctx): return fail()     # §4: the $47k lesson
    return budget_exhausted()                              # §4: graceful stop
```

---

## Summary table — every decision against the runtime principle

| Dimension | Decision | Honors "simple, deterministic, bounded, cheap, narrow"? |
|---|---|---|
| Placement | Module in `DDP_backend`, not a microservice | Simple: no new deployable; reuses auth + streaming |
| Model | Cheapest fast Claude tier default; mid-tier behind a router; never flagship | Cheap + bounded latency |
| Tools | Direct calls into `*functions.py`; **no MCP** at runtime; ~10 frozen tools | Narrow, cheap (no MCP tax) |
| Determinism | Tool budget, loop bound, timeout, circuit breaker, cost caps — all in code | Bounded + deterministic; the $47k guard |
| Context | Minimum-viable, just-in-time, cache-friendly, trimmed | Cheap on tokens + slow links |
| State | `LlmSession` (Postgres) + Redis hot state; no vector memory | Deterministic, isolated per org |
| Async | Write tools enqueue existing Celery/Prefect; push via Channels; never block | Bounded turns; reuse existing infra |
| Ops | Ships with current deploy; per-org cost caps; non-AI fallback | Small-team friendly; safe degradation |
| Framework | Thin loop on Anthropic SDK; Pydantic for typed tool I/O only | Simplest thing that works; no black box |

---

## Sources

Internal: `ai-learnings/research/agent-design.md`, `ai-learnings/research/context-engineering.md`,
`ai-learnings/mcp/05-building-by-category.md`; `DDP_backend` (`ddpui/core/llm_service.py`,
`ddpui/models/llm.py`, `ddpui/api/pipeline_api.py`, `ddpui/api/warehouse_api.py`,
`ddpui/auth.py`, `ddpui/websockets/airbyte_consumer.py`, `ddpui/celeryworkers/tasks.py`,
`ddpui/core/*functions.py`, `ddpui/settings.py`).

External (agent-framework landscape, 2026):
- [Building Effective AI Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents)
- [Effective Context Engineering for AI Agents — Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Don't Build Multi-Agents — Cognition](https://cognition.ai/blog/dont-build-multi-agents)
- [AI Agent Frameworks (2026) + Claude Agent SDK Primitive Reference — Morph](https://www.morphllm.com/ai-agent-framework)
- [Pydantic AI](https://ai.pydantic.dev/) · [pydantic/pydantic-ai (GitHub)](https://github.com/pydantic/pydantic-ai)
- [Choosing an agent framework: LangChain vs LangGraph vs CrewAI vs PydanticAI — Speakeasy](https://www.speakeasy.com/blog/ai-agent-framework-comparison)
- [The 2026 AI Agent Framework Decision Guide — DEV](https://dev.to/linou518/the-2026-ai-agent-framework-decision-guide-langgraph-vs-crewai-vs-pydantic-ai-b2h)

> **Action before coding:** confirm current Claude model IDs + pricing via the `claude-api`
> skill, and confirm the Dalgo MCP server wraps `ddpui/core/*functions.py` (the §0
> assumption). Both are noted inline.
