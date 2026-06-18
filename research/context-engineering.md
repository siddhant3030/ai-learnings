# Context Engineering: A Deep Research Report

> Researched June 2026. Covers the state of context engineering as a production discipline — from foundational concepts to advanced production patterns.

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

### The Fundamental Shift

In June 2025, Andrej Karpathy wrote on X: "+1 for 'context engineering' over 'prompt engineering'. People associate prompts with short task descriptions you'd give an LLM in your day-to-day use. When in every industrial-strength LLM app, context engineering is the delicate art and science of filling the context window with just the right information for the next step."

Shopify CEO Tobi Lütke coined a clean definition around the same time: context engineering is "the art of providing all the context for the task to be plausibly solvable by the LLM."

These two quotes together capture the whole field. Context engineering is not about crafting better sentences. It is about engineering the information environment that the model reasons inside.

### What "Context" Actually Means

The context window is everything the model can see during a single inference call. In a production agent, that includes:

- The system prompt (instructions, persona, constraints)
- Tool definitions (each one costs 300–1,400 tokens before any conversation begins)
- Conversation history (every prior turn, tool call, and tool result)
- Retrieved documents (from RAG pipelines, knowledge graphs, vector stores)
- Structured data injected at runtime (database records, API responses)
- Agent working memory (notes, state, intermediate summaries)
- The current user message

Context engineering is the discipline of deciding what goes in, in what order, at what time, and at what level of compression.

### The Distinction from Prompt Engineering

| Dimension | Prompt Engineering | Context Engineering |
|---|---|---|
| Scope | A single instruction string | The entire information architecture |
| Mindset | "What should I tell the model?" | "What should the model know?" |
| Time horizon | One interaction | A multi-step session or long-running agent |
| Medium | Text strings | Systems, pipelines, memory stores |
| Encoding time | Write-time (static) | Runtime (dynamic) |
| Best for | One-off demos, chat | Production agents, workflows |

Prompt engineering is a subset of context engineering. You still need good prompts. But in production systems, the prompt is one small component of a much larger context assembly system.

### The Mental Models You Need

**Context window as RAM.** The LLM is the CPU. The context window is working memory (RAM). External stores — vector databases, file systems, conversation archives — are disk. Your job is to be the operating system: decide what deserves to be in RAM right now.

**Context as a scarce, depreciating resource.** Every token competes for the model's attention. More tokens does not mean more capability. Past a threshold, adding tokens actively hurts. The LLM's attention budget is finite; you are bidding for it with every piece of information you inject.

**Context as the bottleneck, not the model.** Philipp Schmid, Senior AI Relations Engineer at Google DeepMind: "Most agent failures are not model failures anymore, they are context failures." The model is smart enough. The question is whether it has the right information at the right moment.

**Just-in-time vs. just-in-case.** A beginner loads all relevant data upfront (just-in-case). A production system loads data when the agent needs it (just-in-time), using lightweight identifiers (file paths, query strings, URLs) and tool calls to fetch on demand.

### Common Misconceptions

**Misconception: Bigger context windows solve context problems.** They do not. A 2025 Chroma study tested 18 frontier models including GPT-4.1, Claude, and Gemini. Every single one performed worse as input length grew. Accuracy typically degrades noticeably around 32,000 tokens — well before the million-token limits being marketed. The "lost in the middle" problem (see Section 2) makes large context windows expensive and unreliable if managed naively.

**Misconception: Context engineering is just RAG.** RAG (Retrieval-Augmented Generation) is one retrieval technique. Context engineering is the broader discipline of managing the entire context assembly pipeline — which includes RAG, memory systems, compression, tool design, ordering, and the context window itself.

