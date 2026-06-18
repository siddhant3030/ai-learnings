# Research Automation for Product Managers
### A hands-on workshop you can implement in an afternoon

---

## 1. What PM research automation actually means

Most PMs spend 5–10 hours a week on research tasks that follow a fixed pattern: collect information from many places, read it, extract the signal, write a structured summary, share it. Every step in that pattern except "share it" is something Claude Code can do faster and more consistently than a human.

The three categories worth automating:

**Competitive intelligence** — tracking what competitors are doing, how they are positioning, where they are investing. This is the highest-value automation because it is purely a data-aggregation-and-synthesis problem. The information is all public; the bottleneck is human time.

**User research synthesis** — turning transcripts, notes, and recordings into structured insights. This is high-value but requires a judgment call at the end: you still decide which themes to act on. Claude does the extraction; you do the prioritization.

**Market and trend monitoring** — industry news, regulatory changes, analyst reports, adjacent-market moves. Mostly signal-to-noise filtering. Claude learns what matters for your specific market and surfaces only what passes the filter.

### What can be fully automated vs. what needs human judgment

| Fully automated | Needs human judgment |
|-----------------|----------------------|
| Data collection from public sources | Deciding which competitor moves are actually threats |
| Extracting quotes from transcripts | Deciding which user pain points to prioritize |
| Spotting new patterns in feedback | Setting the product strategy that responds to trends |
| Generating first-draft briefs | Validating that a "pattern" isn't sampling artifact |
| Formatting structured outputs | Communicating findings to stakeholders |

### The ROI

A professional competitive intelligence subscription (Crayon, Klue, Kompyte) costs $20,000–$40,000 per year for a small team. A well-configured Claude Code system using Firecrawl MCP costs roughly $10–$30 per month in API credits at current pricing. The gap is not about quality — it is about whether you are willing to spend two hours configuring a system rather than buying a SaaS license.

### What you will build in this guide

1. A weekly competitive brief system (`/compete-brief` slash command)
2. A user interview synthesis pipeline (`/synthesize-interviews` slash command)
3. A daily market digest
4. A customer feedback synthesis system (`/feedback-synthesis` slash command)
5. A queryable research repository

---

## 2. Automated Competitive Intelligence System

### 2a. What to monitor

For each competitor, monitor these source types. Prioritize based on how frequently they publish signal-rich content.

**High signal, monitor weekly:**
- Pricing page (URL snapshot + diff)
- Feature/product changelog or release notes
- G2 and Capterra reviews (new reviews reveal real user pain)
- Job postings — 10+ ML/AI engineering roles means a product bet; a sudden wave of sales roles means they are going up-market

**Medium signal, monitor bi-weekly:**
- Blog posts and case studies (reveals positioning shifts)
- LinkedIn company updates
- Press releases and funding announcements

**Lower signal, monitor monthly:**
- Twitter/X (mostly noise; useful during product launches)
- Patent filings (signals technical direction 12–18 months out)
- Conference talks and webinars

Job postings deserve special attention. When a competitor opens 5+ roles for "Data Platform Engineer" or "Enterprise Integration Lead," that is a 3–6 month early warning on their roadmap. This is free information most PMs ignore.

### 2b. Setting up Firecrawl MCP

Firecrawl converts any web page into clean markdown, strips ads and navigation, and returns structured data your prompts can work with. It is the right tool for competitive monitoring because it handles JavaScript-rendered pages that plain `curl` cannot read.

**Step 1: Get a Firecrawl API key**

