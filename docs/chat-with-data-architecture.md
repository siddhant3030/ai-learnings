# Chat With Data: RAG vs Agentic vs Hybrid — Architecture Research & Recommendation for Dalgo

> Deep-research report (multi-source, adversarially fact-checked). 22 sources fetched, 104 claims
> extracted, 25 verified by 3-vote adversarial panel, 22 confirmed / 3 refuted, synthesized to 7
> high-confidence findings. Sources are primary vendor engineering blogs, official docs, arXiv
> benchmarks, and open-source repos. Date: 2026-06-19.

---

## 0. The one-line verdict

**Go Hybrid, agent-orchestrated, semantic-layer-first.** Every leading vendor independently converged
on the same shape: a **governed semantic/metadata layer** is the primary grounding mechanism, an
**agentic workflow** orchestrates classify → retrieve → generate → validate, and **RAG** retrieves
*supporting* context (verified queries, value literals, docs) but **never supplies the exact metric
or SQL definition**.

For Dalgo's A/B/C decision, the answer is **"A's pre-population + B's planner/executor, with C as a
guarded fallback"** — not a single one. Concretely:

- **Adopt A's core move** (pre-populate the build-time metadata artifact into context) — this is the
  single highest-leverage change and directly attacks your 60–90s problem.
- **Restructure into B** (a lightweight planner + executor) so the loop stops re-discovering what you
  already computed at build time.
- **Keep C only as a guarded, clearly-labeled exploratory fallback** for questions outside your
  modeled scope. Unguarded one-shot text-to-SQL is the thing the entire industry is engineering
  *away* from — the benchmark evidence below is decisive.

---

## 1. The decisive evidence: production text-to-SQL accuracy collapses

This is the fact that should anchor every architecture decision. Academic leaderboards (Spider ~90%,
BIRD ~70%) **do not** reflect enterprise reality.

| Benchmark (enterprise-realistic) | SOTA accuracy | Notes |
|---|---|---|
| **BEAVER** (real private warehouse) | **10.8%** | SOTA agentic frameworks + GPT-5.2 class model |
| BEAVER **with oracle subtask hints** | 30.1% | Proves the bottleneck is *subtask resolution*, not SQL syntax |
| **BIRD-Ent** (enterprise variant) | 39.1% | "sharp performance drop" from academic BIRD |
| **Spider-Ent** (enterprise variant) | 60.5% | Same — realistic schemas crater accuracy |

**Why it collapses:** enterprise settings have schema scopes of **4,000+ columns**, abbreviated /
domain-specific names, and the knowledge needed to answer a question scattered across documents
totaling **~1.5M tokens**. The model doesn't fail at writing `SELECT` — it fails at *decomposition,
domain grounding, and finding the right tables/columns/definitions*.

> **Implication for Dalgo:** your build-time enricher (grain, keys, time column, answerability, PII)
> is not a nice-to-have — it is precisely the "subtask resolution" scaffolding that the research shows
> is the actual bottleneck. You've already built the expensive part. The architecture mistake would be
> to make the LLM *re-derive* it at query time (which is exactly what your current 8–10 turn loop does).

