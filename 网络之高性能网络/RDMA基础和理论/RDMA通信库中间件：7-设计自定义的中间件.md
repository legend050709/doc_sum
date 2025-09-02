```table-of-contents
```
# 概述
UCL(unified communication library) 是 RDMA 统一通讯库中间件。
业务使用RDMA，可以发挥RDMA的`kernel bypss`,  `offload cpu`、`zero-copy`的特性。
通过`UCL`，屏蔽了底层RDMA编程的众多概念以及编程复杂性，为业务提供`类socket`编程的接口，让业务快速、简易的享受到`RDMA`带来的低时延、高带宽的高性能网络服务。

# 内核相关的socket接口
## 参考
### rsocket 接口
`rsocket` 接口 如下所示：

```c
int rsocket(int domain, int type, int protocol);  
int rbind(int socket, const struct sockaddr *addr, socklen_t addrlen);  
int rlisten(int socket, int backlog);  
int raccept(int socket, struct sockaddr *addr, socklen_t *addrlen);  
int rconnect(int socket, const struct sockaddr *addr, socklen_t addrlen);  
int rshutdown(int socket, int how);  
int rclose(int socket);  
  
ssize_t rrecv(int socket, void *buf, size_t len, int flags);  
ssize_t rrecvfrom(int socket, void *buf, size_t len, int flags,  
    struct sockaddr *src_addr, socklen_t *addrlen);  
ssize_t rrecvmsg(int socket, struct msghdr *msg, int flags);  
ssize_t rsend(int socket, const void *buf, size_t len, int flags);  
ssize_t rsendto(int socket, const void *buf, size_t len, int flags,  
  const struct sockaddr *dest_addr, socklen_t addrlen);  
ssize_t rsendmsg(int socket, const struct msghdr *msg, int flags);  
ssize_t rread(int socket, void *buf, size_t count);  
ssize_t rreadv(int socket, const struct iovec *iov, int iovcnt);  
ssize_t rwrite(int socket, const void *buf, size_t count);  
ssize_t rwritev(int socket, const struct iovec *iov, int iovcnt);  
  
int rpoll(struct pollfd *fds, nfds_t nfds, int timeout);  
int rselect(int nfds, fd_set *readfds, fd_set *writefds,  
     fd_set *exceptfds, struct timeval *timeout);  
  
int rgetpeername(int socket, struct sockaddr *addr, socklen_t *addrlen);  
int rgetsockname(int socket, struct sockaddr *addr, socklen_t *addrlen);  
  
#define SOL_RDMA 0x10000  
enum {  
 RDMA_SQSIZE,  
 RDMA_RQSIZE,  
 RDMA_INLINE,  
 RDMA_IOMAPSIZE,  
 RDMA_ROUTE  
};  
  
int rsetsockopt(int socket, int level, int optname,  
  const void *optval, socklen_t optlen);  
int rgetsockopt(int socket, int level, int optname,  
  void *optval, socklen_t *optlen);  
int rfcntl(int socket, int cmd, ... /* arg */ );  
  
off_t riomap(int socket, void *buf, size_t len, int prot, int flags, off_t offset);  
int riounmap(int socket, void *buf, size_t len);  
size_t riowrite(int socket, const void *buf, size_t count, off_t offset, int flags);

```
## socket的创建和关闭
### 定义
### 使用

## readv 和 writev
### 定义
```c
#include <sys/uio.h>

ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);

struct iovec {
   void  *iov_base;    /* Starting address */
   size_t iov_len;     /* Number of bytes to transfer */
};
```
### 使用


# 内存管理
## 读写零拷贝
读写零拷贝，就涉及到什么时候进行内存的释放的问题？
那么，就需要通过异步的方式来进行释放内存。
**对于读来说**：

**对于写来说**：

### 问题
#### 解决方法
##### 回调函数
传递读和写完成的释放回调函数。

