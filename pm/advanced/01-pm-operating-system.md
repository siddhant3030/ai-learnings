# Building a PM Operating System with Claude Code

A step-by-step workshop guide for product managers who want to stop one-off prompting and start working with a structured AI workspace. Follow this in an afternoon and you will have a fully operational PM OS by end of day.

---

## 1. What a PM OS Is — and Why It Matters

### The core insight: Claude is only as good as the context you give it

Ask Claude "write me a PRD for a notification feature" with no context and you get a generic template. Ask the same question with a CLAUDE.md that defines your product, your users, your terminology, and your output standards, and you get a draft that sounds like it came from your team.

Context is the multiplier. Every PM who has moved from one-off prompting to a structured workspace reports the same thing: the outputs stop sounding like ChatGPT and start sounding like you.

### Why a structured workspace beats one-off prompting

| One-off prompting | PM OS |
|-------------------|-------|
| Re-explain context every session | Context loads automatically |
| Generic outputs in generic formats | Outputs match your product and tone |
| Lose good prompts after closing tab | Commands saved, versioned, shareable |
| Can't hand off to teammates | Team shares the same workspace |
| Claude forgets decisions you made | Decisions file preserves reasoning |

### What you will have at the end of this guide

- A `pm-workspace/` folder with a logical structure for everything you produce
- A filled-in `CLAUDE.md` that loads your product context automatically
- Five context files covering company, product, users, competitors, and key decisions
- Ten slash commands for the most common PM workflows
- An MCP setup connecting Claude to Linear, Notion, and Google Drive
- A day-one checklist and a week-one habit plan

---

## 2. Folder Structure

Here is the recommended structure. Create this once. Every folder has a job.

```
pm-workspace/
├── CLAUDE.md                    ← loads automatically; the brain of the OS
├── context/
│   ├── company.md               ← stage, mission, business model, team
│   ├── product.md               ← what it does, architecture, terminology
│   ├── users.md                 ← personas with pain points and real quotes
│   ├── competitors.md           ← competitive landscape, positioning
│   └── decisions.md             ← log of key decisions and why you made them
├── research/
│   ├── interviews/              ← raw notes and transcripts
│   ├── surveys/                 ← survey exports and syntheses
│   └── market/                  ← industry reports, analyst notes
├── prds/
│   ├── active/                  ← PRDs in progress
│   └── shipped/                 ← PRDs for shipped features (archive)
├── strategy/
│   ├── roadmap.md               ← current roadmap state
│   ├── okrs.md                  ← current OKRs
│   └── bets.md                  ← strategic bets under consideration
├── templates/
│   ├── prd-template.md          ← blank PRD structure
│   ├── brief-template.md        ← feature brief skeleton
│   └── interview-guide.md       ← standard interview guide skeleton
└── .claude/
    └── commands/
        ├── prd.md
        ├── brief.md
        ├── research.md
        ├── compete.md
        ├── prioritize.md
        ├── interview.md
        ├── retro.md
        ├── update.md
        ├── spec.md
        └── decision.md
```

**What goes where:**

- `CLAUDE.md` — Never put ephemeral content here. This is for stable, durable context: who you are, what the product is, who your users are, and how you want Claude to behave. Think of it as your project's constitution.
- `context/` — Stable reference files. Update these when something meaningful changes (new persona, acquired competitor, major pivot). Not every sprint.
- `research/` — Raw and synthesized research. Store originals and summaries side by side.
- `prds/active/` — Work in progress. Move to `shipped/` when the feature is live, so your active folder stays uncluttered.
- `strategy/` — Living documents. These change quarterly or when direction shifts.
- `templates/` — Blank skeletons you reference in commands. Claude can fill them in.
- `.claude/commands/` — Your slash commands. Every `.md` file in here becomes a `/command`.

---

## 3. CLAUDE.md — The Complete Template

Copy this entire block, paste it into `pm-workspace/CLAUDE.md`, and replace the bracketed sections with your actual information. The comments (in italics after `#`) explain what each section does.

