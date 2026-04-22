```table-of-contents
```


# 消息级别的端到端流控(END-TO-END (MESSAGE LEVEL) FLOW CONTROL)

IB传输层，为**RC服务**提供了一种**端到端（或消息级别）的流控制能力**，响应方可利用此能力来优化其接收资源的使用。
本质上，请求方必须拥有适当的**信用 (credits)** 才能发送请求消息。


可以理解为：==RC服务类型下，IB传输层（硬件实现）会基于接收方（QP的Recv Queue，非SRQ）下的可用的WQE的数量作为Credict, 进行消息级别的端到端流控==。

> 注：如果发送端某个操作不消耗接收方的Recv WQE, 此时即使接收端没有可用的Recv WQE，发送方也是可以发送的。

## 信用的定义和作用

每个信用代表接收一个入站请求消息所需的接收资源。具体来说，**一个信用代表一个提交到接收队列的 WQE（工作队列元素）**。

> 注意：尽管存在接收信用，但这不一定意味着有足够的**物理内存**来接收整个入站消息。即使信用充足，仍可能遇到内存不足的情况。

### SRQ 禁用
**任何关联了 SRQ 的 QP 都将禁用端到端流控制机制**, 主要原因如下：

（1）由于共享接收队列 (SRQ) 允许一组接收队列从一个公共 WQE 池中提取资源，单个接收队列无法准确估计公共 WQE 池中的 WQE 数量
（2）SRQ中并不维护对端SQ的信息，缺乏向SQ主动发送credit的机制。

## 流控的特性与要求
- (1) 该端到端信用机制**仅适用于可靠连接 (RC：Reliable Connected) 服务**。
    
- (2) 端到端信用（credit）由**响应方的接收队列生成**，并由**请求方的发送队列消耗**。
信用的存在与否限制了发送端传输**将消耗接收端 WQE** 的请求（即 **SEND 请求**或带有 **Immediate Data 的 RDMA WRITE 请求**）的能力。
    
- (3) HCA（主机通道适配器）的接收队列必须生成端到端信用（除非 QP 关联了 SRQ）
> 对于未关联 SRQ 的 QP，HCA 接收队列必须生成端到端流控制信用。如果 QP **关联了 SRQ**，HCA 接收队列不得生成端到端流控制信用。

- (4) 信用是**按每条消息**发放的，**与消息的大小无关**。
    
- (5) 端到端信用作为**一个编码后的 5 位字段**携带在 AETH 中。
    
- (6) 响应方可以使用**非请求确认包 (Unsolicited Acknowledge Packet)** 向请求方异步发送信用（通过重发最近的确认包）

## 信用从响应方到请求方的传输 (TRANSFERRING CREDITS)
传输信用的机制有两种：
1. **附带信用 (Piggybacked Credits)：**
    
    - 在**正常的确认包**的 AETH 字段中传输信用。
        
    - 信用携带在 `AETH Syndrome[4:0]`字段中。
        
    - 只有当 AETH 中的 **MSN 字段也有效**时，才能附带信用。


**非请求确认包 (Unsolicited Acknowledge Packet)：**

- 从 PSN（数据包序列号）角度来看，非请求确认消息对请求方来说像是**最近一次肯定确认消息的重复**。
    
- 即使最近的肯定确认是 `RDMA READ` 或 `Atomic` 响应，它也始终使用 **Acknowledge 操作码**。
    
- 响应方可随时发送非请求确认包。
    
- 它**仅用于传输信用**，对请求方的其他状态没有影响（因为它是重复响应）。
    
-  非请求确认包的 MSN 字段必须有效。

## 连接协商：初始信用 (NEGOTIATING CONNECTIONS: INITIAL CREDITS)

对于建立的每个连接，**端到端流控**的使用（或不使用）是**针对每个方向单独确定的**。连接的**接收队列**的能力决定了该半连接的流控特性。


