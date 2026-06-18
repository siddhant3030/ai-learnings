## 1. PM Workflow Mapping

This section maps every major PM activity to what Claude actually does in practice, drawing on practitioner reports and production deployments.

### 1.1 Customer Discovery

**What PMs do:** Design interview scripts, conduct research sessions, synthesize findings into opportunity clusters.

**How Claude helps:**
- Generate targeted interview scripts from a problem statement or hypothesis in under 2 minutes
- Synthesize 10-50 transcripts into JTBD (Jobs to Be Done) force maps and opportunity clusters
- Distinguish between what users say they want vs. what their behaviors reveal they need

**Example prompt:**
```
<role>You are a user research analyst synthesizing qualitative interview data for product decisions.</role>
<context>Product: [description]. Target users: [persona]. Research objective: [goal].</context>
<transcripts>[Paste transcripts labeled by participant ID]</transcripts>
<task>Synthesize findings. For each finding: (1) state the insight in one sentence; (2) categorize as pain point, feature request, behavioral pattern, or emotional response; (3) note how many participants expressed this; (4) include 1-2 direct quotes; (5) rate severity 1-5. Distinguish between stated preferences and revealed behavior. Flag gaps this data cannot answer.</task>
<output_format>Group by category ranked by severity. End with Research Gaps section.</output_format>
```

