# AI-Native Product Strategy: A PM's Workshop for Deep Strategic Thinking with Claude

**Who this is for:** Senior PMs and aspiring senior PMs who want Claude as a genuine strategic thought partner — not a content generator, but a tool that sharpens your thinking, surfaces your blind spots, and helps you make better decisions faster.

**What this is not:** A guide to writing PRDs with AI. This is about the harder work — positioning, prioritization, competitive strategy, roadmap sequencing, and the multi-turn conversations that get you to a decision.

---

## 1. Claude as a Strategic Thought Partner (Not a Writing Tool)

### The wrong use

Most PMs discover Claude by asking it to write things: strategy docs, PRD sections, OKRs. This works. It saves time. It is also the least valuable thing Claude can do for you.

The output of a strategy doc is not strategy. The thinking that happens *before* the doc is strategy. If you skip the thinking and go straight to writing, you get a document that looks strategic but has not actually pressure-tested anything.

**Wrong prompt:**
> "Write me a product strategy for our project management tool targeting mid-market companies."

You will get a plausible-sounding document full of conventional wisdom that does not reflect your actual situation, your actual constraints, or the specific bets you need to make.

### The right use

The right use is to load Claude with your actual context — your metrics, your competitors, your customers' words, your team's constraints — and then use it to do the thinking work that is hard to do alone:

- "Pressure-test this positioning. Where is it weakest?"
- "I'm planning to do X. Steelman the case for doing Y instead."
- "What am I not seeing? What's the strongest argument against my current plan?"
- "Play the role of my most skeptical board member. What questions would you ask?"

### Why Claude is unusually good at strategy work

Three properties make Claude genuinely useful here, not just fast:

**Broad knowledge, no political agenda.** Claude has read the strategy literature, the competitive landscapes, the case studies. It has no career stake in your decision. It will not soften a criticism because it reports to you or because it is afraid of conflict. A good outside advisor costs $500/hour. Claude charges per token.

**Infinite patience for working through options.** Good strategy requires exploring paths you end up rejecting. With a human collaborator, this feels wasteful. With Claude, you can run five different framings of the same problem until you find the one that clarifies the real tradeoff.

**Memory within a conversation.** A single strategy session can hold your entire product context, customer interview excerpts, competitive intelligence, and historical decisions. Claude synthesizes across all of it simultaneously — something a human advisor typically cannot do without days of onboarding.

### The mental model

Think of Claude as a brilliant colleague who has read everything and has no horse in the race. They are not smarter than you about your product — you have context they will never have. But they are faster at pattern-matching across industries, more willing to play devil's advocate, and have no political reasons to agree with you.

Your job is to give them your context. Their job is to help you think more clearly.

### What this guide will help you do

- Run structured strategy frameworks (JTBD, RICE+, pre-mortem, steelman) with complete prompts you can copy and paste
- Run a full multi-turn strategy conversation from opening question to decision memo
- Do roadmap strategy: sequencing, dependency mapping, scenario planning, stakeholder simulation
- Do deep prioritization that goes beyond RICE scores
- Run competitive strategy analysis
- Prepare for executive communication
- Build institutional strategic memory

---

## 2. Strategic Frameworks Claude Excels At

### 2a. Jobs-to-Be-Done Analysis

**What it is:** JTBD is a framework for understanding why customers hire a product — the functional, emotional, and social outcome they are trying to achieve. The insight is that customers do not buy features; they hire solutions for a job they need done.

**The prompt:**

```
I am going to share [N] customer interview quotes with you. Your job is to extract the underlying Jobs-to-Be-Done — the functional jobs (what task do they need done), the emotional jobs (how do they want to feel), and the social jobs (how do they want to be perceived by others).

For each job you identify:
- Write the job statement in the standard JTBD format: "When [situation], I want to [motivation/job], so I can [expected outcome]."
- Rate its frequency (how often this job arises) as High / Medium / Low
- Rate its underserved status (how poorly current solutions address it) as High / Medium / Low
- Note which quotes evidence this job

Here are the quotes:

[PASTE QUOTES]

After listing the jobs, synthesize: which 2-3 jobs represent the highest-leverage opportunities (high frequency + high underserved rating)? What would a product built specifically for those jobs look like, and how does it differ from what we are currently building?
```

**Example output (abbreviated):**

*Job 1: When I need to update stakeholders on project status, I want to send a single source-of-truth update, so I can stop fielding individual Slack messages asking "where are we on this?"*
- Frequency: High | Underserved: High
- Evidence: Quotes 3, 7, 12

*Job 2: When leadership asks about project health in a meeting, I want to pull up a real-time view instantly, so I can appear competent and prepared rather than scrambling for data.*
- Frequency: Medium | Underserved: High
- Evidence: Quotes 2, 9

**When to use it:** After user interviews, when deciding what to build next, or when a feature is underperforming and you cannot figure out why.

**Modify the prompt by:** Asking Claude to compare the jobs across different customer segments, or asking it to map your current feature set against each job to find coverage gaps.

---

### 2b. Opportunity Scoring (RICE+ with Qualitative Modifiers)

**What it is:** RICE scoring (Reach, Impact, Confidence, Effort) creates a numerical priority ranking. The problem is that raw RICE ignores strategic context: a 9.0 score for a feature that strengthens a moat is different from a 9.0 score for a feature that just adds table stakes.

**The prompt:**

```
I am going to give you a list of potential features or initiatives. For each one, I want you to produce an enhanced RICE score that incorporates qualitative strategic modifiers.

Scoring instructions:
- Reach (1-10): how many users/customers does this affect per quarter?
- Impact (1-10): how much does it move our North Star metric per user?
- Confidence (0-100%): how confident are we in the above estimates?
- Effort (person-weeks): raw build cost
- Base RICE score = (Reach × Impact × Confidence) / Effort

Then apply these qualitative modifiers (each can add or subtract 20% from the base score):
- Strategic fit: does this strengthen a core differentiator (+) or add table stakes (-)?
- Moat contribution: does this build a defensible advantage (+) or is it easily copyable (-)?
- Dependency enabler: does this unlock other high-value work (+) or stand alone (-)?
- Reversibility: is this easy to undo if wrong (+) or a long-term commitment (-)?

Product context: [describe your product, market, stage, and primary metric]

Initiatives to score:
1. [Initiative 1]
2. [Initiative 2]
3. [Initiative 3]
...

For each initiative: show the base RICE score, the qualitative modifier applied to each dimension, the final adjusted score, and a 1-sentence rationale. Then rank the initiatives and flag any where the adjusted ranking differs significantly from the base RICE ranking — those are the most strategically interesting disagreements.
```

**When to use it:** Quarterly roadmap planning, when stakeholders are debating priority and the argument is getting political, when you want to make tradeoffs explicit.

---

### 2c. Strategy Memo (Amazon-Style Narrative Planning)

