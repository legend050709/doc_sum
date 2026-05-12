```table-of-contents
```
# 简介
## polling（轮询）
中断是外部设备向处理器发起的请求事件，而轮询是处理器主动进行的。
拿网卡收到数据包这个过程举例，中断模式下，当网卡收到一个数据包，将其放入网卡的缓冲队列，
然后给处理器发一个硬中断来告诉处理器有数据包到了，然后处理器响应中断，
将数据包从网卡缓存读到内存。
现在有一个问题，如果遇到I/O密集型应用，使用中断模式，就会导致处理器频繁响应中断，从而导致处理器花费大量时间在响应中断上，造成浪费处理器。

轮询模式如何工作呢，轮询就是处理器主动去check有没有数据包到来，
这样一来就可以省了硬中断的开销，使用轮询一般是I/O密集型的应用，
因为如果数据量很小，使用轮询模式会导致处理器空转，也会浪费处理器，这时候就应该使用中断。
![](attachments/Pasted%20image%2020230918225227.png)
## mmap
## pf_ring
PF_RING是Luca研究出来的基于Linux内核级的高效数据包捕获技术。
PF_RING是一个新类型socket，采用轮询、mmap和环形buffer实现，PF_RING存在一次copy。

# 原理
PF_RING noZC的原理如下所示：

![](attachments/Pasted%20image%2020230918201911.png)
PF_RING的具体流程如下：

- 应用调用mmap进行内存映射；
- 数据包到达网卡后，处理器以轮询polling的方式将数据包写到环形buffer；
- 应用调用read从环形buffer里面读数据包；

![](attachments/Pasted%20image%2020230918195537.png)

# 使用与实现



# 优缺点
PF_RING提出的核心解决方案便是减少报文在传输过程中的拷贝次数。
![](attachments/Pasted%20image%2020230918194607.png)
和基于PF_PACKET套接字的libpcap不同的是，PF_RING机制更为灵活：  
1.PF_RING采用mmap的方式将网络裸数据放在一个用户态可以直接access的地方，而不是通过socket read/write机制的内存拷贝；
2.PF_RING支持下面1到3三种方式将裸数据放到mmap到用户态的环形缓冲区以及4的DNA方式【越高的transparent_mod值，获得报文捕获的速度越快。】：
1. 按照PACKET套接字的方式从netif_receive_skb函数中抓取数据包，这是一种和PACKET套接字兼容的方式，所不同的是数据包不再通过socket IO进入用户态，而是通过mmap；(transparent_mode 0)
2. 直接在NAPI层次将数据包置入到所谓的环形缓冲区，同时NAPI Polling到skb对列，对于这两个路径中的第一个而言，这是一种比2.1介绍的方式更加有效的方式，因为减少了数据包在内核路径的处理长度，但是要求网卡支持NAPI以及PF_RING接口(一般而言，NAPI会将数据包Polling到一个skb队列)。(transparent_mode 1)
3. 和2相同，只是不再执行NAPI Polling。这就意味着，数据包将不会进入内核，而是直接被mmap到了用户态，这特别适合于用户态的完全处理而不仅仅是网络审计，既然内核不需要处理网络数据了，那么CPU将被节省下来用于用户态的网络处理。这可能会将内核串行的网络处理变为用户态并行的处理。(transparent_mode 2)
4. 这是一种更猛的方式，唤作DNA支持的模式，直接绕过内核协议栈的所有路径，也就是说直接在网卡的芯片中将数据包传输到(DMA的方式)所谓环形缓冲区，内核将看不到任何数据包，这种方式和Intel的万兆猛卡结合将是多么令人激动的事啊；(DNA技术)


# 使用

# 比较
在Linux上捕获数据包有多种方式，常见的有libpcap，pf-ring等。DPDK以高性能著称，想必相比传统的数据包捕获方式，一定有其独到之处。
传统的数据包捕获瓶颈往往在于Linux Kernel，数据流需要经过Linux Kernel，就会带来Kernel Spcae和User Space数据拷贝的消耗；系统调用的消耗；中断处理的消耗等。
![](attachments/Pasted%20image%2020230914105244.png)
## libpcap
libpcap采用的是传统的数据包抓取方式。
tcpdump由C语言开发，主要功能通过libpcap库实现，而libpcap是linux平台下的一个网络数据包捕获功能包, 通过内核BPF技术实现数据过滤功能。
tcpdump使用BPF虚拟机的指令集定义过滤器表达式，然后传递给内核，并由解释器执行，这**使得包过滤可以在内核中进行，避免了向用户态进程复制全部数据包**，从而提升数据包的过滤性能。
tcpdump将包过滤指令注入到内核，返回按条件过滤的数据包。
![](attachments/Pasted%20image%2020230914105757.png)
![](attachments/Pasted%20image%2020230914110018.png)

