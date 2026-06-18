# Sentry

Two guides on getting more out of Sentry — one for AI workflows, one for setup.

| # | Guide | For |
|---|-------|-----|
| 1 | [Sentry MCP Tools](01-sentry-mcp-tools.md) | Using the Sentry MCP with Claude Code — AI-assisted debugging, auto-triage, Seer root-cause, 15 experiments to try, setup & security |
| 2 | [Sentry API & Error Tracking](02-sentry-api-error-tracking.md) | Setting up Sentry for the Dalgo stack (Django + Django Ninja, Next.js, FastAPI) — config, distributed tracing, dashboards, alerts, rollout plan |

## Two live issues these guides surfaced in the codebase

1. **Unhandled API 500s are invisible in Sentry.** `ddpui/routes.py` has a catch-all `@src_api.exception_handler(Exception)` that returns a 500 response but never calls `sentry_sdk.capture_exception` — so Django's `got_request_exception` signal never fires and auto-capture is bypassed. One-line fix (guide 2, Phase 0). It also leaks `str(exc)` to end users.

2. **The harness calls a dead MCP tool.** `.claude/commands/engineering/debug-issue.md` and `.claude/agents/debugger.md` invoke `mcp__plugin_sentry_sentry__get_sentry_resource`, which doesn't exist on the current Sentry MCP. Replace with `mcp__claude_ai_Sentry__get_issue_details` (guide 1, Section 8).

Both are independent of the research and worth fixing in their repos.
