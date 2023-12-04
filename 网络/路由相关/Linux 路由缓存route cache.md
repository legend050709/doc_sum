```table-of-contents
```
# 概述
**3.6**版本一定算得上是**Linux**网络子系统中一个特别的版本, 这个版本(补丁[patch](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=89aef8921bfbac22f00e04f8450f6e447db13e42))移除了查找**FIB**之前的缓存查找。本文就来谈谈路由缓存的前世今生。

# 基本概念
## 路由
将**skb**按照**规则**送到该去的地方；这个地方可能是本机，也可能是局域网中的其他主机,或者更远的主机。从这个角度来说，它一个**动词**。
那么**路由**发生在哪个时候呢？ 
我们知道**路由**是网络层(**L3**)的概念。
接收方向，它需要决定收到的**skb**是应该**上送本机**还是**转发**；
发送方向，它需要决定**skb**从哪个网络接口发出。

下图原本是描述`Netfilter`在内核中的钩子位置的,但我觉得用来说明**路由**的位置也是比较合适的。
![](attachments/Pasted%20image%2020231129191832.png)

与此同时，**路由**也可以特指上面所说的**规则**,这是**名词**的用法。
**路由**从哪来？ 
一般来说有三个来源：
1. 用户主动配置；
2. 2.内核生成； 
3. 其他一些路由协议进程(`OSPF`、`BGP`)生成。

普通主机上可能没有最后一种，所以，为了理解方便，你可以将**路由**就理解为你用`route`命令看到的内容。
```
[root@tristan]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.99.0    0.0.0.0         255.255.255.0   U     0      0        0 eth0
192.168.98.42   192.168.99.1    255.255.255.255 UGH   0      0        0 eth0
127.0.0.0       0.0.0.0         255.0.0.0       U     0      0        0 lo
0.0.0.0         192.168.99.254  0.0.0.0         UG    0      0        0 eth0
```

## FIB
`FIB`：全称是(**F**orwarding **I**nformation **B**ase)，翻译过来就是**转发信息表**。
**内核会将路由翻译成FIB中的表项**。我们习惯说的**查询路由**，对于内核来说，应该叫**查询FIB**。

# 为什么要route cache
**缓存**无处不在。现代计算机系统中，**Cache**是**CPU**与内存间存在一种容量较小但速度很高的存储器，用来存放`CPU`刚使用过或最近使用的数据。
**路由缓存就是基于这种思想的软件实现**。

- **查询原则**
> 内核查询**FIB**前，固定先查询**cache**中的记录，如果**cache**命中(**hit**)，那就直接用就好了，不必查询**FIB**。
> 如果没有命中(**miss**), 就回过头来查询**FIB**，最终将结果保存到**cache**，以便下次不再需要需要查询**FIB**。

# route cache的特点
缓存是精确匹配的, 每一条缓存表项记录了匹配的源地址和目的地址、接收发送的`dev`，以及与内核邻居系统(**L2**层)的联系(`negghbour`)。

区别：
`FIB`中存储的也就是路由信息，它常常是范围匹配的，比如像`ip route 1.2.3.0/24 dev eth0`这样的网段路由。

# 内核中的route cache
## 3.6版本以前的路由缓存
下图是**3.6**版本以前的本机发送**skb**的路由过程….
![](attachments/Pasted%20image%2020231129192025.png)

看上去的确可能能提高性能! 只要**cache**命中率足够高。
要获得高的**cache**命中率有以下两个途径：
1. 存储更多的表项; 
2. 存储更容易命中的表项

缓存中存放的表项越多，那么目标报文与表项匹配的可能性越大。
但是**cache**又不能无限制地增大，**cache**本身占用内存是一回事，更重要的是越多的表项会导致查询**cache**本身变慢。使用**cache**的目的是为了加速，如果不能加速，那要这劳什子有有什么用呢？

### route cache的匹配
前面说了，**cache**的特点决定了它只能做**精确匹配**。也就是说，只有目标数据报文与**cache**中的表项完全一致，才算匹配成功。最简单的**cache**查找过程应该是下面这样：遍历**cache**中的所有表项，直到遇到匹配的表项跳出循环。
```
foreach entry in cache:
then
    if entry match skb
    then
        /* 条件匹配，将缓存表项中记录的结果设置到skb上 */
        skb->dst <= entry->dst
        return
    endif
end
```
显然，**cache**表项的数目越多，那么查找的过程就越长! 当然，内核不会这么蠢地将所有**cache**拉成一个线，而是使用**hash**桶，看上去应该是这么一个结构。
![](attachments/Pasted%20image%2020231129192239.png)
内核首先根据目标报文的一些特征计算**hash**，找到对应的**hash**冲突链表。在链表上一个一个地进行比较遍历。

