# Calico 网络专论：节点即路由器与包的路径

> Calico 之名源自 Felix（卡通形象「菲力猫」），由 Metaswitch 于 2015 年发起，后由 Tigera 维护并以开源形式进入 CNCF 生态。它把数据中心网络的老问题——「容器之间如何互通」——收敛到一个最朴素的答案：**让每台节点即一台路由器**。

先记住总纲：

> **Calico 不是又一层 overlay，而是一台「分布式路由集群」：**  
> 转发数据包的是 Linux 内核；Felix、BIRD、confd 只负责把内核的状态写对、把路由告诉别的机器。控制面写表，数据面查表——三个守护进程都不在包的路径上。[1][2][10]

可与本库 [《Kubernetes 控制面原则》](./23-kubernetes-control-plane-doctrine.md) 对照：彼处讲控制面如何收敛；此处讲同一集群中，包在节点上究竟如何行走。两文合看，方知编排与网络是同一台「分布式控制计算机」的两个面。

全文采用同一组示例，便于对照追踪：

| | 节点 IP | 上面的 Pod |
|--|---------|------------|
| **Node A** | `172.18.203.10` | Pod A1 `10.65.0.24`、Pod A2 `10.65.0.25` |
| **Node B** | `172.18.203.126` | Pod B `10.65.0.21` |

---

## 目录

