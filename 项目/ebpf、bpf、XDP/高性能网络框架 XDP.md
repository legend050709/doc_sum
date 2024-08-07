```table-of-contents
```
# XDP 介绍
## ebpf 简介
在讨论xdp之前我想先简单介绍一下（e）bpf，**bpf是一个通用的RISC指令集（因此需要配在虚拟机中执行，虚拟机中可以识别该指令集）**，最初设计用于在C的子集中编写程序，可以通过编译器后端（例如LLVM）将其编译为BPF指令，然后通过内核中的JIT编译器将其映射到本机操作码中，以在内核中实现最佳执行性能。

## ebpf 优点
### 快速
eBPF 几乎总是比 iptables 快，这是有技术原因的。
- eBPF 程序本身并不比 iptables 快，但 eBPF 程序更短。
- iptables 基于一个非常庞大的内核框架（Netfilter），这个框架出现在内核 datapath 的多个地方，有很大冗余。

### 灵活

**你可以用 eBPF 做几乎任何事情**。

eBPF 基于内核提供的一组接口，运行 JIT 编译的字节码，并将计算结果返回给内核。例如 **内核只关心 XDP 程序的返回是 PASS, DROP 还是 REDIRECT。至于在 XDP 程序里做什么， 完全看你自己**。

### 数据与功能分离
eBPF separates data from functionality.
**eBPF 真正的优势**是将“数据与功能分离”这件事情做地非常干净：可以在 eBPF 程序不中断的情况下修改它的运行方式。具体方式是修改它访 问的配置数据或应用数据，例如黑名单里规定的 IP 列表和域名。

![](attachments/Pasted%20image%2020240715173936.png)

`nftables` 和 `iptables` 也能干这个事情，但功能没有 eBPF 强大。例如，eBPF 可以使 用 per-cpu 的数据结构，因此能取得更极致的性能。

## XDP 简介

XDP：eXpress Data Path，快速数据路径。其实它就是一个快速处理==Rx数据包==的数据面技术。
在Linux4.8以上版本，提供了XDP技术支持。  XDP可以理解成一个**Linux内核网络栈的最底层（即驱动层）的数据处理机制**，利用钩子为BPF提供了一个加载的入口。而这个入口可以**直接操作数据而无需劳动内核其它的部分**。
==XDP只能用来处理Rx路径上的数据包而不会处理Tx路径上的数据包==。

![](attachments/Pasted%20image%2020240711195940.png)

XDP hook 位于网络驱动的快速路径上，XDP 程序直接从接收缓冲区（receive ring）中将 包拿下来，无需执行任何耗时的操作，例如分配 `skb` 然后将包推送到网络协议栈，或者 将包推送给 GRO 引擎等等。因此，只要有 CPU 资源，XDP BPF 程序就能够在最早的位置执 行处理。

# 依赖

![](attachments/Pasted%20image%2020240719110156.png)

# XDP系统组成
## xdp hook
### 介绍

XDP驱动钩子：网卡驱动中XDP程序的一个hook，XDP程序可以对数据包进行逐层解析、按规则进行过滤，或者对数据包进行封装或者解封装，修改字段对数据包进行转发等；

### 分类
根据不同的工作模式其挂载点也不同，如下图所示，根据网卡支持情况有三处挂载点(蓝色方框内)：

![](attachments/Pasted%20image%2020240711200243.png)

![](attachments/Pasted%20image%2020240709195334.png)

==如上所示，无论是哪种模式的 XDP，都是在TC之前的。即：native XDP--->申请skb--->gro --> gxdp----> tcpdump--->TC--->netfilter==

- Native XDP（默认xdp，驱动xdp、原生xdp）：即驱动模式，在这种模式中，XDP BPF 程序直接运行在网络驱动的早期接收路径上，需要驱动支持。
    
- Offloaded XDP（卸载xdp）：在该模式下XDP BPF程序直接offload到网卡，相较于Native，具有更高的性能，需要网卡支持。
    
- Generic XDP（通用xdp，gxdp）：对于还没有实现Native或Offloaded XDP的驱动，内核提供了一个Generic XDP选项。该模式下的XDP BPF Program运行于驱动程序之后的位置，无需驱动程序的支持，但性能较差，主要面向测试程序的开发者。

