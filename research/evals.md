# LLM / AI Agent Evaluation Systems: A Deep Research Report

**Audience**: Product builders who want to go beyond tutorials and understand how experienced teams think about evals.  
**Last updated**: June 2026  
**Scope**: LLM evals and AI agent evaluation systems — concepts, production practices, design patterns, tools, and learning paths.

---

## 1. Core Concepts

### What are Evals?

An **eval** (evaluation) is a test that provides an input to an AI system and then applies grading logic to measure success. This definition from Anthropic's engineering team is precise: it is not just "running some prompts" — it is a structured test with defined inputs and measurable success criteria.

Evals exist at two distinct levels that practitioners often confuse:

- **Foundation model benchmarks** (MMLU, HumanEval, HellaSwag, SWE-bench): standardised benchmarks used to compare base models. These are academic-adjacent and relevant when choosing a model, not when shipping a product.
- **Product evals**: domain-specific evaluation systems that measure whether your specific LLM-powered product does what it's supposed to do for your users. This report is primarily about the second type.

Eugene Yan writes in [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/): "How important evals are to the team is a major differentiator between folks rushing out hot garbage and those seriously building products."

### Essential Terminology (Anthropic's Definitions)

Anthropic's [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) provides five terms every team should adopt verbatim to prevent vocabulary disagreements:

| Term | Definition |
|---|---|
| **Task / Problem** | A single test with defined inputs and success criteria |
| **Trial** | One attempt at a task; multiple trials produce statistically reliable results |
| **Grader** | The logic that scores agent performance; a task can have multiple graders |
| **Transcript / Trace** | The complete record of a trial — outputs, tool calls, reasoning, and interactions |
| **Evaluation Harness** | Infrastructure that runs evals end-to-end with concurrent task execution |

### The Three Grader Types

Every evaluation system uses some combination of these:

**1. Code-based graders**
Methods: string matching, regex, binary tests, static analysis, outcome verification, tool call verification.  
Strengths: fast, cheap, deterministic, debuggable.  
Weaknesses: brittle to valid variation; limited nuance.

**2. Model-based graders (LLM-as-judge)**
Methods: rubric scoring, natural language assertions, pairwise comparison, reference-based evaluation.  
Strengths: flexible, scalable, captures nuance, handles open-ended tasks.  
Weaknesses: non-deterministic, expensive, introduces its own biases, requires calibration.

**3. Human graders**
Methods: domain expert review, crowdsourced annotation, spot-check sampling, A/B testing.  
Strengths: gold-standard quality, aligns with expert judgment, calibrates model-based systems.  
Weaknesses: expensive, slow, doesn't scale.

The general rule: use the cheapest, most reliable method for each check. Prefer code-based graders wherever a deterministic check exists; use model-based graders only when nuance is required; use humans to calibrate model graders and catch what automation misses.

### Capability Evals vs. Regression Evals

A distinction Anthropic makes that most teams miss:

- **Capability evals**: "What can this system do well?" — Start at low pass rates on difficult tasks. The goal is to create measurable improvement pathways. As agents improve, these graduate into regression suites.
- **Regression evals**: "Does the system still handle previously-working tasks?" — Should maintain ~100% pass rates. Wire these into CI/CD to fail builds on regressions.

### The Evaluation Hierarchy (Cost vs. Impact)

From Hamel Husain's [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/):

1. **Unit tests / assertions / regex** — Cheapest. Run on every commit.
2. **Human & model eval** — Manual inspection plus LLM-based critique on regular cadence.
3. **A/B testing** — Most expensive. Reserved for mature products with real users and measurable outcomes.

Don't skip to expensive layers first. Most improvements emerge from disciplined unit tests and error analysis.

### Non-Determinism Metrics (Agent-specific)

Two important statistical measures for agents that run multiple times:

- **pass@k**: Probability of at least one correct solution in k attempts. Rises as k increases. Relevant for scenarios where one working solution suffices.
- **pass^k**: Probability that all k trials succeed. Falls as k increases. Critical for customer-facing agents requiring reliable behavior every time.

### Common Misconceptions

1. **"Evals are a QA step after building"** — Wrong. The best teams treat evals as the central engineering activity, done before and during building. Eval-Driven Development starts with evals that define planned capabilities, then implements the agent to fulfill them.

