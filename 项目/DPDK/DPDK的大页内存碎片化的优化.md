```table-of-contents
```
# 背景
DPDK（Data Plane Development Kit）是一个用于高性能数据包处理的开源项目，常用于构建各类高性能网络应用。

基于 DPDK 的设计思想及其提供的线程/内存模型，SPDK(Storage Performance Development Kit)被开发出来，广泛用于构建各类高性能存储应用。  

为了提升应用程序的性能，DPDK 基于大页内存实现了一套自己的内存管理接口，这些内存接口被广泛应用于SPDK/DPDK 的各类应用中，而且不少应用会在 IO 路径中进行内存的分配与释放。DPDK 常用的内存接口有：

```c
void *rte_malloc(const char *type, size_t size, unsigned align);
void *rte_realloc(void *ptr, size_t size, unsigned int align);
void rte_free(void *ptr);
```

在实际使用过程中，我们发现 **DPDK 原生的内存分配接口在一些场景下有着较大的性能问题**。本文将结合实际遇到的问题，为大家介绍如何优化 DPDK 内存碎片问题。

# 问题现象
当接到 DPDK 内存碎片化的问题反馈后，经过多方排查，最终将怀疑点锁定到大页内存的分配上，并将实际使用时内存分配的逻辑与参数进行抽象后，编写了如下的测试 case：
```c

uint64_t entry_time, time;
size_t size = 4096;
unsigned align = 4096;
for (int j = 0; j < 10; j++) {
        entry_time = rte_get_timer_cycles();
        for (int i = 0; i < 2000; i++) {
                rte_malloc(NULL, size, align);
        }
        time = (rte_get_timer_cycles()-entry_time) * 1000000 /
                rte_get_timer_hz();
        printf("total open time %lu us, avg time %lu us\n", time, time/2000);
}
```
单线程执行内存分配，总共分配 1 万个 4k 大小的对象,  每 2000 次分配后计算平均耗时，中间不释放内存，各阶段单次平均耗时如下表：

![](attachments/Pasted%20image%2020240414175245.png)

从上表可以看到：
**随着分配次数的增加，后续每分配 4K内存的耗时在线性增加；
按照我们之前的理解，==单线程==下多次分配内存耗时不会过高，每次分配4K内存的耗时应该差不多**；
但是当前结果与预期不符，针对该场景我们进行了深入分析。

# 分析
首先结合代码分析了 DPDK 分配的逻辑，然后结合火焰图对怀疑点加了些调试手段以进一步 debug，最终得到了结论，并对结论进行了的验证。

## 代码分析
DPDK 通过 heap 管理每个 NUMA 节点上的大页内存，对于一块连续内存，首先转换成一个 `struct malloc_elem` 对象 elem，再将 elem 插入链表中，并保证链表中每项的地址按大小排序。
每个 heap 都维护了 13 个链表，每个链表中的 elem 大小在相同范围内：**比如大小 0-256 的 elem 在同一个链表，256-1024 的 elem 在同一个链表，1024-4096 的 elem 在同一个链表……**

DPDK内存分配的主要逻辑如下图所示：

![](attachments/Pasted%20image%2020240414175348.png)

1. 根据参数和当前线程所在 CPU 决定从哪个 heap 上分配内存。
    
2. 根据要分配的大小，找到对应的链表（对应代码中的 `free_head[N]`），遍历该链表中所有的 elem，找到合适（满足**大小、对齐**等要求）的 elem。
    
3. 将 elelm 从链表上摘除，根据大小进行切分并返回。

## 火焰图分析
![](attachments/Pasted%20image%2020240414175416.png)

可以看到 `find_subitable_element` 函数耗时非常高，这个函数的作用就是遍历链表上的 elem，找到满足要求的 elelm。以分配 4K 内存为例，首先遍历第三个链表即 `free_head[2]` 上的所有 elem，逐个检查是否满足要求，如果找到合适的 elem，就从链表上摘除，根据大小进行切分并返回，如果 `free_head[2]` 上找不到，会依次在`free_head[3]`、`free_head[4]` 上查找，直到找到为止。

从代码分析单次判断 elem 是否满足要求耗时应该很短，所以怀疑是链表上元素过多，导致整个遍历耗时过久。那么我们可以尝试添加些 log，看看各个链表上 elem 的个数是否有变化。

