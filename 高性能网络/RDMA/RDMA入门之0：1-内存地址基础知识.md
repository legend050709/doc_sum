```table-of-contents
```
# 硬件架构
# 地址概述
## 物流地址(PA)
## 虚拟地址(VA)
## 地址空间
## MMU
## IOMMU

### IOMMU和MMU
## MMIO
MMIO（Memory-Mapped I/O，内存映射I/O）。
MMIO是一种用于进行输入输出操作的技术。它通过将外围设备的寄存器映射到计算机的内存地址空间，使得CPU可以通过访问内存，使用和访问这些外设的寄存器。
在MMIO中，设备的控制和数据寄存器被分配一段连续的内存地址，并且可以通过读取和写入这些内存地址来与外部设备进行通信。

比如：CPU通过MMIO向RNIC发送消息来启动网络操作（这个过程即是Doorbell机制）。

### MMIO和IOMMU
MMIO是让CPU通过内存地址访问设备寄存器，进而给设备下发命令或者执行操作。
而IOMMU是让设备在访问内存时通过地址转换得到设备的物理地址来进行访问。
换句话说，**MMIO是CPU到设备的访问，而IOMMU是设备到内存的访问**。
它们可能在某些情况下一起使用，比如当一个设备需要通过MMIO被CPU访问，同时该设备执行DMA时又需要IOMMU来转换地址。

![](attachments/Pasted%20image%2020250319104327.png)



## 总线地址(bus address)
## IO虚拟地址(IOVA)
## DMA地址

# DMA
## 什么是DMA？

DMA （Direct Memory Access：直接内存访问）技术是一种不需要CPU参与，就能实现host外设比如pcie设备对host memory的访问。

有了DMA机制之后，外设跟主存之间的数据交互基本都是由dma controller完成，从而避免了cpu的参与，降低cpu使用率。

![](attachments/Pasted%20image%2020250315230136.png)

DMA就是我们在主板上放一块独立的芯片。在进行内存和 I/O 设备的数据传输的时候，我们不再通过 CPU 来控制数据传输，而直接通过 DMA 控制器（DMA Controller，简称 DMAC）。这块芯片，我们可以认为它其实就是一个协处理器（Co-Processor）。

## DMA有什么用？

DMAC最有价值的地方体现在，当我们**要传输的数据特别大、速度特别快**；或者**传输的数据特别小、速度特别慢**的时候。

例如，我们用千兆网卡或者硬盘传输大量数据的时候，如果都用 CPU 来搬运的话，肯定忙不过来，所以可以选择 DMAC。而当数据传输很慢的时候，DMAC 可以等数据到齐了，再发送信号，给到 CPU 去处理，而不是让 CPU 在那里忙等待。

在 DMAC 控制数据传输的过程中，我们还是需要 CPU 的进行控制，但是具体数据的拷贝不再由 CPU 来完成。

### 没有DMA的情况
原本，计算机所有组件之间的数据拷贝（流动）必须经过 CPU，如下图所示：

![](attachments/Pasted%20image%2020250313204956.png)

### 存在DMA的情况
DMA 代替 CPU 负责内存与磁盘以及内存与网卡之间的数据搬运，CPU 作为 DMA 的控制者，如下图所示：
![](attachments/Pasted%20image%2020250313205014.png)

## DMA的实现简述

在实现DMA传输时，是由DMA控制器直接掌管总线，因此，存在着一个总线控制权转移问题。即DMA传输前，CPU要把总线控制权交给DMA控制器，而在结束DMA传输后，DMA控制器应立即把总线控制权再交回给CPU。
CPU只是在每个数据块传输的开始和结束实施控制，在数据块的传输过程中则由DMA控制器控制。

DMA控制方式传送数据的过程可以分为三个阶段：预处理、数据传送、传送后处理。
### 预处理阶段

DMA控制器的命令/状态寄存器（CR）接收到CPU发出的I/O命令，将内存起始地址（输入数据到内存）或内存源地址（内存输出数据）写入DMA控制器的内存地址寄存器（MAR），同时将要传送的字（字节）数存入DMA控制器的字计数器（DC）。随后将外设中的源地址（或目标地址）直接送入 DMA控制器的I/O控制逻辑，然后启动DMA控制器进行数据传送。


