```table-of-contents
```
# 概述

伴随着摩尔定律的失效，现在CPU的性能几乎很难进行提升了。那么提高性能的方式就只能是软件不够硬件来凑了，让许多工作从CPU卸载到硬件去。`dmadev lib`顾名思义就是用来使用`dma`设备的框架；`dmadev lib`库是DPDK中的一个软件库，提供管理和配置`DMA poll mode drivers`的软件，定义了统一的操作接口。支持一系列不同的DMA操作。

# DPDK的DMA MAP

## 问题
正式开始介绍前，先抛出三个问题：
`rte_mbuf`结构体中有两个和地址相关的指针，分别是`buf_addr`和`buf_iova`，如下图1所示。
![](attachments/Pasted%20image%2020250423224013.png)

**问题1**、当IOVA as VA模式时，buf_addr 和buf_iova都是一个虚地址；当IOVA as PA模式时，buf_addr的值是一个虚地址，buf_iova的值需要是一个物理地址。why？

**问题2**、VA模式下，当我们调用rte_pktmbuf_alloc申请一个rte_mbuf，`rte_mbuf->buf_addr == rte_mbuf->buf_iova`，也就是说当我们拿到rte_mbuf时，dpdk已经帮我们把地址都设置好了，不需要我们来关心这两个值，那么dpdk是在啥时候做的呢？

**问题3**、VA模式下，如果我们想将buf_addr指向一块我们自己mmap/malloc出来的内存，那么这个buf_addr和buf_iova的值应该怎么给？



## 背景知识

### mmap
```c
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```
`mmap`函数根据指定的长度**将用户空间的一段内存区域映射到内核空间**，映射成功后，用户对这段内存区域的修改可以直接反映到内核空间；
同样，内核空间对这段区域的修改也直接反映用户空间。
mmap调用成功的返回值就是用户空间内存的起始地址，这个地址是一个虚地址。


### MMU和IOMMU
![](attachments/Pasted%20image%2020250423224442.png)

CPU访问内存和设备访问内存是不一样的。CPU访问内存需要借助于MMU（Memory Management Unit），MMU将CPU给的虚地址翻译成物理地址。

而当IOVA as VA模式时，设备访问内存要借助于IOMMU（Input-OutputMemory Management Unit）将设备dma地址翻译成物理地址，这个dma地址是虚地址。如果IOVA as PA，那么这个dma地址是物理地址，不需要IOMMU的翻译，直接用于访问物理内存。


MMU把CPU的虚拟地址(va)转换成物理地址(pa)，IOMMU的作用就是把DMA的虚拟地址(iova)转换成物理地址(pa)。
MMU转换时用到了pagetable，IOMMU转换也要用到io pagetable，两者都是软件负责创建pagetable，硬件负责转换。

#### io pagetable
IOMMU的pagetable和MMU的pagetable一模一样，转换方式也一样，都支持4KB/2M/1G大小的page，都支持4级和5级页表，4级和5级的区别就是va/iova的长度是48位还57位。

## 问题1的分析
再次阅读上面的背景知识，现在来回答问题1。

**问题1**：
既然rte_mbuf位于用户空间，那么buf_addr的值肯定是一个虚地址，但是buf_iova的值填的是dma地址，在VA模式下，dma地址需要通过iommu做一次翻译，其实就是iommu根据dma地址查内部的页表（这个页表啥时候创建的，在后面会解释），找到对应的物理地址，所以buf_iova地址是虚地址可以解释了。那么PA模式下呢，因为PA模式下，dma地址是不经过IOMMU转换的，直接用来访问物理内存，所以这时候DMA地址必须是物理地址。

## 问题2的分析

### rte_mbuf的初始化
在回答问题2之前，说一下rte_mbuf的由来，首先当dpdk调用rte_eal_init初始化时，内部会多次调用mmap完成大页映射。然后应用程序自己调用dpdk提供的接口，如rte_pktmbuf_pool_create申请大页内存创建和初始化mempool，同时指定了rte_mbuf的个数。
```c
struct rte_mempool* rte_pktmbuf_pool_create(const char * name,  //mempool的名字
                                            unsigned n,         //rte_mbuf的数量
                                            unsigned cache_size, // mempool在每个core的cache的个数
                                            uint16_t priv_size,  // mempool的元素ele的私有数据的大小
                                            uint16_t data_room_size,
                                            int socket_id )
```

