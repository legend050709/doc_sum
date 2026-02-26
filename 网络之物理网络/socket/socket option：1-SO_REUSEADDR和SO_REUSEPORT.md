```table-of-contents
```
# 简介
`SO_REUSEPORT`选项在Linux 3.9被引入内核，在这之前也有一个很像的选项`SO_REUSEADDR`。
![](attachments/Pasted%20image%2020230619192218.png)
> 如上，man 7 socket 可查询介绍。

# 背景
## 连接的标识
TCP/UDP用`五元组`唯一标识一个连接。**任何时候**，两条连接的五元组都不能完全相同，否则当收到一个报文时，协议栈没办法判断它是属于哪个连接的。
```c
五元组
{<protocol>, <src addr>, <src port>, <dest addr>, <dest port>}
```

## socket 和 五元组成员的绑定
五元组里，`protocol`在创建socket时确定；
`<src addr>`和`<src port>`在`bind()`时确定；
`<dest addr>`和`<dest port>`在`connect()`时确定。
当然，`bind()`和`connect()`在一些时候并不需要显式使用，不过这不在本文的讨论范围里。
> 注：其中UDP是无连接的，UDP socket可以在未与目的端口连接的情况下使用。但UDP也可以在某些情况下先与目的地址和端口建立连接后使用。
> 在使用无连接UDP发送数据的情况下，如果没有显式地调用bind()，操作系统会在**第一次发送数据时**自动将UDP socket与本机的地址和某个端口绑定（否则的话程序无法接受任何远程主机回复的数据）。
> 同样的，一个没有绑定地址的TCP socket也会在**connect建立连接时**被自动绑定一个本机地址和端口。

如果我们手动绑定一个端口，我们可以将socket绑定至端口0，绑定至端口0的意思是让系统自己决定使用哪个端口（一般是从一组操作系统特定的提前决定的端口数范围中），所以也就是任何端口的意思。
同样的，我们也可以使用一个通配符来让系统决定绑定哪个源地址（ipv4通配符为0.0.0.0，ipv6通配符为::）。而与端口不同的是，一个socket可以被绑定到主机上所有接口所对应的地址中的任意一个。基于连接在本socket的目的地址和路由表中对应的信息，操作系统将会选择合适的地址来绑定这个socket，并用这个地址来取代之前的通配符IP地址。

## socket绑定的限制
==默认情况下，两个sockets不能绑定相同的源地址和源端口组合上==。比如说我们将socketA绑定在A:X地址，将socketB绑定在B:Y地址，其中A和B是IP地址，X和Y是端口。那么在`A==B`的情况下X!=Y必须满足，在`X==Y`的情况下A!=B必须满足。

需要注意的是，如果某一个socket被绑定在通配符IP地址下，那么事实上本机所有IP都会被系统认为与其绑定了。例如一个socket绑定了0.0.0.0:21，在这种情况下，任何其他socket不论选择哪一个具体的IP地址，其都不能再绑定在21端口下。因为==通配符IP0.0.0.0与所有本地IP都冲突==。

## tcp socket关闭的延迟时间
每一个TCP socket都有其相应的发送缓冲区（buffer）。当成功调用其send()方法的时候，实际上我们所要求发送的数据并不一定被立即发送出去，而是被添加到了发送缓冲区中。


对于TCP socket来说，在将数据添加到发送缓冲区之后，可能需要等待相对较长的时间之后数据才会被真正发送出去。因此，当我们关闭了一个TCP socket之后，其发送缓冲区中可能实际上还仍然有等待发送的数据。但此时因为send()返回了成功，我们的代码认为数据已经实际上被成功发送了。

如果TCP socket在我们调用close()之后直接关闭，那么所有这些数据都将会丢失，而我们的代码根本不会知道。但是，TCP是一个可靠的传输层协议，直接丢弃这些待传输的数据显然是不可取的。实际上，如果在socket的发送缓冲区中还有待发送数据的情况下调用了其close()方法，socket将会持续尝试发送缓冲区的数据直到所有数据都被成功发送或者直到超时，超时被触发的情况下socket将会被强制关闭。

