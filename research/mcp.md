# Model Context Protocol (MCP): Deep Research Report

*Research date: June 2026. Sources fetched directly from primary documentation, engineering blogs, research papers, and practitioner discussions.*

---

## 1. Core Concepts

### What MCP Is

Model Context Protocol (MCP) is an open protocol for connecting AI applications to external systems. It standardizes how AI models discover, access, and invoke tools, data sources, and workflows. The canonical analogy: MCP is the USB-C port for AI — a single standard connector replacing hundreds of custom cables.

MCP was created by Anthropic, open-sourced in November 2024, and donated in December 2025 to the Agentic AI Foundation (AAIF) under the Linux Foundation. As of mid-2026, it is governed jointly by Anthropic, Google, OpenAI, Microsoft, AWS, and Cloudflare.

### Architecture: Three Roles

MCP uses a strict client-server architecture with three distinct participants:

- **Host**: The AI application the user interacts with (Claude Desktop, VS Code, Cursor, ChatGPT). The host creates and manages one or more MCP clients.
- **Client**: A component *inside* the host that maintains a dedicated one-to-one connection to a specific MCP server. Each server gets its own client instance — they are isolated.
- **Server**: A program (local or remote) that exposes capabilities (tools, data, prompts) via the MCP protocol. A server can be as simple as a Python script spawned as a subprocess.

This means: one host can connect to many servers, but each connection is isolated. There is no broadcast; each client talks to exactly one server.

### Two Layers

**Data layer** (inner): JSON-RPC 2.0-based protocol defining message structure, lifecycle management, primitives, and notifications.

**Transport layer** (outer): Two mechanisms:
- **Stdio**: Standard input/output for local processes on the same machine. Zero network overhead. Used for spawning local server processes.
- **Streamable HTTP**: HTTP POST for client-to-server messages, with optional Server-Sent Events (SSE) for streaming. Used for remote servers. Supports OAuth authentication.

### Six Primitives (the Most Important Concept)

MCP defines primitives that servers expose and primitives that clients expose. Getting this distinction right matters for both design and security.

**Server-side primitives (what servers offer):**

1. **Tools** — Executable functions the AI model can invoke (file operations, API calls, database queries). *Model-controlled*: the LLM decides when to call them. Side effects are expected.

2. **Resources** — Data sources providing contextual information (file contents, DB records, API responses). *Application-controlled*: the host decides when to load them, not the model. Think: read-only context injection, not action-taking.

3. **Prompts** — Reusable templates structuring model interactions (system prompts, few-shot examples). *User-controlled*: the human selects which prompt to activate.

**Client-side primitives (what clients offer back to servers):**

4. **Sampling** — A server can ask the client to generate an LLM completion. This inverts the usual direction: the server borrows the host's model without needing its own API key. Useful for agentic servers that need to reason without bundling LLM credentials. The *client* stays in control of which model runs and can reject requests.

5. **Elicitation** — A server can pause execution and request structured input from the user through the client. Enables human-in-the-loop patterns: an agent can pause, prompt the user for approval or missing data, then continue.

6. **Roots** — Allows servers to ask clients about URI or filesystem boundaries they are authorized to operate within.

**Experimental cross-cutting primitive:**

7. **Tasks** — A durable execution wrapper decoupling work initiation from retrieval. Tools are synchronous and block until complete; Tasks return a `taskId` immediately and allow polling or webhook callbacks. Enables long-running workflows that survive connection drops.

### Mental Models to Internalize

**The control plane split**: Tools are for the model, Resources are for the application, Prompts are for the user. Putting a read-only data lookup into a Tool wastes tokens and expands the attack surface. Putting a write operation in a Resource creates a dangerous misconception. The primitive type signals who initiates and what the risk profile is.

**MCP does not own context management**: The spec says nothing about how the host presents tool results to the model. It defines the exchange protocol, not the reasoning loop. The host decides whether to stuff everything into the context window or do retrieval selectively.

**Capability negotiation at initialization**: Every connection starts with an `initialize` / `initialized` handshake where both sides declare supported capabilities. A server that declares `"tools": {"listChanged": true}` can push `notifications/tools/list_changed` when its tool catalog changes. Clients that don't declare `elicitation` capability will never receive elicitation requests. This means you cannot assume a capability is available — you must check what was negotiated.

**Stateful by default, stateless optionally**: MCP is a stateful protocol. Streamable HTTP transport can be made stateless for horizontal scaling, but this requires explicit session management. The 2026 roadmap makes stateless Streamable HTTP a priority because production deployments need it.

### Common Misconceptions

**"MCP replaces function calling."** It does not. Function calling is how a model *expresses intent* (returning structured JSON to say "I want to call X"). MCP is how that intent gets *executed* across a standardized infrastructure layer. Most production systems use both: function calling for intent expression, MCP for execution routing.

**"Every MCP tool call is the model deciding."** Only Tools are model-controlled. Resources are application-controlled. Prompts are user-controlled. Sampling runs in the opposite direction. Many developers expose everything as Tools and wonder why their context window fills up.

**"MCP is only for Claude."** Anthropic created it, but OpenAI adopted it in April 2025, Microsoft in July 2025, AWS in November 2025. All major AI clients now support it. The spec is governed by a multi-stakeholder foundation, not Anthropic.

**"Tool descriptions are just documentation."** They are instructions to the model. The LLM reads tool descriptions and decides which tool to call based on them. A poorly written description causes wrong tool selection. A maliciously crafted description (tool poisoning) can hijack agent behavior.

