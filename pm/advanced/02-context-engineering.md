# Context Engineering for Product Managers
## A Hands-On Workshop Guide

---

You have used Claude. You asked it to write a product brief or analyze a competitive landscape, and it gave you something technically correct, competently structured, and completely useless for your actual situation. It did not know your users. It did not know your constraints. It could not distinguish your product from a thousand others.

This is not Claude's fault. Claude knows exactly what you told it. The problem is what you told it — or rather, what you did not.

This guide fixes that systematically.

---

## 1. Why Your Claude Outputs Are Generic (and It Is Not Claude's Fault)

### The Context Window Mental Model

Every Claude session starts blank. There is no memory of your last conversation, no awareness of your product, no knowledge of your users or your market. Claude is extraordinarily capable, but it operates entirely on what you put in front of it in this session. Everything it knows about your situation lives in a single window — the context window — and when that session ends, everything disappears.

Think of it this way: you have hired the best product consultant in the world, but each day they wake up with complete amnesia. They still have all their expertise, judgment, and frameworks. But they do not know who you are, what you are building, or what you have already decided. Every morning, you have to brief them from scratch.

Most PMs give that consultant a one-line brief: "Write me a positioning statement for our B2B SaaS product."

The consultant does their best with a blank slate. They write something technically sound. They use the patterns that have worked for B2B SaaS in the abstract. The output is generic because the input was generic.

Context engineering is the practice of briefing that consultant properly.

### Prompt Engineering vs. Context Engineering

These two terms are often confused. They are not the same thing.

**Prompt engineering** is what you do inside a single message. It is about how you ask — the phrasing, the structure, the constraints you put in one request. Prompt engineering produces one-off improvements. A better prompt gets you a better answer this time.

**Context engineering** is about what information fills the context window before you ever type your question. It is about what Claude knows — your product, your users, your principles, your voice — when it begins processing your request. Context engineering is systematic and persistent. You build it once, and it makes every single future prompt better.

The relationship is this: prompt engineering is tactics. Context engineering is strategy.

Here is a precise definition from the AI engineering community: "Context engineering is the discipline of deliberately designing and managing everything the model sees or knows when it generates a response — instructions, retrieved knowledge, memory, examples, and prior outputs — all structured so the model can use them effectively."

Prompt engineering is what you do inside the context window. Context engineering is how you decide what fills the window.

### Why the Same Prompt Gives a 10x Better Answer With Proper Context

The output quality of any Claude response is roughly proportional to the relevance and specificity of what Claude knows about your situation. Generic input produces generic output. Specific, rich, accurate context produces specific, rich, accurate output.

This is not a mystery. Claude is synthesizing and reasoning over the information it has been given. If that information is sparse, Claude fills the gaps with statistical patterns — what is most common for this type of product, this type of company, this type of PM. The more your situation resembles the average, the less this matters. The more your situation is specific — your users, your constraints, your market — the more those gaps hurt.

### Concrete Before/After: Same Question, Two Contexts

**The question is the same in both cases:**

> "Write positioning copy for our product. Two sentences, for our homepage hero."

---

**Without context (bare prompt):**

```
User: Write positioning copy for our product. Two sentences, for our homepage hero.

Claude: [Product Name] empowers teams to work smarter, not harder.
Streamline your workflows and achieve more with our all-in-one platform.
```

Generic. Could be any software product in the world. Useless.

---

**With context loaded (from context files described in this guide):**

```
User: Write positioning copy for our product. Two sentences, for our homepage hero.

Claude: Fieldwork turns the survey data your program managers already collect into
the donor reports your funders actually require — in minutes, not weeks.
Built for NGOs who run on impact, not on IT budgets.
```

Specific. Differentiated. Speaks to the actual user and the actual problem. Usable.

The question was identical. The context was different. That is the entire point.

---

## 2. The Context Stack

Context comes in layers. Understanding which layer you are working with changes how you maintain it and when you load it.

### Layer 1: Persistent Context

This is context that is always present, automatically, in every session. It lives in files that Claude Code reads at startup — primarily your `CLAUDE.md` file and any context files it imports.

Persistent context answers the question: "What does Claude need to know about me and my product in every single session, no matter what I am working on?"

Put here: your role, your product description, your key users, your voice and tone rules, your non-negotiables, your terminology conventions.

Do not put here: anything that changes more than once a month, or anything session-specific.

### Layer 2: Session Context

This is what accumulates through the current conversation. Every message you send, every answer Claude gives, every file you reference — all of this becomes context within the session.

Session context is why you do not need to repeat yourself within a single working session. Tell Claude something once, and it knows it for the rest of that session.

Session context resets when the session ends. This is why persistent context matters.

### Layer 3: Task Context

This is context you deliberately load for a specific task. You are about to write a competitive analysis — you load your `market.md` context file. You are writing a PRD for a specific feature — you load the relevant user research transcript, the relevant stakeholder constraints, the relevant metrics.

Task context is targeted. You bring in what this task needs and nothing else. This keeps the context window clean and focused.

### Layer 4: Retrieved Context

This is live information you pull in during a session — a competitor's pricing page, a user interview transcript from this week, the latest dashboard export, a Slack thread from yesterday. Retrieved context is dynamic. It brings current, specific information that could not live in a static file.

### How the Layers Work Together

```
Session start
     │
     ▼
CLAUDE.md loads automatically ──────────► Persistent context always present
     │
     ▼
You reference @context/users.md ────────► Task context: users loaded for this task
     │
     ▼
You paste a competitor's pricing page ──► Retrieved context: live competitive data
     │
     ▼
You ask your question ──────────────────► Claude synthesizes all four layers
```

A well-engineered context stack means the question itself can be short. Claude already knows the rest.

---

## 3. Building a Modular Context Library: The Teresa Torres Approach

Teresa Torres, author of *Continuous Discovery Habits* and one of the most rigorous thinkers in product management, has publicly described her context engineering system. Her key insight: instead of one massive context file that grows unwieldy and expensive, build a modular library of small, focused files. One file per topic area. Load only what you need.