```markdown
# PM Workspace — Project Memory

## Who I Am
I am a product manager at [Company Name].
My focus area: [e.g., "growth and activation for B2B SaaS", "core platform for NGO data teams"].
My seniority: [e.g., "Senior PM", "Group PM owning the data pipeline squad"].

## Company Context
**Company:** [Name]
**Stage:** [e.g., Series B, bootstrapped, NGO/nonprofit]
**Mission:** [One sentence — what problem the company exists to solve]
**Business model:** [e.g., SaaS subscription, usage-based, grant-funded nonprofit]
**Team size:** [e.g., 40 people, 4-person engineering squad]
**Key stakeholders I work with:** [e.g., "CTO (Asha), Head of Design (Ben), 3 engineers"]

## Product Context
**Product name:** [Name]
**What it does:** [2-3 sentences. What problem it solves, for whom, how]
**Current stage:** [e.g., "In production, 500 DAU, growing 15% MoM"]
**Tech stack (high level):** [e.g., "React frontend, Django API, PostgreSQL, deployed on AWS"]
**Key integrations:** [e.g., "Airbyte for data ingestion, dbt for transformation, Superset for dashboards"]

## Terminology Rules
ALWAYS use these terms. Never use the alternatives.
- Use "Pipeline" not "Workflow" or "Job"
- Use "Data Source" not "Connector" or "Integration"  
- Use "Organisation" (British spelling) not "Organization"
- Use "Partner NGO" not "Customer" or "Client"
- Use "Run" not "Execution" or "Job run"
Add your own: [term] not [alternative]

## User Personas
Keep these brief but specific. Full personas live in context/users.md.

**Primary user — [Persona Name]:**
Role: [e.g., "Data Coordinator at a mid-size NGO (50-500 staff)"]
Goal: [e.g., "Automate their monthly reporting so they stop rebuilding Excel sheets"]
Pain point: [e.g., "Spends 2 days/month manually collating data from 4 systems"]
Tech comfort: [e.g., "Comfortable with Excel, nervous about anything that looks like code"]
Representative quote: "[e.g., 'I just want to click a button and have the report be right.']"

**Secondary user — [Persona Name]:**
Role: [e.g., "IT Admin or CTO at the NGO"]
Goal: [e.g., "Set up the platform once and not get support calls"]
Pain point: [e.g., "No engineering resources to maintain custom scripts"]
Tech comfort: [e.g., "Technically literate, but not a full-time developer"]

## Writing Style
ALWAYS follow these rules in every output:
- Active voice. "Users click the button" not "The button is clicked by users."
- Short sentences. Maximum 20 words per sentence in user-facing copy.
- No jargon in user-facing copy. Explain technical terms if you must use them.
- Headers use sentence case: "Feature requirements" not "Feature Requirements"
- Bullet points for lists of 3+ items. Never comma-separated in running prose.
- Numbers: spell out one through nine, use numerals for 10 and above.
- Oxford comma always.

## Output Formats
When I ask for a PRD: use the structure in templates/prd-template.md
When I ask for a brief: use the structure in templates/brief-template.md
When I ask for a status update: lead with decisions made, then blockers, then next steps
When I ask for research synthesis: use "Insight → Evidence → Implication" format
When I ask for meeting notes: lead with decisions, then action items with owners and dates

## What Claude Should Always Do
- Ask one clarifying question before generating any major artifact (PRD, strategy doc, competitive analysis)
- Reference the relevant persona from context/users.md when writing user-facing copy
- Flag assumptions explicitly: "I'm assuming X — correct me if wrong"
- Offer a shorter version when output exceeds 500 words
- When uncertain, say so and offer 2-3 options instead of picking one

## What Claude Should Never Do
- Never invent feature requirements not in the brief
- Never use the word "leverage" (say "use" instead)
- Never use the phrase "in today's fast-paced world" or any similar filler opener
- Never generate a decision without showing at least two alternatives
- Never write a PRD without acceptance criteria
- Never skip the "open questions" section in any strategy document

## Key Decisions Log
(Keep the most recent 5 here. Full log in context/decisions.md)
- [Date] — [Decision]: [Brief rationale]
- [Date] — [Decision]: [Brief rationale]

## Sub-Agent Reviewers
When I ask you to "review as [role]", adopt that persona:
- **Engineer**: Focus on technical feasibility, scope, and edge cases
- **Designer**: Focus on UX clarity, user flow, and interaction states
- **Executive**: Focus on business impact, resourcing, and strategic fit
- **Skeptic**: Find the holes. What's underspecified? What could go wrong?
- **User (primary persona)**: Would [Persona Name] actually understand and use this?
- **Data Analyst**: What's measurable? Are the success metrics rigorous?

## MCP Connections
(Document which tools are connected so context is clear)
- Linear: connected — can read/write issues and projects
- Notion: connected — can read/write workspace docs
- Google Drive: connected — can read shared documents
```

**What makes this CLAUDE.md good:**
- It's under 120 lines. Claude reads it every session — bloat slows everything down.
- Every section gives behavioral instructions, not just facts. "ALWAYS use Pipeline not Workflow" is actionable. "We have pipelines" is not.
- The "never do" section prevents the most common bad outputs before they happen.
- Sub-agent reviewers give you a built-in peer review system without extra setup.

---

## 4. Context Files — How to Write Them

Context files are the long-form reference that CLAUDE.md points to. Write them once, update them quarterly or after major changes.

### context/company.md

