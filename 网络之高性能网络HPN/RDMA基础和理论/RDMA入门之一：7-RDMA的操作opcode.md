```table-of-contents
```
# 概述
RDMA的全称是Remote Direct Memory Access，从字⾯意思可以看出，RDMA要实现直接访问远程内存，RDMA的很多操作就是关于如何在本地节点和远程节点之间实现内存访问。

# 操作分类

## 内存Verbs 和 消息Verbs

RDMA API (Verbs)主要有两种Verbs：

（1）内存Verbs(Memory Verbs)，也叫`One-Sided RDMA`。包括`RDMA Reads`, `RDMA Writes`, `RDMA Atomic`。
这种模式下的RDMA访问完全不需要远端机的CPU参与。

（2）消息Verbs(Messaging Verbs)，也叫`Two-Sided RDMA`。包括`RDMA Send`, `RDMA Receive`。这种模式下的RDMA访问需要远端机CPU的参与。

## 单边操作和双边操作

“单边”和“双边”本质都是在本地和远程节点之间共享内存。
对于双边来说，需要双⽅节点的CPU共同参与；
⽽单边则仅仅需要⼀⽅CPU参与即可，对于另⼀⽅的CPU是完全透明的，不会触发中断。

在实际中， SEND /RECEIVE多用于连接控制类报文，而数据报文多是通过READ/WRITE来完成的 。

- **双边操作**（Send/Send with Imm）：依赖接收方缓冲区 → **通用性强**（支持所有类型）。
- **单边操作**（Read/Write/Atomic）：需主动访问远程内存 → **依赖连接状态**（RC/UC/XRC）。

### 单边操作（One-Sided）

- **定义**：**仅需单端CPU参与**，远程端无需感知操作。数据直接从本地内存写入/读取远程内存。
- **典型操作**：
    - RDMA Write（客户端直接写远程内存）
    - RDMA Read（客户端直接读远程内存）
- **特点**：
    - **零拷贝**：绕过远程CPU、操作系统和网络栈。
    - **低延迟**：无需远程进程参与。
    - **需预注册内存**：远程内存需提前注册并授权访问。
- **适用场景**：
    - 高频交易、分布式存储（如NVMe-oF）、大规模数据传输。

### 双边操作（Two-Sided）

- **定义**：**需要两端CPU协同**，接收方需显式发布接收请求（Post Recv）。
- **典型操作**：
    - Send/Recv（发送方发送数据，接收方需提前发布接收缓冲区）
- **特点**：
    - **类似Socket通信**：需要发送方和接收方协同工作。
    - **灵活性高**：适合动态交互场景。
    - **延迟较高**：涉及接收方CPU参与（缓冲区准备、确认）。
- **适用场景**：
    - 传统消息传递（如MPI）、需要握手的协议。



## 单向操作和双向操作
### 单向操作（Unidirectional）

- **定义**：数据仅在一个方向上传输（客户端→服务端，无需反向数据流）。
- **典型场景**：
    - RDMA Write（客户端直接写入服务端内存）
    - RDMA Send（发送方→接收方，但接收方需要提前发布接收缓冲区）
- **特点**：
    - **延迟更低**：无需等待接收方响应（但可能需要同步机制）。
    - **吞吐量高**：适合单向数据流（如视频流、广播）

### 双向操作（Bidirectional）

- **定义**：数据在双向传输（客户端↔服务端，需要来回交互）。
- **典型场景**：
    - RDMA Read（客户端从服务端读取数据，需先发请求，再收响应）
    - 传统的TCP/IP通信（请求-响应模型）
- **特点**：
    - **延迟较高**：需要等待对端响应（RTT主导延迟）。
    - **适合交互式场景**：如数据库查询、RPC调用。

### 单向/双向和单边/双边的区别

- **单边/双边**：取决于**是否需要远程CPU参与**（单边操作完全绕过远程CPU）。
- **单向/双向**：取决于**数据是否双向流动**（如Read需要双向交互）。

# 操作的整体流程

![](attachments/Pasted%20image%2020250323232553.png)

2. 提交任务：App 通过将 WQE 放入 SQ / RQ 来提交一个任务。
3. 完成任务：RNIC 根据 WQE 执行任务，然后生成一个 CQE，包含该任务的完成信息，并放入 CQ。
4. 检查结果：App 检查 CQ，确认任务完成情况。如果失败，则可以查看 CQE 信息来了解失败的原因。


