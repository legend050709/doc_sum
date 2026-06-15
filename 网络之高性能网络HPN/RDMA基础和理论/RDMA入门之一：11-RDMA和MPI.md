```table-of-contents
```

# Tag Matching(TM：标签匹配)
## 背景

MPI 的点对点语义是 `MPI_Send(buf, dest, tag, comm)` / `MPI_Recv(buf, source, tag, comm)`；

**发送端和接收端并不是同时执行**。
允许:
- 接收方用通配符 `MPI_ANY_SOURCE`、`MPI_ANY_TAG` 接收消息;
- 发送和接收的发起顺序可以不一致(应用先 post 哪个 recv 不一定对应先到达的消息)。

这意味着网络层不能简单地"先到先服务"塞入缓冲区,而必须根据 `(source Rank, tag, communicator)` 三元组去匹配到正确的应用缓冲区。这与 RDMA 本身"按地址直接搬数据、绕过 CPU"的理念是冲突的——如果每条消息都要 CPU 介入做匹配判断,RDMA 省 CPU 的优势就被抵消了。Tag Matching 就是为了解决"如何在保留 RDMA 零拷贝/旁路 CPU 优势的前提下,完成基于标签的消息寻址"。


## 原理与机制

- 应用侧 post 一个 receive 时,会生成一条"posted list"条目,包含 `(tag, tag_mask, source, comm, buffer addr, length)`。
- 消息到达时,匹配引擎(软件或硬件)用消息头里的 `(tag, source, comm)` 去遍历/查找 posted list,按照 FIFO+通配符规则找到第一个匹配项。
- 命中:直接把数据放入该缓冲区(对于小消息是 eager 直接拷贝/DMA;对于大消息只是匹配到一个"控制头",见下文 Rendezvous)。
- 未命中:消息被放入"unexpected queue"暂存,等应用稍后 post 对应 recv 时再来匹配消费。



### Posted Receive Queue (PRQ)
Posted Receive Queue：已经发布但尚未完成的接收请求。应用提前调用 MPI_Irecv 之类的接口,把"我要接收什么样的消息"登记进来,此时数据可能还没到。

### Unexpected Message Queue (UMQ)
Unexpected Message Queue (UMQ)：消息已经到达,但还没有匹配的 receive 被 post,先暂存起来(连带数据,或者只存 envelope,数据另行处理)。


### 查找流程

新消息到达时,去 PRQ 里查找匹配项(支持精确匹配和带通配符的匹配,如 MPI_ANY_SOURCE、MPI_ANY_TAG);找到就直接消费,找不到就放进 UMQ,等后续有匹配的 receive post 进来时再去 UMQ 里捞。

匹配规则通常还要求**顺序保持**:对同一个 source,消息必须按发送顺序被匹配,这给查找算法增加了约束,不能简单地用 hash 表去重排。

### 我的理解
Tag Matching 的本质：在接收端找到与消息匹配的 Receive Request（MPI Recv WR）。
匹配条件是 ：Communicator + Source Rank + Tag；
匹配成功，则消息传递给 对应的 Receive Request。

## 软件实现

传统做法(MPICH、Open MPI ob1 等)是在 MPI 库内部维护两个链表——posted Receive queue 与 unexpected queue,每条消息到达时由 CPU 线性遍历做比较。

### 优缺点
（1）优点：
软件 Tag Matching 的优势是灵活,可以支持任意复杂的匹配语义。

（2）缺点：
每条消息到达都要 CPU 介入做查找、搬运、回复 ACK 等；在 UMQ 很长(大量乱序消息堆积)时,匹配本身会成为性能瓶颈,这是 MPI 程序里经典的"unexpected queue 过长拖慢通信"问题。

假设有100万条Receive Request，message到达时，匹配查找 Posted Queue，CPU开销巨大，比如：Cache Miss等。


## TM 硬件 Offload

为了消除上面的 CPU 瓶颈,NVIDIA/Mellanox 自 ConnectX-5 起在网卡固件里实现了 **Hardware Tag Matching(HW TM)**,典型通过 UCX 框架暴露(`UCX_TM_*` 相关特性,底层走 `mlx5` provider 的 accelerated verbs/DC transport):

- posted queue 直接维护在网卡内部(或网卡可访问的内存结构中);
- 数据包到达时，由网卡硬件做标签查找与匹配,匹配成功后硬件直接把数据 DMA 到目标用户缓冲区,不需要 CPU 参与,也不需要先放到中间 buffer 再拷贝；
- 匹配失败(unexpected)的情况：未匹配的消息会被硬件放入一个预先注册的"eager pool"缓冲区,等待软件后续认领。

所以 offload 通常是"常见路径走硬件,边缘情况走软件"的混合模型。这极大降低了小消息场景下的延迟和 CPU 占用,是 UCX、libfabric重点支持的优化路径。使得即使是大量小消息的点对点通信,也能维持接近"纯 RDMA 写"延迟的水平。

