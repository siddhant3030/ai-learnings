# Skills in the Dalgo Harness

**Status:** Active
**Owner:** Engineering
**Last updated:** 2026-06-17

How to push the Dalgo engineering harness further using Claude Code **Agent Skills**.
Combines the current state of `.claude/skills/` with the latest Anthropic documentation.

The harness thesis holds here: *the discipline is in the scaffolding, not the code.*
A skill is scaffolding made cheap — its instructions cost ~100 tokens until something
actually needs them, then load on demand. That makes skills the natural home for the
"how we do X here" knowledge that would otherwise bloat CLAUDE.md or get re-explained
every session.

Sources (Anthropic, fetched 2026-06-17):
- Overview — https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- Authoring best practices — https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- Claude Code skills — https://code.claude.com/docs/en/skills

---

## 1. Feature reference

### What a skill is

A skill is a directory with a `SKILL.md` file. The frontmatter (`name`, `description`)
tells Claude *when* to use it; the markdown body tells Claude *how*. Supporting files
(reference docs, templates, scripts) live alongside and load only when referenced.

```
my-skill/
├── SKILL.md          # required — frontmatter + instructions
├── reference.md      # loaded on demand
├── examples.md       # loaded on demand
└── scripts/
    └── validate.sh   # executed, never loaded into context
```

In Claude Code, **custom commands have been merged into skills**. A file at
`.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create
`/deploy`. Existing `.claude/commands/` files keep working; skills are the recommended
form because they support supporting files, invocation control, and auto-discovery.

### The three loading levels (progressive disclosure)

| Level | When loaded | Cost | Content |
|-------|-------------|------|---------|
| 1 — Metadata | Always, at startup | ~100 tokens/skill | `name` + `description` |
| 2 — Instructions | When the skill triggers | Under ~5k tokens (keep <500 lines) | SKILL.md body |
| 3 — Resources | As needed, per file | Effectively unlimited | reference `.md`, templates, scripts |

This is the whole point: you can bundle huge reference material with no startup penalty.
The cost is paid only when a task reaches for it. (Overview → "How Skills work".)

**One caveat specific to Claude Code:** once a skill is invoked, its rendered body enters
the conversation as a message and **stays for the rest of the session** — Claude Code does
not re-read the file on later turns. So the Level-2 body is a *recurring* cost, not a
one-time one. Write the body as standing instructions, and keep it lean. (Claude Code →
"Skill content lifecycle".)

### The description field is a trigger, not a summary

This is the single highest-leverage thing to get right. At startup Claude sees only each
skill's `name` and `description`. It picks the right skill — out of potentially 100+ —
from the description alone. So write the description **for the model's "should I fire?"
decision**, not as a human-readable blurb.

Rules from the best-practices guide:

- **Write in third person.** The description is injected into the system prompt;
  first/second person ("I can help you…", "You can use this to…") causes discovery problems.
- **State both what it does AND when to use it.** Include concrete trigger words and
  contexts a user would actually say.
- **Be specific; include key terms.** Vague descriptions ("Helps with documents",
  "Processes data") don't match anything reliably.
- **Put the key use case first.** The combined `description` + `when_to_use` text is
  truncated at 1,536 characters in the listing; with many skills installed, descriptions
  for rarely-used skills get shortened and the tail is dropped.

Anthropic's canonical good example:

```yaml
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

Note the shape: `<verb-led what it does>. Use when <concrete triggers/keywords>.`

### Degrees of freedom — rigid vs flexible skills

Match specificity to the task's fragility. Anthropic's robot-on-a-path analogy:

- **Narrow bridge with cliffs (low freedom):** one safe way forward. Give exact commands,
  "do not modify". Example: a migration that must run in sequence.
- **Open field (high freedom):** many paths work. Give direction and trust the model.
  Example: a code review where context decides the approach.

Concretely:
- **High freedom** → prose steps ("Analyze structure, check edge cases, suggest improvements").
- **Medium freedom** → a template/pseudocode with parameters the model fills in.
- **Low freedom** → an exact script with no parameters ("Run exactly: `python migrate.py --verify --backup`").