# 单边(one-side)操作（Memory verbs）

可以看出“单边”传输才是被⽤来传输⼤量数据的主要⽅法。但是“单边”传输也⾯临这下列挑战：

Write API：单端操作，Sender 主动执行，只需要本端明确源和目的内存地址，不需要告知 Receiver，Receiver 的 CPU 也不参与，也不会被通知数据的到达。适用于大批量数据传输。
 Read API：同上。

## 挑战
（1）由于RDMA在数据传输过程中不需要内核参与，所以内核也⽆法帮助RDMA缓存数据，因此RDMA要求在写⼊数据的时候，数据的⼤⼩不能超过接收⽅准备好的共享内存⼤⼩，否则出错。所以发送⽅和接收⽅在写数据前必须约定好每次写数据的⼤⼩。

（2）由于RDMA在数据传输过程中不需要内核参与，因此有可能内核会把本地节点要通过RDMA共享给远程节点的内存给交换出去，所以RDMA必须要跟内核申请把共享的内存空间常驻内存，这样保证远程节点通过RDMA安全访问本地节点的共享内存。

（3）虽然RDMA需要把本地节点跟远程节点共享的内存空间注册到内核，以防内核把共享内存空间交换出去，但是内核并不保证该共享内存的访问安全。即本地节点的程序在更新共享内存数据时，有可能远程节点正在访问该共享内存，导致远程节点读到不⼀致的数据；反之亦然，远程节点在写⼊共享内存时，有可能本地节点的程序也正在读写该共享内存，导致数据冲突或不⼀致。使⽤RDMA编程的开发者必须⾃⾏保证共享内存的数据⼀致性，这也是RDMA编程最复杂的关键点。

总之，RDMA在数据传输过程中绕开了内核，极⼤提升性能的同时，也带来很多复杂度，特别是关于内存管理的问题，都需要开发者⾃⾏解决。

## RDMA Read
### 单向数据 Read API 流程
![](attachments/Pasted%20image%2020250323232708.png)

（1）Local App 将 WQE 下发到 SQ，表示一个请求读取任务。
（2）Local RNIC 从 SQ 中获取到 WQE。
（3）Local RNIC 解析 WQE 的内容，并封装成数据包发送到 Remote RNIC。
（4）Remote RNIC 接收到数据包，并解析内含的虚拟地址，然后将虚拟地址转换为物理地址，并通过 DMA 的方式从 Main Memory 获得相应的数据。
（5）Remote RNIC 将获取到的数据封装成数据包发送到 Local RNIC。
（6）Local RNIC 接收到数据包后对其进行解封装，获取到内含的数据之后，根据 WQE 的描述，通过 DMA 将数据放置到指定的 Main Memory 中。
（7）Local RNIC 发送 CEQ 到 CQ。
（8）Local App 从 CQ 得到任务完成的反馈。



## RDMA Write
### 单向数据 Write API 流程

![](attachments/Pasted%20image%2020250323232340.png)


（1）Local App 将 WQE 下发到 SQ，表示一个请求发送任务。
（2）Local RNIC 从 SQ 中获取到 WQE。
（3）Local RNIC 解析出 WQE 中包含的虚拟地址，并通过 RNIC 中的 MPT、MTT 表转换为相应的 Main Memory 物理地址，然后从 Main Memory 中取得数据，封装为数据包。
（4）Local RNIC 将数据包发送到 Remote RNIC。
（5）Remote RNIC 接收到数据包，并解析内含的 Payload 数据、虚拟地址、rkey 等信息，并根据 RNIC 的 MPT、MTT 表将虚拟地址转换为 Main Memory 得物理地址，然后通过 DMA 的方式将 Payload 写入到 Main Memory 想要的位置。
（6）Remote RNIC 返回 ACK 到 Local RNIC。
（7）Local RNIC 接收到 ACK 后，发送 CEQ 到 CQ。
（8）Local App 从 CQ 得到任务完成的反馈。

![](attachments/Pasted%20image%2020250323232653.png)

### write操作的保序
默认乱序，需FENCE标志保序。

#### 带 fence的 write

### write的原子操作
#### Compare-and-Swap
#### Fetch-and-Add

## RDMA Read 和 RDMA Write 对比

在 RDMA 网络中，**Write 操作是“推（Push）”模式，而 Read 操作是“拉（Pull）”模式**。