2. **"A high eval score means the product is good"** — Only if the eval is well-designed. An uncalibrated LLM judge can show green dashboards while having Cohen's kappa of 0.31 against domain experts. The dashboard is evidence of a working eval, not a working product.

3. **"Off-the-shelf metrics are fine"** — Hamel Husain: "All you get from prefab evals is false confidence that's unjustified." ROUGE and BERTScore don't catch domain-specific failures. MMLU produces significantly different rankings depending on implementation details.

4. **"You need hundreds of test cases to start"** — 20-50 cases drawn from real failures is an excellent start. In early development, each change has large effect sizes; small sample sizes suffice.

5. **"Model upgrades make evals obsolete"** — The opposite is true. Teams with evals can evaluate a new model in days. Teams without them spend weeks testing and often ship blind.

---

## 2. Why It Matters

### The Problem Evals Solve

LLM applications are non-deterministic. Unlike traditional software where the same input always produces the same output, LLMs vary across runs, across model versions, and across subtle prompt changes. This creates three structural problems:

- **Gulf of Comprehension**: You can't manually review every user interaction to understand system behavior at scale.
- **Gulf of Specification**: Prompts may not fully capture intended instructions, causing LLMs to guess intent.
- **Gulf of Generalization**: Even well-written prompts don't guarantee reliable performance across all inputs.

Without evals, teams operate on vibes — modifying prompts, running a few test cases, and shipping if it "looks good." This fails silently as products grow.

### Why It Matters Now

Several forces are converging:

1. **Model versions change without notice** — API providers update models. Without regression tests, you can't know if a silent model update degraded your product.

2. **Agents make compounding decisions** — A single failing tool call can cascade through subsequent agent steps in ways that are nearly impossible to debug reactively.

3. **Production traffic is the first signal** — Teams without evals discover failures from user complaints, not from internal systems. This is too late and too noisy.

4. **Eval infrastructure is becoming a moat** — YC CEO Garry Tan: "Evals are emerging as the real moat for AI startups — not the latest model, not the cleverest prompts, but the ability to consistently measure and improve quality."

### What Breaks if Ignored

From documented cases and practitioner accounts:

- A team used a same-family model as judge and saw essentially perfect metrics for three months. The judge's actual agreement with domain-expert review was Cohen's kappa ~0.31 (below the "moderate agreement" floor). The real product quality had been degrading the whole time.
- The AI leasing assistant startup NurtureBoss "plateaued" when prompt engineering alone couldn't scale — systematic evaluation revealed the issue wasn't one problem but interconnected failure modes that only structured testing made visible.
- Notion could only deploy frontier models within 24 hours of release after building robust evals. Before that, model upgrades took weeks of manual testing.

---

## 3. How Practitioners Use Evals in Production

### DoorDash: The Simulation and Evaluation Flywheel

DoorDash built **AutoEval**, an LLM-powered, human-in-the-loop system for automated search result quality assessment. Results:
- 98% reduction in evaluation turnaround time
- Maintained parity with human rater accuracy

