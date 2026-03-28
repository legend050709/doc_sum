```table-of-contents
```
# 概述
DPDK 是一个为高速网络环境而设计的工具库，通过绕过内核（`kernel bypsss`）的方式获得强大的数据处理性能。DPDK 为了获得高吞吐，充分利用了硬件特点并灵活使用了各种数据结构，本文将探索其中的一个重要数据结构——无锁队列。

# 无锁环形队列的特点
无锁环形队列就能理解这是一种环形队列，它没有锁。队列是一种我们常见的数据结构，保证了元素的先进先出，环形队列则是头尾相连，它是一个环形，通常呢是一个固定大小的闭环队列。
![](attachments/Pasted%20image%2020250412113159.png)

它的特点也很明显：

- 它能够保证数据是**先进先出**的，这部分是由队列来保证的。
- 由于是固定的的一个闭环，一开始我们可以规定好队列的元素个数大小，同时当生产者把队列放满时会选择丢弃掉，当然我们也可以设计放在另一个缓冲区处理。
- **内存空间可以重复利用**，避免内存分配和释放的开销。


# 设计理念
无锁环形队列作为一个常用数据结构，在网络应用程序中更是必不可少。

## 队列使用双向链表的优缺点
（1）优点
队列通常用双向链表来实现，得益于链表的特点，基于链表的队列具有不浪费内存、易于扩展的优点。

（2）缺点
在追求性能的并发的场景下，链表的实现会产生诸多缺点：
1. 链表的增删操作需要多次对指针进行赋值，需要加锁来避免并发错误，使用锁会大大降低性能。
2. 数据局部性差，频繁分配和回收内存有开销。链表元素通常分配在堆区，分散在内存中，数据局部性不佳，尤其是如果需要一次入（出）队多个元素时，可能造成多次 cache 不命中。
3. 为了消除这些缺点，实现起来会非常复杂。

## 队列使用数组的优缺点

基于这些考量，DPDK 中的队列（ring）没有使用链表，而是使用一段连续的数组做队列。
这样带来的缺点就是队列大小固定，不可扩展，且元素较少时占用了不必要的内存。但为了性能，DPDK 让内存首先做了牺牲，同时考虑到网络程序往往都是固定的缓冲区大小，扩展性相对来说也可以牺牲。

此外，连续数组可以应对基于链表的队列的缺点。由于**固定大小且连续，局部性良好**，无分配内存开销；
相比于链表增删操作需要多次赋值，数组只需要一次赋值即可实现入队和出队，因此可以用 **Compare-and-Swap （CAS）来替代锁**，提高效率。

## 小结
无锁环形队列使用连续数组：
（1）队列大小固定，不可扩展，且元素较少时占用了不必要的内存。
（2）局部性良好。
（3）只需一次赋值即可实现入队和出队，可以用 **CAS** 来替代锁。


# 基本原理
首先定义进行入队操作的进程为生产者（producer），进行出队操作的进程为消费者（consumer）。
DPDK 的 ring（以下简称 ring）设置了四个指针来标记队头和队尾，分别是

1. “生产者头”: `prod_head`
2. “生产者尾”: `prod_tail`
3. “消费者头”: `cons_head`
4. “消费者尾”: `cons_tail`

这个名称可能有些令人迷惑，但我们可以先简单地认为：**大部分时间里，生产者头尾相同，消费者头尾相同，且生产者头尾指向 ring 的队尾，消费者头尾指向 ring 的队头**。


|变量名称|含义|
|---|---|
|g_prod_head|环形队列生产者head|
|g_prod_tail|环形队列生产者tail|
|g_cons_head|环形队列消费者head|
|g_cons_tail|环形队列消费者tail|
|l_prod_head|临时队列生产者head位置|
|l_prod_tail|临时队列生产者tail位置|
|l_cons_head|临时队列消费者head位置|
|l_cons_tail|临时队列消费者tail位置|
|l_prod_head_next|临时存放生产者需要移动到的位置。|
|l_cons_tail_next|临时存放生产者需要移动到的位置|