```markdown
# Company Context

## Overview
[Company Name] was founded in [year] by [founders]. We are [stage].
Core mission: [one sentence].
We are [profitable/pre-revenue/grant-funded/etc.].

## Business Model
Revenue comes from [describe]. 
Primary growth lever: [e.g., "word-of-mouth among NGO networks"].
Unit economics: [CAC, LTV, or equivalent if you know them — leave blank if sensitive].

## Team
Total headcount: [N]
Engineering: [N people, what they own]
Product: [N people, what they own]
Design: [N people or "no dedicated designer"]
Key external partners: [e.g., "Prefect for orchestration infrastructure, Airbyte OSS"]

## What We're Good At
- [Strength 1 — specific, not generic]
- [Strength 2]
- [Strength 3]

## What We're Not Good At (Yet)
- [Honest gap 1]
- [Honest gap 2]

## Strategic Context for This Year
We are trying to [strategic goal 1] and [strategic goal 2].
The biggest risk to our strategy is [honest assessment].
```

**What makes this good:** It's honest about weaknesses. Claude needs to know your constraints to give useful advice.

### context/product.md

```markdown
# Product Context

## What It Does
[Product name] lets [user type] [accomplish goal] without [the painful alternative].

The core loop: [describe the main user journey in 3-5 steps]
1. User [step 1]
2. System [step 2]
3. User [step 3]

## Current Feature Set
**Core features (shipped, stable):**
- [Feature]: [What it does in one sentence]
- [Feature]: [What it does]

**In active development:**
- [Feature]: [Status and expected ship date]

**Known gaps:**
- [Gap users complain about most]
- [Gap that blocks enterprise deals]

## Architecture (PM-level understanding)
Frontend: [e.g., "Next.js web app, mobile-responsive"]
Backend: [e.g., "Django REST API"]
Data layer: [e.g., "PostgreSQL + Redis for caching"]
External services: [e.g., "Airbyte OSS for connectors, Prefect for orchestration"]
Deployment: [e.g., "AWS, customer-managed or cloud-hosted"]

## Pricing
[Describe tiers or "not yet monetized" — affects feature prioritization decisions]

## Key Metrics We Track
- Primary: [e.g., "Weekly active organisations"]  
- Engagement: [e.g., "Pipelines run per week per org"]
- Health: [e.g., "Pipeline success rate"]
- Business: [e.g., "Renewal rate"]
```

### context/users.md

```markdown
# User Personas

## How to Use This File
Every PRD, user story, and research synthesis should reference at least one persona.
When writing copy or designing flows, ask: "Would [persona name] understand this?"

---

## Persona 1: [Name] — [Role Title]

**Demographics:**
- Works at: [org type and size]
- Team size: [their team]
- Location/context: [e.g., "India, Bangladesh, East Africa — low-bandwidth environments common"]

**Goals:**
- Primary: [what success looks like for them this quarter]
- Secondary: [what else they care about]

**Frustrations:**
- [Specific pain point — be concrete, not vague]
- [Second pain point]
- [Third pain point]

**How They Work:**
- Uses [tools] daily
- Technically comfortable with: [e.g., "Excel formulas, Google Sheets, basic SQL"]
- Technically uncomfortable with: [e.g., "anything with a terminal, YAML configs"]
- Accesses product via: [browser/mobile/desktop app]
- Typical session length: [e.g., "20-minute weekly check-in, not daily power user"]

**Representative Quotes (from real research):**
- "[Direct quote from an interview or survey]"
- "[Second quote]"

**Jobs To Be Done:**
When [situation], I want to [motivation], so I can [outcome].

---

## Persona 2: [Name] — [Role Title]
[Repeat structure above]

---

## Anti-Personas (Who We Are Not Building For)
- [Type of user]: [Why we don't optimise for them]
- [Type of user]: [Why]
```

**What makes a context file good vs. bad:**

| Good | Bad |
|------|-----|
| "Spends 2 days/month rebuilding the donor report in Excel" | "Has manual data workflows" |
| "Nervous about anything with a terminal or YAML" | "Non-technical user" |
| "Uses product weekly for ~20 minutes, not a daily power user" | "Occasional user" |
| Includes real quotes from interviews | Describes users from imagination |
| Updated after each research round | Written once, never touched |

### context/competitors.md

```markdown
# Competitive Landscape

Last updated: [date]

## Our Positioning
We win when: [specific scenario where we're clearly the right choice]
We lose when: [honest scenario where a competitor wins]
Our moat: [what's genuinely hard to replicate about us]

---

## [Competitor Name]

**What they do:** [One sentence]
**Pricing:** [Price point and model]
**Strengths:** [Specific, not generic]
**Weaknesses:** [What their users complain about — use G2/Capterra reviews]
**Who they target:** [Specific customer profile]
**Why we win against them:** [Concrete reason]
**Why we lose to them:** [Honest answer]

---

## [Competitor Name]
[Repeat structure]

---

## Positioning Matrix
| Capability | Us | Competitor A | Competitor B |
|------------|-----|--------------|--------------|
| [Key capability] | ✓ | ✓ | ✗ |
| [Key capability] | ✓ | ✗ | ✗ |
| [Key capability] | ✗ | ✓ | ✓ |
```

---

## 5. Custom Slash Commands — The 10 Most Valuable for PMs

Each command is a `.md` file in `.claude/commands/`. Create the file, type `/command-name` in Claude Code, and it runs.

### /prd — Generate a PRD from a brief