WRITE在RDMA和PCIe级别上需要更少的状态维护，因为发起者不需要等待响应。然而，对于READ，请求必须在发起者的内存中维护，直到响应到达。
同样，在PCIe级别，从target机器上看，RDMA READ是使用non-posted的事务执行的，而RDMA WRITE使用开销更低的posted事务。

因此，RDMA Write适合数据路径，RDMA Read 适合元数据或者控制路径。

![](attachments/Pasted%20image%2020260205114441.png)

### 为什么 Unsignaled RDMA Write 和 Signaled RDMA Write 的时延差距显著？

Signaled / Unsignaled 的区别，不在网络、不在 PCIe，而在：  “这条 WR 是否在本端产生 CQE（Completion Queue Entry）”。

```bash


(1) RDMA read:

READ request (signaled)
 → READ response
 → DMA 写数据
 → 本端 RNIC DMA 写 CQE
 → App poll CQ

-----------

(2) Signaled Write:

Signaled verb（带信号的 WR）: 提交的 WR 设置了 `IBV_SEND_SIGNALED`，完成后会在本端 CQ 里生成一个 CQE，应用线需要轮询CQ；

App
 └─ post WR
      ↓
   RNIC 执行
      ↓
   Target DMA write
      ↓
   本端 RNIC DMA 写 CQE
      ↓
   App poll CQ


- 过程：
你的程序发送数据 -> 硬件处理 -> 硬件写回一个状态报告（CQE） -> 你的程序通过轮询（Polling）看到这个报告，确认任务真的结束了。

- 代价：
产生和处理 CQE 会消耗 PCIe 带宽和 CPU 周期。

- 原文理解：
文中提到在 Signaled 模式下，WRITE 和 READ 的延迟相似。这是因为无论哪种操作，CPU 都必须等待硬件走完整个流程并生成回执，这个等待过程抹平了两者微小的执行差异。

------

(3) Unsignaled Write:

Unsignaled verb（不带信号的 WR）:提交的 WR 没有 `IBV_SEND_SIGNALED`，不会产生 CQE，WR 执行完成后对应用是“静默的”；

App
 └─ post WR
      ↓
RNIC
 └─ 发 RDMA WRITE 包
      ↓
Target RNIC
 └─ DMA 写 host memory
      ↓
(结束，无 CQE)

特点：
- 单向网络
- PCIe posted write
- 没有 CQE DMA, 没有 CQ poll


- 过程：
你的程序把任务丢给硬件后，不再等待回执，直接假设它会成功（或者每隔 100 个任务才要一个信号来确认进度）。
    
- 优点：
减少了硬件写回 CQE 的开销，降低了 CPU 的中断/轮询压力。

- 原文理解：
当不要求回执时（Unsignaled），WRITE 的“真身”——即数据包单向发送的特性就体现出来了。由于不需要像 READ 那样等待目标端把数据传回来，WRITE 的延迟几乎只有 READ 的一半。

---------

```


|**特性**|**Signaled (带信号)**|**Unsignaled (不带信号)**|
|---|---|---|
|**完成通知**|每个操作都会生成一个 CQE（回执）|只有显式要求的操作才生成 CQE|
|**CPU 消耗**|较高（需要频繁检查回执）|极低（“发后即忘”）|
|**延迟表现**|相对稳定，受限于回执处理|**极低**（反映了真实的单向网络传输耗时）|
|**吞吐量**|受限于 CQE 处理速度|**极高**（硬件处理路径极简化）|
|**安全性**|容易追踪每个包是否成功发送|需开发者通过其他手段（如定期插入一个 Signaled 包）确保队列不溢出|


#### 小结

**同样的数据大小，`Unsignaled WRITE` 的时延 只有 `READ` 的 `1/2` 左右； `Signaled WRITE` 和 `READ` 的时延接近**。


```bash
`Signaled inline WRITE` 可以比 `Signaled WRITE` 更快，因为 避免了一次 `DMA` 操作，避免的是`payload DMA`, 但是`CQE DMA` 仍然存在。

因此，小 payload：
- inline → 少一次 DMA
- signaled → 仍有 CQE
```


### RDMA WRITE 为什么是 Pcie posted？

这里讨论的是 `RNIC ↔ Host Memory` 的 PCIe 行为，不是网络包本身。==从PCIe角度，“读比写更慢”==。

