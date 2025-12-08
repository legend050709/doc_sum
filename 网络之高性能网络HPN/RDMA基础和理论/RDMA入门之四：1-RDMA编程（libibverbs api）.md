```table-of-contents
```
# RDMA编程模型
RDMA的编程模型是基于消息的方式来实现网络传输，并且用队列来管理待发送的消息和接收到的消息。

# RDMA编程相关操作

RDMA 采用了基于 MQ Channel（消息队列通道）的 P2P 全双工通信模型，定义了 2 大类型的队列：

**(1) WQ（Work Queue）**：
App 要收/发数据，就会放置一个 WR（Work Request）到 WQ 作为 WQE（WQ Element）。WQE 是 RNIC 硬件执行任务单元，包含了软件需要硬件执行的动作。RNIC 会获取到 WQE 进行处理。

**(2) CQ（Complete Queue）**：
RNIC 每处理完一个 WQE 之后，就会写入一个 CQE 到 CQ，App 从 CQE 中确认一个 WC（Worker Completion）。

RDMA的网络传输相关操作基本上都是跟队列相关的操作：
比如把要发送的消息放入发送队列，消息发送完成后在完成队列里放一个发送完成消息，以供用户程序查询消息发送状态；再比如接收队列里收到消息，也要在完成队列里放个接收完成消息，以供用户程序查询有新消息要处理。

![](attachments/Pasted%20image%2020250316225230.png)

## RDMA 队列
### 队列分类
由上面的描述可以看出RDMA的队列分为几种：发送队列Send Queue (SQ)，接收队列Receive Queue(RQ)，和完成队列Completion Queue (CQ)。
其中SQ和RQ统称工作队列Work Queue (WQ)，也称为Queue Pair (QP)。

### 操作接口
此外，RDMA提供了两个接口，`ibv_post_send`和`ibv_post_recv`，由用户程序调用，分别用于发送消息和接收消息。

（1）用户程序调用`ibv_post_send`把发送请求`Send Request` (SR 或 WR)插入SQ，成为发送队列的一个新的元素`Send Queue Element (SQE 或 WQE)`；

（2）用户程序调用`ibv_post_recv`把接收请求`Receive Request` (RR 或 WR)插入RQ，成为接收队列的一个新元素`Receive Queue Element (RQE 或 WQE)`。

SQE和RQE也统称工作队列元素Work Queue Element (WQE)。

### 操作类型
两种操作类型：  单端操作(SEND & RECV) 和 双端操作(READ & WRITE)。

#### 单端操作(SEND & RECV)
- 发送、接收数据，一端Send前需要对端先Receive
- 对端CPU需要感知每次收发操作。
- 用于交换元数据等小数据量操作。

#### 双端操作(READ & WRITE)
- 直接对目标主机内存进行读写，请求发送前后对端无需进行操作。
- 对端CPU不感知单端操作，读写完成后网卡自动响应。
- RDMA的主要优势所在，适用于大数据量传输。

### CQ
当SQ里有消息发送完成，或RQ有接收到新消息，RDMA会在CQ里放入一个新的完成队列元素Completion Queue Element (CQE)，用以通知用户程序。

#### 查询CQ的方式
用户程序有两种同步的方式来查询CQ：

（1）用户程序调用`ibv_cq_poll`来轮询CQ，一旦有新的CQE就可以及时得到通知，但是这种轮询方式很消耗CPU资源；

（2）用户程序在创建CQ的时候，指定一个完成事件通道`ibv_comp_channel`，然后调用`ibv_get_cq_event`接口等待该完成事件通道来通知有新的`CQE`，如果没有新的CQE，则调用`ibv_get_cq_event`时发生阻塞，这种方法比轮询要节省CPU资源，但是阻塞会降低程序性能。

#### 注意事项
关于RDMA的CQE，有个需要注意的地方。
（1）对于RDMA的Send和Receive这种**双边操作**，发送端在发送完成后能收到CQE，接收端在接收完成后也能收到CQE。

（2）对于RDMA的Read和Write这种**单边操作**，比如节点A从节点B读数据，或节点A往节点B写数据，只有发起Read和Write操作的一端，即节点A在操作结束后能收到CQE，另一端节点B完全不会感知到节点A发起的Read或Write操作，节点B也不会收到CQE。

# verbs接口
## RDMA Verbs API 编程
![](attachments/Pasted%20image%2020250323233112.png)

## cq相关
### ibv_create_cq

# 参考
```bash
# RDMA网络编程：常见libibverbs API
https://mp.weixin.qq.com/s/xdXRLSUpPEe4EPZtvQzw0A

# 【verbs】ibv_create_cq()
https://blog.csdn.net/bandaoyu/article/details/116462707
```