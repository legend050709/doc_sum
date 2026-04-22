```table-of-contents
```
# RD
## 为什么没有实现RD服务类型

# XRC
## 为什么需要XRC
当前的计算节点一般都有多核，因此可以运行多进程。在这样的计算节点组成的集群中，如果想用RC连接建立full mesh的全连接拓扑时，每个节点就需要建立N*p*p个QP(这里假设集群有N个节点，每个节点上有p个进程，需要让任何2个进程都连通)。当集群扩张，N和p同时增长时，一个节点所需的RC QP资源将变得不可接受。

XRC的思想是当一个进程想与某个远程节点的p个进程通信时不需要跟各个进程建立p个连接而只需要跟对端节点建立一个连接，连接上传输的报文携带了对端目的进程号(XRC SRQ)，报文到达连接对端(XRC TGT QP)时根据进程号分发至各个进程对应的XRC SRQ。这样源端进程只需要创建一个源端连接(XRC INI QP)就能跟对端所有进程通信了，这样所需总的QP数量就会除以p。

![](attachments/Pasted%20image%2020260401164014.png)

## 概念

**XRC INI QP**：XRC发起端QP，是XRC操作的源端队列，用于发出XRC操作，但它没有接收XRC操作的功能，对比常规RC QP来说可以认为它是只有SQ没有RQ。XRC操作在对端由XRC TGT QP处理。

**XRC TGT QP**：XRC接收端QP，它处理XRC操作将其分发至报文SRQ number对应的SRQ。XRC TGT QP只能接收XRC操作，但它没有发出XRC操作的功能，对比常规RC QP来说可以认为它是只有RQ没有SQ。XRC操作在对端由XRC INI QP发出。

**XRC SRQ**：接收缓冲区(receive WQE)被放在XRC SRQ中以接收XRC请求，XRC请求中携带了XRC SRQ number，所以XRC TGT QP收到报文后会从报文指定的XRC SRQ中取receive WQE来存放XRC请求。

**XRC domain**：用于关联XRC TGT QP和XRC SRQ，XRC报文只能指定与XRC TGT QP在同一domain内的XRC SRQ，否则报文会被丢弃。这起到了隔离资源的作用，防止攻击报文随意指定XRC SRQ。

XRC INI QP和XRC TGT QP是一一对应的，host2上的每个进程在远端节点host0上都有自己对应的XRC TGT QP。XRC的共享体现在一个XRC TGT QP可以分发至多个XRC SRQ。一个进程一般只有一个XRC SRQ，它可以接收多个XRC TGT QP来的包。

# DC
## 为什么需要Dynamically Connected  (DC)
![](attachments/Pasted%20image%2020260401163031.png)

UD虽然**连接扩展性**很好，但是不支持read/write单边语义。RC虽然支持read/write单边语义，但是连接扩展性不好。
DCT的初衷就是融合2者的优点，保持RC的read/write单边语义和可靠连接特性，同时像UD一样用一个QP去跟多个远端通信，保持良好的可扩展性。DCT一般用于稀疏数据场景。
```bash
N 个节点全互联时，每个节点需要 (N-1) 个 RC QP
集群规模 → QP 数量：
  100 节点  → 每节点 99 个 QP
  1000 节点 → 每节点 999 个 QP
  10000 节点 → 每节点 9999 个 QP
```

具体问题：

|   |   |
|---|---|
|问题|影响|
|QP 内存占用|每个 RC QP 在 HCA 内需要 QPC（数 KB～数十 KB），千节点规模下 MB 级内存|
|HCA QP 上限|大多数 HCA 支持最多 64K QP，千节点集群接近上限|
|QP context cache miss|HCA 内 QPC cache 内存有限；QP 数量多，context cache miss高，P99 延迟抖动大|
|地址交换开销|每条 RC 连接需要 QP 地址交换|
|QP 状态机迁移|INIT→RTR→RTS 每个阶段都需要 ibv_modify_qp，建立连接慢|



## 什么是DC（Dynamic Connected）
![](attachments/Pasted%20image%2020260401163116.png)
**DC（Dynamic Connected，动态连接）** 是 NVIDIA/Mellanox 在 ConnectX-4 引入的 mlx5 专有传输类型；

DC具有非对称的API：DC在发送侧的部分称为`DC initiator(DCI)`，在接收侧的部分称为`DC target(DCT)`。DCI和DCT不过是特殊类型的QP，它们依然遵循基本的QP操作，比如`post send/receive`。

