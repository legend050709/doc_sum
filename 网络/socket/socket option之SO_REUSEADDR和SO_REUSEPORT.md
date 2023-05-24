# 简介
`SO_REUSEPORT`选项在Linux 3.9被引入内核，在这之前也有一个很像的选项`SO_REUSEADDR`。
![](attachments/Pasted%20image%2020230619192218.png)
> 如上，man 7 socket 可查询介绍。

# 背景
TCP/UDP用`五元组`唯一标识一个连接。**任何时候**，两条连接的五元组都不能完全相同，否则当收到一个报文时，协议栈没办法判断它是属于哪个连接的。
```c
五元组
{<protocol>, <src addr>, <src port>, <dest addr>, <dest port>}
```
五元组里，`protocol`在创建socket时确定，`<src addr>`和`<src port>`在`bind()`时确定，`<dest addr>`和`<dest port>`在`connect()`时确定。
当然，`bind()`和`connect()`在一些时候并不需要显式使用，不过这不在本文的讨论范围里。

默认情况下，两个sockets不能绑定相同的源地址和源端口。
***那么，如果对socket设置了`SO_REUSEADDR`和`SO_REUSEPORT`选项，它们什么时候起作用呢？ 答案是`bind()`，也就在确定`<src addr>`和`<src port>`时。***
>不同操作系统内核对待`SO_REUSEADDR`和`SO_REUSEPORT`的行为有少许差异，但它们都源自**BSD**。因此，接下来就以**BSD**的实现为标准进行说明。

# SO_REUSEADDR
假设我现在需要`bind()`将`socketA`绑定到`A:X`，将`socketB`绑定到`B:Y`(不考虑`X=0`或者`Y=0`，因为`0`表示让内核自动分配端口，一定不会冲突)。
如果`X!=Y`，那么无论`A`和`B`的关系如何，两个`bind()`都会成功。但如果`X==Y`，那么结果会是下面这样:
```
SO_REUSEADDR       socketA        socketB       Result
---------------------------------------------------------------------
  ON/OFF       192.168.0.1:21   192.168.0.1:21    Error (EADDRINUSE)
  ON/OFF       192.168.0.1:21      10.0.0.1:21    OK
  ON/OFF          10.0.0.1:21   192.168.0.1:21    OK
   OFF             0.0.0.0:21   192.168.1.0:21    Error (EADDRINUSE)
   OFF         192.168.1.0:21       0.0.0.0:21    Error (EADDRINUSE)
   ON              0.0.0.0:21   192.168.1.0:21    OK
   ON          192.168.1.0:21       0.0.0.0:21    OK
  ON/OFF           0.0.0.0:21       0.0.0.0:21    Error (EADDRINUSE)
```
第一列表示是否设置`SO_REUSEADDR``注`，最后一列表示**后**绑定的socket是否能绑定成功。
这里设置的对象是指**后**绑定的socket(也就是说不关心前一个是否设置)。
## 应用
- 应用一：在**非**`TCP_LISTEN`状态复用本地地址。
**BSD**的实现中`SO_REUSEADDR`可以让**一个使用通配地址(0.0.0.0)，一个使用指定地址(192.168.1.0)的socket同时绑定相同的端口成功**。

在**Linux中**，作为**Server**的TCP Socket一旦绑定到了具体的端口，启动了LISTEN，即使它之前设置过`SO_REUSEADDR`, 也不会生效。这一点Linux比BSD更加严格
    
    ```
    SO_REUSEADDR       socketA        socketB       Result
    ---------------------------------------------------------------------
      ON/OFF      192.168.0.1:21   0.0.0.0:21    Error (EADDRINUSE)
    ```

在**Linux3.9**版本之前,作为**Client**的Socket，`SO_REUSEADDR`选项具有BSD中的`SO_REUSEPORT`的效果。这一点Linux又比BSD更加宽松。
    
    ```
    SO_REUSEADDR      socketA            socketB           Result
    ---------------------------------------------------------------------
      ON        192.168.0.2:55555   192.168.0.2:55555      OK
    ```


