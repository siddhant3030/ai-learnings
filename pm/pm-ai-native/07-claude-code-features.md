## 7. Claude Code Features PMs Should Learn

Ordered by learning priority for non-engineering users.

---

### 7.1 CLAUDE.md — Priority: Highest

**What it does:** A markdown file in your project directory that Claude Code reads at the start of every session. It is the persistent instruction set that shapes all Claude behavior in that workspace.

**Why a PM should care:** Without CLAUDE.md, every Claude session starts from zero. With it, every session starts with full product context, terminology, and constraints pre-loaded. The quality of Claude's output is directly proportional to the quality of your CLAUDE.md.

**PM-specific use cases:**
- Define your product, target user, north star metric, and key terminology
- Set output format standards (e.g., "always use our 4-section PRD template")
- Add anti-goals and constraints (e.g., "never suggest features that require new backend infrastructure")

**Template:**
```markdown
# Product Context

## What We Build
[One paragraph description of your product and its users]

## Our Users
- Primary persona: [description]
- Secondary persona: [description]
- Key user jobs: [list the 3-5 core jobs users hire our product for]

## Our Business
- North star metric: [metric]
- Current stage: [early/growth/scale]
- Key competitors: [list]

## PM Standards
- PRD template: [link or paste template]
- Voice: [e.g., "clear, direct, avoids jargon"]
- Prioritization framework: [e.g., RICE]

## Constraints
- Never suggest: [things Claude should avoid]
- Always flag: [things you want flagged]
```

**Difficulty:** Low — it's just writing a markdown file.

**Learning priority:** Do this first. Everything else builds on it.

---

### 7.2 Skills — Priority: High

**What it does:** Reusable instruction modules stored as `SKILL.md` files in `.claude/skills/`. Claude auto-invokes relevant skills based on context, or you invoke them explicitly with `/skill-name`.

**Why a PM should care:** Skills allow you to encode your best PM practices once and reuse them forever. A skill for user research synthesis will apply your specific framework every time, without re-typing the prompt.