|类型|行为|
|---|---|
|PCIe Write|posted（不等返回）|
|PCIe Read|non-posted（必须等 completion）|

结果就是： Pcie Write 可以 流水线，Pcie Read 需要 往返事务（RTT）；


**（1）RDMA WRITE 的本质语义：主动方 push 数据 → 被动方内存**。

```bash
在 target 机器上发生了什么？

[ Network ]
    |
    v
RNIC
    |
    |  PCIe Memory Write (posted)
    v
Host Memory


关键点：
- RNIC 收到网络包
- RNIC：
    - 已经有 payload
    - 目标地址已知

- 所以：
    - 直接发 PCIe Memory Write
    - 不需要再读内存
    - 不需要 completion

```

**（2）RDMA READ 的本质语义：主动方 pull 数据 ← 被动方内存**
```bash
在 target 机器上发生了什么？

[ Network ]
    |
    v
RNIC (收到 READ request)
    |
    |  PCIe Memory Read (non-posted)
    v
Host Memory
    |
    |  Completion + Data
    v
RNIC
    |
    |  Network Send
    v
Requester

关键点：
2. RNIC 手里没有数据
3. 数据在 host memory
4. RNIC 必须：
    - 发 PCIe Memory Read（non-posted）
    - 等 completion
    - 拿到 data
5. 再把数据封装成 RDMA READ response 发回去

```


**（3）对比**
把两个路径并排看就很直观了
```bash
(1) RDMA WRITE（target机器视角）：

RX packet
   ↓
RNIC
   ↓  posted write
Host memory


- 1 次 PCIe 事务
- 无 completion
- 无额外网络包


（2）RDMA READ（target）
RX read request
   ↓
RNIC
   ↓ non-posted read
Host memory
   ↑ completion + data
RNIC
   ↓ network response
Requester

- 1 次 PCIe read request
- 1 次 PCIe completion
- 1 次网络 response
- RNIC 内部要维护状态

无论从 PCIe 还是网络角度，都更重。

这和你之前看到的“WRITE 延迟≈READ 一半”完全一致“。
现在可以从硬件层闭环了：
- WRITE：
    - 单向
    - pcie posted
    - 无 completion
        
- READ：
    - 双向
    - pcie non-posted
    - completion + response

```

### 性能上
#### RDMA Read 的Read After Read (RAR) Blocking 问题

在 RDMA 中，网卡处理 Read 请求时有一个限制：**未完成的 Read 操作（Outstanding Reads）数量是有限的**。 

**未完成读请求（Outstanding Read Requests）限制**：每个队列对（QP）通常有一个硬件限制，即允许同时进行的未完成 RDMA Read 操作的最大数量（例如 InfiniBand 中默认为 16）。当应用程序连续发出多个 RDMA Read （post_send 发送多个 read）请求且未等待完成时，如果未完成的读请求数量已达到该限制，再发出的新读请求将会被阻塞（即 `post_send` 可能返回资源不足错误，或请求在软件队列中等待），直到某些读操作完成并释放资源。

![](attachments/Pasted%20image%2020260317151358.png)

```bash
max_qp_rd_atom：单个 QP 的 outstanding RDMA Read / Atomic 数量；
max_sge_rd：一个 RDMA Read 请求中，最多支持多少个 SGE（scatter/gather entries）
```

当你发很多 Read，如下所示，如果 `inflight == max_outstanding_read`,  后续 Read 无法继续发出（post_send 发送read 失败），也就是 **read block&** 了。

```bash
比如：最多同时 in-flight 的 Read = 16

那么：
Read1  → 发出（未完成）
Read2  → 发出
...
Read16 → 发出
Read17 → ❌ 不能发（必须等前面完成）
```

#### 为什么RDMA Read有max_outstanding限制，RDMA Write没有？

RDMA Read 是一个请求-响应型操作。RDMA Write 是一个推送型操作（即发即弃）。
**RDMA Read 受限是因为它需要“跟踪未完成事务（stateful）”；  而 RDMA Write 是“无状态流式发送（stateless pipeline）”，不需要这种窗口限制**。

即：RDMA 硬件需要为每个未完成的读请求（outstanding RDMA read）**维护状态**（如跟踪响应报文、保证数据正确写入本地内存），而**硬件资源（如片上内存）有限**。因此协议规定了一个最大未完成读请求数。

