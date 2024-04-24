```table-of-contents
```
# 前言
要查看Linux系统中的TCP连接，可以使用以下几个命令：

1. netstat命令：使用netstat命令可以显示出当前系统中的网络连接情况。可以通过加上参数-t或-a来过滤只显示TCP连接，如：  
“`  
netstat -t #只显示TCP连接  
netstat -ta #显示所有TCP连接，包括监听和已建立连接  
“`

2. ss命令：ss命令是在netstat的基础上进行改进的，更加高效。可以使用以下命令来查看TCP连接：  
“`  
ss -t #只显示TCP连接  
ss -ta #显示所有TCP连接，包括监听和已建立连接  
“`

3. lsof命令：lsof命令可以列出系统中打开的文件和网络连接。可以通过加上参数-i来过滤只显示TCP连接，如：  
“`  
lsof -i tcp #只显示TCP连接  
“`

4. /proc文件系统：Linux系统中的/proc文件系统提供了很多关于系统状态的信息，包括当前的网络连接。可以通过访问`/proc/net/tcp`和`/proc/net/tcp6`文件来查看TCP连接的详细信息，如：  
“`  
cat /proc/net/tcp #显示IPv4的TCP连接信息  
cat /proc/net/tcp6 #显示IPv6的TCP连接信息  
“`

以上是在Linux系统中查看TCP连接的几种常用方法，根据实际情况选择合适的命令来查看。

# ss命令
## 介绍
ss是Socket Statistics的缩写。

netstat命令大家肯定已经很熟悉了，但是在2001年的时候netstat 1.42版本之后就没更新了，之后取代的工具是ss命令，是iproute2 package的一员。

> ​ rpm -ql iproute | grep ss  
> ​ /usr/sbin/ss

netstat的替代工具是nstat，当然netstat的大部分功能ss也可以替代；

## ss 和  netstat 比较

ss可以显示跟netstat类似的信息，但是速度却比netstat快很多。

netstat是基于/proc/net/tcp获取 TCP socket 的相关统计信息，用strace跟踪一下netstat查询tcp的连接，会看到他open的是/proc/net/tcp的信息。

ss快的秘密就在于它利用的是TCP协议的tcp_diag模块，而且是从内核直接读取信息，**当内核不支持 tcp_diag 内核模块时，会回退到 /proc/net/tcp 模式**。

/proc/net/snmp 存放的是系统启动以来的累加值，netstat -s 读取它  
/proc/net/tcp 是存放目前活跃的tcp连接的统计值，连接断开统计值清空， ss -it 读取它

## 参数说明

