# LangGraph best practices vs Dalgo's Chat with Data

Audience: Dalgo engineers deciding what to adopt next. Companion to
`langgraph-two-agent-architecture.md` (read that first for how our system works).
Every external claim below was verified by fetching the source in September 2026.

## 1. Sources

| Source | What it is | Date | Link |
|---|---|---|---|
| LangChain docs: Multi-agent | Official patterns (subagents, handoffs, skills, router) with call/token benchmarks | living docs, fetched 2026-09 | https://docs.langchain.com/oss/python/langchain/multi-agent |
| LangChain docs: Build a SQL agent | Official text-to-SQL tutorial: tools, safety, query checker | living docs, fetched 2026-09 | https://docs.langchain.com/oss/python/langchain/sql-agent |
| LangChain docs: Human-in-the-loop | HumanInTheLoopMiddleware, decision types, resume | living docs, fetched 2026-09 | https://docs.langchain.com/oss/python/langchain/human-in-the-loop |
| LangChain docs: Middleware | Hooks + built-in middleware, when to use vs custom graphs | living docs, fetched 2026-09 | https://docs.langchain.com/oss/python/langchain/middleware |
| LangGraph docs: Durable execution | Checkpointers, thread_id rules, production guidance | living docs, fetched 2026-09 | https://docs.langchain.com/oss/python/langgraph/durable-execution |
| LangGraph docs: Case studies | Official list of production users (Uber, Replit, Klarna, LinkedIn, Elastic...) — names, no metrics | living docs, fetched 2026-09 | https://docs.langchain.com/oss/python/langgraph/case-studies |
| Deep Agents docs: Context engineering | Offloading + summarization thresholds LangChain ships | living docs, fetched 2026-09 | https://docs.langchain.com/oss/python/deepagents/context-engineering |
| langgraph-swarm-py README | Swarm pattern: active_agent persistence, Command.PARENT handoffs | fetched 2026-09 | https://github.com/langchain-ai/langgraph-swarm-py |
| LinkedIn Engineering: Practical text-to-SQL | Production SQL Bot: knowledge graph retrieval, validators, evals | 2024-12-09 | https://www.linkedin.com/blog/engineering/ai/practical-text-to-sql-for-data-analytics |
| Cognition: Don't Build Multi-Agents | Case against parallel subagents; share full traces | 2025-06-12 | https://cognition.com/blog/dont-build-multi-agents |
| Anthropic: Multi-agent research system | Orchestrator-worker lessons: task descriptions, effort scaling, failure modes, evals | 2025-06-13 | https://www.anthropic.com/engineering/built-multi-agent-research-system |

Not used: Klarna/Uber metrics circulating in secondary posts (85M users, 21k hours
saved) — the official case-studies page lists the companies but no numbers, and I
could not fetch a primary source for the numbers.

## 2. Pattern-by-pattern comparison

### Routing: supervisor agent vs cheap classifier

| Industry | Dalgo |
|---|---|
| LangChain names four patterns. "Router": a classification step directs input to specialized agents — 3 calls for a one-shot request, the cheapest along with skills. "Subagents" (supervisor calling agents as tools) costs 4+ calls and repeats the full flow on every follow-up. | `route_node` is one cheap LLM call, no tools, no loop. `route_decision` turns the label into an edge. |

**Verdict: aligned.** We picked the pattern the official benchmark says is cheapest
for single-domain turns. A supervisor agent would add a model call and latency per
turn for users on slow connections.

### Handoff mechanics: Command.PARENT vs return_direct + edge

| Industry | Dalgo |
|---|---|
| Swarm handoff tools return `Command(goto=..., graph=Command.PARENT)` — the child node updates the parent graph's state and jumps directly to the target agent. | `handoff_to_platform_guide` is `return_direct=True`; the agent loop ends on the ToolMessage, and the parent's `after_sql_agent` conditional edge detects the handoff and routes to `guide_agent`. |

