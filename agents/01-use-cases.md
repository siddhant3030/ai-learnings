# Agent Use Cases for Dalgo — What AI Agents Could Do for NGO Users

> What runtime AI agents could realistically do *inside the Dalgo product* for non-technical
> NGO staff — grounded in the actual Dalgo MCP tool surface, the real users, and what comparable
> data platforms ship today.
>
> Companion to `../research/agent-design.md` (the deep agent-design report). Read that for the
> principles cited here in short form.

---

## The one idea to hold while reading this

There are two very different things people both call "an agent":

- A **deterministic workflow** — fixed steps written in code, where an LLM only fills in the
  fuzzy parts (turning words into a query, summarizing a log). Predictable, cheap, testable.
- A **genuine agentic loop** — the model decides its own next step, calls tools in a loop until
  it thinks it's done. Flexible, but unpredictable in cost, results, and behavior.

Anthropic's own guidance is blunt: *find the simplest solution possible, and only increase
complexity when needed — which might mean not building agentic systems at all.* For Dalgo's
users — non-technical NGO staff, on slow internet, with a tiny engineering team to maintain
this, and **beneficiary PII in the warehouse** — the bias must be heavily toward the workflow
end. Cheap models, narrow tools, bounded steps, results shown before anything is saved.

So for every use case below, the most important column is not "how cool" — it's **"workflow or
agent?"** and **"read-only or mutating?"**. The sharpest finding of this whole analysis:

> **The use case users will ask for first — "just let me ask questions about my data" — is the
> one that most violates the simplicity principle.** It is open-ended generation over arbitrary
> schemas that contain PII. By Dalgo's own rule it should be the *most constrained* surface, not
> the freest. That tension drives the recommendations at the end.

---

## What exists today (and a caveat)

**Could not fully verify the backend.** I was unable to inspect `DDP_backend` directly for
existing LLM/agent code (sandbox restrictions; the repo's source did not surface in public
search). **Before building, confirm whether any Anthropic/OpenAI/LangChain integration already
exists** — if it does, several recommendations below shift from "greenfield" to "extend what's
there." Treat the build estimates as assuming a clean start.

**What we do know exists:** a **Dalgo MCP server** already exposes the platform as tools
(warehouse, pipelines, sources, dashboards/charts, reports, transforms/dbt, notifications, org,
docs). This is the natural action surface for any agent — the agent calls these tools rather than
touching the database or Prefect directly. Every use case below is expressed in terms of those
tools.

**Comparable platforms** have already converged on the same patterns Dalgo should copy:

- **Apache Superset** (Dalgo's own BI layer) ships an AI assistant *via MCP* — list datasets,
  describe a chart in words, AI builds it, **preview before saving**, and **full RBAC enforced**
  on every query. ([Superset docs](https://superset.apache.org/user-docs/using-superset/using-ai-with-superset/))
- **Preset AI Assist** is explicitly scoped to **text-to-SQL** — turn words into SQL, *show the
  SQL*, help find the right tables, save results as a dataset. It is a bounded assist, not an
  autonomous loop. ([Preset blog](https://preset.io/blog/building-preset-ai-assist-how-we-brought-text-to-sql-into-apache-superset/))
- Industry framing of text-to-SQL stresses **schema awareness** and making data accessible to
  **non-technical users** — exactly Dalgo's audience. ([Select Star](https://www.selectstar.com/resources/text-to-sql-tools))

The lesson from all three: the winning shape is **"narrow assist + show the work + human saves
it,"** not "autonomous agent does it all."

---

## Use Case 1 — Data Q&A assistant (text-to-SQL over the warehouse)

**User asks:** *"How many beneficiaries did we reach last quarter?"* / *"Which districts have no
data this month?"*

**What the agent does:** Reads the schema with `list_schemas` → `list_tables` →
`get_table_columns` (just-in-time, not pre-loaded), forms a SQL query, runs a **read-only**
fetch via `get_table_data` / `get_table_row_count`, and writes the answer in plain language.
Ideally constrained to a **curated set of "answerable" tables** (a semantic layer), not the raw
warehouse.

**Value:** Highest of any use case. This is the core promise — a program manager who can't write
SQL gets a number in seconds instead of waiting days for the data team. It's also what every
comparable platform leads with.

**Risk: HIGH despite being read-only.** Three reasons, all sharp:
1. **PII exposure.** Tables hold beneficiary data. An agent that can fetch any row from any table
   can surface names, locations, case details. Read-only does *not* mean safe.
2. **Accuracy.** Text-to-SQL is wrong often enough that a confidently-stated wrong number is
   *worse* than no answer for a non-technical user who can't sanity-check it.
3. **Indirect prompt injection** (OWASP's #1 LLM risk for 2025). Warehouse rows contain text that
   field staff or external sources typed. A malicious or accidental string in a data field can
   hijack the agent. Any agent that reads warehouse content is exposed.

**Workflow or agent?** Should be a **tightly-bounded workflow**, not a free loop. Required rails:
curated/whitelisted datasets only; **always show the generated SQL** and the row count behind the
number; **cap returned rows** (e.g. aggregates only, never raw PII dumps); read-only tools only;
enforce org-scoping and RBAC on every call; a "this might be wrong, verify before deciding" frame.

**Difficulty: HIGH.** Not the LLM call — the *guardrails*. Semantic layer, PII handling, accuracy
evals against real NGO questions, injection defense. This is the most expensive to do safely.

---

## Use Case 2 — Pipeline / sync failure explainer

**User asks:** *"Why did my sync fail last night?"*

**What the agent does:** `list_pipelines` / `get_pipeline` → `get_pipeline_run_history` →
`get_flow_run` → `get_flow_run_logs` (and `get_sync_history` for Airbyte). Reads the Prefect/dbt
logs and translates the failure into plain language: *"The connection to your Google Sheet timed
out. This usually means the sheet was renamed or access was revoked. Here's how to fix it."*

**Value:** Very high, and uniquely well-suited to an LLM. Today a failed sync means a non-technical
user files a support ticket and waits; the engineering team is tiny. An explainer deflects a large
share of those tickets. The user gets an answer at 9am instead of in two days.

**Risk: LOW.** All tools are **read-only**. Logs may contain some sensitive strings (an injection
surface — sanitize), but no data is changed and no PII tables are queried.

**Workflow or agent?** **Workflow.** Fixed retrieval path (find run → get logs → explain). A small
amount of ReAct-style looping is fine ("logs were truncated, fetch the dbt node detail too"), but
it should be bounded to a handful of read calls. Cheap model is enough.

**Difficulty: MEDIUM.** The hard part is a good prompt that maps common Airbyte/dbt/Prefect error
signatures to plain-language causes and fixes. Bounded and very testable.

---

## Use Case 3 — Docs / onboarding "how do I…?" assistant

**User asks:** *"How do I connect a new Google Sheet?"* / *"What's a transformation?"*

**What the agent does:** `search_docs` / `list_docs` / `get_doc` — retrieval-augmented answers
**grounded only in Dalgo's own product docs.** Answers, cites the doc, links to it.

**Value:** High, and broad — every user, every day, especially during onboarding. Reduces "how do
I" support load and lowers the barrier for new NGO staff. Also raises adoption of features people
don't know exist.

**Risk: LOWEST of all.** Read-only, **no PII, no warehouse access, no mutations.** Docs are
trusted internal content, so even the injection surface is minimal. The worst failure is a wrong
or made-up answer — mitigated by grounding strictly in retrieved docs and citing them.

**Workflow or agent?** Pure **RAG workflow** (retrieve → answer with citations). Not an agent at
all. Cheapest possible model.

**Difficulty: LOW.** The classic, well-understood pattern. Highest reliability-to-effort ratio on
this entire list.

---

## Use Case 4 — Dashboard / chart builder

**User asks:** *"Make me a chart of enrollments by month."*

**What the agent does:** Resolves the data (`list_tables` / `get_table_columns` / `get_data_types`),
picks chart type + metric + dimension, then `create_chart` / `render_chart` / `update_chart`, and
optionally `create_dashboard` / `update_dashboard`.

**Value:** High and very visible — turns a request into a working visual without learning the
chart builder. Superset already proves users want exactly this.

**Risk: MEDIUM — this one MUTATES.** `create_chart`, `update_chart`, `create_dashboard`,
`update_dashboard` change saved objects. The proven mitigation (Superset's pattern):
**`render_chart` to PREVIEW first, then a human clicks save.** Never auto-persist. `delete_chart` /
`delete_dashboard` must **never** be agent-autonomous.

**Workflow or agent?** **Workflow with a mandatory human-approval gate.** Generate → preview →
*user confirms* → save. The 30-second human review is the safety mechanism.

**Difficulty: MEDIUM–HIGH.** Mapping vague requests to correct chart config is fiddly; depends on
the same schema-understanding as Use Case 1. Build *after* Data Q&A's data-resolution layer exists.

---

## Use Case 5 — Data quality / monitoring (proactive)

**User wants:** To be told *before* they notice — *"this source hasn't updated in 5 days,"*
*"beneficiary count dropped 40% vs last month,"* *"last night's run failed."*

**What it does:** Scheduled checks: `get_sync_history` / `get_pipeline_run_history` for failed or
stale runs; `get_table_row_count` for volume anomalies; surfaces via `list_notifications` /
existing notification channels.

**Value:** Very high — proactive beats reactive, and it catches silent failures (a sync that
"succeeded" but pulled zero rows) that users would otherwise miss for weeks.

**Risk: LOW** *if built correctly* (read-only detection). The real risk is **cost/architecture**:
the wrong way is to run an LLM agent loop on a schedule across every org — unbounded token spend
for no added intelligence.

**Workflow or agent? Mostly NOT an agent.** Detection must be **deterministic code** — thresholds,
freshness checks, run-status queries. Cheap, predictable, reliable. The LLM appears **only** as a
thin *explanation layer*: once a deterministic check fires, one cheap LLM call turns the alert into
plain language. Don't burn an agentic loop on a cron.

**Difficulty: MEDIUM.** The detection rules and per-org thresholds are the work; the LLM part is
trivial. High value, and the deterministic core is robust.

---

## Use Case 6 — Report generation (periodic leadership summaries)

**User wants:** *"Email me a monthly summary of our key numbers."*

**What it does:** On a schedule, `create_report` makes a point-in-time dashboard snapshot
(date-filtered); the LLM writes a short plain-language narrative over the snapshot's numbers
(*"Enrollments up 12% this quarter, driven by the Bihar program; two sources had gaps in week 3"*).

**Value:** High for NGO leadership, who want a story over a dashboard. Directly serves the
"replace manual reporting" mission.

**Risk: MEDIUM.** `create_report` mutates (creates an artifact) but is non-destructive. The
narrative step reads warehouse-derived data → an **injection surface** + a **PII surface** if raw
records leak into the summary. Keep the LLM on **aggregates only**, never raw rows.

**Workflow or agent? Workflow.** Scheduled snapshot + single summarization call. No loop. The data
selection should be **fixed/configured by a human once**, not chosen freshly by the model each run.

**Difficulty: MEDIUM.** Snapshot plumbing likely exists via `create_report`; the work is a reliable
summarization prompt and keeping it factual (no invented numbers).

---

## Use Case 7 — Transform (dbt) helper

**User asks:** *"What does this model do?"* / *"Help me build a model that joins X and Y."*

**What it does:**
- *Understand (read-only):* `get_dbt_workspace`, `get_transform_graph`, `get_node_details`,
  `get_node_columns`, `get_sources_models`, `get_git_status` — explain a model, trace lineage.
- *Build (mutating, dangerous):* `create_operation` / `edit_operation`, `run_dbt`,
  `publish_changes`, `sync_sources`, plus canvas lock tools.

**Value:** Medium for *most* NGO users (only data-savvier staff touch transforms), but **high for
that smaller group**, who today depend entirely on the engineering team for every model change.

**Risk: SPLIT.** The *explain* half is **LOW** (read-only). The *build* half is **HIGH**:
`run_dbt` and `publish_changes` change what the warehouse produces and what every downstream chart
shows. A wrong model silently corrupts every dashboard built on it.

**Workflow or agent?** **Explain = simple read-only workflow, build now.** **Generate-and-write =
defer**, and when built, use the **evaluator-optimizer** pattern (generator writes SQL → checker
validates referenced columns/tables actually exist and syntax is valid → revise) plus a
**mandatory human-approval gate** before `run_dbt` / `publish_changes`. This is the "looked right
but was wrong" failure mode; never autonomous for non-technical users.

**Difficulty: LOW (explain) / VERY HIGH (build).** Reliable dbt generation is among the hardest
things on this list and the smallest audience. Build the explainer; treat generation as a later,
gated experiment.

---

## The destructive cluster — never autonomous at runtime

These tools change or destroy things and must **always sit behind an explicit human-approval gate**.
For non-technical NGO users who may not grasp the consequences, *no agent calls these on its own*:

| Tool | Why it's dangerous |
|------|--------------------|
| `delete_chart`, `delete_dashboard`, `delete_report`, `delete_pipeline`, `delete_source` | Irreversible data/artifact loss |
| `create_pipeline`, `update_pipeline`, `trigger_pipeline_run` | Kicks off real orchestration / cost |
| `run_dbt`, `publish_changes` | Changes warehouse output → silently breaks every downstream chart |
| `create_operation`, `edit_operation`, `terminate_chain` | Alters transform logic |
| `delete_source`, `add_source_to_canvas` | Changes ingestion |

Pattern for all of them: **agent proposes → shows exactly what will happen → human confirms →
then execute.** Never the reverse.

---

## Two cross-cutting safety rules (apply to every use case)

1. **Org-scoping + RBAC on every tool call.** Multi-tenant platform; an agent must never see or
   touch another org's data. Enforce server-side, in the MCP layer — not in the prompt.
2. **Treat all warehouse/source content as an injection surface.** Use Cases 1, 5, 6 read text
   that field staff and external connectors put into the data. A crafted string in a data field
   can try to hijack the agent. Strip/escape tool output, keep destructive tools out of any agent
   that reads warehouse content, and never let read-content + write-actions live in the same
   autonomous loop.

---

## Prioritization (impact vs effort vs risk)

| # | Use case | Impact | Effort | Risk | Read-only? | Build as |
|---|----------|--------|--------|------|-----------|----------|
| 3 | Docs / onboarding assistant | High | **Low** | **Lowest** | Yes | RAG workflow |
| 2 | Pipeline failure explainer | **Very High** | Medium | Low | Yes | Workflow |
| 5 | Data quality / monitoring | **Very High** | Medium | Low* | Yes (detect) | Deterministic + thin LLM |
| 1 | Data Q&A (text-to-SQL) | **Highest** | **High** | **High** | Yes (but PII) | Bounded workflow |
| 6 | Report generation | High | Medium | Medium | No (creates) | Scheduled workflow |
| 4 | Chart / dashboard builder | High | Med-High | Medium | No (mutates) | Workflow + approval gate |
| 7a | Transform *explainer* | Medium | **Low** | Low | Yes | Read-only workflow |
| 7b | Transform *builder* | Med (niche) | **Very High** | **High** | No | Defer; gated eval-optimizer |

\* monitoring risk is low only if detection is deterministic; an LLM loop on a schedule is a cost risk.

### What to build FIRST — and why not the obvious one

Build first the **read-only, low/no-PII assistants that establish the pattern and the guardrails**,
*before* the flashy high-PII one:

1. **Docs / onboarding assistant (UC3)** — cheapest, safest, broadest reach. Proves the plumbing
   (MCP wiring, model choice, citation/grounding, the answer UI) with essentially zero blast radius.
2. **Pipeline failure explainer (UC2)** — highest value-to-risk ratio. Read-only, bounded,
   directly relieves the tiny support team. This is the **strongest standalone first bet** if you
   want one win that users feel immediately.
3. **Data quality monitoring (UC5)** — deterministic core + thin LLM explanation. Proactive value,
   robust, cheap.

**Only then ship Data Q&A (UC1)** — once the guardrail muscle (org-scoping, PII handling,
injection defense, accuracy evals, "show your work" UI) is proven on the safer assistants. It's the
flagship, but shipping it first means learning all those lessons directly against beneficiary PII.

### What to avoid (for now)

- **Transform *builder* (UC7b)** — very high effort, high risk, smallest audience. Defer.
- **Any autonomous loop touching the destructive cluster.** No agent that deletes, publishes,
  runs dbt, or triggers pipelines without a human pressing the button.
- **Multi-agent anything.** Per the research (Google DeepMind, Cognition), multi-agent *degrades*
  sequential tasks by 39–70% and amplifies errors. Every use case here is single-agent or a plain
  workflow. There is no Dalgo use case that justifies multi-agent today.

---

## The dev-time vs runtime distinction (the core principle)

Runtime agents serve non-technical NGO users at scale, so they must be **simple, deterministic,
bounded, cheap-model, narrow-tool**. Rich, open-ended agentic loops belong at **dev time** (an
engineer supervising, able to catch a runaway), not in a program manager's browser where cost and
behavior must be predictable. Mapping each use case:

| Use case | Right shape | Why |
|----------|-------------|-----|
| Docs assistant (3) | **Deterministic** (RAG) | No reasoning loop needed; retrieve + answer |
| Failure explainer (2) | **Mostly deterministic** workflow | Fixed retrieval path; tiny bounded loop at most |
| Monitoring (5) | **Deterministic detection** + thin LLM | Never run an LLM loop on a schedule |
| Report gen (6) | **Deterministic** (scheduled snapshot + 1 summary call) | Data selection fixed by a human, not the model |
| Data Q&A (1) | **Bounded workflow**, *not* a free agent | Constrain hard *because* it's high-impact + PII |
| Chart builder (4) | **Workflow + human-save gate** | Generation is fine; persistence needs a human |
| Transform builder (7b) | The *only* candidate for a richer (evaluator-optimizer) loop — and **gated, deferred** | Generation genuinely needs iterate-and-check |

**Rule of thumb:** if a step has a fixed, knowable sequence (find the failed run → get its logs →
explain), it's a **workflow** — write it in code, use the LLM only for the fuzzy part. Reserve a
real agentic loop only where the next step *genuinely* can't be known in advance and a human is
positioned to catch failures. Almost nothing at Dalgo runtime qualifies.

---

## What NOT to build as an agent (a form or a doc is better)

- **Creating a connection / source.** A guided **form** with validation beats an agent. The fields
  are known; an LLM only adds failure modes and cost. (Let the *docs assistant* answer "how do I",
  but the doing is a form.)
- **Scheduling a pipeline.** Picking a cron and a connection is a **form**, not a conversation.
- **Deleting anything.** A button with a confirm dialog. Never a prompt.
- **Fixed-format reports** where the layout never changes — a **template + scheduled snapshot**,
  no LLM narrative needed. Add the LLM summary only where leadership wants a *story*, not a table.
- **Anything where being wrong is expensive and hard to detect** — irreversible writes, financial
  or compliance numbers presented as fact. Prefer a deterministic computation the team can audit.
- **A single "do everything" Dalgo agent** with all ~70 MCP tools. The research is explicit: 30+
  tools causes selection confusion and burns thousands of context tokens before the user speaks.
  Build **separate, narrow assistants** (a docs bot, a failure explainer), each with only the
  handful of tools it needs. Never one God Agent.

---

## Sources

- [Using AI with Superset — Superset docs](https://superset.apache.org/user-docs/using-superset/using-ai-with-superset/)
- [Building Preset AI Assist: Text-to-SQL in Superset — Preset](https://preset.io/blog/building-preset-ai-assist-how-we-brought-text-to-sql-into-apache-superset/)
- [Best Text-to-SQL Tools for AI Analytics — Select Star](https://www.selectstar.com/resources/text-to-sql-tools)
- [How We Built a Text-to-SQL AI Agent — Salesforce](https://www.salesforce.com/blog/text-to-sql-agent/)
- Internal: `../research/agent-design.md` (Anthropic "Building Effective AI Agents", Cognition
  "Don't Build Multi-Agents", Google DeepMind scaling study, OWASP indirect prompt injection,
  ZenML "dumb agents, smart orchestration")