### Native XDP
Native XDP（原生模式/本地模式） 支持Linux内核中的高性能可编程数据包处理。它在软件中尽可能早地运行BPF程序，即在网络驱动程序接收数据包时，需要**驱动支持**。大部分广泛使用的 10G 及更高速的网卡都已经支持这种模式 。

xdp 使用ebpf 做包过滤，相对于dpdk将数据包直接送到用户态，用户态当做快速数据处理平面；xdp是在驱动层创建了一个数据快速平面。**在数据被网卡硬件dma到内存，分配skb之前，对数据包进行处理**。由于完全不存在锁操作。且bypass了协议栈，非常适合用修改数据包并转发，数据探针，执行丢包。

![](attachments/Pasted%20image%2020240709195735.png)

#### 支持 native XDP 的驱动

- **Broadcom**
    
    - bnxt
- **Cavium**
    
    - thunderx
- **Intel**
    
    - ixgbe
    - ixgbevf
    - i40e
- **Mellanox**
    
    - mlx4
    - mlx5
- **Netronome**
    
    - nfp
- **Others**
    
    - tun
    - virtio_net
- **Qlogic**
    
    - qede
- **Solarflare**
    
    - sfc （XDP for sfc available via out of tree driver as of kernel 4.17, but will be upstreamed soon）


###  Generic XDP 

将部分逻辑下放（offload）到网卡执行，通过硬件处理提高效率。 但是不是所有网卡都支持这个功能，所以内核引入了 Generic XDP 这样一个环境，如果网卡不支持 XDP， 那 XDP 程序就会推迟到这里来执行。它并不能提升效率，所以主要用来测试功能。

### Offloaded XDP

Offloaded XDP （卸载模式）是XDP程序直接运行到网卡上，相较于Native，具有更高的性能，需要**网卡硬件支持**。

这种 offload 通常由智能网卡实现，这些网卡有多 线程、多核流处理器（flow processors），一个位于内核中的 JIT 编译器（ in-kernel JIT compiler）将 BPF 翻译成网卡的原生指令。支持 offloaded XDP 模式的驱动通常也支持 native XDP 模式，因为 BPF 辅助函数可 能目前还只支持后者。

#### 支持 offloaded XDP 的驱动

- **Netronome**
    
    - nfp （Some BPF helper functions such as retrieving the current CPU number will not be available in an offloaded setting）

## bpf部分

开发者用**C语言的一个子集（内核运行，不可用标准C库）** 开发程序，然后用LLVM/clang编译器将其编译成eBPF指令（Bytecode），在eBPF验证器（Verifier）检验通过后被内核中的即时编译器（JIT Compiler）将eBPF指令映射成处理器的原生指令（opcode）再加载到内核各个模块预设的钩子（Hooks）处。其中XDP框架是内核在网卡驱动开辟的一个网络数据快速路径的钩子（Hooks）。
内核其他典型钩子（Hooks）分别为内核函数 (kprobes)、用户空间函数 (uprobes)、系统调用、fentry/fexit、跟踪点、网络路由、TC、TCP拥塞算法、套接字等模块。

### eBPF虚拟机
XDP 程序在 Extended BPF (eBPF) 虚拟机中执行。
虚拟机中执行 XDP 程序的字节码，以及对字节码执行 JIT 以提升性能。
XDP程序通过Clang编译成BPF字节码，而BPF字节码加载到内核中是运行在eBPF虚拟机上，eBPF VM支持XDP程序的动态加载和卸载。



### eBPF maps

eBPF程序的内核态与用户态数据交换通过BPF maps来实现，其类似进程间通信的共享内存访问。其支持的数据类型有Hash表、数组、LRU缓存(Least Recently Used)、 环形队列、堆栈轨迹、LPM路由表(Longest Prefix match)。

用户态程序可以在BPF Maps中预定义规则，XDP程序可以匹配Maps中的规则对数据包进行过滤等；XDP程序也可以将数据包统计信息等存入Maps，用户态程序可访问Maps获取数据包统计信息。