Their simulation + evaluation flywheel allows them to scale LLM chatbot testing without proportional human reviewer scaling. Source: [DoorDash engineering blog](https://careersatdoordash.com/blog/doordash-simulation-evaluation-flywheel-to-develop-llm-chatbots-at-scale/)

### Notion: AI Evaluation at Scale Across 70 Engineers

Notion built a multi-layer eval stack:
- Lightweight unit tests run frequently in CI
- More expensive offline regression tests gated behind specific triggers
- "AI Data Specialists" — a hybrid role combining QA expertise, prompt engineering, and product thinking
- Custom LLM-as-judge systems with criteria specific to each feature

They transitioned from manual JSONL files to a sophisticated workflow powered by Braintrust, achieving a **10x increase in issue resolution speed** (3 → 30 issues/day). They can now deploy frontier models within 24 hours of release.

Their philosophy: run both **regression evals** (catch breakage) and **frontier evals** (reveal where new models genuinely improve) to enable faster deployment. Source: [Braintrust: How Notion evaluates AI](https://www.braintrust.dev/blog/notion)

### Honeycomb: LLM Judge Alignment in Three Iterations

Honeycomb built a custom LLM judge for their query assistant. By working iteratively with a domain expert using Hamel Husain's "critique shadowing" method, they achieved >90% alignment between the LLM judge and the domain expert in just three iterations.

Side effect: "Seeing how the LLM breaks down its reasoning made me realize I wasn't being consistent about how I judged certain edge cases." The evaluation process itself clarified evaluation standards. Source: [Using LLM-as-a-Judge For Evaluation](https://hamel.dev/blog/posts/llm-judge/index.html)

### NurtureBoss: Simple Data Viewer as High-ROI Investment

An AI leasing assistant moved from vibe-based development by:
1. Building a simple internal data viewer showing conversations with collapsible context
2. Discovering that just three issues — date handling, handoff failures, and conversation flow — accounted for most problems
3. Creating code-based evals for deterministic date-parsing tasks
4. Building an LLM-as-judge for the subjective handoff decision

The entire shift started with a custom data viewer built in under a day. Source: [The Pragmatic Engineer: A pragmatic guide to LLM evals](https://newsletter.pragmaticengineer.com/p/evals)

### Rechat's Lucy Assistant: Systematic Testing Revealing Interconnected Failures

Rechat's real estate assistant plateaued at a certain quality level. Systematic evaluation revealed the issue wasn't one problem but interconnected failure modes — a "whack-a-mole" situation. Structured testing made these visible and addressable.

Human-LLM evaluator alignment required iterative prompt refinement tracked in spreadsheets, with agreement improving from ~60% to ~90% over several months. Source: [Hamel Husain's evals blog](https://hamel.dev/blog/posts/evals/)

### Notion / Stripe / Vercel: Braintrust-Powered Evaluation

Multiple leading AI product teams use Braintrust as their evaluation platform. Key workflow:
1. Every production trace is automatically logged
2. Any trace can be added to a dataset with one click
3. Offline evals catch regressions before deployment
4. Changes that pass eval ship to production
5. Online scoring monitors production quality
6. Low-scoring traces feed back into datasets

This creates a closed loop where production failures automatically become test cases. Source: [Braintrust](https://www.braintrust.dev/)

---

## 4. Design Patterns and Best Practices

### Pattern 1: Error Analysis First, Metrics Second

The single most important and most commonly skipped step. Before building any infrastructure:

1. Collect 100+ diverse traces from your system
2. Do "open coding" — review each trace and document observed problems descriptively without categories
3. Do "axial coding" — group observations into 5-10 themes
4. Use frequency counts to prioritize failure modes

Then build evaluators only for the failure modes you discovered, not the ones you imagined.

Hamel Husain calls this the "benevolent dictator pattern": appoint one domain expert as quality decision-maker rather than managing committee consensus. This eliminates annotation conflicts.

### Pattern 2: Binary Pass/Fail Over Scales

Reject Likert scales (1-5 ratings) as your primary evaluation signal. Binary pass/fail decisions:
- Force clarity about what "acceptable" means
- Improve consistency across reviewers
- Reduce annotation time
- Enable precise precision/recall measurement

Use sub-component checks to track granular progress instead of rolling multiple dimensions into a single score.

### Pattern 3: The Critique Shadowing Method (for LLM judges)

Hamel Husain's seven-step process for building reliable LLM judges:

1. Find the principal domain expert
2. Create a diverse dataset (across features, scenarios, personas)
3. Expert makes binary pass/fail decisions with detailed reasoning critiques
4. Fix obvious system errors before building the judge
5. Build judge iteratively — test against expert examples, refine until >90% agreement
6. Perform error analysis — classify failure patterns
7. Create specialized judges only after understanding failure modes

Key: critiques must be "detailed enough so that you can use in a few-shot prompt." Terse explanations fail to guide improvements.

### Pattern 4: Eval-Driven Development (EDD)

Treat evals like tests in Test-Driven Development:

1. Define evals for planned capabilities before implementing them
2. Red: agent fails the eval
3. Build: implement to pass
4. Green: eval passes
5. Wire into CI/CD so regressions fail the build
6. Close the loop: failing production traces flow back into the offline eval set

This makes changes explainable (accepted only when they improve metrics) and reduces regression risk.

### Pattern 5: The Swiss Cheese Defense

No single evaluation method catches every issue. Like safety engineering's Swiss Cheese Model, layer multiple methods:

| Method | Best for | Limitation |
|---|---|---|
| Automated evals | Fast iteration, regression prevention | Requires upfront investment, can give false confidence |
| Production monitoring | Real user behavior, empirical ground truth | Reactive, noisy signals |
| A/B testing | Measuring actual user outcomes | Slow (days/weeks), only tests deployed changes |
| User feedback | Unanticipated problems | Sparse, self-selected, skews toward severe issues |
| Manual transcript review | Building failure-mode intuition | Doesn't scale, inconsistent coverage |
| Human studies | Gold-standard judgments | Expensive, slow |

### Pattern 6: Agent Evaluation in Three Layers

For agents specifically (from Google Cloud and Anthropic):

- **End-to-end**: Black-box assessment — did the task succeed?
- **Trajectory-level**: Was the path efficient and sound? Tool calls, reasoning steps, retries.
- **Component-level**: Which specific retriever, tool, or sub-agent broke?

Google's "silent failure" insight: an agent can produce a correct output through an incorrect process. End-to-end pass/fail misses this. You need trajectory analysis.

### Pattern 7: Synthetic Data Bootstrap

When you have no production data yet:
1. Define dimensions that capture variation (user types, query complexity, edge cases)
2. Create 20 representative examples manually
3. Scale via LLM generation
4. Human-review a sample for quality

Synthetic data is not a substitute for production data but dramatically reduces cold-start time for eval suites.

### Decision Framework: Which Evaluator to Use?

```
Does a deterministic check exist?
├── Yes → Code-based grader (string match, regex, outcome check)
└── No → Is the judgment subjective or requires context?
    ├── Simple subjective → LLM-as-judge with binary rubric
    ├── Complex/high-stakes → Human grader (calibrates LLM judge)
    └── RAG-specific → RAGAS metrics (faithfulness, context precision)
```

### Tradeoffs and Calibration Requirements

**LLM judge calibration**: Every LLM judge requires a calibration set of 100-500 examples with human-generated ground truth labels. Run the judge against it, measure the gap, iterate the prompt until the gap closes.

**Known biases in LLM judges**:
- **Position bias**: ~40% of GPT-4 verdicts reverse when response order is swapped. Mitigation: evaluate both (A,B) and (B,A) orderings.
- **Verbosity bias**: ~15% score inflation for longer responses. Mitigation: use 1-4 scales; reward conciseness explicitly.
- **Self-enhancement bias**: 5-7% self-preference boost. Mitigation: use a different model family as judge than as generator (if your app runs on Claude, judge with GPT-4o or Gemini Pro).

### Anti-Patterns

1. **Metric sprawl**: Using 1-5 ratings across multiple dimensions creates "unactionable" data where you can't explain why something scores 3 vs 4.

2. **Outsourcing error analysis**: External annotators lack tacit domain knowledge. Breaking the feedback loop between observing failures and improving the product kills organizational learning.

3. **Infrastructure before insight**: Building elaborate eval platforms before understanding what failure modes exist. Build only what error analysis justifies.

4. **Model switching as first resort**: "Don't switch models without evidence that error analysis suggests that's actually the problem." Most improvements come from understanding failure modes, not architecture changes.

5. **Static datasets**: Eval suites without ongoing maintenance suffer from "concept drift" as user behavior evolves. Treat eval suites as living artifacts.

6. **Class-imbalanced evals**: Testing only positive cases (when behavior should occur) creates one-sided optimization. A system that always triggers will pass a positive-only eval.

7. **Rigid step checking for agents**: Checking specific step sequences penalizes agents that discover valid unanticipated solution paths. Grade outcomes, not process.

8. **Deploying without a gating eval**: The equivalent of merging uncompiled code. An architectural anti-pattern for any LLM feature.

---

## 5. Advanced Insights

### The 60-80% Time Rule

Practitioner reports consistently put evaluation at 60-80% of development time in successful AI product teams — and most of that time is spent understanding failures, not building automated checks. This sounds alarming until you realize that understanding failures is the same as building the product. Evals are debugging, not a separate line item.

### Eval Saturation: When Green Dashboards Lie

When agents pass nearly all solvable tasks, **eval saturation** occurs. Capability improvements appear as tiny score increases, potentially creating false impressions about progress or regression. Signs of saturation:
- Pass rates consistently >95% on capability evals
- Model upgrades produce <1% score differences
- Teams feel uncertain whether changes are actually improvements

Resolution: continuously add harder tasks from production failures. Difficulty must grow with agent capability.

### The Ambiguous Task Problem

A well-designed task is one where two domain experts, working independently, would reach the same pass/fail verdict. If they can't agree, the task is broken — not the agent. A 0% pass@100 on a task almost always signals a broken task specification rather than an incapable agent.

### Statistical Rigor in Evals

Anthropic's [statistical approach to model evals](https://www.anthropic.com/research/statistical-approach-to-model-evals) highlights a critical gap: most teams report eval scores without confidence intervals.

Key recommendations:
- Report Standard Error of the Mean (SEM) alongside eval scores
- Use clustered standard errors when eval sets contain related question groups — ignoring clustering can underestimate uncertainty by over **300%**
- Apply paired-difference analysis when comparing models on identical question sets (reduces variance from question difficulty)
- Run power analysis before evaluating to determine adequate sample sizes

### Reference-Free vs. Reference-Based Evaluation

Production creates a hard problem: you often can't obtain stable ground truth for open-ended outputs (writing, conversations, reasoning). Two approaches:

- **Reference-based**: Compare to a known ground truth. Works for classification, extraction, coding, math. Required for regression evals.
- **Reference-free**: Assess outputs directly against a rubric without a gold answer. Used in production monitoring, guardrails, and complex conversations.

RAGAS (for RAG systems) offers reference-free metrics like faithfulness (are claims grounded in context?) and answer relevancy. This enables production monitoring without human labeling at scale.

### Agent Evals: The New Frontier

Agent evaluation differs fundamentally from single-turn evaluation:

- **Multi-step cascades**: A single failing tool call can propagate through subsequent steps in non-obvious ways.
- **Process vs. outcome**: A correct final answer reached through a flawed trajectory is a silent failure (Google Cloud's term). You need trajectory-level analysis.
- **Non-determinism compounds**: With n agent steps, each stochastic, pass^k (all k trials succeed) degrades multiplicatively.
- **Environment isolation**: Each trial needs a clean environment. Shared state causes correlated failures or artificial performance inflation that masks real problems.

Key agent metrics beyond pass/fail:
- Tool selection accuracy (correct tool chosen for task)
- Argument correctness (correct parameters passed)
- Step efficiency (no unnecessary calls or loops)
- Plan adherence (execution followed intended workflow)

### Process Reward Models: The Emerging Frontier

Traditional evals score final outcomes. A significant emerging trend (visible in papers like AgentPRM and AgentRM) is **process reward models** — training models to evaluate the quality of intermediate reasoning steps, not just final outputs. This is especially relevant for agentic RL (reinforcement learning for agents).

From the ArXiv survey on LLM-based agent evaluation (March 2025): current benchmarks focus on end-to-end success, but there are major gaps in granular metrics that can isolate where in a multi-step trajectory things went wrong.

### Expert Disagreements in the Field

**Disagreement 1: How many examples to start with**
Hamel Husain says 20-50 is enough to start. Others argue for 100+. Resolution: both are right at different stages — 20-50 for initial direction-finding, 100+ for reliable CI gates.

**Disagreement 2: LLM judge vs. human judge primacy**
Some argue you should use humans exclusively and LLM judges only to scale confirmed good judgment. Others argue LLM judges are "less noisy than humans" because their biases are systematic (Eugene Yan's framing). The nuanced view: both are true — LLM judges are more consistent but more biased. Use humans to calibrate and audit, LLM judges to scale.

**Disagreement 3: Separate eval vs. embedded eval**
Some teams run evals as a separate pipeline from the product. Others embed evaluation into production serving (every response is scored). The trend is toward embedding, with asynchronous evaluation of production traces as the standard.

---

## 6. Curated Reading List

### Primary Sources

**1. [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) — Hamel Husain**  
Why read it: The most practical, opinionated guide to building product-specific evals. Contains the three-level hierarchy, real case studies, and the argument for why error analysis is everything.  
Difficulty: Beginner–Intermediate  
Time: 25 minutes  
Key takeaway: Start with a custom data viewer. Do error analysis before building infrastructure.

**2. [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Anthropic Engineering**  
Why read it: The definitive guide to agent-specific evaluation. Defines all terminology, covers grader types, pass@k vs pass^k, the zero-to-one roadmap, and a comparison of all evaluation methods.  
Difficulty: Intermediate  
Time: 40 minutes  
Key takeaway: Grade outcomes not process steps; eval saturation is a real failure mode; combine multiple evaluation methods.

**3. [Using LLM-as-a-Judge For Evaluation](https://hamel.dev/blog/posts/llm-judge/index.html) — Hamel Husain**  
Why read it: The complete guide to building calibrated LLM judges. Contains the critique shadowing method, the Honeycomb case study, and practical calibration strategies.  
Difficulty: Intermediate  
Time: 30 minutes  
Key takeaway: Binary pass/fail with detailed critiques beats Likert scales. The critique process itself clarifies your own evaluation standards.

**4. [LLM Evals: Everything You Need to Know](https://hamel.dev/blog/posts/evals-faq/) — Hamel Husain & Shreya Shankar**  
Why read it: FAQ format covering every practical question about evals, drawn from their Maven course. Covers CI vs production evals, synthetic data, tooling choices, and team structure.  
Difficulty: Intermediate  
Time: 1 hour (complete), or 15 minutes (selective)  
Key takeaway: Generic metrics mislead. "Error analysis is the systematic process of reviewing traces, noting problems, categorizing errors."

**5. [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan**  
Why read it: Broad survey of LLM system patterns with a strong section on evaluation. Covers G-Eval, AlpacaEval, LLM-as-judge biases, and practical implementation guidance.  
Difficulty: Intermediate  
Time: 45 minutes  
Key takeaway: "Vibe-based eval cannot be underrated" — qualitative watching of outputs during development provides signal beyond automated metrics alone.

**6. [A Methodical Approach to Agent Evaluation](https://cloud.google.com/blog/topics/developers-practitioners/a-methodical-approach-to-agent-evaluation) — Google Cloud**  
Why read it: Three-pillar framework (success/quality, process/trajectory, trust/safety) with detailed metrics per pillar. Good on "silent failures" — agents producing correct outputs through flawed processes.  
Difficulty: Intermediate  
Time: 25 minutes  
Key takeaway: An agent can produce the right output through a broken process. Trajectory evaluation is not optional.

**7. [A Pragmatic Guide to LLM Evals for Devs](https://newsletter.pragmaticengineer.com/p/evals) — The Pragmatic Engineer**  
Why read it: Accessible overview with good real-world examples (NurtureBoss case study). Strong on the three "gulfs" problem and the Analyze → Measure → Improve → Automate flywheel.  
Difficulty: Beginner  
Time: 20 minutes  
Key takeaway: Build a custom data viewer first. Most improvements come from understanding the three to five most common failure modes.

**8. [A Statistical Approach to Model Evaluations](https://www.anthropic.com/research/statistical-approach-to-model-evals) — Anthropic Research**  
Why read it: The most rigorous treatment of eval statistics. Explains why most eval scores are reported without adequate confidence intervals, and how clustering of related questions can cause 300% underestimation of uncertainty.  
Difficulty: Advanced  
Time: 30 minutes  
Key takeaway: Always report SEM with eval scores. Use paired-difference analysis when comparing models.

**9. [Building an LLM Evaluation Framework: Best Practices](https://www.datadoghq.com/blog/llm-evaluation-framework-best-practices/) — Datadog**  
Why read it: Production-focused guide covering pre-production and post-production evaluation, with monitoring, alerting, and trace-tagging strategies.  
Difficulty: Intermediate  
Time: 20 minutes  
Key takeaway: Ingest live requests into evaluation pipelines that score outputs before sending metrics to monitoring dashboards.

**10. [LLM Agent Evaluation Complete Guide](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide) — Confident AI**  
Why read it: Comprehensive treatment of agent-specific metrics — tool calling, planning, task completion, reasoning, and trace-based evals. Clear taxonomy of which metrics apply to which agent types.  
Difficulty: Intermediate  
Time: 25 minutes  
Key takeaway: Different agent types (generator, tool-calling, planning, autonomous) require different metric stacks.

### Tools and Repos

**[openai/evals](https://github.com/openai/evals)** — OpenAI's open-source eval framework and benchmark registry. Useful as reference architecture for building eval infrastructure.

**[RAGAS](https://arxiv.org/html/2309.15217v1)** — Reference-free RAG evaluation framework measuring faithfulness, answer relevancy, context precision, and context recall. De facto standard for RAG pipelines.

**[PromptFoo](https://www.promptfoo.dev/)** — CLI and library for evaluating prompts, comparing models, and red-teaming. Declarative YAML-driven, supports 50+ providers, runs completely locally. Standard tool for CI/CD-integrated prompt regression testing.

**[Braintrust](https://www.braintrust.dev/)** — End-to-end eval platform used by Notion, Stripe, Zapier, Vercel. Production → dataset → offline eval → CI → production loop in one tool. Strong for teams with 10+ engineers working on AI.

**[Langfuse](https://langfuse.com/)** — Open-source LLM observability and evaluation. Strong for teams that want self-hosted tracing + evaluation.

**[Arize Phoenix](https://arize.com/phoenix/)** — Open-source LLM tracing and evaluation built on OpenTelemetry. Strong evaluation rigor, vendor and framework agnostic.

---

## 7. Learning Path

### If You Have 30 Minutes

1. Read [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) by Hamel Husain (25 min)
2. Skim the terminology section of [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (5 min)

After 30 minutes you will understand: what evals are, why vibes-based development fails, the three grader types, and what your first step should be (build a data viewer, do error analysis).

### If You Have 2 Hours

1. [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) (25 min)
2. [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (40 min)
3. [Using LLM-as-a-Judge](https://hamel.dev/blog/posts/llm-judge/index.html) (30 min)
4. [Pragmatic Guide to LLM Evals](https://newsletter.pragmaticengineer.com/p/evals) (20 min)

After 2 hours you will understand: the complete eval system architecture, how to build calibrated LLM judges, how to run error analysis, and the NurtureBoss / Honeycomb case studies.

### If You Want to Become Highly Knowledgeable Over a Week

**Day 1**: [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) + [Pragmatic Guide to LLM Evals](https://newsletter.pragmaticengineer.com/p/evals). Build a minimal data viewer for a project you're working on.

**Day 2**: [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents). Implement your first code-based grader and first LLM-as-judge grader.

**Day 3**: [LLM Evals: Everything You Need to Know](https://hamel.dev/blog/posts/evals-faq/) (Hamel + Shreya). Focus on the "error analysis" and "CI vs production" sections.

**Day 4**: [Patterns for Building LLM Systems](https://eugeneyan.com/writing/llm-patterns/) + [Methodical Approach to Agent Evaluation](https://cloud.google.com/blog/topics/developers-practitioners/a-methodical-approach-to-agent-evaluation). Run error analysis on 100+ traces from your system.

**Day 5**: [Using LLM-as-a-Judge](https://hamel.dev/blog/posts/llm-judge/index.html). Build a calibrated judge for your most subjective evaluation dimension.

**Day 6**: [Statistical Approach to Model Evaluations](https://www.anthropic.com/research/statistical-approach-to-model-evals) + [Datadog evaluation framework](https://www.datadoghq.com/blog/llm-evaluation-framework-best-practices/). Add confidence intervals and production monitoring to your eval system.

**Day 7**: Wire your evals into CI/CD using PromptFoo or Braintrust. Set up production trace logging. Review the [LLM Agent Evaluation guide](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide) if your system uses agents.

---

## 8. Practical Application

### For Any AI Product You're Building Today

**Before writing any eval infrastructure**:
1. Build or use a trace viewer tailored to your domain — conversations with collapsible context, filterable by user, scenario, and outcome. This is the highest-ROI single investment.
2. Manually review 100 traces. Write down what you observe without categories. Group into 5-10 themes.
3. Count which failure modes are most frequent. These are your first evals.

**For a simple feature (single-turn)**:
- Write code-based graders for deterministic aspects (format, presence of required fields, absence of forbidden content)
- Write an LLM-as-judge grader for the subjective dimension (tone, helpfulness, relevance)
- Calibrate the judge by comparing to 30 hand-labeled examples
- Target >90% alignment before using the judge as a CI gate

**For a RAG pipeline**:
- Use RAGAS metrics: faithfulness (are claims grounded in retrieved context?), context precision (is retrieved context relevant?)
- Add answer relevancy as a production monitoring signal
- Monitor for retrieval failures separately from generation failures — they require different fixes

**For an agent**:
- Start with end-to-end task completion (did the task succeed?)
- Add tool selection accuracy as soon as you have tool-calling steps
- Add argument correctness for any tool call with parameters that matter
- Use pass@k early (at least one correct in k attempts), transition to pass^k as reliability requirements grow
- Ensure clean environment isolation per trial — shared state creates correlated failures

### Applying to Dalgo Specifically

Dalgo builds data pipelines and transformations for NGOs. Key eval opportunities:

**Data pipeline quality**:
- Code-based graders: does the generated dbt SQL compile? Does it produce the expected schema? Does it reference only available source columns?
- LLM-as-judge: is the transformation logic correct for the stated business intent? Is the description clear to a non-technical NGO user?

**Natural language to pipeline translation** (if you build NL interfaces):
- Does the generated pipeline match the user's stated intent?
- Does it handle edge cases (empty data, missing columns) gracefully?
- Error analysis: what categories of user requests fail most often?

**Agent behaviors** (e.g., an AI assistant helping NGO staff):
- Task completion: did the user accomplish their stated goal?
- Safety: did the agent avoid suggesting operations that could irreversibly destroy data?
- Transparency: did the agent explain what it was doing in language a non-technical user can understand?

**Eval suite structure for Dalgo**:
1. A small golden dataset (50-100 cases) of representative user requests with expected outputs, covering: typical cases, edge cases (unusual data shapes), and adversarial cases (ambiguous or impossible requests)
2. Code-based graders for all structural requirements (valid SQL, correct schema references)
3. An LLM-as-judge calibrated against one NGO domain expert for qualitative output quality
4. CI integration that blocks deploys if pass rate drops below threshold
5. Production trace logging with weekly error analysis review

**Guardrails vs. evaluators**: In Dalgo's case, guardrails (blocking destructive operations in real-time) are different from evaluators (measuring quality post-hoc). Both matter — but don't conflate them. A guardrail that blocks `DROP TABLE` in the wrong context is not an eval; an eval that measures whether the generated pipeline is logically correct is not a guardrail.

---

## 9. Sources

- [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) — Hamel Husain
- [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Anthropic Engineering
- [Using LLM-as-a-Judge For Evaluation](https://hamel.dev/blog/posts/llm-judge/index.html) — Hamel Husain
- [LLM Evals: Everything You Need to Know](https://hamel.dev/blog/posts/evals-faq/) — Hamel Husain & Shreya Shankar
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [A Methodical Approach to Agent Evaluation](https://cloud.google.com/blog/topics/developers-practitioners/a-methodical-approach-to-agent-evaluation) — Google Cloud
- [A Statistical Approach to Model Evaluations](https://www.anthropic.com/research/statistical-approach-to-model-evals) — Anthropic Research
- [A Pragmatic Guide to LLM Evals for Devs](https://newsletter.pragmaticengineer.com/p/evals) — The Pragmatic Engineer
- [Building an LLM Evaluation Framework: Best Practices](https://www.datadoghq.com/blog/llm-evaluation-framework-best-practices/) — Datadog
- [LLM Agent Evaluation Complete Guide](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide) — Confident AI
- [How Notion Evaluates AI at Scale Across 70 Engineers](https://www.braintrust.dev/blog/notion) — Braintrust
- [Define Success Criteria and Build Evaluations](https://platform.claude.com/docs/en/docs/test-and-evaluate/develop-tests) — Anthropic Docs
- [DoorDash Simulation and Evaluation Flywheel](https://careersatdoordash.com/blog/doordash-simulation-evaluation-flywheel-to-develop-llm-chatbots-at-scale/) — DoorDash Engineering
- [Survey on Evaluation of LLM-Based Agents](https://arxiv.org/html/2503.16416v2) — ArXiv
- [LLM-as-a-Judge: The Complete Guide](https://galtea.ai/blog/llm-as-a-judge-the-complete-guide) — Galtea
- [Evaluation-Driven Development and Operations of LLM Agents](https://arxiv.org/html/2411.13768v3) — ArXiv
- [PromptFoo Documentation](https://www.promptfoo.dev/docs/intro/) — PromptFoo
- [RAGAS: Automated Evaluation of RAG](https://arxiv.org/html/2309.15217v1) — ArXiv
- [Braintrust Platform](https://www.braintrust.dev/)
- [Arize Phoenix](https://arize.com/phoenix/)
