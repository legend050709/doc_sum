```table-of-contents
```
# epoll的多线程扩展问题
`epoll` 的多线程扩展性的问题主要体现在做多核之间负载均衡上，有两个典型的场景：
**(1) listen-sock的 accept 的 scale out 问题**：
一个 TCP 服务器，对同一个 `listen fd` 在多个 CPU 上调用 `accept` 系统调用

**(2) 普通sock的 read 的 scale out问题**：
大量 TCP 连接下调用 `read` 系统调用

## 多线程下的`epoll`模型
### 多个线程共享一个`epollfd`
(1) 只创建一个`epoll`, 多个`worker`线程中对同一个`epoll` 进行`epoll_wait`。

### 多个线程共享一个`listen-sock`
(2) 只有一个`listen-sock`, 每个`worker`都创建`epoll`, 监听同一个`listen-sock`的`EPOLLIN`事件。每个`worker`从`listen-sock`中`accept`产生的`new-fd`，加入到自身的`epoll`中监听。

### 每个线程独占`epollfd`&`listen-sock`& `common-sock`
(3) 存在两种类型的线程：`receiver`和`worker`；每个线程创建自己的`epoll`，`receiver`中创建`listen-sock`，并被自己的`epoll`监听，`accept`产生的`new-fd`，通过管道的方式传入到某个`worker`中，然后加入到这个`worker`的`epoll`中监听。


#### 问题

`reuseport` 选项主要解决了两个问题：
<1>（A 图）单个 `listen socket` 遇到的性能瓶颈。
<2>（B 图）单个 `listen socket` 多个线程同时 `accept`，但是多个线程资源分配不均。

![](attachments/Pasted%20image%2020250531222817.png)

其实`reuseport`还解决了一个很重要的问题：
在 `tcp` 多线程场景中，（B 图）服务端如果所有新链接只保存在一个 `listen socket` 的 `全链接队列` 中，那么多个线程去这个队列里获取（accept）新的链接，势必会出现多个线程对一个公共资源的争抢，争抢过程中，大量资源的损耗。

而（C 图）有多个 `listener` 共同 `bind/listen` 相同的 `IP/PORT`，也就是说每个进程/线程有一个独立的 `listener`，相当于每个进程/线程独享一个 `listener` 的全链接队列，不需要多个进程/线程竞争某个公共资源，能充分利用多核，减少竞争的资源消耗，效率自然提高了。



## accept 的 scale out 问题：同一个 `listen fd` 在多个 CPU 上调用 `accept`的问题
### 背景
一个典型的场景是一个需要处理大量短连接的 HTTP 1.0 服务器，由于需要 accept() 大量的 TCP 建连请求，所以希望把这些 accept() 分发到不同的 CPU 上来处理，以充分利用多 CPU 的能力。

这在实际生产环境是存在的， Tom Herbert 报告有应用需要处理每秒 4 万个建连请求；当有这么多请求的时候，很显然，将其分散到不同的 CPU 上是合理的。

然后实际上，事情并没有这么简单，直到 Linux 4.5 内核，都无法通过 epoll(2) 把这些请求水平扩展到其他 CPU 上。下面我们来看看 epoll 的两种模式 LT(level trigger, 水平触发) 和 ET(edge trigger, 边缘触发) 在处理这种情况下的问题。

### 线程模型
一个愚蠢的做法是是将同一个 `epoll fd` 放到不同的线程上来 `epoll_wait()`，这样做显然行不通；
同样，将同一个`listen fd` 加到不同的线程中的 `epoll fd`中也行不通，下面就是对于这种情况的分析。

本来==期望==是：**同一个`listen fd` 加到不同的线程中的 `epoll fd`中，然后进行 `accept`，通过增大线程个数来达到新建能力的水平扩展**。
实际这样做，是存在问题的，分析如下。

### 水平触发的问题：不必要的唤醒
因为 epoll 的水平触发模式和 `select(2)` 一样存在 “惊群效应（thundering herd）”，在不加特殊标志的水平触发模式下，当一个新建连接请求过来时，所有的 worker 线程都都会被唤醒，下面是一个这种 case 的例子：
```bash
1. 内核：收到一个新建连接的请求  
2. 内核：由于 "惊群效应" ，唤醒两个正在 epoll_wait() 的线程 A 和线程 B  
3. 线程A：epoll_wait() 返回  
4. 线程B：epoll_wait() 返回  
5. 线程A：执行 accept() 并且成功  
6. 线程B：执行 accept() 失败，accept() 返回 EAGAIN


英文为：
Kernel: Receives a new connection.
Kernel: Notifies two waiting threads A and B. Due to "thundering herd" behavior with level-triggered notifications kernel must wake up both.
Thread A: Finishes epoll_wait().
Thread B: Finishes epoll_wait().
Thread A: Performs accept(), this succeeds.
Thread B: Performs accept(), this fails with EAGAIN.
```

其中，线程 B 的唤醒完全没有必要，仅仅只是浪费宝贵的 CPU 资源而已，水平触发模式的 epoll 的扩展性很差。

### 边缘触发的问题：不必要的唤醒和饥饿问题
#### 不必要的唤醒
既然水平触发模式不行，那是不是边缘触发模式会更好呢？实际上并没有。我们来看看下面这个例子：
```bash
1. 内核：收到第一个连接请求。线程 A 和 线程 B 两个线程都在 epoll_wait() 上等待。由于采用边缘触发模式，所以只有一个线程会收到通知。这里假定线程 A 收到通知  
2. 线程A：epoll_wait() 返回  
3. 线程A：调用 accpet() 并且成功  「注：边缘触发情况下，用户会通过循环来调用 accept 」
4. 内核：收到第二个建连请求  
5. 内核：此时线程 A 可能还在执行 accept() 处理，只剩下线程 B 在等待 epoll_wait()，于是唤醒线程 B  
6. 线程A：继续执行 accept() 直到返回 EAGAIN  
7. 线程B：执行 accept()，并返回 EAGAIN，此时线程 B 可能有点困惑("明明通知我有事件，结果却返回 EAGAIN")  
8. 线程A：再次执行 accept()，这次终于返回 EAGAIN

```
可以看到在上面的例子中，线程 B 的唤醒是完全没有必要的。

