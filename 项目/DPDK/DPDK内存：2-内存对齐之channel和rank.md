```table-of-contents
```
# 前置知识点
从CPU到实际的存储节点，依据层级划分：Channel > DIMM > Rank > Chip > Bank > Row /Column

![](attachments/Pasted%20image%2020250421115418.png)


## channel
### 概念
channel是 CPU Socket 到 Main Memory 的传输通道，每个channel对应一个CPU的内存控制器，每个channel可以配有多个DIMM。

双通道（2个channel）：CPU外核或北桥有两个内存控制器，每个控制器控制一个内存通道。理论上内存带宽增加一倍。四通道同理。
服务器规格 一个socket（NUMA） 通常是 2 / 4 / 6 Memory Channels。

### 作用

将变量（比如一个三层网络的Pkt），平均分布于不同的channel之上（多依靠padding），可以减少channel拥塞，显著提升性能。如图：

![](attachments/Pasted%20image%2020250420223025.png)
```bash
Two Channels and Quad-ranked DIMM Example 范例图
```
## DIMM
**DIMM（Dual Inline Memory Modules，双列直插式存储模块）**：就是一条条可插拔的内存条。每个 Channel 可以配备多个 DIMM 插槽。

以前的主机是直接将存储芯片（chip）插在主板上的，然后发展出SIMM（Single In-line Memory Module），将多个chip焊在一片电路板上，成为内存模块，再将它插到主板上。

## rank

DIMM上一部分或所有chip组成一个rank（64bit），因此内存至少需要有16片4bit的chip或者8bit的chip（不存在4bit和8bit芯片混搭的情况）。
内存控制器只允许CPU每次与内存进行一组64bits的数据交换，如果 Rank 规格也是 64bit 的，那么一次交换就会在一个 Rank 上完成。
rank也可以理解为连接到同一个CS（chip select）的一组chip。

DIMM上的rank数目是可访问DIMM完整数据位宽的独立DIMM集合的数量。 由于多个rank共享相同的数据通道（channel），因此多个rank不能被同时访问。

### rank分类

- Single-Rank（1R），要动用到DIMM上所有的chip，这些chip由同一个片选信号控制。
- Double-Rank（2R），产生2个64位rank，由2个片选信号控制，这2个片选信号是交错的，不争抢内存总线。
- Quad-Rank（4R），产生4个64位rank,由4个片选信号控制，这4个片选信号是交错的，不争抢内存总线。

在地址选择时，只有当片选信号有效时，此片所连的地址线才有效。


### channel和rank比较

|概念|说明|
|---|---|
|内存通道（channel）|CPU 与内存之间的“高速通道”，多通道可并行传输|
|Rank|每个 DIMM 上可被单独访问的 DRAM 组，但它们共享通道|
|优化方式|给每个对象加合适的 padding，使它们分布在不同的通道/rank 上|
|目标|让 CPU 同时利用多个内存通道并行加载数据，提升吞吐|
|实用场景|包处理（L3、分类）只读前 64 字节时效果最明显|
|启用方式|使用 EAL 启动参数 `--memory-channels=N` 显式设置|



|项目|含义与角色|
|---|---|
|**Channel**（通道）|是 **CPU内存控制器与 DIMM（内存条）之间的数据通路**，可以视为**逻辑访问通道**，**是真正可以并行工作的路径**。多个 Channel 可以 **同时传输数据**。|
|**Rank**（秩）|是 DIMM 中的 **物理 DRAM 芯片组合**，能提供一次完整数据访问（如 64 位）。**同一时间只能激活一个 Rank**，因为多个 Rank 共用同一数据线。|

|属性|Channel|Rank|
|---|---|---|
|是否并行可用|✅ 是|❌ 否（每个通道内同时只能激活一个 rank）|
|是什么|逻辑访问通道（控制器<->DIMM）|DIMM 内的物理 DRAM 组|
|控制器视角|可以并发发起多个请求|每个请求只能打到一个 rank|
|通常设置|主板上指定的通道数（如 dual-channel）|DIMM 内部由硬件厂商决定（1R、2R、4R）|


## chip
内存条上的黑色芯片就是chip，提供4bit/8bit/16bit/32bit的数据，提供4bit的芯片记作x4，提供8bit的芯片记作x8。

## 其他

再往下的bank、Row /Column这里可以暂时不用关心了，通过上文的示意图中了解一下就行。

# 内存对齐
根据X86架构上的硬件内存配置，可以通过在对象之间添加**特定的内存填充（padding）** 来极大地提高性能。目的是==确保每个对象的起始位置被均匀的分布在不同的channel和rank上，以便实现所有通道的负载均衡==。