（1）发送完成，会在`send cq`中产生`cqe`， 则此时可以调用回调函数进行释放。
（2）接收完成，会在`recv cq`中产生`cqe`，自己定义一个类`skb`的结构，其中包含`addr, len, wr_id 等等信息`；基于`cqe 的 wr_id` 查找对应的类`skb`的结构，然后应用程序完成读取之后，调用读完成的回调函数进行释放。


## 内存模型

# 连接管理

## 控制连接

## QP连接

## 连接的释放
### 问题
**（1）问题一**：
类似于`TCP/IP Socket`编程，对于RDMA而言，`close_conn`之后，不应该继续通过 `write/send  API` 接口继续通过这个`conn`发送数据。
另外还有一个问题，就是应用程序(CPU驱动)关闭连接的时候，连接对应的QP此时存在`outstanding`未完成的WR，RNIC网卡通过`DMA`方式可能正在访问这个`WR`对应的内存地址，那么此时是不可以释放这个`QP`的。

**（2）问题二**：
RC服务类型的`QP`「RNIC上需要维护带状态的QP」的数量存在过多时，那么会很大程度上影响性能。
对于UD类型的`QP`，由于不带有状态，应该没有该问题。

那么，RC服务类型的`QP`数量过多影响性能的问题，解决方法是：
1> SQP(共享QP，即多个连接进行QP的复用)，适用于服务端和客户端。
2> XRC: 接收端多个QP共享一个QP。只适用于接收端。

