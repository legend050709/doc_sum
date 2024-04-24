```table-of-contents
```
# iptable概念
## 整体概述

![](attachments/Pasted%20image%2020231010102334.png)

![](attachments/Pasted%20image%2020240506141658.png)

![](attachments/Pasted%20image%2020240506104305.png)

![](attachments/Pasted%20image%2020240506104520.png)

## 表
## 链
### PRE_ROUTING
#### 代码位置


```c
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt, struct net_device *orig_dev)
{
    ...
    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, skb, dev, NULL,
               ip_rcv_finish);
    ...
}
```

### LOCAL_IN
### FORWARD
### LOCAL_OUT
### POST_ROUTING

# iptable规则查询
# iptables规则管理
# iptables匹配条件
# iptables扩展模块
iptables扩展功能需要配合模块实现；

**隐式扩展**：
在使用-p选项指明了特定的协议时，无需再用-m选项指明扩展模块的扩展机制，不需要手动加载扩展模块。
即有的模块只需要指定协议名称，不需要指定模块名称，这样在定义协议时，间接的使用了模块，如TCP、UDP、icmp协议，因为协议名与模块名同名；

**显示扩展**：
必须使用-m选项指明要调用的扩展模块的扩展机制(手动指定引用的模块名称)，要手动加载扩展模块。

**查看iptables支持的扩展模块**：
```bash
# rpm -ql iptables | grep xtable
/usr/lib64/libxtables.so.10
/usr/lib64/libxtables.so.10.0.0
/usr/lib64/xtables
/usr/lib64/xtables/libip6t_DNAT.so
/usr/lib64/xtables/libip6t_DNPT.so
/usr/lib64/xtables/libip6t_HL.so
/usr/lib64/xtables/libip6t_LOG.so
/usr/lib64/xtables/libip6t_MASQUERADE.so
/usr/lib64/xtables/libip6t_NETMAP.so
/usr/lib64/xtables/libip6t_REDIRECT.so
/usr/lib64/xtables/libip6t_REJECT.so
/usr/lib64/xtables/libip6t_SNAT.so
/usr/lib64/xtables/libip6t_SNPT.so
/usr/lib64/xtables/libip6t_ah.so
/usr/lib64/xtables/libip6t_dst.so
/usr/lib64/xtables/libip6t_eui64.so
/usr/lib64/xtables/libip6t_frag.so
/usr/lib64/xtables/libip6t_hbh.so
/usr/lib64/xtables/libip6t_hl.so
/usr/lib64/xtables/libip6t_icmp6.so
/usr/lib64/xtables/libip6t_ipv6header.so
/usr/lib64/xtables/libip6t_mh.so
/usr/lib64/xtables/libip6t_rt.so
/usr/lib64/xtables/libipt_CLUSTERIP.so
/usr/lib64/xtables/libipt_DNAT.so
/usr/lib64/xtables/libipt_ECN.so
/usr/lib64/xtables/libipt_LOG.so
/usr/lib64/xtables/libipt_MASQUERADE.so
/usr/lib64/xtables/libipt_MIRROR.so
/usr/lib64/xtables/libipt_NETMAP.so
/usr/lib64/xtables/libipt_REDIRECT.so
/usr/lib64/xtables/libipt_REJECT.so
/usr/lib64/xtables/libipt_SAME.so
/usr/lib64/xtables/libipt_SNAT.so
/usr/lib64/xtables/libipt_TTL.so
/usr/lib64/xtables/libipt_ULOG.so
/usr/lib64/xtables/libipt_ah.so
/usr/lib64/xtables/libipt_icmp.so
/usr/lib64/xtables/libipt_realm.so
/usr/lib64/xtables/libipt_ttl.so
/usr/lib64/xtables/libipt_unclean.so
/usr/lib64/xtables/libxt_AUDIT.so
/usr/lib64/xtables/libxt_CHECKSUM.so
/usr/lib64/xtables/libxt_CLASSIFY.so
/usr/lib64/xtables/libxt_CONNMARK.so
/usr/lib64/xtables/libxt_CONNSECMARK.so
/usr/lib64/xtables/libxt_CT.so
/usr/lib64/xtables/libxt_DSCP.so
/usr/lib64/xtables/libxt_HMARK.so
/usr/lib64/xtables/libxt_IDLETIMER.so
/usr/lib64/xtables/libxt_LED.so
/usr/lib64/xtables/libxt_MARK.so
/usr/lib64/xtables/libxt_NFLOG.so
/usr/lib64/xtables/libxt_NFQUEUE.so
/usr/lib64/xtables/libxt_NOTRACK.so
/usr/lib64/xtables/libxt_RATEEST.so
/usr/lib64/xtables/libxt_SECMARK.so
/usr/lib64/xtables/libxt_SET.so
/usr/lib64/xtables/libxt_SYNPROXY.so
/usr/lib64/xtables/libxt_TCPMSS.so
/usr/lib64/xtables/libxt_TCPOPTSTRIP.so
/usr/lib64/xtables/libxt_TEE.so
/usr/lib64/xtables/libxt_TOS.so
/usr/lib64/xtables/libxt_TPROXY.so
/usr/lib64/xtables/libxt_TRACE.so
/usr/lib64/xtables/libxt_addrtype.so
/usr/lib64/xtables/libxt_bpf.so
/usr/lib64/xtables/libxt_cgroup.so
/usr/lib64/xtables/libxt_cluster.so
/usr/lib64/xtables/libxt_comment.so
/usr/lib64/xtables/libxt_connbytes.so
/usr/lib64/xtables/libxt_connlabel.so
/usr/lib64/xtables/libxt_connlimit.so
/usr/lib64/xtables/libxt_connmark.so
/usr/lib64/xtables/libxt_conntrack.so
/usr/lib64/xtables/libxt_cpu.so
/usr/lib64/xtables/libxt_dccp.so
/usr/lib64/xtables/libxt_devgroup.so
/usr/lib64/xtables/libxt_dscp.so
/usr/lib64/xtables/libxt_ecn.so
/usr/lib64/xtables/libxt_esp.so
/usr/lib64/xtables/libxt_hashlimit.so
/usr/lib64/xtables/libxt_helper.so
/usr/lib64/xtables/libxt_iprange.so
/usr/lib64/xtables/libxt_ipvs.so
/usr/lib64/xtables/libxt_length.so
/usr/lib64/xtables/libxt_limit.so
/usr/lib64/xtables/libxt_mac.so
/usr/lib64/xtables/libxt_mark.so
/usr/lib64/xtables/libxt_multiport.so
/usr/lib64/xtables/libxt_nfacct.so
/usr/lib64/xtables/libxt_osf.so
/usr/lib64/xtables/libxt_owner.so
/usr/lib64/xtables/libxt_physdev.so
/usr/lib64/xtables/libxt_pkttype.so
/usr/lib64/xtables/libxt_policy.so
/usr/lib64/xtables/libxt_quota.so
/usr/lib64/xtables/libxt_rateest.so
/usr/lib64/xtables/libxt_recent.so
/usr/lib64/xtables/libxt_rpfilter.so
/usr/lib64/xtables/libxt_sctp.so
/usr/lib64/xtables/libxt_set.so
/usr/lib64/xtables/libxt_socket.so
/usr/lib64/xtables/libxt_standard.so
/usr/lib64/xtables/libxt_state.so
/usr/lib64/xtables/libxt_statistic.so
/usr/lib64/xtables/libxt_string.so
/usr/lib64/xtables/libxt_tcp.so
/usr/lib64/xtables/libxt_tcpmss.so
/usr/lib64/xtables/libxt_time.so
/usr/lib64/xtables/libxt_tos.so
/usr/lib64/xtables/libxt_u32.so
/usr/lib64/xtables/libxt_udp.so
/usr/sbin/xtables-multi
```
如上所示，比如，tcp、time、udp、limit、iprange、state 等等扩展模块。



