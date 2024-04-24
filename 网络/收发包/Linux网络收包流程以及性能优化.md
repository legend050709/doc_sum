```table-of-contents
```
# 收包流程

## 整体流程

![](attachments/Pasted%20image%2020230805164733.png)

![](attachments/Pasted%20image%2020240513155251.png)

```bash
1. Packets arrive at the NIC
2. NIC will verify `MAC` (if not on promiscuous mode) and `FCS` and decide to drop or to continue
3. NIC will [DMA packets at RAM](https://en.wikipedia.org/wiki/Direct_memory_access), in a region previously prepared (mapped) by the driver
4. NIC will enqueue references to the packets at receive [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer) queue `rx` until `rx-usecs` timeout or `rx-frames`
5. NIC will raise a `hard IRQ`
6. CPU will run the `IRQ handler` that runs the driver's code
7. Driver will `schedule a NAPI`, clear the `hard IRQ` and return
8. Driver raise a `soft IRQ (NET_RX_SOFTIRQ)`
9. NAPI will poll data from the receive ring buffer until `netdev_budget_usecs` timeout or `netdev_budget` and `dev_weight` packets
10. Linux will also allocate memory to `sk_buff`
11. Linux fills the metadata: protocol, interface, setmacheader, removes ethernet
12. Linux will pass the skb to the kernel stack (`netif_receive_skb`)
13. It will set the network header, clone `skb` to taps (i.e. tcpdump) and pass it to tc ingress
14. Packets are handled to a qdisc sized `netdev_max_backlog` with its algorithm defined by `default_qdisc`
15. It calls `ip_rcv` and packets are handed to IP
16. It calls netfilter (`PREROUTING`)
17. It looks at the routing table, if forwarding or local
18. If it's local it calls netfilter (`LOCAL_IN`)
19. It calls the L4 protocol (for instance `tcp_v4_rcv`)
20. It finds the right socket
21. It goes to the tcp finite state machine
22. Enqueue the packet to the receive buffer and sized as `tcp_rmem` rules
    1. If `tcp_moderate_rcvbuf` is enabled kernel will auto-tune the receive buffer
23. Kernel will signalize that there is data available to apps (epoll or any polling system)
24. Application wakes up and reads the data
```


1.⽹卡收到⽹线上的packet，⾸先检查packet的CRC校验，保证完整性，然后将packet头去掉，得到frame。（⽹卡会检查MAC包内的⽬的MAC地址是否和本⽹卡的MAC地址⼀样，不⼀样则会丢弃。） 
2.⽹卡将frame拷贝到预分配的ring buffer缓冲。 
3.⽹卡驱动程序通知内核处理，经过TCP/IP协议栈层层解码处理。 
4.应⽤程序从socket buffer 中读取数据。

![](attachments/Pasted%20image%2020230805170337.png)
接收数据包是一个复杂的过程，涉及很多底层的技术细节，但大致需要以下几个步骤：

1. 网卡收到数据包。
2. 将数据包从网卡**硬件缓存**转移到服务器内存中。
3. 通知内核处理。
4. 经过TCP/IP协议逐层处理。
5. 应用程序通过read()从socket buffer读取数据。

## 具体流程

![](attachments/Pasted%20image%2020230805213734.png)

![](attachments/Pasted%20image%2020230805171011.png)

![](attachments/Pasted%20image%2020230806152104.png)


物理网卡收到数据包的处理流程如上图所示，详细步骤如下：

1. 网卡收到数据包，先将高低电平转换到网卡fifo存储，网卡申请ring buffer的描述符，根据描述符找到具体的物理地址，从fifo队列物理网卡会使用DMA将数据包写到了该物理地址,，其实就是skb_buffer中.
2. 这个时候数据包已经被转移到skb_buffer中，因为是DMA写入，内核并没有监控数据包写入情况，这时候NIC触发一个硬中断，每一个硬件中断会对应一个中断号，且指定一个vCPU来处理.
3. 硬件中断的中断处理程序，调用驱动程序完成，a.启动软中断
4. 硬中断触发的驱动程序会禁用网卡硬中断，其实这时候意思是告诉NIC，再来数据不用触发硬中断了，把数据DMA拷入系统内存即可
5. 硬中断触发的驱动程序会启动软中断，启用软中断目的是将数据包后续处理流程交给软中断慢慢处理，这个时候退出硬件中断了，但是注意和网络有关的硬中断，要等到后续开启硬中断后，才有机会再次被触发
6. NAPI触发软中断（signal  NET_RX_SOFTIRQ to ksoftirqd 内核线程），触发napi系统
7. 内核协议栈从消耗ring buffer指向的skb_buffer
8. NAPI循环处理ring buffer数据，处理完成
9. 启动网络硬件中断，有数据来时候就可以继续触发硬件中断，继续通知CPU来消耗数据包.

其实上述过程过程简单描述为：
>网卡收到数据包，DMA到内核内存，中断通知内核数据有了，内核按轮次处理消耗数据包，一轮处理完成后，开启硬中断。

其核心就是：
>网卡和内核其实是生产和消费模型，网卡生产，内核负责消费，中间的媒介就是ring buffer，生产者需要通知消费者消费；如果生产过快会产生丢包，如果消费过慢也会产生问题。也就说在高流量压力情况下，只有生产消费优化后，消费能力够快，此生产消费关系才可以正常维持，所以如果物理接口有丢包计数时候，未必是网卡存在问题，也可能是内核消费的太慢。

### 网卡接收数据包

信息在链路层是以二进制传递的，也就是说网卡收到的数据包里只有一个个的高低电平所代表的二进制位。网卡将这些二进制位转换到网卡FIFO存储，然后再拷贝到系统主存（内存）中。其中涉及到网卡控制器，CPU，DMA，驱动程序，在OSI模型中属于物理层和链路层，如下图所示：
![](attachments/Pasted%20image%2020230805170719.png)

网卡工作在物理层和数据链路层，主要由PHY/MAC芯片、Tx/Rx FIFO、DMA等组成，其中网线通过变压器接PHY芯片、PHY芯片通过MII接MAC芯片、MAC芯片接PCI总线。
其中：
- PHY芯片主要负责：CSMA/CD、模数转换、编解码、串并转换
- MAC芯片主要负责：比特流和帧的转换：7字节的前导码Preamble和1字节的帧首定界符SFD CRC校验
- Packet Filtering：L2 Filtering、VLAN Filtering、Manageability / Host Filtering


其他：
网线上的packet首先被网卡获取，网卡会检查packet的CRC校验，保证完整性，然后将packet头去掉，得到frame。网卡会检查MAC包内的目的MAC地址，如果和本网卡的MAC地址不一样则丢弃(混杂模式除外)。

- 网卡的缓冲区
网卡中的缓冲区既不属于内核空间，也不属于用户空间。它属于硬件缓冲，允许网卡与操作系统之间有个缓冲；
>注意：通常需要快速的拷贝网络数据包到系统内存，因为网卡上接收网络数据包的缓存大小固定，而且相比系统内存也要小得多。所以上述拷贝动作一旦被延迟，必然造成网卡 FIFO 缓存溢出 - 进入的数据包占满了网卡的缓存，后续的包只能被丢弃，这也应该就是 ifconfig 里的 overrun 的来源。

