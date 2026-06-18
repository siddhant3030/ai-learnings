# Harness Engineering — Overview & Mental Model

**Status:** Active research
**Created:** 2026-06-17
**Companion to:** [`../harness-evolution.md`](../harness-evolution.md) (the iterative plan) — this folder is the *research* behind it.

## What this folder is

Six deep-dives into how to use Claude Code's feature set to make the Dalgo
engineering harness ship features more reliably and autonomously. Each file pairs a
**feature reference** (how the capability works, current best practices from Anthropic's
docs) with **prioritized, concrete recommendations** for *this* harness — not generic advice.

| File | Capability | One-line thesis |
|------|-----------|-----------------|
| [`subagents.md`](subagents.md) | Subagents | Isolate context per role; restrict tools to each agent's contract; add a `code-reviewer`. |
| [`skills.md`](skills.md) | Skills | The `description` field is a trigger, not a summary; two skills are invisible (no SKILL.md). |
| [`commands-and-hooks.md`](commands-and-hooks.md) | Commands + hooks | **Highest leverage.** Hooks turn rules into mechanical guardrails and feedback loops. We have zero. |
| [`mcp-and-tools.md`](mcp-and-tools.md) | MCP + tools | The Dalgo MCP is the biggest untapped asset — and has no permission guardrails. |
| [`plan-mode-memory-context.md`](plan-mode-memory-context.md) | Plan mode, memory, context | We have the *artifacts* (plan.md, memory dir) but not the *runtime gates*. |
| [`orchestration-workflows.md`](orchestration-workflows.md) | Multi-agent orchestration | Make the spec→PR pipeline parallel, resumable, and adversarially self-checking. |

## How to think about harness engineering

