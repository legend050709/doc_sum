```table-of-contents
```
# 介绍
TIMELY（RTT-based Congestion Control for the Datacenter）是 Google 在 2016 年提出。研究人员发现，依赖交换机打 ECN 标记（如 DCQCN）存在两个问题：
```bash
1. 需要交换机硬件支持特定功能，部署门槛高。
2. ECN 阈值设置困难，且反应速度受限于 RTT（往返时延）。
```
当拥塞发生时，队列堆积会导致 **RTT (Round Trip Time)** 增加。RTT 的变化比丢包或 ECN 标记更早、更敏感地反映拥塞。

TIMELY是业界首个在数据中心内完全基于延迟测量(RTT)的拥塞控制协议，它的出现是为了解决对交换机ECN功能的依赖（不需要交换机支持 ECN，只需标准以太网交换机），通过端侧（End-to-End）自身的测量来感知拥塞。

# 原理
**信号来源**：端到端往返时间（RTT）。通过高精度地测量数据包的RTT变化来感知路径拥塞程度。

**控制方式**：基于速率的梯度计算。它不只看RTT的绝对值，更关注**RTT的梯度（即变化率）**。当检测到RTT有增加的趋势（正梯度），判断为拥塞，执行速率衰减；当RTT趋于稳定或下降（负梯度），则尝试增加速率。
```bash
1. RTT 测量: 发送端记录每个发出的包的时间戳，收到 ACK 时计算 RTT。
2. 梯度计算: ΔRTT=RTTcurrent−RTTold
```

**特点**：不依赖交换机ECN功能，但要求网卡硬件支持**高精度（微秒级）的RTT测量**。

# 架构
## RTT测量引擎(RTT Measurement Engine)
## 速率计算引擎(Rate Computation Engine)
## 速率整形引擎 (Pacing Engine)

# 参考
```bash
# Timely SIGCOMM 2015
## RTT-based Congestion Control
https://yi-ran.github.io/2019/03/27/Timely-SIGCOMM-2015/
```