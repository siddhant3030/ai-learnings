# The Agent Lifecycle for Dalgo

> How to operate and continuously improve a runtime agent over time — without making
> the runtime smart.
>
> **Core principle (from the team):** Intelligence belongs at **build-time**. The
> runtime agent stays **dumb-but-reliable** — a small, deterministic loop over a fixed
> set of Dalgo MCP tools. The *lifecycle* is the loop where a sophisticated dev-time
> process (evals, failure analysis, prompt/tool tuning, CI gates) continuously upgrades
> that runtime agent. Every section below answers one question: **how does this keep
> the runtime simple while the dev loop gets smarter?**

This is the operational companion to the rest of this folder. Where the other docs cover
*what to build* (architecture, tools, evaluation), this one covers *how to run it and
keep improving it after launch* — the part where 95% of enterprise AI investments produce
no measurable return ([MIT/NANDA, via agent-design research](../research/agent-design.md)),
almost always because there was no improvement loop.

---

## 1. The Lifecycle Loop

The lifecycle is a four-stage loop that **straddles the build-time / runtime boundary**.
The runtime does only two things: emit traces and receive shipped config. Everything
intelligent — evaluation, failure analysis, improvement — happens at build-time, off the
hot path, where a human and a dev-time harness can take their time.

```
                         BUILD-TIME  (smart, slow, human-in-loop, off the hot path)
        ┌───────────────────────────────────────────────────────────────────────┐
        │                                                                         │
        │     ②  EVALUATE                          ③  IMPROVE                     │
        │   ┌──────────────────┐               ┌──────────────────────┐          │
        │   │ • eval suite run │               │ Cheapest lever first: │          │
        │   │ • failure triage │──────────────▶│ tool desc → prompt →  │          │
        │   │ • LLM-as-judge   │   failures    │ retrieval → tool set →│          │
        │   │   (10–20% sample)│   become      │ model tier → finetune │          │
        │   │ • new eval cases │   eval cases  │ (config + prompt edits)│         │
        │   └──────────────────┘               └──────────┬───────────┘          │
        │            ▲                                     │                      │
        │            │ traces + feedback                   │ candidate version    │
        └────────────┼─────────────────────────────────────┼──────────────────────┘
                     │                                      │
        ═════════════╪══════════════════════════════════════╪═══════════ boundary ═
                     │                                      ▼
        ┌────────────┴─────────────┐              ┌──────────────────────┐
        │  ①  OBSERVE              │              │  ④  SHIP             │
        │ runtime emits:           │              │ gated by eval suite: │
        │ • every turn + tool calls│◀─────────────│ • regression gate    │
        │ • latency, tokens, cost  │  new config  │ • canary: 1 org first│
        │ • refusals, 👍/👎        │  deployed    │ • staged rollout     │
        │ • redacted args (kavach) │              │ • instant rollback   │
        └──────────────────────────┘              └──────────────────────┘
                  RUNTIME  (dumb, fast, deterministic — a fixed loop over Dalgo MCP tools)
```

Read it as a clockwise cycle: **Observe** (production traces) → **Evaluate** (eval suite +
failure analysis at build-time) → **Improve** (edit prompt/context/tools, cheapest lever
first) → **Ship** (gated by evals, canary to one org) → back to Observe. The runtime never
changes its *shape*; only the config it loads changes.

This is the same loop the most mature teams run. Discord runs evals as unit tests before
every deploy; Amazon Finance moved retrieval accuracy from 49% to 86% **by mining
observability data, not by changing models**; Braintrust formalizes the
"production failure → regression test" path
([ai-observability research](../research/ai-observability.md)). Dalgo's version is the same
loop, sized for a small team and a ₹2L/yr-per-NGO budget.

---

## 2. Observability — Close the 89% / 44.8% Gap

The single most important statistic for this whole document:

> **89%** of production teams have tracing. Only **44.8%** run online evaluations.
> ([LangChain State of Agent Engineering](../research/ai-observability.md))

