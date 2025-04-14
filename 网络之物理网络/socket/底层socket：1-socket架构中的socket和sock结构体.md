```table-of-contents
```
# 概述
socket架构实现中，其实是采用了两个主要的结构体，struct socket 和 struct sock。
这两个结构体一起支撑起了socket架构，socket结构体和sock结构体实际上是同一事物的两个侧面，这就好像一枚硬币的正方面，这个比喻挺恰当。

不妨说：
socket结构体是面向进程和系统调用界面的一侧，也就是用户空间一侧
sock结构体则是面向驱动层一侧。
![](attachments/Pasted%20image%2020231204165820.png)
# 应用层socket
## 结构体
```c
typedef enum {
    SS_FREE = 0,         /* not allocated */
    SS_UNCONNECTED,      /* unconnected to any socket */
    SS_CONNECTING,       /* in process of connecting */
    SS_CONNECTED,        /* connected to socket */
    SS_DISCONNECTING     /* in process of disconnecting */
} socket_state;

struct socket
{
    socket_state state;      // 该state用来表明该socket的当前状态
    unsigned long flags;     // 该成员可能的值如下，该标志用来设置socket是否正在忙碌
                            /*
                                #define SOCK_ASYNC_NOSPACE 0
                                #define SOCK_ASYNC_WAITDATA 1
                                #define SOCK_NOSPACE 2
                            */
    struct proto_ops *ops;   //依据协议绑定到该socket上的特定的协议族的操作函数指针，例如IPv4 TCP就是inet_stream_ops
    struct inode *inode;     //表明该socket所属的inode
    struct fasync_struct *fasync_list; //异步唤醒队列
    struct file *file;       //file回指指针
    struct sock *sk;         //sock指针
    wait_queue_head_t wait;  //sock的等待队列，在TCP需要等待时就sleep在这个队列上
    short type;              //表示该socket在特定协议族下的类型例如SOCK_STREAM,
    unsigned char passcred;  //在TCP分析中无须考虑
};

```

## 函数
### socket系统调用
从 Socket 系统调用开始。
```c
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
  int retval;
  struct socket *sock;
  int flags;
......
  if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
    flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

  retval = sock_create(family, type, protocol, &sock);
......
  retval = sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
......
  return retval;
}
```
Socket 系统调用会调用 sock_create 创建一个 struct socket 结构，然后通过 sock_map_fd 和文件描述符对应起来。
第三个参数是 protocol，是协议。协议数目是比较多的，也就是说，多个协议会属于同一种类型。常用的 Socket 类型有三种，分别是 `SOCK_STREAM`、`SOCK_DGRAM` 和 `SOCK_RAW`。

