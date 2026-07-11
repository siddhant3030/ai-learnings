# LangGraph & LangChain — concepts, usage, and chat-with-data thinking

Research pack produced by 5 parallel agents (July 2026): one read the actual Dalgo
production code, three researched the frameworks from official 1.x docs and
engineering posts, one surveyed how the industry builds chat-with-data.

## Reading order

| # | File | What it answers |
|---|------|-----------------|
| 1 | [01-how-dalgo-uses-langgraph-langchain.md](01-how-dalgo-uses-langgraph-langchain.md) | How `DDP_backend/ddpui/core/chat_with_data/` actually uses the frameworks today — concept-by-concept map with file:line refs, gaps, and risks |
| 2 | [02-langchain-core-concepts.md](02-langchain-core-concepts.md) | LangChain 1.x concepts: models, messages, tools, structured output, `create_agent`, middleware; what's legacy (`langchain-classic`) and honest criticisms |
| 3 | [03-langgraph-core-concepts.md](03-langgraph-core-concepts.md) | LangGraph concepts: StateGraph, checkpointers, `interrupt()` HITL, streaming modes, Send/subgraphs, durability; when it's overkill |
| 4 | [04-when-langchain-when-langgraph-when-both.md](04-when-langchain-when-langgraph-when-both.md) | The decision guide: the 1.x layering (create_agent IS a LangGraph graph), when each alone suffices, when both, when neither — mapped to chat-with-data stages |
| 5 | [05-how-to-think-about-chat-with-data.md](05-how-to-think-about-chat-with-data.md) | Mental models + industry landscape: Uber QueryGPT, Pinterest, LinkedIn SQL Bot, Snowflake/Databricks, Vanna/WrenAI/dbt; design axes, failure modes, evals as the moat |

## The one-paragraph synthesis

"LangChain vs LangGraph" is a false dichotomy in 1.x: `create_agent` compiles to a
LangGraph graph, so the real question is *high-level agent API + middleware* vs
*dropping down to the graph API*. Dalgo today is entirely on the high-level path
(prebuilt `create_agent` loop, middleware for prompts/trimming/retry-kill-switch,
Postgres checkpointer, `content_and_artifact` tools) with zero custom graph — which
matches how LangChain themselves recommend starting. The graph API becomes worth it
when you need deterministic multi-stage pipelines, typed state beyond messages,
bounded self-correction loops, `interrupt()`-grade human approval, or fan-out. For
chat-with-data specifically, the industry consensus (Uber, LinkedIn, Snowflake) is:
trust is the product, not SQL generation — semantic-layer-first context, fixed
pipelines over free-running agents, bounded retries, and a golden-query eval set as
the real moat.

## Notable findings about our own code (doc 01)

- Entire agent is `create_agent` + middleware; topology never customized — clean, but
  the dashboard-confirm flow is prompt-enforced only (no `interrupt()`), the softest
  guarantee in the write path.
- `langfuse` is imported but not declared in `pyproject.toml` — tracing may be silently off.
- No `adelete_thread` on session deletion: checkpoints (and any PII in them) accrete forever.
- Four bare `ChatAnthropic` sidecar calls (router, reflection, validator, titles) hand-roll
  JSON parsing instead of `with_structured_output`.
