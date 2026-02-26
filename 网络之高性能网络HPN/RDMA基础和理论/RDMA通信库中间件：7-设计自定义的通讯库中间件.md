```table-of-contents
```
# 为什么要实现基于RDMA的通讯库中间件
## 服务器的网络传输的瓶颈从带宽到CPU的转变
之前服务器的网络带宽不足的时候，比如服务器的网卡10G,25G的IDC中，服务器的CPU往往是过剩的。
但是到了100G网卡以及200G/400G网卡之后，相当于带宽，这个时候服务器的CPU往往是瓶颈了。
尤其是AI快速发展对于高性能网络的需求，需要更大的带宽/更低的时延，IDC中的服务器的网络带宽都进行了升级，这个时候通算场景(基于CPU)的业务，相比于网络带宽，CPU成为了瓶颈。这个时候就特别需要通过各种途径减少CPU的消耗。

### 减少CPU使用的方法
#### 网络侧
使用RDMA、用户态协议栈，可以通过零拷贝传输、bypass kernel (减少系统调用和上下文切换)来减少CPU的使用。
另外，RDMA还可以通过`offload CPU`的方式减少CPU的损耗(即RDMA可以将协议栈放在了网卡上「比如报文的封装/解封装都是在网卡上」) 。  


#### 业务侧

业务层：比如`rpc`通信，序列化和反序列化可以使用其他方式，比如RAW方式或者KV的方式取代PB，甚至不进行序列化和反序列化来达到节省CPU的效果。
之前有听`RPC`的同学，通TOR下的RPC通信，序列化和反序列化可能消耗了一个`RPC`请求/响应一半的时延。

之前社科应该使用了Raw(没有序列化和反序列化)+rdma，相对于基于内核TCP的rpc性能提升了70%；然后百度开源的底层基于RDMA的brpc相对于基于内核tcp的brpc性能应该是提升了17%。

### 其他
统一封装的通讯库中间件(比如：KUCL)，可以达到网络侧和业务侧分层解耦，业务层关注业务侧的逻辑，不需要关注底层的网络传输协议。
底层的网络传送协议可以随意的切换，业务侧代码都不需要改动。另外就是可以为业务侧屏蔽底层(网络层)各种复杂的概念、编程接口等等。  

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

# 设计思想
## 零拷贝进行发送和接收
### 背景
#### 基于linux内核的应用的发送和接收
应用程序通过 系统调用 `read/write` 进行数据的接收和发送，这个就涉及到数据的拷贝，在用户态和内核态进行数据拷贝。
```c
ssize_t read(int fd, void *buf, size_t count);

ssize_t write(int fd, const void *buf, size_t count);
```
比如：`write` 进行数据的发送，就涉及到将数据从用户态拷贝到内核态中的发送缓冲区；
成功返回之后，业务就可以将用户态的这块数据释放了，而不必关心内核什么时候以及是否将发送缓冲区中的数据发送出去。

#### 零拷贝的问题
##### 发送数据
拿发送举例，通过注册接口，提前注册一大块内存到RDMA/用户态协议，然后应用每次发送数据的内存，都是从这块已经注册过的内存空间申请的。

如果需要发送零拷贝，应用程序在调用 `write_zw` 接口成功返回之后「==write_zw 必须是异步非阻塞的==」，此时数据是否发送出去，其实是不知道的。
只有数据成功发送出去之后（对于可靠连接，就是发送出去，并且收到对方的ACK； 对于非可靠连接，本端发送出去之后，就可以进行释放）才可以将这块内存进行释放。

由于 应用程序在调用 `write_zw` 接口成功返回之后，并不知道数据是否发送出去。至于数据是否发送出去，只有底层的通讯库才知道，因此这块内存的释放，只能是通过底层通讯库进行释放，那么就需要在应用程序在调用 `write_zw` 接口的时候，将内存释放的函数作为参数传入。

