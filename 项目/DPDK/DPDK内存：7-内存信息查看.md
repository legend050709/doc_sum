```table-of-contents
```

# mempool查看
# ring查看
# heap查看
```c
void rte_malloc_dump_stats(FILE *f, __rte_unused const char *type)
{
    struct rte_mem_config *mcfg = rte_eal_get_configuration()->mem_config;
    unsigned int heap_id;
    struct rte_malloc_socket_stats sock_stats;

    /* Iterate through all initialised heaps */
    for (heap_id = 0; heap_id < RTE_MAX_HEAPS; heap_id++) {
        struct malloc_heap *heap = &mcfg->malloc_heaps[heap_id];

        malloc_heap_get_stats(heap, &sock_stats);

        fprintf(f, "Heap id:%u\n", heap_id);
        fprintf(f, "\tHeap name:%s\n", heap->name);
        fprintf(f, "\tHeap_size:%zu,\n", sock_stats.heap_totalsz_bytes);
        fprintf(f, "\tFree_size:%zu,\n", sock_stats.heap_freesz_bytes);
        fprintf(f, "\tAlloc_size:%zu,\n", sock_stats.heap_allocsz_bytes);
        fprintf(f, "\tGreatest_free_size:%zu,\n",
                sock_stats.greatest_free_size);
        fprintf(f, "\tAlloc_count:%u,\n",sock_stats.alloc_count);
        fprintf(f, "\tFree_count:%u,\n", sock_stats.free_count);
    }
    return;
}

```

![](attachments/Pasted%20image%2020251014223150.png)

```c
struct rte_malloc_socket_stats {
    size_t heap_totalsz_bytes; /**< Total bytes on heap */
    size_t heap_freesz_bytes;  /**< Total free bytes on heap */
    size_t greatest_free_size; /**< Size in bytes of largest free block */
    unsigned free_count;       /**< Number of free elements on heap */
    unsigned alloc_count;      /**< Number of allocated elements on heap */
    size_t heap_allocsz_bytes; /**< Total allocated bytes on heap */
};

```

|字段名|类型|含义|举例 / 备注|
|---|---|---|---|
|**heap_totalsz_bytes**|`size_t`|当前 NUMA 节点（socket）上 DPDK hugepage 堆的 **总大小（字节）**。|这代表该 socket 上通过 `rte_malloc_heap_create()` 创建的 heap 容量（来自 hugepage 内存）。例如，如果通过 `--socket-mem=1024,1024` 启动，则每个 socket 堆的总大小可能为 1GB。|
|**heap_freesz_bytes**|`size_t`|当前 heap 上尚未分配的 **空闲总字节数**。|表示当前还有多少空间可供 `rte_malloc()` 分配。`heap_totalsz_bytes - heap_freesz_bytes = 已分配空间`。|
|**greatest_free_size**|`size_t`|当前 heap 中最大的 **连续空闲块的大小**。|用于衡量内存碎片程度。例如：heap 总空闲 300MB，但最大空闲块只有 2MB，则说明严重碎片化。|
|**free_count**|`unsigned`|heap 中当前空闲块（free element）的数量。|每个空闲块是一段未被分配的 hugepage 区间。数量越多说明碎片越多。|
|**alloc_count**|`unsigned`|当前已分配的块（allocated element）的数量。|对应通过 `rte_malloc()` / `rte_zmalloc()` 成功分配的次数（未释放的块）。|
|**heap_allocsz_bytes**|`size_t`|当前 heap 上实际已分配的总字节数。|通常等于 `heap_totalsz_bytes - heap_freesz_bytes`。|

## 使用场景

|使用场景|说明|
|---|---|
|**性能分析 / 内存碎片检测**|通过 `greatest_free_size` 和 `free_count` 判断碎片化问题。|
|**NUMA 亲和调试**|比较不同 socket 上 heap 使用情况，分析内存均衡度。|
|**内存泄漏检测（粗粒度）**|观察 `alloc_count` 长期增加但 `free_count` 不变。|
|**容量监控**|监控 `heap_freesz_bytes` 接近 0 的情况，预警“内存耗尽”。|

## heap 说明
### 内部heap
默认情况下，不使用外部内存时，DPDK针对每个 socket 维护一个堆（heap），每个堆的起始大小是通过`--socket-mem`设置。
比如 `--socket-mem=1024,1024` 启动，则每个 socket 堆的总大小可能为 1GB。
heap-id、heap-name 分别为：`0 / socket_0` 以及 `1 、socket_1`

### 外部heap
```c
// 基于名称申请一个heap
int rte_malloc_heap_create(const char *heap_name);

// 设置heap管理的空间地址
int rte_malloc_heap_memory_add(const char *heap_name, void *va_addr, size_t len, 
	rte_iova_t iova_addrs[], unsigned int n_pages, size_t page_sz);

int rte_malloc_heap_get_socket(const char *name);
```



# seg查看

## seg 和 seglist

# zone查看

# 关联
## DPDK内存概述

来自于chatgpt：
![](attachments/Pasted%20image%2020251014231700.png)

![](attachments/Pasted%20image%2020251014231830.png)




## DPDK中 heap，seg, seglist, memzone的关系

来自于chatgpt的说明：
![](attachments/Pasted%20image%2020251014225535.png)

![](attachments/Pasted%20image%2020251014225359.png)



heap 是按 NUMA 分的堆；
seglist 是 heap 管理的 hugepage 段集合；
seg 是实际 hugepage 内存；
memzone 是从 heap 中切出的一块带名字的内存区域。

## rte_malloc接口从哪里获取内存呢，和heap，seglist, seg, zone 的关系

来自于chatgpt的说明：

![](attachments/Pasted%20image%2020251014225852.png)

![](attachments/Pasted%20image%2020251014230115.png)

## rte_malloc 接口 和 rte_memzone_reserve 接口

来自于 chatgpt的说明：

![](attachments/Pasted%20image%2020251014230750.png)

![](attachments/Pasted%20image%2020251014230933.png)



# 参看
```bash

```