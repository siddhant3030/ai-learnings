# Tool Calling in LLMs: A Deep Research Report

*Last updated: June 2026. Primary sources: Anthropic engineering blog, OpenAI function calling docs, Berkeley Function Calling Leaderboard (BFCL), practitioner engineering blogs, research papers.*

---

## 1. Core Concepts

### What tool calling actually is

Tool calling (also called function calling or tool use) is a mechanism that lets a language model request the execution of an external function rather than generating text that describes what a function would return. The model emits a structured call — a function name plus typed arguments — your application runs the actual function, and you feed the result back to the model in the next turn.

The key distinction is execution authority: the model decides *which* tool to call and *with what arguments*, but your code actually runs the tool. The model never directly executes anything.

### The request-response loop

Every tool-use interaction follows the same agentic loop:

1. You send a message with a `tools` array describing available functions (name, description, JSON Schema for parameters).
2. The model returns either a direct response or a `tool_use` block (Anthropic) / `tool_calls` array (OpenAI) with the function name and arguments.
3. Your code executes the function.
4. You send a new message containing the `tool_result` (Anthropic) / a message with `role: "tool"` (OpenAI).
5. The model reads the result and either calls another tool or produces a final answer.

This loop can repeat many times for complex tasks. Production agents commonly run 10–20 turns; some complex workflows run 50+.

### Server tools vs client tools

Anthropic distinguishes two execution modes:
- **Client tools**: Your application runs the function. The API returns `stop_reason: "tool_use"` and you handle execution.
- **Server tools**: Anthropic runs the function on their infrastructure (web search, code execution, file reading). You see results directly without handling execution.

OpenAI's equivalent distinction is "built-in tools" (code interpreter, file search) vs "function calling" (you handle execution).

### Key terminology

| Term | Meaning |
|------|---------|
| Tool / Function | A capability you expose to the model |
| Tool call | The structured request the model emits |
| Tool result | The output you return after execution |
| Agentic loop | The multi-turn cycle of call → execute → respond |
| `tool_choice` | Parameter controlling whether/how the model calls tools |
| Parallel tool calls | Multiple tools called in a single model turn |
| Strict mode | Schema enforcement via constrained decoding |

### Common misconceptions

**Misconception 1: More tools = more capability.**
Accuracy degrades significantly as the tool catalog grows. At 50 tools, frontier models maintain 84–95% selection accuracy. At 200 tools, this drops to 41–83%. Naive "give the model everything" approaches fail at scale.

**Misconception 2: The model executes the tools.**
The model never runs code. It generates a *request* for your code to execute. The security boundary lives entirely in your execution layer.

**Misconception 3: Structured output (JSON mode) guarantees correctness.**
JSON mode guarantees *valid JSON syntax*. Strict schema mode (constrained decoding) guarantees *syntactic schema conformance* with below 0.1% failure rate — but neither prevents semantically wrong values. A confidence field constrained to be a float between 0 and 1 can always output `0.99` regardless of actual confidence.

**Misconception 4: Tool calling is just prompt engineering.**
Tool calling has a distinct token cost. Each model adds 264–804 system prompt tokens automatically when tools are enabled (varies by model and `tool_choice` setting). With 50 tools, schema definitions can consume 25,000–75,000 tokens before any user message is processed.

**Misconception 5: Reasoning models parallelize tool calls.**
As of 2025, OpenAI's reasoning models (o1, o3) do not call functions in parallel — they default to sequential execution. Standard models (GPT-4o, Claude) do support parallel tool calls.

---

## 2. Why It Matters

### The problem tool calling solves

Language models have a static knowledge cutoff and no ability to take actions in the world. Tool calling bridges the gap by granting models structured access to:

- Live data (APIs, databases, search)
- Computation (code execution, calculators)
- Stateful operations (CRM writes, email sends, file mutations)
- Other agents (sub-agents wrapped as tools)

Without tool calling, agents are limited to generating text that *describes* what should happen. With tool calling, they can make things happen.

### Why it's become critical now (2025–2026)

Three forces converged:

1. **Standardization**: Anthropic open-sourced the Model Context Protocol (MCP) in November 2024. By April 2025, OpenAI and Google both adopted it. MCP grew from 100,000 SDK downloads (November 2024) to 97 million monthly downloads by March 2026. Governance moved under the Linux Foundation's Agentic AI Foundation in December 2025. Building on proprietary tool protocols in 2026 is now a strategic liability.