==总结：发送接口，为了达到零拷贝，需要应用层通过`write`接口向通讯库传递释放函数，具体的释放是可以感知到发生完成的底层通讯库来进行调用进行释放的。==

##### 接收数据
对于接收数据同样如此，提前注册一大块内存，RDMA/用户态协议栈收到数据之后，DMA将数据拷贝到某块内存空间，然后给业务上报`In`事件(即： 可读事件)；
业务调用 `read_zc` 接口进行数据的读取。这个时候的接口，就不能是 类似于 `ssize_t read(int fd, void *buf, size_t count)` 这种接口了。
而是需要 类似于 `readv` 的这种接口，`iovec`中得到要读取的数据的地址，以及长度；另外需要在 `iovec` 中封装释放函数：即业务读取数据结束，并且处理完毕，后续不再使用这块数据之后，可以调用释放函数来释放数据。

```c
#include <sys/uio.h>

ssize_t readv(int fd, const struct iovec *iov, int iovcnt);

ssize_t writev(int fd, const struct iovec *iov, int iovcnt);

ssize_t preadv(int fd, const struct iovec *iov, int iovcnt,
			  off_t offset);

ssize_t pwritev(int fd, const struct iovec *iov, int iovcnt,
			   off_t offset);

struct iovec {
   void  *iov_base;    /* Starting address */
   size_t iov_len;     /* Number of bytes to transfer */
};
```

为什么基于Linux内核的 `read` 不需要释放 函数呢？
因为 基于Linux内核的 `read` 涉及到从内核态到用户态的数据拷贝，在`read`结束之后，内核态中这块内存其实是可以释放了的。

==总结：接收接口，为了达到零拷贝，通讯库需要通过`read`接口向应用层传递释放函数，具体的释放是业务不再使用这块内存的数据之后，主动调用释放函数进行释放的。==

### 发送(write)接口

### 接收(read)接口

### 和基于内核的发送接收对比

（1）数据路径对比
```bash
(1) 内核 read/write：
用户态buf ← memcpy ← 内核缓冲区 ← DMA ← NIC
用户态buf → memcpy → 内核缓冲区 → DMA → NIC

(2) 用户态零拷贝：
用户态 DMA 缓冲区 ← DMA ← NIC
用户态 DMA 缓冲区 → DMA → NIC
无任何 memcpy
```

## 回调函数
### 什么是回调函数

**回调函数本质上就是：把一个函数当作参数传给另一个函数，在合适的时机再被"回头调用"。**

那我们来个生活中的例子：
想象你去火锅店吃饭，但发现需要排队。有两种方式等位：
1. **傻等法**：站在门口一直盯着前台，不停问"到我了吗？到我了吗？"
2. **回调法**：拿个小 buzzer（呼叫器），该干嘛干嘛去，等轮到你时，buzzer 会自动震动提醒你。

**回调函数的核心思想是："控制反转"（IoC）**—— 把"何时执行"的控制权交给了别人，而不是自己一直轮询检查。

### 为什么需要回调函数

在深入代码前，我们先搞清楚为啥需要这玩意儿？回调函数解决了哪些问题？

1. **解耦**：调用者不需要知道被调用者的具体实现
2. **异步处理**：可以在事件发生时才执行相应代码，不需要一直等待
3. **提高扩展性**：同一个函数可以接受不同的回调函数，实现不同的功能
4. **实现事件驱动**：GUI编程、网络编程等领域的基础；
5. **延迟执行** - 在特定条件满足时才执行代码
6. **控制反转（IoC）** - 把"何时执行"的控制权交给调用者


最后用一句话总结回调函数：**把"怎么做(什么时候做)"的权力交给别人，自己只负责"做什么"的一种编程技巧。**

