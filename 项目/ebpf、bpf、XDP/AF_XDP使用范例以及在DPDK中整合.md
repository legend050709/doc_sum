```table-of-contents
```


# AF_XDP Socket

## AF_XDP 概述

![](attachments/Pasted%20image%2020240710103100.png)

内核新增AF_XDP协议族，在内核XDP框架中被匹配的数据包通过其送抵用户态，这又将**XDP的支持从内核拓展到用户态应用场景。**

AF_XDP类似于AF_NET（TCP/IP等）等一样，是一个协议族，大家有没有记得在TCP网络编程时经常写的一行代码“.sin_family = AF_INET”，就是这个意思。
XDP中，可以通过`XDP_REDIRECT`进行数据的重定向，而`AF_XDP`则利用函数`bpf_redirect_map`实现数据的重定向。所谓**重定向，其实就是把数据传输到指定的位置。一般来说这个位置是一块内存或一个设备等**，此函数会重定向一块内存中，这就大大方便了开发者的数据处理方式。


很多同学容易将XDP和AF_XDP技术给弄混淆。  
- XDP技术是基于BPF技术的一种新的网络技术。
- AF_XDP是XDP技术的一种应用场景，AF_XDP是一种高性能Linux socket。
```bash
socket(AF_XDP, SOCK_RAW, 0);
```

![](attachments/Pasted%20image%2020240709204457.png)


此外需要注意的事，**AF_XDP socket不再通过 send()/recv()等函数实现报文收发，而实通过直接操作ring来实现报文收发**。

## AF_XDP和其他类型的socket对比

### AF_XDP 和 AF_INET对比

所谓`AF_XDP`，和`AF_INET`一样，也是`address family`的一种，用于规定socket通讯的类型。相当于socket底层通讯方式的不同实现（多态）。一般的，`AF_INET`可以用于IPv4类型地址的通讯，在实际通讯中应用自己的那套具体实现（TCP/IP协议栈等），`AF_XDP`就是一套基于XDP的通讯的实现。

![](attachments/Pasted%20image%2020240710115429.png)

XDP程序在Kernel提供的网卡驱动中直接取得网卡收到的数据帧，然后直接送到用户态应用程序。应用程序利用`AF_XDP`类型的socket接收数据。

用虚拟化领域的完全虚拟化和半虚拟化概念类比，如果DPDK是”完全Kernel bypass”，那么AF_XDP就是“半Kernel bypass”。

### AF_XDP 和 AF_PACKET 对比

AF_PACKET类型的socket，它的性能着实一般，所有的数据都得在用户态和内核态之间做转换，而且在高并发的情况下还有大量的中断。

![](attachments/Pasted%20image%2020240710151139.png)



## UMEM共享内存

 用户程序，XDP驱动程序会操作一个共享的内存区域，称之为UMEM。

具体就是：XDP程序会把数据帧送到一个在用户态可以读写的队列Memory Buffer中，即`UMEM`。用户态应用在`UMEM`中完成数据帧的读取和写入。

UMEM是一串连续的内存块，被分割为相等大小的”帧“（帧的实际大小由调用者指定）；每个帧都能够储存一个独立的网络包。

![](attachments/Pasted%20image%2020240710163246.png)

在应用分配内存之后，该数组就通过 setsockopt() 系统调用的 XDP_UMEM_REG 命令进行注册。
```bash
setsockopt(umem->fd, SOL_XDP, XDP_UMEM_REG, &mr, sizeof(mr));
```

UMEM共享内存通常以4K或者2K为一个单元，每个单元可以存储一个数据包，UMEM共享内存通常为4096个单元。接收和发送的数据包都是存储在UMEM内存单元。

用户程序和内核都可以直接操作这块内存区域，所以发送和接收数据包时，只是简单的内存拷贝，不需要进行系统调用。用户程序需要维护一个UMEM内存使用记录，记录每一个UMEM单元是否已被使用，每个记录都会有一个相对地址，用于定位UMEM内存单元地址。

## 无锁环形队列

AF_XDP socket总共有4个无锁环形队列，分别为：

- 填充队列（FILL RING）
    
- 已完成队列（COMPLETION RING）
    
- 发送队列（TX RING）  
    
- 接收队列（RX RING）

==注意：这四个队列都是单生产者单消费者模式==。

![](attachments/Pasted%20image%2020240710145603.png)


```bash
//创建FILL RING
setsockopt(fd, SOL_XDP, XDP_UMEM_FILL_RING, &umem->config.fill_size, sizeof(umem->config.fill_size));

//创建COMPLETION RING
setsockopt(fd, SOL_XDP, XDP_UMEM_COMPLETION_RING, &umem->config.comp_size, sizeof(umem->config.comp_size));

//创建RX RING
setsockopt(xsk->fd, SOL_XDP, XDP_RX_RING, &xsk->config.rx_size, sizeof(xsk->config.rx_size));

//创建TX RING
setsockopt(xsk->fd, SOL_XDP, XDP_TX_RING, &xsk->config.tx_size, sizeof(xsk->config.tx_size));
```

我们使用普通的 `socket()` 系统调用创建一个AF_XDP套接字（XSK）。

![](attachments/Pasted%20image%2020240710163811.png)

(1) **每个XSK都有两个ring：`RX RING` 和 `TX RING`**。
> 通过 socket 系统调用创建 AF_XDP socket，创建之后每个 socket 都各自分配了一个 RX ring 和 TX ring。这两个 ring 需要通过 socket 选项 XDP_RX_RING 和 XDP_TX_RING 进行注册。每个 socket 必须至少具有其中一个ring。RX 或 TX ring 存储着描述符集合，每个描述符指向 UMEM 中的一个 frame，描述符通过引用 frame 在 UMEM 中的 偏移量来引用 frame。


(2) **UMEM也有两个 ring：`FILL RING` 和 `COMPLETION RING`**。

![](attachments/Pasted%20image%2020240716104346.png)

> 应用程序使用 `FILL RING` 向内核发送可以承载报文的 `addr` (该 addr 指向UMEM中某个chunk)，以供内核填充RX数据包数据。每当收到数据包，对这些 chunks 的引用就会出现在RX环中。另一方面，`COMPLETION RING`包含内核已完全传输的 chunks 地址，可以由用户空间再次用于 TX 或 RX。

> ==其实就和传统网络IO过程中给DMA填写的接收和发送环形队列很类似==。在**RX**过程中，**用户应用程序**在`FILL ring`中填入接收数据包的地址，**XDP程序**会将接收到的数据包放入该地址中，并在socket的RX ring中填入对应的descriptor。
> 但`COMPLETION ring`中保存的并非用户应用程序“将要”发送的帧的地址，而是已经完成发送的帧的地址。这些帧可以用来被再次发送或者接收。==“将要”发送的帧的地址是在socket的TX ring中，同样由用户应用填入==。


![](attachments/Pasted%20image%2020240710151517.png)

可以看到，这里有四个环，`RX RING` 和 `TX RING`环中的数据是描述符(xdp_desc)，而`FILL RING` 和 `COMPLETION RING`是地址(u64)。

**我的理解**：

![](attachments/Pasted%20image%2020240718155317.png)

`RX RING` 和 `TX RING`是 per AF_XDP 的，即每个 AF_XDP socket 都有这2个ring。ring中的元素是 描述符， 指向`UMEM`中真正保存帧的地址。

`FILL RING` 和 `COMPLETION RING`是 per UMEM的，有可能多个 AF_XDP 共享一个 UMEM的。其中 `FILL RING` 可以理解为 UMEM 中可用的内存缓冲区，`COMPLETION RING`理解为  UMEM 中完成传输（即完成使用）的 内存缓冲区，可再次被使用的缓冲区。

### Rx Ring

Rx Ring的生产者是XDP程序，消费者是用户态程序；
XDP程序消耗 Fill Ring，获取可以承载报文的desc并将报文拷贝到desc中指定的地址，然后将desc填充到 Rx Ring 中，并通过socket IO机制通知用户态程序从 Rx Ring 中接收报文。

### Fill Ring

