```table-of-contents
```
# 介绍
ARP 的全称是 Address Resolution Protocol，直译过来是 **地址解析协议**。对应的 RFC 文档是 [RFC826](https://tools.ietf.org/html/rfc826)。它的作用是把 IP 地址转换为 MAC 地址。

为什么需要做这件事呢？
这是因为 TCP/IP 网络协议栈是分层的，每层负责不同的功能。IP 层（layer 3）负责路由寻路，换句话说，如果目的机器和客户端不在同一个网络，IP 层会穿过错综复杂的中间网络（互联网）找到目的机器所在的网络。

当报文在某一个网络中传播时（可能源机器和目的机器本来就在同一个网络，也可能报文在路由过程中执行下一跳步骤），IP 层的功能就没有用了，这时候起作用的是 2 层网络（链路层），大多数情况下就是以太网。以太网负责把多个机器连到一起，组成一个最小单位的局域网。在以太网中，不同机器的标识是 MAC 地址，MAC 地址是机器在生产的时候厂商为机器设定的。可以使用 `ip link` 命令查看网卡的 MAC 地址。

有了 MAC 地址，同一个以太网络上的两台机器才能够通信。

# ARP 缓存
如果每次主机通信都要发送 ARP 协议去查询 MAC 地址无疑是低效的，为了提供性能最常见的做法是在系统层面保存 ARP 的结果。因为 IP 地址和 MAC 地址并不会经常变化，而且主机间第一次通信之后有很大的可能性会在短时间内再次通信， 所以 ARP 缓存能够大大提高网络效率。

在 Linux 系统中，可以通过 `ip neighbour`（可以简写为 `ip neigh`） 命令管理 ARP 缓存。

## arp状态
这个表项表明 `172.16.13.18` 对应的 MAC 地址是 `50:7b:9d:e0:8b:a8`，`dev enp0s25` 是说发送给 `172.16.13.18` 的报文需要通过这个 interface，`REACHABLE` 表明目标地址是可达的
```
sudo ip neigh
172.16.13.18 dev enp0s25 lladdr 50:7b:9d:e0:8b:a8 REACHABLE
```

其他状态还包括：

- `permanent`：这个 ARP 缓存项没有过期时间，会一直生效，直到管理员把它删除
- `noarp`：这个 ARP 表项是有效的，不需要验证它的真实性，但是它有过期时间，过期之后会被自动删除
- `stale`：ARP 表项虽然有效，但是可疑，需要发送验证 ARP 报文
- `failed`：该表项不可用，ARP 请求报文没有应答

![](attachments/Pasted%20image%2020231106153422.png)


# ARP 工作过程
**第一步**：机器 A 想和同一个以太网络的机器 B 通信，A 会现在自己的 ARP 表中查找 B 的 MAC 地址，如果能找到就直接发送以太网帧；如果没有找到，就跳到第二步  
**第二步**：机器 A 发送 ARP 请求报文去查询机器 B 的 MAC 地址，这是个以太网广播报文，因此交换机会广播到网络中所有的机器  
**第三步**：各个主机接收到 ARP 请求报文，如果发现 ARP 报文中询问的 IP 地址和自己不同，则直接丢弃；机器 B 发现 ARP 报文中询问的 IP 地址是自己的主机地址，则生成一个 ARP 应答报文，把自己的 MAC 地址填到报文中，发送给机器 A，这个报文是单播报文，不会发送给其他主机；同时机器 B 也会把机器 A 的 ARP 记录缓存起来  
**第四步**：机器 A 接收到 B 发来的 ARP 应答，读取报文中 B 的 MAC 地址，使用这个 MAC 地址和机器 B 进行后续的通信，同时把它缓存到系统中

从机器 A `ping` 机器 B 时，用 `wireshark` 抓包，过滤出其中的 ARP，显示的结果如图：
![](attachments/Pasted%20image%2020231106153835.png)
可以看到一共有两对 ARP 请求应答的过程，我们来逐个分析。

## ARP请求
首先是机器 A 发送的 ARP 请求，详细的报文内容如下：
![](attachments/Pasted%20image%2020231106153905.png)

这个以太网帧的源地址是 `50:7b:9d:ca:08:f0`，也就是机器 A 的 MAC 地址；目的地址是 `ff:ff:ff:ff:ff:ff`，这是一个以太网广播地址，**交换机**会把报文发送给网络中所有的主机。
以太网帧类型（Type）字段的值是 `0x0806`，表示该数据帧是个 ARP 请求或者应答。
帧的长度是 42，其中包括 14 字节以太网帧头部，以及 28 字节 ARP 请求数据。

以太网帧内部是 ARP 请求的具体内容，`Opcode` 为 1，表示这是 ARP 请求，发送端（也就是机器 A）的 IP 地址是 `172.16.13.16`，目的端（也就是机器 B）的 IP 地址是 `172.16.13.18`。发送端的 MAC 地址是填写的，而目的端 MAC 地址为空（全部为 0）。

## ARP应答
第二个报文是 ARP 应答，详细报文内容如下图：
![](attachments/Pasted%20image%2020231106154006.png)

和上个报文不同，这是一个单播报文，直接发送到机器 A，帧的目的 MAC 地址是 `50:7b:9d:ca:08:f0`。因为机器 B 能够从接收到的 ARP 请求报文中获取机器 A 的 IP 地址和 MAC 地址，可以直接发送单播。

`Opcode` 字段的值为 2，表示这是个 ARP 应答报文，而且 ARP 报文中四个地址都填上了对应的值（因为此时机器已经知道通信双方的所有地址）。

另外一点需要注意的是，这个以太网帧的长度是 60 字节，因为它的以太网帧后面加了 18 字节的 padding（填充字符）。**这是以太网帧的最小长度值.**

那为什么 ARP 请求报文的长度可以低于 60 呢？
>这和 wireshark 抓包的原理有关。发包的时候，wireshark 捕获的包并不是真正发送到线路上的帧，而是发送给网卡驱动的数据。因此，如果从机器 B 上抓包，会发现机器 B 接收到的这个 ARP 请求长度也是 60 字节。

## ARP cache validation 报文
后两个 ARP 报文也是一对：请求和应答，它们是从机器 B 发来查询机器 A MAC 地址的。
但是比较奇怪的是，ARP 请求报文以太网帧的目的地址居然不是广播，而是机器 A 的 MAC 地址？为什么已经知道对方的 MAC 地址还要发送 ARP 报文去查询呢？

注意到，第三个报文距离上一个的时间间隔是 5s 左右，真正的 ICMP 报文（ping 应用传输的报文）已经在这个时间间隙中发送，也就是说机器 B 已经通过 A 发送的 ARP 报文把 ARP 保存到自己的缓存中了。第三个奇怪的 ARP 报文是为了**验证缓存的有效性**！

ARP 缓存都会有有效期，在 Linux 实现中，它会验证 ARP 缓存的有效性，并更新缓存记录的状态。因为已经知道对方的 ARP 记录，所以就没有必要再通过广播机制造成额外的网络资源浪费，这个 ARP 请求可以理解为： **请问 172.16.13.16，你的 MAC 地址还是 50:7b:9d:ca:08:f0吗？**。如果收到应答，说明该缓存记录有效；如果一直没有收到应答，则需要把缓存记录设置为失效或者删除，并在需要通信的时候使用正常的 ARP 请求获取对方的 MAC 地址。

我们可以在 `RFC1122` 的 `2.3.2.1` 小节（ARP Cache Validation）找到对应的文档说明：

```
Unicast Poll – Actively poll the remote host by periodically sending a point-to-point ARP Request to it, and delete the entry if no ARP Reply is received from N successive polls. Again, the  
timeout should be on the order of a minute, and typically N is 2.
```
**NOTE**：不同系统实现 ARP 缓存的机制不同，管理缓存有效期的方法也会有差异，并不能假设所有的系统都会使用上面的方法来验证缓存的有效性。

## Gratuitous ARP
除了标准的 ARP 之外，还有一种特殊的 ARP 报文，称为 Gratuitous ARP（免费 ARP）。这个报文也是广播报文，它的特殊性在于，它的报文中发送端 IP 地址和接收端 IP 地址都被设置为发送该报文的主机 IP。

为什么要有这样一个特殊的报文呢？因为它有用，比如：

- 检测 IP 冲突。如果免费 ARP 请求接收到应答，说明当前网络上有另外一个和发送机器有相同 IP 的主机
- 可以用来更新网络中当前机器的 ARP 缓存。如果机器重新配置了 IP 地址，那么免费 ARP 报文能够把新的 IP-MAC 匹配关系广播到网络中，接收到报文的机器更新自己的 ARP 缓存记录，这样就不会有因为 ARP 缓存失效导致的网络问题

如果机器 A 重新配置了 IP 地址，那么 的对应关系就发生了变化，网络中保存的旧 ARP 表项都失效，无法继续使用，会导致 ping 错误。Linux 系统中可以使用 `arping` 命令行来发送 Gratuitous ARP，让网络中所有主机更新当前机器的 ARP 记录：
```
arping -A -I eth0 172.16.42.161

这个命令就是把机器上 `eth0` 绑定的 MAC 地址和 `172.16.42.161` 作为 ARP 记录发送出去。
```

# /proc/sys/net/ipv4/neigh目录
如下所示：/proc/sys/net/ipv4/neigh/default 目录下的各个配置。
![](attachments/Pasted%20image%2020231027174848.png)

## arp_ignore：收到arp请求是否响应
### 介绍
Linux [内核文档](https://www.baidu.com/link?url=2x1snmPs3FowXF4IgsntmrWh6TbFiojKKvF3lTRYe8K5oci44n83yIJvXRGJz-0V0rXxUhmqkaeWUBT7C_7AE7u-Vq2YlwBb-PkUhodVa5C&wd=&eqid=a1e7ed7e0000f7730000000359a423c3) 中的描述：
![](attachments/Pasted%20image%2020231106163909.png)

arp_ignore 参数的作用是控制系统在收到外部的 ARP 请求时，是否要返回 ARP 响应。参数常用到的有 0，1，2 三个值，3 ~ 8 较少用到：

- 0：响应任意网卡上接收到的对本机IP地址的 ARP 请求（包括环回网卡上的地址），而不管该目的 IP 是否在接收网卡上。
- 1：只响应目的IP地址为接收网卡上的本地地址的 ARP 请求。
- 2：只响应目的IP地址为接收网卡上的本地地址的 ARP 请求，并且发送 ARP 请求的源IP必须和接收网卡同网段。
- 3：如果 ARP 请求数据包所请求的IP地址对应的本地地址其作用域（scope）为主机（host），则不回应 ARP 响应数据包，如果作用域为全局（global）或链路（link），则回应 ARP 响应数据包。
- 4~7：保留，未使用。
- 8：不回应所有的 ARP 请求

### 注意
`/etc/sysctl.conf` 中包含 all 和 eth/lo（具体网卡）的 arp_ignore 参数，取其中较大的值生效。

> 注：ipv6 不使用ARP，而是使用 ND 协议，不需要设置 arp_ignore 相关的参数。

### 示例
![](attachments/Pasted%20image%2020231106164315.png)
当 arp_ignore 参数配置为 0 时，eth0 网卡上收到目的IP为环回网卡IP的 ARP 请求，但是 eth0 也会返回 ARP 响应，并且把自己的 MAC 地址告诉对端。

![](attachments/Pasted%20image%2020231106164557.png)
当 arp_ignore 参数配置为 1 时，eth0 网卡上收到目的IP为环回网卡IP的 ARP 请求，发现请求的IP不是自己网卡（eth0)上的IP，就不会返回 ARP 响应。

### DR 模式下的应用

因为 DR 模式下，每个真实服务器节点都要在**环回网卡**上绑定VIP。
如果客户端对于VIP的 ARP 请求广播到了各个真实服务器节点，如果 arp_ignore 参数配置为 0，则各个真实服务器节点都会响应该 ARP 请求，此时客户端就无法正确获取 LVS 节点上正确的虚拟服务IP所在网卡的 MAC 地址。

假如某个真实服务器节点 A 的网卡 eth1 响应了该 ARP 请求，客户端把 A 节点的 eth1 网卡的 MAC 地址误认为是 LVS 节点的虚拟服务IP所在网卡的 MAC ，从而将业务请求消息直接发到了 A 节点的 eth1 网卡。这时候虽然因为 A 节点在环回网卡上也绑定了VIP，所以 A 节点也能正常处理请求，业务暂时不会受到影响。但时此时由于客户端请求没有发到 LVS 的VIP上，所以 LVS 的负载均衡能力没有生效。造成的后果就是，A节点一直在单节点运行，业务量过大时可能会出现性能瓶颈。
所以要求 arp_ignore 参数要求配置为1。
配置 `/etc/sysctl.conf` ，然后使用 `sysctl -p` 刷新到内存即可立即生效：
```c
net.ipv4.conf.all.arp_ignore = 1
```


## arp_announce： 发送arp请求sip的选择
### 介绍
`arp_announce` 参数是定义 Linux 主机发送 ARP 请求数据包时如何选择数据包中使用的发送方 IP 地址（即 Sender IP address）。

![](attachments/Pasted%20image%2020231106164133.png)

**背景**
在系统准备通过网卡发送一个 IP 数据包前，该 IP 数据包的源 IP 地址和目标 IP 地址通常是已经知道的，同时发送的网卡也已经确定，那数据链路层的源 MAC 地址当然也确定了，最后剩下的就是确定数据链路层的目标 MAC 地址了，而该目标 MAC 地址就需要通过 ARP 地址解释协议来获取，于是系统首先需要发送 ARP 请求数据包获取目标 MAC 地址。

**取值**
我们知道，发送 ARP 请求数据包时需要包含发送方 IP 地址，该 IP 地址应该是什么呢？
大家可能想当然的以为就是要发送的 IP 数据包的源 IP 地址，其实这个是不一定的，尤其是主机有多个网络接口和 IP 地址时，而 `arp_announce` 正是控制该发送方 IP 地址的选择条件的。
arp_announce 参数常用的取值有 0，1，2：

- 0：允许使用任意网卡上的IP地址作为 ARP 请求的源IP，通常就是待发送数据包的源IP。
- 1：尽量避免使用不和目的地址同网段的本地地址作为发送 ARP 请求的源IP地址。
- 2：忽略IP数据包的源IP地址，选择该发送网卡上最合适的本地地址作为 ARP 请求的源IP地址（一个网口可能会配置多个IP地址）。

### 注意

`/etc/sysctl.conf` 中包含 all 和 eth/lo（具体网卡）的 arp_announce 参数，取其中较大的值生效。

> 注：ipv6 不使用ARP，而是使用 ND 协议，不需要设置 arp_ignore 相关的参数。

### 示例
![](attachments/Pasted%20image%2020231106164902.png)

当 arp_announce 参数配置为 0 时，系统要发送的IP包源地址为 eth1 的地址，IP包目的地址根据路由表查询判断需要从 eth0 网卡发出，这时会先从 eth0 网卡发起一个 ARP 请求，用于获取目的IP地址的 MAC 地址。该 ARP 请求的源MAC 自然是 eth0 网卡的 MAC 地址，但是源 IP 地址会选择 eth1 网卡的地址。


![](attachments/Pasted%20image%2020231106164910.png)

当 arp_announce 参数配置为 2 时，eth0 网卡发起arp请求时，源IP地址会选择 eth0 网卡自身的IP地址。

### DR 模式下的应用
每个机器或者交换机中都有一张 ARP 表，该表用于存储对端通信节点IP地址和 MAC 地址的对应关系。当收到一个未知IP地址的 ARP 请求，就会在本机的 ARP 表中新增对端的IP和 MAC 记录；当收到一个已知IP地址（ ARP 表中已有记录的地址）的 ARP 请求，则会根据 ARP 请求中的源 MAC 刷新自己的 ARP 表。

如果 arp_announce 参数配置为0，则网卡在发送 arp 请求时，可能选择的源IP地址并不是该网卡自身的IP地址，这时候收到该 ARP 请求的其他节点或者交换机上的 ARP 表中记录的该网卡IP和 MAC 的对应关系就不正确，可能会引发一些未知的网络问题，存在安全隐患。

所以要求 arp_announce 参数要求配置为2。

配置 `/etc/sysctl.conf` ，然后使用 `sysctl -p` 刷新到内存即可立即生效：
```c
net.ipv4.conf.all.arp_announce = 2
```
## arp_filter
arp_filter 和 arp_ignore 类似， 更多的使用 arp_ignore。

### arp_filter 和 arp_ignore 区别
arp_ignore=1开销小，可实现各网卡响应各自ip的arp请求。 arp_filter=1开销大一些（查路由表），且只解决重复arp响应问题，难以实现各网卡响应各自ip的arp请求。

`arp_filter=1`含义：sip的路由出口为本网卡才进一步处理。`arp_filter=1`主要用来防攻击，比如可过滤sip不规范的arp报文，如跨网段的arp报文。arp_filter是源地址校验，是rp_filter的arp版本的补充。 
`arp_ignore=1`含义：dip为本网卡ip才响应。


## arp_notify
ARP 通知链操作
- 0：不做任何操作    
- 1：当设备up/down或硬件地址(mac地址)改变时自动产生一个 ARP 请求

## arp_accept

## proxy_arp

## 配置arp
# 查看
## ip neigh查看arp表项
## /proc/net/stat/arp_cache说明
ARP的缓存时间约10分钟。
APR缓存列表没有对方的MAC地址或缓存过期的时候，会发送ARP请求获取MAC地址，在没有获取到MAC地址之前，用户发送出去的UDP数据包会被内核缓存到arp_queue这个队列中，默认最多缓存3个包，多余的UDP包会被丢弃。
被丢弃的UDP包可以从/proc/net/stat/arp_cache的最后一列的unresolved_discards看到。
当然我们可以通过echo 30 > /proc/sys/net/ipv4/neigh/eth1/unres_qlen来增大可以缓存的UDP包。

# ARP 的缺陷

ARP 协议很简单，通过缓存机制和 Gratuitous ARP 能够提供便利和高效的地址解析功能。尽管如此，和所有的网络协议一样，它并不是完美的，根据上面的解析，我们知道能发现它的两个不足之处：

1. ARP 报文没有任何认证，假设所有的机器都可靠而且诚信，所以 ARP 报文（尤其是应答）可以伪造。（窃听报文，或者报文转发）-> ARP spoofing
2. ARP 报文没有状态，机器可以在没有收到请求的时候直接发送应答。虽然特性能够有效地来更新 ARP 记录，也可能被恶意利用


# 参考
```c
# [Linux内核参数之arp_ignore和arp_announce]
https://www.cnblogs.com/lipengxiang2009/p/7451050.html

https://syxdevcode.github.io/2021/03/01/Linux%E4%B8%8B%E7%BD%91%E7%BB%9C%E4%B8%A2%E5%8C%85%E6%95%85%E9%9A%9C%E5%AE%9A%E4%BD%8D/
【查看arp相关部分】

# arp_ignore=1与arp_filter=1之争
https://zhuanlan.zhihu.com/p/661896100

# ARP 协议解析
https://cizixs.com/2017/07/31/arp-protocol/
```