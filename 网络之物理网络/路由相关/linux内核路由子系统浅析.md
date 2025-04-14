```table-of-contents
```
# 前置知识
## 路由缓存
每当匹配到1条路由规则，内核都会放到路由缓存中，当下次查询路由信息的时候先查看路由缓存是否有匹配到的规则，再查询路由表。

路由缓存用于减少路由表查找的时间。路由缓存的核心是与协议无关的目的缓存（Protocol Independent Destination Cache DST）。尽管采用策略路由可有效地创建多张路由表，但所有这些路由表都共享一个路由缓存。


# 路由子系统初始化
初始化函数是net/ipv4/route.c文件中ip_rt_init函数；
初始化主要做了下面几件事：
1、根据可用的内存确定路由缓存的容量
2、创建用于分配路由缓存元素的内存池
3、初始化路由缓存
4、初始化gc_thresh，即垃圾回收的阈值
5、调用ip_fib_init函数，初始化路由表，注册外部事件函数
6、调用devinet_init函数，注册ip命令可以调用的处理函数
7、初始化路由的proc文件系统

# 路由查找

![](attachments/Pasted%20image%2020240429121325.png)

![](attachments/Pasted%20image%2020240429121413.png)

## 入口流量(收包)路由处理



**入口流量路由查找函数**
ip_route_input: 
先再缓存中查找，如果没命中，判断目的地址是否是多播地址，多播地址就调用ip_route_mc，否则调用ip_route_input_slow函数，该函数会查找路由表，最终会调用fib_lookup函数查找。

**IP入口流量处理**
IP入口流量处理函数是ip_rcv_finish, 报文进入协议栈由路由表判断丢弃还是转发，决策是由ip_route_input完成，如果缓存没命中，最终会调用ip_route_input_slow查找路由表，，ip_route_input_slow最终会调用dst->input和dst->output两个虚函数。

**dst->input指向的函数**
列表如下：

![](attachments/Pasted%20image%2020240429113651.png)

**入口流量路由处理的几种情况**：

1、如果封包需要被转发，dst->input 初始化为ip_forward, dst->output初始化为ip_output

2、如果封包到本地，dst->input 初始化为ip_local_deliver, dst->output初始化为ip_rt_error

3、如果目的地址不可达，dst->input 初始化为ip_error, 并且会产生1个ICMP不可达消息

## 出口流量(发包)路由处理

**出口流量路由查找函数**
`__ip_route_output_key`: 同上缓存如果没命中，调用ip_route_output_slow函数。


**dst->output指向的函数**
列表如下：

![](attachments/Pasted%20image%2020240429113755.png)


**出口流量路由处理的几种情况**：

1、如果目的地址是远程主机，dst->output 初始化为ip_output, dst->input初始化为ip_error,这是为了捕获bug

2、如果目的地址是本地网卡，dst->input初始化为ip_locak_deliver,dst->output 初始化为ip_output,因为报文发出后，由于是本地地址，又触发ip_rcv_finish调用，继而调用input函数指向的ip_locak_deliver函数，本地接收到这个包

3、如果目的地址是多播地址，dst->output 初始化为ip_mc_output, 如果内核开启支持多播路由，dst->input初始化为ip_mr_input


# proc文件系统route相关参数

![](attachments/Pasted%20image%2020240429114341.png)

ip_forward:开启或关闭全局的IP转发
icmp_echo_ignore_boadcast:开启或关闭广播过滤

# 网络协议栈
## ip包送到 ip层 
```c
netif_receive_skb_list_internal->__netif_receive_skb_list->__netif_receive_skb_list_core
```
函数会根据包的协议，假如是 udp 包，会将包依次送到 ip_rcv(), udp_rcv() 协议处理函数中中处理。

```cpp
static int __netif_receive_skb_core(struct sk_buff **pskb, bool pfmemalloc,
				    struct packet_type **ppt_prev)
{
 
	//将数据送入抓包点，tcpdump
	list_for_each_entry_rcu(ptype, &ptype_all, list) {
		if (pt_prev)
			ret = deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}
 
    //取出协议
	type = skb->protocol;
 
	/* deliver only exact match when indicated */
	if (likely(!deliver_exact)) {
		deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
				       &ptype_base[ntohs(type) &
						   PTYPE_HASH_MASK]);
	}
...
	return ret;
}
 
//遍历list
static inline void deliver_ptype_list_skb(struct sk_buff *skb,
					  struct packet_type **pt,
					  struct net_device *orig_dev,
					  __be16 type,
					  struct list_head *ptype_list)
{
	struct packet_type *ptype, *pt_prev = *pt;
 
	list_for_each_entry_rcu(ptype, ptype_list, list) {
		if (ptype->type != type)
			continue;
		if (pt_prev)
			deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}
	*pt = pt_prev;
}
 
//fun
static inline int deliver_skb(struct sk_buff *skb,
			      struct packet_type *pt_prev,
			      struct net_device *orig_dev)
{
 
	return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
}
```

函数处理的任务