> 为什么会有临时变量的存在？
- 因为每个CPU的core 都独占一份内存，这些临时变量存储在core的本地cache中尚未更新到全局的环形队列上的值。环境队里中的变量则是存储共享的环形队列值。

## 生产者/消费者模式
```c

#define RING_F_SP_ENQ 0x0001 /**< The default enqueue is "single-producer". */
#define RING_F_SC_DEQ 0x0002 /**< The default dequeue is "single-consumer". */
/**
 * Ring is to hold exactly requested number of entries.
 * Without this flag set, the ring size requested must be a power of 2, and the
 * usable space will be that size - 1. With the flag, the requested size will
 * be rounded up to the next power of two, but the usable space will be exactly
 * that requested. Worst case, if a power-of-2 size is requested, half the
 * ring space will be wasted.
 */
#define RING_F_EXACT_SZ 0x0004
#define RTE_RING_SZ_MASK  (0x7fffffffU) /**< Ring size mask */

#define RING_F_MP_RTS_ENQ 0x0008 /**< The default enqueue is "MP RTS". */
#define RING_F_MC_RTS_DEQ 0x0010 /**< The default dequeue is "MC RTS". */

#define RING_F_MP_HTS_ENQ 0x0020 /**< The default enqueue is "MP HTS". */
#define RING_F_MC_HTS_DEQ 0x0040 /**< The default dequeue is "MC HTS". */



/** prod/cons sync types (st) */
enum rte_ring_sync_type {
    RTE_RING_SYNC_MT,     /**< multi-thread safe (default mode) */
    RTE_RING_SYNC_ST,     /**< single thread only */
    RTE_RING_SYNC_MT_RTS, /**< multi-thread relaxed tail sync */
    RTE_RING_SYNC_MT_HTS, /**< multi-thread head/tail sync */
};

```

`rte_ring`接口中的`prod.sysc_type`在这里用来区分模式。先来看sysc_type怎么设置的。
```c
static int get_sync_type(uint32_t flags, enum rte_ring_sync_type *prod_st,
    enum rte_ring_sync_type *cons_st)
{
    static const uint32_t prod_st_flags =
        (RING_F_SP_ENQ | RING_F_MP_RTS_ENQ | RING_F_MP_HTS_ENQ);
    static const uint32_t cons_st_flags =
        (RING_F_SC_DEQ | RING_F_MC_RTS_DEQ | RING_F_MC_HTS_DEQ);

    switch (flags & prod_st_flags) {
    case 0:
        *prod_st = RTE_RING_SYNC_MT;
        break;
    case RING_F_SP_ENQ:
        *prod_st = RTE_RING_SYNC_ST;
        break;
    case RING_F_MP_RTS_ENQ:
        *prod_st = RTE_RING_SYNC_MT_RTS;
        break;
    case RING_F_MP_HTS_ENQ:
        *prod_st = RTE_RING_SYNC_MT_HTS;
        break;
    default:
        return -EINVAL;
    }

    switch (flags & cons_st_flags) {
    case 0:
        *cons_st = RTE_RING_SYNC_MT;
        break;
    case RING_F_SC_DEQ:
        *cons_st = RTE_RING_SYNC_ST;
        break;
    case RING_F_MC_RTS_DEQ:
        *cons_st = RTE_RING_SYNC_MT_RTS;
        break;
    case RING_F_MC_HTS_DEQ:
        *cons_st = RTE_RING_SYNC_MT_HTS;
        break;
    default:
        return -EINVAL;
    }

    return 0;
}
```
默认生产者模式：`RTE_RING_SYNC_MT`  
默认消费者模式：`RTE_RING_SYNC_MT`

如果是单生产单消费者，`flag`会被设置为`RING_F_SP_ENQ | RING_F_SC_DEQ`，
对应的`prod.sync_type`为  `RTE_RING_SYNC_ST` ；`cons.sync_type`为`RTE_RING_SYNC_ST`。