##### RDMA Read 需要状态管理 （stateful）

当你发起一个 RDMA Read 操作时，本地硬件实际上做了两件事：
1. **发送请求**：本地 NIC 向远端 NIC 发送一个数据读取请求包。
2. **等待响应**：本地 NIC 必须等待远端 NIC 将数据作为响应包返回。

在这个过程中，**本地硬件需要为每一个未完成的 Read 请求维护一个状态**：
```bash
- 请求发给谁（remote QP / addr / rkey）
- 读多少数据
- 数据回来要放哪里
- 如何匹配 response
```
- **上下文存储**：硬件需要记住这个 Read 请求要将数据写入本地内存的哪个具体位置（虚拟地址、L_key、偏移量）。因为当远端数据返回时，它是一个独立的网络数据包，硬件必须知道这个包应该填到哪里。
- **数据重组**：对于大块数据的 Read，可能涉及多个响应包，硬件需要进行序列号匹配和重组。
- **资源占用**：NIC 内部有有限的资源（如“标记”或“上下文槽位”）来跟踪这些正在等待响应的请求。

**结论**：由于每个 Read 操作都需要消耗硬件资源来等待遥端的回应，硬件必须限制同时进行的 Read 操作数量，以防止硬件资源耗尽。这个限制本质上是为了保护**本地 NIC 的资源**。

##### RDMA Write 不需要响应 （stateless）

此中 拿 **unsignaled RDMA write** 举例：

当你发起一个 RDMA Write 操作时，情况完全不同：
1. **发送数据**：本地 NIC 将数据包直接推送到网络上，发往远端 NIC。
2. **结束**：数据发出后，本地 NIC 的任务就完成了。**它不需要等待任何来自远端的确认包。**

Write 的路径是`本端 NIC → 远端内存（DMA 写）`, 数据出 NIC → 就结束了；NIC 不需要：
```bash
- 等远端响应
- 匹配返回数据
- 维护 transaction state
```

**结论**：因为不需要等待响应，本地硬件没有资源耗尽的风险，所以不存在 `max_outstanding_write` 的概念。唯一可能阻止你发送 Write 的就是发送队列满了(send queue 满了)，这是所有操作（包括 Read 和 Write）都有的通用限制。

##### unsignaled RDMA Write 没有响应，怎么保证数据写进去了？

这就是 RDMA 依赖**可靠连接（RC）**的原因。

**unsignaled RDMA Write** 虽然没有应用层的确认(没有CQE产生)，但它依赖底层的**传输层确认机制**：
- 如果远端 NIC 因为缓冲区满等原因丢弃了某个 Write 数据包，远端 NIC 不会发送肯定确认（ACK）。
- 本地 NIC 在超时后未收到 ACK，会重传该数据包。

这个重传机制是由硬件自动完成的，对上层软件完全透明。上层软件看到的只是 WQE 在数据被成功提交给硬件（即数据包已发出）后就完成了，无需等待远端应用确认。

#### 小结

**RDMA Read 有 RAR block 问题**， RDMA Write 没有  block 问题；
RDMA 硬件需要为每个未完成的读请求（outstanding RDMA read）维护状态（如跟踪响应报文、保证数据正确写入本地内存），而硬件资源（如片上内存、标记槽位）有限。因此协议规定了一个最大未完成读请求数

### 使用 RDMA Read 的场景

#### 大规模数据分发(一对多)

- **场景：** 一个中心节点存储了元数据，多个从节点需要访问。
- **理由：** 如果使用 Write，中心节点需要记录所有从节点的内存地址并主动推送，维护成本极高。使用 Read，中心节点只需要把数据放在那，让各节点自便，实现了解耦。

#### 降低“数据一致性”风险

在某些复杂的分布式事务中，使用 Write 可能会导致对端内存被意外覆盖（如果偏移量计算错误）。
- **场景：** 数据库远程读取（Remote Fetch）。
- **理由：** Read 操作只具有“读”权限，不会破坏对端的内存状态。对于数据的安全性要求极高的金融或核心存储系统，Read 提供了一种天然的隔离感。

### 小结

|**维度**|**选 RDMA Write**|**选 RDMA Read**|
|---|---|---|
|**性能追求**|**极高**（追求满带宽、低延迟）|中等（受 RTT 延迟影响明显）|
|**控制中心**|发送方（数据源头）|接收方（数据需求方）|
|**资源消耗**|极低|较高（占用网卡 Outstanding 资源）|
|**典型应用**|存储后端复制、大规模计算梯度同步|分布式哈希表查询、配置拉取|