==调用 `close()` 后等待 buffer 内数据全部写出去的时间叫做 Linger Time==。Linger Time 结束后开始正常的 TCP 挥手过程。
操作系统的kernel在强制关闭一个socket之前的最长等待时间被称为延迟时间（Linger Time）。在大部分系统中延迟时间都已经被全局设置好了，并且相对较长（大部分系统将其设置为2分钟）。
我们也可以在初始化一个socket的时候使用SO_LINGER选项来特定地设置每一个socket的延迟时间。我们甚至可以完全关闭延迟等待。但是需要注意的是，将延迟时间设置为0（完全关闭延迟等待）并不是一个好的编程实践。因为优雅地关闭TCP socket是一个比较复杂的过程，过程中包括与远程主机交换数个数据包（包括在丢包的情况下的丢失重传），而**这个数据包交换的过程所需要的时间也包括在延迟时间中**。
如果我们停用延迟等待，socket不止会在关闭的时候直接丢弃所有待发送的数据，而且总是会被强制关闭（由于TCP是面向连接的协议，不与远端端口交换关闭数据包将会导致远端端口处于长时间的等待状态）。所以通常我们并不推荐在实际编程中这样做。
并且实际上，如果我们禁用了延迟等待，而我们的程序没有显式地关闭socket就退出了，BSD（可能包括其他系统）会忽略我们的设置进行延迟等待。例如，如果我们的程序调用了exit()方法，或者其进程被使用某个信号终止了（包括进程因为非法内存访问之类的情况而崩溃）。所以我们无法百分之百保证一个socket在所有情况下忽略延迟等待时间而终止。

## SO_REUSEADDR 和  SO_REUSEPORT 生效时机
***那么，如果对socket设置了`SO_REUSEADDR`和`SO_REUSEPORT`选项，它们什么时候起作用呢？ 答案是`bind()`，也就在确定`<src addr>`和`<src port>`时。***
>不同操作系统内核对待`SO_REUSEADDR`和`SO_REUSEPORT`的行为有少许差异，但它们都源自**BSD**。因此，接下来就以**BSD**的实现为标准进行说明。

# SO_REUSEADDR
假设我现在需要`bind()`将`socketA`绑定到`A:X`，将`socketB`绑定到`B:Y`(不考虑`X=0`或者`Y=0`，因为`0`表示让内核自动分配端口，一定不会冲突)。
如果`X!=Y`，那么无论`A`和`B`的关系如何，两个`bind()`都会成功。
但如果`X==Y`，那么结果会是下面这样:
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
这里设置的对象是指**后**绑定的socket(**也就是说不关心前一个是否设置**)。

## 作用
### 作用一
如果在一个socket绑定到某一地址和端口之前设置了其SO_REUSEADDR的属性，那么除非本socket尝试与另一个socket绑定到完全相同的源地址和源端口组合会导致冲突，否则的话这个socket就可以成功的绑定这个地址端口对。

- **如果不用SO_REUSEADDR的话**，如果我们将socketA绑定到0.0.0.0:21，那么任何将本机其他socket绑定到端口21的举动（如绑定到192.168.1.1:21）都会导致EADDRINUSE错误。因为0.0.0.0是一个通配符IP地址，意味着任意一个IP地址，所以任何其他本机上的IP地址都被系统认为已被占用。

- **如果设置了SO_REUSEADDR选项**，因为0.0.0.0:21和192.168.1.1:21并不是完全相同的地址端口对（其中一个是通配符IP地址，另一个是一个本机的具体IP地址），所以这样的绑定是可以成功的。

> 注：需要注意的是，无论socketA和socketB初始化的顺序如何，只要设置了SO_REUSEADDR，绑定都会成功；而只要没有设置SO_REUSEADDR，绑定都不会成功。

### 作用二

如果SO_REUSEADDR选项没有被设置，处于TIME_WAIT阶段的socket任然被认为是绑定在原来那个地址和端口上的。直到该socket被完全关闭之前（结束TIME_WAIT阶段），任何其他企图将一个新socket绑定该该地址端口对的操作都无法成功。

这一等待的过程可能和延迟等待的时间一样长。所以我们并不能马上将一个新的socket绑定到一个刚刚被关闭的socket对应的地址端口对上。在大多数情况下这种操作都会失败。

如果我们在新的socket上设置了SO_REUSEADDR选项，如果此时有另一个socket绑定在当前的地址端口对且处于TIME_WAIT阶段，那么这个已存在的绑定关系将会被忽略。
> 事实上处于TIME_WAIT阶段的socket已经是半关闭的状态，将一个新的socket绑定在这个地址端口对上不会有任何问题。这样的话原来绑定在这个端口上的socket一般不会对新的socket产生影响。但需要注意的是，在某些时候，将一个新的socket绑定在一个处于TIME_WAIT阶段但仍在工作的socket所对应的地址端口对会产生一些我们并不想要的，无法预料的负面影响。但这个问题超过了本文的讨论范围。而且幸运的是这些负面影响在实践中很少见到。