**iptables-extensions**
通过`man iptables-extensions`也可以查看iptables支持的扩展模块以及对应的参数，如下所示：
```bash
# iptables -m addrtype -h
Address type match options:
 [!] --src-type type[,...]      Match source address type
 [!] --dst-type type[,...]      Match destination address type
     --limit-iface-in           Match only on the packet's incoming device
     --limit-iface-out          Match only on the packet's outgoing device

Valid types:
                                UNSPEC
                                UNICAST
                                LOCAL
                                BROADCAST
                                ANYCAST
                                MULTICAST
                                BLACKHOLE
                                UNREACHABLE
                                PROHIBIT
                                THROW
                                NAT
                                XRESOLVE
```
![](attachments/Pasted%20image%2020240405170428.png)
![](attachments/Pasted%20image%2020240405170743.png)



## 协议相关的扩展模块
### tcp
#### 介绍
**tcp模块**
```bash
iptables -m tcp -h
```
![](attachments/Pasted%20image%2020240403145527.png)


**tcpmss模块**
```bash
# iptables -m tcpmss -h
tcpmss match options:
[!] --mss value[:value]	Match TCP MSS range.
				(only valid for TCP SYN or SYN/ACK packets)
```
#### 使用
```bash
iptables -A INPUT -s 192.168.39.6 -p tcp --dport 80 -j ACCEPT
#filter表input链添加一条规则，允许192.168.39.6通过tcp协议访问本机的80端口；-p指定tcp协议时间接使用了tcp模块

iptables -A INPUT -s 192.168.39.6 -p tcp --dport 80:90 -j ACCEPT
#允许192.168.39.6通过tcp协议访问本机的80-90之间的所有端口；连续的端口号只会占用一条规则

iptables -A INPUT -s 192.168.39.100 -p tcp --tcp-flags SYN,ACK,FIN,RST,URG,PSH SYN -j REJECT
#filter表input链添加规则，指定192.168.39.100三次捂手的第一次握手直接拒绝；指定6个标记位，三次握手的第一次握手只有SYN标记位为1，其余都为0，所以最后的SYN表示为SYN为1；标记位只属于TCP协议，所以需要指定协议为TCP；第二次握手为SYN,ACK为1，其余为0；第三次握手为ACK为1，其余为0；并且标记位不存在6个都为1或者都为0的报文

iptables -A INPUT -s 192.168.39.100 -p tcp --syn -j REJECT
#--syn也表示三次握手的第一次握手，即只检查6个标记位中的4个，并且syn位为1
```

