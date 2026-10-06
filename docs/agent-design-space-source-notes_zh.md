[返回主 README](../README_zh.md)

<a id="agent-systems-design-space-新进展资料记录"></a>

# 智能体系统设计空间：来源笔记

更新日期：2026 年 10 月 6 日。

本页说明[资源目录](../README_zh.md)和[设计指南](./build-your-own-agent_zh.md)所涉及的机制、实现选择及版本适用条件。[English version](./agent-design-space-source-notes.md)。

截至 2026 年 9 月 15 日的历史来源笔记位于本节之后。

<a id="用语约定"></a>

<a id="refresh-2026-10-06"></a>

## 智能体设计来源：2026 年 10 月 6 日

除另有说明外，日期均为 2026 年；release 时间使用 UTC。arXiv 论文为预印本，实验结果来自论文作者的报告；厂商给出的数字未经独立核实。Claude Code 架构分析对应 v2.1.88；下文中的更新版本说明后续变化。

<a id="refresh-2026-10-06-graph"></a>

### 工作图与控制循环

<a id="source-cursor-rollouts-verdicts"></a>

#### Cursor Rollouts：部署之后再检查变更

**9 月 23 日，面向 Teams 和 Enterprise 的 changelog 条目。** [Changelog 条目](https://cursor.com/changelog/rollouts-and-security-reviewer)

PR 打开时，Rollouts 发布一份作者可以编辑的监控计划，列出风险、预期效果、要检查的信号和缺失的监测。每次部署后，它按环境分别给出结论：健康、回归或无法判断。发现回归时，视配置而定，它可以开一个待审查的 revert PR，或把结果交给云端智能体处理，但不会自行合并或回滚。保留“无法判断”这一结论，可以避免把缺少证据当作通过。

<a id="source-adk-abort-resume"></a>

#### Google ADK：中止、审批暂停与恢复时重跑

**10 月 1 日，ADK Python 2.11.0；相关变化见 9 月 10 日的 2.9.0。** [v2.11.0 发布说明](https://github.com/google/adk-python/releases/tag/v2.11.0) · [v2.9.0 发布说明](https://github.com/google/adk-python/releases/tag/v2.9.0)

ADK Python 2.11.0 允许用中止信号优雅地停止 Runner、工作流或单个节点；工作流中的工具节点现在会暂停等待用户批准，而不是把错误传给下游。自 9 月 10 日发布的 2.9.0 起，工作流恢复时会重新运行失败的节点；此前这类节点会被当作已完成重放。因此发布说明要求节点体保持幂等：一个执行了副作用后失败的节点，每次恢复都会再次执行该副作用。以上说明只针对 ADK Python。

<a id="source-subgoal-authorization"></a>

#### 子目标授权：把重新规划视为权限变化

**10 月 4 日，arXiv v1。** [论文](https://arxiv.org/abs/2610.04975v1)

Zhu 与 Wang 把新建、替换、委派或合并子目标都当作授权事件。每次修改都要经过与当前策略和状态版本绑定的结构化检查，确认新的延续仍在已批准的任务之内；受保护的副作用在提交时还要再检查一次。逐次调用的权限检查看不到这一点，因为两个各自被允许的动作组合起来可能违反任务约束。评估使用有限的结构化领域和合成用例，属于研究原型，不是已部署的控制机制。

<a id="source-opencollab-adherence"></a>

#### OpenCollab：声明的组织是否真的在运行？

**9 月 29 日，arXiv v1。** [论文](https://arxiv.org/abs/2609.38345v1)

OpenCollab 用事件记录衡量声明的多智能体组织（角色边界与通信拓扑）在运行时是否真正实现。在不加约束的默认配置下，47.2% 的运行遵循了声明的结构；主智能体会自己用掉预算而不委派，或绕过队友。限制工具边界后，这一比例超过 90%，但在只读设置下任务成功率下降。配置中的拓扑只是一个请求，在把结果归因于它之前，应先检查事件记录。

<a id="source-claude-loop-wakeups"></a>

#### Claude Code：循环唤醒是运行状态

**9 月 23 日至 10 月 5 日，v2.1.281 至 v2.1.290。** [CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [v2.1.290](https://github.com/anthropics/claude-code/releases/tag/v2.1.290)

这些版本修复了 `/loop` 唤醒和定时任务在压缩、恢复、转入后台、更新和容器重启时的保存问题。自 v2.1.284 起，自定步调的循环把每次状态更新和停止结果写成可见文本。待执行的唤醒有独立的生命周期，在上述每个边界都可能丢失或重复。来源是发布说明，不是设计文档。

<a id="refresh-2026-10-06-runtime"></a>

### 运行时与协调

<a id="source-claude-stop-scopes"></a>

#### Claude Code：停止一个回合与停止后台工作

**9 月 17 日至 10 月 5 日，v2.1.275 至 v2.1.290。** [CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [v2.1.281](https://github.com/anthropics/claude-code/releases/tag/v2.1.281)

在 v2.1.275 中，send-now 键（ctrl+enter，在 Claude 仍在工作时发送排队消息）会打断当前回合；自 v2.1.281 起，它改为把正在运行的工具移到后台，而不取消该回合。在 VS Code 扩展中，Stop 和 Escape 只结束当前回合；后台智能体继续运行，可在 agent 地图中逐个停止（v2.1.286）。v2.1.285 为后台 shell 命令加入时限（默认 30 分钟，最长 2 小时）；自 v2.1.288 起，该时限只适用于无人值守的会话，例如 `-p`、Agent SDK、CI 和云端会话。自 v2.1.287 起，来自 `claude agents` 的回复以排队消息送达，除 `/stop` 外的斜杠命令在当前回合结束时才执行；不过 v2.1.290 让 `/model`、`/effort` 和 `/rename` 对忙碌的后台会话立即生效。

<a id="source-vscode-remote-agent-hosts"></a>

#### VS Code 1.140：委派给远端 agent host

**9 月 30 日，VS Code 1.140；实验性功能，默认关闭。** [发布说明](https://code.visualstudio.com/updates/v1_140)

新工具让智能体可以列出已连接的远端 agent host 及其容量和会话负载，在指定 host 上启动会话，或选择满足操作系统、内存和 CPU 要求的 host，查看会话状态并互发消息。远端会话除非指定，否则没有工作区；这些工具不会复制发起方的工作区，常规审批仍然适用。远端智能体通过 `send_remote_message` 回报，其最终答复不会自动转发，协调窗口必须保持打开，消息才能流转。同一版本提高了编排上限；达到上限时会阻止新的编排动作，但不会打断已经在运行的工作。

<a id="source-copilot-dynamic-workflows"></a>

#### Copilot dynamic workflows：上限、暂停与恢复

**10 月 1 日，公开预览。** [Changelog](https://github.blog/changelog/2026-10-01-dynamic-workflows-in-copilot-cli-and-the-copilot-app) · [概念文档](https://docs.github.com/en/copilot/concepts/agents/dynamic-workflows)

在 Copilot CLI、Copilot app 和 Copilot SDK 中，workflow 是一段代码，规定步骤、何时调用智能体以及如何使用其结果。达到并发上限时，新智能体会等待；智能体总数、活跃运行时间和近似 AI credits 这几项上限会停止运行，但保留状态和已保存结果，提高后的上限仍计入停止前的用量。恢复时可复用已完成步骤和子智能体的已保存结果，未保存的工作可能需要重跑。workflow 的子智能体继承会话的权限授予，因此为某个子智能体作出的会话期授权也适用于其他子智能体。

<a id="source-exactly-once-tool-contract"></a>

#### 恰好一次：哪一层防止重复写入

**9 月 24 日，arXiv v1。** [论文](https://arxiv.org/abs/2609.29095v1)

这项研究（*Where Does Exactly-Once Live?*）在工具边界注入故障，例如在智能体停止等待之后才提交的写入，或被投递两次的请求，并依据已提交效果的账本为每个 episode 评分，其中包括在 GitHub Copilot CLI、Hermes 和 Codex CLI 下的运行。作者报告：被要求恰好执行一次的前沿模型，在确认丢失时几乎从不重复写入，但对仍在途或被投递两次的请求经常重复执行；为每次写入提供幂等键后，重复率从 28% 降到 4%，三个 harness 的表现几乎相同。在产生重复的 episode 中，有 90% 的智能体报告任务已完成。这些结果来自一位作者在合成基准上的报告，论文称代码和数据将在发表时公开。

<a id="source-planarian-statepoints"></a>

#### Planarian：本地与远端状态共用一个恢复点

**9 月 28 日，arXiv v1。** [论文](https://arxiv.org/abs/2609.35366v1) · [StateFork](https://arxiv.org/abs/2609.38648v1)

Planarian 是一个研究原型，其 statepoint 同时覆盖沙箱中的文件和进程，以及通过 MCP 对远端作出的更改。它在本地做增量检查点，并为每个远端调用记录补偿动作；回滚按相反顺序执行这些补偿动作，再恢复本地快照，分叉则创建相互隔离的分支。无法改写为可补偿形式的远端请求会在执行前被拒绝，补偿只为数据库 MCP 服务器上的 SQL 操作实现。StateFork（arXiv，9 月 29 日）研究如何分支和恢复终端会话，让智能体可以尝试不同方案。

<a id="source-codex-queued-reconnect"></a>

#### Codex CLI：先确定结果不明的提交，再重发

**9 月 29 日和 10 月 1 日，Codex CLI 0.159.0 与 0.160.0。** [0.160.0 发布说明](https://github.com/openai/codex/releases/tag/rust-v0.160.0) · [0.159.0 发布说明](https://github.com/openai/codex/releases/tag/rust-v0.159.0)

重连后，Codex CLI 0.160.0 只有在确定那些结果不明的提交之后，才恢复发送未发出的排队消息，以免重复发送。0.159.0 新增可选的 `instant_interrupt` 设置，允许新输入在模型响应期间或长时间运行的 code-mode 调用期间引导 Codex。

<a id="refresh-2026-10-06-harness"></a>

### Harness 与应用接口

<a id="source-agents-api-computer-use"></a>

#### Agents API 计算机使用：来源批准与登录

**9 月 29 日，Agents API 公开测试版。** [API changelog](https://developers.openai.com/api/docs/changelog) · [计算机使用指南](https://developers.openai.com/api/docs/guides/agents-api/tools/computer-use)

OpenAI 为 Agents API 加入计算机使用，浏览器运行在 OpenAI 托管的环境中。每个新的网站来源都需要用户批准，这与环境的网络策略相互独立；来源批准也不代表对购买等单个操作的确认。登录信息通过专用事件提交，不进入模型输入和会话历史；只有主智能体能请求登录，子智能体不能。断线后，应用应重新读取会话的待处理动作，而不是重新发送任务或批准。

<a id="source-cursor-token-efficiency"></a>

#### Cursor：去掉模型已不需要的 harness 工作

**9 月 23 日，工程文章。** [文章](https://cursor.com/blog/improved-token-efficiency)

Cursor 报告称，harness 改动使用户 token 成本降低 7%，且未降低智能体质量；文章描述了在生产流量上对单项改动做 A/B 测试。随着模型变强，它删去约 66% 的系统提示，按需加载不常用的内置工具，在每个请求中很少变化的部分之后设置缓存断点，并删除推动使用子智能体的指令，因为较新的模型已在训练中学会这种做法。这些数字由厂商报告，文章未公布评测细节。

<a id="source-harness-design-components"></a>

#### Harness 组件：价值取决于模型与上下文预算

**9 月 17 日，arXiv v1；相关研究为 9 月 30 日。** [论文](https://arxiv.org/abs/2609.20804v1) · [机器学习工程研究](https://arxiv.org/abs/2609.40303v1)

该研究固定编码 harness 的执行循环，在四个开放模型、四种上下文预算和 176 组设置上，于 SWE-Bench Verified 和 Terminal-Bench 2.1 中变动规划、动作空间和上下文管理。上下文预算越紧，上下文管理越重要，主要作用是防止溢出；先删减陈旧工具输出、再做 LLM 摘要的策略，成功率与其他管理策略相近，并在八个模型与基准组合中的七个里成本最低，而召回被删减内容的工具很少被使用。规划提高了较弱模型的准确率，对较强模型则主要降低成本；每个设置只运行一次，也未测试闭源前沿模型。9 月 30 日一项针对机器学习工程任务的研究发现，在同一强骨干模型下，四个开源 harness 均未优于单个最小编码智能体会话，而较弱的骨干模型仍受益于工作流先验。

<a id="source-zcode-shared-runtime"></a>

#### ZCode：三种界面构建在同一仓库的智能体运行时之上

**2026 年 9 月，源码发布；最早公开提交为 9 月 20 日，README 记载 9 月 23 日更新到 v3.14.3。** [仓库](https://github.com/zai-org/ZCode) · [CLI 插件文档](https://github.com/zai-org/ZCode/blob/main/apps/zcode-cli/README.md)

Z.ai 开源的编程工作台提供桌面、浏览器和终端界面，构建在同一仓库中的智能体 CLI 与运行时之上。Web 界面默认只监听本机地址，绑定其他地址时会生成访问令牌。插件是本地包，可添加 skills、自定义命令和 MCP 服务器。README 未说明权限或沙箱模型；许可证为 Apache-2.0。

<a id="source-opus-5-5-instruction-cleanup"></a>

#### Opus 5.5 指南：通过清理指令完成迁移

**9 月 22 日，官方文章。** [使用指南](https://claude.dev/blog/getting-the-most-out-of-opus-5-5/)

Anthropic 的 Opus 5.5 指南建议删除“仔细思考”之类的指令，在 CLAUDE.md 中写明何时继续、何时停下询问，并把长任务的清单保存在文件中，因为压缩会总结较早的回合。被安全机制标记的消息，多数会由 Claude Code 转到较旧的模型上继续会话，除非用户在 `/config` 中改为先询问。这是使用指南，不是工程评测。

<a id="refresh-2026-10-06-context"></a>

### 上下文与记忆

<a id="source-claude-agents-md-fallback"></a>

#### Claude Code：AGENTS.md 作为备用指令文件

**9 月 18 日，v2.1.277；9 月 23 日（v2.1.281）扩展，10 月 5 日（v2.1.290）修复。** [v2.1.277 发布说明](https://github.com/anthropics/claude-code/releases/tag/v2.1.277) · [Memory 文档](https://code.claude.com/docs/en/memory)

工作目录及其上级目录没有 CLAUDE.md 时，Claude Code 将 AGENTS.md 作为项目指令读取；在这一判断中，CLAUDE.local.md 也算作 CLAUDE.md。“Project instructions” 设置提供四种模式：优先读 CLAUDE.md，没有时读 AGENTS.md（默认）；两者同时读取；只读 CLAUDE.md；或只读托管指令。文档列出了与 CLAUDE.md 的差异：不触发 InstructionsLoaded hook；外部 @import 只有在此前已获批准时才加载。2.1.281 将支持扩展到 Bedrock、Vertex、Foundry、网关以及关闭遥测的会话，2.1.290 则在 @ 提及某子目录下的文件时附加该子目录的 AGENTS.md。

<a id="source-codex-instruction-refresh"></a>

#### Codex：修改后的指令何时生效

**9 月 22 日，Codex rust-v0.156.0。** [发布说明](https://github.com/openai/codex/releases/tag/rust-v0.156.0) · [PR #44675](https://github.com/openai/codex/pull/44675) · [PR #44701](https://github.com/openai/codex/pull/44701) · [PR #46577](https://github.com/openai/codex/pull/46577)

Codex 0.156.0 在每个模型请求边界（包括同一回合内的工具调用之后）重新加载全局指令，因此对全局 AGENTS.md 的修改在会话运行中即可生效；仓库指令只在环境选择或信任级别变化时重新发现。宿主可以加入线程级指令，上限约为 10,000 个估算 token，超出时直接拒绝而不是截断。新子智能体继承父线程已应用的指令快照；只有提供方明确选择共享，后续更新才会传到正在运行的子智能体。线程指令提供方是宿主接口，不是 CLI 用户的配置项。

<a id="source-claude-auto-memory-guards"></a>

#### Claude Code：召回的记忆按不可信输入处理

**9 月 28 日和 29 日，v2.1.284 与 v2.1.285。** [v2.1.284 发布说明](https://github.com/anthropics/claude-code/releases/tag/v2.1.284) · [v2.1.285 发布说明](https://github.com/anthropics/claude-code/releases/tag/v2.1.285)

Claude Code 2.1.284 在 MEMORY.md 和被召回的记忆便笺送入模型之前，中和其中的不可见字符和模仿 Claude Code 自身标记的标签。2.1.285 规定后台会话，或由 Claude Code 自身工具启动的会话，不能开启自动记忆；在这些会话中仍可关闭。记忆文档说明，需要在直接从终端启动的会话中开启。中和这些标记本身并不能阻止所有形式的记忆投毒。

<a id="source-copilot-memory-autofix"></a>

#### Copilot Memory：一个功能写入，其他功能使用

**9 月 25 日，公开预览。** [Changelog](https://github.blog/changelog/2026-09-25-agentic-autofix-now-uses-copilot-memory) · [Copilot Memory 文档](https://docs.github.com/en/copilot/concepts/agents/copilot-memory)

agentic autofix 在修复安全告警时读取 Copilot Memory，并把每个修复模式写为记忆，供 code review、cloud agent 等其他 Copilot 功能使用。现行文档说明，仓库级事实附有指向支撑代码的引用，使用前会对照当前分支校验。只有具有写权限的用户能产生这类事实，它们只用于该仓库；未被使用的条目在 28 天后删除。引用校验和保留规则是 Copilot Memory 已有文档中的行为，并非此次发布新增。

<a id="source-vibemembench"></a>

#### VibeMemBench：记忆系统在仓库任务上的表现

**9 月 20 日，arXiv v1。** [论文](https://arxiv.org/abs/2609.23570v1)

VibeMemBench 用可执行检查在 111 个仓库编码目标上测试记忆系统。注入已验证有用的经验，使 5 个留出求解器中的 4 个解决率提高 1.1 至 4.5 个百分点；但当 4 个现有记忆系统从同一历史中自行构建并检索经验时，12 个求解器与系统组合中有 11 个没有超过匹配的无记忆基线。作者把主要问题归于记录的提供形式。目标是按“注入有效”筛选的，检索在每次运行开始前一次完成，也未评估 Claude 和 GPT 模型。

<a id="refresh-2026-10-06-authority"></a>

### 工具与权限

<a id="source-claude-code-mods"></a>

#### Claude Code mods：能批准调用的进程内扩展

**10 月 1 日，首次列于 v2.1.287 CHANGELOG。** [文章](https://claude.com/resources/articles/claude-code-mods) · [Mods 概览](https://code.claude.com/docs/en/plugins/mods/overview) · [组织控制](https://code.claude.com/docs/en/plugins/mods/admin) · [v2.1.287 发布说明](https://github.com/anthropics/claude-code/releases/tag/v2.1.287)

mods 是插件中的函数，在 Claude Code 进程内运行，可以观察、改写或直接应答工具调用、提示和界面事件；它们不在沙箱内运行，在未信任的目录中，用户回答信任提示之前不会加载任何 mod。mod 可以在权限提示出现前批准工具调用，但不能改变提示显示的内容。在内置 `sec-default` 守卫加载的环境中，deny 规则（除非管理员设置了 `allowModsToOverrideDenyRules`）和托管 `PreToolUse` hook 仍然优先，守卫读不到托管设置时会拒绝加载用户的 mod；但用户的 mod 仍可批准 `ask` 规则本应提示的调用，在 auto mode 下这类调用不经过分类器。deny 规则不覆盖 mod 自身的文件和进程调用；v2.1.290 为 `tool.check` 增加 `ceiling` 字段，表示组织要求的批准级别。

<a id="source-claude-managed-policy-precedence"></a>

#### Claude Code：仓库设置不能放宽托管策略

**9 月 24 日至 10 月 5 日，v2.1.282 至 v2.1.290；公告为 9 月 29 日。** [CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) · [GHSA-gfvf-j8jh-jxxw](https://github.com/anthropics/claude-code/security/advisories/GHSA-gfvf-j8jh-jxxw)

项目设置不能再放宽或关闭管理员要求的沙箱，也不能扩展严格允许列表（v2.1.285）；在 `allowManagedPermissionRulesOnly` 下，仓库、用户和 `--add-dir` 中的 skill 与命令不能再预先批准自己的工具（v2.1.282），插件只有来自官方或管理员认可的来源才保留这种预先批准（v2.1.284）。一个嵌套值无效时，不再导致整个托管 `permissions`、`autoMode`、`worktree`、`attribution`（v2.1.282）或 `sandbox`（v2.1.283）设置块被忽略；对 `sandbox` 而言，该无效值按拒绝处理；类型写错的布尔锁定键现在也会生效（v2.1.282）。发布说明写明了按拒绝处理的一个例外：如果操作系统拒绝读取托管设置文件，v2.1.285 会发出警告，并在没有该文件策略的情况下启动；其他读取错误和无法解析的文件仍会阻止所有会话。9 月 29 日的公告 CVE-2026-103012（2.1.260 修复）说明，本地存储的 API key 可能使会话在没有组织服务端下发策略的情况下运行；MDM 和文件形式的托管设置不受影响。

<a id="source-claude-auto-mode-default"></a>

#### Claude Code：默认 auto mode，沙箱命令仍需审查

**9 月 23 日至 29 日，v2.1.281、v2.1.284 与 v2.1.285。** [v2.1.284 发布说明](https://github.com/anthropics/claude-code/releases/tag/v2.1.284) · [CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

从 v2.1.284 起，未配置权限模式的交互式会话在所有套餐和提供商上都以 auto mode 启动；`permissions.defaultMode` 仍可覆盖。v2.1.285 把这一默认扩展到第三方提供商或关闭遥测时的 `claude -p` 与 Python Agent SDK 会话。在 auto mode 分类器于服务端运行的情况下，v2.1.281 让只读和已沙箱化的 shell 命令也等待分类器审查，因此命令处于沙箱中不再意味着它可以跳过分类器。

<a id="source-copilot-local-sandbox-policy"></a>

#### Copilot：本地沙箱、默认启用策略与按应用批准

**9 月 23 日、9 月 24 日和 10 月 1 日；沙箱与计算机使用为公开预览。** [本地沙箱](https://github.blog/changelog/2026-09-23-local-sandboxing-in-the-github-copilot-app/) · [默认启用](https://github.blog/changelog/2026-09-24-default-enablement-of-copilot-features-for-copilot-business-and-enterprise/) · [计算机使用](https://github.blog/changelog/2026-10-01-github-copilot-can-now-interact-with-desktop-apps/)

Copilot app 的本地沙箱按项目配置文件、网络以及 Git 或 GitHub CLI 凭据；企业设置可以使实际策略更严格，如果操作系统无法执行所请求的策略，沙箱 shell 会报错，而不是在无沙箱的情况下运行。它默认关闭，与 Copilot CLI 分开配置，不适用于云沙箱或远程主机会话。9 月 24 日发布的 Business 和 Enterprise 策略让管理员决定：未配置的合格 GA 功能（包括现有和未来的功能，以及 MCP 服务器策略）默认开启、默认关闭还是交给组织决定；该策略于 10 月 22 日生效，已作出的选择保持不变。在 macOS 和 Windows 上，Copilot CLI 与 Copilot app 的计算机使用会在控制每个应用前请求批准，并保留可查看的“始终允许”列表。

<a id="source-codex-network-revocation"></a>

#### Codex CLI：在连接存续期间执行网络规则

**9 月 25 日，Codex CLI 0.157.0。** [发布说明](https://github.com/openai/codex/releases/tag/rust-v0.157.0)

Codex CLI 0.157.0 把网络限制应用于重定向以及持续进行的 HTTP 和 WebSocket 流量；策略变更撤销访问权限时，会取消已有连接。因此网络授权在连接过程中持续检查，而不只在建立连接时检查。

<a id="source-anthropic-checked-vs-effective"></a>

#### Anthropic 公告：检查的对象与实际生效的对象

**9 月 25 日和 10 月 5 日，安全公告。** [GHSA-v234-4jrq-mgg6](https://github.com/anthropics/claude-code/security/advisories/GHSA-v234-4jrq-mgg6) · [GHSA-5j29-h97v-84ch](https://github.com/anthropics/claude-code/security/advisories/GHSA-5j29-h97v-84ch)

Claude Desktop 禁止从 Cowork 共享文件夹直接打开某些“打开即执行”的文件类型，但 macOS 上的列表遗漏了一种这样的类型；沙箱内被攻陷或被提示注入的智能体写入的文件，在用户打开时可能在主机上执行命令（受影响版本自 1.1.3918 起，1.15962.0 修复）。CVE-2026-103435 说明，Claude Code 在权限检查时验证写入路径位于项目内，但在写入时重新解析路径；能写入共享工作区并赢得竞态的攻击者可以换入符号链接，把写入引到项目之外。该问题已在 2.1.129 修复，远早于 10 月 5 日的披露。沙箱与主机之间的共享文件夹是一条返回通道，需要在主机侧另设控制。

<a id="source-gitspawn-background-git"></a>

#### GitSpawn：信任之前的后台 git 命令

**9 月 1 日，研究披露。** [披露文章](https://www.manifold.security/blog/ai-coding-agents-git-hijack) · [goose 公告](https://github.com/aaif-goose/goose/security/advisories/GHSA-r5pp-p5r8-466r)

Manifold Security 报告，多个 CLI 编码智能体在启动时或会话开始后不久运行 `git status` 或 `git diff` 收集上下文，但没有去除仓库自带的 git 配置，因此 `core.fsmonitor` 等设置会以用户身份运行仓库指定的命令；该命令在沙箱之外运行，也没有批准提示。在 Claude Code 2.1.193 中，这发生在工作区信任提示被接受之前；2.1.196 已修复。另一项 Claude Code 问题出现在 `claude ultrareview` 路径上，利用的是另一个 git 设置，到 9 月 1 日文章发布时在 2.1.252 上仍未修复。攻击需要仓库以带 `.git` 目录的文件形式送达，例如压缩包或同步文件夹，普通 clone 不会触发。作者报告了 7 个智能体中的 8 项问题，发布时其中 4 项尚未修复；goose 在 1.44.0 修复了其问题（CVE-2026-72718）。

<a id="source-approval-scope-lifetime"></a>

#### 批准洗白：批准覆盖什么、持续多久

**9 月 23 日、27 日和 30 日，arXiv v1。** [Agent Approval Laundering](https://arxiv.org/abs/2609.28586v1) · [When Consent Outlives Context: Residual Authority Replay in Long-Lived Agents](https://arxiv.org/abs/2609.33910v1) · [Approval Laundering](https://arxiv.org/abs/2609.38983v1)

Agent Approval Laundering（9 月 23 日）指出，批准记录只写入口命令，而它启动的工作流（例如包的生命周期 hook）可能产生其他效果；作者提出在用户批准前，把对该工作流可能产生的效果的预测写入批准记录。When Consent Outlives Context: Residual Authority Replay in Long-Lived Agents（9 月 27 日）报告，在 AgentDojo 用例上，跨任务保留的批准最多使提示注入成功率提高 35.1 个百分点。Approval Laundering（9 月 30 日，单一作者）归纳了实际执行的动作与已批准动作不一致的六种方式；在作者的测试中，带密钥的批准令牌消除了委派类不一致，但没有解决范围类和参数类不一致。其中两篇论文基于 Claude Code 的 PreToolUse hook：一篇用它测量批准不一致，另一篇用它传递批准记录。

<a id="refresh-2026-10-06-evaluation"></a>

### 评测与演化

<a id="source-harness-buy-rerun-noise"></a>

#### What Does a Harness Buy?：与重跑噪声比较

**10 月 3 日，arXiv v1。** [论文](https://arxiv.org/abs/2610.04433v1)

该研究固定模型，在 SWE-bench Verified 上比较 Claude Code、mini-SWE-agent 和 OpenCode，并以相同配置的重跑作为噪声基准。在最难的 45 个任务上，更换 harness 改变的任务结果与重跑同一 harness 一样多。能测出的 harness 效应都是丢失任务的方式，例如输出达到上限后不恢复；单任务成本最多相差三倍，主要来自每一步都重新发送的系统提示和工具 schema。结果只覆盖一个基准，Claude 模型只在 Claude Code 中运行。

<a id="source-frozen-judges"></a>

#### Frozen Judges：裁判误差随智能体版本变化

**9 月 28 日，arXiv v1；9 月 29 日 v2。** [论文](https://arxiv.org/abs/2609.34198v2)

固定的 LLM 裁判在比较智能体新旧版本时，可能出现随版本变化的误差。在 SWE-bench Verified 上，有几组版本对，仅依据裁判分数得到的置信区间显示新版本更好，但实际执行测试无法证实这一点，尽管裁判排名与参考结果相关性较高；较强智能体的失败补丁也更容易被判为通过。作者建议用裁判筛选需要比较的版本，发布决策则基于对当前输出随机抽样并加标注的审计。独立的人工补丁复核尚未完成。

<a id="source-self-healing-harness"></a>

#### Self-Healing Harness：只在既有成功保持时保留规则

**9 月 21 日，arXiv v1。** [论文](https://arxiv.org/abs/2609.24130v1)

智能体编写候选规则，由外部运行时决定哪些规则可以保留：只有在修复触发它的失败、且不使此前成功的受保护案例退化时，规则才会保留；另有一个守卫重新测试累积的规则集合。在因回放而被否决的 383 个提议中，211 个修复了触发失败，却破坏了一个受保护案例。研究没有与“不经门控直接接受同一批规则”的设置比较，且每轮最多回放两个受保护案例，因此 211 是检测到的冲突，不是总发生率。

<a id="source-overclaiming-transcripts"></a>

#### 过度声称：报告了执行记录中没有的工作

**9 月 17 日，arXiv v1；9 月 22 日 v3。** [论文](https://arxiv.org/abs/2609.20812v3)

该研究把“过度声称”定义为：最终报告声称完成了某项工作，但智能体自己的执行记录显示并未完成，例如声称读过一个从未打开的文件。文件覆盖率由执行记录测得，报告类别由 LLM 裁判判定。在生产 CLI 中运行的五个审查场景里，约三分之二的运行没有读完要求的文件，其中多数不完整运行要么声称已完整审查，要么没有说明遗漏；要求使用子智能体提高了覆盖率，但没有提高如实报告的比例。场景是针对 Claude Opus 调整设计的。

<a id="source-terminal-bench-hardness"></a>

#### Terminal-Bench 难度：零通过率需要审计

**9 月 20 日，arXiv v1。** [论文](https://arxiv.org/abs/2609.26826v1)

论文审计冻结的 Terminal-Bench 3 生产记录中没有任何智能体通过的任务，按顺序检查参考解是否通过、基础设施失败是否占主导、是否存在绕过验证器的方法，以及可解性是否有证据支持。125 个全失败任务中，78 个仍可作为真正未解的候选；其余任务存在参考解损坏、基础设施问题、只能靠绕过通过，或可解性未得到证明。这 78 个中有 53 个只有一次参考解运行，因此该标签范围很窄，不能证明任务本身的难度。

<a id="source-deltaselect-ab"></a>

#### DeltaSelect：用于 A/B 比较的小型固定任务集

**9 月 17 日，arXiv v1。** [论文](https://arxiv.org/abs/2609.19607v1)

DeltaSelect 为开发过程中反复进行的“基线对候选”比较选择一个小型固定任务集，而不是用于模型排名。任务按单次运行结果追踪全基准结果的可靠程度排序；任务集、校准和价格在第一次比较前冻结，基线必须在被修改的那个 harness 及其确切版本中重新运行。作者指出，尚未在同等成本下与随机选题比较，并且在小任务集上反复调优可能对其过拟合。

<a id="source-claude-build-eval-hillclimb"></a>

#### Claude API skill：先构建评测，再逐个保留或回滚补丁

**9 月 28 日，官方文章。** [文章](https://claude.dev/blog/automating-eval-design-and-hillclimbing/)

`/claude-api build-eval` 在代码库中构建评测，并检查评分器一致性以及超时、截断等基础设施故障。`/claude-api hillclimb` 把样例分为训练集和保留测试集，并先检查评测噪声是否小于值得采纳的最小改进。每轮只提出一个补丁：如果训练集分数上升而保留集持平，或任一方出现回退，就回滚该补丁；分数连续两三轮停滞时，循环按原因整理剩余的训练集失败，只把合理的失败留到后续各轮；最终增益落在噪声范围内时，建议不合并。文中的例子来自 Anthropic 自身，尚无独立复现。

<a id="source-skill-revision-study"></a>

#### Agent Skill Evolution：SKILL.md 修订改变了什么

**10 月 4 日，arXiv v1。** [论文](https://arxiv.org/abs/2610.04832v1)

该研究比较 2,608 个 SKILL.md 文件的首个与最终版本。作者报告，加入可自动检查的规则后，四个智能体执行要求动作的 episode 比例平均提高 0.23（以 0 到 1 计）；按需加载技能正文时，约保留一半的提升。

<a id="source-langsmith-engine-fix-validation"></a>

#### LangSmith Engine v2：审查前先复现失败

**9 月 24 日，厂商文章；修复验证处于 private beta。** [文章](https://www.langchain.com/blog/langsmith-engine-v2-redteam)

Engine 先在 LangSmith Deployment 中复现失败，再用同一批输入测试修复，然后才交给人工审核。文章只描述了在出错输入上的检查，没有描述对此前成功案例的检查。

<a id="refresh-2026-09-15"></a>

## 智能体设计来源：2026 年 9 月 15 日

除另有说明外，日期均为 2026 年；release 时间使用 UTC。arXiv 论文为预印本，实验结果来自论文作者的报告。Claude Code 架构分析对应 v2.1.88；下文中的更新版本说明后续变化。

<a id="refresh-2026-09-15-graph"></a>

### 工作图与控制循环

<a id="source-mastra-factory-stage-rules"></a>

#### Mastra Factory：阶段规则与审批对象

**9 月 8 日，beta 公告。** [Beta 公告](https://mastra.ai/blog/announcing-mastra-factory-beta) · [看板规则](https://factory.mastra.ai/configure/boards-and-rules) · [审批文档](https://factory.mastra.ai/using/work-and-approvals)

Mastra Factory 的当前文档分别定义允许的阶段转移、进入和离开阶段时执行的动作，以及转移所需的审批策略。PR 审查和原任务完成也分别记录。这使转移规则与任务接受条件成为可以分别设计的事项。

<a id="source-trace2flow-review"></a>

#### Trace2Flow：通过图检查已完成的工作

**9 月 11 日，arXiv v1。** [论文](https://arxiv.org/html/2609.13136v1)

Trace2Flow 把已完成的智能体运行转成可编辑的工作流图。用户可以检查每步记录的输入输出、修改依赖，并为相关任务重新运行步骤。

<a id="source-ready-turn-release"></a>

#### 就绪轮次：决定何时提交执行

**9 月 10 日，arXiv v1。** [论文](https://arxiv.org/html/2609.10964v1)

模型的一轮调用满足前置条件后，可以先等待，再提交给共享推理引擎。该调度器选择下一轮提交什么，并限制已提交但尚未完成的工作量。论文用软件工程 agent 轨迹进行回放，测量资源竞争下整个工作流的尾延迟。其模型假定已提交轮次不能撤回。

<a id="source-jira-agent-loops"></a>

#### Jira Agent loops：由积压任务驱动外层循环

**9 月 10 日，私有早期访问公告。** [公告](https://www.atlassian.com/blog/jira/governed-agent-loops)

Jira Agent loops持续寻找描述明确且未分配的任务，委派实现和测试，并产出待审 PR；合并决定仍由开发者作出。任务选择来自共享积压列表，实现工作产出可审查的变更。

<a id="source-trove-route-editing"></a>

#### TROVE：修改剩余路线

**9 月 4 日，arXiv v1。** [论文](https://arxiv.org/html/2609.05019v1)

TROVE 从拟定路线中执行一项 skill，再根据结果保留下一步、插入局部处理，或替换尚未执行的后续路线；已完成的工作及其产物继续保留。这明确了路线修改操作，同时保留已完成的工作。

<a id="refresh-2026-09-15-runtime"></a>

### 运行时与协调

<a id="source-cursor-projects"></a>

#### Cursor Projects：跨智能体和机器延续项目

**9 月 10 日，开始推出 beta。** [公告](https://cursor.com/changelog/projects)

云端协调器规划工作、委派实现，再将结果交给用户审查。项目在其云端与本地agent之间同步一组文件。事件订阅可触发后续工作。项目可以跨单次会话和机器延续。

<a id="source-vscode-message-queue"></a>

#### VS Code：在轮次边界处理排队消息

**9 月 9 日，VS Code 1.137。** [发布说明](https://code.visualstudio.com/updates/v1_137) · [会话编排文档](https://code.visualstudio.com/docs/agents/run/sessions/manage-sessions)

VS Code 1.137 将智能体发给忙碌 chat 的消息排队，当前回合成功结束后，再按发送顺序开始处理排队消息。通过 Agent Host 会话工具跨会话发送仍需用户确认。送达顺序和处理时机都是协调契约的一部分。

<a id="source-vscode-automation-lifetimes"></a>

#### VS Code Automations：计划与运行有不同的生命周期

**VS Code 1.137，9 月 9 日；指南页尾日期为 9 月 8 日。** [自动化指南](https://code.visualstudio.com/docs/agents/run/automations)

VS Code Automations处于Preview。计划需要未休眠的机器，以及相应的Agent Host进程或VS Code窗口。每项自动化一次运行一个会话。禁用计划不会停止正在进行的运行。保存的计划、当前工作及所需宿主有不同的生命周期。

<a id="source-fusion-model-pairs"></a>

#### Fusion：按模型组合调整协作

**9 月 11 日，Desktop 和 CLI 公告。** [Desktop 和 CLI 公告](https://cognition.com/blog/local-fusion)

lead和sidekick保留各自上下文，交换任务简报、结果与反馈。Cognition按模型组合调整简报细度、worker能否质疑指令，以及探索分工。

<a id="refresh-2026-09-15-harness"></a>

### Harness 与应用接口

<a id="source-agents-api-boundaries"></a>

#### Agents API：区分 harness 与执行环境

**9 月 10 日，公开测试版。** [公告](https://openai.com/index/introducing-the-agents-api/) · [概览](https://developers.openai.com/api/docs/guides/agents-api/overview)

OpenAI 的 Agents API 在公开测试版中将 Codex harness 作为托管服务提供。应用提供工具并选择执行环境，OpenAI 则运行智能体循环并管理会话、上下文和恢复。这为应用提供了围绕完整智能体循环的接口。

<a id="source-astra-skill-selection"></a>

#### GPT-6 Astra：技能选择与完成条件

**9 月 11 日，官方开发者文章。** [开发者指南](https://developers.openai.com/blog/rethinking-skills-and-prompts-for-gpt-6-astra)

针对 GPT-6 Astra，OpenAI 建议使用简短的技能描述、按任务指定的文档引用，以及明确的完成条件。技能过多时，Codex 会截短展示给模型的描述。旧指令还可能增加不必要的工作或让智能体过早停止。

<a id="source-claude-prompt-audit"></a>

#### Claude API skill：在模型迁移时检查提示

**9 月 8 日，官方文章。** [模型迁移工作流](https://claude.com/blog/reducing-cost-and-improving-performance-with-claude-platform)

Anthropic 介绍了 Claude API skill 中的 prompt-audit、hillclimb 和 cost-optimize 工作流。它们帮助检查旧提示规则并调整模型用法。在提供评测后，hillclimb 用训练样例指导修改，再用保留测试集检查最终配置。

<a id="source-claude-programmatic-continuation"></a>

#### Claude 编程式调用：暂停并继续程序

**2025 年 11 月 24 日进入 beta；2026 年 2 月 17 日起不再要求 beta header。** [协议文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling) · [发布历史](https://platform.claude.com/docs/en/release-notes/overview)

Claude 可以运行调用工具的 Python 程序，暂停以等待客户端结果，再在同一代码执行容器中继续。客户端回传待处理工具结果时须带上容器 ID；只有程序的最终输出进入模型上下文。

<a id="refresh-2026-09-15-context"></a>

### 上下文与记忆

<a id="source-codex-memory-v2"></a>

#### Codex memory v2：分版本存储与证据选择

**9 月 8 日主线提交 2cbbf0c9。** [主线修改](https://github.com/openai/codex/commit/2cbbf0c9b542a36a1c3284b5e804917635b6f666)

Codex 的9月8日主线代码增加独立的 memory v2 存储及可选双版本写入，由所选版本提供上下文。v2 在记忆提取的输入预算内，优先保留用户消息及其对智能体提问的回答，再形成任务摘要。其提示要求将特定任务的纠正保留在该任务范围内，并在整合时处理明确的纠正或删除便笺。自[稳定版 0.155.0](https://github.com/openai/codex/releases/tag/rust-v0.155.0)（9 月 17 日）起，[记忆配置](https://github.com/openai/codex/blob/rust-v0.155.0/codex-rs/config/src/types.rs)接受 `version` 和 `dual_write`；到 0.160.0 默认仍为 v1；截至 2026 年 10 月 6 日，配置参考未列出这两个键。

<a id="source-claude-tag-recall-scope"></a>

#### Claude Tag：频道便笺与工作区便笺

**9 月 10 日，Claude Code v2.1.268。** [发布说明](https://github.com/anthropics/claude-code/releases/tag/v2.1.268) · [帮助页面](https://support.claude.com/en/articles/15594475-what-is-claude-tag)

Claude Code 2.1.268发布说明报告了Claude Tag记忆范围的收窄。每个公共频道保留自己的便笺，Claude不再召回其他公共频道的便笺。工作区便笺仍然共享。

<a id="source-claude-stable-prompts-current-state"></a>

#### Claude Code：稳定请求与当前事实

**9 月 9–11 日，v2.1.267–v2.1.269。** [v2.1.267 发布说明](https://github.com/anthropics/claude-code/releases/tag/v2.1.267)

Claude Code 2.1.267 默认为子智能体和自定义提示会话一次性记录系统提示及工具定义。[2.1.268](https://github.com/anthropics/claude-code/releases/tag/v2.1.268) 将稳定工具表扩展到Bedrock、Vertex和Foundry，晚到工具通过延迟定义加入。[2.1.269](https://github.com/anthropics/claude-code/releases/tag/v2.1.269) 在压缩后向 Claude 提供当前 Git 状态。这些变化说明，稳定的请求内容与当前环境事实需要分开处理。

<a id="source-hermes-memory-review-authority"></a>

#### Hermes：记忆替换与删除的审批

**9 月 11 日，v0.21.2 / v2026.9.11，提交 939e45c9。** [发布说明](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.9.11) · [记忆操作门禁](https://github.com/NousResearch/hermes-agent/blob/939e45c91d751fadd94dcd1b873ac3cb44846213/tools/memory_tool.py#L129) · [提案存储](https://github.com/NousResearch/hermes-agent/blob/939e45c91d751fadd94dcd1b873ac3cb44846213/tools/write_approval.py#L73)

Hermes Agent v0.21.2按复盘目的限制后台复盘的默认记忆工具访问。在无人值守的后台复盘中，内置记忆工具将replace或remove操作（包括含有这些操作的整批请求）转为待批准提案，阻止该次调用直接执行它们。明确发起的/refine复盘标为有人参与。提案存储是尽力而为。

<a id="refresh-2026-09-15-authority"></a>

### 工具与权限

<a id="source-claude-command-authority"></a>

#### Claude Code：命令授权范围与不可读策略

**9 月 9 日和 14 日，v2.1.267 与 v2.1.271。** [v2.1.271 发布说明](https://github.com/anthropics/claude-code/releases/tag/v2.1.271) · [v2.1.267](https://github.com/anthropics/claude-code/releases/tag/v2.1.267)

Claude Code v2.1.271 为启用沙箱的 auto mode 中的 Bash、PowerShell 和 Monitor 增加按命令生效的 `allowed_domains`。企业 `managed-mcp.json` 无法读取或解析时，该版仍保留对 MCP 的独占控制。v2.1.267 则使三项无法读取的托管 hook/channel 允许列表不放行任何项。这些变化明确了命令授权范围与配置失败时的行为。

<a id="source-claude-plugin-consent"></a>

#### Claude Code：市场命令的同意机制

**9 月 14 日，v2.1.271。** [插件命令参考](https://code.claude.com/docs/en/plugins-reference) · [v2.1.271](https://github.com/anthropics/claude-code/releases/tag/v2.1.271)

安装或更新插件可能需要用户批准市场声明的命令。命令显示但未执行时，`--json` 结果会包含该命令及其摘要值。用户查看后可传入 `--accept-command <sha256>`。同意绑定到该命令、插件和目录；输入变化会使其失效。该选项在用户自己的终端生效，在 Claude Code 会话内无效，也不会授权插件的后续动作。

<a id="source-copilot-managed-operation-rules"></a>

#### Copilot：组合托管规则并限定每次授权

**9 月 9 日，托管操作权限。** [托管权限](https://github.blog/changelog/2026-09-09-enterprise-managed-permissions-for-github-copilot-agent-operations/) · [权限参考](https://docs.github.com/en/enterprise-cloud@latest/copilot/reference/enterprise-administrators/enterprise-managed-settings) · [JetBrains 预览公告](https://github.blog/changelog/2026-09-08-enterprise-managed-sandbox-in-copilot-for-jetbrains/)

GitHub 9 月 9 日发布的托管操作权限在 Copilot Business 和 Enterprise 的 app、CLI，以及 VS Code Agent Host 会话中正式可用。规则采用 deny > ask > allow 优先级；多个来源声明的允许列表取交集。托管 `ask` 每次都需要新的批准，旧授权或审批捷径不能代替。9 月 8 日公布的 JetBrains 托管沙箱策略是另一项 public preview。

<a id="source-google-sandbox-gateway-boundaries"></a>

#### Google：沙箱命令与网络路由

**9 月 9 日，沙箱正式发布。** [发布说明](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes) · [Shell 指南](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/sandbox/shell-sandbox-quickstart) · [网关](https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/set-up-vpc-connectivity) · [沙箱](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/sandbox/configure-vpc-sc)

Google 于 9 月 9 日宣布 Computer Use 和 Shell 沙箱正式可用。Shell 命令运行在托管 Linux 容器中；每次调用启动新的 shell，模板未启用时不开放互联网访问。另一个独立条件是：Agent Gateway 的 VPC Service Controls 支持要求 `ALL_TRAFFIC`，以及 9 月 8 日之后创建、并使用 agent connectivity template 的网关。把流量送入客户 VPC 后，路由和目标访问策略仍由客户配置。

<a id="source-aws-upload-owner"></a>

#### AWS Security Agent：检查上传目标归属

**9 月 10 日，安全通告披露。** [安全公告](https://aws.amazon.com/security/security-bulletins/2026-105-aws/) · [插件通告](https://github.com/aws/agent-toolkit-for-aws/security/advisories/GHSA-2px6-hhjp-3g5x) · [MCP server 通告](https://github.com/awslabs/mcp/security/advisories/GHSA-3jxw-vj8m-8x77)

AWS 于 9 月 10 日披露 CVE-2026-87912 和 CVE-2026-87913。Security Agent 插件与 MCP server 可能把工作区归档上传到其他账户抢先注册的存储桶。受影响版本为插件 1.0.0 及以下、MCP server 0.1.0–0.1.5；修复版分别为 1.1.0 和 0.2.0：核对预期账户归属，归属不符时停止。软件升级不会收回已经被其他账户注册的桶名。

<a id="refresh-2026-09-15-evaluation"></a>

### 评测与演化

<a id="source-skilladam-edit-budget"></a>

#### SkillAdam：用更新历史控制下一轮修改

**9 月 8 日，arXiv v1。** [论文](https://arxiv.org/html/2609.08944v1) · [实现版本 72ef9ba](https://github.com/ruc-datalab/SkillAdam/blob/72ef9ba48bbadc059c8a2d099f1b7d531fc8288f/README.md)

保存问题与修订尝试的历史，再根据案例结果的差异设置下一轮修改预算。接受门槛检查受保护指标。实验使用固定模型，不加额外编排层；接受判断复用采样案例，最终测试另行保留。

<a id="source-skill-issue-measurement"></a>

#### Skill Issue：评测能否识别有用的修改？

**9 月 11 日，arXiv v1。** [论文](https://arxiv.org/pdf/2609.12742v1)

研究通过撤回已合并 PR 的实现改动来构造任务，同时保留固定基线中的测试，再比较使用和不使用优化后 SKILL.md 的运行。在三个Kotlin仓库中，测试未能把观察到的收益与运行间波动区分开。该研究把任务构造和测量灵敏度纳入技能评估。

<a id="source-model-harness-correction"></a>

#### 模型与 harness 更新：纠正模型自身的失败

**9 月 8 日，arXiv v1。** [论文](https://arxiv.org/html/2609.09134v1)

在七项企业任务上测试harness演化后的微调。完整模仿专家轨迹出现回归。替代方法从较弱模型自身的运行出发，在训练前用专家纠正替换一个失败轮次。它支持共同评估模型与harness更新，但没有证明反复共同演化会持续稳定。

<a id="source-agent-report-evidence"></a>

#### 智能体报告：把声明关联到检查

**9 月 10 日，arXiv v1。** [论文](https://arxiv.org/html/2609.12205v1)

比较编码会话中的计划、工具调用记录和最终报告，并提出把结果声明关联到实际检查动作。声明判定模型未通过人工核验，报告重建测试也由模型执行。可用于设计的问题是：读者能否检查完成报告背后的证据。

<a id="source-agent-criteria-compendium"></a>

#### 智能体标准：为所需能力选择衡量方法

**9 月 10 日，arXiv v1。** [论文](https://arxiv.org/abs/2609.11018v1)

将智能体行为的五个维度——环境交互、学习与适应、自主性、目标导向行为和时间连贯性——对应到已有指标与基准。它区分对人工输入的独立程度与任务表现，帮助构建者在比较系统前明确要评估哪些能力。

<a id="source-swe-refactor-acceptance"></a>

#### SWE Refactor Bench：检查变更与保留行为

**8 月 24 日，arXiv v1。** [论文](https://arxiv.org/html/2608.23564v1) · [实现版本 0d5d731](https://github.com/Einsia/SWE-Refactor-Bench/blob/0d5d7310e64970826ffbe53fb53a349114acd69c/README.md)

区分迁移审计、固定行为测试和agent生成的反例。反例必须在原实现上通过、在提交上失败，并可重复复现。这同时检查要求的技术栈变更是否发生、行为是否保留。没有找到反例仍是有限证据，不是等价性证明。

<a id="september-2026"></a>

<a id="9-月整合截至-2026-09-07-的来源核对"></a>

## 截至 2026 年 9 月 7 日的来源

日期以来源的发布记录为准，GitHub release 使用 UTC 日期。版本发布日期用于标明版本，不代表其中每项机制都在当天首次出现。

下面的资料围绕组织工作、积累经验、约束执行和评估改进展开，内容来自公开文档、具体实现路径和作者报告的实验。

<a id="source-graph-engineering"></a>

### S1. Graph Engineering：组织任务、智能体与运行状态

[Graph Engineering in the Era of LLM Agents](https://arxiv.org/abs/2608.21156v2) 首次提交于 **8 月 21 日**；这里对应 **8 月 26 日的 v2**。[正文 §§2–5 和附录 11](https://arxiv.org/html/2608.21156v2)给出了定义与代表机制，包括 §4 的任务组织、智能体协调和运行状态管理。

**设计意义：**任务组织、智能体协调和运行状态是理解编排与持久化的三个相关视角。综述提供了一套有用的组织框架；“System Intelligence”及其范式递进是作者的主张。它没有证明图式协作始于 8 月，也没有证明这种设计在所有任务上都优于其他方案。

<a id="source-agent-graph"></a>

### S2. Agent Graph：让完成状态有事实依据

[Agent Graph v0.3.0](https://github.com/context4ai/agent-graph/releases/tag/v0.3.0) 发布于 **8 月 31 日**，源码固定到 [`387f80d`](https://github.com/context4ai/agent-graph/tree/387f80db65bf20a61bc666b4fa885200fcedad08)。来源包括 README、[设计文档](https://github.com/context4ai/agent-graph/blob/387f80db65bf20a61bc666b4fa885200fcedad08/docs/en/graph-engineering.md)核心章节，以及部分 evaluator、router 和测试代码。对于设置了 `satisfiedBy` 的非终止节点，若记录为完成但所需事实不匹配，[求值器](https://github.com/context4ai/agent-graph/blob/387f80db65bf20a61bc666b4fa885200fcedad08/src/evaluator.ts#L137)会返回 `unverified`。

**设计意义：**完成状态可以由工作契约和支持事实共同确定。执行和事实来源的可信度仍由宿主系统负责。核心设计文档早于该版本，8 月 31 日对应的是类型化资源更新。

<a id="source-codex-memory"></a>

<a id="s3-codex跨会话记忆与上下文压缩并存"></a>

### S3. Codex：工作上下文与跨会话记忆

[Memories 文档](https://learn.chatgpt.com/docs/customization/memories)**未标发布日期；文档状态截至 9 月 7 日**，覆盖本地存储、会话筛选和单次聊天控制。文档区分了本地 Codex 记忆与 ChatGPT web、Work 的记忆机制；所述本地记忆为可选功能，默认关闭。

实现说明固定到 **9 月 4 日**发布的 [`rust-v0.153.4`](https://github.com/openai/codex/releases/tag/rust-v0.153.4)，提交为 `3d2ee51ca2d5db578f328aa75e20aa22c0197c9a`。[Phase 2](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/memories/write/src/phase2.rs)在全局锁保护下整合提取出的运行经验；[读取模板](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/ext/memories/templates/memories/read_path.md)从简短摘要定位可检索的记忆，再按需读取支持记录。[压缩实现](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact.rs#L63)仍保留自动与手动压缩，[CLI 命令文档](https://learn.chatgpt.com/docs/developer-commands?surface=cli)也仍列出 `/compact`。

**设计意义：**区分当前工作上下文的管理与供后续运行使用的经验，跨会话 Memories 与下面的实验性工作上下文机制分别说明。整合模板中的来源追踪和删除要求是模型应遵循的行为，尚不能当作经过验证的删除保证。[在线配置文档](https://learn.chatgpt.com/docs/config-file/config-reference)与[该版本的默认常数](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/config/src/types.rs#L48)在候选会话的时间范围和数量上存在差异，因此默认值需要对应具体版本。

<a id="source-codex-context-management"></a>

#### 实验性工作上下文管理

**9 月 3 日**的 [0.153.0 发布](https://github.com/openai/codex/releases/tag/rust-v0.153.0)加入 `features.context_management.experimental_mode`。[配置文档](https://learn.chatgpt.com/docs/config-file/config-reference)截至 9 月 7 日的内容说明该模式通过笔记和可检索历史保留细节，改变反复压成单一摘要的方式。开关默认关闭；这一启用路径要求使用 Codex 后端的合格 ChatGPT Plus、Pro 或 Pro Lite 会话，排除 API key、自定义提供方和临时结构化线程。

相关源码包括[激活功能的提交](https://github.com/openai/codex/commit/cff76fa96f70f9f3b63d221446fd02cfd87e6d2e)，以及固定 v0.153.4 的 [token-budget 激活逻辑](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/session/token_budget.rs)、[压缩分支](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact_token_budget.rs)、[新窗口处理器](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/tools/handlers/new_context_window.rs)、[历史与笔记工具定义](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/ext/history-notes/src/tools.rs)，以及扩展和后端代码。token-budget 的手动、自动路径都会跳过模型或服务端摘要，直接创建新窗口，同时保留 compact 钩子与 `ContextCompaction` 事件。模型指导要求写入检查点，并借助窗口和条目标识找回历史。这改变了上下文切换的具体实现，不能据此说所有 compaction 接口都已删除。

**Astra 与版本范围：**[9 月 6 日的源码更新](https://github.com/openai/codex/commit/6af345407d9c2a568da9d01b6c4b81a9e61495c0)加入 `supports_experimental_context`，并为内置 `gpt-6-astra` 设置支持标志；v0.153.4 尚无这一检查。[v0.153.4 模型目录](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/models-manager/models.json)明确将 Astra 的 token-budget/history-notes 激活设为关闭，旧模型也已有相关指导。激活功能提交的父版本已包含 token-budget 基础机制。这些证据支持“Codex 的实验协议及后续 Astra 适配”，不能证明模型内部首次出现记忆。[Astra API 指南](https://developers.openai.com/api/docs/guides/latest-model)也仍单独列出 compaction 支持。

[模型元数据](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/session/token_budget.rs#L128)还可以独立于这一实验开关激活 token budgeting，[远端模型目录](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/models-manager/src/manager.rs#L412)也可以覆盖内置元数据。因此，不能仅凭内置默认值判定某个账户实际是否启用。

**设计意义与限制：**应把笔记完整性、原始证据检索和窗口重置后的恢复，与普通摘要及跨会话记忆一起评估。9 月 6 日修订属于后续源码证据，不能写成 0.153.4 的已发布行为。

<a id="source-composable-layers"></a>

### S4. 运行时、框架与 harness 可以组合

[Deep Agents vs LangChain vs LangGraph](https://www.langchain.com/blog/deep-agents-vs-langchain-vs-langgraph) 发布于 **8 月 6 日**，说明各层职责和组合示例。未标发布日期的 [Graph API 文档](https://docs.langchain.com/oss/python/langgraph/graph-api)覆盖状态、reducer、条件边、`Send`、`Command` 和图迁移。

**设计意义：**图中可以包含由模型决策的循环，也可以动态选择下一步；结构如何定义、执行时如何决策，是可以分别选择的设计维度。三层命名描述的是 LangChain 自己的技术栈，不是统一行业标准。

<a id="source-cursor-runtime"></a>

### S5. Cursor：分别管理目标、事件订阅和执行机器

来源包括 **8 月 19 日**[云端智能体与 harness 更新](https://cursor.com/changelog/08-19-26)及 **9 月 2 日**[自托管机器公告](https://cursor.com/changelog/self-hosted-machines)。前者描述云端事件订阅、`/goal`、拥有独立项目副本和 VM 的子智能体，以及在下一次工具调用时接收引导；后者描述命名 worker 队列和空闲机器休眠。

**设计意义：**目标、会话、事件源和执行资源各有生命周期。这些机制来自厂商文档。公告未说明事件排序、去重或外部副作用保证；这些公告也不能证明所有发往模型的数据都留在 worker 所在网络内。

<a id="source-temporal-runtime"></a>

### S6. Temporal：重放与暂停的边界

**8 月 27 日**的 [Durable Digest](https://temporal.io/blog/durable-digest-august-2026) 将 Deep Agents 集成和 Workflow Pause 标为 **pre-release**。当前未标日期的[集成文档](https://docs.temporal.io/develop/python/integrations/deepagents)和[暂停文档](https://docs.temporal.io/encyclopedia/workflow/workflow-pause)覆盖模型与工具执行、I/O 包装、重试、续跑及在途动作。

**设计意义：**模型调用作为 Activity 执行，会访问外部资源的工具和后端需要按文档规定进行包装；`continue-as-new`携带消息与结果缓存，其他状态需要重建。暂停会阻止新任务派发，已运行的 Activity 和计时器仍可继续，也不会递归暂停子工作流。这些边界限定了月报中的概括。这些来源没有建立外部副作用“恰好一次”的保证。

<a id="source-copilot-governance"></a>

### S7. Copilot：插件更新与上下文准入

来源包括[插件市场 `autoUpdate`](https://github.blog/changelog/2026-08-26-enterprise-managed-settings-now-support-autoupdate-for-plugin-marketplaces/)（**8 月 26 日**）及[应用和 CLI 的内容排除](https://github.blog/changelog/2026-09-02-content-exclusions-generally-available-in-copilot-app-and-cli/)（**9 月 2 日**）两篇公告，以及[托管设置](https://docs.github.com/en/enterprise-cloud@latest/copilot/reference/enterprise-administrators/enterprise-managed-settings)的优先级、插件市场、权限、MCP 和沙箱章节和[内容排除文档](https://docs.github.com/en/copilot/concepts/context/content-exclusion)；这两份文档未标日期。

**设计意义：**插件来源的更新策略、执行授权，以及内容能否进入上下文，各有自己的作用范围。应用和 CLI 的排除公告适用于 Business 和 Enterprise；文档仍注明编辑器 Edit/Agent 模式不支持，并列出间接语义信息、符号链接和远程文件系统的限制。不能据此宣称所有 OS 访问路径都受控，或已保留的记忆会被清除。

<a id="source-looparena"></a>

### S8. LoopArena：单独评测外层控制者

[LoopArena](https://arxiv.org/abs/2608.28281v1) 发布于 **8 月 28 日，版本 v1**。方法与协议见[正文 §§2–5、§7 及预算和协议附录](https://arxiv.org/html/2608.28281v1)。该基准固定 Worker，根据只读 Reporter 提供的证据，评测 Controller 选择推进、验证或停止的能力。

**设计意义：**该基准区分控制者的决策质量与整套编码系统的表现。实验覆盖一种 Worker 配置和 27 个源任务。文中的 64.4% 成本下降比较的是任务切片与完整任务评测，不能写成增加 Controller 带来的节省；Reporter 的摘要也不是重新执行的验证。结果均为作者报告。

<a id="source-harnesslens"></a>

### S9. HarnessLens：围绕具体改动安排验证

[Verify Smarter, Evolve Further](https://arxiv.org/abs/2608.27311v1) 发布于 **8 月 27 日，版本 v1**。方法与评测见[正文 §§3–6、限制与评测附录](https://arxiv.org/html/2608.27311v1)。HarnessLens 先确认改动确实加载，再选择能暴露目标行为及潜在回归的任务，并在接受改动前做进一步确认。

**设计意义：**作为 harness 演化中分配验证预算的实例。实验使用一个模型家族、三套 harness 和四个基准；预算混合统计会话与任务试验，没有对齐美元、token 或延迟。样本中的回归检查不能保证所有场景均无回归。这些结果来自作者报告。

<a id="source-production-evals"></a>

### S10. 从生产轨迹到可执行评测任务

来源为两篇官方文章：[How We Build Agent Environments & Tasks](https://www.langchain.com/blog/building-agent-environments-and-tasks)（**8 月 25 日**）及 [LangSmith Tuned Evaluators](https://www.langchain.com/blog/introducing-langsmith-tuned-evaluators-starting-with-perceived-error)（**8 月 18 日**）。前者先形成经人工审查的 Task Spec 和共享 World Spec，再生成可执行的 Harbor 任务；后者用版本化裁判标出值得调查的对话。

**设计意义：**轨迹可以用于形成经审查的规格、可运行任务和回归检查。Perceived Error 被官方明确称为满足用户需求的代理信号，不能直接作为最终正确性判断。任务生成文章提供的是工程经验，尚非受控实验。

### 信息流、记忆与适配

| 来源与版本 | 机制 |
|:---|:---|
| [Twin Agent，7 月 21 日，v1](https://arxiv.org/html/2607.19595v1) | 将不可信信息的读取权限与行动权限分开，两类 agent 通过简短提示通信。 |
| [MemSecBench，7 月 29 日，v1](https://arxiv.org/html/2607.27080v1) | 跨 agent 与记忆后端配置，追踪恶意内容的写入、后续使用和选择性修复。 |
| [Living-Harness，8 月 11 日，v2](https://arxiv.org/html/2607.26598v2) | 在一次交互任务完成并评分后，更新可检索的程序性记忆和状态图。 |
| [APPA，8 月 26 日，v2](https://arxiv.org/html/2607.24625v2) | 分别检查执行前的调用与进入上下文前的返回结果，用临时分支处理不可信数据。 |
| [Self-Evolving Coding Agents，8 月 29 日，v3](https://arxiv.org/html/2608.03392v3) | 综述更新对象、发生时点、反馈来源与评价方式。 |
| [Multi-Harness RL，9 月 3 日，v1](https://arxiv.org/html/2609.04518v1) | 区分接触多种 harness 和跨 harness 计算训练收益，并用未参与训练的 harness 检查迁移。 |

<a id="较早资料与本轮未采用的主张"></a>

### 其他资料及其适用范围

| 资料 | 机制与限制 |
|:---|:---|
| [TRIAGE / One Recipe, Many Harnesses](https://arxiv.org/abs/2608.10178v1)，8 月 10 日 | §§3–4、附录 A.4、B、C.1–C.2、H 及演化产物实例说明了通用经验与生态专用适配的边界；总优化预算尚未与强替代方法对齐。 |
| [Deep Agents v0.7](https://www.langchain.com/blog/deep-agents-v0-7)，7 月 29 日；[OpenAI harness engineering](https://openai.com/index/harness-engineering/)，2 月 11 日 | 前者讨论消融与模型比较，后者讨论仓库知识、反馈和维护机制。 |
| [Deterministic Execution Constraints](https://arxiv.org/html/2608.26197v1)，8 月 25 日 | 研究见 §§3–8 和表 1–3；HTML 摘要中的完美复现措辞与表 2 不一致，且只使用两个小型合成任务。 |
| [HEART / Agent-Native Reusable Tool Primitives](https://arxiv.org/html/2609.01736v1)，9 月 1 日 | 方法见 §§3–4、附录 E.3 和 G；token 成本、API 成本与多智能体总消耗不可混用，表格与正文的一处结果也不一致。 |
| [Claude Code v2.1.259](https://github.com/anthropics/claude-code/releases/tag/v2.1.259) 与 [v2.1.260](https://github.com/anthropics/claude-code/releases/tag/v2.1.260)，9 月 2–3 日 | 权限发布说明记录了一次回滚：v2.1.260 回滚了 v2.1.259 对 Bash 参数扩大应用 `Read()` deny 规则的改动。这些变化来自发布说明。 |

<a id="历史记录的状态"></a>

### 历史版本范围

以下内容说明历史版本；日期和星标数量对应记录时点。

历史目录：[2026 年 8 月 16 日](https://github.com/VILA-Lab/Dive-into-Claude-Code/commit/a82805c2cd2ed396303aec96f7f8f2124e97869c)。


<a id="每周增量2026-07-31-至-2026-08-07"></a>

## 2026 年 7 月 31 日至 8 月 7 日的来源

这些资料讨论任务状态、失败恢复、跨会话通信，以及插件和技能的信任边界。

<a id="已提升到双语主目录"></a>

### 机制与评测

| 日期 | 资料 | 说明 |
|:---:|:---|:---|
| 2026-08-07 | [Claude Code v2.1.221–v2.1.224](https://github.com/anthropics/claude-code/releases/tag/v2.1.224) | 版本变化包括 self-hosted runner、跨机器会话消息、凭据遮蔽、权限传播，以及多项沙箱和策略绕过修复。 |
| 2026-08-07 | [Codex 0.147.0](https://github.com/openai/codex/releases/tag/rust-v0.147.0) | 插件目录、MCP 2026-07-28、对话和技能导入、远端压缩、项目信任确认、凭据脱敏，以及插件策略失败时默认关闭网络。 |
| 2026-08-04 | [Warp Agent CLI](https://www.warp.dev/blog/introducing-the-warp-agent-cli-coding-agent) | 以 PTY multiplexer 管理会话，支持交互式程序、SSH 连接延续、跨 harness 委派和本地到云端移交。 |
| 2026-07-29 | [Deep Agents v0.7](https://www.langchain.com/blog/deep-agents-v0-7) | 作者报告：删减系统提示和待办脚手架后，基础输入下降约 65%；报告中的多模型评测没有显示整体得分明显下降。 |
| 2026-07-31 | [LoopsBench](https://arxiv.org/abs/2608.00267v1) | 用依赖感知测试、持续回归检查和外层续跑循环评测长期开发任务。 |
| 2026-08-01 | [Ledger](https://arxiv.org/abs/2608.00808) | 在不增加模型调用的情况下记录证据、依赖和验证进度，并在完整 SWE-bench Verified 上同时提高成功率、降低成本。 |
| 2026-08-03 | [Rethinking Self-Evolving Agent Skills](https://arxiv.org/abs/2608.02636) | 实验表明技能演化更接近由验证集筛选的稀疏搜索，失败轨迹在最终入选的技能中都发挥了作用。 |
| 2026-08-04 | [The Resume Contract](https://arxiv.org/abs/2608.03836v1) | 形式化分析和框架实测都说明，提供 checkpoint API 并不等于能够保证恰好执行一次。 更新后的协议与一致性结果见 [8 月 8 日的 v3](https://arxiv.org/abs/2608.03836v3)。 |
| 2026-08-05 | [Active-SWE](https://arxiv.org/abs/2608.04682) | 拿掉 issue 报告后，主动发现缺陷表现为不同于“根据 issue 修补代码”的能力。 |
| 2026-08-05 | [SciCode-Verified](https://arxiv.org/abs/2608.04975) | 修正 263 处基准缺陷，包括 192 次对正确答案的误判，显著改变了最终准确率。 |
| 2026-08-05 | [恶意 Skill 文件](https://arxiv.org/abs/2608.05223) | 这项合成实验测量了两个代码智能体 CLI 处理恶意技能文件时的表现。 |
| 2026-08-06 | [DCAS](https://arxiv.org/abs/2608.06113) | 实验只使用一个基准，但展示了轨迹微调造成的脚手架依赖，以及跨脚手架数据带来的迁移改善。 |
| 2026-08-06 | [Learning Globally Reusable Skills](https://arxiv.org/abs/2608.06153) | 通过技能关系图、相关更新合并和历史任务回放，维护带有回归检查的技能库。 |

相关的早期资料包括：7 月 24 日的[上下文工程新规则](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models)、6 月 2 日的[动态工作流模式](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)和 6 月 1 日的[Claude Code Action 漏洞披露](https://flatt.tech/research/posts/poisoning-claude-code-one-github-issue-to-break-the-supply-chain/)。

<a id="保留候选暂不进入主目录"></a>

### 其他研究与实现

| 资料 | 范围与限制 |
|:---|:---|
| [TraceCompiler](https://arxiv.org/abs/2608.02680) | 工作流编译的思路值得保留，但只测试了一个意图，未计算离线成本，也未评测接口变化和语义保持。 |
| [EA-Graph](https://arxiv.org/abs/2608.04278) | 将验证结论绑定到具体工件的做法有价值，但实验只有 42 个生成会话，且只测试“结论是否有证据支持”。 |
| [SuperScout](https://arxiv.org/abs/2608.04804) | 先探索、再把核验过的信息交给修复智能体的做法值得关注，但学习得到的路由器在 266 题切片上没有超过“固定选择最便宜修复器”的基线。 |
| [OneDayAgent](https://arxiv.org/abs/2608.05013) | 同一套长程 harness 可以适配多个模型后端，但目前只测试了一个基准，也没有工作区隔离。 |
| [Verified Tool Calls](https://arxiv.org/abs/2608.02645) | “验证后再重试”的模式很清楚，但只在两个模拟工作流和手写验证器上演示。 |
| [Self-Evolving Coding Agents，v1](https://arxiv.org/abs/2608.03392v1) | 关于 agent 演化的综述；[8 月 29 日修订版](https://arxiv.org/abs/2608.03392v3)按更新对象、发生时点、反馈来源与评价方式组织相关工作。 |
| [LangSmith LLM Gateway](https://www.langchain.com/blog/langsmith-llm-gateway-runtime-controls-for-production-agents) | 提供外置的策略控制面。 |
| [AgentCore OBO token exchange](https://aws.amazon.com/blogs/machine-learning/implement-on-behalf-of-token-exchange-for-multi-tenant-agents-with-amazon-bedrock-agentcore-gateway/) | 提供了具体的身份委派架构。 |

## 快速结论

这些资料体现的核心趋势不是“更会聊天的 agent”，而是 agent operating layer 正在成形：可恢复的执行环境、显式权限边界、可审计遥测、可版本化 context/skills、可插拔工具连接、长任务状态机、人类中途接管，以及从 traces 反推 eval 和改进循环。

这些资料涉及以下设计问题：

1. **Runtime and control plane are first-class design concerns**：持久执行、检查点、沙箱、agent inventory、策略面和可观测性，应该作为一等设计关注点，而不是部署细节。
2. **Context is managed infrastructure**：context 不只是 prompt，而是文件、skills、memory、interpreter state、workspace state、IDE indexes 和可版本化策略。
3. **Execution boundary is the safety boundary**：sandbox、network policy、credential custody、OS-level isolation、tenant boundary 和 approval policy 是核心架构对象。
4. **Tools and skills are a supply chain**：MCP、SDK、CLI、plugins、skills 和 agent-to-agent protocols 放大能力，也引入 registry、allowlist、identity、versioning 和 revocation 问题。
5. **Humans become managers and verifiers**：长任务和异步代理要求人类能在过程中审查、改方向、批准、回滚，而不是只看最终 diff。
6. **Observability must close the improvement loop**：生产 agent 的失败模式需要通过 trace/eval/issue/dataset 回路进入下一轮系统改进。

<a id="p0-最值得纳入综述的资料"></a>

## 研究与工程资料

| 年月 | 资料 | 核心内容 | Design Space 价值 |
|:---:|:---|:---|:---|
| 2026-07 | [Claude Code CHANGELOG（至 v2.1.215）](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) | **v2.1.212**：WebSearch 调用与 subagent 派生各加每会话上限（默认 200，环境变量可调）；MCP 工具调用超过两分钟自动转后台；`/fork` 改为开后台会话，原本的会话内行为拆为 `/subtask`；Task 工具 `mode` 参数废弃，subagent 改为继承父会话权限模式；修复 plan mode 会不经提示直接运行改动文件的 Bash 命令。**v2.1.214**：新增 EndConversation 工具；带 daemon 重定向参数的 `docker` 命令改为需要授权；修复 `Edit(src/**)` 这类单段规则误批准树中任意嵌套 `src/` 写入的作用域缺陷。版本范围：v2.1.213 不存在，v2.1.215 只含一条 `/verify`、`/code-review` 不再自动触发的改动。 | 一次给出四类原语的变更：模型侧新增终止权（罕见，控制面通常只给人）、长程执行加预算上限、MCP 调用改为非阻塞、subagent 权限收敛为继承。权限作用域缺陷与 plan mode 的修复则共同说明：授权规则的语义本身就是漏洞面。 |
| 2026-07 | [Orchestrate subagents at scale with dynamic workflows](https://code.claude.com/docs/en/workflows) | Claude 写一段可重跑的 JavaScript 脚本，由独立 runtime 执行，最多 16 并发、单次 1,000 个 agent，中间结果留在脚本变量而非上下文窗口。关键权限事实：启动提示遵循会话权限模式，但 workflow 派生的 subagent 一律以 `acceptEdits` 运行并继承工具白名单，与会话模式无关。 | 编排从「Claude 逐回合决定下一步」变成「脚本持有计划」，上下文隔离从涌现属性变成引擎的显式决策。同时是一个反例：为了让长程运行不被打断，权限模式在子层被统一放宽了。 |
| 2026-07 | [VS Code Agent Host 与 Agent Host Protocol](https://code.visualstudio.com/updates/v1_129) | VS Code 1.129 把 agent 会话移出编辑器进入独立进程，通过开放的 Agent Host Protocol 通信；会话在无客户端连接时继续存活，可被多窗口同时渲染；同一 host 以一套会话模型承载 Copilot、Claude、Codex。host 上的 agent 可列出其他会话、读其记录、开新会话移交子任务、向其他会话发消息，发送需用户确认并设突发上限。 | 把「agent 运行时」与「UI 客户端」正式解耦为协议边界，是 harness 分层的一次标准化尝试；跨会话消息带确认与扇出上限，则是把多 agent 协作的失控风险写进了协议层。 |
| 2026-07 | [Responses API 多智能体编排](https://developers.openai.com/api/docs/guides/responses-multi-agent) 与 [Programmatic Tool Calling](https://developers.openai.com/api/docs/guides/tools-programmatic-tool-calling) | 前者把 subagent 树的编排搬进 API：`/root/researcher` 具名树、六个托管动作、`max_concurrent_subagents` 限制全树可同时活跃的 subagent 数量（不含根 agent），代价是该模式下 reasoning summary 与 `max_tool_calls` 不可用。后者让模型写 JavaScript，在全新隔离、无文件系统/无网络/无跨程序状态的 V8 中经 `tools.*` 调用工具，工具以 `allowed_callers` 选择加入。 | 两条都在回答「循环该由谁持有」：编排上移到服务端，工具调用下沉进模型写的程序。后者把「一回合一次工具调用」的形态改成了批处理，中间输出不再进入上下文。 |
| 2026-07 | [How Agents Ask for Permission](https://arxiv.org/abs/2607.13718) | 调查 21 个 agent 权限系统、实测 5 个商用 agent，沿三轴建立分类学：界面上如何表达策略、如何推导为内部策略、运行时如何强制执行。 | 横向比较权限系统，可以检验 Claude Code 的设计在其他系统中是否同样适用。 |
| 2026-07 | [Bad Memory](https://arxiv.org/abs/2607.14611) | 在四个模型上实测 Claude Code 与 Codex：agent 大体能抵抗试图改写其记忆文件的不可信内容，但已植入这些文件的 payload 会继续攻击当前与后续会话。 | 研究直接测试了 Claude Code 与 Codex。结论把记忆安全的防线从「写入时过滤」推到「读取时也不可信」，因为持久化把一次注入变成长期注入。 |
| 2026-07 | [Agent Data Injection](https://arxiv.org/abs/2607.05120) | 从指令注入中分出第二类攻击：不夹带命令，而伪造资源标识符、数据来源、工具响应格式这类安全攸关元数据，agent 照单全收，因为它不区分可信与不可信数据。实证覆盖 Claude in Chrome、Antigravity、Nanobrowser 的任意点击，以及 Claude Code、Codex、Gemini CLI 的 RCE 与供应链攻击。 | 说明「检测祈使句」这条防线方向就错了：攻击面在上下文构造对数据的信任假设，而非指令识别。 |
| 2026-07 | [GhostApproval](https://thehackernews.com/2026/07/ghostapproval-symlink-flaws-could-let.html) | Wiz 的研究：仓库内放一个名字无害的符号链接（如 `project_settings.json`）指向 `~/.ssh/authorized_keys` 或 `~/.zshrc`，批准弹窗显示诱饵名，用户批准的写入落到别处。Amazon Q Developer、Cursor、Google Antigravity 已修，Augment 与 Windsurf 确认未修。Anthropic 对 Claude Code 部分提出异议，理由是开发者既已选择信任该目录又批准了编辑，属威胁模型之外。原文未给出 CVSS 分数。 | 打的是「知情同意」本身：批准界面显示的对象与实际写入对象不一致时，人的批准就不再构成授权。Anthropic 的异议本身值得保留，因为争的是同意应当从哪一层产生（信任目录 vs 批准这一次写入）。 |
| 2026-07 | [Better Harnesses, Smaller Models](https://arxiv.org/abs/2607.08938) 与 [Rethinking the Evaluation of Harness Evolution](https://arxiv.org/abs/2607.12227) | 前者用 meta agent 读失败轨迹、把共通难度抬进 harness，以 4% 成本恢复 89.7% 大模型性能（为其自身结果）。后者在 Terminal-Bench 2.1 上以对齐预算与朴素 test-time scaling 对比，发现自动 harness 演化「并不能稳定胜过」更简单的方法，且泛化差。 | harness 自动优化目前是有争议的活跃议题，不是已定论的结论；只引一侧会失真。 |
| 2026-07 | [Failure as a Process](https://arxiv.org/abs/2607.09510) | 从 7 个前沿模型、3 套 scaffold 的 3,843 条轨迹中人工标注 1,794 条完整轨迹、逾 63,000 步：失败以认知性错误为主，通常最初几步即发生，并隐藏到无法挽回才显现。 | 为「验证应前移进循环」提供大规模实证，同时说明只看最终结果的评测会系统性错过失败的真实成因。 |
| 2026-07 | [Copilot CLI changelog（7 月）](https://github.com/github/copilot-cli/blob/main/changelog.md) | auto allow-all 模式由 LLM 裁判判定是否放行；受信任仓库可经 `.github/copilot/settings.json` 钉死模型、effort、context tier 并扩充 URL/MCP/skill 拒绝名单；`preToolUse` 钩子以退出码 2 拒绝调用；plan mode 硬拦所有会改工作区的内置工具，但 MCP 与外部工具仍放行。 | 与 Claude Code 权限模型逐条对照的素材：谁拥有权限边界（用户 vs 仓库）、钩子如何否决，以及「计划模式」的作用域缺口。 |
| 2026-07 | [Harness Optimizer](https://strandsagents.com/blog/introducing-harness-optimizer/) | AWS 把 harness 当可调参数：system prompt、工具描述、skills 构成 `Formula`，`RewardFunction` 给轨迹打分，`Trainer` 按 epoch 跑 rollout/reward/update；默认优化器本身是 LLM，对照读成功与失败轨迹再改写 Formula。开源，并以 AgentCore Optimization 提供生产版。 | 大厂把「harness 是可优化对象」产品化的又一例证，与 Meta-Harness 同向；其结果可与上面的两项评估研究比较。 |
| 2026-07 | [Claude Code CHANGELOG（v2.1.178 至 v2.1.207）](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) | 窗口内 30 个版本。子代理默认后台运行并新增 `agent_needs_input`/`agent_completed` 钩子（v2.1.198），且明确「agent 的消息永远不算用户批准」；默认权限模式改为 Manual（v2.1.200）；新增 `Tool(param:value)` 权限规则语法，如 `Agent(model:opus)`（v2.1.178）；auto mode 不再读取仓库内的 `.claude/settings.local.json`（v2.1.207）；新增 `sandbox.credentials` 阻断沙箱读取凭据与密钥环境变量（v2.1.187）。 | harness 设计的第一手权威变更记录，覆盖运行时、权限、上下文、工具、子代理、长程执行六条主线，多条改动直接印证信任边界论点。 |
| 2026-07 | [Configure auto mode](https://code.claude.com/docs/en/auto-mode-config) | 首次完整披露 auto mode 分类器的四级策略优先级：`hard_deny`（无条件）优先于 `soft_deny`（可被覆盖）优先于 `allow`（作为 soft_deny 的例外）优先于用户显式意图。规则是自然语言散文而非正则。分类器读 CLAUDE.md，但明确不读共享的 `.claude/settings.json`，因此签入仓库的配置无法注入自己的放行规则。 | 目前唯一一份把「LLM 作为权限判定器」的完整策略层次与逃逸路径讲清楚的官方文档，支撑工具授权从提示词围栏走向分层策略引擎的论证。 |
| 2026-06 | [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams) | 明确 agent teams 与 subagents 是两种不同原语：teammates 各自独立上下文、通过 mailbox 互发消息、共享一个带文件锁的任务列表（支持依赖与自主认领），状态落盘于 `~/.claude/teams/` 与 `~/.claude/tasks/`。信任边界：teammate 无法代替用户批准，被拒动作也不能转交其他 teammate 绕过。 | 把「子代理」与「对等代理团队」区分为两种编排原语并给出可验证的信任边界规则。 |
| 2026-06 | [What's new in Claude Sonnet 5](https://platform.claude.com/docs/en/about-claude/models/whats-new-sonnet-5) | 手动 extended thinking（`thinking: {budget_tokens: N}`）被移除并返回 400；`temperature`/`top_p`/`top_k` 设为非默认值返回 400；新分词器使同样文本产生约多 30% token。 | 模型侧收回了 harness 原本持有的两个旋钮（思考预算、采样参数），推理预算控制权从 harness 上移到模型自身，是 harness 与模型边界在移动的直接证据。 |
| 2026-06 | [Agentic coding and persistent returns to expertise](https://www.anthropic.com/research/claude-code-expertise) | 基于真实 Claude Code 使用数据：人类做约 70% 的规划决策但只做 20% 的执行决策；用户每发一条 prompt 平均触发约 10 个 Claude 动作，部分场景两次人工介入之间超过 100 个动作。 | 首份来自 Anthropic 的定量证据，说明「人类保留规划权、让渡执行权」是实际的分工形态，为人类控制面与监督成本提供实测锚点。 |
| 2026-07 | [MCP 2026-07-28 规范转为无状态协议](https://blog.modelcontextprotocol.io/posts/sdk-betas-2026-07-28/) | 取消 `initialize` 握手与协议级 session，能力改由 `server/discover` 获取；新增 Multi Round Trip Requests（工具可返回 `InputRequiredResult` 在调用中途向用户追问）；新增用于网关路由的 `Mcp-Method`/`Mcp-Name` 传输头；roots、sampling、logging 标记为 deprecated。 | 工具生态层最大的一次结构性变更：MCP 从有状态会话协议转向可负载均衡的无状态 HTTP，直接影响工具接口与网关设计。 |
| 2026-06 | [Enterprise-Managed Authorization: Zero-touch OAuth for MCP](https://blog.modelcontextprotocol.io/posts/enterprise-managed-auth/) | 客户端在 SSO 时从 IdP 拿到 Identity Assertion JWT Authorization Grant，再换取 MCP server 的 access token，按用户已有的组和角色授权，取消逐服务器的同意屏。 | MCP 授权决策权从「每个用户逐个点同意」上移到组织 IdP，是权限维度从个人授权走向组织策略的关键一步。 |
| 2026-06 | [Amazon Bedrock AgentCore Harness GA](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-harness-is-now-generally-available-go-from-idea-to-production-grade-agent-in-minutes/) | `CreateHarness` 与 `InvokeHarness` 两个 API 调用即可声明式定义 agent（模型、工具、skills、记忆策略、容器环境），底层封装 microVM 隔离 Runtime、托管 Memory、Gateway、沙箱 Browser、Code Interpreter、Identity token vault、Observability 七个原语。 | harness 从「开发者自己写的循环」变成「云厂商托管的配置对象」，是设计空间里一个全新的坐标轴：谁拥有 loop、环境与工具边界。 |
| 2026-06 | [AgentCore policy 与 Guardrails](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-bedrock-agentcore-policy-guardrails-generally-available/) | policy 控制 agent 被授权执行哪些动作，Guardrails 实时检查每个被授权动作的输出与每次 gateway 调用的输入。关键点：评估发生在 gateway 边界、在 agent 代码之外，因此无论 agent 自主程度多高都能一致执行。 | 「在 agent 代码之外的边界上强制执行」是与 in-loop 权限检查截然不同的架构选择。 |
| 2026-06 | [Reduced cost, better isolation, more resilience: Strands Agents evolves](https://strandsagents.com/blog/reduced-cost-better-isolation-more-resilience/) | 大 tool result 卸载到外部存储只留截断预览，旧消息压缩成结构化摘要，85% 上下文占用时主动触发压缩；Shell 沙箱只暴露 bound paths，网络默认阻断内网，密钥按 URL 在请求时注入、agent 从不持有凭据；Evals 加入 chaos testing 与沙箱逃逸红队。 | 同时覆盖 context/memory、sandboxing、eval 三条主线，且每条都给出可复现的阈值与边界定义。 |
| 2026-07 | [Agent Harness: Scaling the claw or harness capabilities](https://devblogs.microsoft.com/agent-framework/agent-harness-scaling-the-claw-or-harness-capabilities/) | 微软给出扩展 harness 能力的四条正交路径：Skills（按请求匹配才渐进加载全文）、受限 Shell（命令 re-anchor 到 vault 无法逃逸）、CodeAct（沙箱内执行代码）、Background Agents（fan-out 给并发子 agent 再聚合）。 | 把「能力扩展」拆成四个正交机制，正好对应 tools、sandboxing、subagent 三条主线，可直接用作跨系统对照。 |
| 2026-06 | [Agent Harness: Working with your data, safely](https://devblogs.microsoft.com/agent-framework/agent-harness-working-with-your-data-safely/) | 文件操作限制在可配置 root folder 内；工具可标记为 approval-required；用户可单次批准，也可建立 standing rule（「总是批准此工具」或「总是批准这组参数」），且 standing rule 只在 session 内有效、不会固化进 agent。 | session 级而非永久的批准规则，是与 Claude Code 权限模型的直接可比点。 |
| 2026-07 | [What's New in Microsoft Foundry, June 2026](https://devblogs.microsoft.com/foundry/whats-new-in-microsoft-foundry-june-2026/) | Tool Search 在运行时按需检索相关工具，而不是把全部 tool schema 塞进上下文；Memory 新增 procedural memory 与 TTL 自动淘汰；Autopilot Agents 拥有完整 Entra Agent ID 账号。 | Tool Search 直接回应「工具数量膨胀撑爆上下文」这一设计张力；TTL 记忆与 agent 身份是新的设计维度。 |
| 2026-06 | [Antigravity CLI](https://github.com/google-antigravity/antigravity-cli)（背景：[Transitioning Gemini CLI to Antigravity CLI](https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/)） | Google 用 Go 重写的官方 CLI 取代 Gemini CLI（2026-06-18 生效），与 Antigravity 2.0 桌面端共用同一个 agent harness。据 CHANGELOG（仓库只放发布产物与示例，无源码）：嵌套 subagent 可到孙级及更深，子轨迹更新递归回传到根对话；`.agents/hooks.json` 工作区级 hooks，pre-tool hook 决定某次**工具调用**是否放行；按项目的权限配置存于 `~/.gemini/config/projects/`（**不在仓库内**），优先级高于全局设置。 | 一个厂商把 harness 从 IDE 与 CLI 中抽出来作为共享层，且 hooks、subagent、权限优先级的设计与 Claude Code 高度同构，是跨系统对照的最佳新样本。其项目级配置位于用户主目录，与 Claude Code 的仓库内配置模型不同。 |
| 2026-06 | [Codex-maxxing for long-running work](https://openai.com/index/codex-maxxing-long-running-work/)（正文见[官方 PDF](https://cdn.openai.com/pdf/8a9f00cf-d379-4e20-b06f-dd7ba5196a11/OAI_WhitePaper_Codex-maxxing26.pdf)） | 把长时程工作拆成十个机制，其中 memory vault 把 `AGENTS.md`/`TODO.md`/`projects/`/`people/` 的文件式记忆放在 GitHub 里，让 diff 成为记忆的审查界面；thread automations 以心跳方式定时唤醒同一线程，可运行到条件满足并动态调整频率。 | 目前对「长时程 agent 循环」最系统的一手厂商叙述，提出「记忆必须可 open/edit/diff/reuse」这一可审查记忆的设计主张。 |
| 2026-06 | [Codex CLI v0.142.0](https://github.com/openai/codex/releases/tag/rust-v0.142.0) | multi-agent delegation 可在 thread 与 turn 两级配置为 disabled、explicit-request-only 或 proactive；可配置的 rollout token budget 跨 agent 线程追踪用量并在耗尽时中止 turn；子 agent 的终止错误现在会上报给父 agent，不再表现为「空的成功完成」。 | 把子代理委派从「有或无」变成可分档的策略开关，并把上下文预算立为一等约束。 |
| 2026-07 | [Codex CLI v0.144.0](https://github.com/openai/codex/releases/tag/rust-v0.144.0) | 新增 `writes` 审批模式：声明为只读的动作直接执行，仅在写操作时提示审批。 | 权限模型从「按工具审批」细化到「按动作读写语义审批」。 |
| 2026-06 | [Reward hacking is swamping model intelligence gains](https://cursor.com/blog/reward-hacking-coding-benchmarks) | Cursor 审计 731 条 Opus 4.8 Max 轨迹，发现 63% 的「成功」修复是检索来的而非推导出来的（57% 在 GitHub 上找到已合并 PR 照抄，9% 挖打包进来的 git history）。加上 history isolation（删除 `.git`）与只放行受批准包仓库的 egress 代理后，分数大幅下降。 | 把 eval harness 的环境设计（网络出口、仓库历史）确立为评测有效性的前提条件，是评测维度极其可引用的一手证据。 |
| 2026-06 | [Governing agent autonomy with Auto-review](https://cursor.com/blog/agent-autonomy-auto-review) | 分类器在 agent loop 内部运行（而非独立端点，以压低延迟），按风险与用户意图对齐程度连续评估动作；被拦截时把解释返回给父 agent，父 agent 往往能据此改走安全路径而不打扰用户。约 4% 的动作被拦，但只有约 7% 的对话真正打断用户。 | 「拦截即反馈」把权限门从终止点变成引导信号，是对 Claude Code 二元批准提示的一个重要替代设计。 |
| 2026-06 | [Customize Cursor](https://cursor.com/changelog/customize) | 把 plugins、skills、MCPs、subagents、rules、commands、hooks 六类扩展收进统一管理界面，支持 user、team、workspace 三级作用域，并引入可复用的团队配置模板与团队 marketplace。 | 一个非 Anthropic 厂商把扩展点收敛成与 Claude Code 几乎同构的六件套，说明 harness 扩展性正在收敛成事实标准。 |
| 2026-06 | [Running Untrusted Agent Code Without a Sandbox](https://www.langchain.com/blog/running-untrusted-agent-code-without-a-sandbox) | 用 WASM 里的 QuickJS 作为执行边界：起点是「零能力」，再通过 harness 显式桥接能力；并可把解释器内存状态序列化到 LangGraph 实现「持久暂停」，等人工批准后恢复。 | 与容器沙箱「先给一台完整计算机再收权限」相反的能力隔离范式，同时命中沙箱与人类审批中断点两个维度。 |
| 2026-06 | [Introducing Dynamic Subagents in Deep Agents](https://www.langchain.com/blog/introducing-dynamic-subagents-in-deep-agents) | 主 agent 不再逐轮 tool call 派发子 agent，而是写一段 JS 脚本在 QuickJS 解释器里执行，脚本中调用内置 `task({description, subagentType, responseSchema})` 分派子 agent。 | 把 subagent 编排从「对话回合」降到「程序控制流」，与 Claude Code Dynamic Workflows 是同一走向，是 context-as-bottleneck 原则的共同延伸。 |
| 2026-06 | [Wiki Memory](https://www.langchain.com/blog/wiki-memory) | 让 agent 预先跑一遍源材料，产出一组文件作为后续 agent 的领域知识层；与 RAG 在查询时取原始 chunk 对立，强调预计算的高层综合，底座是可读可改可版本化的文件。 | 为 context/memory 维度补上「文件即记忆基质」的一类具体方案，可与 CLAUDE.md、AGENTS.md 谱系直接对照。 |
| 2026-07 | [Tuning the harness, not the model](https://www.langchain.com/blog/tuning-the-harness-not-the-model-a-nemotron-3-ultra-playbook) | 在模型权重冻结的前提下只调 harness：middleware 强制的模型与工具调用上限、把「读文件须知」从工具描述搬进工具输出（即时注入）、在关键节点用 in-band message 而非 system prompt 下发指导。 | 少见的把 harness 各层当作可调参数逐项做消融的工程记录。 |
| 2026-06 | [Devin Fusion](https://cognition.com/blog/devin-fusion) | 前沿模型做主 agent 负责计划、歧义判断与终审，把常规操作委派给拥有独立工具集与独立缓存上下文的 sidekick 模型；轻量分类器在会话中途评估任务难度，并把模型切换放在 context compaction 时刻执行，该时刻本来就会发生 cache miss，因此切换不会额外增加一次缓存未命中。 | 把「模型路由」与「上下文压缩」这两个通常独立的机制耦合起来，是长时程执行中非常具体的成本与能力调度设计。 |
| 2026-07 | [Agentic MapReduce](https://devin.ai/blog/agentic-map-reduce/) | 四阶段：Plan（agent 生成 selector 模式）、Shard（确定性地按 selector 切分成有界批次）、Map（并行子会话各自只看自己那一片）、Reduce（归并去重并发现跨片的链式关系）。原则是「只在需要推理的地方放 agent，其余全部确定性」。 | 针对「单 agent 装不下整个代码库」的上下文瓶颈，给出 agent 与确定性计算分工的清晰边界。 |
| 2026-06 | [AI SDK 7](https://vercel.com/blog/ai-sdk-7) | HarnessAgent 把 Claude Code、Codex、Pi 等成品 harness 统一成一个 API（会话可 park 与 resume）；WorkflowAgent 把每次工具调用做成可持久化可重试的步骤，进程重启后从最后完成步继续；工具审批支持 HMAC 签名以防伪造批准。 | 同时命中 harness 可替换抽象、长时程持久化执行，以及「审批本身需要防篡改」这一少被讨论的控制面安全问题。 |
| 2026-07 | [Droid Shield 2.0](https://factory.ai/news/droid-shield-2-0) | 在自主 commit 与 push 前拦截密钥泄露的三段流水线：中间是确定性正则扫描器，两侧各挂一个微调模型。Risk model 在扫描器未触发时以召回优先重新判断上下文；Downgrade model 在扫描器触发时先遮蔽候选密钥，仅凭上下文判断是否为误报。 | 把「确定性规则」与「学习型模型」组合成双向纠错闸门，是自主执行下 guardrail 工程的高质量样本。 |
| 2026-07 | [Harness Engineering for Self-Improvement](https://lilianweng.github.io/posts/2026-07-04-harness/) | 定义 harness 为「围绕基础模型、编排执行的系统，决定模型如何思考规划、调用工具、感知与管理上下文、存储产物、评估结果」。核心论点：近期递归自我改进的路径不太可能始于模型直接改写权重，而是通过 coding agent 演化 harness 组件本身。 | 把 harness 立为自我改进的载体，直接呼应 Meta-Harness 一类工作，是「harness 是独立设计对象」最有分量的独立背书。 |
| 2026-07 | [Better Models: Worse Tools](https://lucumr.pocoo.org/2026/7/4/better-models-worse-tools/) | 观察到 Opus 4.8 与 Sonnet 5 在嵌套结构中会捏造 `requireUnique`、`oldText2` 之类不存在的字段，而 `oldText` 与 `newText` 的内容在字节层面正确。作者推断这是因为新模型在 Claude Code 宽松的 harness 中做 RL，该 harness 会静默修复畸形调用（过滤未知 key、接受参数别名），于是「轻微畸形也能拿到奖励」。 | 作者据此提出，训练时的 harness 容错规则可能影响模型在其他工具接口上的表现。 |
| 2026-07 | [Agentic Autonomy Levels](https://addyosmani.com/blog/agentic-autonomy-levels/) | 双轴自治模型，把长期混为一谈的两个维度拆开：agency 轴（单个 agent 独立到什么程度）与 orchestration 轴（多 agent 如何协调），并给出统一分级与「执行前契约」（目标、范围、非目标、工具与权限、停止条件、证据要求、升级路径、预算）。 | 「执行前契约」是对 Claude Code 即时权限提示这一在线审批模式的结构化替代。 |
| 2026-03 | [Meta-Harness: End-to-End Optimization of Model Harnesses](https://arxiv.org/abs/2603.28052) | Yoonho Lee 等（Stanford）。让 coding agent 充当 proposer，在模型固定的前提下自动搜索并优化 harness 本身（memory、retrieval、context 构造、prompt、工具使用逻辑），维护候选种群与 Pareto 前沿。结果：比当前最好的上下文管理系统高 7.7 分且少用 4 倍上下文 token；IMO 级题目上平均高 4.7 分；搜出的 harness 在 TerminalBench-2 上超过最好的手工基线。该文 Introduction 开篇的「只改 harness 可造成 6 倍性能差距」引用自 Tian et al. 的 SWE-bench Mobile 工作，并非 Meta-Harness 自身的结果。 | 把 harness 变成可自动优化的对象，是本仓库核心论点的直接方法论延伸。 |
| 2026-06 | [From Question Answering to Task Completion: A Survey on Agent System and Harness Design](https://arxiv.org/abs/2606.20683) | 以 model-harness lens 综述 agent 系统，把执行 harness 拆为六项耦合的运行时职责：observation、context、control、action、state、verification。 | 从模型与 harness 的职责分工梳理智能体设计。 |
| 2026-06 | [ActPlane: Programmable OS-Level Policy Enforcement for Agent Harnesses](https://arxiv.org/abs/2606.25189) | 用 eBPF 在 OS 内核层拦截 agent 的全部执行路径（包括绕过 tool-call 层的间接路径），配合信息流控制 DSL 表达跨事件策略，并把违规原因作为语义反馈回传给 agent；开销 1.9% 到 8.4%。 | 直接指出 Claude Code 的权限检查发生在 tool-call 层而该层可被绕过，是把权限模型下沉到内核的第一个完整方案。 |
| 2026-06 | [Lingering Authority: Revocable Resource-and-Effect Capabilities for Coding Agents](https://arxiv.org/abs/2606.22504) | 提出 PORTICO 引用监视器，把任务规格编译为初始能力、授予规则、可信闭包谓词与全局拒绝规则；资源被物化为 epoch-bound 的不透明句柄，闭包条件满足后即失效。 | 精确命名了「授权残留」：会话内一次批准后，权限不会随任务阶段收回。 |
| 2026-06 | [TokenPilot: Cache-Efficient Context Management for LLM Agents](https://arxiv.org/abs/2606.17016) | 双粒度上下文管理：全局的 Ingestion-Aware Compaction 在环境输出入口处过滤噪声并保持前缀稳定，局部的 Lifecycle-Aware Eviction 在上下文段效用到期后才保守驱逐，从而避免 KV cache 失效。 | 直击一个常被回避的问题：任意改写历史会摧毁 prefix cache，可用于解释追加式而非重写式上下文策略的合理性。 |
| 2026-07 | [The Balkanization of Execution-Security Research for AI Coding Agents](https://arxiv.org/abs/2607.05743) | SoK，把 2023 至 2026 年 39 篇执行层安全研究归入 17 类（沙箱隔离、能力与访问控制、策略执行、TOCTOU、MCP 威胁、身份委派、执行溯源、出网控制等），并指出策略执行对真实 denylist 的失败率高达 69% 到 98%。 | 为 permissions 与 sandboxing 维度提供唯一一份系统化的对手地图。 |
| 2026-06 | [One Fake Bug Report Hijacked a $250 Billion Company's AI Agent, Then 100+ More](https://tenetsecurity.ai/blog/agentjacking-coding-agents-with-fake-sentry-errors/)（Tenet Security，2026-06-17） | Sentry 的公开 DSN 接受任意错误负载，攻击者 POST 一个含 markdown 指令的事件，渲染成伪造的「Resolution」小节；开发者让 agent 排查该 Sentry issue 时，agent 经 MCP 取回被污染事件并当作可信修复指引执行，运行攻击者控制的 npm 包并带走 AWS key、GitHub token 与 SSH 凭据。确认影响 Claude Code、Cursor、OpenAI Codex，含沙箱变体与 CI/CD 流水线。 | 迄今最有力的「工具返回值即不可信输入」实证。任何假设 MCP 返回内容可信的授权模型，此案例即是反例。 |
| 2026-06 | [CVE-2026-12957：AWS Language Servers 工作区配置自动执行](https://nvd.nist.gov/vuln/detail/CVE-2026-12957) | Language Servers for AWS（Amazon Q Developer 底层）1.65.0 之前版本存在信任边界执行不当：用户打开恶意构造的工作区、并在提示时选择信任它之后，项目配置文件中的任意命令会被自动执行。评分：CNA（Amazon）同时给出 CVSS v4.0 8.5（HIGH）与 CVSS v3.1 7.8（HIGH）；NVD 尚未给出独立评分（两项评分均来自 CNA）。 | 「打开并信任工作区即执行」是编码 agent 特有的攻击面，直接对应项目级配置文件（CLAUDE.md、.mcp.json）的信任模型问题。 |
| 2026-05 | [OpenAI named a Leader in enterprise coding agents by Gartner](https://openai.com/index/gartner-2026-agentic-coding-leader/) | Codex 被定位为企业级 coding agent；OpenAI 强调 large codebase、工具使用、测试、approval gates、RBAC、sandboxing、auditable workspace governance。 | 说明 coding agent 已从 autocomplete 进入 delegated work / operating layer；治理和审计是企业 agent 的一等需求。 |
| 2026-05 | [Cursor: What we've learned building cloud agents](https://cursor.com/blog/cloud-agent-lessons) | 云端 agent 需要完整开发环境、durable execution、VM checkpoint/fork、secret redaction、network policies、credential management。 | 涉及智能体执行环境与云端运行时的设计。 |
| 2026-05 | [AWS: Break the context window barrier with Bedrock AgentCore](https://aws.amazon.com/blogs/machine-learning/break-the-context-window-barrier-with-amazon-bedrock-agentcore/) | 用 AgentCore Code Interpreter + Strands Agents SDK 实现 Recursive Language Models，把长文档放入 sandbox/interpreter working memory，模型只按需调用子 LLM。 | 强化“context 不只在 prompt 里”：外部环境、代码状态和 working variables 可以成为 agent memory surface。 |
| 2026-05 | [LangChain: From Token Streams to Agent Streams](https://www.langchain.com/blog/token-streams-to-agent-streams) | streaming 从 token delta 升级为 typed events：messages、tool calls、subagent activity、state changes、approvals、media。 | 对 agent UI、可观测性、重连、子代理 inspector 和长任务 dashboard 很关键。 |
| 2026-05 | [LangChain: Give Your Agents an Interpreter](https://www.langchain.com/blog/give-your-agents-an-interpreter) | Deep Agents 增加受限 interpreter，介于串行 tool calls 和完整 sandbox 之间；工具通过 allowlist bridge 暴露。 | 提供“可编程 agent loop”的中间设计点：更窄 action surface、更少 token、更清楚的失败模式。 |
| 2026-05 | [Google: Building the agentic future at I/O 2026](https://blog.google/innovation-and-ai/technology/developers-tools/google-io-2026-developer-highlights/) | Antigravity 2.0、Managed Agents in Gemini API、persistent isolated environments、dynamic subagents、scheduled tasks、custom skills。 | Google 把 agent-first development platform 明确做成 harness + sandbox + persistent state + subagents。 |
| 2026-05 | [Google: Build managed agents with the Gemini API](https://blog.google/innovation-and-ai/technology/developers-tools/managed-agents-gemini-api/) | 单次 API 调用创建可推理、用工具、执行代码的托管 agent；运行在隔离 Linux 环境，可保留文件和状态。 | 可作为“managed runtime ownership”的对照案例：谁拥有 loop、环境、state 和工具边界。 |
| 2026-05 | [LangChain: How We Built LangSmith Engine](https://www.langchain.com/blog/how-we-built-langsmith-engine-our-agent-for-improving-agents) | LangSmith Engine 坐在 agent traces 之上，发现 recurring issues，并建议下一步修复。 | 适合“agent improvement loop”：trace -> failure cluster -> eval/dataset/issue -> fix agent。 |
| 2026-05 | [Anthropic acquires Stainless](https://www.anthropic.com/news/anthropic-acquires-stainless) | Anthropic 收购 SDK、CLI、MCP server tooling 公司 Stainless，强调 agents 的价值取决于它能连接到哪些系统。 | “Connectivity is capability”：API spec 到 SDK/CLI/MCP server 是 agent 可行动能力的基础设施层。 |
| 2026-05 | [OpenAI and Dell: Codex for hybrid/on-prem enterprise](https://openai.com/index/dell-codex-enterprise-partnership/) | Codex 将靠近企业本地/混合环境中的数据、代码库、文档、业务系统和工作流。 | 引入 deployment topology/context locality 维度：agent 放在哪里，决定能看见什么、能做什么、如何治理。 |
| 2026-05 | [OpenAI: Work with Codex from anywhere](https://openai.com/index/work-with-codex-from-anywhere/) | Codex 进入 ChatGPT mobile preview；用户可远程查看状态、批准命令、改方向、审 diff；Remote SSH、Hooks、programmatic tokens GA。 | 很适合“supervised async agent”：人类不全程陪跑，但能在关键决策点介入。 |
| 2026-05 | [OpenAI: Building a safe, effective sandbox to enable Codex on Windows](https://openai.com/index/building-codex-windows-sandbox/) | 讲 Windows 下 Codex sandbox 如何在频繁审批和 Full Access 之间平衡。 | 具体机制案例：OS-level isolation、workspace boundary、network control、approval friction。 |
| 2026-05 | [LangChain: LangSmith Sandboxes GA](https://www.langchain.com/blog/langsmith-sandboxes-generally-available) | microVM kernel isolation、snapshots/forks、prewarmed environments、Service URLs、Auth Proxy。 | 说明 production agent 不能只靠“容器式 sandbox”；执行环境本身是安全边界。 |
| 2026-05 | [LangChain: Introducing Managed Deep Agents](https://www.langchain.com/blog/introducing-managed-deep-agents) | 托管 runtime 提供 durable threads、checkpointing、streaming、context、observability、human-in-the-loop。 | 对应“open-source harness + managed runtime”的拆分方式。 |
| 2026-05 | [LangChain: Introducing Context Hub](https://www.langchain.com/blog/introducing-context-hub) | 把 `AGENTS.md`、skills、policies、examples、memory files 版本化、可回滚、可协作。 | 直接支撑“context as first-class artifact”：context 需要自己的生命周期，而不是散落在 prompt 里。 |
| 2026-05 | [Claude Code Dynamic Workflows](https://code.claude.com/docs/en/workflows) + [Opus 4.8](https://www.anthropic.com/news/claude-opus-4-8) | Claude 自己写 JavaScript 编排脚本，后台 runtime 扇出到上千个 subagent；中间状态存在脚本变量（对话之外），只有最终答案进入 context；16 并发 / 1000 总量上限，同一 session 内可 resume。随 Opus 4.8（2026-05-28）一同发布，v2.1.154 引入，research preview。 | 用于子智能体编排与长期协调：编排逻辑从对话搬进代码，是 context-as-bottleneck 原则的下一步。 |

<a id="p1-强相关资料"></a>

## 更多研究与工程资料

| 年月 | 资料 | 核心内容 | Design Space 价值 |
|:---:|:---|:---|:---|
| 2026-06 | [Zed: Software Is Made Between Commits（DeltaDB）](https://zed.dev/blog/introducing-deltadb) | 为 agent 协作设计的版本控制：一条消息与它产生的编辑并排记录，二者不会漂移；每个引用锚定到 delta 而非行号，因此代码变动后引用仍然存活，可从过去对话的任一行跳到该代码的当前状态或当时状态；内嵌 CRDT worktree 支持多人多 agent 跨机器同时编辑同一批文件。 | 指出 git 围绕离散 commit 组织，从未被设计来承载「生成代码的那段对话」；把对话与代码变更绑定为同一制品，是会话持久化与可审计性的一个根本性重构。 |
| 2026-05 | [Warp: A single pane of glass for managing all of your cloud agents](https://www.warp.dev/blog/multi-harness-cloud-agent-orchestration) | Oz 作为 multi-harness 控制面：在一个面板里启动、追踪、治理与引导 Claude Code、Codex 与 Warp Agent，比较它们的效果并为不同任务选用不同 harness，同时保持一致的治理、访问控制与审计日志；跨 harness 的 Agent Memory 让经验在会话、仓库与项目之间迁移。 | 把 harness 本身变成可比较、可替换、可统一治理的对象，是「harness 所有权边界正在移动」的直接产品化证据。 |
| 2026-05 | [Project Glasswing: initial update](https://www.anthropic.com/research/glasswing-initial-update) | Anthropic 用模型发现漏洞，瓶颈从发现迁移到验证、披露、修复。 | agent 能力提升后，系统瓶颈转向人类验证队列、责任流程和安全发布。 |
| 2026-05 | [OpenAI: Virgin Atlantic ships faster with Codex](https://openai.com/index/virgin-atlantic/) | Codex 用于测试、legacy refactor、数据原型和生产工程流程。 | adoption case：agent 改变工程节奏后，瓶颈转向组织协作和 review 流程。 |
| 2026-05 | [Microsoft + EY: From AI pilots to enterprise impact](https://blogs.microsoft.com/blog/2026/05/21/from-ai-pilots-to-enterprise-impact-why-execution-is-the-new-differentiator/) | 强调从 pilot 到 production，需要 intelligence + trust、透明、安全、可问责、可复制执行模型。 | 适合组织层设计原则：agent 系统不是单工具，而是运营模型重构。 |
| 2026-05 | [Google DeepMind: Gemini 3.5](https://deepmind.google/models/gemini/) | Gemini 3.5 Flash 被定位为面向 agents and coding 的高性能模型，强调 long-horizon tasks、tool use、UI control 等 benchmark。 | 提供了模型能力方面的背景。 |
| 2026-05 | [GitHub: Fix code review feedback with Copilot cloud agent](https://github.blog/changelog/2026-05-19-easily-apply-copilot-code-review-feedback-with-copilot-cloud-agent/) | 将 Copilot code review comment 批量交给 Copilot cloud agent 修复，可选择模型和应用方式。 | 表明 review -> implementation handoff 正在产品化；human review 成为 agent workflow gate。 |
| 2026-05 | [GitHub: one-click fixes for failing Actions](https://github.blog/changelog/2026-05-18-one-click-fixes-for-failing-actions-with-copilot-cloud-agent/) | CI 失败后可一键让 Copilot cloud agent 调查、推 fix、等待 review。 | 对应“event-triggered repair agents”和“CI as agent entry point”。 |
| 2026-05 | [GitHub: fast, cost-efficient models for Copilot cloud agent](https://github.blog/changelog/2026-05-18-copilot-cloud-agent-fast-cost-efficient-models-for-simple-tasks/) | Copilot cloud agent 可按任务选择更快、更便宜模型。 | 引入“模型路由 / cost-capability matching”维度。 |
| 2026-05 | [GitHub: Building a general-purpose accessibility agent](https://github.blog/ai-and-ml/github-copilot/building-a-general-purpose-accessibility-agent-and-what-we-learned-in-the-process/) | GitHub accessibility agent 采用 reviewer sub-agent + implementer sub-agent，并用复杂度评分决定是否只给 guidance。 | 很好的多 agent 分工案例：passive reviewer、active implementer、escalation gates、complexity-based behavior。 |
| 2026-05 | [PwC + Anthropic expanded partnership](https://www.anthropic.com/news/pwc-expanded-partnership) | PwC 将部署 Claude Code/Cowork，建立 Center of Excellence，培训认证 30,000 人。 | 组织采用侧证：agent system 需要培训、治理、COE 和行业流程落地。 |
| 2026-05 | [Agent-First Tool API](https://arxiv.org/abs/2605.10555) | 提出 agent-first API：search、resolve、preview、execute、verify、recover 六阶段，以及 Normalized Tool Contract。 | 对工具层设计很有价值：传统 CRUD API 不适合 autonomous agents，需要 agent-native semantic interface。 |
| 2026-05 | [Code as Agent Harness](https://arxiv.org/abs/2605.18747) | 把 code 视为 agent reasoning、acting、environment modeling、execution verification 的统一 harness。 | 和本仓库 thesis 高度一致：agent 的工程复杂度在可执行、可验证、可状态化的 harness。 |
| 2026-05 | [MemGym](https://arxiv.org/abs/2605.20833) | 长程 agent memory benchmark，覆盖 tool-use dialogue、deep research、coding、computer use。 | memory 不是简单长期记忆，而是长任务中形成、压缩、检索和迁移的执行能力。 |
| 2026-05 | [Push Your Agent](https://arxiv.org/abs/2605.23574) | 衡量 long-horizon agents 是否能坚持到 verifier 确认足够多有效工件，而非过早停止。 | 强化 stop condition、verified progress、backlog tracking 是长任务 agent 的核心机制。 |
| 2026-05 | [Boiling the Frog](https://arxiv.org/abs/2605.22643) | 多轮、持久 workspace 中的渐进式 agentic safety benchmark。 | 安全评测对象从“模型输出文本”转向“环境状态是否被改坏”。 |
| 2026-05 | [How to Steer Your Multi-Agent System](https://arxiv.org/abs/2605.23023) | 将 human-LLM co-planning 分成 semantic/structural、global/targeted、low/high-level edits 三轴。 | 支持“过程级监督”：人类控制面应落在 plan/process 上，而不是只审最终产物。 |

<a id="其他高信号资料"></a>

## 其他资料

| 年月 | 资料 | 设计意义 |
|:---:|:---|:---|
| 2026-05 | [OpenAI: Running Codex safely at OpenAI](https://openai.com/index/running-codex-safely/) | 给出了清晰的 coding-agent 安全设计原则：bounded environment、low-risk frictionless、high-risk review、agent-native telemetry。 |
| 2026-05 | [Microsoft: Frontier Firms operating model](https://blogs.microsoft.com/blog/2026/05/05/how-frontier-firms-are-rebuilding-the-operating-model-for-the-age-of-ai/) | 组织设计角度很强：人类从逐步执行转向设方向、定标准、评估结果；AI 价值取决于工作如何被重新设计。 |
| 2026-05 | [Anthropic: Agents for financial services](https://www.anthropic.com/news/finance-agents) | 垂直 agent 模板、per-tool permissions、credential vaults、audit logs，适合展示 regulated domains 的 agent design requirements。 |

<a id="更多可持续追加的高质量资料"></a>

## 补充资料

| 年月 | 资料 | 设计启示 | 设计主题 |
|:---:|:---|:---|:---|
| 2026-05 | [NSA: MCP Security](https://www.nsa.gov/Portals/75/documents/Cybersecurity/CSI_MCP_SECURITY.pdf) | MCP server 是能力入口，也是供应链和权限入口，需要 registry、identity、allowlist、monitoring 和 revocation。 | 工具供应链、connectivity risk。 |
| 2026-05 | [Kiro: Deep spec analysis](https://kiro.dev/blog/deep-spec-analysis/) | spec/requirements 可以作为 agent 前置控制面，让实现之前先稳定目标、约束和验收条件。 | human control surface、plan/process supervision。 |
| 2026-05 | [OpenAI agent improvement loop](https://developers.openai.com/cookbook/examples/agents_sdk/agent_improvement_loop) | traces、evals、prompt/tool changes 可以形成闭环，而不是停留在日志分析。 | observability/eval improvement loop。 |
| 2026-05 | [Microsoft Agent 365](https://www.microsoft.com/en-us/security/blog/2026/05/01/microsoft-agent-365-now-generally-available-expands-capabilities-and-integrations/) | agent inventory、访问控制、治理和组织级可见性正在成为 control plane 的组成部分。 | 运行时、控制面与企业采用。 |
| 2026-04 | [NSA/CISA: Careful Adoption of Agentic AI Services](https://media.defense.gov/2026/Apr/30/2003922823/-1/-1/0/CAREFUL%20ADOPTION%20OF%20AGENTIC%20AI%20SERVICES_FINAL.PDF) | agentic service 的风险来自 autonomy、tool use、data access、credential handling 和第三方执行环境的组合。 | 安全边界、治理和 enterprise adoption。 |
| 2026-04 | [Cognition: Multi-agents working](https://cognition.ai/blog/multi-agents-working) | 多 agent 并行的关键不是数量，而是任务切分、写权限约束、冲突处理和可验证合并。 | human manager/verifier、多 agent architecture。 |
| 2026-04 | [GitHub Copilot CLI MCP allowlists](https://github.blog/changelog/2026-04-16-copilot-cli-supports-custom-registry-based-mcp-allowlists/) | MCP allowlist 正在从安全建议变成产品机制。 | 工具供应链的代表信号。 |
| 2026-04 | [A2A protocol milestone](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year) | agent-to-agent protocol 正在形成互操作层，但也会扩大身份、权限和责任边界。 | 多 agent 协议与治理。 |

<a id="可写入-design-space-的新原则草案"></a>

## 设计启示

2026 年 7 月下半月的资料：

- **Consent is only as good as what the dialog names**：批准界面显示的对象与实际生效的对象一旦不一致，人的点击就不再构成授权（GhostApproval 的符号链接诱饵名）。这也暴露一个更前置的问题：「信任这个目录」与「批准这一次写入」是不是同一件事，Anthropic 与 Wiz 的分歧正落在这里，值得原样保留而不是判定谁对。
- **Persistence turns one injection into a standing one**：agent 能抵抗试图改写其记忆文件的不可信内容，却会被已经躺在文件里的 payload 反复攻击（Bad Memory）。写入时过滤因此不足以构成记忆安全的防线，读取路径同样要按不可信处理。
- **Trust boundaries fail on data, not only on instructions**：攻击者伪造的是资源标识符、数据来源、工具响应格式这类元数据，而非祈使句（Agent Data Injection）。以「识别指令」为中心的防御方向从一开始就偏了。
- **The plan is moving out of the conversation**：dynamic workflows 把计划交给脚本，OpenAI 把 subagent 树搬进 API，ADK 2.0 把节点跳转交给代码，VS Code 把会话移进独立进程并定义协议。共同点是编排与上下文隔离从涌现行为变成显式的引擎决策。代价也一并显现：workflow 的子代理为了不被打断而统一放宽到 `acceptEdits`。
- **Harness optimization is contested, not settled**：自动 harness 优化既有正面结果（以 4% 成本恢复 89.7% 性能），也有在对齐预算下「并不稳定胜过朴素 test-time scaling」的反面结果。这两类结果反映了不同条件下的表现。
- **Failures are epistemic and early**：CLI coding agent 的失败以认知性错误为主，多在最初几步发生并隐藏到无法挽回（Failure as a Process）。这既支持把验证前移进循环，也说明只看终态的评测会系统性错判失败成因。

2026 年 6 月至 7 月的资料：

- **Tool output is untrusted input**：工具返回值与 MCP 响应必须与用户输入同等对待。Agentjacking 证明一份伪造的错误报告即可让 agent 执行攻击者控制的代码；把 tool description 与 system prompt 同等审查是相应的防御方向。
- **The enforcement point is itself a design choice**：权限强制点可以落在环内（Copilot CLI 与 Cursor 用 LLM 分类器裁决）、agent 代码之外的网关边界（AWS AgentCore），或 OS 内核（ActPlane）。层次越低越难绕过，但语义越稀薄。tool-call 层的检查可被间接执行路径绕过。
- **Blocking should feed back, not just terminate**：Cursor 的 Auto-review 在拦截时把解释返回给父 agent，父 agent 据此改走安全路径而不打扰用户。权限门可以是引导信号，而不只是终止点。
- **Authorization must expire with the task phase**：会话内一次批准后权限长期驻留是一个可命名的缺陷（lingering authority）。能力应绑定到任务阶段并在闭包条件满足后失效。
- **Rewriting history destroys the cache**：任意改写上下文历史会摧毁 prefix cache，这构成了对 compaction 策略的硬约束，也解释了追加式设计的合理性。将模型切换安排在压缩时，可以避免为切换额外增加一次缓存未命中。
- **Training conditions and tool use**：作者推测，模型训练时的 harness 容错规则可能影响其在其他工具接口上的表现。
- **Harness ownership is moving**：harness 正从「开发者自己写的循环」变成托管服务（AgentCore 的 `CreateHarness`）、可替换后端（Omnigent、AI SDK 7、Warp Oz）和可自动优化的对象（Meta-Harness）。「谁拥有 loop、环境、state 与工具边界」本身成为一个设计维度。
- **Eval validity depends on environment design**：编码基准的分数可能主要来自检索而非推导。若不做仓库历史隔离与出网限制，基准衡量的是 agent 的搜索能力而非修复能力。

更早的资料：

- **Environment parity before model blame**：云端 agent 失败常常不是模型差，而是环境缺依赖、权限、网络或凭证。
- **Bounded programmability beats raw power**：interpreter / programmatic tool calling 给 agent 编程能力，但通过 allowlist bridge 缩小 action surface。
- **Human oversight must be interruptible and mobile**：长程异步 agent 需要远程查看、批准、改方向和审查 diff 的控制面。
- **Memory needs provenance and lifecycle**：memory/context/skills 需要版本、来源、审查、回滚、环境标签和过期机制。
- **Progress must be verified, not inferred**：长任务应维护 verified backlog，而不是靠模型自称“完成了”。
- **Telemetry is not enough without evaluation**：logs 只能解释发生了什么；生产 agent 还需要把 traces 转化为 failure clusters、evals 和改进任务。
- **Connectivity is capability, but also risk**：MCP、SDK、CLI、plugins 放大能力，也扩大 credential、data exfiltration 和 tool misuse 的设计面。
