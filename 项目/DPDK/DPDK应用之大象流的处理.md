```table-of-contents
```
# 背景

# 大象流（Elephant flow）


# 网络测量sketch

# DPDK SW Eventdev
通过 DPDK 中提供的软件库 `Eventdev` 来解决大象流的问题。

# 硬件 Eventdev

# HSDLB 使用 intel DLB 解决大象流的问题

## 背景
现代 DPDK 程序都会基于 RSS 收包多核扩展技术来将不同的 IP 5-tuple Traffic 映射到特定的 Core 上进行处理。

但当出现大象流时，10% 的大象流就占据了总流量的 90%，继而造成某些 Core 忙死，而另外一些 Core 则闲死的情况。更甚者，即便忙死了某个 Core 也依旧无法满足大象流的处理需求，而导致丢包。

![](attachments/Pasted%20image%2020250123123607.png)

## 思路

为了解决大象流问题，HDSLB 基于以下思路进行了 3 方面的优化：

1. **大象流识别**：首先，要识别出大象流（Heavy）和老鼠流（Light）。
2. **大象流拆分**：然后，将大象流能够拆分并映射到多个 Cores 上并行处理，而不仅仅映射到一个 Core 上。
3. **大象流重排**：最终，还需要将拆分到多个 Cores 上并行处理的流量再进行合法性排序。

![](attachments/Pasted%20image%2020250123123634.png)

从上述原理图可以看出，这里面的关键技术是由 Intel CPU 硬件提供的 DLB（Dynamic Load Balancer）特性。基于 DLB 可以实现：

1. 收包时：将大象流切分到多个 Cores 中进行处理。
2. 发包时：将多个 Cores 上的流量进行汇聚并合法化排序。

![](attachments/Pasted%20image%2020250123123702.png)

### 详细介绍

具体而言，HDSLB 的大象流处理方案需要在 Main Core 上实现一个基于 Intel NIC FDIR 硬件特性的 Switch Filter，用于完成大象流和老鼠流的识别、标记并分类映射到不同的 Core，通过硬件的方式减少了软件上的匹配和查表，性能更高；而在 Worker Cores 上还需要实现基于 Intel CPU DLB 硬件特性的大象流拆分。

如下图所示：

**（1）Main Core（master core）**:

![](attachments/Pasted%20image%2020250123123916.png)


**（2）worker core **:
![](attachments/Pasted%20image%2020250123123933.png)


## intel DLB介绍
英特尔动态负载均衡加速器 (Intel Dynamic Load Balancer，英特尔 DLB) 是一个**硬件队列管理器和负载平衡器**，通过数据平面开发套件 (DPDK) 的 **Eventdev** 抽象设备，从软件中卸载队列和调度任务。

第四代英特尔至强可扩展处理器（intel xeon CPU）中引入了英特尔 DLB 技术，可有效地解决高并发软件架构遇到的性能挑战。==英特尔DLB 是集成在 CPU 内部的硬件队列管理器==，软件通过入队、 出队的方式与英特尔 DLB 进行交互。其中，入队方称为生产者，出 队方称为消费者。

## intel DLB的特性
英特尔 DLB 有两个主要的特点，即动态与负载均衡。

负载均衡要解决的是因为待处理数据在处理器核心之间分发不均匀，导致的处理器核心负载不均衡的问题。
与一些软件方案所使用的静态调度算法不同，英特尔DLB 在分发待处理数据的过程中，能够根据每个处理器核心的负载情况，动态地选出最合适的核心，并将数据分发给其进行处理。
### 四种队列模型

为了实现动态特性，英特尔 DLB 设计了四种队列模型，来应对不同应用场景的需求:

- Direct Queue:适用于多个生产者但只有一个消费者的场景，无负载均衡;
- Unorder Queue: 适用于多个生产者以及多个消费者的场景，不关心任务的先后顺序，将每个任务调度给当前负载最低的处理器核心去处理;
- OrderQueue:适用于多个生产者及多个消费者的场景，关心任务的先后顺序;当多个任务被多个处理器核心处理完时，需要按照原始顺序重新排列;
- Atomic Queue:适用于多个生产者以及多个消费者的场景，任务按照一定的规则进行分组;处理这些任务时使用同一组资源，关心同一分组内的任务先后顺序。

![](attachments/Pasted%20image%2020250123124237.png)




## intel DLB的应用
英特尔DLB技术在多个服务器CPU核心之间高效分配网络流量，使得负载均衡器、网关和内容分发网络（CDN）等应用在运行时能够减少时延，处理大核无法处理大象流的丢包问题，提高流量管理的精确性。
其应用场景包括软件定义广域网（SD-WAN）、流量监控、速率限制以及IPsec或传输层安全（TLS）网关等。

## 基于英特尔® DLB 技术的无锁限速方案阐述

### 需求
现有的优化限速方案性能的方法，集中于降低“锁”的开销， 也因此引入了精度问题。另外一种思路是使用无锁的限速方案，这种方案通过给网卡下发特定规则或是在软件中按照预定的算法，将同一条流的网络报文调度到同一个处理器核心，通过在同一个处理器核心上 访问同一个令牌桶，实现无锁的限速方案。这些方案的问题在于报文的调度规则是静态的，无法根据处理器核心的负载情况做出动态调整，极易因网络突发流量导致部分处理器核心过载，进而产生丢包的情况。

### 思路
是否存在一种方法，可以在多核处理器中，既能去掉保护全局令牌桶的“锁”，又能保证多核的负载均衡?
利用英特尔® DLB 的 Atomic Queue 特性，即可以在多核心的场景下实现无锁限速方案。将待处理的网络报文按照其所属的限速网络数据流进行分组， 英特尔® DLB 的 Atomic Queue 能够把属于同一分组的报文调度到同一个处理器核心进行处理;
另外，Atomic Queue 还会为每一条流动态地选择处理器核心，当有多条网络数据流时，流量能够较为均匀地分散到各个处理器核心，确保处理器中多个核心的负载均衡。

在无锁限速方案中，处理器核心被分成了两组，从队列操作的角度，分别被称为生产者和消费者。
生产者为每个报文生成 Atomic Queue 所需的 Flow ID，随后将报文入队到 DLB 的 Atomic Queue中。
DLB 在消费者线程间分发消息，同时保证原子性。消费者从Atomic Queue 获取报文之后，以无锁的方式安全地访问 Flow ID 对 应的全局令牌桶，完成限速相关操作。

在无锁限速方案中，由于只使用了全局令牌桶，因此不存在低速率时本地令牌桶预留令牌导致的限速后速率偏低，以及预取令牌导致 的限速后速率偏高的精度问题。

![](attachments/Pasted%20image%2020250123124612.png)

# 参考
```bash

# 动态负载均衡与IPsec 工作负载调整
https://www.intel.cn/content/www/cn/zh/customer-spotlight/cases/dynamic-load-balancing-scaling-of-ipsec-workloads.html

# Intel HDSLB 高性能四层负载均衡器 — 代码剖析和高级特性
https://juejin.cn/post/7380579037113204786

# 大象流的危害以及处理
https://blog.csdn.net/legend050709/article/details/126286710

https://www.zhihu.com/question/50171430/answer/371316260 （大象流，老鼠流定义）
（大象流，老鼠流如何识别/ 大象流影响老鼠流???）
https://www.sdnlab.com/22685.html (网络测量，sketch)
https://www.sdnlab.com/23514.html （网络测量，mv-sketch）
源码：https://github.com/barrust/count-min-sketch  （count-min sketch）
```