### 数据传送阶段

CPU将总线控制权让给DMA控制器，并且在数据传送期间不再使用总线。DMA控制器按照内存地址寄存器（MAR）的指示，不断在设备与内存之间进行数据传输，并随时修改内存地址寄存器（MAR）和字计数器（DC）的值。

当一个数据块传输完毕或数据计数器的值减少到0时（所有数据都已传输完毕），传输停止并且向CPU发出中断信号。

### 传送后处理阶段

CPU响应DMA控制器的中断请求。如果数据传送完成，则转向相应的中断处理程序进行后续处理；如果还有数据需要传送，则按照相同方法重新启动剩余数据的传送。

在DMA控制方式中，CPU只是在每个数据块传输的开始和结束实施控制，在数据块的传输过程中则由DMA控制器控制。


##  DMA的scatter/gather(SG-DMA)

Scatter：离散
Gather：聚合

Linux 在内核 2.4 版本里引入了 DMA 的 scatter/gather – 分散/收集功能，只要确保 Linux 版本高于 2.4 即可。

### 场景1：Scatter Transfer
**场景1：将一片连续内存数据搬运到一片不连续的的内存空间（且间隔是相等的）**
源内存：连续
目的内存：离散
这个时候就可以用到`DMA`的 `Scatter Transfer`

![](attachments/Pasted%20image%2020250313202630.png)


### 场景2：Gather transfer
**场景2：将一片内存区域中等间隔的多段数据拷贝到一段连续内存中。（常见的2D矩形抠图就是这种场景的典型应用）**

源内存：分散
目的内存：聚合
这种场景就可以使用DMA的`Gather transfer`

![](attachments/Pasted%20image%2020250313202822.png)


### scatter/gather DMA 与 block DMA

在DMA传输数据的过程中，要求源物理地址和目标物理地址必须是连续的。但是在某些计算机体系中，如IA架构，连续的存储器地址在物理上不一定是连续的（CPU以虚拟地址寻址），所以DMA传输要分成多次完成。

#### block DMA
DMA控制器在传输完一块物理上连续的数据后引起一次中断，然后再由CPU触发进行下一块物理存储空间上的数据传输，这种方式被称为 Block DMA。

传统的block DMA像这样：
![](attachments/Pasted%20image%2020250313224146.png)

#### scatter/gather DMA
**Scatter-gather DMA(分散聚集式直接内存访问)** 方式则不同，将物理上分散的多个不连续存储空间以链表（struct scatterlist，链表或数组实现）形式组织起来，然后把链表首地址告诉`DMA master`。`DMA master`在传输完一块物理连续的数据后，不用发起中断，而是根据链表来传输下一块物理上连续的数据，直到传输完毕后再发起一次中断。

先进的scatter-gather DMA像这样：
![](attachments/Pasted%20image%2020250313224150.png)

即：`scatter-gather DMA`允许一次传输多个物理上不连续的块，完成传输后只发起一次中断。这样做的好处是大大减少了中断的次数，提高了数据传输的效率。



### scatter/gather DMA的应用

![](attachments/Pasted%20image%2020250313224208.png)
```c
.txmode = { 
    ...                                                  
    .offloads = DEV_TX_OFFLOAD_MULTI_SEGS,
    ...
}
```

Scatter-Gather DMA允许网卡从多个不连续的内存区域收集数据，合并成一个数据包发送出去；或者反过来，接收的数据分散到不同内存位置。在发送数据包时，如果数据分散在多个缓冲区，SG-DMA能够有效地处理这些分散的数据，提升性能。

