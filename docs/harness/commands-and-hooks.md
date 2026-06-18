# Slash Commands + Hooks — Harness Reference & Recommendations

**Status:** Reference + proposal
**Scope:** How Claude Code slash commands and hooks work, what the Dalgo harness has today, and the concrete hooks/commands to add next.
**Thesis link:** This is the highest-leverage area for "the discipline is in the scaffolding, not the code." Commands *encode procedures*; hooks *encode constraints mechanically and build feedback loops*. Most of what `harness-evolution.md` calls Iteration 2 (self-review), Iteration 3 (mechanical enforcement), and Iteration 5 (verify-ran) is implemented as **hooks**.

Docs cited (Anthropic, fetched June 2026):
- Hooks reference — https://code.claude.com/docs/en/hooks
- Skills + custom commands — https://code.claude.com/docs/en/slash-commands (redirects from `/en/docs/claude-code/slash-commands`)
- Settings — https://code.claude.com/docs/en/settings
- Permissions — https://code.claude.com/docs/en/permissions
- Subagents — https://code.claude.com/docs/en/sub-agents

---

## 1. Feature reference

### 1a. Slash commands are now the legacy surface of *skills*

The single most important recent change: **custom commands have been merged into skills.**
A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md`
both create `/deploy` and behave the same. Existing `.claude/commands/*.md` files keep
working unchanged and support the same frontmatter. Skills add: a directory for supporting
files, `context: fork` execution, and automatic (model-driven) invocation.
(https://code.claude.com/docs/en/slash-commands)

So treat "slash command authoring" and "skill authoring" as one system. The differences:

| | `.claude/commands/foo.md` | `.claude/skills/foo/SKILL.md` |
|---|---|---|
| Invocation | `/foo` (you type it) | `/foo` **and** Claude can auto-load when relevant |
| Supporting files | No (single file) | Yes (whole directory) |
| Run in subagent | No | Yes (`context: fork` + `agent:`) |
| Frontmatter | Same set | Same set + skill-only fields |

If a skill and command share a name, the **skill wins**.

### 1b. Frontmatter fields (commands and skills)

```yaml
---
description: One-line summary (shown in /help and to Claude for auto-invocation)
argument-hint: "[spec-path]"           # autocomplete hint, e.g. [issue-number]
arguments: [spec, version]             # named positional args → $spec, $version
allowed-tools: Read Grep Bash(gh *)    # granted without prompting while active
disallowed-tools: AskUserQuestion      # removed from the pool while active
disable-model-invocation: true         # only YOU can invoke (use for deploy/commit)
model: claude-opus-4-...               # pin a model for this command
context: fork                          # run in an isolated subagent (skills only)
agent: Explore                         # which subagent type for context: fork
---
```

Key behaviors:
- `allowed-tools` **grants** permission without prompting; it does **not** restrict the
  pool. Use `disallowed-tools` or `deny` rules to actually remove a tool.
- `disable-model-invocation: true` is the right default for anything with side effects
  (deploy, commit, PR-open) — you don't want Claude deciding to run it.

### 1c. Arguments

| Placeholder | Meaning |
|---|---|
| `$ARGUMENTS` | The full argument string as typed |
| `$ARGUMENTS[N]` / `$N` | The Nth arg, 0-based. `$0` = first, `$1` = second |
| `$name` | Named arg declared in `arguments:` frontmatter, mapped by position |

Indexed args use shell-style quoting: `/migrate "Search Bar" React` → `$0` = `Search Bar`,
`$1` = `React`. If a command has no `$ARGUMENTS`, Claude appends `ARGUMENTS: <value>` to the
content so it still sees the input. Escape a literal with a backslash: `\$1.00`.

### 1d. Bash injection (`!`) and file references (`@`)

Run a command **at expansion time** and inline its output into the prompt — preprocessing,
not something Claude executes:

```markdown
## Pull request context
- Diff: !`gh pr diff`
- Changed files: !`gh pr diff --name-only`
```

Multi-line uses a fenced block opened with ```` ```! ````. The `!` is only recognized at
line start or after whitespace. `@path/to/file` inlines a file's contents into the prompt.
This is preprocessing: Claude sees the rendered result, not the command.
A managed-settings switch `disableSkillShellExecution: true` neuters this.

### 1e. Invoking subagents from a command

Two routes:
1. **`context: fork` + `agent:`** (skill only) — the skill body becomes the subagent's task
   prompt; runs isolated with no conversation history. Good for read-only research
   (`agent: Explore`) or a self-contained review.
2. **Spawn via the Task tool from the command body** — the current pattern: the command
   text instructs Claude to launch a subagent (e.g. `debugger`, `senior-product-manager`).
   Keeps orchestration in the main session.

---

### 1f. Hooks — the mechanical layer

Hooks are shell commands (or HTTP/MCP/agent handlers) that Claude Code runs **at lifecycle
points**, configured in `settings.json`. They are the harness's only way to enforce a rule
*mechanically* — the model can ignore a CLAUDE.md instruction; it cannot ignore a hook that
denies the tool call.

**Canonical events** (the set the recommendations below rely on):

| Event | Fires | Can block? | Primary use |
|---|---|---|---|
| `PreToolUse` | before a tool runs | **Yes** | deny/allow/ask, or modify input |
| `PostToolUse` | after a tool succeeds | No (already ran) | feed lint/test results back to Claude |
| `UserPromptSubmit` | user submits a prompt | Yes | inject context, filter prompts |
| `Stop` | Claude finishes responding | Yes (force continue) | check verification ran before "done" |
| `SubagentStop` | a subagent finishes | Yes | gate subagent completion |
| `SessionStart` | session start/resume | No | inject branch/repo state as context |
| `PreCompact` | before context compaction | Yes | preserve state |
| `Notification` | Claude sends a notification | No | desktop/Slack alerts |

> The docs list many more events (e.g. `SessionEnd`, `PostToolUseFailure`, `PermissionRequest`,
> handler types `prompt`/`http`/`agent`, `if` permission conditions). They exist, but verify
> against the live hooks page before relying on them — the recommendations here use only the
> canonical layer above.

**The JSON I/O contract**

Every hook receives JSON on **stdin**:
```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/dir",
  "permission_mode": "default|plan|acceptEdits|bypassPermissions",
  "hook_event_name": "PreToolUse",
  "tool_name": "Edit",
  "tool_input": { "file_path": "/abs/path", "old_string": "...", "new_string": "..." },
  "tool_output": "..."          // PostToolUse only
}
```

A hook controls flow two ways:

**(1) Exit code** (simplest):

| Code | Effect |
|---|---|
| `0` | Success. stdout JSON is parsed (only on exit 0). |
| `2` | **Blocking error.** stderr is shown *to Claude*. Blocks the action on blocking events; on `PostToolUse` it can't un-run the tool but feeds stderr back so Claude self-corrects. |
| other | Non-blocking error. First line of stderr shown to user; execution continues. |

**(2) JSON on stdout** (richer). Top-level fields:
```json
{
  "continue": true,                    // false = stop Claude entirely
  "stopReason": "message when continue=false",
  "suppressOutput": false,
  "systemMessage": "warning shown to the user",
  "decision": "block",                 // event-specific
  "reason": "why",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "shown to Claude",
    "additionalContext": "injected into Claude's context",
    "updatedInput": { }                // PreToolUse: rewrite the tool call
  }
}
```

**PreToolUse specifically** can:
- **deny** — `permissionDecision: "deny"` + reason → tool call is blocked, reason goes to Claude.
- **allow** — `permissionDecision: "allow"` → run without the normal permission prompt.
- **ask** — `permissionDecision: "ask"` → force the permission dialog.
- **modify** — `updatedInput` → rewrite the command/args before it runs (e.g. swap a raw
  `git push` for `git push --no-verify`, or normalize a path).

**PostToolUse cannot prevent the edit** — the file already changed. Its leverage is the
**feedback loop**: run a linter/test, and on failure exit 2 with the errors on stderr (or
emit `additionalContext`). Claude reads that and fixes the code in the same turn.

### 1g. settings.json hooks + permissions config

```json
{
  "permissions": {
    "allow": ["Read", "Edit", "Bash(ls *)"],
    "deny":  ["Read(**/.env*)", "Bash(cat **/.env*)"],
    "additionalDirectories": ["../DDP_backend", "../webapp_v2"]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/guard.sh" }
        ]
      }
    ]
  }
}
```

- **Matcher** matches `tool_name`. Plain string or `A|B` list = exact match; anything with
  regex chars (`^`, `.*`, `mcp__memory__.*`) = regex; `"*"`/`""`/omitted = match all.
- **`${CLAUDE_PROJECT_DIR}`** resolves to the project root — always use it for script paths.
- **Hook scope:** `.claude/settings.json` (shared, committed), `.claude/settings.local.json`
  (gitignored, personal), `~/.claude/settings.json` (all projects), managed policy (org).

> **Cross-repo gotcha that defines every Dalgo hook (verify-before-shipping):**
> Hooks, subagents, and commands are **not loaded from `additionalDirectories`.** So a
> linter hook placed in `DDP_backend/.claude/settings.json` will **not** run in a
> dalgo-core session. **All hooks must live in dalgo-core's `.claude/settings.json` and
> route by file path** to the right sibling repo and its environment. Every script below
> does `file_path → repo → that repo's tool/venv`.

