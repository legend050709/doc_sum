```table-of-contents
```
# 背景
router 之间传输数据包大小的上限称为 MTU。当 router 发现数据包大小超出下一站 router 的上限时，它有2个选择:
1. 拆成更小的数据包送出。
2. 通知来源用更小的数据包大小 (送出 ICMP “packet too big” message)。

IPv4 有支持 1 & 2，但 1 会带给 router 太多负担 (memory & CPU)，可能会被用作攻攻击手段，所以很多 router 不支持1。IPv6 只支援 2。

# 分析
通过 ICMP 通知 “packet too big” 不断尝试找出端到端可行的 MTU，称为 `Path MTU Discovery (PMTUD)`。

但是有些 router 基于安全理由 (#1)，会过滤掉 ICMP 封包，所以 PMTUD 也不可靠。若用了太大的数据包，且中间的 router 回传的 ICMP “packet too big”到不了源，结果是 TCP handshake成功，但后面的大包传输失败，程序卡主。

最稳妥的作法是一开始就不要让数据包超出 MTU 上限。IPv4 規定 router 至少要能處理 576 bytes，IPv6 是1280 bytes。使用最小值固然能避开最坏情况，但是这样传输效率不好。

幸好如今的 Internet 几乎都用 Ethernet，Ethernet 标准规定至少有 1500 bytes，所以可以安心地假定 MTU = 1500。

Linux 上可以用 ping 作 PMTUD，下面是計算本機到 8.8.8.8 (Google DNS) 的 PMTU:
```c
$ ping -c 1 -M do -s 2000 8.8.8.8  
PING 8.8.8.8 (8.8.8.8) 2000(2028) bytes of data.  
ping: local error: Message too long, mtu=1500
```
# pmtu是什么
由于各种原因，分片被认为是有害的。为了避免数据包碎片，PMTUD的概念作为一种变通方法而得以实现。当然，一个好的操作系统应该使用PMTUD来最大程度地减少碎片。
# pmtu和mtu

# pmtu的原理
# 内核实现细节
## PMTU多种策略
```c
#define IP_PMTUDISC_DONT        0   /* Never send DF frames */
#define IP_PMTUDISC_WANT        1   /* Use per route hints     */
#define IP_PMTUDISC_DO             2   /* Always DF                   */
#define IP_PMTUDISC_PROBE       3   /* Ignore dst pmtu         */
#define IP_PMTUDISC_INTERFACE   4   /* 使用出接口的设备MTU值 */
#define IP_PMTUDISC_OMIT            5   /* 忽略DF位 */

```
- IP_PMTUDISC_DONT：
表示从不设置DF位，即不进行PMTU发现（参见函数ip_dont_fragment）。
- IP_PMTUDISC_WANT：
根据路由中表项是否锁定了MTU，来决定是否设置DF位，如锁定，不设置DF位。
- IP_PMTUDISC_DO：
总是设置DF位，除非内核设置了忽略df（ignore_df），参见以下内容。
- IP_PMTUDISC_INTERFACE：
不设置DF位，不发送设置了DF位并且长度超过出接口设备MTU的数据包。
- IP_PMTUDISC_OMIT：
其与IP_PMTUDISC_INTERFACE含义相同。唯一区别在于，即使数据包设置了DF位，内核也会对长度超过出接口设备MTU的数据包进行分片处理并发送。

## PMTU默认策略
通过porc文件ip_no_pmtu_disc 设置pmtu的全局默认策略（/proc/sys/net/ipv4/ip_no_pmtu_disc），在sock创建时根据此值初始化pmtudisc变量。
内核初始将其设置为0，即系统缺省的pmtu策略为IP_PMTUDISC_WANT，尝试进行pmtu发现。如果置0，不进行pmtu发现（IP_PMTUDISC_DONT），系统不发送IP头带有DF标志的报文。
```c
static int inet_create(struct net *net, struct socket *sock, int protocol, int kern)
{
    struct inet_sock *inet;
    //初始化sock中的pmtudisc值。
    if (net->sysctl_ip_no_pmtu_disc)
        inet->pmtudisc = IP_PMTUDISC_DONT;
    else
        inet->pmtudisc = IP_PMTUDISC_WANT;
}
```

## 使能PMTU路径发现（DF标志置位）
IP报头的DF标志，用于PMTU发现，由ip_queue_xmit函数可知(如下所示)，在满足以下两个条件之一：
1）pmtudisc等于IP_PMTUDISC_DO；  
2）pmtudisc等于IP_PMTUDISC_WANT，并且mtu没有在路由表中锁定。

内核没有设置忽略DF的条件下，设置IP报头的DF标志位，进行PMTU发现操作。IP_PMTUDISC_DO为使能PMTU发现策略，IP_PMTUDISC_WANT会根据mtu是否锁定进行pmtu发现。

```c
static inline int ip_dont_fragment(const struct sock *sk, const struct dst_entry *dst)
{
    u8 pmtudisc = READ_ONCE(inet_sk(sk)->pmtudisc);

    return  pmtudisc == IP_PMTUDISC_DO ||
        (pmtudisc == IP_PMTUDISC_WANT &&
         !ip_mtu_locked(dst));
}

/* Note: skb->sk can be different from sk, in case of tunnels */
int ip_queue_xmit(struct sock *sk, struct sk_buff *skb, struct flowi *fl)
{
    ...

    skb_push(skb, sizeof(struct iphdr) + (inet_opt ? inet_opt->opt.optlen : 0));
    skb_reset_network_header(skb);
    iph = ip_hdr(skb);
    *((__be16 *)iph) = htons((4 << 12) | (5 << 8) | (inet->tos & 0xff));
    if (ip_dont_fragment(sk, &rt->dst) && !skb->ignore_df)
        iph->frag_off = htons(IP_DF);
    else
        iph->frag_off = 0;
    iph->ttl      = ip_select_ttl(inet, &rt->dst);
    iph->protocol = sk->sk_protocol;
    ip_copy_addrs(iph, fl4);

    ...

    skb->priority = sk->sk_priority;
    skb->mark = sk->sk_mark;

    res = ip_local_out(net, sk, skb);
    rcu_read_unlock();
    return res;
}
```


sock结构体中pmtudisc变量可通过setsockopt系统调用进行设置。用户也可使用ip命令对MTU值进行锁定，不允许进行修改，如下命令，锁定lock到网关192.168.1.1的mtu值为1300字节大小：
```c
static int do_ip_setsockopt(...)
{
    struct inet_sock *inet = inet_sk(sk);
    
    case IP_MTU_DISCOVER:
        if (val < IP_PMTUDISC_DONT || val > IP_PMTUDISC_OMIT)
            goto e_inval;
        inet->pmtudisc = val;
}

# ip 命令锁定 mtu
ip route add 0.0.0.0 via 192.168.1.1 mtu lock 1300
```

## 关闭路径MTU发现
路径mtu发现策略设置为IP_PMTUDISC_INTERFACE或者IP_PMTUDISC_OMIT的时候，内核都不会保存ICMP消息中发送的新MTU值。ipv4_sk_update_pmtu函数判断之后直接返回。

除去这两种PMTU策略外，其它情况下，内核还是会保存通过ICMP接收到的新MTU值，这样在用户之后修改pmtu策略后能马上生效。另外在策略配置为非IP_PMTUDISC_DONT时，设置sock错误标志。
```c
static inline bool ip_sk_accept_pmtu(const struct sock *sk)
{
    return inet_sk(sk)->pmtudisc != IP_PMTUDISC_INTERFACE &&
           inet_sk(sk)->pmtudisc != IP_PMTUDISC_OMIT;
}

void ipv4_sk_update_pmtu(struct sk_buff *skb, struct sock *sk, u32 mtu)
{
    if (!ip_sk_accept_pmtu(sk))
        goto out;
}

void __udp4_lib_err(struct sk_buff *skb, u32 info, struct udp_table *udptable)
{
    case ICMP_DEST_UNREACH:
        if (code == ICMP_FRAG_NEEDED) { /* Path MTU discovery */
            ipv4_sk_update_pmtu(skb, sk, info);
            if (inet->pmtudisc != IP_PMTUDISC_DONT) {
                err = EMSGSIZE;
                harderr = 1;
                break;
            }
        }
}
```
## 强制关闭PMTU发现（忽略DF位）
在`ip_queue_xmit`函数中看到，如果`skb->ignore_df`为真，就会清除IP报头的DF位。
而ignore_df变量是通过函数ip_sk_ignore_df赋值。当`pmtudisc`策略设置成`IP_PMTUDISC_DONT`、`IP_PMTUDISC_WANT`或者`IP_PMTUDISC_OMIT`的时候，`ignore_df`变量为真，内核将会在发出的报文中清除DF标志位。
```c
static inline bool ip_sk_ignore_df(const struct sock *sk)
{
    return inet_sk(sk)->pmtudisc < IP_PMTUDISC_DO ||
           inet_sk(sk)->pmtudisc == IP_PMTUDISC_OMIT;
}
```

在分片处理函数中，即便设置了不允许分片DF标志位，只要ignore_df为真，强制进行分片。
```c
static int ip_fragment(...)
{
    if ((iph->frag_off & htons(IP_DF)) == 0)
        return ip_do_fragment(sk, skb, output);
 
    if (unlikely(!skb->ignore_df ||
             (IPCB(skb)->frag_max_size && IPCB(skb)->frag_max_size > mtu))) {
        icmp_send(skb, ICMP_DEST_UNREACH, ICMP_FRAG_NEEDED, htonl(mtu));
        return -EMSGSIZE;
    }
    return ip_do_fragment(sk, skb, output);
}
```

# 其他
## TCP的MTU Probe
TCP三次握手时MSS选项的值：一般情况下，都是由出口路由的MTU大小决定：MTU-40。也就是说，TCP在握手阶段，通过MSS选项，通知对端本端可以接收的最大报文长度是多少。
但是TCP连接只是一个“虚拟”的连接，其下层是无连接的IP网络。在IP网络中，数据包的传输路径是可变的，也就是说一个TCP连接，其报文可能从不同的IP路径传输到对端。不同的传输路径，自然会经过不同的网络设备，其MTU值自然不同。这样的话，即使对端按照MSS的值发送TCP报文，也可能会超过其中间路径的MTU值，导致数据包发送失败。

这就引出了一个问题：TCP如何避免这种PMTU（Path MTU）发生变化的情况呢？这就引出了`TCP PMTU Probing`。这个TCP功能是一个系统级别的参数，可以通过`/proc/sys/net/ipv4/tcp_mtu_probing`来打开这个功能。
### 思想
在TCP发送失败时，发送方会不断尝试降低MSS的大小，直至满足PMTU的限制，成功发送数据。
当PMTU变小时，MTU Probe通过丢包发现这种情况，从而不断的降低当前MSS值，达到成功发送的目的。

### 初始化
在内核发送或者接收syn报文时，就会调用tcp_mtup_init。
![](attachments/Pasted%20image%2020231129153824.png)
这个函数负责MTU探测的初始化，设置当前探测的上限、下限等。这里的下限比较明确，是通过系统设置的最小MSS值（默认为512字节）转换为MTU（加上40字节）。上限则是由rx_opt（接收的对端选项）的mss_clamp决定的。对于主动连接来说，其值为MSS的默认值（目前是536字节，在RFC1122和RFC2581中定义）。
### 探测时机
那么探测的行为什么时候发生呢？
第一个念头是通过定时器，定期的去探测PMTU。然而这实际上是非必要的。内核在这块的处理比较巧妙。TCP MTU探测的工作函数`tcp_mtu_probing`是在`tcp_write_timeout`中调用的。
当TCP重传超过设置的`sysctl_tcp_retries1`值（`/proc/sys/net/ipv4/tcp_retries1`）时，就会调用`tcp_mtu_probing`。
当`PMTU`小于`MSS`时，TCP报文就会传输失败——因为默认情况下，系统都会设置禁止IP分片，这时就需要进行`tcp_mtu_probing`。

### 探测实现
![](attachments/Pasted%20image%2020231129154208.png)
在这份代码中，MTU的下线探测还是比较激进的。
首先，取探测下限search_low计算的MSS的一半值，然后与系统配置的tcp_base_mss相比，取较小值。这个较小值，不能低于可能的最小值68-tcp_header_len，并根据结果重新设置了探测下限。通过这样的方法，内核会探测到真实的PMTU，从而保证TCP报文可以顺利发送。

## 设置
### socket 设置
在Linux中，PMTUD由`IP_MTU_DISCOVER`套接字选项控制 pmtu的开启关闭。
也可以通过 TCP_MAXSEG 的 osocketopt 设置 TCP socket 的 MSS。

### 内核设置
#### ip_no_pmtu_disc
要设置ip_no_pmtu_disc=0(默认值), 表示启用pmtu discovery， 这样报文发送的时候才会设置DF标记。

#### tcp_mtu_probing
Linux内核默认情况下未开启TCP的MTU探测功能。
```bash
#  cat /proc/sys/net/ipv4/tcp_mtu_probing
0
```
想要启用tcp mtu probe， 先要设置ip_no_pmtu_disc=0(默认值), 表示启用pmtu discovery， 这样tcp发送的时候才会设置DF标记。
通过DF标记，中间路由设备如果需要分片就会返回ICMP消息通知， 但是有可能因为防火墙等原因，发送方收不到ICMP消息。因此发送方一直发送探测包，却一直没收到回应， 这个就称为black hole。

```bash
# ping -s 2500  -M do  8.8.8.8 
PING 8.8.8.8 (8.8.8.8) 2500(2528) bytes of data.
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500
^C
--- 8.8.8.8 ping statistics ---
10 packets transmitted, 0 received, +10 errors, 100% packet loss, time 9213ms
```
![](attachments/Pasted%20image%2020231129145014.png)

### mtu(mss) 和 路由
通过`ip route` 设置 advmss。
![](attachments/Pasted%20image%2020231129155214.png)
#### MTU被路由缓存

```
$ ip route get 10.33.32.157
10.33.32.157 via 10.1.22.1 dev eth0  src 10.1.22.194 
    cache  expires 591sec mtu 1422
    
$ sudo ip route flush cache

$ ip route get 10.33.32.157
10.33.32.157 via 10.1.22.1 dev eth0  src 10.1.22.194 
    cache
```

- **查询路由缓存**
```shell
# ipv4 路由缓存
# ip -s route show table cache
or 
# route -C
or
# netstat -rn -C
or
# ip -s route get x.x.x.x

# ipv6 路由缓存
# ip -6 -s route show table cache
or 
# route -6 -C
or
# netstat -rn -C
```
![](attachments/Pasted%20image%2020231129174856.png)

# 测试
使用ping命令即可测试PMTU策略：
```text
ping 
       -M pmtudisc_opt
           Select Path MTU Discovery strategy.  pmtudisc_option may be
           either do (prohibit fragmentation, even local one), want (do PMTU
           discovery, fragment locally when packet size is large), or dont
           (do not set DF flag).
```
例如发送长度超过超过MTU值（1500）的数据包，并且设置IP头的DF位，系统提示message too long：
```c
ping -c 3 -s 1473 -M do 192.168.1.133
PING 192.168.1.133 (192.168.1.133) 1473(1501) bytes of data.
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500
ping: local error: Message too long, mtu=1500

--- 192.168.1.133 ping statistics ---

3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 1999ms
```
# 参考
```c
# 路径MTU（PMTU）发现控制与DF位
https://blog.csdn.net/sinat_20184565/article/details/80326262
```