eBPF 程序在触发内核事件时执行（例如，触发 XDP 程序执行的，是收包事件）。 程序每次执行时，初始状态都是相同的（即程序是无状态的），它们**==无法直接访问==** 内核中的持久存储（BPF map）。为此，内核提供了访问 BPF map 的 helper 函数。

BPF map 是 key/value 存储，**==在加载 eBPF 程序时定义==**（defined upon loading an eBPF program）。



#### Map 类型
- Hash tables, Arrays
- LRU (Least Recently Used)
- Ring Buffer
- Stack Trace
- LPM (Longest Prefix match)


#### 作用

![](attachments/Pasted%20image%2020240715194716.png)

1. 持久存储。例如一个 eBPF 程序每次执行时，都会从里面获取上一次的状态。
2. 用于协调两个或多个 eBPF 程序。例如一个往里面写数据，一个从里面读数据。
3. 用于用户态程序和内核 eBPF 程序之间的通信。




### eBPF verifier

由于eBPF代码直接运行在内核地址空间，此它能直接访问（破坏）任何内存。为防止这种情况发生，Verifier（程序校验器）需要在XDP字节码加载到内核之前对字节码进行安全检查。

加载 BPF 程序时，位于内核中的校验器首先会对字节码程序进行静态分析，以确保
- 程序中没有任何不安全的操作（例如访问任意内存），
- 程序会终止（terminate）。通过下面这两点来实现：
    - **禁止循环操作**
    - **限制程序最大指令数**


### ebpf helper func

eBPF通过提供**辅助函数来弥补标准C库的缺失**。
常见的如获取随机数、获取当前时间、map访问、获取进程/cgroup上下文、处理网络数据包和转发、访问套接字数据、执行尾调用、访问进程栈、访问系统调用参数等，在实际开发中可通过man bpf-helpers命令获取更多帮助信息。

```text
有哪些Helpers？
	随机数
	获取当前时间
	map访问
	获取进程/cgroup 上下文
	处理网络数据包和转发
	访问套接字数据
	执行尾调用
	访问进程栈
	访问系统调用参数
	...
```

#### eBPF Tail and Function Calls

![](attachments/Pasted%20image%2020240715194305.png)

（1）尾调用有什么用？
将程序链接在一起
将程序拆分为独立的逻辑组件
使 BPF 程序可组合

（2）函数调用有什么用？
重用内部的功能程序
减少程序大小（避免内联）

### ebpf 可以做什么

![](attachments/Pasted%20image%2020240715194831.png)

## XDP Action

![](attachments/Pasted%20image%2020240715112232.png)

XDP程序对于报文的处理有如下几种方式：

- XDP_DROP：在驱动层丢弃报文，通常用于实现DDos或防火墙。
    
- XDP_PASS：允许报文上送到内核网络栈，同时处理该报文的CPU会分配并填充一个skb，将其传递到内核协议栈。
    
- XDP_TX：从当前网卡发送出去。
    
- XDP_REDIRECT：将包重定向到其他网络接口（包括虚拟机的虚拟网卡），或者其他的CPU、或者通过AF_XDP socket重定向到用户空间。
    
- XDP_ABORTED：表示程序产生了异常，其行为和XDP_DROP相同，但XDP_ABORTED会经过trace_xdp_exception tracepoint，因此可以通过tracing工具来监控这种非正常行为。


## 流程
### 整体流程

![](attachments/Pasted%20image%2020240716140435.png)

NAPI poll 机制不断调用驱动实现的 poll 方法，后者处理 RX 队列内的包，并最终将包送到正确的程序，也就是我们所说的 XDP 程序。所以很明显这需要网卡驱动的支持，如果驱动支持 XDP ，那 XDP 程序将在 poll 机制内执行。如果不支持，那 XDP 程序将只能在更后面的位置被执行，即上图中的receive_skb中。

**gxdp之前的流程**
（1）创建skb
如果不支持XDP，poll机制会将报文送给 clean_rx()，该函数会创建一个skb，并skb进行一些硬件校验何检查，然后较给 gro_receive() 函数；