也就是说，`mempool`初始化完成后，这个`mempool`里面的`rte_mbuf`也会初始化好。

### 解答
现在问题2就很好回答了，`rte_mbuf`的初始化是在创建和初始化`mempool`时完成的，也就是说`rte_mbuf->buf_addr`和`rte_mbuf->buf_iova`也是在那时候填充的。调用`rte_pktmbuf_alloc`只是从`mempool`里面取了一个`rte_mbuf`出来，并没有做初始化的工作。
`dpdk`内部的代码，如下所示。

```c
/* dpdk初始化时完成大页映射的过程 */
rte_eal_init(int argc, char **argv)
   --> eal_hugepage_info_init() //获取用户的大页内存配置信息
          --> hugepage_info_init()
   --> rte_eal_memory_init() //实现大页内存到用户空间的映射
          --> rte_eal_hugepage_init()
                 --> eal_legacy_hugepage_init()
                        --> mmap() //实现映射的最终函数

/* mempool初始化时，完成rte_mbuf的初始化 */
rte_pktmbuf_pool_create
   --> rte_pktmbuf_pool_create_by_ops
          --> rte_mempool_populate_default
                 --> rte_mempool_populate_iova
                        --> mempool_add_elem

static void mempool_add_elem(struct rte_mempool *mp, __rte_unused void *opaque,
		 void *obj, rte_iova_t iova)
{
	struct rte_mempool_objhdr *hdr;
	struct rte_mempool_objtlr *tlr __rte_unused;

	/* set mempool ptr in header */
	hdr = RTE_PTR_SUB(obj, sizeof(*hdr));
	hdr->mp = mp;
	hdr->iova = iova;      
	STAILQ_INSERT_TAIL(&mp->elt_list, hdr, next);
	mp->populated_size++;

#ifdef RTE_LIBRTE_MEMPOOL_DEBUG
	hdr->cookie = RTE_MEMPOOL_HEADER_COOKIE2;
	tlr = __mempool_get_trailer(obj);
	tlr->cookie = RTE_MEMPOOL_TRAILER_COOKIE;
#endif
}
```

那为什么==`IOVA as VA时`，`buf_addr`和`buf_iova`的值会相等==？这个要结合`dma map`才能清楚的回答。

## DMA MAP
### 标准dma map流程
dma map的关键函数是mmap和ioctl，如下代码所示。
```c
dma_map.vaddr = mmap(0, 1024*1024, PORT_READ | PORT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
dma_map.size = 1024*1024;
dma_map.iova = 0;  //一般给0，也可以给其他值，比如vaddr。
dma_map.flags = VFIO_DMA_MAP_FLAG_READ | VFIO_DMA_MAP_FLAG_WRITE;

ioctl(container, VFIO_IOMMU_MAP_DMA, &dma_map);
```

首先，利用`mmap`映射出1MB字节的虚拟空间，因为物理地址对于用户态不可见，只能通过虚拟地址访问物理空间。
然后执行`ioctl`的`VFIO_IOMMU_MAP_DMA`命令，传入参数主要包含`vaddr`及`iova`，其中`iova`代表的是设备发起DMA请求时要访问的地址，也就是`IOMMU`映射前的地址，`vaddr`就是`mmap`的地址。

`VFIO_IOMMU_MAP_DMA`命令会为虚拟地址vaddr找到物理页并pin住（因为设备DMA是异步的，随时可能发生，物理页面不能交换出去），然后找到Group对应的Contex Entry，建立页表项，页表项能够将iova地址映射成上面pin住的物理页对应的物理地址上去，这样对用户态程序完全屏蔽了物理地址，实现了用户空间驱动。
一句话概述，**`VFIO_IOMMU_MAP_DMA`这个命令就是将`iova`通过`IOMMU`映射到`vaddr`对应的物理地址上去**。

![](attachments/Pasted%20image%2020250424102828.png)

### dpdk dma map流程

dpdk有两种内存需要dma map，一是提前申请好的大页内存，二是**临时mmap**出来的小页内存。

#### 大页内存dma map
对于提前申请好的大页内存，dpdk在调用rte_eal_init初始化时就通过`标准dma map流程`的方式建立好了dma地址到物理内存地址的映射，并将映射表存在`iommu`内。调用链见下图。
![](attachments/Pasted%20image%2020250424104002.png)