- 应用二：在`TIME_WAIT`状态复用本地地址(如支持服务端快速重启)。
在`TCP`中存在一个`TIME_WAIT`状态，它是指主动关闭的一端最后停留的阶段。假设`socketA`绑定到`A:X`，在完成TCP通信后主动使用`close()`,进入`TIME_WAIT`，此时，如果`socketB`也去绑定`A:X`，那么同样会得到`EADDRINUSE`错误，但如果`socketB`设置了`SO_REUSEADDR`，那么就可以绑定成功。

## 其他
SO_REUSEADDR 这个选项对应的linux内核参数是_tcp_tw_reuse_。
![](attachments/Pasted%20image%2020230619193148.png)

# SO_REUSEPORT
如果理解了`SO_REUSEADDR`，那么`SO_REUSEPORT`就很好理解了，它让两个socket可以绑定完全相同的`<IP:Port>`
```
SO_REUSEPORT       socketA        socketB       Result
---------------------------------------------------------------------
    ON         192.168.0.1:21   192.168.0.1:21    OK
```
**提醒一下，以上的结果都是BSD的结果，Linux内核有一些不一样的地方**，具体表现为:

**Linux在3.9版本**才开始支持`SO_REUSEPORT`，在3.9版本之前已经存在 SO_REUSEADDR，通过 SO_REUSEADDR 来实现 SO_REUSEPORT 的特性。

## 背景
运行在Linux系统上网络应用程序，为了利用多核的优势，一般使用以下比较典型的多进程/多线程服务器模型：

1. 单线程listen+accept，多个工作线程接收任务分发，虽CPU的工作负载不再是问题，但会存在：
    - 单线程listener，在处理高速率海量连接时，一样会成为瓶颈
    - CPU缓存行丢失套接字结构(socket structure)现象严重
2. 所有工作线程都accept()在同一个服务器套接字上呢，一样存在问题：
    - 多线程accept从半连接队列取连接锁竞争严重
    - 高负载下，线程之间处理不均衡，有时高达3:1不均衡比例
    - 导致CPU缓存行跳跃(cache line bouncing)
    - 在繁忙CPU上存在较大延迟


上面模型虽然可以做到线程和CPU核绑定，但都会存在：
- 单一listener工作线程在高速的连接接入处理时会成为瓶颈
- 缓存行跳跃
- 很难做到CPU之间的负载均衡
- 随着核数的扩展，性能并没有随着提升

比如HTTP CPS(Connection Per Second)吞吐量并没有随着CPU核数增加呈现线性增长：
![](attachments/Pasted%20image%2020230620104601.png)

## SO_REUSEPORT介绍