**File:** `.claude/commands/prd.md`

```markdown
---
description: Generate a PRD from a feature brief or rough notes
argument-hint: [paste your brief or describe the feature]
---

You are writing a Product Requirements Document for: $ARGUMENTS

Read context/product.md and context/users.md before starting.

Ask one clarifying question if the brief is ambiguous. Then generate the PRD using this structure:

## [Feature Name] — PRD

**Status:** Draft | **Author:** [from CLAUDE.md] | **Date:** [today]

### Problem Statement
[What problem this solves, for which persona, with evidence from users.md]

### Success Metrics
- Primary: [measurable outcome]
- Guardrail: [what we must not break]
- Anti-metric: [what would indicate we got it wrong]

### User Stories
[3-5 stories in "As a [persona], I want [action] so that [outcome]" format]
Each story must include acceptance criteria.

### Scope
**In scope:** [bulleted list]
**Out of scope:** [bulleted list — equally important]
**Future consideration:** [what we're intentionally deferring]

### Open Questions
[At least 3 unresolved questions that block or could change the spec]

### Dependencies
[Teams, systems, or external factors this relies on]
```

**When to use:** Whenever you have a brief, a Slack thread, or rough notes and need a first-draft PRD in minutes.

**Example:** Type `/prd Users need to be able to schedule their pipelines to run at specific times. Currently everything runs on manual trigger. Key constraint: cannot break existing manual runs.`

You get a structured PRD with problem statement, success metrics, and user stories — calibrated to your actual personas and terminology.

---

### /brief — Turn rough notes into a structured brief

**File:** `.claude/commands/brief.md`

```markdown
---
description: Turn rough notes, Slack messages, or meeting scribbles into a structured feature brief
argument-hint: [paste your raw notes]
---

Convert these rough notes into a clean feature brief: $ARGUMENTS

A good brief answers: What is the problem? Who has it? What does success look like? What are we NOT solving?

Output format:

## Feature Brief: [Name you infer from the notes]

**Problem:** [1-2 sentences. What is broken or missing?]
**Who:** [Which persona from context/users.md experiences this most acutely?]
**Evidence:** [What signals suggest this is real? Quote back any evidence in the notes.]
**Success looks like:** [Concrete, observable outcome]
**Not solving:** [What adjacent problems we are explicitly not tackling]
**Open questions before scoping:** [What needs answering before we can write a PRD]
**Suggested next step:** [What the PM should do next]

Flag anything in the notes that is contradictory or unclear.
```

**When to use:** After a stakeholder meeting, after reading a Slack thread, after a customer call. Takes messy input and gives you something you can share.

---

### /research — Synthesize research into insights

**File:** `.claude/commands/research.md`

```markdown
---
description: Synthesize interview notes, survey results, or research into structured insights
argument-hint: [paste raw notes or describe what research you have]
---

Synthesize the following research material: $ARGUMENTS

Use the Insight → Evidence → Implication format for each finding.

## Research Synthesis

**Source material:** [describe what you analyzed]
**Date range:** [when the research was conducted, if mentioned]
**Participants/sample:** [who was in the research, if mentioned]

### Key Findings

For each insight:

**Insight [N]: [Short headline — what you learned]**
- Evidence: [Direct quotes or data points supporting this]
- Frequency: [How many participants/data points — "5 of 8 interviewees said..."]
- Implication: [What this means for the product]
- Confidence: [High/Medium/Low — based on sample size and consistency]

### Contradictions or Tensions
[Where the data pulls in different directions]

### What We Still Don't Know
[Gaps this research didn't answer — critical for next research sprint]

### Recommended Actions
[3-5 specific follow-on actions for the PM]
```

**When to use:** After user interviews, after processing a survey, after reading analyst reports. Turns raw material into decisions.

---

### /compete — Competitive analysis on a topic

**File:** `.claude/commands/compete.md`

```markdown
---
description: Competitive analysis on a specific feature, market, or strategic question
argument-hint: [the feature or topic to analyze]
---

Run a competitive analysis on: $ARGUMENTS

Read context/competitors.md first to understand our current competitive landscape.

Structure your analysis as:

## Competitive Analysis: $ARGUMENTS

**Question being answered:** [Restate the analysis question clearly]

### What Competitors Are Doing
For each relevant competitor in competitors.md:
- **[Competitor]:** [What they do on this topic, how, and how well]
  - Evidence: [Where you're getting this from — product pages, reviews, etc.]

### Pattern Analysis
[What do most competitors have in common on this topic?]
[Where are they diverging?]
[What gaps exist that nobody is filling well?]

### Our Current Position
[How do we compare on this specific topic?]

### Strategic Options
1. [Option]: [What we could do] — Pros: [...] Cons: [...]
2. [Option]: [What we could do] — Pros: [...] Cons: [...]
3. [Option]: [What we could do] — Pros: [...] Cons: [...]

### Recommendation
[Which option fits our positioning and why]

Note: Flag any assumptions where you lack current data.
```