- 网卡的缓冲区满的理解：
如果将rx_fifo error就理解为硬件的队列满了。我理解分为两种情况：
1>情况一：程序性能或者内核的软中断处理的比较慢，导致ring  buffer满，ring  buffer 满了之后，dma无法获取到可用的描述符（DMA从寄存器中读取描述符，驱动通过doorbell通知DMA读取），那么也会导致网卡的硬件队列堆积，进而满了，导致有丢包。
2>情况二：pcie性能不足，比如带宽宽度，速率不足，或者mps，mrrs不够，导致dma通过pcie填充报文性能不足，进而导致硬件队列满，然后丢包。
3> 理解三：ethtool查看队列大小以及队列个数，我理解可能就是ring  buffer的大小以及个数。

- 网卡内核缓冲区是否有多队列
我理解应该是没有的，可能就是一个大的缓冲区。然后这个缓冲区可以对应多个ring-buffer.
比如：通过RSS hash，将指定的包hash到指定的 ring-buffer中。
如果网卡的硬件缓冲区还区分多个队列，那么不灵活，第二个也不容易更改硬件的多队列。


### 数据包从网卡到内核内存
![](attachments/Pasted%20image%2020230806152115.png)

NIC在接收到数据包之后，首先需要将数据同步到内核中，这中间的桥梁是rx ring buffer。它是由NIC和驱动程序共享的一片区域，事实上，rx ring buffer存储的并不是实际的packet数据，而是一个描述符，这个描述符指向了它真正的存储地址（物理地址供网卡的DMA使用，虚拟地址供内核协议栈的ksoftirqd使用，以及空间的长度）。

![](attachments/Pasted%20image%2020230805171825.png)
具体流程如下：
1. 驱动在内存中分配一片缓冲区用来接收数据包，叫做sk_buffer缓冲区;
2. 将上述缓冲区的地址和大小（即接收描述符），加入到rx ring buffer。描述符中的缓冲区地址是DMA使用的物理地址;
3. 驱动通知网卡有一个新的描述符;
4. 网卡从rx ring buffer中取出描述符，从而获知缓冲区的地址和大小;
5. 网卡收到新的数据包;
6. 网卡将新数据包通过DMA直接写到sk_buffer中。

当驱动处理速度跟不上网卡收包速度时，驱动来不及分配缓冲区，NIC接收到的数据包无法及时写到sk_buffer，就会产生堆积，当**NIC内部缓冲区**写满后，就会丢弃部分数据，引起丢包。这部分丢包为rx_fifo_errors，在 /proc/net/dev中体现为fifo字段增长，在ifconfig中体现为overruns指标增长。

### 协议栈处理数据包
这个时候，数据包已经被转移到了sk_buffer中。前文提到，这是驱动程序在内存中分配的一片缓冲区，并且是通过DMA写入的，这种方式不依赖CPU直接将数据写到了内存中，意味着对内核来说，其实并不知道已经有新数据到了内存中。那么如何让内核知道有新数据进来了呢？答案就是中断，通过中断告诉内核有新数据进来了，并需要进行后续处理。

提到中断，就涉及到硬中断和软中断，首先需要简单了解一下它们的区别：

- 硬中断： 由硬件自己生成，具有随机性，硬中断被CPU接收后，触发执行中断处理程序。中断处理程序只会处理关键性的、短时间内可以处理完的工作，剩余耗时较长工作，会放到中断之后，由软中断来完成。硬中断也被称为上半部分。
- 软中断： 由硬中断对应的中断处理程序生成（**和硬中断在同一个CPU core**），往往是预先在代码里实现好的，不具有随机性。（除此之外，也有应用程序触发的软中断，与本文讨论的网卡收包无关。）也被称为下半部分。


当NIC把数据包通过DMA复制到内核缓冲区sk_buffer后，NIC立即发起一个硬件中断。CPU接收后，首先进入上半部分，网卡中断对应的中断处理程序是网卡驱动程序的一部分，之后由它发起软中断，进入下半部分，开始消费sk_buffer中的数据，交给内核协议栈处理。
![](attachments/Pasted%20image%2020230805172130.png)

通过中断，能够快速及时地响应网卡数据请求，但如果数据量大，那么会产生大量中断请求，CPU大部分时间都忙于处理中断，效率很低。
>为了解决这个问题，现在的内核及驱动都采用一种叫NAPI（new API）的方式进行数据处理，其原理可以简单理解为 中断+轮询，在数据量大时，一次中断后通过轮询接收一定数量包再返回，避免产生多次中断。


- 为什么分为硬中断和软中断？
内核的**==软中断系统==**是一种**在硬中断处理上下文（驱动中）之外执行代码**的机制。 **硬中断处理函数（handler）执行时，会屏蔽部分或全部（新的）硬中断**。中断被屏蔽的时间 越长，丢失事件的可能性也就越大。所以，**==所有耗时的操作都应该从硬中断处理逻辑中剥 离出来==**，硬中断因此能尽可能快地执行，然后再重新打开硬中断。
软中断相对于，硬中断将耗时操作转移出去，到了软中断。

- 软中断和硬中断的CPU相同么？
驱动的硬中断处理函数做的事情很少，但软中断将会在和硬中断相同的 CPU 上执行。**这就 是为什么给每个 CPU 一个特定的硬中断非常重要：这个 CPU 不仅处理这个硬中断，而且通 过 NAPI 处理接下来的软中断来收包**。


- 中断合并（Interrupt coalescing）
中断合并会将多个硬件中断事件放到一起，累积到一定阈值后才向 CPU 发起中断请求。
这可以防止**中断风暴**，提升吞吐，降低 CPU 使用量，但延迟也变大；中断数量过多则相反。
```c
$ sudo ethtool -c eth0
Coalesce parameters for eth0:
Adaptive RX: off  TX: off
stats-block-usecs: 0
sample-interval: 0
pkt-rate-low: 0
pkt-rate-high: 0
...

```

#### 内核线程
内核协议栈的运行，是按照一个内核线程的方式吗？在内核中，又是如何执行网络协议栈的呢？
在内核协议栈中处理数据包，需要消耗CPU资源。那么什么东西是资源分配（CPU、内存）的基本单元呢？答案是线程。网络收发的软中断处理，就有专门的内核线程 ksoftirqd。每个 CPU 都会绑定一个 ksoftirqd 内核线程，比如， 2 个 CPU 时，就会有 ksoftirqd/0 和 ksoftirqd/1 这两个内核线程。

不过要注意，并非所有网络功能，都在软中断内核线程中处理。内核中还有很多其他机制（比如硬中断、kworker、slab 等），这些机制一起协同工作，才能保证整个网络协议栈的正常运行。

>注：**==可以把软中断系统想象成一系列内核线程==**（每个 CPU 一个），这些线程执行针对不同 事件注册的处理函数（handler）。如果用过 `top` 命令，可能会注意到 `ksoftirqd/0` 这个内核线程，其表示这个软中断线程跑在 CPU 0 上。内核子系统（比如网络）能通过 `open_softirq()` 注册软中断处理函数。


- 软中断的监控统计
```c
$ cat /proc/softirqs
                    CPU0       CPU1       CPU2       CPU3
          HI:          0          0          0          0
       TIMER: 2831512516 1337085411 1103326083 1423923272
      NET_TX:   15774435     779806     733217     749512
      NET_RX: 1671622615 1257853535 2088429526 2674732223
       BLOCK: 1800253852    1466177    1791366     634534
BLOCK_IOPOLL:          0          0          0          0
     TASKLET:         25          0          0          0
       SCHED: 2642378225 1711756029  629040543  682215771
     HRTIMER:    2547911    2046898    1558136    1521176
         RCU: 2056528783 4231862865 3545088730  844379888
```
`NET_RX` 一行显示的是软中断在 CPU 间的分布。如果分布非常不均匀，那某一列的 值就会远大于其他列，这预示着下面要介绍的 Receive Packet Steering / Receive Flow Steering 可能会派上用场。但也要注意：不要太相信这个数值，`NET_RX` 太高并不一定都 是网卡触发的，下面会看到其他地方也有可能触发之。