---

## 2. Current state in the Dalgo harness

**Commands present** (`.claude/commands/`, all single-file, `$ARGUMENTS` only):

| Command | Args | Notes |
|---|---|---|
| `product/write-spec` | `$ARGUMENTS` | mode A/B inferred from arg |
| `product/prototype` | `$ARGUMENTS` | |
| `design/design-handoff` | `$ARGUMENTS` + `--auto` flag (parsed in prose) | |
| `engineering/plan-feature` | `$ARGUMENTS` | invokes `plain-writing` skill |
| `engineering/execute-plan` | `$ARGUMENTS` | follows `executing-feature-plans` skill |
| `engineering/validate-spec` | `$ARGUMENTS` | runs `git diff main...HEAD` |
| `engineering/review-pr` | `$ARGUMENTS` | uses `gh pr view/diff` |
| `engineering/debug-issue` | `$ARGUMENTS` | spawns `debugger` agent |

Observations:
- **No frontmatter at all** — no `description`, `argument-hint`, `allowed-tools`, or
  `disable-model-invocation`. So none of these are auto-invocable-with-the-right-guardrails,
  none autocomplete an arg hint, and each tool call still prompts for permission.
- **No `!`/`@` injection** — `review-pr` and `validate-spec` *describe* running `gh pr diff`
  and `git diff` in prose instead of inlining them with `` !`...` ``. The model re-derives the
  command each run; injection would make the context deterministic and save a round-trip.