In Dalgo terms: `plain-writing` and `design-review` are correctly *high-to-medium freedom*
(rules + a self-check, model adapts). `executing-feature-plans` is correctly *low freedom*
on the load-bearing parts (branch first, one failing test, two-PR order) — it has a
**Red Flags — STOP** list, which is the rigid-skill pattern done well.

### Scoping — one capability per skill

Each skill should cover one capability. Split when a skill starts serving two audiences or
two distinct triggers, because the description can only point at one "when". Signs you
should split: the description needs an "and also" clause for an unrelated trigger; the body
has two workflows that never run together; reference files cluster into disjoint groups.

Anthropic's counter-pattern (Pattern 2, "domain-specific organization") shows the
alternative to splitting: keep one skill but shard the *reference files* by domain
(`reference/finance.md`, `reference/sales.md`) so a sales question never loads finance
context. Split into separate skills when the *triggers* differ; shard reference files when
only the *content* differs.

### Progressive disclosure — keep SKILL.md lean

- **Cap the body at 500 lines.** Past that, split into reference files.
- **Reference files one level deep.** SKILL.md → reference.md is fine. SKILL.md →
  advanced.md → details.md is not: Claude may `head -100` a nested file and read it
  partially. All reference files should link directly from SKILL.md.
- **Reference files >100 lines get a table of contents** at the top, so a partial read
  still reveals the full scope.
- **Name files for their content:** `form-validation-rules.md`, not `doc2.md`.
- **Scripts over generated code** for deterministic steps: more reliable, token-free
  (only the output enters context). State whether Claude should *execute* ("Run
  `analyze.py`") or *read* it ("See `analyze.py` for the algorithm").

### How skills compose with commands and agents

Claude Code adds three composition levers (these are Claude-Code-specific, beyond the open
standard):

- **`disable-model-invocation: true`** — only the user can invoke (`/name`); Claude won't
  auto-fire it. Use for side-effecting actions (`/deploy`, `/commit`) and for command-style
  pipelines you want triggered explicitly. Also removes the description from context.
- **`user-invocable: false`** — only Claude can invoke; hidden from the `/` menu. Use for
  pure background knowledge that isn't a meaningful user action (e.g. "how the legacy auth
  system works").
- **`context: fork` + `agent: <type>`** — runs the skill's body as the *prompt* for a
  forked subagent (e.g. `agent: Explore` for read-only research). The skill content must
  be an actionable task, not just guidelines, or the subagent returns nothing useful.

The inverse direction: a **subagent can preload skills** via a `skills` field in its
definition — the full skill content is injected at startup (not just the description).

Other useful frontmatter: `allowed-tools` (pre-approve tools while active),
`disallowed-tools` (remove tools — e.g. block `AskUserQuestion` in a background loop),
`paths` (auto-trigger only when working on matching files), `model`/`effort` (override per
skill), and `!`cmd`` **dynamic context injection** (runs a shell command and inlines its
output before Claude sees the body — e.g. `!`git diff HEAD``).

### Personal vs project vs plugin

| Location | Path | Scope | Override order |
|----------|------|-------|----------------|
| Enterprise | managed settings | whole org | highest |
| Personal | `~/.claude/skills/<name>/` | all your projects | over project |
| Project | `.claude/skills/<name>/` | this repo only (commit it) | base |
| Plugin | `<plugin>/skills/<name>/` | where plugin enabled | `plugin:name` namespace, no clash |

For Dalgo: harness skills belong in the repo's `.claude/skills/` (committed, shared with
the team). One subtlety — `permissions.additionalDirectories` in settings.json grants file
access but **does not** load skills from sibling repos. Only `--add-dir`/`/add-dir` loads a
sibling's `.claude/skills/`. So DDP_backend's and webapp_v2's own skills won't auto-load
during a harness session unless the sibling is added with `--add-dir`. This is why
`executing-feature-plans` correctly tells the agent to read each repo's CLAUDE.md manually.

---

## 2. Current state in the Dalgo harness

Eight skill directories in `.claude/skills/`. What they do, and where they fall short of
the practices above.

### What's working well

- **`executing-feature-plans`** — the model skill. Sharp description with literal trigger
  phrases ("execute this plan", "implement this feature"), correct low-freedom rigidity on
  the load-bearing rules, a **Red Flags — STOP** list, and explicit deference to
  `superpowers:test-driven-development` for the generic parts (good composition — it doesn't
  re-teach TDD).
