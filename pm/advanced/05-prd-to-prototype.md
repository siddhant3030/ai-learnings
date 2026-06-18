# PRD to Prototype in 30 Minutes: A PM's Hands-On Guide to Claude Code

You have a product idea. Normally you'd write a spec, get it prioritized, wait two weeks for a sprint slot, then get a mockup back that still doesn't answer whether the idea actually works. This guide shows you a different path: a working prototype in 30 minutes, built by you, with no coding required.

This is a step-by-step tutorial. Every prompt, command, and template is copy-paste ready.

---

## 1. What "Prototyping" Means for a Non-Technical PM

Forget pixel-perfect mockups. A prototype here means **a working thing you can interact with** — you click a button, data appears, filters change, results update. It runs in a browser. You can share a link. You can show it to users and watch what they do.

Claude Code builds this for you. You describe what you want in plain English. It writes the code, installs the dependencies, and tells you how to run it. You never open a code file.

**What Claude Code can build for you:**

- **Streamlit apps** — Python-based web apps that run locally or on the cloud. A chart, a filter, a table, a form — Streamlit makes these easy. Cat Wu, Head of Product for Claude Code at Anthropic, used Claude Code to build Streamlit apps analyzing thousands of user feedback responses without writing a single line of code by hand.
- **Data dashboards** — Upload a CSV export from your product analytics, CRM, or survey tool. Ask Claude to build a dashboard from it.
- **Internal tools** — A form that generates a document. A search tool over your notes. A calculator for a specific metric.
- **API prototypes** — Test what an API endpoint might return before engineering builds it.
- **Analysis scripts** — Run a one-time analysis on a dataset and get back a chart or table.
- **Automated workflows** — Pull data from a spreadsheet, transform it, send a summary somewhere.

**The Dennis Yang (Chime) workflow:** Dennis Yang, Principal PM for Generative AI at Chime, writes a product requirement in markdown — essentially a brief — and then prompts Claude Code to build a working prototype from it. Brief to prototype in a single session, under 20 minutes. He keeps PRDs in markdown files under source control and generates Jira tickets and status reports from the same files.

**The Cat Wu (Anthropic) workflow:** Wu joined Anthropic's research PM team and immediately used Claude Code to analyze large-scale user feedback by building Streamlit apps, create RL environments to understand model training, and run eval systems — all without writing code herself. Her stack: Claude.ai for strategic thinking, Claude Code for building prototypes and scripts.

**Why this matters:** You can validate an assumption in an afternoon instead of a quarter. A two-week sprint just to review a mockup becomes a 30-minute build-and-show. You bring stakeholders a working demo, not a slide. Users can interact with the actual thing and tell you immediately if the concept works.

---

## 2. Twenty Things You Can Build Right Now

Each of these is a real prototype a non-technical PM can build with Claude Code in under 45 minutes.

**1. Data analysis dashboard from a CSV export**
Takes a CSV file you already have (product analytics, cohort data, sales data) and builds a filterable, chartable dashboard. Use it when you need to understand a dataset before deciding what to build next. Ask: "Build a Streamlit dashboard from this CSV with filters by date and user segment."

**2. User interview synthesis tool**
Paste interview transcripts as text files. Ask Claude to extract themes, pain points, and Jobs-to-be-Done patterns. Use it after a research sprint to turn 10 hours of transcripts into a 2-page synthesis in 20 minutes.

**3. Competitor pricing tracker**
Paste competitor pricing pages into a spreadsheet. Ask Claude to build a simple app showing pricing side-by-side, with a change log for when you update the data week over week. Use it for competitive review meetings.

**4. Feature prioritization calculator**
Input features, score them on reach, impact, confidence, and effort. Claude builds a RICE score calculator with a sortable output table. Use it before roadmap planning.

**5. A/B test results analyzer**
Paste test results (variant, conversions, samples) into a CSV. Ask Claude to build a significance calculator with a plain-English summary. Use it when you have test data but no analyst handy.

**6. Customer feedback tagger**
Paste customer support tickets or NPS comments. Ask Claude to tag each by theme (performance, pricing, UX, missing feature). Use it quarterly to understand what's actually driving churn and NPS.

**7. NPS score tracker**
A simple app that takes NPS survey exports and shows trend over time, breakdown by segment, and top verbatims per score bucket. Use it before quarterly business reviews.

