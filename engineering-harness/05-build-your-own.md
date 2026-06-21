# Build Your Own Engineering Harness — and What a Small Team Should Actually Do

> Capstone synthesis. Prepared June 2026.
> Audience: a team deciding what to build first, not what's possible to build.

This is the last doc in the folder. The others did the legwork:

- **00 — Overview**: what a harness is, why it became a named discipline.
- **01 — Team case studies**: OpenAI Codex, Stripe Minions, Anthropic Research, LangChain, SWE-agent.
- **02 — Architecture patterns**: PEV loop, constraint layers, ACI, compaction, defense-in-depth.
- **03 — Environments & sandboxing**: isolation, fast feedback, devbox reuse.
- **04 — Evals & what wins**: pass@k vs pass^k, tokens-to-success, trace analysis.

This doc pulls those into **one reference architecture, one maturity ladder, and one
concrete plan for a small team** — the Dalgo lens. It is synthesis, not new research;
the deep evidence lives in `../research/harness-engineering.md`. Read that first if a
claim here surprises you.

---

## 1. The Core Thesis

**The harness is the binding constraint, not the model. The discipline is in the
scaffolding, not the code.**

Restated crisply: an agent is `Model + Harness`. The model is a stateless token
predictor — strong but spiky. The harness is everything that makes that prediction
reliable across real work: the loop, the tool interface, the feedback environment, the
verification gates, the memory, the safety enforcement. **At today's model capability,
the harness is where capability gets multiplied or wasted.** You cannot prompt your way
out of a bad harness, and you barely need to prompt at all inside a good one.

The strongest evidence, in order of how hard it is to argue with:

| Evidence | What changed | What stayed the same |
|---|---|---|
| **Edit-format experiment** (blog.can.ac) | Grok Code Fast 1: **6.7% → 68.3%** on coding; just switched patch format → hashline format | The model. The failure mode was mechanical text-matching, not reasoning. |
| **SWE-agent ACI** (NeurIPS 2024) | GPT-4 on SWE-Bench: **~2% → ~12%** from a purpose-built agent-computer interface | The model (GPT-4). |
| **LangChain deepagents** | Terminal Bench 2.0: **52.8% → 66.5%** (outside top 30 → top 5) | The model. Only middleware changed: doom-loop detection, a verification gate, a reasoning sandwich. |
| **mini-swe-agent** | A **100-line** harness scores **>74%** on SWE-Bench Verified | Beats far more complex harnesses. Minimal-but-right wins. |

Four independent teams, four single-variable demonstrations, all pointing the same way:
**move the harness, the number moves; hold the harness, swapping models barely helps.**

The corollary that makes this *actionable* rather than just true: harness quality is the
one thing a small team actually controls. You will not out-train a frontier lab's model.
You can absolutely out-engineer the average team's scaffolding — and because model
capability converges across providers while harness quality is team-specific
infrastructure, **the harness is the compounding asset.** Mitchell Hashimoto's
formulation is the whole discipline in one line: *"Anytime you find an agent makes a
mistake, you take the time to engineer a solution so that the agent never makes that
mistake again."*

---

## 2. A Reference Architecture

Six layers, least to most sophisticated. Each layer is built on the one below it. You
climb only when the layer beneath is solid and a real failure mode forces you up.

