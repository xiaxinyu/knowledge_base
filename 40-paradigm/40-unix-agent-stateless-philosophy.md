# 从 Unix 到 Agent：Claude Code 的无状态设计哲学

> 互联网早期的工程传统，塑造了一套可复用的「构建工具的方法论」：**工具应当职责明确、可组合、可观察**。AI Agent（以 Claude Code 为代表）并非脱离历史的全新范式，而是在这些传统之上，把「文本输入」升级为「意图输入」，把「输出文本」升级为「执行动作」。
>
> 本文沿 Unix / CLI 与 REPL 的演化、状态与无状态的思想脉络、设计收益与现实权衡，再到从聊天机器人到 Agent 的跃迁展开；并以 Claude Code 选择 grep 而非持久向量索引为线索，回扣开篇争议。文中关键判断尽量对照可核验来源，文末附参考文献。

无状态不是「不保存任何数据」，而是**审慎选择**状态的存放位置与生命周期；错误的不是状态本身，而是**失控**的状态。每个计算时代都在重新发现同一个道理：**有时候，遗忘比记忆更强大。**

先给一个直接答案：

> **Agentic search（glob / grep）与向量索引并非简单的「先进 / 倒退」，而是场景、确定性、隐私、延迟与 token 成本之间的不同取舍。**

---

## 目录

