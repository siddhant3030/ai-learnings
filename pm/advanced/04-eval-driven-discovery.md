# Eval-Driven Product Discovery
## The Most Important New PM Skill for AI Products

> "Each new model release changes what's possible, and in building Datadog's Bits AI SRE agent
> we study its strengths and failure modes through offline evaluation on real-world production
> incidents."
>
> — Kai Xin Tai, Senior PM, Datadog

---

## Who This Guide Is For

You are a PM working on an AI product, or a PM who wants to understand why AI products behave
the way they do. You have heard the word "evals" but you associate it with engineers running
test scripts — not with product discovery.

This guide will change that.

By the end you will know how to:
- Build an eval set from real user interactions (no code required)
- Use eval failures as a product discovery signal
- Present eval results in planning meetings and PRDs
- Run evals continuously as part of your product rhythm

This is a hands-on workshop. Every section has a template you can copy today.

---

## Section 1: Why Evals Are the New User Stories

### The Traditional PM Artifact: User Stories

User stories describe intended behavior from the user's point of view:

> "As an on-call engineer, I want to see the probable root cause of an alert so that I can
> respond faster."

User stories are useful. They align the team on what to build. But they describe *intent*, not
*reality*. Once you ship the feature, the user story cannot tell you whether the AI is actually
doing what you hoped.

### The AI PM Artifact: Evals Measure Actual Behavior

An eval for the same feature looks like this:

| Input | Expected Output | Grading Criteria |
|-------|----------------|-----------------|
| Alert: "CPU spike 94% on prod-db-3, duration 12 min, coincides with deploy at 14:32" | Root cause: recent deploy. Recommendation: rollback or investigate deploy artifact. | Must name the deploy as probable cause. Must recommend an action. Must not hallucinate services not mentioned. |

The user story says what you want. The eval tells you whether you have it.

### Why You Cannot Write a User Story for AI Behavior

Traditional software is deterministic. You write a user story, an engineer implements it, and
a QA engineer writes a test that either passes or fails. The test is binary.

AI output is probabilistic. The same input can produce different outputs. "Good" and "bad" are
not binary — they exist on a spectrum, they depend on context, and they change as the model
changes. A user story captures none of this.

The discipline that captures it is evaluation.

### The Kai Xin Tai Insight: Offline Evals on Production Incidents

At Datadog, Kai Xin Tai and her team build Bits AI SRE — an AI agent that acts as an on-call
engineer during incidents. Their discovery method is not user interviews alone. It is running
their agent in "offline" mode against real production incidents that already happened, then
grading the agent's responses against what actually worked.

This approach gives them a dataset of real inputs (actual incidents) with known ground truth
(what the on-call engineers actually did). Every offline eval run is a discovery session:
they learn not just whether the agent works, but *where* it fails and *why*.

The product roadmap follows the failure clusters.

### Concretely: The Difference Matters

| Traditional PM | Eval-Driven PM |
|---------------|---------------|
| "Users want fast incident response" | "Agent resolves P1 incidents correctly in <2 min on 80% of cases" |
| "The summary should capture key points" | "Eval pass rate on summary accuracy: 72%. Target: 90% by Q2" |
| "Users want relevant recommendations" | "Recommendation eval: 61% match expert judgment. Gap = feature gap" |
| "The search should return good results" | "Eval on top-3 relevance: 58% pass. Failing cluster: multi-word queries" |

The left column is a wish. The right column is a product spec.

---

## Section 2: What Evals Are and How They Work

### The Simple Mental Model

An eval is a test case for AI behavior.

Just as a software test checks "given input X, does function Y return Z?", an eval checks
"given input X, does the AI produce an output that meets quality criteria Z?"

The difference: software tests have one correct answer. Evals have *grading criteria* — a rubric
that defines what "good enough" means.

### The Three Types of Evals

**a. Unit Evals — Does this specific input produce the right output?**

One input, one expected output or rubric, graded in isolation. These are the easiest to build
and the fastest to run.

Example: Given "Summarize this 500-word support ticket in 2 sentences," does the AI produce
a 2-sentence summary that covers the core issue?

**b. Integration Evals — Does the full workflow work end-to-end?**

Multiple steps in sequence, tested together. These catch failures that only appear when
components interact.

Example: Given a customer complaint → the AI triages it → drafts a response → suggests a
knowledge base article → does the full chain reach a correct, non-hallucinated result?

**c. Human Evals — Does a human judge this as good?**

A person reviews the output and scores it. These are the gold standard but do not scale.
Use them for calibration (Section 7) and for high-stakes cases.

