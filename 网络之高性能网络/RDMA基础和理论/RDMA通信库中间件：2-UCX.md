```table-of-contents
```

# 市面上的统一多通信库
阿里的X-RDMA，更有名气的UCX，华为内部的UMDK等等。
UCX是Mellanox、NVIDIA、IBM的人联合起来搞的，所以相对来说技术支持以及性能表现上确实可能会更好一些。

# UCX 的意义

随着DPU的面市、各类DSA芯片的广泛使用，加之在数据中心和HPC系统中广泛部署的RDMA（InfiniBand 和 RoCE）、GPU、共享内存通信等，**不同的硬件资源对应不同的通信编程接口**（比如TCP/IP是Socket API，RDMA是verbs API ），这给开发人员带来了很大的挑战。如何在这之上**抽象出统一的内存访问语义和统一的通信方式**是一个很有价值课题。


如果应用程序本身使用UCX，则依赖于统一通信基础设施的高级编程在运行时可以更好地相互操作以及应用程序之间的互操作，并且整个软件堆栈的可移植性将大大提高。**当新硬件交付时**，实现少部分的 UCX 接口足以实现对多种编程范例和应用程序的支持。

# UCX 通信接口简介

UCX 的全称是 Unified Communication X。正如它名字所展示的，UCX 旨在提供一个**统一的抽象通信接口，能够适配任何通信设备**，并支持各种应用的需求。

# UCX的架构
下图是 UCX 官方提供的架构图：

![](attachments/Pasted%20image%2020250316141120.png)

从架构图看，UCX 框架由四个主要组件组成：UC服务(UCS)、UC内存(UCM)、UC传输 (UCT) 和UC协议 (UCP)。**这些组件中的每一个都公开一个公共 API，并且可以用作独立的库**。

## UCM
UCS (Unified Communication Memory)， 拦截内存分配和释放事件，该事件由内存注册缓存使用。


## UCS
UCS (Unified Communication Service)是一个服务层，用于**可移植**，常用的数据结构，算法和系统实用程序的集合。

该层公开以下服务：
- 用于访问平台特定功能（原子操作、线程安全等）的抽象；
- 高效内存管理工具（内存池、内存分配器）；
- 常用数据结构（散列、树、列表）;

## UCT

**UCT**(Unified Communication Transport)是一个传输层， UCT 适配了各种通信设备：从单机的共享内存，到常用的 TCP Socket，以及数据中心常用的 RDMA 协议，甚至新兴的 GPU 上的通信，都有很好的支持。

为此，UCT依赖于低级驱动程序, 如 InfiniBand Verbs、Cray 的uGNI、libfabrics 、 共享内存、CUDA 等等。底层的UCT适配了各种通信设备：从单机的共享内存，到常用的TCP/IP Socket,以及数据中心的RDMA协议，甚至GPU上的通信，都有很好的支持。

此外，该层还提供通信上下文管理（基于线程和应用程序级别, 如: ucs_async_context_create, uct_worker_create）以及设备特定存储器（包括加速器中的存储器）的分配和管理的构造。

在通信 API方面，UCT定义了立即（短消息,如: uct_ep_am_short）、缓冲拷贝并发送（bcopy,如: uct_ep_am_bcopy）和零拷贝（zcopy, 如: uct_ep_am_zcopy）通信操作的接口。
short 短操作针对可以就地发布和完成的小消息进行了优化；bcopy操作针对通常通过所谓的弹跳缓冲区发送的中等大小的消息进行了优化。最后，zcopy 操作展示零拷贝内存到内存通信语义。

### 支持的传输协议
-  Infiniband
- Omni-Path
- RoCE
- Cray Gemini and Aries
- CUDA
- Shared Memory(posix,sysv,cma,knem,xpmem)
- TCP/IP

## UCP

UCP(Unified Communication Protocol)则是在UCT不同设备的基础上，封装了更抽象的通信接口，以方便应用使用。
UCP通过使用UCT层公开的接口来实现消息传递 (MPI) 和PGAS 编程模型等较高级别协议。

### 功能
UCP负责以下功能：库的初始化、通信传输的选择、消息分段和多轨通信。