注：以上所有内容只要我们对**新的socket设置了SO_REUSEADDR**就成立。至于原有的已经绑定在当前地址端口对上的，处于或不处于TIME_WAIT阶段的socket是否设置了SO_REUSEADDR并无影响。

决定bind操作是否成功的代码仅仅会**检查新的被传递到bind()方法的socket的SO_REUSEADDR选项**。其他涉及到的socket的SO_REUSEADDR选项并不会被检查。

## 应用
### 应用一：在非`TCP_LISTEN`状态复用本地地址

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


### 应用二：在`TIME_WAIT`状态复用本地地址(如支持服务端快速重启)

在`TCP`中存在一个`TIME_WAIT`状态，它是指主动关闭的一端最后停留的阶段。假设`socketA`绑定到`A:X`，在完成TCP通信后主动使用`close()`,进入`TIME_WAIT`，此时，如果`socketB`也去绑定`A:X`，那么同样会得到`EADDRINUSE`错误，但如果`socketB`设置了`SO_REUSEADDR`，那么就可以绑定成功。


## 其他
SO_REUSEADDR 这个选项对应的linux内核参数是 `tcp_tw_reuse`。

![](attachments/Pasted%20image%2020230619193148.png)

### tcp_tw_reuse 和 SO_REUSEADDR
- tcp_tw_reuse 是内核选项，主要用在**连接的发起方**。TIME_WAIT 状态的连接创建时间超过 1 秒后，新的连接才可以被复用，注意，这里是连接的发起方；
- SO_REUSEADDR 是**用户态的选项**，SO_REUSEADDR 选项用来告诉操作系统内核，如果端口已被占用，**但是 TCP 连接状态位于 TIME_WAIT ，可以重用端口**。如果端口忙，而 TCP 处于其他状态，重用端口时依旧得到“Address already in use”的错误信息。注意，这里一般都是连接的服务方。

关于tcp_tw_reuse和SO_REUSEADDR的区别，可以概括为：tcp_tw_reuse是为了缩短time_wait的时间，避免出现大量的time_wait链接而占用系统资源；SO_REUSEADDR是为了解决time_wait状态带来的端口占用问题，以及支持同一个port对应多个ip，解决的是bind时的问题

###  UDP使用SO_REUSEADDR
对于UDP来说，SO_REUSEADDR允许完全重复的捆绑：即当一个IP地址和端口已经绑定到某个套接字上时，同样的IP地址和端口还可以捆绑到另一个套接字上。TCP则不行。

本特性用于多播（组播）时， 允许在同一个主机上同时运行同一个应用程序的多个副本。当一个UDP数据报需要由这些重复捆绑套接字中的一个接收时，所用规则为：如果该数据报的目的地址是一个**广播地址或多播地址**，那就给每个匹配的套接字发送一个该**数据报的副本**；如果该数据报的目的地址是一个单播地址，那么它只发送给单个套接字。

# SO_REUSEPORT

如果理解了`SO_REUSEADDR`，那么`SO_REUSEPORT`就很好理解了，它让两个socket可以绑定完全相同的`<IP:Port>`
```
SO_REUSEPORT       socketA        socketB       Result
---------------------------------------------------------------------
    ON         192.168.0.1:21   192.168.0.1:21    OK
```

**提醒一下，以上的结果都是BSD的结果，Linux内核有一些不一样的地方**，具体表现为:

==**Linux在3.9版本**才开始支持`SO_REUSEPORT`，在3.9版本之前已经存在 SO_REUSEADDR，通过 SO_REUSEADDR 来实现 SO_REUSEPORT 的特性==。

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

==SO_REUSEPORT支持多个进程或者线程绑定到同一端口，提高服务器程序的性能，解决的问题==：
- 允许多个套接字 `bind()/listen()` 同一个`TCP/UDP`端口
    - 每一个线程拥有自己的服务器套接字
    - 在服务器套接字上没有了锁的竞争
- 内核层面实现负载均衡
- 安全层面，监听同一个端口的套接字只能位于同一个用户下面


## SO_REUSEPORT 和 SO_REUSEADDR 区别与联系

==SO_REUSEPORT允许我们将任意数目的socket绑定到完全相同的源地址端口对上，只要所有之前绑定的socket都设置了SO_REUSEPORT选项==。

### 注意

**如果第一个绑定在该地址端口对上的socket没有设置SO_REUSEPORT，无论之后的socket是否设置SO_REUSEPORT，其都无法绑定在与这个地址端口完全相同的地址上。除非第一个绑定在这个地址端口对上的socket释放了这个绑定关系**。