### What an Eval Set Is

An eval set is a collection of records, each containing:

1. **Input** — what the user (or system) sends to the AI
2. **Expected output or criteria** — what a correct response looks like
3. **Grade** — pass/fail or a score after the AI responds

Think of it as a spreadsheet where each row is one test case.

### How Grading Works

| Grading Method | When to Use | Accuracy |
|---------------|------------|---------|
| **Deterministic** — exact match, keyword check, regex | Factual outputs, structured data, code | Highest — binary, fast |
| **LLM-as-judge** — Claude grades Claude | Prose, tone, relevance, helpfulness | Good when calibrated (see Section 7) |
| **Human review** — a person scores the output | Edge cases, high-stakes, calibration | Highest — but slow and costly |

You do not need to write code to create or run evals. You need a spreadsheet, a grading
rubric, and access to the AI you are evaluating.

---

## Section 3: Building Your First Eval Set (No Code Required)

### Step-by-Step Process

**Step 1: Gather 20 real user interactions**

Your best sources:
- Support tickets your AI handled (or should have handled)
- Session replays showing where users got stuck
- User interview transcripts with representative tasks
- Slack or email threads where users described what they needed
- Any existing logs from your AI feature

You want *real* inputs, not made-up ones. Real inputs contain the messiness that breaks AI —
typos, ambiguity, implicit context. Synthetic inputs are too clean.

**Step 2: For each interaction, write the input, ideal output, and grading criteria**

Do not write what the AI actually said. Write what it *should* have said. If you cannot write
the ideal output, you do not yet have a clear enough definition of "good" — and that itself is
a discovery.

**Step 3: Structure them in a spreadsheet or markdown file**

Use the template below. You do not need a database or a special tool for your first 20.

**Step 4: Run them through the AI and grade the outputs**

Paste each input into the AI. Compare the actual output to your criteria. Mark pass or fail.
If you are using a grading rubric with multiple dimensions, score each dimension.

**Step 5: Calculate a pass rate**

Pass rate = (number of evals that passed) / (total evals) × 100

A pass rate tells you where you are starting from. Every future improvement is measured against
this baseline.

### The Eval Set Template

Copy this structure into a spreadsheet or Markdown file:

```
| ID | Input | Expected Output | Grading Criteria | Actual Output | Pass/Fail | Notes |
|----|-------|----------------|-----------------|---------------|-----------|-------|
| E01 | ... | ... | ... | (fill after running) | | |
```

**Column definitions:**
- **ID** — unique identifier (E01, E02, etc.)
- **Input** — exact text sent to the AI, including any system context
- **Expected Output** — what a correct response looks like (can be a description, not a verbatim answer)
- **Grading Criteria** — the specific rules for pass/fail (see Section 6 for templates)
- **Actual Output** — what the AI actually produced (fill this in after running)
- **Pass/Fail** — your grade
- **Notes** — what was interesting, ambiguous, or worth following up

### Complete Worked Example: 5 Evals for "Dalgo AI Data Assistant"

Imagine Dalgo adds an AI assistant that helps NGO program managers understand their data.
Here are 5 eval cases:

---

**E01 — Simple factual query**

| Field | Content |
|-------|---------|
| Input | "How many beneficiaries did we reach in Maharashtra last month?" |
| Expected Output | A number pulled from the connected warehouse, with the source table named |
| Grading Criteria | PASS if: (1) returns a specific number, (2) names the source, (3) does not hallucinate data not in warehouse. FAIL if: returns a range without explanation, makes up data, or refuses without explanation. |
| Actual Output | *(run and fill)* |

---

**E02 — Trend interpretation**

| Field | Content |
|-------|---------|
| Input | "Our school enrollment numbers dropped 18% in November. What could explain this?" |
| Expected Output | 2-3 plausible explanations relevant to NGO context (school holidays, data lag, seasonal migration) |
| Grading Criteria | PASS if: mentions at least 2 plausible causes, does not present speculation as fact, does not require follow-up to get a useful answer. FAIL if: gives only one explanation, states causes as certain without evidence, or asks for more data before giving any response. |
| Actual Output | *(run and fill)* |

---

**E03 — Action recommendation**

| Field | Content |
|-------|---------|
| Input | "Which of our 12 districts should I prioritize visiting this quarter based on the data?" |
| Expected Output | A prioritized list with reasoning tied to actual metrics visible in the data |
| Grading Criteria | PASS if: recommends specific districts (not "it depends"), ties recommendation to at least one metric, acknowledges what data is and isn't available. FAIL if: refuses to prioritize, recommends all districts, or makes recommendations without citing data. |
| Actual Output | *(run and fill)* |