**Time saved:** Teresa Torres (Product Talk) reports that what used to be a half-day of synthesis now takes 20 minutes, allowing 3 rounds of discovery in the time previously spent on one. Source: [Lenny's Newsletter — Claude Code for product managers](https://www.lennysnewsletter.com/p/claude-code-for-product-managers)

**Limitations:**
- AI summaries can miss 20-40% of nuanced detail (Torres warns against pure AI synthesis without human validation)
- Empathy and live rapport-building remain irreplaceable in actual interview sessions
- Claude cannot observe non-verbal cues or probe follow-up in real time

**Best practices:**
- Always have humans validate clustering outputs against raw transcripts
- Use Claude to augment individual analysis, then compare notes across team members
- Maintain consistent transcript format (participant profile, notes, full transcript) to enable reliable cross-interview querying

---

### 1.2 User Research

**What PMs do:** Analyze survey data, review support tickets, process NPS comments, monitor app store reviews.

**How Claude helps:**
- Process bulk feedback from CSV exports into thematic clusters with frequency counts
- Generate severity/frequency matrices for bug reports or feature requests
- Build cross-segment analysis dashboards as interactive HTML from structured data

**Example workflow (from elsevanderberg.substack.com):**

Three query modes for 50 interview transcripts:

1. **Conversational:** "What's the most common pain point across all interviews?" and "Give me all quotes around [topic]."

2. **Tabular/CSV extraction:** "Go through all interview files in Research/interviews/. For each, extract: Name, Role, Seniority, Company Type, Region, Main Friction (categorized), Severity (low/medium/high/critical), Magic Wand Wish. Output as CSV."

3. **Visual dashboard:** "Create an interactive HTML dashboard from the CSV with horizontal bar charts for categorical data, filter dropdowns for seniority/company type/region, and a data table."

Source: [Claude Code + 50 interview transcripts: three ways I query the data](https://elsevanderberg.substack.com/p/claude-code-for-interview-synthesis)

**Time saved:** Full-day synthesis → 20-30 minutes

**Limitations:** Claude cannot conduct research; it can only analyze what you give it. Garbage-in, garbage-out applies strongly.

---

### 1.3 Competitive Analysis

**What PMs do:** Monitor competitor positioning, pricing, features, job listings, and messaging.

**How Claude helps:**
- Build structured competitive briefs from homepage, pricing, and feature page copy
- Track changes across competitor websites weekly using automated snapshot-and-diff workflows
- Map feature comparison matrices and positioning gaps

**Example prompt chain (from Evelance.io):**

*Prompt 1 — Positioning extraction:*
```
<task>Based on the competitor materials below, extract for each competitor: (1) their implied target user; (2) core value proposition; (3) top 3 differentiators; (4) pricing structure and market positioning.</task>
<competitor_data>Competitor A: [website copy, feature page, pricing page]. Competitor B: [same].</competitor_data>
<constraints>Use only the provided materials. Note ambiguities explicitly.</constraints>
```

*Prompt 2 — Gap mapping:*
```
<task>Using the competitor analysis and my product information below, produce: Overlap zones (same need, same way); Differentiation zones (where I serve needs competitors don't); White space (needs neither serves well).</task>
<my_product>[positioning, feature list, pricing]</my_product>
```

*Prompt 3 — Positioning strategy:*
```
<task>Based on the analysis, recommend: (1) Two areas to compete aggressively; (2) One area to concede; (3) One white space worth exploring. For each, explain the strategic logic and primary risk.</task>
```

**Automated monitoring architecture** (from Crustdata/Claude integration):
- Configure Claude Desktop with Crustdata MCP server (no coding required)
- Query: "Enrich acme-corp.com. Include headcount, funding, founders, C-suite, job openings, web traffic, SEO data."
- Set Watcher API alerts for headcount changes, funding announcements, executive transitions
- Cost: single-digit dollars per research cycle vs. $20,000-40,000/year for traditional platforms

Source: [How to Automate Competitive Intelligence with Claude](https://crustdata.com/blog/how-to-automate-competitive-intelligence-with-claude)

---

### 1.4 Product Strategy

**What PMs do:** Develop product vision, OKRs, prioritization frameworks, positioning, and strategic bets.

**How Claude helps:**
- Critique strategy drafts using frameworks like Porter's Five Forces, Blue Ocean, Jobs-to-be-Done
- Generate strategic alternatives and stress-test assumptions
- Build structured TAM models with scenario analysis

**Strategy assumption testing prompt:**
```
<strategy>[Paste your strategy document]</strategy>
<task>Act as a critical evaluator. Identify the 3 strongest assumptions this strategy depends on. For each: (1) explain what it assumes is true; (2) describe the specific failure scenario; (3) propose a low-cost test (under 2 weeks, under $5,000) to validate it before committing resources; (4) rate as high/medium/low risk based on how much the strategy depends on it AND how much evidence currently supports it.</task>
```

**Time saved:** Cat Wu (Head of Product, Claude Code at Anthropic) reports spending 30-40% less time in strategy drafting cycles by using Claude.ai as a strategic thinking partner. Source: [Product management on the AI exponential](https://claude.com/blog/product-management-on-the-ai-exponential)

**Limitations:** Claude lacks live market data, cannot talk to customers, and may replicate common strategic frameworks without surfacing genuinely contrarian insights.

---

### 1.5 Roadmapping

**What PMs do:** Prioritize features, sequence delivery, communicate timelines, manage scope.

**How Claude helps:**
- Score features using RICE, ICE, weighted scoring
- Sequence items given capacity constraints and dependencies
- Generate Now/Next/Later roadmap documents from a prioritized list

**RICE scoring prompt:**
```
<task>Score features using RICE (Reach × Impact × Confidence / Effort). Calculate scores. Flag any feature where Confidence < 50% and recommend what data would raise it. Note features where a small variable change would significantly shift rankings.</task>
<features>[Feature 1: Reach: X, Impact: X, Confidence: X%, Effort: X person-months]</features>
<output_format>Ranked table with columns: Feature, Reach, Impact, Confidence, Effort, RICE Score, Flags. Followed by a sensitivity analysis paragraph.</output_format>
```

**Quarterly roadmap prompt:**
```
<inputs>
Prioritized feature list: [paste]
Team capacity: [person-weeks]
Hard constraints: [deadlines, dependencies]
Strategic priorities: [top 3 goals]
</inputs>
<task>Structure a quarterly roadmap: (1) Group into 2-4 themes aligned with strategic priorities; (2) Sequence by dependencies (flag circular ones); (3) Assign to months with capacity validation; (4) Identify 3 highest-risk items with contingencies; (5) List deferred items with explanations.</task>
<output_format>Now/Next/Later framework. Follow with risk analysis.</output_format>
```

---

### 1.6 PRD Creation

**What PMs do:** Document feature requirements, user stories, acceptance criteria, technical constraints.

**How Claude helps:**
- Generate first-draft PRDs from a problem statement + context in under 5 minutes
- Ensure acceptance criteria are testable, not vague
- Adapt PRD content for different audiences (engineering vs. design vs. leadership)

**PRD generation prompt:**
```
<role>You are writing a PRD for an engineering team that will build from this document. Precision matters more than comprehensiveness. Every section must be specific enough that an engineer can begin without follow-up questions.</role>
<context>Product: [name+description]. Target user: [persona]. Business objective: [success metric]. Research summary: [key findings]. Technical constraints: [limitations].</context>
<task>Write a PRD: (1) Problem statement with evidence; (2) Proposed solution (functional description); (3) User stories with acceptance criteria using INVEST; (4) Scope definition with explicit exclusions; (5) Success metrics with specific targets and measurement methods; (6) Edge cases and error states (minimum 5); (7) Dependencies and risks; (8) Open questions with owners.</task>
<quality_bar>Acceptance criteria must be testable. Success metrics must include number, timeframe, and measurement method. Flag any section where you lack information as [NEEDS INPUT] rather than filling with assumptions.</quality_bar>
```

**Test results:** In a head-to-head test comparing Claude (free), Cursor, and ChatPRD ($10/month) across 7 evaluation criteria, Claude scored 91% vs. 86% for Cursor and 77% for ChatPRD. Source: [Writing PRDs in the Age of AI: I Tested 3 Methods So You Don't Have To](https://prompttoproduct.substack.com/p/writing-prds-in-the-age-of-ai-i-tested)

**Time saved:** PRD creation from 4-6 hours to 30-45 minutes for a first draft.

---

### 1.7 Stakeholder Communication

**What PMs do:** Write status updates, executive summaries, engineering briefs, release notes.

**How Claude helps:**
- Generate audience-tailored updates from a single set of raw notes
- Draft one core product update, then adapt into: technical brief, sales enablement summary, executive update, customer-facing release note — four versions from one source

**Audience-aware status update prompt:**
```
<raw_notes>[Unstructured notes: progress, blockers, decisions made, open questions, metrics changes]</raw_notes>
<audience>[executive | engineering | cross-functional]</audience>
<task>
If executive: 200 words. Lead with status (on track/at risk/blocked). Highlight 1-2 decisions needing executive input. Include key metric changes. Skip implementation details.
If engineering: 300 words. Lead with blockers and dependencies. Include technical decisions and rationale. Note scope changes. Include upcoming milestones.
If cross-functional: 250 words. Lead with progress summary. Include specific asks from other teams with deadlines. Note timeline changes and their impact.
</task>
<constraints>No filler phrases. Every sentence must contain information the reader doesn't already know. If a section has no updates, say "No changes" rather than padding.</constraints>
```

---

### 1.8 Data Analysis

**What PMs do:** Analyze product funnels, A/B test results, cohort data, NPS scores.

**How Claude helps:**
- Process CSV files into funnel analysis with hypothesis generation for drop-off points
- Run A/B test statistical analysis with practical significance calculations
- Generate narrative metric summaries that explain changes, not just report them

**Funnel analysis prompt:**
```
<context>Product: [description]. Funnel: [e.g., signup to first value moment].</context>
<funnel_data>[Step | Users Entering | Users Completing table]</funnel_data>
<task>(1) Calculate conversion rates between each step; (2) Identify the step with largest absolute AND percentage drop-off; (3) Generate 5 hypotheses for why users drop off at the highest-drop step; (4) For each hypothesis, propose a specific 2-week test; (5) Recommend which to test first with reasoning.</task>
```

**A/B test analysis prompt:**
```
<test_data>
Test name: [description]
Control: [sample size, conversion rate]. Variant: [sample size, conversion rate]. Duration: [days]. Primary metric: [what you measured]. Segments: [if available]
</test_data>
<task>(1) Calculate statistical significance at 95% confidence with math shown; (2) Calculate practical significance in business terms; (3) Check for Simpson's paradox across segments; (4) Recommend: ship / extend (specify duration) / kill — with reasoning.</task>
<constraints>Do not recommend shipping based solely on statistical significance if practical impact is negligible.</constraints>
```

---

### 1.9 Experiment Design

**What PMs do:** Design A/B tests, hypothesis validation experiments, usability studies.

**How Claude helps:**
- Generate structured experiment plans from hypotheses
- Calculate required sample sizes
- Design guardrail metrics to prevent shipping regressions

**Time saved:** Experiment design documentation from 2-3 hours to 20-30 minutes.

---

### 1.10 Launch Planning

**What PMs do:** Coordinate cross-functional launch plans, write release notes, prep support materials.

**How Claude helps:**
- Generate tiered release notes (internal changelog, customer-facing, executive summary)
- Draft launch checklists across teams
- Convert engineering commit logs into user-benefit language

**Release notes prompt:**
```
<inputs>[Engineering changelog or list of shipped changes]</inputs>
<audience>End users. Technical literacy: low to moderate.</audience>
<task>Rewrite as customer-facing release notes. Translate technical descriptions to user-benefit language. Group by theme (new features, improvements, fixes). Maximum 2 sentences per item. Open with the most impactful change.</task>
```

---
