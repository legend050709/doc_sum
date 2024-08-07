```table-of-contents
```

# 接收侧tc能否流量分叉+RSS复杂均衡
## 需求
比如，一个机器上存在DPDK程序（比如dpvs）处理业务流量，还存在控制流量(比如，bgp的引流流量)。目前是DPDK程序接管了网卡，bgp的流量和业务流量共享了网卡的接收队列，如果业务流量过大，可能导致网卡队列丢包，可能丢失的就是bgp的流量，进而导致bgp断流。
期望的是，通过VF(不同的设备)或者直接是FDIR导流到不同的网卡队列，达到BGP流量和业务流量不共享网卡队列的效果。

目前已知的 VF应该不太支持（pf默认基于dstmac转发到vf，ethtool设置fdir难以生效），DPDK中设置FDIR规则，可能也不太容易（还没有尝试，反正通过ethtool 设置FDIR queue-group是无法成功的，很多的网卡驱动，比如ixgbe不支持设置 fdir
的queue-group）。

那么，是否存在其他的方法，达到流量分叉到多队列，以及多个网卡队列的负载均衡呢？

## TC 设置queue-groups
参考：[# TC queue based filtering](https://docs.kernel.org/networking/tc-queue-filters.html)

![](attachments/Pasted%20image%2020240731143747.png)


### 范例
#### 范例一
![](attachments/Pasted%20image%2020240731144526.png)

**了解TC命令**
通过以下命令来介绍tc命令的配置
```bash
# tc qdisc add dev eth4 root mqprio num_tc 3 map 2 2 1 0 2 2 2 2 2 2 2 2 2 2 2 2 queues 4@0 6@4  2@10 hw 1

上面的命令配置3种流量类别。HW队列根据命令末尾的配置映射到每个流量类。
    a)  tc0从队列0（q0-q3）开始有4个队列
    b)  tc1有6个从队列4（q4-q9）开始的队列
    c)  tc2有2个从队列10开始的队列（q10，q11）

关键字“ map”后的字符串是流量类别map的优先级（从0 到15）。它将每个优先级0-15映射到指定的流量类别。映射中的索引（从零开始）是优先级，给定索引处的值指定流量类别编号。
    a) 优先级3映射到tc0，并且tc0包含硬件队列0-3。
    b) 优先级2映射到tc1，tc1包含硬件队列4-9。
    c) 优先级0,1,4-15映射到tc2，其中包含硬件队列10和11。
```

**流量分叉**
```bash
#tc qdisc add dev $iface root mqprio num_tc 2 map 0 1 queues 2@0 $queues@2 hw 1 mode channel

#tc qdisc add dev $iface ingress
以上命令中配置了两个流量类别。tc0有两个队列0,1。tc1中的队列数是根据用户的输入进行配置的。优先级0映射到tc0，优先级1映射到tc1。

# tc filter add dev $iface protocol ip parent ffff: prio 1 flower \  
       dst_ip $traddr/32 ip_proto tcp dst_port $trsvcid skip_sw hw_tc 1
       
配置流量类别后，将使用目标地址（traddr）和端口号（trsvcid）配置tc过滤器来控制数据包。在target机器上，传入的数据包将在目标ip字段中具有目标的地址，而在目标端口字段中将具有trsvcid（即nvmf_tgt侦听端口）。该过滤器与tc1相关联。在启动器端，将过滤器配置为使用nvmf_tgt应用程序使用源地址和源端口进入的数据包。

tc配置仅影响Rx流量。Tx流量是根据流量类别映射配置的优先级由套接字优先级配置的。
```


```bash
验证TC正确地创建了
# tc qdisc show dev $iface

验证TC filter
# tc filter show dev $iface ingress
```

#### 范例二
![](attachments/Pasted%20image%2020240731145054.png)

```bash
tc qdisc add dev <i/f> root mqprio num_tc 3 map 0 1 2 queues 2@0 32@2 
8@34 hw 1 mode channel

tc filter add dev <i/f> protocol ip ingress prio 1 flower dst_ip 
192.168.0.2 ip_proto tcp dst_port 1234 skip_sw hw_tc 1

tc filter add dev <i/f> protocol ip ingress prio 1 flower dst_ip 
192.168.0.3 ip_proto tcp dst_port 1234 skip_sw hw_tc 2
```

# 参考
```bash
# SPDK总体技术介绍 （包含多个ADQ相关的介绍）
https://spdk.io/cn/articles/

```