---

**E04 — Error handling**

| Field | Content |
|-------|---------|
| Input | "Show me the data for District 14" (District 14 does not exist in the database) |
| Expected Output | Acknowledges that District 14 is not found, offers to show available districts or asks for clarification |
| Grading Criteria | PASS if: does not hallucinate data for a non-existent district, clearly states the district was not found, offers a next step. FAIL if: returns fabricated data, silently returns empty results with no explanation, or errors out without a useful message. |
| Actual Output | *(run and fill)* |

---

**E05 — Multi-step reasoning**

| Field | Content |
|-------|---------|
| Input | "We're deciding whether to expand our health program to Pune or Nagpur. What does our data say?" |
| Expected Output | Comparative analysis of both cities on relevant health metrics, with a clear framing of what the data does and does not show |
| Grading Criteria | PASS if: compares both cities, uses at least 2 metrics, explicitly states limitations of the data for this decision. FAIL if: recommends one city without comparison, does not surface data limitations, or produces analysis that only covers one city. |
| Actual Output | *(run and fill)* |

---

After running these 5 evals, calculate your pass rate. If 3 pass and 2 fail, you are at 60%.
Your two failures are your first product insights.

---

## Section 4: Using Evals for Discovery

This is the core insight of this guide. Evals are not just a testing tool — they are a
discovery method. Here is how to use each eval technique to learn something you did not know.

### a. Baseline Eval — What Can the AI Already Do?

Before you build anything, run 20 evals on the raw model with no customization.

This tells you:
- What the AI handles well out of the box (do not build a workaround for something that already works)
- Where the AI consistently fails (these gaps are where product investment pays off)
- What "good" looks like for your domain (writing evals forces you to define this)

A PM who has run a baseline eval can say: "Claude handles factual queries well (85% pass) but
struggles with multi-district comparisons (40% pass). The investment case is in comparison logic,
not basic retrieval."

That is a product decision based on data, not a guess.

### b. Failure Analysis — Failure Clusters = User Needs

When evals fail, do not just count the failures. *Read* them.

Cluster the failures into themes:
1. Print or copy all failed eval cases
2. Group them by what went wrong (e.g., "hallucinated data," "refused to answer," "wrong tone for NGO context")
3. Name each cluster
4. Count how many failures fall into each cluster

Each cluster is a user need you did not know about.

**Example from practice** (paraphrased from the NurtureBoss case described by The Pragmatic
Engineer):

A team reviewing 100+ conversation traces found their AI failures clustered into:
- Cluster A: Wrong date extraction (37% of failures)
- Cluster B: Premature handoff to human (28% of failures)
- Cluster C: Irrelevant knowledge base articles cited (21% of failures)
- Cluster D: Other (14%)

Without evals, they would have built based on intuition. With failure clustering, they built in
priority order: fix date extraction first (biggest impact), then handoff logic, then retrieval.

The eval data *was* the product roadmap.

### c. Edge Case Mining — What Will Your Users Hit?

The hardest evals reveal the edge cases your users will encounter in real use.

When you build your eval set, deliberately include:
- Inputs with typos and non-standard phrasing
- Requests for data you do not have
- Ambiguous questions that could mean multiple things
- Multi-part questions that require sequential reasoning
- Questions from users with less context than you assume

If an eval is hard to grade — if two people on your team disagree whether the output passes —
that is a signal. The ambiguity in grading reflects a real product gap: you have not yet
defined what "good" means for that case. Resolve the grading disagreement, and you have
resolved a product ambiguity.

### d. Regression Detection — Protect Against Unintended Changes

Every time you change your AI product — new prompt, new model, new retrieval logic — run your
eval set before and after.

Compare:
- Overall pass rate (did it go up or down?)
- Pass rates by category (did one area improve while another got worse?)
- Individual cases that flipped from pass to fail (these are regressions)

A regression is a product risk you caught before your users did. Without evals, regressions
surface as support tickets, user complaints, or churn — weeks after you shipped the change.

### e. Competitive Benchmarking — Objective Competitive Data

Run the same eval set on your competitors' AI products.

This gives you something rare in product management: objective data about where you are better
and where you are behind. When you tell your team "our eval pass rate on multi-step reasoning
is 72%, competitor X is 58%, but competitor Y is 81%," that is a competitive positioning
statement grounded in evidence.

Note the caveats: you can only test competitor *public* interfaces, you do not know their
system prompts, and their products may be optimized for different user types. State these
caveats when sharing results. But even with caveats, comparative evals are more rigorous than
"I tested it and it seemed good."

