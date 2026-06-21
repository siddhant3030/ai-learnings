# How Different Teams Build Engineering Harnesses

A comparative look at the scaffolding — context, tools, environments, verification, orchestration — that turns a raw LLM into a system that reliably ships software. Companion to the topic deep-dive in [`../research/harness-engineering.md`](../research/harness-engineering.md); this folder is the team-by-team, pattern-by-pattern, evidence-first treatment.

| # | Guide | Angle |
|---|-------|-------|
| 1 | [Team Case Studies](01-team-case-studies.md) | 16 teams (Devin, Cursor, Claude Code, Codex, Amp, Factory, Augment, OpenHands, SWE-agent, Goose, Stripe Minions, Jules/Antigravity, Aider, Cline, Windsurf, Zed) compared |
| 2 | [Architecture Patterns](02-architecture-patterns.md) | The 9 recurring building blocks — agent loop, context, ACI/edit format, verification, orchestration, hooks |
| 3 | [Environments & Sandboxing](03-environments-and-sandboxing.md) | Where the agent's code actually runs — runtimes, isolation, reusing dev infra, the speed lever |
| 4 | [Evals & What Wins](04-evals-and-what-wins.md) | The evidence that harness beats model — benchmarks, ablations, the production gap |
| 5 | [Build Your Own](05-build-your-own.md) | Synthesis — layered reference architecture, maturity ladder, anti-patterns, a Dalgo plan |

## The thesis, and the cleanest proof of it

**The harness is often the binding constraint, not the model.** The single sharpest piece of evidence: **Terminal-Bench 2.0 surfaces the harness as its own leaderboard column** — GPT-5.5 scores 84.7% / 83.1% / 82.2% under three different harnesses, *same model*. Edit format alone swings scores 33–62 points with the model frozen (Aider: GPT-4 Turbo 20%→61%; a practitioner experiment: 6.7%→68.3%), because a correctly-reasoned edit the harness can't apply scores zero.

## What recurs across every team

- **Verification beats generation.** Agents structurally declare premature success; a cheap gate that *checks* (tests/lint in the loop, a checklist middleware that blocks "done") recovers more reliability than a smarter model that *generates*.
- **"Good for humans is good for agents."** Stripe's Minions (1,300 PRs/week) reuse existing dev VMs, internal APIs, and CI — the only novel piece is a blueprint alternating deterministic and agentic nodes. The biggest lever is often *not building agent infrastructure at all*.
- **Speed is the hidden lever.** Warm-pool boots (Stripe ~10s) and fast snapshots (Devin 30min→15s) are what make the inner loop converge — invest in a sub-1-minute test loop before reaching for Firecracker.
- **Nobody runs a naive swarm.** Cognition argues "Don't Build Multi-Agents," yet Devin ships *sequential* subagents for narrow questions; the rule is never split a single *decision* across parallel contexts.
- **Context strategy is a deliberate fork** — deterministic structure (Sourcegraph SCIP, Aider tree-sitter+PageRank) vs fresh semantic indexing (Augment, Windsurf) vs Claude Code's grep + persistence-tiering. Decided by codebase scale and privacy, not by a universal winner.
- **Isolation leaks; layer it.** Egress allowlisting fails (the SOCKS5 null-byte bypass proved it) — "99% is a failing grade," so the sandbox underneath is what caps the damage.

## The reference architecture (guide 5)

Six layers, climbed in order, only as far as the task forces:

```
L0  rules + context (CLAUDE.md / .cursorrules)
L1  agent loop + fast feedback environment (tests/lint in the loop)
L2  tool/edit interface done well (the ACI)
L3  verification gates (test-in-loop, self-review, doom-loop detection)
L4  orchestration (subagents, pipelines, parallelism) — only when forced
L5  mechanical guardrails (hooks/policy), memory, evals/regression gates
        observability spans all layers
```

## For a small team (the Dalgo plan)

**First:** a pre-completion verification gate that refuses "done" until the repo's real tests/lint/typecheck pass — highest ROI, and it kills victory-declaration bias where the NGO trust budget is thinnest. (Guide 5 ties this to `dalgo-core`'s existing `harness-evolution.md` and the still-empty `hooks` block in `settings.json`.)

This connects directly back to the [`dalgo-core/docs/harness/`](../../dalgo-core/docs/harness/) research that opened this work — the auto-lint feedback hook and self-review gate are exactly the L1/L3 moves the wider field has converged on.
