# Architecture Patterns in Engineering Harnesses

> A pattern catalog. Researched June 2026.
>
> This is the **pattern-complement** to three companion reports in `../research/`:
> `harness-engineering.md` (the discipline and its case studies), `agent-design.md`
> (the agent as a system), and `context-engineering.md` (the finite-attention problem).
> Those documents explain the *concepts and the "why."* This one is organized around the
> **recurring building blocks** that show up across teams — and leads with the
> comparative axis the prose reports lack: **how different teams implement the same
> pattern differently, and what that buys them.**
>
> Where a concept is already explained in depth elsewhere (context rot, lost-in-the-middle,
> the single-vs-multi-agent debate, doom loops), this catalog cross-links rather than
> repeats, and spends its budget on the team-by-team comparison and the live debates.

---

## How to read this catalog

Each pattern follows the same five-part shape:

- **What it is** — the mechanism in one or two sentences.
- **Why it matters** — the failure it prevents or the leverage it provides.
- **How teams implement it** — the comparative axis. This is the part the prose reports don't have.
- **Tradeoffs** — what you give up.
- **Anti-pattern** — the recognizable failure mode.

A summary table sits at the end. The three highest-leverage patterns, by the weight of
current evidence, are **#3 Context Management**, **#4 Tool & Edit-Format Design**, and
**#5 Verification Loops** — these are where harness changes alone move benchmark scores
the most, and they are where this catalog concentrates.

---

## Pattern 1 — The Core Agent Loop

### What it is

The control loop that turns a stateless model into something that acts: **observe → think →
act → verify → repeat, until a termination condition fires.** Every coding harness is some
elaboration of this loop. The two archetypes are **ReAct** (interleave reasoning and acting
one step at a time, re-deciding after every observation) and **plan-then-execute** (produce
a full plan first, then carry it out, optionally re-planning when reality diverges).

### Why it matters

The loop's shape determines cost, latency, and brittleness. ReAct adapts to surprises but
pays a model call per step and can wander; plan-then-execute is cheaper and more predictable
but shatters when the plan meets an unexpected tool result. The comparison table of ReAct vs
Plan-and-Execute vs ReWOO vs Reflexion is laid out in `../research/agent-design.md` — not
repeated here.

### How teams implement it

The interesting variation is **how minimal the loop can be** and **how it terminates**:

- **mini-swe-agent (Princeton SWE-agent team)** is the extreme minimalist case: a ~100-line
  ReAct loop whose *only* tool is `bash`, no special tool-calling interface. It scores **>74%
  on SWE-bench Verified** (70.6% with Claude Sonnet 4.5, 74.2% with Gemini 3 Pro) — proof that
  a clean loop beats an elaborate one.
  [mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent)
- **Claude Code** runs a single ReAct-style loop but layers an `Explore` sub-agent and a
  three-tier search hierarchy on top (see Pattern 3). Termination is the model deciding it is
  done, bounded by deterministic Stop hooks (Pattern 8).
