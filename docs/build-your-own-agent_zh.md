[返回主 README](../README_zh.md) · [English](build-your-own-agent.md)

# 构建你自己的 AI 智能体：设计空间指南

> 本指南从 Claude Code 的架构分析出发，结合其他系统和近期研究，梳理构建生产级 AI 智能体时需要做出的设计决策。

每个生产级编码智能体都要回答同样的几个设计问题。Claude Code 给出的是其中一套答案。本指南介绍这些选择如何组合，以及你需要什么证据来判断它们是否适合自己的系统。

**阅读范围：** 本文的 Claude Code 架构示例以本仓库分析的 v2.1.88 源码快照为准，后续产品变化和跨系统对照会单独标明。来源核验截至 2026 年 10 月 6 日；证据及限制见[架构分析](architecture_zh.md)和[来源说明](agent-design-space-source-notes_zh.md)。

---

## 决策 1：推理放在哪里？

**问题：** 多少决策逻辑放在模型里，多少放在 harness 代码里？

| 方法 | 示例 | 权衡 |
|:---------|:--------|:----------|
| **模型主导的循环** | Claude Code v2.1.88（按本项目的代码分类，约 1.6% 属于 AI 决策逻辑） | 模型自主选择行动，harness 管理执行、上下文和权限。可靠性仍取决于模型及其运行环境。 |
| **显式状态图** | LangGraph | 状态、依赖和转移更容易检查；节点内部和路由仍可使用模型判断。工作流变化时，也需要维护图结构。 |
| **显式规划与任务跟踪** | 多阶段工作流 | 中间目标更清楚；当观察结果推翻原有假设时，需要及时修订计划。 |

**从 Claude Code 得到的启发：** 一个简洁的推理循环可以依赖大量基础设施。上下文供给、工具执行、恢复和验证都值得单独设计。应结合自己的工作负载评估模型与 harness；代码占比或榜单分差都不足以决定需要多少脚手架。