She uses an Obsidian vault called `LLM Context` with separate folders for different domains. Her global `CLAUDE.md` is lean — it tells Claude where to find specific information, not what that information is. Her writing style guide is its own file. Her audience profiles are their own files. Each module is small, maintainable, and independently loadable.

This is the modular context library. You build it once. You update each module independently. You load specific modules for specific tasks.

### What a Modular Context Library Is

A folder of markdown files, each covering one focused topic area. Each file is:
- Short enough to read in two minutes
- Specific enough to actually constrain Claude's outputs
- Stable enough to be worth maintaining

The whole library lives somewhere accessible — your project folder, your Obsidian vault, a dedicated `context/` directory. In Claude Code, you reference any file with `@path/to/file.md`.

### The 10 Context Modules Every PM Needs

Below are the 10 modules, what to include in each, what to exclude, and a complete, copy-paste-ready template for each one.

---

### Module a: `company.md` — Mission, Strategy, Constraints

**What to include:** Your company's reason for existing (the actual mission, not the PR version), the strategy you are currently executing, the constraints that are genuinely fixed. This is the context that makes Claude understand your situation rather than the generic company in its training data.

**What to exclude:** Marketing copy, aspirational vision that does not actually constrain decisions, financial details that change quarterly.

**Template:**

```markdown
# Company Context

## Who We Are
[Company name] is a [description in one sentence: what you do, for whom].

**Founded:** [Year]
**Stage:** [Seed / Series A / B / Profitable SMB / etc.]
**Team size:** [Number]
**Geography:** [Where you operate]

## Mission
[One sentence. What would be lost if this company did not exist?]

## Current Strategy (valid through [quarter/year])
[Two to three sentences. What are you betting on right now?
What is the theory of growth you are executing?]

## Fixed Constraints
These are not up for debate in any context:
- [Constraint 1: e.g., "We are AGPL-3.0 open source. No proprietary lock-in."]
- [Constraint 2: e.g., "We do not sell to for-profit companies."]
- [Constraint 3: e.g., "Budget: ~$X per year total engineering. All decisions live in that constraint."]

## Business Model
[How you make money, or how you sustain yourselves if nonprofit]

## What Success Looks Like in 12 Months
[Two to three specific, observable outcomes — not metrics, just observable reality]
```

**Filled-in example:**

```markdown
# Company Context

## Who We Are
Dalgo is an open-source data platform that helps NGOs replace manual Excel workflows
with automated data pipelines, transformation, and dashboards.

**Founded:** 2021
**Stage:** Sustained by PT4D, open source
**Team size:** ~8 (engineering + product)
**Geography:** India-headquartered, global NGO partners

## Mission
Put the data infrastructure that large INGOs take for granted within reach of small
NGOs operating on tight budgets and no dedicated IT staff.

## Current Strategy (valid through Q3 2026)
Deepen the self-serve onboarding experience so NGOs can go from signup to their first
working pipeline without contacting support. Land the first five NGOs who configure
Dalgo entirely without engineering help.

## Fixed Constraints
- AGPL-3.0 open source. No feature is closed-source.
- Budget: ~₹2L/year per NGO partner. Every decision lives in that constraint.
- Users are non-technical. "Non-technical" means no SQL, no APIs, no CLI.

## Business Model
Dalgo is funded by Project Tech 4 Dev (PT4D). Partners pay a small annual contribution.
The platform is free for NGOs to self-host.

## What Success Looks Like in 12 Months
Five new NGO partners running production pipelines with zero engineering support.
The onboarding flow needs no explanation from the Dalgo team.
```

---

### Module b: `product.md` — What You Build, For Whom, Current State

**What to include:** The product's scope, the current feature set at a summary level, the version/state it is in, what it does NOT do (equally important), and the key architectural or product decisions that define its shape.

**What to exclude:** Roadmap items (those go in `roadmap.md`), detailed specs (those are task context loaded on demand).

**Template:**

```markdown
# Product Context

## What [Product] Is
[One paragraph: what the product does, for whom, and what the core value exchange is]

**Current version:** [X.X]
**Status:** [Alpha / Beta / GA / Mature]

## Core Capabilities (what it does today)
- [Capability 1]
- [Capability 2]
- [Capability 3]
- [Capability 4]

## Explicit Non-Scope (what it does NOT do)
- [Non-scope 1 — and why if it matters]
- [Non-scope 2]

## Key Product Decisions That Define Its Shape
- [Decision 1: e.g., "We orchestrate with Prefect, not Airflow. This is locked."]
- [Decision 2: e.g., "All UI is in the web app. No desktop client."]

## The User Journey (high-level)
1. [Step 1]
2. [Step 2]
3. [Step 3]

## Current Health Assessment
[Two to three honest sentences: where is the product strong, where is it weak,
what is the biggest open problem]
```

---

### Module c: `users.md` — Personas, Jobs-to-Be-Done, Pain Points

This is often the highest-value context module. When Claude knows your users specifically — their language, their frustrations, what they are actually trying to accomplish — its outputs shift dramatically.

**What to include:** Two to four specific personas with enough texture to be real, the job-to-be-done for each (what they are hiring your product to do), their actual vocabulary, their biggest frustrations, what success looks like from their perspective.

**What to exclude:** Demographic data that does not affect product decisions, aspirational personas you hope to serve someday but do not yet.

**Template:**

```markdown
# User Context

## Persona 1: [Name] — [Role]

**Who they are:** [Two sentences. Specific enough that a real person comes to mind.]

**Job to be done:** When [situation], I want to [motivation], so I can [outcome].

**Their actual vocabulary:**
- They say "[term A]", never "[term B we use internally]"
- They call the thing we call "[X]" a "[Y]"

**Biggest frustrations (verbatim-style):**
- "I spend three days every month just cleaning up the data before I can even look at it."
- "Every time the government changes the reporting format, we have to redo everything."

**What success looks like to them:**
[One sentence from their perspective, not ours]

**What they fear:**
[What outcome would make them feel the product failed]

---

## Persona 2: [Name] — [Role]

[Same structure]

---

## Jobs We Are NOT Hired For
- [Non-job 1: e.g., "Statistical analysis or research — they use R/Stata for that"]
- [Non-job 2]

## Key User Insight
[One sentence: the non-obvious thing about your users that most products get wrong]
```

