# Orchestration Workflows for the Dalgo Engineering Harness

**Status:** Research / proposal
**Scope:** How to orchestrate the Dalgo pipeline (spec → design → plan → build → review)
with multi-agent patterns, parallel dispatch, background/scheduled agents, and durable
state files.
**Audience:** Whoever maintains the harness (`dalgo-core`), not application engineers.

This doc combines (a) the current Dalgo pipeline and command/agent definitions with
(b) the latest Anthropic / Claude Code documentation on subagents, parallel execution,
headless/background mode, scheduled agents, and the Agent SDK. Every external claim is
cited inline.

---

## 1. Feature reference — orchestration patterns and Claude Code primitives

### 1.1 The five workflow patterns (Anthropic's vocabulary)

Anthropic's *Building Effective Agents* distinguishes **workflows** (LLM steps wired
together on predefined paths — deterministic, you control the control flow) from **agents**
(the model decides its own steps dynamically). The five composable patterns:

1. **Prompt chaining** — fixed sequence; each step's output feeds the next. Add a
   programmatic *gate* between steps to catch failures early. This is exactly our pipeline:
   spec → plan → build → validate.
2. **Routing** — classify the input, send it down a specialized path (e.g. UI feature vs
   backend-only → different downstream stages). Our design gate is a router.
3. **Parallelization** — two flavors:
   - **Sectioning** — split into independent subtasks run concurrently, aggregate after
     (e.g. backend slice + frontend slice; or review-for-security + review-for-design +
     review-for-NGO-usability).
   - **Voting** — run the *same* task several times for higher confidence (e.g. three
     independent reviewers, flag anything ≥1 flags).
4. **Orchestrator–workers** — an orchestrator decides at runtime which subtasks to spawn;
   subtasks are *not* predefined. Use when you can't enumerate the work up front.
5. **Evaluator–optimizer** — a generator produces, an evaluator critiques against criteria,
   loop until it passes. Our validate-loop and design-review-loop are this pattern.

Source: <https://www.anthropic.com/research/building-effective-agents>

**Key principle from Anthropic:** start with the simplest thing that works. *"The most
successful implementations use simple, composable patterns rather than complex frameworks."*
A single well-prompted agent beats a multi-agent maze for most tasks. Multi-agent
implementations typically use **3–10× more tokens** than single-agent ones (research-style
fan-out can hit ~15×).
Source: <https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them>

### 1.2 Deterministic pipeline vs model-driven delegation

| | Deterministic pipeline (workflow) | Model-driven delegation (agent) |
|---|---|---|
| Control flow | You hardcode the stages | Orchestrator picks stages at runtime |
| Predictability | High — same path every time | Lower — adapts to the input |
| Best for | Well-understood, repeatable work (our pipeline) | Open-ended research where subtasks can't be enumerated |
| Failure mode | Rigid — handles only the cases you wired | Drifts, over-spawns, hard to debug |

Our spec→build→review pipeline is a **known, repeatable shape** → it should stay a
**deterministic workflow** with explicit gates, not a free-roaming orchestrator. Reserve
model-driven delegation for the genuinely open-ended step inside it: blast-radius discovery
and codebase research, where the planner can't know in advance how many surfaces to explore.

### 1.3 Fan-out / fan-in (the orchestrator–worker pattern in practice)

Anthropic's Research system uses a **lead agent** that analyzes the query, spawns subagents
to explore facets **in parallel**, and synthesizes their curated results. It beat
single-agent Opus by **90.2%** on their internal research eval and cut research time ~90% on
complex queries — at ~15× the tokens. Fan-out wins because the work is *embarrassingly
parallel* (comb hundreds of sources) and each worker needs **minimal shared context**.
Source: <https://www.anthropic.com/engineering/multi-agent-research-system> ·
<https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them>

**The load-bearing condition: low shared context.** Fan-out helps when each worker can do
its slice without the others' intermediate state. It *hurts* when subtasks are tightly
coupled and constantly need each other's in-progress decisions — Anthropic explicitly warns
against **problem-centric decomposition** ("one agent writes code, another tests it"), which
creates coordination overhead and "telephone game" information loss at every handoff.

This is precisely why `executing-feature-plans/SKILL.md` keeps implementation **inline**:
an endpoint and the UI consuming it share an API contract that must stay in working memory.
Splitting that across agents pays ~15× tokens to *lose* the contract.

### 1.4 Pipeline vs barrier; verify-then-proceed gates