### 通信模型
一般来说，和底层通信设备模型最匹配的接口具有最高的性能，其它不匹配的接口都会有一次软件转换过程。
另一方面，同一种 UCP 接口发送不同大小的消息可能也会使用不同的 UCT 方法。例如在 RDMA 网络中，由于内存注册也有不小的开销，因此对于小消息来说，拷贝到预注册好的缓冲区再发送的性能更高。这些策略默认是由 UCX 自己决定的，用户也可以通过设置环境变量的方式手动修改。

#### stream
- **Stream流**： 是一种 **面向连接的、基于字节流的通信模式**，提供有序（In-order）、可靠的（Reliable）数据传输，类似于 TCP 的字节流语义。
- **特点**：
    - **有序性**：数据按发送顺序到达接收方。
    - **可靠性**：确保数据完整传输，自动处理丢包和错误。
    - **流式接口**：数据被视为连续的字节流，无显式消息边界。
- **适用场景**：
    - 需要连续、有序数据传输的应用（如文件传输、视频流）。
    - 兼容传统 TCP 语义的迁移场景。
- **性能优化**：
    - 适用于中到大消息（>4KB），通过批量化降低协议开销。

#### Tag-Matching
- **Tag流**： 常用于高性能计算 MPI 程序中，是一种 **基于消息标签的通信模式**，支持无序（Out-of-order）但可靠的消息传递，类似于 MPI 的标签匹配机制。每条消息都会附带一个 64 位整数作为 tag，接收方每次可以指定接收哪种 tag 的消息。
- **特点**：
    - **标签匹配**：每个消息附带一个唯一标签（Tag），接收方通过标签选择接收特定消息。
    - **无序性**：消息可以乱序到达，但同一标签的消息按发送顺序处理。
    - **零拷贝支持**：可直接对接 RDMA 硬件，实现高吞吐。
- **适用场景**：
    - 并行计算中的任务分发（如 MPI 点对点通信）。
    - 需要分类处理消息的场景（如请求-响应模型）。
- **性能优化**：
    - 适合小到中等消息（<64KB），利用标签匹配减少同步开销。

##### 应用
在我们的系统中，使用了 UCP Tag 接口并基于此实现了轻量级的 RPC。
在 RPC 场景下，Tag 可以用于区分不同上下文的消息：
每个链接双方首先随机生成一个 tag 作为请求的标识，对于每次请求再随机生成一个 tag 作为回复的标识。
此外 Tag 接口还支持 IO Vector，即将不连续的多个内存段合并成一个消息发送。
这个特性可以用来将用户提供的数据缓冲区和 RPC 请求打包在一起，一定程度上避免数据拷贝。

#### Active Message
- **AM（Active Message）**： 是一种 **事件驱动的通信模式**，允许接收方在消息到达时立即触发预定义的处理函数（回调： callback），无需显式接收操作。

- **特点**：
    - **主动触发**：消息到达后自动调用处理函数，类似远程过程调用（RPC）。
    - **低延迟**：绕过显式接收流程，最小化端到端延迟。
    - **灵活性**：支持自定义消息处理逻辑。
- **适用场景**：
    - 高频小消息通信（如分布式锁、原子操作）。
    - 需要快速响应的任务分发（如事件驱动架构）。
- **性能优化**：
    - 针对极小消息（<1KB），通过回调机制减少上下文切换。


#### RMA / Atomic
RMA（远程内存访问） / Atomic：是对远程直接内存访问（RDMA）的抽象。通信双方可以直接读写远端的内存，但是需要有额外的内存注册过程。

### 服务类型(传输模式)

服务类型(传输模式) 有 UD和 RC。

- **RC**：Reliable Connection，表示基于可靠连接的传输模式（确保消息按序、完整到达）。

#### 服务类型和通信模式组合

**UCX-AM-RC**：