（2）分片重组
GRO可以理解为LRO的软件实现，相比LRO只针对TCP报文，GRO可以处理更多其他类型的报文，总之在 gro_receive() 函数中，如果是分片报文则进行分片重组然后交给 receive_skb() 函数；如果不是分片报文，则直接交给 receive_skn() 函数进行处理；

### XDP流程
![](attachments/Pasted%20image%2020240710103100.png)

上图是基于XDP/AF_XDP系统数据流示例图。实线为数据面流向，虚线的控制面流向。

网卡收到包之后，会先执行挂载的XDP eBPF程序，读取由用户态的控制平面写入到BPF maps的数据包处理规则（用户态应用在此之前通过bpf map下放规则），基于规则实现数据包的过滤分发，即xdp action处理，是DROP，重定向到AF_XDP，还是PASS到内核协议栈。

从图中可以看出，不同eBPF程序之间可以通过BPF maps进行通信，并且内核态也可以通过BPF map与用户态应用进行交互，从而实现数据共享。


# XDP的优缺点

## 优点
### 对系统无侵入，不用改驱动，不用改内核
eBPF的程序完全运行在一个沙盒里，程序启动时加载进网卡就好了，而且eBPF程序真的很简单。
- 完整绕过了内核的协议栈，性能++
- 可以操作数据包的每一个Byte了，这个过程交由用户态程序处理
- 高度兼容性，相比DPDK没有那么苛刻的硬件要求

### 在设备驱动中执行，无需上下文切换

XDP 程序在网络设备驱动中执行，网络设备每收到一个包，程序就执行一次。

相关代码实现为一个内核库函数（library function），因此程序直接 在设备驱动中执行，无需切换到用户空间上下文。

### 在软件最早能处理包的位置执行，性能最优

![](attachments/Pasted%20image%2020240715142555.png)

回到上面图  可以看到：程序在网卡收到包之后最早能处理包的位置执行 —— 此时内核还没有为包分配 `struct sk_buff` 结构体， 也没有执行任何解析包的操作。



## 缺点
### eBPF 程序的限制

前面提到，加载到 eBPF 虚拟机的程序必须保证其安全性（不会破坏内核），因此对 eBPF 程序作了一下限制，归结为两方面：
1. 确保程序会终止：在实现上是通过禁止循环和限制程序的最大指令数（max size of the program）；
2. 确保内存访问的安全：通过 寄存器状态跟踪（register state tracking）来实现。

#### 校验逻辑偏保守

由于*校验器的首要职责是保证内核的安全，因此其校验逻辑比较保守， 凡是它不能证明为安全的，一律都拒绝。有时这会导致假阴性（**==false negatives==**）， 即某些实际上是安全的程序被拒绝加载；

这方面在持续改进。
- 校验器的错误提示也已经更加友好，以帮助开发者更快定位问题。
- 近期已经支持了 BPF 函数调用（function calls）。
- 正在计划支持有限循环（bounded loops）。
- 正在提升校验器效率，以便处理更大的 BPF 程序。

#### 缺少标准库
相比于用户空间 C 程序，eBPF 程序的另一个限制是缺少标准库，包括 **内存分配、线程、锁**等等库。

1. 内核的生命周期和执行上下文管理（life cycle and execution context management ）部分地弥补了这一不足，（例如，加载的 XDP 程序会为每个收到的包执行），
2. 内核提供的 helper 函数也部分地弥补了一不足。

#### 一个网卡的接收队列只能 attach 一个 XDP 程序
确切的说，应该是网卡的一个接收队列只能attach 到一个 XDP程序。
这个限制其实也是可以**绕过**的：将 XDP 程序组织成程序数组，通过尾 调用，根据包上下文在程序之间跳转，或者是将几个程序做 chaining。


### 其他
- XDP不提供缓存队列（qdisc），TX太慢（这里的Tx，应该是XDP_TX，而不是从协议栈发包，从协议栈发包不经过XDP程序）时会直接丢包，因而不能在RX比TX快的设备上使用XDP。
- 由于不具备缓存，对于IP分片不太友好，无法分片重组，不容易处理分片。
- XDP程序是专用的，不具备网络协议栈的通用性。