tcp-flags使用：
```bash
#示例
iptables -t filter -I INPUT -p tcp -m tcp --dport 22 --tcp-flags SYN,ACK,FIN,RST,URG,PSH SYN -j REJECT
iptables -t filter -I OUTPUT -p tcp -m tcp --sport 22 --tcp-flags SYN,ACK,FIN,RST,URG,PSH SYN,ACK -j REJECT
iptables -t filter -I INPUT -p tcp -m tcp --dport 22 --tcp-flags ALL SYN -j REJECT
iptables -t filter -I OUTPUT -p tcp -m tcp --sport 22 --tcp-flags ALL SYN,ACK -j REJECT

注：
可以用ALL表示”SYN,ACK,FIN,RST,URG,PSH”。
使用”–syn”选项相当于使用”–tcp-flags SYN,RST,ACK,FIN  SYN”，也就是说，可以使用”–syn”选项去匹配tcp新建连接的请求报文。
```



### icmp模块
```bash
iptables -A INPUT -p icmp --icmp-type 0 -j ACCEPT
#filter表input链添加规则，允许客户端ping的响应报文；icmp协议中，type为0的表示为响应报文，为8的表示为请求报文；间接引用了icmp模块
```


## iprange扩展模块
### 介绍
在不使用任何扩展模块的情况下，使用-s选项或者-d选项即可匹配报文的源地址与目标地址，而且在指定IP地址时，可以同时指定多个IP地址，每个IP用”逗号”隔开，但是，-s选项与-d选项并不能一次性的指定一段连续的IP地址范围，如果我们需要指定一段连续的IP地址范围，可以使用iprange扩展模块。

使用iprange扩展模块可以指定”一段连续的IP地址范围”，用于匹配报文的源地址或者目标地址。
iprange扩展模块中有两个扩展匹配条件可以使用
–src-range
–dst-range

```bash
查看：iprange扩展模块中参数；
iptables -m iprange -h
```
![](attachments/Pasted%20image%2020240403141827.png)

### 范例
![](attachments/Pasted%20image%2020240325111604.png)
上例表示如果报文的源IP地址如果在192.168.1.127到192.168.1.146之间，则丢弃报文，IP段的始末IP使用”横杠”连接，–src-range与–dst-range和其他匹配条件一样，能够使用”!”取反。

### 其他
也可以使用 `ipset` 达到和 `iprange` 一样的效果。
但是ipset比iprange更加灵活，可以在不更改 iptables规则的情况下，更改ipset的内容，来达到更新网段的作用。

### 小结
包含的扩展匹配条件如下
–src-range：指定连续的源地址范围
–dst-range：指定连续的目标地址范围
```bash
#示例
iptables -t filter -I INPUT -m iprange --src-range 192.168.1.127-192.168.1.146 -j DROP
iptables -t filter -I OUTPUT -m iprange --dst-range 192.168.1.127-192.168.1.146 -j DROP
iptables -t filter -I INPUT -m iprange ! --src-range 192.168.1.127-192.168.1.146 -j DROP
```

## limit限速模块
### 介绍
`limit`模块主要是用于限制一个时间段内进入系统的数据包，主要的用途是减少泛洪攻击的影响。

我们能控制某条规则在一段时间内的匹配次数（也就是能匹配的包的数量），这样就能够减少 `DoS`、`syn flood`攻击的影响。这是他的主要作用，当然，更有非常多其他作用（注：比如，对于某些不常用的服务能限制连接数量，以免影响其他服务）。

`limit`模块引入了**令牌桶**的机制，使用`--limit-burst`指定令牌桶的大小，当令牌桶中有令牌时，则可以匹配到数据包，如果没有了令牌，则需要等到`--limit`指定的时间单位间隔向令牌桶加入令牌，才能继续匹配数据包。

