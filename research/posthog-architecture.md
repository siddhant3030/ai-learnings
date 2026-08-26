# PostHog AI Architecture — Learnings for Dalgo

*Studied from a local clone of the PostHog repo (`~/Documents/posthog`), August 2026.
Focus: how PostHog's "Max" AI assistant stores conversations and memory, traced
through every AI-related Django migration they wrote — and what that history
predicts for Dalgo's Chat with Data.*

---

## Why this matters to Dalgo

Dalgo's Chat with Data and PostHog's Max share the same core design: a LangGraph
agent whose conversation state is saved by a **checkpointer** (LangGraph's
persistence plug-in — it snapshots the agent's state to Postgres after every step,
keyed by a `thread_id`). PostHog is ~18 months ahead of us on the same road.
Their migration history is a preview of the schema changes Dalgo will need, in
roughly the order we'll need them.

---

## The two storage philosophies

**The rule:** Both platforms store a lightweight "session" row in Django and keep
the actual messages in LangGraph checkpoints. They differ on *who owns the
checkpoint tables*.

**Example:** When Priya asks "how many farmers did we train last quarter?", Dalgo
writes one `ChatWithDataSession` row (owner, title, `thread_id`) and LangGraph's
stock `AsyncPostgresSaver` writes the messages into its own `checkpoints` tables —
tables Django doesn't know exist. In PostHog, the same question lands in a
`Conversation` row plus `ConversationCheckpoint` rows that are **regular Django
models** with foreign keys, written by their hand-rolled `DjangoCheckpointer`.

**Why it matters:** Dalgo's choice costs zero custom code but the tables live
outside Django migrations (we need a separate `manage.py chat_with_data_setup`
step per environment, and forgetting it produces the
`relation "checkpoints" does not exist` error). PostHog's choice costs ~400 lines
of serialization code they must maintain, but buys cascade deletes, ORM
queryability, normal migrations, and the ability to compact old checkpoints.

| | Dalgo (Chat with Data) | PostHog (Max) |
|---|---|---|
| Session row | `ChatWithDataSession` | `Conversation` |
| Checkpointer | Stock `AsyncPostgresSaver` (psycopg3 pool, bypasses ORM) | Custom `DjangoCheckpointer` on Django models |
| Checkpoint tables | `checkpoints`, `checkpoint_blobs`, `checkpoint_writes` — created by a setup command, invisible to Django | `ConversationCheckpoint`, `...Blob`, `...Write` — normal models, FK → `Conversation`, cascade delete |
| Messages stored twice? | No — checkpointer is the single source of truth; UI history is replayed from it (`ddpui/core/ai/chat/history.py`) | No — same replay approach (plus `messages_json` for their non-LangGraph runtime) |
| Long-term memory | None yet (table cards describe the warehouse, not the org) | `CoreMemory` (per-team fact sheet) + `AgentMemory` (embedded, retrieval-based) |
| Checkpoint cleanup | None yet | Compaction sweep after 7 idle days |

---

## The migration timeline (what they built, in order, and why)