#### 饥饿问题
由于边缘触发下，一般是通过循环来调用 `accept` ，直到 返回`-1，errno= EAGAIN`.
事实上边缘触发模式还存在饥饿的问题，我们来看下面这个例子：
```bash
1. 内核：接收到两个建连请求。线程 A 和 线程 B 两个线程都在等在 epoll_wait()。由于采用边缘触发模式，只有一个线程会被唤醒，我们这里假定线程 A 先被唤醒  
2. 线程A：epoll_wait() 返回  
3. 线程A：调用 accpet() 并且成功  「注：边缘触发情况下，用户会通过循环来调用 accept 」
4. 内核：收到第三个建连请求。此时线程 A 还没有处理完(没有返回 EAGAIN)。  
5. 线程A：继续执行 accept() 希望返回 EAGAIN后 再进入 epoll_wait() 等待，然而它又 accept() 成功并处理了一个新连接  
6. 内核：又收到了第四个建连请求  
7. 线程A：又继续执行 accept()，结果又返回成功
8. 虽然期间每次来一个请求，只剩下线程 B 在等待 epoll_wait()，会唤醒线程 B ，但是线程B，可能每次都可能 accept() 返回 EAGAIN；
   导致线程`B`没必要的唤醒，以及一直处于饥饿状态。
```
在这个例子中个，所有的建连请求全都会给线程 A，导致这个负载均衡根本没有生效，线程 A 很忙而线程 B 没有活干。


### 正确的做法
既然水平触发和边缘触发都不行，那怎样才是正确的做法呢？有两种 workaround 的方式。

#### 方法一：水平触发 + EPOLLEXCLUSIVE
Linux 4.5+ 开始出现的水平触发模式新增的 `EPOLLEXCLUSIVE` 标志，这个标志会保证一个事件只有一个 epoll_wait() 会被唤醒，避免了 “惊群效应”，并且可以在多个 CPU 之间很好的水平扩展。

##### 注意
(1) `EPOLLEXCLUSIVE` 是 **Linux 4.5 引入**的选项，用于解决多进程/线程下的`epoll`监听`同一 socket` 时的 **惊群问题**（thundering herd）。
(2) `EPOLLEXCLUSIVE` **必须与 `EPOLLLT`（默认模式）配合使用**。
如果尝试在 `EPOLLET`（边缘触发）模式下使用，内核会直接忽略该标志（不会报错，但行为等同于未设置）。

#### 方法二：边缘触发 + EPOLLONESHOT
当内核不支持 `EPOLLEXCLUSIVE` 时，可以通过 ET 模式下的 `EPOLLONESHOT` 来模拟 LT + `EPOLLEXCLUSIVE` 的效果。

**流程**
```bash
1. `内核：接收到两个建连请求。线程 A 和 线程 B 两个线程都在等在 epoll_wait()。由于采用边缘触发模式，只有一个线程会被唤醒，我们这里假定线程 A 先被唤醒`
2. `线程A：epoll_wait() 返回`
3. `线程A：调用 accpet() 并且成功` 「注：边缘触发情况下，用户会通过循环来调用 accept 」
4. `线程A：调用 epoll_ctl(EPOLL_CTL_MOD)，这样会重置 EPOLLONESHOT 状态，并将这个 socket fd 重新准备好 “`
```

**缺点**
当然这样是有代价的，需要在每个事件处理完之后额外多调用一次 epoll_ctl(EPOLL_CTL_MOD) 重置这个 fd。这样做可以将负载均分到不同的 CPU 上；
同一时刻，只能有一个 worker 调用 accept(2)。显然，这样又限制了处理 accept(2) 的吞吐。

#### 其他方案：`SO_REUSEPORT`
如果不依赖于 epoll() 的话，也还有其他方案。一种方案是使用 `SO_REUSEPORT` 这个 `socket option`，创建多个 `listen socket` 共用一个端口号。

**缺点**
这种方案其实也存在问题: 当一个 `listen socket fd` 被关了，已经被分到这个 `listen socket fd` 的 `accept` 队列上的请求会被丢掉。

**解决方法**
从 Linux 4.5 开始引入了 `SO_ATTACH_REUSEPORT_CBPF` 和 `SO_ATTACH_REUSEPORT_EBPF` 这两个 BPF 相关的 socket option。
通过巧妙的设计，应该可以避免掉建连请求被丢掉的情况。

## read 的 scale out 问题
### 背景
除了 上面说的 `accept` 的水平扩展(`scale out`)问题之外， 普通的 `read` 在多核系统上也会有扩展性的问题。
设想以下场景：一个 `HTTP` 服务器，需要跟大量的 `HTTP client` 通信，你希望尽快的处理每个客户端的请求。而每个客户端连接的请求的处理时间可能并不一样，有些快有些慢，并且不可预测，因此将它们均匀分配到工作线程的桶中可能会增加平均延迟。

一种更好的排队策略可能是：用一个 `epoll fd` 「多个`worker`共享一个`epollfd`」来管理这些连接并设置 `EPOLLEXCLUSIVE` ，然后多个 `worker` 线程来`epoll_wait()`，取出就绪的连接并处理。油管上有个视频介绍这种称之为 `combined queue` 的模型。

下面我们来看看 `epoll` 处理这种模型下的问题：

### 水平触发的问题：数据乱序
实际上，由于水平触发存在的 “惊群效应”，我们并不想用该模型。另外，即使加上 `EPOLLEXCLUSIVE` 标志，仍然存在数据竞争的情况，我们来看看下面这个例子：
```bash
1. 内核：收到 2047 字节的数据  
2. 内核：线程 A 和线程 B 两个线程都在 epoll_wait()，由于设置了 EPOLLEXCLUSIVE，内核只会唤醒一个线程，假设这里先唤醒线程 A  
3. 线程A：epoll_wait() 返回  
4. 内核：内核又收到 2 个字节的数据  「一共是2049B的数据」
5. 内核：线程 A 还在干活，当前只有线程 B 在 epoll_wait()，内核唤醒线程 B  
6. 线程A：调用 read(2048) 并读走 2048 字节数据  
7. 线程B：调用 read(2048) 并读走剩下的 1 字节数据
```

这上述场景中，数据会被分片到两个不同的线程，如果没有锁保护的话，数据可能会存在乱序。