**Verdict: diverged, with reason.** Same effect, but our way (a) ends the turn
segment on a ToolMessage, dodging Anthropic's prefill rejection when the next agent
starts; (b) keeps all topology visible in the parent graph instead of hidden inside
a tool; (c) needs no swarm dependency. Cost: handoff detection is a
history-scanning function (`turn_handed_off`) rather than an explicit state field.

### Context passing between agents

| Industry | Dalgo |
|---|---|
| Two camps. LangChain "subagents": isolated context per subagent, compressed outcome returned — strong isolation, repeated calls. Cognition: "Share context, and share full agent traces, not just individual messages" — partial context causes conflicting decisions. Swarm default: one shared `messages` key across all agents. | Shared full history (swarm-style) so the guide sees everything the SQL agent discovered. Poisoning cost paid via three mitigations: `repair_foreign_tool_errors` (durable by-id message replacement), prompt rule 9, router `last_responder_line`. |

**Verdict: aligned with Cognition/swarm; diverged from LangChain's isolation camp,
with reason.** Our handoffs happen mid-conversation ("go ahead" → build the chart
we just discussed) — a compressed summary would drop the table names and values
the guide needs. The invalid-tool-error poisoning we hit is exactly the class of
cross-agent confusion Cognition predicts; our repair middleware is the price of
sharing. No public equivalent of the by-id durable repair was found.

### Human-in-the-loop approval

| Industry | Dalgo |
|---|---|
| Official: `HumanInTheLoopMiddleware` with `interrupt_on` per tool; four decision types (approve / edit / reject / respond); a `when` predicate can gate only risky argument values; "You must configure a checkpointer to persist the graph state across interrupts." The SQL tutorial suggests interrupting before `sql_db_query`. | Same middleware. Two pause kinds on one mechanism: approval (execute_sql + guide creation tools) and question (ask_user answered via "respond"). Paused turns durable: graph state in Postgres, pending card in Redis 24h; survives server restart. |

**Verdict: aligned.** Two gaps: we don't expose **edit** (let Priya fix the SQL or
chart title before approving — the docs warn to edit conservatively), and we don't
use **`when` predicates** (e.g. auto-approve a plain SELECT under N rows to reduce
approval fatigue).

### Checkpointing and durability

| Industry | Dalgo |
|---|---|
| "Use a persistent checkpointer for production" — PostgresSaver named first. Keep thread_id under 255 chars, use UUIDs. Checkpoints accumulate; clean up periodically (cron). Anthropic: resume from checkpoint instead of restarting was a key production lesson. | `AsyncPostgresSaver` over a shared psycopg3 pool; UUID thread_ids; only the parent graph compiled with the checkpointer, subgraphs inherit; the checkpoint IS the conversation (no parallel message table). |

**Verdict: aligned**, one gap: no checkpoint pruning/TTL job. Checkpoint tables
grow forever; the docs explicitly recommend periodic cleanup.

### SQL guardrails

| Industry | Dalgo |
|---|---|
| Official tutorial: prompt instruction ("DO NOT make any DML statements"), an LLM-based `sql_db_query_checker`, LIMIT-5 defaults, and the honest caveat that its tools are "not intended to be secure or used in production"; scope DB permissions narrowly. LinkedIn: validators check table/field existence + syntax via EXPLAIN, then a self-correction agent fixes errors before delivery. | `guards/sql_guard.py`: sqlglot **AST** validation — SELECT-only enforced structurally (forbidden node types anywhere in the tree, so a DELETE inside a CTE is caught), schema allowlist, LIMIT injected/clamped, comment tricks can't bypass node-type checks. Plus execution timeout, row caps, human approval gate, and errors returned as tool text for model self-correction. |