```
POSTHOG AI MIGRATIONS — evolution of Max's storage
══════════════════════════════════════════════════════════════════════

ee/ app (where it all started)
│
├─ 0018  ┌───────────────────────────────────────────────┐
│        │ FOUNDATION                                    │
│        │  Conversation                                 │
│        │  ConversationCheckpoint      ◄─┐              │
│        │  ConversationCheckpointBlob    ├ custom Django│
│        │  ConversationCheckpointWrite ◄─┘ checkpointer │
│        └───────────────────────────────────────────────┘
├─ 0019   blob refactor: + checkpoint_ns, + thread FK on blobs
├─ 0020   CoreMemory                    ── long-term org memory
├─ 0021   Conversation.status           ── concurrency control
├─ 0024   Conversation.type             ── chats vs tool-call threads
├─ 0026   + title, created_at, updated_at, index(updated_at)
│                                        ── conversation history UI
├─ 0028   type += "deep_research"       ── new agent mode
├─ 0031   AgentArtifact                 ── agent outputs (viz, notebooks)
│
├─ 0033   type += "slack"        ─┐
├─ 0034   + slack_thread_key,     ├───── Max in Slack
│         + slack_workspace_domain│
├─ 0035   unique(team, slack_key) ┘  (index built CONCURRENTLY)
│
├─ 0036   + is_internal                 ── hide support-agent sessions
├─ 0037   + approval_decisions          ── human-in-the-loop approvals
├─ 0039   + messages_json,              ── 2nd runtime ("sandbox",
│         + sandbox_task/run_id            no LangGraph checkpoints)
├─ 0044   + deleted, deleted_at         ── soft delete
│
├─ 0048 ─┐ SeparateDatabaseAndState: models MOVE OUT of ee/
├─ 0049 ─┘ (state-only — tables keep their ee_* names, zero downtime)
│               │
▼               ▼
products/posthog_ai/ app (current home)
│
├─ 0001   AgentMemory                   ── embeddings-backed recall
├─ 0002   adopts the moved models       ── pairs with ee/0048–0049
├─ 0003   Conversation.topic            ── classify chats for analytics
└─ 0004   + agent_runtime, + task FK    ── formalize sandbox runtime,
                                           deprecate 0039's raw UUIDs
```

### Migration-by-migration: the need behind each one

**`ee/0018` — the foundation.**
Everything at once: the `Conversation` session record plus the three checkpoint
tables backing their custom checkpointer. From day one PostHog chose ORM-managed
checkpoint storage over LangGraph's stock saver — the decision Dalgo made the
other way.

**`ee/0019` — blob schema fix for subgraphs.**
Added `checkpoint_ns` (namespace — which subgraph node a checkpoint belongs to,
e.g. `"child|grandchild"`) and a direct `thread` foreign key on blobs. LangGraph
versions blobs per *thread + channel*, not per checkpoint; the original
lookup-through-checkpoint was wrong once multi-node graphs arrived.

**`ee/0020` — `CoreMemory`.**
First long-term memory: one plain-text fact sheet per team (max 10,000
characters), seeded by scraping the customer's website during onboarding, then
edited by the agent and the user over time. This is how Max knows "this company
sells B2B SaaS; activation means completing a survey" in every conversation.

**`ee/0021` — `status`.**
`idle / in_progress / canceling` — a lock so two turns can't run on the same
conversation at once, and a place for "stop generating" to live.

**`ee/0024`, `0028`, `0033` — `type`.**
Max's tools, deep-research mode, and later Slack all run graphs that need
checkpoint threads. `type` keeps tool-call threads out of the user's chat
sidebar. Example: when Max internally calls a "create insight" tool that itself
runs a graph, that thread is `type=tool_call` and never appears in history.

**`ee/0026` — history UI.**
Titles, timestamps, and an index on `updated_at`. Needed the moment there is a
"past conversations" list sorted by recency — Dalgo already has the equivalent
on `ChatWithDataSession`.

**`ee/0031` — `AgentArtifact`.**
The agent started producing durable outputs (visualizations, notebooks) that
must outlive the message stream and be addressable by a short 4-character id.

**`ee/0034`, `0035` — Slack.**
A `{workspace}:{channel}:{thread_ts}` key maps each Slack thread to exactly one
conversation (unique constraint), so a reply in the same Slack thread resumes
the same memory. `0035` builds its index `CONCURRENTLY` via raw SQL — the
big-table lesson: never take a table lock on a hot production table.

**`ee/0036` — `is_internal`.**
PostHog support agents can impersonate customers. Their test chats are stamped
internal and hidden from the customer's history.

**`ee/0037` — `approval_decisions`.**
Human-in-the-loop: when the agent proposes a dangerous operation, the
pending/approved/rejected state must survive across requests and page reloads.