```
                    ENGINEERING HARNESS — LAYERED REFERENCE ARCHITECTURE

   ┌──────────────────────────────────────────────────────────────────────────┐
   │  L5  GUARDRAILS · MEMORY · EVALS                                           │
   │      mechanical hooks (pre/post-tool, policy) · persistent project memory  │
   │      regression-gated harness changes · eval suite (20–50 tasks)           │
   │      buys: the harness stops regressing; mistakes can't recur              │
   ├──────────────────────────────────────────────────────────────────────────┤
   │  L4  ORCHESTRATION   (add ONLY when forced)                                │
   │      subagents · pipelines/blueprints · parallel fan-out                   │
   │      buys: parallelism + context isolation for genuinely independent work  │
   ├──────────────────────────────────────────────────────────────────────────┤
   │  L3  VERIFICATION GATES                                                     │
   │      test-in-loop · pre-completion self-review · doom-loop detection       │
   │      buys: defeats victory-declaration bias; recovers compounding-error loss│
   ├──────────────────────────────────────────────────────────────────────────┤
   │  L2  TOOL / EDIT INTERFACE  (the ACI)                                       │
   │      reliable edits · curated tools · errors that prescribe the fix        │
   │      buys: the single largest measured swing (see §1); unlocks the model   │
   ├──────────────────────────────────────────────────────────────────────────┤
   │  L1  AGENT LOOP + FAST FEEDBACK ENVIRONMENT                                 │
   │      act → observe → repeat · tests/lint/typecheck IN the loop · <1 min     │
   │      buys: the agent can self-correct instead of guessing                  │
   ├──────────────────────────────────────────────────────────────────────────┤
   │  L0  RULES FILE + GOOD CONTEXT                                              │
   │      CLAUDE.md / AGENTS.md / .cursorrules · project map · conventions       │
   │      buys: cheap orientation; fewer wrong-path starts                      │
   └──────────────────────────────────────────────────────────────────────────┘
          ▲ more autonomy / more value, but only if every layer below holds ▲

   Cross-cutting (spans every layer): OBSERVABILITY — trace every tool call,
   track tokens-to-success. You cannot improve a layer you cannot see.
```

### Layer 0 — Rules file + good context

