# Chat With Data: Agent Workflow Design (Staff-Engineer Reference)

> Third doc in the series. See also: [`chat-with-data-architecture.md`](./chat-with-data-architecture.md)
> (the RAG-vs-agentic-vs-hybrid *decision*) and [`chat-with-data-build-playbook.md`](./chat-with-data-build-playbook.md)
> (build order, tech stack, PII pipeline, eval harness). **This doc is the end-to-end agent workflow
> design** — stage-by-stage, context engineering, reliability/recovery, single-vs-multi-agent, MVP-vs-prod.
>
> Deep-research backed (24 sources, 116 claims, 24 adversarially confirmed). **[verified]** = survived a
> 3-vote panel; **[judgment]** = engineering call grounded in Dalgo's stack, not web-verified. Vendor
> blogs are existence proofs of *patterns*, not comparative benchmarks. Two research frameworks
> (SQL-of-Thought, ReFoRCE) are 2025 academic preprints — research-validated, not production-validated.
> Date: 2026-06-19.

---

## The one-paragraph thesis

Every production team converges on the **same shape**: a *staged, mostly-deterministic pipeline* with the
LLM confined to **small specialized units of work** — not one monolithic prompt, not a freewheeling ReAct
agent. The pipeline is: **classify intent + route to a domain → scope & retrieve a pruned, semantic-layer-
grounded context (never raw schema) → generate SQL → validate deterministically (EXPLAIN/dry-run) *before*
executing → on error, run a *bounded, structured* self-correction loop → execute → analyze results →
generate response**, with clarification and validation gates throughout. Anthropic's own guidance anchors
the architecture call: **default to the simplest workflow; add agentic complexity only when it
demonstrably pays off**, because agents trade latency and cost for flexibility. **[verified]**

---

## 1. Agent workflow design — the principles

**How an agent should process a question (the verified consensus):**