填充环（Fill Ring）是用户空间程序为接收环生成新描述符的环，以便接收环始终有足够的描述符可供使用。
Fill Ring 的生产者是用户态程序，消费者是内核态中的XDP程序；用户态程序通过Fill Ring 将可以用来承载报文的 UMEM frames 传到内核，然后内核消耗Fill Ring 中的描述符desc，并将报文拷贝到desc中指定地址（该地址即UMEM frame的地址）。

### Tx Ring

Tx Ring的生产者是用户态程序，消费者是XDP程序；
用户态程序将要发送的报文拷贝 Tx Ring 中 desc指定的地址中，然后 XDP程序 消耗 Tx Ring 中的desc，将报文发送出去，并通过 Completion Ring 将成功发送的报文的desc告诉用户态程序；

### Completion Ring
Completion Ring的生产者是XDP程序，消费者是用户态程序；
当**内核完成XDP报文的发送**，会通过 completion_ring 来通知用户态程序，哪些报文已经成功发送，然后用户态程序消耗 completion_ring 中 desc(只是更新consumer计数相当于确认)；

###  描述符在4个ring之前的状态转换

我们可以使用一个状态机来跟踪这些转换。下图显示了所有可能的有效转换。

![](attachments/Pasted%20image%2020240718195745.png)


### af-xdp socket、umem、xdp驱动程序、ebpf-map、接收队列的关系

![](attachments/Pasted%20image%2020240718195830.png)

rx-ring、tx-ring 是 per af-xdp socket的；
fill-ring、comlete-ring 是 per umem的；

多个AF-xdp socket 可以对应 一个 umem，也可以对应多个 umem。
每个 af-xdp socket 最好只关联一个 接收队列，且各不相同。

如果说 网卡的队列个数 = unem的个数；用户线程个数 = af xdp socket 个数 ；
xdp prog 程序就是一个，xdp prog 中的 map 个数 可以为 n 个。

**只要保证网卡队列个数（即xdp驱动程序） =  fill/comlete 队列个数； 用户线程个数 = xdp socket个数 = rx/tx ring 个数，就可以保证 4个ring （rx/tx/fill/complete）中的每个ring 的操作都是 SPSC**。

至于 umem的 fill/complete 个数 和 xdp_socket 的 rx/tx个数没有必然的1:1的关系。
可以一个 umem 创建多个 fill/complete ring, 因为 umem 只是一段内存区域。

![](attachments/Pasted%20image%2020240723151353.png)

其实没有必要将多个xdp socket绑定到一个网卡的队列，然后在xdp驱动程序中进行轮询负载均衡。因为网卡支持RSS进行负载均衡到多个队列。如果网卡只有一个接受队列，可以这样来在XDP程序中进行分发到多个xdp socket中。
即：无论怎么样，保证4个ring的操作都是SPSC。

## AF_XDP高效的原因

### 总体概述
**AF_XDP之所以高效，主要有三大原因：**  

- （1）内核旁路技术  
    内核旁路技术在处理网络数据包的时候，可以跳过Linux内核协议栈，相当于走了捷径，这样可以降低链路开销。

- （2）内存映射  
    用户程序和内核共享UMEM内存和无锁环形队列，采用mmap技术将内存进行映射，用户操作UMEM内存**不需要进行系统调用**，减少了系统调用上下文切换成本。  

- （3）无锁环形队列
    无锁环形队列采用原子变量实现，可以减少线程切换和上下文切换成本。  

- （4）支持多队列
AF_XDP socket支持多个队列，可以将不同的网络流量路由到不同的队列中，从而实现更好的负载均衡和多核利用。

- （5）支持用户空间协议栈
AF_XDP socket可以与用户空间中的协议栈结合使用，从而可以在用户空间中实现网络协议栈，提高了网络应用程序的性能和灵活性。

总之，AF_XDP socket是一种高性能的网络数据传输方式，适用于需要处理大量数据的高性能网络应用程序。

### 详细分析 

![](attachments/Pasted%20image%2020240716174138.png)

XDP本质是内核 快路径的处理程序，而不是bypass-kernel的处理。AF_XDP 可以将数据帧送到用户态进行处理，bypass了kernel。

**提升AF_XDP程序的性能的方式**：
- （1）AF_XDP 使用 SPSC 来操作 ring queue 来提高性能：

1> rx ring的单生产者：af_xdp 绑定到了一个网卡的接收队列，NAPI-softirq 保证了一个接受队列就一个core来处理（通过中断和cpu core绑定来实现）。

2> rx ring的单消费者：应用程序 从 rx-ring 中读取描述符。


- （2）4个这样的SPSC的ring
如果多个AF_XDP socket 对应同一个 Umem，那么 Umem的 Fill / Complete 就不是单生产者单消费者队列了，可能就需要加锁了??


- (3) AF_XDP socket 绑定到单个 rx 队列

由于性能原因，一般将 AF_XDP socket绑定到单个 rx queue；如果绑定到多个 queue，那么对于 AF_XDP socket 的 rx-ring 等队列操作 就不再是 SPSC 操作了，可能需要加锁了。

为了实现绑定到对个队列之后，应用程序可以收到包。那么就需要在网卡上配置规则进行过滤，将特定的流量导流到指定的 队列。

> 注：如果期望 网卡队列有多少个，就创建多少个 AF_XDP socket，这样尽可能的一对一 。



## AF_XDP接收数据包

![](attachments/Pasted%20image%2020240710154123.png)

AF_XDP接收数据包需要FILL RING，RX RING两个环形队列配合工作。

（1）第一步：XDP程序获取可用UMEM单元。
FILL RING记录了可以用来接收数据包的UMEM单元数量，用户程序根据UMEM使用记录，定期的往FILL RING生产可用UMEM单元。

（2）第二步：XDP填充新的接收数据包
XDP程序消费FILL RING中UMEM单元用于存放网络数据包，接收完数据包后，将UMEM单元和数据包长度重新打包，填充至RX RING队列，生产一个待接收的数据包。

（3）第三步：用户程序接收网络数据包
用户程序检测到RX RING有待接的收数据包，消费RX RING中数据包，将数据包信息从UMEM单元中拷贝至用户程序缓冲区，同时用户程序需要再次填充FILL RING队列推动XDP继续接收数据。


## AF_XDP 发送数据包

![](attachments/Pasted%20image%2020240710154244.png)

AF_XDP发送数据包需要COMP RING，TX RING两个环形队列配合工作。

（1）第一步：用户程序确保有足够的UMEM发送单元

COMP RING记录了已完成发送的数据包（UMEM单元）数量，用户程序需要回收这部分UMEM单元，确保有足够的UMEM发送单元。  

（2）第二步：用户程序发送数据包

用户程序申请一个可用的UMEM单元，将数据包拷贝至该UMEM单元，然后生产一个待发送数据包填充值TX RING。

（3）第三步：XDP发送数据包

XDP程序检测到TX RING中有待发送数据包，从TX RING消费一个数据包进行发送，发送完成后，将UMEM单元填充至COMP RING，生产一个已完成发送数据包，用户程序将对该数据包UMEM单元进行回收。


## 高性能 的AF_XDP程序

### 调优xdp程序

（1）网卡队列独占的 XDP bpf prog 和 BPF_MAP_TYPE_XSKMAP bpf map。