### 原理
#### 总体原理
libpcap 主要由两部份组成：

- 网络分接头（Network Tap）
- 数据过滤器（Packet Filter）

网络分接头从网络设备驱动程序（NIC driver）中拷贝收集数据拷贝，过滤器决定是否接收该数据包。

网络分接口（Network Tap）从网络设备驱动程序中收集数据拷贝（旁路机制，并不干扰系统自身的网路协议栈的处理）；
BPF过滤器（Packet Filter）决定是否接收该数据包。bpf根据已经定义好的包过滤规则对数据包进行过滤，匹配成功的，进行**拷贝**放到内核缓冲区，并等待用户态调系统调用去取（read）。
![](attachments/Pasted%20image%2020230914111449.png)

注：上面的图不要理解错误；并不是网络分接口「应该是有抓包程序在某个接口上抓包就会有网络分接口」的地方将每个包都进行拷贝，而是进来一个包，先进行数据包的BPF packet filetr进行匹配，匹配成功后，进行拷贝到缓冲区（缓冲区大小一般为4KB，tcpdump可以通过 -B 来调整）。匹配不成功，正常走后续的内核协议栈。【参考下图的网络收包抓取的图更加准确】
![](attachments/Pasted%20image%2020230914173201.png)

#### 包处理路径
libpcap绕过了Linux内核收包流程中协议栈部分的处理，使得用户空间API可以直接调用套接字PF_PACKET从**链路层**驱动程序中获得数据报文的拷贝，将其从内核缓冲区拷贝至用户空间缓冲区。进而调用recvfrom函数获得捕获的报文.
![](attachments/Pasted%20image%2020230914112212.png)
```c
fd＝socket(PF_PACKET,sock_RAW,htons(ETH_P_ALL))  
libpcap 函数库注册的报文接收类型为 ETH_P_ALL，即接收所有的网络数据帧，其处理函数为 packet_rcv()。该函数工作在数据链路层。
packet_rcv() 函数将直接调用 skb_queue_tail() 将数据报文存放在代表相应网络连接控制结构（struct sock）的接收队列 receive_queue 中。这样数据报文在接收过程中就绕过了 TCP 层和 IP 层繁琐的协议处理过程。最后，睡眠在 sk 等待队列上的函数 packet_recvmsg() 会接收链路层数据帧并将该数据帧直接拷贝到应用程序缓冲区中。
```

#### BPF过滤器
![](attachments/Pasted%20image%2020230914170007.png)
BPF本质上来说是一个设备驱动(device driver)，能够被应用程序用来读取网络上通过这个网络适配器的包。
BPF正常情况下被用作**诊断工具**，去检查与本机相连的网络的流通状况。
一个BPF设备能够配置一个filter，根据这个filter的特征，来忽略或者接收到来的包。

BPF拥有两个组件：the network tap 和 the packet filter。
- the network tap
收集来自网络设备驱动的包的一个拷贝，并把它专递给监听程序。
- the packet filter
决定是否接收这个包并且把它拷贝给监听程序。

通常情况下，当一个包到达网络接口时，DMA将把它发送到内核。但是当BPF在这个接口上面监听时，网络设备驱动将首先调用BPF的network tap函数。这个tap函数将包送入每一个监听程序的filter。
而用户定义的filter决定：是否接收这个包，以及每一个包有多少字节将会被保存。
如果filter接收这个包，那么tap将会从内核的数据包缓存中拷贝一定数目的字节数到与这个filter关联的buffer中。
同时，网络接口的设备驱动将会重新获得控制权，且正常的协议处理将会进行。


### 网络收包抓取
![](attachments/Pasted%20image%2020230914110747.png)

应用接收报文时，在网络设备层，驱动程序首先调用内核函数netif_receive_skb()，通过deliver_skb()调用回调函数packet_rcv()，并使用BPF运行函数__bpf_prog_run()，来执行BPF程序过滤数据包，然后将数据包存入队列，最终复制数据包给tcpdump。而应用接收数据包则根据包的协议，选择udp或者tcp将报文送到用户进程。

### 网络发包抓取
![](attachments/Pasted%20image%2020230914111017.png)
应用在发送报文时，首先通过邻居子系统进入网络设备层，然后调用内核函数dev_hard_start_xmit（），该函数同样使用网络收包流程中使用的deliver_skb()函数调用回调函数packet_rcv()，并通过调用BPF运行函数__bpf_prog_run()，来执行BPF程序过滤数据包，然后将数据包存入队列，最终复制数据包给tcpdump。而应用发送数据包则通过驱动程序发送出去。


