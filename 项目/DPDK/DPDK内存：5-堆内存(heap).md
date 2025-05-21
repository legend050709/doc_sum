```table-of-contents
```

# 外部heap
```c
mmap: 比如通过mmap申请大小为size的大页内存
rte_malloc_heap_create
rte_malloc_heap_memory_add
rte_malloc_heap_get_socket
```

## 在外部heap上进行`rte_malloc`
```c
rte_malloc_socket
```


## rte_extmem_register
参考：`app/test/test_external_mem.c`
感觉：`rte_extmem_register` 就是外部自己申请的空间，然后通过heap管理的封装。


# 参考
```bash
# 3、DPDK内存管理——堆内存介绍
https://zhuanlan.zhihu.com/p/659317132

# 2、DPDK内存管理 —— mempool介绍
https://zhuanlan.zhihu.com/p/654772667

# 4、DPDK内存管理 —— DMA MAP
https://zhuanlan.zhihu.com/p/585094736

# 5、DPDK内存管理 —— 大页内存初始化
https://zhuanlan.zhihu.com/p/586610129

# 1、DPDK内存管理概述
https://zhuanlan.zhihu.com/p/658824633
```