- **Flags parsed in prose** (`--auto`) rather than via `arguments:` positional args.

**Hooks present: NONE.** `settings.json` and `settings.local.json` contain only
`permissions` and a `statusLine`. There is no `hooks` key anywhere in the repo.

**Permissions:** broad `allow` (Read/Edit/Write/MultiEdit), a good `.env` `deny` block,
and `additionalDirectories` for both sibling repos. `settings.local.json` has accumulated a
long, ad-hoc `allow` list (specific worktree-creation commands, one-off `sed`/`grep` lines).

**The gap:** every rule in CLAUDE.md and the skills is advisory. "Branch first," "no local
imports," "`@has_permission` on every endpoint," "use `toastSuccess` not raw `toast`,"
"run tests before declaring done" — all are prose the model *should* follow. Nothing is
enforced mechanically, and there is no automatic feedback loop. This is exactly the
"encode constraints MECHANICALLY / build FEEDBACK LOOPS" layer the thesis calls for, and
it's empty.

---

## 3. Recommendations

All hook scripts assume a shared helper that maps an edited path to its repo. Create
`.claude/hooks/_repo.sh`:

```bash
#!/usr/bin/env bash
# Echoes the repo root for a given file path, or empty if not a code repo.
# Handles worktrees: DDP_backend_wt_xxx and webapp_v2_wt_xxx count too.
repo_for() {
  case "$1" in
    *"/DDP_backend"/*|*"/DDP_backend_wt_"*) echo "backend" ;;
    *"/webapp_v2"/*|*"/webapp_v2_wt_"*)     echo "frontend" ;;
    *) echo "" ;;
  esac
}
# Echoes the repo root directory containing the file (walks up to the .git boundary).
root_for() {
  local d; d="$(dirname "$1")"
  while [ "$d" != "/" ]; do
    [ -e "$d/.git" ] && { echo "$d"; return; }
    d="$(dirname "$d")"
  done
}
```

### P1 — Highest leverage (mechanical constraints + the core feedback loop)

**P1.1 — PostToolUse auto-format/lint on edit (FEEDBACK LOOP, Iteration 3).**
After any `Edit`/`Write`, run the right repo's formatter+linter on the changed file and feed
failures back. Backend = black + isort (already in pre-commit); frontend = prettier + eslint.

