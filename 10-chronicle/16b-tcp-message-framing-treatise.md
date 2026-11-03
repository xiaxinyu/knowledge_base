# TCP 消息分帧：粘包 · 半包 · 应用层边界

> 应用协议以离散消息为语义单位；TCP 在 socket 上交付的是**可靠、有序、无记录边界的字节流**。[^rfc9293][^tcp7]  
> 由此出现两类读出形态：多次 `send` 的内容被一次 `recv` 一并返回（粘包）；一次 `send` 的内容被多次 `recv` 分段返回（半包 / 拆包）。二者并非内核缺陷，而是应用层未在字节流上建立消息边界。  
> 工程对策称为 **message framing（消息分帧）**：由应用协议定义边界，接收端据此将字节流还原为消息。[^cleary-framing]
>
> 对照：[16a](./16a-tcp-ip-illustrated-treatise.md)（TCP 与 socket）、[16](./16-http-protocol-chronicle.md)（HTTP 定界）、[28](../20-architecture/28-http-rpc-microservice-communication-treatise.md)（RPC 内置分帧）。

**结论：**

> **`send` 与 `recv` 的调用次数无对应关系。** 在 TCP 上还原消息必须依赖应用层分帧。工业常用三类方案，并与 Netty 解码器同构：定长、分隔符、长度字段；自定义二进制协议宜优先**长度前缀**（4 字节网络序长度 + payload）。`TCP_NODELAY` 与 PSH 仅影响发送时机或交付提示，**不能**替代分帧。[^cleary-framing][^netty-len][^rfc9293]

**10-chronicle 位置：** [16a](./16a-tcp-ip-illustrated-treatise.md) 论述传输层字节流合同；本文论述应用层如何在该流上**界定并还原消息**。

## 摘要

RFC 9293 将 TCP 定义为 *reliable, in-order, byte-stream service*，并指出段边界与应用读写缓冲边界无对应保证。[^rfc9293] Linux `tcp(7)`：*TCP does not preserve record boundaries*。[^tcp7] 中文语境中的粘包与半包，是同一合同下的两种读出形态；Stephen Cleary 将补齐边界的工作称为 *message framing*。[^cleary-framing] Netty 以 `FixedLengthFrameDecoder`、`DelimiterBasedFrameDecoder`、`LengthFieldBasedFrameDecoder` 实现「先切帧、再解业务」。[^netty-frame] UDP 每次接收至多一个数据报，不存在同类流式粘包。[^udp7] 全文结构：失效场景 → 协议语义 → 现象界定 → 成因 → 分帧选型 → 长度前缀实现 → 误区。

**关键词：** message framing；粘包；半包；长度前缀；分隔符；定长；Nagle；TCP_NODELAY；PSH；Netty

---

## 目录

