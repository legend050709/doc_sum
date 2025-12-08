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


## RDMA write with imm

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
1. 数据已成功写入预期的内存区域。
2. 可以从写入的目标地址开始读取/处理数据了。
3. 立即数本身可以作为一种信令或元数据**（例如，表示写入的数据长度、消息类型或序列号）。


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