# XDP 范例

## 丢弃所有icmp包

```c
// ping_drop.c
 
#include <linux/bpf.h>
#include <linux/in.h>
#include <linux/if_ether.h>
#include <linux/if_packet.h>
#include <linux/ip.h>
 
#define SEC(NAME) __attribute__((section(NAME), used))
 
SEC("prog") //prog 作为一个hook ，介于自定义处理程序和XDP之间
 
int ping_drop(struct xdp_md *ctx)
{
    void *data = (void*)(long)ctx->data;         //报文数据开始处
    void *end = (void*)(long)ctx->data_end;      //报文数据结束点
 
    struct ethhdr *eh;                           //以太头
    eh = data;
    if (data > end)                              //这个检测有点多余，一个合格驱动会保证
        return XDP_PASS;                         //data一定是小于end的
    if ((void*)(eh+1) > end)                //这个检测非常重要，否则在下面读取 eh->h_proto
        return XDP_PASS;                    //的时候，无法通过bpf verifier的验证，程序就无法加载
 
    if (eh->h_proto != __constant_htons(ETH_P_IP))   //不是IP报文，放过
        return XDP_PASS;
 
    struct iphdr *iph;
    iph = (void*)(eh + 1);
 
    if ((void*)(iph+1) > end)               //这里的检测也非常重要，原因同上
        return XDP_PASS;
 
    if (iph->protocol == IPPROTO_ICMP)      //判断如果是ping报文，丢弃
        return XDP_DROP;                    //返回 XDP_DROP，会导致无法ping通主机，其他如ssh等不受影响
 
    return XDP_PASS;
}
 
char __license[] SEC("license") = "GPL";
```

XDP hook 点在网络驱动中，基于eBPF 的事件驱动机制，当XDP 收到网络数据包时，我们的处理程序就会被执行。

传入eBPF 处理程序的ctx 其实就是XDP 元数据(xdp metadata: xpd_md)，没有sk_buff结构，只有一个 struct xdp_md 指针。
```c
/* user accessible metadata for XDP packet hook
 * new fields must be added to the end of this structure
 */
struct xdp_md {
    __u32 data; //数据包开始指针
    __u32 data_end; //数据包结束指针
    __u32 data_meta; //初始阶段它是一个空闲的内存地址，供XDP程序与其他层交换数据包元数据时使用。
    /*
      分别是接收数据包接口的索引和对应的RX 队列的索引。当访问这两个值时，BPF 代码会在内核内部重写，
      以访问实际持有这些值的内核结构struct xdp_rxq_info。
    */
    /* Below access go through struct xdp_rxq_info */
    __u32 ingress_ifindex; /* rxq->dev->ifindex */
    __u32 rx_queue_index;  /* rxq->queue_index  */
};
```

Clang 编译生成对象文件，并加载
```bash
clang -Wall -target bpf -c ping_drop.c -o ping_drop.o
```

然后用iproute2 里面的 ip link 命令加载到某个NIC 上，如ens192
```bash
ip link set dev ens192 xdp object ping_drop.o
```

iproute2 需要打开 HAVE_ELF 这个宏，默认CentOS 7 并没有，需要编译iproute2 并打开。

![](attachments/Pasted%20image%2020240716145940.png)

```bash
git clone git://git.kernel.org/pub/pub/scm/network/iproute2/iproute2.git
 
cd iproute2/
./configure --prefix=/usr
make -j8 && make install
```

BPF map 和 bpf 程序作为内核资源只能通过文件描述符访问，其背后是内核中的匿名 inode。所以内核实现了一个最小内核空间 BPF 文件系统，BPF map 和 BPF 程序 都可以钉到（pin）这个文件系统内，这个过程称为 object pinning（钉住对象）。

![](attachments/Pasted%20image%2020240716151234.png)

挂载BPF FS，允许BPF 程序从虚拟文件系统固定和获取map
```bash
mount -t bpf /sys/fs/bpf /sys/fs/bpf
ip link set dev ens192 xdp object ping_drop.o # 之后，在其他节点ping 当前节点就会显示无法ping 通。
ip link set dev ens192 xdp off  之后，ping # 之后，恢复正常。
```