**8. Internal search tool over your documents**
Give Claude a folder of meeting notes, PRDs, or research reports. Ask it to build a simple search interface over them. Use it when your team's institutional knowledge is scattered across Notion pages.

**9. PRD generator from a brief**
Write a one-page brief. Ask Claude to generate a full PRD in your team's standard format. Use it when you're converting a rough idea into something engineering can react to.

**10. Meeting notes to action items extractor**
Paste raw meeting notes. Ask Claude to extract action items with owners and due dates, and output them as a formatted list or a CSV you can paste into Jira. Use it after every product review.

**11. Onboarding flow prototype**
Describe the steps in your onboarding flow. Ask Claude to build a simple click-through prototype showing each screen as a Streamlit page. Use it for usability testing before engineering writes a line of code.

**12. Email campaign analyzer**
Paste email performance data from Mailchimp or HubSpot. Ask Claude to build a dashboard showing open rate, click rate, and conversion by campaign, segment, and date. Use it when the native analytics aren't flexible enough.

**13. Cohort retention calculator**
Input a user cohort CSV with signup dates and activity dates. Ask Claude to calculate and chart week-over-week retention curves. Use it when you need to show retention trends to leadership but don't have a data analyst.

**14. Feature flag toggle UI**
Describe your feature flags and their current state. Ask Claude to build a simple toggle interface showing which flags are on or off. Use it as a demo tool when explaining your rollout strategy to stakeholders.

**15. API explorer for your own product**
Give Claude your API documentation. Ask it to build a simple form-based interface for calling key endpoints and seeing responses. Use it to help customer success teams understand the API without using code.

**16. Survey response analyzer**
Export survey results as a CSV. Ask Claude to build an analysis tool showing distribution, trends, and open-ended response themes. Use it to synthesize quarterly user surveys in one afternoon.

**17. Roadmap visualizer**
Paste your roadmap items into a CSV with quarter, theme, team, and status. Ask Claude to build a visual roadmap view. Use it to create a shareable roadmap without Productboard or Jira formatting headaches.

**18. Sprint planning tool**
Input your backlog with estimates and priority scores. Ask Claude to build a capacity calculator that shows how many stories fit in a sprint given team velocity. Use it during sprint planning prep.

**19. Customer health score calculator**
Define health score inputs (login frequency, feature adoption, support tickets, payment status). Ask Claude to build a tool that takes a customer CSV and outputs a health score and tier for each account. Use it before account review calls.

**20. Churn prediction input tool**
Build a form where your CS team inputs risk signals for an account and gets back a risk tier (red/yellow/green). Not a real ML model — a rule-based calculator. Use it to standardize how CS teams flag at-risk accounts.

---

## 3. Getting Started with Claude Code (30-Minute Setup)

### What the terminal is

The terminal is a text-based interface to your computer. You type commands, press enter, and the computer runs them. You do not need to understand what the commands do — you just need to type them correctly. On a Mac, open it by pressing Command + Space and typing "Terminal."

### Prerequisites

You need a Claude Pro subscription ($20/month) or higher. Go to claude.ai and subscribe before continuing.

You also need Node.js installed on your Mac. Open Terminal and type:

```
node --version
```

If you see a version number (like `v20.11.0`), you have it. If you see "command not found," install it by typing:

```
brew install node
```

If brew itself isn't installed, paste this in Terminal first:

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Install Claude Code

Open Terminal and type:

```
npm install -g @anthropic-ai/claude-code
```

Wait for it to finish (30–60 seconds). Then verify it worked:

```
claude --version
```

You should see a version number. If you see "command not found," restart Terminal and try again.

### Your first session

Navigate to a folder you want to work in. Create a new folder for your prototype:

```
mkdir my-prototype
cd my-prototype
claude
```

Claude Code will open in your terminal. It will ask you to authenticate — follow the prompts to connect to your Claude account.

You are now in a Claude Code session. You can type in plain English. Claude will read and write files in the folder you started from.

### The mental model

Claude Code is not a chatbot. It is an agent that can read files, write files, and run commands on your computer. When you say "build a Streamlit app from this CSV," it:
1. Reads the CSV file
2. Writes a Python file with the app code
3. Installs the required libraries
4. Tells you the command to run it