---

## Section 5: The PM Eval Workflow

### Week-by-Week Cadence

**Week 1: Build Your Baseline Eval Set**

Goal: 20 eval cases grounded in real user interactions.

Tasks:
- Pull 20 real user inputs (support tickets, session replays, interviews)
- Write the expected output and grading criteria for each
- Enter them into your eval spreadsheet
- Run them through the AI (paste each input manually if needed)
- Grade each output and calculate your baseline pass rate

Deliverable: A spreadsheet with 20 rows and a pass rate number.

**Week 2: Run Failure Analysis**

Goal: Identify the top 3 product gaps from your failing evals.

Tasks:
- Review every failed eval case in detail
- Group failures into clusters (aim for 3-5 clusters)
- Count failures per cluster
- Write a one-paragraph product insight for each cluster: what users need that the AI is not providing
- Bring the failure clusters to your next team sync

Deliverable: A failure cluster analysis with 3 product gaps ranked by frequency.

**Week 3: Build or Adjust, Then Re-Evaluate**

Goal: Move the pass rate on your top cluster.

Tasks:
- Work with engineering to address the top failure cluster (prompt change, new data source, added guard)
- Re-run your eval set on the updated version
- Calculate the new pass rate
- Check for regressions (evals that were passing and now fail)

Deliverable: Before/after pass rate comparison. Report: "We moved from 60% to 74% on [cluster]. One regression found in [category], addressed."

**Ongoing: Add New Evals as You Discover New Needs**

Every time you encounter a new user scenario — from a support ticket, interview, or session
replay — add it to your eval set. Keep the set fresh and representative of what users
actually do, not just what you imagined they would do when you first built the feature.

### Using Eval Results in PM Artifacts

**In planning meetings:**
Lead with the pass rate. "We're at 68% on our customer support eval set. The two biggest
failure clusters are [A] and [B]. If we fix A, we estimate we get to 80%."

**In PRDs:**
Add an "Evaluation Criteria" section alongside acceptance criteria:

> Acceptance criteria: Feature is live and accessible to all users.
> Evaluation criteria: Pass rate on eval set E (20 cases) reaches 80% before ship. No
> regression on eval sets A, B, C from previous features.

**In stakeholder updates:**
Use the eval dashboard format (Section 8). Keep it to: current pass rate, trend since last
period, top failure cluster, and what is being done about it.

---

## Section 6: Eval Templates for Common PM Scenarios

### a. Customer Support AI

**Input format:**
```
System: You are a support agent for [product]. Answer the user's question accurately
and empathetically.
User: [customer message]
Context: [relevant knowledge base articles, if any]
```

**Output format:** A response the AI would send to the customer.

**Grading rubric:**

| Dimension | PASS | FAIL |
|-----------|------|------|
| Accuracy | Every factual claim is correct and supported by the knowledge base | Any factual claim is wrong or not supported |
| Completeness | Addresses the user's core question | Ignores the main question or only addresses part of it |
| Tone | Empathetic, professional, not condescending | Dismissive, overly formal, or patronizing |
| Action clarity | Gives the user a clear next step | Leaves the user uncertain what to do |
| No hallucination | Does not reference features, policies, or steps that do not exist | References something that does not exist |

Overall: PASS requires passing all 5 dimensions. FAIL on any dimension = FAIL overall.

---

### b. Document Summarization

**Input format:**
```
Summarize the following document in [N] sentences / bullet points.
Focus on: [key points the PM cares about]
Document: [full text]
```

**Output format:** A summary of the specified length.

**Grading rubric:**

| Dimension | PASS | FAIL |
|-----------|------|------|
| Key point coverage | Covers at least [N] of the [M] key points defined by the PM | Misses more than [M-N] key points |
| Length compliance | Meets the specified length constraint (±10%) | Significantly over or under the target length |
| No added content | Does not introduce claims not in the source document | Introduces information not present in the source |
| Readability | Can be understood by a non-expert reader in the target audience | Requires expertise to parse or contains jargon not in the original |

---

### c. Recommendation System

**Input format:**
```
User profile: [relevant attributes — role, history, stated preferences]
Context: [current situation or goal]
Task: Recommend [N items / actions] for this user.
Available options: [list of options the AI can draw from]
```

**Output format:** A ranked list of recommendations with reasoning.

**Grading rubric:**

