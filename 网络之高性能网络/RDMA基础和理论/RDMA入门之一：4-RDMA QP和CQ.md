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

## 其他
### QPN分Send Queue和receive Queue吗？SRQ情况下，QPN的值是什么样的？


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


## send/recv独立的cq or 共享的cq

### 背景

![](attachments/Pasted%20image%2020250506143413.png)

如上所示，最简单的使用send/recv方式发送请求和响应。
对于其中一端而言，比如拿发送端来举例，产生的`send CQE`和`recv CQE`可以在一个CQ中，也可以在两个CQ中。


### 独立cq的问题
对于发送端而言，如果使用2个CQ，即`send CQE`在`send CQ`中，`recv CQE`在另外一个`CQ「recv CQ」`中。
正常来说，应该是先获取到`send CQE`，然后获取到`recv CQE`。
实际上获取`send CQE`和`recv CQE`无法保序，即无法保证获取`send CQE`和`recv CQE`的先后顺序。

#### 潜在问题
如果`send CQ`和 `recv CQ`对应2个不同的`CQ`，那么从2个`CQ`中获取`CQE`无法保序。 无法保序就可能存在一些问题。

比如：业务的逻辑是，RPC的请求和响应是一一对应的，为了保证一一对应，一般请求和响应都有相同的 `id`。即收到响应的时候，需要基于响应中的 `id`查询在链表或者`hash`表中到RPC请求。
而RPC请求的`id`什么时候加入到 链表或者`hash`表中呢？
如果业务的逻辑实现不是很优雅的话，比如是在收到 `send cqe`的时候才进行加入。那么就可能存在问题。
因为 `send cqe`  和 `recv cqe`的获取无法保序，有可能先获取到 `recv cqe`，即先获取到响应，那么此时基于 `id`是无法查询到请求的。

#### 原因分析
潜在的可能性存在两种：
1》发送端收到的ACK和响应的`RoceV2 RDMA`的路径不一致（比如五元组不一致），导致到达的先后顺序无法保证。

2》发送端的`send CQ`和 `recv CQ`对应2个不同的`CQ`，就会出现从2个`CQ`中获取`CQE`无法保序。
比如发送端的线程使用 `Polling` 方式从2个`CQ`中取`CQE`，先从`Send CQ`中取`CQE`，再从`recv CQ`中取`CQE`。
有可能`Send CQ`取`CQE`的流程结束之后，此时产生了`Send CQE`，然后产生了 `recv CQE`，后续从`recv CQ`中取`CQE`。那么就是先获取到`recv CQE`，在下一轮`Polling` 中才可以获取到`Send CQE`。

注：其中`1》`不一定成立，需要验证。

#### 解决
（1）`send CQ`和 `recv CQ`共享相同的`CQ`。
（2）业务逻辑修改。
不要在获取到`send cqe`才进行插入处理，可以完成`send`的准备之后就提前插入；
然后在收到`recv cqe`进行查询。

####  小结
即对于一端而言，如果`send CQ`和 `recv CQ`对应2个不同的`CQ`。那么==从2个`CQ`中获取`CQE`无法保序==。 


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
#### ibv_qp
```c
struct ibv_qp {
	struct ibv_context     *context;
	void		       *qp_context;
	struct ibv_pd	       *pd; /* qp所属的 pd */
	struct ibv_cq	       *send_cq; 
	struct ibv_cq	       *recv_cq;
	struct ibv_srq	       *srq;
	uint32_t		handle;
	uint32_t		qp_num;
	enum ibv_qp_state       state;
	enum ibv_qp_type	qp_type; /* service type: RC/UD等 */

	pthread_mutex_t		mutex;
	pthread_cond_t		cond;
	uint32_t		events_completed;
};
```
### API 接口
```c

```
## 创建CQ
### 数据结构
#### ibv_comp_channel
```c
/* complete channel: 完成通道 */
struct ibv_comp_channel {
	struct ibv_context     *context;
	int			fd;
	int			refcnt;
};
```
#### ibv_cq
```c
struct ibv_cq {
	struct ibv_context     *context;
	struct ibv_comp_channel *channel;
	void		       *cq_context;
	uint32_t		handle;
	int			cqe;

	pthread_mutex_t		mutex;
	pthread_cond_t		cond;
	uint32_t		comp_events_completed;
	uint32_t		async_events_completed;
};
```

#### ibv_cq_ex
```c
/*
* ex: extend。
* 	struct ibv_cq_ex 的前面n个字节必须和 struct ibv_cq完全一样。
*	因为，存在 struct ibv_cq_ex* 到 struct ibv_cq* 的类型强转。
*/
struct ibv_cq_ex {
	/* 下面是 struct ibv_cq 结构中也存在的 */
	struct ibv_context     *context;
	struct ibv_comp_channel *channel;
	void		       *cq_context;
	uint32_t		handle;
	int			cqe;

	pthread_mutex_t		mutex;
	pthread_cond_t		cond;
	uint32_t		comp_events_completed;
	uint32_t		async_events_completed;

	/* 下面是 struct ibv_cq_ex 结构中独有的 */
	uint32_t		comp_mask;
	enum ibv_wc_status status;
	uint64_t wr_id;
	int (*start_poll)(struct ibv_cq_ex *current,
			     struct ibv_poll_cq_attr *attr);
	int (*next_poll)(struct ibv_cq_ex *current);
	void (*end_poll)(struct ibv_cq_ex *current);
	enum ibv_wc_opcode (*read_opcode)(struct ibv_cq_ex *current);
	uint32_t (*read_vendor_err)(struct ibv_cq_ex *current);
	uint32_t (*read_byte_len)(struct ibv_cq_ex *current);
	__be32 (*read_imm_data)(struct ibv_cq_ex *current);
	uint32_t (*read_qp_num)(struct ibv_cq_ex *current);
	uint32_t (*read_src_qp)(struct ibv_cq_ex *current);
	unsigned int (*read_wc_flags)(struct ibv_cq_ex *current);
	uint32_t (*read_slid)(struct ibv_cq_ex *current);
	uint8_t (*read_sl)(struct ibv_cq_ex *current);
	uint8_t (*read_dlid_path_bits)(struct ibv_cq_ex *current);
	uint64_t (*read_completion_ts)(struct ibv_cq_ex *current);
	uint16_t (*read_cvlan)(struct ibv_cq_ex *current); /* vlan in wc ??? */
	uint32_t (*read_flow_tag)(struct ibv_cq_ex *current);
	void (*read_tm_info)(struct ibv_cq_ex *current,
			     struct ibv_wc_tm_info *tm_info);
	uint64_t (*read_completion_wallclock_ns)(struct ibv_cq_ex *current);
};
```
### API接口
```c

```
## Post WR
### 数据结构
#### ibv_send_wr
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