### 边缘触发的问题：数据乱序
既然水平触发模型不行，那么边缘触发呢？实际上也存在相同的竞争，我们看看下面这个例子：
```bash
Kernel: two threads are waiting for the data: A and B. Due to the "edge triggered" behavior only one is notified.
Thread A: finishes epoll_wait()
Thread A: performs a read(2048) and reads full buffer of 2048 bytes
Kernel: the buffer is empty so the kernel arms the file descriptor again
Kernel: receives 1 byte of data
Kernel: one thread is currently waiting in epoll_wait, wakes up Thread B
Thread B: finished epoll_wait()
Thread B: performs read(2048) and gets 1 byte of data
Thread A: retries read(2048), which returns nothing, gets EAGAIN
```

### 正确的做法
实际上，要保证同一个连接的数据始终落到同一个线程上，在上述 `epoll` 模型下，唯一的方法就是 `epoll_ctl` 的时候加上 `EPOLLONESHOT` 标志，然后在每次处理完重新把这个 `socket fd` 加到 `epoll` 里面去。

## 总结

要正确的用好 `epoll` 并不容易，要用 `epoll` 实现负载均衡并且避免数据竞争，必须掌握好 `EPOLLONESHOT` 和 `EPOLLEXCLUSIVE` 这两个标志。

# epoll的惊群问题
## 前言
惊群比较抽象，类似于抢红包 。它多出现在高性能的多进程/多线程服务中，例如：nginx。  

## epoll 原理

![](attachments/Pasted%20image%2020250601152718.png)

**Linux epoll的原理图**, 如上所示。
`epoll`的流程，如下所示：
1. 创建epoll句柄，初始化相关数据结构
2. 为epoll句柄添加文件句柄，注册睡眠entry的回调
3. 事件发生，唤醒相关文件句柄睡眠队列的entry，调用其回调
4. 唤醒epoll睡眠队列的task，搜集并上报数据

### 步骤一：创建epoll句柄，初始化相关数据结构
创建一个epoll文件描述符，注意，后面操作epoll的时候，就是用这个epoll的文件描述符来操作的，所以这就是epoll的句柄，精简过后的epoll结构如下：
```c
 struct eventpoll {
    // 阻塞在epoll_wait的task的睡眠队列
    wait_queue_head_t wq;
    // 存在就绪文件句柄的list，该list上的文件句柄事件将会全部上报给应用
    struct list_head rdllist;
    // 存放加入到此epoll句柄的文件句柄的红黑树容器
    struct rb_root rbr;
    // 该epoll结构对应的文件句柄，应用通过它来操作该epoll结构
    struct file *file;
};
```
### 步骤二：为epoll句柄添加文件句柄，注册睡眠entry的回调
这个步骤中其实有两个子步骤：

**1). 添加文件句柄**
将一个文件句柄，比如`socket`添加到`epoll`的`rbr`红黑树容器中，注意，这里的文件句柄最终也是一个包装结构，和`epoll`的结构体类似：

```c
struct epitem {
    // 该字段链接入epoll句柄的红黑树容器
    struct rb_node rbn;
    // 当该文件句柄有事件发生时，该字段链接入“就绪链表”，准备上报给用户态
    struct list_head rdllink;
    // 该字段封装实际的文件，我已经将其展开
    struct epoll_filefd {
        struct file *file;
        int fd;
    } ffd;
    // 反向指向其所属的epoll句柄
    struct eventpoll *ep;
};
```

以上结构实例就是`epi`，将被添加到`epoll`的`rbr`容器中的逻辑如下：
```c
struct eventpoll *ep = 待加入文件句柄所属的epoll句柄;
struct file *tfile = 待加入的文件句柄file结构体;
int fd = 待加入的文件描述符ID;

struct epitem *epi = kmem_cache_alloc(epi_cache, GFP_KERNEL);
INIT_LIST_HEAD(&epi->rdllink);
INIT_LIST_HEAD(&epi->fllink);
INIT_LIST_HEAD(&epi->pwqlist);
epi->ep = ep;
ep_set_ffd(&epi->ffd, tfile, fd);
...
ep_rbtree_insert(ep, epi);
```

**2）注册睡眠entry回调并poll文件句柄**

在第一个子步骤的代码逻辑中，我有一段“…”省略掉了，这部分比较关键，所以我单独抽取了出来作为第二个子步骤。
我们知道，`Linux`内核的`sleep/wakeup`机制非常重要，几乎贯穿了所有的内核子系统，值得注意的是，这里的`sleep/wakeup`依然采用了`OO`的思想，并没有限制睡眠的`entry`一定要是一个`task`，而是将睡眠的`entry`做了一层抽象，即：
```c
struct __wait_queue {
    unsigned int flags;
    // 至于这个private到底是什么，内核并不限制，显然，它可以是task，也可以是别的。
    void *private;
    wait_queue_func_t func;
    struct list_head task_list;
};
```
以上的这个`entry`，最终要睡眠在下面的数据结构实例化的一个链表上：
```c
struct __wait_queue_head {
    spinlock_t lock;
    struct list_head task_list;
};
```
显然，在这里，==一个文件句柄均有自己睡眠队列用于等待自己发生事件的`entry`在没有发生事件时来歇息==，对于`TCP socket`而言，该睡眠队列就是其`sk_wq`，通过以下方式取到：
```c
static inline wait_queue_head_t *sk_sleep(struct sock *sk)
{
    return &rcu_dereference_raw(sk->sk_wq)->wait;
}
```
我们需要一个`entry`将来在发生事件的时候从上述`wait_queue_head_t`中被唤醒，执行特定的操作，即将自己放入到`epoll`句柄的“就绪链表”中。下面的函数可以完成该逻辑的框架：
```c
// 此处的whead就是上面例子中的sk_sleep返回的wait_queue_head_t实例。
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
                 poll_table *pt)
{
    struct epitem *epi = ep_item_from_epqueue(pt);
    struct eppoll_entry *pwq;
    if (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL)) {
        // 发生事件即调用ep_poll_callback回调函数，该回调函数会将自己这个epitem加入到epoll的“就绪链表”中去。
        init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
        // 是否排他唤醒取决于用户的配置，有些IO是希望唤醒所有entry来处理，有些则不必。注意，这里是针对文件句柄IO而言的，并不是针对epoll句柄的。
        if (epi->event.events & EPOLLEXCLUSIVE)
            add_wait_queue_exclusive(whead, &pwq->wait);
        else
            add_wait_queue(whead, &pwq->wait);


    } 
}
```
至于说什么时候调用上面的函数，`Linux`的`poll`机制仍然是采用了分层抽象的思想，即上述函数会作为另一个回调在相关文件句柄的`poll`函数中被调用。即：
```c

static inline unsigned int ep_item_poll(struct epitem *epi, poll_table *pt)
{
    pt->_key = epi->event.events;
    return epi->ffd.file->f_op->poll(epi->ffd.file, pt) & epi->event.events;
}
```