You describe what you want. Claude builds it. You run it in a browser.

---

## 4. The Brief-to-Prototype Workflow

Every prototype follows the same five steps.

### Step a: Write the brief

One page. Not a PRD — a brief. Four things:

1. **What it does** — One sentence. "A dashboard that shows user activation rates by cohort week."
2. **Who uses it** — "Me and the product team, in a weekly meeting."
3. **What data it needs** — "A CSV export from Amplitude with user ID, signup date, and first key action date."
4. **What it outputs** — "A chart showing retention curve by cohort, and a table showing the raw numbers."

**Brief template:**

```markdown
# Prototype Brief: [Name]

## What it does
[One sentence describing the core function]

## Who uses it and when
[Who will use this and in what context]

## Data inputs
[What files, CSVs, or manual inputs does it need]

## Output
[What does the user see or get from it]

## Nice to have (optional)
[Any secondary features, lower priority]

## What I want to learn from it
[The assumption you're testing with this prototype]
```

### Step b: Set up context with CLAUDE.md

Create a file called `CLAUDE.md` in your prototype folder. This file tells Claude what you're building. Claude reads it at the start of every session.

```markdown
# Prototype Context

## What I'm building
[Copy the "What it does" section from your brief]

## Tech preferences
- Use Streamlit for all web UIs
- Use Python
- Keep the code simple — I am not a developer and will not be maintaining this
- Add comments explaining what each section does

## Style preferences
- Clean, minimal UI
- Charts should have clear labels and titles
- Use a white background

## When something goes wrong
- Tell me the exact command to run to see the error
- Suggest the fix in plain English before writing any code
```

### Step c: The kickoff prompt

This is the message you send at the start of every prototype session. Copy this template and fill in the blanks:

```
I want to build a prototype. Here is the brief:

[Paste your brief here]

Please:
1. Ask me any clarifying questions before you start
2. Describe what you're going to build in plain English
3. Build it as a Streamlit app
4. Tell me the exact command to run it when you're done
5. Keep it simple — I want something working quickly, not perfect

Start with the clarifying questions.
```

The most important line is "Start with the clarifying questions." This forces Claude to surface ambiguities before writing any code, which saves you from rebuilding the same thing twice.

### Step d: The iteration loop

Once the app runs, you refine it through conversation. Good iteration prompts:

- "The chart is showing data by week, but I want to see it by month instead."
- "Add a filter at the top so I can select a specific date range."
- "The table is too wide — only show the top 5 rows by default with an 'expand' option."
- "Export the table to a CSV when I click a button."
- "The colors are hard to distinguish — use a colorblind-friendly palette."
- "When the filter changes, the chart should update automatically."

Each of these is a complete instruction. Claude will make the change and tell you to refresh the app.

### Step e: Testing

To run your app, type this in Terminal (from the folder where your files are):

```
streamlit run app.py
```

A browser tab opens automatically at `http://localhost:8501`.

When testing, check three things:
1. Does it do what the brief said it would do?
2. Can someone unfamiliar with the data understand what they're looking at?
3. Does it answer the assumption you wanted to test?

Share it with one other person before presenting it. Their first reaction to the UI tells you more than 10 minutes of self-review.

### Step f: When to throw it away

Prototypes are disposable. The goal is to learn something, not to ship the prototype. Throw it away when:

- You've validated or invalidated the assumption in the brief
- Engineering is ready to build the real thing
- The complexity to maintain it exceeds the value of having it

The handoff to engineering is: "Here is the prototype I built to validate [assumption]. It worked/didn't work because [finding]. Here is what the real version should do."

---

## 5. Worked Example 1: Data Analysis Dashboard

**Time: 25 minutes**

### The brief

```markdown
# Prototype Brief: User Activation Dashboard

## What it does
Shows how many users complete key activation steps within 7 days of signing up,
broken down by week cohort.

## Who uses it and when
Product and growth team, in our weekly Monday standup. We want to see if our
onboarding changes are improving activation.

## Data inputs
A CSV export from our analytics tool with columns:
- user_id
- signup_date (YYYY-MM-DD)
- first_dashboard_created (date or empty)
- first_data_source_connected (date or empty)
- first_pipeline_run (date or empty)

## Output
1. A line chart showing % of users who completed each step within 7 days,
   by signup week cohort
2. A table showing the raw numbers for each cohort

## What I want to learn from it
Whether the onboarding changes we shipped 3 weeks ago are showing up in the data.
```

