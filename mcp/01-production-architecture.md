# Production Architecture for MCP Servers

A practical guide for an engineer who has *used* MCP servers and now wants to *build and
ship one to production*. The protocol is moving fast — transports changed twice in 2025,
and a stateless-first redesign is now accepted into the standards track. This guide
anchors on the **2025-11-25 spec as the current production baseline** (what you build
against today) and flags **where the protocol is heading** so you don't ship something
that's obsolete in a quarter.

> Version note: MCP versions are date strings (e.g. `2025-11-25`). The current stable
> spec is **2025-11-25**. The previous widely-deployed versions are `2025-06-18` and
> `2025-03-26`. The original `2024-11-05` transport (HTTP+SSE) is deprecated. See
> [§6 Versioning](#6-versioning--lifecycle).

---

## 1. MCP architecture fundamentals — the mental model

MCP is a client/server protocol that lets an AI application call out to external
capabilities in a standardized way. Three roles matter, and people conflate them
constantly:

| Role | What it is | Examples |
|------|-----------|----------|
| **Host** | The AI *application* the user interacts with. Owns the LLM loop, spawns/holds client connections, enforces user consent. | Claude Desktop, Cursor, VS Code, your own agent app |
| **Client** | A connector *inside* the host. Maintains a **1:1 stateful connection** to exactly one server. | One client per connected server |
| **Server** | The process you build. Exposes capabilities. Knows nothing about the LLM. | Your GitHub server, your internal-API wrapper |

The key relationship: **a host contains many clients; each client talks to exactly one
server.** A server never talks to the LLM directly — it hands capabilities to the client,
the host decides what to surface to the model, and the model decides what to call. This
separation is what keeps the user in the loop (the host can gate every tool call behind
consent).

### What a server exposes — the three primitives

A server advertises capabilities during initialization and offers up to three primitive
types ([spec: server features](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)):

| Primitive | Controlled by | Purpose | Analogy |
|-----------|--------------|---------|---------|
| **Tools** | Model | Functions the LLM can *invoke* (with side effects). `tools/list`, `tools/call`. | POST endpoints |
| **Resources** | Application | Readable data the host can load into context (files, records, schemas). `resources/list`, `resources/read`. | GET endpoints / files |
| **Prompts** | User | Reusable templated workflows the user can trigger (e.g. slash commands). `prompts/list`, `prompts/get`. | Saved snippets |

The "controlled by" column is the real mental model: **tools are model-driven, resources
are app-driven, prompts are user-driven.** Most production servers ship mostly *tools*,
because that's what an autonomous agent reaches for. Resources and prompts are
underused but valuable for read-heavy and human-in-the-loop workflows.

### The JSON-RPC layer

Everything is [JSON-RPC 2.0](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports),
UTF-8 encoded. Three message shapes:

- **Request** — has `id` + `method` + `params`; expects a response.
- **Response** — has matching `id` + `result` or `error`.
- **Notification** — `method` + `params`, no `id`; fire-and-forget.

```
Client                         Server
  │   {"jsonrpc":"2.0","id":1,    │
  │    "method":"tools/call",     │
  │    "params":{...}}            │
  │ ────────────────────────────> │
  │                               │  (runs the tool)
  │   {"jsonrpc":"2.0","id":1,    │
  │    "result":{...}}            │
  │ <──────────────────────────── │
```

The protocol is **transport-agnostic** — JSON-RPC semantics are identical whether the
bytes flow over stdin/stdout or HTTP. That decoupling is the whole reason you can write
one server and run it locally *or* remotely.

---

## 2. Transports — stdio vs SSE vs Streamable HTTP

This is where 2025 churned the most. There are now **two standard transports**
([spec](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)):
`stdio` and **Streamable HTTP**. The old **HTTP+SSE** transport is **deprecated**.

### stdio — for local servers

The client launches your server as a **subprocess** and talks over stdin/stdout. JSON-RPC
messages are newline-delimited (and **MUST NOT** contain embedded newlines). `stderr` is
free for logging — but the client *"SHOULD NOT assume stderr output indicates error
conditions."* Critically: *"The server MUST NOT write anything to its stdout that is not a
valid MCP message"* — a stray `print()` to stdout corrupts the stream. This is the #1
beginner bug.

```
Host process
   └─ spawns ─► your-server (subprocess)
        stdin  ◄── JSON-RPC requests
        stdout ──► JSON-RPC responses
        stderr ──► logs (free-form)
```

stdio is **zero-network, zero-auth, single-user, lowest-latency**. The spec says clients
*"SHOULD support stdio whenever possible."* Use it for: dev tools, local CLIs, anything
running on the same machine as the host.

### HTTP+SSE (deprecated) — and why it died

The original remote transport (`2024-11-05`) used **two endpoints**: a `GET /sse` that the
client held open to *receive* messages, and a separate `POST /messages` to *send* them. It
was deprecated in `2025-03-26`. The problems, per
[fka.dev's analysis](https://blog.fka.dev/blog/2025-06-06-why-mcp-deprecated-sse-and-go-with-streamable-http/):

- **Two coordinated connections** — *"like trying to have a conversation using two
  phones — one for speaking and one for listening."*
- **Mandatory long-lived connections** — even a trivial request/response needed an open
  SSE stream held server-side: *"resource-intensive and challenging to scale."*
- **Fragile recovery** — *"if the SSE connection dropped during a long-running operation,
  responses would be lost."*
- Poor fit for modern HTTP infra (load balancers, serverless, HTTP/2+).

### Streamable HTTP — the current recommended remote transport

Introduced in `2025-03-26`, this is **the** transport for any networked/production server.
**One endpoint** (e.g. `https://example.com/mcp`) that handles both `POST` and `GET`
([spec](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)):

- Client **POSTs** every JSON-RPC message to the MCP endpoint. `Accept` header **MUST**
  list both `application/json` and `text/event-stream`.
- For a request, the server chooses per-response: return **one JSON object** (simple,
  serverless-friendly) **or** upgrade to an **SSE stream** (`Content-Type:
  text/event-stream`) for streaming progress + the final result.
- A notification/response from the client gets a bare **`202 Accepted`**.
- Optional **`GET`** opens a server→client SSE stream for unsolicited messages; server
  returns **405** if it doesn't offer one.

```
                  ┌──────────── single endpoint: POST/GET /mcp ───────────┐
Client ── POST request ──►  Server
        ◄── 200 application/json ──            (simple: one JSON body)
              ── OR ──
        ◄── 200 text/event-stream ──           (streaming: SSE, progress + result)
              event: id=42 data:{"progress":...}
              event: id=43 data:{"result":...}
```

The "dynamic upgrade" is the elegance: behaves like plain request/response for the 90%
case, upgrades to streaming only when a tool call is long-running or wants to push
progress. This is why it works behind ordinary load balancers and on serverless.

### Local vs remote at a glance

| | stdio | Streamable HTTP |
|---|-------|-----------------|
| Topology | subprocess, same machine | network service |
| Auth | none (OS process trust) | OAuth 2.1 / bearer tokens |
| Multi-tenant | no — one process per client | yes |
| Latency | lowest | network-bound |
| Scaling concern | none | concurrency, sessions, LB |
| Security must-do | don't pollute stdout | validate `Origin`, bind localhost when local, authenticate |

> **Security (Streamable HTTP):** the spec **MUSTs** you to validate the `Origin` header
> (return `403` if invalid) to prevent DNS-rebinding attacks, **SHOULD** bind only to
> `127.0.0.1` when running locally, and **SHOULD** authenticate all connections.
> *"Without these protections, attackers could use DNS rebinding to interact with local
> MCP servers from remote websites."*

---

## 3. Stateful vs stateless servers

This is the single most important production decision, and the protocol itself is
mid-pivot on it.

### How sessions work today (2025-11-25)

Sessions are **optional** in the current spec. On initialization the server **MAY** return
an `MCP-Session-Id` header on the `InitializeResult`. If it does, the client **MUST** echo
it on every subsequent request. Rules
([spec](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)):

- Session ID **SHOULD** be globally unique + cryptographically secure (UUID / JWT / hash),
  visible-ASCII only.
- Server requiring a session **SHOULD** reply `400` to non-init requests lacking the
  header.
- Server **MAY** terminate a session → then replies `404` to that session ID. Client
  getting a `404` **MUST** re-`initialize`.
- Client **SHOULD** `DELETE` the endpoint with the session header to end a session
  cleanly.

### When you actually need state

| Need state? | Examples |
|-------------|----------|
| **No (prefer stateless)** | Stateless tool calls — wrap a REST API, query a DB, run a calculation. Each call is self-contained. |
| **Yes (accept the cost)** | Multi-step flows that accumulate context (shopping cart → checkout), server-initiated events, resource subscriptions, a persistent per-session knowledge graph or game. |

### The scaling pain that's driving the redesign

Stateful sessions are where production deployments hurt. Session state lives **in-memory on
one instance**, so:

- A stateless L4/L7 round-robin LB will route a client's next request to a *different*
  replica that has no session → broken. You're forced into **sticky sessions**, which the
  [SEP-2575 motivation](https://modelcontextprotocol.io/seps/2575-stateless-mcp) calls
  *"complex and fragile,"* causing uneven load and non-trivial horizontal scaling.
- If the instance holding a session **dies, the session is lost**; the client must
  reconnect and redo the whole init handshake.
- AWS's reference work notes the practical gap bluntly: *"the official MCP SDKs do not
  support external session persistence (in services like Redis or DynamoDB)"* — so teams
  resort to **ALB sticky sessions + custom cookie handling**, even patching in a
  `fetch-cookie` workaround because the TS client SDK lacks native cookie support
  ([AWS samples](https://deepwiki.com/aws-samples/sample-serverless-mcp-servers/1.1-stateful-vs.-stateless-architecture)).

| Aspect | Stateful | Stateless |
|--------|----------|-----------|
| Load balancing | sticky sessions required | any algorithm; any instance serves any request |
| Fault tolerance | session lost on instance death | requests reroute to healthy instances |
| Scaling | non-trivial | horizontal, unconstrained |
| Typical deploy | ECS Fargate + ALB (long-lived) | Lambda + API Gateway, Workers, any container |

**Practical rule for shipping today:** default to **stateless** — make each tool call
self-contained, push any real state into an external store (DB/Redis) keyed by user/tenant
identity from the auth token, not into an in-memory session map. Reach for stateful
sessions only when a use case genuinely requires server-pushed events or cross-request
context, and when you do, isolate per-session state so it can be externalized (this is
exactly what Cloudflare's Durable-Object-per-session model gives you — see §4).

### Where the protocol is heading: stateless-first (SEP-2575, accepted)

This isn't speculation — **[SEP-2575 "Make MCP Stateless"](https://modelcontextprotocol.io/seps/2575-stateless-mcp)
is Final / Standards Track** (created 2025-06-18). It is a **backward-incompatible change
requiring a new protocol version**. Highlights of the accepted direction:

- **Removes the `initialize` handshake** and `notifications/initialized` entirely.
  Capability/version negotiation becomes *per-request*.
- Every request carries `MCP-Protocol-Version` (header **and** `_meta`), plus per-request
  `clientInfo` and `clientCapabilities` in `_meta`. *"Servers MUST NOT infer capabilities
  from prior requests."*
- Adds a **`server/discover`** RPC so clients can fetch supported versions + capabilities
  without a handshake.
- **Removes** `ping`, `resources/subscribe`, `logging/setLevel`, the server→client `GET`
  SSE endpoint, and **resumable streams via `Last-Event-ID`** (durability moves to the
  *tasks* primitive). A dropped connection now *implicitly cancels* the request.
- Backward compat: a server **MAY** keep implementing legacy `initialize` for old clients
  while exposing the new stateless RPCs for new ones.

The throughline to internalize: **stateful is the source of production pain → the spec is
deliberately moving to stateless-first.** If you build stateless today, you are already
aligned with where MCP is going. (Note the FAQ caveat: SSE streams still bundle multiple
messages within one HTTP request, so it's "stateless *by default*," not absolutely
stateless — and you can still build stateful *applications* on top, the same way the web
builds stateful apps on stateless HTTP.)

---

## 4. Deployment & hosting

Remote servers are just web services that speak Streamable HTTP. The interesting part is
how teams handle **auth, session state, and scaling** on each platform.

### Cloudflare Workers + Durable Objects — the canonical edge pattern

Cloudflare's stack is the most documented production path
([Cloudflare blog](https://blog.cloudflare.com/remote-model-context-protocol-servers-mcp/)):

| Component | Role |
|-----------|------|
| `workers-oauth-provider` | OAuth 2.1 library that *"wraps your Worker's code, adding authorization to API endpoints."* |
| **`McpAgent`** (Agents SDK) | Holds the MCP connection; *"uses Durable Objects behind the scenes to hold a persistent connection open."* |
| **Durable Objects** | *"Each MCP client session is backed by a Durable Object… each session can manage and persist its own state, backed by its own SQL database."* |
| `mcp-remote` adapter | Lets local-only clients reach remote servers. |

The clever part is the **session-as-Durable-Object** model: it gives you stateful servers
*without* sticky-session fragility, because the platform routes a session deterministically
to its object and that object owns durable per-session SQL storage. The OAuth flow is also
notable — *"your Worker stores an encrypted access token in Workers KV. It then issues its
own token to the client,"* so the raw upstream token never reaches the MCP client.

```
        ┌───────────── Cloudflare edge (300+ POPs) ─────────────┐
Client ─OAuth─► workers-oauth-provider ─► Worker ─► McpAgent
                                                       │
                                          Durable Object per session
                                            └─ persistent SQL state
```

### AWS — serverless vs container, pick by state

AWS's reference samples split cleanly
([AWS samples](https://deepwiki.com/aws-samples/sample-serverless-mcp-servers/1.1-stateful-vs.-stateless-architecture)):

- **Stateless** → **Lambda + API Gateway** (Node or Python) or **ECS + ALB**. Scales
  horizontally, short-lived connections, serverless-native.
- **Stateful** → **ECS Fargate + ALB** with sticky sessions and custom cookie handling
  (because SDKs lack external session persistence).

### Hosting platforms & registries

| Platform | What it does | Note |
|----------|-------------|------|
| **Cloudflare** | Build + host remote servers at the edge; enterprise portals (below) | [docs](https://developers.cloudflare.com/agents/guides/remote-mcp-server/) |
| **Smithery** | Registry + hosting | **Discontinuing stdio hosting (Sept 7, 2025)** — push toward remote |
| **Glama** | *Hosts and runs* servers for you (not just a directory) | [via Composio](https://composio.dev/blog/smithery-alternative) |
| **Composio** | 100+ managed servers with built-in auth/credential management | integration-first |
| **Apify** | Full stdio support, no deprecation; deploy without rewriting | [Apify blog](https://blog.apify.com/smithery-alternative/) |
| **TrueFoundry** | Enterprise AI gateway: LLM routing + MCP tool governance | control-plane play |
| **MCP.so / MCP Hub** | Discovery directories only (no hosting) | |

### Enterprise: put servers behind a gateway/portal

Cloudflare's [enterprise reference architecture](https://blog.cloudflare.com/enterprise-mcp/)
argues that running raw servers per-team creates *"no centralized authentication, limited
observability, and access control gaps,"* and that locally-hosted servers are *"a security
liability."* Their pattern:

- **MCP server portals** — one endpoint per user that *"immediately reveals every internal
  and third-party MCP servers they are authorized to use,"* with centralized logging,
  policy, and DLP.
- **Cloudflare Access as OAuth provider / identity aggregator** — SSO, MFA, device posture.
- **Code Mode** — collapses many tools into a couple of portal tools: *"52 tools collapse
  into 2 portal tools… roughly 600 tokens, a 94% reduction."*
- **Shadow-MCP detection** — Gateway scans hostnames + JSON-RPC bodies (`tools/call`,
  `initialize`) to find unauthorized servers.

The general lesson regardless of vendor: **front your production servers with a gateway
that owns auth, logging, and rate limiting** rather than reimplementing those in every
server.

---

## 5. Scaling & reliability

### Concurrency & connection management

Streamable HTTP lets the server *"handle multiple client connections"* as an independent
process — so concurrency is ordinary web-service concurrency. The wins come from keeping
responses **non-streaming where possible** (a plain JSON body ties up no long-lived
connection) and only upgrading to SSE for genuinely long calls. For long-lived SSE
streams, the spec lets the server *close the connection while keeping the logical stream
alive* and have the client **poll-reconnect** via `Last-Event-ID` — so you don't pin a
socket for minutes (relevant on `2025-11-25`; note resumable streams are removed under
the upcoming stateless spec — use *tasks* there instead).

### Timeouts & cancellation

Both sides **SHOULD** set per-request timeouts; on timeout the sender **SHOULD** issue a
`CancelledNotification` and stop waiting
([lifecycle spec](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle)).
Progress notifications **MAY** reset the timeout clock — but implementations **SHOULD**
still enforce a **hard maximum** to contain a misbehaving peer. Make timeouts
**configurable per-request** (a 2-second DB lookup and a 5-minute report build want very
different limits).

### Long-running tool calls

Two production options:
1. **Stream progress** over SSE: emit `notifications/progress` with a `progressToken`,
   send the final `result` last. Keeps the user informed and the timeout clock alive.
2. **Async job + polling** (the durable pattern, and the direction the protocol favors via
   the **tasks** primitive): the tool returns a job handle immediately; the client polls
   for status/result. This survives disconnects — essential once resumable streams go
   away, and it's the only sane model on serverless with execution-time caps.

### Rate limiting & idempotency (best practice — not mandated by spec)

The spec doesn't prescribe these; treat them as standard service hygiene applied to the
tool layer:

- **Rate limiting** — enforce at the gateway (per token/tenant) and inside the server for
  expensive tools. Return a JSON-RPC `error` (not a silent failure) so the model can back
  off; surface limits clearly so agents don't hammer in a retry loop. This is a core
  reason enterprises front servers with a portal/gateway (§4).
- **Idempotency** — agents and flaky networks retry. Make write-tools idempotent: accept a
  client-supplied idempotency key (or derive one from inputs), dedupe, and return the
  prior result on replay. Without this, a retried `create_invoice` double-charges. Reads
  are naturally idempotent; concentrate effort on side-effecting tools.

### Reliability checklist

- Validate `Origin` (`403` on mismatch); authenticate every connection.
- Externalize any real state (DB/Redis) keyed by auth identity — don't trust in-memory
  session maps to survive a deploy or instance loss.
- Prefer stateless request/response; reserve SSE for long calls.
- Per-request configurable timeouts + a hard cap; honor cancellation.
- Idempotent writes; rate-limit at the edge; return structured JSON-RPC errors.
- Long jobs → async + poll (tasks), not a 5-minute held socket.

---

## 6. Versioning & lifecycle

### Lifecycle phases (2025-11-25)

[Lifecycle spec](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle):
**Initialization → Operation → Shutdown.**

```
Client                          Server
  │  initialize (protocolVersion, capabilities, clientInfo)
  │ ─────────────────────────────►
  │  result (protocolVersion, capabilities, serverInfo, instructions?)
  │ ◄─────────────────────────────
  │  notifications/initialized
  │ ─────────────────────────────►      ── normal operation ──
```

The client **MUST** open with `initialize`; only `ping`/logging may precede the
`initialized` notification.

### Version negotiation

Versions are **date strings** (`2025-11-25`). The client sends its preferred (latest)
version in `initialize`. If the server supports it, it echoes the *same* version; otherwise
it responds with another version it supports (its latest). If the client can't support the
server's answer, it **SHOULD disconnect**. Over HTTP, every subsequent request **MUST**
carry `MCP-Protocol-Version: <version>`; a missing header makes the server assume
`2025-03-26` for back-compat, and an *unsupported* version **MUST** get a `400`.

### Capability negotiation

Capabilities declared at init decide which optional features are live. Both sides **MUST**
only use what was negotiated.

| Side | Capability | Meaning |
|------|-----------|---------|
| Server | `tools` / `resources` / `prompts` | offers that primitive |
| Server | `logging`, `completions`, `tasks` | structured logs / argument autocomplete / task-augmented requests |
| Client | `roots` | exposes filesystem roots |
| Client | `sampling` | server may ask the client's LLM to generate |
| Client | `elicitation` | server may ask the user for input mid-call |
| Client | `tasks` | supports task-augmented requests |

Sub-flags: `listChanged` (tools/resources/prompts can notify on list changes) and
`subscribe` (resources only). The `tasks` capability on both sides reflects the protocol's
shift toward durable, pollable long-running work.

### Backward compatibility in practice

- **Transport bridging:** a server can host *both* the deprecated SSE endpoints and the new
  `/mcp` endpoint; a client probes by POSTing `initialize` and falling back to a `GET` SSE
  handshake on `400/404/405`.
- **Stateless migration (forward):** under SEP-2575 a server **MAY** keep `initialize` for
  legacy clients while exposing `server/discover` + per-request metadata for new ones; HTTP
  clients detect support via a `400 Unsupported protocol version`, stdio clients probe with
  `server/discover` first.

**Rule:** pin and advertise specific dated versions, keep one or two prior versions
working, and gate behavior on the negotiated version — never assume the latest.

---

## 7. Reference architectures

### A. Local stdio dev-tool server

Single developer, same machine, no auth. The default for IDE/CLI tooling.

```
┌─ Developer laptop ─────────────────────────────────────────┐
│  Host (Cursor / Claude Desktop / VS Code)                   │
│     └─ MCP client ──spawns──► your-server  (stdio subprocess)│
│            stdin/stdout: JSON-RPC      stderr: logs          │
│                          │                                  │
│                          ▼                                  │
│              local FS / git / docker / local DB             │
└─────────────────────────────────────────────────────────────┘
```
- Transport: **stdio**. State: in-process is fine (one user, one process).
- Watch-outs: never write non-JSON to stdout; log to stderr; respect `roots`.
- Ship as an npx/uvx package the host launches by command.

### B. Remote multi-tenant SaaS MCP server

Many orgs, internet-facing, OAuth, must scale. The Cloudflare-style edge pattern.

```
                         OAuth 2.1 (per-user tokens)
   Many hosts ─────► │ Gateway / Portal │ ──► auth, rate-limit, logging, DLP
 (Claude, Cursor…)   └────────┬─────────┘
                              │  Streamable HTTP  (POST/GET /mcp)
                  ┌───────────▼─────────────┐
                  │  Worker / Lambda fleet   │   (stateless request handlers)
                  └───────────┬─────────────┘
              stateful path:  │            stateless path:
       Durable Object per ────┤            ──► tool logic, no session
       session (own SQL)      │
                              ▼
              external state: per-tenant DB / Redis, keyed by token identity
```
- Transport: **Streamable HTTP** (single `/mcp`). Auth: OAuth 2.1; gateway exchanges
  upstream token for its own.
- State: **prefer stateless handlers**; if a flow needs state, isolate it per-session
  (Durable Object or externalized store) so any instance can serve any other request.
- Scale/reliability: edge or autoscaled fleet, per-request timeouts, idempotent writes,
  long jobs via tasks/polling, rate-limit at the gateway.
- This is the shape that aligns with SEP-2575: self-contained requests, per-request auth.

### C. Internal-API-wrapper server (synthesis)

The most common enterprise build: wrap an existing internal REST/gRPC API as MCP tools so
agents can use it safely. Deployed *inside* the network, behind the company gateway.

```
   Internal AI agents / hosts
            │  Streamable HTTP (mTLS or SSO via gateway)
   ┌────────▼─────────┐
   │  MCP wrapper svc  │   stateless container(s) behind internal LB
   │  ┌─────────────┐  │
   │  │ tools/* map  │──┼──► existing internal REST/gRPC API
   │  │ to API calls │  │        (auth via service account / on-behalf-of token)
   │  └─────────────┘  │
   │  resources/* ─────┼──► read-only catalog (schemas, docs)
   └───────────────────┘
        observability: every tools/call logged, rate-limited, audited
```
- Transport: **Streamable HTTP**, stateless (each tool call → one downstream API call).
  Containerized, horizontally scaled behind an internal LB — no sticky sessions needed.
- Identity: propagate the user's identity to the backend (OAuth on-behalf-of / service
  account) so the API's own authz still applies. **Never** let the wrapper become an authz
  bypass — SEP-2575 explicitly warns every request must be independently authenticated.
- Tool design: don't 1:1 mirror every endpoint. Expose *intent-level* tools
  (`find_overdue_invoices`) over raw CRUD; map writes to idempotent operations; return
  structured errors the model can reason about.
- This composes the stateless handler model (B) with internal auth propagation; it's the
  lowest-risk, highest-leverage server most teams ship first.

---

## Sources

- [MCP Spec — Transports (2025-11-25)](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)
- [MCP Spec — Lifecycle (2025-11-25)](https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle)
- [MCP Spec — Server features / Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
- [SEP-2575 — Make MCP Stateless (Final)](https://modelcontextprotocol.io/seps/2575-stateless-mcp)
- [Cloudflare — Build & deploy Remote MCP servers](https://blog.cloudflare.com/remote-model-context-protocol-servers-mcp/)
- [Cloudflare — Enterprise MCP reference architecture](https://blog.cloudflare.com/enterprise-mcp/)
- [Cloudflare Agents — Build a Remote MCP server (docs)](https://developers.cloudflare.com/agents/guides/remote-mcp-server/)
- [fka.dev — Why MCP deprecated SSE for Streamable HTTP](https://blog.fka.dev/blog/2025-06-06-why-mcp-deprecated-sse-and-go-with-streamable-http/)
- [AWS samples — Stateful vs Stateless MCP architecture](https://deepwiki.com/aws-samples/sample-serverless-mcp-servers/1.1-stateful-vs.-stateless-architecture)
- [Composio — Smithery alternatives (hosting landscape)](https://composio.dev/blog/smithery-alternative)
- [Apify — Deploy Smithery stdio MCP servers (stdio deprecation note)](https://blog.apify.com/smithery-alternative/)