根据`标准dma map流程`的描述我们可以知道，如果在`ioctl`的`dma_map`参数里面，`dma_map.iova`的值填`dma_map.vaddr`，那么`dma_map.iova`通过`iommu`找到的物理地址和`dma_map.vaddr`通过`mmu`找到的物理地址是一样的。这也就解释了为啥调用`rte_pkmtbuf_alloc`出来的`rte_mbuf`，`rte_mbuf->buf_addr`和`rte_mbuf->buf_iova`相等的原因。

#### 小页内存dma map
如果我们在使用`rte_mbuf`时，希望`buf_addr`指向我们临时`mmap`出来的内存呢，这时候`buf_addr`和`buf_iova`应该怎么填写。

```c
rte_iova_t iova;
vaddr = mmap(0, 1024*1024, PORT_READ | PORT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);
rte_extmem_register	(void *vaddr, size_t len, rte_iova_t iova_addrs[], unsigned int n_pages, size_t page_sz);	
rte_dev_dma_map	(struct rte_device *dev, void *vaddr, uint64_t iova, size_t len);
/* 填写rte_mbuf的buf_addr和buf_iova */
rte_mbuf->buf_addr = vaddr;
rte_mbuf->buf_iova = iova;
/* 用完了之后需要dma unmap和mem unregister. */
rte_dev_dma_unmap (struct rte_device *dev, void *vaddr,uint64_t iova, size_t len);	
rte_extmem_unregister(void *vaddr,size_t len);
```

其实过程和`标准dma map流程`介绍的大致一样，只不过将`ioctl`换成了`rte_dev_dma_map`函数，我们深入`dpdk`源码考察`rte_dev_dma_map`函数，发现其实底层也是调用的`ioctl`。
```c
rte_dev_dma_map 
  --> dma_map
    --> pci_dma_map 
      --> rte_vfio_container_dma_map 
        --> container_dma_map 
          --> vfio_dma_mem_map 
            --> dma_user_map_func 
              --> vfio_type1_dma_mem_map 
                --> ioctl
```


考虑这样一个场景，如果`mmap`的长度为8k，且是按页对齐的，也就是说`mmap`了两个页，要求不同的页需要使用不同的`rte_mbuf`，这个时候`buf_addr`和`buf_iova`应该怎么填写。

![](attachments/Pasted%20image%2020250424105411.png)

如果`m1->buf_addr = 0x8192，m1->buf_iova = 0`，那么`m2->buf_addr = 0x8192 + 4096，m2->buf_iova = 0 + 4096`。


# API接口
## rte_extmem_register 和 rte_extmem_unregister

### 使用
参考：`app/test/test_external_mem.c`
## rte_dev_dma_map 和 rte_dev_dma_unmap
### 介绍
网卡设备dma映射外部内存。

### 使用
参考：`app/test-pmd/testpmd.c`

# dpdk中的rte_memcpy和dma的选择
## 概述
现代硬件大部分都支持DMA，可以offload一些软件的操作，那什么场景下选择DMA而不是memcpy呢？本文对比一下memcpy和DMA，同时介绍一下DPDK中DMA的一些用法。

![](attachments/Pasted%20image%2020250423221339.png)

## 测试方法
### rte_memcpy的测试
#### 介绍
在大多数情况下，rte_memcpy()比普通的memcpy()更快。这是因为它利用了 Intel 处理器的特殊特性（如 SIMD 指令集）来优化复制操作。而且rte_memcpy()的实现考虑了处理器架构和内存层次结构，能够更好地利用 CPU 的缓存系统。在可以使用rte_memcpy()的场景下，尽可能用rte_memcpy()。

#### 测试方法
编译完DPDK后，可以用./build/app/test/dpdk-test来测试DPDK相关函数的性能，比如rte_memcpy()的。

![](attachments/Pasted%20image%2020250423221536.png)

注意结果是按ticks来计算的，也就是128B的搬运，需要大约7个ticks。

### memcpy的测试
memcpy可以使用`mbw`工具来测试内存带宽。
`mbw`将分配两个指定大小的数组，并将一个数组复制到另一个数组中，通过测量复制所花费的时间来计算内存带宽。测试完成后，会显示每个测试的带宽结果。

可以下载源码，然后编译，进行测试。
```bash
git clone https://github.com/raas/mbw.git
```