如果对端的接收队列发出信号表明它期望生成信用，则本端相应的发送队列必须遵守端到端流控规则。反之，如果接收队列发出信号表明它不会生成端到端流控信用，则相应的发送队列可以随意传输请求消息，无需考虑信用。

如果 CA 支持 SRQ（共享接收队列），并且 QP（队列对）提供RC服务并与 SRQ 关联，则该 QP 必须不生成端到端信用，并应在 `AETH Syndrome[4:0]` 字段中放置值 `5b11111`，以表明信用字段无效。

当接收队列处于 RESET（重置）状态时，传输层应将初始信用计数设置为零。一旦队列对转换到 `INITIALIZED（初始化）、RTR（接收就绪）、SQD（发送队列排水）或 RTS（就绪发送）状态`，每提交一个接收 `WQE（工作队列入口）`，它应增加其信用计数。一旦处于 RTR、SQD 或 RTS 状态，响应端可以利用非请求确认（unsolicited acknowledges）将这些信用传输给请求端。通常，非请求确认是通过重发最近发送的肯定确认包并更新信用字段来创建的。然而，在**初始化时**，由于尚未发送任何确认包，因此无法使用创建非请求确认的常规方法。因此，在初始化时，非请求确认是通过将初始 PSN 减去 “1”来创建的。因此，如果接收队列处于 RESET 状态时 PSN 初始化为 `0x000000`，那么初始非请求确认的 PSN 应为 `0xFFFFFF`。

![](attachments/Pasted%20image%2020251117144811.png)

## 响应方计算信用的算法
对于在**未关联 SRQ** 的 QP 上使用RC服务的 HCA:
**递增 (Increment)：** 每提交一个 WQE 到接收队列，信用计数递增。
**递减 (Decrement)：** 每接收到一个消耗 WQE 的入站请求消息时，信用计数递减。
**不调整：** 接收到 `RDMA READ`、`不带 Immediate Data 的 RDMA WRITE` 或 `ATOMIC Operation` 请求时，信用计数**不调整**（因为它们不消耗接收 WQE）。

注：如果接收队列与 SRQ 关联，无论可能有多少 WQE 提交到接收队列，响应端都不调整其信用计数。
对于生成的每个确认消息（无论是普通确认消息还是非请求确认消息），接收队列应将当前编码后的信用计数（如第 378 页表 50 所示）插入到 `AETH Syndrome[4:0]`字段中。例如，如果接收队列有五个可用信用，它应在 AETH 中插入 5 位值 `b00100`。它还包括其当前 MSN 值。如果 QP 与 SRQ关联，接收队列应在 AETH 中插入 5 位值 `5b11111`。


## 请求端行为

==信用的存在与否限制了发送端传输**将消耗接收端 Recv WQE** 的请求==（send/send with imm/ RDMA with imm 三种操作）的能力。

当发送队列没有可用信用时，其行为应按照下面的规定执行。

（1）**请求端始终可以发送不消耗接收 WQE 的请求**（不带 Immediate Data 的 RDMA WRITE 请求、RDMA READ 请求或 ATOMIC Operation 请求），无需考虑信用。

（2）特别是，请求端**不得在发送队列中搜索不消耗接收 WQE 的请求并打乱顺序进行传输**，也不得违反有关 `fenced WQEs`的规则。

![](attachments/Pasted%20image%2020251117144523.png)

可用信用被编码并承载在 `AETH Syndrome[4:0]` 中；`MSN` 承载在 AETH 的最低 3 个字节中。下表显示了每个有效编码信用所代表的实际信用数量。

![](attachments/Pasted%20image%2020251117151914.png)

从逻辑上讲，请求端将一个连续的发送序列号（SSN：Send Sequence Number）与提交到发送队列的每个 WQE 相关联。SSN 与响应端在每个响应包中返回的 MSN具有一一对应关系。因此，请求端将 MSN 解释为代表响应端完成的最新请求的 SSN。

