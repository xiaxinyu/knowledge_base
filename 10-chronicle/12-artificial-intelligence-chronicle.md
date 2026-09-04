# 人工智能发展编年史：承诺、能力与技术本质

> 七十余年不是直线起飞，而是锯齿线。寒冬多半是**承诺跑得比能力快**；爆发多半是表示、数据与算力重新对齐。
>
> 大模型不是无源突变。它是深度学习在规模、架构与产品形态上的延续。产业含义见 [50](../50-strategy/50-ai-industry-disruption-strategy.md)。

## 摘要

「人工智能」（artificial intelligence）一词系统见于 1955 年达特茅斯夏季研究提案，因 1956 年会议成为学科之名。思想前史可溯至 1943 年 McCulloch–Pitts 神经元模型；研究起点常记 1950 年图灵测试。[1][2][3]

技术上宜分清六种机制：符号搜索、专家系统、统计机器学习、深度学习、大模型、智能体。史上可按五阶段把握：符号主义 → 专家系统（两次寒冬穿插其间）→ 机器学习 → 深度学习 → 大模型（智能体是其延伸，不是另起的史段）。产业叙事另可粗分为计算智能、感知智能、生成智能。[4]

贯穿始终的可观察主线有三条：**表示**（世界被写成什么）、**搜索或拟合**（运行时在算什么）、**经费与产品**（谁付钱、什么破圈）。阶段跃迁，几乎都是三条同时改写。

**三个直接答案**

1. **词从哪来？** 1955 年提案写出 *artificial intelligence*；1956 年达特茅斯会议立科。  
2. **经历哪些阶段？** 上列五阶段；对照 §2 的六种机制与 §10 的对齐表。  
3. **大模型属哪一段？** 第五阶段。本质是大规模条件生成；对话产品使之破圈，工具循环使之能办事。

**关键词：** 符号主义；专家系统；机器学习；深度学习；Transformer；大模型；智能体；AI 寒冬

---

## 目录