**与SO_REUSEADDR不同的是 ，处理SO_REUSEPORT的代码不仅会检查当前尝试绑定的socket的SO_REUSEPORT，而且也会检查之前已绑定了当前尝试绑定的地址端口对的socket的SO_REUSEPORT选项**。

如果当前socket已经处于TIME_WAIT阶段，而这个设置了SO_REUSEPORT选项的新socket尝试绑定到当前地址，这个绑定操作也会失败。为了能够将新的socket绑定到一个当前处于TIME_WAIT阶段的socket对应的地址端口对上，我们要么需要在绑定之前设置这个新socket的SO_REUSEADDR选项，要么需要在绑定之前给两个socket都设置SO_REUSEPORT选项。
==当然，同时给socket设置SO_REUSEADDR和SO_REUSEPORT选项是也是可以的==。

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

## SO_REUSEPORT 的负载均衡算法

使用`(remote_ip, remote_port, local_ip, local_port)`四元组来进行哈希，因此可以保证同一个client的包可以路由到同一个进程。
但是，==当一个listen的进程加进来或者`terminate`的时候，由于没有实现`一致性哈希`，结果可能导致有些请求由于路由到另外一个进程上，client-server的三次握手过程可能会被重置==。


## 限制

（1）第一个进程必须`enable`了`SO_REUSEPORT`之后，后续的进程才可以通过`enable`这个选项将`socket`绑定到同一个端口上。

（2）绑定到同一个端口的多个进程的`effective user id`必须一致。
> 上述规定是为了避免`hijacking`劫持：恶意用户通过监听相同的端口来获取用户信息。

## 相关问题
### Connect()返回EADDRINUSE？

有些时候bind()操作会返回EADDRINUSE错误。但奇怪的是，在我们调用connect()操作时，也有可能得到EADDRINUSE错误。这是为什么呢？

正如本文之前所说，一个连接关系是由一个五元组确定的。对于任意的连接关系而言，这个五元组必须是唯一的。否则的话，系统将无法分辨两个连接。而现在当我们采用了地址复用之后，我们可以将两个采用相同协议的socket绑定到同一地址端口对上。这意味着对这两个socket而言，五元组里的{sip, sport, proto }已经相同了。在这种情况下，如果我们尝试将它们都连接到同一个远程地址端口上，这两个连接关系的五元组将完全相同。也就是说，产生了两个完全相同的连接。在TCP协议中这是不被允许的（UDP是无连接的）。如果这两个完全相同的连接种的某一个接收到了数据，系统将无法分辨这个数据到底属于哪个连接。所以在这种情况下，至少这两个socket所尝试连接的远程主机的地址和端口不能相同。只有如此，系统才能继续区分这两个连接关系。

所以当我们将两个采用相同协议的socket绑定到同一个本地地址端口对上后，如果我们还尝试让它们和同一个目的地址端口对建立连接，第二个尝试调用connect()方法的socket将会报EADDRINUSE的错误，这说明一个拥有完全相同的五元组的socket已经存在了。

### tcp socket和udp socket使用SO_REUSEPORT的区别

对于设置了SO_REUSEPORT选项的socket，Linux kernel还会执行一些别的系统所没有的特别的操作：
==(1) 对于绑定于同一地址端口组合上的UDP socket，kernel尝试在它们之间平均分配收到的数据包==；
==(2) 对于绑定于同一地址端口组合上的TCP监听socket，kernel尝试在它们之间平均分配收到的连接请求（调用accept()方法所得到的请求）==。

比如一个简单的服务器进程的几个不同实例可以方便地使用`SO_REUSEPORT`来实现一个简单的负载均衡，而且这个负载均衡有kernel负责， 对程序来说完全免费！


### 单机多进程使用reuseport导致进程间“流量劫持”问题
#### 问题
在 Linux 网络编程中，`SO_REUSEPORT` 确实引入了这种“内核级负载均衡”的机制，同时也带来了你所担心的进程间“流量劫持”或“竞争”的隐患。

简单来说：如果另一个具有足够权限的进程也使用了 `SO_REUSEPORT` 监听同一个端口，内核会将一部分新连接分发给它，而你的进程对此完全无感。

##### 为什么你的程序感知不到？

`SO_REUSEPORT` 的设计初衷是为了提高多核性能，让内核在多个监听相同套接字的线程之间进行 哈希分发。

- **内核视角**：内核维护了一个监听相同端口的套接字列表。每当有新的 `SYN` 包到达时，内核会根据源 IP 和源端口计算哈希值，然后决定将连接放入哪个套接字的 `accept` 队列。
    