#### 内核收包的函数处理流程
![](attachments/Pasted%20image%2020230805172257.png)

```c
bpftrace -e 'kprobe:netif_receive_skb_internal {printf("%s\n",kstack);}'        netif_receive_skb_internal+1  
        napi_gro_receive+186        ///End of pulling data out of rx ring buffer  
        igb_poll+1153  
        net_rx_action+329  
        __do_softirq+222  
        irq_exit+186  
        do_IRQ+127  
        ret_from_intr+0  
        cpuidle_enter_state+182  
        do_idle+552  
        cpu_startup_entry+111  
        start_secondary+420  
        secondary_startup_64+164
```

RPS 是早期单队列网卡上将软中断负载均衡到多个`CPU Core`的技术，它对数据流进行 hash 并分配到对应的`CPU Core`上，发挥多核的性能，即RPS+RFS（一个基于packet，一个基于Flow）是RSS的软件实现。不过现在基本都是多队列网卡，不会开启这个机制。

## 缓冲区
Linux 网络的收发流程。这个流程涉及到了多个队列和缓冲区，包括：
- 网卡收发网络包时，通过 DMA 方式交互的环形缓冲区；
- 网卡中断处理程序为网络帧分配的，内核数据结构 sk_buff 缓冲区；
- 应用程序通过套接字接口，与网络协议栈交互时的套接字缓冲区。

### ring buffer 环形缓冲区
NIC 在接收到数据包之后，首先需要将数据同步到内核中，这中间的桥梁是 `rx ring buffer`。它是由 NIC 和驱动程序共享的一片区域，事实上，`rx ring buffer` 存储的并不是实际的 packet 数据，而是一个描述符，这个描述符指向了它真正的存储地址（描述符包含：物理地址供DMA使用，虚拟地址供内核使用，以及包长度）。

收包时：
1》DMA作为生产者
2》内核协议栈(ksoftirqd)作为消费者
3》ring buffer作为DMA和内核协议栈的媒介。
注：每个CPU core对应一个接收包的ring buffer和发送包的 ring buffer。

```c
Again, the ring buffer is basically the fixed-size circular FIFO queue ( which contains descriptor or simply pointer to a memory address ).

Then if there’s a queue with a fixed-size, there will be a exceeded queue length situation.

When you send / receive too many data than you can process, the queue is full and there will be packet dropped. Luckily we can check it by using ethtool, like this:

#ethtool -S enp8s0f0 | grep drop  
	dropped_smbus: 0  
	tx_dropped: 0  
	rx_queue_0_drops: 0  
	rx_queue_1_drops: 0  
	rx_queue_2_drops: 0  
	rx_queue_3_drops: 0  
	rx_queue_4_drops: 0  
	rx_queue_5_drops: 0  
	rx_queue_6_drops: 0  
	rx_queue_7_drops: 0
```

#### 查看网卡是否支持多队列
判断当前系统环境是否支持多队列网卡，执行命令:
```c
lspci -s PCI-ID -vv
```
如果在Ethernet项中。含有MSI-X: Enable+ Count=9 Masked-语句，则说明当前系统环境是支持多队列网卡的，否则不支持。
![](attachments/Pasted%20image%2020230805221738.png)

通过加载网卡驱动，获取网卡型号和网卡硬件的队列数；但是在初始化 misx vector 的时候，还会结合系统在线 CPU 的数量，通过 Sum = Min(网卡队列，CPU Core) 来激活相应的网卡队列数量，并申请 Sum 个中断号。
比如：我们线上的 CPU 一般是 48 个逻辑 core，就会生成 48 个中断号，如果是两块网卡做了 bond，也就会生成 96 个中断号。

#### 设置网卡队列个数
注：其实就是设置对应的ring buffer的个数
```c
# 查看
# ethtool -l eth03
Channel parameters for eth03:
Pre-set maximums:
RX:		0
TX:		0
Other:		1
Combined:	63
Current hardware settings:
RX:		0
TX:		0
Other:		1
Combined:	32

# 设置
# ethtool -L eth03 combined 16
# ethtool -l eth03
Channel parameters for eth03:
Pre-set maximums:
RX:		0
TX:		0
Other:		1
Combined:	63
Current hardware settings:
RX:		0
TX:		0
Other:		1
Combined:	16

# 确认
ls /sys/class/net/eth03/queues/
```
![](attachments/Pasted%20image%2020230805221829.png)
![](attachments/Pasted%20image%2020230805222033.png)
#### 设置网卡队列大小
注：其实就是设置每个ring buffer的大小。
```c
# 查看eth0网卡Ring Buffer最大值和当前设置
$ ethtool -g eth0
Ring parameters for eth0:
Pre-set maximums:
RX:     4096   
RX Mini:    0
RX Jumbo:   0
TX:     4096   
Current hardware settings:
RX:     1024   
RX Mini:    0
RX Jumbo:   0
TX:     1024   

# 修改网卡eth0接收与发送硬件缓存区大小
$ ethtool -G eth0 rx 4096 tx 4096
Pre-set maximums:
RX:     4096   
RX Mini:    0
RX Jumbo:   0
TX:     4096   
Current hardware settings:
RX:     4096   
RX Mini:    0
RX Jumbo:   0
TX:     4096
```
> 注：更大的 ring buffer size 可以缓解丢包，但是不可以解决问题。另外，ring buffer的大小变大，可能会数据包在队列中缓存更长时间，导致时延变大。

#### 中断、队列与CPU之间关系
##### 概念
Linux 内核对计算机上所有的设备进行管理，进行管理的方式是内核和设备之间的通信。解决通信的方式有两种：   
1. 轮询。轮询是指内核对设备状态进行周期性的查询   
2. 中断。中断是指在设备需要CPU的时候主动发起通信