---

### Module d: `market.md` — Competitive Landscape, Positioning

**What to include:** The competitive set your users actually consider (not the ones you consider), how each competitor is positioned, where you are differentiated, the market dynamics that matter to your decisions.

**What to exclude:** Exhaustive competitive feature matrices (load those on demand for competitive analysis tasks).

**Template:**

```markdown
# Market Context

## Competitive Set
These are the alternatives our users actually consider when evaluating Dalgo:

### [Competitor 1]
- **What they do:** [One sentence]
- **Their positioning:** [How they describe themselves]
- **Why users choose them:** [Honest assessment]
- **Why users choose us instead:** [Honest assessment]
- **Their price point:** [Rough pricing]

### [Competitor 2]
[Same structure]

### The Status Quo Competitor
Many of our users are not choosing between us and software.
They are choosing between us and [Excel / manual process / consultants].
This is often the real competition.

## Our Positioning
[One paragraph: who we are for, what we uniquely do for them, what we are not]

## Market Dynamics
- [Dynamic 1: e.g., "Donor reporting requirements are increasing; NGOs have no
  good tools to handle this"]
- [Dynamic 2: e.g., "The market is bifurcated: enterprise INGOs (Salesforce, 
  PowerBI) vs. small NGOs (nothing)"]

## Where We Win
[Specific situations or use cases where we reliably beat alternatives]

## Where We Lose
[Honest: where do users choose something else, and why]
```

---

### Module e: `data.md` — Key Metrics, Definitions, Baselines

Without this module, Claude makes up numbers or reasons from generic SaaS benchmarks. With it, Claude reasons from your actual data.

**What to include:** The metrics that actually drive decisions, their precise definitions (this is critical — "active user" means different things to different companies), current baselines, and the metrics you explicitly do NOT optimize for.

**What to exclude:** Every metric on your dashboard. Include only the ones that actually change product decisions.

**Template:**

```markdown
# Data and Metrics Context

## North Star Metric
**[Metric name]:** [Precise definition]
**Current value:** [X]
**Target:** [Y by date]
**Why this:** [One sentence on why this metric represents real value delivered]

## Key Product Metrics

| Metric | Definition | Current | Target | Direction |
|--------|-----------|---------|--------|-----------|
| [Metric 1] | [Exact definition] | [X] | [Y] | [Up/Down] |
| [Metric 2] | [Exact definition] | [X] | [Y] | [Up/Down] |
| [Metric 3] | [Exact definition] | [X] | [Y] | [Up/Down] |

## Metric Definitions (Critical — Do Not Guess These)
- **[Term 1]:** [Exact definition. Be specific: "A user who has logged in AND
  completed at least one sync in the last 28 days."]
- **[Term 2]:** [Exact definition]

## Anti-Metrics (What We Do NOT Optimize For)
- [Anti-metric 1]: We do not optimize [X] because [reason]
- [Anti-metric 2]: [Same]

## Known Data Gaps
- [Gap 1: e.g., "We cannot currently measure time-to-first-value accurately
  because our onboarding tracking is broken in the mobile flow."]
```

---

### Module f: `decisions.md` — Key Decisions Made and Why

This is the module most PMs skip. It is also among the most valuable. When Claude knows what you have already decided and why, it stops relitigating settled questions and reasons forward from your actual position.

**What to include:** The most consequential decisions from the last 12 months, the reasoning behind them, and the constraints they create going forward.

**What to exclude:** Decisions that might be revisited in the next sprint, or small tactical decisions that do not constrain future work.

**Template:**

```markdown
# Key Decisions Log

## [Decision 1 — short title] (Made: [Month Year])

**What we decided:** [One sentence]

**Why:** [Two to three sentences. The actual reasoning, not the post-rationalization.]

**What this constrains:** [What this decision closes off going forward]

**Trigger to revisit:** [What would have to be true for us to reconsider this]

---

## [Decision 2 — short title] (Made: [Month Year])

[Same structure]

---

## Decisions We Have Explicitly NOT Made
Sometimes it is important to document that a question is open:
- [Open question 1: e.g., "We have not decided whether to build a mobile app.
  This is a live question."]
```

---

### Module g: `principles.md` — Product Principles and Non-Negotiables

Product principles are not values statements. They are decision-making tools. A good principle tells you what to do when two reasonable options are in tension. A bad principle is something everyone agrees with all the time.

**What to include:** Four to eight principles that have actually resolved real debates at your company. For each, include the tension it resolves and a concrete example.

**What to exclude:** Principles that are universally accepted (no one argues against "build for users"). Those are not principles, they are platitudes.

**Template:**

```markdown
# Product Principles

These are the principles that resolve real tradeoffs at [Company].
Each one has lost something in practice — that is how you know it is real.

## [Principle 1 — stated as a directive, not a value]

**The tension it resolves:** [What two things are in conflict]

**What it means in practice:**
- We do [X] even when it is tempting to do [Y]
- Example: [Concrete past decision this resolved]

**What it does NOT mean:** [Common misapplication to avoid]

---

## [Principle 2]

[Same structure]

---

## Our Non-Negotiables
These are not principles (which involve tradeoffs) — they are lines we do not cross:
- [Non-negotiable 1]
- [Non-negotiable 2]
```

---

### Module h: `stakeholders.md` — Who Matters, Their Goals, Communication Style

**What to include:** The people and organizations whose input shapes your decisions, what they care about, how they communicate, and what they need from your product team.

**What to exclude:** The full org chart. Focus on the people who actually influence product decisions.

**Template:**

```markdown
# Stakeholder Context

## Internal Stakeholders

### [Name / Role]
- **What they care about:** [Their actual goals, not the official ones]
- **Communication style:** [How they like to receive information]
- **Typical concerns about product decisions:** [What they push back on]
- **What earns their trust:** [What kind of evidence moves them]

---

## External Stakeholders / Partners

### [Organization / Type]
- **Their relationship to us:** [One sentence]
- **What they need from us:** [Specific output or outcome]
- **Their constraints:** [Budget, timeline, technical]
- **How to communicate with them:** [Style, format, frequency]

---

## Decision Authority Map
[Who makes which types of decisions — keep it to decisions that actually
come up in your product work]

| Decision type | Authority | Who consults |
|--------------|-----------|-------------|
| [Type 1] | [Person/role] | [Who] |
| [Type 2] | [Person/role] | [Who] |
```