常用的参数有：
```bash
-q：静默模式，禁止显示信息性消息。  
-a：禁止打印每个测试的平均值。  
-n <number>：设置每个测试运行的循环次数。  
-t <number>：选择要运行的测试类型。默认情况下，会运行所有的测试。-t0代表memcpy()测试，
	-t1 代表dumb（b[i]=a[i]样式）测试;
	-t2 代表具有任意块大小的memcpy()测试。  
-b <bytes>：对于-t2测试，设置块大小（以字节为单位）。  
-h：显示快速帮助。
```
一般选择-t2，可以自定义块大小，测试命令如下：
```bash
# 1024B，-t2是可以配置block size，-b就是大小。-a是数组的长度，-q是不输出额外信息。  
$ mbw -n 1024 -t2 -b 1024 -a 4096 -q  
0 Method: MCBLOCK Elapsed: 0.58694 MiB: 4096.00000 Copy: 6978.567 MiB/s  
1 Method: MCBLOCK Elapsed: 0.58510 MiB: 4096.00000 Copy: 7000.573 MiB/s  
2 Method: MCBLOCK Elapsed: 0.58520 MiB: 4096.00000 Copy: 6999.257 MiB/s  
# 9194B  
$ mbw -n 1024 -t2 -b 9194 -a 4096 -q  
0 Method: MCBLOCK Elapsed: 0.33778 MiB: 4096.00000 Copy: 12126.200 MiB/s  
1 Method: MCBLOCK Elapsed: 0.33688 MiB: 4096.00000 Copy: 12158.488 MiB/s  
2 Method: MCBLOCK Elapsed: 0.33698 MiB: 4096.00000 Copy: 12155.204 MiB/s
```


### dma的测试
可以运行`dpdk-test`工具中的`dmadev_autotest`测试`DMA`的`API`。

![](attachments/Pasted%20image%2020250423222053.png)

DPDK官方通用的case没找到perf方面的，真正的perf测试case在厂商提供的SDK里会有，intel的DMA外设一般是DSA，而ARM芯片的种类就比较多了，一般是DPI。

通过在ARMv9平台的芯片测试，发现>10kiB时，DMA的用时和memcpy接近，后续DMA和memcpy性能相差无几。也就是当超过64kiB时，DMA的用时也是和memcpy类似的。不过DMA可以给CPU留出时间，做些其它的操作。

测试需配置的常见参数有：
```bash
类型：同步，异步。  
scatter/gather: 多个segment合并后传输，或不合并分开传输。  
segment个数。  
segment的size大小。  
传输的内存类型：内存或者包。  
等待方式：轮询或者event通知。  
重复次数：repeat。
```

一般重复次数会选1000次以上，确保结果的准确性。segment的size可以选择以太网MTU的大小，或者平时常用的包size。

## 适用场景
个人觉得，平时还是用`rte_memcpy()`更为方便和通用，性能差不多，代码也简单。因为DMA需要额外的配置，初始化session，配置队列，完成的通知机制等。
除非真的涉及到外设的内存搬运，比如`scatter/gather DMA`，直接把多个小包组成大包发给CPU。

### DMA常用场景
DMA常用场景, ==外设（磁盘，网卡，GPU、FPGA等）和内存之间的数据交换==:

**硬盘数据读写**： 
当从硬盘读取数据或向硬盘写入数据时，可以使用 DMA 技术，不经过 CPU 就可以将数据直接从硬盘移动到内存，或从内存移动到硬盘。

**网络数据传输**： 
在网络设备和计算机之间传输数据时，可以使用 DMA 技术将数据直接从网络设备复制到内存，或者从内存复制到网络设备。现在RDMA也非常火。

**图像和音频处理**：
在处理大量图像或音频数据时，使用 DMA 可以直接将数据从图像或音频设备传输到内存，或者从内存传输到设备，无需 CPU 的参与。

**硬件加速器通信**： 
在高性能计算或机器学习应用中，可以使用 DMA 技术在 CPU 和硬件加速器（例如 GPU 或 FPGA）之间传输大量数据。

### 小结
至于`memcpy()`，DPDK官方更推荐`rte_memcpy()`，有一些针对架构的额外优化，而且支持非对齐首地址的拷贝。
总的原则是，单个搬运小于10kiB的，就是rte_memcpy()更划算。

# 参考
```bash

# DPDK内存管理 —— DMA MAP
https://zhuanlan.zhihu.com/p/585094736

# DPDK中rte_memcpy与DMA的选择
https://mp.weixin.qq.com/s/jwC0M1ZYFMQkO7rOXzHsWg

# DPDK dmadev lib学习
https://mp.weixin.qq.com/s/Cb_DpyqPAr5rRImm9CiBjg


```