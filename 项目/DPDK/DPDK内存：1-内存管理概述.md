```table-of-contents
```
# 概述
DPDK 考虑了 NUMA 以及多内存通道的访问效率，会在系统运行前要求配置Linux 的 HugePage，初始化时申请其内存池，用于 DPDK 运行的主要内存资源。Linux 大页机制利用了处理器核上的的 TLB(Translation Lookaside Buffers) 的 HugePage 表项，这可以减少内存地址转换的开销。

# 内存多通道(channel)的使用
现代的内存控制器都支持内存多通道(channel)，比如 Intel 的 E5-2600V3 系列处理器，能支持 4个通道，可以同时读和取数据。依赖于内存控制器及其配置，内存分布在这些通道上。每一个通道都有一个带宽上限，如果所有的内存访问都只发生在第一个通道上，这将成为一个潜在的性能瓶颈。

因此，**DPDK 的 mempool 库缺省是把所有的对象分配在不同的内存通道上**，保证了在系统极端情况下需要大量内存访问时，尽可能地将内存访问任务均匀平滑。

# 内存拷贝( rte_memcpy)
很多 libc 的 API 都没有考虑性能，因此，不要在高性能数据平面上用 libc 提供的 API，比如，memcpy()或 strcpy()。虽然 DPDK 也用了很多 libc 的 API，但均只是在软件配置方面用于方便程序移植和开发。

DPDK 提供了一个优化版本的 **rte_memcpy() API，它充分利用了 Intel 的 SIMD 指令集**，也考虑了数据的对齐特性和 cache 操作友好性。

# 内存动态分配（rte_malloc）
在某些情况下，应用程序使用 libc 提供的动态内存分配机制是必要的，如 malloc()函数，它是一种灵活的内存分配和释放方式。但是，因为管理零散的堆内存代价昂贵，并且这种内存分配器对于并行的请求分配性能不佳，所以**不建议在数据平面处理上使用类似malloc()的函数进行内存分配**。

在有动态分配的需求下，建议**使用 DPDK 提供的 rte_malloc() API**，该 API 可以在后台保证从本 NUMA 节点内存的 HugePage 里分配内存，并实现 cache line 对齐以及无锁方式访问对象等功能。

注：其实**对于数据面转发，其实更加建议的是，使用 mempool，提前申请好固定大小的内存空间**。

# NUMA考虑（memzone/ring/rte_malloc/mempool的per numa分配）
NUMA（Non Uniform Memory Access Architecture）与 SMP（Symmetric Multi Processing）是两种典型的处理器对内存的访问架构。随着处理器进入多核时代，对于内存吞吐量和延迟性能有了更高的要求，NUMA 架构已广泛用于最新的英特尔处理器中，为每个处理器提供分离的内存和内存控制器，以避免 SMP 架构中多个处理器同时访问同一个存储器产生的性能损失。

在双路服务器平台上，NUMA 架构存在本地内存与远端内存的差异。本地和远端是个相对概念，是指内存相对于具体运行程序的处理器而言，如图所示。
![](attachments/Pasted%20image%2020240313215446.png)

在 NUMA 体系架构中，**CPU 进行内存访问时，本地内存的要比访问远端内存更快**。因为**访问远端内存时，需要跨越 QPI 总线**，在应用软件设计中应该尽量避免。在高速报文处理中，这个访问延迟会大幅降低系统性能。

**DPDK 提供了一套在指定 NUMA 节点上创建 memzone、ring, rte_malloc 以及 mempool 的API，可以避免远端内存访问这类问题**。
在一个 NUMA 节点端，对于内存变量进行读取不会存在性能问题，因为该变量会在 CPU cache 里。
但对于跨 NUMA 架构的内存变量读取，会存在性能问题，可以采取复制一份该变量的副本到本地节点（内存）的方法来提高性能。

# 大页内存
## 大页内存作用

从虚拟地址映射到物理地址的转换工作主要是TLB(Translation Lookaside Buffers)与MMU一起完成的。例如，使用4KB的系统页，虚拟地址寻址时，首先在TLB中查找，如果不命中，则需要通过MMU加载的页表基地址进行多次寻表来找到对应的物理地址，如果查不到表，则产生缺页，缺页异常会有相应的handle处理，来填充页表和更新TLB。