**When to use:** Before a roadmap review, before a competitive deal, before pricing decisions. Takes 30 seconds instead of 2 hours.

---

### /prioritize — RICE or ICE scoring on a feature list

**File:** `.claude/commands/prioritize.md`

```markdown
---
description: Score a feature list with RICE or ICE prioritization
argument-hint: [list your features, one per line]
---

Prioritize the following features: $ARGUMENTS

Ask me: "RICE (Reach/Impact/Confidence/Effort) or ICE (Impact/Confidence/Ease)?" before scoring.
Wait for my answer, then proceed.

For each feature, score on 1-10 scale and explain your reasoning briefly.

Output as a table:

| Feature | [R/I] | [I] | C | [E] | Score | Rationale |
|---------|-------|-----|---|-----|-------|-----------|
| [Feature] | [score] | [score] | [score] | [score] | [calculated] | [1 sentence why] |

Sort by score descending.

After the table:
**Top 3 to do next:** [with brief justification]
**Features worth cutting:** [features scoring below threshold]
**Key assumptions:** [what you'd need to validate to change the rankings]

Use context/product.md and context/users.md to calibrate your scoring.
```

**When to use:** Quarterly planning, sprint kickoff, stakeholder debates about what to build next.

---

### /interview — Generate interview questions for a persona

**File:** `.claude/commands/interview.md`

```markdown
---
description: Generate a user interview guide for a specific persona and topic
argument-hint: [persona name and topic, e.g. "Data Coordinator about reporting workflow"]
---

Create a user interview guide for: $ARGUMENTS

Read context/users.md to understand the persona.

## Interview Guide: [Persona] — [Topic]

**Goal of this interview:** [What we're trying to learn]
**Duration:** 45-60 minutes
**Format:** Semi-structured — use questions as prompts, not a script

### Warm-up (5 min)
[3 questions to establish context and make the participant comfortable]

### Core Questions (30 min)
Group into 3-4 themes. For each theme:
**Theme: [Name]**
- Opening question: [broad, open-ended]
- Follow-up 1: [dig into specifics]
- Follow-up 2: [surface the pain]
- Probe: "Tell me about the last time..." [concrete recent example]

### Concept Test (if applicable) (10 min)
[Placeholder — fill in with your prototype or concept to test]

### Closing (5 min)
- What would you tell a colleague about [product/feature]?
- If you could change one thing about [current workflow], what would it be?
- Who else should we talk to?

### What to Listen For
[Specific signals — language, behaviors, emotions — that would be meaningful]

### What Would Surprise Us
[What finding would most change our current assumptions]
```

**When to use:** Before a discovery sprint, when entering a new persona segment, when a competitor is winning deals you don't understand.

---

### /retro — Synthesize feedback into action items

**File:** `.claude/commands/retro.md`

```markdown
---
description: Synthesize feedback, retro notes, or post-launch reviews into clear action items
argument-hint: [paste retro notes, feedback, or support tickets]
---

Synthesize the following feedback into action items: $ARGUMENTS

## Retrospective Synthesis

**Source:** [describe input material]
**Period covered:** [if mentioned]

### What's Working (Keep Doing)
[Evidence-backed positives — not just "good vibes" but specific patterns]

### What's Not Working (Stop or Fix)
For each issue:
- **Issue:** [specific problem]
- **Evidence:** [how many times it appeared, direct quotes]
- **Impact:** [who it affects and how severely]
- **Recommended action:** [concrete next step with owner role]

### What to Try (New Experiments)
[Ideas that surfaced — mark as "hypothesis, not commitment"]

### Action Items
| Action | Owner | Due | Priority |
|--------|-------|-----|----------|
| [Action] | [Role] | [Timeframe] | High/Med/Low |

### Patterns to Watch
[Issues that appeared once but could be signals — flag for next retro]
```

**When to use:** After sprints, after launches, after support ticket reviews, after any feedback batch.

---

### /update — Turn notes into an executive update

**File:** `.claude/commands/update.md`

```markdown
---
description: Turn messy notes into a clean executive status update
argument-hint: [paste your notes or describe what happened this week]
---

Turn these notes into an executive status update: $ARGUMENTS

Executive readers want: decisions made, risks surfaced, asks — in under 3 minutes of reading.

## [Product Area] — Status Update
**Date:** [today] | **Author:** [from CLAUDE.md]

### Summary (2-3 sentences)
[What is the most important thing to know right now]

### Decisions Made This Period
- [Decision]: [Why we made it, who approved]

### In Progress
| Initiative | Status | On Track? | Notes |
|------------|--------|-----------|-------|
| [Initiative] | [Brief status] | ✓/⚠/✗ | [Key context] |

### Risks and Blockers
- [Risk/Blocker]: [What it is, what we're doing about it, what we need]

### Asks
- [Ask]: [Specific action needed from the reader, with deadline]

### Metrics Snapshot
[3-5 key metrics with trend direction — ↑ ↓ →]

### Next Two Weeks
[What will we have done by the next update]
```

