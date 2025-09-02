```table-of-contents
```

# 有序性的层级
## 操作内有序(Intra-Operation Ordering)
 **数据包顺序（Packet Order）/ 操作内有序(Intra-Operation Ordering)**： 
 单个操作（如Send或Write）可能被拆分成多个网络数据包传输，这些包是否按发送顺序到达。

## 操作间有序(Inter-Operation Ordering)

**操作顺序（Operation Order）** 
多个独立的RDMA操作（如连续发起两个Write）的完成顺序是否与提交顺序一致。

## 事件流顺序

**事件流顺序（Event Flow Order）** 
跨QP或跨节点的全局事件顺序（如内存更新与通知的组合），需应用层协议保证。

# 典型乱序
## 原因
### 网络路径差异
包通过不同交换机路径传输（尤其无连接服务如UD/RD）。
- **解决**：无连接服务需应用层重排序。

注：无连接服务，类似于UDP，每次发包的UDP五元组不一样？？有连接服务，一个QP的五元组是固定的？

### 传输层重传
可靠服务（RC/RD）中丢失的包重传后，新包可能先到达。
```bash
发送：包1(PSN=1) → 包2(PSN=2) 
包1丢失 → 重发包1(PSN=1) 
到达顺序：包2 → 包1 ===> 接收端按PSN=1,2排序执行
```

- **结论**：重传不影响RC/RD的最终有序性。

### HCA处理优化
为提升吞吐，HCA可能并发处理操作（UC/UD）。

## 场景
### UC/UD传输的乱序
```bash
发送顺序: [操作A] → [操作B] 
到达顺序: [操作B] → [操作A] // 网络抖动导致乱序
```

### RD的多消息传输
```bash
发送: [消息1] → [消息2] 
到达顺序: [消息2] → [消息1] // 消息间无序
```

# 有序性的级别
## 严格有序（Strict Ordering）
**(1) 机制**：
- 硬件保证**同一QP内**的操作按提交顺序完成（如RC服务）。
- 发送队列（SQ）顺序 = 网络包顺序（PSN严格递增） = 接收端执行顺序。

**(2) 服务类型**：仅 **RC** 支持。

**(3) 代价**：等待前序操作ACK增加延迟

**(4) 范例**
```bash
// RC QP示例：提交顺序即执行顺序
ibv_post_send(qp, &write_wr1); // 操作1：写入数据A
ibv_post_send(qp, &write_wr2); // 操作2：写入数据B → 保证B在A后执行
```

## 消息内有序（Intra-Message Ordering）

**(1) 机制**：
- 单个大消息被拆分为多个包时，包按发送顺序到达（如RD服务）。
> 注：==消息间仍可能乱序==。

**(2) 服务类型**：RD 支持。

**(3) 范例**
```bash
Send消息X [包X1][包X2] → 保证X1在X2前到达 
Send消息Y [包Y1][包Y2] → 但Y可能比X先到达
```

## 松散有序（Relaxed Ordering）
 **（1）机制**：
- 通过 `IBV_SEND_INLINE`或`Fence`选择性控制有序性。
- 允许后续操作**绕过前序未完成的操作**提前执行。

## 完全无序（No Ordering）
**(1)机制**：
- 无任何顺序保证，数据包按网络路径自由到达。

**(2) 服务类型**：UD和 UC的默认行为。

**(3) 解决方法**
UD应用层，将应用层序号也作为数据的一部分传递；对端收到数据之后，解析出序号，进行排列组合。
```c
// UD应用需添加序列号
struct Message {
    uint32_t seq; // 应用层序号
    char data[];
};
// 接收端重组乱序消息
```



# 有序性的实现




# 服务类型和有序性

|**服务类型**|**数据包顺序**|**操作顺序**|**典型场景**|
|---|---|---|---|
|**RC (可靠连接)**|✅ **严格保证**|✅ **严格保证**|分布式数据库、存储系统（如NVMe-oF）|
|**UC (不可靠连接)**|❌ **不保证**|❌ **不保证**|实时数据流（容忍乱序）|
|**UD (不可靠数据报)**|❌ **不保证**|❌ **不保证**|行情广播、日志收集|
|**RD (可靠数据报)**|✅ **消息内包有序**|❌ **消息间操作无序**|分布式共享内存的通信层|