![](attachments/Pasted%20image%2020240716151322.png)
![](attachments/Pasted%20image%2020240716151325.png)

## 丢弃源 IP 命中黑名单的 ARP 包

![](attachments/Pasted%20image%2020240715173522.png)

ebpf_map ，可用于 在内核和用户态之间传递数据，例如通过一个特殊的系统从用户态向 map 里插入数据。


# XDP应用
## DDoS 防御、防火墙

XDP可以直接在应用服务器上部署包过滤程序来防御此类攻击，无须修改应用代码。如果应用部署在虚拟机里，XDP程序还可以部署在宿主机上，保护机器上所有的虚拟机。其性能单核可以轻松处理10Gbps的最小包Dos流量。这种DDOS防御的部署更加灵活。
>  XDP 在业界最出名的一个应用场景就是 Facebook 基于 XDP 实现高效的防 DDoS 攻击，其本质上就是实现尽可能早地实现“丢包”，而不去消耗系统资源创建完整的网络栈链路。

相比iptables相对较晚的hook点，XDP的丢包速率要比iptables高4倍左右。

XDP BPF 的一个基本特性就是用 `XDP_DROP` 命令驱动将包丢弃，由于这个丢弃的位置 非常早，因此这种方式可以实现高效的网络策略，平均到每个包的开销非常小（ per-packet cost）。这对于那些需要处理任何形式的 DDoS 攻击的场景来说是非常理 想的，而且由于其通用性，使得它能够在 BPF 内实现任何形式的防火墙策略，开销几乎为零， 例如，作为 standalone 设备（例如通过 `XDP_TX` 清洗流量）；或者广泛部署在节点 上，保护节点的安全（通过 `XDP_PASS` 或 cpumap `XDP_REDIRECT` 允许“好流量”经 过）。

Offloaded XDP 更进一步，将本来就已经很小的 per-packet cost 全部下放到网卡以 线速（line-rate）进行处理。

## 转发和负载均衡（load balancing)

XDP 的另一个主要使用场景是包转发和负载均衡，这是通过 `XDP_TX` 或 `XDP_REDIRECT` 动作实现的。

XDP 层运行的 BPF 程序能够任意修改（mangle）数据包，即使是 BPF 辅助函数都能增 加或减少包的 headroom，这样就可以在将包再次发送出去之前，对包进行任何的封装/解封装。

利用 `XDP_TX` 能够实现 hairpinned（发卡）模式的负载均衡器，这种均衡器能够 在接收到包的网卡再次将包发送出去，而 `XDP_REDIRECT` 动作能够将包转发到另一个 网卡然后发送出去。

`XDP_REDIRECT` 返回码还可以和 BPF cpumap 一起使用，对那些目标是本机协议栈、 将由 non-XDP 的远端（remote）CPU 处理的包进行负载均衡。


负载均衡的Tunnel转发模式中，通过对包头进行哈希，以此选择目标应用服务器，然后将数据包进行封装，发送给应用服务器，应用解封，处理请求，会包给客户端。在次过程中，XDP服务哈希，封包发送。通过bpf map进行配置，其性能比Linux内核IPVS高4倍左右。

## 栈前（Pre-stack）过滤/处理

除了策略执行，XDP 还可以用于加固内核的网络栈，这是通过 `XDP_DROP` 实现的。 这意味着，XDP 能够在可能的最早位置丢弃那些与本节点不相关的包，这个过程发生在 内核网络栈看到这些包之前。例如假如我们已经知道某台节点只接受 TCP 流量，那任 何 UDP、SCTP 或其他四层流量都可以在发现后立即丢弃。
    
这种方式的好处是包不需要再经过各种实体（例如 GRO 引擎、内核的 flow dissector 以及其他的模块），就可以判断出是否应该丢弃，因此减少了内核的 受攻击面。正是由于 XDP 的早期处理阶段，这有效地对内核网络栈“假装”这些包根本 就没被网络设备看到。
    