![](attachments/Pasted%20image%2020251117152955.png)

请求端**每次收到包含有效信用的确认包**时，都会计算一个新的 LSN。请求端还会**动态调整 LSN**：对于它希望发送的**不消耗接收 WQE** 的每个请求（RDMA READ 请求、不带 Immediate Data 的 RDMA WRITE 请求或 ATOMIC Operation 请求），它会**将 LSN 加一**。这种调整机制允许请求端发送不消耗接收 WQE 的请求。

从逻辑上讲，MSN 加上信用计数的总和是请求端的 **限制序列号（LSN：Limit Sequence Number）**。
（1）请求端可以自由传输任何 **SSN 小于或等于**计算出的 **LSN** 的请求。
LSN 计算：LSN = MSN + Credit Value
（2）任何 **SSN 大于当前计算出的 LSN** 的请求被称为**受限（limited）**。
（3）如果响应端在 AETH 中返回“无效”代码而非信用计数，则请求端可以随意传输请求。
响应端使用“无效”代码来表明 AETH 信用计数字段无效，原因在于响应端不生成端到端信用。即使是生成端到端信用的响应端，也可以选择在 AETH 中发送“无效”代码。然而，一旦请求端从响应端收到“无效”代码，请求端可以选择忽略该连接上未来所有事务的 AETH 信用计数字段。因此，如果响应端在发出“无效”信号后又恢复返回有效信用，结果可能是不可预测的。

## 请求端行为 - 受限发送 WQE(LIMITED SEND WQES)

### 什么是受限 WQE
当请求端在其发送队列中遇到一个**没有可用信用**的 WQE 时，该 WQE 被称为**受限 WQE**。

### 发送队列在遇到受限 WQE 时的行为
下面的文字RDMA 规范中关于请求端（Requester）在信用不足时如何处理受限（Limited）WQE 的详细规则，这些规则确保了流控的遵守和事务的有序性。发送队列在遇到受限 WQE 时的行为应如下：

（1）对于不消耗 WQE 的请求：
如果受限请求 WQE 是 RDMA READ 请求、不带 Immediate Data 的 RDMA WRITE 请求或 ATOMIC Operation 请求，它可以正常发送，无需考虑信用的可用性。
**队头阻塞问题**：
正常的请求排序规则仍然适用（即，发送队列不得通过搜索已提交 WQE 列表来尝试查找非受限 WQE并打乱顺序发送）。
请求方发送不消耗 WQE 的请求后，会动态地将 LSN 递增 1，以允许发送下一个不消耗信用的请求。

（2）对于 SEND 请求：
发送队列最多只能传输该请求消息的一个数据包，然后必须停止并等待确认包。**为确保响应方响应，请求方必须在该数据包中设置 AckReq 位**。

（3）对于带 Immediate Data 的 WRITE 请求：
请求端可以在停止传输并等待确认包之前，传输整个请求消息。这是允许的。
因为**实际消耗接收 WQE 的是请求中包含 `Immediate Data` 的那个单个数据包**。为确保响应端会生成响应，请求端应在请求消息的最后一个数据包中设置 AckReq 位。

## 小结


# RC类型下基于消息级别的端到端流控存在的问题
## SRQ不支持
一是因为SRQ中的WQE为共享资源，共享SRQ的接收端无法向发送端的每个SQ分配一个准确的credit值。
二是因为SRQ中并不维护对端SQ的信息，缺乏向SQ主动发送credit的机制。

## 对头阻塞问题
受限制的SEND操作会阻塞后续的不消耗对方Recv WQE的 WRITE、READ操作。即使WRITE/READ不消耗对方的WQE，也无法绕过前面受限的SEND。

## 硬件层的流控不感知软件层的资源
如果RQ中WQE充足一直能接收QP数据，但上层应用不读取，会导致接收内存一直增加。

