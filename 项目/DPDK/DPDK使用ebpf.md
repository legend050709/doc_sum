# 背景
## ebpf介绍

eBPF（extended Berkeley packet filter）是为扩展linux内核的一种技术能力，用户可以根据自己的需要不断增加eBPF模块来增强内核的功能。一般来说要向内核添加新功能，需要修改内核源代码或者编写内核模块来实现。
而 eBPF 允许程序在不修改内核源代码，或添加额外的内核模块情况下运行。
![](attachments/image%20(17).png)
其中：
- eBPF程序：这是一个小型的、用特定的eBPF字节码编写的程序。这些程序通常使用高级语言（如C）编写，并使用专门的编译器（如LLVM的BPF后端）编译成eBPF字节码。
- 加载和验证：当eBPF程序准备好被执行时，它首先被加载到内核中。在这个过程中，内核的eBPF验证器(Verifier)会检查程序，确保它是安全的（例如，不会导致无限循环或访问非法内存）。这是eBPF的一个关键安全特性。
- JIT编译：为了提高性能，内核通常会将eBPF字节码即时编译（JIT）成本地机器代码。
- eBPF Maps：这是eBPF程序用来存储数据的关键数据结构。它们允许内核和用户空间之间的数据交换。
- 挂钩点（Hook Points）：eBPF程序需要在特定的内核代码路径上执行。这些路径被称为“挂钩点”。例如，网络相关的eBPF程序可能在数据包进入或离开系统时执行。Linux内核提供了许多这样的挂钩点，不仅限于网络，还包括调度、系统调用等。

## ebpf原理
最初eBPF是专门为过滤网络数据包而设计的，所以eBPF非常适合编写基于数据包路由和虚拟化程序。
但随着需求发展eBPF已经不限于在网络数据包过滤功能了，逐步发展成**类似一个内核执行各种用户态指令的虚拟机**。其实eBPF是个基于RISC指令寄存器的虚拟机（如下图）。

用户用C语言或者eBPF汇编指令编写的BPF程序，然后通过LLVM/Clang这类编译器编译成eBPF字节码，再通过bpf函数将字节码载入到内核，
最后字节码会在JIT(Just in time)技术翻译成可执行的机器码(Native code)后保存到对应的内核功能路径上，当内核运行到这些路径时就会执行这对应eBPF机器码，整个过程有点像Java虚拟机的AOP技术。
用户用于编写BPF程序的C语言和汇编语言是BPF规范制定的，其中汇编指令是eBPF自己的系统指令，而BPF C语言是一种限制性C，只继承了标准C的语法，其程序结构和函数库要遵循eBPF规范。这从而限制了编写大规模复杂程序的可能性。
![](attachments/image%20(18).png)

# ebpf应用
根据其功能和应用场景，我们可以将eBPF应用程序大致划分为以下三类：
- 跟踪与监控：这类eBPF程序主要用于从内核及应用程序中抽取运行信息，帮助我们深入了解系统的实时运行状况。
- 网络处理：这类程序主要针对网络数据包，进行深度的过滤和处理，使我们能够详细了解并精准控制网络数据的传输流程。
- 安全与策略扩展：这部分的eBPF程序主要聚焦于系统的安全性，通过BPF来增强和细化安全策略和控制。


![](attachments/image%20(20).png)
eBPF 在网络方面最常用的两个钩子是 XDP（eXpress Data Path） 和 TC（Traffic Control），他们在数据包处理流程中的不同位置，提供了不同的操作能力。

- XDP：位于网卡驱动中，最接近物理网卡的位置，此处**只能处理 Ingress方向数据包**，且支持的能力比较简单，因此最适合一些简单高效的操作，常见有两种类型应用：
	- 安全检测过滤：由于XDP收包链路最靠近网卡的位置，所以是运行包过滤的理想选择，可用于丢弃恶意或非预期的流量、进行 DDOS 攻击保护等场景；
	- 高性能网关：eBPF在XDP中提供了一个轻量化的kernel bypass方案，可以绕过冗长的内核协议栈实现高效的数据包收发（如Facebook L4负载均衡Katran），与DPDK相比，eBPF性能略弱但应用更灵活，对硬件依赖更少；
    