**Verdict: ahead of the official tutorial** (structural parse-tree checks beat
prompt rules and LLM checkers), aligned with LinkedIn's validate-then-self-correct
loop. Two hardening ideas from the industry: an EXPLAIN dry-run before execution
(LinkedIn) and read-only warehouse credentials as defense-in-depth under the guard
(official docs) — our guard is one sqlglot parsing bug away from being the only
write barrier.

### Retry budgets and runaway loops

| Industry | Dalgo |
|---|---|
| Anthropic's failure list: agents spawning 50+ subagents, endless searching, continuing past sufficient results. Their fix: explicit effort-scaling rules embedded in prompts (simple query = 3-10 tool calls). LangChain ships ModelCallLimit / tool-retry middleware. | `sql_retry_limiter`: after 5 failed execute_sql calls, deterministic `jump_to: "end"` with an apology. The prompt states the same budget so the model stops gracefully first. `recursion_limit=160` sized by a realistic-turn test as backstop. |

**Verdict: aligned.** Stating the budget in the prompt AND enforcing it in
middleware is exactly Anthropic's lesson (guidance in prompt, hard limit in code).

### Prompt engineering: dynamic prompts, org memory, examples

| Industry | Dalgo |
|---|---|
| Middleware docs: dynamic prompt transformation is a first-class hook. Anthropic: each agent needs objective, output format, tool guidance, task boundaries — vague handoffs caused duplicated work. LinkedIn: per-user custom instructions and **certified example queries** in a vector store, injected into generation; a knowledge graph of metadata + query logs for retrieval. | `@dynamic_prompt org_system_prompt` rebuilds the system prompt per model call from runtime context (dialect, schema allowlist, curated org memory). Handoffs carry a one-line summary. `retrieve_context_node` primes the SQL agent before it runs. |

**Verdict: aligned on mechanics; behind on retrieval depth.** We inject curated
org memory; LinkedIn additionally retrieves verified example SQL per question.
Few-shot certified queries were central to their 95%-satisfaction accuracy.

### Context-window management

| Industry | Dalgo |
|---|---|
| Deep Agents: offload tool results >20k tokens to files with a path + preview; **summarize at 85% of the window**, keep 10% recent, write a canonical record to the filesystem; the summary keeps session intent, artifacts, next steps. | `trim_history` caps the model request at 60k tokens (request-only view; checkpoint keeps everything). `ContextEditingMiddleware` drops bulky old tool results past 40k, keeping the 5 newest. |

**Verdict: aligned in spirit, one gap.** We trim; the industry trend is
**summarize**. On a very long thread our trim silently drops the oldest turns —
the user's original goal can fall out of the request while a summary would keep it.
Our checkpoint already plays the "canonical record" role.

### Evals

| Industry | Dalgo |
|---|---|
| LinkedIn: 130+ question benchmark across 10 product areas; metrics = table recall, hallucination rate, syntax correctness; LLM-judge aligned with humans within 1 point only ~75% of the time. Anthropic: one LLM judge with a rubric beat specialist judges; judge end state not process; human testing stays essential. | Golden JSONL in git, seeded to Langfuse; **gates are executed comparisons** (router intent equality; gold SQL and agent SQL both executed and results compared), **judges only inform** — measured after `eval_sql_judge` false-failed 8 of 11 items that provably return identical results. Judges run on a different model family. Canary subset for cheap loops. |