### sock_create
我们重点看 SOCK_STREAM 类型和 IPPROTO_TCP 协议。
为了管理 family、type、protocol 这三个分类层次，内核会创建对应的数据结构。
```c
int __sock_create(struct net *net, int family, int type, int protocol,
       struct socket **res, int kern)
{
  int err;
  struct socket *sock;
  const struct net_proto_family *pf;
......
  sock = sock_alloc();
......
  sock->type = type;
......
  pf = rcu_dereference(net_families[family]);
......
  err = pf->create(net, sock, protocol, kern);
......
  *res = sock;

  return 0;
}
```
这里先是分配了一个 struct socket 结构。接下来我们要用到 family 参数。这里有一个 net_families 数组，我们可以以 family 参数为下标，找到对应的 struct net_proto_family。
```c
/* Supported address families. */
#define AF_UNSPEC  0
#define AF_UNIX    1  /* Unix domain sockets     */
#define AF_LOCAL  1  /* POSIX name for AF_UNIX  */
#define AF_INET    2  /* Internet IP Protocol   */
......
#define AF_INET6  10  /* IP version 6      */
......
#define AF_MPLS    28  /* MPLS */
......
#define AF_MAX    44  /* For now.. */
#define NPROTO    AF_MAX

struct net_proto_family __rcu *net_families[NPROTO] __read_mostly;
```
我们可以找到 net_families 的定义。每一个地址族在这个数组里面都有一项，里面的内容是 net_proto_family。
> 每一种地址族都有自己的 net_proto_family，IP 地址族的 net_proto_family 定义如下，里面最重要的就是，create 函数指向 inet_create。
```c
//net/ipv4/af_inet.c
static const struct net_proto_family inet_family_ops = {
  .family = PF_INET,
  .create = inet_create,//这个用于socket系统调用创建
......
}
```
### inet_create
回到函数 `__sock_create`。接下来，在这里面，这个 `inet_create` 会被调用。
```c
static int inet_create(struct net *net, struct socket *sock, int protocol, int kern)
{
  struct sock *sk;
  struct inet_protosw *answer;
  struct inet_sock *inet;
  struct proto *answer_prot;
  unsigned char answer_flags;
  int try_loading_module = 0;
  int err;

  /* Look for the requested type/protocol pair. */
lookup_protocol:
  list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {
    err = 0;
    /* Check the non-wild match. */
    if (protocol == answer->protocol) {
      if (protocol != IPPROTO_IP)
        break;
    } else {
      /* Check for the two wild cases. */
      if (IPPROTO_IP == protocol) {
        protocol = answer->protocol;
        break;
      }
      if (IPPROTO_IP == answer->protocol)
        break;
    }
    err = -EPROTONOSUPPORT;
  }
......
  sock->ops = answer->ops;
  answer_prot = answer->prot;
  answer_flags = answer->flags;
......
  sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot, kern);
......
  inet = inet_sk(sk);
  inet->nodefrag = 0;
  if (SOCK_RAW == sock->type) {
    inet->inet_num = protocol;
    if (IPPROTO_RAW == protocol)
      inet->hdrincl = 1;
  }
  inet->inet_id = 0;
  sock_init_data(sock, sk);

  sk->sk_destruct     = inet_sock_destruct;
  sk->sk_protocol     = protocol;
  sk->sk_backlog_rcv = sk->sk_prot->backlog_rcv;

  inet->uc_ttl  = -1;
  inet->mc_loop  = 1;
  inet->mc_ttl  = 1;
  inet->mc_all  = 1;
  inet->mc_index  = 0;
  inet->mc_list  = NULL;
  inet->rcv_tos  = 0;

  if (inet->inet_num) {
    inet->inet_sport = htons(inet->inet_num);
    /* Add to protocol hash chains. */
    err = sk->sk_prot->hash(sk);
  }

  if (sk->sk_prot->init) {
    err = sk->sk_prot->init(sk);
  }
......
}
```
在` inet_create` 中，我们先会看到一个循环 `list_for_each_entry_rcu`。在这里，第二个参数 type 开始起作用。因为循环查看的是 `inetsw[sock->type]`。

> 这里的 `inetsw` 也是一个数组，type 作为下标，里面的内容是 `struct inet_protosw`，是协议，也即 `inetsw` 数组对于每个类型有一项，这一项里面是属于这个类型的协议。
```c
static struct inet_protosw inetsw_array[] =
{
  {
    .type =       SOCK_STREAM,
    .protocol =   IPPROTO_TCP,
    .prot =       &tcp_prot,
    .ops =        &inet_stream_ops,
    .flags =      INET_PROTOSW_PERMANENT |
            INET_PROTOSW_ICSK,
  },
  {
    .type =       SOCK_DGRAM,
    .protocol =   IPPROTO_UDP,
    .prot =       &udp_prot,
    .ops =        &inet_dgram_ops,
    .flags =      INET_PROTOSW_PERMANENT,
     },
     {
    .type =       SOCK_DGRAM,
    .protocol =   IPPROTO_ICMP,
    .prot =       &ping_prot,
    .ops =        &inet_sockraw_ops,
    .flags =      INET_PROTOSW_REUSE,
     },
     {
        .type =       SOCK_RAW,
      .protocol =   IPPROTO_IP,  /* wild card */
      .prot =       &raw_prot,
      .ops =        &inet_sockraw_ops,
      .flags =      INET_PROTOSW_REUSE,
     }
}
```

在` inet_create` 中，接下来，`struct socket *sock` 的 `ops` 成员变量，被赋值为 answer 的 ops。对于 TCP 来讲，就是` inet_stream_ops`。后面任何用户对于这个 socket 的操作，都是通过` inet_stream_ops` 进行的。

接下来，我们创建一个 `struct sock *sk` 对象。
> socket 和 sock 看起来几乎一样，容易让人混淆，这里需要说明一下，socket 是用于负责对上给用户提供接口，并且和文件系统关联。而 sock，负责向下对接内核网络协议栈。