## `rte_ring` 数据结构
```c
/**
 * structures to hold a pair of head/tail values and other metadata.
 * Depending on sync_type format of that structure might be different,
 * but offset for *sync_type* and *tail* values should remain the same.
 */
struct rte_ring_headtail {
	volatile uint32_t head;      /**< prod/consumer head. */
	volatile uint32_t tail;      /**< prod/consumer tail. */
	RTE_STD_C11
	union {
		/** sync type of prod/cons */
		enum rte_ring_sync_type sync_type;
		/** deprecated -  True if single prod/cons */
		uint32_t single;
	};
};

struct rte_ring {
	char name[RTE_RING_NAMESIZE] __rte_cache_aligned;
	/**< Name of the ring. */
	int flags;               /**< Flags supplied at creation. */
	const struct rte_memzone *memzone;
			/**< Memzone, if any, containing the rte_ring */
	uint32_t size;           /**< Size of ring. */
	uint32_t mask;           /**< Mask (size-1) of ring. */
	uint32_t capacity;       /**< Usable size of ring */

	char pad0 __rte_cache_aligned; /**< empty cache line */

	/** Ring producer status. */
	RTE_STD_C11
	union {
		struct rte_ring_headtail prod;
		struct rte_ring_hts_headtail hts_prod;
		struct rte_ring_rts_headtail rts_prod;
	}  __rte_cache_aligned;

	char pad1 __rte_cache_aligned; /**< empty cache line */

	/** Ring consumer status. */
	RTE_STD_C11
	union {
		struct rte_ring_headtail cons;
		struct rte_ring_hts_headtail hts_cons;
		struct rte_ring_rts_headtail rts_cons;
	}  __rte_cache_aligned;

	char pad2 __rte_cache_aligned; /**< empty cache line */
};
```

## 创建`rte_ring`
如下所示，创建 rte_ring的时候，实际的内存是 `sizeof(struct rte_ring) + (ssize_t)count * sizeof(void*);`。

即：`struct rte_ring` 后面其实就是一个大小为`count`的指针数组，用于存储这个ring中的元素。
单/多生产者以及单/多消费者，就是将元素放入到这个数组，或者从这个数组中取元素。 

```bash
struct rte_ring * rte_ring_create(const char *name, unsigned int count, int socket_id,
		unsigned int flags)
{
	return rte_ring_create_elem(name, sizeof(void *), count, socket_id,
		flags);
}


struct rte_ring * rte_ring_create_elem(const char *name, unsigned int esize, unsigned int count,
		int socket_id, unsigned int flags)
{
	...
	ring_size = rte_ring_get_memsize_elem(esize, count);
	if (ring_size < 0) {
		rte_errno = -ring_size;
		return NULL;
	}
	...
}

ssize_t rte_ring_get_memsize_elem(unsigned int esize, unsigned int count)
{
	ssize_t sz;

	/* Check if element size is a multiple of 4B */
	if (esize % 4 != 0) {
		RTE_LOG(ERR, RING, "element size is not a multiple of 4\n");

		return -EINVAL;
	}

	/* count must be a power of 2 */
	if ((!POWEROF2(count)) || (count > RTE_RING_SZ_MASK )) {
		RTE_LOG(ERR, RING,
			"Requested number of elements is invalid, must be power of 2, and not exceed %u\n",
			RTE_RING_SZ_MASK);

		return -EINVAL;
	}

	sz = sizeof(struct rte_ring) + (ssize_t)count * esize;
	sz = RTE_ALIGN(sz, RTE_CACHE_LINE_SIZE);
	return sz;
}
```

### rte_ring_create 和 rte_ring_create_elem


|维度|rte_ring_create|rte_ring_create_elem|
|---|---|---|
|元素大小|固定为 `sizeof(void *)`（指针大小，64 位系统 8 字节）|自定义元素大小（bytes），可设任意值|
|设计初衷|早期版本接口，仅支持存储指针|新版增强接口，支持存储任意结构 / 数据块|
|内存分配|内部按「指针数」分配内存|按「元素数 × 自定义元素大小」分配内存|
|核心操作接口|`rte_ring_enqueue/dequeue`（操作指针）|`rte_ring_enqueue_elem/dequeue_elem`（操作自定义元素）|
|兼容性|兼容所有 DPDK 版本|DPDK 18.05+ 引入，低版本无此接口|