SO_REUSEPORT 功能解决了什么问题？我们先看看 2013 年 3.9+ 版本内核提交的这个 Linux 内核功能 [补丁](https://github.com/torvalds/linux/commit/da5e36308d9f7151845018369148201a5d28b46d?branch=da5e36308d9f7151845018369148201a5d28b46d&diff=split) 的注释。
```c
soreuseport: TCP/IPv4 implementation Allow multiple listener sockets to bind to the same port. Motivation for soresuseport would be something like a web server binding to port 80 running with multiple threads, where each thread might have it's own listener socket. This could be done as an alternative to other models: 1) have one listener thread which dispatches completed connections to workers. 2) accept on a single listener socket from multiple threads. In case #1 the listener thread can easily become the bottleneck with high connection turn-over rate. In case #2, the proportion of connections accepted per thread tends to be uneven under high connection load (assuming simple event loop: while (1) { accept(); process() }, wakeup does not promote fairness among the sockets. We have seen the disproportion to be as high as 3:1 ratio between thread accepting most connections and the one accepting the fewest. With so_reusport the distribution is uniform. Signed-off-by: Tom Herbert <therbert@google.com> Signed-off-by: David S. Miller <davem@davemloft.net> master v5.13 … v3.9-rc1 @davem330 Tom Herbert authored and davem330 committed on 24 Jan 2013 1 parent 055dc21 commit da5e36308d9f7151845018369148201a5d28b46d
```

linux man文档中一段文字描述其作用：
```c
The new socket option allows multiple sockets on the same host to bind to the same port, and is intended to improve the performance of multithreaded network server applications running on top of multicore systems.
```

SO_REUSEPORT支持多个进程或者线程绑定到同一端口，提高服务器程序的性能，解决的问题：
- 允许多个套接字 bind()/listen() 同一个TCP/UDP端口
    - 每一个线程拥有自己的服务器套接字
    - 在服务器套接字上没有了锁的竞争
- 内核层面实现负载均衡
- 安全层面，监听同一个端口的套接字只能位于同一个用户下面


## 原理
reuseport 选项主要解决了两个问题：
1. （A 图）单个 listen socket 遇到的性能瓶颈。
2. （B 图）单个 listen socket 多个线程同时 accept，但是多个线程资源分配不均。

![](attachments/Pasted%20image%2020230619194148.png)

在 tcp 多线程场景中，（B 图）服务端如果所有新链接只保存在一个 listen socket 的 `全链接队列` 中，那么多个线程去这个队列里获取（accept）新的链接，势必会出现多个线程对一个公共资源的争抢，争抢过程中，大量资源的损耗。

而（C 图）有多个 listener 共同 bind/listen 相同的 IP/PORT，也就是说每个进程/线程有一个独立的 listener，相当于每个进程/线程独享一个 listener 的全链接队列，不需要多个进程/线程竞争某个公共资源，能充分利用多核，减少竞争的资源消耗，效率自然提高了。

TCP 客户端链接服务端，第一次握手，服务端被动收到第一次握手 SYN 包，内核就通过哈希算法，将客户端的链接分派到内核半链接队列，三次握手成功后，再将这个链接从半链接队列移动到某个 listener 的全链接队列中，提供 accept 获取。


SO_REUSEPORT的核心的实现主要有三点：
- 扩展 socket option，增加 SO_REUSEPORT 选项，用来设置 reuseport。
- 修改 bind 系统调用实现，以便支持可以绑定到相同的 IP 和端口
- 修改处理新建连接的实现，查找 listener 的时候，能够支持在监听相同 IP 和端口的多个 sock 之间均衡选择。

- 三次握手流程
![](attachments/Pasted%20image%2020230619195726.png)
- 查找合适的listener
服务端被动第一次握手，查找合适的 listener，详看源码（Linux 5.0.1）
![](attachments/Pasted%20image%2020230619195817.png)
```c
/* include/net/inet_hashtables.h */
static inline struct sock *__inet_lookup(struct net *net,
                     struct inet_hashinfo *hashinfo,
                     struct sk_buff *skb, int doff,
                     const __be32 saddr, const __be16 sport,
                     const __be32 daddr, const __be16 dport,
                     const int dif, const int sdif,
                     bool *refcounted)
{
    u16 hnum = ntohs(dport);
    struct sock *sk;

    /* skb 包，从 established 哈希表中查找是否已有 established 的包。*/
    sk = __inet_lookup_established(net, hashinfo, saddr, sport,
                       daddr, hnum, dif, sdif);
    *refcounted = true;
    if (sk)
        return sk;
    *refcounted = false;

    /* 上面没找到，那么就找一个合适的 listener，去走三次握手流程。 */
    return __inet_lookup_listener(net, hashinfo, skb, doff, saddr,
                      sport, daddr, hnum, dif, sdif);
}

/* net/ipv4/inet_hashtables.c */
struct sock *__inet_lookup_listener(struct net *net,
                    struct inet_hashinfo *hashinfo,
                    struct sk_buff *skb, int doff,
                    const __be32 saddr, __be16 sport,
                    const __be32 daddr, const unsigned short hnum,
                    const int dif, const int sdif) {
    struct inet_listen_hashbucket *ilb2;
    struct sock *result = NULL;
    unsigned int hash2;

    /* 通过目标 ip/port 哈希值从 hashinfo.lhash2 查找对应的 slot。*/
    hash2 = ipv4_portaddr_hash(net, daddr, hnum);
    ilb2 = inet_lhash2_bucket(hashinfo, hash2);

    /* 再从对应的 slot 上，搜索哈希链上的数据。 */
    result = inet_lhash2_lookup(net, ilb2, skb, doff,
                    saddr, sport, daddr, hnum,
                    dif, sdif);
    ...
    return result;
}

/* called with rcu_read_lock() : No refcount taken on the socket */
static struct sock *inet_lhash2_lookup(struct net *net,
                struct inet_listen_hashbucket *ilb2,
                struct sk_buff *skb, int doff,
                const __be32 saddr, __be16 sport,
                const __be32 daddr, const unsigned short hnum,
                const int dif, const int sdif) {
    bool exact_dif = inet_exact_dif_match(net, skb);
    struct inet_connection_sock *icsk;
    struct sock *sk, *result = NULL;
    int score, hiscore = 0;
    u32 phash = 0;

    /* 遍历哈希链，获取合适的 listener。 */
    inet_lhash2_for_each_icsk_rcu(icsk, &ilb2->head) {
        sk = (struct sock *)icsk;

        score = compute_score(sk, net, hnum, daddr, dif, sdif, exact_dif);
        /* 统计分数，获取最大匹配分数的 socket。*/
        if (score > hiscore) {
            if (sk->sk_reuseport) {
                /* 算出哈希值。 */
                phash = inet_ehashfn(net, daddr, hnum, saddr, sport);
                /* 在数组里，通过哈希获得 sk。 */
                result = reuseport_select_sock(sk, phash, skb, doff);
                if (result)
                    return result;
            }
            result = sk;
            hiscore = score;
        }
    }

    return result;
}
```

## 应用
- 为**多核多线程**Server绑定相同的IP/PORT对，提升新连接的分配性能。
该功能允许多个进程/线程 bind/listen 相同的 IP/PORT，提升了新链接的分配性能（针对accept）。如可以启用多个worker线程，这些worker线程绑定相同的地址和端口。当新接入一条流时，内核会使用流哈希算法选择使用哪个socket。

>注：与`SO_REUSEADDR` 选项类似，使用`SO_REUSEPORT`选项时，同样要求复用者和被复用者同时设置该选项，如果被复用者没有设置，即使复用者设置了该选项，最终绑定还是失败的。

reuseport 也是内核解决 `惊群问题` 的优秀方案。

1. 每个进程可以 bind/listen 相同的 IP/PORT，相当于每个进程拥有独立的 listen socket 的完全队列，避免了共享 listen socket 的资源争抢，提升了并发的吞吐。
    
2. 内核通过哈希算法，将新链接相对均衡地分配到各个开启了 reuseport 属性的进程，所以资源的负载均衡得到解决。

## 使用
SO_REUSEPORT 功能使用，可以通过网络选项进行设置，在 bind 前面设置即可，使用比较简单。
```c
int fd, reuse = 1;
fd = socket(PF_INET, SOCK_STREAM, IPPROTO_IP);
setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, (const void *)&reuse, sizeof(int));
```

## 应用场景

### 性能随CPU个数水平扩展&CPU之间平衡处理
以前通过`fork`形式创建多个子进程，现在有了SO_REUSEPORT，可以不用通过`fork`的形式，让多进程监听同一个端口。通过SO_REUSEPORT，各个进程中`accept socket fd`不一样，有新连接建立时，内核只会唤醒一个进程来`accept`，并且保证唤醒的均衡性。

- 模型简单，维护方便
进程的管理和应用逻辑解耦，进程的管理水平扩展权限下放给程序员/管理员，可以根据实际进行控制进程启动/关闭，增加了灵活性。

### 新特性测试或多个版本共存
可以很方便的测试新特性，同一个程序，不同版本同时运行中，根据运行结果决定新老版本更迭与否。

针对对客户端而言，表面上感受不到其变动，因为这些工作完全在服务器端进行。

### 服务器无缝升级/切换（热更新/热升级）
我们迭代了一版本，需要部署到线上，为之启动一个新的进程后，稍后关闭旧版本进程程序，服务一直在运行中不间断，需要平衡过度。这就像Erlang语言层面所提供的热更新一样。

有一个[hubtime](https://github.com/amscanne/huptime)开源工具，原理为`SIGHUP信号处理器+SO_REUSEPORT+LD_RELOAD`，可以帮助我们轻松做到。

## SO_REUSEPORT已知问题

SO_REUSEPORT根据数据包的四元组{src ip, src port, dst ip, dst port}和当前绑定同一个端口的服务器套接字数量进行数据包分发。若服务器套接字数量产生变化，内核会把本该上一个服务器套接字所处理的客户端连接所发送的数据包（比如三次握手期间的半连接，以及已经完成握手但在队列中排队的连接）分发到其它的服务器套接字上面，可能会导致客户端请求失败，一般可以使用：
- 使用固定的服务器套接字数量，不要在负载繁忙期间轻易变化
- 允许多个服务器套接字共享TCP请求表(Tcp request table)
- 不使用四元组作为Hash值进行选择本地套接字处理，挑选隶属于同一个CPU的套接字

与RFS/RPS/XPS-mq协作，可以获得进一步的性能：
- 服务器线程绑定到CPUs
- RPS/RSS分发TCP SYN包到对应CPU核上
- TCP连接被已绑定到CPU上的线程accept()
- XPS-mq(Transmit Packet Steering for multiqueue)，传输队列和CPU绑定，发送数据
- RFS/RPS保证同一个连接后续数据包都会被分发到同一个CPU上
- 网卡接收队列已经绑定到CPU，则RFS/RPS则无须设置
- 需要注意硬件支持与否
> 目的：数据包的软硬中断、接收、处理等在一个CPU核上，并行化处理，尽可能做到资源利用最大化。

## 范例
nginx 开启 reuseport 功能后，性能有立竿见影的提升，我们结合 tcp 协议分析 nginx 的 reuseport 功能。

## linux中reuseport的演进
### Linux < 3.9
不支持 SO_REUSEPORT, 支持 SO_REUSEADDR。内核socket使用`skc_reuse`字段表示是否设置了`SO_REUSEADDR`。
```
 struct sock_common {
 	/* omitted */
    unsigned char		skc_reuse;
    /* omitted */
}

int sock_setsockopt(struct socket *sock, int level, int optname,...
{
    ......
    case SO_REUSEADDR:
 	sk->sk_reuse = (valbool ? SK_CAN_REUSE : SK_NO_REUSE);
 	break;
}
```
`inet_bind_bucket`表示一个绑定的端口。
```
struct inet_bind_bucket {
    /* omitted */
	unsigned short		port;
	signed short		fastreuse;
	int			num_owners;
	struct hlist_node	node;
	struct hlist_head	owners;
};
```
上面结构中的`fastreuse`表示该端口是否支持共享，所有共享该端口的socket挂到`owner`成员上。在用户使用`bind()`时，内核使用**TCP**:`inet_csk_get_port()`,**UDP**:`udp_v4_get_port()`来绑定端口。
```
/* inet_connection_Sock.c: inet_csk_get_port() */
tb_found:
	if (!hlist_empty(&tb->owners)) {
        ......
		if (tb->fastreuse > 0 &&
		    sk->sk_reuse && sk->sk_state != TCP_LISTEN &&
		    smallest_size == -1) {
			goto success;
```
所以，当该端口支持共享，且socket也设置了`SO_REUSEADDR`并且不为`LISTEN`状态时，此次`bind()`可以成功。

### 3.9 =< Linux < 4.5
`3.9`版本内核增加了对`SO_REUSEPORT`的支持，`listener`可以绑定到相同的`<IP:Port>`了。这个时候，当Server收到Client发送的SYN报文时，会选择其中一个socket进行响应.

具体到实现，`3.9`版本扩展了`sock_common`，将原来记录`skc_reuse`进行了拆分.
```
struct sock_common {
 	unsigned short		skc_family;
 	volatile unsigned char	skc_state;
-	unsigned char		skc_reuse;
+	unsigned char		skc_reuse:4;
+	unsigned char		skc_reuseport:4;


@@ int sock_setsockopt(struct socket *sock, int level, int optname,
 	case SO_REUSEADDR:
 		sk->sk_reuse = (valbool ? SK_CAN_REUSE : SK_NO_REUSE);
 		break;
+	case SO_REUSEPORT:
+		sk->sk_reuseport = valbool;
+		break;

```
然后对`inet_bind_bucket`也相应进行了扩展
```
struct inet_bind_bucket {
 	/* omitted */
 	unsigned short		port;
-	signed short		fastreuse;
+	signed char		fastreuse;
+	signed char		fastreuseport;
+	kuid_t			fastuid;
```
当Client的SYN报文到达时，Server会首先根据本地端口(SYN报文的`<dport>`)计算出一条hash冲突链，然后遍历该链表上的所有Socket，根据四元组匹配程度进行打分;如果使能了reuseport，那么可能有多个Socket都将拿到最高分，此时内核将随机选择一个进行后续处理。
```
/* inet_hashtables.c  */
struct sock *__inet_lookup_listener(struct......)
{
	struct sock *sk, *result;
	unsigned int hash = inet_lhashfn(net, hnum);
	struct inet_listen_hashbucket *ilb = &hashinfo->listening_hash[hash]; // 根据本地端口找到hash冲突链
    /* code omitted */
	result = NULL;
	hiscore = 0;
	sk_nulls_for_each_rcu(sk, node, &ilb->head) {
		score = compute_score(sk, net, hnum, daddr, dif); // 根据匹配程度进行打分
		if (score > hiscore) {
			result = sk;
			hiscore = score;
			reuseport = sk->sk_reuseport;
			if (reuseport) {
				phash = inet_ehashfn(net, daddr, hnum,
						     saddr, sport);
				matches = 1;                             // 如果是reuseport 则累计多少个socket满足
			}
		} else if (score == hiscore && reuseport) {
			matches++;
			if (reciprocal_scale(phash, matches) == 0)
				result = sk;
			phash = next_pseudo_random32(phash);
		}
	}
	/*
	 * if the nulls value we got at the end of this lookup is
	 * not the expected one, we must restart lookup.
	 * We probably met an item that was moved to another chain.
	 */
	return result;
}
```
举个栗子，假设内核有4条listening socket的hash冲突链，然后用户建立了4个Server：A、B、C、D，监听的地址和端口如下图所示，A和B使能了`SO_REUSEPORT`。冲突链是以端口为Key的，因此A、B、D会挂到同一条冲突链上。如果此时收到对端一个SYN报文<192.168.10.1, 21>,那么内核会遍历`listening_hash[0]`，为上面的7个socket进行打分，而由于B监听的是精确的地址，所以B的得分会比A高，内核最终选择出一个SocketB进行后续处理。
![](attachments/Pasted%20image%2020230619181527.png)

### 4.5 < Linux
从上面的例子可以看出，当收到SYN报文时，内核一定会遍历一条完整hash冲突链，为每一个socket进行打分，这稍微有些多余。因此，在4.5版本中，内核引入了`reuseport groups`，它将绑定到同一个IP和Port，并且设置了`SO_REUSEPORT`选项的socket组织到一个`group`内部。
![](attachments/Pasted%20image%2020230619181619.png)

这个特性在4.5版本只支持UDP,而在4.6版本开始支持TCP([patch](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/net/ipv4/inet_hashtables.c?id=c125e80b88687b25b321795457309eaaee4bf270))。这样在查找listen socket时，内核将不用再遍历整个冲突链，而是在找到一个合格的socket时，如果它设置了`SO_REUSEPORT`,就直接找到它所属的`reuseport group`,从中选择一个进行后续处理.

# 参考
```c
https://www.cnblogs.com/charlieroro/p/14096252.html
http://www.blogjava.net/yongboy/archive/2015/02/12/422893.html
https://switch-router.gitee.io/blog/reuseport/
https://wenfh2020.com/2021/10/12/thundering-herd-tcp-reuseport/
```