另外，如果内核接收路径上某个潜在 bug 导致 ping of death 之类的场景，那我们能 够利用 XDP 立即丢弃这些包，而不用重启内核或任何服务。而且由于能够原子地替换 程序，这种方式甚至都不会导致宿主机的任何流量中断。
    
栈前处理的另一个场景是：在内核分配 `skb` 之前，XDP BPF 程序可以对包进行任意 修改，而且对内核“假装”这个包从网络设备收上来之后就是这样的。对于某些自定义包 修改（mangling）和封装协议的场景来说比较有用，在这些场景下，包在进入 GRO 聚 合之前会被修改和解封装，否则 GRO 将无法识别自定义的协议，进而无法执行任何形 式的聚合。
    
XDP 还能够在包的前面 push 元数据（非包内容的数据）。这些元数据对常规的内核栈 是不可见的（invisible），但能被 GRO 聚合（匹配元数据），稍后可以和 tc ingress BPF 程 序一起处理，tc BPF 中携带了 `skb` 的某些上下文，例如，设置了某些 skb 字段。

## 流抽样（Flow sampling）和监控

XDP 还可以用于包监控、抽样或其他的一些网络分析。

例如作为流量路径中间节点 的一部分；或运行在终端节点上，和前面提到的场景相结合。
对于复杂的包分析，XDP 提供了设施来高效地将网络包（截断的或者是完整的 payload）或自定义元数据 push 到 perf 提供的一个快速、无锁、per-CPU 内存映射缓冲区，或者是一 个用户空间应用。

这还可以用于流分析和监控，对每个流的初始数据进行分析，一旦确定是正常流量，这个流随 后的流量就会跳过这个监控。感谢 BPF 带来的灵活性，这使得我们可以实现任何形式 的自定义监控或采用。

# XDP的未来展望

ebpf/XDP作为Linux网络革新技术正在悄悄的改变着Linux网络发展模式，当前，XDP技术被OVS、Cilium、Polycube等用于网络快速路径的新选择，DPDK也相应的做了AF_XDP PMD。XDP程序在CPU可用来处理的最早时间点被执行，尤其适合DDoS防御、防火墙、负载均衡。基于XDP+eBPF的的ACL解决方案也有望改善目前的性能瓶颈，有望取代iptables解决方案。

XDP作为一个安全、快速、可编程、集成到操作系统内核的包处理框架。XDP性能虽然与基于kernel bypass的DPDK仍有差距，但优异的可扩展性，可编程性等提供了非常有竞争力的优势。相比于kernel bypass这种非此即彼、完全绕开内核的方式，我们相信XDP有更广阔的的应用前景。

# 参考
```bash
# 内核中的 af_xdp 英文介绍，涉及到一些 FAQ （+++++++）
https://www.kernel.org/doc/html/latest/networking/af_xdp.html

# [译] [论文] XDP (eXpress Data Path)：在操作系统内核中实现快速、可编程包处理（ACM，2018）
https://arthurchiao.art/blog/xdp-paper-acm-2018-zh/

# [译] Cilium：BPF 和 XDP 参考指南（2021）
https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/

# [译] 基于 BPF/XDP 实现 K8s Service 负载均衡 (LPC, 2020)
https://arthurchiao.art/blog/cilium-k8s-service-lb-zh/

# [译] Cilium：BPF 和 XDP 参考指南（2021）
https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/

# 内核对于AF_XDP的官方介绍
https://www.kernel.org/doc/Documentation/networking/af_xdp.rst

# XDP 官方的教程、范例、使用方法、介绍。
https://github.com/xdp-project

# 官方对于 xdp 的 metadata的理解
https://www.kernel.org/doc/Documentation/networking/xdp-rx-metadata.rst

# Bringing TSO/GRO and Jumbo frames to XDP
https://lpc.events/event/11/contributions/939/attachments/771/1551/xdp-multi-buff.pdf

# eBPF 系列文章
https://mp.weixin.qq.com/s/Vr7At9-wG3p6mfY6L49nyg

#  一文读懂Linux网络新基石——XDP技术
https://mp.weixin.qq.com/s/6w5_kv_dEtTAQYOCpzdIFg

```