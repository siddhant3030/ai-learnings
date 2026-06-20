# Where People Build MCP Workflows: Agent Frameworks, IDEs & Automation Platforms

*A builder's guide to the tools that **consume** MCP — how the experience differs, what you'd build in each, and how to choose.*

Last updated: June 2026. MCP moved fast through 2025; dates are pinned to source announcements.

---

## TL;DR

The Model Context Protocol (MCP) started as Anthropic's way to connect Claude to tools
([Anthropic, Nov 2024](https://www.anthropic.com/news/model-context-protocol)). By mid-2025
it had become the de-facto **universal connector layer**: OpenAI, Microsoft, Google, and every
major IDE and automation platform now consume MCP. The interesting question is no longer
"does X support MCP" (almost everything does) but **where you should build your workflow** —
which depends on whether you want no-code or code, local or remote, hosted or self-hosted.

This guide covers eight families of MCP **consumers** (clients/hosts), plus the cross-cutting
splits: no-code vs code, local vs remote, and the universal-connector trend.

> **Terminology.** In MCP, a **server** exposes tools/data; a **client** (inside a "host"
> app) consumes them. Many platforms below can play **both** roles — n8n, Zapier, Make, and
> Copilot Studio can each *consume* MCP servers and *be* an MCP server. Watch the direction
> when reading each section.

---

## Comparison Table

| Platform | MCP role | No-code / Code | Local + Remote MCP | Hosted vs Self-host | Best for |
|---|---|---|---|---|---|
| **Claude Code / Desktop / claude.ai** | Client (host) | Mixed (chat + CLI) | Desktop/Code: local stdio + remote; claude.ai: remote-only | Anthropic-hosted; Desktop local | PMs/devs wiring tools into a reasoning loop |
| **Cursor / Windsurf** | Client (host) | Code (IDE) | Local stdio + remote | App on your machine | AI-assisted coding with project context |
| **VS Code + Copilot** | Client (host) | Code (IDE) | Local stdio + remote | App on your machine | Agentic coding inside a mainstream editor |
| **n8n** | **Both** (client node + server trigger) | No-code/low-code | Self-host can run local; cloud = remote | Cloud or self-host | Visual automations that AI can trigger, or that call MCP |
| **Zapier** | **Both** (server + client beta) | No-code | Remote-only (cloud) | Hosted | Giving an AI 9,000+ app actions with zero setup |
| **Make** | **Both** (server + client) | No-code (visual) | Remote-only (cloud) | Hosted | Visual scenarios as AI-callable tools |
| **LangChain / LangGraph** | Client (library) | **Code** (Python/JS) | Both (stdio, SSE, HTTP) | You host | Programmatic multi-MCP agents, full control |
| **OpenAI Agents SDK / ChatGPT Apps** | Client (host) | Code (SDK) + no-code (ChatGPT) | SDK: both; ChatGPT: remote-only | OpenAI-hosted / you host server | Reaching ChatGPT's user base; OpenAI-native agents |
| **Copilot Studio** | **Both** | Low-code (maker) | Remote-only (enterprise cloud) | Microsoft-hosted | Enterprise agents with governance/DLP |
| **Goose (Block)** | Client (host) | Code/CLI + Desktop | Local stdio + remote | Local | Local-first open-source agent, MCP reference impl |

---

## 1. Claude Code & Claude Desktop / claude.ai — the Anthropic ecosystem

MCP originated here, so the Anthropic surfaces are the most mature MCP **hosts**. All three —
Claude Desktop, Claude Code (CLI), and claude.ai in the browser — consume MCP servers, exposed
to end users as **"connectors."**

- **Connectors** are one-click integrations built on MCP that let Claude search, analyze, and
  take action inside your tools. They launched **July 14, 2025**, and by February 2026 the
  Connectors Directory listed 50+ curated integrations across productivity, communication,
  design, engineering, finance, and healthcare.
  ([Claude Help Center](https://support.claude.com/en/articles/11176164-use-connectors-to-extend-claude-s-capabilities))
- **Custom connectors** let you point Claude at *your own* remote MCP server URL (with optional
  OAuth client ID/secret), available on Free, Pro, Max, Team, and Enterprise plans.
  ([Building custom connectors](https://claude.com/docs/connectors/building),
  [Get started with remote MCP](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp))
- **Local vs remote:** Claude **Desktop** and **Claude Code** run local `stdio` MCP servers
  (a process on your machine) *and* remote servers. **claude.ai** in the browser is remote-only.
- **Projects + MCP:** Claude Projects bundle instructions and knowledge; connectors attached to
  the workspace give a Project live access to tools (e.g., a Project for "investor updates" with
  a Google Drive + analytics connector). The **API MCP connector** lets developers attach remote
  MCP servers to Messages API calls programmatically.
  ([MCP connector API](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector))

**What you'd build here.** A PM connects Linear + Notion + a custom analytics MCP server, then
asks Claude to "summarize this sprint's shipped work and draft the changelog." A dev uses Claude
Code with a local Postgres MCP server to query and refactor against a live schema.

**Strengths:** deepest MCP fidelity (Anthropic ships the spec), strong reasoning loop, one-click
UX for non-devs. **Limits:** connector ecosystem is curated/Anthropic-centric; remote OAuth
connectors have had stability hiccups (a Dec 18, 2025 Desktop update broke OAuth custom
connectors —
[issue](https://github.com/anthropics/claude-ai-mcp/issues/5)).

---

## 2. Cursor / Windsurf / other AI IDEs — MCP for coding

AI-native IDEs were among the **first** non-Anthropic adopters of MCP, because coding agents
constantly need external context: databases, docs, issue trackers, browser automation.

- **Cursor** supports MCP servers configured per-project or globally; its agent can call MCP
  tools mid-edit (read a DB, pull GitHub issues, run a browser).
- **Windsurf** added MCP to its **Cascade** agent in 2025. Per Composio's comparison, Windsurf
  ships a set of built-in MCP integrations and uses a slightly different config field
  (`serverUrl`) than Cursor — treat exact counts/paths as version-specific, not fixed facts.
  ([Composio: Cursor vs Windsurf](https://composio.dev/blog/cursor-vs-windsurf),
  [Windsurf MCP setup](https://fast.io/resources/windsurf-mcp-setup-guide/))
- **Local vs remote:** both run **local `stdio`** servers (the default for dev tools — Postgres,
  filesystem, Git) and **remote** HTTP/SSE servers.

**What you'd build here.** Wire a Postgres MCP server + a Playwright/browser MCP server into the
IDE agent so it can "check the failing E2E test, query the orders table to see what data the page
renders, and fix the component." The agent reads your repo *and* live systems in one loop.

**Strengths:** project context + tools in one place; fast inner-loop. **Limits:** scoped to a
coding session (not a standing automation); each IDE has its own config format, so MCP setups
aren't perfectly portable.
([Boston Institute of Analytics overview](https://bostoninstituteofanalytics.org/blog/from-cursor-to-windsurf-how-mcp-integration-is-revolutionizing-ai-coding-tools/),
[multi-IDE MCP setup guide](https://chatforest.com/guides/mcp-setup-ai-coding-tools/))

---

## 3. n8n — the big one for no-code/low-code automation

n8n is the standout for builders who want **visual** workflows. It's distinctive because it
plays **both MCP roles** with dedicated nodes.

- **MCP Server Trigger node** — turns n8n *into* an MCP server. It exposes a URL endpoint;
  when an MCP client (Claude Desktop, ChatGPT, Cursor) calls it, the node parses the tool name +
  inputs and runs your workflow. Your tested workflows become AI-callable tools.
  ([n8n docs: MCP Server Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/))
- **MCP Client node** — lets an n8n workflow *consume* external MCP servers: discover their
  tools and call them as ordinary workflow steps.
  ([n8n MCP tools reference](https://docs.n8n.io/advanced-ai/mcp/mcp_tools_reference/))
- **Local vs remote:** **self-hosted** n8n can run alongside local MCP servers; n8n Cloud is
  remote. n8n has also announced an **instance-level MCP server** (in preview) that lets external
  clients build/test/publish workflows directly inside an instance — treat this as
  recently-announced rather than long-GA.

**What you'd build here (concrete).** A support team builds an n8n workflow that checks a
customer account, looks up billing status, and posts a summary to Slack. They expose it via the
**Server Trigger**, so when a support agent asks an AI assistant *"check this customer and
summarize the account status,"* the assistant invokes the **pre-built, tested** workflow instead
of touching production systems directly.
([UI Bakery n8n MCP guide](https://uibakery.io/blog/n8n-mcp-guide))
Other common builds: AI-powered ticket triage across Slack/Linear, and a "send email" or "create
calendar event" tool exposed to an agent.
([Leanware guide](https://www.leanware.co/insights/n8n-nodes-mcp-guide),
[Hostinger tutorial](https://www.hostinger.com/tutorials/how-to-use-n8n-with-mcp))
There's even a community template to **build your own n8n-workflows MCP server**.
([n8n template #3770](https://n8n.io/workflows/3770-build-your-own-n8n-workflows-mcp-server/))

**Strengths:** visual, governable, self-hostable; bidirectional MCP; huge node library to wrap.
**Limits:** AI-generated/triggered workflows still need human review for credentials, error
handling, and security; verbose tool schemas add token cost; self-hosting means you own patching
and access control.

---

## 4. Zapier / Make — MCP in mainstream automation

These bring MCP to the **largest non-technical audiences**, both as **remote, hosted** services.

### Zapier MCP
Zapier exposes its automation catalog as a remote MCP server. Connect your AI client (Claude,
ChatGPT, Cursor), let Zapier import your existing app connections, and the AI can act across
**9,000+ apps / 30,000+ actions** in plain language. Each enabled action becomes a named tool —
e.g., enabling Gmail's send action gives the AI a `gmail_send_email` tool.
**Setup takes ~5 minutes, no terminal or config files**, and MCP is bundled into all plans;
each tool call uses **2 tasks** from your quota. Zapier also has an **MCP Client** (beta) so Zaps
can call *external* MCP servers.
([Zapier MCP](https://zapier.com/mcp),
[Connect remote MCP servers](https://help.zapier.com/hc/en-us/articles/38777069364109-Connect-remote-MCP-servers-to-Zapier-using-MCP-Client),
[zapier/zapier-mcp on GitHub](https://github.com/zapier/zapier-mcp))

**What you'd build here.** Give Claude a Zapier MCP connector scoped to Gmail + Google Sheets +
Slack, then: *"When I forward an invoice, log it to the Finance sheet and ping #accounting."* No
flow-builder — the AI orchestrates the actions.

### Make MCP
Make mirrors the pattern with a visual twist. The **Make MCP Server** turns your visual scenarios
into tools that AI agents (Claude, GPT, Cursor) can run with precise inputs/outputs — instant
access to your workflows across **3,000+ apps / 30,000+ actions**, no local setup. The **Make MCP
Client** lets scenarios securely call external MCP servers (e.g., Monday, GitHub). Make also lets
you attach MCP tools directly to its native **AI Agents**.
([Make MCP](https://www.make.com/en/mcp),
[Make MCP Client](https://www.make.com/en/integrations/mcp-client),
[MCP tools for AI agents](https://help.make.com/mcp-tools-for-ai-agents))

**Strengths:** unmatched breadth of pre-built app integrations, zero infra, accessible to
non-devs. **Limits:** remote/cloud-only (no local servers, data leaves your machine);
per-call task billing; you trust the vendor's auth model.

---

## 5. LangChain / LangGraph — programmatic multi-MCP agents

For builders who want **code and full control**, LangChain's `langchain-mcp-adapters`
(released **March 1, 2025**) is the bridge.
([LangChain changelog](https://changelog.langchain.com/announcements/mcp-adapters-for-langchain-and-langgraph),
[GitHub](https://github.com/langchain-ai/langchain-mcp-adapters),
[PyPI](https://pypi.org/project/langchain-mcp-adapters/))

- It **converts MCP tools into LangChain/LangGraph-compatible tools**, so the hundreds of
  published MCP servers drop straight into an agent.
- **`MultiServerMCPClient`** connects to **multiple MCP servers at once**, each with its own
  transport (`stdio`, SSE, Streamable HTTP), and presents a unified tool interface — no manual
  client management.
- Pair it with LangGraph's `create_react_agent` to build a ReAct agent that pulls tools from
  several servers and chains entire tasks autonomously.
  ([LangChain MCP docs](https://docs.langchain.com/oss/python/langchain/mcp))

**What you'd build here (concrete).** A LangGraph ReAct agent connected to three MCP servers —
a GitHub server, a Postgres server, and a Slack server — that triages an incident: read the error
from GitHub, query the affected rows in Postgres, post a root-cause summary to Slack, all in one
autonomous chain.
([Fetch.ai multi-server example](https://innovationlab.fetch.ai/resources/docs/next/examples/mcp-integration/multi-server-agent-example),
[AgentLoop tutorial](https://medium.com/agentloop/building-autonomous-task-chains-with-langgraph-react-agents-and-multi-server-mcp-clients-90667eb30ff6))

**Strengths:** total control over orchestration, memory, retries, human-in-the-loop; both local
and remote transports; mix MCP tools with native LangChain tools. **Limits:** it's a library, not
an app — you write, host, and operate everything. For non-devs this is the wrong tool.

---

## 6. OpenAI ecosystem — Agents SDK & ChatGPT Apps

OpenAI **officially adopted MCP in March 2025** across the Agents SDK, Responses API, and the
ChatGPT desktop app — a watershed for MCP as a cross-vendor standard.
([OpenAI for Developers 2025](https://developers.openai.com/blog/openai-for-developers-2025))

- **Agents SDK** natively supports MCP servers over **stdio, SSE, and Streamable HTTP**, for
  connecting models to external data/tools programmatically.
  ([Building MCP servers for ChatGPT/API](https://developers.openai.com/api/docs/mcp))
- **Apps SDK** is OpenAI's framework that **extends MCP to add UIs** — developers define both the
  logic (MCP server) and an interactive interface that renders **inside ChatGPT**. Streamable HTTP
  is recommended.
  ([Introducing Apps in ChatGPT](https://openai.com/index/introducing-apps-in-chatgpt/),
  [Apps SDK MCP server concept](https://developers.openai.com/apps-sdk/concepts/mcp-server))
- **ChatGPT Apps / connectors timeline:** Apps became available on **all plans (incl. Business,
  Enterprise, Education) on Nov 13, 2025**; on **Dec 17, 2025** "connectors" were renamed
  **"apps"** to unify UI-apps and search/reference connectors.
  ([Connectors in ChatGPT](https://help.openai.com/en/articles/11487775-connectors-in-chatgpt))
- In the API Playground you add an MCP server via **Tools → Add → MCP Server** and paste an HTTPS
  endpoint. ChatGPT Apps themselves are remote-only.

**What you'd build here.** A startup builds an MCP server for its product, wraps it with the Apps
SDK to render an interactive widget, and ships it as a ChatGPT App — reaching ChatGPT's user base
where users invoke it conversationally. Or, with the Agents SDK, an internal agent that mounts
a remote MCP server for company data.
([MintMCP build guide](https://www.mintmcp.com/blog/build-mcp-enable-apps-with-chatgpt))

**Strengths:** distribution to ChatGPT's massive audience; UI layer on top of MCP; first-class
SDK. **Limits:** ChatGPT-side apps are remote-only and gated by OpenAI's review/availability;
the Apps SDK is OpenAI-specific even though the underlying server is standard MCP.

---

## 7. Microsoft — Copilot Studio, VS Code, Semantic Kernel

Microsoft drove **enterprise** MCP adoption hardest.

- **Copilot Studio:** MCP went **generally available on May 29, 2025**. Makers add AI apps/agents
  with a few clicks, with **enterprise security and governance** — Virtual Network integration,
  Data Loss Prevention (DLP) controls, and multiple auth methods. GA added tool listing, enhanced
  tracing (which server/tool fired at runtime), Streamable transport, and **MCP resources**
  (preview) so agents can pull files/API responses/DB records. A marketplace of pre-built
  MCP-enabled connectors is available.
  ([MCP GA in Copilot Studio](https://www.microsoft.com/en-us/microsoft-copilot/blog/copilot-studio/model-context-protocol-mcp-is-now-generally-available-in-microsoft-copilot-studio/),
  [Extend agents with MCP](https://www.microsoft.com/en-us/microsoft-copilot/blog/copilot-studio/introducing-model-context-protocol-mcp-in-copilot-studio-simplified-integration-with-ai-apps-and-agents/))
- **VS Code + GitHub Copilot:** **Agent mode** (multi-step: reads codebase, edits, runs terminal/
  tests, self-corrects) shipped to all VS Code users, and **MCP support in VS Code reached GA on
  July 14, 2025**. GitHub also released an open-source local **GitHub MCP server**. MCP is still
  in preview in Visual Studio, Eclipse, and JetBrains Copilot editors.
  ([Agent mode + MCP rollout](https://github.blog/news-insights/product-news/github-copilot-agent-mode-activated/),
  [VS Code MCP GA](https://github.blog/changelog/2025-07-14-model-context-protocol-mcp-support-in-vs-code-is-generally-available/),
  [Enhance agent mode with MCP](https://docs.github.com/en/copilot/tutorials/enhance-agent-mode-with-mcp))
- **Semantic Kernel:** Microsoft's agent framework (the code path, analogous to LangChain) gained
  MCP support so .NET/Python agents can consume MCP servers — the programmatic counterpart to
  Copilot Studio's low-code path.

**What you'd build here.** A regulated enterprise builds a Copilot Studio agent that answers HR
questions, backed by an internal MCP server, with DLP policies ensuring no sensitive data leaves
the tenant — governed, audited, deployed to Teams. Devs use VS Code agent mode + a database MCP
server to ship code against live schemas.

**Strengths:** governance, DLP, VNet, auth, marketplace — the enterprise checklist. **Limits:**
Copilot Studio MCP is remote/cloud-tenant-oriented; deepest value is inside the Microsoft stack.

---

## 8. Other notable consumers

- **Goose (Block)** — open-source agent (**CLI + Desktop**) that **runs locally** and connects to
  LLMs (Claude, GPT, Gemini, local Llama/Qwen). It extends via MCP servers (called
  **"extensions,"** 70+ documented). Goose was effectively the **reference implementation that
  shaped MCP** before the protocol was named; new MCP features often land there first. Launched
  Jan 2025; in **Nov 2025 Block donated Goose to the Agentic AI Foundation** alongside Anthropic's
  donation of MCP.
  ([Block intro](https://block.xyz/inside/block-open-source-introduces-codename-goose),
  [Arcade: Goose & MCP](https://www.arcade.dev/blog/goose-the-open-source-agent-that-shaped-mcp/),
  [goose docs](https://goose-docs.ai/))
  *Build here:* a local-first autonomous agent with full filesystem/tool access and scheduled tasks.
- **Cline** — autonomous coding agent plugin for VS Code; consumes MCP servers (local + remote)
  for tools beyond the editor. Good when you want an agentic coder but not GitHub Copilot.
- **LibreChat** — open-source multi-LLM chat app (self-hostable) that supports MCP servers — a
  self-hosted alternative to Claude Desktop / ChatGPT for teams wanting their own front-end.
- **Dify** — open-source LLM app platform; its agents call external tools/MCP over SSE, supporting
  ReAct and function-calling strategies — a no/low-code app builder with MCP reach.
  ([PulseMCP client directory](https://www.pulsemcp.com/clients))

---

## No-code vs Code: the split

| | No-code / Visual | Code / Programmatic |
|---|---|---|
| **Tools** | n8n, Zapier, Make, Copilot Studio, ChatGPT Apps, claude.ai connectors | LangGraph/LangChain, OpenAI Agents SDK, Semantic Kernel, Goose/Cline |
| **You build by** | Clicking, wiring nodes, enabling actions | Writing agent code, hosting servers |
| **Control over orchestration** | Low–medium (platform decides loop) | Full (you own retries, memory, branching) |
| **Who it's for** | PMs, ops, analysts, NGO staff, solo builders | Engineers, platform teams |
| **Hosting burden** | None (or self-host n8n) | You host and operate everything |

**Rule of thumb.** If the workflow is "let an AI take actions across apps I already use," go
**no-code** (Zapier/Make for breadth, n8n for control + self-hosting). If you need custom
orchestration, multiple MCP servers chained with bespoke logic, or to embed agents in your own
product, go **code** (LangGraph or Agents SDK). Copilot Studio sits in between — low-code with
enterprise governance.

---

## Local vs Remote MCP — per-platform mapping

This is often the deciding constraint (data residency, security, what can run on a laptop).

- **Local `stdio` servers supported (run a process on your machine, plus remote):**
  Claude **Desktop**, Claude **Code**, **Cursor**, **Windsurf**, **VS Code + Copilot**, **Goose**,
  **Cline**, **LibreChat** (self-hosted), and **n8n self-hosted**.
- **Remote-only (cloud, HTTP/SSE):** **Zapier**, **Make**, **Copilot Studio**, **ChatGPT Apps**,
  and **claude.ai** in the browser. Data passes through the vendor's cloud.
- **Both, by design:** **LangChain/LangGraph** and the **OpenAI Agents SDK** support stdio + SSE +
  Streamable HTTP — you choose per server.

**Implication for NGOs / privacy-sensitive teams:** prefer local-stdio hosts (Claude Desktop,
VS Code, Goose) or **self-hosted n8n** when data can't leave your environment. The big cloud
automation platforms (Zapier/Make) are fastest to build in but route data through their servers.

---

## The trend: MCP as the universal connector layer

Through 2025, MCP went from "Anthropic's thing" to an **industry standard adopted by every major
player** in roughly nine months:

- **Nov 2024** — Anthropic introduces MCP.
- **Mar 2025** — OpenAI adopts MCP (Agents SDK, Responses API, ChatGPT desktop); LangChain ships
  MCP adapters.
- **May 2025** — MCP GA in Microsoft Copilot Studio.
- **Jul 2025** — MCP GA in VS Code; Claude connectors launch.
- **Nov 2025** — ChatGPT Apps on all plans; Block donates Goose, Anthropic donates MCP, to the
  Agentic AI Foundation.
- **Dec 2025** — OpenAI renames connectors → apps.

**What this means for a builder:**

1. **Write the integration once, run it everywhere.** A single MCP server you build is callable
   from Claude, ChatGPT, Cursor, VS Code, n8n, LangGraph, Copilot Studio, Goose — no per-client
   rewrites. The integration is now decoupled from the agent.
2. **The choice of platform is about UX and ops, not capability.** Since they all speak MCP, you
   pick based on no-code vs code, local vs remote, hosting, and governance — not "which tools are
   available."
3. **Governance moves to the protocol edge.** Enterprises (Copilot Studio) and platforms (Zapier
   task limits, n8n review gates) wrap MCP with auth, DLP, and audit — because the same server can
   be reached by many agents.
4. **Standardization → consolidation, then differentiation.** With connectivity commoditized,
   platforms compete on reasoning quality, governance, distribution (ChatGPT's user base), and
   visual/no-code ergonomics — not on whether they can reach your Slack.

**Bottom line for choosing.** Decide on two axes first — **no-code vs code** and **local vs
remote** — and the platform usually picks itself: Zapier/Make for fast cloud no-code; **n8n** for
visual + self-hosted control; **LangGraph/Agents SDK** for programmatic multi-MCP agents;
**Claude Desktop / Cursor / VS Code / Goose** for local-first interactive work; **Copilot Studio**
for governed enterprise. The MCP server you build behind any of them is portable to all the rest.

---

### Sources (primary references)

Anthropic MCP intro · Claude connectors docs · n8n MCP docs (Server Trigger, tools reference,
template #3770) · Zapier MCP + GitHub · Make MCP · LangChain MCP adapters (changelog, GitHub,
PyPI, docs) · OpenAI Agents/Apps SDK + ChatGPT help · Microsoft Copilot Studio MCP GA + VS Code
MCP GA + GitHub agent-mode docs · Block/Goose + Arcade. Inline links above point to each.