- **What it buys you.** Cheap orientation. The agent spends thousands of tokens just
  mapping your repo, tooling, and conventions before doing useful work (the "orientation
  tax"). A rules file pays that down once, persistently, and cuts wrong-path starts.
- **When to add it.** Day one. There is no excuse not to have it.
- **How to do it well.** Keep it under ~200–500 lines; sprawling files get ignored.
  Every rule concrete and actionable: *"Never run DROP without explicit confirmation"*
  beats *"be careful with the database."* Cover project context, conventions, behavioral
  rules, file structure. Reuse per-repo files so the agent reads the right one in the
  right place.

### Layer 1 — Agent loop + fast feedback environment

- **What it buys you.** The ability to self-correct. An agent that can run the tests,
  the linter, the type-checker *inside its loop* fixes its own mistakes instead of
  guessing. This is the single highest-leverage piece of infrastructure after the rules
  file — and it's mostly *reuse of your existing dev infra*, not new building.
- **When to add it.** Immediately after L0. A loop with no feedback environment is a
  blindfolded agent.
- **How to do it well.** The inner loop must be fast. OpenAI's Codex team held a **strict
  one-minute maximum**, cycling build tools (Make → Bazel → Turbo → NX) to keep it.
  Stripe pre-warmed devboxes to a 10-second start. Slow feedback is the silent killer:
  the agent stops checking its work because checking is expensive.

### Layer 2 — Tool / edit interface (the ACI)

- **What it buys you.** The largest measured performance swing in this entire document
  (§1). A bad edit format hides a capable model's ability entirely behind "exact string
  not found" errors. Good tools with clear, single-purpose descriptions stop the agent
  going "down completely wrong paths."
- **When to add it.** As soon as the agent is editing files or calling real tools — which
  is basically always. Anthropic's rule of thumb: *invest as much effort in the
  agent-computer interface (ACI) as you would in a human-computer interface (HCI).*
- **How to do it well.** Reliable edits over clever ones. Curated tool subsets, not a
  kitchen sink — 500 tools in context is actively harmful; progressive disclosure beats
  upfront exposure. Each tool single-purpose, marked destructive/idempotent/open-world.
  Errors must *prescribe the fix*: `use logger.info({event,...})` not `console.log`, not
  merely "violation detected."

### Layer 3 — Verification gates

- **What it buys you.** Defeats the two deadliest agent failure modes —
  **victory-declaration bias** (marking done without testing) and the **doom loop**
  (10+ variations on a broken approach). The math: at 85% per-step accuracy over 10
  steps, success is `0.85^10 ≈ 20%`. A verification gate after each major phase recovers
  most of that loss. **Verification beats generation.**
- **When to add it.** The moment you let an agent declare its own success. For any work
  that ships, this is non-negotiable.
- **How to do it well.** Three concrete gates, proven by LangChain:
  - *Test-in-loop / pre-completion checklist* — intercept the agent's exit, force a
    verification pass against the task spec before "done" is allowed.
  - *Self-review* — a final pass that re-reads the diff against acceptance criteria.
  - *Doom-loop detection* — track per-file edit counts; after N edits without progress,
    inject a reconsideration prompt. Bound the loop (Stripe caps CI retries at **two**
    cycles, then a human looks).

### Layer 4 — Orchestration (add only when forced)

- **What it buys you.** Parallelism and context isolation for genuinely independent
  subtasks. Anthropic's multi-agent research system beat single-agent by **90.2%** on
  research tasks.
- **When to add it.** Only when a task passes the **Anthropic test**: (a) it needs more
  than one context window, (b) it has genuinely parallel independent subtasks, or (c) it
  has deep tool integration that benefits from specialization. If none hold, use one
  agent. The cost is real: multi-agent ran **~15x** the tokens. For most coding work on a
  shared codebase, a single focused agent wins.
- **How to do it well.** Prefer Stripe's **blueprint** shape — alternate *deterministic*
  nodes (lint, CI, push) with *agentic* nodes (implement, fix). Determinism for what you
  can anticipate; AI for genuine unknowns. Give each subagent a concrete objective,
  explicit boundaries, an output format. Have subagents write to the filesystem directly
  rather than funneling everything through the orchestrator.

### Layer 5 — Mechanical guardrails, memory, evals

- **What it buys you.** The harness stops regressing, and fixed mistakes *stay* fixed.
  This is where a harness becomes a compounding asset instead of a pile of prose.
- **When to add it.** Once the harness is load-bearing — multiple people or many runs
  depend on it — and prose rules are demonstrably being ignored.
- **How to do it well.**
  - *Mechanical hooks/policy* — enforce in code what prose can't. A `pre-tool` hook that
    blocks `DROP`/destructive ops is enforcement; a CLAUDE.md line saying "be careful" is
    a suggestion the agent can rationalize past. Make taste a **hard CI failure**, not a
    warning; disable inline suppressions so the agent can't `# noqa` its way through.
  - *Memory* — persistent project knowledge, but couple it to just-in-time verification.
    **Stale-but-confident memory is more dangerous than no memory** — verify a remembered
    claim against current state (grep/read/API) before acting on it.
  - *Evals + regression gates* — 20–50 tasks, not thousands; effect sizes are large
    early. Gate harness changes on the eval so a "fix" can't silently break three other
    things. This closes the Hashimoto loop mechanically.

---

## 3. The Maturity Ladder

Three rungs. Each is a coherent, shippable state — not a checklist you must finish before
getting value. Most small teams should aim for solid **Walk** and stop there.

```
   CRAWL ─────────────────▶ WALK ─────────────────▶ RUN
   one good workflow         verified + specialized   self-improving harness
   (L0–L1, partial L2)       (full L2–L3, L4, L5a)    (full L5, L4 pipelines)
```

### CRAWL — rules file + tests-in-loop + one good workflow

**Goal:** one repeatable task the agent does reliably, end to end.

Milestones:
- [ ] A rules file per repo (L0), under 500 lines, concrete rules only.
- [ ] The agent runs tests/lint/typecheck *in its loop* (L1); inner loop under ~1 min.
- [ ] A reliable edit/tool interface — edits land, errors are readable (L2, partial).
- [ ] One documented workflow (e.g. "implement a planned feature") that works start to
      finish without hand-holding.
- [ ] Tool calls are logged. You can read back what the agent actually did.

**Exit criterion:** the same task succeeds twice in a row, unattended, and you can trace
why when it doesn't.

### WALK — verification gate + subagents + evals

**Goal:** the agent's "done" is trustworthy, and the harness specializes where it pays.

Milestones:
- [ ] A **pre-completion verification gate** (L3) — no task is "done" until tests/lint
      pass against the spec. *(This is the highest-ROI single addition past Crawl.)*
- [ ] **Doom-loop detection** (L3) — bounded retries, reconsideration prompt on stall.
- [ ] **Self-review pass** (L3) — final diff re-read against acceptance criteria.
- [ ] Subagents for genuinely independent work only (L4), gated by the Anthropic test.
- [ ] A small **eval suite** (L5a) — 20–50 tasks, run before and after harness changes.
- [ ] **Tokens-to-success** tracked per workflow, not just pass/fail.

**Exit criterion:** you trust the agent to merge low-risk work behind the gate, and a
harness change that regresses quality gets caught by the eval, not by a user.

### RUN — mechanical hooks + orchestration + regression-gated harness changes

**Goal:** the harness enforces itself and improves itself.

Milestones:
- [ ] **Mechanical hooks** (L5) replace prose rules wherever a rule is load-bearing —
      pre-tool blocks on destructive ops, post-tool auto-lint feedback, policy gates.
- [ ] **Taste invariants as hard CI failures**, suppressions disabled.
- [ ] **Pipelines/blueprints** (L4) — deterministic + agentic nodes for multi-step flows.
- [ ] **Persistent memory** with just-in-time verification (L5).
- [ ] **Regression-gated harness changes** — every harness edit runs the eval; trace
      analysis (auto-fetch traces → cluster failures → propose harness fix) feeds the
      Hashimoto loop.

**Exit criterion:** when the agent makes a new class of mistake, you encode a guard so it
can't recur — and the eval proves the guard didn't break anything else.

---

## 4. Cross-Cutting Lessons

These recur across **every** team in the case studies. They are the load-bearing
generalizations.

**1. Simplest thing that works; climb the ladder only when forced.**
The 100-line mini-swe-agent beats complex harnesses. Anthropic: add complexity *"only
when it demonstrably improves outcomes."* Every layer you add is latency, tokens, and a
new thing to debug. Add it in response to an *observed* failure mode, never speculatively.

**2. Good for humans is good for agents.**
Stripe's deepest lesson: they reused their existing human devbox infra (fast feedback,
isolated environments, standard tooling) and it directly enabled agents. Fast tests,
clear errors, good linters, reproducible environments — the things that make *you*
productive make the *agent* productive. You probably already have most of L1; wire it
into the loop before you build anything new.