（2）使用类 DPDK 的 PMD 模式去设计用户态应用程序。
比如，SO_PREFER_BUSY_POLL类型的 socket option。
参考：[# Introduce preferred busy-polling](https://lwn.net/Articles/837010/)

其中 PMD 模式就涉及 CPU 亲和性和 NUMA 配置，为了内核处理逻辑、用户态应用程序处理逻辑都在同一个 CPU 上处理同一个网络包，使得网络包所在内存的 CPU 亲和性最大化（最大程度降低 CPU cache miss）。

（3）使用 `ethtool ntuple` 在网卡里配置 RSS 进行网络包分发。
合适的 `ethtool ntuple` 配置，会按需将网络包分发到对应的网卡 queue 中；比如，同一个源地址的网络包只会分发到同一个 queue 中。

（4）用户态的AF_XDP程序和内核态的xdp公用一个CPU core的时候
内核中的XDP，可以通过配置网卡的FDIR规则，将指定的流量分到某个队列queue 或 queue group。然后队列存在中断，将收包中断绑定到某个CPU上。
用户态的程序，也可以通过 taskset 将程序绑定到某个CPU上。如果是DPDK AF_XDP程序，可以通过`-l` 来自定 cpu core。



## 其他

### 内核是如何为每个接收的网络包选择 `receive queue` 的

（1）类型为 BPF_MAP_TYPE_XSKMAP 的 bpf map
该 bpf map 是一个数组，数组中的每个槽位都是 AF_XDP socket 对应的文件描述符。绑定了 UMEM 的进程通过 `bpf()` 系统调用将文件描述符保存到 bpf map 中；当然，实际上保存的是应用程序不可见的内核内部指针。

（2）加载到驱动中的 BPF 程序
BPF 程序用于分类进来的网络包，并且收包的队列信息查 bpf map, 将网络包放入到所选的 bpf map 条目对应 AF_XDP socket 的 rx-ring 描述符对应的空间中。没有 bpf map 和 BPF 程序的话，AF_XDP socket 就无法接收网络包。

### AF_XDP socket 绑定到指定的网卡

通过 `bind()` 系统调用将 socket 绑定到指定的网卡，且可能是网卡里的某个硬件网络包队列。网卡本身就可配置成将网络包送到那队列，如果 AF_XDP socket 背后的程序可以处理。

理解：
将 AF_XDP绑定到指定的网卡或网卡队列，那么这个数据包到达网卡或网卡队列后，DMA到 指定的ring-buffer 对应的空间之后。bgp程序（XDP bpf prog）主要的作用就是用于分流：判断哪些网络包要 redirect 到 UMEM，其它网络包直接 PASS。

当要将网络包 redirect 到 UMEM 时，需要调用 `bpf_redirect_map()` 帮助函数（helper func）进行 redirect。

### AF_XDP socket 可以使用 epoll/poll/select 么？

XDP Scoket也是一个文件描述符，因此可以通过poll/epoll/select来等待IO事件，需要说明的是：收/发的数据包是原始的以太网帧，因此在包处理上要麻烦一些。

## AF_XDP的范例
### 通过BPF将网络数据包从XDP Hook点旁路到用户态的XDP Socket

参考：[advanced03-AF_XDP](https://github.com/xdp-project/xdp-tutorial/tree/master/advanced03-AF_XDP)

#### 驱动xdp程序
advanced03-AF_XDP/af_xdp_kern.c 文件如下所示：
```c
// bpf/bpf_helpers.h 中

#define __uint(name, val) int (*name)[val]
#define __type(name, val) typeof(val) *name
/*
 * Helper macro to place programs, maps, license in
 * different sections in elf_bpf file. Section names
 * are interpreted by elf_bpf loader
 */
#define SEC(NAME) __attribute__((section(NAME), used))


/*
 * Helper structure used by eBPF C program
 * to describe BPF map attributes to libbpf loader
 */
struct bpf_map_def {
    unsigned int type;
    unsigned int key_size;
    unsigned int value_size;
    unsigned int max_entries;
    unsigned int map_flags;
};


/* user accessible metadata for XDP packet hook
 * new fields must be added to the end of this structure
 */
struct xdp_md { // xdp metadata 
    __u32 data;
    __u32 data_end;
    __u32 data_meta;
    /* Below access go through struct xdp_rxq_info */
    __u32 ingress_ifindex; /* rxq->dev->ifindex */
    __u32 rx_queue_index;  /* rxq->queue_index  */
};
```


```c
struct bpf_map_def SEC("maps") xsks_map = {
    .type = BPF_MAP_TYPE_XSKMAP,
    .key_size = sizeof(int),
    .value_size = sizeof(int),
    .max_entries = 64,  /* Assume netdev has no more than 64 queues */
};

SEC("xdp_sock")
int xdp_sock_prog(struct xdp_md *ctx)
{
    int index = ctx->rx_queue_index;

    if (bpf_map_lookup_elem(&xsks_map, &index))
        return bpf_redirect_map(&xsks_map, index, 0);
    
    return XDP_PASS;
}
```

程序解析：
(1) `struct {} xsks_map SEC(".maps");`： 定义了一个`BPF_MAP_TYPE_XSKMAP`类型的映射表，当采用SEC("maps")方式来显示定义时，将在生成的bpf目标文件的ELF格式中看到相关描述。
> 当BPF程序被加载到内核时，会自动创建名为“xsks_map”的描述符， 用户态可通过查找“xsks_map”来获取该map的描述符，这样用户态和内核BPF程序就可以共同访问该map

(2) `type = BPF_MAP_TYPE_XSKMAP`：指定该map的类型，它与bpf_redirect_map() 结合使用以将收到的帧传递到指定套接字。

(3) `key_size = sizeof(int)，value_size = sizeof(int)`：指定key，value长度;
针对以上key，value需要说明一下：对于`BPF_MAP_TYPE_XSKMAP`类型的map，value必须是XDP socket描述符，key必须是int类型，原因在于bpf_redirect_map()的第二个参数，参见下面。

(4)  `max_entries = 64`：指定map最多存储64个元素

(5) `SEC("xdp_sock")`：指定prog函数符号，应用层可通过查找"xdp_sock"加载该prog，并绑定到指定网卡.

(6) `int xdp_sock_prog(struct xdp_md *ctx)`：当网卡收到数据包时，会在xdp hook点调用该函数。

(7) `int index = ctx->rx_queue_index`： 获取该数据包来自网卡到哪个rx队列ID，ctx有许多成员，比如：网卡ID，数据帧等等

(8) `if (bpf_map_lookup_elem(&xsks_map, &index))`： 判断xsks_map是否存在key为index（即rx队列号）的数据，注意，这里实际上就是判断该网卡队列是否绑定了xdp Socket。

(9) `bpf_redirect_map(&xsks_map, index, 0)`：`bpf_redirect_map`函数作用就是重定向，比如：将数据重定向到某个网卡，CPU， AF XDP Socket等等；
当`bpf_redirect_map`函数的第一个参数的map类型为`BPF_MAP_TYPE_XSKMAP`时，则表示将数据重定向到XDP Scoket。

（10）`bpf_redirect_map（）`会查找参数1即xsks_map 中 key为index 的 value 是否存在，若存在，则检查value是否是一个XDP Scoket，并且是否绑定到了该网卡（可以绑定到任意有效队列）

#### 用户程序

用户态程序 af_xdp_user.c
> 该程序实现bpf加载到网卡
> 创建XDP Scoket并绑定到网卡的指定队列，并通过XDP Scoket收发数据；
> 这里仅分析xXDP Scoket相关部分

```c
int main(int argc, char **argv)
{
    ...
    bpf_obj = load_bpf_and_xdp_attach(&cfg);
    map = bpf_object__find_map_by_name(bpf_obj, "xsks_map");
    ...
    xsks_map_fd = bpf_map__fd(map);
    ...
    umem = configure_xsk_umem(packet_buffer, packet_buffer_size);
    ...
    xsk_socket = xsk_configure_socket(&cfg, umem);
    ...
    rx_and_process(&cfg, xsk_socket);
    ...
}

static struct xsk_socket_info *xsk_configure_socket(struct config *cfg,
                            struct xsk_umem_info *umem)
{
    ...
    ret = xsk_socket__create(&xsk_info->xsk, cfg->ifname,
                 cfg->xsk_if_queue, umem->umem, &xsk_info->rx,
                 &xsk_info->tx, &xsk_cfg);
    ...
}


static void rx_and_process(struct config *cfg,
               struct xsk_socket_info *xsk_socket)
{
    struct pollfd fds[2];
    int ret, nfds = 1;

    memset(fds, 0, sizeof(fds));
    fds[0].fd = xsk_socket__fd(xsk_socket->xsk);
    fds[0].events = POLLIN;
    
    while(!global_exit) {
        if (cfg->xsk_poll_mode) {
            ret = poll(fds, nfds, -1);
            if (ret <= 0 || ret > 1)
                continue;
        }
        handle_receive_packets(xsk_socket);
    }
}
```
程序解析：

（1） `bpf_obj = load_bpf_and_xdp_attach(&cfg)`: 加载bpf程序，并绑定到网卡；

（2） `map = bpf_object__find_map_by_name(bpf_obj, "xsks_map")`： 查找bpf程序内定义的xsks_map；

（3） `umem = configure_xsk_umem(packet_buffer, packet_buffer_size)`： 为XDP Scoket准备UMEM；

（4） `xsk_configure_socket()`通过调用bpf helper函数xsk_socket__create（）创建XDP Scoket并绑定到cfg->ifname网卡的cfg->xsk_if_queue队列，默认情况下将该【cfg->xsk_if_queue， xsk_info->xsk fd】添加到xsks_map, 这样bpf程序就可以重定向到该XDP Scoket, 除非指定XSK_LIBBPF_FLAGS__INHIBIT_PROG_LOAD标志。




# libxdp/libbpf介绍

## 特性

### XDP的copy模式和zero-copy模式

![](attachments/Pasted%20image%2020240716163247.png)

#### copy模式

DMA将报从网卡传送到ring-buffer的描述符指定的内存中，此内存不是xdp的缓冲区(UMEM)。 从ring-buffer的描述符指定的内存需要拷贝数据帧frame到xdp的缓冲区(UMEM)。


#### zero-copy模式
对于 AF_XDP“零复制”的支持，**需要网卡的驱动程序支持，并且公开注册的 API，以便直接在 NIC RX ring 结构中注册和使用 UMEM 区域进行 DMA 传输**。

#### 使用

在具有“zero-copy”功能的驱动程序上使用“copy复制模式”仍然有意义。比如，并非 RX 队列上的所有流量都用于 AF_XDP 套接字， XDP 程序基于规则在 XDP_REDIRECT 和 XDP_PASS之间进行分流 ，那么就需要使用“复制模式”。

如果此时使用 zero-copy模式，就需要为 XDP_PASS的流量 申请内容，并且拷贝数据帧到这样的空间。
> 注：测试了一下，在Intel E810网卡上，使用 zero-copy模式，进行流分叉，是可以的。没有什么问题。

```bash
对于 dpdk 的testpmd 而言，设置为：
	--vdev net_af_xdp,iface=ens786f1,force_copy=1
默认为 zero-copy模式，如果 copy模式，则需要强制设置。

```

对应底层为：
```c
// /usr/include/linux/if_xdp.h

/* Options for the sxdp_flags field */
#define XDP_SHARED_UMEM (1 << 0)
#define XDP_COPY        (1 << 1) /* Force copy-mode */
#define XDP_ZEROCOPY    (1 << 2) /* Force zero-copy mode */
/* If this option is set, the driver might go sleep and in that case
 * the XDP_RING_NEED_WAKEUP flag in the fill and/or Tx rings will be
 * set. If it is set, the application need to explicitly wake up the
 * driver with a poll() (Rx and Tx) or sendto() (Tx only). If you are
 * running the driver and the application on the same core, you should
 * use this option so that the kernel will yield to the user space
 * application.
 */
#define XDP_USE_NEED_WAKEUP (1 << 3)
```

#### 不同网卡的不同特性

intel E810 25G网卡（ice驱动）的所有队列都是支持zc这个特性的。但是Mellanox Cx4-Lx 25G（mlx5-core）驱动不是所有队列支持zc，只有后半部分队列支持zc。

参考：[af-xdp zc 问题](https://lore.kernel.org/xdp-newbies/51ddb56f-5155-aabd-19b3-1bae187009ac@cesnet.cz/T/)

[# AF_XDP infrastructure improvements and mlx5e support](https://patchwork.ozlabs.org/project/netdev/cover/20190612155605.22450-1-maximmi@mellanox.com/)

![](attachments/Pasted%20image%2020240806172544.png)

![](attachments/image%20(10).png)
```text
如下所示：intel的所有队列支持zc。Mellanox的部分队列支持ZC。

Let me expand a bit on why I'm asking these qid questions.

It's unfortunate that vendors have different view/mapping on
"qids". For Intel, we allow to bind a zero-copy socket to all Rx
qids. For Mellanox, a certain set of qids are allowed for zero-copy
sockets.

This highlights a need for a better abstraction for queues than "some
queue id from ethtool". This will take some time, and I think that we
have to accept for now that we'll have different behavior/mapping for
zero-copy sockets on different NICs.

Let's address this need for a better queue abstraction, but that
shouldn't block this series IMO. Other than patch:

"[PATCH bpf-next v4 07/17] libbpf: Support drivers with non-combined channels"

which I'd like to see a bit more discussion on, I'm OK with this
series. I haven't been able to test it (no hardware "hint, hint"), but
I know Jonathan has been running it.
```


#### 注意

内核版本需要 >= 5.4 才可以使用 驱动的 zero-copy, zero-copy 使能的话，一般需要和 ring need-wakeup 一起配合使用。

是否开启zero-copy， 可以通过接口来判断。

![](attachments/Pasted%20image%2020240723153353.png)




### xdp的 need-wakeup 特性

默认情况下，驱动程序(作为消费者)将主动检查 TX 和 FILL 环以查看是否需要完成工作。
通过在bind xdp socket的时候设置 XDP_USE_NEED_WAKEUP 标记，就是告诉驱动
不需要主动去检查 tx/fill ring是否准备好，而是依赖用户进程通过系统调用去触发。

need-wakeup 特性 更多的是在 用户态程序和内核程序在一个core时使用。

#### 使用

![](attachments/Pasted%20image%2020240723152655.png)

无论内核 驱动和用户态程序是否使用同一个core，都是建议开启 NEED_WAKEUP的，尤其是当内核驱动和用户程序在同一个core的时候；

开启 NEED_WAKEUP 应该是可以带来更好的性能。内核驱动和用户程序不在同一个core时，可以较少发包的时候的系统调用。

#### 限制

 need-wakeup 特性是在**内核 5.4** 及其以后才支持的。如果是内核版本>=5.4，那么DPDK AF_XDP自动使能了 need-wakeup。
```bash
https://github.com/torvalds/linux/blob/v5.4/include/uapi/linux/if_xdp.h
```

![](attachments/Pasted%20image%2020240723175052.png)

### preferred busy polling
参见：[# Introduce preferred busy-polling](https://lwn.net/Articles/837010/)

preferred busy polling 即 busy-polling 优先；busy-polling 特性 更多的是 为了提升高负载下af-xdp的单核性能（即 一个core 一方面做应用层的处理，一方面中断收包处理）。**内核5.11**才支持 af-xdp的 busy-polling 特性。
> 注：need-wakeup 特性是在**内核 5.4** 及其以后支持的。

#### 背景
##### 正常的NAPI

![](attachments/Pasted%20image%2020240724111334.png)

以前的网络设备驱动程序架构已经不能适用于每秒产生数千个中断的高速网络设备，并且它可能导致整个系统处于饥饿状态（译者注：饥饿状态的意思是系统忙于处理中断程序，没有时间执行其他程序）。有些网络设备具有中断合并，或者将多个数据包组合在一起来减少中断请求这种高级功能。

在内核没有使用NAPI来支持这些高级特性之前，这些功能只能全部在设备驱动程序中结合抢占机制（例如基于定时器中断），甚至中断程序范围之外的轮询程序（例如：内核线程，tasklet等）中实现。

 NAPI(New API)就是混合中断和轮询(busy polling)的方式来收包，当有中断来了，驱动关闭中断，通知内核收包，内核软中断轮询当前网卡，在规定时间尽可能多的收包。时间用尽或者没有数据可收，内核再次开启中断，准备下一次收包。

> NAPI 对数据包到达的事件的处理采用轮询方法（即**中断驱动或事件驱动 + 轮询处理**），在数据包达到的时候，NAPI 就会强制执行`dev->poll`方法。而和不像以前的驱动那样为了减少包到达时间的处理延迟，通常采用中断的方法来进行。

NAPI 是 Linux 上采用的一种提高网络处理效率的技术，它的核心概念就是不采用中断的方式读取数据，而代之以首先采用中断唤醒数据接收的服务程序，然后 POLL 的方法来轮询数据。随着网络的接收速度的增加，NIC 触发的中断能做到不断减少。

目前 NAPI 技术已经在网卡驱动层和网络层得到了广泛的应用，驱动层次上已经有 E1000 系列网卡，RTL8139 系列网卡，3c50X 系列等主流的网络适配器都采用了这个技术；而在网络层次上，NAPI 技术已经完全被应用到了著名的`netif_rx `函数中间，并且提供了专门的 POLL 方法--`process_backlog` 来处理轮询的方法；根据实验数据表明采用NAPI技术可以大大改善短长度数据包接收的效率，减少中断触发的时间。


##### NAPI 流程介绍

![](attachments/Pasted%20image%2020240724111645.png)

基于NAPI接口， 一般的网络传输(接收)有如下几个步骤：

- 网络设备驱动加载与初始化（配置IP等）
- 数据包从网络侧发送到网卡(`Network Interface Controller`, NIC)
- 通过DMA(Direct Memory Access)s)，将数据从网卡拷贝到内存的环形缓冲区(ring buffer)
- 数据从网卡拷贝到内存后, NIC产生硬件中断告知内核有新的数据包达到
- 内核收到中断后, 调用相应中断处理函数, 此时就会调用NAPI接口`__napi_schedule`开启poll线程（实际是触发一个软中断`NET_RX_SOFTIRQ`）(常规数据传输, 一般在处理NIC的中断时调用`netif_rx_action`处理网卡队列的数据）
- `ksoftirqd`（每个CPU上都会启动一个软中断处理线程）收到软中断后被唤醒, 然后执行函数`net_rx_action`, 这个函数负责调用NAPI的`poll`接口来获取内存环形缓冲区的数据包
- 解除网卡`ring buffer`中的DMA内存映射(unmapped), 数据由CPU负责处理, `netif_receive_skb`传递回内核协议栈
- 如果内核支持数据包定向分发(`packet steering`)或者NIC本身支持多个接收队列的话, 从网卡过来的数据会在不同的CPU之间进行均衡, 这样可以获得更高的网络速率
- 网络协议栈处理数据包，并将其发送到对应应用的`socket`接收缓冲区



##### NAPI缺陷

对于上层的应用程序而言，**系统不能在每个数据包接收到的时候都可以及时地去处理它**，而且随着传输速度增加，累计的数据包将会耗费大量的内存。

#### af_xdp socket支持busy polling

参考： [# [RFC,bpf-next,0/7] busy poll support for AF_XDP sockets](https://patchwork.ozlabs.org/project/netdev/cover/1556786363-28743-1-git-send-email-magnus.karlsson@intel.com/)

[**AF_XDP new prefer busy poll**](https://lore.kernel.org/xdp-newbies/2eefacdbbee1bac291abbdfffb40b09d58c21831.camel@coverfire.com/T/)

通过 busy-poll, 可以在 af-xdp socket的应用程序的上下文中通过系统调用(比如：poll、recvmsg/sendmsg、read/write) 就触发了驱动的执行。

这样做的好处是，应用程序的执行和驱动的执行可以发生在一个core上，可以减少core和core之间cache切换。如果没有busy-poll，应用程序和驱动的执行可能在不同core上，就需要core和core之间的cache切换。


> 即：通过 busy-poll， af-xdp应用程序可以通过系统调用触发xdp驱动代码的执行，并且应用程序和驱动代码可以在同一个core上执行。

##### 优缺点
**（1）优点**：
从系统的角度看，不需要过多的core去进行软中断ksoftirqd/softirq的处理，因为所有的处理都可以在应用所在的core上进行。即应用的线程和驱动在同一个core上。

![](attachments/Pasted%20image%2020240730215525.png)


1> 更简单的去配置应用程序和驱动程序都在同一核心上高效运行。
busy-polling模式下，应用程序只需要一个系统调用即可。

2> 从每个core的角度看，它更快，因为我们没有任何核心之间的通信。
内核和应用程序之间的所有头部和描述符传输都是在同一核心上进行的，这要快得多；可伸缩性也会更好。
例如，64字节描述符 + 64字节数据包头部 ，那么 每个数据包减少在核心之间的互连上的流量，可以节省128字节。在每个核心上达到20Mpps时，这约等于20Gbit/s（`20M*128*8bit`），如果存在20个核心时，这将减少约400Gbit/s的互连流量。

> 注：如果从单个应用程序的角度来看，core足够的话，可能驱动处理和应用处理在不同的core上，应用程序的吞吐更高。

3> 它为无缝替换DPDK中的用户空间驱动程序提供了一种方式，改用内核空间中的Linux驱动程序。
DPDK的模型是应用程序和驱动程序都在同一核心上运行，因为它们都在用户空间。在af_xdp上，如果我们可以提供相同的模型（都在同一核心上高效运行，而不是在用户空间中的驱动程序），那么DPDK用户很容易进行切换。比如，如果系统构建者在他的设备中有12个core，并且有12个DPDK应用程序实例，每个核心一个DPDK程序；那么他/她在使用AF_XDP时，重新分配应用程序和驱动程序核心时会如何推理？8个应用程序核心和4个驱动程序核心，或者各6个？也许还与数据包相关？
如果我们有一种有效的方法在同一核心上运行它们两者，迁移就会简单得多。

**（2）缺点**：
  
busy-poll 的缺点是 从单个应用程序角度看到的最大吞吐量将会较低（由于系统调用），但在从每个核心的角度上，它通常会更高，因为正常模式在两个核心上运行（一个core上驱动程序redirect到xdp socket，一个core是在应用层进行处理），忙轮询在单个核心上运行。


##### 实现机制

对于接受测而言，存在busy-polling后，af-xdp应用程序，发现rx ring中无法获取到包，那么应用程序就可以进行系统调用(比如：poll、recvmsg/sendmsg、read/write) -----> 进入到 netdev 的 napi poll 机制（此时napi 不是中断触发，而是系统调用触发，即减少中断的发生）-----> 存在数据包时，则将包 重定向到 xdp socket ring。

**busy-polling 模式下是否存在中断？**
存在中断，但是一开始启动的时候，中断被disable了。我们希望尽量避免中断，因为当它们触发时，我们将回到非忙碌轮询（非 busy-polling）模型，即应用程序和驱动代码在两个单独的核心上处理，上述优势将消失。
那么，如何实现尽可能的避免中断呢？
一种方法是使用我们在af-xdp socket 绑定的队列上创建一个没有关联中断或已禁用中断的 napi 上下文。在这种情况下，af-xdp 套接字只能在调用 poll() 系统调用时接收和发送数据包。如果不调用 poll()，就不会收到任何数据包，也不会发送任何数据包。也就是，将poll() 的超时值设置为0，这将具有最佳性能。
在某个时刻, 可以通过设置poll的超时时间>0 来重新启动中断。在应用程序调用 poll() 时，其所在的core将不断的调用禁用了中断的 napi 上下文，直到找到数据包；如果一直没有包，也最多持续一段时间，直到忙碌轮询超时（就像今天的常规忙碌轮询一样）。如果poll超时，我们将进入 poll() 调用的常规超时处理，启用与 napi 关联的队列的中断，并将进程置于睡眠状态。一旦被中断唤醒（即收到了数据包），napi 的中断将再次被禁用，并将控制返回给应用程序。
通过这种方案，我们将在具有中断禁用的core本地（即：af-xdp应用和xdp驱动程序在一个core上）上处理绝大多数数据包，只有在负载较低且在 poll 处于休眠/等待状态时，我们才会使用与中断绑定的核心上，通过中断处理一些数据包。

**busy-polling 模式下驱动代码是否会starve? **

在 busy-polling 模式下，应用程序通过系统调用来执行驱动代码，那么是否意味着驱动会被用户程序给饿死（starve）呢？

驱动程序不会饿死，通过 如下的 time out  以及  defer_count 机制，可以确保驱动程序不会被饿死。
```bash
echo 2 | sudo tee /sys/class/net/ens785f1/napi_defer_hard_irqs
echo 200000 | sudo tee /sys/class/net/ens785f1/gro_flush_timeout

注：如果为了更好的时延，可能对于这2个参数需要进行调整。但是时延和吞吐可能是互斥的。
```
即：busy-polling 模式下，如果应用程序未能在指定的超时时间内（the watchdog timer defined by gro_flush_timeout）轮询(即没有执行系统调用)，内核将在超时的时候进行驱动程序轮询，在 `defer count`次 都没有包的情况下，硬件中断将会被使能。这将确保驱动代码不会被用户态starve。
```bash
If the application fails to poll within the specified timeout,
the kernel will do driver polling at a pace of the timeout, and if
there are no packets after "defer count" times, interrupts will be
enabled. This to ensure that the driver is not starved by userland.
```

**系统调用如何选择**
recvmsg/sendmsg 以及 poll（poll比较全，可以用于接受侧和发送侧），但是poll的开销会比 send/recv 大一些。
> 注：应该是对于内核之前的poll进行了扩展，支持af-xdp socket。
Busy-poll 模式下，如果 fill ring 为空(接收侧)，或 complete ring (发送侧)满了， 调用poll 进行驱动的rx/tx，则会返回失败（POLLERR）；

```bash
recvmsg/sendmsg and poll (which means read/recvfrom/recvmsg, and
corresponding on the write side). Use readfrom for rx queues, and
sendto for tx queues.  Poll works as well, but the overhead for poll
is larger than send/recv.
```

**af_xdp的busy-poll 和 af_inet socket的 busy-polling**

![](attachments/Pasted%20image%2020240730152951.png)

在使用普通的`poll()`函数处理`AF_XDP`套接字时，应用程序会在一个核心上运行，而驱动程序处理会在另一个核心上以`softirq/ksoftirqd`上下文中运行（比如系统调度、irqbalance的原因）。在这种模式（busy-polling）下，当af-xdp应用程序执行`poll`系统调用时，`napi`上下文会从系统调用上下文中调用。

当然我们也可以通过设置中断绑定到某个core，以及应用程序的线程绑定到该core(taskset或者代码写死)，强行将应用程序和中断的驱动绑定到一个core上执行。但由于两者之间的上下文切换，这并不是那么高效。更有效的方法是在内核中调用`napi`循环，因为在执行`poll()`系统调用时，你已经在内核中。这就是经典的在`AF_INET`套接字上操作的忙碌轮询机制所做的事情。在`poll()`调用内部，它执行驱动程序的`napi`上下文，直到找到一个数据包（如果是接收侧的话），然后返回给应用程序，然后应用程序处理数据包。我想为`AF_XDP`套接字采用类似的机制（当然af-xdp socket和 af-inet socket也存在一些差异）。

从API的角度来看，使用AF_XDP套接字的繁忙轮询（busy-poll）方式，用户会将af-xdp socket绑定到一个接收队列，并将其应用程序设置为在特定核心core上执行，然后应用程序和驱动程序的执行都将仅在该核心上进行。在我看来，这比使用常规轮询或当前状态下的 AF_XDP（即：在接收端不使用系统调用）更简单，因为后者需要设置中断的cpu亲和性以及af-xdp应用程序的cpu亲和性。

**af-xdp socket的busy-poll模式理解***
此中的 busy-poll 设置（busy-budeget 设置）是 per af-xdp socket的，那么也只有属于这个af-xdp socket的流量（一般是某个网卡的队列收到的流量）才会 进行 busy-poll。


#### SO_BUSY_POLL 和 SO_PREFER_BUSY_POLL






#### 配置
![](attachments/Pasted%20image%2020240723155128.png)

busy-polling 特性在内核版本>=5.11时，默认是开启的。busy-polling 可以提升 高负载下 AF_XDP的单核性能（即 一个core 一方面做应用层的处理，一方面中断收包处理）。
DPDK的 `busy_budget` 参数（默认值为64）来设置 NAPI 上下文中，内核一次最多可以取多少个包进行处理。 如果 `busy_budget` 配置为0，意味着 DPDK 应用禁用了 busy polling 模式。
如果使用了 busy-polling ，最好还要设置一些其他的系统参数。比如：
```bash
echo 2 | sudo tee /sys/class/net/ens786f1/napi_defer_hard_irqs
echo 200000 | sudo tee /sys/class/net/ens786f1/gro_flush_timeout

```

DPDK 使用 busy-polling 配置如下：
```bash
1. Set the hardware queues:
    
    ethtool -L eth0 combined 1
    
2. Configure busy polling settings:
    
    echo 0 >> /sys/class/net/eth0/napi_defer_hard_irqs
    echo 0 >> /sys/class/net/eth0/gro_flush_timeout
    
3. Start testpmd:
    
    ./x86_64-native-linuxapp-gcc/app/dpdk-testpmd --no-pci \
    --vdev=net_af_xdp0,iface=eth0,busy_budget=0 \
    --log-level=pmd.net.af_xdp,debug \
    --forward-mode=macswap
```

![](attachments/Pasted%20image%2020240723155451.png)

#### 限制

busy-polling 特性是在内核 5.11 及其以后才支持的。

#### 应用

我的理解是：使用busy-polling的话，会通过系统调用的方式来触发Napi，减少收包是的中断/软中断的次数。

如果af-xdp应用程序所在的core和中断收包的core不同，使用busy-polling，那么应用程序所在的core的 sys 的cpu占比会提高；中断收包的core中的 hi、si 比例会降低。
> 因此，如果core足够多的话，就不要开启 busy-polling了，这样 af-xdp应用程序 可以减少系统调用，即减少 af-xdp应用程序所在core的压力；收包的core会多了一些收包的软中断。

如果 af-xdp 应用程序所在的core和中断收包的core相同，使用busy-polling，

### umem的chunk大小

#### XDP_UMEM_UNALIGNED_CHUNK_FLAG
![](attachments/Pasted%20image%2020240723170016.png)

默认情况下，mem 的 ring的大小需要是2048和page_size之间的2的N次方；
这个就比较受到限制了，比如 chunk大小为2k，那么多个chunk的起始地址只能是0,2k,4k,6k,8k...等等。如果存在跨越页边界，可能存在较大的浪费。

如果在 `struct xdp_umem_reg>flags`中设置了 XDP_UMEM_UNALIGNED_CHUNK_FLAG 标记 ，那么可以打破限制，比如设置为 3000；

注：`XDP_UMEM_UNALIGNED_CHUNK_FLAG` 是内核中定义的，在 `include/uapi/linux/if_xdp.h`中定义，应该只有高版本的内核才存在该定义。
```c
/* Flags for xsk_umem_config flags */
#define XDP_UMEM_UNALIGNED_CHUNK_FLAG (1 << 0)


参见：https://github.com/torvalds/linux/blob/v5.4/include/uapi/linux/if_xdp.h

内核的 5.4版本中存在 XDP_UMEM_UNALIGNED_CHUNK_FLAG，5.3中不存在。
```


对于DPDK的 AF_XDP而言，通过查看是否定义了 `XDP_UMEM_UNALIGNED_CHUNK_FLAG` 来决定是否启动 umem和 af_xdp pmd之间的 ZC（即 umem 和dpdk程序之间的 zc）。

因为，正常情况下，xdp驱动程序传到用户态程序的是 原始的 frame，使用的是umem的内存空间。在dpdk中使用的 mempool 的mbuf 来进行数据包的组织，这2个空间可能不是一个空间，那么就需要进行拷贝，如果使用同一个空间，那么就不需要进行拷贝了。

![](attachments/Pasted%20image%2020240723171805.png)


##### 区分
需要对 umem 和 pmd 之间的 ZC (称之为 PMD的 zero copy)和 网卡 DMA 到 UMEM 进行区分。

如果网卡 DMA 到 网卡自身的 ring-buffer 对应的描述符的空间，那么从 ring-buffer 描述符的空间到 UMEM 应该还有一个拷贝。

对于testpmd而言：
umem 和 pmd 之间的 ZC 是根据内核版本决定的，主要是编译的时候（保证：编译机和运行机器的内核版本一致）；
网卡 DMA 到 网卡自身的 ring-buffer 对应的描述符的空间是否是unem的空间，则是 通过启动参数(force_copy)决定的。默认是进行ZC的。
```bash
--vdev net_af_xdp,iface=ens786f1,force_copy=1

注：force_copy 是 eal的 参数（af pmd的参数），而不是 testpmd的参数。 
```


### Multi-Buffer 巨帧支持

一个巨帧，可能需要占用多个 frame/buffer, 比如一个9K的巨帧，每个frame/chunk大小是4k，那么就需要占用3个。

详细参见：[AF_XDP](https://www.kernel.org/doc/Documentation/networking/af_xdp.rst), 这里就不仔细介绍了。

## 结构体

## 函数实现

# AF_XDP 用法
## 用户态程序
### 创建AF_XDP的socket
```c
xsk_fd = socket(AF_XDP, SOCK_RAW, 0);
```
### 为UMEM申请内存
UMEM是一块包含固定大小chunk的内存，我们可以通过malloc/mmap/hugepages申请。

```c
// linux 内核的 sample/bpf/xdpsock_user.c

    /* Reserve memory for the umem. Use hugepages if unaligned chunk mode */
    void * bufs = mmap(NULL, NUM_FRAMES * opt_xsk_frame_size,
            PROT_READ | PROT_WRITE,
            MAP_PRIVATE | MAP_ANONYMOUS | opt_mmap_flags, -1, 0);
    if (bufs == MAP_FAILED) {
        printf("ERROR: mmap failed\n");
        exit(EXIT_FAILURE);
    }
```

### 向AF_XDP socket注册UMEM

其中xdp_umem_reg结构定义如下：
```c
// tools/include/uapi/linux/if_xdp.h

struct xdp_umem_reg {
    __u64 addr; /* Start of packet data area */
    __u64 len; /* Length of packet data area */
    __u32 chunk_size;
    __u32 headroom;
    __u32 flags;
};

// tools/lib/bpf/xsk.h
struct xsk_umem_config {
    __u32 fill_size;  // fill ring的大小；默认2048
    __u32 comp_size;  // complete ring的大小; 默认2048
    __u32 frame_size; // 帧大小, 默认4096
    __u32 frame_headroom; // 默认0;
    __u32 flags; // 默认0；
};

```
**xdp_umem_reg成员解析：**
- addr就是UMEM内存的起始地址；
- len是整个UMEM内存的总长度；
- chunk_size就是每个chunk的大小；
- headroom，如果设置了，那么报文数据将不是从每个chunk的起始地址开始存储，而是要预留出headroom大小的内存，再开始存储报文数据，headroom在隧道网络中非常常见，方便封装外层头部；
- flags, UMEM还有一些更复杂的用法，通过flag设置，后面再进一步展开；比如，设置了XDP_UMEM_UNALIGNED_CHUNK_FLAG，那么 umem可以考虑

```c
    memset(&mr, 0, sizeof(mr));
    mr.addr = (uintptr_t)umem_area;
    mr.len = size;
    mr.chunk_size = umem->config.frame_size;
    mr.headroom = umem->config.frame_headroom;
    mr.flags = umem->config.flags;

	// umem->fd, 即上面创建的 xsk_fd;
    err = setsockopt(umem->fd, SOL_XDP, XDP_UMEM_REG, &mr, sizeof(mr));
    if (err) {
        err = -errno;
        goto out_socket;
    }
```



### 创建FILL RING 和 COMPLETION RING

我们通过 setsockopt() 设置 FILL/COMPLETION/RX/TX ring的大小（在我看来这个过程相当于创建，不设置大小的ring是没有办法使用的）。

FILL RING 和 COMPLETION RING是UMEM必须，RX和TX则是 AF_XDP socket二选一的，例如AF_XDP socket只收包那么只需要设置RX RING的大小即可。

```c
    err = setsockopt(fd, SOL_XDP, XDP_UMEM_FILL_RING,
             &umem->config.fill_size,
             sizeof(umem->config.fill_size));
    if (err)
        return -errno;

    err = setsockopt(fd, SOL_XDP, XDP_UMEM_COMPLETION_RING,
             &umem->config.comp_size,
             sizeof(umem->config.comp_size));
    if (err)
        return -errno;
```
上述操作相当于创建了 FILL RING 和 和 COMPLETION RING，创建ring的过程主要是初始化 producer 和 consumer 的下标，以及创建ring数组。


### 将FILL RING 映射到用户态

上文提到，用户态程序是 FILL RING 的生产者和 CONPLETION RING 的消费者，上面2个 ring 的创建是在内核中创建了 ring 并初始化了其相关成员。那么用户态程序如何操作这两个位于内核中的 ring 呢？所以接下来我们需要将整个 ring 映射到用户态空间。

第一步是获取内核中ring结构各成员的偏移，因为从5.4版本开始后，ring结构中除了 producer、consumer、desc外，又新增了一个flag成员。所以用户态程序需要先获取 ring 结构中各成员的准确偏移位置，才能在mmap() 之后准确识别内存中各成员位置。

```c

// tools/include/uapi/linux/if_xdp.h
struct xdp_ring_offset {
    __u64 producer;
    __u64 consumer;
    __u64 desc;
    __u64 flags;
};

struct xdp_mmap_offsets {
    struct xdp_ring_offset rx;
    struct xdp_ring_offset tx;
    struct xdp_ring_offset fr; /* Fill */
    struct xdp_ring_offset cr; /* Completion */
};


// net/xdp/xsk.h
struct xdp_ring_offset_v1 {
    __u64 producer;
    __u64 consumer;
    __u64 desc;
};

struct xdp_mmap_offsets_v1 {
    struct xdp_ring_offset_v1 rx;
    struct xdp_ring_offset_v1 tx;
    struct xdp_ring_offset_v1 fr;
    struct xdp_ring_offset_v1 cr;
};


// tools/lib/bpf/xsk.c
static int xsk_get_mmap_offsets(int fd, struct xdp_mmap_offsets *off)
{
    socklen_t optlen;
    int err;

    optlen = sizeof(*off);
    // fd 即 创建的 af-xdp socket fd;
    err = getsockopt(fd, SOL_XDP, XDP_MMAP_OFFSETS, off, &optlen);
    if (err)
        return err;

    if (optlen == sizeof(*off))
        return 0;

    if (optlen == sizeof(struct xdp_mmap_offsets_v1)) {
        xsk_mmap_offsets_v1(off);
        return 0;
    }

    return -EINVAL;
}
```


一切就绪，开始将内核中的 FILL RING 映射到用户态程序中：
```c

// tools/lib/bpf/xsk.h
#define DEFINE_XSK_RING(name) \
struct name { \
    __u32 cached_prod; \
    __u32 cached_cons; \
    __u32 mask; \
    __u32 size; \
    __u32 *producer; \
    __u32 *consumer; \
    void *ring; \
    __u32 *flags; \
}

DEFINE_XSK_RING(xsk_ring_prod);
DEFINE_XSK_RING(xsk_ring_cons);


// tools/lib/bpf/xsk.c 中的 xsk_create_umem_rings

    map = mmap(NULL, off.fr.desc + umem->config.fill_size * sizeof(__u64),
           PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE, fd,
           XDP_UMEM_PGOFF_FILL_RING); // 此中的fd即 af-xdp sock fd;
    if (map == MAP_FAILED)
        return -errno;

    fill->mask = umem->config.fill_size - 1;
    fill->size = umem->config.fill_size;
    fill->producer = map + off.fr.producer;
    fill->consumer = map + off.fr.consumer;
    fill->flags = map + off.fr.flags;
    fill->ring = map + off.fr.desc;
    fill->cached_cons = umem->config.fill_size;

    map = mmap(NULL, off.cr.desc + umem->config.comp_size * sizeof(__u64),
           PROT_READ | PROT_WRITE, MAP_SHARED | MAP_POPULATE, fd,
           XDP_UMEM_PGOFF_COMPLETION_RING);// 此中的fd即 af-xdp sock fd;
    if (map == MAP_FAILED) {
        err = -errno;
        goto out_mmap;
    }

    comp->mask = umem->config.comp_size - 1;
    comp->size = umem->config.comp_size;
    comp->producer = map + off.cr.producer;
    comp->consumer = map + off.cr.consumer;
    comp->flags = map + off.cr.flags;
    comp->ring = map + off.cr.desc;
```

上面代码需要关注的一点是 mmap() 函数中指定内存的长度——  `off.fr.desc + umem->config.fill_size * sizeof(__u64)`, `umem->config.fill_size * sizeof(__u64)`没什么好说的，就是ring数组的长度；
而`off.fr.desc`是desc相对 ring 结构体起始地址的偏移。

![](attachments/Pasted%20image%2020240728185820.png)

## 内核态程序


# AF_XDP实现流量分叉

## 背景


## 实现

## 理解

一个af_xdp socket 只是和具体的某个网卡的队列进行绑定。
并不代表某个监听的port；有可能一个网卡队列收到的包 redirect 给一个 af-xdp socket，其实是给多个监听port的。

比如：网卡存在16个队列，那么可以在af-xdp应用程序中，创建16个线程，每个线程都存在一个af-xdp socket ，并且绑定到对应的网卡接收队列中。其中，af-xdp socket 所在的线程，其实是监听了多个port。

# AF_XDP的时延问题

https://hal.science/hal-04458274v1/document



# DPDK中 整合 AF_XDP

## 依赖

![](attachments/Pasted%20image%2020240719110156.png)



## AF_XDP PMD

在DPDK中，我们实现了AF_XDP的PMD，它是一层对AF_XDP基本操作的封装，这样DPDK应用程序可以不经修改，直接通过命令行指定AF_XDP VDEV，从而享受AF_XDP带来的益处。

![](attachments/Pasted%20image%2020240710160120.png)

![](attachments/Pasted%20image%2020240710161552.png)


## zero-copy

![](attachments/Pasted%20image%2020240710161952.png)

![](attachments/Pasted%20image%2020240710162034.png)


# DPDK AF_XDP 调优
![](attachments/Pasted%20image%2020240716165139.png)

参考：[# DPDK PMD for AF_XDP Tests](https://doc.dpdk.org/dts/test_plans/af_xdp_test_plan.html)

## 流程拆分
存在2部分工作：
1> 某个队列收到的包, 然后驱动Napi之后经过xdp驱动程序进行处理；
2> dpdk程序处理AF_XDP缓冲区中的包。

对于上诉的1和2，可以使用同一个core来处理，也可以使用不同的core来处理。
dpdk程序的线程使用的core是通过 dpdk的eal参数`'-c’ ‘-l’ or ‘–lcores’`来决定的。
内核中的xdp程序的处理core是通过 `/proc/irq/<irq_no> 、/proc/interrupts` 来决定哪个队列的包的中断由哪个core来处理。

![](attachments/Pasted%20image%2020240716165837.png)

## XDP驱动程序和DPDK线程使用不同core

如果存在足够多的 cpu core，那么 将 AF_XDP PMD程序和 XDP驱动程序放入到不同的 core上，可能会有更好的性能。这样就相当于是 pipeline 模式（即 receiver-worker 模式）了。

绑定了某个接收队列的XDP驱动程序的CPU，是通过将对应的网卡队列的中断设置到某个CPU上决定的。
>注： 一些网卡存在中断脚本，set_irq_affinity.sh 可以轻易的设置网卡的中断的亲和性。

DPDK程序的CPU是通过启动的时候，`-l` 参数决定的。

注意：XDP驱动程序和DPDK程序使用不同的core时，建议DPDK参数中添加`busy_budget=0`。`busy_budget=0` 将会禁用 `preferred busy polling`。

另外，设置下面的内核参数。
```
echo 0 >> /sys/class/net/eth0/napi_defer_hard_irqs
echo 0 >> /sys/class/net/eth0/gro_flush_timeout
```

## XDP驱动程序和DPDK线程使用同一个core

参考：[ preferred busy-polling 介绍](https://lwn.net/Articles/837010/)

当 XDP驱动程序和DPDK程序使用相同的core时，为了优化，建议存在下列的配置：
```bash
(1) DPDK启动参数
busy_budget=xxx: xxx非0；

(2) 内核配置
echo 2 | sudo tee /sys/class/net/eth0/napi_defer_hard_irqs
echo 200000 | sudo tee /sys/class/net/eth0/gro_flush_timeout
```

因为DPDK程序和XDP驱动程序在同一个core，而DPDK程序为了高性能，一般是期望批量处理+polling模式的；但是，此时收包的硬件中断可能会影响DPDK程序的执行。

因此，上面配置延迟中断的使能，softirq 中 NAPI的调度是通过watchdog的超时，当DPDK程序退出Polling模式处理时，watchdog 定时器就会超时，然后正常的 softirq 中的 NAPI 就会恢复。


# 相关问题
## 确保af xdp scoket的4个ring的操作都是SPSC

在 Linux 上，一种名为 NAPI 的机制可以在每次网络接口接收数据包时防止发生 CPU 中断。它指示网络驱动程序在很短的时间间隔内处理一定数量的数据包。一个符合 NAPI 标准的网络驱动程序可保证，与 NAPI 上下文绑定的数据包 `(struct napi_struct *napi)` 不会在多个处理器上同时处理。
一般而言，设备的每个队列都有一个 NAPI 上下文，即每个 AF_XDP 套接字及其关联的一组环形缓冲区（RX、TX、FILL、COMPLETION）。

## dpdk af-xdp pmd中收到的rte_mbuf 无法利用硬件 offload 特性

### 内存拷贝
**（1）可能的拷贝一：umem 和 mbuf 之间的拷贝**
1> 如果没有启动 XDP_UMEM_UNALIGNED_CHUNKS，则存在 umem 和 mbuf 之间的拷贝。
xdp驱动程序 redirct 到 用户态程序的 umem的 是  raw frame帧，对于dpdk应用程序而言，需要将其封装为 rte_mbuf的形式。那么就需要将 umem 中的包 拷贝到 rte_mbuf 中。

2> 如果存在 XDP_UMEM_UNALIGNED_CHUNKS， 则 umem 和 mbuf 之间是零拷贝。
零拷贝，即umem不申请实体空间，而是 从 mbuf的mempool中申请多个mbuf，然后每个mbuf的地址 作为 umem实体 传递给 rx/tx/fill/complete 作为描述符。

**（2）可能的拷贝二**：

### rte_mbuf 中的metedata无法利用硬件特性
dpdk收包都是从网卡设备dev收包，对于一些虚拟的pmd，比如，af-xdp pmd 是通过创建虚拟的 vdev设备，然后将各种 af-xdp的处理封装为 dev的 ops。

由于 需要将 umem中收到的数据 拷贝封装到 rte_mbuf中，此时应该无法利用硬件的 offload 特性 以及 GRO 特性，因此 rte_mbuf 中的一些元数据 比如 ol_flags （offload features） 就无法赋值。

# 参考
```bash

# 一个调试故事：AF_XDP 中的受损数据包；是内核错误还是用户错误？
https://blog.cloudflare.com/a-debugging-story-corrupt-packets-in-af_xdp-kernel-bug-or-user-error-zh-cn

# 内核中的 af_xdp 英文介绍
https://www.kernel.org/doc/html/latest/networking/af_xdp.html
https://ebpf-docs.dylanreimerink.nl/linux/concepts/af_xdp/


# Accelerating networking with AF_XDP
https://lwn.net/Articles/750845/

# AF_XDP PMD in DPDK
https://mp.weixin.qq.com/s/nfCQSs8VhWyY5SGOvoxSEw

# DPDK  AF_XDP PMD
https://doc.dpdk.org/guides-23.11/nics/af_xdp.html

# Integrating AF_XDP into DPDK
https://www.dpdk.org/wp-content/uploads/sites/35/2019/07/14-AF_XDP-dpdk-summit-china-2019.pdf

# Knot ： af_xdp 实现的 dns
https://www.knot-dns.cz/docs/3.3/singlehtml/  （knot 介绍）

# kxdpgun: af_xdp实现的 压力测试工具
https://www.knot-dns.cz/docs/latest/html/man_kxdpgun.html （kxdpgun 介绍）
https://blog.csdn.net/qq_45675153/article/details/131595753
https://blog.csdn.net/qq_45675153/article/details/131647719

## AF_XDP技术详解
https://rexrock.github.io/post/af_xdp1/

# af_xdp 范例 
1》可以看内核里的范例：samples/bpf/xdpsock_user.c、xdpsock_kern.c
比如：https://github.com/xdp-project/bpf-examples/tree/master/AF_XDP-example

2》其他的范例：
https://blog.csdn.net/tunmang5421/article/details/128195304

# The Path to DPDK Speeds for AF XDP
http://vger.kernel.org/lpc_net2018_talks/lpc18_paper_af_xdp_perf-v2.pdf

# DPDK  AF_XDP PMD 中文理解
https://blog.csdn.net/force_eagle/article/details/117302414

# Introduce preferred busy-polling socket option
https://lwn.net/Articles/837010/

# DPDK PMD for AF_XDP Tests
https://doc.dpdk.org/dts/test_plans/af_xdp_test_plan.html
```