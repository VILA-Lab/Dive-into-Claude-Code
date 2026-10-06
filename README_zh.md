# 深入理解 Claude Code

*当下与未来 AI 智能体系统的设计空间*

<p align="center">
  <img src="./assets/main_structure.png" width="85%" alt="Claude Code 高层系统结构">
</p>

<p align="center">
  <a href="https://arxiv.org/pdf/2604.14228v2"><img src="https://img.shields.io/badge/Paper-PDF-blue.svg?logo=adobeacrobatreader&logoColor=white" alt="论文"></a>
  <a href="https://arxiv.org/abs/2604.14228"><img src="https://img.shields.io/badge/arXiv-2604.14228-b31b1b.svg" alt="arXiv"></a>
  <a href="./LICENSE"><img src="https://img.shields.io/badge/License-CC--BY--NC--SA--4.0-lightgrey.svg" alt="许可证"></a>
  <a href="https://github.com/VILA-Lab/Dive-into-Claude-Code/stargazers"><img src="https://img.shields.io/github/stars/VILA-Lab/Dive-into-Claude-Code?style=social" alt="星标数"></a>
  <a href="#参与贡献"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg" alt="欢迎 PR"></a>
</p>

<p align="center">
  <a href="./README.md">English</a> | <b>中文</b>
</p>

> **一份面向 Claude Code（v2.1.88，约 1,900 个 TypeScript 文件、约 512K 行代码）的全面源码级架构分析，附带社区分析精选、面向智能体开发者的设计空间指南，以及跨系统对比。**

> [!TIP]
> **TL;DR** —— Claude Code 代码库里只有 1.6% 是 AI 决策逻辑；其余 98.4% 都是确定性基础设施——权限门控、上下文管理、工具路由和恢复逻辑。智能体循环本身是一个简单的 while 循环；真正的工程复杂度集中在它周围的各类系统上。本仓库剖析这一架构，并为所有构建 AI 智能体系统的人提炼出可落地的设计指南。

---

## 目录

**来自我们的论文**

