# Loop Engineering: The Techniques Behind Agents That Prompt Themselves

> For builders who want to know what loop engineering actually is, which techniques are worth copying,
> and how much of the hype is backed by evidence.
>
> Compiled from primary sources: Addy Osmani's original essay, Geoffrey Huntley's Ralph post, Anthropic
> engineering, Andrew Ng's *The Batch*, Gergely Orosz's ~210-reply investigation, Cobus Greyling's pattern
> repository, and the first peer-reviewed study of the practice (Lulla et al., JAWs@ASE 2026, which mined
> 36,487 GitHub repositories). All links point to the original source. Research date: 28 Oct 2026.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Why It Matters (and the Timeline)](#2-why-it-matters-and-the-timeline)
3. [How Practitioners Use It](#3-how-practitioners-use-it)
4. [Techniques Catalog](#4-techniques-catalog)
5. [Deep Dive: Event-Driven, Verification, and Hill-Climbing Loops](#5-deep-dive-event-driven-verification-and-hill-climbing-loops)
6. [Advanced Insights: What the Evidence Actually Says](#6-advanced-insights-what-the-evidence-actually-says)
7. [Curated Reading List](#7-curated-reading-list)
8. [Learning Path](#8-learning-path)
9. [Practical Application](#9-practical-application)

---

## 1. Core Concepts

### The one-line definition

Addy Osmani, who named the practice:

> "Loop engineering is replacing yourself as the person who prompts the agent. You design the system that does it instead."

Source: [Loop Engineering — Addy Osmani (7 Jun 2026)](https://addyosmani.com/blog/loop-engineering/)

The more rigorous definition, from the first academic study:

> "Loop engineering is the practice of designing automated control structures that repeatedly invoke coding
> agents, triggered on a schedule or by events, with each agent loop bounded by a machine-checkable stop
> condition. A well-engineered loop persists state across agent loops, verifies results independently of the
> implementation agents, bounds inference costs, restricts what an unattended agent loop may change without
> approval, and defines when humans should intervene."

Source: [Lulla et al., *Loop Engineering: Building Blocks, Adoption, and Impact* (arXiv 2608.21884)](https://arxiv.org/abs/2608.21884)

### The four-layer ladder

Loop engineering is the latest rung in a sequence where each layer subsumes the one below:

| Layer | Unit of concern | Example artifacts |
|---|---|---|
| Prompt engineering | One instruction | Prompts, templates |
| Context engineering | One model call | `AGENTS.md`, `CLAUDE.md`, rules |
| Harness engineering | One agent loop | Skills, subagents, hooks, MCP config |
| **Loop engineering** | **Repeated agent loops over time** | Schedules, stop conditions, state files, run logs, budgets, verifier agents |

Source: Table 1 in [Lulla et al.](https://arxiv.org/abs/2608.21884)

The key distinction is **agent loop vs. loop**. An *agent loop* is the model's own cycle of calling tools until it
stops (the ReAct loop inside Claude Code or Codex). A *loop* sits outside that: it decides when an agent loop
starts, checks what it produced, saves the result, and starts it again. Example from the paper: an agent can run
tests and revise inside one session; a loop starts a *new session* every time the build fails.

### The sharpest definitional line: a feedback signal

A commenter in Orosz's thread argued that an agent fired by cron or a webhook is "still largely a workflow". It
only becomes a loop when **one iteration's outcome feeds the next**. If your scheduled job does not read what the
previous run did, it is automation, not a loop. ([Orosz, LinkedIn thread](https://www.linkedin.com/posts/gergelyorosz_what-is-loop-engineering-to-you-anyway-share-7482371640924737536-tmm8/), via Lulla et al. O1)

### The anatomy everyone agrees on

Across sources, a well-built loop has the same parts ([Lulla et al. O3](https://arxiv.org/abs/2608.21884); [Osmani](https://addyosmani.com/blog/loop-engineering/)):

| Part | Job |
|---|---|
| **Trigger** | A schedule or an event (failing build, new issue, new Sentry error) starts the run |
| **Goal / stop condition** | A check a machine can run that says "done" |
| **State on disk** | Progress kept in files, not in the context window: "the agent forgets, the repo doesn't" |
| **Skills** | Project knowledge written once, read every run |
| **Isolation** | Git worktrees so parallel agents don't collide |
| **Verifier** | A separate checker that can reject the maker's work |
| **Connectors** | MCP access to trackers, Slack, CI, so the loop can open the PR itself |
| **Budget + kill switch** | Token/iteration caps and a documented way to pause |
| **Constraints + escalation** | What it may change without approval, and when it must call a human |

---

## 2. Why It Matters (and the Timeline)

The term is under five months old. Dates matter because most "loop engineering" content online was written in a
six-week burst.

| Date | Event |
|---|---|
| Mar 2024 | Andrew Ng reports GPT-3.5 "wrapped in an agent loop" hits 95.1% on HumanEval vs 67.0% for zero-shot GPT-4 ([Lulla et al. §2.2](https://arxiv.org/abs/2608.21884)) |
| 14 Jul 2025 | Geoffrey Huntley publishes the **Ralph Wiggum** technique: a bash loop that re-runs the agent forever ([ghuntley.com/ralph](https://ghuntley.com/ralph/)) |
| Mar 2026 | Claude Code ships `/loop` (recurring runs) |
| Apr 2026 | OpenAI Codex ships goal objects: "keep working until this outcome is true" |
| May 2026 | Claude Code, Hermes and others ship `/goal`, which builds the Ralph pattern into the tool ([Orosz](https://newsletter.pragmaticengineer.com/p/what-is-loop-engineering)) |
| 2 Jun 2026 | Boris Cherny (creator of Claude Code): "I don't prompt Claude anymore. I have loops that are running… My job is to write loops." ([WorkOS interview](https://www.youtube.com/watch?v=RkQQ7WEor7w)) |
| 7 Jun 2026 | Peter Steinberger: "you shouldn't be prompting coding agents anymore. You should be designing loops that prompt your agents" (8M+ views) ([X](https://x.com/steipete/status/2063697162748260627)). Same day, Osmani's essay names it. |
| 26 Jun 2026 | Ng calls it "a hot buzzphrase" and frames three loops at three timescales ([The Batch](https://www.deeplearning.ai/the-batch/three-key-loops-for-building-great-software)) |
| 14 Jul 2026 | Orosz publishes his investigation from ~210 practitioner replies ([Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/what-is-loop-engineering)) |
| 17–18 Jul 2026 | Steinberger asks "did we shift to graphs yet?"; Hamel Husain posts "Loop Engineering Is Dead. Enter Graph Engineering" (see §6 for why that's partly a joke) |
| 22 Oct 2026 | arXiv v3 of Lulla et al., the first empirical study |

### Why people care

- **The leverage point moved.** Cherny's point, per Osmani, "isn't that the work got easier. It's that the leverage point moved." You write the system once and it does the prompting.
- **Review bandwidth, not typing, is the bottleneck.** Prompt-by-prompt supervision stopped scaling, so practitioners moved the human from "sends every prompt" to "designs the gates and reads the output".
- **Context windows are finite; files aren't.** Restarting with a fresh context and keeping progress on disk lets work run for hours or days without the model drowning in its own history.

---

## 3. How Practitioners Use It

From Orosz's ~210 replies ([Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/what-is-loop-engineering)) and the paper's mining study:

**Event-triggered**
- Open a PR for each new Sentry issue
- Triage support tickets and outages
- Investigate test failures before a human looks
- Automatic PR review (the most common real-world loop by far: 180 of the 217 confirmed loops in the GitHub study were event-triggered, mostly PR review)

**Scheduled**
- Nightly end-to-end test repair with automatic reruns
- Flaky-test stabilization
- Large migrations run in small slices every 30 minutes
- Daily triage of CI failures, issues, and dependency updates

**Run-until-done**
- Keep running design reviews until no new issues appear
- `/goal "all tests in test/auth pass and lint is clean"`

**Claimed scale (self-reported, see §6)**
- Stripe's internal "Minions" system: ~1,300 agent-written PRs per week ([How I AI podcast with Steve Kaliski](https://podcasts.apple.com/us/podcast/how-stripe-built-minions-ai-coding-agents-that-ship/id1809663079?i=1000757255000))
- Mozilla: 423 Firefox security fixes in one month via a loop (reported by Osmani)
- A $50,000 contract delivered for $297 in API costs using Ralph ([The Register](https://www.theregister.com/2026/01/27/ralph_wiggum_claude_loops/), [VentureBeat](https://venturebeat.com/ai/how-ralph-wiggum-went-from-the-simpsons-to-the-biggest-name-in-ai-right-now))

---

## 4. Techniques Catalog

Each technique below: what it is, how to do it, and where it breaks.

### 4.1 The Ralph loop (fresh context + state on disk)

**What it is.** The original loop. In its purest form:

```bash
while :; do cat PROMPT.md | claude-code ; done
```

Each run starts with an empty context, reads the prompt and the files on disk, does one thing, commits, and exits.
The next run sees the changes. ([Huntley](https://ghuntley.com/ralph/))

**How to do it.**
- **One item per loop.** Huntley repeats this twice: "One item per loop. I need to repeat myself here—one item per loop." Small scope keeps each run inside the useful part of the context window.
- **File conventions:**
  - `PROMPT.md`: the instructions run every time. It is never "perfect"; you keep editing it as you watch failures.
  - `@fix_plan.md`: a prioritized TODO list the agent updates. Throw it away and regenerate it when Ralph drifts.
  - `@specs/*`: one spec file per area, read each run.
  - `@AGENT.md`: how to build and run the project, plus learnings the agent writes back.
- **Backpressure.** Type checkers, tests, linters, and security scanners reject bad code before it lands. "If you're using a dynamically typed language, the importance of wiring in a static analyser/type checker… cannot be overstated."
- **Fan out reads, serialize builds.** Many subagents for searching and analysis, but only **one** subagent for builds/tests.
- **"Signs."** When Ralph repeats a mistake, add a line to the prompt that blocks it. E.g. it kept re-implementing existing code, so: "Before making changes search codebase (don't assume an item is not implemented) using subagents."

**Where it breaks.** Placeholder implementations (the model optimizes for "compiles", not "correct"), duplicated code, broken builds that need a `git reset --hard`, drift from the goal. Huntley says it works best on **greenfield** projects and still needs senior engineers: "There is no way this is possible without senior expertise."

### 4.2 Pair the trigger with the stop rule (`/loop` + `/goal`)

**What it is.** Two primitives now built into Claude Code and Codex:

| Primitive | Behavior | Use for |
|---|---|---|
| `/loop` | Runs on a cadence | Recurring discovery: triage, sweeps, briefings |
| `/goal` | Runs until a condition is true | Bounded work with a clear finish line |

The pairing practitioners describe: use `/loop` to schedule the check that finds work, then `/goal` to finish each piece. Osmani describes `/goal` as
having a separate small model check after every turn whether you're done, "so the agent that wrote the code isn't
the one grading it." ([Osmani](https://addyosmani.com/blog/loop-engineering/))

**Choose start and stop as one decision** ([agentpatterns.ai](https://agentpatterns.ai/loop-engineering/loop-trigger-selection/)):

| Trigger | Starts when | Stops when | Best for |
|---|---|---|---|
| Goal-bounded | Previous turn finishes | Evaluator confirms done, or judges it impossible | Work with a definable finish line |
| Scheduled | Time elapses | Operator stops it or agent decides work is done | Environments that keep producing work |
| Event-driven | External signal arrives | Reaction to that signal completes | State changes you'd otherwise poll for |

"A clock supplies no such condition": a scheduled loop must define what "done" means for **each firing**.

**Where it breaks.** Firings that don't read prior state (wasted polling), stop conditions the transcript can't see,
subjective goals ("is this design good?"), session-scoped schedulers that silently skip runs.

### 4.3 Maker/checker separation (the most important technique)

**What it is.** The agent that did the work must not be the one that decides it's done. Every source puts this
first. Osmani calls the verifier the reason you "can walk away", and: "The model that wrote the code is way too nice
grading its own homework."

**How to do it — four variants, weakest to strongest:**

1. **Separate verifier subagent** with different instructions (`.claude/agents/`, `.codex/agents/`). Greyling's verifier skill **rejects by default** ([loop-engineering repo](https://github.com/cobusgreyling/loop-engineering)).
2. **Generator/evaluator with a sprint contract** (Anthropic). Before coding, generator and evaluator agree on what "done" means for the chunk and which behaviors to test. The evaluator then uses the running app through Playwright like a user would. ([Anthropic: Harness design for long-running apps](https://www.anthropic.com/engineering/harness-design-long-running-apps))
3. **Calibrate the evaluator.** Anthropic's first evaluators approved flawed work. They fixed it with few-shot examples carrying detailed score breakdowns and weighted criteria (design quality, originality, craft, functionality). Getting an evaluator to be strict is "far more tractable than making a generator critical of its own work."
4. **Two tiers: LLM verifier + deterministic hard gate.** The LLM checks intent ("does this solve the actual problem?"); a deterministic gate (tests, SAST, secret scanning, complexity rules) is what actually halts the loop, because it gives the same answer every run. ([Sonar](https://www.sonarsource.com/blog/loop-engineering-without-verification-is-just-automation/))

**Design the verifier first.** André Lindenberg argues the verifier comes before the loop, because "a loop satisfies the gate you wrote", not the goal behind it ([LinkedIn](https://www.linkedin.com/pulse/rom-prompts-loops-next-step-agent-engineering-andr%C3%A9-lindenberg-wnqre/)).

**Where it breaks.** "Verifier theater": a checker that approves without really checking. Reward hacking: the agent learns to pass the gate rather than do the work.

### 4.4 Machine-checkable stop conditions, budgets, and a kill switch

**What it is.** Every run needs an exit that doesn't depend on the agent's opinion, plus caps that stop a runaway.

**How to do it.**
- Write the goal as something a script can test: "all tests in `test/auth` pass and lint is clean", not "make auth robust".
- Set **iteration cap + token/dollar cap + wall-clock cap**. Any one hitting its limit ends the run.
- Document a pause mechanism (Greyling's "kill switch": a label, a file, a disabled workflow).
- Define **escalation points**: product questions, anything touching auth/billing/data, N failed attempts → stop and file it for a human. Ng's argument for this: "so long as the human knows something the AI does not, human-in-the-loop is needed."
- Write down what an unattended run may change without approval (`loop-constraints.md`).

**Where it breaks.** Infinite fix loops; a "CI sweeper" reportedly burned ~8 million tokens in 48 hours ([Lulla et al. O5](https://arxiv.org/abs/2608.21884)).

### 4.5 State and handoffs on disk

**What it is.** "The model forgets everything between runs so the memory has to be on disk and not in the context. The agent forgets, the repo doesn't." ([Osmani](https://addyosmani.com/blog/loop-engineering/))

**How to do it.** Greyling's conventions ([repo](https://github.com/cobusgreyling/loop-engineering)):

| File | Purpose |
|---|---|
| `LOOP.md` | Human-readable spec of the loop |
| `STATE.md` | What's done, in progress, failed, next (required before L2) |
| `loop-budget.md` | Token limits and spend |
| `loop-constraints.md` | What it may and may not touch |
| `loop-run-log.md` | Append-only history of runs |

Anthropic uses the same idea between agents: "one agent would write a file, another agent would read it and respond
either within that file or a new file." A Linear board works as state too.

**Where it breaks.** Drift between the committed instructions and the state file. A 30-day-old `STATE.md` is not
evidence the loop is healthy; Greyling's tooling now weights recent runs over stale files.

### 4.6 Context resets vs. compaction — and pruning the harness

**What it is.** Two ways to handle a full context window:
- **Reset**: kill the session, start fresh with a structured handoff file. Fixes "context anxiety" (the model rushing to wrap up), costs more tokens and time.
- **Compaction**: summarize the history in place. Keeps continuity but doesn't remove context anxiety.

Anthropic needed resets for Sonnet 4.5. With Opus 4.6 they removed resets, removed sprints, and moved the evaluator
to one pass at the end. ([Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps))

**The general rule:** "Every component in a harness encodes an assumption about what the model can't do on its own,
and those assumptions are worth stress testing." When a new model ships, remove one component at a time and measure.

**Cost reference:** a solo run of their game-maker task took 20 min / $9; the full harness took 6 hours / $200. The
simplified v2 harness built a digital audio workstation in 3 h 50 min for $124.70.

### 4.7 Isolation (worktrees) and skills

- **Worktrees**: one git worktree per parallel agent. "Two agents writing the same file is the exact same headache as two engineers committing to the same lines." Worktrees solve file collisions, but **your review bandwidth is still the ceiling on parallelism**.
- **Skills**: write project conventions once in `SKILL.md`. "Without skills the loop re-derives your whole project from zero every cycle, with skills it kind of compounds." ([Osmani](https://addyosmani.com/blog/loop-engineering/))

### 4.8 Staged autonomy: earn each level

**Greyling's readiness levels** ([repo](https://github.com/cobusgreyling/loop-engineering)):
- **L0** documented intent → **L1 Report** (agent observes and proposes; human acts) → **L2 Assisted** (agent acts within guardrails; human verifies after) → **L3 Unattended**.
- Rule: only move up **after the verifier has been accurate for a week**.

**Osmani's hand-off ladder** — loops classified by what you give up ([via Lulla et al.](https://arxiv.org/abs/2608.21884)):
1. **Turn-based**: you review every turn
2. **Goal-based**: you hand off the stop condition
3. **Time-based**: you hand off the trigger
4. **Proactive**: you hand off the prompt itself

His governing principle: **"The autonomy you grant an agent is the level you can cheaply verify."** ([LinkedIn](https://www.linkedin.com/posts/addyosmani_agentic-autonomy-levels-ugcPost-7486670405957505024-ZjLa/))
He also separates *agency* (how far one agent goes alone) from *orchestration* (how many agents run and who coordinates).

### 4.9 The autoresearch pattern (fixed budget + single metric + keep/discard)

**What it is.** Karpathy's `autoresearch` applies the loop to ML experiments. The agent edits `train.py`, trains for
a **fixed 5 minutes**, measures one number (`val_bpb`, lower is better), keeps the change if it improved and discards
it if not. ~12 experiments an hour, ~100 overnight. The human edits `program.md`, which Karpathy calls "programming
the program." ([The New Stack](https://thenewstack.io/karpathy-autonomous-experiment-loop/))

**Why it's worth copying beyond ML.** It's the cleanest example of a loop with a perfect verifier: a fixed budget
makes runs comparable, a single scalar makes keep/discard automatic, and git makes discard free. Any problem you can
reduce to "change → measure one number → keep or revert" (prompt tuning, routing rules, performance) fits.

### 4.10 Ready-made loop patterns

Greyling's registry, with suggested cadence and starting level ([repo](https://github.com/cobusgreyling/loop-engineering)):

| Pattern | Cadence | Start at | Cost |
|---|---|---|---|
| Daily triage (issues, CI, deps) | 1d–2h | L1 report | Low |
| Thin loop (GitHub Actions, minimal files) | event + 1d | L1 | Very low |
| Issue triage | 2h–1d | L1 propose-only | Low |
| Changelog drafter | 1d or on tag | L1 draft | Low |
| Post-merge cleanup | 6h–1d | L1 | Low |
| Dependency sweeper | 6h–1d | L2 patch-only | Medium |
| PR babysitter | 5–15m | L1 watch | High |
| CI sweeper | 5–15m | L2 cautious | Very high |

Note the cost column: the fast-cadence patterns are the expensive ones.

### 4.11 Ng's three loops (keep the human in the right loop)

| Loop | Timescale | Who drives |
|---|---|---|
| Agentic coding loop | Minutes | Agent writes, tests, iterates |
| Developer feedback loop | Tens of minutes to hours | Human decides features and UI direction |
| External feedback loop | Hours to weeks | Users, A/B tests, production |

The point: automating the inner loop doesn't remove the outer two. Humans have a "context advantage" about users that
the agent lacks. ([Ng, The Batch](https://www.deeplearning.ai/the-batch/three-key-loops-for-building-great-software))

---

## 5. Deep Dive: Event-Driven, Verification, and Hill-Climbing Loops

These three answer different questions, and a mature setup uses all of them:

| Loop | Question it answers | Timescale | What it changes |
|---|---|---|---|
| **Event-driven** | *When* does an agent run start? | Seconds after something happens | Nothing by itself; it starts the work |
| **Verification** | *Is this attempt actually done?* | Minutes, inside one run | The agent's current output |
| **Hill-climbing** | *Is the agent getting better?* | Hours to days, across many runs | The agent itself (prompt, tools, model, effort) |

How they fit together: a Sentry error **triggers** a run (event-driven). The agent fixes the bug and keeps going until tests pass and a separate checker agrees (**verification**). Traces of hundreds of these runs become the eval set you use to tune the agent's prompt one change at a time (**hill-climbing**).

```
  event ──▶ [ agent run ──▶ verify ──▶ fail? retry ]──▶ draft PR ──▶ human
                 ▲                                         │
                 └──── hill-climb: tune prompt/tools ◀──── traces + outcomes
```

### 5.1 Event-driven loops

**What it is.** Something outside the agent fires a run: a webhook from Sentry, GitHub (issue opened, CI failed, PR opened), PagerDuty, Stripe, or another agent's output. In the GitHub study, this is by far the most common real loop: **180 of 217** confirmed loops were event-triggered, mostly automatic PR review ([Lulla et al.](https://arxiv.org/abs/2608.21884)).

**Where the tooling is.** Claude Code and Codex can run inside GitHub Actions on `pull_request`, `issues`, `workflow_run` or `repository_dispatch` events. Anthropic's Managed Agents add inbound triggers (GitHub/Stripe/Sentry firing a routine through `routines.fire`) and outbound webhooks when a session finishes or passes/fails its rubric ([Hookdeck](https://hookdeck.com/blog/anthropic-managed-agent-webhooks)). Self-hosted options like [Pylon](https://pylon.to/) run Claude Code in a Docker sandbox per event and post the result to chat for approval.

**Worked example: Sentry error → draft PR** ([Understanding Data](https://understandingdata.com/posts/sentry-errors-spawn-local-agents/))
1. Sentry fires a webhook on a **new issue**, not on every event.
2. The endpoint inserts a row keyed on the Sentry issue ID with `onConflictDoNothing`, so each issue spawns exactly one agent. "One logic bug throwing ten thousand times a day is one issue, not ten thousand events."
3. The agent gets the stack trace and issue link, reproduces the bug locally, fixes it, and opens a **draft PR only**. "The agent proposes, I dispose."
4. Guardrails: **one agent at a time** (so a bad deploy doesn't spawn fifty), a **daily cap** on attempts, no auto-merge.

**The engineering that makes it safe** — these are ordinary webhook problems, but an agent run costs dollars, so getting them wrong is expensive:

| Problem | Fix |
|---|---|
| Delivery is at-least-once; retries resend the same event | Dedupe on the event ID (Anthropic's retries reuse `event.id`) — or better, on a business key like the issue ID |
| Retried triggers start duplicate sessions | `routines.fire` has no idempotency key, so "every retried call to fire a routine creates a new Claude session" — dedupe *before* you call it ([Hookdeck](https://hookdeck.com/blog/anthropic-managed-agent-webhooks)) |
| Events arrive out of order | Don't assume order; read current state when the run starts |
| Flapping errors re-trigger forever | Once-per-issue dedup; cooldown window |
| Event storms (bad deploy, outage) | Concurrency cap + daily budget; route storms to a human |
| Forged or replayed webhooks | Verify the HMAC signature and timestamp (Anthropic's window is 5 minutes) |
| Endpoint keeps failing | Anthropic auto-disables an endpoint after ~20 consecutive failures — monitor it |

**Where it breaks.** It's only a *loop* if one run's outcome affects the next, e.g. the fix attempt is written back to the issue so a re-fire knows what was tried. Otherwise it's an event handler. And event triggers multiply review load fastest, because you don't control the rate.

### 5.2 Verification loops

**What it is.** The agent keeps working until an external check says the work is right, instead of stopping when it *thinks* it's done. This is the single highest-value technique. Boris Cherny argues that giving the AI a way to verify its own work improves the quality of the final result by "2-3x" ([VentureBeat](https://venturebeat.com/technology/the-creator-of-claude-code-just-revealed-his-workflow-and-developers-are)). He tests every UI change with the Claude Chrome extension: open a browser, click through, iterate until it works.

**Inner vs. outer.** Philipp Schmid separates the **inner loop** (edit → write test → run → see failure → fix, all inside one task: "reliability within a task") from the **outer loop** (learning across sessions: "getting smarter over time"). The agent's tool-calling cycle is the same everywhere: "The loop is hardcoded. What the model does *inside* the loop is not." Verification happens because the prompt and tools make it happen. ([philschmid.de, Feb 2026](https://www.philschmid.de/inner-loop-vs-outer-loop))

**The verification ladder — weakest to strongest:**

| Level | Check | Reliability |
|---|---|---|
| 0 | Agent says "done" | None. Models declare success on stubs and placeholders. |
| 1 | Self-critique: "review your answer and fix it" | Weak. Without outside feedback, models often can't correct their own reasoning and sometimes make it worse ([Huang et al., *LLMs Cannot Self-Correct Reasoning Yet*, ICLR 2024](https://arxiv.org/abs/2310.01798)) |
| 2 | Separate verifier agent with different instructions, rejecting by default | Better; avoids "grading its own homework" |
| 3 | Calibrated evaluator using the real product (Playwright, browser) against a pre-agreed contract | Strong for UI and behavior ([Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps)) |
| 4 | Deterministic gate: tests, type checker, linter, SAST, a script that exits 0 | Strongest; same answer every run. This is what should actually stop the loop ([Sonar](https://www.sonarsource.com/blog/loop-engineering-without-verification-is-just-automation/)) |

Best practice is to stack them: deterministic gate as the hard stop, an LLM verifier for "does this solve the real problem?", a human for product judgment.

**How to design one.** Ask: *if a human reviewed this, what would they actually do to approve it — and can the agent run that same check?* Then:
- Make the check runnable: `pytest tests/auth && ruff check && mypy`, a Playwright script, a curl against staging.
- Agree on the check **before** the work (Anthropic's sprint contract), so the agent can't redefine "done" afterwards.
- Surface the real failure. Cline found agents "assuming success without verification" and command exit codes not reaching the model; fixing those lifted their benchmark score ([Cline](https://cline.bot/blog/a-practical-guide-to-hill-climbing)).
- Cap retries. A verification loop with no iteration limit is an infinite fix loop.
- Protect the check. An agent that can edit the tests can make them pass by deleting assertions. Keep test files read-only for the fixing agent, or have the verifier diff them.

**Where it breaks.** Verifier theater (approves without checking), weak tests the agent satisfies trivially, subjective goals with no check at all, and Lindenberg's rule: the loop "satisfies the gate you wrote", not the goal behind it.

### 5.3 Hill-climbing loops

**What it is.** A loop that improves the *agent itself*. Make one change (prompt wording, a tool description, the model, the reasoning effort), score it on an eval, keep it if the score goes up, revert if not, repeat. Karpathy's autoresearch (§4.9) is hill-climbing on a training script; the same method now gets applied to agents and harnesses.

**Why it got famous.** On 28 Sep 2026 Anthropic's Lance Martin published "Automating eval design and hillclimbing with Claude", adding two commands to the `claude-api` skill:
- `/claude-api build-eval` interviews you and builds an eval in your codebase
- `/claude-api hillclimb` improves your app against it "one change at a time, with a held-out set of examples to catch overfitting"

Reported results (as summarized by The Neuron; I could not reach Anthropic's original post to check these figures): an internal support agent went from **74.4% accuracy at $4.60/ticket** (Opus 4.8, high effort) to **90.5% on the held-out set at about one-fifth the cost**. The loop *downgraded* the model, effort, and prompts while improving accuracy. Hill-climbing the `claude-api` skill took it from 66% to ~88% and found two bugs where the eval's task spec was wrong. ([The Neuron](https://www.theneuron.ai/explainer-articles/anthropic-built-a-better-way-to-improve-ai-agents-without-fooling-yourself/); method details from [Arize](https://arize.com/blog/claude-hillclimb-production-traces/). Related talk: [Evals for Taste: Hill-Climbing a Slide-Generation Agent, Code w/ Claude 2026](https://claude.com/code-with-claude/session/sf-ext-eval-driven-agent-development))

Other public results:
- **Cline**: 47% → 57% on Terminal Bench (89 tasks) by fixing one failure pattern at a time ([Cline](https://cline.bot/blog/a-practical-guide-to-hill-climbing))
- **LangChain**: Terminal Bench 2.0 from 52.8% → 66.5% with harness-only changes; one "reasoning sandwich" config scored 63.6% vs 53.9% for uniform max reasoning ([agentpatterns.ai](https://github.com/agentpatterns-ai/website/blob/main/patterns/agent-design/harness-hill-climbing.md))

**The method, step by step**

1. **Build the eval from production, not imagination.** Start with real traces; keep cases a human judged hard. "Eval tasks mirror production." ([Arize](https://arize.com/blog/claude-hillclimb-production-traces/))
2. **Split train/test at random, with a fixed seed.** The climber reads only train failures; it never sees test.
3. **Measure noise first.** Run the baseline several times. Cline's six runs of the same config scored 0.49, 0.43, 0.45, 0.44, 0.48, 0.46: a 6-point spread. Any "improvement" smaller than the noise isn't one. Average 3–6 runs for close calls.
4. **Read the failures, find one root cause.** Group failures (timeouts, unverified success, swallowed exit codes) and pick the biggest group.
5. **Make one change.** One change per round, so you know what caused the score change and can undo it cleanly.
6. **Score on both sets. Keep only if both improve.** If train goes up while test stays flat, treat it as overfitting and revert.
7. **Commit and repeat.** Git makes "revert" free.

**What to tune** ([agentpatterns.ai](https://github.com/agentpatterns-ai/website/blob/main/patterns/agent-design/harness-hill-climbing.md)): system prompt wording, tool descriptions, reasoning budget, a pre-completion checklist (fewer early exits), loop-detection thresholds, when context gets injected, and model choice (often a cheaper model wins after tuning).

**Where it breaks**
- **Overfitting to the eval.** Signs: the tuning score rises while production errors don't move; changes are narrow prompt lines that echo eval phrasing; train and test diverge. Fix: held-out set, rotate tasks from fresh production traces, and treat production outcomes as the real score.
- **Noisy judges.** An LLM-as-judge adds its own variance. Prefer deterministic graders (tests) where possible.
- **Coupled knobs.** If changing the prompt changes how the tools get used, one-at-a-time attribution stops working.
- **Cost vs. benefit.** For a narrow agent, editing by hand can be faster than building the eval.
- **Wrong evals.** Anthropic's climb found the *eval* was wrong twice. When the climber gets stuck on a case, check the case.

**The key point:** hill-climbing is only as good as the verifier underneath it. Event-driven loops decide when to work, verification loops decide whether a single run's output is right, and hill-climbing uses that same verification signal across hundreds of runs to make the agent itself better.

---

## 6. Advanced Insights: What the Evidence Actually Says

### The benefit claims are anecdotes

Lulla et al. reviewed the gray literature and concluded: **"These sources provide no systematic evidence of the
claimed benefits."** Stripe's 1,300 PRs/week, Mozilla's 423 fixes, and the $297 contract are all self-reported. Even
a practitioner playbook warns that circulating numbers "are mostly secondhand summaries and should be treated as
rough reference." ([arXiv 2608.21884, O4/O6](https://arxiv.org/abs/2608.21884))

Also: the practitioner sources aren't independent. Greyling's repo, HuaShu's playbook and Lindenberg all build on
Osmani's posts, so "everyone agrees on the anatomy" partly means "everyone read the same essay."

### Real adoption is small, and nobody commits the recommended files

The mining study scanned 36,487 actively maintained GitHub repos (Aug 2026):

- **217 repos (0.59%)** run confirmed autonomous agent loops.
- **189 of 217** use Claude Code; then Codex (11), OpenCode (6), Gemini CLI (5), Cursor (4), Copilot (2).
- **180** are event-triggered (mostly automatic PR review); only 21 are scheduled.
- Almost all committed the **trigger** (a GitHub Actions workflow). The file heuristics found **zero** dedicated state, budget, stop-condition, or verifier files of the kind the literature prescribes.
- But reading the workflow prompts and YAML of the 256 inspected repos turned up the mechanisms themselves: a goal/stop condition in 55, persisted state in 35, a spend or turn limit in 101, a verifier in 76, a cost record in 10. **The mechanisms are fairly common; the file conventions aren't.** People write budgets and gates into the trigger config instead.
- Gotcha: GitHub marked some workflow runs "successful" even though the agent step was skipped. One repo logged 6,290 runs, and none of the runs the authors checked reached the agent step.

In plain terms: what people actually run today is "agent reviews every PR", not elaborate self-sustaining loops.

### The skeptics have a point

- **"A renamed cron job."** Orosz found most real examples fall into two familiar buckets: cron jobs and event triggers, now with a smarter worker. The counter-argument: the new part is a non-deterministic worker, which is exactly why you need engineered guardrails.
- **"Tokenmaxxing."** Loops burn tokens, and some developers suspect the labs benefit from promoting them.
- **It will be absorbed.** Max Kanat-Alexander expects anything developers must know about a tool to "eventually become baked into its default workflow." With `/loop` and `/goal` built in, Orosz called loop engineering "as good as obsolete" outside AI infrastructure.
- **Some people tried it and it didn't help.** Agents drifted across iterations and the loops were abandoned.

### The failure modes are well documented

| Failure | What happens |
|---|---|
| Verifier theater | The checker approves without real checks |
| Gate gaming | "A loop satisfies the gate you wrote", not the goal behind it |
| Infinite fix loops / token blowups | ~8M tokens in 48h for one CI sweeper |
| Review fatigue | Nightly loops open more PRs than the team can review; the human gate becomes a "rubber stamp" while "the pipeline still reports green" |
| Comprehension debt | "The faster the loop ships code you did not write, the bigger the gap between what exists and what you actually get" |
| Cognitive surrender | "When the loop runs itself it's very tempting to stop having an opinion and just take whatever it gives back" |
| State drift | Committed instructions and the state file disagree |

Osmani's warning sums it up: "A loop running unattended is also a loop making mistakes unattended." And: "Two people
can build the exact same loop and get completely opposite results. One uses it to move faster on work they
understand deeply. The other uses it to avoid understanding the work at all."

One observation from Orosz's thread worth remembering: **loops survive only where someone is already paid to review
their output.**

### "Graph engineering" — the next buzzword, handle with care

On 17 Jul 2026 Steinberger asked whether we'd "shifted to graphs yet"; within hours Hamel Husain posted "Loop
Engineering Is Dead. Enter Graph Engineering." The idea: design a **network** of loops (which run in parallel, what
state crosses between them, who can veto, when everything stops) instead of one loop
([Towards Data Science](https://towardsdatascience.com/graph-engineering-for-ai-agents-from-prompts-and-loops-to-workflows/), [Lindenberg](https://www.linkedin.com/pulse/from-loops-graphs-andr%C3%A9-lindenberg-m8jae)).

But coverage of the posts ([Turing Post](https://www.turingpost.com/p/is-graph-engineering-real-why-everyone-is-talking-about-it), [WiLine](https://wec.wiline.com/docs/news/loop-engineering-graph-engineering-what-survives/)) reports that both
founding posts were at least partly jokes about how fast the buzzwords were cycling. The academic paper treats graphs
as a reasonable next research direction. Treat it as an idea to watch, not an established successor.

---

## 7. Curated Reading List

**Start here**
1. [Addy Osmani — Loop Engineering](https://addyosmani.com/blog/loop-engineering/) — the essay that named it; the anatomy and the warnings
2. [Geoffrey Huntley — Ralph Wiggum as a "software engineer"](https://ghuntley.com/ralph/) — the original technique, very practical
3. [Lulla et al. — Loop Engineering: Building Blocks, Adoption, and Impact](https://arxiv.org/abs/2608.21884) — the evidence check; read §3 and §4.1

**Go deeper**
4. [Anthropic — Harness design for long-running apps](https://www.anthropic.com/engineering/harness-design-long-running-apps) — generator/evaluator, sprint contracts, resets vs compaction, real costs
5. [Cobus Greyling — loop-engineering repo](https://github.com/cobusgreyling/loop-engineering) — pattern registry, readiness levels, failure-mode docs, `npx @cobusgreyling/loop` scaffolding
6. [agentpatterns.ai — Loop trigger selection](https://agentpatterns.ai/loop-engineering/loop-trigger-selection/) — trigger/stop pairing plus related patterns (go/no-go gate, loop budgeting, goal contract)
7. [Sonar — Loop engineering without verification is just automation](https://www.sonarsource.com/blog/loop-engineering-without-verification-is-just-automation/) — the two-tier verifier

**Perspective and skepticism**
8. [Gergely Orosz — What is "loop engineering"?](https://newsletter.pragmaticengineer.com/p/what-is-loop-engineering) — ~210 practitioner replies, the most grounded view
9. [Andrew Ng — Three key loops for building great software](https://www.deeplearning.ai/the-batch/three-key-loops-for-building-great-software)
10. [Birgitta Böckeler — Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html) — the layer underneath loops
11. [Karpathy autoresearch explainer (The New Stack)](https://thenewstack.io/karpathy-autonomous-experiment-loop/)

**Event-driven, verification, hill-climbing**
12. [Hookdeck — What Anthropic's Managed Agent webhooks unlock](https://hookdeck.com/blog/anthropic-managed-agent-webhooks) — at-least-once delivery, ordering, idempotency gaps
13. [Sentry errors should spawn agents on your own machine](https://understandingdata.com/posts/sentry-errors-spawn-local-agents/) — a complete event-driven loop with dedup and guardrails
14. [Philipp Schmid — Agents: inner loop vs outer loop](https://www.philschmid.de/inner-loop-vs-outer-loop) — where verification lives
15. [Huang et al. — LLMs Cannot Self-Correct Reasoning Yet](https://arxiv.org/abs/2310.01798) — why self-critique alone isn't verification
16. [Arize — Claude's hillclimb loop: start with production traces](https://arize.com/blog/claude-hillclimb-production-traces/) and [The Neuron's explainer](https://www.theneuron.ai/explainer-articles/anthropic-built-a-better-way-to-improve-ai-agents-without-fooling-yourself/) — Anthropic's `/claude-api build-eval` + `hillclimb`
17. [Cline — A practical guide to hill climbing](https://cline.bot/blog/a-practical-guide-to-hill-climbing) — measuring noise, one change per round, real numbers
18. [agentpatterns.ai — Harness hill-climbing](https://github.com/agentpatterns-ai/website/blob/main/patterns/agent-design/harness-hill-climbing.md) — what to tune, when it backfires

---

## 8. Learning Path

1. **Run a Ralph loop on a toy greenfield project** (an afternoon). Write `PROMPT.md`, `fix_plan.md`, one spec, a test suite. Watch it fail and add "signs." This teaches more than any article.
2. **Use `/goal` with a machine-checkable condition** on a real task ("tests in X pass and lint clean"). Notice how much the wording of the condition matters.
3. **Build an L1 report-only loop** in GitHub Actions on a schedule: daily triage that writes a summary issue. No write access to code.
4. **Add a separate verifier** that rejects by default, plus a deterministic gate. Track how often it disagrees with the maker.
5. **Add state + budget files** (`STATE.md`, `loop-budget.md`) and make each run read the last run's output. Now it's actually a loop, not a cron job.
6. **Promote to L2** only after a week of accurate verification. Measure tokens per useful PR.
7. **Read Anthropic's harness post and try removing components** one at a time to see which are load-bearing.
8. **Hill-climb one agent.** Build a 30–50 case eval from real traces, split train/test with a seed, measure noise with 3+ baseline runs, then make one change per round and keep it only if both sets improve.
9. **Wire one event trigger** (e.g. new Sentry issue → draft PR) with dedup on the issue ID, a concurrency cap of 1, and a daily budget.

---

## 9. Practical Application

### Go/no-go: should this task be a loop at all?

Loop it only if all four hold:
- The work **recurs** or is **large enough to split** into many small items
- "Done" can be **checked by a machine** (tests, lint, a metric, a scanner)
- Someone has **review bandwidth** for the output (the loop survives only if someone is paid to read it)
- The **token cost** per useful result is lower than the human time it saves

If the goal is subjective ("is this good?"), keep a human in the turn-based loop.

### Minimal loop spec (copy into `LOOP.md`)

```markdown
# Loop: <name>

Trigger:      schedule (daily 06:00) | event (CI failure on main) | goal
Goal / stop:  <command that exits 0 when done>   e.g. `pytest tests/auth && ruff check`
Scope:        one item per run
Reads:        STATE.md, last entry of loop-run-log.md, skills/<x>
Writes:       branch + PR only; never pushes to main
Verifier:     separate agent, rejects by default; then deterministic gate (tests, SAST)
Budgets:      max 10 iterations | max 500k tokens | max 30 min per run
Kill switch:  add label `loop:pause` or delete .loop-enabled
Escalate if:  same item fails 3 times; touches auth/billing/migrations; product question
Level:        L1 (report only) — promote after 7 days of accurate verification
Owner:        <human who reviews the output>
```

### The ten rules, compressed

1. Design the **verifier first**; the loop will satisfy exactly the gate you write.
2. Separate **maker and checker**; prefer a deterministic gate as the final stop.
3. Pair every **trigger with a stop rule**; a clock is not a stop condition.
4. **One item per run**, fresh context, state on disk.
5. Make each run **read the previous run's outcome**, or it's just cron.
6. Cap **iterations, tokens, and time**, and document a kill switch.
7. Start at **report-only**; earn each autonomy level with a week of accurate verification.
8. Grant only the autonomy you can **cheaply verify**.
9. **Read the output.** Comprehension debt and cognitive surrender are the real long-term costs.
10. When the model improves, **remove harness components** and measure; every piece is an assumption about what the model can't do.