**When to use:** Weekly PM reports, board updates, steering committee prep. Takes 5 minutes instead of 45.

---

### /spec — Turn a PRD section into engineering requirements

**File:** `.claude/commands/spec.md`

```markdown
---
description: Turn a PRD section into precise engineering requirements
argument-hint: [paste the PRD section or describe the feature]
---

Convert this into engineering-ready requirements: $ARGUMENTS

Read context/product.md to understand the tech stack before writing.

## Technical Specification: [Feature Name]

**Source PRD:** [reference the PRD if mentioned]
**Scope:** [what engineering needs to build]

### Functional Requirements
For each requirement use this format:
- **FR-[N]:** [The system shall / must / should...] — [Rationale]

Distinguish:
- **MUST** — required for launch
- **SHOULD** — important but not blocking
- **COULD** — nice-to-have if time allows

### Non-Functional Requirements
- Performance: [response time, throughput, concurrency expectations]
- Security: [auth, data access, PII considerations]
- Accessibility: [WCAG level, specific requirements]
- Browser/device: [supported environments]

### Edge Cases and Error States
[Enumerate the non-happy paths — what happens when things go wrong]

### API / Data Contracts (if relevant)
[Describe expected inputs/outputs at a conceptual level — not code, but clear enough for engineers]

### Acceptance Criteria (QA-testable)
- Given [context], when [action], then [expected outcome]
[One criterion per user story from the PRD]

### Open Technical Questions
[What the PM doesn't know that engineering needs to answer]
```

**When to use:** When handing off a PRD to engineering. Reduces back-and-forth and sprint planning surprises.

---

### /decision — Structure a decision with options and tradeoffs

**File:** `.claude/commands/decision.md`

```markdown
---
description: Structure a decision with clear options, tradeoffs, and a recommendation
argument-hint: [describe the decision you're facing]
---

Structure the following decision: $ARGUMENTS

Read context/decisions.md for precedent before generating options.

## Decision: [Decision title]

**Date:** [today] | **Decision owner:** [from CLAUDE.md]
**Decision deadline:** [when this needs to be made]
**Who needs to approve:** [stakeholders]

### Context
[Why this decision is needed now. What changes if we don't make it.]

### Options

#### Option A: [Name]
- **What it is:** [Description]
- **Pros:** [Specific benefits]
- **Cons:** [Specific costs and risks]
- **Effort:** [Time and resources]
- **Reversibility:** [Easy to undo / Hard to undo / Irreversible]

#### Option B: [Name]
[Same structure]

#### Option C: [Name — always include a "do nothing / defer" option]
[Same structure]

### Decision Criteria
[What matters most in making this call — and who set those criteria]

### Recommendation
**Recommended option:** [A/B/C]
**Rationale:** [Why this option best fits the criteria]
**Key assumptions:** [What must be true for this recommendation to hold]
**What would change the recommendation:** [Conditions under which a different option is better]

### Open Questions Before Deciding
[What information would change the recommendation if you had it]
```

**When to use:** Before any significant decision — build vs. buy, pricing changes, roadmap bets, team structure. Surfaces options you hadn't thought of and forces explicit criteria.

---

## 6. MCP Connections That Give PMs Superpowers

MCP (Model Context Protocol) lets Claude Code read and write directly to your tools. You stop copying and pasting between apps.

### Setup: Where the config lives

Add MCP servers to your project's `.mcp.json` file (for team-shared config) or to your global Claude Code config (`~/.claude/claude.json`) for personal tools.

```json
{
  "mcpServers": {
    "linear": {
      "command": "npx",
      "args": ["-y", "@mseep/linear-mcp"],
      "env": {
        "LINEAR_API_KEY": "your_linear_api_key_here"
      }
    },
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "NOTION_API_KEY": "your_notion_integration_token_here"
      }
    }
  }
}
```

### The six MCP servers worth setting up

**1. Linear (or Jira)**

Add with: `claude mcp add linear -- npx -y @mseep/linear-mcp`

Get your API key: Linear → Settings → API → Personal API Keys

What it unlocks:
- "Create a Linear ticket from this PRD section" — auto-generates issues with acceptance criteria
- "Summarize all tickets tagged [label] this sprint" — instant sprint review prep
- "What are the open blockers across our current projects?" — status without a meeting
- "Which tickets have been sitting in backlog longer than 30 days?" — backlog grooming

**2. Notion**

Add with the Notion MCP server. Get your token: Notion → Settings → Integrations → Create new integration → share pages you want Claude to access.

What it unlocks:
- "Find the decision we made about pricing in Q3" — searches across all your workspace docs
- "Update the product wiki with today's PRD" — writes directly to Notion pages
- "Summarize this 40-page spec doc" — read Notion pages into active context

**3. Google Drive**

What it unlocks:
- Read strategy docs, research reports, and OKR sheets into context without copy-pasting
- "What does our current roadmap doc say about Q2 bets?" — answers from Drive
- Particularly useful for orgs where the source of truth lives in Slides/Docs/Sheets

