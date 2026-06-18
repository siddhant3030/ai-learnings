# Subagents in the Dalgo Harness

How to push the Dalgo engineering harness further with Claude Code subagents. Combines
the latest Anthropic docs (June 2026) with the harness as it exists today in
`.claude/agents/`. Optimized for concrete changes to *this* repo.

Companion docs: `docs/harness-evolution.md` (roadmap), `docs/domain-map.md` (blast radius).
Deep orchestration (multi-agent pipelines, parallel fan-out) is covered separately; this
file stays focused on **how to design and configure individual subagents**.

---

## 1. Feature reference

A **subagent** is a specialized assistant Claude Code spawns to handle a focused task in
**its own fresh context window**, with its own system prompt, tool access, and model. The
parent only receives the subagent's final message — all the intermediate tool calls,
file reads, and logs stay in the subagent's context.

**Why this matters (the core benefit): context isolation.** When a side task would flood
the main conversation with search results, logs, or file contents you won't reference
again, a subagent does that work in its own window and returns only a summary. This is the
single most important reason to reach for a subagent.

Subagents also let you:
- **Enforce constraints** — limit which tools a subagent can use (least privilege).
- **Specialize** — a focused system prompt for one domain.
- **Control cost** — route cheap work to Haiku, reserve Opus for hard reasoning.

### How they are defined

Markdown files with YAML frontmatter in `.claude/agents/` (project) or `~/.claude/agents/`
(user). Body = system prompt. Required fields: `name`, `description`. Key optional fields:

| Field | Purpose |
|-------|---------|
| `description` | **Drives auto-delegation.** Claude reads it to decide when to hand off. Include "Use proactively after X" to encourage delegation. |
| `tools` | Allowlist. Omit = inherit all tools. e.g. `Read, Grep, Glob`. |
| `disallowedTools` | Denylist applied to the inherited set. e.g. `Write, Edit`. Supports `mcp__<server>` patterns. |
| `model` | `opus` / `sonnet` / `haiku` / `inherit` / full ID. **Defaults to `inherit`.** |
| `skills` | Preload full skill content into the subagent at startup (not just the description). |
| `memory` | `project` / `user` / `local` — persistent dir for cross-session learning. |

Full frontmatter table (incl. `permissionMode`, `hooks`, `maxTurns`, `isolation: worktree`,
`background`, `effort`, `color`): see https://code.claude.com/docs/en/sub-agents

### What a subagent inherits at startup

Fresh, isolated context — **no parent conversation history, no files the parent already
read, no skills already invoked.** It gets: its own system prompt + env details, the
delegation prompt Claude writes, CLAUDE.md / memory hierarchy, and a git-status snapshot.
The only parent→child channel is the delegation prompt string, so any file path, error, or
decision the subagent needs must be in that prompt. (Built-in Explore and Plan skip
CLAUDE.md and git status to stay fast.)

### Built-in subagents (no definition needed)

- **Explore** — Haiku, read-only. Fast codebase search. The harness already uses these in
  `plan-feature`.
- **Plan** — inherits model, read-only. Research during plan mode.
- **general-purpose** — inherits model, all tools. Multi-step explore + modify.

### Model selection guidance

- **Haiku** — fast, cheap, low-latency: search, file discovery, mechanical checks.
- **Sonnet** — balances capability and speed: code analysis, advisory lenses, spec writing.
- **Opus** — hard reasoning, cross-service tracing, high-stakes review.
- Use **aliases** (`opus`/`sonnet`/`haiku`), not pinned IDs like `claude-opus-4-8`, so
  agents survive model upgrades. (`fable` also exists as an alias; not characterized here.)

### Subagent vs main thread vs skill

- **Main thread** when: frequent back-and-forth, phases share context (plan→build→test),
  quick targeted change, or latency matters (subagents start cold).
- **Subagent** when: verbose output you don't need in main context, you want tool/permission
  restrictions, or the work is self-contained and returns a summary.
- **Skill** when: you want a reusable prompt/workflow that runs *in the main context* — no
  isolation, no separate tools. (Decision rule worth internalizing for the harness.)

### Parallel vs sequential (brief — orchestration covered elsewhere)

- **Parallel**: spawn multiple subagents for genuinely independent subtasks; they finish in
  the time of the slowest, not the sum. Best when paths don't depend on each other.
  Caveat: each subagent's result returns to the parent — many detailed results can
  themselves bloat the main context.
- **Sequential / chained**: when step N's output feeds step N+1. Claude passes relevant
  context forward between hops.
- The harness already does both: `plan-feature` fans out Explore agents; `design-handoff`
  spawns one Figma agent per surface in parallel.

### Claude Agent SDK relevance

