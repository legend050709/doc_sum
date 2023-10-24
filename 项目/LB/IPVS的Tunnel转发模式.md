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
1. 加载ipip模块。

```undefined
modprobe ipip
```

2. 启动tunl0虚拟网卡

```undefined
ifconfig tunl0 up
```

3. 将vip绑定在tunl0网卡上。（注意这里也是与DR模式不同的地方，DR模式是将vip绑在LO网卡）

```csharp
ip addr add vip dev tunl0
```

4. 设置内核参数

```ruby
echo "0" > /proc/sys/net/ipv4/ip_forward
echo "2" > /proc/sys/net/ipv4/conf/all/arp_announce
echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore
echo "0" > /proc/sys/net/ipv4/conf/all/rp_filter
echo "0" > /proc/sys/net/ipv4/conf/tunl0/rp_filter
```

 注意Ip Tunnel模式下需要将RS上的rp_filter参数配为0，否则无法正常工作，因为RS是在物理网卡收到请求，但是VIP是绑在虚拟网卡tunl0上的。


# 缺点
## 大包问题
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
# 参考
```c

```