| Dimension | PASS | FAIL |
|-----------|------|------|
| Relevance | Top recommendation is appropriate for this user's profile and context | Top recommendation is generic or inappropriate for the user's context |
| Reasoning | States why each recommendation was made | Makes recommendations without explanation |
| Scope | Recommends only from the available options | Recommends options not in the available set |
| Rank validity | Higher-ranked items are more appropriate than lower-ranked items | Ranking is random or inversely appropriate |

---

### d. Code Assistant

**Input format:**
```
Language: [Python / JS / SQL / etc.]
Task: [description of what the code should do]
Constraints: [any requirements — must use X library, must handle Y edge case, etc.]
```

**Output format:** Working code with brief explanation.

**Grading rubric:**

| Dimension | PASS | FAIL |
|-----------|------|------|
| Correctness | Code runs without errors and produces the correct output for the given task | Code has syntax errors, runtime errors, or produces wrong output |
| Constraint compliance | All stated constraints are met | Any stated constraint is violated |
| Edge case handling | Handles at least the edge cases mentioned in the input | Crashes or gives wrong result on stated edge cases |
| Explanation clarity | Explanation is correct and a developer can follow it | Explanation contradicts the code or is misleading |

---

### e. Search / Retrieval

**Input format:**
```
Query: [user's search query]
Retrieved results: [top N results returned by the system]
```

**Output format:** A relevance judgment — does this result answer the query?

**Grading rubric:**

| Dimension | PASS | FAIL |
|-----------|------|------|
| Top-1 relevance | The first result directly answers or strongly relates to the query | The first result is off-topic or only tangentially related |
| Top-3 coverage | At least 2 of the top 3 results are relevant | Fewer than 2 of the top 3 results are relevant |
| No topic drift | Results are from the expected domain | Results are from an unrelated domain |
| Completeness | For known-item queries, the expected item appears in top 3 | For known-item queries, the expected item is not in top 3 |

---

## Section 7: LLM-as-Judge — Using Claude to Grade Claude

### Why This Works

LLM-as-judge is grading AI outputs with another AI. It sounds circular, but it works for
a specific reason: evaluation is easier than generation. Judging whether a summary captures
the key points is a simpler task than generating the summary. A model that would produce a
mediocre summary can often reliably identify whether a given summary is good or bad.

This asymmetry — evaluation being easier than generation — is why LLM-as-judge is reliable
enough for most PM use cases, provided it is properly calibrated.

### When It Does Not Work

LLM-as-judge breaks down in four situations:
1. **Same-family bias** — using GPT-4 to judge GPT-4 outputs. The judge over-rates outputs
   from its own model family. Always use a different model family for judging.
2. **Length-confidence bias** — LLM judges tend to rate longer, more verbose answers higher,
   even when a shorter answer is more correct. Include "penalize unnecessary length" in your rubric.
3. **Uncalibrated judges** — a judge that has never been compared to human grades is a
   judge you cannot trust. Calibrate before you deploy (see below).
4. **Novel domains** — if your product domain is highly specialized (medical diagnostics,
   legal compliance, rare NGO contexts), the judge may lack the domain knowledge to grade reliably.
   In these cases, add a human review layer.

### The Grading Prompt Template

Copy and adapt this for your use case:

```
You are a strict quality evaluator for an AI assistant.

Grade the following AI response against the criteria below.

--- CRITERIA ---
[Paste your grading rubric here — each dimension as a bullet point]

--- INPUT ---
[The input that was sent to the AI]

--- AI RESPONSE ---
[The AI's response being evaluated]

--- GRADING INSTRUCTIONS ---
For each criterion, state PASS or FAIL, then give a one-sentence reason.
Then give an OVERALL verdict: PASS (all criteria pass) or FAIL (any criterion fails).
Return your response in this exact format:

Criterion 1 (name): PASS/FAIL — [reason]
Criterion 2 (name): PASS/FAIL — [reason]
...
OVERALL: PASS/FAIL
Confidence: HIGH / MEDIUM / LOW
Note any ambiguity or edge cases below:
```

### How to Calibrate: Comparing LLM Grades to Human Grades

1. Take 50 eval cases from your set (more is better; 200 is ideal for production)
2. Grade them yourself (or with a domain expert)
3. Run the same cases through your LLM judge
4. Calculate Cohen's kappa between your grades and the judge's grades

**Interpreting kappa:**

| Kappa | Interpretation | Action |
|-------|---------------|--------|
| < 0.4 | Poor agreement — judge is unreliable | Revise your rubric; add examples to the judge prompt; consider a different model |
| 0.4 – 0.6 | Moderate agreement — use with caution | Acceptable for low-stakes monitoring; add human spot checks |
| 0.6 – 0.8 | Good agreement — suitable for most PM use cases | Trust the judge for routine evals; keep calibrating monthly |
| > 0.8 | Strong agreement | High confidence; reduce human review frequency |