The guiding thesis (from `harness-evolution.md`, after OpenAI's harness-engineering work):
**the discipline lives in the scaffolding, not the code.** When an agent writes the code,
your job shifts from writing lines to *engineering the environment the agent works in*. That
environment does three things:

1. **Encodes constraints mechanically.** A rule the agent *can* ignore (a sentence in
   CLAUDE.md saying "branch first") is weaker than a rule it *cannot* ignore (a PreToolUse
   hook that denies an edit on `main`). Push every load-bearing rule down the ladder:
   prose → skill instruction → command structure → **hook / permission deny**. The lower it
   sits, the less it depends on the agent choosing to comply.

2. **Builds feedback loops so the agent catches its own failures.** The most valuable hook
   is the one that runs the linter or test after an edit and feeds the failure *back into the
   same turn* — the agent self-corrects before a human ever sees it. Verification the agent
   performs on itself scales; verification only a human performs does not.

3. **Manages context as a budget.** Long multi-repo builds drown in context. Subagents exist
   primarily to *protect the main thread's context* — a researcher burns 6,000 tokens and
   returns a 400-token answer. Plan-mode gates, memory files, and `tasks.md` checkpoints all
   serve the same goal: keep the working set small and survive compaction.

A useful test for any proposed harness change: **"Does this move a rule down the ladder toward
mechanical enforcement, add a feedback loop, or shrink the context budget?"** If it does none
of those, it's probably documentation, not harness engineering.

## The ladder of enforcement

```
Weakest  →  Prose in CLAUDE.md           ("please branch first")
            Skill instruction            (loaded only if the skill triggers)
            Command structure            (the step exists in the pipeline)
            Subagent with restricted tools (agent physically lacks Edit)
            Permission rule (ask/deny)   (tool call paused or blocked)
Strongest → Hook (PreToolUse deny / PostToolUse feedback)  (deterministic, unskippable)
```

Most of the recommendations below are about moving specific rules one or more rungs down.

## Cross-cutting priorities (synthesized across all six files)

These surfaced in *multiple* research streams, which is why they lead.

### P1 — do these first

- **Build the first hooks.** We have none today, and this is the single highest-leverage gap.
  Start with two: a **PostToolUse auto-lint/format** hook that routes an edited file to its
  repo's toolchain and feeds errors back (the core feedback loop), and a **PreToolUse
  branch-guard** that denies edits while a target repo is on `main`. All hooks live in
  `dalgo-core/settings.json` and route by file path — hooks are *not* loaded from sibling-repo
  settings. See [`commands-and-hooks.md`](commands-and-hooks.md).

- **Fix the stale Sentry MCP tool names.** Both `debug-issue.md` and `debugger.md` call
  `mcp__plugin_sentry_sentry__*`, but the live server is `mcp__claude_ai_Sentry__*`. Sentry-URL
  debugging silently no-ops today. Flagged independently by the subagents and MCP streams —
  verify exact post-auth tool names before editing. See [`subagents.md`](subagents.md),
  [`mcp-and-tools.md`](mcp-and-tools.md).

- **Give `backend-architecture` and `frontend-architecture` real SKILL.md files.** They have
  only `landmarks.md`, no frontmatter — so they're invisible to autodiscovery and only work
  because `plan-feature.md` hardcodes their paths. See [`skills.md`](skills.md).

- **Put permission guardrails on the Dalgo MCP.** There are zero `mcp__*` rules today, so
  destructive production tools (`dalgo_delete_*`, `run_dbt`, `trigger_pipeline_run`) have no
  guardrail. Classify by mutation risk: allow read-only, `ask` on create/update, `deny` on
  destructive. Never wire a mutating tool into an autonomous step. Commit a project-scoped
  `.mcp.json` so the harness's biggest untapped asset is shared, not per-machine. See
  [`mcp-and-tools.md`](mcp-and-tools.md).

- **Add the self-review / `code-reviewer` gate.** This is Iteration 2 of the evolution plan
  and it still doesn't exist. A *fresh, independent* read-only reviewer over `git diff
  main...HEAD`, run between validate and docs, that must paste real test output (verifiers
  routinely rubber-stamp). Independence — clean context — is the key lever. Appears in the
  subagents, skills, hooks, and orchestration streams. See
  [`subagents.md`](subagents.md), [`orchestration-workflows.md`](orchestration-workflows.md).

### P2 — high value, after the P1 foundation

- **Add per-agent tool restrictions.** All four agents inherit every tool. Advisory lenses
  should be read-only; the debugger should diagnose without `Edit`/`Write`. See
  [`subagents.md`](subagents.md).
- **Add a plan-mode gate for risky slices** (auth, org-scoping, migrations) in
  `executing-feature-plans` — a *runtime* read-only gate, not just the plan.md artifact. See
  [`plan-mode-memory-context.md`](plan-mode-memory-context.md).
- **Use the Dalgo MCP in validation and debugging** — live row counts and chart renders in
  `validate-spec`; live Prefect/dbt run logs in `debug-issue`. See
  [`mcp-and-tools.md`](mcp-and-tools.md).
- **Parallelize the review stage across lenses** (security/multi-tenancy, design+NGO, correctness)
  — fan-out then union-of-blocking-findings. Keep *implementation* inline; don't split a coupled
  endpoint+UI contract across agents. See [`orchestration-workflows.md`](orchestration-workflows.md).
- **Make the resume protocol survive compaction** — move the "re-read plan.md + tasks.md"
  contract into CLAUDE.md (re-read from disk) rather than a skill listing (dropped after
  `/compact`). Seed the empty memory system. See [`plan-mode-memory-context.md`](plan-mode-memory-context.md).

### P3 — once the loop is solid

- Promote the `pipeline.md` orchestrator out of `.claude/worktrees/*` onto the working branch,
  then finally **run Experiment 0** (command-mode vs agent-mode) and fill the empty results
  table. Default to agent-mode (clean context can't drift). See
  [`orchestration-workflows.md`](orchestration-workflows.md).
- Wire scheduled/headless automation: weekly doc-gardening, daily stale-PR triage,
  `claude -p` CI gates.
- Clean up the `productivity` skill (name/dir mismatch, foreign cruft); promote `grill-me`
  and `tal-lens` to top-level skills. See [`skills.md`](skills.md).

## How this connects to the evolution plan

`harness-evolution.md` lists numbered iterations; this research grounds several of them:

| Evolution iteration | Where the research lands |
|---------------------|--------------------------|
| Iteration 1b (architecture skills) | `skills.md` — give them SKILL.md files |
| Iteration 2 (self-review gate) | `subagents.md` + `orchestration-workflows.md` — `code-reviewer` |
| Iteration 3 (auto-lint/test) | `commands-and-hooks.md` — PostToolUse feedback hook |
| Iteration 4 (doc-gardening) | `orchestration-workflows.md` — scheduled headless runs |
| Iteration 5 (completion gates) | `commands-and-hooks.md` — Stop hook requiring test output |
| Experiment 0 (command vs agent) | `orchestration-workflows.md` — run it, default agent-mode |

## Suggested reading order

1. This overview.
2. [`commands-and-hooks.md`](commands-and-hooks.md) — the highest-leverage, mostly-empty area.
3. [`subagents.md`](subagents.md) + [`skills.md`](skills.md) — the parts of the harness that already exist and need tightening.
4. [`mcp-and-tools.md`](mcp-and-tools.md) — the untapped asset.
5. [`plan-mode-memory-context.md`](plan-mode-memory-context.md) + [`orchestration-workflows.md`](orchestration-workflows.md) — making long, multi-repo builds robust and resumable.
