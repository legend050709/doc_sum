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

# 操作的整体介绍

![](attachments/Pasted%20image%2020250323232553.png)

1. 提交任务：App 通过将 WQE 放入 SQ / RQ 来提交一个任务。
2. 完成任务：RNIC 根据 WQE 执行任务，然后生成一个 CQE，包含该任务的完成信息，并放入 CQ。
3. 检查结果：App 检查 CQ，确认任务完成情况。如果失败，则可以查看 CQE 信息来了解失败的原因。


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


## RDMA Send 
## RDMA Receive

## 单边和双边的关系
通常在进行 Read / Write API 等单边操作之前，都需要先完成 Send-Receive API 双边操作，交换一些 QP 配置控制信令，包括：
（1）Local 和 Remote Memory Region 信息
（2）Local 和 Remote rkey（内存钥匙，控制内存的访问权限）信息
（3）etc…



# 服务类型(RC/UD)和单边/双边操作的关系

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