接收端的 recv buff 主要只用三个地方：
```bash
1> RQ/SRQ 中的 WQE：一个WQE带有一个SGE的话，就认为是占用了一个recv buff；
2> 网络侧的缓存：比如通讯库中的缓存，业务没有来得及读取。
3> 业务处理占用
```
recv buff 可能不够用，导致网卡无法接收更多的消息。主要的消耗点：
```bash
1> 通讯库中的缓存，业务没有来得及读取。
2> 业务处理过程中的占用，没有来得及释放。
```

# 大厂自定义通讯库的流控
## UCX中的流控
### 背景
RDMA RC（Reliable Connected）和 DC（Dynamic Connected）服务类型是可靠传输，每一个发出去的消息都必须被对端接收侧有对应的 Receive Work Request（RWR）来消费。
接收缓冲区有两种组织方式：

|   |   |   |
|---|---|---|
|模式|描述|RQ 资源归属|
|Private RQ（不使用 SRQ）|每个 RC QP 独占一个 RQ，大小 = rx_queue_len|每 QP 独立|
|SRQ（Shared Receive Queue）|多个 QP 共享一个 SRQ|全局共享|


UCX 始终使用 SRQ 模式，不使用 per-QP 的私有 RQ， 而SRQ是不存在基于Credict的Message级别的流控的，需要在软件层自己实现一套流控（FC)的机制。

#### RNR NAK 的危害
当接收方 SRQ 已无空闲 WR 时，硬件会返回 RNR NAK（Receiver Not Ready）：

RNR NAK 的代价极高：
- UCX 默认 RNR_TIMEOUT = 1ms，重传需等待 1ms+，高并发场景下，RNR 可导致吞吐量下降 10x~100x
- RNR 超过 RNR_RETRY_COUNT（默认 7 次）会导致 QP 进入 Error 状态，连接中断
- DC 模式下 DCI 被 RNR 阻塞，影响所有共享该 DCI 的 EP

### 基本概念
#### AM（Active Message，主动消息）

**是什么：** UCX 传输层最基本的消息发送原语。发送方主动发送一个带有 am_id（handler ID）的消息，接收方根据 am_id 自动调用对应的回调函数处理。

**为什么需要流控：** AM 发送时会消耗接收方 SRQ 中的一个 WR（Work Request）槽位。如果发得太快，SRQ 被填满，硬件返回 RNR NAK，性能崩溃。

**关键特性：** AM header 的 8 bits = **低5位（AM ID）+ 高3位（FC 控制信息）**，FC 控制信息是"免费搭车"的。

```c
// rc_ep.h:41-50
// AM header 编码：
// bit[0:4] = AM ID（最多32种 handler）
// bit[5]   = SOFT_REQ
// bit[6]   = HARD_REQ
// bit[7]   = FC_GRANT
```
#### EP（Endpoint，端点）
**是什么：** 代表一条到对端的逻辑连接。RC 传输层中每个 EP 对应一个 QP（Queue Pair）。
```c
struct uct_rc_ep {
    uct_rc_txqp_t       txqp;      // 硬件 TX QP（WQE 可用数）
    uct_rc_fc_t         fc;        // fc.fc_wnd：当前剩余信用（倒计数）
    uint8_t             flags;     // 待捎带的 FC 控制位（GRANT/SOFT/HARD）
    ucs_arbiter_group_t arb_group; // 发送被阻塞时的 pending 队列
};
```
**核心作用：** fc.fc_wnd 是整个流控机制的"心脏"——它记录了"我还能向对端发多少个 AM 而不需要等授权"。

Peer EP（对端端点）： 接收方视角中，代表"来自某个发送方的连接"。在 RC 中，它和本端的 EP 是一一对应的（共用同一条 RC QP 的两端）。

