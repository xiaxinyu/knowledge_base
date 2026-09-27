# 大语言模型编年：从循环网络到 Transformer

> 自然语言曾是文明延续的桥梁；机器要学会在同一条桥上走，先得学会「下一个词最可能是什么」。
>
> 本文是大语言模型（Large Language Model，LLM）的技术编年：从神经元与马尔可夫链的数学萌芽，经循环网络与注意力，到 Transformer、预训练范式、规模跃迁与 ChatGPT 破圈。可与本库 [12](./12-artificial-intelligence-chronicle.md)（六种机制与五次换代）、[11](./11-information-theory-chronicle.md)（信息与预测）、[14](./14-internet-history-chronicle.md)（入口与收束）、[50](../50-strategy/50-ai-industry-disruption-strategy.md)（产业穿透）对照：一篇写智能如何换代，本文把第五阶段——语言侧的规模生成——剖开写细。

先给一个直接答案：

> **大语言模型的核心不是「有了一个灵魂」，而是在海量文本上学会条件续写：给定前文，按概率生成下一个 token。** 人脑约 860 亿神经元、突触以十万亿计，是生物演化的指挥中心；LLM 以参数规模与训练算力换表示能力——参数不等于神经元，传闻中的「1.76 万亿」亦非官方口径。[^param-rumor] 路线上可记四段：**前史与循环网络**（McCulloch–Pitts → RNN / LSTM）→ **注意力与 Transformer**（2017）→ **预训练 + 微调 / 少样本**（GPT / BERT → GPT-3）→ **对齐、产品破圈与多模态**（InstructGPT / ChatGPT → GPT-4 / 开源潮）。幻觉是默认失效模式；RAG、工具调用与人类反馈是工程上的按住方式，不是神话结局。

**10-chronicle 系列位置：** [12](./12-artificial-intelligence-chronicle.md) 写整部 AI 机制与阶段；[11](./11-information-theory-chronicle.md) 写「不确定性如何被度量」。本文写**语言建模一条河**：序列如何被表示、依赖如何被捕捉、规模如何变成产品。术语先看第 1.1 节。

## 摘要

自然语言处理长期在统计与神经两条河上汇流。1906 年马尔可夫链为序列概率提供数学语言；1943 年 McCulloch–Pitts 神经元与 1950 年图灵测试框出「机器能否用语言表现智能」；1956 年达特茅斯会议立科。1980–1990 年代反向传播、Hopfield、RNN / LSTM / BRNN 与 IBM 对齐模型，把序列学习做成可训练的工程。2003 年 Bengio 等神经概率语言模型用分布式词表示对抗维数灾难；2013–2016 年 Word2Vec、Seq2Seq、GRU 与注意力机制把神经机器翻译推上主流。2017 年 *Attention Is All You Need* 去掉循环与卷积，Transformer 成为此后几乎一切大模型的底座。2018–2019 年 GPT-1（Decoder-Only）与 BERT（Encoder-Only）证明「预训练 + 微调」；GPT-2 / GPT-3 把能力推向零样本与少样本；T5 把任务统一成文本到文本。2020 年前后缩放律、RAG 与指令对齐并行生长；2022 年 BLOOM 与 Stable Diffusion 推开源与 AIGC，ChatGPT（2022-11-30）用对话框把能力交给大众。2023–2024 年 GPT-4、Claude、Gemini、Llama / Qwen / Mistral 等开源与闭源竞逐，多模态与推理模型（如 o1 一脉）把「会聊」推向「会想步骤」。本文是节点编年，不是厂商评测；参数、份额与路线图会动，冲突时以论文、官方公告与可复核基准为准。

**关键词：** 大语言模型；Transformer；注意力；RNN；LSTM；预训练；GPT；BERT；缩放律；RAG；ChatGPT；Decoder-Only；对齐；幻觉

---

## 目录

