```table-of-contents
```

# IP隧道接收处理框架
## 隧道注册
内核使用inet_add_protocol注册了三种类型的隧道接收处理函数，三种类型分别为IPv4-in-IPv4隧道类型IPPROTO_IPIP(4)、IPv6-in-IPv4隧道IPPROTO_IPV6(41)和MIPS-in-IP隧道IPPROTO_MPLS(137)。

接收处理函数分别为：tunnel4_rcv、tunnel64_rcv和 tunnelmpls4_rcv。具体参见tunnel4_protocol、tunnel64_protocol 和 tunnelmpls4_protocol 网络协议结构体的实例。
```c
static int __init tunnel4_init(void)
{
    if (inet_add_protocol(&tunnel4_protocol, IPPROTO_IPIP))
        goto err;
    if (inet_add_protocol(&tunnel64_protocol, IPPROTO_IPV6))
        goto err;
    if (inet_add_protocol(&tunnelmpls4_protocol, IPPROTO_MPLS))
        goto err;
}
```

IPPROTO_IPIP类型的网络协议结构tunnel4_protocol如下，其handler回调函数指定为tunnel4_rcv：
```c
static const struct net_protocol tunnel4_protocol = {
    .handler    =   tunnel4_rcv,
    .err_handler    =   tunnel4_err,
    .no_policy  =   1,  /* no XFRM_POLICY_IN */
};
```

函数xfrm4_tunnel_register负责向系统中注册xfrm_tunnel结构。目前内核支持的xfrm_tunnel结构分为三个大类：

（1）对应于AF_INET协议族（即IPPROTO_IPIP）的 ipip_handler（net/ipv6/sit.c），优先级为1；ipip_handler（net/ipv4/ipip.c），优先级2；以及xfrm_tunnel_handler（net/ipv4/xfrm4_tunnel.c），优先级为3。

（2）对应于AF_INET6协议族（即IPPROTO_IPV6）的sit_handler（net/ipv6/sit.c），优先级为1；和xfrm64_tunnel_handler（net/ipv4/xfrm4_tunnel.c），优先级为2。

（3）最后是，AF_MPLS协议族（即IPPROTO_MPLS协议）的mplsip_handler（net/ipv4/ipip.c），优先级为1；和mplsip_handler（net/ipv6/sit.c），优先级为2。



## 隧道数据包接收

IPPROTO_IPIP类型数据包由注册的函数tunnel4_rcv处理。其为一个分发函数，遍历所有注册在tunnel4_handlers链表上的xfrm_tunnel类型结构，调用其成员中的handler函数指针处理接收到的数据包。优先级高的xfrm_tunnel结构先处理数据包，成功处理后（结果为0），终止遍历。

### 隧道接收流程
```bash
 ip_local_deliver_finish
    |     		
 inet_protos[proto]
    |        
 IPPROTO_IPIP (4)
    |         |
    |    tunnel4_rcv
    |         |
    |         +---- ipip_rcv (net/ipv4/ipip.c)
    |         |         |----- ipip_tunnel_rcv(skb, IPPROTO_IPIP)
    |         |
    |         +---- ipip_rcv (net/ipv6/sit.c)
    |         |         |----- sit_tunnel_rcv(skb, IPPROTO_IPIP)
    |         |		   
    |         +---- xfrm_tunnel_rcv (net/ipv4/xfrm4_tunnel.c)       
    |                   |----- xfrm4_rcv_spi(skb, IPPROTO_IPIP, ip_hdr(skb)->saddr)
    |        
 IPPROTO_IPV6 (41)
    |         |
    |    tunnel64_rcv
    |         |
    |         +---- ipip6_rcv (net/ipv6/sit.c)
    |         |
    |         +---- xfrm_tunnel_rcv (net/ipv4/xfrm4_tunnel.c)
    |                    |----- xfrm4_rcv_spi(skb, IPPROTO_IPIP, ip_hdr(skb)->saddr)
    |
 IPPROTO_MPLS (137)
    |         |
    |    tunnelmpls4_rcv
    |         |
    |         +---- mplsip_rcv (net/ipv4/ipip.c)
    |         |          |----- ipip_tunnel_rcv(skb, IPPROTO_MPLS)
    |         |
    |         +---- mplsip_rcv (net/ipv6/sit.c)
   +++                   |----- sit_tunnel_rcv(skb, IPPROTO_MPLS)
```


**物理口收到隧道包的处理**

![](attachments/Pasted%20image%2020240429105618.png)

如上所示，解隧道封装之后，==接收端口变为隧道口，然后重新通过 `gro_cells_receive` 重走内核协议栈==。


# 问题
IPIP隧道最大的问题就是外层只有三层，没有四层。存在转发不均，接收不均的问题。
## 转发不均

路由器基于dip进行转发。如果是多路径的话，比如ECMP，可能基于更多的信息来选择。但是IPIP只有三层，可能打不均匀。
  
## 接收不均
### 问题
网卡的RSS对于TCP/UDP而言，可以基于四元组hash选择接收队列。但是IPIP隧道而言，只有2元祖，很可能接受队列收包不均衡。进而硬中断/软中断也不均衡。

###  解决
（1）**RPS以及RFS**
```
调整RPS:
    echo fff > /sys/class/net/隧道口/queues/rx-0/rps_cpus
    echo 4096 > /sys/class/net/隧道口/queues/rx-0/rps_flow_cnt


调整RFS:
    sysctl -w net.core.rps_sock_flow_entries=131072
```

(2) **taskset**

可能经过RPS/RFS的设置，依然存在软中断高的问题。可以考虑程序的绑核。将软中断高的CPU排除掉。

(3) **隧道外层使用多个SIP**

如果隧道外层的IP存在多个，那么就可以将流量hash进行打散。



# 参考
```bash
（1）自我总结之：IPVS的Tunnel转发模式

（2）# 内核IP隧道接收处理框架
https://blog.csdn.net/sinat_20184565/article/details/85067126


```