如果你追求**极致的数据传输性能**，请设计基于 Write 的架构；
如果你追求**系统解耦和接收端的流量控制**，Read 会让你写代码时更省心。


## RDMA Write with imm

### 理解

#### RDMA write with imm 是否需要提前post recv wr

RDMA Write with Immediate 必须有 Recv WR，否则对端 QP 报错；但这个 Recv WR 只用于接收通知，不接收数据。

Recv WR 在这个场景中：
- **不会接收数据 payload**；
- **只用于生成一个 CQE**，其中 `wc.imm_data` 字段包含 immediate 值；
- 应用可以从 `wc.byte_len` 得知 0；
- 数据本身已经写到预先注册好的远端内存里。


### RDMA Write 完成后，对端的应用程序如何知道数据已经写到了？
#### 问题

#### 方案一：RDMA WRITE with Immediate

##### 机制原理

- **（1）发送方（Sender）：** 
执行 `RDMA WRITE_WITH_IMM` 操作。个操作除了将数据直接写入对端的内存（即 Remote Direct Memory Access），还会附带一个 4 字节的 **立即数（Immediate Data）**。

- **（2）接收方（Receiver）的 HCA (网卡)：**
硬件将数据写入目标内存地址。
在数据写入完成后，硬件会生成一个 完成队列条目（Completion Queue Entry, CQE），将其推送到接收方的 **完成队列（Completion Queue, CQ）** 中。
这个 CQE 中会包含那个 **立即数**。

- **（3）接收方应用程序：**
应用程序通过 轮询（Polling）或 中断（Interrupts）机制检查其 CQ。当它获取到包含立即数的 CQE 时，就知道：
3. 数据已成功写入预期的内存区域。
4. 可以从写入的目标地址开始读取/处理数据了。
5. 立即数本身可以作为一种信令或元数据**（例如，表示写入的数据长度、消息类型或序列号）。


##### 优缺点
**（1）优点**：
完全由硬件完成通知，效率高。数据和信令同时到达并处理。

**（2）缺点**：
发送方必须通过某种提前的通信机制，获取到接收方允许写入的目标内存的地址和密钥。

#### 方案二：Rdma Write + send
`WRITE + SEND`这种组合通常用于将**大数据** (`WRITE`) 和**控制信号** (`SEND`) 分开传输。


##### 优缺点
**乱序风险（仅限 UC/UD）：** 
虽然 RC（可靠连接）模式下，WR 是按顺序处理的，但在 UC/UD 模式下，存在 **`SEND` 信号比 `WRITE` 数据先到达并处理**的风险，导致数据处理错误。`WRITE with Immediate` 将数据和信号绑定在一个原子单元中，避免了这种风险。

**网络和处理延迟累加：** 
理论上，你需要等待 `WRITE` 操作完成以及 `SEND` 操作被接收方处理。虽然这两个请求可以在网络上并行，但由于它们是序列化的逻辑事件（先写数据，再发信号），总的端到端时延是：
```bash
LatencyTotal​≈Latency_WRITE​+Latency_SEND
```

#### 方案三：应用程序层面的轮询（Polling）
##### 机制原理

**内存设计：** 
在接收方注册的内存区域中，预留一个小的内存区域（例如，一个 64 位整数）作为标志位（Flag） 或 序列号（Sequence Number）。

**（1）发送方：**
- 首先执行 `RDMA WRITE`*将主体数据写入目标内存。
- 最后执行一个小的 `RDMA WRITE`*操作，专门去修改那个标志位或序列号。

**（2）接收方应用程序：**
- 应用程序在一个循环中，周期性地检查（轮询） 本地内存中的这个标志位或序列号。
- 当发现标志位被修改或序列号增加时，就知道新数据已经到达，然后开始处理。


##### 优缺点
**（1）缺点**：
- 占用 CPU 资源
持续的轮询会消耗 CPU 周期，违背了 RDMA 旨在解放 CPU 的初衷。

- 有延迟：
轮询间隔决定了数据感知延迟。

### RDMA write with imm 和 send/recv 对比

#### 总览

- （1）`Send/Recv`：
数据被**放入接收方预先 post 的 Recv buffer**（接收方决定存放位置）。