- 中断的原理
中断使得硬件可以发送知给处理器。例如敲击键盘时，键盘就会产生一个中断，通知操作系统有键被按下。中断本质上是一种电信号，由硬件设备生成，送入中断控制器的输入引脚中。中断控制器(如8259A)是个简单的电子芯片。当产生一个中断后，处理器会检测到一个电信号，中断自己的当前正在运行的程序，通知内核。内核调用一个称为中断处理程序（interrupt handler)或中断服务例程(interrupt service routine）的特定程序。中断处理程序或中断服务例程可以在中断向量表中找到，而这个中断向量表位于内存中的固定地址中。CPU处理中断后，就会恢复执行之前被中断的程序。
不同设备同时中断如何知道哪个中断是来自硬盘、哪个来自网卡呢?这个很容易，系统上的每个硬件设备都会被分配一个 IRQ 号，通过这个唯一的IRQ号就能区别不同硬件设备了。
![](attachments/Pasted%20image%2020230810115020.png)

- 中断的分类
中断可以分为NMI（不可屏蔽中断）和INTR（可屏蔽中断）
其中 NMI 是不可屏蔽中断，它通常用于电源掉电和物理存储器奇偶校验；INTR是可屏蔽中断，可以通过设置中断屏蔽位来进行中断屏蔽，它主要用于接受外部硬件的中断信号，这些信号由中断控制器传递给 CPU。

##### 原理
- 收包流程
接收到数据包之后，RSS进行hash得到队列信息，DMA通过PCIe将数据包发送给了某个ring buffer上，然后DMA发送硬件中断给内核。
- CPU和队列的绑定原理
支持RSS的网卡，通过多队列技术，每个队列对应一个中断号。
即：每个 队列 对应一个中断 + 中断和CPU core的绑定---->  cpu core和队列的绑定，即 达到了某个core上绑定的线程（DPDK用户态线程或者 内核的ksoftirqd线程）收取指定的 队列的数据包。

##### 硬中断相关文件
- /proc/interrupts：该文件存放了每个I/O设备的对应中断号、每个CPU的中断数、中断类型。
- /proc/irq/：该目录下存放的是以IRQ号命名的目录，如/proc/irq/40/，表示中断号为40的相关信息
- /proc/irq/[irq_num]/smp_affinity：该文件存放的是CPU位掩码（十六进制）。修改该文件中的值可以改变CPU和某中断的亲和性
- /proc/irq/[irq_num]/smp_affinity_list：该文件存放的是CPU列表（十进制）。注意，CPU核心个数用表示编号从0开始，如cpu0,cpu1等
- smp_affinity_list和smp_affinity任意更改一个文件都会生效，两个文件相互影响，只不过是表示方法不一致，但一般都是修改smp_affinity 文件

```c
以8核CUP为例，列出相关文件中如何表示CPU列表
 cup     二进制     smp_affinity_list      smp_affinity（十六进制）
cpu0     0001              0                                   1        
cpu1     0010              1                                   2
cpu2     0100              2                                   4
cpu3     1000              3                                   8
cpu4     010000            4                                   10
cpu5     0100000           5                                   40
......如上类推

这里可以得出一个公式:   
Python语法：
> smp_affinity = hex(2**(N-1))
N代表的是CPU核心数。那么，为什么是”N-1”呢。原因很简单，因为对于多核服务器而言，cpu编号是从cpu0开始的。比如24核心的服务器，cpu编号为cpu0-cpu23。
```

##### 设置和查看中断绑定的CPU
-  硬中断亲和性（IRQ affinities）的设置：
1》查看是否启动了 irqbalance
2》查看某个队列的中断号
3》设置该中断的CPU亲和性
![](attachments/Pasted%20image%2020230806153735.png)

**查看硬中断与对应CPU之间关系：**
```c
- 动态监控CPU中断情况，观察中断变化
  > watch -d -n 1 ‘cat /proc/interrupts’
- 查看网卡中断相关信息
  > cat /proc/interrupts | grep -E “eth|CPU”
```
![](attachments/Pasted%20image%2020230805222413.png)
如图所示，第一列是中断号, 中间部分是对应CPU处理该中断的次数,最后一列em*，rx*表示网卡队列的中断。该命令可以看出来cpu对应的中断数。

**硬中断和CPU的亲和性**
```c
[root@ccs ~]# ls  /proc/irq/
0  1  10  11  12  13  14  15  2  24  25  26  27  28  29  3  30  31  32  33  34  35  36  37  38  4  5  6  7  8  9  default_smp_affinity
```

smp_affinity_list显示CPU序号. 比如 0 代表 CPU0, 2代表 CPU2 smp_affinity 是十六进制显示. 比如 2 为10, 代表 CPU1 (第二个CPU，CPU0是第一个CPU core。)
```c
一个脚本查看中断号对应绑定的cpu：
#!/bin/bash
for ((i=31;i<=40;i+=1))
do
    cpu=$(cat /proc/irq/$i/smp_affinity_list)
    echo "Interrput $i belongs to cpu $cpu"
done
 
结果：
Interrput 31 belongs to cpu 6
Interrput 32 belongs to cpu 7
Interrput 33 belongs to cpu 1
Interrput 34 belongs to cpu 1
Interrput 35 belongs to cpu 2
Interrput 36 belongs to cpu 3
Interrput 37 belongs to cpu 6
Interrput 38 belongs to cpu 5
Interrput 39 belongs to cpu 5
Interrput 40 belongs to cpu 7

```
需要说明的一点是：
- /proc/irq/{IRQ_ID}/smp_affinity，中断IRQ_ID的CPU亲和配置文件，16进制
- /proc/irq/{IRQ ID}/smp_affinity_list，10进制，与smp_affinity相通，修改一个相应改变。

每个IRQ的默认的smp affinity在这里：cat /proc/irq/default_smp_affinity 。


##### irqbalance

- 相关文件与服务
![](attachments/Pasted%20image%2020230810112620.png)
![](attachments/Pasted%20image%2020230810113001.png)
说明：
命令行中设置 isolcpus 提供给DPDK程序的转发线程使用，那么操作系统给其他的进程分配的core就是剩余的非 isolcpus 的core。另外，DPDK程序屏蔽了收发包的中断，irqbalance中断平衡，也不要将其他的中断（比如管理口收发包）平衡到 DPDK程序转发线程所在的Core上。
因此，IRQBALANCE_BANNED_CPUS 是 启动参数中 isolate的16进制（即进制将中断分配给这些core），go_keepalived 这个控制进程的Core分配的是剩余的其他的非 isolcpus 的 core。

##### taskset 设置进程的亲和性
```c
查看进程的亲和性：
taskset -p PID

设置进程的亲和性：
taskset -p -c xxx PID
```

![](attachments/Pasted%20image%2020230810114024.png)
### sk_buff 缓冲区
sk_buff 缓冲区，是一个维护网络帧结构的双向链表，链表中的每一个元素都是一个网络帧（Packet）。虽然 TCP/IP 协议栈分了好几层，但上下不同层之间的传递，实际上只需要操作这个数据结构中的指针，而无需进行数据复制。

### socket缓冲区
套接字缓冲区，则允许应用程序，给每个套接字配置不同大小的接收或发送缓冲区。应用程序发送数据，实际上就是将数据写入缓冲区；而接收数据，其实就是从缓冲区中读取。
> 注：实际上，sk_buff、套接字缓冲、连接跟踪等，都通过 slab 分配器来管理。你可以直接通过 /proc/slabinfo，来查看它们占用的内存大小。
# 统计计数
## 丢包排查思路
网卡工作在数据链路层，数据链路层，会做一些校验，封装成帧。我们可以查看校验是否出错，确定传输是否存在问题。然后从软件层面，是否因为缓冲区太小丢包。
### 网卡工作模式
查看工作模式是否正常
```text
[root@localhost ~]# ethtool eth0 | egrep 'Speed|Duplex'
Speed: 1000Mb/s
Duplex: Full
```
### 查看crc检验是否正常
```text
[root@localhost ~]# ethtool -S eth0 | grep crc
rx_crc_errors: 0
```
如果crc统计存在增加，则：
![](attachments/Pasted%20image%2020230805215319.png)

>注意：speed，Duplex，CRC 之类的都没问题，基本可以排除物理层面的干扰。


## ifconfig 统计
```c
# ifconfig eth0  
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500  
inet 192.168.1.135 netmask 255.255.255.0 broadcast 192.168.1.255  
inet6 fe80::20c:29ff:fe9b:52d3 prefixlen 64 scopeid 0x20<link>  
ether 00:0c:29:9b:52:d3 txqueuelen 1000 (Ethernet)  
RX packets 833 bytes 61846 (60.3 KiB)  
RX errors 0 dropped 0 overruns 0 frame 0  
TX packets 122 bytes 9028 (8.8 KiB)  
TX errors 0 dropped 0 overruns 0 carrier 0 collisions 0
```

- 1) RX errors
表示总的收包的错误数量，这包括 too-long-frames 错误，Ring Buffer 溢出错误，crc 校验错误，帧同步错误，fifo overruns 以及 missed pkg 等等。