**What it is:** Amazon's working-backwards approach starts from the customer outcome and reasons backwards to required actions. The output is a 6-page narrative memo, not slides. The discipline of writing in prose forces clarity of logic.

**The prompt:**

```
Help me write an Amazon-style strategy narrative for the following initiative. The narrative should follow this structure:

1. The customer and the problem (who exactly, what job are they trying to do, why do current solutions fail)
2. The solution (what we are building and why it addresses the job better than alternatives)
3. The business case (why this matters to our business — growth, retention, moat, margin)
4. The key risks and mitigations (the 3 things most likely to make this fail, and what we will do about each)
5. The definition of success (specific, measurable outcomes at 3 months, 6 months, 12 months)
6. The required decisions (what must be true / decided before we can start)

Initiative context:
- What we are considering building: [describe]
- Target customer: [describe]
- Why now: [market timing, competitive pressure, etc.]
- Key constraints: [headcount, timeline, technical]

Write the memo as a first draft, then flag: where did you have to make assumptions due to gaps in my context? List those assumptions and their risk level (high/medium/low). I will fill in the gaps and we will iterate.
```

**When to use it:** Before committing engineering resources to a major initiative. The process of filling in Claude's gaps is itself the strategy work.

---

### 2d. Pre-Mortem

**What it is:** A pre-mortem imagines that a strategy has already failed and asks: what went wrong? This sidesteps optimism bias and surfaces risks that forward-looking planning misses.

**The prompt:**

```
I am going to describe a product strategy we are planning to execute. Your job is to run a pre-mortem.

Assume it is 18 months from now and this strategy has clearly failed. The product did not achieve its goals. Leadership is disappointed. The team is demoralized.

Diagnose what went wrong. Specifically:

1. List 5-7 plausible failure modes, ordered from most to least likely
2. For each failure mode: describe the early warning signal we would have seen at month 3-6 that we probably ignored
3. Identify the single assumption in our strategy that is most fragile — the one whose failure would cause the most damage
4. Suggest one decision we could make in the next 30 days that would reduce our exposure to the top failure mode

The strategy:
[Describe your strategy here — what you are doing, why, and what success looks like]
```

**When to use it:** Before any major strategy presentation or resource commitment. Run it before the exec review, not after.

---

### 2e. Steel-Manning (The Strongest Case Against Your Direction)

**What it is:** Steel-manning is the opposite of a straw man — you construct the *best possible* version of the opposing argument. It forces you to genuinely reckon with the alternative rather than dismiss it.

**The prompt:**

```
I am going to describe the strategic direction I am currently planning to pursue. Your job is to steel-man the strongest case for the alternative direction.

Do not give me a balanced view. Do not hedge. Argue the opposing case as forcefully as you can — as if you were a brilliant, well-informed PM who genuinely believes the alternative is the right call.

My current direction: [describe what you plan to do]
The alternative I keep dismissing: [describe the alternative]

After making the strongest possible case for the alternative, give me one paragraph of honest synthesis: given both cases, what is the actual crux of the disagreement? What single question, if answered, would resolve which direction is right?
```

**When to use it:** When you have made a strategic choice but still feel nagging doubt. When a strong internal voice is pushing back and you cannot tell if they are right. When you want to arrive at a decision meeting having already processed the opposing view.

---

### 2f. Market Sizing (TAM/SAM/SOM with Exposed Assumptions)

**The prompt:**

```
Help me build a market sizing analysis for [market/segment]. Use a bottom-up approach where possible, but supplement with top-down where needed.

Structure the analysis as:
- TAM (Total Addressable Market): the full universe of potential customers
- SAM (Serviceable Addressable Market): the portion we could realistically serve given our model, geography, and go-to-market
- SOM (Serviceable Obtainable Market): what we could realistically capture in 3 years

For each layer:
- State the key assumption you are making
- Give a low / base / high estimate based on varying that assumption
- Flag which assumptions are most uncertain — these are the ones we need to test before betting the roadmap on this market

Context about our product and go-to-market: [describe]

After the analysis, tell me: what would have to be true about the market for this opportunity to be 10x larger than your base estimate? What would have to be true for it to be 10x smaller?
```

---

### 2g. Positioning Canvas

**The prompt:**

```
I want to run a positioning analysis for our product. Help me build a competitive positioning canvas.

Step 1 — Extract the relevant competitive dimensions: given the following list of competitors and our product, identify the 6-8 most strategically important dimensions that buyers actually use to make decisions (not the dimensions that make us look good — the real ones).

Step 2 — Score each player: rate each competitor and us on each dimension (1-5). Where you lack data, flag it explicitly.

Step 3 — Identify white space: where are there dimension combinations that no current player owns?

Step 4 — Stress test our claimed positioning: for each point of differentiation we claim, ask: (a) is it actually true? (b) is the buyer willing to pay for it? (c) can a competitor copy it in 12 months?

Our product: [describe]
Competitors to include: [list]
Our claimed positioning: [describe what we say makes us different]
```

---

### 2h. Assumption Mapping

**The prompt:**

```
I am going to describe a product initiative. Help me run an assumption mapping exercise.

First, generate the full list of assumptions embedded in this initiative — across four categories:
- Desirability: assumptions about what users want
- Feasibility: assumptions about what we can build
- Viability: assumptions about what makes business sense
- Stakeholder alignment: assumptions about what the organization will support

For each assumption:
- State it explicitly (many assumptions are implicit — surface them)
- Rate importance: if this assumption is wrong, how badly does the initiative fail? (1-5)
- Rate uncertainty: how confident are we that this is actually true? (1-5, where 5 = highly uncertain)
- Priority score = importance × uncertainty

Sort by priority score and identify the top 5 assumptions we should test before committing resources to build.

For each of the top 5: suggest the cheapest, fastest experiment that would tell us whether the assumption holds. Include what "pass" and "fail" look like.

The initiative: [describe]
```

---

### 2i. Working Backwards

**The prompt:**

```
I want to work backwards from an ideal outcome to understand what has to be true and what we need to do.

Ideal outcome: [describe the state of the world 2 years from now if this strategy succeeds — be specific about metrics, customer behavior, and competitive position]

Now reason backwards:
1. What must be true 12 months from now for the 24-month outcome to be achievable?
2. What must be true 6 months from now for the 12-month milestone to be achievable?
3. What must be true 3 months from now for the 6-month milestone to be achievable?
4. What must we decide or start building in the next 30 days?

For each step, flag: what is the single constraint that is most likely to make this path fail? What is the dependency that is hardest to control?

At the end, give me the critical path: the minimum sequence of events that has to happen for this to work.
```

---

### 2j. Adjacent Possible

**The prompt:**