**`ee/0039` → `posthog_ai/0004` — the sandbox runtime.**
A second, non-LangGraph execution mode storing messages as plain `messages_json`.
`0039` used raw UUID pointers; `0004` replaced them with a real `task` foreign
key and an `agent_runtime` discriminator stamped at creation. A visible
"iterated in production" trail — the first cut of a new runtime rarely has the
right schema.

**`ee/0044` — soft delete.**
Same reasoning as Dalgo's `deleted_at`: users delete chats; audit and recovery
need the row to survive.

**`ee/0048`–`0049` + `posthog_ai/0001`–`0002` — the app move.**
`SeparateDatabaseAndState` (a Django migration type that changes Django's
bookkeeping without touching the database) relocated all AI models from `ee/`
into `products/posthog_ai/`. Tables kept their `ee_*` names — zero downtime,
pure code reorganization as AI became a product line.

**`posthog_ai/0001` — `AgentMemory`.**
Second memory system: many small memories per team/user, pushed through their
embedding pipeline so the agent *retrieves* relevant ones per question
(RAG-style — Retrieval-Augmented Generation) instead of injecting one big blob
like `CoreMemory`.

**`posthog_ai/0003` — `topic`.**
Classify each conversation's product domain from the first question — analytics
on what people actually ask the assistant.

### Not in the timeline, but part of the story: compaction

**The rule:** Checkpointing writes a full state snapshot every graph step, so
checkpoint tables grow much faster than message count.

**Example:** One 10-question chat can produce 50+ checkpoints (every tool call
and node transition snapshots). PostHog's sweep (`ee/hogai/django_checkpoint/compaction.py`)
finds conversations idle for 7+ days and deletes intermediate checkpoints and
blobs while keeping the conversation resumable — they measured that ~99.6% of
resumes happen within a week.

**Why it matters:** Dalgo's stock saver keeps every checkpoint forever. Fine at
NGO scale today; it is the known long-term storage liability of our design.

---

## Final shape of PostHog's AI storage

```
                        ┌────────────────┐
   Team ───────────────►│  Conversation  │◄─────────── User
   │                    └───────┬────────┘
   │       ┌────────────────────┼────────────────────┐
   │       ▼                    ▼                    ▼
   │  ConversationCheckpoint  AgentArtifact   messages_json (sandbox)
   │       │        │
   │       ▼        ▼
   │   ...Blob   ...Write     ◄── the actual message/state bytes
   │
   ├──► CoreMemory   (1 per team, plain-text facts, 10k chars)
   └──► AgentMemory  (many per team/user, embedded for retrieval)
```

---

## What this predicts for Dalgo

PostHog started with exactly what Dalgo has today (session + checkpoints), and
every later migration was forced by a real production need. Likely order for us:

1. **A `status` field on `ChatWithDataSession`** the first time a user double-sends
   or asks to cancel a slow answer (their `0021`).
2. **Checkpoint compaction or TTL** once the `checkpoints` table is measured in
   gigabytes (their `compaction.py`) — harder for us because our tables are
   outside the ORM.
3. **A `CoreMemory`-style org fact sheet** when answers need context the warehouse
   schema can't provide — e.g. Priya's org calls beneficiaries "shakti members",
   and the agent should know that in every chat. Our table cards describe tables,
   not the organization.
4. **Artifacts** if chat answers start producing saved charts or reports that
   should outlive the conversation (their `0031`).
5. **A `type`/scope discriminator** — we already partially have this via
   `scope_type` (org vs dashboard chats).

One operational lesson to adopt now: PostHog's checkpoint tables are covered by
ordinary `manage.py migrate`; ours require the extra `chat_with_data_setup`
command. Every new environment (staging, prod, a teammate's laptop) will hit
`relation "checkpoints" does not exist` until that command runs — worth wiring
into the deploy script rather than tribal knowledge.