1. type = skb->protocol取出协议信息，
2. 遍历注册在这个协议上的回调函数列表， ptype_base 是 hash table初始化时注册的
3. pt_prev->func 协议层注册的处理函数ip_rcv

## IP协议层处理

```c
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt,
	   struct net_device *orig_dev)
{
	struct net *net = dev_net(dev);
	skb = ip_rcv_core(skb, net);
 
	return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING,
		       net, NULL, skb, dev, NULL,
		       ip_rcv_finish);
}
 
static int ip_rcv_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	struct net_device *dev = skb->dev;
	int ret;
 
	skb = l3mdev_ip_rcv(skb);
	ret = ip_rcv_finish_core(net, sk, skb, dev, NULL);
 
	if (ret != NET_RX_DROP)
		ret = dst_input(skb);
	return ret;
}
```

NF_HOOK 是⼀个钩⼦函数，钩子执行完后执行 `ip_rcv_finish`：
```c
函数调用关系
ip_rcv_finish_core
        ip_route_input_noref
                ip_route_input_rcu
                        ip_route_input_mc
                                rt_dst_alloc
--------------

ip_rcv_finish_core
        ip_route_input_noref
                ip_route_input_rcu
                        ip_route_input_slow
                                rt_dst_alloc

```


```c
static int ip_route_input_slow(struct sk_buff *skb, __be32 daddr, __be32 saddr,
             u8 tos, struct net_device *dev,
             struct fib_result *res)
{
  struct in_device *in_dev = __in_dev_get_rcu(dev);

  ....

  /*
   *  Now we are ready to route packet.
   */
  fl4.flowi4_oif = 0;
  fl4.flowi4_iif = dev->ifindex;
  fl4.flowi4_mark = skb->mark;
  fl4.flowi4_tos = tos;
  fl4.flowi4_scope = RT_SCOPE_UNIVERSE;
  fl4.flowi4_flags = 0;
  fl4.daddr = daddr;
  fl4.saddr = saddr;
  fl4.flowi4_uid = sock_net_uid(net, NULL);

  if (fib4_rules_early_flow_dissect(net, skb, &fl4, &_flkeys)) {
    flkeys = &_flkeys;
  } else {
    fl4.flowi4_proto = 0;
    fl4.fl4_sport = 0;
    fl4.fl4_dport = 0;
  }

  err = fib_lookup(net, &fl4, res, 0); // 路由查询
  if (err != 0) {
    if (!IN_DEV_FORWARD(in_dev))
      err = -EHOSTUNREACH;
    goto no_route;
  }

  if (res->type == RTN_BROADCAST)
    goto brd_input;

  if (res->type == RTN_LOCAL) { // dip为本机ip的流量处理
    err = fib_validate_source(skb, saddr, daddr, tos,
            0, dev, in_dev, &itag);
    if (err < 0)
      goto martian_source;
    goto local_input;
  }

  if (!IN_DEV_FORWARD(in_dev)) { // 非本机流量，查看入接口是否开启了 forwarding 转发；
    err = -EHOSTUNREACH;
    goto no_route;
  }
  if (res->type != RTN_UNICAST)
    goto martian_destination;

  err = ip_mkroute_input(skb, res, in_dev, daddr, saddr, tos, flkeys); // 非本机流量，可以进行转发，则创建路由缓存
out:  return err;

  ....
}

```


```c
struct rtable *rt_dst_alloc(struct net_device *dev,
			    unsigned int flags, u16 type,
			    bool nopolicy, bool noxfrm, bool will_cache)
{
		rt->dst.output = ip_output;
		if (flags & RTCF_LOCAL)
			rt->dst.input = ip_local_deliver;
        ...
}
```

skb_dst(skb)->input 调⽤的 input ⽅法就是路由⼦系统赋的 ip_local_deliver
```c
int ip_local_deliver(struct sk_buff *skb)
{
 
	struct net *net = dev_net(skb->dev);
    ...
	return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN,
		       net, NULL, skb, skb->dev, NULL,
		       ip_local_deliver_finish);
}
 
static int ip_local_deliver_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	__skb_pull(skb, skb_network_header_len(skb));
 
	ip_protocol_deliver_rcu(net, skb, ip_hdr(skb)->protocol);
 
	return 0;
}
 
void ip_protocol_deliver_rcu(struct net *net, struct sk_buff *skb, int protocol)
{
	const struct net_protocol *ipprot;
	int raw, ret;
 
resubmit:
	raw = raw_local_deliver(skb, protocol);
	ipprot = rcu_dereference(inet_protos[protocol]);
	if (ipprot) {
		ret = INDIRECT_CALL_2(ipprot->handler, tcp_v4_rcv, udp_rcv,skb);
        ...
		__IP_INC_STATS(net, IPSTATS_MIB_INDELIVERS);
	}
...
}
```
inet_protos 中保存着 tcp_v4_rcv() 和 udp_rcv() 的函数地址。


# 参考
```bash
# linux内核路由子系统浅析
https://zhuanlan.zhihu.com/p/602101642


# 路由：查找 【这个作者的文章很全面；++++++++++】
https://yxj-books.readthedocs.io/zh-cn/latest/linux/ULNI/ch35.html

```