```bash
       -n, --numeric
              Do not try to resolve service names. Show exact bandwidth
              values, instead of human-readable.
       -l, --listening
              Display only listening sockets (these are omitted by
              default).
       -o, --options
              Show timer information. For TCP protocol, the output
              format is:

              timer:(<timer_name>,<expire_time>,<retrans>)

              <timer_name>
                     the name of the timer, there are five kind of timer
                     names:

                     on : means one of these timers: TCP retrans timer,
                     TCP early retrans timer and tail loss probe timer

                     keepalive: tcp keep alive timer

                     timewait: timewait stage timer

                     persist: zero window probe timer

                     unknown: none of the above timers

              <expire_time>
                     how long time the timer will expire

              <retrans>
                     how many times the retransmission occurred

       -e, --extended
              Show detailed socket information. The output format is:

              uid:<uid_number> ino:<inode_number> sk:<cookie>

              <uid_number>
                     the user id the socket belongs to

              <inode_number>
                     the socket's inode number in VFS

              <cookie>
                     an uuid of the socket
       -m, --memory
              Show socket memory usage. The output format is:

              skmem:(r<rmem_alloc>,rb<rcv_buf>,t<wmem_alloc>,tb<snd_buf>,
                            f<fwd_alloc>,w<wmem_queued>,o<opt_mem>,
                            bl<back_log>,d<sock_drop>)

              <rmem_alloc>
                     the memory allocated for receiving packet

              <rcv_buf>
                     the total memory can be allocated for receiving
                     packet

              <wmem_alloc>
                     the memory used for sending packet (which has been
                     sent to layer 3)

              <snd_buf>
                     the total memory can be allocated for sending
                     packet

              <fwd_alloc>
                     the memory allocated by the socket as cache, but
                     not used for receiving/sending packet yet. If need
                     memory to send/receive packet, the memory in this
                     cache will be used before allocate additional
                     memory.

              <wmem_queued>
                     The memory allocated for sending packet (which has
                     not been sent to layer 3)

              <opt_mem>
                     The memory used for storing socket option, e.g.,
                     the key for TCP MD5 signature

              <back_log>
                     The memory used for the sk backlog queue. On a
                     process context, if the process is receiving
                     packet, and a new packet is received, it will be
                     put into the sk backlog queue, so it can be
                     received by the process immediately

       -i, --info
              Show internal TCP information. Below fields may appear:

              ts     show string "ts" if the timestamp option is set

              sack   show string "sack" if the sack option is set

              ecn    show string "ecn" if the explicit congestion
                     notification option is set

              ecnseen
                     show string "ecnseen" if the saw ecn flag is found
                     in received packets

              fastopen
                     show string "fastopen" if the fastopen option is
                     set

              cong_alg
                     the congestion algorithm name, the default
                     congestion algorithm is "cubic"

              wscale:<snd_wscale>:<rcv_wscale>
                     if window scale option is used, this field shows
                     the send scale factor and receive scale factor

              rto:<icsk_rto>
                     tcp re-transmission timeout value, the unit is
                     millisecond

              backoff:<icsk_backoff>
                     used for exponential backoff re-transmission, the
                     actual re-transmission timeout value is icsk_rto <<
                     icsk_backoff

              rtt:<rtt>/<rttvar>
                     rtt is the average round trip time, rttvar is the
                     mean deviation of rtt, their units are millisecond

              ato:<ato>
                     ack timeout, unit is millisecond, used for delay
                     ack mode

              mss:<mss>
                     max segment size

              cwnd:<cwnd>
                     congestion window size

              pmtu:<pmtu>
                     path MTU value

              ssthresh:<ssthresh>
                     tcp congestion window slow start threshold

              bytes_acked:<bytes_acked>
                     bytes acked

              bytes_received:<bytes_received>
                     bytes received

              segs_out:<segs_out>
                     segments sent out

              segs_in:<segs_in>
                     segments received

              send <send_bps>bps
                     egress bps

              lastsnd:<lastsnd>
                     how long time since the last packet sent, the unit
                     is millisecond

              lastrcv:<lastrcv>
                     how long time since the last packet received, the
                     unit is millisecond

              lastack:<lastack>
                     how long time since the last ack received, the unit
                     is millisecond

              pacing_rate <pacing_rate>bps/<max_pacing_rate>bps
                     the pacing rate and max pacing rate

              rcv_space:<rcv_space>
                     a helper variable for TCP internal auto tuning
                     socket receive buffer

              tcp-ulp-mptcp flags:[MmBbJjecv]
              token:<rem_token(rem_id)/loc_token(loc_id)> seq:<sn>
              sfseq:<ssn> ssnoff:<off> maplen:<maplen>
                     MPTCP subflow information

       -s, --summary
              Print summary statistics. This option does not parse
              socket lists obtaining summary from various sources. It is
              useful when amount of sockets is so huge that parsing
              /proc/net/tcp is painful.

       -N NSNAME, --net=NSNAME
              Switch to the specified network namespace name.

       -b, --bpf
              Show socket classic BPF filters (only administrators are
              allowed to get these information).

       -4, --ipv4
              Display only IP version 4 sockets (alias for -f inet).

       -6, --ipv6
              Display only IP version 6 sockets (alias for -f inet6).

       -0, --packet
              Display PACKET sockets (alias for -f link).

       -t, --tcp
              Display TCP sockets.

       -u, --udp
              Display UDP sockets.

       -d, --dccp
              Display DCCP sockets.

       -w, --raw
              Display RAW sockets.

       -x, --unix
              Display Unix domain sockets (alias for -f unix).



              <sock_drop>
                     the number of packets dropped before they are de-
                     multiplexed into the socket

```

## ss 查看Buffer窗口
### -m 输出说明
```bash
-m, --memory  //查看每个连接的buffer使用情况  
              Show socket memory usage. The output format is:  
  
              skmem:(r<rmem_alloc>,rb<rcv_buf>,t<wmem_alloc>,tb<snd_buf>, 
                            f<fwd_alloc>,w<wmem_queued>,o<opt_mem>,  
                            bl<back_log>,d<sock_drop>)
                    
```

![](attachments/Pasted%20image%2020240515132315.png)

### -m 源码分析

```c

      printf(" skmem:(r%u,rb%u,t%u,tb%u,f%u,w%u,o%u",  
               skmeminfo[SK_MEMINFO_RMEM_ALLOC],  
               skmeminfo[SK_MEMINFO_RCVBUF],  
               skmeminfo[SK_MEMINFO_WMEM_ALLOC],  
               skmeminfo[SK_MEMINFO_SNDBUF],  
               skmeminfo[SK_MEMINFO_FWD_ALLOC],  
               skmeminfo[SK_MEMINFO_WMEM_QUEUED],  
               skmeminfo[SK_MEMINFO_OPTMEM]);  
  
        if (RTA_PAYLOAD(tb[attrtype]) >=  
                (SK_MEMINFO_BACKLOG + 1) * sizeof(__u32))  
                printf(",bl%u", skmeminfo[SK_MEMINFO_BACKLOG]);  
  
        if (RTA_PAYLOAD(tb[attrtype]) >=  
                (SK_MEMINFO_DROPS + 1) * sizeof(__u32))  
                printf(",d%u", skmeminfo[SK_MEMINFO_DROPS]);  
  
        printf(")");  
          
          
net/core/sock.c line:3095  
void sk_get_meminfo(const struct sock *sk, u32 *mem)  
{  
	memset(mem, 0, sizeof(*mem) * SK_MEMINFO_VARS);  
  
	mem[SK_MEMINFO_RMEM_ALLOC] = sk_rmem_alloc_get(sk);  
	mem[SK_MEMINFO_RCVBUF] = sk->sk_rcvbuf;  
	mem[SK_MEMINFO_WMEM_ALLOC] = sk_wmem_alloc_get(sk);  
	mem[SK_MEMINFO_SNDBUF] = sk->sk_sndbuf;  
	mem[SK_MEMINFO_FWD_ALLOC] = sk->sk_forward_alloc;  
	mem[SK_MEMINFO_WMEM_QUEUED] = sk->sk_wmem_queued;  
	mem[SK_MEMINFO_OPTMEM] = atomic_read(&sk->sk_omem_alloc);  
	mem[SK_MEMINFO_BACKLOG] = sk->sk_backlog.len;  
	mem[SK_MEMINFO_DROPS] = atomic_read(&sk->sk_drops);  
}

```