```bash
iptables -m limit -h
```
![](attachments/Pasted%20image%2020240403142328.png)
### 理解
limit match的工作方式就像一个单位大门口的保安，当有人要进入时，需要找他办理通行证。早上上班时，保安手里有一定数量的通行证，来一个人，就签发一个，当通行证用完后，再来人就进不去了，但他们不会等，而是到别的地方去（在iptables里，这相当于一个包不符合某条规则，就会由后面的规则来处理，如果都不符合，就由缺省的策略处理）。
但有个规定，每隔一段时间保安就要签发一个新的通行证。这样，后面来的人如果恰巧赶上，也就能进去了。如果没有人来，那通行证就保留下来，以备来的人用。如果一直没人来，可用的通行证的数量就增加了，但不是无限增大的，最多也就是刚开始时保安手里有的那个数量。也就是说，刚开始时，通行证的数量是有限的，但每隔一段时间就有新的通行证可用。
`limit match`有两个参数就对应这种情况：
`--limit-burst`：指定刚开始时有多少通行证可用；其实`--limit-burst`就是指定令牌桶的最大容量。
`--limit`：指定要隔多长时间才能签发一个新的通行证；其实`--limit`表示向令牌桶放入令牌的速率。

要注意的是，我这里强调的是“签发一个新的通行证”，这是以iptables的角度考虑的。在你自己写规则时，就要从这个角度考虑。比如，你指定了--limit 3/minute --limit-burst 5 ，意思是开始时有5个通行证，用完之后每20秒增加一个（这就是从iptables的角度看的，要是以用户的角度看，说法就是每一分钟增加三个或每分钟只能过三个）。你要是想每20分钟过一个，只能写成--limit 3/hour --limit-burst 5，也就是说你要把时间单位凑成整的。


###  使用
`--limit`和`--limit-burst`规则匹配是iptables对数据包众多的匹配的方式中的一种，使用之前需要用"-m limit"，表示使用`limit`模块。

`--limit`和`--limit-burst`是配合使用的，如果`iptables`规则只使用了`--limit`，则表示使用`--limit-burst`的默认值为5。如果只指定了`--limit-burst`，则`--limit`默认值`3/hour`。

`limit match`也能用英文感叹号取反，如：`-m limit ! --limit 5/s`表示在数量超过限定值后，所有的包都会被匹配。

### 个数or速率限制
```bash
# --limit
iptables -A INPUT -m limit --limit 3/hour
# 为limit match设置最大平均匹配速率，也就是单位时间内limit match能匹配几个包。
他的形式是个数值加一个时间单位，能是/second /minute /hour /day 。
默认值是每小时3次（用户角度），即3/hour ，也就是每20分钟一次（iptables角度）。
```

```bash
iptables -A INPUT -p icmp -m limit --limit 6/m --limit-burst 5 -j ACCEPT
iptables -P INPUT DROP
# 然后从另一部主机上ping这部主机，就会发生如下的现象：
# 首先我们能看到前四个包的回应都非常正常，然后从第五个包开始，我们每10秒能收到一个正常的回应。# 这是因为我们设定了单位时间(在这里是每分钟)内允许通过的数据包的个数是每分钟6个，也即每10秒钟一个；# 其次我们又设定了事件触发阀值为5，所以我们的前四个包都是正常的，只是从第五个包开始，限制规则开始生效，故只能每10秒收到一个正常回应。
# 假设我们停止ping，30秒后又开始ping，这时的现象是：
# 前两个包是正常的，从第三个包开始丢包，这是因为在这里我的允许一个包通过的周期是10秒，如果在一个周期内系统没有收到符合条件的包，系统的触发值就会恢复1# 所以如果我们30秒内没有符合条件的包通过，系统的触发值就会恢复到3，如果5个周期内都没有符合条件的包通过，系统都触发值就会完全恢复。
```

### 小结
常用的扩展匹配条件如下
–limit-burst：类比”令牌桶”算法，此选项用于指定令牌桶中令牌的最大数量。
–limit：类比”令牌桶”算法，此选项用于指定令牌桶中生成新令牌的频率，可用时间单位有second、minute 、hour、day。
```bash
#示例 #注意，如下两条规则需配合使用，具体原因上文已经解释过，忘记了可以回顾。
iptables -t filter -I INPUT -p icmp -m limit --limit-burst 3 --limit 10/minute -j ACCEPT
iptables -t filter -A INPUT -p icmp -j REJECT
```

## string扩展模块
### 介绍
使用string扩展模块，可以指定要匹配的字符串，如果报文中包含对应的字符串，则符合匹配条件。
```bash
查看  string 模块的参数：
iptables -m string -h
```
![](attachments/Pasted%20image%2020240403144111.png)
### 范例
比如，如果报文中包含字符”OOXX”，我们就丢弃当前报文。
首先，我们在IP为146的主机上启动http服务，然后在默认的页面目录中添加两个页面，页面中的内容分别为”OOXX”和”Hello World”，如下图所示，在没有配置任何规则时，126主机可以正常访问146主机上的这两个页面。
![](attachments/Pasted%20image%2020240325105820.png)

那么，我们想要达到的目的是，如果报文中包含”OOXX”字符，我们就拒绝报文进入本机，所以，我们可以在126上进行如下配置。