- TC：TC 比 XDP 更靠近协本地议栈，此时包含更多上层信息，如cgroup、pid，且支持ingress和egress双向处理，还提供了灵活的重定向能力，所以常见有两种应用：
	- 容器/进程的流量监控；
	- overlay网络加速和控制
		- 收包方向，通过数据包重定向，可以绕过宿主机的本地协议栈直接转发给目的网卡；
		- 发包方向，根据对应cgroup的访问策略执行丢包或限速；
## 网络处理eBPF程序
![](attachments/image%20(19).png)
网络类的 eBPF 程序在 Linux 网络堆栈的多个层面都有应用。以下是一些常见的网络类 eBPF 程序类型和它们的应用场景：
1. XDP (eXpress Data Path)：XDP 允许 eBPF 程序在网络驱动层面处理数据包，这是数据包进入内核的最早阶段。由于其高效性，XDP 常用于高速数据包过滤、负载均衡和DDoS防御。
2. Traffic Control (tc) BPF：tc BPF 允许在更高层的网络栈中处理数据包，常用于复杂的数据包过滤、路由。
3. Socket-level BPF：这种 eBPF 程序可以与特定的套接字关联，允许对套接字级别的数据进行过滤和处理。常用于套接字级别的策略执行、安全和监控。
BPF Cgroup hooks：这种 eBPF 程序允许在cgroup层面进行数据包处理，从而为特定的进程组提供网络策略。常用于容器和进程级别的网络策略和监控。
## XDP有三种运行模式
XDP定义为BPF_PROG_TYPE_XDP，它在网卡驱动程序接收到数据包时触发。
由于XDP程序直接在数据包接收时介入，避免了复杂的内核网络协议栈处理，因此它能够提供高效的网络处理能力。

XDP有三种运行模式：
- 通用模式：此模式不依赖特定的网卡或其驱动程序。在这里，XDP程序在内核中运行，就像传统的网络协议栈那样。由于其通用性，性能可能不是最佳，但它非常适合测试和原型开发。
- 原生模式：这要求网卡的驱动程序提供支持。在原生模式下，XDP程序在网卡驱动的早期处理路径中执行，从而实现更高的性能。
- 卸载模式：在此模式下，网卡的固件必须支持XDP卸载功能。这意味着XDP程序直接在网卡上运行，完全绕过主机CPU，从而实现最高的性能。

处理完数据包后，XDP程序必须根据其执行结果确定数据包的下一步。这些决策是基于XDP程序的五种可能的结果码来进行的：

| 动作分类 | 说明 |
| :----- | :----- |
|XDP_DROP|丢弃|
|XDP_PASS|不做处理，传递到内核协议栈|
|XDP_TX|转发数据包到同一网卡|
|XDP_REDIRECT|转发数据包到不同网卡|
|XDP_ABORTED|XDP程序运行错误，数据包丢弃|

## eBPF内核态程序与用户态程序的交互
在eBPF中，映射（Maps）是一种用于在用户态程序和内核态程序之间交换数据的重要机制。映射类似于键值对存储，允许用户态和内核态程序在共享的数据结构中读取和写入数据。
下面是eBPF用户态和内核态如何通过 map 交互的详细步骤：
1. 创建和管理Map:
  使用 bpf_create_map 或SEC("maps")宏创建一个 map。
2. 内核态的操作:
  在 eBPF 程序中，可以使用 BPF helper 函数，如 bpf_map_update_elem, bpf_map_lookup_elem 和 bpf_map_delete_elem 来操作 map 的内容。
3. 用户态的操作:
  在用户空间，同样可以使用对应的helper，如 bpf_map_lookup_elem, bpf_map_update_elem 和 bpf_map_delete_elem，通过前面创建 map 时获得的文件描述符来访问和管理 map 的内容。
4. 双向交互:
  内核中的 eBPF 程序可能会根据数据包内容或其他条件更新 map 的值。
  同时，用户空间应用程序可能会读取这些 map 来获得统计信息、状态或其他数据，或者更新 map 以影响 eBPF 程序的行为。
5. 共享和引用:
  eBPF map 还可以被多个 eBPF 程序引用，或者被用作“固定(pin)”map，使其在多个程序或用户空间应用程序之间持久共享。
  
# dpdk对于ebpf的支持
dpdk 18.05 引入 ebpf，实现为 librte_bpf 库。