Sources: BEAVER ([arXiv 2409.02038](https://arxiv.org/pdf/2409.02038)), enterprise variants
([OpenReview gXkIkSN2Ha](https://openreview.net/forum?id=gXkIkSN2Ha)).

---

## 2. Industry landscape — what the leaders actually built

The verified pattern across vendors: **agents for orchestration/validation, semantic layer for
grounding, RAG for examples & literals, deterministic compilation for the final query.**

### Snowflake Cortex Analyst (the most documented blueprint)
A genuine **multi-agent** system:
- A **collection of SQL-generation agents, each backed by a different LLM** (different models excel at
  different question types).
- A **classification agent** that rejects ambiguous / unanswerable questions *upfront* (this is your
  `needs_clarification` intent, done as a first-class gate).
- An **error-correction agent** that runs generated SQL through the **SQL compiler** to catch
  syntactic/semantic errors and *invented entities/functions* before returning anything.
- **RAG two ways:** retrieves **semantically similar human-verified queries** as trustworthy
  exemplars, and does **semantic search over literals** to fix the vocabulary gap (user says "USA",
  column holds "United States of America").
- Grounding via a **semantic model** that supplies descriptive names, synonyms, descriptions, metrics,
  and filters — "the missing context a super-smart analyst who is new to your data would need."

[snowflake.com/engineering-blog/snowflake-cortex-analyst-behind-the-scenes](https://www.snowflake.com/en/engineering-blog/snowflake-cortex-analyst-behind-the-scenes/)

### ThoughtSpot Spotter
- Semantic model as the **"Rosetta Stone"** — encodes business definitions, security rules, join logic,
  and calculation semantics "so autonomous agents can interpret intent probabilistically and return
  deterministic results."
- The LLM emits an **intermediate representation (TML)**, *not* final SQL. Proprietary,
  database-specific generation then **deterministically compiles** that into SQL. The LLM never hands
  you the query string directly.

[thoughtspot.com/blog/spotter-semantics](https://www.thoughtspot.com/blog/spotter-semantics)

### Databricks Genie (next-gen)
- Databricks explicitly **"chose not to build another error-prone text-to-SQL interface."**
- Next-gen Genie's agent architecture **"draws on the most relevant and trusted assets — certified
  Genie Spaces, governed dashboards, Databricks Apps — reusing the logic already embedded there,"**
  with metadata guiding routing so higher-trust sources win.
- ⚠️ *Verified caveat:* this is a **prioritization, not a prohibition** — trusted assets still
  generate SQL underneath. The panel **refuted** the stronger claim that routing alone "prevents
  hallucination." Routing reduces risk; it is not a guarantee.

[databricks.com/blog/next-generation-databricks-genie](https://www.databricks.com/blog/next-generation-databricks-genie)

### dbt (Semantic Layer + MetricFlow + MCP server)
- **MetricFlow compiles metric definitions to SQL deterministically.** The LLM's job shrinks to
  *selecting the right metric + dimensions*, not writing the aggregation/joins.
- On dbt's proprietary **ACME Insurance benchmark**, the Semantic Layer beat raw text-to-SQL:
  - claude-sonnet-4-6: **98.2% vs 90.0%**
  - gpt-5.3-codex: **100.0% vs 84.1%**
  - (⚠️ *in-scope/modeled questions only*; all-questions SL accuracy was **72.7%**, ~83% of addressable
    questions answered, small 11-question / 20-run benchmark — treat as **directional**, not guaranteed.)
- Model-grounding alone (feeding schemas/descriptions) raised text-to-SQL from **64.5%** on raw tables
  to **84.1–90.0%**.

[getdbt.com/blog/semantic-layer-vs-text-to-sql-2026](https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026),
[getdbt.com/blog/mcp](https://www.getdbt.com/blog/mcp)

**Cross-vendor takeaway:** four independent teams, same conclusion — *don't let the LLM emit the final
query unconstrained; ground it in governed definitions and validate before returning.*

---

## 3. The role of RAG — what belongs, what never belongs

RAG is **support, not source of truth.**

**✅ Put in RAG (retrieve at query time):**
- **Verified/golden queries** — human-approved Q→SQL pairs as few-shot exemplars (Cortex's biggest
  accuracy lever).
- **Value literals** — distinct column values for the vocabulary gap ("MGNREGA" ↔ "rural employment
  scheme"; "Maharashtra" ↔ "MH"). This is exactly your `get_distinct_values` need, but pre-indexed.
- **Documentation, business glossary, historical analyses** — narrative context.
- **Table/column descriptions** for relevance ranking — use **hybrid BM25 + semantic** search.

**❌ Never derive from RAG (must be deterministic / authoritative):**
- **Exact metric definitions** — "active farmer", "completion rate". These come from the semantic
  layer / your enriched artifact, not nearest-neighbor guessing.
- **The final SQL** — compile or validate it; don't trust retrieval to have produced a correct query.
- **Join logic and grain** — authoritative metadata, not similarity.

dbt's own guidance: *"always prefer Semantic Layer tools like `query_metrics` that enforce certified
logic"* and treats free-form *"SQL execution as powerful but less controllable."*

**Retrieval best practices (verified):** hybrid retrieval (BM25 + embeddings) beats either alone;
enrich each chunk with metadata; and — critically for you — **your artifact is tiny** (tens of tables
per dashboard, not millions of docs). You do **not** need a vector DB. Embed table/metric descriptions
at build time, do **in-memory cosine + keyword** at query time.

---

## 4. Where RAG fails (and why pure-RAG is not enough)

Verified failure modes that retrieval *cannot* solve alone:
- **Wrong/similar metric selection** — two metrics with near-identical descriptions; retrieval returns
  both, picks wrong.
- **Multi-hop / decomposition** — "compare enrollment vs completion across programs" needs planning,
  not a single retrieval.
- **Scattered context** — the 1.5M-token reality; no top-k retrieval reliably assembles it.
- **Ambiguity** — needs a *clarification turn*, not a better embedding.
- **Retrieval thrash / tool storms / context bloat** — documented agentic failure modes when the loop
  has no plan (this is literally your 8–10 turn problem).

> ⚠️ **Refuted over-claims (don't repeat these):** The panel killed three tempting claims:
> (a) "text-to-SQL fails silently while the semantic layer always fails loudly" — **false** (0-3);
> (b) "correct metric selection *guarantees* a correct query" — **false** (1-2);
> (c) "routing to trusted assets alone *prevents* hallucination" — **false** (1-2).
> A semantic layer is **necessary but not sufficient**. Cross-system and out-of-scope queries can still
> hallucinate. Keep your validator.

---

## 5. Agentic workflows — what to make agentic vs deterministic

| Make it **agentic** (LLM reasoning) | Keep it **deterministic** (code) |
|---|---|
| Intent classification & ambiguity rejection | Allowlist enforcement, row limits |
| Metric/table selection & planning | Metric → SQL compilation (where modeled) |
| Decomposing multi-hop questions | SQL syntactic validation (compiler/parser) |
| Repairing a rejected query (error-correction agent) | PII masking |
| Narrating/explaining results | Result-set comparison in evals |

**Where agents add risk:** unbounded tool loops (cost + latency + thrash), and treating a generated
plan as ground truth with no recovery path. **Cost/latency control** in production systems comes from:
upfront classification (kill bad questions before they spend tokens), routing to vetted assets,
parallel tool execution, and caching.

---

## 6. RAG vs Agentic vs Hybrid — the matrix

| Dimension | RAG-only | Agent-only | **Hybrid (recommended)** |
|---|---|---|---|
| **Strength** | Fast, cheap, simple | Handles multi-hop, self-corrects | Grounded *and* flexible |
| **Weakness** | Can't plan/validate; wrong-metric | Slow, expensive, can thrash | More moving parts |
| **Latency** | Low (1 retrieval + 1 gen) | High (serial turns) | Medium (pre-populate + short loop) |
| **Cost/query** | Low | High (many LLM calls) | Medium |
| **Reliability** | Brittle on ambiguity | Variable | Highest (validation gate) |
| **Scalability** | Degrades on big schemas | Degrades on cost | Best with build-time precompute |
| **Observability** | Easy | Hard (need traces) | Needs traces + retrieval metrics |

**By stage:** Early → semantic-layer + retrieval + one short generation pass. Growth → add agentic
planning + validation + evals. Enterprise → multi-agent, full observability, governance, tenant
isolation. **Dalgo is "growth": you already have the grounding; you need the orchestration + evals.**

---

## 7. Chat for dbt projects specifically

The **dbt MCP server** is the reference grounding interface and worth studying even if you don't adopt
it directly:
- **Semantic Layer tools** (`query_metrics`, `get_dimensions`, `get_dimension_values`, `get_entities`,
  `list_metrics`, `get_metrics_compiled_sql`) — query governed metrics, get **compiled SQL** back,
  never re-derive definitions.
- **`text_to_sql`** grounded in project context + **`execute_sql`** for the exploratory long tail.
- Grounds answers with **compiled SQL, model schemas/descriptions, lineage, and test results** as
  *factual* context.

**What to retrieve vs tool-call vs pre-assemble (the core design question for you):**
- **Pre-assemble** (inject before the loop): the chart→table allowlist, the enriched metadata for the
  1–3 tables most relevant to the question, schema snippets, resolved time scope. *This is the fix.*
- **Tool-call** (live, at query time): distinct value lookups for filters, and the actual
  `run_sql_query`. These are the only genuine unknowns.
- **Retrieve** (RAG): verified-query exemplars, value literals, glossary/docs.

⚠️ The Semantic Layer only answers **modeled/in-scope** questions and **errors on coverage gaps** (dbt:
~83% of addressable questions). You need an **explicit policy for the long tail** — guarded free-form
SQL or graceful refusal. (Open question; see §10.)

---

## 8. Production readiness — the gap you must close first

You have **no golden dataset / no evals**. The research is unambiguous that this is *the* blocker.

**Evaluation (do this before any further architecture change):**
- Measure **execution accuracy** — run expected vs predicted SQL and **compare result DataFrames**
  (Ragas recommends `datacompy`), not SQL string similarity. "The ultimate test of correctness" is
  whether the *numbers match*.
- **Iterative, error-analysis-driven prompt refinement** with generic schema-grounded guardrails
  (target the 3–5 highest-frequency error patterns, not per-row fixes) drove a documented run from
  **2% → 60.6% → 70.7%** execution accuracy. *(⚠️ single first-party source; weak 2% baseline inflates
  the headline — real working range ~60–70%. Still: the method is sound and it's the loop you lack.)*
- Build **~30 golden questions per dashboard** from real partner NGOs, tagged by type
  (time_filter, count, multi_table, group_by). Run on every PR.

**Observability:** retrieval-quality metrics, agent/tool traces, per-query latency + token cost +
warehouse calls persisted on each message. (You have `timing_breakdown` — persist and aggregate it.)

**Security/governance (thinly covered by sources — flagged as your responsibility):** tenant isolation,
row/column access control, prompt-injection defense at the tool layer, metric auditability,
reproducibility. Don't infer these from vendor blogs; design them explicitly for multi-tenant NGO data.

**Add an answer-faithfulness gate:** your semantic verifier checks "does this SQL answer the question?"
— add a check for "does the natural-language answer faithfully describe the rows the SQL returned?"
(A correct query can still be narrated wrong.)

---

## 9. Recommended architecture for Dalgo

```
                         ┌─────────────────────── BUILD TIME ───────────────────────┐
                         │  allowlist builder · dbt manifest parse · warehouse schema │
                         │  LLM enricher → {grain, keys, time col, PII, answerability}│
                         │  + embed table/metric descriptions (in-memory index)       │
                         │  → metadata artifact in Postgres (+ staleness fingerprint) │
                         └────────────────────────────┬───────────────────────────────┘
                                                      │  (you already built this — the expensive part)
   USER QUESTION                                      ▼
        │     ┌──────────────────────────────────────────────────────────────────────┐
        ├────▶│ 0. CONTEXT ASSEMBLY (deterministic, no LLM)                            │
        │     │    hybrid BM25+cosine over artifact → top 1–3 tables                   │
        │     │    inject: their full metadata + schema snippets + resolved time scope │
        │     │    + retrieve: verified-query exemplars, value literals                │
        │     └──────────────────────────────┬───────────────────────────────────────┘
        │                                     ▼
        │     ┌──────────────────────────────────────────────────────────────────────┐
        │     │ 1. CLASSIFY / PLAN  (1 LLM call — the "planner")                       │
        │     │    ambiguous? → clarification turn (persist what's missing)            │
        │     │    in-scope & modeled? → metric-select path                            │
        │     │    needs exploration? → guarded free-form SQL path                     │
        │     │    emits a structured plan {tables, filters, time, output}             │
        │     └──────────────────────────────┬───────────────────────────────────────┘
        │                                     ▼
        │     ┌──────────────────────────────────────────────────────────────────────┐
        │     │ 2. EXECUTE  (the "executor" — parallel where possible)                 │
        │     │    resolve filter literals · generate/compile SQL · run query          │
        │     └──────────────────────────────┬───────────────────────────────────────┘
        │                                     ▼
        │     ┌──────────────────────────────────────────────────────────────────────┐
        │     │ 3. VALIDATE  (deterministic + LLM)                                     │
        │     │    allowlist guard · SQL compiler check (invented entities/functions)  │
        │     │    · semantic verifier · answer-faithfulness check · PII mask          │
        │     │    fail → repair loop (bounded) or fall back / refuse                  │
        │     └──────────────────────────────┬───────────────────────────────────────┘
        │                                     ▼
        └────────────────────────  STREAM answer + progress events + [View SQL]
```

**This keeps everything you've built** (allowlist, enricher, guard, verifier, PII mask, WebSocket) and
changes *where the work happens*: from re-discovery at query time to pre-population + a 2-stage
planner/executor. Realistic outcome: 60–90s → ~15–25s, and ~5–10s for simple questions on the
fast path. With streaming it *feels* production-grade.

---

## 10. The A/B/C decision, answered

| Option | Verdict | Why |
|---|---|---|
| **A — pre-populate the tool loop** | ✅ **Adopt the pre-population** | Highest leverage; directly fixes the "re-discovering build-time metadata" waste. But pre-population *inside the existing loop* isn't enough on its own. |
| **B — planner + executor** | ✅ **Adopt as the loop structure** | Collapses 8–10 serial turns to ~2 stages; matches every vendor's orchestration shape; enables parallel execution. |
| **C — one-shot text-to-SQL** | ⚠️ **Fallback only, guarded & labeled** | Unguarded one-shot is what the 10.8% BEAVER number warns against. Fine as a *secondary, validated* exploratory path for out-of-scope questions; never the primary path. |

**Net: A's pre-population + B's structure, with C as a guarded fallback.** Not a single letter — the
right answer is the combination the industry converged on.

### Build order (do not reorder the first two)
1. **Golden dataset + execution-accuracy evals** (~30 Qs/dashboard, datacompy result comparison).
   *Without this, every change below is a blind guess.*
2. **Context pre-population** (A) — hybrid search over the artifact, inject top-3 tables' metadata +
   schema + time scope before any LLM turn.
3. **Planner/executor restructure** (B) + parallel tool execution.
4. **Streaming** output + progress events (you already emit `DashboardChatProgressStage`).
5. **Staleness banner** (compare fingerprint on session open).
6. **Cost/latency tracking** persisted per message.
7. **Clarification turn** made stateful (persist what's missing, resume).
8. **Answer-faithfulness gate** + **value-literal retrieval** for the vocabulary gap.
9. **Out-of-scope policy** (guarded free-form SQL vs refusal) — see open questions.

---

## 11. Caveats & open questions (read before committing engineering effort)

**Caveats on the evidence:**
- **Vendor-blog bias:** architecture descriptions are appropriate primary sources, but the *accuracy
  numbers* (dbt 98–100%, ThoughtSpot "deterministic") are self-reported. dbt's benchmark is small
  (11 Qs, 20 runs) and in-scope-only. Treat as **directional**.
- **Fast-moving:** several sources are 2026-dated and shifting (Cortex's multi-LLM design is Aug-2024;
  Genie's next-gen architecture is Apr-2026). Re-verify before building.
- **Split votes:** Databricks "rejected text-to-SQL" and "routes rather than synthesizes" are
  prioritizations, not absolute prohibitions — trusted assets still generate SQL underneath.

**Open questions the research did NOT resolve (your design calls):**
1. **Out-of-scope behavior** — what happens to the ~17% of questions the modeled layer can't answer?
   Guarded free-form SQL, or refuse? Needs an explicit Dalgo policy.
2. **Concrete latency/cost budgets** — no published per-query figures; you must measure whether
   planner/executor actually lands under your threshold.
3. **Bootstrapping evals** with **no historical query logs** and **PII constraints** in a multi-tenant
   NGO context — methodology (Ragas/datacompy) is clear, but the cold-start dataset is on you.
4. **Tenant isolation / access control / prompt-injection** at the tool layer — thinly covered by
   sources; design explicitly, don't infer from vendor blogs.

---

## Sources (primary, verified)

- Snowflake Cortex Analyst — behind the scenes: https://www.snowflake.com/en/engineering-blog/snowflake-cortex-analyst-behind-the-scenes/
- ThoughtSpot Spotter semantics: https://www.thoughtspot.com/blog/spotter-semantics
- Databricks next-gen Genie: https://www.databricks.com/blog/next-generation-databricks-genie
- dbt Semantic Layer vs text-to-SQL (2026): https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026
- dbt MCP announcement: https://www.getdbt.com/blog/mcp
- dbt MCP server (repo): https://github.com/dbt-labs/dbt-mcp
- How the dbt Semantic Layer works: https://www.getdbt.com/blog/how-the-dbt-semantic-layer-works
- BEAVER enterprise text-to-SQL benchmark: https://arxiv.org/pdf/2409.02038
- Enterprise text-to-SQL (BIRD-Ent/Spider-Ent): https://openreview.net/forum?id=gXkIkSN2Ha
- Ragas text-to-SQL evaluation: https://docs.ragas.io/en/stable/howtos/applications/text2sql/
- Production-readiness benchmarks: https://arxiv.org/pdf/2503.21602 · https://arxiv.org/pdf/2406.13352 · https://arxiv.org/pdf/2506.08837
- Agentic text-to-SQL / RAG failure modes: https://towardsdatascience.com/agentic-rag-failure-modes-retrieval-thrash-tool-storms-and-context-bloat-and-how-to-spot-them-early/ · https://medium.com/data-science-collective/building-agentic-text-to-sql-why-rag-fails-on-enterprise-data-lakes-156d5d5c3570
