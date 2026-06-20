# Building MCP Servers for Production

Research and practical guides on how teams design, secure, ship, and maintain Model Context Protocol servers in production — with real case studies, code, and a build-it-yourself tutorial.

| # | Guide | What it covers |
|---|-------|----------------|
| 1 | [Production Architecture](01-production-architecture.md) | Transports (Streamable HTTP), stateful vs stateless, deployment, scaling, reference architectures |
| 2 | [Server Design Best Practices](02-server-design-best-practices.md) | Tools vs resources vs prompts, tool design, response shaping, the "too many tools" problem, token efficiency |
| 3 | [Security & Auth](03-security-and-auth.md) | OAuth 2.1 Resource-Server pattern, tool poisoning, the lethal trifecta, multi-tenancy, supply chain, checklist |
| 4 | [Case Studies](04-case-studies.md) | 20+ real production MCP servers (Sentry, GitHub, Stripe, Block, Notion, databases…) grouped by category + cross-cutting lessons |
| 5 | [Building by Category](05-building-by-category.md) | Patterns by server type: database, API-wrapper, SaaS, internal-tools, filesystem, browser, gateway — and when NOT to build one |
| 6 | [Testing & Observability](06-testing-and-observability.md) | MCP Inspector, evals for tool use, production telemetry, circuit breakers, versioning without rug-pulls |
| 7 | [Build Your First MCP](07-build-your-first-mcp.md) | End-to-end TypeScript tutorial: a Support Tickets server from `npx` to an authenticated, containerized Streamable HTTP service |

## The recurring lesson across every guide

**Tools are not API endpoints.** Independent teams — Block (Square's 200+ endpoints → 3 layered tools), Notion (dropped nested JSON for Markdown), Datadog (rebuilt after a 1:1-wrap v1 failed), Anthropic (code-execution-with-MCP: 150k → 2k tokens) — all reached the same conclusion separately. The instinct to mechanically map your existing API surface to tools is actively wrong for LLM consumption.

## Reading order

- **Building one?** 1 → 2 → 5 → 7, with 3 (security) before you ship.
- **Deciding whether to?** Start with 5's "when NOT to build an MCP server" and 4's case studies.
- **Already in production?** 6 (testing/observability) + 3 (security audit).

## Cross-cutting findings worth knowing

- **Streamable HTTP is the production transport; SSE is deprecated.** MCP is also moving stateless-first (SEP-2575).
- **Tool descriptions are a higher-ROI lever than model upgrades** — and the description string *is* the API; guard it with evals.
- **"Too many tools" causes accuracy collapse, not graceful decay** — ~13% vs ~43% tool-selection accuracy with vs without retrieval; schemas can eat 40–50% of context.
- **Read-only-by-default breaks the exfiltration leg** of the lethal trifecta — the cheap mitigation databases converged on after the Supabase MCP incident.
- **MCP is not always the answer** — one eval showed MCP costing >6x and taking ~5x longer than equivalent CLI+Skills. Use MCP when many teams/agents need the same integration surface; otherwise a CLI, skill, or plain function may be better.