- **`plain-writing`** — excellent. Third-person-ish trigger-rich description, a reusable
  mini-block template, before/after examples, and a self-check. Correctly invoked *by name*
  from `plan-feature` (`Skill(skill="plain-writing")`) rather than path-pasted.
- **`design-review`** — clean SKILL.md + two reference files (`checklist.md`, `patterns.md`)
  referenced one level deep. Good progressive disclosure.
- **`documentation`** — strong information architecture; SKILL.md is a navigation hub over
  seven reference files (`workflow.md`, `style-*.md`). Textbook Pattern 1.
- **`linear-issue-from-plan`** — tight scope, clear field-mapping table, names its MCP tool
  with the fully-qualified ID (matches the "MCP tool references" best practice).

### Gaps

**G1 — Two "skills" have no SKILL.md and cannot autodiscover.**
`backend-architecture/` and `frontend-architecture/` contain only `landmarks.md` — no
`SKILL.md`, no frontmatter. They are **invisible to Level-1 discovery**. The only reason
they work today is that `plan-feature.md` hard-codes the path
(`.claude/skills/backend-architecture/landmarks.md`). Any other command or an
auto-triggered flow will never find them. They are reference *files masquerading as skills*.
The Iteration 1b note in `harness-evolution.md` ("skill files become thin pointers") is
half-done: the pointer (SKILL.md) is missing.

**G2 — `productivity` is mis-built and looks imported from another project.**
- `name: prd-generator` does not match its directory `productivity/` — confusing, and the
  body references `work/specs/outputs/`, the `ulid` npm package, a "customer plugin
  PostToolUse sync hook", and "mySecond". None of that exists in Dalgo. This is foreign
  cruft that will mislead the agent.
- It **nests two sub-skills** inside it (`productivity/grill-me/SKILL.md`,
  `productivity/tal-lens/SKILL.md`). Per the docs, a nested `.claude/skills/` is what
  triggers nested discovery — but these sit *inside another skill's directory*, not under a
  `.claude/skills/`. They are good skills (decent descriptions) trapped in the wrong place.
- Net: one capability (PRD writing) that overlaps `/product/write-spec`, plus two unrelated
  good skills buried as subdirectories.

**G3 — Weak/duplicated triggering across the writing skills.**
`plain-writing`, `productivity` (PRD), and the `/product/write-spec` command all touch
"write a spec / requirements doc". A user saying "write a feature spec" could match the
PRD skill or the spec command. Trigger overlap means non-deterministic selection.

**G4 — Inconsistent description quality.**
- `design-review` and `documentation` descriptions are good but lack the strongest literal
  trigger words a user would type ("is this UI usable?", "review this screen").
- `backend/frontend-architecture` have **no description at all** (G1).
- None of the project skills use `when_to_use` to separate the "what" from the trigger
  phrases, so all triggers compete for the 1,536-char budget inside `description`.

**G5 — Missing skills the harness clearly needs.**
The harness has commands and agents for design, debugging, PM, and review — but several
recurring, hard-coded chunks of knowledge have no skill home:
- **Backend architecture as a real skill** (org-scoping, `@has_permission`, multi-tenancy,
  Ninja patterns) — currently only `landmarks.md`, no how-to body.
- **Frontend architecture as a real skill** (component patterns, `toastSuccess/Error`,
  CSS-variable colors, auth store) — same problem.
- **Self-review / pre-PR gate** — Iteration 2 of the evolution plan wants this; a
  `reviewing-own-diff` skill (the rigid checklist: missing `@has_permission`, hardcoded
  hex, `console.log`, missing org filter) would be reusable by `execute-plan` and humans.
- **Test-credentials / local-stack** knowledge is inlined in `executing-feature-plans`;
  a `running-the-stack` skill would let `/verify` and any agent share it.

**G6 — Commands paste skill file paths instead of invoking the skill.**
`design-handoff.md` reads `.claude/skills/design-review/checklist.md` by hardcoded path
instead of invoking the `design-review` skill. This couples the command to the skill's
internal file layout — rename `checklist.md` and the command silently breaks. `plan-feature`
already does it right with `Skill(skill="plain-writing")`; the pattern should be uniform.

---

## 3. Recommendations