---

## 2. Why It Matters

### The N×M Integration Problem

Before MCP, connecting N AI applications to M tools required N×M custom integrations. Each combination needed bespoke authentication, error handling, schema translation, and maintenance. A team running Claude + GPT + Gemini connecting to GitHub + Slack + databases + internal APIs faced a combinatorial explosion.

MCP reduces this to N+M: build one MCP server per tool, one MCP client per AI application. Any client can use any server.

### Why Now

Several forces converged in 2024-2025:

- **Agents became real**: As AI systems moved from chat to action-taking, the lack of standardized tool protocols became a genuine bottleneck, not a theoretical concern.
- **Ecosystem fragmentation was hurting everyone**: GitHub, Stripe, Cloudflare, and hundreds of other companies were building bespoke AI integrations and seeing them break every time an AI client updated.
- **OpenAI's adoption was the tipping point**: When OpenAI adopted MCP in April 2025, it signaled the protocol was no longer Anthropic's experiment but an emerging standard. Downloads went from 2M/month to 22M/month within weeks.
- **Linux Foundation governance removed lock-in concerns**: Anthropic donating MCP to the AAIF in December 2025 resolved enterprise objections about single-vendor control.

### What Breaks Without It

- **Tool logic is duplicated**: The same GitHub integration gets reimplemented by every team building an AI product. Bugs get fixed in one place but not others.
- **Auth and security are inconsistent**: Without a standard, every integration handles credentials differently. Most do it badly.
- **Agents can't evolve independently of applications**: If tool implementations live inside the AI application, updating a tool requires a new application release. MCP servers can update independently.
- **No observable ecosystem**: Without standardized tool registries, users have no way to discover what integrations exist, and developers have no way to build on each other's work.

---

## 3. How Practitioners Use It

### Production Deployments (Documented)

**GitHub** (open-sourced November 2024): GitHub's MCP server is the single most widely deployed example. It acts as a "source-of-truth interface between GitHub and any LLM." Documented production uses include: converting GitHub issues to Markdown for website content, compiling weekly PR digests posted to Slack, natural language to structured GitHub API operations, and proactive alerts on stale reviews. GitHub open-sourced the server explicitly to let the community inspect and contribute to it rather than maintaining a black box.

**Stripe**: Extensive payment workflow management via MCP, exposing tools like `balance.read`, `payment_links.create`, and `refunds.create`. Clients can discover capabilities dynamically rather than hardcoding API surface area. Used in production by AI coding assistants and agent workflows.

**Cloudflare**: Deployed MCP as part of its Workers infrastructure. MCP servers run at the edge for global low latency. Cloudflare's own internal tooling uses MCP for AI-assisted operations.

**Block** (Square): Built Square AI and Moneybot using MCP. One of the three co-founders of the AAIF alongside Anthropic and OpenAI. Block's production usage helped shape the protocol's enterprise authentication requirements.

**AWS**: Added platform-level MCP support in November 2025. The `awslabs/mcp` GitHub repository contains open-source MCP servers for AWS services including Bedrock, CDK, core AWS services, cost analysis, DynamoDB, Lambda, RDS, and more.

**Healthcare analytics**: Documented at the MCP one-year anniversary as processing hundreds of thousands of data points through MCP-enabled agent workflows. Specific company not named but use case: complex multi-step data analysis with human approval steps via Elicitation.

**Context7 (developer tooling)**: MCP server providing LLMs with current, version-specific documentation and code samples. Addresses the hallucination problem where models generate code for deprecated API versions by injecting real-time docs as Resources.

**Playwright/Selenium**: Both have released MCP servers that expose browser automation capabilities, enabling AI agents to drive UI testing. Widely used in CI/CD pipelines.

### What the Numbers Look Like

From Anthropic's November 2025 milestone post and subsequent reports:
- ~2,000 servers in the official registry (November 2025)
- 97 million monthly SDK downloads (March 2026)
- 10,000+ active public MCP servers
- 58 active maintainers
- 2,900+ Discord contributors

### Patterns from the Yaw.sh Team (Built 9 Production Servers)

This is one of the most candid practitioner write-ups available. Key findings from building Tailscale, SSH, Caddy, npm, and other servers:

- Tool description quality determines agent behavior more than implementation quality. The LLM chooses tools based solely on name and description. "List all devices in the tailnet with their online status, IP addresses, and tags" outperforms "Get devices endpoint."
- Return structured JSON, not human-readable strings. If a tool returns "Successfully created auth key tskey-auth-abc123", an agent parsing that string for follow-up operations is fragile. Return `{"key": "tskey-auth-abc123", "expires": "..."}`.
- Error messages must be actionable. "Forbidden" is useless. Include what permissions are missing, what the current scopes are, and what the user needs to do. Agents self-correct based on error content.
- 89 tools in a single server is too many. They measured client failures from large tool lists and are now refactoring toward fewer tools with an `action` parameter grouping related operations.
- Build a compliance test harness. Their servers worked in Claude Code but failed in other clients. Edge cases around error codes, pagination, and schema validation are easy to get wrong.

---

## 4. Design Patterns and Best Practices

### The Right Primitive for the Job

