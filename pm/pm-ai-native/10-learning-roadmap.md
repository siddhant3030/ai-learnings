## 10. Learning Roadmap

### 30 Minutes — What to Learn Immediately

1. Read [Product management on the AI exponential](https://claude.com/blog/product-management-on-the-ai-exponential) — 15 minutes. Understand how PMs at Anthropic are actually using these tools.
2. Open Claude.ai and create a Project for your product. Upload 1 company brief, 1 PRD template, and 1 persona document. Write a 1-paragraph system instruction.
3. Run one prompt from the PRD generation library on a feature you're currently working on.

**Goal:** Get one real work output from Claude before doing any more reading.

---

### 2 Hours — Highest ROI

1. Complete your Claude Project setup: add your roadmap, 2-3 prior PRDs, and competitive context.
2. Take all the interview transcripts from your most recent research and run the synthesis prompt. Compare output to your own synthesis.
3. Generate PRD section drafts for a current feature using the PRD prompt.
4. Set up the competitive analysis 3-prompt chain on your top 2 competitors.

**Goal:** Replace one half-day of PM work with a 30-minute Claude session. Verify the quality yourself.

---

### 1 Week — Become Productive

**Day 1-2:** Core Claude.ai setup
- Build a full Project for your main product area
- Upload all relevant context documents (personas, roadmap, recent PRDs, research summaries)
- Create a system instruction that captures your product's voice and constraints
- Run the research synthesis prompt on existing user interview data

**Day 3:** Install Claude Code
- Follow the installation guide from [ccforpms.com](https://ccforpms.com/)
- Create your first CLAUDE.md in a product workspace folder
- Run one competitive analysis using /competitive-brief
- Generate tickets from a current PRD using the ticket generation workflow

**Day 4-5:** Build your first slash commands
- Create `/weekly-review` that reads your project notes and generates a stakeholder update
- Create `/synthesize [folder]` that processes a transcript folder
- Create `/competitive-check` that pulls a competitive update

**Day 6-7:** Install skill collections
- Install [lenny-skills](https://github.com/RefoundAI/lenny-skills)
- Try 3-5 skills that map to your current work
- Identify gaps and write one custom skill

**Goal:** Have a working PM OS — CLAUDE.md, 3-5 slash commands, a skill collection — handling at least 3 recurring weekly tasks.

---

### 30 Days — Become Highly Effective

**Week 1:** Core workflow mastery
- Nail the PRD, research synthesis, and competitive analysis workflows
- Build context library with modular files (product.md, personas.md, competitors.md, strategy.md, decisions.md)
- Identify your 10 most time-consuming weekly tasks; automate at least 5 with slash commands

**Week 2:** MCP integrations
- Connect Linear or Jira MCP: generate and update tickets from Claude
- Connect GitHub MCP: automate release notes from commits
- Connect your primary analytics tool (PostHog or Amplitude MCP)

**Week 3:** Advanced patterns
- Read [Dynamic Workflows (Product Compass)](https://www.productcompass.pm/p/claude-code-dynamic-workflows)
- Build a fan-out workflow for processing 10+ interview transcripts in parallel
- Set up automated competitive monitoring with weekly digests

**Week 4:** PM OS refinement
- Audit your system: which workflows are you actually using?
- Write 3-5 custom skills for your most specialized PM activities
- Set up hooks to automate context loading at session start
- Share your system with 1-2 colleagues; teach = deepen understanding

**Goal:** Claude handling 40-60% of your operational PM work. You are spending more time on strategy, discovery, and judgment; less time on documentation and communication formatting.

---

### 90 Days — Build Repeatable AI-Powered Workflows

**Month 2:** Organizational leverage
- Teach your PM team how to use the skill collections and slash commands you've built
- Create a shared PM OS repository for your team with standardized CLAUDE.md templates, skill collections, and context libraries
- Build a shared competitive intelligence system feeding weekly digests to the whole team

**Month 3:** Advanced agentic workflows
- Build a fully automated research pipeline (cron-scheduled paper/news ingestion → summarization → digest delivery)
- Build a launch readiness agent that monitors checklist completion across Linear, GitHub, and Notion
- Experiment with multi-agent research synthesis for large-scale customer discovery (100+ interviews)
- Develop your evals capability: learn to measure Claude output quality systematically

**Goal:** A PM OS that runs autonomously in the background, surfacing insights and maintaining documentation without manual effort. Able to run 3-5 customer discovery cycles in the time previously spent on 1.

---