![](attachments/Pasted%20image%2020240325105833.png)

上图中，’-m string’表示使用string模块，’--algo bm’表示使用bm算法去匹配指定的字符串，’ --string “OOXX” ‘则表示我们想要匹配的字符串为”OOXX”

设置完上图中的规则后，由于index.html中包含”OOXX”字符串，所以，146的回应报文无法通过126的INPUT链，所以无法获取到页面对应的内容。

### 小结
总结一下string模块的常用选项：
–algo：指定对应的匹配算法，可用算法为bm、kmp，此选项为必需选项。
–string：指定需要匹配的字符串
```bash
#示例
iptables -t filter -I INPUT -p tcp --sport 80 -m string --algo bm --string "OOXX" -j REJECT
iptables -t filter -I INPUT -p tcp --sport 80 -m string --algo bm --string "OOXX" -j REJECT
```

## time扩展模块
### 介绍
我们可以通过time扩展模块，根据时间段区匹配报文，如果报文到达的时间在指定的时间范围以内，则符合匹配条件。

### 范例
比如，”我想要自我约束，每天早上9点到下午6点不能看网页”，擦，多么残忍的规定，如果你想要这样定义，可以尝试使用如下规则。
![](attachments/Pasted%20image%2020240325110101.png)
上图中”-m time”表示使用time扩展模块，`--timestart`选项用于指定起始时间，`--timestop`选项用于指定结束时间。

如果你想要换一种约束方法，只有周六日不能看网页，那么可以使用如下规则。
![](attachments/Pasted%20image%2020240325110144.png)
使用–weekdays选项可以指定每个星期的具体哪一天，可以同时指定多个，用逗号隔开，除了能够数字表示”星期几”,还能用缩写表示，例如：Mon, Tue, Wed, Thu, Fri, Sat, Sun；

当然，你也可以将上述几个选项结合起来使用，比如指定只有周六日的早上9点到下午6点不能浏览网页。
![](attachments/Pasted%20image%2020240325110209.png)

使用–monthdays选项可以具体指定的每个月的哪一天，比如，如下图设置表示指明每月的22号，23号。
![](attachments/Pasted%20image%2020240325110233.png)

前文已经总结过，**当一条规则中同时存在多个条件时，多个条件之间默认存在”与”的关系**。
下图中的设置表示匹配的时间必须为星期5，并且这个”星期5″同时还需要是每个月的22号到28号之间的一天，所以，下图中的设置表示每个月的第4个星期5：
![](attachments/Pasted%20image%2020240325110317.png)


除了使用–weekdays选项与–monthdays选项，还可以使用–datestart 选项与-datestop选项，指定具体的日期范围，如下。
![](attachments/Pasted%20image%2020240325110340.png)

### 小结
常用扩展匹配条件如下
–timestart：用于指定时间范围的开始时间，不可取反
–timestop：用于指定时间范围的结束时间，不可取反
–weekdays：用于指定”星期几”，可取反
–monthdays：用于指定”几号”，可取反
–datestart：用于指定日期范围的开始日期，不可取反
–datestop：用于指定日期范围的结束时间，不可取反
```bash
#示例
iptables -t filter -I OUTPUT -p tcp --dport 80 -m time --timestart 09:00:00 --timestop 19:00:00 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 443 -m time --timestart 09:00:00 --timestop 19:00:00 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --weekdays 6,7 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --monthdays 22,23 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time ! --monthdays 22,23 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --timestart 09:00:00 --timestop 18:00:00 --weekdays 6,7 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --weekdays 5 --monthdays 22,23,24,25,26,27,28 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --datestart 2017-12-24 --datestop 2017-12-27 -j REJECT
```

## connlimit扩展模块
### 介绍
使用connlimit扩展模块，根据每个客户端IP做并发连接数数量匹配限制，可防止Dos(Denial of Service，拒绝服务)攻击。
注意：我们不用指定IP，其默认就是针对”每个客户端IP”，即对单IP的并发连接数限制。

```bash
iptables -m connlimit -h
```
![](attachments/Pasted%20image%2020240403144550.png)

### 范例
我们想要限制，每个IP地址最多只能占用两个ssh链接远程到server端，我们则可以进行如下限制。
![](attachments/Pasted%20image%2020240325110737.png)

上例中，使用”-m connlimit”指定使用connlimit扩展，使用”–connlimit-above 2″表示限制每个IP的链接数量上限为2，再配合-p tcp –dport 22，即表示限制每个客户端IP的ssh并发链接数量不能高于2。

**centos6**中，我们可以对–connlimit-above选项进行取反，没错，老规矩，使用”!”对此条件进行取反，示例如下
![](attachments/Pasted%20image%2020240325110754.png)
上例表示，每个客户端IP的ssh链接数量只要不超过两个，则允许链接。
上例的规则**并不能表示**：每个客户端IP的ssh链接数量超过两个则拒绝链接。
也就是说，即使我们配置了上例中的规则，也不能达到”限制”的目的，所以我们通常并不会对此选项取反，因为既然使用了此选项，我们的目的通常就是”限制”连接数量。

