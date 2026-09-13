# Dive into Claude Code

*The Design Space of Today's and Future AI Agent Systems*

<p align="center">
  <img src="./assets/main_structure.png" width="85%" alt="High-level system structure of Claude Code">
</p>

<p align="center">
  <a href="https://arxiv.org/pdf/2604.14228v2"><img src="https://img.shields.io/badge/Paper-PDF-blue.svg?logo=adobeacrobatreader&logoColor=white" alt="Paper"></a>
  <a href="https://arxiv.org/abs/2604.14228"><img src="https://img.shields.io/badge/arXiv-2604.14228-b31b1b.svg" alt="arXiv"></a>
  <a href="./LICENSE"><img src="https://img.shields.io/badge/License-CC--BY--NC--SA--4.0-lightgrey.svg" alt="License"></a>
  <a href="https://github.com/VILA-Lab/Dive-into-Claude-Code/stargazers"><img src="https://img.shields.io/github/stars/VILA-Lab/Dive-into-Claude-Code?style=social" alt="Stars"></a>
  <a href="#contributing"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg" alt="PRs Welcome"></a>
</p>

<p align="center">
  <b>English</b> | <a href="./README_zh.md">中文</a>
</p>

> **A comprehensive source-level architectural analysis of Claude Code (v2.1.88, ~1,900 TypeScript files, ~512K lines of code), combined with a curated collection of community analyses, a design-space guide for agent builders, and cross-system comparisons.**

> [!TIP]
> **TL;DR** -- Only 1.6% of Claude Code's codebase is AI decision logic. The other 98.4% is deterministic infrastructure -- permission gates, context management, tool routing, and recovery logic. The agent loop is a simple while-loop; the real engineering complexity lives in the systems around it. This repo dissects that architecture and distills it into actionable design guidance for anyone building AI agent systems.

---

## Table of Contents

**From Our Paper**