Target: kappa ≥ 0.6 before trusting your judge for product decisions.

### The Production Calibration Mistake

This is a real failure mode documented in production systems:

A team deployed a GPT-4 judge to score GPT-4 outputs for a groundedness metric. For three
months, every dashboard showed green — high pass rates, no alerts. The team felt confident
about their AI quality.

Then a new domain expert joined and hand-labeled 50 production outputs. When they compared
their grades to the judge's grades, the Cohen's kappa was **0.31** — poor agreement.

The judge had been systematically over-rewarding GPT-4 outputs (same-family bias) and
under-penalizing fluent hallucinations (it scored verbose, confident-sounding answers highly
even when they contained fabricated claims).

The dashboards had been lying for three months. The team had shipped multiple model updates
during that window with no idea of the real quality trajectory.

**The lesson:** Green dashboards without calibration are not a product signal — they are a
false sense of security. Calibrate your judge before you trust it, and recalibrate monthly.

### Calibration Cadence

- **Before launch:** Verify kappa ≥ 0.6 on at least 50 labeled examples
- **Monthly:** Re-run calibration on 50 new examples from recent production traffic
- **Alert condition:** If kappa drops below 0.6, pause automated reporting until you diagnose the cause

---

## Section 8: Communicating Eval Results to Stakeholders

### For Engineers

Engineers want specificity. Give them:
- The exact eval cases that failed (with inputs, expected outputs, and actual outputs)
- The failure cluster and how many cases are in it
- A proposed acceptance condition: "this cluster is resolved when pass rate on these 7 cases exceeds 90%"

Avoid: vague requests like "make it more accurate." Replace with: "these 7 cases all fail on
the accuracy criterion. Here they are."

### For Executives

Executives want trend and risk. Give them:
- Pass rate trend (this week vs. last week vs. launch)
- Top risk: the failure cluster that most impacts user outcomes
- What is being done and by when

One-sentence format: "Our AI support agent is at 74% pass rate on our 20-case eval set, up
from 62% last month. The top failure cluster is multi-product queries (6 cases failing); we
are targeting 85% by Q3 with a retrieval improvement shipping in two weeks."

### For Designers

Designers want to understand the failure modes that are interaction design problems, not just
AI problems. When you share failure clusters, flag the ones where:
- The user's input was ambiguous because the UI did not set expectations correctly
- The user asked for something the AI cannot do — and the UI did not communicate this limitation
- The failure could be prevented with better progressive disclosure or clearer affordances

### The Eval Dashboard

Track these four metrics in your team's weekly sync:

| Metric | Description | Target |
|--------|-------------|--------|
| Overall pass rate | % of eval cases passing | ↑ toward your defined target |
| Pass rate by cluster | Pass rate broken down by failure category | Each cluster trending up |
| Regression count | # of cases that newly failed this week | 0 (or investigated) |
| Eval coverage | # of unique scenarios covered by your eval set | Growing over time |

### Language That Works in Product Conversations

Instead of: "The AI seems to be getting better."
Say: "Our eval pass rate moved from 62% to 74% over the last 4 weeks."

Instead of: "We think the prompt change helped."
Say: "After the prompt change, the multi-district comparison cluster moved from 40% pass
to 71% pass. No regressions."

Instead of: "Users are running into edge cases."
Say: "We have identified 3 failure clusters. The largest — error handling on missing data —
accounts for 8 of our 12 failing eval cases. It is the top engineering priority for this sprint."

---

## Section 9: Tools for PMs (No-Code Options)

### Overview

| Tool | Type | PM-Friendliness | Best For |
|------|------|----------------|---------|
| Claude Projects | Built-in AI tool | Very easy | Getting started; manual evals |
| Spreadsheet | Any spreadsheet tool | Easy | Small eval sets (<100 cases) |
| Langfuse | Open source platform | Moderate | Teams who want observability + evals |
| Braintrust | Commercial platform | Easy–Moderate | PMs who want a dedicated eval UI |
| Claude Code scripts | One-time code gen | Easy (with help) | Automating repeatable eval runs |

---

### Claude Projects as an Eval Runner

**What it is:** Claude's built-in project workspace where you can store a system prompt and
run a consistent AI persona across multiple conversations.

**How to use it for evals:**
1. Create a new Project called "Eval Runner — [Feature Name]"
2. In the project instructions, paste your grading prompt template (Section 7)
3. In each conversation, paste: the eval input + the AI's response
4. Claude will grade it against your rubric
5. Copy the grade into your spreadsheet

