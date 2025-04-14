```table-of-contents
```

# RDMA通信模型
## 概述
RDMA是**基于消息的传输协议**，数据传输都是**异步**操作。

RDMA一共支持三种队列，发送队列(SQ)和接收队列(RQ)，完成队列(CQ)。
SQ和RQ都属于WQ(work queue)。
另外，SQ和RQ通常成对创建，被称为Queue Pairs(QP)。

定义了 2 大类型的队列: WQ和CQ。
![](attachments/Pasted%20image%2020250323230221.png)

## WQ
WQ（Work Queue）：App 要收/发数据，就会放置一个 WR（Work Request）到 WQ 作为 WQE（WQ Element）。WQE 是 RNIC 硬件执行任务单元，包含了软件需要硬件执行的动作。RNIC 会获取到 WQE 进行处理。

### SQ 和 RQ

因为 RDMA 支持全双工通信，所以 WQ 进一步细分为 SQ 和 RQ，并称为 QP（Queue Pairs）。通信双方使用一对 QP，通过 BTH QPN 唯一标识，并以此创建 Channel。1 个 RDMA App 可以按需创建多对不同的 QPs 和 Channels。这些 QP 可以用于不同的通信目的，例如：使用不同的服务类型。

SQ（Send Queue）：存放 Send WQE。
RQ（Receive Queue）：存放 Receive WQE。


![](attachments/Pasted%20image%2020250323230314.png)

## CQ
CQ（Complete Queue）：RNIC 每处理完一个 WQE 之后，就会写入一个 CQE 到 CQ，App 从 CQE 中确认一个 WC（Worker Completion）。

# QP
## Work queue
## SQ(send queue)
## RQ(receive queue)
## SRQ(shared receive queue)
## QP 标识
### lid
#### sm_lid
`sm_lid` 指的是 **Subnet Manager Local Identifier**。它是 InfiniBand 网络中用于标识子网管理器（Subnet Manager, SM）的一种标识符。

### gid
### qpn


# QP连接
## QP 建链
建立一对 QP 之间的 Channel，过程中协商通信参数。包括：

（1）GID（Global Identifier，全局 ID）：GID 是 IB 网络的唯一标识。IB 网络中用于标识和寻址网络中的节点或端口。
（2）QPN：QP 的唯一标识，确定建链的对象，GID+QPN 可以在 IB 网络中确定唯一的一个 QP。
（3）VA（虚拟地址）：App 希望访问的虚拟地址。
（4）rkey：remote key, 同上。
（5）qkey：queue key, 是 UD（不可靠数据报）服务类型中专用的 Key，用于校验数据报的合法性。

### 链路和连接
QP 建立 “链路（Channel）” 和 “连接（Connection）” 是两个不同的概念。

RDMA 支持 4 种基本的服务类型，以满足不同服务对可靠性和传输速率的不同需求。
![](attachments/Pasted%20image%2020250323231959.png)

其中，RC、UC 是存在 Connection 的，而 RD、UD 则不存在 Connection，而是直接传输 Datagram。

![](attachments/Pasted%20image%2020250323232050.png)

RC 服务类型类比 TCP 协议，进行通信的 QP 之间需要建立一对一 Connection。RC 通过 ACK 确认、重传、保序等机制，确保数据能在 QP 间进行有序、可靠的传输，适用于对数据可靠性和完整性较高的场景。但相对的，由于连接机制和可靠性保障机制的存在，导致 RC 的通信开销较大。当节点数增加时，将占用更多的网卡和内存等资源。




### 作用



# CQ
## QP和CQ的关系

![](attachments/Pasted%20image%2020250318144055.png)

一个QP包含一个Send Queue(SQ)，一个Receive Queue(RQ)以及对应的Send Completion Queue(SCQ)和Receive Completion Queue(RCQ)。
用户发送请求的时候，把请求封装为一个Work Queue Element(WQE)发送到SQ里面，然后RDMA网卡会把这个WQE发送出去，当这个WQE完成的时候，对应的SCQ里面会被放一个Completion Queue Element(CQE)，然后用户可以从SCQ里面Poll这个CQE并通过检查状态来确认对应的WQE是否成功完成。
需要指出的是，**不同的QP可以共用CQ来减少SRAM的存储消耗**。

