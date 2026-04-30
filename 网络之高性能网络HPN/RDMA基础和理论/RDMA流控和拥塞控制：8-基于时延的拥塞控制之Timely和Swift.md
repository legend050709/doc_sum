```table-of-contents
```
# Timely
## 前言
低延迟、高吞吐一直是网络传输追求的目标，不管是在广域网，还是datacenter。
TIMELY是数据中心里第一个基于延迟的拥塞控制算法。

### 为什么之前没有人在datacenter使用delay作为拥塞信号
我们先说说为啥在datacenter使用delay作为拥塞信号不合适，因为datacenter里对RTT的精度要求为微秒，而微秒级别的queuing delay很容易被host delay淹没掉。所以DCTCP避开了基于延迟的方案，因为它认为“准确测量排队延迟的微小增加是一项艰巨的任务。”

那timely为什么认为使用基于delay的拥塞控制方案可行呢？有两个点：一是这归功于硬件能力的提升，新型的NIC支持给数据包打微秒级时间戳；二是timely的rtt并不包含host delay，排除host delay的干扰后使得switch queuing的测量变得可行。

## RTT作为拥塞信号的价值
**RTT可以直接反映延迟**
网络排队造成的端到端延迟波动可以直接被测量。这是ECN这类方法无法做到的，因为只有当数据包在队列中的数量超过了设定的阈值才会打标记，这无法转换成延迟。

**RTT可以被精确测量**
精确测量RTT是datacenter里的一个难点，datacenter里RTT的一般是微秒级，广域网里RTT一般是毫秒级，另外RTT的测量会被诸多因素干扰，比如由于内核调度导致的可变性；NIC 性能技术，包括卸载（GSO/TSO、GRO/LRO）； 协议处理，例如 TCP 延迟确认。 在这些因素中的每一个都对数据中心中RTT的测量造成严重影响。

**RTT的局限**
虽然RTT作为拥塞信号很有价值，但RTT包括往返两个方向的延迟测量，我们实际关心的是数据包发送方向经历的拥塞，那么ACK 经历的反向路径拥塞也会误导我们认为网络发生了拥塞。解决办法就是提高ACK包的优先级，避免ACK数据包经历queuing delay。

![](attachments/Pasted%20image%2020260426180451.png)


## TIMELY介绍
TIMELY（RTT-based Congestion Control for the Datacenter）是 Google 在 2016 年提出。研究人员发现，依赖交换机打 ECN 标记（如 DCQCN）存在两个问题：
```bash
1. 需要交换机硬件支持特定功能，部署门槛高。
2. ECN 阈值设置困难，且反应速度受限于 RTT（往返时延）。
```
当拥塞发生时，队列堆积会导致 **RTT (Round Trip Time)** 增加。RTT 的变化比丢包或 ECN 标记更早、更敏感地反映拥塞。

TIMELY是业界首个在数据中心内完全基于延迟测量(RTT)的拥塞控制协议，它的出现是为了解决对交换机ECN功能的依赖（不需要交换机支持 ECN，只需标准以太网交换机），通过端侧（End-to-End）自身的测量来感知拥塞。

## TIMELY原理
**（1）信号来源**：
端到端往返时间（RTT）。通过**高精度地测量数据包的RTT变化来感知路径拥塞程度**。

**（2）控制方式**：
基于速率的梯度计算：
	它不只看RTT的绝对值，更关注**RTT的梯度（即变化率）**。当检测到RTT有增加的趋势（正梯度），判断为拥塞，执行速率衰减；当RTT趋于稳定或下降（负梯度），则尝试增加速率。
```bash
1. RTT 测量: 发送端记录每个发出的包的时间戳，收到 ACK 时计算 RTT。
2. 梯度计算: ΔRTT=RTTcurrent−RTTold
```

**（3）特点**：
不依赖交换机ECN功能，但要求**网卡硬件支持高精度（微秒级）的RTT测量**。

