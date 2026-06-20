# Building MCP Servers by Category

A practical guide to the distinct design patterns for different *types* of MCP
server. You already know what you want to expose — a database, an API, internal
tools, a browser. This picks the right pattern for that category, with current
real-world examples and the failure modes each one hits.

The through-line across every category: **tools are not API endpoints.** An MCP
tool is a unit of agent reasoning. The agent has a finite context window, can't
see your docs, and decides what to call from a one-line description. The servers
that work well are the ones designed for *that* consumer, not for a human reading
Swagger.

> Sources are linked inline. CVE numbers in §5 were each verified against the
> disclosing vendor or the GitHub Security Advisory before being cited — do not
> trust the attribution of a CVE number you see floating around; several similar
> git/filesystem MCP CVEs exist and are easy to conflate (see the note in §5).

---

## The cross-cutting principle: design for the agent, not the API

Two industry write-ups frame everything below.

**Block** (60+ internal MCP servers) argues you should work *top-down from the
workflow* you want to automate and design tools backwards from it — not bottom-up
from your API surface. Their Square MCP server exposes **30+ APIs and 200+
endpoints behind only 3 MCP tools** using a "layered tool pattern" (a `type` +
`method` dispatch rather than one tool per endpoint). For databases they
explicitly prefer **a single generic `query_database` / SQL tool over
one-tool-per-query**, because it gives the model raw access and lets it do
analytics on the fly instead of chaining narrow calls.
([Block's Playbook for Designing MCP Servers](https://engineering.block.xyz/blog/blocks-playbook-for-designing-mcp-servers),
[Build MCP Tools Like Ogres… With Layers](https://engineering.block.xyz/blog/build-mcp-tools-like-ogres-with-layers))

**Datadog** shipped a thin "wrap every API as a tool" v1, watched agents fail on
it, and rebuilt almost everything. Their lessons: *flexible* tools (one
well-designed tool replaces several narrow ones), opt-in **toolsets** so the
default context stays small, and **token-efficient responses** (CSV/TSV is ~50%
smaller than JSON for tabular data; paginate by token budget, not row count).
([Designing MCP tools for agents — Datadog](https://www.datadoghq.com/blog/engineering/mcp-server-agent-tools/))

Keep both in mind: **right altitude** (not too granular, not too coarse) and
**token discipline** (every tool definition and every result competes for context).

---

## 1. Database / data-warehouse MCP

**What it's for.** Letting an agent query a SQL database or warehouse —
exploratory analysis, debugging, reporting, sometimes writes/migrations.

**The design pattern.**
- **Generic query tools, not per-query tools.** This is the Block lesson: expose
  `execute_query` (or a read-only/mutation pair) that takes arbitrary SQL, plus a
  handful of schema-introspection tools. Do **not** generate one tool per saved
  query or per table — it explodes the tool count and removes the agent's ability
  to compose.
- **Split read from write by *transaction mode*, not by trust.** The robust
  pattern is an `execute_readonly_query` that runs inside a `READ ONLY`
  transaction (the database enforces it — `SET TRANSACTION READ ONLY`), and a
  separate, gated `execute_mutation_query`. Don't try to block destructive SQL by
  string-matching for `DROP`/`DELETE` — that's defeated by comments, casing, and
  CTEs. Push the guarantee into the engine: a read-only role, a read replica, or a
  read-only transaction.
- **Schema introspection as first-class tools:** `list_schemas`,
  `list_objects(schema)`, `describe_object(name)`. The agent needs cheap ways to
  learn the schema before it writes SQL, instead of `SELECT *`-ing to discover columns.
- **Result-size limits are mandatory.** Cap rows *and* bytes/tokens. Auto-append
  `LIMIT`, truncate wide columns, and return a "N more rows omitted" marker.
  Returning a 50k-row result set will blow the context window and the bill.

**Key challenges.** Query-safety enforcement (do it in the engine, not regex);
result-size blowups; long-running / locking queries (set a `statement_timeout`);
multi-statement injection; and credential scope (give the server its own
least-privilege DB role).

**Real examples.**
- [`crystaldba/postgres-mcp`](https://github.com/crystaldba/postgres-mcp) —
  "Postgres MCP Pro." Has an explicit **Restricted (read-only) vs Unrestricted**
  access mode; read-only mode runs queries in read-only transactions. Also ships
  `explain_query` / hypothetical-index analysis — a good example of a tool at the
  right altitude (the agent asks "is this query slow?" not "run EXPLAIN").
- [`ClickHouse/mcp-clickhouse`](https://github.com/ClickHouse/mcp-clickhouse) —
  official; **read-only mode recommended**, results as JSON. ClickHouse's own
  write-up on warehouse MCPs is a good primer:
  [MCP and Data Warehouses](https://clickhouse.com/resources/engineering/mcp-data-warehouse-everthing-you-need-to-know).
- [`alexander-zuev/supabase-mcp-server`](https://github.com/alexander-zuev/supabase-mcp-server)
  (community) and the [official Supabase MCP](https://supabase.com/blog/mcp-server) —
  Supabase adds project/auth/storage management on top of SQL, and added
  **read-only by default** plus per-project scoping after early write-safety concerns.
- [`LucasHild/mcp-server-bigquery`](https://github.com/LucasHild/mcp-server-bigquery) —
  schema exploration + query against BigQuery.

**Tool surface (sketch).**
```jsonc
// Read path — DB enforces read-only, not a regex
execute_readonly_query(sql: string, max_rows?: int = 100)
  -> { columns, rows, truncated: bool, row_count }

// Write path — separate, gated, often disabled by default
execute_mutation_query(sql: string, confirm: bool)   // requires explicit confirm

// Schema introspection — cheap discovery before SQL
list_schemas() -> [name]
list_objects(schema: string, type?: "table"|"view") -> [object]
describe_object(schema: string, name: string)
  -> { columns: [{name, type, nullable}], indexes, constraints }

// Performance (Postgres Pro example)
explain_query(sql: string) -> { plan, estimated_cost }
```

---

## 2. REST-API-wrapper MCP

**What it's for.** Wrapping an existing HTTP API (internal or third-party) so an
agent can use it. **The most common kind people build** — and the one most often
built wrong.

**The design pattern — get the altitude right.**
- **Do NOT map endpoints 1:1 to tools.** A 200-endpoint API becomes 200 tools,
  each eating context, and the agent mis-routes between near-duplicates. This is
  the exact mistake Datadog made in v1.
- **Map to *workflows*, not routes.** Combine the 3–4 calls a real task needs into
  one tool (Block's "top-down" rule). Or use a **layered/dispatch tool**: a small
  number of tools (`search`, `get`, `act`) that take a `resource`/`method`
  parameter — Square's 3-tools-for-200-endpoints pattern.
- **Offer opt-in toolsets** so the default install is small and the agent only
  loads the surface it needs (Datadog).
- **Auth passthrough.** Prefer OAuth 2.1 for hosted servers (the agent's identity
  flows through); for local/internal, read a token from env. Never bake a shared
  god-token into the server.
- **Pagination & errors are *your* job, not the agent's.** Agents are bad at
  cursor-chasing and at parsing deep error JSON. Auto-follow a bounded number of
  pages up to a token budget; translate HTTP errors into short, actionable
  messages ("rate limited, retry in 30s" — not a 40-line stack of nested JSON).
- **Trim responses.** Return the fields the task needs; drop hypermedia links,
  envelopes, and audit metadata.

**Key challenges.** Tool-count explosion; nested-JSON responses agents can't
parse; pagination; error mapping; auth scoping; rate limits.

**Real example.** The cleanest reference is Datadog's own engineering write-up
([Designing MCP tools for agents](https://www.datadoghq.com/blog/engineering/mcp-server-agent-tools/))
plus their [`datadog-labs/mcp-server`](https://github.com/datadog-labs/mcp-server),
which shows toolsets and trimmed responses in practice. For a community pattern of
"wrap one API," see [`GeLi2001/datadog-mcp-server`](https://github.com/GeLi2001/datadog-mcp-server)
(a thinner wrapper — useful as the *contrast* to the official one).

**Tool surface (sketch).**
```jsonc
// WRONG: one tool per endpoint
get_user(); list_users(); get_user_orders(); get_order(); list_order_items(); ... // x200

// RIGHT (workflow altitude): collapse the common task into one tool
get_customer_overview(customer_id) -> { profile, recent_orders, lifetime_value }

// RIGHT (layered/dispatch, Square-style): few tools, many resources
api_search(resource: "orders"|"customers"|..., filters: {...})
api_get(resource: string, id: string, fields?: [string])
api_act(resource: string, action: string, payload: {...})  // writes, gated
```

---

## 3. SaaS-product MCP

**What it's for.** Exposing *your own product* to agents — what Linear, Notion,
Stripe, Sentry, Atlassian, and Figma have shipped. The agent becomes another
first-class client of your product.

**The design pattern.**
- **Hosted + OAuth 2.1, multi-tenant.** Modern product MCP servers are remote and
  use OAuth 2.1 rather than static bearer tokens, so the agent acts *as the
  logged-in user* with that user's permissions. Linear runs at `mcp.linear.app/mcp`
  (built with Cloudflare + Anthropic); Stripe at `mcp.stripe.com`. The server is
  multi-tenant: one deployment, many orgs, isolation enforced by the OAuth
  identity. ([Linear MCP](https://linear.app/docs/mcp),
  [Stripe MCP docs](https://docs.stripe.com/mcp))
- **Separate read tools from action tools**, and gate actions. Stripe explicitly
  recommends **human confirmation of write tools** and warns about prompt-injection
  when combining with other servers (a write tool + a server that reads untrusted
  text = an attacker can drive the writes). Use restricted/limited API keys for the
  agent path.
- **Curate the product surface — don't dump the whole API.** Ship the verbs that
  map to how users actually work ("create issue," "update status," "comment"), not
  every CRUD permutation. Linear's server is ~25 tools tuned for agent workflows,
  not a mirror of its GraphQL schema.
- **The Notion lesson — agents love Markdown, hate nested JSON.** Notion's original
  block model returns deeply nested `children` JSON; agents burned context on chatty
  round-trips and mangled the structure. Notion built a **"Notion-flavored Markdown"**
  representation so the agent creates/reads/updates pages as enhanced Markdown
  instead of block JSON — far fewer tokens, far fewer tool calls, and the model
  manipulates it naturally. Generalize this: **return your content as Markdown/text,
  reserve JSON for genuinely structured fields.**
  ([Notion's hosted MCP server: an inside look](https://www.notion.com/blog/notions-hosted-mcp-server-an-inside-look),
  [`makenotion/notion-mcp-server`](https://github.com/makenotion/notion-mcp-server))

**Key challenges.** Tenant isolation; OAuth scope mapping; write-safety &
confirmation; prompt-injection through your own product's user-generated content;
choosing the *product* surface (not the API surface); content representation
(Markdown vs JSON).

**Tool surface (sketch).**
```jsonc
// Reads
list_issues(filter: {assignee, state, project}) -> [issue]   // trimmed fields
get_page(id) -> { title, content_markdown }                  // Notion lesson

// Actions — distinct verbs, confirmation on writes
create_issue(title, description_markdown, project_id, assignee?)
update_issue(id, { state?, assignee?, ... })
update_page(id, content_markdown)         // Markdown in, not nested block JSON
// payment-grade action: require explicit confirm + restricted key
create_refund(charge_id, amount, confirm: true)
```

---

## 4. Internal-tools / DevOps MCP

**What it's for.** Wrapping internal scripts, CI/CD, and infra ops — deploys,
Terraform plans, Kubernetes actions, runbooks. The highest-blast-radius category:
a wrong tool call can take down production.

**The design pattern.**
- **Default to read/observe; gate every mutation.** Make the safe surface (status,
  logs, plan, diff) always-on, and put anything that *changes* infra behind an
  explicit switch. The HashiCorp **Terraform MCP server** does exactly this:
  destructive tools are **disabled by default** and require `ENABLE_TF_OPERATIONS=true`;
  its `create_run` supports `plan_and_apply` that **plans first and applies only if
  approved**. ([Terraform MCP server reference](https://developer.hashicorp.com/terraform/mcp-server/reference))
- **Dry-run as a distinct tool / mode.** Expose `plan`/`diff`/`--dry-run` as the
  default and `apply` as a separate, confirmed step. Never let one tool both decide
  and execute without a checkpoint.
- **Approval gates / human-in-the-loop.** For anything irreversible, the tool
  returns a plan and *requires a confirm token* or routes through an existing
  approval system (PR, change ticket). MCP **elicitation** (server-initiated
  confirmation prompts) is the spec-native way to do this.
- **Least privilege + audit.** The server runs with a scoped service account, logs
  every call, and ideally enforces allow-lists of environments/namespaces. Assume
  prompt-injection can reach it (see §5/§3) and design so the *worst* a hijacked
  agent can do is bounded.

**Key challenges.** Blast radius; irreversible actions; secret handling; auditing;
preventing an agent (or an injected instruction) from running `terraform destroy`
or `kubectl delete` on prod.

**Real examples.**
[Terraform MCP server](https://developer.hashicorp.com/terraform/mcp-server/reference)
(approval-gated, ops disabled by default); the
[`rohitg00/awesome-devops-mcp-servers`](https://github.com/rohitg00/awesome-devops-mcp-servers)
list for Kubernetes/CI/CD/cloud servers; and roll-your-own wrappers around internal
runbooks. For Kubernetes management with environment scoping, see Qovery's MCP
([overview](https://www.infoworld.com/article/4096223/10-mcp-servers-for-devops.html)).

**Tool surface (sketch).**
```jsonc
// Always-on: observe
get_deploy_status(service, env) -> { version, health }
get_logs(service, env, since) -> [line]      // tail + token cap
plan_change(spec) -> { diff, blast_radius_summary }   // dry-run, no mutation

// Gated: mutate — separate tool, confirm token, scoped env
apply_change(plan_id, confirm: true, env: "staging")  // prod requires extra gate
// server config: destructive tools off unless ENABLE_OPS=true
```

---

## 5. Filesystem / code / Git MCP

**What it's for.** Local dev-tool servers — read/write files, run git, navigate a
repo. Powerful and *dangerous*, because they run on the developer's machine, often
with the user's full privileges.

**The design pattern.**
- **Sandbox by allowed roots.** Constrain every operation to an explicit list of
  allowed directories. The hard part is doing this *correctly*.
- **Validate paths the right way.** Resolve the *real* path (`realpath`, resolving
  symlinks) and check it's `startsWith(allowed + separator)` — **not** a raw string
  prefix, and **after** symlink resolution. The CVEs below are exactly what happens
  when this is done naively.
- **No shell string-building.** Pass git arguments as an argv array to a non-shell
  exec; never interpolate user/agent input into a shell command, and never pass
  agent-controlled values where they can be read as flags.

**Cautionary CVEs (each verified against the disclosing source).** These are real
and recent — they're the reason this category needs care.

- **Anthropic Filesystem MCP — "EscapeRoute":**
  - **CVE-2025-53110** (CVSS 7.3) — directory-containment bypass via a *naive prefix
    string check*: a path like `/private/tmp/allowed_dir_evil` passes because it
    starts with the allowed prefix. Lets the agent read/write outside scope.
  - **CVE-2025-53109** (CVSS 8.4) — symlink bypass chaining to **code execution**: a
    crafted symlink defeats the check (falls back to parent-dir validation), giving
    full read/write and RCE via e.g. dropped launch agents / cron.
  - Disclosed by **Cymulate**; affects versions before the fix; patched in npm
    **2025.7.1** (July 2025).
    ([Cymulate write-up](https://cymulate.com/blog/cve-2025-53109-53110-escaperoute-anthropic/),
    [Embrace The Red](https://embracethered.com/blog/posts/2025/anthropic-filesystem-mcp-server-bypass/))

- **Anthropic reference Git MCP server (the "Cyata" flaws):** three issues reported
  by **Cyata**, enabling file access / RCE via prompt injection —
  **CVE-2025-68143** (path traversal: `git_init` accepted arbitrary paths, e.g. into
  `~/.ssh`), **CVE-2025-68144** (argument injection: `git_diff`/`git_checkout` passed
  agent-controlled values straight to the git CLI — injecting `--output=…` could
  overwrite files), and **CVE-2025-68145** (repo-path restriction bypass). Chained
  with the filesystem server + git smudge/clean filters → RCE from a poisoned README
  or issue. Anthropic shipped **2025.12.18**, which hardened path validation, fixed
  argument handling, and **removed `git_init` entirely**.
  ([OSV.dev — CVE-2025-68143](https://osv.dev/vulnerability/CVE-2025-68143) (primary record),
  [The Vulnerable MCP Project — full RCE chain](https://vulnerablemcp.info/vuln/cve-2025-68145-anthropic-git-mcp-rce-chain.html),
  [The Hacker News](https://thehackernews.com/2026/01/three-flaws-in-anthropic-mcp-git-server.html))

- **Do not conflate these with `@cyanheads/git-mcp-server`** — a *different,
  community* git server — **CVE-2025-53107**, a command-injection flaw (input passed
  to `child_process.exec` with shell metacharacters), disclosed by **@dellalibera**
  and fixed in **2.1.5**. Same category, different project, different reporter.
  ([GitHub Advisory GHSA-3q26-f695-pp76](https://github.com/advisories/GHSA-3q26-f695-pp76))

> Attribution caveat: the three buckets above (filesystem EscapeRoute / Anthropic
> git / cyanheads git) are routinely mixed up in secondary write-ups. The CVE→project→
> reporter mapping here was checked against the disclosing vendor or the GitHub
> Security Advisory. If you cite one, link the primary source, not a roundup.

**Real example (the canonical one).**
[`modelcontextprotocol/servers` — filesystem](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem)
(use a *current* version; pre-2025.7.1 is vulnerable) and the reference git server in
the same repo (use ≥ 2025.12.18).

**Tool surface (sketch).**
```jsonc
// All paths resolved to realpath, then checked against allowed roots
read_file(path) ; write_file(path, content) ; list_directory(path)
move_file(source, destination) ; search_files(root, pattern)
// Git: argv array, no shell, no agent-controlled flags
git_status(repo) ; git_diff(repo, ref?) ; git_log(repo, n)
// NOTE: git_init was REMOVED from the reference server post-CVE-2025-68143
```

---

## 6. Browser / automation MCP

**What it's for.** Driving a real browser — navigate, click, fill forms, scrape,
test, do multi-step web tasks. The defining trait of this category is **state**:
there's a live browser/page that persists between tool calls.

**The design pattern.**
- **Statefulness is the core design problem.** Unlike a stateless DB query, a click
  only makes sense relative to the current page. The server holds a live session;
  tools operate on "the current page." This means session lifecycle (open/close),
  and ideally persistent contexts (cookies/login) so the agent doesn't re-auth every
  step.
- **Accessibility snapshot over screenshots.** Microsoft's **Playwright MCP** is the
  reference: it drives the page via a **structured accessibility tree**, not pixels.
  Each interactive element gets a stable `ref`; the agent clicks by ref. This is
  ~200–400 tokens per snapshot vs thousands for a DOM dump or screenshot, it's
  deterministic, and it needs no vision model.
  ([`microsoft/playwright-mcp`](https://github.com/microsoft/playwright-mcp),
  [Playwright MCP docs](https://playwright.dev/mcp/introduction))
- **One snapshot → many actions.** The loop is: `navigate` → `snapshot` (get refs) →
  `click/type` by ref → re-`snapshot`. Tools take a human-readable element
  description *plus* the ref so the model's intent is auditable.
- **Persistent vs isolated profiles.** Playwright MCP keeps login state in a
  persistent user-data dir by default (stateful), with an isolated mode for clean
  runs. Pick based on whether the workflow needs to stay logged in.

**Key challenges.** Session/state management; flaky pages (need `wait_for`); auth &
cookie handling; the security surface (a browser can reach anything — combine with
prompt injection and it's an SSRF/exfil vector); resource cleanup (close tabs/contexts).

**Real example.**
[`microsoft/playwright-mcp`](https://github.com/microsoft/playwright-mcp) — 40+ tools
across navigation, forms, network, storage, tracing; Chrome/Firefox/WebKit/Edge.

**Tool surface (sketch).**
```jsonc
browser_navigate(url)
browser_snapshot() -> accessibility tree with { ref, role, name } per element
browser_click(element: "Submit button", ref: "e17")   // description + ref
browser_type(element, ref, text)
browser_wait_for(text? | time?)        // handle async pages
browser_take_screenshot()              // fallback / verification
browser_close()                        // lifecycle / cleanup
```

---

## 7. Aggregator / gateway MCP

**What it's for.** Putting *many* backend MCP servers (or APIs) behind one server —
so the agent connects once, and you get central auth, governance, and a single tool
namespace. The standard enterprise answer to "we have 20 MCP servers."

**The design pattern.**
- **The problem it solves: "too many tools."** Connecting several servers can eat a
  huge fraction of context *before the agent acts* — every tool definition is loaded
  up front. The gateway's job is to make that scale.
- **Progressive disclosure / dynamic tool discovery.** Instead of exposing all N
  tools, expose a few **meta-tools** — `find`/`search`, `schema`, `exec`/`call` —
  and let the agent discover and load specific tools on demand. The **AIRIS MCP
  Gateway** puts 60+ tools behind ~7 meta-tools and reports ~97% context-token
  reduction. **Docker's MCP Gateway** (Desktop ≥ 4.50) adds `mcp-find` / `mcp-add` /
  `code-mode` and can register servers whose tools **don't** appear in `tools/list`
  until discovered.
  ([Docker: Dynamic MCPs](https://www.docker.com/blog/dynamic-mcps-stop-hardcoding-your-agents-world/),
  [Docker MCP Toolkit](https://docs.docker.com/ai/mcp-catalog-and-toolkit/toolkit/))
- **Aggregation + middleware.** A gateway multiplexes one client session to N
  backends, and adds cross-cutting concerns: tool **filtering/aliasing** (resolve
  name clashes), **per-tool access control / scopes**, OAuth, version pinning, and
  cached aggregation. ([`metatool-ai/metamcp`](https://github.com/metatool-ai/metamcp),
  [`agentic-community/mcp-gateway-registry`](https://github.com/agentic-community/mcp-gateway-registry))

**Key challenges.** The context blow-up (the whole reason to use one); namespace
collisions; auth fan-out to heterogeneous backends; one slow/broken backend
degrading the whole gateway (circuit breakers, HOT/COLD lifecycle); and governance/audit.

**Real examples.**
[MetaMCP](https://github.com/metatool-ai/metamcp) (aggregator + middleware in one
container), [MCP Gateway & Registry](https://github.com/agentic-community/mcp-gateway-registry)
(OAuth, dynamic discovery, scopes), [Docker MCP Toolkit](https://docs.docker.com/ai/mcp-catalog-and-toolkit/toolkit/),
and the curated [`e2b-dev/awesome-mcp-gateways`](https://github.com/e2b-dev/awesome-mcp-gateways).

**Tool surface (sketch).**
```jsonc
// Meta-tools instead of N backend tools (progressive disclosure)
find_tools(query: "create a jira ticket") -> [{ tool_id, description }]
get_tool_schema(tool_id) -> { input_schema }
call_tool(tool_id, args: {...})         // routes to the right backend
// plus governance the agent never sees: auth, scopes, aliasing, circuit breakers
```

---

## 8. When NOT to build an MCP server

MCP is not free. A standard multi-server setup can consume a large share of the
context window (one analysis: ~72% / ~143K of a 200K-token model across three
servers) **before the agent does anything** — tool definitions are always-on. Often
a CLI, an Agent Skill, or a plain function is the better tool.

**Prefer a CLI when** the task is local, frequent, terminal-native, and
debuggable. The agent already knows how to run shell commands; a good CLI with
`--help` gives it everything, loads zero tokens until used, and you can test it by
hand. ("CLI shines in the inner loop.")

**Prefer an Agent Skill (or CLI + Skill) when** the capability is a *procedure*
the agent should follow, not a live service to call. Skills run locally with no
server to deploy/maintain. In one eval, on the hardest tier **MCP cost >6× and
took ~5× longer** than skills, making ~12 tool calls vs ~5. If the task is
terminal-native *but* too important to leave to improvisation, codify it as CLI +
Skill.
([You Probably Don't Need an MCP](https://medium.com/@danielbanales/you-probably-dont-need-an-mcp-0ba0f7bb6057),
[MCPs, CLIs, and skills: when to use what](https://jngiam.bearblog.dev/mcps-clis-and-skills-when-to-use-what/),
[Arize eval](https://arize.com/blog/mcp-vs-cli-skills-for-agents-what-our-eval-found-and-which-you-should-use/))

**Prefer a plain function / direct API call when** *your own code* is the consumer.
MCP's value is the *protocol* — a standard surface for *external* agents. If you're
orchestrating your own LLM calls, just call the function; MCP adds a network hop,
a server to run, and serialization overhead for no benefit.

**Build an MCP server when** the same integration surface is needed by **many teams
or many agent clients**, and you want to centralize auth, evolve the interface in
one place, and stop everyone rolling their own wrapper — GitHub, Postgres, Slack,
Sentry, your own SaaS product. That's the case MCP was designed for; the categories
above are how to build it well.

---

### One-line summary per category

| Category | Core pattern | Watch out for |
|---|---|---|
| Database | Generic read-only/mutation SQL tools + schema introspection | Enforce read-only in the *engine*; cap result size |
| REST wrapper | Workflow/layered tools, **not** 1:1 endpoints | Tool explosion; nested-JSON & pagination handling |
| SaaS product | OAuth 2.1 multi-tenant; read vs gated action tools | Prompt injection on writes; Markdown > nested JSON |
| Internal/DevOps | Read on by default, mutations gated + dry-run | Blast radius; irreversible ops; least privilege |
| Filesystem/Git | Sandbox to allowed roots; realpath + symlink-safe; argv not shell | The verified CVEs in §5 |
| Browser | Stateful session; accessibility snapshot + refs | State/lifecycle; injection → exfil surface |
| Aggregator | Meta-tools + progressive disclosure | Context blow-up; namespace clashes; backend failure |
| (none) | CLI / Skill / plain function | Don't pay MCP's context cost for a local/own-code task |