简单来说，**`rte_ring_create` 是 `rte_ring_create_elem` 在 `esize = sizeof(void *)` 时的特例**。

|场景|rte_ring_create（指针）|rte_ring_create_elem（自定义元素）|
|---|---|---|
|存储指针（如 mbuf）|最优（原生设计）|无优势，且浪费内存（元素大小≥指针）|
|存储小数据块（≤64B）|需指针解引用（缓存 miss 风险）|更优（数据直接存在 ring 内存，缓存友好）|
|多生产者 / 多消费者|无锁性能一致|无锁性能一致（底层无锁逻辑相同）|
|内存开销|小（仅存指针）|大（元素数 × 自定义大小）|
|操作耗时|极短（仅拷贝指针）|随元素大小增加（拷贝数据块）|


其他操作：

|操作|传统 API（基于指针）|元素 API（基于大小）|
|---|---|---|
|入队（单对象）|`rte_ring_enqueue(r, ptr)`|`rte_ring_enqueue_elem(r, obj, esize)`|
|出队（单对象）|`rte_ring_dequeue(r, &ptr)`|`rte_ring_dequeue_elem(r, obj, esize)`|
|批量入队|`rte_ring_enqueue_bulk(r, ptr_array, n, NULL)`|`rte_ring_enqueue_bulk_elem(r, obj_array, esize, n, NULL)`|
|批量出队|`rte_ring_dequeue_bulk(r, ptr_array, n, NULL)`|`rte_ring_dequeue_bulk_elem(r, obj_array, esize, n, NULL)`|



## 环形队列的缓冲区大小
### 背景
使用**连续数组**作为队列环形缓冲区，涉及到取模运算来保证下标始终在数组的范围内，并且需要**始终留空一个位置以判断是否用满**。
因此需要用取模运算来处理，例如计算 `prod_next` 和计算队列长度时，都需要进行取模运算（结果为正数）：

```c
prod_next = (prod_head + 1) % buffer_size
queue_length = (ring->prod_tail - ring_cons_head) % buffer_size
```
### 优化
为了能够简化这个过程，避免取模的开销，DPDK 直接**约定整个 ring 的空间大小必须是 2 的幂**，也就是说实际可容纳的元素数量为  `2^n−1` 个元素，并设定变量 `mask` 的值为 `2^n−1` 。在计算 `prod_next` 时也无需取模，直接加 1。

在放入元素时，不是直接用 `prod_next` 作为下标，而是 `prod_next & mask`，因为规定了 `mask` 是  `2^n−1`，也就是说 `mask` 的二进制是 n 个 1，通过一个按位与操作即能够用对应了取模操作，计算速度非常快，并由此获得了合理的下标。

而计算队列长度时，也同样不需要模运算，而替换成 `(ring->prod_tail - ring_cons_head) & mask` 即可。

#### 说明
（1）优点
这样操作的话， `prod_next`就会一直增加，但事实上，即使是溢出了也没有关系，只要都与 `mask` 进行按位与操作就不会有问题。
例如缓冲区大小为 8，`mask = 7`，我们假设这些所有字段都是用一个字节表示的无符号数（表示范围为 0～255）。
`ring->prod_tail = 2`，`ring->cons_head = 254`，那么队列长度为 `(2 - 254) & 7 = 4` ，含义就是 `254 & 7 = 6`、`255 & 7 = 7`、`0 & 7 = 0`、`1 & 7 = 1` 这四个下标对应的位置是队列中的元素。
实际上这些字段都是 `uint32_t` 类型，所以缓冲区最多是 4 GB 大小。  
这里也解释了前面没有用到的 `ring->cons_tail` 和 `ring->prod_tail` 的用处 ，那就是用来计算队列长度或缓冲区可用空间，可以在 `ring->cons_head` 和 `ring->prod_head` 刚更新但出/入队操作尚未完成时，依然获得到准确的队列长度信息。

（2）缺点
这个方式也就带来了这个高速无锁队列最大的缺点，就是内存消耗大，队列占用空间只能从 2 的幂次中进行选择，可选择的余地很小，所以常常需要开辟过大的内存。