| Scenario | Primitive | Reason |
|---|---|---|
| Query a database | Resource (if read-only, deterministic) or Tool | Tool if query params vary; Resource if it's static context |
| Write to a database | Tool only | Has side effects |
| System prompt | Prompt | User-selected, reusable |
| Real-time API call | Tool | Model-controlled, dynamic |
| Large document as context | Resource | Application-controlled, loaded when needed |
| Agent needs to reason mid-execution | Sampling | Server borrows client's LLM |
| Human approval required | Elicitation | Pause, collect, resume |
| Long-running compute | Tasks (experimental) | Async, survives disconnects |

### One Tool Per Intent

The most-repeated design principle: one tool, one clear responsibility. Avoid "god tools" that do multiple things based on parameters. A tool named `manage_database` with a `mode` parameter is harder for an LLM to call correctly than `query_database`, `insert_record`, and `delete_record`.

The exception: when you have 80+ tools in one server, grouping related low-frequency operations under a single tool with an `action` enum prevents context window overflow. This is a pragmatic tradeoff, not ideal design.

### Tool Descriptions as an Interface Contract

Write descriptions for the model, not for humans reading documentation:
- Lead with what the tool does, not what it is
- Include when to use it vs. when not to
- Specify what format inputs should be in
- Describe what the output looks like and what a caller should do with it

From Microsoft Research's tool-space study: tools with ambiguous descriptions cause wrong tool selection. Naming collisions (32 different servers have a tool called "search") confuse models across multi-server deployments.

### Server Granularity

**Narrow over broad**: Each MCP server should have one clear, well-defined purpose. Cloudflare's guidance: "deploy several focused MCP servers, each with narrowly scoped permissions" rather than one server with broad access. This limits blast radius, improves auditability, and makes it easier to reason about authorization.

Avoid the direct-API-wrapper trap (the most common beginner mistake): do not create 1:1 mappings between MCP tools and existing REST API endpoints. Model the tasks a user wants to accomplish, not the API surface. "Create and send invoice" is one tool. Separately exposing `POST /invoices`, `POST /invoices/{id}/send` is API documentation, not a tool catalog.

### Transport Selection

| Scenario | Transport |
|---|---|
| Local development | Stdio |
| Spawned subprocesses (same machine) | Stdio |
| Remote server, single instance | Streamable HTTP |
| Remote server, horizontally scaled | Streamable HTTP (stateless mode, 2026 roadmap) |
| Edge deployment | Streamable HTTP via Cloudflare Workers |

SSE via nginx requires `Connection: ''` (empty string), not `Connection: upgrade` which is for WebSockets — a well-known deployment gotcha.

### Authentication Pattern for Remote Servers

OAuth 2.1 with PKCE is the current standard for remote server auth (mandated since the March 2025 spec update). Key implementation notes:
- MCP clients often run in open environments (desktop apps) and cannot protect a `client_secret`, so the Authorization Code grant with PKCE is the correct choice.
- Avoid Dynamic Client Registration (DCR) deployed without strict redirect URI validation — it is the broadest attack surface in MCP auth.
- Prefer Client ID Metadata Documents (CIMD) over DCR for scale. CIMD allows a single client to connect to thousands of previously unknown servers without requiring each server to register the client.
- Resource Indicators (RFC 8707) are mandatory in the June 2025 spec revision — they prevent token reuse across different MCP servers (confused deputy attacks).
- 53% of real-world MCP servers still use static API keys. Only 8.5% use OAuth. The gap between spec requirements and production reality is large.

### Handling Context Window Pressure

MCP tool definitions consume tokens. One server with 89 tools costs thousands of tokens before the model processes any user message. Teams have hit walls where three MCP servers consumed 143,000 of 200,000 available tokens.

Mitigation strategies (from research and practitioner experience):
1. **Semantic tool selection**: Use vector embeddings to dynamically load only the most relevant tools for the current query. Research shows 99.6% token reduction while maintaining 97.1% hit rate, retrieval under 91ms.
2. **Server specialization**: Instead of one mega-server, use multiple narrow servers. Connect only what's needed for the task.
3. **Tool grouping with action parameters**: Consolidate rarely-used tools into groups. Reduces schema count without losing capability.
4. **Progressive disclosure**: Tools that accept a `search` or `help` parameter allowing the model to discover sub-capabilities on demand.
5. **Schema compression**: Strip verbose descriptions from rarely-used parameters. Keep critical ones full.
6. **Lazy loading**: Don't connect all available servers on session start. Connect them when the task requires it.

### Observability

Production MCP deployments need instrumentation. Key tools:
- **Datadog LLM Observability**: Auto-instruments the MCP Python client, traces every session initialization, tool listing, and invocation as spans linked to the LLM trace.
- **OpenTelemetry**: Vendor-neutral. Treat MCP servers like any other service: spans for tool calls, metrics for per-tool latency/error rates/call volumes. Elastic APM and SigNoz have specific MCP tracing guides.
- Key alert thresholds: transport handshake success rate < 99% (critical), tool error rate > 5% (high), tool success rate < 80% (high).

---

## 5. Advanced Insights

### The Security Landscape is Genuinely Concerning

MCP's security story in production is immature. Key issues that practitioners argue about:

**Tool poisoning is a structural vulnerability (CVE-2025-54136)**. A malicious MCP server can embed instructions inside tool descriptions that the LLM reads but the user never sees. Example from Simon Willison's research: a simple-looking `add()` function instructs the model to "read ~/.cursor/mcp.json and pass its content as a sidenote" before any arithmetic is done. The user sees no indication of this. The attack only needs to happen once — unlike input-level prompt injection which must be repeated per session.

Unlike input prompt injection (which requires convincing the model on each use), tool poisoning embeds instructions permanently in the tool metadata loaded at session start.

