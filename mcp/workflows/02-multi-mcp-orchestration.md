# Multi-MCP Orchestration Patterns

*A practical guide to composing several MCP servers (GitHub + Sentry + Postgres + Slack + …)
into one coherent agentic workflow — the patterns that work, the ones that blow up your
context window, and the security trap waiting in the middle.*

Last updated: June 2026. Every major claim is linked to a primary source at the point it appears.

---

## 0. TL;DR

- **The whole point of multi-MCP is cross-system leverage**: read from system A, act in system B.
  The canonical loop is *Sentry error → GitHub code → Postgres data → Slack notify*.
- **Naively connecting many servers breaks the agent.** Three servers (GitHub + Slack + Sentry,
  ~40 tools) once ate **143K of a 200K context window — 72%** — before the user said a word
  ([Junia](https://www.junia.ai/blog/mcp-context-window-problem)).
- **The fix is not "fewer servers" but "load tools lazily."** Tool Search (defer tool definitions)
  and code execution (treat MCP servers as code APIs) both cut tool-related tokens ~85–98%.
- **The "read-from-A / write-to-B" pattern is also the lethal trifecta.** The most useful
  composition is the most dangerous; gate the write step.

---

## 1. Why Multi-MCP — Composing Tools Across Systems

A single MCP server makes an agent good at *one* system. The value of an agent for a working
engineer comes from the **seams between systems** — the manual copy-paste work a human does
today: read the stack trace in Sentry, find the file on GitHub, check the row in Postgres,
post the result in Slack.

Each of those is a different MCP server. Composing them is what turns "a chatbot with a
GitHub plugin" into "an agent that closes the loop."

### The canonical incident-response flow

```
  ┌──────────┐   error id    ┌──────────┐   file + line   ┌──────────┐
  │  SENTRY  │──────────────▶│  GITHUB  │────────────────▶│  AGENT   │
  │  (read)  │  stack trace  │  (read)  │  offending code │ reasons, │
  └──────────┘               └──────────┘                 │ writes   │
                                                          │  a fix   │
        ┌─────────────────────────────────────────────────┘
        ▼                                   ▼
  ┌──────────┐   "is this row    ┌──────────┐   PR opened +   ┌──────────┐
  │ POSTGRES │    actually bad?"  │  GITHUB  │  Sentry linked  │  SLACK   │
  │  (read)  │◀──────────────────│  (write) │────────────────▶│ (write)  │
  └──────────┘   confirm w/ data └──────────┘                 │  notify  │
                                                              └──────────┘
```

This exact loop is described as a real Claude Code workflow: *"an alert fires in Sentry, you
paste the issue ID into Claude Code, and the agent reads the stack trace, pulls the offending
file from the repo, writes the fix, opens a PR, and links the PR back to the Sentry issue"*
([Code Velocity Academy](https://www.codevelocity.academy/en/blog/claude-code-mcp-integrations)).
Add Postgres for "confirm the data is actually wrong" and Slack for "tell the team," and you
have a four-server agent doing end-to-end work no single server could.

The key asymmetry to internalize: **reads are cheap and safe; writes are powerful and
dangerous.** Most of the design tension in this guide comes from that line.

---

## 2. Composition Patterns

There are four building blocks. Real workflows mix them.

### 2.1 Sequential chaining — output of A feeds B

The simplest pattern: a linear assembly line where each server's output becomes the next
server's input ([Knit](https://www.getknit.dev/blog/advanced-mcp-agent-orchestration-chaining-and-handoffs)).

```
 [Sentry.getIssue] ──▶ [GitHub.getFileContents] ──▶ [GitHub.createPR] ──▶ [Slack.postMessage]
       │                      │                            │                      │
   issue_id           file path from           PR url from            PR url into
   in → trace out     trace → code out         fix → url out          message in
```

Strength: predictable, debuggable, each step's contract is clear.
Weakness: the data passes *through the model's context at every hop* — the Sentry trace, then
the file, then the diff — which is exactly the token problem Section 6 solves.

### 2.2 Fan-out / gather — query several systems, synthesize

When the answer requires evidence from multiple systems *in parallel*, fan out, then gather.
This is the "Parallel Evaluator" pattern
([TrueFoundry](https://www.truefoundry.com/blog/multi-agent-system-with-mcp)).

```
                    ┌──▶ [GitHub: recent commits to file] ──┐
   "why did the     │                                        │
    p99 latency  ───┼──▶ [Postgres: query volume last 24h] ──┼──▶ [AGENT synthesizes]
    spike?"         │                                        │     ──▶ [Slack: post RCA]
                    └──▶ [Sentry: new error signatures]   ───┘
```

Strength: latency — three independent reads run concurrently instead of in series.
Weakness: you must reconcile possibly-contradictory evidence, and three full result sets land
in context at once.

### 2.3 Read-from-A / write-to-B — the most common AND most dangerous

This is the dominant production shape — and it is **structurally identical to the lethal
trifecta** (Section 9). The moment an agent can *read* from one place and *write/send* to
another, an attacker who controls the read side can steer the write side.

```
   READ SIDE (may be untrusted)              WRITE SIDE (exfiltration channel)
   ┌─────────────────────────┐               ┌──────────────────────────────┐
   │ GitHub public issue text│──── agent ────▶│ GitHub PR / Slack DM / HTTP   │
   │ Postgres free-text field│   reasons on   │ POST — sends data OUTWARD     │
   │ Email / web page content│  untrusted in  │                              │
   └─────────────────────────┘               └──────────────────────────────┘
            ▲                                          ▲
     attacker can plant text here          ... and it ends up here
```

Treat *every* read-from-A/write-to-B composition as a security decision, not just an
engineering one. See Section 9.

### 2.4 Human-in-the-loop gates between steps

The mitigation that makes 2.3 survivable: insert an **approval gate** before any consequential
write. Reads run autonomously; the irreversible/outbound step pauses for a human.

```
 [reads: Sentry+GitHub+Postgres]  ──▶  AGENT drafts PR + Slack message
                                            │
                                            ▼
                                   ┌──────────────────┐
                                   │  HUMAN APPROVES?  │ ◀── gate on the WRITE only
                                   └──────────────────┘
                                       │           │
                                     yes           no
                                       ▼           ▼
                              [GitHub.createPR]   discard / revise
                              [Slack.postMessage]
```

Gate placement rule of thumb: **autonomous on reads, gated on writes — especially writes that
leave the system** (PRs, emails, outbound HTTP, Slack messages to channels). This is the
practical form of "avoid combining all three trifecta legs"
([Simon Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)).

---

## 3. The "Too Many Tools / Too Many Servers" Problem

Connecting many MCP servers does not degrade gracefully — it falls off a cliff. Two distinct
failures stack on top of each other.

### 3.1 Context bloat — the 72% finding

MCP's default behavior dumps **all** tool definitions into the context at session start,
whether or not they'll be used — a protocol-level design choice where cost scales with the
number of tools ([Junia](https://www.junia.ai/blog/mcp-context-window-problem)).

> One team reported **three MCP servers (GitHub + Slack + Sentry, ~40 tools) consuming 143,000
> of 200,000 tokens — 72% of the context window — on tool definitions alone**, before any work
> began ([Junia](https://www.junia.ai/blog/mcp-context-window-problem)).

That's 72% of your budget spent on a *menu* the model mostly won't order from.

### 3.2 Tool-selection collapse — accuracy degrades as N grows

Even if the definitions fit, more tools make the model *worse at choosing*. Functions blur
together; the model picks the wrong one. The **RAG-MCP** paper
([arXiv 2505.03275](https://arxiv.org/abs/2505.03275)) demonstrates both halves:

- **Selection accuracy falls as the tool pool grows** — performance is high with a handful of
  tools and degrades steadily as you add more, because the prompt bloats and candidates blur.
- **Retrieval beats dumping.** Comparing *methods* on their stress test: the **"Blank
  Conditioning" baseline scored 13.62%**, while **RAG-MCP (retrieve only relevant tools first)
  scored 43.13%** — more than triple, with **>50% fewer prompt tokens**
  ([HuggingFace paper page](https://huggingface.co/papers/2505.03275)).

  *(Note: 43.13% vs 13.62% is a method-vs-method comparison — retrieval vs naive baseline — not
  a single before/after curve. The "accuracy falls as N grows" finding is a separate result in
  the same paper. Don't conflate them.)*

### 3.3 Mitigations

| Mitigation | What it does | Source |
|---|---|---|
| **Tool Search / deferred loading** | Load only tool *names* at start; fetch full definitions on demand. Adding servers barely touches context. | [Anthropic — advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use) |
| **Retrieval over tools (RAG-MCP)** | Vector-index tool metadata; retrieve top-k relevant tools per query before calling the model. | [arXiv 2505.03275](https://arxiv.org/abs/2505.03275) |
| **MCP gateway** | Front many servers behind one endpoint; centralize routing, scoping, auth (Section 4). | [Cloudflare](https://blog.cloudflare.com/zero-trust-mcp-server-portals/) |
| **Server scoping per task** | Connect only the 3–5 servers a given repo/task needs; project-level config so each repo gets only its tools. | [Code Velocity Academy](https://www.codevelocity.academy/en/blog/claude-code-mcp-integrations) |
| **Namespacing** | Prefix tools per server (`github.createPR`, `sentry.getIssue`) so the model disambiguates collisions. | [Portkey](https://portkey.ai/blog/orchestrating-multiple-mcp-servers-in-a-single-ai-workflow/) |
| **Collapse similar tools** | Merge near-duplicate tools into fewer task-level tools. | [Junia](https://www.junia.ai/blog/mcp-context-window-problem) |

**Practical default:** start with **3–5 servers**, add more only when needed
([Nimbalyst](https://nimbalyst.com/blog/claude-code-mcp-setup/)).

#### Tool Search, by the numbers

Anthropic's Tool Search Tool marks tools `defer_loading: true`. Only names + server
instructions load at start; when the model needs e.g. GitHub tools it searches, and the API
returns 3–5 relevant `tool_reference` blocks that expand into full definitions
([Anthropic](https://www.anthropic.com/engineering/advanced-tool-use)):

- Context dropped from **~77K → 8.7K tokens — ~85% reduction**, preserving 95% of the window.
- Searching "github" loads *only* `github.createPullRequest` and `github.listIssues` — **not
  your other 50+ Slack / Jira / Drive tools**.
- MCP-eval accuracy on large tool libraries: **Opus 4: 49% → 74%**, **Opus 4.5: 79.5% → 88.1%**.

In Claude Code 2.1.x this is on by default (env var `ENABLE_TOOL_SEARCH`)
([atcyrus](https://www.atcyrus.com/stories/mcp-tool-search-claude-code-context-pollution-guide)).

---

## 4. MCP Gateways & Routers

A gateway is a **session-aware reverse proxy + control plane** that fronts many MCP servers
behind one endpoint, adding routing, central authn/authz, policy, observability, and lifecycle
management ([Skywork](https://skywork.ai/blog/mcp-server-vs-mcp-gateway-comparison-2025/)).
The agent sees *one* MCP connection; the gateway fans out to N servers behind it.

```
                          ┌──────────────────────────────────┐
                          │            MCP GATEWAY            │
   ┌────────┐  one MCP    │  routing · auth · tool allowlist  │   ┌────────────┐
   │ AGENT  │────────────▶│  policy · audit log · secrets     │──▶│  github srv│
   │ (1 conn)│  endpoint   │  context optimization            │   ├────────────┤
   └────────┘             │                                   │──▶│  sentry srv│
                          │                                   │   ├────────────┤
                          └───────────────────────────────────┘──▶│ postgres   │
                                                                  ├────────────┤
                                                                  │  slack srv │
                                                                  └────────────┘
```

Guidance heard repeatedly: *one agent shouldn't connect to 15 MCP servers — build a gateway
that composes them behind a single interface*, authenticate once to a control plane that issues
scoped credentials, and audit all tool calls in one place
([TrueFoundry](https://www.truefoundry.com/blog/multi-agent-system-with-mcp)).

### Gateway products people actually use

| Gateway | What it gives you | Source |
|---|---|---|
| **Docker MCP Gateway** | Each server in an isolated container w/ minimal host privileges; Docker secrets keep creds out of env vars; built-in OAuth; per-server **tool allowlists** (enable/disable individual tools in a profile); dynamic discovery. | [GitHub docker/mcp-gateway](https://github.com/docker/mcp-gateway) |
| **Cloudflare MCP Server Portals** | Single SSO-style gateway for all servers; **context-optimization** to cut tool-definition tokens; aggregates all MCP request logs in one place; portal traffic can route through Cloudflare Gateway for DLP scanning. | [Cloudflare blog](https://blog.cloudflare.com/zero-trust-mcp-server-portals/), [Cloudflare One docs](https://developers.cloudflare.com/cloudflare-one/access-controls/ai-controls/mcp-portals/) |
| **IBM ContextForge** | OTel-instrumented MCP gateway with provider delegation chains. | [IBM docs](https://ibm.github.io/mcp-context-forge/latest/architecture/observability-otel/) |
| **Roll-your-own (Cloudflare Workers + Hono)** | Multi-domain gateway with a Streamlit test client — a documented DIY pattern. | [Medium — Jamalla Zawia](https://medium.com/@jamala.zawia/building-a-multi-domain-mcp-gateway-with-cloudflare-workers-hono-and-streamlit-30a0561eb7df) |

Docker MCP Gateway, concretely:

```bash
docker mcp gateway run                                    # default profile
docker mcp gateway run --profile dev-tools --port 8080 --transport streaming
docker mcp tools ls                                        # list aggregated tools
docker mcp profile tools <profile-id> --enable github.createPullRequest
```

The per-tool allowlist (`--enable <server>.<tool>`) is the gateway-level version of "server
scoping": you expose only the slice of each server a task needs, which directly attacks both
the bloat (Section 3.1) and the selection-collapse (Section 3.2) problems
([docker/mcp-gateway](https://github.com/docker/mcp-gateway)).

---

## 5. Orchestration: Who Drives?

The single most important architectural decision. It maps directly onto Anthropic's
**workflow vs. agent** distinction ([Anthropic — Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)):

> **Workflows** are systems where LLMs and tools are orchestrated through *predefined code
> paths*. **Agents** are systems where LLMs *dynamically direct their own processes and tool
> usage.*

```
  AGENT-DRIVEN (model decides next MCP tool)     WORKFLOW-DRIVEN (code decides, fixed order)

   ┌─────────────────────────────────┐            sentry = mcp.call("sentry.getIssue", id)
   │  loop:                          │            file   = mcp.call("github.getFile", path)
   │    model looks at state         │            fix    = llm("write a fix", file, sentry)
   │    model picks an MCP tool      │            pr     = mcp.call("github.createPR", fix)
   │    tool runs, result → context  │            if human_approves(pr):
   │    repeat until done            │                mcp.call("slack.post", pr.url)
   └─────────────────────────────────┘
        flexible, opaque, costlier              deterministic, auditable, cheaper
```

| | Agent-driven | Workflow-driven |
|---|---|---|
| **Who picks the next MCP tool** | The model, each turn | Your code, fixed sequence |
| **Best when** | Path is unknown / varies per task (open-ended debugging, research) | Structure is stable enough to encode (nightly RCA report, ticket triage) |
| **Cost** | Inference at every step | Inference only at the decision points *you* chose |
| **Failure mode** | Wrong tool picked, loops, cost blowups | Brittle if reality deviates from the encoded path |
| **Auditability** | Harder — non-deterministic | Easy — same path every run |

Anthropic's central advice: **start simple; add agentic autonomy only when simpler solutions
fall short.** *"Workflows beat agents whenever a task's structure is stable enough to encode in
code, because they pay the inference cost only at decision points the developer chose"*
([Anthropic](https://www.anthropic.com/research/building-effective-agents)). For multi-MCP
specifically: if you can name the servers and their order in advance (Sentry→GitHub→Slack every
night), prefer a workflow. Reserve the agent loop for genuinely open-ended tasks.

---

## 6. MCP + Code Execution — Composing Many Servers Token-Efficiently

This is the most important recent technique, and it directly solves the bloat *and* the
intermediate-result problem at once. Source:
[Anthropic — Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp)
(Adam Jones & Conor Kelly, Nov 2025).

### The problem it solves

Two leaks in the traditional "direct tool call" model:

1. **Tool-definition bloat** — hundreds of definitions loaded before the model reads the request.
2. **Intermediate-result duplication** — chained calls pass data *through the model context at
   every hop*. A meeting transcript fetched from Drive and written to Salesforce traverses
   context **twice**.

### The pattern: MCP servers as code APIs

Instead of exposing tools as callable functions, present them as files in a **filesystem the
agent navigates** and writes code against:

```
servers/
├── google-drive/getDocument.ts
├── salesforce/updateRecord.ts
├── github/createPullRequest.ts
└── sentry/getIssue.ts
```

The agent writes TypeScript/Python that calls them and **filters/transforms data locally**,
returning only the final result to the model:

```typescript
const transcript = (await gdrive.getDocument({ documentId: 'abc123' })).content;
await salesforce.updateRecord({
  objectType: 'SalesMeeting',
  recordId:   '00Q5f...',
  data:       { Notes: transcript },
});
// transcript never enters the model's context
```

### The numbers

> Drive→Salesforce workflow: **150,000 tokens → 2,000 tokens — a 98.7% reduction**
> ([Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp)).

A separate validation scaled the code-first pattern to **112 GitHub tools while keeping the
~98% token reduction**
([modelcontextprotocol discussion #629](https://github.com/orgs/modelcontextprotocol/discussions/629)).

Anthropic's **Programmatic Tool Calling** (the productized version) reports **43,588 → 27,297
tokens, a 37% reduction on complex research tasks**, with accuracy gains (GIA 46.5%→51.2%),
because Claude writes code that uses `asyncio.gather()` to **parallelize calls across many
servers and process results without round-tripping intermediate data to context**
([Anthropic — advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use)).

### Why this is the multi-MCP unlock

```
  TRADITIONAL: 5 tools = 5 inference passes, data through context 5×
  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │ A │─▶│ B │─▶│ C │─▶│ D │─▶│ E │   (model re-reads everything each hop)
  └───┘  └───┘  └───┘  └───┘  └───┘

  CODE EXECUTION: 1 inference pass writes code that calls A..E in the sandbox
  ┌─────────────────────────────────────────────┐
  │  one code block: gather(A,B,C,D,E), filter   │  → returns only the answer
  └─────────────────────────────────────────────┘
```

Bonus benefits Anthropic calls out: **progressive disclosure** (load tool defs on-demand by
navigating the filesystem), **privacy** (the MCP client can tokenize PII so sensitive data
flows server-to-server *without entering model context*), and **state persistence** (save
intermediate results and reusable code as "skills").

**Trade-off:** code execution needs sandboxing, resource limits, and monitoring — real
operational complexity vs. plain tool calls
([Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp)).

---

## 7. Statefulness Across Servers

Multi-MCP workflows are only useful if context flows between calls. Two layers matter.

### 7.1 Passing IDs / context between MCP calls

The glue is usually a **correlation ID threaded through the chain.** In the incident loop, the
**Sentry issue ID is the spine**: it identifies the error, points at the file in GitHub, names
the PR, and gets **linked back into the Sentry issue** so the loop closes
([Code Velocity Academy](https://www.codevelocity.academy/en/blog/claude-code-mcp-integrations)).

```
  sentry.issue_id ──┬──▶ github.getFile(path_from_trace)
                    ├──▶ github.createPR(title="Fix SENTRY-1234")
                    └──▶ sentry.linkPR(issue_id, pr_url)   ← loop closes on the same ID
```

In the **code-execution** model, state is even simpler: intermediate results live in the
**sandbox execution environment**, not in model context, and the agent can **save them as
reusable skills for resumable workflows**
([Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp)). The variable
holding the Sentry trace is just a variable — passed to the GitHub call directly.

### 7.2 Trace propagation & correlation (observability)

For debugging *the agent itself* across servers, the emerging standard is **OpenTelemetry with
W3C Trace Context.** When the agent calls a tool, OTel injects trace headers; the tool server
picks up the context and continues the same trace — giving one coherent picture from prompt →
tool → downstream API across language boundaries (a Python client's spans link to a Node tool
server's spans) ([SigNoz](https://signoz.io/blog/mcp-observability-with-otel/)).

- A shared **TraceId** ties all related activity across the request series; **parent SpanId**
  forms the activity graph ([SigNoz](https://signoz.io/blog/mcp-observability-with-otel/)).
- **Baggage** carries metadata downstream — but propagate `traceparent` while keeping baggage
  internal so you don't **leak tenant/user metadata to external servers**
  ([SigNoz](https://signoz.io/blog/mcp-observability-with-otel/)).
- There's an open proposal to standardize OTel trace IDs *in the MCP protocol itself*
  ([modelcontextprotocol discussion #269](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/269)).
- Tooling exists today: **FastMCP** ships native OTel instrumentation
  ([FastMCP docs](https://gofastmcp.com/servers/telemetry)); IBM **ContextForge** instruments
  provider delegation chains
  ([IBM docs](https://ibm.github.io/mcp-context-forge/latest/architecture/observability-otel/)).

Rule: **one application-level correlation ID** (the Sentry issue) for the *business* workflow,
plus **one OTel trace** spanning all servers for *operational* debugging. They answer different
questions — "did the work get done?" vs. "where did the agent spend time / fail?"

---

## 8. Real Multi-MCP Setups (Documented, With Sources)

| Setup | Servers (3+) | Workflow | Source |
|---|---|---|---|
| **Claude Code incident loop** | Sentry + GitHub + (Postgres/Slack) | Paste Sentry issue ID → agent reads stack trace, pulls offending file from GitHub, writes fix, opens PR, links PR back to Sentry. | [Code Velocity Academy](https://www.codevelocity.academy/en/blog/claude-code-mcp-integrations) |
| **Dev-workflow stack** | GitHub + PostgreSQL + Slack + Figma + Context7 | Issues/PRs (GitHub), schema & data checks (Postgres), deploy notices (Slack), design refs (Figma), library docs (Context7). Recommended starter combo. | [Nimbalyst](https://nimbalyst.com/blog/best-claude-code-mcp-servers/), [Clarista](https://www.clarista.io/blog/claude-code-mcp-plugins-guide) |
| **Sentry-first triage** | Sentry + GitHub | Sentry MCP surfaces error + suspect commit; GitHub MCP opens the fix — argued as the *second* server to install after GitHub. | [Tygart Media](https://tygartmedia.com/claude-code-sentry-mcp-server-second-install/), [getsentry/sentry-mcp](https://github.com/getsentry/sentry-mcp) |
| **OTel-debugging agent** | Jaeger + Tempo + Traceloop (multi-backend) | One MCP server unifies trace queries across backends so an agent can analyze distributed traces for automated debugging. | [traceloop/opentelemetry-mcp-server](https://github.com/traceloop/opentelemetry-mcp-server) |
| **Production patterns survey** | Router-worker over many servers | Documents chaining, handoffs, router-worker, parallel-evaluator patterns observed in production multi-agent MCP systems. | [dev.to — 9 MCP production patterns](https://dev.to/dohkoai/9-mcp-production-patterns-that-actually-scale-multi-agent-systems-2026-4ap3) |

A minimal multi-server Claude Code config (two servers, mixed transports) — extend with
`sentry` and `slack` blocks the same way
([Code Velocity Academy](https://www.codevelocity.academy/en/blog/claude-code-mcp-integrations)):

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-postgres", "postgresql://localhost/myapp"]
    },
    "github": {
      "type": "http",
      "url": "https://api.github.com/mcp"
    }
  }
}
```

**Honest caveat:** most public "multi-server" write-ups document *per-server capabilities and a
shared config*, and assert the cross-system loop, rather than publishing a fully-traced
end-to-end automation. The Sentry→GitHub→PR loop is the most concretely documented chain; treat
four-server Postgres+Slack extensions as the natural composition, validated piecewise.

---

## 9. Failure Modes & Best Practices

### 9.1 The lethal trifecta — the headline risk

Simon Willison's framing: an agent is dangerous when it has **all three** of
([Simon Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)):

```
   ┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
   │ 1. PRIVATE DATA     │ + │ 2. UNTRUSTED CONTENT │ + │ 3. EXTERNAL COMMS   │
   │  (Postgres, private │   │ (public GitHub issue,│   │ (PR, Slack, outbound│
   │   repo, email)      │   │  email, web page)    │   │  HTTP)              │
   └─────────────────────┘   └─────────────────────┘   └─────────────────────┘
                              ▼  combine all three  ▼
            indirect prompt injection → automated data-theft pipeline
```

**Multi-MCP makes this the default, not the edge case** — MCP explicitly encourages combining
diverse servers, and many servers provide data access, accept hostile input, *and* enable
outbound comms ([Simon Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)).
The real **GitHub MCP** vulnerability had all three: read public issues (untrusted), access
private repos (private data), open PRs (exfiltration) — a malicious public issue could exfil
private repo data via a PR
([Simon Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)).

The root cause is architectural: an LLM has **no built-in way to separate trusted commands from
untrusted data — both arrive as the same token stream**
([practical-devsecops](https://www.practical-devsecops.com/glossary/mcp-indirect-prompt-injection/)).

**Primary defense (the only reliable one): don't give one agent all three legs.** Break the
chain — remove the exfiltration channel, or gate it behind a human (Section 2.4). Willison is
explicit that "95% protection" guardrails are *not* enough; security needs near-perfect rates
([Simon Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)).

### 9.2 The other failure modes

| Failure | Cause | Design-around |
|---|---|---|
| **Wrong tool picked** | Too many tools blur together (Section 3.2). | Tool Search / RAG-MCP; namespacing; collapse duplicates; scope to 3–5 servers. |
| **Context bloat** | All defs loaded upfront — 72% of window gone. | Deferred loading; gateway context-optimization; code execution. |
| **Cross-server data leakage** | Private data flows through model context, or trace baggage leaks tenant info to external servers. | Code execution keeps intermediates in the sandbox + PII tokenization; keep OTel baggage internal. |
| **Cost blowups** | Agent loop re-inferences every hop; large results re-read each step. | Workflow-driven orchestration; programmatic/code-execution tool calling (fewer inference passes). |
| **Cascading failure** | One server down stalls a sequential chain. | Gateway lifecycle mgmt + health checks; design fan-out steps to degrade gracefully. |
| **Untrusted secrets in env** | Creds in env vars across many servers. | Gateway-managed secrets (Docker secrets / OAuth) instead of env vars. |

### 9.3 Best-practice checklist

- [ ] **Start with 3–5 scoped servers**, not 15. Add only when a task needs it.
- [ ] **Defer tool loading** (Tool Search on) so adding servers doesn't cost context.
- [ ] **Namespace tools** per server to avoid collisions and mis-selection.
- [ ] **Prefer a workflow** when the server order is known; reserve the agent loop for open-ended work.
- [ ] **Use code execution** for multi-hop data movement — keep intermediates out of context.
- [ ] **Audit the trifecta on every read→write pair.** If all three legs are present, break one.
- [ ] **Gate consequential writes** (PRs, Slack, outbound HTTP) behind human approval.
- [ ] **Front many servers with a gateway** for central auth, scoping, audit, and secrets.
- [ ] **Thread one correlation ID** for the business loop + **one OTel trace** for ops debugging.
- [ ] **Keep OTel baggage internal**; don't propagate tenant/user metadata to external servers.

---

## Sources

**Anthropic (primary):**
- [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) — 150K→2K, filesystem-as-API, privacy, state.
- [Advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use) — Tool Search (77K→8.7K, ~85%), Opus 4 49%→74%, Opus 4.5 79.5%→88.1%, Programmatic Tool Calling 43.6K→27.3K.
- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — workflow vs. agent.

**The "too many tools" problem:**
- [Junia — MCP context window problem](https://www.junia.ai/blog/mcp-context-window-problem) — the 143K/200K (72%) finding.
- [RAG-MCP — arXiv 2505.03275](https://arxiv.org/abs/2505.03275) / [paper page](https://huggingface.co/papers/2505.03275) — 43.13% vs 13.62%, >50% fewer tokens.
- [Writer — RAG-MCP deep dive](https://writer.com/engineering/rag-mcp/).
- [atcyrus — MCP Tool Search](https://www.atcyrus.com/stories/mcp-tool-search-claude-code-context-pollution-guide).

**Gateways:**
- [docker/mcp-gateway](https://github.com/docker/mcp-gateway) · [Cloudflare blog](https://blog.cloudflare.com/zero-trust-mcp-server-portals/) · [Cloudflare One docs](https://developers.cloudflare.com/cloudflare-one/access-controls/ai-controls/mcp-portals/) · [Skywork comparison](https://skywork.ai/blog/mcp-server-vs-mcp-gateway-comparison-2025/) · [Max — 5 best gateways](https://www.getmaxim.ai/articles/5-best-mcp-gateways-for-developers-in-2026-2/).

**Orchestration patterns & real setups:**
- [Portkey — orchestrating multiple MCP servers](https://portkey.ai/blog/orchestrating-multiple-mcp-servers-in-a-single-ai-workflow/) · [Knit — chaining & handoffs](https://www.getknit.dev/blog/advanced-mcp-agent-orchestration-chaining-and-handoffs) · [TrueFoundry](https://www.truefoundry.com/blog/multi-agent-system-with-mcp) · [dev.to — 9 production patterns](https://dev.to/dohkoai/9-mcp-production-patterns-that-actually-scale-multi-agent-systems-2026-4ap3) · [Code Velocity Academy](https://www.codevelocity.academy/en/blog/claude-code-mcp-integrations) · [Nimbalyst](https://nimbalyst.com/blog/best-claude-code-mcp-servers/) · [getsentry/sentry-mcp](https://github.com/getsentry/sentry-mcp).

**Statefulness / observability:**
- [SigNoz — MCP observability with OTel](https://signoz.io/blog/mcp-observability-with-otel/) · [FastMCP telemetry](https://gofastmcp.com/servers/telemetry) · [MCP OTel trace proposal #269](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/269) · [traceloop/opentelemetry-mcp-server](https://github.com/traceloop/opentelemetry-mcp-server).

**Security / lethal trifecta:**
- [Simon Willison — the lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) · [practical-devsecops — MCP indirect prompt injection](https://www.practical-devsecops.com/glossary/mcp-indirect-prompt-injection/) · [Oso — lethal trifecta](https://www.osohq.com/learn/lethal-trifecta-ai-agent-security).