> 注：centos7中iptables为我们提供了一个新的选项，`--connlimit-upto`，这个选项的含义与`! –commlimit-above`的含义相同，即链接数量未达到指定的连接数量之意，所以综上所述，`--connlimit-upto`选项也不常用。


刚才说过，–connlimit-above默认表示限制”每个IP”的链接数量，其实，我们还可以配合–connlimit-mask选项，去限制”某类网段”的链接数量，示例如下：
![](attachments/Pasted%20image%2020240325111019.png)
上例中，”–connlimit-mask 24″表示某个C类网段，没错，mask为掩码之意，所以将24转换成点分十进制就表示255.255.255.0，所以，上图示例的规则表示，一个最多包含254个IP的C类网络中，同时最多只能有2个ssh客户端连接到当前服务器，看来资源很紧俏啊！254个IP才有2个名额，如果一个IP同时把两个连接名额都占用了，那么剩下的253个IP连一个连接名额都没有了。

即：在不使用–connlimit-mask的情况下，连接数量的限制是针对”每个IP”而言的，当使用了–connlimit-mask选项以后，则可以针对”某类IP段内的一定数量的IP”进行连接数量的限制

### 小结
常用的扩展匹配条件如下
–connlimit-above：单独使用此选项时，表示限制每个IP的链接数量。
–connlimit-mask：此选项不能单独使用，在使用–connlimit-above选项时，配合此选项，则可以针对”某类IP段内的一定数量的IP”进行连接数量的限制，如果不明白可以参考上文的详细解释。
```bash
#示例
iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 2 -j REJECT
iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 20 --connlimit-mask 24 -j REJECT
iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 10 --connlimit-mask 27 -j REJECT
```

## comment模块
### 介绍
```bash
man iptables-extensions
```
![](attachments/Pasted%20image%2020240408122607.png)



## u32模块
### 介绍
```bash
man iptables-extensions
```
![](attachments/Pasted%20image%2020240408121754.png)

从tcp/udp的头或者载荷中最多提取4B的数据，查看是否是指定值。


## ipset扩展模块
### 介绍
```bash
man iptables-extensions
```
![](attachments/Pasted%20image%2020240408121141.png)

### 使用
```bash
查看 IPtables中使用 ipset:
iptables -m set -h
```
![](attachments/Pasted%20image%2020240403142112.png)

### 范例
```bash
[ ! ] --match-set setname flag [,flag] ...
```
 flag可以通过指定为src或dst，flag最多数量为6个，中间使用逗号隔开。

```bash
# 创建名称为test的ipset
 
ipset create test hash:ip,port
 
ipset add test 192.168.1.105,tcp:80
 
# 丢弃来自源IP192.168.1.105，源端口80的包
 
iptables -A INPUT -m set --match-set test src,src -j DROP
 
# 丢弃来自源IP192.168.1.105，目的端口80的包
 
iptables -A INPUT -m set --match-set test src,dst -j DROP
 
# 丢弃访问目的IP192.168.1.105，目的端口80的包
 
iptables -A INPUT -m set --match-set test dst,dst -j DROP
 
 
 
# 创建名称为test1的ipset
 
ipset create test1 hash:ip,mac
 
ipset add test1 192.168.1.105,3C:7C:3F:D7:94:AF
 
# 丢弃来自源IP192.168.1.105，源MAC 3C:7C:3F:D7:94:AF的包
 
iptables -A INPUT -m set --match-set test1 src,src -j DROP
 
其他情况同test一样
```


## statistic 模块
### 介绍
```bash
# man iptables-extensions
```
![](attachments/Pasted%20image%2020240408121029.png)
## conntrack 模块
### 介绍
```bash
# iptables -m conntrack -h
```
![](attachments/Pasted%20image%2020240408120149.png)

```bash
# man iptables-extensions
```
![](attachments/Pasted%20image%2020240408120315.png)


## state 模块
### 背景
当我们通过http的url访问某个网站的网页时，客户端向服务端的80端口发起请求，服务端再通过80端口响应我们的请求，于是，作为客户端，我们似乎应该理所应当的放行80端口，以便服务端回应我们的报文可以进入客户端主机，于是，我们在客户端放行了80端口。

在client上，一般都是我们主动请求80端口，80端口回应我们，但是一般不会出现80端口主动请求我们的情况。

**潜在问题**：
不管是”响应”我们的报文，还是”主动发送”给我们的报文，应该都是可以通过这两个端口的，那么仔细想想，这样是不是不太安全呢？

