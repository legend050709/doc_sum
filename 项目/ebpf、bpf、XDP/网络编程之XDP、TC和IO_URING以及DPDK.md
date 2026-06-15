```table-of-contents
```


# 原始的内核协议栈
## iptables/netfilter的问题

iptables/netfilter 是上个时代Linux网络提供的优秀的防火墙技术，扩展性强，能够满足当时大部分网络应用需求。但该框架也存在很多明显问题：

- （1）路径太长
netfilter 框架在IP层，报文需要经过链路层，IP层才能被处理，如果是需要丢弃报文，会白白浪费很多CPU资源，影响整体性能；

- （2）O(N)匹配

![](attachments/Pasted%20image%2020240716101553.png)

如上图所示，极端情况下，报文需要依次遍历所有规则，才能匹配中，极大影响报文处理性能；iptables规则逐渐增加，遍历iptables效率变得很低。

- （3）无法增量更新
iptables的还有一个缺点，无法实现增量更新。每次添加新规则时，必须更新整个规则列表。一个例子：装配2万个Kubernetes服务产生16万条的iptables规则需要耗时5个小时。

## 内核协议栈的问题

![](attachments/Pasted%20image%2020240715195515.png)

协议栈复杂的处理逻辑：
>**各类链表在多CPU环境下的同步开销。**
>**不可睡眠的软中断路径过长。**
>**sk_buff的分配和释放。**
>**内存拷贝的开销。**
>**上下文切换造成的cache miss。**
>**…**

于是，内核协议栈各种优化措施应着需求而来：
>**网卡RSS，多队列。**
**中断线程化。**
**分割锁粒度。**
**Busypoll。**
>**…**

但却都是见招拆招，治标不治本。问题的根源不是这些机制需要优化，而是这些机制需要推倒重构。蒸汽机车刚出来的时候，马车夫为了保持竞争优势，不是去换一匹昂贵的快马，而是卖掉马去买一台蒸汽机装上。基本就是这个意思。

## 解法

重构的思路很显然有两个：

**upload方法**：别让应用程序等内核了，让应用程序自己去网卡直接拉数据。
**offload方法**：别让内核处理网络逻辑了，让网卡自己处理。

总之，绕过内核就对了，内核协议栈背负太多历史包袱。可以在用户层或内核层直接处理数据包，避免了用户态和内核态切换的开销。

**DPDK**让用户态程序直接处理网络流，bypass掉内核，使用独立的CPU专门干这个事。
**XDP**让灌入网卡的eBPF程序直接处理网络流，bypass掉内核，使用网卡NPU专门干这个事**。**
如此一来，内核协议栈就不再参与数据平面的事了，留下来专门处理诸如路由协议，远程登录等控制平面和管理平面的数据流。


# gxdp和tcpdump

xdp有三种工作模式，不论哪一种模式，在接收数据包时都比bpf后门要早。
tcpdump这种抓包工具的原理和bpf后门是一样的，也是工作在链路层。所以网卡接收到数据包后，会先经过xdp ebpf后门，然后分别经过bpf后门和tcpdump。
如果xdp ebpf后门在接收到恶意指令后把数据包丢掉，tcpdump就抓不到数据包。

![](attachments/Pasted%20image%2020240709200256.png)


`__netif_receive_skb_core` 完成**将数据送到协议栈**这一繁重工作。这里面做的事情非常多， 按顺序包括：

1. 处理 skb 时间戳；
2. **==Generic XDP==**：软件执行 XDP 程序（XDP 是硬件功能，本来应该由硬件网卡来执行）；
3. 处理 VLAN header；
4. TAP 处理：例如 **==tcpdump 抓包==**、流量过滤；
5. TC：TC 规则或 **==TC BPF 程序==**；
6. Netfilter：处理 iptables 规则等。

