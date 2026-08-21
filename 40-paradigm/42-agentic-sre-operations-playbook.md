# SRE / DevOps 的智能体化：SLM、LLM、Agent 如何分工，以及 MTTR 的可验证压缩

> 监控加「AI」标签并不自动缩短故障恢复时间。行业真正发生的变化，是把 **SRE 的事故工作流**与 **DevOps 的变更工作流**，改造成可由 **小模型（SLM，Small Language Model）、大模型（LLM，Large Language Model）、智能体（Agent）** 分工执行的人机系统。可核对的价值，集中在一件事上：**压缩平均恢复时间（MTTR，Mean Time to Repair / Restore）中那些可并行、可检索、可证据化的时间段**——而不是承诺无人值守自愈。
>
> 本文先交代行业从何而来（监控 → AIOps / Event Intelligence → AI SRE），再锚定 SRE / DORA 等可核验约束，最后给出分工模型、架构与落地顺序。关键数字均为公开案例或研究观察，**须用本组织基线复核**，不可直接当 KPI 保证值。

先给一个直接答案：

> **经典 AIOps 主要缩短「发现」；LLM + RAG + Agent 主要缩短「理解、取证与协同」。生产上不要用一个大模型包办全链路——窄任务交给 SLM，跨源解释交给 LLM，工具编排与权限门闸交给 Agent；写操作默认人在环上。**

---

## 目录

