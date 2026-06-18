# MCP & Tool Design in the Dalgo Harness

**Status:** Draft
**Owner:** Engineering
**Topic:** How the Dalgo engineering harness should use MCP servers and tool design to push agent autonomy further.

This is a reference + recommendations doc, in the spirit of `harness-evolution.md`: **the discipline is in the scaffolding.** MCP tools are scaffolding — they let an agent observe and act on live state instead of guessing from code. The leverage is in wiring the right tools into the right commands, with guardrails so a mutating tool can never fire inside an autonomous step.

---

## 1. Feature reference — MCP, many-tool handling, tool design

### 1.1 What MCP is

The **Model Context Protocol (MCP)** is an open standard for connecting an agent to external tools, data, and APIs. An MCP *server* exposes three things: **tools** (callable functions), **resources** (referenceable data, like a file or a schema), and **prompts** (canned templates). Claude Code is an MCP *client* — it connects to servers and surfaces their tools to the model. See the protocol spec at <https://modelcontextprotocol.io/introduction> and Claude Code's MCP reference at <https://code.claude.com/docs/en/mcp>.

Connect a server "when you find yourself copying data into chat from another tool" (Claude Code MCP docs). For Dalgo that's the live platform: warehouse rows, pipeline run logs, dashboard configs — state that lives in the running product, not in `DDP_backend`/`webapp_v2` source.

### 1.2 How Claude Code consumes MCP servers

- **Transports:** `http` (recommended for remote; `streamable-http` is an alias), `sse` (deprecated), `stdio` (local process), `ws` (WebSocket, for push). Source: MCP docs §transports.
- **Scopes** (where the config lives, who gets it):

  | Scope | Loads in | Shared | Stored in |
  |-------|----------|--------|-----------|
  | `local` (default) | current project only | no | `~/.claude.json` |
  | `project` | current project only | **yes, via git** | `.mcp.json` in repo root |
  | `user` | all your projects | no | `~/.claude.json` |

  Project-scoped servers in `.mcp.json` require approval before first use; reset with `claude mcp reset-project-choices`. **This repo currently has no `.mcp.json`** — every MCP server in this session is local/user-scoped and therefore invisible to teammates and CI.
- **Tool naming:** tools are addressed as `mcp__<server>__<tool>`, e.g. `mcp__dalgo__dalgo_get_table_row_count`. Plugin-bundled servers use `mcp__plugin_<plugin>_<server>__<tool>`. This exact string is what you put in permission rules, a skill's `allowed-tools`, or a subagent's `tools` field.
- **Output limits:** Claude Code warns when a single tool result exceeds **10,000 tokens**; raise with `MAX_MCP_OUTPUT_TOKENS`. Per-server tool timeout via `timeout` (ms) in the server's config.
- **Resources** are referenced with `@server:protocol://path` mentions and are auto-attached.

### 1.3 The many-tools problem and Tool Search