**3. Verification beats generation.**
Agents declare premature success — this is structural, not a model bug. The compounding-
error math (`0.85^10 ≈ 20%`) means a generator without a verifier degrades fast over long
horizons. A cheap gate that *checks* recovers more reliability than a smarter model that
*generates*. If you build one thing past Crawl, build the gate.

**4. The edit/tool interface matters more than the model.**
6.7% → 68.3% from an edit-format change. ~2% → ~12% from an ACI. The model wasn't failing
at reasoning; it was failing at mechanical text-matching the harness imposed on it. Spend
ACI effort proportional to HCI effort.

**5. Context engineering is the dominant lever.**
Token usage explains ~80% of multi-agent performance variance. Context *rot* is
structural — middle-of-window information is attended to less, so longer windows don't
mean better access. Reload relevant context at each major phase; compact aggressively;
prefer symbol/semantic indexing over file-grepping (reported ~120x token reduction).
What's in the window beats how big the window is.

**6. Determinism where it counts, autonomy where it's safe.**
Stripe's blueprint: fixed code for what you can anticipate (lint, CI, push), AI for
genuine unknowns. The market moved decisively from "fully autonomous" to "autonomous
within explicit boundaries." Every increase in autonomy must be matched by an increase in
observability — autonomy you can't trace is autonomy you can't trust.