### route cache的管理
为了避免**cache**表项过多，内核还会在**一定时机**下清除**过期**的表项。
有两个这样的**时机**：
1. 添加新的表项时，如果冲突链的表项过多，就删除一条已有的表项；
2. 内核会启动一个专门的定时器周期性地**老化**一些表项.

### 该版本 route cache的缺陷
获得更高的**cache**命中率的第二个途径是**存储更容易命中的表项**
什么是更容易命中的呢？ **那就是真正有效的报文**。
遗憾的是，内核一点也不聪明：
> 只要输入路由系统的报文不太离谱，它就会生成新的缓存表项。坏人正好可以利用这一点，不停地向主机发送垃圾报文，内核因此会不停地刷新**cache**。
> 这样每个**skb**都会先在**cache表**中进行搜索，再查询**FIB表**，最后再创建新的**cache表项**，插入到**cache表**。这个过程中还会涉及为每一个新创建的**cache表项**绑定邻居，这又要查询一次`ARP`表。

要知道，一台主机上的路由表项可能有很多，特别是对于网络交换设备，由OSPF或者BGP等路由协议动态下发的表项有上万条是很正常的事。而邻居节点却不可能达到这个数量。
对于转发或者本机发送的skb来说，路由系统能帮它们找到下一跳邻居就足够了。

#### 解决思路
如果你的系统配置了大量的路由表项，其中只有少量的路由项是你的数据发出时要理由的(WEB服务器更多的面临这种情况)。
不幸的是，最长前缀匹配算法无法保证拥有最大流量的那个连接的发送方向路由查找以最快的速度返回，因此还是需要一个cache系统来保证这一点，而我的方式就是，将路由项保存在已经确认有效的socket结构体中。值得注意的是，此优化对于forward数据包的路由无效，因为forward数据无法跟一个socket建立关联。此优化最大的用武之地是，你的机器提供对外的服务，别人经常从你的机器下载数据，这样这个优化的收益是非常可观的，因为每一个数据包的路由查找开销都被节省了下来！

> 顺便说一下，在现有的 Linux 协议栈实现中，已经有`sk_dst_set/get`这类API将一个socket与一个路由项关联，然而我看来，它只是指示了一些经由该路由发出的数据包的“链路特征”，比如`MTU，MSS，RTT`之类，并没有使用它来免除最长前缀匹配的路由查找。

### 小结
总结起来就是，**3.6**版本以前的这种路由缓存在**skb**地址稳定时的确可能提高性能。但这种根据**skb**内容决定的性能却是不可预测和不稳定的，即系统并不知道这些被cache住的路由是不是有效的路由。
> 有效的路由就是那些真正为本机所用的路由。


## 3.6版本及其以后的路由缓存
正如前面所说，**3.6**版本移除了**FIB**查找前的路由缓存，取而代之的是下一跳缓存。
这意味着每一个接收以及发送的**skb**现在都必须要进行**FIB**查找了。这样的好处是现在查找路由的代价变得**稳定(consistent)**了。

路由缓存完全消失了吗? 
> 并没有！在**3.6**以后的版本, 你还可以在内核代码中看到**dst_entry**。这是因为，**3.6**版本实际上是将**FIB**查找缓存到了**下一跳(fib_nh)**结构上，也就是**下一跳缓存**。即 **route cache 变为 nh cache（nexthop cache）**。

### 为什么需要缓存下一跳
我们可以先来看下没有下一跳缓存的情况。以**转发forward** 过程为例，相关的伪代码如下：
```
FORWARD:

fib_result = fib_lookup(skb)
dst_entry  = alloc_dst_entry(fib_result)
skb->dst = dst_entry;

skb->dst.output(skb)   
nexthop = rt_nexthop(skb->dst, ip_hdr(skb)->daddr)
neigh = ipv4_neigh_lookup(dev, nexthop)
dst_neigh_output(neigh,skb)
release_dst_entry(skb->dst)
```
内核利用**FIB**查询的结果申请**dst_entry**, 并设置到**skb**上，然后在发送过程中找到下一跳地址，继而查找到**邻居**结构(查询**ARP**)，然后邻居系统将报文发送出去，最后释放**dst_entry**。