---

### Module i: `roadmap.md` — Current Roadmap, Upcoming Bets

**What to include:** The current committed roadmap, the bets behind it, and the explicit assumptions each bet depends on.

**What to exclude:** Wish-list items, anything more than two quarters out (that level of specificity is usually noise).

**Template:**

```markdown
# Roadmap Context

## Current Quarter: [Q X Year]

### In Progress
- **[Feature/Initiative 1]:** [One sentence on what it is and why now]
  - Status: [% done / milestone]
  - Assumption it depends on: [What must be true for this to matter]
  
- **[Feature/Initiative 2]:** [Same]

### Committed (Next Quarter)
- **[Feature/Initiative 3]:** [One sentence]
  - The bet: [Why we believe this is worth doing]
  - Key risk: [What could make this bet wrong]

## Explicit Deprioritizations
Things we have consciously decided NOT to work on this half:
- [Thing 1] — because [reason]
- [Thing 2] — because [reason]

## Open Questions the Roadmap Depends On
[Assumptions that, if proven wrong, would change the roadmap materially]
- [Assumption 1]
- [Assumption 2]
```

---

### Module j: `voice.md` — Writing Style, Tone, Format Preferences

This module transforms Claude from a generic business writer into something that sounds like you. Teresa Torres specifically credits her writing style guide as one of the highest-leverage context files she has built — it is what lets her prompt Claude with "gimme feedback" and get back output that matches her actual voice.

**What to include:** Your voice and tone, what you always do, what you never do, format preferences, and examples of writing that represents your standard.

**What to exclude:** Generic writing advice that applies to everyone.

**Template:**

```markdown
# Voice and Writing Style

## Tone
[Two to three adjectives, then explain each one with an example of what it means
in practice]

- **[Adjective 1]:** We are [X], which means we [specific behavior]. 
  We are NOT [common misapplication].
- **[Adjective 2]:** [Same]

## What We Always Do
- Use [specific structural pattern]
- Lead with [what]
- Define [terms that might be ambiguous] on first use
- [Other specific positive instructions]

## What We Never Do
- Never use [specific banned phrase or construction]
- Never [behavior that violates our voice]
- Never lead with [thing]

## Terminology (Always Use These, Never the Alternatives)
| Use | Never use | Why |
|-----|-----------|-----|
| [Term] | [Alternative] | [Reason] |
| [Term] | [Alternative] | [Reason] |

## Format Preferences
- **Documents:** [Structure preference: headers / prose / bullets]
- **Emails:** [Length and structure preference]
- **Slack messages:** [Style: formal / casual, length]
- **Meeting notes:** [Format preference]

## Example of Writing That Represents Our Standard
[Paste one paragraph of actual writing from your team that you consider exemplary.
Claude will pattern-match to this.]
```

---

## 4. Context Loading Patterns

### When to Load Everything vs. Specific Modules

Loading all 10 modules at once consumes significant context window space and can reduce the signal-to-noise ratio for tasks that only need some of that information. The general rule:

**Load everything** when: starting a new product initiative, doing a strategic review, onboarding a new person or stakeholder, or doing any work where you genuinely do not know which modules are relevant.

**Load specific modules** when: you know the task is focused. Writing a PRD? Load `users.md`, `product.md`, and `principles.md`. Analyzing a competitor? Load `market.md` and `product.md`. Writing a stakeholder update? Load `stakeholders.md` and `voice.md`.

### How to Reference Context Files in Claude Code

In Claude Code, the `@` symbol loads a file into context:

```
@context/users.md   — loads the full users context file
@context/           — shows a directory listing (not file contents)
@context/users.md @context/market.md   — loads multiple files
```

You can also reference external resources through MCP integrations:
```
@linear:issue://PM-234
@github:issue://123
```

CLAUDE.md files load hierarchically. A `CLAUDE.md` in your home directory loads globally. A `CLAUDE.md` in your project folder loads for that project. A `CLAUDE.md` in a specific subdirectory loads for work in that directory. Deeper files override shallower ones when they conflict.

### Building Slash Commands That Auto-Load Context

In Claude Code, you can create custom slash commands that automatically load the right context for each task type. These live in `.claude/commands/` as markdown files.

**Example: `/pm/discovery` command**

Create `.claude/commands/pm/discovery.md`:

```markdown
You are helping with product discovery work.

Load context:
@context/users.md
@context/product.md
@context/data.md

Discovery session focus: $ARGUMENTS

For discovery work, follow these principles:
- Lead with user problems, not solutions
- Challenge assumptions explicitly
- Ask what we would need to believe for a solution to work
- Identify what we do not yet know
```

Now running `/pm/discovery "synthesize this week's user interviews"` automatically loads users, product, and data context before Claude begins.

**Other useful commands to create:**
- `/pm/strategy` — loads company, market, decisions, roadmap
- `/pm/write-prd` — loads users, product, principles, voice
- `/pm/stakeholder-update` — loads stakeholders, voice, roadmap
- `/pm/competitive` — loads market, product, data

### The Context Sandwich

Structure your prompt in three parts:

```
[1. Context layer — what Claude needs to know]
[2. Task — what you want]
[3. Constraints — what good output looks like]
```

Example:

```
@context/users.md @context/product.md

[1. Context loaded above via @ references]

[2. Task]
I need a 3-question survey to send to our Priya personas after their first 
pipeline run. We want to understand whether they understood what happened 
and whether they trust the output.

[3. Constraints]
- Questions must use plain language — no technical jargon, no SQL, no "pipeline"
- Must be answerable in under 2 minutes
- Avoid leading questions
- Use our voice: direct, warm, no corporate filler
```