- 2) RX dropped
表示数据包已经进入了 Ring Buffer，但是由于内存不够等系统原因，导致在拷贝到内存的过程中被丢弃。

- 3) RX overruns
表示了 fifo 的 overruns，这是由于 Ring Buffer(aka Driver Queue) 传输的 IO 大于 kernel 能够处理的 IO 导致的。很明显，overruns 的增大意味着数据包没到 Ring Buffer 就被网卡物理层给丢弃了。

# 丢包分析
入队（入 ring buffer）丢包主要集中在PCIe异常“降速”方面。因为报文从网卡到系统是通过DMA经过PCIe总线来传输的，PCIe总线的吞吐将直接影响入队的速率。

出队(出ring buffer)问题主要集中在==应用程序性能不高、程序设计不优和CPU错误降频、CPU被其他的进程抢占==等方面。

## 丢包统计
```c
# ethtool ­S eth0
......
rx_dropped: 0
rx_fifo_errors: 0
tx_fifo_errors: 0
```
rx_fifo_errors 如果不为 0 的话（在 ifconfig 中体现为 overruns 指标增长）.
## pcie带宽以及参数设置
### pcie速率 与宽度
![](attachments/Pasted%20image%2020230805224511.png)
即看网卡的实际状态（LnkSta）中的speed以及宽度是否和能力（LnkCap）中一致。
注：对于PCIe的宽度以及速率出现问题， 很容易通过lspci看出问题。解决方法就是：更换网卡或者将网卡更换PCIe插槽，
### pcie的MPS以及MRRS
![](attachments/Pasted%20image%2020230805224638.png)

对于是否应该调整MPS以及MRRS，通过一下的判断标准：
1》网卡存在imiss/rx_fifo_error等丢包
2》应用程序的cpu使用率不高
3》ethtool 中存在 pause 帧发送统计/pci背压的统计

```c
# ethtool -S eth03 |grep -i -e control -e pause  -e pci
     tx_flow_control_xon: 0
     rx_flow_control_xon: 0
     tx_flow_control_xoff: 0
     rx_flow_control_xoff: 0

# ethtool -S eth03 |grep -i -e pause -e control -e pci
     tx_mac_control_phy: 0
     rx_mac_control_phy: 0
     rx_pause_ctrl_phy: 0
     tx_pause_ctrl_phy: 0
     rx_pci_signal_integrity: 0
     tx_pci_signal_integrity: 3
     outbound_pci_stalled_rd: 0
     outbound_pci_stalled_wr: 0
     outbound_pci_stalled_rd_events: 0
     outbound_pci_stalled_wr_events: 0
     rx_global_pause: 0
     rx_global_pause_duration: 0
     tx_global_pause: 0
     tx_global_pause_duration: 0
     rx_global_pause_transition: 0
     tx_pause_storm_warning_events: 0
     tx_pause_storm_error_events: 0
```
## ring buffer的大小设置
第一个要监控和调优的就是网卡的 RingBuffer，我们使用ethtool -g ethx 来来查看一下 Ringbuffer 的大小。
```c
# 查看
ethtool -g eth1

# 设置
ethtool ­G eth1 rx 4096 tx 4096
```
在 Linux 的整个网络栈中，RingBuffer 起到一个任务的收发中转站的角色。对于接收过程来讲，网卡负责往RingBuffer 中写入收到的数据帧，ksoftirqd 内核线程负责从中取走处理。只要 ksoftirqd 线程工作的足够快，RingBuffer 这个中转站就不会出现问题。但是我们设想一下，假如某一时刻，瞬间来了特别多的包，而 ksoftirqd处理不过来了，会发生什么？这时 RingBuffer 可能瞬间就被填满了，后面再来的包网卡直接就会丢弃，不做任何处理！

## 队列、中断和cpu的调整
硬中断的情况可以通过内核提供的伪文件 /proc/interrupts 来进行查看。拿飞哥手头的一台虚机来举例：
![](attachments/Pasted%20image%2020230805224948.png)
上述结果网卡的输入队列 virtio1-input.0 的中断号是 27，总的中断次数是 1109986815，并且 27 号中断都是由 CPU3 来处理的。

那么为什么这个输入队列的中断都在 CPU3 上呢？这是因为内核的一个中断亲和性配置，在我机器的伪文件系统中可以查看到。
```c
# cat /proc/irq/27/smp_affinity
8
```
smp_affinity 里是CPU的亲和性的绑定，8 是二进制的 1000, 第4位为 1。代表的就是当前的第 27 号中断的都由第 4 个 CPU 核心 - CPU3 来处理。

上面实例中，这台机器上 virtio 这块虚拟网卡上有四个输入队列，其硬中断号分别是 27、29、31 和 33。有独立的中断号就可以独立向某个 CPU 核心发起硬中断请求，让对应 CPU 来 poll 包。中断和 CPU 的对应关系还是通过 cat/proc/irq/{中断号}/smp_affinity 来查看。通过将不同队列的 CPU 亲和性打散到多个 CPU 核上，就可以让多核同时并行处理接收到的包了。这个特性叫做 RSS（Receive Side Scaling，接收端扩展），如图 2.3。这是加快 Linux内核处理网络包的速度非常有用的一个优化手段。
![](attachments/Pasted%20image%2020230805225204.png)

在网卡支持多队列的服务器上，想提高内核收包的能力，直接简单加大队列数就可以了，这比加大 RingBuffer 更为有用。因为加大 RingBuffer 只是给个更大的空间让网络帧能继续排队，而加大队列数则能让包更早地被内核处理。ethtool 修改队列数量方法如下：
```c
#ethtool ­L eth0 combined 32
```
不过在一般情况下，队列中断号和 CPU 之间的亲和性并不需要手工维护，有一叫irqbalance的服务来自动管理。通过 ps 命令可以查看到这个进程。
```c
# ps -ef | grep irqb
root       920     1  0 Jan09 ?        01:11:33 /usr/sbin/irqbalance --foreground
root     28904 21115  0 22:56 pts/0    00:00:00 grep --color=auto irqb
```

rqbalance 会根据系统中断负载的情况，自动维护和迁移各个中断的 CPU 亲和性，以保持各个 CPU 之间的中断开销均衡。如果有必要，irqbalance 也会自动把中断从一个 CPU 迁移到另一个 CPU 上。如果确实想自己维护亲和性，那得先关掉 irqbalance，然后再修改中断号对应的 smp_affinity。
```c
# service irqbalance stop
# echo 2 > /proc/irq/30/smp_affinity
```

## 硬中断合并
硬中断合并是指的攒一堆数据包后再通过一次硬中断来通知一次 CPU。

当网络包通过DMA到 内核的内存 后，接下来通过硬中断通知 CPU。那么你觉得从整体效率上来讲，是有包到达就发起中断好呢，还是攒一些数据包再通知 CPU 更好。

先允许我来引用一个实际工作中的例子，假如你是一位开发同学，和你对口的产品经理一天有10 个小需求需要让你帮忙来处理。她对你有两种中断方式：
- 第一种：产品经理想到一个需求，就过来找你，和你描述需求细节，然后让你帮你来改。
- 第二种：产品经理想到需求后，不来打扰你，等攒够 5 个来找你一次，你集中处理。
我们现在不考虑及时性，只考虑你的工作整体效率，你觉得那种方案下你的工作效率会高呢？显然第二种方案更好。对人脑来讲，频繁的中断会打乱你的计划，你脑子里刚才刚想到一半技术方案可能也就废了。当产品经理走了以后，你再想捡起来刚被中断之的工作的时候，很可能得花点时间回忆一会儿才能继续工作。