### The kickoff prompt

```
I want to build a prototype. Here is the brief:

[paste of brief above]

Please:
1. Ask me any clarifying questions before you start
2. Describe what you're going to build in plain English
3. Build it as a Streamlit app
4. Tell me the exact command to run it when you're done
5. Keep it simple — I want something working in 20 minutes

Start with the clarifying questions.
```

### The conversation

**Claude:** "Before I start, a few questions:
1. Should the cohort be defined by signup week (Monday–Sunday) or calendar month?
2. When you say '7 days,' do you mean calendar days or 7 days from the user's signup date specifically?
3. Do you want to see all three steps on the same chart, or separate charts per step?
4. Do you have a sample CSV I can look at, or should I generate some sample data to build against?"

**You:** "Signup week, calendar days from signup, all three steps on the same chart, and generate sample data — I'll replace it with real data later."

**Claude:** "Got it. Here's what I'll build: a Streamlit app with a line chart showing activation rate (percentage of that week's signups who completed each step within 7 days) for the last 12 weeks. Below the chart, a table with the raw numbers. I'll generate realistic sample data so you can see it immediately. Give me a few minutes."

[Claude writes app.py, generates sample_data.csv, installs Streamlit and pandas]

**Claude:** "Done. Run this command to start the app:

```
streamlit run app.py
```

It will open in your browser automatically."

**You:** [runs it, sees the chart] "This is great. Can you add a dropdown at the top so I can filter to show only one step at a time?"

**Claude:** [makes the change] "Done. Refresh your browser and you'll see the dropdown."

**You:** "Can you add a vertical dashed line on the week we shipped the onboarding change? I want to mark it as 'Onboarding v2 launch.'"

**Claude:** "I'll add a date input at the top where you can enter the launch date, and it will draw a labeled vertical line on the chart. What date was the launch?"

**You:** "April 7, 2025. But make it a text input so I can change it without restarting the app."

[Claude updates the app. Total time: 22 minutes.]

### What the app looks like

A clean white page with a title ("User Activation Dashboard"), a dropdown to filter by step, and a date input labeled "Onboarding change date." The chart shows three colored lines (one per activation step) across 12 week cohorts on the x-axis and percentage on the y-axis. A vertical dashed line at the selected date is labeled "Onboarding v2 launch." Below the chart, an expandable table shows the raw numbers.

### What the PM learned

The chart showed a clear step-change in "first dashboard created" rates after week 15 — exactly when onboarding v2 launched. The two harder steps (data source connected, pipeline run) hadn't moved. This told the team that the new onboarding was helping users understand the product but not completing setup. The next sprint prioritized the connection flow.

Total time: 25 minutes.

---

## 6. Worked Example 2: User Research Synthesis Tool

**Time: 20 minutes**

### The brief

```markdown
# Prototype Brief: Interview Synthesis Tool

## What it does
Takes raw user interview transcripts as text input and produces a structured
Jobs-to-be-Done synthesis: the core job, supporting jobs, pain points,
and key quotes.

## Who uses it and when
Me, after a research sprint. I have 8 interview transcripts and I want to
synthesize them in one afternoon instead of two days.

## Data inputs
Text files — one per interview transcript. Each file is the raw transcript
from a 30-minute user interview.

## Output
A structured synthesis document with:
1. The primary JTBD (one sentence)
2. 3–5 supporting jobs
3. Top 5 pain points (with frequency count)
4. Representative quotes for each pain point
5. Opportunities (areas where pain is high and current solutions are weak)

## What I want to learn from it
Whether a pattern is emerging across interviews that would justify a new feature
investment in Q3.
```

### The kickoff prompt

```
I want to build a prototype research synthesis tool. Here is the brief:

[paste of brief above]

I have 8 interview transcript files (plain text, .txt format). I want to
be able to:
1. Upload multiple transcript files at once
2. Click "Synthesize" and get the JTBD output
3. Export the synthesis to a markdown file I can share

Please build this as a Streamlit app. Ask clarifying questions first.
```

### The conversation