**When to use it:** When you are starting out and have fewer than 30 eval cases. Zero setup,
works today.

**Difficulty:** Very easy. No code, no signup beyond Claude.

---

### Spreadsheet-Based Eval Tracking

**What it is:** A Google Sheet or Airtable with your eval set structure.

**Setup:**
- Columns: ID, Input, Expected Output, Criteria, Actual Output, Pass/Fail, Notes, Date Run, Version
- One row per eval case
- One sheet per eval set; a summary sheet with pass rate formula

**When to use it:** Always. Even if you adopt a dedicated tool, keep your eval set definition in
a spreadsheet. It is your source of truth and is shareable with non-technical stakeholders.

**Difficulty:** Easy. The best starting point.

---

### Langfuse

**What it is:** Open-source AI engineering platform covering tracing, evals, prompt management,
and datasets. Free to self-host; hosted version available.

**PM-relevant features:**
- Annotation queues: create a queue of AI outputs for a human reviewer (you) to score in the UI
- Dataset management: build and manage eval sets through the UI without code
- Experiment UI: compare two versions of your AI side-by-side on the same eval set
- Score analytics: dashboard showing pass rates, trends, and breakdowns over time

**When to use it:** When you want to move beyond a spreadsheet and your engineering team is
already using it (or open to adopting it). Best when engineering and PM share the same tool.

**Difficulty:** Moderate. The UI is accessible to non-engineers, but initial setup requires
engineering support.

---

### Braintrust

**What it is:** Commercial AI observability and eval platform designed for teams shipping AI
products. Emphasis on PM and QA accessibility alongside engineering workflows.

**PM-relevant features:**
- Playground: configure an eval without writing code — define test cases, connect your AI
  endpoint, run evals in the UI
- No-code scorers: describe your scoring criterion in plain English; the platform generates
  a scorer automatically
- Annotation interface: custom review UIs for non-engineers to label outputs
- 25+ built-in scorers for accuracy, relevance, safety — use these as starting points

**When to use it:** When your team wants a dedicated eval product and is willing to pay for it.
Good for teams where the PM owns the eval process and engineers own the infrastructure.

**Difficulty:** Easy to moderate. The playground is genuinely no-code. Advanced experiments
require SDK integration.

---

### Simple Python Scripts via Claude Code

**What it is:** A short Python script that reads your eval set spreadsheet, sends each input
to the AI API, and records the outputs — automating the "paste and run" process.

**How to get it without writing code:**
1. Export your eval spreadsheet as CSV
2. Tell Claude Code: "Write me a Python script that reads evals.csv, sends the Input column
   to Claude claude-sonnet-4-5 with this system prompt, saves the outputs to a new column,
   and calculates the pass rate using these grading criteria."
3. Run it once to verify; re-run it whenever you update your product

**When to use it:** When you have more than 30 eval cases and manual running becomes slow.
This step-change in automation is worth doing early.

**Difficulty:** Easy (with Claude Code). You do not write the code; you describe what you need.

---

## Section 10: Worked Example — End-to-End Eval-Driven Discovery

### Scenario

Dalgo ships an AI feature: **"Ask your data"** — a natural language interface for NGO program
managers to query their warehouse without writing SQL.

Three weeks after launch, the team gets user complaints:

> "It keeps saying 'I don't have enough data to answer that' even when the data is clearly there."
> "It gave me a number that doesn't match what I see in the dashboard."
> "Half the time it just doesn't understand what I'm asking."

A traditional PM response: schedule user interviews, write a "fix the AI" ticket, wait.

An eval-driven PM response:

---

**Step 1: Build the Eval Set from the Complaints**

The three complaint themes map directly to eval categories:
- "I don't have enough data" → over-refusal evals (inputs where data exists, AI should answer)
- "Number doesn't match dashboard" → accuracy evals (inputs with verifiable expected outputs)
- "Doesn't understand what I'm asking" → query comprehension evals (ambiguous or informal queries)

Build 7 evals per category = 21 total. Use real queries from support tickets as inputs.

**Step 2: Run the Baseline**

Run all 21 evals on the current version. Results:

| Category | Pass | Fail | Pass Rate |
|----------|------|------|-----------|
| Over-refusal | 2 | 5 | 29% |
| Accuracy | 5 | 2 | 71% |
| Query comprehension | 3 | 4 | 43% |
| **Overall** | **10** | **11** | **48%** |

The baseline is 48%. This is a concrete number, not a vibe.

**Step 3: Failure Analysis**

