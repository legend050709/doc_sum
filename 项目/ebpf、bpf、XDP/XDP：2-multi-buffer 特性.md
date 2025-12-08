```table-of-contents
```
# 背景
默认情况下，xdp 和 jumbo frame 是冲突的。

传统的 XDP 架构与 Jumbo Frame (巨型帧，通常 MTU 达到 9000 字节，即 9K) 存在冲突，原因在于 **XDP 的设计约束**：

## XDP 的性能基础
XDP 的高性能来源于其极简的数据结构和 **Zero-Copy (零拷贝)** 思想。
早期 XDP 框架传递给 eBPF 程序的数据结构 (`xdp_buff`) 假定数据缓冲区是**连续**的。这是为了让 eBPF 程序能够简单地通过指针和偏移量访问整个数据包，即直接在网卡接收到的原始数据缓冲区上操作，而无需将数据拷贝到内核的 `sk_buff` 结构中。。


## Jumbo Frame 的物理大小
- Linux 内核的默认内存页（PageSize）大小通常是 **4KB**。
- Jumbo Frame（巨型帧）的 MTU 通常为 **9000 字节（9KB）**。
因此：在引入 multi-buffer 之前，**XDP 缓冲区限制** 假设一个传入的数据包（帧）必须**完全包含在一块连续的物理内存内(通常是 1 个 page，4 KB）**。

```bash
XDP 程序的 `ctx->data` 到 `ctx->data_end` 指向的就是这段连续 buffer。

因此，XDP 的基本假设：一个 packet == 一块连续内存

```

```c
/* User return codes for XDP prog type.
 * A valid XDP program must return one of these defined values. All other
 * return codes are reserved for future use. Unknown return codes will
 * result in packet drops and a warning via bpf_warn_invalid_xdp_action().
 */
enum xdp_action {
	XDP_ABORTED = 0,
	XDP_DROP,
	XDP_PASS,
	XDP_TX,
	XDP_REDIRECT,
};

struct xdp_buff {
	void *data;
	void *data_end;
	void *data_meta;
	void *data_hard_start;
	unsigned long handle;
	struct xdp_rxq_info *rxq;
};
```

## 内存分配问题
由于 9KB 远大于 4KB 的单个内存页，网卡驱动程序必须将一个完整的 Jumbo Frame 分散存储在至少两个或三个不连续的内存页中。
如果强制要求将 9KB 数据包存储在一块**连续**的内存区域，驱动程序将需要执行额外的**数据重组（Coalescing）** 或 **昂贵的内存拷贝**操作。这会打破 XDP 追求的零拷贝和极简设计的原则，引入不可接受的性能开销。

因此，一个 9K 的巨型帧必须被分散存储在两个或更多的不连续内存页中。

```bash
网络设备接收Jumbo Frame（如 MTU 9K）时就会将这个 packet 拆成多个 buffer（scatter-gather）
[ buf0 (4K) ] -> [ buf1 (4K) ] -> [ buf2 (1K) ]

而早期 XDP 程序 看不到 buf1、buf2，它只能看到第一块 buffer：ctx->data ~ ctx->data_end   (仅 buf0)；
所以 它无法访问整个 packet，因此：Jumbo frame 与 XDP 在早期模型下天然冲突。
```





## 问题

### XDP程序和 jumbo frame不兼容

### XDP程序和RDMA 大的MTU不兼容
RDMA使用大的MTU可以提升性能。
但是使用大的MTU，可能导致XDP程序无法使用。即：网卡设置了大的MTU，可能就无法挂载XDP程序了。


# XDP multi-buffer 机制
Linux 5.17/5.18 引入 **XDP multi-buffer** 支持。multi-buffer 机制 是 Linux 内核为解决 XDP 在处理超出单个PageSize(默认4K)的数据包时的瓶颈和兼容性问题而引入的一项重要改进。

XDP multi-buffer 的本质是：允许 XDP 程序访问由多个内存段组成的非线性 packet，使 XDP 可以处理 Jumbo Frame 这样的超大包，而无需将 packet 线性化。

## Multi-buffer 的核心思想
XDP multi-buffer 解决的核心问题：让 XDP 程序能访问由多个内存 buffer（fragment）组成的一个 packet。

即：让 XDP 也能够处理 non-linear（多段）buffer，就像 SKB 那样。而不是强制将收到的 packet 线性化。

可以把它理解为：**XDP 之前只能处理线性内存，现在可以处理 scatter-gather。**

## 作用
最直接的作用是解决了 **Jumbo Frame** 和 **XDP** 的共存问题，让用户可以在高带宽场景下使用 9K MTU 并同时享受 XDP 的超高性能。



## 内核改动

内核支持multi-buffer需要内核和驱动两部分支持。内核主要是提供基础的multi-buffer的操作函数，比如读写非线性区域内存。驱动需要在收包逻辑中组装分片为一个xdp_buff。
1. 数据结构struct bpf_prog_aux增加xdp_has_frags字段，标记是否支持分段。
2. 数据结构struct xdp_buff增加flags字段，标记是否有分段。
3. xdp_buff的flags操作函数，has/set/clear frags flags
4. 数据结构struct xdp_frame增加flags字段，标记是否有分段。
5. xdp数据操作函数，xdp_get_buff_len、xdp_load_bytes、xdp_store_bytes、xdp_adjust_tail、xdp_copy，主要是为了读写分段的数据。
6. bpf map相关修改，相关性待定。


![](attachments/Pasted%20image%2020251114174236.png)

```c
enum xdp_buff_flags {
	XDP_FLAGS_HAS_FRAGS		= BIT(0), /* non-linear xdp buff */
	XDP_FLAGS_FRAGS_PF_MEMALLOC	= BIT(1), /* xdp paged memory is under
						   * pressure
						   */
	/* frags have unreadable mem, this can't be true for real XDP packets,
	 * but drivers may use XDP helpers to construct Rx pkt state even when
	 * XDP program is not attached.
	 */
	XDP_FLAGS_FRAGS_UNREADABLE	= BIT(2),
};

struct xdp_buff {
	void *data;
	void *data_end;
	void *data_meta;
	void *data_hard_start;
	struct xdp_rxq_info *rxq;
	struct xdp_txq_info *txq;

	union {
		struct {
			/* frame size to deduce data_hard_end/tailroom */
			u32 frame_sz;
			/* supported values defined in xdp_buff_flags */
			u32 flags;
		};

#ifdef __LITTLE_ENDIAN
		/* Used to micro-optimize xdp_init_buff(), don't use directly */
		u64 frame_sz_flags_init;
#endif
	};
};

```

在 `xdp_buff` 结构的 `flags` 字段中引入了一个 frags 位（`XDP_FLAGS_HAS_FRAGS`），用于通知 BPF/网络层：
- 如果该位被设置，则这是一个非线性 XDP 帧。
- 如果该位未设置，则这是一个线性 XDP 帧（即数据连续）。

只有具备 XDP 分段能力的驱动程序才会为**非线性帧**设置此分段标记。这样做是为了保持接收线性帧的能力，而无需引入额外的开销，因为只有在设置了 `XDP_FLAGS_HAS_FRAGS` 位时，第一个缓冲区末尾的 `skb_shared_info` 结构才会被初始化。

## 应用场景
**巨型帧 (Jumbo-frames)：** 解决像 9K MTU 这种数据包大小超过单个内存页的问题。
**数据包头部/负载分离 (Packet Header Split)：**（请参考 Google 在 NetDevConf 0x14 上的用例 [0]）。
**XDP_REDIRECT 的 TSO/GRO 支持：** 为 `XDP_REDIRECT` 机制提供**传输分段卸载（TSO）**和**通用接收卸载（GRO）**的支持。

## 小结
简而言之，XDP multi-buffer 机制是 **XDP 架构向支持大 MTU 数据包所做的结构性升级**，通过在内核层面实现对分散存储数据的抽象和管理，保证了 XDP 性能的同时解决了与 Jumbo Frame 的冲突。

# 其他
## AF_XDP Multi-Buffer

**传统限制**：
默认情况下，AF_XDP 的单个数据包必须完全容纳在一个 UMEM 帧中。若数据包大小超过 frame_size，会被内核丢弃。

**多缓冲区(Multi-Buffer)模式**：
允许一个数据包分散在多个 UMEM 帧中（类似 Scatter-Gather DMA）。
支持超大包（如 64KB 巨型帧）、IP 分片或 GRO/GSO 聚合包的处理。

### 启用多缓冲区的必要条件
#### 用户空间配置：`XDP_USE_SG` 绑定标志

- **作用**：通知内核 AF_XDP 套接字支持多缓冲区数据包。
- **设置方式**：在绑定套接字时添加到 `bind_flags`：
```bash
struct xsk_socket_config cfg = {
    .bind_flags = XDP_USE_SG  // 启用多缓冲区支持
};
xsk_socket__create(&xsk, "eth0", 0, umem, &rx_ring, &tx_ring, &cfg);
```


#### 内核 XDP 程序：使用 `xdp.frags` Section
- **作用**：XDP 程序需声明支持多缓冲区处理模式。
- **实现方式**：在 XDP 程序的代码中指定 Section 名称：
```bash
SEC("xdp.frags")  // 关键：使用 xdp.frags 而非 xdp
int xdp_process(struct xdp_md *ctx) {
    // 处理多缓冲区数据包
    return XDP_PASS;
}

加载方式：
ip link set dev eth0 xdp obj xdp_prog.o sec xdp.frags
```

### 数据包处理流程变化
```bash
(1) 未启用多缓冲区:
网卡收到数据包 → XDP 程序检查长度 → 超过 frame_size → 丢弃


(2) 启用多缓冲区:
网卡收到数据包 → XDP 程序分割到多个 UMEM 帧 → 用户空间通过多个 RX 描述符接收
(需内核 ≥ 5.5 支持)



```



# 参考
```bash

```