在 inet_create 函数中，接下来创建一个 struct inet_sock 结构，这个结构一开始就是 struct sock，然后扩展了一些其他的信息，剩下的代码就填充这些信息。
> 将一个结构放在另一个结构的开始位置，然后扩展一些成员，通过对于指针的强制类型转换，来访问这些成员。


# 内核中的sock

## 常用inet sock的继承关系
```c
tcp_sock -> inet_connection_sock -> inet_sock -> sock -> sock_common
                        udp_sock -> inet_sock -> sock -> sock_common
                                    unix_sock -> sock -> sock_common
```
从中我们可以明显地看出，TCP套接字是一种面向连接的INET套接字；UDP套接字只是INET套接字，不需要维护连接信息；UNIX域套接字和INET域套接字一样都继承于`sock`基类。

## sock族类
### 共有的 sock
#### sock_common
`sock_common`包含了套接字最基础的内容，一些mini sock会直接继承这个类，而非继承`sock`。它包含了地址对、端口对、套接字协议族、连接状态等内容。

值得一提的是，不管是流套接字还是数据报套接字，它们都使用TCP的状态来填充`sock_common`成员中的连接状态，这就非常灵性，也许是因为可靠传输协议的状态集可以涵盖不可靠传输协议的状态集？
```c
enum {
	TCP_ESTABLISHED = 1,
	TCP_SYN_SENT,
	TCP_SYN_RECV,
	TCP_FIN_WAIT1,
	TCP_FIN_WAIT2,
	TCP_TIME_WAIT,
	TCP_CLOSE,
	TCP_CLOSE_WAIT,
	TCP_LAST_ACK,
	TCP_LISTEN,
	TCP_CLOSING,	/* Now a valid state */
	TCP_NEW_SYN_RECV,

	TCP_MAX_STATES	/* Leave at the end! */
};
```

#### sock
`sock`结构体是BSD通用套接字`socket`在网络栈中的表示形式。每一个具有完整功能的套接字（区别于mini sock）肯定应该具有读写数据缓冲区，即会进行`sk_buff`的管理。
`sock`继承自`sock_common`，这大概是对所有完整套接字和辅助套接字又做了一层抽象，做成了`common`。
```c
struct sock {
    struct sock_common  __sk_common;
    socket_lock_t       sk_lock;
    struct sk_buff_head sk_receive_queue;

    union {
        struct sk_buff  *sk_send_head;
        struct rb_root  tcp_rtx_queue;
    };

    struct sk_buff_head sk_write_queue;

    /* ... */
};
```
- `sk_lock`是锁。
- `sk_receive_queue`和`sk_write_queue`是接收和发送缓冲区。
- `sk_send_head`是等待传输队列的头指针。
- TCP协议已经重要到可以让抽象基类用`union`留出位置，为其开一个专用接口。`tcp_rtx_queue`是TCP协议的重传队列。不得不说，TCP和UDP毕竟是`sock`最大的客户。
#### inet_sock
```c
struct inet_sock {
    /* sk and pinet6 has to be the first two members of inet_sock */
    struct sock     sk;
#if IS_ENABLED(CONFIG_IPV6)
    struct ipv6_pinfo   *pinet6;
#endif
    /* Socket demultiplex comparisons on incoming packets. */
#define inet_daddr      sk.__sk_common.skc_daddr
#define inet_rcv_saddr      sk.__sk_common.skc_rcv_saddr
#define inet_dport      sk.__sk_common.skc_dport
#define inet_num        sk.__sk_common.skc_num

    __be32          inet_saddr; // Sending source
    __s16           uc_ttl; // Unicast TTL
    __u16           cmsg_flags;
    __be16          inet_sport; // Source port
    __u16           inet_id;    // ID counter for DF pkts

    struct ip_options_rcu __rcu *inet_opt;
    int         rx_dst_ifindex;
    __u8            tos;
    __u8            min_ttl;
    __u8            mc_ttl;
    __u8            pmtudisc;
    __u8            recverr:1,
                is_icsk:1,
                freebind:1,
                hdrincl:1,
                mc_loop:1,
                transparent:1,
                mc_all:1,
                nodefrag:1;
    __u8            bind_address_no_port:1,
                defer_connect:1; /* Indicates that fastopen_connect is set
                          * and cookie exists so we defer connect
                          * until first data frame is written
                          */
    __u8            rcv_tos;
    __u8            convert_csum;
    int         uc_index;
    int         mc_index;
    __be32          mc_addr;
    struct ip_mc_socklist __rcu *mc_list;
    struct inet_cork_full   cork; // info to build ip hdr on each ip frag while socket is corked
 */
};
```
inet_sock这是INET域专用的一个socket表示，它是在struct sock的基础上进行的扩展。
在基本socket的属性已具备的基础上，struct inet_sock提供了INET域专有的一些属性，比如TTL，组播列表，IP地址，端口等。



