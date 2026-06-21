# Evals and What Wins: How Teams Measure Harnesses, and Why the Harness Beats the Model

> Prepared: June 2026
> Audience: builders who already know *what* a harness is (see `research/harness-engineering.md`) and *how* to build an eval (see `research/evals.md`), and now want the **benchmark/measurement** complement: which benchmarks exist, how harness-sensitive each is, what actually moves scores, and how to run your own ablations.

**Central thesis:** At today's model capability, *the harness is usually the binding constraint, not the model.* The cleanest proof is the same model scoring wildly differently when only the scaffold changes. This document assembles that evidence with citations.

A note on sourcing discipline (matching the existing research docs): numbers below are tagged **[primary]** when pulled from the source's own page/paper, **[reported]** when from a vendor announcement relayed by coverage, and **[aggregator]** when they come from a 2026 benchmark-tracking content site that should be treated as a pointer, not gospel. The thesis rests entirely on **[primary]** evidence; aggregator numbers are color, not load-bearing.

---

## Table of Contents

1. [The coding-agent benchmarks — what each measures, how harness-sensitive](#1-the-coding-agent-benchmarks)
2. [What moves scores — harness, not model (the evidence table)](#2-what-moves-scores--harness-not-model)
3. [Verification: the highest-leverage component for the premature-success failure mode](#3-verification-the-highest-leverage-component-for-one-failure-mode)
4. [The benchmark-to-production gap](#4-the-benchmark-to-production-gap)
5. [How to run harness ablations (the recipe)](#5-how-to-run-harness-ablations-a-recipe)
6. [Cost and latency as harness metrics](#6-cost-and-latency-as-harness-metrics)
7. [Sources](#7-sources)

---

## 1. The coding-agent benchmarks

A coding-agent benchmark score is never a property of a model alone. It is a property of `model + harness + benchmark`. The leaderboard tables below should be read as "this *system* scored Y," not "this *model* scored Y." The single most important literacy skill is knowing which benchmarks force a standard harness (comparable across models) versus which let every vendor bring their own (not comparable).

### Benchmark map

| Benchmark | What it measures | Size / form | Harness sensitivity | Key gotcha |
|---|---|---|---|---|
| **SWE-bench** (full) | Resolve real GitHub issues; patch must pass hidden tests | 2,294 task instances | **Very high** — invented the agent-scaffold-matters story | Many issues predate model training cutoffs → contamination |
| **SWE-bench Verified** | Human-filtered subset OpenAI vetted as solvable | 500 instances | **Very high** | Self-reported scores dominate leaderboards; OpenAI retired it as a *frontier* eval over contamination |
| **SWE-bench Multimodal** | Issues that *require* a screenshot/diagram to solve | 617 instances, 17 JS libs; 83.5% need the visual [aggregator] | High | Tests vision + code; top resolve rate ~12.2% [aggregator] — far from saturated |
| **SWE-bench Pro** | Harder, longer-horizon, more languages/repos; some held private | 731 public problems [aggregator] | **Very high** — Scale runs a standardized SEAL harness *and* publishes a private subset | The standardized SEAL numbers are the only directly model-comparable ones |
| **SWE-bench Multilingual** | Same task shape across 9 languages | 300 tasks | High | Exposes English/Python-centric harness tuning |
| **Multi-SWE-bench** | Multi-language extension of the SWE-bench methodology | multi-repo | High | Same scaffold-conflation issues as the original |
| **SWE-Lancer** (OpenAI) | Real Upwork freelance tasks, paid; end-to-end tests + manager-choice tasks | 1,400+ tasks ≈ $1M payouts [reported] | High | Closest to "did it earn money"; frontier models still can't solve the majority [reported] |
| **Terminal-Bench / 2.0** | Complex terminal tasks (SWE, security, biology, gaming); binary pass/fail | 2.0 = 89 tasks | **Very high** — leaderboard shows the *harness* as a first-class column | The cleanest public demonstration of same-model-different-harness spread |
| **Aider polyglot** | 225 hard Exercism exercises, 6 languages; 2 attempts with test feedback | 225 problems | **Very high** — *reports edit-format compliance as its own metric* | The benchmark that made "edit format is a score variable" undeniable |
| **SWE-PRBench** | Quality of AI code *review* against real PR feedback | PR-derived | Medium | A different axis — review, not authoring |

### Why "Model X gets Y%" is close to meaningless

Every public coding-agent leaderboard conflates the model with the harness wrapped around it: tool definitions, retry logic, context management, the edit format, the turn budget, and the verification loop. Two facts make this concrete:

- **The same model, three harnesses, three scores.** On the official Terminal-Bench 2.0 leaderboard (June 2026), **GPT-5.5** appears at the top under three different agent harnesses: NexAU-AHE **84.7% ±2.1**, Capy **83.1% ±2.1**, and Codex CLI **82.2% ±2.2** — identical model, a ~2.5-point spread that is purely harness. [primary, tbench.ai] The leaderboard deliberately surfaces the **Agent (harness)** column and links verified trajectories so this is visible rather than hidden.

- **The leaderboards are mostly self-reported.** On the SWE-bench Verified tracker, the overwhelming majority of entries are vendor-submitted with the vendor's own scaffold; only a tiny fraction are independently verified. [aggregator] Scale's SWE-bench Pro is the exception that proves the rule: it runs every model through one standardized SEAL harness, and *those* numbers are the only ones that isolate model capability — and they sit far below the self-reported headline numbers. [aggregator]

Epoch AI, which runs SWE-bench Verified under its own scaffolds, is explicit that it reports *multiple* scaffolds (a simple loop, Claude Code, Codex via Inspect-SWE) side by side, and that a February 2026 scaffold upgrade (v2.0.0) "led to model performance improving significantly" — so much so that Epoch only charts v2.0.0-and-later results together, an implicit admission that scores across harness versions are not comparable. [primary, epoch.ai] **The harness version moved the score enough to break comparability — with no model change at all.**

**Literacy rule:** before quoting a coding-agent number, ask three questions — *which harness, self-reported or standardized, and which benchmark version?* If you can't answer all three, the number is a vibe.

---

## 2. What moves scores — harness, not model

This is the core evidence table. The decisive column is **Change type**: a *clean single-variable* ablation (one knob moved, model and everything else fixed) is far stronger evidence than a *bundled* set of changes, because bundled improvements can't rule out interactions. The strongest entries below are clean single-variable swings.

| # | Change (model held fixed) | Benchmark | Before → After | Delta | Change type | Source |
|---|---|---|---|---|---|---|
| 1 | Edit format: SEARCH/REPLACE → **unified diff** (gpt-4-1106 / GPT-4 Turbo) | Aider code-edit | **20% → 61%** | +41 pp | **Clean single-variable** | [primary, aider.chat/docs/unified-diffs] |
| 2 | Edit format: SEARCH/REPLACE → **unified diff** (gpt-4-0613 / June GPT-4) | Aider code-edit | **26% → 59%** | +33 pp | **Clean single-variable** | [primary, aider.chat/docs/unified-diffs] |
| 3 | Edit format: patch → **hashline** (Grok Code Fast 1) | edit-format harness test | **6.7% → 68.3%** | +61.6 pp | **Clean single-variable** | [primary, blog.can.ac — confirmed in `research/harness-engineering.md`] |
| 4 | Edit format: patch → **hashline** (GPT-4 Turbo) | edit-format harness test | **26% → 59%** | +33 pp | **Clean single-variable** | [primary, blog.can.ac] |
| 5 | **Agent-Computer Interface** vs plain Linux shell (GPT-4 Turbo) | SWE-bench (300-issue subset) | ablation: shell baseline → +ACI | **+10.7 pp** | **Clean single-variable** | [primary, arxiv:2405.15793 ablation] |
| 6 | Adding the SWE-agent ACI (GPT-4 Turbo) vs prior best non-interactive RAG system | SWE-bench full | ~3.8% (prior SOTA) → **12.5%** pass@1 | +8.7 pp | Bundled (whole ACI) | [primary, arxiv:2405.15793 abstract] |
| 7 | **Full harness rebuild** (loop detection + verification middleware + local-context mapping + reasoning sandwich), model = gpt-5.2-codex | Terminal-Bench 2.0 | **52.8% → 66.5%** | +13.7 pp | Bundled (stack of changes) | [primary, langchain.com blog] |
| 8 | Reasoning-budget allocation: **xhigh-only vs high-only vs sandwich** (same model & harness) | Terminal-Bench 2.0 | xhigh **53.9%**, high **63.6%**, sandwich **66.5%** | up to +12.6 pp | **Clean single-variable** (budget schedule) | [primary, langchain.com blog] |
| 9 | **Context editing / pruning** in a 100-turn agent loop (Anthropic) | internal web-search eval | **+29%** task performance, 84% fewer tokens [aggregator-summary of Anthropic] | +29% | Single-variable (context strategy) | [reported, Anthropic via search] |
| 10 | **SWE-Pruner** task-aware context pruning | SWE-bench Verified | success rate *improved* while cutting **23–54%** of tokens | + (small) | Single-variable | [primary, arxiv:2601.16746] |

### Reading the table

- **The biggest swings are edit-format, not model upgrades.** Rows 1–4 are 33–62 points from a single mechanical change — how the model is asked to express an edit. The model's reasoning never changed; the *interface for committing that reasoning to disk* did. Aider's own explanation: with unified diffs "GPT acts more like it's writing textual data intended to be read by a program, not talking to a person," which "encourages rigor" and cut lazy/placeholder code by 3×. [primary] The patch format's failure mode (exact-string-not-found errors) was *hiding the model's real coding ability entirely.*

- **Weaker models gain the most.** In the hashline experiment, Grok Code Fast 1 jumped 6.7→68.3 — a ten-fold change — because the old format penalized mechanical text-matching, not reasoning. Capable models clear the format hurdle on their own; the harness change rescues the ones that were drowning in it. [primary]

- **The ACI ablation is the canonical clean result** (row 5): hold the model fixed, swap a purpose-built file-nav/edit/test interface for a raw shell, and resolve rate rises 10.7 points on a 300-issue subset. This is the result that named the discipline ("Agent-Computer Interface"). [primary, arxiv:2405.15793]

- **LangChain is the headline "harness-only" case but NOT a clean single-variable ablation.** They moved from outside the top 30 to top 5 (+13.7 pp) with the model fixed (gpt-5.2-codex) — but by LangChain's own account they "only tweaked the harness... not an ablation study of individual variables." It bundles prompt changes, three middlewares, and the reasoning sandwich. Treat row 7 as proof that *harness work pays off*, and row 8 (the reasoning-budget schedule) as the clean single-variable result *inside* their work. [primary]

- **Context management is its own lever.** Pruning isn't just a cost optimization — rows 9–10 show pruning that *improves accuracy* while cutting tokens, because removing noise reduces the attention dilution ("context rot") that degrades long runs.

---

## 3. Verification: the highest-leverage component for one failure mode

Calibrating the claim: across the evidence above, the *largest* single-variable swings are edit-format and ACI, not verification. So the honest framing is narrower and still powerful: **verification (test-in-the-loop) is the highest-leverage harness component for the specific, pervasive failure mode of agents declaring premature success.**

### The premature-success failure mode

Agents routinely mark a task "done" without running the tests — "victory declaration bias." The fix is a gate that *intercepts the exit* and forces a verification pass against the spec. LangChain's `PreCompletionChecklistMiddleware` does exactly this, and the build-verify loop (Planning → Build → Verify → Fix) was a core contributor to their +13.7 pp. [primary] The Aider benchmark bakes verification into its protocol: two attempts, with the unit-test results from attempt 1 shown before attempt 2 — i.e., test feedback *is* the harness, and it measurably lifts pass rates. [primary]

The mathematical motivation (from `research/harness-engineering.md`): at 85% per-step accuracy over 10 steps, naive success is 0.85^10 ≈ 20%. A verification gate after each major phase recovers much of that compounding loss — which is why verification's leverage is structural, not incidental.

### The verifier that rubber-stamps (the calibration problem)

Verification only helps if the verifier is *calibrated*. The failure mode is a gate that passes everything — a green light that means nothing. From `research/evals.md`: a team that used a same-family model as judge saw essentially perfect metrics for three months while the judge's actual agreement with domain experts was **Cohen's kappa ≈ 0.31** (below the "moderate agreement" floor). The dashboard was green; the product was degrading. [evals.md, primary-adjacent]

Known LLM-judge biases that make a verifier rubber-stamp (from `research/evals.md`): position bias (~40% of GPT-4 verdicts flip when response order swaps), verbosity bias (~15% inflation for longer outputs), self-enhancement bias (5–7% self-preference). Mitigations: judge with a *different model family* than the generator, evaluate both orderings, and calibrate against 30+ human-labeled examples before trusting the gate.

**Net:** the highest-leverage move for the "agent lies about being done" problem is a *calibrated, test-backed* verification gate. An uncalibrated verifier is worse than none — it manufactures false confidence at scale.

---

## 4. The benchmark-to-production gap

A green benchmark is not a mergeable PR. The gap has three sources.

### 4.1 Contamination — the score is partly memorization

SWE-bench draws on public GitHub issues that were filed and fixed *before* current models' training cutoffs. If the repo was in training data, the model may have seen the answer. Reported evidence:

- **OpenAI retired SWE-bench Verified as a frontier eval**, stating improvements "increasingly reflect how much the model was exposed to the benchmark at training time" rather than real software-engineering gains. [reported, via search]
- One analysis attributes roughly a third of successful patches to solution leakage and notes models recall correct file paths ~76% of the time from training data. [aggregator — treat as directional]
- The tell-tale signature: frontier models sit in the **76–81%** band on SWE-bench Verified but drop to the **sub-25%** band on contamination-resistant, enterprise-style variants and **<50%** on never-before-seen private code. [aggregator] The benchmark measures partly *familiarity*, not capability.

### 4.2 A passing test ≠ a mergeable PR

Meter (an AI-evaluation org) had repository maintainers review SWE-bench solutions from recent agents the way a real PR gets reviewed. Finding: **agent solutions merged at roughly half the rate of human golden solutions.** Human golden patches were rejected ~40% of the time; agent patches at roughly twice that rate (~80% rejected) — *even though they passed the hidden tests.* Maintainers catch what tests don't: style, scope creep, maintainability, subtle wrongness. [reported, Meter via MindStudio]

A subtle interpretation point also flagged in that coverage: a 70% SWE-bench score means "succeeds on ~70% of *problem types*," not a 70% per-attempt reliability — and the metric is sensitive to fitting choices (regularized vs fixed-slope logistic shifts horizon estimates ~35%). [reported]

### 4.3 What teams measure in production instead

Benchmark pass rate is replaced by operational signals:

- **PR merge rate on your own repo, your own maintainers, your own problem types** — Meter's explicit recommendation. [reported]
- **Human-intervention rate** — how often a human had to step in.
- **Bounded-iteration caps as a quality proxy.** Stripe caps the agent at **two push-and-test CI cycles** before routing to human review, citing "diminishing marginal returns if an LLM is running against indefinitely many rounds of a full CI loop." [primary, from `research/harness-engineering.md` → Stripe Minions] The cap is both a cost control and a signal: tasks that survive two cycles are the mergeable ones.
- **pass^k, not just pass@1** — for anything customer-facing. At 75% per-trial, pass@3 ≈ 98% but pass^3 ≈ 42%. Production cares about pass^k. [`research/harness-engineering.md`, Anthropic]

**Bottom line:** SWE-bench tells you a model/harness *can* solve a class of problems in a clean room. It does not tell you the output is mergeable, uncontaminated, or reliable on *your* code. Those require production instrumentation.

---

## 5. How to run harness ablations: a recipe

The whole point of the evidence above is that you can *find your own* +30-point harness wins — but only with disciplined ablation. The method is the scientific method applied to your scaffold.

### The core protocol: one variable, model fixed, measured

1. **Pin the model and decoding.** Same model version, same temperature/reasoning budget, same seed policy. The model is your control, not your variable. (This is exactly what makes rows 1–5 and row 8 in §2 trustworthy and row 7 weaker.)

2. **Pin the benchmark and the environment.** Same task set, same sandbox image, clean environment *per trial* (shared state causes correlated failures that masquerade as model behavior — `research/evals.md`). For coding work, start with 20–50 tasks drawn from your *real* failures, not a public benchmark you might be contaminated on.

3. **Change exactly one harness knob.** The high-yield knobs, ranked by demonstrated swing:
   - **Edit format** (SEARCH/REPLACE vs unified diff vs hashline) — biggest demonstrated lever (rows 1–4).
   - **The ACI** — file navigation, editor commands, structured test output, error-message formatting (row 5).
   - **Reasoning-budget schedule** — flat vs sandwich (row 8).
   - **Verification gate** — exit interception + test-in-loop (on/off).
   - **Context strategy** — full history vs pruned/compacted (rows 9–10).
   - **Loop/turn budget** — and a doom-loop detector (on/off).

4. **Run enough trials for a signal, and report uncertainty.** Agent runs are stochastic — the *same task* can vary up to ~30× in tokens [reported] and flip pass/fail. Report mean ± SEM, not a single run. The tbench leaderboard models this well: every score carries a ± band (e.g., 84.7% ±2.1). Use clustered standard errors if tasks come in related groups (`research/evals.md` — ignoring clustering can underestimate uncertainty by >300%).

5. **Measure both axes: success AND cost.** Record pass rate *and* tokens-to-success / dollars-to-success. A harness change that adds 3 points but doubles tokens may lose to one that adds 2 points for free (see §6).

6. **Read the traces, don't just diff the scores.** The largest wins come from *seeing the failure mode* — patch-format errors, doom loops, premature exits — then targeting it. LangChain built a Trace Analyzer (fetch traces → parallel error-analysis agents → synthesize → propose harness fixes); `research/evals.md` calls the manual version "error analysis," the single most-skipped, highest-ROI step. Pick one: automate it or do it by hand, but do it.

7. **Promote a change only if it survives a held-out task set.** A win on your dev tasks that vanishes on held-out tasks was overfitting to the eval, not improving the harness.

### What a clean ablation log looks like

```
baseline:    gpt-5.2-codex, default harness, 50 tasks, 5 trials → 52.8% ±2.0, 1.2M tok/task
+ edit-format=hashline (only change)        → 61.3% ±1.9, 1.1M tok/task   ✅ promote
+ verification gate (on top of hashline)    → 64.0% ±2.1, 1.4M tok/task   ✅ promote (cost ok)
+ reasoning=xhigh-flat (replaces sandwich)  → 58.1% ±2.4, 2.1M tok/task   ❌ revert (timeouts, worse, costlier)
```

Each line changes one thing from the line above it; each carries a band and a cost; each has an explicit promote/revert decision. That is the entire discipline.

---

## 6. Cost and latency as harness metrics

Success rate alone is a half-measure. The right unit is **tokens-to-success** (or dollars-to-success), because agentic loops are where money disappears.

- **Agentic coding is ~1000× more token-hungry than chat.** Input tokens (re-reading files, re-sending context every turn), not output tokens, drive the bill. [reported, via search of arxiv:2604.22750] This is why context pruning (§2, rows 9–10) is a *first-class* harness lever, not a nice-to-have: cutting 23–54% of tokens with no accuracy loss is a direct margin win. [primary, arxiv:2601.16746]

- **Cost is wildly stochastic.** Runs on the *same task* can differ by up to ~30× in total tokens. [reported] This means a cost number without a distribution is as meaningless as a score without a confidence band — measure the spread, not just the mean.

- **The reasoning-budget tradeoff is real and measurable.** LangChain's xhigh-everywhere config scored *worse* (53.9%) than the cheaper sandwich (66.5%) — partly because xhigh ran into agent timeouts. [primary] More compute is not monotonically better; the harness has to spend reasoning where it pays (planning, verification) and conserve it elsewhere.

- **Bounded iteration is a cost control with a quality rationale.** Stripe's two-CI-cycle cap (§4) exists because additional CI rounds have diminishing returns — the harness should *know when to stop spending.* [primary]

- **The frontier-vs-budget gap is steep.** The very top SWE-bench models cost on the order of 20–50× more per output token than budget alternatives for single-digit score gains. [aggregator] Whether that's worth it is a per-task economic decision — exactly the calculation a harness metric of dollars-to-merged-PR makes visible and a bare leaderboard score hides. Epoch AI's longer-run framing: the price to hit a given capability milestone has been falling 9×–900× per year depending on the milestone [reported] — so cost-efficiency, not peak score, is where most of the practical movement is.

**Metric to adopt:** for any agent you ship, track **dollars-per-merged-PR** and **tokens-per-success**, with their distributions, alongside pass rate. A harness change is only a win if it improves at least one without regressing the others past your budget.

---

## Implications for Dalgo

- Dalgo's agents (the MCP tools: `sync_sources`, `run_dbt`, `trigger_pipeline_run`) operate on real NGO data with tiny budgets. The §6 lesson is acute: **measure tokens-to-success**, because an unbounded agentic loop on a misconfigured dbt run burns money with nothing to show.
- The §3 lesson — a verification gate that actually runs the check (does the dbt model compile? does the schema match?) before declaring success — is the highest-leverage single addition, and it must be code-based (deterministic), not an LLM rubber-stamp.
- When evaluating which model to put behind a Dalgo agent, ignore raw SWE-bench headline numbers. Run §5's recipe on 20–50 *real Dalgo tasks* with your harness fixed. The edit-format / tool-interface lever will likely move your results more than the model choice.

---

## 7. Sources

Primary (fetched and confirmed):
- [Unified diffs make GPT-4 Turbo 3X less lazy (aider.chat)](https://aider.chat/docs/unified-diffs.html) — 20%→61%, 26%→59% on edit format alone
- [Aider LLM Leaderboards (aider.chat)](https://aider.chat/docs/leaderboards/) — edit-format-compliance metric, diff vs whole, top-model pass rates
- [Improving Deep Agents with Harness Engineering (LangChain)](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering) — 52.8%→66.5%, model fixed (gpt-5.2-codex), reasoning-sandwich 53.9/63.6/66.5, "not an ablation study of individual variables"
- [Terminal-Bench 2.0 official leaderboard (tbench.ai)](https://www.tbench.ai/leaderboard/terminal-bench/2.0) — GPT-5.5 at 84.7/83.1/82.2 across three harnesses; harness shown as a column; ± bands
- [SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering (arxiv:2405.15793)](https://arxiv.org/abs/2405.15793) — 12.5% pass@1 on SWE-bench; +10.7 pp ACI ablation vs shell
- [SWE-bench Verified (Epoch AI)](https://epoch.ai/benchmarks/swe-bench-verified) — multiple scaffolds reported; v2.0.0 scaffold upgrade broke cross-version comparability
- [SWE-Pruner: Self-Adaptive Context Pruning for Coding Agents (arxiv:2601.16746)](https://arxiv.org/pdf/2601.16746) — 23–54% token cut while improving success
- [blog.can.ac — the edit-format / hashline experiment](https://blog.can.ac/2026/02/12/the-harness-problem/) — Grok Code Fast 1 6.7%→68.3% (also confirmed in `research/harness-engineering.md`)

Reported (vendor/coverage):
- [Introducing the SWE-Lancer benchmark (OpenAI)](https://openai.com/index/swe-lancer/) — 1,400+ tasks, ~$1M payouts; frontier models can't solve the majority (page 403 at fetch; figures via [InfoQ](https://www.infoq.com/news/2025/03/openai-swe-benchmark/) and [Analytics Vidhya](https://www.analyticsvidhya.com/blog/2025/02/openais-swe-lancer-benchmark/))
- [SWE-Bench Score vs. Real Merge Rate (MindStudio)](https://www.mindstudio.ai/blog/swe-bench-score-vs-real-merge-rate-agent-benchmark-gap) — Meter study: agent patches merge at ~half the human rate
- [How Do AI Agents Spend Your Money? (arxiv:2604.22750)](https://arxiv.org/abs/2604.22750) — agentic ≈ 1000× chat tokens; ~30× same-task variance
- [SWE-bench Verified (Epoch AI)](https://epoch.ai/benchmarks/swe-bench-verified) and [Epoch cost analyses](https://epoch.ai/) — 9×–900×/yr price-to-milestone decline

Aggregator / benchmark-tracking sites (directional, treat as pointers not fact):
- [SWE-bench in 2026: Benchmarks vs Scaffolding Reality (digitalapplied)](https://www.digitalapplied.com/blog/swe-bench-verified-june-2026-benchmark-vs-scaffolding-analysis) — vendor-vs-SEAL spread, contamination %s, 76–81% / sub-25% bands
- [SWE-bench Pro Leaderboard (morphllm)](https://www.morphllm.com/swe-bench-pro) · [Terminal-Bench 2.0 (morphllm)](https://www.morphllm.com/terminal-bench-2) · [SWE-bench Multimodal (swebench.com)](https://www.swebench.com/multimodal)
- [Artificial Analysis Coding Agents](https://artificialanalysis.ai/agents/coding-agents) — cost/latency per task

Companion docs (this is their benchmark/measurement complement):
- `../research/harness-engineering.md` — what a harness is, patterns, the ACI/Stripe/LangChain case studies in depth
- `../research/evals.md` — eval methodology, LLM-judge calibration, pass@k vs pass^k, error analysis