`.claude/hooks/lint-changed.sh`:
```bash
#!/usr/bin/env bash
source "$(dirname "$0")/_repo.sh"
FILE="$(jq -r '.tool_input.file_path // empty')"
[ -z "$FILE" ] && exit 0
REPO="$(repo_for "$FILE")"; ROOT="$(root_for "$FILE")"
[ -z "$REPO" ] && exit 0

if [ "$REPO" = "backend" ] && [[ "$FILE" == *.py ]]; then
  cd "$ROOT" || exit 0
  OUT="$(.venv/bin/black "$FILE" 2>&1 && .venv/bin/isort "$FILE" 2>&1)" || {
    echo "Formatting failed for $FILE:\n$OUT" >&2; exit 2; }
elif [ "$REPO" = "frontend" ] && [[ "$FILE" =~ \.(ts|tsx|js|jsx)$ ]]; then
  cd "$ROOT" || exit 0
  OUT="$(npx prettier --write "$FILE" 2>&1 && npx next lint --file "$FILE" 2>&1)" || {
    echo "Lint failed for $FILE:\n$OUT" >&2; exit 2; }
fi
exit 0
```
```json
"PostToolUse": [
  { "matcher": "Edit|Write|MultiEdit",
    "hooks": [ { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/lint-changed.sh", "timeout": 120 } ] }
]
```
Black/isort/prettier on a single file finish in well under a second; the exit-2-with-stderr
path turns a lint error into self-correction in the same turn. (Run `--write` so formatting
auto-applies; reserve exit 2 for errors the model must fix, like an undefined name.)

**P1.2 — PreToolUse "not on main" guard (MECHANICAL CONSTRAINT, branch-first).**
The executing-feature-plans skill says "branch first." Enforce it: block any
`Edit`/`Write` whose target repo is currently on `main`/`master`.

`.claude/hooks/guard-branch.sh`:
```bash
#!/usr/bin/env bash
source "$(dirname "$0")/_repo.sh"
FILE="$(jq -r '.tool_input.file_path // empty')"
ROOT="$(root_for "$FILE")"; [ -z "$ROOT" ] && exit 0
[ -z "$(repo_for "$FILE")" ] && exit 0   # only guard code repos
BRANCH="$(git -C "$ROOT" branch --show-current 2>/dev/null)"
if [ "$BRANCH" = "main" ] || [ "$BRANCH" = "master" ]; then
  jq -n --arg b "$BRANCH" '{hookSpecificOutput:{hookEventName:"PreToolUse",
    permissionDecision:"deny",
    permissionDecisionReason:("Refusing to edit on \($b). Create a feature branch or worktree first (see executing-feature-plans).")}}'
  exit 0
fi
exit 0
```
```json
"PreToolUse": [
  { "matcher": "Edit|Write|MultiEdit",
    "hooks": [ { "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/guard-branch.sh" } ] }
]
```
This is the cleanest example of a rule that *cannot be ignored*. Pair it with the worktree
workflow already in `settings.local.json`.

**P1.3 — PreToolUse secret/scope guard (MECHANICAL CONSTRAINT).**
The `.env` `deny` rules cover Read/cat, but not, e.g., an `Edit` that writes a literal secret
or a `Bash` that pipes `.env` through `base64`. A PreToolUse `Bash` matcher that greps the
command for `\.env`, `base64 .*env`, `curl .* -d`, etc., and denies, hardens the existing
deny list against paraphrase. Low effort, high blast-radius reduction (multi-tenant + PII).

**P1.4 — Add frontmatter to all 8 commands (cheap, immediate).**
Give each command a `description` and `argument-hint`, and set
`disable-model-invocation: true` on the side-effecting ones (`execute-plan`, `review-pr`,
anything that opens PRs). Example for `execute-plan.md`:
```yaml
---
description: Implement a feature end-to-end from its plan.md across DDP_backend and webapp_v2.
argument-hint: "[features/{name}/v1/plan.md]"
disable-model-invocation: true
---
```
Result: arg-hints in autocomplete, and Claude won't spontaneously start *building a feature*
just because a plan.md is open in context.

### P2 — Feedback loops at completion boundaries

**P2.1 — Stop hook: "did verification run?" (Iteration 5).**
Before Claude is allowed to declare done, check that tests actually ran this session. The Stop
hook can scan the transcript for a test invocation and `decision: "block"` if absent, forcing
Claude to run them.