- **Pipeline (streaming):** stage N+1 starts as soon as stage N produces *its* output.
- **Barrier (join):** wait for *all* parallel workers to finish before proceeding. Fan-in
  is a barrier — synthesis can't start until every worker reports.
- **Verify-then-proceed gate:** between stages, a cheap check decides go / no-go / retry.
  Our `pipeline.md` status table is the gate ledger; the validate and design-review loops
  are the gates. Anthropic: a verification subagent works well because *"verification
  requires minimal context transfer by nature"* — the verifier needs the criteria and the
  diff, not the full build history.
  Source: <https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them>

### 1.5 Adversarial / independent verification

The strongest quality lever: a **fresh agent that did not write the code** checks it against
explicit criteria. Two cautions from Anthropic:

- Verifiers *"commonly mark outputs as passing without thorough testing."* **Require the
  verifier to run the full suite and paste real output** — evidence, not assertion. (This
  matches the repo's own `superpowers:verification-before-completion` skill.)
- The verifier must be **independent** — clean context, no memory of the rationalizations
  the builder made. A fresh subagent spawn gives you that for free.

### 1.6 Claude Code primitives (the building blocks)

| Primitive | What it is | Use for | Doc |
|---|---|---|---|
| **Subagent** (`Task`) | A fresh Claude in its own context window, own tools/model, returns **only a summary**. Works within one session; **subagents can't talk to each other** mid-flight. | Read-only bookends (explore, review), verification, fan-out research | <https://code.claude.com/docs/en/sub-agents> |
| **Parallel subagents** | Multiple `Task` calls dispatched together run concurrently (as of Apr 2026 subagents + MCP init in parallel, cutting startup). Costs ~Nx tokens for N agents. | Independent sections (multi-lens review) | <https://code.claude.com/docs/en/sub-agents> |
| **Background agents** (`claude agents`) | Many **independent full sessions** running without a terminal, monitored from one screen ("Needs input / Working / Completed"). | Long pipelines you don't want tying up your session; several features at once | <https://code.claude.com/docs/en/agent-view> |
| **Agent teams** | Multiple sessions that **communicate** with each other. | Genuinely collaborative multi-session work | <https://code.claude.com/docs/en/agent-teams> |
| **Headless / `-p`** | Run non-interactively from CLI; `--output-format json` / `stream-json`; `--bare` skips auto-discovery for reproducible CI; resume with `--resume <session_id>`. | CI/CD, scripted gates, pre-commit linters | <https://code.claude.com/docs/en/headless> |
| **Agent SDK** | Same harness as Claude Code, as Python/TS packages — structured output, tool-approval callbacks, message objects. | Programmatic orchestration outside the terminal | <https://code.claude.com/docs/en/agent-sdk/overview> |
| **Scheduled tasks / Routines** | Cloud agents on cron syntax (`cron_create` / `cron_list` / `cron_delete`), run on Anthropic infra independent of your laptop; daily caps per plan tier. Also `/schedule` and `/loop` in-session. | Recurring maintenance: doc gardening, stale-PR triage | <https://code.claude.com/docs/en/scheduled-tasks> |

**Three orchestration tiers, smallest first:**
*subagents* (one session, summaries back to parent) → *background agents* (independent
sessions, one dashboard) → *agent teams* (sessions that talk). Pick the smallest that fits.
Source: <https://code.claude.com/docs/en/sub-agents> (the Note distinguishing all three).

---

## 2. Current state in the Dalgo harness

### 2.1 How the pipeline is orchestrated today

Two parallel realities exist in this repo:

**Live (`.claude/commands/` on `feature/rbac`):** the pipeline is a set of **standalone
slash commands** the human runs one at a time — `/product/write-spec`,
`/design/design-handoff`, `/engineering/plan-feature`, `/engineering/execute-plan`,
`/engineering/validate-spec`, `/engineering/review-pr`. There is **no single orchestrator**
and **no `pipeline.md`**. The only durable state is:
- `tasks.md` — the build checkpoint (`execute-plan.md` and `executing-feature-plans/SKILL.md`
  create/resume from it).
- The feature-folder artifacts (`spec.md`, `plan.md`, `research.md`, `design.md`).

`execute-plan` builds **inline, one session, red-green-refactor**, and uses subagents only
as **read-only bookends** (Explore before, code-review after) — a deliberate, well-justified
choice that matches Anthropic's "don't split tightly-coupled coding across agents" guidance.

**Experimental (in `.claude/worktrees/*`):** a more advanced orchestration already exists but
isn't promoted to the live tree:
- `commands/engineering/ship-feature.md` — **command mode**: one orchestrator runs the
  whole pipeline *in the current session*, spawns a sub-agent per stage, reads **only
  `pipeline.md`** between stages, writes outcomes back immediately, stops at blockers.
- `commands/engineering/ship-feature-bg.md` — a thin launcher that spawns…
- `agents/ship-orchestrator.md` — **agent mode**: same pipeline, but a **fresh isolated
  context**, purely state-driven from `pipeline.md`, runnable in the background.
- `agents/planner.md` + an `engineer` agent — the stage workers.

Both modes write a shared `pipeline.md` with a stage-status table, `Mode:` field, and
counters (`Validate attempts`, `Design review attempts`, `Human interventions`). This is the
**durable orchestration backbone** the live tree lacks.

### 2.2 Experiment 0 — command-mode vs agent-mode (engaging directly)

`docs/harness-evolution.md` frames the open question: does orchestrator mode affect output
quality? The trade-off, in Claude Code's own terms:

| | Command mode (`ship-feature`) | Agent mode (`ship-feature-bg` → `ship-orchestrator`) |
|---|---|---|
| Context | Carries the current session's conversation | **Fresh isolated context**, zero prior memory |
| State source | `pipeline.md` (but session context bleeds in) | `pipeline.md` **only** (clean-start determinism) |
| UX | Human sees every step, natural pause points | Runs in background via `claude agents` / `-p` |
| Risk | **Session drift** — a decision made 50 messages ago, now stale, leaks into a stage | No drift; but no benefit from a clarification the user gave earlier |
| Resumable | Yes (re-run reads `pipeline.md`) | Yes (re-run reads `pipeline.md`) |

**The docs settle the mechanism, not the verdict.** Anthropic positions subagents/background
agents as **context-protection** tools: *"keeping exploration and implementation out of your
main conversation"*, each subagent *"clean and focused."* That is the agent-mode advantage —
the orchestrator can't make a stale-context error because it has **no** prior context. The
command-mode advantage (carrying a just-given clarification) is real but small, and is better
served by writing the clarification into the spec/`pipeline.md` than by relying on session
memory. **Recommendation below: default to agent-mode orchestration; the orchestrator should
be a context-light coordinator regardless of mode.** Both modes already enforce the right
discipline — *"between stages, read only `pipeline.md`"* — which is what neutralizes drift.

The experiment's result fields in `harness-evolution.md` are still empty. The single most
valuable thing is to **run it once each way** on comparable features and fill the table; the
metrics it proposes (human interventions, validate attempts, blast-radius misses, time-to-PR)
are the right ones.

### 2.3 Gaps

1. **No live orchestrator.** The advanced `ship-feature` / `ship-orchestrator` /
   `pipeline.md` machinery sits in worktrees, not in `.claude/commands` on the working
   branch. The pipeline is currently human-driven, command-by-command.
2. **No `pipeline.md` in the live flow.** `tasks.md` checkpoints the *build* stage only;
   there's no cross-stage ledger, so resume is per-command, not per-pipeline.
3. **No parallel dispatch anywhere.** Even where work is independent (backend vs frontend
   slices that *don't* share a contract; multi-lens review), everything is sequential.
4. **Verification is single-lens and not adversarial.** `validate-spec` and `review-pr`
   exist, but there's no spawn of *independent* reviewers checking each other, and no
   enforced "paste the real test output" rule at the gate.
5. **No background / scheduled use.** Iteration 4's doc-gardening agent and the backlog's
   stale-PR triage are described but not wired to `/schedule` or Routines.
6. **Self-review gate (Iteration 2) not built.** No agent reviews its own diff before a human
   sees the PR.

---

## 3. Recommendations

### P1 — high value, low risk

**P1.1 — Promote the `pipeline.md` orchestrator to the live tree.**
Move `ship-feature.md`, `ship-feature-bg.md`, and `ship-orchestrator.md` out of the
worktrees into `.claude/commands/engineering/` and `.claude/agents/`. This gives one
resumable command for the whole spec→PR pipeline with a durable state ledger. Keep the
existing per-stage commands for manual control. The orchestrator rule that makes this safe is
already written: **between stages read only `pipeline.md`; spawn a fresh sub-agent per stage;
write the outcome back immediately.**

**P1.2 — Default the orchestrator to agent-mode (fresh context).**
Per §2.2, make `ship-feature-bg` (spawns `ship-orchestrator` with a clean context) the
recommended path, and have command-mode `ship-feature` enforce the same "read only
`pipeline.md` between stages" discipline so it can't drift. Capture any user clarification by
**writing it into the spec or `pipeline.md`**, never by relying on session memory. Then run
Experiment 0 once each way and record the table in `harness-evolution.md`.

**P1.3 — Add an adversarial self-review gate (Iteration 2) — and make it independent.**
Insert a stage between `validate` and `docs`: spawn a **fresh** review sub-agent over
`git diff main...HEAD` that did not write the code. It must:
- run the **full** backend + frontend suites and **paste the actual output** (counter the
  "marks passing without testing" failure mode);
- check the planted-violation set (missing `@has_permission`, hardcoded hex, `console.log`,
  org-scoping) named in `harness-evolution.md`;
- fix blocking findings, loop ≤2×, record `Self-review attempts` in `pipeline.md`.

Example orchestrator step:
```
Spawn fresh `reviewer` sub-agent (clean context):
  Input:  git diff main...HEAD  +  docs/SECURITY.md criteria
  Task:   Run full test suites, PASTE output. Flag blocking violations.
          Fix them. Report findings + evidence. Do NOT claim pass without pasted output.
```

**P1.4 — Make the build's read-only bookends explicitly parallel.**
`executing-feature-plans` already allows an Explore subagent (before) and a code-review
subagent (after). Keep implementation inline (correct per Anthropic), but **dispatch the
orientation explorers in parallel** — e.g. one mapping the backend pattern, one mapping the
frontend pattern — in a single batch of `Task` calls, fan-in their `file:line` pointers.
This is true sectioning: independent, low-shared-context, read-only.

### P2 — higher value, more design

**P2.1 — Parallelize review across lenses (sectioning + voting).**
At the review stage, fan out **independent** reviewers concurrently, each with a narrow
criteria set, then fan-in (barrier) and aggregate:
- security / multi-tenancy (`docs/SECURITY.md`)
- design + NGO-usability (`design-review` skill + Priya lens) — frontend diffs only
- correctness / test-evidence (run the suites)

Each is low-shared-context (needs the diff + its own criteria), so fan-out is a clean fit and
parallel dispatch is genuinely faster. Aggregate by **union of blocking findings** (voting:
if any lens blocks, the gate blocks). Caution from the docs: N reviewers ≈ N× tokens, and for
sub-minute tasks the split overhead isn't worth it — only fan out when each lens has real
work.
Source: <https://code.claude.com/docs/en/sub-agents>

**P2.2 — Parallelize build only across *truly decoupled* slices.**
Default stays inline. But when the plan contains slices with **no shared API contract**
(e.g. a backend-only migration and an unrelated frontend copy change), the orchestrator may
dispatch them as separate **background agents** (`claude agents`) in separate worktrees,
fan-in at validate. The hard rule from `executing-feature-plans` and Anthropic both: **never
split a single endpoint+UI contract across agents.** Decoupling must be proven from the plan,
not assumed.
Sources: <https://code.claude.com/docs/en/agent-view> ·
<https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them>

**P2.3 — Two-PR fan-in as an explicit barrier.**
The skill already mandates backend-PR-first with cross-links. Encode this in `pipeline.md` as
a barrier: `pr-backend` and `pr-frontend` are separate rows; the frontend PR row stays
`blocked` until the backend PR row is `complete`, and the orchestrator states merge order in
both bodies.

### P3 — automation / scheduled

**P3.1 — Doc-gardening Routine (Iteration 4) on cron.**
Wire a weekly cloud Routine (`cron_create`, or `/schedule`) that scans `docs/` and the code
repos for drift and un-issued TODOs and opens a cleanup PR. Cloud Routines run on Anthropic
infra regardless of whether a laptop is on; mind the per-tier daily run cap. Test per
`harness-evolution.md`: plant a TODO with no issue reference, confirm the next run surfaces it.
Source: <https://code.claude.com/docs/en/scheduled-tasks> ·
<https://claude.com/blog/whats-new-in-claude-managed-agents>

**P3.2 — Stale-PR triage (backlog) on cron.**
A daily Routine that flags Dalgo PRs older than 48h, pings the author, and posts a summary.
Read-heavy, low-context, independent runs — an ideal headless/scheduled job.

**P3.3 — Headless `-p` gates in CI.**
Use `claude -p --bare --output-format json` as a CI step (e.g. a typo/security pass over
`git diff main`, piped so no Bash permission is needed). `--bare` skips auto-discovery so CI
gets the **same result on every machine**; parse `total_cost_usd` from the JSON to track spend
per run. This is the mechanical-enforcement complement to Iteration 3's linters.
Source: <https://code.claude.com/docs/en/headless>

**P3.4 — `/loop` for status polling.**
For "watch this background pipeline until it needs me" use `/loop` (self-paced) rather than
foreground sleeps, surfacing only when a stage hits a blocker.
Source: <https://code.claude.com/docs/en/scheduled-tasks>

### State files as the resumable backbone (cross-cutting)

`pipeline.md` (orchestrator state) and `tasks.md` (build checkpoint) are what make every
recommendation above **resumable and crash-safe**:
- **`pipeline.md`** — one row per stage (`pending` / `in-progress` / `complete` / `skipped` /
  `blocked`), plus `Mode`, `Branch`, `PR`, and the counters. The orchestrator's *only* memory.
  Re-running any mode reads it and skips done stages → idempotent resume.
- **`tasks.md`** — per-slice checkpoint inside the build stage; mark done as each
  red-green-refactor slice lands; commit per slice so `tasks.md` + clean commits are the
  resume points.

This mirrors Claude Code's own filesystem-state design (tasks written to disk so agents
coordinate and resume across sessions) — *"keeping exploration and implementation out of your
main conversation"* with the durable record on disk, not in context. Two rules that keep it
honest: (1) **never route a sub-agent's findings through memory alone — write them to the
file**; (2) **a stage/task is only `complete` with evidence behind it** (pasted test output,
a green run) — not on an agent's say-so.
Sources: <https://code.claude.com/docs/en/sub-agents> ·
<https://venturebeat.com/orchestration/claude-codes-tasks-update-lets-agents-work-longer-and-coordinate-across>

---

## 4. Open questions / experiments

1. **Experiment 0, for real.** Run command-mode and agent-mode on two comparable features;
   fill the table in `harness-evolution.md`. Specifically: does command-mode ever make a
   wrong call from stale session context that agent-mode avoids? (Hypothesis: yes, rarely,
   and the "read only `pipeline.md`" rule already mostly prevents it.)
2. **Parallel-review payoff threshold.** At what diff size does fanning review across 3 lenses
   beat one sequential reviewer, given ~3× tokens? Measure tokens + wall-clock + catch-rate
   on a few real PRs. (Anthropic warns sub-minute tasks lose to the split overhead.)
3. **Does adversarial self-review hit the ≥80% catch target?** Plant the three violations from
   Iteration 2; measure catch-rate of a fresh independent reviewer vs the inline code-review
   bookend. Does *independence* (clean context) measurably beat same-context review?
4. **Decoupled-slice detection.** Can the planner reliably emit a machine-readable
   "these slices share no contract / these do" annotation in `plan.md` so the orchestrator
   knows what's safe to parallelize — without a human deciding each time?
5. **Background-agent fan-in ergonomics.** If two features build as background agents in
   separate worktrees, what's the cleanest fan-in/merge-order story, and does `claude agents`
   view give enough visibility to babysit several at once?
6. **Routine cost ceiling.** With per-tier daily run caps and N× tokens for any fan-out,
   what's a sane budget for the gardening + triage Routines on a ~₹2L/yr-per-NGO operation?

---

## Sources

- Building Effective Agents (workflow patterns; workflows vs agents):
  <https://www.anthropic.com/research/building-effective-agents>
- When to use multi-agent systems (3–10× tokens; verification subagents; avoid
  problem-centric decomposition):
  <https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them>
- Multi-agent research system (orchestrator–worker fan-out; 90.2% / ~15× tokens):
  <https://www.anthropic.com/engineering/multi-agent-research-system>
- Subagents (isolated context, summaries-only, can't talk mid-flight; tier distinction):
  <https://code.claude.com/docs/en/sub-agents>
- Background agents / agent view (`claude agents`, independent sessions, one dashboard):
  <https://code.claude.com/docs/en/agent-view>
- Agent teams (communicating sessions): <https://code.claude.com/docs/en/agent-teams>
- Headless / Agent SDK CLI (`-p`, `--bare`, `--output-format`, `--resume`):
  <https://code.claude.com/docs/en/headless>
- Agent SDK overview: <https://code.claude.com/docs/en/agent-sdk/overview>
- Scheduled tasks / Routines (cron, `cron_create`/`list`/`delete`, `/schedule`, `/loop`):
  <https://code.claude.com/docs/en/scheduled-tasks>,
  <https://claude.com/blog/whats-new-in-claude-managed-agents>
- Tasks / cross-session coordination (filesystem state):
  <https://venturebeat.com/orchestration/claude-codes-tasks-update-lets-agents-work-longer-and-coordinate-across>
