```table-of-contents
```
# 背景

## 为什么要使用 TCP Pacing？

在没有 Pacing 的情况下，TCP 的行为通常表现为 **"Micro-bursts"（微突发）**：

- **ACK 压缩**：当大量的 ACK 包在回传路径上被压缩在一起时，发送端会瞬间收到大量 ACK，从而触发瞬间连续发送大量数据包。
    
- **缓存溢出**：即使网络平均带宽足够，这种瞬间的“微爆发”也可能超过中间路由器或交换机的瞬时缓存（Buffer）容量，导致丢包。
    
- **RTT 波动**：大量包堆积在队列中会导致往返时间（RTT）剧增。

在传统的 TCP 协议中，只要拥塞窗口（cwnd）允许，发送端会尽可能快地将所有数据包送入网络。这就像是在绿灯亮起时，所有的车瞬间同时涌入路口，极易造成瞬间拥塞。而 Pacing 就像是一个红绿灯限速器，让车一辆接一辆匀速通过。


# 介绍

TCP的Pacing机制是一种流量整形技术，用于**控制数据包的发送节奏，避免数据突发（Burst）导致的网络拥塞或丢包**。
传统TCP的拥塞控制（如**慢启动、拥塞避免**）主要调整发送窗口大小，但窗口内的数据仍可能以背靠背方式**突发**发送。

Pacing通过将窗口内的数据包**以均匀时间间隔发送**，而非一次性突发，使流量更**平滑**，提升网络效率和公平性。


## TCP的突发发送缺陷
传统发送模式：==TCP发送方根据拥塞窗口（cwnd）和接收窗口（rwnd）计算可用窗口，一旦窗口允许，立即连续发送多个报文段（如一个cwnd内的所有数据）==。

### 负面影响
**网络拥塞**：突发流量可能超过中间路由器/交换机的缓冲容量，导致丢包或全局同步（多个流同时拥塞）。
AQM（Active Queue Management）机制失效：如RED（随机早期检测）等主动队列管理依赖平滑流量，突发会干扰其判断。
**资源竞争带来的不公平性**：突发性强的流容易抢占带宽，降低其他流的公平性。

### 小结：TCP 天生是"Micro-bursts"（微突发）的

经典 TCP 的发送逻辑是：
- 拥塞窗口 `cwnd`
- 只要 `inflight < cwnd`
- 就可以立即发送


这会导致：
- ACK 一回来；大量包瞬间进入队列

在高速网络（25G/100G）中尤其严重：
- NIC TX ring
- 交换机队列
- 对端接收缓冲


造成 **队列抖动、时延放大、丢包**


# TCP pacing的原理
## 核心思想
它的核心思想是：**将原来“爆发式”发送的数据包，在时间维度上平滑地均匀分布。**

即：将窗口内的数据包以均匀时间间隔发送，而非一次性突发。
类比：水管放水——拥塞控制决定水管口径（窗口大小），Pacing控制阀门开合频率（发送节奏），使水流平稳而非喷涌。

### 核心参数

#### Pacing Rate
Pacing Rate（ pacing_rate ）：计算单位时间内允许发送的数据量，通常基于当前拥塞窗口和RTT（往返时间）。
```bash
pacing_rate = cwnd / RTT。

例如：cwnd = 10 MSS，RTT = 100ms，则 pacing_rate = 10 MSS / 0.1s = 100 MSS/s。


增强调整：
保守策略：为避免过度激进，实际实现可能设置 pacing_rate = k * cwnd / RTT（k为略小于1的系数，如0.9）。
动态适应：当cwnd或RTT变化时（如收到新ACK），实时更新pacing_rate。

```

#### 间隔时间（ interval ）
间隔时间（ interval ）：每个数据包之间的发送延迟，例如 interval = MSS / pacing_rate。

### 范例
在一个 RTT 内：
```bash
允许发送数据量 = cwnd
发送时间跨度 = RTT

pacing_rate = cwnd / RTT
```

然后：

- 使用定时器 / 高精度时钟，控制包发送时间间隔，达到 均匀发包。

```bash
假设：
`cwnd = 120 KB`
`RTT = 10 ms`

pacing_rate = 12 MB / s

均匀发送：比如：
每 1 ms 发 12 KB
而不是 10 ms 一次性发 120 KB
```

## Pacing的触发时机和实现方式
### 触发时机
当有新数据需发送且窗口允许时，检查距离上次发送的时间是否达到`interval`。
若未达到，启动定时器延迟发送；若已达到，立即发送。

### 实现方式

在 Linux 内核中，Pacing 通常由 **FQ (Fair Queueing)** 调度器配合实现。内核不会在收到 ACK 后立即把所有数据塞给网卡，而是设置一个高精度的定时器，按需放行。


