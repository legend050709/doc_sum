```table-of-contents
```
# 发包流程

![](attachments/Pasted%20image%2020240513155409.png)

```bash
1. Application sends message (`sendmsg` or other)
2. TCP send message allocates skb_buff
3. It enqueues skb to the socket write buffer of `tcp_wmem` size
4. Builds the TCP header (src and dst port, checksum)
5. Calls L3 handler (in this case `ipv4` on `tcp_write_xmit` and `tcp_transmit_skb`)
6. L3 (`ip_queue_xmit`) does its work: build ip header and call netfilter (`LOCAL_OUT`)
7. Calls output route action
8. Calls netfilter (`POST_ROUTING`)
9. Fragment the packet (`ip_output`)
10. Calls L2 send function (`dev_queue_xmit`)
11. Feeds the output (QDisc) queue of `txqueuelen` length with its algorithm `default_qdisc`
12. The driver code enqueue the packets at the `ring buffer tx`
13. The driver will do a `soft IRQ (NET_TX_SOFTIRQ)` after `tx-usecs` timeout or `tx-frames`
14. Re-enable hard IRQ to NIC
15. Driver will map all the packets (to be sent) to some DMA'ed region
16. NIC fetches the packets (via DMA) from RAM to transmit
17. After the transmission NIC will raise a `hard IRQ` to signal its completion
18. The driver will handle this IRQ (turn it off)
19. And schedule (`soft IRQ`) the NAPI poll system
20. NAPI will handle the receive packets signaling and free the RAM
```

1.应用程序的数据包，在TCP层增加TCP报文头，形成可传输的数据包。 
2.在IP层增加IP报头，形成IP报文。 
3.经过数据网卡驱动程序将IP包再添加14字节的MAC头，构成frame（暂⽆CRC），frame（暂⽆CRC）中含有发送端和接收端的MAC地址。 
4.驱动程序将frame（暂⽆CRC）拷贝到网卡的缓冲区，由网卡处理。 
5.⽹卡为frame（暂⽆CRC）添加头部同步信息和CRC校验，将其封装为可以发送的packet，然后再发送到网线上，这样说就完成了一个IP报文的发送了，所有连接到这个网线上的网卡都可以看到该packet。




# socket层
# 协议层
# 网络层
## `ip_queue_xmit` 
kernel提供很多个函数来让上层协议发送数据通过IP层. `ip_queue_xmit` 是其中用的相对多的函数,流程如下:
![](attachments/Pasted%20image%2020231019145754.png)
- 查找路由
kernel利用这样的事实,来自同一个socket的所有包有相同的目的地址,所以路由不需要每次都决定, 之后会分析的 `(struct dst_entry *)skb->_skb_dst` 结构指向相应的路由信息.

一旦 `ip_send_check` 为这个包生成checksum后,kernel调用 `ip_local_out`, 并调用netfilter hook `NF_INET_LOCAL_OUT`. 最后 `dst_output` 被调用.这个函数在 `skb->dst->output` 中.一般 是`ip_output`, 该函数把包发送到与目的相符的网络适配器.

## `ip_output`
`ip_output` 的流程如下, 根据包是否需要分片,函数被分成两部分处理:
![](attachments/Pasted%20image%2020231019145939.png)
首先netfilter hook `NF_INET_POST_ROUTING` 被调用,紧接着 `ip_finish_output`.
通过查看传送介质的MTU决定是否需要分片.如果不需要直接调用 `ip_finish_output2`.这个函数查看socket buffer是否有足够空间给硬件头.必要的话,调用 `skb_realloc_headroom` 添加额外的空间.为了完成传送到network access layer, 由路由层设置的 `dst->neighbour->output` 函数被调用, 通常是函数 `dev_queue_xmit`.

### Packet Fragmenting
IP包被分成小包通过函数 `ip_fragment`, 如下图:
![](attachments/Pasted%20image%2020231019150149.png)
IP 分片比较直观的.在循环的每次,与相应 MTU相容大小的分片从包中提取出来.一个新的IP头稍微改变的socket buffer被创建来放提取的数据分片.并做如下修改:

- 共同的fragment ID被分配给所有分片来支持在目的系统组装.
- fragment的序列里fragment offset为基础计算设置.
- more fragments位设置.只有最后分片设置为0.

每个分片在通过 `ip_send_check` 生成checksum后调用 `ip_output` 来发送.

### Routing
Routing是IP实现中很重要的一部分,不仅仅转发外部包需要,传递本地生成的数据同样需要.
每个收到的包属于如下3中类别:

1. 发给本地host.
2. 发给直接与现在host相连的设备.
3. 发给要通过中间系统达到的远端设备.

Cache和hash表被使用来加速运行,因为许多路由任务是消耗时间的.这里不具体分析在kernel找寻正确路由的机制.仅仅查看kernel被用来做路由结果的数据结构.
#### `ip_route_input`
路由的起始点是 `ip_route_input` 函数,它先在路由cache中找寻路由.
如没有找到, `ip_route_input_slow` 被调用，这个函数依赖 `fib_lookup` ,它的返回值是 `fib_result` 结构（fib: _forwarding information base_）。

### 路由结果与socket buffer相关联
路由结果与socket buffer相关联是通过它的dst元素指向 `dest_entry` 结构的实例.这个实例是当lookup时被填充.这个数据结构的部分定义:
```c
// include/net/dst.h
struct dst_entry
{
  struct net_device *dev;
  int (*input)(struct sk_buff*);
  int (*output)(struct sk_buff*);
  struct neighbour *neighbour;
};

// include/net/neighbour.h
struct neighbour
{
  struct net_device *dev;
  unsigned char ha[ALIGN(MAX_ADDR_LEN, sizeof(unsigned long))];
  int (*output)(struct sk_buff *skb);  //通常是：dev_queue_xmit
};
```
- input 和 output被调用来处理入口和出口的包.
- dev 指定来处理包的网络设备.

input和output被赋予不同的函数依赖于包的类型.

- 本地传递时,input设置为 `ip_local_deliver` ,而output是 `ip_rt_bug` .(后者函数只是打印error信息给kernel logs,因为对本地包调用output是不应该发生的错误情况).
- 转发时,input设置为 `ip_forward` ,而output是 `ip_output` .



# 网络设备子系统层
# 驱动层

# 监控统计
# 性能调优

# 其他
## 发包时如何选择哪个发包队列的
## 发包时tcpdump/tc/netfilter/gso等的顺序

# 参考
```c
# [译]Linux 网络栈监控和调优：发送数据（2017）
https://colobu.com/2019/12/09/monitoring-tuning-linux-networking-stack-sending-data/
# [译] Linux 网络栈监控和调优：发送数据（2017）
https://arthurchiao.art/blog/tuning-stack-tx-zh/

# Linux网络栈中的队列
https://cxd2014.github.io/2016/08/16/linux-network-stack/

# Linux 网络发包流程：详细
https://www.cnblogs.com/edisonfish/p/17637507.html

# 代码层面的发包流程
https://juejin.cn/post/6981790483215286309

# 小林coding：# 2.3 Linux 系统是如何收发网络包的？
https://xiaolincoding.com/network/1_base/how_os_deal_network_package.html#linux-%E5%8F%91%E9%80%81%E7%BD%91%E7%BB%9C%E5%8C%85%E7%9A%84%E6%B5%81%E7%A8%8B

# # Linux网络数据包接受过程（说明的很清楚）
https://simonzgx.github.io/2020/08/17/Linux%E7%BD%91%E7%BB%9C%E6%95%B0%E6%8D%AE%E5%8C%85%E6%8E%A5%E5%8F%97%E8%BF%87%E7%A8%8B/

发包流程：
https://www.ithome.com/0/644/289.htm

发包优化：
https://mp.weixin.qq.com/s/JR-qqjNG9ClHCYoRiFg-CQ

# Network layer
https://wiki.dreamrunner.org/public_html/Linux/Networks/network-layer.html
```