1. [引言：一个看似倒退的选择](#1-引言一个看似倒退的选择)
2. [Unix 哲学与 CLI 的传统](#2-unix-哲学与-cli-的传统)
3. [REPL 的演化史](#3-repl-的演化史)
4. [理解状态的本质](#4-理解状态的本质)
5. [无状态思想的历史脉络](#5-无状态思想的历史脉络)
6. [无状态设计的优势](#6-无状态设计的优势)
7. [现实的权衡](#7-现实的权衡)
8. [从聊天机器人到 Agent](#8-从聊天机器人到-agent)
9. [AI 时代的新思考](#9-ai-时代的新思考)
10. [附录 A：RAG（向量索引）](#附录-arag向量索引)
11. [参考文献](#参考文献)

---

## 1. 引言：一个看似倒退的选择

最近，Claude Code 的技术选择引发了不少讨论。

向量数据库厂商 Zilliz / Milvus 在技术博客中批评道：Claude Code 与 Gemini 一类工具选择了不建代码索引的路线，其 grep-only 式检索「会烧掉太多 tokens」[^milvus-grep]。

在 Hacker News 的讨论中，也有开发者质疑：Claude 用 grep，Cursor 用向量搜索——这是不是一种技术倒退？[^hn-rag]

当主流 AI 编程助手纷纷采用向量索引实现语义搜索时，Claude Code 却选择了 grep——这个诞生于 1973 年前后的命令行工具家族。它不建立持久的代码向量索引，不预计算「编码意图」，每次搜索都是对当前工作区的实时查询。

根据 Latent Space 对 Claude Code 团队的访谈《Claude Code: Anthropic's Agent in Your Terminal》，早期版本确实尝试过 RAG（用 Voyage 等方案对代码库建索引）。团队测试了多种检索方案后，最终选择了 **agentic search**——让 Agent 按需调用 glob、grep 等常规代码搜索工具。Boris Cherny 表示，这种方式「大幅优于」其他方案；代价则是更高的延迟与 token 消耗，换来的是无需维护索引、以及更少的安全与隐私负担[^latent-space]。

这种取舍并非一时兴起，而是一条贯穿计算机科学数十年的设计哲学：从 Unix 管道到 REST API，从 MapReduce 到 Serverless，**无状态（Stateless）设计**一次次证明了它的价值——通过放弃复杂的状态管理，系统获得更好的可组合性、可靠性和可扩展性。

本文将探讨「无状态」这个设计理念——它不是简单的「不保存数据」，而是关于如何明智地选择在哪里、以什么方式管理必要的状态。理解了这个设计哲学，就能更清楚地看到 Claude Code 选择背后的深层逻辑。

每个计算时代都在重新发现同一个道理：**有时候，遗忘比记忆更强大。**

---

## 2. Unix 哲学与 CLI 的传统

> This is the Unix philosophy:  
> Write programs that do one thing and do it well.  
> Write programs to work together.  
> Write programs to handle text streams, because that is a universal interface.  
> — Doug McIlroy（经 Peter Salus 整理转述）[^mcilroy-unix]

### 2.1 Unix 哲学：单一职责、组合性、透明性

1969 年，Ken Thompson 和 Dennis Ritchie 在贝尔实验室创造了 Unix。它不只是一个操作系统，更是一套影响深远的设计哲学。McIlroy 在 1978 年的 Bell System Technical Journal 前言中，已把「每个程序只做好一件事」「程序输出应能成为另一未知程序的输入」等准则写进了 Unix 共同体的共同语言[^bstj-1978]。

可概括为三条原则：

1. **单一职责**：每个程序只做一件事，并把它做好
2. **组合性**：程序之间通过标准接口（文本流）协作
3. **透明性**：程序的行为可预测、可观察

这些原则在今天的 Claude Code 中依然清晰可见。

### 2.2 管道：最早的「工具调用」

Unix 管道（`|`）把复杂任务拆成可组合步骤：前一个程序的输出成为后一个程序的输入。

```bash
history | awk '{print $2}' | sort | uniq -c | sort -rn | head -10
```

执行链路如下：

1. `history` 输出命令历史记录
2. `awk` 提取第二列（命令名）
3. `sort` 排序
4. `uniq -c` 统计重复次数
5. `sort -rn` 按数字倒序排列
6. `head -10` 取前 10 条

每个程序只做一件事，通过「管道」组合完成目标。Claude Code 的工具链也具有相同结构：用户意图会被拆解为一串小工具的调用与组合。

### 2.3 文件系统：统一的抽象

Unix 的另一个伟大设计是「一切皆文件」。网络连接、设备、进程信息等，都通过文件系统接口访问。统一抽象的价值在于：用一致的接口覆盖多种能力，从而降低认知负担。

Claude Code 继承了这个思想：无论是读取本地文件、调用 MCP 服务器，还是执行 shell 命令，都通过同一套工具调用机制完成。

### 2.4 标准输入输出：接口即契约

Unix 程序通过 `stdin` / `stdout` / `stderr` 通信，其价值在于解耦与可组合：程序不需要关心数据来源与去向，只需遵守输入输出契约。

Agent 的工具系统同样强调「契约」，以结构化输入与统一输出保证可组合性：

```ts
type Tool = {
  name: string
  description: string
  inputSchema: ToolInputJSONSchema
  execute(input: I, context: ToolUseContext): Promise<ToolResult<O>>
}
```

### 2.5 CLI 的黄金时代：为什么 CLI 适合 Agent

从 1970 年代到 2000 年代，CLI 工具长期统治软件世界（vim、emacs、make、grep、sed、awk 等）。它们之所以经久不衰，是因为 CLI 天生具备几类对 Agent 友好的工程属性：

- **可脚本化**：天然可被编排、可自动化调用
- **可组合**：管道与重定向让工具之间具备低摩擦协作方式
- **低开销**：不依赖 GUI 生态，部署与运行门槛低
- **可远程**：通过 SSH 等方式在任何地方工作

Claude Code 选择 CLI 作为主要界面，不是因为 GUI 不好，而是因为这些特性对「可执行的智能体」至关重要。官方文档也强调其可组合性：可把日志管道进 Claude，可在 CI 中调用，可与其他 Unix 工具串联[^claude-code-docs]。

### 2.6 make：最早的「任务编排」

make 用依赖图描述任务，只重新构建需要更新的部分。这种「依赖 + 状态 + 增量执行」的思想，也出现在现代 Agent 的任务编排之中。

```makefile
app: main.o utils.o
	gcc -o app main.o utils.o

main.o: main.c
	gcc -c main.c

utils.o: utils.c
	gcc -c utils.c
```

### 2.7 从 CLI 到 TUI：终端的进化

随着终端能力增强，出现了 TUI（Terminal User Interface）：在终端中渲染更复杂的交互界面。Claude Code 使用 Ink（React for CLI）构建 TUI，带来组件化、声明式渲染与状态驱动的开发体验。

```tsx
function App() {
  return (
    <Box flexDirection="column">
      <Header />
      <MessageList messages={messages} />
      <InputBox onSubmit={handleSubmit} />
    </Box>
  )
}
```

### 2.8 Unix 哲学在 Claude Code 中的体现

| Unix 原则 | Claude Code 的实现 |
| ------ | ------ |
| 单一职责 | 每个工具只做一件事（例如搜索、读文件、编辑、执行命令） |
| 组合性 | 工具可以链式调用，前一个输出可作为下一个输入 |
| 透明性 | 工具调用过程可展示给用户，支持审查与中断 |
| 文本流 | 工具通过结构化文本（如 JSON）进行数据交换 |
| 可脚本化 | 支持非交互 / print 模式，可被脚本或流水线调用 |
| 管道 | 支持 `stdin` 输入（例如 `cat file \| claude -p "分析这个"`） |

### 2.9 小结

Unix 的三条原则（单一职责、组合性、透明性）本质上是在约束「工具应该怎样被构建」。Claude Code 站在这套传统之上，把原则延伸到 AI Agent：用明确的工具边界保证可控，用组合能力放大效率，用透明可观察保障可靠性。

---

## 3. REPL 的演化史

REPL（Read–Eval–Print Loop）是最小化的交互式编程环境。它持续降低「从意图到执行」的门槛，而 Claude Code 将这条演化路径推进到了「行动可验证」的阶段。

### 3.1 什么是 REPL

REPL 是 Read–Eval–Print Loop 的缩写：

1. **Read**：读取用户输入
2. **Eval**：执行或求值
3. **Print**：打印结果
4. **Loop**：回到第一步

这是最简单的交互式编程环境。你输入一行代码，立刻看到结果。

```bash
$ python
>>> 1 + 1
2
>>> "hello".upper()
'HELLO'
>>> [x**2 for x in range(5)]
[0, 1, 4, 9, 16]
```

### 3.2 REPL 的起源：Lisp（1958）

REPL 的概念来自 Lisp。John McCarthy 于 1958 年提出 Lisp；交互式求值环境随后成为 Lisp 文化的核心部分。它打破了「编写—编译—运行」的传统循环，让程序员可以即时探索。

```lisp
> (+ 1 2)
3
> (defun square (x) (* x x))
SQUARE
> (square 5)
25
```

这个「即时反馈」的思想，是所有现代交互式工具的基础。

### 3.3 Shell：操作系统的 REPL

Unix Shell（sh、bash、zsh）把操作系统能力暴露为一套可交互、可编排的命令环境：

```bash
$ ls -la
$ cd src
$ grep -r "TODO" .
$ git status
```

Shell 的 REPL 循环：

- **Read**：读取命令行输入
- **Eval**：解析命令，fork 子进程执行
- **Print**：显示输出
- **Loop**：显示新的提示符

Shell 的伟大之处在于：通过一个简单的文本界面，把操作系统的能力暴露给了用户。

### 3.4 Node.js REPL：JavaScript 的即时环境

2009 年，Node.js 带来了服务端 JavaScript，也带来了 Node.js REPL：

```bash
$ node
> const arr = [1, 2, 3, 4, 5]
undefined
> arr.filter(x => x % 2 === 0)
[ 2, 4 ]
> arr.reduce((sum, x) => sum + x, 0)
15
```

Node.js REPL 的特点：支持多行输入、自动补全、历史记录，以及 `require` 模块。

### 3.5 IPython / Jupyter：科学计算的 REPL

2001 年，Fernando Pérez 创建了 IPython，后来演化为 Jupyter Notebook。Jupyter 把 REPL 的概念推向了新高度：

```text
# Cell 1
import pandas as pd
df = pd.read_csv('data.csv')
df.head()
# 立刻显示表格

# Cell 2
df.describe()
# 立刻显示统计信息

# Cell 3
df.plot(kind='bar')
# 立刻显示图表
```

Jupyter 的创新：

- **Cell 模型**：代码分块执行，每块有独立输出
- **富媒体输出**：不只是文本，还有图表、表格、HTML
- **持久状态**：变量在 cells 之间共享
- **叙事性**：代码和文档混合

许多现代 Agent（包括 Claude Code）支持编辑 Notebook，正是因为 Notebook 已成为数据科学工作流的核心载体。

### 3.6 ChatGPT：对话即 REPL

2022 年，ChatGPT 的出现把 REPL 的概念带到了自然语言层面：

```text
用户：帮我写一个快速排序
ChatGPT：[给出代码]
用户：改成支持自定义比较函数
ChatGPT：[修改代码]
用户：加上单元测试
ChatGPT：[添加测试]
```

这是一个新型的 REPL：

- **Read**：读取自然语言输入
- **Eval**：LLM 理解并生成响应
- **Print**：输出文本 / 代码
- **Loop**：继续对话

但纯聊天式 REPL 有一个根本限制：它只能生成文本，不能可靠地改变外部环境。

### 3.7 Claude Code：行动即 REPL

Claude Code 将自然语言 REPL 与工具执行结合，使「评估」不仅发生在模型内部，也发生在真实系统之中：

```text
用户：帮我找出项目里所有未使用的导入并删除
Claude Code：
→ Glob：找到相关源文件
→ Read：读取每个文件
→ 分析未使用的导入
→ Edit：删除未使用的导入
→ Bash：运行类型检查 / 测试验证
→ 完成，共修改若干文件
```

这个 REPL 的特点：

- **Read**：读取自然语言意图
- **Eval**：规划并执行工具调用链
- **Print**：显示执行过程和结果
- **Loop**：等待下一个指令

### 3.8 关键维度：从「计算」到「行动」

| 系统 | 输入 | 执行能力 | 状态持久 | 输出 | 可组合性 |
| ------ | ------ | ------ | ------ | ------ | ------ |
| Lisp REPL | 代码 | 计算 | 会话内 | 值 | 低 |
| Shell | 命令 | 系统操作 | 会话内 | 文本 | 高（管道） |
| Jupyter | 代码 | 计算 + 可视化 | Notebook 内 | 富媒体 | 中 |
| ChatGPT | 自然语言 | 文本生成 | 对话内 | 文本 | 低 |
| Claude Code | 自然语言 | 系统操作 + 计算 + 工具链 | 对话内 + 文件系统 | 文本 + 动作结果 | 高（工具链） |

### 3.9 Claude Code 的 REPL 设计

从产品行为观察，Claude Code 的核心循环可概括为：

```text
1. 显示提示符，等待用户输入
2. 解析输入（斜杠命令 or 自然语言）
3. 如果是斜杠命令，直接执行
4. 如果是自然语言，提交给模型 / 查询引擎
5. 流式获取响应
6. 响应中包含工具调用时，执行工具
7. 工具结果回填到对话，继续生成
8. 显示最终结果
9. 回到第 1 步
```

这个循环有几个关键设计：

- **流式显示**：响应是流式的，用户不必等待完整响应才看到内容
- **工具调用透明**：工具调用过程可展示，用户能看到 Agent 在做什么
- **可中断**：用户可随时中断当前操作
- **历史可恢复**：对话历史保存在本地，可恢复之前的会话

> 注：具体实现细节（内部模块划分、工具清单数量等）会随版本快速变化；本文关注的是可观察的产品行为与设计原则，而非某一版源码快照。

### 3.10 小结

REPL 的演化史是一部降低交互门槛的历史：

- **Lisp REPL**：让程序员可以即时探索代码
- **Shell**：让用户可以即时操作操作系统
- **Jupyter**：让数据科学家可以即时探索数据
- **ChatGPT**：让普通人可以用自然语言交互
- **Claude Code**：让开发者可以用自然语言操作代码库

每一步都在降低「从想法到执行」的摩擦。要理解它为什么有效，需要先理解「状态」是什么。

---

## 4. 理解状态的本质

### 4.1 什么是状态？

> **本节要解决的问题**：把「有状态 / 无状态」从口号还原为可操作的工程判断——`Output = f(Input)` 与 `Output = f(Input, History)` 分别对应何种系统行为。

想象两种不同的计算方式：

- **有状态的计数器**：就像一个记账本，每次调用都会在之前的基础上累加。第一次返回 1，第二次返回 2……它「记住」了之前发生的一切，每次的输出都依赖于历史。
- **无状态的加法器**：就像计算器的加法功能，输入 2 和 3，无论何时、调用多少次，结果永远是 5。它不知道也不关心之前发生了什么，每次都是全新的计算。

用数学语言表达：

```text
无状态：Output = f(Input)
有状态：Output = f(Input, History)
```

### 4.2 生活中的状态

- **银行账户是有状态的**。余额是所有历史交易的累积结果。银行必须记住每一笔存取款。
- **汇率转换是无状态的**。100 美元换人民币，只需要当前汇率，不需要知道你上次换了多少。
- **对话是有状态的**。当朋友说「还记得昨天那件事吗？」，需要共同记忆才能继续。
- **单句翻译可以近似无状态**。把 "Hello" 译成「你好」，通常不需要依赖更早的翻译历史（跨句指代例外）。

这个区别看似简单，却深刻影响着系统设计。理解了这一点，就能沿着历史看到「无状态」如何被一次次发明与复用。

### 4.3 小结

「无状态」描述的不是「系统里没有数据」，而是：**处理一次请求时，是否必须依赖服务器侧保存的、跨请求可变的历史**。把状态显式化、外置化（客户端携带、数据库持久化、事件日志回放），往往比把状态藏在进程内存里更可控。

---

## 5. 无状态思想的历史脉络

### 5.1 数学起源：纯函数的优雅

无状态的思想并非始于计算机。数学函数本身就是无状态的：`f(x) = x²`，无论何时计算 `f(3)`，结果都是 9。这种确定性与可预测性，成为后来所有无状态设计的理论基础。

### 5.2 Unix 革命：管道的哲学（1970s）

真正将无状态思想带入大规模工程实践的，是 Unix 的创造者们。

Doug McIlroy 推动了管道（pipe）的落地。他用过一个生动比喻：需要某种方式把程序连接起来，就像花园里的水管——当需要以另一种方式处理数据时，只需拧上另一段管子。

在 Unix 中，复杂任务可以通过管道符号（`|`）将多个简单工具串联完成：

```bash
cat file.txt | grep "error" | sort | uniq -c | head -10
```

每个工具都尽量保持「无会话状态」：筛选工具不知道数据来自文件读取，排序工具不关心后续会被统计。每个工具只做一件事。

这种设计带来了组合威力——少量正交工具可以形成大量有用流水线。如果这些工具彼此强依赖内部会话状态，大部分组合都会失效。这就是无状态设计的魔力之一：**通过放弃记忆，获得组合可能。**

### 5.3 函数式编程的批判（1977）

1977 年，John Backus 在图灵奖演讲《Can Programming Be Liberated from the von Neumann Style?》中，批判了主流命令式编程对可变状态的依赖[^backus-1977]：

```python
# 冯·诺依曼风格（有状态）
sum = 0
for i in array:
    sum = sum + i  # 不断修改状态

# 函数式风格（无状态）
sum = reduce(add, array)  # 纯函数组合
```

传统求和像记账：变量不断被修改，每一步依赖前一步。函数式方式把求和看作纯数学运算，数据在函数间流动，而不是被就地改写。Backus 认为，不受约束的状态修改是程序复杂性的重要来源——这一观点深刻影响了后来的语言与系统设计。

### 5.4 REST 与网络架构：无状态作为扩展前提（2000）

2000 年，Roy Fielding 在博士论文中提出 REST 架构风格，将 **无状态（Stateless）** 列为核心约束之一：通信必须是无状态的，每个请求包含理解该请求所需的全部信息，不能依赖服务器保存的会话上下文[^fielding-rest]。

对比两种设计：

- **有状态会话**：服务器记住用户会话；请求落到另一台机器就可能失效，需要会话同步或粘性路由。
- **无状态请求**：每个请求携带必要凭证与上下文（如 token）；任意副本都能处理，横向扩展更简单。

这就是 REST 能支撑互联网规模应用的关键原因之一——以无状态通信为前提，降低横向扩容的复杂度。

### 5.5 Serverless 与 Lambda：强制无状态的编程模型（2014）

2014 年，AWS Lambda 普及了一种激进承诺：开发者应当假设「每次函数调用都可能在全新环境中运行」。

这个编程模型很纯粹：你不应依赖全局变量、临时文件或连接在调用之间必然存活。但实现层往往会复用执行环境以提升性能——这是「约束编程模型、优化运行时」的经典交易。

为什么这么设计？传统长驻服务器的痛点包括：

- 负载突增时扩容慢
- 闲置时仍有固定成本
- 单机崩溃带走进程内状态

无状态函数让扩缩容与故障隔离更容易。但「纯粹无状态」在现实中总要妥协：Lambda 提供临时磁盘、允许（但不保证）环境复用；Cloudflare Workers 等平台用更强约束换取边缘部署；Azure Durable Functions 则明确承认某些工作流需要状态编排。

**Serverless 的「无状态」不是技术洁癖，而是一种交易——用编程模型的约束，换取运维简化与成本弹性。**

### 5.6 小结

从纯函数、Unix 管道、函数式批判，到 REST 与 Serverless，历史反复证明：把可变状态从「默认」变成「显式决策」，能显著提升系统的可组合性与可扩展性。下一章具体看这些优势如何兑现。

---

## 6. 无状态设计的优势

### 6.1 可组合性：乐高积木 vs 精密手表

无状态组件就像乐高积木，可以自由组合来解决不同问题：

```bash
# 今天：找出错误日志中的 IP
cat app.log | grep ERROR | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | sort -u

# 明天：统计每个 IP 的错误次数
cat app.log | grep ERROR | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | sort | uniq -c

# 后天：找出错误最多的前 10 个 IP
cat app.log | grep ERROR | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | sort | uniq -c | sort -rn | head -10
```

每个新需求都是在已有组合上的微调。我们不需要重写整个程序，只需要替换或添加几个模块。每个工具都不知道会被如何组合，所以能被任意组合。

相比之下，强耦合的有状态系统更像精密手表——齿轮彼此咬合，牵一发而动全身。Unix 哲学历经数十年依然生命力旺盛，正是因为通过保持组件独立性，获得了应对未知需求的灵活性。

### 6.2 并行的自然性：无冲突的世界

无状态计算天然适合并行：

- 搜索文件 A 不会影响搜索文件 B
- 没有共享可变状态需要加锁
- 没有因状态竞争导致的隐蔽时序 bug
- 多核 / 多机可以真正独立工作

如果工具必须维护全局可变状态（共享进度、强制全局排序、跨任务写同一缓存），并行化会迅速变得复杂。这也是 MapReduce、Spark 等大数据框架强调无共享可变状态计算模型的原因之一——不是「不要状态」，而是「不要让状态阻碍规模化」。

### 6.3 简单性：没有沉重的生命周期管理

有状态服务常常背负更重的生命周期：

**启动**：初始化连接池、加载配置、恢复运行态、重建缓存……  
**关闭**：落盘、排空请求、优雅断连、通知下游……  
**崩溃恢复**：一致性检查、事务恢复、索引重建……

无状态服务更接近「计算器」：插电可用，断电即停，重启即可恢复服务能力。复杂生命周期管理往往是 bug 温床——泄漏、死锁、恢复逻辑漏洞等。

### 6.4 可测试性：确定性的力量

测试无状态函数更接近测试数学公式：相同输入应得到相同输出。不需要精心准备共享环境，也不容易被「上一个测试留下的脏状态」污染。

测试有状态系统则更像化学实验：环境污染、依赖模拟、时序与并发干扰都会让失败原因变模糊。无状态带来的确定性，让「失败更可能指向逻辑本身」。

### 6.5 小结

可组合、可并行、生命周期更简单、更易测——这些优势不是口号，而是「减少跨请求隐式依赖」的直接后果。但真实系统很少能纯无状态，下一章讨论何时必须保留状态、以及如何混合。

---

## 7. 现实的权衡

### 7.1 什么时候需要状态

有些场景，状态不是可选项，而是必需品：

- **游戏世界需要持续性**：装备、等级、地图进度是体验核心。
- **用户界面需要响应性**：购物车、表单草稿、滚动位置虽未必永久保存，但会话期内必须保持。
- **资源管理需要经济性**：连接池、线程池用少量状态换取巨大性能收益。

### 7.2 如何选择？一个简单的判断标准

> **「如果系统崩溃重启，用户能接受从零开始吗？」**

| 场景 | 能否接受从零开始 | 倾向 |
| ------ | ------ | ------ |
| 编译器崩溃 | 通常可以重新编译 | 无状态 |
| RPG 存档丢失 | 通常不可接受 | 有状态 |
| 一次代码搜索失败 | 通常可以重试 | 无状态 |
| 购物车清空 | 通常很恼人 | 有状态 |

这个问题能帮你快速定位系统的本质需求。

### 7.3 混合策略：现实世界的智慧

中文语境里的「无 / 有状态」容易被理解成布尔开关。英文的 *stateless / stateful* 更像程度词：强调依赖的轻重与记录方式的取舍，而非「完全没有 / 完全存在」。

真实系统几乎总是混合的。最常见的模式是：

**无状态计算 + 有状态存储**

- 无状态的 API 服务器 → 有状态的数据库
- 无状态的 Lambda 函数 → 有状态的 DynamoDB / 对象存储
- 无状态的容器 → 有状态的 Redis / 磁盘卷

**Event Sourcing：用不可变事件构建可变视图**

不直接存「当前余额」作为唯一真相，而是存所有导致状态变化的事件；当前状态是事件流的折叠结果。既保留历史，又能重建任意时刻的视图。

### 7.4 核心洞察

**选择无状态还是有状态，不是技术信仰，而是工程权衡。** 无状态不是目的，而是手段——它帮助我们构建更简单、更可靠、更可扩展的系统。

**状态并不是坏的，无管理的状态才是问题的根源。** 最好的设计不是完全无状态，而是在正确的地方、以正确的方式管理必要的状态。

---

## 8. 从聊天机器人到 Agent

> An agent is anything that can be viewed as perceiving its environment through sensors and acting upon that environment through actuators.  
> —— Russell & Norvig，《Artificial Intelligence: A Modern Approach》[^aima]

### 8.1 聊天机器人的本质局限

2022 年 ChatGPT 发布后，全世界都在讨论 AI。但早期主流产品形态仍是：

```text
输入文本 → [LLM] → 输出文本
```

这个模式有一个根本限制：它只能生成文本，不能可靠地改变世界状态。你可以让它写代码，但它不能替你运行；可以让它分析 bug，但它不能替你改文件；可以让它设计系统，但它不能替你部署。

**文本生成 ≠ 行动执行。**

### 8.2 工具调用：打破文本的牢笼

2023 年 6 月，OpenAI 为 Chat Completions API 引入了 Function Calling[^openai-fc]。Anthropic 则在 Claude 上推进 Tool Use（公开 beta 约在 2024 年 4 月，同年 5 月 GA）[^anthropic-tools]。名称不同，核心机制相近：模型输出结构化「调用意图」，宿主执行工具，再把结果回填给模型。

```text
用户：今天北京天气怎么样？
LLM 决策：需要调用天气 API
→ 生成工具调用：get_weather(city="北京")
→ 系统执行工具，返回：{"temp": 15, "weather": "晴"}
→ LLM 基于结果生成回答：今天北京天气晴，气温 15 度。
```

这个机制让 LLM 从「只能说」变成「能做」。

### 8.3 Agent 的定义

在 AI 领域，Agent 通常能够：

1. **感知环境（Perceive）**：读取文件、执行命令、获取信息
2. **做出决策（Decide）**：规划下一步行动
3. **执行动作（Act）**：调用工具改变环境状态
4. **观察结果（Observe）**：看到动作的效果
5. **循环迭代（Loop）**：基于结果调整计划

**感知 → 决策 → 行动 → 观察 → 再迭代** 的闭环，是 Agent 区别于单次问答的核心骨架。

### 8.4 ReAct：推理与行动的结合

2022 年，Yao 等人（Princeton + Google Research）提出 ReAct（Reasoning + Acting）框架：让 LLM 以交错方式生成推理轨迹与具体行动，从而在外部工具 / 环境中获取信息并修正计划[^react]。

```text
思考：我需要找出项目里所有的 TODO 注释
行动：Grep("TODO", path="src/")
观察：找到 23 个 TODO，分布在 8 个文件中
思考：我应该按优先级整理这些 TODO
行动：Read("src/main.ts")
观察：[文件内容]
思考：这个文件有 3 个 TODO，其中 2 个是高优先级
```

关键洞察：让模型在行动之前先「思考」，通常能提高任务完成质量与可解释性。现代 coding agent（包括 Claude Code）的多轮工具循环，都可以看作这一范式的工程化落地。

### 8.5 Claude Code 的 Agent 循环

从用户可见行为抽象，Claude Code 的核心是一个 Agent 循环：

```text
┌─────────────────────────────────────────────┐
│ Agent 循环                                   │
│                                              │
│ 用户输入                                     │
│ ↓                                            │
│ 构建消息列表（含历史）                       │
│ ↓                                            │
│ 调用 Claude API（流式）                      │
│ ↓                                            │
│ ┌─────────────────────────────────┐          │
│ │ 响应解析                         │          │
│ │ ├── 文本块 → 流式显示             │          │
│ │ ├── 思考块 → 内部 / 可展开查看    │          │
│ │ └── 工具调用块 → 执行工具         │          │
│ └─────────────────────────────────┘          │
│ ↓                                            │
│ 工具结果回填到消息列表                       │
│ ↓                                            │
│ 是否需要继续？                               │
│ ├── 是 → 回到「调用 Claude API」             │
│ └── 否 → 返回最终结果                        │
└─────────────────────────────────────────────┘
```

关键特性：

- **多轮工具调用**：一次用户请求可能触发多轮 API 调用
- **并行工具执行**：一次响应中可请求多个独立工具调用
- **上下文累积**：工具结果加入消息列表，模型能看到执行历史
- **自动终止**：模型认为任务完成时停止请求工具，生成最终回答

### 8.6 聊天机器人 vs Agent：关键差异

| 维度 | 聊天机器人 | Agent（以 Claude Code 为例） |
| ------ | ------ | ------ |
| 输出 | 文本 | 文本 + 动作 |
| 环境状态 | 通常不改变外部世界 | 可改变文件系统、进程、远程系统 |
| 循环 | 单轮或浅层多轮对话 | 多轮感知→决策→行动→观察 |
| 工具 | 无或有限插件 | 内置工具 + MCP 扩展 |
| 目标 | 回答问题 | 完成任务 |
| 失败处理 | 主要靠重新提问 | 可重试、调整策略、请求确认 |
| 时间跨度 | 秒级为主 | 分钟到更长 |

### 8.7 Agent 的新挑战

能力提升带来新挑战：

- **可靠性**：多步骤任务中任一步出错都可能放大失败
- **安全性**：真实动作可能造成不可逆损害，需要权限模型与确认机制
- **可观察性**：用户需要知道 Agent 在做什么
- **上下文管理**：长任务消耗大量 token，需要压缩与预算策略
- **成本控制**：多轮 API 调用成本高，需要追踪与约束

### 8.8 从工具调用到 Agent 系统

工具调用是基础，完整 Agent 系统通常还需要：

```text
工具调用
+ 上下文管理（记住做了什么）
+ 任务规划（知道下一步做什么）
+ 错误处理（出错时怎么办）
+ 权限控制（什么能做什么不能做）
+ 状态持久化（中断后能恢复）
= Agent 系统
```

### 8.9 小结

从聊天机器人到 Agent，本质上是从「生成文本」到「执行动作」的跨越。这个跨越需要：工具调用机制、Agent 循环，以及完整的工程系统（可靠性、安全性、可观察性）。Claude Code 是这一方向上高度工程化的实现之一；理解它的设计，有助于理解当代 Agent 系统工程的关键取舍。

---

## 9. AI 时代的新思考

> **本章作用**：回到第 0 章的质疑，用「Unix—无状态—Agent」已建立的框架，解释为何 grep / agentic search 在 Anthropic 的实测中能成为一条合理路线，并与索引类方案并置对照——**不是谁绝对更好，而是各自默认优化的目标不同**。

### 9.1 Claude Code 的选择：具体的技术对比

回到开头的质疑。根据 Latent Space 访谈与 Boris 在 HN 上的说明：早期 Claude Code 用过 RAG；内部评测（含大量主观体验）发现 agentic search 更适合其使用场景；同时避免了索引同步、第三方存放索引带来的安全与运维复杂性[^latent-space][^hn-rag]。

当前主流 AI 编程助手大致可对照三条路线：

1. **Cursor 的向量索引方案**  
   用 Merkle 树跟踪代码库变更，将代码按语法块切分后生成 embedding，存入远程向量库（如 turbopuffer）。查询时做语义近邻检索，再在本地读取对应源码片段拼进上下文[^cursor-index]。优势是语义联想——搜「用户认证」也能碰到 `login` / `authenticate` 等命名。

2. **JetBrains 的传统 IDE 索引**  
   经过长期打磨的 PSI / stub 等结构化索引，支撑跳转、重构、补全与精确符号导航。这是企业级 IDE 智能的基石，优化目标首先是**确定性的代码智能**，而非对话式 Agent。

3. **Claude Code 的无向量索引方案（Agentic Search）**  
   不依赖预构建向量索引；Agent 在任务过程中按需使用文件系统搜索与读取（grep / glob / read 等，具体工具形态会随版本演进）。像资深工程师在终端里工作：搜索、打开、追引用、再搜索。

| 维度 | JetBrains 传统索引 | Cursor 向量索引 | Claude Code agentic search |
| ------ | ------ | ------ | ------ |
| 核心 | PSI / 符号索引 | Embedding + 向量库 | 实时搜索 + LLM 推理 |
| 启动 | 需建索引 | 需建 / 同步向量 | 几乎零准备 |
| 实时性 | 增量更新 | 增量更新 | 始终读当前磁盘 |
| 搜索方式 | 精确符号匹配 | 语义相似度 | 文本匹配 + 多轮推理 |
| 主要代价 | 索引复杂度 | 嵌入 / 同步 / 远程依赖 | 延迟与 token |
| 适合 | 补全 / 跳转 / 重构 | 语义探索、RAG 式检索 | 可组合、可审计的 Agent 任务 |

### 9.2 为什么「健忘」有时反而更好？

这个反直觉选择，背后至少有四类收益（同时要诚实承认代价）：

1. **零索引启动与可组合性**  
   大型项目里，建索引或上传嵌入可能耗时数分钟。Agentic search 可以立刻开始工作。更重要的是管道组合：

   ```bash
   tail -f app.log | claude -p "如果看到异常就通过 Slack 通知我"
   ```

   这种 Unix 式组合，是「必须先有完整语义索引」的系统很难天然提供的。

2. **确定性与可调试性**  
   向量检索失败时，原因可能是嵌入质量、切块边界、索引过期或相似度阈值。grep 失败的原因通常更单一：模式不匹配。调试复杂问题时，这种确定性往往更宝贵。

3. **减少「持久化代码表示」的攻击面**  
   Cursor 官方说明：会把代码块送去生成 embedding，远程保存的是向量与元数据，而不是永久存一份明文源码；查询命中后再从本地读原文[^cursor-index]。即便如此，**embedding 本身并非绝对不可逆**：学术工作已表明，可从文本嵌入中重建相当比例的原文信息[^embedding-inversion]。  
   Claude Code 不维护代码向量库，避免了「索引存放在何处、如何同步、如何防泄漏」这一整类问题。但需注意：Agent 读取的文件内容仍会进入模型 API 上下文——它优化的是**索引链路的隐私与运维**，不是「代码永不离开本机」。

4. **维护成本接近于零**  
   没有「索引卡住」「缓存损坏」「后台嵌入进程吃 CPU」等常见故障面。每次搜索都基于当前工作区真相。

**对应代价（Anthropic 自己也承认）**：更高的延迟与 token 消耗[^latent-space]。Milvus / Zilliz 的批评正聚焦于此，并据此推广基于向量检索的 MCP 插件以降低 token[^milvus-grep]。两边说的往往是同一枚硬币的两面。

### 9.3 不同场景，不同选择

这不是技术优劣的绝对判断，而是设计哲学与场景匹配：

- **向量索引**适合创意 / 探索式编程——你大概知道要什么，但不确定具体符号名。
- **IDE 结构化索引**适合需要可靠重构、精确类型导航的企业级日常开发。
- **Agentic search**适合重视简单、可控、可组合、以及不愿维护代码向量索引的 Agent 工作流。

正如 Boris 在访谈中强调的：

> Claude Code is not a product as much as it's a Unix utility.  
> —— Boris Cherny[^latent-space]

### 9.4 无状态之美：在 AI 时代的新意义

Claude Code 的选择，让我们重新思考什么是「智能」。在一个 AI 无处不在的时代，真正稀缺的往往不只是智能本身，还有**可预测性**；不只是功能丰富，还有**行为确定**；不只是记住一切，还有**知道何时遗忘**。

这就是为什么 grep 在今天依然重要，为什么 Unix 哲学历经数十年依然闪光，为什么 Claude Code 的「倒退」在特定目标函数下其实是一种清醒。

简单的工具活得最久；「健忘」的设计，有时最自由。

---

## 附录 A：RAG（向量索引）

RAG 是 Retrieval-Augmented Generation 的缩写，常译为「检索增强生成」。核心思路是：让模型在回答之前，先从外部知识库（文档、代码、Wiki、工单、规范等）检索出相关材料，再把这些材料作为上下文交给模型生成答案。

在工程实现中，「向量索引」通常指：把知识库切分成文本块（chunk），为每段计算 embedding，并存入向量数据库或本地向量索引。用户提问时，同样把问题嵌入为向量，通过相似度检索出 Top-K 片段，拼进 prompt，模型基于「检索到的证据」生成回答。

RAG 主要解决三类问题：

1. **上下文窗口有限**：知识库很大，不可能一次性塞进模型；RAG 通过「只取相关片段」节省窗口。
2. **语义检索需求**：不确定关键词时（想找「用户认证」但代码里叫 `login`），向量相似度往往更友好。
3. **减少幻觉与提高可追溯性**：模型引用检索片段回答，更便于回溯来源（前提是检索足够准）。

RAG 的代价与风险也很现实：

- **索引成本**：切分、嵌入、建库；仓库频繁变更时，增量更新与一致性会变复杂。
- **片段质量依赖**：chunk 粒度、边界、元数据会直接影响检索质量。
- **token 成本**：检索片段要进上下文才有用；片段越多越长，注入成本越高。
- **隐私与合规**：embedding 可能带来信息泄露风险；即便不保存原文，也可能存在从向量反推内容的风险[^embedding-inversion]。

在 AI 编程助手语境里，RAG（向量索引）更像一种「把代码库变成可被语义检索的知识库」的方法：探索陌生项目、关键词不确定时很有用；当更重视确定性、可控性、零索引维护，以及 Unix 式可组合时，agentic search（grep / glob + 多轮推理）往往更贴近另一套工程哲学。

两者也可以互补：例如通过 MCP 给 Claude Code 增加语义检索工具——这正是社区与厂商正在尝试的方向[^milvus-grep]。

---

## 参考文献

[^milvus-grep]: Zilliz / Milvus, [*Why I’m Against Claude Code’s Grep-Only Retrieval? It Just Burns Too Many Tokens*](https://milvus.io/blog/why-im-against-claude-codes-grep-only-retrieval-it-just-burns-too-many-tokens.md).

[^hn-rag]: Boris Cherny on Hacker News, confirming Claude Code does not currently use RAG and that agentic search outperformed it in their testing: [comment](https://news.ycombinator.com/item?id=43164253).

[^latent-space]: swyx / Latent Space, [*Claude Code: Anthropic's Agent in Your Terminal*](https://www.latent.space/p/claude-code)（访谈 Boris Cherny、Cat Wu；含 agentic search vs RAG 的讨论）。

[^mcilroy-unix]: Doug McIlroy 的 Unix 哲学三句概括，见 Peter H. Salus 整理转述；Eric S. Raymond, [*The Art of Unix Programming*](http://catb.org/esr/writings/taoup/html/ch01s06.html) 亦有引用。

[^bstj-1978]: M. D. McIlroy, E. N. Pinson, B. A. Tague, “UNIX Time-Sharing System: Forward,” *Bell System Technical Journal*, 57(6), 1978.

[^claude-code-docs]: Anthropic, [Claude Code 文档](https://code.claude.com/docs/en/)（强调 Unix 式可组合、管道与 CI 用法）。

[^backus-1977]: John Backus, “Can Programming Be Liberated from the von Neumann Style? A Functional Style and Its Algebra of Programs,” *Communications of the ACM*, 1978（基于 1977 年图灵奖演讲）。

[^fielding-rest]: Roy Thomas Fielding, [*Architectural Styles and the Design of Network-based Software Architectures*](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm), PhD dissertation, UC Irvine, 2000（REST 的 Stateless 约束）。

[^openai-fc]: OpenAI, [*Function calling and other API updates*](https://openai.com/index/function-calling-and-other-api-updates/)（2023-06-13）.

[^anthropic-tools]: Anthropic, [*Claude can now use tools*](https://claude.com/blog/tool-use-ga)（Tool Use GA, 2024-05-30）.

[^react]: Shunyu Yao et al., [*ReAct: Synergizing Reasoning and Acting in Language Models*](https://arxiv.org/abs/2210.03629), 2022.

[^aima]: Stuart Russell & Peter Norvig, *Artificial Intelligence: A Modern Approach*.

[^cursor-index]: Cursor, [*Securely indexing large codebases*](https://cursor.com/blog/secure-codebase-indexing)；turbopuffer, [*Cursor scales code retrieval…*](https://turbopuffer.com/customers/cursor).

[^embedding-inversion]: John Morris et al., [*Text Embeddings Reveal (Almost) As Much As Text*](https://arxiv.org/abs/2310.06816), EMNLP 2023.
