# MCP Workflows — What People Build *With* MCP

The parent [`mcp/`](../) folder is about *building* MCP servers. This subfolder is about *using* them — the real workflows people build by connecting MCP servers to agents.

| # | Guide | Axis |
|---|-------|------|
| 1 | [Use-Cases Catalog](01-use-cases-catalog.md) | **What** people build — ~30 real workflows across dev, data, product, sales, support, personal, tagged by evidence level |
| 2 | [Multi-MCP Orchestration](02-multi-mcp-orchestration.md) | **How** to compose multiple servers — chaining, fan-out, gateways, agent- vs workflow-driven, code execution |
| 3 | [Automation Platforms](03-automation-platforms.md) | **Where** you build — Claude Code, Cursor, n8n, Zapier, LangGraph, OpenAI, Copilot, Goose |
| 4 | [Domain Playbooks](04-domain-playbooks.md) | **Who** builds what — role-based starter stacks for 7 roles |

## The throughline

Every guide lands on the same tension from a different angle: **the most useful MCP workflows are also the most dangerous ones.** The read-from-A / write-to-B pattern that makes multi-MCP powerful *is* the lethal trifecta. Read-only-by-default + human approval on consequential writes is the cross-cutting defense that shows up in every workflow that actually shipped.

## Findings worth knowing

- **Connectivity is commodity; context is the moat.** ClickHouse's internal assistant handles ~70% of warehouse questions for 250+ users — the leap from toy to primary interface came from a *business glossary* feeding the model, not the MCP connection.
- **The winning design move is reductive.** Block collapsed a 30+ tool Linear server to *two* query tools and the agent got more reliable (1 call vs 4–6). Wrapping every endpoint as a tool is the common, wrong instinct.
- **3–6 servers is the consensus sweet spot.** Past that, tool-selection accuracy collapses (it doesn't degrade gracefully) — unless you use lazy tool-loading (~85% token cut) or code execution (150K→2K tokens).
- **The platform choice collapsed to two axes** — no-code vs code, and local vs remote. Nearly everything speaks MCP now; local-vs-remote is the privacy-relevant split.
- **The ecosystem is quietly insecure.** Trend Micro found 492 internet-exposed servers with no auth; even Anthropic's reference SQLite server shipped a SQL-injection→prompt-injection hole. The viral "agent does my job" demos run on shaky infrastructure.

## Dalgo-relevant note

No warehouse MCP integrates with dbt — a dbt-heavy data team must run dbt's own 60+ tool server *alongside* the warehouse server, which immediately strains the 3–6 server sweet spot. Worth weighing for any Dalgo data-assistant work.