### 范例
```bash
#ss -m | xargs -L 1 | grep "ESTAB" | awk '{ if($3>0 || $4>0) print $0 }'

#ss -m -n | xargs -L 1 | grep "tcp EST" | grep "t[1-9]"

# ss -m | xargs -L 1 | grep -E "[rwfto][1-9]"

# ss -itmpn 

# ss -itn
```

![](attachments/Pasted%20image%2020240515132703.png)


![](attachments/Pasted%20image%2020240515132916.png)

tb指可分配的发送buffer大小，不够还可以动态调整（应用没有写死的话）;


#### 查看socket的发送缓冲区、接收缓冲区

![](attachments/Pasted%20image%2020240515133045.png)



## ss 查看拥塞窗口、RTO


![](attachments/Pasted%20image%2020240515134157.png)

### RTO计算算法

 rto的定义，不让修改，必须通过rtt计算所得
1HZ 一般是1秒
```bash
#define TCP_RTO_MAX ((unsigned)(120*HZ))  

#define TCP_RTO_MIN ((unsigned)(HZ/5)) //在rt很小的环境中计算下来RTO基本等于TCP_RTO_MIN

```

rto和rtt单位都是毫秒，一般rto最小为200ms、最大为120秒。

RTO的计算依赖于RTT值，或者说一系列RTT值。rto=f(rtt)

```text
1.1. 在没有任何rtt sample的时候，RTO <- TCP_TIMEOUT_INIT (1s)  
多次重传时同样适用指数回避算法(backoff)增加RTO  
  
1.2. 获得第一个RTT sample后，  
SRTT <- RTT  
RTTVAR <- RTT/2  
RTO <- SRTT + max(G, K * RTTVAR)  
其中K=4, G表示timestamp的粒度(在CONFIG_HZ=1000时，粒度为1ms)  
  
1.3. 后续获得更多RTT sample后，  
RTTVAR <- (1 - beta) * RTTVAR + beta * |SRTT - R|  
SRTT <- (1 - alpha) * SRTT + alpha * R  
其中beta = 1/4, alpha = 1/8  
  
1.4. Whenever RTO is computed, if it is less than 1 second, then the  
RTO SHOULD be rounder up to 1 second.  
  
1.5. A maximum value MAY be placed on RTO provided it is at least 60 seconds.
```

RTTVAR表示的是平滑过的平均偏差，SRTT表示的平滑过的RTT。

### 从系统cache中查看 tcp_metrics item

![](attachments/Pasted%20image%2020240515134517.png)

每个连接的ssthresh默认是个无穷大的值，但是内核会cache对端ip上次的ssthresh（大部分时候两个ip之间的拥塞窗口大小不会变），这样大概率到达ssthresh之后就基本拥塞了，然后进入cwnd的慢增长阶段。



## ss 查看 timer 状态

```bash
ss -atonp

```

![](attachments/Pasted%20image%2020240515135033.png)


## ss -s 统计所有链接的状态

统计所有连接的状态

![](attachments/Pasted%20image%2020240515135251.png)


## 过滤
### 过滤地址和端口号

过滤目标端口是80的或者源端口是1723的连接，dst后面要跟空格然后加“：”：

![](attachments/Pasted%20image%2020240515134705.png)

```bash


ss -ant dport = :80 or sport = :1723


ss -ant dst 111.161.68.235

```

###  按连接状态过滤

```bash

ss -o state established '( dport = :http or sport = :http )'
```


# knetstat工具
最后给出的一个工具，knetstat（需要单独安装），也可以查看tcp的状态下的各种参数，需要单独安装。

## 查看已建立连接的tcp option


example(3306是本地server，4192是后端MySQL）：

![](attachments/Pasted%20image%2020240515123738.png)


3306对应的client上：

![](attachments/Pasted%20image%2020240515123605.png)

# /proc/net/tcp文件
# 参考
```bash
# 「八」Linux文件/proc/net/tcp分析
https://guanjunjian.github.io/2017/11/09/study-8-proc-net-tcp-analysis/

## 就是要你懂网络监控--ss用法大全
https://plantegg.github.io/2016/10/12/ss%E7%94%A8%E6%B3%95%E5%A4%A7%E5%85%A8/
```