**Claude:** "A few questions before I start:
1. Should the synthesis be generated by calling the Claude API, or should I use simpler pattern matching? (Using the Claude API will give much better synthesis quality but requires an API key.)
2. Do you want to see individual interview summaries in addition to the cross-interview synthesis?
3. Should the quote extraction prefer verbatim quotes or paraphrased summaries?"

**You:** "Use the Claude API — I have an API key. Yes to individual summaries. Verbatim quotes only."

**Claude:** "Perfect. I'll build an app with a file uploader for multiple .txt files, a button that calls the Claude API to synthesize each transcript and then do a cross-interview synthesis, and an export button. You'll need to set your API key as an environment variable. I'll give you the command for that. Starting now."

[Claude builds the app — 8 minutes of generation]

**Claude:** "Done. Before running it, set your API key by typing this in Terminal (replace with your actual key):

```
export ANTHROPIC_API_KEY=your-key-here
```

Then run:

```
streamlit run synthesis_tool.py
```

Upload your .txt files, click Synthesize, and it will process each one and then generate the cross-interview synthesis. It takes about 2 minutes to run on 8 transcripts."

**You:** [runs it] "This is excellent. One change — I want the pain points to show a count like (mentioned by 5 of 8 users) next to each one."

**Claude:** [updates the prompt used for synthesis] "Done. Re-upload your files and run it again."

### What the PM got

A synthesis document showing: the primary job ("make our program data credible to funders without a data team"), four supporting jobs, five pain points each with a frequency count and two verbatim quotes, and three opportunities ranked by pain severity. Total research synthesis time: 2 hours including 20 minutes to build the tool.

---

## 7. Worked Example 3: Internal Competitive Tracker

**Time: 35 minutes**

### The brief

```markdown
# Prototype Brief: Competitive Pricing Tracker

## What it does
A simple web app that shows competitor pricing and key features side-by-side.
Updated from a Google Sheet. Shows what changed since last week.

## Who uses it and when
Product and sales team, weekly. We spend 30 minutes in competitive review
updating a spreadsheet manually. I want to replace that with a shareable
dashboard.

## Data inputs
A Google Sheet (exported as CSV) with columns:
- competitor_name
- plan_name
- monthly_price
- annual_price
- feature_1 through feature_10 (boolean: Y/N)
- last_updated (date)
- notes

## Output
1. A pricing comparison table (competitors as columns, plans as rows)
2. A feature matrix (features as rows, competitors as columns)
3. A "changes this week" section showing what's different from the previous
   week's CSV

## Nice to have
Send a Slack message with the summary of changes each Monday morning.

## What I want to learn from it
Whether seeing the competitive landscape weekly changes how fast we respond
to pricing moves.
```

### The kickoff prompt

```
I want to build a competitive pricing tracker. Here is the brief:

[paste of brief above]

For the "changes this week" feature, I'll keep last week's CSV as a separate
file and upload both. Start with the core dashboard — skip the Slack integration
for now, we can add it after the basic thing works.

Ask me clarifying questions first.
```

### The conversation

**Claude:** "Questions:
1. How many features are you tracking — should the feature matrix show all columns or let you select which features to display?
2. For the pricing comparison, should plans be grouped by competitor, or should all plans across all competitors appear in the same table sorted by price?
3. For 'changes this week' — should it highlight price changes, feature additions/removals, or both?"

**You:** "Show all features but let me hide/show them. Group plans by competitor. Show both price and feature changes."

[Claude builds the app — 12 minutes]

**You:** [runs it] "The feature matrix is hard to read — can you add color coding? Green for Y, red for N?"

**Claude:** [updates] "Done. Refresh."

**You:** "Now add the Slack integration. I want a button that sends a formatted summary of changes to a Slack webhook."

**Claude:** "To send to Slack, I need your Slack webhook URL. This is a URL that Slack gives you when you set up an 'Incoming Webhook' for a channel. Do you have one, or do you want instructions on how to create one?"

**You:** "Give me instructions."

**Claude:** [explains how to create a Slack incoming webhook in 4 steps, then adds the button to the app]

Total time: 35 minutes. The app now:
- Shows a color-coded competitive feature matrix
- Shows pricing comparison grouped by competitor
- Highlights what changed between last week and this week
- Has a "Send to Slack" button that posts a formatted summary to a channel

---

## 8. The Full PRD Pipeline

