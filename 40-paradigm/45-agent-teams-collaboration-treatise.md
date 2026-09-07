# 从 ReAct 到 Agent Teams：工程师视角的 Agent 协作机制

> 做了两个月的 Agent 开发，我越来越确信：当前 Agent 能真正完成任务，很大程度取决于 **ReAct（Reasoning + Acting）** 模式的出现。ReAct 来自 Yao 等人 2022 年的论文（ICLR 2023），核心循环是「Thought → Action → Observation」——思考下一步该做什么，执行动作，观察环境返回的信息，再进入下一轮思考。
>
> 单 Agent 的 ReAct 循环已足够完成大量任务；下一步的瓶颈不在「会不会循环」，而在 **一群 Agent 如何像真实团队那样讨论、对齐、跟进与复盘**。基础设施（MCP / A2A / ANP / Matrix）已能让 Agent「发现工具、发现彼此、说话」；缺的是协作语义——该说什么、如何达成共识、说完如何沉淀。
>
> 本文是工程实践与学术对照的专论：从 ReAct 第一性原理出发，经无状态与上下文管理，到 Leader–Worker 的不足、协议层与产业底座、业界五阶段探索，再到一套可落地的 Agent Team 机制提案。关键判断尽量对照可核验来源；文中机制设计本身是实践提案，不是行业标准。数字均为公开实验 / 产品观察，**须用本场景复核**。

先给一个直接答案：

> **Agent ≈ ReAct 循环 + 工具 + 上下文。多 Agent 的增益主要来自协作与知识转移机制，而不是「人头数」——EvoChamber 的受控消融显示：关掉跨 Agent 知识转移后，20 Agent 池与单 Agent 持平。要做成真正的 Team，需要：有兜底能力的 Leader、讨论→共识→执行协议、Worker 横向通信、协商式 OKR、Mission/岗位锚定，以及集体复盘驱动的团队演化；MCP（工具）与 A2A/ANP（Agent 间）解决通道，不自动解决语义。**

**40-paradigm 系列导读：** 上游见 [40](./40-unix-agent-stateless-philosophy.md)（无状态与可组合工具）、[41](./41-ai-engineering-paradigm.md)（组织如何承接提效）。运维侧落地见 [42](./42-agentic-sre-operations-playbook.md)、产业前沿见 [44](./44-agentic-ops-frontier-treatise.md)。横切镜片见 [43](./43-entropy-complex-systems-philosophy.md)。本文回答：**单 Agent 的 ReAct 循环之上，协作机制应如何设计。**

## 摘要

当前 Agent 能真正完成任务，很大程度取决于 ReAct 的「Thought → Action → Observation」循环：外部 Observation 把推理锚定在事实上，相对纯 Chain-of-Thought 显著抑制幻觉与错误累积。大模型无状态，「上下文即记忆」在持续学习彻底解决灾难性遗忘之前仍是可靠工程路径；工程团队的主战场是行动（工具）与观察（结构化反馈）。行业正从完善单 Agent 走向 Agent Teams；主流层级编排仍偏「调度 + 验收」，缺方案讨论、协商式目标、执行期横向通信与集体复盘。协议层上 MCP 管垂直工具接入，A2A / ANP 管水平互通；AgentTeams、Magentic-One 等证明工程底座已可采购，瓶颈仍在协作语义。本文沿「分类 → 分工 → 目标 → 经验 → 演化」串起学术探索，并提出七项机制。Meta-Team 显示手工 MAS 在 9 列评测中有 6 列不及单 Agent；EvoChamber 显示无跨 Agent 转移时多 Agent 池可不优于单 Agent——堆人不是加法。

**关键词：** ReAct；Agent Teams；Leader–Worker；OKR；A2A；MCP；横向通信；集体复盘；上下文管理

---

## 目录