```
We are planning to build [feature/capability]. Once we have built it and it is working well, what adjacent problems does this open up — for us, for our users, and for our competitive position?

Think in three horizons:
- Horizon 1 (direct adjacencies): problems we are now uniquely positioned to solve that we could not solve before
- Horizon 2 (second-order adjacencies): what does solving those problems open up?
- Horizon 3 (strategic platform potential): if we pursued these adjacencies aggressively, what kind of company could we become?

Also tell me: is there a risk that building [feature/capability] creates an adjacent problem that a competitor will solve before we can? If so, which competitor and what would that look like?

Context: [describe your product, customers, and the capability you are planning to build]
```

---

## 3. The Strategy Conversation Workflow

The frameworks above are single prompts. The real power of Claude for strategy is the multi-turn conversation — where each exchange builds on the previous one and you arrive at a decision you could not have reached alone.

Here is the workflow.

### 3a. Context Loading

Before any strategy session, give Claude a context brief. Do this once at the start of the conversation — not piecemeal.

**Context brief template:**

```
Before we start our strategy session, here is the context you need:

Product: [what it does, in one paragraph]
Stage: [seed / series A / growth / public — and key metrics: ARR, users, growth rate]
Customer: [who uses it, what job they hire it for, one representative customer quote]
Competitive landscape: [top 3 competitors and how we differ from each]
Our current strategy: [what we are focused on this year and why]
Key constraints: [headcount, runway, tech debt, market window, anything that limits our options]
Open strategic question we are working through: [the decision or question this session is about]

As you engage with me, push back when my thinking is unclear or when I seem to be avoiding a hard question. Your job is not to validate my thinking — it is to sharpen it.
```

### 3b. The Opening Move

Frame the strategic question specifically. Vague questions get vague responses.

**Weak:** "What should our strategy be?"

**Strong:** "We are facing a specific decision: do we invest the next two quarters in moving upmarket (enterprise features, higher ACV, longer sales cycle) or do we defend our current mid-market position where we are starting to see churn pressure from cheaper, simpler tools? Help me structure the key factors I should weigh."

### 3c. The Pressure Test

Once Claude has responded with analysis, do not accept the first answer. Push it.

**Pressure test prompts:**

```
That is a reasonable analysis. Now tell me where it is wrong. What are you most uncertain about? What would change your conclusion?
```

```
You seem to be favoring option A. Make the strongest possible case for option B. Do not hedge — argue it directly.
```

```
What question am I not asking that I should be asking?
```

```
If you were the skeptic in the room when I present this decision, what would you attack?
```

### 3d. The Synthesis

After 4-6 exchanges, ask Claude to synthesize.

```
Based on everything we have discussed, summarize the core tensions and tradeoffs in this decision. What are the two or three things that are genuinely uncertain and cannot be resolved with analysis alone — they require a judgment call? What is the single most important question I need to answer before I can make this decision with confidence?
```

### 3e. The Decision Memo

Turn the conversation into a document.

```
Based on our conversation, write a 1-page decision memo. Structure it as:

1. The decision (one sentence)
2. Context (2-3 sentences: why this decision now)
3. The options considered (brief bullets)
4. The key factors that drove the decision
5. The decision and rationale (the actual recommendation)
6. The risks and mitigations
7. The success criteria (how we will know if this was the right call, and when)
8. The open questions that remain

Keep it to one page. Write for an executive audience — direct, no throat-clearing.
```

---

### 3f. Complete Example: "Should We Build X or Y?"

**Scenario:** You are a PM at Notion-like tool for small teams. You have two competing initiatives for Q3: (A) build an AI writing assistant to compete with new AI-first entrants, or (B) invest in team permissions and access controls to unblock enterprise deals. You have 4 engineers for 3 months.

---

**PM:** *(context brief already loaded)*

I need help thinking through a resource allocation decision. We have 4 engineers for Q3 and two competing priorities: (A) AI writing assistant — our top retention driver is daily writing, and four AI-first startups launched writing features in the last 6 months, creating direct competitive pressure. (B) Enterprise permissions and access controls — our sales team has 12 deals stuck at security review waiting for this, representing roughly $300K ARR. We can do one well or both badly. Help me think through this.

**Claude:**

This is a classic "defend vs. grow" tension. Let me try to structure it.

The two initiatives are not just feature choices — they represent different theories of your business over the next 18 months.

Option A (AI writing) bets that: (1) the competitive threat from AI-first tools is urgent enough that a 3-month delay meaningfully increases churn risk; (2) your users value AI writing enough to stay for it; and (3) you can ship something differentiated enough to matter, not just table stakes.

Option B (Enterprise permissions) bets that: (1) those 12 deals will actually close once unblocked; (2) the $300K converts at reasonable rates; and (3) enterprise growth will offset any churn from AI competitive pressure.

The crux question is: what is your current churn signal from AI competitors? Is it actual churn, or is it sales team anxiety about future churn?

If users are leaving *now* for AI tools, that changes the calculus significantly. If it is mostly fear of future switching, you may have more runway on the AI front than the sales team is communicating.

What does your retention data actually show?

---

**PM:** Retention is actually stable — churn is flat at 3.5% monthly. But NPS has dropped 8 points over the last two quarters and the top theme in detractor responses is "AI features." The sales team is confident about those 12 deals but they have been saying that for 2 quarters.

**Claude:**

That is a really important distinction. Flat churn with declining NPS is a leading indicator of future churn, not current emergency. You have time — but probably less than you think.

The sales team confidence problem is significant. Twelve deals stuck for 2 quarters with confident forecasts suggests something other than permissions is the real blocker. Permissions may be necessary but not sufficient. Do you know how many of those 12 deals also have other objections that would surface post-permissions build?

This changes how I would think about Option B: you are not buying $300K ARR, you are buying an *option* on $300K ARR, subject to conditions the sales team may not be fully surfacing.

That shifts the risk calculus toward Option A — but I want to pressure-test the other side. What would have to be true for Option B to be the clearly right call?

---

**PM:** For Option B to be clearly right: the deals have to close, not just unblock. Also the $300K has to be additive, not just compensating for future churn from the AI problem.

**Claude:**

Exactly. And here is the compounding risk with Option B: if you build permissions, close some of those deals, and then lose SMB customers to AI competitors 6-9 months later, you have traded your core market segment for an enterprise segment you are not yet confident you can serve at scale. Enterprise customers won, SMB momentum lost.

The AI-first tools attacking your retention are likely targeting the segment that drives your usage density, your word-of-mouth, and your product velocity. SMB and prosumers move fast and talk loudly. Losing their NPS trust while chasing enterprise deals is a strategic compounding error that is hard to reverse.

I am not arguing Option A is obviously right. I am saying the framing of "AI feature vs. enterprise feature" understates the strategic stakes. This decision is really about which customer segment you are willing to prioritize — and that deserves to be explicit.

---

**PM:** That reframe is useful. Let me ask the pressure test from the other side: make the strongest case that we should do Option B first.