TLB的出现是为了减小MMU通过寻址查表带来的CPU开销。TLB容量有限(不同的SoC不同)，系统缺页是不可避免的。而采用大页加上TLB功能，可减小缺页异常，减小TLB miss 。例如，2MB大小的内存访问，如果是4K大小的系统页粒度，则会产生512次缺页异常，但若使用2M的页，一次数据访问只会出发一次缺页异常。页粒度的增大，使得同样大小的TLB空间可以保存更多虚拟地址到物理地址的映射。充分发挥TLB的优势，减小MMU的的使用，减小查表寻址和缺页异常处理的开销，从而提高应用程序的性能。

DPDK之所以称为高性能数据面的应用，一部分原因也得益大页内存的作用。实测发现：
DPDK运行于无大页模式，使用4K页，实测有比较大的TLB miss，而使用大页，TLB miss几乎为0。

## no-huge 非大页支持

DPDK使用大页内存作为性能优化的一个手段。但大页内存在云计算等环境下可能会出现内存资源浪费的情况，作为售卖资源的云服务商，希望能找到更充分的内存资源利用的方法。在此背景下，DPDK引入了no-huge机制，即不使用hugepage，从而解放更多的系统资源。

eal层参数设置了`--no-huge`参数，即无大页模式，也是传统内存分配机制之一。因为在该配置下，使用`RTE_PGSIZE_4K`页大小，将会有很多页，每个页一个文件，文件描述符将非常多，因此会使能`single_file_segments`。以`rte_config:mem_config:memsegs[0] 的 rte_memseg_list`，来关联`--socket_mem`传入的预留的内存数，计算`memseg`个数，并按此预留的内存大小映射到进程。



# 预留内存(`--socket-mem/-m`)
## 没有预留内存的问题

现在新版本的 DPDK默认都是**按需申请大页**，比如系统启动时，一共分配了100个大页，但是DPDK程序启动的时候，不会将100个大页全部占用，而是基于实际需要多少个大页，就从系统中申请多少个大页。

如果DPDK程序存在频繁的配置变更，每次配置下发都需要进行内存申请，然后进行释放。

正常是先从DPDK管理的内存中申请，如果DPDK管理的内存不足，则从系统中申请一个大页；使用完毕之后，DPDK应用将内存还给DPDK管理，DPDK管理时发现管理的内存超过一个大页且连续，可能又将大页还给了系统，即DPDK是按需管理内存。

**如果存在大量的配置变更，DPDK需要频繁的从系统中申请大页，以及将大页还给系统。这个是比较耗时的**。

> 注：在老版本的DPDK，系统申请100个大页，在DPDK系统启动的时候，直接从系统中申请100个大页全部给占用了，实际运行中，不存在频繁的从系统中申请大页以及释放大页给系统。

## DPDK中的预留内存

![](attachments/Pasted%20image%2020260410102527.png)

### `--socket-mem`
```bash
`--socket-mem`: Memory to allocate from hugepages on specific sockets. In dynamic memory mode, this memory will also be pinned (i.e. not released back to the system until application closes).
```



### `--socket-limit`
```bash
--socket-limit：限制dpdk内部使用的最大的内存的大小，不能无限增长。
```

### 双NUMA的预留内存以及内存上限分配

**（1）原则**：

1》**跨 NUMA fallback 的触发条件**：`malloc_heap_alloc_on_heap_id()` 对当前 NUMA 彻底返回 NULL（包括动态扩展失败）之后，才在 `malloc_heap_alloc()` 内的 for 循环中尝试其他 NUMA。

2》**动态扩展优先于跨 NUMA**：`socket-mem` 用完后，先在**同一 NUMA** 向 OS 申请新大页；只有 OS 的本地 NUMA 大页池也耗尽（或者达到 -`-socket-limit` 上限），才会去用另一个 NUMA 的 `socket-mem`。

