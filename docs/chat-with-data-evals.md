# How We Built Evals for Chat with Data — and What They Taught Us

> Companion to [`chat-with-data-architecture.md`](./chat-with-data-architecture.md) and
> [`chat-with-data-build-playbook.md`](./chat-with-data-build-playbook.md). Those docs said
> "eval harness in CI" as a build-order step; this doc is the case study of actually building
> it — what we shipped, in what order, what broke, and the learnings that generalize to any
> agent eval. Unlike [`research/evals.md`](../research/evals.md) (industry research), every
> claim here comes from our own commits and runs. Date: 2026-09-24.

The system under test: Dalgo Copilot's SQL agent — question in, routed, SQL generated and
executed against a real warehouse, answer narrated. Code lives in
`DDP_backend/ddpui/core/ai/evals/` (runner + `sql_compare.py` + README), driven by
`manage.py chat_with_data_eval`.

---

## 1. What we built (the 60-second version)

Golden questions in JSONL (in git), replayed through the **real TurnGraph** against the
**real dev warehouse**, scored five ways, pushed to Langfuse where runs sit side by side:

| Score | Type | How |
|---|---|---|
| `eval_routing` | **gate** | router intent == expected intent, string equality |
| `eval_sql_correct` | **gate** | hand-written gold SQL **executed**, result sets compared |
| `eval_faithful` | inform | LLM judge: answer's claims supported by the result table |
| `eval_expectations` | inform | LLM judge vs. a one-sentence expectation (for un-SQL-able questions) |
| `eval_sql_judge` | inform | LLM judge: agent SQL ≈ gold SQL, judged as text |

Gates block; judges inform. Judges run on OpenAI — deliberately a different model family
than the agent — and every judge is fail-open. Each eval item runs single-turn on a fresh
in-memory checkpointer with human-in-the-loop disabled; the runner does the scoring, the
product validator stays out.

One run ≈ $1–2 and ~10 min; `--tag canary` gives a ~5-item, ~30¢ inner loop; `--no-judge`
halves cost. The harness itself is covered by offline unit tests with a scripted model —
zero API cost.

## 2. The build, in commit order

The sequence matters because almost every step exists as a *reaction* to the previous one:

1. **`7439b82b` — the harness.** autoevals scorers + Langfuse dataset runs. Naive first
   version: routing check, executed-SQL comparison, one faithfulness judge.
2. **`9c7b74b7` — first judge fix.** The faithfulness criteria had to be built at call time
   (with the actual result table), and we routed autoevals through an explicit OpenAI client —
   otherwise judge traffic silently goes through autoevals' default Braintrust gateway.
   *Read your dependency's network path before pointing production data at it.*
3. **`ea5ef17d` — a real dataset.** 14 work-order questions (NITI + GDGS) taken from an
   actual partner question set, not invented ones.
4. **`f9316b0b` — the calibration commit.** First full run: **8/14**. Nearly every failure
   was the *dataset's* fault, not the agent's. After fixing ambiguous wording and wrong
   golds: **13/14** — with no agent change at all.
5. **`145f9340` — comparator learns presentation-tolerance.** Agents invent row labels when
   the numbers already identify rows; strict equality over-failed. Multi-row golds got a
   label-tolerant fallback. Every such rule now carries a regression test.
6. **`b177fa3c` — the judge-vs-execution experiment.** We added an LLM SQL-equivalence judge
   *alongside* the execution-based metric, plus an agreement report. Result: they disagreed
   on **8 of 11** items — all eight were the judge false-failing queries that provably
   return identical results.
7. **`6381efd3` — the README.** Only after the loop worked did we document how anyone adds
   a dataset (the 4-step loop: write JSONL → execute every gold → seed → run and fix).
8. **`616d0c51` — `--from-langfuse`.** Run items straight from a Langfuse dataset, so the
   scoreboard can also be a source.

## 3. The learnings

### L1. Your first eval run grades your dataset, not your agent

8/14 → 13/14 purely from fixing item wording and gold mistakes. "How many beneficiaries are
enrolled?" defensibly means 200 *or* 171 depending on the table — the agent flips between
them and the item is flaky forever. Budget the first full run for dataset debugging, and
treat ambiguity as an item bug, not agent noise.

### L2. Never trust a gold you haven't executed

The strongest failure diagnostic we ever printed:

```
[FAIL sql] How many NGOs are working on GDGS work orders?
    agent sql : SELECT COUNT(DISTINCT ngo_name) ... AND ngo_name <> 'Unknown'
    agent rows: [['214']]
    gold rows : [{'n': 215}]      ← the agent was right; the gold forgot 'Unknown'
```