- **进程视角**：每个进程/线程只管从自己的 `epoll` 实例中读取就绪事件。如果连接被分发给了进程 B，进程 A 的 `epoll_wait` 根本不会返回，它会认为此时没有新连接。

##### 安全性限制：谁能“抢”你的流量？

Linux 为了防止恶意进程劫持流量，对 `SO_REUSEPORT` 设定了权限门槛：

- **UID 必须一致**：默认情况下，只有运行在**同一个有效用户 ID (EUID)** 下的进程，才能对同一个端口开启 `SO_REUSEPORT`。
    
- **示例**：如果你的程序以 `www-data` 用户运行，黑客尝试用 `nobody` 用户启动一个监听相同端口的程序，内核会直接返回 `EADDRINUSE` 错误，阻止其加入负载均衡组。
    
- **特权用户例外**：`root` 用户可以“劫持”任何用户的端口。

#### 范例

```c
# cat server.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <pthread.h>
#include <sys/epoll.h>
#include <errno.h>

#define PORT 8888
#define MAX_EVENTS 10
#define THREAD_COUNT 3
#define PROCESS_COUNT 2

void *worker_thread(void *arg) {
    int process_id = *(int *)arg;
    pthread_t tid = pthread_self();

    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;

    // 核心：设置 SO_REUSEPORT
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));

    struct sockaddr_in address;
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);

    if (bind(listen_fd, (struct sockaddr *)&address, sizeof(address)) < 0) {
        perror("bind failed");
        return NULL;
    }

    listen(listen_fd, 128);

    int epoll_fd = epoll_create1(0);
    struct epoll_event ev, events[MAX_EVENTS];
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

    printf("[Process %d][Thread %lx] Listening on port %d\n", process_id, (unsigned long)tid, PORT);

    while (1) {
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_fd) {
                struct sockaddr_in client_addr;
                socklen_t addrlen = sizeof(client_addr);
                int conn_sock = accept(listen_fd, (struct sockaddr *)&client_addr, &addrlen);

                if (conn_sock > 0) {
                    char msg[128];
                    sprintf(msg, "Response from Process %d, Thread %lx\n", process_id, (unsigned long)tid);
                    send(conn_sock, msg, strlen(msg), 0);
                    close(conn_sock);
                }
            }
        }
    }
    return NULL;
}

int main() {
    for (int i = 0; i < PROCESS_COUNT; i++) {
        pid_t pid = fork();
        if (pid == 0) { // 子进程
            pthread_t threads[THREAD_COUNT];
            int process_id = i + 1;
            for (int j = 0; j < THREAD_COUNT; j++) {
                pthread_create(&threads[j], NULL, worker_thread, &process_id);
            }
            for (int j = 0; j < THREAD_COUNT; j++) {
                pthread_join(threads[j], NULL);
            }
            exit(0);
        }
    }
    // 父进程等待
    while(wait(NULL) > 0);
    return 0;
}
```

（1）进程1监听相同的port：
```bash
# gcc server.c -o reuseport_server -lpthread -std=gnu99
# ./reuseport_server
[Process 1][Thread 7fa5bd744700] Listening on port 8888
[Process 1][Thread 7fa5bcd43700] Listening on port 8888
[Process 1][Thread 7fa5bc342700] Listening on port 8888
[Process 2][Thread 7fa5bcd43700] Listening on port 8888
[Process 2][Thread 7fa5bc342700] Listening on port 8888
[Process 2][Thread 7fa5bd744700] Listening on port 8888

```

（2）进程2监听相同的port：
```bash
# socat TCP4-LISTEN:8888,reuseport,fork EXEC:"echo I AM AN INTRUDER"
```

