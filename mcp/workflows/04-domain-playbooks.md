# Role-Based MCP Playbooks

Ready-to-adopt MCP workflow starter kits, organized by **who you are and what job you do**.
Pick your role, wire up the 3-4 servers in its starter stack, and run the workflows.

## How to read this

Each role has a **playbook**: its highest-value MCP workflows, the servers each needs,
a concrete description, an example prompt, and the expected payoff. Then two things that
matter most for adoption: **the one workflow to start with**, and **the main risk/guardrail**.

Every workflow is tagged so you can tell evidence from suggestion:

- **[Documented]** — there is a public write-up, vendor doc, or practitioner post describing
  this exact pattern. Link follows.
- **[Illustrative]** — a plausible, well-grounded setup constructed for this playbook from
  real servers, but not lifted from a single documented case. Treat as a starting template,
  not a proven recipe.

The servers named are real and shipping as of mid-2026. The *combinations and prompts* are
mostly ours unless marked [Documented].

> **One cross-cutting rule, stated once.** Read-only-by-default is the cheapest, highest-value
> guardrail across every role. It breaks the exfiltration leg of the "lethal trifecta"
> (untrusted content + sensitive data access + a way to send data out). Where a role's job
> genuinely needs writes (DevOps remediation, PM ticket creation, support replies), put those
> specific actions behind explicit approval. The per-role "risk/guardrail" notes below say
> exactly where. See [MCP security risks 2026](https://www.practical-devsecops.com/mcp-security-vulnerabilities/)
> and [read-only mitigation](https://deepsense.ai/blog/is-mcp-killing-your-security-a-wake-up-call-for-developers-and-users/).

---

## 1. Software Engineer

The dev's job is a loop: understand code, change it, verify it, ship it, then debug what
broke in production. MCP collapses the context-switching across GitHub, Sentry, the database,
and the browser into one conversation. Practitioner consensus: **three to six servers is the
sweet spot** — tool-selection accuracy collapses past that, it doesn't gracefully degrade.
([nimbalyst](https://nimbalyst.com/blog/best-claude-code-mcp-servers/),
[Bito](https://bito.ai/ai-tools/claude-code-mcp-servers/))

| Workflow | Servers | What it does | Payoff |
|---|---|---|---|
| **Production bug triage** [Documented] | Sentry, GitHub | Pull the error event, stack trace, and frequency from Sentry; locate the offending code and recent commits in GitHub; propose a fix. | Cuts the "open Sentry → copy trace → find file → blame" hop chain to one prompt. ([nimbalyst](https://nimbalyst.com/blog/best-claude-code-mcp-servers/)) |
| **Assigned-bug auto-fix** [Documented] | GitHub, Linear | Read open bugs assigned to you, pick the smallest-scope one, open a branch, write the patch, link the ticket from the PR. | End-to-end task completion without context switching. ([nimbalyst](https://nimbalyst.com/blog/best-claude-code-mcp-servers/)) |
| **PR review prep** [Illustrative] | GitHub | Summarize a PR's diff, surface risky changes, check it against linked issues, draft review comments. | Faster, more consistent reviews; less reviewer fatigue. |
| **DB-backed debugging** [Illustrative] | Postgres (read-only) | Inspect schema, run read queries to confirm what the data *actually* looks like behind a bug, validate a migration's effect. | Stops guessing about data shape; confirms hypotheses against the real DB. |
| **Browser QA / regression** [Documented] | Playwright | Drive a real Chromium instance to reproduce a UI bug, run a click-through smoke test, screenshot before/after, check console errors. | AI-driven QA and visual checks without hand-writing a test first. ([Bito](https://bito.ai/ai-tools/claude-code-mcp-servers/)) |
| **Version-accurate coding** [Documented] | Context7 | Fetch live, version-pinned library docs on demand so generated code matches the API you actually depend on. | Kills the "hallucinated API" failure mode. ([nimbalyst](https://nimbalyst.com/blog/best-claude-code-mcp-servers/)) |

**Example interaction** (production bug triage):
> "Pull the top Sentry issue in the `checkout` project from the last 24h. Get the stack
> trace, find the function in the repo, show me the last 3 commits that touched it, and
> tell me the most likely cause."

**Start with:** Sentry + GitHub. Production bug triage is the single highest-leverage dev
workflow and needs only two servers.

**Main risk/guardrail:** The database server is the sharp edge. Connect it with a
**read-only role** (not your app's write credentials) so no prompt — or poisoned tool
description — can `DELETE` or `DROP`. Skip the Filesystem and Memory MCPs: Claude Code's
built-in file tools and `CLAUDE.md` already cover those, and they only burn your tool budget.
([nimbalyst](https://nimbalyst.com/blog/best-claude-code-mcp-servers/))

---

## 2. Data / Analytics Professional

The analyst's pain is the round trip: write SQL, wait, eyeball results, rewrite, repeat —
plus the dbt layer and the dashboard layer that live in separate tools. MCP lets you ask
questions in English against the warehouse, then trace a number from a dashboard back through
the dbt model to the source table. Note up front: **no warehouse MCP server integrates with
dbt** — you run dbt's official server (60+ tools) *alongside* your warehouse server.
([ChatForest data warehouses](https://chatforest.com/reviews/data-warehouse-lakehouse-mcp-servers/))

| Workflow | Servers | What it does | Payoff |
|---|---|---|---|
| **Natural-language warehouse query** [Documented] | Snowflake / BigQuery / ClickHouse | Translate a business question into SQL, execute read-only, summarize the result. Snowflake-Labs/mcp adds Cortex Search/Analyst; ClickHouse/mcp-clickhouse ships strong read-only safety defaults. | Self-serve answers without hand-writing every query. ([ChatForest](https://chatforest.com/reviews/data-warehouse-lakehouse-mcp-servers/)) |
| **dbt model exploration** [Documented] | dbt (official) | Ask which models feed a metric, what a model's tests assert, where docs live, and what's downstream of a change. | Brings lineage and test context into the conversation. ([ChatForest data & analytics](https://chatforest.com/guides/best-data-analytics-mcp-servers/)) |
| **Cross-source ad-hoc analysis** [Documented] | MotherDuck / DuckDB | Query across local files, S3, and cloud in one place for exploratory work that spans sources. | Best option for cross-source analytical work without an ETL job. ([ChatForest](https://chatforest.com/reviews/data-warehouse-lakehouse-mcp-servers/)) |
| **Metric-to-source trace** [Illustrative] | Warehouse (read-only) + dbt | Take a suspicious dashboard number, walk it back through the dbt DAG to the source table, find where it diverges. | Turns "this number looks wrong" into a root cause in minutes. |
| **Data-quality spot check** [Illustrative] | Warehouse (read-only) | Profile a table — null rates, row counts, distinct values, freshness — before trusting it in a report. | Catches silently-broken pipelines before they reach stakeholders. |
| **Schema-aware query drafting** [Illustrative] | Warehouse (read-only) | Read schema and column types, then draft correct joins and filters without you memorizing the model. | Lowers the SQL skill floor for the whole team. |

**Example interaction** (natural-language query, read-only):
> "In our analytics warehouse, what were the top 10 referral sources by signups last month,
> and how does that compare to the prior month? Read-only — just show me the SQL and the
> result."

**Start with:** your warehouse server (Snowflake, BigQuery, or ClickHouse) in read-only mode.
One server, immediate self-serve querying. Add dbt second.

**Main risk/guardrail:** **Read-only role, scoped to non-sensitive tables.** Beyond blocking
writes, deny the agent access to columns holding PII (emails, names, payment data) — an analyst
exploring freely plus an LLM that summarizes is exactly the data-exfiltration risk read-only
was designed to contain. Prefer servers with explicit read-only modes (ClickHouse's defaults,
Snowflake read-only configs). ([ChatForest](https://chatforest.com/reviews/data-warehouse-lakehouse-mcp-servers/))

---

## 3. Product Manager

This is the AI-native PM theme in practice: the PM's edge is no longer writing the ticket, it's
**holding the whole context at once** — delivery state (Linear/Jira), design (Figma), knowledge
(Notion/Confluence), and user evidence (analytics/feedback) — and synthesizing across them.
A documented workflow turns a complex Figma file into 6 epics and 20 user stories in ~10 minutes,
hands-free. ([Product Compass](https://www.productcompass.pm/p/mcp-case-study-jira-figma),
[prodmgmt.world](https://www.prodmgmt.world/resources/mcp))

| Workflow | Servers | What it does | Payoff |
|---|---|---|---|
| **Figma → epics & stories** [Documented] | Figma, Jira (or Linear) | Read a design file, decompose it into epics and well-formed user stories, create them in the tracker. | ~6 epics + 20 stories in ~10 min, no keyboard. ([Product Compass](https://www.productcompass.pm/p/mcp-case-study-jira-figma)) |
| **Impact-ranked bug triage** [Documented] | Linear, PostHog (analytics) | Pull the top open bugs, check each against user-impact data, draft a prioritized ticket for the highest-impact one. | Prioritization grounded in real usage, not vibes. ([prodmgmt.world](https://www.prodmgmt.world/resources/mcp)) |
| **Context-rich PRD draft** [Documented] | Linear, GitHub, Slack, Notion | Pull related tickets, the relevant implementation from GitHub, a project Slack channel's summary, and draft a PRD — in one prompt. | Days of context-gathering compressed into one pass. ([prodmgmt.world](https://www.prodmgmt.world/resources/mcp)) |
| **Knowledge-base synthesis** [Illustrative] | Notion / Confluence | Pull prior decisions, specs, and research into a prompt so new docs build on what the team already concluded. | Stops re-litigating settled decisions. |
| **Competitive / market research** [Illustrative] | Web fetch/search + Notion | Gather competitor feature and pricing changes, write a structured comparison into the workspace. | Always-current competitive intel without manual trawling. |
| **Status roll-up** [Illustrative] | Linear/Jira + Slack | Summarize sprint progress and blockers from the tracker, post a digest to the team channel. | 4-6 hrs/week saved on PM admin. ([MindStudio](https://www.mindstudio.ai/blog/mcp-servers-claude-code-business-automation)) |

**Example interaction** (impact-ranked triage):
> "Pull the top 5 open bugs from Linear. For each, check PostHog for how many users hit it.
> Then draft a prioritized ticket for the highest-impact one with a clear repro and acceptance
> criteria."

**Start with:** Linear (or Jira) alone — read tickets, draft and create well-formed issues.
It's the spine everything else hangs off, and it delivers value with a single server.

**Main risk/guardrail:** **Scope write permissions on ticket creation.** An agent that can
mass-create epics and stories can also spam your backlog or overwrite priorities. Keep creation
in a review-before-create posture (draft → you confirm → create), and start with read-only on
Figma/Notion so the agent gathers context but only *writes* where you've deliberately allowed it.

---

## 4. DevOps / Platform / SRE

Highest blast radius of any role here. MCP's promise for SRE is real — query logs, correlate
metrics with deploys, and get an evidence-backed root cause — but the whole discipline is built
on **governed tools + human approval**. The documented pattern (AWS DevOps Agent, MWAA +
CloudWatch) is explicit: read/diagnose freely, but every write action sits behind approvals,
allowlists, validation, and audit.
([AWS agentic SRE](https://aws.amazon.com/blogs/devops/building-an-end-to-end-agentic-sre-using-aws-devops-agent/),
[safe K8s troubleshooting](https://www.linuxinsider.com/story/using-mcp-for-safe-autonomous-kubernetes-troubleshooting-177618.html))

| Workflow | Servers | What it does | Payoff |
|---|---|---|---|
| **Incident root-cause** [Documented] | Kubernetes, CloudWatch/monitoring, GitHub | Query logs and metrics, retrieve deployment history, correlate metric anomalies with deploy events, build topology, propose a root cause + mitigation. | Faster MTTR with evidence-backed analysis. ([AWS](https://aws.amazon.com/blogs/devops/building-an-end-to-end-agentic-sre-using-aws-devops-agent/)) |
| **Governed K8s troubleshooting** [Documented] | Kubernetes (read), approval layer | Diagnose a failing pod/deployment with read-only cluster access; any `kubectl` write or restart pauses for human approval. | Safe autonomy in production. ([LinuxInsider](https://www.linuxinsider.com/story/using-mcp-for-safe-autonomous-kubernetes-troubleshooting-177618.html)) |
| **Approval-gated pipeline action** [Documented] | MWAA / CI-CD, CloudWatch | Trigger a DAG / re-run a job — but the write action sits behind approval, allowlist, validation, and audit logging. | Self-service ops without removing the safety net. ([DZone](https://dzone.com/articles/agentic-dataops-with-guardrails-mcp-mwaa)) |
| **Cloud resource inspection** [Illustrative] | AWS (read-only), Cloudflare | Read-only audit of resources, security-group drift, DNS/WAF config — answer "what's deployed and is it configured right?" | Fast posture checks without console clicking. |
| **Deploy-aware on-call** [Illustrative] | GitHub, monitoring | At alert time, surface what shipped recently and whether the timing lines up with the anomaly. | Cuts the "what changed?" question to seconds. |
| **Runbook execution assist** [Illustrative] | Monitoring + approval layer | Walk an on-call engineer through a runbook, gathering each diagnostic, proposing each remediation step for explicit go/no-go. | Codifies tribal knowledge; keeps a human in the loop. |

**Example interaction** (governed troubleshooting):
> "The `payments` deployment is crash-looping. Read the pod logs and recent events, correlate
> with the last deploy from GitHub, and tell me the likely cause. **Do not** restart anything —
> propose the fix and wait for my approval."

**Start with:** read-only monitoring + Kubernetes diagnostics. Get the root-cause workflow
working with **zero write access** first; earn the right to add gated writes later.

**Main risk/guardrail:** **Approval gates on every create/modify/delete/restart, plus audit
logging.** This is the role where the cross-cutting "read-only default" becomes non-negotiable:
diagnosis is read-only; remediation pauses for a human. Use allowlists for which actions are
even *proposable*, and log every tool call.
([AWS MCP security](https://builder.aws.com/content/34ehRAhM6rygBjYNee6sZ6AjPSi/model-context-protocol-mcp-security-implementation-guide-for-aws))

---

## 5. Marketing / Growth

Marketing ops is a data-routing job: pull numbers from analytics and the CRM, turn them into
content and briefs, and watch what competitors are doing. The big three vendors now ship
official servers — Google's analytics MCP covers the full GA4 stack, HubSpot's covers the
whole CRM — which made MCP-native marketing workflows agency-viable in 2026.
([Knak](https://knak.com/blog/popular-mcp-servers-for-marketing-operations/),
[Wix](https://www.wix.com/seo/learn/resource/how-to-use-mcp-in-marketing))

| Workflow | Servers | What it does | Payoff |
|---|---|---|---|
| **Plain-English analytics** [Documented] | Google Analytics (GA4, official) | Ask traffic-by-source, landing-page performance, bounce, and engagement questions in natural language. | No more wrestling the GA4 UI for routine answers. ([Wix](https://www.wix.com/seo/learn/resource/how-to-use-mcp-in-marketing)) |
| **CRM querying & ops** [Documented] | HubSpot (official) / Salesforce | Read and act on contacts, deals, lifecycle stages, workflows, and reporting. | Pipeline answers and updates conversationally. ([Knak](https://knak.com/blog/popular-mcp-servers-for-marketing-operations/)) |
| **Data-backed content briefs** [Documented] | SEO/keyword + SERP tools | Generate detailed briefs with target keywords, competitor gaps, and suggested headings from real keyword/SERP data. | Briefs in minutes, grounded in data. ([Wix](https://www.wix.com/seo/learn/resource/how-to-use-mcp-in-marketing)) |
| **Cross-channel performance roll-up** [Illustrative] | GA4 + ad platforms (Google/Meta Ads) | Pull spend and conversions across channels into one weekly performance summary. | One report instead of five dashboards. |
| **Content publishing** [Illustrative] | Notion / Webflow / WordPress | Draft, format, and stage content into the CMS from an outline. | Shortens idea → published. |
| **Competitive / market intel** [Illustrative] | Web fetch/search + Notion | Track competitor messaging, pricing, and launches; write a living comparison doc. | Always-current intel without manual monitoring. |

**Example interaction** (analytics + content):
> "Using GA4, find my 5 top-performing blog posts by engaged sessions last quarter. For the
> theme they share, draft a content brief — target keywords, competitor gaps, and an outline."

**Start with:** Google Analytics (GA4) MCP. It's official, fully managed, and answers the
questions marketers ask hourly.

**Main risk/guardrail:** **Prompt injection on fetched external content**, plus PII in the CRM.
Competitive-intel and content workflows pull untrusted web pages — a poisoned page can try to
steer the agent. Run a guardrail that strips injection patterns ("ignore previous instructions")
from fetched content before it reaches the model, and keep CRM access read-only until you
specifically need to write.
([Practical DevSecOps](https://www.practical-devsecops.com/mcp-security-vulnerabilities/))

---

## 6. Customer Support / Success

Support is context retrieval under time pressure: who is this customer, what's their history,
what does the knowledge base say, and what's the right reply. The Zendesk MCP server (reported
~29 tools) gives an agent search/retrieve with full ticket context, profile lookups, internal
notes, public replies, and conversation summaries; Intercom's covers chat threads, user records,
and help-center analytics.
([Vinkius](https://blog.vinkius.com/customer-support-mcp-servers-zendesk-freshdesk-intercom),
[Composio Zendesk](https://composio.dev/toolkits/zendesk),
[mcpbundles](https://www.mcpbundles.com/skills/zendesk))

| Workflow | Servers | What it does | Payoff |
|---|---|---|---|
| **Ticket triage & routing** [Documented] | Zendesk / Intercom | Classify incoming tickets by intent, priority, and tier; route to the right team. | Faster, more consistent routing. ([Vinkius](https://blog.vinkius.com/customer-support-mcp-servers-zendesk-freshdesk-intercom)) |
| **KB-grounded reply drafting** [Documented] | Zendesk + help center | Draft a reply using ticket history, macros, and help-center content. | Agents draft from canon, not memory. ([Vinkius](https://blog.vinkius.com/customer-support-mcp-servers-zendesk-freshdesk-intercom)) |
| **Long-thread summarization** [Documented] | Zendesk | Summarize a long, multi-message ticket so the next agent gets instant context. | No re-reading 40-message threads. ([Composio](https://composio.dev/toolkits/zendesk)) |
| **Support analytics in English** [Documented] | Zendesk analytics (e.g. Improvado) | Ask about ticket trends, SLA breaches, CSAT, and agent performance in plain English. | Self-serve reporting for team leads. ([Improvado](https://improvado.io/mcp/zendesk)) |
| **Account / entitlement lookup** [Illustrative] | CRM/billing (read-only) | Pull plan, entitlements, and recent activity to answer "what is this customer allowed to do?" | Fewer escalations to other teams. |
| **Escalation packaging** [Illustrative] | Zendesk + GitHub/Linear | Bundle a bug report with repro and context, file it for engineering, link it back to the ticket. | Clean handoffs; engineering gets what it needs. |

**Example interaction** (KB-grounded reply):
> "Pull ticket #4821, summarize the customer's issue, find the relevant help-center article,
> and draft a reply. Flag it for my review before sending — don't send."

**Start with:** Zendesk (or Intercom) MCP. One server unlocks triage, drafting, and
summarization immediately.

**Main risk/guardrail:** **Scope account lookups and gate the "send."** Support data is
PII-dense — limit the agent to the requesting customer's records, not the whole base. And keep
public replies in **draft-for-review**, not auto-send: a wrong or injected reply goes straight
to a customer. Watch for prompt injection inside ticket bodies themselves (a hostile customer
can embed instructions).

---

## 7. Founder / Generalist / Solo Operator

You do everything, so your stack has to be lean and cover the widest surface with the fewest
servers. The pattern that works: a **comms-and-knowledge loop** — read from one app, act in
another, write the result to a third. Reported savings of 4-6 hrs/week on the admin layer.
The hard constraint to internalize: **Claude is reactive, not proactive — every automation needs
an explicit trigger.**
([MindStudio](https://www.mindstudio.ai/blog/mcp-servers-claude-code-business-automation),
[saranfn](https://saranfn.substack.com/p/become-a-power-user-of-claude-slack))

| Workflow | Servers | What it does | Payoff |
|---|---|---|---|
| **Email → summary → channel** [Documented] | Gmail, Slack, Notion | Pull an email thread, summarize it, post the summary to the right Slack channel and log it in Notion. | A full comms loop, automated. ([MindStudio](https://www.mindstudio.ai/blog/connect-claude-code-notion-gmail-mcp-servers)) |
| **Second-brain knowledge ops** [Documented] | Notion (official) | Capture notes, retrieve prior decisions, and keep the workspace as the single source of truth across all your hats. | One place that actually stays current. ([Notion MCP](https://developers.notion.com/docs/mcp)) |
| **Research → structured doc** [Illustrative] | Web fetch/search + Notion | Research a market, competitor, or hire; write a structured brief into the workspace. | Days of legwork into a single pass. |
| **Inbox / calendar triage** [Illustrative] | Gmail, Calendar | Summarize what needs a reply, draft responses, surface scheduling conflicts. | The morning-admin block, compressed. |
| **Lightweight CRM/pipeline** [Illustrative] | Notion (or HubSpot free) | Track leads, deals, and follow-ups conversationally without a heavyweight CRM. | CRM discipline without CRM overhead. |
| **Cross-app status digest** [Illustrative] | Slack + Notion + tracker | One prompt that pulls the week's state across tools into a digest you actually read. | Stay on top of everything in 5 minutes. |

**Example interaction** (comms loop):
> "Read the latest thread from [investor] in Gmail, summarize the asks and decisions, post a
> short update to #fundraising in Slack, and log the action items in my Notion CRM."

**Start with:** Notion as the hub, plus one comms server (Gmail or Slack). The
read-summarize-write loop is the founder's highest-leverage automation and needs just two servers.

**Main risk/guardrail:** **Prompt injection on everything you fetch, plus send-scope on comms.**
A solo operator wiring Gmail + Slack + web research is assembling exactly the lethal trifecta:
untrusted content + access to your accounts + a way to send messages. Keep email and Slack in
**draft-for-review** rather than auto-send, and treat any fetched web/email content as hostile
until proven otherwise. Stay lean — every extra server widens the attack surface and dilutes
tool-selection accuracy.

---

## Starter stacks

The minimum viable set per role — wire these up first, add more only when a workflow demands it.
(Three to six servers total is the consensus sweet spot;
[nimbalyst](https://nimbalyst.com/blog/best-claude-code-mcp-servers/).)

| Role | Starter servers (3-4) | Start-with workflow | Primary guardrail |
|------|----------------------|---------------------|-------------------|
| **Software Engineer** | Sentry, GitHub, Postgres (read-only), Playwright | Production bug triage (Sentry + GitHub) | Read-only DB role; skip Filesystem/Memory MCPs |
| **Data / Analytics** | Warehouse server (Snowflake/BigQuery/ClickHouse, read-only), dbt | NL warehouse query (read-only) | Read-only role, no PII columns |
| **Product Manager** | Linear (or Jira), Figma, Notion, analytics (PostHog/Amplitude) | Impact-ranked bug triage | Review-before-create on ticket writes |
| **DevOps / Platform / SRE** | Kubernetes (read), CloudWatch/monitoring, GitHub, + approval layer | Incident root-cause (read-only first) | Approval gates on all writes + audit log |
| **Marketing / Growth** | Google Analytics (GA4), HubSpot/Salesforce, SEO/SERP, Notion/CMS | Plain-English GA4 analytics | Injection guard on fetched content; CRM read-only |
| **Customer Support / Success** | Zendesk (or Intercom), help center/KB, CRM (read-only) | KB-grounded reply drafting | Scope account lookups; draft-for-review on replies |
| **Founder / Solo Operator** | Notion, Gmail, Slack, web fetch/search | Email → summary → channel loop | Injection guard + send-scope; stay lean |

---

## Sources

- [nimbalyst — Best Claude Code MCP Servers (2026)](https://nimbalyst.com/blog/best-claude-code-mcp-servers/)
- [Bito — Top MCP servers for Claude Code](https://bito.ai/ai-tools/claude-code-mcp-servers/)
- [ChatForest — Data Warehouse & Lakehouse MCP Servers](https://chatforest.com/reviews/data-warehouse-lakehouse-mcp-servers/)
- [ChatForest — Best Data & Analytics MCP Servers](https://chatforest.com/guides/best-data-analytics-mcp-servers/)
- [Product Compass — MCP case study: Figma → Jira](https://www.productcompass.pm/p/mcp-case-study-jira-figma)
- [prodmgmt.world — MCPs for Product Managers](https://www.prodmgmt.world/resources/mcp)
- [AWS — Building an end-to-end agentic SRE using AWS DevOps Agent](https://aws.amazon.com/blogs/devops/building-an-end-to-end-agentic-sre-using-aws-devops-agent/)
- [LinuxInsider — Using MCP for Safe, Autonomous Kubernetes Troubleshooting](https://www.linuxinsider.com/story/using-mcp-for-safe-autonomous-kubernetes-troubleshooting-177618.html)
- [DZone — Agentic DataOps with Guardrails: MCP + MWAA](https://dzone.com/articles/agentic-dataops-with-guardrails-mcp-mwaa)
- [AWS — MCP Security Implementation Guide](https://builder.aws.com/content/34ehRAhM6rygBjYNee6sZ6AjPSi/model-context-protocol-mcp-security-implementation-guide-for-aws)
- [Knak — Popular MCP Servers for Marketing Operations](https://knak.com/blog/popular-mcp-servers-for-marketing-operations/)
- [Wix — 9 ways to use MCP in your marketing stack](https://www.wix.com/seo/learn/resource/how-to-use-mcp-in-marketing)
- [Vinkius — Customer Support MCP Servers](https://blog.vinkius.com/customer-support-mcp-servers-zendesk-freshdesk-intercom)
- [Composio — Zendesk MCP](https://composio.dev/toolkits/zendesk)
- [Improvado — Zendesk MCP analytics](https://improvado.io/mcp/zendesk)
- [MindStudio — MCP servers for business automation](https://www.mindstudio.ai/blog/mcp-servers-claude-code-business-automation)
- [MindStudio — Connect Claude Code to Notion, Gmail](https://www.mindstudio.ai/blog/connect-claude-code-notion-gmail-mcp-servers)
- [Notion — Official MCP docs](https://developers.notion.com/docs/mcp)
- [Practical DevSecOps — MCP Security Vulnerabilities (2026)](https://www.practical-devsecops.com/mcp-security-vulnerabilities/)
- [deepsense.ai — Is MCP killing your security?](https://deepsense.ai/blog/is-mcp-killing-your-security-a-wake-up-call-for-developers-and-users/)