对于CPU来讲也是一样，CPU要做一件新的事情之前，要加载该进程的地址空间，load进程代码，读取进程数据，各级别 cache 要慢慢热身。因此如果能适当降低中断的频率，多攒几个包一起发出中断，对提升 CPU 的整体工作效率是有帮助的。所以，网卡允许我们对硬中断进行合并。

- 查看
```c
intel ixgbe 82599 10G网卡：
# ethtool -c eth03
Coalesce parameters for eth03:
Adaptive RX: off  TX: off
stats-block-usecs: 0
sample-interval: 0
pkt-rate-low: 0
pkt-rate-high: 0

rx-usecs: 1
rx-frames: 0
rx-usecs-irq: 0
rx-frames-irq: 0

tx-usecs: 0
tx-frames: 0
tx-usecs-irq: 0
tx-frames-irq: 0

rx-usecs-low: 0
rx-frame-low: 0
tx-usecs-low: 0
tx-frame-low: 0

rx-usecs-high: 0
rx-frame-high: 0
tx-usecs-high: 0
tx-frame-high: 0


----------

Mellanox Cx4-Lx 25G网卡：
# ethtool -c eth03
Coalesce parameters for eth03:
Adaptive RX: on  TX: on
stats-block-usecs: 0
sample-interval: 0
pkt-rate-low: 0
pkt-rate-high: 0

rx-usecs: 32
rx-frames: 64
rx-usecs-irq: 0
rx-frames-irq: 0

tx-usecs: 8
tx-frames: 128
tx-usecs-irq: 0
tx-frames-irq: 0

rx-usecs-low: 0
rx-frame-low: 0
tx-usecs-low: 0
tx-frame-low: 0

rx-usecs-high: 0
rx-frame-high: 0
tx-usecs-high: 0
tx-frame-high: 0

```
我们来说一下上述结果的大致含义：
- Adaptive RX:：自适应中断合并，网卡驱动自己判断啥时候该合并啥时候不合并
- rx-usecs：当过这么长时间过后，一个 RX interrupt 就会被产生
- rx-frames：当累计接收到这么多个帧后，一个 RX interrupt 就会被产生

- 设置
如果你想好了修改其中的某一个参数了的话，直接使用 ethtool -C 就可以，例如：
```c
# ethtool -­C eth0 adaptive­rx on
```
不过需要注意的是，减少中断数量虽然能使得 Linux 整体网络包吞吐更高，不过一些包的延迟也会增大，所以用的时候得适当注意。

## 软中断 budget 调整

net_rx_action 功能就是轮询调用 poll 方法，这里就是 ixgbe_poll。一次轮询的数据包数量不能超过内核参数 net.core.netdev_budget 指定的数量（默认值 300），并且轮询时间不能超过 2 个时间片。这个机制保证了单次软中断处理不会耗时太久影响被中断的程序。

- **==`netdev_budget`==**：一个 **==CPU 单次轮询所允许的最大收包数量==**。 单次 poll 收包时，所有注册到这个 CPU 的 NAPI 变量收包数量之和不能大于这个阈值。
```c
Maximum number of packets taken from all interfaces in one polling cycle (NAPI poll). In one polling cycle interfaces which are registered to polling are probed in a round-robin manner. Also, a polling cycle may not exceed netdev_budget_usecs microseconds, even if netdev_budget has not been exhausted.
```

- **==`netdev_budget_usecs`==**：每次 NAPI poll cycle 的最长允许时间，单位是 `us`。
```c
Maximum number of microseconds in one NAPI polling cycle. Polling will exit when either netdev_budget_usecs have elapsed during the poll cycle or the number of packets processed reaches netdev_budget.
```

>注：触发netdev_budget，netdev_budget_usecs 二者中任何一个条件后，都会导致一次轮询结束。

再举个日常工作相关的例子，不知道你有没有听说过番茄工作法这种高效工作方法。它的大致意思就是你在工作的时候，要有一整段的不被打扰的时间，集中精力处理某一项工作。这一整段时间时长被建议是 25 分钟。对于我们的Linux的处理软中断的 ksoftirqd 来说，它也和番茄工作法思路类似。一旦它被硬中断触发开始了工作，它会集中精力处理一波儿网络包（绝不只是1个），然后再去做别的事情。

我们说的处理一波儿是多少呢，策略略复杂。我们只说其中一个比较容易理解的，那就是net.core.netdev_budget 内核参数。

```
# 查看
$ sysctl -a | grep netdev_budget
net.core.netdev_budget = 2400
net.core.netdev_budget_usecs = 8000

# 修改
$ sudo sysctl -w net.core.netdev_budget=3600
$ sudo sysctl -w net.core.netdev_budget_usecs = 10000
```

NAPI的方法避免了每个包一个中断，通过一次中断，多次轮询的方式。ksoftirqd 处理一次软中断， 然后最多轮询处理300个包，处理够了就会把 CPU 主动让出来，以便 Linux 上其它的任务可以得到处理。
那么假如说，我们现在就是想提高内核处理网络包的效率。那就可以让 ksoftirqd 进程多干一会儿网络包的接收，再让出 CPU。至于怎么提高，直接修改这个参数的值就好了。
```c
#sysctl ­w net.core.netdev_budget=600
注：如果要保证重启仍然生效，需要将这个配置写到/etc/sysctl.conf
```

## netdev的 netdev_max_backlog 设置


netdev_max_backlog 是内核从 NIC 收到包后，交由协议栈（如 IP、TCP ）处理之前的缓冲队列。每个 CPU 核都有一个 backlog 队列，与 Ring Buffer 同理，当接收包的速率大于内核协议栈处理的速率时， CPU 的 backlog 队列不断增长，当达到设定的 netdev_max_backlog 值时，数据包将被丢弃。

### 统计
```c
# cat /proc/net/softnet_stat
2e8f1058 00000000 000000ef 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
0db6297e 00000000 00000035 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
09d4a634 00000000 00000010 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
0773e4f1 00000000 00000005 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
```
其中： 
每一行代表每个 CPU 核的状态统计，从 CPU0 依次往下；
每一列代表一个 CPU 核的各项统计：
第一列：`sd->processed`，是处理的网络帧的数量。如果你使用了 ethernet bonding， 那这个值会大于总的网络帧的数量，因为 ethernet bonding 驱动有时会触发网络数据被 重新处理（re-processed）。
第二列：`sd->dropped`，是因为处理不过来而 drop 的网络帧数量，代表由于 netdev_max_backlog 队列溢出而被丢弃的包总数。 
第三列：`sd->time_squeeze`，由于 budget 或 time limit 用完而退出 `net_rx_action` 循环的次数。
接下来的 5 列全是 0
第九列，`sd->cpu_collision`，是为了发送包而获取锁的时候有冲突的次数
第十列，`sd->received_rps`，是这个 CPU 被其他 CPU 唤醒去收包的次数
最后一列，`flow_limit_count`，是达到 flow limit 的次数。flow limit 是 RPS 的特性。

### 设置
netdev_max_backlog 的默认值是 1000，在高速链路上，可能会出现上述第二列统计不为 0 的情况，可以通过修改内核参数 net.core.netdev_max_backlog 来解决：
```c
sysctl -w net.core.netdev_max_backlog=2000
```
注：我们线上服务器的内核版本及网卡都支持 NAPI，而 NAPI 的处理逻辑是不会走到`enqueue_to_backlog`中的，`enqueue_to_backlog`主要是非 NAPI 的处理流程中使用的。