### 回调函数的基本结构：举例说明
```c
// 1. 定义回调函数类型（函数指针类型）
typedef void (*CallbackFunc)(int);

// 2. 实际的回调函数
void onTaskCompleted(int result) {
    printf("哇！任务完成了！结果是: %d\n", result);
}

// 3. 接收回调函数的函数
void doSomethingAsync(CallbackFunc callback) {
    printf("开始执行任务...\n");
    // 假设这里是一些耗时操作
    int result = 42;
    printf("任务执行完毕，准备调用回调函数...\n");
    // 操作完成，调用回调函数
    callback(result);
}

// 4. 主函数
int main() {
    // 把回调函数传递过去
    doSomethingAsync(onTaskCompleted);
    return 0;
}

```

上面的代码中：

7. `CallbackFunc` 是一个函数指针类型，它定义了回调函数的签名
8. `onTaskCompleted` 是实际的回调函数，它会在任务完成时被调用
9. `doSomethingAsync` 是接收回调函数的函数，它在完成任务后会调用传入的回调函数
10. 在 `main` 函数中，我们将 `onTaskCompleted` 作为参数传给了 `doSomethingAsync`

这就是回调函数的基本结构！**核心就是把函数的地址当作参数传递，然后在合适的时机调用它；当然也可以定义一个结构体，其中含有函数指针成员，在函数之间传递该结构体**。


### 回调函数的陷阱与最佳实践

#### 生命周期问题：回调函数中引用了已经被销毁的对象。
```c
void dangerousCallback() {
    char* buffer = new char[100];
    
    // 注册一个在未来执行的回调函数
    registerCallback([buffer]() {
        // 危险！此时buffer可能已经被删除
        strcpy(buffer, "Hello");
    });
    
    // buffer在这里被删除
    delete[] buffer;
}

```

#### double-free 问题



## 异步非阻塞(异步编程)
### IO多路复用

### 异步非阻塞+回调

## 线程模式
### Polling 和 Event模式
`Polling` 模式和 `Event`模式是 `per-thread`的，即一个进程中，允许部分线程是`polling`模式，部分线程是`event`模式。

## 线程模型
### RTC 和 Pipeline模型


## 分层
比如：
（1）抽象层：类似于DPDK中的 EAL层，向上负责直接对用户暴漏接口，向下则是作为多个传输协议层的抽象。
（2）传输层：
（2.1）传输层之RDMA传输协议
RDMA层，又分为RC/UD服务类型，以及自子实现的VRC（线程和线程之间只有一个RC类型的QP）/VQP（进程和进程之间只有一个RC类型QP）等服务类型。
（2.2）传输层之基于DPDK的用户态协议栈(U-TCP)传输协议
（2.3）传输层之基于内核协议栈(Kernel-TCP)传输协议

```bash
						抽象层
						  |
						  |
						  |
传输层：         RDMA  Or  U-TCP  or K-TCP
		         |
		         |
		         |
服务类型：    RC UD VRC VQP

```
### 分层设计在多线程大型程序中的应用
分层设计（Layered Architecture）是将复杂的系统拆分成几个逻辑上独立的层级；
==每一层只向上提供服务，并只向下依赖或使用服务==。

|**层次 (Layer)**|**作用和职责**|**依赖关系**|
|---|---|---|
|**应用逻辑层 (Application Logic Layer)**|实现业务的**核心逻辑**。例如，用户请求处理、状态机管理、业务规则校验。|依赖于服务层、数据层。|
|**服务层 (Service/Concurrency Layer)**|管理并发和线程。例如，**线程池**、事件驱动（Epoll/Reactor）、任务调度、异步 I/O 管理。|依赖于数据层、驱动层。|
|**数据层 (Data/Persistence Layer)**|负责数据的存储和访问。例如，缓存管理、数据库连接池、内存数据结构（哈希表、队列）。|依赖于驱动层。|
|**驱动层 (Driver/Utility Layer)**|封装操作系统底层接口和通用工具。例如，Socket/文件 I/O、内存管理、日志打印、网络协议封装。|不依赖其他层。|


### 分层设计的优点

- **隔离性 (Isolation)：** 任何一层的修改，只需要保证接口不变，就不会影响到上层。
    
