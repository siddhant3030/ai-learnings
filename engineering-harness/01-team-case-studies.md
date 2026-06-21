# Engineering Harnesses: A Comparative Case Study

> Prepared: June 2026
> Companion to `../research/harness-engineering.md` (concepts, patterns, the OpenAI/SWE-agent/OpenHands/Stripe deep dives).
> This document goes **team by team**, documenting the actual scaffolding each group ships.

**Engineering harness** = everything around the model that turns a raw LLM into a system that ships code: the agent loop, the prompts/context strategy, the tool & edit interface, the sandbox, the verification loop, planning, and orchestration. For each team below the structure is the same: **Architecture · Context · Tool/Edit interface · Verification · Environment/Sandbox · Distinctive · Lessons.**

A reading note on sources: where a claim comes from a vendor blog or marketing page (especially performance numbers), it is flagged. Independent/primary numbers (Cursor's own Composer report, the SWE-agent paper, Devin's published merge rate) are treated as more reliable than third-party aggregator summaries. Anything I could not confirm from a primary source is marked *(uncertain)*.

---

## 1. Cognition / Devin — the autonomous cloud SWE agent

**Architecture.** Devin is a fully autonomous engineering agent that lives in the cloud rather than in your editor. Each task spins up its own isolated virtual machine; Devin opens a shell, browses docs, edits code, runs tests, and submits a pull request from a single natural-language prompt. Devin 2.0 introduced a cloud IDE where a user can run **multiple parallel Devin instances**, each in its own VM, and as of 2026 "Devin Local" is a rewrite that adds **subagent support** — spawning parallel sub-sessions that report to a main agent — mirroring what Devin Cloud already did. Sources: [Devin's 2025 Performance Review (Cognition)](https://cognition.com/blog/devin-annual-performance-review-2025), [Agent-Native Development: Devin 2.0's Technical Design (Medium)](https://medium.com/@takafumi.endo/agent-native-development-a-deep-dive-into-devin-2-0s-technical-design-3451587d23c0).

**Context.** Devin invests heavily in codebase comprehension. Cognition cites generating "docs for 5M lines of COBOL or 500GB repos" and credits a doubled merge rate to better whole-codebase understanding. It produces architecture diagrams, maps dependencies, and flags breaking changes before acting. Source: [Devin's 2025 Performance Review](https://cognition.com/blog/devin-annual-performance-review-2025).

**Tool/Edit interface.** Devin acts through a real shell + editor + browser inside its VM (not a constrained ACI), plus Slack/Teams as the task-intake surface. Exact edit-diff format is not publicly documented *(uncertain)*.

**Verification.** Devin plans, executes, **and validates** multi-step tasks, running tests in its sandbox. Cognition is explicit that "code quality is not straightforwardly verifiable" and that human review remains required for logic and design — verification is real but bounded, not a substitute for review. Source: [Devin's 2025 Performance Review](https://cognition.com/blog/devin-annual-performance-review-2025).

**Environment/Sandbox.** Per-task isolated cloud VM. The agent has full filesystem access inside the VM; the human never has their machine touched. Pre-scoped, async, "fleet of Devins" model for parallel migrations and test generation.

**Distinctive — the multi-agent paradox.** Cognition published one of the most-cited harness essays, ["Don't Build Multi-Agents"](https://cognition.com/blog/dont-build-multi-agents), arguing that parallel subagents are *fragile* because decision-making gets dispersed across agents that lack shared context. Their two principles: **(1)** "Share context, and share full agent traces, not just individual messages"; **(2)** "Actions carry implicit decisions, and conflicting decisions carry bad results." The famous illustration: one subagent builds a Mario-style background while another builds a mismatched bird, and the final agent can't reconcile them. The paradox: Devin Cloud and Devin Local nonetheless *do* run subagents — but **sequentially, for narrow, well-defined questions**, not as parallel decision-makers. The lesson isn't "never parallelize," it's "never split a decision across contexts."

**Lessons (published).** PR merge rate climbed from **34% → 67%** year over year, ~4× faster problem-solving, ~2× more resource-efficient. Devin works best with "clear, upfront requirements and verifiable outcomes"; it handles upfront scoping well but **not mid-task requirement changes**, and cannot own ambiguous projects end-to-end like a senior engineer. The org-level lesson: humans must learn to *manage* the agent (scope precisely) rather than expect autonomy. Source: [Devin's 2025 Performance Review](https://cognition.com/blog/devin-annual-performance-review-2025).

---

## 2. Cursor — the IDE-integrated harness + a trained-for-the-harness model

**Architecture.** Cursor is a VS Code fork whose agent runs an **orchestration loop**: the model picks a tool, the orchestrator executes it, collects the result (search hits, file contents, test output), **rebuilds the working context**, and sends it back for the next step. In Cursor 2.0 (Oct 29, 2025) they shipped **Composer**, their first in-house agentic coding model, and a multi-agent interface where several agents run on isolated git worktrees. Sources: [Cursor 2.0 changelog](https://cursor.com/changelog/2-0), [How Cursor Shipped its Coding Agent (ByteByteGo)](https://blog.bytebytego.com/p/how-cursor-shipped-its-coding-agent).

**Context.** Cursor uses **custom retrieval models** to self-gather context — the agent pulls relevant files, functions, and patterns from the codebase without the user attaching them manually, keeping only the most relevant snippets in context to avoid window overflow. Composer was specifically trained with access to "codebase-wide semantic search," grep, and terminal tools. Sources: [How Cursor Shipped its Coding Agent](https://blog.bytebytego.com/p/how-cursor-shipped-its-coding-agent), [Composer (Cursor blog)](https://cursor.com/blog/composer).

**Tool/Edit interface — the apply model.** Cursor's signature harness component is **Fast Apply / speculative edits**. The frontier model emits a *terse* edit (often with `// ... existing code ...` elisions); a separate, specially-trained ~70B "apply" model then merges that edit into the full file. The trick is **speculative edits**: the existing source is fed as "draft tokens," so most of the file validates instantly at temperature 0, reaching **~1,000 tokens/sec (~13×)**. The speculation is always validated by deterministic greedy generation, so it can't silently corrupt the file. This is a deliberate split: let the smart-but-slow model *decide* the change, let a cheap fast model *transcribe* it. Sources: [How Cursor built Fast Apply using Speculative Decoding (Fireworks)](https://fireworks.ai/blog/cursor), [How Cursor Composer and Apply Work (Morph)](https://www.morphllm.com/blog/cursor-composer-and-apply).

**Verification.** Composer was trained with **reinforcement learning across diverse dev environments**, rewarded for efficient tool use, parallelism, and presumably passing tests/lints in those environments. At inference the agent runs terminal commands and reads test output through the orchestration loop. Source: [Composer (Cursor blog)](https://cursor.com/blog/composer).

**Environment/Sandbox.** Local by default (your machine, your repo). Training ran "hundreds of thousands of concurrent sandboxed coding environments in the cloud." Cursor 2.0's multi-agent mode isolates agents on separate **git worktrees** so parallel agents don't collide. Source: [Composer (Cursor blog)](https://cursor.com/blog/composer).

**Distinctive.** Cursor is the clearest example of **co-designing the model with the harness**: Composer is a mixture-of-experts model RL-trained specifically to call *Cursor's own* tool set, optimized for *speed in the interactive loop* rather than raw benchmark score ("4× faster than similarly intelligent models"). The apply-model split is a harness invention that other tools (Morph, Windsurf) later copied.

**Lessons.** Speed inside the editing loop is itself a harness feature — a model that keeps the developer "in flow" beats a marginally smarter but slower one. The 46.9% token-reduction figure for dynamic context that circulates online is from a **third-party blog**, not Cursor, and should be treated as illustrative *(uncertain)*. Source: [Cursor Dynamic Context (supergok)](https://supergok.com/cursor-dynamic-context-ai-token-optimization/).

---

## 3. Anthropic / Claude Code — the terminal harness (skills, subagents, hooks)

**Architecture.** Claude Code runs a main agent in the terminal, with a layered system around it: a **harness** that loads instructions at specific moments, **skills** (in-context procedures), **subagents** (isolated workers), **hooks** (deterministic lifecycle interception), and **rules/CLAUDE.md** (persistent constraints). The mental model from Anthropic: harness runs the main agent; skills run *in the same context window*; subagents are *isolated* with one-way return; agent teams are *separate processes* with bidirectional comms. Sources: [Steering Claude Code (Anthropic/Claude)](https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more), [Inside Claude Code (Penligent)](https://www.penligent.ai/hackinglabs/inside-claude-code-the-architecture-behind-tools-memory-hooks-and-mcp/).

**Context — selective, persistence-tiered loading.** This is the distinctive part. Different instruction types load at different times with different persistence:
- **Skills** lazy-load: only the name + description are in context at session start; the full `SKILL.md` body loads *only when invoked*, then garbage-collects under a shared budget.
- **Subagents** run in a fully isolated context; only their *summary* returns to the parent, so exploration noise never lands in the main session. They can nest up to ~5 levels.
- **Rules** can be **path-scoped** (a `paths:` field) so they load only when relevant files are touched.
- **CLAUDE.md** stays persistent and is **re-read after compaction**, surviving context resets.

Source: [Steering Claude Code](https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more).

**Tool/Edit interface.** File read/edit, bash, web, plus **MCP** for external tools. Edits are structured str-replace / patch operations against read-before-edit'd files.

**Verification.** Verification is *configured*, not built-in: **hooks** fire on lifecycle events (PreToolUse, file edits, Stop) and can **block a tool call via exit code** — "a hook that blocks a tool call cannot be reasoned around." Teams wire hooks to run tests/linters and gate completion. Anthropic's own eval philosophy (pass@k vs pass^k, 20–50 task suites) lives in [Demystifying Evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (covered in the companion doc).

**Environment/Sandbox.** Runs locally on the developer's machine by default. Anthropic added OS-level sandboxing / permission modes; isolation is lighter than the per-VM cloud agents because the user is in the loop.

**Distinctive.** Claude Code pushes *deterministic control of a probabilistic agent* via **hooks** — institutional knowledge ("always run the linter," "never push to main") is encoded as **harness-enforced logic** the model literally cannot route around, rather than as prose in CLAUDE.md that the model may ignore. The clean separation — skills (knowledge), subagents (context isolation), hooks (enforcement), rules (constraints) — is its own design lesson: the article notes that *confusing these boundaries* (routing logic in a subagent prompt instead of a hook) "is the root cause of most 'my agent setup is a mess' situations." Source: [Steering Claude Code](https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more).

**Lessons.** Subagents protect the main context window — the single most reusable Claude Code idea. Hooks vs. prose: enforcement belongs in code, guidance belongs in markdown.

---

## 4. OpenAI — Codex CLI + "harness engineering" as a named discipline

The companion doc covers OpenAI's landmark **harness-engineering essay** (Ryan Lopopolo, Feb 2026: 1M LOC, ~1,500 PRs, zero human-written code, dependency layers Types→Config→Repo→Service→Runtime→UI, linters that teach, 1-minute inner loop). This section adds the **product harness** — Codex CLI — that those ideas ship inside.

**Architecture.** Codex is a **multi-surface** agent: a local CLI, an IDE extension, and a cloud delegate. The CLI runs a classic iterative agent loop — read codebase → run commands in an OS-level sandbox → patch files → repeat — and can hand long tasks to **Codex Cloud**. Sources: [Codex CLI (OpenAI Developers)](https://developers.openai.com/codex/cli), [Codex CLI architecture (ZenML LLMOps)](https://www.zenml.io/llmops-database/building-production-ready-ai-agents-openai-codex-cli-architecture-and-agent-loop-design).

**Context.** Codex reads **AGENTS.md** before doing any work, layering global guidance with per-repo overrides — this is the file format OpenAI later contributed to the Agentic AI Foundation as a cross-tool standard. Source: [Custom instructions with AGENTS.md (OpenAI)](https://developers.openai.com/codex/guides/agents-md).

**Tool/Edit interface.** Shell + apply-patch file edits, sandboxed. Tools are the OS itself within the sandbox boundary.

**Verification.** Bounded by the **approval policy** rather than an automatic test gate; the harness essay's verification story is structural (taste invariants as hard CI failures, teaching linters) rather than runtime.

**Environment/Sandbox — the cleanest public sandbox model.** Codex separates two controls explicitly: **the sandbox** defines *technical* boundaries (default: **no network**, writes limited to the active workspace, enforced by OS-level mechanisms — Seatbelt on macOS, Landlock/seccomp on Linux); **the approval policy** decides *when Codex must stop and ask* before crossing them. Presets like `--sandbox workspace-write --ask-for-approval on-request` let it read/edit/run inside the workspace freely but escalate for anything outside. Sources: [Sandboxing (OpenAI Codex)](https://developers.openai.com/codex/concepts/sandboxing), [Agent approvals & security (OpenAI Codex)](https://developers.openai.com/codex/agent-approvals-security).

**Distinctive.** OpenAI is the team that **named the discipline** and ran the most extreme public experiment (the agent-first internal product). The Codex CLI's distinctiveness is the **sandbox-vs-approval orthogonality** — two independent dials instead of one "autonomy level" slider — which is a cleaner safety model than most competitors.

**Lessons (from the essay, see companion doc).** The harness matures *gradually*; reliability is an accumulated asset, not a day-one property; "anytime an agent makes a mistake, engineer a fix so it never makes that mistake again."

---

## 5. Sourcegraph — Amp / Cody, structured context for big code

**Architecture.** Sourcegraph positions itself as "the shared intelligence layer for both developers and AI agents." Amp is its agent; the differentiator is what feeds it. Source: [Context Engineering (Sourcegraph)](https://sourcegraph.com/blog/context-engineering).

**Context — the deepest "structured retrieval" bet.** Where most agents do vector/embedding RAG, Sourcegraph leans on a **deterministic code graph** built from **SCIP** (a Protobuf code-intelligence protocol). Asking for a symbol returns *the actual definition + real call sites* rather than "50 files that mention the string" — turning probabilistic text match into deterministic structure. Their stated thesis: "the retrieval pillar at production scale is rarely a single vector database" — combine deterministic structure where it exists, semantic search as fallback. Source: [Context Engineering (Sourcegraph)](https://sourcegraph.com/blog/context-engineering).

**Tool/Edit interface.** Amp exposes Sourcegraph's **MCP server** with ~13 tools: keyword + semantic search, symbol resolution, dependency tracing, cross-repo navigation, commit history. Source: [Context Engineering (Sourcegraph)](https://sourcegraph.com/blog/context-engineering).

**Verification.** Standard run-tests-in-loop; the leverage is *upstream* — better context means fewer wrong edits to verify.

**Environment/Sandbox.** Agent runtime varies (CLI/IDE/CI); the durable asset is the **indexed code graph**, not the sandbox.

**Distinctive.** Of all the teams, Sourcegraph bets hardest that **retrieval quality, not the agent loop, is the bottleneck on large/multi-repo codebases** — code has structure (call graphs, types, cross-repo refs) that text retrieval throws away.

**Lessons (vendor-reported, treat as directional).** Their CodeScaleBench (March 2026) reports file recall 0.127→0.277, precision@5 0.140→0.478, and a Kubernetes task dropping from a 2-hour timeout to **89 seconds** with structured retrieval. These are Sourcegraph's own benchmark numbers *(vendor-sourced)*. Source: [Context Engineering (Sourcegraph)](https://sourcegraph.com/blog/context-engineering).

---

## 6. Factory — Droids (enterprise multi-agent)

> Factory's deepest architecture page (DeepWiki) was unreachable at write time, so this section leans on Factory's own announcements plus search summaries and is **lighter on primary engineering detail**.

**Architecture.** A **coordinator agent decomposes work and dispatches to specialized droids** — Code, Review, Docs, Test, and Knowledge — with explicit role boundaries rather than one generalist. Each droid maintains its own conversational state and executes tools. Sources: [Droid #1 on Terminal-Bench (Factory)](https://factory.ai/news/terminal-bench), [Factory funding (SiliconANGLE)](https://siliconangle.com/2025/09/25/factory-unleashes-droids-software-agents-50m-fresh-funding/).

**Context.** Droids **bootstrap each new session with salient system information** (broad environmental scan) to save tokens and improve success on troubleshooting-heavy tasks, and **ingest organizational context** — version control, issue trackers, incident systems — so agents "onboard like seasoned engineers." Source: [Droid on Terminal-Bench (Factory)](https://factory.ai/news/terminal-bench).

**Tool/Edit interface.** Tool execution against the codebase + external systems; transparent **review workflow** for every modification. Edit-diff format not public *(uncertain)*.

**Verification.** Dedicated **Review** and **Test** droids in the pipeline; the explicit-role split *is* the verification design.

**Environment/Sandbox.** Each droid runs in a **sandboxed workspace that mirrors the project toolchain** — same category as Cursor cloud agents / Replit Agent, tuned for multi-agent coordination.

**Distinctive.** Factory is the strongest counter-bet to Cognition's "don't build multi-agents": it commits fully to a **coordinator + specialized-role** topology for the enterprise SDLC. Customers cited: MongoDB, EY, Zapier, Bayer.

**Lessons (vendor-reported).** Droid hit **58.75% on Terminal-Bench, SOTA as of Sept 25, 2025** — date-stamp this; the leaderboard has almost certainly moved since *(stale-likely)*. Source: [Droid on Terminal-Bench (Factory)](https://factory.ai/news/terminal-bench).

---

## 7. Augment Code — the context engine as the product

> Augment publishes more *positioning* than engineering internals; treat scale/latency numbers as **vendor claims** unless noted.

**Architecture.** Augment's core is not the agent loop but a **context engine** — a semantic index of the whole codebase that any agent (theirs, or via MCP, anyone's) can query. Source: [Context Engine (Augment)](https://www.augmentcode.com/context-engine), [Context Is the New Compiler (WorkOS)](https://workos.com/blog/augment-code-context-is-the-new-compiler).

**Context — real-time, branch-aware indexing.** Augment indexes hundreds of thousands of files with embeddings and **keeps the index live as the codebase changes**. The branch-aware claim is the distinctive one: a developer on branch A gets retrieval results from branch A in real time while a colleague on branch B gets B's — the index tracks per-branch state. Queries against a massive corpus return small, relevant slices "within hundreds of milliseconds." Sources: [Context Engine (Augment)](https://www.augmentcode.com/context-engine), [How Augment Solved the Large Codebase Problem (Codacy)](https://blog.codacy.com/ai-giants-how-augment-code-solved-the-large-codebase-problem). *(latency/scale numbers are vendor-stated.)*

**Tool/Edit interface.** Augment's agent does file edits + terminal; the context engine is also exposed as a standalone **MCP server** so other agents can use Augment's retrieval. Source: [Context Engine MCP now live (Augment)](https://www.augmentcode.com/blog/context-engine-mcp-now-live).

**Verification.** Standard; Augment's pitch is reuse-detection ("good code is often no new code at all" — Chris Kelly) — surfacing an existing library so the agent builds on it rather than reinventing. Source: [Context Is the New Compiler (WorkOS)](https://workos.com/blog/augment-code-context-is-the-new-compiler).

**Distinctive.** Augment is the purest expression of "**context is the moat**": they unbundled the retrieval layer from the agent and sell it as infrastructure. Where Sourcegraph uses *deterministic* SCIP structure, Augment leans on *semantic* embeddings + real-time/branch-aware freshness.

**Lessons.** "Context, not completion, is what makes code enterprise-ready." The reuse-over-generation philosophy is a genuine harness stance, not just marketing.

---

## 8. All-Hands / OpenHands (formerly OpenDevin) — the open-source platform

The companion doc covers OpenHands' five-stage compaction, defense-in-depth, and event-bus separation. This section pins the **runtime** specifics.

**Architecture — three layers + an event stream.** A **Backend** spawns agents and the event stream; an **Execution Server** runs inside a Docker container managing bash/browser/plugins; a **Client** mediates between backend and sandbox over REST. Every action and observation flows through a central **EventStream** (publish-subscribe). Source: [Runtime Architecture (OpenHands Docs)](https://docs.openhands.dev/openhands/usage/architecture/runtime).

**Context.** Microagents (domain-specific instruction files triggered by keywords) + the compaction pipeline from the companion doc.

**Tool/Edit interface — the action/observation model.** Distinctively, OpenHands models *everything* as an **Action** the agent emits and an **Observation** it receives back (run command, edit file, browse, IPython). The client "receives actions from the backend, executes them in the sandbox, and sends back observations." This uniform event abstraction is what makes the system composable and replayable. Source: [Runtime Architecture (OpenHands Docs)](https://docs.openhands.dev/openhands/usage/architecture/runtime).

**Verification.** Tests run as actions inside the sandbox; observations feed back through the event stream.

**Environment/Sandbox — Docker, with reproducibility engineering.** Each session gets a Docker sandbox addressing security, consistency, resource control, isolation, and reproducibility. A **three-tag image strategy** (versioned → lock/dependency → source-hash) balances rebuild speed against exact reproducibility — avoiding rebuilds when source is unchanged. Source: [Runtime Architecture (OpenHands Docs)](https://docs.openhands.dev/openhands/usage/architecture/runtime).

**Distinctive.** The most **architecturally legible** open-source agent — the action/observation event bus makes each layer (intelligence / events / runtime / plugins) independently replaceable and testable. It's the reference implementation people read to learn how a full agent platform is wired.

**Lessons.** Separation of concerns via an event bus is the durable idea; failure in one layer doesn't corrupt the others.

---

## 9. SWE-agent (Princeton) — the ACI thesis

The companion doc states the headline (2% → ~12.5% on SWE-bench from interface design alone; mini-swe-agent in ~100 lines hitting >74% on SWE-bench Verified). This section pins **what the ACI actually is**, because it's the intellectual root of the whole field.

**The thesis.** Just as humans use IDEs rather than raw shells, an LM needs a purpose-built **Agent-Computer Interface (ACI)**. Interface design — not model capability — explains a large share of performance variance. Source: [SWE-agent (NeurIPS 2024 / arXiv 2405.15793)](https://arxiv.org/abs/2405.15793).

**The four ACI design principles.** From the paper and docs:
1. **Actions should be simple and easy to understand** — LM-friendly commands, not man-page flags.
2. **Actions should be compact/efficient** — file navigation and editing consolidated into as few actions as possible.
3. **Environment feedback should be informative** — concise, specific feedback about each command's effect every turn.
4. **Guardrails mitigate error compounding** — e.g., the editor **runs a linter on every edit and rejects the edit if the code is syntactically invalid**, stopping a single bad edit from cascading.

Source: [Agent-Computer Interface (SWE-agent docs)](https://swe-agent.com/0.7/background/aci/).

**Concrete components.** A special **file viewer** (line-windowed, not `cat`), a **custom editor** with the lint-gate, **search** commands scoped for LMs, structured test execution. The ablation: the ACI added **+10.7 points** over a vanilla Linux-shell agent. Source: [SWE-agent paper (arXiv 2405.15793)](https://arxiv.org/pdf/2405.15793).

**Distinctive.** SWE-agent is *research, not a product* — its contribution is the **concept** every other team now builds on. The lint-on-edit guardrail in particular shows up (in spirit) in Aider, Cursor, and OpenHands.

**Lesson.** "Good ACI design helps as much as good prompt engineering." The interface the agent uses matters as much as the instructions it's given.

---

## 10. Block / Goose — the open-source, on-machine, model-agnostic agent

**Architecture.** Goose is a general-purpose agent that runs **natively on your machine** (macOS/Linux/Windows desktop app, full CLI, embeddable API), **built in Rust**. It is model-agnostic (15+ LLM providers) and extension-first. Sources: [Introducing Goose (Marc Nuri)](https://blog.marcnuri.com/goose-on-machine-ai-agent-cli-introduction), [Goose: the open-source agent that shaped MCP (Arcade)](https://www.arcade.dev/blog/goose-the-open-source-agent-that-shaped-mcp/).

**Context.** Named sessions, chat history, and **skills** for custom context; **recipes** (declarative, shareable task definitions) parameterize and constrain a run. Source: [Subagents vs Subrecipes (Goose blog)](https://block.github.io/goose/blog/2025/09/26/subagents-vs-subrecipes/).

**Tool/Edit interface — MCP-native.** Goose was the **first public MCP client** and is effectively MCP's reference implementation — Block was contributing to the spec before it shipped. Tools come almost entirely through MCP (70+ extensions). Source: [Goose: the open-source agent that shaped MCP (Arcade)](https://www.arcade.dev/blog/goose-the-open-source-agent-that-shaped-mcp/).

**Verification.** Recipes can include retry/validation steps; subagents run sub-recipes that "coordinate while handling failures gracefully." Source: [Subagents and Tasks (DeepWiki/Goose)](https://deepwiki.com/block/goose/4.1.4-subagents-and-tasks).

**Environment/Sandbox.** Runs on the local machine by default (on-machine is the point); container isolation for subagents is on the roadmap. Source: [Goose OSS Roadmap (GitHub Discussion)](https://github.com/aaif-goose/goose/discussions/6973).

**Distinctive.** Goose's place in history is **MCP**: it proved out the protocol that the whole industry now uses for tools. Architecturally it's converging on a **lead/meta-agent orchestrating parallel subagents via recipes** — explicitly choosing *subrecipes* (isolated, parameterized) over freeform subagents for reliability. In Dec 2025 Block donated Goose to the **Linux Foundation's Agentic AI Foundation**, alongside MCP and AGENTS.md. Source: [Goose review (The AI Agent Index)](https://theaiagentindex.com/agents/goose).

**Lessons.** "Stop building agents, start harnessing" — Goose's framing is that the leverage is in recipes/extensions (the harness), not the loop.

---

## 11. Stripe / Minions — internal harness reusing dev infra

> Covered in depth in the companion doc (blueprint architecture, Devbox/EC2 reuse, Toolshed ~500 tools, bounded 2-cycle iteration, "what's good for humans is good for agents," 1,300 PRs/week). **New angle here, not a re-summary:**

The Stripe lesson that distinguishes it from every other team in this document: **Stripe built almost no agent-specific infrastructure.** The sandbox is their *existing* developer environment (pre-warmed EC2 "Devboxes"); the tools are their *existing* internal APIs wrapped in one MCP server; the verification is their *existing* CI. The harness innovation is the **blueprint** — alternating deterministic nodes (lint, push, run CI) with agentic nodes (implement, fix CI) — so the agent is only invoked for genuine unknowns. Everything else is human infra repurposed. This is the strongest real-world data point for the thesis that *agent reliability is mostly an infrastructure problem you may already have solved for humans.* Source: [Minions Part 2 (Stripe Dev)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2).

---

## 12. Google — Jules, Gemini CLI, Antigravity

> Public engineering depth here is thinner than for the labs above; this is a **conscious lighter section**.

**Jules — async cloud agent.** Jules (Google Labs, Gemini 2.5 Pro) runs each task in an **isolated Google-managed Cloud VM** with its own repo checkout. A Gemini Pro instance drafts a step-by-step plan; the agent edits, runs tests, iterates on failures; delivers a **pull request** on a new branch with summary + diff + test output; then the **VM is destroyed**. Architecturally it's Devin-shaped (async, per-task VM, PR delivery) with Google's cloud + credential isolation as the selling point. Source: [Jules (Google blog)](https://blog.google/innovation-and-ai/models-and-research/google-labs/jules/).

**Antigravity — agent-first platform.** Google's agent-first dev platform (2.0 at I/O 2026) ships an **Agent Manager** surface to spawn, orchestrate, and observe multiple async agents across workspaces, plus a Managed Agents API and (with AgentKit 2.0) 16 specialized agents. It's Google's bet on the same multi-agent orchestration topology as Factory/Antigravity, productized. Sources: [Build with Google Antigravity (Google Developers)](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/), [Agent Manager (Antigravity docs)](https://antigravity.google/docs/agent-manager).

**Distinctive.** Google's edge is **cloud + credential custody**: secrets stay under Google's control, the agent never touches the developer's machine, VMs are ephemeral per task.

---

## 13. The IDE cohort — Windsurf, Cline, Aider, Zed

Grouped because each contributes one sharp harness idea.

**Windsurf (Cascade + Riptide).** Cascade is a **stateful agent** that decomposes requests into execution plans, with **real-time "flow awareness"** (tracking your edits/terminal/activity) feeding context, plus a memory bank (`.windsurfrules` + autonomously generated memories). Its retrieval engine, **Riptide**, trains a dedicated LLM to score code-snippet relevance — Codeium claims **+200% retrieval recall** over embeddings and "40× faster, 1000× cheaper" than third-party APIs *(all vendor claims — uncertain)*. Distinctive: passive **flow/activity context** that the user never has to attach. Sources: [Windsurf (Contrary Research)](https://research.contrary.com/company/windsurf), [Windsurf 2 Deep Dive (DigitalApplied)](https://www.digitalapplied.com/blog/windsurf-2-deep-dive-cascade-agents-flows-2026).

**Cline (Plan/Act + open).** Open-source, runs in VS Code (and now JetBrains/Zed/Neovim/CLI). Distinctive: an explicit **Plan mode vs Act mode** toggle — in Plan mode the agent is *directed to be more exploratory*, read more files, ask clarifying questions, and produce a strategy with **zero file writes**; only after the human aligns does Act mode execute, **approving each edit/command individually**. Uses AST-based analysis + dynamic summarization for context; `.clinerules` for project rules. The clearest productization of "**separate the thinking phase from the doing phase, with a human gate between**." Sources: [Cline (GitHub)](https://github.com/cline/cline), [Cline (Latent Space)](https://www.latent.space/p/cline).

**Aider (repo map + edit formats).** The most-studied *minimal* harness. Distinctive: the **tree-sitter repo map** — Aider parses every file into an AST, builds a graph of symbol definitions/references, runs **PageRank** to rank importance, and emits a token-budgeted map of the most relevant signatures (not full files). Its **edit-format** work is foundational: multiple coders (search/replace blocks, unified diff, whole-file, patch) **auto-selected by model capability**, with lint-and-auto-commit-to-git baked into the loop. Aider proved that *repo map + edit format* is most of what a coding harness needs. Sources: [Building a better repo map with tree-sitter (Aider)](https://aider.chat/2023/10/22/repomap.html), [Repository map (Aider docs)](https://aider.chat/docs/repomap.html).

**Zed (ACP + concurrent agents).** Zed's contribution is **protocol, not a single agent**: the **Agent Client Protocol (ACP)** — a JSON-RPC "LSP for coding agents" (created Aug 2025) — lets *any* agent run as a separate process while Zed handles the UI/diff review. Claude Code, OpenAI Codex, Gemini CLI, and OpenCode all run inside Zed via ACP. Zed's architectural bet: the future is **concurrent agents on isolated git worktrees**, with the human reviewing/merging. Sources: [Agent Client Protocol (Zed)](https://zed.dev/acp), [Zed 2025 Recap](https://zed.dev/2025).

---

## Comparison Table

| Team | Context strategy | Edit format | Verification | Sandbox | Open/Closed |
|---|---|---|---|---|---|
| **Cognition / Devin** | Whole-codebase comprehension; deep upfront scoping; full-trace sharing across sequential subagents | Real shell + editor in VM (format not public) | Plans + runs tests in VM; human review required ("not straightforwardly verifiable") | Per-task isolated cloud VM; parallel "fleet" | Closed |
| **Cursor** | Custom retrieval models self-gather; codebase-wide semantic search; Composer RL-trained for the harness | Terse edit + **Fast Apply** model (speculative decoding, ~1000 tok/s) | RL across dev envs; terminal/test output in orchestration loop | Local default; cloud sandboxes for training; **git worktrees** for parallel agents | Closed (Composer proprietary) |
| **Anthropic / Claude Code** | Persistence-tiered: lazy **skills**, isolated **subagents**, path-scoped **rules**, compaction-surviving CLAUDE.md | str-replace / patch (read-before-edit) | **Hooks** block tool calls deterministically; pass@k/pass^k eval discipline | Local + OS-level sandbox; user-in-loop | Closed (CLI; configs open) |
| **OpenAI / Codex** | **AGENTS.md** layered global+repo; iterative read→run→patch loop | apply-patch | Approval policy + structural CI/linters (taste invariants) | **Sandbox ⊥ approval**: OS-level (Seatbelt/Landlock), no-net default, workspace-write | Closed CLI; AGENTS.md open standard |
| **Sourcegraph / Amp** | **Deterministic code graph (SCIP)** + semantic fallback; ~13 MCP tools | Standard file edits | Run-tests; leverage is upstream retrieval quality | Varies (CLI/IDE/CI); durable asset is the index | Closed (Amp); Cody/SCIP partly open |
| **Factory / Droids** | Session bootstrap scan + org context (VCS/issues/incidents) | Tool edits w/ review workflow (format not public) | Dedicated **Review + Test droids** in pipeline | Sandboxed workspace mirroring toolchain | Closed |
| **Augment Code** | **Real-time, branch-aware semantic index**; sub-second retrieval; reuse-detection | Agent file edits | Standard | Local/CI; durable asset is the context engine | Closed (engine exposed via MCP) |
| **OpenHands** | Microagents + 5-stage compaction; **action/observation event bus** | Edit-via-Action (uniform action model) | Tests run as actions; observations fed back | **Docker** per session; 3-tag image reproducibility | **Open source** |
| **SWE-agent** | LM-friendly file viewer + search (ACI) | Custom editor w/ **lint-gate on every edit** | Lint guardrail rejects invalid edits; structured test exec | Container; research harness | **Open source** |
| **Block / Goose** | Sessions + skills + **recipes/subrecipes** | MCP tools (edit via extensions) | Recipe retry/validation; subagent failure handling | **On-machine** (local); containers on roadmap | **Open source** (Apache 2.0) |
| **Stripe / Minions** | Reuses human dev context; Toolshed ~500 tools (curated subsets) | (internal) | **Blueprint**: deterministic CI nodes + bounded 2-cycle iteration | **Reused human Devbox/EC2** (pre-warmed) | Internal |
| **Google / Jules** | Gemini plan → edit → test; repo checkout in VM | apply → PR diff | Runs tests, iterates on failures, delivers PR | **Ephemeral Google Cloud VM** per task | Closed |
| **Aider** | **tree-sitter repo map + PageRank**, token-budgeted | **Multiple formats auto-selected** (diff/whole/udiff/patch) + lint + auto-commit | Lint after edit; git commit per change | Local | **Open source** |
| **Cline** | AST analysis + dynamic summarization; `.clinerules` | str-replace w/ per-edit approval | Human approves each action; **Plan/Act split** | Local; browser via Puppeteer | **Open source** |
| **Windsurf** | **Flow awareness** (passive activity) + **Riptide** retrieval; memory bank | Cascade edits | Standard agent loop | Local/IDE | Closed |
| **Zed** | Editor context + delegated agents | Visual diff review of agent edits | Per-agent; human merge of worktrees | Local; **concurrent agents on git worktrees** | **Open source editor**; **ACP open standard** |

---

## What Differentiates the Leaders

Synthesizing across all sixteen, four axes separate the strong harnesses from the rest:

**1. They co-design the model *with* the harness, or co-design the harness *with* their infra.** Cursor RL-trained Composer to call *its own* tools and optimized for *loop speed*, not benchmark score. Stripe built almost no new infrastructure and instead bent the agent to fit pre-warmed human Devboxes and existing CI. Both directions beat "general model + bolted-on tools." The teams still wiring a frontier model to a generic shell are leaving the most on the table.

**2. They treat context as an engineered asset with structure, not a RAG afterthought.** The leaders split into two camps on *how*: **deterministic structure** (Sourcegraph's SCIP code graph; Aider's tree-sitter + PageRank repo map) vs. **fresh semantic indexing** (Augment's real-time branch-aware engine; Windsurf's Riptide). Both beat naive embedding-over-files. Claude Code adds a third dimension — *persistence tiering* (what loads when, and what survives compaction). Context strategy, not model choice, is where these teams actually compete.

**3. They make verification and safety deterministic — code, not prose.** The strongest harnesses move enforcement out of the prompt and into the machinery: Claude Code's **hooks** that block tool calls un-reason-aroundably; SWE-agent's **lint-gate** that rejects a syntactically-invalid edit before it lands; Stripe's **blueprint** with deterministic CI nodes; OpenAI's **sandbox-⊥-approval** orthogonality and taste-invariant CI failures. A guardrail the model can talk itself past is not a guardrail.

**4. They have a clear, defensible stance on orchestration — and most leaders are *single-threaded by default*.** Cognition argues parallel multi-agents are fragile and runs subagents only sequentially for narrow questions; Anthropic uses isolation, not parallelism, as the main subagent value; Cline/Devin gate the doing-phase behind a planning-phase. The pure multi-agent bets (Factory's coordinator+roles, Google Antigravity's Agent Manager, Goose's evolving meta-agent) are real and growing, but they invest heavily in **explicit role boundaries and full-trace sharing** precisely to avoid the failure Cognition warned about. The naive "swarm of agents" is nobody's production architecture.

The meta-point, consistent with the companion doc: **across every team, the harness — not the model — is the compounding, team-specific asset.** Model capability converges across providers; harness quality is where the durable gap lives.

---

## Sources

- [Devin's 2025 Performance Review (Cognition)](https://cognition.com/blog/devin-annual-performance-review-2025)
- [Don't Build Multi-Agents (Cognition)](https://cognition.com/blog/dont-build-multi-agents)
- [Agent-Native Development: Devin 2.0's Technical Design (Medium)](https://medium.com/@takafumi.endo/agent-native-development-a-deep-dive-into-devin-2-0s-technical-design-3451587d23c0)
- [Cursor 2.0 changelog](https://cursor.com/changelog/2-0) · [Composer (Cursor blog)](https://cursor.com/blog/composer) · [How Cursor Shipped its Coding Agent (ByteByteGo)](https://blog.bytebytego.com/p/how-cursor-shipped-its-coding-agent) · [Fast Apply / Speculative Decoding (Fireworks)](https://fireworks.ai/blog/cursor) · [Cursor Composer & Apply (Morph)](https://www.morphllm.com/blog/cursor-composer-and-apply)
- [Steering Claude Code (Claude)](https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more) · [Inside Claude Code (Penligent)](https://www.penligent.ai/hackinglabs/inside-claude-code-the-architecture-behind-tools-memory-hooks-and-mcp/)
- [Codex CLI (OpenAI)](https://developers.openai.com/codex/cli) · [Sandboxing (OpenAI Codex)](https://developers.openai.com/codex/concepts/sandboxing) · [Agent approvals & security (OpenAI Codex)](https://developers.openai.com/codex/agent-approvals-security) · [AGENTS.md (OpenAI)](https://developers.openai.com/codex/guides/agents-md)
- [Context Engineering (Sourcegraph)](https://sourcegraph.com/blog/context-engineering) · [Agentic Coding in 2026 (Sourcegraph)](https://sourcegraph.com/blog/agentic-coding)
- [Droid #1 on Terminal-Bench (Factory)](https://factory.ai/news/terminal-bench) · [Factory funding (SiliconANGLE)](https://siliconangle.com/2025/09/25/factory-unleashes-droids-software-agents-50m-fresh-funding/)
- [Context Engine (Augment)](https://www.augmentcode.com/context-engine) · [Context Is the New Compiler (WorkOS)](https://workos.com/blog/augment-code-context-is-the-new-compiler) · [Context Engine MCP now live (Augment)](https://www.augmentcode.com/blog/context-engine-mcp-now-live) · [How Augment Solved the Large Codebase Problem (Codacy)](https://blog.codacy.com/ai-giants-how-augment-code-solved-the-large-codebase-problem)
- [Runtime Architecture (OpenHands Docs)](https://docs.openhands.dev/openhands/usage/architecture/runtime)
- [SWE-agent (arXiv 2405.15793)](https://arxiv.org/abs/2405.15793) · [Agent-Computer Interface (SWE-agent docs)](https://swe-agent.com/0.7/background/aci/)
- [Introducing Goose (Marc Nuri)](https://blog.marcnuri.com/goose-on-machine-ai-agent-cli-introduction) · [Goose & MCP (Arcade)](https://www.arcade.dev/blog/goose-the-open-source-agent-that-shaped-mcp/) · [Subagents vs Subrecipes (Goose)](https://block.github.io/goose/blog/2025/09/26/subagents-vs-subrecipes/) · [Goose review (AI Agent Index)](https://theaiagentindex.com/agents/goose)
- [Minions Part 2 (Stripe Dev)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- [Jules (Google blog)](https://blog.google/innovation-and-ai/models-and-research/google-labs/jules/) · [Build with Google Antigravity (Google Developers)](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/) · [Agent Manager (Antigravity docs)](https://antigravity.google/docs/agent-manager)
- [Aider repo map with tree-sitter](https://aider.chat/2023/10/22/repomap.html) · [Repository map (Aider docs)](https://aider.chat/docs/repomap.html)
- [Cline (GitHub)](https://github.com/cline/cline) · [Cline (Latent Space)](https://www.latent.space/p/cline)
- [Windsurf (Contrary Research)](https://research.contrary.com/company/windsurf) · [Windsurf 2 Deep Dive (DigitalApplied)](https://www.digitalapplied.com/blog/windsurf-2-deep-dive-cascade-agents-flows-2026)
- [Agent Client Protocol (Zed)](https://zed.dev/acp) · [Zed 2025 Recap](https://zed.dev/2025)
