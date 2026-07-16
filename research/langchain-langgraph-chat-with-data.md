# LangChain & LangGraph for Chat-with-Data: How Others Build It

> Deep-research report, July 2026. 5 search angles → 19 sources fetched → 95 claims
> extracted → top 25 adversarially verified (3 independent votes each): 20 confirmed,
> 5 refuted. Confidence labels reflect that verification. Audience: the Dalgo
> chat-with-data team (router → agent loop → validator on LangGraph) deciding which
> patterns to adopt. Companion: [`posthog-agent-architecture.md`](./posthog-agent-architecture.md).

---

## Table of Contents

1. [Headline: The Industry Converged on Your Topology](#1-headline-the-industry-converged-on-your-topology)
2. [The Flagship Production Case Study: LinkedIn SQL Bot](#2-the-flagship-production-case-study-linkedin-sql-bot)
3. [The Official LangChain/LangGraph Patterns](#3-the-official-langchainlanggraph-patterns)
4. [Schema Retrieval: The Highest-Leverage Component](#4-schema-retrieval-the-highest-leverage-component)
5. [SQL Safety: The Verified Hierarchy](#5-sql-safety-the-verified-hierarchy)
6. [State and Checkpointing](#6-state-and-checkpointing)
7. [Evals: A Convergent Taxonomy](#7-evals-a-convergent-taxonomy)
8. [What Did NOT Survive Verification](#8-what-did-not-survive-verification)
9. [Open Questions](#9-open-questions)
10. [What This Means for Dalgo](#10-what-this-means-for-dalgo)

---

## 1. Headline: The Industry Converged on Your Topology

Production chat-with-data systems on LangChain/LangGraph converge on a **hybrid
topology, not a pure ReAct loop**: deterministic pipeline stages (routing, schema
context, validation) wrapped around a bounded agent loop with a self-correction
feedback edge. This is true of the strongest production case study (LinkedIn SQL
Bot), of LangChain's own official SQL-agent tutorial, and of the LangGraph docs'
first-class workflow patterns (routing, evaluator-optimizer).

The router → agent loop → validator shape is not a Dalgo idiosyncrasy — it is the
convergent industry design. The differences worth studying are in the *components*:
how schema context is retrieved, where safety is enforced, and how evals are run.

Notably: **no confirmed evidence surfaced of teams abandoning these frameworks for
this use case.** The contrarian angle (Octomind's "why we left LangChain" post, HN
threads, 12-factor-agents) was searched deliberately; nothing survived adversarial
verification as a chat-with-data-specific abandonment case. Treat that as a research
gap, not proof of universal satisfaction — the known critiques target LangChain's
*abstractions* (chains, prompt hiding), not LangGraph's checkpointing/state layer.

---

## 2. The Flagship Production Case Study: LinkedIn SQL Bot

**Confidence: high (3-0 verified, multiple claims).** Source: [LinkedIn Engineering
blog, Dec 2024](https://www.linkedin.com/blog/engineering/ai/practical-text-to-sql-for-data-analytics),
corroborated by their peer-reviewed paper ([arXiv:2507.14372](https://arxiv.org/abs/2507.14372))
and a [Trino Summit 2024 talk](https://trino.io/assets/blog/trino-summit-2024/trino-summit-2024-linkedin-ai.pdf).

- **What:** multi-agent text-to-SQL "SQL Bot," built on LangChain + LangGraph,
  embedded in their internal DARWIN data-science platform.
- **Schema retrieval as a pipeline, not prompt-stuffing:** embedding-based retrieval
  over a *curated knowledge graph* (table schemas, field descriptions, top-K
  categorical values, example queries) → LLM re-ranker selects top 7 tables from 20
  candidates → second LLM re-ranker filters fields.
- **Validation is deterministic-first with an agentic repair loop:** validators check
  table/field existence and run `EXPLAIN`; errors feed a dedicated self-correction
  agent that can retrieve *additional* tables/fields before rewriting the query.
  This is direct production precedent for a validator feeding back into an agent loop.
- **Evals:** 130+ expert-curated benchmark questions across 10 product areas;
  LLM-as-judge scores within 1 point of human scores 75% of the time; **~60% of
  questions have multiple valid answers**; ground truth refreshed by quarterly expert
  review. Lesson: text-to-SQL evals need multi-answer ground truth and periodic human
  maintenance, not a static test set.

Caveat: self-reported, ~19 months old, metrics not independently audited.

---

## 3. The Official LangChain/LangGraph Patterns

**Confidence: high (3-0 verified).** Sources: [workflows-agents docs](https://docs.langchain.com/oss/python/langgraph/workflows-agents),
[SQL-agent tutorial](https://docs.langchain.com/oss/python/langgraph/sql-agent).

- The docs sanction router → loop → validator explicitly: **routing** (classify
  input, dispatch to context-specific flows) and **evaluator-optimizer** (generate →
  evaluate → feed errors back until acceptance) are documented first-class workflow
  patterns. Agent loops are prescribed for *unpredictable* problem spaces;
  predictable, decomposable stages should be fixed workflow nodes.
- The current official SQL-agent tutorial is itself a **hybrid**: a deterministic
  prefix (`list_tables` and `get_schema` nodes with **forced tool calls** via
  `tool_choice="any"`) → `generate_query` → a `check_query` validation node →
  `run_query`, with a `run_query → generate_query` back-edge for self-correction.
  Fixed pipeline steps wrapping a bounded loop — not pure ReAct.
- Schema context is fetched **agentically at runtime**: `sql_db_list_tables`, then
  `sql_db_schema` returning DDL **plus 3 sample rows per table** (grounding with real
  values, the same instinct as Dalgo's `profile_column`). Suited to small schemas;
  LinkedIn's retrieval pipeline is the pattern for large warehouses.
- Nuance from verification: the docs *demonstrate* this hybrid rather than mandate
  it, and they do NOT frame workflows-vs-agents as a hard binary — real systems mix
  both (that stronger claim was refuted 0-3).

---

## 4. Schema Retrieval: The Highest-Leverage Component

Every serious system treats "which tables/columns does the model see" as the main
accuracy lever:

- **LinkedIn:** knowledge graph + EBR + two LLM re-rank stages (§2).
- **Official docs:** agentic on-demand fetch with sample rows (§3).
- **Peer-reviewed (confidence: medium):** a bidirectional design — table-first and
  column-first retrieval pathways run in parallel over an augmented query (question
  decomposition + keyword extraction), results unioned — closes **roughly 50% of the
  execution-accuracy gap** between full-schema and oracle-schema settings on BIRD-dev
  (+5.41% EX for GPT-4o-mini, +5.77% for Gemini-2.0-Flash), with no SQL-correction
  step ([arXiv:2510.14296](https://arxiv.org/pdf/2510.14296), EACL 2026 Findings,
  [public code](https://github.com/mahadi-nahid/BiSchemaLink)).
- **Important counterweight (why the stronger claim was refuted 1-2):** "full schema
  always degrades quality" is NOT settled. Counter-literature ("The Death of Schema
  Linking?", [arXiv:2408.07702](https://arxiv.org/abs/2408.07702)) argues frontier
  models can win with the full schema when it fits in context. The tradeoff depends
  on model strength and schema size — at NGO scale (dozens of tables), aggressive
  retrieval may matter less than card/description *quality*.

---

## 5. SQL Safety: The Verified Hierarchy

The verified sources establish a clear defense-in-depth ordering, strongest first:

1. **Driver/permission-level read-only enforcement.** AWS's official text-to-SQL
   sample opens the database with `sqlite:///file:...?mode=ro`, so
   INSERT/UPDATE/DELETE/DROP **fail at the driver regardless of prompt injection**
   ([aws-samples/sample-text2sql-deep-agent-evalulation](https://github.com/aws-samples/sample-text2sql-deep-agent-evalulation),
   verified at code level). The transferable pattern: a read-only warehouse
   role/connection, beneath all application logic.
2. **Deterministic validators.** LinkedIn's table/field existence checks + `EXPLAIN`;
   AST-level guards. Code, not prompts.
3. **LLM-based checkers.** The official `check_query` node (NULL handling, UNION vs
   UNION ALL, type mismatches). Useful for *correctness*, not trustworthy for *safety*:
   LangChain's own eval notebook observed models sometimes skip a prompted check step.
4. **Prompt-level prohibitions.** The docs themselves warn verbatim that the demo
   tools are "not intended to be secure or used in production. Use narrowly scoped
   database permissions and add application-specific validation."

The AWS sample adds two production-monitoring ideas: an **online safety evaluator**
scanning every production trace for DML keywords (sampling rate 1.0), and offline
eval assertions that no DML appeared in executed queries. Safety as a continuously
measured property, not a launch-time checkbox.

---

## 6. State and Checkpointing

**Confidence: high** for the core mechanism; specifics of backend offerings refuted.

- Multi-turn state = LangGraph **thread-level persistence**: compile with a
  checkpointer, invoke with `thread_id` in `configurable` — the stable 2026 API that
  deprecated the legacy LangChain memory classes
  ([add-memory docs](https://docs.langchain.com/oss/python/langgraph/add-memory)).
- A checkpoint (full state snapshot) is written after **every super-step** —
  storage grows with turns × steps unless something compacts it. None of the verified
  sources describe a compaction strategy; the only known production example remains
  PostHog's idle-thread sweep (see companion doc §6) — genuinely uncommon knowledge.
- Refuted (0-3): the specific enumeration of production backends
  (PostgresSaver/MongoDBSaver/RedisSaver/OracleSaver each requiring `setup()`
  migrations). The concept is right; re-verify backend specifics against current docs
  before relying on them. (Dalgo already runs `AsyncPostgresSaver` in production —
  that choice is not in question.)

---

## 7. Evals: A Convergent Taxonomy

Two independent official sources ([LangSmith cookbook](https://github.com/langchain-ai/langsmith-cookbook/blob/main/testing-examples/agent-evals-with-langgraph/langgraph_sql_agent_eval.ipynb),
[AWS sample](https://github.com/aws-samples/sample-text2sql-deep-agent-evalulation)) plus
LinkedIn converge on the same eval taxonomy for SQL agents:

| Eval type | What it checks | Example |
|---|---|---|
| Response | final answer vs reference, LLM-as-judge | rubric-graded answer correctness |
| Single-step / first-decision | the agent's *first* tool call is right | expects `sql_db_list_tables` first |
| Trajectory | tool-call sequence (any-order / in-order / exact) | discovery before generation |
| Multi-turn | context-dependent follow-ups resolve correctly | "now break that down by state" |
| Environment/state & safety | side-effects and executed SQL | assert no DML ever executed |
| Online | production traces, continuously scored | DML-keyword scan at sampling 1.0 |

Plus LinkedIn's meta-lessons: multiple accepted answers per question (~60% of their
set) and scheduled human refresh of ground truth. Run as `@pytest.mark.langsmith`
tests in the AWS sample — i.e., evals live in the normal test harness, not a separate
system. (Cookbook repo archived Feb 2026; treat its patterns as conceptual.)

---

## 8. What Did NOT Survive Verification

Worth recording — these are claims a casual read of the sources would repeat:

| Refuted claim | Vote | Why it matters |
|---|---|---|
| LangGraph docs draw a hard workflows-vs-agents binary | 0-3 | Docs encourage mixing; don't cite them for purity arguments |
| Full schema in prompt always degrades quality | 1-2 | Actively debated; depends on model + schema size (§4) |
| The official reference SQL agent is a minimal 2-node loop | 0-3 | That was the *older* cookbook; current docs teach the multi-stage hybrid |
| Specific checkpointer backend list w/ `setup()` migrations | 0-3 | Concept right, specifics stale — re-verify |
| `langchain-ai/text-to-sql-agent` repo is a sequential pipeline | 0-3 | Repo description didn't match; don't cite it |

Also: nothing on framework *abandonment* for chat-with-data survived verification —
the question "who left, and why" remains open (§9).

---

## 9. Open Questions

1. Documented cases of teams moving OFF LangChain/LangGraph for text-to-SQL (to plain
   orchestration, Pydantic-AI, DeepAgents) — at what scale do abstractions become a
   liability?
2. Current recommended production checkpointer backend and its operational
   requirements (the specifics were refuted; needs fresh verification).
3. At what schema size / model capability does retrieval-based schema linking beat
   full-schema-in-context — the unresolved "Death of Schema Linking" debate, asked at
   NGO warehouse scale (dozens to low hundreds of tables)?
4. **Per-user/per-org authorization**: none of the verified sources addressed
   multi-tenancy or row-level security beyond read-only roles. Dalgo's
   `RunContext`-based tenancy is ahead of the published state of the art here.

---

## 10. What This Means for Dalgo

Mapping the verified findings onto the current design (TurnGraph: route →
retrieve_context → sql_agent → validate; AST guard; Haiku validator; M5 table cards):

**Validated by this research — keep:**
- The topology itself. LinkedIn + official docs + LangGraph patterns all converge on
  it. No need to re-litigate.
- AST-guard-fails-closed + LLM-checks-fail-open matches the verified safety hierarchy
  (deterministic > LLM > prompt).
- Table cards + BM25 is a legitimate point on the schema-retrieval spectrum; at NGO
  schema sizes the debate in §4 suggests card *quality* (descriptions, value quirks)
  matters more than retrieval sophistication.
- Multiple-valid-answers eval design (LinkedIn's ~60%) should shape the golden
  dataset from day one — don't build a single-answer eval set.

**Gaps this research exposes:**
1. **No driver-level read-only enforcement.** The AST guard is layer 2 of the
   hierarchy; layer 1 (a read-only warehouse role/connection for chat queries) is
   missing and is the only guarantee that survives a guard bug. Highest-value
   addition from this research.
2. **Validator could gain deterministic pre-checks.** LinkedIn runs existence checks
   + `EXPLAIN` *before* any LLM judgment. An `EXPLAIN` dry-run in `execute_sql` (or
   the guard) catches syntax/reference errors deterministically and cheaply, saving
   agent retry turns.
3. **Self-correction with expanded context.** LinkedIn's repair agent can fetch
   *additional* tables/fields on failure. Dalgo's retry loop reuses whatever context
   the agent already had; after M5, a failed query could trigger injection of the
   next-ranked table cards.
4. **First-decision and trajectory evals** are cheap to add once the eval harness
   exists (e.g. assert discovery-before-SQL on cold questions) and catch regressions
   answer-level evals miss.
5. **Online safety monitoring**: a trivial DML-keyword scan over `sql_queries` in
   `ChatWithDataTurnAudit` rows replicates the AWS online evaluator with zero new
   infrastructure — belt-and-suspenders proof the guard holds in production.

---

*Verification stats: 101 agents, 19 sources, 95 claims extracted, 25 verified
(3 votes each), 20 confirmed / 5 refuted / 0 unverified. All doc-based claims
reflect LangChain/LangGraph documentation as of July 2026; the ecosystem moves fast.*