- **可测试性 (Testability)：** 可以独立测试每一层。
    
- **可维护性 (Maintainability)：** 职责清晰，代码的可读性，定位和修复 Bug 更容易。


### 分层编码

（1）上层使用下层的服务；下层的数据结构/函数，对于上层是不可见的。
下层通常将各种操作封装为`ops`；
上层可以基于下层的类型，选择对应的  `ops`进行操作，达到上层不需要知道下层的数据结构以及具体的函数实现。

（2）一般而言，上层不能直接把结构暴露给下层 → 解耦需要回调函数！
     如果过于复杂，特殊情况下，下层可以使用上层抽象的数据结构，函数等。

```bash
(1) AI agent:  Chatgpt, Gemini

linux C多线程的大型程序中，分层设计。回调函数的妙用，使用场景，作用。

(2) google搜索：
linux C多线程的大型程序中，分层设计。回调函数的妙用，使用场景，作用。
```



### 分层中的回调函数

设计大型 Linux C 多线程程序时，**分层设计**和**回调函数**是实现高内聚、低耦合、可扩展性和高并发性的两个关键技术。
在这种架构中，**回调函数（callback functions）** 扮演非常重要的角色，常用于==解耦、异步事件处理、跨层通信==等。


####  回调函数的作用

在多线程的分层设计中，**回调函数 (Callback Function)** 是实现层间`反向控制`和`解耦`的利器。
它允许底层组件在完成任务后，通知上层组件执行特定的操作，而底层组件本身并不知道上层组件的具体业务逻辑。

|**作用**|**描述**|**为什么使用回调**|
|---|---|---|
|**解耦 (Decoupling)**|底层模块只定义一个**抽象的接口**（即函数指针），不依赖上层模块的具体实现。|底层代码（如线程池）可以保持通用性，适应各种业务逻辑。|
|**异步通知 (Async Notification)**|允许底层任务在**完成时**或**发生事件时**，主动通知上层执行后续操作。|避免上层进行阻塞等待或频繁轮询，提高系统效率。|
|**策略注入 (Strategy Injection)**|允许上层（应用层）将**具体业务逻辑**作为参数（函数指针）传递给底层（服务层）。|实现了**控制反转 (IoC)**，让底层执行机制统一，而上层业务逻辑灵活多变。|

#### 


##### 跨线程通信使用回调的场景

回调函数在不同线程间进行通信和协作时，是一种非常强大的机制，尤其适用于**异步操作完成通知**和**将任务结果返回给发起线程**的场景。
这种用法是大型多线程程序中实现**高效解耦**和**避免阻塞**的关键。

回调函数跨线程使用的核心目的是在**工作线程**完成任务后，将结果或状态报告给**主线程（或发起线程）**，而无需主线程进行忙等待 (Busy Waiting) 或周期性轮询。

##### 场景一：异步任务完成通知 (Worker → Main/Caller)

- **场景描述：** 主线程（或I/O线程）将一个耗时的任务提交给线程池中的**工作线程**去执行（例如，数据库查询、复杂的计算、文件I/O）。主线程不需要等待结果，可以继续处理其他请求。当工作线程完成任务后，需要将结果返回给主线程。
    
- **回调作用：**
    
    1. **解耦：** 工作线程（底层执行者）**无需知道**上层（主线程/发起者）的具体业务逻辑。它只知道在任务完成时，应该调用哪个函数。
        
    2. **非阻塞通知：** 主线程无需阻塞。工作线程将结果打包，调用主线程提供的回调函数，**在主线程/I/O线程的上下文中**执行后续处理（如更新UI、发送网络响应等）。
        
- **实现细节：**
    
    - 主线程提交任务时，传入**任务数据**和**回调函数指针**。
        
    - 工作线程完成任务，将结果和回调函数指针一起放入一个**线程安全的结果队列**。
        
    - 主线程或专门的通知线程不断轮询这个结果队列，取出结果后，**在自己的线程上下文**中调用回调函数。