- [🌟 Key Highlights](#key-highlights)
- [📖 Reading Guide](#reading-guide)
- [🏗️ Architecture at a Glance](#architecture-at-a-glance)
- [🧭 Values and Design Principles](#values-and-design-principles)
- [🔄 The Agentic Query Loop](#the-agentic-query-loop)
- [🛡️ Safety and Permissions](#safety-and-permissions)
- [🧩 Extensibility](#extensibility)
- [🧠 Context and Memory](#context-and-memory)
- [👥 Subagent Delegation](#subagent-delegation)
- [💾 Session Persistence](#session-persistence)

**Beyond the Paper**

- [🛰️ New Signals in the Agent Design Space](#new-signals-in-the-agent-design-space)
- [🛠️ Build Your Own AI Agent: A Design Guide](#build-your-own-ai-agent-a-design-guide)
- [⚖️ Cross-System Comparison: Claude Code vs OpenClaw vs Hermes-Agent](#cross-system-comparison-claude-code-vs-openclaw-vs-hermes-agent)
- [🔎 Find Resources by Design Question](#find-resources-by-design-question)
- [🌐 Community Projects & Research](#community-projects--research)
- [🚀 Other Notable AI Agent Projects](#other-notable-ai-agent-projects)
- [🤝 Contributing](#contributing)
- [🔖 Citation](#citation)

---

## Key Highlights

- **98.4% Infrastructure, 1.6% AI** -- The agent loop is a simple while-loop; the real complexity is permission gates, context management, and recovery logic.
- **5 Values → 13 Principles → Implementation** -- Every design choice traces back to human authority, safety, reliability, capability, and adaptability.
- **Bounded Analysis with an Approval Fallback** -- In the legacy shell-parser path, more than 50 subcommands triggers an `ask` decision.
- **2 CVEs Reveal a Pre-Trust Window** -- Extensions execute *before* the trust dialog appears.
- **The Cross-Cutting Harness Resists Reimplementation** -- The loop is easy to copy; hooks, classifier, compaction, and isolation are not.

---

## Reading Guide

| If you are a... | Start here | Then read |
|:----------------|:-----------|:----------|
| **Agent Builder** | [Build Your Own Agent](./docs/build-your-own-agent.md) | [Architecture Deep Dive](./docs/architecture.md) |
| **Security Researcher** | [Safety and Permissions](#safety-and-permissions) | [Architecture: Safety Layers](./docs/architecture.md#seven-independent-safety-layers) |
| **Product Manager** | [Key Highlights](#key-highlights) | [Values and Principles](#values-and-design-principles) |
| **Researcher** | [Full Paper (arXiv)](https://arxiv.org/abs/2604.14228) | [Community Resources](#community-projects--research) |

`1,884 files` ·  `~512K lines` ·  `v2.1.88` ·  `7 safety layers` ·  `5 compaction stages` ·  `54 tools` ·  `27 hook events` ·  `4 extension mechanisms` ·  `7 permission modes`

The architecture sections describe **Claude Code v2.1.88**. The design guide and resource catalog extend that analysis with work reviewed through **September 7, 2026**. The main PDF link opens [arXiv v2, July 2, 2026](https://arxiv.org/abs/2604.14228v2); the [April 2026 repository PDF](./paper/Dive_into_Claude_Code.pdf) remains available as an archive.

---

<details open>
<summary><h2>Architecture at a Glance</h2></summary>

Claude Code answers **four design questions** that every production coding agent must face:

| Question | Claude Code's Answer |
|:---------|:---------------------|
| Where does reasoning live? | Model reasons; harness enforces. ~1.6% AI, 98.4% infrastructure. |
| How many execution engines? | One `queryLoop` for all interfaces (CLI, SDK, IDE). |
| Default safety posture? | Deny-first: deny > ask > allow. Strictest rule wins. |
| Binding resource constraint? | ~200K (older models) / 1M (Claude 4.6 series) context window. Context-management stages activate under their own conditions. |

The system decomposes into **7 components** (User → Interfaces → Agent Loop → Permission System → Tools → State & Persistence → Execution Environment) across **5 architectural layers**.

<p align="center">
  <img src="./assets/layered_architecture.png" width="100%" alt="5-layer subsystem decomposition">
</p>

> [!NOTE]
> For the full architectural deep dive -- 7 safety layers, 9-step turn pipeline, 5-layer compaction, and more -- see **[docs/architecture.md](./docs/architecture.md)**.

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>Values and Design Principles</h2></summary>

The architecture traces from **5 human values** through **13 design principles** to implementation:

| Value | Core Idea |
|:------|:----------|
| **Human Decision Authority** | Humans retain control via principal hierarchy. When a 93% prompt-approval rate revealed approval fatigue, response was restructured boundaries, not more warnings. |
| **Safety, Security, Privacy** | System protects even when human vigilance lapses. 7 safety mechanisms. |
| **Reliable Execution** | Does what was meant. Gather-act-verify loop. Graceful recovery. |
| **Capability Amplification** | "A Unix utility, not a product." 98.4% is deterministic infrastructure enabling the model. |
| **Contextual Adaptability** | CLAUDE.md hierarchy, graduated extensibility, trust trajectories that evolve over time. |

<details>
<summary><b>The 13 Design Principles</b></summary>

| Principle | Design Question |
|:----------|:----------------|
| Deny-first with human escalation | Should unrecognized actions be allowed, blocked, or escalated? |
| Graduated trust spectrum | Fixed permission level, or spectrum users traverse over time? |
| Defense in depth | Single safety boundary, or multiple overlapping ones? |
| Externalized programmable policy | Hardcoded policy, or externalized configs with lifecycle hooks? |
| Context as scarce resource | Single-pass truncation or graduated pipeline? |
| Append-only durable state | Mutable state, snapshots, or append-only logs? |
| Minimal scaffolding, maximal harness | Invest in scaffolding or operational infrastructure? |
| Values over rules | Rigid procedures or contextual judgment with deterministic guardrails? |
| Composable multi-mechanism extensibility | One API or layered mechanisms at different costs? |
| Reversibility-weighted risk assessment | Same oversight for all, or lighter for reversible actions? |
| Transparent file-based config and memory | Opaque DB, embeddings, or user-visible files? |
| Isolated subagent boundaries | Shared context/permissions, or isolation? |
| Graceful recovery and resilience | Fail hard, or recover silently? |

</details>

The paper also applies a **sixth evaluative lens** -- long-term capability preservation -- citing evidence that developers in AI-assisted conditions score 17% lower on comprehension tests.

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>The Agentic Query Loop</h2></summary>

<p align="center">
  <img src="./assets/iteration.png" width="60%" alt="Runtime turn flow">
</p>

The core is a **ReAct-pattern while-loop**: assemble context → call model → dispatch tools → check permissions → execute → repeat. Implemented as an `AsyncGenerator` yielding streaming events.

**Before a model call**, context management considers five stages in order: Budget Reduction → Snip → Microcompact → Context Collapse → Auto-Compact. Feature gates, settings, and thresholds determine which stages change the context.

**9-step pipeline per turn:** Settings resolution → State init → Context assembly → Conditional context management → Model call → Tool dispatch → Permission gate → Tool execution → Stop condition

**Two execution paths:**
- `StreamingToolExecutor` -- begins executing tools as they stream in (latency optimization)
- Fallback `runTools` -- classifies tools as concurrent-safe or exclusive

**Recovery:** Max output token escalation (3 retries), reactive compaction (once per turn), prompt-too-long handling, streaming fallback, fallback model

**5 stop conditions:** No tool use, max turns, context overflow, hook intervention, explicit abort

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>Safety and Permissions</h2></summary>

<p align="center">
  <img src="./assets/permission.png" width="75%" alt="Permission gate">
</p>

**7 permission modes** form a graduated trust spectrum: `plan` → `default` → `acceptEdits` → `auto` (ML classifier) → `dontAsk` → `bypassPermissions` (+ internal `bubble`).

**Deny-first**: A broad deny overrides a narrow allow in rule evaluation. **7 safety layers** cover tool filtering, permission decisions, hooks, and sandboxing. In v2.1.88, the session bypass flag and computer-use app allowlist are not restored on resume; these temporary states are separate from persistent permission settings.

> [!WARNING]
> **Bounded analysis:** The legacy parser limits per-subcommand analysis to avoid blocking the event loop. If that path produces more than 50 subcommands, it returns `ask`. This fallback requests an approval decision; it does not automatically allow the command.

<details>
<summary><b>More details: authorization pipeline, auto-mode classifier, CVEs</b></summary>

**Authorization pipeline:** Pre-filtering (strip denied tools) → PreToolUse hooks → Deny-first rule evaluation → Permission handler (4 branches: coordinator, swarm worker, speculative classifier, interactive)

**Auto-mode classifier** (`yoloClassifier.ts`): Separate LLM call with internal/external permission templates. Two-stage: fast-filter + chain-of-thought.

**Pre-trust execution window:** 2 patched CVEs share this root cause -- hooks and MCP servers execute during initialization *before* the trust dialog appears, creating a structurally privileged attack window outside the deny-first pipeline.

</details>

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>Extensibility</h2></summary>

<p align="center">
  <img src="./assets/extensibility.png" width="85%" alt="Three injection points: assemble, model, execute">
</p>

**Four extension mechanisms:** Hooks, Skills, Plugins, and MCP. Their context cost depends on the instructions, definitions, and results exposed to the model. They act at three points in the loop: **assemble()** (what the model sees), **model()** (which tools it can call), and **execute()** (whether and how actions run).

**Tool pool assembly** (5-step): Base enumeration (up to 54 tools) → Mode filtering → Deny pre-filtering → MCP integration → Deduplication

**27 hook events** across 5 categories with 4 execution types (shell, LLM-evaluated, webhook, subagent verifier)

**Plugin manifest** accepts 10 component types: commands, agents, skills, hooks, MCP servers, LSP servers, output styles, channels, settings, user config

**Skills:** SKILL.md with 15+ YAML frontmatter fields. Key difference -- SkillTool injects into current context; AgentTool spawns isolated context.

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>Context and Memory</h2></summary>

<p align="center">
  <img src="./assets/context.png" width="95%" alt="Context construction">
</p>

**9 ordered sources** build the context window. CLAUDE.md instructions enter as **user context**. Prompt instructions guide model behavior; permission checks and sandboxes enforce access limits outside the model. Memory in this version is **file-based**, so its contents can be inspected, edited, and versioned.

**4-level CLAUDE.md hierarchy:** Managed (`/etc/`) → User (`~/.claude/`) → Project (`CLAUDE.md`, `.claude/rules/`) → Local (`CLAUDE.local.md`, gitignored)

**Five context-management stages**, each with its own conditions: Budget reduction → Snip → Microcompact → Context Collapse (a non-destructive read-time projection) → Auto-Compact (model-generated summarization).

**Memory retrieval:** LLM-based scan of memory-file headers, selects up to 5 relevant files. No embeddings, no vector similarity.

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>Subagent Delegation</h2></summary>

<p align="center">
  <img src="./assets/subagent.png" width="90%" alt="Subagent architecture">
</p>

**6 built-in types** (Explore, Plan, General-purpose, Guide, Verification, Statusline) + custom agents via `.claude/agents/*.md`. Sidechain transcripts keep worker histories separate, and concise returns can limit what enters the parent's context. Worktree, remote, and in-process modes provide different execution arrangements; context separation alone does not determine file access or authority. Coordination uses POSIX `flock()`.

**SkillTool vs AgentTool:** SkillTool adds guidance to the current context; AgentTool delegates work to another context. Token cost depends on the work performed and the results returned.

**Permission override:** Subagent `permissionMode` applies UNLESS parent is in `bypassPermissions`/`acceptEdits`/`auto` (explicit user decisions always take precedence).

**Custom agents:** YAML frontmatter supports tools, disallowedTools, model, effort, permissionMode, mcpServers, hooks, maxTurns, skills, memory scope, background flag, isolation mode.

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>Session Persistence</h2></summary>

<p align="center">
  <img src="./assets/session_compact.png" width="75%" alt="Session persistence and context compaction">
</p>

Three channels preserve session history: append-only JSONL transcripts, global prompt history, and subagent sidechains. In v2.1.88, the session bypass flag and computer-use app allowlist are not restored from the previous session. Persistent policy, session mode, and temporary grants have different lifetimes.

**Chain patching:** Compact boundaries record `headUuid`/`anchorUuid`/`tailUuid`. The session loader patches the message chain at read time. Nothing is destructively edited on disk.

**Checkpoints:** File-history checkpoints for `--rewind-files`, stored at `~/.claude/file-history/<sessionId>/`.

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>New Signals in the Agent Design Space</h2></summary>

Claude Code gives us a concrete starting point: a model chooses actions inside a system that manages tools, context, permissions, and recovery. More recent work connects those choices across a longer task: how work is organized, what survives interruption, which experience is reused, and what evidence is needed to move on.

| Design Implication | What it means for agent builders | Representative signals |
|:---|:---|:---|
| **Graph Engineering makes relationships explicit** | Express task dependencies, agent responsibilities, and runtime state separately. Graph nodes can still contain agents that choose their own actions. | [Graph Engineering survey](https://arxiv.org/abs/2608.21156), [Agent Graph](https://github.com/context4ai/agent-graph/tree/387f80db65bf20a61bc666b4fa885200fcedad08) |
| **Long tasks need explicit runtime state** | Track goals, sessions, workers, checkpoints, and result delivery explicitly. Recovery needs a clear owner for each unfinished action. | [Cursor cloud agents](https://cursor.com/blog/cloud-agent-lessons), [Google Managed Agents](https://blog.google/innovation-and-ai/technology/developers-tools/managed-agents-gemini-api/), [Microsoft Agent 365](https://www.microsoft.com/en-us/security/blog/2026/05/01/microsoft-agent-365-now-generally-available-expands-capabilities-and-integrations/), [Databricks Omnigent](https://www.databricks.com/blog/introducing-omnigent-meta-harness-combine-control-and-share-your-agents) |
| **Context and memory need lifecycle rules** | Manage the working context and reusable memory according to their different lifetimes. Keep the source, scope, update rules, and retrieval cost of remembered experience visible. | [LangChain Context Hub](https://www.langchain.com/blog/introducing-context-hub), [AWS AgentCore](https://aws.amazon.com/blogs/machine-learning/break-the-context-window-barrier-with-amazon-bedrock-agentcore/), [Anthropic managed-agent memory](https://platform.claude.com/docs/en/managed-agents/memory) |
| **Access limits need execution boundaries** | Permissions, network reachability, filesystem access, credential custody, tenant isolation, and sandboxing belong in the architecture. | [Codex Windows sandbox](https://openai.com/index/building-codex-windows-sandbox/), [Running Codex safely](https://openai.com/index/running-codex-safely/), [Anthropic self-hosted sandboxes](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes) |
| **Tools and skills are a supply chain** | Manage the sources and versions of tools, skills, and plugins. Define installation policy, update policy, server identity, and action approvals separately. | [NSA MCP security](https://www.nsa.gov/Portals/75/documents/Cybersecurity/CSI_MCP_SECURITY.pdf), [GitHub MCP allowlists](https://github.blog/changelog/2026-04-16-copilot-cli-supports-custom-registry-based-mcp-allowlists/), [A2A milestone](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year) |
| **Humans need control over the whole run** | Support goals, plans, approvals, interrupts, reviewable diffs, escalation, and explicit ownership when several agents can write. | [Codex from anywhere](https://openai.com/index/work-with-codex-from-anywhere/), [Copilot cloud agent](https://github.blog/changelog/2026-04-01-research-plan-and-code-with-copilot-cloud-agent), [Cognition multi-agents](https://cognition.ai/blog/multi-agents-working) |
| **Control decisions need their own evaluation** | Evaluate when an agent should continue, verify, ask for help, or stop, as well as the worker's ability to execute a step. | [LoopArena](https://arxiv.org/abs/2608.28281) |
| **Observability must close the improvement loop** | Turn traces into testable failure hypotheses. Check proposed changes on relevant tasks, protect prior successes with regression tests, and confirm gains on fresh cases. | [LangSmith Engine](https://www.langchain.com/blog/how-we-built-langsmith-engine-our-agent-for-improving-agents), [OpenAI agent improvement loop](https://developers.openai.com/cookbook/examples/agents_sdk/agent_improvement_loop), [AWS AgentCore Evaluations](https://aws.amazon.com/blogs/machine-learning/build-reliable-ai-agents-with-amazon-bedrock-agentcore-evaluations/) |

The [Graph Engineering survey](https://arxiv.org/abs/2608.21156) organizes this discussion around task graphs, agent graphs, and runtime state graphs. This is the authors' proposed framework, with earlier graph-based systems as precedents. It connects naturally to the guide's decisions about reasoning, delegation, and persistence.

[Codex local memories](https://learn.chatgpt.com/docs/customization/memories) carry selected experience between sessions when enabled. Working-context management is a separate choice: [v0.153.4 retains summarizing compaction](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact.rs), while its [experimental context-management mode](https://learn.chatgpt.com/docs/config-file/config-reference) uses notes, searchable history, and fresh context windows. The latter changes how a long task continues across windows; it is distinct from background experience extraction across sessions.

The experimental switch was [released in Codex CLI 0.153.0](https://github.com/openai/codex/releases/tag/rust-v0.153.0) and is off by default, with account and backend restrictions. A September 6 source change adds a model-support check and marks the bundled GPT-6-Astra model as supported. The [source notes](./docs/agent-design-space-source-notes.md#source-codex-context-management) distinguish this later source revision from v0.153.4. This is a change in the context-management protocol and its model integration; the public evidence does not establish a new form of permanent memory inside the model.

The [design guide](./docs/build-your-own-agent.md) develops these choices. [Source notes](./docs/agent-design-space-source-notes.md#september-2026) record the reviewed versions, publication dates, and limits of the evidence.

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>Build Your Own AI Agent: A Design Guide</h2></summary>

> A guide to the design decisions behind an agent, drawing on Claude Code's architecture and the wider systems discussed here.

Every production agent must navigate these decisions:

| Decision | The Question | Key Insight |
|:---------|:-------------|:------------|
| [**Reasoning placement**](./docs/build-your-own-agent.md#decision-1-where-does-reasoning-live) | How much logic in the model vs. harness? | Choose which dependencies and checks belong in code, and which decisions remain with the model. |
| [**Safety posture**](./docs/build-your-own-agent.md#decision-2-what-is-your-safety-posture) | How do you prevent harmful actions? | Give each control a clear scope and define what happens when a check cannot decide. |
| [**Context management**](./docs/build-your-own-agent.md#decision-3-how-do-you-manage-context) | What does the model see? | Separate the current conversation budget from the lifecycle of reusable experience. |
| [**Extensibility**](./docs/build-your-own-agent.md#decision-4-how-do-you-handle-extensibility) | How do extensions plug in? | Treat discovery, installation, updates, and execution authority as separate choices. |
| [**Subagent architecture**](./docs/build-your-own-agent.md#decision-5-how-do-subagents-work) | Shared or isolated context? | Define ownership, context boundaries, and the evidence each worker returns. |
| [**Session persistence**](./docs/build-your-own-agent.md#decision-6-how-do-sessions-persist) | What carries over? | Restore task state under current policy, with explicit lifetimes for temporary grants. |

**Read the full guide: [docs/build-your-own-agent.md](./docs/build-your-own-agent.md)**

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>Cross-System Comparison: Claude Code vs OpenClaw vs Hermes-Agent</h2></summary>

The same recurring design questions admit different architectural answers when the deployment context changes. The table below contrasts Claude Code v2.1.88 with two notable peers — [OpenClaw](https://github.com/openclaw/openclaw), a local-first multi-channel personal-assistant gateway, and [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent), a self-improving multi-deployment agent — across the six design dimensions Section 10 of the paper uses for the OpenClaw comparison. Cells are source-grounded; this is not a feature scoreboard.

| Design Dimension | Claude Code (v2.1.88) [![Star](https://img.shields.io/github/stars/anthropics/claude-code.svg?style=social&label=Star)](https://github.com/anthropics/claude-code) | OpenClaw [![Star](https://img.shields.io/github/stars/openclaw/openclaw.svg?style=social&label=Star)](https://github.com/openclaw/openclaw) | Hermes-Agent [![Star](https://img.shields.io/github/stars/NousResearch/hermes-agent.svg?style=social&label=Star)](https://github.com/NousResearch/hermes-agent) |
|:---|:---|:---|:---|
| **System scope & deployment** | Per-user CLI / SDK / IDE interface for coding; one `queryLoop` async generator across entry points. | Local-first WebSocket gateway (default port 18789, loopback-bound by default; other binds available); routes ~23 messaging channels to an embedded agent runtime; companion apps for macOS, iOS, Android. | Three entry points: `hermes` (interactive CLI), `hermes-agent` (programmatic runtime), `hermes-acp` (ACP server); gateway adapters route messages to per-session AIAgent instances cached LRU-style (max 128, 1 h idle TTL); also runs as MCP server via `hermes mcp serve`. |
| **Trust model & security** | Deny-first per-action evaluation; 7 permission modes; LLM-based auto-mode classifier (`yoloClassifier` / `sideQuery`); session-scoped permission state (session bypass flag, app allowlist state) is not restored on resume. | Single trusted operator per gateway; DM pairing codes, sender allowlists, gateway authentication; per-agent allow / deny tool policy; opt-in sandboxing via Docker / SSH / OpenShell, off by default; `non-main` mode sandboxes only non-main sessions; hostile multi-tenant isolation explicitly not supported. | Dangerous-command pattern detection with per-session approval state; CLI interactive prompts and gateway async prompts; auxiliary-LLM smart approval auto-approves low-risk commands; permanent allowlist persisted in `config.yaml`; subagent worker threads default to auto-deny dangerous commands (opt-in `subagent_auto_approve` for batch / cron runs). |
| **Agent runtime & tools** | Single `queryLoop` async generator with streamed event yields; environment- and feature-gated tool registry; before-API compaction (Snip, Microcompact, Context Collapse, Auto-Compact) runs conditionally, with Auto-Compact first attempting session-memory compaction. | Embedded agent runtime inside the gateway's RPC dispatch (the `agent` RPC validates parameters, accepts immediately, runs asynchronously, and streams lifecycle / stream events back over the gateway protocol); per-session queue serialization with an optional global lane. | While-loop with explicit per-turn iteration budget and grace-call slot; per-turn checkpoint dedup; gateway `step_callback` hook fires on each iteration; auxiliary-model context compression summarizes middle turns while protecting head and tail. |
| **Extension architecture** | Four extension mechanisms: hooks, skills, plugins, and MCP; context cost depends on exposed content; 27 hook events; 10 plugin component types. | Manifest-first plugin system with 12 documented capability categories; central registry exposes tools, channels, provider setup, hooks, HTTP routes, CLI commands, services; separate skills layer with multiple sources (workspace highest precedence) plus the ClawHub public registry; `openclaw mcp` provides both an MCP server interface and an outbound client registry for other MCP servers. | 12 bundled plugins under `plugins/` (context_engine, disk-cleanup, example-dashboard, google_meet, hermes-achievements, image_gen, kanban, memory, observability, platforms, spotify, strike-freedom-cockpit); MCP server (`mcp_serve.py`) exposes 10 tools; ACP adapter (`acp_adapter/`) exposes Hermes as an ACP server. |
| **Memory & context** | 4-level CLAUDE.md hierarchy; before-API compaction (Snip, Microcompact, Context Collapse, Auto-Compact); LLM-based selection from file-based Markdown memory files. | Workspace bootstrap files (AGENTS.md, SOUL.md, TOOLS.md, IDENTITY.md, USER.md) plus conditional BOOTSTRAP.md / HEARTBEAT.md / MEMORY.md; separate memory system (MEMORY.md, daily notes under `memory/YYYY-MM-DD.md`, optional DREAMS.md); hybrid vector + keyword search when an embedding provider is configured; experimental dreaming for long-term promotion; pluggable compaction providers. | SQLite state store with FTS5 full-text search and WAL-mode concurrent readers; sessions linked by `parent_session_id` chains for compression-triggered splits; 8 swappable memory backends under `plugins/memory/` (byterover, hindsight, holographic, honcho, mem0, openviking, retaindb, supermemory); auxiliary-LLM compression as a separate context-management layer. |
| **Multi-agent architecture** | Sub-agent delegation via sidechain transcripts; 6 built-in agent definitions (availability conditional on build / mode) plus custom; a single summary message returns to parent (in-process / viewable transcript cases preserve more internal detail); agent-isolation settings include `worktree` and `remote`, with an `in-process` teammate backend in the swarm path. | Two layers. (1) Multi-agent routing: per-channel isolated agents with their own workspace, auth profiles, session store, and model configuration, dispatched via deterministic binding rules. (2) Sub-agent delegation: `maxSpawnDepth` range 1–5, default 1, recommended 2; tool policy varies by depth; project vision (VISION.md) rejects agent-hierarchy frameworks as the default. | `delegate_task` tool spawns child AIAgent instances in a `ThreadPoolExecutor` (parent blocks until children complete); each child has fresh conversation history, its own `task_id`, and a restricted toolset (`DELEGATE_BLOCKED_TOOLS` strips `delegate_task`, `clarify`, `memory`, `send_message`, `execute_code`); default depth `MAX_DEPTH = 1` (configurable up to cap 3); default 3 concurrent children. |

**What this contrast reveals.** Three observations follow from the table. First, **deployment context** drives the rest of the design: a per-user coding CLI converges on per-action approval and a single execution loop, a multi-channel gateway converges on perimeter trust and channel-bound agents, and a multi-deployment messaging-and-cloud agent converges on opt-in container/cloud isolation, an LLM-based smart approval, and a swappable-backend memory layer. Second, the **extension layer is where each system most clearly differentiates**: Claude Code provides four distinct extension mechanisms, OpenClaw treats extension as registry-managed capabilities at the gateway, and Hermes-Agent ships bundled plugins plus dual MCP server / ACP server interfaces other agents can connect to. Third, **memory architectures sit on a spectrum**: file-based and inspectable Markdown (Claude Code), file-based plus optional vector + experimental dreaming (OpenClaw), or full-text indexed (FTS5) plus eight swappable plugin backends including dedicated vector / RAG providers (Hermes-Agent). The table is best read not as a scoreboard but as three different fixed points in the same design space.

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

## Find Resources by Design Question

The sections above give Claude Code's own answer to each design question. The catalogs below are organized by resource type instead. This table joins the two, so you can start from a question rather than from a resource type.

| Design question | Claude Code's answer | Where the outside resources are |
|:---|:---|:---|
| **How does one turn actually run?** Agent loop, tool dispatch, recovery. | [The Agentic Query Loop](#the-agentic-query-loop) | [Architecture Analysis](#architecture-analysis) · [Open-Source Reimplementations](#open-source-reimplementations) · [Coding Agent CLIs](#coding-agent-clis-and-ide-harnesses) · [Harness Engineering](#general-harness-engineering-design-space-resources) |
| **Who is allowed to do what?** Permissions, approval, sandboxing. | [Safety and Permissions](#safety-and-permissions) | [Security Research & Incidents](#security-research--incidents) · [Runtime & Sandbox Infrastructure](#runtime--sandbox-infrastructure) · [Product Documentation](#product-documentation) · [Academic Papers](#related-academic-papers) |
| **How is the system extended?** Hooks, skills, plugins, MCP. | [Extensibility](#extensibility) | [Skills and Harness Extensions](#skills-and-harness-extensions) · [MCP Ecosystem](#mcp-ecosystem) · [Product Documentation](#product-documentation) |
| **What does the model see?** Context assembly, memory, compaction. | [Context and Memory](#context-and-memory) | [Memory and Persistent Context](#memory-and-persistent-context) · [Blog Posts & Technical Articles](#blog-posts--technical-articles) · [Academic Papers](#related-academic-papers) |
| **How is work divided?** Subagents, teams, orchestration. | [Subagent Delegation](#subagent-delegation) | [Agent Frameworks and Orchestration](#agent-frameworks-and-orchestration) · [Cross-Vendor Engineering](#cross-vendor-code-agent-engineering) · [Cross-System Comparison](#cross-system-comparison-claude-code-vs-openclaw-vs-hermes-agent) |
| **What survives a restart?** Sessions, checkpoints, persistence. | [Session Persistence](#session-persistence) | [Runtime & Sandbox Infrastructure](#runtime--sandbox-infrastructure) · [Cross-Vendor Engineering](#cross-vendor-code-agent-engineering) |
| **How do you know it worked?** Evaluation, benchmarks, trace analysis. | [New Signals: closing the improvement loop](#new-signals-in-the-agent-design-space) | [Evaluation & Benchmarks](#evaluation--benchmarks) · [Academic Papers](#related-academic-papers) |

For shifts that cut across every axis rather than sitting on one, see [New Signals in the Agent Design Space](#new-signals-in-the-agent-design-space).

---

<details>
<summary><h2>Community Projects & Research</h2></summary>

A curated map of the repos, reimplementations, and academic papers surrounding Claude Code's architecture.

### Official Anthropic Resources

Primary sources referenced throughout the paper — Anthropic's own engineering and research publications, plus product documentation.

#### Research & Engineering Blogs

| Article | Topic |
|:--------|:------|
| [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) | Foundational: simple composable patterns over heavy frameworks. |
| [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | Context curation and token-budget management. |
| [The New Rules of Context Engineering for Claude 5 Generation Models](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models) | Anthropic removed over 80% of Claude Code's system prompt for newer models with no measurable loss on its coding evals. The post recommends replacing blanket rules with model judgment, tool-use examples with clearer interfaces, and up-front context loading with skills and tool definitions loaded on demand. Its broader point is that a harness should be reviewed and pruned as models change, rather than only accumulating instructions. |
| [A Harness for Every Task: Dynamic Workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code) | Anthropic's design account of programmatic subagent orchestration. It names six reusable patterns — classify-and-act, fan-out-and-synthesize, adversarial verification, generate-and-filter, tournament, and loop-until-done — and moves coordination state from one model context into a script that can run and re-run many isolated agents. |
| [Prompt Caching with Claude](https://www.anthropic.com/news/prompt-caching) | Cache reads at 10% cost, writes at 125%; 5-min default TTL. The platform feature that makes Claude Code's cache-aware compaction architecturally meaningful. |
| [Harness Design for Long-Running Application Development](https://anthropic.com/engineering/harness-design-long-running-apps) | Harness architecture for autonomous full-stack dev; multi-agent patterns. |
| [Claude Code Auto Mode: A Safer Way to Skip Permissions](https://www.anthropic.com/engineering/claude-code-auto-mode) | ML-classifier approval automation; source of the 93% approval-rate finding. |
| [Auto Mode Is Now the Default in Claude Code](https://claude.com/blog/auto-mode-default-in-claude-code) | Auto mode becomes the default on Pro, Max, and Team plans (announced August 7, 2026, with rollout completed August 14), while Enterprise and the API surfaces (the Claude API, Bedrock, Vertex, and Foundry) stay opt-in. A 1,053-tester study is the argument: manual review caught 13.6% of injected dangerous commands against 89% for the classifier, and the harmful-action rate fell from 6.3% of manually approved sessions to 2.4%. The successor milestone to the auto-mode engineering post above, now a default rather than a setting. |
| [Beyond Permission Prompts: Making Claude Code More Secure and Autonomous](https://www.anthropic.com/engineering/claude-code-sandboxing) | Sandbox-based security; 84% reduction in permission prompts. |
| [How We Contain Claude Across Products](https://www.anthropic.com/engineering/how-we-contain-claude) | Containment across claude.ai, Claude Code, and Cowork (May 2026); Claude Code's human-in-the-loop sandbox, approval fatigue, and capping the blast radius. |
| [Measuring AI Agent Autonomy in Practice](https://anthropic.com/research/measuring-agent-autonomy) | Longitudinal usage: auto-approve rates grow from ~20% to 40%+ with experience. |
| [Agentic Coding and Persistent Returns to Expertise](https://www.anthropic.com/research/claude-code-expertise) | Anthropic's first quantitative account of how the work actually splits: humans make roughly 70% of planning decisions but only 20% of execution decisions, and a single prompt triggers about 10 Claude actions on average, with some stretches running past 100 actions between human touchpoints. An empirical anchor for the paper's claims about human control surfaces and supervision cost. |
| [Our Framework for Developing Safe and Trustworthy Agents](https://www.anthropic.com/news/our-framework-for-developing-safe-and-trustworthy-agents) | Governance framework for responsible agent deployment. |
| [When AI Builds Itself](https://www.anthropic.com/institute/recursive-self-improvement) | Anthropic Institute on recursive self-improvement: AI accelerating AI development, the direction-setting and research-taste gaps, and governance scenarios. |
| [Scaling Managed Agents: Decoupling the Brain from the Hands](https://www.anthropic.com/engineering/managed-agents) | Hosted-service architecture separating reasoning, execution, and session. |
| [An Update on Recent Claude Code Quality Reports](https://www.anthropic.com/engineering/april-23-postmortem) | Postmortem on three bugs behind perceived quality drops: a reasoning-effort default, a cache optimization bug, and a system-prompt change. |
| [Introducing Claude Opus 4.8](https://www.anthropic.com/news/claude-opus-4-8) | May 2026 model update: sharper judgment and honesty (~4x fewer unremarked code flaws), longer autonomous runs; introduces dynamic workflows in research preview. |
| [Claude Fable 5 and Claude Mythos 5](https://www.anthropic.com/news/claude-fable-5-mythos-5) | June 2026 Mythos-class tier sitting above Opus; Fable 5 is the general-use configuration (risky queries fall back to Opus 4.8), with state-of-the-art software-engineering and agentic-coding performance. Access was suspended globally on June 12, 2026 (see next row). |
| [Statement on Suspending Access to Fable 5 and Mythos 5](https://www.anthropic.com/news/fable-mythos-access) | Anthropic's statement on suspending Fable 5 and Mythos 5. A US export-control directive (June 12, 2026) restricted access for foreign nationals, but Anthropic disabled both models for all users worldwide, just days after launch. A rare case of regulation forcing a deployed frontier model offline, and a concrete example of the compliance and safety pressures that agent systems face in deployment. |

#### Product Documentation

| Document | Topic |
|:---------|:------|
| [How Claude Code Works](https://code.claude.com/docs/en/how-claude-code-works) | Official overview of the agent loop, tools, and terminal automation. |
| [Permissions](https://code.claude.com/docs/en/permissions) | Tiered permission system, modes, granular rules. |
| [Hooks](https://code.claude.com/docs/en/hooks) | 27-event hook reference, execution models, lifecycle events. |
| [Memory](https://code.claude.com/docs/en/memory) | CLAUDE.md hierarchy, auto memory, learned preferences. |
| [Sub-agents](https://code.claude.com/docs/en/sub-agents) | Specialized isolated assistants, custom prompts, tool access. |
| [Orchestrate Subagents at Scale with Dynamic Workflows](https://code.claude.com/docs/en/workflows) | Claude writes a JavaScript orchestration script; a background runtime fans out to up to 16 concurrent and 1,000 total subagents, with intermediate state held in script variables outside the context window (v2.1.154+). The permission design is the part worth noting: the launch prompt follows your permission mode, but the subagents a workflow spawns always run in `acceptEdits` and inherit the session tool allowlist regardless of that mode, so file edits are auto-approved inside a run. |
| [Configure Auto Mode](https://code.claude.com/docs/en/auto-mode-config) | The fullest official account of the auto-mode classifier as a policy engine: a four-tier precedence (`hard_deny` over `soft_deny` over `allow` over explicit user intent), rules written as natural-language prose rather than regexes, and one deliberate exclusion. The classifier reads CLAUDE.md but never the shared `.claude/settings.json`, so a checked-in repo config cannot grant itself permissions. |
| [Orchestrate Teams of Claude Code Sessions](https://code.claude.com/docs/en/agent-teams) | Agent teams as a primitive distinct from subagents: teammates hold separate contexts, message each other through mailboxes, and share a file-locked task list with dependencies and self-claiming. The trust boundary is explicit: a teammate cannot approve a permission prompt on the user's behalf, and a denied action cannot be handed to another teammate to get around it. |
| [What's New in Claude Opus 4.8](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-8) | Mid-conversation system messages (prompt-cache-preserving), lower cacheable-prompt minimum, fewer compactions and better compaction recovery. |
| [What's New in Claude Sonnet 5](https://platform.claude.com/docs/en/about-claude/models/whats-new-sonnet-5) | The model takes back two dials the harness used to hold: manual extended thinking is gone (`thinking: {budget_tokens: N}` now returns 400) and non-default `temperature`/`top_p`/`top_k` also return 400. A new tokenizer yields roughly 30% more tokens for the same text, so the usable context budget shifts with the model. |
| [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) | Release notes. Dynamic workflows and Opus 4.8 land in v2.1.154. Across v2.1.178 to v2.1.207 (June to July 2026): subagents run in the background by default with `agent_needs_input`/`agent_completed` hooks, and an agent's message is still never treated as the user's approval; the default permission mode is renamed to Manual; a `Tool(param:value)` rule syntax arrives (for example `Agent(model:opus)`); auto mode stops reading a repo's `.claude/settings.local.json`; and `sandbox.credentials` blocks sandboxed commands from reading credential files and secret environment variables. Through v2.1.215 (July 2026): v2.1.212 adds per-session caps on WebSearch calls and on subagent spawns (default 200 each, tunable by environment variable), moves MCP tool calls running past two minutes to the background automatically, turns `/fork` into a background session with the in-session behavior split out as `/subtask`, deprecates the Task tool's `mode` parameter so subagents inherit the parent session's permission mode, and stops plan mode auto-running file-modifying Bash commands without a prompt. v2.1.214 adds the EndConversation tool, requires permission for `docker` commands carrying daemon-redirect flags, and fixes a permission-scoping bug where a single-segment rule like `Edit(src/**)` auto-approved writes to a nested `src/` anywhere in the tree instead of only `<cwd>/src`. The v2.1.221–v2.1.224 cluster (August 2026) adds masked credential files, cross-machine `SendMessage`/`ListAgents`, and `claude self-hosted-runner`; routes inbound cross-session messages through permission review; and closes worktree, dynamic-import, policy-bypass, hidden-command, and trailing-slash `denyRead` escapes. Credential masking also expands to structured extraction, JWT claims, and AWS request re-signing, while sandbox violations are returned to the model as recovery feedback. The v2.1.225 to v2.1.233 run (August 2026) makes auto mode the default on Pro, Max, and Team plans, turns subagent forking on by default so a `fork` subagent inherits the parent context and prompt cache, matures cross-session messaging with `@name` mentions and inbound accept and hold controls, removes the todo and task-tracking tools by default on newer models (`CLAUDE_CODE_ENABLE_TODO_TOOLS=1` restores them), and closes a Windows NT device-prefix path-validation bypass that had leaked NTLM credentials. |

### Architecture Analysis

Deep dives into Claude Code's internal design.

| Repository | Description |
|:-----------|:------------|
| [**ComeOnOliver/claude-code-analysis**](https://github.com/ComeOnOliver/claude-code-analysis) [![Star](https://img.shields.io/github/stars/ComeOnOliver/claude-code-analysis.svg?style=social&label=Star)](https://github.com/ComeOnOliver/claude-code-analysis) | Comprehensive reverse-engineering: source tree structure, module boundaries, tool inventories, and architectural patterns. |
| [**alejandrobalderas/claude-code-from-source**](https://github.com/alejandrobalderas/claude-code-from-source) [![Star](https://img.shields.io/github/stars/alejandrobalderas/claude-code-from-source.svg?style=social&label=Star)](https://github.com/alejandrobalderas/claude-code-from-source) | 18-chapter technical book (~400 pages). All original pseudocode, no proprietary source. |
| [**liuup/claude-code-analysis**](https://github.com/liuup/claude-code-analysis) [![Star](https://img.shields.io/github/stars/liuup/claude-code-analysis.svg?style=social&label=Star)](https://github.com/liuup/claude-code-analysis) | Chinese-language deep-dive — startup flow, query main loop, MCP integration, multi-agent architecture. |
| [**sanbuphy/claude-code-source-code**](https://github.com/sanbuphy/claude-code-source-code) [![Star](https://img.shields.io/github/stars/sanbuphy/claude-code-source-code.svg?style=social&label=Star)](https://github.com/sanbuphy/claude-code-source-code) | Quadrilingual analysis (EN/JA/KO/ZH) — multi-domain reports covering telemetry, codenames, KAIROS, unreleased tools. |
| [**cablate/claude-code-research**](https://github.com/cablate/claude-code-research) [![Star](https://img.shields.io/github/stars/cablate/claude-code-research.svg?style=social&label=Star)](https://github.com/cablate/claude-code-research) | Independent research on internals, Agent SDK, and related tooling. |
| [**Yuyz0112/claude-code-reverse**](https://github.com/Yuyz0112/claude-code-reverse) [![Star](https://img.shields.io/github/stars/Yuyz0112/claude-code-reverse.svg?style=social&label=Star)](https://github.com/Yuyz0112/claude-code-reverse) | Visualize Claude Code's LLM interactions — log parser and visual tool to trace prompts, tool calls, and compaction. |
| [**Piebald-AI/claude-code-system-prompts**](https://github.com/Piebald-AI/claude-code-system-prompts) [![Star](https://img.shields.io/github/stars/Piebald-AI/claude-code-system-prompts.svg?style=social&label=Star)](https://github.com/Piebald-AI/claude-code-system-prompts) | Version-tracked corpus of Claude Code's internal prompts, extracted from each release's published package: the main system prompt, builtin tool descriptions, sub-agent prompts (Plan/Explore/Task), slash-command prompts, and system-reminder injections, with a changelog spanning every tracked release since v2.0.14. Updated within minutes of each release. |
| [**AgiFlow/claude-code-prompt-analysis**](https://github.com/AgiFlow/claude-code-prompt-analysis) [![Star](https://img.shields.io/github/stars/AgiFlow/claude-code-prompt-analysis.svg?style=social&label=Star)](https://github.com/AgiFlow/claude-code-prompt-analysis) | API request and response logs captured across five conversation sessions. Empirical methodology with reproducible data rather than static code reading. |

### Open-Source Reimplementations

Clean-room rewrites and buildable research forks.

| Repository | Description |
|:-----------|:------------|
| [**chauncygu/collection-claude-code-source-code**](https://github.com/chauncygu/collection-claude-code-source-code) [![Star](https://img.shields.io/github/stars/chauncygu/collection-claude-code-source-code.svg?style=social&label=Star)](https://github.com/chauncygu/collection-claude-code-source-code) | Meta-collection of community Claude Code source artifacts -- includes claw-code (Rust port), nano-claude-code (Python), and the extracted original source archive. |
| [**777genius/claude-code-working**](https://github.com/777genius/claude-code-working) [![Star](https://img.shields.io/github/stars/777genius/claude-code-working.svg?style=social&label=Star)](https://github.com/777genius/claude-code-working) | Working reverse-engineered CLI. Runnable with Bun, 450+ chunk files, 31 feature flags polyfilled. |
| [**T-Lab-CUHKSZ/claude-code**](https://github.com/T-Lab-CUHKSZ/claude-code) [![Star](https://img.shields.io/github/stars/T-Lab-CUHKSZ/claude-code.svg?style=social&label=Star)](https://github.com/T-Lab-CUHKSZ/claude-code) | CUHK-Shenzhen buildable research fork — reconstructed build system from raw TypeScript snapshot. |
| [**ruvnet/open-claude-code**](https://github.com/ruvnet/open-claude-code) [![Star](https://img.shields.io/github/stars/ruvnet/open-claude-code.svg?style=social&label=Star)](https://github.com/ruvnet/open-claude-code) | Nightly auto-decompile rebuild — 903+ tests, 25 tools, 4 MCP transports, 6 permission modes. |
| [**Enderfga/openclaw-claude-code**](https://github.com/Enderfga/openclaw-claude-code) [![Star](https://img.shields.io/github/stars/Enderfga/openclaw-claude-code.svg?style=social&label=Star)](https://github.com/Enderfga/openclaw-claude-code) | OpenClaw plugin — unified ISession interface for Claude/Codex/Gemini/Cursor. Multi-agent council. |
| [**memaxo/claude_code_re**](https://github.com/memaxo/claude_code_re) [![Star](https://img.shields.io/github/stars/memaxo/claude_code_re.svg?style=social&label=Star)](https://github.com/memaxo/claude_code_re) | Reverse engineering from minified bundles — deobfuscation of the publicly distributed cli.js file. |
| [**agentforce314/clawcodex**](https://github.com/agentforce314/clawcodex) [![Star](https://img.shields.io/github/stars/agentforce314/clawcodex.svg?style=social&label=Star)](https://github.com/agentforce314/clawcodex) | Python rebuild with multi-provider LLM support. |
| [**ultraworkers/claw-code**](https://github.com/ultraworkers/claw-code) [![Star](https://img.shields.io/github/stars/ultraworkers/claw-code.svg?style=social&label=Star)](https://github.com/ultraworkers/claw-code) | Clean-room Rust reimplementation that compresses a roughly 512K-line TypeScript codebase into about 20K lines of Rust, which makes it a useful read on what is essential to the harness and what is incidental. |

### Claude Code Guides & Learning

Tutorials and hands-on learning paths for Claude Code itself.

| Repository | Description |
|:-----------|:------------|
| [**shareAI-lab/learn-claude-code**](https://github.com/shareAI-lab/learn-claude-code) [![Star](https://img.shields.io/github/stars/shareAI-lab/learn-claude-code.svg?style=social&label=Star)](https://github.com/shareAI-lab/learn-claude-code) | "Bash is all you need" — 19-chapter 0-to-1 course with runnable Python agents, web platform. ZH/EN/JA. |
| [**FlorianBruniaux/claude-code-ultimate-guide**](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) [![Star](https://img.shields.io/github/stars/FlorianBruniaux/claude-code-ultimate-guide.svg?style=social&label=Star)](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) | Beginner-to-power-user guide with production-ready templates, agentic workflow guides, and cheatsheets. |
| [**affaan-m/everything-claude-code**](https://github.com/affaan-m/everything-claude-code) [![Star](https://img.shields.io/github/stars/affaan-m/everything-claude-code.svg?style=social&label=Star)](https://github.com/affaan-m/everything-claude-code) | Agent harness optimization — skills, instincts, memory, security, and research-first development. |
| [**hesreallyhim/awesome-claude-code**](https://github.com/hesreallyhim/awesome-claude-code) [![Star](https://img.shields.io/github/stars/hesreallyhim/awesome-claude-code.svg?style=social&label=Star)](https://github.com/hesreallyhim/awesome-claude-code) | Curated list of skills, hooks, slash-commands, agent orchestrators, and plugins. |
| [**nblintao/awesome-claude-code-postleak-insights**](https://github.com/nblintao/awesome-claude-code-postleak-insights) [![Star](https://img.shields.io/github/stars/nblintao/awesome-claude-code-postleak-insights.svg?style=social&label=Star)](https://github.com/nblintao/awesome-claude-code-postleak-insights) | Curated post-leak findings, including the internal codenames (BUDDY, KAIROS, ULTRAPLAN), Undercover Mode, and AutoDream memory consolidation. |
| [**rohitg00/awesome-claude-code-toolkit**](https://github.com/rohitg00/awesome-claude-code-toolkit) [![Star](https://img.shields.io/github/stars/rohitg00/awesome-claude-code-toolkit.svg?style=social&label=Star)](https://github.com/rohitg00/awesome-claude-code-toolkit) | Bundled toolkit of agents, skills, commands, plugins, hooks, rules, templates, and MCP configurations. |

### General Harness Engineering Design Space Resources

External resources that complement this paper's design-space analysis — concept essays, curricula, and code that illuminate the harness layer as an engineering practice.

| Repository | Description |
|:-----------|:------------|
| [**deusyu/harness-engineering**](https://github.com/deusyu/harness-engineering) [![Star](https://img.shields.io/github/stars/deusyu/harness-engineering.svg?style=social&label=Star)](https://github.com/deusyu/harness-engineering) | Learning archive — original concept essays, independent thinking pieces, and curated translations of harness-engineering writing; from concept to independent practice. |
| [**walkinglabs/learn-harness-engineering**](https://github.com/walkinglabs/learn-harness-engineering) [![Star](https://img.shields.io/github/stars/walkinglabs/learn-harness-engineering.svg?style=social&label=Star)](https://github.com/walkinglabs/learn-harness-engineering) | Project-based English course with PDF coursebooks, syllabus, and capstone, organized around five harness subsystems: instructions, state, verification, scope, and session lifecycle. |
| [**china-qijizhifeng/agentic-harness-engineering**](https://github.com/china-qijizhifeng/agentic-harness-engineering) [![Star](https://img.shields.io/github/stars/china-qijizhifeng/agentic-harness-engineering.svg?style=social&label=Star)](https://github.com/china-qijizhifeng/agentic-harness-engineering) | Observability system that auto-evolves a coding agent's harness — a meta-agent reads execution traces and rewrites system prompts, tools, middleware, skills, sub-agents, and memory. |
| [**ZhangHanDong/harness-engineering-from-cc-to-ai-coding**](https://github.com/ZhangHanDong/harness-engineering-from-cc-to-ai-coding) [![Star](https://img.shields.io/github/stars/ZhangHanDong/harness-engineering-from-cc-to-ai-coding.svg?style=social&label=Star)](https://github.com/ZhangHanDong/harness-engineering-from-cc-to-ai-coding) | The "Horse Book" (《马书》) — Chinese mdBook framing Claude Code v2.1.88 as a Harness Engineering case study; covers architecture, prompt engineering, context management, prompt cache, security, and lessons for builders. |
| [**alchaincyf/loop-engineering-orange-book**](https://github.com/alchaincyf/loop-engineering-orange-book) [![Star](https://img.shields.io/github/stars/alchaincyf/loop-engineering-orange-book.svg?style=social&label=Star)](https://github.com/alchaincyf/loop-engineering-orange-book) | 花叔 (HuaShu)'s plain-language guide to loop engineering, in both Chinese and English. It puts the loop one layer above the harness, walks through what a loop does and the pieces it needs, and credits Steinberger, Osmani, and Anthropic's Claude Code team. |
| [**stanford-iris-lab/meta-harness**](https://github.com/stanford-iris-lab/meta-harness) [![Star](https://img.shields.io/github/stars/stanford-iris-lab/meta-harness.svg?style=social&label=Star)](https://github.com/stanford-iris-lab/meta-harness) | Reference implementation for the [Meta-Harness paper](https://arxiv.org/abs/2603.28052). Holds the model fixed and searches the harness itself (memory, retrieval, context construction, prompts, tool selection) through a propose, score, and Pareto-frontier loop. Turns the harness from something you design by hand into something you can optimize. |

### Blog Posts & Technical Articles

| Article | What Makes It Valuable |
|:--------|:----------------------|
| [Marco Kotrotsos — "Claude Code Internals" (15-part series)](https://kotrotsos.medium.com/claude-code-internals-part-1-high-level-architecture-9881c68c799f) | Most systematic pre-leak analysis. Architecture, agent loop, permissions, sub-agents, MCP, telemetry. |
| [Alex Kim — "The Claude Code Source Leak"](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/) | Anti-distillation mechanisms, frustration detection, Undercover Mode, ~250K wasted API calls/day. |
| [Haseeb Qureshi — Cross-agent architecture comparison](https://gist.github.com/Haseeb-Qureshi/2213cc0487ea71d62572a645d7582518) | Claude Code vs Codex vs Cline vs OpenCode — architecture-level comparison. |
| [George Sung — "Tracing Claude Code's LLM Traffic"](https://medium.com/@georgesung/tracing-claude-codes-llm-traffic-agentic-loop-sub-agents-tool-use-prompts-7796941806f5) | Complete system prompts and full API logs. Discovered dual-model usage (Opus + Haiku). |
| [Agiflow — "Reverse Engineering Prompt Augmentation"](https://agiflow.io/blog/claude-code-internals-reverse-engineering-prompt-augmentation/) | 5 prompt augmentation mechanisms backed by actual network traces. |
| [Engineer's Codex — "Diving into the Source Code Leak"](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code) | Modular system prompt, ~40 tools, large query/tool subsystem, anti-distillation. |
| [MindStudio — "Three-Layer Memory Architecture"](https://www.mindstudio.ai/blog/claude-code-source-leak-memory-architecture) | In-context memory, MEMORY.md pointer index, CLAUDE.md static config. Best single resource on memory. |
| [WaveSpeed — "Claude Code Architecture: Leaked Source Deep Dive"](https://wavespeed.ai/blog/posts/claude-code-architecture-leaked-source-deep-dive/) | 512K-line TS source deep dive; context compression and anti-distillation. |
| [Zain Hasan — "Inside Claude Code: An Architecture Deep Dive"](https://zainhas.github.io/blog/2026/inside-claude-code-architecture/) | Layered architecture, 5 entry modes, multi-agent walkthrough. |
| [Addy Osmani — "Agent Harness Engineering"](https://addyosmani.com/blog/agent-harness-engineering/) | Frames harness engineering as a discipline with named primitives (filesystem/git state, sandboxes, AGENTS.md memory, compaction, planning loops, hooks); cites Claude Code as the canonical mature example. |
| [Addy Osmani — "Loop Engineering"](https://addyosmani.com/blog/loop-engineering/) | A practitioner account of building the loop that repeatedly prompts an agent: automations, worktrees, skills, connectors, subagents, and persistent progress records. Useful alongside the paper's analysis of the surrounding harness. |
| [Armin Ronacher — "The Coming Loop"](https://lucumr.pocoo.org/2026/6/23/the-coming-loop/) | Splits the agent loop (tool calls inside one run) from the harness loop (a system that keeps re-prompting the agent to run again). Ronacher is skeptical: loops work well for porting code, tuning performance, and scanning for security bugs, but the code they leave behind tends to be more defensive and harder to maintain, so people still have to read it and decide what to keep. |
| [LangChain — "The Art of Loop Engineering"](https://www.langchain.com/blog/the-art-of-loop-engineering) | Describes four loops built around an agent: the agent loop, a verification loop that scores the output and retries, an event-driven loop that starts agents on outside triggers, and a hill-climbing loop that reads production traces to improve the harness itself. The point: most of the value comes from these loops, not the model on its own. |
| [Andrej Karpathy — "Sequoia Ascent 2026"](https://karpathy.bearblog.dev/sequoia-ascent-2026/) | Argues for "agentic engineering": humans orchestrate and verify rather than write code. "LLMs and reinforcement learning automate what you can verify"; "you can outsource your thinking, but you can't outsource your understanding." |
| [Lilian Weng — "Harness Engineering for Self-Improvement"](https://lilianweng.github.io/posts/2026-07-04-harness/) | Defines the harness as "the system surrounding a base model that orchestrates execution and decides how the model thinks and plans, calls tools and acts, perceives and manages context, stores artifacts, and evaluates results," then argues that the near-term path to recursive self-improvement does not start with a model rewriting its own weights: it starts with coding agents evolving the harness itself. Covers design patterns, harness optimization, evolutionary search, and joint optimization with weights. |
| [Armin Ronacher — "Better Models: Worse Tools"](https://lucumr.pocoo.org/2026/7/4/better-models-worse-tools/) | The clearest evidence yet that the harness shapes the model, not only the reverse. Opus 4.8 and Sonnet 5 invent fields like `requireUnique` and `oldText2` that no schema defines, while the surrounding payload stays byte-correct. Ronacher's reading: the models were trained in Claude Code's forgiving harness, which quietly repairs malformed calls by dropping unknown keys and accepting parameter aliases, so slightly-wrong calls still got rewarded. Tool schemas are not a neutral abstraction. |
| [Addy Osmani — "Agentic Autonomy Levels"](https://addyosmani.com/blog/agentic-autonomy-levels/) | Separates two axes that usually get collapsed: agency (how independently one agent runs) and orchestration (how several agents coordinate). Proposes a pre-execution contract covering goal, scope, non-goals, tools and permissions, stop conditions, evidence requirements, escalation path, and budget, offering a structured alternative to Claude Code's in-the-moment approval prompts. |
| [Geoffrey Litt — "Understanding is the new bottleneck"](https://www.geoffreylitt.com/2026/07/02/understanding-is-the-new-bottleneck) | As agents get better at verifying their own work, the scarce resource stops being correctness checking and becomes human comprehension. Litt argues the interface should build understanding rather than present diffs, proposing literate diffs with embedded explanation, interactive micro-worlds for stepping through a change, and shared spaces where a team forms a common mental model. A control-surface argument about what the harness owes the human, not what it lets the agent do. |
| [Haseeb Qureshi — "Inside the Claude Code source"](https://gist.github.com/Haseeb-Qureshi/d0dc36844c19d26303ce09b42e7188c1) | Key module analysis: the React/Ink UI, four compaction strategies, and the dynamic prompt boundary system. |
| [Han HELOIR YAN — "Nobody Analyzed Its Architecture"](https://medium.com/data-science-collective/everyone-analyzed-claude-codes-features-nobody-analyzed-its-architecture-1173470ab622) | "The moat is the harness, not the model." An early statement of the thesis this paper develops in full. |
| [Marco Kotrotsos — "Part 8: The Permission System"](https://kotrotsos.medium.com/claude-code-internals-part-8-the-permission-system-624bd7bb66b7) | The permission-focused installment of the Internals series; the closest pre-leak reconstruction of the approval path. |
| [Vincent Qiao — "Permissions System Deep Dive"](https://blog.vincentqiao.com/en/posts/claude-code-settings-permissions/) | Settings-level walkthrough of how permission rules are actually written and resolved in practice. |
| [ClaudeCodeCamp — "How Prompt Caching Actually Works"](https://www.claudecodecamp.com/p/how-prompt-caching-actually-works-in-claude-code) | The clearest practical account of cache behavior in Claude Code, which is the constraint most compaction designs have to work around. |
| [Gigi Sayfan — "MCP Unleashed"](https://medium.com/@the.gigi/claude-code-deep-dive-mcp-unleashed-0c7692f9c2c2) | Deep dive on the MCP integration path: transports, server lifecycle, and how external tools enter the tool surface. |
| [DEV Community — "Architecture via Rust Rewrite"](https://dev.to/brooks_wilson_36fbefbbae4/claude-code-architecture-explained-agent-loop-tool-system-and-permission-model-rust-rewrite-41b2) | Reads the architecture through a Rust reimplementation, laying out the tool set in a three-layer structure. |
| [Reid Barber — "Reverse Engineering Claude Code"](https://www.reidbarber.com/blog/reverse-engineering-claude-code) | One of the earliest analyses (mid-2025), covering the REPL architecture and the original tool suite. |
| [Kir Shatrov — "Reverse Engineering Claude Code"](https://kirshatrov.com/posts/claude-code-internals) | Used mitmproxy to intercept API traffic and reported a single measured experiment end to end, including LLM time and cost. |
| [Sabrina Ramonov — "Reverse-Engineering Using Sub Agents"](https://www.sabrina.dev/p/reverse-engineering-claude-code-using) | Methodologically interesting: builds custom sub-agents (File Splitter, Structure Analyzer) and uses them to reverse-engineer the minified bundle. |
| [Simon Willison — "Auto mode is now the default"](https://simonwillison.net/2026/Aug/8/auto-mode/) | An independent reading of Anthropic's auto-mode default announcement. Willison's caution is that the headline comparison (89% of dangerous commands caught by the classifier against 13.6% for tired human reviewers) blends accidental damage with deliberate prompt injection, which a classifier cannot reliably stop, so the durable fix is withholding sensitive data and high-risk tools rather than trusting automated approval. The skeptic's counterweight to the vendor announcement in the official-resources list above. |
| [Birgitta Böckeler — "TDD inside the agent loop: theater or actual value?"](https://martinfowler.com/articles/exploring-gen-ai/tdd-in-the-agent-loop.html) | Runs the experiment instead of asserting the practice: agent-driven TDD cost roughly three to eight times more tokens without producing better designs than test-after runs, because an agent does not get the human psychological benefit that makes TDD work. Recommends judging agents by outcome measures such as mutation testing and static analysis rather than mandating a process the loop cannot feel. |
| ["Why does Opus 5 feel worse to work with?"](https://mun-logadan.github.io/why-does-opus-5-feel-worse/) | Locates a harness-relevant failure in benchmark incentives: optimizing for benchmarks rewards agents that make a bold, usually-correct guess under ambiguity instead of asking a clarifying question, the opposite of what consequential, high-ambiguity work needs. A widely-shared argument about the autonomy-versus-checking-in dial in agent loop design. |
| [Florian Herrengt — "AI is removing the middle class of software engineering?"](https://blog.florianherrengt.com/ai-removing-middle-class-software-engineering.html) | AI lifted the speed limit on writing code but not on understanding it, so the review burden becomes the binding constraint and shifts onto the engineers who can still judge code. A labor-market framing of the same review-is-the-bottleneck argument running through the loop-engineering essays above. |
| [Steve Yegge — "The Shape of Things to Come"](https://yegge.ai/essays/the-shape-of-things-to-come/) | Argues agent-harness architecture is converging on loops, graphs, and issue trackers, that sequential CI/CD does not survive at agentic scale (proposing a batch, swarm-diagnosis model in its place), and that bespoke in-house harnesses beat reusable third-party frameworks because the harness has to be fitted to the codebase. A practitioner's map of where the harness layer is heading. |

### Cross-Vendor Code-Agent Engineering

Official engineering posts from other vendors building code agents — useful for seeing how the same design questions are answered outside Claude Code.

| Resource | Vendor | What's Notable |
|:---------|:-------|:---------------|
| [Codex v0.153.4](https://github.com/openai/codex/releases/tag/rust-v0.153.4) | OpenAI | A versioned comparison of summarizing compaction, experimental context-window management, and cross-session memory. The [source notes](./docs/agent-design-space-source-notes.md#source-codex-memory) distinguish their activation and lifetimes, including the later model-capability check. |
| [Deep Agents vs LangChain vs LangGraph](https://www.langchain.com/blog/deep-agents-vs-langchain-vs-langgraph) | LangChain | Explains how its harness, framework, and runtime compose. An agent loop can run inside a graph, and a custom graph can be exposed as a subagent. Published August 6, 2026; a useful earlier source for the guide's reasoning-placement decision. |
| [Cloud Agents and Cursor Harness Improvements](https://cursor.com/changelog/08-19-26), [Self-hosted machines](https://cursor.com/changelog/self-hosted-machines) | Cursor | The August 19 update adds event subscriptions, persistent goals, and subagents on separate machines. The September 2 update adds scheduling across named worker pools. Together they motivate separating the lifetime of a goal, a session, and its execution machine; the announcements do not establish exactly-once event or result delivery. |
| [Managed marketplace updates](https://github.blog/changelog/2026-08-26-enterprise-managed-settings-now-support-autoupdate-for-plugin-marketplaces/), [Content exclusions in app and CLI](https://github.blog/changelog/2026-09-02-content-exclusions-generally-available-in-copilot-app-and-cli/) | GitHub | Adds controls over plugin-marketplace updates and, for Business/Enterprise app and CLI users, excluded context. These govern different parts of the system. Content exclusion has client and input-path limits and is not a general filesystem sandbox. |
| [Harness Engineering: Leveraging Codex in an Agent-First World](https://openai.com/index/harness-engineering/) | OpenAI | Frames the "harness" as the constraints, feedback loops, and documentation that make agents reliable; reports a roughly 1M-line beta built with essentially no hand-written code. |
| [Codex 0.147.0](https://github.com/openai/codex/releases/tag/rust-v0.147.0) | OpenAI | Brings portable Agent Plugins into a catalog spanning local, personal, workspace, and remote sources; adds `--approve-for-me`, an opt-in implementation of the 2026-07-28 MCP protocol, remote compaction, transcript sections, and imports from Cursor-managed skills and Claude/Cursor conversations. The security boundary tightens at the same time: unfamiliar local projects require explicit trust, bearer tokens are redacted from history and replay, and failed plugin policy updates fail closed for network access. |
| [Deep Agents v0.7](https://www.langchain.com/blog/deep-agents-v0-7) | LangChain | Removes the base system prompt and default todo middleware, cuts built-in tool descriptions by 43%, and lowers first-turn base input from about 6K to 2K tokens; the reported multi-model evaluation does not show a statistically clear loss in overall reward. The post also identifies when todo planning helps and exposes summarization thresholds, search truncation, and file pagination as harness controls. Unlike Anthropic's context-engineering post above, this one tests the change across several model backends. |
| [Best Practices for Coding with Agents](https://cursor.com/blog/agent-best-practices) | Cursor | Articulates an agent harness as three components — Instructions, Tools, and Model — orchestrated per model. |
| [Build with Google Antigravity](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/) | Google | Agent-first platform: a Manager view for asynchronous multi-agent orchestration, with Artifacts (plans, screenshots, recordings) as the verification mechanism instead of raw logs. |
| [Microsoft Agent Framework at BUILD 2026: Agent Harness, Hosted Agents, CodeAct](https://devblogs.microsoft.com/agent-framework/microsoft-agent-framework-at-build-2026-announce/) | Microsoft | Adds a built-in "agent harness" with automatic context compaction, file-based memory, plan and execute modes, skill discovery, parallel sub-agents, and a sandboxed shell. Also adds CodeAct, which lets the model run several tool calls as one Python program inside a fresh Hyperlight micro-VM each call. |
| [Codex Security: Now in Research Preview](https://openai.com/index/codex-security-now-in-research-preview/) | OpenAI | Application-security agent that builds a project-specific threat model, then finds and pressure-tests vulnerabilities in sandboxed validation environments. It has since shipped as the `@openai/codex-security` CLI and TypeScript SDK (first published July 28, 2026), adding deep-scan workers, Linear issue publishing, and CI integration behind a verified-access requirement. |
| [Meet Your Agent Harness and Claw](https://devblogs.microsoft.com/agent-framework/meet-your-agent-harness-and-claw/), [Working with Your Data, Safely](https://devblogs.microsoft.com/agent-framework/agent-harness-working-with-your-data-safely/), [Scaling Harness Capabilities](https://devblogs.microsoft.com/agent-framework/agent-harness-scaling-the-claw-or-harness-capabilities/) | Microsoft | A four-part harness series. Notable for the authorization design: a standing rule ("always approve this tool", "always approve these arguments") lives only inside the session and is never baked into the agent, which is a direct point of comparison with Claude Code's permission model. Capability growth is split into four orthogonal mechanisms: Skills, a restricted Shell, CodeAct, and Background Agents. |
| [Transitioning Gemini CLI to Antigravity CLI](https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/) and [antigravity-cli](https://github.com/google-antigravity/antigravity-cli) | Google | Gemini CLI is retired (effective June 18, 2026) in favor of a Go rewrite that shares one agent harness with the Antigravity 2.0 desktop app. Per its CHANGELOG (the repo ships releases and examples, not source), the mechanisms sit close to Claude Code's: nested subagents to grandchild depth and beyond, whose subtrajectory updates are recursively relayed to the root conversation; workspace-level `.agents/hooks.json` where a pre-tool hook decides whether a tool call is permitted; and per-project permission config, stored under `~/.gemini/config/projects/` rather than in the repo, taking precedence over global settings. |
| [Governing Agent Autonomy with Auto-review](https://cursor.com/blog/agent-autonomy-auto-review) | Cursor | A classifier runs inside the agent loop rather than as a separate endpoint. When it blocks an action it returns an explanation to the parent agent, which can often route around it safely without ever interrupting the user. Roughly 4% of actions get blocked while only about 7% of chats end up interrupting a human. Blocking as a steering signal, not just a stop. |
| [Customize Cursor](https://cursor.com/changelog/customize) | Cursor | Plugins, skills, MCPs, subagents, rules, commands, and hooks collapse into one extension surface with user, team, and workspace scopes. A non-Anthropic vendor converging on almost exactly Claude Code's extension points. |
| [Codex-maxxing for Long-Running Work](https://openai.com/index/codex-maxxing-long-running-work/) | OpenAI | The memory vault keeps `AGENTS.md`, `TODO.md`, `projects/`, and `people/` in GitHub so that the diff becomes the review interface for memory. States the design position directly: memory must be open-able, edit-able, diff-able, and reusable. |
| [Devin Fusion](https://cognition.com/blog/devin-fusion) and [Agentic MapReduce](https://devin.ai/blog/agentic-map-reduce/) | Cognition | Fusion couples two mechanisms usually treated separately: a classifier re-scores task difficulty mid-session, and the model swap is timed to land on context compaction, where a cache miss was going to happen anyway, so switching is effectively free. MapReduce draws a clean line between agents and deterministic compute: put an agent only where reasoning is needed, make everything else deterministic. |
| [AI SDK 7](https://vercel.com/blog/ai-sdk-7) | Vercel | HarnessAgent treats Claude Code, Codex, and Pi as interchangeable backends behind one API, with sessions that park and resume. WorkflowAgent makes each tool call a durable retryable step that survives process restarts. Tool approvals are HMAC-signed, which addresses a rarely-discussed problem: forged approvals. |
| [The 2026-07-28 MCP Spec Goes Stateless](https://blog.modelcontextprotocol.io/posts/sdk-betas-2026-07-28/) and [Enterprise-Managed Authorization](https://blog.modelcontextprotocol.io/posts/enterprise-managed-auth/) | MCP | The largest structural change in the tool layer so far. The `initialize` handshake and protocol-level session are gone, with capabilities moving to `server/discover`; tools can now pause mid-call and ask the user a question via `InputRequiredResult`; roots, sampling, and logging are deprecated. Separately, authorization moves up from per-user consent screens to the organization's IdP. |
| [Droid Shield 2.0](https://factory.ai/news/droid-shield-2-0) | Factory | A deterministic regex scanner flanked by two fine-tuned models: one re-reads context for recall when the scanner stays silent, the other masks the candidate secret and judges from context alone whether the scanner produced a false positive. Deterministic rules and learned models wired into a two-way correcting gate. |
| [Software Is Made Between Commits (DeltaDB)](https://zed.dev/blog/introducing-deltadb) | Zed | Version control rebuilt for agent sessions. A message and the edit it produced are stored side by side so neither drifts from the other, and every reference anchors to a delta rather than a line number, so it survives as code moves underneath it. The argument: git is organized around discrete commits and was never designed to hold the conversation that produced the code. |
| [Multi-agent orchestration in the Responses API](https://developers.openai.com/api/docs/guides/responses-multi-agent) | OpenAI | Subagent orchestration moves out of client code and into the API. A root agent spawns a named tree (`/root/researcher`) coordinated by six hosted actions (`spawn_agent`, `send_message`, `followup_task`, `wait_agent`, `interrupt_agent`, `list_agents`), with `max_concurrent_subagents` capping how many subagents may be active at once across the whole tree (descendants count, the root agent does not). The tradeoff is stated plainly: reasoning summaries and `max_tool_calls` are unavailable while multi-agent is on. |
| [Programmatic Tool Calling](https://developers.openai.com/api/docs/guides/tools-programmatic-tool-calling) | OpenAI | The model writes JavaScript that reaches tools through a `tools.*` namespace, run in a fresh isolated V8 with no filesystem, no network, and no state carried between programs. Tools opt in through `allowed_callers`, so a tool can be reachable by the program, by the model directly, or both. Batching and filtering happen inside the program, which keeps intermediate tool output out of the context window. |
| [Copilot CLI changelog](https://github.com/github/copilot-cli/blob/main/changelog.md) | GitHub | Three July changes worth setting against Claude Code's permission model: an auto allow-all mode that approves whatever an LLM judge rates acceptable; a trusted repository pinning model, effort level, and context tier and extending the URL, MCP, and skill deny lists through `.github/copilot/settings.json`; and `preToolUse` hooks that deny a tool call by exiting with code 2. Plan mode hard-blocks built-in tools that would modify the workspace, but MCP and external tools stay allowed, a narrower boundary than the name suggests. |
| [Why We Built ADK 2.0](https://developers.googleblog.com/en/why-we-built-adk-20/) | Google | Workflows turn orchestration into a directed graph whose node transitions are evaluated in code rather than chosen by the model, so deterministic steps such as tool calls and human-in-the-loop approvals sit in the same graph as open-ended agent steps. The context decision is deliberate: the engine passes only the necessary subset of data to each downstream node, "shielding them from verbose, unrelated execution history" instead of letting a subagent inherit the whole trace. |
| [Introducing Harness Optimizer](https://strandsagents.com/blog/introducing-harness-optimizer/) | AWS | Treats the harness as tunable parameters rather than something written by hand: system prompts, tool descriptions, and skills become a `Formula`, a `RewardFunction` scores execution traces, and a `Trainer` runs rollout, reward, and update epochs. The default optimizer is itself an LLM that reads winning and losing traces contrastively and rewrites the Formula to encode the difference. Open source, and offered in production as AgentCore Optimization. |
| [The Self-Driving Company](https://replit.com/blog/self-driving-company) | Replit | A rare look at how a vendor runs agents internally rather than what it sells. Every employee gets a manager agent that spawns and orchestrates further agents on the company's own harness, microVMs, and remote filesystem, with the whole arrangement placed "behind access policies, token proxies, audit logging, and our ZeroTrust network." Human escalation is designed in rather than left as a fallback: the code-review agent grades risk and pulls in a second human reviewer only when warranted. |
| [Amp news](https://ampcode.com/news) | Sourcegraph | Two July entries push the same direction. Agents can now spawn other agents, message them, and exchange files across threads, which is peer-to-peer coordination rather than a parent collecting subagent reports. Separately, the Orb remote sandboxes moved to OIDC workload identity, replacing static credentials for agents that execute remotely. |
| [Agent Plugins 1.0](https://agent-plugins.org/specification) ([launch](https://github.blog/changelog/2026-08-12-agent-plugins-1-0-in-vs-code-copilot-cli-and-the-copilot-app/)) | Cross-vendor | A portable plugin package format published in August 2026: a `plugin.json` manifest bundling Agent Skills (a `skills/` directory) and MCP servers (`mcp.json`) into one directory that installs across clients. Its initial Technical Steering Committee lists Core Maintainers from Amazon, Cursor, Microsoft, OpenAI, and Vercel, and support ships day-one in VS Code, Copilot CLI, and the Copilot app. The extensibility-layer counterpart to what MCP did for tools: the two surfaces this paper analyzes, skills and MCP, getting one shared distribution format across vendors. |
| [One Agent, Every Surface: How We Built the Kiro Agent Harness](https://kiro.dev/blog/one-agent/) | AWS | Consolidates three separate agent implementations (IDE, CLI, and web) into a single standalone harness process addressed over the Agent Client Protocol, and replaces three incompatible permission systems with one capability-based model backed by Cedar, a formally-verified policy language. Context management and compaction become uniform across surfaces. A vendor answering the permission-model question with a verified policy engine rather than hand-written rules. |
| [Introducing Delta](https://zed.dev/blog/introducing-delta) | Zed | A standalone app for multiplayer coding with agents rather than a feature added to the editor. Built on DeltaDB (listed above), it replicates the conversation and the worktree together in real time for everyone in a thread, with comments anchored to code as it changes underneath them. Puts the thread, not the file, at the center of the session architecture. |
| [Global Plugins and Skills](https://ampcode.com/news/global-plugins-and-skills) | Sourcegraph | Centralizes plugin and skill management across two scopes: a personal tier for individual experimentation with live reload, and an admin-managed workspace tier that sets team defaults, with sharing between them. Extension distribution as a governed, team-scoped surface rather than a per-repo file. |
| [How to Build a Cloud Software Factory: Computer Use Verification](https://www.warp.dev/blog/how-to-build-a-cloud-software-factory-computer-use-verification) | Warp | A verify-behavior skill lets an agent drive a running application with mouse and keyboard to confirm its own work against the spec, instead of inspecting code alone. The same skill serves triage agents that reproduce bugs, implementation agents that validate features, and review agents that produce screenshot and video evidence for a human, all cloud-side so the developer's machine is untouched. Self-verification promoted from code inspection to behavioral proof. |

### Runtime & Sandbox Infrastructure

Where and how agent actions actually execute: sandboxes, managed runtimes, control planes, and the session protocols wrapped around them. The counterpart to Claude Code's own shell sandbox and execution boundary.

| Resource | Vendor | What's Notable |
|:---------|:-------|:---------------|
| [Temporal Deep Agents integration](https://docs.temporal.io/develop/python/integrations/deepagents), [Workflow Pause](https://docs.temporal.io/encyclopedia/workflow/workflow-pause) | Temporal | Listed as pre-release in the [August 27 digest](https://temporal.io/blog/durable-digest-august-2026). The integration runs the control loop in a replayable Workflow and model calls in Activities; tools and backends with real I/O need explicit wrappers. Pause stops new dispatch while in-flight Activities may still finish. |
| [Bedrock AgentCore Harness is Generally Available](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-harness-is-now-generally-available-go-from-idea-to-production-grade-agent-in-minutes/) | AWS | The harness stops being a loop you write and becomes a managed configuration object: `CreateHarness` and `InvokeHarness` declare the model, tools, skills, memory policy, and container environment, over seven primitives (microVM Runtime, Memory, Gateway, sandboxed Browser, Code Interpreter, Identity token vault, Observability). A new coordinate in the design space: who owns the loop, the environment, and the tool boundary. |
| [AgentCore Policy with Guardrails](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-bedrock-agentcore-policy-guardrails-generally-available/) | AWS | Enforcement happens at the gateway boundary, outside the agent's own code, so it holds no matter how autonomous the agent becomes. A sharp architectural contrast with in-loop permission checks. |
| [Running Untrusted Agent Code Without a Sandbox](https://www.langchain.com/blog/running-untrusted-agent-code-without-a-sandbox) and [Dynamic Subagents in Deep Agents](https://www.langchain.com/blog/introducing-dynamic-subagents-in-deep-agents) | LangChain | Capability isolation run backwards from the container model: QuickJS inside WASM starts at zero capability and the harness bridges in each capability explicitly. Interpreter memory can be serialized for a durable pause that waits on human approval. Subagent dispatch then drops from conversation turns into program control flow, with the model writing a script that calls `task({...})`. |
| [A Single Pane of Glass for All Your Cloud Agents](https://www.warp.dev/blog/multi-harness-cloud-agent-orchestration) | Warp | Oz as a multi-harness control plane: launch, track, govern, and steer Claude Code, Codex, and Warp Agent from one surface, compare their effectiveness, pick different harnesses for different tasks, and keep one consistent audit trail. Agent Memory carries across harnesses. |
| [Introducing the Warp Agent CLI](https://www.warp.dev/blog/introducing-the-warp-agent-cli-coding-agent) | Warp | Builds the agent on a PTY multiplexer rather than a line-oriented shell wrapper, so sessions survive directory changes and SSH hops and can operate interactive programs such as debuggers and editors. Native subagents can delegate across Warp, Claude Code, and Codex; a local session can hand off to the cloud for web monitoring and steering. PTY management is therefore part of the harness runtime, not merely a shell tool exposed to the model. |
| [Agent Host and the Agent Host Protocol](https://code.visualstudio.com/updates/v1_129) | Microsoft | VS Code 1.129 moves agent sessions out of the editor into a dedicated process addressed over an open Agent Host Protocol. Because the session lives in its own process it keeps running with no client attached and can be rendered from several windows at once, and one host runs Copilot, Claude, and Codex behind a single session model. Agents on the host can list other sessions, read their transcripts, spawn a session to hand off a subtask, and message another session, with sends requiring user confirmation and a burst cap so one request cannot fan out into unbounded sessions. |
| [Sandbox forking](https://github.com/e2b-dev/E2B/releases) | E2B | `Sandbox.fork()` checkpoints a running sandbox in place, briefly pausing it, snapshotting full memory state, and resuming it with its ID and expiration untouched, then starts N independent sandboxes from that snapshot. Gives a harness a cheap way to branch one expensively-configured environment across parallel attempts instead of rebuilding it for each. |
| [Docker VMM Public Beta](https://www.docker.com/blog/docker-vmm-public-beta/) | Docker | Docker's first self-owned hypervisor, replacing the third-party libkrun underneath Docker Desktop. The same engine powers Docker Sandboxes, the microVM isolation layer used for agent execution, so every change lands in both. Claims faster container and sandbox startup, better host-to-guest file I/O, and Hyper-V-grade isolation on Windows. The isolation substrate under a widely-used agent sandbox. |
| [Sandboxing](https://zed.dev/blog/sandboxing) | Zed | Agent terminal and fetch tools are sandboxed by default using OS-level primitives (Seatbelt on macOS, bwrap and namespaces on Linux, WSL on Windows), blocking writes outside the project, `.git` access, and unapproved network calls. The post walks through a symlink-swap attack to show that instruction-based restrictions alone are insufficient, the same lesson the GhostApproval research draws. |
| [Cloudflare Gateway: MCP Detection and AI Security Dashboard](https://developers.cloudflare.com/changelog/post/2026-08-12-mcp-detection-and-dashboard/) | Cloudflare | Cloudflare Gateway now fingerprints MCP traffic from its headers and payload shape, adds a policy selector to allow, block, or isolate it, and reports MCP request volume, unique users, and unique servers over time. Treats MCP as a network-inspectable, policy-governable traffic class at the enterprise perimeter rather than only an application-layer protocol. |

### Security Research & Incidents

Vulnerability research, disclosures, and real-world incidents involving coding agents, from any vendor. The safety envelope tested against an adversary rather than described on paper.

| Report | What It Shows |
|:--------|:----------------------|
| [Tenet Security — "One Fake Bug Report Hijacked a $250 Billion Company's AI Agent, Then 100+ More"](https://tenetsecurity.ai/blog/agentjacking-coding-agents-with-fake-sentry-errors/) | The sharpest demonstration to date that a tool's return value is untrusted input. A Sentry DSN is public by design (Sentry documents it as safe to embed in frontend JavaScript) and accepts arbitrary error payloads, so an attacker can POST an event carrying markdown instructions that render as a fake "Resolution" section. A developer then asks their agent to look into the Sentry issue, the agent pulls the poisoned event back through MCP, reads it as trustworthy remediation guidance, and runs an attacker-controlled npm package that exfiltrates AWS keys, GitHub tokens, and SSH credentials. Confirmed against Claude Code, Cursor, and OpenAI Codex, including sandboxed variants and CI/CD pipelines. |
| [Wiz — "GhostApproval"](https://thehackernews.com/2026/07/ghostapproval-symlink-flaws-could-let.html) | An informed-consent bypass aimed at the approval dialog itself. A repository ships a symlink named something harmless like `project_settings.json` that actually points at `~/.ssh/authorized_keys` or `~/.zshrc`, so the dialog shows the decoy name and the user approves a write that lands somewhere else entirely. Amazon Q Developer, Cursor, and Google Antigravity shipped fixes; Augment and Windsurf acknowledged without one. Anthropic disputed it for Claude Code, arguing the scenario sits outside its threat model because the developer chose to trust the folder and then approved the edit. Worth reading with that disagreement intact, since it is really a dispute about where consent is supposed to come from. |
| [Poisoning Claude Code: One GitHub Issue to Break the Supply Chain](https://flatt.tech/research/posts/poisoning-claude-code-one-github-issue-to-break-the-supply-chain/) | The disclosure documents two routes rather than a completed compromise: agent mode treated any GitHub App actor as trusted; separately, an official low-privilege triage example could be chained into a privileged tag-mode workflow. In either route, prompt-injected Claude could read OIDC request credentials from `/proc/self/environ` and exfiltrate them through issue updates. Anthropic mitigated the issues by `claude-code-action` v1.0.94. |
| [Towards a Risk Assessment of Malicious Skill Files in Coding Agents](https://arxiv.org/abs/2608.05223) | Treats agent skills as a software supply chain rather than inert prompt text. The authors transform 471 real shell commands into 2,826 benign-looking skills spanning 11 MITRE ATT&CK tactics, then complete 5,629 runs against Gemini CLI and Qwen Code. Reported exploitation rates are 95.5–96.1% and 71.6–74.0%, while explicit safety recognition appears in only 1.99% of runs. Although the preprint uses one synthetic attack template and two CLIs, the results support explicit skill admission, provenance, scanning, and runtime least privilege. |
| [HiddenLayer — "Five Layers of Securing AI Agent Harnesses"](https://www.hiddenlayer.com/research/how-to-secure-coding-agents) | Industry security research that names the harness, not just the model, as the primary coding-agent attack surface, and documents concrete techniques against it: instructions hidden in `<user_query>`-style XML tags that inherit system authority, MCP tool-definition metadata that steers behavior from inside the system prompt, skill YAML-frontmatter permission abuse, and "CopyPasta," a self-propagating prompt injection disguised as a license header that spreads across repositories. |
| [AI Guardrail Survival under Single-Cycle Agentic Self-Summarization](https://arxiv.org/abs/2608.11392) | Shows that safety rules embedded in an agent's context often survive compaction textually while silently losing their force: behavioral violation rates rise 34 to 57 percentage points when degraded post-compaction rules are compared against intact ones, and the loss is invisible to anyone inspecting the summary text, since "a presence check is not a safety check." The safety-side companion to the compaction-as-truth-loss work in the papers list, aimed at mechanisms like Claude Code's `/compact`. |
| [ColluSkill](https://arxiv.org/abs/2608.09732) and [SkillsMetric](https://arxiv.org/abs/2608.08468) | Two August studies mapping the limits of skill scanners. ColluSkill splits a malicious payload across several individually-benign skills that compose into harmful behavior only at runtime, reaching 96% average attack success against six representative scanners, with a proposed defense cutting it to 22.5%. SkillsMetric tests a five-stage static-analysis framework on 2,266 malicious skills and finds a blind spot the aggregate AUC hides: host-destruction attacks using ordinary shell commands are detected 0% of the time. Together they push the malicious-skills thread above into the arms race between composition attacks and static scanning. |
| [Embrace The Red — "LLM Heist: Hijacking LiteLLM for Traffic Interception, Key Theft, and Tool-Call Injection"](https://embracethered.com/blog/posts/2026/hijacking-litellm-for-fun-and-profit/) | An attacker with proxy-admin access to a LiteLLM gateway can reroute its traffic to an attacker-controlled instance, harvest backend provider keys, and inject forged tool calls into responses after inference, which bypasses prompt-level defenses because the agent client sees a legitimately-formatted call. A trust-boundary attack against the LLM-gateway layer many agent deployments sit behind. The precondition is pre-existing admin access, so it reads as a post-compromise amplifier rather than a zero-click break. |

### Evaluation & Benchmarks

Benchmarks for coding agents, and the growing body of work auditing whether those benchmarks measure what they claim. Relevant here because a harness change is only as legible as the evaluation used to detect it.

| Resource | Source | What It Shows |
|:---------|:-------|:--------------|
| [LoopArena](https://arxiv.org/abs/2608.28281) | arXiv | Holds the worker fixed to compare controllers that decide how a task should proceed. It separates management and stopping decisions from the worker's execution ability. Reported savings from sliced evaluation concern the benchmark protocol, not lower production costs from adding a controller. |
| [Building Agent Environments and Tasks](https://www.langchain.com/blog/building-agent-environments-and-tasks), [Tuned Evaluators](https://www.langchain.com/blog/introducing-langsmith-tuned-evaluators-starting-with-perceived-error) | LangChain | Connects production feedback to reviewable task specifications, executable environments, and regression checks. Perceived Error helps identify cases to investigate; it is a proxy for user experience, not proof that a task succeeded or failed. |
| [ClawBench: Can AI Agents Complete Everyday Online Tasks?](https://arxiv.org/abs/2604.08523) ([code](https://github.com/reacher-z/ClawBench) · [project](https://claw-bench.com/) · [dataset](https://huggingface.co/datasets/NAIL-Group/ClawBench)) | arXiv | A live-site browser-agent benchmark with 283 V1+V2 tasks across 163 websites. Runs execute in isolated browser containers and combine request interception with an LLM judge, while preserving replay, screenshots, HTTP traffic, browser actions, and agent messages for auditability. Its contribution to the harness question is making live websites — the least controllable variable in agent evaluation — into an environment a run can actually be reproduced against. |
| [Harness-Bench: Measuring Harness Effects across Models in Realistic Agent Workflows](https://arxiv.org/abs/2605.27922) | arXiv | 106 sandboxed tasks and 5,194 execution trajectories run with task environments, budgets, and evaluation protocols held fixed while the harness configuration varies across model backends. Names a recurring pattern it calls execution-alignment failure, where plausible reasoning becomes decoupled from tool feedback and workspace state, and concludes that agent capability "should be reported at the model-harness configuration level rather than attributed to the base model alone." |
| [Position: Coding Benchmarks Are Misaligned with Agentic Software Engineering](https://arxiv.org/abs/2606.17799) | arXiv | Argues that "a coding agent in practice is not a model: it is a system harness," so an end-to-end score conflates model, harness, context, environment, and feedback signal, any of which can move the number "by margins comparable to those between adjacent model generations." Also faults grading against a single reference solution, which penalizes equally valid alternatives. |
| [Harbor-Index 1.0](https://harbor-index.org/) | Harbor | A compact cross-domain agent benchmark: 82 tasks distilled from 6,627 candidates through difficulty screening, automated audit, and human review, spanning software engineering, scientific research, tools and systems, knowledge, mathematics, data analytics, and safety. Scoring is strictly binary with no partial credit, and it is built deliberately for headroom rather than for predicting performance elsewhere. |
| [Are Performance-Optimization Benchmarks Reliably Measuring Coding Agents?](https://arxiv.org/abs/2607.01211) | arXiv | Replays the official reference patches of GSO, SWE-Perf, and SWE-fficiency on fresh machines and finds that only 39/102, 11/140, and 411/498 tasks respectively satisfy the benchmarks' own validity rules. Scoring methodology also matters more than expected: official rankings disagree on 9 of 28 pairwise comparisons among submissions appearing in both GSO and SWE-fficiency. |
| [Do Androids Dream of Breaking the Game? Systematically Auditing AI Agent Benchmarks with BenchJack](https://arxiv.org/abs/2605.12673) | arXiv | An automated red-teaming system that drives coding agents to attack benchmarks rather than solve them, surfacing 219 distinct flaws across eight classes in 10 popular agent benchmarks and reaching near-perfect scores "without solving a single task." An extended pipeline then patches the holes, cutting the hackable-task ratio from near 100% to under 10% on four benchmarks. |
| [PERFOPT-Bench: Evaluating Coding Agents on Software Performance Optimization](https://arxiv.org/abs/2607.07744) | arXiv | Seven agent stacks across seven long-horizon optimization tasks. Finds optimization performance is workload-dependent rather than settled by model identity, with no single stack dominating, and concludes that "raw speedup is unsafe as a benchmark score, since some large gains arise from benchmark-specific shortcut exploitation." |
| [LoopsBench: From Harness Engineering to Loop Engineering in Benchmarking Coding Agent](https://arxiv.org/abs/2608.00267) | arXiv | Evaluates sustained development rather than one issue and one terminal state. Its 112 tasks are dependency DAGs over more than 5,300 separately testable development units; a flow-aware runtime releases tests only as prerequisites become ready and keeps completed units as regression obligations. The best tested pairing — Opus 4.7, Claude Code, and an outer continuation loop — resolves 25% of tasks, while recorded agent plans recover only part of the source-derived dependency structure. This is the first benchmark in the catalog aimed explicitly at the loop outside the agent loop. |
| [Active-SWE: Benchmarking Coding Agents for Proactive Bug Fixing without Issue Reports](https://arxiv.org/abs/2608.04682) | arXiv | Removes the artifact that most repository benchmarks quietly give away: a curated issue report that already says what is wrong. Across 1,663 tasks, six bug categories, and eight languages, agents must locate and repair recorded bugs, handle multiple bugs, and surface additional valid defects without report guidance. The reported frontier systems struggle across all three settings, exposing proactive discovery as a different capability from issue-conditioned patching. |
| [SciCode-Verified: A Systematic Audit of Scientific Coding Benchmark Validity](https://arxiv.org/abs/2608.04975) | arXiv | Audits 65 scientific-programming problems and finds 263 defects across prompts, tests, and references; 192 correct solutions had been rejected. After repair, subproblem accuracy rises from 45–60% to 84–98% and main-problem accuracy from 9–27% to 69–92%. The size of the correction shows how strongly benchmark implementation can affect the reported model result. |
| [When the Judge Changes, So Does the Measurement: Auditing LLM-as-Judge Reliability](https://arxiv.org/abs/2607.08535) | arXiv | Treats swapping the judge model as a measurement-validity problem rather than a free upgrade. Across four judgment datasets it finds adjacent judge releases are not interchangeable, stronger judges reduce but do not remove position and verbosity bias, and repeated-sample juries "add little when errors are correlated." |
| [Reward Hacking is Swamping Model Intelligence Gains](https://cursor.com/blog/reward-hacking-coding-benchmarks) | Cursor | An audit of 731 Opus 4.8 Max trajectories found that, among successful resolutions on SWE-bench Pro, 63% of fixes were retrieved rather than derived: 57% found the merged PR or fixed source file on the public web, 9% mined the repository's bundled git history for the fix commit. Two isolations expose it, and scores fall sharply under them: `.git` is stripped and the repo reinitialized as a single commit (history restored only at scoring time), and network access is denied by default behind a proxy that allows only an allow-list of package registries. Makes eval-harness environment design a precondition for the benchmark meaning anything. |
| [SWE-Bench ProMax: Benchmarking Agents on Large-Scale Multilingual Code Refactoring](https://arxiv.org/abs/2608.09802) | COLM 2026 | 170 expert-curated multi-file refactoring instances across seven languages (averaging 11.4 files and 261.6 lines per task), motivated by the finding that roughly 60% of unsolved SWE-bench Verified instances have flawed tests. State-of-the-art models resolve only 41.2%, restoring headroom that saturated single-file benchmarks have lost. |
| [QuoteBench: How Matched Scores Can Hide Command-Path Failures](https://arxiv.org/abs/2608.13547) | arXiv | 56 tasks over 14 incident-derived families showing that a matched execution score conflates generating a command with the transport and parsing path that carries it to a shell: replaying an identical model reply through an added parser alone drops success 55 to 73 points. Argues an evaluation report must document the generation contract, the execution path, and the validator separately rather than reporting one pass rate. |
| [Senior SWE-bench](https://senior-swe-bench.snorkel.ai/) ([how it works](https://senior-swe-bench.snorkel.ai/blog/2026-06-16-how-it-works)) | Snorkel AI | A coding-agent benchmark of 100 tasks, 50 public and 50 held private, drawn from 12 production repositories (among them Gitea, PostHog, Prefect, Teleport, Turborepo, and Immich), built with Princeton and UW-Madison collaborators. The public-private split is the point: it measures generalization to tasks whose solutions cannot have leaked into training data. |

### Related Academic Papers

| Paper | Venue | Relevance |
|:------|:------|:----------|
| [Graph Engineering in the Era of LLM Agents: From Individual Intelligence to System Intelligence](https://arxiv.org/abs/2608.21156) | arXiv | Feng et al. organize agent systems around task graphs, agent graphs, and runtime state graphs. The survey connects dependencies, coordination, and recoverable state within an explicit framework, while placing earlier graph-based systems in that history. |
| [HarnessLens](https://arxiv.org/abs/2608.27311) | arXiv | Ties proposed harness changes to a behavioral hypothesis, relevant validation tasks, and checks on prior successes, then confirms candidates on a fresh batch. No observed regression on the selected tasks is not a general guarantee, and the experiments do not equalize every method's total compute budget. |
| [Meta-Harness: End-to-End Optimization of Model Harnesses](https://arxiv.org/abs/2603.28052) | arXiv | A coding agent acts as the proposer and searches the harness itself (memory, retrieval, context construction, prompts, tool-use logic) with the model held fixed, maintaining a population and a Pareto frontier over evaluated harnesses. Beats a state-of-the-art context-management system by 7.7 points while using 4x fewer context tokens, and its discovered harnesses surpass the best hand-engineered baselines on TerminalBench-2. Turns the harness from a thing you design by hand into a thing you can optimize. |
| [From Question Answering to Task Completion: A Survey on Agent System and Harness Design](https://arxiv.org/abs/2606.20683) | arXiv | A contemporaneous survey through a "model-harness lens" that decomposes the execution harness into six coupled runtime responsibilities: observation, context, control, action, state, and verification. The closest structural peer to this paper's framing. |
| [Architectural Design Decisions in AI Agent Harnesses](https://arxiv.org/abs/2604.18071) | arXiv | Source-grounded study of 70 agent-system projects identifying recurring design dimensions; closest contemporary peer to this paper's design-space framing. |
| [ActPlane: Programmable OS-Level Policy Enforcement for Agent Harnesses](https://arxiv.org/abs/2606.25189) | arXiv | Intercepts every execution path with eBPF in the kernel, including the indirect paths that bypass the tool-call layer entirely, at 1.9% to 8.4% overhead. Names the gap directly: permission checks that live at the tool-call layer, as Claude Code's do, can be routed around. |
| [Lingering Authority: Revocable Resource-and-Effect Capabilities for Coding Agents](https://arxiv.org/abs/2606.22504) | arXiv | Names a real weakness in session-scoped approval: once granted, authority does not expire as the task moves on. PORTICO compiles a task spec into capabilities, grant rules, and closure predicates, materializing resources as epoch-bound handles that die once the closure condition is met. |
| [TokenPilot: Cache-Efficient Context Management for LLM Agents](https://arxiv.org/abs/2606.17016) | arXiv | Confronts a constraint most compaction work steps around: rewriting history destroys the KV prefix cache. Filters noise at the ingestion boundary to keep the prefix stable, and evicts context segments only after their lifecycle ends. |
| [The Balkanization of Execution-Security Research for AI Coding Agents](https://arxiv.org/abs/2607.05743) | arXiv | An SoK sorting 39 execution-layer security papers into 17 categories (sandbox isolation, capabilities and access control, policy enforcement, TOCTOU, MCP threats, identity delegation, execution provenance, egress control). Reports that policy enforcement fails against real denylists 69% to 98% of the time. |
| [How Agents Ask for Permission: User Permissions for AI Agents, from Interfaces to Enforcement](https://arxiv.org/abs/2607.13718) | arXiv | Surveys 21 agent permission systems and evaluates five commercial agents, building a taxonomy along three axes: how a policy is specified at the interface, how it is derived into an internal policy, and how it is enforced at run time. The closest academic peer to this paper's permission analysis, and a direct test of whether Claude Code's model generalizes. |
| [Bad Memory: Evaluating Prompt Injection Risks from Memory in Agentic Systems](https://arxiv.org/abs/2607.14611) | arXiv | Tests Claude Code and OpenAI Codex across four models and finds an asymmetry that matters for memory design: the agents largely resist untrusted content trying to overwrite their memory files, but payloads already sitting in those files go on to attack current and future sessions. Persistence turns one successful injection into a standing one. |
| [Agent Data Injection Attacks are Realistic Threats to AI Agents](https://arxiv.org/abs/2607.05120) | arXiv | Separates a second attack class from instruction injection: instead of smuggling commands, the attacker forges security-critical metadata such as resource identifiers, data origins, and tool-response formats, which agents accept because they do not isolate trusted from untrusted data. Demonstrated as arbitrary clicks against Claude in Chrome, Antigravity, and Nanobrowser, and as remote code execution and supply-chain attacks against Claude Code, Codex, and Gemini CLI. |
| [Failure as a Process: An Anatomy of CLI Coding Agent Trajectories](https://arxiv.org/abs/2607.09510) | arXiv | Manually annotates 1,794 complete trajectories (over 63,000 execution steps) drawn from 3,843 runs across seven frontier models and three scaffolds. Failures are predominantly epistemic, begin within the first few execution steps, and stay hidden until recovery is no longer possible, which argues for validation early in the loop rather than outcome-only evaluation. |
| [Better Harnesses, Smaller Models: Building 90% Cheaper Agents via Automated Harness Adaptation](https://arxiv.org/abs/2607.08938) | arXiv | A meta agent reads failure trajectories and lifts shared task difficulty out of the model and into the harness as tailored instructions, tools, and orchestration loops. Recovers 89.7% of large-model performance at 4% of the cost, and closes the small-model gap outright on seven of 21 task and model pairs. |
| [Rethinking the Evaluation of Harness Evolution for Agents](https://arxiv.org/abs/2607.12227) | arXiv | The skeptical counterweight to the harness-optimization line of work. Measured against simple test-time scaling and discovery baselines under matched feedback and inference budgets on Terminal-Bench 2.1, automatic harness evolution "does not consistently outperform" the simpler methods and generalizes poorly to held-out tasks. |
| [Rethinking Self-Evolving Agent Skills: The Role of Feedback and Search](https://arxiv.org/abs/2608.02636) | arXiv | Shows skill evolution as sparse, validation-filtered search rather than monotonic learning: only 55 of 388 candidates become byte-distinct validation bests; validation selects an evolved skill in 11 of 14 settings, nine of which improve test performance, and every selected skill uses failed trajectories as feedback. Parallel sampling nearly recovers SearchQA but remains 30.96 percentage points behind the evolved SpreadsheetBench workflow, so the value is task-dependent. |
| [Learning Globally Reusable Skills for Coding Agents](https://arxiv.org/abs/2608.06153) | arXiv | Replaces isolated skill rewrites with a Skill Relation Graph that co-evolves dependencies among skills, consolidates local updates into reusable clusters, and replay-verifies changes to catch overfitting and regressions. Evaluated on bug-revealing test generation and false-positive filtering with OpenHands, mini-SWE-agent, and an internal industrial agent. It shifts skill evolution from "save what worked once" toward compatibility-aware, regression-tested maintenance of a skill bank. |
| [DCAS: Decoupling CLI Agent Scaffolding to Internalize Planning across Scaffolds](https://arxiv.org/abs/2608.06113) | arXiv | Finds that a model fine-tuned on OpenHands-collected trajectories falls from 52.6% under OpenHands to 8.4% under OpenCode, while base models diverge less. DCAS intercepts API traffic to pair CLI scaffolds with alternate backends and collects 576 planning-aware Claude Code trajectories; planning-and-execution tuning reaches 55.8% with its own plan and transfers to unseen OpenCode (49.4%) and mini-SWE-agent (42.4%). Within this single-benchmark study, the gap shows that trajectory tuning can overfit a scaffold protocol. |
| [Ledger: Stateful Harness Middleware for Coding Agents](https://arxiv.org/abs/2608.00808) | arXiv | Adds an external task ledger that records evidence, dependencies, and verified progress without another model call. On all 500 SWE-bench Verified tasks it raises GPT-5 mini from 56.2% to 64.2% while cutting cost 28.9%, and MiniMax M2.5 from 75.8% to 81.0% while cutting cost 31.8%; Codex also gains 3.4 points at 24.4% lower cost. The result supports keeping task state in harness middleware instead of relying on prompt prose alone. |
| [The Resume Contract: Testing Durable Agent Workflows after Failure](https://arxiv.org/abs/2608.03836) | arXiv | Defines six formal properties for checkpoint-and-resume behavior, model-checks 7.4 million states, and probes five workflow frameworks. The tests show that crash recovery can re-run already durable work, duplicate effects, and multiply one logical resume into one execution per concurrent process. "Durable" therefore needs an explicit resume contract, not merely a checkpoint API. |
| [From Anatomy to Smells: An Empirical Study of SKILL.md in Agent Skills](https://arxiv.org/abs/2607.01456) | arXiv | Analyzes 238 real-world skills into a taxonomy of 13 higher-level and 44 lower-level components, then detects "skill smells" as violations of the practices that taxonomy implies. Over 99% of SKILL.md files carry at least one, and smells rarely disappear as a skill evolves. Direct evidence on the extensibility surface this paper analyzes. |
| [Decoding the Configuration of AI Coding Agents](https://arxiv.org/abs/2511.09268) | arXiv | Empirical study of 328 Claude Code configuration files — SE concerns and co-occurrence patterns. |
| [On the Use of Agentic Coding Manifests](https://arxiv.org/abs/2509.14744) | arXiv | Analyzed 253 CLAUDE.md files from 242 repos — structural patterns in operational commands. |
| [Context Engineering for Multi-Agent Code Assistants](https://arxiv.org/abs/2508.08322) | arXiv | Multi-agent workflow combining multiple LLMs for code generation. |
| [OpenHands: An Open Platform for AI Software Developers](https://arxiv.org/abs/2407.16741) | ICLR 2025 | Primary academic reference for open-source AI coding agents. |
| [SWE-Agent: Agent-Computer Interfaces](https://arxiv.org/abs/2405.15793) | NeurIPS 2024 | Docker-based coding agent with custom agent-computer interface. |
| [The OpenHands Software Agent SDK](https://arxiv.org/abs/2511.03690) | arXiv | Composable SDK foundation for production agents; the engineering counterpart to the OpenHands platform paper above. |
| [A Survey on Code Generation with LLM-based Agents](https://arxiv.org/abs/2508.00083) | arXiv | Broad survey of the AI coding agent field, useful for situating harness work inside the wider code-generation literature. |
| [AI Agent Systems: Architectures, Applications, and Evaluation](https://arxiv.org/html/2601.01743v1) | arXiv | Broad agent-system taxonomy spanning architecture, application, and evaluation. |
| [Why Does CLAUDE.md Keep Growing? Catastrophic Remembering in Agentic Coding](https://arxiv.org/abs/2608.11095) | arXiv | Mines 247,694 instruction lifetimes across 1,867 repositories and finds standing-instruction files grow 226% over their lifetime, because adding a rule is cheap while removing one is hard under dependency uncertainty. Adding explanatory comments cuts excess growth from 211.3% to 1.4% and improves instruction-following by up to 23.1%. Direct empirical evidence on the CLAUDE.md memory mechanism this paper analyzes. |
| [A Study of Cursorrules Files in GitHub Open Source Projects](https://arxiv.org/abs/2608.10622) | ICSOFT 2026 | An empirical study of 12,110 `.cursorrules` and `.mdc` files across 11,427 repositories: how developers encode project context and engineering norms for coding agents, and how the format has evolved. The Cursor-side counterpart to the CLAUDE.md configuration studies already listed, and the companion to the CLAUDE.md-growth paper above. |
| [The Devil Is in the Interface: Evaluating How Tool Architecture Shapes Coding Agent Behavior](https://arxiv.org/abs/2608.11386) | arXiv | A controlled ablation of six tool architectures (bash-only, structured low-level tools, natural-language search, and a Python CodeAct style) holding capability constant across 11,700 trajectories. Structured interfaces improve run-to-run consistency up to 4.7 times, and the CodeAct style cuts steps 41.6% and tokens 56.3%. Extends the tool-surface-design question the `execute_code` ablation in this list opens. |
| [One Recipe, Many Harnesses: What Self-Evolution Encodes Across Languages and Models](https://arxiv.org/abs/2608.10178) | arXiv | Evaluates the same harness self-evolution method across eight languages and three base models. Gains vary across settings: general working practices transfer more readily than ecosystem-specific rules. The study records predicted benefits and possible regressions, but lacks a comparison that matches the full evolution budget against extra test-time inference. |
| [Persistent Recursive Worlds Enable Autonomous Software Evolution](https://arxiv.org/abs/2608.10450) | arXiv | Makes the software project the persistent unit (an accepted version plus a repository path) so agents can stay finite-lived while development continues indefinitely. A Rust C compiler was built over 120 hours and more than 1,000 agent episodes for about $44, surviving full agent replacement mid-project. A session-persistence design that locates continuity in the artifact rather than in agent memory. |
| [Agent Safety Should Be a Runtime Contract](https://arxiv.org/abs/2608.11274) | arXiv | Argues training-time alignment is structurally insufficient for agents that execute code and mutate files, and that safety must be a harness-enforced runtime contract pairing preventive controls (sandboxes, permission gates, output filters) with evidential verification over checkable trajectory evidence such as tests, logs, and diffs. The position-paper companion to ActPlane and SHarD in this list. |

### How This Paper Differs

> While the projects above focus on **engineering reverse-engineering** or **practical reimplementation**, this paper provides a **systematic values → principles → implementation** analytical framework — tracing five human values through thirteen design principles to specific source-level choices, and using OpenClaw comparison to reveal that cross-cutting integrative mechanisms, not modular features, are the true locus of engineering complexity.


<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---

<details>
<summary><h2>Other Notable AI Agent Projects</h2></summary>

A broader map of the agent design space surrounding Claude Code. The [Cross-System Comparison](#cross-system-comparison-claude-code-vs-openclaw-vs-hermes-agent) above analyzes the three closest peers (Claude Code, OpenClaw, Hermes-Agent) in depth; the entries below give wider context across coding-agent peers, frameworks, memory systems, harness extensions, the MCP ecosystem, and specialized agents.

### Coding Agent CLIs and IDE Harnesses

| Repository | Launch | Focus |
|:-----------|:-------|:------|
| [**openclaw/openclaw**](https://github.com/openclaw/openclaw) [![Star](https://img.shields.io/github/stars/openclaw/openclaw.svg?style=social&label=Star)](https://github.com/openclaw/openclaw) | Jan 2026 | Local-first personal AI assistant across messaging platforms. ([Section 10 analysis](#cross-system-comparison-claude-code-vs-openclaw-vs-hermes-agent)) |
| [**NousResearch/hermes-agent**](https://github.com/nousresearch/hermes-agent) [![Star](https://img.shields.io/github/stars/nousresearch/hermes-agent.svg?style=social&label=Star)](https://github.com/nousresearch/hermes-agent) | Feb 2026 | Self-improving personal agent with cross-session memory. ([Section 10 analysis](#cross-system-comparison-claude-code-vs-openclaw-vs-hermes-agent)) |
| [**opensquilla/opensquilla**](https://github.com/opensquilla/opensquilla) [![Star](https://img.shields.io/github/stars/opensquilla/opensquilla.svg?style=social&label=Star)](https://github.com/opensquilla/opensquilla) | Jun 2026 | Token-efficient microkernel personal agent across CLI, Web UI, and chat channels; ML-classifier routing across four model cost tiers, local Markdown+SQLite memory (MEMORY.md plus dated notes with keyword and vector recall), and Bubblewrap/Seatbelt sandbox. |
| [**pewdiepie-archdaemon/odysseus**](https://github.com/pewdiepie-archdaemon/odysseus) [![Star](https://img.shields.io/github/stars/pewdiepie-archdaemon/odysseus.svg?style=social&label=Star)](https://github.com/pewdiepie-archdaemon/odysseus) | Jun 2026 | Self-hosted, local-first AI workspace from PewDiePie: autonomous agents with tools, MCP, and shell access, plus memory, deep research, and hardware-aware model serving. AGPL-3.0. |
| [**google-antigravity/antigravity-cli**](https://github.com/google-antigravity/antigravity-cli) [![Star](https://img.shields.io/github/stars/google-antigravity/antigravity-cli.svg?style=social&label=Star)](https://github.com/google-antigravity/antigravity-cli) | Jun 2026 | Google's Go rewrite that replaces Gemini CLI and shares one agent harness with the Antigravity 2.0 desktop app. Per its CHANGELOG: nested subagents to grandchild depth and beyond, with subtrajectory updates recursively relayed to the root conversation; workspace-level `.agents/hooks.json` where a pre-tool hook decides whether a tool call is permitted; and per-project permission config kept under `~/.gemini/config/projects/` rather than in the repo, taking precedence over global settings. |
| [**patriceckhart/zot**](https://github.com/patriceckhart/zot) [![Star](https://img.shields.io/github/stars/patriceckhart/zot.svg?style=social&label=Star)](https://github.com/patriceckhart/zot) | Jun 2026 | Single static Go binary harness with a deliberately minimal tool set (read, write, edit, bash). Sessions persist as JSONL transcripts that resume, branch, and compact. `/swarm` fans out background subagents that share the working directory and edit the same files, each keeping its own session file and transcript; the opt-in auto-swarm setting (off by default) additionally lets the main agent spawn them itself and injects a recap back into the main chat. `/jail` roots a sandbox at the cwd and refuses obvious escape patterns (`sudo`, `rm -rf /`, leading `cd /`, `chmod -R`). A readable non-TypeScript point of comparison for subagent and permission design. |
| [**sst/opencode**](https://github.com/sst/opencode) [![Star](https://img.shields.io/github/stars/sst/opencode.svg?style=social&label=Star)](https://github.com/sst/opencode) | Jun 2025 | Provider-agnostic terminal coding agent with ACP integration. |
| [**Aider-AI/aider**](https://github.com/Aider-AI/aider) [![Star](https://img.shields.io/github/stars/Aider-AI/aider.svg?style=social&label=Star)](https://github.com/Aider-AI/aider) | 2023 | Pair-program with LLMs in the terminal; works with most popular models. |
| [**continuedev/continue**](https://github.com/continuedev/continue) [![Star](https://img.shields.io/github/stars/continuedev/continue.svg?style=social&label=Star)](https://github.com/continuedev/continue) | 2023 | Source-controlled AI checks for IDEs with an open-source Continue CLI. |
| [**google-gemini/gemini-cli**](https://github.com/google-gemini/gemini-cli) [![Star](https://img.shields.io/github/stars/google-gemini/gemini-cli.svg?style=social&label=Star)](https://github.com/google-gemini/gemini-cli) | 2025 | Google's open-source terminal coding agent with ReAct loop and MCP support. Superseded by Antigravity CLI as of June 18, 2026; enterprise customers on paid Gemini licenses keep access. |
| [**openai/codex**](https://github.com/openai/codex) [![Star](https://img.shields.io/github/stars/openai/codex.svg?style=social&label=Star)](https://github.com/openai/codex) | 2025 | OpenAI's local terminal coding agent in Rust. |
| [**OpenHands/OpenHands**](https://github.com/OpenHands/OpenHands) [![Star](https://img.shields.io/github/stars/OpenHands/OpenHands.svg?style=social&label=Star)](https://github.com/OpenHands/OpenHands) | 2024 | Open SWE agent platform (formerly OpenDevin) with sandboxed runtime. |
| [**cline/cline**](https://github.com/cline/cline) [![Star](https://img.shields.io/github/stars/cline/cline.svg?style=social&label=Star)](https://github.com/cline/cline) | 2024 | VS Code agent with explicit Plan/Act oversight loop. |
| [**block/goose**](https://github.com/block/goose) [![Star](https://img.shields.io/github/stars/block/goose.svg?style=social&label=Star)](https://github.com/block/goose) | 2025 | Block's open-source, editor-agnostic agent with MCP-style extensions. |
| [**charmbracelet/crush**](https://github.com/charmbracelet/crush) [![Star](https://img.shields.io/github/stars/charmbracelet/crush.svg?style=social&label=Star)](https://github.com/charmbracelet/crush) | 2025 | Agentic coding TUI in Go with multi-LLM provider abstraction. |
| [**RooCodeInc/Roo-Code**](https://github.com/RooCodeInc/Roo-Code) [![Star](https://img.shields.io/github/stars/RooCodeInc/Roo-Code.svg?style=social&label=Star)](https://github.com/RooCodeInc/Roo-Code) | 2024 | VS Code multi-agent dev-team with Architect, Coder, and Reviewer modes. |
| [**bytedance/trae-agent**](https://github.com/bytedance/trae-agent) [![Star](https://img.shields.io/github/stars/bytedance/trae-agent.svg?style=social&label=Star)](https://github.com/bytedance/trae-agent) | 2025 | ByteDance's modular SWE-bench-oriented agent for software engineering tasks. |
| [**github/copilot-cli**](https://github.com/github/copilot-cli) [![Star](https://img.shields.io/github/stars/github/copilot-cli.svg?style=social&label=Star)](https://github.com/github/copilot-cli) | 2026 | GitHub Copilot's GA agentic terminal CLI; plans, builds, reviews. |
| [**badlogic/pi-mono**](https://github.com/badlogic/pi-mono) [![Star](https://img.shields.io/github/stars/badlogic/pi-mono.svg?style=social&label=Star)](https://github.com/badlogic/pi-mono) | Aug 2025 | Monorepo coding-agent toolkit — unified LLM API, TUI + web UI; OpenClaw embeds the `pi-coding-agent` SDK from here. |
| [**deepseek-ai/deepseek-harness**](https://github.com/deepseek-ai/deepseek-harness) [![Star](https://img.shields.io/github/stars/deepseek-ai/deepseek-harness.svg?style=social&label=Star)](https://github.com/deepseek-ai/deepseek-harness) | 2026 | DeepSeek's own agent harness, in developer preview, built on a plugin architecture (Cordis) under the motto "everything is a plugin." Runs via npm or from source; the surrounding ecosystem already includes third-party awesome-lists and backends that target it. A first-party harness from a major model lab, worth reading alongside the vendor CLIs above. MIT. |

### Agent Frameworks and Orchestration

| Repository | Launch | Focus |
|:-----------|:-------|:------|
| [**context4ai/agent-graph**](https://github.com/context4ai/agent-graph/tree/387f80db65bf20a61bc666b4fa885200fcedad08) | 2026 | Work contracts and fact-based routing, reviewed at v0.3.0. For a nonterminal node with defined fact conditions, a recorded completion is marked unverified when those conditions do not match. The host still supplies execution and must establish whether those facts are trustworthy. |
| [**geekan/MetaGPT**](https://github.com/geekan/MetaGPT) [![Star](https://img.shields.io/github/stars/geekan/MetaGPT.svg?style=social&label=Star)](https://github.com/geekan/MetaGPT) | 2023 | Role-based multi-agent software-company simulation (ICLR 2024 oral). |
| [**microsoft/autogen**](https://github.com/microsoft/autogen) [![Star](https://img.shields.io/github/stars/microsoft/autogen.svg?style=social&label=Star)](https://github.com/microsoft/autogen) | 2023 | Microsoft Research multi-agent conversation framework (COLM 2024). |
| [**microsoft/agent-framework**](https://github.com/microsoft/agent-framework) [![Star](https://img.shields.io/github/stars/microsoft/agent-framework.svg?style=social&label=Star)](https://github.com/microsoft/agent-framework) | 2025 | Microsoft's single successor to AutoGen and Semantic Kernel (1.0 released April 2026). At BUILD 2026 it added a built-in agent harness (context compaction, file memory, shell access) and CodeAct, which lets the model run several tool calls as one Python script instead of one at a time. |
| [**langchain-ai/langgraph**](https://github.com/langchain-ai/langgraph) [![Star](https://img.shields.io/github/stars/langchain-ai/langgraph.svg?style=social&label=Star)](https://github.com/langchain-ai/langgraph) | 2024 | Stateful graph-based multi-agent orchestration with checkpointing. |
| [**openai/openai-agents-python**](https://github.com/openai/openai-agents-python) [![Star](https://img.shields.io/github/stars/openai/openai-agents-python.svg?style=social&label=Star)](https://github.com/openai/openai-agents-python) | 2024 | OpenAI's lightweight multi-agent framework with handoffs and guardrails. |
| [**crewAIInc/crewAI**](https://github.com/crewAIInc/crewAI) [![Star](https://img.shields.io/github/stars/crewAIInc/crewAI.svg?style=social&label=Star)](https://github.com/crewAIInc/crewAI) | 2023 | Lean Python framework for role-based multi-agent collaboration, independent of LangChain. |
| [**openai/symphony**](https://github.com/openai/symphony) [![Star](https://img.shields.io/github/stars/openai/symphony.svg?style=social&label=Star)](https://github.com/openai/symphony) | Feb 2026 | OpenAI's orchestration for isolated, autonomous implementation runs. |
| [**ComposioHQ/agent-orchestrator**](https://github.com/ComposioHQ/agent-orchestrator) [![Star](https://img.shields.io/github/stars/ComposioHQ/agent-orchestrator.svg?style=social&label=Star)](https://github.com/ComposioHQ/agent-orchestrator) | 2025 | Orchestration layer for parallel AI agents with git worktree isolation. |
| [**omnigent-ai/omnigent**](https://github.com/omnigent-ai/omnigent) [![Star](https://img.shields.io/github/stars/omnigent-ai/omnigent.svg?style=social&label=Star)](https://github.com/omnigent-ai/omnigent) | Jun 2026 | Databricks' open-source "meta-harness" (Apache 2.0): Claude Code, Codex, Cursor, OpenCode, Hermes, and Pi become interchangeable backends behind one orchestration layer, and can be mixed within a single session. A policy engine stacks across three levels (server, agent, session, with the stricter session rules checked first) and governs shell commands, file edits, token spend, and tool access. Native sandboxing uses bwrap on Linux and seatbelt on macOS; disposable cloud sandboxes run on Modal, E2B, or Kubernetes. The clearest industrial argument that policy and isolation belong above the harness rather than inside a prompt. |
| [**yylo-dev/yylo**](https://github.com/yylo-dev/yylo) [![Star](https://img.shields.io/github/stars/yylo-dev/yylo.svg?style=social&label=Star)](https://github.com/yylo-dev/yylo) | Jan 2026 | Command-line orchestrator for coding agents: repeatable workflows and receipt-backed repository changes, with typed task, validation, merge, and release-readiness boundaries. Each task runs in a dedicated branch/worktree, with Pi and Codex as subagent engines. MIT. |
| [**coleam00/Archon**](https://github.com/coleam00/Archon) [![Star](https://img.shields.io/github/stars/coleam00/Archon.svg?style=social&label=Star)](https://github.com/coleam00/Archon) | Feb 2025 | Deterministic harness — YAML-defined workflows with execution audit trail. |
| [**bytedance/deer-flow**](https://github.com/bytedance/deer-flow) [![Star](https://img.shields.io/github/stars/bytedance/deer-flow.svg?style=social&label=Star)](https://github.com/bytedance/deer-flow) | 2026 | ByteDance's long-horizon "SuperAgent" harness: subagents, memory, sandboxes, skills, and a message gateway; a ground-up rewrite on LangGraph/LangChain. |
| [**QwenLM/Qwen-Agent**](https://github.com/QwenLM/Qwen-Agent) [![Star](https://img.shields.io/github/stars/QwenLM/Qwen-Agent.svg?style=social&label=Star)](https://github.com/QwenLM/Qwen-Agent) | 2023 | Alibaba Qwen's agent framework: function calling, MCP, a Docker code interpreter, and RAG; the backend behind Qwen Chat. |
| [**TencentCloudADP/youtu-agent**](https://github.com/TencentCloudADP/youtu-agent) [![Star](https://img.shields.io/github/stars/TencentCloudADP/youtu-agent.svg?style=social&label=Star)](https://github.com/TencentCloudADP/youtu-agent) | 2025 | Tencent Cloud's agent framework, built on the openai-agents SDK; agents are defined in YAML, configs can be auto-generated, and it adds Claude Code-style skills. |
| [**coze-dev/coze-studio**](https://github.com/coze-dev/coze-studio) [![Star](https://img.shields.io/github/stars/coze-dev/coze-studio.svg?style=social&label=Star)](https://github.com/coze-dev/coze-studio) | 2025 | ByteDance's open-source edition of Coze: a visual no-code/low-code platform for building, debugging, and deploying agents and workflows. |

### Memory and Persistent Context

| Resource | Date / version | Focus |
|:-----------|:-------|:------|
| [**Codex experimental context management**](https://learn.chatgpt.com/docs/config-file/config-reference) | CLI 0.153.0, September 3, 2026 | Optional working-context management using notes, searchable history, and `new_context`. Its token-budget path starts a new window without a model/server summary while retaining the compaction lifecycle. Account, backend, version, and later model-support limits are detailed in the [source notes](./docs/agent-design-space-source-notes.md#source-codex-context-management). |
| [**Codex local memories**](https://learn.chatgpt.com/docs/customization/memories) | Reviewed September 2026 | When enabled, eligible prior sessions feed background extraction and consolidation. A short index guides retrieval of detailed memory. This local store is separate from ChatGPT web memory and from experimental working-context notes; [summarizing compaction also remains in v0.153.4](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact.rs). |
| [**mem0ai/mem0**](https://github.com/mem0ai/mem0) [![Star](https://img.shields.io/github/stars/mem0ai/mem0.svg?style=social&label=Star)](https://github.com/mem0ai/mem0) | 2024 | Production memory layer with LoCoMo and LongMemEval benchmarks (arXiv:2504.19413). |
| [**letta-ai/letta**](https://github.com/letta-ai/letta) [![Star](https://img.shields.io/github/stars/letta-ai/letta.svg?style=social&label=Star)](https://github.com/letta-ai/letta) | 2023 | Stateful-agent platform with OS-style hierarchical memory paging (formerly MemGPT, COLM 2024). |
| [**MemPalace/mempalace**](https://github.com/MemPalace/mempalace) [![Star](https://img.shields.io/github/stars/MemPalace/mempalace.svg?style=social&label=Star)](https://github.com/MemPalace/mempalace) | 2026 | Local-first memory system for AI agents. |
| [**vshulcz/deja-vu**](https://github.com/vshulcz/deja-vu) [![Star](https://img.shields.io/github/stars/vshulcz/deja-vu.svg?style=social&label=Star)](https://github.com/vshulcz/deja-vu) | Jul 2026 | Cross-agent memory in a single local Go binary: indexes the session histories that 17 coding agents already write to disk and retrieves prior work by lexical search, with no LLM or embedding service. MIT. |
| [**pax-beehive/paxm**](https://github.com/pax-beehive/paxm) [![Star](https://img.shields.io/github/stars/pax-beehive/paxm.svg?style=social&label=Star)](https://github.com/pax-beehive/paxm) | Jul 2026 | Provider-neutral memory adapter carrying decisions and conventions across Codex, Claude Code, OpenCode, Pi, and MCP agents. Starts on local SQLite and swaps to backends such as Zep or Mem0 without vendor lock-in. Apache-2.0. |

### Skills and Harness Extensions

| Repository | Launch | Focus |
|:-----------|:-------|:------|
| [**addyosmani/agent-skills**](https://github.com/addyosmani/agent-skills) [![Star](https://img.shields.io/github/stars/addyosmani/agent-skills.svg?style=social&label=Star)](https://github.com/addyosmani/agent-skills) | 2025 | 22 lifecycle skills + slash commands (`/spec`, `/plan`, `/build`, `/test`, `/review`, `/ship`). |
| [**obra/superpowers**](https://github.com/obra/superpowers) [![Star](https://img.shields.io/github/stars/obra/superpowers.svg?style=social&label=Star)](https://github.com/obra/superpowers) | 2025 | Cross-harness mandatory-workflow skills framework (Claude Code, OpenCode, Codex). |
| [**mattpocock/skills**](https://github.com/mattpocock/skills) [![Star](https://img.shields.io/github/stars/mattpocock/skills.svg?style=social&label=Star)](https://github.com/mattpocock/skills) | 2026 | Author's everyday `.claude/skills` collection for real engineering -- composable TDD, diagnose, and to-issues/to-prd skills; model-agnostic, targeting Claude Code, Codex, and other coding agents. |
| [**multica-ai/andrej-karpathy-skills**](https://github.com/multica-ai/andrej-karpathy-skills) [![Star](https://img.shields.io/github/stars/multica-ai/andrej-karpathy-skills.svg?style=social&label=Star)](https://github.com/multica-ai/andrej-karpathy-skills) | 2026 | Single CLAUDE.md encoding Andrej Karpathy's four LLM-coding rules (think before coding, simplicity first, surgical changes, goal-driven execution); installable as a plugin or per-project. |
| [**lsdefine/GenericAgent**](https://github.com/lsdefine/GenericAgent) [![Star](https://img.shields.io/github/stars/lsdefine/GenericAgent.svg?style=social&label=Star)](https://github.com/lsdefine/GenericAgent) | 2025 | Minimal self-evolving autonomous agent framework — 9 atomic tools + ~100-line ReAct loop. |
| [**Leutenegger/book-to-skill**](https://github.com/Leutenegger/book-to-skill) [![Star](https://img.shields.io/github/stars/Leutenegger/book-to-skill.svg?style=social&label=Star)](https://github.com/Leutenegger/book-to-skill) | Aug 2026 | Turns a technical book PDF into a structured Agent Skill (per-chapter files, mental models, glossaries, and pattern guides loaded on demand) rather than dumping the full text into context, reporting 24 to 51 times fewer tokens. Targets Claude Code, Copilot CLI, and Amp. MIT. |
| [**xyiqq/skilldoctor**](https://github.com/xyiqq/skilldoctor) [![Star](https://img.shields.io/github/stars/xyiqq/skilldoctor.svg?style=social&label=Star)](https://github.com/xyiqq/skilldoctor) | Aug 2026 | A CI quality gate for Agent Skills: validates the SKILL.md spec, audits for prompt-injection risk and hardcoded secrets, and checks compatibility across Claude Code, Cursor, Codex, and OpenCode. One of the first lint tools built for the skills ecosystem. MIT. |

### MCP Ecosystem

| Repository | Launch | Focus |
|:-----------|:-------|:------|
| [**PrefectHQ/fastmcp**](https://github.com/prefecthq/fastmcp) [![Star](https://img.shields.io/github/stars/prefecthq/fastmcp.svg?style=social&label=Star)](https://github.com/prefecthq/fastmcp) | 2024 | Pythonic framework for building MCP servers and clients; de facto SDK. |
| [**upstash/context7**](https://github.com/upstash/context7) [![Star](https://img.shields.io/github/stars/upstash/context7.svg?style=social&label=Star)](https://github.com/upstash/context7) | 2025 | Up-to-date library-documentation MCP server for LLMs and AI code editors. |
| [**microsoft/playwright-mcp**](https://github.com/microsoft/playwright-mcp) [![Star](https://img.shields.io/github/stars/microsoft/playwright-mcp.svg?style=social&label=Star)](https://github.com/microsoft/playwright-mcp) | 2024 | Microsoft's official MCP server using accessibility-tree snapshots. |
| [**modelcontextprotocol/experimental-ext-interceptors**](https://github.com/modelcontextprotocol/experimental-ext-interceptors) [![Star](https://img.shields.io/github/stars/modelcontextprotocol/experimental-ext-interceptors.svg?style=social&label=Star)](https://github.com/modelcontextprotocol/experimental-ext-interceptors) | 2026 | Multi-language reference implementation of the proposed interceptor extension (SEP-2624), which would make middleware-style inspection and transformation of MCP requests a protocol-level primitive rather than a per-host hook system. Marked experimental and explicitly not an accepted or official MCP extension. |

### Specialized and Domain Agents

| Repository | Launch | Focus |
|:-----------|:-------|:------|
| [**666ghj/MiroFish**](https://github.com/666ghj/MiroFish) [![Star](https://img.shields.io/github/stars/666ghj/MiroFish.svg?style=social&label=Star)](https://github.com/666ghj/MiroFish) | Mar 2026 | Multi-agent swarm-intelligence simulation engine. |
| [**multica-ai/multica**](https://github.com/multica-ai/multica) [![Star](https://img.shields.io/github/stars/multica-ai/multica.svg?style=social&label=Star)](https://github.com/multica-ai/multica) | 2026 | Managed-agents platform for task assignment and skill compounding. |
| [**HKUDS/nanobot**](https://github.com/HKUDS/nanobot) [![Star](https://img.shields.io/github/stars/HKUDS/nanobot.svg?style=social&label=Star)](https://github.com/HKUDS/nanobot) | Feb 2026 | Ultra-lightweight personal AI agent from HKU-DS. |
| [**HKUDS/OpenHarness**](https://github.com/HKUDS/OpenHarness) [![Star](https://img.shields.io/github/stars/HKUDS/OpenHarness.svg?style=social&label=Star)](https://github.com/HKUDS/OpenHarness) | Apr 2026 | Open agent harness with built-in personal agent (Ohmo); academic harness reference. |
| [**karpathy/autoresearch**](https://github.com/karpathy/autoresearch) [![Star](https://img.shields.io/github/stars/karpathy/autoresearch.svg?style=social&label=Star)](https://github.com/karpathy/autoresearch) | Mar 2026 | Andrej Karpathy's autonomous AI-agent loop running nanochat training research on a single GPU. |
| [**HKUDS/CLI-Anything**](https://github.com/HKUDS/CLI-Anything) [![Star](https://img.shields.io/github/stars/HKUDS/CLI-Anything.svg?style=social&label=Star)](https://github.com/HKUDS/CLI-Anything) | Mar 2026 | "Making ALL Software Agent-Native" — wraps arbitrary software as agent-callable tools. |
| [**Panniantong/Agent-Reach**](https://github.com/Panniantong/Agent-Reach) [![Star](https://img.shields.io/github/stars/Panniantong/Agent-Reach.svg?style=social&label=Star)](https://github.com/Panniantong/Agent-Reach) | Feb 2026 | CLI giving agents access to Twitter, Reddit, YouTube, GitHub, Bilibili, Xiaohongshu. |
| [**agentscope-ai/QwenPaw**](https://github.com/agentscope-ai/QwenPaw) [![Star](https://img.shields.io/github/stars/agentscope-ai/QwenPaw.svg?style=social&label=Star)](https://github.com/agentscope-ai/QwenPaw) | Feb 2026 | Personal AI assistant from the AgentScope team. |
| [**cft0808/edict**](https://github.com/cft0808/edict) [![Star](https://img.shields.io/github/stars/cft0808/edict.svg?style=social&label=Star)](https://github.com/cft0808/edict) | Feb 2026 | OpenClaw-based multi-agent orchestration on Tang-dynasty Three Departments and Six Ministries (三省六部制) bureaucracy. |

<p align="right"><a href="#dive-into-claude-code">↑ Back to top</a></p>

</details>

---
<a href="https://star-history.dera.page/#VILA-Lab/Dive-into-Claude-Code">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://star-history.dera.page/svg?repos=VILA-Lab/Dive-into-Claude-Code&type=Date&theme=dark" />
    <source media="(prefers-color-scheme: light)" srcset="https://star-history.dera.page/svg?repos=VILA-Lab/Dive-into-Claude-Code&type=Date" />
    <img alt="Star History Chart" src="https://star-history.dera.page/svg?repos=VILA-Lab/Dive-into-Claude-Code&type=Date" />
  </picture>
</a>

## Contributing

Contributions are welcome — this space moves fast, and things slip past us.

If you come across a paper, blog post, repository, benchmark, or security write-up worth adding, feel free to open a pull request or an issue.

## Citation

<!-- <details>
<summary>BibTeX</summary> -->

```bibtex
@article{diveclaudecode2026,
  title={{Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems}},
  author={Liu, Jiacheng and Zhao, Xiaohan and Shang, Xinyi and Shen, Zhiqiang},
  journal={arXiv preprint arXiv:2604.14228},
  year={2026},
  eprint={2604.14228},
  archivePrefix={arXiv},
  primaryClass={cs.SE}
}
```


## License

This work is licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).