### raw sock相关
`struct raw_sock` 是RAW协议专用的一个socket的表示，它是在`struct inet_sock`基础上的扩展，因为`RAW`协议要处理`ICMP`协议的过滤设置。
```c
struct raw_sock {  
    struct inet_sock   inet;  
    struct icmp_filter filter;  
}; 
```
### udp sock相关
`struct udp_sock` 是UDP协议专用的一个`socket`表示，它是在`struct inet_sock`基础上的扩展，其定义如下
```c
struct udp_sock {  
    struct inet_sock inet;  
    int             pending;  
    unsigned int    corkflag;  
    __u16           encap_type;  
    __u16           len;  
}; 
```

### TCP sock 相关
`struct tcp_sock`并不直接从`struct inet_sock`上扩展，而是从`struct inet_connection_sock`基础上进行扩展，`struct inet_connection_sock`是所有面向连接的socket的表示。

#### inet_connection_sock
该类似乎也是一个抽象类，用于表示“面向连接”的特征。
```c
struct inet_connection_sock {
    /* inet_sock has to be the first member! */
    struct inet_sock      icsk_inet;
    struct request_sock_queue icsk_accept_queue;
    struct inet_bind_bucket   *icsk_bind_hash;
    unsigned long         icsk_timeout;
    struct timer_list     icsk_retransmit_timer;
    struct timer_list     icsk_delack_timer;
    __u32             icsk_rto;
    __u32                     icsk_rto_min;
    __u32                     icsk_delack_max;
    __u32             icsk_pmtu_cookie;
    const struct tcp_congestion_ops *icsk_ca_ops;
    const struct inet_connection_sock_af_ops *icsk_af_ops;
    const struct tcp_ulp_ops  *icsk_ulp_ops;
    void __rcu        *icsk_ulp_data;
    void (*icsk_clean_acked)(struct sock *sk, u32 acked_seq);
    struct hlist_node         icsk_listen_portaddr_node;
    unsigned int          (*icsk_sync_mss)(struct sock *sk, u32 pmtu);
    __u8              icsk_ca_state:5,
                  icsk_ca_initialized:1,
                  icsk_ca_setsockopt:1,
                  icsk_ca_dst_locked:1;
    __u8              icsk_retransmits;
    __u8              icsk_pending;
    __u8              icsk_backoff;
    __u8              icsk_syn_retries;
    __u8              icsk_probes_out;
    __u16             icsk_ext_hdr_len;
    struct {
        __u8          pending;   /* ACK is pending             */
        __u8          quick;     /* Scheduled number of quick acks     */
        __u8          pingpong;  /* The session is interactive         */
        __u8          retry;     /* Number of attempts             */
        __u32         ato;       /* Predicted tick of soft clock       */
        unsigned long     timeout;   /* Currently scheduled timeout        */
        __u32         lrcvtime;  /* timestamp of last received data packet */
        __u16         last_seg_size; /* Size of last incoming segment      */
        __u16         rcv_mss;   /* MSS used for delayed ACK decisions     */
    } icsk_ack;

    ....

}
```

从中我们可以看到，`inet_connection_sock`包含了连接本身所需要的信息，例如RTO、最小RTO、重传计时器、拥塞控制接口成员，以及若干拥塞控制所需要的核心信息，例如拥塞状态等。