1. **Decompose, don't monolith.** Uber, AWS, Google, and the research frameworks independently arrive at
   a staged pipeline "because LLMs perform best on small, specialized units of work" (Uber's words). A
   single prompt that does retrieval + generation + validation underperforms. **[verified]**
2. **Deterministic-first.** Anything that *can* be code *should* be code — it's faster, cheaper, testable,
   and can't hallucinate. The LLM is the exception, not the default. **[verified]**
3. **Validate before executing, recover with structure.** Catch errors with a parser/EXPLAIN before they
   hit the warehouse; on failure, feed *structured* guidance back — not the raw error in a naive loop. **[verified]**
4. **Ground, then generate.** The semantic layer is the default path; raw SQL is a *guarded fallback* used
   only when defined metrics don't cover the ask (Anthropic, WrenAI, AWS all state this explicitly). **[verified]**

**What is deterministic vs LLM-driven** (the dividing line production teams actually draw):

| Deterministic (code) | LLM-driven |
|---|---|
| Domain/workspace routing (dictionary string-match) | Intent classification (constrained, small label set) |
| Schema/catalog sync, value profiling | Table & column selection (schema linking) |
| SQL parse / EXPLAIN / dry-run validation | SQL generation |
| Allowlist, row limits, RBAC, timeouts | Correction-plan reasoning on a failed query |
| Result-set PII masking | Result narration / explanation |
| Retrieval (vector + keyword) mechanics | Clarification-question generation |

---

## 2. The agent architecture — stage by stage

For each stage: **purpose · inputs · outputs · failure modes · guardrails · who does it (code/LLM)**.

### Stage 0 — Context assembly (build-time + session-open) · *deterministic*
- **Purpose:** pre-assemble everything knowable before the question. Auto-sync catalog schemas; LLM-identify
  categorical dimensions worth profiling; populate **top-K distinct values** so the model knows what
  `status=4` actually means (AWS pattern). Embed table/metric descriptions for in-memory retrieval. **[verified]**
- **In:** dbt manifest, warehouse schema, prior enriched artifact. **Out:** the semantic-layer artifact +
  value profiles + in-memory index + dashboard→table allowlist.
- **Failure modes:** stale artifact after a dbt change. **Guardrail:** staleness fingerprint, recompute on drift.

### Stage 1 — Intent classification + domain routing · *LLM (tiny) ∥ deterministic, in parallel*
- **Purpose:** decide *what kind* of question this is and *which domain* it belongs to, to narrow the
  retrieval radius. AWS runs **two things in parallel**: a lightweight model classifies intent into a small
  fixed set (e.g. `INFO / QUERY / SHOW_METRIC / UNKNOWN`) via few-shot **constrained output**, while domain
  classification is **NOT an LLM** — it's deterministic string-match against an abbreviation/alias dictionary
  (tokenized to avoid false positives). Uber's "Workspaces" do the same: "picking a business domain allowed
  us to **drastically narrow the search radius for RAG**." **[verified]**
- **In:** user question, domain dictionary, conversation history. **Out:** `{intent, domain, confidence}`.
- **Failure modes:** wrong domain → wrong tables downstream; ambiguous intent. **Guardrails:** constrained
  label set; `UNKNOWN`/low-confidence routes to clarification (Stage 4-bis), not to a guess.
- **Dalgo mapping [judgment]:** the dashboard the user is chatting on *is* the domain — your allowlist is
  the workspace. Run a cheap model for intent; keep domain routing deterministic from the allowlist.

### Stage 2 — Metadata discovery + context retrieval (schema linking) · *deterministic retrieval → LLM selection*
- **Purpose:** assemble a **small, relevant** context — the opposite of dumping the schema. Two-stage
  retrieval (Google): (a) vector/semantic search to find candidate datasets/tables/columns; (b) load
  supporting context — schema annotations, similar verified SQL, business rules, recent query history.
  For large schemas, pattern-based table grouping + **LLM-guided schema linking** compresses DB info
  (ReFoRCE reports >96% compression). **[verified]**
- **In:** question + domain scope. **Out:** candidate tables/columns + retrieved exemplars + value literals.
- **Failure modes:** relevant table missed (under-retrieval); context bloat (over-retrieval). **Guardrails:**
  scope to domain first; hybrid BM25+cosine; cap candidate count.

### Stage 2-bis — Table approval (human-in-the-loop) · *user*
- **Purpose:** Uber added this because **auto-picked tables were frequently wrong** and hallucination is
  unsolved. Surface the selected tables; user clicks "looks good" or edits before any SQL is generated. **[verified]**
- **Guardrail [judgment]:** make it confidence-gated (auto-proceed when confident, ask only when not) so it
  doesn't add friction to every query. Needs bidirectional transport → your WebSocket (not SSE).

### Stage 3 — Column prune · *LLM*
- **Purpose:** drop irrelevant columns to fit token limits **and cut latency** (Uber's dedicated Column
  Prune agent). **[verified]**
- **In:** approved tables' full schemas. **Out:** pruned schema. **Guardrail:** never prune below the
  columns the plan references; keep keys/time/grain columns.

### Stage 4 — SQL generation · *LLM*
- **Purpose:** generate the query (or, preferably, a **semantic-layer metric selection** that compiles to
  SQL deterministically — see architecture doc). Research frameworks insert a **query-plan / subproblem**
  step *before* SQL (SQL-of-Thought: schema linking → subproblem id → query plan → SQL), which improves
  accuracy. **[verified]**
- **In:** pruned schema + semantic definitions + exemplars + resolved time scope. **Out:** SQL (+ the plan).
- **Failure modes:** hallucinated tables/columns/metrics (**unsolved** — mitigate, don't assume gone);
  wrong join grain. **Guardrails:** ground in semantic layer + value profiles; prefer metric-compile over
  free SQL; the plan is checkable downstream.

### Stage 5 — SQL validation (deterministic, PRE-execution) · *code*
- **Purpose:** catch errors **before** touching the warehouse. Multi-team standard: run **EXPLAIN / dry-run
  / parse** ("check for syntax errors without actually running the query" — AWS; "a dry run of the
  generated SQL" — Google). This is a non-AI signal. **[verified]**
- **In:** SQL. **Out:** `valid | {error, details}`. **Guardrails:** also enforce allowlist (only approved
  tables), single statement, mandatory `LIMIT`, read-only role. **This catches *syntactic/structural*
  errors only — not semantic ones (§4).**

### Stage 6 — Self-correction loop (bounded + structured) · *LLM, guided*
- **Purpose:** recover from a failed validation/execution. **The key finding:** route the error through a
  **structured correction step** — SQL-of-Thought feeds `{execution error, error taxonomy (9 categories /
  31 types), incorrect SQL}` to a **Correction Plan Agent** that emits a CoT correction plan, looping until
  success or max attempts. Google: "when provided an example of a mistake and some guidance, models can
  typically address what they got wrong." **[verified]**
- **⚠️ Critical (2-1 vote, hold as strong-but-not-unanimous):** **unguided** retry *actively hurts* — "led
  to worse correction and often resulted in the **repetition of the same correction steps**." Do **not**
  just paste the raw error back and retry. **Guardrails:** hard cap on attempts; structured/taxonomy-guided
  feedback; abort to graceful failure if the cap is hit.

### Stage 7 — Execution · *code*
- **Purpose:** run the validated query as the tenant's RBAC role. **Guardrails:** statement timeout, row
  cap, per-tenant connection, cost/row-scan limit.

### Stage 8 — Result analysis + response generation · *code (mask) → LLM (narrate)*
- **Purpose:** turn rows into an answer. **⚠️ PII-critical:** mask the result DataFrame (Presidio +
  column-level classification) **before** the LLM narrates it — the result set is the real leak surface,
  not the prompt (see build playbook §4). **[judgment, grounded in verified Presidio guidance]**
- **Failure modes:** correct query, *wrong narration* (answer doesn't match the rows). **Guardrail:**
  answer-faithfulness check (architecture doc §8); show `[View SQL]`.

---

## 3. Context engineering

The dominant lesson: **ground in a curated semantic layer, not raw schema, and aggressively prune.** **[verified]**

- **Semantic layer as context, not the database schema.** Encode metrics/models as structured objects.
  WrenAI's MDL: "Raw schemas describe storage. MDL describes how data should be *understood and queried*."
  AWS encodes each metric as JSON (domain/name/definition + SQL expression/source/filters/granularity).
  Anthropic: "the governed semantic layer is the **mandatory default path** for every data question; raw
  SQL is the fallback, used only after the semantic-layer path is shown not to cover the ask." **[verified]**
- **Avoid metadata overload** via (a) domain scoping first, (b) two-stage retrieval, (c) schema
  compression / table grouping, (d) a dedicated column-prune step. **[verified]**
- **Value profiling.** Pre-populate top-K distinct categorical values so the model knows what codes mean
  (`status=4` → "cancelled"). Auto-maintained by a deterministic sync job. **[verified]**

**What to pre-assemble vs retrieve dynamically vs keep in session memory [judgment, synthesizing the above]:**

| Pre-assemble (build-time, in context before the loop) | Retrieve dynamically (query-time) | Session memory |
|---|---|---|
| Semantic-layer definitions, allowlist, value profiles, embedded descriptions, column PII taxonomy | Distinct-value lookups for *specific* filters; the actual query run; similar verified-SQL exemplars | Conversation history, resolved entities, prior clarifications, the approved table set |

> dbt metadata, semantic definitions, and business glossary → **pre-assembled** (they change at build time).
> Query history → **session memory** (for follow-ups) + **retrieval** (verified exemplars). Don't dump any
> of it wholesale; scope by domain and prune.

---

## 4. Reliability & recovery

- **Correct SQL** comes from deterministic pre-execution validation (EXPLAIN/dry-run/parse) + grounding,
  not from trusting raw LLM output. **[verified]**
- **Recovery from failed queries:** bounded, **structured** error-feedback loop (taxonomy/correction-plan
  guided). Naive retry repeats the same mistake — avoid it. **[verified, 2-1 on the "naive hurts" part]**
- **Clarifying questions — detect-then-clarify or defer, never guess.** Google orchestrates LLM calls to
  "first try to identify if a question **can actually be answered**… and if not, generate the necessary
  follow-up questions." Anthropic's workflow *starts* by clarifying. A consensus/voting variant (ReFoRCE)
  accepts high-confidence candidates and **defers ambiguous ones** rather than committing. **[verified]**
  - ⚠️ ReFoRCE's "defer" is about *SQL-candidate vote disagreement*, not *natural-language intent*
    ambiguity — don't conflate. NL-intent ambiguity → ask the user.
- **Reducing hallucination:** there is **no full solution** — table/column/metric hallucination "hasn't
  been completely solved" (Uber). Mitigations, layered: **HITL table approval**, semantic-layer + value-
  profiling grounding, deterministic allowlist validation, and a recursive **Validation agent** that tries
  to fix ungrounded references. **[verified]**
- **⚠️ The failure class your validation CANNOT catch:** *semantic* errors that produce **no execution
  error** — wrong join grain, wrong filter, plausible-but-wrong numbers, empty-but-valid results. EXPLAIN/
  dry-run pass these silently. This is *the* open reliability gap (see §9) and the reason execution-accuracy
  evals (result-set comparison, not "did it run") are non-negotiable. **[research open question]**

---

## 5. Evaluations — what to eval at each step

Cross-reference the build playbook §5 for the full harness (datacompy result-set comparison, multiple-
valid-SQL per question, Uber's 4-signal scorecard, pruned-context LLM-judge, Langfuse CI). **Per-stage,
the eval targets are:**

| Stage | What to measure |
|---|---|
| Intent/domain routing | Classification accuracy on a labeled set |
| Table selection | **Table-overlap score** (0–1) vs golden tables (Uber) |
| SQL generation | **Execution accuracy** via result-set comparison + judge-similarity |
| Validation | % of bad SQL caught pre-execution (precision/recall of the gate) |
| Self-correction | Recovery rate within the attempt cap; no-regression on retries |
| **Semantic correctness** | **The critical pre-launch eval** — does the *right-looking* query return the *right numbers*? Result-set match against golden, not "did it run." |
| Response | Answer-faithfulness (narration matches rows) |

**Critical before production launch:** the golden NL→SQL set with **execution-accuracy gating in CI**, and
explicit coverage of the **silent-semantic-error** class. Per the prior docs, *this is Dalgo's blocking gap.*

---

## 6. Security & governance

Cross-reference build playbook §4 (the full PII pipeline) and architecture doc §8. Workflow-level summary:

- **Enforce permissions at the connection, not the prompt** — query runs as the tenant's RBAC role against
  row/column-secured views; the agent is *structurally* incapable of crossing tenants. **[judgment, per
  verified Cortex "RBAC + governance boundary by default"]**
- **PII: the result set is the real surface.** Column-level classification + mask the result DataFrame
  before the LLM narration step. Presidio-on-the-prompt is necessary but insufficient. **[verified basics, judgment on result-set]**
- **Prompt-injection:** Plan-Then-Execute (fix the plan before reading untrusted query results → control-
  flow integrity) and Action-Selector (templated parameterized SQL) for the guarded fallback. **[verified]**

---

## 7. What leading teams are actually doing (2025–2026)

| Team / project | Architecture | Pattern worth stealing |
|---|---|---|
| **Uber QueryGPT** | Multi-agent (Intent / Table / Column-Prune) | Domain "Workspaces" narrow RAG; **HITL table approval**; column prune for latency. **[verified]** |
| **AWS RRDA** (Bedrock) | Fixed multi-stage pipeline | Intent (constrained LLM) ∥ domain (deterministic dict) in parallel; metric-as-JSON dictionary; EXPLAIN-then-self-fix; auto value-profiling. **[verified]** |
| **Google Cloud** | Two-stage retrieval + validation | Vector-find then load-context; dry-run validation fed back for a second pass; can-this-be-answered gate. **[verified]** |
| **Anthropic** | *Two* designs, by use case | Data-Analyst Agent = a **single ReAct loop** (Python sandbox, no SQL); internal text-to-SQL = **fixed multi-stage skill pipeline** with adversarial-review sub-agents (+6% accuracy). Governed semantic layer is the mandatory default path. **[verified]** |
| **WrenAI** (OSS) | Agent over an MDL semantic layer | "Correctness as primitives": rich schema retrieval, **dry-plan validation, structured errors with hints, value profiling**, eval runner (partly roadmap). **[verified]** |
| **Vanna 2.0** (OSS) | Agentic, tool-based | Reframed from prompt-to-SQL to **Agentic Retrieval** + tool registry (`RunSqlTool`) — a complete redesign from the legacy single-stage class. **[verified]** |
| **dbt** | Structured context / MCP | Semantic models + MetricFlow as governed, compilable context. (See architecture doc.) |

**Cross-team lesson:** the spectrum is real and *chosen deliberately* — single ReAct loop (simplest),
fixed multi-stage pipeline (most production systems), multi-agent decomposition (Uber, at scale). Anthropic:
"find the **simplest** solution possible, and only increase complexity when needed." **[verified]**

---

## Final deliverables

### A. Recommended workflow (step-by-step)

```
BUILD TIME (deterministic): catalog sync → semantic-layer artifact (metrics, joins, grain, PII taxonomy)
                            → value profiling (top-K distinct values) → embed descriptions (in-memory index)
                                                   │
USER QUESTION                                      ▼
  │   1. INTENT (tiny LLM, constrained)  ∥  DOMAIN ROUTE (deterministic dict / dashboard allowlist)
  │        └─ UNKNOWN / low-confidence ─────────────────────────────► CLARIFY (ask user) ──┐
  │   2. RETRIEVE (deterministic 2-stage: vector-find → load context)                       │
  │        └─ schema-link / table-select (LLM)                                              │
  │   2b. TABLE APPROVAL (human-in-the-loop, confidence-gated)  ◄── edit ──► user           │
  │   3. COLUMN PRUNE (LLM)                                                                  │
  │   4. PLAN → SQL  (LLM; prefer metric-compile over free SQL)                             │
  │   5. VALIDATE (deterministic: parse / EXPLAIN / dry-run + allowlist + LIMIT)            │
  │        └─ fail ─► 6. STRUCTURED CORRECTION (taxonomy-guided, bounded) ─┐                │
  │                          └── cap hit ──► graceful failure / fallback ──┘                │
  │   7. EXECUTE (tenant RBAC role, timeout, row cap)                                       │
  │   8. MASK RESULT-SET (deterministic PII) → NARRATE (LLM) → FAITHFULNESS CHECK           │
  └────────────────────► STREAM answer + progress events + [View SQL] ◄────────────────────┘
```

### B. Recommended agent architecture
A **fixed multi-stage pipeline** (not a free ReAct agent, not a monolith) where 3–4 stages are LLM calls
(intent, table-select, column-prune, generate) and the rest is deterministic code, with **two control
gates**: HITL table approval and bounded structured correction. This is the Uber/AWS/Google shape adapted
to Dalgo's allowlist-as-domain.

### C. Key design decisions & tradeoffs
- **Fixed pipeline over ReAct loop** — predictable, debuggable, cheaper; you give up some open-ended
  flexibility (acceptable: chat-with-*dashboard* is a scoped domain). **[verified principle]**
- **Semantic-layer default, raw SQL guarded fallback** — accuracy + governance vs coverage of the long tail.
- **HITL table approval** — accuracy/trust vs friction → mitigate by confidence-gating it.
- **Structured correction over naive retry** — recovery that works vs a loop that repeats mistakes.
- **Deterministic validation pre-execution** — cheap safety, but **cannot catch semantic errors** → evals must.

### D. Common mistakes to avoid
1. **One monolithic prompt** doing retrieval+generation+validation. (Decompose.)
2. **Dumping the raw schema** into context. (Scope, link, prune.)
3. **Naive/unguided retry** on errors — it repeats the same wrong fix. (Structure the feedback, bound it.)
4. **Trusting "it ran" as "it's correct"** — semantic errors return plausible wrong numbers silently.
5. **Guessing on ambiguous questions** instead of clarifying or deferring.
6. **PII masking only the prompt**, ignoring the result set the user actually sees.
7. **Permissions in the prompt** instead of the DB connection/RBAC.
8. **Adding multi-agent complexity before it pays off** — start simplest.

### E. MVP vs Production *(thinly evidenced — [judgment], flagged as open question §9)*

| | **MVP (v1)** | **Production (hardened)** |
|---|---|---|
| Scope | One dashboard's allowlist | Multi-tenant, many dashboards |
| Pipeline | Intent → retrieve → generate → validate(EXPLAIN) → execute → narrate | + column-prune, query-plan step, HITL approval, structured correction loop |
| Context | Semantic artifact + schema | + value profiling, verified-SQL exemplars, session memory |
| Reliability | EXPLAIN gate + 1 retry | Taxonomy-guided bounded correction + validation agent + faithfulness gate |
| Ambiguity | Clarify on `UNKNOWN` | Confidence-gated clarify/defer |
| PII | Block known PII columns | Full column taxonomy + result-set masking + RBAC views |
| Evals | ~30 golden Qs, manual | Execution-accuracy CI gate + semantic-error coverage + Langfuse prod monitoring |
| Cost/latency | Single capable model | Tiered models (cheap intent / capable generate), prompt+result caching |

### F. What I'd build today from scratch (for Dalgo)
A **fixed multi-stage Python pipeline** in the Django service: tiny model for intent + deterministic
allowlist routing → in-memory hybrid retrieval over the semantic artifact → confidence-gated HITL table
approval over the existing WebSocket → column prune → metric-compile-or-guarded-SQL generation →
deterministic EXPLAIN/allowlist validation → bounded taxonomy-guided correction → RBAC-scoped execution →
result-set PII masking → narration + faithfulness check. Orchestrate with **LangGraph or Pydantic AI**
(typed graph, not a crew), gateway through **LiteLLM** (Presidio guardrail attaches here), observe with
**Langfuse** (experiments in CI, observation-level in prod). **Build the golden-set execution-accuracy eval
FIRST** — it's the gate that catches the silent-semantic-error class nothing else can. **[judgment]**

---

## 8. Caveats
- Sources skew to first-party vendor/engineering blogs (AWS, Uber, Google, Anthropic) — authoritative for
  "what this team built," self-reported, not independently benchmarked. Treat as **pattern existence
  proofs**, not comparative performance.
- SQL-of-Thought & ReFoRCE are 2025 arXiv preprints on academic benchmarks (Spider/Spider 2.0) — the
  error-taxonomy and "unguided correction is worse" results are research-validated (the latter 2-1, not
  unanimous), **not** production-validated.
- **Refuted, excluded:** the specific "Uber fetches exactly 3 tables and 7 SQL samples" figure (0-3) — the
  *pattern* (domain narrows RAG) holds, the *counts* did not. Don't cite retrieval counts.
- MVP-vs-production progression and concrete latency/cost budgets were only principle-level in the sources.

## 9. Open questions (your design calls / things to measure)
1. **Silent semantic errors** — how to detect wrong-but-non-erroring queries (wrong grain, plausible wrong
   numbers) beyond golden-set result comparison. The single most important unsolved reliability gap.
2. **MVP→prod progression** — which specific failures trigger adding which stage; measure, don't guess.
3. **Latency/cost budgets** per query for fixed-pipeline vs multi-agent at Dalgo's volume — instrument first.
4. **Eval cold-start** — building the golden set with no query logs under multi-tenant PII constraints.

---

## Sources (primary, verified this round)
- AWS — conversational data assistant (RRDA, text-to-SQL with Bedrock): https://aws.amazon.com/blogs/machine-learning/build-a-conversational-data-assistant-part-1-text-to-sql-with-amazon-bedrock-agents/
- Uber QueryGPT: https://www.uber.com/en-GB/blog/query-gpt/
- Google Cloud — techniques for improving text-to-SQL: https://cloud.google.com/blog/products/databases/techniques-for-improving-text-to-sql
- Anthropic — building effective agents: https://www.anthropic.com/research/building-effective-agents
- Anthropic — self-service data analytics with Claude: https://claude.com/blog/how-anthropic-enables-self-service-data-analytics-with-claude
- Anthropic — Data Analyst Agent cookbook: https://github.com/anthropics/claude-cookbooks/blob/main/managed_agents/data_analyst_agent.ipynb
- SQL-of-Thought (arXiv 2509.00581): https://arxiv.org/pdf/2509.00581
- ReFoRCE (Snowflake Labs): https://github.com/Snowflake-Labs/ReFoRCE
- WrenAI (MDL semantic layer): https://github.com/Canner/WrenAI
- Vanna 2.0 (agentic redesign): https://github.com/vanna-ai/vanna
- dbt — structured context for AI: https://www.getdbt.com/blog/bringing-structured-context-to-ai-with-dbt
- OpenAI — in-house data agent: https://openai.com/index/inside-our-in-house-data-agent/