The Dalgo session has **70+ `dalgo_*` tools** plus serena, playwright, figma, headroom, Sentry, Gmail, Calendar, Drive. Loading every schema upfront is expensive — Anthropic measured a 5-server setup at ~55K tokens before the conversation even starts (<https://www.anthropic.com/engineering/advanced-tool-use>).

**Claude Code's answer is Tool Search, on by default.** Only **tool names + each server's instructions** load at session start; full schemas are *deferred* and fetched on demand via a `ToolSearch` call that returns the 3–5 most relevant tools. Source: <https://code.claude.com/docs/en/mcp> §"Scale with MCP Tool Search". Quote: *"adding more MCP servers has minimal impact on your context window… Only the tools Claude actually uses enter context."* Reported ~85% token reduction.

This is exactly the model in play here — the deferred-tool list in this session, and the requirement to call `ToolSearch` (`select:<name>`) before invoking a `dalgo_*` tool, is Tool Search at work.

Config knobs (env var or `settings.json` `env`):

| `ENABLE_TOOL_SEARCH` | Behavior |
|----------------------|----------|
| (unset) | Default. All MCP tools deferred, loaded on demand. |
| `true` | Force-defer everywhere (incl. Vertex/proxies; needs Sonnet 4.5 / Opus 4.5+). |
| `auto` | Load upfront if schemas fit in **10% of context**, defer the overflow. |
| `auto:N` | Same, custom percentage (`auto:5` = 5%). |
| `false` | Load everything upfront, no deferral. |

Notes that bite: **Haiku models don't support `tool_reference` blocks** (no Tool Search). It's auto-disabled on Vertex AI and non-first-party `ANTHROPIC_BASE_URL` proxies. You can disable the tool itself with `permissions.deny: ["ToolSearch"]`.

**Implication for server authors (us, for the dalgo MCP):** with Tool Search on, the **server instructions** field carries more weight — it's what Claude reads to decide whether to search your server at all. Claude Code **truncates tool descriptions and server instructions at 2KB each** — keep them tight, critical info first. The dalgo server already ships good instructions (warehouse / pipelines / dashboards / docs categories); that's why the model can find the right tool without all 70 schemas loaded.

### 1.4 Tool design best practices

From Anthropic's "Writing effective tools for AI agents" (<https://www.anthropic.com/engineering/writing-tools-for-agents>) and "Advanced tool use" (<https://www.anthropic.com/engineering/advanced-tool-use>):

- **Build few high-value tools, not many fragmented ones.** Consolidate related operations so the agent isn't choosing between near-duplicates — as one summary of Anthropic's guidance puts it, one powerful tool beats five fragmented ones (agentailor.com, paraphrasing the Anthropic post).
- **Namespace clearly.** Consistent prefixes (the `dalgo_` prefix, grouped by resource: `dalgo_get_*`, `dalgo_list_*`, `dalgo_create_*`) signal which capabilities belong together and prevent collisions. Dalgo already follows this well.
- **Name parameters unambiguously.** `user_id`, not `user`. `flow_run_id`, not `id`.
- **Return high-signal, semantic context** — human-readable names over opaque IDs the agent must look up again. Offer a `response_format` / verbosity flag (`concise` vs `detailed`) so the agent pulls only what it needs.
- **Token efficiency:** paginate, truncate, filter by default. Keep responses **under ~25,000 tokens** for best performance (and note Claude Code's own 10K-token warning threshold). The guiding rule: *"the smallest set of high-signal tokens that maximize the likelihood of your desired outcome."*
- **Error messages should teach, not just fail.** Replace cryptic codes with guidance the agent can act on: *"Found 847 expenses. This is too large to return. Please narrow your date range or specify a category filter."* This steers the agent toward pagination/filtering instead of a retry loop.
- **Descriptions = onboarding a new hire.** Spell out query formats, niche terms, and how resources relate. These are loaded into context and steer behavior.
- **Use `input_examples`** (1–5 realistic invocations) for tools with nested/optional params — Anthropic measured complex-parameter accuracy rising 72% → 90%.
- **Iterate against evals**, don't hand-tune once.

### 1.5 MCP vs CLI vs Skill — when to use which

| Build a… | When | Dalgo example |
|----------|------|---------------|
| **MCP server/tool** | The agent needs to **read or act on live external state** through a stable, typed contract, reusable across many sessions/repos. Worth it when the data is behind auth or an API, not on disk. | The dalgo MCP — live warehouse rows, pipeline logs, dashboard configs. |
| **CLI tool (Bash)** | A deterministic, scriptable, local operation where exact text output matters and you want it in version control. Cheaper than an MCP server for one-off or repo-local needs. | `python manage.py makemigrations --check`, `npm run lint`, `gh pr view`. The harness already leans on these in `validate-spec`. |
| **Skill** | You need to encode **procedure and judgment** — *how* to do a multi-step task, which tools to call in what order, what conventions apply. Skills are prompt-context, not execution. | `executing-feature-plans`, `documentation`. A skill can *instruct the agent to call* dalgo MCP tools; it doesn't replace them. |

Rule of thumb: **MCP = new capability (verbs/data); Skill = how to wield existing capabilities; CLI = deterministic local action.** They compose — the best harness moves have a skill telling the agent which MCP tool to call at which step.

---

## 2. Current state in the Dalgo harness

### 2.1 What's wired up

MCP servers present in this session:

| Server | Tools | Used in harness today? |
|--------|-------|------------------------|
| **dalgo** | ~70 `dalgo_*` — warehouse, pipelines, sources, dbt, charts, dashboards, reports, docs, notifications, org | **No.** Zero references in any command, agent, or skill. |
| **serena** | semantic code (find_symbol, references, edits) | Available; not referenced in commands (plan-feature points agents at landmark files + Explore subagents instead). |
| **playwright** | browser drive | **Yes** — `executing-feature-plans` step 5 uses it for the UI smoke test. |
| **figma** | design read/write | Via design-handoff plugin skills (out of scope here). |
| **headroom** | context compression | Available; no harness wiring. |
| **Sentry / Gmail / Calendar / Drive** | issue + Google data | Sentry referenced by `debug-issue` + `debugger` (but see gap below). |

### 2.2 Gaps

1. **The dalgo MCP is completely unused by the harness.** It's the single biggest untapped asset — it's the only way an agent sees live product state. None of `validate-spec`, `debug-issue`, `plan-feature`, `review-pr`, `execute-plan`, or the `documentation` skill calls a single `dalgo_*` tool.

2. **`validate-spec` never checks live state.** It validates spec→code by reading the git diff and running lint/tests (read-only, by design). It cannot answer *"did the migration actually populate rows?"* or *"does the new chart render against the live warehouse?"* — questions only live tools (`dalgo_get_table_row_count`, `dalgo_render_chart`) can answer.

3. **`debug-issue` has a stale Sentry tool name AND no live pipeline access.**
   - It calls `mcp__plugin_sentry_sentry__get_sentry_resource` / `mcp__plugin_sentry_sentry__search_issues`. The Sentry server actually present in this session is `mcp__claude_ai_Sentry__*` — **the namespaces don't match**, so the documented call would fail. (A worktree copy even hardcodes a third name, `mcp__16395e8d-…__get_sentry_resource`.) This is a concrete, fixable drift.
   - Many Dalgo bugs are pipeline failures (Airbyte/dbt/Prefect). `dalgo_get_flow_run_logs`, `dalgo_get_flow_run`, and `dalgo_get_pipeline_run_history` exist and would let the debugger read the *actual* failing run — but debug-issue never mentions them.

4. **The `documentation` skill ignores the dalgo docs tools.** It works entirely off the local `dalgo_docs/` Docusaurus repo + a screenshot script. `dalgo_search_docs` / `dalgo_get_doc` / `dalgo_list_docs` (the *published* product docs) are never consulted — so a generated page can't cross-check against what users actually see today, and can't reuse canonical terminology from live docs.

5. **No MCP permission rules at all.** `.claude/settings.json` has `allow`/`deny` for Read/Edit/Write and `.env` blocking, but **zero `mcp__*` rules.** Consequence: every dalgo tool — including `dalgo_delete_dashboard`, `dalgo_delete_pipeline`, `dalgo_publish_changes`, `dalgo_run_dbt`, `dalgo_trigger_pipeline_run` — falls to default behavior with no classification. Read-only tools generate needless prompts; **mutating, production-touching tools have no guardrail.**

6. **Nothing is shared.** No `.mcp.json` at repo root means the dalgo MCP wiring lives only on individual machines. Teammates and any background/CI agent don't get it.

---

## 3. Recommendations

Precedence reminder (from <https://code.claude.com/docs/en/permissions>): **rules evaluate `deny` → `ask` → `allow`; a `deny` always wins** regardless of file or order. Tool-name globs are allowed only *after* a literal `mcp__<server>__` prefix.

### P1 — Safety + cheap wins (do first)

**P1.1 — Classify every dalgo MCP tool by mutation risk in `settings.json`.** This is the guardrail that makes everything else in P2/P3 safe.

```jsonc
{
  "permissions": {
    "allow": [
      // Read-only dalgo tools — auto-approve to cut prompt noise
      "mcp__dalgo__dalgo_get_*",
      "mcp__dalgo__dalgo_list_*",
      "mcp__dalgo__dalgo_search_docs",
      "mcp__dalgo__dalgo_render_chart"
    ],
    "ask": [
      // Row-level reads — PII risk (see Open Question #4); ask wins over the
      // dalgo_get_* allow glob because ask is evaluated before allow
      "mcp__dalgo__dalgo_get_table_data",
      "mcp__dalgo__dalgo_get_chart_data",
      // Mutating but low-blast-radius — human confirms each call
      "mcp__dalgo__dalgo_create_*",
      "mcp__dalgo__dalgo_update_*",
      "mcp__dalgo__dalgo_edit_operation",
      "mcp__dalgo__dalgo_add_source_to_canvas",
      "mcp__dalgo__dalgo_publish_changes",
      "mcp__dalgo__dalgo_mark_notifications_read"
    ],
    "deny": [
      // Destructive or production-execution — never auto-run, and an agent must
      // never reach for these unprompted. To run one, a human edits settings.json
      // (a deny cannot be approved past in-session — that's the point).
      "mcp__dalgo__dalgo_delete_*",
      "mcp__dalgo__dalgo_run_dbt",
      "mcp__dalgo__dalgo_trigger_pipeline_run",
      "mcp__dalgo__dalgo_sync_sources",
      "mcp__dalgo__dalgo_terminate_chain"
    ]
  }
}
```

   - **Hard rule, state it in CLAUDE.md:** *never wire a mutating dalgo tool into an autonomous pipeline step.* Live-state checks in commands use **read-only tools only** (`dalgo_get_*`, `dalgo_list_*`, `dalgo_render_chart`). A `deny` entry makes this mechanical, not advisory — matching the harness philosophy of encoding constraints, not documenting them.
   - Why `delete`/`run_dbt`/`trigger_pipeline_run`/`sync_sources` are `deny` not `ask`: they execute against a **live NGO warehouse**. A wrong `dalgo_run_dbt` or `dalgo_delete_pipeline` can corrupt or drop a partner's data — donor-compliance / AGPL territory per CLAUDE.md. A `deny` blocks them entirely; it **cannot** be approved past in-session (unlike `ask`). To run one, a human must consciously remove it from `deny` in `settings.json` — that friction is the safeguard. If you'd rather allow case-by-case in-session confirmation instead, move them to `ask`; the doc's stance is that production-execution tools warrant the stronger `deny`.
   - **PII carve-out:** `dalgo_get_table_data` / `dalgo_get_chart_data` return raw rows that may include beneficiary PII (Open Question #4). They sit in `ask` even though the broader `dalgo_get_*` glob is in `allow` — and `ask` wins, because precedence is `deny` → `ask` → `allow`. Aggregate/schema reads (`dalgo_get_table_row_count`, `dalgo_get_table_columns`, `dalgo_list_*`) stay auto-approved.

**P1.2 — Fix the stale Sentry tool name in `debug-issue.md` and `debugger.md`.** Replace `mcp__plugin_sentry_sentry__get_sentry_resource` / `…search_issues` with the actually-wired server name (this session: `mcp__claude_ai_Sentry__*`). Pin one canonical name and stop the worktrees from drifting (the UUID-named copy proves drift is already happening). Until fixed, Sentry-URL debugging silently no-ops.

**P1.3 — Commit a project-scoped `.mcp.json`** at repo root for the dalgo server so the wiring is shared and version-controlled, with tight server instructions (≤2KB, since Claude Code truncates there). Keep secrets out — use `headers`/`headersHelper` or env, not a literal token in the file. This is what lets teammates and any future background agent use the same tools.

### P2 — Live-state validation & debugging (the high-leverage moves)

**P2.1 — Add a "Live State Verification" step to `validate-spec`** (read-only tools only). After the existing lint/test/diff scan passes, for features that touch the live platform:

   - **Migration populated data:** if the diff adds a model/table, call `dalgo_list_tables` → `dalgo_get_table_row_count` → `dalgo_get_table_columns` to confirm the table exists with the expected columns and isn't empty. Tests can pass on an empty DB; this catches "shipped but no data flows."
   - **Chart/dashboard features render:** call `dalgo_render_chart` / `dalgo_get_chart_data` against a known chart to confirm it renders without erroring on the live warehouse — catches the domain-map's classic failure: a column rename upstream silently breaking a chart's `extra_config`.
   - **Pipeline features actually run:** `dalgo_get_pipeline_run_history` on the affected pipeline to confirm recent runs are green.
   - This turns validate-spec from "the code matches the spec" into "the code matches the spec **and the live system agrees**." Frame it as a soft gate: findings are reported, never auto-fixed (consistent with validate-spec's read-only contract).

**P2.2 — Give `debug-issue` / the `debugger` agent live pipeline access.** Add to the Gather phase: when the bug is a pipeline/sync/dbt failure (Airbyte / Prefect / dbt indicators it already lists), pull the real run:
   - `dalgo_get_pipeline_run_history` → find the failing run → `dalgo_get_flow_run` + `dalgo_get_flow_run_logs` for the actual error, and `dalgo_get_sync_history` / `dalgo_get_git_status` (dbt) for context.
   - This is the difference between guessing from a stack trace and reading the failure. Pairs naturally with the existing Sentry path (Sentry for app errors, dalgo for orchestration errors).

**P2.3 — Cross-check generated docs against live product docs in the `documentation` skill.** Before writing a page, call `dalgo_search_docs` / `dalgo_list_docs` / `dalgo_get_doc` to (a) find the canonical term the product already uses, (b) avoid duplicating an existing page, (c) match existing structure. Read-only, zero risk, directly improves doc consistency.

**P2.4 — Use the dalgo MCP to ground `plan-feature` blast-radius analysis.** `domain-map.md` is the source of truth, but its many `draft` / `tribal-knowledge-needed` entries could be *verified against live state* during planning: `dalgo_list_charts` / `dalgo_list_dashboards` / `dalgo_list_reports` to enumerate *actual* consumers of an entity before claiming the blast radius is complete. (Read-only; complements, doesn't replace, the map.)

### P3 — Additions & longer bets

**P3.1 — Add a Postgres/BigQuery **read-only** MCP server** scoped to a **staging** warehouse, for deeper SQL the dalgo tools don't expose (ad-hoc joins, EXPLAIN). Strictly read-only role at the DB grant level — defense in depth beyond the permission rules. Only if P2.1's dalgo tools prove insufficient.

**P3.2 — Add a Linear MCP** so the backlog item "agent picks up a ticket and runs the full pipeline" (harness-evolution backlog) works end-to-end — read ticket → plan → build → PR → comment back. Today there's a `linear-issue-from-plan` skill but no read path.

**P3.3 — Add a Grafana/Loki (LogQL) + Prometheus (PromQL) MCP** for the harness-evolution backlog item "Log/metrics access wired into verify step." Lets the verify/validate step confirm a change didn't spike error rates.

**P3.4 — Trim deferred-tool surface where useful.** With 70+ dalgo tools, if Tool Search ever mis-selects, consider `ENABLE_TOOL_SEARCH=auto` (load common tools upfront, defer the long tail) or splitting rarely-used dalgo tools. Measure first — default deferral is working in this session.

**P3.5 — Treat dalgo MCP tool descriptions as a product surface we own.** Since we control the dalgo MCP server, apply §1.4 to it: add `input_examples` to nested-param tools (`dalgo_create_chart`'s `extra_config` JSON blob is the prime candidate — the domain-map flags it as "brittle to LLM introspection"), and write error messages that guide (e.g. "table not found; call dalgo_list_tables for valid schema.table names"). This improves every agent that touches Dalgo, not just this harness.

---

## 4. Open questions / experiments

1. **Which org/warehouse does the dalgo MCP point at — staging or a partner NGO's production?** This is load-bearing for everything in P2. If it's **production NGO data**, read-only-only isn't just tidy — it's a **data-privacy requirement** (donor compliance, AGPL framing in CLAUDE.md), and even read tools need care (don't dump beneficiary PII into a transcript or a committed report). **Resolve this before enabling P2.1/P2.2.** Ideal: a dedicated staging/dev org for the harness.

2. **Does live-state validation reduce validate-spec false-passes?** Experiment: ship a feature where tests pass but the migration didn't populate (the harness-evolution Iteration 5 "tests pass but UI breaks" pattern). Does P2.1's `dalgo_get_table_row_count` catch it? Measure first-pass quality before/after.

3. **Does live pipeline log access shorten debug time?** Plant a known Airbyte/dbt failure, run `debug-issue` with and without P2.2's `dalgo_get_flow_run_logs`. Compare root-cause accuracy and number of code-reading round-trips.

4. **PII in MCP output.** Read tools like `dalgo_get_table_data` can return beneficiary records. Should there be a `deny` on row-level data tools (keep only aggregate/count/schema tools in `allow`), or a redaction hook? The existing `kavach` PII-guard skill may be the right enforcement point.

5. **Tool Search selection accuracy at 70+ tools.** Does the model reliably pick `dalgo_get_flow_run_logs` over the dozen other `dalgo_get_*` tools? If not, sharpen those descriptions (§3.5) and/or test `ENABLE_TOOL_SEARCH=auto`.

6. **Background/CI agents and OAuth.** Remote dalgo MCP auth via `/mcp` OAuth is interactive. For a scheduled doc-gardening or PR-triage agent (harness-evolution Iter 4), how does a non-interactive agent authenticate? Likely a headers-based service token via `headersHelper` — needs design.

---

## Sources

- Claude Code MCP reference — <https://code.claude.com/docs/en/mcp> (scopes, transports, tool naming, Tool Search config, output limits)
- Claude Code permissions — <https://code.claude.com/docs/en/permissions> (deny→ask→allow precedence, `mcp__server__` glob rules)
- MCP protocol — <https://modelcontextprotocol.io/introduction>
- Anthropic, "Writing effective tools for AI agents" — <https://www.anthropic.com/engineering/writing-tools-for-agents>
- Anthropic, "Introducing advanced tool use" — <https://www.anthropic.com/engineering/advanced-tool-use> (Tool Search, programmatic calling, input_examples, token figures)
- agentailor.com, "Writing Effective Tools for AI Agents" (secondary summary of the Anthropic post, used where the primary page was unreachable) — <https://blog.agentailor.com/posts/writing-tools-for-ai-agents>

**Further reading (not cited above):** Anthropic, "Effective context engineering for AI agents" — <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>
