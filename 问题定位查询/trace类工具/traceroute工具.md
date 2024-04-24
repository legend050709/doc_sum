```table-of-contents
```

# 原理

![](attachments/Pasted%20image%2020240514141323.png)

`traceroute` 的原理很精妙。它利用了 IP 协议本身的特性：每一个 IP 包都有一个 `TTL` 字段，表示这个包还能在网络中被转发多少次，每次路由器转发一个 IP 包就将其 -1，如果一个路由器发现 IP 包 -1 之后是 0 了，就直接丢弃，并且给 IP 包的 Source IP 发送一个 ICMP 包（包含此 hop 自己的 IP），说这个包气数已尽，无法送达目的地。

`traceroute` 想要知道去往另一个 IP，中间都会经过哪些 IP 节点。它发送一个 TTL=1 的 ping 包出去，包会挂在第一个 hop 上，第一个 hop 发回去 ICMP，`traceroute` 就知道了第一个 hop 的 IP 地址；它再发送一个 TTL=2 的 ping 包，就知道了第二个 hop 的 IP 地址…… 直到收到正确的 ping 响应，就算到头了。


# traceroute的难点
## 流量经过LB的转换

### 问题描述
一条流量经过LB进行了源目IP,Port的转换后，报文转换后在设备上生成ICMP差错报文，LB如何将ICMP差错报文透传给TraceRoute发起者？
如下所示：

### 问题解决


## Overlay封装
### 问题描述
TraceRoute发起者发起的TraceRoute是对于某条underlay流的trace，经过中间设备之后，underlay流量被封装为Overlay（虽然外层Overlay也可以）。那么TraceRoute发起者发出trace后，经过overlay封装，此时是无法知道overlay数据包所经过的路径，因为overlay封装之后，即使产生了icmp差错报文也是给了其他的设备，而不是TraceRoute发起者。
另外一个问题是，vxlan封装之后，外层IP头的TTL和内层不一样的问题。

### 问题解决
思路：
可以基于外层Sport的计算方法，基于内层的五元组信息，


# 相关问题
## 路由 path 的一致性问题

如果中间某一跳到下一跳有多个节点可以选择，正确的情况下，对于 `(src ip, src port, dst ip, dst port)` 这样的四元组，应该固定走一条线路。



### traceroute 使用 icmp 探测
ICMP Ping 没有端口的概念，但是对于 `(src ip, dst ip)`，路线应该是一致的。即，我们在对某一个 IP mtr 的结果中，每一跳都应该只有一个唯一 IP。

![](attachments/Pasted%20image%2020240514141817.png)

### traceroute 使用 tcp 探测

mtr 有 tcp traceroute 的功能，我用 tcpdump 看了下，原理和 traceroute 相同，只不过 IP 包的内容不是 ICMP 了，而是 TCP。
路由器丢了 IP 包，不管里面是 TCP 还是 ICMP，都会发回来 ICMP 消息。

于是 tcp traceroute，我们就可以用 tcp 包探测出来中间的转发节点。

mtr 使用的 tcp 包是 syn 包，remote port 可以指定，但是 local port 每一个包都是随机选择的。这就导致，每一次发起探测，都是一个不同的四元组：`(src ip, **src port**, dst ip, dst port)` ，因为 `src port` 每次都在变，所以路径每次都是不一样的。这样的话相当于可以将路径中所有的路由设备 IP 都探测出来。

![](attachments/Pasted%20image%2020240514141852.png)

如图可以看出，一跳可能有多个 IP。




# 问题查询范例
## TCP乱序问题

### 问题
TCP 协议是基于 IP 协议的。IP 协议不保证顺序，只能说**尽力**保证包的顺序。**如果发生乱序，[TCP 的性能就会下降很多](https://www.kawabangga.com/posts/5181)**。

最近就遇到一个 TCP 下载速度很慢的问题，抓包分析发现有很多乱序的包。


### 分析

网络中的 TCP，相同的四元组（sip,sport,dip,dport），应该走同一条路，来尽量保持到达顺序和发送顺序一致。

利用这个原理，假设我们使用的 local port 一直保持不变，那么 TCP traceroute 的结果应该是，**每一跳都只有一个 IP**。

### TCP multipath 多路径探测

mtr 没有提供固定 local ip 来 trace 的功能，我用 scapy 写了一个。

```python3
from scapy.all import *

hostname = "10.0.0.1"

# add timeout

for i in range(1, 20):
    pkt = IP(dst=hostname, ttl=i) / TCP(sport=22334, dport=9100)
    # Send the packet and get a reply
    reply = sr1(pkt, verbose=0, timeout=5)
    if reply is None:
        print("%d hop ??" % i)
        continue
    elif TCP in reply:
        # We've reached our destination
        print("Done!", reply.src)
        break
    else:
        # We're in the middle somewhere
        print("%d hops away: " % i, reply.src)

```

**分析**：
```text
代码非常简单，就是发送 `ttl` 不同的包，然后检查收到的 ICMP 包。限制最多 20 次，如果 20 次还没到达终点就放弃。如果 `reply` 里面有 TCP 包，说明收到了正常的回复（即使是 `RST` 也算）。

因为我们每次都用同一个 local port，所以不用多线程，每次只能有一个探测存在。
```

**结果**：
```text
实际测试发现，收到的 ICMP 响应并不是很稳定，有的时候 hop 4 丢了，有的时候 hop 5 丢了。可以使用一个 bash 循环多测试几次。

```

```bash
while true; do sudo python3 tra.py  >> traceroute.log; done
```

跑一段时间之后，我们可以查看这个日志文件，过滤掉错误的包 (`??`)，然后排序之后，去重。理论上，每一跳都应该只有一个 IP 才对，如果有某一个 hop 出现了两个 IP，那说明就是出现了 multiple，会知道乱序。

```bash
$ grep -v '??' traceroute.log | grep -v Done | sort | uniq
1 hops away: 10.50.20.3
2 hops away: 10.60.40.50
3 hops away: 10.70.80.90
4 hops away: 10.80.70.60
5 hops away: 10.90.80.70
5 hops away: 10.90.80.71
6 hops away: 10.100.110.120
7 hops away: 10.110.120.130
8 hops away: 10.12.130.14
```

如上输出，在 hop 5 出现了两个 IP，说明是 hop 4 到 hop 5 的时候出现了 multipath。



# 参考
```bash
# 探测 TCP 乱序问题
https://www.kawabangga.com/posts/5876
```