## 5. Claude Workflows Worth Learning

### 5.1 Claude Projects (Context Persistence)

**Setup:** Create a Claude Project, upload: company background, PRD template, voice samples, 3-5 prior PRDs, roadmap document, and prioritization framework. Set a detailed system instruction in "Project knowledge."

**Benefits:** All Claude sessions within the Project start with this context loaded. Output quality improves 3-5x compared to cold-start prompts because Claude can calibrate to your product's specific terminology, audience, and standards.

**Tradeoffs:** Requires upfront investment to curate the right context files. Context can drift if you forget to update product documents.

**Who should use it:** Every PM. This is the highest-leverage Claude.ai feature and requires no technical setup.

**Workflow:**
1. Create a Project named after your product area
2. Upload: 1-page company/product overview, your PRD template with examples, 2-3 recent PRDs you're proud of, current roadmap, persona definitions
3. Write a system instruction: "You are a senior PM working on [product]. Our users are [description]. Our north star metric is [metric]. Always [key constraint]. Never [anti-goal]."

---

### 5.2 Long-Context Document Analysis

**Setup:** Upload long documents (PRDs, competitive reports, transcripts, strategy decks, research papers) directly into Claude's context window.

**Benefits:** Claude can synthesize across 200K tokens — roughly a 150,000-word document — in one session. This enables analysis of entire interview transcript libraries, competitor documentation, regulatory filings, or board deck history.

**Workflow (Teresa Torres' approach):**
- Upload all 50 interview transcripts as a single batch
- Query the entire library conversationally: "What's the most common theme across all participants?"
- Export structured data: "Extract Name, Role, Main Friction, Severity for each. Output as CSV."
- Generate visual dashboard: "Create an interactive HTML dashboard from this CSV."

**Tradeoffs:** Context window limits mean very large document sets may need chunking or batching. Quality degrades slightly at the outer edge of context windows.

---

### 5.3 Artifacts Mode (Side-by-Side Editing)

**Setup:** Enable Artifacts in Claude.ai settings. Request documents by asking Claude to create an Artifact.

**Benefits:** Produced documents appear in a panel alongside the chat, allowing iterative editing without losing the conversation thread. Unlike a chat response, an Artifact can be opened, edited, and refined across multiple sessions.

**Best use cases:** PRDs, roadmap documents, strategy memos, stakeholder communications.

**Tradeoffs:** Artifacts can drift from original intent across many revisions. Periodically regenerate from scratch rather than accumulating edits.

---

### 5.4 Research Workflows (Iterative Synthesis)

**Setup:** No special setup. The discipline is sequential prompting across research stages.

**Pattern:**
1. **Broad extraction:** "Given these 20 interview transcripts, what are the 10 most common pain points?"
2. **Cluster:** "Group these 10 pain points into 3-4 themes. For each theme, list the specific pain points it covers."
3. **Prioritize:** "Rank these themes by: (a) frequency across participants; (b) emotional intensity of language; (c) presence of current workarounds."
4. **Synthesize:** "Write a 1-page research summary for a product review, using only the themes and supporting quotes from this conversation."
5. **Validate gaps:** "What important questions do these findings not answer? What user segments are underrepresented?"

**Time savings:** Research synthesis cycles from half a day to 20-30 minutes.

---

### 5.5 Parallel Multi-Stakeholder Communication

**Setup:** No special setup. Single prompt generates multiple versions.

**Pattern:**
```
<core_update>[Key facts, decisions, metrics changes, risks]</core_update>
<task>Generate 4 versions of this update: (1) Executive summary (200 words, leads with status and key decisions needed); (2) Engineering brief (300 words, leads with blockers and scope changes); (3) Sales enablement (200 words, leads with customer-facing implications); (4) Customer release note (100 words, user-benefit language, non-technical).</task>
```

**Why it matters:** Four tailored communications from one draft. Eliminates the coordination overhead of maintaining four separate documents.

---
