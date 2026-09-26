# GraphRAG: What It Is, Who Actually Runs It, and How Anyone Evaluates It

> Deep-research report, 2026-09-25. Four parallel research agents → ~60 primary sources
> (arXiv papers with PDFs text-extracted rather than summarized, vendor docs, engineering
> blogs, GitHub issues), findings cross-checked across agents; every number below is
> labelled *verified-at-source*, *vendor-claim*, or *secondhand*, and the ones that
> dissolved under checking are in §9. Audience: the Dalgo chat-with-data team, deciding
> whether any part of the GraphRAG stack belongs in our text-to-SQL agent.
> Companions: [`semantic-layer.md`](./semantic-layer.md) (read that first — it covers the
> same decision from the schema-enrichment side), [`evals.md`](./evals.md) (general eval
> theory; this report assumes it and goes graph-specific),
> [`../docs/chat-with-data-evals.md`](../docs/chat-with-data-evals.md) (our own harness).

---

## Table of Contents

1. [The one-paragraph thesis](#1-the-one-paragraph-thesis)
2. [Four unrelated things are called "GraphRAG"](#2-four-unrelated-things-are-called-graphrag)
3. [Where graphs actually win, and by how much](#3-where-graphs-actually-win-and-by-how-much)
4. [What it costs](#4-what-it-costs)
5. [Production reality: the verified list is short](#5-production-reality-the-verified-list-is-short)
6. [Failure modes, ranked](#6-failure-modes-ranked)
7. [How GraphRAG gets evaluated](#7-how-graphrag-gets-evaluated) ← the long one
8. [Decision rules and hybrid patterns](#8-decision-rules-and-hybrid-patterns)
9. [What we could not verify](#9-what-we-could-not-verify)
10. [What this means for Dalgo](#10-what-this-means-for-dalgo)

---

## 1. The one-paragraph thesis

GraphRAG's verified win condition is **narrow**: multi-hop reasoning and global/thematic
sensemaking over a document corpus. On single-hop factual lookup — the majority of real
query mixes — plain RAG with a reranker ties or beats every graph variant, in three
independent 2025-26 benchmarks, one of which (ICLR'26) opens by stating that "GraphRAG
frequently underperforms vanilla RAG on many real-world tasks." The wins that do exist
were mostly measured with a pairwise LLM-judge protocol that a debiased replication can
swing 30-50 points on position and length alone — and which returns **90/10 when you
compare a system to itself**. Meanwhile the indexing bill is real (~$13/M source tokens,
41× vanilla RAG's construction time), the cheap fix everyone cites (LazyGraphRAG) was
never released in Microsoft's open-source library, ~40% of the entities in an LLM-built
graph are duplicates, and the incremental-update path has an open silent-corruption bug.
For a text-to-SQL team none of this is the relevant question anyway: the evidence that
*metadata* graphs help SQL generation is strong and separate (LinkedIn's 48%→9% ablation,
data.world's 16.7%→54.2%), it requires no LLM entity extraction, and the largest single
jump in that literature came not from the graph but from an **ontology-based query
validator** bolted on afterwards.

---

## 2. Four unrelated things are called "GraphRAG"

Asking "should we use GraphRAG?" is unanswerable until you say which of these you mean.
They share a word and almost nothing else — different costs, different failure modes,
different evidence bases.

| # | Architecture | Canonical implementation | What the graph is | Cost profile |
|---|---|---|---|---|
| **1** | **Community-summary map-reduce** | Microsoft GraphRAG ([arXiv:2404.16130](https://arxiv.org/abs/2404.16130) v2, 2025-02) | LLM-extracted entity KG + Leiden communities + pregenerated summaries per level | Heaviest: LLM call per chunk *and* per community, per level |
| **2** | **Graph-as-traversal-index** | HippoRAG 2 (Personalized PageRank, [arXiv:2502.14802](https://arxiv.org/abs/2502.14802), ICML'25), LightRAG ([arXiv:2410.05779](https://arxiv.org/abs/2410.05779)), GraphReader (agentic walk) | Entity graph walked at query time; no summaries | Moderate; HippoRAG2 ~$2.85/M tokens |
| **3** | **Hierarchical summarization, no graph** | RAPTOR ([arXiv:2401.18059](https://arxiv.org/abs/2401.18059), ICLR'24) | Recursive cluster-and-summarize tree | ~$6.38/M tokens |
| **4** | **Query the real database in its query language** | Text2Cypher, text-to-SQL | The graph *is* your schema; nothing is extracted | ~Free; deterministic |

Neo4j's own pattern catalogue ([graphrag.com](https://graphrag.com/reference/), updated
2025-07-11, vendor-run) counts category 4 as GraphRAG — a definitional choice that
flatters their product, but also the category Dalgo is already in.

**Microsoft GraphRAG's own scope statement**, verbatim from the paper, is worth pinning
because almost every downstream citation drops it:

> "RAG fails on global questions directed at an entire text corpus, such as 'What are the
> main themes in the dataset?', since this is inherently a query-focused summarization
> (QFS) task, rather than an explicit retrieval task… For a class of global sensemaking
> questions over datasets in the **1 million token range**, we show that GraphRAG leads to
> substantial improvements over a conventional RAG baseline for both the
> **comprehensiveness and diversity** of generated answers."

Not accuracy. Comprehensiveness and diversity, LLM-judged, at 1M tokens.

### The LazyGraphRAG problem

Everyone cites LazyGraphRAG as the answer to GraphRAG's cost (indexing at vector-RAG cost,
0.1% of full GraphRAG, >700× lower global query cost — Microsoft Research blog,
2024-11-25). **It has never been released in the open-source library.** The maintainer
said on [discussion #1490](https://github.com/microsoft/graphrag/discussions/1490)
(2024-12-09) that "Lazy GraphRAG is indeed the next milestone for our repo"; 22 months
later there is no release, the thread has unanswered requests running into July 2026, and
the repo README now says it is "largely in maintenance mode, and won't be accepting new
PRs or implementing new features" (bug fixes continue — v3.2.0 shipped 2026-09-24).
Third-party reimplementations exist and appear in academic benchmarks, but if you budget
on LazyGraphRAG's numbers you are budgeting on software Microsoft does not ship.

---

## 3. Where graphs actually win, and by how much

Three independent 2025-26 evaluations, none vendor-authored, agree on the shape of the
answer and disagree only on margins.

### 3.1 Head-to-head under one protocol

[arXiv:2502.11371](https://arxiv.org/html/2502.11371v3) v3 (2026-03-04, KDD'26 — Michigan
State + Meta + IBM/U. Oregon), Llama 3.1-8B, identical chunking/embeddings/generation:

| Benchmark | Plain RAG | Best graph variant | Delta |
|---|---|---|---|
| Natural Questions (single-hop) F1 | **64.78** | 63.01 (Community-GraphRAG local) | **RAG wins** |
| HotpotQA F1 | 60.04 | 63.01 (HippoRAG2) | +3.0 graph |
| MultiHop-RAG accuracy | 67.02 | 70.27 (HippoRAG2) | +3.3 graph |

> "RAG excels on detailed single-hop queries"; "Global search retrieves high-level
> community summaries, which can lose fine-grained evidence and hurt detail-centric QA."

Construction time for that ~3-point multi-hop gain: **135s (RAG) vs 5,560s
(Community-GraphRAG) vs 7,702s (KG-GraphRAG)** — 41-57×.

### 3.2 GraphRAG-Bench (ICLR 2026)

[arXiv:2506.05690](https://arxiv.org/html/2506.05690v3) v3 (2026-02-22) — the abstract is
the citation: *"GraphRAG frequently underperforms vanilla RAG on many real-world tasks."*
Its four-level task ladder localizes the win:

| Task level | Winner | Novel-dataset accuracy |
|---|---|---|
| L1 Fact retrieval | **Basic RAG w/ rerank** | 60.92 vs 49.29-60.14 for the GraphRAG family |
| L2 Complex reasoning | GraphRAG | 42.93 (RAG) vs 53.38 (HippoRAG2) |
| L3 Contextual summarize | GraphRAG | 51.30 vs 64.10 |
| L4 Creative generation | GraphRAG | 38.26 vs 48.28 |

On *medical* fact retrieval the spread is brutal: Basic RAG 64.73 vs a GraphRAG family
floor of 38.63 — a badly-matched variant loses by 26 points.

### 3.3 The 2026 reframing: graph, or just more turns?

[arXiv:2604.09666](https://arxiv.org/abs/2604.09666) ("Do We Still Need GraphRAG?",
2026-04-01, NYU Shanghai) is the current state of the argument. Single-shot, GraphRAG's
multi-hop advantage is large and its general-QA advantage is noise:

| | NQ | PopQA | TriviaQA | HotpotQA | 2Wiki | MuSiQue |
|---|---|---|---|---|---|---|
| Dense (Qwen-2.5-7B) | 46.62 | 32.14 | 58.60 | 19.00 | 35.53 | 20.99 |
| GraphRAG | 48.31 | 32.82 | 57.65 | **46.70** | **62.56** | **47.95** |

> "GraphRAG yields only marginal improvements in this setting, with an average gain of
> +0.47 [general QA]… GraphRAG substantially outperforms dense retrieval on multi-hop QA,
> delivering an average improvement of +27.23."

But add agentic multi-round retrieval to the dense baseline and the multi-hop gap starts
closing — "reduced by 32.3% relative to the second-best GraphRAG variant." For an agent
that can already issue a second query and look at the result, an extra retrieval turn is
far cheaper than an extraction pipeline.

### 3.4 Even Microsoft concedes the local case

From the BenchmarkQED blog (2025-06-05) — a concession against interest, which makes it
unusually credible vendor content:

> "conventional vector-based RAG excels at local queries because the regions containing
> the answer to the query resemble the query itself… These queries tend to benefit most
> from Vector RAG's ranking of directly relevant chunks."

---

## 4. What it costs

Five independent primary-source cost tables now exist; the 2025 complaint that
quality-per-dollar was "the missing axis" has been substantially answered.

**Indexing, dollars per 1M source tokens** ([arXiv:2604.09666](https://arxiv.org/abs/2604.09666) Table 8, verified):

| Method | Construction time /1M tok | Cost /1M tok | Avg query context |
|---|---|---|---|
| LinearRAG | 0.68h | **$0** | 4,600 tok |
| HippoRAG2 | 1.19h | $2.85 | 3,229 tok |
| HypergraphRAG | 1.37h | $3.93 | 1,680 tok |
| RAPTOR | 1.70h | $6.38 | 814 tok |
| **MS GraphRAG** | 1.72h | **$13.19** | **22,160 tok** |

**Query-time prompt bloat never amortizes.** Per-query prompt tokens on the Novel dataset
([arXiv:2506.05690](https://arxiv.org/html/2506.05690v3) Tables 6/7/15): vanilla RAG 879;
HippoRAG2 1,008; RAPTOR 3,441; MS-GraphRAG local 38,707; **MS-GraphRAG global 331,375**.
That is ~80-370× vanilla RAG's per-query token cost for the win rates in §7.

**Microsoft's own Table 2** shows the escape hatch: root-level community summaries (C0)
use 2.3-2.6% of the tokens that full source-text map-reduce needs — "9x-43x fewer" — while
still winning 72% comprehensiveness / 62% diversity against vector RAG. If you ever do run
category-1 GraphRAG, run it at C0.

**The $33,000 figure everyone quotes is an extrapolation, not a bill.** It traces to
[KET-RAG](https://arxiv.org/html/2502.09304v2) (KDD'25), which measured **$21 for a 3.2MB
HotpotQA sample** with GPT-4o-mini and then wrote "processing a single 5GB legal case
incurs an *estimated* $33K." Secondary blogs have laundered the estimate into "one
practitioner indexed a 5GB legal dataset for roughly $33,000." Do not cite it as measured.

**Chunk size is the cost/recall dial and there is no free setting.** Microsoft measured
GPT-4-turbo extracting "almost twice as many entity references when the chunk size was 600
tokens than when it was 2400" — halving chunk size roughly doubles both extraction calls
and extracted entities.

---

## 5. Production reality: the verified list is short

After a full day of primary-source hunting, the organizations we can defend as running
graph-structured retrieval in production with traceable evidence number **four**: LinkedIn
(twice), ByteDance, Ant Group, and Glean (as a product, not a published result). Every one
is a self-report. **Zero independently audited production case studies exist.**

### 5.1 LinkedIn customer-service KG-RAG — the most-cited number

[arXiv:2404.17723](https://arxiv.org/abs/2404.17723), SIGIR'24. Tickets parsed into an
intra-issue tree (each section a node) plus inter-issue edges (explicit tracker links +
embedding-derived implicit ones); retrieval returns a **sub-graph**, not chunks.

| | MRR | Recall@1 | NDCG@1 | BLEU | ROUGE |
|---|---|---|---|---|---|
| Baseline (text EBR) | 0.522 | 0.400 | 0.400 | 0.057 | 0.183 |
| KG-RAG | **0.927** | **0.860** | **0.860** | **0.377** | **0.546** |

Online A/B on a randomized split of the customer-service team (Table 3): resolution time
**mean 40h → 15h, median 7h → 5h, P90 87h → 47h**. The paper headlines
**"reducing the median resolution time per issue by 28.6%."**

> **Cross-check worth recording.** The "63% reduction" figure circulating in secondary
> posts is *not* fabricated, as one of our agents initially concluded — it is the **mean**
> reduction (40h → 15h = 62.5%), present in the paper's own Table 3. The authors headline
> the median because resolution-time distributions are heavily right-skewed (P90 of 87
> hours). Quote the median; if you quote the mean, say it's the mean.

### 5.2 LinkedIn SQL Bot — the one that matters for us

[arXiv:2507.14372](https://arxiv.org/abs/2507.14372) (2025-07-18, 18 authors) +
[the 2024-12-09 blog](https://www.linkedin.com/blog/engineering/ai/practical-text-to-sql-for-data-analytics).
The graph is **of the warehouse, not of documents**: nodes are tables, columns, users, and
table clusters, fed from DataHub schemas, code repos, Trino query logs, company jargon, and
crowdsourced domain knowledge submitted through SQL Bot's own UI. Refresh is
heterogeneous — table/column and usage indexes weekly, domain knowledge instantly.
Retrieval is retrieve (K=20) → LLM rank to 7 tables → write → validate-and-fix (≤2 loops).

Benchmark: 133 questions, 167 ground-truth tables. Table recall 78%, column recall 56%,
quality (≥4) 48%, compilation success 96%. Expert review: 53% rated 4-5, 77% rated 3+.
Most common failure: **"filter is incorrect" at 24%**; incorrect joins only 4%.

**The ablation is the single most decision-relevant number in this report:**

| Configuration | Quality |
|---|---|
| Full metadata KG | **48%** |
| Bare schemas only | **9%** |
| — contribution of example queries | ~24 points |
| — contribution of table clustering | ~15 points |
| — contribution of node attributes | ~13 points |

Adoption: 300+ weekly active users, <60s latency, 33% of chat sessions end with code pasted
into the SQL editor. Published negative results: multi-query + self-consistency didn't
help; query decomposition/planning *reduced* recall; query expansion gave no gain.

> **Don't conflate their two headline numbers.** The blog's "~95% of users rated accuracy
> Passes or above" and the paper's "53% correct" describe the same system. The 95% bar is
> "Passes (queries require some modifications)"; the 53% is expert review against ground
> truth.

### 5.3 The counterexample: Uber shipped the same product with no graph

[QueryGPT](https://www.uber.com/en-CA/blog/query-gpt/) (2024-09-19): multi-agent,
**vector RAG only**. Curated per-domain *workspaces* (Mobility, Ads, Core Services), an
Intent Agent that picks the workspace, a Table Agent the user confirms, a Column Prune
Agent for token budget. ~10 min → ~3 min per query, ~300 DAU, 78% of users said it reduced
authoring time. LinkedIn's "table clustering" (worth ~15 points in their ablation) is
arguably the same idea implemented as a graph.

### 5.4 The structured-data lane, and how much of it has eroded

**data.world** ([arXiv:2311.07509](https://arxiv.org/abs/2311.07509), GRADES-NDA'24) is the
origin of the "KG triples text-to-SQL accuracy" claim — GPT-4 zero-shot over an insurance
P&C schema, SQL vs SPARQL-over-ontology:

| Quadrant | w/o KG (SQL) | w/ KG (SPARQL) |
|---|---|---|
| **All questions** | **16.7%** | **54.2%** |
| Low question / low schema | 25.5% | 71.1% |
| High question / low schema | 37.4% | 66.9% |
| Low question / **high schema** | **0%** | 35.7% |
| High question / **high schema** | **0%** | 38.5% |

The follow-up ([arXiv:2405.11706](https://arxiv.org/abs/2405.11706), 2024-05) adds **OBQC**
— an Ontology-Based Query Check that validates generated SPARQL against the ontology's
semantics — plus LLM repair, reaching **72.55%** (+8% honest "I don't know"). So the series
is 16.7% → 54.2% → 72.6%, and *the validator was worth nearly as much as the graph*.

**But dbt Labs re-ran that exact benchmark in April 2026** and the 2023 gap has largely
closed by model improvement alone
([docs.getdbt.com](https://docs.getdbt.com/blog/semantic-layer-vs-text-to-sql-2026), public
repo `dbt-labs/dbt-llm-sl-bench`):

| | 2023 (GPT-4) | 2026, original schema | 2026, modeled data |
|---|---|---|---|
| Text-to-SQL | 32.7% | 64.5% | 84.1-90.0% |
| Semantic layer | 60.5% | 72.7% | 98.2-100% |

Three findings: raw text-to-SQL roughly **doubled in 2.5 years with no architectural
change**; **data modeling bought more than the semantic layer did** (three new dbt models
moved text-to-SQL from 64.5% to 84-90%); and there is **no knowledge graph anywhere in the
2026 benchmark** — MetricFlow entities/metrics/dimensions deliver the win. Their failure-mode
line is the one to remember: *"Text-to-SQL can return plausible but incorrect answers;
Semantic Layer returns error messages instead."*

### 5.5 Everything else is thinner than it looks

| Claimed vertical | What actually exists |
|---|---|
| Healthcare | MediGRAF (Frontiers Digital Health, 2026-03-11): **10 patients**, 89 documents, explicitly not deployed |
| Biomedical | AstraZeneca BIKG is a real 10.9M-node KG — but for ML, with **no verified LLM/RAG layer** |
| Legal | ByteDance names it as a scenario (no metrics). Stanford (JELS 2025) found Lexis+ AI and Westlaw AI hallucinate **17-33%** of the time |
| Security / threat intel | Academic frameworks only; no primary source for CrowdStrike or Recorded Future running GraphRAG |
| Financial services | Demos and vendor benchmarks; no traceable production case with numbers |
| Microsoft's own customers | The Project GraphRAG page names exactly one deployment: Microsoft Discovery, internal |
| AWS Bedrock GraphRAG (GA 2025-03) | No named customer; its accuracy evidence is a **partner's self-graded** benchmark (Lettria, 50%→80%, "manually assessed… with a detailed evaluation grid," no sample sizes) |

Directionally significant despite the thin case studies: **Microsoft Fabric IQ** (ontology +
graph + data agents over OneLake, GA announced Build 2026) means the
structured-data-ontology pattern is now a first-class hyperscaler product. *(Secondhand —
we could not fetch the primary Fabric IQ page.)*

We also could not find a **single fetchable first-person "we shipped GraphRAG and here's
what went wrong" engineering post**. That absence is itself a finding.

---

## 6. Failure modes, ranked

Ranked by likelihood × damage, each with its evidence.

**1. Entity duplication / resolution failure.** The strongest number in this section:
[arXiv:2510.14271](https://arxiv.org/html/2510.14271v1) ("Less is More: Denoising Knowledge
Graphs for RAG", 2025-10) applied entity resolution to LLM-built KGs and removed **~40% of
entities and 30-60% of relations across four datasets — while *improving* QA** (Agriculture
win rate 42.4→57.6, CS 41.6→58.4, Legal 42.4→51.6). Their example of the redundancy: "LLMs"
co-occurring with "LLM", "llms", "Large Language Models", and "modelos de lenguaje grandes".
The paper also notes it is "the first comprehensive exploration of entity resolution in
LLM-generated KGs" — as of late 2025 nobody had systematically studied the step
practitioners call the hardest.

Typing is as broken as identity: one practitioner index had 1,123 entities (12.7%) assigned
2+ types, one entity getting 7. *(secondhand; the article's stated date is implausible.)*
And a schema doesn't save you — *"Schema can constrain the LLM to 'only pick from these
types,' but can't constrain it to 'pick only one for the same entity.'"*

Errors compound per hop: at 95% per-entity resolution accuracy a 5-hop answer is ~77%
accurate; at 85%, ~44%. *(This is arithmetic under an independence assumption, not a
measurement — use it as an illustration.)*

**2. Paying full indexing cost for a query mix that doesn't need it.** §3 + §4: real query
mixes are dominated by single-hop lookups where plain RAG ties or wins, so the 41× build
and 40× per-query prompt buy nothing on most traffic.

**3. Staleness and re-indexing.** Microsoft's own tracking
[issue #741](https://github.com/microsoft/graphrag/issues/741) frames the hard part as
community drift: "If certain thresholds are met, recompute may be required, so the worst
case degrades to the same performance as a normal indexing." Worse,
[issue #2540](https://github.com/microsoft/graphrag/issues/2540) — opened **2026-09-03,
still open, no maintainer response** — reports that `graphrag update`'s entity-merge
remapping is applied only in `update_text_units.py`, so "the community merge never receives
it, so delta communities are concatenated with their `entity_ids` and `relationship_ids`
still pointing at the discarded ids." That is silent corruption of exactly the structure
global search depends on. The CLI also writes to a separate `update_output` folder — you
own index versioning.

**4. Global search losing fine-grained evidence.** Microsoft and independent evaluators
agree (§3.1, §3.4).

**5. Query-time prompt bloat and latency.** 4×10⁴-token prompts (GraphRAG-Bench),
seconds-to-minutes global queries. Never amortizes.

**6. Extraction quality drift.** Chunk-size changes silently halve or double entity yield;
domain prompt tuning is mandatory enough that Microsoft ships an auto-tuner; a silent
extraction regression corrupts the graph for weeks before anyone notices.

**7. Evaluation self-deception.** See §7 — this is arguably #1 in disguise.

**8. Poisoning via shared relations.** [GragPoison](https://arxiv.org/abs/2501.14050)
("GraphRAG under Fire", rev 2025-10) achieves "up to 98% success rate" using "less than 68%
poisoning text" compared to prior attacks, by exploiting shared relations. Relevant to any
multi-tenant or user-content corpus: one poisoned document contaminates every path through
the entities it touches.

**9. Premature graph-database adoption.** Microsoft GraphRAG itself outputs Parquet and
needs no graph DB. For the 1-2 hop lookups that dominate GraphRAG access patterns, Postgres
(± Apache AGE) is competitive; graph DBs earn their keep at 3+ hops. From
[HN's "graph databases for agentic use-cases"](https://news.ycombinator.com/item?id=45436010)
thread: *"Graph databases are one of those things that sound neat but you'll be hard
pressed to find people using them that don't regret it"*; a founder who abandoned Neo4j:
*"agents are smart enough to traverse a database in a graph like manner if you provide them
with the right tooling."* Where it did pay for them: cloud security posture (EC2 → security
group → IAM role), genuinely multi-hop and genuinely relational.

---

## 7. How GraphRAG gets evaluated

This is the section the question was really about. General eval theory (golden sets,
LLM-as-judge, online vs offline) lives in [`evals.md`](./evals.md); what follows is
graph-specific, and the short version is that **most published GraphRAG wins were measured
with a protocol that cannot distinguish a real gain from a formatting artifact.**

### 7.1 Microsoft's original protocol, and what it actually claims

[arXiv:2404.16130](https://arxiv.org/abs/2404.16130) v2, §3.3, verbatim:

> "Given the lack of gold standard answers to our activity-based sensemaking questions, we
> adopt the head-to-head comparison approach using an LLM evaluator that judges relative
> performance according to specific criteria."

Four criteria — **comprehensiveness, diversity, empowerment**, and **directness** as a
deliberate *control*:

> "We include it to behave as a reference against which we can judge the soundness of
> results for the other criteria. Since directness is effectively in opposition to
> comprehensiveness and diversity, we would not expect any method to win across all four
> criteria."

That control-criterion idea is genuinely good practice and worth stealing regardless of
what you think of the rest.

Scoring: win 100 / tie 50 / loss 0, averaged over 5 runs, Wilcoxon signed-rank with
Holm-Bonferroni correction. Questions generated by an LLM from a *corpus description* via
personas × tasks × questions, K=M=N=5 → 125 questions.

**The judge prompt, copyable** (Appendix F.1):

```
---Role---
You are a helpful assistant responsible for grading two answers to a question that are
provided by two different people.

---Goal---
Given a question and two answers (Answer 1 and Answer 2), assess which answer is better
according to the following measure:

{criteria}

Your assessment should include two parts:
- Winner: either 1 (if Answer 1 is better) and 2 (if Answer 2 is better) or 0 if they are
  fundamentally similar and the differences are immaterial.
- Reasoning: a short explanation of why you chose the winner with respect to the measure
  described above.

Format your response as a JSON object with the following structure:
{{
    "winner": <1, 2, or 0>,
    "reasoning": "Answer 1 is better because <your reasoning>."
}}
```

Note what is **absent**: no instruction to ignore length, no instruction to ignore position,
no order swapping. That absence is the entire basis of §7.2.

**What the famous number actually says.** "72-83% comprehensiveness win rate" is *global
approaches vs vector RAG*. Against **TS** — map-reduce over source text, i.e. global
summarization with **no graph at all** — GraphRAG's community summaries win only **56-64%**,
and the paper says so: "community summaries generally provided a small but consistent
improvement… except for root-level summaries." Two further corrections that get dropped in
retelling:

- **Empowerment was mixed, not a win.** Root-level GraphRAG (C0) *loses* empowerment to
  vector RAG on both datasets, 43 vs 57.
- **Vector RAG won directness on every comparison**, as the control predicted (SS beats C0
  at 65/35 on podcasts).

**Microsoft's own ground-truth check (v2, Experiment 2).** They extracted atomic claims
(Claimify) and defined comprehensiveness = mean unique claims, diversity = mean
agglomerative clusters of those claims. Result: 47,075 claims, ~31/answer; all global
methods beat vector RAG significantly (News: C0 34.18 claims vs SS 25.23). The agreement
number is the one to quote: after majority-voting the 5 judgments, only **33% / 39%** of
pairwise comparisons had a non-tie majority at all, and on those the judge matched the
claim-based label **78%** (comprehensiveness) and **69-70%** (diversity) of the time. So
judge and objective metric disagree roughly 1 time in 4, on the subset where the judge was
confident.

### 7.2 The debiasing critique — the most useful citation in this report

Zeng et al., *How Significant Are the Real Performance Gains? An Unbiased Evaluation
Framework for GraphRAG*, [arXiv:2506.06331](https://arxiv.org/abs/2506.06331) v2
(2026-08-13). They attack both halves of the protocol.

**Flaw 1 — the questions are unanswerable.** "the LLM is only provided with vague summaries
of the passages… as a result, the questions do not involve the fine-grained details of the
passages." Their Harry Potter example: *"What quantitative methods can predict box office
success from dataset analysis?"*

**Flaw 2 — three measured biases:**

| Bias | How measured | Result |
|---|---|---|
| **Position** | LightRAG vs itself, AB then BA | "LLMs favor the answer that comes in the front, and the win rates can differ by over 30%" |
| **Length** | Same system, position controlled, length gap varied | "a length gap of 25 tokens, which is relatively small compared with the average answer length of 200 tokens, can lead to a win rate gap of over 50%… adding meaningless tokens to an answer may improve its chances of winning" |
| **Trial** | NaiveRAG vs FGRAG, 5 identical runs | ties, wins, and losses all occur — "With a single trial, we may arrive at any of the 3 possible conclusions" |

**The sanity check that should settle the argument:**

> "We use the same model (i.e., LightRAG) to generate two sets of answers for the same
> question set… instance A achieves a win rate of 90%, while instance B only reaches a win
> rate of 10%… This translates to an 80% advantage for A and falsely suggests significant
> performance gains."

**What changes once you debias:** LightRAG's reported 66.70% win over NaiveRAG becomes
**39.06%** — a loss. Final ranking flips to FGRAG > MGRAG > NaiveRAG > LightRAG, and
"with the exception of FGRAG and MGRAG, which improve LightRAG by 21% and 11%… the relative
win rates between the other methods are all below 8%," with tie rates above 50% on many
per-aspect breakdowns.

**Their protocol, copyable:**

- **Length alignment** by "generate-adjust": re-issue the shorter system's *prompt* with a
  target length (not post-hoc padding — "this gives the LLM more room to re-organize the
  retrieved contexts"), 10-word tolerance; **succeeds on 85% of pairs, discard the rest**.
- **Position exchange**: score AB and BA, average per answer.
- **Trial statistics**: N=2 per position × M=25 trials, report a **box plot**, not a point.
- **Replace Diversity with Relevance** — "diversity is not effective when evaluating a
  single answer; instead, the relevance of an answer to the question is more indicative."
- **Absolute 0-5 rubrics instead of pick-a-winner.** Their directness rubric verbatim:

```
0: The answer is extremely indirect, failing to address the question specifically and clearly.
1: The answer is indirect and deviates significantly from the question, making it hard to discern the intended response.
2: The answer is somewhat indirect, occasionally straying from the question, but still touching on relevant points.
3: The answer is moderately direct, addressing the question with some clarity but could be more specific and focused.
4: The answer is clear and direct, effectively addressing the question with specificity and clarity.
5: The answer is exceptionally direct, precisely and specifically addressing the question without any ambiguity.
```

- **Allow ties**, and report `Relative Win Rate = (A_win − B_win) / (A_win + B_win + Tie)`.
- **Generate questions from the graph, not a summary**: sample a node, an edge, or a
  random-walk subgraph (≥50 non-repeating hops) and feed the structure *plus the original
  source chunk* to the LLM — "To prevent the quality of the knowledge graph itself from
  being strongly correlated with the quality of the problem." Validation: 150 summary-based
  questions covered **11 distinct entities** (53.3% clustered on one minor character); 150
  graph-grounded ones covered **64**. A 40-participant study rated graph-grounded questions
  ~4.4/5 on comprehensibility, specificity, and answerability vs ~3.6/3.4/2.9.

Independent confirmation of position bias from [arXiv:2502.11371](https://arxiv.org/html/2502.11371v3) v3:
*"position bias is clearly present in LLM-as-a-Judge evaluations for summarization, as
reversing the order of presented summaries leads to substantially different, and in some
cases opposite, judgments."* Their reference-grounded (ROUGE/BERTScore) evaluation **flips
the comprehensiveness direction** — RAG is consistently preferred on comprehensiveness,
GraphRAG only on diversity.

### 7.3 BenchmarkQED: Microsoft's own correction

[Microsoft Research, 2025-06-05](https://www.microsoft.com/en-us/research/blog/benchmarkqed-automated-benchmarking-of-rag-systems/)
+ [github.com/microsoft/benchmark-qed](https://github.com/microsoft/benchmark-qed). The
current pairwise system prompt is effectively an admission:

```
---Important Guidelines---
- No position biases: The order in which the answers are presented should NOT influence your judgment.
- Ignore length: Do NOT let the length of the answers affect your evaluation.
- Ignore formatting style: Do NOT consider the structure or presentation (e.g., bullet points
  vs. paragraphs) when judging answer quality. Focus only on the content.
```

Plus a forced three-step process (identify claims → extract supporting evidence → compare),
**Directness replaced by Relevance**, and counterbalancing enforced in code
(`benchmark_qed/autoe/config.py`: `trials: int = 4` with a validator — *"The number of
trials must be even to allow for counterbalancing of conditions"*).

**AutoQ's query taxonomy is worth stealing wholesale**: source (data-driven vs
activity-driven) × scope (local vs global) → DataLocal, DataGlobal, ActivityLocal,
ActivityGlobal. Sampling your eval set across that 2×2 forces you to notice when you've only
tested one corner.

Note where LazyGraphRAG failed even on Microsoft's own harness: *"failing to reach
significance only for the relevance of answers to **DataLocal** queries"* — the class
closest to a text-to-SQL lookup.

### 7.4 Benchmarks, and which ones are cheatable

**HotpotQA and 2WikiMultiHopQA are weak GraphRAG benchmarks.** The DiRe ("disconnected
reasoning") score from [MuSiQue](https://arxiv.org/abs/2108.00573) (TACL 2022) measures how
much of a multi-hop benchmark is reachable *without* multi-hop reasoning:

| | HotpotQA-20K | 2WikiMultiHopQA-20K | MuSiQue-Ans |
|---|---|---|---|
| Human | 84.5 | 83.2 | 78.0 |
| Single-paragraph model | 64.8 | 60.1 | **32.0** |
| **DiRe score (answer)** | **68.8** | **63.4** | **37.8** |
| DiRe score (support) | 93.0 | 98.5 | 63.4 |

> "even disconnected reasoning (bypassing reasoning steps) can achieve such high scores. In
> contrast, this number is significantly lower (37.8) for MuSiQue-Ans."

If you benchmark on HotpotQA and see a graph gain, ~63-69 points of headroom were reachable
with no hop at all. **Use MuSiQue.**

**MultiHop-RAG** ([arXiv:2401.15391](https://arxiv.org/abs/2401.15391)) contributes the
query type everyone forgets: **Null** — "the answer cannot be derived from the retrieved
set… an LLM should produce a null response instead of hallucinating." Metrics MAP@K, MRR@K,
Hit@K. (Connective fact: its news corpus *is* the "News articles" dataset in Microsoft's
GraphRAG paper — the most-cited GraphRAG result and the most-cited multi-hop RAG benchmark
share a corpus.)

**Two different papers are both called GraphRAG-Bench**, both June 2025 — 
[arXiv:2506.02404](https://arxiv.org/abs/2506.02404) (HK PolyU + Tencent, 1,018 college-level
questions, notable for scoring **Rationale Accuracy** against a gold rationale) and
[arXiv:2506.05690](https://arxiv.org/html/2506.05690v3) (ICLR'26, the leaderboard one).
Cite the arXiv ID, not the name.

**WildGraphBench** ([arXiv:2602.02053](https://arxiv.org/abs/2602.02053), 2026-02) is the
best-designed recent one: ground truth = citation-linked Wikipedia statements, corpus = the
actual cited external pages crawled raw (boilerplate kept). Multi-fact questions pass a
strict filter — an LLM judge must confirm no single reference suffices. Its statement-level
metrics are directly copyable:

```
Recall    = (1/|S*|) Σ_{s∈S*}  max_{ŝ∈Ŝ} Match(s, ŝ)
Precision = (1/|Ŝ|)  Σ_{ŝ∈Ŝ}  max_{s∈S*} Match(s, ŝ)
```

Its result table is a useful reality check: NaiveRAG 59.79 avg accuracy, HippoRAG2 64.33,
MS GraphRAG local **38.23**, and **human 85.66**.

### 7.5 Grading the graph itself

[arXiv:2506.05690](https://arxiv.org/html/2506.05690v3) §3.3 gives the only peer-reviewed
structural metrics with formulas: **node count**, **edge count**, **average degree**
(`(1/|V|) Σ deg(v)`), and **average clustering coefficient**
(`C(v) = 2·T(v)/(deg(v)·(deg(v)−1))`). WildGraphBench adds **isolated-node fraction** —
"a more organized graph should avoid excessive isolated nodes, because graph-based retrieval
relies on connectivity."

Same corpus, wildly different graphs:

| Novel dataset | MS-GraphRAG | HippoRAG2 | LightRAG | Fast-GraphRAG | HippoRAG |
|---|---|---|---|---|---|
| Avg degree | 1.48 | **8.75** | 2.10 | 3.19 | 1.73 |
| Avg clustering coeff | 0.315 | **0.657** | 0.212 | 0.324 | 0.100 |

And density tracks downstream quality: "HippoRAG2 produces significantly denser graphs…
This enhanced graph density improves both information connectivity and coverage… consistent
with the retrieval performance."

**Caveat to state loudly: these are proxies. None checks whether a triple is true. A
hallucinated relation raises average degree.** Which leads to two real gaps:

- **Extraction precision/recall against a gold KG is not measured anywhere in the GraphRAG
  literature.** We looked. The circulating "98.82% precision / 93.18% recall" numbers come
  from domain-specific KG-construction papers, not GraphRAG evaluations. The reportable
  finding is the absence: the field measures graph *shape* and downstream answers, never
  extraction accuracy.
- **No standard metric exists for community-summary quality.** The nearest instruments are
  an indirect A/B against RAPTOR-style chunk summaries
  ([arXiv:2503.04338](https://arxiv.org/abs/2503.04338) found "community reports serve as
  more effective high-level information than the chunk summaries used in RAPTOR"), and
  Microsoft's own in-prompt self-score: *"generate an integer score between 0-100 that
  indicates how helpful is this response"* — a map-stage filter, but a signal you can log.

For entity resolution, track **both directions separately**: pairwise P/R/F1 hides the two
failures that matter — under-merging (one entity as three nodes ⇒ paths break) and
over-merging (two orgs collapsed ⇒ false paths). Add a cluster-level metric (B³ or cluster
P/R). *(ER metric definitions: standard, secondhand.)*

### 7.6 Retrieval metrics — and the gap in every mainstream framework

GraphRAG-Bench's retrieval formulas (Appendix F), copyable:

```
CONTEXT RELEVANCE = (1/|C|) Σ_{c∈C} R(c, Q, E)      # C = retrieved contexts, E = evidence
EVIDENCE RECALL   = (1/|R|) Σ_{c∈R} 1(S(c, C))      # R = reference claims
FAITHFULNESS      = |{c ∈ A : S(c, C)}| / |A|       # A = claims in the response
EVIDENCE COVERAGE = |{e ∈ E : M(e, G)}| / |E|       # G = generated answer
ANSWER ACCURACY   = α·FC + (1−α)·SS,  α = 0.75      # FC = statement-level F1, SS = cosine
```

**The single most decision-relevant measurement in this whole report**: MS-GraphRAG on the
Medical dataset scores **Context Relevance 5.67 / 4.25 / 5.24 / 2.76** across the four task
levels — its retrieved context is >94% irrelevant by this measure — while its *evidence
recall* stays respectable. Recall goes up, precision collapses. **If you deploy a graph
retriever and only measure recall, you will not see this.**

**BenchmarkQED ships two graph-aware retrieval metrics nobody else has:**

- **Cluster-level recall** — map each text unit to a topic cluster; recall = fraction of
  reference-relevant *clusters* the retrieved set touched. Catches "grounded, but only
  covers one corner of the corpus."
- **Fidelity** — Jensen-Shannon divergence / total-variation distance between the
  distribution of retrieved relevant units across clusters and the reference distribution.
  The closest thing in existence to "did you cover the communities proportionally."
- Plus a 0-3 Bing-style chunk-relevance judge (binary precision at threshold 2, reported
  macro- and micro-averaged) and a binary assertion scorer.

**And the framework gap:** as of September 2026, **no mainstream RAG evaluation framework
ships a graph-native retrieval metric.** They all treat graph-derived context as a bag of
text.

| Framework | Graph-native metrics | Nearest useful thing |
|---|---|---|
| **RAGAS** | No | **Context Entities Recall** = `|RCE ∩ RE| / |RE|`; Context Precision@K with LLM and non-LLM (Levenshtein) variants; **an SQL category with query-equivalence and execution-based metrics** |
| **RAGAS testset generation** | Yes, on the *generation* side | Builds an actual `KnowledgeGraph` (extractors → `JaccardSimilarityBuilder`/`OverlapScoreBuilder` → `apply_transforms()`), then synthesizers traverse it: `find_two_nodes_single_rel(...)`. The mainstream version of "generate multi-hop questions from the graph." |
| **DeepEval** | No | Five RAG metrics; strongest CI/CD integration |
| **RAGChecker** (Amazon, NeurIPS'24 D&B) | No, but claim-level granularity is right | Claim Recall, Context Precision, **Relevant/Irrelevant Noise Sensitivity**, Hallucination, Context Utilization — the noise split is what would have caught the 5% context relevance above |
| TruLens / Phoenix / LangSmith | None found *(secondhand)* | Groundedness / context relevance / traces |

### 7.7 What production teams actually report

LinkedIn (§5.1) publishes retrieval IR metrics + BLEU/ROUGE/METEOR + one operational KPI
from a randomized A/B. data.world (§5.4) publishes execution accuracy over N runs,
stratified 2×2. **Nobody — LinkedIn, data.world, Microsoft, any vendor — publishes
graph-construction quality, entity-resolution accuracy, or graph drift over time.**

data.world's metric ladder is the closest prior art to our own harness and worth quoting
exactly:

- **Execution Accuracy (EA)** — "An execution is accurate if the result of the query matches
  the answer for the query. Note that the order or the labels of the columns are not taken
  in account."
- **Overall Execution Accuracy (OEA)** — "Given the non-deterministic nature of LLMs, there
  is no guarantee that given an input question, the generated query will always be the
  same… every question has a OEA score which is calculated as (# of EA)/Total Number of
  runs."
- **AOEA** — mean OEA over a question set or quadrant.

Two design ideas to steal directly: **run every question N times and score the fraction that
passes** (non-determinism is data, not noise), and **stratify by schema complexity
separately from question complexity** — the two produce completely different failure
profiles, and averaging them hid two 0% quadrants.

### 7.8 How to evaluate a GraphRAG system — build order

Each step tied to a source you can hand a skeptical engineer.

1. **Golden set with labeled *evidence*, not just answers.** Gold answer + gold supporting
   units + difficulty label. Stratify by reasoning complexity **and** schema complexity
   independently. *(data.world Table 1; LinkedIn SIGIR'24 §4.1)*
2. **Measure retrieval before answers.** MRR, Recall@K, NDCG@K, Hit@K against labeled
   evidence — cheapest, most stable, least gameable, and where LinkedIn's real gain showed
   up (0.522 → 0.927). *(LinkedIn Tables 1-2; MultiHop-RAG §2.3)*
3. **Generate questions from your own graph, then validate the questions.** Node/edge/
   subgraph sampling with the source chunk attached (Zeng et al.); or BenchmarkQED's
   bridge/comparison/intersection/temporal prompts with built-in 1-5 self-scoring; or RAGAS
   TestsetGenerator off the shelf. **Do not** use persona×task summary-based generation as
   your only source — 53% of those questions clustered on one minor entity.
4. **Score against ground truth before running any win rate.** Contain-EM and token-F1 for
   short answers; statement-level P/R/F1 for long ones; **execution accuracy** for
   text-to-SQL. N runs per question, report the pass fraction.
5. **Claim-level diagnostics to separate retriever from generator failure.** Faithfulness,
   evidence coverage, noise sensitivity — and **context relevance, not just recall**
   (83% recall alongside 5% context relevance is a real observed combination).
6. **Put the graph under regression test.** Node/edge count, average degree, clustering
   coefficient, isolated-node fraction on every re-index; alert on drift. Proxies, but they
   catch extraction-prompt regressions and model swaps for free. Track ER health separately,
   both directions.
7. **Only now run pairwise LLM-judge, and only with all four controls**: counterbalance
   order and average; even trial count (BenchmarkQED enforces 4); length alignment by
   generate-adjust (discard the ~15% that won't align); ties allowed and a distribution
   reported, not a point estimate. Use comprehensiveness/diversity/empowerment/**relevance**
   with the debiasing guidelines block, prefer 0-5 absolute rubrics, and keep a control
   criterion you *expect to lose*. **Sanity-check the harness by running your system against
   itself — if it isn't ~50/50, the harness is broken.**
8. **Validate the judge against something objective** (claim counts, statement F1) and
   report the agreement rate. Microsoft's own is 78% / 69-70%, on the 33-39% of comparisons
   where the judge reached a majority.
9. **Report cost and latency in the same table as quality, always.** A 72% win rate at 330k
   tokens/query is a different product decision than the same win rate at 4k.
10. **Close the loop online with one operational KPI** — LinkedIn's randomized team split is
    the template.
11. **Keep a null / unanswerable slice.** A system that never abstains scores well on
    everything else.

---

## 8. Decision rules and hybrid patterns

**Use a graph when:** queries are genuinely multi-hop or global/thematic; the corpus has
real cross-document entity structure; and you can amortize the index. **Don't when:** any of
the nine conditions below holds.

1. **Your queries are mostly single-hop factual lookups.** Basic RAG w/ rerank beat every
   GraphRAG variant on GraphRAG-Bench fact retrieval (60.92 vs 49.29-60.14) and beat
   Community-GraphRAG on NQ (64.78 vs 63.01 F1).
2. **You haven't shipped and measured a hybrid vector+BM25+reranker baseline.** Practitioners
   converge on instrument-first; reranking is repeatedly cited as the highest-ROI addition
   *(exact percentages unverified)*.
3. **Your corpus changes incrementally.** Community drift can force full recomputation;
   `graphrag update` has an open correctness bug and non-in-place semantics.
4. **You have interactive latency budgets.** 4×10⁴-token prompts, seconds-to-minutes global
   queries.
5. **Your corpus is small or flat.** Community hierarchy buys nothing with no cross-document
   themes.
6. **You can't fund ongoing entity resolution.** ~40% redundancy is the default state of an
   LLM-built KG and it is not self-correcting.
7. **Your corpus contains user-supplied or multi-tenant content without provenance
   controls.** 98% poisoning success via shared relations.
8. **Your success metric is only measurable by an LLM judge.** If you can't score with
   F1/EM/execution accuracy, you can't distinguish a gain from a 30-point position artifact.
9. **You're about to stand up Neo4j for 1-2 hop lookups.** Postgres, or plain Parquet, is the
   cheaper default.

**Hybrid patterns practitioners actually recommend** (Towards Data Science, 2026-09-20;
VentureBeat, 2026-08-02 — opinion-grade, ordered by latency):

1. **Text-to-Cypher / query generation** — "The LLM does not guess the relationships; the KG
   already has them."
2. **Parallel hybrid** — vector and graph simultaneously, merge.
3. **Sequential graph-first** — graph finds entities, doc IDs filter the vector search.
4. **Sequential vector-first ("graph as expander")** — semantic search finds chunks,
   entities in those chunks seed traversal. Best for fuzzy intent.
5. **Adaptive router agent** — only send global/thematic questions to the graph; trades
   router overhead and misclassification risk for cost.
6. **Agentic GraphRAG (ReAct over both stores)** — "incurs significant latency measured in
   minutes. Reserved for offline analytical queries only."

---

## 9. What we could not verify

**Actively debunked or distorted:**

- **"Microsoft GraphRAG moved accuracy from 16.7% to 56.2%, a 3.4× improvement."** A
  laundered composite: 16.7% → 54.2% is **data.world's** SQL-vs-SPARQL result; 56.2% and
  "3.4x" come from **Diffbot's** KG-LM benchmark. Neither is a Microsoft GraphRAG result.
- **"$33,000 to index 5GB."** An explicit linear *estimate* inside the KET-RAG paper
  (measured datum: $21 for 3.2MB), re-told by secondary sources as a practitioner's invoice.
- **"AWS Quick Suite's Conversation Service, architected with GraphRAG, serves 310,000+
  accounts."** Attributed to a Neo4j NODES 2025 talk; fetching that talk page shows a
  different speaker on a different topic, with no mention of GraphRAG or Quick Suite.
- **"Snowflake: BIRD 57% → 78% just by adding a semantic model."** Not present at any
  Snowflake primary source.
- **"LinkedIn: 40h → 15h, a 63% improvement."** *Not* fabricated — see §5.1. It is the
  paper's **mean**; the headline median reduction is 28.6%. Label whichever you use.
- **METEOR "0.372 vs 0.145-0.263"** attributed to the ORAN benchmark — that paper reports no
  METEOR. Discard.

**Vendor numbers with no independent replication:** LazyGraphRAG's 0.1% indexing / 700×
query-cost claims (one corpus, LLM-judged, self-benchmarked); Lettria's 50%→80% (vendor
self-graded, no sample sizes, reused by AWS); Writer's 86.31% RobustQA (self-run against
competitor configs Writer tuned); FalkorDB's ">90% accuracy" / "90% hallucination
reduction"; Neo4j/IDC's "$4M annual value, 7.8-month payback"; "GraphRAG delivers 300-320%
ROI"; "72-80% of enterprise RAG implementations fail to reach production" (no methodology
anywhere).

**Unverified but directionally supported:** the "hybrid beats the best single baseline by
6.4%" figure from arXiv:2502.11371 — one agent reported it as verified, another could not
find it in the fetched text, so treat it as unconfirmed; "up to 3× more accurate than raw
SQL schema" for KG-as-semantic-layer text-to-SQL (an individual author's unpublished
benchmark in a meetup repo — flagged loudly because it is the most Dalgo-adjacent claim in
circulation); cross-encoder reranking "33-40% at 120ms p50"; global-search latencies of
110-281s.

**Gaps rather than errors:** we could not fetch the Azure AI Search + GraphRAG blog body or
the Microsoft Fabric IQ primary page (both secondhand); BenchmarkQED has no arXiv paper we
could locate (blog + repo only; judge model quoted as GPT-4.1, blog used 6 trials, the
shipped default is 4); no paper anywhere defines a direct community-summary quality metric
(absence of evidence, and we found none); extraction precision/recall is unmeasured across
the GraphRAG literature; and we found **no fetchable first-person production post-mortem**
(the most promising candidate, Brian Godsey's "The Quest for Production-Quality Graph RAG",
returned HTTP 403).

**Methodological note:** our evals agent found that a summarizing fetch **fabricated table
values** from the Microsoft paper (claiming all directness win rates = 50% when the real
matrix runs 54-65%) and switched to downloading PDFs and extracting text with pypdf. Every
numeric table in §7 came through raw extraction. Treat single-model page summaries of
numeric tables as unreliable.

---

## 10. What this means for Dalgo

Mapped against `dalgo-core/features/chat-with-data/v3/plan.md` (table cards + value profiles + full
schema injection, no embeddings, no metric layer) and our eval harness in
`ddpui/core/ai/evals/`.

**The headline: nothing in categories 1-3 of §2 belongs in our roadmap.** Our "corpus" is a
database schema plus modest docs and example queries — small, already structured, already
relational. Building an LLM-extracted entity graph over it would incur every failure mode in
§6 in order to synthesize relationships **that already exist as declared foreign keys**. A
schema graph read from `information_schema` is deterministic, free, never stale, and has no
entity-resolution problem. That is not GraphRAG; don't let the label import GraphRAG's cost
structure.

**What the graph literature does validate, and it's the semantic-layer thesis again:**

- **LinkedIn's 48% → 9% ablation is the strongest argument for curated metadata we have
  found anywhere**, and it decomposes exactly onto our v3 plan: example queries ~24 points,
  table clustering ~15, node attributes ~13. [`semantic-layer.md`](./semantic-layer.md) §10
  already recommends adding a verified-queries slice on the strength of the +18 figure from
  LinkedIn's blog; the paper's ablation is the harder version of the same evidence. **This
  raises verified queries from "plan amendment" to the highest-value unbuilt item.**
- **Whether that metadata lives in a graph DB is, on this evidence, an implementation
  detail nobody has shown to matter.** Uber hit comparable production outcomes with curated
  workspaces + vector RAG and no graph. Postgres is the right store.

**Three concrete, cheap things worth considering, in order of evidence strength:**

1. **An ontology-based query validator, our version of OBQC.** data.world's series is
   16.7% → 54.2% (graph) → **72.6%** (validator + repair), so the validator delivered nearly
   as much as the graph and needs none of it. It targets exactly the dominant failure both
   LinkedIn ("filter is incorrect," 24% of errors) and dbt ("plausible but incorrect
   answers") identify. For us that means checking generated SQL against declared semantics —
   join validity against FKs, filter values against the value profiles M1 is already
   building, aggregation compatibility — **before** execution, then feeding violations back
   into the existing SQL-retry loop. Note the shape of the win: it converts silent wrong
   answers into loud failures, which is the same trade `chat-with-data-evals.md` L5 made
   with absence questions.
2. **A foreign-key join-path graph for schema linking.** SchemaGraphSQL
   ([arXiv:2505.18363](https://arxiv.org/abs/2505.18363), EACL'26 Findings) builds the graph
   purely from FK constraints, extracts source and destination tables with one LLM call, then
   runs **classical path-finding** — training-free, claims SOTA schema linking on BIRD. This
   is Dijkstra over `information_schema`, not GraphRAG. Caveat: `semantic-layer.md` §8 is
   unambiguous that at dozens of tables we should ship the full enriched schema rather than
   invest in retrieval sophistication, and LinkedIn's incorrect-join rate was only 4%. So
   this is a **watch item for when an org's warehouse outgrows full injection**, not an M1
   task.
3. **Glossary-term → column links mined from query logs.** A small, curated,
   hand-verifiable graph — the "small accurate graph beats large noisy one" case. The
   natural producer is the `--describe` pass plus promotion from router/audit vocabulary
   mismatches already sketched in `semantic-layer.md` §10 amendment 4.

**What §7 changes about our eval harness** — this is where the research pays off fastest,
because we already run gates-and-judges:

- **Our architecture is already the recommended one.** §7.8's build order is ground-truth
  first, LLM-judge last and only with controls; that is exactly `chat-with-data-evals.md`
  L3 ("gates block, judges inform — and this must be *measured*"). The 8-of-11
  judge-vs-execution disagreement we measured is the same phenomenon Zeng et al. measured at
  scale. **We got this right; the literature is catching up to it.**
- **Two controls we are missing.** We never run the **self-comparison sanity check** (score
  our system against itself — anything far from 50/50 means the harness is broken), and our
  judges score single answers without **length or position controls**. The first is nearly
  free and belongs in the offline scripted-model unit tests. The second matters whenever we
  compare two agent versions head-to-head, which is exactly what we'll want when the v3
  semantic layer lands.
- **Stratify the golden set by schema complexity, separately from question complexity.**
  data.world's 2×2 hid two 0% quadrants inside a 16.7% average. Our datasets are stratified
  by nothing right now. This is a JSONL field and a report grouping — cheap, and it would
  tell us whether our failures are model failures or schema-shape failures.
- **Run every item N times and score the pass fraction (OEA), not pass/fail.** We already
  know from L6 that flaky items are findings, not noise; OEA turns that observation into a
  number instead of a note. Cost-wise this fits our existing gradient (L9) if N applies only
  to the `canary` tag.
- **Add a table-selection score now.** `expected_tables` is already in our schema and
  unscored; LinkedIn reports table recall (78%) and column recall (56%) as their primary
  retrieval metrics and their gain showed up there first. §7.8 step 2 says measure retrieval
  before answers — we currently do the opposite.
- **Keep a null / unanswerable slice.** MultiHop-RAG's Null type is our
  `answer_expectations` lane (L5) — good, and worth naming as a deliberate category rather
  than an exception.

**One thing to stop citing.** If GraphRAG comes up in a funder or partner conversation, the
honest summary is: it is a narrow tool for multi-hop sensemaking over documents, its
production evidence is four self-reports, its headline numbers were measured with a protocol
that returns 90/10 when a system is compared to itself, and the part of it that would help
us — curated metadata over a warehouse schema — needs no graph database and no LLM
extraction at all.