Priority: **P1** = do now (correctness/discovery bugs), **P2** = clear wins, **P3** = nice.

### P1 — Give backend/frontend-architecture real SKILL.md files (fixes G1)

They cannot be discovered without frontmatter. Add a SKILL.md to each that points at the
existing `landmarks.md` (and future `patterns.md`/`templates.md`), one level deep.

`backend-architecture/SKILL.md`:

```yaml
---
name: backend-architecture
description: Patterns and file landmarks for DDP_backend (Django + Django Ninja). Use when planning or implementing a backend change — adding an API endpoint, model, migration, permission, or org-scoped query — or when you need to find where something lives in DDP_backend.
---

# DDP_backend Architecture

Load this before writing or planning backend code.

- **Where things live:** see [landmarks.md](landmarks.md) — file paths + line ranges.
- **How to build a slice:** [patterns.md](patterns.md) — endpoint, service, schema, test.
- **Org-scoping & permissions:** every query filters by `request.orguser.org`; every
  endpoint is gated by `@has_permission([...])`. (Expand into a SECURITY reference.)
```

Do the same for `frontend-architecture` (component patterns, `toastSuccess/Error`,
CSS-variable colors, auth store, feature-folder layout). This is the missing half of
Iteration 1b — the thin pointer the plan already called for.

### P1 — Fix or remove `productivity` (fixes G2)

It's the messiest item and actively misleading. Two clean options:

1. **Promote the two good sub-skills, delete the PRD body.** Move
   `productivity/grill-me/` → `.claude/skills/grill-me/` and
   `productivity/tal-lens/` → `.claude/skills/tal-lens/` (they already have valid
   frontmatter). Delete the `prd-generator` body — its PRD job overlaps
   `/product/write-spec`. Drop the `productivity/` wrapper.
2. If a PRD skill is genuinely wanted, rewrite it Dalgo-native: `name: writing-prds`,
   strip the `ulid`/`mySecond`/`work/specs/outputs` cruft, point output at
   `prototypes/{name}/` or `features/{name}/`, and reconcile its trigger with
   `/product/write-spec` (see P2 below).

Recommended: option 1. It removes foreign cruft and surfaces two useful skills that are
currently undiscoverable because they're nested one level too deep.

### P1 — Invoke skills by name from commands, not by file path (fixes G6)

In `design-handoff.md`, replace the hardcoded reads of
`.claude/skills/design-review/checklist.md` with an invocation:

```
Skill(skill="design-review")
```

…and let the skill pull its own `checklist.md`/`patterns.md`. This decouples the command
from the skill's internal layout, matching how `plan-feature` invokes `plain-writing`.

### P2 — Rewrite descriptions for stronger triggering (fixes G4)

Use the shape `<what it does>. Use when <literal triggers>.` and move trigger phrases into
`when_to_use` so they don't eat the `description` budget.

`design-review` (add literal user phrasings):
```yaml
description: Combined UX-expert and NGO-user evaluation lens for any UI component, screenshot, or screen.
when_to_use: Use when reviewing UI for usability, accessibility, or fit for non-technical NGO users — e.g. "is this screen usable?", "review this component", "would a program manager understand this?", or after building any webapp_v2 UI.
```

`documentation` (add the commit-range and post-ship triggers):
```yaml
description: Generate, update, or review Dalgo user-facing Docusaurus docs.
when_to_use: Use when asked to "write docs for X", "update the docs for #142", refresh docs after a PR ships, or review a docs page for accuracy.
```

`linear-issue-from-plan` already has good literal triggers — leave it. Apply the same pass
to every skill: open with one verb-led sentence of "what", put the phrases a user actually
types into `when_to_use`.

### P2 — De-conflict the writing/spec triggers (fixes G3)

After P1 removes `prd-generator`, make the boundary explicit:
- `/product/write-spec` owns "spec / feature spec / what to build".
- `plain-writing` owns *how to write* any document — it's a style lens, not a doc type, so
  its description should say "writing rules / style for plans, specs, research" and avoid
  claiming the "write a spec" trigger.

If a `writing-prds` skill survives (P1 option 2), have its description explicitly defer:
"Use for a product-requirements doc when no `/product/write-spec` flow is running."

### P2 — Add a `reviewing-own-diff` skill (fills G5, enables Iteration 2)