**7. Tokens-to-success is the real cost metric.**
Not pass/fail, not latency alone. An agent that "succeeds" at 10x the tokens is a
liability at scale — autonomous loops amplify token use 10–100x over chat. Measure cost
*per success*, and measure **pass^k**, not just pass@1, for anything that must work every
time: a 75% per-trial rate is pass@3 ≈ 98% (fine if a human picks one good PR) but
pass^3 ≈ 42% (not a product you can ship unattended).

---

## 5. Anti-Patterns

What teams reliably get wrong. Each is the shadow of a lesson above.

| Anti-pattern | What it looks like | The fix |
|---|---|---|
| **God agent / tool kitchen-sink** | One agent with 50+ tools loaded upfront; descriptions overlap; agent picks wrong paths | Curated subsets, progressive disclosure, single-purpose tools |
| **Multi-agent for sequential work** | Spawning subagents for steps that depend on each other; 15x tokens, brittle, undebuggable | Single agent unless the Anthropic test passes; orchestrate only truly parallel work |
| **No verification gate** | Agent says "done," tests never ran; defects ship; "it worked in the demo" | Pre-completion checklist gate (L3) — the cheapest high-ROI thing you can add |
| **Slow environment** | 5-minute test runs; agent stops checking because checking is expensive; guesses instead | Get the inner loop under a minute; reuse human dev infra |
| **Trusting benchmark scores** | Picking a model on its leaderboard number; ignoring that the *harness* moves the number more | Build a 20–50 task eval on *your* tasks; measure tokens-to-success and pass^k |
| **Premature orchestration** | Building a multi-agent swarm before a single agent reliably does one task | Make Crawl solid first; most teams never need L4 |
| **Rules-as-prose instead of mechanical enforcement** | "Be careful with the DB" in CLAUDE.md; agent rationalizes past it; the line is a suggestion | A hook that *blocks* the op (L5); hard CI failures, suppressions disabled |
| **Spec drift** | Long runs with no living spec to verify against; agent solves the wrong problem thoroughly | Maintain a structured plan/spec artifact the verifier checks against |
| **Stale-but-confident memory** | Agent acts on a remembered fact that's now wrong; takes a destructive action on a false belief | Couple memory to just-in-time verification before acting |

The meta-anti-pattern: **building up the ladder before the lower rungs hold.**
Orchestration on top of a flaky ACI multiplies flakiness. Evals on top of no verification
gate measure noise. Climb in order.

---

## 6. A Pragmatic Plan for a Small Team — the Dalgo Lens

Dalgo is the test case for "small team, real constraints": an open-source data platform
for ~20 partner NGOs, small engineering team, non-technical users on slow connections, an
AGPL codebase split across `DDP_backend` (Django/Ninja), `webapp_v2` (Next.js), and
`prefect-proxy` (FastAPI), orchestrated from `dalgo-core`.

### Where Dalgo already is on the ladder

The existing `dalgo-core` harness is **further along than most teams** — it has earned
the right to skip ahead in places:

- **L0 (rules/context): strong.** Per-repo `CLAUDE.md` files, a documented dev workflow,
  the `plain-writing` skill, a domain map referenced for blast-radius analysis. The model
  reads the right rules in the right repo.
- **L2 (workflow surface): unusually mature.** A full command pipeline
  (`/product/write-spec` → `/design/design-handoff` → `/engineering/plan-feature` →
  `/engineering/execute-plan`), specialized agents (`debugger`, `senior-product-manager`,
  `ux-design-expert`, `ngo-data-platform-consultant`), and evaluation-lens skills
  (`design-review`, plus `backend-architecture` / `frontend-architecture` per CLAUDE.md).
