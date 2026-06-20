# MCP Use-Case Catalog — What People Actually Build With Model Context Protocol

**Purpose:** A catalog of *real, documented* workflows people build by connecting MCP servers to AI
agents. For builders looking for concrete inspiration: what the workflow does, which server(s) it
uses, how the agent drives it, the value, and a source link.

**Scope & honesty note:** Every entry below is backed by a public source I retrieved (blog, vendor
cookbook, engineering post, practitioner writeup, or directory). Where a "workflow" is described by a
vendor as a *capability* or *example prompt* rather than a documented deployment with outcomes, it is
marked **[illustrative / vendor example]**. Documented production deployments with named orgs and/or
metrics are marked **[production]**. Researched June 2026; ecosystem is moving fast.

---

## Summary table

*(Curated highlights — a subset of the ~30 workflows below; see the body for the full set.)*

| # | Domain | Workflow (one line) | Server(s) | Status | Source |
|---|--------|---------------------|-----------|--------|--------|
| 1 | Dev | Weekly performance-triage bot opens PRs with fixes | Sentry, GitHub | Documented (vendor cookbook) | Sentry |
| 2 | Dev | "Fix this issue" loop: Sentry error → suspect commit → patch | Sentry, GitHub | Documented | Continue / Sentry |
| 3 | Dev | Copilot coding agent verifies its own changes in a real browser | Playwright | Production | Microsoft/GitHub |
| 4 | Dev | Self-healing / exploratory E2E test generation | Playwright | Documented | Microsoft, testomat |
| 5 | Dev | Cross-tool incident context: Jira ↔ Sentry ↔ GitHub | Sentry, GitHub, Jira | Illustrative | Sentry |
| 6 | Dev | Internal calendar/issue ops via SQL-over-MCP | Block Linear/Calendar MCP | Production | Block Eng |
| 7 | Data | "DWAINE" NL warehouse assistant, 250 users, ~70% of Qs | ClickHouse + LibreChat | Production | ClickHouse |
| 8 | Data | Read-only NL→SQL over BigQuery with dry-run validation | BigQuery MCP | Documented | ergut, Skyvia |
| 9 | Data | Product-analytics deep dives in plain English | PostHog/Amplitude/Mixpanel | Illustrative | Apigene |
| 10 | Data | Infra/observability "why is it slow" queries | Datadog/Grafana/New Relic | Illustrative | Apigene |
| 11 | Product/Design | Code → Figma → code round-trip | Figma MCP | Documented | Figma, Builder.io |
| 12 | Product/Design | Test feedback in Notion → aggregated → Linear tickets | Notion, Linear | Production (Gradient Labs) | Pragmatic Engineer |
| 13 | Product/Design | PRD/Notion doc → structured Linear project | Linear, Notion | Illustrative | Apigene, Notion |
| 14 | Product/Design | Notion as content engine: 30 posts → auto-publish | Notion + Make | Documented (user story) | Manus |
| 15 | Sales/Mktg | Competitor SEO intel + content-gap analysis | Semrush, Ahrefs | Documented (vendor) | Semrush, Ahrefs |
| 16 | Sales/Mktg | Prospect → enrich → sequence → track | Apollo, HubSpot | Illustrative | Apigene |
| 17 | Sales/Mktg | CRM enrichment, lead scoring, stale-deal flagging | Salesforce, HubSpot, Attio | Illustrative | Apigene, Fastio |
| 18 | Support | Ticket triage: classify, prioritize, route, draft reply | Zendesk, Intercom | Documented (vendor) | Workato, Composio |
| 19 | Support | Bot deflection-rate investigation | Intercom | Illustrative | Vinkius |
| 20 | Support | SRE-style customer-issue triage agent | Azure SRE + MCP | Documented | Microsoft |
| 21 | Productivity | Focus-time blocking + email triage + meeting follow-ups | Google Workspace + Zapier | Documented (user story) | aimaker |
| 22 | Sales/Mktg | Supplier research → evaluate → draft outreach in Notion | Manus + Notion | Documented (user story) | Manus |
| 23 | Cross-domain | "Process my open tickets" → 5-step automation | IT-support MCP | Demo | Automation Anywhere |
| 24 | Cross-domain | Block-wide: query data, detect fraud, manage incidents | 60+ servers via Goose | Production | Block / Elastic |

