```table-of-contents
```
# 背景

## 案例
### 案例1：PFC风暴
一个使用RDMA的云存储（测试）集群曾经遇到了一场持续时间较长的PFC风暴导致的网络范围内的大幅度流量下降。这是由一个大型incast事件和一个供应商bug触发的，该错误导致交换机无限期地发送PFC暂停帧。

由于incast事件和拥塞是这类集群中的常态，并且我们不确定是否会有其他供应商的错误导致PFC风暴，因此我们决定尽最大努力防止任何PFC暂停。因此，我们调整了CC算法以快速降低速率（TD小），并保守地增加速率（TD大），以避免触发PFC暂停。
我们确实得到较少的PFC暂停（风险较低），但网络中的平均链路利用率非常低（成本较高）。

### 案例2：惊人的长延迟（老鼠流延迟受到大象流排队影响）
一个机器学习（ML）应用程序抱怨短消息的平均延迟>100us；预期RDMA的尾部时延<50us。我们最终发现，延迟时间长的原因是网络中的队列主要被同一集群中的带宽密集型云存储系统占用。因此，我们必须通过将ML应用程序部署到新集群来分离这两个应用程序。考虑到ML应用程序不太需要带宽，新集群的利用率较低（成本较高）。

## DCQCN 和 TIMELY的问题
HPCC之前，在RDMA网络已有相关的CC工作提出，比如TIMELY、DCQCN，但他们都或多或少存在一些下文说的弊端：

**（1）收敛慢**
对于粗粒度反馈信号，如ECN或RTT，当前的CC方案不知道增加或减少多少发送速率。因此，他们使用启发式来猜测速率更新，并尝试迭代收敛到稳定的速率分布。这种迭代方法处理大规模拥塞事件的速度很慢，如案例1所示。

**（2）不可避免的排队时延**
DCQCN 需队列累积后才被 ECN 标记发现拥塞；TIMELY 需 RTT 上升才检测拥塞——两者都"先排再减"
因此，sender仅在buffer出现排队后才开始降低速率， 这时可能为时已晚，排队会显著增加网络时延。

**（3）复杂的调参机制**
当前CC算法用于调整发送速率的启发式算法有许多参数需要针对特定网络环境进行调整。
DCQCN 需设置 15 个参数，不同流量模式/拓扑/链路速率/Buffer 大小均需单独调参，耗时数月。

### DCQCN 的两对矛盾
#### 带宽 vs 稳定性的矛盾
- Ti 小 Td 大 → 高吞吐但 PFC 频发；
- Ti 大 Td 小 → 低 PFC 但吞吐低；

#### 带宽 vs 延迟的矛盾
- ECN 阈值低 → 队列小但发送方保守不敢增流；
- ECN 阈值高 → 延迟大但带宽利用率好；


## DCQCN 和 TIMELY的问题分析
HPCC paper认为上述三个限制的根本原因是传统网络中==缺乏细粒度的网络负载信息==。
ECN是终端主机可以从交换机获得的唯一反馈，而RTT是纯粹的端到端测量，没有交换机的参与 。

然而，这种情况最近发生了变化。随着新的交换ASIC中提供的网络遥测（INT，In-network telemetry）功能，在生产网络中获得细粒度网络负载信息并使用它改进CC已成为可能。



# 基本概念 

INT：in-network telemetry，in-band network telemetry，网内遥测，带内网络遥测，In-band network telemetry via programmable dataplanes

QCN(Quantized  Congestion Notification,量化拥塞通知)是一套应用于L2的端到端拥塞通知机制

PFC，priority flow control

Coarse-grained / fine-grained：粗粒度、细粒度

NACK：接收端主动通告没有收到的报文，发送方收到后go -back-N，从N开始全部重新发送

Tail latency：尾部延迟， 指客户端很少看到的高延迟

# HPCC
## 介绍
阿里巴巴的技术人员研发了新一代高速云网络拥塞控制协议**HPCC （高精度拥塞控制：High Precision Congestion Control）**，旨在同时实现高速云网络的极致性能和超高稳定性。目前这一成果已被计算机网络方向世界顶级学术会议ACM SIGCOMM 2019收录，引起了国内外广泛关注。

## 原理
### 思想
HPCC 的核心创新是 利用 INT 获取精确链路负载信息，直接计算精确的速率更新，而非启发式迭代。

### INT机制

每个数据包从发送方到接收方途经的每台交换机，在包头插入该出端口的实时状态：

