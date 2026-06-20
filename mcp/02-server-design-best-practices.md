# MCP Server Design Best Practices

*A practical guide to designing the tools, resources, and responses of an MCP server so AI
agents use it reliably and token-efficiently.*

> Audience: an engineer building an MCP server who wants agents (Claude, etc.) to use it
> reliably and cheaply.
>
> Last updated: June 2026. Primary sources: Anthropic's
> [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents),
> [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp),
> the [MCP specification (2025-06-18)](https://modelcontextprotocol.io/specification/2025-06-18/server/tools),
> and the [RAG-MCP paper (arXiv:2505.03275)](https://arxiv.org/abs/2505.03275).

---

## The one idea behind everything

A tool is **a contract between a deterministic system and a non-deterministic agent**.
Anthropic frames tools as "a new kind of software which reflects a contract between
deterministic systems and non-deterministic agents"
([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)).

Traditional software guarantees identical output for identical input. An agent may
hallucinate arguments, refuse a task, or pursue several valid strategies for the same goal.
So you are not designing an API for a programmer who reads docs once and memorizes them.
You are designing for a reader who sees your tool descriptions fresh on every call, has a
finite context budget, and decides *whether and how* to call you based almost entirely on
the words in your schema. Every design decision below follows from that.

---

## 1. Tools vs Resources vs Prompts

MCP defines three server primitives. The split is not about *what data* they touch — it is
about **who decides when they run**
([MCP spec](https://modelcontextprotocol.io/specification/2025-06-18/server/tools),
[Stacktree](https://stacktr.ee/blog/mcp-resources-vs-tools-vs-prompts)).

| Primitive | Control plane | "Who decides when this runs?" | Use it for |
|-----------|---------------|-------------------------------|------------|
| **Tool** | Model-controlled | The model invokes it automatically based on context | Actions and side effects — write, delete, send, publish, query a live system |
| **Resource** | Application-controlled | The host app pulls it in as context by URI | Read-only, addressable data — a file, a DB schema, a config blob |
| **Prompt** | User-controlled | The user explicitly selects it (often a slash command) | Named, repeatable workflows the user kicks off deliberately |

**Decision rules** (from [Stacktree](https://stacktr.ee/blog/mcp-resources-vs-tools-vs-prompts)):

- *Side effect?* → **Tool.** "Anything with a side effect (writing, deleting, sending,
  publishing) belongs here." The MCP spec adds that there **SHOULD** always be a human in
  the loop able to deny a tool invocation
  ([MCP spec](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)).
- *"The client should be able to look this up by an address"?* → **Resource.** Each
  resource has a unique URI, so the host can fetch it without routing a lookup through the
  model.
- *A known workflow the user starts on purpose?* → **Prompt.** A template that wires tools
  and resources together, surfaced as e.g. a slash command.

**Common confusion — the lookup-as-tool anti-pattern.** The most frequent mistake is
modeling every read as a tool. If the host can address the data by URI and the agent does
not need to *reason about whether to fetch it*, it is a resource. Forcing every lookup
through a model-controlled tool call burns a reasoning turn and context tokens for nothing
([Stacktree](https://stacktr.ee/blog/mcp-resources-vs-tools-vs-prompts)).

**A caveat for 2026:** resources and prompts are unevenly supported across MCP clients.
Some hosts surface tools but ignore resources/prompts. If your target client does not
support resources, you may have to expose reads as tools anyway — but design them like
resources (cheap, read-only, well-cached) and annotate them `readOnlyHint: true`.

---

## 2. Tool design (the highest-leverage area)

Tool design is where you win or lose. Anthropic notes that "even small refinements to tool
descriptions can yield dramatic improvements"
([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)).

### 2a. Naming conventions

- **Namespace with a prefix.** Group related tools under a common prefix so the agent can
  see boundaries: `asana_search`, `asana_projects_search`, `jira_search`. Anthropic reports
  that prefix vs. suffix namespacing had "non-trivial effects" on their evals
  ([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)).
- **Name the *intent*, not the endpoint.** `schedule_event` beats `post_calendar_v2`.
- **Be consistent.** `verb_noun` throughout (`create_pipeline`, `list_pipelines`,
  `get_pipeline`). Mixed conventions make the agent guess.
- **Disambiguate parameters too.** Prefer `user_id` over `user`, `image_url` over `image`.

### 2b. Descriptions are instructions to the model

The description is not documentation a human skims once — it is the prompt the model reads
*every time it considers the tool*. Anthropic's mental model: write it as if **onboarding a
new employee** who knows nothing about your system. Make implicit context explicit
([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)).

**Bad description**

```
get_data — gets data for a user.
```

This tells the agent nothing about *which* data, *which* user identifier, what it costs,
when to use it vs. another tool, or what comes back.

**Good description**

```
Fetch a customer's billing history for a date range. Use this when a user asks about
charges, refunds, or invoices for a specific customer.

Requires `customer_id` (the numeric ID from the customer record, NOT their email).
Returns up to 50 transactions per page, newest first; use `cursor` to page.
Do NOT use this to look up a customer by name — call `customer_search` first to get
the `customer_id`.
```

The good version encodes: when to call it, which identifier to pass, the relationship to
other tools, the response shape, and pagination. That last "Do NOT… call X first" line is
how you prevent the agent from misusing the tool or chaining it wrong.

**Evidence this matters more than it looks.** Anthropic found that Claude-optimized tool
descriptions *outperformed human-written ones* on held-out evals for their internal Slack
and Asana tools, and that "precise refinements to tool descriptions" drove Claude
3.5 Sonnet to state-of-the-art on SWE-bench Verified by "dramatically reducing error
rates" ([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)). The
practical takeaway: a description rewrite is often a higher-ROI lever than waiting for a
model upgrade — invest there first.

### 2c. Granularity — the "right altitude" question

> "More tools don't always lead to better outcomes."
> — [Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)

Do not wrap every API endpoint one-to-one. Build tools around **high-impact workflows**,
consolidating the steps an agent would otherwise have to orchestrate itself.

| Too low (one tool per endpoint) | Right altitude |
|---------------------------------|----------------|
| `list_users` + `list_events` + `create_event` | `schedule_event` (does the lookups internally) |
| `read_logs` (dumps everything) | `search_logs` (returns only matching lines + context) |
| `get_user` + `get_orders` + `get_address` to answer one question | one tool that returns the joined view the workflow needs |

The wrong altitude shows up two ways: **too low** (the agent must call five tools and stitch
results, burning turns and tokens) and **too high** (one mega-tool with 20 optional
parameters that the agent can't reason about). Aim for tools that map to *a thing a user
would actually ask for*. ([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents))

### 2d. Parameter design, constraints, defaults, enums

Constrain the input space so the agent literally cannot pass garbage.

- **Use enums** for closed sets. `status: "active" | "paused" | "archived"` beats a free
  string the model has to guess.
- **Set sane defaults.** `limit` defaults to 20, `format` defaults to `"concise"`. The
  agent should get a good result with the *minimum* arguments.
- **Express constraints in JSON Schema**, not just prose: `minimum`, `maximum`, `pattern`,
  `enum`, `required`. The client can validate before the call reaches you.
- **Provide an `outputSchema`** when results are structured. The MCP spec says that if you
  provide one, "Servers **MUST** provide structured results that conform to this schema"
  and clients **SHOULD** validate against it — this gives the agent reliable types to parse
  ([MCP spec](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)).
- **Use tool annotations** to signal behavior: `readOnlyHint`, `destructiveHint`,
  `idempotentHint`, `openWorldHint`. (Clients **MUST** treat annotations from untrusted
  servers as untrusted — they are hints, not guarantees
  ([MCP spec](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)).)

### 2e. The structured checklist for a good tool

Distilled from Anthropic's five principles
([source](https://www.anthropic.com/engineering/writing-tools-for-agents)):

- [ ] **Strategic** — maps to a real workflow, not an arbitrary endpoint.
- [ ] **Namespaced** — clear prefix grouping related tools.
- [ ] **Right altitude** — consolidates multi-step operations; not too low, not too high.
- [ ] **High-signal name + description** — reads like onboarding instructions; says *when*
      to use it and how it relates to other tools.
- [ ] **Constrained inputs** — enums, defaults, required fields, validation in schema.
- [ ] **Meaningful field names** — `name`/`file_type`, not `uuid`/`mime_type`.
- [ ] **Token-bounded output** — pagination/filtering/truncation with sane defaults.
- [ ] **Self-correcting errors** — error text tells the agent what to do next.
- [ ] **Backed by an eval** — there is a held-out task that exercises this tool.

---

## 3. Response design for agents

The response is context the model must now reason over. Design it for **signal per token**.

### Return meaningful fields, not machine identifiers

Prioritize semantic relevance over raw technical completeness. Anthropic's example:
return `name`, `image_url`, and `file_type` instead of `uuid`, `256px_image_url`, and
`mime_type` ([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)).
An opaque UUID costs tokens and gives the model nothing to reason with; a human-readable
name lets it act.

### Let the agent choose verbosity

Expose a `ResponseFormat` enum (`"concise"` | `"detailed"`) so the agent asks for detail
only when it needs it. In Anthropic's Slack example, the detailed response cost 206 tokens
vs. 72 for concise — a ~3× difference for the same call
([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)).

### Structured vs. natural language

- **Structured** (JSON conforming to `outputSchema`) when the agent will parse, filter, or
  pass the result onward. Pair it with a TextContent serialization for backward compat, as
  the spec recommends ([MCP spec](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)).
- **Natural language** when the result is meant to be read and summarized (e.g. a status
  digest). Often the best response is *structured data + a one-line natural-language
  summary*.

### Pagination, truncation, and defaults

- Cap responses. Anthropic restricts Claude Code tool responses to **25,000 tokens by
  default** ([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)).
- Paginate with a `cursor`; never dump an unbounded list.
- When you truncate, **say so and steer the agent**: include a message that encourages
  "many small targeted searches instead of a single broad search"
  ([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)).

### What to omit

Drop internal IDs the agent will never pass back, debug fields, null-heavy objects,
redundant nesting, and anything that doesn't change what the agent does next. Every field
you return is context the model pays for on this turn *and* carries forward.

---

## 4. The "too many tools" problem

Tool definitions are not free — they sit in context *before the agent reads the user's
request*. At scale this dominates the context budget and degrades tool selection.

### The evidence

- **Schema bloat.** When an agent has dozens of tools, the JSON schemas can occupy
  **40–50% of the available context window**, raising cost and latency and causing "LLM
  confusion" where the model is less accurate because it is drowning in irrelevant
  definitions (reported via
  [The New Stack](https://thenewstack.io/how-to-reduce-mcp-token-bloat/) /
  [AgentMarketCap](https://agentmarketcap.ai/blog/2026/04/08/mcp-context-bloat-enterprise-scale-tool-definitions-agent-context-budget)).
- **Accuracy collapse.** The RAG-MCP paper measured tool-selection accuracy of **13.62%**
  when all tools were dumped in the prompt, vs. **43.13%** when only the relevant tools were
  retrieved first — more than triple, with **>50% fewer prompt tokens**
  ([arXiv:2505.03275](https://arxiv.org/abs/2505.03275)).
- **It's a cliff, not a slope.** Practitioner evals report near-perfect selection around
  ~10 tools, then a sharp drop rather than graceful degradation past a threshold
  ([Atlassian](https://www.atlassian.com/blog/development/mcp-compression-preventing-tool-bloat-in-ai-agents)).

### Guidance on tool count

Sources differ, so treat this as a range, not a hard rule:

- Practitioner guidance commonly lands at **10–15 tools** per server before confusion sets
  in ([Atlassian](https://www.atlassian.com/blog/development/mcp-compression-preventing-tool-bloat-in-ai-agents)).
- A looser ceiling people cite is **~20–30 tools** before you should seriously consider
  splitting or dynamic loading.

Both point the same direction: **keep the active toolset small.**

### How to keep it small

1. **Specialize servers.** One server per domain (a "billing" server, a "warehouse"
   server) rather than one server exposing everything.
2. **Consolidate** at the right altitude (Section 2c) — fewer, more powerful tools.
3. **Dynamic / progressive tool loading.** Don't load all definitions up front. Approaches:
   - *Retrieval (RAG-MCP):* semantically retrieve only the relevant tool definitions for
     the current query before calling the model
     ([arXiv:2505.03275](https://arxiv.org/abs/2505.03275)).
   - *On-demand discovery (MCP-Zero / code execution):* let the agent load the full schema
     for a tool only when it decides it needs that tool.

---

## 5. Token efficiency: tool definitions and code execution with MCP

Two things consume context: **tool definitions** (loaded up front) and **intermediate
results** (every tool output passing through the model). Anthropic's *Code execution with
MCP* tackles both ([source](https://www.anthropic.com/engineering/code-execution-with-mcp)).

**The problem, in their words:**

- *Definition overload* — "As agents connect to more tools… they'll need to process
  hundreds of thousands of tokens before reading a request."
- *Intermediate-result bloat* — a 2-hour meeting transcript that an agent retrieves and
  then writes to another system passes through context **twice**, potentially "an
  additional 50,000 tokens."

**The fix:** present MCP servers as **code APIs / files** the agent can call from a code
execution environment, instead of direct tool calls.

| Technique | What it does |
|-----------|--------------|
| **Progressive disclosure** | Agent reads tool definitions on-demand by exploring the filesystem, "rather than reading them all up-front." |
| **In-environment filtering** | Agent filters/transforms results in code *before* returning them, keeping intermediate data out of context. |
| **Control flow in code** | Loops, conditionals, error handling use familiar code patterns instead of chaining individual tool calls. |
| **Privacy by default** | Intermediate results "stay in the execution environment by default," so sensitive data needn't enter the model. |

**The headline number:** for a workflow that pulled a transcript through context twice,
this approach "reduces the token usage from **150,000 tokens to 2,000 tokens — a time and
cost saving of 98.7%**"
([Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp)).

**What this means for *your* server design**, even if your client doesn't do code
execution yet:

- Make tools **filterable** so the agent never has to pull a big blob to extract one field
  (`search_logs`, not `read_logs`).
- Keep tool definitions **lean** — long verbose descriptions multiply across every tool in
  context. Be thorough but not padded.
- Return **references** (resource links / IDs) the agent can re-fetch, rather than inlining
  large payloads it may not need.

---

## 6. Error handling and resilience

Errors are not failures to hide — they are a chance to *steer the agent toward recovery*.

### Two error channels (use the right one)

The MCP spec defines two mechanisms
([source](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)):

1. **Protocol errors** (JSON-RPC `error`) — for "unknown tool," "invalid arguments,"
   server faults. These signal *the call itself was malformed*.
2. **Tool execution errors** — returned in a *normal result* with `isError: true` — for API
   failures, bad input data, business-logic errors. Crucially, these **come back to the
   model as content**, so the model can read them and try again.

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [{
      "type": "text",
      "text": "Failed: customer_id 9182 not found. It must be the numeric ID from the customer record. Call customer_search with the customer's name or email to get a valid customer_id, then retry."
    }],
    "isError": true
  }
}
```

Notice the error *tells the agent how to self-correct*. Anthropic's guidance: provide
actionable guidance rather than opaque tracebacks
([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)). Contrast:

| Bad error | Good error |
|-----------|-----------|
| `Error: 500` | `Upstream timed out after 30s. This is transient — retry once; if it fails again, the warehouse may be syncing (try in ~2 min).` |
| `KeyError: 'customer_id'` | `Missing required argument customer_id (numeric). Call customer_search first to obtain it.` |
| `Invalid input` | `date_from must be ISO-8601 (YYYY-MM-DD); you passed "last week". Resolve relative dates before calling.` |

### Partial failures and rate limits

- **Partial success:** return what succeeded *plus* a structured list of what failed and
  why, so the agent can retry only the failures instead of redoing the batch.
- **Rate limits:** signal them explicitly and tell the agent to back off — e.g.
  `Rate limit hit (100 req/min). Retry after 12s.` A bare `429` makes the agent guess.
- **Validate and sanitize.** The spec requires servers to validate all inputs, enforce
  access controls, rate-limit invocations, and sanitize outputs
  ([MCP spec](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)).

---

## 7. Evaluating your MCP server

You cannot tune tools you don't measure. Anthropic's loop is **Prototype → Evaluate →
Collaborate** ([source](https://www.anthropic.com/engineering/writing-tools-for-agents)):

1. **Prototype** the tools locally (e.g. in Claude Code with the API docs).
2. **Evaluate** with realistic, verifiable tasks.
3. **Collaborate** — have Claude analyze transcripts and *rewrite its own tools*. Their
   Claude-optimized tools beat the human-written versions on held-out tests.

### What makes a strong eval task

| Strong (mirrors real workflows, multi-call) | Weak (toy, single-call) |
|---------------------------------------------|--------------------------|
| "Customer ID 9182 reported being charged three times. Find all relevant log entries and determine whether other customers were affected." | "Search payment logs for `purchase_complete`." |
| "Schedule a meeting with Jane next week, attach notes from the last planning meeting, and reserve a conference room." | "List my calendar events." |

Pair each prompt with a **verifiable outcome** — exact comparison or Claude-as-judge.

### Metrics beyond accuracy

Collect, per task ([Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents)):

- **Accuracy** — did it get the right answer?
- **Tool-call count** — fewer is usually better; a spike reveals a missing consolidation.
- **Token consumption** — where is the context budget going?
- **Runtime / latency.**
- **Error rate** — which tools get misused, and how?

Read the *transcripts*, not just the scores. The failure patterns (wrong tool picked,
parameter misread, broad search where a targeted one was needed) are your prioritized list
of description/altitude fixes.

---

## 8. Worked example: a BAD design vs. a GOOD redesign

Hypothetical internal API: an **order-management service** for a support agent. The agent's
job: *"Customer says their order arrived damaged — find the order and issue a refund."*

### BAD design — one tool per endpoint, leaky fields, opaque errors

```json
{
  "name": "get",
  "description": "Gets a record.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "table": { "type": "string" },
      "id":    { "type": "string" }
    },
    "required": ["table", "id"]
  }
}
```
```json
{
  "name": "query",
  "description": "Runs a query.",
  "inputSchema": {
    "type": "object",
    "properties": { "sql": { "type": "string" } },
    "required": ["sql"]
  }
}
```
```json
{
  "name": "post_refund",
  "description": "Posts a refund.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "oid":    { "type": "string" },
      "amt":    { "type": "number" },
      "rsn":    { "type": "string" }
    },
    "required": ["oid", "amt"]
  }
}
```

Sample BAD response from `get`:

```json
{
  "uuid": "a3f9c2e1-...", "cust_uuid": "f1c8...", "ln_items": [...],
  "amt_cents": 4999, "mime": null, "_v": 3, "created_ts": 1718800000
}
```

Sample BAD error: `{"error": "constraint violation"}`

**Everything wrong here, mapped to the principles it breaks:**

| Anti-pattern | Principle violated |
|--------------|--------------------|
| `get`/`query` are generic endpoint wrappers; the agent must compose raw SQL and stitch results | §2c right altitude |
| `query` accepting arbitrary `sql` is an injection/exfiltration hazard | §6 / spec security |
| Names `get`, `query`, `post_refund`; params `oid`, `amt`, `rsn` | §2a naming |
| One-line descriptions say nothing about *when* to use the tool | §2b descriptions |
| Response leaks `uuid`, `cust_uuid`, `amt_cents`, `_v`, `mime` | §3 high-signal fields |
| No pagination, no response cap | §3 / §4 token efficiency |
| `constraint violation` gives the agent no recovery path | §6 errors |
| `amt` not required to match the order; nothing stops a wrong refund | §2d constraints |
| A pure read (`get`) routed through a model-controlled tool | §1 lookup-as-tool |

### GOOD redesign — workflow tools, meaningful fields, self-correcting errors

```json
{
  "name": "orders_search",
  "title": "Search orders",
  "description": "Find a customer's orders. Use this FIRST when a customer reports a problem and you only know their email or name — it returns order_id values you can pass to other order tools. Returns the 20 most recent matching orders, newest first; pass `cursor` to page. Do NOT guess an order_id; always search for it.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "customer_email": { "type": "string", "format": "email",
        "description": "Customer's email. Provide this OR customer_name." },
      "customer_name":  { "type": "string",
        "description": "Full name. Provide this OR customer_email." },
      "status": { "type": "string",
        "enum": ["any", "delivered", "shipped", "processing", "cancelled"],
        "default": "any" },
      "limit":  { "type": "integer", "minimum": 1, "maximum": 50, "default": 20 },
      "cursor": { "type": "string", "description": "From a previous response's next_cursor." }
    },
    "anyOf": [{ "required": ["customer_email"] }, { "required": ["customer_name"] }]
  },
  "annotations": { "readOnlyHint": true, "openWorldHint": false }
}
```

```json
{
  "name": "orders_get",
  "title": "Get order detail",
  "description": "Get full detail for one order, including line items and refund eligibility. Use after orders_search to confirm the item and amount before issuing a refund. Returns human-readable fields; amounts are decimal in the order's currency.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string",
        "description": "The order_id from orders_search (e.g. 'ORD-10293'), NOT an internal UUID." }
    },
    "required": ["order_id"]
  },
  "outputSchema": {
    "type": "object",
    "properties": {
      "order_id":        { "type": "string" },
      "customer_name":   { "type": "string" },
      "status":          { "type": "string" },
      "order_total":     { "type": "number" },
      "currency":        { "type": "string" },
      "refundable_total":{ "type": "number",
        "description": "Max amount still eligible for refund." },
      "items": { "type": "array", "items": {
        "type": "object",
        "properties": {
          "name":      { "type": "string" },
          "quantity":  { "type": "integer" },
          "price":     { "type": "number" }
        }, "required": ["name", "quantity", "price"] } }
    },
    "required": ["order_id", "status", "refundable_total", "currency", "items"]
  },
  "annotations": { "readOnlyHint": true }
}
```

```json
{
  "name": "orders_issue_refund",
  "title": "Issue a refund",
  "description": "Refund all or part of an order. Call orders_get first to read `refundable_total`; `amount` must not exceed it. This MODIFIES billing and emails the customer — confirm the amount and reason before calling. Idempotent on `idempotency_key`: reusing the same key will not double-refund.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string", "description": "From orders_search / orders_get." },
      "amount":   { "type": "number", "exclusiveMinimum": 0,
        "description": "Decimal amount in the order's currency. Must be <= refundable_total." },
      "reason":   { "type": "string",
        "enum": ["damaged", "not_received", "wrong_item", "customer_request", "other"] },
      "idempotency_key": { "type": "string",
        "description": "Unique per logical refund; reuse to safely retry without double-charging." }
    },
    "required": ["order_id", "amount", "reason", "idempotency_key"]
  },
  "annotations": { "readOnlyHint": false, "destructiveHint": true, "idempotentHint": true }
}
```

**GOOD response** from `orders_get` (high-signal, no leaked internals):

```json
{
  "order_id": "ORD-10293",
  "customer_name": "A. Rivera",
  "status": "delivered",
  "order_total": 49.99,
  "currency": "USD",
  "refundable_total": 49.99,
  "items": [{ "name": "Ceramic Mug (Blue)", "quantity": 1, "price": 49.99 }]
}
```

**GOOD error** from `orders_issue_refund` when the amount is too high — note it gives the
agent the exact number to retry with:

```json
{
  "result": {
    "content": [{
      "type": "text",
      "text": "Refund rejected: amount 79.99 exceeds refundable_total 49.99 for ORD-10293. Re-issue with amount <= 49.99. To check eligibility, call orders_get first."
    }],
    "isError": true
  }
}
```

**How the redesign maps fix → principle:**

| Fix | Principle |
|-----|-----------|
| Three workflow tools (`search → get → refund`) replace generic `get`/`query` | §2c altitude |
| Removed arbitrary-SQL `query`; reads are scoped, read-only, annotated | §1 / §6 security |
| `orders_*` namespace; `order_id`, `amount`, `reason` | §2a naming |
| Descriptions say *when* to call and *which tool comes first* | §2b descriptions |
| `enum` reasons, `maximum` limit, `exclusiveMinimum` amount, `idempotency_key` | §2d constraints |
| `outputSchema` returns `customer_name`/`items`, not `uuid`/`amt_cents`/`_v` | §3 high-signal |
| `limit`/`cursor` pagination with defaults | §3 / §4 token efficiency |
| Error states the exact corrective action and value | §6 self-correcting errors |
| Annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`) declare behavior | §2d / spec |

---

## TL;DR checklist for a server that agents use well

- [ ] Pick the right primitive: **side effect → tool, addressable read → resource, named
      workflow → prompt.**
- [ ] **Few, powerful, well-named tools** at the workflow altitude — aim small (10–15;
      certainly review past ~20–30), specialize servers, consider dynamic loading.
- [ ] **Descriptions written as onboarding instructions** — when to use, which params,
      relationships to other tools. This often beats waiting for a model upgrade.
- [ ] **Constrain inputs** (enums, defaults, validation) and **declare output schemas.**
- [ ] **Token-efficient responses** — meaningful fields, concise/detailed toggle,
      pagination, truncation that steers the agent.
- [ ] **Self-correcting errors** — say what went wrong *and what to do next*; signal rate
      limits and partial failures explicitly.
- [ ] **Evaluate with realistic multi-call tasks**, track tool-count/tokens/errors, and let
      Claude help rewrite its own tools.

---

## Sources

- Anthropic — [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents)
- Anthropic — [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp)
- Anthropic — [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- MCP specification (2025-06-18) — [Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)
- Gan & Sun — [RAG-MCP: Mitigating Prompt Bloat in LLM Tool Selection (arXiv:2505.03275)](https://arxiv.org/abs/2505.03275)
- Stacktree — [MCP resources vs tools vs prompts: when to use each](https://stacktr.ee/blog/mcp-resources-vs-tools-vs-prompts)
- Atlassian — [MCP Compression: Preventing tool bloat in AI agents](https://www.atlassian.com/blog/development/mcp-compression-preventing-tool-bloat-in-ai-agents)
- The New Stack — [10 strategies to reduce MCP token bloat](https://thenewstack.io/how-to-reduce-mcp-token-bloat/)
- AgentMarketCap — [MCP's Context Bloat Crisis](https://agentmarketcap.ai/blog/2026/04/08/mcp-context-bloat-enterprise-scale-tool-definitions-agent-context-budget)