## 优化的目标
确保每个对象的起始地址**落在不同的内存通道（channel）和 rank 上**，  从而使所有内存通道被**均衡地使用**，避免瓶颈。

特别是在执行 **L3 转发（forwarding）** 或 **流分类（flow classification）** 的时候，性能差异明显。原因是：
通常只访问数据包的**前 64 字节**（例如只看头部做决策）；
如果所有包的起始地址都落在同一个通道，那 CPU 就只能从一个通道拉数据，造成瓶颈。而如果起始地址均匀分布在多个通道上，**多个通道（channel）可以并行工作，访问带宽显著提升**。


## 内存地址和channel以及rank的关系



# 查看设备的channel和rank
查看channel个数信息：
```bash
# dmidecode -t 17 | grep Locator
Bank Locator: NODE 0 CHANNEL 0 DIMM 0
Bank Locator: NODE 0 CHANNEL 0 DIMM 1
Bank Locator: NODE 0 CHANNEL 0 DIMM 2
Bank Locator: NODE 0 CHANNEL 1 DIMM 0
Bank Locator: NODE 0 CHANNEL 1 DIMM 1
Bank Locator: NODE 0 CHANNEL 1 DIMM 2
Bank Locator: NODE 0 CHANNEL 2 DIMM 0
Bank Locator: NODE 0 CHANNEL 2 DIMM 1
Bank Locator: NODE 0 CHANNEL 2 DIMM 2
Bank Locator: NODE 0 CHANNEL 3 DIMM 0
Bank Locator: NODE 0 CHANNEL 3 DIMM 1
Bank Locator: NODE 0 CHANNEL 3 DIMM 2
Bank Locator: NODE 1 CHANNEL 0 DIMM 0
Bank Locator: NODE 1 CHANNEL 0 DIMM 1
Bank Locator: NODE 1 CHANNEL 0 DIMM 2
Bank Locator: NODE 1 CHANNEL 1 DIMM 0
Bank Locator: NODE 1 CHANNEL 1 DIMM 1
Bank Locator: NODE 1 CHANNEL 1 DIMM 2
Bank Locator: NODE 1 CHANNEL 2 DIMM 0
Bank Locator: NODE 1 CHANNEL 2 DIMM 1
Bank Locator: NODE 1 CHANNEL 2 DIMM 2
Bank Locator: NODE 1 CHANNEL 3 DIMM 0
Bank Locator: NODE 1 CHANNEL 3 DIMM 1
Bank Locator: NODE 1 CHANNEL 3 DIMM 2
```

查看channel上插的内存条信息
```bash
# dmidecode -t 17 | grep Size
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
Size: 4096 MB
Size: No Module Installed
Size: No Module Installed
```

这两个信息结合起来就可以看到当前的内存channel信息了。 结合上面两个就可以看出cpu node0上插了4个4G的内存，DIMM0，Channel 0,1,2,3。 node1上也是一样。

# DPDK中的使用channel和rank
在 EAL 参数项中有两个参数 `-n`和 `-r`，分别用于指定内存 channel 数和 rank 数，这两个参数会影响 mempool 的 ele 的大小，dpdk 会根据这两个参数的值和用户指定的 mempool 的 ele  大小来确定一个更优的 ele的 大小。

## mempool的ele的大小
```c

/* get the header, trailer and total size of a mempool element. */
uint32_t rte_mempool_calc_obj_size(uint32_t elt_size, uint32_t flags,
	struct rte_mempool_objsz *sz)
{
	struct rte_mempool_objsz lsz;

	sz = (sz != NULL) ? sz : &lsz;

	sz->header_size = sizeof(struct rte_mempool_objhdr);
	if ((flags & RTE_MEMPOOL_F_NO_CACHE_ALIGN) == 0)
		sz->header_size = RTE_ALIGN_CEIL(sz->header_size,
			RTE_MEMPOOL_ALIGN);

#ifdef RTE_LIBRTE_MEMPOOL_DEBUG
	sz->trailer_size = sizeof(struct rte_mempool_objtlr);
#else
	sz->trailer_size = 0;
#endif

	/* element size is 8 bytes-aligned at least */
	sz->elt_size = RTE_ALIGN_CEIL(elt_size, sizeof(uint64_t));

	/* expand trailer to next cache line */
	if ((flags & RTE_MEMPOOL_F_NO_CACHE_ALIGN) == 0) {
		sz->total_size = sz->header_size + sz->elt_size +
			sz->trailer_size;
		sz->trailer_size += ((RTE_MEMPOOL_ALIGN -
				  (sz->total_size & RTE_MEMPOOL_ALIGN_MASK)) &
				 RTE_MEMPOOL_ALIGN_MASK);
	}

	/*
	 * increase trailer to add padding between objects in order to
	 * spread them across memory channels/ranks
	 */
	if ((flags & RTE_MEMPOOL_F_NO_SPREAD) == 0) {
		unsigned new_size;
		new_size = arch_mem_object_align
			    (sz->header_size + sz->elt_size + sz->trailer_size);
		sz->trailer_size = new_size - sz->header_size - sz->elt_size;
	}

	/* this is the size of an object, including header and trailer */
	sz->total_size = sz->header_size + sz->elt_size + sz->trailer_size;

	return sz->total_size;
}
```