Go to [firecrawl.dev](https://firecrawl.dev), create an account. The free tier gives you 500 credits, sufficient for a month of light monitoring.

**Step 2: Add Firecrawl to Claude Code**

```bash
claude mcp add firecrawl-mcp \
  -e FIRECRAWL_API_KEY=fc-YOUR-API-KEY-HERE \
  -- npx -y firecrawl-mcp
```

Or add it manually to `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "firecrawl-mcp": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": {
        "FIRECRAWL_API_KEY": "fc-YOUR-API-KEY-HERE"
      }
    }
  }
}
```

**Step 3: Verify**

Start a Claude Code session and test:

```
> Scrape https://competitor.com/pricing and summarize the plans
```

If Firecrawl is working, Claude will return clean content without raw HTML noise.

**Available Firecrawl tools (what Claude can call):**

| Tool | What it does | When to use |
|------|-------------|-------------|
| `scrape` | Single URL → clean markdown | Pricing pages, specific blog posts |
| `search` | Web search + optional scrape | Finding recent news about a competitor |
| `crawl` | Multi-page crawl | Mapping all changelog entries |
| `map` | Discover all URLs on a domain | Finding hidden pricing tiers, docs |
| `extract` | Structured JSON extraction | When you want data in a specific schema |
| `batch_scrape` | Multiple URLs in parallel | Scraping 5 competitors at once |

### 2c. The weekly competitive brief — `/compete-brief`

Create this file at `.claude/commands/compete-brief.md`:

````markdown
---
description: Generate weekly competitive intelligence brief from all tracked competitors
---

You are a competitive intelligence analyst. Generate a weekly brief covering all
tracked competitors. Today is $CURRENT_DATE.

## Step 1: Load competitor list

Read the file `research/competitive/competitors.yaml` to get the list of competitors
and their key URLs to monitor.

## Step 2: Collect data

For each competitor in the list, use Firecrawl to:
1. Scrape their pricing page (if URL provided)
2. Scrape their changelog or "what's new" page (if URL provided)
3. Search the web for "[competitor name] news OR announcement OR launch" in the last 7 days
4. Scrape any blog posts or press releases found in the search

Collect all content. Note which competitor each piece of content belongs to.

## Step 3: Analyze for signals

For each competitor, identify and flag:
- **New features or product launches** — what capability did they add?
- **Pricing changes** — any new tiers, price increases, feature gates?
- **Positioning shifts** — new messaging, tagline, ICP language on homepage?
- **Hiring signals** — new roles that reveal investment areas (check LinkedIn and their careers page)
- **Customer sentiment** — new G2/Capterra reviews with strong language about what users love or hate
- **Strategic moves** — partnerships, integrations, market expansions, executive hires

## Step 4: Write the brief

Output a structured brief in this exact format:

---

# Weekly Competitive Brief — [DATE]

## Executive Summary
[2–3 sentences: what matters most this week]

## Competitor Updates

### [Competitor Name]
**Signal level:** [High / Medium / Low]
**Summary:** [2–3 sentence summary of this week's activity]

**What changed:**
- [Specific change with source URL]
- [Specific change with source URL]

**Why it matters:**
[1–2 sentences connecting this to your product or positioning]

**Questions this raises:**
- [Strategic question for the product team]

[Repeat for each competitor]

## Cross-Competitor Themes
[Patterns appearing across multiple competitors this week — e.g., "three competitors added AI copilot features", "two competitors shifted messaging toward enterprise"]

## Recommended Actions
- [Specific action for product team]
- [Specific action for marketing/positioning]

---

Save the completed brief to `research/competitive/briefs/YYYY-MM-DD.md`.

````

**The competitors.yaml file** — create at `research/competitive/competitors.yaml`:

```yaml
competitors:
  - name: "Acme Analytics"
    website: "https://acmeanalytics.com"
    pricing_url: "https://acmeanalytics.com/pricing"
    changelog_url: "https://acmeanalytics.com/changelog"
    g2_url: "https://www.g2.com/products/acme-analytics/reviews"
    careers_url: "https://acmeanalytics.com/careers"
    notes: "Main competitor in mid-market. Watch for enterprise tier moves."

  - name: "DataFlow Pro"
    website: "https://dataflowpro.io"
    pricing_url: "https://dataflowpro.io/pricing"
    changelog_url: "https://dataflowpro.io/updates"
    g2_url: "https://www.g2.com/products/dataflow-pro/reviews"
    careers_url: "https://dataflowpro.io/jobs"
    notes: "VC-backed. Moving up-market. Watch headcount growth."

your_product:
  name: "Dalgo"
  differentiators:
    - "Open source"
    - "NGO/social sector focus"
    - "Low-cost total ownership"
    - "No-code pipeline builder"
  icp: "Non-technical NGO program managers and data coordinators"
  watch_for:
    - "Any competitor targeting the social sector or NGO segment"
    - "Open source alternatives launching"
    - "Pricing drops that close the cost gap"
```

**Run the command:**

```
> /compete-brief
```

### 2d. Deep-dive analysis template

When a competitor makes a significant move (funding round, major product launch, pricing overhaul), use this prompt for a deeper investigation. Save as `.claude/commands/compete-deep-dive.md`:

````markdown
---
description: Deep competitive analysis on a specific competitor or event
argument-hint: "<competitor name> — <what happened>"
---

Conduct a deep competitive analysis. The subject: $ARGUMENTS

## Analysis framework

### 1. What exactly happened
Use Firecrawl to scrape all relevant pages (announcement, product page, pricing page,
any linked resources). Search for third-party coverage and analyst reaction.
Summarize factually what changed — no interpretation yet.

### 2. Why they did it
Based on the evidence, what strategic logic explains this move? Consider:
- Market pressure they might be responding to
- Customer feedback signals (check G2/Capterra reviews from last 3 months)
- Funding or cost pressure implications
- Competitive response to someone else's move

### 3. Who it affects
- Which customer segments does this change their appeal to?
- Does this close a gap where we currently win deals against them?
- Does this open a new gap we could exploit?

### 4. Threat level
Rate: Critical / High / Medium / Low / Monitor
Explain the rating in one paragraph.

### 5. Response options
Generate three strategic options:
- **Option A — Respond directly:** [specific product or positioning response]
- **Option B — Differentiate harder:** [double down on where we are distinct]
- **Option C — Ignore and monitor:** [rationale for not acting now]

### 6. 30-day watch items
What would cause you to upgrade or downgrade the threat rating?

---

Save to `research/competitive/deep-dives/[competitor]-[YYYY-MM-DD].md`.

````

### 2e. Example output

Here is an illustrative example of what `/compete-brief` produces:

```
# Weekly Competitive Brief — 2026-06-19

## Executive Summary
DataFlow Pro shipped native Airtable integration and cut their Starter plan price
by 30%. Combined, these moves directly address the NGO segment's most common stack.
Monitor closely.

## Competitor Updates

### DataFlow Pro
**Signal level:** High
**Summary:** Shipped Airtable integration (announced Monday), lowered Starter plan
from $199/mo to $139/mo, opened 3 new "nonprofit partnerships" roles on LinkedIn.

**What changed:**
- Airtable integration live — https://dataflowpro.io/changelog/airtable
- Pricing page updated — Starter now $139/mo (was $199/mo)
- 3 LinkedIn job posts: "Nonprofit Partnerships Manager" posted June 17

**Why it matters:**
Airtable is used by ~60% of NGOs we talk to. At $139/mo this now competes
directly with our entry tier.

**Questions this raises:**
- Do we need to move faster on our own Airtable connector?
- Should we respond with a pricing adjustment or lean harder into open-source framing?

### Acme Analytics
**Signal level:** Low
**Summary:** Published one blog post on "Data governance for nonprofits." No product
or pricing changes detected.

**What changed:**
- Blog: "5 Data Governance Practices for Nonprofits" — not a product announcement

**Why it matters:**
Acme is sniffing around the NGO messaging space but has no dedicated offering yet.
Content-only play, not a product signal.

## Cross-Competitor Themes
Both competitors published NGO-adjacent content this week. Neither has a dedicated
nonprofit offering, but the messaging convergence suggests the segment is becoming
visible to them.

## Recommended Actions
- Product: Accelerate Airtable connector — tag as P1 if not already
- Marketing: Publish explicit NGO pricing comparison page before DataFlow Pro does
```

---

## 3. User Interview Synthesis Pipeline

### 3a. The problem with manual synthesis

Manual interview synthesis has three failure modes:

1. **Time pressure crushes depth.** You read 8 transcripts the night before a planning meeting and extract the loudest quotes, not the most representative ones.
2. **Recency and vividness bias.** The last interview you read, and the most emotional one, overweight your takeaways.
3. **Patterns across interviews are invisible.** When 7 of 12 people mention the same workaround in different words, manual review usually catches 3 of them. Claude catches all 7.

### 3b. The pipeline

```
Raw transcript (txt / docx / Otter.ai export)
        ↓
Step 1: Quote extraction by theme
        ↓
Step 2: Cross-interview pattern analysis
        ↓
Step 3: Jobs-to-be-Done mapping
        ↓
Step 4: Opportunity statement generation
        ↓
Step 5: Frequency + intensity scoring
        ↓
Structured insight file → feeds PRD context
```

### 3c. The `/synthesize-interviews` command

Create `.claude/commands/synthesize-interviews.md`:

````markdown
---
description: Synthesize user interview transcripts into structured JTBD insights
argument-hint: "<path to transcripts folder or single transcript>"
---

You are a senior UX researcher specializing in Jobs-to-be-Done analysis.
Your task is to synthesize user interview transcripts into structured insights.

Source: $ARGUMENTS
Context about our product and users: Read `research/interviews/context.md` first.

---

## STEP 1 — Extract quotes by theme

Read all transcript files in the specified path.

For each transcript, extract every quote that relates to:
- **Pain points** — frustration, difficulty, things that don't work
- **Workarounds** — how they currently solve the problem without our product
- **Triggers** — what caused them to look for a solution
- **Goals** — what outcome they are trying to achieve
- **Language** — exact words they use to describe their situation

Format each extracted quote as:
```
[THEME] "[exact quote]"
— Participant: [ID or role], File: [filename]
```

Do not paraphrase. Preserve exact language — this is critical for PRD writing.

---

## STEP 2 — Identify cross-interview patterns

Review all extracted quotes across all interviews.

For each pattern you identify:
1. Name the pattern (use the user's language, not product language)
2. List every quote that supports it, with participant IDs
3. Count: how many distinct participants mentioned it?
4. Rate intensity: 1 (mild frustration) to 5 (explicit pain, strong emotion, multiple mentions per interview)

Format:
```
### Pattern: [Name in user's words]
Count: [N of M participants] | Intensity: [1-5]

Supporting quotes:
- "[quote]" — P1
- "[quote]" — P3
- "[quote]" — P7
```

---

## STEP 3 — Map to Jobs-to-be-Done

For each significant pattern (count ≥ 3 OR intensity ≥ 4), write a Job Statement:

**Format:** "When [situation/context], I want to [motivation/action], so I can [expected outcome]."

Then identify:
- **Push forces:** What frustrations in their current situation are driving them to change?
- **Pull forces:** What outcome is attracting them to a new solution?
- **Anxieties:** What concerns might prevent them from switching or adopting?
- **Habits:** What existing behavior are they anchored to?

---

## STEP 4 — Generate opportunity statements

For each Job, write an opportunity statement in this format:

"How might we help [user type] [do the job] when [context], so they can [outcome] — without [current constraint]?"

Example:
"How might we help program coordinators consolidate field data from 5 sources
into one place every Monday morning, so they can have their weekly report ready
by 9am — without needing to know SQL?"

---

## STEP 5 — Priority matrix

Create a summary table:

| Job Statement | Participants | Intensity | Current Solution | Opportunity Score |
|---------------|-------------|-----------|-----------------|-------------------|
| [job] | N/M | 1-5 | [what they do now] | [frequency × intensity / 10] |

Sort by Opportunity Score descending.

---

## Final output

Write a synthesis document to `research/interviews/synthesized/[date]-synthesis.md` with:
1. Key themes (bullet summary, 5 bullets max)
2. Full pattern analysis (Step 2 output)
3. JTBD map (Step 3 output)
4. Opportunity statements (Step 4 output)
5. Priority matrix (Step 5 output)
6. Recommended next steps (what to validate, what to build)

````

**The context file** — create `research/interviews/context.md`:

```markdown
# Interview Research Context

## Product
Dalgo — open-source data platform for NGOs. Automates data ingestion, transformation,
and reporting for organizations that currently use Excel/Google Sheets.

## Primary users being interviewed
- Program coordinators at NGOs (20–200 person orgs)
- Data coordinators who manage field data collection
- Finance managers who produce donor reports

## What we are trying to learn
- How do they currently get data from field systems into reports?
- Where does the process break down?
- What does "good enough" look like for them?

## What we already know (do not re-surface as new insight)
- Users are non-technical — SQL is a barrier
- Internet connectivity is often poor
- Small budgets — total software spend < $500/year is common

## Jobs we have already validated
- "Create donor report from field data" — confirmed, building for this
- "Monitor program KPIs weekly" — confirmed, in roadmap

## Jobs we are still investigating
- Data collection from mobile/paper forms
- Multi-country program consolidation
- Sharing reports with external stakeholders
```

### 3d. Handling multiple interview formats

| Format | How to prepare it |
|--------|------------------|
| Otter.ai / Rev transcript | Download as `.txt`, drop in `research/interviews/raw/` |
| Zoom recording | Run through `whisper` CLI first: `whisper recording.mp4 --output_format txt` |
| Manual notes | Write as free-form `.md` — Claude handles messy notes fine |
| Typeform / Google Form responses | Export as CSV, rename to `.txt`, Claude reads it |
| Loom recordings | Use Loom's auto-transcript export or Whisper |

Whisper setup (one-time):
```bash
pip install openai-whisper
whisper interview-recording.mp4 --output_format txt --output_dir research/interviews/raw/
```

### 3e. Building a living research repository

After each synthesis run, Claude appends to a master insights file. Add this to the bottom of your `synthesize-interviews.md` command:

````markdown
## STEP 6 — Update master insights registry

Append new confirmed jobs and patterns to `research/interviews/MASTER-INSIGHTS.md`.
Use this format:

```markdown
## [Job Statement] — Added [DATE]
- **Confidence:** [Low / Medium / High]
- **Evidence:** [N participants across N interview batches]
- **Key quote:** "[most representative quote]"
- **Source files:** [list of synthesis docs that support this]
- **Status:** [Investigating / Validated / In roadmap / Shipped]
```

Do not duplicate entries that already exist. If a new batch confirms an existing
job, update its confidence level and evidence count.
````

This master file becomes queryable. When writing a PRD, you can say:

```
> Read research/interviews/MASTER-INSIGHTS.md and tell me everything users have said about sharing reports with external stakeholders
```

---

## 4. Market and Trend Monitoring

### 4a. What to monitor

Set up monitoring in these categories based on your product's exposure:

| Category | What to watch | Frequency |
|----------|--------------|-----------|
| Industry news | "NGO data management" "nonprofit technology" | Daily |
| Analyst reports | Gartner, Forrester releases on ELT/data stack | Weekly |
| Regulatory | Data privacy laws, donor reporting requirements by country | Monthly |
| Adjacent markets | Salesforce NPSP, Kobo Toolbox, CommCare, ODK updates | Weekly |
| Funding news | Competitors' funding rounds, acqui-hires | Weekly |
| Talent flows | Where CI talent is moving (LinkedIn job changes) | Monthly |

### 4b. Daily digest setup

Create `.claude/commands/market-digest.md`:

````markdown
---
description: Generate morning market intelligence digest
---

You are a market intelligence analyst for a data platform serving NGOs.
Generate today's digest. Today is $CURRENT_DATE.

Read `research/market/signal-filter.md` to understand what matters to us.

## Search tasks (use Firecrawl search tool)

Run these searches and collect top 5 results for each:

1. `NGO data platform news {current_week}`
2. `nonprofit technology funding announcement {current_month}`
3. `ELT pipeline open source announcement {current_week}`
4. `[competitor 1] announcement OR launch {current_week}`
5. `[competitor 2] announcement OR launch {current_week}`
6. `data governance nonprofit {current_month}`

For each result that passes the signal filter: scrape the full article.
Discard results that do not pass the filter.

## Write the digest

Format:

---
# Market Digest — [DATE]

## Must-read today
[0–2 items only — things that require action or immediate awareness]

## Worth knowing
[3–5 items — relevant but not urgent]

## Filed for later
[Other items worth archiving — link only, one-line summary]

## Noise (filtered out)
[Count of items filtered; one example of what was filtered and why]

---

Append digest to `research/market/digests/YYYY-MM.md` (monthly file).

````

**The signal filter** — create `research/market/signal-filter.md`:

```markdown
# Market Signal Filter

## Pass (include in digest)
- Any competitor product launch, pricing change, or funding round
- Any news about the NGO/nonprofit technology sector with > 100 org impact
- Regulatory changes affecting data handling in LMIC countries
- Open source ELT tools gaining significant traction (> 1000 GitHub stars in a week)
- Any analyst report on ELT, data democratization, or social sector data
- Partnerships between adjacent tools (Kobo + Salesforce, ODK + BigQuery, etc.)

## Filter out (exclude from digest)
- Generic AI hype articles with no product specifics
- Funding rounds for B2B SaaS with no NGO/social sector angle
- Conference announcements more than 3 months away
- News about companies with > $100M ARR (different market than ours)
- Opinion pieces without data or specific claims
- Anything we covered in the last 14 days

## Escalate immediately (flag as Must-read)
- Any competitor raising > $5M
- Any news about a competitor targeting NGOs or nonprofits specifically
- Any data breach or compliance event in our category
- Open source alternative to Airbyte or dbt with strong early traction
```

### 4c. Feeding monitoring into strategy context

Add this to the bottom of your `market-digest.md` command:

````markdown
## Update strategy context

If today's digest contains any "Must-read" items, append a brief entry to
`research/market/STRATEGY-CONTEXT.md`:

```markdown
## [DATE] — [1-line description of the development]
[2–3 sentence impact statement for product strategy]
**Action owner:** [PM / Engineering / Marketing]
**Review date:** [DATE + 30 days]
```

This file is read automatically by `/engineering/plan-feature` when creating
implementation plans. The strategy context tells Claude what the competitive
environment looks like when planning new features.
````

---

## 5. Customer Feedback Synthesis

### 5a. Sources to batch-process

Collect feedback from all of these into `research/feedback/raw/`:

| Source | How to export | Filename convention |
|--------|--------------|---------------------|
| Intercom/Zendesk | CSV export filtered by date range | `support-YYYY-MM.csv` |
| NPS tool | CSV of verbatim comments | `nps-YYYY-MM.csv` |
| App store reviews | Manual copy or API export | `app-reviews-YYYY-MM.txt` |
| Slack community | Export specific channels | `community-YYYY-MM.txt` |
| Sales call notes | CRM export or manual | `sales-notes-YYYY-MM.md` |

### 5b. The `/feedback-synthesis` command

Create `.claude/commands/feedback-synthesis.md`:

````markdown
---
description: Synthesize customer feedback from all sources into structured insights
argument-hint: "YYYY-MM (month to process) or path to specific file"
---

You are a customer insights analyst. Process all customer feedback for the
specified period and generate a structured insights report.

Period: $ARGUMENTS
Read context from: `research/feedback/signal-filter.md`

## STEP 1 — Load and normalize

Read all files in `research/feedback/raw/` matching the period.
For each piece of feedback, create a normalized entry:

```
SOURCE: [support/nps/review/community/sales]
DATE: [if available]
SENTIMENT: [positive/neutral/negative]
TEXT: [verbatim content, trimmed to 200 chars max]
```

## STEP 2 — Theme clustering

Group all entries by theme. Use inductive coding — let themes emerge from
the data, do not force them into pre-existing categories.

For each theme:
- Name it using the customer's language
- List the 3 most representative verbatim quotes
- Count: total mentions, breakdown by source
- Sentiment breakdown: % positive / neutral / negative within theme

## STEP 3 — Trend detection

Compare this period's themes against `research/feedback/insights/MASTER-THEMES.md`
(if it exists).

Flag:
- **New themes** — appearing this period that were not in previous periods
- **Escalating themes** — themes with > 20% increase in mentions vs. last period
- **Declining themes** — themes with > 30% decrease (potential sign that a fix worked)
- **Stable themes** — persistent pain that has not changed

## STEP 4 — Severity scoring

For each theme, score severity (1–5):
- 1 = Nice to have, mentioned once or twice
- 2 = Recurring but users work around it
- 3 = Meaningful friction, affects regular workflows
- 4 = Blocking for some users, mentioned with frustration
- 5 = Causing churn or preventing adoption

Severity = (mention frequency × emotional intensity) / normalizing factor

## STEP 5 — Action recommendations

For each theme with severity ≥ 3:
- Link to any existing GitHub issue or roadmap item (check `research/feedback/known-issues.md`)
- If no existing item: draft a one-paragraph problem statement suitable for a GitHub issue
- Recommend: Fix now / Add to roadmap / Investigate further / Acknowledge and decline

## Final output

Write to `research/feedback/insights/YYYY-MM-insights.md`:

1. **Headline summary** (3 bullets — what you would say in a 30-second standup)
2. Theme clusters with quotes
3. Trend analysis (new, escalating, declining)
4. Severity-ranked action list
5. Draft GitHub issues for new high-severity items

Update `research/feedback/insights/MASTER-THEMES.md` with this period's data.

````

### 5c. New pattern vs. noise

Add this decision rule to your feedback signal filter:

```markdown
# Feedback Signal Filter

## Treat as a real pattern (act on it)
- 3+ mentions of the same specific issue in a single month
- Any mention that explicitly caused a downgrade, cancellation, or blocked adoption
- A theme that is new AND has severity ≥ 3 in its first month
- Verbatim quotes that show the user tried the thing and gave up

## Treat as noise (monitor but don't act yet)
- Single mentions of edge cases that affect < 1% of users
- Feature requests for things that conflict with product strategy
- Complaints about integrations we don't control (e.g., Airbyte connector bugs)
- Feedback from users who are clearly outside our ICP

## Escalate to engineering immediately
- Security or data integrity concerns, regardless of frequency
- A workaround that creates data corruption risk
- Anything where a user's production data was affected
```

---

## 6. Building a Queryable Research Repository

### Folder structure

```
research/
├── interviews/
│   ├── raw/                   ← original transcripts (.txt, .md)
│   ├── synthesized/           ← Claude synthesis outputs
│   ├── context.md             ← product + user context for synthesis
│   └── MASTER-INSIGHTS.md     ← living registry of validated jobs
├── competitive/
│   ├── competitors.yaml       ← competitor list + URLs
│   ├── briefs/                ← weekly briefs (YYYY-MM-DD.md)
│   └── deep-dives/            ← per-competitor analysis
├── market/
│   ├── digests/               ← monthly digest files (YYYY-MM.md)
│   ├── signal-filter.md       ← what counts as signal for our market
│   └── STRATEGY-CONTEXT.md    ← living strategy implications log
└── feedback/
    ├── raw/                   ← exports from support, NPS, etc.
    ├── insights/              ← monthly synthesis outputs
    ├── known-issues.md        ← link table: theme → GitHub issue
    ├── signal-filter.md       ← noise filter for feedback
    └── MASTER-THEMES.md       ← living registry of feedback themes
```

### Making the repository queryable

The MASTER files are the key. They are designed to be read by Claude as context when doing other work. To use them effectively:

**In PRD writing:**
```
> Before we write this PRD, read research/interviews/MASTER-INSIGHTS.md
  and research/feedback/MASTER-THEMES.md and tell me everything we know
  about how users manage multi-country program data.
```

**In planning sessions:**
```
> Read research/market/STRATEGY-CONTEXT.md and research/competitive/briefs/
  (last 4 weeks) and summarize the competitive context I should consider
  when planning Q3.
```

**In stakeholder updates:**
```
> Read research/feedback/insights/ for the last 3 months and write a
  "voice of the customer" section for our board update — 5 bullet points,
  quoting verbatim user language where possible.
```

The repository becomes useful when it is consistent. Run `/compete-brief` every Monday, `/feedback-synthesis` every two weeks, and `/synthesize-interviews` within 48 hours of each interview batch. Keep the MASTER files up to date and they compound in value.

---

## 7. Complete Setup Checklist

### Install (one-time, 30 minutes)

- [ ] Install Claude Code: `npm install -g @anthropic-ai/claude-code`
- [ ] Get Firecrawl API key at [firecrawl.dev](https://firecrawl.dev) (free tier is enough to start)
- [ ] Add Firecrawl MCP: `claude mcp add firecrawl-mcp -e FIRECRAWL_API_KEY=fc-YOUR-KEY -- npx -y firecrawl-mcp`
- [ ] Optional: Install Whisper for audio transcription: `pip install openai-whisper`

### Create folder structure (5 minutes)

```bash
mkdir -p research/interviews/{raw,synthesized}
mkdir -p research/competitive/{briefs,deep-dives}
mkdir -p research/market/digests
mkdir -p research/feedback/{raw,insights}
```

### Create context files (30 minutes — this is the real work)

- [ ] `research/competitive/competitors.yaml` — list your 3–5 top competitors with URLs
- [ ] `research/interviews/context.md` — describe your product, users, and known insights
- [ ] `research/market/signal-filter.md` — define what counts as signal for your market
- [ ] `research/feedback/signal-filter.md` — define noise thresholds for feedback

### Create slash commands (10 minutes — copy from this guide)

- [ ] `.claude/commands/compete-brief.md`
- [ ] `.claude/commands/compete-deep-dive.md`
- [ ] `.claude/commands/synthesize-interviews.md`
- [ ] `.claude/commands/market-digest.md`
- [ ] `.claude/commands/feedback-synthesis.md`

### Day 1 actions

- [ ] Run `/compete-brief` — verify it scrapes all competitor URLs correctly
- [ ] Run `/market-digest` — verify it returns useful results, not noise
- [ ] Put 2 existing interview transcripts in `research/interviews/raw/` and run `/synthesize-interviews`
- [ ] Put last month's support export in `research/feedback/raw/` and run `/feedback-synthesis`

### Week 1 calibration

- [ ] Review compete-brief output — add URLs to `competitors.yaml` for any pages that were missed
- [ ] Review market-digest output — update `signal-filter.md` to filter out things that came through as noise
- [ ] Review interview synthesis — check if the JTBD mapping matches your existing intuition
- [ ] Adjust prompt wording in any command where the output format did not match what you needed

---

## 8. Time Savings Calculator

These estimates assume you are currently doing this work manually and spending the time.

| Task | Manual time/week | Automated time/week | Savings |
|------|-----------------|--------------------|---------| 
| Weekly competitive brief (3 competitors) | 3–4 hours | 15 min (review + distribute) | ~3.5 hrs |
| Deep dive when competitor makes a move | 4–6 hours (monthly) | 45 min | ~1 hr/week |
| Interview synthesis (2 interviews/week) | 2–3 hours | 20 min (review output) | ~2.5 hrs |
| Market monitoring and triage | 1–2 hours | 10 min (read digest) | ~1.5 hrs |
| Customer feedback synthesis (bi-weekly) | 3–4 hours | 30 min | ~1.5 hrs |
| **Total** | **~9–12 hrs/week** | **~1.5 hrs/week** | **~8–10 hrs/week** |

At a PM rate of $80–120/hour (fully-loaded), that is $32,000–$62,000/year in reclaimed time.

Cost of running the system: Firecrawl free tier covers basic monitoring. At higher volumes (daily scraping of 10 competitors), you will spend $15–30/month on Firecrawl credits plus your Claude API costs. Budget $50–100/month for a well-used system.

**The real return is not the hours.** It is that competitive briefs go from "once a month when someone has time" to "every Monday without fail." Pattern detection across interviews goes from "whatever the PM noticed" to "every quote, every transcript, every time." Consistency is the ROI.

---

*Sources used in building this guide:*
- [Firecrawl MCP documentation](https://www.firecrawl.dev/use-cases/ai-mcps)
- [Firecrawl MCP server setup guide](https://use-apify.com/blog/firecrawl-mcp-server-setup)
- [Competitive Intelligence Automation: The 2026 Playbook](https://arisegtm.com/blog/competitive-intelligence-automation-2026-playbook)
- [Claude Code for Product Managers — Teresa Torres, Lenny's Newsletter](https://www.lennysnewsletter.com/p/claude-code-for-product-managers)
- [How to Use Claude Code: Slash Commands, Agents, Skills](https://www.producttalk.org/how-to-use-claude-code-features/)
- [How AI Helps with Competitive Intelligence in 2026 — Klue](https://klue.com/topics/how-ai-helps-with-competitive-intelligence)
- [User Interview Synthesis — Miskies AI](https://www.miskies.app/how-to-apply-ai/product-management/discovery/user-interview-synthesis)
- [Jobs-to-be-Done Interview Guide 2026](https://www.koji.so/blog/jobs-to-be-done-interview-guide-2026)
- [Connect Claude Code to tools via MCP — Official Docs](https://code.claude.com/docs/en/mcp)