Read the 11 failing evals. Findings:

- Over-refusal cluster: 4 of 5 failures involve queries that reference two different data
  sources (e.g., "Compare our field survey data to our MIS data"). The AI does not know it
  can query both simultaneously.
- Accuracy cluster: Both failures involve date arithmetic (e.g., "last quarter" resolved
  to the wrong quarter).
- Query comprehension cluster: 3 of 4 failures use NGO-specific jargon ("beneficiaries,"
  "ASHA workers," "anganwadi") that the AI treats as generic nouns rather than recognizing
  as domain terms.

**Step 4: Product Decisions from Failure Clusters**

| Failure Cluster | Root Cause | Product Decision |
|----------------|-----------|-----------------|
| Over-refusal on cross-source queries | AI does not know its data model | Add data schema to system prompt |
| Date arithmetic errors | Prompt does not specify timezone / fiscal year convention | Add date context (current date, fiscal year start) to system prompt |
| NGO jargon not recognized | No domain glossary | Add NGO glossary as retrieval context |

These are three specific, cheap prompt-level fixes. No new features needed.

**Step 5: Implement and Re-Evaluate**

Engineering makes the three prompt changes in one sprint. Re-run the same 21 evals:

| Category | Before | After | Change |
|----------|--------|-------|--------|
| Over-refusal | 29% | 71% | +42pp |
| Accuracy | 71% | 86% | +15pp |
| Query comprehension | 43% | 71% | +28pp |
| **Overall** | **48%** | **76%** | **+28pp** |

One sprint. Eval-driven. From 48% to 76% pass rate.

**Step 6: Communicate to Stakeholders**

> "Three weeks after launch, our 'Ask your data' feature was at 48% pass rate on our 21-case
> eval set. Failure analysis identified three clusters: cross-source queries (over-refusal),
> date arithmetic, and NGO jargon. We made targeted prompt changes. Current pass rate: 76%.
> Target: 85% by end of quarter. Remaining gap is in query comprehension; we're adding 10 new
> evals this week from recent support tickets."

That is a product update. Not "we fixed some AI stuff." A product update.

---

## Quick Reference

### The Eval Flywheel

```
Real user inputs
      ↓
Build eval set (20 cases)
      ↓
Run baseline → pass rate
      ↓
Cluster failures → product gaps
      ↓
Build / adjust feature
      ↓
Re-run evals → measure improvement
      ↓
Add new evals from new user needs
      ↓
(repeat)
```

### The 5 Eval Questions Every PM Should Ask

1. What is our current pass rate on our eval set?
2. What is the top failure cluster this week?
3. Did we regress anywhere after our last ship?
4. Are our new evals representative of what users are actually doing?
5. When did we last calibrate our LLM judge against human grades?

### The 3 Things That Kill Eval Programs

1. **Synthetic evals** — evals built from imagined use cases rather than real user interactions.
   They pass easily and miss the real problems.
2. **Uncalibrated judges** — trusting green dashboards from an LLM judge that has never been
   compared to human grades. (The 0.31 kappa case.)
3. **Eval sets that go stale** — building 20 evals at launch and never adding more. Your users'
   behavior evolves; your eval set must too.

---

## Sources

- [Product management on the AI exponential — Claude / Anthropic](https://claude.com/blog/product-management-on-the-ai-exponential)
- [Kai Xin Tai — Datadog Summit Chicago speaker profile](https://events.datadoghq.com/summits/datadog-summit-chicago/speakers/kai-xin-tai/)
- [Improve AI agent quality with Bits Evals — Datadog Engineering Blog](https://www.datadoghq.com/blog/bits-evals/)
- [Evals for PMs: A practical guide to AI product quality — Braintrust](https://www.braintrust.dev/blog/evals-for-pms)
- [A pragmatic guide to LLM evals for devs — The Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/evals)
- [LLM-as-Judge Best Practices in 2026: Calibration, Bias, and Cost — FutureAGI](https://futureagi.com/blog/llm-as-judge-best-practices-2026)
- [Evaluation of LLM Applications — Langfuse Documentation](https://langfuse.com/docs/evaluation/overview)
- [Building Effective AI Agents — Anthropic](https://www.anthropic.com/research/building-effective-agents)
- [Anthropic's Guide to AI Agent Evals: What Support Teams Need to Know — Inkeep](https://inkeep.com/blog/anthropic-s-guide-to-ai-agent-evals-what-support-teams-need)
- [Top 5 platforms for agent evals in 2025 — Braintrust](https://www.braintrust.dev/articles/top-5-platforms-agent-evals-2025)