The research on LLM performance shows that important information at the beginning and end of context performs better than the same information buried in the middle. Put your key constraints and the most critical context at the top. Put your most important instruction at the end, right before Claude responds.

---

## 5. Dynamic Context: Pulling in Live Information

Static context files cover what you know and have documented. Dynamic context covers the live, specific information needed for a particular analysis. Here is how to do each type.

### Loading User Interview Transcripts for Synthesis

You have just run five discovery interviews. You want synthesis.

```
I have five user interview transcripts below. 
@context/users.md is loaded so you know who these users are.

Please synthesize:
1. The three most common pain points (with quotes supporting each)
2. Any surprises — things that contradict our current user assumptions
3. Jobs-to-be-done that came up that are not yet in our users.md

[TRANSCRIPT 1 - PRIYA SHANKAR, DATA COORDINATOR, MARCH FOUNDATION]
[paste transcript]

[TRANSCRIPT 2 - ...]
```

### Loading Competitor Pricing Pages for Analysis

Copy the full content of a competitor's pricing page into the message:

```
@context/market.md @context/product.md

Analyze the following competitor pricing page against our positioning.
I want to understand:
1. What they are emphasizing that we are not
2. Any pricing signals that tell us something about their target customer
3. Any language or framing we should respond to in our own positioning

[COMPETITOR PRICING PAGE CONTENT]
[paste full page text]
```

### Loading Metrics Data for Analysis

Export a CSV from your dashboard, then:

```
@context/data.md

I am attaching our product metrics for Q2. The metric definitions in data.md
should govern how you interpret these numbers.

Please analyze:
- Which cohorts are showing the best retention and what they have in common
- Where in the funnel we are losing the most users relative to last quarter
- What the data suggests about the success of the onboarding changes we shipped
  in week 3 of the quarter

[paste CSV data or describe the data in structured form]
```

### Loading Slack Threads for Stakeholder Synthesis

Copy a Slack thread (or use the Slack MCP integration):

```
@context/stakeholders.md

I am pasting a Slack thread from our last engineering sync below.
I need to understand:
1. What the engineering team's current top concerns are about the Q3 roadmap
2. Any blockers or dependencies they raised
3. Where their priorities seem to conflict with the roadmap we have communicated

Write this as a 200-word briefing I can bring to my next 1-on-1 with the 
engineering lead.

[SLACK THREAD]
[paste thread]
```

---

## 6. Context Hygiene: Keeping It Fresh

Context files degrade silently. Nothing breaks. Claude just starts giving you advice based on an outdated picture of your product, your users, or your market. You may not notice for weeks.

### How Often to Update Each Module

| Module | Update when | Minimum frequency |
|--------|-------------|------------------|
| `company.md` | Strategy pivots, major constraint changes | Quarterly |
| `product.md` | Major feature launches, deprecations | Monthly |
| `users.md` | After user research cycles, after significant customer feedback | Monthly |
| `market.md` | Competitor major moves, market events | Monthly |
| `data.md` | After baseline shifts, metric redefinitions | After any change to metrics |
| `decisions.md` | Every major decision | Within a week of the decision |
| `principles.md` | Rare — only when you have resolved a debate that reveals a new principle | Quarterly |
| `stakeholders.md` | After org changes, after stakeholder preference shifts | Quarterly |
| `roadmap.md` | After planning cycles, after significant pivots | Start of each quarter |
| `voice.md` | Rarely — when you deliberately shift brand voice | Bi-annually |

### Signs Your Context Is Stale

- Claude gives you advice that contradicts a decision you know you made six months ago
- Claude refers to a persona that no longer represents your actual users
- Claude recommends a competitor feature that you already built
- Claude's output sounds right but misses the current urgency or priority
- You find yourself saying "that is not quite our situation anymore" more than once per session

### The 15-Minute Weekly Context Refresh Routine

Every Monday, before starting any substantive Claude work:

1. **Open `decisions.md`.** Did you make any significant decisions last week? Add them. (3 minutes)
2. **Open `roadmap.md`.** Is anything materially different from what is written? Update status on in-progress items. (5 minutes)
3. **Open `data.md`.** Did any metrics change significantly? Update baselines. (3 minutes)
4. **Skim `users.md`.** Did anything from this week's conversations challenge your user assumptions? Make a note to update it properly if so. (2 minutes)
5. **Check `product.md`.** Did anything ship? Update current capabilities. (2 minutes)

Total: 15 minutes. This keeps your context library within a week of current.

### Using Claude to Help Update Its Own Context Files

At the end of any rich working session:

```
Based on everything we have discussed today, what did you learn about our 
product, users, or situation that should be captured in my context files?

Please give me:
1. Any new facts I should add to specific modules
2. Any existing content that this session suggests is outdated
3. A draft update for whichever module most needs it

You have access to: @context/users.md @context/product.md @context/data.md
```

This is Teresa Torres' practice: "Claude, what'd you learn today that we should document?" Every productive session becomes an opportunity to make the context library smarter.

---

## 7. Measuring Context Quality

### How to Know If Your Context Is Working

Good context has three observable effects:

1. **Fewer corrections.** You stop having to say "actually, our users are not like that" or "no, we already decided not to do that." Claude gets it right without correction.

2. **Less setup per prompt.** Your prompts get shorter because you are not re-explaining background. The context files do that work.

3. **Output is immediately usable.** You read the output and it sounds like it came from someone who knows your product, not someone who has just learned what a product is.

### The Cold Start Test

This is the definitive quality test. Open a completely fresh Claude session — no prior conversation, nothing loaded yet. Then ask a question you would normally ask mid-session:

```
What is the biggest product risk we face in the next quarter?
```

**Without context:** Claude will ask clarifying questions or give a generic "B2B SaaS risks" answer.

**With good persistent context:** Claude will answer specifically from your situation because your CLAUDE.md has already been loaded.

Run this test after you build your initial context library. If the answer feels generic, your persistent context is not doing its job.

### Red Flags in Claude's Responses That Mean Your Context Is Weak

Watch for these specific patterns:

- **"Typically, B2B SaaS companies..."** — Claude is falling back on training data because it does not have enough specific context about you.
- **"Depending on your target market..."** — Claude does not know your target market. Load `users.md` and `market.md`.
- **Suggestions that contradict your principles** — Your `principles.md` is not loaded or is too vague.
- **Wrong terminology** — Your `voice.md` is not loaded. Claude is using its default vocabulary, not yours.
- **Generic personas** — "Your power users" and "casual users" instead of your actual personas. Load `users.md`.

### Iterating to Better Context

Context engineering is iterative. After any session where you found Claude's output mediocre:

1. Identify the gap. What did Claude not know that, if it had known, would have produced better output?
2. Find where that gap belongs. Which context module should contain that information?
3. Add it. Concisely.
4. Test it. Ask the same question in a fresh session.

Your context library gets better through use, not through planning. The first version will be imperfect. That is fine.

---

## 8. Advanced: Context for Different PM Modes

Different types of product work need different context. Load the right modules for the mode you are in.

### Strategy Mode

You are doing strategic planning, OKR setting, or big-picture positioning work.

**Load:** `company.md`, `market.md`, `decisions.md`, `data.md`, `roadmap.md`

**Sample prompt:**

```
@context/company.md @context/market.md @context/decisions.md 
@context/data.md @context/roadmap.md

I need to set our H2 product strategy. Given everything you know about our 
situation, identify:
1. The two or three biggest opportunities we are under-investing in
2. The bets in our current roadmap most at risk of being wrong
3. The questions we need to answer before locking in H2 commitments

Be direct. Challenge the assumptions in our current roadmap.
```

### Discovery Mode

You are in user research mode — planning interviews, synthesizing findings, generating hypotheses.

**Load:** `users.md`, `product.md`, `data.md`

**Sample prompt:**

```
@context/users.md @context/product.md @context/data.md

I am planning a round of discovery interviews focused on the onboarding experience.
Based on what you know about our users and the current product state:

1. What are the highest-priority hypotheses to test?
2. What do we currently believe about our Priya persona that most needs validation?
3. Draft an interview guide — 5 questions max, open-ended, no leading questions.

These interviews are with first-time users who ran their first pipeline in the
last two weeks.
```

### Execution Mode

You are in sprint planning, writing specs, creating Jira tickets, or doing delivery work.

**Load:** `product.md`, `users.md`, `principles.md`, `roadmap.md`

**Sample prompt:**

```
@context/product.md @context/users.md @context/principles.md @context/roadmap.md

I need a spec for the following feature: [feature description]

Use our standard PRD format:
- Problem statement (from the user's perspective)
- Success criteria (observable outcomes, not metrics)
- Scope: what is in, what is out
- Key decisions required
- Risks and dependencies

Flag any conflict with our principles or current roadmap priorities.
```

### Communication Mode

You are writing a stakeholder update, preparing for a review, or synthesizing for leadership.

**Load:** `stakeholders.md`, `voice.md`, `roadmap.md`, `data.md`

**Sample prompt:**

```
@context/stakeholders.md @context/voice.md @context/roadmap.md @context/data.md

I need to write a monthly product update for our NGO partners.

These are non-technical users. They care about: impact, reliability, what is
coming next, and any issues that might affect them.

Draft a 300-word update covering:
- What we shipped in the past month
- The one or two things they should notice working differently
- What is coming next
- Anything they need to do or know

Use our voice: direct, warm, no jargon. If you need to mention something 
technical, translate it.
```

---

## 9. Complete Worked Example

Let us build a context library from scratch for a fictional PM. Her name is Ananya. She is the sole product manager at Groundwork, a B2B SaaS company that helps impact-first small businesses measure and report their social impact to investors and grant committees. She has used Claude for six months and keeps getting outputs that feel "right but not quite."

### Step 1: Starting Fresh (No Context)

Ananya opens a session and asks:

```
Write me a one-pager on why impact investors should care about Groundwork.
```

**Claude's response (no context):**

```
Why Impact Investors Should Care About [Company]

In today's landscape, impact investors face increasing pressure to demonstrate the
real-world outcomes of their capital deployment. [Company] provides the tools and
frameworks that allow portfolio companies to measure, track, and communicate their
social impact with the rigor that institutional investors require.

Our platform offers:
• Standardized impact frameworks aligned with IRIS+ and B Corp metrics
• Automated data collection from portfolio companies
• Real-time dashboards for fund-level reporting
• Audit-ready documentation for LPs and stakeholders

For impact investors managing multiple portfolio companies, [Company] eliminates the
manual survey-and-spreadsheet workflows that currently consume weeks of time each
quarter. [...]
```

Technically correct. Completely generic. Ananya would have to rewrite 80% of this to make it usable.

### Step 2: Building the Context Library

Ananya spends 90 minutes building her context library. Here are the key files:

**`context/company.md`:**

```markdown
# Company Context

## Who We Are
Groundwork is a B2B SaaS tool that helps impact-first small businesses (fewer than
50 employees) collect, track, and report their social impact data to investors
and grant committees.

**Founded:** 2022
**Stage:** Seed ($1.8M raised)
**Team size:** 9 (3 engineering, 2 product/design, 3 go-to-market, 1 ops)
**Geography:** US-based, English only

## Mission
Make rigorous impact reporting accessible to small mission-driven businesses that
cannot afford a dedicated impact analyst.

## Current Strategy (valid through Q3 2026)
Win the "pre-B Corp" segment — companies in the 1-3 year range that want B Corp
certification but feel overwhelmed by the process. Groundwork becomes the onboarding
path to B Corp.

## Fixed Constraints
- We do not serve individual investors or nonprofits (different compliance needs)
- No enterprise sales motion — no contracts over $500/month
- Self-serve only: no implementation services, no onboarding calls

## Business Model
SaaS. $149/month or $1,199/year. No freemium.
```

**`context/users.md`:**