**解决方法**
针对对应的端口，我用–tcp-flags去匹配tcp报文的标志位，把外来的”第一次握手”的请求拒绝，是不是也可以呢？那么如果对方使用的是UDP协议或者ICMP协议呢？
似乎总是有一些不完美的地方。

**分析**
造成上述问题的”根源”在哪里，我们为了让”提供服务方”能够正常的”响应”我们的请求，于是在主机上开放了对应的端口，开放这些端口的同时，也出现了问题，别人利用这些开放的端口，”主动”的攻击我们，他们发送过来的报文并不是为了响应我们，而是为了主动攻击我们，好了，我们似乎找到了问题所在？


### 介绍
**（1）问题总结**：
怎样判断这些报文是为了回应我们之前发出的报文，还是主动向我们发送的报文呢？

我们可以通过iptables的state扩展模块解决上述问题，但是我们需要先了解一些state模块的相关概念，然后再回过头来解决上述问题。从字面上理解，state可以译为状态，但是我们也可以用一个高大上的词去解释它，**state模块可以让iptables实现”连接追踪”机制**。

**（2）什么是连接**
既然是”连接追踪”，则必然要有”连接”。
咱们就来聊聊什么是连接吧，一说到连接，你可能会下意识的想到tcp连接，但是，对于state模块而言的”连接”并不能与tcp的”连接”画等号；
**在TCP/IP协议簇中，UDP和ICMP是没有所谓的连接的，但是对于state模块来说，tcp报文、udp报文、icmp报文都是有连接状态的**；
我们可以这样认为，对于state模块而言，只要两台机器在”你来我往”的通信，就算建立起了连接

### 状态理解
对于state模块的连接而言，”连接”其中的报文可以分为5种状态，报文状态可以为NEW、ESTABLISHED、RELATED、INVALID、UNTRACKED

```bash
# iptables -m state -h
state match options:
 [!] --state [INVALID|ESTABLISHED|NEW|RELATED|UNTRACKED][,...]
				State(s) to match
```
![](attachments/Pasted%20image%2020240408120531.png)

各种状态的理解：
![](attachments/Pasted%20image%2020240403165021.png)

**NEW**：
连接中的第一个包，状态就是NEW，我们可以理解为新连接的第一个包的状态为NEW。
连接追踪信息库中不存在此连接的相关信息条目，则将其识别为第一次发出的请求，即new为一次连接的第一个请求；如断开连接后，重新建立连接，再次发送请求，此请求也为new；

**ESTABLISHED**：
NEW状态之后，连接追踪信息库中为其建立的条目失效之前期间内所进行的通信状态；
我们可以把NEW状态包后面的包的状态理解为ESTABLISHED，表示连接已建立。
![](attachments/Pasted%20image%2020240403165052.png)

**RELATED**：新发起的但与已有连接相关联的连接，如：ftp协议中的数据连接与命令连接之间的关系；

FTP服务端会建立两个链接，一个命令链接，一个数据链接。
命令连接负责服务端与客户端之间的命令传输；
数据连接负责服务端与客户端之间的数据传输；
但是具体使用什么端口进行数据传输，是由命令链接去控制的，所以，”数据连接”中的报文与”命令连接”是有”关系”的。
那么，”数据连接”中的报文可能就是RELATED状态，因为这些报文与”命令连接”中的报文有关系。
(注：如果想要对ftp进行连接追踪，需要单独加载对应的内核模块nf_conntrack_ftp，如果想要自动加载，可以配置/etc/sysconfig/iptables-config文件)

**INVALID**：
无效的连接：如果一个包没有办法被识别，或者这个包没有任何状态，那么这个包的状态就是INVALID；如flag标记位不正确，如TCP6个标记位都为0或者都为1；
我们可以主动屏蔽状态为INVALID的报文。

**UNTRACKED**：
未进行追踪的连接；报文的状态为untracked时，表示报文未被追踪，当报文的状态为Untracked时通常表示无法找到相关的连接。
如raw表中关闭追踪；

### 使用

（1）服务器禁止新用户通过22端口进行ssh连接，老客户不受到影响。
```bash
1、iptables -A INPUT -p tcp --dport 22 -m state --state ESTABLISHED,RELATED -j ACCEPT
#ESTABLISHED为已经建立连接的，RELATED为与已有连接相关的连接，这两个状态都代表老用户；允许老用户通过tcp协议访问22端口

2、iptables -A INPUT -j REJECT  #配合第一条规则使用，拒绝新用户连接22端口
```


（2）禁止别人主动发过来的指定端口的请求；只允许本端主动发起请求，别人进行响应；
![](attachments/Pasted%20image%2020240403165246.png)
当前主机IP为104，当放行ESTABLISHED与RELATED状态的包以后，并没有影响通过本机远程ssh到IP为77的主机上，但是无法从77上使用22端口主动连接到104上。