#### Grant（信用授予）
Grant：接收方告诉发送方"你的 fc_wnd 可以恢复了，继续发吧"的机制。Grant 是 FC 机制的"解锁钥匙"。
**收到 Grant 后的动作：**
```c
// rc_iface.c:380-395
cur_wnd = ep->fc.fc_wnd;
uct_rc_fc_restore_wnd(iface, &ep->fc);  // fc_wnd = fc_wnd_size（恢复满窗）
if (cur_wnd <= 0) {                       // 发送方之前被阻塞
    arbiter_group_schedule(ep->arb_group);
    arbiter_dispatch();                   // 唤醒 pending 队列继续发
}
```
##### FC_GRANT（信用授予标志，夹带/捎带「Piggybacked」形式）

**是什么：** 接收方在发出下一个 AM 时，在 am_id 的 `bit[7]` 上携带的授权信号。**不是独立消息，是搭载在正常 AM 上的**。

**触发时机：** 接收方曾收到过 SOFT_REQ 或 HARD_REQ，且 ep->flags 中有 FC_GRANT 标志。

**发出时的处理：**
```c
// 发送方（此时兼任"接收方授权发出者"）
am_id |= ep->flags & FC_MASK;  // 把 FC_GRANT bit 加入 am_id
// ...post WQE...
ep->flags &= ~FC_MASK;         // 清除标志，避免重复发
```

**接收方收到 FC_GRANT（含在 AM 中）：**
```c
if (fc_hdr & FC_GRANT) {
    ep->fc.fc_wnd = fc_wnd_size;  // 恢复满窗
    // 唤醒 pending（若之前被阻塞）
}
// 继续处理 AM payload（因为还有真实数据）
invoke_am_handler(real_am_id, ...);
```

##### PURE_GRANT（纯信用授予，独立消息）
**是什么：** 一个**没有任何 AM payload** 的独立消息，唯一用途就是授予信用。`am_id = 0xE0（SOFT_REQ | HARD_REQ | GRANT 三位全1）`。

**触发时机：**
1. 收到 HARD_REQ 时，接收方没有其他 AM 可以搭载 Grant → 必须发独立 PURE_GRANT
2. 收到 SOFT_REQ 后，长时间没有 AM 要发，信用请求还没被满足时也可能走到这里

**接收方收到 PURE_GRANT：**
```c
if (fc_hdr == UCT_RC_EP_FC_PURE_GRANT) {  // 0xE0 = 三位全1
    ep->fc.fc_wnd = fc_wnd_size;           // 恢复满窗
    // 注意：不调用 AM handler！纯控制消息
    return UCS_OK;
}
```

#### SOFT_REQ（软信用请求）

**是什么：** "我的信用快用完了，你有空的时候顺便给我补一下"。

**触发条件：** fc_wnd 精确等于 fc_soft_thresh（默认 256，即 wnd_size 的 50%）时，**边沿触发一次**。

**实现方式：** 在即将发出的 AM 的 am_id 上设置 `bit[5]`，完全不额外占用带宽。

**接收方响应：**
```c
// 接收方收到 SOFT_REQ：
ep->flags |= UCT_RC_EP_FLAG_FC_GRANT;  // 标记：下次发 AM 时捎带 Grant
// 不立即回，等到接收方自己发下一个 AM 时顺带
```

**特点：** 懒惰式（lazy）授权，最理想情况下零额外消息。

#### HARD_REQ（硬信用请求）

**是什么：** "我的信用快耗尽了，你现在必须立即给我！"

**触发条件：** fc_wnd 精确等于 fc_hard_thresh（默认 128，即 wnd_size 的 25%）时，**边沿触发一次**。

**实现方式：** 同样 piggyback(夹带/捎带) 在 AM 的 am_id 的 `bit[6]` 上。

**接收方响应：**
```c
// 接收方收到 HARD_REQ：必须立即发 PURE_GRANT
fc_req = mpool_get(pending_mp);
fc_req->func = uct_rc_ep_fc_grant;
status = uct_rc_ep_fc_grant(fc_req);  // 立即尝试发 PURE_GRANT
if (status == NO_RESOURCE) {
    push_head(ep->arb_group, fc_req); // 没资源就推队头（最高优先级）
    arbiter_schedule(...);
}
```

