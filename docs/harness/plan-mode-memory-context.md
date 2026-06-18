# Plan Mode, Memory & Context Management in the Dalgo Harness

**Status:** Research / recommendations
**Date:** 2026-06-17
**Companion to:** `docs/harness-evolution.md` (this extends Iteration 1: Context Architecture)

Goal: push the Dalgo engineering harness further using Claude Code's plan mode,
memory, and context-management features. The harness runs **long, multi-repo
sessions** (spec → design → plan → build → review across `DDP_backend` and
`webapp_v2`), which is exactly where context discipline pays off.

---

## 1. Feature reference

### 1a. Plan mode (a runtime state — not our `plan.md`)

**Critical distinction up front.** The harness already has a "plan": the document
`plan.md` produced by `/engineering/plan-feature`. Claude Code's **plan mode** is a
different thing — a *runtime permission mode*, not a file. One is a durable artifact;
the other is a live safety gate. The best setup uses both together (see §3).

Plan mode is one of Claude Code's permission modes. Per the official docs, plan mode
allows **"Reads only"** without asking — "Claude reads files, runs shell commands to
explore, and writes a plan, but does not edit your source." The read-only guarantee is
the value: no `Edit`, `Write`, `MultiEdit`, no state-changing Bash, no commits until you
approve. (Source: [Permission modes](https://code.claude.com/docs/en/permission-modes#analyze-before-you-edit-with-plan-mode).)

- **Enter:** `Shift+Tab` cycles `default → acceptEdits → plan`; or prefix one prompt with
  `/plan`; or start with `claude --permission-mode plan`; or set
  `"permissions": {"defaultMode": "plan"}` in `.claude/settings.json`.
- **Exit / approve:** when the plan is ready Claude presents it and offers: approve and
  start in auto mode, approve and accept edits, approve and review each edit, or keep
  planning with feedback. Approving exits plan mode into the chosen edit mode. `Shift+Tab`
  again leaves plan mode *without* approving. `Ctrl+G` opens the plan in your editor first.
- **Read-only guarantee caveat:** it is a *permission* guarantee (no writes), enforced by
  the client — not a sandbox. Protected paths (`.git`, `.claude`, etc.) are never
  auto-approved in any mode except `bypassPermissions`.

**When to gate work behind an approved plan:** "whenever the cost of an unintended action
exceeds the cost of approval" — multi-file refactors, migrations, auth, multi-tenancy /
org-scoping, production config, ambiguous specs. (Source:
[Plan mode guide](https://www.claudedirectory.org/blog/claude-code-plan-mode-guide).)

### 1b. Memory

Two complementary systems, both loaded at the start of **every** session
(Source: [How Claude remembers your project](https://code.claude.com/docs/en/memory)):

| | CLAUDE.md files | Auto memory |
|---|---|---|
| Who writes | You | Claude |
| Contains | Instructions, rules | Learnings, patterns |
| Scope | project / user / org | per-repo, shared across worktrees |
| Loaded | in full, every session | `MEMORY.md` first 200 lines / 25 KB |

**CLAUDE.md hierarchy** (broadest → most specific, later wins):
managed policy → `~/.claude/CLAUDE.md` (user) → `./CLAUDE.md` or `./.claude/CLAUDE.md`
(project) → `./CLAUDE.local.md` (gitignored personal). Ancestor files load in full at
launch; files in subdirectories load on demand when Claude reads files there.

**Best practices** (official):
- **Size:** target **under 200 lines** per CLAUDE.md. Longer files consume more context
  and *reduce adherence*. "If you cannot skim it between meetings, it is too long."
- **Imports:** `@path/to/import` pulls a file in — relative to the importing file, max
  depth **4 hops**. Imports help *organization* but **do not reduce context** — imported
  content expands inline at launch.
- **Specificity beats vibes:** "Run `npm test` before committing" > "test your changes."
- **Path-scoped rules** (`.claude/rules/*.md` with `paths:` frontmatter) load only when
  Claude touches matching files — the right home for repo- or area-specific rules.
- **HTML comments** (`<!-- ... -->`) in CLAUDE.md are stripped before injection — free
  maintainer notes that cost zero tokens.
- CLAUDE.md is **context, not enforcement**. For "must always happen at point X," use a
  **hook**, not a memory instruction.

**Auto memory** (v2.1.59+, on by default): Claude writes notes to itself at
`~/.claude/projects/<project>/memory/`, indexed by `MEMORY.md` (only its first 200 lines /
25 KB load at startup; topic files load on demand). Browse/edit via `/memory`. Add a
memory by telling Claude "remember that…". One-fact-per-file + a concise index is the
intended pattern.

**Agent memory:** subagents can keep their own auto memory if their frontmatter has
`memory:` — it loads a *separate* `MEMORY.md`, not the main session's
(Source: [Subagent memory](https://code.claude.com/docs/en/sub-agents#enable-persistent-memory)).

### 1c. Context management

The context window (~200K tokens) fills with: system prompt, auto-memory index,
CLAUDE.md (full), MCP tool names, skill listing, then your prompt + every file read +
tool output. **File reads dominate.**
(Source: [Explore the context window](https://code.claude.com/docs/en/context-window).)

- **`/context`** — shows what is using space (CLAUDE.md, MCP servers, skills, history).
  Disabling unused MCP servers frees real space.
- **`/compact`** — replaces the conversation with a structured summary. **What survives:**
  system prompt, CLAUDE.md (re-read from disk and re-injected), auto memory, MCP tools.
  **What does NOT:** the **skill listing is not re-injected** — only skills you actually
  invoked are preserved; and any instruction given *only in conversation* is lost unless it
  was summarized. Nested (subdirectory) CLAUDE.md files are not re-injected until Claude
  next reads a file there.
- **Auto-compaction** — fires automatically as the window fills; same survival rules.
- **`/clear`** — wipes history entirely; fresh start, nothing preserved.
- **Subagent isolation** — a subagent gets its **own** context window: loads CLAUDE.md +
  MCP/skills, but **not** your conversation history or the main session's auto memory.
  Only its final summary returns to you. The docs' worked example: a research subagent read
  **6,100 tokens of files and returned a 420-token summary** — that delta is the whole point.

---

## 2. Current state in the Dalgo harness

### What exists and works

- **CLAUDE.md is healthy.** ~133 lines — already under the 200-line target. Clean
  structure: repo table, workflow, agents, skills, artifacts, constraints. It correctly
  documents that `DDP_backend` / `webapp_v2` are **siblings**, so their CLAUDE.md does
  **not** auto-load — and `executing-feature-plans` tells the agent to read each repo's
  CLAUDE.md before writing code there. This is the right call given the load rules.
- **`tasks.md` checkpointing.** `executing-feature-plans` already uses `tasks.md` as a
  resume checkpoint and commits after each green slice. This is exactly the
  survive-compaction pattern — durable state on disk, not in conversation.
- **Subagents as read-only bookends.** The skill already restricts subagents to the two
  documented-safe uses: an `Explore` agent to map patterns (breadth, returns `file:line`
  pointers) and a code-review agent over a milestone diff. Implementation stays inline
  because tightly-coupled API-contract work is the documented weak spot for subagents.
  This is correct and matches the 6,100→420 isolation rationale above.
- **`docs/` is always-discoverable** (not skill-gated), which is the cheaper context path
  for reference material the agent needs to find on its own.

### Gaps

- **Plan mode is used nowhere.** A grep of `.claude/` and `docs/` finds **zero**
  references to plan mode / `ExitPlanMode` / permission modes. The pipeline has no
  read-only gate anywhere — risky slices (auth, org-scoping, migrations) go straight from
  plan.md to edits with no enforced "propose-before-touch" checkpoint. The harness conflates
  "we have a plan.md" with "we are protected from premature edits." We have the artifact, not
  the gate.
- **Auto memory is latent, not active.** The directory
  `~/.claude/projects/-Users-siddhant-Documents-Dalgo-dalgo-core/memory/` exists but is
  **empty — there is no `MEMORY.md`**, and both `.claude/agent-memory/*` dirs are empty too.
  The system is *available* (v2.1.59+ default-on) but **unpopulated**: no learnings are
  accumulating across the long sessions where they'd help most.
- **No compaction-survival design for long builds.** The resume protocol (read plan.md +
  tasks.md, start at first unfinished task) lives **only inside the skill**. But the skill
  *listing is not re-injected after compaction* — so a long `execute-plan` session that
  auto-compacts mid-build cannot rely on the skill being re-triggered to remember how to
  resume. The anchors that *do* survive are `plan.md` + `tasks.md` on disk, but nothing in
  durable context tells a post-compaction agent to go re-read them.
- **No context-hygiene rules.** Nothing tells long sessions when to `/compact`, when to
  offload to a subagent, or which MCP servers to drop. The available MCP surface here is
  large (dalgo, serena, playwright, figma, …) and each loaded server costs context.
- **`settings.json` doesn't set a default mode** and `.claude/rules/` is unused.

---

## 3. Recommendations

### P1 — High value, low effort

**P1.1 — Insert a plan-mode gate for risky slices in `executing-feature-plans`.**
The read-only guarantee makes plan mode the natural fit for the *Orient* phase and for
high-blast-radius slices. Add to the skill's process:

> Before implementing any slice that touches **auth, multi-tenancy / org-scoping, a DB
> migration, or a shared/cross-cutting module**, enter plan mode (`Shift+Tab` to `plan`),
> let the agent propose the change read-only, and require approval (`ExitPlanMode`) before
> editing. For ordinary CRUD slices, proceed normally.

This ties directly to **Key Constraints** (multi-tenant NGO data) and to
harness-evolution Iteration 1's security theme. Plan mode's read-only guarantee is
*mechanical* — it can't edit even if it "decides" to — which is the harness philosophy
("discipline in the scaffolding") applied to the edit boundary.

**P1.2 — Make the resume protocol survive compaction.** Move the 3-line resume contract
out of the skill (which isn't re-injected) into something that *is* re-read after
`/compact`. Add to project `CLAUDE.md` (or a non-path-scoped `.claude/rules/resume.md`):

> **Resuming a feature build:** the durable state is on disk, not in this conversation.
> If context was compacted mid-build, re-read `features/{name}/{version}/plan.md` and
> `tasks.md`, then continue from the first unchecked task. Never restart from scratch.

CLAUDE.md survives compaction by design; the skill listing does not. ~4 lines, no bloat.

**P1.3 — Seed auto memory deliberately.** The system is on but empty. Create the index and
let it accumulate. Suggested `MEMORY.md` skeleton (one fact per topic file):

```
# Dalgo harness memory (index)
- build/test commands per repo → backend.md, frontend.md
- recurring gotchas → gotchas.md
- org-scoping / multi-tenancy patterns → security.md
```

Then, in `executing-feature-plans` step 5, add: "If you hit and fix a non-obvious
build/test/setup gotcha, save it to auto memory (`# remember …`) so the next build
doesn't re-discover it." Keep `MEMORY.md` under 200 lines — detail goes in topic files,
which load on demand.

### P2 — Medium effort, compounding value

**P2.1 — Context-hygiene checklist for long builds.** Add a short section to
`executing-feature-plans` (or a rule):

- Run `/context` at the start of a multi-repo build; **disable MCP servers you won't use**
  this build (e.g. figma for a backend-only feature, playwright unless smoke-testing).
- Offload **breadth** to subagents (Explore for "where is X done across both repos?"),
  keep **the slice you're editing** inline. (Already the rule — restate the *why*: a
  subagent read of N files returns one summary; inline reads stay in your window forever.)
- `/compact` *before* starting a new milestone, not mid-slice — so the API contract you're
  holding in working memory survives into the GREEN step.
- Commit after every green slice (already required) — commits + `tasks.md` are the
  resume anchors if compaction or a crash intervenes.

**P2.2 — Path-scoped rules instead of growing CLAUDE.md.** Since CLAUDE.md is healthy,
*keep it that way*: when a backend- or frontend-specific rule comes up, put it in
`.claude/rules/backend.md` / `frontend.md` with `paths:` frontmatter so it loads only when
Claude touches those files. (Note: the sibling repos' own CLAUDE.md already carry
repo-specific rules; use harness-level rules for cross-repo workflow only.) This is the
"add without bloating" answer — the harness's CLAUDE.md should stay an index, not a manual.

**P2.3 — Default mode + plan-mode setting.** Consider
`"permissions": {"defaultMode": "plan"}` in `.claude/settings.json` for the *planning*
commands' working context, so exploration can't accidentally edit. (Leave `execute-plan`
in default/acceptEdits — it needs to write.) Test before committing team-wide; plan-mode
default can surprise users mid-flow.

### P3 — Experimental / larger

**P3.1 — Agent memory for the recurring reviewers.** Give the code-review and
`debugger` agents `memory:` frontmatter so they accumulate Dalgo-specific findings
(common org-scoping misses, recurring frontend toast/color violations from
harness-evolution Iteration 3) across features — a separate `MEMORY.md` that doesn't
pollute the main session.

**P3.2 — Plan-mode-first `/engineering/plan-feature`.** Today plan-feature writes plan.md
in a normal session. Running its discovery in plan mode would make the read-only guarantee
explicit during research and let the human approve the plan via `ExitPlanMode` before any
artifact is even written — unifying "the plan.md artifact" and "the approved plan gate."

**P3.3 — Compaction-aware checkpoints in `tasks.md`.** Have `execute-plan` write a
one-line "current slice / API contract in flight" note at the top of `tasks.md` before
each slice, so a post-compaction resume recovers not just *which task* but the *contract
detail* it was holding in working memory.

---

## 4. Open questions / experiments

- **Does a plan-mode gate reduce blast-radius misses?** Measure against
  harness-evolution's existing metric: ship two org-scoped features, one with the P1.1
  gate and one without; compare validate-retry count and PR review findings.
- **Does seeded auto memory cut repeat setup churn?** Track how often a build re-discovers
  a gotcha already in `gotchas.md`. Target: zero repeats after the first.
- **Where's the compaction cliff in a real multi-repo build?** Instrument a full
  `execute-plan` run with `/context` snapshots — when does auto-compaction first fire, and
  does the P1.2 resume rule actually fire correctly afterward? (Plant an interruption right
  after a compaction and see if the agent re-reads tasks.md vs restarts.)
- **Plan mode + auto mode interplay:** if the team adopts auto mode for long builds, note
  that classifier "boundaries" stated in conversation can be lost to compaction — for hard
  guarantees (never push to main, never touch prod) use deny rules, not chat instructions.
- **Subagent memory vs. main `docs/`:** is a `debugger` agent's private memory better than
  writing findings to `docs/`? Private memory is cheaper context but invisible to humans
  and other agents; `docs/` is discoverable but costs a read. Pick per signal type.

---

### Sources

- [Permission modes (plan mode, read-only guarantee)](https://code.claude.com/docs/en/permission-modes)
- [How Claude remembers your project (CLAUDE.md, imports, auto memory)](https://code.claude.com/docs/en/memory)
- [Explore the context window (compaction, subagent isolation, /context)](https://code.claude.com/docs/en/context-window)
- [Subagent persistent memory](https://code.claude.com/docs/en/sub-agents#enable-persistent-memory)
- [Ultraplan — plan mode in the cloud](https://code.claude.com/docs/en/ultraplan)
- [Plan mode: when to use it](https://www.claudedirectory.org/blog/claude-code-plan-mode-guide)
