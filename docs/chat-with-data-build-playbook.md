# Chat With Dashboard: Production Build Playbook for Dalgo

> Companion to [`chat-with-data-architecture.md`](./chat-with-data-architecture.md). That doc settled
> the **architecture decision** (Hybrid, semantic-layer-first, planner+executor, RAG-for-support-only).
> **This doc is the build playbook** — concrete phases, technology choices, the PII/governance pipeline,
> the eval harness, and current industry patterns. It does *not* re-litigate RAG-vs-agentic.
>
> Deep-research backed (25 sources, 123 claims, 23 adversarially confirmed). Findings are labeled
> **[verified]** (survived a 3-vote panel) or **[judgment]** (engineering call grounded in Dalgo's
> stack, not web-verified). Accuracy numbers from vendor blogs are GPT-4o / GPT-4-Turbo-era
> self-reports — treat as **directional, not targets**. Date: 2026-06-19.

---

## 0. The one-paragraph build order

**Semantic layer first → guarded planner/executor agent → PII & governance pipeline → eval harness in CI
→ observability.** This is the sequence Uber, Snowflake, and the agent-security literature independently
endorse. The single most important reframe from the research: **decompose the work into small specialized
agents** (Uber's words: *"LLMs worked really well when given a small unit of specialized work to do"*),
and **put a human-in-the-loop checkpoint where the user approves the selected tables before any SQL is
generated** — Uber added this specifically because auto-picked tables were frequently wrong. **[verified]**

---

## 1. The reference production architecture: Uber QueryGPT

Uber's QueryGPT is the closest public production system to what Dalgo is building and the canonical
"planner + executor" shape. It decomposes text-to-SQL into **specialized agents**, not one monolithic
prompt: **[verified]**

| Agent | Job | Dalgo mapping |
|---|---|---|
| **Intent / domain agent** | Maps the question to a business domain → *and by extension a curated set of tables + SQL samples for that domain*. This **"drastically narrows the search radius for RAG."** | Use the dashboard the user is chatting on (or its dbt domain) as the scope. You already have the chart→table allowlist — that *is* your domain scope. |
| **Table agent (HITL)** | Selects candidate tables, then **shows them to the user to approve or edit** before generating SQL. Added because *"the tables picked were not correct."* | A "Using these tables — looks right? [edit]" step before the executor runs. Net-new vs the prior design. |
| **Column-prune agent** | Drops irrelevant columns to fit token limits and **cut latency**. | Your build-time enricher already knows the relevant columns per table; prune to those. |

> **Lesson for Dalgo:** the prior doc's "planner/executor" is correct, but make the planner a *small chain
> of narrow agents* (intent → table-select → prune), and **surface the table selection to the user**.
> Cheap to build, and it converts the most common silent failure (wrong table) into a one-click correction.

The benchmark-to-production gap is the empirical reason to bother: Snowflake reports GPT-4o scoring **90%+
on Spider but ~51% on their real-world internal eval** given only a raw schema. **[verified, but: vendor
blog, undisclosed eval set, GPT-4o-era — one data point, not a law.]** The fix the whole industry
converges on is the semantic layer (§2) plus decomposition (above).

---

## 2. Build the semantic layer FIRST (over your dbt models)

Raw schemas are insufficient — they *"lack critical knowledge like business process definitions and
metrics handling."* A semantic model encoding **metrics, dimensions, synonyms, descriptions, and join
paths** is the core accuracy mechanism, and Cortex Analyst keeps **no data/metadata/prompts leaving the
governance boundary** and **fully integrates with RBAC**. **[verified]**

⚠️ *Don't overstate it:* the panel **refuted** the claim that a semantic model *"guarantees"* correct
business-logic application. It is necessary, not sufficient — keep your validator (§5).

**Technology choice for Dalgo [judgment — grounded in your dbt/Postgres stack, not web-verified]:**

- **You already have most of a semantic layer**: the build-time enriched metadata artifact (grain, keys,
  time column, PII flags, answerability) described in the architecture doc. Formalize it, don't restart.
- **Carry forward the prior doc's retrieval conclusion**: your artifact is tiny (tens of tables per
  dashboard), so **in-memory hybrid BM25 + cosine** over embedded table/metric descriptions — **no vector
  DB, no pgvector**. Don't reopen this.