- [摘要](#摘要)
1. [读法与术语](#1-读法与术语)
    - [1.1 术语对照](#11-术语对照)
    - [1.2 边界](#12-边界)
2. [前史：神经元、马尔可夫与智能立科](#2-前史神经元马尔可夫与智能立科)
3. [循环网络：从感知器到 LSTM](#3-循环网络从感知器到-lstm)
4. [统计语言与对齐：SMT 的河床](#4-统计语言与对齐smt-的河床)
5. [预训练前夜：神经语言、Seq2Seq 与注意力](#5-预训练前夜神经语言seq2seq-与注意力)
6. [Transformer：注意力就是一切](#6-transformer注意力就是一切)
    - [6.1 QKV 与多头注意力](#61-qkv-与多头注意力)
    - [6.2 编码器、解码器与三种组装](#62-编码器解码器与三种组装)
7. [2018—2019：预训练范式立住](#7-20182019预训练范式立住)
    - [7.1 GPT-1 与 BERT](#71-gpt-1-与-bert)
    - [7.2 GPT-2 到 T5、ALBERT](#72-gpt-2-到-t5albert)
8. [2020—2022：规模、开源与检索增强](#8-20202022规模开源与检索增强)
    - [8.1 Meena、GPT-3 与独家授权](#81-meenagpt-3-与独家授权)
    - [8.2 DeBERTa、LaMDA、RAG 与开源节点](#82-debertalamdarag-与开源节点)
9. [2022：ChatGPT 破圈](#9-2022chatgpt-破圈)
10. [2023—2024：多模态、开源潮与推理模型](#10-20232024多模态开源潮与推理模型)
    - [10.1 2023：GPT-4、Claude 与开源底座](#101-2023gpt-4claude-与开源底座)
    - [10.2 2024：旗舰迭代与推理分支](#102-2024旗舰迭代与推理分支)
11. [尾声：工具、仿生与尚未开始的旅程](#11-尾声工具仿生与尚未开始的旅程)
12. [本章要点](#12-本章要点)
13. [参考文献](#13-参考文献)

---

## 1. 读法与术语

自然语言是文明延续的桥梁；机器学习则尝试从数据里算出可复用的规律——这些规律叫**模型**，在人类世界常叫**经验**。人脑约 **860 亿**神经元、突触约 **100 万亿**量级，只作数量级对照，不是精确审计；参数也不等于神经元。[^brain-scale] 大语言模型是另一条工程路径：用可微分网络与海量文本，逼近「理解与表达」。整部 AI 的六种机制与寒冬见 [12](./12-artificial-intelligence-chronicle.md)；本文只盯**语言与序列**。

### 1.1 术语对照

| 术语 | 一句话 |
| ---- | ------ |
| **语言模型（LM）** | 估计词 / token 序列的概率；现代 LLM 多做「给定前文，预测下一个 token」。 |
| **Token / Tokenization** | 模型读写的基本单位；可以是子词、字节对、标点等，不一定是「单词」。 |
| **参数（parameter）** | 训练中学到的权重数量。规模常写作百万（M）、十亿（B）、万亿（T）；**不等于**神经元个数。 |
| **RNN / LSTM / GRU** | 循环网络及其门控变体：按时间步传递隐状态，擅长序列，却难并行、难抓极长依赖。 |
| **注意力（Attention）** | 生成每一步时，动态决定「看」序列中哪些位置；可写成对 Value 的加权和，权重由 Query 与 Key 的相似度决定。 |
| **Transformer** | 2017 年提出的架构：以（多头）自注意力与前馈层为主，去掉对循环 / 卷积的依赖，便于并行堆算力。 |
| **Encoder-Only / Decoder-Only / Encoder–Decoder** | 三种常见组装：偏理解（如 BERT）、偏自回归生成（如 GPT）、偏序列到序列（如原版 Transformer、T5）。 |
| **预训练 + 微调** | 先在大规模无标注（或自监督）语料上学通式表示，再在下游任务上少量标注适配。 |
| **Zero-Shot / Few-Shot** | 不给或只给极少示例，靠提示完成任务；GPT-2 / GPT-3 把它推到公众视野。 |
| **对齐（Alignment）** | 让模型输出更符合人类意图与安全偏好；InstructGPT / RLHF 是产品破圈前的关键一环。 |
| **幻觉（Hallucination）** | 流畅却无据或错误的生成。续写目标本身不奖励「我不知道」。 |
| **RAG** | 检索增强生成：先取外部文档，再与提示一并生成，用非参数记忆补参数记忆之不足。 |
| **缩放律（Scaling Laws）** | 在一定范围内，损失随模型、数据、算力呈可描述的幂律趋势；最优配比会随研究修正（如 Chinchilla）。 |
| **SOTA** | State of the Art：当时公开基准上的最佳水平；榜单会动，正文取方向不冻结分数。 |

### 1.2 边界

1. **本文写语言 / 生成主线，不重写整部 AI 史。** 符号主义、专家系统、寒冬与智能体办事，见 [12](./12-artificial-intelligence-chronicle.md)。  
2. **参数量与训练代价以论文 / 官方为准。** 传闻（如 GPT-4「1.76T」）单独标注，不写成事实。[^param-rumor]  
3. **开源时间线密集，正文取结构节点。** 每周新权重不是编年义务；Llama、Qwen、Mistral、Claude、Gemini 等代表路线，不构成完整产品目录。  
4. **口语钩子可保留，判断须可核验。** 「Close AI」一类嘲讽是舆论史，不是架构结论。  
5. **仿生是隐喻。** 循环半圆、兴奋环路启发了网络想象；现代 LLM 仍是高维函数拟合，不是突触的字面复制。

---

## 2. 前史：神经元、马尔可夫与智能立科

### 2.1 1901—1933：环路想象

1901 年，西班牙神经学家圣地亚哥·拉蒙·卡哈尔（Santiago Ramón y Cajal）在小脑皮质中观察到由平行纤维、浦肯野细胞和颗粒细胞形成的「循环半圆」——此处「循环」指解剖学上的环状结构。[^cajal]

1906 年，俄国数学家安德烈·马尔可夫（Andrey Andreyevich Markov）提出**马尔可夫链**：下一状态只依赖当前状态的随机过程模型。它为描述序列数据提供了数学基础，并启发了后来的统计语言模型（Statistical Language Model，SLM）与许多序列方法。[^markov]

1933 年，拉斐尔·洛伦特·德诺（Rafael Lorente de Nó）通过高尔基染色法描述神经元的「循环、相互连接」，并提出兴奋环路可解释前庭眼反射的某些方面。反馈结构的讨论，与把神经系统理解为纯粹前馈的图景形成对照。[^lorente]

### 2.2 1943—1956：可计算的神经元与学科之名

1943 年，沃伦·麦卡洛克（Warren McCulloch）与沃尔特·皮茨（Walter Pitts）研究包含循环连接的人工神经网络，提出 **McCulloch–Pitts 神经元模型**：用逻辑阈值刻画神经元兴奋与抑制。[^mcculloch-pitts]

1950 年，艾伦·图灵（Alan Mathison Turing）在 *Mind* 发表 *Computing Machinery and Intelligence*，提出以自然语言的理解与生成任务评估机器智能——后称**图灵测试**。自然语言处理（Natural Language Processing，NLP）的问题域由此进入公共讨论，但当时尚未作为独立学科被系统阐述。[^turing-1950]

1955 年，约翰·麦卡锡（John McCarthy）筹备小组澄清「思维机器」相关想法；领域曾有控制论、自动机理论、复杂信息处理等名称。为保持中立，他选用 **Artificial Intelligence**。1956 年达特茅斯夏季研讨会由麦卡锡牵头，联合克劳德·香农（Claude Shannon）、纳撒尼尔·罗切斯特（Nathaniel Rochester）、马文·明斯基（Marvin Minsky）等召开，被誉为人工智能的「制宪会议」，标志学科正式诞生。[^dartmouth]

**所以 · 边界在哪：** 这一段还没有「大模型」。它留下三样东西：序列的概率语言、可计算的神经元，以及「机器能否用语言表现智能」这一考题。

---

## 3. 循环网络：从感知器到 LSTM

### 3.1 1960—1961：感知器与交叉耦合

1960 年，弗兰克·罗森布拉特（Frank Rosenblatt）提出「闭环交叉耦合感知器」一类三层网络：中间层神经元之间存在循环连接，并按赫布规则调整，意在模拟联想记忆。1961 年，他在 *Principles of Neurodynamics* 中描述闭环交叉耦合与反向耦合感知器，并讨论赫布学习；完全交叉耦合网络在理论上可与极深的前馈结构相联系。罗森布拉特是人工神经网络的早期开拓者；「深度学习之父」这一称号在通行叙事里更常留给后来的 Hinton 等人——正文不把荣誉头衔写死。[^rosenblatt]

### 3.2 1982—1989：Hopfield、反向传播与 RNN 之名

1982 年，约翰·霍普菲尔德（John Hopfield）提出 **Hopfield 网络**，可用于内容可寻址存储；2024 年诺贝尔物理学奖将其与 Hinton 的工作一并表彰，见 [12](./12-artificial-intelligence-chronicle.md)。[^hopfield]

1986 年，大卫·鲁梅尔哈特（David Rumelhart）、杰弗里·辛顿（Geoffrey Hinton）与罗纳德·威廉姆斯（Ronald J. Williams）发表 *Learning Representations by Back-propagating Errors*，扩展反向传播以训练含循环连接的网络，并讨论时间序列；**循环神经网络（Recurrent Neural Network，RNN）** 一词在此后的文献与教学中被正式定着。[^backprop-1986]

1987 年，罗伯特·艾伦（Robert B. Allen）演示用前馈网络把自动生成的英语句子译成西班牙语：输入输出层大小被设成恰好容纳最长句——因为网络尚无「把任意长序列压成固定表示」的机制。他在总结中暗示自动联想式编码器—解码器的可行性。[^allen-1987]

1989 年，Williams 与大卫·泽普瑟（David Zipser）深入研究训练递归网络的方法，包括**时间反向传播（Backpropagation Through Time，BPTT）**，为序列预测与 NLP 中的梯度训练提供了路径。[^bptt]

### 3.3 1991—1997：LSTM 与双向 RNN

1991 年前后，尤尔根·施密德胡贝尔（Jürgen Schmidhuber）等系统探讨 RNN 的长期依赖困难。1995–1997 年，塞普·霍赫赖特（Sepp Hochreiter）与施密德胡贝尔提出**长短期记忆网络（Long Short-Term Memory，LSTM）**：用门控与恒定误差轮播（CEC）等机制缓解梯度消失 / 爆炸，显著提升对长期依赖的捕捉。[^lstm]

1997 年，迈克·舒斯特（Mike Schuster）与库尔迪普·帕利瓦尔（Kuldip K. Paliwal）提出**双向循环神经网络（BRNN）**：两个相反方向的 RNN 处理同一输入，以利用双向上下文。简单记：**LSTM 是单元改进，BRNN 是结构改进**；后来二者常结合为 BiLSTM。[^brnn]

**所以 · 边界在哪：** 序列终于可以被「一步步」训练。代价是难并行、长程依赖仍脆——这两笔债，要等到注意力与 Transformer 来还。

---

## 4. 统计语言与对齐：SMT 的河床

1990 年，IBM 研究团队提出著名的 **IBM 对齐模型（IBM Alignment Models）**，为统计机器翻译（Statistical Machine Translation，SMT）奠基：词汇翻译概率、位置对齐、生育率等概念，系统化处理源语言与目标语言的对齐。[^ibm-align]

2000 年前后，IBM 与 Google 等用隐马尔可夫模型（HMM）等技术继续改进机器翻译质量。2001 年，IBM 启动以第一任 CEO Thomas J. Watson Sr. 命名的问答系统 **Watson**，目标是回答自然语言问题；2011 年在电视问答节目 *Jeopardy!* 中击败传奇选手布拉德·鲁特（Brad Rutter）与肯·詹宁斯（Ken Jennings）。[^watson] 那是检索、评分与知识工程的高峰，还不是今天的生成式 LLM——但「用自然语言问机器」的产品想象已经公开站上舞台。

**所以 · 边界在哪：** SMT 证明：语言可以当成对齐与概率问题来做工程。神经方法后来取代的是这条河床上的主流实现，不是取消「概率」本身。

---

## 5. 预训练前夜：神经语言、Seq2Seq 与注意力

### 5.1 2003—2012：分布式表示与数据洪流

2003 年，约书亚·本吉奥（Yoshua Bengio）等发表 *A Neural Probabilistic Language Model*：用神经网络学习词的分布式表示，并在连续空间中估计序列概率，以对抗维数灾难。[^bengio-2003] **不宜写成「2003 年 SMT 已被 NMT 取代」**——端到端神经机器翻译要到 2014 年前后才成主流；2003 年的贡献是神经语言建模与词向量思想的关键火种。

2006 年起，BiLSTM 等架构在语音识别、语言建模等任务上不断刷新记录；LSTM 与 CNN 的结合也改进了图像字幕等跨模态任务。2009 年前后，互联网语料使大规模统计语言模型在多数任务上压过符号规则系统。2012 年深度学习在图像上的突破（AlexNet）之后，同一套表示学习浪潮迅速打进语言建模——感知侧的故事见 [12](./12-artificial-intelligence-chronicle.md)。

2013 年，Mikolov 等 **Word2Vec** 把「词向量」做成可复用的工业零件：相近词在向量空间中靠近，类比运算一时成为科普符号。[^word2vec]

### 5.2 2014—2016：GRU、注意力与 NMT 上台

2014 年 6 月，赵京贤（Kyunghyun Cho）等提出 **GRU（Gated Recurrent Unit）**：用重置门与更新门简化 LSTM，降低计算开销。[^gru] 同年，Sutskever 等与 Cho 等把编码器—解码器做成端到端 **Seq2Seq**；9 月，巴赫达瑙（Dzmitry Bahdanau）、赵京贤与本吉奥引入**注意力机制**——解码时动态关注源序列不同部分，显著提升神经机器翻译。[^attention-2014]

2015 年 12 月，山姆·奥特曼（Sam Altman）、埃隆·马斯克（Elon Musk）、伊利亚·苏茨凯弗（Ilya Sutskever）、格雷格·布罗克曼（Greg Brockman）等创立 **OpenAI**。[^openai-2015]

2016 年，Google 将翻译服务转向深度 LSTM + Seq2Seq 的 **NMT**，神经网络成为机器翻译主流路径。[^gnmt] 同年，克莱芒·德朗格（Clément Delangue）、朱利安·肖蒙（Julien Chaumond）与托马斯·沃尔夫（Thomas Wolf）以 Unicode「🤗」拥抱表情在纽约成立 **Hugging Face**，早期业务是面向青少年的聊天机器人；后来转型为模型与数据集的基础设施枢纽。[^hf-2016]

**所以 · 边界在哪：** 「读入序列 → 压成表示 → 再生成序列」已经齐了；注意力让模型学会**对齐着看**。还缺一块：摆脱逐步循环，好在 GPU 集群上横向铺开——那就是下一节。

---

## 6. Transformer：注意力就是一切

### 6.1 QKV 与多头注意力

2017 年，谷歌与多伦多大学等研究者在 NeurIPS 介绍 *Attention Is All You Need*：不再依赖循环与卷积，提出完全基于注意力的编码器—解码器架构——**Transformer**。NLP 与此后大模型的计算方式由此改写。[^transformer]

在此之前，RNN 靠反馈逐步理解先后；词语顺序不同，语义不同。Transformer 用 **QKV（Query、Key、Value）** 结构做信息路由：

| 角色 | 直觉 |
| ---- | ---- |
| **Query（Q）** | 当前要关注的位置发出的「查询」：我需要什么信息？ |
| **Key（K）** | 各位置提供的「索引」：我这里有什么可被匹配的？ |
| **Value（V）** | 真正被聚合的内容：匹配上之后取出什么？ |

注意力输出可理解为：用 Q 与各 K 的相似度决定权重，再对 V 加权求和——教学上常写成 \(\mathrm{Attention}(Q,K,V)=\mathrm{softmax}(QK^\top/\sqrt{d})V\)。[^transformer]

教学上可用一句话抓住直觉：用问题去对索引、再取出内容——但真实语音助手里，地点检索与业务工具往往在注意力层之外；层内做的仍是**对表示做加权聚合**，不是字面「查天气 API」。

许多人以为大模型在「思考后回答」；机制上，它首先是**根据已生成前缀，预测下一个最可能的 token**。问答、翻译、对话都是这一能力在不同提示下的外显。

为提升表达力与效率，可将表示拆成多个子空间分别做注意力再拼接——**多头注意力（Multi-Head Attention）**：从不同「角度」看同一序列。

### 6.2 编码器、解码器与三种组装

原生 Transformer 面向机器翻译等 Seq2Seq 任务，含 **Encoder** 与 **Decoder**：

- **编码器**：读入源序列，得到上下文表示。  
- **解码器**：结合编码器输出与已生成前缀，逐步生成目标序列。

教学隐喻：浏览俄文网页时点「翻译」——编码器把源文读成内部表示，解码器据此写出中文。工程上更枯燥：分词（Tokenization）→ 嵌入为向量 → 多层自注意力与前馈网络（Feed-Forward）→ 按概率采样下一 token，循环直至结束。

每层中，token 表示常与三个权重矩阵 \(W_Q,W_K,W_V\) 相乘得 Q、K、V；Q 与各 K 点积得分数，经 softmax 归一化后对 V 加权。任务不同，编码器与解码器的取舍也不同：

| 模式 | 典型代表 | 擅长 |
| ---- | -------- | ---- |
| **Encoder-Only** | BERT | 双向上下文理解、分类、抽取 |
| **Decoder-Only** | GPT 系列、LLaMA | 自回归生成、对话、续写 |
| **Encoder–Decoder** | 原版 Transformer、T5、部分 GLM | 翻译、摘要等明确的序列到序列 |

**所以 · 边界在哪：** 计算从「逐步隐状态」换成「全局可并行的注意力」。规模化第一次在工程上变得顺理成章——帷幕由此拉开。

---

## 7. 2018—2019：预训练范式立住

### 7.1 GPT-1 与 BERT

2018 年 6 月 11 日，OpenAI 发布 Decoder-Only 的 **GPT-1**（Generative Pre-trained Transformer）：约 **1.17 亿**参数（常写作 110M / 117M），在 BooksCorpus 等数据上先做生成式语言建模，再对下游任务微调，证明 NLP 上「预训练 + 微调」有效。[^gpt1]

同年 10 月，Google 发布 Encoder-Only 的 **BERT**：大号约 **3.4 亿**参数，双向掩码语言建模；在多项理解基准上表现突出，迅速进入搜索与工业检索栈。[^bert]

### 7.2 GPT-2 到 T5、ALBERT

2019 年 2 月，OpenAI 发布 **GPT-2**（约 **15 亿**参数）：预训练数据含 Reddit 上高 karma 外链文章等，体量约四十 GB 量级；强调**零样本（Zero-Shot）**——尽量不靠微调、直接靠提示完成任务。因担心滥用，最初未完整开源权重，舆论戏称 「Close AI」。[^gpt2]

7 月，Facebook AI 的 **RoBERTa** 用更大数据、更长训练与不同超参复现并强化 BERT，并取消下一句预测（NSP）。10 月，Hugging Face 的 **DistilBERT** 用知识蒸馏压缩 BERT。同期 Google 的 **T5**（Text-to-Text Transfer Transformer，最大约 **11B**）把翻译、摘要、分类、问答等统一成「文本进、文本出」：

| 字母 | 含义 |
| ---- | ---- |
| T1 Text | 输入是文本 |
| T2 To | 转化过程 |
| T3 Text | 输出是文本 |
| T4 Transfer | 迁移学习 |
| T5 Transformer | 架构底座 |

12 月，Google 与丰田工业大学芝加哥分校等推出 **ALBERT**：参数共享与嵌入因式分解，降低显存与计算。[^albert]

**所以 · 边界在哪：** 「先通吃语料，再适配任务」成为默认语法。编码器路线擅长理解；解码器路线擅长生成——后一条在规模上来之后，逐渐主导通用助手形态。

---

## 8. 2020—2022：规模、开源与检索增强

### 8.1 Meena、GPT-3 与独家授权

2020 年，Google 发布对话模型 **Meena**（约 **26 亿**参数，训练数据约 341 GB 量级），容量约为 GPT-2 的 1.7 倍。[^meena]

同年 6 月，OpenAI 发布 **GPT-3**：参数约 **1750 亿（175B）**；论文写明在约 **3000 亿 token** 上训练。过滤前 Common Crawl 压缩文本曾约 **45TB**，过滤后约 570GB——不宜把「45TB」直接写成「训练数据量」。[^gpt3] 在提示里给少量示例的**少样本（Few-Shot）**能力被系统展示；许多任务不再必须先做有监督微调。微软于 2020 年 9 月 22 日宣布获得 GPT-3 的独家授权接入。[^gpt3-ms]

缩放律研究（Kaplan 等，2020）把「做大」部分地写成可讨论的工程曲线；Chinchilla（2022）又修正了算力最优下的数据—参数配比——细节见 [12](./12-artificial-intelligence-chronicle.md)。[^scaling]

### 8.2 DeBERTa、LaMDA、RAG 与开源节点

2021 年 1 月，微软 **DeBERTa** 以解耦注意力等改进 BERT 一脉。4 月 28 日起，Hugging Face 等推动 **BigScience** 协作；5 月 18 日，Google 将 Meena 路线更名为 **LaMDA**（Language Model for Dialogue Applications）。

**RAG（Retrieval-Augmented Generation）** 的关键论文由 Lewis 等于 **2020** 年提出：参数记忆（生成模型）+ 非参数记忆（可检索文档索引）。[^rag] 大模型会「一本正经地胡说八道」——行业称**幻觉**；RAG 把检索到的片段与提示一并送入模型，是落地时最常用的缓解之一。不做全量微调时，外接领域知识库、先检索再生成，是同一结构的工程形态。

2022 年中，开源与生成侧连出三记：7 月 12 日 **BLOOM**（176B）在法国 Jean Zay 超算上发布，覆盖 46 种自然语言与 13 种编程语言，中文语料占比公开材料中约百分之十几量级；团队对比架构后，**causal Decoder-Only** 表现最佳，强化了此后 SOTA 路线的选择。[^bloom] 8 月 22 日，Stability AI、CompVis、Runway 等推出 **Stable Diffusion**，文生图门槛骤降，AIGC（AI Generated Content）进入大众创作。10 月前后见 **BLOOMZ** 等指令微调变体；11 月 Meta 的 **Galactica** 因不准确与争议很快下线——开放域事实性不会随参数自动解决。对话框破圈留给下一节。

**所以 · 边界在哪：** 规模证明「提示即可用」；开源证明「不止一家能训」；RAG 证明「不会的可以去查」。缺的是一个普通人每天都愿意打开的入口。

---

## 9. 2022：ChatGPT 破圈

2022 年 11 月 30 日，OpenAI 面向消费者推出基于 **GPT-3.5** 系列、并对齐对话的 **ChatGPT**。[^chatgpt] 上线约五天注册用户过百万；对话框把已有的续写与指令跟随能力交给非技术用户。宜分开两件事：**模型能力**（GPT-3 已展示少样本）与 **产品形态**（对话 + 对齐后的可用行为）。技术史上的破圈，常常是后者。

产业侧，Google 内部出现对搜索被分流的忧虑，并加速对话产品；全球 NLP / 应用团队迅速进入「大模型 +」试点。结构事实是：**会提示、会检索、会评测**成为新的工程常识——舆论里的饭碗叙事，不必写成史实判断。

对齐技术上，**InstructGPT**（2022 年）与 RLHF（来自人类反馈的强化学习）让模型更听指令、少有害输出；ChatGPT 是这一脉的消费级界面。[^instructgpt]

**所以 · 边界在哪：** 大模型从基准榜单走进浏览器标签页。NLP 的默认交付物，从「一个任务一个模型」变成「一个通用生成器 + 一堆应用壳」。

---

## 10. 2023—2024：多模态、开源潮与推理模型

### 10.1 2023：GPT-4、Claude 与开源底座

2023 年 2 月 6 日，Google 推出基于 LaMDA 的 **Bard**。同期 Meta 发布 **LLaMA**（7B / 13B / 30B / 65B 等，权重申请制开放），Decoder-Only 路线在全球被广泛二次训练——效果参差，但「可分叉的底座」改变了研究与创业的成本结构。[^llama]

3 月 14 日，OpenAI 发布 **GPT-4**：图像理解进入公众演示；准确率与可对齐性显著提升。参数量**未官方公布**；外传约 1.76 万亿等数字保持传闻地位。[^param-rumor] 同期 Anthropic 发布以克劳德·香农为灵感命名的 **Claude**；智谱 AI 发布 **ChatGLM**（早期公开版本约 62 亿参数量级，可在消费级显卡试验部署）。

4 月，Arthur Mensch、Guillaume Lample、Timothée Lacroix 成立 **Mistral AI**；搜狗创始人王小川成立**百川智能**。6–8 月，百川 **Baichuan**、智谱 **ChatGLM2**、阿里 **Qwen**、字节**豆包**、智谱**清言**等密集出现。7 月 18 日 Meta 与微软合作发布 **Llama 2**（7B / 13B / 70B，上下文至 4096）；训练语料中中文比例极低（公开讨论常引用约 0.13% 量级），中文能力多靠继续预训练或微调补齐。[^llama2] 同月 Anthropic 发布 **Claude 2**（支持文档上传）。9 月 IBM **Granite**；9 月 27 日 Mistral **Mistral-7B**（Apache 许可）。11 月百川 **Baichuan 2**；12 月 6 日 Google **Gemini**（次年与 Bard / Duet 等品牌整合）；12 月 11 日 Mistral **Mixtral 8x7B**（混合专家，总参数约 467 亿量级、稀疏激活）；微软 **Phi-2**（2.7B）展示小模型质量路线。

### 10.2 2024：旗舰迭代与推理分支

2024 年 2 月，阿里 **Qwen 1.5** 多尺度版本；OpenAI 公布视频模型 **Sora**（逐步开放）。3 月 Anthropic **Claude 3**（Haiku / Sonnet / Opus）。4 月 Mistral **Mixtral 8x22B**；Meta **Llama 3**（8B / 70B）。5 月 13 日 OpenAI **GPT-4o**（omni）：更自然的实时多模态对话。6 月快手**可灵**、阿里 **Qwen2**、Apple 宣布接入 ChatGPT 能力；Anthropic **Claude 3.5 Sonnet**。7 月 **GPT-4o mini**；9 月 **o1-preview / o1-mini**——用强化学习强化多步推理，面向科学、策略与编码等慢思考任务。[^o1] 同月阿里 **Qwen 2.5** 与通义万相视频；10 月 Black Forest Labs **Flux**、OpenAI **GPT-4o with canvas** 等把生成嵌进创作工作区。

主流大模型——含多模态与大量视觉骨干——在表示学习上多向 Transformer 一族靠拢。行业进入融合与基础设施化：模型是底座，检索、工具、评测与权限是房子。

**所以 · 边界在哪：** 竞争维度从「谁参数更大」扩展到**推理成本、上下文、工具使用、开放权重与多模态时延**。Decoder-Only 仍是通用助手主航道；MoE、小模型与推理模型是同一河床上的分汊。

---

## 11. 尾声：工具、仿生与尚未开始的旅程

回到开头。在人类工具史里，「仿生」反复出现：每个时代都发明替代或模仿生命功能的装置。现阶段的大语言模型，本质仍是**复杂的条件概率机器**——一个日益顺手的工具，提供草稿、翻译、代码与检索中介，而不是已盖章的智慧主体。

参数规模会继续涨，但力量不必只体现在「答对一题」，更体现在对多维信息的整合、工具循环与可验证推理。思维链（chain-of-thought）与慢推理模型让「先想步骤」变得可产品化；RAG 与引用让「先查再写」成为默认礼貌。或许未来某一天，讨论焦点会从规模与架构，转向人类智慧与机器智能如何在交汇处分工——那是通用人工智能（AGI）的叙事，不是今天这篇编年的结论。

科幻里的场景会一点点变成工程里程碑；就语言建模这条河而言，Transformer 之后的旅程，才刚刚把船推到中流。

---

## 12. 本章要点

1. **核心机制：** LLM 主要做条件续写（下一 token 预测）；流畅 ≠ 真知。  
2. **前史：** 马尔可夫链、McCulloch–Pitts、图灵测试与达特茅斯立科，给出序列概率与「语言考智能」的题面。  
3. **循环时代：** 反向传播、RNN / LSTM / BRNN 解决「能训序列」；长程依赖与并行仍是债。  
4. **统计河床：** IBM 对齐与 SMT / Watson 证明工程可行；2003 神经语言模型是火种，NMT 主流在 2014–2016。  
5. **注意力 → Transformer（2017）：** QKV 动态聚合；可并行，规模化成为工程默认。  
6. **预训练范式：** GPT 走 Decoder-Only 生成；BERT 走 Encoder-Only 理解；T5 统一文本到文本。  
7. **规模与补丁：** GPT-3 少样本；缩放律指导做大；RAG（2020）与对齐（InstructGPT / RLHF）按住幻觉与行为。  
8. **破圈与之后：** ChatGPT（2022-11-30）是产品事件；2023–2024 是开源底座、多模态与推理模型并行的军备年。

---

## 13. 参考文献

[^param-rumor]: GPT-4 官方技术报告未公布总参数量。「约 1.76 万亿」等数字来自第三方推测与媒体转述，正文仅作传闻标注，不采信为事实。

[^brain-scale]: 人脑神经元数量通行估计约 \(8.6\times10^{10}\)（Azevedo 等，2009 等综述口径）。突触总数估计跨度大，正文取「约 100 万亿」为数量级教学数，非精确解剖审计。

[^cajal]: Santiago Ramón y Cajal 小脑皮质与神经元学说相关组织学工作（20 世纪初）。「循环半圆」为解剖学描述的译写，不是现代 RNN 的直接发明声明。

[^markov]: A. A. Markov, 1906 年起关于链式依赖的概率工作；为 n-gram 与 SLM 提供古典概率语言。

[^lorente]: Rafael Lorente de Nó 关于神经元回路与前庭眼反射的组织学 / 生理学论述（1930 年代）。属神经科学史节点，与后来工程 RNN 是启发关系。

[^mcculloch-pitts]: W. S. McCulloch & W. Pitts, “A Logical Calculus of the Ideas Immanent in Nervous Activity,” *Bulletin of Mathematical Biophysics*, 1943.

[^turing-1950]: A. M. Turing, “Computing Machinery and Intelligence,” *Mind*, 1950.

[^dartmouth]: J. McCarthy et al., Dartmouth Summer Research Project on Artificial Intelligence（提案 1955，会议 1956）。

[^rosenblatt]: F. Rosenblatt, *Principles of Neurodynamics*（1961）及感知器相关工作。荣誉头衔从宽不从窄：记开拓者，不与后来的「深度学习三巨头」叙事抢冠名。

[^hopfield]: J. J. Hopfield, 1982, 内容可寻址记忆网络。诺贝尔物理学奖 2024 授奖理由见诺贝尔委员会公告。

[^backprop-1986]: D. E. Rumelhart, G. E. Hinton, R. J. Williams, “Learning representations by back-propagating errors,” *Nature*, 1986.

[^allen-1987]: R. B. Allen, 1987 前后关于前馈网络做句级翻译的实验报告；正文取「固定长度槽位」的结构教训。

[^bptt]: R. J. Williams & D. Zipser, 关于 BPTT 与实时循环学习的工作（1989 前后）。

[^lstm]: S. Hochreiter & J. Schmidhuber, “Long Short-Term Memory,” *Neural Computation*, 1997（思想与技术报告更早至 1995–1997）。

[^brnn]: M. Schuster & K. K. Paliwal, “Bidirectional Recurrent Neural Networks,” *IEEE Trans. Signal Processing*, 1997.

[^ibm-align]: IBM Model 1–5 等对齐模型（Brown 等，1990 年代）；SMT 经典教材与论文中的标准起点。

[^watson]: IBM Watson / *Jeopardy!*（2011）。系统含检索、候选生成与置信度排序，不等同于 2020 年代 Decoder-Only LLM。

[^bengio-2003]: Y. Bengio et al., “A Neural Probabilistic Language Model,” *JMLR*, 2003.

[^word2vec]: T. Mikolov et al., Word2Vec 相关工作（2013）。

[^gru]: K. Cho et al., “Learning Phrase Representations using RNN Encoder–Decoder…,” 2014.

[^attention-2014]: D. Bahdanau, K. Cho, Y. Bengio, “Neural Machine Translation by Jointly Learning to Align and Translate,” 2014/2015；另见 Sutskever et al. Seq2Seq（2014）。

[^openai-2015]: OpenAI 成立公告与创始名单（2015-12）。人员列表以当时公开为准。

[^gnmt]: Google Neural Machine Translation 生产切换相关技术博客与论文（2016）。

[^hf-2016]: Hugging Face 公司沿革：早期聊天应用 → Transformers 库与模型托管枢纽。

[^transformer]: A. Vaswani et al., “Attention Is All You Need,” NeurIPS 2017.

[^gpt1]: A. Radford et al., “Improving Language Understanding by Generative Pre-Training,” OpenAI, 2018.

[^bert]: J. Devlin et al., “BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding,” 2018/2019.

[^gpt2]: A. Radford et al., “Language Models are Unsupervised Multitask Learners,” OpenAI, 2019.

[^albert]: ALBERT、RoBERTa、DistilBERT、T5（Raffel et al., 2019/2020）均为预训练族谱上的公开节点；参数量取各论文最大常用配置。

[^meena]: Google Meena 技术报告（2020）；后续对话产品线演进为 LaMDA / Bard / Gemini，品牌会换，底座叙事连续。

[^gpt3]: T. Brown et al., “Language Models are Few-Shot Learners,” 2020。训练 token 约 300B；Common Crawl 过滤前约 45TB compressed plaintext 为语料来源描述，不是「模型吃进 45TB」的同义反复。

[^gpt3-ms]: Microsoft–OpenAI GPT-3 独家授权相关公告（2020-09）。

[^scaling]: Kaplan et al., Scaling Laws for Neural Language Models（2020）；Hoffmann et al., Chinchilla（2022）。

[^bloom]: BigScience / Hugging Face BLOOM 发布（2022-07-12）；176B；Jean Zay / GENCI；架构消融支持 causal decoder-only。中文占比以 ROOTS / 项目文档为准，正文取约百分之十几量级。

[^rag]: P. Lewis et al., “Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks,” NeurIPS 2020.

[^chatgpt]: OpenAI, “Introducing ChatGPT,” 2022-11-30；微调自 GPT-3.5 系列对话模型。

[^instructgpt]: Ouyang et al., InstructGPT / RLHF（2022）。与 ChatGPT 产品发布是同一技术族谱上的先后环。

[^llama]: Meta LLaMA 论文与权重申请制发布（2023-02）。

[^llama2]: Meta Llama 2（2023-07）。中文语料比例见社区对公开数据配比的讨论与二次预训练实践；正文取「极低、需补齐」结构。

[^o1]: OpenAI o1-preview / o1-mini（2024-09）：强化多步推理；具体内部链不必当作已公开架构说明书。

**声明：** 正文是语言建模与 LLM 产品化的编年整理，不是投资建议或能力担保。参数、许可证、基准分数与产品名称会迭代；冲突时以一手论文、模型卡与官方博客为准。与 [12](./12-artificial-intelligence-chronicle.md) 重叠处，本文更细，阶段判断以 12 的六种机制表为准。