**Claude:**

The case for Option B:

Your retention is flat. AI competitive pressure is real but not yet critical. NPS drop is a warning, not a crisis. In 3 months, you can close $300K ARR — roughly 15-20% of current ARR if you are at a $1.5-2M run rate — and use that capital proof point to get more resources for Q4. Then you build the AI features from a stronger financial position, potentially with a dedicated team rather than cannibalizing your current one.

The AI features, if you rush them with 4 engineers in 3 months, will likely be underwhelming. The AI-first competitors have dedicated teams building nothing else. A rushed AI feature that users try and dismiss ("Notion has AI too, but it's not as good") may do more brand damage than no AI feature at all. Better to wait 6 months and ship something genuinely differentiated than ship a me-too feature that confirms users' perception that the incumbents are playing catch-up.

Enterprise customers, once landed, give you stability and reference customers. If those 12 deals have real intent — and a thorough review of the pipeline notes would tell you this — they provide a runway buffer that reduces existential risk.

That is the honest best case for Option B. Do you find that argument credible?

---

**PM:** Partially. The "rushed AI will hurt brand" point is real. But I do not trust the pipeline. Can you help me write the decision memo?

**Claude:**

**Decision Memo: Q3 Engineering Resource Allocation**

**The decision:** Invest Q3 engineering capacity (4 engineers, ~12 weeks) in Option A: AI writing assistance, with a lightweight permissions foundation as a supporting work stream.

**Context:** We face simultaneous pressure from AI-first entrants (NPS decline, stable churn for now) and stuck enterprise deals (12 deals, ~$300K ARR, pending permissions). We cannot do both well.

**Options considered:**
- Option A: AI writing assistant (full 4-engineer capacity)
- Option B: Enterprise permissions and access controls (full 4-engineer capacity)
- Hybrid: AI writing (3 engineers) + permissions foundation (1 engineer)

**Key factors:**
- Churn is currently stable but NPS has dropped 8 points in 2 quarters with AI features as top detractor theme. This is a leading indicator with a 2-3 quarter lag.
- Pipeline confidence is low — same 12 deals have been "close to closing" for 2 quarters; permissions may be necessary but not sufficient.
- A rushed AI feature risks brand damage; however, a modular architecture can deliver a high-quality narrow capability rather than broad feature parity.
- AI competitive window is not closed — we have time to do this deliberately rather than defensively.

**Decision:** Option A, with a modified scope. Ship one high-quality AI writing capability (scoped to the single most-used writing job in the product) rather than broad AI features. Allocate one engineer to build the permissions data model foundation so we are not starting from scratch in Q4. Use the quality bar of the AI feature as a forcing function to identify our differentiated angle, not just catch up.

**Risks and mitigations:**
- Risk: Enterprise deals lost to competitors in Q3. Mitigation: Sales team communicates a concrete Q4 permissions commitment with a hard date; a deal delay is better than a bad product.
- Risk: AI feature ships and is underwhelming. Mitigation: Narrow scope to one job, internal dogfood for 3 weeks before launch, explicit kill criteria if early retention signal is flat.

**Success criteria:** At end of Q3 — AI feature shipped and actively used by >40% of daily writers; at least 3 enterprise deals have received concrete Q4 permissions commitment in writing; NPS detractor theme on AI features reduced by 30%.

**Open questions:** What is actually blocking those 12 pipeline deals — get a PM to join 3 deal calls in the next 2 weeks to verify permissions is the real blocker.

---

## 4. Roadmap Strategy with Claude

### 4a. What Claude Needs

For roadmap strategy sessions, load this context:

```
Here is our current roadmap context:

Product vision: [one sentence]
Current quarter focus: [key themes and their rationale]
Next 2-3 quarter roadmap (rough): [list the major initiatives you are considering]
Key strategic bets: [the 2-3 things we are betting on being true about the market]
Team capacity: [engineers, designers, weeks per quarter]
Known dependencies: [what blocks what]
Key stakeholder priorities: [what engineering, sales, design, exec each care most about]
```

### 4b. Sequencing Analysis

```
Given the following roadmap initiatives and our strategic context, help me determine the optimal sequencing — which bets to make first and why.

For each proposed ordering, explain:
- What this sequence optimizes for (learning speed, revenue, risk reduction, etc.)
- What the key dependencies are that make this ordering necessary vs. optional
- What we would lose if we reordered the top two priorities

Then propose the sequencing you would recommend if you were the PM, and defend it as if you expected pushback from a skeptical CPO.

Initiatives (in rough priority order as currently planned):
[list]

Strategic context: [from context brief]
```

### 4c. Dependency Mapping

```
I want to understand the full dependency structure for this initiative. Think broadly — not just technical dependencies, but also: market conditions that need to exist, capabilities we need to have built, customer behaviors that need to change, organizational capabilities we need to develop.

For each dependency:
- Type: technical / market / organizational / behavioral
- Blocking or enabling: does this have to exist before we start, or does it just make us more likely to succeed?
- Time to resolve: how long would it take to address this dependency if we start now?
- Controllability: is this in our control, partially in our control, or outside our control?

Initiative: [describe]

After the dependency map, identify: which dependency is the most dangerous sleeper — the one we are most likely to overlook because it seems solved but is not?
```

### 4d. Scenario Planning

```
Our roadmap is built on [N] key strategic assumptions. I want to stress-test it against scenarios where one of those assumptions turns out to be wrong.

For each scenario:
- Describe what the world looks like when this assumption fails
- Assess: does our roadmap still make sense? What has to change?
- Identify: which roadmap items become MORE valuable, and which become LESS valuable?
- Recommend: what low-cost hedge could we build in now that keeps our options open?

Assumptions to test:
1. [Assumption 1 — e.g., "Enterprise buyers will prioritize our use case over more established vendors"]
2. [Assumption 2 — e.g., "AI models will improve fast enough to make our AI features genuinely better than alternatives within 6 months"]
3. [Assumption 3]

Current roadmap: [describe]
```

### 4e. Stakeholder Simulation

```
I am going to present this roadmap to a cross-functional review. Before I do, help me simulate how each stakeholder group will react.

For each group, tell me:
- Their primary concern with this roadmap
- The question they are most likely to ask
- The concession they will push for
- The argument that would most effectively address their concern

Stakeholder groups: Engineering leadership, Sales, Design, Executive team

Roadmap I am presenting: [describe]

After the simulation, help me identify: where are the strongest objections likely to come from? What do I need to prepare for that I have not prepared for yet?
```

### 4f. The Narrative