**特点：** 紧急授权，有可能触发一个额外的独立消息（PURE_GRANT）。

#### Piggyback：在数据前面添加AM头作为流控信息
Piggyback = 把 FC 控制信号"塞进"正常数据消息的 header 高位，零额外网络开销。

### 流程

![](attachments/image%20(9)%202.png)

### UCX DC 传输层流控流程
DC（Dynamic Connected）传输层与 RC 的本质区别：多个 EP 共享少量 DCI（DC Initiator QP），没有固定的 per-EP QP 用于回送 Grant。

这带来以下挑战：
```bash
1> 无反向 RC QP：无法像 RC 一样直接从同一条连接 piggyback GRANT
2> Grant 寻址问题：Grant 消息需要携带足够信息（ep 指针 + seq + GID）让接收方找到对应的 EP。
3> DCI 竞争：Grant 发送时 DCI 可能被其他 EP 占用；
```

#### DC是可靠的，为什么UCX的DC流控存在超时重传的逻辑

**（1）DC 的可靠性保证了什么，没有保证什么？**

|操作类型|DC 硬件是否保证可靠？|
|---|---|
|普通 AM / RDMA Write / Read|✅ 硬件 ACK + 重传|
|HARD_REQ 消息（从发送方 → 接收方）|✅ 可靠，DC 硬件保证|
|**PURE_GRANT 消息（从接收方 → 发送方）**|可靠，但发送路径依赖 fc_ep|

Grant 消息本身是可靠传输的，但发送 Grant 的 fc_ep 对应的 DCI 可能出故障！

```bash
发送方 EP₁: fc_wnd → 0，阻塞
  发出 HARD_REQ ──(DC可靠)──► 接收方收到 ✓

  接收方尝试用 fc_ep 发 PURE_GRANT:
    fc_ep 的 DCI 出错（Error状态）
    → Grant 没发出去！
    → 加入 pending 队列等 DCI 恢复

  DCI 恢复了，但是……
    fc_ep 的 DCI 在故障期间 WQE 全被 flush，
    pending 里的 Grant 请求需要被重新调度发送

发送方 EP₁ 永远等不到 Grant ← 死锁！
```

**(2) 超时重传的真正作用：防死锁**

超时重传的逻辑是："我发了 HARD_REQ，你那边虽然收到了，但你回 Grant 时 fc_ep 的 DCI 挂了，Grant 没发出来。我不知道这件事，所以我等了 5 秒后，主动再发一次 HARD_REQ 触发你重新发 Grant。"

**(3) 小结**：

|问题根因|说明|
|---|---|
|**DC 可靠 ≠ FC 系统可靠**|DC 保证单条消息投递可靠，但 FC 是一个多步协议（HARD_REQ → Grant），任何一步的**执行资源**（如 fc_ep 的 DCI）出错，整个协议就卡死|
|**fc_ep DCI 是单点**|所有 Grant 都经由同一个 fc_ep 发出，DCI 出错时 Grant 无法发出|
|**异步性**|发送方不知道接收方的 fc_ep 是否正常；接收方不知道发送方是否还在等 Grant|
|**防死锁**|没有超时重传，fc_wnd=0 的 EP 可能永久阻塞，整个通信死锁|

一句话：DC 本身的数据传输是可靠的，但 FC Grant 的"发送执行路径"（fc_ep 的 DCI）可能故障，发送端超时重传HARD_REQ 是防止这个故障导致整个 FC 协议死锁的保险机制。

