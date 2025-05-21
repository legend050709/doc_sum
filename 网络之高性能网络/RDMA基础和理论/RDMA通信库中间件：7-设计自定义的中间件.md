```table-of-contents
```
# 概述
UCL(unified communication library) 是 RDMA 统一通讯库中间件。
业务使用RDMA，可以发挥RDMA的`kernel bypss`,  `offload cpu`、`zero-copy`的特性。
通过`UCL`，屏蔽了底层RDMA编程的众多概念以及编程复杂性，为业务提供`类socket`编程的接口，让业务快速、简易的享受到`RDMA`带来的低时延、高带宽的高性能网络服务。

# 内核相关的socket接口
## 参考
### rsocket 接口
rsocket 接口 如下所示：

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


# 应用层发送/接收datagram或stream形式的数据
## 背景
socket层面存在stream和datagram，RDMA 本身没有 message 和 stream的概念，
在RDMA中看来，就是WR(send wr, recv wr)以及产生的WC(send cqe, recv cqe)，其就类似于 message（即类似于UDP的数据报）；


## 分析
上层应用（调用类socket接口的程序）发送数据，是以message的形式发送，还是以stream的形式发送。

（1）如果是以message的形式发送，那么就意味着对端应用层调用RDMA通讯库中间件的接收接口，一次调用刚好收到的就是发送的message的大小。就类似于socket编程中，发送方通过调用一次send发送数据报，接收方通过调用一次recv接收数据报，收发的字节数相同。类似于udp中的message的大小存在限制，RDMA通讯库中间件发送message的大小应该也存在限制。

（2）如果是以stream的形式发送，那么就意味着发送方发送一段数据，可能调用多次RDMA通讯库中间件的send接口，接收方也调用多次recv接口。具体什么时候接收结束，是不确定的。
对于tcp socket而言，由于socket层面进行接收数据时不存在边界，不知道什么时候接受结束。
一般会通过先发送一个头，头中指定接下来要发送的数据的大小来强行设置数据内容的边界。
接收端也是先接收头，然后收到头之后，基于头中的数据长度，接下来强行接受指定长度的数据内容，直到接收指定长度的数据成功，否则一直接收。

## 范例

（1）应用需要发送3个message，每个message的大小是5k。发送端底层的RDMA中的每个sge对应的block的最大大小是4k。
底层RDMA建立QP的时候，如果是message类型的QP，则 可以将 QP的 max_send_sge 或者 max_recv_sge 设置为 16，每个sge最大是4k（一个sge的大小对应提前申请的block的大小）；
QP的 max_send_sge不可以设置为1。因为对于发送message类型的消息而言，不知道发送的message多大，如果max_recv_sge 设置过小，一个WR可能装不下要发送的message，通过多个WR发送一个message，可能就不是message形式了。
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
（1）如果是以message的形式发送一段数据，发送方发送一个message，就是对应一个Send WR，至于这个message被拆分为多少一个block，那么就对应着这个Send WR下存在多少个sge，反正就是一个message 对应一个send WR，然后消耗一个接收方的recv WR。发送的message形式的数据大小受到限制，主要受限于QP属性中一个WR下的max_send_sge的设置，一个sge 最大大小。

（2）如果是以stream的形式发送一段数据，发送方发送一段数据，可能对应N个send WR, 至于一个stream形式的QP的send WR下最多存在多少个sge，建议设置为1，如果设置过多，那么存在内存浪费。
   同理，接收方的QP的recv WR下的最大sge个数也设置为1。接收方接收这段数据，也是消耗N个recv WR。

（3）发送方和接收方的 block的大小都是中间库自己设置的，可以设置为不同的大小。另外，发送方的QP和接收方的QP的属性中的 max_send_sge/max_recv_sge, max_send_wr/max_recv_wr 也可以设置为不同。
比如，要发送22k的message形式的数据，发送方的block的大小为4k，则需要消耗一个send wr, wr下存在6个sge，前5个sge是4k大小，最后一个sge是2k大小。
接收方的block如果是20k大小，那么只需要消耗一个recv wr，wr下存在2个sge，前一个sge是20k大小，最后一个是2k大小。


# io模型
## 事件
### 可读事件
`recv cq` 产生一个`cqe`，说明`receive queue`消耗了一个`wqe`, 即收到了一份数据。那么此时就是一个可读事件, 同时还需要继续给`receive queue`补充一个`wqe`「类似于令牌桶需要补充tokens」。

### 可写事件
`send cq` 产生一个`cqe`，说明`send queue`消耗了一个`wqe`, 即发送数据完成。那么此时就是一个可写事件。此时不需要给`send queue`补充`wqe`。
业务需要发送数据的时候，会自动实时的给`send queue`进行`post WR「即补充wqe」`。


## 非阻塞异步IO
## 异步polling模式
## 异步epoll事件模式


# 资源管理

## 资源优化
### SRQ
### QP复用
#### DCT方式
#### 多个连接复用QP

# 参考
```bash

```