（3）查看
```bash
# ss -lnept | grep -e 8888 -i -e local
State      Recv-Q Send-Q Local Address:Port               Peer Address:Port
LISTEN     0      5            *:8888                     *:*                   users:(("socat",pid=39375,fd=5)) ino:3696446054 sk:16 <->
LISTEN     0      128          *:8888                     *:*                   users:(("reuseport_serve",pid=13976,fd=7)) ino:3696280256 sk:3 <->
LISTEN     0      128          *:8888                     *:*                   users:(("reuseport_serve",pid=13976,fd=5)) ino:3696422975 sk:4 <->
LISTEN     0      128          *:8888                     *:*                   users:(("reuseport_serve",pid=13976,fd=3)) ino:3696372190 sk:5 <->
LISTEN     0      128          *:8888                     *:*                   users:(("reuseport_serve",pid=13975,fd=7)) ino:3696327382 sk:6 <->
LISTEN     0      128          *:8888                     *:*                   users:(("reuseport_serve",pid=13975,fd=5)) ino:3696412799 sk:7 <->
LISTEN     0      128          *:8888                     *:*                   users:(("reuseport_serve",pid=13975,fd=3)) ino:3696402722 sk:8 <->


# ll /proc/13975/fd
total 0
lrwx------ 1 root root 64 Dec 22 11:16 0 -> /dev/pts/0
lrwx------ 1 root root 64 Dec 22 11:16 1 -> /dev/pts/0
lrwx------ 1 root root 64 Dec 22 11:15 2 -> /dev/pts/0
lrwx------ 1 root root 64 Dec 22 11:16 3 -> socket:[3696402722]
lrwx------ 1 root root 64 Dec 22 11:16 4 -> anon_inode:[eventpoll]
lrwx------ 1 root root 64 Dec 22 11:16 5 -> socket:[3696412799]
lrwx------ 1 root root 64 Dec 22 11:16 6 -> anon_inode:[eventpoll]
lrwx------ 1 root root 64 Dec 22 11:16 7 -> socket:[3696327382]
lrwx------ 1 root root 64 Dec 22 11:16 8 -> anon_inode:[eventpoll]

# ll /proc/39375/fd
total 0
lrwx------ 1 root root 64 Dec 22 11:28 0 -> /dev/pts/2
lrwx------ 1 root root 64 Dec 22 11:28 1 -> /dev/pts/2
lrwx------ 1 root root 64 Dec 22 11:28 2 -> /dev/pts/2
lrwx------ 1 root root 64 Dec 22 11:28 3 -> socket:[3696446052]
lrwx------ 1 root root 64 Dec 22 11:28 4 -> socket:[3696446053]
lrwx------ 1 root root 64 Dec 22 11:28 5 -> socket:[3696446054]
```

（4）client：
```python3
# cat client.py
import socket
import concurrent.futures

def connect_once(i):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect(('127.0.0.1', 8888))
        response = s.recv(1024).decode().strip()
        s.close()
        return response
    except Exception as e:
        return str(e)

def run_test():
    print("Starting load balance test (50 connections)...")
    results = {}

    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        futures = [executor.submit(connect_once, i) for i in range(50)]
        for future in concurrent.futures.as_completed(futures):
            res = future.result()
            results[res] = results.get(res, 0) + 1

    print("\n--- Statistics ---")
    for worker, count in sorted(results.items()):
        print(f"{worker}: {count} hits")

if __name__ == "__main__":
    run_test()
```

```bash
# python3 ./client.py
Starting load balance test (50 connections)...

--- Statistics ---
I AM AN INTRUDER: 11 hits
Response from Process 1, Thread 7fa5bc342700: 9 hits
Response from Process 1, Thread 7fa5bcd43700: 7 hits
Response from Process 1, Thread 7fa5bd744700: 11 hits
Response from Process 2, Thread 7fa5bc342700: 2 hits
Response from Process 2, Thread 7fa5bcd43700: 6 hits
Response from Process 2, Thread 7fa5bd744700: 4 hits
```

#### 如何确保流量不被“劫持”
##### 检查端口占用情况
在程序启动或运行期间，你可以通过读取 `/proc/net/tcp` 或调用工具`ss -lnetp`指令来确认监听者的身份。

##### 用 eBPF
你可以编写一个简单的 eBPF 程序（挂载到 `inet_bind` 或相关内核点），监控哪些进程正在尝试对特定端口设置 `SO_REUSEPORT`。如果发现非法 PID 或 UID，直接拦截。

##### 利用 `SO_ATTACH_REUSEPORT_CBPF`

Linux 允许你为 `REUSEPORT` 组绑定一个 BPF 过滤器。通过这个机制，你可以自定义负载均衡逻辑。虽然它主要用于优化分发，但也可以用来确保只有符合特定条件的套接字能接收流量。

##### 架构设计的权衡

如果你非常担心“不可控”的进程竞争，可以考虑以下替代方案：

1. **单 Listen 模式**：只由一个主线程/进程 `listen` 端口。接收到新连接（`accept`）后，通过 `Unix Domain Socket` 使用 `sendmsg` 将文件描述符 (fd) 传递给工作线程/进程。这种方式流量完全由你控制，但会有跨进程传递 fd 的开销。
    
2. **独占监听**：不使用 `SO_REUSEPORT`。如果你只有一个进程，可以通过 `epoll` 的 `EPOLLEXCLUSIVE` 标志（Linux 4.5+）来解决多线程同时 `wait` 产生的“惊群效应”，同时保证了该端口在系统层面的排他性。