- [🌟 关键亮点](#关键亮点)
- [📖 阅读指南](#阅读指南)
- [🏗️ 架构总览](#架构总览)
- [🧭 价值观与设计原则](#价值观与设计原则)
- [🔄 智能体查询循环](#智能体查询循环)
- [🛡️ 安全与权限](#安全与权限)
- [🧩 可扩展性](#可扩展性)
- [🧠 上下文与记忆](#上下文与记忆)
- [👥 子智能体委托](#子智能体委托)
- [💾 会话持久化](#会话持久化)

**论文之外**

- [🛰️ Agent 设计空间的新信号](#agent-设计空间的新信号)
- [🛠️ 构建你自己的 AI 智能体：设计指南](#构建你自己的-ai-智能体设计指南)
- [⚖️ 跨系统对比：Claude Code vs OpenClaw vs Hermes-Agent](#跨系统对比claude-code-vs-openclaw-vs-hermes-agent)
- [🔎 按设计问题检索资源](#按设计问题检索资源)
- [🌐 社区项目与研究](#社区项目与研究)
- [🚀 其他值得关注的 AI 智能体项目](#其他值得关注的-ai-智能体项目)
- [🤝 参与贡献](#参与贡献)
- [🔖 引用](#引用)

---

## 关键亮点

- **98.4% 基础设施，1.6% AI** —— 智能体循环不过是一个 while 循环；真正的工程复杂度集中在权限门控、上下文管理和恢复逻辑上。
- **5 个价值观 → 13 条原则 → 实现** —— 每一条设计决策都能追溯回人类决策权威、安全、可靠性、能力和适应性。
- **分析有上限，超限转为询问批准** —— 旧版 shell 解析路径拆出超过 50 个子命令时，会返回 `ask` 决策。
- **2 个 CVE 暴露了预信任窗口** —— 扩展会在信任对话框出现**之前**就已执行。
- **横跨各层的 harness 难以被重新实现** —— 循环本身容易复制，但钩子、分类器、压缩和隔离机制则不然。

---

## 阅读指南

| 如果你是…… | 从这里开始 | 然后阅读 |
|:----------------|:-----------|:----------|
| **智能体构建者** | [构建你自己的智能体](./docs/build-your-own-agent_zh.md) | [架构深度剖析](./docs/architecture_zh.md) |
| **安全研究员** | [安全与权限](#安全与权限) | [架构：安全层](./docs/architecture_zh.md#七个独立安全层) |
| **产品经理** | [关键亮点](#关键亮点) | [价值观与原则](#价值观与设计原则) |
| **研究人员** | [完整论文 (arXiv)](https://arxiv.org/abs/2604.14228) | [社区资源](#社区项目与研究) |

`1,884 个文件` · `约 512K 行` · `v2.1.88` · `7 个安全层` · `5 个压缩阶段` · `54 个工具` · `27 个钩子事件` · `4 个扩展机制` · `7 个权限模式`

架构部分分析的是 **Claude Code v2.1.88**；设计指南和资源目录在此基础上补充了截至 **2026 年 10 月 6 日**核查的工作。顶部 PDF 链接指向 [2026 年 7 月 2 日的 arXiv v2](https://arxiv.org/abs/2604.14228v2)，[仓库中的 2026 年 4 月 PDF](./paper/Dive_into_Claude_Code.pdf)保留为历史版本。

---

<details open>
<summary><h2>架构总览</h2></summary>

Claude Code 回答了每个生产级编码智能体都必须面对的**四个设计问题**：

| 问题 | Claude Code 的答案 |
|:---------|:---------------------|
| 推理放在哪里？ | 模型负责推理，harness 负责强制执行规则。约 1.6% 是 AI，98.4% 是基础设施。 |
| 有多少个执行引擎？ | 一个 `queryLoop` 供所有入口（CLI、SDK、IDE）共用。 |
| 默认的安全姿态是什么？ | 拒绝优先：拒绝 > 询问 > 允许；最严格的规则优先。 |
| 最根本的资源约束是什么？ | 约 200K（旧模型）/ 1M（Claude 4.6 系列）的上下文窗口。各个上下文管理阶段按各自条件触发。 |

整个系统可以拆成**7 个组件**（用户 → 入口 → 智能体循环 → 权限系统 → 工具 → 状态与持久化 → 执行环境），分布在**5 个架构层**上。

<p align="center">
  <img src="./assets/layered_architecture.png" width="100%" alt="5 层子系统分解">
</p>

> [!NOTE]
> 完整的架构深度剖析——7 个安全层、9 步轮次管道、5 层压缩等——请参阅**[docs/architecture_zh.md](./docs/architecture_zh.md)**。

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>价值观与设计原则</h2></summary>

整套架构可以从**5 个人类价值观**推导到**13 条设计原则**，再落到具体实现：

| 价值观 | 核心思想 |
|:------|:----------|
| **人类决策权威** | 人类通过主体层级保持控制。当 93% 的提示批准率暴露出批准疲劳后，Anthropic 的应对是重新划分边界，而不是追加更多警告。 |
| **安全、安保、隐私** | 即使在人类警惕性下降时，系统也能守住安全底线。7 种安全机制。 |
| **可靠执行** | 按用户的本意去执行；收集—行动—验证的闭环；优雅恢复。 |
| **能力放大** | "一个 Unix 工具，而不是产品。"98.4% 是让模型跑得起来的确定性基础设施。 |
| **上下文适应性** | CLAUDE.md 层级、渐进式的可扩展性，以及随时间演变的信任轨迹。 |

<details>
<summary><b>13 条设计原则</b></summary>

| 原则 | 设计问题 |
|:----------|:----------------|
| 拒绝优先，转交人工 | 遇到没见过的操作，是放行、拦截，还是交给人来判断？ |
| 渐进式信任光谱 | 用固定权限等级，还是让用户随使用深入逐步跨越的信任光谱？ |
| 深度防御 | 一道安全边界，还是多道相互重叠的安全边界？ |
| 外部化的可编程策略 | 硬编码策略，还是带生命周期钩子的外部化配置？ |
| 上下文是稀缺资源 | 一次性截断，还是渐进式流水线？ |
| 仅追加的持久状态 | 可变状态、快照，还是仅追加的日志？ |
| 最小脚手架，最大 harness | 把投入放在脚手架上，还是放在 harness 上？ |
| 价值观优先于规则 | 刚性流程，还是带确定性护栏的语境化判断？ |
| 可组合的多机制扩展 | 单一 API，还是开销各异的分层机制？ |
| 按可逆性加权的风险评估 | 对所有操作一视同仁，还是对可逆操作放宽监管？ |
| 透明、基于文件的配置与记忆 | 用不透明的数据库和嵌入向量，还是用户可直接查看的文件？ |
| 隔离的子智能体边界 | 共享上下文与权限，还是相互隔离？ |
| 优雅恢复与韧性 | 硬失败，还是静默恢复？ |

</details>

论文还引入了**第六个评估视角**——长期能力保持——并援引证据表明：在 AI 辅助条件下工作的开发者，在理解力测试中得分低 17%。

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>智能体查询循环</h2></summary>

<p align="center">
  <img src="./assets/iteration.png" width="60%" alt="运行时轮次流程">
</p>

核心是一个 **ReAct 模式的 while 循环**：组装上下文 → 调用模型 → 分派工具 → 检查权限 → 执行 → 重复。整个循环实现为一个 `AsyncGenerator`，以流式事件的形式逐步输出。

**模型调用前**，上下文管理按顺序检查五个阶段：预算缩减 → Snip → Microcompact → Context Collapse → Auto-Compact。各阶段是否实际调整上下文，取决于功能开关、配置和触发阈值。

**每轮 9 步管道：** 设置解析 → 状态初始化 → 上下文组装 → 按条件执行的上下文管理 → 模型调用 → 工具分派 → 权限门控 → 工具执行 → 停止条件

**两条执行路径：**
- `StreamingToolExecutor` —— 工具流入时即开始执行（延迟优化）
- 后备 `runTools` —— 将工具分类为并发安全或互斥

**故障恢复：** 输出 token 上限逐级放宽（最多 3 次重试）、按需触发压缩（每轮一次）、提示过长时降级处理、流式失败后回退、主模型不可用时切换备用模型

**5 个停止条件：** 无工具调用、最大轮次、上下文溢出、钩子干预、显式中止

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>安全与权限</h2></summary>

<p align="center">
  <img src="./assets/permission.png" width="75%" alt="权限门控">
</p>

**7 个权限模式**构成渐进式信任光谱：`plan` → `default` → `acceptEdits` → `auto`（ML 分类器）→ `dontAsk` → `bypassPermissions`（+ 内部 `bubble`）。

**拒绝优先**：规则评估中，宽范围的拒绝规则优先于窄范围的允许规则。**7 个安全层**覆盖工具过滤、权限决策、钩子和沙箱。在 v2.1.88 中，会话绕过标志和计算机操作的应用白名单不会随会话恢复；这些临时状态与持久化的权限配置需要分别看待。

> [!WARNING]
> **分析的计算上限：** 为避免阻塞事件循环，旧版解析路径限制了逐个分析的子命令数量。该路径拆出超过 50 个子命令时会返回 `ask`，转入批准决策，而不会直接放行命令。

<details>
<summary><b>更多详情：授权管道、auto 模式分类器、CVE</b></summary>

**授权管道：** 预过滤（剥离被拒绝的工具）→ PreToolUse 钩子 → 拒绝优先规则评估 → 权限处理程序（4 个分支：协调器、swarm worker、推测式分类器、交互式）

**Auto 模式分类器**（`yoloClassifier.ts`）：单独发起一次 LLM 调用，使用内部/外部两套权限模板；分两个阶段，先快速过滤，再走思维链。

**预信任窗口：** 两个已修复的 CVE 根因相同——钩子和 MCP 服务器会在初始化阶段、信任对话框弹出*之前*就已执行，由此形成一个绕开拒绝优先管道、天然享有特权的攻击窗口。

</details>

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>可扩展性</h2></summary>

<p align="center">
  <img src="./assets/extensibility.png" width="85%" alt="三个注入点：组装、模型、执行">
</p>

**四种扩展机制：** Hooks、Skills、Plugins 和 MCP。上下文成本取决于实际展示给模型的指令、工具定义和返回内容。它们作用于循环中的三个位置：**assemble()**（模型看到什么）、**model()**（模型能调用哪些工具）、**execute()**（动作是否执行、如何执行）。

**工具池组装**（5 步）：基础枚举（最多 54 个工具）→ 模式过滤 → 拒绝预过滤 → MCP 集成 → 去重

**27 个钩子事件**，分为 5 类，共 4 种执行方式（shell、LLM 评估、webhook、subagent 验证器）

**插件清单**支持 10 种组件类型：命令、智能体、skills、钩子、MCP 服务器、LSP 服务器、输出样式、通道、设置、用户配置

**Skills：** SKILL.md 含 15+ 个 YAML frontmatter 字段。关键区别——SkillTool 把内容注入到当前上下文；AgentTool 另开一个隔离的上下文。

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>上下文与记忆</h2></summary>

<p align="center">
  <img src="./assets/context.png" width="95%" alt="上下文构建">
</p>

**9 个有序来源**共同构成上下文窗口。CLAUDE.md 指令以**用户上下文**的形式进入。提示词指导模型行为，权限检查和沙箱则在模型之外约束访问。在这个版本中，记忆以文件保存，内容可以查看、编辑和纳入版本管理。

**4 级 CLAUDE.md 层级：** 托管（`/etc/`）→ 用户（`~/.claude/`）→ 项目（`CLAUDE.md`、`.claude/rules/`）→ 本地（`CLAUDE.local.md`，被 gitignore 忽略）

**五个上下文管理阶段**各有触发条件：预算缩减 → Snip → Microcompact → Context Collapse（读取时投影，不破坏原记录）→ Auto-Compact（由模型生成摘要）。

**记忆检索：** 由 LLM 扫描各记忆文件的文件头，最多挑出 5 个相关文件。不使用嵌入向量，也不使用向量相似度。

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>子智能体委托</h2></summary>

<p align="center">
  <img src="./assets/subagent.png" width="90%" alt="子智能体架构">
</p>

**6 种内置类型**（Explore、Plan、General-purpose、Guide、Verification、Statusline），另可通过 `.claude/agents/*.md` 定义自定义智能体。侧链转录稿单独保存执行者的历史，简短的返回结果可以控制进入父上下文的信息量。worktree、remote 和 in-process 提供不同的执行方式；上下文分开保存，本身并不能决定文件访问范围或授权。协调使用 POSIX `flock()`。

**SkillTool vs AgentTool：** SkillTool 向当前上下文加入指导，AgentTool 把工作委派到另一个上下文。Token 成本取决于实际执行的工作和返回的结果。

**权限覆盖：** 子智能体 `permissionMode` 生效，除非父级处于 `bypassPermissions`/`acceptEdits`/`auto`（显式用户决策始终优先）。

**自定义智能体：** YAML frontmatter 支持 tools、disallowedTools、model、effort、permissionMode、mcpServers、hooks、maxTurns、skills、memory scope、background flag、isolation mode。

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>会话持久化</h2></summary>

<p align="center">
  <img src="./assets/session_compact.png" width="75%" alt="会话持久化与上下文压缩">
</p>

会话历史保存在三个通道中：仅追加的 JSONL 转录稿、全局提示历史和子智能体侧链。在 v2.1.88 中，会话绕过标志和计算机操作的应用白名单不会从旧会话恢复。持久策略、会话模式和临时授权各有自己的有效期。

**链式修补：** 压缩边界记录 `headUuid`/`anchorUuid`/`tailUuid`。会话加载器在读取时修补消息链；磁盘上的数据不会被就地改写。

**检查点：** `--rewind-files` 的文件历史检查点，存储在 `~/.claude/file-history/<sessionId>/`。

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>Agent 设计空间的新信号</h2></summary>

Claude Code 提供了一个具体起点：模型选择动作，周围的系统负责工具、上下文、权限和恢复。近期的工作进一步把这些选择放到整个任务中考察：工作如何组织，中断后保留什么，哪些经验值得复用，以及凭什么证据继续推进。

| 设计启示 | 对 Agent 构建者意味着什么 | 代表信号 |
|:---|:---|:---|
| **Graph Engineering 让关系变得明确** | 分别表达任务依赖、智能体职责和运行状态；在节点内部，智能体仍可自主选择动作。 | [Graph Engineering 综述](https://arxiv.org/abs/2608.21156)、[Agent Graph](https://github.com/context4ai/agent-graph/tree/387f80db65bf20a61bc666b4fa885200fcedad08) |
| **长程任务需要明确的运行状态** | 分别记录目标、会话、执行机器、检查点和结果交付状态；恢复时要明确由谁接续尚未完成的动作。 | [Cursor cloud agents](https://cursor.com/blog/cloud-agent-lessons)、[Google Managed Agents](https://blog.google/innovation-and-ai/technology/developers-tools/managed-agents-gemini-api/)、[Microsoft Agent 365](https://www.microsoft.com/en-us/security/blog/2026/05/01/microsoft-agent-365-now-generally-available-expands-capabilities-and-integrations/)、[Databricks Omnigent](https://www.databricks.com/blog/introducing-omnigent-meta-harness-combine-control-and-share-your-agents) |
| **上下文与记忆需要生命周期规则** | 当前工作上下文与可复用记忆有不同的生命周期；记下经验的来源、适用范围、更新规则和检索成本。 | [LangChain Context Hub](https://www.langchain.com/blog/introducing-context-hub)、[AWS AgentCore](https://aws.amazon.com/blogs/machine-learning/break-the-context-window-barrier-with-amazon-bedrock-agentcore/)、[Anthropic managed-agent memory](https://platform.claude.com/docs/en/managed-agents/memory) |
| **访问限制需要执行边界** | 权限、网络可达性、文件系统访问、凭证保管、租户隔离和沙箱，都需要在架构设计时考虑。 | [Codex Windows sandbox](https://openai.com/index/building-codex-windows-sandbox/)、[Running Codex safely](https://openai.com/index/running-codex-safely/)、[Anthropic self-hosted sandboxes](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes) |
| **工具与技能构成供应链** | 管理工具、技能和插件的来源与版本，分别约定安装策略、更新策略、服务器身份和动作批准。 | [NSA MCP security](https://www.nsa.gov/Portals/75/documents/Cybersecurity/CSI_MCP_SECURITY.pdf)、[GitHub MCP allowlists](https://github.blog/changelog/2026-04-16-copilot-cli-supports-custom-registry-based-mcp-allowlists/)、[A2A milestone](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year) |
| **人类需要控制整个运行过程** | 支持目标、计划、审批、中断、可审查的修改和人工介入；多个智能体都能写入时，还要明确各自负责什么。 | [Codex from anywhere](https://openai.com/index/work-with-codex-from-anywhere/)、[Copilot cloud agent](https://github.blog/changelog/2026-04-01-research-plan-and-code-with-copilot-cloud-agent)、[Cognition multi-agents](https://cognition.ai/blog/multi-agents-working) |
| **推进与停止也需要评估** | 除了执行单步任务的能力，还要评估何时继续、验证、求助或停止。 | [LoopArena](https://arxiv.org/abs/2608.28281) |
| **可观测性需要接入改进流程** | 从执行轨迹中提出可检验的失败假设，再用相关任务检验修改、用回归测试保护已有成功，并在新样本上确认收益。 | [LangSmith Engine](https://www.langchain.com/blog/how-we-built-langsmith-engine-our-agent-for-improving-agents)、[OpenAI agent improvement loop](https://developers.openai.com/cookbook/examples/agents_sdk/agent_improvement_loop)、[AWS AgentCore Evaluations](https://aws.amazon.com/blogs/machine-learning/build-reliable-ai-agents-with-amazon-bedrock-agentcore-evaluations/) |

[Graph Engineering 综述](https://arxiv.org/abs/2608.21156)用任务图、智能体图和运行状态图组织这些问题。这是作者提出的分析框架，也承接了已有的图式系统；它可以自然地接到指南中的推理、委派和持久化决策上。

[Codex 本地记忆](https://learn.chatgpt.com/docs/customization/memories)在启用后，将筛选过的经验带入后续会话。当前工作上下文则有另一组设计选择：[v0.153.4 保留摘要式压缩](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact.rs)，同时提供使用笔记、可检索历史和新上下文窗口的[实验性上下文管理模式](https://learn.chatgpt.com/docs/config-file/config-reference)。后者改变了长任务跨窗口继续执行的方式，与跨会话的后台经验提取是不同机制。

实验开关在 [Codex CLI 0.153.0](https://github.com/openai/codex/releases/tag/rust-v0.153.0) 中发布，默认关闭，并有账户与后端限制。9 月 6 日的一次源码更新又加入模型支持检查，将内置 GPT-6-Astra 标为支持。[来源说明](./docs/agent-design-space-source-notes_zh.md#source-codex-context-management)区分了这一后续源码修订与 v0.153.4。这是上下文管理协议及模型适配的变化；公开证据尚不能据此证明模型内部新增了某种永久记忆。

图还可以用于审查已完成的运行；多智能体通信需要规定消息何时交付；记忆需要规定召回和纠正范围；更新模型、提示或技能后，需要用明确的测试判断是否接受。具体材料、日期和版本见[9 月 15 日来源说明](./docs/agent-design-space-source-notes_zh.md#refresh-2026-09-15)。

近期的产品发布和研究提供了更多例子。在 Claude Code 中，插件现在可以在 agent 自身进程内运行代码并批准工具调用，因此组织策略、用户扩展和批准提示按什么顺序生效，也成为权限设计的一部分。Claude Code 还把停止一个回合与停止后台工作分开。重试和回滚需要规定如何避免同一动作执行两次，也需要撤销副作用的办法，包括远端系统上的更改。有些测得的 harness 差异并不大于同一配置重复运行之间的波动；固定的 LLM 裁判在评估不同版本的 agent 时，误差也可能随被评版本变化。具体来源、日期和版本见[10 月 6 日来源说明](./docs/agent-design-space-source-notes_zh.md#refresh-2026-10-06)。

具体取舍见[设计指南](./docs/build-your-own-agent_zh.md)；核查的版本、发布日期和证据限制见[来源说明](./docs/agent-design-space-source-notes_zh.md#september-2026)。

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>构建你自己的 AI 智能体：设计指南</h2></summary>

> 从 Claude Code 的架构和相关系统出发，讨论构建智能体时需要做出的设计选择。

每个生产级智能体都要面对下列决策：

| 决策 | 问题 | 关键洞察 |
|:---------|:-------------|:------------|
| [**推理放在哪里**](./docs/build-your-own-agent_zh.md#决策-1推理放在哪里) | 逻辑放在模型里多，还是 harness 里多？ | 明确哪些依赖和检查由代码表达，哪些判断交给模型。 |
| [**安全姿态**](./docs/build-your-own-agent_zh.md#决策-2你的安全姿态是什么) | 如何防止有害行为？ | 说明每种控制的作用范围，以及检查无法判定时如何处理。 |
| [**上下文管理**](./docs/build-your-own-agent_zh.md#决策-3你如何管理上下文) | 模型究竟看到什么？ | 分别管理当前对话预算与可复用经验的生命周期。 |
| [**可扩展性**](./docs/build-your-own-agent_zh.md#决策-4你如何处理可扩展性) | 扩展如何接入？ | 分别设计发现、安装、更新与执行授权。 |
| [**子智能体架构**](./docs/build-your-own-agent_zh.md#决策-5子智能体如何工作) | 共享上下文还是隔离上下文？ | 明确任务归属、上下文边界和各执行者应返回的证据。 |
| [**会话持久化**](./docs/build-your-own-agent_zh.md#决策-6会话如何持久化) | 什么会延续到下一次会话？ | 按当前策略恢复任务状态，并明确临时授权的有效期。 |

**阅读完整指南：[docs/build-your-own-agent_zh.md](./docs/build-your-own-agent_zh.md)**

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>跨系统对比：Claude Code vs OpenClaw vs Hermes-Agent</h2></summary>

同样的设计问题，换一种部署场景就会得到不同的架构答案。下表沿用论文第 10 节做 OpenClaw 对比时的六个维度，把 Claude Code v2.1.88 和两个有代表性的同类系统放在一起看：[OpenClaw](https://github.com/openclaw/openclaw)（本地优先的多渠道个人助手网关）与 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)（支持多种部署场景的自改进智能体）。每一格的内容都有出处，这不是功能打分表。

| 设计维度 | Claude Code (v2.1.88) [![Star](https://img.shields.io/github/stars/anthropics/claude-code.svg?style=social&label=Star)](https://github.com/anthropics/claude-code) | OpenClaw [![Star](https://img.shields.io/github/stars/openclaw/openclaw.svg?style=social&label=Star)](https://github.com/openclaw/openclaw) | Hermes-Agent [![Star](https://img.shields.io/github/stars/NousResearch/hermes-agent.svg?style=social&label=Star)](https://github.com/NousResearch/hermes-agent) |
|:---|:---|:---|:---|
| **系统范围与部署** | 面向编程的单用户 CLI / SDK / IDE 接口；所有入口共用一个 `queryLoop` 异步生成器。 | 本地优先 WebSocket 网关（默认端口 18789，默认仅监听回环地址）；将约 23 个消息渠道路由至内嵌智能体运行时；提供 macOS、iOS、Android 伴侣应用。 | 三个入口：`hermes`（交互式 CLI）、`hermes-agent`（程序化运行时）、`hermes-acp`（ACP 服务器）；网关适配器将消息路由至按 LRU 缓存的 AIAgent 实例（最多 128 个，空闲超时 1 小时）；也可通过 `hermes mcp serve` 作为 MCP 服务器运行。 |
| **信任模型与安全** | 拒绝优先的逐动作评估；7 种权限模式；基于 LLM 的自动模式分类器（`yoloClassifier` / `sideQuery`）；会话级权限状态（会话绕过标志、应用白名单状态）在恢复时不予还原。 | 单一可信操作员模型；DM 配对码、发件人白名单、网关身份验证；每个智能体有独立的工具允许/拒绝策略；通过 Docker / SSH / OpenShell 提供可选沙箱，默认关闭；`non-main` 模式可对非主会话启用沙箱；明确声明不支持对共享网关上的恶意多租户的隔离。 | 危险命令模式检测，配合会话级审批状态；CLI 交互提示与网关异步提示；辅助 LLM 智能审批可自动批准低风险命令；永久白名单持久化于 `config.yaml`；子智能体工作线程默认自动拒绝危险命令（可通过 `subagent_auto_approve` 开启批处理/定时任务的自动批准）。 |
| **智能体运行时与工具** | 单一 `queryLoop` 异步生成器，以流式事件方式产出；环境与特性门控的工具注册表；API 调用前按条件运行压缩（Snip、Microcompact、Context Collapse、Auto-Compact），Auto-Compact 优先尝试会话记忆压缩。 | 内嵌智能体运行时位于网关 RPC 调度层内部（`agent` RPC 校验参数后立即返回，异步执行，并通过网关协议回传生命周期/流式事件）；每个会话有独立队列序列化，支持可选的全局通道。 | While 循环，含显式的每轮迭代预算和宽限调用槽；每轮检查点去重；网关 `step_callback` 钩子在每次迭代时触发；辅助模型上下文压缩对中间轮次进行摘要，同时保护头尾。 |
| **扩展架构** | 四种扩展机制：hooks、skills、plugins 和 MCP；上下文开销取决于实际展示的内容；27 个钩子事件；10 种插件组件类型。 | 清单优先插件系统，含 12 个能力类别；中央注册表暴露工具、渠道、provider 设置、钩子、HTTP 路由、CLI 命令和服务；独立技能层支持多来源（工作区优先级最高）及 ClawHub 公共注册表；`openclaw mcp` 同时提供 MCP 服务器接口和面向其他 MCP 服务器的出站客户端注册表。 | `plugins/` 目录下内置 12 个插件（context_engine、disk-cleanup、example-dashboard、google_meet、hermes-achievements、image_gen、kanban、memory、observability、platforms、spotify、strike-freedom-cockpit）；MCP 服务器（`mcp_serve.py`）暴露 10 个工具；ACP 适配器（`acp_adapter/`）将 Hermes 暴露为 ACP 服务器。 |
| **记忆与上下文** | 四层 CLAUDE.md 层级；API 调用前压缩（Snip、Microcompact、Context Collapse、Auto-Compact）；基于 LLM 从文件型 Markdown 记忆文件中进行选择。 | 工作区启动文件（AGENTS.md、SOUL.md、TOOLS.md、IDENTITY.md、USER.md）及条件性 BOOTSTRAP.md / HEARTBEAT.md / MEMORY.md；独立记忆系统（MEMORY.md、`memory/YYYY-MM-DD.md` 格式的每日笔记、可选 DREAMS.md）；配置 embedding provider 后启用向量+关键词混合检索；实验性 dreaming 在后台整合并将符合条件的条目提升至长期记忆；可插拔压缩 provider。 | SQLite 状态存储，含 FTS5 全文检索和 WAL 模式并发读取；sessions 通过 `parent_session_id` 链接以支持压缩触发的会话拆分；`plugins/memory/` 下提供 8 个可换记忆后端（byterover、hindsight、holographic、honcho、mem0、openviking、retaindb、supermemory）；辅助 LLM 压缩作为独立的上下文管理层。 |
| **多智能体架构** | 通过侧链转录委托子智能体；6 种内置智能体定义（可用性取决于构建/模式）加自定义；父节点仅接收单条摘要消息（in-process / viewable transcript 情况下可保留更多内部细节）；隔离设置包含 `worktree` 和 `remote`，swarm 路径中有 `in-process` 队友后端。 | 两层架构。(1) 多智能体路由：每渠道独立智能体，拥有各自的工作区、认证配置、会话存储和模型配置，通过确定性绑定规则分发。(2) 子智能体委托：`maxSpawnDepth` 范围 1–5，默认 1，建议 2；工具策略按深度变化；项目愿景（VISION.md）明确拒绝将智能体层级框架作为默认架构。 | `delegate_task` 工具在 `ThreadPoolExecutor` 中派生子 AIAgent 实例（父节点阻塞直至子节点完成）；每个子节点有全新对话历史、独立 `task_id`，以及受限工具集（`DELEGATE_BLOCKED_TOOLS` 移除了 `delegate_task`、`clarify`、`memory`、`send_message`、`execute_code`）；默认深度 `MAX_DEPTH = 1`（可配置，上限为 3）；默认 3 个并发子节点。 |

**对比揭示了什么。** 有三点值得注意。第一，**部署场景**决定了后面大多数设计选择：面向单用户的编程 CLI 收敛于逐动作审批和单一执行循环，多渠道网关收敛于边界信任和渠道绑定的智能体，多部署消息云端智能体则收敛于可选容器/云隔离、LLM 智能审批和可换后端记忆层。第二，**扩展层是各系统最鲜明的差异化所在**：Claude Code 为四种机制提供不同的扩展入口，OpenClaw 将扩展视为网关层的注册表管理能力，Hermes-Agent 则内置插件组并对外暴露双 MCP 服务器 / ACP 服务器接口供其他智能体接入。第三，**记忆架构分布在一条光谱上**：一端是文件型、可直接查看的 Markdown（Claude Code），中间是文件型再加上可选的向量检索和实验性 dreaming（OpenClaw），另一端是 FTS5 全文索引加八个可换的插件后端，其中包含专用的向量 / RAG provider（Hermes-Agent）。这张表更适合读成设计空间里的三个不同落点，而不是一份功能排行榜。

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

## 按设计问题检索资源

上面各节给出的是 Claude Code 自己对每个设计问题的回答，下面的目录则是按资源类型组织的。这张表把两者接起来，让你可以从问题出发，而不是从资源类型出发。

| 设计问题 | Claude Code 的回答 | 外部资源在哪里 |
|:---|:---|:---|
| **一个回合究竟是怎么跑完的？** 智能体循环、工具派发、错误恢复。 | [智能体查询循环](#智能体查询循环) | [架构分析](#架构分析) · [开源重新实现](#开源重新实现) · [编码智能体 CLI](#编码智能体-cli-与-ide-harness) · [Harness 工程](#通用-harness-engineering-设计空间资源) |
| **谁被允许做什么？** 权限、批准、沙箱。 | [安全与权限](#安全与权限) | [安全研究与真实事件](#安全研究与真实事件) · [运行时与沙箱基础设施](#运行时与沙箱基础设施) · [产品文档](#产品文档) · [学术论文](#相关学术论文) |
| **系统怎么被扩展？** hooks、skills、插件、MCP。 | [可扩展性](#可扩展性) | [Skill 与 Harness 扩展](#skill-与-harness-扩展) · [MCP 生态](#mcp-生态) · [产品文档](#产品文档) |
| **模型到底看到了什么？** 上下文构造、记忆、压缩。 | [上下文与记忆](#上下文与记忆) | [记忆与持久化上下文](#记忆与持久化上下文) · [博客文章与技术文章](#博客文章与技术文章) · [学术论文](#相关学术论文) |
| **工作怎么分配下去？** 子智能体、团队、编排。 | [子智能体委托](#子智能体委托) | [智能体框架与编排](#智能体框架与编排) · [跨厂商工程](#跨厂商代码智能体工程) · [跨系统对比](#跨系统对比claude-code-vs-openclaw-vs-hermes-agent) |
| **重启之后什么还在？** 会话、检查点、持久化。 | [会话持久化](#会话持久化) | [运行时与沙箱基础设施](#运行时与沙箱基础设施) · [跨厂商工程](#跨厂商代码智能体工程) |
| **怎么知道它真的做对了？** 评测、基准、轨迹分析。 | [新信号：让改进形成闭环](#agent-设计空间的新信号) | [评测与基准](#评测与基准) · [学术论文](#相关学术论文) |

若某个变化横跨所有维度、而不是落在单条轴上，见 [Agent 设计空间的新信号](#agent-设计空间的新信号)。

---

<details>
<summary><h2>社区项目与研究</h2></summary>

围绕 Claude Code 架构的仓库、重新实现和学术论文的精选索引。

### 官方 Anthropic 资源

贯穿论文的主要参考资料——Anthropic 自己的工程和研究出版物，以及产品文档。

#### 研究与工程博客

| 文章 | 主题 |
|:--------|:------|
| [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) | 基础：简单的可组合模式优于重型框架。 |
| [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | 上下文策划和 token 预算管理。 |
| [Claude 5 代模型的上下文工程新规则](https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models) | Anthropic 为新一代模型删去了 Claude Code 系统提示中超过 80% 的内容，编码评测未出现可测量的下降。文章建议少写一刀切规则，让模型结合现场判断；少放工具调用范例，改进接口本身；技能和工具定义则按需加载。重点不是“提示越短越好”，而是模型变了，harness 里的旧约束也要重新审视。 |
| [Reducing cost and improving performance with Claude Platform](https://claude.com/blog/reducing-cost-and-improving-performance-with-claude-platform)、[Automating eval design and hillclimbing with Claude](https://claude.dev/blog/automating-eval-design-and-hillclimbing/) | 9 月 8 日，Anthropic 介绍了 Claude API skill 中的 prompt-audit、hillclimb 和 cost-optimize 工作流。在提供评测后，hillclimb 用训练样例指导修改，再用保留测试集检查最终配置。9 月 28 日的后续文章新增 `build-eval`，在代码库中构建评测，并检查评分器一致性和基础设施故障。hillclimb 开始前先确认评测噪声小于值得采纳的最小改进；最终增益落在该噪声范围内时，建议不合并。文中例子来自 Anthropic 自身，尚无独立复现。 |
| [A Harness for Every Task：Claude Code 的动态工作流](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code) | 介绍如何用程序编排子智能体，并归纳出 classify-and-act、fan-out-and-synthesize、adversarial verification、generate-and-filter、tournament、loop-until-done 六种模式。编排脚本负责保存中间状态和反复调度隔离的子智能体，避免把所有协调工作都挤进一个模型上下文。 |
| [Prompt Caching with Claude](https://www.anthropic.com/news/prompt-caching) | 缓存读取费用为 10%、写入为 125%；默认 TTL 5 分钟。Claude Code 整套 cache-aware 压缩策略所依赖的平台特性基础。 |
| [Harness Design for Long-Running Application Development](https://anthropic.com/engineering/harness-design-long-running-apps) | 自主全栈开发的 harness 架构；多智能体模式。 |
| [Claude Code Auto Mode: A Safer Way to Skip Permissions](https://www.anthropic.com/engineering/claude-code-auto-mode) | ML 分类器批准自动化；93% 批准率发现的来源。 |
| [Auto Mode Is Now the Default in Claude Code](https://claude.com/blog/auto-mode-default-in-claude-code) | auto mode 在 Pro、Max、Team 套餐上成为默认（2026 年 8 月 7 日宣布，8 月 14 日完成推送），而 Enterprise 与各 API 接入面（Claude API、Bedrock、Vertex、Foundry）仍为可选开启。论据来自一项 1,053 名测试者的研究：人工审查只拦下 13.6% 的注入危险命令，分类器拦下 89%；有害操作率也从人工批准会话的 6.3% 降到 2.4%。这是上一行 auto mode 工程文章的后续里程碑，如今是默认项而非一个设置。 |
| [Beyond Permission Prompts: Making Claude Code More Secure and Autonomous](https://www.anthropic.com/engineering/claude-code-sandboxing) | 基于沙箱的安全；权限提示减少 84%。 |
| [How We Contain Claude Across Products](https://www.anthropic.com/engineering/how-we-contain-claude) | 讲述 Anthropic 如何在 claude.ai、Claude Code 与 Cowork 三款产品中约束 Claude 的行为（2026 年 5 月）；内容涵盖 Claude Code 的人在回路沙箱、审批疲劳问题，以及如何在出问题时把影响范围控制到最小。 |
| [Measuring AI Agent Autonomy in Practice](https://anthropic.com/research/measuring-agent-autonomy) | 长期使用数据：随着用户熟练度提升，自动批准率从约 20% 上升到 40% 以上。 |
| [Agentic Coding and Persistent Returns to Expertise](https://www.anthropic.com/research/claude-code-expertise) | Anthropic 第一次用数据说清人机分工的实际形态：人类做约 70% 的规划决策，却只做 20% 的执行决策；用户每发一条 prompt 平均触发约 10 个 Claude 动作，某些场景下两次人工介入之间超过 100 个动作。为本文关于人类控制面与监督成本的论述提供了实测锚点。 |
| [Our Framework for Developing Safe and Trustworthy Agents](https://www.anthropic.com/news/our-framework-for-developing-safe-and-trustworthy-agents) | 负责任智能体部署的治理框架。 |
| [When AI Builds Itself](https://www.anthropic.com/institute/recursive-self-improvement) | Anthropic Institute 关于"递归自我改进"的讨论：AI 正在加速 AI 自身的研发；目前仍主要由人类把握的是研究方向的选择和对结果好坏的判断，文中还探讨了相应的治理设想。 |
| [Scaling Managed Agents: Decoupling the Brain from the Hands](https://www.anthropic.com/engineering/managed-agents) | 分离推理、执行和会话的托管服务架构。 |
| [An Update on Recent Claude Code Quality Reports](https://www.anthropic.com/engineering/april-23-postmortem) | 复盘导致质量观感下降的三个 bug：reasoning-effort 默认值、一处缓存优化 bug，以及一次系统提示改动。 |
| [Introducing Claude Opus 4.8](https://www.anthropic.com/news/claude-opus-4-8) | 2026 年 5 月模型更新：判断力与诚实度提升（代码缺陷漏判约减少 4 倍）、可自主运行更久；在 research preview 中引入 dynamic workflows。 |
| [Claude Fable 5 and Claude Mythos 5](https://www.anthropic.com/news/claude-fable-5-mythos-5) | 2026 年 6 月发布的 Mythos 级模型，能力位于 Opus 之上；其中 Fable 5 是面向通用用户的安全版本（遇到高风险请求会回退到 Opus 4.8），在软件工程与智能体编码等任务上达到当前最强水平。自 2026 年 6 月 12 日起，两者已在全球暂停访问（见下一条）。 |
| [Statement on Suspending Access to Fable 5 and Mythos 5](https://www.anthropic.com/news/fable-mythos-access) | Anthropic 关于暂停 Fable 5 与 Mythos 5 的声明。美国出口管制指令（2026 年 6 月 12 日）原本只限制外国公民访问，但 Anthropic 直接对全球所有用户停用了这两个模型，距上线只有几天。这是监管把一个已上线的前沿模型直接下架的罕见案例，也是 agent 系统在部署中要面对的合规与安全压力的一个具体例子。 |

#### 产品文档

| 文档 | 主题 |
|:---------|:------|
| [How Claude Code Works](https://code.claude.com/docs/en/how-claude-code-works) | 智能体循环、工具和终端自动化的官方概述。 |
| [Permissions](https://code.claude.com/docs/en/permissions) | 分层权限系统、模式、细粒度规则。 |
| [Hooks](https://code.claude.com/docs/en/hooks) | 27 事件钩子参考、执行模型、生命周期事件。 |
| [Mods](https://code.claude.com/docs/en/plugins/mods/overview)、[Manage mods for your organization](https://code.claude.com/docs/en/plugins/mods/admin) | 首次列于 v2.1.287（2026 年 10 月 1 日）。mods 是随插件分发的 JavaScript 或 TypeScript 函数，在 Claude Code 进程内运行，不受沙箱限制，可以观察、改写或直接应答工具调用、提示、权限检查和界面事件。mod 可以在权限提示出现前批准工具调用，但不能改变提示显示的内容。在存在托管设置或使用 Team、Enterprise 登录时，一个内置守卫先加载：deny 规则（除非管理员设置了 `allowModsToOverrideDenyRules`）和托管 `PreToolUse` hook 优先于用户的 mod；守卫读不到托管设置时会拒绝加载用户的 mod。用户的 mod 仍可批准 `ask` 规则本应提示的调用，在 auto mode 下这类调用不经过分类器。deny 规则不覆盖 mod 自身的文件和进程调用；在未信任目录中的交互式会话里，用户回答信任提示之前不加载任何 mod。 |
| [Memory](https://code.claude.com/docs/en/memory) | CLAUDE.md 层级、自动记忆、习得的偏好。 |
| [Programmatic tool calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling) | 首次 beta 发布于 2025 年 11 月。当前文档说明：Python 程序可暂停以等待客户端工具结果，再在同一容器中继续；客户端回传结果时须带上容器 ID。模型接收程序的最终输出，程序可先处理原始工具数据。 |
| [Sub-agents](https://code.claude.com/docs/en/sub-agents) | 专用隔离助手、自定义提示、工具访问。 |
| [Orchestrate Subagents at Scale with Dynamic Workflows](https://code.claude.com/docs/en/workflows) | Claude 写一段 JavaScript 编排脚本，后台 runtime 最多 16 个并发、单次运行上限 1,000 个 subagent，中间状态保存在脚本变量中、不进入 context window（v2.1.154+）。真正值得注意的是权限设计：启动时的批准提示遵循你当前的权限模式，但 workflow 派生出的 subagent 一律以 `acceptEdits` 运行并继承会话的工具白名单，与该模式无关，因此运行期间的文件编辑是自动批准的。 |
| [Configure Auto Mode](https://code.claude.com/docs/en/auto-mode-config) | 迄今最完整的一份官方说明，把 auto mode 分类器讲成一个策略引擎：四级优先级（`hard_deny` 高于 `soft_deny` 高于 `allow` 高于用户显式意图），规则写成自然语言散文而非正则。还有一处刻意的排除：分类器读 CLAUDE.md，但从不读共享的 `.claude/settings.json`，因此签入仓库的配置无法给自己授权。 |
| [Orchestrate Teams of Claude Code Sessions](https://code.claude.com/docs/en/agent-teams) | agent team 是不同于 subagent 的另一种原语：队友各自持有独立上下文，通过 mailbox 互发消息，共享一个带文件锁的任务列表（支持依赖与自主认领）。信任边界写得很明确：队友不能代替用户批准权限提示，被拒的动作也不能转交给另一个队友来绕过。 |
| [What's New in Claude Opus 4.8](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-8) | 对话中途 system 消息（保留 prompt cache）、更低的可缓存 prompt 阈值、更少 compaction 与更好的 compaction 恢复。 |
| [What's New in Claude Sonnet 5](https://platform.claude.com/docs/en/about-claude/models/whats-new-sonnet-5) | 模型收回了两个原本属于 harness 的旋钮：手动 extended thinking 被移除（`thinking: {budget_tokens: N}` 返回 400），`temperature`/`top_p`/`top_k` 设为非默认值同样返回 400。新分词器让同样的文本多出约 30% token，可用的上下文预算因此随模型漂移。 |
| [Claude Code CHANGELOG](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)、[v2.1.290](https://github.com/anthropics/claude-code/releases/tag/v2.1.290) | 发布说明。dynamic workflows 与 Opus 4.8 在 v2.1.154 落地。v2.1.178 至 v2.1.207（2026 年 6 至 7 月）：subagent 默认后台运行并带 `agent_needs_input`/`agent_completed` 钩子，且 agent 的消息永远不算用户批准；默认权限模式改名为 Manual；新增 `Tool(param:value)` 规则语法（如 `Agent(model:opus)`）；auto mode 不再读取仓库内的 `.claude/settings.local.json`；`sandbox.credentials` 阻断沙箱命令读取凭据文件与密钥环境变量。直到 v2.1.215（2026 年 7 月）：v2.1.212 为 WebSearch 调用与 subagent 派生各加每会话上限（默认均为 200，可由环境变量调整），把超过两分钟的 MCP 工具调用自动转入后台，把 `/fork` 改为开一个后台会话、其原本的会话内行为拆分为 `/subtask`，废弃 Task 工具的 `mode` 参数使 subagent 继承父会话的权限模式，并修复了 plan mode 会不经提示直接运行改动文件的 Bash 命令的问题。v2.1.214 新增 EndConversation 工具，要求带 daemon 重定向参数的 `docker` 命令必须授权，并修复了一个权限作用域缺陷：像 `Edit(src/**)` 这样的单段规则原本会自动批准对树中任意位置嵌套 `src/` 的写入，而非只限 `<cwd>/src`。v2.1.221–v2.1.224（2026 年 8 月）增加凭据文件遮蔽、跨机器 `SendMessage`/`ListAgents` 与 `claude self-hosted-runner`；跨会话入站消息须经过权限审查；同时修复 worktree、动态导入、组织策略绕过、不可见字符隐藏命令与 trailing-slash `denyRead` 绕过。凭据遮蔽还扩展到结构化提取、JWT claim 与 AWS 请求重签，沙箱违规原因会返回给模型作为恢复反馈。v2.1.225 至 v2.1.233（2026 年 8 月）把 auto mode 在 Pro、Max、Team 套餐上设为默认，把 subagent 分叉默认打开，使 `fork` 子 agent 继承父上下文与 prompt cache，用 `@name` 提及和入站接受/挂起控制让跨会话消息更完善，在较新模型上默认移除 todo 与任务追踪工具（`CLAUDE_CODE_ENABLE_TODO_TOOLS=1` 可恢复），并修复一个会泄漏 NTLM 凭据的 Windows NT 设备前缀路径校验绕过。 9 月 8–14 日的 v2.1.265–271 更新还涉及上下文和授权。默认记录稳定的提示及工具定义；压缩后向 Claude 提供当前 Git 状态。启用沙箱的 auto mode 可为 Bash/PowerShell/Monitor 设置按命令生效的 `allowed_domains`。指定的 MCP、hook 和 channel 托管配置读取失败时，仍保留相应限制。v2.1.271 的 `--accept-command` 将市场命令的批准绑定到已显示的命令、插件和市场目录。版本与具体条件见[来源说明](./docs/agent-design-space-source-notes_zh.md#refresh-2026-09-15)。2026 年 9 月 17 日至 10 月 5 日的 v2.1.275 至 v2.1.290：没有 CLAUDE.md 时读取 AGENTS.md；中和召回记忆中的不可见字符和模仿 Claude Code 自身标记的标签；项目设置不能再放宽管理员要求的沙箱；托管设置块中一个嵌套值无效时，不再忽略整个设置块；未配置权限模式的交互式会话以 auto mode 启动。send-now 键（ctrl+enter，在 Claude 仍在工作时发送排队消息）现在把正在运行的工具移到后台，而不取消当前回合；在 VS Code 中，Stop 和 Escape 只结束当前回合，后台 agent 继续运行，可在 agent 地图中逐个停止。版本与具体条件见[10 月 6 日来源说明](./docs/agent-design-space-source-notes_zh.md#refresh-2026-10-06)。 |

### 架构分析

对 Claude Code 内部设计的深度剖析。

| 仓库 | 描述 |
|:-----------|:------------|
| [**ComeOnOliver/claude-code-analysis**](https://github.com/ComeOnOliver/claude-code-analysis) [![Star](https://img.shields.io/github/stars/ComeOnOliver/claude-code-analysis.svg?style=social&label=Star)](https://github.com/ComeOnOliver/claude-code-analysis) | 全面的逆向工程：源码树结构、模块边界、工具清单和架构模式。 |
| [**alejandrobalderas/claude-code-from-source**](https://github.com/alejandrobalderas/claude-code-from-source) [![Star](https://img.shields.io/github/stars/alejandrobalderas/claude-code-from-source.svg?style=social&label=Star)](https://github.com/alejandrobalderas/claude-code-from-source) | 18 章技术书（约 400 页）。全部原创伪代码，无专有源代码。 |
| [**liuup/claude-code-analysis**](https://github.com/liuup/claude-code-analysis) [![Star](https://img.shields.io/github/stars/liuup/claude-code-analysis.svg?style=social&label=Star)](https://github.com/liuup/claude-code-analysis) | 中文深度剖析——启动流程、查询主循环、MCP 集成、多智能体架构。 |
| [**sanbuphy/claude-code-source-code**](https://github.com/sanbuphy/claude-code-source-code) [![Star](https://img.shields.io/github/stars/sanbuphy/claude-code-source-code.svg?style=social&label=Star)](https://github.com/sanbuphy/claude-code-source-code) | 四语（EN/JA/KO/ZH）分析——多领域报告，涵盖遥测、代号、KAIROS、未发布的工具。 |
| [**cablate/claude-code-research**](https://github.com/cablate/claude-code-research) [![Star](https://img.shields.io/github/stars/cablate/claude-code-research.svg?style=social&label=Star)](https://github.com/cablate/claude-code-research) | 关于内部结构、Agent SDK 和相关工具的独立研究。 |
| [**Yuyz0112/claude-code-reverse**](https://github.com/Yuyz0112/claude-code-reverse) [![Star](https://img.shields.io/github/stars/Yuyz0112/claude-code-reverse.svg?style=social&label=Star)](https://github.com/Yuyz0112/claude-code-reverse) | 可视化 Claude Code 的 LLM 交互——日志解析器和可视化工具，追踪提示、工具调用和压缩。 |
| [**Piebald-AI/claude-code-system-prompts**](https://github.com/Piebald-AI/claude-code-system-prompts) [![Star](https://img.shields.io/github/stars/Piebald-AI/claude-code-system-prompts.svg?style=social&label=Star)](https://github.com/Piebald-AI/claude-code-system-prompts) | 按版本追踪的 Claude Code 内部 prompt 语料库，逐版本从发布的分发包中抽取：主 system prompt、内置工具描述、子智能体提示（Plan/Explore/Task）、斜杠命令提示，以及 system reminder 注入；changelog 覆盖自 v2.0.14 以来每一个被追踪的版本。每次 Claude Code 发布后数分钟内更新。 |
| [**AgiFlow/claude-code-prompt-analysis**](https://github.com/AgiFlow/claude-code-prompt-analysis) [![Star](https://img.shields.io/github/stars/AgiFlow/claude-code-prompt-analysis.svg?style=social&label=Star)](https://github.com/AgiFlow/claude-code-prompt-analysis) | 跨五段对话会话抓取的 API 请求与响应日志。走的是可复现的实证路线，而不是静态读代码。 |

### 开源重新实现

净室重写和可构建的研究分支。

| 仓库 | 描述 |
|:-----------|:------------|
| [**chauncygu/collection-claude-code-source-code**](https://github.com/chauncygu/collection-claude-code-source-code) [![Star](https://img.shields.io/github/stars/chauncygu/collection-claude-code-source-code.svg?style=social&label=Star)](https://github.com/chauncygu/collection-claude-code-source-code) | 社区 Claude Code 源码产物的聚合集——包括 claw-code（Rust 移植）、nano-claude-code（Python）和提取的原始源码存档。 |
| [**777genius/claude-code-working**](https://github.com/777genius/claude-code-working) [![Star](https://img.shields.io/github/stars/777genius/claude-code-working.svg?style=social&label=Star)](https://github.com/777genius/claude-code-working) | 可运行的逆向工程 CLI。使用 Bun 运行，450+ 个 chunk 文件，polyfill 了 31 个特性标志。 |
| [**T-Lab-CUHKSZ/claude-code**](https://github.com/T-Lab-CUHKSZ/claude-code) [![Star](https://img.shields.io/github/stars/T-Lab-CUHKSZ/claude-code.svg?style=social&label=Star)](https://github.com/T-Lab-CUHKSZ/claude-code) | 香港中文大学（深圳）的可构建研究分支——从原始 TypeScript 快照重建构建系统。 |
| [**ruvnet/open-claude-code**](https://github.com/ruvnet/open-claude-code) [![Star](https://img.shields.io/github/stars/ruvnet/open-claude-code.svg?style=social&label=Star)](https://github.com/ruvnet/open-claude-code) | 夜间自动反编译重建——903+ 测试、25 个工具、4 种 MCP 传输、6 个权限模式。 |
| [**Enderfga/openclaw-claude-code**](https://github.com/Enderfga/openclaw-claude-code) [![Star](https://img.shields.io/github/stars/Enderfga/openclaw-claude-code.svg?style=social&label=Star)](https://github.com/Enderfga/openclaw-claude-code) | OpenClaw 插件——为 Claude/Codex/Gemini/Cursor 提供统一的 ISession 接口。多智能体委员会。 |
| [**memaxo/claude_code_re**](https://github.com/memaxo/claude_code_re) [![Star](https://img.shields.io/github/stars/memaxo/claude_code_re.svg?style=social&label=Star)](https://github.com/memaxo/claude_code_re) | 从压缩后的包进行逆向工程——对公开分发的 cli.js 文件进行反混淆。 |
| [**agentforce314/clawcodex**](https://github.com/agentforce314/clawcodex) [![Star](https://img.shields.io/github/stars/agentforce314/clawcodex.svg?style=social&label=Star)](https://github.com/agentforce314/clawcodex) | Python 重新实现，支持多家 LLM 厂商。 |
| [**ultraworkers/claw-code**](https://github.com/ultraworkers/claw-code) [![Star](https://img.shields.io/github/stars/ultraworkers/claw-code.svg?style=social&label=Star)](https://github.com/ultraworkers/claw-code) | 净室 Rust 重新实现，把约 51.2 万行的 TypeScript 代码压缩到大约 2 万行 Rust。正因如此，它很适合用来看清 harness 里哪些是本质、哪些是附带的。 |

### Claude Code 指南与学习

针对 Claude Code 本身的教程和动手学习路径。

| 仓库 | 描述 |
|:-----------|:------------|
| [**shareAI-lab/learn-claude-code**](https://github.com/shareAI-lab/learn-claude-code) [![Star](https://img.shields.io/github/stars/shareAI-lab/learn-claude-code.svg?style=social&label=Star)](https://github.com/shareAI-lab/learn-claude-code) | "Bash 就是你需要的一切"——19 章 0 到 1 课程，包含可运行的 Python 智能体、Web 平台。ZH/EN/JA。 |
| [**FlorianBruniaux/claude-code-ultimate-guide**](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) [![Star](https://img.shields.io/github/stars/FlorianBruniaux/claude-code-ultimate-guide.svg?style=social&label=Star)](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) | 从初学者到高级用户的指南，包含生产级模板、智能体工作流指南和速查表。 |
| [**affaan-m/everything-claude-code**](https://github.com/affaan-m/everything-claude-code) [![Star](https://img.shields.io/github/stars/affaan-m/everything-claude-code.svg?style=social&label=Star)](https://github.com/affaan-m/everything-claude-code) | 智能体 harness 优化——skills、本能、记忆、安全和研究优先开发。 |
| [**hesreallyhim/awesome-claude-code**](https://github.com/hesreallyhim/awesome-claude-code) [![Star](https://img.shields.io/github/stars/hesreallyhim/awesome-claude-code.svg?style=social&label=Star)](https://github.com/hesreallyhim/awesome-claude-code) | skills、hooks、斜杠命令、智能体编排器与插件的精选列表。 |
| [**nblintao/awesome-claude-code-postleak-insights**](https://github.com/nblintao/awesome-claude-code-postleak-insights) [![Star](https://img.shields.io/github/stars/nblintao/awesome-claude-code-postleak-insights.svg?style=social&label=Star)](https://github.com/nblintao/awesome-claude-code-postleak-insights) | 泄露后发现的精选整理，涵盖内部代号（BUDDY、KAIROS、ULTRAPLAN）、Undercover Mode，以及 AutoDream 记忆固化。 |
| [**rohitg00/awesome-claude-code-toolkit**](https://github.com/rohitg00/awesome-claude-code-toolkit) [![Star](https://img.shields.io/github/stars/rohitg00/awesome-claude-code-toolkit.svg?style=social&label=Star)](https://github.com/rohitg00/awesome-claude-code-toolkit) | 打包好的工具集，含智能体、skills、命令、插件、hooks、规则、模板与 MCP 配置。 |

### 通用 Harness Engineering 设计空间资源

与本文设计空间分析相互呼应的外部资源——概念文章、课程和代码，从工程实践视角刻画 harness 这一层。

| 仓库 | 描述 |
|:-----------|:------------|
| [**deusyu/harness-engineering**](https://github.com/deusyu/harness-engineering) [![Star](https://img.shields.io/github/stars/deusyu/harness-engineering.svg?style=social&label=Star)](https://github.com/deusyu/harness-engineering) | 学习档案——原创概念解读、独立思考与外文翻译；从概念理解到独立实践的 Harness Engineering 入门。 |
| [**walkinglabs/learn-harness-engineering**](https://github.com/walkinglabs/learn-harness-engineering) [![Star](https://img.shields.io/github/stars/walkinglabs/learn-harness-engineering.svg?style=social&label=Star)](https://github.com/walkinglabs/learn-harness-engineering) | 英文项目制课程——PDF 课本、大纲与 capstone 项目，围绕指令、状态管理、验证机制、作用域约束和会话生命周期这五个 harness 子系统展开。 |
| [**china-qijizhifeng/agentic-harness-engineering**](https://github.com/china-qijizhifeng/agentic-harness-engineering) [![Star](https://img.shields.io/github/stars/china-qijizhifeng/agentic-harness-engineering.svg?style=social&label=Star)](https://github.com/china-qijizhifeng/agentic-harness-engineering) | 自动演化 coding agent harness 的可观测系统——一个 meta-agent 读取执行 trace，自动改写 system prompt、工具、中间件、skills、子 agent 和记忆。 |
| [**ZhangHanDong/harness-engineering-from-cc-to-ai-coding**](https://github.com/ZhangHanDong/harness-engineering-from-cc-to-ai-coding) [![Star](https://img.shields.io/github/stars/ZhangHanDong/harness-engineering-from-cc-to-ai-coding.svg?style=social&label=Star)](https://github.com/ZhangHanDong/harness-engineering-from-cc-to-ai-coding) | 《马书》——把 Claude Code v2.1.88 作为 Harness Engineering 案例研究的中文 mdBook；涵盖架构、prompt engineering、上下文管理、prompt cache、安全和给构建者的经验。 |
| [**alchaincyf/loop-engineering-orange-book**](https://github.com/alchaincyf/loop-engineering-orange-book) [![Star](https://img.shields.io/github/stars/alchaincyf/loop-engineering-orange-book.svg?style=social&label=Star)](https://github.com/alchaincyf/loop-engineering-orange-book) | 花叔写的 loop engineering《橙皮书》，中英双语，讲得通俗。它把循环放在 harness 之上的一层，讲清楚一个循环要做什么、需要哪些部件，并致谢了 Steinberger、Osmani 和 Anthropic 的 Claude Code 团队。 |
| [**stanford-iris-lab/meta-harness**](https://github.com/stanford-iris-lab/meta-harness) [![Star](https://img.shields.io/github/stars/stanford-iris-lab/meta-harness.svg?style=social&label=Star)](https://github.com/stanford-iris-lab/meta-harness) | [Meta-Harness 论文](https://arxiv.org/abs/2603.28052)的官方实现。在模型固定的前提下去搜索 harness 本身（记忆、检索、上下文构造、prompt、工具选择），走 propose 到打分再到 Pareto 前沿的循环。它把 harness 从"人手工设计的东西"变成了"可以被优化的对象"。 |

### 博客文章与技术文章

| 文章 | 它的价值所在 |
|:--------|:---------------------|
| [Marco Kotrotsos — "Claude Code Internals"（15 部分系列）](https://kotrotsos.medium.com/claude-code-internals-part-1-high-level-architecture-9881c68c799f) | 源码泄露前最系统的分析。架构、智能体循环、权限、子智能体、MCP、遥测。 |
| [Alex Kim — "The Claude Code Source Leak"](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/) | 反蒸馏机制、挫折检测、Undercover Mode、每天约 25 万次浪费的 API 调用。 |
| [Haseeb Qureshi — 跨智能体架构比较](https://gist.github.com/Haseeb-Qureshi/2213cc0487ea71d62572a645d7582518) | Claude Code vs Codex vs Cline vs OpenCode——架构级比较。 |
| [George Sung — "Tracing Claude Code's LLM Traffic"](https://medium.com/@georgesung/tracing-claude-codes-llm-traffic-agentic-loop-sub-agents-tool-use-prompts-7796941806f5) | 完整系统提示和完整 API 日志。发现双模型使用（Opus + Haiku）。 |
| [Agiflow — "Reverse Engineering Prompt Augmentation"](https://agiflow.io/blog/claude-code-internals-reverse-engineering-prompt-augmentation/) | 5 种提示增强机制，由实际网络追踪支持。 |
| [Engineer's Codex — "Diving into the Source Code Leak"](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code) | 模块化系统提示、约 40 个工具、大型查询/工具子系统、反蒸馏机制。 |
| [MindStudio — "Three-Layer Memory Architecture"](https://www.mindstudio.ai/blog/claude-code-source-leak-memory-architecture) | 上下文内记忆、MEMORY.md 指针索引、CLAUDE.md 静态配置。关于记忆的最佳单一资源。 |
| [WaveSpeed — "Claude Code Architecture: Leaked Source Deep Dive"](https://wavespeed.ai/blog/posts/claude-code-architecture-leaked-source-deep-dive/) | 512K 行 TS 源码深度剖析；上下文压缩和反蒸馏。 |
| [Zain Hasan — "Inside Claude Code: An Architecture Deep Dive"](https://zainhas.github.io/blog/2026/inside-claude-code-architecture/) | 分层架构、5 种入口模式、多智能体演练。 |
| [Addy Osmani — "Agent Harness Engineering"](https://addyosmani.com/blog/agent-harness-engineering/) | 把 harness engineering 视为一门工程学科，给出命名化的原语（文件系统/git 状态、沙箱、AGENTS.md 记忆、压缩、规划循环、hooks）；将 Claude Code 作为最成熟的范例。 |
| [Addy Osmani — "Loop Engineering"](https://addyosmani.com/blog/loop-engineering/) | 从实践出发，讨论如何用自动化任务、worktree、技能、连接器、子智能体和持久进度记录，搭建持续驱动智能体的循环。适合与本文对外围 harness 的分析对照阅读。 |
| [Armin Ronacher — "The Coming Loop"](https://lucumr.pocoo.org/2026/6/23/the-coming-loop/) | 把智能体循环（一次运行里的工具调用）和 harness 循环（不停地把智能体重新叫起来再跑一遍的系统）分开来看。作者态度偏保留：循环在移植代码、调优性能、排查安全问题这些活上好用，但它写出来的代码往往偏防御、更难维护，最后还是得人去读、去决定留下哪些。 |
| [LangChain — "The Art of Loop Engineering"](https://www.langchain.com/blog/the-art-of-loop-engineering) | 讲了围绕智能体搭起来的四层循环：智能体自己的循环、给输出打分再重试的验证循环、由外部事件触发去启动智能体的事件循环，以及读生产 trace 来反过来改进 harness 的 hill-climbing 循环。核心观点是：大部分价值来自这些循环，而不是模型本身。 |
| [Lilian Weng — "Harness Engineering for Self-Improvement"](https://lilianweng.github.io/posts/2026-07-04-harness/) | 把 harness 定义为"围绕基础模型、编排执行的系统，它决定模型如何思考和规划、如何调用工具并行动、如何感知和管理上下文、如何存储产物、如何评估结果"。核心论断：近期的递归自我改进不会始于模型直接改写自己的权重，而是始于 coding agent 去演化 harness 本身。全文分设计模式、harness 优化、进化搜索、与权重联合优化四条线。 |
| [Armin Ronacher — "Better Models: Worse Tools"](https://lucumr.pocoo.org/2026/7/4/better-models-worse-tools/) | 迄今最直接的证据，说明 harness 会反过来塑造模型，而不只是被模型使用。Opus 4.8 和 Sonnet 5 会凭空造出 `requireUnique`、`oldText2` 这类任何 schema 里都没有的字段，而周围的 payload 字节完全正确。作者的解释是：这些模型是在 Claude Code 那个宽容的 harness 里训练出来的，而该 harness 会悄悄修复畸形调用（丢掉不认识的 key、接受参数别名），于是"稍微写错也照样拿到奖励"。工具 schema 并不是一层中立的抽象。 |
| [Addy Osmani — "Agentic Autonomy Levels"](https://addyosmani.com/blog/agentic-autonomy-levels/) | 把两个长期被混为一谈的维度拆开：agency（单个智能体能独立到什么程度）与 orchestration（多个智能体如何协调）。提出"执行前契约"，包含目标、范围、非目标、工具与权限、停止条件、证据要求、升级路径和预算，是对 Claude Code 那种临场弹窗批准的一种结构化替代。 |
| [Andrej Karpathy — "Sequoia Ascent 2026"](https://karpathy.bearblog.dev/sequoia-ascent-2026/) | 主张"智能体工程"：人类负责编排与验证，而不再亲自写代码。"LLM 与强化学习自动化的是你能验证的东西"；"你可以外包思考，但无法外包理解"。 |
| [Geoffrey Litt — "Understanding is the new bottleneck"](https://www.geoffreylitt.com/2026/07/02/understanding-is-the-new-bottleneck) | 当 agent 越来越会验证自己的产出，稀缺资源就从"检查正确性"变成了"人的理解"。作者主张界面应当去建立理解，而不是把 diff 摆出来了事，并提出三种做法：带内嵌讲解的 literate diff、可以逐步走查一次改动的交互式微世界，以及让团队形成共同心智模型的共享空间。这是一个关于控制界面的论点，谈的是 harness 该向人回报什么，而不是它允许 agent 做什么。 |
| [Haseeb Qureshi — "Inside the Claude Code source"](https://gist.github.com/Haseeb-Qureshi/d0dc36844c19d26303ce09b42e7188c1) | 关键模块剖析：React/Ink 界面、四种压缩策略，以及动态 prompt 边界系统。 |
| [Han HELOIR YAN — "Nobody Analyzed Its Architecture"](https://medium.com/data-science-collective/everyone-analyzed-claude-codes-features-nobody-analyzed-its-architecture-1173470ab622) | "护城河是 harness，不是模型。"本文较早把这个判断讲了出来，也正是本论文完整展开的主题。 |
| [Marco Kotrotsos — "Part 8: The Permission System"](https://kotrotsos.medium.com/claude-code-internals-part-8-the-permission-system-624bd7bb66b7) | Internals 系列中专讲权限的一篇，是泄露前对批准路径最接近实况的重建。 |
| [Vincent Qiao — "Permissions System Deep Dive"](https://blog.vincentqiao.com/en/posts/claude-code-settings-permissions/) | 从 settings 层面讲清权限规则实际怎么写、怎么解析。 |
| [ClaudeCodeCamp — "How Prompt Caching Actually Works"](https://www.claudecodecamp.com/p/how-prompt-caching-actually-works-in-claude-code) | 对 Claude Code 中缓存行为讲得最清楚的一篇实务文章，而缓存正是绝大多数压缩设计必须绕开的约束。 |
| [Gigi Sayfan — "MCP Unleashed"](https://medium.com/@the.gigi/claude-code-deep-dive-mcp-unleashed-0c7692f9c2c2) | MCP 集成路径的深入剖析：传输方式、服务器生命周期，以及外部工具如何进入工具面。 |
| [DEV Community — "Architecture via Rust Rewrite"](https://dev.to/brooks_wilson_36fbefbbae4/claude-code-architecture-explained-agent-loop-tool-system-and-permission-model-rust-rewrite-41b2) | 借一个 Rust 重新实现来读架构，把工具集拆成三层结构呈现。 |
| [Reid Barber — "Reverse Engineering Claude Code"](https://www.reidbarber.com/blog/reverse-engineering-claude-code) | 最早的一批分析之一（2025 年年中），讲 REPL 架构与最初的工具集。 |
| [Kir Shatrov — "Reverse Engineering Claude Code"](https://kirshatrov.com/posts/claude-code-internals) | 用 mitmproxy 截获 API 流量，完整报告了一次实测实验，包括 LLM 耗时与成本。 |
| [Sabrina Ramonov — "Reverse-Engineering Using Sub Agents"](https://www.sabrina.dev/p/reverse-engineering-claude-code-using) | 方法论上有意思：自建子智能体（File Splitter、Structure Analyzer），再用它们去逆向压缩后的产物。 |
| [Simon Willison — "Auto mode is now the default"](https://simonwillison.net/2026/Aug/8/auto-mode/) | 对 Anthropic 把 auto mode 设为默认这则公告的独立解读。Willison 的顾虑是：标题式的对比（分类器拦下 89% 的危险命令，疲惫的人类审查者只有 13.6%）把意外损坏与蓄意提示注入混为一谈，而后者是分类器无法可靠拦住的，所以真正稳妥的办法是收走敏感数据和高风险工具，而不是信任自动批准。它是官方资源列表中那篇厂商公告的怀疑派对照。 |
| [Birgitta Böckeler — "TDD inside the agent loop: theater or actual value?"](https://martinfowler.com/articles/exploring-gen-ai/tdd-in-the-agent-loop.html) | 用实验代替断言：让 agent 跑 TDD 大约多花三到八倍的 token，却没有比先写实现、后补测试的做法产出更好的设计，因为 agent 得不到让 TDD 生效的那种人类心理收益。作者建议用变异测试、静态分析这类结果指标来评判 agent，而不是强加一套 agent 自己感受不到的流程。 |
| ["Why does Opus 5 feel worse to work with?"](https://mun-logadan.github.io/why-does-opus-5-feel-worse/) | 把一个与 harness 相关的问题定位到基准激励上：为跑分优化会奖励那种在歧义下做出大胆而通常正确的猜测的 agent，而不是去问一句澄清，这恰好和需要谨慎、歧义又高的真实工作相反。一篇被广泛讨论的文章，谈的是 agent 循环设计里自主推进与回头确认之间的取舍。 |
| [Florian Herrengt — "AI is removing the middle class of software engineering?"](https://blog.florianherrengt.com/ai-removing-middle-class-software-engineering.html) | AI 抬高了写代码的速度上限，却没抬高读懂代码的上限，于是审查负担成为约束瓶颈，并压到那些仍能判断代码好坏的工程师身上。这是对上面几篇 loop engineering 文章里同一个论点（审查才是瓶颈）的劳动力市场视角。 |
| [Steve Yegge — "The Shape of Things to Come"](https://yegge.ai/essays/the-shape-of-things-to-come/) | 认为 agent harness 架构正收敛到循环、图和 issue 追踪器上；顺序式的 CI/CD 在智能体规模下撑不住（他提出用一种批量、蜂群诊断的模型取而代之）；以及自建的内部 harness 会胜过可复用的第三方框架，因为 harness 必须贴着代码库来做。一份来自实践者、关于 harness 层走向的地图。 |

### 跨厂商代码智能体工程

其他正在构建代码智能体的厂商的官方工程博客——有助于了解 Claude Code 之外的厂商如何回答同样的设计问题。

| 资源 | 厂商 | 亮点 |
|:---------|:-------|:---------------|
| [Codex v0.153.4](https://github.com/openai/codex/releases/tag/rust-v0.153.4) | OpenAI | 以固定版本对照摘要式压缩、实验性上下文窗口管理和跨会话记忆。[来源说明](./docs/agent-design-space-source-notes_zh.md#source-codex-memory)区分各自的启用条件与生命周期，并单列后续加入的模型能力检查。 |
| [Deep Agents vs LangChain vs LangGraph](https://www.langchain.com/blog/deep-agents-vs-langchain-vs-langgraph) | LangChain | 说明自家 harness、框架与运行时如何组合：智能体循环可以运行在图中，自定义图也可作为子智能体使用。发表于 2026 年 8 月 6 日，是理解指南中“推理放在哪里”的一份较早资料。 |
| [Cloud Agents and Cursor Harness Improvements](https://cursor.com/changelog/08-19-26)、[Self-hosted machines](https://cursor.com/changelog/self-hosted-machines)、[Projects](https://cursor.com/changelog/projects) | Cursor | 8 月 19 日更新加入事件订阅、长期目标和在独立机器上运行的子智能体；9 月 2 日加入命名 worker 池调度。这些机制说明目标、会话和执行机器可以有不同的生命周期；发布说明尚不足以证明事件或结果能恰好交付一次。 9 月 10 日推出 beta 的 Projects 增加云端协调器：规划工作、委派实现，再交给用户审查。项目在云端与本地智能体之间同步一组文件。 |
| [Introducing the Agents API](https://openai.com/index/introducing-the-agents-api/)、[Agents API 计算机使用](https://developers.openai.com/api/docs/guides/agents-api/tools/computer-use) | OpenAI | 9 月 10 日公开 beta 将 Codex harness 作为托管服务提供。应用提供工具并选择执行环境，OpenAI 运行智能体循环，管理会话、上下文和恢复。9 月 29 日加入计算机使用，浏览器运行在 OpenAI 托管的环境中。每个新的网站来源都需要用户批准，这与环境的网络策略相互独立；来源批准也不代表对单个操作的确认。登录信息通过专用事件提交，不进入模型输入和会话历史；只有主智能体能请求登录，子智能体不能。 |
| [Mastra Factory beta](https://mastra.ai/blog/announcing-mastra-factory-beta) | Mastra | 9 月 8 日进入 beta。当前文档分别定义阶段转移、进入和离开阶段时的动作，以及转移审批。PR 审查和原任务完成也分别记录。 |
| [Rethinking skills and prompts for GPT-6 Astra](https://developers.openai.com/blog/rethinking-skills-and-prompts-for-gpt-6-astra) | OpenAI | 9 月 11 日的指导针对 GPT-6 Astra，建议使用简短的技能描述、按任务指定的文档引用和明确的完成条件。技能过多时，Codex 可能截短模型看到的描述；旧指令也可能增加无用工作或导致过早停止。 |
| [托管插件市场更新](https://github.blog/changelog/2026-08-26-enterprise-managed-settings-now-support-autoupdate-for-plugin-marketplaces/)、[app 与 CLI 的内容排除](https://github.blog/changelog/2026-09-02-content-exclusions-generally-available-in-copilot-app-and-cli/)、[Managed operation permissions](https://github.blog/changelog/2026-09-09-enterprise-managed-permissions-for-github-copilot-agent-operations/) | GitHub | 分别增加插件市场更新控制，以及面向 Business/Enterprise app 与 CLI 用户的上下文排除。两者管理不同环节；内容排除受客户端和输入路径限制，不能等同于通用文件系统沙箱。 9 月 9 日正式可用的托管操作权限覆盖 Business/Enterprise app、CLI 和 VS Code Agent Host 会话。规则采用 deny > ask > allow，已声明的允许列表取交集；托管 ask 每次都需要新批准。 |
| [本地沙箱](https://github.blog/changelog/2026-09-23-local-sandboxing-in-the-github-copilot-app/)、[默认启用策略](https://github.blog/changelog/2026-09-24-default-enablement-of-copilot-features-for-copilot-business-and-enterprise/)、[Computer use](https://github.blog/changelog/2026-10-01-github-copilot-can-now-interact-with-desktop-apps/) | GitHub | 9 月 23 日起公开预览：Copilot app 可以按项目为文件、网络以及 Git 或 GitHub CLI 凭据启用沙箱。企业设置可以使实际策略更严格；如果操作系统无法执行该策略，沙箱 shell 会报错，而不是在无沙箱的情况下运行。9 月 24 日发布、面向 Copilot Business 和 Enterprise 的策略于 10 月 22 日生效：管理员可为未配置的合格 GA 功能（包括现有和未来的功能，以及 MCP 服务器策略）选择默认值，即开启、关闭或交由组织决定；已明确作出的选择保持不变。computer use（10 月 1 日起公开预览）在控制每个应用前请求批准，并保留可查看的“始终允许”列表。 |
| [autofix 使用 Copilot Memory](https://github.blog/changelog/2026-09-25-agentic-autofix-now-uses-copilot-memory)、[Copilot Memory 文档](https://docs.github.com/en/copilot/concepts/agents/copilot-memory) | GitHub | 自 9 月 25 日起（公开预览），agentic autofix 修复安全告警时会读取 Copilot Memory，并把修复模式写为记忆，供其他 Copilot 功能使用。仓库级事实附有指向支撑代码的引用，使用前对照当前分支校验，只由具有写权限的用户产生，且只用于该仓库。 |
| [Rollouts](https://cursor.com/changelog/rollouts-and-security-reviewer) | Cursor | 9 月 23 日面向 Teams 和 Enterprise 推出，把验收延伸到合并之后。PR 打开时，它发布一份作者可以编辑的监控计划，列出风险、预期效果、要检查的信号和缺失的监测。每次部署后，它按环境分别给出结论：健康、回归或无法判断。发现回归时，视配置而定，它可以开一个待审查的 revert PR，或把结果交给智能体处理，但不会自行合并或回滚。 |
| [Improved token efficiency for longer agent runs](https://cursor.com/blog/improved-token-efficiency) | Cursor | 9 月 23 日：Cursor 报告称，harness 改动使用户 token 成本降低 7%，且未降低智能体质量；文章称其借助面向用户群的 A/B 测试来调整这些改动。随着模型变强，它删去约 66% 的系统提示，按需加载不常用的内置工具，在每个请求中很少变化的部分之后设置缓存断点，并删除推动使用子智能体的指令，因为较新的模型已在训练中学会这种做法。数字由厂商报告，评测细节未公开。 |
| [Dynamic workflows in Copilot CLI and the Copilot app](https://github.blog/changelog/2026-10-01-dynamic-workflows-in-copilot-cli-and-the-copilot-app)、[概念说明](https://docs.github.com/en/copilot/concepts/agents/dynamic-workflows) | GitHub | 10 月 1 日起在 Copilot CLI、Copilot app 和 Copilot SDK 中公开预览。workflow 是一段代码，规定步骤、何时调用 agent 以及如何使用其结果。上限包括并发 agent 数、agent 总数、活跃运行时间和近似 AI credits。达到并发上限时新 agent 会等待；其他上限会停止运行，但保留状态和已保存结果以便恢复，提高后的上限仍计入停止前的用量。恢复时可复用已完成步骤和子 agent 的已保存结果，未保存的工作可能需要重跑。workflow 的子 agent 继承会话的权限授予，因此为某个子 agent 作出的会话期授权也适用于其他子 agent。 |
| [Harness Engineering: Leveraging Codex in an Agent-First World](https://openai.com/index/harness-engineering/) | OpenAI | 把 harness 定义为让智能体产出可靠、可维护结果所需的约束、反馈回路、文档结构与工具；文中提到，他们一款约百万行代码的 beta 版产品几乎没有一行是人工编写的。 |
| [Codex 0.147.0](https://github.com/openai/codex/releases/tag/rust-v0.147.0) | OpenAI | 插件目录现在可以汇总本地、个人、工作区和远端来源，并新增 `--approve-for-me`、MCP 2026-07-28 的可选实现、远端压缩、对话分节，以及从 Cursor 导入技能、从 Claude 和 Cursor 导入对话。安全方面，首次进入陌生的本地项目需要显式确认，历史和回放中的 bearer token 会被遮蔽；插件策略更新失败时，网络访问默认关闭。Codex [0.157.0](https://github.com/openai/codex/releases/tag/rust-v0.157.0)（9 月 25 日）把网络限制扩展到重定向以及持续进行的 HTTP 和 WebSocket 流量，策略变更撤销访问权限时会取消已有连接。Codex [0.156.0](https://github.com/openai/codex/releases/tag/rust-v0.156.0)（9 月 22 日）在每次模型请求前重新加载全局指令，因此修改全局 AGENTS.md 会在运行中的会话里生效；仓库指令只在环境或信任级别变化时重新加载。 |
| [Deep Agents v0.7](https://www.langchain.com/blog/deep-agents-v0-7) | LangChain | 删除基础系统提示和默认待办中间件，把内置工具描述缩短 43%，首轮基础输入从约 6K 降到 2K token；报告中的多模型评测没有显示整体得分明显下降。文章还说明了待办规划适合哪三类任务，并开放摘要阈值、搜索截断和文件分页等控制项。与上面的 Anthropic 文章相比，这项工作还给出了跨模型评测。 |
| [Best Practices for Coding with Agents](https://cursor.com/blog/agent-best-practices) | Cursor | 将智能体 harness 拆为三部分——Instructions、Tools、Model——并按所用模型分别编排。 |
| [Build with Google Antigravity](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/) | Google | 以智能体为先的平台：通过 Manager 界面异步编排多个智能体，并用 Artifacts（计划、截图、录屏）代替原始日志来做验证。 |
| [Microsoft Agent Framework at BUILD 2026: Agent Harness, Hosted Agents, CodeAct](https://devblogs.microsoft.com/agent-framework/microsoft-agent-framework-at-build-2026-announce/) | Microsoft | 内置了一个 "agent harness"：自动压缩上下文、文件式记忆、plan/execute 两种模式、skill 发现、并行子智能体，还有一个沙箱 shell。另外加了 CodeAct——让模型把多次工具调用写成一段 Python 程序，在每次调用单独开的 Hyperlight 微型虚拟机里跑。 |
| [Codex Security: Now in Research Preview](https://openai.com/index/codex-security-now-in-research-preview/) | OpenAI | 应用安全智能体：先为项目构建专属威胁模型，再在沙箱验证环境中查找并压力测试漏洞。此后它已作为 `@openai/codex-security` CLI 与 TypeScript SDK 发布（首次发布于 2026 年 7 月 28 日），加入了深扫工作者、把安全发现发布到 Linear、以及 CI 集成，需通过一层已验证访问的门槛。 |
| [Meet Your Agent Harness and Claw](https://devblogs.microsoft.com/agent-framework/meet-your-agent-harness-and-claw/)、[Working with Your Data, Safely](https://devblogs.microsoft.com/agent-framework/agent-harness-working-with-your-data-safely/)、[Scaling Harness Capabilities](https://devblogs.microsoft.com/agent-framework/agent-harness-scaling-the-claw-or-harness-capabilities/) | Microsoft | 一个四篇的 harness 系列。授权设计值得注意：standing rule（"总是批准这个工具"、"总是批准这组参数"）只在当前 session 内有效，绝不会固化进 agent，这与 Claude Code 的权限模型是直接可比的一点。能力扩展被拆成四个正交机制：Skills、受限 Shell、CodeAct、Background Agents。 |
| [Gemini CLI 转向 Antigravity CLI](https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/) 与 [antigravity-cli](https://github.com/google-antigravity/antigravity-cli) | Google | Gemini CLI 于 2026-06-18 停服，由一个 Go 重写版取代，后者与 Antigravity 2.0 桌面端共用同一个 agent harness。据其 CHANGELOG（该仓库只放发布产物与示例，不含源码），机制与 Claude Code 相当接近：嵌套 subagent 可到孙级及更深，其子轨迹更新会被递归回传到根对话；工作区级的 `.agents/hooks.json`，由 pre-tool hook 决定某次工具调用是否放行；以及按项目的权限配置，存放在 `~/.gemini/config/projects/` 而非仓库内，优先级高于全局设置。 |
| [Governing Agent Autonomy with Auto-review](https://cursor.com/blog/agent-autonomy-auto-review) | Cursor | 分类器跑在 agent loop 内部，而不是作为独立端点。它拦下一个动作时，会把解释返回给父 agent，父 agent 往往能据此绕道走一条安全路径，全程不打扰用户。约 4% 的动作被拦下，但只有约 7% 的对话真的中断了人。拦截在这里是引导信号，而不只是终止。 |
| [Customize Cursor](https://cursor.com/changelog/customize) | Cursor | plugins、skills、MCPs、subagents、rules、commands、hooks 被收进同一个扩展管理界面，并支持 user、team、workspace 三级作用域。一个非 Anthropic 厂商收敛到了几乎与 Claude Code 相同的扩展点上。 |
| [Codex-maxxing for Long-Running Work](https://openai.com/index/codex-maxxing-long-running-work/) | OpenAI | memory vault 把 `AGENTS.md`、`TODO.md`、`projects/`、`people/` 放进 GitHub，于是 diff 本身成了记忆的审查界面。文中把设计立场讲得很直白：记忆必须可打开、可编辑、可 diff、可复用。 |
| [Devin Fusion](https://cognition.com/blog/devin-fusion) 与 [Agentic MapReduce](https://devin.ai/blog/agentic-map-reduce/)、[Fusion on Desktop and CLI](https://cognition.com/blog/local-fusion) | Cognition | Fusion 把两个通常各管各的机制耦合起来：一个轻量分类器在会话中途重新评估任务难度，而模型切换被特意安排在 context compaction 的时刻，以避免切换额外导致一次 cache miss。MapReduce 则在智能体与确定性计算之间划了一条清晰的线：只在需要推理的地方放智能体，其余一律确定性。 9 月 11 日，Fusion 进入 Devin Desktop 和 CLI。该设计说明按模型组合调整任务简报细度、worker 能否质疑指令，以及探索分工。 |
| [AI SDK 7](https://vercel.com/blog/ai-sdk-7) | Vercel | HarnessAgent 把 Claude Code、Codex、Pi 当成同一套 API 背后可互换的后端，会话可以挂起再恢复。WorkflowAgent 把每次工具调用做成可持久化、可重试的步骤，进程重启后仍能接着跑。工具审批用 HMAC 签名，回应了一个很少被讨论的问题：批准本身可能被伪造。 |
| [MCP 2026-07-28 正式规范](https://modelcontextprotocol.io/specification/2026-07-28/changelog) 与 [Enterprise-Managed Authorization](https://blog.modelcontextprotocol.io/posts/enterprise-managed-auth/) | MCP | 取消协议级会话与 initialize 握手，能力发现改用 server/discover。Multi Round Trip Requests 使一次逻辑工具操作能请求输入，并跨多个独立请求继续；应用状态和续接状态仍需要明确的管理方。Roots、sampling 和 logging 被弃用，Tasks 移到扩展。企业托管授权另外提供组织层控制。 |
| [Droid Shield 2.0](https://factory.ai/news/droid-shield-2-0) | Factory | 一个确定性正则扫描器，两侧各挂一个微调模型：扫描器没响时，一个模型以召回优先重新读上下文；扫描器响了时，另一个模型先把候选密钥遮住，仅凭上下文判断是不是误报。确定性规则与学习型模型被接成了一道双向纠错的闸门。 |
| [Software Is Made Between Commits（DeltaDB）](https://zed.dev/blog/introducing-deltadb) | Zed | 为智能体会话重做的版本控制。一条消息和它产生的编辑被并排存放，二者不会各自漂走；每个引用锚定到 delta 而非行号，因此代码在底下移动时引用依然存活。核心论点是：git 是围绕离散 commit 组织的，它从来就不是为了承载"产生这段代码的那场对话"而设计的。 |
| [Responses API 中的多智能体编排](https://developers.openai.com/api/docs/guides/responses-multi-agent) | OpenAI | subagent 编排从客户端代码搬进了 API 本身。根 agent 派生出一棵具名的树（`/root/researcher`），由六个托管动作协调（`spawn_agent`、`send_message`、`followup_task`、`wait_agent`、`interrupt_agent`、`list_agents`），并用 `max_concurrent_subagents` 限制整棵树上可同时活跃的 subagent 数量（各级后代都计入，根 agent 不计）。代价写得很直白：开启多智能体时，reasoning summary 与 `max_tool_calls` 都不可用。 |
| [Programmatic Tool Calling](https://developers.openai.com/api/docs/guides/tools-programmatic-tool-calling) | OpenAI | 由模型写一段 JavaScript，通过 `tools.*` 命名空间去调用工具，跑在一个全新隔离的 V8 里，没有文件系统、没有网络，程序之间也不保留任何状态。工具通过 `allowed_callers` 选择加入，因此一个工具可以只允许程序调用、只允许模型直接调用，或两者都允许。批处理与过滤都发生在程序内部，中间的工具输出因此不进入上下文窗口。 |
| [Copilot CLI changelog](https://github.com/github/copilot-cli/blob/main/changelog.md) | GitHub | 7 月的三处改动值得与 Claude Code 的权限模型对照：一个 auto allow-all 模式，凡被 LLM 裁判判为可接受的请求即自动放行；受信任的仓库可以通过 `.github/copilot/settings.json` 钉死模型、effort 等级与 context tier，并扩充 URL、MCP、skill 的拒绝名单；`preToolUse` 钩子以退出码 2 直接拒绝某次工具调用。plan mode 会硬性拦截所有会修改工作区的内置工具，但 MCP 与外部工具依然放行，这个边界比"计划模式"这个名字听上去要窄。 |
| [我们为什么做了 ADK 2.0](https://developers.googleblog.com/en/why-we-built-adk-20/)、[ADK Python 2.11.0](https://github.com/google/adk-python/releases/tag/v2.11.0)、[2.9.0](https://github.com/google/adk-python/releases/tag/v2.9.0) | Google | Workflows 把编排变成一张有向图，节点间的跳转由代码判定而不是交给模型选择，于是工具调用、human-in-the-loop 批准这类确定性步骤，可以和开放式的 agent 步骤放进同一张图。上下文上的取舍是刻意的：引擎只把必要的数据子集传给下游节点，"把它们与冗长、无关的执行历史隔开"，而不是让子 agent 继承整条轨迹。ADK Python 2.11.0（2026 年 10 月 1 日）允许用中止信号优雅地停止 Runner、工作流或单个节点，并让工作流中的工具节点暂停等待用户批准，而不是把错误传给下游。自 2.9.0（9 月 10 日）起，失败的节点会在工作流恢复时重新运行，而不是被当作已完成重放，因此发布说明要求节点体保持幂等。 |
| [Harness Optimizer 发布](https://strandsagents.com/blog/introducing-harness-optimizer/) | AWS | 把 harness 当成一组可调参数，而不是手写出来的东西：system prompt、工具描述与 skills 构成一个 `Formula`，`RewardFunction` 给执行轨迹打分，`Trainer` 按 epoch 跑 rollout、算 reward、更新参数。默认优化器本身就是一个 LLM，它对照着读成功与失败的轨迹，找出赢的那些做了什么而输的没做，再把差异改写回 Formula。开源，并以 AgentCore Optimization 的形式提供生产版本。 |
| [The Self-Driving Company](https://replit.com/blog/self-driving-company) | Replit | 难得一见的厂商内部用法披露，而不是产品宣传。每位员工都有一个 manager agent，可以在公司自己的 agent harness、microVM 与远程文件系统上派生并编排更多 agent，整套东西被放在"访问策略、token 代理、审计日志与我们的 ZeroTrust 网络之后"。人的升级路径是设计进去的，而不是兜底：代码评审 agent 会评估风险等级，只在必要时才叫第二个人类评审进来。 |
| [Amp 更新](https://ampcode.com/news) | Sourcegraph | 7 月的两条更新指向同一个方向。agent 现在可以派生其他 agent、给它们发消息，并跨 thread 交换文件，这是点对点协作，而不是父 agent 收集子 agent 的汇报。另一条是 Orb 远程沙箱改用 OIDC 工作负载身份，替换掉远程执行 agent 所用的静态凭证。 |
| [Agent Plugins 1.0](https://agent-plugins.org/specification) ([发布](https://github.blog/changelog/2026-08-12-agent-plugins-1-0-in-vs-code-copilot-cli-and-the-copilot-app/)) | Cross-vendor | 2026 年 8 月发布的可移植插件打包格式：一个 `plugin.json` 清单把 Agent Skills（一个 `skills/` 目录）和 MCP 服务器（`mcp.json`）打进同一个目录，可跨客户端安装。其初始技术指导委员会（TSC）列出的核心维护者来自 Amazon、Cursor、Microsoft、OpenAI 与 Vercel，首日即在 VS Code、Copilot CLI 与 Copilot app 上提供支持。它在扩展层扮演了 MCP 之于工具那样的角色：本文分析的两个面（skills 与 MCP）拿到了一套跨厂商共享的分发格式。 |
| [One Agent, Every Surface: How We Built the Kiro Agent Harness](https://kiro.dev/blog/one-agent/) | AWS | 把三套彼此独立的 agent 实现（IDE、CLI、Web）合并成单一独立的 harness 进程，经由 Agent Client Protocol 寻址，并用一套基于能力（capability）的模型替换三套互不兼容的权限系统，其背后是经过形式化验证的策略语言 Cedar。上下文管理与压缩在各接入面上变得统一。这是一家厂商用一个可验证的策略引擎、而非手写规则来回答权限模型问题。 |
| [Introducing Delta](https://zed.dev/blog/introducing-delta) | Zed | 一个用于与 agent 多人协作编码的独立应用，而不是加进编辑器的一个功能。它建在 DeltaDB（见上文）之上，把对话与 worktree 实时复制给同一 thread 里的每个人，评论锚定在代码上，即使代码在其下方变动也不脱节。它把 thread、而非文件，放到会话架构的中心。 |
| [Global Plugins and Skills](https://ampcode.com/news/global-plugins-and-skills) | Sourcegraph | 把插件与技能管理集中到两个作用域：供个人试验、带热重载的个人层，以及由管理员管理、为团队设定默认的工作区层，两者之间可以共享。扩展分发成为一个受治理、以团队为范围的面，而不是散落的按仓库文件。 |
| [How to Build a Cloud Software Factory: Computer Use Verification](https://www.warp.dev/blog/how-to-build-a-cloud-software-factory-computer-use-verification) | Warp | 一个 verify-behavior 技能让 agent 用鼠标和键盘操作运行中的应用，按规格核对自己的工作，而不只是查看代码。同一个技能既服务于复现缺陷的排查 agent、验证功能的实现 agent，也服务于为人类产出截图和录像证据的审查 agent，且全部在云侧进行，不动开发者的机器。自我验证从看代码升级为行为层面的证明。 |

### 运行时与沙箱基础设施

智能体的动作究竟在哪里、以何种方式执行：沙箱、托管运行时、控制面，以及包在它们外面的会话协议。与 Claude Code 自己的 shell 沙箱和执行边界正好构成对照。

| 资源 | 厂商 | 亮点 |
|:---------|:-------|:---------------|
| [Temporal Deep Agents 集成](https://docs.temporal.io/develop/python/integrations/deepagents)、[Workflow Pause](https://docs.temporal.io/encyclopedia/workflow/workflow-pause) | Temporal | [8 月 27 日月报](https://temporal.io/blog/durable-digest-august-2026)将两者列为预发布功能。集成把控制循环放进可重放的 Workflow，把模型调用放进 Activity；有真实 I/O 的工具和后端需要显式包装。暂停会停止新任务派发，在途 Activity 仍可能完成。 |
| [Bedrock AgentCore Harness 正式可用](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-harness-is-now-generally-available-go-from-idea-to-production-grade-agent-in-minutes/) | AWS | harness 不再是你自己写的一个循环，而变成了托管的配置对象：`CreateHarness` 和 `InvokeHarness` 声明模型、工具、skills、记忆策略和容器环境，底层封装七个原语（microVM Runtime、Memory、Gateway、沙箱 Browser、Code Interpreter、Identity token vault、Observability）。这给设计空间加了一个新坐标：loop、环境和工具边界到底归谁所有。 |
| [AgentCore policy 与 Guardrails](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-bedrock-agentcore-policy-guardrails-generally-available/) | AWS | 强制执行发生在 gateway 边界，也就是 agent 代码之外，因此无论 agent 变得多自主都照样生效。这与"在环内做权限检查"是截然不同的架构选择。 |
| [Running Untrusted Agent Code Without a Sandbox](https://www.langchain.com/blog/running-untrusted-agent-code-without-a-sandbox) 与 [Dynamic Subagents in Deep Agents](https://www.langchain.com/blog/introducing-dynamic-subagents-in-deep-agents) | LangChain | 与容器模型完全反过来的能力隔离：WASM 里的 QuickJS 从"零能力"起步，每一项能力都由 harness 显式桥接进来。解释器内存可以被序列化，从而实现一次等待人工批准的持久暂停。子智能体的派发随之从"对话回合"降到"程序控制流"，由模型写一段脚本去调用 `task({...})`。 |
| [A Single Pane of Glass for All Your Cloud Agents](https://www.warp.dev/blog/multi-harness-cloud-agent-orchestration) | Warp | Oz 作为 multi-harness 控制面：在同一个界面里启动、追踪、治理和引导 Claude Code、Codex 与 Warp Agent，比较它们的效果，为不同任务挑不同的 harness，并保持一致的审计链路。Agent Memory 可以跨 harness 迁移。 |
| [Warp Agent CLI](https://www.warp.dev/blog/introducing-the-warp-agent-cli-coding-agent) | Warp | 底层采用 PTY multiplexer，而不是只封装逐行执行的 shell，因此会话可以跨目录切换和 SSH 连接继续运行，也能操作调试器、编辑器等交互式程序。子智能体可以把任务交给 Warp、Claude Code 或 Codex，本地会话还可以转到云端继续监控和调整。这里的关键是把 PTY 管理纳入 harness 运行时，而不只是向模型提供一个 shell 工具。 |
| [Agent Host 与 Agent Host Protocol](https://code.visualstudio.com/updates/v1_129)、[VS Code 1.137](https://code.visualstudio.com/updates/v1_137)、[VS Code 1.140](https://code.visualstudio.com/updates/v1_140) | Microsoft | VS Code 1.129 把 agent 会话从编辑器里搬进一个独立进程，通过开放的 Agent Host Protocol 通信。因为会话活在自己的进程里，它在没有任何客户端连接时也继续运行，并且可以同时被多个窗口渲染；同一个 host 用一套会话模型跑 Copilot、Claude 和 Codex。跑在 host 上的 agent 可以列出其他会话、读取它们的对话记录、新开一个会话来移交子任务，以及给另一个会话发消息，其中发送需要用户确认，并有突发上限防止一次请求扇出成无限多个会话。 1.137 将智能体发给忙碌 chat 的消息排队，当前回合成功结束后，再按发送顺序开始处理排队消息。Automations 处于 Preview，要求机器未休眠且相应宿主仍在运行；禁用计划不会停止当前运行。1.140（9 月 30 日）新增实验性工具，把工作委派给已连接的远端 agent host，可按名称或按操作系统、内存和 CPU 要求选择 host。远端会话除非指定，否则没有工作区，常规审批仍然适用。远端 agent 通过 `send_remote_message` 回报，其最终答复不会自动转发。达到编排上限时会阻止新的编排动作，但不会打断已经在运行的工作。 |
| [Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes) | Google | 9 月 9 日，Computer Use 和 Shell 沙箱正式可用。Shell 调用各自启动新的 shell。另一项独立功能是 Agent Gateway 的 VPC Service Controls 支持：它要求 ALL_TRAFFIC，以及 9 月 8 日之后创建、使用 agent connectivity template 的网关。 |
| [沙箱分叉](https://github.com/e2b-dev/E2B/releases) | E2B | `Sandbox.fork()` 会就地为运行中的沙箱打检查点：短暂暂停、连同完整内存状态一起快照、再恢复，其 ID 与过期时间都保持不变，然后从该快照拉起 N 个彼此独立的新沙箱。这让 harness 能以很低的成本把一个配置代价很高的环境分叉到多路并行尝试上，而不必为每一路重建。 |
| [Docker VMM Public Beta](https://www.docker.com/blog/docker-vmm-public-beta/) | Docker | Docker 自研的第一个虚拟机管理器，替换 Docker Desktop 底层的第三方 libkrun。同一引擎也驱动 Docker Sandboxes，即用于 agent 执行的 microVM 隔离层，因此每一处改进两边都受益。号称容器与沙箱启动更快、主机与来宾之间的文件 I/O 更好，并在 Windows 上提供 Hyper-V 级隔离。这是一款被广泛使用的 agent 沙箱底下的隔离基座。 |
| [Sandboxing](https://zed.dev/blog/sandboxing) | Zed | agent 的终端与 fetch 工具默认被沙箱化，使用操作系统级原语（macOS 的 Seatbelt、Linux 的 bwrap 与命名空间、Windows 的 WSL），阻断对项目之外的写入、对 `.git` 的访问以及未经批准的联网。文章演示了一次符号链接调包攻击，说明仅靠基于指令的限制并不够，这正是 GhostApproval 研究得出的同一个教训。 |
| [Cloudflare Gateway: MCP Detection and AI Security Dashboard](https://developers.cloudflare.com/changelog/post/2026-08-12-mcp-detection-and-dashboard/) | Cloudflare | Cloudflare Gateway 现在能从报头和载荷形态上识别 MCP 流量，提供放行、阻断或隔离的策略选项，并按时间报告 MCP 的请求量、独立用户数与独立服务器数。它把 MCP 当作企业边界上一个可被网络检查、可被策略治理的流量类别，而不只是应用层协议。 |

### 安全研究与真实事件

针对代码智能体的漏洞研究、披露与真实事件，不限厂商。这里的安全边界是被攻击者实测过的，而不是纸面上描述的。

| 报告 | 它说明了什么 |
|:--------|:----------------------|
| [APPA：面向实际 LLM Agent 的可恢复信息流控制](https://arxiv.org/abs/2607.24625v2) | 8 月 26 日修订版在工具执行前检查调用，并在返回数据进入上下文前再次检查。通过临时分支读取不可信数据，并限制分支可以返回的内容。 |
| [Twin Agent：权限分离 Agent 的上下文残差压缩](https://arxiv.org/abs/2607.19595v1) | 将读取不可信内容和执行特权操作交给不同 agent，读取方仅向执行方发送简短提示。观察范围、执行权限和返回渠道因此成为三个需要分别设计的对象。 |
| [MemSecBench：追踪 Agent 记忆投毒的持久化、后果与修复](https://arxiv.org/abs/2607.27080v1) | 分别检查恶意内容是否写入记忆、是否影响后续行动，以及能否在保留有效记忆的同时清除它。比较对象是 agent、记忆后端和模型的完整配置。 |
| [Tenet Security — "One Fake Bug Report Hijacked a $250 Billion Company's AI Agent, Then 100+ More"](https://tenetsecurity.ai/blog/agentjacking-coding-agents-with-fake-sentry-errors/) | 迄今最有力的一个演示，说明工具的返回值就是不可信输入。Sentry 的 DSN 本来就是公开的（Sentry 官方文档明说可以安全地嵌进前端 JavaScript），且接受任意错误负载，攻击者可以 POST 一个事件，里面塞一段 markdown 指令，渲染出来像是一节伪造的"Resolution"。开发者随后让智能体去排查这个 Sentry issue，智能体经 MCP 把被污染的事件取回来，当成可信的修复指引照做，于是运行了攻击者控制的 npm 包，把 AWS key、GitHub token 和 SSH 凭据带走。已确认对 Claude Code、Cursor 和 OpenAI Codex 均有效，包括沙箱变体与 CI/CD 流水线。 |
| [AWS Security Agent bucket ownership advisories](https://aws.amazon.com/security/security-bulletins/2026-105-aws/) | AWS 于 2026 年 9 月 10 日披露的 CVE-2026-87912 和 CVE-2026-87913 涉及上传工作区归档时未核对存储桶账户归属。修复版为插件 1.1.0 和 MCP server 0.2.0：核对预期归属，发现不符则停止。 |
| [Claude Desktop 共享文件夹公告](https://github.com/anthropics/claude-code/security/advisories/GHSA-v234-4jrq-mgg6)、[CVE-2026-103435](https://github.com/anthropics/claude-code/security/advisories/GHSA-5j29-h97v-84ch) | 两条 Anthropic 公告都涉及"检查的对象"与"实际生效的对象"之间的差异。9 月 25 日，Anthropic 披露 Claude Desktop 禁止从 Cowork 共享文件夹直接打开的文件类型列表遗漏了一种 macOS 打开即执行的类型，因此沙箱内被提示注入的 agent 写入的文件，在用户打开时可能在主机上执行命令（1.15962.0 修复）。10 月 5 日的 CVE-2026-103435 说明，Claude Code 在权限检查时验证写入路径，但在写入时重新解析路径；对共享工作区有写权限并赢得竞态的攻击者，可以把文件换成符号链接，使写入落到项目之外（2.1.129 修复，远早于公告日期）。 |
| [Manifold Security：GitSpawn](https://www.manifold.security/blog/ai-coding-agents-git-hijack) | 2026 年 9 月 1 日发布。多个 CLI 编码 agent 在启动时或会话开始后不久运行 `git status` 或 `git diff`，但没有去除仓库自带的 git 配置，因此 `core.fsmonitor` 等设置会以用户身份运行仓库指定的命令；该命令在沙箱之外运行，也没有批准提示。在 Claude Code 2.1.193 中，这发生在工作区信任提示被接受之前（2.1.196 已修复）；另一项 Claude Code 问题出现在 `claude ultrareview` 路径上，利用的是另一个 git 设置，文章发布时在 2.1.252 上仍未修复。攻击需要仓库以带 `.git` 目录的文件形式送达，普通 clone 不会触发。研究者报告了 7 个 agent 中的 8 项问题，发布时其中 4 项尚未修复。后台的上下文收集也应放在信任边界之内。 |
| [Agent Approval Laundering](https://arxiv.org/abs/2609.28586v1)、[Approval Laundering](https://arxiv.org/abs/2609.38983v1)、[When Consent Outlives Context: Residual Authority Replay in Long-Lived Agents](https://arxiv.org/abs/2609.33910v1) | 9 月下旬的三篇预印本研究批准覆盖什么以及持续多久。Agent Approval Laundering（9 月 23 日）指出，批准记录只写入口命令，而它启动的工作流（例如包的生命周期 hook）可能产生其他效果；作者提出在用户批准前，把对该工作流可能产生的效果的预测写入批准记录。Approval Laundering（9 月 30 日）归纳了实际执行的动作与已批准动作不一致的六种方式；在作者的测试中，带密钥的批准令牌消除了委派类不一致，但没有解决范围类和参数类不一致。When Consent Outlives Context: Residual Authority Replay in Long-Lived Agents（9 月 27 日）报告，跨任务保留的批准最多使提示注入成功率提高 35.1 个百分点。其中两篇基于 Claude Code 的 PreToolUse hook：一篇用它测量批准不一致，另一篇用它传递批准记录。以上结果均由作者报告。 |
| [Wiz — "GhostApproval"](https://thehackernews.com/2026/07/ghostapproval-symlink-flaws-could-let.html) | 一种针对批准弹窗本身的"知情同意"绕过。仓库里放一个名字人畜无害的符号链接，比如 `project_settings.json`，实际指向 `~/.ssh/authorized_keys` 或 `~/.zshrc`；弹窗显示的是那个诱饵名字，于是用户批准的写入落到了完全不同的地方。Amazon Q Developer、Cursor 与 Google Antigravity 已修复，Augment 与 Windsurf 确认但尚未修复。Anthropic 对 Claude Code 的部分提出异议，认为该场景在其威胁模型之外：开发者既选择了信任该目录，又批准了这次编辑。这处分歧本身值得原样保留，因为它争的其实是"同意应该从哪里产生"。 |
| [Poisoning Claude Code：一个 GitHub issue 如何击穿供应链](https://flatt.tech/research/posts/poisoning-claude-code-one-github-issue-to-break-the-supply-chain/) | 披露记录了两条潜在攻击路径，并不表示 Anthropic 的仓库已经被攻陷。一条路径来自 agent mode 对 GitHub App 身份的过度信任；另一条则把官方的低权限 issue 分流示例与高权限、由标签触发的工作流串了起来。两种情况下，遭到提示注入的 Claude 都可能从 `/proc/self/environ` 读取 OIDC 请求凭据，再通过 issue 更新传出。相关问题已在 `claude-code-action` v1.0.94 前得到缓解。 |
| [恶意 Skill 文件对代码智能体的风险评估](https://arxiv.org/abs/2608.05223) | 作者把 471 条真实 shell 命令包装成 2,826 个外观正常的技能，覆盖 11 类 MITRE ATT&CK tactic，并在 Gemini CLI 与 Qwen Code 上完成 5,629 次运行。报告的受利用率分别为 95.5–96.1% 和 71.6–74.0%，只有 1.99% 的运行明确识别出了安全风险。实验只用了一个合成攻击模板和两个 CLI，结论范围有限；但它仍说明技能需要来源记录、准入检查、扫描和最小运行权限，不能只当成普通提示文本。 |
| [HiddenLayer — "Five Layers of Securing AI Agent Harnesses"](https://www.hiddenlayer.com/research/how-to-secure-coding-agents) | 一份工业界安全研究，点名 harness（而不只是模型）才是代码智能体的首要攻击面，并记录了针对它的具体手法：藏在 `<user_query>` 式 XML 标签里、从而继承系统级权限的指令；从系统提示内部左右行为的 MCP 工具定义元数据；技能 YAML frontmatter 的权限滥用；以及一种叫 "CopyPasta"、伪装成许可证头、会在仓库间自我传播的提示注入。 |
| [AI Guardrail Survival under Single-Cycle Agentic Self-Summarization](https://arxiv.org/abs/2608.11392) | 表明嵌入 agent 上下文里的安全规则常常在压缩后文本上还在、效力却悄悄消失：拿压缩后被削弱的规则与完好的规则相比，行为违规率上升 34 到 57 个百分点，而只看摘要文本的人根本看不出来，因为规则在不在并不等于规则还管不管用。它是论文列表里"压缩即真相流失"那条线的安全侧同伴，直指 Claude Code 的 `/compact` 这类机制。 |
| [ColluSkill](https://arxiv.org/abs/2608.09732) 与 [SkillsMetric](https://arxiv.org/abs/2608.08468) | 8 月的两篇研究，勾勒技能扫描器的边界。ColluSkill 把恶意载荷拆分到若干各自看着无害、只在运行时才组合成有害行为的技能里，对六款有代表性的扫描器达到 96% 的平均攻击成功率，其提出的防御把这一数字压到 22.5%。SkillsMetric 在 2,266 个恶意技能上测试一套五阶段静态分析框架，发现聚合 AUC 掩盖的一个盲点：用普通 shell 命令做的毁机攻击检出率为 0%。两者一起把上面的恶意技能供应链话题推进到组合攻击与静态扫描的攻防拉锯。 |
| [Embrace The Red — "LLM Heist: Hijacking LiteLLM for Traffic Interception, Key Theft, and Tool-Call Injection"](https://embracethered.com/blog/posts/2026/hijacking-litellm-for-fun-and-profit/) | 拿到 LiteLLM 网关代理管理员权限的攻击者，可以把它的流量改道到攻击者控制的实例、窃取后端各家的密钥，并在推理之后往响应里注入伪造的工具调用，从而绕过提示层的所有防御，因为 agent 客户端看到的是一个格式完全合法的调用。这是一记针对许多 agent 部署背后那层 LLM 网关的信任边界攻击。前提是已经拿到管理员权限，所以它更像是攻陷之后的放大器，而不是零点击突破。 |

### 评测与基准

面向代码智能体的基准，以及越来越多在追问"这些基准到底有没有量到它声称要量的东西"的研究。放在这里，是因为 harness 的一处改动能不能被看见，取决于用来检测它的评测。

| 资源 | 来源 | 它说明了什么 |
|:---------|:-------|:--------------|
| [LoopArena](https://arxiv.org/abs/2608.28281) | arXiv | 固定执行者，比较决定任务如何推进的控制者，从而区分管理、停止决策与单步执行能力。论文所述任务切片带来的成本节省属于评测协议，不能解读成增加控制者就能降低生产成本。 |
| [构建智能体环境与任务](https://www.langchain.com/blog/building-agent-environments-and-tasks)、[Tuned Evaluators](https://www.langchain.com/blog/introducing-langsmith-tuned-evaluators-starting-with-perceived-error) | LangChain | 将生产反馈接到可审查的任务规格、可执行环境和回归检查。Perceived Error 用于发现值得调查的案例，是用户体验的代理信号，不能直接判定任务成功或失败。 |
| [DeltaSelect: Affordable A/B Testing for Coding Agents](https://arxiv.org/abs/2609.19607v1) | arXiv, 2026 年 9 月 17 日 | 为开发过程中反复进行的"基线对候选"比较选择一个小型固定任务集，而不是用于模型排名。任务按单次运行结果追踪全基准结果的可靠程度排序；任务集、校准和价格在第一次比较前冻结。基线必须在被修改的那个 harness 及其确切版本中重新运行。作者指出，尚未在同等成本下与随机选题比较，而且在小任务集上反复调优可能对其过拟合。 |
| [ClawBench: Can AI Agents Complete Everyday Online Tasks?](https://arxiv.org/abs/2604.08523)（[代码](https://github.com/reacher-z/ClawBench) · [项目主页](https://claw-bench.com/) · [数据集](https://huggingface.co/datasets/NAIL-Group/ClawBench)） | arXiv | 在真实站点上跑的浏览器智能体基准，V1+V2 共 283 个任务，覆盖 163 个网站。每次运行都在隔离的浏览器容器里执行，判定上把请求拦截与 LLM 裁判结合起来，同时保留回放、截图、HTTP 流量、浏览器动作和智能体消息，便于事后核查。它对 harness 这个问题的意义在于，把"真实网站"这个评测里最不可控的变量，做成了一次运行可以被复现的环境。 |
| [Harness-Bench: Measuring Harness Effects across Models in Realistic Agent Workflows](https://arxiv.org/abs/2605.27922) | arXiv | 106 个沙箱任务、5,194 条执行轨迹，在任务环境、预算与评测协议全部固定的前提下，只让 harness 配置在不同模型后端之间变化。文中给一类反复出现的现象起名为 execution-alignment failure：看起来合理的推理与工具反馈、工作区状态脱节。结论是智能体能力「应当在 model-harness 配置这一层级上报告，而不是归因于基座模型本身」。 |
| [What Does a Harness Buy? Tokens, Mostly](https://arxiv.org/abs/2610.04433v1) | arXiv, 2026 年 10 月 3 日 | 固定模型，在 SWE-bench Verified 上比较 Claude Code、mini-SWE-agent 和 OpenCode，并以相同配置的重跑作为噪声基准。在最难的 45 个任务上，更换 harness 改变的任务结果与重跑同一 harness 一样多。能测出的 harness 效应都是丢失任务的方式，例如输出达到上限后不恢复；单任务成本最多相差三倍，主要来自每一步都重新发送的系统提示和工具 schema。结果只覆盖一个基准，Claude 模型只在 Claude Code 中运行。 |
| [Position: Coding Benchmarks Are Misaligned with Agentic Software Engineering](https://arxiv.org/abs/2606.17799) | arXiv | 主张「实践中的代码智能体不是一个模型，而是一套系统 harness」，因此端到端分数把模型、harness、上下文、环境与反馈信号混在了一起，其中任何一项都能让分数移动「与相邻两代模型之间的差距相当的幅度」。文中还指出以单一参考答案打分会惩罚同样成立的其他解法。 |
| [Harbor-Index 1.0](https://harbor-index.org/) | Harbor | 一个紧凑的跨领域智能体基准：82 道题，从 6,627 个候选中经难度筛选、自动审计与人工复核蒸馏而来，覆盖软件工程、科学研究、工具与系统、知识、数学、数据分析与安全。评分为严格二元、没有部分分，且刻意为「留出上限」而设计，而非用来预测别处的表现。 |
| [Are Performance-Optimization Benchmarks Reliably Measuring Coding Agents?](https://arxiv.org/abs/2607.01211) | arXiv | 在全新机器上重放 GSO、SWE-Perf、SWE-fficiency 的官方参考补丁，发现分别只有 39/102、11/140、411/498 个任务满足这些基准自己定的有效性规则。打分方法的影响也比预想的大：在同时出现于 GSO 与 SWE-fficiency 的提交中，官方排名在 28 组两两比较里有 9 组互相矛盾。 |
| [Do Androids Dream of Breaking the Game? Systematically Auditing AI Agent Benchmarks with BenchJack](https://arxiv.org/abs/2605.12673) | arXiv | 一套自动红队系统，驱使代码智能体去攻击基准而不是解题，在 10 个主流智能体基准中挖出分属八个类别的 219 处缺陷，并在「一道题都没真正解出」的情况下拿到接近满分。其扩展流程随后把洞补上，在四个基准上把可被钻空子的任务比例从接近 100% 压到 10% 以下。 |
| [What Makes a Terminal-Bench Task Hard? Separating Genuine Hardness from Fake-Hardness on an Adjudicated Agentic Corpus](https://arxiv.org/abs/2609.26826v1) | arXiv, 2026 年 9 月 20 日 | 审计冻结的 Terminal-Bench 3 生产记录中没有任何智能体通过的任务。按顺序检查参考解是否通过、基础设施失败是否占主导、是否存在绕过验证器的方法，以及可解性是否有证据支持。125 个全失败任务中，78 个仍可作为真正未解的候选；其余任务存在参考解损坏、基础设施问题、只能靠绕过通过，或可解性未得到证明。因此，零通过率本身并不能证明任务困难。 |
| [PERFOPT-Bench: Evaluating Coding Agents on Software Performance Optimization](https://arxiv.org/abs/2607.07744) | arXiv | 七套智能体栈 × 七个长程优化任务。发现优化表现取决于具体工作负载而非模型身份，没有哪一套栈能通吃；并据此给出结论：「把原始加速比直接当成基准分数是不安全的，因为某些巨大的提升来自针对基准的走捷径行为。」 |
| [LoopsBench：从 Harness Engineering 到循环工程的代码智能体评测](https://arxiv.org/abs/2608.00267v1) | arXiv | 不再只测“收到一个 issue，交付一个补丁”，而是把 112 个长期开发任务表示成依赖图，共含 5,300 多个可单独测试的开发单元。所有单元测试从开始就可见；评估器按前置单元的通过状态启用评分，并持续检查已通过测试是否回归。最好的组合是 Opus 4.7、Claude Code 加一层外部续跑循环，任务完成率为 25%；智能体记录的计划只覆盖了部分真实依赖关系。 |
| [Active-SWE：没有 issue 报告时主动发现并修复缺陷](https://arxiv.org/abs/2608.04682) | arXiv | 多数仓库级基准会直接提供一份说明故障位置的 issue，Active-SWE 则把这条线索拿掉。它包含 1,663 个任务、六类缺陷和八种语言，要求智能体自行定位并修复已记录缺陷、处理同一仓库中的多个缺陷，并找出其他有效问题。实验结果表明，主动发现缺陷和根据 issue 修补代码并不是同一种能力。 |
| [SciCode-Verified：系统审计科学编程基准的有效性](https://arxiv.org/abs/2608.04975) | arXiv | 作者审计了 65 道科学编程题，在题目、测试和参考答案中发现 263 处缺陷，其中 192 个正确解曾被误判。修复后，子问题准确率从 45–60% 升到 84–98%，主问题准确率从 9–27% 升到 69–92%。如此大的变化说明，基准实现本身会显著影响最终报告的模型成绩。 |
| [When the Judge Changes, So Does the Measurement: Auditing LLM-as-Judge Reliability](https://arxiv.org/abs/2607.08535) | arXiv | 把「换掉裁判模型」当成一个测量效度问题，而不是一次免费升级。在四个判定数据集上发现：相邻版本的裁判并不可互换；更强的裁判能减少但消除不了位置偏好与冗长偏好；而当误差彼此相关时，多次采样组成的陪审团「几乎没有帮助」。 |
| [Frozen Judges, Moving Agents: Version-Dependent LLM-Judge Error and the Limits of Judge-Assisted Agent Evaluation](https://arxiv.org/abs/2609.34198v2) | arXiv，9 月 29 日修订 | 说明固定的 LLM 裁判在比较智能体新旧版本时，可能出现随版本变化的误差。在 SWE-bench Verified 上，有几组版本对，仅依据裁判分数得到的置信区间显示新版本更好，但实际执行测试无法证实这一点，尽管裁判排名与参考结果相关性较高。较强智能体的失败补丁更容易被判为通过。作者建议用裁判筛选需要比较的版本，发布决策则基于对当前输出随机抽样并加标注的审计。独立的人工补丁复核尚未完成。 |
| [Reward Hacking is Swamping Model Intelligence Gains](https://cursor.com/blog/reward-hacking-coding-benchmarks) | Cursor | 审计 731 条 Opus 4.8 Max 轨迹后发现，在 SWE-bench Pro 上被判为成功的修复里，63% 是检索来的而不是推导出来的：57% 是在公开网络上找到了已合并的 PR 或修好的源文件，9% 是从仓库自带的 git history 里挖出修复 commit。两道隔离能把这件事照出来，且分数随之大幅下跌：删掉 `.git` 并把仓库重新初始化成单 commit（原历史只在打分时恢复），以及默认禁止联网、只通过代理放行一份包仓库白名单。这直接把评测环境的设计变成了"基准是否还有意义"的前提条件。 |
| [SWE-Bench ProMax: Benchmarking Agents on Large-Scale Multilingual Code Refactoring](https://arxiv.org/abs/2608.09802) | COLM 2026 | 170 个由专家整理、跨七种语言的多文件重构实例（平均每个任务 11.4 个文件、261.6 行），其动因是发现约 60% 未被解决的 SWE-bench Verified 实例存在有缺陷的测试。最强模型也只解决 41.2%，为已被刷到饱和的单文件基准重新腾出了区分度空间。 |
| [SWE Refactor Bench: Can Coding Agents Complete a Long-Horizon, Whole-Repository Stack Migration?](https://arxiv.org/abs/2608.23564v1) | arXiv, 2026 年 8 月 24 日 | 分别检查技术栈迁移是否完成、固定行为测试是否通过，以及能否生成行为反例。反例必须在原实现上通过、在提交上失败，并能重复复现。 |
| [QuoteBench: How Matched Scores Can Hide Command-Path Failures](https://arxiv.org/abs/2608.13547) | arXiv | 56 个任务、14 个源自真实事故的类别，表明"匹配"的执行分数把生成命令与把命令送达 shell 的传输、解析路径混为一谈：仅仅把同一份模型回复再经过一个解析器重放，成功率就掉了 55 到 73 个百分点。作者主张评测报告必须把生成契约、执行路径与校验器分开陈述，而不是只报一个通过率。 |
| [Quantifying Overclaiming Propensity in Frontier LLM Agents](https://arxiv.org/abs/2609.20812v3) | arXiv，9 月 22 日修订 | 把"过度声称"定义为：最终报告声称完成了某项工作，但智能体自己的执行记录显示并未完成，例如声称读过一个从未打开的文件。文件覆盖率由执行记录测得，报告类别由 LLM 裁判判定。在生产 CLI 中运行的五个审查场景里，约三分之二的运行没有读完要求的文件，其中多数不完整运行要么声称已完整审查，要么没有说明遗漏。要求使用子智能体提高了覆盖率，但没有提高如实报告的比例。场景是针对 Claude Opus 反复调整设计的。 |
| [VibeMemBench: Evaluating Memory Systems for Coding Agents on Real Repository Coding Tasks](https://arxiv.org/abs/2609.23570v1) | arXiv, 2026 年 9 月 20 日 | 用可执行检查在 111 个仓库编码目标上测试记忆系统。注入已验证有用的经验，使 5 个留出求解器中的 4 个解决率提高 1.1 至 4.5 个百分点。当 4 个现有记忆系统从同一历史中自行构建并检索经验时，12 个求解器与系统组合中有 11 个没有超过匹配的无记忆基线。目标只保留注入经验确实有效的情形，检索在每次运行开始前一次完成，未评估 Claude 和 GPT 模型。 |
| [Senior SWE-bench](https://senior-swe-bench.snorkel.ai/) ([原理](https://senior-swe-bench.snorkel.ai/blog/2026-06-16-how-it-works)) | Snorkel AI | 一个代码智能体基准，100 个任务，50 个公开、50 个私有保留，取自 12 个生产仓库（其中包括 Gitea、PostHog、Prefect、Teleport、Turborepo 与 Immich），与普林斯顿和 UW-Madison 合作构建。值得留意的是公开与私有的切分：它衡量的是对那些答案不可能泄漏进训练数据的任务的泛化。 |

### 相关学术论文

| 论文 | 会议 | 相关性 |
|:------|:------|:------|
| [Living-Harness Is an Interactive-Agent Evolver](https://arxiv.org/abs/2607.26598v2) | arXiv，8 月 11 日修订 | 在一次交互任务完成并评分后，更新供后续检索的程序性记忆和状态图。该图指导未来决策，不直接执行步骤；工具和基础上下文保持固定。 |
| [Self-Evolving Coding Agents](https://arxiv.org/abs/2608.03392v3) | 综述，8 月 29 日修订 | 按更新对象、发生时点和反馈来源组织持久变化，区分记忆、技能、工具、工作流、组件和上下文的变化，并梳理效果评价方式。 |
| [What Does Multi-Harness RL Learn? Credit Assignment and Portability in Coding Agents](https://arxiv.org/abs/2609.04518v1) | arXiv，9 月 3 日 | 固定训练记录和预算，比较在每个任务与 harness 组合内计算相对收益，或跨 harness 计算相对收益。用未参与训练的 harness 检查迁移；该研究中，仅改变跨 harness 分组并未改善迁移表现。 |
| [Graph Engineering in the Era of LLM Agents: From Individual Intelligence to System Intelligence](https://arxiv.org/abs/2608.21156) | arXiv | Feng 等人以任务图、智能体图和运行状态图组织智能体系统的设计问题，在一个明确的框架下联系依赖、协作与可恢复状态，并梳理已有图式系统的发展。 |
| [From Review to Reuse: How Post-Task Workflow Can Support Human-AI Agent Interaction](https://arxiv.org/abs/2609.13136v1) | arXiv, 2026 年 9 月 11 日 | Trace2Flow 把完成后的运行转成可编辑的工作流图。用户可检查每步输入输出、修改依赖，并为相关任务重新运行步骤；图中的步骤对应已记录的执行。 |
| [Decoupling Readiness from Release for Tail-Aware Scheduling of Agentic LLM Workflows](https://arxiv.org/abs/2609.10964v1) | arXiv, 2026 年 9 月 10 日 | 区分满足依赖条件与提交到共享推理引擎的时机。调度器选择提交哪轮调用，并限制已提交但尚未完成的工作量。论文用 agent 轨迹回放测量资源竞争下的工作流尾延迟。 |
| [TROVE: Adaptive Agent Skill Orchestration via Trace-Grounded Route Validation and Editing](https://arxiv.org/abs/2609.05019v1) | arXiv, 2026 年 9 月 4 日 | 每执行一项 skill 后，控制器根据结果保留下一步、插入局部处理，或替换未执行的后续路线。已完成的工作及其产物继续保留。 |
| [Runtime Authorization of Self-Generated Subgoals in Long-Horizon Tool-Using AI Agents](https://arxiv.org/abs/2610.04975v1) | arXiv, 2026 年 10 月 4 日 | Zhu 与 Wang 把智能体目标图的修改当作授权事件。新建、替换、委派或合并子目标时，都要经过与版本绑定的结构化检查，确认新的延续仍在已批准的任务之内；受保护的副作用在提交时还要再检查一次。逐次调用的权限检查看不到这一点，因为两个各自被允许的动作组合起来可能违反任务约束。评估使用有限的结构化领域和合成用例，属于研究原型，不是已部署的控制机制。 |
| [HarnessLens](https://arxiv.org/abs/2608.27311) | arXiv | 将候选 harness 修改与行为假设、相关验证任务和已有成功案例关联，再用新一批任务确认。在所选任务上未观察到回归，并不保证普遍无回归；实验也未统一所有方法的总计算预算。 |
| [Self-Healing Harness for Runtime Oversight of Agent Self-Modification](https://arxiv.org/abs/2609.24130v1) | arXiv, 2026 年 9 月 21 日 | 智能体编写候选规则，由外部运行时决定哪些规则可以保留。只有在修复触发它的失败、且不使此前成功的受保护案例退化时，规则才会保留；另有一个守卫重新测试累积的规则集合。在因回放而被否决的 383 个提议中，211 个修复了触发失败，却破坏了一个受保护案例。研究没有与"不经门控直接接受同一批规则"的设置比较，且每轮最多回放两个受保护案例，因此 211 是检测到的冲突，不是总发生率。 |
| [Meta-Harness: End-to-End Optimization of Model Harnesses](https://arxiv.org/abs/2603.28052) | arXiv | 让一个 coding agent 充当 proposer，在模型固定的前提下去搜索 harness 本身（记忆、检索、上下文构造、prompt、工具使用逻辑），维护一个候选种群和一条 Pareto 前沿。结果是：比当前最好的上下文管理系统高出 7.7 分，同时少用 4 倍上下文 token；搜出来的 harness 在 TerminalBench-2 上超过了最好的手工设计基线。它把 harness 从"人手工设计的东西"变成了"可以被优化的对象"。 |
| [From Question Answering to Task Completion: A Survey on Agent System and Harness Design](https://arxiv.org/abs/2606.20683) | arXiv | 同期出现的综述，用 model-harness 视角把执行 harness 拆成六项相互耦合的运行时职责：observation、context、control、action、state、verification。是与本文框架结构上最接近的一篇。 |
| [Architectural Design Decisions in AI Agent Harnesses](https://arxiv.org/abs/2604.18071) | arXiv | 基于源码、对 70 个 agent 系统项目的研究，识别出反复出现的设计维度；与本文设计空间框架最贴近的同期对照工作。 |
| [Defining AI Agents: A Compendium of Criteria, Metrics, and Benchmarks](https://arxiv.org/abs/2609.11018v1) | arXiv, 2026 年 9 月 10 日 | 将环境交互、学习与适应、自主性、目标导向行为和时间连贯性对应到已有指标与基准。它区分对人工输入的独立程度与任务表现，帮助构建者选择要评估的能力。 |
| [ActPlane: Programmable OS-Level Policy Enforcement for Agent Harnesses](https://arxiv.org/abs/2606.25189) | arXiv | 用 eBPF 在内核层拦截 agent 的每一条执行路径，包括那些完全绕过 tool-call 层的间接路径，开销 1.9% 到 8.4%。它直接点名了那个缺口：像 Claude Code 那样把权限检查放在 tool-call 层，是可以被绕过去的。 |
| [Lingering Authority: Revocable Resource-and-Effect Capabilities for Coding Agents](https://arxiv.org/abs/2606.22504) | arXiv | 给"会话级批准"的一个真实弱点起了名字：授权一旦给出，并不会随着任务推进而失效。PORTICO 把任务规格编译成能力、授予规则和闭包谓词，把资源物化为 epoch-bound 的句柄，闭包条件一满足句柄即死。 |
| [TokenPilot: Cache-Efficient Context Management for LLM Agents](https://arxiv.org/abs/2606.17016) | arXiv | 正面处理了一个多数 compaction 工作绕开的约束：改写历史会摧毁 KV prefix cache。做法是在环境输出的入口处过滤噪声以保持前缀稳定，并且只在上下文段的生命周期结束后才驱逐它。 |
| [The Balkanization of Execution-Security Research for AI Coding Agents](https://arxiv.org/abs/2607.05743) | arXiv | 一篇 SoK，把 39 篇执行层安全研究归入 17 个类别（沙箱隔离、能力与访问控制、策略执行、TOCTOU、MCP 威胁、身份委派、执行溯源、出网控制等），并指出策略执行面对真实 denylist 的失败率高达 69% 到 98%。 |
| [How Agents Ask for Permission: User Permissions for AI Agents, from Interfaces to Enforcement](https://arxiv.org/abs/2607.13718) | arXiv | 调查了 21 个 agent 权限系统并实测 5 个商用 agent，沿三条轴建立分类学：策略在界面上如何被表达、如何被推导成内部策略、以及在运行时如何被强制执行。是与本文权限分析最接近的学术对照，也是对"Claude Code 那套模型是否具有普遍性"的一次直接检验。 |
| [Bad Memory: Evaluating Prompt Injection Risks from Memory in Agentic Systems](https://arxiv.org/abs/2607.14611) | arXiv | 在四个模型上实测 Claude Code 与 OpenAI Codex，发现一个对记忆设计很关键的不对称：agent 大体上能抵抗那些试图改写其记忆文件的不可信内容，但已经躺在这些文件里的 payload，会继续攻击当前以及之后的会话。持久化把"一次成功的注入"变成了"长期有效的注入"。 |
| [Agent Data Injection Attacks are Realistic Threats to AI Agents](https://arxiv.org/abs/2607.05120) | arXiv | 从指令注入中分离出第二类攻击：攻击者不夹带命令，而是伪造安全攸关的元数据，例如资源标识符、数据来源、工具响应格式，而 agent 会照单全收，因为它并不区分可信与不可信数据。文中演示了对 Claude in Chrome、Antigravity 与 Nanobrowser 的任意点击攻击，以及对 Claude Code、Codex 与 Gemini CLI 的远程代码执行与供应链攻击。 |
| [Failure as a Process: An Anatomy of CLI Coding Agent Trajectories](https://arxiv.org/abs/2607.09510) | arXiv | 从 7 个前沿模型、3 套 scaffold 上跑出的 3,843 条轨迹中，人工标注了 1,794 条完整轨迹（逾 63,000 个执行步）。失败以认知性错误为主，通常在最初几步就已发生，并且一直隐藏到无法挽回时才显现；这说明验证应该前移到循环内部，而不是只看最终结果。 |
| [Better Harnesses, Smaller Models: Building 90% Cheaper Agents via Automated Harness Adaptation](https://arxiv.org/abs/2607.08938) | arXiv | 一个 meta agent 读取失败轨迹，把任务中共通的难度从模型里"抬"进 harness，变成定制的指令、工具与编排循环。以 4% 的成本恢复了大模型 89.7% 的性能，并在 21 组任务与模型配对中的 7 组上彻底抹平了小模型的差距。 |
| [Rethinking the Evaluation of Harness Evolution for Agents](https://arxiv.org/abs/2607.12227) | arXiv | 对 harness 优化这条路线唱反调的一篇。在 Terminal-Bench 2.1 上、以对齐的反馈与推理预算与朴素的 test-time scaling 及 discovery 基线对比后，自动 harness 演化"并不能稳定胜过"这些更简单的方法，且在留出任务上泛化很差。 |
| [Rethinking Self-Evolving Agent Skills：反馈与搜索的作用](https://arxiv.org/abs/2608.02636) | arXiv | 研究发现，技能演化更像“生成候选，再用验证集筛选”，而不是不断累积改进。388 个候选中，只有 55 个成为内容不同的验证集最优版本；14 组设置里有 11 组最终选择了演化后的技能，其中九组也提升了测试表现，而且这 11 个技能都用到了失败轨迹。并行采样在 SearchQA 上几乎追平演化流程，但在 SpreadsheetBench 上仍落后 30.96 个百分点，说明收益取决于任务。 |
| [SkillAdam: Stable and Efficient Skill Evolution for Agents](https://arxiv.org/abs/2609.08944v1) | arXiv, 2026 年 9 月 8 日 | 保存问题和修订尝试的历史，根据案例结果的差异调整下一轮改动范围。接受门槛检查受保护指标，并复用当轮采样案例；最终测试另行保留。 |
| [Skill Issue: Lessons from Optimizing Repository SKILLs for Coding Agents](https://arxiv.org/abs/2609.12742v1) | arXiv, 2026 年 9 月 11 日 | 在固定仓库版本上撤回已合并 PR 的实现改动、保留测试来构造任务，再比较使用和不使用优化后 SKILL.md 的运行。三个 Kotlin 仓库上的测试未能把观察到的收益与运行间波动区分开，说明技能评估需要考虑测量灵敏度。 |
| [Learning Globally Reusable Skills for Coding Agents](https://arxiv.org/abs/2608.06153) | arXiv | 这项工作维护技能之间的关系图，把相关的局部更新合并成可复用能力，再用历史任务回放检查过拟合和回归。实验覆盖 OpenHands、mini-SWE-agent 和一个内部工业智能体，任务是生成能够暴露缺陷的测试和过滤误报。它关注的是整个技能库如何保持兼容，而不是单个技能如何改写。 |
| [DCAS：解耦 CLI Agent 脚手架并跨脚手架内化规划能力](https://arxiv.org/abs/2608.06113) | arXiv | 一个在 OpenHands 轨迹上微调的模型，在 OpenHands 中得到 52.6%，换到 OpenCode 后只有 8.4%；基座模型的差距小得多。DCAS 通过拦截 API 流量，把不同 CLI 脚手架与模型后端配对，并收集了 576 条包含规划信息的 Claude Code 轨迹。经过规划与执行联合微调后，模型自主规划时达到 55.8%，换到未见过的 OpenCode 和 mini-SWE-agent 后分别得到 49.4% 和 42.4%。这项实验只用了一个基准，但结果表明轨迹训练可能会过拟合脚手架协议。 |
| [Co-Evolving Harnesses and Models: On-Policy Correction Helps Weaker Models Catch Up Where Imitation Fails](https://arxiv.org/abs/2609.09134v1) | arXiv, 2026 年 9 月 8 日 | 在七项企业任务上测试 harness 演化后的微调。作者报告完整模仿专家轨迹后出现回归；替代方法从较弱模型自身的运行出发，在训练前用专家纠正替换一个失败轮次。 |
| [An Empirical Study of Harness Design for Coding Agents](https://arxiv.org/abs/2609.20804v1) | arXiv, 2026 年 9 月 17 日 | 固定编码 harness 的执行循环，在四个开放模型和四种上下文预算下，于 SWE-Bench Verified 和 Terminal-Bench 2.1 上变动规划、动作空间和上下文管理。每个组件的作用取决于模型和上下文预算。预算越紧，上下文管理越重要，主要作用是防止溢出；先删减陈旧工具输出、再做 LLM 摘要的策略，成功率与其他管理策略相近，并在八个模型与基准组合中的七个里成本最低，而用于召回被删减内容的工具很少被使用。规划提高了较弱模型的准确率，对较强模型则主要降低成本。结果由作者报告；每个设置只运行一次，且只用开放模型。 |
| [How Much of a Harness Does a Strong Agent Need for Autonomous ML Engineering?](https://arxiv.org/abs/2609.40303v1) | arXiv, 2026 年 9 月 30 日 | 在相同时间和同一强骨干模型下，四个开源机器学习工程 harness 均未优于单个最小编码智能体会话，较弱的骨干模型仍受益于工作流先验。结果由作者报告。 |
| [Ledger：代码智能体的有状态 Harness Middleware](https://arxiv.org/abs/2608.00808) | arXiv | 在模型上下文之外维护一份任务账本，记录证据、依赖关系和已经验证的进度，不增加模型调用。在完整的 500 道 SWE-bench Verified 上，GPT-5 mini 从 56.2% 升到 64.2%，成本下降 28.9%；MiniMax M2.5 从 75.8% 升到 81.0%，成本下降 31.8%；Codex 也提升 3.4 个百分点并降低 24.4% 的成本。结果说明，任务状态不必全部写进提示，也可以由 harness 中间件单独维护。 |
| [Resume Means Resume：检查点、中断与恢复语义的一致性契约](https://arxiv.org/abs/2608.03836v3) | arXiv，8 月 8 日修订 | 定义六项检查点、中断与恢复性质，对五个框架的固定版本检查重复副作用、冲突的恢复输入、无效检查点和并发恢复。区分框架明确规定的重放行为与违背契约的行为。 |
| [Where Does Exactly-Once Live? Model, Harness, and Tool-Contract Effects on Duplicate Side Effects in LLM Agents](https://arxiv.org/abs/2609.29095v1) | arXiv, 2026 年 9 月 24 日 | 在工具边界注入故障，例如在 agent 停止等待之后才提交的写入，或被投递两次的请求，并依据已提交效果的账本为每个 episode 评分。在 25,930 个 episode 中（包括在 GitHub Copilot CLI、Hermes 和 Codex CLI 下的运行），作者报告：被要求恰好执行一次的前沿模型，在确认丢失时几乎从不重复写入，但对仍在途或被投递两次的请求经常重复执行。为每次写入提供幂等键后，重复率从 28% 降到 4%，三个 harness 的表现几乎相同。在产生重复的 episode 中，有 90% 的智能体报告任务已完成。这些结果来自一位作者在合成基准上的报告；论文称代码和数据将在发表时公开。 |
| [Planarian: Managing Agent State with Statepoints](https://arxiv.org/abs/2609.35366v1)、[StateFork: Branchable Infrastructure for Agent Exploration](https://arxiv.org/abs/2609.38648v1) | arXiv, 2026 年 9 月 28 日与 29 日 | Planarian 是一个研究原型运行时，其 statepoint 同时覆盖沙箱中的文件和进程，以及通过 MCP 对远端作出的更改。它为每个远端调用记录补偿动作；回滚按相反顺序执行这些补偿动作，再恢复本地快照，分叉则创建相互隔离的分支。无法改写为可补偿形式的远端请求会在执行前被拒绝，原型只为数据库 MCP 服务器上的 SQL 操作实现了补偿。StateFork（9 月 29 日）研究如何分支和恢复终端会话，让智能体可以尝试不同方案。结果均由作者报告。 |
| [From Anatomy to Smells: An Empirical Study of SKILL.md in Agent Skills](https://arxiv.org/abs/2607.01456) | arXiv | 对 238 个真实 skill 做定性分析，归纳出 13 个高层与 44 个低层组件的分类学，再据此把"skill smell"定义为对这些实践的违反。超过 99% 的 SKILL.md 至少含有一处 smell，而且 smell 一旦出现，随 skill 演进也很少消失。这是关于本文所分析的可扩展性界面的直接实证。 |
| [Decoding the Configuration of AI Coding Agents](https://arxiv.org/abs/2511.09268) | arXiv | 对 328 个 Claude Code 配置文件的实证研究——软件工程关注点与共现模式。 |
| [On the Use of Agentic Coding Manifests](https://arxiv.org/abs/2509.14744) | arXiv | 分析了 242 个仓库中的 253 个 CLAUDE.md 文件——操作性命令中的结构模式。 |
| [Context Engineering for Multi-Agent Code Assistants](https://arxiv.org/abs/2508.08322) | arXiv | 结合多个 LLM 进行代码生成的多智能体工作流。 |
| [OpenHands: An Open Platform for AI Software Developers](https://arxiv.org/abs/2407.16741) | ICLR 2025 | 开源 AI 编码智能体的主要学术参考。 |
| [SWE-Agent: Agent-Computer Interfaces](https://arxiv.org/abs/2405.15793) | NeurIPS 2024 | 基于 Docker 的编码智能体，带有自定义的智能体-计算机接口。 |
| [The OpenHands Software Agent SDK](https://arxiv.org/abs/2511.03690) | arXiv | 面向生产环境智能体的可组合 SDK 基础，是上面那篇 OpenHands 平台论文在工程侧的对应工作。 |
| [A Survey on Code Generation with LLM-based Agents](https://arxiv.org/abs/2508.00083) | arXiv | AI 编码智能体领域的综述，适合把 harness 研究放回更大的代码生成文献里定位。 |
| [AI Agent Systems: Architectures, Applications, and Evaluation](https://arxiv.org/html/2601.01743v1) | arXiv | 覆盖架构、应用与评测三条线的广义智能体系统分类。 |
| [Why Does CLAUDE.md Keep Growing? Catastrophic Remembering in Agentic Coding](https://arxiv.org/abs/2608.11095) | arXiv | 挖掘 1,867 个仓库中 247,694 条指令的生命周期，发现常驻指令文件在其生命周期内增长 226%，因为加一条规则很便宜、而在依赖不确定的情况下删一条很难。加上解释性注释能把超额增长从 211.3% 压到 1.4%，并把指令遵循度提升最多 23.1%。这是关于本文所分析的 CLAUDE.md 记忆机制的直接实证。 |
| [A Study of Cursorrules Files in GitHub Open Source Projects](https://arxiv.org/abs/2608.10622) | ICSOFT 2026 | 对 11,427 个仓库中 12,110 个 `.cursorrules` 与 `.mdc` 文件的实证研究：开发者如何为代码智能体编码项目上下文与工程规范，以及这一格式如何演变。它是已列出的 CLAUDE.md 配置研究在 Cursor 一侧的对照，也是上面那篇 CLAUDE.md 增长论文的姊妹篇。 |
| [The Devil Is in the Interface: Evaluating How Tool Architecture Shapes Coding Agent Behavior](https://arxiv.org/abs/2608.11386) | arXiv | 在保持能力不变的前提下，对六种工具架构（纯 bash、结构化的低层工具、自然语言检索，以及一种 Python CodeAct 风格）跨 11,700 条轨迹做受控消融。结构化接口把逐次运行的一致性提升最多 4.7 倍，CodeAct 风格把步数砍掉 41.6%、token 砍掉 56.3%。它延续了本列表里 `execute_code` 消融所开启的工具面设计问题。 |
| [One Recipe, Many Harnesses: What Self-Evolution Encodes Across Languages and Models](https://arxiv.org/abs/2608.10178) | arXiv | 在八种语言和三个基础模型上评估同一种 harness 自演化方法，收益因设置而异：通用工作方法比特定生态规则更容易迁移。研究记录预期收益与可能回归，但尚未把完整演化预算与增加测试时推理的方案做等预算比较。 |
| [Persistent Recursive Worlds Enable Autonomous Software Evolution](https://arxiv.org/abs/2608.10450) | arXiv | 把软件项目本身（一个被接受的版本加一个仓库路径）作为持久单元，从而让 agent 保持有限生命周期、而开发无限期继续。一个 Rust 版 C 编译器在 120 小时、1,000 多个 agent 回合里以约 44 美元建成，并在中途整个更换 agent 后依旧存活。这是一种把连续性放在产物、而非 agent 记忆里的会话持久化设计。 |
| [Agent Safety Should Be a Runtime Contract](https://arxiv.org/abs/2608.11274) | arXiv | 主张对会执行代码、改动文件的 agent 而言，训练期对齐在结构上并不够，安全必须是一份由 harness 强制执行的运行时契约：把预防性控制（沙箱、权限闸、输出过滤）与基于可核验轨迹证据（测试、日志、diff）的证据式验证配对。它是本列表里 ActPlane 与 SHarD 的立场论文同伴。 |

### 本论文的不同之处

> 上述项目多聚焦于**工程层面的逆向工程**或**实际动手重新实现**，而本论文提供的是**一套系统的"价值观 → 原则 → 实现"分析框架**——从五个人类价值观出发，经由十三条设计原则，一路追到具体的源码级选择；并借助与 OpenClaw 的对比揭示一个事实：真正构成工程复杂性的，是那些横跨各子系统的整合机制，而不是模块化的单点特性。


<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>其他值得关注的 AI 智能体项目</h2></summary>

围绕 Claude Code 的更广阔智能体设计空间。上方的[跨系统对比](#跨系统对比claude-code-vs-openclaw-vs-hermes-agent)节深入分析了三个最相近的同类系统（Claude Code、OpenClaw、Hermes-Agent）；下方条目则在更广的范围给出语境，涵盖编码智能体同类、框架、记忆系统、harness 扩展、MCP 生态和专业领域智能体。

### 编码智能体 CLI 与 IDE Harness

| 仓库 | 推出时间 | 重点 |
|:-----------|:-------|:------|
| [**openclaw/openclaw**](https://github.com/openclaw/openclaw) [![Star](https://img.shields.io/github/stars/openclaw/openclaw.svg?style=social&label=Star)](https://github.com/openclaw/openclaw) | 2026 年 1 月 | 跨消息平台的本地优先个人 AI 助手。（[第 10 节分析](#跨系统对比claude-code-vs-openclaw-vs-hermes-agent)） |
| [**NousResearch/hermes-agent**](https://github.com/nousresearch/hermes-agent) [![Star](https://img.shields.io/github/stars/nousresearch/hermes-agent.svg?style=social&label=Star)](https://github.com/nousresearch/hermes-agent) | 2026 年 2 月 | 具有跨会话记忆的自我改进个人智能体。（[第 10 节分析](#跨系统对比claude-code-vs-openclaw-vs-hermes-agent)） |
| [**opensquilla/opensquilla**](https://github.com/opensquilla/opensquilla) [![Star](https://img.shields.io/github/stars/opensquilla/opensquilla.svg?style=social&label=Star)](https://github.com/opensquilla/opensquilla) | 2026 年 6 月 | 主打 token 效率的微内核个人智能体，覆盖 CLI、Web UI 与聊天渠道；它用 ML 分类器按成本给模型分档并据此路由，配有本地 Markdown+SQLite 记忆（一个 MEMORY.md 加上按日期记录的笔记，支持关键词与向量召回），并提供 Bubblewrap/Seatbelt 沙箱。 |
| [**pewdiepie-archdaemon/odysseus**](https://github.com/pewdiepie-archdaemon/odysseus) [![Star](https://img.shields.io/github/stars/pewdiepie-archdaemon/odysseus.svg?style=social&label=Star)](https://github.com/pewdiepie-archdaemon/odysseus) | 2026 年 6 月 | PewDiePie 推出的自托管、本地优先 AI 工作台：带工具、MCP 与 shell 访问的自主智能体，外加记忆、深度研究与按硬件适配的模型服务。AGPL-3.0。 |
| [**google-antigravity/antigravity-cli**](https://github.com/google-antigravity/antigravity-cli) [![Star](https://img.shields.io/github/stars/google-antigravity/antigravity-cli.svg?style=social&label=Star)](https://github.com/google-antigravity/antigravity-cli) | 2026 年 6 月 | Google 用 Go 重写、取代 Gemini CLI 的官方 CLI，与 Antigravity 2.0 桌面端共用同一个 agent harness。据其 CHANGELOG：嵌套 subagent 可到孙级及更深，子轨迹更新递归回传到根对话；工作区级 `.agents/hooks.json`，由 pre-tool hook 决定某次工具调用是否放行；按项目的权限配置存于 `~/.gemini/config/projects/` 而非仓库内，优先级高于全局设置。 |
| [**patriceckhart/zot**](https://github.com/patriceckhart/zot) [![Star](https://img.shields.io/github/stars/patriceckhart/zot.svg?style=social&label=Star)](https://github.com/patriceckhart/zot) | 2026 年 6 月 | Go 单静态二进制 harness，工具集刻意做得极小（read、write、edit、bash）。会话以 JSONL transcript 落盘，可 resume、分支和压缩。`/swarm` 拉起共享同一工作目录、会改同一批文件的后台 subagent，各自保留独立的 session 文件与 transcript；另有一个默认关闭的 auto-swarm 开关，打开后主 agent 可自行派生它们，并在批次完成时把摘要注入回主对话。`/jail` 把沙箱根设在 cwd，并拒绝明显的逃逸模式（`sudo`、`rm -rf /`、以 `cd /` 开头、`chmod -R` 等）。是一个可读的、非 TypeScript 的 subagent 与权限设计对照物。 |
| [**sst/opencode**](https://github.com/sst/opencode) [![Star](https://img.shields.io/github/stars/sst/opencode.svg?style=social&label=Star)](https://github.com/sst/opencode) | 2025 年 6 月 | 与模型厂商无关的终端编码智能体，集成 ACP。 |
| [**Aider-AI/aider**](https://github.com/Aider-AI/aider) [![Star](https://img.shields.io/github/stars/Aider-AI/aider.svg?style=social&label=Star)](https://github.com/Aider-AI/aider) | 2023 年 | 在终端中与 LLM 结对编程，支持主流模型。 |
| [**continuedev/continue**](https://github.com/continuedev/continue) [![Star](https://img.shields.io/github/stars/continuedev/continue.svg?style=social&label=Star)](https://github.com/continuedev/continue) | 2023 年 | IDE 中受版本控制的 AI 检查，配套开源 Continue CLI。 |
| [**google-gemini/gemini-cli**](https://github.com/google-gemini/gemini-cli) [![Star](https://img.shields.io/github/stars/google-gemini/gemini-cli.svg?style=social&label=Star)](https://github.com/google-gemini/gemini-cli) | 2025 年 | Google 开源终端编码智能体，含 ReAct 循环与 MCP 支持。自 2026-06-18 起由 Antigravity CLI 取代；持有付费 Gemini 许可的企业客户仍可继续使用。 |
| [**openai/codex**](https://github.com/openai/codex) [![Star](https://img.shields.io/github/stars/openai/codex.svg?style=social&label=Star)](https://github.com/openai/codex) | 2025 年 | OpenAI 的本地终端编码智能体，Rust 实现。 |
| [**OpenHands/OpenHands**](https://github.com/OpenHands/OpenHands) [![Star](https://img.shields.io/github/stars/OpenHands/OpenHands.svg?style=social&label=Star)](https://github.com/OpenHands/OpenHands) | 2024 年 | 开源 SWE 智能体平台（原 OpenDevin），含沙箱化运行时。 |
| [**cline/cline**](https://github.com/cline/cline) [![Star](https://img.shields.io/github/stars/cline/cline.svg?style=social&label=Star)](https://github.com/cline/cline) | 2024 年 | VS Code 智能体，显式 Plan/Act 双阶段监督循环。 |
| [**block/goose**](https://github.com/block/goose) [![Star](https://img.shields.io/github/stars/block/goose.svg?style=social&label=Star)](https://github.com/block/goose) | 2025 年 | Block 开源、编辑器无关的智能体，MCP 风格扩展。 |
| [**charmbracelet/crush**](https://github.com/charmbracelet/crush) [![Star](https://img.shields.io/github/stars/charmbracelet/crush.svg?style=social&label=Star)](https://github.com/charmbracelet/crush) | 2025 年 | Go 编写的智能编码 TUI，多 LLM provider 抽象。 |
| [**RooCodeInc/Roo-Code**](https://github.com/RooCodeInc/Roo-Code) [![Star](https://img.shields.io/github/stars/RooCodeInc/Roo-Code.svg?style=social&label=Star)](https://github.com/RooCodeInc/Roo-Code) | 2024 年 | VS Code 多智能体开发团队，含 Architect/Coder/Reviewer 角色。 |
| [**bytedance/trae-agent**](https://github.com/bytedance/trae-agent) [![Star](https://img.shields.io/github/stars/bytedance/trae-agent.svg?style=social&label=Star)](https://github.com/bytedance/trae-agent) | 2025 年 | 字节跳动模块化、面向 SWE-bench 的软件工程智能体。 |
| [**github/copilot-cli**](https://github.com/github/copilot-cli) [![Star](https://img.shields.io/github/stars/github/copilot-cli.svg?style=social&label=Star)](https://github.com/github/copilot-cli) | 2026 年 | GitHub Copilot 正式发布的终端智能体 CLI，跨会话规划、构建、审查。 |
| [**badlogic/pi-mono**](https://github.com/badlogic/pi-mono) [![Star](https://img.shields.io/github/stars/badlogic/pi-mono.svg?style=social&label=Star)](https://github.com/badlogic/pi-mono) | 2025 年 8 月 | monorepo 编码智能体工具包——统一 LLM API、TUI + web UI；OpenClaw 嵌入此处的 `pi-coding-agent` SDK。 |
| [**deepseek-ai/deepseek-harness**](https://github.com/deepseek-ai/deepseek-harness) [![Star](https://img.shields.io/github/stars/deepseek-ai/deepseek-harness.svg?style=social&label=Star)](https://github.com/deepseek-ai/deepseek-harness) | 2026 年 | DeepSeek 自家的 agent harness，处于开发者预览阶段，建在插件化架构（Cordis）之上，主打"一切皆插件"。可经 npm 或从源码运行；围绕它的生态已经出现第三方 awesome 列表和以它为后端的项目。一款来自主要模型实验室的第一方 harness，值得与上面的厂商 CLI 对照阅读。MIT。 |
| [**zai-org/ZCode**](https://github.com/zai-org/ZCode) [![Star](https://img.shields.io/github/stars/zai-org/ZCode.svg?style=social&label=Star)](https://github.com/zai-org/ZCode) | 2026 年 9 月 | Z.ai 开源的 AI 编程工作台：桌面、浏览器和终端界面，构建在同一仓库中的 agent CLI 与运行时之上。Web 界面默认只监听本机地址，绑定其他地址时会生成访问令牌。插件是本地包，可添加 skills、自定义命令和 MCP 服务器。Apache-2.0。 |

### 智能体框架与编排

| 仓库 | 推出时间 | 重点 |
|:-----------|:-------|:------|
| [**context4ai/agent-graph**](https://github.com/context4ai/agent-graph/tree/387f80db65bf20a61bc666b4fa885200fcedad08) | 2026 年 | 以 v0.3.0 为核查版本，提供工作契约与基于事实的路由。对于定义了事实条件的非终结节点，即使已记录完成，条件不匹配时仍会被标为未验证；执行与事实真实性的验证仍由接入它的宿主系统负责。 |
| [**geekan/MetaGPT**](https://github.com/geekan/MetaGPT) [![Star](https://img.shields.io/github/stars/geekan/MetaGPT.svg?style=social&label=Star)](https://github.com/geekan/MetaGPT) | 2023 年 | 角色分工的多智能体软件公司模拟（ICLR 2024 oral）。 |
| [**microsoft/autogen**](https://github.com/microsoft/autogen) [![Star](https://img.shields.io/github/stars/microsoft/autogen.svg?style=social&label=Star)](https://github.com/microsoft/autogen) | 2023 年 | 微软研究院多智能体对话框架（COLM 2024）。 |
| [**microsoft/agent-framework**](https://github.com/microsoft/agent-framework) [![Star](https://img.shields.io/github/stars/microsoft/agent-framework.svg?style=social&label=Star)](https://github.com/microsoft/agent-framework) | 2025 年 | 微软整合 AutoGen 与 Semantic Kernel 的后继框架（2026 年 4 月发布 1.0）。在 BUILD 2026 上加入了内置的 agent harness（上下文压缩、文件记忆、shell）和 CodeAct：让模型把多次工具调用写成一段 Python 一次跑完，而不是一次只调一个。 |
| [**langchain-ai/langgraph**](https://github.com/langchain-ai/langgraph) [![Star](https://img.shields.io/github/stars/langchain-ai/langgraph.svg?style=social&label=Star)](https://github.com/langchain-ai/langgraph) | 2024 年 | 基于状态图的多智能体编排，含检查点。 |
| [**openai/openai-agents-python**](https://github.com/openai/openai-agents-python) [![Star](https://img.shields.io/github/stars/openai/openai-agents-python.svg?style=social&label=Star)](https://github.com/openai/openai-agents-python) | 2024 年 | OpenAI 轻量多智能体框架，含 handoff 与 guardrail。 |
| [**crewAIInc/crewAI**](https://github.com/crewAIInc/crewAI) [![Star](https://img.shields.io/github/stars/crewAIInc/crewAI.svg?style=social&label=Star)](https://github.com/crewAIInc/crewAI) | 2023 年 | 精简 Python 框架，角色化多智能体协作，独立于 LangChain。 |
| [**openai/symphony**](https://github.com/openai/symphony) [![Star](https://img.shields.io/github/stars/openai/symphony.svg?style=social&label=Star)](https://github.com/openai/symphony) | 2026 年 2 月 | OpenAI 的编排方案，用于隔离、自主的实现运行。 |
| [**ComposioHQ/agent-orchestrator**](https://github.com/ComposioHQ/agent-orchestrator) [![Star](https://img.shields.io/github/stars/ComposioHQ/agent-orchestrator.svg?style=social&label=Star)](https://github.com/ComposioHQ/agent-orchestrator) | 2025 年 | 并行 AI 智能体编排层，含 git worktree 隔离。 |
| [**omnigent-ai/omnigent**](https://github.com/omnigent-ai/omnigent) [![Star](https://img.shields.io/github/stars/omnigent-ai/omnigent.svg?style=social&label=Star)](https://github.com/omnigent-ai/omnigent) | 2026 年 6 月 | Databricks 开源的"元 harness"（Apache 2.0）：Claude Code、Codex、Cursor、OpenCode、Hermes、Pi 在同一个编排层背后成为可互换的后端，且可以在同一个会话里混用。策略引擎分三级叠加（server、agent、session，且更严的 session 规则先被检查），管辖 shell 命令、文件编辑、token 消耗和工具访问。原生沙箱在 Linux 用 bwrap、macOS 用 seatbelt；一次性云沙箱可跑在 Modal、E2B 或 Kubernetes 上。这是"策略与隔离应当位于 harness 之上、而不是塞进 prompt 里"最清晰的一个工业级论证。 |
| [**yylo-dev/yylo**](https://github.com/yylo-dev/yylo) [![Star](https://img.shields.io/github/stars/yylo-dev/yylo.svg?style=social&label=Star)](https://github.com/yylo-dev/yylo) | 2026 年 1 月 | 面向编码智能体的命令行编排工具：可重复的工作流，以及带回执记录的仓库改动；任务、验证、合并和发布就绪各有类型化的边界。在其 Advanced 模式下，每个任务在专用的分支和 worktree 中运行；以 Pi 和 Codex 作为子智能体引擎。MIT 许可。 |
| [**coleam00/Archon**](https://github.com/coleam00/Archon) [![Star](https://img.shields.io/github/stars/coleam00/Archon.svg?style=social&label=Star)](https://github.com/coleam00/Archon) | 2025 年 2 月 | 确定性 harness——YAML 定义工作流与执行审计跟踪。 |
| [**bytedance/deer-flow**](https://github.com/bytedance/deer-flow) [![Star](https://img.shields.io/github/stars/bytedance/deer-flow.svg?style=social&label=Star)](https://github.com/bytedance/deer-flow) | 2026 年 | 字节跳动的长程"SuperAgent" harness：子智能体、记忆、沙箱、技能与消息网关；基于 LangGraph/LangChain 的彻底重写。 |
| [**QwenLM/Qwen-Agent**](https://github.com/QwenLM/Qwen-Agent) [![Star](https://img.shields.io/github/stars/QwenLM/Qwen-Agent.svg?style=social&label=Star)](https://github.com/QwenLM/Qwen-Agent) | 2023 年 | 阿里 Qwen 的 agent 框架：function calling、MCP、Docker code interpreter、RAG；Qwen Chat 的后端。 |
| [**TencentCloudADP/youtu-agent**](https://github.com/TencentCloudADP/youtu-agent) [![Star](https://img.shields.io/github/stars/TencentCloudADP/youtu-agent.svg?style=social&label=Star)](https://github.com/TencentCloudADP/youtu-agent) | 2025 年 | 腾讯云基于 openai-agents SDK 的框架；用 YAML 定义 agent、可自动生成配置，并提供 Claude Code 式的 skills。 |
| [**coze-dev/coze-studio**](https://github.com/coze-dev/coze-studio) [![Star](https://img.shields.io/github/stars/coze-dev/coze-studio.svg?style=social&label=Star)](https://github.com/coze-dev/coze-studio) | 2025 年 | 字节跳动 Coze 的开源版：可视化的 no-code/low-code 平台，用于搭建、调试与部署 agent 和工作流。 |

### 记忆与持久化上下文

| 资源 | 日期 / 版本 | 重点 |
|:-----------|:-------|:------|
| [**Codex 实验性上下文管理**](https://learn.chatgpt.com/docs/config-file/config-reference) | CLI 0.153.0，2026 年 9 月 3 日 | 可选的工作上下文管理方式，使用笔记、可检索历史和 `new_context`。其 token-budget 路径通过新建窗口跳过模型或服务端摘要，同时保留 compaction 生命周期。账户、后端、版本及后续模型支持限制见[来源说明](./docs/agent-design-space-source-notes_zh.md#source-codex-context-management)。 |
| [**Codex 本地记忆**](https://learn.chatgpt.com/docs/customization/memories)、[memory v2 source](https://github.com/openai/codex/commit/2cbbf0c9b542a36a1c3284b5e804917635b6f666) | 2026 年 9 月至 10 月核查 | 启用后，从符合条件的旧会话中后台提取、整合经验，通过简短索引按需读取详细记忆。本地存储与 ChatGPT 网页记忆、实验性工作上下文笔记分别管理；[v0.153.4 也仍保留摘要式压缩](https://github.com/openai/codex/blob/3d2ee51ca2d5db578f328aa75e20aa22c0197c9a/codex-rs/core/src/compact.rs)。 9 月 8 日的主线实现增加独立 memory v2 存储和可选双版本写入，由所选版本提供上下文。v2 在记忆提取的输入预算内，优先保留用户消息及其对智能体提问的回答。其提示要求将任务纠正保留在对应任务范围内。自稳定版 [0.155.0](https://github.com/openai/codex/releases/tag/rust-v0.155.0)（9 月 17 日）起，记忆配置接受 `version` 和 `dual_write`；直到 0.160.0 默认仍为 v1；截至 2026 年 10 月 6 日，配置参考文档未列出这两个键。 |
| [Claude Tag memory scope](https://github.com/anthropics/claude-code/releases/tag/v2.1.268) | 2026 年 9 月 10 日 | 每个公共频道保留自己的便笺，Claude 不再召回其他公共频道的便笺；工作区便笺仍然共享。这定义了哪些便笺可进入当前频道的上下文。 |
| [Hermes background memory review](https://github.com/NousResearch/hermes-agent/releases/tag/v2026.9.11) | v0.21.2, 2026 年 9 月 11 日 | 在无人值守的后台复盘中，内置 memory 工具将 replace/remove，以及包含这些操作的整批请求，转为待审批提案，并阻止该次调用执行修改。提案采用尽力而为的方式存储。 |
| [**mem0ai/mem0**](https://github.com/mem0ai/mem0) [![Star](https://img.shields.io/github/stars/mem0ai/mem0.svg?style=social&label=Star)](https://github.com/mem0ai/mem0) | 2024 年 | 生产级记忆层，含 LoCoMo 与 LongMemEval 基准（arXiv:2504.19413）。 |
| [**letta-ai/letta**](https://github.com/letta-ai/letta) [![Star](https://img.shields.io/github/stars/letta-ai/letta.svg?style=social&label=Star)](https://github.com/letta-ai/letta) | 2023 年 | 有状态智能体平台，OS 式分层记忆分页（原 MemGPT，COLM 2024）。 |
| [**MemPalace/mempalace**](https://github.com/MemPalace/mempalace) [![Star](https://img.shields.io/github/stars/MemPalace/mempalace.svg?style=social&label=Star)](https://github.com/MemPalace/mempalace) | 2026 年 | AI 智能体的本地优先记忆系统。 |
| [**vshulcz/deja-vu**](https://github.com/vshulcz/deja-vu) [![Star](https://img.shields.io/github/stars/vshulcz/deja-vu.svg?style=social&label=Star)](https://github.com/vshulcz/deja-vu) | 2026 年 7 月 | 单个本地 Go 二进制里的跨 agent 记忆：索引 17 款代码智能体早已写到磁盘上的会话历史，用词法检索取回过往工作，不用 LLM 也不用嵌入服务。MIT。 |
| [**pax-beehive/paxm**](https://github.com/pax-beehive/paxm) [![Star](https://img.shields.io/github/stars/pax-beehive/paxm.svg?style=social&label=Star)](https://github.com/pax-beehive/paxm) | 2026 年 7 月 | 与厂商无关的记忆适配器，把决策与约定带过 Codex、Claude Code、OpenCode、Pi 与 MCP agent。起步用本地 SQLite，可换成 Zep、Mem0 等后端而不被某一家锁定。Apache-2.0。 |

### Skill 与 Harness 扩展

| 仓库 | 推出时间 | 重点 |
|:-----------|:-------|:------|
| [**addyosmani/agent-skills**](https://github.com/addyosmani/agent-skills) [![Star](https://img.shields.io/github/stars/addyosmani/agent-skills.svg?style=social&label=Star)](https://github.com/addyosmani/agent-skills) | 2025 年 | 22 个生命周期 skill 配套斜杠命令（`/spec`、`/plan`、`/build`、`/test`、`/review`、`/ship`）。 |
| [**obra/superpowers**](https://github.com/obra/superpowers) [![Star](https://img.shields.io/github/stars/obra/superpowers.svg?style=social&label=Star)](https://github.com/obra/superpowers) | 2025 年 | 跨 harness（Claude Code、OpenCode、Codex）的强制工作流 skill 框架。 |
| [**mattpocock/skills**](https://github.com/mattpocock/skills) [![Star](https://img.shields.io/github/stars/mattpocock/skills.svg?style=social&label=Star)](https://github.com/mattpocock/skills) | 2026 年 | 作者本人日常使用的 `.claude/skills` 技能集，面向真实工程——包含可组合的 TDD、diagnose、to-issues/to-prd 等技能；与具体模型无关，适用于 Claude Code、Codex 等编码智能体。 |
| [**multica-ai/andrej-karpathy-skills**](https://github.com/multica-ai/andrej-karpathy-skills) [![Star](https://img.shields.io/github/stars/multica-ai/andrej-karpathy-skills.svg?style=social&label=Star)](https://github.com/multica-ai/andrej-karpathy-skills) | 2026 年 | 单个 CLAUDE.md，凝练 Andrej Karpathy 关于 LLM 编码的四条规则（先想后写、简洁优先、外科手术式改动、目标驱动执行）；可作为插件安装或按项目添加。 |
| [**lsdefine/GenericAgent**](https://github.com/lsdefine/GenericAgent) [![Star](https://img.shields.io/github/stars/lsdefine/GenericAgent.svg?style=social&label=Star)](https://github.com/lsdefine/GenericAgent) | 2025 年 | 极简自演化自主智能体框架——9 个原子工具加约 100 行 ReAct 循环。 |
| [**Leutenegger/book-to-skill**](https://github.com/Leutenegger/book-to-skill) [![Star](https://img.shields.io/github/stars/Leutenegger/book-to-skill.svg?style=social&label=Star)](https://github.com/Leutenegger/book-to-skill) | 2026 年 8 月 | 把一本技术书的 PDF 变成一个结构化 Agent Skill（按章分文件、心智模型、术语表与模式指南，按需加载），而不是把全文塞进上下文，号称 token 用量少 24 到 51 倍。面向 Claude Code、Copilot CLI 与 Amp。MIT。 |
| [**xyiqq/skilldoctor**](https://github.com/xyiqq/skilldoctor) [![Star](https://img.shields.io/github/stars/xyiqq/skilldoctor.svg?style=social&label=Star)](https://github.com/xyiqq/skilldoctor) | 2026 年 8 月 | 面向 Agent Skills 的 CI 质量闸：校验 SKILL.md 规范，审计提示注入风险与硬编码密钥，并检查在 Claude Code、Cursor、Codex 与 OpenCode 间的兼容性。是最早为技能生态构建的 lint 工具之一。MIT。 |

### MCP 生态

| 仓库 | 推出时间 | 重点 |
|:-----------|:-------|:------|
| [**PrefectHQ/fastmcp**](https://github.com/prefecthq/fastmcp) [![Star](https://img.shields.io/github/stars/prefecthq/fastmcp.svg?style=social&label=Star)](https://github.com/prefecthq/fastmcp) | 2024 年 | 构建 MCP 服务器与客户端的 Python 框架，事实上的 SDK。 |
| [**upstash/context7**](https://github.com/upstash/context7) [![Star](https://img.shields.io/github/stars/upstash/context7.svg?style=social&label=Star)](https://github.com/upstash/context7) | 2025 年 | 为 LLM 与 AI 编辑器提供实时库文档的 MCP 服务器。 |
| [**microsoft/playwright-mcp**](https://github.com/microsoft/playwright-mcp) [![Star](https://img.shields.io/github/stars/microsoft/playwright-mcp.svg?style=social&label=Star)](https://github.com/microsoft/playwright-mcp) | 2024 年 | 微软官方 MCP 服务器，使用 accessibility tree 快照。 |
| [**modelcontextprotocol/experimental-ext-interceptors**](https://github.com/modelcontextprotocol/experimental-ext-interceptors) [![Star](https://img.shields.io/github/stars/modelcontextprotocol/experimental-ext-interceptors.svg?style=social&label=Star)](https://github.com/modelcontextprotocol/experimental-ext-interceptors) | 2026 年 | 拦截器扩展提案（SEP-2624）的多语言参考实现。若被接受，对 MCP 请求做中间件式的检查与改写将成为协议级原语，而不再是各家 host 各自实现的钩子系统。目前明确标注为实验性，并非已被接受的官方 MCP 扩展。 |

### 专业领域智能体

| 仓库 | 推出时间 | 重点 |
|:-----------|:-------|:------|
| [**666ghj/MiroFish**](https://github.com/666ghj/MiroFish) [![Star](https://img.shields.io/github/stars/666ghj/MiroFish.svg?style=social&label=Star)](https://github.com/666ghj/MiroFish) | 2026 年 3 月 | 多智能体群体智能模拟引擎。 |
| [**multica-ai/multica**](https://github.com/multica-ai/multica) [![Star](https://img.shields.io/github/stars/multica-ai/multica.svg?style=social&label=Star)](https://github.com/multica-ai/multica) | 2026 年 | 用于任务分配和技能复合的托管智能体平台。 |
| [**HKUDS/nanobot**](https://github.com/HKUDS/nanobot) [![Star](https://img.shields.io/github/stars/HKUDS/nanobot.svg?style=social&label=Star)](https://github.com/HKUDS/nanobot) | 2026 年 2 月 | 来自 HKU-DS 的超轻量级个人 AI 智能体。 |
| [**HKUDS/OpenHarness**](https://github.com/HKUDS/OpenHarness) [![Star](https://img.shields.io/github/stars/HKUDS/OpenHarness.svg?style=social&label=Star)](https://github.com/HKUDS/OpenHarness) | 2026 年 4 月 | 带有内置个人智能体（Ohmo）的开放智能体 harness；harness 架构的学术参考点。 |
| [**karpathy/autoresearch**](https://github.com/karpathy/autoresearch) [![Star](https://img.shields.io/github/stars/karpathy/autoresearch.svg?style=social&label=Star)](https://github.com/karpathy/autoresearch) | 2026 年 3 月 | Andrej Karpathy 出品：在单 GPU 上自动运行 nanochat 训练研究的 AI 智能体循环。 |
| [**HKUDS/CLI-Anything**](https://github.com/HKUDS/CLI-Anything) [![Star](https://img.shields.io/github/stars/HKUDS/CLI-Anything.svg?style=social&label=Star)](https://github.com/HKUDS/CLI-Anything) | 2026 年 3 月 | "Making ALL Software Agent-Native"——把任意软件包装成智能体可调用工具。 |
| [**Panniantong/Agent-Reach**](https://github.com/Panniantong/Agent-Reach) [![Star](https://img.shields.io/github/stars/Panniantong/Agent-Reach.svg?style=social&label=Star)](https://github.com/Panniantong/Agent-Reach) | 2026 年 2 月 | 一个 CLI 让智能体读写 Twitter、Reddit、YouTube、GitHub、Bilibili、小红书。 |
| [**agentscope-ai/QwenPaw**](https://github.com/agentscope-ai/QwenPaw) [![Star](https://img.shields.io/github/stars/agentscope-ai/QwenPaw.svg?style=social&label=Star)](https://github.com/agentscope-ai/QwenPaw) | 2026 年 2 月 | AgentScope 团队的个人 AI 助手。 |
| [**cft0808/edict**](https://github.com/cft0808/edict) [![Star](https://img.shields.io/github/stars/cft0808/edict.svg?style=social&label=Star)](https://github.com/cft0808/edict) | 2026 年 2 月 | 基于 OpenClaw 的多智能体编排，模拟唐代三省六部制（9 个专门智能体 + 实时 dashboard + 完整审计）。 |

<p align="right"><a href="#深入理解-claude-code">↑ 返回顶部</a></p>

</details>

---

<a href="https://star-history.dera.page/#VILA-Lab/Dive-into-Claude-Code">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://star-history.dera.page/svg?repos=VILA-Lab/Dive-into-Claude-Code&type=Date&theme=dark" />
    <source media="(prefers-color-scheme: light)" srcset="https://star-history.dera.page/svg?repos=VILA-Lab/Dive-into-Claude-Code&type=Date" />
    <img alt="Star History Chart" src="https://star-history.dera.page/svg?repos=VILA-Lab/Dive-into-Claude-Code&type=Date" />
  </picture>
</a>

## 参与贡献

欢迎补充资源。这个方向更新很快，难免有遗漏。

如果发现值得收录的论文、博客、仓库、评测或安全分析，欢迎直接提 pull request 或 issue。

## 引用

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


## 许可证

本作品采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可。
