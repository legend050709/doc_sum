```table-of-contents
```
# 概述

ICMP处理流程，如下所示：
![](attachments/Pasted%20image%2020231129162604.png)

# icmp报文接收`icmp_rcv`
在 ip层判断是icmp报文之后，会调用 `icmp_rcv` 来处理icmp类型的报文。
```c
/*
 *	Deal with incoming ICMP packets.
 */
int icmp_rcv(struct sk_buff *skb)
{
	struct icmphdr *icmph;
	struct rtable *rt = skb_rtable(skb);
	struct net *net = dev_net(rt->dst.dev);
	// 基于策略的高扩展性的网络安全架构，对于这个内核子架构不清楚此处分析不了，跳过。
	if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
		struct sec_path *sp = skb_sec_path(skb);
		int nh;
 
		if (!(sp && sp->xvec[sp->len - 1]->props.flags &
				 XFRM_STATE_ICMP))
			goto drop;
 
		if (!pskb_may_pull(skb, sizeof(*icmph) + sizeof(struct iphdr)))
			goto drop;
 
		nh = skb_network_offset(skb);
		skb_set_network_header(skb, sizeof(*icmph));
 
		if (!xfrm4_policy_check_reverse(NULL, XFRM_POLICY_IN, skb))
			goto drop;
 
		skb_set_network_header(skb, nh);
	}
 
	ICMP_INC_STATS_BH(net, ICMP_MIB_INMSGS);
	
	//验证校验和信息
	switch (skb->ip_summed) {
	case CHECKSUM_COMPLETE:
		if (!csum_fold(skb->csum))
			break;
		/* fall through */
	case CHECKSUM_NONE:
		skb->csum = 0;
		if (__skb_checksum_complete(skb))
			goto csum_error;
	}
 
	if (!pskb_pull(skb, sizeof(*icmph)))
		goto error;
	
	//获取icmp头部
	icmph = icmp_hdr(skb);
 
	ICMPMSGIN_INC_STATS_BH(net, icmph->type);
	/*
	 *	18 is the highest 'known' ICMP type. Anything else is a mystery
	 *
	 *	RFC 1122: 3.2.2  Unknown ICMP messages types MUST be silently
	 *		  discarded.
	 */
	 //type类型错误直接丢掉
	if (icmph->type > NR_ICMP_TYPES)
		goto error;
 
 
	/*
	 *	Parse the ICMP message
	 */
	
	//判断是否丢弃掉多播类型的icmp数据包	
	//只处理echo、timestamp、address_mask_request、address_mask_reply类型的多播icmp数据包
	if (rt->rt_flags & (RTCF_BROADCAST | RTCF_MULTICAST)) {
		/*
		 *	RFC 1122: 3.2.2.6 An ICMP_ECHO to broadcast MAY be
		 *	  silently ignored (we let user decide with a sysctl).
		 *	RFC 1122: 3.2.2.8 An ICMP_TIMESTAMP MAY be silently
		 *	  discarded if to broadcast/multicast.
		 */
		if ((icmph->type == ICMP_ECHO ||
		     icmph->type == ICMP_TIMESTAMP) &&
		    net->ipv4.sysctl_icmp_echo_ignore_broadcasts) {
			goto error;
		}
		if (icmph->type != ICMP_ECHO &&
		    icmph->type != ICMP_TIMESTAMP &&
		    icmph->type != ICMP_ADDRESS &&
		    icmph->type != ICMP_ADDRESSREPLY) {
			goto error;
		}
	}
	//根据icmp数据包类型，调用相应的处理函数
	icmp_pointers[icmph->type].handler(skb);
 
drop:
	kfree_skb(skb);
	return 0;
csum_error:
	ICMP_INC_STATS_BH(net, ICMP_MIB_CSUMERRORS);
error:
	ICMP_INC_STATS_BH(net, ICMP_MIB_INERRORS);
	goto drop;
}
```