The Agent SDK defines subagents programmatically (`AgentDefinition`: `description`,
`prompt`, `tools`, `model`, `memory`, …) — same concepts, code instead of markdown. **Not
something to adopt now.** Filesystem `.claude/agents/` is the right choice for the
interactive harness. The SDK is the **future headless / CI path** — relevant when the
backlog items land: "Linear ticket → full pipeline" and the scheduled doc-gardening agent
(`/schedule`). Do not rewrite the existing agents in the SDK.
Ref: https://code.claude.com/docs/en/agent-sdk/subagents

---

## 2. Current state in the Dalgo harness

Four project agents in `.claude/agents/`:

| Agent | Model | `tools` | `memory` | Role |
|-------|-------|---------|----------|------|
| `debugger` | opus | (inherits all) | — | Diagnose bugs across the stack |
| `senior-product-manager` | sonnet | (inherits all) | — | Strategy + spec writing |
| `ux-design-expert` | sonnet | (inherits all) | — | UI/UX decisions |
| `ngo-data-platform-consultant` | sonnet | (inherits all) | `project` | "Priya" NGO-user lens |

### What's done well

- **Strong, specific system prompts.** All four have rich, domain-grounded personas with
  concrete output formats. The Priya persona and the PM's "Excel test" are genuinely useful
  evaluation lenses, not generic.
- **Sensible model tiers in principle** — Opus for the cross-stack debugger, Sonnet for the
  advisory/spec roles.
- **Aliases, not pinned IDs** — already correct; agents will survive model upgrades.
- **Commands lean on built-in Explore agents** (`plan-feature`) and parallel custom agents
  (`design-handoff`) — the isolation benefit is already being exploited for research.

### Gaps

1. **No tool restrictions on any agent.** All four inherit every tool — including Write,
   Edit, Bash, and ~90 MCP tools (Dalgo, Figma, Playwright, Serena, Sentry…). Advisory
   agents that should only produce text can edit files and run shell commands. This is the
   biggest, easiest win.
2. **No code-reviewer / self-review agent** — despite `harness-evolution.md` Iteration 2
   explicitly calling for a self-review subagent between validate and docs. `review-pr` is a
   *command* that reviews inline in the main context; it does not isolate review or run as a
   restricted reviewer.
