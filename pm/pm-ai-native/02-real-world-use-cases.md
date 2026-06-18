## 2. Real-World PM Use Cases

These are documented, sourced case studies from practicing PMs.

---

### 2.1 Cat Wu — Head of Product, Claude Code (Anthropic)

**Source:** [Product management on the AI exponential, Anthropic blog](https://claude.com/blog/product-management-on-the-ai-exponential)

**Workflow:**
- Uses three tools in a division-of-labor model: Claude.ai for strategic thinking, Claude Code for building Streamlit apps and evals, Cowork for administrative tasks (todos, slide decks, Slack history search)
- "Not a single line of code written by hand" across projects that took hundreds of hours of prompting
- Side-quest model: self-directed afternoon prototypes outside the official roadmap. This is how Claude Code Desktop, AskUserQuestion tool, and todo lists emerged as features

**Key insight:** Claude Code, for a PM, is not a coding tool — it is a prototyping and validation tool. The distinction matters because prototypes built by PMs can anchor the entire product direction.

---

### 2.2 Bihan Jiang — Director of Product, Decagon

**Source:** [Anthropic PM blog](https://claude.com/blog/product-management-on-the-ai-exponential)

**Workflow:**
- Start in Claude Cowork, pulling context from Slack, codebase, and docs
- Move into Claude Code to have something demo-able in a couple of hours

**Key insight:** "Claude has raised the ceiling on what good product teams can build, and dramatically shortened the distance between idea and prototype. Getting something tangible in front of customers used to take weeks of building. Now I'll start in Claude Cowork…then move into Claude Code to have something demo-able in a couple of hours."

**Why it matters:** The PM is no longer blocked by engineering availability to validate ideas. Customer feedback cycles are compressing from weeks to hours.

---

### 2.3 Kai Xin Tai — Senior PM, Datadog (Bits AI SRE Agent)

**Source:** [Anthropic PM blog](https://claude.com/blog/product-management-on-the-ai-exponential)

**Workflow:**
- Study model strengths and failure modes through offline evaluation on real production incidents
- Design tight feedback loops using evals (not traditional user research) as the primary validation mechanism
- Refine UX based on where the agent struggles, converting agent failures into product improvements

**Key insight:** "Being a PM in an AI-native world is both creative and academic…a PM's craft has shifted from defining certainty upfront to accelerating discovery."

**Why it matters:** For AI products, evaluations (evals) replace user stories as the primary PM artifact. Understanding what the model gets wrong is the core product work.

---

### 2.4 Dennis Yang — PM, Chime

**Source:** [Builder.io — Claude Code for Product Managers](https://www.builder.io/blog/claude-code-for-product-managers)

**Workflow:**
- Writes a PRD in markdown
- Opens a terminal, types `claude`
- Working prototype ready 20 minutes later

**Key insight:** The PRD-to-prototype pipeline collapses the concept-to-validation cycle from weeks to an afternoon. The prototype is not polished but is functional enough to get real user reactions.

---

### 2.5 Teresa Torres — Founder, Product Talk (Continuous Discovery Habits)

**Source:** [Lenny's Newsletter — Claude Code for product managers](https://www.lennysnewsletter.com/p/claude-code-for-product-managers) and [ChatPRD blog](https://www.chatprd.ai/how-i-ai/teresa-torres-claude-code-obsdian-task-management)

**Workflow — Task management system:**
- Migrated from Trello to terminal-based markdown task system
- Each task is stored as an individual markdown file with YAML frontmatter (due dates, tags)
- Custom `/today` slash command generates daily markdown file with tasks due today, overdue items, in-progress projects, research digest link
- Natural language task creation: "new task, send thank you to Claire, do today" → Claude auto-creates a formatted file

**Workflow — Research automation:**
- Two Python cron scripts: one searches arXiv daily and Google Scholar weekly on predefined keywords; one processes downloaded PDFs nightly and generates summaries focused on methodology and effect size
- Results appear in a daily "Research Digest" file

**Workflow — "Lazy prompting" context library:**
- Modular Obsidian vault with separate folders for business and personal topics
- Index files acting as intelligent routing maps
- Main `claude.md` file with routing instructions
- Simple prompts like "Claude blog post review, gimme feedback" trigger automatic loading of only relevant context

**Key insight:** The power of Claude Code for PMs is not in individual prompts but in building systems — modular context files, cron jobs, slash commands — that run autonomously and improve the quality of every subsequent interaction.

---

### 2.6 Ondrej Machart — PM (13 Products Built with Claude Code)

**Source:** [13 Claude Code Projects That Changed My Product Manager Role](https://medium.com/@ondrej.machart/13-claude-code-projects-that-changed-my-product-manager-role-over-the-last-6-months-7057b9045d51)

**Notable projects:**

| Project | Purpose | Business Impact |
|---------|---------|-----------------|
| Product Portfolio Coach | Interactive canvas mapping 30+ company products | Directly informed C-level brand consolidation decision |
| Vibe-Coding Workshops | Internal training for PM colleagues | Shifted his role "from operational delivery toward tackling complex challenges through rapid prototyping" |
| TinyStakeholders.com | Converted Lenny's 300 podcast transcripts to parenting wisdom | 5,000+ readers |
| Infographics Tool Prototype | Sports infographics for editorial teams | Evolved toward production tool with senior engineer oversight |

**Key mindset shifts:**
1. Start by solving your own problem — personal projects give immediate feedback
2. Your project can die and still win — learning and influence are valid outcomes
3. Respect the challenge — prototype capability does not equal production readiness
4. Embrace non-determinism — stop fighting AI randomness; it excels for creative exploration

**Why it matters:** This case study shows the PM role transforming from "influencing through persuasion" to "influencing through building."

---

### 2.7 Every.to PM (Custom Analytics System)

**Source:** [Every.to — Claude Code for Product Managers](https://every.to/p/claude-code-for-product-managers)

**Workflow — The /pulse command:**
- Daily analytics briefing surface: active users, chats/messages/drafts created, response times, conversation quality ratings
- Claude grades conversations 1-5, flags anomalies with visual indicators, explains findings in plain English
- Results integrated into the same Claude thread for immediate problem-solving

**Workflow — The /prioritize command:**
- Daily automated sweep of GitHub project columns (later, next, now, in progress, done)
- Checks for stale tickets and confirms weekly work allocation
- Triages bug reports and feature requests into backlog

**Key insight:** The GitHub project becomes "the product's working memory" — a structured artifact that Claude can query, update, and reason about autonomously.

---

### 2.8 Felix Rieseberg — PM, Electron (Microsoft)

**Source:** [Hacker News discussion on Claude Cowork](https://eu.36kr.com/en/p/3639916972339968)

**Key claim:** Almost all of Cowork's core code was autonomously generated by Claude in 1.5 weeks.

**Key insight:** Reusability of skills matters more than raw model performance. The shift from one-off prompts to reusable, versioned skills is where the real productivity multiplier lives.

---
