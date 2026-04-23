[返回主 README](../README.md)

# 相关资源：社区分析

> 围绕 Claude Code 架构的仓库、重新实现、博客文章和学术论文的精选索引。

---

## 架构分析

对 Claude Code 内部设计的深度剖析。

| 仓库 | 描述 |
|:-----------|:------------|
| [**ComeOnOliver/claude-code-analysis**](https://github.com/ComeOnOliver/claude-code-analysis) | 全面的逆向工程：源码树结构、模块边界、工具清单和架构模式。 |
| [**alejandrobalderas/claude-code-from-source**](https://github.com/alejandrobalderas/claude-code-from-source) | 18 章技术书（约 400 页）。全部原创伪代码，不含专有源码。 |
| [**liuup/claude-code-analysis**](https://github.com/liuup/claude-code-analysis) | 中文深度剖析——启动流程、查询主循环、MCP 集成、多智能体架构。 |
| [**sanbuphy/claude-code-source-code**](https://github.com/sanbuphy/claude-code-source-code) | 四语（EN/JA/KO/ZH）分析——10 个领域、75 份报告。涵盖遥测、代号、KAIROS、未发布的工具。 |
| [**cablate/claude-code-research**](https://github.com/cablate/claude-code-research) | 关于内部结构、Agent SDK 和相关工具的独立研究。 |
| [**Yuyz0112/claude-code-reverse**](https://github.com/Yuyz0112/claude-code-reverse) | 可视化 Claude Code 的 LLM 交互——日志解析器与可视化工具，追踪提示、工具调用和压缩。 |
| [**AgiFlow/claude-code-prompt-analysis**](https://github.com/AgiFlow/claude-code-prompt-analysis) | 跨 5 个对话会话的 API 请求/响应日志。独特的实证方法论，配以可复现数据。 |

---

## 开源重新实现

净室重写和可构建的研究分支。

| 仓库 | 描述 |
|:-----------|:------------|
| [**chauncygu/collection-claude-code-source-code**](https://github.com/chauncygu/collection-claude-code-source-code) | 聚合集——claw-code（Rust，30K+ 星）、nano-claude-code（Python 约 5K 行），以及原始源码存档。 |
| [**ultraworkers/claw-code**](https://github.com/ultraworkers/claw-code) | 净室 Rust 重新实现。9 天内拿到 17.9 万星——GitHub 历史上最快到达 10 万星的仓库。把 512K 行 TypeScript 压缩到约 20K 行。 |
| [**777genius/claude-code-working**](https://github.com/777genius/claude-code-working) | 可运行的逆向工程 CLI。使用 Bun 运行，450+ 个 chunk 文件，polyfill 了 30 个特性标志。 |
| [**T-Lab-CUHKSZ/claude-code**](https://github.com/T-Lab-CUHKSZ/claude-code) | 香港中文大学（深圳）的可构建研究分支——从原始 TypeScript 快照重建构建系统。 |
| [**ruvnet/open-claude-code**](https://github.com/ruvnet/open-claude-code) | 夜间自动反编译重建——903+ 测试、25 个工具、4 种 MCP 传输、6 个权限模式。 |
| [**Enderfga/openclaw-claude-code**](https://github.com/Enderfga/openclaw-claude-code) | OpenClaw 插件——为 Claude/Codex/Gemini/Cursor 提供统一的 ISession 接口。多智能体委员会。 |
| [**memaxo/claude_code_re**](https://github.com/memaxo/claude_code_re) | 从压缩后的包进行逆向工程——对公开分发的 cli.js 文件进行反混淆。 |
| [**agentforce314/clawcodex**](https://github.com/agentforce314/clawcodex) | Python 重新实现，支持多家 LLM 厂商。 |

---

## 指南与学习

教程和动手学习路径。

| 仓库 | 描述 |
|:-----------|:------------|
| [**shareAI-lab/learn-claude-code**](https://github.com/shareAI-lab/learn-claude-code) | "Bash 就是你需要的一切"——19 章 0 到 1 课程，包含可运行的 Python 智能体、Web 平台。ZH/EN/JA。 |
| [**FlorianBruniaux/claude-code-ultimate-guide**](https://github.com/FlorianBruniaux/claude-code-ultimate-guide) | 从初学者到高级用户的指南，包含生产级模板、智能体工作流指南和速查表。 |
| [**affaan-m/everything-claude-code**](https://github.com/affaan-m/everything-claude-code) | 智能体 harness 优化——skills、本能、记忆、安全和研究优先开发。50K+ 星标。 |
| [**nblintao/awesome-claude-code-postleak-insights**](https://github.com/nblintao/awesome-claude-code-postleak-insights) | 源码泄露后整理得最好的资源集合。BUDDY、KAIROS、ULTRAPLAN、Undercover Mode，以及 AutoDream（记忆巩固）。 |
| [**hesreallyhim/awesome-claude-code**](https://github.com/hesreallyhim/awesome-claude-code) | skills、hooks、斜杠命令、智能体编排器和插件的精选列表。 |
| [**rohitg00/awesome-claude-code-toolkit**](https://github.com/rohitg00/awesome-claude-code-toolkit) | 135 个智能体、35 个 skills、42 个命令、176+ 插件、20 个 hooks、15 条规则、7 个模板、14 个 MCP 配置。 |

---

## 博客文章与技术文章

### 源码泄露前的逆向工程（2025 年——2026 年初）

| 文章 | 价值所在 |
|:--------|:---------------------|
| [Marco Kotrotsos — "Claude Code Internals"（15 部分系列）](https://kotrotsos.medium.com/claude-code-internals-part-1-high-level-architecture-9881c68c799f) | 源码泄露前最系统的分析。架构、智能体循环、权限、子智能体、MCP、遥测。基于 v2.0.76。 |
| [George Sung — "Tracing Claude Code's LLM Traffic"](https://medium.com/@georgesung/tracing-claude-codes-llm-traffic-agentic-loop-sub-agents-tool-use-prompts-7796941806f5) | 完整系统提示和完整 API 日志。发现 Claude Code 同时调用两个模型（主循环走 Opus，元数据走 Haiku）。 |
| [Reid Barber — "Reverse Engineering Claude Code"](https://www.reidbarber.com/blog/reverse-engineering-claude-code) | 最早的分析之一（2025 年年中）。REPL 架构与工具套件。 |
| [Kir Shatrov — "Reverse Engineering Claude Code"](https://kirshatrov.com/posts/claude-code-internals) | 用 mitmproxy 拦截 API 调用。单次实验：LLM 耗时 40 秒，成本 0.11 美元。 |
| [Sabrina Ramonov — "Reverse-Engineering Using Sub Agents"](https://www.sabrina.dev/p/reverse-engineering-claude-code-using) | 创造性方法论：使用自定义子智能体（File Splitter、Structure Analyzer）对压缩后的 JS 进行逆向工程。 |

### 源码泄露后的分析（2026 年 3 月–4 月）

| 文章 | 价值所在 |
|:--------|:---------------------|
| [Alex Kim — "The Claude Code Source Leak"](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/) | 权威的首发分析。反蒸馏机制、挫折检测正则、Undercover Mode、每天约 25 万次浪费的 API 调用。在 Hacker News 上广泛流传。 |
| [Haseeb Qureshi — "Inside the Claude Code source"](https://gist.github.com/Haseeb-Qureshi/d0dc36844c19d26303ce09b42e7188c1) | 关键模块分析。React/Ink UI、四种压缩策略、动态提示边界系统。 |
| [Haseeb Qureshi — 跨智能体架构对比](https://gist.github.com/Haseeb-Qureshi/2213cc0487ea71d62572a645d7582518) | Claude Code vs Codex vs Cline vs OpenCode——架构级对比。 |
| [Engineer's Codex — "Diving into the Source Code Leak"](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code) | 模块化系统提示、约 40 个工具、46K 行查询引擎、反蒸馏机制。面向广泛读者，可读性强。 |
| [Han HELOIR YAN — "Nobody Analyzed Its Architecture"](https://medium.com/data-science-collective/everyone-analyzed-claude-codes-features-nobody-analyzed-its-architecture-1173470ab622) | "护城河是 harness，而不是模型。"与我们的报告论点一致。 |
| [Agiflow — "Reverse Engineering Prompt Augmentation"](https://agiflow.io/blog/claude-code-internals-reverse-engineering-prompt-augmentation/) | 5 种提示增强机制，全部有实际网络抓包为证。文章还展示了 Skills 的两步语义匹配流程。 |

### 专业深度剖析

| 主题 | 资源 |
|:------|:---------|
| **记忆架构** | [MindStudio — "Three-Layer Memory Architecture"](https://www.mindstudio.ai/blog/claude-code-source-leak-memory-architecture) — 上下文内记忆、MEMORY.md 指针索引、CLAUDE.md 静态配置。目前关于记忆最好的一份资源。 |
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
| [Decoding the Configuration of AI Coding Agents](https://arxiv.org/abs/2511.09268) | arXiv | 对 328 个 Claude Code 配置文件的实证研究——软件工程关注点与共现模式。 |
| [On the Use of Agentic Coding Manifests](https://arxiv.org/abs/2509.14744) | arXiv | 分析了 242 个仓库中的 253 个 CLAUDE.md 文件——操作性命令中的结构模式。 |
| [Context Engineering for Multi-Agent Code Assistants](https://arxiv.org/abs/2508.08322) | arXiv | 结合多个 LLM 进行代码生成的多智能体工作流。 |
| [OpenHands: An Open Platform for AI Software Developers](https://arxiv.org/abs/2407.16741) | ICLR 2025 | 开源 AI 编码智能体的主要学术参考。 |
| [SWE-Agent: Agent-Computer Interfaces](https://arxiv.org/abs/2405.15793) | NeurIPS 2024 | 基于 Docker 的编码智能体，带有自定义的智能体-计算机接口。 |
| [The OpenHands Software Agent SDK](https://arxiv.org/abs/2511.03690) | arXiv | 生产级智能体的可组合 SDK 基础。 |
| [A Survey on Code Generation with LLM-based Agents](https://arxiv.org/abs/2508.00083) | arXiv | AI 编码智能体领域最佳综述。 |
| [AI Agent Systems: Architectures, Applications, and Evaluation](https://arxiv.org/html/2601.01743v1) | arXiv 2026 | 智能体系统的宏观分类。 |

---

## 本论文的不同之处

上述项目多聚焦于**工程层面的逆向工程**或**实际动手重新实现**，而本论文提供的是**一套系统的"价值观 → 原则 → 实现"分析框架**——从五个人类价值观出发，经由十三条设计原则，一路追到具体的源码级选择；并借助与 OpenClaw 的对比揭示一个事实：真正构成工程复杂性的，是那些横跨各子系统的整合机制，而不是模块化的单点特性。

---

*知道有应该列在这里的资源吗？打开 issue 或 PR。*