---

## 1. Software development

The single most-developed MCP domain. ~42 of the top-50 most-searched MCP servers are used by
engineers ([PulseMCP via mcpmanager](https://mcpmanager.ai/blog/most-popular-mcp-servers/)).

### 1.1 Weekly performance-triage bot that opens PRs — [documented, vendor cookbook]
Sentry's own cookbook walks through a Claude Code scheduled task: **every Monday 9 AM**, the agent
queries Sentry for the worst performance issues and opens GitHub PRs with fixes. The chain is explicit:
`search_issues` (separate single-topic queries for N+1 and slow DB calls) → `get_issue_details` on the
top 3 by event count (pulls traces + stack traces) → `analyze_issue_with_seer` for root cause → opens
up to 2 GitHub PRs per run, each with the Sentry trace link, root-cause analysis, and code changes,
with dedup checks. Value: eliminates manual weekly performance review while keeping human code review
before merge.
Source: [Sentry — Automate weekly performance triage](https://sentry.io/cookbook/performance-bot-sentry-claude/)

### 1.2 "Fix this issue" loop from a production error — [documented]
The general pattern Sentry promotes: instead of switching to the dashboard, copying a stack trace, and
pasting it back into the editor, the agent pulls issue details, searches events, and invokes Sentry's
AI (Seer) for root cause — all inside the IDE. Continue's guide builds an automated loop that analyzes
Sentry issues, finds patterns, and **creates actionable GitHub issues with suggested fixes**.
Sources: [Sentry MCP repo](https://github.com/getsentry/sentry-mcp) ·
[Continue — Sentry MCP error monitoring](https://docs.continue.dev/guides/sentry-mcp-error-monitoring) ·
[Sentry blog — Smarter debugging with Cursor](https://blog.sentry.io/smarter-debugging-sentry-mcp-cursor)

### 1.3 Coding agent that verifies its own change in a real browser — [production]
GitHub Copilot's coding agent ships Playwright MCP built in. When asked to implement a feature or fix a
bug, it opens the browser, navigates to the app, and verifies the change before declaring done — a
`prompt → generate → verify-in-browser → confirm` loop. Agents drive the page via the **accessibility
tree** (element roles/names/refs), which is deterministic and needs no vision model.
Sources: [Microsoft Dev Blog — Complete Playwright E2E story](https://developer.microsoft.com/blog/the-complete-playwright-end-to-end-story-tools-ai-and-real-world-workflows) ·
[microsoft/playwright-mcp](https://github.com/microsoft/playwright-mcp)

### 1.4 Self-healing / exploratory E2E test generation — [documented]
Practitioners use Playwright MCP for agentic loops that benefit from persistent state and iterative
reasoning over page structure: exploratory automation, **self-healing tests**, and long-running
autonomous test generation. The agent navigates, reads content, fills forms, screenshots, inspects
network traffic, and checks console errors — then writes the test.
Sources: [testomat — Playwright MCP zero to hero](https://testomat.io/blog/playwright-mcp-modern-test-automation-from-zero-to-hero/) ·
[testcollab — Playwright MCP guide](https://testcollab.com/blog/playwright-mcp)

### 1.5 Cross-tool incident context — [illustrative / vendor example]
Sentry pitches chained prompts like: *"Find the Sentry error linked to this Jira ticket, get the
suspect commit from GitHub, and suggest a fix."* End-to-end across the toolchain in one prompt. Shown
as an example, not a documented deployment with metrics.
Source: [Sentry MCP repo](https://github.com/getsentry/sentry-mcp)

### 1.6 Internal dev ops via SQL-over-MCP — [production]
Block redesigned its internal servers so the LLM is more efficient. Their **Linear MCP v3** collapsed
30+ mirrored GraphQL tools down to two (`execute_readonly_query`, `execute_mutation_query`) so the
agent can answer "all issues assigned to X" in one GraphQL call instead of 4-6 sequential tool calls.
Their **Google Calendar MCP v2** loads events into DuckDB so employees can ask "what does my day look
like?" or find common meeting times via SQL. Block's stated position: *"the usage of MCP is no longer
a proof of concept at Block, it's how we work."*
Source: [Block Engineering — Block's playbook for designing MCP servers](https://engineering.block.xyz/blog/blocks-playbook-for-designing-mcp-servers)

---

## 2. Data & analytics

### 2.1 "DWAINE" — natural-language warehouse assistant — [production, with metrics]
ClickHouse built an internal assistant (Data Warehouse AI Natural Expert) connecting **LibreChat to
their ClickHouse warehouse via MCP**, enriched with a comprehensive **business glossary** giving the
model context on the data model and business logic. Documented outcomes: **250+ internal users, 200+
daily messages, ~70% of data-warehouse questions handled, analyst workload reduced an estimated
50-70%.** This is one of the strongest documented "agent reduces real analytical work" cases. The key
lesson: semantic context (the glossary) + direct DB access is what makes it a *primary* interface, not
an aux toy.
Source: [ClickHouse — Building a data platform for agents](https://clickhouse.com/blog/building-a-data-platform-for-agents)

### 2.2 Read-only NL→SQL over BigQuery with safety rails — [documented]
BigQuery MCP servers are **read-only by design — only SELECT statements allowed**, and every query is
validated by BigQuery's own **dry-run planner** before execution. Non-technical users ask "Show me last
quarter's revenue by region" without knowing table names or SQL. The read-only + dry-run pattern is the
template most warehouse MCP servers converge on.
Sources: [ergut/mcp-bigquery-server](https://github.com/ergut/mcp-bigquery-server) ·
[Skyvia — MCP server for BigQuery](https://skyvia.com/blog/mcp-server-for-google-bigquery/) ·
[ClickHouse — MCP and data warehouses](https://clickhouse.com/resources/engineering/mcp-data-warehouse-everthing-you-need-to-know)

### 2.3 Product-analytics deep dives in plain English — [illustrative / vendor example]
Pattern: agents query PostHog/Amplitude/Mixpanel so a PM can ask *"What's our 7-day retention for users
who activated the new onboarding flow?"* and get a data-backed answer without writing SQL.
Source: [Apigene — 19 MCP use cases](https://apigene.ai/blog/mcp-use-cases)

### 2.4 Observability "why is it slow" queries — [illustrative / vendor example]
Agents pull live metrics, deployments, and error rates from Datadog/New Relic/Grafana so an engineer
can ask *"What's causing the latency spike on the payments service?"* without tab-switching.
Source: [Apigene — 19 MCP use cases](https://apigene.ai/blog/mcp-use-cases)

### 2.5 Cloud-asset querying for security/finops — [documented]
CloudQuery exposes a read-only MCP server so any LLM can query synced cloud-asset inventory in natural
language (e.g., misconfigurations, untagged resources, cost outliers).
Source: [CloudQuery MCP Server docs](https://www.cloudquery.io/docs/platform/features/mcp-server)

---

## 3. Product & design

### 3.1 Code → Figma → code round-trip — [documented]
The newer Figma workflow isn't just design→code. The loop is: generate UI from a prompt in Claude Code
→ **push that design into Figma as editable layers via the Figma MCP** → edit in Figma normally → sync
changes back to code. Figma rolled out native MCP integration (connect to VS Code+Copilot or Cursor in
a few clicks). `get_design_context` / `get_code` work across Claude Code, Codex, Cursor, VS Code.
Sources: [Figma blog — Claude Code to Figma](https://www.figma.com/blog/introducing-claude-code-to-figma/) ·
[Builder.io — Claude Code to Figma](https://www.builder.io/blog/claude-code-to-figma) ·
[arinspunk/claude-talk-to-figma-mcp](https://github.com/arinspunk/claude-talk-to-figma-mcp)

### 3.2 Test feedback → aggregated DB → Linear tickets — [production, real team]
A Gradient Labs engineer (reported as Theo Windebank in coverage of the piece) described a real testing
workflow: people logged feature-test
results and feedback as rows in a **Notion** database; **Claude Code** built a *new* database with
results aggregated/categorized by underlying issue; then the **Linear MCP** created a new project and
filled it with tickets generated from the aggregated rows. A genuine "messy human data → structured
project plan" automation, not a vendor demo.
Source: [Pragmatic Engineer — Building MCP servers in the real world](https://newsletter.pragmaticengineer.com/p/mcp-deepdive)

### 3.3 PRD / Notion doc → structured Linear project — [illustrative / vendor example]
Common pattern: turn a PRD or Notion doc into a structured Linear project, summarize project progress,
or create issues from docs. Notion's own guidance shows connecting custom agents to Linear/Jira via MCP.
Sources: [Apigene — 19 MCP use cases](https://apigene.ai/blog/mcp-use-cases) ·
[Notion — Connect custom agents to MCP](https://www.notion.com/help/guides/connect-custom-agents-to-mcp-integrations)

### 3.4 Notion as a content engine — [documented user story]
A content creator generated 30 Instagram posts from one prompt; Manus populated a structured Notion DB,
and Make.com auto-published to multiple platforms daily. A marketing team set a recurring Monday task:
Manus researches TikTok trends and updates a Notion DB — "from reactive trend-chasers to proactive
strategists." An HR manager turned a static Notion onboarding page into a web app with progress tracking
and an AI chatbot while "Notion remained the source of truth."
Source: [Manus — Notion MCP use cases](https://manus.im/blog/manus-notion-mcp-use-cases)

### 3.5 Competitive / web research — [illustrative / vendor example]
Agents research competitors, extract pricing, and monitor market changes via Firecrawl / Exa / Bright
Data scraping servers. Exa is reportedly the **most-used search MCP server** in 2026.
Sources: [Apigene — 19 MCP use cases](https://apigene.ai/blog/mcp-use-cases) ·
[mcpmanager — 50 most popular](https://mcpmanager.ai/blog/most-popular-mcp-servers/)

---

## 4. Sales / marketing / growth

### 4.1 Competitor SEO intelligence + content-gap analysis — [documented, vendor]
Semrush's MCP knowledge base documents specific prompts users actually run:
- *"Pull data on which competitors gained the most organic traffic in our core market. List the
  keywords and pages driving that growth."*
- *"Find new keywords competitors started ranking for this month that we do not currently target."*
- *"Analyze [domain]'s organic keywords and suggest content topics where rankings are between
  positions 8-20 and could realistically reach the top 5."*
- *"Generate a weekly intelligence brief summarizing competitor organic growth..."* — a recurring agent
  that pulls traffic/keyword data daily and surfaces ranking gains, traffic shifts, and new entries.
Ahrefs MCP covers the same shape: keyword research, traffic-trend tracking across a competitor list.
Sources: [Semrush — MCP common use cases](https://www.semrush.com/kb/1621-mcp-common-use-cases) ·
[Ahrefs MCP](https://www.ahrefs.com/mcp)

### 4.2 Prospect → enrich → sequence → track — [illustrative / vendor example]
Full outbound loop across Apollo/HubSpot/ZoomInfo: *"Find 20 VP Engineering contacts at Series B
fintech companies in Austin"* → enrich → add to a sequence → personalize → track opens/clicks.
Source: [Apigene — 19 MCP use cases](https://apigene.ai/blog/mcp-use-cases)

### 4.3 CRM enrichment, scoring, stale-deal flagging — [illustrative / vendor example]
A CRM MCP server translates MCP into Salesforce/HubSpot API calls, exposing tools like `get_contact`,
`update_deal`. The HubSpot MCP gives agents full CRM access — list/create/search/update contacts,
companies, deals; read engagement data (calls, emails, meetings); send emails. Cross-platform agents
read and reason across both Salesforce and HubSpot through one protocol, removing the multi-CRM
"integration tax."
Sources: [Fastio — best CRM MCP servers](https://fast.io/resources/best-mcp-servers-crm/) ·
[Huble — 10 HubSpot MCP use cases](https://huble.com/blog/hubspot-mcp-ai-use-cases) ·
[Vantage Point — MCP multi-platform CRM](https://vantagepoint.io/blog/sf/insights/mcp-multi-platform-crm-salesforce-hubspot-integration)

### 4.4 Supplier research → evaluate → outreach — [documented user story]
An entrepreneur had Manus identify 30 suppliers, evaluate them, and draft personalized outreach emails,
all organized into a Notion database — compressing days of manual research into one automated run.
Source: [Manus — Notion MCP use cases](https://manus.im/blog/manus-notion-mcp-use-cases)

---

## 5. Customer support / ops

### 5.1 Ticket triage: classify, prioritize, route, draft — [documented, vendor]
The Zendesk MCP connects Claude/Cursor directly to a Zendesk account. Documented workflow: the agent
reads incoming tickets, searches similar past issues, suggests priority/group based on SLA policies,
classifies by intent/priority/customer-tier, routes to the right team, and drafts responses from help-
center content — escalating complex issues to humans. Workato split this into a **Ticket Management**
layer (queue, triage, SLA, replies) and a **Knowledge Base** layer (article search/creation/updates)
for full coverage under one governed workflow.
Sources: [Workato — MCP Monday: Zendesk KB](https://www.workato.com/product-hub/mcp-monday-zendesk-knowledge-base-is-live/) ·
[Composio — Zendesk MCP + Claude Code](https://composio.dev/toolkits/zendesk/framework/claude-code) ·
[dev.to — Zendesk MCP](https://dev.to/curatedmcp/zendesk-mcp-give-your-ai-agent-direct-access-to-your-support-tickets-49hd)

### 5.2 Bot deflection-rate investigation — [illustrative / vendor example]
Support leaders use the Intercom MCP to evaluate bot deflection rates; if deflection drops, the agent
searches the logs to identify which unresolved queries caused the increase.
Source: [Vinkius — customer support MCP servers](https://blog.vinkius.com/customer-support-mcp-servers-zendesk-freshdesk-intercom)

### 5.3 SRE-style customer-issue triage agent — [documented]
Microsoft documents extending an SRE Agent with MCP to build an agentic workflow that triages customer
issues — connecting the agent to operational tooling so it can investigate and act.
Source: [Microsoft — Extend SRE Agent with MCP](https://techcommunity.microsoft.com/blog/appsonazureblog/extend-sre-agent-with-mcp-build-an-agentic-workflow-to-triage-customer-issues/4480710)

### 5.4 PayPal commerce/checkout agent — [production]
A deployed agent connects to the PayPal MCP server so users can chat to shop, compare prices, and check
out — actions executed through the MCP, not a demo.
Source: [Elastic — MCP overview and emerging use cases](https://www.elastic.co/search-labs/blog/mcp-current-state)

---

## 6. Personal productivity

### 6.1 Workspace automation: focus blocks, email triage, follow-ups — [documented user story]
With Claude + Google Workspace (read) + Zapier MCP (write), one practitioner runs five recurring
workflows:
1. **Focus-time blocking** — Claude reads deadlines and creates "FOCUS: [task]" calendar events 9 AM-1 PM.
2. **Email intelligence** — scans the inbox for urgent threads, drafts context-aware replies (no more
   digging through 3 weeks of threads).
3. **Meeting follow-ups** — summarizes notes (via Fathom), creates follow-up events, emails docs —
   *"saves me at least 5+ hours every week."*
4. **Newsletter triage** — auto-labels AI newsletters, summarizes, files ideas into Google Docs.
5. **Client comms** — reads client email, checks project status in Google Sheets, drafts the reply,
   turning a "30-minute client update process into a 5-minute task."
Source: [aimaker — 5 things I do now that Claude has Google Workspace access](https://aimaker.substack.com/p/5-things-i-actually-do-now-that-claude-access-google-workspace-email-calendar-drive-zapier-mcp)

### 6.2 Calendar scheduling assistant — [documented]
Google's own Workspace MCP servers let agents schedule meetings, search/retrieve Drive docs, draft and
send Gmail, and generate Docs content as part of an automated workflow.
Sources: [Google — Configure Workspace MCP servers](https://developers.google.com/workspace/guides/configure-mcp-servers) ·
[MindStudio — Google Workspace MCP with Claude Code](https://www.mindstudio.ai/blog/google-workspace-mcp-server-claude-code-codex)

### 6.3 File/knowledge organization — [documented user story]
A developer extracted 100 scattered AI-prompt examples from GitHub folders into a searchable,
filterable Notion database with direct GitHub links, ready for team sharing.
Source: [Manus — Notion MCP use cases](https://manus.im/blog/manus-notion-mcp-use-cases)

---

## 7. Cross-domain "agent does my job" workflows

### 7.1 Block: query data, detect fraud, manage incidents via Goose — [production, at scale]
The most ambitious documented deployment. Block built **60+ MCP servers** for Git, Snowflake, Jira, and
Google Workspace; employees use **Goose** (Block's agent) to **query data, detect fraud, and manage
incidents** without coding. MCP is "how we work" at Block, not a PoC.
Sources: [Block Engineering — MCP playbook](https://engineering.block.xyz/blog/blocks-playbook-for-designing-mcp-servers) ·
[Elastic — MCP current state](https://www.elastic.co/search-labs/blog/mcp-current-state)

### 7.2 "Process my open tickets" → autonomous multi-step run — [demo]
An IT-support demo: an agent opens a queue of 50+ tickets; the user types "process my open tickets";
behind the scenes five steps fire (request hits the MCP server, credentials exchanged, automation
triggered) with no custom per-tool integration. Framed as a product demo.
Source: [Automation Anywhere — March 2026 Product Club: MCP inbound support](https://community.automationanywhere.com/pathfinder-blog-85009/march-2026-product-club-mcp-inbound-support-91245)

### 7.3 Scientific research agent — [illustrative / vendor example]
Agents search 260,000+ bioRxiv preprints, access clinical-trial data, and run single-cell analysis
(10x Genomics Cloud) conversationally — a research-assistant-does-the-literature-review pattern.
Source: [Apigene — 19 MCP use cases](https://apigene.ai/blog/mcp-use-cases)

### 7.4 Multi-tool dev incident context — [illustrative]
The recurring "agent stitches my stack" pattern: pull the Sentry error linked to a Jira ticket, get the
suspect GitHub commit, propose a fix — or ask "why did we change the cache logic?" and have the agent
read PR discussions + commit history + Confluence to answer in one shot.
Sources: [dev.to — automated dev workflow with GitHub/Notion/Jira MCP](https://dev.to/pavanbelagatti/this-is-how-i-automated-my-dev-workflow-with-mcps-github-notion-jira-and-saved-hours-5ag2) ·
[Sentry MCP](https://github.com/getsentry/sentry-mcp)

---

## Patterns: what's actually working vs. demos that don't survive production

**What delivers value (recurring traits of the documented-with-outcomes cases):**

1. **Read-heavy, bounded-scope workflows win.** The strongest documented outcomes — ClickHouse DWAINE
   (~70% of warehouse questions), BigQuery NL→SQL — are **read-only** with hard safety rails (SELECT-only,
   dry-run validation). Writes are where breakage and risk concentrate.
2. **Semantic context is the differentiator, not raw DB access.** DWAINE's leap from "toy" to "primary
   interface" came from the **business glossary**, not the MCP connection itself. Builders consistently
   under-invest here and get disappointing demos.
3. **Human-in-the-loop on the write step.** The Sentry performance bot opens **PRs**, not commits;
   support agents **draft** replies and **escalate**. The successful pattern is "agent does the research
   and proposes; human approves the mutation."
4. **Tool consolidation beats 1:1 API mirroring.** Block's headline lesson: collapsing 30+ mirrored
   GraphQL tools into 2 query tools made the agent dramatically more reliable (1 call vs. 4-6). Thin
   wrappers around every endpoint produce slow, error-prone agents. ([Block](https://engineering.block.xyz/blog/blocks-playbook-for-designing-mcp-servers))
5. **Scheduled/recurring beats one-shot.** The workflows people keep are crons: Monday performance
   triage, daily SEO intel briefs, Monday trend research, weekly intelligence briefs. The agent earns
   its keep by removing a *repeated* chore.
6. **Deterministic interfaces beat vision.** Playwright MCP's accessibility-tree approach (roles/refs,
   no screenshots) is why the browser-verify loop is reliable enough to ship in Copilot.

**What doesn't survive production:**

1. **Security is the #1 killer.** Documented real failures: Asana's June 2025 MCP integration let one
   customer access another's data (multi-tenant access-control bug); Anthropic's reference SQLite MCP
   server had a SQL-injection→prompt-injection flaw and was archived after 5,000+ forks. Trend Micro
   found **492 MCP servers exposed to the internet with zero auth**; research found 7.2% of
   implementations exploitable, 43% with command-injection, 30% allowing SSRF. The fix that survives:
   gateway architecture, centralized auth, every tool call attributed to a human identity, full request
   logging. ([Towards Data Science](https://towardsdatascience.com/the-mcp-security-survival-guide-best-practices-pitfalls-and-real-world-lessons/) ·
   [TrueFoundry](https://www.truefoundry.com/blog/mcp-security-risks-best-practices) ·
   [Brightbean](https://brightbean.xyz/blog/mcp-servers-production-security-scaling-governance/))
2. **Over-broad permissions.** Teams prototype with admin-level API keys and never scope them down —
   the most common path from working demo to breach.
3. **Aspirational "Jarvis" demos.** Elastic explicitly flags many showcased examples (AWS's D&D server,
   "Jarvis" comparisons) as PoC/fun, not production utility. The honest read: a lot of viral MCP demos
   are capability showcases that haven't been load-bearing in a real org.
   ([Elastic](https://www.elastic.co/search-labs/blog/mcp-current-state))
4. **Vendor "example prompts" ≠ deployed workflows.** Much of the CRM/marketing/support material is
   vendor capability marketing (marked [illustrative] above). Real outcomes are scarcer than the volume
   of content implies — treat metrics-free vendor examples as inspiration, not proof.

---

## Most-mentioned MCP servers (what people build workflows around)

From the popularity/usage data ([mcpmanager / PulseMCP](https://mcpmanager.ai/blog/most-popular-mcp-servers/),
[Apigene](https://apigene.ai/blog/mcp-use-cases), and the sources above):

- **Exa** — reportedly the most-used *search* MCP server in 2026.
- **Playwright (Microsoft)** — ~30k+ GitHub stars by mid-2026; the dominant browser-automation server;
  shipped inside GitHub Copilot's coding agent.
- **Context7** — #1 on FastMCP by views/installs (docs/library context for coding agents).
- **GitHub** — the backbone of nearly every dev workflow (PRs, issues, commits).
- **Sentry** — error/performance debugging loops; official, with Seer root-cause analysis.
- **Database/warehouse servers** — Supabase, Snowflake, BigQuery, Databricks, ClickHouse, Postgres;
  database querying is repeatedly cited as *the* top use case.
- **Notion + Linear/Jira (Atlassian)** — the knowledge-ops + project-management pairing.
- **Figma** — design↔code, now with native MCP.
- **Semrush / Ahrefs** — SEO & competitive intel.
- **Zendesk / Intercom** — support triage & KB.
- **Google Workspace (Gmail/Calendar/Drive)** — personal productivity.
- **Salesforce / HubSpot / Apollo** — CRM & outbound.

Ecosystem scale for context: **10,000+ public MCP servers** built since late-2024 (some registries
index ~17,000); the top-50 servers see 622,000+ monthly searches worldwide, and **42 of the top 50 are
used by engineers.** ([mcpmanager](https://mcpmanager.ai/blog/most-popular-mcp-servers/))

---

### Source index
Sentry cookbook · Sentry MCP repo · Continue docs · Microsoft Playwright blog · playwright-mcp repo ·
testomat · testcollab · Block Engineering · ClickHouse (agents blog, warehouse guide) · BigQuery MCP
(ergut, Skyvia) · CloudQuery · Figma blog · Builder.io · claude-talk-to-figma-mcp · Pragmatic Engineer ·
Notion guides · Manus · Apigene · Semrush KB · Ahrefs · Fastio · Huble · Vantage Point · Workato ·
Composio · Vinkius · Microsoft SRE · Elastic · aimaker · Google Workspace docs · MindStudio ·
Automation Anywhere · Towards Data Science · TrueFoundry · Brightbean. (All linked inline above.)