|INT 字段|含义|用途|
|---|---|---|
|ts (timestamp)|包离开出端口的时间戳|计算 txRate|
|txBytes|该端口累计发送字节数|计算 txRate|
|qLen|当前队列长度|直接反映排队|
|B|链路带宽容量|归一化计算|

接收方将所有 INT 元数据复制到 ACK 中返回发送方。

###  控制 Inflight Bytes（而非 Rate）

HPCC 是 基于窗口的 CC 方案，控制 inflight bytes（已发未确认数据量）。

为什么控制 inflight bytes 而不是速率？

- 无拥塞时：inflight Bytes(带宽时延积) = rate × T（T 为基础传播 RTT），二者等价
- 有拥塞时：纯速率方案在反馈延迟到达前持续发包 → 网络过载；窗口方案在达到窗口限制时立即停发，不管反馈延迟多久 → 网络天然稳定

### 拥塞信号与控制律
归一化 inflight bytes：
```bash
U_j = I_j / (B_j × T) = (qLen_j + txRate_j × T) / (B_j × T)

其中：
- I_j = 链路 j 的总 inflight bytes = 队列中数据 + 管线中数据
- qLen_j 来自 INT
- txRate_j = (ack1.txBytes - ack0.txBytes) / (ack1.ts - ack0.ts) 由连续两个 ACK 计算
- txRate_j × T 估算管线中数据量
```
目标：让每条链路的 I_j ≈ η × B_j × T（η 接近 1，如 95%），即利用率接近满载但无排队。

窗口调整公式：
```bash
W_i = Wc × max(U_j) / η + W_AI
```
第 1 项：乘法增减 (MI/MD) — 快速收敛到合适速率（效率+稳定性）
第 2 项：加法增加 (AI) — 逐步实现公平性（长流公平）

### 防过度反应机制

问题：连续 ACK 报告的是同一批包/队列状态 → 逐 ACK 反应会过度降窗。

解决方案：引入 参考窗口 Wc（per-RTT 更新）：
- 仅当 ACK.seq > lastUpdateSeq 时才更新 Wc = W（即新一轮窗口的 ACK 到达）
- 其余 ACK 只更新 U 值，但用固定的 Wc 计算新 W

结果：同 RTT 内多个 ACK 不会连续缩小窗口；但 U 的变化仍可触发调整 → 快反应不过度



## 目标
1》充分利用带宽且不拥塞；
2》不排队实现超低时延；
3》解决公平性问题
4》易于部署

## 挑战
在CC中利用INT信息并不简单。设计HPCC有两个主要挑战。
（1）首先，链路拥塞会延迟数据包上承载的INT信息，从而延迟流量降低以解决拥塞。在HPCC中，我们的CC算法旨在限制和控制繁忙链路的飞行中总字节数（total inflight bytes），防止发送方发送额外流量，即使反馈延迟。
（2）第二，尽管所有ACK数据包中都包含INT信息，但如果发送方盲目地对所有信息做出快速反应，则可能会出现破坏性的过度反应。我们的CC算法通过结合每次确认和每次RTT反应，选择性地使用INT信息，实现快速反应而无过度反应。


## 流程

![](attachments/Pasted%20image%2020260426162729.png)


INT 包格式与开销：
```bash
UDP Header | nHop(4bit) pathID(12bit) | 1st-Hop(64bit: B+TS+txBytes+qLen) | 2nd-Hop(64bit) | ... | IB BTH
```
- nHop: 跳数计数器（交换机逐一加 1）
- pathID: 所有交换机 ID 的 XOR（检测路径变更）
- 每跳 64 bit INT 元数据
- 5 跳路径 ≈ 42 字节（42=`8*5+2`, 在 1KB 包中仅占 4.2%）


## 依赖

|依赖项|说明|
|---|---|
|带INT 功能的可编程交换机|Barefoot Tofino、Broadcom Tomahawk3/Trident3 等支持 INT 的可编程 ASIC|
|可编程 NIC (FPGA)|发送方 CC 算法实现在 NIC FPGA 上（约 2000 行 Verilog）|
|RoCEv2 基础设施|标准 RoCEv2 网络栈（IB BTH + UDP/IP 封装）|
|P4 可编程性|交换机 INT 插入逻辑约 300 行 P4 代码|
|均匀 RTT 假设|数据中心 Clos 拓扑中大多数服务器对的 RTT 很接近（HPCC 利用此假设）|


## 优缺点
### 优点

