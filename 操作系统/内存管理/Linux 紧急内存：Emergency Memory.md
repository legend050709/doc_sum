```table-of-contents
```

# 为什么要有紧急保留内存(Emergency reverse memory)

内存耗尽时，系统本身的"自救"流程（如 OOM 回收、网络通知、日志写入）需要分配内存——但内存已经没了，自救就会死锁。Emergency memory 是预留的最后一块内存，专门给这些关键路径用，普通分配路径永远拿不到。

如果这些路径和普通进程竞争同一块内存池，就会产生死锁——"要分配内存才能释放内存"。预留区打破了这个死锁。



# 区域水线（mem zone watermark）
每个 " 内存区域 " 都有3条水线 :

**高水线 :** High Water Marker , 内存区域 空闲页数 大于 高水线 , 内存充足 ;

**低水线 :** Low Water Marker , 内存区域 空闲页数 小于 低水线 , 内存轻微不足 ;

**最低水线 :** Min Water Marker , 内存区域 空闲页数 小于 最低水线 , 内存严重不足 ;

最低水线（`WMARK_MIN`）以下的内存是 **紧急保留内存(Emergency reverse memory)**  ；当内存低于 `WMARK_MIN`时，普通分配全部失败，只允许紧急分配（GFP_ATOMIC）和 特权分配（PF_MEMALLOC） 。
并且得到紧急保留内存的进程必须承若 ”==分配少量内存 , 释放更多内存==" ;

## GFP_ATOMIC（紧急分配）
```c
kmalloc(size, GFP_ATOMIC);
```

特点：
- 不允许睡眠（interrupt/softirq 用）
- 可以突破 watermarks
- 使用 emergency reserve


典型场景：
- 网络收包（RX）
- 中断处理

## PF_MEMALLOC（特权进程）

某些线程会打这个标志，允许使用保留内存（即使系统 OOM）
```bash
current->flags |= PF_MEMALLOC;
```

特权进程比如：
- kswapd（内存回收线程）
- 网络栈关键**控制路径（ACK/Fin/Rst）**：==只让“能释放内存的流”使用 emergency memory==
```c
ACK: 收到ACK，可以接下来释放发送缓冲区的数据。
FIN/RST: 断开连接，接下来可以释放socket的接受缓冲区和发送缓冲区。

sk_memalloc_socks

目的：
	在内存耗尽时仍然可以：
	- 接收 TCP ACK/RST/FIN
	- 避免连接彻底卡死

否则会出现：“死锁”：没内存 → 收不到 ACK → 连接永远卡住
```


## 从正常到崩溃的工作流程
```bash
正常
  ↓
内存下降
  ↓
低于 LOW → 开始回收
  ↓
低于 MIN → 进入紧急状态
         → 普通分配失败
         → 仅 emergency 分配可用
  ↓
回收失败
  ↓
触发 OOM killer

注：比如看到 `Out of memory: Kill process xxx` 日志，就是 emergency memory 保证 OOM 能被触发；
```

# 紧急内存介绍
`emergency memory` 是**一整套保留内存 + 特权分配机制**，用于在系统**内存极度紧张甚至 OOM 边缘时，仍然保证关键路径（尤其是网络和内存回收）能继续运行**。

如果没有 `emergency memory`，会发生：
```bash
- 网络收不到 ACK → TCP 卡死
- 内核无法分配内存 → 无法回收内存
- OOM killer 无法运行 → 系统直接“假死”
```

所以必须有一部分内存：**普通路径不能用，但关键路径可以用**

# 紧急内存在网络收包中的意义
## 在linux内核收包中
Linux 网络栈中`emergency memory`的核心目标是：在内存快耗尽时，优先保证**控制路径**（TCP ACK / FIN / RST / reclaim）永远能跑。
否则就会出现经典死锁：**没内存 → 收不到 ACK → 连接释放不了 → 更没内存**

### 紧急内存中是如何保护的？
#### 问题
NIC 收到的每个包，都会先占用一块 buffer（page / DMA buffer），此时是无法提前知道接下来要收到的是数据报文还是控制报文。
那么，也就是这一块内存无论如何都要占用。

#### 收包的完整路径
完整路径拆：
**（1）Step 1：NIC DMA 收包**：
`NIC → DMA → RX ring buffer（page）`