### UCX 流控的 优点
**基本消除 RNR**：通过 fc_wnd_size ≤ rx_queue_len 的约束确保 SRQ 始终有空闲 WR。
**零额外 WQE（正常情况）**：FC 控制信息 `piggyback`(捎带/夹带) 在 `AM header` 高 3 位，不占用额外 `send/Recv WQE`。
**低延迟 Grant 机制**：SOFT_REQ：对端空闲时捎带（零额外消息）；HARD_REQ：立即触发独立 Grant；
**公平性保证**：每 EP 独立 fc_wnd，防止某个 EP 独占 SRQ 资源。


### UCX 流控的 问题
#### UCX FC 是有意识的不精确设计，而非精确流控

- SRQ 容量 = rx_queue_len = 4095，被所有 EP 共享
- fc_wnd_size（默认 512）是每个 EP 的最大 in-flight AM 数

```bash
// rc_iface.c:719-722
/* Assume that number of recv buffers is the same on all peers.
 * Then FC window size is the same for all endpoints as well.
 * TODO: Make wnd size to be a property of the particular interface.
 * We could distribute it via rc address then. */
self->config.fc_wnd_size = ucs_min(config->fc.wnd_size,
                                   config->super.rx.queue_len);  // min(512, 4095)
```



##### 为什么说FC不精确
**为什么说它不精确？**
假设有 N 个远端对等体（peer），每个 peer 对应一个 RC EP：
```bash
SRQ 容量 = 4095 个 WR（全局共享）
每个 EP 的 fc_wnd_size = 512

如果同时有 10 个 peer 都打满 fc_wnd：
  理论最大 in-flight AM = 10 × 512 = 5120 > 4095 (SRQ 容量)
  ← 理论上 SRQ 可能被耗尽！
```

##### UCX FC依靠什么保证实际安全

(1) 假设 1：通信不是全对全同时打满的

fc_wnd_size = 512 是每个 EP 的**最大 in-flight 上限**，但实际场景中：

- 多个 peer 同时向同一个节点发送 AM，且**全都打满 fc_wnd**，这种情况极少发生
- 典型 MPI AllReduce、点对点通信模式下，热点 peer 数量有限

这是一个**统计保证**，不是数学保证。

(2) 假设 2：接收方 progress 线程持续补充 WR

只要接收方 progress 在正常运转，每消费一个 WR 后立即 repost（srq_post_recv），SRQ 的瞬时 in-use WR 数量远小于理论最大值，实际上几乎不可能填满。

##### 理想精确FC

**理想情况下，fc_wnd_size 应该在建立连接时通过 RC 地址协商，==动态分配 SRQ 配额==**（即 SRQ_capacity / 连接数 = 精确的 per-EP 配额）。但 UCX 目前没有实现这个，使用的是静态全局配置。

```bash
## 精确流控的理想做法（UCX 未实现的 TODO）

接收方 SRQ 容量 = 4095
当前 EP 数量 = N

精确 per-EP 配额 = floor(4095 / N)
→ 建连时通过 rc_address 把这个值告诉对端
→ 每个 EP 的 fc_wnd_size = 动态计算值

优点：严格保证 SRQ 不被耗尽
缺点：EP 数量变化时（新连接/断开）需要重新协商，复杂度高
```

##### 小结

|UCX FC 实际做法|理想精确做法（TODO）|
|---|---|
|**fc_wnd_size**|静态配置，全局相同（默认 512）|动态协商，= SRQ容量/EP数|
|**精确性**|保守上限估计，统计安全|数学严格保证|
|**SRQ 耗尽风险**|极低（工程假设成立时）|零（数学保证）|
|**实现复杂度**|低|高（需要连接时协商）|
|**实际效果**|足够，生产中未见大规模 RNR|理论上更安全|

**一句话：UCX RC FC 是一个基于"peer 不会同时全饱和"工程假设的统计安全机制，而非数学精确的 SRQ 配额分配。代码注释里的 TODO 本身就承认了这个设计的局限性，但在实践中这已经足够了。**

#### client端的发送缓冲区(pending队列)没有资源限制
client端pending队列的长度没有限制，没有反压到上层应用；若上层应用持续发送，则内存会一直增加。需要队列长度限制并向上层应用通知。

