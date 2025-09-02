```table-of-contents
```

# 外部heap

## 整体使用流程
```c
（1）申请外部内存
mmap: 比如通过mmap申请大小为size的大页内存

（2）创建heap
rte_malloc_heap_create

（3）heap关联外部内存
rte_malloc_heap_memory_add

（4）基于heap_name获取heap的 socket_id;
rte_malloc_heap_get_socket
注：heap的标识是 heap_name;

(5) 将外部内存地址进行设备的`dma`映射[可选]。
rte_dev_dma_map
rte_extmem_unregister：取消注册外部内存

(6) 在外部heap上进行`rte_malloc`
rte_malloc_socket
```

## 在外部`heap`上进行`rte_malloc_socket`



## rte_extmem_register
参考：`app/test/test_external_mem.c`
感觉：`rte_extmem_register` 就是外部自己申请的空间，然后通过`heap`管理的封装。


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