**Misconception: More rules in the system prompt make the agent more reliable.** Adding more constraints often backfires. A bloated system prompt trains the model to ignore parts of it. The goal is the "minimal set of information that fully outlines your expected behavior" (Anthropic's framing). Start minimal, add only what failure modes demand.

**Misconception: You can treat context engineering as an afterthought.** Teams that ignore it ship demos, not production systems. The gap between a working prototype and a reliable production agent is almost always a context problem, not a model capability problem.

---

## 2. Why It Matters

### The Problem It Solves

Production AI agents fail for predictable reasons:

1. **Context Rot:** As context accumulates, the model's attention dilutes across more tokens. Transformer attention is quadratic — 100,000 tokens creates roughly 10 billion pairwise relationships. The model physically cannot attend equally to all of them. Performance degrades.

2. **Lost in the Middle:** Stanford research documented this as a U-shaped performance curve. Information at the beginning and end of the context window is used reliably. Information in the middle gets ignored. In a naive agent that appends everything to conversation history, most of your injected knowledge ends up in the worst possible position.

3. **Context Poisoning:** A hallucinated fact from turn 3 gets stored and referenced in turns 7, 12, and 18. Errors compound. The model treats its own prior output as ground truth.

4. **Context Distraction:** Accumulated history causes the model to over-rely on what it has seen rather than reasoning freshly. It starts pattern-matching on context rather than thinking.

5. **Context Confusion:** Superfluous or outdated content actively misleads. If you injected documentation for v1 of your API and the user is on v2, the model will confidently use wrong endpoints.

6. **Context Clash:** Contradictory information from multiple sources (e.g., a tool result that conflicts with the system prompt) creates unpredictable behavior.

### Why It Is Becoming Important Now

Three forces converged around 2024–2025:

**Agents replaced chatbots as the deployment model.** A chatbot runs one turn. An agent runs dozens or hundreds of turns, making tool calls and accumulating state. At that scale, naive context management fails catastrophically. The shift to agents made context engineering urgent.

**Context windows grew large enough to be dangerous.** When context windows were 4K tokens, engineers were forced to be disciplined. At 128K–1M tokens, the temptation is to dump everything in. Teams that do this discover the performance and cost problems the hard way.

**Model intelligence is no longer the bottleneck.** GPT-4-era models are capable enough for most production tasks. The constraint is now giving them the right information. This is why practitioners say "context engineering is the new prompt engineering" — the leverage point has shifted.

### What Breaks If You Ignore It

**Cost.** A poorly engineered context sends 128K tokens per request when 5K would suffice. At frontier model pricing, this is a 25x cost multiplier. For enterprise scale, the difference is millions of dollars annually (see the MCP Tax analysis in Section 3).

**Reliability.** Agents that work in demos fail in production because production tasks run longer, encounter more edge cases, and accumulate more context rot. Without deliberate context management, reliability degrades as session length grows.

**Trust.** Context poisoning causes confident hallucinations that contradict user-provided information. In a business context (legal, medical, financial), this is a hard stop.

---

## 3. How Practitioners Use It

### Anthropic's Production Patterns

Anthropic published a detailed engineering post on context engineering for agents. Their core framing: "Claude is already smart enough — intelligence is not the bottleneck, context is."

Their five core techniques in production:

**System prompt hygiene.** Use XML tags or Markdown headers to separate distinct concerns. Start with the minimal viable prompt on capable models. Add examples based on observed failure modes, not preemptively. Organize into: role/identity, task description, constraints, output format, examples.

**Tool discipline.** The single most common failure mode is "bloated tool sets that cover too much functionality or lead to ambiguous decision points about which tool to use." If a human engineer can't definitively say which tool handles a given situation, an LLM can't be expected to do better. Keep tools minimal, non-overlapping, and self-contained. Each tool result should be token-efficient — strip fields that the agent doesn't need for its current decision.

**Just-in-time retrieval.** Rather than pre-loading all relevant data at session start, agents maintain lightweight identifiers (file paths, queries, URLs) and use tool calls to fetch data on demand. This is how Claude Code works: it does not load entire codebases into context. It uses targeted bash queries and file reads to bring in only what the current step requires.

**Compaction for long-horizon tasks.** When a session approaches the context window limit, the agent summarizes its conversation, discards low-signal content (old tool results, intermediate reasoning), and reinitializes a fresh context window with the summary. Anthropic's compaction strategy: prioritize recall first (capture everything relevant) then improve precision (eliminate the redundant). Crucially, preserve error traces — knowing what failed and why is high-signal information that prevents repeated mistakes.

**Sub-agent architecture.** Specialized sub-agents handle focused tasks with clean context windows. They return condensed summaries (typically 1,000–2,000 tokens) to a coordinating lead agent rather than their full traces. This isolates complexity without requiring the lead agent to wade through thousands of tokens of sub-task reasoning.

Sources: [Anthropic Engineering: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

### Real-World Company Results

**Five Sigma Insurance.** Architected AI systems that ingest policy data, claims history, and regulatory guidelines simultaneously using RAG and dynamic context assembly. Result: 80% reduction in claim processing errors, 25% increase in adjuster productivity, over 95% accuracy in underwriting after deployment cycles. Source: [MarkTechPost Case Studies](https://www.marktechpost.com/2025/08/12/case-studies-real-world-applications-of-context-engineering/)

**Microsoft.** Deployed AI code assistants with architectural context (codebase structure, coding conventions, organizational patterns) rather than just syntax knowledge. Result: 26% increase in completed software tasks, 65% fewer errors, 55% faster onboarding for new engineers, 70% better output quality. Source: [MarkTechPost Case Studies](https://www.marktechpost.com/2025/08/12/case-studies-real-world-applications-of-context-engineering/)

**ZoomInfo.** When proper context was provided (user project history, coding standards, documentation), developers achieved a 33% acceptance rate for AI suggestions and 72% developer satisfaction scores. Source: [Context Engineering Case Studies](https://www.marktechpost.com/2025/08/12/case-studies-real-world-applications-of-context-engineering/)

**Block (formerly Square).** Implemented Anthropic's Model Context Protocol to connect LLMs with live payment and merchant data rather than static prompts, improving operational automation significantly. Source: [MarkTechPost Case Studies](https://www.marktechpost.com/2025/08/12/case-studies-real-world-applications-of-context-engineering/)

### The MCP Tax: A Production Warning

The Model Context Protocol (MCP) standardizes tool connections but creates a significant hidden cost that production teams are now grappling with.

**The numbers:**
- A single production-grade tool definition: 550–1,400 tokens
- A typical developer workflow agent (GitHub, database, Slack, Jira, cloud, monitoring): ~150 tool definitions = 60,000–90,000 tokens before any conversation begins
- This represents 45–72% of a standard 128K–200K context window, consumed by tool metadata alone
- Enterprise cost estimate at scale (1,000 developers, 5 sessions/day): ~$4M annually just in unused tool definition tokens

**Performance degradation:**
- 5–10 tools: >90% accuracy
- 50+ tools: significant failure rates
- Claude Opus 4 with large toolsets: 49% accuracy (coin-flip territory)

**Anthropic's own mitigation:** A "Tool Search" feature that loads tool definitions dynamically rather than upfront reduced token cost by 85% and improved accuracy from 49% to 88.1%. A code-execution approach achieved 98.7% token savings (150,000 tokens → 2,000 tokens).

**The lesson:** MCP is useful for rapid prototyping. In production, you need dynamic tool loading, tool search, or a code-execution approach that bypasses JSON schema definitions entirely. Source: [The MCP Tax](https://www.mmntm.net/articles/mcp-context-tax)

### JetBrains Research: Observation Masking vs. LLM Summarization

JetBrains published research comparing two popular context management strategies for long-running agents:

**Observation Masking:** Replaces older environment observations with placeholders while preserving reasoning and actions. Maintains a rolling window (optimally 10 turns in testing). Simple, fast, inexpensive.

**LLM Summarization:** Uses a separate AI model to compress interaction histories into summaries. Theoretically enables infinite scaling.

**Surprising result:** Observation masking matched or exceeded summarization in 4 of 5 test scenarios. With Qwen3-Coder 480B: masking achieved 2.6% better solve rates at 52% lower cost. Why? Summarization masks failure signals — the model can't see where it went wrong, so it continues past optimal stopping points. Summary generation also triggers costly API calls with minimal cache reuse.

**The insight:** Simple approaches often outperform sophisticated ones due to reduced overhead. Default to masking. Use summarization only as a fail-safe at severe context bloat thresholds.

Source: [JetBrains Research: Efficient Context Management](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)

### Cognition AI: The Case Against Multi-Agent Parallelism

Cognition AI (makers of Devin) published a counterintuitive production finding: parallel multi-agent architectures are fundamentally fragile.

Their argument: "Share context, and share full agent traces, not just individual messages." When subagents operate independently, they lack visibility into decisions made elsewhere. Actions carry implicit decisions. Conflicting decisions produce compounding errors that are nearly impossible to reconcile.

Their recommendation: favor linear single-threaded agents with continuous context for most production tasks. Introduce parallelism only for genuinely independent subtasks with no shared state. Sub-agents should return condensed summaries (1,000–2,000 tokens) to a coordinator, not full traces. Context coherence is more valuable than parallelism speedup for reliability-sensitive workloads.

Source: [Cognition AI: Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents)

### 12-Factor Agents: The Production Engineering Framework

Dex Horthy (HumanLayer) popularized the 12-Factor Agents framework at the AI Engineer World's Fair. LangChain highlighted it as recommended reading, noting that most of the points boil down to better context engineering.

The three most context-relevant factors:

**Factor 3: Own Your Context Window.** Actively curate what enters the LLM's context. Pre-fetch and structure relevant data strategically. Don't let frameworks manage context on your behalf.

**Factor 9: Compact Errors into Context.** When operations fail, summarize errors concisely (what went wrong and why) rather than passing raw exception stacks. Error traces are high-signal context for recovery.

**Factor 12: Make Your Agent a Stateless Reducer.** Model agents as pure functions: (context + new_event) → (new_state + actions). Each invocation should be independently reproducible given the same inputs.

Source: [12-Factor Agents GitHub](https://github.com/humanlayer/12-factor-agents)

---

## 4. Design Patterns and Best Practices

### The Four Core Operations

Every context engineering technique falls into one of four operations (from Anthropic's framework):

**Write.** What goes into context — system prompts, tool definitions, retrieved documents, conversation history, agent notes.

**Select.** What gets chosen from available information — relevance filtering, retrieval ranking, which tool definitions to include.

**Compress.** How information gets condensed — summarization, extraction, truncation, masking, chunking.

**Isolate.** How information gets separated — sub-agents with clean contexts, partitioned memory, scoped tool sets.

### Context Assembly Order Matters

The "lost in the middle" problem means placement is a design decision, not an afterthought. Recommended production ordering:

1. System prompt (always first — highest recency + primacy)
2. Static tool definitions (cacheable, placed early for KV cache reuse)
3. Long-term retrieved memory (relevant facts from past sessions)
4. Task-relevant document chunks (most relevant first, not appended naively)
5. Short-term episodic summary (compressed recent session history)
6. Recent raw conversation turns (last N turns verbatim)
7. Current user message (always last — highest recency)

This ordering keeps retrieved content away from the "middle" attention dead zone and maximizes KV cache efficiency for static prefixes.

Source: [LLM Context Window Management Engineering Patterns](https://tanujgarg.com/blog/llm-context-window-management-production)

### KV Cache as a Context Engineering Lever

Prompt caching (available in Claude, GPT-4, Gemini) stores key-value pairs from the attention computation and reuses them across requests. The practical implication: structure your context so the cacheable parts (system prompt, static tool definitions) form a stable prefix that never changes between requests.

Rules for KV cache optimization:
- **Stable prefixes.** Keep variable content out of the prefix. The moment a token changes, all subsequent cache is invalidated.
- **Append-only context.** Never rewrite prior messages. Append correction events. Rewriting breaks cache matching downstream.
- **Separate static from dynamic.** Static content (system prompt, tool schemas) belongs at the top. Dynamic content (conversation history, retrieved docs) belongs at the bottom.
- **Avoid mid-session tool changes.** Dynamically adding or removing tool definitions mid-iteration invalidates the KV cache for everything after the change point.

Source: [Context Engineering: KV Cache, File Management, Prefill](https://medium.com/@joycebirkins/context-engineering-for-complex-agent-systems-kv-cache-file-management-prefill-prompts-and-rag-c7e0f3ba2cd3)

### Chunking and RAG Patterns That Work in Production

Naive RAG (chunk documents → embed → cosine similarity → stuff into prompt) was always a prototype, not a production system. Patterns that work:

**Semantic chunking over fixed-size chunking.** Split on paragraph and section boundaries using similarity thresholds rather than fixed token counts. Preserves semantic coherence, especially for technical documentation.

**Two-stage retrieval (recall + rerank).** Stage 1: broad vector search retrieves a large candidate set (top 20–50). Stage 2: a cross-encoder reranker scores candidates against the actual query. Final: pass only top 3–5 re-ranked chunks to the LLM. This dramatically improves precision without sacrificing recall.

**Hybrid search.** Combine semantic (vector) search with keyword (BM25) search. Semantic search handles conceptual queries; BM25 handles exact-match lookups (product names, error codes, identifiers). Neither alone is sufficient.

**Position matters.** Place your highest-confidence retrieved documents at the beginning and end of the context. The middle is where attention degrades. Never naively concatenate retrieved chunks in order.

**Agentic RAG.** Instead of a fixed retrieval pipeline, the agent controls retrieval: it decides when to search, can reformulate queries when results are insufficient, and iterates until confident. GraphRAG (Microsoft) builds knowledge graphs over documents, enabling multi-hop reasoning across connected entities.

Source: [RAG Is Not Dead: Advanced Retrieval Patterns That Actually Work](https://dev.to/young_gao/rag-is-not-dead-advanced-retrieval-patterns-that-actually-work-in-2026-2gbo), [State of Context Engineering in 2026](https://www.newsletter.swirlai.com/p/state-of-context-engineering-in-2026)

### Memory Architecture for Long-Running Agents

A production memory system is not a single vector database. It is a tiered architecture:

| Tier | Contents | Storage | Latency |
|---|---|---|---|
| Working memory | Current task state, active constraints | Context window | Instant |
| Short-term episodic | Last N turns verbatim + rolling summary | In-memory / Redis | Sub-millisecond |
| Long-term episodic | Compressed session summaries | Vector store | Low ms |
| Semantic memory | Facts, user preferences, domain knowledge | Vector store / Graph DB | Low ms |
| Procedural memory | How-to knowledge, learned patterns | Files / Vector store | Low ms |

The agent decides what to move between tiers. Letta (formerly MemGPT) implements this via explicit memory management tool calls — the agent calls `archival_memory_insert` to move information to long-term storage and `archival_memory_search` to retrieve it. This LLM-managed tiered memory is a reference implementation for stateful long-horizon agents.

Note: Vector databases are appropriate for documentation context and long-conversation recall. They are NOT appropriate for critical facts, permissions, or financial data. Use structured storage with deterministic retrieval for anything where accuracy is non-negotiable.

Source: [Mem0 vs Letta: AI Agent Memory Compared](https://vectorize.io/articles/mem0-vs-letta), [AI Agent Memory Architectures](https://zylos.ai/research/2026-04-05-ai-agent-memory-architectures-persistent-knowledge/)

### Token Budget Management

Treat the context window as an explicit budget with allocations:

- System prompt: 10–15%
- Tool schemas: 10–20% (minimize aggressively; use dynamic loading)
- Retrieved context: 30–40%
- Conversation history: 20–30%
- Buffer for response: 10–15%

If any component exceeds its allocation before the request goes out, it gets compressed or truncated. The system prompt and current user message are never trimmed. Everything else is fair game, starting with the lowest-relevance items.

Compression pipeline for a given component: relevance filter → semantic deduplicate → extractive summarize → sentence prune. Combined: 50–80% token reduction with minimal information loss.

Source: [The Hidden Costs of Context: Managing Token Budgets](https://tianpan.co/blog/2025-11-11-managing-token-budgets-production-llm-systems)

### Anti-Patterns to Avoid

**The wall-of-text dump.** Pasting all potentially relevant content as one monolithic block with no structure. The model can't reason about what's important. Use XML tags, Markdown headers, and explicit section labels.

**Context-as-afterthought.** Building the agent logic first, then worrying about what goes in the context window. Context is the core engineering decision. Design it first.

**Growing the context window to fix bugs.** When something breaks, engineers instinctively add more instructions, more examples, more constraints. Past a threshold this makes things worse — the model starts ignoring inconsistent instructions and attention dilutes. Instead, diagnose what specific information is missing, add only that, and consider whether older context should be compressed.

**Uniform tool loading.** Loading all tool definitions for every request regardless of which tools are relevant. For MCP-heavy systems, this consumes the majority of the context window before any reasoning begins.

**Ignoring ordering.** Appending retrieved chunks to conversation history in arbitrary order. Placement directly affects attention weighting. Design the ordering.

**Storing everything in a single memory tier.** Using one vector database for all agent memory types. Different information has different retrieval semantics. A fact you looked up 3 sessions ago and a permission rule that governs current behavior should not be retrieved by the same cosine similarity query.

**Not testing context assembly.** Context assembly is business logic. It should have deterministic unit tests: verify summaries capture key facts, confirm chunk ordering matches relevance scores, validate trimming removes lowest-scored items first. Source: [Why You're Doing Context Engineering Wrong](https://materialize.com/blog/why-youre-doing-context-engineering-wrong-live-data-architecture/)

---

## 5. Advanced Insights

### Nuances Beginners Miss

**System prompt altitude.** System prompts need to be "specific enough to guide behavior effectively yet flexible enough to provide strong heuristics." The failure mode is writing a system prompt at the wrong altitude — either so vague it provides no guidance, or so specific it breaks on any edge case not explicitly enumerated. Good system prompts define the shape of behavior, not every behavior.

**Error traces are high-signal context.** Most engineers treat errors as things to hide or suppress. In context engineering, error traces are among the most valuable tokens you can provide. Knowing what failed and why directly prevents repeated mistakes. Never compress away error information during compaction.

**Compaction is lossy by design, and that's okay.** You cannot preserve everything. The goal is to preserve the right things: task state, key decisions, error history, current objectives. The rest (intermediate tool results, exploratory reasoning that didn't lead anywhere) is noise. Design compaction to be aggressively lossy for process and lossless for outcomes.

**Few-shot examples are more efficient than rules.** Anthropic's guidance: instead of enumerating edge cases as rules, curate a diverse set of canonical examples that portray expected behavior. Examples communicate constraints, tone, format, and reasoning style simultaneously in a compact form the model reads fluently.

**Progressive disclosure for tools.** Rather than loading full tool definitions upfront, load tool descriptions first (brief summaries), then load full schemas only when the agent selects a specific tool. This matches how human engineers navigate API documentation — they scan method names and docstrings before reading parameter details.

### Expert Disagreements

**Context engineering vs. RAG + memory + orchestration stack.** A legitimate skeptical question: is "context engineering" just a new name for the RAG / memory / orchestration practices that already existed? The emerging answer from practitioners is that the framing matters. Calling it "context engineering" shifts the mental model from "build a retrieval pipeline" to "manage a finite resource with architectural intent." Whether the terminology represents a genuine conceptual advance or rebranding is still debated. Source: [The Great Context Engineering Debate](https://aipmguru.substack.com/p/the-great-context-engineering-debate)

**Fine-tuning vs. context engineering.** When should you fine-tune vs. engineer better context? The emerging practitioner consensus: use fine-tuning and RL to build durable domain knowledge (style, domain vocabulary, reasoning patterns). Use context engineering for local, rapidly-changing, session-specific facts. They are complementary, not competing. Context engineering is not a substitute for fine-tuning when the knowledge needs to be consistently encoded in model weights.

**Observation masking vs. summarization.** JetBrains research found masking outperforms summarization in most scenarios. But practitioners at other companies (Anthropic, Mem0) invest heavily in compaction/summarization. The disagreement may be task-specific: masking works for coding agents where the action history is dense with signal; summarization may be necessary for long conversations where the semantic content of prior turns matters more than the exact sequence.

**When to build multi-agent vs. single-agent.** Cognition AI says don't build multi-agents. Anthropic uses sub-agent architectures in Claude Code. The practical resolution: use sub-agents for genuinely isolated subtasks where context independence is a feature, not a compromise. Use single agents where coherence across steps matters more than parallelism.

### Emerging Trends

**Context routing.** Before loading any context sources, classify the query to determine which retrieval paths are relevant. LLM-based routing is accurate but adds latency; rule-based routing is fast but rigid; hybrid approaches combine both. The open problem: LLM routing can hallucinate routing decisions, requiring fallback logic.

**Agentic RAG.** The agent controls its own retrieval strategy, can reformulate queries, and iterates until confident. Self-RAG pushes further: the model assesses whether retrieval is even needed before calling a retrieval tool. This reduces unnecessary retrieval latency and avoids context pollution from irrelevant chunks.

**Knowledge runtimes.** The trend toward context as managed infrastructure: systems that handle retrieval, verification, access control, and audit trails as integrated operations — similar to container orchestrators for application workloads. Tools like Letta, Mem0, and emerging knowledge graph systems position themselves here.

**Context as code.** Treating context assembly definitions as versioned, tested, deployable artifacts — not as strings embedded in application code. This means context assembly pipelines get the same engineering rigor (CI, testing, version control, rollback) as application code.

**The durability question.** As models improve, will context engineering become less important? The current consensus: context engineering as a discipline will remain central, but the specific techniques will evolve. Models that can better attend to long contexts shift which optimizations matter. But the finite resource constraint never disappears — it just moves.

---

## 6. Curated Reading List

### Primary Sources

**[Anthropic Engineering: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)**
Why read it: The most authoritative production guide from the team that built Claude. Covers compaction, tool design, just-in-time retrieval, and sub-agent architectures with specific implementation guidance. Primary source, not a summary.
Difficulty: Intermediate
Time: 25 minutes
Key takeaways: Context is a finite resource. Error traces are high-signal. Sub-agents should return condensed summaries. Start with minimal prompts, add based on failure modes.

**[12-Factor Agents (GitHub)](https://github.com/humanlayer/12-factor-agents)**
Why read it: The most widely-cited practical framework for production agent engineering. Read the README and the linked talk. Every factor has direct context engineering implications.
Difficulty: Intermediate
Time: 30 minutes
Key takeaways: Own your context window. Compact errors into context. Make agents stateless reducers. Most production agents are mostly deterministic code with strategic LLM augmentation.

**[Cognition AI: Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents)**
Why read it: A counterintuitive primary-source argument from a team that ships production coding agents. Forces you to think carefully about when parallelism actually helps.
Difficulty: Beginner
Time: 10 minutes
Key takeaways: Context coherence is more valuable than parallelism for most reliability-sensitive tasks. Sub-agents should return summaries, not full traces.

**[State of Context Engineering in 2026 (SwirlAI Newsletter)](https://www.newsletter.swirlai.com/p/state-of-context-engineering-in-2026)**
Why read it: The most comprehensive practitioner survey of where the field is as of 2026. Covers progressive disclosure, compression, routing, retrieval evolution, and tool management with honest tradeoffs.
Difficulty: Intermediate
Time: 30 minutes
Key takeaways: Each pattern has specific tradeoffs. OpenAI recommends fewer than 20 tools per agent; accuracy degrades past 10. Avoid dynamic tool changes mid-iteration to preserve KV cache.

**[JetBrains Research: Efficient Context Management](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)**
Why read it: Empirical research comparing two dominant context management strategies. Results were surprising and contradict conventional wisdom.
Difficulty: Intermediate
Time: 20 minutes
Key takeaways: Observation masking outperformed LLM summarization in 4 of 5 scenarios. Summarization can mask failure signals. Simple approaches often outperform sophisticated ones.

**[The MCP Tax: Hidden Costs of Model Context Protocol](https://www.mmntm.net/articles/mcp-context-tax)**
Why read it: Concrete production numbers on a specific, real failure mode. A 30x cost differential is documented with methodology.
Difficulty: Intermediate
Time: 20 minutes
Key takeaways: 6 MCP servers can consume 72% of a 200K context window before any conversation. Dynamic tool loading is essential. Code-execution approaches bypass the problem entirely.

### Supporting Reading

**[Redis: Context Engineering Best Practices](https://redis.io/blog/context-engineering-best-practices-for-an-emerging-discipline/)**
Why read it: Vendor-authored but substantive overview of context engineering from an infrastructure perspective. Good on memory architecture and the pyramid approach for prompt ordering.
Difficulty: Beginner
Time: 20 minutes
Key takeaways: Pyramid structure for context. Context as infrastructure. Context failures vs. model failures distinction.

**[Materialize: Why You're Doing Context Engineering Wrong](https://materialize.com/blog/why-youre-doing-context-engineering-wrong-live-data-architecture/)**
Why read it: Makes the case for live data architecture as a context engineering strategy. Strong on diagnosing the failure modes.
Difficulty: Intermediate
Time: 15 minutes
Key takeaways: Four failure modes (confusion, poisoning, distraction, clash). Live data beats stale context. Metadata freshness is often the hidden problem.

**[Neo4j: Context Engineering vs. Prompt Engineering](https://neo4j.com/blog/agentic-ai/context-engineering-vs-prompt-engineering/)**
Why read it: Thorough comparison with specific detail on when each approach applies and what skills the transition requires.
Difficulty: Beginner
Time: 20 minutes
Key takeaways: Minimum Viable Context principle. Knowledge graphs for multi-hop retrieval. The new skills required for context engineers vs. prompt engineers.

**[Awesome Context Engineering (GitHub)](https://github.com/Meirtz/Awesome-Context-Engineering)**
Why read it: Curated survey of hundreds of papers, frameworks, and implementation guides. Use as a reference for diving deeper on any specific sub-topic.
Difficulty: Reference
Time: Browse as needed
Key takeaways: Comprehensive taxonomy of the field's technical sub-areas.

**[LLM Context Window Management: Engineering Patterns for Production](https://tanujgarg.com/blog/llm-context-window-management-production)**
Why read it: Detailed technical breakdown of context assembly ordering, chunking strategies, and memory tier design.
Difficulty: Advanced
Time: 30 minutes
Key takeaways: The production context assembly order. Why memory tiers need different retrieval semantics. Testing requirements for context assembly logic.

**[Simon Willison on Context Engineering](https://simonwillison.net/2025/Jun/27/context-engineering/)**
Why read it: Brief but historically important — documents the moment "context engineering" entered mainstream usage. Good for understanding the conceptual transition from prompt engineering.
Difficulty: Beginner
Time: 5 minutes
Key takeaways: Why terminology matters. The Karpathy and Lütke definitions.

---

## 7. Learning Path

### If You Have 30 Minutes

1. Read [Simon Willison on Context Engineering](https://simonwillison.net/2025/Jun/27/context-engineering/) (5 min) — understand the conceptual shift.
2. Read [Cognition AI: Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) (10 min) — the most counterintuitive production insight.
3. Read [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (15 min) — the authoritative production guide.

The core insight you will take away: context engineering is resource management for finite attention. The model is good enough. Give it the right information, in the right structure, at the right time.

### If You Have 2 Hours

Work through in order:

1. [Simon Willison on Context Engineering](https://simonwillison.net/2025/Jun/27/context-engineering/) (5 min) — conceptual grounding.
2. [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) (25 min) — primary source depth.
3. [Cognition AI: Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) (10 min) — multi-agent tradeoffs.
4. [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) (30 min) — production engineering framework.
5. [State of Context Engineering in 2026](https://www.newsletter.swirlai.com/p/state-of-context-engineering-in-2026) (30 min) — comprehensive current state.
6. [The MCP Tax](https://www.mmntm.net/articles/mcp-context-tax) (20 min) — a specific, quantified production failure mode.

By the end: you understand the full stack from concepts to production patterns to specific quantified failure modes.

### If You Want to Become Highly Knowledgeable (One Week)

**Day 1: Foundations**
- Simon Willison, Karpathy's tweet, the Neo4j comparison piece
- Read the Firecrawl blog on context engineering vs. prompt engineering
- Goal: have clean mental models before diving into techniques

**Day 2: Anthropic's Production Stack**
- Anthropic: Effective Context Engineering for AI Agents (read fully)
- Anthropic: Effective Harnesses for Long-Running Agents
- Anthropic: Building Agents with the Claude Agent SDK
- Goal: understand how the most mature agent production team thinks about context

**Day 3: The 12-Factor Framework and Anti-Patterns**
- 12-Factor Agents (README + linked talk)
- Six Critical Mistakes in Agentic AI Engineering (DecodingAI)
- Materialize: Why You're Doing Context Engineering Wrong
- Goal: have a framework for what goes wrong and why

**Day 4: Retrieval and Memory**
- JetBrains Research: Efficient Context Management
- LLM Context Window Management: Engineering Patterns for Production
- Mem0 vs Letta comparison
- Goal: understand the retrieval and memory stack in detail

**Day 5: The MCP and Tool Ecosystem**
- The MCP Tax
- State of Context Engineering in 2026 (SwirlAI)
- KV Cache context engineering (Joyce Birkins)
- Goal: understand tool design and MCP implications for production

**Day 6: Advanced Patterns**
- Awesome Context Engineering GitHub repo (browse the papers section)
- Context Engineering 2.0 paper (arXiv 2510.26493)
- Read 2–3 papers from the memory survey GitHub repo
- Goal: understand the research frontier

**Day 7: Apply and Reflect**
- Re-read this document
- Sketch the context engineering design for a system you are building or want to build
- Identify which failure modes your current system is most susceptible to
- Write down three concrete changes you will make

---

## 8. Practical Application

### For Builders of AI Products Today

**Start here: context audit.** Before optimizing, understand what is currently in your context window for a typical production request. Log the full context for 10 real sessions. Measure: total tokens, breakdown by component (system prompt / tools / history / retrieved content / current message), and average session length. This audit will reveal where the waste is. Most teams discover they are spending 30–60% of their token budget on stale history or overly broad tool definitions.

**Apply the MVC principle.** Minimum Viable Context: user goals + highly relevant retrieved information only + necessary tool definitions + applicable policies + compacted memory summaries. Nothing else. Build toward this as an explicit constraint, not as an afterthought.

**For agents: implement compaction before you need it.** Do not wait until you hit context limits in production. Build compaction logic early, test it on representative long sessions, and tune the recall/precision tradeoff before real users trigger it.

### For the Dalgo Platform Specifically

Context engineering is directly relevant to Dalgo's AI features. Dalgo agents interact with data pipelines (Airbyte), transformation graphs (dbt), orchestration (Prefect), and warehouse schemas. Here is how each context engineering pattern applies:

**System prompt design.** Dalgo's AI features are used by non-technical NGO program managers. The system prompt altitude matters enormously: the assistant needs to understand Dalgo domain vocabulary (sources, connections, pipelines, models, transformations) without requiring technical user input. Design system prompts that encode domain-specific vocabulary as compact definitions, not as lengthy explanations. Use examples that show the expected output format for common tasks.

**Tool discipline for MCP.** If Dalgo exposes its MCP tools to agents, the MCP tax applies directly. The current MCP tool list includes ~80+ tools. Injecting all of them for every request would consume a significant portion of the context window before any reasoning begins. The solution: tool routing. Classify the user's request first (is this about pipelines? sources? dashboards? transforms?) and load only the relevant tool subset.

**Just-in-time retrieval over schema pre-loading.** When an agent needs to answer questions about the user's data, it should not pre-load all schema information. Instead, it should use `list_schemas` → `list_tables` → `get_table_columns` as needed, loading only the tables relevant to the current task. Claude Code's approach to large codebases (targeted queries rather than full loads) is the direct model.

**Context memory for multi-session continuity.** NGO users return to the same data analysis tasks repeatedly. A memory layer that persists user preferences (preferred chart types, frequently-used filters, dashboard layout conventions) and past analytical decisions would significantly reduce context window usage per session while improving response quality. Mem0 or a simpler Redis-backed pattern would work here.

**Evaluation of context quality.** Track per-session: token usage by component, task completion rate by context length, and error/retry rate. These metrics will reveal whether context engineering improvements are working.

**Guardrails and poisoning prevention.** Dalgo operates on real NGO data. Context poisoning risk is real: if a hallucinated schema name is stored in agent memory and referenced in subsequent turns, it will cause downstream failures. Design the memory tier so retrieved facts are tagged with provenance (which API call returned this) and treated as needing verification before being used for irreversible actions (pipeline triggers, schema modifications).

### For Evaluations and Testing

Context assembly is deterministic business logic and should be tested as such:

- Unit test: given a session with N turns and M retrieved documents, verify the assembled context is within budget, correctly ordered, and contains the expected components.
- Regression test: verify that compaction preserves key decisions and error traces across a representative set of synthetic long sessions.
- Performance test: for varying context lengths (1K, 5K, 20K, 50K tokens), measure task completion rate and verify it does not degrade unacceptably.
- Cost test: track token usage per request in CI and alert on regressions.

---

## Sources Index

All primary sources cited in this report:

- [Anthropic Engineering: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Andrej Karpathy on X (June 2025)](https://x.com/karpathy/status/1937902205765607626)
- [Simon Willison: Context Engineering](https://simonwillison.net/2025/Jun/27/context-engineering/)
- [12-Factor Agents GitHub (HumanLayer)](https://github.com/humanlayer/12-factor-agents)
- [Cognition AI: Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents)
- [State of Context Engineering in 2026 (SwirlAI)](https://www.newsletter.swirlai.com/p/state-of-context-engineering-in-2026)
- [JetBrains Research: Efficient Context Management](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)
- [The MCP Tax: Hidden Costs of Model Context Protocol](https://www.mmntm.net/articles/mcp-context-tax)
- [Context Engineering vs. Prompt Engineering (Neo4j)](https://neo4j.com/blog/agentic-ai/context-engineering-vs-prompt-engineering/)
- [Why You're Doing Context Engineering Wrong (Materialize)](https://materialize.com/blog/why-youre-doing-context-engineering-wrong-live-data-architecture/)
- [LLM Context Window Management: Engineering Patterns for Production](https://tanujgarg.com/blog/llm-context-window-management-production)
- [Redis: Context Engineering Best Practices](https://redis.io/blog/context-engineering-best-practices-for-an-emerging-discipline/)
- [Context Engineering vs. Prompt Engineering (Firecrawl)](https://www.firecrawl.dev/blog/context-engineering)
- [Six Critical Mistakes in Agentic AI Engineering (DecodingAI)](https://www.decodingai.com/p/agentic-ai-engineering-guide-6-mistakes)
- [MarkTechPost: Case Studies — Real-World Applications of Context Engineering](https://www.marktechpost.com/2025/08/12/case-studies-real-world-applications-of-context-engineering/)
- [Mem0 vs Letta: AI Agent Memory Compared](https://vectorize.io/articles/mem0-vs-letta)
- [AI Agents in 2026: Tools, Memory, Evals, and Guardrails](https://andriifurmanets.com/blogs/ai-agents-2026-practical-architecture-tools-memory-evals-guardrails)
- [Awesome Context Engineering (GitHub)](https://github.com/Meirtz/Awesome-Context-Engineering)
- [KV Cache Context Engineering (Joyce Birkins, Medium)](https://medium.com/@joycebirkins/context-engineering-for-complex-agent-systems-kv-cache-file-management-prefill-prompts-and-rag-c7e0f3ba2cd3)
- [Context Engineering: From Prompts to Corporate Multi-Agent Architecture (arXiv)](https://arxiv.org/pdf/2603.09619)
- [RAG Is Not Dead: Advanced Retrieval Patterns That Actually Work in 2026](https://dev.to/young_gao/rag-is-not-dead-advanced-retrieval-patterns-that-actually-work-in-2026-2gbo)
- [Context Engineering for Reliable AI Agents (Kubiya)](https://www.kubiya.ai/blog/context-engineering-ai-agents)
- [The Great Context Engineering Debate (AI PM Guru)](https://aipmguru.substack.com/p/the-great-context-engineering-debate)