```markdown
# User Context

## Persona 1: Jordan — The Founder Who Wants to Prove It

**Who they are:** Jordan is the founder of a 20-person catering company that
donates 5% of every order to food security nonprofits. They did it because it felt
right. Now their angel investor wants to see "impact data" in the quarterly report
and Jordan does not know where to start.

**Job to be done:** When a funder asks for impact data I cannot fake, I want to
collect and report real numbers without becoming a data analyst, so I can prove that
my model works and raise my next round.

**Their actual vocabulary:**
- They say "impact numbers" or "our impact stats", not "KPIs" or "metrics"
- They say "grant application" not "grant proposal"
- They say "investors" for everyone who gives them money, whether equity or grant

**Biggest frustrations:**
- "I know we are doing good. I just cannot prove it."
- "My accountant handles financial reporting. Who handles impact reporting?"
- "Every funder asks for different things. It is exhausting."

**What success looks like to them:**
A report they can send with confidence, knowing the numbers are right.

## Key User Insight
Our users do not think of themselves as data people and feel mild shame about
not being more sophisticated. Design and language must never make them feel dumb.
```

### Step 3: Running the Same Prompt With Context

```
@context/company.md @context/users.md

Write me a one-pager on why impact investors should care about Groundwork.
The audience is a seed-stage impact investor considering Groundwork for their
portfolio company recommendations.
```

**Claude's response (with context):**

```
Why Your Portfolio Companies Need Groundwork

Your portfolio companies are doing real work. But when you ask them for impact data,
you get one of three things: a spreadsheet cobbled together the night before your
call, a set of numbers no one can explain, or an apology.

This is not a measurement problem. It is an infrastructure problem. Small
mission-driven businesses — the 5-50 person operators who make up most impact
portfolios — have no dedicated analyst and no system for collecting impact data
consistently. They report what they can, when they can.

Groundwork fixes this at the company level so you do not have to fix it at the
portfolio level.

What it does: Groundwork gives founders a simple way to define their impact metrics
once (aligned to B Corp and IRIS+ standards), collect that data automatically
from their existing tools, and produce investor-ready reports on demand.

What changes for you: Portfolio companies stop treating impact data as a quarterly
scramble. They start treating it as an ongoing practice. Your reporting becomes
consistent, comparable, and defensible.

Who it is for: Pre-B Corp companies in the 1-3 year range. Companies that want to
do this right but cannot justify a full-time analyst.

Pricing: $149/month. No implementation, no onboarding calls, no contract.

The ask: Share Groundwork with your portfolio companies. We offer a 60-day trial
for new portfolio companies referred by investors. If their numbers improve, yours do.
```

This is immediately usable. Ananya estimates she would have spent 45 minutes getting to this quality without context. With context, she spent zero minutes — she just read it and sent it.

### Step 4: The Compounding Effect

After one week of using her context library, Ananya has a session where Claude references a competitor she had mentioned in `market.md` and correctly notes that it targets enterprises, not small businesses like Groundwork's users. She did not prompt for that — Claude inferred it from context.

After one month, Ananya runs the cold start test. She opens a fresh session and asks: "What is our biggest product risk this quarter?" Claude answers in terms of the B Corp certification pipeline, the specific concerns about the IRIS+ framework mapping, and the onboarding abandonment problem she had documented in `data.md`.

The context library has paid back its 90-minute construction cost many times over.

---

## 10. Templates: Copy-Paste Ready Context Files

Below are clean, blank versions of all 10 modules. Replace the bracketed content with your specifics.

### Master Index: `context/index.md`

Use this as your routing file. Put it in your CLAUDE.md as: `For detailed context on specific topics, see @context/index.md`.

```markdown
# Context Library Index

| Module | File | Last updated | Contents |
|--------|------|-------------|---------|
| Company | @context/company.md | [date] | Mission, strategy, constraints |
| Product | @context/product.md | [date] | Current product, scope, decisions |
| Users | @context/users.md | [date] | Personas, JTBD, pain points |
| Market | @context/market.md | [date] | Competitors, positioning |
| Data | @context/data.md | [date] | Key metrics, definitions, baselines |
| Decisions | @context/decisions.md | [date] | Key decisions and rationale |
| Principles | @context/principles.md | [date] | Product principles |
| Stakeholders | @context/stakeholders.md | [date] | Who matters, their goals |
| Roadmap | @context/roadmap.md | [date] | Current roadmap, bets |
| Voice | @context/voice.md | [date] | Writing style, tone, terminology |
```

### Lean CLAUDE.md (What Goes in Your Root File)

```markdown
# [Your Name] — Product Context

## Who I Am
I am the [role] at [company]. I work primarily on [product/domain].

## How to Use My Context Library
For detailed context on specific topics, see @context/index.md.

When helping with product work, default to:
- @context/users.md for anything involving user decisions
- @context/product.md for anything about what we build
- @context/voice.md for all writing tasks

## My Working Style
- I prefer direct feedback over diplomatic hedging
- Lead with the answer, then the reasoning
- Flag conflicts with my principles explicitly
- Challenge my assumptions — do not just confirm them

## Quick Reference
**Product:** [One sentence]
**Users:** [One sentence]  
**Biggest current problem:** [One sentence]
```

### All 10 Module Blank Templates

```markdown
# company.md

## Who We Are
[Company name] is a [description].

**Founded:** [Year]
**Stage:** [Stage]
**Team size:** [Number]
**Geography:** [Where]

## Mission
[One sentence]

## Current Strategy (valid through [quarter/year])
[Two to three sentences]

## Fixed Constraints
- [Constraint 1]
- [Constraint 2]

## Business Model
[How you make money or sustain yourselves]

## What Success Looks Like in 12 Months
[Two to three observable outcomes]
```

```markdown
# product.md

## What [Product] Is
[One paragraph]

**Current version:** [X.X]
**Status:** [Alpha / Beta / GA]

## Core Capabilities
- [Capability 1]
- [Capability 2]
- [Capability 3]

## Explicit Non-Scope
- [Non-scope 1]
- [Non-scope 2]

## Key Product Decisions That Define Its Shape
- [Decision 1]
- [Decision 2]

## Current Health Assessment
[Two to three honest sentences]
```

