# CPU 访存：从虚地址到 Cache 与 DRAM

> `int x = array[100];` 在程序员眼里是一行赋值；在 CPU 眼里，是地址生成、地址翻译、TLB、页表、多级 Cache，以及可能的 DRAM 与缺页。
>
> 本文把「CPU 如何找到内存中的数据」拆成一条可跟的链路：**Virtual Address → MMU / TLB / Page Table → Physical Address → Cache Hierarchy → Memory Controller → DRAM**。可与本库 [10](../10-chronicle/10-computing-cloud-chronicle.md)（计算如何池化）、[40](../40-paradigm/40-unix-agent-stateless-philosophy.md)（工具边界与可观察）、[22](./22-distributed-consistency-treatise.md)（跨节点之后的另一套「找数据」）对照：本文写单机上一次 Load 的硬件路径，不是分布式副本协议。

先给一个直接答案：

> **CPU 不是拿着程序里的地址去 RAM 里「搜索标签」。** 程序给出的多半是**虚拟地址**；MMU 借助 TLB（及必要时的页表遍历）把它译成**物理地址**；随后在 L1D→L2→L3 的 Cache 层级里按 **Cache Line** 查找；全未命中才经 Memory Controller 访问 DRAM，并通常整行填入 Cache。TLB 回答「这一页映到哪」；Cache 回答「附近是否已有这份数据」；DRAM 子系统回答「物理介质上的位置」。访问模式友好时极快，TLB Miss / Cache Miss 叠加时极贵——慢的往往不是算术，而是这条链路。

**20-architecture 位置：** [20](./20-enterprise-architecture-treatise.md)–[25](./25-architecture-thinking-cto-treatise.md) 多写企业与平台架构。本文写**机器内部的访存架构**：一次 Load 经过哪些硬件与 OS 交界。术语先看第 1.1 节。

## 摘要

当程序执行 `int x = array[100];` 时，编译器生成含地址计算的机器指令，CPU 在返回该元素之前通常经过：虚拟地址生成；MMU 将虚拟页号译为物理页号（页内偏移不变）；优先查 TLB，Miss 则多级页表遍历（x86-64 常见四级，可选 LA57 五级），必要时 Page Fault 由 OS 处理；再查 Cache（常见 64B Cache Line）。L1 常按页内偏移做虚拟索引，与 TLB 重叠，标签仍用物理地址比对；L1D / L2 / L3 逐级 Miss 后才到 Memory Controller 与 DRAM，并回填 Cache Line。本文给出简化流水与心智模型，标明真实处理器会乱序、推测与重叠访存；并说明 Cache Miss 与 TLB Miss 是两类不同代价。理解这条链路，是理解「同样算术、天差地别的墙钟时间」的关键之一。

**关键词：** 虚拟地址；MMU；TLB；页表；Page Fault；Cache Line；L1D；DRAM；局部性；Working Set

---

## 目录

