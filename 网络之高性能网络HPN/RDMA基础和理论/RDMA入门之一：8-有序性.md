```table-of-contents
```

# 有序性的层级
## 操作内有序(Intra-Operation Ordering)
 **数据包顺序（Packet Order）/ 操作内有序(Intra-Operation Ordering)**： 
 单个操作（如Send或Write）可能被拆分成多个网络数据包传输，这些包是否按发送顺序到达。

## 操作间有序(Inter-Operation Ordering)

**操作顺序（Operation Order）** 
多个独立的RDMA操作（如连续发起两个Write， 或者RC服务类型下先Post write，然后Post Send）的完成顺序是否与提交顺序一致。

### 有序性和多个操作类型的关系
- 问题：
RDMA RC服务类型，先post一个write wr，然后post 一个send wr，可以保序吗，即write 的wc一定先于send wc吗？ 保序是否和操作类型有关系吗？

- 结论：
**在 RC 服务类型下，操作在同一个发送队列 (SQ) 上是严格保序的，与操作类型无关。**




## 事件流顺序

**事件流顺序（Event Flow Order）** 
跨QP或跨节点的全局事件顺序（如内存更新与通知的组合），需应用层协议保证。

# 典型乱序
## 原因
### 网络路径差异
包通过不同交换机路径传输（尤其无连接服务如UD/RD）。
- **解决**：无连接服务需应用层重排序。

注：无连接服务，类似于UDP，每次发包的UDP五元组不一样？？有连接服务，一个QP的五元组是固定的？

### 传输层重传
可靠服务（RC/RD）中丢失的包重传后，新包可能先到达。
```bash
发送：包1(PSN=1) → 包2(PSN=2) 
包1丢失 → 重发包1(PSN=1) 
到达顺序：包2 → 包1 ===> 接收端按PSN=1,2排序执行
```

- **结论**：重传不影响RC/RD的最终有序性。

### HCA处理优化
为提升吞吐，HCA可能并发处理操作（UC/UD）。

## 场景
### UC/UD传输的乱序
```bash
发送顺序: [操作A] → [操作B] 
到达顺序: [操作B] → [操作A] // 网络抖动导致乱序
```

### RD的多消息传输
```bash
发送: [消息1] → [消息2] 
到达顺序: [消息2] → [消息1] // 消息间无序
```

# 有序性的级别
## 严格有序（Strict Ordering）
**(1) 机制**：
- 硬件保证**同一QP内**的操作按提交顺序完成（如RC服务）。
- 发送队列（SQ）顺序 = 网络包顺序（PSN严格递增） = 接收端执行顺序。

**(2) 服务类型**：仅 **RC** 支持。

**(3) 代价**：等待前序操作ACK增加延迟

**(4) 范例**
```bash
// RC QP示例：提交顺序即执行顺序
ibv_post_send(qp, &write_wr1); // 操作1：写入数据A
ibv_post_send(qp, &write_wr2); // 操作2：写入数据B → 保证B在A后执行
```

### RC服务类型的严格保序
对于 RC 服务类型，RDMA 规范提供了以下关键保证：

#### A. 发送队列 (SQ) 上的提交顺序

所有提交（`post`）到同一个 **队列对 (QP) 的 发送队列 (SQ)** 上的工作请求（WRs），
无论它们是 `WRITE`、`READ`、`SEND` 还是 `ATOMIC` 操作，都必须按照它们被提交的顺序执行。

```bash
Post(WR1​)→Post(WR2​）=====>
	WR1​ 在网络和对端执行上优先于 WR2​
```

#### B. 工作完成 (WC) 的顺序

如果两个 WR 都标记为 `signaled` (即要求生成 WC)，那么它们在完成队列 (CQ) 中出现的顺序，必须严格反映它们在发送队列 (SQ) 中的提交顺序。
```bash
比如：
    1. `Post(WR_1: WRITE)`
    2. `Post(WR_2: SEND)`

`WC(WR_1)` 一定先于 `WC(WR_2)` 出现在 CQ 中。
```





## 消息内有序（Intra-Message Ordering）

**(1) 机制**：
- 单个大消息被拆分为多个包时，包按发送顺序到达（如RD服务）。
> 注：==消息间仍可能乱序==。

**(2) 服务类型**：RD 支持。

**(3) 范例**
```bash
Send消息X [包X1][包X2] → 保证X1在X2前到达 
Send消息Y [包Y1][包Y2] → 但Y可能比X先到达
```

## 松散有序（Relaxed Ordering）
 **（1）机制**：
- 通过 `IBV_SEND_INLINE`或`Fence`选择性控制有序性。
- 允许后续操作**绕过前序未完成的操作**提前执行。