对于`TCP socket`而言，其`file_operations`的`poll`回调即：
```c
unsigned int tcp_poll(struct file *file, struct socket *sock, poll_table *wait)
{
    unsigned int mask;
    struct sock *sk = sock->sk;
    const struct tcp_sock *tp = tcp_sk(sk);
    // 此函数会调用poll_wait->wait._qproc
    // 而wait._qproc就是ep_ptable_queue_proc
    sock_poll_wait(file, sk_sleep(sk), wait);
    ...
}
```

现在，我们可以把子步骤1中的逻辑补全了：
```c
struct eventpoll *ep = 待加入文件句柄所属的epoll句柄;
struct file *tfile = 待加入的文件句柄file结构体;
int fd = 待加入的文件描述符ID;
struct epitem *epi = kmem_cache_alloc(epi_cache, GFP_KERNEL);
INIT_LIST_HEAD(&epi->rdllink);
INIT_LIST_HEAD(&epi->fllink);
INIT_LIST_HEAD(&epi->pwqlist);
epi->ep = ep;
ep_set_ffd(&epi->ffd, tfile, fd);
// 这里会将wait._qproc初始化成ep_ptable_queue_proc
init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
// 这里会调用wait._qproc即ep_ptable_queue_proc，安排entry的回调函数ep_poll_callback，并将entry“睡眠”在socket的sk_wq这个睡眠队列上。
revents = ep_item_poll(epi, &epq.pt);
ep_rbtree_insert(ep, epi);
// 如果刚才的ep_item_poll取出了事件，随即将该item挂入“就绪队列”中，并且wakeup阻塞在epoll_wait系统调用中的task！
if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
    list_add_tail(&epi->rdllink, &ep->rdllist);
    if (waitqueue_active(&ep->wq))
        wake_up_locked(&ep->wq);
}
```

### 步骤三：事件发生，唤醒相关文件句柄睡眠队列的entry，调用其回调
现在我们假设一个`TCP Listen socket`上来了一个连接请求，已经完成了三次握手，内核希望通知`epoll_wait`返回，然后去取`accept`。
内核在`wakeup`这个`socket`的`sk_wq`时，最终会调用到`ep_poll_callback`回调，这个函数我们说了好几次了，现在看看它的真面目：

```c
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
    unsigned long flags;
    struct epitem *epi = ep_item_from_wait(wait);
    struct eventpoll *ep = epi->ep;
    // 这个lock比较关键，操作“就绪链表”相关的，均需要这个lock，以防丢失事件。
    spin_lock_irqsave(&ep->lock, flags);
    // 如果发生的事件我们并不关注，则不处理直接返回即可。
    if (key && !((unsigned long) key & epi->event.events))
        goto out_unlock;


    // 实际将发生事件的epitem加入到“就绪链表”中。
    if (!ep_is_linked(&epi->rdllink)) {
        list_add_tail(&epi->rdllink, &ep->rdllist);
    }
    // 既然“就绪链表”中有了新成员，则唤醒阻塞在epoll_wait系统调用的task去处理。注意，如果本来epi已经在“就绪队列”了，这里依然会唤醒并处理的。
    if (waitqueue_active(&ep->wq)) {
        wake_up_locked(&ep->wq);
    }


out_unlock:
    spin_unlock_irqrestore(&ep->lock, flags);
    ...
}
```
没什么好多说的。现在“就绪链表”已经有`epi`了，接下来就要唤醒`epoll_wait`进程去处理了。

### 步骤四：唤醒epoll睡眠队列的task，搜集并上报数据
这个逻辑主要集中在`ep_poll`函数，精简版如下：
```c
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
           int maxevents, long timeout)
{
    unsigned long flags;
    wait_queue_t wait;


    // 当前没有事件才睡眠
    if (!ep_events_available(ep)) {
        init_waitqueue_entry(&wait, current);
        __add_wait_queue_exclusive(&ep->wq, &wait);
        for (;;) {
            set_current_state(TASK_INTERRUPTIBLE);
            ...// 例行的schedule timeout
        }
        __remove_wait_queue(&ep->wq, &wait);
        set_current_state(TASK_RUNNING);
    }
    // 往用户态上报事件，即那些epoll_wait返回后能获取的事件。
    ep_send_events(ep, events, maxevents);
}
```
其中关键在`ep_send_events`，这个函数实现了非常重要的逻辑，包括`LT`和`ET`的逻辑，我不打算深入去解析这个函数，只是大致说下流程：
```c
ep_scan_ready_list()
{
    // 遍历“就绪链表”
    ready_list_for_each() {
        // 将epi从“就绪链表”删除
        list_del_init(&epi->rdllink);
        // 实际获取具体的事件。
        // 注意，睡眠entry的回调函数只是通知有“事件”，具体需要每一个文件句柄的特定poll回调来获取。
        revents = ep_item_poll(epi, &pt);
        if (revents) {
            if (__put_user(revents, &uevent->events) ||
                __put_user(epi->event.data, &uevent->data)) {
                // 如果没有完成，则将epi重新加回“就绪链表”等待下次。
                list_add(&epi->rdllink, head);
                return eventcnt ? eventcnt : -EFAULT;
            }
            // 如果是LT模式，则无论如何都会将epi重新加回到“就绪链表”，等待下次重新再poll以确认是否仍然有未处理的事件。这也符合“水平触发”的逻辑，即“只要你不处理，我就会一直通知你”。
            if (!(epi->event.events & EPOLLET)) {
                list_add_tail(&epi->rdllink, &ep->rdllist);
            }
        }
    }
    // 如果“就绪链表”上仍有未处理的epi，且有进程阻塞在epoll句柄的睡眠队列，则唤醒它！(这将是LT惊群的根源)
    if (!list_empty(&ep->rdllist)) {
        if (waitqueue_active(&ep->wq))
            wake_up_locked(&ep->wq);
    }
}
```