#### tcp_sock
TCP套接字，它除了是一个面向连接的套接字外，还保有TCP协议本身需要的信息，是面向连接套接字的实例化类。
```c
struct tcp_sock {
    /* inet_connection_sock has to be the first member of tcp_sock */
    struct inet_connection_sock inet_conn;
    u16 tcp_header_len; /* Bytes of tcp header to send      */
    u16 gso_segs;   /* Max number of segs per GSO packet    */

/*
 *  Header prediction flags
 *  0x5?10 << 16 + snd_wnd in net byte order
 */
    __be32  pred_flags;

/*
 *  RFC793 variables by their proper names. This means you can
 *  read the code and the spec side by side (and laugh ...)
 *  See RFC793 and RFC1122. The RFC writes these in capitals.
 */
    u64 bytes_received; /* RFC4898 tcpEStatsAppHCThruOctetsReceived
                 * sum(delta(rcv_nxt)), or how many bytes
                 * were acked.
                 */
    u32 segs_in;    /* RFC4898 tcpEStatsPerfSegsIn
                 * total number of segments in.
                 */
    u32 data_segs_in;   /* RFC4898 tcpEStatsPerfDataSegsIn
                 * total number of data segments in.
                 */
    u32 rcv_nxt;    /* What we want to receive next     */
    u32 copied_seq; /* Head of yet unread data      */
    u32 rcv_wup;    /* rcv_nxt on last window update sent   */
    u32 snd_nxt;    /* Next sequence we send        */
    u32 segs_out;   /* RFC4898 tcpEStatsPerfSegsOut
                 * The total number of segments sent.
                 */
    u32 data_segs_out;  /* RFC4898 tcpEStatsPerfDataSegsOut
                 * total number of data segments sent.
                 */
    u64 bytes_sent; /* RFC4898 tcpEStatsPerfHCDataOctetsOut
                 * total number of data bytes sent.
                 */
    u64 bytes_acked;    /* RFC4898 tcpEStatsAppHCThruOctetsAcked
                 * sum(delta(snd_una)), or how many bytes
                 * were acked.
                 */
    u32 dsack_dups; /* RFC4898 tcpEStatsStackDSACKDups
                 * total number of DSACK blocks received
                 */
    u32 snd_una;    /* First byte we want an ack for    */
    u32 snd_sml;    /* Last byte of the most recently transmitted small packet */
    u32 rcv_tstamp; /* timestamp of last received ACK (for keepalives) */

    ...

}
```

很多字段是RFC标准中所定义的变量。另外一些变量维护Fast Open功能、RTT估计、TCP选项内容、慢启动参数、拥塞控制具体信息等。

# QA
## sock结构、socket结构和tcp_sock结构三者之间的关系
对于struct socket和struct sock，它们的区别在于：
socket结构体是对应于用户态，是为应用层提供的统一结构，也就是所谓的general BSD socket。
而sock结构体是对应于内核态，是socket在网络层的表示(network layer representation of sockets)。
它们两者是一一对应的，在struct socket中有一个指针指向对应的struct sock。
![](attachments/Pasted%20image%2020231204163358.png)

> 这一幕我们会经常看到，将一个结构放在另一个结构的开始位置，然后扩展一些成员，通过对于指针的强制类型转换，来访问这些成员。

## 为什么不把socket和sock两个数据结构合并成一个呢？
每个socket数据结构都有一个sock数据结构成员，sock是对socket的扩充，两者一一对应，`socket->sk`指向对应的sock，`sock->socket` 指向对应的socket；为什么不把两个数据结构合并成一个呢？

每一个打开的文件、`socket`等等都用一个`file`数据结构代表，这样文件和`socket`就通过`inode->u(union)`中的各个成员来区别：
```c
struct inode {
    .....................
    union {
        struct ext2_inode_info ext2_i;
        struct ext3_inode_info ext3_i;
        struct socket socket_i;
        .....................
    } u;
};
```
因为socket是inode结构中的一部分，即把inode结 构内部的一个union用作socket结构。由于插口操作的特殊性，这个数据结构中需要有大量的结构成分，如果把这些成分全部放到socket 结构中，则inode结构中的这个union就会变得很大，从而inode结构也会变得很大，而对于其他文件系统这个union是不需要这么大的， 所以会造成巨大浪费，系统中使用inode结构的数量要远远超过使用socket的数量，故解决的办法就是把插口分成两部分，把与文件系 统关系密切的放在socket结构中，把与通信关系密切的放在另一个单独结构sock中。


# 参考
```c
# 【Linux】网络专题（四）——核心数据结构sock族类和net_device
https://void-star.icu/archives/962

#  Socket内核数据结构：如何成立特大项目合作部？
https://time.geekbang.org/column/article/105980
```