```markdown
# users.md

## Persona 1: [Name] — [Role]

**Who they are:** [Two sentences]

**Job to be done:**
When [situation], I want to [motivation], so I can [outcome].

**Their actual vocabulary:**
- They say "[term A]", never "[term B]"

**Biggest frustrations:**
- "[Quote 1]"
- "[Quote 2]"

**What success looks like to them:**
[One sentence]

---

## Persona 2: [Name] — [Role]
[Same structure]

## Jobs We Are NOT Hired For
- [Non-job 1]

## Key User Insight
[One sentence]
```

```markdown
# market.md

## Competitive Set

### [Competitor 1]
- **What they do:** [One sentence]
- **Their positioning:** [How they describe themselves]
- **Why users choose them:** [Honest]
- **Why users choose us instead:** [Honest]
- **Their price point:** [Rough]

## The Status Quo Competitor
Many users are choosing between us and [Excel / manual process / consultants].

## Our Positioning
[One paragraph]

## Where We Win
[Specific situations]

## Where We Lose
[Honest]
```

```markdown
# data.md

## North Star Metric
**[Metric]:** [Definition]
**Current:** [X]  **Target:** [Y by date]

## Key Product Metrics

| Metric | Definition | Current | Target |
|--------|-----------|---------|--------|
| [Metric 1] | [Definition] | [X] | [Y] |
| [Metric 2] | [Definition] | [X] | [Y] |

## Precise Definitions
- **[Term 1]:** [Exact definition]
- **[Term 2]:** [Exact definition]

## Anti-Metrics
- We do not optimize [X] because [reason]

## Known Data Gaps
- [Gap 1]
```

```markdown
# decisions.md

## [Decision 1 Title] (Made: [Month Year])

**What we decided:** [One sentence]

**Why:** [Two to three sentences]

**What this constrains:** [What this closes off]

**Trigger to revisit:** [What would have to be true]

---

## Open Questions (Not Yet Decided)
- [Question 1]
- [Question 2]
```

```markdown
# principles.md

## [Principle 1 — stated as a directive]

**The tension it resolves:** [What two things are in conflict]

**In practice:**
- We do [X] even when tempted to do [Y]
- Example: [Past decision this resolved]

**Not to be confused with:** [Common misapplication]

---

## Non-Negotiables
- [Non-negotiable 1]
- [Non-negotiable 2]
```

```markdown
# stakeholders.md

## Internal Stakeholders

### [Name / Role]
- **What they care about:** [Their actual goals]
- **Communication style:** [How they like to receive information]
- **Typical concerns:** [What they push back on]
- **What earns their trust:** [What kind of evidence moves them]

---

## External Stakeholders

### [Organization / Type]
- **Their relationship to us:** [One sentence]
- **What they need from us:** [Specific output]
- **Their constraints:** [Budget, timeline, technical]

---

## Decision Authority Map

| Decision type | Authority | Who consults |
|--------------|-----------|-------------|
| [Type 1] | [Person] | [Who] |
```

```markdown
# roadmap.md

## Current Quarter: [Q X Year]

### In Progress
- **[Initiative 1]:** [One sentence]
  - Status: [Status]
  - Assumption: [What must be true]

### Committed (Next Quarter)
- **[Initiative 2]:** [One sentence]
  - The bet: [Why we believe this]
  - Key risk: [What could make this wrong]

## Explicit Deprioritizations
- [Thing 1] — because [reason]

## Open Assumptions
- [Assumption 1]
```

```markdown
# voice.md

## Tone
- **[Adjective 1]:** We are [X], meaning [specific behavior].
- **[Adjective 2]:** [Same]

## Always
- [Positive instruction 1]
- [Positive instruction 2]

## Never
- Never use "[banned phrase]"
- Never [behavior]

## Terminology
| Use | Never use | Why |
|-----|-----------|-----|
| [Term] | [Alternative] | [Reason] |

## Format Preferences
- **Documents:** [Structure preference]
- **Emails:** [Length/structure]
- **Slack:** [Style]

## Example of Our Standard (pattern-match to this):
[Paste a real paragraph that represents your voice at its best]
```

---

## Getting Started: Your 2-Hour Bootstrap

You do not need all 10 modules to start. Build them in this order:

**Hour 1: The essentials (biggest impact first)**
1. `company.md` — 20 minutes
2. `users.md` — 25 minutes
3. `voice.md` — 15 minutes

**Hour 2: The decision infrastructure**
4. `product.md` — 15 minutes
5. `decisions.md` — 20 minutes
6. Lean `CLAUDE.md` that imports them — 10 minutes
7. Cold start test — 5 minutes

Run the cold start test before building the remaining four modules. If you feel the difference after just these six, you will be motivated to finish.

Build the rest over the following two weeks, one module per session. By the end, every Claude session will start with Claude already knowing what took you two paragraphs to explain before.

---

*Sources used in the research for this guide:*

- [Teresa Torres — Claude Code for Product Managers (Lenny's Newsletter)](https://www.lennysnewsletter.com/p/claude-code-for-product-managers)
- [How I AI: Teresa Torres's Claude Code System (ChatPRD)](https://www.chatprd.ai/how-i-ai/teresa-torres-claude-code-obsdian-task-management)
- [CLAUDE.md for Product Managers (CCforPMs)](https://ccforpms.com/fundamentals/project-memory)
- [Context Engineering: The Next Frontier (deepset)](https://www.deepset.ai/blog/context-engineering-the-next-frontier-beyond-prompt-engineering)
- [Context Engineering for Product Managers (Medium)](https://medium.com/@rakesh.malloju/context-engineering-for-product-managers-the-next-big-10x-skill-38de541e8b9b)
- [Referencing Files in Claude Code (Steve Kinney)](https://stevekinney.com/courses/ai-development/referencing-files-in-claude-code)
- [CLAUDE.md Guide (Hannah Stulberg)](https://hannahstulberg.substack.com/p/claude-code-for-everything-the-best-personal-assistant-remembers-everything-about-you)
- [Claude Code for Product Managers (Builder.io)](https://www.builder.io/blog/claude-code-for-product-managers)
- [Context Engineering Guide (Prompting Guide)](https://www.promptingguide.ai/guides/context-engineering-guide)
- [PM Claude Code Setup (aakashg GitHub)](https://github.com/aakashg/pm-claude-code-setup)