Most teams can see *that* something happened but not *whether it was good*. The lifecycle
loop closes this gap: Observe (§1①) feeds Evaluate (§1②). Tracing without evals is half a
loop. **Do not stop at tracing.**

### What to instrument (every turn)

For a Dalgo data-Q&A / pipeline-debug agent acting over the Dalgo MCP, capture these as
spans on every request:

| Signal | Captured as | Why it matters for Dalgo |
|--------|-------------|--------------------------|
| **Turn** | One trace per user question | The end-to-end unit; groups all spans below |
| **Tool calls + args** | One span per MCP call: name, args, result, latency, retries | The dominant failure mode — wrong tool or bad params |
| **Tool-selection mix** | Aggregate of which tools fire | Detects the agent reaching for `dalgo_run_dbt` when it should `dalgo_get_flow_run_logs` |
| **Latency** | TTFT + end-to-end + per-span | Slow internet users; a 28s `dalgo_get_table_data` is invisible without span timing |
| **Token cost** | input/output tokens × price, per request **and per org** | ₹2L/yr budgets — cost must roll up by NGO |
| **Refusals** | Boolean + reason | Over-refusal kills trust ("when AI breaks, users abandon it for months" — Sentry) |
| **User feedback** | 👍 / 👎, "this was wrong", correction text | The raw material for new eval cases (§3) |

Span types follow the four-type agent model: **tool-call**, **reasoning**, **state
transition**, **memory operation** spans — each maps to a distinct failure mode
([Braintrust agent observability](../research/ai-observability.md)). Tag *every* span with
`org_id`, `session_id`, `agent_version`, and `prompt_version`. That single discipline
unlocks cost-by-org, quality drill-down by NGO, and per-version regression analysis.

### Concrete Dalgo span tree

```
Trace: "why did my pipeline fail last night?"   [org_id=ngo_42, agent_v=3, prompt_v=11]
  ├── Reasoning span: plan = inspect recent runs, then read logs
  ├── Tool span: dalgo_get_pipeline_run_history  (220ms, 1 result)
  ├── Tool span: dalgo_get_flow_run_logs         (1.8s, 14KB → 4.2k tokens ⚠️)
  ├── Reasoning span: identify dbt test failure on null check
  └── Generation: llm_call  (1.1s, in=5.9k out=180, $0.004, finish=stop)
```

Note the ⚠️: a single `dalgo_get_flow_run_logs` call dumped 4.2k tokens. **Tool outputs
consume up to 100x more tokens than user messages** — without span-level token attribution
you never see that 80% of your bill comes from one verbose tool call
([ai-observability research](../research/ai-observability.md)). This is exactly the signal
that drives the cheap improvement levers in §4.

### PII redaction — use kavach, don't hand-wave it

Dalgo serves NGOs with **sensitive beneficiary PII**. Tool args and results often contain
real data (`dalgo_get_table_data` returns warehouse rows). **Redact before storage.**
Dalgo already has a PII guard — **kavach** (see the `kavach:status` skill). Route all
captured args/results through kavach's detectors before they hit the trace store. This is
a deterministic, cheap, build-side check — it does *not* make the runtime smarter, it makes
the *observability pipe* safe.

### Where to store it — Django **and** Langfuse

A clean two-store split that respects the budget and the open-source ethos:

| Store | What lives there | Why |
|-------|------------------|-----|
| **Langfuse (self-hosted)** | Traces, spans, generations, prompt versions, eval datasets, LLM-as-judge scores | Open-source (MIT), self-hostable in minutes via Helm/Docker, OTel-native (no vendor lock-in), data stays on Dalgo infra. The leading OSS choice for exactly this profile. ([Langfuse](https://langfuse.com/), [self-hosting](https://langfuse.com/self-hosting)) |
| **Django models** | `AgentTrace`, `AgentFeedback`, `EvalCase`, cost rollups — joined to existing **org / RBAC** models | Feedback and cost-per-org need to live next to org data and permissions. On the current `feature/rbac` branch, org-scoping is the natural hook for both cost attribution (§7) and canary rollout (§5). |

Sketch of the Django side (lightweight — these are thin records, not the trace store):

```python
class AgentTrace(models.Model):
    org = models.ForeignKey(Org, on_delete=models.CASCADE)        # RBAC-scoped
    langfuse_trace_id = models.CharField(max_length=64, db_index=True)
    agent_version = models.CharField(max_length=32)
    prompt_version = models.CharField(max_length=32)
    input_tokens = models.IntegerField(default=0)
    output_tokens = models.IntegerField(default=0)
    cost_usd = models.DecimalField(max_digits=10, decimal_places=5, default=0)
    tools_used = models.JSONField(default=list)   # ["dalgo_get_flow_run_logs", ...]
    refused = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

class AgentFeedback(models.Model):
    trace = models.ForeignKey(AgentTrace, on_delete=models.CASCADE, related_name="feedback")
    org = models.ForeignKey(Org, on_delete=models.CASCADE)
    rating = models.SmallIntegerField()           # +1 / -1
    correction = models.TextField(blank=True)     # "this was wrong, the answer is X"
    promoted_to_eval = models.BooleanField(default=False)   # see §3 flywheel
    created_at = models.DateTimeField(auto_now_add=True)

class EvalCase(models.Model):
    org = models.ForeignKey(Org, null=True, on_delete=models.SET_NULL)  # provenance
    source_feedback = models.ForeignKey(AgentFeedback, null=True, on_delete=models.SET_NULL)
    question = models.TextField()
    expected = models.TextField(blank=True)       # gold answer or rubric
    scorer = models.CharField(max_length=32)      # "exact" | "llm_judge" | "tool_match"
    tags = models.JSONField(default=list)
    created_at = models.DateTimeField(auto_now_add=True)
```

Langfuse holds the heavy telemetry; Django holds the org-scoped rollups, feedback, and the
eval corpus. Both are queryable by the build-time harness.

---

## 3. Feedback Loops — The Flywheel

The flywheel is the heart of the lifecycle: **production failures → new eval cases →
improvements → fewer failures.** This is what turns observability into improvement and
closes the 89%/44.8% gap.

```
  👎 / "this was wrong" / correction        (captured in AgentFeedback)
              │
              ▼  weekly triage (human, 30 min — see §9)
   cluster failures by pattern  ──▶  is it a recurring pattern (2+)? 
              │                                    │ yes
              │ no → log, move on                  ▼
              │                          write an EvalCase from the real trace
              │                          (question + gold answer/rubric + scorer)
              │                                    │
              ▼                                    ▼
        next week                         add to Langfuse eval dataset
                                                   │
                                                   ▼
                              runs in CI on every agent change (§8)
                                                   │
                                                   ▼
                              online scorer watches for the same pattern in prod
```

### Make it concrete for Dalgo

A real failure → eval-case walkthrough:

1. **Capture.** An NGO program manager asks *"how many children were enrolled last
   quarter?"* The agent calls `dalgo_get_table_data` on the wrong table and answers with a
   number from a staging table. They hit 👎 and type *"that's the raw import, use the
   cleaned `fct_enrollments` model."* Stored in `AgentFeedback` with the linked trace.
2. **Triage.** In the weekly review, this clusters with two other "queried staging instead
   of mart" failures. **2+ occurrences = a pattern** (the reflect rule from
   [memory-systems research](../research/memory-systems.md)).
3. **Promote to eval.** Create an `EvalCase`: question = the real question, expected = uses
   `fct_enrollments` / does not read `stg_*` tables, scorer = `tool_match` (deterministic:
   did the trace call the mart, not staging?). `promoted_to_eval = True` on the feedback.
4. **Gate.** The case joins the Langfuse dataset and runs in CI on every prompt/tool change
   (§8). Any future regression that re-introduces "reads staging" fails the build.
5. **Watch.** An online scorer flags any production trace that reads `stg_*` when answering
   a metric question, so the same pattern resurfaces in the dashboard if it recurs.

Two sampling tiers keep cost sane (the production consensus):
**heuristic/deterministic scorers on 100%** of traces (cheap — refusals, staging-table
reads, schema violations), **LLM-as-judge on a 10–20% sample** for semantic quality.
Evaluating 100% with an LLM judge doubles inference cost for no statistical gain.

The key property: **production traffic is the ground truth.** Synthetic test data misses the
long tail of how real, non-technical NGO staff actually phrase questions. Every eval case
should trace back to a real `AgentFeedback` row.

---

## 4. Improvement Levers — Cheapest First

When an eval fails or a pattern emerges, **reach for the cheapest lever that fixes it.**
The research is emphatic and consistent: tool descriptions and context beat model upgrades.
"Most quality failures trace back not to model capability but to poor context management."
([ZenML 1,200 deployments](../research/agent-design.md))

| # | Lever | Cost / effort | Evidence it works | Dalgo example |
|---|-------|---------------|-------------------|---------------|
| 1 | **Tool descriptions** | Minutes, no deploy risk | Input examples: **72% → 90%** accuracy; descriptions are prompt engineering ([Anthropic](../research/agent-design.md)) | Clarify in `dalgo_get_flow_run_logs` desc: "use this to diagnose a *failed* run; for run *history* use `dalgo_get_pipeline_run_history`" |
| 2 | **Context / prompt** | Hours | Most agent failures are context failures, not model failures; static-first for prompt caching | Pin the org's warehouse schema summary; teach broad-to-narrow ("list schemas before querying a table") |
| 3 | **Retrieval** | Days | Amazon Finance: **49% → 86%** by fixing chunking/retrieval, *not* the model | Better doc retrieval for `dalgo_search_docs`; surface the right dbt model names so the agent stops guessing |
| 4 | **Tool set** | Days | Fewer tools = better: Cubic's agent *degraded* with more tools; <30 tools → 3x selection accuracy; Tool Search **49% → 74%**, −85% tokens ([Anthropic](../research/agent-design.md)) | Don't expose all ~80 Dalgo MCP tools to a Q&A agent. Give it a read-only subset (`dalgo_get_*`, `dalgo_list_*`, `dalgo_search_docs`); hide `dalgo_create_*`/`dalgo_delete_*` |
| 5 | **Model tier** | Quick to try, ongoing $ | Checkr cut cost **94%** by routing to the right tier, not always-big | Use a cheaper tier for routing/classification, a stronger tier only for the final answer over warehouse data |
| 6 | **Fine-tuning** | Weeks + infra | **Almost never** for a small team — hard to audit, delete, update; retrieval/prompt nearly always wins | Not for Dalgo. If schema knowledge is the problem, that's a *retrieval/context* problem (lever 2–3), not a weights problem |

**How a small team iterates:** start at lever 1 every time. Most eval failures are fixed by
a sharper tool description or a context tweak — both are **build-time config edits that ship
without touching runtime code**. Only escalate down the table when the cheaper lever
provably fails the eval. This ordering is the operational expression of "intelligence at
build-time": you are encoding intelligence into descriptions, prompts, and tool curation —
not into a more expensive runtime.

Watch one trap: **context rot begins around 50k tokens** — stuffing more schema/log context
*hurts* past that point ([agent-design research](../research/agent-design.md)). The fix is
just-in-time retrieval (lever 3), not a bigger prompt.

---

## 5. Versioning & Safe Rollout

The runtime loads a versioned **agent config**: `{prompt_version, tool_set, model_tier,
retrieval_config}`. Changing the agent = shipping a new config version. Because the runtime
is dumb, a "deploy" is config, not code — which makes rollout and rollback cheap.

### Version everything

- **Prompts** — in Langfuse Prompt Management: every version gets an ID + label
  (`production`, `canary`), full version history, one-click rollback to any prior version
  ([Langfuse prompt versioning](https://langfuse.com/docs/prompt-management/features/prompt-version-control)).
- **Tool set & model tier** — a small JSON config record in Django, versioned, org-scoped.
- **Eval datasets** — Langfuse versions datasets too (every add/edit produces a new version;
  you can fetch a dataset as-of a timestamp), so you can prove *which* eval set gated *which*
  ship.

### Don't rug-pull users

NGO users build mental models of how the agent behaves. A silent behavior change ("rug pull")
erodes the trust that is the entire product. Guard against it:

- **Pin model/judge versions.** Providers retune models without bumping the string — a silent
  breaking change. Pin both the runtime model *and* the LLM-as-judge model; upgrades go
  through the same eval gate as your own changes.
- **Regression gate before ship.** No config ships if the eval suite regresses. Concretely:
  build fails if average quality drops >5%, if any *previously-passing* case now fails, or if
  p95 latency/cost exceeds threshold. The "production failure → regression test" cases from §3
  are the gate.

### Staged rollout — one org first

This is where the `feature/rbac` org-scoping pays off:

```
  new config ──▶ eval gate ──▶ CANARY: 1 friendly NGO (org_id pinned)
                  (CI, §8)            │  watch 👎-rate, refusals, cost for N days
                                      ▼
                              STAGED: 10% of orgs ──▶ watch dashboards (§9)
                                      │
                                      ▼
                              FULL: all orgs
                                      │
                              ◀── instant rollback to prior prompt_version
                                  if 👎-rate or cost spikes
```

Roll out by `org_id` using the RBAC scoping already in the models. Pick one
technically-comfortable partner NGO as the canary. Rollback is just re-pointing the
`production` label at the previous prompt version — seconds, no deploy. Lightweight in Django:
a `default_agent_config_version` field on `Org`, defaulting to the global `production` version,
overridable per-org for canary.

---

## 6. Memory & Learning Over Time — Minimal at Runtime

The agent *should* get less amnesiac over time — but the learning belongs in
**build-time-curated context**, not a complex runtime memory system. Keep runtime memory
minimal; this is the same principle as everywhere else in this doc.

### What's worth remembering for Dalgo

- **Org-specific schema quirks** — "ngo_42 stores enrollment in `fct_enrollments`, not the
  staging table"; "their dates are DD/MM/YYYY." (Semantic memory)
- **Common question patterns** — the handful of questions each NGO asks every month.
- **Recent debugging context** — when a pipeline fails, the recent history of changes, so the
  agent doesn't start cold. (Episodic memory)

### Start with state-based memory, not retrieval

For a **bounded, schema-validatable** domain like per-org preferences, the
[memory-systems research](../research/memory-systems.md) recommends the OpenAI-cookbook
**state-based** pattern as the *starting* architecture for Dalgo — a structured per-org
profile injected at session start, not a vector store. Letta's own benchmark found a plain
filesystem scores **74%** on memory tasks, beating specialized vector libraries — **simplicity
wins** until you have evidence of failure at scale. Concretely: a curated per-org
`org_context.md`-style block (the CLAUDE.md pattern) holding verified schema facts and
preferences. It lives in context, is fully visible, has zero retrieval infrastructure.

### Store provenance, verify at read time

The single highest-leverage anti-poisoning rule, from GitHub Copilot's production memory
system: **store memories with citations, then verify just-in-time at read.** Copilot re-reads
the cited `file:line` when a memory is recalled, exploiting the asymmetry that *retrieval is
hard but verification is easy*.

Dalgo's version: a learned fact like "metric X lives in `fct_enrollments`" is stored **with a
provenance pointer** (the dbt model / column it came from). At read time, the agent does a
cheap `dalgo_get_node_columns` / `dalgo_get_table_columns` check to confirm the column still
exists before trusting the memory. If the schema changed, the stale fact is dropped, not used.

### Write-gating and staleness

- **Never persist model-invented facts.** Only write memories that are **tool-verified or
  user-confirmed**, atomic, timestamped, and scoped to an org. This is the primary defense
  against memory poisoning (OWASP ASI06; the MINJA attack injects memory through query-only
  interaction). For Dalgo: only persist a schema fact after a `dalgo_get_*` tool confirmed it,
  or the user explicitly corrected the agent (the §3 correction text).
- **Greenfield skills go stale.** A curated context block written today rots as the NGO's dbt
  models evolve. Treat the per-org context as a **build-time-curated artifact with a
  freshness check** — the weekly review (§9) re-validates it, and read-time verification
  catches drift between reviews. This keeps "learned context" honest without a runtime
  consolidation engine.

Net: the runtime reads a small, verified, per-org context block. All the *learning* — what to
write, what to drop, what's stale — happens at build-time (weekly curation) and at a cheap
read-time verification check. No sleep-time agents, no temporal knowledge graph. Add that
complexity only with evidence of failure at scale.

---

## 7. Cost Management Over the Lifecycle

For ₹2L/yr-per-NGO budgets, token cost is a first-class lifecycle metric, not an
afterthought. The cautionary tale: one startup's infinite agent loop cost **$47,000 before
detection** — a failure traditional monitoring would have caught instantly
([ai-observability research](../research/ai-observability.md)).

### Track cost per org and per feature

`AgentTrace` already records `cost_usd`, `tools_used`, and `org` (§2). That gives, for free:

- **Cost per NGO** — roll up `AgentTrace.cost_usd` by `org`. Surfaces the org that's 10x the
  median (often one verbose tool dominating, e.g. a chatty `dalgo_get_flow_run_logs`).
- **Cost per feature** — tag traces by feature (Q&A vs pipeline-debug). Tells you when a
  feature isn't worth its token cost. If pipeline-debug costs more per use than the eng time
  it saves, that's a kill/redesign signal, not a model-upgrade signal.

### Levers (in lifecycle order)

1. **Prompt caching** — put static content (org schema summary, tool defs, system prompt) at
   the *start* of the prompt so it caches; put dynamic content (current question, errors) at
   the end. Cheapest, biggest win for a Q&A agent that re-sends the same schema every turn.
2. **Cache identical sub-results** — repeated `dalgo_list_schemas` / `dalgo_get_table_columns`
   within a session need not re-hit the warehouse or re-tokenize.
3. **Compress tool outputs** — the 4.2k-token `dalgo_get_flow_run_logs` from §2 should be
   summarized to the relevant error lines before entering context (cuts tokens *and* fights
   context rot).
4. **Model-tier tuning** — Checkr's 94% cost cut came from routing cheap-vs-expensive by need
   (§4 lever 5). Cheap tier for routing/classification; strong tier only for the final answer.
5. **Graduated budget alerts + circuit breakers** — alert at 50/80/95% of an org's monthly
   token budget; hard circuit-breaker on conversation length and cost-per-session. This is the
   guardrail that catches the next $47k loop.

All five are build-time/config decisions. None makes the runtime smarter; they make it
cheaper. Observability (§2) is what makes them *possible* — you can't cut what you can't see
(Checkr needed visibility *first*).

---

## 8. The Dev-Time Harness That Improves the Runtime Agent

This is where the sophistication lives. The broader **Dalgo engineering harness**
(`dalgo-core`) — its dev-time agents, skills, and CI — is the machinery that regression-tests
and improves the dumb runtime agent. The connection is direct:

```
   PRODUCTION                         DEV-TIME HARNESS (dalgo-core)
  (dumb runtime) ──traces/👎──▶  build-time agents triage failures (§3)
                                        │
                                        ▼
                              eval suite (Langfuse datasets, §2/§3)
                                        │
                                        ▼
                     CI gate on every agent-config change ───┐
                                        │                    │ if pass
                          regression gate (§5) ◀─────────────┘
                                        │
                                        ▼
                          ship new prompt_version (§5) ──▶ back to PRODUCTION
```

Concretely, the harness should:

- **Run the eval suite in CI** on every change to a prompt, tool description, or tool set —
  exactly as Discord runs evals as unit tests before deploy, and CircleCI runs LLM-as-judge in
  CI on every PR ([ai-observability research](../research/ai-observability.md)). The eval suite
  *is* the test suite for the agent.
- **Reuse the dev-time agents you already have.** The same `debugger` / planning agents in
  `dalgo-core` can triage a production failure cluster and draft the fix (a sharper tool
  description, a prompt tweak) as a PR — the human reviews and merges. This is "agents writing
  their own improved prompts after failure analysis," which cut task time **40%** in Anthropic's
  multi-agent system ([agent-design research](../research/agent-design.md)).
- **Treat the runtime agent's config as a versioned artifact in the repo**, so changes get the
  same review, CI, and rollback as any code change.

The asymmetry is the whole point: the **dev loop is allowed to be slow, expensive, and smart**
(LLM-as-judge, multi-agent triage, human review). The **runtime stays fast, cheap, and dumb.**
Every bit of intelligence the harness develops gets compiled down into a config edit the
runtime simply loads.

---

## 9. Team Practices for a Small Team

A small team can run this loop — but only if the rituals are lightweight and the alerts don't
cause fatigue.

### Ownership

- **One owner.** A single named **agent owner** holds the agent's quality, cost, and weekly
  review. Diffused ownership = no ownership. It can rotate, but it's one person at a time.

### The weekly review ritual (~30–45 min)

The heartbeat of the lifecycle. Once a week, the owner:

1. **Reviews the dashboard** — 👎-rate, refusal rate, cost-per-org, tool-selection mix,
   eval-suite pass rate. (4 numbers, not 40.)
2. **Triages the week's 👎 / corrections** — clusters failures, promotes 2+-occurrence
   patterns to eval cases (§3).
3. **Re-validates one org's curated context** (§6) for staleness — rotating, so every org gets
   refreshed over a cycle.
4. **Picks one improvement** — usually a §4 lever-1 tool-description fix — and opens a PR
   through the harness (§8).

This is a sustainable cadence: failures → evals → one fix per week compounds fast. "Once you
have even a modest feedback loop, quality improvements accelerate."

### Metrics to watch (and only these)

| Metric | Healthy signal | Alert when |
|--------|----------------|-----------|
| 👎-rate | trending down | jumps after a ship (rollback signal) |
| Refusal rate | low + stable | spikes (over-refusal kills trust) |
| Eval-suite pass rate | 100% on ship | any regression (hard CI gate) |
| Cost-per-org | flat | one org >3x median, or a loop |
| p95 end-to-end latency | within SLO | regression after a ship |

### Avoid alert fatigue

- **Page on two things only:** cost circuit-breaker tripped (a loop), and eval-gate failure on
  ship. Everything else is a dashboard the owner reads weekly, not a page.
- **Graduated, not binary** — 50/80/95% budget alerts (§7), so the 95% page is rare and means
  something.
- **Tune the alert set in the weekly review.** An alert that fires without action gets removed.

---

## 10. Phased Maturity Model — Crawl / Walk / Run

Concrete milestones, sized for Dalgo. Each phase ships a usable loop; don't skip to Run.

### 🐣 Crawl — ship one bounded agent + basic logging + a tiny eval set

*Goal: a working, observable loop on one narrow use case.*

- [ ] Ship **one bounded agent** — read-only data-Q&A over the Dalgo MCP. Read-only tool subset
      (`dalgo_get_*`, `dalgo_list_*`, `dalgo_search_docs`); **no** `create`/`delete` tools.
- [ ] Stand up **Langfuse self-hosted** (Docker/Helm). Trace every turn with `org_id`,
      `prompt_version`, tool calls, tokens, cost.
- [ ] Route all captured args through **kavach** for PII redaction before storage.
- [ ] Add **👍/👎 + correction** capture (`AgentFeedback` model).
- [ ] Write a **tiny eval set — 20–50 cases** from real questions (and any early 👎), stored in
      Langfuse. Run it **manually** before each change.
- [ ] One **named owner**.

*Exit criteria:* every production turn is traced and PII-safe; you can run 20 evals by hand;
one person owns it.

### 🚶 Walk — feedback loop + dashboards + regression gates

*Goal: the flywheel turns automatically; ships are gated.*

- [ ] **Weekly review ritual** live (§9): triage 👎 → promote patterns to eval cases.
- [ ] **Dashboards**: 👎-rate, refusal rate, cost-per-org, tool-selection mix.
- [ ] **Eval suite in CI** — runs on every prompt/tool change. **Regression gate** blocks ship
      on >5% quality drop or any newly-failing case (§5).
- [ ] **Online evals** — heuristic scorers on 100% (refusals, staging-table reads), LLM-as-judge
      on a 10–20% sample. *(This is the step that closes the 89%/44.8% gap.)*
- [ ] **Prompt versioning + one-click rollback** in Langfuse; pin model + judge versions.
- [ ] **Canary rollout to one org** via RBAC org-scoping (§5).
- [ ] **Graduated budget alerts** + cost circuit-breaker (§7).
- [ ] Per-org **curated context block** with read-time verification (§6, state-based).

*Exit criteria:* no config ships without passing evals; failures become eval cases within a
week; you can roll out to one org and roll back in seconds.

### 🏃 Run — multi-surface, automated eval-gated deploys

*Goal: the loop runs across surfaces with minimal human toil.*

- [ ] **Second agent surface** (e.g. pipeline-debug) on the same harness + eval infra.
- [ ] **Automated eval-gated deploys** — pass evals → auto-canary → staged → full, with
      automatic rollback on 👎/cost spike.
- [ ] **Harness drafts fixes** — dev-time agents triage failure clusters and open PRs for
      tool-description/prompt fixes; human reviews (§8).
- [ ] **Cost-per-feature** tracking → retire features that aren't worth their token cost (§7).
- [ ] **Model-tier routing** (cheap tier for routing, strong for answers).
- [ ] Memory only **if evidence demands it** — graduate from state-based to retrieval only when
      scale proves the simple version fails (§6). Likely never needed.

*Exit criteria:* shipping an agent improvement is as routine and safe as shipping code; the
runtime is still dumb; all the intelligence lives in the harness.

---

## The One-Paragraph Summary

Keep the **runtime agent dumb** — a fixed, deterministic loop over a read-only subset of Dalgo
MCP tools — and put **all the intelligence in a build-time lifecycle loop**: Observe production
traces (Langfuse + Django, PII-redacted via kavach), Evaluate them (eval suite + failure
triage, closing the 89%/44.8% tracing-vs-eval gap), Improve with the **cheapest lever first**
(tool descriptions and context beat model upgrades — 72%→90% from input examples, 49%→86% from
retrieval, not a bigger model), and Ship gated by evals with canary-to-one-org rollout and
instant rollback. A small team runs it on a weekly review ritual. Start at **Crawl**: one
bounded read-only agent, Langfuse tracing, 👍/👎 capture, and a 20-case eval set you run by hand.

---

### Sources

- [AI Observability research](../research/ai-observability.md) — tracing, the 89%/44.8% gap,
  cost attribution, production-failure-to-regression-test flywheel, Langfuse self-hosting.
- [Memory Systems research](../research/memory-systems.md) — state-based memory, GitHub Copilot
  citation verification, write-gating, Letta 74% filesystem result, Dalgo memory section.
- [Agent Design research](../research/agent-design.md) — dumb agents / smart orchestration, tool
  design (72%→90%), fewer tools, context rot, Anthropic self-improving prompts (40%).
- [LangChain State of Agent Engineering](https://www.langchain.com/state-of-agent-engineering)
- [Langfuse](https://langfuse.com/) · [self-hosting](https://langfuse.com/self-hosting) ·
  [prompt versioning](https://langfuse.com/docs/prompt-management/features/prompt-version-control) ·
  [datasets](https://langfuse.com/docs/evaluation/experiments/datasets)
- [Arize: from production traces to better agents — automating the LLMOps feedback loop](https://arize.com/blog/from-production-traces-to-better-ai-agents-automating-the-llmops-feedback-loop/)
- [ZenML: LLMOps in production case studies](https://www.zenml.io/blog/llmops-in-production-287-more-case-studies-of-what-actually-works)
- Dalgo `kavach` PII-guard (`kavach:status` skill) · Dalgo MCP tool surface (`dalgo_*`).
</content>
</invoke>