## 应用
### 为多核多线程Server绑定相同的IP/PORT对，提升新连接的分配性能。
该功能允许多个进程/线程 bind/listen 相同的 IP/PORT，提升了新链接的分配性能（针对accept）。如可以启用多个worker线程，这些worker线程绑定相同的地址和端口。当新接入一条流时，内核会使用流哈希算法选择使用哪个socket。

>注：与`SO_REUSEADDR` 选项类似，使用`SO_REUSEPORT`选项时，同样要求复用者和被复用者同时设置该选项，如果被复用者没有设置，即使复用者设置了该选项，最终绑定还是失败的。

==reuseport 也是内核解决 `惊群问题` 的优秀方案==。

## 多进程中每个进程可以 bind/listen 相同的 IP/PORT

相当于每个进程拥有独立的 listen socket 的完全队列，避免了共享 listen socket 的资源争抢，提升了并发的吞吐。
内核通过哈希算法，将新链接相对均衡地分配到各个开启了 reuseport 属性的进程，所以资源的负载均衡得到解决。

## 使用
SO_REUSEPORT 功能使用，可以通过网络选项进行设置，**在 bind 前面设置即可**，使用比较简单。
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
**不支持 SO_REUSEPORT, 支持 SO_REUSEADDR**。
内核socket使用`skc_reuse`字段表示是否设置了`SO_REUSEADDR`。