```
I need to turn this roadmap into a compelling story — not a list of features, but a narrative that explains why these bets are the right ones, in this order, at this time.

The narrative should:
- Start with the customer and market moment (why now is a critical window)
- Connect each quarter's theme to the previous one (show the logic, not just the list)
- Acknowledge what we are choosing NOT to do and why
- End with the world we are trying to create 18 months from now

Audience: [executive team / all-hands / board — specify]
Tone: [direct / inspirational / analytical]

Roadmap content: [describe]

After writing the narrative, flag: where did I have to make the strategic logic feel tighter than it actually is? Where are there genuine gaps in the logic that the audience might probe?
```

---

## 5. Prioritization at Depth

### 5a. RICE + Strategic Context

```
Please score these initiatives using an enhanced RICE framework. Use the standard formula (Reach × Impact × Confidence / Effort), then apply these strategic modifiers:

+20% if the initiative strengthens a core differentiator (not just adds features)
+20% if the initiative creates a data or network effect moat
+20% if the initiative is an enabling dependency for other high-value work
-20% if the initiative is table stakes (a competitor already has it, no differentiation value)
-20% if the initiative is highly reversible (we can do it later without significant cost)

Product context: [describe product, North Star metric, stage]

Initiatives:
[list each with brief description and any data you have on reach/impact]

Flag disagreements: any initiative where the adjusted score ranks significantly differently from the base RICE score? Those are the most strategically interesting — explain why the adjustment matters for each.
```

### 5b. The "What Are We Not Doing" Prompt

```
Here is our current roadmap for the next two quarters. I want to understand the implicit tradeoffs — what are we choosing NOT to do, and what does that reveal about our actual strategy versus our stated strategy?

For each major category of work we are deprioritizing:
- Name the category (e.g., performance, platform API, mobile experience)
- What is the cost of not doing this work (to users, to business, to competitive position)?
- Who inside the organization is most affected by this choice?
- What assumption are we making that makes this deprioritization acceptable? If that assumption is wrong, how bad does it get?

Roadmap: [describe]
Stated strategic priorities: [describe]

After the analysis, tell me: is there anything we are not doing that we are so consistently not doing that it is starting to accumulate strategic debt? What is the thing we keep pushing to "next quarter" that will eventually become a crisis?
```

### 5c. Customer Value vs. Business Value Matrix

```
Score the following initiatives on two dimensions and place them in a 2x2:

Customer value (high/medium/low): how much does this directly improve the customer's experience, solve their job, or reduce their pain?

Business value (high/medium/low): how much does this move our core business metrics — revenue, retention, acquisition, or defensibility?

For each initiative:
- Give the scoring with a 1-2 sentence rationale
- Categorize it in the matrix:
  * High customer + high business: invest now, highest priority
  * High customer + low business: invest if it protects retention (defensive moat)
  * Low customer + high business: invest with caution — if users do not feel the value, business metrics may not follow
  * Low customer + low business: deprioritize aggressively

Initiatives: [list]

After scoring, tell me: is there any initiative in the "low customer + high business" quadrant? Those are often the most dangerous — explain why each one might be tricking us.
```

### 5d. Constraint Analysis

```
Given the following constraint, help me find the optimal initiative portfolio.

Constraint: [e.g., "We have 6 engineers for Q3 instead of the 10 we planned for — 2 engineers left for competitive offers"]

For our current roadmap: which initiatives survive this constraint, which get cut, and which get descoped?

Optimize for: [choose one — revenue impact, retention, strategic positioning, or learning speed]

Provide:
- The revised portfolio (what we do with 6 engineers)
- What we cut and why
- The single biggest strategic risk of this constrained portfolio vs. the original plan
- Whether there is any creative re-sequencing that reduces the damage

Current roadmap: [describe]
```

### 5e. Regret Minimization

```
I want to apply Jeff Bezos's regret minimization framework to this prioritization decision.

It is 3 years from now. I am looking back at the choices I made in [current period]. 

For each of the initiatives I am currently considering:
1. If we did this and it worked: what does success look like in 3 years? What compounding value did it create?
2. If we did this and it failed: what did we lose? How recoverable is the loss?
3. If we did NOT do this and we watch a competitor do it successfully: how bad does that feel? Is there a version where we can never catch up?

The question is not "what will probably work" — it is "what choice will I most regret NOT making, even if it was risky?"

Initiatives to evaluate: [list]

After the analysis, tell me: is there one initiative I am currently deprioritizing that, on a regret basis, I should actually be doing first?
```

---

## 6. Competitive Strategy

### 6a. Competitor Simulation

```
I want you to play the role of the PM at [competitor]. You are a smart, well-resourced PM with a clear view of your competitive position.

Your product: [describe competitor's product briefly]
Your current strategic focus: [what they appear to be building/doing]

I am going to tell you a move our product is making. Your job is to respond as that PM — not politely, but with an honest internal competitive assessment. What do you do in response? What do you ignore? What worries you?

Our move: [describe your initiative, new feature, or pricing change]

After playing the competitor's response, step back and tell me: what does this simulation reveal about how well our move will actually work? Are we making a move that the competitor can easily neutralize, or one that creates a genuine problem for them?
```

### 6b. Disruption Analysis

```
How could a well-funded startup disrupt our current product in the next 3 years?

Think like a YC partner evaluating pitches against us. The startup has:
- $5M seed funding
- A team of 8 (4 engineers, 2 designers, 2 GTM)
- 18 months to prove product-market fit before needing Series A
- No legacy constraints — they can make choices we cannot make

Describe:
1. The attack angle (what assumption of our business model do they exploit?)
2. The beachhead customer (who do they win first, and why will those customers switch?)
3. The wedge feature (the single thing they build that is genuinely better than what we offer)
4. The compounding advantage (why, once they win the beachhead, do they keep winning?)

Our product: [describe]
Our key vulnerabilities (as honestly as you can): [describe]

After the disruption analysis: what is the single thing we could do in the next 6 months that would make this attack angle significantly harder to execute?
```

### 6c. Moat Analysis

```
Assess the defensibility of our product honestly. Do not tell me what I want to hear.

For each potential moat type, assess whether we actually have it and how durable it is:

1. Switching costs: how hard is it for a customer to leave? Is this sticky because of genuine value or because of lock-in?
2. Network effects: does the product get more valuable as more users join? Is this network effect real or hoped for?
3. Data moat: do we have proprietary data that gets better over time and is hard to replicate?
4. Economies of scale: are there meaningful cost advantages as we grow?
5. Brand: is our brand a genuine defensibility driver, or would customers switch for a 20% price cut?

For each moat: rate durability (strong / moderate / weak / none) and give the honest reason why.

Then tell me: what is the most honest assessment of our defensibility today? If a well-resourced competitor decided to clone us in 18 months, what exactly would they struggle to replicate, and what could they copy without difficulty?

Our product: [describe]
```

### 6d. Positioning Stress Test