## TIMELY架构
timely框架有三个组成部分，分别是RTT测量引擎，速率计算引擎和速率控制引擎如图4所示。

![](attachments/Pasted%20image%2020260426180647.png)


### RTT测量引擎(RTT Measurement Engine)

为了消除host delay造成的影响，timely的RTT只包含传播延迟和发生在switch上的queuing delay两部分。具体就是NIC在将数据包发送出去时打一个硬件时间戳，然后收到响应的ACK包时，NIC再打一个时间戳。但是数据包从NIC被发送到网络这个过程也是有时间的，这取决于数据包大小和NIC速率，所以RTT等于：
![](attachments/Pasted%20image%2020260426180808.png)

### 速率计算引擎(Rate Computation Engine)：基于速率的拥塞控制

速率计算引擎的输入为RTT测量引擎计算得到的RTT，输出为flow rate；
这个rate是根据运行在RTT计算引擎内部的拥塞控制算法得到的。

![](attachments/Pasted%20image%2020260426181100.png)

timely拥塞控制算法同FAST TCP，Compound TCP一样都是基于delay的，只不过timely适用于datacenter。

### 速率整形引擎 (Pacing/shaping Engine)：主机侧peacing 控制发包间隔
当message准备发送时，速率控制引擎将大的message分解为多个segments（这样可以防止突发），并依次将每个segment发送给调度器。为了提高运行时效率，timely实现了一个处理所有流的调度器（整流引擎）。

### 小结
TIMELY 的设计哲学是：**把“控制逻辑”放在软件，把“精确测量”交给硬件**;

```bash
  +----------------------+
  |   Host Software      |
  |----------------------|
  | RTT smoothing        |
  | RTT gradient         |
  | Rate control (AI/MD) |
  | Pacing logic         |
  +----------+-----------+
			 |
			 v
  +----------------------+
  |        NIC           |
  |----------------------|
  | Timestamping         |
  | Packet TX/RX         |
  | (optional pacing)    |
  +----------------------+
```

|引擎|实现位置|
|---|---|
|RTT计算引擎|**NIC硬件（timestamp） + 软件处理**|
|速率计算引擎|**主机软件（核心逻辑）**|
|流量整形引擎|**主机软件为主（可选NIC辅助pacing）**|

① RTT 计算引擎 — **硬件 + 软件协同**
分工小结：**NIC 硬件**提供精确时间戳 + 及时 ACK 生成；**主机软件**完成 RTT 数值计算。
关键原因：内核软件时间戳会引入调度、中断等噪声，而硬件时间戳能将噪声降到微秒级别以下，这是 TIMELY 能用 RTT 作为拥塞信号的前提。

② 速率计算引擎（Rate Computation Engine）— **纯软件**

③ 流量整形（Pacing）引擎 — **主机软件为主，可借助 NIC 硬件**
数据首先被批量组成 64 KB 的段，然后调度器计算两个段之间的 pacing delay 并将段交给 NIC 发送。Pacing 逻辑在主机软件中计算，部分 NIC（如支持硬件 pacing 的 Mellanox ConnectX）可以承接硬件级别的 pacing 执行。


## TIMELY优缺点

## 后续发展

没有哪一代主流数据中心拥塞控制是“全在硬件里完成”的。
真正的演进方向是：**控制决策尽量在软件，精确测量与执行尽量下沉到 NIC 硬件**。
TIMELY、Swift、Falcon 都遵循这个思路，只是“下沉程度”越来越深。

### 演进趋势（关键主线）
从 TIMELY 开始，业界发现两个问题：
1. 软件控制 loop 太慢（µs 级网络）
2. 每包参与 CPU，成本太高

演进方向变成：**控制逻辑逐步向 NIC 下沉（offload），但不完全硬件化**

# Swift

# 参考
```bash
# Timely SIGCOMM 2015
## RTT-based Congestion Control
https://yi-ran.github.io/2019/03/27/Timely-SIGCOMM-2015/
```