> dpdk在ip分片的实现中，采用了一种称作零拷贝的技术。而这种实现方式的底层，正是由`scatter/gather DMA`支撑的。dpdk的多个分片包采用了链式管理，同一个数据包的多个分片，分散存储在不连续的块中（mbuf结构）。这就要求DMA一次操作，需要从不连续的多个块中搬移数据到网卡，然后发送出去。附上e1000驱动发包部分代码：
```c
uint16_t
eth_em_xmit_pkts(void *tx_queue, struct rte_mbuf **tx_pkts,
        uint16_t nb_pkts)
{
	...
	txq = tx_queue;
	sw_ring = txq->sw_ring;
	txr = txq->tx_ring;
	tx_id = txq->tx_tail;
	txe = &sw_ring[tx_id];
	...
    //e1000驱动部分代码
    for (nb_tx = 0; nb_tx < nb_pkts; nb_tx++) {
	    ...
	    tx_pkt = *tx_pkts++;
	    ...
	    m_seg = tx_pkt;
	    do {
	        txd = &txr[tx_id];
	        txn = &sw_ring[txe->next_id];
	 
	        if (txe->mbuf != NULL)
	            rte_pktmbuf_free_seg(txe->mbuf);
	        txe->mbuf = m_seg;
	 
	        /*
	        * DMA 映射：将数据包的物理内存地址映射到网卡的 DMA 引擎可访问的地址空间（IOVA或DMA地址）。
	        * Set up Transmit Data Descriptor.
	        * 需要配置描述符（Descriptor），
	        * 这些描述符告诉DMA引擎每个内存块的位置和长度。
	        */
	        slen = m_seg->data_len;
	        buf_dma_addr = rte_mbuf_data_iova(m_seg); //获取dma地址(iova地址)
	 
	        txd->buffer_addr = rte_cpu_to_le_64(buf_dma_addr);
	        txd->lower.data = rte_cpu_to_le_32(cmd_type_len | slen);
	        txd->upper.data = rte_cpu_to_le_32(popts_spec);
	 
	        txe->last_id = tx_last;
	        tx_id = txe->next_id;
	        txe = txn;
	        m_seg = m_seg->next;
	    } while (m_seg != NULL);
	 
	    /*
	    * The last packet data descriptor needs End Of Packet (EOP)
	    */
	    cmd_type_len |= E1000_TXD_CMD_EOP;
	    txq->nb_tx_used = (uint16_t)(txq->nb_tx_used + nb_used);
	    txq->nb_tx_free = (uint16_t)(txq->nb_tx_free - nb_used);
	    ...
    }
}
```

#### NIC是否支持SG-DMA

```bash
ethtool -k enp6s0
Features for enp6s0:
rx-checksumming: off [fixed]
tx-checksumming: off
        tx-checksum-ipv4: off [fixed]
        tx-checksum-ip-generic: off [fixed]
        tx-checksum-ipv6: off [fixed]
        tx-checksum-fcoe-crc: off [fixed]
        tx-checksum-sctp: off [fixed]
scatter-gather: on
        tx-scatter-gather: on
        tx-scatter-gather-fraglist: off [fixed]
tcp-segmentation-offload: off
        tx-tcp-segmentation: off [fixed]
        tx-tcp-ecn-segmentation: off [fixed]
        tx-tcp-mangleid-segmentation: off [fixed]
        tx-tcp6-segmentation: off [fixed]
udp-fragmentation-offload: off
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
rx-vlan-offload: off [fixed]
tx-vlan-offload: off [fixed]
ntuple-filters: off [fixed]
receive-hashing: off [fixed]
highdma: on
rx-vlan-filter: on [fixed]
vlan-challenged: off [fixed]
tx-lockless: off [fixed]
netns-local: off [fixed]
tx-gso-robust: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-gso-partial: off [fixed]
tx-sctp-segmentation: off [fixed]
tx-esp-segmentation: off [fixed]
fcoe-mtu: off [fixed]
tx-nocache-copy: off
loopback: off [fixed]
rx-fcs: off [fixed]
rx-all: off [fixed]
tx-vlan-stag-hw-insert: off [fixed]
rx-vlan-stag-hw-parse: off [fixed]
rx-vlan-stag-filter: off [fixed]
l2-fwd-offload: off [fixed]
hw-tc-offload: off [fixed]
esp-hw-offload: off [fixed]
esp-tx-csum-hw-offload: off [fixed]
rx-udp_tunnel-port-offload: off [fixed]
```

# 参考
```bash
# RDMA之内存地址基础知识
https://zhuanlan.zhihu.com/p/463199854

# RDMA为什么要Pin内存？
https://mp.weixin.qq.com/s/6ESmh8RKAAdW5nm3J2a9VQ
```