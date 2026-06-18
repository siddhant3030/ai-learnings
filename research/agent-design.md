# Agent Design: A Deep Research Report

> For product builders who want production-grade understanding, not tutorial-level summaries.
>
> Compiled from primary sources: Anthropic engineering, OpenAI, Cognition/Devin, Google DeepMind,
> LangChain's 1,340-respondent State of Agent Engineering survey, and ZenML's 1,200-deployment
> LLMOps database. All links point to the original source.

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

### The Canonical Four-Part Model

Lilian Weng (OpenAI Head of Safety) established the foundational taxonomy in June 2023:

> **Agent = LLM + Memory + Planning + Tool Use**

The LLM serves as the central controller — the "brain." Memory, planning mechanisms, and tools are
modules that extend it beyond what model weights alone can do. Every agent system maps to these four
components. Use this as the first lens when evaluating any agent architecture.

Source: [LLM Powered Autonomous Agents — Lilian Weng's Blog](https://lilianweng.github.io/posts/2023-06-23-agent/)

### Workflows vs. Agents: The Most Important Distinction

Anthropic makes a distinction that most practitioners overlook:

- **Workflow**: LLMs and tools are orchestrated through *predefined code paths*. The control flow is
  deterministic; the LLM fills in variable parts.
- **Agent**: The LLM *dynamically directs its own process and tool usage*. The model decides what to
  do next.

Most production "agents" are actually workflows. This matters enormously: workflows are more
reliable, cheaper to debug, and easier to test. Calling a workflow an "agent" adds complexity and
creates inflated expectations.

Anthropic's core recommendation: "find the simplest solution possible, and only increase complexity
when needed — which might mean not building agentic systems at all."

Source: [Building Effective AI Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents)

### Software 3.0: The Paradigm Shift

Andrej Karpathy articulated a useful mental model for understanding the substrate agents run on:

- **Software 1.0**: Explicit code
- **Software 2.0**: Learned neural network weights
- **Software 3.0**: LLMs where the context window is RAM, model weights are the CPU, and natural
  language (plus structured inputs) is the programming language

The implication for practitioners: building agents is systems programming on a new substrate.
Context architecture, information routing, and interface design replace traditional concerns about
data structures and control flow.

December 2025 was Karpathy's identified inflection point when "agentic coding actually started to
work" — agents reliably exceeded what he would write by hand.

Source: [Andrej Karpathy — Software 3.0 and Agentic Engineering](https://www.latent.space/p/s3)

### The Agentic Loop

The fundamental execution model for any agent:

```
while not done:
    1. Observe (read state, tool results, memory)
    2. Think (reason about next action)
    3. Act (call a tool, produce output, or stop)
    4. Update context with result
```

Production-hardened loops add: parallel branch execution, fault tolerance, interrupt/resume,
human-approval gates, and checkpoint recovery (load a prior state, fork execution from that point).

### Memory Types

The human memory analogy maps cleanly to agent systems:

| Memory Type | Agent Equivalent | Bounded? |
|-------------|-----------------|----------|
| Sensory | Embedding representations of raw inputs | Yes |
| Working / Short-term | In-context window (the LLM's active prompt) | Yes — hard limit |
| Episodic | Past interaction logs (what happened, when) | External storage |
| Semantic | Persistent facts and knowledge | External storage |
| Procedural | Learned "how to" behaviors | External storage |

Short-term memory is bounded by the context window. Long-term memory requires external vector
stores with retrieval algorithms (HNSW, FAISS, ANNOY). The production "write-through cache" pattern:
append to session buffer, query the long-term index for relevant entries, run an async worker to
extract and consolidate new facts.

Source: [A Practical Guide to Memory for Autonomous LLM Agents — Towards Data Science](https://towardsdatascience.com/a-practical-guide-to-memory-for-autonomous-llm-agents/)

### Common Misconceptions

**Misconception 1: "More capable model = better agent."**
Most agent failures are reliability failures, not capability failures. A 90% benchmark accuracy
becomes 70-80% in production once stochasticity, tool failures, and context degradation compound.
Infrastructure and context engineering matter more than model intelligence.

**Misconception 2: "More agents = more capability."**
Google DeepMind's December 2025 quantitative study (180 configurations) found multi-agent systems
can *degrade* performance by 39-70% on sequential tasks due to coordination overhead. Adding agents
only improves performance on genuinely parallelizable tasks.

**Misconception 3: "More context = better reasoning."**
Context rot — a real, measured phenomenon — means that adding tokens past roughly 50k-150k actively
hurts performance. Models pay less attention to information buried in the middle of long contexts
(the "lost in the middle" effect, documented by Stanford and Meta with a 15-20 percentage point
accuracy drop).

**Misconception 4: "Frameworks make agents easier to maintain."**
Anthropic explicitly recommends reducing abstraction layers in production: "don't hesitate to reduce
abstraction layers and build with basic components as you move to production." Frameworks accelerate
prototyping but create black boxes that are difficult to debug, audit, and optimize in production.

---

## 2. Why It Matters

### What Problem Agents Solve

Traditional software is brittle: every input case must be anticipated by the programmer. LLMs handle
novel inputs gracefully but a single call is limited — it cannot execute code, browse the web, query
a database, or take actions. Agents bridge this gap by giving LLMs the ability to act on the world,
not just describe it.

Agents are best suited to: tasks too complex for a single prompt, requiring multiple steps of
information gathering or action, where the exact sequence of steps depends on intermediate results.

Customer support that needs to look up account data, diagnose an issue, and apply a fix is a
canonical example. Data analysis that requires generating code, running it, observing output, and
iterating is another.

### Why Now (2024-2026)

Four shifts converged:

1. **Model capability**: Frontier models now reliably follow complex multi-step instructions and use
   tools correctly — this was not true before 2023.
2. **Tool use maturity**: Standardized tool-calling APIs (function calling, MCP) made agent-tool
   interfaces reliable.
3. **Context window expansion**: 100k-1M token windows enable keeping entire task contexts in
   memory.
4. **Infrastructure ecosystem**: Observability (LangSmith, Langfuse), orchestration (LangGraph,
   OpenAI Agents SDK), and deployment (containerized VMs, stateful execution) caught up to the
   models.

LangChain's State of Agent Engineering survey (1,340 respondents, November-December 2025): 57.3% of
organizations have agents in production, up from 51% the year before. Gartner forecasts 40% of
enterprise applications will incorporate agents by 2026, up from less than 5% in 2025.

Source: [State of Agent Engineering — LangChain](https://www.langchain.com/state-of-agent-engineering)

### What Breaks Without Good Agent Design

- **Runaway costs**: Multi-agent systems consume ~15x more tokens than single chat interactions.
  Without cost controls and checkpointing, a single malformed prompt can trigger hundreds of dollars
  in compute.
- **Compounding errors**: A small misunderstanding in step 1 propagates across hundreds of reasoning
  steps. Error amplification in independent multi-agent systems is 17.2x (vs. 4.4x with a
  centralized orchestrator).
- **Security exploits**: Prompt injection via external content (retrieved documents, web pages,
  tool results) is OWASP's #1 LLM risk for 2025. Agents that read and act on external content are
  uniquely vulnerable to indirect injection.
- **Zero measurable returns**: A 2025 MIT Media Lab / Project NANDA report found 95% of enterprise
  AI investments produced no measurable returns. The root cause: isolated experiments without
  architectural foundations for production.

---

## 3. How Practitioners Use It

### Anthropic: Multi-Agent Research System (2025)

**Architecture**: Orchestrator-worker pattern. Claude Opus 4 as lead agent; Claude Sonnet 4
subagents running in parallel.

**Results**: 90.2% performance improvement over single-agent Opus 4. Up to 90% research time
reduction through parallelization (3-5 subagents spawned simultaneously, each running 3+ tools in
parallel).

**Critical finding**: "Token usage by itself explains 80% of variance" in browsing performance.
Multi-agent systems must justify their ~15x token cost against single-agent alternatives.

**Key engineering lessons from building it:**
- Agents given the task of writing their own improved prompts after failure analysis reduced task
  completion time by 40%.
- Effort-scaling rules must be embedded in the prompt: simple queries warrant 1 agent and 3-10 tool
  calls; complex research warrants 10+ subagents with divided responsibilities.
- "Broad-to-narrow search" prompting (use short, broad queries before narrowing) corrected the
  tendency toward over-specific queries that return nothing.
- "The last mile often becomes most of the journey" — production reliability required months of
  detail-oriented prompt and tool tuning after the core architecture was built.

Source: [How We Built Our Multi-Agent Research System — Anthropic Engineering](https://www.anthropic.com/engineering/multi-agent-research-system)

### Cognition (Devin): Cloud Agent Infrastructure

Cognition built the most-discussed production coding agent. Their engineering learnings are
unusually candid.

**The security problem nobody talks about**: Simple containerization is not enough. Containers
sharing a kernel mean one compromised session can access every other container's filesystems,
credentials, and network. Cognition spent over one year building microVM-level isolation where each
agent session runs on its own dedicated kernel with fully isolated storage, networking, and compute.

**State persistence for async workflows**: Containers fail when agents need to wait — for CI, for
code review, for test reruns. Cognition built hypervisor-level state snapshots — full machine state
preservation (memory, process trees, filesystem) — allowing agents to shut down while idle and
resume precisely when triggered. This was more engineering effort than any other infrastructure
component.

**What Devin actually does well vs. poorly** (from 18 months of production):
- Excels: Clear requirements, verifiable outcomes, tasks a junior engineer would take 4-8 hours on.
  Security vulnerability fixes (one org saved "5-10% of total developer time"); ETL migration (a
  bank saw "10x improvement"); test coverage (typically 50-60% → 80-90%).
- Struggles: Ambiguous end-to-end projects, mid-task requirement changes, anything requiring
  judgment.
- Real metrics: 4x faster problem-solving, PR merge rates jumped from 34% to 67%.

**The organizational insight**: "Engineers extracting the most value from autonomous agents are not
the ones delegating the hardest problems. They decompose work into agent-suitable chunks, review AI
output with the same rigor they would apply to a junior developer's code, and make architectural
decisions that agents cannot."

Sources: [What We Learned Building Cloud Agents — Cognition](https://cognition.ai/blog/what-we-learned-building-cloud-agents),
[Devin's 2025 Performance Review — Cognition](https://cognition.ai/blog/devin-annual-performance-review-2025)

### Stripe: Minions Architecture (mid-2025)

Stripe ships approximately 1,300 AI-written pull requests per week using a structured agent harness
called "Minions." The architectural principle: narrow, well-defined agents operating on strict
"rails" governed by a directed acyclic graph.

> "Large language models perform well when the problem is contained and struggle when asked to
> understand a massive codebase holistically and make sweeping changes."

Complex compliance reviews are decomposed into bite-sized tasks, with agents prevented from
"rabbit-holing" on irrelevant data.

Source: [Stripe AI Agent Harness Architecture — MindStudio](https://www.mindstudio.ai/blog/what-is-ai-agent-harness-stripe-minions)

### Intercom: Fin AI Agent

Fin has resolved over 40 million conversations at a 67% resolution rate (December 2025). It
connects to Shopify, Salesforce, Stripe, and Jira via MCP/data connectors, turning it from a
knowledge bot into an action-taking agent. Intercom moved to outcome-based pricing (charging per
resolved conversation) — which only works when agent reliability is high enough to justify paying
for results.

### ZenML: What 1,200 Production Deployments Reveal

ZenML maintains the world's largest LLMOps case study database (1,182 entries). Key cross-cutting
findings:

- **Stripe**: Fraud detection improved from 59% to 97% accuracy.
- **Amazon Rufus**: 60% higher purchase completion rates.
- **Fewer tools = better performance**: Cubic's AI code review agent *degraded* when given more
  tools. Removing capabilities improved results.
- **Memory systems remain unsolved**: No consensus approach exists. Teams still experimenting.
- **MCP has become boring infrastructure** (which means it's working).
- **The shift that matters**: "Most quality failures trace back not to model capability but to poor
  context management."

Source: [What 1,200 Production Deployments Reveal About LLMOps — ZenML](https://www.zenml.io/blog/what-1200-production-deployments-reveal-about-llmops-in-2025)

### ZenML: The Eight Stages of a Production Coding Agent

The gap between a demo agent and a production agent is failure handling, cost awareness, and the
ability to stop, inspect, and resume at any point. The eight stages:

| Stage | Cost | Primary Failure Mode |
|-------|------|---------------------|
| Issue Analysis | $0.50-$2.00 | LLM misunderstands requirements |
| Codebase Exploration | Dozens of file reads | Critical files missed |
| Plan Formulation | One additional LLM call | Flawed strategy despite correct understanding |
| Human Approval Gate | Near-zero | Prevents $5-$20 in wasted compute |
| Code Generation | Highest (multiple LLM calls) | Generated code contains bugs |
| Testing | Medium | Where most demo agents fall apart |
| Fix Loop | Per-attempt cost | Requires per-attempt checkpointing |
| PR Creation | Low | Final packaging and description |

"A 30-second human review of the plan saves hours of wasted compute."

Infrastructure that is not optional: checkpoint caching, real process suspension (not polling
loops), per-stage cost visibility, artifact inspection.

Source: [The Anatomy of a Production Coding Agent — ZenML](https://www.zenml.io/blog/anatomy-of-a-production-agent)

---

## 4. Design Patterns and Best Practices

### The Five Canonical Workflow Patterns (Anthropic's Taxonomy)

In increasing order of complexity and autonomy:

**1. Prompt Chaining**
Sequential LLM calls where each call processes the previous output. Use when tasks have clean
sequential dependencies: write outline → write sections → edit for tone. Add programmatic validation
checkpoints between steps.

**2. Routing**
A classifier LLM directs inputs to specialized downstream handlers. Use for customer service triage,
difficulty-based model routing, or domain-specific pipelines.

**3. Parallelization (Sectioning + Voting)**
- *Sectioning*: Independent subtasks run simultaneously
- *Voting*: Same task runs multiple times across multiple models/prompts for higher confidence

Use for guardrails, code review across multiple perspectives, multi-source research.

**4. Orchestrator-Workers**
A central LLM dynamically breaks down unpredictable tasks and delegates to specialized workers.
The key pattern for research, coding agents, and anything where the decomposition is uncertain.

**5. Evaluator-Optimizer**
One LLM generates; another provides iterative feedback. Works well for literary translation, complex
writing, and tasks where quality criteria can be articulated programmatically.

Source: [Building Effective AI Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents)

### The Four Reasoning Loop Patterns

| Pattern | Mechanism | LLM Calls | Best For | Weakness |
|---------|-----------|-----------|----------|----------|
| **ReAct** | Think → Act → Observe, repeat | Per step | Exploratory, adaptive tasks | Linear cost scaling |
| **Plan-and-Execute** | Plan once upfront; cheaper model executes | 1-2 + replans | Predictable, structured workflows | Brittle when reality diverges from plan |
| **ReWOO** | Plan with placeholders, parallel tool execution, synthesize | 2 total | Multi-hop Q&A, independent data lookups | Breaks on unexpected tool results |
| **Reflexion** | Evaluate output, self-critique, retry with memory | Multiple | Coding, math, tasks with clear success criteria | Expensive; needs automated validator |

**Production hybrid** (most common): ReAct + Reflexion. Run a ReAct loop; if the result fails
validation, enter a Reflexion retry cycle. Plan-and-Execute with ReAct sub-agents is the default in
LangGraph and CrewAI.

Reflexion improved GPT-4 HumanEval pass rates from 80% to 91% in the original paper.

**Meta-principle**: "If you can't measure whether the agent succeeded, no pattern will save you."
Most failures come from flaky tools, not reasoning architecture. Establish evaluation metrics before
selecting patterns.

Source: [ReAct vs Plan-and-Execute vs ReWOO vs Reflexion — The AI Engineer](https://theaiengineer.substack.com/p/the-4-single-agent-patterns)

### The Single vs. Multi-Agent Decision Framework

**Use a single agent when:**
- One model's context window can hold all relevant state
- The task is sequential (not decomposable into parallel work)
- Coordination overhead would outweigh specialization benefits
- Reliability and debuggability are top priorities
- You are still validating whether the use case has legs

**Use multi-agent when:**
- Tasks are genuinely parallelizable (independent subtasks run simultaneously)
- Context requirements exceed a single window
- Distinct specialization is required (different tools, system prompts, or models per task type)
- The task benefits from mutual verification (one agent checking another's work)

**Google DeepMind's quantitative evidence** (December 2025 paper, 180 configurations):
- Parallelizable tasks: centralized multi-agent coordination improved performance by +81%
- Sequential tasks: multi-agent caused 39-70% degradation across all architectures
- Error amplification: independent agents = 17.2x; centralized orchestrator = 4.4x
- A predictive model classified the optimal architecture for 87% of unseen tasks using two
  properties: task decomposability and tool count

Source: [Towards a Science of Scaling Agent Systems — Google Research](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/)

### Tool Design: The Most Underinvested Area

Anthropic's team reports spending more time optimizing tools than overall agent prompts during
development. The principles:

**Right-size your tool set.** Do not wrap existing API endpoints. Build tools that match real agent
workflows. Keeping tool selections under 30 tools gives 3x better tool selection accuracy (using
RAG to select the relevant subset).

**Tool descriptions are prompt engineering.** Treat descriptions as explanations to a new team
member. Include example usage, edge cases, niche terminology, and boundaries between overlapping
tools. Small description refinements yield dramatic performance improvements.

**Return high-signal data.** Agents have limited context. An address book tool returning all
contacts is worse than a filtered search tool. Return only what's needed for the next decision.

**Add input examples.** JSON Schema defines structural validity but cannot express "when to include
optional parameters" or "which combinations make sense." Adding `input_examples` arrays improved
accuracy from 72% to 90% on complex parameter handling in Anthropic's internal testing.

**Use Poka-yoke (mistake-proofing).** Require absolute file paths instead of relative paths. Make
the easy thing also the correct thing.

**Dynamic tool loading for large tool sets.** When tool definitions exceed 10K tokens, use deferred
loading with a search mechanism. Anthropic's Tool Search Tool reduced token usage by 85% while
improving accuracy from 49% to 74% on complex tasks (Opus 4) and from 79.5% to 88.1% (Opus 4.5).

Sources: [Writing Effective Tools for AI Agents — Anthropic Engineering](https://www.anthropic.com/engineering/writing-tools-for-agents),
[Advanced Tool Use — Anthropic Engineering](https://www.anthropic.com/engineering/advanced-tool-use)

### Context Engineering: The Core Production Discipline

Context engineering emerged in mid-2025 as the practical successor to prompt engineering. Tobi
Lutke (Shopify CEO) defined it: "the art of providing all the context for the task to be plausibly
solvable by the LLM."

Where prompt engineering asks "what do I say?", context engineering asks "what should the model
know, see, and remember at the moment of action?"

**The seven components of agent context:**
1. System prompt (instructions, examples, constraints)
2. User prompt (immediate task)
3. State and history (conversation memory)
4. Long-term memory (persistent preferences, project summaries)
5. Retrieved information (RAG from documents, databases, APIs)
6. Available tools (function definitions)
7. Structured output schema

**Eight production context engineering techniques:**
1. **Structured prompt specs** — Replace prose with explicit objective, constraints, tools, and
   output contracts.
2. **Separate static and dynamic context** — Static content (coding standards, business rules) at
   the start for prompt caching; dynamic content (current state, errors) at the end to leverage
   recency bias.
3. **Minimal high-signal tokens** — Summarize documents, remove stale tool results, curate
   ruthlessly.
4. **Just-in-time retrieval** — Maintain lightweight identifiers; load data dynamically at runtime
   rather than pre-loading.
5. **Tool-augmented prompting** — Give agents retrieval tools instead of pre-loading all context.
6. **Context compaction** — Distill message history preserving architectural decisions and critical
   state while dropping raw intermediate results.
7. **Be careful with chain-of-thought in long contexts** — CoT prompting can *worsen* context rot
   by adding more tokens that push critical information into low-attention zones.
8. **Lock your judge model versions** — Model providers quietly update underlying LLMs; judges
   score differently after updates unless pinned.

Sources: [Context Engineering vs Prompt Engineering — Firecrawl](https://www.firecrawl.dev/blog/context-engineering),
[The New Skill in AI is Context Engineering — Phil Schmid](https://www.philschmid.de/context-engineering)

### Human-in-the-Loop: Three Production Patterns

**Pattern 1 — Pre-execution approval (mandatory for consequential actions)**
The agent pauses and presents a plan before executing. A 30-second human review prevents hours of
wasted compute and irreversible mistakes. LangGraph's `interrupt()` primitive is the standard
implementation.

**Pattern 2 — Human-on-the-loop (HOTL)**
The agent acts autonomously but a human monitor can interrupt, override, or roll back within a
defined window. Lower latency, but requires reliable rollback semantics.

**Pattern 3 — Confidence-based escalation**
Agent operates autonomously under normal conditions; halts and escalates when risk signals are
detected (low confidence, novel input type, high-stakes action type).

Source: [AI Human-in-the-Loop Production Oversight Patterns — Redis](https://redis.io/blog/ai-human-in-the-loop/)

### Security: The Minimal Footprint Principle

"The Unix least-privilege principle adapted for a world where your code makes runtime decisions
about what it needs to do next."

- Request only permissions required for the current task
- Avoid persisting sensitive data beyond task scope
- Prefer reversible over irreversible actions
- Scope tool access to present intent

**Just-in-time (JIT) access**: Credentials are minted at task start, scoped to the specific task,
and revoked on completion. This bounds the blast radius of any single agent failure.

**The production safety stack (four layers)**:
1. Model alignment — probabilistic, shapes behavior at training time
2. Pre-action authorization — deterministic, blocks unauthorized tool calls before execution
3. Sandboxed execution — contains blast radius of actions that pass authorization
4. Post-hoc evaluation — finds systemic issues, feeds improvements back

Source: [The Minimal Footprint Principle — TianPan.co](https://tianpan.co/blog/2026-04-17-minimal-footprint-principle-autonomous-ai-agents)

### Anti-Patterns

**1. The God Agent** — One agent given 30+ tools that does everything. Tools consume 50,000+ tokens
of context before the user says anything. Tool selection confusion is a primary failure mode.
"Twenty tools can easily consume 3,000-5,000 tokens of your context window before the user even
says anything." Break into specialists.

**2. No evaluations** — Only 12% of agent pilots reach production. The average cost of a failed
pilot is $340K. Most failures are the same seven architectural mistakes. The root cause: building
without a systematic way to measure performance. Every production agent needs test cases and
automated evaluation before shipping.

**3. Context accumulation without compression** — Appending every tool call result verbatim. After
10 calls, context is dominated by raw tool output the model re-reads unnecessarily. Compress or
summarize intermediate results before they enter the next step.

**4. Retry loops without circuit breakers** — When a tool call fails, naive agents retry
indefinitely. Rate limit errors cause agents to abandon tasks rather than retry intelligently;
schema drift errors propagate through subsequent calls. After N failures, escalate to human or
return a structured failure.

**5. Prompt-as-architecture** — Using the system prompt to implement complex business logic (state
machines, multi-step workflows). The prompt is for behavior guidance; control flow belongs in code.

**6. Shadow security** — Containerizing agents and giving them access to repos without VM-level
isolation. One compromised session can access every other container's filesystems, credentials, and
network.

---

## 5. Advanced Insights

### The Expert Disagreement: Don't Build Multi-Agents

Cognition (builders of Devin) published a counterintuitive piece: "Don't Build Multi-Agents."

Their argument: multi-agent systems suffer from two structural problems.

1. **Fragmented context**: Subagents operating with incomplete information make conflicting implicit
   decisions. One subagent builds a Super Mario-style background; another builds an inconsistent
   bird character; the combining agent cannot reconcile incompatible outputs.
2. **Implicit decision conflicts**: "Actions carry implicit decisions, and conflicting decisions
   carry bad results." Agents that cannot see each other's work build on conflicting assumptions.

Their recommendation: single-threaded linear agents for most cases. For longer tasks risking context
overflow, use a compression model to distill history into key decisions.

They remain "optimistic about long-term possibilities" but conclude that in the current state of the
technology, "running multiple agents in collaboration only results in fragile systems."

This directly contradicts multi-agent framework enthusiasm (CrewAI, AutoGen). The reconciliation:
multi-agent works when subagents do genuinely independent work with no shared state requirements. It
fails when subagents have interdependencies or need to coordinate on implicit decisions.

Source: [Don't Build Multi-Agents — Cognition](https://cognition.ai/blog/dont-build-multi-agents)

### Context Rot: The Hidden Performance Cliff

Context rot is not "context window full." It is active performance degradation well before the hard
limit:

- Stanford/Meta "Lost in the Middle" study: with 20 retrieved documents (~4,000 tokens), accuracy
  at middle positions drops 15-20 percentage points vs. first/last positions. Adding retrieved
  context actively made answers *worse* in some conditions.
- Databricks: model correctness begins dropping around 32,000 tokens for Llama 3.1 405b — far
  below its 128K limit.
- "Context rot often begins between 50k-150k tokens regardless of theoretical limits."
- Chain-of-thought prompting can *worsen* context rot by adding more tokens that push critical
  instructions into low-attention middle zones.

Production implication: do not stuff context — architect it. The winning pattern is just-in-time
injection: dynamically assemble only what's needed based on the immediate task state.

Source: [Context Rot Explained — Redis](https://redis.io/blog/context-rot/)

### The Evaluation Gap: Observability Without Evaluation

LangChain's survey reveals a critical gap in production practice:
- 89% of teams have observability implemented (they can see what happened)
- Only 52% run offline evaluations on test sets (they can measure quality over time)
- 37% run online evaluations monitoring real-world performance

Teams have tracing. Most do not have systematic quality measurement. This means most production
agents are flying without instruments that detect regressions.

**The LLM-as-judge problem**: When LLMs write test assertions, assertions reflect the current
implementation rather than intended behavior — humans must validate assertions. Model providers
quietly update underlying LLMs, causing judges to score differently; lock judge model versions.
Multi-judge validation (three independent evaluations with majority vote) helps mitigate individual
judge bias.

Source: [State of Agent Engineering — LangChain](https://www.langchain.com/state-of-agent-engineering)

### Programmatic Tool Calling: The Context Efficiency Breakthrough

Anthropic's Programmatic Tool Calling (PTC) — where Claude writes Python code orchestrating
multiple tools in a sandboxed environment — is one of the highest-leverage production optimizations:

- Average token usage dropped from 43,588 to 27,297 tokens (a **37% reduction**) on complex
  research tasks.
- Intermediate results stay isolated from the model's context until final processing completes.
- Enables loops, conditionals, and parallel operations without polluting the model's context with
  intermediate data.

Most valuable for: workflows requiring 3+ dependent tool calls, large dataset processing where only
summaries matter, and parallel operations across multiple items.

Source: [Advanced Tool Use — Anthropic Engineering](https://www.anthropic.com/engineering/advanced-tool-use)

### The Token Budget Non-Linearity for Reasoning Models

Extended thinking introduces a non-obvious tradeoff:
- Performance scales with thinking budget up to a point.
- Beyond that point, **performance drops** — excess reasoning introduces "unnecessary deliberation
  without improving decision quality."
- GPT-5.4 Mini is ~3x cheaper per token but uses ~3x more reasoning tokens, leading to comparable
  total costs with no savings.
- Filler tokens in extended thinking can represent 30%+ of total output.

The right thinking budget is not "maximum" but "tuned to task complexity."

### Indirect Prompt Injection: The Structural Security Problem

OWASP ranks indirect prompt injection as the #1 LLM risk for 2025. Unlike direct injection (user
input overrides system prompt), indirect injection arrives through content the agent reads:
- Web pages fetched during a task
- RAG documents
- Tool results
- Email or calendar content the agent processes

An attacker never touches the agent's input channel directly. Any retrieved content is an attack
surface. Standard input-filter guardrails do not address propagation pathways in multi-agent
systems.

Source: [Indirect Prompt Injection — Lakera](https://www.lakera.ai/blog/indirect-prompt-injection)

### Karpathy's Framing: Agentic Engineering vs. Vibe Coding

Karpathy's December 2025 inflection point came with a key distinction:

- **Vibe coding**: Raises the floor — anyone can build software without understanding syntax.
- **Agentic engineering**: Raises the ceiling — "the disciplined practice of coordinating AI coding
  agents to ship professional software at scale while preserving security, maintainability, and the
  quality bar."

Emerging skills that are becoming as fundamental as code review: designing prompts, managing
sessions, evaluating output.

Source: [Andrej Karpathy — From Vibe Coding to Agentic Engineering](https://websearchapi.ai/blog/andrej-karpathy-from-vibe-coding-to-agentic-engineering)

### Frameworks vs. Raw SDK: The Resolved Debate

The 2023-2024 criticism of LangChain ("piles abstractions over abstractions until pipelines become
dependency graphs that look like spaghetti code") drove the LCEL and LangGraph refactor in v1.0
(October 2025). The "black box" criticism has largely been resolved.

The remaining principle: frameworks are great for learning and rapid prototyping; they become
liabilities when you need to debug, optimize, or audit production behavior. Anthropic's position:
"reduce abstraction layers as you move to production." The practical resolution: start with a
framework to learn the patterns, then decide whether the abstraction earns its keep in your specific
production context.

---

## 6. Curated Reading List

### Foundational (Start Here)

**1. Building Effective AI Agents — Anthropic (December 2024)**
- Why: The most-cited and most actionable production guide in the field. Written by practitioners
  who observed many enterprise teams fail and succeed with agents.
- Difficulty: Beginner/Intermediate
- Time: 25 minutes
- Key takeaways: Workflow vs. agent distinction; five core workflow patterns; when not to build an
  agent; tool design principles; real examples from Coinbase, Intercom, Thomson Reuters
- Link: [anthropic.com/research/building-effective-agents](https://www.anthropic.com/research/building-effective-agents)

**2. LLM Powered Autonomous Agents — Lilian Weng (June 2023)**
- Why: The canonical taxonomy of agent components. Everything else builds on this. Written by
  OpenAI's Head of Safety.
- Difficulty: Intermediate
- Time: 45 minutes
- Key takeaways: Agent = LLM + Memory + Planning + Tools; memory types taxonomy; MRKL systems;
  HuggingGPT pattern; challenges and failure modes
- Link: [lilianweng.github.io/posts/2023-06-23-agent](https://lilianweng.github.io/posts/2023-06-23-agent/)

**3. Writing Effective Tools for AI Agents — Anthropic Engineering (2025)**
- Why: The most underinvested area of agent design. Concrete guidance on tool API design that
  directly impacts agent reliability.
- Difficulty: Intermediate
- Time: 20 minutes
- Key takeaways: Namespacing, high-signal returns, input examples (72% → 90% accuracy improvement),
  error communication, iterative evaluation workflow
- Link: [anthropic.com/engineering/writing-tools-for-agents](https://www.anthropic.com/engineering/writing-tools-for-agents)

### Production-Focused

**4. How We Built Our Multi-Agent Research System — Anthropic Engineering (2025)**
- Why: Rare inside look at a real production multi-agent system with specific numbers and failure
  modes.
- Difficulty: Intermediate/Advanced
- Time: 30 minutes
- Key takeaways: 90.2% improvement with orchestrator-worker pattern; 40% task time reduction via
  self-improving prompts; token usage explains 80% of browsing performance variance; "the last mile
  is most of the journey"
- Link: [anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system)

**5. What We Learned Building Cloud Agents — Cognition (2025)**
- Why: The most honest account of the infrastructure work required for production agents. Security
  isolation, state persistence, organizational change — not the glamorous AI work.
- Difficulty: Intermediate
- Time: 20 minutes
- Key takeaways: Container sharing is a security vulnerability; VM-level isolation required;
  hypervisor-level state snapshots for async workflows; change management is as hard as the tech
- Link: [cognition.ai/blog/what-we-learned-building-cloud-agents](https://cognition.ai/blog/what-we-learned-building-cloud-agents)

**6. Don't Build Multi-Agents — Cognition (2025)**
- Why: Contrarian and evidence-based. Forces you to actually justify multi-agent complexity before
  adding it.
- Difficulty: Beginner
- Time: 10 minutes
- Key takeaways: Fragmented context causes conflicting implicit decisions; single-threaded linear
  agents are the default for most tasks in 2025
- Link: [cognition.ai/blog/dont-build-multi-agents](https://cognition.ai/blog/dont-build-multi-agents)

**7. Advanced Tool Use — Anthropic Engineering (2025)**
- Why: The most concrete optimization guide available. Actual performance numbers on token reduction
  and accuracy improvement.
- Difficulty: Advanced
- Time: 25 minutes
- Key takeaways: Tool Search reduces token usage by 85%; Programmatic Tool Calling reduces tokens
  by 37%; Input Examples improve accuracy from 72% to 90%
- Link: [anthropic.com/engineering/advanced-tool-use](https://www.anthropic.com/engineering/advanced-tool-use)

**8. Anatomy of a Production Coding Agent — ZenML (2025)**
- Why: Translates the abstract agent loop into a concrete eight-stage system with real cost figures
  and failure modes per stage.
- Difficulty: Intermediate
- Time: 20 minutes
- Key takeaways: Eight stages with distinct failure modes; checkpoint caching is not optional;
  human approval gate prevents major cost overruns; full workflow costs $5-$20
- Link: [zenml.io/blog/anatomy-of-a-production-agent](https://www.zenml.io/blog/anatomy-of-a-production-agent)

### Research and Evidence

**9. Towards a Science of Scaling Agent Systems — Google DeepMind (December 2025)**
- Why: The first quantitative study of when multi-agent systems work vs. fail. Empirical evidence
  over intuition.
- Difficulty: Advanced
- Time: 45 minutes
- Key takeaways: +81% on parallelizable tasks; -39% to -70% on sequential tasks; 17.2x error
  amplification for independent agents; architecture predictable from task properties
- Link: [research.google/blog/towards-a-science-of-scaling-agent-systems](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/)

**10. ReAct vs Plan-and-Execute vs ReWOO vs Reflexion — The AI Engineer**
- Why: Clearest comparison of the four foundational reasoning patterns with tradeoffs and production
  guidance.
- Difficulty: Intermediate
- Time: 20 minutes
- Key takeaways: Start with ReAct; move to Plan-and-Execute when re-deriving the same plan
  repeatedly; add Reflexion as outer loop when output quality matters more than speed
- Link: [theaiengineer.substack.com/p/the-4-single-agent-patterns](https://theaiengineer.substack.com/p/the-4-single-agent-patterns)

**11. State of Agent Engineering — LangChain (November-December 2025, 1,340 respondents)**
- Why: The most comprehensive survey of what teams actually do in production. Data over anecdote.
- Difficulty: Beginner
- Time: 30 minutes
- Key takeaways: 57% have agents in production; 89% have observability but only 52% have evals (a
  dangerous gap); quality is the top barrier, not cost; 75% use multiple models
- Link: [langchain.com/state-of-agent-engineering](https://www.langchain.com/state-of-agent-engineering)

### Context Engineering

**12. The New Skill in AI is Context Engineering — Phil Schmid**
- Why: Clear technical articulation of the paradigm shift, with practical implementation guidance.
- Difficulty: Beginner/Intermediate
- Time: 15 minutes
- Key takeaways: Seven components of context; context engineering is a system not a template; most
  agent failures are context failures not model failures
- Link: [philschmid.de/context-engineering](https://www.philschmid.de/context-engineering)

**13. Context Rot Explained — Redis**
- Why: Defines the most important and most overlooked production failure mode in agent systems.
  Concrete detection methods and mitigation strategies.
- Difficulty: Intermediate
- Time: 20 minutes
- Key takeaways: 15-20% accuracy drop from positional bias alone; context rot begins before 150K
  tokens; external memory architecture as the mitigation
- Link: [redis.io/blog/context-rot](https://redis.io/blog/context-rot/)

---

## 7. Learning Path

### If You Have 30 Minutes

1. Read Anthropic's "Building Effective AI Agents" (25 min)
2. Skim Cognition's "Don't Build Multi-Agents" (5 min)

After these two: you understand the workflow-vs.-agent distinction, the five canonical workflow
patterns, why simplicity matters, and the single most common mistake in agent architecture.

### If You Have 2 Hours

In this sequence:

1. Anthropic's "Building Effective AI Agents" (25 min)
2. Lilian Weng's "LLM Powered Autonomous Agents" (45 min) — the canonical taxonomy
3. Anthropic's "Writing Effective Tools" (20 min) — the most underinvested area
4. "ReAct vs Plan-and-Execute vs ReWOO vs Reflexion" (20 min) — reasoning patterns
5. Cognition's "Don't Build Multi-Agents" (10 min) — the important counterargument

After 2 hours: full mental model of how agents work, the four reasoning patterns, the tool design
principles, and a calibrated view of when multi-agent is and is not appropriate.

### If You Want to Become Highly Knowledgeable Over One Week

**Day 1 — Foundations**
- Lilian Weng's agent survey
- Anthropic's "Building Effective AI Agents"
- Phil Schmid's context engineering piece

**Day 2 — Reasoning Patterns and Architecture**
- ReAct vs. Plan-and-Execute vs. ReWOO vs. Reflexion comparison
- Google DeepMind's scaling agent systems paper (the quantitative evidence)
- Redis on context rot

**Day 3 — Production Systems**
- Anthropic's multi-agent research system build post
- Cognition's cloud agents learnings + Devin performance review
- ZenML's anatomy of a production coding agent

**Day 4 — Tool Design and Context Engineering**
- Anthropic's "Writing Effective Tools"
- Anthropic's "Advanced Tool Use" (Tool Search, Programmatic Tool Calling, Input Examples)
- Firecrawl's context engineering production techniques

**Day 5 — Evaluation, Observability, and Security**
- LangChain's State of Agent Engineering survey
- ZenML's "What 1,200 Production Deployments Reveal"
- Lakera on indirect prompt injection
- Minimal footprint principle

**Day 6 — Expert Disagreements and Advanced Topics**
- Cognition's "Don't Build Multi-Agents"
- Karpathy's agentic engineering framing
- LangChain/framework vs. raw SDK debate

**Day 7 — Hands-On Application**
Build a simple agent using the Anthropic or OpenAI SDK directly (not a framework). Implement:
a two-stage workflow (routing + execution), at least one tool with proper documentation and input
examples, and a basic eval script that scores outputs against a rubric. Observe cost and token usage
per step. Compare against your mental model from Days 1-6.

---

## 8. Practical Application

### For Product Builders Right Now

**Start with a workflow, not an agent.** Before building an autonomous agent, ask: can this be a
deterministic workflow where LLMs fill in the variable parts? Most tasks that seem to require agents
are well-served by prompt chaining or orchestrator-worker workflows with predefined code paths.

**Evaluation first.** You cannot improve what you cannot measure. Before writing the agent loop,
define what success looks like and how you will measure it programmatically. LLM-as-judge is
acceptable but lock your judge model version, validate judge assertions with humans, and use
majority vote across three judges.

**One tool at a time.** The most common production failure is giving one agent 30+ tools. Keep tool
sets under 20. Use tool descriptions as prompt engineering. Use dynamic tool loading for larger
libraries.

**Instrument before you deploy.** Minimum viable observability: trace every tool call with
input/output, record token usage per step, log success/failure per run, sample 5% of runs for human
review.

**The "dumb agents, smart orchestration" principle** (from ZenML field notes): The best production
agents are simple, focused, and predictable, combined with smart orchestration that handles routing,
retries, costs, and human touchpoints. Resist the urge to make the agent smarter; invest in the
surrounding system.

### Applying This to Dalgo (MCP, Multi-Agent, Evaluations, NGO Context)

**MCP tool design for Dalgo**: Design each Dalgo MCP tool following Anthropic's principles. Return
high-signal data, not raw database dumps. Add `input_examples` to all tool definitions — this is
the highest-ROI documentation investment. Write descriptions as explanations to a new team member.
For tools with large response sizes (e.g., table data), implement pagination and filtering so agents
never receive more than they need.

**Context engineering for Dalgo agents**: Agent context should be structured as: (1) the user's
organization context and permissions at the start (static, benefits from prompt caching), (2) the
current task and relevant conversation history, (3) only the retrieved data needed for the immediate
decision (just-in-time, not pre-loaded). Avoid loading all available schema/table information into
context upfront.

**Single vs. multi-agent for Dalgo features**: Use single agents for most interactive features
(answering questions about pipelines, explaining errors, generating dbt models). Consider
orchestrator-worker only for genuinely parallel tasks: a research agent spawning subagents to
simultaneously explore different schemas, or a bulk operation agent running the same check across
multiple pipelines. Do not add multi-agent complexity for sequential workflows.

**Evaluations for NGO users**: Human evaluation is the gold standard here — non-technical users'
trust is the metric that matters. Build a sample of 20-50 test cases from real NGO usage. Establish
rubrics (was the answer accurate? was it understandable without technical background?). Run against
these test cases before deploying prompt or tool changes.

**Guardrails for sensitive NGO data**: Implement the four-layer security stack — prompt-level
guardrails, pre-action authorization for any write operations (modifying pipelines, publishing
dashboards, running dbt), sandboxed execution, and audit trails. Treat every external data source
the agent reads as a potential indirect prompt injection vector.

**Human approval gates for irreversible actions**: Any destructive or irreversible action must
require explicit human approval with a checkpoint pattern — showing the user what will happen and
asking for confirmation. This is non-negotiable for NGO users who may not understand the
consequences of pipeline or data changes. The 30-second review prevents irreversible mistakes.

**Evaluator-optimizer for dbt generation**: When generating dbt models: (1) generator writes the
SQL, (2) evaluator checks against schema (are referenced columns real?), runs syntax check, verifies
the transformation logic matches the user's intent, (3) generator revises. This catches the "looked
right but was wrong" failure mode before it reaches users.

---

## Sources

All claims in this report are sourced to primary sources:

- [Building Effective AI Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents)
- [How We Built Our Multi-Agent Research System — Anthropic Engineering](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Writing Effective Tools for AI Agents — Anthropic Engineering](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Advanced Tool Use — Anthropic Engineering](https://www.anthropic.com/engineering/advanced-tool-use)
- [LLM Powered Autonomous Agents — Lilian Weng](https://lilianweng.github.io/posts/2023-06-23-agent/)
- [What We Learned Building Cloud Agents — Cognition](https://cognition.ai/blog/what-we-learned-building-cloud-agents)
- [Devin's 2025 Performance Review — Cognition](https://cognition.ai/blog/devin-annual-performance-review-2025)
- [Don't Build Multi-Agents — Cognition](https://cognition.ai/blog/dont-build-multi-agents)
- [Towards a Science of Scaling Agent Systems — Google Research](https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/)
- [State of Agent Engineering — LangChain](https://www.langchain.com/state-of-agent-engineering)
- [ReAct vs Plan-and-Execute vs ReWOO vs Reflexion — The AI Engineer](https://theaiengineer.substack.com/p/the-4-single-agent-patterns)
- [Context Rot Explained — Redis](https://redis.io/blog/context-rot/)
- [The New Skill in AI is Context Engineering — Phil Schmid](https://www.philschmid.de/context-engineering)
- [Context Engineering vs Prompt Engineering — Firecrawl](https://www.firecrawl.dev/blog/context-engineering)
- [Anatomy of a Production Coding Agent — ZenML](https://www.zenml.io/blog/anatomy-of-a-production-agent)
- [What 1,200 Production Deployments Reveal — ZenML](https://www.zenml.io/blog/what-1200-production-deployments-reveal-about-llmops-in-2025)
- [Minimal Footprint Principle — TianPan.co](https://tianpan.co/blog/2026-04-17-minimal-footprint-principle-autonomous-ai-agents)
- [Indirect Prompt Injection — Lakera](https://www.lakera.ai/blog/indirect-prompt-injection)
- [Andrej Karpathy on Agentic Engineering](https://websearchapi.ai/blog/andrej-karpathy-from-vibe-coding-to-agentic-engineering)
- [AI Human-in-the-Loop Production Patterns — Redis](https://redis.io/blog/ai-human-in-the-loop/)
- [Stripe AI Agent Harness Architecture — MindStudio](https://www.mindstudio.ai/blog/what-is-ai-agent-harness-stripe-minions)
- [The God Agent Anti-Pattern — DEV Community](https://dev.to/thedailyagent/the-god-agent-mistake-why-one-mega-agent-always-fails-in-production-1fk1)
- [ReliabilityBench: Evaluating LLM Agent Reliability Under Production Stress — arXiv](https://arxiv.org/pdf/2601.06112)