## 生产者/消费者的uint32索引
在实际实现中，`prod_head`、`prod_tail`、`cons_head` 和 `cons_tail` 索引并不像假设的那样介于 0 和 size(ring)-1 之间。
==索引介于 0 和 2^32 -1 之间，我们在访问对象表（环形数组）时屏蔽它们的值「通过索引 & mask 来屏蔽」==。 

32 位模还意味着如果结果溢出 32 位数字范围，对索引的操作（例如，加/减）将自动执行 2^32 模。
> 即：==索引在增加n之后，不需要执行模运算，模运算会影响性能；因为uint32的值溢出的话，相当于自动模运算==。

```c
/* true if x is a power of 2 */
#define POWEROF2(x) ((((x)-1) & (x)) == 0)

int rte_ring_init(struct rte_ring *r, const char *name, unsigned int count,
    unsigned int flags)
{
    ...
    if (flags & RING_F_EXACT_SZ) {
        r->size = rte_align32pow2(count + 1);
        r->mask = r->size - 1;
        r->capacity = count;
    } else {
        if ((!POWEROF2(count)) || (count > RTE_RING_SZ_MASK)) {
            RTE_LOG(ERR, RING,
                "Requested size is invalid, must be power of 2, and not exceed the size limit %u\n",
                RTE_RING_SZ_MASK);
            return -EINVAL;
        }
        r->size = count;
        r->mask = count - 1;
        r->capacity = r->mask;
    }
    ...

}

```

下面是两个示例，有助于解释如何在环中使用索引。
为了简化说明，使用模 16 位运算而不是模 32 位运算，即四个索引被定义为无符号 16 位整数，而不是更实际情况下的无符号 32 位整数。
如下所示，假如 `prod_head`、`prod_tail`、`cons_head` 和 `cons_tail` 为 `uint16_t`类型，最大值为 65535。`rte_ring`的创建的时候 大小为 16384(2^14)
![](attachments/Pasted%20image%2020250413200511.png)
如上所示，这个队列已经使用了11000个对象。

![](attachments/Pasted%20image%2020250413200725.png)
如上所示，这个队列已经使用了12536个对象。

为了便于理解，我们**在上面的例子中使用了模65536运算。在实际执行情况下，这是多余的，效率低下，但在结果溢出时自动完成**。

代码始终保持生产者和消费者之间的距离在 0 和 size(ring)-1 之间。

**（1）访问`ring`的环形数组下标**：
```c
/**
 * Force a function to be inlined
 */
#define __rte_always_inline inline __attribute__((always_inline))


static __rte_always_inline void __rte_ring_enqueue_elems_64(struct rte_ring *r, uint32_t prod_head,
        const void *obj_table, uint32_t n)
{
    unsigned int i;
    const uint32_t size = r->size;
    uint32_t idx = prod_head & r->mask; // 与上掩码，获取下标
    uint64_t *ring = (uint64_t *)&r[1];
    const unaligned_uint64_t *obj = (const unaligned_uint64_t *)obj_table;
    if (likely(idx + n <= size)) {
        for (i = 0; i < (n & ~0x3); i += 4, idx += 4) {
            ring[idx] = obj[i];
            ring[idx + 1] = obj[i + 1];
            ring[idx + 2] = obj[i + 2];
            ring[idx + 3] = obj[i + 3];
        }
        switch (n & 0x3) {
        case 3:
            ring[idx++] = obj[i++]; /* fallthrough */
        case 2:
            ring[idx++] = obj[i++]; /* fallthrough */
        case 1:
            ring[idx++] = obj[i++];
        }
    } else {
        for (i = 0; idx < size; i++, idx++)
            ring[idx] = obj[i];
        /* Start at the beginning */
        for (idx = 0; i < n; i++, idx++)
            ring[idx] = obj[i];
    }
}
```

**（2）获取 `used_count` 的方式**。详细如下所示。
>注：其实更加简单的方式是：`uint32_t entries = (prod_tail - cons_head) & mask;` 因为在任何时候，`used_entries` 和 `free_entries` 都在 0 和 `size(ring)-1` 之间，即使只有第一个减法项溢出了，也可以得到正确值。

