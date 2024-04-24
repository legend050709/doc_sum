```table-of-contents
```
# 介绍
Ip Tunnel，又叫IP隧道，顾名思义，LVS通过在IP数据包外面再封装了一层Ip Tunnel 头部，将数据包的源地址改写为LVS自身的 IP地址，目的地址改写为RS的 IP地址，从而实现跨网段访问RS。整个过程看起来好像LVS和RS之间有一条隧道，数据包通过这条虚拟的隧道进行传输。  
如下图所示：
![](attachments/Pasted%20image%2020231023145216.png)

Ip Tunnel模式下，客户端的请求包到达负载均衡器的VS后，负载均衡器不会改写请求包的IP和端口，但是会在数据包IP层外面再封装一个IP层，然后将数据包转发；真实服务器收到请求后，会先将外面封装的Ip Tunnel头去掉，然后处理里面实际的请求报文；与DR模式类似，响应包也不再经过LVS，而是直接返回给客户端。

# 场景
- 回向流量不经过KGW
减轻KGW的压力，以及数量。特别适用于下载业务，CDN回源业务。

- 相比DR，具有跨网络的优势
Ip Tunnel模式的性能比不上DR，那为什么还要用它呢？ 因为它可以跨网段转发！

# 配置
## RS配置
### 加载ipip模块。

```undefined
modprobe ipip
```
### 创建虚拟口
创建并启动tunl0虚拟网卡

```undefined
ip tunnel add xxx type ipip lcoal xxxx remote xxxx
ifconfig xxx up
```

### vip绑定
将vip绑定在tunl0网卡上。（注意这里也是与DR模式不同的地方，DR模式是将vip绑在LO网卡）

```csharp
ip addr add vip dev tunl0
```

### 设置内核参数

```ruby
echo "0" > /proc/sys/net/ipv4/ip_forward
echo "2" > /proc/sys/net/ipv4/conf/all/arp_announce
echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore
echo "0" > /proc/sys/net/ipv4/conf/all/rp_filter
echo "0" > /proc/sys/net/ipv4/conf/tunl0/rp_filter
```
#### arp_ignore
**背景**
默认 情况下，`/proc/sys/net/ipv4/conf/all/arp_ignore = 0`，即：响应任意网卡上接收到的对本机IP地址的 ARP 请求（包括环回网卡上的地址），而不管该目的 IP 是否在接收网卡上。

**问题场景**
假如，VIP地址和RS的RIP地址在同一个网段，且在同一个TOR下挂载，那么交换机在转发DIP为VIP的流量时，如果没有ARP表项，就会发送ARP广播。那么LB以及RS都会收到VIP的ARP请求。
在RS上，VIP配置在隧道口，但是是物理口收到的ARP请求，如果 `arp_ignore = 0`，那么就可以响应 ARP 请求。那么有可能引流DIP为VIP的流量到RS上，那么LB就无法达到负载均衡的效果，一直是一个RS提供服务。


**设置**
通过设置 `/proc/sys/net/ipv4/conf/all/arp_ignore = 1`，即 只响应目的IP地址为接收网卡上的本地地址的 ARP 请求。

#### arp_announce

**问题场景**
RS主动访问其他设备：
如果DIP为同网段，则需要在本机上存在同网段的DIP的ARP表项；
如果DIP不同网段，则需要在本机上存在下一跳IP的ARP表项；

如果上诉的ARP表项不存在，就需要发送ARP请求，那么ARP请求的SIP如何选择呢？

如果随机选择，且选择的SIP是接口的VIP，那么就会在 交换机/路由器上学习到 VIP的ARP表项。
那么，就有可能影响到 交换机/路由器上的流量转发，将本来应该导流到LB的DIP为VIP的流量导流到了 RS上。

**设置**

因此，通过设置 `/proc/sys/net/ipv4/conf/all/arp_announce = 2` ，那么就会基于报文的出接口选择 IP 作为 ARP请求的 SIP，而不是随机选择，防止误选择到VIP作为 SIP。

#### ip_forward
**问题场景**
TUN模式，后端RS收到隧道封装的健康检查/业务流量之后，先进行隧道解封装。
解封装之后，DIP为VIP，如果没有在隧道口上配置VIP（即：不存在VIP的本机路由），在 接收口的 forwarding 为  1的情况下，就会将  目的IP为VIP的流量转发出去，大概率继续落入到LB上。这个是不合理的。

**设置**
防止在 后端RS上没有配置VIP时，再次将DIP为VIP的流量转发出去。

注：也可以考虑使用 **iptables在forwarding 链加黑名单**
ipset 中添加vip；
iptables中的 forwarding 中添加 dstip为 ipset中的ip，则黑名单进行丢弃。


#### rp_filter

**问题场景**
TUN模式，后端RS在物理口（即隧道的local-ip所在的接口）上收到隧道封装的报文；
一般情况下，RS的响应报文是underlay的，直接从 RS的物理口（即隧道的 local-ip所在的接口）返回给 Client-ip。

对于收包：收到隧道封装的健康检查/业务流量之后，先进行隧道解封装。
解开封装后，查找路由，如果发现   dip（VIP）的出口（即虚拟隧道口） 和 sip的出口（ local-ip所在的 物理口）不一样，就会基于 rp_filter 的取值，决定是否丢弃。


**设置**
注意 `Ip Tunnel`模式下需要将RS上的`rp_filter`参数配为0 或者 2，否则无法正常工作，因为RS是在物理网卡收到请求，但是VIP是绑在虚拟网卡tunl0上的。

