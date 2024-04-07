```table-of-contents
```
# conntrack模块
nf_conntrack模块在kernel 2.6.15（2006-01-03 发布） 被引入，支持IPv4 和IPv6；取代只支持IPv4 的ip_connktrack，用于跟踪一个连接的状态。

**连接状态跟踪可以供其他模块使用，最常见的两个使用场景是 iptables 的 nat 表的 state 模块**。


# 配置和查看
## 内核相关参数
conntrack 的 相关系统配置：
```bash
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


说明：
```bash
# 启用连接跟踪流记帐。每个流添加64位字节和数据包计数器。(BOOLEAN：默认为零)  
nf_conntrack_acct

# 哈希表的大小，如果在模块加载期间未指定该参数，则通过将总内存除以16384来计算默认大小以确定存储区的数量，但是哈希表将永远不会少于32并且限制为16384个存储区。 对于内存超过4GB的系统，它将是65536个桶。 此sysctl只能在初始网络命名空间中写入。（INTEGER）  
nf_conntrack_buckets  

# 验证传入数据包的校验和。校验和错误的数据包处于INVALID状态。如果启用此选项，则不会考虑此类数据包进行连接跟踪。(BOOLEAN：默认为非零)  
nf_conntrack_checksum  

# 当前分配的流条目数（INTEGER）  
nf_conntrack_count  

# 如果启用此选项，连接跟踪代码将通过ctnetlink为用户空间提供连接跟踪事件。（BOOLEAN：默认为非零）  
nf_conntrack_events  

# 期望表的最大大小。 默认值为nf_conntrack_buckets/256，最小值为1。（INTEGER）  
nf_conntrack_expect_max  

# 用于重组IPv6片段的最大内存。 当为此目的分配nf_conntrack_frag6_high_thresh字节的内存时，片段处理程序将抛出数据包，直到达到nf_conntrack_frag6_low_thresh。（INTEGER：默认是262144）  
nf_conntrack_frag6_high_thresh  

# 参见nf_conntrack_frag6_low_thresh（INTEGER：默认是196608）  
nf_conntrack_frag6_low_thresh  

# 将IPv6片段保留在内存中的时间（INTEGER：单位秒）  
nf_conntrack_frag6_timeout  

# 通用超时的默认值。 这指的是第4层未知/不支持的协议。（INTEGER：默认为600，单位秒）  
nf_conntrack_generic_timeout  

# 启用自动conntrack帮助程序分配。如果禁用，则需要设置iptables规则以将帮助程序分配给连接。 有关详细信息，请参阅iptables-extensions（8）手册页中的CT目标描述。  
nf_conntrack_helper  

# ICMP超时时间（INTEGER：默认为30秒）  
nf_conntrack_icmp_timeout  

# ICMP6超时时间（INTEGER：默认为30秒）  
nf_conntrack_icmpv6_timeout  

# 记录value指定类型的无效数据包（INTEGER）  
nf_conntrack_log_invalid  

# 连接跟踪表的大小（INTEGER：默认为nf_conntrack_buckets * 4）  
nf_conntrack_max  

# 在你所做的事情上保守一点，在你接受别人的事情上保持自由。如果它不是零，我们只将窗口RST段标记为无效（BOOLEAN：默认为零）  
nf_conntrack_tcp_be_liberal  

# 如果设置为零，我们将禁用拾取已建立的连接（BOOLEAN：默认为非零）  
nf_conntrack_tcp_loose  

# 在未收到来自目标的（可接受）ACK的情况下可以重新传输的最大数据包数。 如果达到此数量，将启动更短的计时器（INTEGER：默认为3）  
nf_conntrack_tcp_max_retrans  

# TCP连接状态为close的记录超时时间（INTEGER：默认为10秒）  
nf_conntrack_tcp_timeout_close  

# TCP连接状态为close_wait的记录超时时间（INTEGER：默认为60秒）  
nf_conntrack_tcp_timeout_close_wait  

# TCP连接状态为established的记录超时时间（INTEGER：默认为432000秒）  
nf_conntrack_tcp_timeout_established  

# TCP连接状态为fin_wait的记录超时时间（INTEGER：默认为120秒）  
nf_conntrack_tcp_timeout_fin_wait  

# TCP连接状态为last_ack的记录超时时间（INTEGER：默认为30秒）  
nf_conntrack_tcp_timeout_last_ack  

# （INTEGER：默认为300秒）  
nf_conntrack_tcp_timeout_max_retrans  

# TCP连接状态为syn_recv的记录超时时间（INTEGER：默认为60秒）  
nf_conntrack_tcp_timeout_syn_recv  

# TCP连接状态为syn_sent的记录超时时间（INTEGER：默认为120秒）  
nf_conntrack_tcp_timeout_syn_sent 

# TCP连接状态为syn_sent的记录超时时间（INTEGER：默认为120秒）  
nf_conntrack_tcp_timeout_time_wait  

# （INTEGER：默认为300秒）  
nf_conntrack_tcp_timeout_unacknowledged  

# （BOOLEAN：默认为零）  
nf_conntrack_timestamp  

# （INTEGER：默认为30秒）  
nf_conntrack_udp_timeout  

# （INTEGER：默认为120秒）  
nf_conntrack_udp_timeout_stream  

# （INTEGER：默认为30秒）  
nf_conntrack_gre_timeout  

# 如果检测到GRE流，将使用此扩展超时（INTEGER：默认为180秒）  
nf_conntrack_gre_timeout_stream
```


## 查看
```bash
conntrack内核参数列表：sudo sysctl -a | grep conntrack；

conntrack超时相关参数：sudo sysctl -a | grep conntrack | grep timeout；

conntrack跟踪表的大小（桶的数量）：sudo sysctl net.netfilter.nf_conntrack_buckets；

conntrack最大跟踪连接数：sudo sysctl net.netfilter.nf_conntrack_max；

netfilter模块加载时的默认值：sudo dmesg | grep conntrack；

conntrack跟踪表使用情况：sudo sysctl net.netfilter.nf_conntrack_count；

四层协议类型和连接数：sudo cat /proc/net/nf_conntrack | awk '{sum[$3]++} END {for(i in sum) print i, sum[i]}'；

TCP 连接各状态对应的条数：sudo cat /proc/net/nf_conntrack | awk '/^.*tcp.*$/ {sum[$6]++} END {for(i in sum) print i, sum[i]}'；

三层协议类型和连接数：sudo cat /proc/net/nf_conntrack | awk '{sum[$1]++} END {for(i in sum) print i, sum[i]}'；

连接数最多的10个IP地址：sudo cat /proc/net/nf_conntrack | awk '{print $7}' | cut -d "=" -f 2 | sort | uniq -c | sort -nr | head -n 10；
```

# 参考