# 深入理解 Claude Code

<p align="center">
  <img src="./assets/main_structure.png" width="85%" alt="Claude Code 高层系统结构">
</p>

<p align="center">
  <a href="./paper/Dive_into_Claude_Code.pdf"><img src="https://img.shields.io/badge/Paper-PDF-blue.svg?logo=adobeacrobatreader&logoColor=white" alt="论文"></a>
  <a href="https://arxiv.org/abs/2604.14228"><img src="https://img.shields.io/badge/arXiv-2604.14228-b31b1b.svg" alt="arXiv"></a>
  <a href="./LICENSE"><img src="https://img.shields.io/badge/License-CC--BY--NC--SA--4.0-lightgrey.svg" alt="许可证"></a>
  <a href="https://github.com/VILA-Lab/Dive-into-Claude-Code/stargazers"><img src="https://img.shields.io/github/stars/VILA-Lab/Dive-into-Claude-Code?style=social" alt="星标数"></a>
</p>

> **对 Claude Code（v2.1.88，约 1,900 个 TypeScript 文件，约 512K 行代码）进行全面源代码级架构分析，辅以社区分析精选集、面向智能体构建者的设计空间指南，以及跨系统比较。**

> [!TIP]
> **TL;DR** -- Claude Code 代码库中只有 1.6% 是 AI 决策逻辑。其他 98.4% 是确定性基础设施——权限门控、上下文管理、工具路由和恢复逻辑。智能体循环是一个简单的 while 循环；真正的工程复杂度存在于围绕它的系统中。本仓库解剖了这一架构，并为任何构建 AI 智能体系统的人提炼出可行的设计指导。

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
- [🧠 上下文与内存](#上下文与内存)
- [👥 子智能体委托](#子智能体委托)
- [💾 会话持久化](#会话持久化)

**论文之外**

- [🛠️ 构建你自己的 AI 智能体：设计指南](#构建你自己的-ai-智能体设计指南)
- [🌐 社区项目与研究](#社区项目与研究)
- [🚀 其他值得关注的 AI 智能体项目](#其他值得关注的-ai-智能体项目)
- [🔖 引用](#引用)

---

## 关键亮点

- **98.4% 基础设施，1.6% AI** -- 智能体循环不过是一个 while 循环；真正的复杂性在于权限门控、上下文管理和恢复逻辑。
- **5 个价值观 → 13 条原则 → 落地实现** -- 每一条设计决策都能追溯到人类权威、安全、可靠性、能力扩展和适应性。
- **深度防御存在共享故障模式** -- 7 层安全防护，但彼此共享性能约束。50+ 子命令绕过安全分析。
- **4 个 CVE 揭示了预信任窗口** -- 扩展在信任对话框出现**之前**就已执行。
- **跨切面 Harness 难以被复现** -- 循环容易复制；但钩子、分类器、压缩和隔离机制则不然。

---

## 阅读指南

| **智能体构建者** | [构建你自己的智能体](./docs/build-your-own-agent.md) | [架构深度剖析](./docs/architecture.md) |
| **安全研究员** | [安全与权限](#安全与权限) | [架构：安全层](./docs/architecture.md#七个独立安全层) |
| **产品经理** | [关键亮点](#关键亮点) | [价值观与原则](#价值观与设计原则) |
| **研究人员** | [完整论文 (arXiv)](https://arxiv.org/abs/2604.14228) | [社区资源](#社区项目与研究) |

`1,884 个文件` · `约 512K 行` · `v2.1.88` · `7 个安全层` · `5 个压缩阶段` · `54 个工具` · `27 个钩子事件` · `4 个扩展机制` · `7 个权限模式`

---

<details open>
<summary><h2>架构总览</h2></summary>

Claude Code 回答了**每个生产级编码智能体必须面对的四个设计问题**：

| 问题 | Claude Code 的答案 |
|:---------|:---------------------|
| 推理放在哪里？ | 模型负责推理；harness 负责强制执行。约 1.6% AI，98.4% 基础设施。 |
| 有多少个执行引擎？ | 一个 `queryLoop` 用于所有接口（CLI、SDK、IDE）。 |
| 默认安全姿态？ | 拒绝优先：拒绝 > 询问 > 允许。最严格的规则优先。 |
| 绑定资源约束？ | 约 200K（旧模型）/ 1M（Claude 4.6 系列）上下文窗口。每次模型调用前有 5 层压缩。 |

系统分解为**7 个组件**（用户 → 接口 → 智能体循环 → 权限系统 → 工具 → 状态与持久化 → 执行环境），跨越**5 个架构层**。

<p align="center">
  <img src="./assets/layered_architecture.png" width="100%" alt="5 层子系统分解">
</p>

> [!NOTE]
> 完整的架构深度剖析——7 个安全层、9 步轮次管道、5 层压缩等——请参阅**[docs/architecture.md](./docs/architecture.md)**。

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

</details>

---

<details open>
<summary><h2>价值观与设计原则</h2></summary>

架构从**5 个人类价值观**追溯到**13 条设计原则**再到实现：

| 价值观 | 核心思想 |
|:------|:----------|
| **人类决策权威** | 人类通过主级 hierarchy 保持控制。当 93% 的提示批准率揭示批准疲劳时，响应是重构边界，而不是更多警告。 |
| **安全、安保、隐私** | 系统即使在人类警惕性下降时也保护安全。7 个独立安全层。 |
| **可靠执行** | 做应该做的事。收集-行动-验证循环。优雅恢复。 |
| **能力放大** | "一个 Unix 工具，而不是产品。"98.4% 是使模型能够运作的确定性基础设施。 |
| **上下文适应性** | CLAUDE.md hierarchy、渐进式可扩展性、随时间演变的信任轨迹。 |

<details>
<summary><b>13 条设计原则</b></summary>

| 原则 | 设计问题 |
|:----------|:----------------|
| 拒绝优先并人工升级 | 未知操作应该被允许、阻止还是升级？ |
| 渐进式信任光谱 | 固定权限级别，还是用户随时间推移的光谱？ |
| 深度防御 | 单个安全边界，还是多个重叠的？ |
| 外部化可编程策略 | 硬编码策略，还是具有生命周期钩子的外部化配置？ |
| 上下文作为稀缺资源 | 单次截断还是渐进式管道？ |
| 追加式持久状态 | 可变状态、快照还是追加式日志？ |
| 最小脚手架，最大 harness | 投资脚手架还是运营基础设施？ |
| 价值观优先于规则 | 刚性程序还是具有确定性guardrails的上下文判断？ |
| 可组合的多机制可扩展性 | 一个 API 还是不同成本的分层机制？ |
| 可逆性加权风险评估 | 全部相同监督，还是对可逆操作更轻？ |
| 透明的文件配置和内存 | 不透明的数据库、embeddings，还是用户可见的文件？ |
| 隔离的子智能体边界 | 共享上下文/权限，还是隔离？ |
| 优雅恢复与韧性 | 硬失败还是静默恢复？ |

</details>

论文还应用了**第六个评估透镜**——长期能力保持——引用了证据，表明在 AI 辅助条件下开发的开发者在理解测试中得分低 17%。

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>智能体查询循环</h2></summary>

<p align="center">
  <img src="./assets/iteration.png" width="60%" alt="运行时轮次流程">
</p>

核心是一个 **ReAct 模式的 while 循环**：组装上下文 → 调用模型 → 分派工具 → 检查权限 → 执行 → 重复。实现为一个产生流式事件的 `AsyncGenerator`。

**每次模型调用前**，五个压缩塑形器按顺序执行（从最便宜的开始）：预算削减 → 剪枝 → 微压缩 → 上下文折叠 → 自动压缩。

**每轮 9 步管道：** 设置解析 → 状态初始化 → 上下文组装 → 5 个预模型塑形器 → 模型调用 → 工具分派 → 权限门控 → 工具执行 → 停止条件

**两条执行路径：**
- `StreamingToolExecutor` -- 工具流入时即开始执行（延迟优化）
- 后备 `runTools` -- 将工具分类为并发安全或互斥

**故障恢复：** 最大输出 token 升级（3 次重试）、反应式压缩（每轮一次）、提示过长处理、流式后备、后备模型

**5 个停止条件：** 无工具调用、最大轮次、上下文溢出、钩子干预、显式中止

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

</details>

---

<details open>
<summary><h2>安全与权限</h2></summary>

<p align="center">
  <img src="./assets/permission.png" width="75%" alt="权限门控">
</p>

**7 个权限模式**构成渐进式信任光谱：`plan` → `default` → `acceptEdits` → `auto`（ML 分类器）→ `dontAsk` → `bypassPermissions`（+ 内部 `bubble`）。

**拒绝优先**：广泛的拒绝*始终*覆盖狭窄的允许。**7 个独立安全层**，从工具预过滤到 shell 沙箱再到钩子拦截。权限在恢复时**永不恢复**——每次会话都重新建立信任。

> [!WARNING]
> **共享故障模式：** 深度防御在层级共享约束时会降级。per-subcommand 解析导致事件循环饥饿——超过 50 个子命令的命令为防止 REPL 冻结，会完全绕过安全分析。

<details>
<summary><b>更多详情：授权管道、auto 模式分类器、CVE</b></summary>

**授权管道：** 预过滤（剥离被拒绝的工具）→ PreToolUse 钩子 → 拒绝优先规则评估 → 权限处理程序（4 个分支：coordinator、swarm worker、speculative 分类器、interactive）

**Auto 模式分类器**（`yoloClassifier.ts`）：使用内部/外部权限模板的单独 LLM 调用。两阶段：快速过滤 + 思维链。

**预信任执行窗口：** 2 个修补的 CVE 共享此根本原因——钩子和 MCP 服务器在初始化期间*在*信任对话框出现之前执行，创建了一个结构上 privileged 的攻击窗口，位于拒绝优先管道之外。

</details>

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>可扩展性</h2></summary>

<p align="center">
  <img src="./assets/extensibility.png" width="85%" alt="三个注入点：组装、模型、执行">
</p>

**四个渐进式上下文成本机制：** 钩子（零成本）→ Skills（低成本）→ 插件（中成本）→ MCP（高成本）。智能体循环中的三个注入点：**assemble()**（模型看到的内容）、**model()**（它能触及的内容）、**execute()**（操作是否/如何运行）。

**工具池组装**（5 步）：基础枚举（最多 54 个工具）→ 模式过滤 → 拒绝预过滤 → MCP 集成 → 去重

**27 个钩子事件**，跨越 5 个类别，4 种执行类型（shell、LLM 评估、webhook、subagent 验证器）

**插件清单**支持 10 种组件类型：命令、智能体、skills、钩子、MCP 服务器、LSP 服务器、输出样式、渠道、设置、用户配置

**Skills：** SKILL.md 包含 15+ 个 YAML frontmatter 字段。关键区别——SkillTool 注入到当前上下文；AgentTool 生成隔离上下文。

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

</details>

---

<details open>
<summary><h2>上下文与内存</h2></summary>

<p align="center">
  <img src="./assets/context.png" width="95%" alt="上下文构建">
</p>

**9 个有序来源**构建上下文窗口。CLAUDE.md 指令作为**用户上下文**（概率合规）传递，而非系统提示（确定性）。内存是**基于文件的**（无向量数据库）——完全可检验、可编辑、可版本控制。

**4 级 CLAUDE.md hierarchy：** 托管（`/etc/`）→ 用户（`~/.claude/`）→ 项目（`CLAUDE.md`、`.claude/rules/`）→ 本地（`CLAUDE.local.md`，gitignore）

**5 层压缩**（渐进式惰性降级）：预算削减 → 剪枝 → 微压缩 → 上下文折叠（读取时投影，非破坏性）→ 自动压缩（完整模型摘要，最终手段）

**内存检索：** 基于 LLM 的内存文件头扫描，最多选择 5 个相关文件。无 embeddings，无向量相似度。

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>子智能体委托</h2></summary>

<p align="center">
  <img src="./assets/subagent.png" width="90%" alt="子智能体架构">
</p>

**6 个内置类型**（Explore、Plan、通用、Guide、验证、状态行）+ 通过 `.claude/agents/*.md` 的自定义智能体。**侧链转录稿**：只有摘要返回给父级（父级的上下文受到*保护*免受子智能体冗长信息干扰）。三种隔离模式：worktree、remote、in-process。通过 POSIX `flock()` 协调。

**SkillTool vs AgentTool：** SkillTool 注入到当前上下文（廉价）。AgentTool 生成隔离上下文（昂贵，但防止上下文膨胀）。

**权限覆盖：** 子智能体 `permissionMode` 生效，除非父级处于 `bypassPermissions`/`acceptEdits`/`auto`（显式用户决策始终优先）。

**自定义智能体：** YAML frontmatter 支持 tools、disallowedTools、model、effort、permissionMode、mcpServers、hooks、maxTurns、skills、memory scope、background flag、isolation mode。

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

</details>

---

<details>
<summary><h2>会话持久化</h2></summary>

<p align="center">
  <img src="./assets/session_compact.png" width="75%" alt="会话持久化与上下文压缩">
</p>

三个通道：追加式 JSONL 转录稿、全局提示历史、子智能体侧链。**恢复时权限永不恢复**——每次会话都重新建立信任。设计偏好**审计能力超过查询能力**。

**链式补丁：** 压缩边界记录 `headUuid`/`anchorUuid`/`tailUuid`。会话加载器在读取时修补消息链。磁盘上无破坏性编辑。

**检查点：** `--rewind-files` 的文件历史检查点，存储在 `~/.claude/file-history/<sessionId>/`。

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

</details>

---

## 构建你自己的 AI 智能体：设计指南

> 不是编码教程。是关于**你必须做出的设计决策**的指南，源自架构分析。

每个生产级智能体都必须导航这些决策：

| 决策 | 问题 | 关键洞察 |
|:---------|:-------------|:------------|
| [**推理放置位置**](./docs/build-your-own-agent.md#决策-1-推理放在哪里) | 模型 vs. harness 中放置多少逻辑？ | 随着模型能力趋同，harness 成为差异化因素。 |
| [**安全姿态**](./docs/build-your-own-agent.md#决策-2-你的安全姿态是什么) | 如何防止智能体做有害事情？ | 深度防御在层共享故障模式时失败。 |
| [**上下文管理**](./docs/build-your-own-agent.md#决策-3-你如何管理上下文) | 模型看到什么？ | 从第一天就为上下文稀缺设计。渐进式 > 单次通过。 |
| [**可扩展性**](./docs/build-your-own-agent.md#决策-4-你如何处理可扩展性) | 扩展如何插入？ | 并非所有扩展都需要消耗上下文 token。 |
| [**子智能体架构**](./docs/build-your-own-agent.md#决策-5-子智能体如何工作) | 共享还是隔离上下文？ | plan 模式下的智能体团队成本约 7× token。子智能体仅摘要返回防止上下文膨胀。 |
| [**会话持久化**](./docs/build-your-own-agent.md#决策-6-会话如何持久化) | 什么会延续？ | 恢复时绝不恢复权限。审计能力 > 查询能力。 |

**阅读完整指南：[docs/build-your-own-agent.md](./docs/build-your-own-agent.md)**

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

---

## 社区项目与研究

Claude Code 架构周围的 repos、重实现和学术论文的精选图。

### 官方 Anthropic 资源

贯穿论文的主要参考资料——Anthropic 自己的工程和研究出版物，以及产品文档。

#### 研究与工程博客

| 文章 | 主题 |
|:--------|:------|
| [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) | 基础：简单的可组合模式优于重型框架。 |
| [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | 上下文策划和 token 预算管理。 |
| [Harness Design for Long-Running Application Development](https://anthropic.com/engineering/harness-design-long-running-apps) | 自主全栈开发的 harness 架构；多智能体模式。 |
| [Claude Code Auto Mode: A Safer Way to Skip Permissions](https://www.anthropic.com/engineering/claude-code-auto-mode) | ML 分类器批准自动化；93% 批准率发现的来源。 |
| [Beyond Permission Prompts: Making Claude Code More Secure and Autonomous](https://www.anthropic.com/engineering/claude-code-sandboxing) | 基于沙箱的安全；权限提示减少 84%。 |
| [Measuring AI Agent Autonomy in Practice](https://anthropic.com/research/measuring-agent-autonomy) | 纵向使用：自动批准率随经验从约 20% 增长到 40%+。 |
| [Our Framework for Developing Safe and Trustworthy Agents](https://www.anthropic.com/news/our-framework-for-developing-safe-and-trustworthy-agents) | 负责任智能体部署的治理框架。 |
| [Scaling Managed Agents: Decoupling the Brain from the Hands](https://www.anthropic.com/engineering/managed-agents) | 分离推理、执行和会话的托管服务架构。 |

#### 产品文档

| 文档 | 主题 |
|:---------|:------|
| [How Claude Code Works](https://code.claude.com/docs/en/how-claude-code-works) | 智能体循环、工具和终端自动化的官方概述。 |
| [Permissions](https://code.claude.com/docs/en/permissions) | 分层权限系统、模式、细粒度规则。 |
| [Hooks](https://code.claude.com/docs/en/hooks) | 27 事件钩子参考、执行模型、生命周期事件。 |
| [Memory](https://code.claude.com/docs/en/memory) | CLAUDE.md hierarchy、自动记忆、学习偏好。 |
| [Sub-agents](https://code.claude.com/docs/en/sub-agents) | 专用隔离助手、自定义提示、工具访问。 |

### 架构分析

对 Claude Code 内部设计的深度剖析。

| 仓库 | 描述 |
|:-----------|:------------|
| [**ComeOnOliver/claude-code-analysis**](https://github.com/ComeOnOliver/claude-code-analysis) | 综合反向工程：源树结构、模块边界、工具清单和架构模式。 |
| [**alejandrobalderas/claude-code-from-source**](https://github.com/alejandrobalderas/claude-code-from-source) | 18 章技术书（约 400 页）。全部原创伪代码，无专有源代码。 |
| [**liuup/claude-code-analysis**](https://github.com/liuup/claude-code-analysis) | 中文深度剖析——启动流程、查询主循环、MCP 集成、多智能体架构。 |
| [**sanbuphy/claude-code-source-code**](https://github.com/sanbuphy/claude-code-source-code) | 四语种分析（EN/JA/KO/ZH）——多领域报告，涵盖遥测、代号、KAIROS、未发布的工具。 |
| [**cablate/claude-code-research**](https://github.com/cablate/claude-code-research) | 关于内部结构、Agent SDK 和相关工具的独立研究。 |
| [**Yuyz0112/claude-code-reverse**](https://github.com/Yuyz0112/claude-code-reverse) | 可视化 Claude Code 的 LLM 交互——日志解析器和可视化工具，追踪提示、工具调用和压缩。 |

### 开源重实现

干净室重写和可构建的研究分支。

| 仓库 | 描述 |
|:-----------|:------------|
| [**chauncygu/collection-claude-code-source-code**](https://github.com/chauncygu/collection-claude-code-source-code) | 社区 Claude Code 源 artifacts 的元收集——包括 claw-code（Rust 移植）、nano-claude-code（Python）和提取的原始源存档。 |
| [**777genius/claude-code-working**](https://github.com/777genius/claude-code-working) | 可运行的重构 CLI。使用 Bun，450+ chunk 文件，31 个特性标志 polyfill。 |
| [**T-Lab-CUHKSZ/claude-code**](https://github.com/T-Lab-CUHKSZ/claude-code) | CUHK-深圳可构建研究分支——从原始 TypeScript 快照重建构建系统。 |
| [**ruvnet/open-claude-code**](https://github.com/ruvnet/open-claude-code) | 夜间自动反编译重建——903+ 测试、25 个工具、4 个 MCP 传输、6 个权限模式。 |
| [**Enderfga/openclaw-claude-code**](https://github.com/Enderfga/openclaw-claude-code) | OpenClaw 插件——Claude/Codex/Gemini/Cursor 的统一 ISession 接口。多智能体委员会。 |
| [**memaxo/claude_code_re**](https://github.com/memaxo/claude_code_re) | 从精简包反向工程——对公开分发的 cli.js 文件的反混淆。 |

### 指南与学习

教程和动手学习路径。

| 仓库 | 描述 |
|:-----------|:------------|
| [**shareAI-lab/learn-claude-code**](https://github.com/shareAI-lab/learn-claude-code) | "Bash 就是你需要的一切"——19 章 0 到 1 课程，包含可运行的 Python 智能体、网络平台。ZH/EN/JA。 |
| [**FlorianBruniaux/claude-code-ultimate-guide**](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) | 从初学者到高级用户的指南，包含生产就绪模板、智能体工作流指南和速查表。 |
| [**affaan-m/everything-claude-code**](https://github.com/affaan-m/everything-claude-code) | 智能体 harness 优化——skills、本能、记忆、安全和研究优先开发。50K+ 星标。 |

### 博客文章与技术文章

| 文章 | 它的价值所在 |
|:--------|:---------------------|
| [Marco Kotrotsos — "Claude Code Internals"（15 部分系列）](https://kotrotsos.medium.com/claude-code-internals-part-1-high-level-architecture-9881c68c799f) | 泄漏前最系统的分析。架构、智能体循环、权限、子智能体、MCP、遥测。 |
| [Alex Kim — "The Claude Code Source Leak"](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/) | 反蒸馏机制、挫折检测、Undercover Mode、每天约 25 万次浪费的 API 调用。 |
| [Haseeb Qureshi — 跨智能体架构比较](https://gist.github.com/Haseeb-Qureshi/2213cc0487ea71d62572a645d7582518) | Claude Code vs Codex vs Cline vs OpenCode——架构级比较。 |
| [George Sung — "Tracing Claude Code's LLM Traffic"](https://medium.com/@georgesung/tracing-claude-codes-llm-traffic-agentic-loop-sub-agents-tool-use-prompts-7796941806f5) | 完整系统提示和完整 API 日志。发现双模型使用（Opus + Haiku）。 |
| [Agiflow — "Reverse Engineering Prompt Augmentation"](https://agiflow.io/blog/claude-code-internals-reverse-engineering-prompt-augmentation/) | 5 种提示增强机制，由实际网络追踪支持。 |
| [Engineer's Codex — "Diving into the Source Code Leak"](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code) | 模块化系统提示、约 40 个工具、大型查询/工具子系统、反蒸馏。对广大受众友好。 |
| [MindStudio — "Three-Layer Memory Architecture"](https://www.mindstudio.ai/blog/claude-code-source-leak-memory-architecture) | 上下文内记忆，MEMORY.md 指针索引，CLAUDE.md 静态配置。关于记忆的最佳单一资源。 |
| [WaveSpeed — "Claude Code Architecture: Leaked Source Deep Dive"](https://wavespeed.ai/blog/posts/claude-code-architecture-leaked-source-deep-dive/) | 512K 行 TS 源深度剖析；上下文压缩和反蒸馏。 |
| [Zain Hasan — "Inside Claude Code: An Architecture Deep Dive"](https://zainhas.github.io/blog/2026/inside-claude-code-architecture/) | 分层架构、5 种入口模式、多智能体演练。 |

### 相关学术论文

| 论文 | 会议 | 相关性 |
|:------|:------|:------|
| [Decoding the Configuration of AI Coding Agents](https://arxiv.org/abs/2511.09268) | arXiv | 328 个 Claude Code 配置文件的实证研究——软件工程关注点和共现模式。 |
| [On the Use of Agentic Coding Manifests](https://arxiv.org/abs/2509.14744) | arXiv | 分析了 242 个仓库中的 253 个 CLAUDE.md 文件——操作命令中的结构模式。 |
| [Context Engineering for Multi-Agent Code Assistants](https://arxiv.org/abs/2508.08322) | arXiv | 结合多个 LLM 进行代码生成的多智能体工作流。 |
| [OpenHands: An Open Platform for AI Software Developers](https://arxiv.org/abs/2407.16741) | ICLR 2025 | 开源 AI 编码智能体的主要学术参考。 |
| [SWE-Agent: Agent-Computer Interfaces](https://arxiv.org/abs/2405.15793) | NeurIPS 2024 | 具有自定义智能体-计算机接口的基于 Docker 的编码智能体。 |

### 本论文的不同之处

> 上述项目专注于**工程反向工程**或**实际重实现**，而本论文提供了**系统的价值观 → 原则 → 实现**分析框架——追溯五个人类价值观通过十三条设计原则到特定源代码级选择，并使用 OpenClaw 比较揭示跨切线集成机制，而不是模块化特性，才是工程复杂性的真正所在。

**查看带有更多资源的完整精选列表：[docs/related-resources.md](./docs/related-resources.md)**

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

---

## 其他值得关注的 AI 智能体项目

最近推出（2025-2026 年）的 Claude Code 生态系统外的开源 AI 智能体项目。

| 仓库 | 推出时间 | 重点 |
|:-----------|:-------|:------|
| [**openclaw/openclaw**](https://github.com/openclaw/openclaw) | 2026 年 1 月 | 跨消息平台的本地优先个人 AI 助手。 |
| [**sst/opencode**](https://github.com/sst/opencode) | 2025 年 6 月 | provider 无关的终端编码智能体。 |
| [**NousResearch/hermes-agent**](https://github.com/nousresearch/hermes-agent) | 2026 年 2 月 | 具有跨会话记忆的自我改进个人智能体。 |
| [**666ghj/MiroFish**](https://github.com/666ghj/MiroFish) | 2026 年 3 月 | 多智能体群体智能模拟引擎。 |
| [**MemPalace/mempalace**](https://github.com/MemPalace/mempalace) | 2026 年 | AI 智能体的本地优先记忆系统。 |
| [**multica-ai/multica**](https://github.com/multica-ai/multica) | 2026 年 | 用于任务分配和技能复合的托管智能体平台。 |
| [**badlogic/pi-mono**](https://github.com/badlogic/pi-mono) | 2025 年 8 月 | monorepo 编码智能体工具包——统一 LLM API（OpenAI/Anthropic/Google）、TUI + web UI、per-cwd 会话持久化。 |
| [**coleam00/Archon**](https://github.com/coleam00/Archon) | 2025 年 2 月 | 确定性 harness——YAML 定义的工作流与执行审计跟踪；与模型驱动的智能体循环对比。 |
| [**HKUDS/nanobot**](https://github.com/HKUDS/nanobot) | 2026 年 2 月 | 来自 HKU-DS 的超轻量级个人 AI 智能体；与重型 harness 的简约设计对比。 |
| [**HKUDS/OpenHarness**](https://github.com/HKUDS/OpenHarness) | 2026 年 4 月 | 带有内置个人智能体（Ohmo）的开放智能体 harness；harness 架构的学术参考点。 |
| [**openai/symphony**](https://github.com/openai/symphony) | 2026 年 2 月 | OpenAI 的隔离自主实现运行编排——并行会话设计轴。 |

<p align="right"><a href="#dive-into-claude-code-the-design-space-of-todays-ai-agent-system">↑ 返回顶部</a></p>

---

[![星标历史图表](https://api.star-history.com/svg?repos=VILA-Lab/Dive-into-Claude-Code&type=Date)](https://star-history.com/#VILA-Lab/Dive-into-Claude-Code&Date)

## 引用

<!-- <details>
<summary>BibTeX</summary> -->

```bibtex
@article{diveclaudecode2026,
  title={Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems},
  author={Jiacheng Liu, Xiaohan Zhao, Xinyi Shang, and Zhiqiang Shen},
  year={2026},
  eprint={2604.14228},
  archivePrefix={arXiv},
  primaryClass={cs.SE},
}
```

</details>


## 许可证

本作品采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可。