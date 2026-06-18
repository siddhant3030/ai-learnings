# Harness Engineering: A Deep Research Report

> Prepared: June 2026
> Audience: product builders who want to go beyond tutorials

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Why It Matters](#2-why-it-matters)
3. [How Practitioners Use It](#3-how-practitioners-use-it)
4. [Design Patterns and Best Practices](#4-design-patterns-and-best-practices)
5. [Advanced Insights](#5-advanced-insights)
6. [Curated Reading List](#6-curated-reading-list)
7. [Learning Path](#7-learning-path)
8. [Practical Application](#8-practical-application)

---

## 1. Core Concepts

### The Fundamental Equation

The most useful mental model in harness engineering is this formula:

**Agent = Model + Harness**

The model is the stateless token-predicting core. The harness is everything else: the loop, tool execution, context management, memory, verification, safety enforcement, and observability. The model provides reasoning capability; the harness makes that reasoning actionable and repeatable in the real world.

Source: [Harness, Scaffold, and the AI Agent Terms Worth Getting Right (HuggingFace)](https://huggingface.co/blog/agent-glossary)

### Terminology Map

These terms are used inconsistently in the wild. Here is the most precise set of definitions:

**Scaffold**: The behavior-defining layer around the model. It includes system prompts, tool descriptions, response parsing logic, and context management. It shapes what the model sees and how it perceives its environment. Scaffolding is what the model works from.

**Harness**: The execution layer that operates the agent — calling the model, handling tool invocations, routing tool outputs back, and determining when to stop. The harness makes the agent actually run.

**Harness Engineering**: The discipline of designing the entire environment around an agent — scaffolding, constraints, feedback loops, verification, memory, and governance — to make autonomous operation reliable at scale.

**Context Engineering**: The practice of deciding what goes into the model's context window at each step: system prompts, conversation history, tool schemas, retrieved documents, dynamic state. Andrej Karpathy defined it as "the delicate art and science of filling the context window with just the right information for the next step."

Source: [Harness Engineering for AI Coding Agents (Augment Code)](https://www.augmentcode.com/guides/harness-engineering-ai-coding-agents)

### The Layered Hierarchy

Three nested design concerns, from narrowest to broadest:

| Layer | Core Question | Controls |
|---|---|---|
| Prompt Engineering | What should I ask? | Instruction text for a single turn |
| Context Engineering | What should the model see? | All tokens in one context window |
| Harness Engineering | How should the whole environment work? | External constraints, feedback loops, multi-session state |

Harness engineering operates outside the context window. It introduces context resets, structured handoffs between sessions, and phase gates that let an agent do coherent work across more than one context window.

Source: [Beyond Prompts and Context: Harness Engineering for AI Agents (MadPlay)](https://madplay.github.io/en/post/harness-engineering)

### The ETCLOVG Taxonomy

A useful practitioner framework — note that the host site (picrew.github.io) is a GitHub Pages deployment rather than a conference or journal venue, so treat this as a well-structured community taxonomy rather than peer-reviewed research. That said, the taxonomy aligns well with independently verified architectural patterns from Anthropic, Stripe, and OpenHands. Source: [picrew.github.io/LLM-Harness](https://picrew.github.io/LLM-Harness/)

It identifies seven distinct architectural layers:

**Structural Core:**
- **E (Execution)**: Sandbox constraints, managed environments, code runtimes
- **T (Tooling)**: Protocol standards (MCP, A2A), tool discovery, capability invocation
- **C (Context)**: Memory management across short-term, session, and persistent horizons
- **L (Lifecycle)**: Control flow from single-agent loops to multi-agent pipelines

**Control Plane:**
- **O (Observability)**: Tracing, cost tracking, reliability signals
- **V (Verification)**: Benchmarks, failure attribution, evaluation feedback
- **G (Governance)**: Permission models, security constraints, audit infrastructure

### Where Scaffolding Ends and Harness Begins

The confusion arises because the words are often used interchangeably in blog posts. The cleanest distinction: scaffolding is what the model sees and uses (instructions, tool schemas, initial context). The harness is what executes the model's decisions (the loop, retry logic, tool dispatch, state management). In training pipelines, these need separate consideration. In production products like Claude Code or Devin, both layers are present and usually called "the harness" in aggregate.

### Common Misconceptions

**Misconception 1: "The model is the bottleneck."**
Multiple practitioner case studies show the opposite. The binding constraint on agent reliability is infrastructure quality, not model quality. LangChain improved their coding agent from 52.8% to 66.5% on Terminal Bench 2.0 (moving from outside top 30 to top 5) without changing the underlying model at all — only harness changes. Source: primary fetch confirmed.

**Misconception 2: "More tools = better agent."**
Bad tool descriptions "send agents down completely wrong paths" (Anthropic Engineering). Having 500 tools in context is actively harmful. Stripe's Toolshed has ~500 tools but agents receive curated subsets. Progressive tool discovery (lazy loading) outperforms upfront full exposure.

**Misconception 3: "Multi-agent swarms are more powerful."**
Production evidence strongly favors single well-scoped agents with explicit human checkpoints over autonomous swarms. Swarms are brittle, expensive, and nearly impossible to debug at scale. Anthropic found multi-agent is only worth it for tasks that (a) require more than one context window, (b) are highly parallelizable, or (c) have deep tool integration. For most coding work, a single focused agent wins.

**Misconception 4: "Harness engineering is mostly prompting."**
It is mostly infrastructure. Rules files, CI gates, linters configured to provide corrective feedback, sandbox isolation, loop controllers, verification passes — these are closer to DevOps work than copywriting.

---

## 2. Why It Matters

### What Problem This Solves

Before harness engineering became a named discipline, teams would build agents, get impressive demos, and then watch them fail unpredictably in production. The failures were not model failures — they were infrastructure failures:

- Agents would complete tasks without verifying the result (victory declaration bias)
- Agents would enter doom loops, making 10+ variations on a broken approach
- Context would accumulate noise over long runs, degrading output quality (context rot)
- State would bleed between sessions or users
- Identical prompts would produce wildly different results with no traceability

One practitioner estimate (Adnan Masood, PhD — sourced from a Medium essay, not peer-reviewed research) claims "65% of enterprise AI failures trace back to Harness Defects — specifically Context Drift, Schema Misalignment, and State Degradation." Independent verification of that specific number is not available. What is verifiable from primary sources: LangChain's harness-only changes produced a 13.7-point benchmark improvement; Stripe's harness design produces over 1,300 PRs per week; the SWE-agent ACI alone took GPT-4 from ~2% to ~12% on SWE-Bench. The pattern is clear even without the specific percentage. Source: [Agent Harness Engineering — The Rise of the AI Control Plane (Adnan Masood, PhD)](https://medium.com/@adnanmasood/agent-harness-engineering-the-rise-of-the-ai-control-plane-938ead884b1d)

### Why Now

Three things converged around 2024–2026:

1. **Models became strong enough for long-horizon tasks** — but spiky. They could do impressive things but not reliably. The variance was in the harness, not the model.

2. **Token costs and autonomous loops amplified consequences** — autonomous loops amplify token usage 10–100x over chat. A misconfigured agent doesn't just give a wrong answer; it spends money, makes API calls, and potentially takes destructive actions at scale.

3. **The shift from chat to autonomous action** — when agents can execute code, push pull requests, call external APIs, and take multi-step actions, the infrastructure around them needs to enforce safety and correctness in ways a chat wrapper never had to.

Mitchell Hashimoto (HashiCorp co-founder) named the discipline in early February 2026. His formulation: "Anytime you find an agent makes a mistake, you take the time to engineer a solution so that the agent never makes that mistake again." A few days later, OpenAI's Ryan Lopopolo published the definitive post formalizing it.

### What Breaks If This Is Ignored

**At the model interface**: Agents mark tasks complete without testing. They hallucinate successful tool calls. They "one-shot overreach" — attempt entire problem solutions in single execution steps, creating undocumented entangled changes.

**At the system level**: Without context compaction, costs explode on long tasks. Without session isolation, state corrupts across users. Without verification gates, defects escape to production. Without observability, post-incident debugging is impossible.

**At the organization level**: Engineers lose trust in agents and revert to manual work. Investment in AI tooling fails to produce ROI.

---

## 3. How Practitioners Use It

### OpenAI: The Origin Case Study

In February 2026, Ryan Lopopolo published ["Harness engineering: leveraging Codex in an agent-first world"](https://openai.com/index/harness-engineering/) — though openai.com returned 403 at research time, the post was covered extensively with direct quotes in [Latent Space](https://www.latent.space/p/harness-eng) and [InfoQ](https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/). The following details are drawn from those primary-adjacent sources.

A team of 3 engineers built and shipped an internal beta product over five months where zero lines of source code were written manually and zero PRs were human-reviewed before merge (post-merge analysis only). The codebase grew to over one million lines across ~1,500 pull requests, all authored by AI agents. Daily token consumption exceeded 1 billion tokens (~$2–3k/day). The team estimates they built it in about 1/10th the time it would have taken manually. Direct quote from Lopopolo: "The only fundamentally scarce thing is the synchronous human attention of my team."

The key elements of their harness:
- **Dependency layers enforced in sequence**: Types → Config → Repo → Service → Runtime → UI, with structural tests validating compliance
- **Internal docs as single source of truth**: Hierarchical directories with design specs, execution plans, and reference materials — cross-linked and mechanically enforced through linters
- **Linters that teach rather than just report**: Feedback messages that prescribe the correct fix, not just flag the violation
- **Entropy management**: Background agents continuously detecting code divergence and opening refactoring PRs
- **Taste invariants**: Hard CI failures (not warnings) for style and architectural violations
- **500+ NPM packages** with "architecture to excess" to prevent multi-agent interference
- **Six core skills** encoding engineering standards in structured markdown
- **Strict 1-minute maximum inner loop** — they cycled through build tools (Make → Bazel → Turbo → NX) to maintain this

Sources: [Latent Space — Extreme Harness Engineering](https://www.latent.space/p/harness-eng) | [OpenAI Introduces Harness Engineering (InfoQ)](https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/)

### Stripe: Minions — 1,300 PRs Per Week

Stripe's engineering team built "Minions," an internal autonomous coding system that generates over 1,300 pull requests weekly with zero human-written code. Source: [Minions: Stripe's one-shot, end-to-end coding agents — Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)

**Blueprint Architecture**: The key innovation is alternating between deterministic and agentic nodes rather than a fully autonomous loop. A blueprint is a sequence where some nodes run fixed code paths (linting, CI execution, pushing changes) and other nodes run AI-powered reasoning (implement task, fix CI failures). Determinism for things that can be anticipated; AI for genuine unknowns.

**Devbox Infrastructure**: Rather than building custom agent sandboxes, Stripe reused their existing developer environment (AWS EC2 instances). These are pre-warmed for 10-second startup. The deepest lesson here: "What's good for humans is good for agents." Human-optimized infrastructure (fast feedback loops, isolated environments, standardized tooling) directly enabled effective agent deployment.

**The Toolshed**: A centralized internal MCP server with ~500 custom tools for internal systems and SaaS platforms. Agents receive curated subsets, not full access. Per-user customizability. Security controls preventing destructive operations.

**Bounded iteration**: After two push-and-test cycles, code goes to human review. Explicit policy: "diminishing marginal returns if an LLM is running against indefinitely many rounds of a full CI loop."

**Task scope**: Minions specialize in discrete, well-defined tasks — writing unit tests, fixing linter warnings, migrating code to new API versions, updating documentation. Not broad codebase rewrites.

### Anthropic: Multi-Agent Research System

Anthropic shipped a Research feature in April 2025 — an orchestrator-worker system where a lead agent plans, spawns 3–5 parallel subagents (each with its own context window), and synthesizes their findings. Source: [How we built our multi-agent research system (Anthropic Engineering)](https://www.anthropic.com/engineering/multi-agent-research-system)

**Key architectural decisions:**
- Subagents output directly to external filesystems rather than passing everything through the lead agent — prevents information loss during multi-stage processing
- Scaling rules embedded in prompts: simple fact-check = 1 agent with 3–10 tool calls; complex research = 10+ subagents
- A dedicated tool-testing agent improved task completion time by 40%
- Rainbow deployments to avoid disrupting running agent sessions during code updates

**Performance reality**: Multi-agent Claude beat single-agent Claude by 90.2% on their research evaluation. But cost: ~15x the tokens of a normal chat. Token usage explains 80% of performance variance; tool call count and model choice account for another 15%.

**Eight prompt engineering principles they identified**:
1. Build simulations in the console using actual prompts and tools to observe failure modes step-by-step
2. Provide detailed task descriptions including objectives, output formats, tool guidance, and clear boundaries
3. Embed explicit effort-scaling rules
4. Make tools have distinct purposes and clear descriptions
5. Enable self-improvement capability (diagnosing prompt failures)
6. Use broad-to-narrow search strategies
7. Use extended thinking as a controllable scratchpad
8. Execute 3+ tools simultaneously within subagents

### LangChain: Harness-Only Benchmark Improvement

LangChain documented improving their deepagents-cli coding agent by 13.7 percentage points (52.8% → 66.5%) on Terminal Bench 2.0, moving from outside the top 30 to top 5, without any model changes. Source: [Improving Deep Agents with Harness Engineering (LangChain)](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering)

Specific harness changes that drove this:
- **Build-Verify Loop**: Four-phase problem-solving framework (Planning → Build → Verify → Fix). A `PreCompletionChecklistMiddleware` intercepts agent exit and forces a verification pass against task specifications.
- **LoopDetectionMiddleware**: Tracks per-file edit counts. After N edits to the same file without progress, injects a reconsideration prompt. This broke the "doom loop" pattern of agents making 10+ variations on a broken approach.
- **LocalContextMiddleware**: On startup, maps the working directory and discovers available tooling via bash commands. Proactive context delivery rather than reactive correction.
- **Reasoning sandwich**: xhigh reasoning budget for planning, high for implementation, xhigh for final verification. Full xhigh throughout was worse (53.9%) due to timeouts.
- **Trace Analyzer**: Automated system that fetches experiment traces, spawns parallel error analysis agents, synthesizes findings, and generates targeted harness modifications. "Saves hours of time."

### SWE-Agent: Agent-Computer Interface Design

The Princeton NLP group's SWE-agent (NeurIPS 2024) introduced the concept of the Agent-Computer Interface (ACI). The core insight: interface design, not model capability, determines performance. Source: [SWE-agent GitHub (SWE-agent/SWE-agent)](https://github.com/swe-agent/swe-agent)

The SWE-agent ACI provides LM-centric commands and feedback formats:
- Concise file viewing commands with line numbers
- A custom code editor designed for LLM interaction
- Test execution with structured output
- Context-aware error messages

Without the ACI, raw GPT-4 on SWE-Bench resolves ~2% of issues. With the ACI, it resolves ~12–13%. Model same, harness different.

The mini-swe-agent (100 lines of code) now scores >74% on SWE-Bench Verified — demonstrating that a minimal but well-designed harness beats complex ones.

### OpenHands: Open Platform Architecture

OpenHands (All-Hands-AI) is the most architecturally complete open-source agent platform. Source: [Runtime Architecture (OpenHands Docs)](https://docs.openhands.dev/openhands/usage/architecture/runtime)

**Core design principles:**
- Docker-based sandboxed runtime for all code execution
- Central event bus: every action the agent takes and every observation flows through the EventStream
- Three-tier image tagging (source hash → dependency hash → version) to balance rebuild efficiency with reproducibility
- Plugin architecture for Jupyter (IPython execution), VS Code, and domain-specific agent skills

The separation of concerns: Agent Core (intelligence — controller, LLM communication) → Event System (publish-subscribe bus) → Runtime Layer (sandbox environments) → Tool Plugins. Failure in any one layer doesn't corrupt the others.

### The Edit Format Experiment: 6.7% → 68.3% With No Model Change

A practitioner experiment published at blog.can.ac tested three edit formats across 16 LLMs. Grok Code Fast 1 went from 6.7% to 68.3% when switching from the patch format to the hashline format — a format where each line is tagged with a 2–3 character content hash so the model references tags rather than reproducing exact text. The improvement was not because Grok Code Fast 1 became a better model; it was because the patch format's failure mode (exact string not found errors) was hiding the model's actual coding ability entirely.

Other results from the same experiment: GPT-4 Turbo went from 26% to 59%. Grok 4 saw a 61% reduction in output tokens. The weakest models gained the most from the format change.

This is the sharpest single-variable demonstration that harness design matters more than model selection for a given task. Source: [I Improved 15 LLMs at Coding in One Afternoon. Only the Harness Changed. (blog.can.ac)](https://blog.can.ac/2026/02/12/the-harness-problem/)

---

## 4. Design Patterns and Best Practices

### Pattern 1: The PEV Loop (Plan-Execute-Verify)

The most universally applied pattern. All three phases are gated, not optional:

1. **Plan**: Explicit task decomposition with acceptance criteria. Produce a structured artifact (plan.md, spec) before writing code.
2. **Execute**: Bounded by the plan. Pre-execution gates validate tool calls, arguments, and permissions.
3. **Verify**: Check actual outcomes against the plan. Not self-reported — tested, linted, and structurally validated.

The mathematical motivation: with 85% per-step accuracy over 10 steps, system success rate is ~0.85^10 ≈ 20%. Adding a verification gate after each major phase recovers much of this loss.

### Pattern 2: Constraint Layers (Feedforward)

Reduce the solution space before generation rather than catching errors after. Three types:

- **Rules files** (AGENTS.md, CLAUDE.md, .cursorrules): Persistent behavioral guidelines. Keep them under 500 lines — sprawling instruction files are largely ignored. Use concrete, actionable rules: "Never run DROP statements without explicit confirmation" beats "be careful with the database."
- **Linter configuration**: Encode architectural standards as hard CI failures, not warnings. Disable inline suppressions (`// eslint-disable-next-line`) so agents cannot circumvent them.
- **Type systems and dependency layers**: Enforce module boundaries structurally (OpenAI's Types → Config → Repo → Service → Runtime → UI chain).

### Pattern 3: Feedback Loop Architecture

Errors returned to agents must prescribe the fix, not just report the violation. "Violation detected" causes agents to guess. "`use logger.info({event: 'name', ...data})` instead of `console.log`" causes agents to fix.

Bounded feedback loops beat infinite ones. Stripe caps CI retries at two cycles. After two push-and-test attempts, a human reviews. LangChain discovered diminishing returns beyond bounded iteration loops.

### Pattern 4: Sandbox Isolation

Every agent session runs in its own isolated environment. The gold standard is Firecracker microVMs (AWS AgentCore), where each session gets its own CPU, memory, filesystem, and shell. Docker containers are the practical minimum. Local processes are acceptable for low-stakes development work.

Isolation serves five purposes: security, consistency, resource control, session independence, and reproducibility. Without it, state bleeds between users and sessions, and failures compound.

### Pattern 5: Progressive Tool Disclosure

Do not load all tool schemas into context at startup. Use lazy loading and keyword-based tool discovery (OpenDev's `search_tools` MCP command). Stripe's Toolshed: ~500 tools available, curated subsets per agent.

**Tool design principles from Anthropic and MCP best practices:**
- Each tool should have a single clear purpose
- Parameter descriptions should include what values are expected, how they affect behavior, and constraints
- Distinguish tool types: destructive, idempotent, open-world
- Avoid CRUD-style APIs — model tool calls around user goals, not data operations

### Pattern 6: The Reasoning Sandwich

For tasks with clear planning and verification phases, allocate reasoning budget asymmetrically:

- High reasoning compute for **planning** (understanding the problem fully before acting)
- Standard compute for **execution** (the bulk of tool calls)
- High reasoning compute for **final verification** (did we actually satisfy the requirements?)

LangChain's empirical data: full xhigh throughout scored 53.9% due to timeouts; the sandwich scored 63.6%.

### Pattern 7: Context Compaction

Long-running agents accumulate noise. Five-stage progressive compaction (from OpenDev's architecture):

1. Truncation of older observations
2. Summarization of past interactions
3. Abstraction to high-level patterns
4. Removal of redundant information
5. Aggressive compression if needed

Anthropic's multi-agent research system saved plans to persistent memory when context approached 200,000 tokens, then handed off to a fresh subagent with a clean context.

**Practical benchmarks**: Server-side compaction achieves 84% token reduction (documented in awesome-harness-engineering). Symbol graph indexing achieves 120x token reduction over raw file loading.

### Pattern 8: Defense in Depth for Safety

No single safety mechanism is exclusively relied upon. OpenDev's five independent layers:

1. Prompt-level guardrails (security policy, read-before-edit, git workflow enforcement)
2. Schema-level tool restrictions (plan-mode whitelist, per-subagent filtering)
3. Runtime approval system (Manual/Semi-Auto/Auto modes with persistent permission caching)
4. Tool-level validation (dangerous command blocklist, stale-read detection, output truncation, timeouts)
5. Lifecycle hooks (pre-tool blocking via exit code, argument mutation, user-defined scripts)

If any one layer fails, the others hold.

### Anti-Patterns

**The Doom Loop**: Agent commits to an approach, makes 10+ variations on it without progress. Fix: `LoopDetectionMiddleware` tracking per-file edit counts.

**Victory Declaration Bias**: Agent marks task complete without running tests. Fix: `PreCompletionChecklistMiddleware` intercepting exit and forcing verification.

**Context Anxiety**: As context fills, agents accelerate toward completion and cut corners. Fix: Explicit time/token budget warnings injected into context before the agent is deep into a task.

**One-Shot Overreach**: Attempting entire problem solutions in single execution steps. Fix: Plan-Execute-Verify with explicit task decomposition required before implementation.

**Tool Kitchen-Sink**: Loading all available tools into context. Fix: Progressive disclosure via tool discovery.

**Spec Drift**: Running agents for a long time without a living spec to verify against. Fix: Maintain a structured plan artifact that the verifier checks against.

**Chaotic Multi-Agent Swarms**: Launching many agents without explicit coordination contracts. Fix: Bounded workflows with explicit delegation contracts, supervisor patterns, and phase gating. Most production teams use single well-scoped agents, not swarms.

### Tradeoffs

**Cost vs. Quality vs. Speed**: Stronger verification controls improve reliability but add latency and token overhead. Production systems must decide which risks warrant expensive verification versus asynchronous checks.

**Capability vs. Control**: Broader tool access expands task coverage but enlarges potential damage from misaligned actions. Narrow tool sets with strict schemas reduce this surface area.

**Autonomy vs. Debuggability**: More autonomous agents are harder to trace and debug. Every increase in autonomy should be matched by an increase in observability.

**Single-agent vs. Multi-agent**: Single agents are cheaper, easier to debug, and more predictable. Multi-agent is justified only when parallelism genuinely matters or a task exceeds one context window.

---

## 5. Advanced Insights

### The Orientation Tax

Agents spend thousands of tokens just mapping their environment — figuring out directory structure, available tools, code conventions — before doing any useful work. This "Orientation Tax" is a significant hidden cost.

Solutions: proactive environment mapping on startup (LocalContextMiddleware), structured AGENTS.md files as persistent orienteering context, symbol graph indexing that lets agents query code semantics rather than grepping files.

Source: [Agent Harness Engineering — The Rise of the AI Control Plane](https://medium.com/@adnanmasood/agent-harness-engineering-the-rise-of-the-ai-control-plane-938ead884b1d)

### Context Rot Is Structural, Not Random

As conversations grow, LLM attention dilutes. Longer context windows do not guarantee better access — positional biases mean information in the middle of a long context is attended to less than information at the start or end. This has been confirmed empirically: longer windows do not equal better access.

The implication: context management is not just a token cost problem, it is an attention management problem. Agents that reload relevant information at the start of each major phase outperform those that accumulate everything in a single growing context.

### The Harness Is Where Model Capability Gets Multiplied or Wasted

The theoretical model from the "Scaling the Harness" paper (arxiv:2605.26112):

**P_H = Φ(Reasoning, Memory, Context, Skills, Orchestration, Governance)**

Model scaling improves only the Reasoning component. The other five are purely harness concerns. At current model capability levels, improvements to Memory, Context construction, and Skill routing produce larger performance gains than model upgrades. This will invert once models become significantly more capable, but we are not there yet.

### Stale Memory Is More Dangerous Than No Memory

Memory systems that return confident but outdated information cause agents to take destructive actions based on false beliefs. The solution is coupling persistent memory with just-in-time environment verification: before acting on a memory claim, verify it against the current state (grep, file read, API check). "Stale-but-confident" is worse than "uncertain."

Source: [From Model Scaling to System Scaling (arxiv:2605.26112)](https://arxiv.org/html/2605.26112v1)

### pass@k vs. pass^k: Evaluation Tells You Different Things

These two metrics reveal different aspects of agent reliability:

- **pass@k**: Probability that at least one of k attempts succeeds. Rises as k increases. Use when one success is enough (human reviews one PR out of many).
- **pass^k**: Probability that all k attempts succeed. Falls as k increases. Use when consistency matters (customer-facing agent that must work every time).

For a 75% per-trial success rate over 3 trials: pass@3 ≈ 98%, pass^3 ≈ 42%. Teams that only measure pass@1 miss the reliability gap that matters in production.

Source: [Demystifying Evals for AI Agents (Anthropic Engineering)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

### Expert Disagreements

**Disagreement 1: How much structure in orchestration?**
LangGraph favors explicit graph-based state machines with typed edges. CrewAI favors role-based autonomous delegation. OpenAI Swarm favors minimal hand-off conventions. Practitioners report that for production reliability, LangGraph-style explicit control flow wins. For rapid prototyping, CrewAI is faster to start.

**Disagreement 2: When to use multi-agent?**
Anthropic's empirical data: multi-agent beats single-agent by 90.2% on research tasks — but costs 15x more tokens. Most practitioners report that for coding tasks with shared codebase context, single-agent wins. Multi-agent is most clearly justified for tasks with high parallelism and independent subtask boundaries.

**Disagreement 3: How much autonomy to give agents?**
OpenAI's Codex experiment: full autonomy with human review of PRs. Stripe Minions: bounded blueprints with deterministic nodes limiting AI to specific phases. Devin 2.0: human review before execution of plans. The market has clearly moved away from "fully autonomous" toward "autonomous within explicit boundaries."

**Disagreement 4: Framework vs. custom harness?**
Established teams building production systems often write custom harnesses rather than using LangGraph or CrewAI, citing debugging complexity and abstraction leaks. Newer teams lean on frameworks. The consensus: use a framework to learn patterns, then migrate critical paths to custom implementations as you hit framework limits.

### Emerging Trends

**Agent Platforms over Agent Frameworks**: Systems are transitioning from packaging local abstractions (frameworks) to providing durable workspaces, identity, observability, governance, and human handoff across multiple runs and users (platforms). AWS AgentCore, Anthropic Claude's agent mode, and Stripe's internal system are early examples.

**Standardization of Inter-Agent Protocols**: MCP governs agent-to-tool interactions (vertical). Google's A2A protocol and the IETF's WIMSE + OAuth Token Exchange drafts govern agent-to-agent interactions (horizontal). These are converging toward something like a "USB-C for AI" — universal connectivity standards.

**Harness-as-Competitive-Moat**: While model capability converges across providers, harness quality is team-specific infrastructure requiring continuous investment. OpenAI's experiment demonstrates that the harness is the compounding asset. The longer you invest in it, the wider the gap.

**Evaluation as Engineering Discipline**: A 0% pass@100 often signals a broken task specification, not an incapable agent. The Opus 4.5 benchmark case study: an initial 42% CORE-Bench score jumped to 95% after fixing grading bugs and removing unnecessary constraints. Evals need the same engineering rigor as production systems.

---

## 6. Curated Reading List

### Foundational Posts (Read First)

**[Harness engineering: leveraging Codex in an agent-first world (OpenAI, Ryan Lopopolo, Feb 2026)](https://openai.com/index/harness-engineering/)**
Why to read: The definitional post. Documents a real production case study — 1M lines of code, zero manually written, 1,500+ PRs. All the principles emerge from practice, not theory.
Difficulty: Beginner | Time: 20 minutes | Key takeaway: The harness matures gradually; don't expect reliability from day one.

**[How we built our multi-agent research system (Anthropic Engineering)](https://www.anthropic.com/engineering/multi-agent-research-system)**
Why to read: The most technically detailed engineering post from a frontier lab about building a production multi-agent system. Eight concrete prompt engineering principles, real performance data (90.2% improvement), honest cost analysis (15x tokens).
Difficulty: Intermediate | Time: 30 minutes | Key takeaway: Token usage explains 80% of multi-agent performance variance; architecture second; model third.

**[Improving Deep Agents with Harness Engineering (LangChain Blog)](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering)**
Why to read: The most concrete single-team case study of harness-only improvements. Specific middleware patterns, specific benchmark numbers, specific failure modes diagnosed and fixed.
Difficulty: Intermediate | Time: 25 minutes | Key takeaway: Doom loops, victory declaration bias, and context anxiety are fixable infrastructure problems, not model limitations.

**[Minions: Stripe's one-shot, end-to-end coding agents — Part 2 (Stripe Dev Blog)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)**
Why to read: The best example of a blueprint architecture — hybrid deterministic/agentic nodes. Honest about what works at scale (bounded iteration, pre-warmed sandboxes, scoped tasks).
Difficulty: Intermediate | Time: 20 minutes | Key takeaway: "What's good for humans is good for agents" — reusing human developer infrastructure is often the right call.

**[I Improved 15 LLMs at Coding in One Afternoon. Only the Harness Changed. (blog.can.ac)](https://blog.can.ac/2026/02/12/the-harness-problem/)**
Why to read: The sharpest single-variable demonstration that interface design determines outcomes more than model selection. Tests three edit formats across 16 models; Grok Code Fast 1 goes from 6.7% to 68.3% purely from a format change. Short and empirical.
Difficulty: Beginner | Time: 15 minutes | Key takeaway: The model was not failing at reasoning; it was failing at mechanical text matching. Fix the interface, not the model.

### Terminology and Mental Models

**[Harness, Scaffold, and the AI Agent Terms Worth Getting Right (HuggingFace Blog)](https://huggingface.co/blog/agent-glossary)**
Why to read: The clearest disambiguation of terms used interchangeably everywhere else. Covers agent, scaffold, harness, context engineering, skills, sub-agents, and RL-specific terms.
Difficulty: Beginner | Time: 15 minutes | Key takeaway: Scaffolding = what the model sees; harness = what executes the model's decisions.

**[Agent Guardrails, Action Gates, Harnesses, and Governance (Abhishek Tiwari)](https://www.abhishek-tiwari.com/agent-guardrails-action-gates-harnesses-and-governance-four-layers-four-different-jobs/)**
Why to read: The sharpest breakdown of safety architecture into four distinct layers, each with different jobs. Concrete failure modes for each missing layer. Explains why guardrails alone are insufficient.
Difficulty: Intermediate | Time: 20 minutes | Key takeaway: A harness is infrastructure, not a safety feature. AWS AgentCore's Firecracker microVM per session is the production gold standard for isolation.

### Architecture Deep Dives

**[Building AI Coding Agents for the Terminal: Scaffolding, Harness, Context Engineering, and Lessons Learned (arxiv:2603.05344)](https://arxiv.org/html/2603.05344v1)**
Why to read: The most complete architectural description of a production-grade coding agent (OpenDev). Covers every component: dual-memory, five-stage compaction, defense-in-depth safety, LSP integration, subagent delegation, tool registry design.
Difficulty: Advanced | Time: 45 minutes | Key takeaway: Context pressure is the central design constraint — all architecture decisions flow from it.

**[Demystifying Evals for AI Agents (Anthropic Engineering)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)**
Why to read: The most thorough treatment of agent evaluation methodology from a team that has actually built it. Covers pass@k vs pass^k, grader types, evaluation harness design, common pitfalls, and practical implementation roadmap.
Difficulty: Intermediate | Time: 30 minutes | Key takeaway: Start with 20–50 tasks; a 0% pass@100 usually means broken task specification, not incapable agent.

**[From Model Scaling to System Scaling: Scaling the Harness in Agentic AI (arxiv:2605.26112)](https://arxiv.org/html/2605.26112v1)**
Why to read: The theoretical framework for thinking about harness performance as a function of six independent components. Explains why memory staleness is more dangerous than no memory. Strong empirical grounding.
Difficulty: Advanced | Time: 35 minutes | Key takeaway: Model scaling improves only the reasoning component; the other five components (memory, context, skills, orchestration, governance) are purely harness concerns.

**[Agent Harness Engineering: A Survey (picrew.github.io)](https://picrew.github.io/LLM-Harness/)**
Why to read: Academic survey with the ETCLOVG taxonomy — the most complete classification of harness components. Maps 47 lifecycle projects, 21 verification projects, and 14 governance projects. Identifies coverage gaps.
Difficulty: Advanced | Time: 40 minutes | Key takeaway: Observability and governance are underrepresented in open source vs. commercial platforms — operational maturity lags runtime infrastructure.

**[Agent Harness Engineering: The Rise of the AI Control Plane (Adnan Masood, PhD)](https://medium.com/@adnanmasood/agent-harness-engineering-the-rise-of-the-ai-control-plane-938ead884b1d)**
Why to read: Introduces the Orientation Tax, Context Rot, the Ralph Loop, the reasoning sandwich, and the economic math of harness optimization (10x cost reduction through caching and routing). Practitioner-oriented, full of specific patterns.
Difficulty: Intermediate | Time: 25 minutes | Key takeaway: The AI Control Plane framing (harness as operating system, LLM as CPU, context window as RAM) is a useful mental model; note the specific cost figures cited in this post are practitioner estimates, not benchmarked data.

### Research Papers

**[SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering (NeurIPS 2024)](https://arxiv.org/abs/2405.15793)**
Why to read: The paper that formalized the Agent-Computer Interface concept. Empirically demonstrates that interface design explains a larger performance variance than model selection.
Difficulty: Advanced | Time: 45 minutes | Key takeaway: Good ACI design leads to much better results just as good prompt engineering does — the interface the agent uses matters as much as the instructions it receives.

**[OpenHands: An Open Platform for AI Software Developers as Generalist Agents (arxiv:2407.16741)](https://arxiv.org/abs/2407.16741)**
Why to read: Technical foundation for understanding how a complete production agent platform is architected — event bus, sandbox runtime, plugin system, client-server isolation.
Difficulty: Advanced | Time: 40 minutes | Key takeaway: Separation of concerns (intelligence / event bus / runtime / plugins) makes each layer independently replaceable and testable.

### Collections

**[Awesome Harness Engineering (ai-boost/awesome-harness-engineering)](https://github.com/ai-boost/awesome-harness-engineering)**
Why to read: The most comprehensive curated collection of tools, patterns, papers, and production examples organized by the ETCLOVG taxonomy. Active project (1.9K stars).
Difficulty: Reference | Time: Ongoing | Key takeaway: Infrastructure as optimization — harness setup alone swings benchmarks 5+ percentage points. Topology choice improves performance 12–23% over model selection.

---

## 7. Learning Path

### If You Have 30 Minutes

1. Read [OpenAI's harness engineering post](https://openai.com/index/harness-engineering/) (20 min) to understand the origin and the production case study that defined the term.
2. Read the [HuggingFace agent glossary](https://huggingface.co/blog/agent-glossary) (10 min) to get the terminology straight before reading anything else.

Mental model to leave with: an agent is a model plus a harness. The harness is the binding constraint on reliability in 2025–2026, not the model.

### If You Have 2 Hours

1. OpenAI harness engineering post (20 min)
2. HuggingFace agent glossary (10 min)
3. Anthropic multi-agent research system post (30 min) — for multi-agent architecture and the eight prompting principles
4. LangChain deep agents harness engineering post (25 min) — for specific patterns: doom loop detection, verification middleware, reasoning sandwich
5. Four layers of agent safety post (Abhishek Tiwari) (20 min) — for the distinction between guardrails, action gates, harnesses, and governance
6. Stripe Minions Part 2 (20 min) — for the blueprint pattern and the "what's good for humans is good for agents" lesson

Mental model to leave with: harness = loop controller + tool gateway + verifier + retry policy + tracing. Each component addresses a specific failure mode.

### If You Want to Become Highly Knowledgeable Over the Next Week

**Day 1: Foundations**
- OpenAI harness engineering post
- HuggingFace agent glossary
- LangChain deep agents post
- Stripe Minions Part 2

**Day 2: Architecture Depth**
- OpenDev terminal agent paper (arxiv:2603.05344) — the most complete architecture reference
- Abhishek Tiwari's four layers post
- Anthropic multi-agent research system post

**Day 3: Evaluation and Verification**
- Anthropic demystifying evals post
- Work through pass@k vs pass^k examples with your own agent

**Day 4: Production Systems Survey**
- From Model Scaling to System Scaling paper (arxiv:2605.26112) — the theoretical framework
- Agent Harness Engineering survey (picrew.github.io)
- Adnan Masood's AI Control Plane post

**Day 5: Tools and Ecosystem**
- Browse the awesome-harness-engineering repo
- OpenHands Runtime Architecture docs
- MCP best practices on modelcontextprotocol.io

**Day 6: Apply It**
- Build a minimal harness for one task you care about
- Add a doom loop detector
- Add a verification pass before task completion
- Trace it and look at what the agent actually did vs. what you expected

**Day 7: Go Deeper on What You Found Surprising**
- SWE-agent NeurIPS 2024 paper (if you care about ACI design)
- OpenHands SDK paper arxiv:2511.03690 (if you care about platform architecture)
- Memory for Autonomous LLM Agents survey arxiv:2603.07670 (if memory degradation is your bottleneck)

---

## 8. Practical Application

### Building Your First Production Harness

**Start with a minimal viable harness** — three components only:

1. **A rules file** (AGENTS.md or CLAUDE.md): Cover four areas — project context, code conventions, behavioral rules, file structure. Keep it under 200 lines. Every rule should be concrete and actionable. Test it by asking: would a new engineer on the codebase understand this rule immediately?

2. **A verification pass**: Before any agent task is marked complete, force it to run the actual verification — tests, linting, type check. Use a `PreCompletionChecklistMiddleware` pattern or equivalent hook.

3. **Basic tracing**: Log every tool call, the inputs, and the outputs. This is the minimum needed to debug failures. Without this, post-incident analysis is guesswork.

Then add components based on the failure mode you observe, not speculatively.

### Context Engineering for Agents

**The four-layer pipeline** (from OpenDev architecture):
1. **Anchor-based retrieval**: Select tools and files based on relevant concepts in the current task
2. **Multi-step agentic search**: Iterative discovery for complex codebases
3. **Context assembly**: Gather relevant files and patterns
4. **Context optimization**: Apply summarization and compaction

Use symbol graphs and semantic indexing rather than file-grepping. The difference is 120x token reduction (codebase-memory-mcp reports this figure). Loading files by path is the slowest and most expensive way to give an agent code context.

### MCP Tool Design

The most common mistake: wrapping your full API as MCP tools. Instead:
- Design tools around user goals, not data operations
- Each tool should have a single clear purpose with no overlap
- Parameter descriptions must explain expected values, constraints, and side effects
- Mark tools as destructive, idempotent, or open-world — agents make better decisions with this signal
- Use progressive disclosure: start with a small tool set, let agents discover more tools via `search_tools`

For Dalgo specifically: tools like `sync_sources`, `trigger_pipeline_run`, and `run_dbt` are good candidates for explicit destructive markers and confirmation gates.

### Evaluations

**Start with 20–50 tasks, not thousands.** In early development, effect sizes are large — you will see clear signals with small samples.

**Design tasks that are independently verifiable.** If you cannot write a grader that a domain expert would agree with, the task specification is ambiguous.

**Measure pass^k for any customer-facing agent.** A 75% per-trial rate sounds good until you realize pass^3 is 42%. That is not a product you can ship.

**Use LLM-as-judge carefully.** Calibrate your LLM judge against human experts on 20–30 examples before relying on it. Judges can be systematically biased (Anthropic found SEO-optimization bias in their research system's source selection).

**Build trace analysis automation.** Manually reading traces does not scale. The LangChain trace analyzer pattern (fetch traces → spawn parallel error analysis agents → synthesize patterns → generate targeted harness fixes) is worth building early.

### Multi-Agent Orchestration

**Apply the Anthropic test before going multi-agent:**
- Does the task require more information than fits in one context window?
- Are there genuinely independent subtasks that can run in parallel?
- Does the task have deep tool integration that benefits from specialization?

If none of these are true, use a single agent.

If you do go multi-agent:
- Give each subagent a concrete objective, explicit boundaries, a specific output format, and tool guidance
- Write scaling rules into prompts: simple task = 1 agent with 3–10 tool calls; complex task = N subagents with divided responsibilities
- Output directly to external systems rather than routing everything through the orchestrator
- Use rainbow deployments to avoid disrupting running agent sessions during updates

### Guardrails and Safety

Implement in this order (each layer addresses different failure modes):

1. **Input guardrails**: Block prompt injection from retrieved documents; validate input schemas
2. **Action gates**: Before any destructive tool call, verify agent identity and permissions
3. **Sandbox isolation**: Each session in its own container or VM
4. **Output guardrails**: Check for PII, hallucinated citations, policy violations
5. **Governance**: Document what each agent is allowed to do; maintain audit trails

For NGO-context applications: put explicit human approval gates on any operation that is (a) irreversible, (b) touches user data, or (c) costs money. The user trust budget with NGOs is limited — one bad autonomous action undermines adoption.

### Observability

**Minimum viable observability**: Every tool call logged with inputs and outputs. Session-level cost tracking. Task success/failure rates over time.

**What to trace that is specific to agents**: Decision points (why did the agent choose this tool over that one?), intermediate state (what did the agent believe at each step?), interaction structures (how did agents delegate to each other?).

**Production monitoring signals to set up**: Merge/success rates, review cycle times, test pass rates, revert rates. Degradation in any of these typically points to a harness change, not a model change.

---

## Sources Referenced

- [Harness engineering: leveraging Codex in an agent-first world (OpenAI)](https://openai.com/index/harness-engineering/) — primary source; page returned 403 at research time, details sourced from Latent Space and InfoQ coverage
- [Extreme Harness Engineering for Token Billionaires (Latent Space)](https://www.latent.space/p/harness-eng) — primary-adjacent; direct quotes and technical details from Lopopolo
- [I Improved 15 LLMs at Coding in One Afternoon. Only the Harness Changed. (blog.can.ac)](https://blog.can.ac/2026/02/12/the-harness-problem/) — primary source for the edit format experiment
- [OpenAI Introduces Harness Engineering (InfoQ)](https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/)
- [How we built our multi-agent research system (Anthropic Engineering)](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Demystifying Evals for AI Agents (Anthropic Engineering)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Minions: Stripe's one-shot, end-to-end coding agents — Part 2 (Stripe Dev)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- [Improving Deep Agents with Harness Engineering (LangChain)](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering)
- [Harness Engineering for AI Coding Agents (Augment Code)](https://www.augmentcode.com/guides/harness-engineering-ai-coding-agents)
- [Harness Engineering: Making AI Coding Agents Work in 2026 (Faros.ai)](https://www.faros.ai/blog/harness-engineering)
- [Agent Guardrails, Action Gates, Harnesses, and Governance (Abhishek Tiwari)](https://www.abhishek-tiwari.com/agent-guardrails-action-gates-harnesses-and-governance-four-layers-four-different-jobs/)
- [Agent Harness Engineering: The Rise of the AI Control Plane (Adnan Masood, PhD)](https://medium.com/@adnanmasood/agent-harness-engineering-the-rise-of-the-ai-control-plane-938ead884b1d)
- [Harness, Scaffold, and the AI Agent Terms Worth Getting Right (HuggingFace)](https://huggingface.co/blog/agent-glossary)
- [Beyond Prompts and Context: Harness Engineering for AI Agents (MadPlay)](https://madplay.github.io/en/post/harness-engineering)
- [Harness Engineering Is the Primary Lever for Agent Reliability in 2025–2026 (rmax.ai)](https://rmax.ai/notes/harness-new-model-agent-systems-2026/)
- [Runtime Architecture (OpenHands Docs)](https://docs.openhands.dev/openhands/usage/architecture/runtime)
- [OpenHands: An Open Platform for AI Software Developers as Generalist Agents (arxiv:2407.16741)](https://arxiv.org/abs/2407.16741)
- [Building AI Coding Agents for the Terminal (arxiv:2603.05344)](https://arxiv.org/html/2603.05344v1)
- [From Model Scaling to System Scaling: Scaling the Harness in Agentic AI (arxiv:2605.26112)](https://arxiv.org/html/2605.26112v1)
- [Agent Harness Engineering: A Survey (picrew.github.io)](https://picrew.github.io/LLM-Harness/)
- [Awesome Harness Engineering (ai-boost)](https://github.com/ai-boost/awesome-harness-engineering)
- [SWE-agent GitHub (SWE-agent/SWE-agent)](https://github.com/swe-agent/swe-agent)
- [Agent-Native Development: A Deep Dive into Devin 2.0's Technical Design (Medium)](https://medium.com/@takafumi.endo/agent-native-development-a-deep-dive-into-devin-2-0s-technical-design-3451587d23c0)
- [What Is an AI Agent Harness? The Architecture Behind Stripe's 1,300 Weekly AI Pull Requests (MindStudio)](https://www.mindstudio.ai/blog/what-is-ai-agent-harness-stripe-minions)
- [Harness Engineering Emerges as the Fourth Paradigm of AI Engineering (TechTimes)](https://www.techtimes.com/articles/316587/20260513/harness-engineering-emerges-fourth-paradigm-ai-engineering.htm)