#### server端的接收缓冲区没有资源限制
server端发送grant时没有检查接收能力（接收缓冲区的剩余空间），若上层应用一直不处理接收到的数据，则接收buffer的内存占用一直增加。 需要接收能力感知的grant策略。

### 竞品对比

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|方案|流控粒度|机制|保护对象|DC 支持|额外开销|
|UCX RC/DC FC|每 EP（即每个QP/连接）|软件信用窗口 + piggyback|AM（消耗接收方Recv WQE的操作）|（特有机制）|极低（3bit复用）|
|原生 IB Verbs|无（开发者自己管）|依赖 RNR retry|所有操作|❌|无|
|MVAPICH2 MPI|每连接|Prepost + RDMA Write token|MPI 消息|❌|较高（额外消息）|
|libfabric verbs|有限|依赖 RNR retry + 有限 CQ|部分操作|❌|中等|
|NCCL|不需要|仅用 RDMA Write（不消耗 RQ）|GPU 数据|部分|无|

### QA
#### UXC 的 FC 对 RDMA Write/Read 是否有流控保护
无需保护，也没有。
- RDMA Write：不消耗接收方 RQ（无 Receive Work Request），无 RNR 风险
- RDMA Read：由 `iface->tx.reads_available` 控制并发数量，与 FC 机制独立。
- AM（Active Message）：消耗接收方 SRQ WR的操作，需要 FC 保护。


## X-RDMA中的流控
X-RDMA: `Effective RDMA Middleware in Large-scale Production Environments`

（1）Seq-Ack窗口：server端上层应用处理完之后，会通过imm向client端发送ACK（接收窗口大小）。client端需限制已发送但server端未处理的消息。
（2）Flow Control：限制send_uncompleted < N，不满足的送入pending队列。

# 通讯库层流控机制
## 背景
### 问题一：接收端资源占用膨胀问题
如果RQ中WQE充足一直能接收QP数据，但上层应用不读取，会导致接收内存一直增加。最终导致接收资源不足问题。

### 问题二：RC服务使用SRQ时，没有基于消息级别的流控
CQ：CPU作为消费者，需要poll CQ才可以清空；
SRQ：CPU作为生产者，需要不断的填充Recv WQE。
现在的问题是：没有流控 以及 业务逻辑可能也在同线程中运行，有可能任务繁重或者阻塞，导致CQ满了或者SRQ为空。

一旦CQ满了或者SRQ为空，问题就比较严重。
CQ满了，这个CQ上关联的所有的QP都要重置。
SRQ为空，则会收到RNR NAK错误。

#### RNR 的后果
RNR NACK 的后果：
- 发送方 QP 进入 **RNR 退避（backoff）**，等待 RNR timer 超时后重传；
- RNR timer 默认为毫秒级（最大约 491ms），严重影响延迟和带宽；
- 大量 RNR 可触发 QP 错误，导致连接失败。
> 注：InfiniBand 硬件本身有 RNR 重试计数（默认 7 次），一旦耗尽即 QP 进入错误状态。

### 问题三：基于RC（非SRQ）的流控存在队头阻塞的问题
基于RC（非SRQ）的流控存在对头阻塞问题：
受限制的SEND操作会阻塞后续的不消耗对方Recv WQE的 WRITE、READ操作；即使WRITE/READ不消耗对方的WQE，也无法绕过前面受限的SEND。

### 问题四：共享QP的XRC/VQP方案存在公平性问题
为了减少QP的数量，使用了在VRC、VQP这种共享QP的模式。其中：VRC是线程级别的共享，VQP是进程级别的共享。
在VRC、VQP这种共享QP的模式下，共享QP（RC类型）的流控会导致连接不公平的问题，某一个连接过度消耗RQ中的WQE会导致所有的连接受影响。




# 参考
```bash

```