- [摘要](#摘要)
1. [读法与术语](#1-读法与术语)
    - [1.1 术语对照](#11-术语对照)
    - [1.2 边界](#12-边界)
2. [程序给出的通常是虚拟地址](#2-程序给出的通常是虚拟地址)
3. [MMU：虚拟页到物理页](#3-mmu虚拟页到物理页)
4. [TLB：避免每次都走页表](#4-tlb避免每次都走页表)
5. [TLB Miss 与页表遍历](#5-tlb-miss-与页表遍历)
6. [查 Cache：标签要比对物理页](#6-查-cache标签要比对物理页)
7. [Cache Line、Hit 与 Miss](#7-cache-linehit-与-miss)
8. [Cache 全 Miss：Memory Controller 与 DRAM](#8-cache-全-missmemory-controller-与-dram)
9. [串起来：一次 Load 的简化路径](#9-串起来一次-load-的简化路径)
10. [CPU 并不是在「搜索 RAM」](#10-cpu-并不是在搜索-ram)
11. [为何有时访存极贵](#11-为何有时访存极贵)
12. [收束：三条问题](#12-收束三条问题)
13. [本章要点](#13-本章要点)
14. [参考文献](#14-参考文献)

---

## 1. 读法与术语

一次典型的数据 Load，在通用机上常经过下表四问；图示是教学简图，真实微架构会重叠、推测与乱序。

| 环节 | 回答的问题 |
| ---- | ---------- |
| Virtual Address | 这个进程眼里的地址是什么？ |
| MMU + Page Table / TLB | 对应哪一块物理页？ |
| CPU Cache（L1/L2/L3） | 附近是否已有这份数据？ |
| Memory Controller + DRAM | 物理介质上如何选中并读出？ |

### 1.1 术语对照

| 术语 | 一句话 |
| ---- | ------ |
| **Virtual Address（VA）** | 进程地址空间中的地址；程序与多数用户态指针所见通常是它。 |
| **Physical Address（PA）** | 经翻译后、面向物理内存与部分外设映射的地址。 |
| **Page / 页** | 地址翻译的粒度；常见 4 KiB，亦有大页（如 2 MiB / 1 GiB 等，视架构与 OS）。 |
| **Page Offset** | 页内偏移；翻译时通常不变，拼在物理页号之后。 |
| **MMU** | Memory Management Unit：负责（或参与）虚实地址翻译的硬件。 |
| **Page Table** | OS 维护的映射结构：虚拟页 → 物理页（及权限等位）；本身也在内存中。 |
| **TLB** | Translation Lookaside Buffer：缓存近期虚→实翻译，避免每次都走内存中的页表。 |
| **Page-Table Walk** | TLB Miss 后硬件（或软硬协同）遍历多级页表以取得翻译。 |
| **Page Fault** | 页表项表明页不在内存、或权限不符等时触发的异常；常由 OS 调入页或杀进程等处理。 |
| **Cache Line** | Cache 与内存之间常见的传输/存储粒度；许多平台上为 **64 bytes**。 |
| **L1I / L1D** | 一级指令 Cache / 数据 Cache；Load 数据走 L1D。 |
| **Spatial / Temporal Locality** | 空间局部性（附近地址）、时间局部性（不久会再访问）；Cache 设计所利用的统计规律。 |
| **Working Set** | 一段时间窗口内实际触达的页/数据集合；过大则 TLB/Cache 压力陡增。 |

### 1.2 边界

1. **本文写通用 CPU + 分页 + 多级 Cache 的教学路径。** 嵌入式无 MMU、GPU、IOMMU、CXL 等另论。  
2. **级数、页大小、Cache 组织因架构而异。** x86-64 四级/五级、ARM 等表述以厂商手册为准；正文取通行概念。  
3. **简化流不是时序图。** 乱序执行、预取、推测执行、多级 TLB、store buffer 等会改变「看起来的顺序」。  
4. **性能数字不冻结。** 延迟随微架构与频率变化；正文讲结构，不背某一代 CPU 的纳秒表。

---

## 2. 程序给出的通常是虚拟地址

程序使用的地址通常是 **Virtual Address**，而不是物理 RAM 地址。执行 `int value = array[100];` 时，编译器生成含地址计算的机器指令；CPU 最终可能访问类似：

```text
Virtual Address = 0x7F1234567890
```

该地址属于当前进程的 **Virtual Address Space**。每进程通常自有一套虚拟空间：隔离进程，并允许 OS 独立地把虚拟页映到物理页。[^va]

CPU 首先要回答：**这个 VA 对应哪一块物理内存？** ——这就是 **Address Translation**。

**所以 · 边界在哪：** 程序员手里的指针，默认不是 DRAM 上的门牌号。

---

## 3. MMU：虚拟页到物理页

现代 CPU 中包含负责（或参与）内存地址转换的硬件，通常称为 **Memory Management Unit（MMU）**。

一个虚拟地址可以从概念上分成两部分：

```text
Virtual Address
┌───────────────────────┬──────────────┐
│ Virtual Page Number   │ Page Offset  │
└───────────────────────┴──────────────┘
```

内存被划分成固定大小的单元，称为 **Page**。**4 KiB** 是一种常见页大小；现代处理器与 OS 也支持更大的页，以减少页表项数量、提高 TLB 覆盖率（代价是内部碎片与管理复杂度）。[^page-size]

操作系统维护 **Page Table**，描述虚拟页如何映射到物理页。从概念上看：

```text
Virtual Page 1234
       ↓
Physical Page 5678
```

在地址翻译过程中，**Page Offset 不会改变**。例如：

```text
Virtual Address
┌───────────────┬────────────┐
│ Virtual Page  │   Offset   │
└───────────────┴────────────┘
        │
        │ Page Table
        ↓
┌───────────────┬────────────┐
│ Physical Page │   Offset   │
└───────────────┴────────────┘
          Physical Address
```

因此，硬件转换的是虚拟页号，同时保持偏移不变，再拼出物理地址。[^mmu]

**所以 · 边界在哪：** 翻译的粒度是页，不是「每个字节一张地图」。

---

## 4. TLB：避免每次都走页表

这里有一个明显问题：**页表本身也存储在内存中**。如果每次地址翻译都必须访问 RAM，那么一次数据访问可能先要多次访存，才能知道数据到底在哪。

现代 CPU 用一种特殊的翻译缓存解决这个问题：**Translation Lookaside Buffer（TLB）**。TLB 保存最近用过的虚→实翻译。从概念上看：

```text
Virtual Page
     ↓
    TLB
     ↓
Physical Page
```

- 翻译已在 TLB 中 → **TLB Hit**：可避免当场遍历页表。  
- 翻译不在 TLB 中 → **TLB Miss**：需要 **Page-Table Walk**（及可能的缺页处理）。

现代处理器还可缓存页表项或 walk 的中间结果，进一步加速。若页表项表明该页当前不在物理内存中（或权限不允许），处理器会触发 **Page Fault**；操作系统处理异常——可能从存储调入该页、更新映射，然后让指令重试；也可能向进程投递信号或终止进程。[^tlb]

**所以 · 边界在哪：** TLB 是翻译路径上的「L1」；没有它，分页在性能上几乎不可用。

---

## 5. TLB Miss 与页表遍历

现代系统通常使用**多级页表**。例如，在典型的 64 位架构中，CPU 在获得物理页号之前，可能需要遍历多个 paging structure level：

```text
Virtual Address
      │
      ▼
   Level 1
      │
      ▼
   Level 2
      │
      ▼
   Level 3
      │
      ▼
   Level 4
      │
      ▼
Physical Page
```

在 **x86-64** 中，传统的**四级分页**被广泛使用；支持 **LA57** 的处理器还可启用**五级分页**，以扩展可用的虚拟地址宽度（线性地址从约 48 bit 量级扩展到 57 bit 量级）。[^la57]

这些页表结构本身也在内存中，故一次 TLB Miss 可能在 walk 期间触发**多次**访存。OS 还可用大页、透明大页等扩大单次翻译的覆盖（权衡碎片与延迟）。[^drepper]

**所以 · 边界在哪：** Miss 的代价不是「多查一次表」四个字，而是可能拖出一串依赖访存。

---

## 6. 查 Cache：标签要比对物理页

假设 CPU 已成功完成：

```text
Virtual Address
       ↓
Physical Address
```

你可能会以为下一步是：

```text
Physical Address → DRAM → Data
```

但现代 CPU 通常会先检查 **Cache Hierarchy**。DRAM 的延迟与带宽特征远逊于片上 Cache；把热数据留在离核心更近的地方，是性能的基本盘。

一个简化的层次可以表示为：

```text
CPU Core
   │
   ├── L1 Cache
   │
   ├── L2 Cache
   │
   ├── L3 Cache（常为多核共享）
   │
   └── Main Memory (DRAM)
```

L1 通常最小最快；更低层级更大、更慢。L1 常分为 **L1I**（指令）与 **L1D**（数据）；`array[100]` 这类 Load 查的是 **L1D**。[^cache-hier]

教学顺序可以先写成「译出物理页，再查 Cache」。实现上，L1 常见 **VIPT**（虚拟索引、物理标签）：用虚拟地址里**未经翻译的页内偏移**做组索引，因此可以和 TLB 并行；标签比对仍要物理页号。索引若用到页号，同一物理页的不同虚拟别名会打架，所以 L1 的路数与容量受页大小约束。更下级 Cache 更常接近「先有物理地址再查」（PIPT）。[^cache-hier]

**所以 · 边界在哪：** 下一步通常不是 DRAM，而是问 Cache；L1 不必等完整物理地址才开始索引，但命中与否仍由物理标签裁定。

---

## 7. Cache Line、Hit 与 Miss

CPU Cache 通常不会按「每个字节一个独立格子」来组织。它们使用固定大小的数据块，称为 **Cache Line**。许多现代平台上常见 **64 bytes** 一行，具体组织（相联度、组数、写策略）取决于微架构。[^cache-line]

假设 CPU 需要访问地址 `0x1008`。Cache 实际上可能加载整个对齐块，例如：

```text
0x1000 ───────────────── 0x103F
        64-byte Cache Line
```

需要的数据落在这一行里。Cache Line 按边界对齐，**并不是**「从请求地址起再读 64 字节」。这种设计利用了 **Spatial Locality**：若程序访问了一个位置，接下来往往也会访问附近位置。

```c
for (int i = 0; i < 1000; i++) {
    sum += array[i];
}
```

连续元素在内存中相邻时，一次行填充可以服务多次迭代——这正是顺序扫描常比随机跳跃更快的结构原因之一。

CPU 会在对应 Cache 层级中寻找所需 Cache Line：

```text
        Memory Access
             │
             ▼
           L1D
          /    \
       Hit      Miss
       │          │
       ▼          ▼
     Data       L2
               /  \
            Hit    Miss
            │        │
            ▼        ▼
          Data      L3 → … → DRAM
```

- **Cache Hit**：在该级找到，延迟相对低。  
- **Cache Miss**：请求下沉到下一级；各级皆 Miss 则走向主存。

不同 CPU 的 Cache 组织不同，真实硬件还可重叠 Cache 访问与其他内存操作。基本原则不变：**让经常访问的数据尽可能靠近核心。**

**所以 · 边界在哪：** Cache 买卖的是「行」与局部性；一次未命中常常拉回一整行邻居。

---

## 8. Cache 全 Miss：Memory Controller 与 DRAM

各级 Cache 皆 Miss 后，请求走向 **Main Memory**（通常是 DRAM）。内存子系统与 **Memory Controller** 通信，由控制器按物理地址选通位置：

```text
CPU → Cache Hierarchy ─Miss→ Memory Controller → DRAM → Data
```

回填时仍按更大粒度：把数据放入一条 Cache Line，使邻域后续访问有机会命中。一次 DRAM 访问往往是在**填充一行**，而不只是吐回最初那一个字节。[^dram]

在 DRAM 界面上，PA 还会被解码为 Channel、Rank、Bank、Row、Column 等（随 DIMM 与控制器而变）——地址是**选通输入**，不是仓库里的货名标签。

**所以 · 边界在哪：** 主存是层级的最后一站，不是程序员默认的第一站。

---

## 9. 串起来：一次 Load 的简化路径

执行 `x = array[100];` 时，教学用路径可收成：

```text
        CPU executes load
                │
                ▼
      Generate virtual address
                │
                ▼
             Check TLB
            /         \
         Hit           Miss → Page-table walk
         │               /         \
         │         Valid page   Page fault → OS
         │               │
         └───────────────┴─► Physical address
                              │
                              ▼
                        Check L1D → L2 → L3 → DRAM
                              │
                              ▼
                         Data → Core
```

真实处理器会重叠、推测、乱序、预取，并并发多个请求；本图只保**基本原理**。§10–§12 不再重画全图，只钉心智与代价。

**所以 · 边界在哪：** 能跟上这张简图，就够读大多数「为啥这么慢」的讨论。

---

## 10. CPU 并不是在「搜索 RAM」

常见误读是：CPU 喊一个地址，RAM 全文检索后喊「找到了」。地址不是检索标签，而是硬件 **Addressing Mechanism** 的输入——经 TLB/页表得到 PA，再经 Cache 查找或控制器选通 DRAM。[^drepper]

**所以 · 边界在哪：** 找数据 = 翻译 + 查找 + 选通，不是按名字搜库。

---

## 11. 为何有时访存极贵

算术量相近的两段代码，墙钟时间可以差出一个数量级以上：一个反复命中 L1；另一个局部性差，Cache Miss 与 TLB Miss 叠加。二者是**不同类型**的 Miss：

| 概念 | 含义 |
| ---- | ---- |
| **Cache Miss** | 要的数据不在相关 Cache 中 |
| **TLB Miss** | 要的虚→实翻译不在 TLB 中 |

相关旋钮包括：Cache / TLB 局部性、Cache Line、顺序访问、数据布局、Working Set、页大小（含大页）。[^drepper] 工程上常见的拖慢：大图指针追逐、按列扫行主序矩阵、伪共享（false sharing）、工作集远超末级 Cache 与 TLB 覆盖——根子多在这条链，而不在多写了一句 `if`。

**所以 · 边界在哪：** 优化算术前，先问数据离核心有多远、翻译是否总在 TLB 里。

---

## 12. 收束：三条问题

若只留一个心智模型，记住开篇那句：**不是拿着程序地址直接搜 RAM。** 平时用三个问题跟一次 Load：

| 部件 | 问什么 |
| ---- | ------ |
| **TLB** | 这个虚拟页映到哪？ |
| **Cache** | 附近是否已有这份数据？ |
| **Memory Subsystem** | 物理位置如何选通？ |

`x = array[100];` 背后，可能叠着地址生成、翻译、TLB、页表 walk、多级 Cache，以及 DRAM。访问模式友好时极快，不友好时极慢——慢的往往是这条链路。

---

## 13. 本章要点

1. **程序地址多为 VA**；每进程一空间，页表映到物理页。  
2. **MMU 译页号、保偏移**；粒度是页（常 4 KiB，可有大页）。  
3. **TLB 缓存翻译**；Hit 免走页表，Miss 可能多次访存 + 缺页。  
4. **x86-64 常见四级页表**；LA57 可启五级以扩大 VA。  
5. **先问 Cache，再谈 DRAM**；L1 常用 VIPT（页内偏移索引、物理标签）；Load 走 L1D；传输粒度常为 64B Cache Line。  
6. **Hit/Miss 逐级下沉**；全 Miss 经 Memory Controller 到 DRAM，并回填行。  
7. **地址不是搜索标签**；是翻译、查找与选通的输入。  
8. **Cache Miss ≠ TLB Miss**；局部性、布局与 Working Set 决定墙钟时间。

---

## 14. 参考文献

[^va]: 虚拟内存与进程地址空间为现代通用 OS（如 Linux、Windows、macOS）的通行模型。教学表述见 OpenStax 等计算机系统教材中 Virtual Memory 章节；细节以具体 OS 与架构手册为准。

[^page-size]: 4 KiB 页在 x86 与许多 Unix 系统上长期为默认；大页（huge pages）用于降低 TLB 压力。具体可用页大小见处理器与内核文档。

[^mmu]: MMU / 分页翻译的通用模型：VPN → PPN，offset 不变。实现含硬件 walk、软件辅助 walk 等变体；正文取「硬件 MMU + 内存中页表」的常见桌面/服务器路径。

[^tlb]: TLB 作为地址翻译缓存；Miss 触发 page-table walk 或陷阱。多级 TLB、在 walk 时缓存中间页表项，是现代微架构的常见优化。厂商说明见 Intel 软件开发者手册中关于 paging / TLB 的章节。

[^la57]: Intel 5-level paging（CR4.LA57）：在 IA-32e 模式下将线性地址宽度扩至 57 bit；Ice Lake 一代起出现于服务器等产品。未置位时仍用四级分页。见 Intel 白皮书 *5-Level Paging and 5-Level EPT* 及后续 SDM 叙述。

[^cache-hier]: 多级 Cache（L1/L2/L3）为现代 CPU 标配；L1 常分指令/数据。容量与延迟随型号变化；正文只取「越近越快、越远越大」结构。L1 的 VIPT 用页内偏移做组索引、物理地址做标签，索引才能与翻译重叠；组索引一旦用到页号，别名就会冲突。

[^cache-line]: 64-byte Cache Line 在 x86 服务器与客户端上极为常见；并非所有架构皆然。Cache 按行对齐填充，利用空间局部性。见 Intel 关于 Cache 组织的公开文档及系统教材。

[^dram]: 内存控制器将物理地址映射到 DRAM 的 channel/bank/row/column 等；一次未命中常按 burst 填充 Cache Line。细节见 JEDEC / 控制器文档；正文取教学粒度。

[^drepper]: Ulrich Drepper, *What Every Programmer Should Know About Memory*（2007，后续有修订讨论）。系统程序员理解 Cache、TLB、局部性与布局的经典长文；数值已过时，结构仍可用。

**声明：** 正文是访存路径的教学整理，不是某一代 CPU 的周期精确模型，也不替代性能计数器实测。微架构、页大小与 Cache 几何会变；冲突时以处理器手册、内核文档与 profiling 为准。