- （2）`Write with Immediate`：
数据被**写入接收方的远端内存地址（发送方指定）**，同时在接收方 CQ 上产生一个带 `imm_data` 的 CQE（接收方必须事先 post Recv WR 否则 QP 进入 error）。接收的 CQE 只是通知，不包含 payload。

```bash
(1) Send/Recv 流程：

1> Sender: post_send(SEND, local_buf)  -> RNIC A  -> RNIC B
2> Receiver: must post_recv(recv_buf)  -> RNIC B 将数据写入 recv_buf 并产生 Recv CQE
3> App B poll CQ -> 获取 recv_buf（数据已在 recv_buf）


(2) Write with Immediate 流程：

1> Sender: post_send(RDMA_WRITE_WITH_IMM, remote_addr, imm)
         -> RNIC A 直接写 payload 到 remote_addr（不会触发 recv_buf 写）
         -> 同时在远端 CQ 生成一个 CQE（wc.imm_data）
2> Receiver: must have pre-posted recv_wr（用于匹配和接收CQ事件）
3> App B poll CQ -> 得到 imm_data，随后读取 remote_addr（或直接访问已写内存）


```


#### 相同点

`WRITE with Immediate` 因为需要对端接收一个 `CQE`（完成队列条目），看起来确实和 `SEND/RECV` 很像，因为它也需要对端预先投递一个 `RECEIVE` 工作请求（WR）来接收 Immediate Data。

- **双边参与：** 在小消息场景下，`SEND/RECV` 和 `WRITE with Immediate` 都需要接收方**预先投递一个 WR** (`RECEIVE`)，并且在操作完成后**生成一个 `CQE`**。这意味着接收方都需要参与到信令处理中。
    
- **网络往返：** 它们都涉及一次网络往返（请求和确认/完成），因此在**最小消息延迟**方面可能非常接近。有研究表明，在某些硬件上，它们的小消息延迟几乎相同。


#### 对比：核心机制和工作原理

|**特性**|**RDMA WRITE with Immediate**|**SEND / RECV**|
|---|---|---|
|**通信模式**|**混合模式**（数据单边，信令双边）|**双边模式** (Two-Sided)|
|**对端 CPU 参与**|**数据传输：** 无（Zero-Copy Offload）<br><br>  <br><br>**信令：** 极少（只处理 CQE）|**数据传输：** 无（Zero-Copy Offload）<br><br>  <br><br>**信令/数据接收：** 必须投递 `RECV` WR|
|**通知机制**|接收方 HCA 将 **Immediate Data (4 字节)** 作为 CQE 的一部分推送到 CQ，通知应用程序。|接收方 HCA 将接收到的数据放入预投递的缓冲区，然后生成一个包含**数据长度**的 CQE。|
|**内存放置**|**发送方控制**。数据写入接收方内存的**精确目标地址** ($R\text{Addr}$)。|**接收方控制**。数据写入接收方预先分配并投递的 **`RECV` 缓冲区**。发送方不知晓地址。|
|**关键参数**|**需要** $R\text{Key}$ 和 $R\text{Addr}$|**不需要** $R\text{Key}$ 和 $R\text{Addr}$|

#### 对比：性能(Latency & Throughput)

|**性能指标**|**RDMA WRITE with Immediate**|**SEND / RECV**|**优劣分析**|
|---|---|---|---|
|**CPU 负载**|**极低**。接收方 CPU 仅在轮询/等待 CQE 时被短暂唤醒。|**低**。与传统网络相比很低，但仍需要处理 `RECV` WR 和 CQE。|**`WRITE with Imm`** 的**单边数据路径**使其在接收方 CPU 效率上略胜一筹。|
|**大消息吞吐量 (Throughput)**|**极高**。受益于单边、直接内存放置和零拷贝。|**高**。受限于接收缓冲区设计，如果需要将数据从接收缓冲区复制到最终处理位置，可能会有额外的 CPU 拷贝。|**`WRITE with Imm`** / `WRITE` 的直接放置特性，使其在**大规模数据传输**时带宽利用率更高。|
|**小消息延迟 (Latency)**|**非常低**。与 `SEND/RECV` 相当，通常略优。|**非常低**。与 `WRITE with Imm` 相似。|两者在最小延迟上相近，因为它们都需要一个网络往返和一个 CQE 处理。但**`WRITE with Imm`** 通常因更简单的硬件处理路径而略快。|