- [摘要](#摘要)
1. [如何阅读：两把尺](#1-如何阅读两把尺)
2. [六种机制：本质、可观察、失效](#2-六种机制本质可观察失效)
3. [起源：神经元模型、图灵测试与命名](#3-起源神经元模型图灵测试与命名)
4. [符号主义与第一次寒冬](#4-符号主义与第一次寒冬)
5. [专家系统与第二次寒冬](#5-专家系统与第二次寒冬)
6. [统计学习转向、深蓝与深层网络火种](#6-统计学习转向深蓝与深层网络火种)
7. [深度学习上台：从 AlexNet 到 AlphaGo](#7-深度学习上台从-alexnet-到-alphago)
8. [大模型：Transformer、预训练到破圈](#8-大模型transformer预训练到破圈)
9. [智能体：从会聊到办事](#9-智能体从会聊到办事)
10. [回收：阶段、机制与可带走的判断](#10-回收阶段机制与可带走的判断)
11. [节点年表](#11-节点年表)
12. [参考文献](#12-参考文献)

---

## 1. 如何阅读：两把尺

| 尺 | 所问 | 落点 |
|----|------|------|
| **技术尺** | 知识从哪来？运行时在算什么？输出如何检验？ | §2 词典；§4–§9 各段主导机制 |
| **制度尺** | 谁付钱、谁砍经费、何种产品破圈？ | ALPAC、莱特希尔报告、DARPA；ChatGPT [5][6][12] |

结构性张力始终在：**任务定义宏大，可支付能力受制于表示、搜索、算力与数据。** 承诺与能力之间的鸿沟，是寒冬的同构原因；算法、数据、算力对齐，是爆发的同构条件。

```mermaid
flowchart LR
  A["§2 机制词典"] --> B["§3 起源"]
  B --> C["§4–§5 符号 / 专家"]
  C --> D["§6 统计学习"]
  D --> E["§7 深度学习"]
  E --> F["§8 大模型"]
  F --> G["§9 智能体"]
```

§2 的六种机制与 §4–§9 一一对应。五阶段是史切法，六机制是物理解释。智能体记为第五阶段之延伸，不是第六史段。

---

## 2. 六种机制：本质、可观察、失效

给任何系统定性，先问四句：

1. **知识从哪来？** 人手写，还是从数据估计？  
2. **运行时算什么？** 搜索与规则匹配，还是数值前向（可加采样）？  
3. **输出是什么？** 证明链、标签或分数，还是开放生成物、动作轨迹？  
4. **错误如何暴露？** 规则冲突、分布外掉点、幻觉，还是工具调用失败？

| 机制 | 本质 | 运行时可见 | 可观测量 | 典型失效 |
|------|------|------------|----------|----------|
| **符号主义** | 符号状态 + 逻辑 / 搜索 | 规则、搜索树、证明路径 | 展开节点、证明长度、轨迹可否复核 | 组合爆炸；世界一模糊，规则不够 [6] |
| **专家系统** | 窄域知识库 + 推理机 | `if–then`、咨询式建议 | 规则条数、工程师人时、域内准确率 | 获取贵、难维护、域外即死 |
| **统计机器学习** | 从样本估计映射 \(f:x\mapsto y\) | 特征 → 分数 / 标签 | 持出误差、AUC、样本量 | 特征靠人；分布一移就掉点 |
| **深度学习** | 表征也从数据学 | 张量进、张量出 | 榜单误差、FLOPs、GPU 时 | 需大数据与算力；默认偏识别 [38] |
| **大模型** | 大规模序列预测 / 条件生成 | 提示 → token 或跨模态采样 | 基准、上下文、推理成本、少样本 | 幻觉；事实与责任难溯源 [11] |
| **智能体** | 模型 + 工具 + 多步循环 | 计划、调用、观察、再计划 | 任务成功率、步数、介入率、轨迹日志 | 目标含糊、权限与评测不足 [4] |

史实锚点在对应编年节展开。深蓝是封闭规则下的搜索旁支，不要与统计学习或深度学习混为一谈。

---

## 3. 起源：神经元模型、图灵测试与命名

### 3.1 形式先声（1943）

1943 年，Warren S. McCulloch 与 Walter Pitts 用逻辑演算描述神经元网络，把「神经计算」写成可分析的形式系统。[3] 这是连接主义的早期锚点：单元与联结可以计算。它还不是后来靠梯度与大规模数据训练出来的深层感知系统。

### 3.2 可操作的判据（1950）

1950 年，Alan Turing 在 *Mind* 发表 *Computing Machinery and Intelligence*，把「机器能思考吗」改写成模仿游戏：若裁判无法通过对话稳定区分人与机器，则可认为机器表现出相应水平的智能。[2] 测试在哲学上争议不断——它测的是「像人」，还是「真理解」——但作为研究起点的公共坐标已经稳固。空泛的本体论问题，部分被改写成可观察的行为判据。

### 3.3 命名与立科（1955–1956）

1955 年 8 月 31 日，John McCarthy、Marvin Minsky、Nathaniel Rochester、Claude Shannon 提交《达特茅斯人工智能夏季研究项目提案》，拟于 1956 年夏在达特茅斯学院进行约两个月、约十人的研究，并陈述核心猜想：

> learning or any other feature of intelligence can in principle be so precisely described that a machine can be made to simulate it.[1]

**artificial intelligence** 由此成为领域自称。提案列出的议程——使用语言、形成抽象与概念、解决当时仅人类能解决的问题、自我改进——七十年后仍像一份未完成的产品路线图。[1]

1956 年夏会议召开，日程松散，人员进进出出。Allen Newell、Herbert Simon 与 Cliff Shaw 的 **Logic Theorist** 被带到会上：该程序证明了 Whitehead 与 Russell《数学原理》第二章前 52 条定理中的 38 条，并为定理 2.85 找到比原书更短的证明。[7] 符号主义的「黄金十年」由此有了可演示的开场。

命名划出独立于控制论圈子的领地，也埋下定义困难：智能缺少单一可证伪的操作定义，「已经实现」与「已经失败」的口号会周期性回潮。可观察遗产是：学科有了名字、议程与第一批程序；同时留下「一个夏天推进重大问题」的系统性低估。

---

## 4. 符号主义与第一次寒冬

↔ **机制：符号搜索**

### 4.1 黄金十年里看得见的东西

1950 年代末至 1960 年代，主流路径是符号主义：把问题写成符号状态，用逻辑规则与搜索（穷尽或启发式）求解。「智能」的可观察表现是可读的推理轨迹——证明步骤、博弈走法、积木世界指令——而不是拟合曲线。Newell、Shaw 与 Simon 随后的 **General Problem Solver**（通用问题求解器）把「手段—目的分析」做成可运行的启发式搜索。[8]

同期另有几条高光，不必都算符号主义，但同属乐观年代：

- **LISP**（McCarthy，约 1958–1960）长期成为符号 AI 的工作语言。[9]  
- **感知机**（Rosenblatt，1958）：从例子调权重的可训练分类器，并推动 Mark I 硬件；媒体报道一度极度乐观。[16]  
- **Samuel 跳棋程序**（1959）：以自我对弈改进表现，是「机器学习」一词早期有实物对应的案例。[17]  
- **ELIZA**（Weizenbaum，1966）：模式匹配对话，DOCTOR 脚本模仿罗杰斯式心理治疗；作者本意是研究误解，公众却容易把它当成理解。[18]  
- **SHRDLU**（Winograd，1968–1970）：在积木世界里理解并执行自然语言指令，还能回答「刚才为什么这么做」。[19]

在受限微观世界里，这些演示看起来离「通用」只有一步之遥。

### 4.2 乐观为何不可扩展

局限是结构的：

- **知识与规则需人手穷尽。** 真实世界模糊，例外无穷。  
- **搜索空间组合爆炸。** 玩具问题可解，规模一大，启发式也救不过来——这是莱特希尔报告的核心批评。[6]  
- **单层感知机的数学天花板。** 1969 年 Minsky 与 Papert 的 *Perceptrons* 指出单层感知机无法表达异或等结构，沉重打击当时神经网络的经费与士气；多层网络如何训练，当时看不到现实解法，符号路线在制度上更占上风。[20]

ELIZA 与 SHRDLU 的高光也有同一盲点：微观世界一旦打开，模式匹配与手写程序不再够用。

### 4.3 第一次寒冬：从机译收缩到经费总闸

宜把两刀分开，不要写成同一天发生的一件事。

**1966，ALPAC 报告**针对机器翻译：十年投入未达「更快、更便宜、更好」，建议收缩对全自动高质量翻译的资助。这是机译线的重创，还不是整个学科的寒冬。[5]

**1973，莱特希尔报告**受英国科学研究委员会委托，对 AI 作外部评估：诸多子领域未能兑现承诺，玩具方法难以扩展到真实世界规模。[6] **约 1974 年起**，美国 DARPA 等对开放式、缺少明确任务指标的 AI 资助明显收紧。第一次寒冬通常记为 1970 年代中后期。

可观察信号：论文与演示还在，**经费曲线与岗位曲线**先折下去。第一次寒冬不是「AI 永远不行」，而是通用承诺在当时的表示与算力下不可支付。

---

## 5. 专家系统与第二次寒冬

↔ **机制：知识库 + 推理机**

### 5.1 战略收缩：知识就是力量

解冻靠收缩目标：既然造不出通用智能，就先做只懂一件事的专家。Edward Feigenbaum 一派强调「知识就是力量」——可观察架构是 **知识库（大量 `if–then`）+ 推理引擎**。

| 系统 | 时间与域 | 可观察产出 | 限度 |
|------|----------|------------|------|
| DENDRAL | 1960 年代中期起；有机质谱 → 分子结构 | 化学家可复核的结构假设 | 域极窄 |
| MYCIN | 1970 年代；血液感染与抗生素建议 | 域内评估接近专家 | 未成为医院常规工具 [21] |
| XCON / R1 | 约 1980；CMU 为 DEC 配置 VAX 订单 | 进入企业采购，减少配错 | 规则维护随规模恶化 |

专家系统把符号主义推进到可开票的应用：规则条数、知识工程师人时、域内准确率成为可管理指标。1980 年代，Symbolics 等 LISP 专用硬件一度繁荣。日本通产省 **第五代计算机**项目（1982–1992）押注逻辑编程与大规模并行的智能机器，是同期最大的国家级注码。[22]

### 5.2 第二次寒冬：维护成本与硬件泡沫

失效同样可观察：

- **知识获取瓶颈。** 专家口述慢、贵；规则一多就冲突，变更成本超线性。  
- **不会自学。** 域外无规则即无能，与「通用智能」叙事再次脱节。  
- **约 1987。** 通用工作站性能追上昂贵的专用 LISP 机，硬件生态失势；第五代计划 1992 年结束，未达预定目标。[22]

「人工智能」在部分采购与投资语境中成了忌讳词。研究者改挂「机器学习」「模式识别」「数据挖掘」等名——后来所谓寒冬里的伪装。

| | 第一次寒冬 | 第二次寒冬 |
|---|------------|------------|
| 失效机制 | 符号搜索不可扩展 | 知识库不可维护、不学习 |
| 制度信号 | ALPAC（机译）；莱特希尔；DARPA | LISP 机泡沫；第五代落空 |
| 余波 | 实验室收缩 | 改名；统计方法暗中积累 |

同构是承诺—能力鸿沟。异构是：一次偏组合爆炸与评估报告，一次偏知识工程的经济性与硬件生态。

---

## 6. 统计学习转向、深蓝与深层网络火种

↔ **机制：从样本估计映射**（深蓝为搜索旁支）

### 6.1 改名之后的健康化

1990–2000 年代，领域少谈「造出通用智能」，多用可测指标做窄任务：垃圾邮件过滤、推荐、信用评分、语音与视觉中的统计模型（隐马尔可夫模型、支持向量机、提升树等）。优化的是持出样本上的误差或 AUC，不是「是否像人」。特征仍常靠人手设计——这是与深度学习的分界。

### 6.2 深蓝：计算智能标本，不是「学会表征」

1996 年，IBM 深蓝在费城六局中以 2–4 负于卡斯帕罗夫。1997 年 5 月 3–11 日纽约复赛，深蓝以 **3.5–2.5** 取胜，成为在标准赛制下击败在位世界冠军的第一台计算机。IBM 公布的搜索速度约为每秒 **2 亿**个局面；系统靠专用棋类芯片与大规模并行，而不是从棋谱学到深层「棋感」。[23][24]

产业三分法里，这是**计算智能**的高峰：封闭规则、完美信息、强搜索执行器。它不能自动推广到开放世界的感知与生成。勿与统计学习混为一谈，更勿与深度学习混为一谈。

### 6.3 火种：反向传播、LSTM、深信网、ImageNet

连接主义并未死绝，只是长期失宠：

- **1986** Rumelhart、Hinton、Williams 在 *Nature* 阐明用反向传播学习内部表征，使多层网络的梯度训练进入主流视野。数学源头可上溯多次独立发现；这篇使「真能学到有用隐单元」被广泛看见。[25]  
- **1989–1998** Yann LeCun 等将卷积网络用于手写数字，证明局部连接与权值共享在视觉上可行。[26]  
- **1997** Hochreiter 与 Schmidhuber 提出 **LSTM**，缓解长序列上的梯度问题，日后撑起语音与翻译产业中的一批系统。[27]  
- **2006** Hinton、Osindero、Teh 关于深度置信网的工作，重开「深层可预训练」叙事，并为「深度学习」正名。[28]  
- **2009** Deng 等发布 **ImageNet**：约 **1400 万**张标注图像、两万余类；2010 年起举办 ILSVRC（竞赛子集约为 120 万张、1000 类）。赌注是：算法不行，也许首先是数据不够。[29]

算法传统（反传、卷积、LSTM）、GPU 并行、大数据集——三要素在 2012 年前分别就位，尚未在同一公开榜单上同时炸开。

---

## 7. 深度学习上台：从 AlexNet 到 AlphaGo

↔ **机制：可学习的深层表征**

### 7.1 2012：AlexNet 把三要素写成榜单事实

Krizhevsky、Sutskever、Hinton 的 **AlexNet** 在 ILSVRC 2012 取得 **15.3%** 的 top-5 测试错误率（七网络集成；单网络为 18.2%）；第二名（手工视觉特征 + Fisher Vector 等）为 **26.2%**。成熟视觉竞赛里，年度通常只挪一两个百分点；十个百分点是台阶，不是噪声。[38]

网络约 6000 万参数，用两块 NVIDIA GTX 580 训练。卷积与反向传播并不全新——Ciresan 等已在较小竞赛中用 GPU 卷积网络取胜——新的是可观察组合：ImageNet 级数据 + 可训练深度卷积 + 跑通的训练配方（ReLU、Dropout、数据增强）。[38][39]

产业进入**感知智能**：语音、视觉、若干自然语言任务从不好用到可以用。输出仍多为标签、检测框、序列标注——能认出「这是猫」，默认还不会按指令「画一只猫」。[4]

### 7.2 2016–2017：AlphaGo——深度学习加搜索的博弈高峰

围棋分支因子远高于国际象棋，穷尽硬算不可行。2016 年 3 月，DeepMind 的 **AlphaGo** 以 **4:1** 战胜李世石：策略网络与价值网络提供走法与局势估计，蒙特卡洛树搜索负责规划，自我对弈数据用于改进。[30] 第二局第 37 手被职业棋手起初视为失误，随后公认为好棋——可观察印象是：策略可以来自人类棋谱分布之外。

2017 年 10 月，**AlphaGo Zero** 在 *Nature* 发表：不使用人类棋谱，从零自我对弈，并迅速超过战胜李世石的版本。[31]

逻辑归属仍是深度学习高峰（表征学习 + 规划），尚未进入大规模预训练语言模型范式。与深蓝的差异可观察为：深蓝几乎不学棋感表征；AlphaGo 显式学习策略网络与价值网络。

---

## 8. 大模型：Transformer、预训练到破圈

↔ **机制：大规模条件生成**

### 8.1 架构：注意力就是一切（2017）

Vaswani 等 *Attention Is All You Need* 提出 **Transformer**：去掉循环与卷积，仅靠注意力做序列转导。在 WMT 2014 英德翻译上，大号模型达到 28.4 BLEU，并可在八块 GPU 上数日内完成训练。[10] 当时定位是机器翻译的质量与可并行性；此后解码器-only 的 Transformer 成为主流大语言模型底座。可观察遗产是：训练可横向堆加速器，规模化从工程上变得可行。

### 8.2 预训练范式：BERT、GPT 到 GPT-3（2018–2020）

- **GPT-1**（Radford 等，2018）：自回归语言建模 + 下游微调。[32]  
- **BERT**（Devlin 等，2018）：双向编码器与掩码语言模型，强调先通用预训练、再下游微调。[33]  
- **GPT-3**（Brown 等，2020）：约 **1750 亿**参数，系统展示少样本与零样本——许多任务无需梯度更新，仅靠提示中的示例即可完成。[11]

能力形态跨入**生成智能**：按条件产出新的文本候选；扩散等路线随后把同一逻辑扩到图像。这是训练分布上的统计生成，不等于人类意图、理解或问责；但文本与代码的边际生产成本被显著压低。[4]

**缩放律**把「做大」部分地从手艺变成可讨论的工程问题。Kaplan 等（2020）报告交叉熵损失随模型规模、数据量、算力呈幂律下降。[34] Hoffmann 等（2022，Chinchilla）修正了算力最优配比：在给定算力下，许多大模型训练数据不足，参数与 token 宜近似同比例放大。[35] 资本因而更容易为规模下注，但最优配方并非一次写死。

### 8.3 产品破圈：ChatGPT 与对齐（2022）

2022 年 11 月 30 日，OpenAI 发布 **ChatGPT** 研究预览：基于 GPT-3.5 系列，以对话界面提供能力，并强调来自人类反馈的强化学习（RLHF）。InstructGPT 一脉已表明：用人类偏好微调，可使模型更跟随指令、少编造、少有害输出。[12][36]

宜分开两件事：

- **模型能力**——GPT-3 已展示少样本；  
- **产品形态**——对话框 + 对齐后的可用行为。

技术增量未必等于产品革命。把已有能力交给无技术背景的大众，才造成破圈。2023 年 GPT-4 等多模态能力进入公众演示；各地出现密集的大模型发布。竞争从论文榜单扩展到 API、开放权重、推理成本与应用生态。

### 8.4 科学承认（2024）

2024 年诺贝尔**物理学奖**授予 John Hopfield 与 Geoffrey Hinton，「for foundational discoveries and inventions that enable machine learning with artificial neural networks」。[13]

同年**化学奖**：一半授予 David Baker，「for computational protein design」；另一半授予 Demis Hassabis 与 John Jumper，「for protein structure prediction」（AlphaFold）。[14]

这不是口号「AI 拿了诺奖」，而是两条可核对的授奖理由：物理启发的学习方法，与 AI 驱动的科学工具，同时进入最高科学表彰体系。结构预测仍然要实验验证——工具扩展了科学家能尝试的范围，并不自动替换判断。

```mermaid
flowchart LR
  A["感知 / 博弈"] --> B["Transformer"]
  B --> C["预训练"]
  C --> D["对话 + RLHF"]
  D --> E["工具循环"]
```

---

## 9. 智能体：从会聊到办事

↔ **机制：模型 + 工具 + 多步循环**

### 9.1 单次生成与多步办事

无工具时是聊天或生成；有工具与外部状态时才是办事。Yao 等（2022）的 **ReAct** 把推理轨迹与行动交错：模型先想、再调用、再观察、再想。[37] 产业实践中，智能体外挂检索、代码执行、浏览器、业务 API，循环执行：

**计划 → 行动（调用）→ 观察 → 再计划。**

可观察焦点从「单次回答好不好」转为：整条轨迹是否达成目标、调用是否合规、人工介入几次。评测必须含轨迹与副作用，不能只用静态题库。[4]

### 9.2 2025：推理模型与开放权重

2025 年 1 月 20 日，DeepSeek 发布 **DeepSeek-R1** 等推理向模型，强调开放权重、推理基准与相对成本，迅速成为全球讨论焦点。相对闭源旗舰的优劣随版本与基准变化，应以官方技术报告与独立评测为准。[15]

竞争焦点转向推理时算力、开源生态、工具协议，以及企业侧的权限、评测与编排——即本库 [50](../50-strategy/50-ai-industry-disruption-strategy.md) 所说的智能体与企业操作系统。具身智能与人形机器人把同一逻辑伸向物理世界。对「元年」「调用量反超」「全球最大」等通稿，本文不升格为已审计事实。

---

## 10. 回收：阶段、机制与可带走的判断

### 10.1 五阶段 × 六机制

| 史段 | 主导机制 | 编年 | 标志性可观察事实 |
|------|----------|------|------------------|
| 符号主义 | 符号搜索 | §4 | Logic Theorist（38/52）；ALPAC；莱特希尔 |
| 专家系统 | 知识库规则 | §5 | MYCIN / XCON；LISP 机与第五代退潮 |
| 机器学习 | 统计拟合（深蓝为搜索旁支） | §6 | 窄任务指标；深蓝 3.5–2.5 |
| 深度学习 | 深层表征 | §7 | AlexNet 15.3% 对 26.2%；AlphaGo 4:1 |
| 大模型 | 预训练生成 → 延伸为智能体 | §8–§9 | Transformer；GPT-3；ChatGPT；Agent |

### 10.2 产业三分（能力形态，不是另两段史）

| 形态 | 机器在做什么 | 大致对齐 |
|------|--------------|----------|
| 计算 | 封闭规则下算赢 | 深蓝；早期搜索 |
| 感知 | 认出已有结构 | AlexNet 起 |
| 生成 | 产出新候选 | GPT 系、扩散等 |

### 10.3 可带走的判断

1. **先认机制，再谈智能。** 规则、拟合、深层表征、序列生成、工具循环，不是同一个东西。  
2. **可观察优先。** 展开节点、规则条数、持出误差、竞赛错误率、推理成本、任务轨迹，比口号硬。  
3. **寒冬同构。** 承诺不可支付，经费与声誉先走。  
4. **爆发同构。** 表示或算法、数据、算力对齐，再加一层产品界面。  
5. **大模型不是突变。** 规模化条件生成 + 对话产品；智能体是工具循环延伸。  
6. **下一问。** 生成变便宜之后，判断与问责归谁？护城河在专有数据与嵌入工作流的智能。[4]

---

## 11. 节点年表

| 时间 | 事件 | 节 | 出处 |
|------|------|----|------|
| 1943 | McCulloch–Pitts 神经元逻辑模型 | §3 | [3] |
| 1950 | 图灵《计算机器与智能》 | §3 | [2] |
| 1955-08-31 | 达特茅斯 AI 夏季研究提案 | §3 | [1] |
| 1956 | 达特茅斯会议；Logic Theorist（38/52） | §3 | [1][7] |
| 1958–1960 | 感知机；LISP | §4 | [16][9] |
| 1959 | Samuel 跳棋与「机器学习」 | §4 | [17] |
| 1966 | ALPAC 报告；ELIZA | §4 | [5][18] |
| 1968–1970 | SHRDLU | §4 | [19] |
| 1969 | *Perceptrons* | §4 | [20] |
| 1973–约 1974 | 莱特希尔报告；DARPA 收缩 | §4 | [6] |
| 1970s–1980 | MYCIN、XCON | §5 | [21] |
| 1982–1992 | 日本第五代计算机 | §5 | [22] |
| 约 1987 | LISP 机失势；第二次寒冬加深 | §5 | [22] |
| 1986 | 反向传播 *Nature* 文 | §6 | [25] |
| 1997-05 | 深蓝胜卡斯帕罗夫 3.5–2.5；LSTM | §6 | [23][27] |
| 2006 / 2009 | 深信网；ImageNet | §6 | [28][29] |
| 2012 | AlexNet：15.3%（集成）对 26.2% | §7 | [38] |
| 2016–2017 | AlphaGo 4:1；AlphaGo Zero | §7 | [30][31] |
| 2017 | Transformer | §8 | [10] |
| 2018–2020 | GPT-1、BERT、GPT-3；Kaplan 缩放律 | §8 | [32][33][11][34] |
| 2022 | Chinchilla；InstructGPT / RLHF；ChatGPT（11-30）；ReAct | §8–§9 | [35][36][12][37] |
| 2024 | 诺贝尔物理（Hopfield / Hinton）；化学（Baker；Hassabis / Jumper） | §8 | [13][14] |
| 2025-01-20 | DeepSeek-R1；智能体叙事升温 | §9 | [15] |

---

## 12. 参考文献

1. McCarthy, J., Minsky, M., Rochester, N., Shannon, C. E. *A Proposal for the Dartmouth Summer Research Project on Artificial Intelligence* (1955-08-31). <https://www-formal.stanford.edu/jmc/history/dartmouth/dartmouth.html>
2. Turing, A. M. *Computing Machinery and Intelligence*. *Mind* 59(236): 433–460 (1950).
3. McCulloch, W. S., Pitts, W. *A Logical Calculus of the Ideas Immanent in Nervous Activity*. *Bulletin of Mathematical Biophysics* 5: 115–133 (1943).
4. 本库：[`50-ai-industry-disruption-strategy.md`](../50-strategy/50-ai-industry-disruption-strategy.md)。
5. Automatic Language Processing Advisory Committee. *Language and Machines: Computers in Translation and Linguistics* (ALPAC Report). National Academy of Sciences, 1966.
6. Lighthill, J. *Artificial Intelligence: A General Survey* (1973). UK Science Research Council. <https://www.aiai.ed.ac.uk/events/lighthill1973/lighthill.pdf>
7. Newell, A., Shaw, J. C., Simon, H. A. Empirical explorations of the Logic Theory Machine. *Proceedings of the Western Joint Computer Conference*, 1957. 通行叙述：证明《数学原理》第二章前 52 条中的 38 条。
8. Newell, A., Shaw, J. C., Simon, H. A. Report on a general problem-solving program. *Proceedings of the International Conference on Information Processing*, 1959.
9. McCarthy, J. Recursive functions of symbolic expressions and their computation by machine, Part I. *Communications of the ACM* 3(4): 184–195 (1960).
10. Vaswani, A. et al. *Attention Is All You Need*. arXiv:1706.03762 (2017).
11. Brown, T. B. et al. *Language Models are Few-Shot Learners*. arXiv:2005.14165 (2020).
12. OpenAI. *Introducing ChatGPT* (2022-11-30). <https://openai.com/index/chatgpt/>
13. The Nobel Prize in Physics 2024 (Hopfield & Hinton). <https://www.nobelprize.org/prizes/physics/2024/press-release/>
14. The Nobel Prize in Chemistry 2024 (Baker; Hassabis & Jumper). <https://www.nobelprize.org/prizes/chemistry/2024/press-release/>
15. DeepSeek-R1 发布（2025-01-20）；独立评述见 Epoch AI. <https://epoch.ai/gradient-updates/what-went-into-training-deepseek-r1>
16. Rosenblatt, F. The perceptron: A probabilistic model for information storage and organization in the brain. *Psychological Review* 65(6): 386–408 (1958).
17. Samuel, A. L. Some studies in machine learning using the game of checkers. *IBM Journal of Research and Development* 3(3): 210–229 (1959).
18. Weizenbaum, J. ELIZA—a computer program for the study of natural language communication between man and machine. *Communications of the ACM* 9(1): 36–45 (1966).
19. Winograd, T. *Understanding Natural Language*. Academic Press, 1972. 程序完成于 MIT，约 1968–1970。
20. Minsky, M., Papert, S. *Perceptrons*. MIT Press, 1969.
21. Shortliffe, E. H. *Computer-Based Medical Consultations: MYCIN*. Elsevier, 1976. 后续综述见 Buchanan, B. G., Shortliffe, E. H. 编 *Rule-Based Expert Systems* (Addison-Wesley, 1984)。域内评估接近专家，未成为医院常规工具。
22. 日本第五代计算机项目（ICOT / MITI），1982–1992。同期通俗叙述见 Feigenbaum, E. A., McCorduck, P. *The Fifth Generation* (Addison-Wesley, 1983)。LISP 专用工作站在通用 UNIX 工作站冲击下于 1980 年代末失势。
23. Campbell, M., Hoane, A. J., Hsu, F.-h. Deep Blue. *Artificial Intelligence* 134(1–2): 57–83 (2002). 1997 年复赛 3.5–2.5；搜索速度量级见 IBM 公开叙述约 2 亿局面/秒。
24. IBM. Deep Blue. <https://www.ibm.com/history/deep-blue>
25. Rumelhart, D. E., Hinton, G. E., Williams, R. J. Learning representations by back-propagating errors. *Nature* 323: 533–536 (1986).
26. LeCun, Y. et al. Gradient-based learning applied to document recognition. *Proceedings of the IEEE* 86(11): 2278–2324 (1998).
27. Hochreiter, S., Schmidhuber, J. Long short-term memory. *Neural Computation* 9(8): 1735–1780 (1997).
28. Hinton, G. E., Osindero, S., Teh, Y.-W. A fast learning algorithm for deep belief nets. *Neural Computation* 18(7): 1527–1554 (2006).
29. Deng, J. et al. ImageNet: A large-scale hierarchical image database. *CVPR* (2009). 约 1400 万张标注图像。
30. Silver, D. et al. Mastering the game of Go with deep neural networks and tree search. *Nature* 529: 484–489 (2016).
31. Silver, D. et al. Mastering the game of Go without human knowledge. *Nature* 550: 354–359 (2017).
32. Radford, A. et al. *Improving Language Understanding by Generative Pre-Training* (GPT-1). OpenAI, 2018.
33. Devlin, J. et al. *BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding*. arXiv:1810.04805 (2018).
34. Kaplan, J. et al. *Scaling Laws for Neural Language Models*. arXiv:2001.08361 (2020).
35. Hoffmann, J. et al. Training compute-optimal large language models (Chinchilla). *NeurIPS* (2022).
36. Ouyang, L. et al. Training language models to follow instructions with human feedback (InstructGPT). arXiv:2203.02155 (2022).
37. Yao, S. et al. *ReAct: Synergizing Reasoning and Acting in Language Models*. arXiv:2210.03629 (2022).
38. Krizhevsky, A., Sutskever, I., Hinton, G. E. ImageNet Classification with Deep Convolutional Neural Networks. *Advances in Neural Information Processing Systems* 25 (2012). 七网络集成 top-5 测试错误率 15.3%；单网络 18.2%；第二名 26.2%。
39. Ciresan, D. C., Meier, U., Masci, J., Gambardella, L. M., Schmidhuber, J. Flexible, High Performance Convolutional Neural Networks for Image Classification. *IJCNN* (2011).

---

> **可核对的是节点与文献。可对照的是承诺与能力的鸿沟。可带走的是机制编号，以及生成变便宜之后判断仍归谁。**