The traditional path from idea to validated concept:
- Week 1: Write spec
- Week 2–3: Design mockup
- Week 4: Engineering review
- Week 5: User testing on mockup
- Week 6: Revise based on feedback

Six weeks to learn if the concept works.

The Claude Code path:

| Phase | Time | What you do |
|-------|------|-------------|
| Write the brief | 5 minutes | Fill in the brief template from Section 4 |
| Build the prototype | 20 minutes | Run the kickoff prompt, answer clarifying questions, iterate |
| Show to 3 users | 30 minutes | Schedule 10-minute calls, share your screen, watch them use it |
| Synthesize feedback | 10 minutes | Paste notes into Claude, ask for patterns and recommendations |
| **Total** | **65 minutes** | **Validated concept with real user reactions** |

After this 65-minute cycle, you know:
- Does the concept make sense to users?
- What's confusing about the interface?
- What did users try to do that the prototype doesn't support?
- Is the core assumption valid?

Engineering only gets involved once you have answers to these questions. The spec you write after this process is sharper, more confident, and backed by evidence.

---

## 9. Sharing and Presenting Prototypes

### Option 1: Screen share for remote demos

The simplest approach. Run `streamlit run app.py`, share your screen on Zoom or Google Meet, and demo it live. You control the app, users watch.

### Option 2: ngrok for a temporary public URL

ngrok creates a temporary public URL that lets anyone access your locally running app. Useful for asynchronous reviews.

Install ngrok from ngrok.com (free account). Then:

1. Start your Streamlit app: `streamlit run app.py`
2. In a second Terminal window: `ngrok http 8501`
3. Copy the URL it shows (looks like `https://abc123.ngrok.io`)
4. Share that URL with stakeholders

The URL works as long as both Terminal windows are open. When you close them, the URL stops working. This is intentional — prototypes are temporary.

### Option 3: Streamlit Community Cloud for a permanent URL

For prototypes you want to share for more than a day:

1. Push your code to a GitHub repository (Claude can help you do this)
2. Go to streamlit.io/cloud and sign up with your GitHub account
3. Click "New app," connect your repository, and deploy
4. You get a permanent URL like `https://your-app.streamlit.app`

This is free and takes about 5 minutes. The app stays live as long as the GitHub repo exists.

### What to say when presenting

"I built a working prototype to test whether [assumption]. This isn't production-ready — it's a tool for learning. What I want to find out from showing you this is [specific question]. Here's how it works."

This framing sets the right expectation. Stakeholders evaluate the concept, not the visual polish.

---

## 10. What to Do When Claude Gets Stuck

### Common errors and what they mean

**"ModuleNotFoundError: No module named 'streamlit'"**
Streamlit isn't installed. Tell Claude: "I got this error: ModuleNotFoundError: No module named 'streamlit'. How do I fix it?"
Claude will tell you to run `pip install streamlit`.

**"FileNotFoundError: data.csv not found"**
The CSV file isn't in the right folder. Tell Claude the exact file path of your CSV and ask it to update the code to use that path.

**The app runs but shows nothing / blank page**
Tell Claude: "The app runs but the page is blank. Here is the error I see in Terminal: [paste the error text]."

**"Port 8501 is already in use"**
Another Streamlit app is already running. Tell Claude, it will tell you to run the app on a different port: `streamlit run app.py --server.port 8502`

### The "explain what you built and why" technique

When something isn't working and you can't figure out why, ask Claude:

```
Stop trying to fix it. Instead, explain in plain English:
1. What the app is currently trying to do
2. What it's probably doing wrong
3. What a simpler approach would be

Then ask me if I want to try the simpler approach.
```

This resets the conversation and often surfaces an architectural issue that was causing the problem. Nine times out of ten, the simpler approach works.

### When to ask for a completely different approach

If you've iterated on the same problem three times and it still doesn't work, say:

```
This approach isn't working. Let's start over. What are two completely
different ways to build [describe the feature]? Describe each approach
in plain English and tell me the trade-offs.
```

Then pick the simpler one.

---

## 11. From Prototype to Production

### What to hand off to engineering

A prototype handoff has four pieces:

1. **The brief** — what you were trying to build and why
2. **What you learned** — the specific finding from testing the prototype
3. **What worked** — features from the prototype that users found valuable
4. **What to change** — what the prototype got wrong that engineering should fix

