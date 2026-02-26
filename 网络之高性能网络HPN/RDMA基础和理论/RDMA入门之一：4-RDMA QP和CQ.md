```table-of-contents
```

# RDMA通信模型
## 概述
RDMA是**基于消息的传输协议**，数据传输都是**异步**操作。

RDMA一共支持三种队列，发送队列(SQ)和接收队列(RQ)，完成队列(CQ)。
SQ和RQ都属于WQ(work queue)。
另外，SQ和RQ通常成对创建，被称为Queue Pairs(QP)。

定义了 2 大类型的队列: WQ和CQ。
![](attachments/Pasted%20image%2020250323230221.png)

## WQ
WQ（Work Queue）：App 要收/发数据，就会放置一个 WR（Work Request）到 WQ 作为 WQE（WQ Element）。WQE 是 RNIC 硬件执行任务单元，包含了软件需要硬件执行的动作。RNIC 会获取到 WQE 进行处理。

### SQ 和 RQ

因为 RDMA 支持全双工通信，所以 WQ 进一步细分为 SQ 和 RQ，并称为 QP（Queue Pairs）。通信双方使用一对 QP，通过 BTH QPN 唯一标识，并以此创建 Channel。1 个 RDMA App 可以按需创建多对不同的 QPs 和 Channels。这些 QP 可以用于不同的通信目的，例如：使用不同的服务类型。

SQ（Send Queue）：存放 Send WQE。
RQ（Receive Queue）：存放 Receive WQE。


![](attachments/Pasted%20image%2020250323230314.png)

## CQ
CQ（Complete Queue）：RNIC 每处理完一个 WQE 之后，就会写入一个 CQE 到 CQ，App 从 CQE 中确认一个 WC（Worker Completion）。