A rigid (low-freedom) checklist skill the `execute-plan` flow and humans can both invoke
before opening a PR. This is the cheapest path to the Iteration-2 self-review gate.

```yaml
---
name: reviewing-own-diff
description: Pre-PR self-review checklist for Dalgo changes across DDP_backend and webapp_v2.
when_to_use: Use before opening a PR, when asked to "review my diff / my changes", or as the self-review step inside execute-plan.
allowed-tools: Bash(git diff *) Bash(git status *) Read Grep
disable-model-invocation: true
---
```

Body = the mechanical red-flag list from `harness-evolution.md` Iteration 2/3: missing
`@has_permission`, query not filtered by org, hardcoded hex color, raw `toast()` instead of
`toastSuccess/Error`, `console.log`, `any` type, import inside a function body. Run it as a
loop: scan diff → list findings → fix → re-scan. Consider `context: fork` + `agent: Explore`
so it reviews in an isolated read-only context and returns findings.

### P2 — Add a `running-the-stack` skill (fills G5)

Extract the local-stack / test-credentials knowledge currently inlined in
`executing-feature-plans` (dev server on port 3001, creds from
`.claude/test-credentials.local.json`, Playwright smoke flow) into its own skill so
`/verify`, `execute-plan`, and ad-hoc debugging all share one source of truth. Keep the
credential-handling warnings verbatim.

### P3 — Consider a Dalgo skills plugin for sibling-repo sharing

DDP_backend and webapp_v2 don't load harness skills automatically (only `--add-dir` loads a
sibling's skills, not `additionalDirectories`). If the team wants backend/frontend
architecture skills available *while working inside* those repos, package them as a
**plugin** (`<plugin>/skills/...`) the sibling repos enable. Plugin skills get a
`plugin:name` namespace and can't clash. This is the clean way to share one skill library
across three repos.

### P3 — Add `argument-hint` / `paths` where they help

- Commands-as-skills that take an argument (e.g. a future `/review-pr <#>`) should set
  `argument-hint: [pr-number]` for autocomplete.
- `frontend-architecture` could set `paths: ["**/webapp_v2/**"]` and
  `backend-architecture` `paths: ["**/DDP_backend/**"]` so they auto-trigger when the agent
  touches the right repo — turning them from manually-pointed-at files into
  context-aware skills.

---

## 4. Open questions / experiments

1. **Does P1 (real SKILL.md for the architecture skills) improve plan quality without the
   command hard-pointing at landmarks?** Test: run `/engineering/plan-feature` on an
   org-scoped endpoint *after* removing the hardcoded `landmarks.md` read from
   `plan-feature.md`. Does the planner discover and load `backend-architecture` from the
   description alone? Ties directly to the Iteration-1b test already in the evolution plan.

2. **Description budget pressure.** With superpowers, figma, and bundled skills all loaded,
   how close is this session to the 1,536-char-per-skill cap and the 1%-of-context listing
   budget? Run `/doctor` to see how many descriptions are being shortened or dropped — if
   Dalgo's own skills are getting truncated, that's a silent discovery failure. Consider
   `skillOverrides: "name-only"` for rarely-used third-party skills to free budget.

3. **`context: fork` for review and research.** Would `reviewing-own-diff` and a future
   research skill be better as forked subagents (isolated context, read-only tools) than
   inline? Measure token cost and catch-rate vs the inline version.

4. **Rigid vs flexible calibration.** `executing-feature-plans` is deliberately rigid. Is
   it *too* rigid for small features (one-file backend tweak)? Consider a lighter-weight
   sibling skill, or a branch inside it, for sub-milestone changes — without weakening the
   Red Flags list.

5. **Build evals before more skills.** The best-practices guide says write three evaluation
   scenarios *before* documenting a skill. Dalgo has none. Pick `reviewing-own-diff`: plant
   the three deliberate violations from Iteration 2 (missing `@has_permission`, hardcoded
   hex, `console.log`) as a fixed eval, and use it as the regression test for every skill
   change. This is the missing feedback loop for skill quality itself.

6. **Personal vs project boundary.** `grill-me` and `tal-lens` are arguably *personal*
   (individual working style) rather than *project* skills. Decide whether they belong in
   the committed repo at all, or in each engineer's `~/.claude/skills/`.