**Rug pulls**: After installation, a tool's description can change without user notification. The model's behavior changes because its instructions changed, but the user sees the same UI.

**The "SHOULD" problem**: The MCP specification says implementations SHOULD display tool descriptions to users and SHOULD alert them when descriptions change. These are not MUST requirements. Most clients do not do this. Willison argues these SHOULDs should be treated as mandatory.

**Auth security research finding (June 2025)**: Researchers found hundreds of MCP servers bound to `0.0.0.0` without authentication. A study of real-world remote MCP servers found:
- CVE-2025-69898: PKCE validation bypass
- CVE-2025-61510: Unauthenticated server exposing 5,000+ enterprise records

**The Token Passthrough Anti-Pattern**: MCP servers accepting tokens issued to other services without validating audience claims, then forwarding them. Opens up bypassing audit logs and rate limiting.

**Practical threat mitigation** (from IMTI's "Build With It Anyway" analysis):
- For internal data platforms: VPN-only access, pre-registered IAM clients, read-only defaults — eliminates most vectors
- For public-facing: OAuth 2.1, strict scope validation, audit logging, consider MCP gateway with tool description validation
- For untrusted third-party servers: the threat model is essentially unsolvable today. Use with extreme caution.

### The Context Window Ceiling is a Real Architectural Constraint, Not a Tuning Problem

Microsoft Research found that with too many tools, performance degrades up to 85% for some models. One server returned an average of 557,766 tokens per call — exceeding GPT-5's context window. OpenAI itself recommends keeping tools "fewer than 20 functions at any one time."

The performance cliff is not linear. Models can handle 5-10 tools well, 10-20 acceptably, 20-50 with degraded accuracy, 50+ unreliably. This is a hard constraint that requires architectural responses (semantic routing, server specialization), not just larger context windows.

### MCP Is a Protocol, Not a Framework for Agent Reasoning

A nuance many developers miss: MCP defines how tools are exposed and called. It says nothing about agent reasoning loops, memory, planning, or how results are used. A host can call tools in whatever order it wants, with whatever context management strategy it chooses.

This is intentional per the design principles: "Composability over specificity" and "Capability over compensation." The protocol does not try to solve reasoning; it creates the plumbing that reasoning systems can use.

### The MCP vs. A2A Distinction

Google launched Agent-to-Agent (A2A) protocol in April 2025. It is now co-governed with MCP under the AAIF. They solve different problems:

- **MCP**: Single agent accessing tools/resources. Agent-to-tool relationship. Hierarchical (host→client→server).
- **A2A**: Multiple agents coordinating on tasks. Agent-to-agent peer relationship. Uses Agent Cards for capability discovery.

Production multi-agent systems use both: MCP for each agent's individual tool access, A2A for coordination between agents. A research agent uses MCP to call web search and database tools. An orchestrator uses A2A to delegate tasks to the research agent.

They are complementary, not competing. Both are governed by AAIF.

### The Tasks Primitive Changes the Architecture Fundamentally

The experimental Tasks primitive (SEP-1686) is the most significant architectural shift in the November 2025 spec. Without Tasks, all MCP tool calls are synchronous and blocking. Infrastructure kills idle connections after 60-300 seconds. Agents cannot parallelize long-running work.

With Tasks:
- `tools/call` with a `task` field returns a `taskId` immediately
- The agent can fire 100 tasks in milliseconds, then poll or receive webhooks
- Server-side state stores intermediate results, keeping them out of the context window
- Tasks can enter `input_required` state for human approval, bridging machine milliseconds and human timescales
- Production deployments need persistent backends (Redis, PostgreSQL) to survive server restarts

This is the feature that makes MCP viable for non-trivial agentic work. Without it, any operation taking more than a few seconds is unreliable.

### Expert Disagreements

**Is MCP ready for production?**

Victor Dibia (AI researcher): No, not yet. The developer experience is too rough (300+ LOC for basic examples), testing infrastructure is inadequate, and documentation assumes Claude Desktop rather than custom application builders. The protocol is the right direction, but current state does not justify the hype.

IMTI and practitioners who've shipped: Yes, with conditions. For internal data platforms with controlled server implementation, VPN-only access, and read-only defaults — it works well. The protocol flaws are real but survivable with the right constraints.

**Is the security model fixable?**

Simon Willison (pessimistic): After 2.5+ years of known prompt injection problems, there are still no universal mitigations. The fundamental problem of mixing private data, untrusted instructions, and exfiltration vectors has no clean solution. The SHOULD/MUST problem in the spec is a design flaw.

Security researchers (cautiously optimistic): The recent CVE-based research, the June 2025 spec update mandating Resource Indicators and classifying MCP servers as OAuth Resource Servers, and the emerging gateway ecosystem with real-time tool description validation are making real progress.

**Will the context window problem be solved by bigger windows?**

Microsoft Research (no): The 85% performance degradation with excessive tools is not just a context limit problem — it is a model attention and reasoning problem. Throwing more tokens at it degrades quality. The solution is architectural: semantic routing, server specialization, progressive discovery.

### Emerging Trends

- **MCP gateways as a new infrastructure category**: Products like Lasso, Envoy AI Gateway, Kong, and Azure APIM are adding MCP-specific capabilities (tool description validation, credential injection, multi-tenancy, audit trails). This is becoming as standard as API gateways.
- **`.well-known/mcp.json` discovery**: A proposed standard (unshipped as of mid-2026) that would allow programmatic discovery of an organization's MCP servers, similar to `robots.txt` or `openapi.json`.
- **MCP Apps**: An extension layer on top of core MCP enabling interactive applications that run inside AI clients. Still experimental.
- **Triggers and event-driven updates**: On the 2026 roadmap — servers pushing notifications when external events occur, rather than waiting for client polls.
- **Multi-tenancy**: The 2025 spec has no tenant isolation model. This is a priority for the 2026 roadmap for enterprise SaaS deployments.

---

## 6. Curated Reading List

### Primary Sources

**Official MCP Specification (2025-11-25)**
URL: https://modelcontextprotocol.io/specification/latest
Why read: The authoritative source. The MUST/MUST NOT language tells you what is required vs. optional, which matters for security. The architecture section maps the full primitive model.
Difficulty: Intermediate. Assumes familiarity with JSON-RPC and client-server architecture.
Time: 90 minutes for full spec, 20 minutes for architecture + primitives sections.
Key takeaway: The spec distinguishes server-controlled, application-controlled, and user-controlled primitives — a distinction most tutorials obscure.

**MCP Architecture Overview**
URL: https://modelcontextprotocol.io/docs/learn/architecture
Why read: Best single page for understanding host/client/server roles, transport mechanisms, and the initialization/capability negotiation lifecycle. Includes working JSON-RPC message examples.
Difficulty: Beginner.
Time: 25 minutes.
Key takeaway: Every connection is a stateful session with capability negotiation at the start. What you can do in a session is determined by what both sides declared they support.

**MCP Design Principles**
URL: https://modelcontextprotocol.io/community/design-principles
Why read: Essential for understanding *why* the protocol is minimal. "Composability over specificity" and "Stability over velocity" explain why MCP doesn't have built-in agent memory, doesn't handle reasoning loops, and moves slowly. This prevents you from filing issues asking for features MCP intentionally excludes.
Difficulty: Beginner.
Time: 15 minutes.
Key takeaway: "Adding to a protocol as widely adopted as MCP is easy. Removing from it is nearly impossible." Every addition is a permanent commitment.

### Engineering Blogs and Practitioner Write-Ups

**"We Built 9 Open Source MCP Servers — Here's What We Learned" (Yaw.sh)**
URL: https://yaw.sh/blog/we-built-9-open-source-mcp-servers/
Why read: The most candid practitioner retrospective available. Covers tool description craft, structured output discipline, actionable error handling, tool count tradeoffs, and compliance testing. All findings come from production code.
Difficulty: Intermediate.
Time: 20 minutes.
Key takeaway: Tool description quality determines agent behavior more than implementation quality. 89 tools in one server is too many — group related operations.

**"Why We Open-Sourced Our MCP Server" (GitHub Blog)**
URL: https://github.blog/open-source/maintainers/why-we-open-sourced-our-mcp-server-and-what-it-means-for-you/
Why read: Documents GitHub's production journey: the problem (LLMs hallucinating without live GitHub context), the solution (MCP as source-of-truth interface), and four real production use cases. Shows the standardization value in practice.
Difficulty: Beginner.
Time: 15 minutes.
Key takeaway: The three-layer separation (LLM, UX, data/tools) is "modular, testable, and swappable." Each layer evolves independently.

**"MCP Has Prompt Injection Security Problems" (Simon Willison)**
URL: https://simonwillison.net/2025/Apr/9/mcp-prompt-injection/
Why read: The most cited security analysis of MCP. Concrete attack examples, not theoretical. Willison demonstrates a working WhatsApp-MCP exploit. His critique of the SHOULD/MUST spec language is the clearest articulation of the fundamental security problem.
Difficulty: Intermediate.
Time: 20 minutes.
Key takeaway: Tool poisoning only needs to happen once — unlike input-level prompt injection. The fix requires MUST-level enforcement of user-visible tool descriptions and change alerts.

**"No, MCPs Have Not Won (Yet)" (Victor Dibia)**
URL: https://newsletter.victordibia.com/p/no-mcps-have-not-won-yet
Why read: Necessary counterweight to the hype. Dibia has built MCPs and identifies specific DX failures: 300+ LOC for basic examples, cryptic error messages, no good testing workflow, documentation assumes Claude Desktop. Tracks what needs to improve before broader adoption.
Difficulty: Beginner.
Time: 15 minutes.
Key takeaway: There is a gap between people who have *used* MCP in existing tools (impressed) and people who have *built* MCPs (frustrated). The latter's concerns are legitimate.

**"Tool-Space Interference in the MCP Era" (Microsoft Research)**
URL: https://www.microsoft.com/en-us/research/blog/tool-space-interference-in-the-mcp-era-designing-for-agent-compatibility-at-scale/
Why read: The most rigorous empirical study of what happens when you add too many tools. Performance degradation curves, naming collision data, error handling failures, and specific recommendations for server, client, and marketplace developers. Primary-source research, not opinion.
Difficulty: Advanced.
Time: 30 minutes.
Key takeaway: One server in the wild has 256 tools. One tool returned 557,766 tokens per call. Performance can degrade 85% with excessive tools. Flattening parameter nesting improved tool-calling accuracy by 47%.

**MCP vs Function Calling (Portkey)**
URL: https://portkey.ai/blog/mcp-vs-function-calling/
Why read: The clearest explanation of how MCP and function calling complement each other rather than compete. Includes the cross-provider format difference table (why function calling schemas differ between OpenAI, Anthropic, Gemini, Llama) and the hybrid production pattern.
Difficulty: Beginner.
Time: 15 minutes.
Key takeaway: Many production systems use function calling for model intent expression and MCP infrastructure for tool routing and execution. They solve different layers.

**"Your MCP Server Is Eating Your Context Window" (Apideck)**
URL: https://www.apideck.com/blog/mcp-server-eating-context-window-cli-alternative
Why read: Documents the Duet team's experience hitting the 143,000-token wall with three MCP servers. Presents the cost numbers (MCP costs 4-32x more tokens than CLI alternatives for identical tasks on some workloads) and proposes three alternative architectures. A useful forcing function to think about when MCP is the right choice.
Difficulty: Intermediate.
Time: 15 minutes.
Key takeaway: For solo developers building a coding agent with one or two integrations, direct CLI calls are cheaper and more reliable than MCP. MCP's value appears at scale.

**MCP One-Year Anniversary Post**
URL: https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/
Why read: Official reflection on adoption curve, ecosystem growth, and architectural evolution through three phases. Includes the "value is getting people to agree and do something" quote that captures MCP's actual value proposition.
Difficulty: Beginner.
Time: 15 minutes.
Key takeaway: Protocol success is a network coordination problem, not a technical quality problem. The value of standardization compounds with adoption.

**MCP 2026 Roadmap**
URL: https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/
Why read: Tells you where the gaps are today and what will be fixed. Stateless Streamable HTTP, Tasks primitive maturation, enterprise auth (SSO integration), and multi-tenancy are the priority areas. Reading this tells you what not to build custom solutions for (because the spec is handling it) vs. what you must handle yourself today.
Difficulty: Intermediate.
Time: 20 minutes.
Key takeaway: "Stability over velocity" means the roadmap is prioritized and measured. Enterprise readiness gaps will mostly land as extensions, not core spec changes.

### Research Papers

**"MCP Threat Modeling and Analyzing Vulnerabilities to Prompt Injection with Tool Poisoning"**
URL: https://arxiv.org/abs/2603.22489
Why read: First systematic security analysis applying STRIDE+DREAD to MCP. Identifies tool poisoning as the most significant client-side vulnerability. Proposes the static metadata analysis + behavioral anomaly detection defense layer.
Difficulty: Advanced.
Time: 45 minutes.
Key takeaway: Most MCP clients exhibit "insufficient static validation and parameter visibility." Prior research focused on server-side; this fills the client-side gap.

**"From Tool Orchestration to Code Execution: A Study of MCP Design Choices"**
URL: https://arxiv.org/pdf/2602.15945
Why read: Empirical analysis of design choices across the MCP ecosystem. Useful for understanding what patterns are actually being adopted in production vs. what the spec recommends.
Difficulty: Advanced.
Time: 45 minutes.
Key takeaway: Design choices in the wild frequently diverge from spec intent, particularly around error handling and schema design.

**"Semantic Tool Discovery for Large Language Models: A Vector-Based Approach to MCP Tool Selection"**
URL: https://arxiv.org/pdf/2603.20313
Why read: Rigorous evaluation of vector-based tool routing as a solution to the context window problem. 99.6% token reduction, 97.1% hit rate, under 91ms retrieval. This is the evidence base for semantic routing at scale.
Difficulty: Advanced.
Time: 30 minutes.
Key takeaway: Embedding-based tool selection is production-viable. The hit rate is high enough for most applications, and the token savings are dramatic.

### Official Resources

**Anthropic MCP Introduction Course**
URL: https://anthropic.skilljar.com/introduction-to-model-context-protocol
Why read: Canonical Python-focused course. Covers building both servers and clients from Anthropic, so it reflects how Claude's host application actually uses MCP.
Difficulty: Beginner.
Time: 2-3 hours.
Key takeaway: The Python SDK abstracts most of the JSON-RPC layer, but understanding the underlying messages is worth the investment.

**Microsoft MCP for Beginners (Open Source Curriculum)**
URL: https://github.com/microsoft/mcp-for-beginners/
Why read: Only curriculum covering .NET, Java, TypeScript, JavaScript, Rust, and Python. 13 hands-on labs covering schema design, auth, error handling, testing, and hosting. Microsoft's perspective on enterprise MCP patterns.
Difficulty: Beginner to Intermediate.
Time: 8-12 hours (full curriculum).
Key takeaway: Cross-language coverage reveals how much of MCP's value comes from the standard rather than any specific SDK.

---

## 7. Learning Path

### If You Have 30 Minutes

1. Read the MCP Architecture Overview (25 min): https://modelcontextprotocol.io/docs/learn/architecture
   Focus on: the three roles (host/client/server), the six primitives, and the JSON-RPC initialization exchange.
   Skip: the full JSON-RPC examples — skim them but don't work through them.

After 30 minutes you will understand: what MCP is, the mental model of the three participants, what tools/resources/prompts/sampling/elicitation mean, and why the primitive type matters for design.

### If You Have 2 Hours

Block 1 (30 min): MCP Architecture Overview (above).

Block 2 (20 min): MCP Design Principles: https://modelcontextprotocol.io/community/design-principles
Understand why MCP is deliberately minimal. This prevents wasted effort trying to use MCP for things it intentionally excludes.

Block 3 (20 min): Simon Willison on security: https://simonwillison.net/2025/Apr/9/mcp-prompt-injection/
Understand the threat model before building anything real.

Block 4 (20 min): Yaw.sh practitioner lessons: https://yaw.sh/blog/we-built-9-open-source-mcp-servers/
Ground-level production experience on what actually goes wrong.

Block 5 (20 min): Portkey on MCP vs Function Calling: https://portkey.ai/blog/mcp-vs-function-calling/
Clarifies the architecture stack and where MCP fits.

Block 6 (10 min): Scan the 2026 roadmap: https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/
Know what gaps exist today and what's being addressed.

After 2 hours you will understand: the full conceptual model, the security landscape, practitioner-validated design principles, and where MCP sits in the broader agent stack.

### If You Want to Become Highly Knowledgeable Over One Week

**Day 1: Foundations**
- Full MCP specification (90 min): https://modelcontextprotocol.io/specification/latest
- Architecture + Design Principles + Server/Client Concepts docs (60 min)
- One-year anniversary post (15 min): https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/

**Day 2: Build Your First Server**
- Anthropic intro course or Microsoft curriculum (2-3 hours)
- Goal: build a working local MCP server with at least one Tool, one Resource, and one Prompt. Use the Inspector tool to debug.

**Day 3: Security Depth**
- Simon Willison: https://simonwillison.net/2025/Apr/9/mcp-prompt-injection/
- Threat modeling paper: https://arxiv.org/abs/2603.22489
- Auth guide: OAuth 2.1 + PKCE for remote servers: https://auth0.com/blog/mcp-specs-update-all-about-auth/
- WorkOS enterprise guide: https://workos.com/blog/everything-your-team-needs-to-know-about-mcp-in-2026

**Day 4: Production Patterns**
- Microsoft Research tool-space study: https://www.microsoft.com/en-us/research/blog/tool-space-interference-in-the-mcp-era-designing-for-agent-compatibility-at-scale/
- Apideck context window problem: https://www.apideck.com/blog/mcp-server-eating-context-window-cli-alternative
- Semantic tool routing paper: https://arxiv.org/pdf/2603.20313
- Gateway patterns: https://chatforest.com/guides/mcp-gateway-proxy-patterns/

**Day 5: Ecosystem and Advanced Concepts**
- MCP vs A2A: https://auth0.com/blog/mcp-vs-a2a/
- MCP Tasks (async): https://stn1slv.medium.com/architecting-the-asynchronous-agent-a-guide-to-mcp-tasks-7348c6527233
- MCP Elicitation: https://glama.ai/blog/2025-09-03-elicitation-in-mcp-bridging-the-human-ai-gap
- Sampling: https://www.dailydoseofds.com/model-context-protocol-crash-course-part-5/

**Day 6: Critical Perspectives and Tradeoffs**
- Victor Dibia critique: https://newsletter.victordibia.com/p/no-mcps-have-not-won-yet
- IMTI balanced defense: https://imti.co/mcp-defense/
- Thoughtworks production impact: https://www.thoughtworks.com/en-us/insights/blog/generative-ai/model-context-protocol-mcp-impact-2025
- Everything That Is Wrong with MCP (transport critique): https://mitek99.medium.com/mcps-overengineered-transport-and-protocol-design-f2e70bbbca62

**Day 7: Build Something Real**
- Build a server for a real integration you need (a database, an API, an internal service)
- Add observability: instrument with OpenTelemetry
- Add authentication: implement OAuth 2.1 if building a remote server
- Test against multiple clients (Claude Desktop, Cursor, VS Code)
- Run against the mcp-compliance test harness

---

## 8. Practical Application

### For Dalgo (NGO Data Platform Context)

MCP is directly applicable to Dalgo's data stack in several concrete ways:

**Data warehouse access via Resources and Tools**: Expose warehouse tables and schema as MCP Resources (application-controlled, read-only context). Expose query execution as an MCP Tool (model-invoked, with appropriate guardrails). This lets AI agents reason about NGO data without the AI application needing to understand dbt, Airbyte, or Superset internals.

**Pipeline management via Tools**: Expose pipeline trigger, status, and run history as MCP Tools. An NGO data coordinator asking "why did last night's pipeline fail?" in natural language could get a complete answer without navigating the Prefect UI, because the agent calls `get_pipeline_run_history` and `get_flow_run_logs` via MCP.

**Human-in-the-loop for destructive operations**: Use Elicitation before any write operation (schema changes, pipeline deletion, configuration updates). The agent pauses, presents a structured approval request, and resumes only after confirmation. Critical for NGOs where data mistakes have real-world harm.

**Dashboard and chart generation via Tools**: The `mcp__dalgo__dalgo_create_chart`, `dalgo_create_dashboard`, etc. tools that exist in this codebase's MCP server are the right pattern. Keep them narrow (one tool per action), write descriptions for the model (not humans), and return structured JSON.

**Context on NGO users**: Resources exposing documentation, data dictionaries, and field-level metadata give the model the context it needs to generate appropriate queries and interpretations without requiring the NGO user to explain their data structure each time.

### For Agent System Design Generally

**When to choose MCP over direct API calls**:
- You need the same integration to work across multiple AI clients (Claude, GPT, Gemini, your own host)
- More than 5-10 tools that multiple teams will reuse
- You want tool implementations to update independently of the AI application
- You need centralized audit logging of what the AI is doing
- The integration requires authentication that should not be exposed to the model

**When direct function calling is sufficient**:
- Single AI provider, single application, fewer than 5 tools
- Latency-critical (MCP adds process/network hops)
- Prototyping (MCP adds infrastructure overhead)
- You own the full stack and standardization adds no value

**Context engineering with MCP**:
- Use Resources for large, stable context (documentation, schemas, data dictionaries). Load them application-side when relevant, not model-side on every turn.
- Use Tools only for actions and dynamic queries the model needs to initiate.
- Write a tool selection preamble in your system prompt explaining how to choose between tools when multiple are similar.
- Monitor which tools are called most (Datadog / OpenTelemetry) — usage data tells you what your model is actually doing vs. what you expected.

**Guardrails at the MCP layer**:
- MCP gateways (Lasso, Envoy, Kong) can validate tool call arguments before execution, rate-limit expensive operations, and scan for prompt injection in tool descriptions.
- Scope tools to read-only by default. Explicitly declare write tools and make the risk apparent in the description.
- Use separate MCP servers for different permission levels. A server with read-only warehouse access should never be in the same server as one with pipeline deletion capabilities.
- Log every `tools/call` request and response. MCP does not do this for you.

**Production agent architecture with MCP**:

```
User → AI Host (Claude/GPT/custom)
         ↓
    MCP Gateway (auth, audit, guardrails)
         ↓
   ┌─────────────────────────┐
   │ MCP Server A: Warehouse │ (read-only, Resources + Tools)
   │ MCP Server B: Pipelines │ (read + write, Tools + Elicitation)
   │ MCP Server C: Docs      │ (read-only, Resources)
   └─────────────────────────┘
         ↓
  A2A (if multi-agent)
```

Start narrow: one server, read-only, no auth complexity. Add write operations behind Elicitation gates. Add auth when going remote. Add gateways when you have multiple servers or teams.

### Evaluating MCP Servers Before Using Them

Given the security landscape, before installing any third-party MCP server:
1. Inspect the source code if available. Look at every tool description for instructions that shouldn't be there.
2. Run it against the mcp-compliance test harness to check spec conformance.
3. Check the registry listing for maintenance status (last commit, issue response time).
4. Run in a sandboxed environment without access to sensitive credentials.
5. Review what permissions it requests and whether they match the stated purpose.
6. Watch for tool description changes between versions (rug pull risk).

---

## 9. Research Quality Notes and Source Index

This report draws on primary sources fetched directly. All claims are traceable.

**Official Sources**
- MCP Specification: https://modelcontextprotocol.io/specification/latest
- MCP Architecture: https://modelcontextprotocol.io/docs/learn/architecture
- MCP Design Principles: https://modelcontextprotocol.io/community/design-principles
- MCP Blog (One Year): https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/
- MCP Blog (2026 Roadmap): https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/

**Security**
- Simon Willison on prompt injection: https://simonwillison.net/2025/Apr/9/mcp-prompt-injection/
- Threat modeling paper (arXiv): https://arxiv.org/abs/2603.22489
- MCP tool poisoning CVE: https://www.truefoundry.com/blog/blog-mcp-tool-poisoning-gateway-defense
- Auth security research: https://arxiv.org/html/2605.22333v1
- Auth0 spec update (OAuth): https://auth0.com/blog/mcp-specs-update-all-about-auth/

**Production Practitioners**
- Yaw.sh 9 servers: https://yaw.sh/blog/we-built-9-open-source-mcp-servers/
- GitHub open source post: https://github.blog/open-source/maintainers/why-we-open-sourced-our-mcp-server-and-what-it-means-for-you/
- Thoughtworks: https://www.thoughtworks.com/en-us/insights/blog/generative-ai/model-context-protocol-mcp-impact-2025

**Performance and Architecture**
- Microsoft Research tool-space: https://www.microsoft.com/en-us/research/blog/tool-space-interference-in-the-mcp-era-designing-for-agent-compatibility-at-scale/
- Context window problem: https://www.apideck.com/blog/mcp-server-eating-context-window-cli-alternative
- Semantic tool routing: https://arxiv.org/pdf/2603.20313
- Gateway patterns: https://chatforest.com/guides/mcp-gateway-proxy-patterns/

**Critical Perspectives**
- Victor Dibia critique: https://newsletter.victordibia.com/p/no-mcps-have-not-won-yet
- IMTI balanced defense: https://imti.co/mcp-defense/
- MCP enterprise gaps: https://workos.com/blog/everything-your-team-needs-to-know-about-mcp-in-2026

**Ecosystem**
- MCP vs A2A: https://auth0.com/blog/mcp-vs-a2a/
- MCP Tasks async: https://stn1slv.medium.com/architecting-the-asynchronous-agent-a-guide-to-mcp-tasks-7348c6527233
- MCP vs Function Calling: https://portkey.ai/blog/mcp-vs-function-calling/
- Observability: https://www.datadoghq.com/blog/mcp-client-monitoring/
- Vercel perspective: https://vercel.com/blog/model-context-protocol-mcp-explained
- Cloudflare implementation: https://developers.cloudflare.com/agents/model-context-protocol/

**Conflicting Viewpoints Noted**
- Security readiness: Willison (pessimistic) vs. security researchers and spec maintainers (cautiously optimistic)
- Production readiness: Dibia (not ready) vs. IMTI and practitioners who've shipped (ready with constraints)
- Context window: The ceiling is structural (Microsoft Research) vs. solvable via larger context windows (some practitioners)
- Governance: AAIF/Linux Foundation resolves lock-in concerns vs. skeptics who note Anthropic still has disproportionate maintainer presence

---

*Report compiled from primary sources. Fetch dates: June 2026. Specification version: 2025-11-25 (stable), 2026-07-28 (release candidate as of research date).*