## 完全无序（No Ordering）
**(1)机制**：
- 无任何顺序保证，数据包按网络路径自由到达。

**(2) 服务类型**：UD和 UC的默认行为。

**(3) 解决方法**
UD应用层，将应用层序号也作为数据的一部分传递；对端收到数据之后，解析出序号，进行排列组合。
```c
// UD应用需添加序列号
struct Message {
    uint32_t seq; // 应用层序号
    char data[];
};
// 接收端重组乱序消息
```

## 保序性与操作类型的关系

**保序性** 是 RDMA **服务类型 (Service Type)** 的属性，而非 **操作类型 (Operation Type)** 的属性。
比如：在 **RC 服务类型**下，**操作类型（`WRITE`, `SEND`, `READ`）** 不会破坏保序性。保序性的保证正是 RC 的主要特性之一。



|**服务类型 (Service Type)**|**保序性 (Ordering Guarantee)**|**核心特点**|
|---|---|---|
|**RC (Reliable Connected)**|**严格保序**|提供可靠性（有 ACK 和重传机制），并保证同一 SQ 上的操作顺序。|
|**UC (Unreliable Connected)**|**不保证保序**|提供不可靠但有序的连接（无 ACK/重传），理论上可以乱序。|
|**UD (Unreliable Datagram)**|**不保证保序**|无连接、不可靠，操作可能乱序，也可能丢失。|

### 保序性在 WRITE + SEND 组合中的应用

正是因为 RC 提供了这种严格的保序性，`WRITE` + `SEND` 组合才能被安全地用于通信：
**发送方 `Post(WRITE)` → `Post(SEND)`**：
保证数据先被写入，然后信号才被发送。

**接收方：** 
当接收方通过 `SEND` 消息被通知时，它确信 **所有在 `SEND` 之前 `post` 的 `WRITE` 操作的数据，已经在接收方的内存中可见且稳定了。**

如果这个顺序性不能得到保证，那么接收方在收到信号后，去读取 `WRITE` 写入的数据时，可能会读到旧数据或不完整的数据，从而导致程序错误。因此，RC 的保序性对于这种异步控制流至关重要。


# 有序性的实现


# 服务类型和有序性

|**服务类型**|**数据包顺序**|**操作顺序**|**典型场景**|
|---|---|---|---|
|**RC (可靠连接)**|✅ **严格保证**|✅ **严格保证**|分布式数据库、存储系统（如NVMe-oF）|
|**UC (不可靠连接)**|❌ **不保证**|❌ **不保证**|实时数据流（容忍乱序）|
|**UD (不可靠数据报)**|❌ **不保证**|❌ **不保证**|行情广播、日志收集|
|**RD (可靠数据报)**|✅ **消息内包有序**|❌ **消息间操作无序**|分布式共享内存的通信层|



# 有序性和send_flag

## 有序性和`IBV_SEND_FENCE`


### 背景

![](attachments/Pasted%20image%2020250715170520.png)


### fence的作用
Fenced 标记主要用于在需要因果关系（如 Read-Modify-Write）的场景中，强制同步：需要**保证内存操作的严格顺序**，特别是当控制信息或状态信息依赖于先前的批量数据操作时。

![](attachments/Pasted%20image%2020251124122442.png)

当一个工作请求（Work Request）中设置了 Fence 标记（Fence Indicator） 时，Fenced 标记强制发送端 HCA 遵守提交顺序，延迟后续操作的发出，直到它前面所有依赖的操作（如 RDMA Read）的响应数据安全返回。
发送队列（Send Queue）不应开始处理该WR，直到该发送队列上**所有先前的 RDMA Read 和 Atomic 操作都已完成**。



### 存在操作依赖的前者操作完成之后，再执行后面的操作 Vs 两个操作，后者操作带有fence标记

**（1）方法一：先post一个操作(比如 read操作)，然后等第一个操作完成之后，再post第二个操作(比如：write操作)**

**（2）方法二：一次性post两个操作（比如，第一个操作是read操作，第二个操作是write操作，后者依赖前者），第二个操作的WR带有Fence标记**

方法一和方法二，都可以保证**内存操作**的有序性。

但是方法一，相对于方法二，应该是延迟更高。