## type类型对应的icmp报文处理
```c
static const struct icmp_control icmp_pointers[NR_ICMP_TYPES + 1] = {
	[ICMP_ECHOREPLY] = {
		.handler = ping_rcv,
	},
	[1] = {
		.handler = icmp_discard,
		.error = 1,
	},
	[2] = {
		.handler = icmp_discard,
		.error = 1,
	},
	[ICMP_DEST_UNREACH] = {
		.handler = icmp_unreach,
		.error = 1,
	},
	[ICMP_SOURCE_QUENCH] = {
		.handler = icmp_unreach,
		.error = 1,
	},
	[ICMP_REDIRECT] = {
		.handler = icmp_redirect,
		.error = 1,
	},
	[6] = {
		.handler = icmp_discard,
		.error = 1,
	},
	[7] = {
		.handler = icmp_discard,
		.error = 1,
	},
	[ICMP_ECHO] = {
		.handler = icmp_echo,
	},
	[9] = {
		.handler = icmp_discard,
		.error = 1,
	},
	[10] = {
		.handler = icmp_discard,
		.error = 1,
	},
	[ICMP_TIME_EXCEEDED] = {
		.handler = icmp_unreach,
		.error = 1,
	},
	[ICMP_PARAMETERPROB] = {
		.handler = icmp_unreach,
		.error = 1,
	},
	[ICMP_TIMESTAMP] = {
		.handler = icmp_timestamp,
	},
	[ICMP_TIMESTAMPREPLY] = {
		.handler = icmp_discard,
	},
	[ICMP_INFO_REQUEST] = {
		.handler = icmp_discard,
	},
	[ICMP_INFO_REPLY] = {
		.handler = icmp_discard,
	},
	[ICMP_ADDRESS] = {
		.handler = icmp_discard,
	},
	[ICMP_ADDRESSREPLY] = {
		.handler = icmp_discard,
	},
};
```

## unreach 差错报文处理
收到远端发送的 不可达message。`icmp_unreach`的核心逻辑就是：根据icmp中有效载荷的值，调用传输层的错误处理函数进行处理。比如：`tcp_v4_err` 以及 `udp_err`。
```c
/*
 *	Handle ICMP_DEST_UNREACH, ICMP_TIME_EXCEED, and ICMP_SOURCE_QUENCH.
 */
 
static void icmp_unreach(struct sk_buff *skb)
{
	const struct iphdr *iph;
	struct icmphdr *icmph;
	struct net *net;
	u32 info = 0;
 
	net = dev_net(skb_dst(skb)->dev);
 
	/*
	 *	Incomplete header ?
	 * 	Only checks for the IP header, there should be an
	 *	additional check for longer headers in upper levels.
	 */
 
	if (!pskb_may_pull(skb, sizeof(struct iphdr)))
		goto out_err;
	
	//获取icmp首部
	icmph = icmp_hdr(skb);
	iph   = (const struct iphdr *)skb->data;
	
	//判断ip首部是否完整
	if (iph->ihl < 5) /* Mangled header, drop. */
		goto out_err;
	/*仅处理type类型为3或者12的数据包
       1、当类型为3时，仅处理code为frag needed的报文
           a)当系统不支持pmtu时，丢弃该数据包
           b)当系统支持pmtu时，调用ip_rt_frag_needed修改pmtu的值
       2、当type类型为12时，则通过icmph->un.gateway获取出错偏移值(相对于数据包)
    */
	if (icmph->type == ICMP_DEST_UNREACH) {
		switch (icmph->code & 15) {
		case ICMP_NET_UNREACH:
		case ICMP_HOST_UNREACH:
		case ICMP_PROT_UNREACH:
		case ICMP_PORT_UNREACH:
			break;
		case ICMP_FRAG_NEEDED:
			if (ipv4_config.no_pmtu_disc) {
				LIMIT_NETDEBUG(KERN_INFO pr_fmt("%pI4: fragmentation needed and DF set\n"),
					       &iph->daddr);
			} else {
				info = ntohs(icmph->un.frag.mtu);
				if (!info)
					goto out;
			}
			break;
		case ICMP_SR_FAILED:
			LIMIT_NETDEBUG(KERN_INFO pr_fmt("%pI4: Source Route Failed\n"),
				       &iph->daddr);
			break;
		default:
			break;
		}
		if (icmph->code > NR_ICMP_UNREACH)
			goto out;
	} else if (icmph->type == ICMP_PARAMETERPROB)
		info = ntohl(icmph->un.gateway) >> 24;
 
	/*
	 *	Throw it at our lower layers
	 *
	 *	RFC 1122: 3.2.2 MUST extract the protocol ID from the passed
	 *		  header.
	 *	RFC 1122: 3.2.2.1 MUST pass ICMP unreach messages to the
	 *		  transport layer.
	 *	RFC 1122: 3.2.2.2 MUST pass ICMP time expired messages to
	 *		  transport layer.
	 */
 
	/*
	 *	Check the other end isn't violating RFC 1122. Some routers send
	 *	bogus responses to broadcast frames. If you see this message
	 *	first check your netmask matches at both ends, if it does then
	 *	get the other vendor to fix their kit.
	 */
	//对于目的地址是广播的icmp数据包，且需要忽略时，则打印错误并忽略该数据包
	if (!net->ipv4.sysctl_icmp_ignore_bogus_error_responses &&
	    inet_addr_type(net, iph->daddr) == RTN_BROADCAST) {
		net_warn_ratelimited("%pI4 sent an invalid ICMP type %u, code %u error to a broadcast: %pI4 on %s\n",
				     &ip_hdr(skb)->saddr,
				     icmph->type, icmph->code,
				     &iph->daddr, skb->dev->name);
		goto out;
	}
 
	icmp_socket_deliver(skb, info);
 
out:
	return;
out_err:
	ICMP_INC_STATS_BH(net, ICMP_MIB_INERRORS);
	goto out;
}
```

