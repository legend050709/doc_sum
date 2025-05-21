```table-of-contents
```

# 介绍
```c
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```
`mmap`函数根据指定的长度**将用户空间的一段内存区域映射到内核空间**，映射成功后，用户对这段内存区域的修改可以直接反映到内核空间；
同样，内核空间对这段区域的修改也直接反映用户空间。
mmap调用成功的返回值就是用户空间内存的起始地址，这个地址是一个虚地址。



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