3》**`--socket-mem` 本身不是硬限制**，`--socket-limit` 才是。设了 `socket-limit` 的情况下，`alloc_more_mem_on_socket()` 会在检查后直接返回 -1，跳过动态申请，更快触发跨 NUMA fallback。


**(2) `--socket-mem` 和  `--socket-limit` 的使用流程**：



```bash
rte_malloc_socket(size, socket=0)
        │
        ▼
heap_alloc_on_socket(socket=0)
        │
        ├─ heap 有空闲 ──────────────────────► 分配成功 ✅
        │
        └─ heap 不足
                │
                ▼
        alloc_more_mem_on_socket()
                │
                ├─ 设置了 --socket-limit
                │   且已达上限 ──────────────► 返回 NULL ❌
                │                              (不跨 NUMA，不申请新页)
                │
                └─ 无 limit 或未达上限
                        │
                        ▼
                向系统申请新大页 (mmap + mbind)
                        │
                        ├─ 系统大页池有余量 ──► 扩展 heap，分配成功 ✅
                        │
                        └─ 系统大页耗尽 ──────► 返回 NULL ❌
                                               (同样不跨 NUMA)
```

**(3） 本numa的`--socket-mem` 不足是否会跨numa使用`--socket-mem` ，还是本NUMA向系统申请内存？

```bash
使用 SOCKET_ID_ANY 申请内存，当前线程在 NUMA 0
          │
          ▼
  尝试从 NUMA 0 现有 heap（socket-mem 预留）分配
          │
          ├─ 成功 ──────────────────────────────► 返回 ✅
          │
          └─ 失败（socket-mem 用完）
                    │
                    ▼
          向系统动态申请新大页（仍在 NUMA 0）
          [eal_memalloc_alloc_seg_bulk + mbind(NUMA 0)]
                    │
                    ├─ 系统 NUMA 0 有剩余大页 ──► 扩展 heap，分配成功 ✅
                    │                             （不会碰 NUMA 1 的内存）
                    │
                    └─ 系统 NUMA 0 大页也耗尽
                                │
                                ▼
                      fallback：尝试 NUMA 1 的 heap
                      （此时才使用 NUMA 1 的 socket-mem）
                                │
                                ├─ NUMA 1 有剩余 ─► 返回 ✅（跨 NUMA）
                                └─ 全部失败 ──────► 返回 NULL ❌
```



### 动态内存的回调
 动态内存模式也支持了一些内存管理的回调机制，主要由两个API。

#### 内存映射更改时的回调：`rte_mem_event_callback_register`

在动态内存模式下应用的内存是会动态增加和减少的，对于有些模块是需要感知这些变化的，如vfio需要将整个内存pin住，所以就需要及时知道新增的内存，并将其pin住。因此DPDK提供了`rte_mem_event_callback_register`这个API，用于关系内存映射变化的模块注册相应函数。

#### 内存超限时的回调函数：`rte_mem_alloc_validator_callback_register`

在动态内存情况下可以通过`--socket-limit`参数来指定当前`socket`所能使用的内存大小上限。
有时候我们不希望应用程序超出这个限制就一定返回失败，但又希望能够感知这种情况，因此可以通过`rte_mem_alloc_validator_callback_register`这个API注册回调函数，当应用程序申请的内存超过`--socket-limit`时注册函数就会被调用，我们可以在函数中输出一下警告信息，并做一些更温和的处理，如：可以接受超出限制的几百兆字节，但拒绝超出限制千兆字节的情况。


# DPDK内存整体架构
参考: [# DPDK内存管理概述](https://zhuanlan.zhihu.com/p/658824633)

## 大页 & memsge & memheap & memzone & mempool的关系

![](attachments/Pasted%20image%2020260409154338.png)

如上图所示，DPDK的内存管理框架，**从底层往上依次为：`大页系统---> memseg ----> malloc heap ---> memzone ---> mempool` **； 

![](attachments/Pasted%20image%2020260409223652.png)

memseg 表示一个单独的物理大页（物理连续）；


### mbuf

rte_mbuf的内存结构如下图：
![](attachments/Pasted%20image%2020231025111714.png)

```bash
（1）rte_mbuf:
	存储的是元数据的信息。
	mbuf元信息是需要被频繁访问的部分, 它位于mbuf头部, 且被设计地足够小, 目前占用两个cache lines, 其中访问最频繁的信息位于第一个cache line. mbuf元信息由数据结构rte_mbuf表示. 

（2）priv_data:
	一般是用户自己对于rte_mbuf的一层封装结构（第一个成员为 rte_mbuf）的剩余部分；
	如果用户还需要存储业务相关的其他数据, 可以放在mbuf的 priv_data 或者 headroom中, 它是 rte_mbuf与数据之间的一块内存区域.
	
（3）headroom:
	headroom默认大小为128byte，可以通过RTE_PKTMBUF_HEADROOM调整。

（4）data:
	data 里面存放的是真正的网络数据包，包括ether/ip/tcp首部，以及payload。

（5）tailroom:
	tailroom 是一块空闲区域，data 越大，tailroom就越小，data 越小，tailroom就越大
```

### mempool

mempool是DPDK存储固定大小对象的内存池，rte_mbuf就是一种固定大小的对象，也就是说mempool里面也可以是其他对象，只要是固定大小就行。

#### pktbuf pool 和 mbuf的关系
`pktbuf pool`: 基于mempool管理的`mbuf pool`；


![](attachments/Pasted%20image%2020231025113346.png)

![](attachments/Pasted%20image%2020260409224623.png)

需要注意的是，**一个mempool占用的物理内存不一定连续**，且可能会跨多个大页，但是**一个rte_mbuf占用的肯定是连续物理内存**。

创建mempool并不是直接从大页上分配内存，而是通过memzone间接申请。


### memzone

memzone: DPDK中一块**有名字的、物理地址连续**的内存区域，在对应的memheap中申请的一块内存，并管理申请的内存信息。
本质是 heap 上的大块物理地址连续的内存，带元数据（名称、地址、长度、socket、IOVA）。

每个memzone描述的都是一块连续的物理内存，memzone结构本身并不在大页上，但是它描述的连续物理内存在大页上。

#### 接口

网卡驱动的硬件描述符的内存申请接口`rte_eth_dma_zone_reserve()`，`rte_ring`和`rte_mempool`的申请接口，底层都是申请`memzone`，其基本接口是`rte_memzone_reserve_aligned()`。

```c
struct rte_memzone * rte_memzone_reserve_aligned(const char *name, size_t len, int socket_id,
                unsigned flags, unsigned align);
                
int rte_memzone_free(const struct rte_memzone *mz);
```


以下rte_mempool创建过程做个简单示意：

![](attachments/Pasted%20image%2020260424161902.png)

memzone所关联的内存都是来自与malloc_heap中malloc_elem关联的大页内存。


#### 特点
命名 + 跨进程可见：主 / 从进程可按名查找。
强控制：**指定 socket、对齐、边界、物理连续、IOVA 连续**。


#### 应用
memzone的使用场景：当应用需要一块有名字、可全局访问的连续大内存时使用，主要用于系统初始化阶段。
- 网卡队列、设备驱动、共享内存、**需要物理连续的 DMA**。
- 不适合频繁小块分配。

==**典型用途**：`mempool`、`rte_ring`、`rte_hash` 等高层对象的**底层存储载体**==，即创建`mempool`、`rte_ring`等结构体时，使用的内存空间都是从 memzone 中申请的；
对于 mempool 而言，其实mempool中的元素也是在`memzone`中申请的，即： 按照一次申请一批元素（但不是全部元素）进行申请，按照 `ele_total_len「ele_hdr + ele + ele_tailer」 * N `从memzone中申请，而不是一整块元素进行申请。这样所有元素，其实是多块地址连续的块，但是整体不是一个地址连续的整块。


#### memzone 和 mempool
创建mempool并不是直接从大页上分配内存，而是通过memzone间接申请。

一个mempool可能只消耗一个memzone，意思是一个memzone描述的连续物理内存大小可以cover创建mempool的需求。但是当一个memzone不能满足mempool的需求时，一个mempool就需要消耗多个memzone。假设一个memzone描述的最大连续物理内存只有1G，但是用户创建mempool时指定需要4G内存，那么底层实现就需要多个memzone，这时候mempool就是由多个物理连续的内存快组成。

### memheap

Memheap：是DPDK的通用**堆内存分配器**，内部实现类似Linux的伙伴系统（Buddy System），负责管理Memseg。 把 memseg 切成小块（`malloc_elem`），按**大小分级链表**管理空闲块。支持**动态切割、合并、对齐、边界限制**。


DPDK里面设计了5种不同大小的堆链表，如图5所示。大小为0 ～ 2^8之间的**可用连续物理内存块**都会挂在这个链表上。大小为2^8 ～ 2^10之间的**可用连续物理内存块**都会挂载在另外一个链表上。其他链表挂载同理。需要注意的是链表里面的element都是可被申请的。

![](attachments/Pasted%20image%2020260409224830.png)

上文提到一个memzone描述的是一段连续物理内存块，那么memzone描述的内存块是从哪里来的，就是从heap上申请的。
申请时会根据申请的大小找到对应的链表里面能满足需求的最小element，每个element都是一块连续物理内存。一旦申请后，这块被申请的内存将从heap的链表里面的element切割出去。

那被申请出去的内存块如何管理？DPDK的做法是谁申请，谁管理，比如通过memzone申请，那就由memzone管理，memzone结构里面会有指针指向这块被申请出去的内存。

#### 应用
用于rte相关的申请、释放内存。比如：rte_malloc/rte_free; 

**使用场景**：
- 控制面**动态内存**（初始化、配置、非频繁分配）DPDK。
- 不适合**高频数据面**（有锁、碎片、分配慢）。

#### 特点
**(1) 每个socket一个heap，外部内存也可以注册为heap**
每个 NUMA socket 一个 heap，本地分配减少跨 NUMA 访问。
> 注：如果是2个numa，就有2个heap，即heap_id为0和1；对于外部的内存，也可以注册为heap，生成新的heap_id;

**(2)  管理所在heap下，多个不同内存大小的 `free_mem list`。即：每个heap（每个heap_id）下，都有多个不同大小的 free_mem的链表**。

2.1> 起始时，会将初始化的`mem seg list`，按照所在的socket，存放在对应的heap中。

每个heap中，按照不同的内存大小（256B、1kB、4kB、16kB...1GB、4GB），管理多个`free_mem list`。

初始化时，每个`memseg list`作为一个大的空闲内存块，作为一个`malloc_elem`管理起来，放在一个对应的`free mem list`中。

2.2> 当要从heap中申请内存时，按照内存大小，到对应的`free mem list`中申请。如果对应`free mem list`为空，则找一个较大的`free mem list`申请。

2.3> 在一个`free mem list`的成员，即一个`malloc_elem`中申请内存时，根据申请的大小，以及对齐要求，可能会把`malloc_elem`切分为两个`malloc_elem`或三个`malloc_elem`。其中一个返回给上层应用使用，剩下的两个重新按照大小，放在对应的`free mem list`中。

### memseg

memseg：按照配置参数，创建大页文件，映射大页内存地址。一个segment表示一块连续的物理内存（一个大页），也就是上文heap链表里面的element。

**memseg 本质**：DPDK 内存管理的最底层结构，是对操作系统锁定的 Hugepage 物理内存块的描述，对应结构体 `rte_memseg`。

```c
struct rte_memseg {
    rte_iova_t  iova;       // 物理/IO 地址（DMA 用）
    void       *addr;       // 虚拟地址
    size_t      len;        // 段长度
    uint64_t    hugepage_sz;// 页大小 2MB/1GB
    int32_t     socket_id;  // NUMA 节点
    uint32_t    nchannel;   // 内存通道数
    uint32_t    nrank;      // 内存 rank 数
};
```

#### 特点
(1) 按照配置参数--socket-mem或-m，计算出各个socket需要的各种大页个数。优先使用页面大的大页
(2) 每个大页对应一个mem seg，需要在指定的大页文件系统目录下，创建一个大页文件。文件名为`<hugefile-prefix>map_<index>`
(3) **相同页面大小的`mem seg`放在一个`mem seg list`中管理**。由于一个`mem seg list`中存放的`mem seg`有最大值限制，因此，存放相同页面大小的`mem seg`，可能会存放在多个`mem seg list`中。


#### memseg 和 大页的关系

![](attachments/Pasted%20image%2020260409221319.png)

每个大页类型(比如：2M的大页，1G的大页)有多个`rte_memseg_list`管理，每个`rte_memseg_list`关联多个相同大页大小的`memseg`，每个`memseg`记录一个大页。

具体为：
`rte_memseg_list`中的`base_va`指向连续N个大页的起始虚拟地址。memseg中的addr则指向该大页在进程中的虚拟地址，iova则是该大页通过SMMU映射前的iova(在进行SMMU的映射时，iova使用了虚拟地址addr来映射，即VA模式下iova与addr是相同的)。

一个`rte_memseg_list`下`rte_fbarray`管理着这些`segment`。记录该`rte_memseg_list`中哪些memseg可用，哪些已被使用。每个memseg_list都对应一个全局的fd_list保存了该`memseg_list`中所有`memseg`关联大页的大页文件`rte_leemap_X`的文件描述符。


### 大页
大页系统。
**(1)获取系统的大页信息。包括各种大页类型，及其可用大页的数目**。

(2)获取大页文件系统mount的目录(可以参数指定，`--huge-dir <path to hugetlbfs directory>`)，将要创建的大页文件的前缀`<hugefile-prefix>`（可以参数指定，`--file-prefix`）

(3)**每个大页文件系统指定的页的大小是固定的，因此不同页大小的大页，mount的目录是不一样的**。


#### 大页内存申请

动态申请大页的含义及流程，请见如下图所示：

![](attachments/Pasted%20image%2020260424162042.png)

从以上申请内存的接口实现上可以看出，rte_mempool、rte_ring、pkt_mbuf和rte_malloc系列函数对应的内存，来源实际都是通过heap中存放的mmap到进程的大页内存。

但要注意，==DPDK中不是所有的场景都需要使用rte_malloc系列函数申请内存，在控制面上，一般直接使用malloc等C库函数申请内存==即可。



# 外部内存和内部内存
## 内部内存 (预留内存：`--socket-mem/-m`)
DPDK启动的时候，通过`--socket-mem` 以及 `--socket-limit` 启动参数，**预留的内存**，可以认为是内部内存。
此中的DPDK启动，按照`--socket-mem` 的配置在NUMA0和NUMA1上申请大页（对应memseg-->memheap）。
后续的`memzone/mempool/rte_ring/rte_malloc`， 如果没有特别指定`heap`，就是默认从这里内部内存分配的。

## 外部内存
在DPDK新版本中可以支持将外部内存注册给DPDK内存管理，比如自己应用通过非DPDK API malloc的内存或者mmap的内存。将这部分内存注册进DPDK的内存管理中，同样可以使用DPDK的内存API进行访问。
用户在DPDK外部申请的内存，通过DPDK的`heap`接口(`rte_malloc_heap_create, rte_malloc_heap_memory_add`)，可以注册给DPDK来管理 ，后面的`memzone/mempool/rte_malloc`可以指定外部的heap，从外部内存中来分配。


|函数|作用|
|---|---|
|`rte_malloc_heap_create()`|创建一个命名 heap|
|`rte_malloc_heap_memory_add()`|向已有 heap **追加**内存块|
|`rte_malloc_heap_memory_remove()`|从 heap 中移除某段内存|
|`rte_malloc_heap_destroy()`|销毁 heap|
|`rte_malloc_heap_alloc()`|从指定 heap 分配内存|




### 注册外部内存

（1）**外部非大页内存**
```c
void *addr = aligned_alloc(2 * 1024 * 1024, size);  // 2MB 对齐
rte_extmem_register(addr, size, NULL, 0, RTE_PGSIZE_2M);

// DMA 映射，如果这块内存有DMA操作（如 mbuf、rings等），就需要DMA映射，否则不需要。
rte_dev_dma_map(device, virt_addr, 0, len); 
```

（2）**外部大页内存**
```c
    unsigned flag = (21UL << MAP_HUGE_SHIFT) | MAP_PRIVATE | MAP_ANONYMOUS;
    flag |= MAP_HUGETLB;

    orig_addr = mmap(NULL,
                     kcontext.alloc_buff_size + KUCL_MANAGED_HEAP_PGSIZE,
                     PROT_READ | PROT_WRITE, flag, -1, 0);

	rte_extmem_register(orig_addr, kcontext.alloc_buff_size + KUCL_MANAGED_HEAP_PGSIZE, NULL, 0, RTE_PGSIZE_2M);
```

#### rte_extmem_register
```c
int rte_extmem_register(void *va_addr, size_t len,
                        rte_iova_t *iova_addrs,
                        unsigned int n_pages,
                        size_t page_sz);
```
`rte_extmem_register` 的主要作用是将**外部内存注册到 DPDK 的内存子系统（memseg list），使其成为 DPDK 可管理的 memseg。**，让 DPDK 的基础设施"感知"这段内存的存在。


**(1)内部机制**
调用该接口后，DPDK 会：
- **创建 memseg 描述符**
    - 将外部内存纳入 DPDK 的 `memseg_list` 管理。使 `rte_mem_virt2iova()`、`rte_mem_virt2memseg()` 等工具函数能正常工作
- **建立 虚拟地址VA → IOVA 的映射关系**：记录每个页的 IOVA
    - 在 `IOVA=VA` 模式下，IOVA 即虚拟地址。
    - 在 `IOVA=PA` 模式下，需要提供每一页的物理地址。

注册后，这些内存可以被：
- `rte_malloc_heap_memory_add()` 添加到 DPDK 的 heap 中；
- `rte_mempool` 用于创建对象池；
- 设备进行 DMA（前提是完成 IOVA 映射）。


**（2）是否必须调用 rte_extmem_register？**
![](attachments/Pasted%20image%2020260410113743.png)

**（3）通过 `mmap` 申请的大页内存是否需要注册？**

![](attachments/Pasted%20image%2020260410113945.png)

如果这块外部内存需要DPDK管理，那么是否需要 调用： rte_extmem_register
![](attachments/Pasted%20image%2020260410114111.png)

#### rte_dev_dma_map
`rte_dev_dma_map` 的作用是将一段内存**映射到指定设备的 IOMMU/DMA 地址空间**，让该设备可以通过 DMA 直接访问这块内存。
DMA 映射：如果这块内存有DMA操作（如 mbuf、rings等），就需要DMA映射，否则不需要。

```bash
没有 dma_map：
  NIC/设备 ──DMA──► ??? 访问失败 或 IOMMU 拦截

有 dma_map 之后：
  NIC/设备 ──DMA──► [IOMMU 映射表] ──► 物理内存 ✅
```


![](attachments/Pasted%20image%2020260410113248.png)


#### 完整的外部内存使用流程
```c
/* Step 1: 申请大页内存 */
void *mem = mmap(NULL, size, PROT_READ|PROT_WRITE,
                 MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB, -1, 0);

/* Step 2: 获取每个页的 IOVA（物理地址） */
rte_iova_t iova_arr[N];
// 通过 /proc/self/pagemap 或 rte_mem_virt2iova() 获取

/* Step 3: 注册到 DPDK 内存子系统（二选一）*/
// 方式A：只注册，自己管理分配
rte_extmem_register(mem, size, iova_arr, N, page_sz);
// 方式B：注册 + 交给 heap 管理（内部自动注册）
rte_malloc_heap_memory_add("my_heap", mem, size, iova_arr, N, page_sz);

/* Step 4: VFIO+IOMMU 场景下，映射给设备做 DMA */
if (使用VFIO且需要DMA) {
    struct rte_device *dev = ...; // 获取设备句柄
    rte_dev_dma_map(dev, mem, iova_arr[0], size);
    // 或者用 rte_extmem_register + 让 DPDK 自动 map 所有设备：
    // rte_dev_dma_map 需要对每个设备调用一次
}

/* Step 5: 使用内存 */
void *buf = rte_malloc_heap_alloc("my_heap", NULL, buf_size, 0, 0, -1);
```

### 外部内存交给heap管理
将注册的外部内存加入到某个 heap 中：
```c
int rte_malloc_heap_create(const char *heap_name);
int rte_malloc_heap_memory_add(const char *heap_name, void *va_addr, size_t len,
        rte_iova_t iova_addrs[], unsigned int n_pages, size_t page_sz);
```

**页对齐要求**：添加的外部内存必须按 `page_sz` 对齐，且大小是 page_sz 的整数倍。
**IOVA 地址**：调用 `rte_malloc_heap_memory_add` 时有可能需要提供每个页的 IOVA（物理/总线地址）数组`iova_addrs`。
- **IOVA=VA**：通常不需要提供 `iova_addrs`。
- **IOVA=PA**：必须提供每一页的物理地址，否则 DMA 无法使用。

### heap 动态扩容
DPDK 的 heap 本质上是一个 **freelist 链表**，`memory_add` 就是往链表里插入新的空闲块，因此扩展次数没有硬性限制，只要系统内存足够即可。

```c
/* 第一步：创建 heap */
rte_malloc_heap_create("my_heap");

/* 第二步：添加初始的 512MB 外部内存 */
void *mem1 = mmap(NULL, 512MB, ...);   // 外部申请
rte_malloc_heap_memory_add("my_heap", mem1, 512MB, iova_addrs, n_pages, page_sz);

/* 第三步：从 heap 中分配 */
void *obj = rte_malloc_heap_alloc("my_heap", "type", size, align, 0, SOCKET_ID_ANY);

/* 第四步：512MB 不够用了，动态追加更多内存 */
void *mem2 = mmap(NULL, 256MB, ...);   // 再申请一块
rte_malloc_heap_memory_add("my_heap", mem2, 256MB, iova_addrs2, n_pages2, page_sz);

/* 此后 heap 总容量变为 768MB，可继续分配 */
```

**内存碎片**：多次追加不连续的内存块，heap 内部会以多个 freelist segment 管理，分配**大块内存**时可能因碎片失败，这和标准 malloc 行为一致。

典型的场景：
```bash
初始状态：   heap → [seg1: 512MB]
触发扩展后： heap → [seg1: 512MB] + [seg2: 256MB]  ← 动态追加
再次扩展：   heap → [seg1: 512MB] + [seg2: 256MB] + [seg3: 512MB]
```

#### 如何感知需要扩容
在 **DPDK 的 `rte_malloc` heap** 中，**扩容（添加新的外部内存）并不会自动触发**，而是需要由应用程序根据运行时的内存使用情况来“感知”并主动执行扩展操作。DPDK 本身不会像操作系统的 `malloc` 一样自动向系统申请更多内存。

**模式一：失败即扩容（懒扩容）**


### heap 动态缩容

**移除限制**：`rte_malloc_heap_memory_remove` 要求该段内存**没有任何已分配的块**，否则会返回失败。所以动态缩容需要确保该段内存已全部释放。

# 参考
```bash
# DPDK内存管理概述
https://zhuanlan.zhihu.com/p/658824633
https://zhuanlan.zhihu.com/p/702445686

# DPDK 22.11内存管理变化解析 [++++++++]
http://blog.chinaunix.net/uid-28541347-id-5877488.html

# 深入理解DPDK：内存管理模块整体分析 （代码级别讲解；+++++）
https://zhuanlan.zhihu.com/p/712450456

# DPDK内存管理——全网最全篇
https://zhuanlan.zhihu.com/p/702445686

# DPDK内存管理分析
https://blog.csdn.net/weixin_48329334/article/details/139059952

# 一文读懂DPDK rte_mempool创建、使用及信息查询
https://zhuanlan.zhihu.com/p/695112706

--socket-mem 参数说明
https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html
https://doc.dpdk.org/guides/linux_gsg/build_sample_apps.html

# DPDK性能影响因素分析
https://zhuanlan.zhihu.com/p/557294705


```