# Environments & Sandboxing: Where the Agent's Code Actually Runs

> Prepared: June 2026
> Audience: engineers building or operating coding-agent harnesses
> Companion docs: [`research/harness-engineering.md`](../research/harness-engineering.md) (the loop,
> scaffold, verification) and [`mcp/03-security-and-auth.md`](../mcp/03-security-and-auth.md) (lethal
> trifecta, MCP auth, PII). This doc covers the layer those two assume: the **runtime environment**
> the agent acts inside.

The harness-engineering report makes the case that **Agent = Model + Harness**, and that the harness
is the binding constraint on reliability. This document zooms into one component of the harness that
gets less attention than prompts and tool design: **the execution environment** — the sandbox, the
filesystem, the network boundary, the secrets, the speed of the inner loop. This is the
underappreciated half of harness engineering. Not what the agent is told, but **where its code runs,
and what happens when that code is wrong or hijacked.**

---

## 1. Why the Environment Is a First-Class Harness Concern

A coding agent is not a chat model. It runs shell commands, installs dependencies, executes tests,
spins up servers, browses the web, and writes files. Every one of those actions touches a real
machine. The environment is what makes those actions possible — and what contains them when they go
wrong.

**The environment often determines success more than the model.** The harness-engineering report
already documents this for the *scaffold* layer (SWE-agent's ACI took GPT-4 from ~2% to ~12% on
SWE-Bench with no model change). The same is true at the *environment* layer, in a blunter way: an
agent that cannot run the test suite cannot do test-driven development; an agent whose inner loop
takes five minutes per iteration will time out before it converges. OpenAI's Codex team ran a strict
**1-minute maximum inner loop**, cycling through build tools (Make → Bazel → Turbo → NX) specifically
to keep it fast ([Latent Space](https://www.latent.space/p/harness-eng)). The environment was a
gating constraint they engineered against, not an afterthought.

**"What's good for humans is good for agents."** Stripe's most-cited lesson. Rather than building a
bespoke agent sandbox, Stripe ran its Minions agents inside **Devbox** — the exact AWS EC2 developer
environment its human engineers use every day, pre-loaded with the full source tree and warmed caches
([Stripe Dev, Minions Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2);
[ByteByteGo analysis](https://blog.bytebytego.com/p/how-stripes-minions-ship-1300-prs)). Fast feedback
loops, isolated environments, and standardized tooling were built for humans and turned out to be
exactly what agents need. The corollary: if your human dev setup is slow, flaky, or hard to
reproduce, your agents inherit all of it — amplified, because they iterate faster and never get
bored.

**The environment is also the safety boundary.** A chat model that hallucinates produces wrong text.
An agent that hallucinates — or gets prompt-injected — runs wrong *commands*: `rm -rf`, a leaked
credential, an exfiltration call. The environment is the thing standing between a bad decision and a
real consequence. This is where the [lethal trifecta](../mcp/03-security-and-auth.md#4-the-lethal-trifecta)
from the MCP security doc stops being abstract: a coding agent reading a malicious GitHub issue
(untrusted content), with access to `~/.aws/` (private data) and a working `curl` (exfiltration) **is
the trifecta**, fully assembled, on a developer's laptop.

So the environment is doing three jobs at once: **capability** (can the agent do the work?),
**speed** (can it iterate fast enough to converge?), and **containment** (when it's wrong, how much
can it break or leak?). The rest of this doc is about how teams trade those off.

---

## 2. Runtime / Sandbox Approaches

Six families, roughly ordered from most-isolated to least.

### 2.1 Comparison table

| Approach | Isolation tech | Startup | Persistence / snapshot | Best for | Examples |
|---|---|---|---|---|---|
| **MicroVM sandbox** | Firecracker microVM — dedicated guest kernel, hardware isolation | E2B: 80–200 ms; Firecracker ~125–150 ms boot, <5 MiB overhead/VM | Filesystem + process-state snapshots (E2B) | Running untrusted, LLM-generated code at scale | E2B, Fly Machines, AWS AgentCore |
| **Container sandbox** | OS namespaces/cgroups; warm pools | Daytona ~90 ms; Modal sub-second | Filesystem snapshots; Modal adds memory snapshots | Fast, cheap, high-concurrency code exec | Daytona, Modal, OpenHands runtime, GKE Agent Sandbox |
| **gVisor / Kata** | User-space kernel (gVisor) or lightweight VM (Kata) intercepting syscalls | gVisor fast start, 10–50% I/O overhead | Varies | Multi-tenant container hosting wanting >container isolation | gVisor (Google), Kata Containers |
| **Cloud dev workspace** | Per-task VM, full dev environment, repo cloned in | Devin snapshot resume ~15 s; Cursor fresh VM per task | Machine snapshots (Devin); ephemeral per-task (Cursor) | Long autonomous runs, browse + build + test + PR | Devin, Cursor Cloud Agents, Codespaces-style |
| **Reused human dev infra** | Whatever the org already runs (EC2 dev VMs) | Stripe Devbox ~10 s (pre-warmed pool) | "Cattle not pets" — disposable, re-provisioned | Orgs with strong existing dev-env tooling | Stripe Minions (Devbox) |
| **Local-machine (host)** | OS-native sandbox *if configured*, else none | Instant | Git worktrees / branches | Interactive dev on the developer's box | Claude Code (sandboxed), Aider & Goose (host by default) |
| **Ephemeral CI** | The CI runner's container, torn down after | CI job cold start | None — stateless per run | Headless, triggered, unattended tasks | `claude -p` in GitHub Actions, scheduled tasks |

> **Provenance note.** Cold-start figures conflict across sources. E2B's own homepage states "less
> than 200 ms" (and "80 ms" in its quick-start)
> ([e2b.dev](https://e2b.dev/)); Modal's product comparison lists E2B, Modal, Daytona and Fly all at
> "sub-second" ([Modal blog](https://modal.com/blog/top-code-agent-sandbox-products)). Treat sub-100ms
> numbers as best-case, same-region figures from vendor marketing, not guarantees.

### 2.2 Docker-based runtimes (OpenHands, Devin)

The harness-engineering report already covers the **OpenHands** runtime in detail (Docker sandbox,
EventStream, three-tier image tagging). The environment-layer point to add: OpenHands uses a
**client–server split** — the backend talks to an *action-execution server* running *inside* the
container over a REST API, sending actions and receiving observations
([OpenHands runtime docs](https://docs.openhands.dev/openhands/usage/architecture/runtime)). The agent
"brain" never shares a process with the code it runs. The container is built from an arbitrary user
base image, so the agent gets the project's real toolchain; the three-tier tags (source-hash →
dependency-hash → version) make rebuilds fast while staying reproducible. As of the V1 architecture
(Nov 2025) the older `AgentController` + `Runtime` split was collapsed into `Conversation` +
`Workspace`, and OpenHands ships local, remote, and hosted-cloud runtime backends behind the same
interface ([OpenHands SDK paper, arXiv:2511.03690](https://arxiv.org/html/2511.03690v1)).

Docker is the **practical minimum** for isolation: good filesystem and process separation, shared
host kernel. It's the right default for most teams; the gap versus a microVM only matters when you're
running genuinely untrusted code (see §3).

### 2.3 MicroVMs / Firecracker, gVisor

When the code is untrusted — arbitrary LLM output, or third-party-provided — a shared kernel is the
weak point. Two stronger options:

- **Firecracker microVMs** give each sandbox its own guest kernel. An attacker must escape *both* the
  guest kernel *and* the hypervisor. Boot is ~125–150 ms with <5 MiB overhead per VM
  ([Northflank](https://northflank.com/blog/how-to-sandbox-ai-agents);
  [SoftwareSeni](https://www.softwareseni.com/firecracker-gvisor-containers-and-webassembly-comparing-isolation-technologies-for-ai-agents/)).
  This is what **E2B** runs ("Each sandbox is powered by Firecracker, a microVM made to run untrusted
  workflows" — [e2b.dev](https://e2b.dev/)) and what AWS AgentCore uses (one Firecracker microVM per
  session — the "gold standard" the harness report cites).
- **gVisor** intercepts syscalls in a user-space kernel written in Go, so sandboxed code never talks
  to the real host kernel directly. Cheaper than a full VM but adds **10–50% overhead on I/O-heavy
  workloads** ([Northflank](https://northflank.com/blog/how-to-sandbox-ai-agents)). Good for
  compute-bound work; the consensus guidance is that teams running LLM-generated code should default
  to **Firecracker or Kata**, not gVisor, when the code is fully untrusted
  ([SoftwareSeni](https://www.softwareseni.com/firecracker-gvisor-containers-and-webassembly-comparing-isolation-technologies-for-ai-agents/)).

MicroVMs cost ~10–20% more than containers at scale — justified only when a multi-tenant escape would
be catastrophic.

### 2.4 Agent-sandbox products (E2B, Daytona, Modal, Fly)

A whole product category emerged in 2025–2026 to sell "secure infrastructure for running
AI-generated code" as a service. Comparison from Modal's own roundup
([Modal blog, 2025](https://modal.com/blog/top-code-agent-sandbox-products)) plus vendor pages:

| Product | Isolation | Cold start | Snapshots | Pricing (per CPU·s) | Note |
|---|---|---|---|---|---|
| **E2B** | Firecracker microVM | ~80–200 ms (same region) | Filesystem + process state | ~$0.000028 | Open-source runtime; "1b+ started sandboxes," $21M Series A ([e2b.dev](https://e2b.dev/)) |
| **Daytona** | Container + warm pool | ~90 ms | Filesystem | ~$0.000028 | Pivoted Feb 2025 from dev-env mgr to agent sandbox; compliance-first positioning |
| **Modal** | Container (serverless) | Sub-second | Filesystem + **memory** | ~$0.0000131 | Autoscale 0 → 10,000+ units; can run a GPU inside the sandbox |
| **Fly Machines** | Micro-VM | Sub-second | Memory | ~$0.00000053 | Global edge regions; cheapest listed |
| **Together** | Full VM from snapshot | 500 ms resume / 2.7 s cold | Filesystem + memory | ~$0.0000248 | Hot-swappable 2–64 vCPU |

The shared pattern: an SDK that lets the harness `create_sandbox()`, run commands, read/write files,
and snapshot — abstracting the isolation tech away. For a small team, this trades dollars for not
operating Firecracker yourself.

### 2.5 Cloud dev workspaces (Devin, Cursor)

These give the agent a **full development environment in the cloud** — terminal, editor, browser —
and run long, autonomously.

- **Devin** "spins up its own cloud environment with a terminal, code editor, and browser." Its
  signature feature is **machine snapshots**: the workspace *resets to a saved machine state at the
  start of every session*, with repos cloned and tools installed. Cognition cut snapshot creation
  from ~30 min to **~15 s** and time-to-first-message to ~10 s during 2025
  ([Devin 2025 release notes](https://docs.devin.ai/release-notes/2025);
  [Cognition 2025 review](https://cognition.ai/blog/devin-annual-performance-review-2025)). Snapshots
  + Playbooks (repo conventions) + secrets management + native Git are the four pillars of its env
  setup.
- **Cursor Cloud Agents** (formerly Background Agents) clone the repo into an **isolated VM, one fresh
  VM per task** — own filesystem, terminal, network stack, package manager — so "failed builds or
  broken dependencies never affect your local environment"
  ([Cursor Cloud Agent docs](https://cursor.com/docs/cloud-agent)). A Feb 2026 "Computer Use" update
  let agents open a browser, hit `localhost`, and click through the UI to verify changes visually.
  Code "lives only in the isolated VM for the duration of the task and is not retained" (SOC 2 Type
  II). Enterprises wanting code to never leave their perimeter can run
  [self-hosted cloud agents](https://cursor.com/blog/self-hosted-cloud-agents) in their own infra.

### 2.6 Local-machine agents (Claude Code, Goose, Aider)

The agents most developers actually run day-to-day execute on the **host machine** — and historically
with no isolation at all. This is the highest-capability, lowest-containment end of the spectrum.

- **Goose** (Block) "has no built-in sandboxing whatsoever — the agent runs with full user
  permissions," able to run arbitrary shell commands and (on macOS) `osascript` automation. Community
  best practice is to wrap it in Docker/Podman with restricted mounts and egress filtering — but
  that's opt-in, not default
  ([Goose sandbox analysis](https://agent-safehouse.dev/docs/agent-investigations/goose);
  [Docker × Goose](https://www.docker.com/blog/building-ai-agents-with-goose-and-docker/)).
- **Aider** likewise runs as a host-process CLI with the user's permissions; isolation is the user's
  responsibility (run it in a container/devcontainer).
- **Claude Code** is the notable exception: in **October 2025 Anthropic shipped native OS-level
  sandboxing** (covered in §3) — making "agent runs on your laptop" and "agent is contained" no
  longer mutually exclusive.

The "(or lack of)" is the lesson: the default for local agents is **full ambient authority on your
machine**. That's fine for an experienced developer who reads every action; it's a loaded gun for
unattended or auto-approved runs.

### 2.7 Ephemeral CI-based execution

The leanest environment: trigger an agent headless inside a CI runner, let it work, capture the
result, throw the environment away. Claude Code's `claude -p "<prompt>"` runs **one-shot without a
TTY and exits**, ideal for CI stages, PR-comment triggers, nightly crons, and large migrations
([Headless Claude in CI, AgentPatterns](https://agentpatterns.ai/workflows/headless-claude-ci/);
[hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)).
The agent "starts, reads an issue, implements a feature, opens a PR, and shuts down" — leaving a full
decision log behind. The environment is the CI container: stateless, reproducible, already
credential-scoped by your CI's secret model, and torn down after each run. Anthropic's 2026
**Scheduled Tasks** is the cloud-native sibling (cron-driven Claude Code on managed infra).

---

## 3. Isolation & Security

The harness report covers defense-in-depth as a *pattern*. Here is how it's actually implemented at
the environment layer, and where it leaks.

### 3.1 The two isolations that must both hold

Anthropic's Claude Code sandboxing (Oct 2025) is the clearest production reference. It rests on a
principle worth memorizing:

> "Without network isolation, a compromised agent could exfiltrate sensitive files like SSH keys;
> without filesystem isolation, a compromised agent could easily escape the sandbox."
> — [Anthropic, Claude Code sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)

**You need both.** Filesystem-only sandboxing still lets a prompt-injected agent `curl` your secrets
out. Network-only sandboxing still lets it overwrite `~/.bashrc` to escape. The two legs map directly
onto the lethal trifecta: filesystem isolation constrains the *private-data* leg, network isolation
constrains the *exfiltration* leg.

How it's built:

- **Filesystem isolation** — kernel-enforced at the syscall level. Read/write allowed in the working
  directory; writes outside it are blocked. On **Linux via bubblewrap (bwrap)** (the unprivileged
  sandbox Flatpak uses), on **macOS via Seatbelt** (the framework Apple uses to isolate App Store
  apps). Boundaries apply to spawned subprocesses, not just Claude's direct calls.
- **Network isolation** — all traffic routed through a **local proxy that enforces a domain
  allowlist**; new destinations prompt for confirmation. Well-behaved HTTP clients go through the
  proxy; clients that ignore proxy env vars hit a Seatbelt backstop that blocks non-loopback traffic
  at the socket layer.

Internal result: **sandboxing reduced permission prompts by 84%**
([Anthropic](https://www.anthropic.com/engineering/claude-code-sandboxing)). Anthropic open-sourced
the engine as **`srt` (sandbox-runtime)** — a container-free CLI that wraps *any* command or server
with the same filesystem + network rules
([anthropic-experimental/sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime)).

### 3.2 Egress control is the highest-leverage — and leakiest — control

Network egress is where the trifecta is actually broken: cut the exfiltration leg and stolen data has
nowhere to go. But egress allowlisting is **hard to get right**, and a concrete failure shows why.

A Claude Code sandbox-bypass (versions v2.0.24–v2.1.89, ~5.5 months) used a **SOCKS5 hostname
null-byte injection**: a hostname like `attacker.example\x00.google.com` passed the JS policy check
(`.endsWith(".google.com")` → allowed) but the C resolver `getaddrinfo()` stopped at the null byte
and dialed `attacker.example`. The proxy and the OS *disagreed about what the hostname was* — a
classic parser-differential. The proxy also doesn't terminate/inspect TLS, and sandboxed processes
inherit env vars (including credentials) unless explicitly scrubbed
([Penligent writeup](https://www.penligent.ai/hackinglabs/claude-code-sandbox-bypass/)).

The lesson is **not** "sandboxing is useless" — it's exactly Simon Willison's warning from the MCP
security doc: guardrails are probabilistic, and *"in web application security, 99% is a failing
grade."* A sandbox is a strong, necessary layer, not a proof. Pair egress allowlisting with: a
**deny-by-default** policy (allowlist, never blocklist), TLS-aware egress proxies where possible,
**environment scrubbing** so creds aren't sitting in env vars, and the trifecta discipline of keeping
private-data access *off the same agent* as untrusted-content ingestion.

### 3.3 Secrets handling

The iron rule from the MCP doc carries over verbatim: **the model must never see the credential.** At
the environment layer that means:

- **Inject secrets the agent can *use* but not *read* where possible.** A pre-authenticated `gh` CLI
  or a git credential helper lets the agent push without ever holding the token in context. Devin and
  Cursor both provide managed secrets that are present in the env but kept out of the transcript.
- **Scrub the environment in the sandbox.** The null-byte bypass mattered *because* `~/.aws/`,
  `~/.config/gh/`, model API keys, and cloud metadata (`169.254.169.254`) were reachable. Default-deny
  reads of `.env`, `~/.ssh`, `secrets/`, and block the metadata endpoint.
- **Short-lived, narrowly-scoped, per-task credentials.** A CI-issued token scoped to one repo and
  expiring in an hour caps blast radius better than a long-lived PAT with org-wide access.

### 3.4 Blast-radius limits and the `--dangerously-skip-permissions` problem

The most dangerous mode in any local agent is "approve everything." Claude Code's
`--dangerously-skip-permissions` "removes all permission checks; every action proceeds, regardless of
risk." It is **only** safe in a disposable, network-locked environment where "nothing of real value
exists and the whole system can be reset"
([Truefoundry](https://www.truefoundry.com/blog/claude-code-dangerously-skip-permissions)). A
hardcoded check even refuses to run it as root/sudo unless inside a recognized sandbox.

Anthropic's safer replacement — **Auto Mode** (a two-layer classifier defense) — is instructive about
what "blast radius" really requires:

- An **input probe** screens tool outputs for prompt injection *before* they reach the agent; an
  **output classifier** evaluates each action before execution.
- It targets four behaviors: **overeager initiative** (using found credentials, deleting "obstacles"),
  **scope misunderstanding** (treating shared resources as isolated), **prompt injection**, and
  **misalignment**.
- The classifier sees only user messages and tool commands — *"We strip assistant text so the agent
  can't talk the classifier into making a bad call."*
- Two-stage filtering cut false positives from 8.5% → **0.4%**; but it still misses **17% of genuinely
  overeager actions** (false negatives) — Anthropic's own honest number
  ([Anthropic, auto mode](https://www.anthropic.com/engineering/claude-code-auto-mode)).

That 17% is the whole argument for the sandbox underneath: **a classifier reduces the rate of bad
actions; the environment caps the damage of the ones that get through.** Autonomy modes and isolation
are complements, not substitutes.

---

## 4. Statefulness & Checkpointing

Long agent runs need to survive restarts, branch off cleanly, and roll back. Three mechanisms:

- **Machine snapshots** (Devin). The workspace resets to a saved state at session start — repos
  cloned, deps installed, services warmed. This is both a **speed** optimization (no re-setup) and a
  **statefulness** one (every session starts from a known-good baseline). Cutting snapshot creation
  from 30 min → 15 s is what made per-session resets practical
  ([Devin 2025 notes](https://docs.devin.ai/release-notes/2025)).
- **Filesystem / process-state snapshots** (E2B, Modal, Together). The sandbox products expose
  snapshot APIs so a harness can pause a long run, persist it, and resume — Modal and Together even
  snapshot *memory*, so an in-progress process resumes mid-execution
  ([Modal blog](https://modal.com/blog/top-code-agent-sandbox-products)). This is how you survive a
  context-window handoff (the harness report's pattern of saving plans to memory and spawning a fresh
  agent) without losing the *environment* state alongside the *context* state.
- **Git worktrees / branches as isolation + rollback** (covered in §5). Branch-first means every
  agent run is a discardable unit: if it goes wrong, `git worktree remove` and the main tree is
  untouched.

The harness report's "stale memory is more dangerous than no memory" insight has an environment-layer
twin: **a snapshot that's drifted from reality is a trap.** Devin's Playbooks and per-session resets
exist partly so the agent's *environment* matches its *beliefs* about the environment — re-baselining
each session prevents accumulated drift.

---

## 5. Parallelism

The economics of agents push hard toward running many at once — they're cheap relative to engineer
time, and most coding tasks are independent. The constraint is **isolation**: parallel agents must
not step on each other's files, branches, ports, or state.

| Technique | Granularity | Isolation | Scale reported |
|---|---|---|---|
| **Git worktrees** | One working dir + branch per agent | Shared repo history, separate files | 4–8 concurrent per developer "reliably"; above that, **review** is the bottleneck, not the agent |
| **Container/VM per agent** | One sandbox per task | Full FS/net/process | Modal: autoscale to 10,000+ units; Cursor: "as many as you want" |
| **Disposable dev VMs ("cattle")** | One EC2 Devbox per task | Full machine | Stripe: ~6 per engineer in parallel → **1,300+ PRs/week** org-wide |

**Git worktrees** are the cheapest parallelism primitive and now first-class in Claude Code: each
session gets "its own files and branch, sharing the same repository history," so concurrent edits
never collide; you can set `isolation: worktree` on a subagent's frontmatter
([Claude Code worktrees docs](https://code.claude.com/docs/en/worktrees);
[Developers Digest](https://www.developersdigest.tech/blog/git-worktrees-claude-code-parallel-agents-guide)).
The April 2026 Claude Code desktop redesign made worktree isolation a UI feature (multi-session
sidebar). Practical ceiling: **4–8 worktrees per developer** — beyond that the human reviewer is
saturated, which is the real limit on every parallel setup.

**Container-per-agent** is the cloud version: Cursor gives every task a fresh VM and lets you "run as
many agents as you want in parallel" without your laptop online
([Cursor docs](https://cursor.com/docs/cloud-agent)); Modal autoscales sandboxes "from zero to
10,000+ concurrent units." **Stripe's Devbox fleet** is the proof at org scale: "cattle, not pets,"
one disposable EC2 per task, ~6 in parallel per engineer, producing **1,300+ PRs/week with zero
human-written code** ([Stripe Minions Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)).
Note what makes that throughput possible: the **pre-warmed pool** (§6) and the **two-CI-cycle cap**
(§7) — parallelism without fast environments and bounded iteration just multiplies waste.

---

## 6. The Feedback Loop the Environment Enables

The single most important environment property for agent *performance* (not safety) is **how fast the
inner loop runs**. An agent doing red-green-refactor is bottlenecked on test/lint/typecheck latency,
and unlike a human it will burn that latency on every iteration without complaint or shortcut.

- **Pre-warming / caching is the lever.** Stripe's Devboxes boot in **10 seconds** only because Stripe
  *proactively provisions and warms a pool* — cloning repos, warming Bazel and type-check caches, and
  starting background services *ahead of time*
  ([ByteByteGo](https://blog.bytebytego.com/p/how-stripes-minions-ship-1300-prs)). The agent never
  pays setup cost. Devin's machine snapshots do the same: deps pre-installed, services warm, so the
  agent starts working in ~10 s, not after a 5-minute `npm install`.
- **Slow environments kill agent performance directly.** OpenAI's team cycling Make → Bazel → Turbo →
  NX to hold a **1-minute inner loop** is the clearest evidence: when iteration is slow, the agent
  either times out or accumulates context noise across long waits. The harness report's
  "context anxiety" anti-pattern is worse when each iteration is expensive.
- **Errors must be fast *and* legible.** A 10-minute test run that returns "FAILED" with no detail
  costs more than a 30-second run that prescribes the fix. Fast feedback and *prescriptive* feedback
  (the harness report's linter pattern) compound: the agent needs the loop to be both quick and
  informative to converge.

For a small team the takeaway is concrete: **invest in caching and a fast test subset before you
invest in a fancier sandbox.** A Docker container with warm caches and a 20-second test loop will
outperform a Firecracker microVM with cold caches and a 4-minute loop.

---

## 7. Deployment / CI Integration

The environment doesn't end at "agent finished." The output has to land somewhere reviewable, with a
human gate at the boundary.

- **Agents open PRs, humans review.** The dominant production pattern: the agent works in its isolated
  environment, opens a pull request, and a human reviews before merge. Cursor cloud agents, Devin, and
  headless Claude all converge here. The PR is the **handoff artifact** — a diff plus a decision log —
  and the review is the **human attention gate** the harness report calls "the only fundamentally
  scarce thing."
- **Bounded CI retry — the two-cycle cap.** Stripe's explicit policy: after **two push-and-test
  cycles**, the code goes to human review, because there are "diminishing marginal returns if an LLM
  is running against indefinitely many rounds of a full CI loop"
  ([Stripe Minions Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)).
  This is an *environment* control as much as a loop control: each CI cycle costs real compute and
  wall-clock, and unbounded retries are how a stuck agent burns a budget.
- **The CI environment is where ephemeral execution and deployment meet.** `claude -p` triggered by a
  PR comment or issue runs inside CI, scoped by CI's existing secret model, and emits a
  machine-readable result the pipeline acts on. The same container that builds and tests the change is
  the agent's sandbox — minimal extra infrastructure
  ([AgentPatterns](https://agentpatterns.ai/workflows/headless-claude-ci/)).

---

## 8. Reference: Setting Up a Good Agent Environment for a Small Team

Tuned for Dalgo's constraints: a small engineering team, ~20 partner NGOs, **beneficiary PII** (names,
locations, health/financial data — the "private data" leg of the lethal trifecta, per the
[MCP security doc](../mcp/03-security-and-auth.md#8-data-scrubbing-pii)), an AGPL codebase across
`DDP_backend` (Django) and `webapp_v2` (Next.js), and a budget that rules out operating Firecracker
fleets in-house.

### 8.1 Sandbox choice — match isolation to trust

| Scenario | Recommended environment | Why |
|---|---|---|
| Engineer running Claude Code interactively on `DDP_backend` / `webapp_v2` | **Local + native sandbox on** (bwrap/Seatbelt), working dir scoped to the repo | Highest capability, now contained; 84% fewer prompts |
| Unattended / scheduled / `claude -p` in CI | **Ephemeral CI container** with scoped secrets, torn down per run | Stateless, already credential-scoped, no standing blast radius |
| Parallel feature work | **Git worktrees**, one per agent, branch-first | Cheap isolation, clean rollback, native to Claude Code |
| Running genuinely untrusted code (a partner's data-transform snippet, scraped dbt model) | **Managed microVM sandbox (E2B)** or container sandbox (Modal/Daytona) | Don't run untrusted code on a host or shared kernel |

Default to **local + native sandbox** for daily work and **git worktrees** for parallelism. Reach for
a managed sandbox product only when the code being executed is untrusted — buy it, don't operate
Firecracker yourself.

### 8.2 Network policy — deny by default

1. **Allowlist egress, never blocklist.** Start from zero outbound. Add only what's needed: PyPI/npm
   registries, the Dalgo warehouse, GitHub, the LLM API endpoint.
2. **Block the cloud metadata endpoint** (`169.254.169.254`) and private/reserved ranges — the SSRF
   checklist from the MCP doc applies to the agent's egress too.
3. **Keep the private-data path off the untrusted-content path.** An agent reading a partner's
   uploaded CSV or a GitHub issue (untrusted) should **not** also hold warehouse credentials in the
   same session. This is the single highest-leverage trifecta break for an NGO data platform.
4. **Prefer a TLS-aware egress proxy** over a hostname allowlist alone — the null-byte bypass shows
   string-matching hostnames is fragile.

### 8.3 Secrets — the model never sees them

1. **Pre-authenticate tooling** (`gh`, git credential helper, warehouse client) so the agent *uses*
   creds without *reading* them.
2. **Scrub the environment**: deny reads of `.env`, `~/.ssh`, `~/.aws`, `~/.config/gh`, `secrets/`.
3. **Per-task, short-lived, repo-scoped tokens.** A CI token scoped to one repo, expiring in an hour,
   not an org-wide PAT.
4. **Field-level PII redaction before any tool result leaves the warehouse** — the highest-leverage
   control for Dalgo specifically (per the MCP doc): never `SELECT *` beneficiary rows into context.

### 8.4 Speed — pre-warm before you optimize the model

1. **Cache aggressively.** A devcontainer image with deps pre-installed and a warm test cache. For
   `DDP_backend`, a pre-built venv + warm pytest cache; for `webapp_v2`, a warm `node_modules` and
   Next build cache.
2. **Give the agent a fast test subset** — the relevant module's tests, not the whole suite, for the
   inner loop; full suite only at the verification gate.
3. **Target a sub-1-minute inner loop** as a hard goal, the way OpenAI did. If it's slower, fix that
   before touching prompts.

### 8.5 Deployment gate

1. **Agent opens a PR; a human reviews before merge.** Non-negotiable for anything touching
   beneficiary data, irreversible, or costing money (the NGO trust budget is thin — one bad autonomous
   action sets adoption back).
2. **Cap CI retries at two cycles**, then escalate to a human — Stripe's rule, and a budget control.
3. **Keep the decision log.** The PR plus the agent's transcript is your audit trail; retain it
   short-term (7–14 days, per the MCP doc's telemetry guidance) and scrub PII from it.

---

## 9. Security Checklist (Environment Layer)

Copy-usable. Pairs with the MCP doc's §10 checklist (auth, tool-poisoning, supply chain) — this one
covers the *runtime environment*.

### Isolation
- [ ] **Both** filesystem **and** network isolation enabled (neither alone is sufficient).
- [ ] Filesystem writes scoped to the working dir; reads of secret dirs (`.env`, `~/.ssh`, `~/.aws`, `~/.config/gh`, `secrets/`) **denied**.
- [ ] Untrusted / LLM-generated code runs in a **microVM or container sandbox**, never directly on a host or shared kernel.
- [ ] Isolation applies to **spawned subprocesses**, not just the agent's direct calls.

### Network egress
- [ ] Egress is **deny-by-default**; outbound destinations **allowlisted**, not blocklisted.
- [ ] Cloud metadata endpoint (`169.254.169.254`) and private/reserved IP ranges **blocked**.
- [ ] Egress proxy is **TLS-aware** where possible; hostname allowlisting alone is treated as fragile (null-byte / parser-differential risk).
- [ ] Auto-fetch of model-supplied image/link URLs **disabled**.

### Lethal trifecta (environment instantiation)
- [ ] No single agent session simultaneously has: private-data access **+** untrusted-content ingestion **+** working exfiltration. At least one leg broken per session.
- [ ] Private-data credentials **not present** in sessions that read untrusted input (issues, scraped pages, uploaded files).

### Secrets
- [ ] Credentials **injected for use, not reading** (pre-auth'd `gh`/git/warehouse clients); never echoed into tool output or errors.
- [ ] Environment **scrubbed** of standing credentials; per-task, short-lived, narrowly-scoped tokens.
- [ ] Beneficiary **PII redacted at the source** before any tool result leaves the warehouse.

### Autonomy & blast radius
- [ ] `--dangerously-skip-permissions` used **only** in disposable, network-locked, no-value environments — never on a dev box or anything touching prod.
- [ ] Auto/approve-all modes paired with an **underlying sandbox** (classifiers miss ~17%; isolation caps the rest).
- [ ] Destructive/irreversible/cost-incurring operations gated behind explicit human approval.

### Speed & feedback
- [ ] Environment **pre-warmed** (deps installed, caches warm, services up) before the agent starts.
- [ ] Inner loop (test/lint/typecheck) targets **< 1 minute**; fast test subset for iteration, full suite at the gate.
- [ ] Error output is **prescriptive**, not just "FAILED".

### Statefulness & parallelism
- [ ] Long runs **snapshot-able** and resumable; snapshots re-baselined each session to prevent drift.
- [ ] Parallel agents isolated via **worktrees or container-per-agent**; no shared mutable state.
- [ ] Parallelism scaled to **review capacity**, not just compute (the human gate is the real ceiling).

### Deployment
- [ ] Agent output lands as a **PR with a decision log**; **human review before merge** for anything touching PII, irreversible, or costing money.
- [ ] CI retries **bounded** (e.g. two cycles) before human escalation.
- [ ] Transcripts retained short-term (7–14 days) and **PII-scrubbed**.

---

## Sources

**Claude Code sandboxing / autonomy**
- [Making Claude Code more secure and autonomous with sandboxing (Anthropic)](https://www.anthropic.com/engineering/claude-code-sandboxing)
- [How we built Claude Code auto mode (Anthropic)](https://www.anthropic.com/engineering/claude-code-auto-mode)
- [anthropic-experimental/sandbox-runtime (`srt`)](https://github.com/anthropic-experimental/sandbox-runtime)
- [Configure the sandboxed Bash tool (Claude Code docs)](https://code.claude.com/docs/en/sandboxing)
- [Claude Code `--dangerously-skip-permissions` (Truefoundry)](https://www.truefoundry.com/blog/claude-code-dangerously-skip-permissions)
- [Claude Code sandbox bypass — SOCKS5 null-byte egress (Penligent)](https://www.penligent.ai/hackinglabs/claude-code-sandbox-bypass/)
- [Run parallel sessions with worktrees (Claude Code docs)](https://code.claude.com/docs/en/worktrees)
- [Headless Claude in CI (AgentPatterns)](https://agentpatterns.ai/workflows/headless-claude-ci/)
- [Claude Code in CI/CD and headless automation (hidekazu-konishi)](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)

**Stripe Minions / reused dev infra**
- [Minions: Stripe's one-shot, end-to-end coding agents — Part 2 (Stripe Dev)](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- [How Stripe's Minions Ship 1,300 PRs a Week (ByteByteGo)](https://blog.bytebytego.com/p/how-stripes-minions-ship-1300-prs)

**OpenHands runtime**
- [Runtime Architecture (OpenHands docs)](https://docs.openhands.dev/openhands/usage/architecture/runtime)
- [OpenHands Software Agent SDK (arXiv:2511.03690)](https://arxiv.org/html/2511.03690v1)

**Cloud dev workspaces**
- [Devin 2025 release notes](https://docs.devin.ai/release-notes/2025)
- [Devin's 2025 Performance Review (Cognition)](https://cognition.ai/blog/devin-annual-performance-review-2025)
- [Cloud Agents (Cursor docs)](https://cursor.com/docs/cloud-agent)
- [Run cloud agents in your own infrastructure (Cursor)](https://cursor.com/blog/self-hosted-cloud-agents)

**Sandbox products & isolation tech**
- [Top AI Code Sandbox Products in 2025 (Modal)](https://modal.com/blog/top-code-agent-sandbox-products)
- [E2B — The Enterprise AI Agent Cloud](https://e2b.dev/)
- [How to sandbox AI agents in 2026: Firecracker, gVisor & isolation (Northflank)](https://northflank.com/blog/how-to-sandbox-ai-agents)
- [Firecracker, gVisor, Containers, WebAssembly — isolation comparison (SoftwareSeni)](https://www.softwareseni.com/firecracker-gvisor-containers-and-webassembly-comparing-isolation-technologies-for-ai-agents/)

**Local agents (Goose / Aider)**
- [Goose — Sandbox Analysis Report (Agent Safehouse)](https://agent-safehouse.dev/docs/agent-investigations/goose)
- [Building AI agents with Goose and Docker (Docker)](https://www.docker.com/blog/building-ai-agents-with-goose-and-docker/)

**Lethal trifecta (cross-reference)**
- [The lethal trifecta for AI agents (Simon Willison)](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)
- See also: [`mcp/03-security-and-auth.md`](../mcp/03-security-and-auth.md) §4 (lethal trifecta), §7 (secrets), §8 (PII)
</content>
</invoke>