`.claude/hooks/require-tests.sh` (sketch):
```bash
#!/usr/bin/env bash
INPUT="$(cat)"; TRANSCRIPT="$(jq -r '.transcript_path' <<<"$INPUT")"
# Only enforce if code files were edited this session.
grep -qE '"file_path".*(DDP_backend|webapp_v2).*\.(py|tsx?|jsx?)' "$TRANSCRIPT" || exit 0
if ! grep -qE '(pytest|npm (run )?test|jest)' "$TRANSCRIPT"; then
  jq -n '{decision:"block", reason:"You edited code but no test run is visible in this session. Run pytest (backend) / npm test (frontend) and report results before finishing."}'
  exit 0
fi
exit 0
```
Guard against loops: only block once (track a sentinel file keyed by `session_id`), so a
genuinely test-free task can still finish.

**P2.2 — SubagentStop self-review gate (Iteration 2).**
When the engineer subagent finishes, spawn (or require) a diff self-review against the three
planted-violation classes: missing `@has_permission`, hardcoded hex color, `console.log`/raw
`toast()`. A SubagentStop hook can `decision: "block"` with the review checklist as `reason`
until the agent confirms it ran. (Simpler alternative: a PostToolUse `Bash(git commit *)`
hook that greps the staged diff for those patterns and warns.)

**P2.3 — Convert `review-pr`/`validate-spec` to use `!` injection.**
Replace the prose "run `gh pr diff`" with inlined `` !`gh pr diff` `` / `` !`git diff main...HEAD --stat` ``
so the diff is in context deterministically at expansion time. Removes a model round-trip and
makes the command reproducible.

### P3 — Polish, observability, ergonomics

- **P3.1 — SessionStart context injection.** Inject current branch + dirty status of both
  sibling repos as `additionalContext`, so Claude starts every session knowing where it is
  (and the branch guard rarely has to fire).
- **P3.2 — Notification → Slack/desktop.** Wire a `Notification` hook so long autonomous
  runs (execute-plan) ping the engineer when they need input.
- **P3.3 — Garbage-collect `settings.local.json`.** The ad-hoc `allow` list has one-off
  `sed`/`grep`/worktree lines. Replace specific worktree-add entries with a `Bash(git -C * worktree add *)`
  pattern; drop the stale doc-generation lines.
- **P3.4 — Decide: migrate commands → skills?** Since commands are now the legacy surface,
  the engineering pipeline commands that already lean on skills (`plan-feature`→plain-writing,
  `execute-plan`→executing-feature-plans) are candidates to become skills with `context: fork`
  for the research phase and supporting-file directories. Weigh against the cost of churn —
  the existing files keep working, so this is opt-in, not forced.

**Mapping back to harness-evolution.md:** P1.1/P1.3 = Iteration 3 (mechanical enforcement),
P2.2 = Iteration 2 (self-review gate), P2.1 = Iteration 5 (verify step). The hooks are how
those iterations get *teeth* instead of staying advisory.

---

## 4. Open questions / experiments

1. **Lint-hook latency vs. value.** Does PostToolUse formatting on every edit slow the agent
   noticeably on multi-file refactors? Experiment: time an execute-plan run with and without
   P1.1. Mitigation if slow: `async` hooks, or batch-lint on Stop instead of per-edit.
2. **Exit-2 feedback loops.** When does feeding a lint error back help vs. send Claude into a
   fix-relint-fix spin? Cap retries; measure whether validate-spec first-pass rate rises
   (the Iteration 3 success metric).
3. **Stop-hook test enforcement granularity.** Whole-suite vs. just-touched tests? Whole suite
   is slow on DDP_backend; per-file is faster but misses integration breaks. Try running only
   the test module mapped to the edited app.
4. **Self-review: hook-blocking vs. subagent spawn.** Is a SubagentStop `decision: block`
   checklist enough, or does catching the three violation classes need a real diff-reviewing
   subagent? Run the planted-violation test (Iteration 2) both ways.
5. **Cross-repo routing robustness.** The `_repo.sh` path matching must survive new worktree
   naming, nested paths, and `prefect-proxy`. Validate against real worktree paths before
   trusting the branch guard in autonomous runs.
6. **Migrate commands to skills?** Pilot one (`plan-feature`) as a `context: fork` skill and
   compare research quality + token cost vs. the in-session command. Feeds Experiment 0.