下一跳缓存的作用
> 尽量减少最初和最后的申请释放**dst_entry**，它将**dst_entry**缓存在下一跳结构(**fib_nh**)上。


这和之前的路由缓存有什么区别吗？
> 很大的区别！之前的路由缓存是以源IP和目的IP为KEY，有千万种可能性，而现在是和下一跳绑定在一起，一台设备没有那么多下一跳的可能。这就是下一跳缓存的意义！

### 提前分流（早期解复用） early demux

`early demux`是在**skb**接收方向的加速方案。
如前面所说，在取消了**FIB**查询前的路由缓存后，每个**skb**应该都需要查询**FIB**。

**early demux**是基于一种思想：如果一个**skb**可以匹配本机某个应用程序的套接字，那么我们可以将路由的结果缓存在内核套接字结构上；下次同样的报文(四元组)到达后，我们可以在**FIB**查询前就将报文提交给上层，也就是**提前分流(early demux)**。
![](attachments/Pasted%20image%2020231129193628.png)

#### 详细说明
`ip_rcv_finish` 负责为 sk_buff 从 IP Route System 中找到路由目标。
如果是路由到本机，则在下一个处理这个 sk_buff 的协议内(比如上层的 TCP/UDP 协议)还需要从 sk_buff 中找到对应的 socket。也就是说每个收到的数据包都会有两次 demux 工作。
> 解多路复用：
> 一次找到这个数据包该路由到哪里
> 一次是如果路由到本机需要将数据包路由到对应的 Socket。

但是对于类似 TCP 这种协议当 socket 处在 ESTABLISHED 状态后，协议栈不会出现变化，后来的数据包的路由路径跟握手时数据包的路由路径完全相同，所以就有了 Early Demux 机制。用于在收到数据包的时候根据 IP Header 中的 protocol 字段找到上一层网络协议，用上一层网络协议来解析数据包的路由路径，以减少一次查询。

> 拿 TCP 来说，简单来讲就是收到数据包后去 TCP 层查找这个数据包有没有对应的处在 ESTABLISHED 状态的 Socket，有的话直接使用这个 Socket 已经 Cache 住的路由目标作为当前 Packet 的路由目标，从而不用再查找 IP Route System，因为根据 Packet 查找 Socket 是怎么都省不掉的。

需要补充一下 Early Demux 对 Socket 还未处在 ESTABLISHED 状态的 TCP 连接无效。
这就导致这种数据包不但会查一次 IP Route System 还会到 TCP ESTABLISHED 连接表中查一次，之后路由到 TCP 层又要再查一次 Socket 表。总体开销就会比只查一次 IP Route System 还要大。
所以 Early Demux 并不是无代价的，只是大多数场景可能开启后会对性能有提高，所以 Linux 默认是开启的。
但在某些场景下，目前来看应该是大量短连接的场景，连接要不断建立断开，有大量的数据包都是在 TCP ESTABLISHED 表中查不到东西，这个机制开启后性能会有损耗，所以 Linux 提供了关闭该机制的办法：
```c
sysctl -w net.ipv4.ip_early_demux=0
```


#### 小结
如果开启了 ip_early_demux（早期解复用），这是一项优化，为了 TCP 和 UDP 可以提前获得 skb 的 dst_entry（目标入口）；

- 当 skb 为 TCP 报文并且开启了 tcp_early_demux 选项
>则调用 tcp_v4_early_demux 函数，根据 skb 的源地址、目的地址等信息从 ESTABLISHED 连接列表中找到对应的 Socket，把 Socket 中缓存的 sk_rx_dst（struct dst_entry）设置到 skb->dst 中。还会将 Socket 的 struct sock 指针设置到 skb->sk，这样 TCP 层就不用重复查连接列表了；
- 当 skb 为 UDP 报文并且开启了 udp_early_demux 选项
>则调用 udp_v4_early_demux 函数，拿 skb 的 UDP 头信息在 UDP 「解复用表」中寻找 Socket，如果有，把 Socket 中缓存的 dst_entry 设置到 skb->dst；

