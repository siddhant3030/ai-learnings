## 8. PM Operating Systems Built on Claude

### 8.1 Teresa Torres' Research and Task OS

**Architecture:**
```
workspace/
├── CLAUDE.md                    # Master routing instructions
├── tasks/                       # Individual task files (YAML frontmatter)
│   ├── send-thank-you-claire.md
│   └── review-prd-v2.md
├── research/
│   ├── interviews/              # Raw interview transcripts
│   ├── papers/                  # Downloaded academic papers
│   └── digests/                 # Auto-generated daily digest files
├── writing/                     # Blog posts, thought leadership
└── .claude/
    ├── commands/
    │   ├── today.md             # /today: generates daily task file
    │   └── research.md          # /research: queries research library
    └── settings.json
```

**Workflow:**
- Cron job 1: searches arXiv daily + Google Scholar weekly on keywords
- Cron job 2: generates research digest nightly from downloaded PDFs
- `/today` slash command: assembles daily task file from all task files
- Natural language task creation ("new task, review PRD v2, due Thursday")

**Benefits:** Fully automated daily task queue + research pipeline. Teresa's estimate: saves 2-3 hours/week on research processing alone.

Source: [ChatPRD blog on Teresa Torres](https://www.chatprd.ai/how-i-ai/teresa-torres-claude-code-obsdian-task-management)

---

### 8.2 Every.to PM OS

**Architecture:**
```
workspace/
├── CLAUDE.md
├── github-project/              # Acts as product working memory
├── analytics/
│   └── metrics-config.json     # /pulse command configuration
├── research/
│   └── interviews/
└── .claude/
    ├── commands/
    │   ├── pulse.md             # /pulse: daily analytics briefing
    │   ├── prioritize.md        # /prioritize: triage backlog daily
    │   └── roadmap.md           # /roadmap: weekly roadmap update
    └── settings.json            # MCP: GitHub, analytics
```

**Workflow:**
- `/pulse` runs daily: queries analytics, grades conversations 1-5, flags anomalies, integrates findings into same thread for immediate problem-solving
- `/prioritize` runs daily: sweeps GitHub project columns, identifies stale tickets, confirms weekly allocation
- GitHub project = persistent product memory that Claude reads, updates, and reasons about

Source: [Every.to — Claude Code for Product Managers](https://every.to/p/claude-code-for-product-managers)

---

### 8.3 Dean Peters' Product Manager Skills OS

**Architecture:**
```
pm-workspace/
├── CLAUDE.md
├── skills/
│   ├── component/               # 21 template skills (user-story, PRD, persona)
│   ├── interactive/             # 22 guided discovery flows
│   └── workflow/                # 6 end-to-end processes
├── context/
│   ├── company.md               # Company background
│   ├── product.md               # Product description
│   ├── personas.md              # User personas
│   ├── competitors.md           # Competitor profiles
│   └── strategy.md              # Current strategy
└── .claude/
    └── commands/                # Slash commands per skill
```

**Key feature:** "Pedagogy-first" design — skills teach reasoning alongside execution. An agent running the JTBD skill doesn't just output job statements; it explains the reasoning framework it used. This is intentional for building PM capability, not just automating output.

**Philosophy:** "Skills provide expertise; commands provide momentum."

Source: [GitHub — deanpeters/Product-Manager-Skills](https://github.com/deanpeters/Product-Manager-Skills)

---

### 8.4 AI PM OS (Third-Party)

A comprehensive commercial PM workspace (prodmgmt.world) including:
- 200+ PM skills activated by context
- 150+ frameworks organized by product stage
- 11 guided end-to-end workflows
- 7 PRD templates
- 100 customer interview questions
- Context library from 260 Lenny posts

**Key insight:** The entire system is "pre-tuned" — CLAUDE.md and the 5-file context system (company, product, team, goals, constraints) arrive pre-configured. PMs describe their situation once; everything after is calibrated to it.

Source: [prodmgmt.world — Claude Code for PMs](https://www.prodmgmt.world/resources/claude-code)

---

### 8.5 Competitive Intelligence System

**Architecture:**
```
competitive-intel/
├── CLAUDE.md
├── competitors/
│   ├── competitor-a/
│   │   ├── snapshot-2026-06.md  # Monthly snapshots
│   │   ├── pricing.md
│   │   └── features.md
│   └── competitor-b/
├── digests/
│   └── weekly-digest-YYYY-MM-DD.md
└── .claude/
    ├── commands/
    │   └── competitive-monitor.md  # Weekly monitoring run
    └── settings.json               # Crustdata MCP, web browsing
```

**Workflow:**
1. Weekly: `claude /competitive-monitor` fetches competitor pages, diffs against last snapshot, writes structured digest
2. Watcher API alerts trigger immediate notification for: headcount changes, funding announcements, C-suite transitions, new job posting surges

**Benefits:** Continuous competitive monitoring for single-digit dollars per cycle vs. $20K-40K/year for enterprise CI platforms.

Source: [Crustdata blog](https://crustdata.com/blog/how-to-automate-competitive-intelligence-with-claude)

---