```c
static void icmp_socket_deliver(struct sk_buff *skb, u32 info)
{
	//此时的iph，是icmp有效载荷中的ip头部信息，在icmp_rcv中已经将skb->data指向icmp报文的有效载荷部分了
	const struct iphdr *iph = (const struct iphdr *) skb->data;
	const struct net_protocol *ipprot;
	//获取传输层协议值
	int protocol = iph->protocol;
 
	/* Checkin full IP header plus 8 bytes of protocol to
	 * avoid additional coding at protocol handlers.
	 */
	/*检测icmp报文中有效载荷部分内容长度是否大于等于ip头部信息加上8字节
      在发送icmp差错报文时，会将icmp数据部分的值设置为ip头部信息+ ip有效载荷的前8个字节，
      这样就可以判断是传输层的那个应用数据发送出错*/
	if (!pskb_may_pull(skb, iph->ihl * 4 + 8))
		return;
		
	//首先调用raw_icmp_error，将差错信息发送给感兴趣的raw socket
	raw_icmp_error(skb, protocol, info);
 
	rcu_read_lock();
	//根据protocol值，查找符合条件的4层接收处理hash数组inet_protos，
    //调用其错误处理函数进行后续处理
	ipprot = rcu_dereference(inet_protos[protocol]);
	if (ipprot && ipprot->err_handler)
		ipprot->err_handler(skb, info);
	rcu_read_unlock();
}
```
# icmp数据发送`icmp_send`
对于入口数据包处理失败时，上层协议会调用`icmp_send`发送数据（比如：udp在收到一个没有监听端口的报文时，会调用该函数发送端口不可达信息。接口：`__udp4_lib_rvc`）。通过 `icmp_send` 可以发送一个 icmp 差错报文。

## 不可以发送icmp差错的情况
- 对于入口数据包是多播的数据包（mac地址 或者ip地址为多播地址）
- 入口数据包为分片，且不是首片。
注：仅仅对首个分片的入口数据包，发送icmp err message.
- 入口数据包本身就是 icmp error 类型

# 其他
## 内核处理ICMP packet too big
### 整体流程
报文类型：
```c
#define ICMP_DEST_UNREACH       3       /* Destination Unreachable      */
#define ICMP_FRAG_NEEDED        4       /* Fragmentation Needed/DF set  */
```
`icmp_rcv(struct sk_buff *skb)`，根据 ICMP 类型 type 处理skb，`icmp_pointers[icmph->type].handler(skb)`
```c
int icmp_rcv(struct sk_buff *skb)
{
    ....
    icmph = icmp_hdr(skb);
    ....

    success = icmp_pointers[icmph->type].handler(skb);
    ...
}
```
`icmp_pointers`的定义显示，`ICMP_DEST_UNRAECH`的`handler`为`icmp_unreach`，后者获取出ICMP头部的mtu后，投递给`icmp_socket_deliver(skb, mtu)`。
`icmp_socket_deliver`最终将skb和mtu值投递给`ipprot->err_handler(skb, info)`;

#### TCP流程
内层的报文为TCP报文，则 `ipprot->err_handler(skb, info)` 即为 `tcp_v4_err(skb, info)`;
```c
static struct net_protocol tcp_protocol = {
    .early_demux    =   tcp_v4_early_demux,
    .early_demux_handler =  tcp_v4_early_demux,
    .handler    =   tcp_v4_rcv,
    .err_handler    =   tcp_v4_err,
    .no_policy  =   1,
    .netns_ok   =   1,
    .icmp_strict_tag_validation = 1,
};
```
#### UDP流程
内层的报文为UDP报文，则 `ipprot->err_handler(skb, info)` 即为 `udp_err`;
```c
static struct net_protocol udp_protocol = {
    .early_demux =  udp_v4_early_demux,
    .early_demux_handler =  udp_v4_early_demux,
    .handler =  udp_rcv,
    .err_handler =  udp_err,
    .no_policy =    1,
    .netns_ok = 1,
};
```
# 参考
```c
# linux内核协议栈 icmp 报文收发流程
https://blog.csdn.net/wangquan1992/article/details/109567188
```