**Graph Engineering 放在哪里：** 2026 年的 [Graph Engineering 综述](https://arxiv.org/html/2608.21156v2#S4)从任务结构、agent 协调和运行状态三个方面组织问题。这个框架有助于明确依赖关系、职责和完成条件。它讨论的范围比用知识图谱做检索更广，也延续了此前基于图的 agent 研究。

图可以规定依赖和验收条件，节点内的 agent 则自主选择下一步行动。例如，[LangGraph 的 agent 示例](https://docs.langchain.com/oss/python/langgraph/workflows-agents#agents)通过条件边，根据模型是否调用工具来决定继续还是停止。实际设计中，可以把这样的结构与反馈循环结合，由 harness 执行动作并记录结果。需要选择的是哪些关系必须明确表达，哪些决策可以保持开放。

**保留前面已经满足的要求。** [LoopsBench](https://arxiv.org/html/2608.00267v1#S2.SS6)从开始就提供全部附带测试，但只有前置单元通过后，才启用当前开发单元的评分。已完成单元的测试会在后续工作中继续作为回归检查。agent 仍可自行选择编辑位置。

**图也可以用于审查已完成的工作。** [Trace2Flow](https://arxiv.org/html/2609.13136v1#S4) 是一个研究原型，它把已完成的 agent 执行轨迹转成可编辑的工作流图。用户可以查看每步的输入输出、修改依赖并重新运行步骤。控制图指导执行，这种图则在运行后组织已记录的工作。应保留每个步骤与其轨迹证据的联系。

**可以问自己的问题：**

- 哪些决策适合交给模型判断，结果又如何检查？
- 无论模型怎样规划，哪些依赖或审批条件都必须满足？
- 你需要的是已知图中的动态路由，还是执行时修改图本身？

---

## 决策 2：你的安全姿态是什么？

**问题：** 如何防止智能体做出有害的行为？

| 方法 | 示例 | 权衡 |
|:---------|:--------|:----------|
| **拒绝优先与分层检查** | Claude Code 的权限架构 | 在多个边界设置检查；仍需检查它们是否共享假设和故障模式。 |
| **容器隔离** | SWE-Agent、OpenHands | 按挂载、网络、凭据和权限配置限制访问；实际边界由容器配置决定。 |
| **VCS 回滚** | Aider 的 Git 集成 | 有助于恢复受版本控制的文件变更；无法撤回网络请求或其他外部副作用。 |
| **审批门控** | 交互式 agent 工具 | 由用户授权具体操作；预览必须与实际执行一致，反复确认也会消耗注意力。 |

**从 Claude Code 得到的启发：** 分层检查需要明确的回退行为。如果解析、分类或预算限制导致某项检查无法完成，应提前规定是拒绝操作、在隔离环境中执行，还是交给用户审阅。检查数量多，并不说明它们具有独立的故障模式。

**授权有作用域，也有有效期。** 应区分持久策略、运行模式，以及针对某个动作或会话的授权。委派任务或恢复会话时，只应携带对当前资源和环境仍然有效的权限。持久化方面的影响见[决策 6](#决策-6会话如何持久化)。

**区分信息访问与行动权限。** [Twin Agent](https://arxiv.org/html/2607.19595v1#S3.SS3) 的读取者从不可信来源提取提示，在固定长度限制内发给执行者。只有执行者能执行特权动作，它不能直接读取这些来源。应明确每个 agent 的输入与权限，再检查两者之间允许传递什么。

[APPA v2](https://arxiv.org/html/2607.24625v2#S4)在执行前检查工具调用，并在数据进入上下文前检查实际返回值。它可以通过临时分支查看不可信数据，并预先限定返回格式。拒绝接收返回数据，不会撤销已经发生的外部效果。

**可以问自己的问题：**

- 你的智能体可能做出的最糟糕的事情是什么：删除生产数据、误发消息，还是泄露代码？
- 能否通过沙箱减少用户必须做出的决策？
- 授权是否覆盖实际执行的动作，包括参数和目标资源？
- 某项检查超时，或子任务请求扩大访问范围时，该怎样处理？

---

## 决策 3：你如何管理上下文？

**问题：** 上下文窗口是有限的。你如何决定模型看到什么？

下面的机制可以组合使用，它们解决的问题和信息保留时间各不相同。

| 方法 | 示例 | 权衡 |
|:---------|:--------|:----------|
| **渐进式压缩管道** | Claude Code 的五层压缩分析 | 尝试较小幅度的削减，再考虑完整摘要；阶段越多，需要验证的相互影响也越多。 |
| **简单截断** | 基础对话历史 | 容易实现，但可能丢掉重要的早期上下文。 |
| **滑动窗口** | 聊天应用 | 大小可预测，但无法识别信息在语义上的重要性。 |
| **检索** | 仓库搜索与 RAG | 把详细资料留在 prompt 之外；结果需要相关、可溯源，并带有足够的上下文。 |
| **摘要** | 长对话 | 缩减历史长度，但可能丢失约束或未完成工作。 |
| **跨窗口笔记与历史检索** | Codex 实验性上下文管理 | 记录检查点，在窗口重置后按需找回早期细节；恢复依赖笔记、记录标识和历史服务。 |
| **跨会话记忆** | Codex 本地记忆 | 跨任务复用筛选后的经验；需要规定写入、检索、纠正和遗忘的方式。 |

**从 Claude Code 得到的启发：** 上下文稀缺影响着惰性加载、工具 schema 延迟加载、子 agent 回传格式和工具结果预算。应在窗口耗尽之前设计好信息流。

**渐进式方法：** 本项目分析的 Claude Code 管道结合了预算削减、历史裁剪、缓存感知压缩、上下文虚拟投影和完整摘要。实际启用哪些阶段取决于触发条件与配置；这是特定版本的实现，并非所有 agent 都必须依次执行的流程。

**区分当前上下文管理与跨会话经验复用。** Codex 的[本地记忆](https://learn.chatgpt.com/docs/customization/memories)从符合条件的旧会话中提取经验。在正在执行的任务里，[摘要式压缩](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact.rs#L63)与实验性的笔记、历史检索模式，则提供了不同的跨窗口方式。仅看 [CLI 中的 `/compact` 命令名](https://learn.chatgpt.com/docs/developer-commands?surface=cli)，无法判断实际走哪条路径。

实验开关在 [CLI 0.153.0 中加入](https://github.com/openai/codex/releases/tag/rust-v0.153.0)，这一开关需要主动启用，并面向使用 Codex 后端的合格 ChatGPT Plus、Pro 或 Pro Lite 会话，激活 token 预算提示、历史笔记和 `new_context`。协议要求 agent 保存检查点、切换到新窗口，再通过笔记与历史恢复细节。在 v0.153.4 中，[这条压缩分支](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact_token_budget.rs)跳过模型或服务端摘要，但保留 compact 钩子和生命周期事件。后续主线新增模型能力检查，并将 Astra 标为支持；[版本记录](./agent-design-space-source-notes_zh.md#source-codex-context-management)将它与已发布快照分开说明。

这种设计把可编辑的工作笔记与可检索的历史分开，使检查点质量、来源引用和恢复行为成为 agent 控制策略的一部分。需要评估窗口重置后，目标、约束、权限和证据是否得以保留且仍然有效；有历史存储，并不保证 agent 能找回正确细节。

对于可复用经验，需要规定哪些内容值得保存、何时重新核验，以及纠正如何影响后续检索。Codex 的[整合指令](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/memories/write/templates/memories/consolidation.md#L157)要求模型删除仅由已移除输入支持的指导，同时保留仍有来源支持的内容。这是一套需要验证的更新协议，不能保证每条过时记忆都会被正确清除。必须遵守的规则应保存在明确的策略或指令文件中。

Codex 9 月 8 日的[主线更新](https://github.com/openai/codex/commit/2cbbf0c9b542a36a1c3284b5e804917635b6f666)增加记忆 v2，读取路径使用选定版本。自[稳定版 0.155.0](https://github.com/openai/codex/releases/tag/rust-v0.155.0)（2026 年 9 月 17 日）起，[记忆配置](https://github.com/openai/codex/blob/rust-v0.155.0/codex-rs/config/src/types.rs#L294)接受 `version` 和 `dual_write`。在 0.160.0 中默认仍为 v1；截至 2026 年 10 月 6 日，[配置参考文档](https://learn.chatgpt.com/docs/config-file/config-reference)未列出这两个键。

**明确记忆的使用范围和修改权限。** [v2 提取指令](https://github.com/openai/codex/blob/2cbbf0c9b542a36a1c3284b5e804917635b6f666/codex-rs/memories/write/templates/memories/stage_one_system_v2.md)要求模型区分特定任务的要求与长期偏好，并在任务范围内应用后来的纠正。Claude Tag 的[公共频道便笺](https://github.com/anthropics/claude-code/releases/tag/v2.1.268)仅供本频道召回，工作区便笺则仍然共享。[Hermes Agent v0.21.2](https://github.com/NousResearch/hermes-agent/blob/939e45c91d751fadd94dcd1b873ac3cb44846213/tools/memory_tool.py#L129) 的内置记忆工具在无人值守的后台复盘中，要求先获批准才能替换或删除记忆。

**把召回的记忆当作输入，并限制谁能开启记忆。** 根据 [Claude Code 更新日志](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)，v2.1.284（2026 年 9 月 28 日）在 MEMORY.md 和被召回的记忆便笺送入模型之前，中和其中的不可见字符和模仿 Claude Code 自身标记的标签。自 v2.1.285（9 月 29 日）起，后台会话或由 Claude Code 自身工具启动的会话不能开启[自动记忆](https://code.claude.com/docs/en/memory)，但仍可关闭。这项清洗来自厂商的说明，并不能保证防住所有被投毒的记忆。

**明确哪些指令文件生效，以及修改何时生效。** 自 Claude Code v2.1.277（2026 年 9 月 18 日）起，如果工作目录及其上级目录都没有 CLAUDE.md，项目会把 AGENTS.md 作为项目指令读取。[Project instructions 设置](https://code.claude.com/docs/en/memory)提供四种模式：优先读 CLAUDE.md，没有时读 AGENTS.md（默认）；两者都读；只读 CLAUDE.md；或只读托管指令。以这种方式读取的 AGENTS.md 不触发 InstructionsLoaded hook。[Codex 0.156.0](https://github.com/openai/codex/releases/tag/rust-v0.156.0)（9 月 22 日）在每次模型请求前重新加载全局指令，但仓库指令只在环境或信任级别变化时重新发现；新的子 agent 继承父 agent 已应用的指令快照。

**可以问自己的问题：**

- 目标、约束、待办、权限和证据中，哪些内容必须跨压缩或窗口重置保留？
- 哪些信息属于当前工作状态、执行日志、可复用记忆或必须遵守的指令？
- 仓库发生变化或来源被撤回后，旧指导如何失效？

---

## 决策 4：你如何处理可扩展性？

**问题：** 外部工具、自定义指令和用户定制如何插入你的系统？

| 方法 | 示例 | 权衡 |
|:---------|:--------|:----------|
| **不同的扩展入口** | Claude Code 的 hooks、skills、plugins 和 MCP | 分别支持生命周期代码、按需指令和外部工具；上下文开销取决于实际加载、执行和回传的内容。 |
| **单一统一 API** | 工具调用框架 | 为扩展提供统一接口；仍需为发现、schema 加载和返回结果安排预算。 |
| **插件市场** | IDE 扩展 | 便于分发，也引入了来源信任、更新和执行权限方面的决策。 |

**从 Claude Code 得到的启发：** 在相关时再加载扩展指令和 schema。命令 hook 可以不调用模型就处理事件，但回传到对话中的内容仍然占用上下文。同样，MCP 的开销取决于发现和使用方式，协议本身没有固定的上下文成本。

**三个注入点：** 检查 agent 循环时，可以从扩展介入的位置入手：

1. **assemble()** —— 模型看到什么：指令、工具 schema 和检索上下文。
2. **model()** —— 模型能通过已暴露的工具请求哪些动作。
3. **execute()** —— 操作是否执行、怎样执行：权限门控与前置、后置钩子。

这里描述的是概念边界，实现不必使用这些函数名。对每个扩展，都要区分安装或更新权限与实际执行动作的权限。

**明确一次工具操作如何跨越多个请求。** [MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/changelog)移除了协议级会话，应用通过显式句柄传递跨调用状态。工具需要补充输入时，[多轮请求协议](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr)返回 `input_required`；客户端带上 `inputResponses` 重试，并原样回传服务端提供的 `requestState`。这些是同一逻辑操作的多个独立请求，因此需要明确谁保存操作状态，以及重试如何处理可能已经发生的效果。

**将命令授权绑定到已审阅的输入。** 在 [Claude Code v2.1.271](https://code.claude.com/docs/en/plugins-reference#plugin-install) 中，安装或更新插件可能需要用户批准市场声明的命令。命令显示但未执行时，`--json` 结果会包含该命令及其摘要值。用户审阅命令后，在自己的终端中传入 `--accept-command <sha256>`。同意绑定到该命令、插件和市场目录，任一项变化都会使其失效。

**明确进程内扩展与各项权限检查的先后顺序。** Claude Code 的 [mods](https://code.claude.com/docs/en/plugins/mods/overview) 首次列于 v2.1.287 更新日志（2026 年 10 月 1 日），是在 Claude Code 进程内运行、不受沙箱限制的插件函数，可以在权限提示出现前批准工具调用。在存在托管设置或使用 Team、Enterprise 登录的机器上，内置守卫（`sec-default`）会先加载：deny 规则（除非管理员设置了 `allowModsToOverrideDenyRules`）和托管 `PreToolUse` hook 仍然优先；守卫读不到托管设置时会拒绝加载用户的 mod。用户的 mod 仍可批准 `ask` 规则本应提示的调用；在 auto mode 下，这类调用不经过分类器。[管理员指南](https://code.claude.com/docs/en/plugins/mods/admin)列出了这些规则；在赋予扩展批准权之前，应先确定它相对每项检查的位置。

**可以问自己的问题：**

- 智能体会暴露多少工具，它们的 schema 何时进入上下文？
- 谁可以发布或更新扩展，这会怎样影响正在运行的会话？
- 扩展由哪个进程执行，可以访问哪些文件、网络和凭据？

---

## 决策 5：子智能体如何工作？

**问题：** 智能体派生子任务时，它们是共享上下文还是隔离运行？

| 方法 | 示例 | 权衡 |
|:---------|:--------|:----------|
| **隔离上下文 + 选择性回传结果** | Claude Code 的侧链转录稿 | 限制进入父级上下文的子会话历史；交接时仍需保留足以支持下一步决策的证据。 |
| **共享上下文** | 共享历史的多 agent 系统 | 所有参与者都能获得信息，但会增加上下文压力和相互干扰。 |
| **消息传递** | Actor 模型系统 | 通信边界明确，但需要定义进度、失败和完成的消息协议。 |

**从 Claude Code 得到的启发：** 上下文隔离可以把详细探索留在子会话里。委派仍有成本，需要测量总模型工作量、重复探索和协调开销。独立上下文也不意味着文件系统、进程或权限相互隔离。

**定义角色时，也要定义交接条件。** 每项任务都需要负责人、依赖、交付物和验收条件。[Agent Graph v0.3.0](https://github.com/context4ai/agent-graph/blob/387f80db65bf20a61bc666b4fa885200fcedad08/src/evaluator.ts#L137)提供了一个具体例子：对于带 `satisfiedBy` 条件的非终止节点，如果仅被标记为完成，而输入事实不满足条件，结果仍是 `unverified`。这实现了一种完成检查；宿主系统仍需负责提供可信事实。

**在接受并发改动前检查它们能否共存。** 独立 worktree 让 agent 可以分别编辑，但各自的改动仍可能冲突。[Claim Plane 原型](https://arxiv.org/pdf/2607.21909v1)在写入前检查带版本的修改意图；扩大范围时，也要重新检查才能执行新增修改。任务分配、写入权限和合并结果的验收需要分别决定。

**明确谁持有共享会话状态。** 在 [Agent Host Protocol](https://code.visualstudio.com/blogs/2026/08/26/agent-host-architecture) 中，宿主持有会话，并向多个客户端发送快照和有序更新。每个 harness 保留自己的 agent 循环、上下文管理和工具。因此，共同的会话接口仍需明确客户端操作在何时生效。

**明确消息何时生效。** [VS Code 1.137](https://code.visualstudio.com/updates/v1_137#_agent-queued-messages) 会将 agent 发给忙碌 chat 的消息排队，当前回合成功结束后再按发送顺序开始处理排队消息。通过其 [Agent Host 工具](https://code.visualstudio.com/docs/agents/run/sessions/manage-sessions#_orchestrate-sessions-from-agent-host-sessions) 向另一会话发送消息需要用户确认。既要规定发送权限，也要规定交付时机。

**说明每个停止操作结束的是什么。** Claude Code 在 2026 年 9 月 17 日至 10 月 5 日的发布（v2.1.275 至 v2.1.290）中，把停止一个回合与停止后台工作分开。自 v2.1.281 起，send-now 键（ctrl+enter，在 Claude 仍在工作时发送排队消息）把正在运行的工具移到后台，而不取消当前回合；在 VS Code 扩展中，Stop 和 Escape 只结束当前回合，后台 agent 继续运行，可在 agent 地图中逐个停止（v2.1.286）。v2.1.285 为后台 shell 命令加入时限；自 v2.1.288 起，该时限只适用于无人值守的会话，例如 `-p`、Agent SDK、CI 和云端会话（[更新日志](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)）。每个停止操作都应说明它结束的是当前回合、某一项后台任务，还是会话中的全部工作，以及正在运行的工具会怎样处理。

**明确 worker 能否质疑任务简报。** Cognition 在 [Fusion 设计说明](https://cognition.com/blog/local-fusion)中介绍了按模型组合调整简报细度、worker 质疑指令的空间，以及探索分工的做法。应把这些选择纳入委派协议，再用自己的模型和任务进行检验。

**可以问自己的问题：**

- 子任务需要彼此的完整历史，还是只需要特定产物和消息？
- 谁可以修改共享状态，谁负责验证结果？
- 子任务继承哪些权限，如何限定在委派范围内？
- 父级能否干预或停止正在运行的子任务，并区分部分结果与完成结果？

---

## 决策 6：会话如何持久化？

**问题：** 会话结束时会发生什么？什么会延续下来？

| 方法 | 示例 | 权衡 |
|:---------|:--------|:----------|
| **仅追加的 JSONL** | Claude Code 的会话日志 | 容易检查，也便于重建已记录的事件；单靠日志无法捕获所有外部副作用。 |
| **数据库与检查点** | 持久化 agent 运行时 | 支持查询、协调和恢复位置；需要明确的 schema、保留规则和恢复语义。 |
| **无状态请求** | 未保存应用状态的 API 调用方式 | 每次请求更简单；需要连续性、审计或恢复时，由应用补上。 |

**恢复任务状态时，要明确执行权限。** 应区分持久策略、运行模式和临时授权。本项目分析的 Claude Code 快照把 computer-use 应用白名单和会话 bypass 标记设为不持久化；这些例子不足以推出所有权限状态都必须丢弃。后来的 [v2.1.246](https://github.com/anthropics/claude-code/releases/tag/v2.1.246)就修复了特定 VS Code 和 headless 入口恢复时丢失 plan mode 的问题。设计上应保留原本需要生效的限制，并按授权的作用域和有效期重新检查。

**持久化还需要恢复协议。** 应记录哪个 worker 负责哪项任务、哪些外部动作已经完成，以及哪些结果仍待交付。发生崩溃后，要明确谁有权重试，怎样避免或处理重复副作用。仅有检查点，并不能保证外部操作恰好执行一次。

例如，Temporal 的 Deep Agents 集成在[发布时标为 pre-release](https://temporal.io/blog/durable-digest-august-2026)，将可重放的 Workflow 状态与 Activity 中的模型调用、外部 I/O 分开。[集成文档](https://docs.temporal.io/develop/python/integrations/deepagents)要求按规定包装会访问外部资源的工具和后端；持久执行依赖这些边界得到正确实现。

**区分重试与新工作。** [Resume Means Resume v3](https://arxiv.org/html/2608.03836v3#S3)区分普通恢复与有意创建分支，并检查一次批准是否已经被使用。外部效果可能已完成、但结果尚未记录时，应保留操作标识。还要明确暂停时哪些工作仍在进行：[Temporal 处于 pre-release 的暂停功能](https://docs.temporal.io/encyclopedia/workflow/workflow-pause)停止新派发，但正在运行的 Activity 仍可完成。

**把操作标识写进工具契约。** [Where Does Exactly-Once Live?](https://arxiv.org/abs/2609.29095v1)（arXiv，2026 年 9 月 24 日）在工具边界注入故障，例如在 agent 停止等待之后才提交的写入，或被投递两次的请求。作者（仅一人）在合成基准上报告：被要求恰好执行一次的前沿模型，在确认丢失时几乎从不重复写入，但对仍在途或被投递两次的请求经常重复执行；为每次写入提供幂等键后，重复率从 28% 降到 4%。图运行时同样需要注意：自 [Google ADK 2.9.0](https://github.com/google/adk-python/releases/tag/v2.9.0)（9 月 10 日）起，失败的工作流节点会在恢复时重新运行，因此发布说明要求节点体保持幂等；一个执行了副作用后失败的节点，每次恢复都会再次执行该副作用。

**为计划和活跃运行分别设置控制。** [VS Code Automations](https://code.visualstudio.com/docs/agents/run/automations) 处于 Preview。禁用计划会阻止后续计划运行，但不会停止当前运行；停止该会话是另一项操作。计划执行还要求机器保持唤醒：Agent Host 类型的自动化需要宿主进程运行，其他类型需要 VS Code 窗口运行。

**可以问自己的问题：**

- 新 worker 能否区分已完成的工作与结果尚不确定的动作？
- 如果两个 worker 同时恢复同一任务，谁有权继续执行？
- “完成”意味着计算结束、结果已被接受，还是用户已收到交付？

---

<a id="元模式三个反复出现的设计承诺"></a>

## 贯穿六个决策的三个原则

以下三个原则贯穿这六个决策，也把 Claude Code 的分析与更广的设计空间连接起来：

1. **明确每种机制的边界。** 说明一个上下文阶段、权限检查或扩展控制什么，以及失败时怎样处理。

2. **让决策关联可检查的证据。** 日志、检查点和记忆各有用途。保留足够的来源信息，才能重新检查完成声明，或纠正学到的经验。

3. **把模型判断与明确的执行规则结合。** 模型可以在节点或循环中选择行动，系统则检查依赖关系、执行权限约束，并按条件验收结果。

---

## 验证整个运行过程

这些设计决策会在接受工作、恢复任务和复用经验时汇合。扩大循环的自主范围之前，应先明确这些检查。

| 状态转变 | 需要的证据 |
|:-----------|:--------------------|
| **推进或停止** | 当前任务要求、相关检查的结果、未解决的问题，以及继续或停止的理由。预算耗尽可以构成停止理由，但不能证明任务完成。 |
| **恢复或重试** | 当前任务归属、有效授权和此前外部动作的状态；输入或环境变化后，重新检查受影响的证据。 |
| **保留记忆、skill 或 harness 改动** | 支持该更新的轨迹、明确的适用范围、回归检查，以及修订或撤回更新的方式。 |

Mastra Factory 当前的[看板规则](https://factory.mastra.ai/configure/boards-and-rules)分别定义允许的阶段转移、审批策略，以及进入和离开阶段时执行的动作。列出允许的转移不会自动移动卡片。应检查什么事件触发移动、是否需要审批、由谁审批，以及卡片离开或进入阶段时运行哪些动作。

**合并之后继续检查。** PR 打开时，[Cursor Rollouts](https://cursor.com/changelog/rollouts-and-security-reviewer)（2026 年 9 月 23 日，面向 Teams 和 Enterprise）会发布一份作者可以编辑的监控计划；每次部署后，它按环境分别给出结论：健康、回归或无法判断。发现回归时，视配置而定，它可以开一个待审查的 revert PR，或把结果交给 agent 处理，但不会自行合并或回滚。保留“无法判断”这一结论，可以避免把缺少证据当作通过。

[LoopArena](https://arxiv.org/html/2608.28281v1#S2)在固定 Worker 的条件下评估控制决策；它的只读 Reporter 负责整理证据，不能运行测试。[HarnessLens](https://arxiv.org/html/2608.27311v1#S4)围绕候选改动的目标行为选择检查任务，并保留独立测试集。评估技能更新时，应区分决定是否接受修改的案例与留作最终测试的案例：[SkillAdam](https://arxiv.org/html/2609.08944v1#S5.SS3)复用采样案例判断是否接受修改，并另留测试集。

**接受修复时，保护此前成功的案例。** 在 [Self-Healing Harness](https://arxiv.org/abs/2609.24130v1) 研究（arXiv，2026 年 9 月 21 日）中，agent 提出规则，由外部运行时决定保留哪些：只有修复了触发它的失败、且不使此前成功的受保护案例退化的规则才会保留。作者报告，在因回放而被否决的 383 个提议中，211 个修复了触发失败，却破坏了一个受保护案例；每轮最多回放两个受保护案例，因此这些是检测到的冲突，不是总发生率。Anthropic 的 [build-eval 与 hillclimb 文章](https://claude.dev/blog/automating-eval-design-and-hillclimbing/)（9 月 28 日）描述了类似规则：先确认评测噪声小于值得采纳的最小改进，每轮只测试一个补丁；如果训练集分数上升而保留集持平，或任一分数回退，就回滚该补丁。

**明确更新的对象。** 记忆、技能、运行时代码和模型权重需要不同的检查。

| 更新对象 | 代表机制 | 应检查什么 |
|:---------|:---------|:-----------|
| **可检索记忆** | [Living-Harness v2](https://arxiv.org/html/2607.26598v2)在运行经过评价后，更新记忆和状态图。后续任务检索这些记录作为指导。 | 证据、适用范围，以及后续是否检索到纠正后的指导。 |
| **技能及其关系** | [GSE](https://arxiv.org/html/2608.06153v1#S3)修改技能内容和技能间的关系，包括依赖与冲突。 | 回放涉及受影响技能的案例，再用独立案例测试。 |
| **运行时代码与工具** | [Better Harnesses, Smaller Models](https://arxiv.org/html/2607.08938v1#S3)保持执行任务的模型不变，修改工具、hooks、上下文处理和子 agent。 | 使用修改后 harness 的目标模型，其任务结果和成本。 |
| **模型权重** | [Multi-Harness RL](https://arxiv.org/html/2609.04518v1#S3)用多个 harness 产生的经验训练模型，再在未参与训练的 harness 上测试。 | 训练所用接口上的收益，以及迁移到另一接口后的表现。 |

记录哪些案例用于产生更新，哪些决定是否接受更新，哪些衡量最终表现。更新成本应包括失败候选和评价运行。

---

## 构建者资源

- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) —— 逐步构建一个小型编码智能体。
- [ultraworkers/claw-code](https://github.com/ultraworkers/claw-code) —— Rust 重新实现，可用于研究不同的实现选择。
- [Yuyz0112/claude-code-reverse](https://github.com/Yuyz0112/claude-code-reverse) —— 查看 Claude Code 与 LLM 的交互。
- [Haseeb Qureshi 的架构对比](https://gist.github.com/Haseeb-Qureshi/2213cc0487ea71d62572a645d7582518) —— Claude Code、Codex、Cline 和 OpenCode 的架构对比。
- [OpenHands](https://github.com/All-Hands-AI/OpenHands) —— 提供环境隔离的开源编码 agent 平台。
- [SWE-Agent](https://github.com/SWE-agent/SWE-agent) —— 可用于研究 agent–computer interface 的编码智能体实现。
- [Aider](https://github.com/Aider-AI/aider) —— 通过 Git 管理代码变更的编码助手。
