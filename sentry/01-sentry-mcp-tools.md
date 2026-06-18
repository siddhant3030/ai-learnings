# The Sentry MCP Server — A Practical Guide for the Dalgo Harness

*Research compiled June 2026. The Sentry MCP evolves fast — verify exact tool names in your own client with `/mcp` before relying on them in automation.*

This guide is for the person running Dalgo (Django + Django Ninja backend `DDP_backend`, Next.js
frontend `webapp_v2`, FastAPI `prefect-proxy`) who already has a Sentry MCP server connected in
Claude Code and wants to actually use it in their engineering harness.

> **Read this first — your harness has a stale tool name.** Two files in this repo,
> `.claude/commands/engineering/debug-issue.md` and `.claude/agents/debugger.md`, call
> `mcp__plugin_sentry_sentry__get_sentry_resource`. **That tool does not exist in the current
> Sentry MCP**, and the prefix is wrong for your connection. See
> [§8 Fixing your existing harness](#8-fixing-your-existing-harness). This is the single most
> actionable thing in this document.

---

## 1. What the Sentry MCP server is and does

The Sentry MCP server is an official, Sentry-maintained [Model Context Protocol](https://modelcontextprotocol.io)
server that gives an AI agent structured, authenticated access to your Sentry data — issues, events,
stack traces, traces, releases, and Seer (Sentry's AI debugger) — without you copy-pasting error
details out of the Sentry web UI.

Sentry describes the motivation bluntly: *"The bottleneck for self-healing software isn't agent
intelligence. It's that agents have no idea what actually broke."* An agent reading only your source
code will add retry logic to `payment.ts` when the real cause was a dropped DB index in another
service — invisible without production telemetry.
([Sentry blog: Your agent can't fix what it can't see](https://blog.sentry.io/agents-need-production-context/))

It is a real production service, not a toy: Sentry scaled it from 30M to **60M+ monthly requests
serving 5,000+ organizations** with a three-person team, at a ~0.075% tool-call error rate.
([ZenML LLMOps case study](https://www.zenml.io/llmops-database/scaling-an-mcp-server-for-error-monitoring-to-60-million-monthly-requests))

### Hosted vs self-hosted

| Option | Endpoint / command | When to use |
|--------|--------------------|-------------|
| **Hosted (remote)** — recommended | `https://mcp.sentry.dev/mcp` (Streamable HTTP; SSE also supported) | You use sentry.io (SaaS). OAuth in-browser, no token management. This is almost certainly what your `mcp__claude_ai_Sentry__*` connection uses. |
| **Self-hosted / stdio** | `npx @sentry/mcp-server@latest --access-token=TOKEN --host=sentry.example.com` | Self-hosted Sentry, air-gapped, or you want a pinned local version. Needs a User Auth Token. |

Sources: [mcp.sentry.dev](https://mcp.sentry.dev/), [getsentry/sentry-mcp README](https://github.com/getsentry/sentry-mcp),
[Smarter debugging with Sentry MCP + Cursor](https://blog.sentry.io/smarter-debugging-sentry-mcp-cursor/).

The server supports three transports: **stdio**, **Streamable HTTP**, and **SSE**
([ZenML](https://www.zenml.io/llmops-database/scaling-an-mcp-server-for-error-monitoring-to-60-million-monthly-requests)).

### The tools it exposes

The current server exposes ~20–25 tools depending on version and scopes. The **read** tools below
are corroborated by GitHub's own agentic-workflows repo
([gh-aw `sentry.md`](https://github.com/github/gh-aw/blob/main/.github/workflows/shared/mcp/sentry.md))
and the [Speakeasy catalog](https://www.speakeasy.com/product/mcp-platform/catalog/sentry). **Write**
tools come from the Speakeasy / [PolicyLayer](https://policylayer.com/policies/sentry) catalogs.

**Read tools (safe — no mutation):**

| Tool | What it does |
|------|--------------|
| `whoami` | Identify the authenticated user. Use to verify the connection works. |
| `find_organizations` | List orgs you can access. |
| `find_teams` | List teams in an org. |
| `find_projects` | List projects (you need the project *slug* for most other calls). |
| `find_releases` | List releases — for release-health and "what shipped" questions. |
| `find_dsns` | List a project's DSNs. |
| `search_issues` | **Natural-language** search for grouped issues ("unresolved 500s in the last 24h"). Needs an embedded LLM provider on the server. |
| `search_events` | **Natural-language** search across individual events/errors/traces, filterable by time, environment, release, user, trace ID, tags. |
| `list_events` / `list_issue_events` | Deterministic fallback enumeration when no LLM provider is configured. |
| `get_issue_details` | **The workhorse.** Full issue context: stack trace, breadcrumbs, tags, release, environment, suspect commits, user impact, trace IDs. Accepts an issue ID *or* a Sentry issue URL. |
| `get_trace_details` | Inspect a distributed trace (spans across services). |
| `get_event_attachment` | Pull screenshots, log files, or attachments uploaded with an error. |
| `analyze_issue_with_seer` | Trigger **Seer** root-cause analysis (and fix suggestion) on an issue. *(Older builds / some clients expose this as `begin_seer_issue_fix`; the Cursor blog and ZenML case study reference earlier spellings `begin_seer_issue_fix` / `begin_sentry_issue_fix`. Confirm the exact name with `/mcp`.)* |
| `search_docs` / `get_doc` | Search and fetch Sentry's own documentation (SDK setup, etc.). |

**Write tools (mutate Sentry — gate these; see [§6 security](#6-setup--configuration)):**

| Tool | What it does |
|------|--------------|
| `update_issue` | Change issue status (resolve / ignore / assign). |
| `create_project`, `update_project` | Create / configure a project. |
| `create_team` | Create a team. |
| `create_dsn` | Create a new client key/DSN. |

> An important architectural note for context budgeting: Sentry **embeds an AI agent inside the MCP
> tool responses** to filter and summarize before returning data, rather than dumping the whole issue
> page as Markdown — their term for the problem is *"context pollution."*
> ([ZenML](https://www.zenml.io/llmops-database/scaling-an-mcp-server-for-error-monitoring-to-60-million-monthly-requests))
> This is why `get_issue_details` is usually safe to call directly, but see [§9 gotchas](#9-limitations--gotchas).

### How it connects to clients

```bash
# Claude Code — remote/hosted (recommended)
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp
# then run `claude`, type `/mcp`, and complete the in-browser OAuth sign-in
```

There's also an official Claude Code **plugin** that bundles a Sentry subagent for auto-delegation:

```bash
claude plugin marketplace add getsentry/sentry-mcp
claude plugin install sentry-mcp@sentry-mcp        # or @sentry-mcp-experimental
```

Cursor, VS Code (Copilot), Windsurf, and Codex connect the same way — point the client at
`https://mcp.sentry.dev/mcp` and authenticate via OAuth.
([debug-with-sentry-mcp-claude-code cookbook](https://sentry.io/cookbook/debug-with-sentry-mcp-claude-code/),
[Sentry for AI docs](https://docs.sentry.io/ai/))

---

## 2. What people are actually doing with it

### AI-assisted debugging (the core loop)

Sentry's own cookbook lays out the canonical four-step flow with Claude Code
([debug-with-sentry-mcp-claude-code](https://sentry.io/cookbook/debug-with-sentry-mcp-claude-code/)):

1. **List** open issues — *"Use Sentry to list the top 5 open issues in my `<project>` project from
   the last 24 hours. Summarize each: what's failing, how often, which users are affected."* → calls `search_issues`.
2. **Fetch context** — *"Fetch the latest event for issue `<ISSUE-ID>`. Get the full stack trace,
   related logs around the time of the error, and run a root-cause analysis with Seer."* → `get_issue_details` + `analyze_issue_with_seer`.
3. **Fix** — *"Based on the Sentry issue and root cause, locate the relevant code in this repo and
   apply a fix. Show me the diff before changing anything."* → Claude correlates the trace with your code.
4. **Review** — human reviews the diff before commit.

### Seer — Sentry's AI agent — and how it complements MCP

Seer (formerly "Autofix") is Sentry's AI debugger that runs *inside Sentry*, on Sentry's data. Its
Autofix flow is three stages: **Root Cause Analysis → Solution Identification → Code Generation**
(optionally opening a PR or handing off to a coding agent).
([Introducing Seer Agent](https://blog.sentry.io/introducing-seer-agent/),
[Autofix docs](https://docs.sentry.io/product/ai-in-sentry/seer/autofix/))

Crucially, Seer references **distributed traces**, so it catches failures that propagate across
services and latency from contention/resource saturation — not just single-file stack traces.
([Autofix + traces](https://blog.sentry.io/sentry-ai-debugger-autofix-superpower-traces/))
In January 2026 Sentry extended Seer to **local development and code review** with flat,
unlimited pricing.
([Sentry press release](https://sentry.io/about/press-releases/sentry-expands-seer-ai-debugging-agent),
[The New Stack](https://thenewstack.io/sentrys-seer-agent-debug/))

**The division of labour:** Seer does deep root-cause analysis on Sentry's side; the MCP server is
how your agent *pulls Seer's findings* (and raw context) into a session where it also has your
codebase. Cursor "can't independently identify root causes spanning multiple components — that's
where Seer becomes necessary."
([Cursor blog](https://blog.sentry.io/smarter-debugging-sentry-mcp-cursor/))

### Triage automation & release health

Sentry's performance-triage cookbook runs a **scheduled Claude Code task every Monday 9am** that
finds slow endpoints, runs Seer, and opens capped draft PRs — "no custom infrastructure, a few cents
per run."
([Automate weekly performance triage](https://sentry.io/cookbook/performance-bot-sentry-claude/))

### The agentic "fix this Sentry issue" prompt (from Sentry)

> *"Use Sentry MCP to retrieve full issue details: stack trace, breadcrumbs, tags, release,
> environment, distributed traces, suspect commits, and Seer analysis. Identify root cause. Make the
> smallest safe fix. Add regression test. Open draft PR."*
> — [Sentry blog](https://blog.sentry.io/agents-need-production-context/)

Note the recommended scope: **draft PRs with human review**, not autonomous deploys.

Anthropic and Sentry have also documented Sentry's own use of Claude Code internally
([Anthropic customer story](https://www.anthropic.com/customers/sentry)).

---

## 3. How to use it in *your* harness — concrete workflows

Your connection is namespaced `mcp__claude_ai_Sentry__*`. So the call for issue details is
`mcp__claude_ai_Sentry__get_issue_details`, etc. (Confirm exact suffixes with `/mcp`.)

> **Assumption to verify once:** this guide researched the tool names of the official remote /
> open-source `@sentry/mcp-server`, then prefixed them with your client's `mcp__claude_ai_Sentry__`
> namespace. The claude.ai Sentry connector almost certainly proxies that same remote server (so the
> names match), but the only way to confirm against *your live connection* is `/mcp` after OAuth —
> the data tools don't appear until you've authenticated.

Dalgo specifics worth baking into prompts:
- **Backend errors** → Python stack traces, `/api/` paths, Django ORM/PostgreSQL, DBT/Airbyte/Prefect → live in `DDP_backend`.
- **Frontend errors** → `.tsx`/`.ts` traces, React hydration, SWR/Zustand, page routes (not `/api/`) → live in `webapp_v2`.
- **Cross-cutting** → a frontend symptom with a backend root cause; use `get_trace_details` to follow the request across services.

### Workflow A — `/debug-issue` from a Sentry URL (rewrite of your existing command)

| | |
|---|---|
| **Trigger** | `/engineering/debug-issue https://<org>.sentry.io/issues/DALGO-123/` |
| **Tools** | `get_issue_details` → (optional) `analyze_issue_with_seer` → (optional) `get_trace_details` |
| **Prompt** | *"Call `get_issue_details` with this Sentry URL. Pull the full stack trace, breadcrumbs, tags, release, environment, and suspect commits. Classify backend (`DDP_backend`) vs frontend (`webapp_v2`). Then locate the offending code, give me a ranked root-cause hypothesis, a minimal diff, regression risk, and a test that catches it. Show the diff before editing."* |
| **Output** | Diagnosis Report (matches your existing `debug-issue.md` format). |

### Workflow B — Auto-triage new issues

| | |
|---|---|
| **Tools** | `search_issues` → `get_issue_details` (top N) |
| **Prompt** | *"Use `search_issues` for unresolved issues in project `<slug>` from the last 24h. For the top 5 by event count, call `get_issue_details`. Cluster by likely root cause, label each backend/frontend/cross-cutting, flag any touching beneficiary/PII data, and rank by user impact. Don't propose fixes yet."* |

### Workflow C — Pre-deploy release-health check

| | |
|---|---|
| **Tools** | `find_releases` → `search_events` / `search_issues` |
| **Prompt** | *"Use `find_releases` to get the last 3 releases of `<slug>`. With `search_events`, compare new/regressed error issues and error-event volume in the latest release vs the prior one, scoped to environment=production. Is it safe to deploy? List any new error signatures introduced since the last release."* |

### Workflow D — Connect a Sentry error to specific Dalgo code

| | |
|---|---|
| **Tools** | `get_issue_details` (suspect commits + frames) + your repo (Read/Grep) |
| **Prompt** | *"From `get_issue_details` for `<ISSUE>`, take the top in-app stack frame and the suspect commit. Open that file in `DDP_backend` (or `webapp_v2`), read the function, and explain exactly which line raises under what condition. Read the repo's CLAUDE.md before proposing a change."* |

> Suspect commits map the error to a code location even when the deployed stack trace is minified or
> the line numbers drifted — lean on them.

### Workflow E — Seer root-cause via MCP

| | |
|---|---|
| **Tools** | `analyze_issue_with_seer` (+ `get_issue_details` fallback) |
| **Prompt** | *"Run `analyze_issue_with_seer` on issue `<ISSUE>`. Summarize Seer's root cause and proposed solution. If Seer returns nothing actionable (common for performance/N+1 issues), fall back to the trace + stack frames from `get_issue_details` and analyze the code yourself."* |
| **Gotcha** | Per Sentry, *"Seer works best on error-type issues. For performance issues like N+1 queries, Seer may not return a fix."* Always include the fallback. ([performance cookbook](https://sentry.io/cookbook/performance-bot-sentry-claude/)) |

---

## 4. Experiments to try (10–15)

| # | Experiment | Tools | What you'd learn | Difficulty |
|---|------------|-------|------------------|------------|
| 1 | `whoami` + `find_projects` — confirm the connection and list Dalgo's project slugs | `whoami`, `find_projects` | That auth works and which slugs to use everywhere else | Trivial |
| 2 | Summarize the last 20 issues and cluster by root cause | `search_issues`, `get_issue_details` | Where your error budget actually goes; dup clusters | Easy |
| 3 | Weekly error digest (Markdown) — top issues, deltas vs last week, owners | `search_issues`, `find_releases` | A repeatable standup artifact; trend visibility | Easy |
| 4 | Backend-vs-frontend split — auto-route last 30 issues to `DDP_backend` / `webapp_v2` | `search_issues`, `get_issue_details` | How well the agent classifies your stack from traces | Easy |
| 5 | One-issue deep dive: trace → suspect commit → exact line in your repo | `get_issue_details`, `get_trace_details` | The full MCP-to-code correlation loop end to end | Medium |
| 6 | Top-5 fixes as **draft PRs** (capped at 2/run) | `search_issues`, `get_issue_details`, `analyze_issue_with_seer` + `gh` | Whether agentic fixes are review-worthy on your code | Medium |
| 7 | Correlate an error spike with a recent deploy | `find_releases`, `search_events` | Whether release X introduced regression Y | Medium |
| 8 | Pre-deploy gate as a slash command — "safe to ship?" verdict | `find_releases`, `search_issues` | A reusable release checklist | Medium |
| 9 | Seer vs DIY — run `analyze_issue_with_seer` and your own analysis on 5 issues, compare | `analyze_issue_with_seer`, `get_issue_details` | When Seer adds value vs when it whiffs (perf issues) | Medium |
| 10 | Cross-service trace hunt — a `webapp_v2` symptom traced to a `DDP_backend` or `prefect-proxy` cause | `get_trace_details`, `get_issue_details` | Distributed-trace debugging across Dalgo services | Hard |
| 11 | PII audit — scan recent error payloads/breadcrumbs for beneficiary data leaking into Sentry | `search_events`, `get_issue_details` | Whether NGO PII is reaching Sentry (then scrub it) | Medium |
| 12 | Auto-resolve stale issues (dry run first!) — propose `update_issue` resolutions, don't execute | `search_issues`, `update_issue` | How to safely gate write tools; triage hygiene | Medium |
| 13 | Scheduled Monday triage bot (Sentry's recipe, adapted to Dalgo slugs) | `search_issues`, `get_issue_details`, `analyze_issue_with_seer` | Hands-off triage + draft PRs at near-zero cost | Hard |
| 14 | Context-cost benchmark — measure tokens for `get_issue_details` on a fat vs thin issue | `get_issue_details` | When to scope queries to avoid blowing the budget | Easy |
| 15 | Docs-assisted SDK fix — use `search_docs`/`get_doc` to fix a Sentry SDK misconfig in Dalgo | `search_docs`, `get_doc` | Whether the agent can self-serve Sentry SDK setup | Easy |

Start with 1 → 2 → 5. They prove the chain before you automate anything (experiments 6 and 13).

---

## 5. Setup & configuration

### Add to Claude Code (hosted — recommended)

```bash
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp
```

Then run `claude`, type `/mcp`, and complete OAuth in the browser. Or add to `.mcp.json` /
`~/.claude.json` directly:

```json
{
  "mcpServers": {
    "sentry": {
      "type": "http",
      "url": "https://mcp.sentry.dev/mcp"
    }
  }
}
```

OAuth (no static token in the file) is preferred. ([performance cookbook](https://sentry.io/cookbook/performance-bot-sentry-claude/))

### Self-hosted / stdio

```json
{
  "mcpServers": {
    "sentry": {
      "command": "npx",
      "args": ["@sentry/mcp-server@latest", "--access-token=YOUR_TOKEN", "--host=sentry.example.com"],
      "env": {
        "EMBEDDED_AGENT_PROVIDER": "anthropic",
        "ANTHROPIC_API_KEY": "sk-ant-..."
      }
    }
  }
}
```

- The **natural-language search tools** (`search_issues`, `search_events`, `search_docs`) need an
  embedded LLM provider (`EMBEDDED_AGENT_PROVIDER` = `openai` | `anthropic` + key). Without one they
  fall back to the deterministic `list_*` tools.
  ([README](https://github.com/getsentry/sentry-mcp))
- Disable Seer on instances that don't support it: `--disable-skills=seer`.
- For internal HTTP-only hosts: `--insecure-http`.

### Required Sentry token scopes (self-hosted)

```
org:read  project:read  project:write  team:read  team:write  event:write
```
([README](https://github.com/getsentry/sentry-mcp))

### Security — this reads production error data

- **PII is the real risk here, not OAuth boilerplate.** Dalgo handles NGO beneficiary data. Error
  payloads, request bodies, and breadcrumbs can contain that data, and the MCP will faithfully hand
  it to the model. **Configure Sentry's server-side data scrubbing / `beforeSend`** in the Dalgo SDKs
  so PII never reaches Sentry in the first place — then the MCP can't surface it.
  (See Sentry's "Scrubbing Sensitive Data" / Data Scrubbers docs under
  [docs.sentry.io](https://docs.sentry.io/) — search "sensitive data" if the path has moved.) Run
  experiment #11 to check what's actually leaking today.
- **Scope to read-only for everyday use.** The high-value loop (triage, debug, release health) is
  entirely read tools. Keep `update_issue`, `create_project`, `create_dsn`, `update_project`,
  `create_team` out of any unattended/scheduled workflow. In Claude Code, allow-list the read tools
  explicitly and require confirmation for writes.
- **Least-privilege tokens** for self-hosted: drop `project:write`/`team:write`/`event:write` if you
  only read. OAuth scopes are tied to your Sentry role — don't run the MCP as an org admin if a
  member role suffices.
- **Scheduled tasks run on your machine** and **OAuth tokens expire between runs** — a silent failure
  mode; re-auth with `/mcp`. ([performance cookbook](https://sentry.io/cookbook/performance-bot-sentry-claude/))

### Token / context-cost considerations

- A single fat `get_issue_details` (long stack trace + many breadcrumbs + trace) can be large.
  Sentry pre-summarizes, but it's still the biggest line item. Pull details for the *top N* issues,
  not all of them.
- Prefer `search_issues` (grouped) over `search_events` (individual) when you only need the picture.
- The Sentry search API **does not support boolean `OR`/`AND`** — run separate scoped queries
  instead of one combined query. ([performance cookbook](https://sentry.io/cookbook/performance-bot-sentry-claude/))
- Always scope by `project`, `environment=production`, and a time window to keep responses small.

---

## 6. Limitations & gotchas

- **NL search needs an LLM provider** (hosted: handled by Sentry; self-hosted: you must configure it).
- **No boolean operators** in search — separate queries per condition.
- **Seer underperforms on performance issues** (N+1, slow queries) — keep a code-analysis fallback.
- **Ambiguous project/org references** when you have several — always pass explicit slugs.
- **OAuth account confusion** — if you have multiple Sentry accounts/browsers, you can auth as the
  wrong one. ([Cursor blog](https://blog.sentry.io/smarter-debugging-sentry-mcp-cursor/))
- **Expiring tokens** silently break scheduled runs — re-auth via `/mcp`.
- **Self-hosted Sentry may not support Seer** — disable it.
- **Large responses** can still eat context if you fetch details for many issues at once.
- **Invalid-token / connection errors** are a recurring theme in the repo's issues
  ([#340 invalid token](https://github.com/getsentry/sentry-mcp/issues/340),
  [#553 search_events on issue counts](https://github.com/getsentry/sentry-mcp/issues/553)) — when a
  call fails, start with `whoami` to confirm auth before debugging the query.

---

## 7. Best practices for AI + Sentry workflows

1. **Draft PRs, human review** — never auto-merge an agent's Sentry fix. (Sentry's own stance.)
2. **Read tools by default, writes behind a gate** — the whole value loop is read-only.
3. **Cap the blast radius** — limit PRs/issues per run (start at 2), scope by project + environment + time.
4. **Always give Seer a fallback** — code-analyze when it returns nothing actionable.
5. **Scrub PII at the source** — fix `beforeSend` in the Dalgo SDKs so beneficiary data never reaches Sentry.
6. **Chain deliberately**: `search_issues` → `get_issue_details` → `analyze_issue_with_seer` → repo. Don't dump everything up front.
7. **Lean on suspect commits** to jump from error to the exact Dalgo file.
8. **You own the prompt, not the user** — Sentry's lesson: tool/prompt design carries the behavior; don't rely on perfectly phrased requests. ([ZenML](https://www.zenml.io/llmops-database/scaling-an-mcp-server-for-error-monitoring-to-60-million-monthly-requests))
9. **Verify tool names with `/mcp`** before wiring them into commands — they drift across versions.

---

## 8. Fixing your existing harness

`.claude/commands/engineering/debug-issue.md` and `.claude/agents/debugger.md` both call
`mcp__plugin_sentry_sentry__get_sentry_resource`. Two problems:

1. **`get_sentry_resource` is not a current Sentry MCP tool.** The replacement is
   **`get_issue_details`**, which accepts an issue ID *or* a Sentry URL — exactly the input your
   `/debug-issue` command parses. (If your build of the tool only accepts an ID, extract it from the
   URL first, e.g. `DALGO-123` from `.../issues/DALGO-123/`.)
2. **The prefix is wrong** for your connection. Your server is `mcp__claude_ai_Sentry__*`, not
   `mcp__plugin_sentry_sentry__*`.

**Change, in both files:**

```diff
- Fetch issue details using `mcp__plugin_sentry_sentry__get_sentry_resource` with the URL.
+ Fetch issue details using `mcp__claude_ai_Sentry__get_issue_details` (pass the URL or issue ID).
- Use `mcp__plugin_sentry_sentry__search_issues` to find related issues
+ Use `mcp__claude_ai_Sentry__search_issues` to find related issues
```

Then add, in the `debugger` agent's Gather phase: *"After `get_issue_details`, optionally call
`mcp__claude_ai_Sentry__analyze_issue_with_seer` for a root-cause hypothesis, and
`mcp__claude_ai_Sentry__get_trace_details` for cross-service traces."*

> Confirm the exact suffixes with `/mcp` in your Claude Code — this guide researches the *server's*
> tool names; the `mcp__claude_ai_Sentry__` prefix is your *client's* and the post-prefix names should
> match what `/mcp` lists.

---

## Sources

- Sentry MCP service & docs — https://mcp.sentry.dev/ · https://docs.sentry.io/ai/
- getsentry/sentry-mcp (repo) — https://github.com/getsentry/sentry-mcp
- Debug with Sentry MCP + Claude Code — https://sentry.io/cookbook/debug-with-sentry-mcp-claude-code/
- Automate weekly performance triage (Claude Code + Sentry MCP) — https://sentry.io/cookbook/performance-bot-sentry-claude/
- Your agent can't fix what it can't see — https://blog.sentry.io/agents-need-production-context/
- Smarter debugging with Sentry MCP + Cursor — https://blog.sentry.io/smarter-debugging-sentry-mcp-cursor/
- Introducing Seer Agent — https://blog.sentry.io/introducing-seer-agent/
- Autofix + traces — https://blog.sentry.io/sentry-ai-debugger-autofix-superpower-traces/
- Seer Autofix docs — https://docs.sentry.io/product/ai-in-sentry/seer/autofix/
- Seer expands to local dev & code review — https://sentry.io/about/press-releases/sentry-expands-seer-ai-debugging-agent · https://thenewstack.io/sentrys-seer-agent-debug/
- Scaling the MCP server to 60M req/mo (ZenML) — https://www.zenml.io/llmops-database/scaling-an-mcp-server-for-error-monitoring-to-60-million-monthly-requests
- Tool catalogs — https://github.com/github/gh-aw/blob/main/.github/workflows/shared/mcp/sentry.md · https://www.speakeasy.com/product/mcp-platform/catalog/sentry · https://policylayer.com/policies/sentry
- Anthropic customer story (Sentry uses Claude Code) — https://www.anthropic.com/customers/sentry