**4. GitHub**

What it unlocks:
- "What has engineering shipped in the last 2 weeks?" — pull from merged PRs
- "Are there any open issues tagged [persona name] that we should triage?"
- See implementation decisions engineers made that diverge from the spec

**5. Slack (via Slack MCP)**

What it unlocks:
- "Summarize what the #customer-feedback channel said this week" — replaces manual monitoring
- "Find all messages about [feature] in the last 30 days" — research without sifting
- Post generated reports directly to channels without switching apps

**6. PostHog or Amplitude (analytics)**

What it unlocks:
- "Pull last week's activation funnel data and tell me where users are dropping off"
- "Did the new onboarding flow change our 7-day retention?" — no SQL, no BI tool
- Turns analytics from a separate workflow into part of every PRD conversation

**Honest note on setup effort:** Linear and Notion take 10 minutes each. Google Drive OAuth takes 20 minutes and requires some terminal comfort. GitHub is 5 minutes if you already have a token. Start with Linear or Notion — they deliver the highest return for the least setup pain.

---

## 7. Day 1 Setup Checklist

Target: 2 hours. Work through this sequentially.

**Hour 1: Structure and context**

- [ ] Install Claude Code: `npm install -g @anthropic-ai/claude-code` (requires Node.js)
- [ ] Create the workspace folder: `mkdir pm-workspace && cd pm-workspace`
- [ ] Create the folder structure (copy the tree from Section 2 above)
- [ ] Create `CLAUDE.md`: paste the template from Section 3, fill in every section
- [ ] Create `context/company.md`: 20 minutes of honest writing
- [ ] Create `context/product.md`: describe what you've built so far
- [ ] Create `context/users.md`: write at least one full persona with a real quote

**Hour 2: Commands and first session**

- [ ] Create `.claude/commands/` folder
- [ ] Add three commands to start: `prd.md`, `brief.md`, and `decision.md` (from Section 5)
- [ ] Open Claude Code in your workspace folder: `claude`
- [ ] Test CLAUDE.md loaded: ask "What product do I work on?" — Claude should answer from your context
- [ ] Test `/brief` with real notes from your most recent meeting
- [ ] Test `/decision` with a real decision you're sitting on
- [ ] Review the outputs: did they feel calibrated to your product? Adjust CLAUDE.md if not.

**Post-setup: add one MCP connection**

- [ ] Pick one: Linear, Notion, or Google Drive
- [ ] Follow the setup steps in Section 6
- [ ] Test it: ask Claude to read something from that tool
- [ ] Confirm it works before moving on

---

## 8. What to Do in Week 1

The goal of week 1 is not to build a perfect system. It is to build the habit of opening Claude Code instead of a blank doc.

**Day 1 (today):** Complete the Day 1 checklist above.

**Day 2:** Add `context/competitors.md`. Write one competitor entry per the template. Use this to run `/compete` on a feature your main competitor shipped recently.

**Day 3:** Add two more slash commands. Pick two from Section 5 that match work you have coming up this week.

**Day 4:** Fill in `context/decisions.md` with 3-5 decisions your team has made in the last 6 months and why. This becomes the context for every `/decision` run going forward.

**Day 5 (end of week review):** 
- What outputs felt off? Find the cause in CLAUDE.md and fix it.
- What did you explain to Claude in conversation that it should have known from context? Add it to CLAUDE.md or the relevant context file.
- What command do you wish existed? Create a draft `.md` file for it.

**Rule for week 1:** After every Claude session, spend 2 minutes on this question: "What would I need to add to CLAUDE.md so I don't have to explain that again?" That is the entire maintenance loop.

---

## 9. Common Mistakes and How to Avoid Them

**Mistake 1: CLAUDE.md too long**

If CLAUDE.md exceeds 150 lines, Claude will skim it. Move detailed content to context files and reference them from CLAUDE.md. The master file should be the quick-reference layer — behavioral rules, personas in brief, terminology. The deep content lives in `context/`.

**Mistake 2: Context files too generic**

"Our users are non-technical" is useless. "Our users are comfortable with Excel formulas but freeze when they see a terminal window or YAML config" is actionable. Every sentence in a context file should be specific enough to generate a different output than a competitor's context file would.

**Mistake 3: Not updating context after it changes**

A context file that describes a pivot you made 6 months ago actively harms your outputs. Put a calendar reminder for the first Monday of each quarter: "Update context files." It takes 20 minutes and pays back immediately.

**Mistake 4: Treating slash commands as magic one-shots**

Commands work best as first drafts. The workflow is: run the command → get 80% of the way there → refine in conversation. Don't expect a single `/prd` invocation to produce a shippable document. Use it to eliminate the blank page problem.

**Mistake 5: Wrong folder for the wrong content**

Do not put sprint goals, weekly OKRs, or meeting agendas in CLAUDE.md. Put them in `strategy/` or `research/`. CLAUDE.md is for stable, durable context. When it starts changing weekly, it stops working.