- **Stripe Minions** rejects a pure loop entirely: a **blueprint** alternates *deterministic*
  nodes (lint, CI, push) with *agentic* nodes (implement, fix CI). The loop only runs inside
  the agentic nodes; the spine is fixed code.
  [Stripe Minions Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- **LangChain deepagents** wraps the loop in a four-phase frame (Plan → Build → Verify → Fix)
  with middleware intercepting the loop's exit (Pattern 5).
  [LangChain harness engineering](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering)

**Termination mechanics** are a real design surface, not an afterthought. Teams use, in
combination: a hard **turn/iteration limit** (mini-swe-agent caps steps; agents that exceed
it return a structured failure), a **token/cost budget**, a **model-emitted "done" signal**
that a verification gate can reject (Pattern 5), and **deterministic stop conditions** (a Stop
hook, or Stripe's "two push-and-test cycles, then a human"). The robust pattern is **multiple
independent termination conditions** — never trust the model's self-declared "done" alone.

### Tradeoffs

ReAct buys adaptability at the cost of per-step model calls and a tendency to wander.
Plan-then-execute buys predictability and cache efficiency at the cost of brittleness when the
plan is wrong. Higher turn limits raise the ceiling on hard tasks but also raise the cost of a
runaway loop. There is no universal setting — the right loop depends on how verifiable the
task's intermediate states are.

### Anti-pattern

**The unbounded loop that trusts its own "done."** An agent with no turn limit, no cost
budget, and a termination condition of "the model says it finished" will both (a) run up cost
on the hard cases it can't solve and (b) declare victory on cases it didn't actually solve
(see Pattern 5, victory-declaration bias).

---

## Pattern 2 — Planning & Task Decomposition

### What it is

Deciding **upfront** how to break a task into steps, and **persisting that decomposition as
state** the agent reads and updates — a plan file, a TODO list, a spec — rather than holding
the plan only in the model's context.

### Why it matters

A plan that lives only in context decays as context grows (the lost-in-the-middle problem;
see `../research/context-engineering.md`). A plan written to a **durable artifact** survives
compaction, gives the agent a place to check off progress, and gives a human something to
review *before* expensive execution. The ZenML "anatomy of a production coding agent" finds the
plan-formulation and human-approval-gate stages are where a 30-second review "saves hours of
wasted compute" (`../research/agent-design.md`).

### How teams implement it

The axis is **plan-as-prose-in-context vs plan-as-file-on-disk**, and **plan-once vs
spec-then-build**:

- **Claude Code** maintains an explicit TODO list as first-class state, visible and
  updated as the agent works — the plan is a living checklist, not a one-shot preamble.
- **OpenAI Codex** reads hierarchical `AGENTS.md` files and internal design docs as the
  source of truth, with the spec preceding the build (the "internal docs as single source of
  truth" pattern from the Codex case study in `../research/harness-engineering.md`).
- **Spec-then-build harnesses** (including this Dalgo repo's own `/product/write-spec` →
  `/engineering/plan-feature` → `/engineering/execute-plan` flow) split planning into a
  separate, human-reviewable phase that produces `spec.md` and `plan.md` *before* any code.
  The plan file becomes the verification target (Pattern 5) and the resume checkpoint after an
  interruption.
- **Stripe Minions** decompose work into "bite-sized tasks" and explicitly prevent agents
  from "rabbit-holing" — the decomposition is enforced by the blueprint structure, not left to
  the agent (`../research/agent-design.md`).

### Tradeoffs

Upfront planning adds latency and a model call, and a confidently-wrong plan can anchor the
agent down a bad path (the plan-then-execute brittleness from Pattern 1). Reactive agents
adapt but re-derive the same plan repeatedly and lose the human-review checkpoint. Plan files
add durability but must be kept in sync — a stale plan is its own hazard.

### Anti-pattern

**Spec drift** — running an agent for a long time with no living artifact to verify against,
so "done" is whatever the agent decides. Fix: a structured plan the verifier checks against.
(Detailed in `../research/harness-engineering.md`.)

---

## Pattern 3 — Context Management *(highest leverage)*

### What it is

Deciding **what the model sees at each step** — and crucially for coding harnesses, **how the
agent retrieves code from the repository.** The retrieval sub-question — *embeddings/index vs
grep/agentic search* — is the single most consequential architecture fork in coding harnesses
today, and it is the part the prose reports name but don't frame as a *choice*.

### Why it matters

Context is a finite, depreciating resource; past a threshold, adding tokens actively hurts
(context rot, lost-in-the-middle — the mechanism is exhaustively covered in
`../research/context-engineering.md`, not repeated here). For a coding agent the dominant
context cost is *bringing code in*. Get retrieval wrong and you either flood the window with
irrelevant files or miss the one file that mattered.

### How teams implement it

**The retrieval fork — the defining current debate:**

- **Grep / agentic search (Claude Code, Cursor's agent, Devin).** No index. The model
  *actively directs* exploration through a tiered tool hierarchy — **Glob** (paths, near-zero
  cost) → **Grep** (regex content match, lightweight) → **Read** (full file, expensive) — plus
  an isolated `Explore` sub-agent. Anthropic's Boris Cherny: *"Early versions of Claude Code
  used RAG + a local vector db, but we found pretty quickly that agentic search generally works
  better."* A Claude engineer: *"agentic search outperformed [RAG] by a lot, and this was
  surprising."* The stated reasons: **precision** (grep finds exact matches; embeddings
  introduce fuzzy positives that are noise in code), **freshness** (a pre-built index drifts
  the moment code changes; filesystem reads are always current), **privacy** (no proprietary
  code embedded or stored externally), and **simplicity** (no index to build, maintain, or
  debug). [Claude Code doesn't index](https://vadim.blog/claude-code-no-indexing/) ·
  [Why agents use grep not vectors](https://www.mindstudio.ai/blog/is-rag-dead-what-ai-agents-use-instead)

- **Embeddings / index-first (Cursor's editor retrieval, Sourcegraph Cody, Augment).** Build a
  persistent semantic index *before* the agent works. Cursor computes a **Merkle tree** of file
  hashes, syncs only changed branches, splits changed files into syntactic chunks, embeds them,
  and serves semantic similarity at query time. Only **embeddings + metadata** go to the cloud;
  source stays local — the privacy mitigation for the index-first camp. The payoff is fast
  first-query response and *semantic* recall (find "the auth flow" without knowing the
  identifier). [How Cursor indexes](https://towardsdatascience.com/how-cursor-actually-indexes-your-codebase/) ·
  [Securely indexing large codebases (Cursor)](https://cursor.com/blog/secure-codebase-indexing) ·
  [Cursor vs Sourcegraph Cody (Augment)](https://www.augmentcode.com/tools/cursor-vs-sourcegraph-cody-embeddings-and-monorepo-scale)

- **Hybrid is the emerging consensus.** Vector pre-filter to narrow candidates, then
  grep/agentic confirmation for precision and freshness — embeddings answer the first query
  fast, grep verifies. The "RAG is dead" framing is overstated; *hybrid* (semantic + BM25
  keyword) has largely replaced pure vector RAG.
  [Lost nuance of grep vs semantic](https://www.nuss-and-bolts.com/p/on-the-lost-nuance-of-grep-vs-semantic)

The tradeoff lands differently by **codebase scale**: grep/agentic search shines on
actively-edited repos and where privacy matters; index-first wins on huge monorepos where
walking the tree per query is too slow and semantic recall over millions of lines is the point.

**The other three context operations** — covered conceptually in
`../research/context-engineering.md`, listed here only as the team-comparison axis:

- **Compaction / summarization.** Anthropic's Claude Code summarizes and reinitializes when
  approaching the window limit, preserving error traces. JetBrains research found **observation
  masking** (replace old observations with placeholders, keep reasoning) matched or beat LLM
  summarization in 4 of 5 scenarios at 52% lower cost — because summarization hides the failure
  signals the agent needs (summarized agents ran ~15% longer). Default to masking; summarize
  only as a fail-safe.
  [JetBrains research — Cutting Through the Noise](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)
- **Sub-agent context isolation.** A sub-agent burns its own window on a focused task and
  returns a 1,000–2,000-token summary; the lead never wades through the sub-task's reasoning.
  (See Pattern 6.)
- **System-prompt minimalism.** Start with the minimal viable prompt; add only what failure
  modes demand. A bloated prompt trains the model to ignore parts of it.

### Tradeoffs

Grep/agentic search costs a model call per exploration step and can miss semantically-related
code that shares no keyword; index-first costs build/maintenance infrastructure, can serve
stale results during active editing, and ships embeddings off-machine. Compaction is lossy by
design — aggressive on process, lossless on outcomes and error traces, or it destroys the
signal the agent needs.

### Anti-pattern

**Stuffing the window** — pre-loading the whole codebase, all schemas, or every tool
definition "just in case." It buys nothing and triggers context rot. The discipline is
**just-in-time**: lightweight identifiers (paths, queries) plus tool calls that fetch on demand.

---

## Pattern 4 — Tool & Edit-Format Design *(highest leverage)*

### What it is

The **Agent-Computer Interface (ACI)** — the commands, tool descriptions, and crucially the
**edit format** the agent uses to change code. SWE-agent's central finding: interface design,
not model capability, often determines the score.

### Why it matters

This is the sharpest single-variable demonstration in the field that the harness beats the
model. Same GPT-4, same SWE-bench problem: **~2× the resolution rate** with SWE-agent's ACI
tools versus raw bash. And the *edit format* alone — how the model expresses a code change —
swings scores massively, because a model that reasons correctly but emits an edit the harness
can't apply scores **zero** on that task. The model wasn't failing at coding; it was failing at
mechanical text matching. [SWE-agent](https://github.com/swe-agent/swe-agent)

### How teams implement it

**Edit formats — the design space, measured.** Aider supports and benchmarks several:

| Format | What the model emits | Cost | Notes |
|---|---|---|---|
| **whole** | The entire updated file | High tokens | Easiest for the model; caps editable file size; slow and costly |
| **diff** (search/replace) | Git-conflict-style search/replace blocks | Low | The default for most strong models |
| **diff-fenced** | diff with the path inside the fence | Low | Used for Gemini, which mis-fences otherwise |
| **udiff** | Simplified unified-diff | Low | Cut GPT-4 Turbo's "lazy coding" ~3× |
| **architect** | One model reasons the change, a second emits the edit | Higher | Separates reasoning from editing |

Aider's leaderboard measures **two** things: task success *and* **edit-format compliance** —
the percent of tasks where the model produced an applyable edit. The two are coupled: on the
legacy board, o1 hit 84.2% correct with **99.2%** format compliance using `diff` (illustrative
of the coupling, not a current-flagship ranking — the harder polyglot board has since
superseded this one). A model that can't conform to the format loses tasks it could otherwise
solve. The unified-diff format alone raised GPT-4 Turbo
to 61% and reduced laziness 3×.
[Aider edit formats](https://aider.chat/docs/more/edit-formats.html) ·
[Unified diffs make GPT-4 less lazy](https://aider.chat/docs/unified-diffs.html) ·
[Aider leaderboard](https://aider.chat/docs/leaderboards/edit.html)

The most dramatic single-variable result reported (a practitioner experiment, treat as
illustrative not peer-reviewed): switching edit format took one model from **6.7% → 68.3%**,
purely because the old format's "exact string not found" failures were hiding the model's
coding ability entirely. The weakest models gain the most from a better format.
(Discussed in `../research/harness-engineering.md`.)

**Tool descriptions are the real interface.** Anthropic reports spending *more* time tuning
tools than the agent prompt. Treat each description as an explanation to a new team member;
adding `input_examples` arrays took complex-parameter accuracy from 72% → 90% in their testing
(`../research/agent-design.md`). Bad descriptions "send agents down completely wrong paths."

**Keep tool count low.** Accuracy degrades past ~10–20 tools; at 50+ tools Claude Opus 4 hit
~49% (coin-flip). Mitigations: keep the set small and non-overlapping; use **dynamic/deferred
tool loading** — Anthropic's Tool Search cut tool-definition tokens ~85% while raising accuracy
49% → 88.1%. mini-swe-agent's extreme answer: **one tool (bash)**, and it scores >74%.
The MCP-tax math (150 tools = 60–90k tokens before any conversation) is in
`../research/context-engineering.md`.

### Tradeoffs

`whole` is reliable to apply but token-expensive and size-limited; `diff`/`udiff` are cheap but
fail when the model botches the format, costing the whole task. Rich tool descriptions cost
context tokens; terse ones cause mis-selection. A large tool surface covers more tasks but
degrades selection accuracy. The right edit format is **model-specific** — Aider picks per
model for exactly this reason.

### Anti-pattern

**The kitchen-sink ACI** — wrapping a full REST API as 50 CRUD tools, with a brittle edit
format the model frequently mis-formats. The agent spends its context budget on tool metadata
and loses tasks to "couldn't apply the edit" rather than "couldn't solve the problem."

---

## Pattern 5 — Verification Loops *(highest leverage)*

### What it is

**Feeding the agent's own output back through real checks — tests, linters, type-checkers,
reviewers — and blocking "done" until they pass.** The defining sub-pattern is *verification
middleware that intercepts the agent's attempt to finish* and forces a pass against the spec.

### Why it matters

With 85% per-step accuracy over 10 steps, naive success is ~0.85¹⁰ ≈ 20%; a verification gate
after each major phase recovers much of that loss
(`../research/harness-engineering.md`). The dominant failure it kills is **victory-declaration
bias** — the agent marks a task complete without running the tests.

### How teams implement it

The axis is **what the verifier is and whether the agent can route around it**:

- **LangChain deepagents** make this an explicit middleware stack and credit it for a
  benchmark jump (52.8% → 66.5% on Terminal-Bench 2.0, model unchanged):
  - **`PreCompletionChecklistMiddleware`** intercepts the agent's *exit* and forces a
    verification pass against the task spec — the canonical "block premature success" gate.
  - **`LoopDetectionMiddleware`** tracks per-file edit counts and injects a reconsideration
    prompt after N edits without progress — **doom-loop detection**.
  [LangChain harness engineering](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering)
- **Stripe Minions** put verification in *deterministic* blueprint nodes (run CI, run lint) and
  cap iteration: **two push-and-test cycles, then a human** — "diminishing marginal returns if
  an LLM runs against indefinitely many rounds of a full CI loop."
- **OpenAI Codex** encodes **linters that teach** — error messages prescribe the fix, not just
  flag the violation — and makes taste/architecture violations **hard CI failures, not
  warnings**, with inline suppressions disabled so the agent can't circumvent them
  (`../research/harness-engineering.md`).
- **Adversarial / reviewer sub-agents** are now standard: a separate evaluator grades each
  result in its *own* context window against a rubric and **sends it back to revise until it
  meets the bar.** Anthropic's June 2026 "Performance Outcomes" over Dynamic Workflows
  productizes exactly this loop. (See Pattern 6.)
  [Claude Code subagents 2026](https://www.developersdigest.tech/blog/claude-code-agent-teams-subagents-2026)

A key empirical nuance from `../research/context-engineering.md`: **don't summarize away the
failure signals.** The verifier's value is the error trace; compaction that hides *why* a check
failed makes the agent repeat the mistake.

### Tradeoffs

Every verification pass adds latency and tokens. Unbounded retry loops waste compute on
unsolvable cases (hence Stripe's two-cycle cap). Self-review is cheap but the model is a biased
judge of its own work; an independent reviewer agent is stronger but costs a second context.
Stronger gates = higher reliability, lower throughput.

### Anti-pattern

**The bypassable gate** — a linter the agent can silence with an inline suppression, or a
"did you test it?" prompt with no mechanical check behind it. If the agent can reason its way
around the gate, it will. Verification must be enforced by code (or a hook, Pattern 8), not
requested in the prompt.

---

## Pattern 6 — Orchestration & Sub-Agents

### What it is

Whether one agent does the work or a **lead agent fans out to workers**, each with its own
context window, tools, and sometimes model — and how their results are combined.

### Why it matters

Sub-agents are the primary tool for **context isolation** (Pattern 3) and **parallelism**. But
the multi-agent question is genuinely contested, and the evidence is covered in depth in
`../research/agent-design.md`: Anthropic's research system beat single-agent by **90.2%** but
cost **~15×** the tokens; Google DeepMind's 180-config study found multi-agent **degrades
sequential tasks by 39–70%** while improving parallelizable ones by +81%; Cognition argues
*"Don't Build Multi-Agents"* because fragmented context produces conflicting implicit decisions.
Not repeated here — the catalog entry is the *how*.

### How teams implement it

- **Orchestrator-workers (Anthropic Research).** A lead plans, spawns 3–5 parallel sub-agents,
  each runs 3+ tools, and writes results **directly to an external store** rather than
  funneling everything back through the lead — avoiding information loss at the join.
- **Single-threaded + compression (Cognition/Devin).** Default to one linear agent; for
  long tasks risking overflow, a compression model distills history into key decisions rather
  than forking into parallel agents.
- **Sub-agents as roles (Claude Code).** Named sub-agents (`Explore`, `code-reviewer`) with
  focused prompts and scoped tool lists, invoked when the role applies. Each returns a summary;
  the lead's transcript stays clean. **June 2026 Dynamic Workflows** let the lead fan out *tens
  to hundreds* of sub-agents in one session, with a grader looping each until it meets a rubric.
  [Claude Code subagents playbook 2026](https://www.totalum.app/blog/claude-code-subagents-totalum)

The decision rule that survives the disagreement: **fan out only for genuinely independent,
parallelizable subtasks with no shared implicit decisions.** For sequential work over a shared
codebase, a single agent wins on cost, debuggability, and coherence.

### Tradeoffs

Multi-agent buys parallelism and clean context isolation at ~15× tokens, harder debugging, and
the risk of conflicting decisions across workers. Single-agent is cheap, debuggable, coherent,
but capped by one context window and one thread of execution.

### Anti-pattern

**The chaotic swarm** — many agents launched without explicit delegation contracts or a join
strategy, building on conflicting assumptions. And its inverse on *sequential* tasks: paying
15× tokens for multi-agent on work that was never parallelizable.

---

## Pattern 7 — Human-in-the-Loop

### What it is

**Where a human reviews or approves before the agent proceeds** — and how much autonomy the
agent has between checkpoints. The autonomy spectrum runs **suggest → propose a PR → act
autonomously within boundaries.**

### Why it matters

A 30-second human review of a *plan* prevents hours of wasted compute and irreversible
mistakes (`../research/agent-design.md`). The whole market has converged on *autonomy within
explicit boundaries* rather than full autonomy — the position is argued in
`../research/harness-engineering.md`.

### How teams implement it

Three production patterns (detailed in `../research/agent-design.md`), with team examples:

- **Pre-execution approval** — agent presents a plan and pauses (LangGraph's `interrupt()`;
  Devin 2.0 reviews plans before execution; this Dalgo repo's spec/plan review gates).
- **Human-on-the-loop** — agent acts; a human can interrupt/override/roll back within a window
  (requires reliable rollback, e.g. Cognition's hypervisor state snapshots).
- **Confidence-based escalation** — agent runs autonomously, halts and escalates on risk
  signals (low confidence, high-stakes action).
- **Post-merge review** — the far autonomous end: OpenAI's Codex beta merged AI-authored PRs
  with *post*-merge analysis only. Viable only behind a very strong verification harness.

**The autonomy spectrum, by team:** Cursor/Copilot (suggest in-editor) → Aider/Claude Code
(propose edits/PRs, human approves) → Stripe Minions (autonomous within a bounded blueprint,
human reviews after two cycles) → OpenAI Codex beta (autonomous, post-merge analysis).
Autonomy should rise only as verification and observability rise with it.

### Tradeoffs

More gates = more safety and trust, less throughput and more human attention (the scarcest
resource — Lopopolo: *"the only fundamentally scarce thing is the synchronous human attention
of my team"*). Fewer gates = faster, but every removed checkpoint must be backed by a stronger
mechanical guarantee.

### Anti-pattern

**Autonomy without a matching verification floor** — granting an agent irreversible,
money-spending, or data-touching actions with no approval gate *and* no mechanical guard. For
non-technical users (Dalgo's NGO context), one bad autonomous action burns a limited trust
budget; irreversible actions need an explicit gate regardless of the autonomy level elsewhere.

---

## Pattern 8 — Determinism & Guardrails

### What it is

**Encoding rules as mechanical constraints the agent cannot reason around** — hooks, policy
gates, type/dependency layers, sandbox limits — rather than asking the model to behave in the
prompt. The thesis: *if a rule must always hold, enforce it in code, not in instructions.*

### Why it matters

A prompt instruction is a *probabilistic* nudge the model can forget, ignore, or talk itself
out of under context pressure. A mechanical gate is *deterministic*. This is the cleanest
expression of Mitchell Hashimoto's framing of the discipline: *"Anytime an agent makes a
mistake, you engineer a solution so the agent never makes that mistake again"*
(`../research/harness-engineering.md`). The distinction is exactly **probabilistic alignment
vs deterministic pre-action authorization** from the four-layer security stack in
`../research/agent-design.md`.

### How teams implement it

- **Claude Code hooks** are the reference implementation of mechanical enforcement. Shell
  commands fire at lifecycle events — **PreToolUse, PostToolUse, UserPromptSubmit, Stop,
  SubagentStop, SessionStart, Notification.** A **PreToolUse** hook fires *after* the model has
  chosen a tool and arguments but *before* execution; it receives the full tool call as JSON on
  stdin and can **approve, block** (the agent cannot bypass, forget, or reason around it), or —
  since v2.0.10 — **modify the tool input** and let execution proceed with corrected
  parameters. Structured JSON output (`decision: block|approve`, `reason`, `continue`) gives
  granular control; a Stop hook can deterministically prevent the agent from ending.
  [Claude Code hooks guide](https://code.claude.com/docs/en/hooks-guide) ·
  [Hooks as deterministic control](https://dotzlaw.com/insights/claude-hooks/) ·
  [Why each of my 95 hooks exists](https://blakecrosley.com/blog/claude-code-hooks)
- **Stripe Minions blueprints** make determinism structural: the deterministic nodes (lint,
  CI, push) are fixed code the agent runs through, not steps it can skip.
- **OpenAI Codex** enforces **dependency layers** (Types → Config → Repo → Service → Runtime →
  UI) with structural tests, and **hard CI failures** for taste/architecture violations with
  inline suppressions disabled (`../research/harness-engineering.md`).
- **Sandbox isolation** is the mechanical guard on *blast radius*: Firecracker microVMs (AWS
  AgentCore), Cognition's per-session dedicated kernels, Docker as the practical minimum
  (`../research/agent-design.md`).

The unifying move across all of these: **convert a behavioral rule into a mechanism.** "Never
run DROP without confirmation" becomes a PreToolUse hook that blocks `DROP`. "Always format on
save" becomes a PostToolUse hook. The agent's good behavior stops being a hope.

### Tradeoffs

Mechanical guards are rigid: a hook that blocks a command blocks it even in the rare legitimate
case, and a thicket of hooks adds latency and maintenance (one practitioner runs 95). Over-
constraining can also block the agent from valid recovery paths. The balance: encode the rules
that must *always* hold mechanically; leave judgment calls to prompt-based or agent-based hooks.

### Anti-pattern

**Rule-by-prompt for invariants** — putting "never force-push to main" or "always run the
tests" in the system prompt and trusting it. Under context pressure the model will violate it.
If it must always hold, it belongs in a hook or a CI gate, not in prose.

---

## Pattern 9 — Memory & Learning Across Runs

### What it is

**What the harness persists between sessions** so the agent doesn't relearn the same things —
rules files, skills, and longer-term memory stores.

### Why it matters

Without persistence, every run pays the "orientation tax" — thousands of tokens rediscovering
directory structure, conventions, and tooling (`../research/harness-engineering.md`). Persisted
knowledge amortizes that cost and is how a harness *compounds* over time — the harness, not the
model, becomes the team's accumulating asset.

### How teams implement it

The axis runs from **static rules** through **reusable skills** to **dynamic memory**:

- **Rules files — converging on a standard.** `AGENTS.md` has become the closest thing to a
  cross-tool standard: an open format now stewarded by the Linux Foundation's Agentic AI
  Foundation, read by **30+ agents** (Codex, Claude Code, Cursor, Copilot, Gemini CLI, Jules,
  Aider, Zed, Windsurf, Devin) across **60,000+ projects.** Codex concatenates `AGENTS.md`
  files root-to-leaf, nearer files overriding. Tool-specific variants persist alongside it
  (`CLAUDE.md`, `.cursor/rules/`, `.github/copilot-instructions.md`). Keep them concrete and
  under a few hundred lines — sprawling rule files get ignored.
  [AGENTS.md](https://agents.md/) ·
  [Codex AGENTS.md guide](https://developers.openai.com/codex/guides/agents-md) ·
  [AGENTS.md vs CLAUDE.md vs .cursorrules](https://www.morphllm.com/agents-md-guide)
- **Skills — procedural memory as reusable units.** Structured markdown (plus optional scripts)
  encoding "how to do X," loaded on demand. OpenAI's Codex team distilled engineering standards
  into **six core skills**; Claude Code ships a Skills system; this Dalgo repo's `.claude/skills/`
  are exactly this pattern. Skills differ from rules files: rules are *always-on context*,
  skills are *progressively disclosed* when relevant — which keeps the always-on budget small.
- **Dynamic memory stores — still unsolved.** Tiered memory (working / episodic / semantic /
  procedural) with vector or graph backing; Letta/MemGPT lets the agent manage tiers via
  explicit tool calls. ZenML's 1,200-deployment survey is blunt: **"memory systems remain
  unsolved"** — no consensus approach (`../research/context-engineering.md`).

The critical safety caveat from `../research/harness-engineering.md`: **stale memory is more
dangerous than no memory.** A confident-but-outdated fact drives destructive actions. Couple
persistent memory with just-in-time verification — check the claim against current state
(grep, file read, API call) before acting on it. Tag retrieved facts with provenance.

### Tradeoffs

Static rules are reliable and cheap but go stale and bloat the always-on context if overgrown.
Skills keep the budget small but add a discovery step. Dynamic memory is powerful but
unsolved, and risks poisoning (a hallucinated fact persisted and reused) and stale-but-confident
recall. More persistence = less orientation tax but more surface for drift.

### Anti-pattern

**The 800-line CLAUDE.md, and stale-but-confident memory.** A rules file so long the model
ignores half of it; or a memory store that returns an outdated schema name the agent then acts
on. Keep rules short and concrete; verify memory against live state before irreversible actions.

---

## Summary table

| # | Pattern | Core mechanism | The implementation fork | Defining anti-pattern |
|---|---------|----------------|--------------------------|------------------------|
| 1 | **Core agent loop** | observe→think→act→verify until termination | ReAct vs plan-then-execute; minimal (bash-only) vs elaborate; multiple termination conditions | Unbounded loop trusting its own "done" |
| 2 | **Planning & decomposition** | persist the plan as durable state | plan-in-context vs plan-as-file; plan-once vs spec-then-build | Spec drift — no living artifact to verify against |
| 3 | **Context management** ★ | what the model sees; how it retrieves code | **grep/agentic search vs embeddings/index** (hybrid emerging); compaction vs masking; sub-agent isolation | Stuffing the window "just in case" |
| 4 | **Tool & edit-format** ★ | the ACI; how the model expresses an edit | whole vs diff/udiff vs architect (model-specific); low tool count; deferred loading | Kitchen-sink CRUD tools + brittle edit format |
| 5 | **Verification loops** ★ | feed checks back; block premature "done" | self-review vs adversarial reviewer; middleware gate vs deterministic CI node; bounded retries | The bypassable gate the agent reasons around |
| 6 | **Orchestration & sub-agents** | one agent vs lead + workers | single-threaded+compression vs orchestrator-workers; fan out only for parallel, independent subtasks | Chaotic swarm; multi-agent on sequential work |
| 7 | **Human-in-the-loop** | review/approve before proceeding | pre-execution approval vs HOTL vs post-merge; autonomy rises with verification | Autonomy without a matching verification floor |
| 8 | **Determinism & guardrails** | encode rules as mechanism, not prompt | hooks (PreToolUse block/modify) vs blueprint nodes vs dependency layers vs sandbox | Rule-by-prompt for hard invariants |
| 9 | **Memory & learning** | persist knowledge across runs | rules files (AGENTS.md standard) vs skills (progressive) vs dynamic memory (unsolved) | 800-line rules file; stale-but-confident memory |

★ = highest-leverage patterns (where harness-only changes move scores the most).

---

## The three highest-leverage patterns, in one paragraph each

**Context management (#3)** is the biggest lever because the dominant cost and failure mode of a
coding agent is *bringing the right code into a finite window.* The live architecture fork —
grep/agentic search (Claude Code: precision, freshness, privacy, simplicity) versus
embeddings/index (Cursor/Cody: semantic recall, speed at monorepo scale), with hybrid emerging —
is the single most consequential decision a team makes, and it is decided by codebase scale and
privacy posture, not by which is universally "better."

**Tool & edit-format design (#4)** is where the harness most visibly beats the model: the same
model swings from ~2× (ACI vs raw bash) to a reported 6.7%→68.3% (edit format alone) with no
weight change, because a correctly-reasoned edit the harness can't apply scores zero. Pick the
edit format per model, keep the tool count low (or defer-load it), and treat tool descriptions
as the real interface.

**Verification loops (#5)** convert spiky model capability into reliable output by feeding real
checks back and *mechanically blocking premature "done."* The winning implementations make the
gate un-bypassable (CI hard-fails with suppressions disabled; a `PreCompletionChecklistMiddleware`
intercepting exit; an adversarial reviewer sub-agent that loops until a rubric passes) and
preserve error traces rather than summarizing them away — and they bound the retry loop so the
agent doesn't burn compute on the unsolvable cases.

---

## Sources

Primary sources verified for this catalog (June 2026):

- [Claude Code doesn't index your codebase (Vadim's blog)](https://vadim.blog/claude-code-no-indexing/)
- [Why agents use grep not vectors (MindStudio)](https://www.mindstudio.ai/blog/is-rag-dead-what-ai-agents-use-instead)
- [On the lost nuance of grep vs semantic search](https://www.nuss-and-bolts.com/p/on-the-lost-nuance-of-grep-vs-semantic)
- [How Cursor actually indexes your codebase (Towards Data Science)](https://towardsdatascience.com/how-cursor-actually-indexes-your-codebase/)
- [Securely indexing large codebases (Cursor)](https://cursor.com/blog/secure-codebase-indexing)
- [Cursor vs Sourcegraph Cody: embeddings and monorepo at scale (Augment)](https://www.augmentcode.com/tools/cursor-vs-sourcegraph-cody-embeddings-and-monorepo-scale)
- [Aider edit formats](https://aider.chat/docs/more/edit-formats.html)
- [Unified diffs make GPT-4 Turbo 3X less lazy (Aider)](https://aider.chat/docs/unified-diffs.html)
- [Aider code editing leaderboard](https://aider.chat/docs/leaderboards/edit.html)
- [SWE-agent (GitHub)](https://github.com/swe-agent/swe-agent)
- [mini-swe-agent — 100 lines, >74% SWE-bench Verified (GitHub)](https://github.com/SWE-agent/mini-swe-agent)
- [Claude Code hooks guide](https://code.claude.com/docs/en/hooks-guide)
- [Claude Code hooks: the deterministic control layer (Dotzlaw)](https://dotzlaw.com/insights/claude-hooks/)
- [Why each of my 95 hooks exists (Blake Crosley)](https://blakecrosley.com/blog/claude-code-hooks)
- [Claude Code subagents 2026 playbook (Developers Digest)](https://www.developersdigest.tech/blog/claude-code-agent-teams-subagents-2026)
- [Claude Code subagents 2026 (Totalum)](https://www.totalum.app/blog/claude-code-subagents-totalum)
- [AGENTS.md (open standard)](https://agents.md/)
- [Custom instructions with AGENTS.md (OpenAI Codex)](https://developers.openai.com/codex/guides/agents-md)
- [AGENTS.md vs CLAUDE.md vs .cursorrules (Morph)](https://www.morphllm.com/agents-md-guide)
- [Improving Deep Agents with Harness Engineering (LangChain)](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering)
- [Minions: Stripe's one-shot end-to-end coding agents — Part 2 (Stripe)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- [JetBrains research: efficient context management](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)

Companion reports (the "why" behind the concepts cross-linked above):
`../research/harness-engineering.md`, `../research/agent-design.md`, `../research/context-engineering.md`.