Do not hand off the prototype code itself as a starting point for production. Prototype code is fast and dirty by design. Engineering will build the real thing correctly from scratch.

### How the prototype becomes a spec

The prototype is evidence. Use it to write a tighter spec:

```
I built a prototype of [feature]. Here are my findings from user testing:
[paste findings]

Based on this, write a product spec covering:
1. The problem we're solving (one paragraph)
2. The solution (what we're building and why this approach)
3. User stories (as a [user], I want to [do X] so that [Y])
4. Acceptance criteria for each user story
5. What we're explicitly NOT building in this version
```

This is a better spec than one written from first principles, because it's grounded in what actually worked.

### The CLAUDE.md handoff

When you hand off to engineering, share your prototype's CLAUDE.md. It already contains:
- What you were trying to build
- Why you made the design decisions you did
- What data the prototype uses

Add a `## Prototype Findings` section and it becomes a living brief that engineering can use as context.

---

## 12. The 30-Day Challenge

Four prototypes. One per week. Each builds on the last.

### Week 1: Data you already have

Take a CSV export you've been meaning to look at — product metrics, user survey, support tickets, anything. Build a dashboard from it.

**Your prompt:**
```
I have a CSV file called [filename.csv]. I want to understand [specific
question about the data]. Build me a Streamlit dashboard that helps me
answer that question. It should have at least one chart and one filterable table.
```

Goal: Learn the basic workflow. Get comfortable running Streamlit locally.

### Week 2: Something you do manually today

Pick one recurring task that currently lives in a spreadsheet or a document — a competitive comparison, a feature scoring matrix, a status report template. Build a tool that does it faster.

**Your prompt:**
```
I currently [describe the manual task] every [week/month]. It takes me
[X] minutes. I want to build a tool that does this faster.

Here is what I do manually: [describe the steps]
Here is the data I work with: [describe the data]

Build me a Streamlit app that automates this. Start by asking me what
the most time-consuming step is.
```

Goal: Make something useful, not just interesting. You should actually use it.

### Week 3: Something you'd show a stakeholder

Build a dashboard or prototype you'd genuinely show in a team meeting — not a proof of concept, but something that would change how people think about a decision.

**Your prompt:**
```
I have a meeting with [stakeholder] on [topic] next week. I want to show
them a working prototype instead of a slide.

The decision we're trying to make is: [decision]
The data I have: [data sources]
What would help them make the decision: [what insight you want to surface]

Build me something I can demo in a 10-minute meeting.
```

Goal: Present the prototype in a real meeting. Note what questions it prompts.

### Week 4: Something with external data or integrations

Go beyond your local files. Build something that pulls from an API, sends to Slack, reads from a Google Sheet, or processes real-time data.

**Your prompt:**
```
I want to build a tool that [describe function] using data from [source].

The integration I need: [e.g., Google Sheets URL, Slack webhook, API endpoint]

Build this step by step. Start with getting the data connection working
before building the UI. Tell me what credentials I need and how to set
them up safely.
```

Goal: Understand what's possible when prototypes connect to real systems. This is the skill that unlocks the most time savings.

---

After four weeks, you will have built four working tools, learned the core workflow, and shifted how you think about validation. The question stops being "can engineering build this?" and starts being "what did I learn from the prototype?"

That shift is the point.

---

*Sources consulted: [Dennis Yang / Chime workflow](https://www.lennysnewsletter.com/p/cursor-is-a-much-better-product-manager) | [Cat Wu / Anthropic workflow](https://claude.com/blog/product-management-on-the-ai-exponential) | [Claude Code for PMs — Builder.io](https://www.builder.io/blog/claude-code-for-product-managers) | [Claude Code for PMs — ccforpms.com](https://ccforpms.com/) | [CLAUDE.md guide](https://ccforpms.com/fundamentals/project-memory) | [Streamlit Community Cloud](https://docs.streamlit.io/deploy/streamlit-community-cloud) | [ngrok for Streamlit](https://maazirfan.medium.com/a-beginners-guide-to-deploying-streamlit-apps-with-ngrok-0b3f1c50d6af) | [Squadbase Streamlit + Claude Code workflow](https://www.squadbase.dev/en/ebooks/streamlit-bi-claude-code/04-data-and-basic-dashboards)*