这一步：必须发生，无法区分数据 报文和 控制报文（TCP ACK / FIN / RST ）；
内存来源：page pool / driver ring buffer（预分配）
这部分内存 **不是 emergency memory 管的重点**；因为：它是预分配的（ring buffer），数量有限（不会无限增长）。

**（2）Step 2：构造 sk_buff（skb）**
此时需要分配`struct sk_buff` 结构，填充元数据`metadata`。**这里开始使用内核内存分配器（kmalloc / slab）**

**（3）Step 3：协议栈处理（开始分流）**
进入 TCP/IP：`ip_rcv → tcp_v4_rcv`; 此时，就可以分流了：
```bash
- TCP flags（ACK / SYN / FIN / RST）
- payload length
```

**(4) Step 4️：是否进入 socket buffer**
`sk->sk_receive_queue(socket 的接收缓冲区)`，这里才是真正占用大量内存的地方。因为：==skb 会被排队，数据可能堆积，应用没读 ，就会导致内存占用持续增长==。

#### 分析
Q：控制路径才用 emergency memory，那到底分配的是什么？

A：紧急内存保证的不是 page，**紧急内存保证的是内存不足时，`sk_buff` 结构体的分配成功**。通过 `kmem_cache_alloc(skb_cache, GFP_ATOMIC)`，保证 `sk_buff` 结构体的分配必须成功。
**内存不足时，在协议栈层分流时，Linux 会“尽早丢弃数据包”，不 enqueue 到 socket 的接收缓冲区**。
即：==在分流时，发现某个skb中有紧急内存的标记，同时又是数据包，就会尽快的释放==。
```c
if (sk_under_memory_pressure(sk)) {
    if (!is_control_packet)
        drop;
}

drop： 释放 skb，以及对应的page。
```

#### 小结

紧急内存保证的不是 page，**紧急内存保证的是内存不足时，`sk_buff` 结构体的分配成功**，在分流时，发现某个skb中有紧急内存的标记，同时又是数据包，就会尽快的释放。防止数据包进入到接收缓冲区占用内存。

## 在 RDMA / 高性能网络中
DPDK/RDMA 虽然绕过 kernel，但需要自己实现：
```bash
- reserve buffer（类似 emergency memory）
- backpressure（流控）
```

DPDK/RDMA中 如果没有 emergency memory, 就会出现 **`mempool 空 → 收不到 ACK → 流量停 → buffer 不释放 → 死锁`**；和 Linux 一模一样。




# 其他
## 查看
### 紧急内存大小：min_free_kbytes
```bash
cat /proc/sys/vm/min_free_kbytes
echo 262144 > /proc/sys/vm/min_free_kbytes
```
作用：定义系统必须保留的最小空闲内存，低于这个值后普通分配会被拒绝
这部分内存就是`emergency memory` 的“来源”。

### 监控 watermarks
```bash
cat /proc/zoneinfo
```

每个内存 zone 有三个水位：
- `WMARK_HIGH`
- `WMARK_LOW`
- `WMARK_MIN` 

当空闲页低于 `WMARK_MIN` 时，普通分配（GFP_KERNEL）失败，只有紧急路径可以分配。

## TCP 内存三水位

```bash
cat /proc/sys/net/ipv4/tcp_mem

[pressure_low, pressure_high, max]
      │               │         │
      │               │         └── 超过此值：强制拒绝分配
      │               └── 进入 memory pressure 模式：缩小 sndbuf/rcvbuf
      └── 退出 memory pressure 模式
```


## 紧急内存和 OOM 的关系
当内存耗尽时：
```bash
1. 进入 reclaim（回收）
2. 如果失败 → 进入 OOM
3. OOM killer 触发
```

**emergency memory 保证 OOM 能被触发**


## 常见问题
### 问题1：系统内存还有，但分配失败
系统内存已低于 `WMARK_MIN`

### 问题2：网络完全卡死（无 ACK）
emergency reserve 被耗尽

### 问题3：GFP_ATOMIC allocation failure
日志类似：
```bash
page allocation failure: order:0, mode:0x20(GFP_ATOMIC)
```

说明：
```bash
- emergency memory 都没了
- 系统已经非常危险
```


# 参考
```bash

```