```bash
rp_filter = 0 : 不开启源地址校验;
rp_filter = 2 : 开启松散的反向路径校验。对每个进来的数据包，校验其源地址是否可达，即反向路径是否能通（通过任意网口），如果反向路径不通，则直接丢弃该数据包;

rp_filter = 1：开启严格的反向路径校验。对每个进来的数据包，校验其反向路径是否是最佳路径（即反向出口和入向口是否是同一个口）。如果反向路径不是最佳路径，则直接丢弃该数据包
```


# 缺点
## 大包问题
### 问题
每个数据包都要封装一个新的20字节的IP头，如果LVS上收到的数据包就已经达到了Ethernet帧的最大值1514（MTU1500+帧头14），这时候封装层的IP头就无法加进去。

### IPVS处理
#### icmp need frag 差错报文
如果数据报文IP头中设置了DF标志位（Don't Fragment），这时候LVS就无法正常转发该报文。而是会返回一个Type=3，Code=4的ICMP报文给客户端，通知客户端目的地不可达，需要分片，并且在通知报文中指定了本端的MTU为1480。
如果客户端支持PMTUD协议，那么客户端会根据ICMP中通知的MTU值重新计算合适的MSS，对要发送的数据进行分片后再重传给LVS节点。

> 注：这依赖于客户端支持PMTUD，并且还依赖于ICMP通知能正常返回到客户端。因为在实际情况下，ICMP消息在返回客户端的过程中需要经过多跳公网路由，在中间很可能会被拦截过滤掉，这时客户端无法收到LVS返回的ICMP通知，就无法正常的分片重传了，导致LVS转发失败。

```c
client 如何不接收ICMP   need frag报文：
iptables -I FORWARD -p icmp --icmp-type fragmentation-needed -j DROP
```

#### 分片处理
分片处理有2种方式，一种是先封装IPIP头，再进行分片。
一种是先内层分片，然后再各个分片进行封装IPIP头。
目前看DPVS中的处理，是选择的第一种方式。
#### 建议
既发送ICMP  need frag报文，又进行分片处理。

### RS上处理
可以通过减少RS侧的MSS值，比如调到1400。

#### iptables更改syn-ack的mss
这样客户端在和RS三次握手协商MSS的时候，就会使用修改后的MSS值。这样客户端网卡在对数据包进行分片时就会减小单个请求中的data大小，确保LVS上收到的请求大小不会超过1480，从而不会触发到上述的问题。

iptables配置方法如下，实际情况中可以根据需求指定规则所要匹配的ip地址，这样可以减小配置修改的影响范围。
```bash
iptables -A OUTPUT -s xxx -p tcp --tcp-flags ALL SYN,ACK -j TCPMSS --set-mss 1400

iptables -A OUTPUT -s VIRTUAL-IP -p tcp -m tcp --tcp-flags SYN,RST,ACK SYN,ACK -j TCPMSS --set-mss 1440
```
#### 更改路由的mss或者mtu
![](attachments/Pasted%20image%2020231023160724.png)
```c


ip route flush table 42
ip route add table 42 to VIP_NETWORK/24 dev eth0 advmss 1440
ip route add table 42 to default via VIP_NETWORK_GATEWAY advmss 1440
ip rule add from VIP table 42 priority 42
ip route flush cache

```
注：ethx为后端RS回包的真实出口
默认的MSS=1500-20（ip头)-20(tcp头) = 1460  
但是ipip 隧道转发IP报文给realServer的时候会添加20个字节的IP头， 防止PMTUD或者TCP分组的发生，将realServer的mss=1440。

## RSS hash不均问题

## 健康检查流量回绕为业务流量造成异常RS的健康检查抖动问题

### 背景
业务流量模型：
![](attachments/image%20(14).png)

健康检查流量模型：
![](attachments/image%20(15).png)

如上所示，需要在后端RS上配置隧道口以及隧道口的VIP。
健康检查的流量和业务的流量模型类似；对于后端RS来说，收到的是IPIP隧道报文，发出的是underlay的流量。

### 问题
多个后端RS，如果在其中多个后端RS上设置了隧道口，但是没有在隧道口设置VIP；另外一个后端RS上正确设置了隧道口以及VIP。

在漏配的RS上收到健康检查SYN报文，然后进行解封装（解封后的DIP为VIP）；解封装后进行查找路由，查找不到本机路由，如果开启了接口的 ip_forward, 那么就会继续将解封装的健康检查SYN报文转发出去，由于LB发起了VIP的路由，那么解封装的健康检查SYN报文被引流到了LB，然后进行隧道封装转发给了正常的健康的RS。

**影响**
漏配的RS的健康检查的报文被当做 业务流量转发给了正常的检查的RS，最终会造成漏配的RS在LB上的健康检查频繁的up/down抖动 ，对于业务流量有损。


### 解决
**iptables在forwarding 链加过滤**
ipset 中添加vip；
iptables中的 forwarding 中添加 dstip为 ipset中的ip，则进行丢弃。
这样，只对指定的流量有影响。

**ip_forward**
将后端RS的 ip_forward 开关关闭。
但是更改 ip_forward 可能造成所有接口的 ip_forward 都关闭了。如果 通过接口的 forwarding 来关闭，这么这个接口的forwarding 就会关闭。是否会对其他的业务造成影响是未知的。

**小结**
使用 iptablles 的 forwarding 链进行控制更好，这样只对指定的 流量有影响。


# 其他
## 抓包技巧
不使用 host xxx 进行过滤。因为 host xxx ，如果隧道的内层包为 xxx，是无法摘取到这个包的。通过 `tcpdump -nni eth04 | grep xxxx` 则是可以过滤到这个包的。

```bash
tcpdump -nni eth04 | grep 192.21.13.100
tcpdump -nni any | grep 192.21.13.100
```


# 参考
```c

```