The agent noticed a placeholder `'Unknown'` NGO that the human gold-writer missed. Executing
gold SQL at eval time (not trusting a recorded value) is what surfaced this — and it's also
why the verification step ("run every gold by hand before committing it") is step 2 of the
authoring loop, not an afterthought.

### L3. Gates block, judges inform — and this must be *measured*, not asserted

The 8-of-11 disagreement wasn't a reason to delete the SQL judge; it's the reason the judge
can never veto a merge. Agents write structurally different SQL that returns identical
results; a judge can't run it, only squint at it. We keep the judge *because* the agreement
rate is itself the signal — the runner prints `sql hard-metric vs judge agreement: n/m`
every run, so drift in judge trustworthiness is visible, not vibes.

Corollary: put judges on a **different model family** than the agent so they don't share its
blind spots, and make every judge **fail-open** — a judge outage should never fail a run.

### L4. Comparators must be forgiving of presentation, strict on substance

Real agents enrich: extra context columns, the full ranking for a LIMIT-1 question, invented
labels. Strict result-set equality punishes all of that. `sql_compare.py` grew three tiers
(scalar / single-row / multi-row projection matching), each rule born from a real failure
and pinned by a regression test. Two disciplines make this safe:

- **Metric versioning:** a pass-rate jump after touching the comparator means the *scoring*
  changed, not the agent. Never compare runs across comparator versions.
- **The answer carries the assertion:** a scalar gold must appear in the *narrated answer*,
  not just somewhere in the result table — the user reads the sentence, not the rows.

### L5. Absence questions can't have gold SQL

"How much silt was excavated in Maharashtra?" where Maharashtra has no rows: the correct
answer is *"there's no Maharashtra data"*, which a `0.00` gold can't score. That's what
`answer_expectations` + the expectations judge are for. Every dataset needs a
non-SQL-scoreable lane or you'll quietly stop testing the agent's most honest behavior.

### L6. A flaky item is a finding, not noise

Our Q3-vs-Q4 comparison question fails a different way every few runs — empty result set one
time, router diversion the next. That's not eval noise; that's the agent's real weakness on
date bucketing, caught deterministically. Instead of deleting flaky items, promote them to
the `canary` tag so the cheap loop hits them every time.

### L7. Pin the world or lose reproducibility

Dev warehouses accumulate lookalike schemas (`demo.beneficiaries` vs `test_ngo.beneficiaries`).
Unpinned, the agent wanders between them and two runs aren't comparable. `--schemas` pins the
agent's world for the run. Generalization: an eval against a shared mutable environment must
freeze whatever part of the environment the agent explores.

### L8. Let the data's dirt be the test

`status = 'Active'` vs a user typing "active", placeholder `'Unknown'` rows, TEXT date
columns — the traps already in the warehouse beat any trap you'd invent. We took questions
from a real partner question set for the same reason.

### L9. Design the cost gradient up front

Full run ($1–2) / `--no-judge` (half) / `--tag canary` (~30¢) / offline scripted-model unit
tests (free). People run evals exactly as often as the cheapest useful tier lets them. The
canary tier is what makes "run it on every prompt tweak" real rather than aspirational.

### L10. Single-turn first was right — but know what it doesn't test

Each item is one cold question on a fresh thread: cheap, attributable, non-cascading. The
deliberate blind spots: router stickiness on follow-ups ("yes", "make it monthly"), the
clarify loop (`ask_user` falls back because HITL is off), approval flows, and context
resolution ("same but for Pune"). Multi-turn is a runner/schema extension (a `turns:` list
sharing one thread_id), not a rewrite — but until it exists, those behaviors ship untested.

### L11. "Source of truth in git" needs guarding like source code

Post-script: weeks after the calibration work, an unrelated commit (`7f12e1c5`, "raise SQL
retry budget") silently deleted all three golden JSONL files. Nothing failed — evals don't
run in CI, so nothing noticed. Two lessons: (a) dataset files deserve the same review
attention as code, because they *are* the spec; (b) an eval that isn't wired into CI can rot
invisibly. (The files are recoverable: `git show 7f12e1c5^:ddpui/core/ai/evals/golden_v1.jsonl`.)

## 4. What we'd do next

- **Restore the golden datasets** and add a CI canary run (`--tag canary --no-judge` is
  ~free and would have caught L11 the same day).
- **Multi-turn items** for router stickiness and the clarify loop (L10).
- **Table-selection score**: `expected_tables` is already in the schema, unscored — a cheap
  hard metric for "did it even look at the right table".
- **Judge calibration over time**: the agreement line is per-run; trending it across runs in
  Langfuse would tell us when a judge is worth re-prompting or retiring.