1. [开篇：MTTR 卡住了哪里](#1-开篇mttr-卡住了哪里)
2. [行业基础：从监控到 AIOps，再到 AI SRE](#2-行业基础从监控到-aiops再到-ai-sre)
3. [理论锚点：SRE、DORA 与度量诚实](#3-理论锚点sredora-与度量诚实)
4. [为什么传统 AIOps 往往压不动 MTTR](#4-为什么传统-aiops-往往压不动-mttr)
5. [三类载体：SLM、LLM、Agent 如何分工](#5-三类载体slmllmagent-如何分工)
6. [与 SRE / DevOps 的结合面](#6-与-sre--devops-的结合面)
7. [MTTR 拆解：AI 到底缩短哪几段](#7-mttr-拆解ai-到底缩短哪几段)
8. [参考架构：感知 → 认知 → 受控行动](#8-参考架构感知--认知--受控行动)
9. [端到端场景](#9-端到端场景)
10. [边界、评测与落地顺序](#10-边界评测与落地顺序)
11. [结语](#11-结语)
12. [参考文献](#12-参考文献)

---

## 1. 开篇：MTTR 卡住了哪里

值班工程师面对 P1/P2 时，真正耗时的往往不是「点一下回滚」，而是：

1. 在指标、日志、链路、发布记录、工单之间跳转，拼出「刚发生了什么」
2. 判断是不是假警、该不该升级、该喊哪一队
3. 提出假设、验证、推翻、再提假设
4. 在群里反复复述现状，等人对齐
5. 恢复后再从聊天记录拼复盘

Google SRE 手册早已指出：可靠性与 MTTF / MTTR 相关，而**人会引入延迟**；有经过演练的 playbook，相对「临场发挥」，MTTR 可改善约 **3 倍**。[^sre-intro] 这说明：缩短恢复时间，关键杠杆从来不只是「更多监控」，而是**更快进入正确上下文、更快形成可执行决策**。

生成式 AI 与智能体浪潮，切入的正是这段**认知与协同成本**。incident.io 等实践总结把 AI SRE 定义为：用 LLM + RAG（日志、拓扑、Runbook、历史事故）自动承担调查、文档与协调中的大量机械工作，而不是替代人对生产变更的最终问责。[^incident-ai-sre]

一句话对比四代能力：

| 代际 | 典型输出 | 对 MTTR 的作用点 |
| ---- | -------- | ---------------- |
| 规则监控 | 阈值告警 | 发现，但噪声大 |
| 经典 AIOps / ML | 异常分、事件簇 | 降噪，主要缩短「发现」 |
| LLM + RAG | 带证据的解释与时间线 | 缩短「理解」与「协同」 |
| Agent（多步工具调用） | 调查闭环、修复草案、受控执行 | 缩短「取证—方案—执行准备」 |

---

## 2. 行业基础：从监控到 AIOps，再到 AI SRE

### 2.1 名词从哪来，市场怎么变

**AIOps** 一词由 Gartner 在 2010 年代中后期推广：把大数据与机器学习用于 IT 运维，强调跨域摄取遥测、关联分析与加速处置。[^gartner-eis] 此后几乎所有监控、可观测性、ITSM 厂商都开始贴「AIOps」标签，买家期望与实际交付出现系统性落差。

到 **2024 年前后**，Gartner 将该市场从「AIOps Platforms」**重命名为 Event Intelligence Solutions（EIS，事件智能解决方案）**，明确指出：AIOps 被过度营销、定义不清，导致基础设施与运维负责人失望。EIS 的核心能力被收敛为：跨域事件摄取、拓扑组装、关联与富化、模式识别、加速修复——目标是把事件流变成可行动洞察，降低 toil、提升可用性。[^gartner-eis]

这一定义很重要：**行业基础能力首先是事件智能与关联，不是「自治运维」营销话术。**

### 2.2 经典 AIOps 解决了什么，没解决什么

经典 AIOps / EIS 擅长：

- 告警风暴中的去重、聚类、关联
- 异常检测与部分根因线索排序
- 把多工具信号收敛成更少的「事件」

Thoughtworks 基于 2025 年逾 16 家客户、20 个 PoC（其中 11 个进生产）的复盘指出：AIOps 已进入**工程阶段**，但尚未工业化；**最高价值用例集中在知识与 RCA，而非自治动作**——重复事件检测、运维知识检索、根因辅助。跨部署观察中，L1/L2 工单量约降 **35%–40%**，RCA 周期可从小时压到分钟；**自动修复 / 自愈仍受风险、治理与问责约束，难以走出受控环境**。[^tw-aiops-2025]

换言之：上一代 AI 运维把「信号相关」做好了，但值班工程师仍大量时间花在「**为什么、证据在哪、下一步查什么**」。

### 2.3 这一轮：从 Event Intelligence 到 AI SRE / Agentic Ops

新一轮变化不是再买一个关联引擎，而是：

| 能力跃迁 | 含义 | 代表证据 |
| -------- | ---- | -------- |
| **解释取代打分** | LLM + RAG 产出带引用的假设与时间线 | Thoughtworks：相关 ≠ 解释；高价值在知识与 RCA[^tw-aiops-2025] |
| **工具调用取代只读聊天** | Agent 并行拉遥测、变更、拓扑、Runbook | OpenDerisk 等工业框架强调 MCP / 工具编排与上下文工程[^openderisk] |
| **多角色协作取代万能机器人** | Intake / Triage / Investigate / Critic / Scribe | Microsoft Triangle：多 Agent 分诊；ASE 2025 论文所报**特定生产环境**中，triage 准确率最高约 **97%**，Time-to-Engage 最高降约 **91%**——不可外推为通用 SLA[^triangle] |
| **小而专模型进入诊断核心** | 不必处处堆最大闭源模型 | OpsAgent：约 **14B** 推理核 + 多 Agent，在 OPENRCA 基准与联想生产环境验证可部署性[^opsagent] |
| **混合确定性—概率性系统** | LLM 出结构化意图，策略引擎执行 | Thoughtworks Learning 4：LLM → JSON → 确定性引擎[^tw-aiops-2025] |

**AI SRE** 可理解为：把上述能力嵌入事故管理生命周期（检测、分诊、调查、缓解、沟通、复盘），并与 DevOps 变更流打通——因为绝大多数严重事故的第一问仍是「**刚发了什么**」。[^incident-ai-sre]

---

## 3. 理论锚点：SRE、DORA 与度量诚实

在谈「AI 降 MTTR」之前，先固定几条行业约束，避免方案漂成口号。

| 锚点 | 要点 | 对 AI SRE 的含义 |
| ---- | ---- | ---------------- |
| **SRE：人引入延迟，playbook 有效** | 应急响应有效性看能否快速恢复健康；playbook 相对临场发挥可显著改善 MTTR[^sre-intro] | Agent 的第一交付物应是**可执行上下文与步骤草案**，而不只是聊天摘要 |
| **SRE：先缓解，再完美根因** | 客户关心错误是否停止，不关心你是否完全理解根因[^sre-ir] | 调查 Agent 应优先支持缓解路径（回滚、限流、扩容上限），而非无限深挖 |
| **MTTx 均值需谨慎** | Google 研究指出 MTTR/MTTM 等均值在低样本量、高方差下，常不适合做决策与趋势 KPI[^sre-metrics] | 应用**分段耗时分布**与个案对照评测 AI，而不是只报「MTTR 降 X%」 |
| **DORA：AI 是放大器** | 2025 报告：AI 放大高绩效组织的优势，也放大脆弱组织的缺陷；高质量内部平台显著影响 AI 收益；AI 与吞吐正相关，与交付稳定性仍可能负相关[^dora-2025] | 可观测性乱、变更无规范、服务目录缺失时，Agent 会更快放大混乱 |
| **错误预算与 SLO** | SRE 语言是 SLO / 错误预算，不是模型准确率 | 自动化误操作应消耗错误预算；预算耗尽则收回自治权 |

四条合在一起，构成后文的底线：

> **AI 不替代 SRE 工程；它放大你已有的遥测、变更规范、拓扑与治理。底座差，再大的模型也只是更快地幻觉。**

---

## 4. 为什么传统 AIOps 往往压不动 MTTR

### 4.1 先把「从坏到好」拆开

实务上常把响应拆成若干段（命名因组织略有差异）：

```text
故障发生
  → MTTD   检测到异常 / 告警成立
  → MTTI   识别为真实事故并开始响应（含降噪、定级、找人）
  → MTTK   定位到可行动根因 / 关键贡献因素（Know）
  → MTTF   形成修复方案并具备执行条件（Fix plan）
  → 执行   执行修复并确认恢复（狭义 Repair/Resolve）
  → 复盘   时间线与改进项沉淀（常不计入 MTTR，但吃 SRE 带宽）
```

经典 AIOps / EIS 主要优化 **MTTD + 部分 MTTI（降噪）**。若组织真实瓶颈在 **MTTK（定位）与协同**，只上异常检测，整体恢复曲线往往几乎不动——这就是「AIOps 上了、MTTR 没感觉」的常见原因。

### 4.2 单 Agent 套壳为什么不够

把「一个 LLM + 一个长 Prompt」接到告警上，演示好看、生产易失败：

- 告警风暴瞬间灌爆上下文窗口
- 同一模型同时做分诊、调查、协调、写文档，角色互相污染
- 没有工具白名单与权限时，要么不能做事，要么做事不可审计
- 没有拓扑与变更源，只能总结日志措辞，无法做跨源因果候选
- Thoughtworks 观察到：MCP 等协议方向正确，但上下文链膨胀、路径不透明、成本波动仍常见，尚不足以支撑任务关键系统的工业化[^tw-aiops-2025]

前沿工程路线是 **多角色 Agent + 编排 + Critic + 人在环**，而不是万能聊天机器人。[^triangle][^opsagent][^openderisk]

---

## 5. 三类载体：SLM、LLM、Agent 如何分工

这是本文最关键的技术判断：**不要用一个前沿大模型包办 SRE/DevOps 全链路。**

### 5.1 定义与适用面

| 载体 | 典型形态 | 擅长 | 不擅长 | 在运维中的位置 |
| ---- | -------- | ---- | ------ | -------------- |
| **SLM（小模型）** | 约 ≤10B–15B，可私有化；或专用分类器 | 低延迟、低成本、窄任务、可微调、数据不出域 | 开放式跨源长链推理 | 告警分类、严重级初判、日志字段抽取、意图路由、重复事件初筛 |
| **LLM（大模型）** | 通用大模型 + RAG | 跨源综合、假设生成、方案与文档草稿 | 高 QPS 全量在线、强实时裸推全量遥测 | RCA 候选、变更影响说明、复盘初稿、复杂工具规划 |
| **Agent** | 带状态机/图编排的多步系统 | 调工具、保流程、做人机门闸、循环验证 | 替代人类对新颖故障的第一性原理判断 | 把 SLM/LLM 与可观测性、CMDB、CI/CD、工单粘成工作流 |

OpsAgent 一类工作说明：「**小而专 + 可编排**」正在成为可发表、可落地的方向——以约 14B 模型为推理核，配合多 Agent 协作与可审计推理链，而不是只能堆最大闭源模型。[^opsagent]

### 5.2 推荐的模型路由（生产默认）

```text
告警/事件进入
    │
    ├─ SLM：分类 / 降噪标签 / 是否像已知模式 / 是否值得叫醒人
    │         （毫秒～亚秒，可本地，成本可控）
    │
    ├─ 需要跨源解释？ ──否──→ 结束或自动关单/合并
    │         │是
    │         ▼
    ├─ Agent 启动调查图：拉指标、日志、链路、变更、拓扑、Runbook（RAG）
    │         │
    │         ▼
    ├─ LLM：生成「假设 + 证据引用 + 置信度 + 下一步验证」
    │         │
    │         ▼
    ├─ Critic / 规则引擎：检查引用是否存在、动作是否越权
    │         │
    │         ▼
    └─ 人确认后：执行修复 / 回滚 / 扩容（确定性 API，而非模型直接 SSH）
```

**原则：**

1. **能用 SLM 的不要用 LLM**（成本、延迟、隐私、输出可预测性）。
2. **LLM 只在需要解释与综合时唤醒**。
3. **Agent 不替代权限系统**——只能调用白名单工具；写操作走后端 RBAC + 审批。
4. **确定性组件兜底概率性组件**：策略引擎、schema 校验、熔断与错误预算。[^tw-aiops-2025]

### 5.3 DevOps 侧同一分工

| DevOps 环节 | SLM | LLM | Agent |
| ----------- | --- | --- | ----- |
| CI 失败摘要 | 失败类型分类、flake 初筛 | 解释失败与可疑 diff | 拉日志、重跑建议、开草稿 PR |
| 代码评审 | 规范/安全规则命中 | 设计风险说明、测试缺口 | 调用 linters、单测、依赖扫描 |
| 变更风险评估 | 变更类型打标 | 影响面叙述、类似失败历史 | 关联服务拓扑、近期事故、SLO |
| 发布说明 | 字段抽取 | 面向值班的风险摘要 | 写入发布系统 / 事件通道 |

SRE 与 DevOps 的交汇点是 **变更**。把部署、配置、特性开关、IaC 变更做成 Agent 可查询的一等数据源，往往比再调大一个模型更能打 MTTR。DORA 亦提醒：加速变更若下游验证与平台薄弱，稳定性会承压——运维侧 Agent 与交付侧护栏必须一起设计。[^dora-2025]

---

## 6. 与 SRE / DevOps 的结合面

### 6.1 SRE：事故管理闭环

| SRE 活动 | 传统做法 | 与模型/Agent 结合后 | 主要压缩段 |
| -------- | -------- | ------------------- | ---------- |
| 监测与告警 | 阈值/SLI 告警 | SLM/ML 降噪 + 事件聚类 | MTTD / MTTI |
| 分诊与升级 | 人工读告警、查值班表 | SLM 定级 + 服务目录路由；多 Agent 协商归属 | MTTI / TTE |
| 调查 | 人工跨工具查询 | Agent 并行拉遥测/变更；LLM 出假设+引用 | MTTK |
| 缓解与修复 | 人工执行 Runbook | Agent 生成步骤；人批准后调 K8s/云 API | MTTF / 执行段 |
| 沟通 | 群里口头同步 | Agent 持续写现状摘要 | 协同等待 |
| 复盘 | 事后回忆拼时间线 | 事故中自动记时间线；LLM 出复盘草稿 | SRE 带宽（间接稳定 MTTR） |

Microsoft Triangle 的生产结果，说明「找对人」本身就能大幅压缩 Time-to-Engage；这与「找对根因」是不同但同样关键的杠杆。[^triangle]

### 6.2 DevOps：变更与交付闭环

| DevOps 活动 | 结合方式 | 对可靠性的作用 |
| ----------- | -------- | -------------- |
| 提交与构建 | SLM/LLM 辅助失败诊断 | 减少带病变更进入生产 |
| 渐进交付 | Agent 读金丝雀指标，LLM 解释是否像回归 | 缩短「发现坏变更 → 回滚决策」 |
| 变更日历 / 冻结 | 规则 + SLM 分类 | 降低高峰事故率 |
| 配置与 IaC | LLM 解释 drift；Agent 开修复 PR | 缩短配置类 MTTK |
| 值班交接 | 自动生成当前风险与未关闭假设 | 减少交接丢上下文 |

### 6.3 人机职责的硬边界

| 决策类型 | 建议默认 | 理由 |
| -------- | -------- | ---- |
| 只读查询、摘要、假设 | Agent 自动 | 错了可被证据打脸，伤害有限 |
| 工单路由、重复事件合并 | SLM + 规则，抽检 | 高频，需稳定 |
| 重启单副本、扩容有上限 | 可「人在环上」或窄白名单自动 | 需错误预算与熔断 |
| 回滚生产、改流量、删数据、任意 Shell | 必须人批准 | 问责与爆破半径 |

Thoughtworks 的判断与此一致：在成熟 AI 治理到位之前，AIOps 的角色仍是**认知增强，而非自治代理**。[^tw-aiops-2025]

---

## 7. MTTR 拆解：AI 到底缩短哪几段

### 7.1 机制，而不是口号

| 段落 | AI 如何起作用 | 主要载体 | 公开观察（条件化） |
| ---- | ------------- | -------- | ------------------ |
| 检测与降噪 | 少被假警淹没，真警聚成「一个事故」 | ML + SLM | EIS / AIOps 主场；工单量可显著下降[^tw-aiops-2025] |
| 定级与找人 | 按服务目录/历史模式路由 | SLM + 多 Agent | Triangle：TTE 最高约降 91%[^triangle] |
| 上下文组装 | 一次拉齐指标、日志、链路、变更、拓扑 | Agent 工具调用 | 砍掉大量面板跳转[^openderisk] |
| 根因候选 | 跨源推理 + 引用；对比历史事故 | LLM + RAG | RCA 可从小时到分钟级（视复杂度）[^tw-aiops-2025] |
| 方案准备 | 生成 Runbook 步骤、回滚命令草案 | LLM + Agent | 缩短 MTTF；执行仍由人/API |
| 协同与记录 | 实时时间线、状态广播、复盘草稿 | LLM + Scribe | 减少「写清现状」的并行成本[^incident-ai-sre] |

### 7.2 一个可计算的心智模型

设一次事故响应时间近似为：

\[
T \approx T_{noise} + T_{context} + T_{hypothesize} + T_{verify} + T_{approve} + T_{execute} + T_{confirm}
\]

- **SLM / 经典 ML** 主要打 \(T_{noise}\)
- **Agent 工具层** 主要打 \(T_{context}\)
- **LLM + RAG** 主要打 \(T_{hypothesize}\)（有时辅助「下一步查什么」）
- **治理与人** 主导 \(T_{approve}\)；**自动化 Runbook** 压缩 \(T_{execute}\)
- 若瓶颈在 \(T_{approve}+T_{execute}\)，却只上聊天式 RCA，改善会令人失望

> 先画自己的时间饼图，再选模型。多数团队高估「自动修复」，低估「自动取证」。对 MTTR 而言，**先把 \(T_{context}+T_{hypothesize}\) 打穿**，通常是 ROI 最高的一步。

同时记住 Google 对 MTTx 均值的警告：应用分段分布、前后对照与引用有效性等过程指标，而不是只盯一个平均值。[^sre-metrics]

### 7.3 与错误预算、SLO 的关系

引入 Agent 后建议增加：

- **调查质量 SLO：** 引用可点击率、假设被采纳率、严重误导率
- **自动化错误预算：** 误操作或错误缓解消耗预算，触发收回自治权
- **成本预算：** 每次事故的 LLM token / SLM 调用上限，防告警风暴打爆账单

---

## 8. 参考架构：感知 → 认知 → 受控行动

综合 Gartner EIS 分层、多 Agent SRE 实践与混合确定性系统观察：

```text
┌─────────────────────────────────────────────────────────────┐
│ 行动层 Action：白名单 API / Runbook / 审批 / 审计 / 回滚      │
├─────────────────────────────────────────────────────────────┤
│ 认知层 Cognition：SLM 路由 · LLM 推理 · 多 Agent 图 · Critic │
├─────────────────────────────────────────────────────────────┤
│ 感知层 Perception：指标·日志·链路·事件·变更·拓扑·知识库      │
└─────────────────────────────────────────────────────────────┘
         ▲ 可观测性与 CMDB/服务目录          ▲ CI/CD 与发布系统
```

### 8.1 感知层：没有它，再强的 LLM 也会幻觉

Thoughtworks 明确写道：AIOps 性能的上限往往不在模型智商，而在**上下文是否可得**；依赖、团队拓扑、变更史、本体、事故史与 Runbook 若仍散落且非 AI-ready，系统就只是碎片数据上的对话壳。[^tw-aiops-2025]

必备输入（缺一则 MTTK 难降）：

1. **遥测：** metrics / logs / traces（统一时间与服务 ID）
2. **变更：** 部署、配置、特性开关、IaC 变更时间线
3. **拓扑：** 服务依赖、归属团队、爆破半径
4. **知识：** Runbook、历史事故、架构决策（RAG 语料）

### 8.2 认知层：多 Agent 角色示例

| 角色 | 模型偏好 | 职责 |
| ---- | -------- | ---- |
| **Intake** | SLM / 规则 | 去重、合并、丰富标签 |
| **Triage** | SLM + 目录 / 多 Agent 协商 | 严重级、是否呼叫、呼叫谁 |
| **Investigator** | Agent + LLM | 并行查询，产出假设与证据 |
| **Planner** | LLM | 缓解步骤、前置条件、回滚触发 |
| **Critic** | 规则 + 较弱/另一模型 | 打假引用、拦越权、拦危险命令 |
| **Scribe** | LLM | 时间线与复盘草稿 |

关键约束：调查 Agent 与 Critic **不要用同一上下文自我打分**；高风险计划必须经 Critic 与人。

### 8.3 行动层：确定性执行

LLM 可以写「建议 `kubectl rollout undo ...`」，但执行应是：

1. 生成结构化动作意图（JSON）
2. Preflight：参数、权限、爆破半径、变更窗口
3. 人确认（或窄场景自动）
4. 后端调用官方 API / 已备案 Runbook
5. 写审计：谁批准、模型版本、证据包 ID、结果

这是混合系统：**概率性负责理解与起草，确定性负责发生。**[^tw-aiops-2025]

---

## 9. 端到端场景

### 9.1 结账延迟升高（事故侧）

**感知：** `checkout` p99 延迟告警；错误率上升；下游 `payment-gateway` CPU 高。

**SLM Intake/Triage：** 判定为疑似 P2；关联近 5 分钟同源告警 → 合成一个事故；按服务目录呼叫 checkout on-call。

**Agent Investigator（并行）：**

- 查 14:28–14:35 的部署：`payment-gateway@2.1.4`
- 查 diff：连接池超时参数被删除
- 查日志：`pool exhausted`
- RAG：命中三月类似事故

**LLM 输出结构：** 假设 + 证据链接 + 建议回滚 + 验证步骤。

**人：** 核对证据链后批准回滚。**行动层**执行回滚 API → 确认 SLO 恢复。**Scribe** 自动时间线 → 复盘草稿大半已成。

压缩的是 \(T_{context}+T_{hypothesize}+\) 协同复述；**回滚按钮仍建议由人按**（除非该故障类已进入有界自治白名单）。

### 9.2 CI 变红导致发布阻塞（DevOps → 预防）

**SLM** 分失败类型 → **Agent** 拉日志与 diff → **LLM** 解释并生成补丁草稿 → **人** 审 PR。

价值是**减少带病变更**，降低未来事故频率——这是 DevOps 对可靠性的贡献，常被纯 AIOps 方案忽略。

### 9.3 有界自动修复（最后才做）

仅当同时满足：已知故障类、证据模板稳定、动作可回滚、错误预算未耗尽、审计完备——才允许 Agent 自动执行「重启某部署 / 扩容至上限」。这是成熟度后端能力，不是起点。

---

## 10. 边界、评测与落地顺序

### 10.1 能力边界（必须写进方案）

| 边界 | 含义 |
| ---- | ---- |
| 幻觉 | 无引用的根因不可执行 |
| 新颖故障 | 无历史与拓扑时，人必须主导 |
| 协议与长链 Agent | 上下文膨胀、路径不透明、成本波动仍常见[^tw-aiops-2025] |
| 问责 | 事故责任在人与组织，不在模型供应商话术 |
| 度量 | 慎用单一 MTTR 均值做采购决策[^sre-metrics] |

### 10.2 最小评测集（比再买一个模型优先）

1. **引用有效性：** 给出的日志/PR/Dashboard 是否真实存在
2. **假设采纳率：** on-call 是否采用或显式否定
3. **有害建议率：** 会导致扩大故障的建议占比
4. **分段耗时：** \(T_{context}\)、\(T_{hypothesize}\) 前后对照（建议 ≥90 天基线）
5. **分诊质量：** 路由准确率、Time-to-Engage
6. **成本：** 每次事故 token / GPU 费用

### 10.3 建议落地顺序

| 阶段 | 做什么 | 成功信号 |
| ---- | ------ | -------- |
| **1** | 统一服务 ID、变更事件、核心遥测；建 Runbook/事故 RAG | Agent 能一次取全上下文 |
| **2** | SLM/ML 降噪 + 分诊路由 | 假警下降；找对人变快 |
| **3** | 只读调查 Agent + LLM 证据包（人仍执行修复） | MTTK / \(T_{context}\) 明显下降 |
| **4** | 修复草案 + 审批执行；事故中时间线 | MTTF + 复盘成本下降 |
| **5** | 少数故障类有界自治 + 错误预算 | 自动化覆盖上升且事故不恶化 |

**不要**从第 5 步或「全能运维机器人」开始。PoC 进不了生产，常见死因是治理缺失、知识未 AI-ready、无人持续调参——而不是模型不够大。[^tw-aiops-2025]

---

## 11. 结语

SRE 与 DevOps 拥抱 Agent，本质上是一次**工作流重构**：

- **小模型**吃高频、窄域、要速度与隐私的任务；
- **大模型**吃跨源解释、方案与文档生成；
- **Agent**把两者接到真实工具、权限与审计上。

对 MTTR 的好处，不应写成模糊的「更智能」，而应写成可验证的机制：**少噪声、快组上下文、早出带证据的假设、缩短协同与复盘 toil；在治理允许时再压缩执行段。**

行业史已经写明：Event Intelligence 解决「看见与关联」；生成式与智能体浪潮解决「解释与行动准备」。若只能做一件事——

> **先让每次严重告警自动附带一份「变更 + 遥测 + 历史事故」证据包，再谈要不要换更大的模型。**

---

## 12. 参考文献

[^sre-intro]: Betsy Beyer et al. (eds.), [*Site Reliability Engineering*](https://sre.google/sre-book/introduction/)（Google SRE Book；可靠性与 MTTF/MTTR；playbook 相对临场发挥约 3× MTTR 改善；人引入延迟）。

[^sre-ir]: Google SRE Workbook, [*Incident Response*](https://sre.google/workbook/incident-response/)（先缓解再完美根因；协调与沟通对缩短解决时间的高杠杆）。

[^sre-metrics]: Štěpán Davidovič, [*Incident Metrics in SRE*](https://sre.google/resources/practices-and-processes/incident-metrics-in-sre/)（Google；论证 MTTR/MTTM 等均值在生产事故场景下常不适合作决策与趋势 KPI）。

[^gartner-eis]: Gartner, *Market Guide for Event Intelligence Solutions*（原 AIOps Platforms 市场；EIS 定义：跨域事件摄取、拓扑、关联富化、模式识别、加速修复）。二级综述见 [Selector 对 2025 Market Guide 的解读](https://www.selector.ai/blog/key-takeaways-from-the-2025-gartner-market-guide-for-event-intelligence-solutions/) 与 [Gartner Peer Insights：Event Intelligence / AIOps](https://www.gartner.com/reviews/market/event-intelligence-solutions)。

[^tw-aiops-2025]: Thoughtworks, [*AIOps: What we learned in 2025*](https://www.thoughtworks.com/insights/blog/generative-ai/aiops-what-we-learned-in-2025)（16+ 客户 / 20 PoC / 11 生产；高价值在知识与 RCA；L1/L2 约降 35–40%；自治受限；混合确定性—概率性系统；上下文工程为上限；MCP 尚未工业化）。

[^dora-2025]: DORA / Google Cloud, [*2025 DORA Report: State of AI-assisted Software Development*](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report)（约 5,000 名从业者；AI 为放大器；吞吐与稳定性张力；高质量内部平台放大 AI 收益）。研究报告页：[research.google](https://research.google/pubs/dora-2025-state-of-ai-assisted-software-development-report/)。

[^incident-ai-sre]: incident.io, [*What is AI SRE?*](https://incident.io/blog/what-is-ai-sre-complete-guide-2026)（LLM + RAG 接地基础设施数据；优先调查/文档/协调；自治修复仍有限；人机问责）。

[^triangle]: Zhaoyang Yu et al., [*Triangle: Empowering Incident Triage with Multi-Agent*](https://www.microsoft.com/en-us/research/publication/triangle-empowering-incident-triage-with-multi-agents/), ASE 2025（会议论文，数字来自 Microsoft 所述特定生产环境，不是跨行业基准）；工程说明见 [Azure Blog](https://azure.microsoft.com/en-us/blog/optimizing-incident-management-with-aiops-using-the-triangle-system/)（triage 准确率最高约 97%，Time-to-Engage 最高约降 91%）。

[^opsagent]: [*From Observability Data to Diagnosis: An Evolving Multi-agent System for Incident Management in Cloud Systems*](https://arxiv.org/abs/2510.24145)（OpsAgent；约 14B 推理核 + 多 Agent；OPENRCA；联想生产部署报告）。

[^openderisk]: [*OpenDerisk: An Industrial Framework for AI-Driven SRE*](https://arxiv.org/html/2510.13561v2)（工业级 AI SRE 框架；MCP / 上下文工程；RAG 知识引擎；人机协同到有界自治的演进路线）。

[^peng-2023]: Sida Peng et al., [*The Impact of AI on Developer Productivity: Evidence from GitHub Copilot*](https://arxiv.org/abs/2302.06590), arXiv:2302.06590, 2023（研发侧个人吞吐参考；外推到组织与运维需谨慎）。

[^cn-gai]: 国家互联网信息办公室等, [《生成式人工智能服务管理暂行办法》](http://www.cac.gov.cn/2023-07/13/c_1690898327029107.htm)（生产数据与生成式服务合规底线）。