- [摘要](#摘要)
1. [典型失效场景](#1-典型失效场景)
2. [协议语义与边界](#2-协议语义与边界)
3. [粘包与半包](#3-粘包与半包)
4. [成因分层](#4-成因分层)
5. [三种分帧与选型](#5-三种分帧与选型)
6. [长度前缀实现](#6-长度前缀实现)
7. [误区与排查](#7-误区与排查)
8. [收束](#8-收束)
9. [本章要点](#9-本章要点)
10. [参考文献](#10-参考文献)

---

## 1. 典型失效场景

设应用协议约定五条消息 `"AAA"` … `"EEE"`，客户端各以一次 `send` 发出（每条 3 字节）。服务端若假定「一次 `recv` 对应一条消息」，且接收缓冲较大、读取相对滞后，则在回环地址上可复现：

```text
发送：5 × send，共 15 字节
接收：1 × recv → AAABBBCCCDDDEEE（15 字节）
```

若将整段缓冲当作单条消息解析，五条记录会被合并为错误输入。反之，服务端每次 `recv(4)`、客户端一次 `send` 19 字节时，同一条消息将拆成多次返回。

回环联调中，`send` 与 `recv` 偶发一一对应，易造成「TCP 保留消息边界」的错觉；跨主机、延迟升高或读侧变慢后，该假设失效。Cleary 将其列为面向 TCP 设计应用协议时最常见的误判之一。[^cleary-framing]

> **结论：** 正确实现须默认：`recv` 返回的是字节流片段，而非一次 `send` 的完整记录。

---

## 2. 协议语义与边界

| 层次 | 保证 | 不保证 |
| ---- | ---- | ------ |
| **TCP** | 连接存活且实现正确时，字节有序、不丢、不重 | `write`/`read` 次数对应；段边界 = 消息边界[^rfc9293] |
| **应用协议** | （须自行定义）消息起止位置 | — |
| **UDP** | 数据报边界 | 可靠与有序；超长可能截断并置 `MSG_TRUNC`[^udp7] |

RFC 9293 §3.7：应用可按消息粒度调用 `write`，但 TCP **不保证**所发 TCP 段边界、以及对端读缓冲边界与这些消息边界一致。[^rfc9293] Linux：*TCP does not preserve record boundaries*。[^tcp7]

流套接字上，`recv` 通常返回当前已到达、且不超过请求长度的数据，并不等待「凑满请求长度 *N*」。[^recv2] 因此接收端必须处理：单次不足一帧、单次超过一帧、以及跨多次调用拼齐一帧。

对照 UDP：每次接收至多一个数据报，边界由传输层保留；TCP 将消息定界完全交予应用。[^udp7]

> **结论：** 粘包并非 TCP 故障，而是将**字节流服务**误当作**消息报文服务**使用的结果。

---

## 3. 粘包与半包

| 术语 | 工程表述 | 接收表现 | 缓冲状态 |
| ---- | -------- | -------- | -------- |
| **粘包** | 多消息一次读出 | 一次 `recv` 含 ≥2 条应用消息的字节 | 多段应用数据已在接收队列，一次取走 |
| **半包 / 拆包** | partial read | 一次 `recv` 不足一条消息 | 请求长度偏小，或数据未到齐 / 跨段到达 |
| **消息分帧** | message framing | — | 在字节流上编码并解析消息边界[^cleary-framing] |

二者同源：流上不存在消息边界。差异仅在于接收队列中的可用字节数与本次 `recv` 请求长度。英文工程讨论通常直接指出缺乏 *message framing*，而非将 sticky packet 视为独立故障类型。[^cleary-framing]

更严格的半包情形：长度头自身被拆分——首次仅读到 4 字节长度字段中的前 2 字节。若实现假定「一次即可读完长度头」，将得到错误的 *L*，导致后续解析整体错位。接收状态机必须允许**长度头与 payload 各自跨越多次 `recv` 拼齐**。

```mermaid
flowchart LR
  classDef root fill:#eef6e8,stroke:#5a8a3a,color:#2f4f1f
  classDef a fill:#fff3e0,stroke:#b8860b,color:#5a4200
  classDef b fill:#e8f1fa,stroke:#3b6ea5,color:#1f3a5f

  R["TCP：无消息边界"]:::root --> S["粘包：一次读 ≥ 多条"]:::a
  R --> H["半包：一次读 < 一条"]:::b
```

---

## 4. 成因分层

成因宜分三层讨论，避免将 Nagle 误判为根因。

### 4.1 合同层（根因）

TCP 仅维护字节流。无论发送端是否合并段，接收 API 均按字节交付，而不按应用消息切分。未实施分帧时，粘包与半包均可出现。

### 4.2 发送层（加剧合并）

Nagle：存在未确认数据时，发送端可缓冲后续用户数据，直至 ACK 到达或可发出接近满尺寸的段，以降低小包比例。[^rfc9293][^rfc896] 与 delayed ACK 叠加时，小写合并更显著。[^rfc1122] Linux 通过 `TCP_NODELAY` 关闭 Nagle。[^tcp7]

Nagle 影响的是**数据上线时机**，并不引入消息边界。关闭后，小消息可能更快单独成段，但接收端仍可能一次取走已到达的多段数据；半包亦不会因此消失。

### 4.3 接收层（决定观测形态）

| 条件 | 更易观测到 |
| ---- | ---------- |
| 连续小写、读侧滞后、或 `recv` 请求缓冲较大 | 粘包 |
| `recv` 请求偏小，或消息跨多个段 / 分批到达 | 半包 |
| 丢包后按序补齐，积压数据一次性交付 | 一次读出多条 |

> **结论：** 调整 Nagle 属于延迟与小包次数的权衡；**建立消息边界**属于正确性要求。

---

## 5. 三种分帧与选型

Cleary：面向 TCP 的消息协议须做 framing；主路径为长度前缀与分隔符，定长可视作长度恒为 *N* 的特例。[^cleary-framing] Netty 将切帧置于管道前端，业务解码器仅处理完整帧：[^netty-frame]

| # | 方法 | 规则 | Netty | 适用 | 主要约束 |
| - | ---- | ---- | ----- | ---- | -------- |
| 1 | **定长** | 每帧固定 *N* 字节，读满再处理 | `FixedLengthFrameDecoder` | 结构固定、长度不变 | 填充开销；变长演进困难 |
| 2 | **分隔符** | 以约定字节序列结束一帧 | `DelimiterBasedFrameDecoder` | 文本行协议 | 载荷须转义或禁出分隔符；须限制最大帧长 |
| 3 | **长度字段** | 固定宽度长度头 + payload | `LengthFieldBasedFrameDecoder` | 二进制、变长、任意字节载荷 | 须校验长度上限；须约定端序 |

生产环境中的常见协议本身即含分帧，通常不表述为「防粘包」：

| 协议 | 定界方式 |
| ---- | -------- |
| HTTP/1.1 | 头部：CRLF；正文：`Content-Length` / chunked[^http11] |
| Redis RESP | CRLF；bulk string 携带长度[^resp] |
| HTTP/2 | 9 字节帧头（含长度）+ payload[^http2] |
| gRPC | 基于 HTTP/2，消息层另含 length-prefixed 帧 |

**选型：**

1. 行文本、宜人工检视 → 分隔符。  
2. 记录长度恒定 → 定长。  
3. 其余自定义二进制 → **长度前缀**。  
4. 同一连接上不宜中途更换分帧方案。

---

## 6. 长度前缀实现

### 6.1 帧格式

```text
| 4 字节长度 L（网络字节序，uint32） | L 字节 payload |
```

*L* 通常表示 **payload 长度**（不含 4 字节头）。两端须固定端序约定；跨语言实现优先 `htonl` / `ntohl`，或明确 big-endian。

### 6.2 发送

`send` 可能只发送部分字节；长度头与 payload 均须循环至写完。生产环境可用 `writev` 一次提交「头 + 体」，降低交错写入窗口。

```c
static int send_all(int fd, const void *buf, size_t len)
{
    const char *p = buf;
    while (len > 0) {
        ssize_t n = send(fd, p, len, 0);
        if (n <= 0) return -1;
        p += n;
        len -= (size_t)n;
    }
    return 0;
}

static int send_frame(int fd, const void *payload, uint32_t len)
{
    uint32_t net = htonl(len);
    if (send_all(fd, &net, 4) < 0) return -1;
    return send_all(fd, payload, len);
}
```

### 6.3 接收

每条连接维护累积缓冲。每次 `recv` 追加字节后尝试解析：

1. 已缓冲 &lt; 4 → 继续读取（长度头半包）。  
2. 解析 *L*；若 *L* 超过约定上限 → 断开连接（防止内存耗尽）。[^cleary-framing]  
3. 已缓冲 &lt; 4 + *L* → 继续读取（payload 半包）。  
4. 否则向上层交付一帧，自缓冲头部移除 `4+L` 字节，并**立即尝试解析下一帧**（处理粘包）。

```c
/* inbuf[0..filled)：未消费字节；MAX_FRAME：单帧 payload 上限 */
while (filled >= 4) {
    uint32_t len;
    memcpy(&len, inbuf, 4);
    len = ntohl(len);
    if (len > MAX_FRAME)
        break; /* 协议错误：长度越界 */
    if (filled < 4 + (size_t)len)
        break; /* 半包：等待后续字节 */
    deliver(inbuf + 4, len);
    memmove(inbuf, inbuf + 4 + len, filled - 4 - len);
    filled -= 4 + len; /* 缓冲中可能仍有完整帧 */
}
```

与 Netty `ByteToMessageDecoder` 一致：字节不足以构成完整帧时不得消费；构成一帧则切出并循环，直至不足。[^netty-frame]

### 6.4 安全与连接关闭

| 项 | 要求 |
| -- | ---- |
| **长度上限** | 对端可声明极大 *L*；分隔符方案亦可永不发送结束符。须设置 `maxFrameLength`（常见 1–16 MiB，依业务而定）并在越界时断连。[^cleary-framing] |
| **连接关闭** | `recv == 0` 表示对端优雅关闭，按连接结束处理，不得视为可重试瞬时错误。 |
| **框架集成** | 使用 gRPC / Netty codec 时，管道中须先完成切帧；业务层不得假定每次读回调即一条业务消息。 |

---

## 7. 误区与排查

| # | 误区 | 纠正 |
| - | ---- | ---- |
| 1 | `TCP_NODELAY` 可消除粘包 | 仅关闭 Nagle；接收语义仍为字节流[^tcp7] |
| 2 | PSH 可作为消息边界 | PSH **不是** record marker[^rfc9293] |
| 3 | 回环验证通过即证明 1 `send` = 1 `recv` | 偶合现象，不得上升为协议合同[^cleary-framing] |
| 4 | 粘包属于内核或网卡缺陷 | 字节流合同下的预期行为；缺失的是应用分帧 |

**排查步骤：**

1. 在 `recv` 返回点输出长度与十六进制内容，区分粘包与半包。  
2. 用 `strace` 核对 `send` / `recv` 调用次数假设。  
3. 用 `tcpdump` / Wireshark 观察 TCP **段**边界（段 ≠ 消息）。  
4. 确认上层协议是否已含分帧（HTTP、RESP、HTTP/2、gRPC）；若有，实现「读满一帧」，勿叠加第二套边界。  
5. 确认存在连接级累积缓冲、「不足则等待」逻辑，以及对 *L* 的上限校验。

缺乏边界时应补齐 framing；不宜将 `TCP_NODELAY` 作为首要手段。

---

## 8. 收束

TCP 保证字节可靠、有序到达；应用协议须另行解决字节如何切分为消息。粘包与半包是同一缺口的两种读出形态。工业对策为 message framing：定长、分隔符或长度字段，并与 Netty 三类 FrameDecoder 同构。自定义二进制宜采用长度前缀，接收端以状态机处理长度头 / payload 半包以及一读多帧。栈与 socket 见 [16a](./16a-tcp-ip-illustrated-treatise.md)；HTTP 应用层定界见 [16](./16-http-protocol-chronicle.md)。

---

## 9. 本章要点

1. TCP 提供有序字节流，**不**保留消息边界。  
2. 粘包与半包同源；对策为 **message framing**。  
3. 分帧三法：定长、分隔符、长度字段；二进制优先长度前缀。  
4. 接收路径须容忍：长度头拆分、payload 拆分、一次读取含多帧。  
5. 须校验最大帧长；`TCP_NODELAY` / PSH **不能**替代应用层分帧。

---

## 10. 参考文献

[^rfc9293]: E. Eddy (Ed.), *Transmission Control Protocol (TCP)*, RFC 9293, 2022（byte-stream；§3.7 Segmentation；PSH 非 record marker）. https://www.rfc-editor.org/rfc/rfc9293 。

[^tcp7]: Linux man-pages, *tcp(7)*. https://man7.org/linux/man-pages/man7/tcp.7.html 。

[^recv2]: Linux man-pages, *recv(2)*. https://man7.org/linux/man-pages/man2/recv.2.html 。

[^udp7]: Linux man-pages, *udp(7)*. https://man7.org/linux/man-pages/man7/udp.7.html 。

[^rfc896]: J. Nagle, *Congestion Control in IP/TCP Internetworks*, RFC 896, 1984. https://www.rfc-editor.org/rfc/rfc896 。

[^rfc1122]: R. Braden (Ed.), *Requirements for Internet Hosts — Communication Layers*, RFC 1122, 1989. https://www.rfc-editor.org/rfc/rfc1122 。

[^cleary-framing]: Stephen Cleary, *Message Framing*, 2009. https://blog.stephencleary.com/2009/04/message-framing.html 。

[^netty-frame]: Netty Project, *ByteToMessageDecoder*（Frame detection）. https://netty.io/4.1/api/io/netty/handler/codec/ByteToMessageDecoder.html 。

[^netty-len]: Netty Project, *LengthFieldBasedFrameDecoder*. https://netty.io/4.1/api/io/netty/handler/codec/LengthFieldBasedFrameDecoder.html 。

[^http11]: R. Fielding et al., *HTTP/1.1*, RFC 9112, 2022. https://www.rfc-editor.org/rfc/rfc9112 。

[^http2]: M. Belshe et al., *HTTP/2*, RFC 7540, 2015. https://www.rfc-editor.org/rfc/rfc7540 。

[^resp]: Redis Ltd., *RESP protocol spec*. https://redis.io/docs/latest/develop/reference/protocol-spec/ 。

[^unp]: W. R. Stevens, B. Fenner & A. M. Rudoff, *UNIX Network Programming, Volume 1*, 3rd ed., Addison-Wesley。

**声明：** 本文论述应用层消息分帧，非内核调参手册。行为以当期 RFC 与系统手册为准。