```
Here is our current positioning statement: [paste positioning]

I want to stress-test each differentiation claim.

For each claim we make:
1. Is it actually true? (Be honest — list evidence for and against)
2. Do customers care? (Is this something buyers actively weight in purchase decisions?)
3. Is it unique? (What competitors make a similar claim? Is ours meaningfully different or just worded differently?)
4. Is it sustainable? (Can a competitor match this in 12-18 months with reasonable investment?)

After the stress test: which of our claimed differentiators survive all four tests? Which fail one or more? What does that imply we should actually be saying — what honest, defensible positioning remains after removing the claims that do not hold up?
```

---

## 7. Executive Communication

### 7a. The Strategy Narrative

```
I need to communicate the following strategic decision to an executive audience. Transform this into an exec-ready narrative.

Decision: [what we decided]
Context: [why this decision mattered]
Alternatives considered: [brief]
Rationale: [why we chose this]

The narrative should:
- Lead with the insight, not the process ("We are going upmarket because X" not "After analysis we considered...")
- Be ruthlessly clear about what we are choosing NOT to do
- Anticipate the two most likely exec objections and address them proactively
- End with a concrete ask (approval, resources, alignment)

Length: maximum 1 page or 3 slides worth of content.

After writing the narrative, tell me: where did you have to paper over genuine uncertainty? I want to know what is actually uncertain so I can prepare for those questions.
```

### 7b. The "So What" Filter

```
I have the following data and analysis: [paste data/analysis]

My job is to translate this into a strategic insight — not just "what is happening" but "what this means for our decisions."

For each significant finding in this data:
- State the finding plainly
- State the "so what" — what does this mean for our strategy?
- State the "now what" — what specific decision or action does this suggest?

Then: what is the single most important insight buried in this analysis that leadership most needs to hear, even if it is uncomfortable?
```

### 7c. OKR Alignment Check

```
Here is our roadmap for the next quarter and our current OKRs. Help me assess whether this roadmap actually moves the OKR metrics.

For each OKR:
- Which roadmap items contribute to this OKR, and how directly?
- What is the realistic impact on the metric if we execute well? (Be honest — most roadmap items have indirect or delayed impact)
- Are there OKRs that have NO roadmap coverage? That is a gap worth flagging.

Then: is there a roadmap item that consumes significant capacity but does not clearly connect to any OKR? Those items need either a clear connection articulated or a hard justification for why they are worth doing.

OKRs: [list]
Roadmap: [list]
```

### 7d. The Difficult Conversation

```
I need to prepare for a difficult conversation with [stakeholder — e.g., "my CPO who believes we should be building X" / "a sales leader whose pipeline depends on a feature we are deprioritizing"].

Help me prepare by:
1. Articulating their position as charitably as possible — what is the strongest version of their argument?
2. Identifying where their concern is legitimate and I should acknowledge it
3. Identifying where their concern is based on a different assumption than mine, and how to surface that constructively
4. Suggesting the framing that most likely leads to alignment rather than entrenchment

The conversation topic: [describe the disagreement]
My position: [what I believe and why]
Their likely position: [what I expect them to argue]

After the prep, tell me: is there anything in their position that should make me update my own view? Or is my analysis sound and the challenge is just communication?
```

---

## 8. Building a Strategy Memory

Great strategic thinking compounds. The decisions you made last quarter, and why, are context for the decisions you are making this quarter. Without a record, you repeat the same debates.

### The decisions.md file

Create a file called `decisions.md` in your team's shared workspace. Every significant strategic decision gets an entry:

```markdown
## [Decision Title] — [Date]

**Context:** Why did this decision need to be made? What triggered it?

**Options considered:** What were the real alternatives?

**Decision:** What did we choose?

**Rationale:** The 2-3 key factors that drove the choice.

**Assumptions:** What has to be true for this to have been the right call?

**Revisit trigger:** What event or signal would cause us to revisit this decision?
```

### How this compounds

At the start of any new strategy session, paste in the relevant entries from decisions.md. Claude can then flag:
- When a new decision is inconsistent with a prior one (and force you to articulate why)
- When assumptions from past decisions have been validated or invalidated
- When you are about to make a decision you already made and reversed

**Consistency check prompt:**

```
Here are the strategic decisions our team has made over the last [N months]: [paste decisions.md entries]

I am now considering: [new decision]

Does this new decision conflict with any of our prior decisions? If so: (1) is the conflict resolvable — did conditions change? Or (2) are we undermining a prior strategic commitment without having explicitly decided to?

Flag the most significant potential consistency issue and help me decide whether to update the prior decision or reconsider the current one.
```

### The retrospective prompt

```
[N months ago], we made the following strategic decision: [paste decision entry]

Our assumptions were: [list]

Here is what actually happened: [describe outcomes]

Help me run a decision quality retrospective:
1. Which assumptions held? Which did not?
2. Was the decision good given what we knew then, even if the outcome was worse than expected? (Decision quality is not the same as outcome quality)
3. What would we do differently with the information we had at the time — not hindsight?
4. What does this tell us about our decision-making process that we should apply going forward?
```

---

## 9. Complete Worked Example: A Full Strategy Session

**Scenario:** Fielder is a B2B SaaS product for field service management — scheduling, dispatching, and job tracking for small businesses with mobile workforces (HVAC companies, plumbing contractors, landscaping). ARR: $4.2M, growing 40% YoY. New threat: ServiceTitan (a well-funded incumbent serving enterprise) has announced a "Lite" tier targeting Fielder's exact segment. Fielder has 3 months to decide its strategic response.

---

**PM (context brief loaded):**

We are facing a strategic decision I want to think through carefully. ServiceTitan — the dominant enterprise player in our space — just announced a Lite tier priced at $79/user/month, close to our $89. They are explicitly targeting small field service businesses, our core market. We have 3 months before their Lite tier is generally available. Our options as I see them: (1) Go upmarket — move to serve mid-market and early enterprise customers where ServiceTitan is already playing but where we have no relationships yet. (2) Hold the middle — differentiate on simplicity and SMB-focus and bet that ServiceTitan's Lite will be too complex for our customers. (3) Cut price — compete on cost, accepting margin compression to retain volume.

**Claude:**

This is a classic "incumbent moves down" competitive scenario. Let me give you the pattern first, then get specific to your situation.

The historical pattern: when an incumbent moves downmarket, they almost always bring too much complexity with them. The Lite tier is rarely actually lite — it is the full product with features disabled, not a product redesigned from scratch for the simpler buyer. This is why enterprise incumbents rarely successfully destroy SMB specialists: the SMB buyer is not buying a simpler version of the enterprise product, they are buying a different product entirely, one built around their workflow and their level of sophistication.

The evidence question is: what do your customers actually hate about complex tools? If the answer is "setup time and onboarding" — that is an execution problem ServiceTitan can solve with better CS. If the answer is "the mental model is wrong for how we work" — that is a product architecture problem ServiceTitan cannot solve by pricing down.

Which is it for your customers?

---