##### 场景二：资源释放或清理通知

- **场景描述：** 当一个共享资源（如 Socket 连接、内存缓冲区）被工作线程使用完毕后，需要通知管理该资源的线程（如连接池线程）进行回收和清理。
    
- **回调作用：**
    
    1. **统一管理：** 确保资源释放逻辑集中在**资源管理者所在的线程**中执行，避免多个工作线程同时操作资源池，简化同步控制。
        
    2. **延迟执行：** 只有当工作线程确实完成任务时，才触发释放操作。


比如：
`conn` 这样的结构 都是在`worker`中进行创建，读写以及释放的；
但是在 `ctrl` 线程中 会进行查询(只进行读)，为了防止`worker`中释放`conn`的过程中，`ctrl`在进行读同一个`conn`；
可以在`worker`和`ctrl`中传递消息：
```bash
(1) `worker`中要释放`conn`之前，给`ctrl`发送消息；
(2) ctrl 收到消息之后，将`conn`从链表中摘除，保证后续不会再读取到这个conn；
然后ctrl 给 worker 发送消息，告诉 worker 可以释放这个conn了。
(3) worker 收到消息之后，知道ctrl不会再读取这个conn了，就可以释放了。
```


#### 分层中使用回调的场景

##### 场景一：服务层（线程池）与应用逻辑层

这是最典型的应用场景，实现了**任务的提交与执行分离**。

- **场景描述：** 服务层管理一个线程池，应用逻辑层将一个需要异步执行的任务提交给线程池。
    
- **实现方式：**
    
    1. 线程池 API (`threadpool_add_task`) 接收两个参数：**任务函数指针**和**任务参数**。
        
    2. 应用逻辑层将**自己的业务处理函数**（例如 `handle_client_request`）作为回调函数传入。
        
    3. 线程池中的工作线程取出任务后，直接调用这个回调函数执行业务逻辑。

```c
// 服务层 (线程池) 定义的通用接口
typedef void (*TaskCallback)(void *arg);

void threadpool_add_task(TaskCallback task_func, void *arg); 
// ...
// 线程内部执行：
// task_func(arg);

```

##### 场景二：驱动层（异步 I/O）与服务层

用于处理 Socket 等 I/O 事件的**非阻塞通知**。

- **场景描述：** 驱动层（如 Epoll 或 Libevent 封装）监听到 Socket 数据可读。它需要通知服务层处理数据。
    
- **实现方式：**
    
    1. 服务层调用 `driver_register_event` 时，传入一个**事件处理函数**（如 `service_handle_read`）作为回调。
        
    2. 当 Epoll 发现事件就绪时，驱动层会查找并调用对应的回调函数。

```c
// 驱动层 (Epoll封装) 
typedef void (*IoHandler)(int fd, uint32_t events, void *ctx);

int driver_register_event(int fd, uint32_t events, IoHandler handler, void *ctx);
// ...
// Epoll 循环发现事件后：
// handler(fd, events, ctx);
```


## 状态机

## 负载均衡和容灾
### 负载均衡
通过RDMA QP的多路径：
即sip和dip中间创建多个QP，由于每个QP的udp的五元组应该是不一样的，那么在以太网中基于五元组进行`ECMP`的`Hash`时，就会选择不一样的路径。

### 容灾
#### 基于QP的多路径容灾
通过RDMA QP的多路径：如果某个链路存在问题，可以切换到其他的路径上。
#### 传输层多传输协议之间的容灾
比如：RDMA传输协议如果存在问题，就`fallback`切换到用户态TCP，最后的内核TCP作为兜底。

#### 小结
基于QP的多路径容灾 优先级高，多传输协议之间的容灾 优先级低。
只有当基于QP的多路径容灾 依然无法满足时，才会在传输层进行`fallback`到其他的传输协议。

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

![](attachments/image2.png)

# 参考
```bash

```