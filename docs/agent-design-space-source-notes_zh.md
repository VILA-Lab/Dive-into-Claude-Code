# Agent Systems Design Space 新进展资料记录

更新月份：2026-07

本页记录与 agent system design space 高度相关、且来源质量足够高的新进展。每条资料保留发布年月，方便后续持续追加和比较。

本页汇总关于 agent system 设计空间的高信号资料，重点关注高层原则、运行时机制、权限与治理、上下文/记忆、工具连接、长程执行、多 agent 编排和评测安全。资料来自并行子代理检索后的人工合并与去重。

## 快速结论

这些资料体现的核心趋势不是“更会聊天的 agent”，而是 agent operating layer 正在成形：可恢复的执行环境、显式权限边界、可审计遥测、可版本化 context/skills、可插拔工具连接、长任务状态机、人类中途接管，以及从 traces 反推 eval 和改进循环。

对本仓库的 Design Space 叙事，最值得强化的设计启示是：

1. **Runtime and control plane are first-class design concerns**：持久执行、检查点、沙箱、agent inventory、策略面和可观测性，应该作为一等设计关注点，而不是部署细节。
2. **Context is managed infrastructure**：context 不只是 prompt，而是文件、skills、memory、interpreter state、workspace state、IDE indexes 和可版本化策略。
3. **Execution boundary is the safety boundary**：sandbox、network policy、credential custody、OS-level isolation、tenant boundary 和 approval policy 是核心架构对象。
4. **Tools and skills are a supply chain**：MCP、SDK、CLI、plugins、skills 和 agent-to-agent protocols 放大能力，也引入 registry、allowlist、identity、versioning 和 revocation 问题。
5. **Humans become managers and verifiers**：长任务和异步代理要求人类能在过程中审查、改方向、批准、回滚，而不是只看最终 diff。
6. **Observability must close the improvement loop**：生产 agent 的失败模式需要通过 trace/eval/issue/dataset 回路进入下一轮系统改进。

## P0: 最值得纳入综述的资料