## 概述
### 惊群现象
多进程/多线程睡眠等待 `共享` 资源，当资源到来时，多个进程/线程被 `无差别` 唤醒，争抢处理资源。

### 惊群影响
惊群导致软件系统工作效率低下：  
（1）部分进程被频繁唤醒却获取资源失败，导致进程上下文频繁切换，系统资源开销大。  
（2）多进程争抢共享资源，有的抢得多，有的抢得少，资源分配不均。


### 惊群原因




惊群现象出现，有的子进程被唤醒但是并没有 `accept` 到链接资源。原因：
两个子进程/线程通过 `epoll_ctl` 添加关注了主进程创建的 `listen-socket`，当该 `listen socket` 没有资源时，子进程都通过 `epoll_wait` 进入了阻塞睡眠状态。也就是子进程分别往 `socket.wq` 等待队列添加了各自的等待事件。
```c
/* include/linux/net.h*/
struct socket {
    ...
    struct socket_wq *wq; /* socket 等待队列。 */
    struct file      *file;
    struct sock      *sk;
    ...
};

区分：
/* epoll 结构对象。*/
struct eventpoll {
    ...
    /* 阻塞在epoll_wait的task的睡眠队列。 */
    wait_queue_head_t wq;
    ...
    /* 存在就绪文件句柄的list，该list上的文件句柄事件将会全部上报给应用 */
    struct list_head rdllist;
    
    /* 存放加入到此epoll句柄的文件句柄的红黑树容器 */
	struct rb_root rbr;
	
	/* 该epoll结构对应的文件句柄，应用通过它来操作该epoll结构 */
	struct file *file;
    ...
};


注：有了 socket.wq 为啥还要有 eventpoll.wq 啊？因为 listen socket 能被多个进程共享，epoll 实例也能被多个进程共享！
```
因为添加的方式是 `add_wait_queue`，而不是 `add_wait_queue_exclusive`，`add_wait_queue` 并没有设置 `WQ_FLAG_EXCLUSIVE` 排它唤醒标识，所以当 `listen socket` 的资源到来时，内核通过 `__wake_up_common` 去唤醒两个子进程去 `accept` 获取资源。
如果只有一个链接资源，那么 `nginx` 的两个子进程被唤醒，当然只有一个子进程能成功，另外一个则无功而返。

## 如何判断发生了惊群？
近期排查了一个问题，epoll惊群的问题，起初我并不认为这是惊群导致，因为从现象上看，只是体现了CPU不均衡。一共fork了20个Server进程，在请求负载中等的时候，**有三四个Server进程呈现出比较高的CPU利用率，其余的Server进程的CPU利用率都是非常低。中断，软中断都是均衡的**；
网卡RSS和CPU之间进行了bind之后依然如故，既然系统层面查不出个所以然，只能从服务的角度来查了。

自上而下的排查首先就想到了`strace`，没想到一下子就暴露了原形：
```bash
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
```
如果仅仅`strace accept`，即加上“-e trace=accept”参数的话，偶尔会有accept成功的现象：
```bash
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, {sa_family=AF_INET, sin_port=htons(39306), sin_addr=inet_addr("172.16.1.202")}, [16]) = 19
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
accept(4, 0x9ecd930, [16])              = -1 EAGAIN (Resource temporarily unavailable)
```

大量的CPU空转，**进一步加大请求负载，CPU空转明显降低，这说明在预期的空转期间，新来的请求降低了空转率…现象明显偏向于这就是惊群导致的**之判断！




## 解决方法
需要围绕两个方面去展开。  
5. 避免共享资源争抢（独占）。  
6. 资源尽量合理分配。

换个角度去思考，如果红包私发，而不是扔进群组里... 这个思路应该是解决惊群问题的关键。

### reuseport

内核解决惊群问题，==目前 nginx 最好的惊群解决方案，基于 linux 内核 `so_reuseport` 端口重用网络特性==。  
7. 每个子进程拥有==独立的 `listen socket` 资源队列（全连接队列、半连接队列）==，避免资源争抢「每个进程/线程的`listen socketfd`不同」；另外，多个队列也提升了并发吞吐。  
8. 新链接通过网络四元组通过哈希分配到各个子进程的 listen socket 资源队列，资源分配相对合理（负载均衡）。

![](attachments/Pasted%20image%2020250601145932.png)

### NGX_EXCLUSIVE_EVENT
内核解决惊群问题，基于 `linux 4.5+` 内核增加的 `epoll` 属性 `EPOLLEXCLUSIVE` 独占资源属性。    
原理非常简单，只唤醒一个睡眠等待的进程处理资源。避免无差别地唤醒多个进程，尽量使得各个进程忙碌起来。  

缺点：  
(1) ==多个进程/线程共享一个 `listen socket` ，争抢共享资源==。  
(2) ==单个资源队列，将会是并发吞吐瓶颈==。

![](attachments/Pasted%20image%2020250601150023.png)



### accept_mutex
应用层解决惊群问题，多个子进程通过应用层抢锁，成功者可以独占`listen socket` 获取资源的权利。    
优点：有效地避免了惊群。  
缺点：  
(1) 因为抢锁时机问题，原来抢到锁的进程下次抢到锁的概率很高，导致有些进程很忙，有些没那么忙，负载不均，资源利用率比较低。  
(2) 一个时间段内，只有一个子进程独占 `listen socket` 的共享资源，无法同时利用多核优势。  
(3) 单个资源队列，将会是并发吞吐瓶颈。

![](attachments/Pasted%20image%2020250601150553.png)

## 方法一：`socket`选项之`reuseport`
`SO_REUSEPORT (reuseport)` 是网络的一个选项设置，它能开启内核功能：网络链接分配 内核负载均衡。该功能允许多个进程/线程 `bind/listen` 相同的 `IP/PORT`，提升了新链接的分配性能。
`nginx` 开启 `reuseport` 功能后，性能有立竿见影的提升，我们结合 `tcp` 协议分析 `nginx` 的 `reuseport` 功能。