```c
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
```c
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
```c
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
**`3.9`版本内核增加了对`SO_REUSEPORT`的支持，`listener`可以绑定到相同的`<IP:Port>`了**。
这个时候，当Server收到Client发送的SYN报文时，会选择其中一个socket进行响应.

具体到实现，`3.9`版本扩展了`sock_common`，将原来记录`skc_reuse`进行了拆分.
```c
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
```c
struct inet_bind_bucket {
 	/* omitted */
 	unsigned short		port;
-	signed short		fastreuse;
+	signed char		fastreuse;
+	signed char		fastreuseport;
+	kuid_t			fastuid;
```
当Client的SYN报文到达时，Server会首先根据本地端口(SYN报文的`<dport>`)计算出一条hash冲突链，然后遍历该链表上的所有Socket，根据四元组匹配程度进行打分;如果使能了reuseport，那么可能有多个Socket都将拿到最高分，此时内核将随机选择一个进行后续处理。
```c
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

# 范例
## 单机监听多个Port
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/epoll.h>
#include <fcntl.h>
#include <pthread.h>
#include <errno.h>

#define LISTEN_START_PORT  9000
#define LISTEN_END_PORT    9300
#define EVENTS_SIZE        2048
#define MAXLINE 80
#define READ_BUF_SIZE 512


int make_socket_non_blocking (int sfd)
{
    int flags, s;
    flags = fcntl (sfd, F_GETFL, 0);
    if (flags == -1) {
        perror ("fcntl");
        return -1;
    }

    flags |= O_NONBLOCK;
    s = fcntl (sfd, F_SETFL, flags);
    if (s == -1) {
        perror ("fcntl");
        return -1;
    }

    return 0;
}

/* 作为server_test 在单机上单个进程监听多个port,测试多个连接 */
int tcp_listen(void * listen_port_arg)
{
    int i;
    int listen_port = 0;
    int listen_sock;
    struct sockaddr_in addr;
    int opt = 1;
    int backlog = 20;
    int epfd = 0;
    struct epoll_event tmp_event;
    struct epoll_event events[EVENTS_SIZE];

    if (listen_port_arg) {
        listen_port = *((int*)listen_port_arg);
    }
    listen_sock = socket(AF_INET, SOCK_STREAM, 0);
    setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    setsockopt(listen_sock, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
    make_socket_non_blocking(listen_sock);
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(listen_port);
    addr.sin_addr.s_addr = INADDR_ANY;
    bind(listen_sock, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_sock, backlog);

    epfd = epoll_create(1);
    if (epfd == -1) {
      perror("epoll_creat error!!\n");
      return -1;
    }
    tmp_event.events = EPOLLIN;
    tmp_event.data.fd = listen_sock;
    if(-1 == epoll_ctl(epfd, EPOLL_CTL_ADD, listen_sock, &tmp_event)) {
        perror("epoll_ctl error!!!\n");
        return -1;
    }


    struct sockaddr_in cli_addr;
    struct sockaddr_in connectedAddr;
    socklen_t addr_length = sizeof(cli_addr);
    int accept_fd;
    int eNum;
    char str[INET_ADDRSTRLEN] = {0};
    char str2[INET_ADDRSTRLEN] = {0};
    int read_n = 0;
    char read_buf[READ_BUF_SIZE];

    while (1) {
        eNum = epoll_wait(epfd, events, EVENTS_SIZE, -1);
        if (eNum == -1) {
            printf("epoll_wait error.\n");
            continue;
        }
        for (i = 0; i < eNum; i++) {
            if(events[i].data.fd == listen_sock) {
                if (!(events[i].events & EPOLLIN)) {
                    continue;
                }
                // accept_fd = accept(events[i].data.fd, (struct sockaddr*) &cli_addr, &addr_length);
                accept_fd = accept(events[i].data.fd, NULL, NULL);
                if (accept_fd >= 0) {
                    /*
                        getsockname(accept_fd, (struct sockaddr *)&connectedAddr, &addr_length);
                        printf("received from %s:%d to %s:%d\n",
                            inet_ntop(AF_INET,  &cli_addr.sin_addr, str, sizeof(str)),
                            ntohs(cli_addr.sin_port),
                            inet_ntoa(connectedAddr.sin_addr),
                            ntohs(connectedAddr.sin_port)
                        );
                        close(accept_fd);
                        accept_fd = -1;
                    */
                    make_socket_non_blocking(accept_fd);
                    tmp_event.events = EPOLLIN | EPOLLET;
                    tmp_event.data.fd = accept_fd;
                    if (-1 == epoll_ctl(epfd, EPOLL_CTL_ADD, accept_fd, &tmp_event)) {
                        perror("epoll_ctl error2.");
                        if (accept_fd >= 0) {
                            close(accept_fd);
                            accept_fd = -1;
                        }
                        continue;
                    }
                } else {
                    if (errno != EAGAIN) {
                      perror("accept error.");
                      printf("");
                      continue;
                    }
                }
            } else {
                if ((events[i].events & EPOLLERR) || (events[i].events & EPOLLHUP || (!(events[i].events & EPOLLIN)))) {
                    // perror("events flags error.");
                    if (events[i].data.fd >= 0) {
                        epoll_ctl(epfd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                        close(events[i].data.fd);
                        events[i].data.fd = -1;
                    }
                    continue;
                }
                if (events[i].data.fd < 0) {
                    perror("events fd error.");
                    continue;
                }
                if (events[i].events & EPOLLIN) {
                    read_n = read(events[i].data.fd, read_buf, sizeof(read_buf));
                    if (read_n <= 0) {
                        if (events[i].data.fd >= 0) {
                            epoll_ctl(epfd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                            close(events[i].data.fd);
                            events[i].data.fd = -1;
                        }
                    }
                }
            }
        }
    }
    return 0;
}


/* 
    compile: gcc -o listen_multi_ports listen_multi_ports.c -lpthread
 */
int main(int argc, char *argv[])
{
    int i, ret;
    int listen_port;
    pthread_t *pthread_ids = (pthread_t *)malloc((LISTEN_END_PORT - LISTEN_START_PORT + 1) * sizeof(pthread_t));

    if (NULL == pthread_ids) {
         printf("malloc filed.\n");
         return -1;
    }
    for (i = 0; i <= LISTEN_END_PORT-LISTEN_START_PORT; i++) {
         pthread_ids[i] = 0;
         listen_port = LISTEN_START_PORT + i;
         ret = pthread_create(&pthread_ids[i], NULL, (void *)tcp_listen, (void*)&listen_port);
         if (ret != 0) {
            printf("Create pthread error!\n");
            return -1;
         }
    }
    for (i = 0; i <= LISTEN_END_PORT-LISTEN_START_PORT; i++) {
        if (pthread_ids[i]) {
            pthread_join(pthread_ids[i], NULL);
        }
    }

    return 0;
}
```

机器的其他配置：
```c
pkill nginx
ulimit -HSn 1024000
sysctl -w fs.file-max=13025552
sysctl -w net.ipv4.ip_local_port_range='1024 64000'
ip link set eth03 up
for a in `seq 11 220`; do ip addr add 192.21.9.${a}/24 dev eth03;done
```

# 参考
```c
https://www.cnblogs.com/charlieroro/p/14096252.html
http://www.blogjava.net/yongboy/archive/2015/02/12/422893.html
https://switch-router.gitee.io/blog/reuseport/
https://wenfh2020.com/2021/10/12/thundering-herd-tcp-reuseport/

https://www.cnblogs.com/schips/p/12553321.html
```