内核定时器：早期Linux使用高精度定时器（hrtimer）调度每个包的发包时间。
软中断优化：现代内核（如Linux的TCP Small Queue）将Pacing与队列管理结合，减少定时器开销。
硬件卸载：部分网卡支持时间戳调度，直接在硬件层面实现Pacing。

# TCP pacing 和拥塞控制的协同

|概念|作用|
|---|---|
|拥塞控制|决定 **能发多少（cwnd）**|
|TCP pacing|决定 **以多快速度发**|

它们是： **正交但协作的机制**

举个例子
```bash
- cwnd = 1MB
- RTT = 10ms

不 pacing：可能 1ms 内全发完
有 pacing：划分时间段来发送，整体来看的话，是按照100MB/s 均匀发送
```   

## 独立角色

拥塞控制决定“发多少”（窗口大小），Pacing决定“怎么发”（时间分布）。
例如在慢启动阶段，cwnd指数增长，Pacing会相应加快发送速率但保持均匀性。

## 相互作用

丢包响应：发生丢包时，拥塞控制减少cwnd，Pacing自动降低pacing_rate。
ACK驱动：每个ACK触发cwnd更新和pacing_rate重算，确保节奏与网络状态同步。


# TCP pacing的优缺点

|**特性**|**开启 Pacing 后**|**未开启 Pacing (Burst)**|
|---|---|---|
|**丢包率**|显著降低（减少了缓冲区溢出丢包）|较高（容易产生突发丢包）|
|**吞吐量**|更加稳定|在长肥网络（LFN）中波动大|
|**CPU 开销**|**较高**（需要大量高精度定时器）|较低|
|**应用场景**|视频流、实时通信、BBR 算法|短连接、低并发请求|


## 优点

## 缺点

# 实际应用
## socket 选项
### 套接字级别代码控制
```c
uint32_t rate = 1250000; // 设置速率为 1.25MB/s (10Mbps)
if (setsockopt(sockfd, SOL_TCP, TCP_MAX_PACING_RATE, &rate, sizeof(rate)) < 0) {
    perror("setsockopt TCP_MAX_PACING_RATE");
}
```

## TCP内核参数
### 系统级别开启（推荐配合 BBR）：切换调度器
```bash
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr
```

```bash
# sysctl -a |grep -i pacing
net.ipv4.tcp_pacing_ca_ratio = 120
net.ipv4.tcp_pacing_ss_ratio = 200
```

```bash
(1) tcp_pacing_ss_ratio - INTEGER

	sk->sk_pacing_rate is set by TCP stack using a ratio applied to current rate. (current_rate = cwnd * mss / srtt) If TCP is in slow start, tcp_pacing_ss_ratio is applied to let TCP probe for bigger speeds, assuming cwnd can be doubled every other RTT.
	
	Default: 200

(2) tcp_pacing_ca_ratio - INTEGER

	sk->sk_pacing_rate is set by TCP stack using a ratio applied to current rate. (current_rate = cwnd * mss / srtt) If TCP is in congestion avoidance phase, tcp_pacing_ca_ratio is applied to conservatively probe for bigger throughput.
	
	Default: 120
```


## BBR拥塞控制算法内置Pacing机制

Pacing 是 Google 提出的 **BBR (Bottleneck Bandwidth and Round-trip propagation time)** 拥塞控制算法的核心支柱。

- **BBR 放弃了丢包驱动**：传统的算法（如 Cubic）通过不断增加窗口直到丢包来探测带宽；而 BBR 则是主动测量带宽，并利用 Pacing 直接按测量到的带宽发包。
    
- **极低延迟**：因为 BBR 始终以刚好匹配瓶颈带宽的速率发包，它不会在交换机队列中积压数据包，从而将延迟（Bufferbloat）降到最低。

# 其他
## Pacing和限速
==Pacing不等于限速==。

- pacing 是 平滑发送
- 限速是 带宽 cap

## pacing 会降低吞吐吗？

通常不会，反而：
- 更容易跑满 BDP
- 减少重传

# 小结
TCP pacing 是在拥塞窗口允许的前提下，把 TCP 数据在 RTT 内均匀发送，从而**减少突发、降低排队尾时延、提升多流公平性**的==发送速率整形机制==。

TCP Pacing 改变了 TCP “快进快出”的本性，使其变得更加绅士。在现代高带宽、大延迟的网络环境下（尤其是涉及 DPDK 或高并发网关时），配合 **FQ 调度器** 开启 Pacing 是提升网络吞吐量和降低长尾延迟（Tail Latency）的关键手段。

TCP Pacing机制通过精细化控制发包节奏，弥补了传统拥塞控制仅关注窗口大小的不足。它像一位“交通调度员”，使数据流从“野蛮生长”变为“有序通行”，提升网络整体效率与公平性。理解Pacing有助于优化高带宽或长RTT网络（如卫星链路、CDN）的性能。

# 参考
```bash

```