拿SQP(共享QP）举例， 多个连接共享一个QP，那么关闭连接，是否进行QP的释放，需要注意。

### 解决：延迟释放+引用计数
**(1) RC服务类型**：
 `close 连接`的时候：
《1.1》如果这个`conn`对应的QP，还存在没有完成（outstanding）的`send wr, recv wr`「每个`conn`中保存 `send_uncompleted`, `recv_uncompleted`, 表示这个conn上未完成的`SR`, `RR`数量」;
  则只是给这个conn打上标记(`closing标记`)，后续不允许通过这个`conn`继续调用`send/writev API`接口继续发送。
  
  至于这个`conn`以及其关联的`QP`的释放，则是延迟释放。在`Poll CQ`得到`WQE`的时候，这个`conn`下的所有的`send wr, recv wr`都得到完成(获取到`cqe`)了之后「无论是否发生错误」，才允许释放这个`conn`及其关联的`qp`。
  > 注：其中，如果是这个`conn`的`send wr`都完成了之后，那么就可以`flush`这个`QP`(通过设置`QP`状态为`Err`)；`Flush` 之后，所有未完成的RR，都会产生带`err`的`WC`。

《1.2》如果这个`conn`对应的QP，不存在没有完成（outstanding）的`send wr, recv wr`，则直接释放`conn`和对应的`QP`。

**(2) SQP服务类型**：
SQP类型，多个连接可能关联到一个`RC`类型的`QP`上。
那么，除了每个`conn`中保存 `send_uncompleted`, `recv_uncompleted`「表示这个conn上未完成的`SR`, `RR`数量」之外。
每个`QP`中还存在`tx_refcnt`和`rx_refcnt`, 分别表示这个QP上关联了多少个存在`SR「send wr」`和`RR「recv WR」`的`conn`。

只有，一个`QP`上的`tx_refcnt`和`rx_refcnt`都为0的时候，才允许对这个QP进行释放。


# 应用层发送/接收datagram或stream形式的数据
## 背景
socket层面存在stream和datagram，RDMA 本身没有 message 和 stream的概念，
在RDMA中看来，就是WR(send wr, recv wr)以及产生的WC(send cqe, recv cqe)，其就类似于 message（即类似于UDP的数据报）；


## 分析
上层应用（调用类socket接口的程序）发送数据，是以`message`的形式发送，还是以`stream`的形式发送。
```c
ucl_socket(AF_INET, SOCK_STREAM, 0);

ucl_socket(AF_INET, SOCK_DGRAM, 0);
```

（1）如果是以message的形式发送，那么就意味着对端应用层调用RDMA通讯库中间件的接收接口，一次调用刚好收到的就是发送的message的大小。
即：==一个 recq的 cqe 接收的刚好就是发送方发送的数据的大小==。
就类似于socket编程中，发送方通过调用一次send发送数据报，接收方通过调用一次recv接收数据报，收发的字节数相同。类似于udp中的message的大小存在限制，RDMA通讯库中间件发送message的大小应该也存在限制。

（2）如果是以stream的形式发送，那么就意味着发送方发送一段数据，可能调用多次RDMA通讯库中间件的send接口，接收方也调用多次recv接口。具体什么时候接收结束，是不确定的。
对于`tcp socket`而言，由于socket层面进行接收数据时不存在边界，不知道什么时候接受结束。
一般会通过先发送一个头，头中指定接下来要发送的数据的大小来强行设置数据内容的边界。
接收端也是先接收头，然后收到头之后，基于头中的数据长度，接下来强行接受指定长度的数据内容，直到接收指定长度的数据成功，否则一直接收。


## 原则
发送端, 应用调用`writev`的 `iov` 如何对应底层的`wr`以及`sge`, 是根据接收端的 `recv wr`决定的：


### 接收端的设置
(1)`stream(tcp)` 形式的`qp`, 设置的 每个`recv wr` 的 `sge` 个数为1，每个`sge`的默认大小是 4k; `recv qp`中默认有126个 `recv wr`;
(2) `datagram(udp)` 形式的`qp`, 设置的 每个`recv wr` 的 `sge` 的个数是16, 每个`sge`的默认大小是 4k; `recv qp`中默认有126个 `recv wr`;

### 发送端writev参数中 多个iov组合为一个wr

#### `stream(tcp)` 形式的qp
发送端，`writev`的 多个 iov 加起来的长度之和，如果低于4k，可以合并到一个wr中，使用多个sge串起来。

「 通信库中间件内部的某个结构(比如：`skb`） 和 sge 可以一一对应，然后另外一个结构 （比如: `mult_skb`）和  wr一一对应 」

比如：writev 有 6个iov，长度分别为：`1k, 2k, 500B, 2k, 1k, 3k`;
那么：前3个iov是一个wr，这个wr下有3个sge，长度分别为：`1k, 2k, 500B`；第四个iov和第五个`iov`是一个`wr`，最后一个`iov`是一个wr。
这样，接收端，第一个recv wr 直接接收到 3.5k的数据，放入到连续内存中「假设通讯库内部的每个`skb`的大小是4k，此时只有一个sge」，虽然发送的时候是一个wr，3个`sge`，地址并不连续。
```bash
即：
发送端：WR1[1k, 2k, 500B], WR2[2k, 1k], WR3[3k];
接收端：WR1[3.5k], WR2[3k], WR3[3k];
```


基本的原则：就是发送端的一个`send wr` 对应 接收端的一个`recv wr`，但是发送端的这个`send wr`下的`sge`的个数，和接收端的`recv wr`的 `sge`的个数可能不同。
所以其实上面的 发送端的 `writev` 有 6个`iov`，也完全可以使用6个`wr`，每个wr是一个`sge`；这样就会消耗 接收端的 6个 `recv wr`; 这样效率不是很高。

#### datagram(udp) 形式的qp
(1) `stream(tcp)` 形式的`socket`没有边界，本端的 m次 writev 调用对应对端的 n次 readv调用; 
(2) `datagram(udp)`的 `socket` 由于是存在边界的，本端的一次 writev 调用 需要对应对端的 一次 readv 调用，就可以完成数据的发和收；
这个也是为什么`datagram(udp)` 形式的qp, 接收端的 每个`recv wr` 的 sge 的个数是16, 每个sge的默认大小是 4k，这个就是说最多可以支持发送端一次发送64k的数据「即发送端一次writev，多个`iov`加起来的长度之和<=`64k`」。
如果发送端一次 writev 的多个 `iov`对应多个`wr`，那么就会消耗接收端的多个recv wr，接收端的 `readv` 就无法知道哪些 `recv wr`对应本次`readv`，最简单的就是一次readv只是读取一个`recv wr`，将这个`recv wr`的一个或者多个`sge`，放入到多个`iov`中。

因此：发送端一次`writev` 的多个 iov，只能对应一个 `wr`，这个`wr`下可能有多个`sge`，多个sge加起来的长度之和小于`64k`。


### 发送端writev参数中 一个iov拆分为多个skb
因为一个`sge`，对应一个通信库中的一个 `skb` 结构，skb 的大小是 `4k`。如果某个`iov`的大小大于4k，那么需要将其拆分为多个 `skb`。

#### stream(tcp) 形式的qp
比如：writev 有 5个iov，长度分别为：`5k, 500B, 13k, 1k, 3k`;
那么，就拆分为：`[4k, 1k], 500B, [4k, 4k, 4k, 1k], 1k, 3k`; 

方式一：消耗8个WR (相对简单一些，但是消耗的wr可能略多一点点):
```bash
第一个wr，有一个sge(4k);
第二个wr，有一个sge(1k);
第三个wr，有一个sge(500B);
第四个wr，有一个sge(4k);
第五个wr，有一个sge(4k);
第六个wr，有一个sge(4k);
第七个wr，有一个sge(1k);
第八个wr，有两个sge(1k, 3k);

即： WR1[4k], WR2[1k], WR3[500B], WR4[4k], WR5[4k], WR6[4k], WR7[1k], WR8[1k, 3k]
```

方式二：消耗7个WR
```bash
第一个wr，有一个sge(4k);
第二个wr，有两个sge(1k,500B);
第三个wr，有一个sge(4k);
第四个wr，有一个sge(4k);
第五个wr，有一个sge(4k);
第六个wr，有两个sge(1k, 1k);
第七个wr，有一个sge(3k);
```

##### 注意点
（1）零拷贝的情况下，`iov`中可能存在释放函数，一个`iov`拆分为多个`skb`, 甚至是多个WR，但是其只能对应一个释放函数进行统一释放iov的空间。

比如：上面采用方式一进行拆分，则释放的时候：
```bash
第一个WR 发送完成之后，不应该释放；而是在第二个WR发送完成之后，直接调用第一个iov的释放函数，直接释放5k的内存；
第三个WR 发送完成之后，直接调用第二个iov的释放函数，直接释放500B的内存；
第四、五、六个WR发送完成之后，不应该释放；而是在第七个WR发送完成之后，直接调用第三个iov的释放函数，直接释放13k的内存；
第八个WR 发送完成之后，基于WC找到WR, 这个WR下存在2个sge，对应第四个、第五个iov，调用其释放函数，分别释放1k和3k的内存；

注：上面的释放顺序，必须保证WR的有序。即：post 2个 wr，必然是第一个WR先产生WC，第二个WR后产生WC。
```

#### datagram(udp) 形式的qp
比如：writev 有 5个iov，长度分别为：`5k, 500B, 13k, 1k, 3k`;
那么，就拆分为：`[4k, 1k], 500B, [4k, 4k, 4k, 1k], 1k, 3k`; 

一共消耗一个WR，其下面有9个`sge(4k, 1k, 500B, 4k, 4k, 4k, 1k, 1k, 3k`), sge总长度是`22.5K < 64K`;


##### 注意点
同上。

## 范例

（1）应用需要发送3个message，每个message的大小是5k。
假设发送端底层的RDMA中的每个sge对应的block的最大大小是4k。
底层RDMA建立QP的时候，如果是message类型的QP，则 可以将 QP的 max_send_sge 或者 max_recv_sge 设置为 16，每个sge最大是4k（一个sge的大小对应提前申请的block的大小）；
QP的 max_recv_sge 不可以设置为1。因为对于发送message类型的消息而言，不知道发送的message多大，如果 max_recv_sge 设置过小，一个WR可能装不下要发送的message，通过多个WR发送一个message，可能就不是message形式了。
设置 QP的  max_recv_sge = 16，每个sge最大是4k，则意味着发送的message的最大的大小不可以超过64k。
所以，回到上面，发送3个message，每个message的大小是5k；那么就是3个message总共对应3个wr，每个message对应一个WR，每个WR下2个sge，一个sge是4k，另外一个seg是1k。
而不是说：发送3个message，每个5k，一共15k，在底层只有一个wr，wr下有4个sge，前三个sge是4k，最后一个sge是3k。因为如果是一个wr，那么对端收的时候是消耗一个receive wr, 这样就类似于一个message消息了，而不是3个message消息。
因此，为了保证message的边界，那么一个message，对应一个WR；message可能拆分为多个block，类似于一个WR下多个sge，一个sge就是一个block。

（2）应用要发送15k的stream的内容时：
由于是stream的消息，就意味着不存在边界。那就底层RDMA就可以随意划分，将15K的内容随意划分到N给WR中。
如果是stream类型的QP，接收端为了防止浪费，那么可以将 每个WR的 max_recv_sge 设置为1.
这样的话，发送15k的stream的内容，对应4个WR，每个WR只有一个sge(sge对应的block大小为4k)。
由于发送端用了4个send WR，接收端也是消费了4个receive wr，应用层面收到这4个receive wr对应的内存的数据后，应用层面自己来进行组装，然后可能会基于内容里自定义的头来区分边界。

## 小结
（1）如果是以message的形式发送一段数据，发送方发送一个message，就是对应一个Send WR，至于这个message被拆分为多少一个block，那么就对应着这个Send WR下存在多少个sge；==反正就是一个message 对应一个send WR，然后消耗一个接收方的recv WR==。
发送的message形式的数据大小受到限制，主要受限于QP属性中一个WR下的 max_recv_sge 的设置，以及一个sge 最大大小。

（2）如果是以stream的形式发送一段数据，发送方发送一段数据，可能对应N个send WR, 至于一个stream形式的QP的send WR下最多存在多少个sge，建议设置为1，如果设置过多，那么存在内存浪费。同理，接收方的QP的recv WR下的最大sge个数也设置为1。接收方接收这段数据，也是消耗N个recv WR。

（3）发送方和接收方的 block的大小都是中间库自己设置的，可以设置为不同的大小。另外，发送方的QP和接收方的QP的属性中的 max_send_sge/max_recv_sge, max_send_wr/max_recv_wr 也可以设置为不同。
比如，要发送22k的message形式的数据，发送方的block的大小为4k，则需要消耗一个send wr, wr下存在6个sge，前5个sge是4k大小，最后一个sge是2k大小。
接收方的block如果是20k大小，那么只需要消耗一个recv wr，wr下存在2个sge，前一个sge是20k大小，最后一个是2k大小。


# 自定义的epoll （io多路复用）
自定义的 `epoll`，实现类似于内核协议栈提供的`epoll`的`API`接口。内核协议栈对外提供的 epoll 的接口如下所示：
```c
int epoll_create(int size); 
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); 
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);
```
自定义的 `epoll`，也需要提供类似于上面的接口。
```c
int xxx_epoll_create(int size); 
int xxx_epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); 
int xxx_epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);
```


## 事件
### EPOLLIN 可读事件
`recv cq` 产生一个`cqe`，说明`receive queue`消耗了一个`wqe`, 即收到了一份数据。那么此时就是一个可读事件, 同时还需要继续给`receive queue`补充一个`wqe`「类似于令牌桶需要补充tokens」。


#### `EPOLLIN` 事件 + `read` 返回值
##### Linux 内核中 `EPOLLIN` 事件 + 调用`read` 返回 0
在 Linux 内核的 `Epoll` 中，ET模式下的非阻塞的`fd`，**判断对端是否关闭连接**（即是否接受到`fin`报文），有两种方法：

1> **方法一**：收到`EPOLLIN` 事件，`while` 循环调用`read`, 查看`read`是否返回 0；
```c
ssize_t read_n(int fd, void *vptr, size_t n)
{
    size_t nleft;
    ssize_t nread;
    char *ptr;

    ptr = vptr;
    nleft = n;
    while (nleft > 0) {
        if ((nread = read(fd, ptr, nleft)) < 0) {
            if ((errno == EINTR) || (errno == EAGAIN))
                nread = 0;      /* and call read() again */
            else
                return (-1);
        } else if (nread == 0)
            break;      /* EOF */

        nleft -= nread;
        ptr += nread;
    }

    return (n - nleft);     /* return >= 0 */
}
```


2> **方法二**：`epoll_ctl` 添加 `EPOLLRDHUP` 事件，查看是否有`EPOLLRDHUP` 事件的产生。 


通过上诉两种方法，某一端在断开`TCP`连接之后，会发送`finish`报文告知对端，对端可以通过上面的两种方法来感知连接的关闭，进而采取

##### Linux 内核中 `EPOLLIN` 事件 + 调用`read` 返回 `-1, errno = EAGAIN`
在 `Linux` 内核的 `Epoll` 中，`ET`模式下的非阻塞的`fd`，收到  `EPOLLIN` 事件，则需要：
循环调用 `read`, 直到返回 `-1, errno = EAGAIN`, 表示接受缓冲区的数据读取完毕，下次再进行重试。

##### RDMA中`EPOLLIN` 事件 + `read` 返回值
不同于`Linux` 内核中，一端调用`close`断开`TCP`连接，会发送`finish`报文来告知对端，然后对端可以感知到。
==对于`RDMA`中而言，一方关闭了QP的连接（销毁QP），没有发送任何信息来告知对端的，对端也无法判断本端的`QP`关闭了==。
> 注：RDMA中的连接可以借助于基于内核的TCP连接来感知，即每个RDMA连接对应一个TCP的连接。TCP连接可以用来：
> （1）通过QP发送数据前，通过TCP连接作为控制连接来协商QP等信息。
> （2）用来保活，感知对端QP的正常。
> （3）通过事件感知对端的连接的关闭等。

因此，==设计通讯库时，也不可以在`RDMA通讯库`的接口`xxx_read/ xxx_recv`，调用`RDMA的相关接口`无法读取到数据时（即读取的数据的长度为0），对用户直接返回`0`，而是应该返回的是`-1, errno = EAGAIN`==。
因为，应用程序的编写，还是按照`Linux 内核下`的`EPOLLIN`事件 + `read`的编程习惯来调用`RDMA通讯库`的接口`xxx_read/ xxx_recv`。
用户会在`xxx_read/ xxx_recv`返回0时，认为对端关闭了连接，进而调用`xxx_close`将本端的连接也给关闭了。

### EPOLLOUT 可写事件
`send cq` 产生一个`cqe`，说明`send queue`消耗了一个`wqe`, 即发送数据完成。那么此时就是一个可写事件。此时不需要给`send queue`补充`wqe`。
业务需要发送数据的时候，会自动实时的给`send queue`进行`post WR「即补充wqe」`。


## 非阻塞异步IO
## 异步polling模式
## 异步epoll事件模式


# 资源管理
## 资源优化
### SRQ
### SQP：多个conn下的QP复用（Shared QP）
#### DCT方式



# 规划
![](attachments/image%20(12).png)

![](attachments/image%20(14).png)

![](attachments/image15.png)


# 参考
```bash

```