# RDMA选择了SQ/RQ/CQ，而非传统的Ring Buffer？
参考：[# 为什么RDMA选择了SQ/RQ/CQ，而非传统的Ring Buffer？](https://mp.weixin.qq.com/s/8kR6MQCguLB4zG64lhsImw)

RDMA的设计选择往往被视为技术决定，但背后蕴含着深刻的工程哲学。这些选择反映了对分布式系统本质的理解，以及对性能、安全性和可扩展性的权衡。




# QP
## Work queue
## SQ(send queue)
## RQ(receive queue)

## WQE
WQE是RDMA系统中表达操作意图的基本单位。它不仅仅是数据描述符，更是操作语义的完整表达。

一个典型的WQE包含以下核心字段：
```c
struct ib_wqe {  
    uint32_t opcode;        // 操作码：SEND, RDMA_WRITE, RDMA_READ等  
    uint32_t flags;         // 标志位：立即数据、屏障等  
    uint32_t wr_id;         // 工作请求ID，用于匹配完成事件  
    uint32_t num_sge;       // Scatter-Gather条目数量  
      
    // 远程地址信息（用于RDMA操作）  
    uint64_t remote_addr;   // 远程虚拟地址  
    uint32_t rkey;          // 远程内存密钥  
      
    // 本地数据描述  
    struct ib_sge sge_list[]; // Scatter-Gather条目数组  
};
```


每个Scatter-Gather条目（SGE）描述一段连续的内存区域：==SGE的设计允许单个WQE描述非连续的内存布局，提高了内存使用的灵活性==。
```c
struct ib_sge {  
    uint64_t addr;      // 内存地址（虚拟或物理）  
    uint32_t length;    // 数据长度  
    uint32_t lkey;      // 本地内存密钥  
};
```
### WQE的语义层次

WQE的设计体现了多层次的语义：

#### 操作语义层

- **传输语义**：SEND（发送消息）、RECV（接收消息）
- **内存语义**：RDMA_READ（远程读取）、RDMA_WRITE（远程写入）
- **原子语义**：ATOMIC_FETCH_ADD、ATOMIC_CMP_SWAP等

#### 控制语义层

- **完成语义**：是否需要完成通知
    
- **顺序语义**：内存屏障、栅栏操作
    
- **立即语义**：附加的立即数据
    
#### 安全语义层

- **访问控制**：通过L-Key和R-Key验证权限
    
- **隔离保证**：Protection Domain的边界检查



## SRQ(shared receive queue)


### SRQ的相关属性
```c
struct ibv_srq_attr {
    uint32_t        max_wr; /* Requested max number of outstanding work requests (WRs) in the SRQ */
    uint32_t        max_sge; /* Requested max number of scatter elements per WR */
    uint32_t        srq_limit;  /* The limit value of the SRQ */
};

struct ibv_srq_init_attr {
    void               *srq_context; /* Associated context of the SRQ */
    struct ibv_srq_attr attr; /* SRQ attributes */
};

struct ibv_srq *ibv_create_srq(struct ibv_pd *pd, struct ibv_srq_init_attr *srq_init_attr);

LATEST_SYMVER_FUNC(ibv_create_srq, 1_1, "IBVERBS_1.1",
           struct ibv_srq *,
           struct ibv_pd *pd,
           struct ibv_srq_init_attr *srq_init_attr)
{
    struct ibv_srq *srq;

    srq = get_ops(pd->context)->create_srq(pd, srq_init_attr);
    if (srq) {
        srq->context          = pd->context;
        srq->srq_context      = srq_init_attr->srq_context;
        srq->pd               = pd;
        srq->events_completed = 0;
        pthread_mutex_init(&srq->mutex, NULL);
        pthread_cond_init(&srq->cond, NULL);
    }

    return srq;
}
```


**属性说明**：
- `max_wr`：SRQ 中允许的最大工作请求数（WR 数量上限）
控制 SRQ 的容量（队列深度）。
当 SRQ 中挂载的 WR 数达到 `max_wr` 时，再次调用 `ibv_post_srq_recv()` 会失败并返回 `ENOMEM` 或 `EINVAL`。
类似于 CQ 的深度或环形队列的长度。

- `max_sge`：每个 WR 中允许的最大 SGE 数量，一个 WR 可以包含多个 SGE，用于描述接收数据时散布到多个内存区域的情形。
控制每个接收 WR 的内存分散程度。
 决定 `ibv_recv_wr.sg_list` 的长度上限。
如果你 `post_recv()` 时传入的 `wr->num_sge > max_sge`，会失败（返回 `EINVAL`）。

- `srq_limit`：SRQ 的低水位线阈值（可选报警），==当 SRQ 中剩余的可用 WR 数量小于或等于这个值时==，HCA 会在 CQ 上生成一个 `SRQ limit reached event`。
提供 SRQ 快要空了的预警机制，当 WR 快耗尽时，驱动通过异步事件机制通知用户补充新的接收 WR。
如果不关心事件通知，可以将其设为 0（或不设置）。

```bash
init_attr.attr.srq_limit = 16;
表示当 SRQ 中剩下 ≤16 个 WR 时，HCA 会触发一个事件通知：


struct ibv_async_event event;
ibv_get_async_event(channel, &event);

事件类型为：IBV_EVENT_SRQ_LIMIT_REACHED
```

`max_wr` 和 `max_sge`的组合：==类似于一个链表数组，`max_wr`控制数组的最大长度（或者说队列的深度），`max_sge`控制每个链表的最大长度==。
如果是普通的QP，那么就会有发送队列(`send QUEUE`)和接收队列(`recv QUEUE`)。
同样有发送队列(`send QUEUE`)的`max_send_wr`和`max_send_sge`，以及
接收队列(`recv QUEUE`)的`max_recv_wr`和`max_recv_sge`。


### SRQ的特点
SRQ：让多个 具有相同接收数据格式 的QP 共享一组 Recv WR，减少内存消耗，提高接收端的 buffer 管理效率。



### SRQ的优缺点

#### SRQ优势
##### 节约内存
![](attachments/Pasted%20image%2020251103113452.png)

从字面的意思上看，SRQ可以节约内存，因为不需要为每个QP，都下发`outstanding recv WR`.
```bash
不使用 SRQ 时：每个 QP 需要预置大量接收 WQE（例如，为 N 个 QP 预置 M 个连续消息，需要 N × M 个 WQE），以避免队列耗尽。这会占用大量内存（每个 WQE 可能指向大块内存区域），即使队列闲置时也如此。
使用 SRQ 时：只需为共享队列预置 K × M 个 WQE（K << N，通常只需个位数），节省了 (N - K) × M 个 WQE 及其关联的内存。这减少了内存压力。

```
##### 批量处理
SRQ 让应用/驱动更容易做**批量 post（一次性 post 一批 recv WR），减少Doorbell 写次数**。

- 减少每条 WR 的 amortized 固定成本（一次 MMIO、一次驱动锁、一次驱动/内核上下文的处理代价），从而提升 IOPS。
- HCA 端可以更顺畅连续消费 WQE（pipeline 更高效，减少 context switch/lookup 成本）。


每个 QP 自带 RQ 时：
- 每个 QP 的 Recv 需要单独 `ibv_post_recv()`，HCA 需要维护多个 doorbell ring。
- 用户态驱动（libibverbs + mlx5 driver）要频繁写不同的 doorbell page（MMIO 操作）。

SRQ 模式下：
- 所有接收 WR 统一 post 到同一个 SRQ。
- Doorbell 写次数减少，驱动锁竞争也少。

##### 灵活的动态 WQE 管理
SRQ 支持“SRQ Limit Reached”异步事件，当剩余 WQE 低于水位线时通知应用及时补充，而非静态预置每个 QP 的完整 RQ。这避免了小连接场景下的过度资源浪费（例如，为少量 QP 预置大 RQ 仍会闲置内存），并在突发小流量时快速响应，防止微小延迟积累。

**避免流控制（Flow Control）引发的延迟**：
（1）在非 SRQ 模式下，如果接收队列（RQ）WQE 不足，发送端会触发流控制机制（例如 RC 连接中的背压），导致发送方暂停传输，增加端到端延迟（可能达微秒级）。
（2）SRQ 通过共享 WQE 池，让硬件自动从共享队列中消费“无主”WQE 处理传入数据，无需每个 QP 过度预置。这避免了流控制的“饥饿”问题，即使突发流量，也能保持平稳传输，显著降低尾部延迟（tail latency）。

##### 性能提升：避免队列的上下文切换

但是实际在测试过程中，发现即使连接数(QP数）很少，在Client开启SRQ，也是可以提升`IOPS`以及时延的。
这是因为：即使只有一个 QP，SRQ 的“无主 WQE”消费机制比独立 RQ 更高效——==硬件无需维护多个队列上下文，减少了 NIC 内部的队列轮询或状态检查开销==。

```bash
猜测是：
	SRQ 简化了硬件的队列管理：接收数据时，硬件直接从 SRQ 头部取出下一个可用 WQE，生成 Completion Queue Entry（CQE）通知应用，而无需切换多个独立 RQ。这减少了 NIC（网络接口卡）内部的上下文切换开销，进一步降低时延。

（1）HCA 内部 pipeline 更短
SRQ 模式减少了一次 per-QP 的 RQ 查表、索引更新和门铃同步路径：
在 ConnectX 设备中：
- 每个 QP 有独立的 RQ context（metadata、index、WQE pointer）。
- SRQ 模式下，多个 QP 共享同一个 SRQ context。
HCA 内部执行时可以直接通过 QP→SRQ 索引表找到 WQE，减少一次 context lookup。

（2）驱动与内核事件路径简化
在 SRQ 模式下，内核驱动不再为每个 QP 单独维护 recv context，而是统一绑定到一个 SRQ context。
这会减少：
- 内核上下文切换时的 metadata 同步；

```

![](attachments/Pasted%20image%2020251103113724.png)
```bash
上面是Client没有开启SRQ的数据，下面的是Client开启了SRQ的数据。
```


#### SRQ缺点

什么时候 SRQ 不一定有利？
- 如果每个 QP 的流量极不均衡，某个“猛流”QP 可能把 SRQ 的 WR 全部抢占，导致其它 QP 阻塞（应用层需要配额/策略）。
> 如果 SRQ 耗尽，没有 WR 可用，所有关联 QP 的 Recv 都会报错（严重）。
- 如果消息非常大（网络传输时间 >> 终端处理开销）， 终端处理开销占比小，SRQ 带来的改进就不明显。
- 如果你的驱动/设备对 SRQ 支持不好（事件丢失、post_srq_recv 非线程安全），可能反而带来问题。


需要应用层管理 WR refill 的逻辑。因此生产系统常常使用：
> **SRQ + 动态 WR 管理 + SRQ limit event**  

### UD模式下不需要SRQ
RC模式下，client和server端，可以任意一端启动SRQ或者都启动SRQ。
但是UD模式下，不会启动SRQ.

**SRQ** 是为**连接型**通信设计的，它需要处理多个会话（每个会话一个QP），并且依赖于对接收队列的管理和共享。而 **UD模式** 的设计哲学是不依赖于连接和会话的，它更像是简单的“数据包广播”，没有会话管理和复杂的连接状态跟踪。即：UD模式的无连接特性，每次传输的消息都是独立的，它不依赖于特定的接收端点。接收方并不会知道这些消息来自哪个发送方或哪个会话，因此它不需要特定的队列来区分每个连接的数据。因此，UD模式与SRQ的工作原理和设计目标不兼容。


#### UD模式下，共享没有意义

- UD本质是“多播/广播”式通信：一个UD QP已经能接收来自任意对端的包；
- 没必要多个UD QP再去共享同一SRQ。

##### 不可靠传输不需要 SRQ 的“抗饥饿”功能

- UD 无流控、无重传，RQ 耗尽就丢包（应用层处理）
- SRQ 的主要价值是“防止 RQ 饥饿导致流控”，在 UD 中无意义。

#### 为什么 RC 可以使用 SRQ？（UCX 积极启用）

**UCX 实现**：UCT_IB_SRQ_ENABLED 默认开启，RC transport（如 rc_mlx5, rc_verbs）自动使用 SRQ。

###### 1. 语义匹配：Send/Recv 模型天然支持“无主接收”

- RC 使用 Send/Recv 语义：发送方 ibv_post_send，接收方提前 ibv_post_recv（WQE）。
- 接收方不知道哪个 QP 会先到达消息 → **“无主 WQE”** 是合理设计。
- SRQ 允许所有 QP 共享一个 WQE 池，硬件自动匹配任意 QP 的传入 Send 消息 → **完美契合 RC 的多连接接收场景**。

###### 2. 资源效率：减少 WQE 预分配

- 不使用 SRQ：每个 QP 需预分配 N 个 WQE → 总计 num_QPs × N
- 使用 SRQ：只需预分配 depth_SRQ 个 WQE（通常远小于 num_QPs × N）
- UCX 在高并发存储（如 SPDK NVMe-oF）中可管理数千 QP，SRQ 节省内存显著。

###### 3. 避免流控阻塞

- RC 有信用机制（Credit-based flow control）
- SRQ 防止单个 QP 的 RQ 耗尽导致背压，保持低延迟



## QP 标识
### lid
#### sm_lid
`sm_lid` 指的是 **Subnet Manager Local Identifier**。它是 InfiniBand 网络中用于标识子网管理器（Subnet Manager, SM）的一种标识符。

### gid
#### 概述
**GID (Global Identifier)** 是RDMA网络中**128位全局唯一地址**，核心作用是为QP提供跨子网的可路由标识：
- **本质**：IPv6格式地址（RFC 2373）
- **绑定层级**：关联到HCA（Host Channel Adapter）的**物理端口或虚拟端口**
- **关键特性**：支持路由（RoCEv2/IB）或本地通信（RoCEv1）

#### 介绍

GID 即全局ID（global id），是Infiniband网络层（Network Layer）的地址，用于在跨子网时做路由。在Infiniband协议中，它的作用跟TCP/IP协议中的网络层地址即IP地址的作用是一样的。
因为目前在市场占据绝对主流的RDMA软件栈——OFED中，Infiniband/RoCE/iWARP共用一套代码，所以即使RoCE v2的网络层为IP协议，但是从使用RDMA软件栈的用户来讲，也使用GID来指定本端和对端的地址。
在RoCE v2的通信中，硬件需要知道要知道对端的地址信息，包括：目的GID（即目的IP）、目的MAC地址以及VLAN等等。这些信息都可以通过用户传入的DGID（Destination GID）直接或间接获取到；因为驱动程序会将GID转换为IP地址，然后通过网络侧的邻居表获取MAC地址和VLAN ID。

#### 作用

|**场景**|**作用**|
|---|---|
|**跨子网路由**|在IB和RoCEv2中实现子网间通信（核心价值）|
|**本地通信**|RoCEv1和IB子网内使用Link-Local GID|
|**多播支持**|通过Multicast GID（ff00::/8）实现组播通信|
|**协议兼容**|RoCEv2通过IPv4-embedded GID（::ffff:0:0/96）支持IPv4通信|
|**多路径容灾**|多GID绑定不同物理路径，实现负载均衡和故障切换|


#### GID的分类与生成机制

##### 通用分类

|**类型**|**前缀**|**作用域**|**生成方式**|
|---|---|---|---|
|**Link-Local**|`fe80::/10`|单子网内（不可路由）|HCA自动生成（基于端口GUID）|
|**Unique-Local**|`fd00::/8`|私有网络内（可路由）|管理员/SM配置|
|**Global**|`2000::/3`|互联网范围（可路由）|公网IPv6地址分配|
|**Multicast**|`ff00::/8`|多播组|应用按需创建|
|**IPv4-Embedded**|`::ffff:0:0/96`|IPv4 over RoCEv2|自动映射（`::ffff:192.168.1.1`）|

##### 协议差异

|**协议**|**GID意义**|**生成特点**|
|---|---|---|
|**InfiniBand**|子网间路由标识|SM（子网管理器）分配Unique/Global GID|
|**RoCEv1**|仅本地标识（无路由）|纯Link-Local GID（基于MAC生成）|
|**RoCEv2**|=IPv6地址（实际路由层）|自动映射IPv4/IPv6地址到GID|

|**操作**|**InfiniBand**|**RoCEv1**|**RoCEv2**|
|---|---|---|---|
|**GID配置**|SM管理|自动生成|跟随IP配置|
|**路由依赖**|IB路由器|无（仅二层）|IPv6路由器|
|**地址解析**|ibv_query_gid()|同左|可用getaddrinfo()|
|**IPv4支持**|不直接支持|不支持|通过IPv4-embedded实现|

|**层**|**InfiniBand**|**RoCEv2**|
|---|---|---|
|**网络层**|IB GID路由|IPv6路由|
|**传输层**|IB BTH头|UDP头 + IB BTH头|
|**链路层**|IB链路协议|以太网帧|

> 📌 **RoCEv1特殊定位**：  
> 在以太网链路层直接封装IB传输层（无IP头），故**GID无实际路由功能**

> ✅ **关键差异**：
> 
> - RoCEv1 **无路由能力** → GID仅本地有效
>     
> - RoCEv2 **依赖IP路由** → GID=IP地址
>     
> - IB **独立路由体系** → SM管理GID路由表




#### 多gid下gid的选择
**选择的原则**
（1）一个RDMA设备可能有多个端口（但通常我们使用端口1）。
（2）每个端口可能有多个GID（Global Identifier），每个GID对应一个IP地址（IPv4或IPv6）。
（3）在连接建立时，客户端和服务器端需要选择兼容的GID（例如，相同的IP版本和网络可达性）。
（4）常见的做法是选择与连接对端IP地址类型相匹配的GID（例如，如果对端是IPv4，则选择IPv4 GID；如果是IPv6，则选择IPv6 GID）。
（5）另外，我们可能希望选择与对端IP地址在同一个子网的GID，或者选择RoCEv2的GID（因为RoCEv2同时支持IPv4和IPv6，但实际上RoCEv2使用IPv4或IPv6作为网络层，但数据包格式是固定的）。


```mermaid
graph TD
    A[通信需求] --> B{目标地址类型}
    B -->|IPv4| C[选择IPv4-embedded GID]
    B -->|IPv6| D[选择同子网IPv6 GID]
    A --> E{是否跨子网}
    E -->|是| F[Unique/Global GID]
    E -->|否| G[Link-Local GID]
    A --> H{多路径/容灾}
    H --> I[绑定多个GID Index轮询/故障切换]
```

#### api接口
```c
ibv_query_gid
ibv_query_gid_ex
ibv_query_gid_table
ibv_query_device
ibv_query_port
```
##### 数据结构
```c
union ibv_gid {
    uint8_t         raw[16];
    struct {
        __be64  subnet_prefix;
        __be64  interface_id;
    } global;
};

enum ibv_gid_type {
    IBV_GID_TYPE_IB,
    IBV_GID_TYPE_ROCE_V1,
    IBV_GID_TYPE_ROCE_V2,
};

struct ibv_gid_entry {
    union ibv_gid gid;
    uint32_t gid_index;
    uint32_t port_num;
    uint32_t gid_type; /* enum ibv_gid_type */
    uint32_t ndev_ifindex;
};
```
##### `ibv_query_device`
##### `ibv_query_port`
##### `ibv_query_gid` 或 `ibv_query_gid_table`


### qpn
QPN是一个节点本地某个QP的唯一编号。既然RDMA通信的基本单位是QP，自然只通过GID找到对端节点是不够的，还要能够找到指定的QP才能进行数据交换。

#### gid 和 qpn结合
如下图所示，==通过GID + QPN的组合，我们就能在网络中唯一确定一个QP了==（通过GID + QPN的组合定位目的QP）：
==类似于TCP/IP中，通过`ip+port` 标识远程的一个服务程序/Socket==，`IP`标识远端主机的一个接口地址，`Port`标识服务或`socket`。
同理，`GID`标识远程主机的一个接口地址，`QPN`标识某个程序中的`QP`。

![](attachments/Pasted%20image%2020250623120044.png)

对于RC服务类型来说，在节点间通过基于`Socket`的数据交互掌握了彼此的`GID + QPN`之后，会通过`Modify QP`的动作将相关信息记录到`QPC(QP context)`中。一切就绪之后，就可以进行`Send/Recv`双边操作了，但是如果想要进行单边的`RDMA Write/Read`操作，这些信息是还不够的。


## 其他
### QPN分Send Queue和receive Queue吗？
QPN 是一个整体编号，标识的是整个 Queue Pair，而不是单独的 Send Queue 或 Receive Queue。
因此，`QPN`不分`Send Queue`和`receive Queue`；但是`PSN`其实是分`send psn`和`recv psn`。如下所示。

```bash
# /opt/mellanox/iproute2/sbin/rdma resource show qp
link mlx5_bond_0/1 lqpn 30278 rqpn 39296 type RC state RTS rq-psn 164604 sq-psn 658420 path-mig-state MIGRATED pdn 128 pid 61655 comm kucl-poll-event
```

### SRQ情况下，QPN的值是什么样的？
SRQ（Shared Receive Queue）允许多个 QP 共享同一个 Receive Queue，以==节约内存，但是不能节省QP的数量==。

在 SRQ 情况下：
（1）每个 QP 仍然有自己的 QPN。
QPN 是分配给 QP 的标识，与是否使用 SRQ 无关。

（2）所有使用 SRQ 的 QP 的接收操作都使用 同一个共享 RQ（Receive Queue），但每个 QP 还是有：
- 自己的 Send Queue；
- 自己的 QPN；

### XRC服务类型下，QPN的值是什么样的？


# QP连接
## QP 建链
建立一对 QP 之间的 Channel，过程中协商通信参数。包括：

（1）GID（Global Identifier，全局 ID）：GID 是 IB 网络的唯一标识。IB 网络中用于标识和寻址网络中的节点或端口。
（2）QPN：QP 的唯一标识，确定建链的对象，GID+QPN 可以在 IB 网络中确定唯一的一个 QP。
（3）VA（虚拟地址）：App 希望访问的虚拟地址。
（4）rkey：remote addr key, 同上。
（5）qkey：queue key, 是 UD（不可靠数据报）服务类型中专用的 Key，用于校验数据报的合法性。

### 链路和连接
QP 建立 “链路（Channel）” 和 “连接（Connection）” 是两个不同的概念。

RDMA 支持 4 种基本的服务类型，以满足不同服务对可靠性和传输速率的不同需求。
![](attachments/Pasted%20image%2020250323231959.png)

其中，RC、UC 是存在 Connection 的，而 RD、UD 则不存在 Connection，而是直接传输 Datagram。

![](attachments/Pasted%20image%2020250323232050.png)

RC 服务类型类比 TCP 协议，进行通信的 QP 之间需要建立一对一 Connection。RC 通过 ACK 确认、重传、保序等机制，确保数据能在 QP 间进行有序、可靠的传输，适用于对数据可靠性和完整性较高的场景。但相对的，由于连接机制和可靠性保障机制的存在，导致 RC 的通信开销较大。当节点数增加时，将占用更多的网卡和内存等资源。




### 作用

# QP状态以及状态机
参考：
[# QP state machine](https://www.rdmamojo.com/2012/05/05/qp-state-machine/)
[# Using the QP states](https://www.rdmamojo.com/2012/05/10/using-the-qp-states/)
[InfiniBand协议学习（2）---- 软件传输接口](https://zhuanlan.zhihu.com/p/110898225)

## 状态机

![](attachments/Pasted%20image%2020250930141546.png)


![](attachments/Pasted%20image%2020250930142725.png)


## QP状态
### QP各个状态下的行为

![](attachments/Pasted%20image%2020250930143106.png)


### ibv_qp_state
```c
// RDMA的QP状态机的各个状态
enum ibv_qp_state {
	IBV_QPS_RESET,
	IBV_QPS_INIT,
	IBV_QPS_RTR, /* RTR: ready to receive */
	IBV_QPS_RTS, /* RTS: ready to send */
	IBV_QPS_SQD, /* Send Queue Drain */
	IBV_QPS_SQE, /*  Send Queue Error */
	IBV_QPS_ERR,
	IBV_QPS_UNKNOWN
};
```

### reset状态(IBV_QPS_RESET)

![](attachments/Pasted%20image%2020250930143617.png)

![](attachments/Pasted%20image%2020250930111600.png)


### init状态(IBV_QPS_INIT)

![](attachments/Pasted%20image%2020250930143809.png)

![](attachments/Pasted%20image%2020250930114932.png)



### ready to receive 状态(IBV_QPS_RTR) 和 ready to send 状态(IBV_QPS_RTS)

![](attachments/Pasted%20image%2020250930144222.png)

![](attachments/Pasted%20image%2020250930115111.png)


### SQD（send queue drained） 状态和 SQE（Send Queue Error）状态

![](attachments/Pasted%20image%2020250930150755.png)

![](attachments/Pasted%20image%2020250930115325.png)



### err状态(IBV_QPS_ERR)

![](attachments/Pasted%20image%2020250930151254.png)


#### 背景
当需要关闭`QP`时，应用程序可能需要调用`ibv_destroy_qp`来销毁`QP`，但在这之前应该确保所有`WQE`都被处理完毕，以避免数据丢失或泄漏。

#### 分析
==当`QP`被通过`ibv_modify_qp`设置为错误状态（IBV_QPS_ERR）时，触发硬件`RNIC`中止所有未完成的`WR`以及停止接收新的`WR`，未完成的`WR`会生成相应的完成事件「`cqe`」。这时软件在`poll_cq`时，通过`cqe`信息，查找到资源，可以安全地释放资源，因为硬件`RNIC`已经停止了对这些内存的访问==。

> 注：每个未完成的`WQE`都会在关联的`CQ`中生成一个完成事件「WC」，状态码通常是`IBV_WC_WR_FLUSH_ERR(值为5)`。用户需要轮询`CQ`来处理这些`WC`，确认所有`WQE`都被正确清理后，才能通过`ibv_destroy_qp`安全销毁`QP`。应用程序需要处理这些`WC`完成事件，以避免资源泄漏。

#### 问题场景与风险
**（1）场景描述**
- 软件在销毁 QP 时，未等待 RNIC 完成所有未完成的 WR（Work Request），直接通过 `wr_id` 释放对应的内存。
- 此时 RNIC 可能仍在通过 DMA 访问该内存（例如正在传输数据或未完成 ACK 确认）。

**（2）风险**
- 内存访问冲突：RNIC 的 DMA 引擎可能正在读写已被释放的内存区域，导致数据损坏、段错误（Segmentation Fault）或硬件异常。
- 数据完整性破坏：若内存被释放后重新分配，新数据可能被 RNIC 的残余 DMA 操作覆盖。

**（3）何时可能出现该问题**
以下操作可能引发问题：
- **过早释放内存**：
在未确认 WR 完成的情况下，直接调用 `ibv_destroy_qp()` 销毁 QP，或手动释放内存。

- **未处理异步事件**：
未正确处理 CQ 事件；
风险：未轮询的完成事件可能导致 WR 残留，RNIC 可能仍持有内存的 DMA 引用。
解决：销毁 QP 前必须清空关联的 CQ中 该QP下的 WR 对应的 CQE。

- **QP 状态迁移不当**：
未将 QP 迁移到 Error State 就尝试销毁，导致 RNIC 继续处理 WR。

##### 同步机制与底层原理
**（1）硬件与软件协作**：
- RNIC 在 QP 进入 Error State 后，会立即停止处理 WR，并生成错误完成事件。
- 轮询 CQ 的行为等同于同步点，确保软件在释放内存前，RNIC 已彻底停止 DMA 操作。

**（2）内存序与 DMA 可见性**：
- 现代 RNIC 通过 PCIe 内存屏障（Memory Barrier）保证 DMA 操作的原子性。
- 内存释放（如 `free()`）需在 DMA 操作完成后执行，避免释放后 DMA 写回（如 Write-Back）。

#### 安全的关闭QP的步骤

**1》修改状态**
通过`ibv_modify_qp`将`QP`状态修改为`IBV_QPS_ERR`，这会停止使硬件停止处理队列中的`WQE`，并开始`flush`刷新未完成的`WQE`。

**2》轮询CQ，释放对应的资源**
未完成的`WQE`会通过关联的完成队列（CQ）生成WC完成事件，`wc` 的 `status`为 `IBV_WC_WR_FLUSH_ERR(值为5)`。
轮询`CQ`来处理这些`WC`，基于`WC`中的信息「比如：`wr_id`」，查找到资源，进行资源的释放。

**3》销毁QP**
确认所有`WQE`都被正确清理后，软件才可以调用`ibv_destroy_qp`安全地销毁`QP`。

##### 范例
```c
#include <infiniband/verbs.h>

void safe_destroy_qp(struct ibv_qp *qp, struct ibv_cq *cq) {
    struct ibv_qp_attr qp_attr = {
        .qp_state = IBV_QPS_ERR
    };

    // 1. 设置 QP 为错误状态
    if (ibv_modify_qp(qp, &qp_attr, IBV_QP_STATE)) {
        fprintf(stderr, "ERROR: Failed to set QP to ERROR state\n");
        return;
    }

    // 2. 处理所有刷新的 WQE
    struct ibv_wc wc;
    int num_comp;
    do {
        num_comp = ibv_poll_cq(cq, 1, &wc);
        if (num_comp < 0) {
            fprintf(stderr, "ERROR: ibv_poll_cq() failed\n");
            break;
        }
        if (num_comp > 0) {
            if (wc.status == IBV_WC_WR_FLUSH_ERR) {
                printf("WQE flushed (expected)\n");
            } else {
                fprintf(stderr, "WQE error: %s\n", 
                        ibv_wc_status_str(wc.status));
            }
            // 释放资源（假设 wr_id 指向用户分配的缓冲区）
            free((void*)wc.wr_id);
        }
    } while (num_comp > 0);

    // 3. 销毁 QP
    if (ibv_destroy_qp(qp)) {
        fprintf(stderr, "ERROR: Failed to destroy QP\n");
    }
}

```

#### 注意事项
**（1）顺序不可逆**
必须先将 `QP` 设为 `IBV_QPS_ERR`，再处理 CQ 事件，最后销毁 QP。直接销毁 QP 会导致未处理的 WQE 残留。

**（2）资源泄漏风险**
每个 `WQE` 可能关联内存缓冲区或其他资源。需在轮询 `CQ` 时通过 `wc.wr_id` 找到这些资源并释放。

**（3）多线程/异步操作** 
在刷新过程中「即将 QP 设为 `IBV_QPS_ERR`」，确保没有新 `WQE` 被提交到 `QP`。可通过锁机制等来实现。

==注：如果是多个`conn`共享一个`QP`，那么就不可以将 将 `QP` 设为 `IBV_QPS_ERR`状态==。

#### 小结
总结来说，正确关闭和销毁`QP`的关键在于：确保所有未完成的`WR`已经被`RNIC`处理或中止，并且软件已经确认这些操作的状态，之后再释放相关内存。
这需要严格的同步机制和状态管理，避免在硬件仍在使用内存时进行释放。

- **核心原则**：QP 销毁前必须确保 RNIC 已停止所有 DMA 操作，且所有未完成 WR 已通过 CQ 通知软件。
- **关键操作**：迁移 QP 到 Error State → 强制轮询 CQ → 按 `wr_id` 释放内存 → 销毁QP资源。

#### 其他问题排查思路

![](attachments/Pasted%20image%2020250930152203.png)

前面说到：如果 QP 是自动转换到 err 状态的，则第一个以错误完成的工作请求将指示错误原因。该队列中的所有后续工作请求以及其他队列中的所有工作请求和发布到这两个队列的新工作请求将以错误被刷新。
```bash
If the QP was transitioned to this state automatically, the first Work Request that completed with error will indicate the reason for the error. All subsequent Work Requests in that queue and all Work Request in the other queue and new Work Requests posted to both of the queues will be flushed with error.
```

但是，如果只看到了 `IBV_WC_WR_FLUSH_ERR` 错误，没有看到其他的错误。此时就看下对端设备的QP是否存在错误，以及本端是否存在异步事件。




# CQ

## CQE (Completion Queue Element)

CQE是RDMA系统中操作完成的最终记录，它提供了操作结果的完整信息。
```c
struct ib_cqe {  
    uint32_t wr_id;         // 对应的工作请求ID  
    uint32_t status;        // 完成状态：成功、失败、错误类型  
    uint32_t opcode;        // 完成的WQE操作码  
    uint32_t vendor_err;    // 厂商特定的错误码  
      
    // 额外信息  
    uint32_t byte_len;      // 传输的字节数  
    uint64_t imm_data;      // 立即数据（如果有）  
    uint32_t qpn;           // 队列对编号  
    uint32_t src_qp;        // 源队列对（用于接收操作）  
};
```

CQE的状态字段使用位编码提供详细的错误信息：
- **IB_WC_SUCCESS**：操作成功完成
- **IB_WC_WR_FLUSH_ERR**：操作被冲刷（QP重置等原因）
- **IB_WC_RETRY_EXC_ERR**：重试次数超过限制
- **IB_WC_RNR_RETRY_EXC_ERR**：接收方无响应重试超过限制


### CQE的拓扑关系
#### CQE与WQE之间的映射关系
```bash
WQE (wr_id = 0x1234) → 执行 → CQE (wr_id = 0x1234, status = SUCCESS)
```

这种ID映射确保了异步操作的确定性：**每个操作都有唯一的标识符，可以在完成时精确定位。**

#### 队列间的层次关系

```bash
应用程序  
    ↓ (提交WQE)  
Send Queue / Receive Queue  
    ↓ (HCA处理)  
执行引擎  
    ↓ (生成CQE)  
Completion Queue  
    ↓ (通知应用)  
应用程序
```


#### 内存布局拓扑
WQE和CQE在内存中的分布：

```bash
[ WQE_0 | WQE_1 | WQE_2 | ... | WQE_N ]  ← SQ/RQ  
[ CQE_0 | CQE_1 | CQE_2 | ... | CQE_M ]  ← CQ
```


![](attachments/2a743ce4a042bb735aaecfa07be7ee15.png)


队列的环形特性通过头尾指针管理：

- **生产者指针**：应用程序更新（SQ/RQ）；
    
- **消费者指针**：HCA更新（SQ/RQ），应用程序读取（CQ）

对于SQ/RQ而言，软件是生产者，硬件HCA是消费者；
对于CQ而言，硬件HCA是生产者，软件是消费者；



## QP和CQ的关系

![](attachments/Pasted%20image%2020250318144055.png)

一个QP包含一个Send Queue(SQ)，一个Receive Queue(RQ)以及对应的Send Completion Queue(SCQ)和Receive Completion Queue(RCQ)。
用户发送请求的时候，把请求封装为一个Work Queue Element(WQE)发送到SQ里面，然后RDMA网卡会把这个WQE发送出去，当这个WQE完成的时候，对应的SCQ里面会被放一个Completion Queue Element(CQE)，然后用户可以从SCQ里面Poll这个CQE并通过检查状态来确认对应的WQE是否成功完成。
需要指出的是，**不同的QP可以共用CQ来减少SRAM的存储消耗**。


## send/recv独立的cq or 共享的cq

### 背景

![](attachments/Pasted%20image%2020250506143413.png)

如上所示，最简单的使用send/recv方式发送请求和响应。
对于其中一端而言，比如拿发送端来举例，产生的`send CQE`和`recv CQE`可以在一个CQ中，也可以在两个CQ中。


### 独立cq的问题
对于发送端而言，如果使用2个CQ，即`send CQE`在`send CQ`中，`recv CQE`在另外一个`CQ「recv CQ」`中。
正常来说，应该是先获取到`send CQE`，然后获取到`recv CQE`。
实际上获取`send CQE`和`recv CQE`无法保序，即无法保证获取`send CQE`和`recv CQE`的先后顺序。

#### 潜在问题
如果`send CQ`和 `recv CQ`对应2个不同的`CQ`，那么从2个`CQ`中获取`CQE`无法保序。 无法保序就可能存在一些问题。

比如：业务的逻辑是，RPC的请求和响应是一一对应的，为了保证一一对应，一般请求和响应都有相同的 `id`。即收到响应的时候，需要基于响应中的 `id`查询在链表或者`hash`表中到RPC请求。
而RPC请求的`id`什么时候加入到 链表或者`hash`表中呢？
如果业务的逻辑实现不是很优雅的话，比如是在收到 `send cqe`的时候才进行加入。那么就可能存在问题。
因为 `send cqe`  和 `recv cqe`的获取无法保序，有可能先获取到 `recv cqe`，即先获取到响应，那么此时基于 `id`是无法查询到请求的。

#### 原因分析
潜在的可能性存在两种：
1》发送端收到的ACK和响应的`RoceV2 RDMA`的路径不一致（比如五元组不一致），导致到达的先后顺序无法保证。

2》发送端的`send CQ`和 `recv CQ`对应2个不同的`CQ`，就会出现从2个`CQ`中获取`CQE`无法保序。
比如发送端的线程使用 `Polling` 方式从2个`CQ`中取`CQE`，先从`Send CQ`中取`CQE`，再从`recv CQ`中取`CQE`。
有可能`Send CQ`取`CQE`的流程结束之后，此时产生了`Send CQE`，然后产生了 `recv CQE`，后续从`recv CQ`中取`CQE`。那么就是先获取到`recv CQE`，在下一轮`Polling` 中才可以获取到`Send CQE`。

注：其中`1》`不一定成立，需要验证。

#### 解决
（1）`send CQ`和 `recv CQ`共享相同的`CQ`。
（2）业务逻辑修改。
不要在获取到`send cqe`才进行插入处理，可以完成`send`的准备之后就提前插入；
然后在收到`recv cqe`进行查询。

####  小结
即对于一端而言，如果`send CQ`和 `recv CQ`对应2个不同的`CQ`。那么==从2个`CQ`中获取`CQE`无法保序==。 


# 生产者和消费者角度理解QP和CQ
(1) 对于WQ来说，Host是生产者，RNIC是消费者。
- Host（CPU）生产WR, 把WR放到WQ中去
- RDMA硬件消费WR

(2) 对于CQ来说，RNIC是生产者，Host是消费者。
- RDMA硬件生产WC, 把WC放到CQ中去
- Host（CPU）消费WC

![](attachments/Pasted%20image%2020250326151110.png)

# QP的flush
## 为什么要flush
## flush的含义
## flush SQ(send queue)
## flush RQ(receive queue: 非 SRQ)
## flush SRQ
## 其他



# QP 相关API
## 数据结构
### ibv_qp
```c
struct ibv_qp {
	struct ibv_context     *context;
	void		       *qp_context;
	struct ibv_pd	       *pd; /* qp所属的 pd */
	struct ibv_cq	       *send_cq; 
	struct ibv_cq	       *recv_cq;
	struct ibv_srq	       *srq;
	uint32_t		handle;
	uint32_t		qp_num;
	enum ibv_qp_state       state;
	enum ibv_qp_type	qp_type; /* service type: RC/UD等 */

	pthread_mutex_t		mutex;
	pthread_cond_t		cond;
	uint32_t		events_completed;
};
```
### ibv_qp_attr
```c
/* qp 属性 */
struct ibv_qp_attr {
	enum ibv_qp_state	qp_state;
	enum ibv_qp_state	cur_qp_state;
	enum ibv_mtu		path_mtu; /* Path MTU (valid only for RC/UC QPs) */
	enum ibv_mig_state	path_mig_state;
	uint32_t		qkey;  /* Q_Key of the QP (valid only for UD QPs) */
	uint32_t		rq_psn; /* PSN for receive queue (valid only for RC/UC QPs) */
	uint32_t		sq_psn; /* PSN for send queue */
	uint32_t		dest_qp_num; /* Destination QP number (valid only for RC/UC QPs) */
	unsigned int		qp_access_flags;  /* Mask of enabled remote access operations (valid only for RC/UC QPs) */
	struct ibv_qp_cap	cap; /* QP capabilities */
	struct ibv_ah_attr	ah_attr;  /* Primary path address vector (valid only for RC/UC QPs) */
	struct ibv_ah_attr	alt_ah_attr; /* Alternate path address vector (valid only for RC/UC QPs) */
	uint16_t		pkey_index; /* Primary P_Key index */
	uint16_t		alt_pkey_index; /* Alternate P_Key index */
	uint8_t			en_sqd_async_notify;
	uint8_t			sq_draining; /* Is the QP draining? (Valid only if qp_state is SQD) */
	uint8_t			max_rd_atomic; /* Number of outstanding RDMA reads & atomic operations on the destination QP (valid only for RC QPs) */
	uint8_t			max_dest_rd_atomic; /* Number of responder resources for handling incoming RDMA reads & atomic operations (valid only for RC QPs) */
	uint8_t			min_rnr_timer; /* Minimum RNR NAK timer (valid only for RC QPs) */
	uint8_t			port_num;  /* Primary port number */
	uint8_t			timeout;
	uint8_t			retry_cnt; /* Retry count (valid only for RC QPs) */
	uint8_t			rnr_retry; /* RNR retry (valid only for RC QPs) */
	uint8_t			alt_port_num; /* Alternate port number */
	uint8_t			alt_timeout; /* Local ack timeout for alternate path (valid only for RC QPs) */
	uint32_t		rate_limit; 
};
```


#### timeout
当发送方发出请求后，在超时时间内没有收到 ACK（即响应），则认为目标 QP 不可达，会触发重传。
timeout的单位不是毫秒，而是指数形式。
**超时时间的公式**：
```bash
Timeout = 4.096 µs × 2^timeout
```

|`timeout` 值|超时时间（µs）|超时时间（毫秒）|
|---|---|---|
|0|4.096 µs|0.004 ms|
|5|131.072 µs|0.131 ms|
|10|4.194 ms|0.004 s|
|15|134 ms|0.134 s|
|20|4.3 秒||
|23|34.4 秒||
|31|8 分钟多|极大|


这意味着：
- `timeout=14` 时，大概是 67 ms；
- `timeout=20` 时，大约是 4.3 秒；
- `timeout=31` 是最大值，大概超过 8 分钟。


**如何选择合适的 `timeout` 值**？
- 局域网环境（数据中心、无丢包）：`timeout=14` 是一个很常见、比较保守的值。
- 高可靠网络、调试环境：`timeout=20` 或更大，用于容忍更多延迟。
- 低延迟、实时场景：`timeout=10` 左右，但要确保网络稳定，否则可能误触超时。
- RoCE 网络（需要考虑丢包）：建议 较高 timeout + 配置重传次数（如 retry_cnt=7）。


**相关参数建议一起调整**：

|参数|含义|建议|
|---|---|---|
|`timeout`|等待 ACK 的超时时间|14~20|
|`retry_cnt`|主动发送失败后的重试次数|最大是 7（含无限重试）|
|`rnr_retry`|接收方未准备好时的重试次数|通常设为 7|
|`max_rd_atomic`|允许的并发 read 操作数|取决于硬件|
|`min_rnr_timer`|RNR 等待时间|可以设为较低值如 12（≈0.5ms）|




#### rnr_rety
#### retry_cnt



# CQ相关API
## WC状态
```c
/* wc(work complete) status */
enum ibv_wc_status {
	IBV_WC_SUCCESS,  // 0
	IBV_WC_LOC_LEN_ERR,
	IBV_WC_LOC_QP_OP_ERR,
	IBV_WC_LOC_EEC_OP_ERR,
	IBV_WC_LOC_PROT_ERR,
	IBV_WC_WR_FLUSH_ERR, // 5
	IBV_WC_MW_BIND_ERR,
	IBV_WC_BAD_RESP_ERR,
	IBV_WC_LOC_ACCESS_ERR,
	IBV_WC_REM_INV_REQ_ERR,
	IBV_WC_REM_ACCESS_ERR,
	IBV_WC_REM_OP_ERR,
	IBV_WC_RETRY_EXC_ERR, // 12
	IBV_WC_RNR_RETRY_EXC_ERR, // 13
	IBV_WC_LOC_RDD_VIOL_ERR,
	IBV_WC_REM_INV_RD_REQ_ERR,
	IBV_WC_REM_ABORT_ERR,
	IBV_WC_INV_EECN_ERR,
	IBV_WC_INV_EEC_STATE_ERR,
	IBV_WC_FATAL_ERR,
	IBV_WC_RESP_TIMEOUT_ERR,
	IBV_WC_GENERAL_ERR,
	IBV_WC_TM_ERR,
	IBV_WC_TM_RNDV_INCOMPLETE,
};
```
### ibv_wc_status_str
打印对应的 `wc status` 的字符串。

### wc的各个错误的原因
参考：[ibv_poll_cq 讲解](https://www.rdmamojo.com/2013/02/15/ibv_poll_cq/)

```c
The struct ibv_wc describes the Work Completion attributes.

struct ibv_wc {
    uint64_t        wr_id;
    enum ibv_wc_status  status;
    enum ibv_wc_opcode  opcode;
    uint32_t        vendor_err;
    uint32_t        byte_len;
    uint32_t        imm_data;
    uint32_t        qp_num;
    uint32_t        src_qp;
    int         wc_flags;
    uint16_t        pkey_index;
    uint16_t        slid;
    uint8_t         sl;
    uint8_t         dlid_path_bits;
};
```

![](attachments/Pasted%20image%2020251025060539.png)

![](attachments/Pasted%20image%2020251025061207.png)


#### IBV_WC_SUCCESS(0)
```bash
- **IBV_WC_SUCCESS (0)** - 
Operation completed successfully: this means that the corresponding Work Request (and all of the unsignaled Work Requests that were posted previous to it) ended and the memory buffers that this Work Request refers to are ready to be (re)used.
```

#### IBV_WC_WR_FLUSH_ERR(5)
```bash
- **IBV_WC_WR_FLUSH_ERR (5)** - 
Work Request Flushed Error: A Work Request was in process or outstanding when the QP transitioned into the Error State.
```
#### IBV_WC_RETRY_EXC_ERR(12)
```bash
IBV_WC_RETRY_EXC_ERR (12)  
Transport Retry Counter Exceeded: The local transport timeout retry counter was exceeded while trying to send this message. 
This means that the remote side didn't send any Ack or Nack. If this happens when sending the first message, usually this mean that the connection attributes are wrong or the remote side isn't in a state that it can respond to messages. 
If this happens after sending the first message, usually it means that the remote QP isn't available anymore. 
Relevant for RC QPs.
```
##### 场景
- 本端发送 WR；
- 对端**没有回应 ACK，也没有返回 RNR NAK**；
- HCA 等待 ACK 超时；
- 根据 QP 属性 `qp_attr.retry_cnt` 设置重试；
- 多次重试仍无 ACK；
- 报错并生成 **`IBV_WC_RETRY_EXC_ERR`**。

```bash
[发送端]
post_send()
  │
  ├─→ 发包 → 等待 ACK
  │
  ├─ 超时 (no ACK)
  │
  ├─ 重试 (根据 retry_cnt 次数)
  │
  ├─ 多次超时未收到 ACK
  │
  └─ 产生错误 CQE: IBV_WC_RETRY_EXC_ERR

```


##### 常见原因

- 对端 QP 崩溃、未响应；
- 网络丢包；
- 对端已关闭或 reset；
- QP 状态异常。

#### IBV_WC_RNR_RETRY_EXC_ERR(13)
```bash
- **IBV_WC_RNR_RETRY_EXC_ERR (13)** - 
RNR Retry Counter Exceeded: The RNR NAK retry count was exceeded. 
This usually means that the remote side didn't post any WR to its Receive Queue. Relevant for RC QPs.
```

##### 场景
- 本端发送 WR；
- 对端没有可用的 Recv WR；
- 对端 HCA 返回 **RNR NAK (Receiver Not Ready)**；
- 本端收到 RNR NAK 后等待指定时间（`min_rnr_timer`），再重发；
- 重发次数超过 `qp_attr.rnr_retry`；
- 报错并生成 **`IBV_WC_RNR_RETRY_EXC_ERR`**。

```bash
[发送端]                     [接收端]
post_send()
  │                              │
  ├─→ 发包                      │
  │                              ├─ 没有 Recv WR → 返回 RNR NAK
  │←─────── RNR NAK ─────────────┤
  │
  ├─ 等待 min_rnr_timer
  ├─ 重发 WR
  │
  ├─ 再次 RNR NAK
  ├─ ...
  ├─ 重试次数超过 rnr_retry
  │
  └─ 产生错误 CQE: IBV_WC_RNR_RETRY_EXC_ERR

```

##### 常见原因

- 对端 Recv queue 没有提前 post；
- QP 参数设置错误。


##### 对比

|情况|建议|
|---|---|
|经常出现 `IBV_WC_RNR_RETRY_EXC_ERR`|在接收端增加 Recv WR 数量；或者增大 `min_rnr_timer` 和 `rnr_retry`。|
|经常出现 `IBV_WC_RETRY_EXC_ERR`|检查对端是否宕机、QP 是否活着、网络是否丢包。|
|调试|使用 `ibv_query_qp` 查看 QP 重试参数。|

`IBV_WC_RETRY_EXC_ERR` 和 `IBV_WC_RNR_RETRY_EXC_ERR` — 是 RC 模式（Reliable Connection）下最典型的发送端错误 Completion（WC）。  
==两者都属于 发送端（local side）产生的错误 WC。  接收端只会影响是否产生 NAK/ACK，不会自己产生这些 WC==。

#### IBV_WC_LOC_PROT_ERR(4)
##### 介绍
**IBV_WC_LOC_PROT_ERR（值为4）** 是 `InfiniBand Verbs (RDMA)` 操作中的一个错误状态码，表示在本地检测到保护相关的错误（**Local Protection Error**）。它通常发生在 RDMA 操作（如发送、接收、原子操作等）的执行过程中，表明本地 RDMA 资源（如内存权限、队列对配置）与请求的操作不兼容。即：`IBV_WC_LOC_PROT_ERR` 的核心原因是 **权限不匹配或配置错误**。

当某个 RDMA 操作（例如 `ibv_post_send` 提交的请求）完成后，若其工作完成状态 (`wc.status`) 为 **`IBV_WC_LOC_PROT_ERR`**，意味着：
- **本地保护检查失败**：目标内存区域（Memory Region, MR）的访问权限不允许当前操作，或者使用的内存没有注册到`MR`。
- **配置不匹配**：队列对（Queue Pair, QP）或内存区域的权限配置存在问题。

##### 常见原因及解决方案
**（1）内存区域（MR）权限不足**
RDMA 内存区域（MR）在注册时需指定访问权限（如 `IBV_ACCESS_LOCAL_WRITE`、`IBV_ACCESS_REMOTE_READ` 等）。若操作的权限超出 MR 的权限范围，会触发此错误。

示例场景：
- 尝试通过 QP 发送一个 **远程写请求**，但对应的 MR 未启用 `IBV_ACCESS_REMOTE_WRITE`。
- 尝试进行 **原子操作**，但 MR 未启用 `IBV_ACCESS_REMOTE_ATOMIC`。

**（2）队列对（QP）与内存区域（MR）的保护域（PD）不匹配**
RDMA 规则要求：**QP 和 MR 必须属于同一个保护域（Protection Domain, PD）**。如果 QP 引用了其他 PD 中的 MR，会导致保护错误。

**（3）队列对（QP）配置错误**
QP 在创建时需设置其操作类型（如 `IBV_QPT_RC`、`IBV_QPT_UD`），若配置的权限与实际操作冲突，会触发此错误。
示例场景：
- QP 类型为 `IBV_QPT_UD`（不可靠数据报），但尝试执行需要可靠连接的原子操作。
- QP 的 `max_rd_atomic`（未完成 RDMA 读/原子操作数）设置过大。

**（4）内存未对齐或地址越界**
RDMA 硬件对内存地址的对齐和范围有严格要求。若操作的内存地址未对齐或超出 MR 的范围，会引发保护错误。
比如：检查操作的 `byte_len` 是否在 MR 的范围内。
```c
struct ibv_sge sge = {
    .addr = (uintptr_t)buffer,   // 内存地址（需在 MR 范围内）
    .length = data_size,         // 数据长度（不可超过 MR 的 size）
    .lkey = mr->lkey            // 使用正确的 MR 的 lkey
};
```

**（5）密钥（lkey/rkey）错误**
每个 MR 都有一个本地密钥 (`lkey`) 和远程密钥 (`rkey`)。若请求中使用了无效的 `lkey` 或 `rkey`，会导致保护错误。




# 参考
```bash
# RDMA cq event机制-ibv_req_notify_cq
https://zhuanlan.zhihu.com/p/688269158


```