### 作用
（1）降低 CPU利用率，
（2）减少中断，Cache Miss，上下文切换
（3）提高消息率(Msg Rate)，尤其是大量小消息


# Rendezvous Protocol（RNDV Proto）
## 背景

## Eager 协议与 Rendezvous 协议
RDMA/MPI 通信一般按消息大小分两种协议:

- **Eager 协议**:发送方不等接收方确认,直接把数据(连带 envelope)一股脑发过去,接收方收到后如果有匹配的 receive 就直接用,没有就先缓存到 UMQ。适合小消息,因为可以把延迟降到最低(发送方不需要等任何握手)。

- **Rendezvous 协议**:发送方先发一个小的控制消息(只含 envelope,不含数据本体),等接收方匹配到对应的 recv、确认了目标 buffer 地址之后,真正的数据传输才发生,通常这一步直接用 RDMA Write 或 RDMA Read 完成 zero-copy 的大块传输。

### 为什么大消息要走 rendezvous 而不是 eager


**（1）避免不必要的内存拷贝**:
大消息如果走 eager,接收方在消息尚未匹配时只能先拷贝到一个临时 buffer,匹配上之后还要再拷贝一次到用户 buffer,两次拷贝对大消息代价很高。Rendezvous 因为提前知道了目标地址,可以直接用 RDMA 做 zero-copy。

**（2）避免无限制地占用接收方内存**:
如果所有大消息都无脑 eager 发送,UMQ 里可能堆积大量未匹配的大块数据,造成内存压力。Rendezvous 通过握手把这个"发多少、什么时候发"的控制权交给了接收方。

**（3）流控**:
rendezvous 本质上提供了一种背压(backpressure)机制——发送方必须等接收方 ready 才发实际数据。

因此，Rendezvous 要解决的核心问题是:**大消息传输如何避免预先大量分配缓冲区、避免中间拷贝**。

## 原理与机制


典型流程(以"Pull"模型即接收方发起 RDMA Read 为例,Open MPI/UCX 常见实现):

（1）发送方判断消息长度超过阈值(如几 KB ~ 数十 KB,可配置),改走 rendezvous 协议;
（2）发送方把待发数据所在的本地内存注册好(获得 rkey),通过一条小的控制消息 **RTS(Ready-To-Send)** 把 `(tag, length, rkey, addr)` 发给接收方;
 (3) 接收方用 RTS 里的 `(tag, source, comm)` 去做 Tag Matching,找到应用真正 post 的目标缓冲区;
(4) 找到匹配后,接收方直接发起 **RDMA Read**,把数据从发送方内存"拉"到自己匹配到的目标缓冲区——这一步是单边操作,完全不需要发送方 CPU 参与;
(5) Read 完成后,接收方再发一条小的 **Fin** 控制消息通知发送方,发送方收到后才能释放/复用发送缓冲区。

另一种是"Push"模型(CTS「clear to send」 后由发送方做 RDMA Write),思路类似,只是握手顺序和发起方相反。
整个过程中,只有 RTS/CTS/Fin 这些很小的控制消息走"软件可见"路径(也正是这部分需要 Tag Matching),真正的大块数据搬运走的是纯硬件单边 RDMA,不经过对端 CPU,不需要中间缓冲拷贝。

```bash
发送方                                    接收方
  |--- RTS (Ready To Send,只含envelope) -->|
  |                                         | 查找/等待匹配的 recv,
  |                                         | 确定目标 buffer 地址
  |<--- CTS (Clear To Send,带目标地址) -----|
  |--- RDMA Write 数据到目标地址直接完成 --->|
  |--- FIN (完成通知,可选) ---------------->|
```

### 软件实现
MPI 库需要处理的细节包括:

**（1）设定协议切换的阈值(eager/rendezvous threshold)**：

**（2）内存注册(pin page + 获取 rkey)的开销很大**：
Rendezvous 走 RDMA 必须先把内存注册到 NIC(获得 lkey/rkey),这个开销不小,所以很多实现会做内存池预注册、或者用 ODP(On-Demand Paging)来避免每次都注册。

**（3）RTS/CTS/Fin 状态机的维护**：
需要和 Tag Matching 模块联动(RTS 到达后要先完成匹配才能知道目标地址);

**（4）流控**：
避免大量并发 rendezvous 操作耗尽网卡资源(QP、内存注册数等)。

**（5）多 rail/多 QP 并发**:
大消息传输时,部分实现会把数据切片,通过多个 QP 或多个网卡并发发送,提升带宽利用率。

### 硬件 Offload
现代网卡(ConnectX-6/7 等)把 Rendezvous 的整个状态机也下放到硬件/固件里,常称为 **"Tag Matching + Rendezvous Offload"**:

RTS 控制消息到达后,网卡硬件自己完成 Tag Matching、自己生成对应的 RDMA Read、自己在完成后发出 Fin,整个过程 CPU 完全不参与(只在最初 post receive 和最终拿到完成事件时介入);

结合 GPUDirect RDMA/GPUDirect Async,甚至可以做到 GPU 直接发起/响应整个 rendezvous 流程,CPU 全程旁路,这对大模型训练里频繁的大张量点对点/集合通信尤为关键。


## Rendezvous 协议 和 Tag Matching
- **Tag Matching（TM）** 解决的是：**消息匹配问题（Message Matching）/ 这条消息该给谁 **
- **Rendezvous（RNDV）** 解决的是：**大消息传输问题（Large Message Transfer）**

Rendezvous：本质是 大消息传输协议，核心思想是两段式接收，先传输元数据，再传输数据部分。

### 两者的搭配与联系

Tag Matching 和 Rendezvous 解决的是两个不同层面的问题,但在实际通信库(Open MPI、MVAPICH、UCX)中是**配合使用**的:

- (1) Tag Matching 解决的是寻址/路由问题：这条消息该交给哪个应用缓冲区。
> 无论走 eager 还是 rendezvous,控制消息(eager 的完整消息,或 rendezvous 的 RTS)到达后都要先过 tag matching 这一关,确定它对应哪个用户的 recv 调用,或者暂存进 UMQ。

- (2) Rendezvous 解决的是数据路径问题：大块数据怎么高效搬运；
一旦 tag matching 确定了目标(无论是立即匹配还是延迟匹配),rendezvous 利用这个匹配结果获得目标地址,然后用 RDMA 原语做实际的零拷贝传输。

(3) 具体配合关系:
- **小消息(eager)**:数据本身随消息一起发送,Tag Matching 直接作用在这条完整消息上——匹配成功立即把数据放进目标缓冲区,匹配失败放进 unexpected queue。
- **大消息(rendezvous)**:Tag Matching 只作用在很小的 RTS 控制头上,先匹配出目标缓冲区地址,之后的大块数据搬运完全绕开匹配引擎,直接走单边 RDMA。也就是说,**Rendezvous 依赖 Tag Matching 来完成"握手"阶段的寻址,但匹配的对象只是控制信息,不是整块数据**。

rendezvous 的 RTS 消息本身就是要走 tag matching 逻辑的——它要在 PRQ/UMQ 里查找。如果硬件做了 tag matching offload,RTS 这种小控制消息可以直接被硬件匹配,匹配后硬件甚至可以直接触发后续的 RDMA 操作,把整个 rendezvous 流程的控制路径也下沉到 NIC,这是目前 smart NIC、DPU(如 BlueField)在做的进一步 offload 方向——不仅 offload 匹配,还 offload 整个状态机。

在硬件 offload 层面,这种配合被进一步固化:网卡的 HW Tag Matching 引擎不仅处理 eager 消息匹配,也处理 RTS 头的匹配,匹配成功后由网卡自动触发后续的 RDMA Read/Write 和 Fin——也就是前面提到的"Tag Matching + Rendezvous Offload"一体化方案,这是目前 HPC/AI 大规模互联里追求的"全程旁路 CPU"理想形态。

### 区别

|维度|Tag Matching|Rendezvous|
|---|---|---|
|解决的问题|消息到缓冲区的寻址/路由(基于 tag/source/comm,支持通配符)|大消息的高效零拷贝搬运(避免预分配大缓冲区和中间拷贝)|
|作用对象|每一条消息(无论大小)|仅超过阈值的大消息|
|是否依赖对方|可独立存在(eager 消息只需匹配,不需要握手)|通常依赖某种寻址机制(在 MPI/UCX 里就是 Tag Matching)来确定目标地址,但理论上也可以用固定地址寻址(如 NCCL 这种没有通配符语义、对等关系固定的场景就不需要 Tag Matching,只需要简单的握手/信号量同步)|
|CPU 参与程度|软件实现需要 CPU 遍历比较;硬件 offload 后几乎不需要|控制消息(RTS/CTS/Fin)需要 CPU 处理;数据搬运本身天然旁路 CPU(单边 RDMA),offload 后控制消息也可不经 CPU|
|典型开销来源|队列遍历、通配符比较|内存注册、握手往返延迟|



简单来说:**Tag Matching 是"找对地方",Rendezvous 是"搬得高效"**。在 MPI/UCX 这类支持通配符语义的通信库里,二者是配合关系——Rendezvous 的握手阶段要靠 Tag Matching 来完成寻址;而在没有通配符语义、点对点关系固定的场景(如 NCCL 的 ring/tree 集合通信),可以只用类似 Rendezvous 的握手机制做大块数据零拷贝传输,而不需要完整的 Tag Matching 引擎。

# 参考
```bash

```