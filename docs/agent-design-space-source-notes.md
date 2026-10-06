[Back to Main README](../README.md)

# Agent Systems Design Space: Source Notes

Updated October 6, 2026.

These notes explain the mechanisms, implementation choices, and version-specific conditions behind the [resource catalog](../README.md) and [design guide](./build-your-own-agent.md). [中文版](./agent-design-space-source-notes_zh.md).

Historical source notes through September 15, 2026 follow the current section.

<a id="writing-conventions"></a>

<a id="refresh-2026-10-06"></a>

## Agent design sources: October 6, 2026

Dates are in 2026 unless stated otherwise; release timestamps use UTC. ArXiv papers are preprints, and experimental results are those reported by their authors; vendor figures have not been independently checked. The Claude Code architecture analysis covers v2.1.88; newer versions below describe later changes.

<a id="refresh-2026-10-06-graph"></a>

### Work graphs and control loops

<a id="source-cursor-rollouts-verdicts"></a>

#### Cursor Rollouts: check a change after it deploys

**September 23, changelog entry for Teams and Enterprise.** [Changelog entry](https://cursor.com/changelog/rollouts-and-security-reviewer)

When a pull request opens, Rollouts posts a monitoring plan that the author can edit: risks, the intended effect, the signals to check, and missing instrumentation. On each deploy it gives a separate verdict for each environment: healthy, regression, or inconclusive. On a regression, depending on configuration, it can open a revert PR for review or hand the finding to a cloud agent, but it does not merge or roll back on its own. Keeping "inconclusive" as a verdict stops missing evidence from counting as a pass.

<a id="source-adk-abort-resume"></a>

#### Google ADK: abort, approval pause, and rerun on resume

**October 1, ADK Python 2.11.0; related change in 2.9.0, September 10.** [v2.11.0 release notes](https://github.com/google/adk-python/releases/tag/v2.11.0) · [v2.9.0 release notes](https://github.com/google/adk-python/releases/tag/v2.9.0)

ADK Python 2.11.0 lets an abort signal stop a Runner, workflow, or node gracefully, and tool nodes in a workflow now pause for user approval instead of passing an error downstream. Since 2.9.0, released September 10, a failed node reruns when the workflow resumes; before that it was replayed as completed. The release notes therefore ask for idempotent node bodies, because a node that performs a side effect and then fails repeats that effect on every resume. These notes cover ADK Python only.

<a id="source-subgoal-authorization"></a>

#### Subgoal authorization: treat replanning as a change of authority

**October 4, arXiv v1.** [Paper](https://arxiv.org/abs/2610.04975v1)

Zhu and Wang treat creating, replacing, delegating, or joining a subgoal as an authorization event. Each change needs a structured check, bound to the current policy and state version, that the new continuation stays within the approved task; protected effects are checked again at commit. Per-call permission checks miss this, because two individually allowed actions can together violate the task. The evaluation uses finite structured domains and synthetic cases, so this is a research prototype, not a deployed control.

<a id="source-opencollab-adherence"></a>

#### OpenCollab: does the declared organization actually run?

**September 29, arXiv v1.** [Paper](https://arxiv.org/abs/2609.38345v1)

OpenCollab uses the event record to measure whether a declared multi-agent organization, with its role boundaries and communication topology, actually happens at run time. With unconstrained defaults, 47.2% of runs followed the declared structure; lead agents spent the budget themselves instead of delegating, or bypassed teammates. Restricting tool boundaries raised this above 90%, although task success fell in the read-only setting. A topology in the configuration is a request, so check the event record before attributing results to it.

<a id="source-claude-loop-wakeups"></a>

#### Claude Code: loop wakeups are runtime state

**September 23 to October 5, v2.1.281 to v2.1.290.** [CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [v2.1.290](https://github.com/anthropics/claude-code/releases/tag/v2.1.290)

These releases fix how `/loop` wakeups and scheduled tasks survive compaction, resume, hand-off to the background, updates, and container restarts. Since v2.1.284, self-paced loops write each status update and the stop outcome as visible text. A pending wakeup has its own lifetime and can be lost or duplicated at each of these boundaries. The source is release notes, not a design document.

<a id="refresh-2026-10-06-runtime"></a>

### Runtime and coordination

<a id="source-claude-stop-scopes"></a>

#### Claude Code: stopping a turn versus stopping background work

**September 17 to October 5, v2.1.275 to v2.1.290.** [CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [v2.1.281](https://github.com/anthropics/claude-code/releases/tag/v2.1.281)

In v2.1.275, the send-now key (ctrl+enter, which sends queued messages while Claude is still working) interrupted the current turn; from v2.1.281 it moves running tools to the background instead of cancelling the turn. In the VS Code extension, Stop and Escape end only the current turn; background agents keep running and can be stopped one by one from the agent map (v2.1.286). Background shell commands gained a time limit in v2.1.285 (default 30 minutes, maximum 2 hours), and from v2.1.288 it applies only to unattended sessions such as `-p`, the Agent SDK, CI, and cloud sessions. From v2.1.287, replies from `claude agents` arrive as queued messages and slash commands other than `/stop` run when the current turn ends, although v2.1.290 applies `/model`, `/effort`, and `/rename` at once to a busy background session.

<a id="source-vscode-remote-agent-hosts"></a>

#### VS Code 1.140: delegate to remote agent hosts

**September 30, VS Code 1.140; experimental and off by default.** [Release notes](https://code.visualstudio.com/updates/v1_140)

New tools let an agent list connected remote agent hosts with their capacity and session load, start a session on a named host or on one that meets operating system, memory, and CPU requirements, check its status, and exchange messages. A remote session has no workspace unless one is named, the tools do not copy the originating workspace, and normal approvals still apply. Remote agents report back with `send_remote_message`; their final answers are not forwarded automatically, and the coordinating window must stay open for messages to flow. The release also raises orchestration limits, and reaching one blocks new orchestration actions without interrupting running work.

<a id="source-copilot-dynamic-workflows"></a>

#### Copilot dynamic workflows: limits, pause, and resume

**October 1, public preview.** [Changelog](https://github.blog/changelog/2026-10-01-dynamic-workflows-in-copilot-cli-and-the-copilot-app) · [Concepts](https://docs.github.com/en/copilot/concepts/agents/dynamic-workflows)

In Copilot CLI, the Copilot app, and the Copilot SDK, a workflow is code that defines the steps, when agents are involved, and how their results are used. Reaching the concurrency limit makes new agents wait; the limits on total agents, active running time, and approximate AI credits stop the run but keep its status and saved results, and a raised limit still counts usage from before the stop. On resume, saved results from completed steps and subagents can be reused, while unsaved work may run again. Workflow subagents inherit the session's permission grants, so a session-long grant made for one subagent applies to the others.

<a id="source-exactly-once-tool-contract"></a>

#### Exactly-once: which layer prevents duplicate writes

**September 24, arXiv v1.** [Paper](https://arxiv.org/abs/2609.29095v1)

This study, *Where Does Exactly-Once Live?*, injects faults at tool boundaries, such as a write that commits after the agent has stopped waiting for it, or a request delivered twice, and grades each episode against a ledger of committed effects, including runs under GitHub Copilot CLI, Hermes, and Codex CLI. The author reports that frontier models told to act exactly once almost never duplicated a write whose acknowledgement was lost, but often duplicated requests still in flight or delivered twice; offering an idempotency key on every write cut duplicates from 28% to 4%, and the three harnesses behaved almost the same. In 90% of episodes that produced a duplicate, the agent reported the task as completed. These are single-author results on a synthetic benchmark, and the paper says code and data will be released on publication.

<a id="source-planarian-statepoints"></a>

#### Planarian: one restore point for local and remote state

**September 28, arXiv v1.** [Paper](https://arxiv.org/abs/2609.35366v1) · [StateFork](https://arxiv.org/abs/2609.38648v1)

Planarian is a research prototype whose statepoints cover a sandbox's files and processes together with remote changes made through MCP. It takes incremental local checkpoints and records a compensating action for each remote call; rollback applies those actions in reverse order and restores the local snapshot, and fork creates isolated branches. Remote requests that cannot be made compensable are rejected before they run, and compensation is implemented only for SQL operations on a database MCP server. StateFork (arXiv, September 29) studies how to branch and restore terminal sessions so an agent can explore alternatives.

<a id="source-codex-queued-reconnect"></a>

#### Codex CLI: resolve uncertain sends before resending

**September 29 and October 1, Codex CLI 0.159.0 and 0.160.0.** [0.160.0 release notes](https://github.com/openai/codex/releases/tag/rust-v0.160.0) · [0.159.0 release notes](https://github.com/openai/codex/releases/tag/rust-v0.159.0)

After a reconnection, Codex CLI 0.160.0 resumes unsent queued messages only once submissions with an uncertain outcome have been resolved, so they are not sent twice. Version 0.159.0 adds an opt-in `instant_interrupt` setting that lets new input steer Codex during a model response or a long-running code-mode call.

<a id="refresh-2026-10-06-harness"></a>

### Harness and application interfaces

<a id="source-agents-api-computer-use"></a>

#### Agents API computer use: origin approval and sign-in

**September 29, Agents API public beta.** [API changelog](https://developers.openai.com/api/docs/changelog) · [Computer use guide](https://developers.openai.com/api/docs/guides/agents-api/tools/computer-use)

OpenAI added computer use to the Agents API, with the browser running in an OpenAI-hosted environment. Each new website origin needs user approval, separate from the environment's network policy, and origin approval does not confirm individual actions such as a purchase. Sign-in values go through a dedicated event that stays outside the model's input and session history, and only the main agent, not a subagent, can request sign-in. After a disconnect, the application rereads the session's required actions instead of resending the task or approvals.

<a id="source-cursor-token-efficiency"></a>

#### Cursor: remove harness work the model no longer needs

**September 23, engineering article.** [Article](https://cursor.com/blog/improved-token-efficiency)

Cursor reports a 7% cut in user token costs from harness changes without reducing agent quality; it describes A/B testing individual changes on production traffic. As models improved, it removed about 66% of the system prompt, loaded rarely used built-in tools on demand, placed cache breakpoints after the parts of each request that rarely change, and dropped instructions that pushed subagent use because newer models had learned that pattern in training. The figures are vendor-reported, and the article does not publish its evaluation details.

<a id="source-harness-design-components"></a>

#### Harness components: value depends on the model and context budget

**September 17, arXiv v1; related study September 30.** [Paper](https://arxiv.org/abs/2609.20804v1) · [ML engineering study](https://arxiv.org/abs/2609.40303v1)

The study holds a coding harness's loop fixed and varies planning, action space, and context management across four open models, four context budgets, and 176 settings on SWE-Bench Verified and Terminal-Bench 2.1. Context management mattered most under tight budgets, mainly by preventing overflow; trimming stale tool output before LLM summarization matched the other managed strategies in success and had the lowest cost in seven of eight model and benchmark panels, and a recall tool for trimmed output was rarely used. Planning raised accuracy for weaker models but mainly cut cost for stronger ones; each setting ran once, and closed frontier models were not tested. A September 30 study on machine-learning engineering tasks found that four open-source harnesses gave no advantage over a minimal coding-agent session with the same strong backbone, while weaker backbones still benefited from workflow priors.

<a id="source-zcode-shared-runtime"></a>

#### ZCode: three interfaces built on one repository's agent runtime

**September 2026, source release; earliest public commit September 20, README notes v3.14.3 on September 23.** [Repository](https://github.com/zai-org/ZCode) · [CLI plugin documentation](https://github.com/zai-org/ZCode/blob/main/apps/zcode-cli/README.md)

Z.ai's open-source coding workspace provides desktop, browser, and terminal interfaces built on the agent CLI and runtime in the same repository. The web interface listens only on localhost by default and generates an access token when bound to other addresses. Plugins are local bundles that add skills, custom commands, and MCP servers. The README does not describe a permission or sandbox model; the license is Apache-2.0.

<a id="source-opus-5-5-instruction-cleanup"></a>

#### Opus 5.5 guide: migrate by cleaning up instructions

**September 22, official article.** [Usage guide](https://claude.dev/blog/getting-the-most-out-of-opus-5-5/)

Anthropic's Opus 5.5 guide recommends removing "think carefully" instructions, stating in CLAUDE.md when to keep going and when to stop and ask, and keeping a long task's checklist in a file because compaction summarizes older turns. Claude Code moves most messages flagged by safeguards to an older model and continues the session there, unless the user sets `/config` to ask first. This is usage guidance, not an engineering evaluation.

<a id="refresh-2026-10-06-context"></a>

### Context and memory

<a id="source-claude-agents-md-fallback"></a>

#### Claude Code: AGENTS.md as a fallback instruction file

**September 18, v2.1.277; extended September 23 (v2.1.281) and fixed October 5 (v2.1.290).** [v2.1.277 release notes](https://github.com/anthropics/claude-code/releases/tag/v2.1.277) · [Memory documentation](https://code.claude.com/docs/en/memory)

Claude Code reads AGENTS.md as project instructions when the working directory and its parents have no CLAUDE.md; a CLAUDE.local.md also counts as a CLAUDE.md for this check. A "Project instructions" setting offers four modes: CLAUDE.md, falling back to AGENTS.md when there is none (the default); both files; CLAUDE.md only; or managed instructions only. The documentation lists differences from CLAUDE.md: InstructionsLoaded hooks do not fire, and an external @import loads only if it was already approved. Version 2.1.281 extends support to Bedrock, Vertex, Foundry, gateways, and sessions without telemetry, and 2.1.290 attaches a subdirectory's AGENTS.md when a file under it is @-mentioned.

<a id="source-codex-instruction-refresh"></a>

#### Codex: when edited instructions take effect

**September 22, Codex rust-v0.156.0.** [Release notes](https://github.com/openai/codex/releases/tag/rust-v0.156.0) · [PR #44675](https://github.com/openai/codex/pull/44675) · [PR #44701](https://github.com/openai/codex/pull/44701) · [PR #46577](https://github.com/openai/codex/pull/46577)

Codex 0.156.0 reloads global instructions at each model-request boundary, including after tool calls within a turn, so edits to a global AGENTS.md apply during a running session; repository instructions are rediscovered only when the environment selection or trust level changes. Hosts can add thread-scoped instructions, capped at about 10,000 estimated tokens and rejected rather than truncated when larger. New subagents inherit the parent's applied snapshot, and later updates reach running descendants only when the provider opts in. The thread instruction provider is a host interface, not a CLI user setting.

<a id="source-claude-auto-memory-guards"></a>

#### Claude Code: recalled memory as untrusted input

**September 28 and 29, v2.1.284 and v2.1.285.** [v2.1.284 release notes](https://github.com/anthropics/claude-code/releases/tag/v2.1.284) · [v2.1.285 release notes](https://github.com/anthropics/claude-code/releases/tag/v2.1.285)

Claude Code 2.1.284 neutralizes invisible characters, and tags that imitate Claude Code's own markup, in MEMORY.md and recalled memory notes before they reach the model. Version 2.1.285 prevents a background session, or a session started by one of Claude Code's own tools, from turning auto memory on; turning it off still works there. The memory documentation says to turn it on from a session started directly in a terminal. Neutralizing these markers does not by itself stop every form of memory poisoning.

<a id="source-copilot-memory-autofix"></a>

#### Copilot Memory: written by one feature, used by others

**September 25, public preview.** [Changelog](https://github.blog/changelog/2026-09-25-agentic-autofix-now-uses-copilot-memory) · [Copilot Memory documentation](https://docs.github.com/en/copilot/concepts/agents/copilot-memory)

Agentic autofix reads Copilot Memory when fixing security alerts and stores each fix pattern as a memory that other Copilot features, such as code review and cloud agent, can use. The current documentation says repository facts carry citations to supporting code and are checked against the current branch before use. Only users with write access create them, they are used only in that repository, and unused entries are deleted after 28 days. The citation check and retention rule are documented behavior of Copilot Memory, not new in this release.

<a id="source-vibemembench"></a>

#### VibeMemBench: memory systems on repository tasks

**September 20, arXiv v1.** [Paper](https://arxiv.org/abs/2609.23570v1)

VibeMemBench tests memory systems on 111 repository coding targets with executable checks. Injecting experience already verified as useful raised resolution on four of five held-out solvers by 1.1 to 4.5 points, but when four existing memory systems built and retrieved experience from the same history, eleven of twelve solver and system pairings did not exceed the matched no-memory baseline. The authors trace the main failure to the form in which records are supplied. Targets were selected where injection helped, retrieval happens once before each run, and Claude and GPT models were not evaluated.

<a id="refresh-2026-10-06-authority"></a>

### Tools and authority

<a id="source-claude-code-mods"></a>

#### Claude Code mods: in-process extensions that can approve calls

**October 1, first listed in the v2.1.287 CHANGELOG.** [Article](https://claude.com/resources/articles/claude-code-mods) · [Mods overview](https://code.claude.com/docs/en/plugins/mods/overview) · [Organization controls](https://code.claude.com/docs/en/plugins/mods/admin) · [v2.1.287 release notes](https://github.com/anthropics/claude-code/releases/tag/v2.1.287)

Mods are plugin functions that run inside the Claude Code process and can observe, rewrite, or answer tool calls, prompts, and interface events; they are not sandboxed, and in an untrusted directory no mod loads before the user answers the trust prompt. A mod can approve a tool call before the permission prompt appears, but it cannot change what the prompt shows. Where the built-in `sec-default` guard loads, deny rules (unless the administrator sets `allowModsToOverrideDenyRules`) and managed `PreToolUse` hooks still take precedence, and the guard refuses users' mods if it cannot read managed settings; a user's mod can still approve a call that an `ask` rule would prompt for, and in auto mode that call skips the classifier. Deny rules do not cover a mod's own file and process calls, and v2.1.290 adds a `ceiling` field to `tool.check` for the approval level the organization requires.

<a id="source-claude-managed-policy-precedence"></a>

#### Claude Code: repository settings cannot widen managed policy

**September 24 to October 5, v2.1.282 to v2.1.290; advisory September 29.** [CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [GHSA-gfvf-j8jh-jxxw](https://github.com/anthropics/claude-code/security/advisories/GHSA-gfvf-j8jh-jxxw)

Project settings can no longer widen or turn off an admin-required sandbox or extend a strict allowlist (v2.1.285), and under `allowManagedPermissionRulesOnly`, repository, user and `--add-dir` skills and commands can no longer pre-approve their own tools (v2.1.282), and plugins keep that pre-approval only from an official or admin-vouched source (v2.1.284). One invalid nested value no longer voids a whole managed `permissions`, `autoMode`, `worktree`, `attribution` (v2.1.282) or `sandbox` (v2.1.283) block; for `sandbox` the invalid value fails closed, and mistyped boolean lock keys now apply the lock (v2.1.282). The release notes state one exception to failing closed: if the OS denies reading the managed settings file, v2.1.285 warns and starts without that file's policies, while other read errors and unparseable files still block all sessions. The September 29 advisory CVE-2026-103012, fixed in 2.1.260, describes a stored API key causing a session to run without its organization's server-delivered policy; MDM and file-based managed settings were not affected.

<a id="source-claude-auto-mode-default"></a>

#### Claude Code: auto mode by default, and sandboxed commands still reviewed

**September 23 to 29, v2.1.281, v2.1.284, and v2.1.285.** [v2.1.284 release notes](https://github.com/anthropics/claude-code/releases/tag/v2.1.284) · [CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

From v2.1.284, interactive sessions with no configured permission mode start in auto mode on every plan and provider; `permissions.defaultMode` still overrides it. Version 2.1.285 extends this default to `claude -p` and Python Agent SDK sessions on third-party providers or with telemetry off. Where the auto-mode classifier runs server-side, v2.1.281 also holds read-only and sandboxed shell commands for its review, so a sandbox no longer means a command skips the classifier.

<a id="source-copilot-local-sandbox-policy"></a>

#### Copilot: local sandbox, default enablement, and per-app approval

**September 23, September 24, and October 1; sandbox and computer use in public preview.** [Local sandboxing](https://github.blog/changelog/2026-09-23-local-sandboxing-in-the-github-copilot-app/) · [Default enablement](https://github.blog/changelog/2026-09-24-default-enablement-of-copilot-features-for-copilot-business-and-enterprise/) · [Computer use](https://github.blog/changelog/2026-10-01-github-copilot-can-now-interact-with-desktop-apps/)

The Copilot app's local sandbox is configured per project for files, network, and Git or GitHub CLI credentials; enterprise settings can make the effective policy stricter, and if the OS cannot enforce the requested policy the sandboxed shell fails instead of running unsandboxed. It is off by default, is configured separately from Copilot CLI, and does not apply to cloud sandboxes or remote host sessions. A September 24 policy for Business and Enterprise lets administrators choose whether eligible generally available features left unconfigured, current and future, including the MCP servers policy, are enabled, disabled, or left to organizations; it takes effect October 22 and keeps choices already made. Computer use in Copilot CLI and the Copilot app on macOS and Windows asks before controlling each app and keeps a reviewable always-allow list.

<a id="source-codex-network-revocation"></a>

#### Codex CLI: network rules for the life of a connection

**September 25, Codex CLI 0.157.0.** [Release notes](https://github.com/openai/codex/releases/tag/rust-v0.157.0)

Codex CLI 0.157.0 applies network restrictions across redirects and to ongoing HTTP and WebSocket traffic, and cancels a connection when a policy change revokes access. A network grant is therefore checked while a connection lasts, not only when it opens.

<a id="source-anthropic-checked-vs-effective"></a>

#### Anthropic advisories: what was checked versus what took effect

**September 25 and October 5, security advisories.** [GHSA-v234-4jrq-mgg6](https://github.com/anthropics/claude-code/security/advisories/GHSA-v234-4jrq-mgg6) · [GHSA-5j29-h97v-84ch](https://github.com/anthropics/claude-code/security/advisories/GHSA-5j29-h97v-84ch)

Claude Desktop blocks opening certain run-on-open file types directly from a Cowork shared folder, but on macOS the list missed one such type, so a file written by a compromised or prompt-injected agent inside the sandbox could run commands on the host when the user opened it (affected from 1.1.3918, fixed in 1.15962.0). CVE-2026-103435 describes Claude Code checking at permission time that a write path is inside the project but resolving it again at write time; an attacker who can write to a shared workspace and wins the race can swap in a symlink and send the write outside the project. That issue was fixed in 2.1.129, well before the October 5 disclosure. A shared folder between sandbox and host is a return path that needs its own host-side control.

<a id="source-gitspawn-background-git"></a>

#### GitSpawn: background git commands before trust

**September 1, research disclosure.** [Disclosure](https://www.manifold.security/blog/ai-coding-agents-git-hijack) · [goose advisory](https://github.com/aaif-goose/goose/security/advisories/GHSA-r5pp-p5r8-466r)

Manifold Security reported that several CLI coding agents ran `git status` or `git diff` at startup or early in a session to gather context without stripping the repository's own git configuration, so a setting such as `core.fsmonitor` ran a repository-chosen command as the user, outside the sandbox, with no approval prompt. In Claude Code 2.1.193 this happened before the workspace trust prompt was accepted; it was fixed by 2.1.196. A second Claude Code finding, on the `claude ultrareview` path and using a different git setting, was still unpatched on 2.1.252 when the post was published on September 1. The attack needs a repository delivered as files with its `.git` directory, such as an archive or a synced folder, not a normal clone. The authors report eight findings across seven agents, four unpatched at publication; goose fixed its case in 1.44.0 (CVE-2026-72718).

<a id="source-approval-scope-lifetime"></a>

#### Approval laundering: what an approval covers and how long it lasts

**September 23, 27, and 30, arXiv v1.** [Agent Approval Laundering](https://arxiv.org/abs/2609.28586v1) · [When Consent Outlives Context: Residual Authority Replay in Long-Lived Agents](https://arxiv.org/abs/2609.33910v1) · [Approval Laundering](https://arxiv.org/abs/2609.38983v1)

Agent Approval Laundering (September 23) shows that an approval record names the entry command while its workflow, such as package lifecycle hooks, can have other effects, and it proposes attaching a prediction of the workflow's effects to the approval record before the user approves. When Consent Outlives Context: Residual Authority Replay in Long-Lived Agents (September 27) reports that approvals kept across tasks raise prompt-injection success by up to 35.1 percentage points on AgentDojo cases. Approval Laundering (September 30, single author) classifies six ways an executed action can differ from the approved one; a keyed approval token removed delegation mismatches in its tests but did not fix scope or argument mismatches. Two of the papers build on Claude Code's PreToolUse hook, one to measure approval mismatches and one to carry its approval record.

<a id="refresh-2026-10-06-evaluation"></a>

### Evaluation and evolution

<a id="source-harness-buy-rerun-noise"></a>

#### What Does a Harness Buy?: compare against rerun noise

**October 3, arXiv v1.** [Paper](https://arxiv.org/abs/2610.04433v1)

The study holds the model fixed and compares Claude Code, mini-SWE-agent, and OpenCode on SWE-bench Verified, using reruns of identical configurations as the noise baseline. On the 45 hardest tasks, swapping the harness changed as many task outcomes as rerunning the same harness. The measurable harness effects were ways to lose tasks, such as no recovery after an output cap, and per-task cost differed up to threefold, mainly from the system prompt and tool schemas resent on every step. Results cover one benchmark, and the Claude model ran only inside Claude Code.

<a id="source-frozen-judges"></a>

#### Frozen Judges: judge error moves with the agent version

**September 28, arXiv v1; v2 September 29.** [Paper](https://arxiv.org/abs/2609.34198v2)

A fixed LLM judge can make version-dependent errors when it compares an agent release with its predecessor. On SWE-bench Verified, for several version pairs, confidence intervals based only on judge scores showed an upgrade that test execution could not confirm, although judge rankings correlated well with the reference, and failed patches from stronger agents were accepted more often. The author recommends using judges to screen comparisons and basing release decisions on a randomly sampled, labeled audit of current outputs. Independent human patch review is still pending.

<a id="source-self-healing-harness"></a>

#### Self-Healing Harness: keep a rule only if prior successes hold

**September 21, arXiv v1.** [Paper](https://arxiv.org/abs/2609.24130v1)

The agent writes candidate rules and an external runtime decides which persist: a rule is kept only if it fixes the failure that triggered it without regressing protected cases that previously succeeded, and a separate guard retests the accumulated rule set. Of 383 proposals rejected by replay, 211 fixed their triggering failure while breaking a protected case. The study did not compare against admitting the same rules without the gate, and it replays at most two protected cases per round, so the 211 are detected conflicts, not total incidence.

<a id="source-overclaiming-transcripts"></a>

#### Overclaiming: reports of work the transcript does not show

**September 17, arXiv v1; v3 September 22.** [Paper](https://arxiv.org/abs/2609.20812v3)

The study defines overclaiming as a final report of work that the agent's own transcript shows it did not do, such as claiming to have read an unopened file. File coverage is measured from transcripts, and an LLM judge classifies the report. Across five review scenarios run in production CLIs, agents left required files unread in about two thirds of runs, most of those incomplete runs claimed a full review or did not disclose the gap, and requiring subagents raised coverage but not honest reporting. The scenarios were tuned against Claude Opus.

<a id="source-terminal-bench-hardness"></a>

#### Terminal-Bench hardness: a zero pass rate needs an audit

**September 20, arXiv v1.** [Paper](https://arxiv.org/abs/2609.26826v1)

The paper audits tasks that no agent passed in a frozen Terminal-Bench 3 production record, checking in order whether the reference solution passes, whether infrastructure failures dominate, whether a verifier bypass exists, and whether solvability is supported. Of 125 all-fail tasks, 78 remained candidates for genuinely unsolved; the rest had broken reference solutions, infrastructure problems, bypass-only passes, or unproven solvability. Of the 78, 53 rest on a single reference-solution run, so the label is narrow and does not prove intrinsic difficulty.

<a id="source-deltaselect-ab"></a>

#### DeltaSelect: small fixed task sets for A/B comparisons

**September 17, arXiv v1.** [Paper](https://arxiv.org/abs/2609.19607v1)

DeltaSelect selects a small fixed task set for repeated baseline-versus-candidate comparisons during development, not for model rankings. Tasks are ranked by how reliably a single run tracks full-benchmark results, and the set, calibration, and prices are frozen before the first comparison; the baseline must be rerun in the exact harness and version being changed. The author notes that the selector was not compared with random selection at equal cost and that repeated tuning on a small set can overfit it.

<a id="source-claude-build-eval-hillclimb"></a>

#### Claude API skill: build an evaluation, then keep or revert each patch

**September 28, official article.** [Article](https://claude.dev/blog/automating-eval-design-and-hillclimbing/)

`/claude-api build-eval` builds an evaluation in the codebase and checks grader consistency and infrastructure failures such as timeouts and truncation. `/claude-api hillclimb` splits cases into train and held-out test sets and first checks that evaluation noise is smaller than the smallest gain worth acting on. Each round proposes one patch, which is reverted if the training score rises while the held-out score stays flat or if either regresses; when the score stalls for two or three rounds, the loop sorts the remaining training failures by cause and continues only on the legitimate ones, and it recommends against merging when the final gain is within noise. The examples are Anthropic's own and are not independently replicated.

<a id="source-skill-revision-study"></a>

#### Agent Skill Evolution: what SKILL.md revisions change

**October 4, arXiv v1.** [Paper](https://arxiv.org/abs/2610.04832v1)

The study compares the first and last versions of 2,608 SKILL.md files. The authors report that adding automatically checkable rules raised the share of episodes in which four agents took the required action by 0.23 on average (on a 0 to 1 scale), and that about half of that gain remained when the skill body loaded on demand.

<a id="source-langsmith-engine-fix-validation"></a>

#### LangSmith Engine v2: reproduce a failure before review

**September 24, vendor article; fix validation in private beta.** [Article](https://www.langchain.com/blog/langsmith-engine-v2-redteam)

Engine first reproduces a failure in LangSmith Deployment, then tests the fix on the same inputs before passing it to human review. The article describes checks on the failing inputs only, not on cases that previously succeeded.

<a id="refresh-2026-09-15"></a>

## Agent design sources: September 15, 2026

Dates are in 2026 unless stated otherwise; release timestamps use UTC. ArXiv papers are preprints, and experimental results are those reported by their authors. The Claude Code architecture analysis covers v2.1.88; newer versions below describe later changes.

<a id="refresh-2026-09-15-graph"></a>

### Work graphs and control loops

<a id="source-mastra-factory-stage-rules"></a>

#### Mastra Factory: stage rules and approval objects

**September 8, beta announcement.** [Beta announcement](https://mastra.ai/blog/announcing-mastra-factory-beta) · [Board rules](https://factory.mastra.ai/configure/boards-and-rules) · [Approval documentation](https://factory.mastra.ai/using/work-and-approvals)

Mastra Factory’s current documentation separates allowed stage transitions from the actions that run on entry and exit, and uses transition policies for approval. It also tracks PR review separately from task completion. This makes transition rules and task acceptance separate choices.

<a id="source-trace2flow-review"></a>

#### Trace2Flow: inspect completed work as a graph

**September 11, arXiv v1.** [Paper](https://arxiv.org/html/2609.13136v1)

Trace2Flow turns a completed agent run into an editable workflow graph. Users can inspect the recorded inputs and outputs of each step, change dependencies, and rerun steps for a related task.

<a id="source-ready-turn-release"></a>

#### Ready turns: decide when to submit work

**September 10, arXiv v1.** [Paper](https://arxiv.org/html/2609.10964v1)

A ready model turn can wait before it enters a shared inference engine. This scheduler chooses which turn to submit and limits submitted work that has not finished. Replay experiments on software-engineering agent traces measure whole-workflow tail latency under contention. Its model assumes submitted turns cannot be withdrawn.

<a id="source-jira-agent-loops"></a>

#### Jira Agent loops: a backlog drives the outer loop

**September 10, private early access announcement.** [Announcement](https://www.atlassian.com/blog/jira/governed-agent-loops)

Jira Agent loops scan for well-defined, unassigned work, delegate implementation and testing, and produce PRs for review. Developers retain the merge decision. Task selection comes from the shared backlog, and implementation produces a reviewable change.

<a id="source-trove-route-editing"></a>

#### TROVE: revise the remaining route

**September 4, arXiv v1.** [Paper](https://arxiv.org/html/2609.05019v1)

TROVE executes one skill from a proposed route, then uses the observed result to keep the next step, insert a local response, or replace the remaining route. Completed work and its artifacts remain available. This makes route editing explicit while preserving completed work.

<a id="refresh-2026-09-15-runtime"></a>

### Runtime and coordination

<a id="source-cursor-projects"></a>

#### Cursor Projects: continuity across agents and machines

**September 10, beta rollout.** [Announcement](https://cursor.com/changelog/projects)

A cloud coordinator plans work and delegates implementation, then returns results for user review. The project synchronizes a set of files across its cloud and local agents. Event subscriptions can start further work. Project continuity can span individual sessions and machines.

<a id="source-vscode-message-queue"></a>

#### VS Code: queue messages at a turn boundary

**September 9, VS Code 1.137.** [Release notes](https://code.visualstudio.com/updates/v1_137) · [Session orchestration documentation](https://code.visualstudio.com/docs/agents/run/sessions/manage-sessions)

VS Code 1.137 queues agent messages sent to a busy chat. It starts them in send order after the active turn succeeds. Cross-session sends through the Agent Host session tools still require user confirmation. Delivery order and processing time are part of the coordination contract.

<a id="source-vscode-automation-lifetimes"></a>

#### VS Code Automations: schedules and runs have different lifetimes

**VS Code 1.137, September 9; guide page dated September 8.** [Automation guide](https://code.visualstudio.com/docs/agents/run/automations)

VS Code Automations is in Preview. Schedules need an awake machine and the relevant Agent Host process or VS Code window. Each automation runs one session at a time. Disabling its schedule leaves the active run in progress. Saved schedules, active work, and the required host have distinct lifetimes.

<a id="source-fusion-model-pairs"></a>

#### Fusion: tune collaboration for each model pair

**September 11, Desktop and CLI announcement.** [Desktop and CLI announcement](https://cognition.com/blog/local-fusion)

The lead and sidekick keep separate contexts and exchange briefs, results, and feedback. Cognition adjusts task brief detail, whether workers can question instructions, and exploration duties for each model pair.

<a id="refresh-2026-09-15-harness"></a>

### Harness and application interfaces

<a id="source-agents-api-boundaries"></a>

#### Agents API: separate the harness from its environment

**September 10, public beta.** [Announcement](https://openai.com/index/introducing-the-agents-api/) · [Overview](https://developers.openai.com/api/docs/guides/agents-api/overview)

OpenAI’s Agents API exposes the Codex harness as a managed service in public beta. Applications provide tools and choose an execution environment, while OpenAI runs the agent loop and manages its sessions, context, and recovery. This creates an application interface around the complete agent loop.

<a id="source-astra-skill-selection"></a>

#### GPT-6 Astra: skill selection and completion conditions

**September 11, official developer article.** [Developer guide](https://developers.openai.com/blog/rethinking-skills-and-prompts-for-gpt-6-astra)

For GPT-6 Astra, OpenAI recommends short skill descriptions, task-specific document references, and explicit completion conditions. When too many skills are present, Codex shortens the descriptions shown to the model. Old instructions can also add needless work or make the agent stop too soon.

<a id="source-claude-prompt-audit"></a>

#### Claude API skill: check prompts during model migration

**September 8, official article.** [Model migration workflows](https://claude.com/blog/reducing-cost-and-improving-performance-with-claude-platform)

Anthropic describes prompt-audit, hillclimb, and cost-optimize workflows in the Claude API skill. They help check old prompt rules and tune model use. Given an evaluation, hillclimb uses training cases to guide changes and a held-out test set to assess the final configuration.

<a id="source-claude-programmatic-continuation"></a>

#### Claude programmatic calls: pause and continue a program

**Beta November 24, 2025; no beta header from February 17, 2026.** [Protocol documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling) · [Release history](https://platform.claude.com/docs/en/release-notes/overview)

Claude can run Python that calls tools, pauses for client results, and resumes in the same code execution container. The client returns the container ID with pending tool results; only the program’s final output enters model context.

<a id="refresh-2026-09-15-context"></a>

### Context and memory

<a id="source-codex-memory-v2"></a>

#### Codex memory v2: versioned stores and evidence selection

**September 8, main-branch commit 2cbbf0c9.** [Main-branch change](https://github.com/openai/codex/commit/2cbbf0c9b542a36a1c3284b5e804917635b6f666)

Codex's September 8 main-branch code adds a separate memory v2 store and optional writes to both versions; the selected version supplies context. V2 prioritizes user messages and answers to agent questions within the extraction input budget, then builds task summaries. The prompts require task-specific corrections to stay with the task and explicit correction or deletion notes to be applied during consolidation. Since [stable 0.155.0](https://github.com/openai/codex/releases/tag/rust-v0.155.0) (September 17), the [memory configuration](https://github.com/openai/codex/blob/rust-v0.155.0/codex-rs/config/src/types.rs) accepts `version` and `dual_write`; v1 remains the default through 0.160.0, and the configuration reference did not list the two keys as of October 6, 2026.

<a id="source-claude-tag-recall-scope"></a>

#### Claude Tag: channel notes and workspace notes

**September 10, Claude Code v2.1.268.** [Release notes](https://github.com/anthropics/claude-code/releases/tag/v2.1.268) · [Help page](https://support.claude.com/en/articles/15594475-what-is-claude-tag)

The Claude Code 2.1.268 release notes report a narrower memory scope for Claude Tag. Each public channel keeps its own notes, and Claude no longer recalls notes from other public channels. Workspace notes remain shared.

<a id="source-claude-stable-prompts-current-state"></a>

#### Claude Code: stable requests and current facts

**September 9–11, v2.1.267–v2.1.269.** [v2.1.267 release notes](https://github.com/anthropics/claude-code/releases/tag/v2.1.267)

By default, Claude Code 2.1.267 records system prompts and tool definitions once for subagents and custom-prompt sessions. [Version 2.1.268](https://github.com/anthropics/claude-code/releases/tag/v2.1.268) extends stable tool lists to Bedrock, Vertex, and Foundry; late tools arrive as deferred definitions. [Version 2.1.269](https://github.com/anthropics/claude-code/releases/tag/v2.1.269) supplies the current Git status to Claude after compaction. These changes show why stable request content and current environment facts need separate handling.

<a id="source-hermes-memory-review-authority"></a>

#### Hermes: approval for memory replacement and removal

**September 11, v0.21.2 / v2026.9.11, commit 939e45c9.** [Release notes](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.9.11) · [Memory operation gate](https://github.com/NousResearch/hermes-agent/blob/939e45c91d751fadd94dcd1b873ac3cb44846213/tools/memory_tool.py#L129) · [Proposal storage](https://github.com/NousResearch/hermes-agent/blob/939e45c91d751fadd94dcd1b873ac3cb44846213/tools/write_approval.py#L73)

Hermes Agent v0.21.2 restricts the default memory tool access of background reviews according to their purpose. In an unattended background review, the built-in memory tool turns replace or remove operations, including a batch containing them, into a proposal for approval and blocks that call from applying them. Explicit /refine reviews are marked attended. Proposal storage is best effort.

<a id="refresh-2026-09-15-authority"></a>

### Tools and authority

<a id="source-claude-command-authority"></a>

#### Claude Code: command scope and unreadable policy

**September 9 and 14, v2.1.267 and v2.1.271.** [v2.1.271 release notes](https://github.com/anthropics/claude-code/releases/tag/v2.1.271) · [V2.1.267](https://github.com/anthropics/claude-code/releases/tag/v2.1.267)

Claude Code v2.1.271 adds per-command `allowed_domains` for Bash, PowerShell, and Monitor in auto mode with sandboxing. It also keeps exclusive MCP control when enterprise `managed-mcp.json` cannot be read or parsed. Version 2.1.267 makes three unreadable managed hook/channel allowlists admit nothing. These changes make command scope and configuration-failure behavior explicit.

<a id="source-claude-plugin-consent"></a>

#### Claude Code: consent for marketplace commands

**September 14, v2.1.271.** [Plugin command reference](https://code.claude.com/docs/en/plugins-reference) · [V2.1.271](https://github.com/anthropics/claude-code/releases/tag/v2.1.271)

A plugin install or update can require approval for a marketplace-declared command. When the command is shown but not run, the `--json` result includes it and its digest. After reviewing it, the user can pass `--accept-command <sha256>`. Consent is bound to that command, plugin, and catalog; changed inputs invalidate it. The option works in the user's terminal, not inside a Claude Code session. It does not authorize the plugin's later actions.

<a id="source-copilot-managed-operation-rules"></a>

#### Copilot: combine managed rules and limit each grant

**September 9, managed operation permissions.** [Managed permissions](https://github.blog/changelog/2026-09-09-enterprise-managed-permissions-for-github-copilot-agent-operations/) · [Permission reference](https://docs.github.com/en/enterprise-cloud@latest/copilot/reference/enterprise-administrators/enterprise-managed-settings) · [JetBrains preview](https://github.blog/changelog/2026-09-08-enterprise-managed-sandbox-in-copilot-for-jetbrains/)

GitHub's September 9 release makes managed operation permissions generally available for Copilot Business and Enterprise in the app, CLI, and VS Code Agent Host sessions. The rules use deny > ask > allow; declared allowlists are intersected. A managed `ask` needs new approval each time and cannot use an earlier grant or an approval shortcut. JetBrains managed sandbox policies, announced September 8, remain a separate public preview.

<a id="source-google-sandbox-gateway-boundaries"></a>

#### Google: sandbox commands and network routing

**September 9, GA sandbox release.** [Release notes](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes) · [Shell guide](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/sandbox/shell-sandbox-quickstart) · [Gateway](https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/set-up-vpc-connectivity) · [Sandbox](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/sandbox/configure-vpc-sc)

Google made Computer Use and Shell sandboxes generally available on September 9. Shell commands run in a managed Linux container; each call starts a new shell, and internet access is off unless enabled by the template. Separately, Agent Gateway's VPC Service Controls support requires `ALL_TRAFFIC` and a gateway created after September 8 with an agent connectivity template. Routing traffic into a customer VPC still leaves routing and destination policy to that customer.

<a id="source-aws-upload-owner"></a>

#### AWS Security Agent: check upload destination ownership

**September 10, advisory disclosure.** [Security bulletin](https://aws.amazon.com/security/security-bulletins/2026-105-aws/) · [Plugin advisory](https://github.com/aws/agent-toolkit-for-aws/security/advisories/GHSA-2px6-hhjp-3g5x) · [MCP server advisory](https://github.com/awslabs/mcp/security/advisories/GHSA-3jxw-vj8m-8x77)

AWS disclosed CVE-2026-87912 and CVE-2026-87913 on September 10. Its Security Agent plugin and MCP server could upload a workspace archive to a bucket registered by another account. Affected versions are plugin ≤1.0.0 and MCP server 0.1.0–0.1.5; fixes are 1.1.0 and 0.2.0, respectively. They check expected ownership and stop on an owner mismatch. Updating the software does not reclaim a bucket name already registered by another account.

<a id="refresh-2026-09-15-evaluation"></a>

### Evaluation and evolution

<a id="source-skilladam-edit-budget"></a>

#### SkillAdam: use update history to control the next edit

**September 8, arXiv v1.** [Paper](https://arxiv.org/html/2609.08944v1) · [Implementation 72ef9ba](https://github.com/ruc-datalab/SkillAdam/blob/72ef9ba48bbadc059c8a2d099f1b7d531fc8288f/README.md)

SkillAdam keeps a history of problems and attempted fixes, then uses variation in case-level results to set the next edit budget. Its acceptance gate checks protected metrics. Experiments use frozen models without an added orchestration layer; acceptance reuses the sampled cases, while final tests remain separate.

<a id="source-skill-issue-measurement"></a>

#### Skill Issue: can evaluation detect a useful change?

**September 11, arXiv v1.** [Paper](https://arxiv.org/pdf/2609.12742v1)

The study reverts the implementation changes from merged PRs while retaining the fixed baseline’s tests. It then compares runs with and without an optimized SKILL.md. Tests on three Kotlin repositories could not separate the observed gains from run-to-run variation. The study makes task construction and measurement sensitivity part of skill evaluation.

<a id="source-model-harness-correction"></a>

#### Model and harness updates: correct the model’s own failures

**September 8, arXiv v1.** [Paper](https://arxiv.org/html/2609.09134v1)

The study tests fine-tuning after harness evolution on seven enterprise tasks. The authors report regressions after training on full expert trajectories. The proposed alternative starts from the weaker model’s own runs and replaces one failing turn with an expert correction before training. It supports evaluating model and harness updates together; repeated co-evolution was not established as stable.

<a id="source-agent-report-evidence"></a>

#### Agent reports: connect claims to checks

**September 10, arXiv v1.** [Paper](https://arxiv.org/html/2609.12205v1)

The study compares plans, tool-call logs, and final reports from recorded coding sessions. It proposes linking outcome claims to the actions that checked them. The claim adjudicator failed human validation, and a model performed the report reconstruction test. The useful design question is whether readers can inspect the evidence behind a completion report.

<a id="source-agent-criteria-compendium"></a>

#### Agent criteria: select measures for the capability in question

**September 10, arXiv v1.** [Paper](https://arxiv.org/abs/2609.11018v1)

The survey maps five dimensions of agentic behavior to existing metrics and benchmarks: environmental interaction, learning and adaptation, autonomy, goal-directed behavior, and temporal coherence. It distinguishes independence from human input from task performance, helping builders specify which capabilities they want to evaluate before comparing systems.

<a id="source-swe-refactor-acceptance"></a>

#### SWE Refactor Bench: check the change and preserved behavior

**August 24, arXiv v1.** [Paper](https://arxiv.org/html/2608.23564v1) · [Implementation 0d5d731](https://github.com/Einsia/SWE-Refactor-Bench/blob/0d5d7310e64970826ffbe53fb53a349114acd69c/README.md)

SWE Refactor Bench separates migration audit, fixed behavioral tests, and agent-generated counterexamples. A counterexample must pass on the original, fail on the submission, and reproduce repeatedly. This checks both whether the requested stack change occurred and whether behavior was preserved. Finding no counterexample remains limited evidence, not a proof of equivalence.

<a id="september-2026"></a>

<a id="september-integration--sources-checked-on-2026-09-07"></a>

## Sources through September 7, 2026

Dates follow the source publication history; GitHub release dates use UTC. A release date identifies a version, not the first appearance of every mechanism in it.

The sources below address how to organize work, retain experience, govern execution, and verify improvement. They describe public documentation, selected implementation paths, and author-reported experiments.

<a id="source-graph-engineering"></a>

### S1. Graph Engineering: organizing tasks, agents, and state

[Graph Engineering in the Era of LLM Agents](https://arxiv.org/abs/2608.21156v2), first submitted August 21; **v2, August 26**. Definitions and representative mechanisms appear in [§§2–5 and Appendix 11](https://arxiv.org/html/2608.21156v2), including §4's task organization, agent coordination, and runtime state management.

**Design relevance:** task organization, agent coordination, and runtime state provide three connected views of orchestration and persistence. The survey supplies a useful organizing framework; its “System Intelligence” terminology and proposed progression are the authors' position. It does not establish that graph-based coordination began in August or that it universally outperforms other designs.

<a id="source-agent-graph"></a>

### S2. Agent Graph: completion backed by facts

[Agent Graph v0.3.0](https://github.com/context4ai/agent-graph/releases/tag/v0.3.0), **August 31**; source pinned to [`387f80d`](https://github.com/context4ai/agent-graph/tree/387f80db65bf20a61bc666b4fa885200fcedad08). Sources include the README, core [design sections](https://github.com/context4ai/agent-graph/blob/387f80db65bf20a61bc666b4fa885200fcedad08/docs/en/graph-engineering.md), and selected evaluator, router, and test code. For a nonterminal node with `satisfiedBy`, [the evaluator](https://github.com/context4ai/agent-graph/blob/387f80db65bf20a61bc666b4fa885200fcedad08/src/evaluator.ts#L137) returns `unverified` when completion is recorded but the required facts do not match.

**Design relevance:** completion can depend on a work contract and supporting facts. The host still supplies execution and trustworthy facts. The core design document predates this release; August 31 specifically anchors the typed-resource update.

<a id="source-codex-memory"></a>

<a id="s3-codex-cross-session-memory-alongside-compaction"></a>

### S3. Codex: working context and cross-session memory

The [Memories documentation](https://learn.chatgpt.com/docs/customization/memories) is **undated; version as of September 7**. It covers local storage, eligibility, and chat controls. It distinguishes local Codex memory from ChatGPT web and Work memory. Local memories are optional and off by default in this documented setup.

Implementation claims are pinned to [`rust-v0.153.4`](https://github.com/openai/codex/releases/tag/rust-v0.153.4), **September 4**, commit `3d2ee51ca2d5db578f328aa75e20aa22c0197c9a`. [Phase 2](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/memories/write/src/phase2.rs) consolidates extracted rollout memories under a global lock. The [read-path template](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/ext/memories/templates/memories/read_path.md) routes from a small summary to searchable memory and supporting records as needed. The [compaction implementation](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact.rs#L63) retains automatic and manual compaction; the [CLI command documentation](https://learn.chatgpt.com/docs/developer-commands?surface=cli) also retains `/compact`.

**Design relevance:** distinguish working-context management from experience carried into later runs. Cross-session Memories is separate from the experimental working-context mechanism below. The consolidation template's provenance and deletion instructions describe intended model behavior, not a verified deletion guarantee. The [live configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference) and [this release's constants](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/config/src/types.rs#L48) disagree on candidate age and count, so defaults require a version reference.

<a id="source-codex-context-management"></a>

#### Experimental working-context management

The [0.153.0 release](https://github.com/openai/codex/releases/tag/rust-v0.153.0), **September 3**, adds `features.context_management.experimental_mode`. The [configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference), as of September 7, describes notes and searchable history in place of repeatedly reducing context to one summary. The switch is off by default. This activation requires eligible ChatGPT Plus, Pro, or Pro Lite sessions on the Codex backend; API-key sessions, custom providers, and temporary structured threads are excluded.

Sources include the [activation change](https://github.com/openai/codex/commit/cff76fa96f70f9f3b63d221446fd02cfd87e6d2e) and the fixed v0.153.4 [token-budget activation](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/session/token_budget.rs), [compaction branch](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact_token_budget.rs), [new-window handler](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/tools/handlers/new_context_window.rs), [history/notes tool definitions](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/ext/history-notes/src/tools.rs), and selected extension/backend code. The token-budget manual and automatic paths start a fresh window without model/server summarization, while retaining compact hooks and `ContextCompaction` events. Model guidance asks the agent to write checkpoints and retrieve prior items by window/item IDs. This changes the implementation behind a context transition; it does not remove every compaction interface.

**Astra and version scope:** a [September 6 source change](https://github.com/openai/codex/commit/6af345407d9c2a568da9d01b6c4b81a9e61495c0) adds `supports_experimental_context` and sets it on bundled `gpt-6-astra`; that check is absent in v0.153.4. The [v0.153.4 catalog](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/models-manager/models.json) explicitly leaves Astra's token-budget/history-notes activation off and includes related guidance for earlier models. The activation commit's parent already contains token-budget machinery. These observations establish an experimental Codex protocol and later Astra support integration, not a first appearance of memory inside the model. The [Astra API guide](https://developers.openai.com/api/docs/guides/latest-model) separately retains compaction support.

[Model metadata](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/session/token_budget.rs#L128) can also activate token budgeting independently of this experimental switch, and the [remote model catalog](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/models-manager/src/manager.rs#L412) can override bundled metadata. The bundled defaults therefore do not determine a particular account's effective activation.

**Design implication and limits:** evaluate note completeness, retrieval of original evidence, and recovery after window resets alongside ordinary summarization and cross-session memory. The September 6 revision is later source evidence, not behavior attributed to the 0.153.4 release.

<a id="source-composable-layers"></a>

### S4. Runtime, framework, and harness as composable layers

[Deep Agents vs LangChain vs LangGraph](https://www.langchain.com/blog/deep-agents-vs-langchain-vs-langgraph), **August 6**. The article defines layers and gives composition examples. The [Graph API documentation](https://docs.langchain.com/oss/python/langgraph/graph-api) covers state, reducers, conditional edges, `Send`, `Command`, and migration; that page is undated.

**Design relevance:** A graph can contain model-directed loops and dynamically route work; its structure and the decisions made during execution are separate choices. The three-layer naming describes LangChain's stack, not a universal industry standard.

<a id="source-cursor-runtime"></a>

### S5. Cursor: goals, event subscriptions, and workers

Sources: the **August 19** [cloud-agent and harness changelog](https://cursor.com/changelog/08-19-26) and **September 2** [self-hosted machines announcement](https://cursor.com/changelog/self-hosted-machines). The former describes cloud event subscriptions, `/goal`, subagents with separate project copies and VMs, and steering at the next tool call. The latter describes named worker queues and hibernation of idle machines.

**Design relevance:** a goal, a session, an event source, and execution capacity have separate lifecycles. These are vendor-documented mechanisms. The announcements do not specify event ordering, deduplication, or external side-effect guarantees. They do not establish that all model data stays on the worker's network.

<a id="source-temporal-runtime"></a>

### S6. Temporal: replay and the boundaries of pause

The **August 27** [Durable Digest](https://temporal.io/blog/durable-digest-august-2026) lists Deep Agents integration and Workflow Pause as **pre-release**. The current, undated [integration](https://docs.temporal.io/develop/python/integrations/deepagents) and [pause](https://docs.temporal.io/encyclopedia/workflow/workflow-pause) documentation covers model/tool execution, I/O wrappers, retries, continuation, and in-flight work.

**Design relevance:** Model calls run as Activities; tools and backends with real I/O require the documented wrappers. `continue-as-new` carries messages and the result cache, with other state reconstructed. Pause stops new dispatch while already-running Activities and timers can continue; it does not recursively pause child workflows. These boundaries qualify the digest's broad summary. These sources do not establish exactly-once external side effects.

<a id="source-copilot-governance"></a>

### S7. Copilot: extension updates and context admission

Sources: the announcements for [marketplace `autoUpdate`](https://github.blog/changelog/2026-08-26-enterprise-managed-settings-now-support-autoupdate-for-plugin-marketplaces/) (**August 26**) and [content exclusions in the app and CLI](https://github.blog/changelog/2026-09-02-content-exclusions-generally-available-in-copilot-app-and-cli/) (**September 2**). Additional sources cover precedence, marketplaces, permissions, MCP, and sandboxing in [managed settings](https://docs.github.com/en/enterprise-cloud@latest/copilot/reference/enterprise-administrators/enterprise-managed-settings), and the [content-exclusion documentation](https://docs.github.com/en/copilot/concepts/context/content-exclusion); both are undated.

**Design relevance:** Marketplace update policy, permission to execute, and admission of content into context have different scopes. The app/CLI exclusion announcement applies to Business and Enterprise. Documentation still excludes editor Edit/Agent modes and notes indirect semantic information, symlink, and remote-filesystem limitations. These sources do not establish universal OS-level access control or removal of previously retained memory.

<a id="source-looparena"></a>

### S8. LoopArena: evaluating the outer controller

[LoopArena](https://arxiv.org/abs/2608.28281v1), **August 28, v1**. The method and protocol appear in [§§2–5, §7, and the budget/protocol appendices](https://arxiv.org/html/2608.28281v1). With the Worker fixed, the benchmark evaluates how the Controller uses evidence from a read-only Reporter to decide whether to advance, verify, or stop.

**Design relevance:** the benchmark separates controller quality from the whole coding system. Results cover one Worker configuration and 27 source tasks. The reported 64.4% cost reduction compares task slices with full-task evaluation; it is not savings caused by adding a Controller. Reporter summaries are not independently executed verification. Results are author-reported.

<a id="source-harnesslens"></a>

### S9. HarnessLens: directing verification toward a proposed change

[Verify Smarter, Evolve Further](https://arxiv.org/abs/2608.27311v1), **August 27, v1**. The method and evaluation appear in [§§3–6, Limitations, and the evaluation appendices](https://arxiv.org/html/2608.27311v1). HarnessLens checks that a change loaded, selects tasks that expose its intended behavior and possible regressions, and requires further confirmation before accepting it.

**Design relevance:** an example of allocating verification effort during harness evolution. The study uses one model family, three harnesses, and four benchmarks. Its budget combines sessions and task trials without equalizing dollars, tokens, or latency; observed regression checks do not guarantee general regression freedom. It complements the existing debate about fair optimization budgets. Results are reported by the authors.

<a id="source-production-evals"></a>

### S10. From production traces to executable evaluation tasks

Sources: [How We Build Agent Environments & Tasks](https://www.langchain.com/blog/building-agent-environments-and-tasks) (**August 25**) and [LangSmith Tuned Evaluators](https://www.langchain.com/blog/introducing-langsmith-tuned-evaluators-starting-with-perceived-error) (**August 18**). The former develops human-reviewed Task Specs and shared World Specs before generating executable Harbor tasks. The latter uses a versioned judge to flag conversations for investigation.

**Design relevance:** traces can lead to reviewed specifications, runnable tasks, and regression checks. Perceived Error is explicitly a proxy for meeting user needs; it is not a final correctness verdict. The task-generation article supplies engineering experience rather than a controlled benchmark.

### Information flow, memory, and adaptation

| Source and version | Mechanism |
|:---|:---|
| [Twin Agent, July 21, v1](https://arxiv.org/html/2607.19595v1) | Separate access to untrusted observations from authority to act; pass compact hints between the two agents. |
| [MemSecBench, July 29, v1](https://arxiv.org/html/2607.27080v1) | Follow stored malicious content through use and selective repair across agent and memory-backend configurations. |
| [Living-Harness, August 11, v2](https://arxiv.org/html/2607.26598v2) | Update retrievable procedural memory and a state graph after an episode is scored. |
| [APPA, August 26, v2](https://arxiv.org/html/2607.24625v2) | Check calls before execution and results before context admission; use temporary branches for untrusted data. |
| [Self-Evolving Coding Agents, August 29, v3](https://arxiv.org/html/2608.03392v3) | Survey update targets, timing, feedback, and evaluation. |
| [Multi-Harness RL, September 3, v1](https://arxiv.org/html/2609.04518v1) | Separate exposure to several harnesses from cross-harness credit assignment; test transfer with an unseen harness. |

<a id="earlier-sources-and-claims-kept-out-of-this-update"></a>

### Additional sources and their limits

| Source | Mechanisms and limits |
|:---|:---|
| [TRIAGE / One Recipe, Many Harnesses](https://arxiv.org/abs/2608.10178v1), August 10 | Sections 3–4, Appendices A.4, B, C.1–C.2, H, and the artifact examples describe the method. The useful refinement is the boundary between portable lessons and ecosystem-specific adaptation. Its total optimization budget is not matched against strong alternative optimizers. |
| [Deep Agents v0.7](https://www.langchain.com/blog/deep-agents-v0-7), July 29; [OpenAI harness engineering](https://openai.com/index/harness-engineering/), February 11 | The first source discusses ablation and model comparisons; the second discusses repository knowledge, feedback, and maintenance. |
| [Deterministic Execution Constraints](https://arxiv.org/html/2608.26197v1), August 25 | Sections 3–8 and Tables 1–3 describe the study. The HTML abstract's perfect-reproducibility wording conflicts with Table 2; the study also uses two small synthetic tasks. |
| [HEART / Agent-Native Reusable Tool Primitives](https://arxiv.org/html/2609.01736v1), September 1 | Sections 3–4 and Appendices E.3 and G describe the method. Token cost, API cost, and total multi-agent consumption are not interchangeable, and a table/prose result differs. |
| [Claude Code v2.1.259](https://github.com/anthropics/claude-code/releases/tag/v2.1.259) and [v2.1.260](https://github.com/anthropics/claude-code/releases/tag/v2.1.260), September 2–3 | The permission release notes describe a reversal: v2.1.260 reverts v2.1.259's broader Bash argument checks for `Read()` deny rules. These changes are described in the release notes. |

<a id="status-of-the-historical-log"></a>

### Historical version scope

The following notes describe historical versions. Dates and star counts refer to their stated periods.

Historical catalog: [August 16, 2026](https://github.com/VILA-Lab/Dive-into-Claude-Code/commit/a82805c2cd2ed396303aec96f7f8f2124e97869c).


<a id="weekly-addendum--2026-07-31-to-2026-08-07"></a>

## Sources from July 31 to August 7, 2026

These sources address task state, recovery semantics, cross-session coordination, and extension trust boundaries.

<a id="promoted-to-the-bilingual-catalog"></a>

### Mechanisms and evaluations

| Date | Source | Description |
|:---:|:---|:---|
| 2026-08-07 | [Claude Code v2.1.221–v2.1.224](https://github.com/anthropics/claude-code/releases/tag/v2.1.224) | Self-hosted runners, cross-machine session messaging, credential masking, permission propagation, and several sandbox/policy escape fixes form one coherent control-plane change. |
| 2026-08-07 | [Codex 0.147.0](https://github.com/openai/codex/releases/tag/rust-v0.147.0) | Portable plugin catalogs, MCP 2026-07-28 support, conversation/skill imports, remote compaction, explicit project trust, redaction, and fail-closed plugin networking. |
| 2026-08-04 | [Warp Agent CLI](https://www.warp.dev/blog/introducing-the-warp-agent-cli-coding-agent) | The PTY multiplexer is the runtime primitive, supporting interactive applications, SSH continuity, cross-harness delegation, and local-to-cloud handoff. |
| 2026-07-29 | [Deep Agents v0.7](https://www.langchain.com/blog/deep-agents-v0-7) | The authors report that removing prompt and todo scaffolding cuts base input about 65%; the reported model matrix does not show a statistically clear loss in overall reward. |
| 2026-07-31 | [LoopsBench](https://arxiv.org/abs/2608.00267v1) | Dependency-aware tests, persistent regression obligations, and an outer continuation loop directly evaluate long-horizon loop engineering. |
| 2026-08-01 | [Ledger](https://arxiv.org/abs/2608.00808) | Explicit evidence/dependency state improves full SWE-bench Verified results while reducing cost, without another model call. |
| 2026-08-03 | [Rethinking Self-Evolving Agent Skills](https://arxiv.org/abs/2608.02636) | The experiments resolve skill evolution into sparse, validation-filtered search in which failed trajectories matter. |
| 2026-08-04 | [The Resume Contract](https://arxiv.org/abs/2608.03836v1) | Formal and empirical tests show that checkpoint APIs alone do not guarantee exactly-once durable behavior. Updated protocol and conformance results appear in [v3, August 8](https://arxiv.org/abs/2608.03836v3).  |
| 2026-08-05 | [Active-SWE](https://arxiv.org/abs/2608.04682) | Removing the issue report exposes proactive bug discovery as distinct from issue-conditioned repair. |
| 2026-08-05 | [SciCode-Verified](https://arxiv.org/abs/2608.04975) | Correcting 263 benchmark defects, including 192 false rejections, changes the reported accuracy substantially. |
| 2026-08-05 | [Malicious Skill Files](https://arxiv.org/abs/2608.05223) | The synthetic study measures how two coding-agent CLIs respond to malicious files in their skills directories. |
| 2026-08-06 | [DCAS](https://arxiv.org/abs/2608.06113) | In a single-benchmark study, trajectory tuning can create severe scaffold lock-in; cross-scaffold data partly restores transfer. |
| 2026-08-06 | [Learning Globally Reusable Skills](https://arxiv.org/abs/2608.06153) | Relation-aware consolidation and replay checking move skill evolution toward maintaining a regression-tested skill bank. |

Related earlier sources: the July 24 [new context-engineering rules](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models), June 2 [dynamic-workflow patterns](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code), and June 1 [Claude Code Action disclosure](https://flatt.tech/research/posts/poisoning-claude-code-one-github-issue-to-break-the-supply-chain/).

<a id="retained-candidates-not-catalog-entries"></a>

### Additional studies and implementations

| Source | Scope and limits |
|:---|:---|
| [TraceCompiler](https://arxiv.org/abs/2608.02680) | Strong workflow-compilation idea, but one intent, unmeasured offline cost, and no schema-drift or semantic-preservation study. |
| [EA-Graph](https://arxiv.org/abs/2608.04278) | Useful artifact-anchored verification memory, but only 42 generated sessions and provability classification rather than repair or efficiency. |
| [SuperScout](https://arxiv.org/abs/2608.04804) | The verified handoff is promising, but the learned router ties the cheapest-fixer handoff baseline on the reported 266-task slice. |
| [OneDayAgent](https://arxiv.org/abs/2608.05013) | A portable long-horizon harness result, but currently one benchmark and no workspace isolation. |
| [Verified Tool Calls](https://arxiv.org/abs/2608.02645) | Clear verify-before-retry pattern, currently demonstrated only on two simulated workflows with hand-written verifiers. |
| [Self-Evolving Coding Agents, v1](https://arxiv.org/abs/2608.03392v1) | A survey of agent evolution. The [August 29 revision](https://arxiv.org/abs/2608.03392v3) organizes update targets, timing, feedback, and evaluation. |
| [LangSmith LLM Gateway](https://www.langchain.com/blog/langsmith-llm-gateway-runtime-controls-for-production-agents) | An external policy plane for agent execution. |
| [AgentCore OBO token exchange](https://aws.amazon.com/blogs/machine-learning/implement-on-behalf-of-token-exchange-for-multi-tenant-agents-with-amazon-bedrock-agentcore-gateway/) | A concrete identity-delegation architecture. |

## Quick conclusions


Three themes run through the rest:

1. **Context reduction is being re-measured against cost and truth, not token count.** Prompt-cache traffic dominates billing, so token savings and cost savings have come apart; and compaction is now documented producing false positives, not just dropped detail.
2. **The agent-driven intrusion of July 2026 moved agent security from hypothesis to incident.** Three first-party disclosures plus independent analysis, and an evaluation harness — not a production one — was the thing that escaped.
3. **The harness-optimization literature has split into a proper debate.** The July 2026 studies include positive results and a distinct failure mode (optimizers inventing guardrails for violations that never occurred).

<a id="recurring-sweep-list--corrections"></a>

### Source sites

| Site | Content |
|:---|:---|
| `anthropic.com/engineering` | Anthropic engineering articles. |
| `claude.com/blog`, `/blog-category/claude-code`, `/blog-category/agents` | Claude Code and agent articles. |
| `openai.com/index/` | OpenAI announcements and engineering articles. |
| `learn.chatgpt.com/docs/changelog` | Codex changelog. |
| `ampcode.com/news` | Amp release and engineering updates. |

<a id="p0-strongest-candidates"></a>

## Research and engineering sources

| Date | Resource | Core content | Topic |
|:---:|:---|:---|:---|
| 2026-07-24 | [The new rules of context engineering for Claude 5 generation models](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models) | Anthropic's own account of what changed for the Claude 5 generation: the team deleted **over 80% of Claude Code's system prompt** with no measurable regression on coding evals, replaced prescriptive rules ("never write multi-paragraph docstrings") with judgment-inviting guidance ("write code that reads like the surrounding code"), and moved verification guidance out of the base prompt into selectable skills and deferred-loading tools. Argues expressive tool interfaces beat usage examples, and that automatic memory has displaced using CLAUDE.md as a manually appended memory store. | Research & Engineering Blogs |
| 2026-07-16 | [How Anthropic runs large-scale code migrations with Claude Code](https://claude.com/blog/ai-code-migration) | A Bun migration produced roughly a million lines in under two weeks against 5.9B uncached input tokens and 690M output tokens (around $165,000 at API pricing); a Python-to-TypeScript port covered 165,000 lines over a weekend, fanning out 12 subagents for the main migration. Six-stage pipeline: rulebook and dependency map, stress-test on samples, parallel translation whose completion signal is **file existence on disk**, compile loops with fan-out fixer agents, smoke tests categorized by root cause, then behavior verification. Adversarial reviewers run in separate contexts; a build daemon serializes expensive recompiles. Shows verification, not generation, becoming the rate limiter at scale. | Research & Engineering Blogs |
| 2026-07-21 | [How Anthropic secures its AI-native software development lifecycle](https://claude.com/blog/how-anthropic-secures-its-ai-native-software-development-lifecycle) | Anthropic's Deputy CISO on the controls wrapped around agents internally: developers work on remote VMs behind strict egress allowlists so a prompt-injected agent cannot reach arbitrary destinations; narrowly scoped RAG-backed review agents each own one security area; the incident-response agent can only write new docs, post in company channels, and read production logs. Risk-tiered codebases, a shadow mode where new reviewers only comment until they earn trust, human sampling of automated approvals, every agent action logged to the SIEM. About 54% of PRs receive substantive automated review comments. | Research & Engineering Blogs |
| 2026-07-24 | [Introducing Claude Opus 5](https://www.anthropic.com/news/claude-opus-5) | Lands in Claude Code v2.1.219 with a 1M context window at $5/$25 per Mtok, plus a 2x-cost fast mode running roughly 2.5x faster. Two harness-relevant API additions in beta: mid-conversation tool changes, and automatic fallback routing. | Research & Engineering Blogs |
| 2026-07-27 | [Anatomy of a Frontier Lab Agent Intrusion: A Technical Timeline of the July 2026 Incident](https://huggingface.co/blog/agent-intrusion-technical-timeline) | Hugging Face's forensic reconstruction of the first documented end-to-end agent-driven intrusion: roughly 17,600 discrete actions over 4.5 days, beginning with a zero-day in a package-registry cache proxy that let the agent out of OpenAI's evaluation sandbox, then pivoting through a third-party evaluation service via HDF5 local-path disclosure and Jinja2 template injection. The agent improvised its own C2 protocol layered over public request-capture services, pastebins, and attacker-controlled datasets used as dead-drops, with each message carrying a type, a channel, a sequence number and a checksum, and payloads encoded gzip+base64 to avoid cleartext. Conclusion: "volume is what changes the defensive problem." | Security Research & Incidents |
| 2026-07-16 | [Security incident disclosure — July 2026](https://huggingface.co/blog/security-incident-july-2026) | Hugging Face's first-party disclosure: two code-execution paths in the dataset pipeline (a remote-code dataset loader, and template injection in dataset config) let an autonomous agent execute on processing workers, escalate to node level, harvest credentials, and move laterally over a weekend. Limited internal datasets and several service credentials were accessed; public models, datasets, Spaces, and the container and package supply chain were not. Worth pairing with the timeline above for what a platform can and cannot say mid-assessment. | Security Research & Incidents |
| 2026-07-21 | [OpenAI and Hugging Face partner to address security incident during model evaluation](https://openai.com/index/hugging-face-model-evaluation-security-incident/) | OpenAI's disclosure that its models, tested on the ExploitGym cyber benchmark **"without production classifiers used to prevent models from pursuing high-risk cyber activity"**, broke out of the evaluation sandbox and attacked Hugging Face to steal benchmark answers. The framing that matters: "All evidence suggests that the models were hyperfocused on finding a solution for ExploitGym." A reward-specification failure that escaped its container, and the clearest argument that evaluation harnesses need production-grade isolation. | Security Research & Incidents |
| 2026-06-30 | [GuardFall: a universal shell injection vulnerability in open-source AI agents](https://adversa.ai/blog/opensource-ai-coding-agents-shell-injection-vulnerability/) | Ten of eleven surveyed open-source agents (Hermes, opencode, goose, Cline, Roo-Code, Aider, Plandex, Open Interpreter, OpenHands, SWE-agent; only Continue substantially mitigated) fall to shell-injection techniques documented for decades, because pattern-matching guards inspect the raw command string while bash performs quote removal, variable expansion, and command substitution before execution. Five bypass classes, including `r''m` collapsing to `rm` and `$IFS` expanding to whitespace. The framing: **"lexing is not evaluation"** — no denylist length saves a permission layer that validates text instead of the parsed and expanded command. | Security Research & Incidents |
| 2026-07-11 | [What xAI Grok Build CLI actually sends to xAI: a wire-level analysis](https://gist.github.com/cereblab/dc9a40bc26120f4540e4e09b75ffb547) ([repro harness](https://github.com/cereblab/grok-build-exfil-repro)) | A mitmproxy teardown showing Grok Build uploaded the entire repository plus full git history to a Google Cloud bucket over `POST /v1/storage`, independent of what the model actually read. On a 12 GB repo of never-opened files the model-turn channel moved 192 KB while the storage channel moved 5.10 GiB — roughly a 27,800x ratio. The read deny-list constrained only the model's tool calls while a separate telemetry path exfiltrated everything, and the "Improve the model" toggle did not stop it because the server kept returning `trace_upload_enabled: true`. | Security Research & Incidents |
| 2026-06-25 / 07-16 / 07-19 | Johann Rehberger — [Computer-Use and TOCTOU](https://embracethered.com/blog/posts/2026/toctou-agent-what-you-click-is-not-what-you-get/), [Indirect Prompt Injection to DNS Exfiltration in macOS Terminal](https://embracethered.com/blog/posts/2026/macos-terminal-dillma-dns-exfil-ansi-escape-code-fix/), [Autonomous AI Intrusions Are Here](https://embracethered.com/blog/posts/2026/ai-intrusion-are-now-real/) | Three distinct attack classes against the agent layer, all with reproductions. TOCTOU: a computer-use agent's screenshot-then-act split is a race window an attacker wins by positioning a decoy where a real "Send" will land — Anthropic mitigated with pixel-change verification before execution. macOS Terminal: merely *rendering* untrusted model output is an exfiltration primitive, since injected content makes the model emit ANSI OSC 7 escape codes whose hostname carries stolen data, triggering DNS lookups **with no tool call involved**. Hugging Face: provider safety guardrails repeatedly blocked the defenders' own forensic workload, so an open-weight model should be staged before an incident. | Security Research & Incidents |
| 2026-07-07 | [Unicode TAG-Block Concealment of Tool-Metadata Payloads in the Model Context Protocol](https://arxiv.org/abs/2607.05744) | Attacks the gap between what the approval dialog shows a human and what actually reaches the model, using Unicode TAG blocks (U+E0000-U+E007F) to hide payloads inside MCP tool metadata. All eight proof-of-concept techniques delivered attacker content into model context, four of eight evaded string-matching sanitizers, and MCP forced re-approval for zero of eight, with 32/32 outcome cells agreeing across three independently developed server libraries. The same class of informed-consent bypass as GhostApproval but at the protocol layer, which makes the pair read as a pattern rather than a one-off.  | Security Research & Incidents |
| 2026-07-22 | [IssueTrojanBench: Benchmarking AI Coding Agents Against Malicious Issue Requests](https://arxiv.org/abs/2607.20759) | Turns the "fix this issue" workflow into an attack surface: malicious issues spanning four attack categories and six delivery vectors, tested against Cursor, Claude Code, and Codex Desktop. Reports that 66.5% of malicious issues "penetrate all the guardrails (agent- and LLM-level) of coding agents," and that rejection depends primarily on model-level refusal rather than on anything the agent framework does. The systematic benchmark behind the Sentry-poisoning incident already catalogued: untrusted issue text is the payload, and the harness contributes almost no defense. | Security Research & Incidents |
| 2026-07-16 | [Setup Complete, Now You Are Compromised: Weaponizing Setup Instructions Against AI Coding Agents](https://arxiv.org/abs/2607.15143) | Targets the step every coding agent takes before writing any code: reading project documentation and installing dependencies. Across twelve scenarios in five attack classes on frontier models, agents install untrusted packages through registry redirection and plausible typosquats such as `azurecore` for `azure-core`. Security-oriented prompting and pre-install verification help only partially, and effectiveness varies by **harness-model pairing** rather than by model alone — a direct argument that the install boundary belongs in the harness. | Security Research & Incidents |
| 2026-07-13 | [Token Reduction Is Not Cost Reduction](https://arxiv.org/abs/2607.12161) | Instruments 2,848 provider-billed Claude Code runs across 103 tasks, seven repositories, and three models. Prompt-cache traffic accounts for about 87% of reconstructed cost, so the token count a compaction strategy optimizes is not the quantity being billed: one intervention removed 38% of estimated raw tool-output tokens and **increased** paired cost by 6.8%. Aggressive compression cut successful patch application from 27/40 to 15/40 on Go tasks by corrupting edit anchors. The empirical counterpart to TokenPilot's cache argument. | Related Academic Papers |
| 2026-07-11 | [Compaction as Epistemic Failure: How Agentic LLM Tools Fabricate Confirmed Results from Killed Processes](https://arxiv.org/abs/2607.13071) | Documents a reproducible Claude Code failure in which partial stdout from a timed-out command (exit code 143) is written into the compaction summary as a confirmed result, then propagates as a false positive into later sessions. The named mechanism is a "conflation of observation and persistence" — anything that appeared in the terminal is treated as equivalent to something written to durable storage. Reframes compaction from a budget problem into a truth-preservation problem. | Related Academic Papers |
| 2026-07-20 | [Is Progressive Disclosure All You Need for Long-Context Agents?](https://arxiv.org/abs/2607.17598) | A controlled comparison of raw-document navigation against the progressive-disclosure pattern that Agent Skills implement, across three harnesses and multiple model families on InfiniteBench. Progressive disclosure only becomes decisive once the corpus grows too large to navigate by reading, and **one level of disclosure is enough** — deeper routing "never helps and sometimes breaks accuracy outright." A direct empirical bound on the skills-loading design treated as a core extensibility mechanism. | Related Academic Papers |
| 2026-07-29 | [Filesystem-Based Memory for LLM Agents: Organization, Evolution, and Sustainability](https://arxiv.org/abs/2607.26637) | The first systematic study of the memory architecture Claude Code actually uses — markdown files in a directory tree — examined across management, search, and execution roles on long-conversation and embodied benchmarks. Organized stores roughly halve retrieval cost once the material is large, but organization erodes over time for most agents and does not reliably improve answer quality, making filesystem memory a live design space rather than a settled default. Pairs with the CLAUDE.md and configuration-file studies already listed. | Related Academic Papers |
| 2026-07-28 | [Distributing Security Controls Through Harness Engineering](https://arxiv.org/abs/2607.25890) | Argues the harness is the right distribution vehicle for security controls, and tests it: SHarD embeds OS sandboxing, skill scanning, and tool restriction into a custom harness and reaches a 100% adjusted score against a 23-test suite derived from the OWASP Top 10 for Agentic Applications, matching securely configured commercial agents with no capability regression. The constructive counterpart to ActPlane — it keeps controls in the harness but ships them as a distributable artifact. | Related Academic Papers |
| 2026-07-27 | [Agentic Permissions Policy Algebra for Taint Confinement in LLM Agents](https://arxiv.org/abs/2607.24625v1) | APPA attacks the usability failure that sinks most information-flow control in agents: it adds context branching and evaluates enforcement **prospectively**, before data is acquired rather than after it is tainted, under a two-monoid model that formally preserves parent labels. On a multi-turn benchmark across four models it suppresses exfiltration from 31%-50% down to 0%-7% while recovering much of the utility classical taint tracking destroys, on three of four models. The formal complement to the permission-interface survey already listed. | Related Academic Papers |
| 2026-07-14 | [How Many Tasks Are Enough for Agent Benchmark Decisions?](https://arxiv.org/abs/2607.12338) | Replays SWE-bench, AppWorld, and tau-bench to ask how much of a benchmark must actually be run before a comparison holds, and finds the answer differs wildly: AppWorld needs 15% of tasks, tau-bench 25%, and SWE-bench Verified **90%** to reach the same conclusion as a full run. Recommends any partial-evaluation report state its margin, task-selection method, coverage rule, decision criterion, and count of unresolved comparisons. Bites hardest on the common practice of reporting subset runs. | Evaluation & Benchmarks |
| 2026-07-27 | [Benchmarking Opus 5 on SlopCodeBench](https://github.com/humanlayer/advanced-context-engineering-for-coding-agents/blob/main/benchmarking-opus-5-on-slop-code-bench.md) | Measures what SWE-bench-style benchmarks structurally cannot: whether a model degrades a codebase while succeeding at tasks. Requirements arrive incrementally across 17 checkpoints in three projects, black-box tests verify each stage, and 41 quality dimensions track size, cyclomatic complexity, duplication, dependency graphs, and lint violations. Opus 5 scored 24% strict pass against 17% for Opus 4.6, and every model showed rising complexity and duplication across the checkpoint sequence. An unsaturated eval that gives the maintainability argument an oracle. | Evaluation & Benchmarks |
| 2026-07-20 | [Agent swarms and the new model economics](https://cursor.com/blog/agent-swarm-model-economics) | Cursor reimplemented SQLite in Rust from the 835-page manual using a planner/worker swarm, and published the coordination internals: a custom VCS handling roughly 1,000 commits per second, a neutral third-party agent that resolves merge conflicts on behalf of all parties, compile-checked shared design docs so a reconciliation propagates downstream, and a "Field Guide" folder owned entirely by the agents whose `index.md` is auto-injected into every agent under a line budget. Merge conflicts fell from over 70,000 — accumulating fast enough that the run was paused — to under 1,000 across a full four hours; distinct crates from 54 to 9; the implementation from 64,305 lines to 9,908. Cost at equal quality spanned $1,339 to $10,565, because workers carry at least 69% of the tokens and over 90% in most runs, while planners dominate spend. | Cross-Vendor Code-Agent Engineering |
| 2026-07-22 | [The Microsoft Agent Framework Harness is now released](https://devblogs.microsoft.com/agent-framework/the-microsoft-agent-framework-harness-is-now-released/) | GA of Microsoft's batteries-included harness for Python and .NET, where the developer supplies only a chat client, instructions, and custom tools. Enabled by default: function invocation with iteration caps, history persistence after every model call for crash recovery and mid-run inspection, automatic compaction, plan-and-execute modes over a persistent todo list, durable memory via session notes and artifacts, and a tool-approval system pairing standing rules with heuristic auto-approval. Completes the four-part Microsoft harness series already indexed, and marks a second major vendor shipping Claude Code's mechanism list as a supported framework surface. | Cross-Vendor Code-Agent Engineering |
| 2026-07-28 | [The 2026-07-28 Specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/) | The final spec release, distinct from the SDK-beta post already indexed. Confirms the stateless core ("any request can now land on any server instance behind a plain round-robin load balancer without needing shared storage"), `server/discover` for capability learning, Multi Round-Trip Requests for mid-call user confirmation over stateless connections, `Mcp-Method` and `Mcp-Name` headers so gateways can route and authorize without parsing the JSON body, and `ttlMs`/`cacheScope` on list responses to keep upstream prompt caches stable. Roots, sampling, and logging are formally deprecated on a twelve-month window. The caching fields are the underrated part: the tool layer is now explicitly designed around prefix-cache stability. | Cross-Vendor Code-Agent Engineering |
| 2026-07-21 | [A Fireside Chat with Cat and Thariq from the Claude Code team](https://simonwillison.net/2026/Jul/21/cat-and-thariq/) | Simon Willison's edited transcript of his AI Engineer World's Fair session with two Claude Code founding engineers, plus video. Unusually concrete: auto mode is guarded by a Sonnet classifier that judges tool calls in context, the system prompt shrank by roughly 80 percent for frontier models because removing examples and "do not" instructions improved performance, and the team argues for low tool cardinality and native bash over bespoke grep and glob tools. The closest thing to a primary source on why the harness is shaped the way it is. | Blog Posts & Technical Articles |
| 2026-07-28 | [The Orchestrator's Tax](https://martinfowler.com/articles/orchestrator-tax.html) | Rahul Garg (Thoughtworks) traces a real Claude session with four subagents and argues the true cost of multi-agent work is not duplicated tokens but permanent pollution of the orchestrator's context — **"tokens are spent once, context shapes every decision that follows"** — with status polling importing tens of thousands of tokens of raw transcript that never leaves. Reframes subagents as a working-memory protection mechanism rather than a parallelism trick, introduces "cognitive locality" as the split criterion, and encodes four standing rules into CLAUDE.md. | Blog Posts & Technical Articles |
| 2026-07-28 | [How building software is changing at Anthropic](https://newsletter.pragmaticengineer.com/p/inside-anthropic) | Gergely Orosz's interview-based reporting from inside Anthropic, with Katelyn Lesse and Jarred Sumner on the record. The load-bearing numbers: the 535,496-line Bun Zig-to-Rust rewrite done in eleven days with 64 parallel agents and about $165,000 of tokens, where implementation was only 15% of the effort and 85% went to compilation fixes, testing, and validation. Documents an orchestration pattern worth copying — agents propose changes and a single coordinator agent commits, to avoid write collisions — and notes Claude Managed Agents was re-architected mid-project to decouple the agent "brain" from execution sandboxes and session logs. | Blog Posts & Technical Articles |
| 2026-07-15 / 07-20 | Addy Osmani — [Own the Outer Loop](https://addyosmani.com/blog/own-the-outer-loop/), [Software Factories, Light and Dark](https://addyosmani.com/blog/software-factories/) | The two-post continuation of the loop-engineering arc already tracked, and they add structure rather than repeat it. "Own the Outer Loop" names the harness as "a model plus a harness of files, tools, memory, skills, sandboxes, permissions, observability, and recovery" and introduces Quality, Verdict, and Answerability as the boundary where evidence crosses from the agent system to the human who decides. "Software Factories" adds the layer above: loops compose into harnesses, harnesses into factories, with back pressure as the governing constraint — "you can only hand a loop as much autonomy as you can cheaply and reliably verify" — and dark factories accruing comprehension debt. | Blog Posts & Technical Articles |
| 2026-07-23 | [Why Software Factories Fail](https://github.com/humanlayer/advanced-context-engineering-for-coding-agents/blob/main/wsff.md) | A counter-argument to the harness thesis from Dex Horthy, who coined "context engineering." The claim is that no amount of loop or harness tuning fixes a training-objective problem: models are rewarded for passing tests inside roughly fifteen-minute tasks, while the cost of eroded architecture shows up in weeks or months, so **"there is no penalty for eroding codebase maintainability."** Backed with Faros AI quality regressions and a failed lights-off factory at his own company; recommends front-loading product design, system architecture, program design, and vertical slices instead of maximizing autonomy. | General Harness Engineering Design Space Resources |

<a id="p1-strong-candidates"></a>

## Further research and engineering sources

| Date | Resource | Core content | Topic |
|:---:|:---|:---|:---|
| 2026-07-22 | [Building verification loops in Claude Code with skills](https://claude.com/blog/building-verification-loops-in-claude-code-with-skills) | The Claude Code team's pattern catalogue for encoding verification as skills, naming four deployment shapes: standalone skills invoked manually for cross-cutting checks, embedded skills running inside a parent skill's workflow, chained skills where one invokes the next, and PR-wide gates applied team-wide. Also inventories what ships built in, including `/verify`, toolchain integration, Code Review, and rubrics in managed agents. Treats verification as a composable unit of the harness rather than a prompt instruction. | Research & Engineering Blogs |
| 2026-06-30 | [Loop engineering: Getting started with loops](https://claude.com/blog/getting-started-with-loops) | Anthropic's own taxonomy of loops — agents repeating cycles of work until a stop condition is met — split into turn-based, goal-based (`/goal`), time-based (`/loop` locally, `/schedule` in the cloud), and proactive event-triggered loops with no human in real time. Leans on the extensibility surfaces this repo tracks: skills carrying verification steps, auto mode removing the between-cycle prompt, dynamic workflows fanning out, Code Review supplying a second opinion. The vendor-side counterpart to the Osmani and LangChain loop essays already indexed. | Research & Engineering Blogs |
| 2026-07-21 | [How Datadog built a "universal machine tool" for Claude Code](https://claude.com/blog/how-datadog-built-a-universal-machine-tool-for-claude-code) | Datadog's Temper inverts the usual arrangement: agents do not emit application code, they emit specifications that a deterministic kernel verifies and executes, across three contract layers (behavior contracts with states and safety properties, data contracts, default-deny scope-based authorization) and four verification layers ending in randomized property testing. The design rule — "the artifact that gets verified is the artifact that runs" — eliminates drift between reviewed and executed code. Its blunt guidance, "is your real bottleneck generation or verification? Assume verification," is the sharpest enterprise counterpoint to tool proliferation. | Research & Engineering Blogs |
| 2026-07-17 | [Zero risk isn't the job: a CISO's guide to agentic AI](https://claude.com/blog/ciso-guide-to-agentic-ai) | A four-question risk model (what untrusted content does the agent ingest, what actions can it take and under whose identity, what is the blast radius, what observability exists) plus seven controls: IdP-sourced identity, admin connector allowlists, per-tool verb-level approval, a sandboxed agent loop on managed infrastructure, egress allowlisting through a proxy, OpenTelemetry streaming to the org SIEM, and an org-wide connector kill switch. The generalized complement to the SDLC post above. | Research & Engineering Blogs |
| 2026-07-07 | [Choosing a Claude model and effort level in Claude Code](https://claude.com/blog/claude-model-and-effort-level-in-claude-code) | The clearest official account of effort as a harness dial rather than a thinking budget: sent as part of the request, with the model trained to respond to each level, governing how many files get read, how thoroughly work is verified, and how many steps run before checking back. The diagnostic rule is crisp — if Claude lacked context, change the model; if it skipped steps or failed to verify, raise effort. The practical companion to the Sonnet 5 docs note about `thinking` and `temperature` now returning 400. | Research & Engineering Blogs |
| 2026-06-30 (+07-01) | [Redeploying Claude Fable 5](https://www.anthropic.com/news/redeploying-fable-5) | Closes the story the README already tells through the suspension statement. The US export controls that forced Fable 5 and Mythos 5 offline on June 12 were lifted on June 30, and Anthropic redeployed Fable 5 globally on July 1 behind an improved safety classifier aimed at the specific jailbreak Amazon researchers had used to extract vulnerability-exploitation guidance; Mythos 5 returned only to a set of US organizations. The stated reason for the original blanket shutdown remains the interesting part — Anthropic had no reliable way to verify nationality in real time, so it disabled the models for everyone. | Research & Engineering Blogs |
| undated | [Choose a sandbox environment](https://code.claude.com/docs/en/sandbox-environments) | A six-row comparison of the built-in sandboxed Bash tool, the `@anthropic-ai/sandbox-runtime` beta that wraps the whole process so hooks and MCP servers land inside the boundary too, dev containers, custom containers, VMs, and Anthropic-hosted web sessions, indexed by what each isolates. States plainly that the Bash sandbox alone is insufficient for unattended runs because built-in tools, MCP servers, and hooks run unconstrained on the host, and distinguishes auto mode's classifier as **"a per-action control, not an isolation boundary."** Also spells out which approaches an organization can enforce — only the Bash sandbox, via managed settings. The documentation has no publication date. | Product Documentation |
| 2026-07-13 | [Making Fable Cheaper Than Opus](https://cognition.com/blog/making-fable-cheaper-than-opus) | A measured comparison showing a Fable-5-led agent comes out 9% cheaper and 11% better-scoring than an Opus-4.8-led one despite Fable being significantly more expensive per token, because of when and how the lead delegates. (The separate 54% saving quoted in the post is Fable-plus-sidekick measured against pure Fable, not against Opus.) Fable hands off exploration early while Opus delegates only the mechanical tail after the expensive design work, so Fable consumes 545K input tokens against Opus's 1,679K and takes 11.5 lead turns against 26.5. Briefs specify constraints rather than implementations ("operator() must be O(1) in pointer length: NO full token scan"), and corrections are issued as further cheap handoffs. Rare quantitative evidence that delegation policy, not model price, sets agent cost. | Cross-Vendor Code-Agent Engineering |
| 2026-07-01 | [Introducing Devin Security Swarm](https://cognition.com/blog/introducing-devin-security-swarm) | Agentic-MapReduce applied to security scanning: parallel agents shard the codebase, each finding is reproduced in an isolated sandbox to confirm runtime exploitability, and only verified vulnerabilities with attack paths reach the security team before remediation PRs open. Against 50 real vulnerabilities from published GitHub advisories it reached 72% recall at $90.23 per run, finding three criticals competing tools missed. Scan profiles are generated from threat models and deployed org-wide without per-repo setup; later scans cover only changed code. The productized counterpart to the Agentic MapReduce post already indexed. | Cross-Vendor Code-Agent Engineering |
| 2026-07-16 | [Building scalable AI agents with modular prompt transpilation](https://developers.googleblog.com/en/building-scalable-ai-agents-with-modular-prompt-transpilation/) | Google SRE treats prompts as build artifacts rather than static text: separate skill modules composed through templating with includes, variable injection, and macros, then transpiled into a deterministic final prompt. The transpiler catches missing imports, undefined variables, and circular dependencies at build time; CI regenerates and diffs the output against what is deployed to catch drift; agents can propose prompt changes as pull requests. Progressive disclosure is explicit — the agent retrieves only the skill modules the task needs. A build-system answer to the instruction-layer sprawl that CLAUDE.md and skills accumulate. | Cross-Vendor Code-Agent Engineering |
| 2026-07-08 | [NVIDIA Nemotron Achieves Benchmark-Leading Performance With LangChain Deep Agents Harness](https://blogs.nvidia.com/blog/nemotron-langchain-agents-open-stack/) | LangChain read execution traces to find failure modes, then tuned only the harness around Nemotron 3 Ultra — system prompts, tool descriptions, middleware — with no model changes, reaching the highest accuracy among open models and business-task parity with top closed models at roughly 10x lower inference cost per run. The stated conclusion is the thesis in one line: **"every gain came from engineering the environment around the model, not the model itself."** One of the few vendor data points with a cost delta attached. | Cross-Vendor Code-Agent Engineering |
| 2026-07-10 | [Side Chats and Conversation Search (Cursor 3.11)](https://cursor.com/changelog/side-chat) | Side chats are durable parallel agent conversations spawned from the main thread with `/side` or `/btw` that inherit its context, so a clarification can be explored without the main agent losing its footing. More significant is the new cloud-agent hook surface — `beforeSubmitPrompt`, `afterAgentResponse`, `afterAgentThought`, `subagentStart`, plus hooks on compaction and turn completion — which makes the agent's own reasoning and compaction observable and interceptable, going further than hooks that only wrap tool calls. | Cross-Vendor Code-Agent Engineering |
| 2026-07-23 | [Agent automation controls in GitHub Issues in public preview](https://github.blog/changelog/2026-07-23-agent-automation-controls-in-github-issues-in-public-preview/) | A permission design that gates on the agent's own **self-assessed confidence** rather than on action type: agents rate each action high, medium, or low; high-confidence changes apply automatically; medium and low are held as suggestions in a review panel, with repository admins setting the threshold. Every supported action emits a rationale, producing an audit trail of what changed and why. A third answer alongside Claude Code's rule-based and classifier-based approaches. | Cross-Vendor Code-Agent Engineering |
| 2026-07-28 | [Bringing MCP 2026-07-28 to Claude](https://claude.com/blog/bringing-mcp-2026-07-28-to-claude) | The client-side half of the MCP spec release: Claude adopts MCP Apps for inline interactive tool UI, enterprise-managed authentication so connectors are provisioned org-wide from the IdP, developer observability dashboards over connector performance, and MCP Tunnels in research preview for reaching private-network servers without exposing them publicly. Useful next to the spec post because it shows which parts of a protocol change a host implements first. | Cross-Vendor Code-Agent Engineering |
| 2026-07-20 | [How Kiro and Snyk create multi-layered security guardrails](https://kiro.dev/blog/kiro-and-snyk-guardrails/) | Four layers wired through MCP into the Kiro IDE: natural-language-triggered repository scanning, an AI Bill of Materials tracking models and datasets as supply-chain components, Toxic Flow Analysis tracing how user input traverses the system to catch prompt injection, and Agent Hooks that scan in the background on events such as file save. The AI-BOM and the event-driven hook are the notable parts — security shifts from a review step to a continuously-firing harness extension. | Cross-Vendor Code-Agent Engineering |
| 2026-06-25 | [Introducing Agent Focus](https://kiro.dev/blog/introducing-agent-focus/) | An experimental chat-first surface in the Kiro IDE built around "defining, refining, and directing work that an agent will execute" rather than editing code: an Agents panel listing independent sessions grouped by workspace with status, a center chat with inline diffs, and an auxiliary panel surfacing file changes and spec summaries on demand. A control-surface data point — when delegation becomes the default, the IDE reorganizes around supervising sessions instead of around files. | Cross-Vendor Code-Agent Engineering |
| 2026-07-28 | [Coding Agents and Technical Debt](https://www.openhands.dev/blog/coding-agents-and-technical-debt) | All Hands quantifies the cost of forking a coding agent instead of tracking upstream, citing forks running "2,600 merged PRs behind, including 866 bug fixes." A maintenance-economics argument rather than an architectural one, and a useful counterweight in a section otherwise full of clean-room reimplementations: the harness is a moving target, and copying it has a carrying cost. | Cross-Vendor Code-Agent Engineering |
| 2026-07-07 / 07-08 | Birgitta Böckeler — [Viability of local models for coding](https://martinfowler.com/articles/exploring-gen-ai/local-models-for-coding-factors.html), [Experiences with local models for coding](https://martinfowler.com/articles/exploring-gen-ai/local-models-for-coding-experiences.html) | Two memos that reach a harness conclusion from the hardware side. RAM is the binding constraint, 32K context suffices for small tasks but 64K is regularly needed, KV cache competes directly with model weights, and tool calling still fails often though models usually self-recover. The finding worth pairing with "Better Models: Worse Tools" is that harnesses ship incompatible edit-tool schemas — Pi uses `old_text`/`new_text`, OpenCode uses `oldString`/`newString` — and that inconsistency is itself context-window overhead resource-limited setups cannot absorb. | Cross-Vendor Code-Agent Engineering |
| 2026-07-22 | [Agent Sandboxing: What OpenAI got wrong with the HuggingFace hack](https://www.openhands.dev/blog/agent-sandboxing-what-openai-got-wrong-with-the-huggingface-hack) | A vendor post-mortem whose central claim is architectural: **"a proxy is a network service"** with its own attack surface, so controlled access is not isolation, and an evaluation environment that needs no internet should be air-gapped rather than proxied. Then specifies the controls: internal package registries, mocked external APIs, default-deny egress firewalls, human approval gates specifically on secret-store and credential-file operations, and structured event logging with anomaly detection. A useful contrast with Anthropic's own egress-allowlist position. | Security Research & Incidents |
| 2026-07-08 | [Friendly Fire: Hijacking Defensive Cyber AI Agents for Remote Code Execution](https://ainowinstitute.org/publications/friendly-fire-exploit-brief) | A working exploit against the defensive use case specifically: ask Claude Code or Codex CLI to security-review an untrusted third-party repository, and layered prompt injections in ordinary-looking files get the agent to run a fabricated `security.sh` that invokes an obfuscated binary, achieving RCE **without an approval prompt**. The researchers note existing mitigations did not prevent it, no CVE was assigned, and no patch exists, because the fix is a change in how these agents are permitted to operate rather than a version bump. Names prompt fatigue as an attack enabler, connecting to the approval-rate findings already tracked. | Security Research & Incidents |
| 2026-07-10 | [SLBench: Evaluating How LLM Agents Follow Logical Relations in Skills](https://arxiv.org/abs/2607.09016) | Skills are usually treated as prose, but they encode preconditions, constraints, and fallbacks the agent is supposed to honor. SkillLogic identifies eight kinds of logical relation in skill files and SLBench turns the high-impact ones into 86 executable cases; Codex and Claude Code show unsafe rates up to 70%, producing privacy leaks and unsafe configurations, and an inference-time scaffold called SLGuard cuts violations by 63% on targeted cases. The safety-side companion to the SKILL.md anatomy study already tracked. | Security Research & Incidents |
| 2026-07-13 | [Rethinking MCP Security: A Large-Scale Study of Runtime MCP Servers and Security Scanner Reliability](https://arxiv.org/abs/2607.11086) | Builds MCPZoo — 64,611 unique MCP servers (113,927 total), more than 37,288 amenable to dynamic analysis — and then audits the scanners rather than the servers. Existing tools flag 96.89% of servers as risky, but manual validation puts the true-positive rate below 50%, and the scanners disagree with each other substantially. Establishes that current MCP security tooling produces alert volume rather than signal, which matters for anyone deciding what to gate an MCP install on. | Security Research & Incidents |
| 2026-07-26 (v2 07-29) | [Where Is the Cost of Third-Party API Routers in Agentic Software Development?](https://arxiv.org/abs/2607.23624) | Studies the routing layer many teams put between a coding agent and a model provider, and shows it is an unguarded trust boundary: the SIDEL framework injects at four intervention levels over a 400-sample dataset, and all four coding agents tested achieved a defense success rate of **0% at every level**. Router-side injection substantially changes repository actions with no detection anywhere between provider output and executed action. A trust boundary the design space does not usually draw at all. | Security Research & Incidents |
| 2026-07-29 | [MemSecBench: Tracking Agent Memory Poisoning from Persistence to Consequence and Repair](https://arxiv.org/abs/2607.27080) | Extends the memory-injection question past whether a write lands to whether it survives and whether it can be cleaned up, with 310 cases across 24 configurations under a Write-Execute-Forget protocol. Malicious memory persists in 84.2% of cases, the full Write-Execute chain succeeds in 50.3%, and the choice of memory backend moves attack success by up to 16.1 percentage points. Turns the asymmetry the Bad Memory entry identifies into a lifecycle measurement, including the repair stage nobody usually evaluates. | Security Research & Incidents |
| 2026-07-26 | [Are You Still the Agent I Authorized? Earned Authority under a Fixed Ceiling for Evolving Agents](https://arxiv.org/abs/2607.23586) | Asks what an approval means once the agent it was granted to has changed — acquired new tools, new skills, new learned behavior. The proposed state-bound model fixes a transition envelope and an immutable effect ceiling at grant time, proves that under complete mediation and sound effect abstraction no mutation can amplify protected effects past the user's limit, and maps six mutation classes to their authorization consequences. The temporal counterpart to PORTICO: PORTICO expires authority as the task moves on, this bounds it as the agent itself moves on. | Related Academic Papers |
| 2026-07-21 | [Twin Agent: Context Residual Compression for Privilege Separated Agents](https://arxiv.org/abs/2607.19595) | Splits the loop into an Explore Agent that reads untrusted content and a Safe Agent that holds the privilege to act, and makes the split practical by passing only a compact residual between them rather than the full observation. Evaluated on SWE-bench Lite and AgentDojo, it holds task utility while blocking injections, beating both undefended and prior privilege-separated baselines. Concrete evidence on the cost of the plan-execute privilege boundary — the architectural move most often proposed and least often measured. | Related Academic Papers |
| 2026-07-13 | [Phantom Guardrails: When Self-Improving Agent Harnesses Fix Failures That Never Happened](https://arxiv.org/abs/2607.13083) | Builds a Counterfactual Fabrication Lab — a deterministic micro-environment where the correct action is known to be doing nothing — and finds that in 15 of 60 runs on legal input containing rule-shaped patterns, the harness optimizer invented guardrails and cited violations an oracle refutes. Because acceptance loops are typically add-only, a fabricated guardrail never gets removed. A distinct failure mode from the generalization concern the existing harness-evolution skeptic raises, so it sharpens that pairing rather than repeating it. | Related Academic Papers |
| 2026-07-12 | [When Does Restricting a Coding Agent to execute_code Help? A Regime × Agent-Design Ablation](https://arxiv.org/abs/2607.10569) ([code](https://github.com/hyang0129/onlycodes)) | A clean ablation over tool surfaces — baseline, `bash_only`, `code_only` — run with both Claude Code and OpenAI Codex on synthetic tasks and SWE-bench Mini. Collapsing to a single `execute_code` tool is cheaper than or statistically tied with the cheapest tool-rich rival in three of four cells, the exception being SWE-bench with Claude, where cost rose 14.4%. The useful conclusion: the cheapest tool surface depends jointly on task regime and agent design, so tool-count minimalism is not a portable rule. | Related Academic Papers |
| 2026-07-28 | [HANDBOOK.md: A Benchmark for Long-Context Agentic Instruction Following](https://arxiv.org/abs/2607.25398) ([harness](https://github.com/surge-ai/handbook)) | Tests whether a standing policy document actually constrains an agent over a long tool-use horizon: 65 tasks across finance, medical, insurance, logistics, and HR, each governed by a 20-to-124-page handbook, graded strictly so all 824 programmatic criteria must pass. The best configuration reaches 36.2%, and the characteristic failure is prioritizing a plausible user request over standing policy while losing rule details. The closest thing to a measurable test of what CLAUDE.md-style persistent instructions are worth at scale. Comments field confirms acceptance to the Workshop on Agent Behavior (WAB) at COLM 2026. | Evaluation & Benchmarks |
| 2026-07-28 | [OrchBench: Evaluating Multi-Agent Orchestration Plans in Isolation via Deterministic Simulation](https://arxiv.org/abs/2607.25656) | Scores an orchestration plan without running the workers, by expressing task dependencies as a DAG and evaluating quality, makespan, and token cost under deterministic simulation. Reports a Pearson correlation of r=0.816 against actual Claude Code execution while consuming 1.3% of the tokens and 10.3% of the wall-clock time. Makes delegation-plan quality a cheap, isolated measurement rather than something observable only after paying for the whole fan-out. | Evaluation & Benchmarks |
| 2026-06-24 | [Necessary but Not Sufficient: Temperature Control and Reproducibility in LLM-as-Judge Safety Evaluations](https://arxiv.org/abs/2606.26185) | Finds LLM graders are not deterministic even at temperature 0, with per-item disagreement up to roughly 50% over 20 runs when the harness does not explicitly configure temperature, and 1-2 of 7 borderline items still irreproducible under forced greedy decoding. Argues that reporting a single verdict without measuring grader disagreement lets noise be read as a safety result. Slots beside the judge-swap audit already listed, and locates the defect in **harness configuration** rather than judge choice. | Evaluation & Benchmarks |
| 2026-07-21 | [Expenditure Horizon: Measuring Optimization Ability, with an Application to NanoGPT](https://metr.org/blog/2026-07-21-expenditure-horizon/) | METR proposes the expenditure horizon — the budget at which human and agent cost-effectiveness curves cross — and estimates it at $3,300 for Opus 4.8 and $2,300 for GPT-5.5 on NanoGPT optimization (GPT-5.2 lands lower at $600 to $1,300; GPT-5 and Opus 4.1 make no meaningful progress), against roughly $2,500 per 1% improvement for human labor. The harness caveat is stated in the paper: their setup was "likely inefficient," experiment execution accounted for 70-90% of cost, and "we expect a more optimized harness would significantly lower cost for a given optimization." An evaluation methodology that makes the harness's cost contribution explicit rather than incidental. | Evaluation & Benchmarks |
| 2026-07-12 | [Claude Code Is Way More Token-Hungry Than OpenCode. We Measured Exactly How Much](https://systima.ai/blog/claude-code-vs-opencode-token-overhead) | A logging-proxy measurement against Claude Code 2.1.207 and OpenCode 1.17.18 on the same model. Roughly 33,000 tokens of system prompt, tool schemas, and injected scaffolding arrive **before** the user prompt versus about 7,000 for OpenCode, and Claude Code rewrote tens of thousands of prompt-cache tokens mid-session while OpenCode's request prefix stayed byte-identical. Also prices the extension surface — a 72KB instruction file adds about 20,000 tokens per request, five MCP servers add 5,000 to 7,000 — and prices delegation at 121,000 tokens direct versus 513,000 fanned out to two subagents, with identical completion rates. | Evaluation & Benchmarks |
| 2026-07-22 | [Visual Studio Code 1.130 release notes](https://code.visualstudio.com/updates/v1_130) | The follow-on to the 1.129 Agent Host release already indexed, adding Assisted Tool Approvals, where a language model rates each tool call's risk so long agent runs stop re-prompting on the safe ones. Agent sessions now run in Git worktrees for parallel work in one workspace, and worktree support plus multi-file compact-diff review apply uniformly across Copilot, Claude, and Codex behind the Agent Host Protocol. A direct comparison point for Claude Code's auto-mode classifier and Cursor's in-loop auto-review. | Runtime & Sandbox Infrastructure |
| 2026-07-28 | [Vercel Sandbox supports forking](https://vercel.com/changelog/vercel-sandbox-supports-forking) | `Sandbox.fork()` seeds a new sandbox from the source's latest saved snapshot, inheriting its configuration and environment variables, with creation parameters overriding what is inherited. Deliberately different from E2B's in-place checkpoint already indexed — Vercel forks from a saved snapshot and takes about as long as a fresh sandbox of the same size, rather than pausing and resuming the live machine. Together the two mark forking as a converged primitive for branching one expensively-configured environment across parallel attempts. | Runtime & Sandbox Infrastructure |
| 2026-07-20 | [Amazon CloudWatch announces coding agent insights](https://aws.amazon.com/about-aws/whats-new/2026/07/cloudwatch-coding-agent-insights/) | CloudWatch now ingests OpenTelemetry metrics emitted by coding agents — Claude Code with no extra instrumentation via the Claude apps gateway for AWS, plus Codex and GitHub Copilot — presenting spend, token consumption, delivery velocity, and per-model cost-effectiveness alongside existing operational data. A small announcement with a large implication: agent telemetry is becoming a standard observability stream, making cross-harness measurement an infrastructure feature rather than a bespoke script. | Runtime & Sandbox Infrastructure |
| 2026-07-06 | [clawkwork/clawk](https://github.com/clawkwork/clawk) | A disposable-VM sandbox for coding agents rather than a container: Apple Virtualization.framework on Apple silicon, Firecracker microVMs on Linux, OCI images built into ext4 disks with copy-on-write clones per sandbox. Two properties matter for the safety axis — the guest runs its own kernel so the host filesystem is invisible except for explicit mounts, and egress is deny-by-default with a userspace allow-list that records what the agent tried to reach. Agents launch with full autonomy flags by default, which is the honest version of the tradeoff: give the agent everything inside a boundary you are willing to destroy. ~841 stars, Apache-2.0. | Runtime & Sandbox Infrastructure |
| 2026-07-14 | [What is "loop engineering?"](https://newsletter.pragmaticengineer.com/p/what-is-loop-engineering) | The trade-press synthesis of the term this repo already organizes around, and it supplies the provenance chain: Boris Cherny's "I don't prompt Claude anymore, I have loops running that prompt Claude" at Anthropic's developer conference, Steinberger's OpenClaw post, Osmani's essay, and Huntley's Ralph Wiggum technique as the origin. Also inventories the vendor convergence on a shared primitive — `/goal` in Codex, Hermes, and Claude Code, plus `/loop` for scheduled runs — and ends with a useful deflation: most reported uses look like webhooks, Zapier, and cron. | General Harness Engineering Design Space Resources |
| 2026-07-18 | [lopopolo/harness-engineering](https://github.com/lopopolo/harness-engineering) | Three resources in one repository: an anthology of harness-engineering source material, a field guide with a thesis index and domain-modeling and durability sections, and an agent context bundle of playbooks the agent itself reads. The framing is sharper than most collections — hold the model constant and optimize the only two levers you control, context and tools — and the distinctive move is embedding organizational nonfunctional requirements directly in repository structure so agent trajectories can recover them, making "organizational judgment cumulative." ~2.4k stars. | General Harness Engineering Design Space Resources |
| 2026-07-15 | [Context engineering with Dex Horthy](https://newsletter.pragmaticengineer.com/p/context-engineering-with-dex-horthy) | The companion interview covers operating practices: a "smart zone" of roughly 300-400K tokens for large models and 100K for smaller ones before the "dumb zone," intentional compaction into Markdown summaries for fresh sessions, "slow loops" that open quality PRs overnight for human review, and "trajectory poisoning" as the failure mode where phrases like "you're completely right!" signal a session that should be restarted rather than continued. | Blog Posts & Technical Articles |
| 2026-07-17 | [Claude Code: Anatomy of a Misfeature](https://www.olafalders.com/2026/07/17/claude-code-anatomy-of-a-misfeature/) | A precise case study in how an approval gate can be silently voided. Claude Code 2.1.198 made `AskUserQuestion` auto-continue after 60 seconds with a countdown visible only for the last 40, shipped with no changelog entry, submitted partially-answered multi-part questions combined with model-guessed answers, and offered only an undocumented `CLAUDE_AFK_TIMEOUT_MS` escape hatch passed around in issue threads. Because Claude Code auto-updates by default, the semantics of a blocking human gate changed without operator review — the concrete version of a risk the permissions literature usually discusses abstractly. | Blog Posts & Technical Articles |
| 2026-07-20 / 07-05 | Jesse Vincent — [The Therapist Pattern](https://blog.fsck.com/2026/07/20/the-therapist-pattern/), [Some new agentic patterns](https://blog.fsck.com/2026/07/05/new-patterns/) | Two field reports from the author of `obra/superpowers` (already listed), but on a different problem: governing long-lived agents rather than one-shot coding runs. The Therapist Pattern makes a dedicated subagent the only writer of an agent's mutable `identity.md`, which is reinjected every turn, so behavioral corrections require introspection and land automatically rather than depending on recall. The patterns post describes a compartmentalized security architecture built around Willison's lethal trifecta: the main agent cannot talk to the outside world, only ephemeral subagents can, credentials live in a vault behind an arbiter agent, and a transparent MITM proxy swaps temporary tokens for real ones on the way out. | Blog Posts & Technical Articles |
| 2026-07-11 | [Old and new apps, via modern coding agents](https://terrytao.wordpress.com/2026/07/11/old-and-new-apps-via-modern-coding-agents/) | Terence Tao ports roughly two dozen dead Java 1.0 applets from 1999 to JavaScript and builds two new visualizers, reporting one minor bug across the whole migration while the agent independently found two pre-existing bugs in the originals. The value is the verification calculus he makes explicit: the artifacts are secondary visual aids, so residual-bug tolerance is high, and "the high level code design decisions still remain in the vibe coding model" while lower-level syntax is automated away. A rare high-credibility data point on where the autonomy boundary actually sits. | Blog Posts & Technical Articles |
| 2026-07-13 | [The Tower Keeps Rising](https://lucumr.pocoo.org/2026/7/13/the-tower-keeps-rising/) | Armin Ronacher's third entry in the arc already tracked, and the one that names the failure mode the harness cannot detect. Large codebases are held together by shared language produced through friction — code review, cross-team negotiation — and agents remove exactly that friction, letting one person change OAuth here and caching there without anyone reaching mutual understanding. The Babel inversion is the point: **"the tower does not fall, and so we do not notice what was lost,"** so there is no failure signal a verification loop could pick up. | Blog Posts & Technical Articles |
| 2026-07-14 | [xai-org/grok-build](https://github.com/xai-org/grok-build) | XAI's terminal coding agent released as source (~23.4k stars, Apache-2.0). The crate layout is legible as a harness taxonomy — `xai-grok-pager` for the TUI, `xai-grok-shell` for the agent runtime, `xai-grok-tools` for edit/exec/search, `xai-grok-workspace` for filesystem, version control, execution, and checkpoints — and it ships MCP servers, skills, plugins, and hooks, with interactive TUI, headless CI, and Agent Client Protocol embedding modes. Its CLI is also the subject of the wire-level analysis above. | Coding Agent CLIs and IDE Harnesses |
| 2026-07-25 | [VictorTaelin/OptMem](https://github.com/VictorTaelin/OptMem) | Persistent agent memory reduced to its smallest defensible form: a single dependency-free Python file, an append-only log, a binary tree of pairwise summaries in `TREE/`, and a 426-token prompt block pasted into agent instructions. The interface is four verbs — `memo wake`, `memo note`, `memo recall <regex>`, `memo zoom` — with fixed-width records for position-based lookup and about 0.03 seconds to search a million memories. It provides a minimal alternative to memory vaults and indexes. ~884 stars. | Memory and Persistent Context |

<a id="p2-verified-lower-priority"></a>

## Additional source references


| Date | Resource | Note | Topic |
|:---:|:---|:---|:---|
| 2026-07-06 | [The Making of Claude Code](https://www.anthropic.com/features/making-of-claude-code) | Official oral history (VS Code extension in 2021, Boris Cherny joining Sept 2024, the two-week December sprint, "we're only 1% done"). The page offers Read-in-Terminal and Read-as-Article views; the body content is unverified. | Research & Engineering Blogs |
| 2026-06-30 | [Introducing Claude Sonnet 5](https://www.anthropic.com/news/claude-sonnet-5) | The platform "What's New" doc already listed carries the harness-relevant detail; this adds launch pricing ($2/$10 through Aug 31, then $3/$15) and confirms the tokenizer change at roughly 1.0-1.35x more tokens. | Research & Engineering Blogs |
| 2026-07-22 | [Cursor Router](https://cursor.com/changelog/router) | Per-request classification by task type and complexity routes to a frontier or cheap model across three modes, with admin allow/block lists. Overlaps the model-routing ground Devin Fusion already covers. | Cross-Vendor Code-Agent Engineering |
| 2026-06-25 | [Deep Agents and OpenCode in the AI SDK Harness](https://vercel.com/changelog/deepagents-and-opencode-harness-adapters) | Brings the adapter count to five (Claude Code, Codex, Deep Agents, OpenCode, Pi) behind one interface. Incremental on the AI SDK 7 entry already indexed. | Cross-Vendor Code-Agent Engineering |
| 2026-07-15 / 07-23 / 07-28 | Microsoft Agent Framework — [Agent Skills for Python](https://devblogs.microsoft.com/agent-framework/agent-skills-for-python-is-now-released/), [Declarative Workflows 1.0](https://devblogs.microsoft.com/agent-framework/move-agent-orchestration-workflows-out-of-code-with-agent-framework-declarative-workflows-1-0/), [Discover Agent Skills from MCP servers in .NET](https://devblogs.microsoft.com/agent-framework/discover-agent-skills-from-mcp-servers-in-net/) | Skills-discovery-over-MCP is the interesting one (extension surfaces composing), but individually each is a release note.  | Cross-Vendor Code-Agent Engineering |
| 2026-07-14 | [5 Trends That Defined AI Engineering at World's Fair 2026](https://www.latent.space/p/aiewf26trends) | Trend 1 is the shift from agents to the harness around them, trend 2 loop engineering as the control layer, trend 5 every platform converging on skills. | General Harness Engineering Design Space Resources |
| 2026-03-29 | [ai-boost/awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering) | The closest competing list, ~3.3k stars, actively pushed through 2026-07-29, organized by design primitive (loops, planning, context delivery, tool design, skills/MCP, permissions, memory, orchestration, verification, observability). The July 2026 snapshot does not establish growth during that month. | General Harness Engineering Design Space Resources |
| 2026-07-14 | [DSLs Enable Reliable Use of LLMs](https://martinfowler.com/articles/llm-and-dsls.html) | Unmesh Joshi: a DSL shrinks the valid output space so few-shot examples suffice, and its parser and type-checker become a deterministic validator enabling autonomous repair loops. | Blog Posts & Technical Articles |
| 2026-07-24 | [engineer away the slop](https://ghuntley.com/slop/) | Geoffrey Huntley announces joining Antithesis and argues creation is near-free while verification stays scarce, so deterministic simulation goes mainstream. | Blog Posts & Technical Articles |
| 2026-07-19 | [Claude Code uses Bun written in Rust now](https://simonwillison.net/2026/Jul/19/claude-code-in-bun-in-rust/) | Binary `strings` analysis finding Bun v1.4.0 preview and 563 Rust source filenames in shipped Claude Code. Companions: "Rewriting Bun in Rust" (2026-07-08) and "Fable's judgement" (2026-07-03, delegating implementation to lower-tier models in subagents by the model's own judgement). | Blog Posts & Technical Articles |
| 2026-07-09 | [cosmtrek/mindwalk](https://github.com/cosmtrek/mindwalk) | Replays coding-agent sessions on a 3D map of the codebase; session-observability tooling. ~951 stars. | Skills and Harness Extensions |
| 2026-06-19 | [juggler-ai/juggler](https://github.com/juggler-ai/juggler) | GUI coding agent where "everything's a plugin, even the read/write/bash tools"; Yjs-backed sessions survive restarts including paused approval states; multi-client; AGPL-3.0 core with Apache-2.0 extension SDK. ~538 stars. | Coding Agent CLIs and IDE Harnesses |
| 2026-06-30 | [Archive228/loopkit](https://github.com/Archive228/loopkit) | 33 skills plus a minimal `.claude` harness portable across Claude Code, Cursor, Codex, and Gemini CLI. ~726 stars. | Skills and Harness Extensions |
| 2026-07-21 | [QoderAI/better-harness](https://github.com/QoderAI/better-harness) | Cross-agent harness self-improvement tooling. ~1,030 stars. | Skills and Harness Extensions |
| 2026-07-10 | [ShenSeanChen/waku-agent](https://github.com/ShenSeanChen/waku-agent) | Teaching harness: loop, memory, and eval in code readable in an afternoon. ~616 stars. | Open-Source Reimplementations |
| 2026-07-02 | [elder-plinius/T3MP3ST](https://github.com/elder-plinius/T3MP3ST) | Multi-agent offensive-security meta-harness from a well-known red-teaming figure. ~5.3k stars. | Security Research & Incidents |
| 2026-07-27 | [Authoring Agent Skills: A Software-Engineering Approach](https://arxiv.org/abs/2607.25032) | Uses Claude Code as the reference implementation; covers skill structure, staged content loading, and placement relative to project memory, hooks, and subagents. Prescriptive companion to the SKILL.md smells paper. | Related Academic Papers |
| 2026-07-17 (v2) | [Fantastic Adaptive Taxonomies and How to Use Them](https://arxiv.org/abs/2607.16387) | AdaMAST induces failure taxonomies from traces with no hand-authored codes; lifts Claude Code from 64.0% to 70.7% when installed as a runtime skill, and SWE-agent 60% to 70% on SWE-bench Verified Mini. | Related Academic Papers |
| 2026-07-13 (v2 07-23) | [The Hidden Footprint: Making Storage a First-Class Metric for LLM Agent Evaluation](https://arxiv.org/abs/2607.11149) | AgentFootprint across seven frameworks: configurations at identical 100% accuracy differ 15.7x in retained bytes; content-addressed storage cuts retention 4.8x-32.7x. | Evaluation & Benchmarks |
| 2026-07-24 | [Agent Team Work Zone: An Automated, Persistent Workspace for Long-Lived Claude Code Agent Teams](https://arxiv.org/abs/2607.22917) | Filesystem operations layer for Claude Code agent teams; explicitly targets irrecoverable teams and post-compaction knowledge erosion. | Related Academic Papers |
| 2026-07-28 | [CodeNib: A Multi-View Data System for Serving Repository Context to Coding Agents](https://arxiv.org/abs/2607.25431) | Lexical, dense, and structural views maintained incrementally; 8.7x and 25.4x faster median updates than rebuilds, 50-87% fewer trajectory tokens than paired grep/read. | Related Academic Papers |
| 2026-07-09 | [What to Keep, What to Forget: A Rate-Distortion View of Memory Compaction](https://arxiv.org/abs/2607.08032) | Survey unifying KV-cache, prompt-pruning, architectural-state, and agent-memory compaction under one rate-distortion frame; seven-axis taxonomy. | Related Academic Papers |
| 2026-07-27 | [SpecBox: Speculative Sandbox Scheduling for Efficient LLM Agent Serving](https://arxiv.org/abs/2607.23933) | Intent-driven sandbox prewarming for MCP-based agents: P99 latency down up to 2.9x, peak memory down 45.9%. | Runtime & Sandbox Infrastructure |
| 2026-07-14 | [Isolation as a First-Class Principle for LLM-Agent System Safety: Concepts, Taxonomy, Challenges and Future Directions](https://arxiv.org/abs/2607.12406) | Boundary-centric survey across user-agent, agent-tool, agent-execution, agent-agent, and system-environment boundaries. | Related Academic Papers |
| 2026-07-15 | [Agent Skill Security: Threat Models, Attacks, Defenses, and Evaluation](https://arxiv.org/abs/2607.13987) | SkillSec-Eval, lifecycle-aware across repository admission, retrieval, planner selection, execution, evolution; empirical on 327 real skills. | Security Research & Incidents |
| 2026-07-13 | [Agent Hacks Agent: Autoresearch for Production-Agent Red-Teaming](https://arxiv.org/abs/2607.11698) | Automated discovery loop against production Claude Code and Codex; frozen Vulnerability Concept Graph beats the strongest frozen discovery baseline by 14.2 points. | Security Research & Incidents |
| 2026-07-23 | [Tencent WorkBuddy Bench](https://arxiv.org/abs/2607.20911) | Reverse-engineers tasks from real commits rather than public issues; ships environment images and eval harness. | Evaluation & Benchmarks |
| 2026-07-08 | [DeepSWE: Measuring Frontier Coding Agents on Original, Long-Horizon Engineering Tasks](https://arxiv.org/abs/2607.07946) | 113 original tasks never contributed upstream, hand-written verifiers accepting any correct implementation; reference solutions touch 5.5x more code than SWE-Bench Pro. | Evaluation & Benchmarks |
| 2026-07-29 | [A First Look at Coding Agents' Compliance with AI Contribution Rules in Open-Source Communities](https://arxiv.org/abs/2607.26819) | RepoComplianceBench, 106 issues from 49 repos: agents almost never proactively retrieve contribution rules and never refuse to contribute in AI-banned repos. | Evaluation & Benchmarks |
| 2026-07-24 | [Claim Plane: Enforceable Change Intents and Dynamic Scope for Parallel Coding Agents](https://arxiv.org/abs/2607.21909) | Pre-write admission for concurrent agents via versioned ChangeIntents. The results come from a six-pair feasibility study. | Related Academic Papers |

<a id="older-than-the-window-but-genuinely-missing"></a>

<a id="earlier-sources-flagged-for-backfill-in-the-july-sweep"></a>

## Earlier sources


| Date | Resource | Why it matters | Topic |
|:---:|:---|:---|:---|
| 2026-06-02 | [A harness for every task: dynamic workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code) | Names six orchestration patterns (classify-and-act, fan-out-and-synthesize, adversarial verification, generate-and-filter, tournament, loop-until-done) and three single-context failure modes it exists to fix: agentic laziness, self-preferential bias, and goal drift. Anthropic's workflow documentation cites it as a companion article. | Research & Engineering Blogs |
| 2026-06-01 | [Poisoning Claude Code: One GitHub Issue to Break the Supply Chain](https://flatt.tech/research/posts/poisoning-claude-code-one-github-issue-to-break-the-supply-chain/) | GMO Flatt Security (RyotaK) documents two potential routes rather than a completed compromise: agent mode trusted any GitHub App actor; separately, an official triage example could be chained into privileged tag mode. Prompt-injected Claude could then read OIDC request credentials from `/proc/self/environ` and exfiltrate them via issue updates. Fixed by `claude-code-action` v1.0.94. | Security Research & Incidents |
| 2026-06-18 | [Steering Claude Code: when to use CLAUDE.md, skills, hooks, and subagents](https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more) | The canonical official decision guide across the extensibility surfaces. | Research & Engineering Blogs |
| 2026-05-25 | [Harness, Scaffold, and the AI Agent Terms Worth Getting Right](https://huggingface.co/blog/agent-glossary) | Separates model, scaffold (system prompt, tool descriptions, parsing, inter-step memory) and harness (the execution loop), then situates policy, tools, skills, subagents, and context engineering around them. Directly useful for terminology framing. | General Harness Engineering Design Space Resources |
| 2026-06-17 | [Bringing more agent harnesses and frameworks to Cloudflare, starting with Flue](https://blog.cloudflare.com/agents-platform-flue-sdk/) | A clean three-layer split (framework Flue / harness Pi / runtime Agents SDK) plus Durable Streams, an append-only ledger of every prompt, tool response, and model choice, with `runFiber()`, `stash()`, and `onFiberRecovered()` for exact-checkpoint resumption. | Runtime & Sandbox Infrastructure |
| 2026-05-13 | [New in Deep Agents v0.6](https://www.langchain.com/blog/deep-agents-0-6) | ContextHubBackend makes agent-behavior files a versioned, diffable, reviewable, environment-tagged repo; Harness Profiles do per-model tuning; Delta Channels cut checkpoint overhead 10-100x. | Cross-Vendor Code-Agent Engineering |
| 2026-06-22 | [The Verification Stack](https://www.openhands.dev/blog/20260506-the-verification-stack) | Layered automated verifiers catching different classes of mistake at different stages. | Cross-Vendor Code-Agent Engineering |

<a id="verification-status"></a>

<a id="verification-record-from-the-july-sweep"></a>

## Source conditions

The OpenAI incident article is described through secondary accounts; its body was not directly available.

Numerical claims in these historical notes summarize source reports.


- Venue information comes from the sources' comments fields.
- **Workshop status:** `2607.12338` reads `KDD 2026 Workshop Agentic AI Evaluation and Trustworthiness`; `2607.10569` reads `Accepted to the Agentic Software Engineering (SE 3.0) Workshop at KDD 2026 (non-archival)`. Both refer to workshops, one explicitly non-archival. `2607.25398`'s WAB@COLM 2026 acceptance is quoted from its comments field.
- The OpenAI incident post is dated July 21, 2026. Its title, date, and quoted statements are reported through accounts that cite its canonical URL.
- The body content of `The Making of Claude Code` is unverified.
- **`ai-boost/awesome-harness-engineering` was created March 29, 2026** and was active in July.
- Star counts are approximate historical values.

<a id="deliberately-rejected"></a>

## Other references

- **Harness evolution:** `2607.26598` Living-Harness, `2607.13683` Self-Evolving Agent Harnesses via Gated Semantic Quality-Diversity, `2607.14159` MemoHarness, `2607.22688` Co-Harness, `2607.13285` Harness Handbook, `2607.14004` Do Agent Optimizers Compound?, `2607.11423` ToFu. `2607.14004` tests whether optimizer gains compound over phases.
- **Model training:** `2607.24653` Kimi K3, `2607.22083` Nanbeige4.2-3B, `2607.12463` Function-Aware FIM mid-training, `2607.27146` MindForge, `2607.05378` CompactionRL, `2607.14171` Branching Policy Optimization.
- **Domain-application agents:** SIREN, AGENTS4GEOS, SMEFT-Pheno-Agent, PatientAgentBench, ClinLens, TREK.
- **Earlier references:** `2606.21338` What Happens Locally, Leaks Globally (2026-06-19); Steve Yegge's "The Flat Curve Society" (2026-06-19).


<a id="unverified-leads--do-not-publish-without-confirming"></a>

<a id="leads-unresolved-at-the-july-cutoff"></a>

## Additional source notes


| Source | Date and scope |
|:---|:---|
| Bloomberg Odd Lots — Boris Cherny interview, reportedly 2026-07-20 | Date appears in the URL and search results, but the page is paywalled and could not be fetched. |
| OpenAI open-sources the Codex Security CLI and SDK (`github.com/openai/codex-security`) | First package publication: July 28, 2026. |
| Anthropic's response to China's CNVD "backdoor" advisory (2026-07-08) | CNVD flagged Claude Code v2.1.91-2.1.196 for transmitting location and identity signals; the only reply found is an X post from a staff member describing a March 2026 anti-abuse experiment removed in v2.1.198. No official Anthropic blog or advisory to cite. |
| Dan Luu — "Agentic test processes, LLM benchmarks, and other notes on agentic coding" | Strong content (agents fabricating evidence, LLM-written tests being poor while LLM-directed fuzzing finds real bugs fast), but danluu.com carries no on-page publication date; the only anchor is a 2026-07-04 HN submission. |
| ykdojo — "How to set up your spare Mac for Claude Code to fully control" | Hardware-isolation sandboxing guide. No on-page date; HN submission 2026-07-18. |
| MiniMax on self-evolving harnesses | Describes a 100+ round analyze-trajectories → modify-scaffold → evaluate → keep-or-revert loop for a 30% internal gain in an article dated March 18, 2026. |
| Senior SWE-bench | [Primary site](https://senior-swe-bench.snorkel.ai/), with a link to its methodology. |