| 年月 | 资料 | 核心内容 | Design Space 价值 |
|:---:|:---|:---|:---|
| 2026-07 | [Claude Code CHANGELOG（v2.1.178 至 v2.1.207）](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md) | 窗口内 30 个版本。子代理默认后台运行并新增 `agent_needs_input`/`agent_completed` 钩子（v2.1.198），且明确「agent 的消息永远不算用户批准」；默认权限模式改为 Manual（v2.1.200）；新增 `Tool(param:value)` 权限规则语法，如 `Agent(model:opus)`（v2.1.178）；auto mode 不再读取仓库内的 `.claude/settings.local.json`（v2.1.207）；新增 `sandbox.credentials` 阻断沙箱读取凭据与密钥环境变量（v2.1.187）。 | harness 设计的第一手权威变更记录，覆盖运行时、权限、上下文、工具、子代理、长程执行六条主线，多条改动直接印证信任边界论点。 |
| 2026-07 | [Configure auto mode](https://code.claude.com/docs/en/auto-mode-config) | 首次完整披露 auto mode 分类器的四级策略优先级：`hard_deny`（无条件）优先于 `soft_deny`（可被覆盖）优先于 `allow`（作为 soft_deny 的例外）优先于用户显式意图。规则是自然语言散文而非正则。分类器读 CLAUDE.md，但明确不读共享的 `.claude/settings.json`，因此签入仓库的配置无法注入自己的放行规则。 | 目前唯一一份把「LLM 作为权限判定器」的完整策略层次与逃逸路径讲清楚的官方文档，支撑工具授权从提示词围栏走向分层策略引擎的论证。 |
| 2026-06 | [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams) | 明确 agent teams 与 subagents 是两种不同原语：teammates 各自独立上下文、通过 mailbox 互发消息、共享一个带文件锁的任务列表（支持依赖与自主认领），状态落盘于 `~/.claude/teams/` 与 `~/.claude/tasks/`。信任边界：teammate 无法代替用户批准，被拒动作也不能转交其他 teammate 绕过。 | 把「子代理」与「对等代理团队」区分为两种编排原语并给出可验证的信任边界规则。 |
| 2026-06 | [What's new in Claude Sonnet 5](https://platform.claude.com/docs/en/about-claude/models/whats-new-sonnet-5) | 手动 extended thinking（`thinking: {budget_tokens: N}`）被移除并返回 400；`temperature`/`top_p`/`top_k` 设为非默认值返回 400；新分词器使同样文本产生约多 30% token。 | 模型侧收回了 harness 原本持有的两个旋钮（思考预算、采样参数），推理预算控制权从 harness 上移到模型自身，是 harness 与模型边界在移动的直接证据。 |
| 2026-06 | [Agentic coding and persistent returns to expertise](https://www.anthropic.com/research/claude-code-expertise) | 基于真实 Claude Code 使用数据：人类做约 70% 的规划决策但只做 20% 的执行决策；用户每发一条 prompt 平均触发约 10 个 Claude 动作，部分场景两次人工介入之间超过 100 个动作。 | 首份来自 Anthropic 的定量证据，说明「人类保留规划权、让渡执行权」是实际的分工形态，为人类控制面与监督成本提供实测锚点。 |
| 2026-06 | [MCP 2026-07-28 规范转为无状态协议](https://blog.modelcontextprotocol.io/posts/sdk-betas-2026-07-28/) | 取消 `initialize` 握手与协议级 session，能力改由 `server/discover` 获取；新增 Multi Round Trip Requests（工具可返回 `InputRequiredResult` 在调用中途向用户追问）；新增用于网关路由的 `Mcp-Method`/`Mcp-Name` 传输头；roots、sampling、logging 标记为 deprecated。 | 工具生态层最大的一次结构性变更：MCP 从有状态会话协议转向可负载均衡的无状态 HTTP，直接影响工具接口与网关设计。 |
| 2026-06 | [Enterprise-Managed Authorization: Zero-touch OAuth for MCP](https://blog.modelcontextprotocol.io/posts/enterprise-managed-auth/) | 客户端在 SSO 时从 IdP 拿到 Identity Assertion JWT Authorization Grant，再换取 MCP server 的 access token，按用户已有的组和角色授权，取消逐服务器的同意屏。 | MCP 授权决策权从「每个用户逐个点同意」上移到组织 IdP，是权限维度从个人授权走向组织策略的关键一步。 |
| 2026-06 | [Amazon Bedrock AgentCore Harness GA](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-harness-is-now-generally-available-go-from-idea-to-production-grade-agent-in-minutes/) | `CreateHarness` 与 `InvokeHarness` 两个 API 调用即可声明式定义 agent（模型、工具、skills、记忆策略、容器环境），底层封装 microVM 隔离 Runtime、托管 Memory、Gateway、沙箱 Browser、Code Interpreter、Identity token vault、Observability 七个原语。 | harness 从「开发者自己写的循环」变成「云厂商托管的配置对象」，是设计空间里一个全新的坐标轴：谁拥有 loop、环境与工具边界。 |
| 2026-06 | [AgentCore policy 与 Guardrails](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-bedrock-agentcore-policy-guardrails-generally-available/) | policy 控制 agent 被授权执行哪些动作，Guardrails 实时检查每个被授权动作的输出与每次 gateway 调用的输入。关键点：评估发生在 gateway 边界、在 agent 代码之外，因此无论 agent 自主程度多高都能一致执行。 | 「在 agent 代码之外的边界上强制执行」是与 in-loop 权限检查截然不同的架构选择。 |
| 2026-06 | [Reduced cost, better isolation, more resilience: Strands Agents evolves](https://strandsagents.com/blog/reduced-cost-better-isolation-more-resilience/) | 大 tool result 卸载到外部存储只留截断预览，旧消息压缩成结构化摘要，85% 上下文占用时主动触发压缩；Shell 沙箱只暴露 bound paths，网络默认阻断内网，密钥按 URL 在请求时注入、agent 从不持有凭据；Evals 加入 chaos testing 与沙箱逃逸红队。 | 同时覆盖 context/memory、sandboxing、eval 三条主线，且每条都给出可复现的阈值与边界定义。 |
| 2026-07 | [Agent Harness: Scaling the claw or harness capabilities](https://devblogs.microsoft.com/agent-framework/agent-harness-scaling-the-claw-or-harness-capabilities/) | 微软给出扩展 harness 能力的四条正交路径：Skills（按请求匹配才渐进加载全文）、受限 Shell（命令 re-anchor 到 vault 无法逃逸）、CodeAct（沙箱内执行代码）、Background Agents（fan-out 给并发子 agent 再聚合）。 | 把「能力扩展」拆成四个正交机制，正好对应 tools、sandboxing、subagent 三条主线，可直接用作跨系统对照。 |
| 2026-06 | [Agent Harness: Working with your data, safely](https://devblogs.microsoft.com/agent-framework/agent-harness-working-with-your-data-safely/) | 文件操作限制在可配置 root folder 内；工具可标记为 approval-required；用户可单次批准，也可建立 standing rule（「总是批准此工具」或「总是批准这组参数」），且 standing rule 只在 session 内有效、不会固化进 agent。 | session 级而非永久的批准规则，是与 Claude Code 权限模型的直接可比点。 |
| 2026-07 | [What's New in Microsoft Foundry, June 2026](https://devblogs.microsoft.com/foundry/whats-new-in-microsoft-foundry-june-2026/) | Tool Search 在运行时按需检索相关工具，而不是把全部 tool schema 塞进上下文；Memory 新增 procedural memory 与 TTL 自动淘汰；Autopilot Agents 拥有完整 Entra Agent ID 账号。 | Tool Search 直接回应「工具数量膨胀撑爆上下文」这一设计张力；TTL 记忆与 agent 身份是新的设计维度。 |
| 2026-06 | [Antigravity CLI](https://github.com/google-antigravity/antigravity-cli)（背景：[Transitioning Gemini CLI to Antigravity CLI](https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/)） | Google 用 Go 重写的官方 CLI 取代 Gemini CLI（2026-06-18 生效），与 Antigravity 2.0 桌面端共用同一个 agent harness。据 CHANGELOG（仓库只放发布产物与示例，无源码，故不可按 code-verified 定级）：嵌套 subagent 可到孙级及更深，子轨迹更新递归回传到根对话；`.agents/hooks.json` 工作区级 hooks，pre-tool hook 决定某次**工具调用**是否放行；按项目的权限配置存于 `~/.gemini/config/projects/`（**不在仓库内**），优先级高于全局设置。 | 一个厂商把 harness 从 IDE 与 CLI 中抽出来作为共享层，且 hooks、subagent、权限优先级的设计与 Claude Code 高度同构，是跨系统对照的最佳新样本。注意其项目级配置在用户主目录而非仓库内，与 Claude Code 的 in-repo 配置模型正好相反，这一差异本身值得写。 |
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
| 2026-06 | [Devin Fusion](https://cognition.com/blog/devin-fusion) | 前沿模型做主 agent 只负责计划、歧义判断与终审，把常规操作委派给拥有独立工具集与独立缓存上下文的 sidekick 模型；轻量分类器在会话中途评估任务难度，并把模型切换放在 context compaction 时刻执行，因为那里本来就会 cache miss，所以切换「免费」。 | 把「模型路由」与「上下文压缩」这两个通常独立的机制耦合起来，是长时程执行中非常具体的成本与能力调度设计。 |
| 2026-07 | [Agentic MapReduce](https://devin.ai/blog/agentic-map-reduce/) | 四阶段：Plan（agent 生成 selector 模式）、Shard（确定性地按 selector 切分成有界批次）、Map（并行子会话各自只看自己那一片）、Reduce（归并去重并发现跨片的链式关系）。原则是「只在需要推理的地方放 agent，其余全部确定性」。 | 针对「单 agent 装不下整个代码库」的上下文瓶颈，给出 agent 与确定性计算分工的清晰边界。 |
| 2026-06 | [AI SDK 7](https://vercel.com/blog/ai-sdk-7) | HarnessAgent 把 Claude Code、Codex、Pi 等成品 harness 统一成一个 API（会话可 park 与 resume）；WorkflowAgent 把每次工具调用做成可持久化可重试的步骤，进程重启后从最后完成步继续；工具审批支持 HMAC 签名以防伪造批准。 | 同时命中 harness 可替换抽象、长时程持久化执行，以及「审批本身需要防篡改」这一少被讨论的控制面安全问题。 |
| 2026-07 | [Droid Shield 2.0](https://factory.ai/news/droid-shield-2-0) | 在自主 commit 与 push 前拦截密钥泄露的三段流水线：中间是确定性正则扫描器，两侧各挂一个微调模型。Risk model 在扫描器未触发时以召回优先重新判断上下文；Downgrade model 在扫描器触发时先遮蔽候选密钥，仅凭上下文判断是否为误报。 | 把「确定性规则」与「学习型模型」组合成双向纠错闸门，是自主执行下 guardrail 工程的高质量样本。 |
| 2026-07 | [Harness Engineering for Self-Improvement](https://lilianweng.github.io/posts/2026-07-04-harness/) | 定义 harness 为「围绕基础模型、编排执行的系统，决定模型如何思考规划、调用工具、感知与管理上下文、存储产物、评估结果」。核心论点：近期递归自我改进的路径不太可能始于模型直接改写权重，而是通过 coding agent 演化 harness 组件本身。 | 把 harness 立为自我改进的载体，直接呼应 Meta-Harness 一类工作，是「harness 是独立设计对象」最有分量的独立背书。 |
| 2026-07 | [Better Models: Worse Tools](https://lucumr.pocoo.org/2026/7/4/better-models-worse-tools/) | 观察到 Opus 4.8 与 Sonnet 5 在嵌套结构中会捏造 `requireUnique`、`oldText2` 之类不存在的字段，而 payload 本身字节正确。作者推断这是因为新模型在 Claude Code 宽松的 harness 中做 RL，该 harness 会静默修复畸形调用（过滤未知 key、接受参数别名），于是「轻微畸形也能拿到奖励」。 | harness 反向塑造模型的直接证据：工具 schema 不是中立抽象，harness 的容错设计会泄漏进模型权重。对「harness 是独立设计对象」既是支撑也是复杂化。 |
| 2026-07 | [Agentic Autonomy Levels](https://addyosmani.com/blog/agentic-autonomy-levels/) | 双轴自治模型，把长期混为一谈的两个维度拆开：agency 轴（单个 agent 独立到什么程度）与 orchestration 轴（多 agent 如何协调），并给出统一分级与「执行前契约」（目标、范围、非目标、工具与权限、停止条件、证据要求、升级路径、预算）。 | 「执行前契约」是对 Claude Code 即时权限提示这一在线审批模式的结构化替代。 |
| 2026-03 | [Meta-Harness: End-to-End Optimization of Model Harnesses](https://arxiv.org/abs/2603.28052) | Yoonho Lee 等（Stanford）。让 coding agent 充当 proposer，在模型固定的前提下自动搜索并优化 harness 本身（memory、retrieval、context 构造、prompt、工具使用逻辑），维护候选种群与 Pareto 前沿。结果：比当前最好的上下文管理系统高 7.7 分且少用 4 倍上下文 token；IMO 级题目上平均高 4.7 分；搜出的 harness 在 TerminalBench-2 上超过最好的手工基线。**注意：该文 Introduction 开篇那句「只改 harness 可造成 6 倍性能差距」是在引用他人工作（Tian et al., SWE-bench Mobile），不是 Meta-Harness 自己的结论，引用时不要张冠李戴。** | 把 harness 变成可自动优化的对象，是本仓库核心论点的直接方法论延伸。虽早于本轮窗口，但此前遗漏，补录。 |
| 2026-06 | [From Question Answering to Task Completion: A Survey on Agent System and Harness Design](https://arxiv.org/abs/2606.20683) | 以 model-harness lens 综述 agent 系统，把执行 harness 拆为六项耦合的运行时职责：observation、context、control、action、state、verification。 | 与本仓库的 design space 框架高度同构且同期出现，是必须正面对话的工作。 |
| 2026-06 | [ActPlane: Programmable OS-Level Policy Enforcement for Agent Harnesses](https://arxiv.org/abs/2606.25189) | 用 eBPF 在 OS 内核层拦截 agent 的全部执行路径（包括绕过 tool-call 层的间接路径），配合信息流控制 DSL 表达跨事件策略，并把违规原因作为语义反馈回传给 agent；开销 1.9% 到 8.4%。 | 直接指出 Claude Code 的权限检查发生在 tool-call 层而该层可被绕过，是把权限模型下沉到内核的第一个完整方案。 |
| 2026-06 | [Lingering Authority: Revocable Resource-and-Effect Capabilities for Coding Agents](https://arxiv.org/abs/2606.22504) | 提出 PORTICO 引用监视器，把任务规格编译为初始能力、授予规则、可信闭包谓词与全局拒绝规则；资源被物化为 epoch-bound 的不透明句柄，闭包条件满足后即失效。 | 精确命名了「授权残留」：会话内一次批准后，权限不会随任务阶段收回。 |
| 2026-06 | [TokenPilot: Cache-Efficient Context Management for LLM Agents](https://arxiv.org/abs/2606.17016) | 双粒度上下文管理：全局的 Ingestion-Aware Compaction 在环境输出入口处过滤噪声并保持前缀稳定，局部的 Lifecycle-Aware Eviction 在上下文段效用到期后才保守驱逐，从而避免 KV cache 失效。 | 直击一个常被回避的问题：任意改写历史会摧毁 prefix cache，可用于解释追加式而非重写式上下文策略的合理性。 |
| 2026-07 | [The Balkanization of Execution-Security Research for AI Coding Agents](https://arxiv.org/abs/2607.05743) | SoK，把 2023 至 2026 年 39 篇执行层安全研究归入 17 类（沙箱隔离、能力与访问控制、策略执行、TOCTOU、MCP 威胁、身份委派、执行溯源、出网控制等），并指出策略执行对真实 denylist 的失败率高达 69% 到 98%。 | 为 permissions 与 sandboxing 维度提供唯一一份系统化的对手地图。 |
| 2026-06 | [One Fake Bug Report Hijacked a $250 Billion Company's AI Agent, Then 100+ More](https://tenetsecurity.ai/blog/agentjacking-coding-agents-with-fake-sentry-errors/)（Tenet Security，2026-06-17） | Sentry 的公开 DSN 接受任意错误负载，攻击者 POST 一个含 markdown 指令的事件，渲染成伪造的「Resolution」小节；开发者让 agent 排查该 Sentry issue 时，agent 经 MCP 取回被污染事件并当作可信修复指引执行，运行攻击者控制的 npm 包并带走 AWS key、GitHub token 与 SSH 凭据。确认影响 Claude Code、Cursor、OpenAI Codex，含沙箱变体与 CI/CD 流水线。 | 迄今最有力的「工具返回值即不可信输入」实证。任何假设 MCP 返回内容可信的授权模型，此案例即是反例。 |
| 2026-06 | [CVE-2026-12957：AWS Language Servers 工作区配置自动执行](https://nvd.nist.gov/vuln/detail/CVE-2026-12957) | Language Servers for AWS（Amazon Q Developer 底层）1.65.0 之前版本存在信任边界执行不当：用户打开恶意构造的工作区、并在提示时选择信任它之后，项目配置文件中的任意命令会被自动执行。评分：CNA（Amazon）同时给出 CVSS v4.0 8.5（HIGH）与 CVSS v3.1 7.8（HIGH）；NVD 尚未给出独立评分（不要误写成「CNA 与 NVD 的双轨评分」）。 | 「打开并信任工作区即执行」是编码 agent 特有的攻击面，直接对应项目级配置文件（CLAUDE.md、.mcp.json）的信任模型问题。 |
| 2026-05 | [OpenAI named a Leader in enterprise coding agents by Gartner](https://openai.com/index/gartner-2026-agentic-coding-leader/) | Codex 被定位为企业级 coding agent；OpenAI 强调 large codebase、工具使用、测试、approval gates、RBAC、sandboxing、auditable workspace governance。 | 说明 coding agent 已从 autocomplete 进入 delegated work / operating layer；治理和审计是企业 agent 的一等需求。 |
| 2026-05 | [Cursor: What we've learned building cloud agents](https://cursor.com/blog/cloud-agent-lessons) | 云端 agent 需要完整开发环境、durable execution、VM checkpoint/fork、secret redaction、network policies、credential management。 | 可直接支撑“agent environment as product”和“cloud agent runtime”章节。 |
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
| 2026-05 | [Claude Code Dynamic Workflows](https://code.claude.com/docs/en/workflows) + [Opus 4.8](https://www.anthropic.com/news/claude-opus-4-8) | Claude 自己写 JavaScript 编排脚本，后台 runtime 扇出到上千个 subagent；中间状态存在脚本变量（对话之外），只有最终答案进入 context；16 并发 / 1000 总量上限，同一 session 内可 resume。随 Opus 4.8（2026-05-28）一同发布，v2.1.154 引入，research preview。 | 直接延伸本文 §8 subagent 编排与 Future「超越 session / sub-agent / memory 的新协调原语」：编排逻辑从对话搬进代码，是 context-as-bottleneck 原则的下一步。 |

## P1: 强相关资料

| 年月 | 资料 | 核心内容 | Design Space 价值 |
|:---:|:---|:---|:---|
| 2026-06 | [Zed: Software Is Made Between Commits（DeltaDB）](https://zed.dev/blog/introducing-deltadb) | 为 agent 协作设计的版本控制：一条消息与它产生的编辑并排记录，二者不会漂移；每个引用锚定到 delta 而非行号，因此代码变动后引用仍然存活，可从过去对话的任一行跳到该代码的当前状态或当时状态；内嵌 CRDT worktree 支持多人多 agent 跨机器同时编辑同一批文件。 | 指出 git 围绕离散 commit 组织，从未被设计来承载「生成代码的那段对话」；把对话与代码变更绑定为同一制品，是会话持久化与可审计性的一个根本性重构。 |
| 2026-05 | [Warp: A single pane of glass for managing all of your cloud agents](https://www.warp.dev/blog/multi-harness-cloud-agent-orchestration) | Oz 作为 multi-harness 控制面：在一个面板里启动、追踪、治理与引导 Claude Code、Codex 与 Warp Agent，比较它们的效果并为不同任务选用不同 harness，同时保持一致的治理、访问控制与审计日志；跨 harness 的 Agent Memory 让经验在会话、仓库与项目之间迁移。 | 把 harness 本身变成可比较、可替换、可统一治理的对象，是「harness 所有权边界正在移动」的直接产品化证据。 |
| 2026-05 | [Project Glasswing: initial update](https://www.anthropic.com/research/glasswing-initial-update) | Anthropic 用模型发现漏洞，瓶颈从发现迁移到验证、披露、修复。 | agent 能力提升后，系统瓶颈转向人类验证队列、责任流程和安全发布。 |
| 2026-05 | [OpenAI: Virgin Atlantic ships faster with Codex](https://openai.com/index/virgin-atlantic/) | Codex 用于测试、legacy refactor、数据原型和生产工程流程。 | adoption case：agent 改变工程节奏后，瓶颈转向组织协作和 review 流程。 |
| 2026-05 | [Microsoft + EY: From AI pilots to enterprise impact](https://blogs.microsoft.com/blog/2026/05/21/from-ai-pilots-to-enterprise-impact-why-execution-is-the-new-differentiator/) | 强调从 pilot 到 production，需要 intelligence + trust、透明、安全、可问责、可复制执行模型。 | 适合组织层设计原则：agent 系统不是单工具，而是运营模型重构。 |
| 2026-05 | [Google DeepMind: Gemini 3.5](https://deepmind.google/models/gemini/) | Gemini 3.5 Flash 被定位为面向 agents and coding 的高性能模型，强调 long-horizon tasks、tool use、UI control 等 benchmark。 | 可作为“model capability substrate”背景，但应避免让模型分数淹没 harness 讨论。 |
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

## 其他高信号资料

| 年月 | 资料 | 为什么仍值得纳入 |
|:---:|:---|:---|
| 2026-05 | [OpenAI: Running Codex safely at OpenAI](https://openai.com/index/running-codex-safely/) | 给出了清晰的 coding-agent 安全设计原则：bounded environment、low-risk frictionless、high-risk review、agent-native telemetry。 |
| 2026-05 | [Microsoft: Frontier Firms operating model](https://blogs.microsoft.com/blog/2026/05/05/how-frontier-firms-are-rebuilding-the-operating-model-for-the-age-of-ai/) | 组织设计角度很强：人类从逐步执行转向设方向、定标准、评估结果；AI 价值取决于工作如何被重新设计。 |
| 2026-05 | [Anthropic: Agents for financial services](https://www.anthropic.com/news/finance-agents) | 垂直 agent 模板、per-tool permissions、credential vaults、audit logs，适合展示 regulated domains 的 agent design requirements。 |

## 更多可持续追加的高质量资料

| 年月 | 资料 | 可吸收的设计启示 | 适合放在哪里 |
|:---:|:---|:---|:---|
| 2026-05 | [NSA: MCP Security](https://www.nsa.gov/Portals/75/documents/Cybersecurity/CSI_MCP_SECURITY.pdf) | MCP server 是能力入口，也是供应链和权限入口，需要 registry、identity、allowlist、monitoring 和 revocation。 | 工具供应链、connectivity risk。 |
| 2026-05 | [Kiro: Deep spec analysis](https://kiro.dev/blog/deep-spec-analysis/) | spec/requirements 可以作为 agent 前置控制面，让实现之前先稳定目标、约束和验收条件。 | human control surface、plan/process supervision。 |
| 2026-05 | [OpenAI agent improvement loop](https://developers.openai.com/cookbook/examples/agents_sdk/agent_improvement_loop) | traces、evals、prompt/tool changes 可以形成闭环，而不是停留在日志分析。 | observability/eval improvement loop。 |
| 2026-05 | [Microsoft Agent 365](https://www.microsoft.com/en-us/security/blog/2026/05/01/microsoft-agent-365-now-generally-available-expands-capabilities-and-integrations/) | agent inventory、访问控制、治理和组织级可见性正在成为 control plane 的组成部分。 | README 的 runtime/control plane 行，或企业采用部分。 |
| 2026-04 | [NSA/CISA: Careful Adoption of Agentic AI Services](https://media.defense.gov/2026/Apr/30/2003922823/-1/-1/0/CAREFUL%20ADOPTION%20OF%20AGENTIC%20AI%20SERVICES_FINAL.PDF) | agentic service 的风险来自 autonomy、tool use、data access、credential handling 和第三方执行环境的组合。 | 安全边界、治理和 enterprise adoption。 |
| 2026-04 | [Cognition: Multi-agents working](https://cognition.ai/blog/multi-agents-working) | 多 agent 并行的关键不是数量，而是任务切分、写权限约束、冲突处理和可验证合并。 | human manager/verifier、多 agent architecture。 |
| 2026-04 | [GitHub Copilot CLI MCP allowlists](https://github.blog/changelog/2026-04-16-copilot-cli-supports-custom-registry-based-mcp-allowlists/) | MCP allowlist 正在从安全建议变成产品机制。 | 工具供应链的代表信号。 |
| 2026-04 | [A2A protocol milestone](https://www.linuxfoundation.org/press/a2a-protocol-surpasses-150-organizations-lands-in-major-cloud-platforms-and-sees-enterprise-production-use-in-first-year) | agent-to-agent protocol 正在形成互操作层，但也会扩大身份、权限和责任边界。 | 多 agent 协议与治理。 |

## 可写入 Design Space 的新原则草案

2026-06 至 2026-07 窗口新增：

- **Tool output is untrusted input**：工具返回值与 MCP 响应必须与用户输入同等对待。Agentjacking 证明一份伪造的错误报告即可让 agent 执行攻击者控制的代码；把 tool description 与 system prompt 同等审查是相应的防御方向。
- **The enforcement point is itself a design choice**：权限强制点可以落在环内（Copilot CLI 与 Cursor 用 LLM 分类器裁决）、agent 代码之外的网关边界（AWS AgentCore），或 OS 内核（ActPlane）。层次越低越难绕过，但语义越稀薄。tool-call 层的检查可被间接执行路径绕过。
- **Blocking should feed back, not just terminate**：Cursor 的 Auto-review 在拦截时把解释返回给父 agent，父 agent 据此改走安全路径而不打扰用户。权限门可以是引导信号，而不只是终止点。
- **Authorization must expire with the task phase**：会话内一次批准后权限长期驻留是一个可命名的缺陷（lingering authority）。能力应绑定到任务阶段并在闭包条件满足后失效。
- **Rewriting history destroys the cache**：任意改写上下文历史会摧毁 prefix cache，这构成了对 compaction 策略的硬约束，也解释了追加式设计的合理性。模型切换若放在本来就会 cache miss 的压缩时刻，则近乎免费。
- **The harness shapes the model, not only the reverse**：模型在特定 harness 中做 RL，harness 的容错设计（静默修复畸形工具调用）会泄漏进模型权重，使模型在别处产生更差的工具调用。harness 不是中立的评测容器。
- **Harness ownership is moving**：harness 正从「开发者自己写的循环」变成托管服务（AgentCore 的 `CreateHarness`）、可替换后端（Omnigent、AI SDK 7、Warp Oz）和可自动优化的对象（Meta-Harness）。「谁拥有 loop、环境、state 与工具边界」本身成为一个设计维度。
- **Eval validity depends on environment design**：编码基准的分数可能主要来自检索而非推导。若不做仓库历史隔离与出网限制，基准衡量的是 agent 的搜索能力而非修复能力。

此前窗口：

- **Environment parity before model blame**：云端 agent 失败常常不是模型差，而是环境缺依赖、权限、网络或凭证。
- **Bounded programmability beats raw power**：interpreter / programmatic tool calling 给 agent 编程能力，但通过 allowlist bridge 缩小 action surface。
- **Human oversight must be interruptible and mobile**：长程异步 agent 需要远程查看、批准、改方向和审查 diff 的控制面。
- **Memory needs provenance and lifecycle**：memory/context/skills 需要版本、来源、审查、回滚、环境标签和过期机制。
- **Progress must be verified, not inferred**：长任务应维护 verified backlog，而不是靠模型自称“完成了”。
- **Telemetry is not enough without evaluation**：logs 只能解释发生了什么；生产 agent 还需要把 traces 转化为 failure clusters、evals 和改进任务。
- **Connectivity is capability, but also risk**：MCP、SDK、CLI、plugins 放大能力，也扩大 credential、data exfiltration 和 tool misuse 的设计面。
