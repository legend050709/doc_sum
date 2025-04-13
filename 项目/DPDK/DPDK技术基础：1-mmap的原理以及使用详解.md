```table-of-contents
```

# mmap和大页内存
## 大页内存申请
```c
unsigned flag = (21UL<< MAP_HUGE_SHIFT) | MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_POPULATE;
if (!(size % (1*1024*1024*1024))) {
    // try 1G page
    flag = (30UL<< MAP_HUGE_SHIFT) | MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_POPULATE;
    addr = mmap(NULL, size, PROT_READ | PROT_WRITE, flag, -1, 0);
    if (addr == MAP_FAILED) {
        // try 2M page
        flag = (21UL<< MAP_HUGE_SHIFT) | MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_POPULATE;
        addr = mmap(NULL, size, PROT_READ | PROT_WRITE, flag, -1, 0);
    }
} else {
    addr = mmap(NULL, size, PROT_READ | PROT_WRITE, flag, -1, 0);
}
```
## 设备DMA map
参考：[# DPDK内存管理 —— DMA MAP](https://zhuanlan.zhihu.com/p/585094736)
参考：[libtpa中的tpa_heap.c文件中关于dma map的使用]()
```c
rte_dev_dma_map
```
# 参考
```bash
# Linux Mmap映射：优化文件访问和共享内存
https://zhuanlan.zhihu.com/p/685848279

```