```c
/* return the number of memory channels */
unsigned rte_memory_get_nchannel(void)
{
	return rte_eal_get_configuration()->mem_config->nchannel;
}

/* return the number of memory rank */
unsigned rte_memory_get_nrank(void)
{
	return rte_eal_get_configuration()->mem_config->nrank;
}

/*
 * Depending on memory configuration on x86 arch, objects addresses are spread
 * between channels and ranks in RAM: the pool allocator will add
 * padding between objects. This function return the new size of the
 * object.
 */
static unsigned int arch_mem_object_align(unsigned int obj_size)
{
	unsigned nrank, nchan;
	unsigned new_obj_size;

	/* get number of channels; 配置的channel的个数：获取配置的eal参数中channel的值，即 -n的值 */
	nchan = rte_memory_get_nchannel();
	if (nchan == 0)
		nchan = 4;

	/* 配置的rank的个数：获取配置的eal参数中rank的值，即 -r的值  */
	nrank = rte_memory_get_nrank();
	if (nrank == 0)
		nrank = 1;

	/* process new object size */
	new_obj_size = (obj_size + RTE_MEMPOOL_ALIGN_MASK) / RTE_MEMPOOL_ALIGN;
	while (get_gcd(new_obj_size, nrank * nchan) != 1)
		new_obj_size++;
	return new_obj_size * RTE_MEMPOOL_ALIGN;
}
```

![](attachments/Pasted%20image%2020250421144144.png)
![](attachments/Pasted%20image%2020250421144303.png)
![](attachments/Pasted%20image%2020250421144336.png)

从 `arch_mem_object_align()` 的逻辑来看，是要让 `mempool的 ele`（此中以mbuf的 mempool 为例子） 的大小（大小：包含头，尾，以及实体）与 `nchannel * nrank`的值互为质数，即它们的最大公约数为 1。
那么，为什么要这样限定？该函数的注释也给出了部分解释，我的理解是：==在 x86 上物理地址和 `<channel_ID，rank_ID>`之间存在映射==，
为了尽可能提高访存的并行度， `mempool的 ele`（此中以mbuf的 mempool 为例子） 之间的首地址最好不要映射到同一对 `<channel_ID，rank_ID>`上。

![](attachments/Pasted%20image%2020250421143543.png)

### channel的并发性
为了充分利用内存带宽，就需要充分利用系统多个内存channel（通道）的并发性。
因为每个bank只能同时被一个channel访问，需要`mempool的obj元素`分布在不同的bank中。
即：==尽可能的让`mempool的obj元素`分散到不同的channel的不同的bank中==。

### 物理地址和channel以及rank的关系
==在 x86 上物理地址和 `<channel_ID，rank_ID>`之间存在映射==，可以基于物理地址，快速的查找到对应的 channel 以及 rank；

### mempool的obj元素的内存对齐是否和cache对齐冲突
不冲突。如代码所示：内存对齐之前，先进行了cacheline的对齐。

# 参考

```bash
# 存储器基本知识：bank，rank，channel
http://blog.chinaunix.net/uid-28541347-id-5795423.html

# dpdk内存管理之内存对齐 （+++++++++）
http://blog.chinaunix.net/uid-28541347-id-5822298.html

# memory-channel.md
https://github.com/youfulife/hexoblog/blob/master/source/_posts/memory-channel.md

# rank介绍
https://www.cnblogs.com/Tohomson/p/18800323

# dpdk中内存channel和rank对mbuf大小的影响
https://blog.csdn.net/choumin/article/details/124414087

# DPDK简介
https://tonydeng.github.io/sdn-handbook/dpdk/introduction.html
```