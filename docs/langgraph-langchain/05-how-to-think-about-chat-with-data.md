# How to Think About Building Chat-with-Data

*A mental-models + industry-landscape doc for the Dalgo team (Python data platform, Postgres warehouse, dbt transforms), based on what companies actually built and published, as of mid-2026.*

---

## 1. TL;DR

- **The hard problem is not SQL generation — it's trust and context.** Frontier LLMs write syntactically fine SQL. What they can't do without help is know *your* definition of "active beneficiary," which of your 400 tables is the certified one, or that your fiscal year starts in April. The dominant failure mode is a **plausible wrong answer**, which is worse than an error.
- **Benchmarks lie by omission.** Models score 90%+ on Spider 1.0 but under ~30% on Spider 2.0, which uses real enterprise schemas and workflows. On BIRD, humans hit ~93% while leading systems sit around ~73% — and BIRD's own scoring only agrees with human experts 62% of the time. Toy-benchmark accuracy does not transfer.
- **Every serious production system converged on the same three moves:** (1) curate context instead of dumping schemas (Uber's workspaces, Databricks Genie spaces, LinkedIn's certified datasets); (2) decompose into small single-purpose agents/steps rather than one big prompt (Uber, LinkedIn); (3) build an eval harness with golden question→SQL sets before scaling (Uber, Snowflake, Databricks Genie Benchmarks).
- **The 2025–2026 shift is semantic-layer-first.** Snowflake Cortex Analyst, dbt's Semantic Layer + MCP server, WrenAI, and ThoughtSpot Spotter all moved query correctness out of the LLM: the model picks *metrics and dimensions*; a deterministic engine compiles the SQL. dbt's 2026 benchmark measured a jump from 10.8% to 76.5% execution accuracy from adding a semantic layer.
- **For a Dalgo-sized team:** semantic-layer-first over your dbt models, a fixed pipeline (not a free-roaming agent), bounded self-correction (one or two structured retries, not loops), always-visible SQL/provenance, and a golden query set from day one. Raw text-to-SQL becomes a clearly-labeled fallback for exploratory questions, never the default path for KPIs.

---

## 2. The core problem: trust and context, not SQL

It's tempting to frame chat-with-data as "text-to-SQL," a translation task. Every team that shipped one discovered the translation is the easy 20%. Three harder problems dominate:

**Schema semantics.** A warehouse schema tells you column names and types. It does not tell you that `status = 3` means "churned," that `users_v2` superseded `users`, or that revenue lives in `fct_payments` but *refund-adjusted* revenue requires joining `fct_refunds`. Pinterest measured this directly: retrieval hit rate for finding the right table was **40% without table documentation in the embeddings, rising linearly to 90%** as documentation weight increased ([Pinterest](https://medium.com/pinterest-engineering/how-we-built-text-to-sql-at-pinterest-30bad30dabff)). The model isn't the bottleneck; your metadata is.

**Business definitions.** "Active user," "revenue," "margin" mean different things to different teams, and nothing in the warehouse disambiguates them ([Omni: Why text-to-SQL fails](https://omni.co/blog/why-text-to-sql-fails)). For Dalgo's NGO users this is acute: "beneficiaries reached this quarter" might mean unique registrations, service interactions, or a funder-specific definition — and the person asking often doesn't know there's a difference.

**Silent wrong answers.** A compiler error is annoying; a query that runs and returns a confidently wrong number is dangerous. As Collate puts it, "failure looks like a plausible but incorrect answer" — a failure you discover in the board meeting, not in the terminal ([Collate](https://www.getcollate.io/blog/your-text-to-sql-problem-is-not-the-llm)). Databricks' own docs concede the problem: Genie returns a results table, users *can* inspect the generated SQL, "but non-technical users might not have the background to interpret the SQL statement or assess the correctness of the answer" ([Databricks](https://docs.databricks.com/aws/en/genie/)).

**The mental model:** you are not building a translator; you are building a *trust boundary*. Everything in the architecture — semantic layers, table approval steps, validation agents, evals — exists to move errors from *silent* to *visible*, and from *runtime* to *build time*.

### Why the benchmarks fooled everyone

- **Spider 1.0 → Spider 2.0.** Models exceed 91% execution accuracy on Spider 1.0's small, well-normalized academic schemas. Spider 2.0 (ICLR 2025 oral) rebuilt the task from 632 real enterprise workflows — thousand-column schemas, BigQuery/Snowflake dialects, multi-step transformations. o1-preview scored ~17–21%; the best published framework (ReFoRCE + o1-preview) reached ~30% ([Spider 2.0 paper](https://arxiv.org/pdf/2411.07763), [GitHub](https://github.com/xlang-ai/Spider2)). A 60-point drop from changing nothing but *realism*.
- **BIRD.** Human data engineers score 92.96% execution accuracy; ChatGPT launched at ~40%, and leading systems have climbed to roughly ~73% — still a wide gap ([BIRD analysis](https://medium.com/@adnanmasood/pushing-towards-human-level-text-to-sql-an-analysis-of-top-systems-on-bird-benchmark-666efd211a2d)). Worse, a 2025 study found BIRD's binary execution-accuracy metric agrees with human expert judgment only **62% of the time**, mostly false negatives, and a CIDR 2026 paper found annotation errors large enough to shift leaderboard ranks by up to three positions ([Text-to-SQL Benchmarks are Broken](https://www.vldb.org/cidrdb/papers/2026/p5-jin.pdf)).
- **The transfer failure has a specific cause.** Toy benchmarks pre-solve the two hardest sub-problems — schema linking (few tables, clean names) and semantics (unambiguous questions). Pinterest said this outright: benchmark datasets "using small numbers of well-normalized tables" don't reflect a warehouse with hundreds of thousands of tables. MotherDuck's read of BIRD is the sharpest summary: the systems that win are the ones that effectively rebuild a data model — "your data model *is* the semantic layer" ([MotherDuck](https://motherduck.com/blog/bird-bench-and-data-models/)).

**Implication for Dalgo:** any vendor or paper accuracy number measured on Spider/BIRD tells you almost nothing about accuracy on an NGO's messy Airbyte-ingested Postgres schema. The only number that matters is the one from *your own* golden query set (Section 6).

---

## 3. The design axes every team must choose on

Three architectural decisions, mostly independent, define the design space. Every system in Section 4 is a point in this space.

### Axis 1: Where does SQL correctness come from?

| Approach | How it works | Correctness guarantee | Coverage |
|---|---|---|---|
| **Direct text-to-SQL** | LLM writes SQL from schema + examples | None — every query is a fresh inference | Unlimited (any question) |
| **Semantic-layer-first** | LLM maps question → metrics/dimensions; deterministic engine compiles SQL | Joins/aggregations guaranteed correct *within model coverage* | Bounded by what's modeled |
| **Agent-with-tools** | LLM iterates with tools: inspect schema, run query, check results | Empirical — the agent can verify by executing | Unlimited, but unbounded latency/cost/behavior |

The trade is **coverage vs. trust**. Direct text-to-SQL answers anything and guarantees nothing. A semantic layer answers only what's modeled but, as the dbt framing goes, "when it can answer, it gets it right; when it can't, it tells you" — it never silently returns wrong data ([dbt 2026 benchmark](https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026)). Production systems in 2026 are hybrids: semantic layer for governed metrics, guarded text-to-SQL for the long tail.

### Axis 2: Fixed pipeline vs. autonomous agent

A **fixed pipeline** (intent → table selection → generation → validation → execution) is debuggable, evaluable per-stage, and has predictable cost/latency. An **autonomous agent** (ReAct-style loop with tools) handles the long tail better but is harder to eval, can wander, and multiplies latency. Uber and LinkedIn both shipped pipelines with LLMs at the stages ("multi-agent" in their language means *decomposed pipeline*, not free-roaming autonomy). Uber's headline learning: "specialized agents performing single units of work outperformed broad generalized tasks significantly" ([Uber](https://www.uber.com/blog/query-gpt/)). The autonomy that *does* pay off is narrow: bounded self-correction (retry on execution error with the error message) and Spotter-style "check your own work" steps — not open-ended planning.

### Axis 3: Context strategy

How does the right schema/business knowledge get in front of the model?

1. **RAG over schema metadata** (Pinterest, Uber v1, Vanna): embed table summaries, docs, and historical queries; retrieve at question time. Scales to huge warehouses; quality is a direct function of metadata quality.
2. **Curated scoping** (Uber workspaces, Databricks Genie spaces, LinkedIn certified datasets): a human pre-selects the tables, example queries, and instructions for a domain. Dramatically shrinks the search space; costs curation effort.
3. **Semantic models / pre-computed definitions** (Cortex Analyst, dbt MetricFlow, WrenAI MDL): business logic encoded once, in code, reviewed like code. Highest trust, highest upfront cost.

These stack. Uber does 1+2; Snowflake recommends 3 with 1 layered on for value lookups (Cortex Analyst + Cortex Search). For Dalgo, **you already have most of layer 3 for free**: dbt models, schema.yml descriptions, and the transform DAG are exactly the curated metadata everyone else had to build from scratch.

---

## 4. What each notable company did

### Uber — QueryGPT ([engineering blog](https://www.uber.com/blog/query-gpt/))

Uber serves ~1.2M interactive queries/month platform-wide; QueryGPT targets the authors of those queries. **V1** (a hackathon build) was naive RAG: embed the question, similarity-search over 7 tables and 20 SQL samples. It degraded immediately as tables were added — similarity search couldn't match natural language to schemas, large schemas (200+ columns) blew 40–60K tokens, and vague questions made table selection unreliable. **20+ iterations later**, the production design is:

- **Workspaces**: 12 curated domain collections (Mobility, Ads, Core Services…) of tables + SQL samples, plus user-defined custom workspaces.
- **Intent agent**: classifies the question into a workspace, shrinking the RAG search radius.
- **Table agent**: proposes tables and **shows them to the user for approval/editing** before generation — human-in-the-loop as an architectural component, not an afterthought.
- **Column prune agent**: an LLM strips irrelevant columns from big schemas before generation, cutting tokens, latency, and cost.
- Outcomes: query authoring ~3 min vs ~10 min manual; 300 DAU in a limited release; 78% of users reported time savings. Hallucination "remains an ongoing challenge" — their proposed recursive validation agent was never fully shipped.

**Key lesson:** decompose into small classifiers, curate context into workspaces, and put a human checkpoint at the highest-leverage spot (table selection). Also: even Uber, with dedicated staff, calls hallucination unsolved.

### Pinterest — Text-to-SQL in Querybook ([engineering blog](https://medium.com/pinterest-engineering/how-we-built-text-to-sql-at-pinterest-30bad30dabff))

Built into their open-source query IDE. V1: user picks tables, LLM writes SQL, streamed over WebSocket. The real bottleneck turned out to be upstream — "identifying the correct tables amongst the hundreds of thousands in our data warehouse." V2 added RAG: offline vector index of table summaries and historical queries; top-K tables re-ranked by an LLM and **validated by the user** before generation. Findings: 35% faster SQL authoring in real usage (vs. 50%+ claimed in controlled studies); first-shot acceptance rose from 20% to 40%+ as users learned to prompt; and the metadata-quality result quoted in Section 2 (40% → 90% hit rate with docs).

**Key lesson:** text-to-SQL performance is downstream of data governance. The 90-day roadmap item that most improves your chatbot may be writing table descriptions, not touching the model.

### LinkedIn — SQL Bot ([engineering blog](https://www.linkedin.com/blog/engineering/ai/practical-text-to-sql-for-data-analytics), [arXiv paper](https://arxiv.org/html/2507.14372v1))

A multi-agent system inside their DARWIN data platform, built on LangChain/LangGraph. Retrieval funnels aggressively: a filtering agent narrows candidates to ~20 tables, a ranking agent to ~7, another agent selects fields, then the writer generates a plan and builds the query incrementally. Context is personalized — table popularity *within the asker's org* informs ranking — and grounded in **certified datasets** plus a knowledge base harvested from existing docs and query logs. Queries are validated and self-corrected before being shown. Result: hundreds of users across verticals; ~95% rated query accuracy "Passes" or above, ~40% "Very Good/Excellent."

**Key lesson:** retrieval is a funnel, not a lookup — successive cheap narrowing steps beat one retrieval call. And "which tables does *this user's team* actually use" is a powerful, cheap signal.

### Snowflake — Cortex Analyst ([docs](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst), [accuracy blog](https://www.snowflake.com/en/blog/engineering/cortex-analyst-text-to-sql-accuracy-bi/))

The flagship semantic-model approach. You write a semantic model (logical tables, dimensions, facts, metrics, verified queries, synonyms); an agentic system answers questions *through* it, applying user-defined measures and filters consistently. Snowflake claims 90%+ SQL accuracy on real-world use cases — nearly 2x single-prompt GPT-4o — measured on an internal 150-question benchmark tiered into filtering/aggregation/trends; the semantic model alone contributed ~20 points of accuracy across four test datasets. They pair it with Cortex Search for literal-value retrieval (e.g., mapping "NYC" to `'New York City'` in the data).

**Key lesson:** the semantic model is where the accuracy is. Also note what a good internal benchmark looks like: small (150), tiered by difficulty, mirroring real BI tasks.

### Databricks — AI/BI Genie ([blog](https://www.databricks.com/blog/aibi-genie-now-generally-available), [trusted assets docs](https://docs.databricks.com/en/genie/trusted-assets.html))

Domain-scoped **Genie spaces**: an analyst curates the datasets (via Unity Catalog), text instructions, example SQL, and SQL functions for one domain. Two ideas worth stealing: **trusted assets** — parameterized queries/UDFs that answer expected recurring questions *exactly*, bypassing generation entirely, so "how much revenue last month" always runs the blessed query; and **Genie Benchmarks** — curated test questions with expected SQL, built into the product so space authors can regression-test their curation. Plus "Ask for Review," an end-user flag that routes dubious answers to the space admin.

**Key lesson:** for known recurring questions, don't generate — dispatch to a verified parameterized query. Generation is for the long tail. And ship the feedback loop (user-flags-answer → curator-fixes-space) as a first-class feature.

### Open source: Vanna.ai, WrenAI, dbt

- **[Vanna](https://github.com/vanna-ai/vanna)** — a Python RAG framework: you "train" it on DDL, documentation, and question→SQL pairs stored in a vector DB; it retrieves them per-question to build the prompt. Its best idea is the **self-learning loop**: user-confirmed correct query pairs get added back to the training store, so accuracy compounds with usage. Cheap to stand up; correctness is still per-query inference with no guarantees.
- **[WrenAI](https://www.getwren.ai/post/wren-ai-vs-vanna-the-enterprise-guide-to-choosing-a-text-to-sql-solution)** — semantic-layer-first OSS: an explicit modeling layer (MDL) defining entities, relationships, and business terms; the Wren engine parses generated SQL into a logical plan, applies semantic rules, dry-run-validates, and transpiles to the target dialect. Governance-oriented where Vanna is retrieval-oriented ([comparison](https://sudiptapathak.com/blog/dissecting-open-source-nl2sql/)).
- **[dbt Semantic Layer / MetricFlow](https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl)** — metrics defined in code next to your dbt models; MetricFlow compiles metric+dimension requests into SQL **deterministically** — the LLM can't produce a wrong join because the LLM never writes the join. The [dbt MCP server](https://docs.getdbt.com/docs/dbt-ai/about-mcp) exposes models, metrics, lineage, and the semantic layer to any MCP client, which is the emerging standard shape for "let an agent query the warehouse safely." dbt's [2026 benchmark](https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026) reports execution accuracy lifting from **10.8% to 76.5%** on their most enterprise-like dataset when the semantic layer answers instead of raw text-to-SQL.

**Key lesson:** the OSS ecosystem split along Axis 1 — Vanna bet on retrieval + learning loop, WrenAI/dbt bet on semantic governance — and by 2026 the governance side is winning the trust argument. For Dalgo, dbt-native is the obvious fit: your users' transform layer is already dbt.

### Brief: ThoughtSpot Spotter, Amazon Q in QuickSight

- **[ThoughtSpot Spotter](https://www.thoughtspot.com/blog/introducing-spotter-ai-analyst)**: an "agentic analyst" that plans, checks its own work, and refines — but grounded in ThoughtSpot's long-standing search-token architecture and an "agentic semantic layer" for explainability. Even the most agent-forward vendor anchors the agent to a semantic layer.
- **[Amazon Q in QuickSight](https://aws.amazon.com/blogs/aws/amazon-quicksight-q-business-intelligence-using-natural-language-questions/)**: NL Q&A over curated "topics" (author-prepared, synonym-enriched datasets), extended with generative dashboards and multi-step "Scenarios" analysis ([SiliconANGLE](https://siliconangle.com/2025/03/25/aws-enhances-quicksights-business-intelligence-ai-agents/)). Same pattern: curation layer first, NL on top.

### The convergent pattern

Independently, everyone landed on: **scope the domain (workspace/space/topic/semantic model) → retrieve within it → generate with a small decomposed pipeline → validate → show provenance → measure with golden questions.** No published production system is "prompt GPT with the schema." That should end any internal debate about whether the naive version will be fine.

---

## 5. Failure modes and mitigations

Think of these as a checklist; each has a known mitigation with a real-world precedent.

| Failure mode | What it looks like | Mitigations |
|---|---|---|
| **Silent semantic error** | Query runs, number is wrong: wrong metric definition, missed filter (`is_deleted`, test orgs), wrong grain, double-counting across a fan-out join | Semantic layer for governed metrics (dbt/Cortex); trusted assets for recurring questions (Genie); show the SQL + tables used + row counts so a human *can* verify; sanity checks (nulls, suspicious zeros, magnitude vs. historical) |
| **Hallucinated tables/columns/joins** | References `orders.customer_name` which doesn't exist; invents a join path | Validate generated SQL against the live catalog before execution (WrenAI's dry-run; `EXPLAIN` on Postgres is nearly free); constrain generation to retrieved schemas only; Uber's never-shipped "validation agent" is table stakes for you |
| **Ambiguous question** | "How many active users?" — active by whose definition? Which time window? | Detect-then-clarify: classify ambiguity *before* generating and ask one targeted question; LinkedIn/Uber-style user confirmation of selected tables; Uber's prompt-enhancer for terse, typo-ridden questions |
| **Large-schema context overflow** | 200-column tables consume 40–60K tokens; retrieval drowns in near-duplicate staging tables | Uber's column-prune agent; LinkedIn's 20→7 funnel; exclude staging/intermediate dbt models from the search space entirely — only expose marts |
| **Wrong table chosen among near-duplicates** | `users`, `users_v2`, `stg_users`, `dim_users` | Certification/curation (LinkedIn certified datasets, Genie space allowlists); usage-based ranking from query logs; in Dalgo, the dbt DAG tells you which model is terminal |
| **Runaway self-correction** | Agent retries in a loop, burning tokens, converging on a *different wrong* answer | Bounded, structured retries only: feed the execution error back **once or twice** with the specific failure; naive "try again" retries measurably hurt (they add cost and often flip correct→incorrect); after N failures, decline and route to a human |
| **Unauthorized access / PII leakage** | Right answer to the wrong person; PII in the result set surfaced into an LLM context | Execute as the *user's* role, not a service superuser (Postgres RLS / per-org schemas in Dalgo); treat result sets, not just prompts, as a PII surface; read-only connection, statement timeout, row limits |

Two meta-points. First, **the mitigation for the worst failure (silent semantic error) is architectural, not prompt-level** — no amount of prompt engineering makes an LLM know your definition of "reached beneficiary"; only a semantic layer or a verified query does. Second, **clarification is a feature, not a failure.** Systems that ask one good question before answering outperform systems that guess; Uber and Pinterest both added explicit user-confirmation steps *after* launch because guessing eroded trust.

---

## 6. Evals: the real moat

Every strong system in Section 4 has one, and it's the least copyable part. Models are commodities; your golden query set encodes your schema, your definitions, and your users' actual questions.

**Uber's harness is the reference design** ([blog](https://www.uber.com/blog/query-gpt/)):
- A manually curated set of **golden question → SQL** pairs.
- **Two evaluation flows**: *vanilla* (end-to-end, from raw question) and *decoupled* (feed ground-truth intent/tables into each stage to measure that stage alone). Decoupled evals are what let you localize a regression to intent classification vs. table selection vs. generation.
- **Multiple signals per question**: intent accuracy; table-overlap score (0–1); does the query execute; does it return records; LLM-judged similarity to the golden SQL (0–1). Tracked over time to catch regressions.

**Snowflake and Databricks confirm the pattern at different scales**: Snowflake's internal suite is just 150 questions, tiered by difficulty (filtering / aggregation / trends); Databricks productized it as Genie Benchmarks so each space author maintains their own test set. You don't need thousands of cases — you need 50–150 *representative* ones with verified answers, run on every prompt/model/retrieval change.

**What "correct" means (harder than it sounds):**
- *Execution match* (compare result sets, not SQL strings) is the strongest signal but has false negatives — column order, equivalent groupings, ties. The FLEX study found BIRD's binary execution accuracy agrees with human experts only **62%** of the time ([benchmark-errors paper](https://www.vldb.org/cidrdb/papers/2026/p5-jin.pdf)). Use it, but expect to adjudicate disagreements manually.
- *LLM-as-judge* (Uber's similarity score) is useful for triage and trend lines, not as ground truth — judges are lenient on plausible-looking SQL, which is precisely the failure mode you care about. Never let a judge score gate a release by itself.
- *Stage-level metrics* (was the right table retrieved? right metric picked?) are cheap, deterministic, and diagnose *why* end-to-end accuracy moved. Prioritize these.

**Practical recipe for Dalgo:** harvest golden questions from real support asks and existing dashboard queries (each chart on a Dalgo dashboard is a verified question→SQL pair — that's a pre-built golden set); verify answers with a human analyst; store as data (question, expected SQL, expected result hash, tables, tags); wire into CI so a prompt tweak that breaks 6 questions fails visibly. Then close the loop in production: log every question, generated SQL, execution outcome, and user thumbs-up/down; promote confirmed-good pairs into few-shot examples (Vanna's self-learning loop) and confirmed-bad ones into the eval set.

If you build one thing before launch, build this. Without it, every model upgrade, prompt edit, and retrieval change is a coin flip you can't see.

---

## 7. Synthesis: a sensible architecture for a mid-sized team

Dalgo's constraints: a small Python team, multi-tenant Postgres warehouses, dbt transforms per org, non-technical NGO users who cannot audit SQL, and reported numbers that go to funders — i.e., the highest possible cost for silent wrong answers, and the least capacity to run a five-person "workspace curation" team. That points to one design:

**Semantic-layer-first, fixed pipeline, bounded self-correction, human-verifiable output.**

```
question
  → scope & intent (which org / domain; is this a metric question, an
    exploration, or out-of-scope?)                        [LLM classifier]
  → ambiguity check → at most ONE clarifying question      [LLM + rules]
  → route:
      (a) matches a defined metric  → MetricFlow / semantic layer
          compiles SQL deterministically                   [no LLM in SQL]
      (b) matches a trusted asset   → run the verified
          parameterized query                              [no LLM at all]
      (c) long tail                 → guarded text-to-SQL over
          RETRIEVED, MART-ONLY schema context              [LLM writes SQL]
  → validate: parse, EXPLAIN against live catalog, read-only role,
    row limit, statement timeout
  → execute; on error, ONE structured retry with the error message;
    then decline gracefully
  → answer with provenance: the number + the SQL + tables/metrics used
    + row count + "verified metric" vs "generated query" badge
```

**Why these choices, given everything above:**
- *Semantic-layer-first* because it's the only mitigation for silent semantic errors (Sections 2, 5), it produced the largest measured accuracy gains anywhere in the literature (Snowflake +20pts, dbt 10.8%→76.5%), and Dalgo uniquely already sits on the ingredients: dbt projects with documented models. Defining 10–20 MetricFlow metrics per org template is incremental work, not a new platform.
- *Fixed pipeline over autonomous agent* because pipelines are stage-evaluable (Uber's decoupled flow requires one), predictable in cost, and every production system that published results shipped one. Save agentic freedom for an internal analyst-facing mode later, not the NGO-facing default.
- *Bounded self-correction* because one structured retry on execution errors captures most of the benefit (LinkedIn validates and self-corrects pre-display) while unbounded loops burn money and can flip right answers to wrong ones.
- *Human-verifiable output* because Genie's docs admit the gap honestly: non-technical users can't audit SQL. So don't ask them to — instead badge answers by trust tier ("verified metric" vs "AI-generated query, please verify"), show which tables/metrics were used in plain language, and give them Databricks-style *Ask for Review* to flag anything off to their Dalgo admin.

**Build order:**

*First (weeks 1–6):* golden query set (~50 questions from real user asks + existing dashboard queries) and the eval harness; read-only execution path with role-scoped access, timeouts, row limits; route (c) only — guarded text-to-SQL over dbt-mart schemas with schema.yml descriptions in the retrieval index — because it exercises the whole pipeline and the evals will tell you exactly how bad it is (expect Spider-2.0-like numbers, not demo numbers).

*Second (weeks 6–12):* the trust tiers — MetricFlow metric definitions for the top KPIs per org template (route a) and trusted parameterized queries for the top recurring questions (route b); ambiguity detection + single clarifying question; provenance badges in the UI.

*Later:* self-learning loop (confirmed-good pairs → few-shot store); dbt MCP server integration so the same semantic access works from Claude/other agents, not just your UI; per-org custom metrics authored by partner analysts; charts/visualization on top of verified results.

*Explicitly not now:* fine-tuning (no eval basis to justify it yet), autonomous multi-step agents, cross-org analytics, voice/multilingual — each is additive later and a distraction first.

The one-sentence summary of the whole landscape: **the LLM should choose *what* to compute; something deterministic should decide *how*; and an eval harness should tell you, continuously, whether the seam between them is holding.**

---

## 8. Sources

**Company engineering blogs**
- Uber QueryGPT: https://www.uber.com/blog/query-gpt/
- Pinterest text-to-SQL: https://medium.com/pinterest-engineering/how-we-built-text-to-sql-at-pinterest-30bad30dabff
- LinkedIn SQL Bot: https://www.linkedin.com/blog/engineering/ai/practical-text-to-sql-for-data-analytics — and the companion paper: https://arxiv.org/html/2507.14372v1
- Snowflake Cortex Analyst docs: https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst — accuracy engineering blog: https://www.snowflake.com/en/blog/engineering/cortex-analyst-text-to-sql-accuracy-bi/ — Cortex Analyst + Cortex Search: https://www.snowflake.com/en/engineering-blog/cortex-analyst-cortex-search-integration/
- Databricks AI/BI Genie GA: https://www.databricks.com/blog/aibi-genie-now-generally-available — trusted assets: https://docs.databricks.com/en/genie/trusted-assets.html — Genie spaces & benchmarks: https://docs.databricks.com/aws/en/genie/

**Open source**
- Vanna: https://github.com/vanna-ai/vanna
- WrenAI vs Vanna (WrenAI's own comparison): https://www.getwren.ai/post/wren-ai-vs-vanna-the-enterprise-guide-to-choosing-a-text-to-sql-solution — independent OSS teardown: https://sudiptapathak.com/blog/dissecting-open-source-nl2sql/
- dbt Semantic Layer / MetricFlow: https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl — dbt MCP server: https://docs.getdbt.com/docs/dbt-ai/about-mcp — semantic layer vs text-to-SQL 2026 benchmark: https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026

**BI vendors (brief)**
- ThoughtSpot Spotter: https://www.thoughtspot.com/blog/introducing-spotter-ai-analyst — agents page: https://www.thoughtspot.com/product/agents
- Amazon Q in QuickSight: https://aws.amazon.com/blogs/aws/amazon-quicksight-q-business-intelligence-using-natural-language-questions/ — 2025 agent updates: https://siliconangle.com/2025/03/25/aws-enhances-quicksights-business-intelligence-ai-agents/

**Benchmarks & research**
- Spider 2.0 (ICLR 2025 oral): https://arxiv.org/pdf/2411.07763 — leaderboard/code: https://github.com/xlang-ai/Spider2
- BIRD top-systems analysis: https://medium.com/@adnanmasood/pushing-towards-human-level-text-to-sql-an-analysis-of-top-systems-on-bird-benchmark-666efd211a2d
- "Text-to-SQL Benchmarks are Broken" (annotation errors, FLEX agreement study), CIDR 2026: https://www.vldb.org/cidrdb/papers/2026/p5-jin.pdf
- MotherDuck, "Your Data Model Is the Semantic Layer": https://motherduck.com/blog/bird-bench-and-data-models/

**Trust & failure-mode commentary**
- Omni, "Why text-to-SQL fails": https://omni.co/blog/why-text-to-sql-fails
- Collate, "The right answer to the wrong question": https://www.getcollate.io/blog/your-text-to-sql-problem-is-not-the-llm
- Cleverbridge, "What makes agentic analytics trustworthy": https://grow.cleverbridge.com/blog/trustworthy-agentic-analytics-raw-database-risks