# UCX社区
- [OpenUcx 官方网站](http://www.openucx.org/ "Project Website")
- [OpenUcx 文档](https://openucx.readthedocs.io/en/master/ "ReadTheDocs")
- [Ucx Github](http://www.github.com/openucx/ucx/ "Github")


# UCX的编程模型
## 模型的核心对象
UCX 采用了以异步 IO 为核心的编程模型。其中 UCP 层定义的核心对象有以下四种：

**Context**：
全局资源的上下文，管理所有通信设备。一般每个进程创建一个即可。

**Worker**：
任务的管理调度中心，以轮询方式执行任务。一般每个线程创建一个，会映射为网卡上的一个队列。

**Listener**：
类似 TCP Listener，用来在 worker 之间创建连接。

**Endpoint**：
表示一个已经建立的连接。在此之上提供了各种类型的通信接口。

它们之间的所属关系如下图所示：
![](attachments/Pasted%20image%2020250316214141.png)

## 流程
### 建立连接
UCX 中双方首先要建立连接，拿到一个 Endpoint 之后才能进行通信。建立连接一般要通过 Listener，过程和 TCP 比较类似：

通信双方 A/B 首先建立各自的 Context 和 Worker，其中一方 A 在 Worker 上创建 Listener 监听连接请求，Listener 的地址会绑定到本机的一个端口上。用户需要通过某种方法将这个地址传递给另一方 B。B 拿到地址后在 Worker 上发起 connect 操作，此时 A 会收到新连接请求，它可以选择接受或拒绝。如果接受则需要在 Worker 上 accept 这个请求，将其转换为 Endpoint。之后 B 会收到 A 的回复，connect 操作完成，返回一个 Endpoint。此后双方就可以通过这对 Endpoint 进行通信了。

### 内存注册

对于常规的通信接口，用户可以直接在 Endpoint 上发起请求。但对于 RMA（远程内存访问）操作，需要被访问的一方首先在自己的 Context 上注册内存，同时指定访问权限，获得一个 Mem handle。然后将这个本地 handle 转化为其他节点可以访问的一串 token，称为 remote key（rkey）。最后想办法把 rkey 传给远端。远端拿着这个 rkey 进行远程内存访问操作。

### 异步任务处理
为了发挥最高的性能，整个 UCX 通信接口是全异步的。所谓异步指的是 **IO 操作的执行不会阻塞当前线程**，一次操作的发起和完成是独立的两个步骤。如此一来 CPU 就可以同时发起很多 IO 请求，并且在它们执行的过程中可以做别的事情。

#### 程序如何知道一个异步任务是否完成了？
程序如何知道一个异步任务是否完成了？
常见的有两种做法：**主动轮询，被动通知**。
前者还是需要占用 CPU 资源，所以一般都采用通知机制。
在 C 这种传统过程式语言中，异步完成的通知一般通过 **回调函数（callback）** 实现：每次发起异步操作时，用户都需要传入一个函数指针作为参数。当任务完成时，后台的运行时框架会调用这个函数来通知用户。

####  UCX 中异步接收接口
下面是 UCX 中一个异步接收接口的定义：
```c
ucs_status_ptr_t ucp_tag_recv_nb (  
  ucp_worker_h worker,  
  void ∗ buffer,  
  size_t count,  
  ucp_datatype_t datatype,  
  ucp_tag_t tag,  
  ucp_tag_t tag_mask,  
  ucp_tag_recv_callback_t cb  // <-- 回调函数  
);  
  
// 回调函数接口的定义  
typedef void(∗ ucp_tag_recv_callback_t) (  
  void ∗request,   
  ucs_status_t status,        // 执行结果，错误码  
  ucp_tag_recv_info_t ∗info   // 更多信息，如收到的消息长度等  
);
```

这个接口的语义是：发起一个异步 `Tag-Matching` 接收操作，并立即返回。
当真的收到 `tag` 匹配(`Tag-Matching`)的消息时，UCX 后台会处理这个消息，将其放到用户提供的 `buffer` 中，最后调用用户传入的 `callback`，通知用户任务的执行结果。

**==(1) 上面提到的“后台处理”到底是什么时候执行的？==**

UCX 并不会自己创建后台线程去执行它们，**所有异步任务的后续处理和回调都是在`worker.progress()`函数中，也就是用户主动向 worker 轮询的过程中完成的。**

`worker.progress()`函数的语义是：“看看你手头要处理的事情，有哪些是能做的？尽力去推动一下，做完的通知我。” 换句话说，Worker 正在处理的所有任务组成了一个状态机，progress 函数的作用就是用新事件推动整个状态机的演进。

**==(2) 异步 IO 的最大难点是编程复杂性==**
回到传统的 C 语言，在这里异步 IO 的最大难点是编程复杂性：多个并发任务在同一个线程上交替执行，只能通过回调函数来描述下一步做什么，会使得**原本连续的执行逻辑被打散到多个回调函数中**。本来局部变量就可以维护的状态，到这里就需要额外的结构体来在多个回调函数之间传递。随着异步操作数量的增加，代码的维护难度将会迅速上升。

##### 异步编程的范例
下面的伪代码展示了在 UCX 中如何通过异步回调函数来实现最简单的 echo 服务：
```c
// 这里存放所有需要跨越函数的状态变量  
struct CallbackContext {  
  ucp_endpoint_h ep;  
  void *buf;  
} ctx;  
  
void send_cb(void ∗request, ucs_status_t status) {  
  //【4】发送完毕  
  ucp_request_free(request);  
  exit(0);  
}  
  
void recv_cb(void ∗request, ucs_status_t status, ucp_tag_recv_info_t ∗info) {  
  //【3】收到消息，发起发送请求  
  ucp_tag_send_nb(ctx->ep, ctx->buf, info->length, ..., send_cb);  
  ucp_request_free(request);  
}  
  
int main() {  
  // 省略 UCX 初始化部分  
  //【0】初始化任务状态  
  ctx->ep = ep;  
  ctx->buf = malloc(0x1000);  
  //【1】发起异步接收请求  
  ucp_tag_recv_nb(worker, ctx->buf, 0x1000, ..., recv_cb);  
  //【2】不断轮询，驱动后续任务完成  
    while(true) {  
    ucp_worker_progress(worker);  
  }  
}
```


作为对比，假如 UCX 提供的是同步接口，那么同样的逻辑只需要以下几行就够了：
```c
int main() {  
  // 省略 UCX 初始化部分  
  void *buf = malloc(0x1000);  
  int len;  
  ucp_tag_recv(worker, buf, 0x1000, &len, ...);  
  ucp_tag_send(ep, buf, len, ...);  
  return 0;  
}
```

##### 其他语言的异步编程
面对传统C语言异步编程带来的“回调地狱”，主流编程语言经过了十几年的持续探索，终于殊途同归，纷纷引入了控制异步的终极解决方案—— async-await 协程。它的杀手锏就是能让开发者**用同步的风格编写异步的逻辑**。


# 其他
## 基于epoll实现RDMA异步操作
### epoll
#### 介绍
Linux的`epoll`机制是`Linux`提供的异步编程机制。`epoll`专门用于处理有大量IO操作请求的场景，检查哪些IO操作就绪，使得用户程序不必阻塞在未就绪IO操作上，而只处理就绪IO操作。

#### epoll接口
- `epoll_create`：用于创建epoll实例，返回epoll实例的句柄；
- `epoll_ctl`：用于给epoll实例增加、修改、删除待检查的IO操作事件；
- `epoll_wait`：用于检查每个通过`epoll_ctl`注册到`epoll`实例的`IO`操作，看每个IO操作是否就绪/有期望的事件发生。

#### epoll的IO事件检查规则
epoll有两种检查规则：边沿触发Edge Trigger (ET)，和电平触发Level Trigger (LT)。
边沿触发和电平触发源自信号处理领域。

(1) 边沿触发指信号一发生变化就触发事件，比如从0变到1就触发事件、或者从1到0就触发事件；
(2) 电平触发指只要信号的状态处于特定状态就触发事件，比如高电平就一直触发事件，而低电平不触发事件。

![](attachments/Pasted%20image%2020250316230348.png)

对应到epoll：

（1）电平触发指的是，只要IO操作处于特定的状态，就会一直通知用户程序。比如当socket有数据可读时，用户程序调用epoll_wait查询到该socket有收到数据，只要用户程序没有把该socket上次收到的数据读完，每次调用epoll_wait都会通知用户程序该socket有数据可读；即当socket处于有数据可读的状态，就会一直通知用户程序。

（2）epoll的边沿触发指的是epoll只会在IO操作的特定事件发生后通知一次。比如socket有收到数据，用户程序epoll_wait查询到该socket有数据可读，不管用户程序有没有读完该socket这次收到的数据，用户程序下次调用epoll_wait都不会再通知该socket有数据可读，除非这个socket再次收到了新的数据；即仅当socket每次收到新数据才通知用户程序，并不关心socket当前是否有数据可读。

### epoll 和 RDMA
`epoll`特别适合有大量IO操作的场景。
比如RDMA的场景，每个RDMA节点同时有很多队列，用于大量传输数据，那么就可以用epoll来查询每个CQ，及时获得RDMA消息的发送和接收情况，同时避免同步方式查询CQ的缺点，要么用户程序消耗大量CPU资源，要么用户程序被阻塞。

注：也可以使用epoll来实现其他RDMA的操作。

### RDMA轮询方式读取CQE

RDMA轮询方式读取CQ非常简单，就是不停调用ibv_poll_cq来读取CQ里的CQE。这种方式能够最快获得新的CQE，直接用户程序轮询CQ，而且也不需要内核参与，但是缺点也很明显，用户程序轮询消耗大量CPU资源。

```c
loop {  
    // 尝试读取一个CQE  
    poll_result = ibv_poll_cq(cq, 1, &mut cqe);  
    if poll_result != 0 {  
        // 处理CQE  
    }  
}
```

### RDMA完成事件通道(cq channel)方式读取CQE
**（1）流程**
`RDMA`用完成事件通道读取`CQE`的流程如下：

(1) 用户程序通过调用`ibv_create_comp_channel`创建完成事件通道；
(2) 接着在调用`ibv_create_cq`创建`CQ`时关联该完成事件通道；
(3) 再通过调用`ibv_req_notify_cq`来告诉CQ当有新的`CQE`产生时从完成事件通道（cq channel）来通知用户程序；
(4) 然后**通过调用`ibv_get_cq_event`查询该完成事件通道，没有新的`CQE`时阻塞**，有新的`CQE`时返回；
(5) 接下来**用户程序从`ibv_get_cq_event`返回之后，还要再调用`ibv_poll_cq`从CQ里读取新的`CQE`，此时调用`ibv_poll_cq`一次就好，不需要轮询**。

**（2）代码示例**
RDMA用完成事件通道读取CQE的代码示例如下：
```c
// 创建完成事件通道  
let completion_event_channel = ibv_create_comp_channel(...);  
// 创建完成队列，并关联完成事件通道  
let cq = ibv_create_cq(completion_event_channel, ...);  
  
loop {  
    // 设置CQ从完成事件通道来通知下一个新CQE的产生  
    ibv_req_notify_cq(cq, ...);  
    // 通过完成事件通道查询CQ，有新的CQE就返回，没有新的CQE则阻塞  
    ibv_get_cq_event(completion_event_channel, &mut cq, ...);  
    // 读取一个CQE  
    poll_result = ibv_poll_cq(cq, 1, &mut cqe);  
    if poll_result != 0 {  
        // 处理CQE  
    }  
    // 确认一个CQE  
    ibv_ack_cq_events(cq, 1);  
}
```

**（3）原理**
用RDMA完成事件通道的方式来读取CQE，**本质是RDMA通过内核来通知用户程序CQ里有新的CQE**。
事件队列是通过一个设备文件，`/dev/infiniband/uverbs0`（如果有多个RDMA网卡，则每个网卡对应一个设备文件，序号从0开始递增），来让内核通过该设备文件通知用户程序有事件发生。用户程序调用`ibv_create_comp_channel`创建完成事件通道，其实就是打开上述设备文件；用户程序调用`ibv_get_cq_event`查询该完成事件通道，其实就是读取打开的设备文件。但是这个设备文件只用于做事件通知，通知用户程序有新的CQE可读，但并不能通过该设备文件读取CQE，用户程序还要是调用`ibv_poll_cq`来从CQ读取CQE。

**（4）完成事件通道方式和轮询方式对比**
用完成事件通道的方式来读取CQE，比轮询的方法要节省CPU资源，但是调用`ibv_get_cq_event`读取完成事件通道会发生阻塞，进而影响用户程序性能。

### 基于epoll异步读取CQE
#### 思路
上面提到用RDMA完成事件通道的方式来读取CQE，本质是用户程序通过事件队列打开`/dev/infiniband/uverbs0`设备文件，并读取内核产生的关于新CQE的事件通知。从完成事件通道`ibv_comp_channel`的定义可以看出，里面包含了一个Linux文件描述符，指向打开的设备文件：
```c
pub struct ibv_comp_channel {  
    ...  
    pub fd: RawFd,  
    ...  
}
```
于是可以借助epoll机制来检查该设备文件是否有新的事件产生，避免用户程序调用`ibv_get_cq_event`读取完成事件通道时（即读取该设备文件时）发生阻塞。

#### 实现方式

（1）首先，用fcntl来修改完成事件通道里设备文件描述符的IO方式为非阻塞：
```c
// 创建完成事件通道  
let completion_event_channel = ibv_create_comp_channel(...);  
// 创建完成队列，并关联完成事件通道  
let cq = ibv_create_cq(completion_event_channel, ...);  
// 获取设备文件描述符当前打开方式  
let flags =  
    libc::fcntl((*completion_event_channel).fd, libc::F_GETFL);  
// 为设备文件描述符增加非阻塞IO方式  
libc::fcntl(  
    (*completion_event_channel).fd,  
    libc::F_SETFL,  
    flags | libc::O_NONBLOCK  
);
```

(2) 接着，创建epoll实例，并把要检查的事件注册给epoll实例：
```c
use nix::sys::epoll;  
  
// 创建epoll实例  
let epoll_fd = epoll::epoll_create()?;  
// 完成事件通道里的设备文件描述符  
let channel_dev_fd = (*completion_event_channel).fd;  
// 创建epoll事件实例，并关联设备文件描述符，  
// 当该设备文件有新数据可读时，用边沿触发的方式通知用户程序  
let mut epoll_ev = epoll::EpollEvent::new(  
    epoll::EpollFlags::EPOLLIN | epoll::EpollFlags::EPOLLET,  
    channel_dev_fd  
);  
// 把创建好的epoll事件实例，注册到之前创建的epoll实例  
epoll::epoll_ctl(  
    epoll_fd,  
    epoll::EpollOp::EpollCtlAdd,  
    channel_dev_fd,  
    &mut epoll_ev,  
)
```
上面代码有两个注意的地方：
- EPOLLIN指的是要检查设备文件是否有新数据/事件可读；
- EPOLLET指的是epoll用边沿触发的方式来通知。

(3) 然后，循环调用epoll_wait检查设备文件是否有新数据可读，有新数据可读说明有新的CQE产生，再调用ibv_poll_cq来读取CQE：
```c
let timeout_ms = 10;  
// 创建用于epoll_wait检查的事件列表  
let mut event_list = [epoll_ev];  
loop {  
    // 设置CQ从完成事件通道(cq channel)来通知用户下一个新CQE的产生  
    ibv_req_notify_cq(cq, ...);  
    // 调用epoll_wait检查是否有期望的事件发生  
    let nfds = epoll::epoll_wait(epoll_fd, &mut event_list, timeout_ms)?;  
    // 有期望的事件发生  
    if nfds > 0 {  
        // 通过完成事件通道查询CQ，有新的CQE就返回，没有新的CQE则阻塞  
        ibv_get_cq_event(completion_event_channel, &mut cq, ...);  
        // 循环读取CQE，直到CQ读空  
        loop {  
            // 读取一个CQE  
            poll_result = ibv_poll_cq(cq, 1, &mut cqe);  
            if poll_result != 0 {  
                // 处理CQE  
                ...  
                // 确认一个CQE  
                ibv_ack_cq_events(cq, 1);  
            } else {  
                break;  
            }  
        }  
    }  
}
```
上面代码有个要注意的地方，因为epoll是用**边沿触发**，所以每次有新CQE产生时，都要调用`ibv_poll_cq`把CQ队列读空。

考虑如下场景，同时有多个新的CQE产生，但是epoll边沿触发只通知一次，如果用户程序收到通知后没有读空CQ，那epoll也不会再产生新的通知，除非再有新的CQE产生，epoll才会再次通知用户程序。

#### 扩展
本实例用epoll机制实现RDMA异步读取CQE的例子，展示了如何实现RDMA的异步操作。RDMA还有类似的操作，都可以基于epoll机制实现异步操作。

# 参考
```bash
# 统一抽象通信接口——UCX(Unified Communication X)
https://mp.weixin.qq.com/s/o2L0VUWO1xeAgKHH0pH_qA

# UCX 演示
演讲: https://ucfconsortium.org/presentations/
视频链接: https://www.youtube.com/watch?v=Yv9nW0Qyjys&t=2713s

# Rust实现RDMA异步编程（二）：async Rust封装UCX通信库
https://mp.weixin.qq.com/s/fvhkXhJ0HcA-TNPDgA5VjQ

# Rust实现RDMA异步编程（一）：基于epoll实现RDMA异步操作
https://mp.weixin.qq.com/s/mIlIGcyhGwQh9wMUYUNz6A

# async-rdma：使高性能网络应用开发更简单
https://mp.weixin.qq.com/s/XvIWA8AA7LIJIVTfKKzQuA

# 用async_rdma库加速 Datenlord KVCache 模块
https://mp.weixin.qq.com/s/M6XTfaU71nIxtLFFq_b6eQ
```