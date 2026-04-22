[返回主 README](../README.md)

# 相关资源：社区分析

> Claude Code 架构周围的仓库、重实现、博客文章和学术论文的精选图。

---

## 架构分析

对 Claude Code 内部设计的深度剖析。

| 仓库 | 描述 |
|:-----------|:------------|
| [**ComeOnOliver/claude-code-analysis**](https://github.com/ComeOnOliver/claude-code-analysis) | 综合反向工程：源树结构、模块边界、工具清单和架构模式。 |
| [**alejandrobalderas/claude-code-from-source**](https://github.com/alejandrobalderas/claude-code-from-source) | 18 章技术书（约 400 页）。全部原创伪代码，无专有源代码。 |
| [**liuup/claude-code-analysis**](https://github.com/liuup/claude-code-analysis) | 中文深度剖析——启动流程、查询主循环、MCP 集成、多智能体架构。 |
| [**sanbuphy/claude-code-source-code**](https://github.com/sanbuphy/claude-code-source-code) | 四语种分析（EN/JA/KO/ZH）——10 个领域、75 份报告。涵盖遥测、代号、KAIROS、未发布的工具。 |
| [**cablate/claude-code-research**](https://github.com/cablate/claude-code-research) | 关于内部结构、Agent SDK 和相关工具的独立研究。 |
| [**Yuyz0112/claude-code-reverse**](https://github.com/Yuyz0112/claude-code-reverse) | 可视化 Claude Code 的 LLM 交互——日志解析器和可视化工具，追踪提示、工具调用和压缩。 |
| [**AgiFlow/claude-code-prompt-analysis**](https://github.com/AgiFlow/claude-code-prompt-analysis) | 跨 5 个对话会话的 API 请求/响应日志。独特的实证方法论与可复现数据。 |

---

## 开源重实现

干净室重写和可构建的研究分支。

| 仓库 | 描述 |
|:-----------|:------------|
| [**chauncygu/collection-claude-code-source-code**](https://github.com/chauncygu/collection-claude-code-source-code) | 元收集——claw-code（Rust，30K+ 星）、nano-claude-code（Python 约 5K 行）和原始源存档。 |
| [**ultraworkers/claw-code**](https://github.com/ultraworkers/claw-code) | 干净室 Rust 重实现。9 天内 179K 星——GitHub 历史上最快达到 100K 的仓库。将 512K 行 TypeScript 减少到约 20K 行。 |
| [**777genius/claude-code-working**](https://github.com/777genius/claude-code-working) | 可运行的重构 CLI。使用 Bun，450+ chunk 文件，30 个特性标志 polyfill。 |
| [**T-Lab-CUHKSZ/claude-code**](https://github.com/T-Lab-CUHKSZ/claude-code) | CUHK-深圳可构建研究分支——从原始 TypeScript 快照重建构建系统。 |
| [**ruvnet/open-claude-code**](https://github.com/ruvnet/open-claude-code) | 夜间自动反编译重建——903+ 测试、25 个工具、4 个 MCP 传输、6 个权限模式。 |
| [**Enderfga/openclaw-claude-code**](https://github.com/Enderfga/openclaw-claude-code) | OpenClaw 插件——Claude/Codex/Gemini/Cursor 的统一 ISession 接口。多智能体委员会。 |
| [**memaxo/claude_code_re**](https://github.com/memaxo/claude_code_re) | 从精简包反向工程——对公开分发的 cli.js 文件的反混淆。 |

---

## 指南与学习

教程和动手学习路径。

| 仓库 | 描述 |
|:-----------|:------------|
| [**shareAI-lab/learn-claude-code**](https://github.com/shareAI-lab/learn-claude-code) | "Bash 就是你需要的一切"——19 章 0 到 1 课程，包含可运行的 Python 智能体、网络平台。ZH/EN/JA。 |
| [**FlorianBruniaux/claude-code-ultimate-guide**](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) | 从初学者到高级用户的指南，包含生产就绪模板、智能体工作流指南和速查表。 |
| [**affaan-m/everything-claude-code**](https://github.com/affaan-m/everything-claude-code) | 智能体 harness 优化——skills、本能、记忆、安全和研究优先开发。50K+ 星标。 |
| [**nblintao/awesome-claude-code-postleak-insights**](https://github.com/nblintao/awesome-claude-code-postleak-insights) | 最佳策展的泄漏后资源。BUDDY、KAIROS、ULTRAPLAN、Undercover Mode、AutoDream 内存整合。 |
| [**hesreallyhim/awesome-claude-code**](https://github.com/hesreallyhim/awesome-claude-code) | skills、hooks、斜杠命令、智能体编排器和插件的精选列表。 |
| [**rohitg00/awesome-claude-code-toolkit**](https://github.com/rohitg00/awesome-claude-code-toolkit) | 135 个智能体、35 个 skills、42 个命令、176+ 插件、20 个 hooks、15 个规则、7 个模板、14 个 MCP 配置。 |

---

## 博客文章与技术文章

### 泄漏前的反向工程（2025 年——2026 年初）

| 文章 | 它的价值所在 |
|:--------|:---------------------|
| [Marco Kotrotsos — "Claude Code Internals"（15 部分系列）](https://kotrotsos.medium.com/claude-code-internals-part-1-high-level-architecture-9881c68c799f) | 泄漏前最系统的分析。架构、智能体循环、权限、子智能体、MCP、遥测。基于 v2.0.76。 |
| [George Sung — "Tracing Claude Code's LLM Traffic"](https://medium.com/@georgesung/tracing-claude-codes-llm-traffic-agentic-loop-sub-agents-tool-use-prompts-7796941806f5) | 完整系统提示和完整 API 日志。发现双模型使用（主循环用 Opus，元数据用 Haiku）。 |
| [Reid Barber — "Reverse Engineering Claude Code"](https://www.reidbarber.com/blog/reverse-engineering-claude-code) | 最早的分析之一（2025 年中）。REPL 架构和工具套件。 |
| [Kir Shatrov — "Reverse Engineering Claude Code"](https://kirshatrov.com/posts/claude-code-internals) | 使用 mitmproxy 拦截 API 调用。单一实验：40 秒 LLM 时间，0.11 美元成本。 |
| [Sabrina Ramonov — "Reverse-Engineering Using Sub Agents"](https://www.sabrina.dev/p/reverse-engineering-claude-code-using) | 创意方法论：自定义子智能体（文件分割器、结构分析器）来反向工程精简的 JS。 |

### 泄漏后分析（2026 年 3 月-4 月）

| 文章 | 它的价值所在 |
|:--------|:---------------------|
| [Alex Kim — "The Claude Code Source Leak"](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/) | definitive 首发分析。反蒸馏、挫折 regexes、Undercover Mode、每天约 25 万次浪费的 API 调用。在 HN 上疯传。 |
| [Haseeb Qureshi — "Inside the Claude Code source"](https://gist.github.com/Haseeb-Qureshi/d0dc36844c19d26303ce09b42e7188c1) | 关键模块分析。React/Ink UI、四种压缩策略、动态提示边界系统。 |
| [Haseeb Qureshi — 跨智能体架构比较](https://gist.github.com/Haseeb-Qureshi/2213cc0487ea71d62572a645d7582518) | Claude Code vs Codex vs Cline vs OpenCode——架构级比较。 |
| [Engineer's Codex — "Diving into the Source Code Leak"](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code) | 模块化系统提示、约 40 个工具、46K 行查询引擎、反蒸馏。对广大受众友好。 |
| [Han HELOIR YAN — "Nobody Analyzed Its Architecture"](https://medium.com/data-science-collective/everyone-analyzed-claude-codes-features-nobody-analyzed-its-architecture-1173470ab622) | "护城河是 harness，而不是模型。"与我们的报告论点一致。 |
| [Agiflow — "Reverse Engineering Prompt Augmentation"](https://agiflow.io/blog/claude-code-internals-reverse-engineering-prompt-augmentation/) | 5 种提示增强机制，由实际网络追踪支持。展示 Skills 两步语义匹配。 |

### 专业深度剖析

| 主题 | 资源 |
|:------|:---------|
| **记忆架构** | [MindStudio — "Three-Layer Memory Architecture"](https://www.mindstudio.ai/blog/claude-code-source-leak-memory-architecture) — 上下文内、内存.md 指针索引、CLAUDE.md 静态配置。关于记忆的最佳单一资源。 |
| **权限系统** | [Marco Kotrotsos — "Part 8: The Permission System"](https://kotrotsos.medium.com/claude-code-internals-part-8-the-permission-system-624bd7bb66b7) |
| **提示缓存** | [ClaudeCodeCamp — "How Prompt Caching Actually Works"](https://www.claudecodecamp.com/p/how-prompt-caching-actually-works-in-claude-code) |
| **MCP 集成** | [Gigi Sayfan — "MCP Unleashed"](https://medium.com/@the.gigi/claude-code-deep-dive-mcp-unleashed-0c7692f9c2c2) |
| **压缩与遥测** | [WaveSpeedAI — "Architecture Deep Dive"](https://wavespeed.ai/blog/posts/claude-code-architecture-leaked-source-deep-dive/) — 三层压缩、挫折指标。 |
| **Rust 重写分析** | [DEV Community — "Architecture via Rust Rewrite"](https://dev.to/brooks_wilson_36fbefbbae4/claude-code-architecture-explained-agent-loop-tool-system-and-permission-model-rust-rewrite-41b2) — 三层结构中的 18 个工具。 |
| **权限配置** | [Vincent Qiao — "Permissions System Deep Dive"](https://blog.vincentqiao.com/en/posts/claude-code-settings-permissions/) |

---

## 相关学术论文

| 论文 | 会议 | 相关性 |
|:------|:------|:------|
| [Decoding the Configuration of AI Coding Agents](https://arxiv.org/abs/2511.09268) | arXiv | 328 个 Claude Code 配置文件的实证研究——软件工程关注点和共现模式。 |
| [On the Use of Agentic Coding Manifests](https://arxiv.org/abs/2509.14744) | arXiv | 分析了 242 个仓库中的 253 个 CLAUDE.md 文件——操作命令中的结构模式。 |
| [Context Engineering for Multi-Agent Code Assistants](https://arxiv.org/abs/2508.08322) | arXiv | 结合多个 LLM 进行代码生成的多智能体工作流。 |
| [OpenHands: An Open Platform for AI Software Developers](https://arxiv.org/abs/2407.16741) | ICLR 2025 | 开源 AI 编码智能体的主要学术参考。 |
| [SWE-Agent: Agent-Computer Interfaces](https://arxiv.org/abs/2405.15793) | NeurIPS 2024 | 具有自定义智能体-计算机接口的基于 Docker 的编码智能体。 |
| [The OpenHands Software Agent SDK](https://arxiv.org/abs/2511.03690) | arXiv | 生产智能体的可组合 SDK 基础。 |
| [A Survey on Code Generation with LLM-based Agents](https://arxiv.org/abs/2508.00083) | arXiv | AI 编码智能体领域最佳综述。 |
| [AI Agent Systems: Architectures, Applications, and Evaluation](https://arxiv.org/html/2601.01743v1) | arXiv 2026 | 广泛的智能体系统分类学。 |

---

## 本论文的不同之处

上述项目专注于**工程反向工程**或**实际重实现**，而本论文提供了**系统的价值观 → 原则 → 实现**分析框架——追溯五个人类价值观通过十三条设计原则到特定源代码级选择，并使用 OpenClaw 比较揭示跨切线集成机制，而不是模块化特性，才是工程复杂性的真正所在。

---

*知道有应该列在这里的资源吗？打开 issue 或 PR。*