**PM-specific use cases:**
- Research synthesis skill (applies your JTBD framework to every transcript batch)
- PRD generation skill (uses your team's specific template)
- Competitive brief skill (applies your competitive framework)

**Install existing PM skill collections:**
```bash
npx skills add RefoundAI/lenny-skills  # 86 skills from Lenny's podcast
```

Or install the Anthropic official PM plugin from claude.com/plugins/product-management.

**Difficulty:** Low to medium. Installing published skills is easy; writing custom ones takes 30-60 minutes per skill.

**Learning priority:** Second. Start with published skill collections, then customize.

---

### 7.3 Custom Slash Commands — Priority: High

**What it does:** Custom commands you define in `.claude/commands/` that execute multi-step workflows with a single `/command` invocation.

**Why a PM should care:** Slash commands eliminate repetitive setup. Instead of typing the same 200-word context prompt every Monday, `/weekly-review` does it automatically.

**PM-specific examples:**

```
/competitive-brief [competitor name]
  → Researches competitor, generates pricing + feature table, identifies gaps

/synthesize [folder path]
  → Reads all transcripts in folder, extracts themes, outputs synthesis doc

/prioritize
  → Reads current backlog, applies RICE scoring, generates ranked list

/pulse
  → Queries analytics, generates narrative metric summary with anomaly flags

/stakeholder-update [audience]
  → Reads recent notes and decisions, generates tailored update
```

**Difficulty:** Low. Commands are markdown files with natural-language instructions.

**Learning priority:** Third. Pick your 3 most repetitive weekly tasks and automate them first.

---

### 7.4 MCP (Model Context Protocol) — Priority: High

**What it does:** A protocol that lets Claude Code connect to external systems. With MCP servers, Claude can read from and write to Linear, Jira, Notion, Slack, GitHub, PostHog, Amplitude, and dozens more tools.

**Why a PM should care:** MCP turns Claude Code from a local file processor into a connected PM command center. You can ask Claude to read a ticket, understand requirements, generate related PRD content, and create follow-up tickets — all in one workflow.

**PM-specific MCP integrations:**
- **Linear/Jira MCP** — generate and update tickets from PRDs; query ticket status
- **GitHub MCP** — generate release notes from commits; analyze PR patterns
- **Slack MCP** — search conversation history for product decisions; extract action items
- **PostHog/Amplitude MCP** — pull live metrics into analysis workflows
- **Notion MCP** — read/write product wiki and documentation
- **Crustdata MCP** — competitive intelligence data

**Setup:** Most MCP servers require adding a JSON config block to Claude Code settings. Takes 5-10 minutes per integration.

**Difficulty:** Low to medium. Basic MCP setup requires no coding; advanced custom MCP servers require some engineering.

**Learning priority:** Fourth. Start with your team's primary project management tool (Linear or Jira).

---

### 7.5 Hooks — Priority: Medium

**What it does:** Shell commands that execute automatically at defined points in Claude's lifecycle (before/after tool use, at session start/end, on each prompt submission).

**Why a PM should care:** Hooks make Claude behavior consistent and automatic. Unlike prompt instructions that Claude might interpret loosely, hooks fire deterministically every time.

**PM-specific examples:**
- **SessionStart hook:** Automatically load the latest competitive intel digest into context
- **PostToolUse hook:** After Claude writes a PRD, automatically send a Slack notification
- **Stop hook:** When Claude finishes a research synthesis, automatically upload to Notion

**Example hook configuration (`.claude/settings.json`):**
```json
{
  "hooks": {
    "SessionStart": {
      "command": "cat /research/competitive-digest-latest.md"
    },
    "Stop": {
      "command": "notify 'Claude finished: {{task_summary}}'"
    }
  }
}
```

**Difficulty:** Medium. Requires editing JSON configuration and basic shell scripting.

**Learning priority:** Fifth. Add hooks after you have stable workflows worth automating.

---

### 7.6 Subagents — Priority: Medium

**What it does:** Isolated Claude instances with their own context windows, specialized prompts, and tool permissions. The main Claude agent can spawn and coordinate multiple subagents.

**Why a PM should care:** Subagents enable parallel research, multi-perspective analysis, and workflows that exceed a single context window.

**PM-specific use cases:**
- Spawn 10 research subagents to process 10 interview transcripts simultaneously (10x speed)
- Run a subagent that red-teams your strategy document while another refines it
- Use a subagent specialized in financial analysis to evaluate pricing scenarios

**Example:** Processing 100 customer interviews through a 6-stage pipeline — extract opportunities from each transcript, cluster synonyms, score by frequency and satisfaction, generate solutions, build prototypes, re-run underperforming stages — in 12.5 minutes with 113 agents. Source: [Product Compass — Dynamic Workflows for PMs](https://www.productcompass.pm/p/claude-code-dynamic-workflows)

**Difficulty:** Medium to high. Requires understanding of how to structure parallel workflows.

**Learning priority:** Sixth. Use after you have mastered skills and slash commands.

---

### 7.7 Plan Mode — Priority: Medium

**What it does:** A safe exploration mode where Claude proposes what it would do without executing anything. Claude can inspect your files, think through an approach, and show you the plan before taking any action.

**Why a PM should care:** For PMs exploring unfamiliar codebases or asking Claude to make significant changes to product documents, Plan Mode provides a review checkpoint.

**PM-specific use cases:**
- "Show me what you would do to refactor this entire PRD structure before making changes"
- "Propose a reorganization of my research folder before moving any files"

**Difficulty:** Very low. Just add "plan mode" or "don't make any changes, just show me what you would do" to your prompt.

**Learning priority:** Use this whenever making major changes.

---

### 7.8 Context Compaction (/compact) — Priority: Medium

**What it does:** Condenses a long conversation history into a summary that preserves key decisions and context while freeing up context window space.

**Why a PM should care:** Long research or writing sessions can fill up Claude's context window, degrading output quality. `/compact` lets you continue long sessions without starting over.

**When to use it:** After completing a major research phase; before switching from discovery work to PRD writing in the same session.

**Difficulty:** Very low. Just type `/compact`.

---

### 7.9 Memory Systems — Priority: Medium

**What it does:** Various approaches to persist information across sessions, from CLAUDE.md (manual, permanent) to vector databases (automated, scalable).

**For most PMs:** Start with CLAUDE.md and a modular markdown knowledge base (separate files for product context, user research summaries, competitive intelligence, strategic decisions). Load only what's relevant to the current task.

**Teresa Torres' approach:** Separate Obsidian vault folders for different topics, with an index file routing Claude to the right context based on task type.

**For advanced PM workflows:** claude-mem (a plugin that automatically records and compresses session history into SQLite) or a full vector database for searching across large knowledge bases.

**Difficulty:** Low (CLAUDE.md) to High (vector databases).

**Learning priority:** Build the CLAUDE.md first, then evolve to a modular knowledge base as your library grows.

---

### 7.10 Dynamic Workflows — Priority: Medium-High

**What it does:** JavaScript programs that coordinate multiple AI agents without model tokens for orchestration. Code handles routing, filtering, and decision-making between agent tasks; models do the actual thinking.

**Why a PM should care:** Dynamic workflows solve three persistent problems in agentic work: agents reviewing their own work too charitably, forgetting requirements mid-session, and stopping prematurely.

**PM-specific patterns:**
- **Fan-out-and-synthesize:** Parallel analysis of all customer transcripts merged into themes
- **Generate-and-filter:** Naming, positioning, or ideation tournaments
- **Adversarial verification:** Fact-checking PRDs or red-teaming strategies
- **Loop-until-done:** Processing backlogs when volume is unknown

Source: [Product Compass — Dynamic Workflows for PMs](https://www.productcompass.pm/p/claude-code-dynamic-workflows)

**Difficulty:** High. Requires writing JavaScript. Best learned after you have a team with engineering support.

**Learning priority:** Advanced. Focus on this after mastering the first 7 features.

---