```c
static inline unsigned int rte_ring_count(const struct rte_ring *r)
{
    uint32_t prod_tail = r->prod.tail;
    uint32_t cons_tail = r->cons.tail;
    uint32_t count = (prod_tail - cons_tail) & r->mask;
    return (count > r->capacity) ? r->capacity : count;
}
```

**（3）获取空闲数量`free_count`的方式**。 详细如下所示：
> 注：其实更加简单的方式是：`uint32_t free_entries = (mask - (prod_tail - cons_head));`
```c
static inline unsigned int rte_ring_free_count(const struct rte_ring *r)
{
    return r->capacity - rte_ring_count(r);
}
```

# 同步模式分类
`rte_ring` 支持生产者和消费者的不同同步模式。这些模式可以在环创建/初始化时通过标志参数指定。那应该可以帮助用户以最适合其特定使用场景的方式配置环。

## 单生产者单消费者：SPSC
单一生产者（/单一消费者）模式。在这种模式下，一次只允许一个线程将对象入队（/出队）到（/从）环中。

**使用 SP/SC 时，更新 ring 的字段时就不使用 CAS，而是直接更新，这样减少了不必要的开销**。

### 单生产者入队

下面以入队一个元素为例子：
图中：**下半部分的变量是指 ring 这个数据结构的字段，上半部分的变量是每个进程的局部变量**。

**（1）单生产者入队第一步**
![](attachments/Pasted%20image%2020250413210649.png)

初始状态，生产者头尾相同，消费者头尾相同。该生产者进程将 `ring->prod_head`、和 `ring->cons_tail` 复制一份成局部变量 `prod_head`和 `cons_tail`，然后将 `prod_next` 指向 `prod_head` 的下一个。

**（2）单生产者入队第二步**
![](attachments/Pasted%20image%2020250413210719.png)
将变量 `ring->prod_head` 赋值为 `prod_next` 的值 ，同时放入新元素`obj4`。

**（3）单生产者入队第三步**
![](attachments/Pasted%20image%2020250413210737.png)
更新变量 `ring->prod_tail` 为与 `ring->prod_head` 相同的值。

由于不涉及并发，流程很简单，且局部变量没有影响。

### 单生产者出队
略
对于单消费者出队操作，只需要把所有的 `prod_` 换成 `cons_` 就可以了。

## 多生产者多消费者：MPMC
多生产者（/多消费者）模式。这是环的默认入队（/出队）模式。在这种模式下，多个线程可以将对象入队（/出队）到（/从）环中。对于“经典”DPDK 部署（每个core一个线程），这通常是最合适和最快的同步模式。

### 多生产者入队
多生产者入队就涉及到了并发的处理。图里 core 1 和 core 2 就是两个生产者，因为 DPDK 往往使用一个核绑定一个进程。

**（1）多生产者入队第 1～3 步**

![](attachments/Pasted%20image%2020250413211004.png)

(A) 同样的，我们认为初始状态，生产者头尾相同，消费者头尾相同。与单生产入队逻辑相同，每个生产者各自把 `ring->prod_head`、和 `ring->cons_tail` 复制一份成局部变量 `prod_head`和 `cons_tail`，然后将 `prod_next` 指向 `prod_head` 的下一个。

(B) 每个生产者分别用 CAS 操作来移动 `ring->prod_head` 为 `prod_next`。CAS 是一个原子操作，CAS 本身不会发生并发错误。具体为：
 1> 检查如果 `ring->prod_head == prod_head`，那么赋值 `ring->prod_head = prod_next`。
 2> 检查如果 `ring->prod_head != prod_head`，说明 `ring->prod_head` 已被别的生产者修改，于是重新从第一步开始。  
 于是 core 1 先修改了 `ring->prod_head`，当 core 2 想要修改 `ring->prod_head` 时就出错了，所以从第一步重新进行，形成了图二的结果。