## 添加调试手段分析

经分析 DPDK 有些内存 dump 的 rpc，其中用到的 malloc_heap_get_stats 函数会遍历 heap 上所有链表上的 elem，我们可以简单修改，查看每个链表上的 elem 个数。

系统刚启动完成时，可以看到 elem 个数不多，大小也比较集中。

![](attachments/Pasted%20image%2020240414175553.png)

分配 2000 个 4K 内存后，看到 free_head[2] 上 elem 数量明显增加：

![](attachments/Pasted%20image%2020240414175604.png)

分配 4000 个 4K 内存后：

![](attachments/Pasted%20image%2020240414175627.png)

可以看到 `free_head[2]` 上 elem 个数随着分配次数的增加线性增加，这会导致每次分配 4K 所需遍历的 elem 个数也会增加，那么为什么会这样呢？

### 原因分析

添加 log 发现，`free_head[2]` 上所有 elem 的大小都小于 4096 字节:

![](attachments/Pasted%20image%2020240414175719.png)

为什么会出现这么多小的 elem 呢？经分析我们发现是在分配时切分 elem 而生成。
切分 elem 示意图如下：

![](attachments/Pasted%20image%2020240414175750.png)

以上图为例：

1. 当调用 `find_suitable_element` 函数后，找到了一个满足要求的 elem，该 elem 大小一般远大于 4096 字节，所以在 `malloc_elem_alloc` 中会切分。
    
2. `malloc_elem_alloc` 中会从 elem 末尾开始，根据大小和对齐算出 `new_elem` 的地址，由于分配内存要求 4K 对齐，所以 `end-start` 很可能大于 4096 ，而我们只要 4096，那么 `new_elem` 后面的 tailer 这段空间为避免浪费，是可以被重新利用的，由于其大小是 2432，也会被插入 `free_head[2]` 中。这也就导致了 `free_head[2]` 上 elem 个数在线性增加。

## 验证
根据上述分析，如果 align 为 64 字节，elem 尾部空间应该小于 64，不会再插入链表，就不会有这个问题，将测试代码中的 align 参数改为 64 后，测试结果如下:
平均耗时 0.7us ~ 0.8us，不再有线性增加的现象。

![](attachments/Pasted%20image%2020240414175859.png)

那么是不是所有的 align 都有这个问题？我们又测了分配 64K 大小，align 为 64K 的情况，也出现了相同的情况。

![](attachments/Pasted%20image%2020240414175943.png)

## 结论

DPDK 在管理内存时会**将相邻大小的 elem 放在一个链表**，分配时根据大小 4K 找到对应的链表 `free_head[2]`，遍历该链表上的 elem 来查找满足要求的 elem，**当分配时指定 align 过大，会使找到的 elem 尾部空间较大**，为避免空间浪费会将尾部空间重新插入 `free_head[2]` 链表中，导致 `free_head[2]` 链表中 elem 数量增多，下次分配时又需要先遍历 `free_head[2]`，使每次分配耗时呈线性增加。


# 解决方案
## 方案一：heap 中新增 free_head_max_size 数组
```c
struct malloc_heap {
        rte_spinlock_t lock;
        LIST_HEAD(, malloc_elem) free_head[RTE_HEAP_NUM_FREELISTS];
        size_t free_head_max_size[RTE_HEAP_NUM_FREELISTS];
        .....
}
```


在 heap 中新增 free_head_max_size 数组，记录每个 freelist 中 elem 的最大 size，遍历时如果要求的 size 大于该 freelist 上最大的size，可以直接跳过，但缺点是每次分配/释放时都要维护该数组。

![](attachments/Pasted%20image%2020240414180215.png)

可以看到单次分配耗时没有再呈线性增加，**平均耗时稳定在 0.5us 左右，对比之前性能提升 30 倍+**：

![](attachments/Pasted%20image%2020240414180238.png)

## 方案二：修改heap链表管理的内存边界

在原有的逻辑中，大小为 4K 的 elem 在 `free_head[2]` 上，4K 刚好处于边界，如果修改链表选择的规则，改到在 `free_head[3]` 进行查找，也可以解决问题。以分配 4K 为例，在 4K-16K 的链表上更容易找到对应的 elem，16K 也是在 16K-64K 的链表上能查找的概率更高。