## LRO / GRO接收处理合并


硬中断合并是指的攒一堆数据包后再通知一次 CPU，不过数据包仍然是分开的。Lro（Large Receive Offload） /Gro（Generic Receive Offload） 还能把数据包合并起来后再往上层传递。
注：**Large Receive Offloading (LRO) 是一个硬件优化，GRO 是 LRO 的一种软件实现。**

两种方案的主要思想都是：**通过合并“足够类似”的包来减少传送给网络栈的包数，这有 助于减少 CPU 的使用量**。例如，考虑大文件传输的场景，包的数量非常多，大部分包都是一 段文件数据。相比于每次都将小包送到网络栈，可以将收到的小包合并成一个很大的包再送 到网络栈。GRO **使协议层只需处理一个 header**，而将包含大量数据的整个大包送到用 户程序。
![](attachments/Pasted%20image%2020230805231209.png)

Lro 和 Gro 的区别是合并包的位置不同。Lro 是在网卡上就把合并的事情给做了，因此要求网卡硬件必须支持才行。而 Gso 是在内核源码中用软件的方式实现的，更加通用，不依赖硬件，在内核协议栈之前。

- 缺点：
这类优化方式的缺点是 **信息丢失**：包的 option 或者 flag 信息在合并时会丢 失。这也是为什么大部分人不使用或不推荐使用 LRO 的原因。

- GRO和tcmdump的关系：
**如果用 tcpdump 抓包，有时会看到机器收到了看起来不现实的、非常大的包**， 这很可能是你的系统开启了 GRO。


- 查看
那么如何查看你的系统内是否打开了 LRO / GRO 呢？
```c
# ethtool -­k eth0
generic­receive­offload: on
large­receive­offload: on
...
```

- 设置
如果你的网卡驱动没有打开 GRO 的话，可以通过如下方式打开。
```c
# ethtool ­K eth0 gro on
# ethtool ­K eth0 lro on
```
注：对于大部分驱动，修改 GRO 配置会涉及先 down 再 up 这个网卡，因此这个网卡上的连接 都会中断。



## socket buffer
每个 Socket 都有一个读写缓存区。
- 读缓冲区，缓存远端发来的数据。如果读缓存区已满，就不能再接收新的数据。
- 写缓冲区，缓存了要发出去的数据。如果写缓冲区已满，应用程序的写操作就会阻塞。
![](attachments/Pasted%20image%2020230805215923.png)

## tcp的半连接队列和全连接队列
### 半连接队列
注：在半连接满的情况下，若启用syncookie机制，并不会直接丢弃 SYN 包，而是回复带有 syncookie 的 SYN+ACK 包，设计的目的是防范 SYN Flood 造成正常请求服务不可用。

### 全连接队列

## tcp的paws
### 背景
PAWS 全名 Protect Againest Wrapped Sequence numbers ，目的是解决在高带宽下， TCP 序列号在一次会话中可能被重复使用而带来的问题。
![](attachments/Pasted%20image%2020230805220517.png)
如上图所示，客户端发送的序列号为 A 的数据包 A1 因某些原因在网络中“迷路”，在一定时间没有到达服务端，客户端超时重传序列号为 A 的数据包 A2，并被对方接受。接下来假设带宽足够，传输用尽序列号空间，重新使用 A ，此时服务端等待的是序列号为 A 的数据包 A3 ，而恰巧此时前面“迷路”的 A1 到达服务端，如果服务端仅靠序列号A就判断数据包合法，就会将错误的数据传递到用户态程序，造成程序异常。
### paws 原理
PAWS 要解决的就是上述问题，它依赖于 timestamp 机制，理论依据是：在一条正常的 TCP 流中，按序接收到的所有 TCP 数据包中的 timestamp 都应该是单调非递减的，这样就能判断那些 timestamp 小于当前 TCP 流已处理的最大 timestamp 值的报文是延迟到达的重复报文，可以予以丢弃。在上文的例子中，服务器已经处理数据包 Z，而后到来的 A1 包的 timestamp 必然小于 Z 包的 timestamp ，因此服务端会丢弃迟到的 A1 包，等待正确的报文到来。

PAWS 机制的实现关键是内核保存了 **Per-Connection** 的最近接收时间戳，如果加以改进，就可以用来优化服务器TIME_WAIT状态的快速回收。

### TIME_WAIT 状态
TIME_WAIT 状态是TCP四次挥手中主动关闭连接的一方需要进入的最后一个状态，并且通常需要在该状态保持 2*MSL （报文最大生存时间），它存在的意义有两个：
1. 可靠地实现 TCP 全双工连接的关闭：关闭连接的四次挥手过程中，最终的 ACK 由主动关闭连接的一方（称为 A ）发出，如果这个 ACK 丢失，对端（称为 B ）将重发 FIN ，如果 A 不维持连接的 TIME_WAIT 状态，而是直接进入 CLOSED ，则无法重传 ACK ， B 端的连接因此不能及时可靠释放。
    
2. 等待“迷路”的重复数据包在网络中因生存时间到期消失：通信双方 A 与 B ， A 的数据包因“迷路”没有及时到达 B ， A 会重发数据包，当 A 与 B 完成传输并断开连接后，如果 A 不维持 TIME_WAIT 状态 2_MSL 时间，便有可能A与 B 再次建立相同源端口和目的端口的“新连接”，而前一次连接中“迷路”的报文有可能在这时到达，并被 B 接收处理，造成异常，维持 2_MSL 的目的就是等待前一次连接的数据包在网络中消失。
3. 
### tcp_tw_recycle
TIME_WAIT 状态的连接需要占用服务器内存资源维持， Linux 内核提供了一个参数来控制 TIME_WAIT 状态的快速回收：tcp_tw_recycle，它的理论依据是：

在 PAWS 的理论基础上，如果内核保存 **Per-Host** 的最近接收时间戳，接收数据包时进行时间戳比对，就能避免 TIME_WAIT 意图解决的第二个问题：前一个连接的数据包在新连接中被当做有效数据包处理的情况。这样就没有必要维持 TIME_WAIT 状态 2*MSL 的时间来等待数据包消失，仅需要等待足够的 RTO （超时重传），解决 ACK 丢失需要重传的情况，来达到快速回收 TIME_WAIT 状态连接的目的。
注：此时是Per-host的PAWS，而不是Per-Connection,是因为conn已经消失了，那就没有必要再per-conn的PAWS了。

### PAWS带来的问题
上述理论在多个客户端使用 NAT 访问服务器时会产生新的问题：同一个 NAT 背后的多个客户端时间戳是很难保持一致的（ timestamp 机制使用的是系统启动相对时间），对于服务器来说，两台客户端主机各自建立的 TCP 连接表现为同一个对端IP的两个连接，按照 Per-Host 记录的最近接收时间戳会更新为两台客户端主机中时间戳较大的那个，而时间戳相对较小的客户端发出的所有数据包对服务器来说都是这台主机已过期的重复数据，因此会直接丢弃。

通过netstat可以得到因PAWS机制timestamp验证被丢弃的数据包统计：
```c
# netstat -s |grep -e "passive connections rejected because of time stamp" -e "packets rejects in established connections because of timestamp”
387158 passive connections rejected because of time stamp
825313 packets rejects in established connections because of timestamp
```
通过sysctl查看是否启用了 tcp_tw_recycle 及 tcp_timestamp :
```c
$ sysctl net.ipv4.tcp_tw_recycle
net.ipv4.tcp_tw_recycle = 1
$ sysctl net.ipv4.tcp_timestamps
net.ipv4.tcp_timestamps = 1
```