### 注意
state连接跟踪功能是通过**conntrack模块**实现的，通过“lsmod | grep conntrack”查看内核中是否加载了该模块。
**iptables规则中调用了state模块，则内核会自动加载conntrack模块**，否则不会加载此模块；


```bash
/proc/net/nf_conntrack         
#此文件为跟踪库，记录了连接过本机的IP，但是记录在内存中，终究是有限制的 

/proc/sys/net/nf_conntrack_max 
#此文件记录了最大跟踪连接的数量，超过此文件数量的连接将会被拒绝,且在messages日志中打印错误日志；连接跟踪会占用较多内存，加大系统负载;
比如：错误日志：nf_conntrack: table full, dropping packet

/proc/sys/net/netfilter/       
#此目录下的文件记录了各个协议的连接超时时长

grep -i conntrack /proc/slabinfo
```

`nf_conntrack` 工作在 3 层，支持 IPv4 和 IPv6；而 `ip_conntrack` 只支持 IPv4。
目前，大多的 `ip_conntrack_*` 已被 `nf_conntrack_*` 取代，
很多 `ip_conntrack_*` 仅仅是个 alias，原先的 `ip_conntrack` 的 `/proc/sys/net/ipv4/netfilter/` 依然存在，
但是新的 `nf_conntrack` 在 `/proc/sys/net/netfilter/` 中；

```bash
conntrack 的 相关系统配置：

# sysctl -a | grep -i conntrack
net.netfilter.nf_conntrack_acct = 0
net.netfilter.nf_conntrack_buckets = 65536
net.netfilter.nf_conntrack_checksum = 1
net.netfilter.nf_conntrack_count = 0
net.netfilter.nf_conntrack_dccp_loose = 1
net.netfilter.nf_conntrack_dccp_timeout_closereq = 64
net.netfilter.nf_conntrack_dccp_timeout_closing = 64
net.netfilter.nf_conntrack_dccp_timeout_open = 43200
net.netfilter.nf_conntrack_dccp_timeout_partopen = 480
net.netfilter.nf_conntrack_dccp_timeout_request = 240
net.netfilter.nf_conntrack_dccp_timeout_respond = 480
net.netfilter.nf_conntrack_dccp_timeout_timewait = 240
net.netfilter.nf_conntrack_events = 1
net.netfilter.nf_conntrack_expect_max = 1024
net.netfilter.nf_conntrack_generic_timeout = 600
net.netfilter.nf_conntrack_helper = 0
net.netfilter.nf_conntrack_icmp_timeout = 30
net.netfilter.nf_conntrack_log_invalid = 0
net.netfilter.nf_conntrack_max = 262144
net.netfilter.nf_conntrack_sctp_timeout_closed = 10
net.netfilter.nf_conntrack_sctp_timeout_cookie_echoed = 3
net.netfilter.nf_conntrack_sctp_timeout_cookie_wait = 3
net.netfilter.nf_conntrack_sctp_timeout_established = 432000
net.netfilter.nf_conntrack_sctp_timeout_heartbeat_acked = 210
net.netfilter.nf_conntrack_sctp_timeout_heartbeat_sent = 30
net.netfilter.nf_conntrack_sctp_timeout_shutdown_ack_sent = 3
net.netfilter.nf_conntrack_sctp_timeout_shutdown_recd = 0
net.netfilter.nf_conntrack_sctp_timeout_shutdown_sent = 0
net.netfilter.nf_conntrack_tcp_be_liberal = 0
net.netfilter.nf_conntrack_tcp_loose = 1
net.netfilter.nf_conntrack_tcp_max_retrans = 3
net.netfilter.nf_conntrack_tcp_timeout_close = 10
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_established = 432000
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_last_ack = 30
net.netfilter.nf_conntrack_tcp_timeout_max_retrans = 300
net.netfilter.nf_conntrack_tcp_timeout_syn_recv = 60
net.netfilter.nf_conntrack_tcp_timeout_syn_sent = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_unacknowledged = 300
net.netfilter.nf_conntrack_timestamp = 0
net.netfilter.nf_conntrack_udp_timeout = 30
net.netfilter.nf_conntrack_udp_timeout_stream = 180
net.nf_conntrack_max = 262144
```


# iptables动作
## 常见动作
## REJECT 动作
## LOG 动作
### 介绍
```bash
man iptables-extensions
```
![](attachments/Pasted%20image%2020240408122804.png)

## SNAT 动作
## DNAT 动作
## MASQUERADE 动作
## REDIRECT 动作

# iptables自定义链
## 介绍
## 应用
## 使用
把自定义规则添加到自定义链中，然后再把自定义链，链接到系统自带的5个链中；
关联自定义链类似于脚本中的函数引用，先定义好函数，然后再引用函数。
# iptables的黑白名单机制
# iptables的防火墙机制
# iptables常用套路
# 参考
```bash
# linux Netfilter在网络层的实现详细分析（iptables）
https://zhuanlan.zhihu.com/p/694494005


```