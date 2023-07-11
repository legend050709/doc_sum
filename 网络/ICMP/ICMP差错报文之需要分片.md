# 背景
# 分片带来的问题
- 重组占用内存
完成分片重组，收到最后一个分片前，需要将所有的分片都缓存下来。
注：对于交换机这种通过硬件转发，而不是CPU转发的设备。硬件转发不能处理复杂的逻辑，并且内存很小。应该无法分片重组。

- 分片以及重组消耗CPU
在Server上进行分片以及重组，会消耗CPU

- 分片不包含四层头的问题
只有第一个分片存在四层头。对于以五元组进行转发的设备，那么第一个分片和后续分片可能分发给了集群中的不同设备。另外，防火墙可能会丢弃不含有四层头的分片报文。

- 分片丢失的问题
分片可能存在着丢失的问题。某个分片丢失了，是没有机制告知其他分片，该分片丢失了。
# 解决
## IPv4 DF标记
禁止中间设备进行分片，比如在IPv4头设置不允许分片标记，当报文大小大于出口的MTU时，则会发送ICMP Need-Frag差错报文，告知发送端该设备出口的MTU，使得发送端调整发送报文的大小。

A solution to these problems was included in the IPv4 protocol. A sender can set the DF (Don't Fragment) flag in the IP header, asking intermediate routers never to perform fragmentation of a packet.

![](attachments/Pasted%20image%2020230703153336.png)

注：对于TCP协议，一般而言，DF标记都是设置的。对于UDP而言，则不一定。

# 场景
比如经过设备之后，添加Vxlan隧道、IPIP隧道。报文大小大于了出口的MTU。此时需要进行分片。
![](attachments/Pasted%20image%2020230703160503.png)

## 1. Client -> Server DF+ / ICMP
![](attachments/Pasted%20image%2020230703163337.png)

## Client -> Server DF- / fragmentation
![](attachments/Pasted%20image%2020230703163357.png)

## 3. Server -> Client DF+ / ICMP
![](attachments/Pasted%20image%2020230703163715.png)

## 4. Server -> Client DF- / fragmentation
![](attachments/Pasted%20image%2020230703163950.png)


# 需要分片的差错报文
## ipv4
```c
type= ICMP_DEST_UNREACH, code = ICMP_UNREACH_NEEDFRAG
```
## ipv6
```c
type = ICMP6_PACKET_TOO_BIG; code = 0
```
# 接受差错报文后的处理
## 服务器配置
服务器接受的ICMP差错报文需要被处理。
那么查询反向路由应该可以查询到，因为发送ICMP差错报文的SIP为需要分片的设备的接口的IP。
注：如果服务器的接口的rp_filter设置为2，应该是没有问题的。

## TCP的处理
TCP收到ICMP差错报文（IPv4 or IPv6）,提取ICMP差错报文的MTU信息，基于 ICMP差错报文的内层的DIP，接下来发送给这个目的IP的存量连接（调整连接的TCP载荷大小）以及新建连接的报文大小（调整Syn中的MSS) 都会和ICMP差错中的MTU适配。

## UDP的处理
对于UDP程序而言，收到了ICMP需要分片的差错报文。基于 ICMP差错报文的内层的DIP，接下来发送给这个目的IP的流量都会基于ICMP中的MTU进行分片，然后进行发送。

# 其他

# 参考
```c
https://blog.cloudflare.com/ip-fragmentation-is-broken/
```