- [摘要](#摘要)
1. [开篇：ReAct 循环何以成为 Agent 的工作原理](#1-开篇react-循环何以成为-agent-的工作原理)
2. [第一性原理分拆：思考、行动、观察](#2-第一性原理分拆思考行动观察)
3. [无状态本质与上下文管理](#3-无状态本质与上下文管理)
4. [从单 Agent 到 Agent Teams：必然的进化方向](#4-从单-agent-到-agent-teams必然的进化方向)
5. [当前 Leader–Worker 架构的不足](#5-当前-leaderworker-架构的不足)
6. [协议层与产业底座：能说话 ≠ 会协作](#6-协议层与产业底座能说话-会协作)
7. [业界已有的探索：五阶段生命周期链](#7-业界已有的探索五阶段生命周期链)
8. [设计提案：真正的 Agent Team 机制](#8-设计提案真正的-agent-team-机制)
9. [与现有框架的差异](#9-与现有框架的差异)
10. [类比人类组织演化](#10-类比人类组织演化)
11. [Agent Autonomy 分级](#11-agent-autonomy-分级)
12. [结语](#12-结语)
13. [参考文献](#13-参考文献)

---

## 1. 开篇：ReAct 循环何以成为 Agent 的工作原理

做了两个月的 Agent 开发，我越来越确信：当前 Agent 能真正完成任务，很大程度取决于 **ReAct（Reasoning + Acting）** 模式的出现。

ReAct 来自 Yao 等人 2022 年的论文（ICLR 2023），核心循环是「**Thought → Action → Observation**」——思考下一步该做什么，执行动作，观察环境返回的信息，再进入下一轮思考。[^react] 这就是人完成任务时的智能表现：推理、行动、根据结果调整，如此往复。

这里的 **Observation** 指 Agent 执行动作后从外部获取的信息——API 返回结果、命令输出、文件内容。它本质上承担了「反馈」功能，让推理始终锚定在事实上，而非模型自己编造。ReAct 相比纯 Chain-of-Thought 的核心优势就在这里：有了外部观察的「接地」，幻觉和错误累积被大幅抑制。论文在问答与事实验证任务上表明：与 Wikipedia API 交互可克服 CoT 中常见的幻觉与错误传播；在 ALFWorld 与 WebShop 等交互决策基准上，相对模仿学习 / 强化学习基线，成功率分别绝对高出约 **34** 与 **10** 个百分点（仅用一到两个 in-context 示例）。[^react]

需要同时看清边界：ReAct 不是唯一编排形态。工业上常见变体包括 Plan-and-Execute（先整体规划再逐步执行）、Reflexion / Self-Refine（对轨迹做反思后再试）、以及把多个 ReAct 子循环嵌进状态机（如 LangGraph）或编排器（如 Magentic-One 的 Orchestrator）。[^magentic-one] **它们共享同一内核：推理必须被工具与环境反馈约束；差异在于规划粒度、反思频率与多 Agent 如何嵌套。**

Agent 本质上就是这个循环的实现，而且可以非常简单。pi-agent[^pi-agent]（文稿撰写时社区统计约 7.6 万 star，以仓库实时数据为准）的核心循环实现通常只需数百行代码量级。把这个循环构建好就能完成任务，后续的记忆压缩、Skill 加载都是「上下文管理」层面的优化，**不改变基本工作原理**。

所以 Agent 可以用任何语言实现。Blade AI 使用 LangGraph 也只是借用了它的状态机编排功能——当需要多个 ReAct Agent 有序串联时，启动一个子 Agent 就是启动一个新的 ReAct 循环。

---

## 2. 第一性原理分拆：思考、行动、观察

既然主流生产 Agent 都可还原为「通用大模型 + ReAct 式循环 + 工具/上下文」，那构建的 Agent 在能力边界上就是通用智能体——理论上可以在任何领域做任何事，区别只在于工具、权限与上下文工程。这不等于「已经在任意领域可靠」：可靠性由评测、护栏与人在环上共同决定。

沿 ReAct 三环节做分析：

### 2.1 思考（Reasoning）：大模型基座决定，是 Agent 的「智商」

模型的推理能力直接决定 Agent 上限——能不能理解复杂指令、做多步推理、在信息不完整时合理假设。底层模型能力越强，后续可扩展的事就越多，这是不可违背的约束。ReAct 原始实验也印证了这点：同框架换更强模型，所有任务表现直接提升。[^react]

### 2.2 行动（Acting）：工具集决定，是 Agent 的「手脚」

同一个模型配不同工具就能做不同的事——给自行车只能短距，给汽车就能远行。ReAct 在 ALFWorld 上展示：有动作空间后成功率相对纯推理大幅提升（论文相对 RL/IL 基线约 +34 个百分点）。光有思考没有行动，智能无法体现。[^react]

行动侧的工程要点不止「挂更多 API」：工具描述是否可机读、参数 schema 是否严格、是否幂等、失败是否可回滚、权限是否按角色切开——这些决定 Observation 质量，也决定多 Agent 时的 blast radius。

### 2.3 观察（Observation）：环境返回的信息，是 Agent 的「感知系统」

必须给大模型真实、结构化的观察信息。如果反馈只说「出错了」但不说为什么，Agent 就像得到模糊评价的人一样陷入混乱——只能瞎猜。工具返回详细的错误描述加修复建议，远比一个错误码有用。

观察也是幻觉抑制的主阀门：ReAct 相对 CoT 的优势，本质上是把「模型自说自话」换成「模型—环境闭环」。

### 2.4 工程团队的着力点

我们无法单独解决模型的「思考」问题，那是算法与预训练团队的主场。但工程可以积极解决「行动」与「观察」——提供更好的工具、返回更结构化的执行结果、把关键写操作留在人在环上。这能极大减少幻觉和弯路，不是因为模型变聪明了，而是因为它做决策的依据更充分了。

与本库 [42](./42-agentic-sre-operations-playbook.md) / [44](./44-agentic-ops-frontier-treatise.md) 一致：运维场景尤应先把只读取证与证据包做扎实，再谈有界自治。

---

## 3. 无状态本质与上下文管理

无状态哲学主线（Unix → Agent、遗忘何以强于失控记忆、状态应落在外部可组合介质）见 [40](./40-unix-agent-stateless-philosophy.md)。本节只补 **Agent Team 所需的工程增量**：为何「上下文即记忆」仍是生产默认，以及多 Agent 时外部记忆为何变成协作前提。

大模型每次调用都是独立前向计算；「记得上次」只是把历史再塞进 context。人学习会改写突触，模型不会——经验若不落盘、下次不回灌，等于没发生。因此 Agent 不会「自动变强」，只在你建了外部记忆的那部分变强：RAG、记忆压缩、Skill、知识库、轨迹仓库，以及团队级组织记忆（工程对照如 Organizational Memory 一类产品能力）[^agno-memory]，都是在替模型做它做不到的持久化。

持续学习（持续预训练 / LoRA 微调 / 对齐）有进展，但灾难性遗忘远未根除。[^continual-2026] 在此之前，「上下文即记忆」仍是最可依赖路径；持续学习是补充，不是替代。即便将来模型能在使用中自我进化，「行动 / 观察」与多 Agent 的共享工件、通信预算仍是工程主战场——个体再强，协作缝上仍会失分。ICLR 2026 前后 *Memory for Agentic Systems* 工作坊亦把记忆标为一等议题。[^iclr-memory]

---

## 4. 从单 Agent 到 Agent Teams：必然的进化方向

当前行业大量精力仍在单 Agent 的构建——完善一个「人」。但只要任务跨越多个专业域、多个工具面、多个时间尺度，单循环就会撞上 context 上限、认知负担与责任边界问题；未来一定会大规模出现 Agent 间的协作问题，就像人类社会从个体生存到部落、城邦、国家的演化。

ReAct 的成功来自对「个体智能」的正确抽象。Agent 间的协作同样可以借鉴人类组织经验——几千年试错演化出的管理规则不一定最优，但经过了充分实践验证。Meta-Team 等工作明确引用组织心理学的「团队反思性（team reflexivity）」等理论来指导 MAS 演化设计，从学术角度验证了这条路。[^meta-team]

Agent 管理比人「简单」的一面：没有情绪、即时响应、沟通摩擦更低。人类管理中最难的部分（情绪、利益博弈、部分信息不对称）在 Agent Teams 中显著减弱。许多人觉得 AI「提不了效」的地方——沟通开会、等回复——恰恰是 Agent 更容易压缩的部分。

Agent 也有劣势，必须写进架构：

| 优势 | 劣势 / 风险 |
| ---- | ----------- |
| 无情绪摩擦、秒级回合 | context window 有限，长轨迹易丢全局 |
| 可并行、可复制 | 缺常识与隐性知识，易「合规但离谱」 |
| 通信可全量审计 | 幻觉可经 handoff 系统性放大 |
| 角色可热插拔 | 通信全连接时 token / 延迟爆炸 |

因此：「必然走向 Teams」不等于「无条件堆 Agent」。后文 Meta-Team / EvoChamber 的证据会反复指向同一句：**没有好的协作与转移机制，多 Agent 可以是减法。**

---

## 5. 当前 Leader–Worker 架构的不足

主流多 Agent 框架（CrewAI、AutoGen / AG2、MetaGPT，以及大量自建编排）常见 **Leader–Worker / Orchestrator–Specialist** 形态。以 CrewAI 的 hierarchical process 为例，manager agent 的职责被概括为协调、委派、验证（coordinates / delegates / validates）。[^crewai] 有基于 capability 的分配和结果验证，但默认交互仍偏「调度器」：方案讨论、共识达成、执行期动态重规划、Worker 间自由求助，通常不是一等公民。

需要立刻加一条公正说明：并非所有框架都「Worker 完全隔离」。AutoGen 系有 GroupChat 等多方对话；Microsoft **Magentic-One** 用 Orchestrator 规划、跟踪进度并在出错时重规划，同时调度 Web/文件/代码等专长 Agent——专长 Agent 之间仍多经由编排器，而非对等协商式 OKR。[^magentic-one] 本文批评的是**产业默认的层级派活模式**，不是否认任何框架存在对话能力。

对比现实技术团队，差距仍明显：

| 维度 | 现实团队 | 常见 Agent Teams 默认态 |
| ---- | -------- | ----------------------- |
| **Leader 角色** | 资深专家 + 管理者，能兜底 | 任务分发器 + 结果验收器 |
| **任务下达** | 先有方案 → 与 Worker 讨论 → 共识后执行 | 直接拆分 → Worker 埋头干 |
| **Worker 间交流** | 自由沟通、互相求助 | 弱横向流；多经 Leader 中转 |
| **进度管理** | OKR / 里程碑 / 风险预警 | 多为最终交付验收 |
| **方案变更** | Leader 审查合理性 | 常无显式变更门闸，出错才返工 |

核心缺陷可以收敛成一句：**缺的不只是「再多一个聊天室」，而是执行期的对等信息流 + 目标层协商 + 变更审查 + 复盘资产化。** 通信拓扑综述（arXiv:2502.14321）识别 Flat / Hierarchical / Team / Society / Hybrid 等架构，并讨论 MCP / A2A / ANP 等协议层；需要灵活交互、频繁互相求助时，扁平 / P2P 结构往往更合适——但必须以通信预算与路由策略约束爆炸。[^comm-survey]

### 5.1 案例：AgentTeams——Manager–Workers 的云原生完成度

阿里 AgentScope 生态下的开源 **AgentTeams**[^agentteams] 是工业界把「多 Agent 协作平台」做成云原生系统的代表作之一（以仓库与架构文档为准）：

- **Kubernetes 控制面**（controller + CRD）声明式管理 Manager / Worker / Team；
- **Higress AI Gateway** 统一托管模型路由、MCP 与凭据（Worker 侧消费令牌，降低密钥扩散）；
- **Matrix**（Tuwunel 等）作为 Agent 与人共用的通信总线；
- **MinIO** 等对象存储提供跨 Worker 共享文件，降低重复塞 context 的 token 成本；
- **多运行时**：OpenClaw / QwenPaw（及兼容别名）/ Hermes 等可在同一 Matrix Room 共存；人类用 Element Web 等旁听与介入。

架构上，Manager 协调 Workers（以及可选的带 Team Leader 的 Teams），Human 经 Matrix 参与；通信「可见、可干预、无隐藏调用」是其产品主张。[^agentteams]

这套设计把 Leader–Worker 的**工程侧**问题解决得相当彻底：凭据安全、通信基础设施、共享存储、可观测性、人在环、多运行时兼容。但结构语义上，它仍首先是 **Manager–Workers 编排平台**：通道与治理强，并不自动附带「方案共同讨论 → OKR 协商 → 分歧仲裁 → 集体复盘 → 团队演化」的完整协作协议。Skills 默认仍更接近角色能力包，而非团队共享的方法论资产。

这个案例反过来印证本文判断：Agent Teams 现阶段的瓶颈，越来越不在「能不能部署一群 Agent」，而在协作机制——基础设施解决了「能不能说话」，还没解决「该说什么、怎么达成共识、说完之后如何沉淀」。

---

## 6. 协议层与产业底座：能说话 ≠ 会协作

2025–2026 年，多 Agent 工程的一个清晰分水岭是：**框架内战让位于协议层互通**。把「工具怎么接」和「Agent 怎么互相发现 / 委托」拆开，比绑死某一个编排库更可持续。

| 协议 | 主要解决 | 典型能力 | 与本文关系 |
| ---- | -------- | -------- | ---------- |
| **MCP**（Model Context Protocol） | **垂直**：Agent ↔ 工具 / 数据 | 工具发现、调用、鉴权边界 | 强化「行动 / 观察」质量；不定义团队语义 |
| **A2A**（Agent2Agent） | **水平**：Agent ↔ Agent | Agent Card 发现、任务委托、消息 / 工件交换；Google 2025-04 发布，社区持续演进 | 提供 P2P / 跨框架委托的通道 |
| **ANP**（Agent Network Protocol） | **开放网络**：发现、身份、协商 | DID 身份、能力描述、去中心发现与安全通道 | 面向跨组织 Agentic Web；企业内更常先落 A2A/MCP |

官方表述上，A2A 明确宣称与 MCP **互补**：MCP 给 Agent 工具与上下文，A2A 让不同框架上的 Agent 作为对等体协作，而不是只把对方当工具。[^a2a][^anp] AgentScope 等亦演示以 Nacos 等作为 A2A Registry，做跨语言 / 跨框架发现与治理。[^agentscope-a2a]

产业选型上，常见组合是：

- **LangGraph**：强状态机与可控工作流；跨框架互通常需额外 A2A 适配；
- **CrewAI**：快角色编排；层级派活默认强；
- **AutoGen / AG2 / Microsoft Agent Framework**：对话式多 Agent 与编排器路线（含 Magentic-One）；
- **Google ADK 等**：偏协议先行的云原生 Agent 运行时。

工程结论应写成：

> **协议成熟度 ≠ 协作成熟度。** MCP/A2A/ANP/Matrix 解决互操作与审计面；讨论共识、OKR、复盘与演化仍要在应用协议之上显式设计——这也是第 8 节提案的位置。

---

## 7. 业界已有的探索：五阶段生命周期链

这几年多 Agent 协作的研究成果，如果一篇篇孤立地看，很像一堆互不相干的技巧：有人在做任务分解，有人在搬 OKR，有人在做失败复盘。但只要把它们串到同一个问题上——「如何把一群各干各的 Agent，组织成一支真正的团队」——它们会立刻落到一条清晰的生命周期链上：

1. 先要知道有哪些组织形态可选（**分类**）；
2. 选定形态后要把活分下去（**角色与流程**）；
3. 分好工要让大家对齐目标（**目标管理**）；
4. 干完一轮要把经验沉淀下来（**经验积累**）；
5. 长期还要让团队组织本身不断进化（**团队演化**）。

这五个阶段不是硬凑的分类，而是一环扣一环——每一环都是在回答上一环留下的问题。下面沿这条链走一遍，每一站标出它解决了什么、又留下了什么缺口；而这些缺口叠加起来，恰好就是第 8 节要补的东西。

### 7.1 阶段① 分类与框架：先搞清有哪些组织形态可选

要组织一支团队，第一步得知道「团队」可以长成什么样。2025 年初的协作机制综述（arXiv:2501.06322）从协作类型（合作／竞争／竞合）、协调策略（规则／角色／模型驱动）、通信结构（中心化／去中心化／层级）、动态性（静态／运行时可变）等维度归纳多 Agent 系统；与通信中心综述合读，可得到 Flat/P2P、Hierarchical、Team、Society、Hybrid 等基本组织形态地图。[^collab-survey][^comm-survey]

最有价值的是「反银弹」：**没有哪一种结构能普适所有场景**；越是需要灵活交互、频繁互相求助的场景，全连接 / 扁平结构往往比纯层级更合适——但必须用路由与预算防止 context 爆炸。

这张地图告诉了我们「有哪些形态」，却没回答「选定一种形态后，Agent 该怎么在里面真正协作起来」——于是问题自然滑向下一站：定了形态，活怎么分？

### 7.2 阶段② 角色与流程：定了形态，活怎么分下去

分工有两种截然不同的思路，MetaGPT 和 Agent-Oriented Planning 正好站在两端——一个把流程写死求稳，一个让分解可变求准。

**MetaGPT（ICLR 2024）** 把人类软件团队的标准作业流程（SOP）直接编码进多 Agent 系统：产品经理写需求文档、架构师出设计、项目经理拆任务、工程师写代码、QA 验收，Agent 之间传递的是结构化文档而不是自由聊天。[^metagpt] 洞察朴素却关键——在容易层层跑偏的多 Agent 场景里，用标准化流程约束交互，比让 Agent 自由对话更能抑制错误传播。代价是 SOP 手工设计、偏静态，角色间多为单向文档流，回头协商与动态调整空间有限。

**Agent-Oriented Planning（ICLR 2025, arXiv:2410.02189）** 补上「死板」：不预设固定 SOP，而由 Meta-Agent 按可解性、完备性、无冗余等原则动态拆解，再用 Reward Model 评估分配质量并触发重规划，同时为 Agent 维护「代表作品」式能力画像以供匹配。[^aop] 关键一步是把「初始分解」当成可推翻的中间结果，而不是终稿。

两篇合起来：既要有流程约束保证不跑偏，又要能动态纠偏保证分得准。共同盲点仍在——分解多自上而下，Worker 很少参与目标制定。问题推到第三站：目标怎么对齐？

### 7.3 阶段③ 目标管理：分好了工，怎么对齐目标

对齐目标这件事，人类组织早有 OKR。**OKR-Agent**（*Agents meet OKR*，arXiv:2311.16542）把层级 Objective / Key Results 生成与多级评估搬进 Agent 世界：递归拆分子目标、按 KR 与职责分配 Agent，并用多级评价精炼方案。[^okr-agent]

它证明 OKR 机制在 Agent 上可行。局限也清楚：主路径仍是「一个层级系统把目标切碎再分下去」，不等于多个执行 Agent 在制定阶段就 KR 可行性做对等协商。Worker 仍偏目标接收者，难以在开工前说「这个 KR 做不到，因为……」。

### 7.4 阶段④ 经验积累：干完一轮，怎么把经验变成下次的能力

**Experiential Co-Learning（ACL 2024）** 让 instructor / assistant 等软件开发 Agent 从历史轨迹提取「捷径经验」（可跳过已知步骤的高效路径），写入经验池，新任务用检索增强推理复用。论文报告相对最强多 Agent 基线 ChatDev，综合 Quality 从约 **0.4267** 提升到约 **0.7304**（Completeness × Executability × Consistency）；去掉双方经验后 Quality 回到约 **0.4267**。[^ecl]

但这类经验仍偏「角色 / 配对内部怎么把某类活干得更熟」，尚未自动上升为「这个团队组合与协作方式」的组织资产。个体会进化了，最后一个问题浮出：团队整体如何变强？

### 7.5 阶段⑤ 团队演化：个体会了，怎么让团队组织整体进化

这是整条链最前沿、也最接近「真正的团队」的一站。

**Meta-Team**（*Evolve as a Team*，arXiv:2605.29790）主张：MAS 不应只作为团队执行，还应作为团队进化。[^meta-team] 它保留各 Agent 本地执行上下文，并在任务后协调通信以交换分布式证据，在三层做自演化：

- **Agent 层**：审视自身执行、更新个体 scaffold / 技能；
- **交互层**：回顾协作史、更新队友画像与沟通方式；
- **团队层**：集体讨论组成是否合理，引入或退休角色、修订共享规则。

相对不演化的手工 MAS，平均约 **+6.6%**；更刺眼的是：**初始手工 MAS 在 9 个评测列中有 6 列不及单 Agent**——没有好的协作，多 Agent 可以是减法。演化后的 Meta-Team 则在所述评测列上同时超过单 Agent 与固定 MAS，长程任务增益更明显。[^meta-team]

**EvoChamber**（arXiv:2605.11136）在个体 / 团队 / 种群三层做测试时共进化：CoDream 在失败或分歧后触发 Reflect → Contrast → Imagine → Debate → Crystallize，并做**非对称**知识转移（强→弱、按 niche 补缺口），避免对称广播抹平专业化；团队层在线选择协作结构，种群层 fork / merge / prune / seed。[^evochamber] 主结果上，相对最强基线在困难数学流等设置有大幅相对提升；消融中去掉 CoDream 带来最大单次跌幅（文中约 **−10.8%** 量级）。更关键的受控隔离：在 AFlow-Stream 的一段数学子序列上，**维持 20 Agent 池但关闭 CoDream 时，表现与单 Agent 完全持平（文中同为 0.633）**——池与生命周期脚手架本身不产生增益，跨 Agent 转移才激活集体学习。[^evochamber]

两篇殊途同归：**多 Agent 的价值主要来自协作与知识转移机制；堆数量本身不产生价值。** 「100%」可作为口号记忆，写作与决策时应以消融条件为准，避免过度外推到一切基准。

### 7.6 五站合读：已解决什么，还缺什么

分类、分工、目标、经验、演化——每一环都有人单独攻克，却少有工作把它们串成可持续运转的团队操作系统。几乎所有工作仍缺席或偏弱的三块是：

1. **执行期 Worker 横向通信**（而不只是任务后演化讨论）；
2. **激发式而非纯命令式的管理**（对齐分布与沟通风格）；
3. **Mission 层方向锚定**（只有演化类工作部分触及组织层修订）。

这三块空白，正是下一节要动手去补的地方。

---

## 8. 设计提案：真正的 Agent Team 机制

下图（文字结构）是本节七个机制的整体关系。Leader 与 Worker 通过讨论共识建立协作，Worker 之间通过横向通信共享信息，进度用 OKR 度量，上层由 Mission 与宗旨锚定方向，下层由集体复盘沉淀经验，形成可自主运转的闭环。

```
                    Mission / 宗旨（方向锚定）
                              │
                    ┌─────────┴─────────┐
                    │   讨论 → 共识 → 执行   │
                    │   （OKR 契约）         │
                    └─────────┬─────────┘
           Leader◄───────────┼───────────►Worker
              │              │              │
         启发式管理      横向 P2P      岗位 JD / Skill
              │              │              │
                    └─────────┬─────────┘
                         集体复盘
                    （方法论 / playbook / 反模式）
                              │
                         团队演化
```

本节为**实践提案**：可与 Meta-Team / EvoChamber / Magentic-One / AgentTeams 对照实现，但不声称已有单一开源仓库完整覆盖。

### 8.1 重新定义 Leader：从分发器到资深专家兼管理者

现有框架中的 Leader 更像调度器——切任务、扔 Worker、收结果。真正能带队的 Leader 不是这样。

对小组长的定义有三条：业务交付、技术竞争力构建、团队建设。落到 Agent 世界同样成立：

- **业务交付**：保证 Team 输出满足需求；
- **技术竞争力构建**：保证方案水准，而不是「能跑就行」；
- **团队建设**：让 Team 随时间变强（沉淀经验、优化协作）。

Agent 侧 Leader 应具备四个特征：

1. **深度参与方案制定**——有大方向，而不是黑盒切分；
2. **与 Worker 讨论后再执行**——执行侧约束前置；
3. **具备兜底能力**——Worker 卡住时能亲自接手；因此 Leader 应用更强模型、更长 context、更全局的历史访问权，而不是同质 prompt 换皮；
4. **负责方案变更审查**——局部最优不得毁掉全局。

Magentic-One 的 Orchestrator「规划—跟踪—失败重规划」是工业上接近「能兜底的编排大脑」的一极；本文还要求它在开工前完成协商式对齐，而不仅是执行中纠偏。[^magentic-one]

### 8.2 启发式管理：Leader 是激发者，不是命令者

这是工程实践里容易被忽视、但可复现的一点。

同样一个 Agent，给它简短、命令式、甚至带负面暗示的描述——「你负责 X，别搞砸」——产出往往保守、机械、只达最低要求；换成鼓励式、探索式引导——「这一块你视角更细，可以先给几种切入，不确定处我们一起讨论」——同一模型同一任务，深度与创造性常有可见提升。

这不是神秘学。主流大模型经 RLHF / DPO 等对齐后，训练分布里「合作、鼓励、探索、开放」语境更常对应高质量、有建设性的续写；「命令、否定、防御」更常激活保守、免责、低风险续写。沟通风格在采样分布上选择行为模式。

这与人类「心理安全感」现象同构但机理不同：人类侧是社会情绪与激励；Agent 侧是条件分布。落地设计：

- 下发：「你的视角比我细，先说你会怎么切入？」而非「去做 X」；
- 失败：「X 因素考虑过吗？换个角度」而非「错了，重做」；
- 验收：既指出缺陷，也锚定做得好的模式，供下次复用。

若 Leader 只做冷冰冰派活与挑错，等于系统性压低 Agent 的探索带宽。「约只用了 60% 能力」是工程体感比喻，不是可迁移的计量定律——但方向成立：管理语气是团队建设的第一层。

### 8.3 讨论 → 共识 → 执行：任务启动的三步走

当前多数框架：任务到达 → Leader 拆分 → Worker 开工，中间缺少「对齐」。人类团队最有价值的一步往往在这里：Leader 说清目标与方向，Worker 反馈可行性、资源与实现层陷阱，修正后再动工——把执行约束前置，避免半途返工。

显式协议建议：

| 阶段 | 名称 | 内容 |
| ---- | ---- | ---- |
| **Phase 1** | Leader Proposal | 初版目标、拆分、角色假设 |
| **Phase 2** | Worker Feedback | 可行性、约束、替代路径 |
| **Phase 3** | Consensus | 修订方案、形成 OKR、acknowledge 后执行 |

Agent 侧沟通成本远低于人类会议：三阶段可异步并行、秒级完成。这正是「开会拖效率」在 Agent Team 里可被压缩的具体落点——前提是协议存在，而不是指望模型自发对齐。

### 8.4 Worker 间横向通信：打破「埋头干」的信息孤岛

真实团队里，分工不等于禁言。Agent Team 至少需要三种模式：

1. **主动广播**：阶段成果或共同风险 push 给相关方；
2. **被动查询**：pull 中间结果或专业判断；
3. **求助升级**：先横向找同伴，再升级 Leader。

通道已具备：A2A、ANP、AgentTeams 的 Matrix、以及编排器内的消息总线。仍缺的是**通信策略**：

- **熟人网络**：历史合作过的边优先；
- **技能索引**：按能力域路由；
- **通信预算**：每任务 / 每 Agent 的消息与 token 上限；
- **主题房间 / 线程**：避免全员风暴。

学术上「执行期对等通信 + 预算」仍相对空白，适合工程先行，并用轨迹审计评估噪声与增益。

### 8.5 基于 OKR 的目标与进度管理

OKR：明确 Objective，拆出可量化 Key Result，执行中对齐完成度。

分层建议：

- **Team OKR**：Leader + Workers 共同讨论的顶层契约；
- **Worker OKR**：认领 KR 并拆成可执行动作。

例：Team Objective =「订单 API P99 从 800ms 降到 200ms」；KR1 慢查询占比 < 5%（DBA）；KR2 缓存命中率 > 95%（缓存）；KR3 压测 3000 QPS 达标（压测）。每个 KR 有 owner、验收方式、时点。

进度机制可直接挂在 OKR 上：里程碑触发检查、偏离预警升级、KR 达成即关闭。相对 OKR-Agent 的单树递归，Agent Team 需要的是**协商式 OKR**：执行 Agent 必须能在制定阶段改写不可达的 KR（「长尾业务下命中率上限约 82%」只有执行侧说得出）。

### 8.6 岗位要求与团队宗旨：让每个 Agent 知道自己为什么在这里

被动匹配是「你有什么能力就派什么活」。真实团队是岗位驱动：按任务定义 JD，再校验谁上岗。

JD 建议包含：能力要求、Skill 池、质量标准、汇报关系、SLA。好处：

1. 复杂度上升时可提高门槛，不被现有 Agent 上限锁死；
2. 同一模型 + 不同 JD = 不同岗位，资源更灵活。

再往上：

| 层级 | 回答 | 示例 |
| ---- | ---- | ---- |
| **Mission** | 为何存在 | 「保障线上稳定性」 |
| **核心宗旨** | 原则 | 「安全优先于速度」 |
| **OKR** | 本周期做什么 | 可量化 KR |

Mission / 宗旨在人类不下场时提供取舍锚点：资源争用、「快但脏 vs 慢但净」、KR 冲突时，局部视角不会自动收敛。它也是启发式管理的边界：鼓励探索时，Agent 仍知道什么不可破。

### 8.7 集体复盘与团队演化：从个体自进化到组织学习

单 Agent 学「我如何把某类任务做得更好」；Team 复盘学「我们这种组合与协作如何更好」。

Leader 主持、相关 Worker 参与，结构化产出三类资产：

1. **方法论**；
2. **协作 playbook**（谁与谁、何种通信模式有效）；
3. **反模式**。

存储分两层：团队共享知识库 + 岗位专属经验。触发节奏可借鉴 Meta-Team：

- 过程中**微复盘**（我的输出如何影响你的决策）；
- 阶段后**中盘**（更新队友画像与沟通）；
- 结束或周期**总复盘**（组成、规则、引人 / 退休）。

EvoChamber 的受控结果再次提醒：没有跨 Agent 转移与协作结构，维持一个大池本身可以零增益。没有讨论共识、横向通信、OKR 跟进与集体复盘，「搞个 Team」就可能是纯成本。

---

## 9. 与现有框架的差异

| 维度 | CrewAI 层级 / 典型派活 | Magentic-One | Meta-Team | 本文 Team 机制 |
| ---- | ---------------------- | ------------ | --------- | -------------- |
| **Leader** | 分发 + 验收 | Orchestrator 规划 / 重规划 | 演化期协作，无固定「管理者」叙事 | 资深专家 + 兜底 + 变更审查 |
| **任务启动** | 常直接拆分 | 执行中规划为主 | 手工初始 scaffold | 讨论后产出协商式 OKR |
| **Worker 交互** | 弱横向 | 多经 Orchestrator | 任务后演化通信强 | 执行中 P2P + 求助升级 |
| **进度管理** | 偏最终验收 | 编排器跟踪进度 | 非 OKR 中心 | OKR + 里程碑 / 偏离预警 |
| **经验沉淀** | 弱 | 模块可插拔，非组织学习核心 | 三层集体进化 | Leader 驱动复盘 → 团队资产 |
| **团队目标** | 任务级 | 任务级 | 演化修订组织规则 | Mission + 宗旨 + OKR |
| **协议位** | 框架内 | AutoGen 生态 | 研究框架 | 假定 MCP + A2A/Matrix 之上 |

说明：上表是相对定位，不是完整能力审计；各项目仍在演进。AgentTeams 更接近「强通道 + 强治理的 Manager–Workers 平台」，可与本文机制叠加，而不是互斥。

---

## 10. 类比人类组织演化

| 阶段 | 类比 | Agent 协作特征 |
| ---- | ---- | -------------- |
| **部落（当前常见）** | 简单分工，做完就散 | 弱记忆、弱沉淀 |
| **城邦（近期）** | 角色、规则、进度 | 「讨论 → 共识 → 执行 → 验收」 |
| **企业（中期）** | Mission + OKR，复盘，人才梯度 | Worker 横向沟通，岗位 JD |
| **生态（远期）** | 多 Team 网络 | 契约、竞合、跨组织 A2A/ANP 知识交换 |

多数生产系统仍在「部落→城邦」；第 8 节提案对应「企业」层最低完备集。生态层依赖协议与治理成熟度，见第 6 节与本库 [44](./44-agentic-ops-frontier-treatise.md)。

---

## 11. Agent Autonomy 分级

| 等级 | 名称 | 含义 |
| ---- | ---- | ---- |
| **L1** | Copilot | 人主导，Agent 辅助 |
| **L2** | Task Agent | 人给明确指令，Agent 执行单步 |
| **L3** | ReAct Agent | 人给意图，Agent 自主多步循环（如 Blade AI Vibe Chaos） |
| **L4** | Team Agent | 多 Agent 协作，有内部协调机制 |
| **L5** | Autonomous Organization | 自主发现问题、组建团队、协作执行、集体演化 |

当前整体处于 **L3 → L4**。Meta-Team / EvoChamber 标志 **L4 → L5** 的研究过渡开始——证明演化机制有增益，不等于生产上可无人值守组建组织。

与本库 [44](./44-agentic-ops-frontier-treatise.md) 的比例制自治（Observe / Advise / Act with approval / Act autonomously）正交：

- 本表描述**协作形态成熟度**；
- 比例制描述**对生产写操作的风险授权**。

L4 Team 仍可整体运行在 Advise 或 Act with approval；切勿把「会开会的 Agent」误写成「可无人批准改生产」。

---

## 12. 结语

ReAct 证明：对人类「推理—行动—反馈」做抽象建模是可行的。多 Agent 协作同样应从人类组织经验提取模式——角色分工、目标管理、进度跟踪、集体复盘、经验传承——并映射为工程协议。Agent 消除了部分情绪摩擦与等待成本，使这些机制能以更高频率、更低成本运转；但幻觉、context 上限与 handoff 放大误差，要求更严的观察质量、通信预算与人在环。

学术界已用 Meta-Team、EvoChamber、OKR-Agent、Experiential Co-Learning 等回应「团队如何对齐与进化」；产业用 MCP / A2A / ANP、AgentTeams、Magentic-One 等回答「如何部署与互通」。夹在中间的空白，正是本文的七项机制：**通道已有，语义未建。**

若只记三句：

1. **Agent ≈ ReAct + 工具 + 上下文**；工程主战场在行动、观察与外部记忆。
2. **多 Agent 的增益来自协作与知识转移，不是人头数**——无转移的池可以等于单 Agent。
3. **讨论共识、横向通信、协商式 OKR、Mission 与集体复盘**，才是 Team 与「一堆 Worker」的分水岭。

一个人的 Agent 时代在 L3 已走通；一群 Agent 的 Team 时代，取决于我们是否愿意把组织经验写成可执行协议，而不是只多起几个进程。

---

## 13. 参考文献

[^react]: Shunyu Yao et al., *ReAct: Synergizing Reasoning and Acting in Language Models*, ICLR 2023（预印本 2022）. <https://arxiv.org/abs/2210.03629>

[^pi-agent]: earendil-works / pi（pi-agent）. <https://github.com/earendil-works/pi>

[^magentic-one]: Microsoft Research, *Magentic-One: A Generalist Multi-Agent System for Solving Complex Tasks*. <https://www.microsoft.com/en-us/research/publication/magentic-one-a-generalist-multi-agent-system-for-solving-complex-tasks/>

[^continual-2026]: LLM 持续学习综述方向代表性条目例：arXiv:2603.12658（*Continual Learning in LLMs*，2026）。以 arXiv 页面与后续正式发表版本为准；文中仅取其「灾难性遗忘未根本解决」之共识性判断。

[^meta-team]: *Evolve as a Team: Collaborative Self-Evolution for LLM-based Multi-Agent Systems*（Meta-Team），arXiv:2605.29790. <https://arxiv.org/abs/2605.29790>；实现 <https://github.com/zz-haooo/Meta-Team>

[^crewai]: CrewAI, Hierarchical Process / manager agent（协调、委派、验证）. 以官方文档为准。

[^comm-survey]: *Beyond Self-Talk: A Communication-Centric Survey of LLM-Based Multi-Agent Systems*, arXiv:2502.14321, 2025. <https://arxiv.org/abs/2502.14321>

[^agentteams]: agentscope-ai / AgentTeams. <https://github.com/agentscope-ai/AgentTeams>；架构说明见仓库 `docs/design/architecture.md`

[^a2a]: Google Developers Blog, *Announcing the Agent2Agent Protocol (A2A)*, 2025-04-09. <https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/>；规范与 SDK 见 <https://github.com/a2aproject/A2A>

[^anp]: Agent Network Protocol Technical White Paper, arXiv:2508.00007. <https://arxiv.org/abs/2508.00007>；社区实现 <https://github.com/agent-network-protocol/anp>

[^agentscope-a2a]: Alibaba Cloud, *Nacos A2A Registry: AgentScope Enables Cross-Language and Cross-Framework Interoperability*, 2026-01. <https://www.alibabacloud.com/blog/nacos-a2a-registry-agentscope-enables-cross-language-and-cross-framework-interoperability_602821>

[^collab-survey]: *Multi-Agent Collaboration Mechanisms: A Survey of LLMs*, arXiv:2501.06322, 2025. <https://arxiv.org/abs/2501.06322>

[^metagpt]: Sirui Hong et al., *MetaGPT: Meta Programming for a Multi-Agent Collaborative Framework*, ICLR 2024. <https://arxiv.org/abs/2308.00352>

[^aop]: *Agent-Oriented Planning in Multi-Agent Systems*, ICLR 2025, arXiv:2410.02189. <https://arxiv.org/abs/2410.02189>

[^okr-agent]: *Agents meet OKR: An Object and Key Results Driven Agent System with Hierarchical Self-Collaboration and Self-Evaluation*, arXiv:2311.16542. <https://arxiv.org/abs/2311.16542>

[^ecl]: Chen Qian et al., *Experiential Co-Learning of Software-Developing Agents*, ACL 2024. <https://aclanthology.org/2024.acl-long.305/>（Quality：ChatDev ≈ 0.4267 → Co-Learning ≈ 0.7304）

[^evochamber]: *EvoChamber: Test-Time Co-evolution of Multi-Agent System at Individual, Team, and Population Scales*, arXiv:2605.11136. <https://arxiv.org/abs/2605.11136>；实现 <https://github.com/Mercury7353/EvoChamber>

[^iclr-memory]: ICLR 2026 Workshop: Memory for Agentic Systems（议程与论文集以会议官网为准）。

[^agno-memory]: Agno Organizational Memory（产品 / 框架文档，作工程对照）。