- **背景与取舍**：[1. 问题与 Calico 的取舍](#1-问题与-calico-的取舍)
- **组件与控制面**：[2. 节点上跑什么：三个进程 + CNI](#2-节点上跑什么三个进程--cni)
- **数据面路径**：[3. 包的实际路径：只是 Linux 路由](#3-包的实际路径只是-linux-路由)
- **路由与地址**：[4. 路由如何出现在每台机器上](#4-路由如何出现在每台机器上) · [5. 地址分配与封装开关](#5-地址分配与封装开关)
- **工程落地**：[6. 选型与排查](#6-选型与排查)
- **收束**：[7. 总结](#7-总结) · [8. 参考文献](#8-参考文献)

```mermaid
flowchart TB
  classDef ctx fill:#e8f1fa,stroke:#3b6ea5,color:#1f3a5f
  classDef comp fill:#eef6e8,stroke:#5a8a3a,color:#2f4f1f
  classDef path fill:#fdeee8,stroke:#c46a2a,color:#5f2f10
  classDef eng fill:#f0e8fa,stroke:#6a3a8a,color:#3a1f5f

  Ctx["**背景与取舍**<br/>K8s 网络模型 · 路由 vs Overlay<br/><i>§1</i>"]:::ctx
  Comp["**组件与控制面**<br/>Felix · BIRD · confd · datastore<br/><i>§2</i>"]:::comp
  Path["**数据面路径**<br/>veth · 路由表 · iptables/eBPF<br/><i>§3</i>"]:::path
  Eng["**落地与排查**<br/>BGP 拓扑 · IPAM · 封装 · MTU<br/><i>§4–§6</i>"]:::eng

  Ctx -->|"明确问题域"| Comp
  Comp -->|"约束转发行为"| Path
  Path -->|"反推控制面需求"| Eng
  Eng -.->|"实践反馈取舍"| Ctx
```

---

## 1. 问题与 Calico 的取舍

Kubernetes 对网络的要求很简洁：每个 Pod 一个 IP，任意两个 Pod 直接互访，中间不做 NAT。[3]

满足该要求常见有两条路径：

| 思路 | 做法 | 代价 |
|------|------|------|
| **Overlay** | 底层仅识别节点 IP，将 Pod 包再封装一层节点 IP 转发 | 每个包多一层头；Pod IP 出不了集群 |
| **纯路由** | 让底层网络也识别 Pod IP，按三层路由转发 | 底层须能转发「不属于节点子网」的地址 |

Calico 默认采用第二条：**每台节点即一台 vRouter**，转发交给 Linux 内核，「该 Pod IP 位于哪台机器」用 BGP 在节点间分发。[10] 仅当底层无法做到（公有云跨子网、交换机不可控）时，才退回 IPIP / VXLAN 封装。[5]

由此推出两件须分开看待的事：

- **能否到达**：由路由表决定。本机 Pod 路由由 Felix 写入；到其他节点 Pod 的路由官方称 cluster routes。[1][2]
- **到达后是否放行**：由 NetworkPolicy 决定，由 Felix 编译为内核规则。[1]

一句话记住整体形状：**控制面写表，数据面查表。** 三个守护进程都不在包的路径上。

---

## 2. 节点上跑什么：三个进程 + CNI

每个节点上有一个 `calico-node` Pod（镜像 `calico/node`），内含三个常驻进程。此外 kubelet 在创建/删除 Pod 时会调用 **CNI 插件**——它不属于这三个常驻进程，负责分配 IP、创建 veth；Felix 随后补齐内核状态。[1][3]

```mermaid
flowchart TB
  classDef store fill:#fff3e0,stroke:#b8860b,color:#5a4200
  classDef proc fill:#eef6e8,stroke:#5a8a3a,color:#2f4f1f
  classDef kern fill:#e8f1fa,stroke:#3b6ea5,color:#1f3a5f
  classDef peer fill:#f0e8fa,stroke:#6a3a8a,color:#3a1f5f

  subgraph 控制面["控制面 · 只改状态，不转发数据包"]
    Store["datastore<br/>kdd: K8s API（后端为 K8s 的 etcd）<br/>etcd 模式: Calico 专用 etcd"]:::store
    Felix["Felix<br/>写接口 / 本机路由 / 策略"]:::proc
    Confd["confd<br/>生成 BIRD 配置"]:::proc
    BIRD["BIRD<br/>BGP 客户端"]:::proc
  end

  subgraph 内核["数据面 · Linux 内核 · 真正转发"]
    Veth["veth / 网卡"]:::kern
    FIB["路由表 FIB"]:::kern
    ACL["iptables 或 eBPF"]:::kern
  end

  Peer["其他节点的 BIRD / ToR"]:::peer

  Store -->|"配置"| Felix
  Store -->|"BGP 配置"| Confd
  Confd -->|"渲染并 reload"| BIRD
  Felix -->|"写接口 / 本机路由 / 策略"| 内核
  FIB -.->|"读本机路由"| BIRD
  BIRD <-->|"BGP 通告 / 学习"| Peer
  BIRD -->|"写回远端路由"| FIB
  Veth --> FIB --> ACL
```

三个进程的边界，按「与内核」「与其他节点」两条线划分：

| | **Felix** | **BIRD** | **confd** |
|--|-----------|----------|-----------|
| 职责 | 将本机 endpoint 所需路由与策略写入内核 [1] | 用 BGP 分发本机路由，并将习得的远端路由写回 [1][2] | 监视集群配置，生成 BIRD 配置并触发其加载 [1] |
| 与内核 | **写**：接口属性、本机 Pod 路由、防火墙规则 | **读**本机路由，**写**远端路由 | **不接触内核** |
| 与其他节点 | 不建立 BGP | 与其他节点 / 交换机建立 BGP | 不建立会话 |
| 不做 | 不建 veth，不跑 BGP | 不建 veth，不写 NetworkPolicy | 不转发，不写 iptables |

### 2.1 datastore：先予澄清，勿与 etcd 混为一谈

文档中反复出现的 **datastore** 是一个抽象，而非某个具体组件。Calico 将所有配置（`IPPool`、`BGPPeer`、`FelixConfiguration`、NetworkPolicy、endpoint 状态……）放入 datastore，Felix / confd / kube-controllers 均从中读取。它有两种实现，**Kubernetes 集群默认并非直接使用 etcd**：[1][12]

| datastore | 实际存储位置 | 访问方式 | 适用场景 |
|-----------|---------|----------------|-----------|
| **Kubernetes API datastore（kdd）** | Kubernetes API server 背后的 etcd（**即 K8s 自身的 etcd**），以 `projectcalico.org/v3` CRD 形式存储 | 经 Kubernetes API（kube-apiserver），与 `kubectl` 同路径 | **K8s 集群默认**；管理简单，可复用 RBAC / audit log |
| **etcd** | 一个**专供 Calico 的 etcd**（可独立于 K8s 的 etcd，也可共享） | Calico 组件直连 etcd gRPC | 非 K8s 平台、多 K8s 集群共用一份 Calico、或希望 Calico 与 K8s 数据存储独立扩容 |

两处易混点：

- **kdd 模式下，物理存储仍是 etcd** —— 因 Kubernetes 自身以 etcd 为后端。区别仅在于 Calico **不直连 etcd**，而是经 kube-apiserver 读写 CRD。故「datastore = etcd」在 kdd 下**只对一半**：物理上对，访问路径上不对。
- **etcd 模式下，该 etcd 归 Calico 专属**，与 K8s 控制面的 etcd 通常为两套，可独立扩容、独立备份。这才是「datastore 指 etcd」最准确的语境。

此区别的重要性：

- **扩容对象不同**：kdd 的瓶颈在 kube-apiserver（以 Typha 缓解）；etcd 模式的瓶颈在专用 etcd（按 etcd 规模调参）。
- **故障域不同**：kdd 下 K8s apiserver 故障则 Calico 亦读不到配置；etcd 模式下两套存储互不影响。
- **运维边界不同**：kdd 复用 `kubectl` 一套工具、一套 RBAC、一套 audit；etcd 模式须单独维护 etcd 的备份、证书、压缩。

概括：**「datastore」是 Calico 的配置存储抽象；在 K8s 默认安装中它是 kube-apiserver 背后的那套 etcd，但 Calico 仅通过 Kubernetes API 访问；唯有显式选择 etcd 模式时，才是 Calico 直连一个专用 etcd。**

### 2.2 Felix：本机执行者

官方列出四项职责，全部发生在**本机**：[1]

| 职责 | 写入位置 | 效果 |
|------|--------|------|
| 接口 | 网卡参数、ARP、转发开关 | 主机以**自身 MAC** 回答 Pod 的 ARP；开启 IP 转发 [1][2] |
| 本机路由 | 路由表 | 「去往本机某 Pod IP，从对应 veth 出去」 |
| 策略 | iptables 或 eBPF | 仅放行 NetworkPolicy 允许的流量，且 Pod 无法绕过 |
| 状态上报 | 写回 datastore | 使本机配置失败可见 |

Felix 不与其他节点交换路由。

### 2.3 BIRD：BGP 客户端

Felix 将本机路由插入内核后，BIRD 用 BGP 将其分发；对端通告的路由，BIRD 再写回本机路由表。[1][2] 它不创建网卡，也不解释 NetworkPolicy。

### 2.4 confd：BIRD 的配置生成器

confd 监视 datastore（kdd 下为 kube-apiserver 中的 CRD，etcd 模式下为专用 etcd）中与 BGP 相关的配置（AS 号、对等体、IPAM 等）。配置变更后，重写 BIRD 配置文件并触发 BIRD 重新加载。[1] BGP 对等体变更链路：

```text
改 BGPPeer / BGPConfiguration
   → 写入 datastore（kubectl apply CRD，或写 etcd）
   → confd 监到变化，生成新 BIRD 配置
   → BIRD reload
   → 会话按新配置建立
```

Felix 写策略 **不经过** confd。

仅需策略、不分发节点间路由时（部分托管云），设 `CALICO_NETWORKING_BACKEND=none`：仅保留 Felix，不运行 BIRD 与 confd。[1]

---

## 3. 包的实际路径：只是 Linux 路由

官方将数据面归纳为三件事：**主机 MAC 答 ARP、按路由表转发、用 iptables 或 eBPF 做防火墙。**[2] 没有黑盒，没有用户态转发。

### 3.0 veth：Pod 与主机之间的虚拟网线

三种场景的首步均为「包从 Pod 出来，经 veth 进主机」。此处一次性讲透，后文不再展开。

**veth 是 Linux 内核的一种虚拟网卡，永远成对出现**：两块接口如同一根虚拟网线相连，从一端注入的包会从另一端弹出。两端可置于不同的 network namespace，故 veth 成为连接两个网络命名空间的标准手段。[3]

Calico 在 K8s 中的用法（Pod 创建时由 **CNI 插件**完成，而非 Felix）：[3]

```mermaid
flowchart LR
  classDef pod fill:#fdeee8,stroke:#c46a2a,color:#5f2f10
  classDef host fill:#e8f1fa,stroke:#3b6ea5,color:#1f3a5f

  subgraph PodNS["Pod 的 network namespace"]
    Eth0["eth0<br/>Pod IP: 10.65.0.24<br/>默认路由 via 169.254.1.1"]:::pod
  end
  subgraph HostNS["主机 network namespace"]
    Cali["caliXXXXXXXXX<br/>MAC: EE:EE:EE:EE:EE:EE<br/>Felix 在此端编程"]:::host
    RT["路由表 / iptables / eBPF"]:::host
  end
  Eth0 ==|"veth pair · 内核虚拟网线"| Cali
  Cali --> RT
```

各步骤按执行者分类：[1][3]

| 对象 | 执行者 | 作用 |
|------|----------|------|
| **veth 这对网卡** | **CNI 插件**（kubelet 调 CNI 时创建） | 一端入 Pod ns，一端留主机 ns；Pod 收发包的物理通道 |
| Pod 端接口名 | CNI 按 CNI args 的 `IfName`，通常为 `eth0` | Pod 内可见的网卡 |
| 主机端接口名 | CNI 命名为 `cali` + 容器 ID 前 11 位（老版本）或 `cali` + `sha1(namespace.pod)[:11]`（新版本），前缀 `cali` 是 Felix `InterfacePrefix` 的默认值 | Felix 据此识别 workload 接口 |
| 主机端 MAC | CNI 设为 `EE:EE:EE:EE:EE:EE` | 固定值，免去内核生成持久 MAC，并便于 Felix 统一处理 |
| Pod 端路由 | CNI 配置：默认网关指向 `169.254.1.1`（link-local），走 `eth0` | Pod 将所有流量发往 eth0，无须感知真实下一跳 |
| 主机端属性 | **Felix**：开启 IP 转发、令主机以自身 MAC 回答 ARP、将 `10.65.0.24` 路由指向该 `cali…` | 包进入主机即可被正确转发与过滤 |
| WorkloadEndpoint | CNI 创建后写入 datastore；记录 `interfaceName: cali…`、Pod IP、MAC、node 等 | Felix 据此识别归属并编程 |

故「**包从 Pod 出来，经 veth 进主机**」可拆解为：

1. Pod 将包发给默认网关 `169.254.1.1`，从 `eth0` 出去；
2. `eth0` 是 veth 的一端，包穿过内核虚拟网线，**出现在主机 ns 的对端 `cali…` 上**——即「进入主机」；
3. 主机内核接管：查路由表、过 iptables/eBPF、决定出口网卡。

关键：**veth 只是通道，不转发也不路由**。真正决定包去向的是主机路由表；Felix 的职责即保证此表与接口属性正确。Pod 一侧无须感知真实拓扑——仅将包发往 eth0。

以下三种场景，前两步相同：包从 Pod 出来，经 veth 进主机，主机以自身 MAC 回答 ARP。区别仅在主机查表后的出口。

### 3.1 同一台机器：A1 → A2

Node A 上 `10.65.0.24 → 10.65.0.25`。此路径不经物理网卡、不经隧道、转发不依赖 BGP。

```mermaid
sequenceDiagram
  autonumber
  participant A1 as Pod A1
  participant V1 as 主机 veth(A1)
  participant RT as Node A 路由表
  participant POL as iptables / eBPF
  participant V2 as 主机 veth(A2)
  participant A2 as Pod A2

  A1->>V1: src=10.65.0.24 dst=10.65.0.25
  V1->>RT: 进入主机命名空间
  RT->>RT: 命中本机路由：10.65.0.25 走 A2 的 veth
  RT->>POL: 过 NetworkPolicy
  POL->>V2: 放行
  V2->>A2: 进入 A2
  Note over A1,A2: 全程不离开 Node A；不经物理网卡、隧道、BGP
```

| 步 | 发生什么 | 谁事先写好的 |
|----|----------|--------------|
| 1 | A1 将包发给默认网关（veth 对端） | CNI 配置的 Pod 内路由 |
| 2 | 主机以自身 MAC 回答 ARP，包进主机 | Felix 的接口/ARP 设置 [2] |
| 3 | 路由表：`10.65.0.25` 直连，出口为 A2 的 veth | Felix 写入的本机路由 [2] |
| 4 | 策略链决定放行或丢弃 | Felix 写入的 iptables / eBPF |
| 5 | 包从 A2 的 veth 进入 A2 | — |

Node A 路由表中这两条均由 Felix 写入：

```text
10.65.0.24   dev cali…A1    # 本机 Pod
10.65.0.25   dev cali…A2    # 本机 Pod
```

### 3.2 跨节点、不封装：A1 → B

当底层识别 Pod IP（节点同二层，或中间路由器已习得这些地址），包以 Pod IP 原样转发。[5]

```mermaid
sequenceDiagram
  autonumber
  participant A1 as Pod A1
  participant NA as Node A 内核
  participant W as 物理网络
  participant NB as Node B 内核
  participant B as Pod B

  A1->>NA: src=10.65.0.24 dst=10.65.0.21
  NA->>NA: 查表：via 172.18.203.126 dev eth0
  NA->>W: 包仍为 Pod IP，下一跳 Node B
  W->>NB: 送到 172.18.203.126
  NB->>NB: 查表：10.65.0.21 走本机 veth（Felix 写入）
  NB->>B: 进入 Pod B
  Note over A1,B: 链路源/目的 IP 始终为 Pod IP；交换机须能转发非节点子网地址
```

| 位置 | 路由表示意 | 写入者 |
|------|------------|--------|
| Node A | `10.65.0.21 via 172.18.203.126 dev eth0` | 默认 BIRD（习得的远端路由）[2] |
| Node B | `10.65.0.21 dev cali…B` | Felix（本机路由）[2] |

链路源/目的 IP 始终为 `10.65.0.24 → 10.65.0.21`。交换机须能转发「目的地址不属于节点子网」的包。

### 3.3 跨节点、IPIP：A1 → B

当底层不识别 Pod IP，再封装一层节点 IP（IP-in-IP，IPv4 协议号 **4**）。隧道口通常为 `tunl0`。[5][7]

```text
内层（原包）   10.65.0.24   →  10.65.0.21
外层（封装头） 172.18.203.10 →  172.18.203.126   proto = 4
```

```mermaid
sequenceDiagram
  autonumber
  participant A1 as Pod A1
  participant NA as Node A 路由表
  participant T0 as Node A tunl0
  participant W as 物理网络
  participant T1 as Node B tunl0
  participant NB as Node B 路由表
  participant B as Pod B

  A1->>NA: dst=10.65.0.21
  NA->>NA: 查表：走 tunl0，下一跳 Node B
  NA->>T0: 内核追加外层节点 IP
  T0->>W: 外层 172.18.203.10 → 172.18.203.126
  W->>T1: 送到 Node B
  T1->>NB: 解封装，露出原包
  NB->>B: 本机路由送入 Pod B
  Note over T0,T1: 外层 proto=4；BGP 仍负责「下一跳是 Node B」
```

| 位置 | 路由表示意 | 写入者 |
|------|------------|--------|
| Node A | `10.65.0.0/26 via 172.18.203.126 dev tunl0` | 默认 BIRD；常见带 `proto bird` |
| Node B | `10.65.0.21 dev cali…B` | Felix |

IPIP **仅支持 IPv4**；Azure 会丢弃此类包，应改用 VXLAN。[4][5]

IPIP 开启时 BGP **默认仍在工作**：BGP 负责「下一跳是 Node B」；`tunl0` 负责「途中封装一层节点 IP」。二者并非互斥。[5]

### 3.4 三条路径对照

| | 同机 A1→A2 | 跨机不封装 A1→B | 跨机 IPIP A1→B |
|--|------------|-----------------|---------------|
| 出 Pod | veth → 主机 | 同上 | 同上 |
| Node A 查表 | 直连 A2 的 veth | `via Node B`，出物理网卡 | `via Node B`，出 `tunl0` |
| 物理链路上的 IP | 包不离开本机 | 仍是 Pod IP | 外层是节点 IP |
| 到对端 | 不经过 Node B | Felix 的本机路由进 Pod B | 先解封装，再走本机路由 |
| 本次转发是否用 BGP | 不用 | 不用（路由已事先习得） | 不用 |
| BGP 的作用 | 向其他节点通告 A1/A2 归属 | 事先将「B 在 Node B」写入 A 的路由表 | 同上 |

末行是关键：BGP 运行于**控制面、事先**；每个数据包仅查内核。

两端 iptables / eBPF 在同机与跨机时均会检查，与是否封装无关。[2]

## 4. 路由如何出现在每台机器上

第 3 节跨机转发成立的前提是：Node A 的路由表中**事先**已有「去 `10.65.0.21` 下一跳为 Node B」。本节阐述这些路由如何产生。

### 4.1 内核中与 Calico 相关的三类对象

| 内核对象 | 默认写入者 | 转发时使用者 |
|----------|----------------|--------------|
| veth 这对虚拟网线 | **CNI** 创建；Felix 再写 ARP、转发等属性 [1][3] | 包进出 Pod |
| 路由表 | **本机 Pod**：Felix；**其他节点上的 Pod**：默认 BIRD（VXLAN 池则为 Felix）[2][5] | 决定下一跳与出口网卡 |
| iptables / eBPF | Felix | 决定放行或丢弃 |

### 4.2 新建 Pod 后的流程

```text
1. kubelet 调 CNI：分配 IP，创建 veth
2. Felix 写入本机：
      接口（主机 MAC 答 ARP、开启转发）
      路由表添加「10.65.0.24 → 该 veth」
      策略规则
3. BIRD 从路由表读取此本机路由
4. BIRD 经 BGP 通告 Node B：「去 10.65.0.24，下一跳 172.18.203.10」
5. Node B 的 BIRD 将对端路由写入本机路由表
```

第 3～5 步仅发生一次（或随 Pod 增删而变化）。此后 **A1 访问 B 的每个包仅查内核，不再涉及 Felix 或 BIRD**。[2]

### 4.3 cluster routes 由谁写入

到其他节点的路由，官方称 cluster routes。默认分工：[5]

- IPIP 池与不封装的池：由 BGP（BIRD）分发；
- VXLAN 池：由 Felix 直接写入。

亦可改为全部由 Felix 写入（`FelixConfiguration.programClusterRoutes=Enabled`，Operator 中为 `clusterRoutingMode: Felix`）。此时集群内部可不运行 BGP；若仍需向机房交换机通告地址，则保留 BGP。[5]

### 4.4 BGP 拓扑

默认为节点间 **iBGP 全连接**（每两台间建立会话）。官方建议量级约 **100 台及以下**；更多则采用路由反射器。默认 AS 号为 **64512**。[4]

| 连接方式 | 含义 | 适用场景 |
|--------|------|------------|
| 节点全连接 | 默认启用 | 中小集群 |
| 路由反射器 | 普通节点连接少数反射器（常用两台做备份）。反射器**只传路由，业务包不经过它** [1][4] | 节点较多时全连接会话数过大 |
| 与机柜交换机对接 | 关闭节点互连，改与 ToR 建 BGP | 机房中希望集群外也能直接访问 Pod IP [3][4] |

查看 BGP 是否建立，使用 `calicoctl node status`，须在**目标节点上**执行，因其查询本机进程。[4] BGP 使用 TCP **179**。

---

## 5. 地址分配与封装开关

### 5.1 地址分配

Calico IPAM 按块分配给节点，IPv4 默认 **/26**（64 个地址）。块大小仅在创建 IP 池时设定。[6]

内核中仍可能出现到某 Pod 的 /32，此为 Felix 写入的本机转发条目；BGP 通告时尽量以整块发布，以减少路由条目。[2][6]

一块用尽后会再申请一块。若无空块，可能从其他节点已占用的块中**借用**地址，此时会为借用地址下发更细的路由，路由表随之膨胀。官方建议：池中块数不少于节点数。[6]

`disableBGPExport: true` 可禁止将该池地址通过 BGP 发出（v3.21 起）。[6]

### 5.2 是否封装

易混点：**IPIP 与 BGP 并非互斥。**

- BGP 回答：去该 Pod，下一跳是哪台机器。
- IPIP / VXLAN 回答：两台机器之间，是否将原包再封装一层。

故开启 IPIP 时，BGP 默认**仍在工作**：它通告「去这段地址，下一跳是对端节点」，本机转发时出口走 `tunl0`。[5]

| | 不封装 | IPIP | VXLAN |
|--|--------|------|-------|
| 链路上所见 | Pod IP | 外层节点 IP，内层原包 | 用 UDP 再封装一层，头比 IPIP 大 |
| 隧道口 | 无 | `tunl0` | `vxlan.calico` |
| 限制 | 底层须能转发 Pod IP | 仅 IPv4；Azure 不可用 [4][5] | 若集群中只有 VXLAN 池，内部可不运行 BGP [5] |
| 额外头部 | 0 | IPv4 约 20 字节 | IPv4 约 50 字节，IPv6 约 70 字节 [11] |

同一 IP 池不能同时开启 IPIP 与 VXLAN。[6] 切换封装模式可能中断已建立的连接，应安排在维护窗口。[5]

`ipipMode` / `vxlanMode` 三个取值：[6]

| 取值 | 实际含义（按官方） |
|------|--------------------|
| **Always** | 从启用 Calico 的主机访问该池中的容器/虚拟机时，均走该封装 |
| **CrossSubnet** | 仅当**目的节点 IP 与本节点不在同一子网**时才封装。节点所属子网记录于 Node 对象，一般由 `calico/node` 自行探测 |
| **Never** | 不使用该封装 |

可用 CrossSubnet 时，官方更建议采用：同一子网不封装，跨子网再封装。适合 AWS 多可用区、Azure VNet，以及「一组机器同二层、组间经路由」的网络。[3][5]  
Google 云为纯三层网络，**无** CrossSubnet 这一档，要封装则整网封装。[3]

安装时有两套「默认」，须区分：

- 用 Operator 安装，IP 池的 `encapsulation` **默认为 IPIP**（相当于 `ipipMode: Always`）；[9]
- 自行创建 `IPPool` 对象，字段默认值为 **Never**。[6]

开启 IPIP 或 VXLAN 时，官方建议同时开启 `natOutgoing`。否则 Pod 与主机之间的来回路径可能不对称，被反向路径检查丢弃。[6]

## 6. 选型与排查

### 6.1 选型：官方建议压缩版

先问一句：底层是否识别 Pod IP？识别则不封装，不识别再封装。Azure 不可用 IPIP。[3][4][5]

| 环境 | 更稳妥的做法 |
|------|----------------|
| 机房，能与交换机做 BGP | 不封装，与 ToR 交换路由；集群外也可直接访问 Pod |
| 机房，节点同二层 | 节点间 BGP、不封装；集群外仍到不了这些 Pod IP |
| 机房，底层不可控 | VXLAN CrossSubnet（或 IPIP CrossSubnet） |
| AWS，希望用 VPC 内 IP | Amazon VPC CNI 做网络，Calico 做策略 |
| AWS / Azure，VPC 地址不足 | Calico 自行组网 + VXLAN CrossSubnet |
| Azure，希望用 VNet 内 IP | Azure CNI + Calico 策略（不要 IPIP） |
| GCP，希望用 VPC 内 IP | 云厂商路由 + host-local IPAM，Calico 做策略 |
| GCP 上自行做 Overlay | VXLAN（或 IPIP）Always，无 CrossSubnet |
| 暂不与底层对接、先快速落地 | VXLAN CrossSubnet |

约一百台以上：BGP 不宜再用全连接，改用反射器或与交换机对接，并开启 **Typha**——它在 datastore（kdd 下即 kube-apiserver）与 Felix / confd 之间做一层缓存与去重，将「N 个节点各自 watch apiserver」收敛为「少数几个 Typha watch，再 fan-out 给数百个 Felix」，否则 apiserver 将被 watch 请求压垮。[1][4]

MTU 默认按当前封装自动计算。修改 MTU **仅对之后新建的 Pod 生效**。[11] 常见 1500 的网络：不封装用 1500，IPIP 用 1480，VXLAN 用 1450，IPv4 WireGuard 用 1440。[11]  
AKS 网卡常显示 1500，底层实际更接近 1400；开启 WireGuard 时须按 1400 再减 60，否则易丢包。[11]

### 6.2 排查顺序

按此顺序即可：**BGP 是否建立 → 路由下一跳是否正确 → 是否与封装设置一致 → Pod 是否可达 → 是否被策略丢弃。**

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 192.168.0.0/16
  blockSize: 26
  ipipMode: CrossSubnet    # Always | CrossSubnet | Never
  vxlanMode: Never
  natOutgoing: true
```

```bash
calicoctl node status              # 须在目标节点上执行
kubectl get ippool -o yaml
ip route get 10.65.0.21            # 从 Node A 查去 Pod B
ip route get 10.65.0.25           # 从 Node A 查去本机 A2
```

| `ip route get` 出口 | 通常表示 |
|---------------------|----------|
| 出口为本机 `cali…` / veth | 同机，或已到达目的节点本机 |
| `dev eth0` 且下一跳为对端节点 | 跨机、未封装 |
| `dev tunl0` | 跨机、IPIP |
| `dev vxlan.calico` | 跨机、VXLAN |
| 无到对端 Pod 的路由 | 远端路由尚未装上：查 BIRD / BGP / IP 池 |

再核对：安全组是否放行 TCP 179、IP 协议 4（IPIP）、UDP 4789（VXLAN）。日志位于 `calico-node` 容器，安装方式不同，命名空间亦不同。

---

## 7. 总结

Calico 的全部设计，可收成一句取舍：**把转发交还给内核，把路由当配置来分发。** 由此推出三条不可轻易动摇的原则：

| 原则 | 含义 | 落地 |
|------|------|------|
| **数据面即内核** | 转发只靠 Linux 路由表与 iptables/eBPF；无用户态进程在包路径上 | Felix 写表，内核查表 |
| **控制面即分发** | 「Pod IP 在哪台机器」是配置，用 BGP（或 Felix 直写）事先分发到每台节点 | BIRD / confd / datastore |
| **封装是退路，不是默认** | 底层识别 Pod IP 就不封装；不识别才退回 IPIP/VXLAN | `ipipMode`/`vxlanMode` 取 Never/CrossSubnet/Always |

由此也推出三条常见误读，须事先排除：

- **「datastore 即 etcd」**——在 K8s 默认安装中，Calico 经 kube-apiserver 读写 CRD，物理后端才是 etcd；唯有显式 etcd 模式才是直连专用 etcd（§2.1）。
- **「开了 IPIP 就不用 BGP」**——二者并非互斥：BGP 管「下一跳是谁」，IPIP 管「途中是否封装」（§3.3、§5.2）。
- **「Felix 转发数据包」**——Felix 只写表；真正转发的是内核。三个守护进程都不在包的路径上（§1、§3）。

> **收束**  
> 接受「底层网络未必识别 Pod IP」的现实；守住「转发交内核、路由当配置分发、封装作退路」三条；把选型落成「先问底层是否识别 Pod IP，再决定是否封装、是否运行 BGP、是否启用 Typha」。  
> 节点即路由器之意，不在多一层智能，而在把复杂的控制面收成可观察的表，让数据面回归最朴素的内核转发。

---

## 8. 参考文献

以 Calico Open Source **3.32** 为准。

| 编号 | 文献 / 文档 | 说明 |
|------|-------------|------|
| [1] | Tigera, *Calico Component Architecture* (v3.32). https://docs.tigera.io/calico/latest/reference/architecture/overview | Felix / BIRD / confd / datastore 职责 |
| [2] | Tigera, *The Calico Data Path*. https://docs.tigera.io/calico/latest/reference/architecture/data-path | 主机 MAC 答 ARP、本机路由、跨节点路由 |
| [3] | Tigera, *Determine the Best Networking Option*. https://docs.tigera.io/calico/latest/networking/determine-best-networking | veth、选型、云平台建议 |
| [4] | Tigera, *Configure BGP Peering*. https://docs.tigera.io/calico/latest/networking/configuring/bgp | BGP 拓扑、AS 号、Azure 与 IPIP |
| [5] | Tigera, *Overlay Networking (VXLAN/IP-in-IP)*. https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip | 封装模式；cluster routes 由谁写 |
| [6] | Tigera, *IP Pool Resource*. https://docs.tigera.io/calico/latest/reference/resources/ippool | `ipipMode`、`blockSize`、`natOutgoing` |
| [7] | RFC 2003, *IP Encapsulation within IP*. https://www.rfc-editor.org/rfc/rfc2003 | IP-in-IP 协议号 4 |
| [8] | Tigera, *Configuring calico/node*. https://docs.tigera.io/calico/latest/reference/configure-calico-node | 三个常驻进程、`CALICO_NETWORKING_BACKEND` |
| [9] | Tigera, *Installation API Reference*. https://docs.tigera.io/calico/latest/reference/installation/api | Operator 默认封装 |
| [10] | Tigera, *Calico over IP Fabrics*. https://docs.tigera.io/calico/latest/reference/architecture/design/l3-interconnect-fabric | 节点当路由器的设计前提 |
| [11] | Tigera, *Configure MTU*. https://docs.tigera.io/calico/latest/networking/configuring/mtu | 封装开销与 MTU 取值 |
| [12] | Tigera, *The Calico Datastore*. https://docs.tigera.io/calico/latest/getting-started/kubernetes/hardway/the-calico-datastore | kdd vs etcd 两种 datastore |

