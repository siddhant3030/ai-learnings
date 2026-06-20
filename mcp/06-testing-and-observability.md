# Testing, Observability & Maintenance for Production MCP Servers

A practical guide for engineers who have built an MCP server and now need to **test it, watch it in production, and keep it healthy over time**. The hard truth about MCP servers is that your real "user" is not a human — it's an LLM agent. A normal API breaks loudly with a 4xx/5xx; a broken MCP tool breaks *silently*, because the agent just hallucinates a workaround ([Nordic APIs](https://nordicapis.com/the-weak-point-in-mcp-nobodys-talking-about-api-versioning/)). That single fact reshapes how you test, monitor, and version everything below.

---

## 1. Testing an MCP Server

Think of testing in three layers — a test pyramid for tool servers ([Anil Goyal, "Three-Layer Test Pyramid"](https://medium.com/@anil.goyal0057/the-complete-guide-to-testing-mcp-server-applications-a-three-layer-test-pyramid-for-ai-powered-027e941be6d4)):

| Layer | What it checks | Tool |
|-------|----------------|------|
| Unit | Each handler's logic, schema, error paths | pytest / vitest + in-memory transport |
| Integration | Real MCP protocol over a real client/transport | SDK `Client` + Inspector CLI |
| Eval | Does an *agent* actually pick & call the tool right? | Arcade Evals, MCPJam, MCP-Eval |

### 1.1 The MCP Inspector (official tool)

The [MCP Inspector](https://github.com/modelcontextprotocol/inspector) is Anthropic's official visual testing/debugging tool. It has two parts: a React web UI (the **MCPI** client) and a Node proxy (**MCPP**) that bridges the browser to your server over stdio, SSE, or Streamable HTTP ([Stainless](https://www.stainless.com/mcp/mcp-inspector-testing-and-debugging-mcp-servers/)).

```bash
# Launch the UI against a local stdio server (UI at http://localhost:6274)
npx @modelcontextprotocol/inspector node build/index.js

# Pass env vars and args through to your server
npx @modelcontextprotocol/inspector -e API_KEY=$API_KEY node build/index.js arg1

# Point at a remote HTTP server
npx @modelcontextprotocol/inspector --cli https://my-server.example.com --transport http
```

The UI gives you panels for **Tools, Resources, Prompts, Request History, and Notifications**. Pick a tool, fill in a form auto-generated from its JSON schema, click **Call**, and inspect the exact JSON response — including the discovery handshake and color-coded protocol logs ([MCPcat](https://mcpcat.io/guides/setting-up-mcp-inspector-server-testing/)).

**CLI mode is the part that matters for CI.** It turns the Inspector into a scriptable client — perfect for smoke tests in a pipeline:

```bash
# List tools (assert the catalog hasn't silently changed)
npx @modelcontextprotocol/inspector --cli node build/index.js --method tools/list

# Call one tool with args and capture the JSON for assertions
npx @modelcontextprotocol/inspector --cli node build/index.js \
  --method tools/call --tool-name get_pipeline --tool-arg id=42
```

Auth is supported: the proxy prints a session token on startup, and for SSE/HTTP servers you can supply a Bearer token in the Authorization header field ([Auth0](https://auth0.com/ai/docs/mcp/guides/test-your-mcp-server-with-mcp-inspector)). Config lives in a familiar `mcpServers` block:

```json
{
  "mcpServers": {
    "dalgo": {
      "command": "node",
      "args": ["build/index.js"],
      "env": { "API_KEY": "your-key" }
    }
  }
}
```

### 1.2 Unit-testing tools / handlers

Don't spawn a subprocess for unit tests. Both official SDKs ship an **in-memory transport** that wires a test `Client` straight to your `Server` instance in the same process — the real MCP protocol, zero network, and you can set breakpoints ([MCPcat unit testing](https://mcpcat.io/guides/writing-unit-tests-mcp-servers/)).

Python (pytest, FastMCP):

```python
import pytest
from fastmcp import Client   # in-memory: pass the server object directly

@pytest.mark.asyncio
async def test_get_pipeline_returns_schema(mcp_server):
    async with Client(mcp_server) as client:          # no subprocess
        result = await client.call_tool("get_pipeline", {"id": 42})
        assert result.data["status"] in {"running", "completed", "failed"}

@pytest.mark.asyncio
async def test_get_pipeline_rejects_bad_id(mcp_server):
    async with Client(mcp_server) as client:
        with pytest.raises(Exception):
            await client.call_tool("get_pipeline", {"id": "not-an-int"})
```

(FastMCP documents this in-memory pattern as the recommended default for tests — [gofastmcp.com/development/tests](https://gofastmcp.com/development/tests).)

TypeScript (vitest) uses `InMemoryTransport` from the official SDK:

```ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { InMemoryTransport } from "@modelcontextprotocol/sdk/inMemory.js";

const [clientT, serverT] = InMemoryTransport.createLinkedPair();
await server.connect(serverT);
await client.connect(clientT);

const res = await client.callTool({ name: "get_pipeline", arguments: { id: 42 } });
expect(res.content[0].type).toBe("text");
```

What to assert at this layer ([DEV — 4 patterns](https://dev.to/klement_gunndu/your-mcp-server-has-no-tests-here-are-4-patterns-to-fix-that-2k59)):
- **Schema correctness** — `tools/list` returns the inputSchema you expect (a regression guard against accidental "rug pulls", §4).
- **Happy path** — valid args produce well-formed content.
- **Error paths** — invalid/missing args, downstream failure, auth failure all return clean MCP errors rather than crashing the transport.

### 1.3 Integration testing against a real client

Integration tests use the SDK's `Client` over a **real transport** (stdio/HTTP) against a running server, or the Inspector CLI in a CI job. This catches what unit tests miss: transport framing, the `initialize` handshake, env/secret wiring, and serialization of large payloads. A simple pattern: start the server, run `tools/list` + a representative `tools/call` for each tool, diff the output against a golden file.

### 1.4 The hard part — testing that the *agent* uses your tools correctly (evals)

Unit + integration prove the tool *works when called correctly*. They say nothing about whether a real model **chooses** your tool and **fills the arguments** right. That requires **evals** — and this is where most production failures actually live.

Scope an MCP eval to the **schema-understanding layer**: given a user request and your tool catalog (as JSON schemas), does the model emit the right tool call with the right args? Don't execute the tool — that removes side effects and API cost and isolates the thing you're testing ([Arcade Evals](https://www.arcade.dev/blog/evaluate-mcp-tools/)).

An Arcade-style eval case has four parts:

```python
# pseudocode after the Arcade Evals structure
case(
  user_message="Show me the last run of the donor-import pipeline",
  expected_tool_calls=[("get_pipeline_run_history", {"name": "donor-import", "limit": 1})],
  critics=[
    BinaryCritic(field="name", weight=FuzzyWeight.CRITICAL),      # exact match
    SimilarityCritic(field="query", weight=FuzzyWeight.MEDIUM),   # fuzzy match
  ],
  rubric=Rubric(fail_threshold=0.8, warn_threshold=0.9),
)
```

Track these metrics across a suite, ideally across multiple models ([MCPJam](https://www.mcpjam.com/blog/mcp-evals)):

| Metric | Question it answers |
|--------|---------------------|
| **Tool-selection accuracy** | % of cases where the model called the *right* tool |
| **Parameter / argument correctness** | Are required args present and correctly valued? |
| **TPR (true-positive rate)** | How *discoverable* is a tool — does the agent reach for it when it should? |
| **FPR (false-positive rate)** | How often is a tool called when it shouldn't be? High FPR ⇒ description too generic |
| **End-to-end task success** | Did the full multi-step task complete? (MCP-Eval grades the *final outcome*, allowing for agent self-correction — [MCP-AgentBench](https://arxiv.org/pdf/2509.09734)) |

This is not academic. One team raised tool-selection success from **60% → 100% with prompt/description changes only, no code change**, driven entirely by evals ([Neon / ZenML](https://www.zenml.io/llmops-database/implementing-evaluation-framework-for-mcp-server-tool-selection)). Treat the eval suite as your real regression test for behavior.

### 1.5 Testing tool *descriptions* (the description is the API)

For an agent, the **description string is the contract** — change it and behavior changes, even with identical code. Run the same eval suite across description variants and compare TPR/FPR. Anthropic's own MCP evals show how much the *presentation* of tools matters: deferring/searching tool schemas instead of dumping them all up front lifted accuracy on **Claude Opus 4 from 49% → 74%**, and **Opus 4.5 from 79.5% → 88.1%**, while cutting tool-definition tokens **~85%** — because large catalogs cause "decision paralysis" ([MarkTechPost / Anthropic evals](https://www.marktechpost.com/2026/05/29/hermes-agent-ships-tool-search-for-mcp-anthropic-evals-show-49-to-74-accuracy-gain-on-opus-4/)). The lesson: a worse description, or one tool too many, silently degrades selection accuracy — so guard descriptions with evals the same way you guard code with unit tests.

---

## 2. Observability in Production

You cannot debug what you cannot see, and MCP failures are invisible by default — the spec doesn't require any audit logging at all ([Rafter](https://rafter.so/blog/mcp-audit-logging-problems)).

### 2.1 What to instrument

| Signal | Why it matters |
|--------|----------------|
| **Tool-call volume** (per tool) | Find dead tools (see §2.4) and load hot-spots |
| **Latency per tool** (p50/p95/p99) | Slow tools blow up agent loop wall-time and token cost |
| **Error rate per tool** | A tool failing 30% of the time teaches the agent to *avoid* it |
| **Token cost of responses** | Verbose tool output is a silent budget leak (§3) |
| **Tool selection mix** | *Which tools agents actually call vs. ignore* — the single most actionable signal |
| **Argument distributions** | Detect misuse / hallucinated params reaching your tool |
| **Identity** (user + agent + model) | Required for attribution, abuse detection, and cost allocation |

### 2.2 Tracing tool calls — OpenTelemetry GenAI conventions

The emerging standard is the **OpenTelemetry GenAI semantic conventions** (still *Development* status as of mid-2026, but already adopted by Datadog, Honeycomb, New Relic, LangChain, CrewAI — [Greptime](https://greptime.com/blogs/2026-05-09-opentelemetry-genai-semantic-conventions)). The relevant span for an MCP tool execution uses these exact attribute names ([OTel GenAI spans](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/)):

| Attribute / field | Value |
|-------------------|-------|
| span name | `execute_tool {gen_ai.tool.name}` (since v1.41) |
| `gen_ai.operation.name` | `execute_tool` |
| `gen_ai.tool.name` | e.g. `get_pipeline` |
| `gen_ai.tool.call.id` | correlates LLM tool-call ↔ execution |
| `gen_ai.tool.type` | `function` (client-generated args) or `extension` (agent-side API call) |
| `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` | token accounting |
| `error.type` | on failure |
| span kind | `INTERNAL` |

**Crucially, `gen_ai.tool.call.arguments` and `gen_ai.tool.call.result` are recorded only when privacy policy permits** — content capture is opt-in by design ([OTel](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/)). Honor that default (see §2.3).

### 2.3 Distributed tracing across client → server (Langfuse)

By default the MCP **client** and **server** emit *separate* traces, so you lose the link between "agent decided to call X" and "server executed X." [Langfuse's MCP tracing](https://langfuse.com/docs/observability/features/mcp-tracing) stitches them into one trace using MCP's `_meta` field to carry W3C Trace Context:

1. **Extract** the active trace context on the client.
2. **Inject** it into the tool call's `_meta` field.
3. **Restore** it on the server so all server spans inherit the client's trace.

The result is one connected trace: agent reasoning → tool discovery → tool execution → downstream API latency ([Langfuse integration](https://langfuse.com/integrations/other/mcp-use), [example repo](https://github.com/langfuse/langfuse-examples/tree/main/applications/mcp-tracing)). This is the view you want when an agent task fails three hops deep.

### 2.4 Logging safely (no PII / secret leakage)

Tool inputs and outputs routinely carry credentials, API keys, and PII — and the spec gives no guidance on protecting them ([Rafter](https://rafter.so/blog/mcp-audit-logging-problems)). Rules:

- **Redact argument values by default; opt-in per field.** Never let API keys, tokens, or customer identifiers hit the log stream ([digitalapplied 75-point audit](https://www.digitalapplied.com/blog/mcp-server-security-audit-75-point-checklist-2026)).
- **Store full tool-response bodies in a separate, access-controlled store**, not in the operational log stream.
- **Log structured metadata** (user id, agent id, model, tool, latency, cost, redacted args) as JSON for Grafana/Splunk/Datadog ingestion.
- For high-sensitivity data, run **PII detection + redaction at a gateway before the server ever sees raw values**, restoring on the way back (zero-trust) ([Tonic Textual](https://www.tonic.ai/blog/announcing-tonic-textual-mcp-server), [Enkrypt gateway](https://www.mintlify.com/enkryptai/secure-mcp-gateway/security/pii-handling)).

### 2.5 Detecting the "too many tools" / unused-tool problem from telemetry

Your **tool selection mix** telemetry is the detector. A tool with near-zero call volume across thousands of sessions is dead weight — and dead weight is expensive, because every unused tool's schema still sits in the prompt. When an agent has dozens of tools, schemas can eat **40–50% of the context window**, making a single response cost **2–3× more** ([Lunar.dev](https://www.lunar.dev/post/why-is-there-mcp-tool-overload-and-how-to-solve-it-for-your-ai-agents)). One real deployment measured **34 tools across 5 servers = ~22,000 tokens/turn, ~50% of the prompt** ([MarkTechPost](https://www.marktechpost.com/2026/05/29/hermes-agent-ships-tool-search-for-mcp-anthropic-evals-show-49-to-74-accuracy-gain-on-opus-4/)). Worse, bloated context causes **context rot** — measured degradation in reasoning as input grows, across 18 models including Claude Sonnet 4 and GPT-4.1 ([Chroma Context Rot, via](https://medium.com/@pankaj_pandey/when-too-many-tools-become-too-much-context-a-deep-dive-into-rag-mcp-9b628c8476d3)). Use telemetry to find the unused tail and cut it (§4.3).

---

## 3. Cost & Performance Monitoring

### 3.1 Token cost of tool definitions and responses

Two distinct costs, both billed every turn:

- **Definition cost** — schemas are sent on *every* request. The "MCP Tools Tax" runs **15,000–60,000 tokens/turn**; one pre-optimization measurement hit **134,000 tokens** ([MarkTechPost](https://www.marktechpost.com/2026/05/29/hermes-agent-ships-tool-search-for-mcp-anthropic-evals-show-49-to-74-accuracy-gain-on-opus-4/)). Deferred/on-demand schema loading (Claude Code, Cursor) cut total agent tokens by **~46.9%** ([DEV / Stacklok](https://medium.com/@pankaj_pandey/when-too-many-tools-become-too-much-context-a-deep-dive-into-rag-mcp-9b628c8476d3)).
- **Response cost** — a chatty tool that returns a 5,000-token blob when the agent needed one field pays that cost on *this* turn and again as history on every later turn. Instrument output token size per tool and trim.

### 3.2 Circuit breakers / runaway-loop protection

The cautionary tale: in November 2025 a market-research pipeline of four LangChain agents coordinating over the **A2A** protocol entered an infinite loop — an Analyzer and a Verifier ping-ponging requests with no termination condition. It ran **264 hours (11 days)** and cost **~$47,000**, discovered only from the billing statement ([DEV — The $47,000 Agent Loop](https://dev.to/waxell/the-47000-agent-loop-why-token-budget-alerts-arent-budget-enforcement-389i)). *(It was an A2A multi-agent loop, not an MCP-specific outage — but the failure mode applies directly to any agent calling MCP tools in a loop.)*

The key lesson is **monitoring is not enforcement**: "alerts are asynchronous — they notify someone who then has to act." By the time a human reads the alert, the spend has already happened. You need **synchronous, infrastructure-layer enforcement** that blocks the *next* call:

```text
# Enforce BEFORE each call, outside the agent's reasoning context
for each LLM/tool call:
    if session_tokens + estimate(next_call) > session_budget:
        terminate_session()          # hard stop, agent cannot override
    else:
        proceed()
```

Layered controls ([Waxell circuit breakers](https://dev.to/waxell/ai-agent-circuit-breakers-the-reliability-pattern-production-teams-are-missing-5bpg)):
- **Per-session token/cost ceiling** — auto-terminate on breach.
- **Loop detection** — trip if the same tool+args repeats N times, or the same agent pair ping-pongs.
- **Per-agent fleet ceiling** — aggregate cap catches "1,000 sessions at 100× normal cost" while normal traffic flows.
- **Rate limiting** at the gateway to prevent API exhaustion ([TrueFoundry](https://www.truefoundry.com/blog/rate-limiting-ai-agents-preventing-llm-api-exhaustion)).

Note the cost shape: a looping session is **O(n²)**, not O(n), because each step re-sends full history — so early per-call costs badly underpredict total spend.

### 3.3 Caching tool results

MCP has **first-class caching** in the draft spec via a `ttlMs` hint on `tools/call`, `*/list`, and resource reads — semantics like HTTP `Cache-Control: max-age`. `ttlMs: 0` ⇒ immediately stale; positive ⇒ client may treat fresh for that many ms ([MCP spec — caching](https://modelcontextprotocol.io/specification/draft/server/utilities/caching)). Guidance:

- **Cache** read-only, deterministic results: DB queries, document fetches, search, stable API responses (80–95% latency cut on repeat calls — [Fast.io](https://fast.io/resources/mcp-server-caching/)).
- **Never cache** writes (create/update/delete), non-deterministic output, or anything needing real-time freshness.
- **Invalidate** with a mix of TTL expiry, event-based busting on related writes, and versioned cache keys ([ChatForest](https://chatforest.com/guides/mcp-caching-strategies/)).

---

## 4. Maintenance & Evolution

### 4.1 Versioning without breaking agents — the "rug pull" risk

The defining maintenance hazard: you tweak a tool's name, a parameter, or even its *description*, and every agent built around the old behavior silently misfires — no error, just hallucinated workarounds ([Nordic APIs](https://nordicapis.com/the-weak-point-in-mcp-nobodys-talking-about-api-versioning/)). MCP has **no standardized tool-versioning layer** yet ([SEP-986/1575 still open](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1915)).

The strongest mitigation is to **treat tool definitions as immutable for a given version identifier** and enforce versioning at the connection level — abandon rolling releases for tools that live agents depend on ([Silent Breakage](https://minherz.medium.com/the-silent-breakage-a-versioning-strategy-for-production-ready-mcp-tools-fbb998e3f71f)). Classify every change:

| Severity | Examples | Action |
|----------|----------|--------|
| **Breaking** | removed tool, new *required* param, renamed param, changed semantics | New tool/version; keep old + deprecate |
| **Warning** | description change, behavior nuance | Re-run evals before shipping |
| **Safe** | new optional param, additive tool | Ship |

And because a description change is invisible, **diff your tool manifest in CI** and alert on any drift — your descriptions can change overnight and nobody notices ([Lukas Kania](https://medium.com/@binarEx/your-mcp-servers-tool-descriptions-changed-last-night-nobody-noticed-e3ad93cf6bc7)).

### 4.2 Deprecating tools gracefully

Don't delete — **deprecate**. Mark the old tool deprecated in the manifest, record the version deprecation started, and ship the new tool alongside so agents/clients can fall back or warn rather than break ([Evolvable MCP](https://medium.com/@kumaran.isk/evolvable-mcp-a-guide-to-mcp-tool-versioning-ae9a612f7710)). Watch the deprecated tool's call-volume telemetry; remove it only after it trends to zero.

### 4.3 Consolidating / removing tools — "few tools beat many"

§2.5 found the unused tail; this is where you act. Fewer, well-scoped tools measurably beat many (Anthropic's evals, §1.5). Periodically: drop zero-call tools, **merge near-duplicates** (two tools with overlapping descriptions inflate FPR), and consider on-demand tool search instead of exposing the full catalog. Re-run the eval suite after every consolidation to confirm selection accuracy went *up*, not down.

### 4.4 Capability negotiation & protocol version upgrades

Every MCP session opens with an `initialize` handshake: the client sends a proposed `protocolVersion` (e.g. `"2025-06-18"`) plus its `capabilities`; the server replies with its own version and capabilities. The effective feature set is the **intersection** — `C_eff = C_client ∩ C_server` — both sides must support a capability for it to turn on; no functional request (list tools, read resources) is legal until the handshake completes ([APXML](https://apxml.com/courses/getting-started-model-context-protocol/chapter-1-architecture-and-fundamentals/capabilities-negotiation), [MCP architecture](https://modelcontextprotocol.io/docs/learn/architecture)). For maintenance this means: **support a range of protocol versions, negotiate down gracefully** when a client is older, and gate new features behind capability flags so old clients keep working instead of erroring. The MCP spec ships dated revisions and continues to evolve (a major spec update landed mid-2026 — [WorkOS](https://workos.com/blog/mcp-2026-spec-agent-authentication)); test against the versions your real clients send, not just the latest.

---

## 5. Production Readiness Checklist

Copy-usable. Aim for every box ticked before you call an MCP server "production."

**Testing**
- [ ] Unit tests per tool via in-memory transport (happy path + error paths + schema)
- [ ] Integration test runs `tools/list` + a representative `tools/call` for every tool against a real client
- [ ] Inspector CLI smoke test wired into CI (`--method tools/list`, key `tools/call`s)
- [ ] Eval suite: tool-selection accuracy, argument correctness, TPR, FPR, end-to-end success
- [ ] Evals run across the model(s) you actually deploy on
- [ ] Tool descriptions covered by evals; description changes re-run evals before merge

**Observability**
- [ ] Per-tool: call volume, latency (p50/p95/p99), error rate, response token size
- [ ] Tool selection mix tracked (called vs. ignored) to find dead tools
- [ ] OpenTelemetry GenAI spans emitted (`execute_tool`, `gen_ai.tool.name`, `gen_ai.tool.call.id`)
- [ ] Client↔server traces linked (W3C context via `_meta`, e.g. Langfuse)
- [ ] Logs redact args by default; secrets/PII never in the log stream
- [ ] Response bodies in a separate access-controlled store, not operational logs
- [ ] Structured JSON logs export to your APM (Grafana/Datadog/Splunk)

**Cost & performance**
- [ ] Tool-definition token cost measured; on-demand/deferred loading if catalog is large
- [ ] **Synchronous** per-session token/cost ceiling enforced (not just alerts)
- [ ] Loop/circuit-breaker: trip on repeated identical calls or per-agent fleet ceiling
- [ ] Gateway rate limiting in place
- [ ] Caching with `ttlMs` for read-only deterministic tools; writes never cached
- [ ] Cache invalidation strategy (TTL + event-based + versioned keys)

**Maintenance & evolution**
- [ ] Tool manifest diffed in CI; alert on any name/param/description drift
- [ ] Tool definitions immutable per version; breaking changes ship as new tools
- [ ] Deprecation path (manifest flag + version) instead of deletion
- [ ] Periodic review consolidates/removes zero-call and duplicate tools, re-validated by evals
- [ ] Protocol-version range supported; capabilities negotiated and feature-gated for older clients

---

### Sources
- MCP Inspector: [GitHub](https://github.com/modelcontextprotocol/inspector) · [Stainless](https://www.stainless.com/mcp/mcp-inspector-testing-and-debugging-mcp-servers/) · [MCPcat setup](https://mcpcat.io/guides/setting-up-mcp-inspector-server-testing/) · [Auth0](https://auth0.com/ai/docs/mcp/guides/test-your-mcp-server-with-mcp-inspector)
- Testing layers / unit tests: [Three-layer pyramid](https://medium.com/@anil.goyal0057/the-complete-guide-to-testing-mcp-server-applications-a-three-layer-test-pyramid-for-ai-powered-027e941be6d4) · [MCPcat unit tests](https://mcpcat.io/guides/writing-unit-tests-mcp-servers/) · [FastMCP tests](https://gofastmcp.com/development/tests) · [4 patterns](https://dev.to/klement_gunndu/your-mcp-server-has-no-tests-here-are-4-patterns-to-fix-that-2k59)
- Evals: [Arcade](https://www.arcade.dev/blog/evaluate-mcp-tools/) · [MCPJam](https://www.mcpjam.com/blog/mcp-evals) · [Neon/ZenML](https://www.zenml.io/llmops-database/implementing-evaluation-framework-for-mcp-server-tool-selection) · [MCP-AgentBench](https://arxiv.org/pdf/2509.09734) · [Anthropic Tool Search evals](https://www.marktechpost.com/2026/05/29/hermes-agent-ships-tool-search-for-mcp-anthropic-evals-show-49-to-74-accuracy-gain-on-opus-4/)
- Observability: [OTel GenAI spans](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-spans/) · [Greptime](https://greptime.com/blogs/2026-05-09-opentelemetry-genai-semantic-conventions) · [Langfuse MCP tracing](https://langfuse.com/docs/observability/features/mcp-tracing) · [Rafter audit logging](https://rafter.so/blog/mcp-audit-logging-problems) · [Tonic Textual](https://www.tonic.ai/blog/announcing-tonic-textual-mcp-server) · [digitalapplied audit](https://www.digitalapplied.com/blog/mcp-server-security-audit-75-point-checklist-2026)
- Cost / loops / caching: [$47k loop](https://dev.to/waxell/the-47000-agent-loop-why-token-budget-alerts-arent-budget-enforcement-389i) · [Circuit breakers](https://dev.to/waxell/ai-agent-circuit-breakers-the-reliability-pattern-production-teams-are-missing-5bpg) · [MCP caching spec](https://modelcontextprotocol.io/specification/draft/server/utilities/caching) · [Fast.io caching](https://fast.io/resources/mcp-server-caching/) · [too-many-tools](https://www.lunar.dev/post/why-is-there-mcp-tool-overload-and-how-to-solve-it-for-your-ai-agents)
- Versioning / evolution: [Silent Breakage](https://minherz.medium.com/the-silent-breakage-a-versioning-strategy-for-production-ready-mcp-tools-fbb998e3f71f) · [Evolvable MCP](https://medium.com/@kumaran.isk/evolvable-mcp-a-guide-to-mcp-tool-versioning-ae9a612f7710) · [Nordic APIs](https://nordicapis.com/the-weak-point-in-mcp-nobodys-talking-about-api-versioning/) · [Descriptions changed overnight](https://medium.com/@binarEx/your-mcp-servers-tool-descriptions-changed-last-night-nobody-noticed-e3ad93cf6bc7) · [Capability negotiation](https://apxml.com/courses/getting-started-model-context-protocol/chapter-1-architecture-and-fundamentals/capabilities-negotiation) · [MCP architecture](https://modelcontextprotocol.io/docs/learn/architecture)