2. **Evaluation maturity**: BFCL V4 (July 2025) shifted from measuring single-call accuracy to evaluating full agentic behavior: multi-turn memory, when-not-to-call judgment, and long-horizon reasoning. This gave teams reliable benchmarks to guide model selection and system design.

3. **Security pressure**: Prompt injection ranked #1 on OWASP's 2025 Top 10 for LLM Applications. Tool abuse is the primary attack surface. The discipline of securing tool execution has forced teams to build execution layers as proper authorization boundaries, not convenience wrappers.

### What breaks if ignored

- **Context collapse**: In a three-server MCP setup (GitHub, Slack, Sentry), a production deployment consumed 143,000 of 200,000 context tokens just for tool schemas — 72% of the budget before processing a single user query.
- **Cost explosion**: A single user request in an agentic workflow consumes 5–10x the tokens of a direct chat completion. Teams without cost instrumentation routinely discover their agent costs are 50x higher than expected.
- **Silent failures**: Tool hallucination (calling nonexistent tools or fabricating parameters) causes tangible damage in database operations, robotic control, and financial workflows. Unlike text hallucinations, tool hallucinations affect real systems.
- **Security incidents**: Notable 2025 CVEs include GitHub Copilot RCE (CVE-2025-53773), Microsoft EchoLeak (CVE-2025-32711, CVSS 9.3), and a Cursor path filtering bypass (CVE-2025-59944) — all exploited via indirect prompt injection through tool results.

---

## 3. How Practitioners Use It

### Pinterest's production MCP ecosystem

Pinterest is the clearest published example of production-scale tool calling. Published in April 2026, their architecture runs domain-specific MCP servers for Presto (data querying), Spark (debugging), and Airflow (workflow management) behind a central registry.

**Scale**: 66,000 tool invocations per month across 844 monthly active users. Estimated 7,000 engineering hours saved per month.

**Key architectural decisions**:
- Cloud-hosted multi-server architecture (not local processes) — each domain server owns a small, coherent set of tools
- Central MCP registry as source of truth with both a web UI (human discovery) and API (agent validation)
- Two-layer authorization: OAuth JWTs for human-in-the-loop flows, SPIFFE mesh identities for low-risk read-only automation
- Decorator-based fine-grained authorization within each server (e.g., `@authorize_tool(policy='...')`)
- Business-group gating: the Presto server appears company-wide, but only returns results for Ads, Finance, or infrastructure teams
- Human-in-the-loop confirmation for sensitive write operations

**Lessons**: Deployment burden matters more than protocol design. Simplified infrastructure reduced friction. Context is real — multiple specialized servers prevent context waste versus a monolithic tool collection. Security from day one enabled flexible authorization without retrofitting.