(C) core 2 第二次进行第二步时 CAS 成功，`ring->prod_head` 被再次后移，随后 core 1 和 core 2 分别把自己想入队的元素放入自己的局部变量 `prod_head` 指向的位置中，入队成功。

**（2）多生产者入队第 4～5 步**

![](attachments/Pasted%20image%2020250413211044.png)

(A) 接下来，两个生产者都希望更新 `ring->prod_tail`。那么与第二步类似，这里也使用 CAS 操作来进行。只有当一个 core 发现 `ring->prod_tail == prod_head` 时，才会更新 `ring->prod_tail = prod_next`。否则就重做此步。这里，core 1 满足这个条件，于是更新了 `ring->prod_tail` 到 `obj4` 的后一个位置，core 1 入队操作完成。

(B) core 2 此时重做第四步，满足了条件，于是也更新 `ring->prod_tail`，操作成功，入队操作完成。

### 多生产者出队
略

## 多生产者单消费者：MPSC
略
## 单生产者多消费者：SPMC
略

## 多线程宽松尾同步：Multi-thread Relaxed Tail Sync (MT-RTS)
### 背景
在 MP/MC 模式下，如果有大量的并发入队出队操作，会频繁触发 CAS。
CAS 虽然总体来说比锁的速度要快，但是在大量并发访问时会出现**过多自旋**（反复 compare 发现不一致，再次读取，或称 Lock-Waiter-Preemption，LWP）。
虽然 `ring->prod_head` 和 `ring->cons_head` 难以避免，但是 `ring->prod_tail` 和 `ring->cons_tail` 可以减少 CAS 的次数。也就是说，当多个进程并发入（出）队时，只由最后一个进程更新 tail，这样至多只需要一次 CAS 来处理 tail，一定程度减少自旋开销。这就是 Relaxed Tail Sync (RTS) 模式，这个模式适合高并发场景。

### MP_RTS 和  MC_RTS
多消费者RTS和多生产者RTS。

具有轻松尾部同步（RTS）模式的多生产者（/多消费者）。与原始MP / MC算法的主要区别在于，尾值（tail）不会由每个完成入队/出队的线程增加，而是由最后一个线程增加的。
这允许线程避免在环尾值(tail值)上CAS自旋「head值的CAS自旋无法避免」，将实际尾值更改留给给定实例的最后一个线程。

该技术**有助于避免尾部更新时的 `Lock-Waiter-Preemption` (`LWP`) 问题，并改善过度使用`rte_ring`的平均入队/出队时间**。

为实现这一点，RTS 需要 2 个 64 位 CAS 用于每个入队（/出队）操作：一个用于头部更新，第二个用于尾部更新。
相比之下，原始 MP/MC 算法需要一个 32 位 CAS 来更新头部和等待/自旋尾部值。


## 多线程头尾同步：Multi-thread Head-Tail Sync(MT-HTS)
Head/Tail Sync (HTS) 要求更新操作一定要按序完成，即入队/出队操作是完全序列化的。
进程在准备操作时，先检查 ring 的 head 和 tail 的值，只有在它们相等的时候才进行操作。仅当 `head.value == tail.value` 时允许线程继续更改 `head.value` 来实现的。 
这样一方面更新 tail 时不会出现并发冲突，同样减轻了 LWP 问题，也适合高并发多写入场景，同时它还可以支持以下两个功能。

（1）所谓线程安全访问（MT safe peek）
能够以不出队的方式读取一个元素，或者在没入队前保留元素空间。这只能在 SP/SC 模式下和 HTS 模式下使用，因为只有这两个模式是完美同步的。

（2）零拷贝
零拷贝是指数据可以直接放入队列中而无需通过临时变量，同样是在 SP/SC 模式和 HTS 模式两个完美同步的模式下才能获得的功能。与上一个功能原理相同，因为可以快速锁定一块空间，所以可以直接将网卡传上来的数据包直接放入，无需多次辗转。

但事实上，**这个模式已经和加锁是类似的了。性能会有所降低**。但毕竟其更新操作步骤少而快，所以依然是远快于链表实现法。

# 参考
```bash
# 高速无锁队列 in DPDK
https://zhuanlan.zhihu.com/p/515046756

```