**Mistake 6: Skipping the clarifying question step**

The commands above ask Claude to request one clarifying question before generating major artifacts. Remove that instruction and you get confident-sounding outputs that miss the real problem. Keep the question step. It slows you down by 30 seconds and saves you 30 minutes of revision.

---

## 10. Real Examples: Before and After

### Example 1: PRD for a scheduling feature

**Without context — prompt sent:**
> "Write a PRD for letting users schedule their data pipelines to run automatically"

**Output without context (excerpt):**
> "The Pipeline Scheduling Feature will allow users to configure automated execution of their data pipelines. This document outlines the requirements for implementing a scheduling system. Target users include data teams and business analysts who need reliable data freshness..."

Generic. Could be for any product. No persona specificity, no company terminology, no success metrics tied to real goals.

**With PM OS — prompt sent:**
> `/prd Users need to schedule pipelines to run automatically. Right now everything is manual trigger. Key constraint: cannot break existing manual runs. Persona: Data Coordinator.`

**Output with PM OS (excerpt):**
> "## Pipeline Scheduling — PRD
> **Problem Statement:** Data Coordinators at partner NGOs spend time manually triggering pipelines that should run on a predictable schedule. Currently 100% of pipeline runs are triggered manually, meaning the coordinator must remember to do it or data goes stale before the weekly report. This is one of the top three complaints surfaced in Q1 research (3 of 5 NGOs interviewed mentioned it unprompted).
>
> **Success Metrics:**
> - Primary: % of active pipelines with a schedule configured (target: 60% within 60 days)
> - Guardrail: Pipeline success rate does not decrease after launch
> - Anti-metric: Support tickets about 'my pipeline ran when I didn't expect it'"

Calibrated to the product, the persona, and the research language. Uses "Partner NGO" not "customer." References real research. Success metrics that mean something.

---

### Example 2: Competitive analysis on notifications

**Without context:**
> "Compare how our competitors handle notification features"

**Output:** A generic comparison of notification best practices with no connection to your product's position, your users' needs, or your competitive differentiation.

**With PM OS:**
> `/compete How do our main competitors handle in-app notifications and alert fatigue?`

**Output:** Claude reads `context/competitors.md`, identifies the 3 relevant competitors, analyzes each specifically on notifications, notes where you currently stand, and produces 3 strategic options — including one that targets the gap where no competitor handles low-bandwidth environments well (because your `context/product.md` mentioned that your users are on slow internet in East Africa).

That last insight is only possible because the context was there. Claude connected a gap in competitors to a user constraint in your context. That is the PM OS working as designed.

---

### Example 3: Decision structuring

**Without context:**
> "Help me decide whether to build our own notification system or use a third-party service"

**Output:** Generic pros and cons of build vs. buy. No mention of your team size (4 engineers), your budget constraints (grant-funded), your prior decisions, or the maintenance burden that matters most for a small team.

**With PM OS:**
> `/decision Build vs. buy for notifications. We have 4 engineers, one is leaving next month. Budget is fixed for 12 months. We already use AWS.`

**Output:** Claude reads `context/decisions.md`, notes that in Q2 you decided to buy rather than build for auth for the same reason (small team, fixed budget), and surfaces that prior decision as a precedent. The recommendation is "buy" with specific vendor suggestions calibrated to AWS and the team size, and flags that the engineer departure makes the build option higher-risk than the standard analysis would show.

---

## Closing Note

The PM OS is not a setup you do once. It is a living system you refine as you learn what makes Claude's outputs better for your specific context. The teams that get the most value treat their `CLAUDE.md` and context files the same way they treat the product: as something worth maintaining, not just shipping.

Start small. Get one context file right. Ship one command that you use every day. Then build from there.

---

*Sources used in researching this guide:*
- [Claude Code for Product Managers — prodmgmt.world](https://www.prodmgmt.world/resources/claude-code)
- [CLAUDE.md for Product Managers — ccforpms.com](https://ccforpms.com/fundamentals/project-memory)
- [PM OS by Sevan Pro](https://sevanpro.com/pm-os)
- [Carl's Product OS — GitHub](https://github.com/carlvellotti/carls-product-os)
- [Product-Manager-Skills — Dean Peters on GitHub](https://github.com/deanpeters/Product-Manager-Skills)
- [PM Claude Code Setup — aakashg on GitHub](https://github.com/aakashg/pm-claude-code-setup)
- [Claude Code MCP Integrations for PMs — mysecond.ai](https://mysecond.ai/blog/mcp-integrations-product-management)
- [Claude Code for Product Managers — Builder.io](https://www.builder.io/blog/claude-code-for-product-managers)
- [Slash Commands Documentation — Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/slash-commands)
- [Creating Reusable Slash Commands — CodeSignal](https://codesignal.com/learn/courses/customizing-claude-code-for-reusable-visualization-workflows/lessons/creating-custom-slash-commands)