解决方法：
如果服务器作为服务端提供服务，且明确客户端会通过 NAT 网络访问，或服务器之前有7层转发设备会替换客户端源IP时，是不应该开启 tcp_tw_recycle 的，而 timestamps 除了支持 tcp_tw_recycle 外还被其他机制依赖，推荐继续开启：
```c
sysctl -w net.ipv4.tcp_tw_recycle=0
sysctl -w net.ipv4.tcp_timestamps=1
```

## cpu是否降低频率
查看系统日志，出现CPU被错误降频。
![](attachments/Pasted%20image%2020230806151659.png)

- 查看当前的CPU频率 与运行模式
```c
# cpupower --help
Usage:	cpupower [-c|--cpu cpulist ] <command> [<args>]
Supported commands are:
	frequency-info
	frequency-set
	idle-info
	idle-set
	set
	info
	monitor
	help

Not all commands can make use of the -c cpulist option.

Use 'cpupower help <command>' for getting help for above commands.
```
![](attachments/Pasted%20image%2020230806151805.png)

查看cpu的每个core的频率：
![](attachments/Pasted%20image%2020230806151818.png)

如果当前运行在powersave模式下，可以将其修改为performance，提升CPU频率。
```c
cpupower frequency-set -g performance。
```

## 应用程序的性能
程序的**实际CPU使用率**达到100%，不能及时处理队列报文，导致队列报文溢出，持续imissed++，此时出队平均速率是小于入队平均速率。
inux系统下可以使用perf性能分析工具，做热点函数分析，perf安装命令yum install perf。perf常用的热点函数定位命令如下：
```c
进程级：perf top -p
线程级：perf top -t
线程tid可以通过pidstat -t -p 或者 top -p PID -H 获取。
```
# 丢包工具
## dropwatch
```c
# dropwatch -l kas
Initalizing kallsyms db
dropwatch> start
Enabling monitoring...
Kernel monitoring activated.
Issue Ctrl-C to stop monitoring
1 drops at sk_stream_kill_queues+50 (0xffffffff81687860)
1 drops at tcp_v4_rcv+147 (0xffffffff8170b737)
1 drops at __brk_limit+1de1308c (0xffffffffa052308c)
1 drops at ip_rcv_finish+1b8 (0xffffffff816e3348)
1 drops at skb_queue_purge+17 (0xffffffff816809e7)
3 drops at sk_stream_kill_queues+50 (0xffffffff81687860)
2 drops at unix_stream_connect+2bc (0xffffffff8175a05c)
2 drops at sk_stream_kill_queues+50 (0xffffffff81687860)
1 drops at tcp_v4_rcv+147 (0xffffffff8170b737)
2 drops at sk_stream_kill_queues+50 (0xffffffff81687860)
```
## perf
`perf` 监视 `kfree_skb` 事件。

```c
# perf record -g -a -e skb:kfree_skb
^C[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 1.212 MB perf.data (388 samples) ]

# perf script
containerd 93829 [031] 951470.340275: skb:kfree_skb: skbaddr=0xffff8827bfced700 protocol=0 location=0xffffffff8175a05c
            7fff8168279b kfree_skb ([kernel.kallsyms])
            7fff8175c05c unix_stream_connect ([kernel.kallsyms])
            7fff8167650f SYSC_connect ([kernel.kallsyms])
            7fff8167818e sys_connect ([kernel.kallsyms])
            7fff81005959 do_syscall_64 ([kernel.kallsyms])
            7fff81802081 entry_SYSCALL_64_after_hwframe ([kernel.kallsyms])
                   f908d __GI___libc_connect (/usr/lib64/libc-2.17.so)
                  13077d __nscd_get_mapping (/usr/lib64/libc-2.17.so)
                  130c7c __nscd_get_map_ref (/usr/lib64/libc-2.17.so)
                       0 [unknown] ([unknown])

containerd 93829 [031] 951470.340306: skb:kfree_skb: skbaddr=0xffff8827bfcec500 protocol=0 location=0xffffffff8175a05c
            7fff8168279b kfree_skb ([kernel.kallsyms])
            7fff8175c05c unix_stream_connect ([kernel.kallsyms])
            7fff8167650f SYSC_connect ([kernel.kallsyms])
            7fff8167818e sys_connect ([kernel.kallsyms])
            7fff81005959 do_syscall_64 ([kernel.kallsyms])
            7fff81802081 entry_SYSCALL_64_after_hwframe ([kernel.kallsyms])
                   f908d __GI___libc_connect (/usr/lib64/libc-2.17.so)
                  130ebe __nscd_open_socket (/usr/lib64/libc-2.17.so)
```
## tcpdrop
`tcpdrop`，它显示了源包和目标包的详细信息，以及 TCP 会话状态(来自内核)、TCP 标志(来自包 TCP 报头)和导致这次丢包的内核堆栈跟踪。
```c
TIME     PID    IP SADDR:SPORT          > DADDR:DPORT          STATE (FLAGS)
05:46:07 82093  4  10.74.40.245:50010   > 10.74.40.245:58484   ESTABLISHED (ACK)
    tcp_drop+0x1
    tcp_rcv_established+0x1d5
    tcp_v4_do_rcv+0x141
    tcp_v4_rcv+0x9b8
    ip_local_deliver_finish+0x9b
    ip_local_deliver+0x6f
    ip_rcv_finish+0x124
    ip_rcv+0x291
    __netif_receive_skb_core+0x554
    __netif_receive_skb+0x18
    process_backlog+0xba
    net_rx_action+0x265
    __softirqentry_text_start+0xf2
    irq_exit+0xb6
    xen_evtchn_do_upcall+0x30
    xen_hvm_callback_vector+0x1af

05:46:07 85153  4  10.74.40.245:50010   > 10.74.40.245:58446   ESTABLISHED (ACK)
    tcp_drop+0x1
    tcp_rcv_established+0x1d5
    tcp_v4_do_rcv+0x141
    tcp_v4_rcv+0x9b8
    ip_local_deliver_finish+0x9b
    ip_local_deliver+0x6f
    ip_rcv_finish+0x124
    ip_rcv+0x291
    __netif_receive_skb_core+0x554
    __netif_receive_skb+0x18
    process_backlog+0xba
    net_rx_action+0x265
    __softirqentry_text_start+0xf2
    irq_exit+0xb6
    xen_evtchn_do_upcall+0x30
    xen_hvm_callback_vector+0x1af
```
# 参考
```c
https://zhuanlan.zhihu.com/p/150086151
https://simonzgx.github.io/2020/08/17/Linux%E7%BD%91%E7%BB%9C%E6%95%B0%E6%8D%AE%E5%8C%85%E6%8E%A5%E5%8F%97%E8%BF%87%E7%A8%8B/

# 网络性能优化：
https://mp.weixin.qq.com/s/JR-qqjNG9ClHCYoRiFg-CQ

# 丢包查看：
https://leeweir.github.io/posts/linux-packet-loss/

收包调优：
https://arthurchiao.art/blog/tuning-stack-rx-zh/#353-using-procnetdev


redis的性能低的排查：
https://www.infoq.cn/article/ux4u1gaidcmtvj8t8xxg

imiss问题：
https://blog.csdn.net/legend050709/article/details/123655712?csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22123655712%22%2C%22source%22%3A%22legend050709%22%7D
```