如果没开启 ip_early_demux 或者开启了上步中没有完成对 skb->dst 的设置，那么就需要调用 ip_route_input_noref 函数去「路由子系统」查询来获得 skb 的 dst_entry，这个过程比较复杂。

Early Demux（早期解复用）和查询 IP Route System（路由子系统）目的都是为了设置 skb->dst，如果 skb 是发给本机器，那么 Early Demux 和查询 IP Route System 获得的 dst_entry 会是同一个函数 ip_local_deliver；如果不是本机器，那么会转发出去。


## 总结
3.6版本内核之前的路由cache，其思想并没有错，错在这个cache放错了地方以至于可能会成为众矢之的而被DDos。
**3.6**版本将**FIB**查询之前的路由缓存移除了，取而代之的是下一跳缓存。
 
# 其他
## route cache查看
`/proc/net/rt_cache`文件存储`route cache`信息，但是ip地址是使用十六进制来表示的，所以看起来很不方便。

```
route -Cn
or 
ip -s route show table cache
or 
ip -s route get x.x.x.x
or
netstat -rn --cache
	-r: Display the kernel routing tables.
	--cache/-C: Print routing information from the route cache.
```

```c
ip -s route show cache 192.168.100.17
显示来自路由缓存的统计信息
```
![](attachments/Pasted%20image%2020231129203321.png)
![](attachments/Pasted%20image%2020231129203341.png)


如下所示：
client 给 DPVS发送 UDP的大包(UDP载荷1470)，经过DPVS之后，DPVS需要进行IPIP Tun封装(外层加一个IP头)，导致超过1500. DPVS给clinet回复一个Icmp need-frag的包。
```
# client抓包
# tcpdump -nni eth04 host 192.22.2.28 or icmp or icmp6
11:35:39.419766 IP 192.20.1.11.47931 > 192.22.2.28.6000: UDP, length 1470
11:35:39.419840 IP 192.22.2.28 > 192.20.1.11: ICMP 192.22.2.28 unreachable - need to frag (mtu 1480), length 556
```
client中查看route cache 如下所示：
![](attachments/Pasted%20image%2020231130115208.png)
> 如上所示，收到icmp need-frag的差错报文， route cache 中的mtu 设置为 1480.
 
- **route cache 的 mtu 的作用**：
> 对于TCP：route cache的 mtu 影响 同一个 sip, dip 的后续包的 mss 协商。
> 对于UDP：route cache的 mtu 影响 同一个 sip,dip的后续包是否分片。


![](attachments/Pasted%20image%2020231130152514.png)
如上所示， icmp need-frag 生成的 route cache中的mtu 影响了后续流的发送。后续流的包的MSS调整了。
但是从上面抓包看，后续流的MSS调整了，但是抓包发现后续发送的包的大小还是超过了MSS(1440)。这个是为什么呢？
> 抓包发现TCP发送的数据大小大于协商的MSS，主要原因是网卡的TSO。即TCP的分段在TSO中进行。tcpdump抓包此时还看不到TCP的分段。如下所示，关闭了网卡的TSO之后`ethtool -K eth04 tso off`，就可以看到后续发送的TCP包的大小小于MSS。
![](attachments/Pasted%20image%2020231130154337.png)
## route cache统计
`/proc/net/stat/rt_cache`

## route cache 超时时间
`net.ipv6.route.gc_timeout` 以及 `net.ipv4.route.gc_timeout`

`net.ipv4.route.gc_timeout` 参数定义了内核在删除未使用的路由缓存条目前等待的时间。

## 清理route cache
`ip route flush cache`，它可以清除 IP 路由缓存。
![](attachments/Pasted%20image%2020231130121249.png)
## route cache的调优
参考：[Tuning Linux IPv4 route cache](https://vincent.bernat.ch/en/blog/2011-ipv4-route-cache-linux)


# 参考
```c
# Linux 路由缓存的前世今生
https://switch-router.gitee.io/blog/routecache/

# Linux 3.6版本内核后关于路由cache的一个优化【dog250】
https://blog.csdn.net/dog250/article/details/51945909

# Tuning Linux IPv4 route cache
https://vincent.bernat.ch/en/blog/2011-ipv4-route-cache-linux
```