`reuseport` 也是内核解决 `惊群问题` 的优秀方案。
(1) 每个进程可以`bind/listen` 相同的 `IP/PORT`，相当于==每个进程/线程拥有独立的 `listen socket` 的完全队列，避免了共享 `listen socket` 的资源争抢==，提升了并发的吞吐。
(2) 内核通过哈希算法，将新链接相对均衡地分配到各个开启了 `reuseport` 属性的进程，所以资源的负载均衡得到解决。

### 介绍（what）
SO_REUSEPORT 是网络的一个选项设置，它允许多个进程/线程 bind/listen 相同的 IP/PORT，在 TCP 的应用中，它是一个新链接分发的（内核）负载均衡功能，它提升了新链接的分配性能（针对 accept ）。

如下所示：[socket(7) — Linux manual page](https://link.zhihu.com/?target=https%3A//man7.org/linux/man-pages/man7/socket.7.html)
```text
Socket options
    The socket options listed below can be set by using setsockopt(2)
    and read with getsockopt(2) with the socket level set to
    SOL_SOCKET for all sockets.  Unless otherwise noted, optval is a
    pointer to an int.
...
    SO_REUSEPORT (since Linux 3.9)
                Permits multiple AF_INET or AF_INET6 sockets to be bound
                to an identical socket address.  This option must be set
                on each socket (including the first socket) prior to
                calling bind(2) on the socket.  To prevent port hijacking,
                all of the processes binding to the same address must have
                the same effective UID.  This option can be employed with
                both TCP and UDP sockets.

                For TCP sockets, this option allows accept(2) load
                distribution in a multi-threaded server to be improved by
                using a distinct listener socket for each thread.  This
                provides improved load distribution as compared to
                traditional techniques such using a single accept(2)ing
                thread that distributes connections, or having multiple
                threads that compete to accept(2) from the same socket.

                For UDP sockets, the use of this option can provide better
                distribution of incoming datagrams to multiple processes
                (or threads) as compared to the traditional technique of
                having multiple processes compete to receive datagrams on
                the same socket.
```
### 作用（why）
`reuseport` 选项主要解决了两个问题：
<1>（A 图）单个 `listen socket` 遇到的性能瓶颈。
<2>（B 图）单个 `listen socket` 多个线程同时 `accept`，但是多个线程资源分配不均。

![](attachments/Pasted%20image%2020250531222817.png)

其实`reuseport`还解决了一个很重要的问题：
在 `tcp` 多线程场景中，（B 图）服务端如果所有新链接只保存在一个 `listen socket` 的 `全链接队列` 中，那么多个线程去这个队列里获取（accept）新的链接，势必会出现多个线程对一个公共资源的争抢，争抢过程中，大量资源的损耗。

而（C 图）有多个 `listener` 共同 `bind/listen` 相同的 `IP/PORT`，也就是说每个进程/线程有一个独立的 `listener`，相当于每个进程/线程独享一个 `listener` 的全链接队列，不需要多个进程/线程竞争某个公共资源，能充分利用多核，减少竞争的资源消耗，效率自然提高了。

### 如何使用（how）
`SO_REUSEPORT` 功能使用，可以通过网络选项进行设置，在 `bind` 前面设置即可，使用比较简单。
```c
int fd, reuse = 1;
fd = socket(PF_INET, SOCK_STREAM, IPPROTO_IP);
setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, (const void *)&reuse, sizeof(int));
```

### 原理
#### 收到第一个SYN包半连接列队的选择
TCP 客户端链接服务端，第一次握手，==服务端被动收到第一次握手 SYN 包，内核就通过哈希算法，将客户端的链接分派到内核半链接队列==，三次握手成功后，再将这个链接从半链接队列移动到某个 listener 的全链接队列中，提供 accept 获取。

![](attachments/Pasted%20image%2020250531223313.png)

服务端被动第一次握手，查找合适的 listener，详看源码（Linux 5.0.1）。
![](attachments/Pasted%20image%2020250531223401.png)

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

### 应用：nginx对于`reuseport`的应用
2013 年 Linux 内核添加了 reuseport 功能后，nginx 在 2015 年，1.9.1 版本也增加对应功能的支持，nginx 开启 reuseport 功能后，性能是原来的 2-3 倍，效果可谓立竿见影！
接下来我们看看 nginx 是如何支持 reuseport 的。
#### 开启 SO_REUSEPORT
修改 nginx 配置，在 nginx.conf 里，listen 关键字后面添加 ‘reuseport’。
```text
# nginx.conf
# vim /usr/local/nginx/conf/nginx.conf

# 启动 4 个子进程。
worker_processes  4;

http {
    ...
    server {
        listen 80 reuseport;
        server_name localhost;
        ...
    }
    ...
}
```

#### 工作流程
启动测试 nginx，1 master / 4 workers，监听 80 端口。

##### 进程
（1）父子进程。
```text
# ps -ef | grep nginx
root      88994   1770  0 14:57 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx
nobody    88995  88994  0 14:57 ?        00:00:00 nginx: worker process      
nobody    88996  88994  0 14:57 ?        00:00:00 nginx: worker process      
nobody    88997  88994  0 14:57 ?        00:00:00 nginx: worker process      
nobody    88998  88994  0 14:57 ?        00:00:00 nginx: worker process      
```

父子进程 LISTEN 80 端口情况。  
因为配置文件设置了 `worker_processes 4` 需要启动 4 个子进程， nginx 进程发现配置文件关键字 listen 后添加了 reuseport 关键字，那么主进程先创建 4 个 socket 并设置 SO_REUSEPORT 选项，然后进行 bind 和 listen。  
当 fork 子进程时，子进程拷贝了父进程的这 4 个 socket，所以你看到每个子进程都有相同 LISTEN 的 socket fd（6，7，8，9）。

```text
# lsof -i:80 | grep nginx 
nginx   88994   root    6u  IPv4 909209      0t0  TCP *:http (LISTEN)
nginx   88994   root    7u  IPv4 909210      0t0  TCP *:http (LISTEN)
nginx   88994   root    8u  IPv4 909211      0t0  TCP *:http (LISTEN)
nginx   88994   root    9u  IPv4 909212      0t0  TCP *:http (LISTEN)
nginx   88995 nobody    6u  IPv4 909209      0t0  TCP *:http (LISTEN)
nginx   88995 nobody    7u  IPv4 909210      0t0  TCP *:http (LISTEN)
nginx   88995 nobody    8u  IPv4 909211      0t0  TCP *:http (LISTEN)
nginx   88995 nobody    9u  IPv4 909212      0t0  TCP *:http (LISTEN)
nginx   88996 nobody    6u  IPv4 909209      0t0  TCP *:http (LISTEN)
nginx   88996 nobody    7u  IPv4 909210      0t0  TCP *:http (LISTEN)
nginx   88996 nobody    8u  IPv4 909211      0t0  TCP *:http (LISTEN)
nginx   88996 nobody    9u  IPv4 909212      0t0  TCP *:http (LISTEN)
nginx   88997 nobody    6u  IPv4 909209      0t0  TCP *:http (LISTEN)
nginx   88997 nobody    7u  IPv4 909210      0t0  TCP *:http (LISTEN)
nginx   88997 nobody    8u  IPv4 909211      0t0  TCP *:http (LISTEN)
nginx   88997 nobody    9u  IPv4 909212      0t0  TCP *:http (LISTEN)
nginx   88998 nobody    6u  IPv4 909209      0t0  TCP *:http (LISTEN)
nginx   88998 nobody    7u  IPv4 909210      0t0  TCP *:http (LISTEN)
nginx   88998 nobody    8u  IPv4 909211      0t0  TCP *:http (LISTEN)
nginx   88998 nobody    9u  IPv4 909212      0t0  TCP *:http (LISTEN)
```

##### 网络初始流程
nginx 是多进程模型，Linux 环境下一般使用 epoll 事件驱动。

![](attachments/Pasted%20image%2020250531224104.png)

strace 监控 nginx 进程的系统调用流程：
```text
# strace -f -s 512 -o /tmp/nginx.log /usr/local/nginx/sbin/nginx
# grep -n -E 'socket\(PF_INET|SO_REUSEPORT|listen|bind|clone|epoll' /tmp/nginx.log

# master
208:88993 socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 6
210:88993 setsockopt(6, SOL_SOCKET, SO_REUSEPORT, [1], 4) = 0
212:88993 bind(6, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
213:88993 listen(6, 511)                    = 0
214:88993 socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 7
216:88993 setsockopt(7, SOL_SOCKET, SO_REUSEPORT, [1], 4) = 0
218:88993 bind(7, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
219:88993 listen(7, 511)                    = 0
220:88993 socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 8
222:88993 setsockopt(8, SOL_SOCKET, SO_REUSEPORT, [1], 4) = 0
224:88993 bind(8, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
225:88993 listen(8, 511)                    = 0
226:88993 socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 9
228:88993 setsockopt(9, SOL_SOCKET, SO_REUSEPORT, [1], 4) = 0
230:88993 bind(9, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
231:88993 listen(9, 511)                    = 0

# master --> fork
250:88993 clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f0d875dfa10) = 88994
274:88994 clone(child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f0d875dfa10) = 88995
305:88994 clone( <unfinished ...>
308:88994 <... clone resumed> child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f0d875dfa10) = 88996
349:88995 epoll_create(512 <unfinished ...>
351:88995 <... epoll_create resumed> )      = 11
413:88995 epoll_ctl(11, EPOLL_CTL_ADD, 6, {EPOLLIN|EPOLLRDHUP, {u32=2270846992, u64=139696082149392}} <unfinished ...>
443:88996 <... epoll_create resumed> )      = 13
445:88994 <... clone resumed> child_stack=0, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f0d875dfa10) = 88998
524:88997 epoll_create(512 <unfinished ...>
526:88997 <... epoll_create resumed> )      = 15
543:88998 epoll_create(512 <unfinished ...>
545:88998 <... epoll_create resumed> )      = 17
564:88997 epoll_ctl(15, EPOLL_CTL_ADD, 8, {EPOLLIN|EPOLLRDHUP, {u32=2270846992, u64=139696082149392}} <unfinished ...>
565:88996 epoll_ctl(13, EPOLL_CTL_ADD, 7, {EPOLLIN|EPOLLRDHUP, {u32=2270846992, u64=139696082149392}} <unfinished ...>
606:88998 epoll_ctl(17, EPOLL_CTL_ADD, 9, {EPOLLIN|EPOLLRDHUP, {u32=2270846992, u64=139696082149392}}) = 0
```

分析总结一下 strace 采集的系统调用日志。
```text
# 如果有 N 个子进程就创建 N 个 socket 并对其设置，绑定地址和监听端口。
N * (socket --> setsockopt SO_REUSEPORT --> bind --> listen) --> 
# fork 子进程
fork -->
# 每个子进程创建 epoll_create 实例进行网络读写工作。
# 注意，listen fd 是父进程创建的，父进程在 fork 子进程时，为每个子进程打上编号了，
# 每个编号的子进程会处理一个 listen fd.
epoll_create --> epoll_ctl EPOLL_CTL_ADD listen fd --> 
epoll_wait --> accept/accept4/read/write
```


源码函数调用层次。
```text
# 函数调用层次关系。
# ------------------------ master ------------------------
main
|-- ngx_init_cycle
    |-- ngx_open_listening_sockets
        |-- socket # 如果有 N 个子进程，那么创建 N 个 socket.
        |-- setsockopt SO_REUSEPORT
        |-- bind
        |-- listen
|-- ngx_master_process_cycle
    |-- ngx_start_worker_processes
        |-- ngx_spawn_process # 每个 fork 出来的子进程，主进程都会传递一个顺序的数字编号进行标识，保存到 ngx_worker
            |-- fork
# ------------------------ worker ------------------------
            |-- ngx_worker_process_cycle
                |-- ngx_worker_process_init
                    |-- ngx_event_process_init
                        |-- ngx_epoll_init
                            |-- epoll_create
                        |-- ngx_epoll_add_event # ngx_add_event
                            |-- epoll_ctl # EPOLL_CTRL_ADD - 对应子进程编号的 listen fd.
                |-- ngx_process_events_and_timers
                    |-- ngx_epoll_process_events # ngx_process_events
                        |-- epoll_wait
                        |-- ngx_event_accept
                            |-- accept/accept4
```

### 小结
（1）每个线程都通过`reuseport`选项，监听相同的`ip:port`, 每个线程的 `listen-fd`的值不同，加入到自身的`epollfd`中。
每个线程的 `listen-fd`对应不同的全连接队列。

## 方法二：应用层的 accept_mutex
### 前言
`nginx` 是 `Master-Worker` 架构。`accept_mutex` 是一个用于控制多个 `Worker` 进程互斥接受新连接的机制。
主进程创建的 `listen socket`，要被 `fork` 出来的子进程共享，但是为了避免多个子进程无差别地消费 `listen socket` 的共享资源，它通过抢锁的方式：使得多个子进程，同一时段，`尽量` 只有一个子进程能获取资源，尽量避免共享资源的争抢问题和尽量使得各个子进程负载均衡。

### 配置
nginx 通过修改配置开启 accept_mutex 功能特性。

```c
# vim /usr/local/nginx/conf/nginx.conf
events {
    ...
    accept_mutex on;
    ...
}
```

### 负载均衡
nginx 子进程通过 ngx_accept_disabled 变量控制抢锁时机。
```c
ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
```
![](attachments/Pasted%20image%2020250601151339.png)

```c
void worker_logic () {
    efd = epoll_create();

    while (1) {
        if (ngx_accept_disabled > 0) {
            ...
            /* 不抢，但是为了避免一直不抢，也要递减它的 disable 程度*/
            ngx_accept_disabled = reduce_disabled();
        } else {
            /* 抢锁 */
            if (ngx_shmtx_trylock()) {
                /* 抢锁成功，epoll 关注 listen_fd 的 POLLIN 事件 */
                if (!is_cur_worker_locked) {
                    epoll_ctl(efd, EPOLL_CTL_ADD, listen_fd, ...);
                    is_cur_worker_locked = true;
                }
            } else {
                if (is_cur_worker_locked) {
                    /* 抢锁失败，epoll 不再关注 listen_fd 事件 */
                    epoll_ctl(efd, EPOLL_CTL_DEL, listen_fd, ...);
                    is_cur_worker_locked = false;
                }
            }
        }

        /* 超时等待链接资源到来 */
        n = epoll_wait(...)
        if (n > 0) {
            if (is_able_to_accept) {
                /* 链接资源到来，取出链接 */
                client_fd = accept();
                /* 每次取出链接后，重新检查 disabled 值 */
                ngx_accept_disabled = check_disabled();
            }
        }

        if (is_cur_worker_locked) {
            ngx_shmtx_unlock();
        }
    }
}
```

```c
/* src/event/ngx_event.c */
ngx_int_t ngx_accept_disabled;   /* 资源分配负载均衡值。 */

/* src/event/ngx_event_accept.c */
void ngx_event_accept(ngx_event_t *ev) {
    ...
    do {
        ...
#if (NGX_HAVE_ACCEPT4)
        if (use_accept4) {
            s = accept4(lc->fd, &sa.sockaddr, &socklen, SOCK_NONBLOCK);
        } else {
            s = accept(lc->fd, &sa.sockaddr, &socklen);
        }
#else
        s = accept(lc->fd, &sa.sockaddr, &socklen);
#endif
        ...
        /* 每次 accept 链接资源后，都检查一下负载均衡数值。*/
        ngx_accept_disabled = ngx_cycle->connection_n / 8
                              - ngx_cycle->free_connection_n;

        c = ngx_get_connection(s, ev->log);
        ...
    } while (ev->available);
}

/* src/event/ngx_event.c */
void ngx_process_events_and_timers(ngx_cycle_t *cycle) {
    ...
    if (ngx_use_accept_mutex) {
        if (ngx_accept_disabled > 0) {
            /* ngx_accept_disabled > 0，说明很少空闲链接了，放弃抢锁。 */
            ngx_accept_disabled--;
        } else {
            /* 通过锁竞争，获得获取资源的权限。 */
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                return;
            }
            ...
        }
    }
    ...
}
```




## 方法三：`epoll`标记之`EPOLLEXCLUSIVE` + 水平触发

### 介绍
`EPOLLEXCLUSIVE` 是 2016 年 `4.5+` 内核新添加的一个 `epoll` 的标识（代码改动较小，详看：[github](https://link.zhihu.com/?target=https%3A//github.com/torvalds/linux/commit/df0108c5da561c66c333bb46bfe3c1fc65905898)。
它降低了多个进程/线程通过 `epoll_ctl` 添加共享 `fd` 引发的惊群概率，使得一个事件发生时，只唤醒一个正在 `epoll_wait` 阻塞等待唤醒的进程/线程（而不是全部唤醒）。
而 `Ngnix` 在 `1.11.3` 之后相应添加了 `NGX_EXCLUSIVE_EVENT` 功能标识（代码改动较小，详看：[github](https://link.zhihu.com/?target=https%3A//github.com/nginx/nginx/commit/5c2dd3913aad5c4bf7d9056e1336025c2703586b)，它使用了 `EPOLLEXCLUSIVE` 特性。
对比 nginx 在应用层的解决方案：[accept_mutex](https://link.zhihu.com/?target=https%3A//wenfh2020.com/2021/10/10/nginx-thundering-herd-accept-mutex/)，NGX_EXCLUSIVE_EVENT 它从内核层面避免惊群问题，它更简洁高效。

### 原理
该功能的工作原和使用相对简单：进程使用 `epoll_ctl` 添加 `listen socket fd` 时，把 `EPOLLEXCLUSIVE` 属性添加进去就可以了。多个进程通过 `epoll_wait` 等待 `listen socket` 事件，当有新链接到来时，内核只唤醒一个等待的进程。

# 参考
```bash
# Marek’s博客中 I/O multiplexing part 系列之三和四 （++++++++）
https://idea.popcount.org/
https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/
https://idea.popcount.org/2017-03-20-epoll-is-fundamentally-broken-22/

# 探索惊群 ① （+++++++++）
https://zhuanlan.zhihu.com/p/435491262
# 探索惊群 ⑥ - nginx - reuseport 
https://zhuanlan.zhihu.com/p/452932307



# 盘点Linux Epoll那些致命弱点
https://news.eeworld.com.cn/mp/yikoulinux/a326954.jspx
https://www.cnblogs.com/codestack/p/16780173.html

## 提高UDP交互性能
https://www.cnblogs.com/charlieroro/p/14096252.html

# 劫起|再谈Linux epoll惊群问题的原因和解决方案
https://mp.weixin.qq.com/s/O_QVxhyS7C3gJluqaLerWQ



```