Source: [Pinterest Engineering Blog](https://medium.com/pinterest-engineering/building-an-mcp-ecosystem-at-pinterest-d881eb4c16f1)

### JetBrains: Debugging tool-calling addiction in Koog

JetBrains published a post-mortem on a production bug where their Koog AI framework developed behavioral loops in the tool-calling layer. After ~100 consecutive tool-call-result exchanges, the model became trapped: even when explicitly asked to summarize, it responded with a `write_file()` call containing the summary — still in tool-calling format.

**Root cause**: The implicit few-shot learning effect of the message structure (100 repeated tool-call patterns) became more powerful than explicit textual instructions. The model learned that "this conversation continues with tool calls."

**Fix**: Structural changes over instruction changes. Wrapping conversation history in XML tags broke the pattern's grip. Combining all messages into a single user message with custom tags disrupted the special token patterns (`<|im_start|>`, `<|im_end|>`) that were reinforcing the loop.

**Production lesson**: Repetitive structures create behavioral ruts that override semantic instructions. When an agent appears stuck, change message structure before adding more instructions.

Source: [JetBrains AI Blog](https://blog.jetbrains.com/ai/2025/07/when-tool-calling-becomes-an-addiction-debugging-llm-patterns-in-koog/)

### Anthropic's internal tooling methodology

Anthropic published their workflow for building production tools using Claude Code itself as the builder and evaluator. Their iterative process:

1. Build quick prototypes using Claude Code with official documentation
2. Generate realistic evaluation tasks requiring multiple tool calls
3. Run programmatic evaluations using simple agentic loops
4. Analyze agent reasoning, tool call patterns, and errors — what agents *omit* is often more revealing than what they include
5. Use Claude Code to automatically optimize tool implementations against evaluations
6. Validate on held-out test sets

**Quantitative finding**: Even after "expert" human-written tool implementations, running held-out test sets revealed additional improvements. Claude Code analyzing transcripts and refactoring tool implementations consistently outperformed human-only iterations.

Source: [Anthropic: Writing Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents)

### LLMCompiler: The parallel tool calling paper

Stanford's SqueezeAI Lab published LLMCompiler at ICML 2024. It treats tool calling like compiler optimization: generate a DAG of tool calls with explicit dependencies, then dispatch them to a task fetching unit that identifies topologically ready tasks and runs them in parallel.

**Results vs ReAct (sequential)**:
- 3.7x latency speedup
- 6.7x cost reduction
- ~9% accuracy improvement (less context pollution from sequential intermediate results)

This paper is the empirical foundation for the current industry consensus that parallel tool execution is a latency lever, not just a nice-to-have.

Source: [LLMCompiler Paper](https://arxiv.org/abs/2312.04511) | [GitHub](https://github.com/SqueezeAILab/LLMCompiler)

### ProjectDiscovery: 59% cost reduction via prompt caching

ProjectDiscovery published a case study on using Anthropic's prompt caching for their security scanning agent. By caching tool definitions and system prompts across the agentic loop, they cut LLM costs by 59% compared to full input-token pricing. Cached reads cost $0.30/M tokens vs $3.00/M for fresh inputs.

Source: [ProjectDiscovery Blog](https://projectdiscovery.io/blog/how-we-cut-llm-cost-with-prompt-caching)

---

## 4. Design Patterns and Best Practices

### Pattern 1: Tool consolidation over proliferation

Do not wrap every API endpoint as a separate tool. Instead, build tools that bundle multiple discrete operations at natural task boundaries. Rather than `list_users`, `list_events`, `create_event` as three separate tools, build a unified `schedule_event` that handles availability checks and scheduling internally.

**Why**: Fewer tools reduces selection confusion. Bundled operations eliminate round-trips. The agent's context sees less noise.

### Pattern 2: Rich, explicit tool descriptions

Tool descriptions are not documentation for human readers — they are the model's primary signal for tool selection. Treat them like system prompts. Include:

- What the tool does and when to use it
- When NOT to use it (explicit negative examples)
- Parameter formats with concrete examples
- Cross-tool dependencies (e.g., "use `get_user_id` before calling this")
- Common errors and what they mean
- Edge cases

**Quantitative evidence**: Rewriting tool descriptions (using the Trace-Free+ method) reduced accuracy degradation at 150+ tool scale by 29.23% and improved query-level success by 60.89%. Adding three concrete usage examples to a tool definition improved complex parameter handling accuracy from 72% to 90% in Anthropic's advanced tool use testing.

### Pattern 3: Hierarchical tool routing at scale

Beyond ~50 tools, load all definitions upfront only for the tools the agent will almost certainly need. Use embedding-based semantic search or BM25 to surface relevant tools on demand for the rest. This is now formalized in Anthropic's Tool Search Tool and OpenAI's tool search feature.

**Numbers**: A five-server MCP setup consuming 55,000 tokens upfront drops to ~8,700 tokens with selective loading — an 85% reduction. Search accuracy jumps from 49% to 74% (Opus 4) and 79.5% to 88.1% (Opus 4.5) when tools are discovered on-demand rather than all loaded upfront.

### Pattern 4: Parallel execution via dependency analysis

Model your tool calls as a DAG. Independent calls (no data dependency between them) should execute concurrently. A four-tool sequence with 300ms calls each takes 1,200ms sequentially vs 300ms in parallel.

Practical performance table from research:
| Metric | Sequential | Parallel | Gain |
|--------|-----------|----------|------|
| Latency | 30+ sec | 6 sec | ~5x |
| Tokens | baseline | ~50% fewer | 2x |
| Cost | baseline | ~40% lower | 1.7x |
| Turns to completion | baseline | ~60% fewer | 2.5x |

**Descending strategy**: Prioritize broad exploration in early stages, then focused exploitation. This outperforms static or ascending strategies by ~6% on agentic benchmarks.

**Optimal parallelism**: Research on BrowseComp, HLE, and GAIA found that 3 parallel tools per turn provides the best cost-latency tradeoff. Beyond this, reasoning overhead and context flooding start to dominate.

### Pattern 5: Strict schema mode everywhere in production

Use `strict: true` (Anthropic) or `"strict": true` in function definitions (OpenAI) for all production tool calls. Constrained decoding achieves below 0.1% syntactic schema violation rate with less than 3% latency overhead.

Requirements for strict mode: `additionalProperties: false`, all `properties` fields marked `required`, optional fields use `type: ["string", "null"]` syntax.

**Important caveat**: Strict mode prevents schema violations. It does not prevent semantically wrong values. A confidence field can be schema-valid and semantically meaningless. Semantic validation requires LLM-as-judge or domain-specific checks on top.

### Pattern 6: The "think" tool for complex reasoning

For long chains of tool calls or complex constraint environments, add a `think` tool — a function that takes a `thought` string parameter and returns nothing. The model can pause and reason explicitly about tool results before deciding the next action.

**Results on τ-Bench**: 54% relative improvement in airline domain (0.570 vs 0.370 pass rate), 0.812 vs 0.783 in retail domain. Largest gains on policy compliance tasks and multi-step reasoning chains.

**When to skip**: Single tool calls or parallel calls completing a task in one turn. The "think" tool adds latency when unnecessary.

Source: [Anthropic: The Think Tool](https://www.anthropic.com/engineering/claude-think-tool)

### Pattern 7: Prompt caching for agentic loops

Cache the stable prefix of your prompt (system instructions + tool definitions) separately from dynamic content (conversation history + tool results). Cache reads cost 10–25% of fresh input tokens across providers.

**Critical finding from research**: "Naively enabling full-context caching can paradoxically increase latency" because dynamic tool results trigger cache writes for content that won't be reused across sessions. Cache only system prompts and static tool definitions. Mark dynamic content (timestamps, tool results) as cache-breaking.

**Enterprise math**: A 100-person team running agents 5,000 times per day saves $2,000+/day in system-prompt caching alone.

### Pattern 8: Structured error handling taxonomy

Four failure types require distinct responses:
- **Transient failures** (429 rate limits, brief 5xx): Exponential backoff with jitter
- **Persistent failures** (outages, fundamental mismatches): Escalation or graceful fallback
- **Validation failures** (schema violations, malformed args): Feed error back to model — one retry resolves most format errors
- **Semantic failures** (correct structure, wrong logic): Require LLM-as-judge or human escalation

Implement circuit breakers for consistent failures. Use per-tool timeouts independent of overall request timeouts. Persist stage outputs immediately so pipelines can resume from checkpoints rather than restarting.

### Design decision framework: Tool calling vs RAG

Use RAG when the problem is "answer questions using documents." Use tool calling when the problem is "execute operations based on user intent." The decision boundary:

| Use RAG | Use Tool Calling |
|---------|-----------------|
| Retrieval happens before model reasoning | Model actively decides when to retrieve |
| Document similarity drives what the model sees | Model can chain multiple operations |
| Read-only, knowledge retrieval | Read-write, actions with side effects |
| Response quality correlates with retrieval quality | Response quality correlates with tool design |

The most sophisticated 2025 systems combine both: Agentic RAG embeds retrieval decisions into the model's reasoning flow via tools, letting the model decide when to retrieve and from which source.

---

## 5. Advanced Insights

### Models don't know when to call tools accurately

Research published May 2025 ("To Call or Not to Call") found systematic misalignment between models' self-perceived need for tools and actual need. Key findings:
- When a model's performance was already high without tools, incorporating them led to negative utility in 34% of cases (the tool call degraded performance)
- Models consistently violated budget constraints, calling more tools than permitted
- Lightweight ML classifiers trained on hidden states improved decision quality *without fine-tuning the base model*

Implication: do not rely solely on `tool_choice: auto` for cost-sensitive applications. Consider lightweight classifiers or explicit budget instructions in system prompts.

Source: [arXiv:2605.00737](https://arxiv.org/html/2605.00737v1)

### Tool hallucination is systematic, not random

BFCL V4 analysis shows hallucination patterns vary significantly by model: Gemini-2.0-Pro shows 0% hallucination but 85.64% tool omission (it never hallucinates but often refuses to call tools), while GPT-4o balances these at 47.16% hallucination / 49.72% omission. These are fundamentally different failure modes requiring different mitigations.

Tool omission and parameter errors are the primary obstacles for reliable production tool calling in 2025 — more so than outright hallucination.

### Reasoning enhancement can amplify tool hallucination

A 2025 paper ("The Reasoning Trap") found that chain-of-thought and extended reasoning can *increase* tool hallucination on some tasks. The model generates elaborate justifications for incorrect tool selections. Extended thinking helps complex tasks but can lock in wrong approaches through over-confident reasoning chains.

Mitigation: Interleaved thinking (where the model can revise reasoning after each tool result) is more reliable than pre-call extended thinking for multi-step tool workflows.

### The "bag of agents" error multiplier

A 2025 analysis showed that multi-agent systems have an exponential error accumulation problem. If each step has 90% reliability, a 10-step pipeline has 35% end-to-end reliability. The "bag of agents" anti-pattern — running many agents in parallel and hoping they agree — multiplies rather than averages errors.

Production agents typically execute at most 10 steps before requiring human intervention (from a 306-practitioner survey). Design for human checkpoints, not full autonomy.

### Context position matters for tool reasoning

Field ordering within structured outputs affects reasoning quality. Placing answers before reasoning in JSON schemas causes 10–15% performance degradation on complex tasks because chain-of-thought quality degrades when the model commits to an answer before reasoning about it. Always put reasoning fields before conclusion fields.

### MCP context bloat is now the critical scaling problem

At enterprise scale, loading tool schemas from 5–10 MCP servers consumes 100,000–200,000 tokens per request. GitHub's MCP server alone contributes ~55,000 tokens in initialization overhead. A developer with 10 MCP servers wastes ~$1,370 annually in schema-related token costs just from the overhead.

Solutions GA or near-GA as of 2026: Lazy schema loading (96% reduction), Tool Search / dynamic discovery (85% reduction), server decomposition (Pinterest-style domain boundaries, ~80% reduction).

### Expert disagreements worth knowing

**Direct API vs framework**: Anthropic's official recommendation for simple linear workflows is "use the Claude API directly with minimal framework overhead." Framework engineering overhead can exceed its benefit. Counter-argument: LangGraph's stateful graph execution with Temporal for durable execution is hard to replicate correctly from scratch in production.

**Parallel vs sequential**: Parallel tool execution is faster but creates "context flooding" where simultaneous results exceed context limits. At 3 parallel tools per turn the research finds optimal balance — but this varies by task type and tool output verbosity.

**Security through architecture vs prompts**: OpenAI acknowledges prompt injection "may never be fully solved." Multiple 2025 CVEs in coding assistants support this. The emerging consensus is that security must be enforced at the tool execution layer (least privilege, authorization decorators, input sandboxing) — not through system prompt instructions telling the model to "be careful."

---

## 6. Curated Reading List

### Primary sources — required reading

**[Anthropic: Writing Effective Tools for AI Agents](https://www.anthropic.com/engineering/writing-tools-for-agents)**
*Why*: Anthropic's own production methodology for tool design, including their iterative eval process. Written by people who built Claude's tool use capability.
*Difficulty*: Intermediate
*Time*: 20 min
*Key takeaway*: Consolidate tools to natural task boundaries; measure what agents omit, not just what they produce; use Claude itself to optimize tool descriptions against evaluations.

**[Anthropic: Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use)**
*Why*: Introduces Tool Search Tool, Programmatic Tool Calling, and Tool Use Examples with concrete numbers. Primary source on the 85% token reduction from dynamic tool discovery.
*Difficulty*: Intermediate
*Time*: 25 min
*Key takeaway*: Tool Search reduces context consumption by 85%; programmatic tool calling reduces tokens 37% by processing intermediate data outside the model's context window; three examples per tool improve accuracy from 72% to 90%.

**[Anthropic: The Think Tool](https://www.anthropic.com/engineering/claude-think-tool)**
*Why*: Explains the "think" tool pattern with concrete performance numbers. Short and dense.
*Difficulty*: Beginner
*Time*: 10 min
*Key takeaway*: A `think` tool that returns nothing but forces explicit reasoning yields 54% relative improvement on complex multi-step tasks.

**[Berkeley Function Calling Leaderboard V4](https://gorilla.cs.berkeley.edu/leaderboard.html)**
*Why*: The authoritative benchmark for function calling. Understand what V4's agentic evaluation tests before choosing a model for production.
*Difficulty*: Beginner
*Time*: 15 min
*Key takeaway*: BFCL scores diverge significantly from MCPMark scores — test your actual workflow, not a benchmark that doesn't match it.

**[Pinterest Engineering: Building an MCP Ecosystem](https://medium.com/pinterest-engineering/building-an-mcp-ecosystem-at-pinterest-d881eb4c16f1)**
*Why*: The most detailed published case study of production-scale MCP deployment with real metrics.
*Difficulty*: Intermediate
*Time*: 25 min
*Key takeaway*: Domain-specific servers + central registry + two-layer auth is the pattern. Security from inception, not retrofit. Measure impact with telemetry from day one.

### Papers

**[LLMCompiler: An LLM Compiler for Parallel Function Calling](https://arxiv.org/abs/2312.04511)** (ICML 2024)
*Why*: Establishes the DAG-based parallel execution model with rigorous benchmarks. The theoretical foundation for production parallel tool calling.
*Difficulty*: Advanced
*Time*: 45 min
*Key takeaway*: 3.7x latency, 6.7x cost, 9% accuracy improvement over sequential ReAct. DAG planning pays off at scale.

**[Learning to Rewrite Tool Descriptions for Reliable LLM-Agent Tool Use](https://arxiv.org/html/2602.20426)**
*Why*: Quantifies exactly how much tool description quality matters. The Trace-Free+ method can be applied to any tool catalog.
*Difficulty*: Advanced
*Time*: 30 min
*Key takeaway*: 60.89% improvement in query-level success from description rewrites alone. Good descriptions are the highest-ROI optimization in tool calling.

**[To Call or Not to Call: A Framework to Assess and Optimize LLM Tool Calling](https://arxiv.org/html/2605.00737v1)**
*Why*: Quantifies the when-not-to-call problem with real data. 34% of tool calls degrade performance when the model already knew the answer.
*Difficulty*: Advanced
*Time*: 30 min
*Key takeaway*: Models consistently miscalibrate their own need for tools. Lightweight hidden-state classifiers can fix this without fine-tuning.

**[Reducing Tool Hallucination via Reliability Alignment](https://arxiv.org/pdf/2412.04141)**
*Why*: Systematic taxonomy of tool hallucination types (selection vs usage) with the reliability alignment approach.
*Difficulty*: Advanced
*Time*: 40 min
*Key takeaway*: Hallucination and omission are separate failure modes requiring different fixes. Model self-perceived reliability diverges from actual reliability.

### Engineering blogs

**[JetBrains: When Tool-Calling Becomes an Addiction](https://blog.jetbrains.com/ai/2025/07/when-tool-calling-becomes-an-addiction-debugging-llm-patterns-in-koog/)**
*Why*: A concrete production bug, root-caused and fixed. Rare honest post-mortem about behavioral patterns in real systems.
*Difficulty*: Intermediate
*Time*: 20 min
*Key takeaway*: Long tool-use sequences create behavioral lock-in. Fix with structural changes (message format), not instruction additions.

**[Klavis: Function Calling and Agentic AI — What Benchmarks Tell Us](https://www.klavis.ai/blog/function-calling-and-agentic-ai-in-2025-what-the-latest-benchmarks-tell-us-about-model-performance)**
*Why*: Honest comparison of BFCL vs MCPMark, with cost-per-successful-task calculations across providers. Practical model selection guide.
*Difficulty*: Beginner-Intermediate
*Time*: 20 min
*Key takeaway*: True cost = per-token rate × retries needed. GPT-5 at $127/run outcompetes Claude Opus at $1,165/run on complex multi-step workflows even with better raw performance.

**[Zylos Research: Tool-Augmented LLM Agents — Production Architecture Patterns](https://zylos.ai/research/2026-04-16-tool-augmented-llm-agents-production-architecture/)**
*Why*: The most thorough production architecture reference. Covers tool registry, routing, error handling, caching, and observability as a cohesive system.
*Difficulty*: Advanced
*Time*: 35 min
*Key takeaway*: "Tool calling is not a feature — it is a discipline. The models are the easy part. The architecture around them determines whether the system survives contact with production."

**[ProjectDiscovery: 59% Cost Reduction with Prompt Caching](https://projectdiscovery.io/blog/how-we-cut-llm-cost-with-prompt-caching)**
*Why*: Concrete numbers from a production system. Validates the prompt caching strategy for agentic loops.
*Difficulty*: Beginner
*Time*: 15 min
*Key takeaway*: Cache tool definitions and system prompts separately from conversation history. The split matters.

---

## 7. Learning Path

### If you have 30 minutes

1. Read [Anthropic's Think Tool post](https://www.anthropic.com/engineering/claude-think-tool) (10 min) — builds the mental model of the agentic loop
2. Skim [Writing Effective Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) (20 min) — the core design principles

You will leave knowing: how the tool-call loop works, why tool descriptions are your highest-ROI optimization, and what the "think" pattern achieves.

### If you have 2 hours

1. Read [Writing Effective Tools for Agents](https://www.anthropic.com/engineering/writing-tools-for-agents) fully (20 min)
2. Read [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) (25 min)
3. Read [Pinterest Engineering MCP post](https://medium.com/pinterest-engineering/building-an-mcp-ecosystem-at-pinterest-d881eb4c16f1) (25 min)
4. Read [JetBrains tool-calling addiction post](https://blog.jetbrains.com/ai/2025/07/when-tool-calling-becomes-an-addiction-debugging-llm-patterns-in-koog/) (20 min)
5. Skim [Klavis benchmarks post](https://www.klavis.ai/blog/function-calling-and-agentic-ai-in-2025-what-the-latest-benchmarks-tell-us-about-model-performance) (20 min)
6. Skim [Zylos production architecture](https://zylos.ai/research/2026-04-16-tool-augmented-llm-agents-production-architecture/) focusing on error handling and observability sections (30 min)

You will leave knowing: the full system design, what production problems look like, how to evaluate model choices for your use case, and the architecture layers you need to build.

### To become highly knowledgeable over a week

**Day 1**: Read all three Anthropic engineering posts (Writing Tools, Advanced Tool Use, Think Tool). Set up the Anthropic API and build a minimal multi-tool agent from scratch — do not use a framework.

**Day 2**: Read LLMCompiler paper (abstract + experiments sections). Implement parallel tool dispatch in your agent using Python asyncio. Measure latency improvement.

**Day 3**: Read the Pinterest MCP post. Study the MCP specification at [modelcontextprotocol.io](https://modelcontextprotocol.io). Build or connect to one MCP server. Understand lazy schema loading.

**Day 4**: Read "Learning to Rewrite Tool Descriptions" paper. Audit your existing tool descriptions using the five-pattern checklist (scope, cross-dependencies, output documentation, parameter constraints, cross-parameter dependencies). Run before/after evals.

**Day 5**: Read "To Call or Not to Call" and "Reducing Tool Hallucination via Reliability Alignment." Build an evaluation harness tracking: tool correctness, argument correctness, step efficiency, and cost per successful task.

**Day 6**: Read the Zylos production architecture post fully. Implement circuit breakers, per-error-code retry policies, and OpenTelemetry tracing in your agent.

**Day 7**: Read the JetBrains post and the Lakera indirect prompt injection post. Red-team your own tool implementations with injected tool results. Implement input validation and authorization at the execution layer.

---

## 8. Practical Application

### Applying this to Dalgo's MCP server

Dalgo already runs a Django Ninja backend that exposes API endpoints. The `django-ninja-mcp` library ([PyPI](https://pypi.org/project/django-ninja-mcp/)) can auto-generate MCP tool definitions from your existing OpenAPI schema. However, auto-generation produces tool descriptions that are too thin for reliable model selection at scale — apply the five-pattern description template to every tool.

**Immediate actions for Dalgo**:

1. **Domain-specific servers**: Follow Pinterest's model. Rather than one monolithic MCP server, expose separate servers for: warehouse (browse/query), pipelines (run/monitor), transforms (dbt), dashboards (visualize). Each server owns a coherent subset of tools with separate auth contexts.

2. **Authorization decorators**: For write operations (trigger pipeline, publish changes, delete source), implement explicit human-in-the-loop confirmation before execution. Read operations can run with service-level auth.

3. **Context budget discipline**: A well-documented Dalgo tool with parameter descriptions costs 500–1,500 tokens. With 40 Dalgo tools, that's 20,000–60,000 tokens before a user query processes. Mark infrequently-needed tools with `defer_loading: true` immediately.

4. **Prompt caching**: Cache the tool definitions and system prompt prefix. Dalgo's agentic loop re-sends the same tool schemas on every turn — this is pure waste at current token prices.

### For any AI product with tool calling

**Start before you code**: Write the tool description first, as if explaining to a new team member. Test it with a real agent before implementing the tool. Bad descriptions are the leading cause of tool selection failures.

**Build the evaluation harness before the agent**: Measure tool correctness, argument correctness, and step efficiency from the start. Without evals, you cannot detect regressions when descriptions, models, or schemas change.

**Treat the execution layer as a security boundary**: No prompt instruction reliably prevents tool abuse. Build authorization at the execution layer. Least privilege per tool. Explicit confirmation for writes. Input validation before execution.

**Make parallel explicit**: Don't rely on the model to notice parallelism opportunities. Structure your tool definitions so independent tools have no shared output parameters. Use `parallel_tool_calls: false` only when you need guaranteed sequential execution for state reasons.

**Instrument everything**: Per-tool call latency (P50/P95/P99), success rates by error type, redundant call detection (signals memory issues), schema violation rates (signals description quality problems), and cost per tool call. These four metrics tell you where to invest optimization effort.

**The agent-as-tool pattern for Dalgo's architecture**: Wrap specialist agents (warehouse agent, pipeline agent, transform agent) as tools callable by a coordinator agent. This maps naturally to Dalgo's domain structure, enables independent testing, and keeps tool counts per agent under the 50-tool accuracy cliff.

---

## 9. Sources

Primary sources consulted in the preparation of this report:

- [Anthropic: Writing Effective Tools for AI Agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Anthropic: Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use)
- [Anthropic: The Think Tool](https://www.anthropic.com/engineering/claude-think-tool)
- [Anthropic: Tool Use Overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [OpenAI: Function Calling Guide](https://developers.openai.com/api/docs/guides/function-calling)
- [Berkeley Function Calling Leaderboard V4](https://gorilla.cs.berkeley.edu/leaderboard.html)
- [BFCL Paper (ICML 2025)](https://proceedings.mlr.press/v267/patil25a.html)
- [Pinterest Engineering: Building an MCP Ecosystem](https://medium.com/pinterest-engineering/building-an-mcp-ecosystem-at-pinterest-d881eb4c16f1)
- [JetBrains: When Tool-Calling Becomes an Addiction](https://blog.jetbrains.com/ai/2025/07/when-tool-calling-becomes-an-addiction-debugging-llm-patterns-in-koog/)
- [LLMCompiler: An LLM Compiler for Parallel Function Calling](https://arxiv.org/abs/2312.04511) (ICML 2024)
- [Learning to Rewrite Tool Descriptions (arXiv:2602.20426)](https://arxiv.org/html/2602.20426)
- [To Call or Not to Call (arXiv:2605.00737)](https://arxiv.org/html/2605.00737v1)
- [Reducing Tool Hallucination via Reliability Alignment (arXiv:2412.04141)](https://arxiv.org/pdf/2412.04141)
- [The Reasoning Trap (arXiv:2510.22977)](https://arxiv.org/pdf/2510.22977)
- [Klavis: Function Calling and Agentic AI Benchmarks](https://www.klavis.ai/blog/function-calling-and-agentic-ai-in-2025-what-the-latest-benchmarks-tell-us-about-model-performance)
- [Zylos Research: Tool-Augmented LLM Agents Production Architecture](https://zylos.ai/research/2026-04-16-tool-augmented-llm-agents-production-architecture/)
- [Zylos Research: Tool Use Standards and Benchmarks](https://zylos.ai/research/2026-04-07-tool-use-function-calling-standards-benchmarks)
- [Zylos Research: Parallel Tool Calling Optimization](https://zylos.ai/research/2026-04-23-parallel-tool-calling-optimization-ai-agents)
- [AgentMarketCap: MCP Context Bloat Crisis](https://agentmarketcap.ai/blog/2026/04/08/mcp-context-bloat-enterprise-scale-tool-definitions-agent-context-budget)
- [ProjectDiscovery: 59% Cost Reduction via Prompt Caching](https://projectdiscovery.io/blog/how-we-cut-llm-cost-with-prompt-caching)
- [Don't Break the Cache (arXiv:2601.06007)](https://arxiv.org/html/2601.06007v2)
- [MCP Adoption Statistics 2026](https://www.digitalapplied.com/blog/mcp-adoption-statistics-2026-model-context-protocol)
- [MCP Wikipedia](https://en.wikipedia.org/wiki/Model_Context_Protocol)
- [Structured Output Reliability in Production](https://tianpan.co/blog/2026-04-20-structured-output-reliability-production)
- [Confident AI: LLM Agent Evaluation Complete Guide](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide)
- [DEV.to: Why LLM Agents Break When You Give Them Tools](https://dev.to/terzioglub/why-llm-agents-break-when-you-give-them-tools-and-what-to-do-about-it-f5)
- [Lakera: Indirect Prompt Injection](https://www.lakera.ai/blog/indirect-prompt-injection)
- [django-ninja-mcp on PyPI](https://pypi.org/project/django-ninja-mcp/)