# 有序性和send_flag

## 有序性和`IBV_SEND_FENCE`

### send_flags之IBV_SEND_FENCE
#### 背景

![](attachments/Pasted%20image%2020250715170520.png)

#### 适用的服务类型和操作场景

```bash
IBV_SEND_FENCE:
Set the fence indicator for this WR. This means that the processing of this WR will be blocked until all prior posted RDMA Read and Atomic WRs will be completed. Valid only for QPs with Transport Service Type IBV_QPT_RC.
```

只有RC服务类型才可以设置`IBV_SEND_FENCE`标记。

#### 什么时候使用`IBV_SEND_FENCE`

![](attachments/Pasted%20image%2020250715171113.png)

即：
==如果2个操作访问的内存地址范围存在重叠的情况下，在`RDMA  READ/Atomic` 之后的操作是 `send/write`等操作，那么可能需要再2个操作之间添加`fence`。
如果2个操作访问的内存地址范围不存在重叠，则不用考虑加`fence`==。

#### 为什么RC保序，还需要`IBV_SEND_FENCE`


## 有序性和`IBV_SEND_INLINE`
### IBV_SEND_INLINE
`IBV_SEND_INLIN`E 减少网卡的`DMA`。(`CPU`直接将数据写入网卡缓冲,而不是等网卡来`DMA`）

```bash
IBV_SEND_INLINE:
    The memory buffers specified in sg_list will be placed inline in the Send Request. 
    This mean that the low-level driver (i.e. CPU) will read the data and not the RDMA device. 
    This means that the L_Key won't be checked, 
    actually those memory buffers don't even have to be registered and they can be reused immediately after ibv_post_send() will be ended. 
    Valid only for the Send and RDMA Write opcodes
```
`sg_list`中指定的内存缓冲区将内联放置在SR（ WQE）中。这意味着底层驱动程序（即CPU）将读取数据，而不是RDMA设备通过DMA来读取。这意味着将不会检查`L_Key`，实际上那些内存缓冲区甚至不必注册(毕竟CPU本来是老大/主子），并且可以在`ibv_post_send（）` 将要结束后，立即重用它们。仅对`Send`和`RDMA write`操作码有效。


#### INLINE的好处

![](attachments/Pasted%20image%2020250715193644.png)

对于`send/write`发送小的数据，通过`inline`，可以提升性能。应该是通过CPU拷贝小的数据内容，比 RNIC通过DMA来读取更快。

#### INLINE 和非INLINE的对比
**（1）两次DMA Vs 一次DMA**
原先是SR（内含`sg_list`）和`sg_list`中指向的内存缓冲区 放置在两个位置，网卡先读取`SR`获得`sg_list`，再根据`sg_list`信息读取`sg_list`指向的内存缓存。需要两次DMA操作。
内联就是将内存缓存区放置在`SR`内，网卡读取`SR`就一并获得内存缓存，只需一次`DMA`。


#### 使用INLINE的陷阱

根据文档, `inline`发送不需要等待`wc`就可以重用发送`buffer`. 不需要等待`wc`就可以继续发消息，但是如果不处理`wc`，那么就不会清理`sr`，连续不断的继续发送`INLINE`消息（而不去处理`wc`），`sr`得不到清理最终会撑爆`sq`，导致最后发不出消息。
所以使用`INLINE`的时候记得在`sq`撑爆之前去处理`wc`。


# 有序性和内存可见性(内存一致性)

# 有序性和立即数imm_data


# 有序性和原子操作的关系



# 注意事项
## RC中独立的send/recv CQ
在RC `send/recv`的操作中，如果`send`和`recv`对应各自的CQ(即：send CQ 和 recv CQ不同)。
那么一个线程中，`post send` 和  `post recv` 的 先后顺序 和 对应产生的 `send cqe`, `recv cqe`的先后顺序 没有直接关系。
即：先`post send` ，再`post recv`，不一定会先产生`send cqe`，再产生`recv cqe`。

==注：如果两者对应的使用一个`CQ`，那么产生`CQE`的顺序和`Post WR`的顺序是一致的==。


# 参考
```bash

```