**PM:** Strongly the second. Our NPS leaders consistently say things like "it just works the way we think about jobs" and "I don't have to train my guys to use it." ServiceTitan's workflow is built around enterprise scheduling logic — dispatchers managing multiple crews with complex routing. Our customers are owner-operators who are also the dispatcher and often on the tools themselves.

**Claude:**

That is the most important insight in this conversation. Write it down.

ServiceTitan is not actually entering your market — they are entering a segment adjacent to your market that happens to overlap on the buyer profile (small field service business). The product fit gap you are describing is structural, not a temporary advantage. They cannot redesign their dispatch workflow for owner-operators without breaking what their enterprise customers need.

This changes the framing significantly. You are not facing a competitive threat — you are facing a marketing threat. ServiceTitan will acquire some of your potential customers, and some of your existing customers will trial the Lite tier. But the customers who convert and stay are likely the ones for whom the enterprise dispatch workflow is actually appropriate — they are growing out of your ICP.

This suggests the optimal response is not upmarket or downmarket — it is to shorten the distance between "ServiceTitan Lite trial" and "back to Fielder." The customers who try ServiceTitan and return become your most loyal customers and your best case studies.

But let me stress-test this before you rely on it. What are your top 3 churn reasons currently, and do any of them overlap with what ServiceTitan Lite would solve?

---

**PM:** Top churn reasons: (1) Customer goes out of business (~35% of churn, structural). (2) Switches to pen-and-paper after employee turnover disrupts the workflow (~25%). (3) Grows and wants features we don't have — reporting, multi-crew dispatching, integrations with QuickBooks Enterprise (~25%). (4) Price sensitivity (~15%).

**Claude:**

Churn reason 3 is the key number. Twenty-five percent of your churn is customers growing *into* ServiceTitan territory. That is not competitive threat, that is actually a sign your product-market fit is tight — you are serving early-stage businesses and losing them gracefully as they mature.

But here is the strategic implication: ServiceTitan Lite is likely to accelerate this churn by being a more convenient next step. Instead of churning at "$89k in revenue with 12 employees," they may now churn at "$60k with 8 employees" because there is a familiar brand option that feels like a logical upgrade.

This means the upmarket argument has real merit — but not the way you originally framed it. You do not need to compete with ServiceTitan in enterprise. You need to capture the "growing out of Fielder" segment before ServiceTitan Lite does.

What are the 3-4 features that your churning "grown too big" customers most commonly cite? Those are your roadmap for the next 2 quarters.

---

**PM:** They want: multi-crew dispatching with visual route planning, QuickBooks Enterprise sync (we have QuickBooks Online only), customer portal for job status, and custom reporting. These are all on our backlog but consistently lose to "make the core better" roadmap debates.

**Claude:**

You have a classic incumbent trap problem — you are protecting your core market at the cost of losing your growth trajectory. The "make the core better" argument is right for retention of current customers. It is wrong as a total portfolio strategy because it does not address the 25% of churn that represents customers who would stay if you grew with them.

The reframe I would bring to your exec team: the ServiceTitan Lite announcement is not a threat to your current customers (your structural advantages there are real). It is a forcing function that makes explicit a strategic drift risk that was already happening: you were losing your growth-stage customers to ServiceTitan by default, and ServiceTitan just made that path more visible.

Strategic recommendation: a "Growth Tier" track — 2-3 features that serve the 5-25 employee business that is becoming sophisticated enough to need more. Not an enterprise tier. A "you are graduating" tier. This addresses the 25% churn risk, differentiates you from a Lite competitor who is coming down from complexity rather than growing up with your customers, and positions the price difference ($89 Fielder vs. $79 ServiceTitan Lite) as "Fielder grows with you, ServiceTitan makes you start over."

---

**PM:** Can you write the decision memo?

**Claude:**

**Strategic Decision Memo: Response to ServiceTitan Lite**
*Fielder — June 2026*

**The decision:** Do not respond to ServiceTitan Lite with a price reduction or a retreat to a defensive SMB position. Instead, accelerate the Fielder Growth Tier — a dedicated track of features that serves customers in the 5-25 employee, $500K-$3M revenue stage who are currently churning to more complex tools.

**Context:** ServiceTitan announced a Lite tier at $79/user targeting our core segment. This announcement revealed a structural vulnerability in our portfolio: 25% of our churn is customers growing into functionality we do not yet offer. ServiceTitan Lite accelerates the path out — not by being better than us for our ICP, but by being a visible off-ramp for growing customers who have not found a reason to stay.

**Options considered:**
- Price reduction: addresses only the 15% of churn that is price-driven; compresses margin without solving the product gap
- Upmarket (full enterprise push): requires sales motion and product investment we are not ready for in 3-6 months
- Defensive SMB hold: protects current customers but abandons the growth segment to ServiceTitan by default
- Growth Tier (recommended): addresses the 25% churn that represents growing customers, without abandoning our SMB core

**Key factors:**
- Our structural advantage is a workflow model built for owner-operators — ServiceTitan cannot replicate this without breaking their enterprise product
- The 25% "grew out of us" churn is not inevitable; it represents customers we could retain with targeted feature investment
- A Growth Tier positions our pricing difference ($89 vs. $79) as value rather than a vulnerability: "Fielder grows with you"

**Decision and rationale:** Build the Fielder Growth Tier over Q3-Q4, prioritized by: (1) QuickBooks Enterprise sync (removes the most-cited churn trigger), (2) multi-crew dispatching with visual map (addresses the workflow gap for 8-15 employee teams), (3) customer portal (creates a stickiness layer that ServiceTitan Lite will not have at launch). Target price: $129/month for growth tier, creating a natural upgrade path rather than a price fight.

**Risks and mitigations:**
- Risk: Growth Tier features delay core improvements, hurting retention of current SMB customers. Mitigation: Ring-fence Growth Tier to 40% of Q3 capacity; core improvements remain on roadmap.
- Risk: We execute poorly on Growth Tier and it validates ServiceTitan's claim of being the more capable option. Mitigation: Scope tightly to 3 features; avoid feature bloat at launch.

**Success criteria:** By end of Q4 — churn from "grew out of Fielder" segment reduced from 25% to 15% of total churn; at least 150 customers upgraded to Growth Tier within 60 days of launch; NPS among 8-25 employee segment improves by at least 5 points.

**Open question requiring immediate answer:** Are our churned customers in this segment actually going to ServiceTitan, or to other alternatives? A 10-minute win/loss call protocol with churned customers in the next 3 weeks will answer this before we commit.

---

## 10. Prompts to Save Immediately

A library of 20 complete, copy-paste-ready strategy prompts. Each works with a context brief loaded first.

---

**1. The Insight Surfacer**
```
Here is our product usage data / customer feedback / market research: [paste data]. Do not summarize it back to me — I have already read it. Tell me what this data implies that is non-obvious. What are the 2-3 insights buried here that a careful strategist would extract but a casual reader would miss?
```