![](attachments/Pasted%20image%2020240414180353.png)

也即每个链表上包含的元素大小范围如下：
```c
heap->free_head[0] - (0   , 2^8)   0 < x < 256
heap->free_head[1] - [2^8 , 2^10)  256 <= x < 1024
heap->free_head[2] - [2^10 ,2^12)  1024 <= x < 4096
heap->free_head[3] - [2^12, 2^14)  4k <= x < 16k
heap->free_head[4] - [2^14, 2^16)  16k <= x < 64k 
...
```

对比测试效果与方案一无明显差异，优点是对 16K/64K 等处于边界点的分配效率也有提升，缺点是可能会对极端情况有影响，需要更多测试。

![](attachments/Pasted%20image%2020240414180447.png)

### 测试验证

#### 单线程

- **bulk**：连续分配 1000 个对象，memset 后，1000 个对象一起释放，统计单次平均分配/释放耗时  
    
- **no-bulk**：分配 1 个对象，memset, delay 5us, 释放，重复 1 万次，统计平均分配和释放的总耗时
    
- **realloc**：分配 1 个对象，memset, realloc 成原本 2 倍大小，memset, realloc 成原本 4 倍大小，memset，释放，重复 1000 次，统计平均总耗时

![](attachments/Pasted%20image%2020240414180609.png)

#### 多线程

- **bulk**：连续分配 1000 个对象，memset 后，1000 个对象一起释放，统计单次平均分配/释放耗时
    
- **no-bulk**：分配 1 个对象，memset, delay 5us, 释放，重复 1 万次，统计平均分配和释放的总耗时
    
- **realloc**：分配 1 个对象，memset, realloc 成原本 2 倍大小，memset, realloc 成原本 4 倍大小，memset，释放，重复 1000 次，统计平均总耗时

![](attachments/Pasted%20image%2020240414180742.png)


16 个线程，每个线程单独绑核后做如下测试：
连续分配 N 个 4K 大小对象，4K 对齐，每次分配之间 delay 5us, 统计耗时，然后连续释放 N 个 4K 对象，不delay，统计单次耗时

![](attachments/Pasted%20image%2020240414180905.png)


### DPDK malloc perf test

**malloc 测试**：
修改前：
![](attachments/Pasted%20image%2020240414180948.png)

修改后：
可以看到没有性能衰退，4K/64K/1M 等大小分配有 15%-50% 的性能提升：

![](attachments/Pasted%20image%2020240414181026.png)


**memzone 测试**：
修改前：

![](attachments/Pasted%20image%2020240414181107.png)

修改后：
memzone测试，修改后没有性能衰退，64K/1M 大小的分配分别有 7% 和 52% 的性能提升：

![](attachments/Pasted%20image%2020240414181129.png)

## 总结
通过以上分析，确定了 DPDK 内存碎片化的诱因是**在分配时要求对齐的大小比较大，导致 elem 切分时产生了很多碎片，并影响了后续内存分配的效率**。

经过完整的测试，最终选用了方案二，改动简单，效果也更好，**不仅解决碎片场景下的问题，使得性能提升 30+ 倍，也在非碎片情况下，使得 4K/64K/1M 等大小的内存分配性能提升 15-50% ，同时也能缓解多进程并发分配时竞争锁的压力**。目前字节跳动 STE 团队已将该补丁提交至 DPDK 社区，并于 2023 年 2 月 合入 DPDK 主线，链接如下：  

https://inbox.dpdk.org/dev/CAJFAV8x5k55jH0A_BHN9jxA1m-3o08tKr7RbCesvVL7o4MKgGQ@mail.gmail.com/  
  

DPDK/SPDK 作为开源项目，凭借其出色的性能表现赢得了众多开发者的青睐，我们也希望在使用开源项目获得收益的同时，能为项目与社区带来更多想法，做出更多贡献，与开发者们一起推进社区的建设。

# 参考
```bash
# DPDK内存碎片优化
https://mp.weixin.qq.com/s?__biz=Mzg3Mjg2NjU4NA==&mid=2247484462&idx=1&sn=406c59905ea57718018c602cba66f40e&chksm=cee9f259f99e7b4f7597eb6754367214a28f120c062c72c827f6e8c4225f9e54e4c65d4395d8&scene=21#wechat_redirect
```