3. **Memory is half-wired and partly orphaned.** `ngo-data-platform-consultant` declares
   `memory: project` but (a) `.claude/agent-memory/ngo-data-platform-consultant/` is empty,
   (b) the agent body has **zero read/write-memory instructions** (docs say you must include
   them or it won't curate), and (c) there's an orphaned `agent-memory/senior-product-strategist/`
   dir matching no agent.
4. **Stale Sentry MCP tool names in `debugger`.** It references
   `mcp__plugin_sentry_sentry__get_sentry_resource` / `__search_issues`. The Sentry MCP in
   this environment uses the `mcp__claude_ai_Sentry__*` prefix — the `plugin_sentry_sentry`
   names appear stale and will silently fail.
5. **Uneven `description` fields** for auto-delegation — `senior-product-manager` is rich;
   `ux-design-expert` and `debugger` are thinner and lack the documented "use proactively"
   trigger wording.

---

## 3. Recommendations

### P1 — do these first

**P1.1 — Add per-agent tool restrictions matching each agent's output contract.**
The grant should match what the agent actually produces — not a blanket template. This is
what makes it Dalgo-specific rather than a copy of the docs' read-only example.

| Agent | Recommended frontmatter | Why |
|-------|-------------------------|-----|
| `ngo-data-platform-consultant` | `tools: Read, Grep, Glob` | Pure advisory lens — produces text only. No Write/Edit/Bash, no MCP. |
| `ux-design-expert` | `tools: Read, Grep, Glob` | Same — design recommendations are text. Add `WebFetch` only if it cites external refs. |
| `senior-product-manager` | `tools: Read, Grep, Glob, Write` | **Writes** specs to `features/{name}/spec.md`, so it needs Write — but not Edit/Bash/MCP. |
| `debugger` | `tools: Read, Grep, Glob, Bash` + Sentry MCP; **no Edit/Write** | Its own Output Format says *propose* a minimal diff and defer multi-service fixes to `/plan-feature`. It is diagnosis-only. |

Note the `debugger` divergence is **deliberate**: Anthropic's example debugger includes
`Edit` because it fixes bugs. Dalgo's debugger is contractually diagnosis-only, so it should
*not* have Edit. Calling this out keeps the divergence intentional, not careless.

Example (`ngo-data-platform-consultant.md` frontmatter):
```yaml
---
name: ngo-data-platform-consultant
description: "Evaluates features, workflows, and docs from Priya's (non-technical NGO
  program manager) perspective. Use proactively when designing user flows, simplifying
  technical concepts, or writing user-facing copy."
model: sonnet
tools: Read, Grep, Glob
---
```

**P1.2 — Create a `code-reviewer` subagent (Iteration 2 of the evolution plan).**
This is the headline recommendation — it advances the harness's own stated roadmap. A
read-only reviewer, spawned between validate and docs in the build pipeline, reviewing the
diff in isolated context and returning blocking findings. Preload the architecture
`landmarks.md` via `skills` so it reviews with conventions in context.

```yaml
---
name: code-reviewer
description: "Reviews a diff for Dalgo conventions before a human sees the PR: missing
  @has_permission / org-scoping, hardcoded colors, raw toast(), 'any' types, import-inside-
  function. Use proactively after implementing a feature and before opening a PR."
model: sonnet
tools: Read, Grep, Glob, Bash
skills:
  - backend-architecture
  - frontend-architecture
---

You are a senior reviewer enforcing Dalgo's backend (Django/Ninja) and frontend (Next.js)
conventions. When invoked: run `git diff` against the base branch, focus on changed files,
and report findings by severity (Critical / Warning / Suggestion). Check: org-scoping and
@has_permission on every endpoint, no hardcoded hex (use CSS vars), toastSuccess/toastError
not raw toast(), no `any`, no imports inside function bodies, test coverage for new paths.
Provide a specific fix for each finding. You do not edit files — you report.
```

Test (from the evolution plan): plant a missing `@has_permission`, a hardcoded hex, and a
`console.log`; confirm the reviewer catches all three before PR.

**P1.3 — Fix the `debugger`'s Sentry tool names.** Verify against the current
`mcp__claude_ai_Sentry__*` server and replace the stale `mcp__plugin_sentry_sentry__*`
references in `debugger.md`. (Flagging the mismatch and direction; confirm exact post-auth
resource/search tool names against the live server before editing.)

### P2 — soon

**P2.1 — Rationalize agent memory.**
- **Remove** `memory: project` from `ngo-data-platform-consultant`. Memory's documented
  purpose is accumulating *codebase patterns, debugging insights, architectural decisions* —
  a fixed persona lens (Priya) doesn't accumulate facts, so the field earns nothing there.
- **Delete** the orphaned `.claude/agent-memory/senior-product-strategist/` dir.
- **Move memory onto the agents that earn it**: add `memory: project` to the new
  `code-reviewer` and to `debugger`, and include explicit memory instructions in the body
  (without them the agent won't curate):
  ```markdown
  Update your agent memory as you discover recurring bug patterns, fragile codepaths, and
  architectural decisions. Write concise notes about what you found and where. Consult your
  memory before starting a new diagnosis.
  ```

**P2.2 — Tighten `description` fields for auto-delegation.** Add the documented
"Use proactively after/when X" trigger to `ux-design-expert` and `debugger` so Claude
delegates without being told. Keep them specific about *when*, not *what*.

**P2.3 — Wire the `code-reviewer` into the build pipeline.** Add a self-review step to the
`executing-feature-plans` flow (or `review-pr`) that spawns `code-reviewer`, applies
blocking fixes, and loops up to twice — closing the Iteration 2 loop end-to-end.

### P3 — later / experimental

- **`docs-gardener` agent** (Iteration 4) — read-mostly agent run via `/schedule` that scans
  for stale docs and TODOs without issue refs, opens cleanup PRs. Restricted tools + memory.
- **`verify-runner` agent** (Iteration 5) — `isolation: worktree` + Playwright MCP scoped
  via `mcpServers`, runs the stack and observes whether UI renders. Worktree isolation keeps
  it from clobbering the main checkout.
- **`PreToolUse` hook on `debugger`/`code-reviewer`** to mechanically block write commands —
  belt-and-suspenders beyond the `tools` allowlist (see the db-reader hook pattern in docs).

---

## 4. Open questions / experiments

1. **Does the isolated `code-reviewer` catch more than inline `review-pr`?** Run both on the
   same diff with planted violations; compare catch rate. Hypothesis: isolation + preloaded
   landmarks beats reviewing inline in a context already biased toward the code it wrote.
2. **Does agent memory actually improve `debugger` over time?** Track whether
   "Related Issues" recall improves across sessions once memory + instructions are added, or
   whether `MEMORY.md` just accumulates noise.
3. **Haiku for a search-only sub-lens?** Could a Haiku `landmark-finder` replace some Explore
   spawns in `plan-feature` for known lookups, cutting cost further?
4. **Skills-preload vs runtime discovery** — does `skills: [backend-architecture, ...]` on
   the reviewer measurably change findings vs letting it discover skills mid-run?
5. **When does this graduate to the Agent SDK?** Define the trigger: once the Linear-ticket
   pipeline or scheduled gardening runs headless in CI, the same `.claude/agents/` files can
   back SDK `AgentDefinition`s — but only then.

---

## Sources

- Create custom subagents — https://code.claude.com/docs/en/sub-agents
- Subagents in the Agent SDK — https://code.claude.com/docs/en/agent-sdk/subagents
- OpenAI Harness Engineering (harness thesis) — https://openai.com/index/harness-engineering/