### 具体流程
libpcap的包捕获机制是在数据链路层增加一个旁路处理，不干扰系统自身的网路协议栈的处理，对发送和接收的数据包通过Linux内核做过滤和缓冲处理，后直接传递给上层应用程序。

1. 数据包到达网卡设备。
    
2. 网卡设备依据配置进行DMA操作。（ **「第1次拷贝」** ：网卡寄存器->内核为网卡分配的缓冲区ring buffer）
    
3. 网卡发送中断，唤醒处理器。
    
4. 驱动软件从ring buffer描述符对应的数据包缓冲区中读取包
    
5. 接着调用netif_receive_skb函数：
    

- 5.1 如果有抓包程序，由网络分接口进入BPF过滤器，将规则匹配的报文拷贝到系统内核缓存 （ **「第2次拷贝」** ）。
    
- 5.2 处理数据链路层的桥接功能；
    
- 5.3 根据skb->protocol字段确定上层协议并提交给网络层处理，进入网络协议栈，进行高层处理。
    

libpcap绕过了Linux内核收包流程中协议栈部分的处理，使得用户空间API可以直接调用套接字PF_PACKET从链路层驱动程序中获得数据报文的拷贝，将其从内核缓冲区拷贝至用户空间缓冲区（ **「第3次拷贝」** ）

### libpcap-mmap
libpcap-mmap是对旧的libpcap实现的改进，新版本的libpcap基本都采用packet_mmap机制。PACKET_MMAP通过mmap，减少一次内存拷贝（ **「第3次拷贝没有了」** ），减少了频繁的系统调用，大大提高了报文捕获的效率。


## pf_ring
![](attachments/Pasted%20image%2020230918200132.png)
上表给出了使用PF_RING后的抓包效果。
首先看第二列和第五列_，_linux有PF_RING的加持后，不管小包还是大包，抓包能力都有了大幅提升，特别是小包，从原来的2.5%直接提升到了75.5，
大包的抓包效果相比于单纯使用NAPI和mmap倒是反倒有一些下降。

pf-ring采用`mmap()`将传统的两次拷贝(不考虑DMA到内核的拷贝)减少至一次拷贝。  
第一步，同样使用DMA技术，把数据从网卡拷贝到主存（RX buffer）中。  
第二步，pf-ring会将网卡通过DMA传输进入主存（RX buffer）的内容拷贝一份放入环形缓冲区中（ring）中，这里进行了一次数据拷贝。  
第三步，使用`mmap()`将环形缓冲区映射至用户空间，用户空间可以直接访问这个环形缓冲区中的数据。

## pf_ring zc
pf-ring zc（zero copy）更是将pf-ring的一次拷贝也省去，达到了零拷贝的目的，具体的：  
第一步，使用DMA技术，把数据从网卡拷贝到主存（RX buffer）中。  
第二步，使用`mmap()`直接将RX buffer的数据映射到用户用户空间，使用户空间可以直接访问RX buffer的数据。

## dpdk zc
![](attachments/Pasted%20image%2020230914104254.png)
DPDK针对Linux Kernel传统的数据包捕获模式的问题，进行了一定程度的优化。DPDK的优化可以概括为:
- UIO+mmap 实现零拷贝（zero copy）
- UIO+PMD 减少中断和CPU上下文切换
- HugePages 减少TLB miss
- 其他代码优化
### uio
UIO（Userspace I/O）是运行在用户空间的I/O技术。Linux系统中一般的驱动设备都是运行在内核空间，而在用户空间用应用程序调用即可，而UIO则是将驱动的很少一部分运行在内核空间，而在用户空间实现驱动的绝大多数功能。
![](attachments/Pasted%20image%2020230914104716.png)

一个设备驱动的主要任务有两个：
- 存取设备的内存
- 处理设备产生的中断
如上图所示，对于存取设备的内存 ，UIO 核心实现了`mmap()`；对于处理设备产生的中断，内核空间有一小部分代码用处理中断，用户空间通过`read()`接口`/dev/uioX`来读取中断。

# 参考
```c
https://www.modb.pro/db/634018
https://z.itpub.net/article/detail/E7282132F901DCA52E32EB06F25D6CF2
https://www.cnblogs.com/x_wukong/p/7058265.html



https://blog.csdn.net/yilun/article/details/124846435
https://www.jianshu.com/p/9b669f7c97ce
```