- 支持的特性：
支持 ebpf 架构（不支持 tail-pointer）
x86_64 与 arm64 架构支持使用 JIT将 ebpf 代码编译为 native code
支持 ebpf 代码校验支持用户定义的辅助函数
支持 rx/tx 包过滤
支持 ebpf 代码访问 mbuf 结构

- 不支持的特性：
cbpf
maps
stail-pointer calls
# dpdk ebpf api 接口
通过注册 rx、tx 函数内的 callback 来实现过滤功能，最终控制的是返回给上层、传输给下层的特定过滤格式的 mbuf。

过滤功能的实现通过 epbf 指令来完成，支持虚拟机模拟与 jit 两种方式。jit 会将 bpf 指令转化为 x86 指令来运行。

目前的编程方式：用 c 等高级语言编写 ebpf 过滤操作，然后编译为 bpf 指令码来加载。

![](attachments/Pasted%20image%2020230926201223.png)

# 范例
## 前置条件
1》config 配置文件中使能如下配置：
	RTE_LIBRTE_GRO
	RTE_LIBRTE_GSO
	RTE_LIBRTE_BPF
	RTE_LIBRTE_BPF_ELF
	RTE_TEST_PMD
	RTE_ETHDEV_RXTX_CALLBACKS
2》使用 clang 编译 examples/bpf/ 目录下的文件为 bpf 指令码
3》运行 testpmd，通过 bpf-load 命令加载第二步生成的指令码

## 具体范例
ebpf 过滤 c 程序源码如下：
```c
#include <stdint.h>
#include <net/ethernet.h>
#include <netinet/ip.h>
#include <netinet/udp.h>
#include <arpa/inet.h>

uint64_t
entry(void *pkt)
{
        struct ether_header *ether_header = (void *)pkt;

        if (ether_header->ether_type != htons(0x0800))
                return 0;

        struct iphdr *iphdr = (void *)(ether_header + 1);
        if (iphdr->protocol != 17 || (iphdr->frag_off & 0x1ffff) != 0 ||
                        iphdr->daddr != htonl(0x1020304))
                return 0;

        int hlen = iphdr->ihl * 4;
        struct udphdr *udphdr = (void *)iphdr + hlen;

        if (udphdr->dest != htons(5000))
                return 0;

        return 1;
}

```
上述代码实现类似 tcpdump -s 1 -d 'dst 1.2.3.4 && udp && dst port 5000’ 命令行的功能，过滤出目的 ip 为 1.2.3.4 且目的端口为 5000 的 udp 报文。

上述源码摘自 dpdk 工程中的 examples/bpf/t1.c，编译命令如下：
```c
clang -O2 -U __GNUC__ -target bpf -c t1.c

```
编译成功后，运行 **testpmd** 程序，执行 **bpf-load** 命令，示例如下：
```text
testpmd> bpf-load rx|tx <portid> <queueid> <load-flags> <filename>
testpmd> bpf-unload rx|tx <portid> <queueid>
```
执行 **bpf-unload** 命令即可卸载 ebpf 规则。

# dpdk ebpf 的安全性
dpdk ebpf 实现做了如下安全性保障：

- 参数合法性校验
- ebpf 指令合法性（正确的格式、有效字段值等）校验
- 校验是否存在不可达的指令或循环
- 加载前模拟执行所有可能存在的分支中的 ebpf 指令
- 校验失败则终止加载
- 其它的安全保障：
	- ebpf jit 翻译完成后将 rte_bpf 所在的页设置为 READ ONLY
	- ebpf jit 翻译后生成的 native code 存储的页设置为读与可执行

# 性能测试
- 测试方法：
	- 基于 dpdk-19.11 testpmd 测试
	- 编写一个直接返回 1 的 ebpf 空规则
	- 使用 testpmd 加载第二步生成的 dpdk 规则
	- 持续打流 100% 带宽观测性能数据

在飞腾 D2000 上使用一个20G网卡测试，测试确定对 512、1518 字节性能几乎没有影响，64 字节下降了不到 3%。

# 缺陷
## 不支持map
- 问题
![](attachments/Pasted%20image%2020230927103628.png)

- 思路
 ![](attachments/Pasted%20image%2020230927104613.png)
 
# 参考
```c

```