## DC：临时建立的RC连接
DC：**临时的RC连接**。

### DCI（DC Initiator）: 发送端
在DCI上发送的每个`send-WR`都携带了目的地址信息。
> 注：DCI 不绑定固定目标，每个 WQE 通过 `Datagram Segment` 指定目标地址（DCT 号 + LID/GID）

**DCI 只能发送，因此`max_recv_sge = 0`**;
如果DCI当前连接的对端不是`send-WR`里携带的对端，则它会首先断开当前的连接，再连接到`send-WR`里携带的对端。只要后续的`send-WR`里携带的都是当前已连接对端，则都可以复用当前已建立的连接。如果`DCI`在一段指定的时间内都没有发送操作则也会断开当前连接。
注意：==DC每次**临时建立**的是一个**RC可靠连接**==。

### DCT（DC Target)：接收端
DCT 类似"服务器监听端口"：始终处于 RTR 状态，接收所有打过来的 DC 消息。

## DC的可靠性保证

DC 与 RC 具备相同的可靠性级别：

- 硬件保证报文序号、ACK/NACK、重传
    
- 支持 RDMA Read/Write/Send/Atomic 全部操作
    
- 固有延迟 +60ns（DCI 动态连接 Datagram seg + CQE解析）

```bash
发送方 DCI  ──[可靠传输]──►  接收方 DCT
              ↑
    硬件自动重传（RNR retry / timeout retry）
    硬件序列号保序
    硬件 ACK/NAK 确认
```
DC 的英文全称是 **Dynamic Connected**。
=="Dynamic"指的是**发送侧的 DCI 动态复用**，接收侧的 DCT 是持久存在的（类似一个"服务端口"），一直在线等待==。

## DC增强连接可扩展性的核心思路

> 发送侧：少量 DCI（DC Initiator QP）服务所有 conn
> 接收侧：一个持久化的 DCT（DC Target QP）接收所有来源的消息

### 池化
**DCT的池化**：每个DCT有一个responders(DCRs)池，新进的DC连接会在这个池里分配一个DCR(DC resource)。当池资源不足时DCT会向发起新建连接的DCI回复connection NAK(CNAK)，同时丢弃来自这个DCI的后续报文。

**DCI的池化**：当我们需要跟多个对端通信时，为避免一个DCI频繁建立/断开连接从而影响性能，一般需要建立一个`<dest DCI>`的哈希表，新连接走最老的DCI(LRU策略)。当池里DCI太少时，一个DCI会在不同的对端频繁切换，严重时建链报文数量会等同数据报文数量，这会大大恶化时延。

### DCI 的动态复用
"Dynamic" 指的是发送侧 DCI 的动态复用。
==DCI 每次发送 WQE 时都携带**目标 DCT 号 + 地址信息（GID/LID）**，硬件根据 WQE 动态决定发给谁。这就是"Dynamic"的含义==。

```bash
多个 EP（也就是conn） 共享少量 DCI：

DC 模式（N=4个peer，2个DCI）：
  EP₁ ─┐
  EP₂ ─┤─► DCI₁ ──[动态切换目标]──► peer₁/peer₂/peer₃/peer₄ 的 DCT
  EP₃ ─┤─► DCI₂                     （每次发送时在WQE里指定目标地址）
  EP₄ ─┘

每次发送：
  1. 从 Pool 取一个可用 DCI
  2. WQE 中嵌入目标地址（DCT_num + LID）
  3. 发送完成后 DCI 可以归还给 Pool
```


范例：
```bash
RC 模型（N=4 节点）：              DC 模型（N=4 节点）：
Node A                             Node A
  QP_AB ──── Node B                  DCI[0] ─→ DCT_B
  QP_AC ──── Node C                  DCI[1] ─→ DCT_C
  QP_AD ──── Node D                  DCI[2] ─→ DCT_D
                                      DCT_A ←─ 所有来源
每节点 3 个 QP，共 12 个            比如：UCX项目中，每节点 33 个 QP（32 DCI + 1 DCT）
1000 节点：每节点 999 个 QP         1000 节点：每节点仍 33 个 QP
```




## 半握手 和 全握手
**半握手**：第一个建链报文之后不等ACK就发数据报文，能有效减少小包时延。
**全握手**：类似TCP三次握手，能减少潜在的竞争条件。