|**特性**|**方法一：显式等待 WC (Blocking)**|**方法二：使用 Fence 标记 (Non-Blocking)**|
|---|---|---|
|**实现机制**|**软件同步/阻塞。** 依赖 CPU 轮询/等待 `poll_wc()`。|**硬件同步/非阻塞。** 依赖 HCA 内部硬件逻辑进行同步。|
|**提交方式**|**两次提交。** 第一次 `post_send()` $\rightarrow$ `poll_wc()` $\rightarrow$ 第二次 `post_send()`。|**一次提交。** 一次性提交两个或多个 WRs。|
|**延迟**|**高延迟。** 引入了**两次上下文切换**（如果等待）、**两次网络往返**的串行等待时间，以及 CPU 轮询的开销。|**低延迟。** 所有的 WRs 都在一个批次中提交，网络传输和 HCA 内部处理是**流水线化**的。|
|**CPU 利用率**|**高。** CPU 必须参与 `poll_wc()` 轮询或阻塞等待。|**低。** 提交后 CPU 立即返回，同步由 HCA 硬件管理。|
|**吞吐量**|**低。** 引入了同步点和两次提交的开销，限制了 I/O 速率。|**高。** 允许 HCA 内部优化 I/O 流水线。|
|**编程复杂度**|**较高。** 需要管理两个独立的 `post_send` 调用和 `poll_wc` 逻辑。|**较低。** 只需要在第二个 WR 上设置一个标志位。|
|**容错性**|如果第一个操作失败（WC 带有错误），第二个操作根本不会被提交，逻辑清晰。|如果第一个操作失败，Fence 保证了第二个操作也不会被目标端观察到（如规范 C10-101.2.1 所述），但 HCA 仍需要处理错误。|


### 为什么RC服务可靠有序，还需要`IBV_SEND_FENCE`

（1）RC 的可靠有序性不代表**内存操作**的严格有序性。
RC 服务保证的是**数据包**的传输不丢失、不重复、按顺序到达目标 HCA。
Fenced 标记保证的是 HCA **内存操作**的执行顺序和可见性。

（2）**主机WC 顺序**不代表内存**操作顺序**
发送端收到 **WC (Work Completion) 的顺序**不代表目标端**内存操作**执行的顺序。


#### RC 的“可靠有序”保证了什么？
RC 的有序性保证的是**传输层包级按序到达**和**message/WQE 提交顺序**，不是**跨 direction 的内存可见顺序**。

RC 服务的可靠性（Reliability）体现在：
- **数据包有序到达：** 目标 HCA 接收到的数据包顺序，与源 HCA 发送的顺序一致（基于 PSN，Packet Sequence Number）。
- **端到端确认 (ACK)：** 保证每个数据包最终会被目标端接收。



RC 的“有序”保证的是 **HCA 之间的传输顺序**，但它**不保证**数据写入目标内存和操作完成的顺序。

**READ 是从远端内存读取，而 WRITE 是对远端内存写入；二者是不同的方向，不在同一 pipeline 上，所以不能依赖 RC 自身保持 ordering。**


##### RC的有序性和 Fence机制对比

|维度|RC可靠有序|Fence机制|
|---|---|---|
|**保证范围**|**传输层**：packet按PSN顺序发送/到达/确认（ACK/NAK），Message（WR）按提交顺序完成（CQE FIFO）。|**应用/内存层**：操作的**因果执行顺序**和**远程可见性**（e.g., Read看到之前Write的结果）。|
|**冲突点**|无：RC不干预内存一致性（Relaxed Ordering），允许硬件优化乱序。|Fence是**用户显式控制**，不改变RC的传输语义。|
|**规范依据**|IB Spec §11.4.2：RC提供“可靠有序传输”，但§12.7.4明确“内存操作需Fence确保一致性”。|libibverbs：Fence是可选标志，不影响PSN/ACK。|

两者为什么不冲突：
RC像“邮政系统”（包有序投递），Fence像“阅读顺序标签”（确保用户读完前一封再读下一封）。传输有序不等于内存可见有序。



#### 顺序
##### 发送端post wr的顺序
##### 接收端数据包的顺序
RC保证的数据包的有序性。
即：第一个操作的数据包先到达，第二个操作的数据包后到达。

##### 接收端内存操作的顺序


##### 发送端WC产生的顺序
发送端WC总是按提交顺序生成（FIFO），Read WC 一定 先于 Write WC 返回。这是由RC的Completion Queue (CQ) 保序 保证的：
发送端WC只表示本地提交完成 + 远程确认（ACK），不保证远程内存中的可见性顺序（e.g., Read WC返回时，远程可能已执行Write）。


#### 小结
（1）RC 的有序性保证的是“包级”和“message/WQE 提交顺序”，不是“跨 direction 的内存可见顺序”。
**（2）两个 WRITE 不需要 fence 就可以保证顺序**：因为两个 WRITE 都是单向操作，走同一个 pipeline。对同一个 QP 方向（send → remote），WRITE 不需要 fence 就已经严格按 WQE 顺序执行。
**（3）Read之后进行Write，Write需要添加fence**：因为READ 是从远端内存读取，而 WRITE 是对远端内存写入；二者是不同的方向，不在同一 pipeline 上，所以不能依赖 RC 自身保持 ordering。没有fence的RMW(read-modify-write), 可能发生 乱序(reordering), 这不是包乱序,而是 RNIC 在处理读写 pipeline 时可能提前执行 READ，导致读到旧值或未来值。