- **L4 (orchestration): partially present, and appropriately so.** Subagents exist as
  *evaluation lenses* and *role specialists*, not autonomous swarms — exactly the
  single-well-scoped-agent-with-human-checkpoints pattern the evidence favors.
- **Resume/checkpoint discipline:** the `executing-feature-plans` skill branches first,
  tracks progress in `tasks.md`, and resumes after interruption — a real harness feature
  most teams lack.

### The gaps — confirmed, not guessed

Two are confirmable from the repo itself:

1. **No mechanical hooks (L5).** `dalgo-core/.claude/settings.json` has a `permissions`
   block (allow/deny lists, `additionalDirectories`) but **no `hooks` block at all.**
   Every behavioral rule today lives as *prose* in CLAUDE.md and skills — i.e. as
   suggestions the agent can rationalize past. This is the rules-as-prose anti-pattern in
   its purest form. The deny-list does block reading `.env*` files, which is good, but
   that's a permission gate, not a feedback or verification hook.

2. **No verification gate in the loop (L3).** The `executing-feature-plans` skill
   *describes* red-green-refactor and "one test at a time," but nothing **mechanically
   intercepts the agent's "done"** to force tests/lint to pass before a task is marked
   complete. The discipline is documented; it is not enforced. (The `superpowers:
   verification-before-completion` skill exists as a lens, but a lens is opt-in prose, not
   a gate.)

One is an inference (the dalgo-core `docs/harness-evolution.md` referenced in CLAUDE.md is
not present on disk, so this maps to its *spirit* as stated in the task): the harness has
no **auto-lint feedback loop** and no **eval/regression suite** — so a change to a command
or skill can silently degrade output with nothing to catch it.

### What to build — first, second, third

Dalgo is at **late-Crawl / early-Walk**: excellent L0–L2, real L4 specialization, but the
**L3 verification gate and L5 enforcement are missing** — and those are exactly the rungs
the evidence says deliver the most reliability per unit of effort. The plan follows the
ladder, not ambition.

**FIRST — the verification gate + auto-lint feedback loop (L3 + L1 completion).**
*The single highest-ROI addition. Do this before anything else.*

- Add a **pre-completion check** to the `execute-plan` flow: before any task in `tasks.md`
  is marked done, run the repo's actual test + lint + typecheck commands (each repo's
  CLAUDE.md already names them) and refuse "done" on failure. This is LangChain's
  `PreCompletionChecklistMiddleware` pattern, adapted to a skill-driven harness.
- Wire **lint/typecheck into the loop as prescriptive feedback** (L1) — surface the
  linter's fix, not just the violation, back to the agent. Django (ruff/black) and Next.js
  (eslint/tsc) both already produce structured output; this is reuse, not new infra.
- *Why first:* it directly kills victory-declaration bias, which on a small team with
  limited reviewer attention is the failure mode that erodes trust fastest. Verification
  beats generation, and the NGO trust budget is thin — one bad shipped change costs more
  here than a slow loop.

**SECOND — mechanical hooks for the irreversible stuff (L5, safety slice first).**
*Convert the load-bearing prose rules into enforcement.*

- Add a `hooks` block to `settings.json` with a **pre-tool hook** that hard-blocks (or
  forces explicit human confirmation on) destructive/irreversible operations — exactly the
  Dalgo MCP verbs the research doc flagged: `sync_sources`, `trigger_pipeline_run`,
  `run_dbt`, `delete_*`, `publish_changes`, and any raw SQL `DROP`. Mark these as
  destructive at the tool boundary.
- Add a **post-edit hook** that auto-runs the formatter/linter on touched files and feeds
  results back — making FIRST's feedback loop mechanical rather than skill-dependent.
