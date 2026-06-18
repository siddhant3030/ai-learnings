## 4. Prompt Libraries for PMs

The most battle-tested prompts, organized by PM workflow. These are sourced from practitioners, not generic guides.

### 4.1 User Research & Discovery

See the detailed prompts in Section 1.1 and 1.2.

**Why the research synthesis prompt works:** The role tag sets expertise level. The XML-structured input separates context from task from output format, which Claude processes as semantic containers with higher fidelity than plain prose. The "behavioral vs. stated preference" constraint forces Claude to do the harder analytical work rather than just summarizing.

**When to use it:** After conducting 5+ interviews; before writing any PRDs; when synthesizing survey or support ticket data.

**How experienced PMs modify it:** Add a section for "competitive implications" — asking Claude to flag any user needs that suggest an unserved market segment. Add a "confidence level" to each finding.

---

### 4.2 PRD Generation

See full prompt in Section 1.6.

**Why it works:** The `<quality_bar>` tag forces Claude to flag knowledge gaps as `[NEEDS INPUT]` rather than hallucinate plausible-sounding content. This is the critical safety mechanism for PM documents.

**When to use it:** After discovery synthesis; when writing the first draft of any feature spec.

**How experienced PMs modify it:** Add their team's specific PRD template structure. Add a section for "anti-goals" — explicitly what the feature will not do.

---

### 4.3 Competitive Analysis (3-Prompt Chain)

See full prompt chain in Section 1.3.

**Why it works:** The three-prompt chain separates extraction from interpretation from prescription. Doing all three in one prompt produces muddled output; separating them forces Claude to complete each stage before moving to the next.

**When to use it:** Quarterly competitive reviews; before any positioning or pricing decisions.

**How experienced PMs modify it:** After Prompt 1, ask Claude to rate its confidence in each extraction on a 1-3 scale. Prompts based on sparse or ambiguous data should be treated skeptically.

---

### 4.4 Strategy & Assumption Testing

See assumption testing prompt in Section 1.4.

**Why it works:** Framing Claude as an adversarial evaluator (not a helper) produces qualitatively different output. The "failure scenario" constraint forces Claude to reason through second-order consequences.

**When to use it:** Before quarterly planning; before major strategic pivots; when a strategy document has been through 2+ internal drafts.

**How experienced PMs modify it:** Run this prompt after an internal review, not instead of one. Use it to identify blind spots the team collectively missed.

---

### 4.5 Roadmap & Prioritization

See RICE and quarterly roadmap prompts in Section 1.5.

**Why the roadmap narrative prompt works:** Executives evaluate roadmaps through the lens of: revenue impact, customer retention, market competitiveness, resource efficiency. The prompt forces alignment to those criteria and demands that a tradeoff be articulated — the most important thing most roadmap documents omit.

**When to use it:** When preparing roadmap communication to leadership; when adapting a technical roadmap into a business narrative.

---

### 4.6 Stakeholder Communication

See status update prompt in Section 1.7.

**Difficult message prompts** (from Evelance.io — sourced from production PM use):

*Pushback message:*
```
<context>Relationship: [your relationship, their seniority]. Their request: [what they asked]. Why problematic: [your reasoning].</context>
<task>Draft a message declining or redirecting this request. The reader should finish feeling: respected, heard, and clear on reasoning. Offer an alternative addressing their underlying need without their specific implementation.</task>
<constraints>Direct but not abrasive. Do not hedge with "maybe" or "I think." State the position clearly.</constraints>
```

*Delay communication:*
```
<context>Stakeholders: [who needs to know]. Original timeline: [date]. New timeline: [date]. Reason: [explanation]. Impact: [what this changes for dependent teams or customers].</context>
<task>Draft a message communicating this delay. Lead with the new timeline. Explain cause in 1-2 sentences without excessive justification. Specify impact on dependent workstreams. Close with what you are doing to prevent further slippage.</task>
```

---

### 4.7 Data Analysis

See funnel analysis and A/B test prompts in Section 1.8.

**Metrics narrative prompt:**
```
<metrics_data>[Weekly/monthly metrics: key numbers, trends, comparisons to targets]</metrics_data>
<task>Transform into a narrative for product review. (1) Identify 3 most noteworthy changes; (2) Explain what likely drove each change; (3) Flag any metric trending in a concerning direction even if not at threshold; (4) Recommend 1-2 specific actions.</task>
<output_format>3 paragraphs with numbers embedded in narrative. No bullet points. Written for an audience reviewing these numbers monthly.</output_format>
```

---

### 4.8 Launch Planning

**Launch readiness checklist generation:**
```
<context>Product: [description]. Launch type: [beta/GA/major release]. Audience: [customer segment]. Launch date: [date]. Dependent teams: [list].</context>
<task>Generate a launch readiness checklist organized by team (engineering, design, marketing, support, sales, legal). For each team: list their specific launch-day tasks; identify any tasks that depend on another team completing first; flag any tasks with lead times exceeding 1 week as "critical path".</task>
```

---