- **dbt MetricFlow / dbt Semantic Layer** is the "proper" path if you want deterministic metric→SQL
  compilation, but it adds operational weight and only covers *modeled* metrics. **Recommendation:** keep
  your own YAML/Postgres artifact as the semantic model now; evaluate adopting MetricFlow later for the
  metrics you can fully model. Open-source text-to-SQL layers (WrenAI, Vanna, Dataherald) are alternatives
  but mean adopting their stack — not worth it when you already have the enricher.

---

## 3. Technology choices, layer by layer

The research deliberately did **not** adversarially verify framework choices (LangGraph vs Temporal,
WS vs SSE, etc.) — those are constraint-driven calls for *your* stack, not web-search questions. Here
are the recommendations, all **[judgment, grounded in Dalgo's Django/Python/Postgres/Superset stack]**:

| Layer | Recommendation | Why for Dalgo |
|---|---|---|
| **Agent orchestration** | **LangGraph** or **Pydantic AI** (pick one; both are native-Python and embed cleanly in a Django service). | You need explicit graph control (intent → table-HITL → execute → validate) with typed state, not a role-playing crew. **CrewAI** is the wrong abstraction (agent personas, not control flow). **Temporal** buys durable long-running execution you don't need at NGO scale yet — revisit only if a single chat run spans minutes and must survive restarts. **OpenAI Agents SDK** ties you to one vendor; avoid for a model-agnostic NGO tool. |
| **LLM gateway** | **LiteLLM proxy** (self-hosted). | One interface across providers, **and it's where your Presidio PII guardrail and prompt/response logging attach** (§4). Model-agnostic matters for cost control and avoiding lock-in. Use the latest capable Claude models as default; keep a cheap model for the intent/classify step. |
| **Retrieval** | **In-memory BM25 + cosine** over the build-time artifact. | Tiny corpus; a vector DB is operational overhead with no payoff. (Carried from architecture doc.) |
| **Streaming / transport** | **Extend the existing WebSocket loop** — don't migrate to SSE. | You already emit `DashboardChatProgressStage` events over WebSocket. WS is also bidirectional, which the **table-approval HITL step** needs (server → "approve these tables?", client → edited selection). SSE is one-way and would force a side-channel for the approval. No reason to rebuild. |
| **SQL execution** | Read-only role, statement timeout, mandatory `LIMIT`, single-statement allowlist. | Deterministic guardrails, not LLM judgment (§5). |
| **State / persistence** | Postgres (you have it) for chat sessions, plan state, per-message timing/cost. | No new infra. |

---

## 4. PII & governance pipeline — the part most likely to be done wrong

There are **two** PII surfaces, and the obvious one is the *less* important:

1. **Prompt PII** (user types a name) — handled by mask-before-LLM.
2. **⚠️ Result-set PII** (the generated query returns a column of beneficiary names, locations, health
   status, caste, disability status) — this streams straight to the user *and into the LLM narration
   step*. **Presidio on the prompt does nothing for this.** For NGO beneficiary data this is the real
   leak and the part a generic "add Presidio" answer gets wrong.

### 4a. Mask-before-LLM (prompt side) — **[verified]**
- **Microsoft Presidio** (MIT, Microsoft-maintained) detects PII/PHI via **NER + regex + rule-based +
  checksum** with context, across languages and entity types (PERSON, EMAIL, PHONE, CREDIT_CARD,
  MEDICAL_LICENSE, …).
- Wire it as a **LiteLLM proxy guardrail** in **`pre_call` mode** so PII is stripped *before* any request
  reaches the LLM. Configure **per-entity `MASK`** (replace with `<PERSON>`) or **`BLOCK`** (reject) —
  `BLOCK` the high-risk categories outright.
- ⚠️ **Microsoft's own docs state there is no guarantee Presidio finds all PII** — *"additional systems
  and protections should be employed."* So Presidio is one layer, never the only one. **[verified]**
- ⚠️ **Implementation caveat:** LiteLLM's PII **restoration** (`output_parse_pii`) has documented failure
  modes in **streaming** and some native-API paths. Masking-in is reliable; **un-masking the response is
  not** — design so correctness never depends on restoration.

### 4b. Column-level classification + result-set governance (the critical layer) — **[judgment, defense-in-depth per verified Presidio guidance]**
This is where you actually protect beneficiaries. Do it **deterministically, at build time and at query
execution**, not by trusting an LLM:

- **Classify every column at build time** (your enricher already emits a `PII` flag — formalize it into
  a taxonomy: direct-identifier / quasi-identifier / sensitive-attribute / safe).
- **Enforce at query execution, not in the prompt:**
  - **Block or mask PII columns in the SELECT** — a query that returns `beneficiary_name` is rejected or
    the column is masked/aggregated before the result leaves the warehouse.
  - **Mask the result DataFrame** with Presidio *again* on free-text columns before it reaches the LLM
    narration step and before it renders to the user.
  - Prefer **aggregates over row-level returns** by policy (chat-with-dashboard is about numbers, not
    name lists) — only allow row-level output for explicitly non-PII columns.
- **RBAC / tenant isolation enforced at the connection, not the agent:** each query runs as the tenant's
  role against row/column-level-secured views. The agent must be **structurally incapable** of crossing
  tenants — never "the prompt says don't." (The architecture doc flagged this as "your responsibility,
  design explicitly." This is the concrete shape.)

### 4c. Prompt-injection defense — **[verified]**
Once the agent ingests **untrusted input — including query results from a multi-tenant warehouse** — that
input must not be able to trigger consequential actions. Apply the patterns from arXiv 2506.08837:
- **Plan-Then-Execute** — fix the plan *before* reading any untrusted tool output → **control-flow
  integrity**: malicious data in a result set *"cannot inject instructions that make the agent deviate
  from its plan."* (Does **not** stop injection in the *user's own prompt* — mitigate with
  context-minimization.) This is exactly why the planner/executor split is a *security* property, not
  just a latency one.
- **Action-Selector (guarded fallback)** — for the out-of-scope long tail, use **templated, parameterized
  SQL with placeholders and no action→output feedback loop**. The paper's SQL-agent case study is
  *"trivially immune… as the LLM never looks at any data directly."* Use this for your guarded fallback
  path instead of free-form generation wherever you can.

---

## 5. Evaluation harness — the blocker you must close first

The architecture doc already established *why* you need evals and the core method (datacompy/Ragas
result-set comparison, ~30 golden Qs/dashboard). This section adds the **net-new specifics** the build
research surfaced.

### 5a. Execution accuracy via result-set comparison — **[verified]**
Compare **returned DataFrames, not SQL strings** — *"the ultimate test of correctness."* Ragas wraps
**datacompy** (Capital One's OSS DataFrame-diff): `datacompy.Compare(..., on_index=True, abs_tol=1e-10,
rel_tol=1e-10)` with **row counts required to match**.

### 5b. Accept multiple correct queries per question — **[verified]**
Strict exact-match alone is inadequate: *"many business questions can be answered in multiple valid
ways"* and exact matching *"often rejects perfectly valid SQL."* Your golden set should store **multiple
human-curated correct queries per question**; a prediction passes if it matches *any* of them.

### 5c. The multi-signal scorecard (Uber's production eval) — **[verified]**
Don't score on one number. Uber grades QueryGPT on four signals — and they **map onto your agents**:

| Signal | Range | What it grades |
|---|---|---|
| **Table overlap** | 0–1 | The **table agent** — did it pick the right tables? |
| **Successful run** | bool | The query executes at all |
| **Run has output** | bool | Returns > 0 records |
| **Qualitative query similarity** | 0–1 | LLM-judge vs golden SQL — the **SQL generator** |

This decomposed scorecard tells you *which agent* regressed, not just that the answer got worse.

### 5d. LLM-as-judge — and prune its context — **[verified]**
- LLM-as-judge is a *"solid proxy / quick check"* — **F1 ~0.70–0.76** with GPT-4-Turbo (directional;
  current models likely differ).
- **Critical, non-obvious finding:** giving the judge the **full schema *degraded* it.** Restricting the
  judge to *"only the schema for tables referenced in the queries"* gave *"significant improvement in both
  false-positive and false-negative rates."* **Prune the judge's context to referenced-table schemas only.**

### 5e. Observability & CI regression — Langfuse — **[verified]**
Use **Langfuse** with its two relevant evaluation levels:
- **Experiment level** — offline, dataset-based runs on your golden set → **wire into CI**, run on every PR.
- **Observation level** — evaluators on individual production LLM calls/retrievals/tool calls
  (*"recommended for production"*, completes in seconds).
- Documented pattern: *"use Experiments during development to validate changes, then deploy
  Observation-level evaluators in production."*
- (LangSmith / Arize Phoenix / OpenLLMetry are viable alternatives; Langfuse is OSS/self-hostable, which
  fits an NGO-budget, no-vendor-lock-in posture. **[judgment]**)

---

## 6. Industry snapshot — concrete patterns to copy

| System | The pattern worth stealing |
|---|---|
| **Uber QueryGPT** *(verified)* | Agent decomposition (intent/table/column-prune); **HITL table approval**; intent-narrows-RAG-scope; the 4-signal eval scorecard. The single best reference for Dalgo. |
| **Snowflake Cortex Analyst** *(verified)* | Semantic model is the accuracy mechanism; RBAC + governance-boundary by default; the 90%→51% benchmark-gap warning. |
| **arXiv 2506.08837** *(verified)* | Plan-Then-Execute and Action-Selector as *security* patterns against prompt injection from untrusted query results. |
| **dbt Semantic Layer / MCP, ThoughtSpot Spotter, Databricks Genie** *(from architecture doc)* | Deterministic metric→SQL compilation; intermediate representation over raw SQL; route to trusted assets. See the architecture doc — not re-verified this round. |
| **WrenAI / Vanna / Dataherald** | OSS GenBI/text-to-SQL stacks; relevant if you ever want an off-the-shelf semantic+agent layer instead of your own enricher. Not adversarially verified here. |

---

## 7. Step-by-step build sequence (do not reorder 1–3)

1. **Golden dataset + execution-accuracy evals** — ~30 Qs/dashboard from real partner NGOs, **multiple
   valid SQL per question**, datacompy result-set comparison, the 4-signal scorecard, judge with
   **pruned context**. *Without this, every step below is a blind guess.* (§5)
2. **Formalize the semantic layer** over your dbt models — promote the enricher artifact into a real
   semantic model (metrics, synonyms, joins, **column PII taxonomy**), in-memory hybrid retrieval. (§2)
3. **Planner/executor as a small agent chain** — intent → **table-select with HITL approval** →
   column-prune → execute → validate. LangGraph/Pydantic AI over the existing WebSocket. (§1, §3)
4. **Governance pipeline** — LiteLLM gateway + Presidio `pre_call` masking; **column-level result-set
   masking + RBAC at the connection**; Plan-Then-Execute control-flow integrity; Action-Selector guarded
   fallback for out-of-scope. (§4)
5. **Observability in CI** — Langfuse experiments on every PR, observation-level evaluators in prod;
   persist per-message latency/cost/tokens/warehouse-calls. (§5e)
6. **Cost/latency hardening** — cheap model for intent/classify, capable Claude model for generation;
   prompt + result caching; column-prune to cut tokens; parallel execution where the plan allows.
   *(Targets/budgets are unmeasured — instrument first, then set thresholds. §8)*

---

## 8. Open questions the research did NOT resolve (your calls to measure)

These genuinely lack verified evidence — flagged honestly rather than guessed:

1. **Concrete latency/cost budgets and caching strategy** at NGO scale — no published per-query figures.
   Instrument with Langfuse, then set thresholds from real traces.
2. **MetricFlow vs custom artifact** — confirmed you need a semantic layer; the *implementing tool* for a
   dbt/Postgres/Superset stack is a judgment call to validate against your modeled-metric coverage.
3. **Exact HITL UX** — how heavyweight the table-approval step should be (always-on vs only-on-low-
   confidence) to balance accuracy against friction.

---

## Sources (primary, verified this round)

- Uber QueryGPT: https://www.uber.com/us/en/blog/query-gpt/
- Snowflake Cortex Analyst (docs): https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst
- Snowflake Cortex Analyst accuracy (eng blog): https://www.snowflake.com/en/blog/engineering/cortex-analyst-text-to-sql-accuracy-bi/
- Agent prompt-injection design patterns (arXiv 2506.08837): https://arxiv.org/pdf/2506.08837 · summary: https://simonwillison.net/2025/Jun/13/prompt-injection-design-patterns/
- Microsoft Presidio: https://github.com/microsoft/presidio
- LiteLLM Presidio PII guardrail: https://docs.litellm.ai/docs/proxy/guardrails/pii_masking_v2
- Ragas text-to-SQL evaluation: https://docs.ragas.io/en/stable/howtos/applications/text2sql/
- Arize — LLM-as-a-judge for text-to-SQL: https://arize.com/blog/text-to-sql-evaluating-sql-generation-with-llm-as-a-judge/
- Langfuse LLM-as-a-judge: https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge

> **Two refuted claims excluded** (failed verification): QueryGPT's specific RAG counts (3 tables / 7 SQL
> samples / kNN) — the *mechanism* (intent narrows RAG scope) holds, the *numbers* did not; and that a
> semantic model *"guarantees"* correct business logic — it's necessary, not sufficient.