**2. The Devil's Advocate**
```
I am going to present a strategy I believe in. Your job is to be the most rigorous, well-informed skeptic in the room — not hostile, but genuinely unconvinced. Ask me the three questions that most undermine my confidence if I cannot answer them well.

My strategy: [describe]
```

**3. The Pattern Match**
```
What historical analogues exist for our current situation? I am facing: [describe situation]. Find 2-3 cases from other industries or time periods where a company faced a structurally similar challenge. What did they do? What happened? What is transferable to our situation and what is not?
```

**4. The Minimum Viable Strategy**
```
If we could only make ONE strategic bet this year — not a feature, a strategic bet — what should it be given our current position? Argue for a single clear choice. Do not give me a balanced view of multiple options. Commit to one and defend it.

Our context: [brief description]
```

**5. The Reverse Engineering**
```
[Competitor] has made the following move: [describe]. Reverse-engineer their strategy. What do they believe about the market? What are they betting on? What does this move tell us about where they think the value is going? And what does it imply about what they are NOT doing — and why?
```

**6. The Honest Assessment**
```
I am going to describe our current strategy. Give me an honest assessment — not brutal for the sake of being brutal, but without softening the parts that need to be challenged. Specifically: (1) where is the logic solid? (2) where is the logic weak or assumption-dependent? (3) what would a smart outside investor say about our approach?
```

**7. The Build vs. Buy vs. Partner Analysis**
```
We need capability X. Help me think through build vs. buy vs. partner.

For each option:
- Time to capability
- Cost (not just money — opportunity cost of the team working on it)
- Quality of outcome (best case)
- Risk profile (what goes wrong)
- Strategic optionality (does this choice close or open future options)

Capability needed: [describe]
Our context: [team size, technical capability, timeline, strategic importance of this capability]
```

**8. The Narrative Test**
```
Here is how I explain our strategy: [describe your current narrative]. Play the role of a smart customer, a smart investor, and a smart competitor who each heard this narrative. What does each of them take away? Where does each find it unclear or unconvincing? Does each audience understand what you are doing and why it matters to them?
```

**9. The Strategic Calendar**
```
Given our strategy and the initiatives we are planning, build me a strategic calendar for the next 12 months. Not a project plan — a calendar of decision points, market windows, and timing risks.

For each quarter: what decisions need to be made by then? What will we know by then that we do not know now? What external events (competitor moves, market cycles, renewal periods) create timing constraints?

Strategy context: [describe]
```

**10. The Competitive Response Playbook**
```
We are about to launch [initiative/product/feature]. Before we do, help me build a competitive response playbook.

For each major competitor: what is the most likely response — ignore, match, undercut, or counter-position? What signals would we see that each response is coming? And what is our counter-response to their most dangerous move?

Our initiative: [describe]
Competitors: [list]
```

**11. The Strategic Alignment Check**
```
Here are our stated company values and strategic principles: [list]. Here is a roadmap decision we are about to make: [describe].

Does this decision actually reflect our values and principles? Or are we making a pragmatic choice that contradicts them? If there is a conflict, name it explicitly — I would rather know before I present this to the team.
```

**12. The Founder's Letter Test**
```
Write a section of a shareholder letter (Bezos-style, end of year) explaining the strategic decision we made in [period]. Write it as if the decision worked out well and we are explaining the logic to long-term shareholders. What did we see that others missed? What bets did we make and why?

Decision context: [describe what we decided]

After writing it, tell me: where does the letter feel like a rationalization of an obvious choice, vs. where does it feel like genuine strategic insight? The parts that feel obvious probably are not our real edge.
```

**13. The Customer Letter**
```
Write a letter from one of our best customers — a real power user of the product — explaining why they love us and why they would never switch to [competitor]. Write it in their voice, not our marketing voice.

Then: how much of what they would actually say matches what we claim in our positioning? Where are the gaps?

Customer profile: [describe your typical power user]
```

**14. The Next CEO Test**
```
Our current CEO or CPO is leaving. The board is hiring from outside. What strategy would an incoming executive — who does not have our institutional assumptions or sunk-cost biases — likely want to change? What would they see as legacy choices that made sense at the time but no longer reflect the best path?

This is not a criticism — it is a useful outside-in view. What is our strategic path of least resistance vs. our optimal strategic path?

Context: [describe current strategy and key decisions made over the last 2 years]
```

**15. The Investment Thesis**
```
Write the investment thesis for our company as a Series B VC would write it after a deep dive. Include:
- The market they believe we are building into
- The core insight about why we win
- The key risks they see and why they are investing anyway
- The scenario where we become a $500M+ company
- The scenario where we plateau at $30M

Be honest about the risks — a real investment memo would be. Then tell me: what does this thesis reveal about what we most need to prove in the next 12 months?

Company context: [describe]
```

**16. The Jobs-at-Risk Scan**
```
Our product currently helps customers do [jobs]. Which of these jobs is most at risk of being solved by AI in the next 2-3 years — not our AI, anyone's AI — such that the job itself disappears or becomes dramatically less valuable?

For each at-risk job: how much of our current revenue depends on it? What adjacent job would those customers need solved next? And what does this mean for our roadmap today?
```

**17. The Pricing Logic**
```
Analyze our current pricing structure and tell me what it optimizes for, implicitly. Then tell me: what would our pricing look like if we optimized for [alternative goal — e.g., land-and-expand in enterprise / maximizing SMB volume / capturing maximum value from power users]?

Current pricing: [describe tiers and prices]
Current goals: [describe what you say pricing is trying to do]
```

**18. The Talent Bet**
```
Our strategy for the next 18 months requires the following capabilities: [list]. Honestly assess: do we have these capabilities today? Which ones are most at risk of being bottlenecks? And for each capability gap, what is the fastest path to close it — hire, partner, or build the capability internally through a focused initiative?
```

**19. The Distribution Reality Check**
```
Here is our go-to-market strategy: [describe]. Play the role of a head of sales or growth who has seen 50 companies try similar go-to-market approaches. What does not work the way product teams think it will? What assumption are we making about customer acquisition that is likely wrong? What would you bet on vs. test first?
```

**20. The Exit Clarity Prompt**
```
At the end of this strategy session, I want a clear, committed output: one sentence stating the strategic decision, the single most important reason we chose it, and the single biggest bet embedded in that choice.

No hedging. No "it depends." A decision and a clear articulation of the bet we are making.

[This prompt goes at the end of any strategy session where you want commitment, not more analysis.]
```

---

*This guide was built from practitioner research including Lenny's Newsletter, Reforge, First Round Review, and a16z product strategy content. The frameworks are not new — JTBD, RICE, pre-mortems, and steelmanning all predate AI. What AI changes is the cost of running them: a framework that used to take a workshop now takes 15 minutes. Use the time saved to run more of them.*
