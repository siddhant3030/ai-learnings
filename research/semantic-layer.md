# Semantic Layers for Data: How People Actually Build Them

> Deep-research report, 2026-07-14. Four parallel research agents → ~40 primary sources
> (vendor docs, engineering blogs, arXiv, GitHub specs), findings cross-checked across
> agents; numbers verified against primary sources where possible, unverifiable items
> flagged inline and in §9. Audience: the Dalgo chat-with-data team, deciding how to
> build the v3 semantic layer (`dalgo-core/features/chat-with-data/v3/plan.md`).
> Companions: [`langchain-langgraph-chat-with-data.md`](./langchain-langgraph-chat-with-data.md),
> [`posthog-agent-architecture.md`](./posthog-agent-architecture.md).

---

## Table of Contents

1. [The one-paragraph thesis](#1-the-one-paragraph-thesis)
2. [The spectrum: four architectures](#2-the-spectrum-four-architectures)
3. [The consensus card: what every format agrees on](#3-the-consensus-card-what-every-format-agrees-on)
4. [What moves accuracy, ranked by evidence](#4-what-moves-accuracy-ranked-by-evidence)
5. [The authoring reality: who writes this stuff](#5-the-authoring-reality-who-writes-this-stuff)
6. [Value profiling mechanics: the concrete numbers](#6-value-profiling-mechanics-the-concrete-numbers)
7. [Staleness: everyone's second problem](#7-staleness-everyones-second-problem)
8. [Schema linking at small scale: don't](#8-schema-linking-at-small-scale-dont)
9. [What we could not verify](#9-what-we-could-not-verify)
10. [What this means for Dalgo v3](#10-what-this-means-for-dalgo-v3)

---

## 1. The one-paragraph thesis

Every serious system converges on the same authoring shape: **automatic harvest**
(schemas, query logs, lineage, even pipeline code) → **LLM enrichment** (descriptions,
suggested examples) → **a thin but mandatory human layer** (certification, verified SQL,
business caveats) → **a promotion loop** that turns real usage into new curated
artifacts. The un-curated baseline is measurably unusable — LinkedIn's schema-only
ablation scored **9%**, Anthropic without their curated layer **21%** — and the two
highest-leverage artifacts are the same everywhere: **example/verified queries** and
**value-decoding knowledge** (what the codes and labels in columns actually mean).
Meanwhile the *heavyweight* end (standalone metric stores) has a commercial graveyard
behind it, and at small-warehouse scale the research says don't bother with retrieval
sophistication at all — ship the whole schema, enriched.

---

## 2. The spectrum: four architectures

Production systems sit on a spectrum from "compile, don't generate" to "no artifact at
all." Position on the spectrum tracks (a) how wide the domain is and (b) who's available
to curate.

| Architecture | Who | The artifact | SQL comes from |
|---|---|---|---|
| **Full semantic layer (compile)** | Anthropic internal, dbt MCP, Cube D3, Looker Conversational Analytics, Malloy Publisher | metric/dimension definitions in versioned files | **compiled deterministically** — the LLM picks named metrics, never writes table SQL |
| **Curated semantic model for an LLM** | Snowflake Cortex Analyst, Databricks Genie | YAML/space per domain: descriptions, synonyms, sample values, verified queries | LLM-generated, grounded in the model |
| **Auto-enriched schema context** | LinkedIn SQL Bot, AWS RRDA, OpenAI in-house agent, Pinterest | knowledge graph / table cards built by sync jobs + LLM passes | LLM-generated, grounded in retrieved cards |
| **Live taxonomy tools (no artifact)** | PostHog Max | none — a `read_taxonomy` tool reads definitions/values at question time | LLM-generated, checks reality via tools |

Two boundary observations:

- **Compile-mode wins on accuracy where it applies** — dbt's (vendor-run, 11-question)
  benchmark: semantic layer 98.2–100% vs 84–90% text-to-SQL; AtScale claims 92.5% vs
  ~20% on TPC-DS questions; Google claims "two-thirds fewer errors" with LookML
  grounding. The stronger argument is qualitative: *"With text-to-SQL, failure looks
  like a plausible but incorrect answer. With the Semantic Layer, failure looks like an
  error message"* ([dbt](https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026)).
  But the same dbt benchmark shows text-to-SQL nearly doubled since 2023 (32.7%→64.5%
  raw) — the gap is largest on complex metric questions and shrinking on simple ones.
- **PostHog's zero-artifact approach is domain-dependent** — event analytics has one
  known schema shape, which is exactly why "read the live taxonomy" works without
  curation. It does not generalize to arbitrary NGO warehouses.

Anthropic's internal system is the fullest statement of the compile position: the
semantic layer is "the mandatory default path for every data question," raw SQL a
guarded fallback, and the results are 21% → **95–99%**
([claude.com blog](https://claude.com/blog/how-anthropic-enables-self-service-data-analytics-with-claude)).
Their sharpest negative result: **raw RAG over thousands of past queries moved accuracy
by less than a point** — history must be distilled into structured reference docs, not
dumped into a vector store.

---

## 3. The consensus card: what every format agrees on

Extracted from the actual specs (Cortex Analyst semantic view YAML, WrenAI MDL, dbt
semantic manifest, Cube data model, Vanna's training corpus, DataHub/OpenMetadata):

**The non-negotiable skeleton — in every structured format:**

1. Table identity + physical binding (logical name → schema.table or defining SQL)
2. Column list with data types (+ SQL expression when logical ≠ physical)
3. **Descriptions at table and column level** — the only universal *semantic* field
4. **Primary key / grain** — every join-capable format makes PK load-bearing
5. **Relationships** — join partner + cardinality + condition (or entity keys to infer it)

Plus typed measures/aggregations in all the BI-lineage formats.

**The LLM-native extras** — fields that exist only because a model is the consumer:

| Extra | Who has it | Why it matters |
|---|---|---|
| `synonyms` per column/metric | Cortex (typed field), Genie knowledge store | maps user vocabulary ("silt dug") to schema vocabulary (`silt_achieved`) |
| `sample_values` | Cortex, Genie, LinkedIn (top-K) | correct WHERE literals. **Snowflake exempts sample values from its ~31K-token model cap** — a strong signal of where they think accuracy lives |
| `is_enum` (exhaustive value list) | Cortex | turns samples into a closed domain the model must filter within |
| **Verified question–SQL pairs** | Cortex `verified_queries`, Genie trusted assets, LinkedIn certified queries, Vanna's whole corpus, DataHub `get_dataset_queries` | the single highest-leverage extra — five ecosystems converged on it independently |
| Named boolean filters ("active_users") | Cortex `filters`, Cube `segments` | reusable business predicates the model doesn't re-derive |
| Freeform LLM instructions | Cortex `module_custom_instructions`, Cube `meta.ai_context` | escape hatch; Databricks explicitly calls text instructions a **last resort** |
| High-cardinality value search hookup | Cortex `cortex_search_service` | when values can't be enumerated, point the agent at a lookup affordance instead |
| Time-dimension designation | Cortex (dedicated block), dbt/Cube (typed) | date questions bind to the right column |
| PII/visibility flags | Cortex `private_access`, Cube `public/mask`, catalog PII tags | keep a column out of the model's reach |

Note what's *absent* from the classic formats (dbt, MDL): synonyms, sample values,
verified queries — the BI-governance lineage vs the AI-native lineage is visible in the
field lists. Cortex Analyst, designed for text-to-SQL from day one, has everything.

**Verified queries hide two different mechanisms** worth keeping distinct: (1) few-shot
exemplars that shape generated SQL, and (2) exact/parameterized match that *replaces*
generation with a trusted template (Genie's "Trusted" badge). Only the second gives a
correctness guarantee.

---

## 4. What moves accuracy, ranked by evidence

The measured deltas, strongest first:

| Artifact | Evidence | Delta |
|---|---|---|
| **Any curation at all** | LinkedIn ablation; Anthropic | schema-only = **9%** answer quality; no-skills = **21%** accuracy — the baseline is unusable |
| **Value-decoding knowledge** | BIRD benchmark ablation ([arXiv:2305.03111](https://arxiv.org/abs/2305.03111)) | GPT-4 **~35% → 54.9%** with human "evidence" hints (mostly value semantics). Value profiling is the automatable approximation |
| **Example / verified queries** | LinkedIn ablation ([arXiv:2507.14372](https://arxiv.org/html/2507.14372v1)) | **+18 points** — their single biggest context contributor |
| **Question-time value retrieval** | CHESS ([arXiv:2405.16755](https://arxiv.org/abs/2405.16755)) | +4.76pp execution accuracy from entity/context retrieval |
| **Table descriptions** | Pinterest ([blog](https://medium.com/pinterest-engineering/how-we-built-text-to-sql-at-pinterest-30bad30dabff)) | table-retrieval hit rate **40% → 90%** (retrieval, not end-to-end) |
| **LLM-generated column descriptions** | [arXiv:2502.20657](https://arxiv.org/abs/2502.20657), [arXiv:2408.04691](https://arxiv.org/abs/2408.04691) | +1–2pp auto-generated (~37–40% of what human docs deliver); notably, machine-verbose descriptions *beat* human-terse gold ones |
| **3 sample rows** | Rajkumar et al ([arXiv:2204.00498](https://arxiv.org/abs/2204.00498)) | 59.9% → **67.0%** over bare DDL; **10 rows was worse (63.3%)** — the source of LangChain's `sample_rows_in_table_info=3` default |

The ranking that falls out — **example queries and value semantics > table descriptions
> column descriptions > sample rows** — matches Databricks' explicit curation priority
for Genie authors (example SQL and SQL expressions before free-text instructions).

---

## 5. The authoring reality: who writes this stuff

- **Nobody hand-writes from scratch anymore.** Snowflake bootstraps from
  information_schema (auto-sampling 25 values per dimension), Databricks "Genie Code"
  reads the data and drafts descriptions/examples for review, dbt ships a wizard, WrenAI
  auto-scaffolds MDL for human enrichment, dbt's own benchmark had an LLM author the
  missing semantic models "without us writing any code." **Auto-scaffold, human-enrich
  is the default authoring mode of 2026.**
- **The human layer is thin but load-bearing.** LinkedIn *mandates* human table
  descriptions at dataset certification; Cortex practitioners' rule of thumb is
  **5–10 verified queries to start**; a Genie deployment went 53% → 100% on its test set
  only after "systematic remodeling, annotation, and iterative benchmarking"
  ([Colrows](https://colrows.com/blogs/cortex-analyst-vs-genie/)). Field estimate worth
  remembering: teams that skimp on the model "get the 90% accuracy claim in benchmarks
  and 60–70% in the real world."
- **The promotion loop is the maintenance engine.** Databricks mines query history and
  author thumbs-up into suggested examples; Snowflake surfaces verified-query candidates
  from real user interactions; LinkedIn lets users certify queries from the editor;
  OpenAI trust-ranks context by query provenance ("queries from popular dashboards
  written by data scientists rank highest"). Curation converges to *reviewing
  suggestions*, not authoring.
- **OpenAI's standout pattern:** a nightly Codex job reads **the pipeline code that
  produces each table** (100–200 tables/night) and derives what the table contains, how
  it's derived, and when to prefer it over a lookalike — semantics from *producing
  code*, not just schema ([via ByteByteGo](https://blog.bytebytego.com/p/how-openai-built-its-data-agent);
  primary page 403'd, secondhand).
- **The graveyard is real.** Standalone metric stores failed commercially pre-AI — per
  Benn Stancil's own retrospective, vendors "weren't able to sell their metric store
  without a built-in viz layer" ([benn.substack.com](https://benn.substack.com/p/the-context-layer)).
  A (vendor-published, methodology-unshown) 2026 survey claims ~18% dbt-SL adoption and
  **35% of teams using no semantic layer at all**. The "semantic layer is dead" genre
  argues agents can infer structure on the fly; the counter-consensus is that AI
  *amplified* semantic drift rather than papering over it. Both camps agree on the
  middle: **match artifact weight to the team that must maintain it.**

---

## 6. Value profiling mechanics: the concrete numbers

The thresholds production systems actually use:

- **Cardinality bands.** The convergent rule: **≤~15 distinct values → enumerate all**
  ("Death of Schema Linking" includes all values below 15; Cortex recommends inline
  `sample_values` for 1–10); **mid-cardinality → curated value dictionary** (Genie's
  hard caps: string columns only, **≤1,024 values, ≤127 chars each, ≤120 columns per
  space**); **high-cardinality → don't inline, give the agent a lookup affordance**
  (Cortex attaches a search service per dimension; CHESS pre-builds a MinHash/LSH index
  over all column values and retrieves top-10 candidates per question keyword —
  cutting value lookup from ~5min to ~5s).
- **How columns are chosen:** AWS's pipeline uses **an LLM to decide which columns are
  categorical dimensions worth profiling** (not a hard rule), then
  `SELECT DISTINCT ... ORDER BY COUNT(*) DESC LIMIT K` daily. Snowflake's generator
  auto-extracts 25 sample values per dimension, 3 per time dimension/measure.
- **Sample rows vs value profiles:** the 2022 evidence (3 rows optimal) predates the
  2024–25 convergence on **typed value profiles + question-time literal retrieval** —
  higher accuracy (CHESS ablation), lower token cost (Uber's wide tables ran
  **40–60K tokens** as raw schema+samples, forcing a column-prune agent), and lower
  leak risk (random rows contain whatever the table contains). No paper directly A/Bs
  rows-vs-profiles — flagged in §9.
- **PII at profile time.** OpenMetadata is the reference implementation of the right
  pattern: at profiling, regex on column names + **Presidio NER over sampled values**
  → tag columns `PII.Sensitive` at ~80/100 confidence → **sample data for tagged
  columns is automatically masked**. Genie excludes row-filtered/column-masked tables
  from value matching entirely. Vanna's posture is architectural: database contents
  never reach the LLM unless `allow_llm_to_see_data=True`. The research papers, by
  contrast, index *everything* — nobody has measured the accuracy cost of excluding
  PII columns from profiles.

---

## 7. Staleness: everyone's second problem

- **The sharpest statement is Anthropic's:** "Skill docs describe a data model that
  changes daily, so without active maintenance they're wrong within weeks." Their fix:
  **colocate the semantic docs in the same repo as the transformation models, with CI
  checks flagging model changes not reflected in the docs.**
- **The standard invalidation triggers,** in increasing sophistication: schema
  fingerprint (hash column names/types, rebuild on mismatch — the simplest workable
  pattern); **successful dbt run touching upstream models** (dbt's own SL cache
  invalidates on exactly this); scheduled re-profiling (AWS refreshes schema + values
  **daily**); lifecycle automation (LinkedIn: popular tables auto-ingest, DataHub
  deprecation signals **auto-offboard** stale datasets, quarterly human review).
- Snowflake's YAML, by contrast, has **no auto-sync at all** — a static artifact with a
  human refresh loop. Nobody defends this; it's the cost of the curated-YAML approach.

---

## 8. Schema linking at small scale: don't

Two papers stake out the debate, and both point the same way for NGO-scale warehouses:

- **"The Death of Schema Linking?"** ([arXiv:2408.07702](https://arxiv.org/abs/2408.07702)):
  for models whose context fits the whole schema, aggressive filtering *hurts* —
  their pipeline hit 71.83% on BIRD (leaderboard top at submission) with **no schema
  linking**, spending effort on value augmentation and correction instead.
- **The rebuttal** ([arXiv:2510.14296](https://arxiv.org/abs/2510.14296)): bidirectional
  retrieval closes ~50% of the full-vs-oracle-schema gap — linking still pays **on
  large schemas**.

At dozens of tables, both say: **ship the full enriched schema; spend the effort on
enrichment quality, not retrieval sophistication.** LinkedIn's retrieval pipeline
(embedding → top-20 → LLM re-rank → 7) is the pattern for the *thousands*-of-tables
regime, not ours.

---

## 9. What we could not verify

- OpenAI data-agent details are secondhand (primary page 403'd); their 600 PB vs 1.5 EB
  scale numbers conflict between tellings.
- Uber: no public info on workspace freshness/ownership; the "140,000 hours/month
  saved" figure is third-party extrapolation, not in Uber's blog.
- Snowflake's 30,980-token limit and 25/3/3 sample-value minimums come from the
  generator repo and secondary guides, not the current core docs.
- The ~18%/35% adoption survey is vendor-published with unshown methodology.
- BIRD's exact no-evidence baseline (34.88%) verified only via secondary summaries.
- No direct A/B exists of "3 raw sample rows vs a value profile" holding all else equal.
- LinkedIn's actual K for top-K values; AWS's K in their profiling LIMIT.
- Google's "two-thirds fewer errors" claim: internal test, methodology unpublished.

---

## 10. What this means for Dalgo v3

Mapped against `features/chat-with-data/v3/plan.md` (table cards + value profiles + full
injection, no embeddings, no metric layer):

**Validated by this research — proceed as planned:**
- **Value profiles as the centerpiece of M1.** The two strongest evidence lines (BIRD's
  ~20pp evidence gap; Snowflake exempting sample values from its token cap) both point
  at value semantics as the payload. Our canary failure (TEXT dates,
  `'Q3 (Oct-Dec 2025)'` labels) is a textbook value-decoding failure.
- **Full-schema injection over retrieval sophistication (M2).** §8 is unambiguous at
  our scale. Rank-for-ordering (top-k full cards + one-liner stubs) is fine; embeddings
  stay out.
- **No metric-compile layer yet.** Its edge is largest on complex metric questions our
  evals don't show us failing; the maintenance graveyard (§5) is real; and Dalgo's
  no-analytics-engineer constraint makes WrenAI's "auto-scaffold, human-enrich" the
  only viable authoring mode — which is what M1 is.
- **Deterministic cards first, LLM `--describe` pass optional.** Generated descriptions
  are worth +1–2pp — real but not the main course; and machine-verbose beats
  human-terse, so the pass is safe to automate when we add it.
- **The existing `profile_column` tool is our high-cardinality answer** — the agent
  already has the "look up values at question time" affordance Cortex builds a search
  service for. Keep it; cards make it rarer, not obsolete.

**Plan amendments this research argues for:**

1. **Add a verified-queries slice (biggest change).** Every ecosystem converged on it;
   LinkedIn measured it as the single biggest contributor (+18); Anthropic's negative
   result says *distill* history, don't RAG it. Dalgo already has the raw material:
   `ChatWithDataTurnAudit` rows with executed SQL, and eval-passing golden items. A
   minimal loop: promote eval-verified and thumbs-up SQL into per-table
   `verified_queries` entries on the card (question + SQL + verified_at), injected as
   few-shot when the question matches the table. Start with 5–10 per org (the Cortex
   practitioner baseline).
2. **PII-classify before profiling (M1, not later).** The OpenMetadata pattern: run
   detection (we already have `agent/pii.py` regexes; column-name heuristics are cheap)
   over candidate values *at profile time*, and exclude tagged columns from value
   profiles entirely. Our current PII middleware masks at chat time — profiling must
   not become the leak path around it.
3. **Mark exhaustive enumerations.** Adopt the `is_enum` distinction: at cardinality
   ≤15, record the value list as *complete* (the model may treat it as a closed domain);
   at 15–50 record top-K with counts as *samples*. One boolean on the card, materially
   different prompt semantics.
4. **Add `synonyms` as an optional card field now, populate later.** Cheap to carry in
   the JSON; the natural producer is the LLM `--describe` pass plus promotion from
   router/audit vocabulary mismatches.
5. **Wire re-sync to pipeline completion in v3, not "future".** The dbt-run trigger is
   the industry-standard invalidation (§7), Dalgo *runs the pipelines* (Prefect blocks
   exist), and Anthropic's "wrong within weeks" warning applies directly to
   NGO warehouses that re-model monthly.

**One deliberate divergence to record:** every commercial system leans on a human
curation UI (Genie spaces, Snowsight, LookML IDE). Dalgo's users can't be the curators —
so our human layer must be *us* reviewing LLM-suggested enrichments, and the promotion
loop must run off signals users produce anyway (thumbs-up, eval results, usage), never
off asking Priya to write YAML.