### ibv_recv_wr
```c
/*  receive work request */
struct ibv_recv_wr {
	uint64_t		wr_id; /* User defined WR ID */
	struct ibv_recv_wr     *next; /* Pointer to next WR in list, NULL if last WR */
	struct ibv_sge	       *sg_list; /* Pointer to the s/g array */
	int			num_sge; /* Size of the s/g array */
};
```

#### ibv_sge
**sge  结构**：
```c
/* scatter/gather entry */
struct ibv_sge {
  uint64_t    addr;
  uint32_t    length;
  uint32_t    lkey;
};
```

#### ibv_wr_opcode
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
#### struct ibv_wc
```c
struct ibv_wc {
	uint64_t		wr_id; /* ID of the completed Work Request (WR): 该 wc对应的wr */
	enum ibv_wc_status	status; /* Status of the operation */
	enum ibv_wc_opcode	opcode; /* Operation type specified in the completed WR */
	uint32_t		vendor_err;
	uint32_t		byte_len; /* Number of bytes transferred; 接收或者发送的字节数 */
	/* When (wc_flags & IBV_WC_WITH_IMM): Immediate data in network byte order.
	 * When (wc_flags & IBV_WC_WITH_INV): Stores the invalidated rkey.
	 */
	union {
		__be32		imm_data;  /* Immediate data (in network byte order) */
		uint32_t	invalidated_rkey; /* Local RKey that was invalidated */
	};
	uint32_t		qp_num;   /* Local QP number of completed WR: wr 所在的qp */
	uint32_t		src_qp;   /* Source QP number (remote QP number) of completed WR (valid only for UD QPs) */
	unsigned int		wc_flags; /* Flags of the completed WR */
	uint16_t		pkey_index;  /* P_Key index (valid only for GSI QPs) */
	uint16_t		slid;  /* Source LID */
	uint8_t			sl; 	/* Service Level */
	uint8_t			dlid_path_bits; 	/* DLID path bits (not applicable for multicast messages) */
};
```

#### ibv_wc_status
```c
/* wc(work complete) status */
enum ibv_wc_status {
	IBV_WC_SUCCESS,
	IBV_WC_LOC_LEN_ERR,
	IBV_WC_LOC_QP_OP_ERR,
	IBV_WC_LOC_EEC_OP_ERR,
	IBV_WC_LOC_PROT_ERR,
	IBV_WC_WR_FLUSH_ERR,
	IBV_WC_MW_BIND_ERR,
	IBV_WC_BAD_RESP_ERR,
	IBV_WC_LOC_ACCESS_ERR,
	IBV_WC_REM_INV_REQ_ERR,
	IBV_WC_REM_ACCESS_ERR,
	IBV_WC_REM_OP_ERR,
	IBV_WC_RETRY_EXC_ERR,
	IBV_WC_RNR_RETRY_EXC_ERR,
	IBV_WC_LOC_RDD_VIOL_ERR,
	IBV_WC_REM_INV_RD_REQ_ERR,
	IBV_WC_REM_ABORT_ERR,
	IBV_WC_INV_EECN_ERR,
	IBV_WC_INV_EEC_STATE_ERR,
	IBV_WC_FATAL_ERR,
	IBV_WC_RESP_TIMEOUT_ERR,
	IBV_WC_GENERAL_ERR,
	IBV_WC_TM_ERR,
	IBV_WC_TM_RNDV_INCOMPLETE,
};
```
#### ibv_wc_opcode
```c
enum ibv_wc_opcode {
	IBV_WC_SEND,
	IBV_WC_RDMA_WRITE,
	IBV_WC_RDMA_READ,
	IBV_WC_COMP_SWAP,
	IBV_WC_FETCH_ADD,
	IBV_WC_BIND_MW,
	IBV_WC_LOCAL_INV,
	IBV_WC_TSO,
	IBV_WC_FLUSH,
	IBV_WC_ATOMIC_WRITE = 9,
/*
 * Set value of IBV_WC_RECV so consumers can test if a completion is a
 * receive by testing (opcode & IBV_WC_RECV).
 */
	IBV_WC_RECV			= 1 << 7,
	IBV_WC_RECV_RDMA_WITH_IMM,

	IBV_WC_TM_ADD,
	IBV_WC_TM_DEL,
	IBV_WC_TM_SYNC,
	IBV_WC_TM_RECV,
	IBV_WC_TM_NO_TAG,
	IBV_WC_DRIVER1,
	IBV_WC_DRIVER2,
	IBV_WC_DRIVER3,
};
```

### API 接口
# 参考
```bash
# RDMA cq event机制-ibv_req_notify_cq
https://zhuanlan.zhihu.com/p/688269158


```