参考：[# Linux 网络栈接收数据（RX）：原理及内核实现（2022）](https://arthurchiao.art/blog/linux-net-stack-implementation-rx-zh/)


# native-xdp和 TC(qdisc)

XDP位于网络栈的最底层，**驱动层**，可以加载到驱动上进行运行。
而**TC是在数据链路层**，最主要的功能就是流量控制，这种流量控制要和TCP窗口流控区别开来。
==TC的控制主要是对数据包进行管理。也因为TC更接近上层，所以可以访问sk_buff（IP报文）这种数据结构，而不象XDP只能自己搞一个xdp_buff==。
也正因为如此，TC和硬件基本没有半毛钱关系了，这和XDP是一个比较明显的不同。

 由于位置的不同，所处的协议栈的层次不同，因此其对数据的处理就有所不同；**XDP位于底层，数据更原始完整，可以进行原始报文的修改控制**；而TC处于上层，但可以**使用更强大的数据结构（sk_buff）处理更复杂的修改需求**。

![](attachments/Pasted%20image%2020240711180224.png)

从图上可以看出，XDP和TC通过Maps调用内核中的eBPF来实现相关的功能。

## xdp、gxdp 、tc、gro、tcpdump的位置关系

在实践中经常遇到的tcpdump抓包程序，**其抓包的位置入口方向在XDP和TC之间，而出口方向位于TC之后**。

![](attachments/Pasted%20image%2020240711194330.png)

即：
**收包方向：native XDP--->分配skb--->GRO处理---->gxdp---->tcpdump---->TC---->netfilter(iptables)**

**发包方向：netfilter(iptables)--->TC--->tcpdump--->GSO**
> 注：==XDP是收包方向的技术，发包方向上不处理。TC主要作用在发包方向上==。


## rps 和 xdp的先后关系



# XDP 和 DPDK

![](attachments/Pasted%20image%2020240710113415.png)

XDP不是内核旁路，是**在网卡和内核协议栈之间增加了一个快速数据路径**。
XDP借助于eBPF技术从而继承了其可编程、即时实现、安全等优良特性。

## XDP和DPDK对比
DPDK在大容量高吞吐等场景有优势。
XDP在云原生等场景有优势。

### XDP的优势

XDP相较与DPDK来说，具有以下优点：

- DPDK会独占CPU资源且需要大页内存。**XDP无需专门硬件，无需大页内存，无需独占CPU**等资源，任何有Linux驱动的网卡都可以支持，无需引入第三方代码库。
    
- **兼容内核协议栈**，可选择性复用内核已有的功能。
    
- 保持了内核的安全边界，提供与内核API一样稳定的接口。
    
- 无需对网络配置或管理工具做任何修改。
    
- 服务不中断的前提下动态重新编程，这意味着可以**按需加入或移除功能，而不会引起任何流量中断**，也能动态响应系统其他部分的的变化。
    
- 主流的发行版中，Linux内核已经内置并启用了XDP，并且支持主流的高速网络驱动，4.8+的内核已内置，5.4+能够完全使用。

# 总结

仍然按照传统的用户层、内核和驱动（硬件NIC）三层来说明几者的关系：
1、IO_URING是一个**异步**编程框架，可以认为是提供给应用层使用的一个网络编程框架，一如开发经常使用epoll,select,poll等。它更多的是向上层提供一套更高效的网络通信接口。

2、DPDK：DPDK属于用户层程序可以直接处理网络数据的一种框架，它可以略过大多数的和内核交互的过程。DPDK的优秀之处已经在应用中体现出来，但是DPDK的不足也同样暴露了出来，

3、XDP、TC（eBPF）则可以认为是与DPDK不同的另外一种方案，DPDK是尽量减少和内核的交互，直接连接用户层和网络数据（驱动或网卡），而XDP等则在仍然与内核保持较强的联系。

可以简单理解为，IO_URING是为应用层服务的。后两者（DPDK、 XDP/TC）是在内核层次上的一次优化。虽然DPDK也实现了用户层的网络数据处理，但从数据处理的方式来看，仍然可以将其划到内核逻辑中去。而XDP等则可以看作内核对DPDK一种快速的应变。

三者的关系图：

![](attachments/Pasted%20image%2020240714171858.png)

## 应用分析
针对这三类技术，做一下整体的技术应用分析：
1、IO_URING:
做存储起家的异步框架，主要是解决IO与CPU、内存的不匹配出现的。应用在网络编程上，再自然不过。它解决的主要痛点就是在上层应用中支持C100K、C1000K甚至更多。通过对内核的交互大幅减少相关的内存拷贝和通信次数。略过一些不必要的过程，同时对外暴露接口，让用户层感受不到内核的复杂。
目前在Java的Netty 中已经封装了IO_URING的应用，不过据说并没有发挥出其最大效用，有兴趣可以用一下。其它的有名气的应用案例目前还没看到。

2、DPDK  
DPDK这个就不用细讲了，在前面已经分析了几十篇，它既可以在上层写传统的SOCKET编程，也可以做为网络监控等的手段。基本上大公司都有基于此框架的应用，开源的更是多，可以去Github上搜索一下就明白了。

3、eBPF(XDP、TC)
XDP之类的应用本身就集成在内核中，再加上eBPF的大力支持，它的应用更多。国内诸如腾讯、阿里等大互联网公司都有相关的应用如蓝鲸；中国移动的磐基，其它开源的也有很多，比如反复提到的Cilium。
不用想，它的应用前景非常被看好，除了各个大公司的支持外，最重要的它是在内核中，免费啊。


在《High-Performance Networking Using eBPF, XDP, and io_uring》，提出了使用IO_URING和XDP以及eBPF一起来实现一个高性能的网络，也就是说，这些技术间其实是互相配合，共同合作为主的。当然，DPDK可能有一点小小小的冲突，这就看大家的实际场景中到底想怎么做了。不过一个普遍比较被大家接受的观点，DPDK还是有一些复杂。

# 参考
```bash

# [译] [论文] XDP (eXpress Data Path)：在操作系统内核中实现快速、可编程包处理（ACM，2018）
https://arthurchiao.art/blog/xdp-paper-acm-2018-zh/

# [译] 深入理解 Cilium 的 eBPF 收发包路径（datapath）（KubeCon, 2019）
https://arthurchiao.art/blog/understanding-ebpf-datapath-in-cilium-zh/#step-2xdp-%E7%A8%8B%E5%BA%8F%E5%A4%84%E7%90%86


# [译] Cilium：BPF 和 XDP 参考指南（2021）
https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/

# [译] 基于 BPF/XDP 实现 K8s Service 负载均衡 (LPC, 2020)
https://arthurchiao.art/blog/cilium-k8s-service-lb-zh/

# [译] Cilium：BPF 和 XDP 参考指南（2021）
https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/

# 网络编程之XDP、TC和IO_URING以及DPDK
https://blog.csdn.net/fpcc/article/details/139892982

# 网络编程之XDP技术的基础eBPF
https://blog.csdn.net/fpcc/article/details/139879107
```