## 优缺点
### 优点
#### 大集群中减少QP数量
在大集群（多连接）的场景下，可以显著减少每个节点的QP的数量。

#### O(1) 连接建立
DC EP（即 conn） 创建只需对端 DCT 号 + GID，无需 QP 状态机迁移（无 INIT→RTR→RTS） ；
新 conn 加入不影响已有 conn。

#### 硬件级可靠性

与 RC 相同的 ACK/NACK/重传机制（RNR、Transport Retry 等保护机制完整）
支持全部 RDMA 操作（Write/Read/Send/Atomic）

### 缺点
#### 固定延迟 +60ns
```bash
原因：每个 WQE 需要额外的 Datagram Segment（目标地址），
      CQE 解析时需要识别来源 DCI。
影响：延迟敏感场景（如 RPC < 1μs 目标）有影响。
```

#### DCI 竞争导致额外排队延迟
```bash
场景：32 个 DCI 全部被占用，新的 EP(conn) 需要等待 DCI 释放。
影响：在 EP(conn)  数量 >> num_dci 且通信突发时，P99/P999 延迟抖动大。
缓解：增大 num_dci（UCX_DC_MLX5_NUM_DCI=64 或更大）。
```

#### 硬件依赖
仅 `Mellanox ConnectX-4+（mlx5）` 支持，其他厂商网卡以及`ConnectX-3` 不可用。


## perftest使用DCT
```bash
# 使用 DC（Dynamic Connection）模式  
ib_write_bw -d mlx5_0 -c DC -q 8 192.168.1.1
```

![](attachments/Pasted%20image%2020260327203038.png)

## DC和RC对比

|维度|RC（Reliable Connected）|DC（Dynamic Connected）|
|---|---|---|
|**连接方式**|一对一固定连接，N个peer需要N个QP|多对多，N个peer共享少量DCI QP|
|**可靠性**|硬件保证可靠（重传、序号、确认）|同样硬件保证可靠|
|**QP数量**|O(N)，100节点需100个QP|O(1)，固定几个DCI即可|
|**连接状态**|有状态（QP绑定到对端）|**无状态接收端（DCT）**|
|**RDMA支持**|Write/Read/Atomic/Send|Write/Read/Atomic/Send 全支持|
|**硬件支持**|所有IB/RoCE|仅 ConnectX-4 及以上（mlx5）|


### DC/RC/UD对比

|   |   |   |   |
|---|---|---|---|
|维度|RC|DC|UD|
|可靠性|✅ 硬件保证|✅ 硬件保证|❌ 不可靠|
|QP 数量（N节点）|O(N)|O(1)|O(1)|
|连接建立|慢（状态机迁移）|快（iface-level）|极快（无连接）|
|额外延迟|基准 0ns|+60ns|基准 0ns（但重传慢）|
|RDMA Read/Write|✅ 支持|✅ 支持|❌ 不支持|
|Atomic 操作|✅ 支持|✅ 支持|❌ 不支持|
|FC 机制|✅ 完整|✅ 完整（更复杂）|❌ 不适用|
|错误处理|✅ 完整|⚠️ 部分策略|❌ 有限|
|硬件要求|全部 IB/RoCE|ConnectX-4+|全部 IB/RoCE|
|消息大小|任意|任意|≤ MTU（4096B）|
|适用规模|小/中集群|大/超大集群|超大集群（不可靠）|



## 其他
### perftest使用大页
```bash
# 配置大页  
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages  
  
# 在测试时使用大页  
ib_write_bw -d mlx5_0 -x 0 --use_huge
```

### UCX中的DC
UCX中实现了DC。

## 场景
### 推荐使用 DC的场景

|   |   |
|---|---|
|场景|原因|
|AI 大规模训练（1000+ GPU）|稀疏通信模式（all-to-all 但每次只和少数节点通信）|
|MPI 大规模集群|MPI AllReduce/Scatter/Gather，多对多，RC QP 数量爆炸|
|参数服务器（PS-Worker）|PS 节点要连接所有 Worker，连接数量极多|
|微服务 RPC（大扇出）|每个服务连接数百个后端，连接频繁建立释放|
|Hadoop/Spark 大集群|多节点随机通信，QP 资源有限|


小集群：数十节点，推荐使用 RC（QP 数量可控，无 DCI 调度开销）。



# 软件层自实现的连接可扩展性
## per-thread的VRC
## per-procoss的VQP

# 参考
```bash

```