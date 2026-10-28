# OpenWiki: How People Actually Use It, and What It Is Not For

> Deep-research report, 2026-10-28. Three parallel research agents (product internals, real-world usage,
> alternatives landscape) → ~60 primary sources: the `langchain-ai/openwiki` README and installed 0.7.2
> source, GitHub issues and commits of adopting repos, LangChain posts, HN, tool docs. Plus our own
> first-hand evidence: Sangam's `openwiki/` (in use since 2026-10-14) and four failed attempts to use
> OpenWiki as cross-session memory (`~/claude-brain`, 2026-10-27). Vendor numbers are flagged as such;
> unverifiable items are in §9. Audience: me, deciding how to give every coding agent in `~/Documents`
> (Claude Code now, Codex/Cursor later) a shared memory.
> Companions: [`memory-systems.md`](./memory-systems.md), [`context-engineering.md`](./context-engineering.md).

---

## Table of Contents

1. [The one-paragraph thesis](#1-the-one-paragraph-thesis)
2. [What OpenWiki is](#2-what-openwiki-is)
3. [How people actually use it](#3-how-people-actually-use-it)
4. [Reported results: vendor vs independent](#4-reported-results-vendor-vs-independent)
5. [Reported pain](#5-reported-pain)
6. [Our own evidence](#6-our-own-evidence)
7. [The alternatives landscape](#7-the-alternatives-landscape)
8. [Cross-agent plumbing: what every agent actually reads](#8-cross-agent-plumbing-what-every-agent-actually-reads)
9. [What we could not verify](#9-what-we-could-not-verify)
10. [What this means for us](#10-what-this-means-for-us)

---

## 1. The one-paragraph thesis

OpenWiki is a **codebase map for agents**: a generated, source-pinned markdown wiki that any coding agent
reads through AGENTS.md or MCP, so it can skip re-reading the code. That job it does tolerably — Sangam's
wiki took 22 minutes, needed 10 claims corrected, and now refreshes for free when nothing changed. It is
**heavily starred but thinly used** (17.5k stars; ~380 public repos commit an `openwiki/` folder, most with
≤1 star, about a third never refreshed), its only performance numbers are LangChain's own (one repo, n=2 on
one benchmark), and its tracker is full of runaway-cost, planning-hang and hallucinated-page reports.
Nobody — us included — has made it work as **memory of past sessions**: personal mode can't read local
files, and code mode stalls planning on conversation logs. The portable layer for that is not a wiki
generator but **plain markdown + AGENTS.md + an MCP search** over transcripts.

---

## 2. What OpenWiki is

Built by LangChain on its DeepAgents JS harness; MIT; repo created 2026-06-22, first release 2026-07-05,
29 releases to v0.7.2 (2026-10-09), nothing released since (only dependency bumps). 17,534 stars, 42k npm
downloads/week (`gh repo view`, api.npmjs.org, 2026-10-28).

| | `code` mode (default) | `personal` mode ("OpenWiki Brains", launched 2026-07-10) |
|---|---|---|
| For | documenting the current repo | your connected sources: Gmail, Notion, Slack, X, HN, web search, git repos, custom MCP |
| Writes to | `<repo>/openwiki/` | `~/.openwiki/wiki` |
| Agent can read | the repo (`.openwikiignore` is the boundary); shell limited to `pwd`, `git rev-parse HEAD` | **only `~/.openwiki/wiki` + connector output**; shell refused ("reads untrusted connector content") |
| Claims | every claim pinned to `repo://path#Lx-Ly`, re-checked on update | none — claims apply to repository evidence only |
| Served to coding agents | yes (MCP + AGENTS.md block) | **no** — "coding-agent integrations support code wikis only" |

**Mechanics** (README, `dist/agent/repository-prompts.js`):
- `--init`: a **planner** explores the repo and calls `submit_plan` once (page paths are final), then page
  workers write pages one by one, each durable on its own; `quickstart.md` last. `INSTRUCTIONS.md` is the
  hand-written brief and is never overwritten.
- `--update`: only pages whose Claims went stale. A clean tree makes **no model call**.
- `.run.json` checkpoints a run; rerunning resumes it.
- 14 providers (default OpenAI `gpt-5.6-terra`); `OPENWIKI_PAGE_CONCURRENCY` 1–8; output cap 16,384 tokens.
- No built-in cost or time guidance, no spend cap.

**Distribution**:
- MCP retrieval tools `openwiki_search`/`openwiki_read` (local, read-only, no model).
- MCP generation tools (`openwiki_begin → submit_plan → next_page → submit_page → finish`) let a
  **host agent write the pages with its own model session** — i.e. Claude Code on a subscription
  instead of a metered API key.
- `openwiki integrations install` for Codex, Claude, Cursor, opencode, Copilot, Kiro, Antigravity and others.
- Maintains an `<!-- OPENWIKI:START -->` block in AGENTS.md (and CLAUDE.md if present).
- `openwiki link` groups repo wikis into a searchable workspace.
- `init` scaffolds `.github/workflows/openwiki-update.yml` (daily cron, opens a PR).

---

## 3. How people actually use it

GitHub code search for `path:openwiki filename:quickstart.md` → **~380 repos** (lower bound; default
branches only). Of 320 looked up: **218 have ≤1 star**, 12 have ≥100, 5 have ≥1,000 — and four of those
five are LangChain's own (langchain, deepagents, openwiki, open-swe).

Freshness, every 6th non-LangChain adopter (n=53): **21** touched `openwiki/` in the last 30 days,
**32** did not; **17** have exactly one commit. Example: nearai/ironclaw (12.7k★) — 4 commits, last
2026-08-05.

| Pattern | Evidence | How common |
|---|---|---|
| **Onboard coding agents to one repo, refreshed by the scheduled CI workflow** | langchain-ai/langchain (16 commits, CI-driven); issues #368, #389, #826 | Dominant |
| **Run locally inside Claude Code instead of CI, to avoid a metered key** | xgourmandin/jig: "A CI job needs a metered Anthropic API key, which is too expensive" | Emerging |
| Use from Codex / Cursor / Antigravity over MCP | #826, #1000; community port SoulKyu/openwiki-cc | Some |
| Multi-repo workspaces (`openwiki link`) | requested in #807; almost no usage found outside the repo | Thin |
| Personal "second brain" (Gmail/Notion/Slack) | issues #380, #591, #630 show attempts; **no first-hand write-up found** | Thin |
| Non-code repos (docs, research) | langchain-ai/docs, WenyuChiou/research-hub | Anecdotal |
| **Memory of past agent sessions** | **none found.** PR #823 "memory retrieval layer" closed unmerged; a LangSmith-traces connector exists, no reported use | None |

Companies or teams publicly using it outside LangChain: none found. The Nebius cookbook is a vendor
tutorial, not a deployment.

---

## 4. Reported results: vendor vs independent

| Claim | Source | Caveat |
|---|---|---|
| v0.3: success 35% → 45%, −14% tokens, −26% tool calls | Brace Sproul (LangChain), X | **n=2** |
| WikiBench: wiki + source beat source alone and cost less; **wiki alone did "much worse"** | LangChain blog | **one repository**; cost $0.44–$9.18 by model |
| "30–40% fewer tokens" on DeepSWE | ZenML write-up | retells LangChain's claim |
| Independent measurement | — | **none found** |

LangChain's own conclusion is the useful one: a wiki works "best as a guide rather than a replacement" for
the source. HN (96 points) was mostly skeptical: *"That's just cruft unless there is a benchmark
demonstrating it's actually better than just asking an agent to write docs"*; *"This could have been a
SKILL."*

---

## 5. Reported pain

| Failure | Issues / commits | Example |
|---|---|---|
| **Runaway cost, no cap** | #51, #934 | #934: ~10M tokens with the process idle; 2.6M tokens and 55 file reads for the first page. #51: 12× variance on the same repo, one run $10.71 |
| **Planning hangs or dies** | #1046, #1067, #1074, #890, #256, #584 | #1046: Anthropic planner's `submit_plan` hits the 16,384-token output cap. #1067: a 600 s timeout kills long planning streams |
| **Hallucinated pages** | #368, #389, #820, #310 | #389: every run rewrote the repo's CI docs with OpenWiki's own OpenRouter/GLM defaults (reproduced by a second user) |
| **Overwrites your files** | #573, #316, #725, #556 | LangChain's own docs PR #6171: "Remove CLAUDE.md again and stop OpenWiki from restoring it" |
| **Stale claims** | #899 | claims keep old line numbers after code moves |
| **Brief ignored** | #1004, #822 | page workers never see `INSTRUCTIONS.md` or the run's request |
| **Doesn't scale** | #162, #648, #826, #919 | monorepos; context overflow; `openwiki_begin` returns >100k chars unpaginated |

**People removing it** (commit messages):
- 4rankng/vfic-chatbot: "never consumed, needed a paid provider key to regenerate, and had drifted"
- m16khb-org/issueops: pages described removed code
- jaronlu/SecRAG: the daily CI "kept failing" and reverted hand edits
- 4rankng/silversea: replaced with repowise

---

## 6. Our own evidence

### Sangam: code mode, where it fits (PR #201, 2026-10-14)

- `make wiki` (Sonnet 5.5, concurrency 2): **22 min → 16 pages**, now 25 pages after 14 commits. OpenWiki
  prints no token count, so the cost is unknown.
- A spot check found **10 wrong claims**, among them the catalog's table count, the Import subagents and the
  funder write order. They were fixed with `make wiki-fix MSG=…`, our own target (a chat turn), not an
  OpenWiki command.
- `make wiki-update` on an unchanged tree: **~18 s, no model call**. An `INSTRUCTIONS.md` edit alone
  rewrites nothing (matches #822).
- **Test with a fresh agent:** "which tools does Import's supervisor have, and which pause?" The agent
  answered correctly through `CLAUDE.md` → the nested `CLAUDE.md` → the wiki page, without opening source.
- **Drift we caused ourselves:** the AGENTS.md block says "the scheduled OpenWiki GitHub Actions workflow
  refreshes the repository wiki". That workflow was never built (M3 is still open), so updates happen only
  when a PR author runs `make wiki-update`.

### `~/claude-brain`: OpenWiki as session memory (2026-10-27), four failures

The input was 118 SessionEnd digests (~4 MB of conversation text) plus 55 memory notes from 10 projects.

| # | Mode / model | What happened | Root cause |
|---|---|---|---|
| 1 | personal / Haiku 4.5 | 19 pages with **invented reasons**, wrong counts ("56 Sangam notes", real number 36), empty decisions/gotchas | personal mode can't read the files, so it wrote from a commit manifest |
| 2 | personal / Sonnet | called an unconfigured custom-mcp tool and stopped | same sandbox |
| 3 | personal / Sonnet | confirmed the agent can only see `~/.openwiki/wiki` and connector manifests | by design (§2) |
| 4 | code / Sonnet 5 | **90 min stuck in planning, 0 pages**, killed | the planner tries to take in the whole input; 4 MB of chat is ~1M+ tokens (matches #934, #1046) |

**Lesson:** this was a tool mismatch, not a setup mistake. Code mode needs *code* to pin claims to and a
size the planner can hold. Personal mode can't read local files. Session memory has neither property.

---

## 7. The alternatives landscape

### (a) Codebase knowledge

| Tool | Runs | Agents read it via | Fit |
|---|---|---|---|
| **Hand-written AGENTS.md / CLAUDE.md hierarchy** | in repo | loaded automatically | the only format every agent reads with zero setup |
| OpenWiki code mode | local, your key (or host-driven via MCP) | MCP + AGENTS.md | worth it for big, fast-changing repos like Sangam |
| Devin DeepWiki + MCP | cloud | MCP `ask_question` | free, no auth, **public repos only**: good for dependencies |
| Google Code Wiki | cloud | web UI, no MCP | public only; private-repo extension waitlisted |
| deepwiki-open (18k★, MIT) | Docker | web UI | self-hosted, no MCP confirmed |
| Aider repo map | local | Aider only | not portable |
| Sourcegraph Cody | cloud | IDE | Free and Pro ended 2025-07-23 |

### (b) Memory across sessions and agents

| Tool | Local? | Captures | Agents | Cost / caveat |
|---|---|---|---|---|
| **Claude auto-memory + CLAUDE.md** | ✅ | Claude writes typed notes | Claude only | free; per repo by default, but `autoMemoryDirectory` can point every repo at **one shared folder** |
| **obra/episodic-memory** (493★, MIT) | ✅ SQLite + local embeddings | archives `~/.claude/projects` **and `~/.codex/sessions`**, opencode | Claude, Codex, Cursor, opencode (MCP) | free with `EPISODIC_MEMORY_SKIP_SUMMARIES=1`; one maintainer |
| Hindsight coding-agents (54.5k★, MIT) | daemon or cloud | hooks send the transcript, an LLM extracts facts | 24+ agents | LLM call per save; Rust on macOS; self-reported benchmarks |
| claude-mem (103.8k★) | ✅ data | 6 hooks + AI compression | Claude, Codex, Cursor | summarising runs on a **paid service** after a 30-day trial |
| Basic Memory (4.4k★, **AGPL**) | ✅ markdown | the agent writes notes over MCP | Claude, Codex, Cursor | free locally |
| Karpathy "LLM wiki" skills | ✅ markdown | the agent compiles raw sources into a wiki | any skill-capable agent | your own model calls; same compile step that failed us |
| Codex memories | ✅ | background summaries | Codex only | off by default |
| Mem0 / OpenMemory MCP | — | — | — | OpenMemory **removed** from mem0 repo 2026-07-29 |
| Zep/Graphiti, Letta | — | graph / own harness | — | graph DB + several LLM calls per episode / replaces your agent |

Star counts for claude-mem and Hindsight exceed Aider's despite both being younger, and Hindsight has
only 109 watchers. Treat that popularity as unproven.

---

## 8. Cross-agent plumbing: what every agent actually reads

| Agent | Global instructions | Per repo |
|---|---|---|
| Claude Code | `~/.claude/CLAUDE.md` (+ `@imports`, `~/.claude/rules/`) | CLAUDE.md; reads AGENTS.md natively only when **no** CLAUDE.md exists, so use `@AGENTS.md` |
| Codex | `~/.codex/AGENTS.md` (32 KiB merged cap) | AGENTS.md |
| Gemini CLI | `~/.gemini/GEMINI.md` (`contextFileName` can be AGENTS.md) | GEMINI.md / AGENTS.md |
| Cursor | User Rules (stored in Cursor's settings, not a file) | AGENTS.md, `.cursor/rules` |

AGENTS.md is read by 23 tools (agents.md). **MCP is the only runtime transport every memory tool shares.**
Instruction files load at startup, so the global file has to say *where* memory lives and *when* to search
it.

---

## 9. What we could not verify

- Any independent token, cost or success measurement of OpenWiki. The X posts returned HTTP 402; their
  numbers come from search snippets.
- Real use of `openwiki link` workspaces, or a first-hand Personal Brain write-up.
- Whether chat-mode corrections (our `wiki-fix`) reconcile Claims exactly like the page-job runner.
- What our four brain runs cost: no token output, and the console wasn't checked.
- Whether Cursor memories still work (staff and blogs disagree); deepwiki-open private-repo/MCP support;
  where OpenMemory went.
- Real per-session costs for claude-mem, Hindsight and Graphiti; Hindsight's "independently verified"
  benchmark.
- Adoption counts are approximate. The freshness sample was systematic over a list sorted by stars, not
  random.

---

## 10. What this means for us

Three separate needs. OpenWiki fits only the first.

| Need | Example question | Use |
|---|---|---|
| **What the code does** | "Which tools does Import's supervisor have?" | OpenWiki code mode **per repo, where it pays**, i.e. Sangam. Elsewhere, a short AGENTS.md hierarchy first |
| **What we did and decided** | "Why did we drop the funder-assessor subagent?" | **search over transcripts** (episodic-memory), with the SessionEnd digests in `~/claude-brain/sessions` as the lasting, greppable record |
| **Who I am / how I work** | "One line per item; always give examples" | **one markdown file**, `~/claude-brain/AGENTS.md`, imported by `~/.claude/CLAUDE.md` and symlinked to `~/.codex/AGENTS.md` / `~/.gemini/GEMINI.md`; one line in Cursor User Rules |

Decisions to review:

1. **Drop OpenWiki from `~/claude-brain`.** Remove the `openwiki code --update` step from `brain-sync.sh`.
   Keep the digest hook: Claude Code deletes transcripts after 30 days (`cleanupPeriodDays`), and the
   digests are the only lasting record.
2. **Install episodic-memory** with summaries off. It is free, local, and already reads Codex sessions.
3. **Promote general preferences to `~/claude-brain/AGENTS.md`**, the one file every agent reads, out
   of the 55 per-project notes. Alternative for Claude alone: `autoMemoryDirectory` pointing at a shared
   folder.
4. **Sangam:** either build the scheduled `openwiki-update` workflow (M3) or fix the AGENTS.md line that
   says one exists. Pin the version (already done: `openwiki@0.7.2`) and keep `wiki-fix` for corrections.
   Consider the host-driven MCP flow, which uses Claude Code's session instead of a metered key, as jig did.
5. **Other repos (dalgo-core, glific, kaapi-backend):** start with a hand-written AGENTS.md. Run
   `openwiki code --init` only when a repo is large enough that agents keep re-reading it, and budget
   for a spot check like Sangam's (10 wrong claims in 16 pages).
6. **Don't** adopt claude-mem (paid summariser), Mem0/OpenMemory (removed) or Letta (replaces the agent).
   Revisit Hindsight only if automatic capture across many agents becomes the need.

---

### Sources

- OpenWiki: [repo + README](https://github.com/langchain-ai/openwiki) · [docs overview](https://docs.langchain.com/oss/openwiki/overview) · [Brains launch](https://www.langchain.com/blog/introducing-openwiki-brains-general-purpose-wiki-memory-for-agents) · [WikiBench](https://www.langchain.com/blog/evaluating-openwiki-with-wikibench) · [ZenML write-up](https://www.zenml.io/llmops-database/agent-optimized-documentation-generation-with-openwiki-cli) · [HN thread](https://news.ycombinator.com/item?id=48752949) · [reworked.co](https://reworked.co/digital-workplace/langchain-launches-openwiki-brains-for-agent-memory/)
- Issues: [#44](https://github.com/langchain-ai/openwiki/issues/44) [#51](https://github.com/langchain-ai/openwiki/issues/51) [#368](https://github.com/langchain-ai/openwiki/issues/368) [#389](https://github.com/langchain-ai/openwiki/issues/389) [#573](https://github.com/langchain-ai/openwiki/issues/573) [#776](https://github.com/langchain-ai/openwiki/issues/776) [#826](https://github.com/langchain-ai/openwiki/issues/826) [#899](https://github.com/langchain-ai/openwiki/issues/899) [#934](https://github.com/langchain-ai/openwiki/issues/934) [#1004](https://github.com/langchain-ai/openwiki/issues/1004) [#1046](https://github.com/langchain-ai/openwiki/issues/1046) [#1067](https://github.com/langchain-ai/openwiki/issues/1067) · PR [#823](https://github.com/langchain-ai/openwiki/pull/823)
- Adopters and removals: [nearai/ironclaw](https://github.com/nearai/ironclaw) · [xgourmandin/jig](https://github.com/xgourmandin/jig/commit/651faa0a0bf5a1c52519fca07b8d30cb948bc0a0) · [4rankng/vfic-chatbot](https://github.com/4rankng/vfic-chatbot/commit/00d25746e52de9c5f9a5ba3c321151b4b6bc9482) · [m16khb-org/issueops](https://github.com/m16khb-org/issueops/commit/b98bc54b19cf7721c3b70f42fd58cf955c2291a1) · [jaronlu/SecRAG](https://github.com/jaronlu/SecRAG/commit/85e228b0aeaab60e808c6ef5f1d7222d2a86b7c6) · [4rankng/silversea](https://github.com/4rankng/silversea/commit/92665fa58004de50f70fd76629073995552ecbc1)
- Alternatives: [episodic-memory](https://github.com/obra/episodic-memory) · [claude-mem](https://github.com/thedotmack/claude-mem) · [Hindsight coding agents](https://hindsight.vectorize.io/sdks/integrations/coding-agents) · [Basic Memory](https://github.com/basicmachines-co/basic-memory) · [DeepWiki MCP](https://docs.devin.ai/work-with-devin/deepwiki-mcp) · [Google Code Wiki](https://developers.googleblog.com/introducing-code-wiki-accelerating-your-code-understanding/) · [deepwiki-open](https://github.com/AsyncFuncAI/deepwiki-open) · [Aider repo map](https://aider.chat/docs/repomap.html) · [karpathy-llm-wiki](https://github.com/Astro-Han/karpathy-llm-wiki) · [Graphiti MCP](https://github.com/getzep/graphiti/tree/main/mcp_server) · [Letta Code](https://docs.letta.com/letta-code)
- Agent plumbing: [Claude Code memory](https://code.claude.com/docs/en/memory) · [Codex AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md) · [Codex memories](https://learn.chatgpt.com/docs/customization/memories) · [Gemini CLI](https://geminicli.com/docs/cli/gemini-md) · [Cursor rules](https://cursor.com/docs/context/rules) · [agents.md](https://agents.md/)
- Ours: Sangam [PR #201](https://github.com/project-tech4dev/sangam/pull/201), `Makefile` §Wiki, `openwiki/`; `~/claude-brain` run logs, 2026-10-27