- *Why second:* the user base makes irreversible actions (touching NGO data, triggering
  real pipelines, costing money) the highest-consequence risk. The guidance is explicit:
  put human approval gates on anything irreversible, data-touching, or costly. A hook
  enforces that; a CLAUDE.md sentence does not. *Build the safety slice of L5 before the
  eval slice* — it protects users now, where evals protect quality later.

**THIRD — a small eval + regression gate for the harness itself (L5, eval slice).**
*Make the harness improvable without fear of regression.*

- Build **20–50 tasks** drawn from real Dalgo work — a representative spec-to-PR, a
  bugfix from a Sentry trace, a dbt transform change, a frontend component. Not thousands;
  effect sizes are large early.
- Run the eval **before/after any change** to a command, agent, or skill, and track
  **tokens-to-success** per workflow, not just pass/fail. This is the regression gate that
  lets you evolve the harness confidently — and it closes the Hashimoto loop: when the
  agent makes a new mistake, encode a guard *and* prove via the eval it broke nothing else.
- *Why third:* it has the longest build time and the least immediate user-facing payoff,
  but it's what turns the harness from a static artifact into a compounding asset. It only
  pays off once FIRST and SECOND have given you something worth protecting.

### What NOT to build (the small-team discipline)

- **No autonomous multi-agent swarms.** Dalgo's existing subagents-as-lenses pattern is
  correct. Resist "agent that does the whole feature unattended" — it fails the Anthropic
  test for most Dalgo work (shared codebase, sequential steps) and costs ~15x tokens a
  small budget can't absorb.
- **No custom sandbox infra yet.** Worktree isolation (`.dalgo-worktrees`, already wired
  into `additionalDirectories`) plus per-repo dev environments are enough at this scale.
  Firecracker microVMs are a frontier-lab answer to a problem Dalgo doesn't have.
- **No framework migration.** The skill/command harness *is* the harness. Don't rebuild it
  on LangGraph/CrewAI to chase patterns you can add directly.

### The Dalgo plan in one line

> **Gate "done" on real tests (FIRST), hook-block the irreversible MCP/SQL ops (SECOND),
> then stand up a 20–50 task eval to regression-gate the harness itself (THIRD)** —
> climbing from late-Crawl to solid Walk, in ladder order, building each rung only because
> the rung below now holds.

---

## Sources

Primary, cited inline:

- [Building Effective AI Agents — Anthropic Engineering](https://www.anthropic.com/engineering/building-effective-agents) — workflows vs. agents; *"add complexity only when it demonstrably improves outcomes"*; invest in the ACI as much as the HCI.
- [How we built our multi-agent research system — Anthropic Engineering](https://www.anthropic.com/engineering/multi-agent-research-system) — the multi-agent test; 90.2% gain / 15x token cost; tokens explain ~80% of variance.
- [Demystifying Evals for AI Agents — Anthropic Engineering](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — pass@k vs pass^k; start with 20–50 tasks.
- [Harness engineering: leveraging Codex in an agent-first world — OpenAI (Lopopolo)](https://openai.com/index/harness-engineering/) — the one-minute inner loop; taste invariants as hard CI failures; the Hashimoto loop.
- [Improving Deep Agents with Harness Engineering — LangChain](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering) — 52.8% → 66.5%; pre-completion checklist, doom-loop detection, reasoning sandwich.
- [Minions: Stripe's one-shot end-to-end coding agents (Part 2) — Stripe](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2) — blueprint architecture; bounded iteration; *"what's good for humans is good for agents."*
- [I Improved 15 LLMs at Coding in One Afternoon. Only the Harness Changed. — blog.can.ac](https://blog.can.ac/2026/02/12/the-harness-problem/) — 6.7% → 68.3% from an edit-format change.
- [SWE-agent: Agent-Computer Interfaces — NeurIPS 2024](https://arxiv.org/abs/2405.15793) — ~2% → ~12% from the ACI; mini-swe-agent.

Full evidence base, terminology, and the complete reading list:
`../research/harness-engineering.md`.
