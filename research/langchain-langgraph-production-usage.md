# How Companies Use LangChain & LangGraph in Production

> Research note, August 2026. Compiled from LangChain's published case studies
> ("Built with LangGraph", "Top LangGraph Agents in Production"), the LangChain
> 2026 State of Agent Engineering survey (1,300+ practitioners, fielded Nov–Dec
> 2025), and third-party production analyses. Audience: Glific/Tech4Dev — what
> real production deployments look like, how autonomous they actually are, and
> which patterns transfer to a customer-facing chatbot platform. Companion:
> [`langchain-langgraph-chat-with-data.md`](./langchain-langgraph-chat-with-data.md).

---

## Table of Contents

1. [Headline: Production Is Real, Full Autonomy Is Not](#1-headline-production-is-real-full-autonomy-is-not)
2. [Named Case Studies](#2-named-case-studies)
3. [The Klarna Epilogue: What Over-Automation Taught the Industry](#3-the-klarna-epilogue-what-over-automation-taught-the-industry)
4. [Survey Data: State of Agent Engineering 2026](#4-survey-data-state-of-agent-engineering-2026)
5. [The Three Tiers of Production Autonomy](#5-the-three-tiers-of-production-autonomy)
6. [Architecture Patterns That Recur](#6-architecture-patterns-that-recur)
7. [What Transfers to Glific](#7-what-transfers-to-glific)
8. [Sources](#8-sources)

---

## 1. Headline: Production Is Real, Full Autonomy Is Not

As of mid-2026, 20+ named enterprises run LangGraph in production, and 57% of
surveyed practitioners have agents in production (up from 51% a year earlier).
But reading the case studies closely, **almost none are truly "auto mode"**:
every flagship deployment is either internal-facing, human-supervised, or scoped
to actions whose failures are cheap and externally verified. The industry's
production consensus is *bounded autonomy with escalation*, not free-running
agents.

---

## 2. Named Case Studies

### Klarna — customer support assistant (customer-facing, highest autonomy)
- **What:** AI assistant handling customer support for **85M active users**,
  built on LangGraph + LangSmith.
- **Outcome:** customer resolution time reduced by **80%**.
- **Autonomy:** talks directly to end customers — the most autonomous named
  deployment. See §3 for the critical epilogue.

### Uber — code migration & unit-test generation (internal, high autonomy)
- **What:** Developer Platform team built a **network of agents** that automate
  unit-test generation during large-scale code migrations.
- **Outcome:** ~**21,000 developer hours saved**.
- **Why high autonomy is safe here:** output is code that still passes review,
  compilation, and test runs — verification is external and deterministic.
  Failure is cheap and caught before it matters.

### LinkedIn — hiring assistant (internal, human-consumed output)
- **What:** AI recruiter with conversational search and candidate matching,
  built as a **hierarchical agent system** (supervisor delegating to
  sub-agents).
- **Autonomy:** a recruiter reviews every match; the agent never autonomously
  rejects or advances candidates.

### LinkedIn — SQL Bot (internal, human-consumed output)
- **What:** **multi-agent** text-to-SQL system letting employees across
  functions query data in natural language.
- **Oversight:** permission-scoped data access; a human reads the results
  before acting on them. (Deep-dive on this system in
  [`langchain-langgraph-chat-with-data.md`](./langchain-langgraph-chat-with-data.md).)

### Replit — Replit Agent (user-facing, HITL by design)
- **What:** code-generation agent, **multi-agent architecture with
  human-in-the-loop** oversight built in from the start.
- **Notable:** Replit explicitly framed human oversight as central to the
  design, not a temporary training wheel.

### Elastic — AI Assistant & attack discovery (internal/analyst-facing)
- **What:** orchestrated agents for security threat detection and SecOps.
- **Notable:** **migrated from plain LangChain to LangGraph** as features grew —
  a common trajectory (prebuilt loop → explicit graph when control is needed).
- **Outcome:** reduced labor-intensive SecOps tasks; analysts stay in the loop.

### AppFolio — Realm-X property-management copilot (user-facing copilot)
- **What:** decision-support copilot; migrated to LangGraph for a
  "controllable agent architecture."
- **Outcome:** response accuracy **2×**, property managers save **10+ hours a
  week**.
- **Autonomy:** copilot pattern — suggests, human decides.

---

## 3. The Klarna Epilogue: What Over-Automation Taught the Industry

Klarna is the most-cited autonomous chatbot deployment in the industry — and in
**mid-2025 it publicly walked back its automation push**, re-introducing human
agents after acknowledging that over-automating support degraded quality and
customer trust.

The lesson from the flagship is therefore **not "full auto works."** It is:

- automate the routine tier, keep humans reachable;
- enforce a quality floor with evals before scaling autonomy;
- design escalation paths as a first-class feature, not a fallback.

This matters more for Glific than any framework detail: Klarna's win (and its
correction) came from conversation design, quality measurement, and escalation —
all framework-portable.

---

## 4. Survey Data: State of Agent Engineering 2026

LangChain's survey of 1,300+ practitioners (fielded Nov–Dec 2025):

- **57% have agents in production**; 30.4% actively building toward it. Large
  enterprises lead adoption.
- **Top use cases:** customer service **26.5%**, research & data analysis
  **24.4%**, internal workflow automation **18%** — plus QA testing,
  knowledge-base search, text-to-SQL, demand planning.
- **Quality is the #1 production blocker (32%)** — hallucinations, consistency,
  tone. Not cost, not infrastructure.
- **The oversight gap:** **89%** have observability (62% with step-level
  tracing), but only **52%** run offline evals and **37%** online evals — and
  ~**60%** of eval-running orgs still rely on human review as the eval method.
  Teams watch their agents far more than they trust them.
- The report publishes **no data** on autonomy levels, approval gates, or
  write-action policies — the industry hasn't standardized "how autonomous" yet.
- A meaningful minority still use agents only for LLM chat or coding
  assistance — "agentic everything" remains early.

---

## 5. The Three Tiers of Production Autonomy

Across the case studies and survey, production deployments cluster into three
tiers:

| Tier | Pattern | Who runs it this way | Why it's safe |
|---|---|---|---|
| 1. Fully autonomous | Agent acts without review | Uber (test gen) | Failures cheap + externally verified (compile/test/review) |
| 2. Autonomous with escalation | Agent handles routine tier, hands off on uncertainty | Klarna (support chat) | Human escalation path + quality floor |
| 3. HITL by design | Agent proposes, human decides | LinkedIn (hiring, SQL), Replit, AppFolio, Elastic | Consequential or write actions gated on approval |

Customer-facing chat — Glific's category — lives in **tier 2**, and tier 2 is
where the industry's scar tissue is (see §3).

---

## 6. Architecture Patterns That Recur

- **Supervisor / hierarchical multi-agent** (LinkedIn hiring, Uber): a routing
  agent delegates to specialized sub-agents. LangGraph's graph model is used to
  make the delegation explicit.
- **Prebuilt loop → explicit graph migration** (Elastic, AppFolio): teams start
  on LangChain's prebuilt agent, then migrate to LangGraph graphs when they need
  controllability. LangChain 1.0 formalized this: `create_agent` now runs *on*
  the LangGraph runtime, so the migration is a gradient, not a rewrite.
- **Durable execution & checkpointing** as the production selling point: resume
  from checkpointed state after crashes/restarts, for runs of hours or days.
  Most cited for long-running internal workflows, less load-bearing for
  short-turn chat.
- **Human-in-the-loop as first-class** (Replit, AppFolio; LangChain 1.0's
  `HumanInTheLoopMiddleware`, LangGraph interrupts): approval gates are a
  framework feature, and the strongest teams design around them rather than
  bolting them on.
- **Observability-first, evals-lagging** (survey-wide): near-universal LangSmith
  tracing, but evals adoption trails ~35 points behind. The teams that close
  this gap are the ones that scale autonomy safely.

---

## 7. What Transfers to Glific

1. **The pattern to copy is tier 2:** autonomous routine handling + uncertainty
   escalation + human handoff. Every element is framework-portable — none of it
   requires LangGraph.
2. **Quality evals are the missing half of production readiness** (89%
   observability vs 52% evals). Investment in an eval suite over real Glific
   conversation flows is what the strongest production teams have and most
   don't.
3. **The Klarna correction is the cautionary tale** for a 150+ client chatbot
   platform: scale autonomy behind a measured quality floor, and treat human
   escalation as a product feature NGO clients will demand.
4. **Multi-agent/supervisor patterns** are proven but appear almost exclusively
   in internal tools with human-consumed output — not a near-term need for
   conversational flows.

---

## 8. Sources

- [Built with LangGraph (case studies)](https://www.langchain.com/built-with-langgraph)
- [Top 5 LangGraph agents in production](https://www.langchain.com/blog/top-5-langgraph-agents-in-production-2024)
- [State of Agent Engineering 2026](https://www.langchain.com/state-of-agent-engineering)
- [Analysis: quality, not cost, is killing agents in production](https://www.nocode.tech/article/langchain-report-quality-not-cost-killing-ai-agents)
- [LangGraph in production: architecture, costs & outcomes](https://www.alphabold.com/langgraph-agents-in-production/)
- [Enterprise LangGraph adoption analysis](https://www.webpronews.com/open-source-ai-agents-surge-in-enterprises-as-langgraph-leads-production-shift/)
- [LangChain & LangGraph 1.0 announcement](https://www.langchain.com/blog/langchain-langgraph-1dot0)