# 生产者和消费者角度理解QP和CQ
(1) 对于WQ来说，Host是生产者，RNIC是消费者。
- Host（CPU）生产WR, 把WR放到WQ中去
- RDMA硬件消费WR

(2) 对于CQ来说，RNIC是生产者，Host是消费者。
- RDMA硬件生产WC, 把WC放到CQ中去
- Host（CPU）消费WC

![](attachments/Pasted%20image%2020250326151110.png)

# 操作相关的 Verb API 
## 创建QP
### 数据结构

### API 接口

## Post WR
### 数据结构
**work request 结构**：
```c
/* send work request */
struct ibv_send_wr {
  uint64_t    wr_id; /* User defined WR ID */
  struct ibv_send_wr     *next; /* Pointer to next WR in list, NULL if last WR */
  struct ibv_sge         *sg_list; /* Pointer to the s/g array, 注，此中其实是数组，不是list */
  int     num_sge; /* Size of the s/g array */
  enum ibv_wr_opcode  opcode; /* Operation type , 比如是： IBV_WR_SEND or IBV_WR_RDMA_WRITE */
  unsigned int    send_flags; /* Flags of the WR properties */
  /* When opcode is *_WITH_IMM: Immediate data in network byte order.
   * When opcode is *_INV: Stores the rkey to invalidate
   */
  union {
    __be32      imm_data; /* Immediate data (in network byte order) */
    uint32_t    invalidate_rkey; /* 使用invalidate操作使之前的rkey失效 */
  };
  union {
    struct {
      uint64_t  remote_addr; /* Start address of remote memory buffer */
      uint32_t  rkey; /* Key of the remote Memory Region */
    } rdma;
    struct {
      uint64_t  remote_addr; /* Start address of remote memory buffer */ 
      uint64_t  compare_add;  /* Compare operand */
      uint64_t  swap; /* Swap operand */
      uint32_t  rkey; /* Key of the remote Memory Region */
    } atomic;
    struct {
      struct ibv_ah  *ah; /* Address handle (AH) for the remote node address */
      uint32_t  remote_qpn; /* QP number of the destination QP */
      uint32_t  remote_qkey; /* Q_Key number of the destination QP */
    } ud;
  } wr; /* work request */
  union {
    struct {
      uint32_t    remote_srqn;  /* Number of the remote SRQ */
    } xrc;
  } qp_type;
  union {
    struct {
      struct ibv_mw *mw; /* Memory window (MW) of type 2 to bind */
      uint32_t    rkey;  /* The desired new rkey of the MW */
      struct ibv_mw_bind_info bind_info; /* MW additional bind information */
    } bind_mw;
    struct {
      void           *hdr; /* Pointer address of inline header */
      uint16_t    hdr_sz; /* Inline header size */
      uint16_t    mss; /* Maximum segment size for each TSO fragment */
    } tso;
  };
};
```

**sge  结构**：
```c
/* scatter/gather entry */
struct ibv_sge {
  uint64_t    addr;
  uint32_t    length;
  uint32_t    lkey;
};
```

**操作码**：
```c
/*  rdma WR(work request) 操作码(opcode)*/
enum ibv_wr_opcode {
  IBV_WR_RDMA_WRITE,
  IBV_WR_RDMA_WRITE_WITH_IMM,
  IBV_WR_SEND,
  IBV_WR_SEND_WITH_IMM,
  IBV_WR_RDMA_READ,
  IBV_WR_ATOMIC_CMP_AND_SWP,
  IBV_WR_ATOMIC_FETCH_AND_ADD,
  IBV_WR_LOCAL_INV,
  IBV_WR_BIND_MW,
  IBV_WR_SEND_WITH_INV,
  IBV_WR_TSO,
  IBV_WR_DRIVER1,
  IBV_WR_FLUSH = 14,
  IBV_WR_ATOMIC_WRITE = 15,
};
```
### API 接口
## Poll WC
### 数据结构

### API 接口
# 参考
```bash
# RDMA cq event机制-ibv_req_notify_cq
https://zhuanlan.zhihu.com/p/688269158


```