#### 对比：应用场景和灵活性


|**维度**|**RDMA WRITE with Immediate**|**SEND / RECV**|
|---|---|---|
|**适用场景**|**高性能数据结构：** 分布式哈希表、队列、日志、内存缓存。**数据必须写入特定位置并附带简单信令。**|**消息传递/RPC：** 传统的请求-响应（Request-Reply）模式、控制消息交换、简单的双边数据流。|
|**Zero-Copy 实现**|**完美实现。** 数据直接从发送方应用内存 $\rightarrow$ 接收方目标应用内存。|**部分实现。** 数据从发送方应用内存 $\rightarrow$ 接收方 HCA $\rightarrow$ 接收方**预定缓冲区**。|
|**内存预注册要求**|**发送方和接收方**都需要注册并交换 $R\text{Key}$/$R\text{Addr}$。|**发送方和接收方**都需要注册缓冲区。|


|**操作**|**核心理念**|**优势**|**适用性**|
|---|---|---|---|
|**`SEND/RECV`**|**双边消息传递**，提供类似 TCP 的消息语义。|**简单、安全**，无需远程地址和密钥交换，是构建 RPC 的理想选择。|**控制流、RPC、小消息**。|
|**`WRITE with Imm`**|**单边数据传输 + 双边信令**。|**最高效**的**数据放置**和**带宽**，同时提供原子通知。|**高速数据流、内存数据库**等需要精确控制远程内存布局的场景。|




# 双边(two-side)操作（Messaging verbs）

SEND/RECEIVE是双边操作，即需要通信双方的参与，并且RECEIVE要先于SEND执行（先下发 Receive WQE，然后 Sender 才会下发 Send WQE）。
因此该过程和传统通信相似，区别在于RDMA的零拷贝网络技术和内核旁路，延迟低，多用于传输短的控制消息。

## 双向控制 Send-Receive API 流程

![](attachments/Pasted%20image%2020250323232141.png)

（1）App B（Receive 端）下发 WQE 到 RQ，描述了一个请求接受任务。
（2）RNIC B 从 RQ 获取到 WQE 并准备开始接收数据。
（3）App A（Send 端）下发 WQE 到 SQ，描述了一个请求发送任务。
（4）RNIC A 从 SQ 获取到 WQE，然后通过 DMA 的方式访问 Main Memory 的指定位置，并获得数据并封装成数据报文。
（5）RNIC A 将数据报文发送到 RNIC B。
（6）RNIC B 收到数据报文后进行校验，然后发送 ACK 到 RNIC A。
（7）RNIC B 解封装数据报文获得数据，然后通过 DMA 的方式将数据存放到指定的 Main Memory 位置。然后生成 CQE 并下发到 CQ 中。
（8）App B 接收到 CQE 的反馈。
（9）RNIC A 接受到 ACK 后，生成 CEQ 并下发到 CQ 中。
（10）App A 接受到 CQE 的反馈。


## Send 

## Send with Immediate

## RDMA Receive

## 单边和双边的关系

### 单边和双边的组合
通常在进行 Read / Write API 等单边操作之前，都需要先完成 Send-Receive API 双边操作，交换一些 QP 配置控制信令，包括：
（1）Local 和 Remote Memory Region 信息
（2）Local 和 Remote rkey（内存钥匙，控制内存的访问权限）信息
（3）etc…


# 服务类型(RC/UD等)和单边/双边操作的关系
## 各个服务类型支持的操作
![](attachments/Pasted%20image%2020250709195855.png)

![](attachments/Pasted%20image%2020250709195934.png)

## 服务类型和单边/双边操作的组合如何选择?

# 参考
```bash
# InfiniBand包头与ibverbs接口实现（一）—— RDMA WRITE分析
https://mp.weixin.qq.com/s/LCwyqruAkLJsGgZ0hzZUkg

# RDMA(11)Send操作：从Packet组织视角看操作逻辑
https://mp.weixin.qq.com/s/ndEL924IvD8xMVfZ8ELV4w

# RDMA(12)WRITE操作：从Packet组织视角看操作逻辑
https://mp.weixin.qq.com/s/yFATO8e3dyO04hhqAs25ww

# RDMA(13)READ操作：从Packet组织视角看操作逻辑
https://mp.weixin.qq.com/s/vxVAUDc8LC9ghuol1XABVg


```