|优点|说明|
|---|---|
|超低延迟|队列接近零（median=0, P99=22.9KB/7.3μs），mice 流延迟接近基础 RTT|
|快速收敛|单瓶颈场景 1 个 RTT 即收敛（vs DCQCN 多轮迭代）|
|仅 3 个参数|η=95%, maxStage=5, W_AI=80bytes — 均非可靠性关键参数|
|高稳定性|窗口限制 inflight → 反馈延迟也不致过载 → 无 PFC storm|
|公平性|MI/MD 保效率稳定，AI 保长流公平；数秒内达到公平|
|部署友好|交换机仅需标准 INT；NIC 上实现简单（顺应现有 RoCEv2 范式）|
|短流 FCT 大幅降低|95% reduction in 99th percentile FCT「Flow Completion Time（流完成时间）」 vs DCQCN|


### 缺点

|缺点|说明|
|---|---|
|硬件依赖|需全网 INT 交换机 + FPGA NIC → 部署门槛高，需 fabric 级改造|
|INT 开销|每包额外 ~42 字节（5 跳），小包场景开销占比更高|
|均匀 RTT 假设|依赖数据中心拓扑的 RTT 规律性，跨域/异构拓扑不适用|
|多瓶颈场景|多瓶颈时 inflight bytes 估算为下界 → 需多轮收敛|
|路径变更检测|需 pathID (XOR of switch IDs) 检测路由变更，变更后丢弃旧状态重建|
|NIC 并发流数限制|FPGA 实现 6 个引擎支持 300 流/25GE，ASIC 可达 9K 流|
|仅支持 RDMA WRITE/READ|完整 IB Verbs 支持未实现（论文声明为 future work）|

## 适用场景

|场景|契合度|说明|
|---|---|---|
|大规模数据中心 RDMA 网络|★★★★★|HPCC 原生设计场景|
|Clos/Fat-tree 拓扑|★★★★★|均匀 RTT 假设成立|
|延迟敏感应用 (ML训练, 分布式存储)|★★★★★|near-zero queue 保证低延迟|
|Incast 频发环境|★★★★☆|单瓶颈 Incast 1 RTT 解决；多瓶颈稍慢|
|PFC 风险环境|★★★★☆|窗口控制避免过载 → 减少 PFC 触发|
|小规模/异构拓扑|★★☆☆☆|RTT 不均匀，INT 交换机可能不具备|
|跨数据中心/WAN|★☆☆☆☆|RTT 差异大，INT 部署不可行|

## 和其他方案对比

|维度|DCQCN|TIMELY|DCTCP|HPCC|
|---|---|---|---|---|
|反馈信号（拥塞信号）|ECN (1-bit)|RTT 变化|ECN (multi-bit)|INT (精确: ts+txBytes+qLen+B)|
|控制方式|Rate-based|Rate-based|Window-based|Window-based (inflight bytes)|
|参数数量|15|多个|少|3|
|收敛速度|慢（迭代多轮）|慢|中等|快（1 RTT，单瓶颈）|
|队列行为|围绕 ECN 阈值固定队列|队列累积后才检测|阈值处队列|接近零|
|稳定性|PFC storm 风险|较好|中等|高（窗口硬限制）|
|硬件要求|标准 RoCEv2 NIC|标准 NIC|标准 NIC|INT 交换机 + FPGA/可编程 NIC|
|部署难度|低|低|低|高（需 fabric 改造）|
|公平性|差（需多轮迭代）|差|中等|好（MI/MD+AI 分离）|
|延迟 (P99)|~35μs+ (mice流)|> DCQCN|< DCQCN|~5.4μs (接近基线RTT)|
|短流 FCT 降低|基准|基准|优于 DCQCN|95% 降低 vs DCQCN|



|指标|HPCC|DCQCN|TIMELY|
|---|---|---|---|
|99th percentile 延迟|~5.4μs|>35μs|> DCQCN|
|队列大小 (median)|0|固定队列 ~几百KB|固定队列|
|队列大小 (P99)|22.9KB (7.3μs)|~几百KB|~几百KB|
|Incast 队列峰值|快速清空|550KB|> DCQCN|
|PFC 触发 (320节点仿真)|不触发|频发|频发|
|带宽利用率|η=95% → 仅损失5%|受ECN阈值制约|受RTT梯度制约|

# 参考
```bash
# 【SIGCOMM】阿里巴巴新一代高速云网络拥塞控制协议HPCC
https://zhuanlan.zhihu.com/p/420896292


```