### Local Invalidate Fencing(本地作废围栏)

![](attachments/Pasted%20image%2020251124123106.png)

### Fence适用的服务类型和操作场景

适用于：**Read-Modify-Write**  这种存在**操作依赖**，即后续的操作依赖于前面的操作的完成。

```bash
IBV_SEND_FENCE:
Set the fence indicator for this WR. This means that the processing of this WR will be blocked until all prior posted RDMA Read and Atomic WRs will be completed. Valid only for QPs with Transport Service Type IBV_QPT_RC.
```

==只有RC服务类型才可以设置`IBV_SEND_FENCE`标记==。

![](attachments/Pasted%20image%2020251124141515.png)



#### 什么时候使用`IBV_SEND_FENCE`

![](attachments/Pasted%20image%2020250715171113.png)

即：
==如果2个操作访问的内存地址范围存在重叠的情况下，在`RDMA  READ` 之后的操作是 `send/write/atomic`等操作，那么可能需要再2个操作之间添加`fence`。
如果2个操作访问的内存地址范围不存在重叠，则不用考虑加`fence`==。


### Fenced WQE 与内存屏障（Memory Barrier）的区别和联系

### 小结

![](attachments/Pasted%20image%2020251124123826.png)




## 有序性和`IBV_SEND_INLINE`
### IBV_SEND_INLINE
`IBV_SEND_INLIN`E 减少网卡的`DMA`。(`CPU`直接将数据写入网卡缓冲,而不是等网卡来`DMA`）

```bash
IBV_SEND_INLINE:
    The memory buffers specified in sg_list will be placed inline in the Send Request. 
    This mean that the low-level driver (i.e. CPU) will read the data and not the RDMA device. 
    This means that the L_Key won't be checked, 
    actually those memory buffers don't even have to be registered and they can be reused immediately after ibv_post_send() will be ended. 
    Valid only for the Send and RDMA Write opcodes
```
`sg_list`中指定的内存缓冲区将内联放置在SR（ WQE）中。这意味着底层驱动程序（即CPU）将读取数据，而不是RDMA设备通过DMA来读取。这意味着将不会检查`L_Key`，实际上那些内存缓冲区甚至不必注册(毕竟CPU本来是老大/主子），并且可以在`ibv_post_send（）` 将要结束后，立即重用它们。仅对`Send`和`RDMA write`操作码有效。


#### INLINE的好处

![](attachments/Pasted%20image%2020250715193644.png)

对于`send/write`发送小的数据，通过`inline`，可以提升性能。应该是通过CPU拷贝小的数据内容，比 RNIC通过DMA来读取更快。

#### INLINE 和非INLINE的对比
**（1）两次DMA Vs 一次DMA**
原先是SR（内含`sg_list`）和`sg_list`中指向的内存缓冲区 放置在两个位置，网卡先读取`SR`获得`sg_list`，再根据`sg_list`信息读取`sg_list`指向的内存缓存。需要两次DMA操作。
内联就是将内存缓存区放置在`SR`内，网卡读取`SR`就一并获得内存缓存，只需一次`DMA`。


#### 使用INLINE的陷阱

根据文档, `inline`发送不需要等待`wc`就可以重用发送`buffer`. 不需要等待`wc`就可以继续发消息，但是如果不处理`wc`，那么就不会清理`sr`，连续不断的继续发送`INLINE`消息（而不去处理`wc`），`sr`得不到清理最终会撑爆`sq`，导致最后发不出消息。
所以使用`INLINE`的时候记得在`sq`撑爆之前去处理`wc`。


# 有序性和内存可见性(内存一致性)

# 有序性和立即数imm_data


# 有序性和原子操作的关系



# 注意事项
## RC中独立的send/recv CQ
在RC `send/recv`的操作中，如果`send`和`recv`对应各自的CQ(即：send CQ 和 recv CQ不同)。
那么一个线程中，`post send` 和  `post recv` 的 先后顺序 和 对应产生的 `send cqe`, `recv cqe`的先后顺序 没有直接关系。
即：先`post send` ，再`post recv`，不一定会先产生`send cqe`，再产生`recv cqe`。

==注：如果两者对应的使用一个`CQ`，那么产生`CQE`的顺序和`Post WR`的顺序是一致的==。


# 参考
```bash

```