**Verdict: ahead on gate design** — executing both SQLs beats judging text, and we
measured judge unreliability just like LinkedIn did. **Behind on scale and
automation**: dataset is ~14 items vs 130+, `expected_tables` is collected but
unscored (LinkedIn's table recall is their headline metric), and nothing runs in CI
— evals are run by hand.

### Observability

| Industry | Dalgo |
|---|---|
| Anthropic: full production tracing of decision patterns was required to debug non-deterministic agents. LangChain assumes LangSmith. | Langfuse: trace per turn, spans per stage and tool, org/env/model/agent tags, eval traces tagged for filtering. GraphInterrupt arrives as an error callback and is deliberately not logged as an error. |

**Verdict: aligned.** Langfuse vs LangSmith is a vendor choice, not a gap.

## 3. Where we're ahead

- **AST-based SQL guard.** The official tutorial's own docs call its approach demo-grade; ours is structural and covered by tests.
- **Execution-based eval gates.** Both SQLs run against the warehouse; judges never veto a merge. LinkedIn's data (judge ≈ human only 75%) confirms the design.
- **Durable cross-agent state repair.** `repair_foreign_tool_errors` rewrites poisoned ToolMessages by id, once, in the checkpoint. No published equivalent found.
- **PII middleware that rewrites state**, so redacted values are what get checkpointed and traced — not just masked at display time.
- **Recursion limit sized by a test**, not a guess — a realistic messy turn must fit, and adding middleware breaks the test, not production.
- **Single persistence story.** The checkpoint is the conversation; no message-table drift to debug.

## 4. Gaps worth adopting (prioritized)

| # | Gap | Source | Effort | Adopt when |
|---|---|---|---|---|
| 1 | Certified example queries: retrieve verified SQL for similar past questions and inject as few-shot context | LinkedIn | M | Golden-set accuracy plateaus, or users keep asking variations of the same questions |
| 2 | Eval canary run in CI (the `--tag canary` subset already exists, ~30¢/run) | Anthropic, LinkedIn | S | A second engineer starts editing prompts/tools |
| 3 | Summarization middleware for long threads instead of silent trim (85%-of-window trigger; keep intent + artifacts) | Deep Agents docs | M | Real threads start hitting the 60k trim cap |
| 4 | Table-recall eval score — `expected_tables` is already in the dataset, unscored | LinkedIn | S | Next eval-runner touch |
| 5 | HITL **edit** decision — let the user fix SQL/title in the approval card | Official HITL docs | S/M (needs UI) | Users reject queries that were almost right |
| 6 | Checkpoint pruning job (TTL on old threads) | Durable-execution docs | S | Checkpoint tables show up in DB growth review |
| 7 | Read-only warehouse credentials beneath the AST guard (defense-in-depth) | Official SQL docs | M (per-warehouse setup) | Adding a new warehouse type, or first guard bypass scare |
| 8 | `when` predicate to auto-approve trivially safe SELECTs | Official HITL docs | S | Users report approval fatigue |

## 5. Things the industry does that we deliberately rejected

| Industry practice | Why we said no |
|---|---|
| Persistent `active_agent` state (swarm remembers the last agent and resumes there) | Our router re-classifies every turn; `last_responder_line` gives follow-up stickiness without a sticky state that can strand the user with the wrong agent after topic changes. |
| Supervisor agent orchestrating subagents as tools | An extra agent call per turn for users on slow connections; the official benchmark itself shows router is cheaper for single-domain turns, and our two domains are cleanly separable by intent. |
| `Command.PARENT` handoff tools | `return_direct` + a parent conditional edge ends the turn on a ToolMessage (avoids Anthropic prefill rejection), keeps topology in one visible place, and drops a dependency. |
| Isolated per-agent context with compressed handoff summaries (LangChain subagents pattern) | Our handoffs are mid-conversation; the guide needs the exact tables, columns, and agreements just discovered. We chose Cognition's "share full traces" and paid for it with the repair middleware. |
| Agent-written long-term memory | Org memory is curated and injected read-only via the dynamic prompt; agents have no memory-write tool. An agent silently remembering things about an NGO's data without consent is not acceptable for our users. |
| Parallel subagents | Anthropic's parallel research pattern fits open-ended research with token budgets to burn; a chat answer about one warehouse gains nothing and inherits the conflicting-decision failure mode Cognition documents. |
| LLM-judge-gated merges | Measured, not assumed: our SQL judge false-failed 8/11 correct queries. Judges inform; executed comparisons gate. |
