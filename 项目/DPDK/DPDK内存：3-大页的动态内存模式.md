```table-of-contents
```
# 前言
作为高性能的数据转发面套件库，DPDK为了最大化发挥系统效能，它基于系统大页内存构建了自己独有的内存管理机制，并提供了常用的内存管理设施，包括Mempool、堆内存分配以及DMA内存管理等功能。

# 内存管理层次
DPDK的内存管理构筑于系统大页内存之上，应用程序在启动或运行时会向系统申请需要的大页内存，然后，DPDK会对这些内存进行层次化管理，构建完整的内存管理框架。DPDK管理内存的层次结构如下：

![](attachments/Pasted%20image%2020240410193344.png)

- 对于从系统中每个拿到的内存大页，DPDK会使用专门的数据结构进行描述，称为Memseg；
- Memseg只是单纯描述了物理大页内存，并不能直接由其它组件或上层应用使用，因此DPDK在Memseg上又构建了一套堆内存管理机制，在DPDK的代码中，使用Malloc_heap进行描述，同时向上提供堆内存分配接口；
-  其它的内存设施，如Memzone、Mempool则通过Malloc Heap申请内存，进行更精细的管理。

# 内存模式
DPDK存在两种内存模式:
- legacy mode (传统模式)
- dynamic mode （动态模式）

DPDK的大型**连续物理内存**的分配是使用大页面完成的。通过类似`rte_memzone_reserve`的`API`可以获取在大页中一段保证物理地址连续的内存区域。

`rte_memzone_reserve`这一类API对于动态模式应该是比较有用的，而`legacy`模式下的物理地址本身就是连续的。
（**对于跨两个页面之间的大页内存需要另外探究，实际测试时，malloc_heap能够将多个大页维护到一个malloc_elem中**）


## 传统内存模式

**legacy mode** :
在早期的DPDK版本中，EAL在默认参数启动时会保留系统中**所有可用的大页内存**，并且不允许在运行时从系统获取或释放大页内存，这些大页在application结束之前
不会归还给OS。并且application使用DPDK的rte_malloc库申请内存时，若剩余可用的内存不足，则直接返回错误。

此外，在这种模式下，EAL还会将所有申请的内存按照物理地址进行排序，从而构建一块巨大的IOVA连续的内存区域，以满足应用或驱动对IOVA连续内存的需求。


## 动态内存模式
在更新的DPDK版本中，EAL默认不启用传统内存模式来保留内存，转而切换到动态内存模式。

**dynamic mode**：即**按需申请大页**
在这种模式下，DPDK应用程序会根据实际应用的内存需求，动态增加和减少大页内存的使用。通过`rte_malloc`、`rte_memzone_reserve`或其它内存分配接口会自动从系统中保留更多的内存，并在不再需要时将内存释放回给系统。

==由于会动态地向OS申请内存，所以虚拟内存连续并不意味着物理内存连续==。
(对于向操作系统动态申请的内存，在application释放这些内存时，DPDK系统会将其归还给OS，不必等到application结束)

这种模式下任何内存分配所使用的大页都是动态增长和释放的。不能保证在此模式下分配的内存是 `IOVA` 连续的，（**IOVA即IO地址，在X86中等同物理地址**）

### 动态模式下预留内存
不过在动态内存模式情况下`-m`或`--socket-mem`参数仍然可以使用，但是其语义和`lagecy`模式有所不同，动态模式下`-m`或`--socket-mem`参数指定的是应用程序预留的{`BANNED`}最佳小内存。这部分内存应用程序不会释放，当需要申请更多的内存时应用程序可以超出这部分预留内存动态添加，为了可以限制应用程序所能使用的{BANNED}最佳大内存，动态内存提供了`--socket-limit`参数来指定当前socket所能使用的内存大小上限。

关系如下图所示：

![](attachments/Pasted%20image%2020240410154529.png)


### 动态内存的实现流程

![](attachments/Pasted%20image%2020240410154702.png)

其中动态内存的初始化入口是eal_dynmem_hugepage_init，与之对应老的lagecy模式的初始化入口是eal_legacy_hugepage_init。

动态内存初始化相关的数据结构，如下图所示：

![](attachments/Pasted%20image%2020240410154928.png)

新的DPDK内存管理是对每个页面（hugepage）创建一个memseg，并且将每个numa node的每个hugepage_size组成memseg_list，而不是像lagecy模式一样将多个连续的hugepage对应为一个memseg外。

还有一个重要变化就是，我们这里预留的memseg个数和系统可用的hugepge相当，并且只有通过-m或--socket-mem指定大小的内存在启动中才会真的进行hugepage文件的创建和mmap，同时初始化对应的memseg，其他memseg都是未初始化的，为的就是系统动态分配内存时可用于临时创建对应hugepage文件并初始化对应memeseg。

### 动态内存的问题
#### 动态申请大页并初始化开销大
**dynamic mode**：即**按需申请大页**
在这种模式下，DPDK应用程序会根据实际应用的内存需求，动态增加和减少大页内存的使用。
内存内存不足的时候，从系统中申请一块大页，然后进行大页的初始化。释放的时候，空闲的内存超过一个大页，有将大页还给系统。
==这样`dpdk`每次从系统中申请大页，并且进行初始化的时候，就会导致开销比较大==。

可能出现一种情况，底层程序使用 `rte_malloc` 申请内存，耗时可能不一致，大部分时间耗时都正常，偶发的存在耗时较长的长尾的情况，此时可能是内存不足，需要新申请一块大页，然后进行初始化，再从新申请的大页上进行分配的情况。



### IOVA连续性
**动态内存模式情况下的内存分配默认不保证是IOVA连续的**。

什么意思呢？换句话说内存分配只保证VA连续（这是肯定的，例如分配一个数据结构肯定是一个连续的虚拟地址空间），但不能保证这段内存在IO视角是连续的地址空间（即物理地址）。
再进一步解释就是，在VA作为IOVA的情况，内存分配保证了VA连续，自然就保证了IOVA连续；
但在PA作为IOVA时，内存分配就只能保证VA连续无法保证IOVA（PA）连续了。
为什么呢？因为从动态内存的实现上也可以看出来，频繁的内存分配/释放，肯定会很少有连续的的大块物理内存的。

那在lagecy（如DPDK 17.11）情况下为什么没有这个问题呢？
在VA作为IOVA的情况，无论是动态内存还是lagecy方式分配内存都是连续的VA，底层也是连续的IOVA（尽管可能跨PA空间，PA不连续）;但是在PA作为IOVA时，lagecy模式由于PA和VA空间是对应的，在PA作为IOVA时，如下图有三块连续的PA空间，那么DPDK初始就有三个memseg，对应三个IOVA空间，所以应用程序分配一块内存（这种情况不能跨memseg），如果是VA连续，一定是IOVA（PA）连续，但是在动态内存场景，如图中（DPDK 18.11），即使PA作为IOVA的情况，VA的组织和IOVA也没有任何关联，这种情况分配一块内存（是可以跨物理page的，也就是可以跨memseg的），所以底层IOVA（PA）不一定是连续的。
![](attachments/Pasted%20image%2020240410155643.png)

#### 小结
在VA作为IOVA时，动态内存和lagecy模式分配的内存都一样，都可以保证IOVA连续（因为VA是连续的）；
在PA作为IOVA时，lagecy分配内存可以保证IOVA（PA）连续，但是动态内存模式无法保证分配出的内存是IOVA（PA）连续的。

#### 问题
动态内存这个特点是比较有意义的，因为即使PA作为IOVA，我们正常的内存结构（如ring，mempool，哈希表等）也仅需要VA连续内存，而不需要底层物理内存连续。==但是有些特殊场景，比如网卡驱动需要参与DMA的mbuf内存必须需要IOVA连续怎么办呢？==

有以下三种方案：

1. 使用`vfio`驱动（前提是设备支持`iommu`）；
2. 使用`lagecy`模式；
3. 在动态模式的情况下，**分配内存使用RTE_MEMZONE_IOVA_CONTIG**，如 在调用`rte_memzone_reserve()`函数时指定`RTE_MEMZONE_IOVA_CONTIG`作为`flag`，将保证底层IOVA连续（或者申请失败）


### 内存相关的回调
动态内存模式也支持了一些内存管理的回调机制，主要由两个API。

#### 内存映射更改时的回调
**rte_mem_event_callback_register()**

  在lagecy模式下大页内存初始化完成后就固定不变了，但是在动态内存模式下应用的内存是会动态增加和减少的，对于有些模块是需要感知这些变化的，如vfio需要将整个内存pin住，所以就需要及时知道新增的内存，并将其pin住。因此DPDK提供了rte_mem_event_callback_register这个API，用于关系内存映射变化的模块注册相应函数。

#### 内存超限时的回调函数
**rte_mem_alloc_validator_callback_register****()**

前面我们介绍过，在动态内存情况下可以通过--socket-limit参数来指定当前socket所能使用的内存大小上限。有时候我们不希望应用程序超出这个限制就一定返回失败，但又希望能够感知这种情况。
因此可以通过rte_mem_alloc_validator_callback_register这个API注册回调函数，当应用程序申请的内存超过--socket-limit时注册函数就会被调用，我们可以在函数中输出一下警告信息，并做一些更温和的处理，如：可以接受超出限制的几百兆字节，但拒绝超出限制千兆字节的情况。

### 动态内存的大页映射管理
我们知道通过`--huge-dir`可以指定DPDK进程创建大页文件的目录，大页文件名称通常类似rtemap_0这种，通过--file-prefix可以指定大页文件的前缀，以便不同进程可以在同一个目录下创建不同的大页内存。

在动态内存模式下大页文件是随时申请创建和删除的，如果在DPDK进程存在内存泄露或者进程crash，则这些大页文件可能会残留无法删除。当然我们可以使用--huge-unlink参数，这个参数可以在每次mmap完大页就立刻删除，但是它有可能删除其他进程创建的大页文件，因此针对这种情况动态内存情况建议使用--in-memory参数。

此外，DPDK进程默认每次启动都会删除当然目录的大页文件，然后花费大量时间创建hugepage以及初始化（清除大页信息）。为此DPDK提供了--huge-unlink=never参数，如果设置了这个启动参数，在启动时默认不会删除和清理原有大页，启动时会将原有memseg标记为RTE_MEMSEG_FLAG_DIRTY，在申请新内存时会将这部分内存进行清理。


## 两种模式的区别
我们熟悉DPDK 17.11及早期版本的都知道，一般应用程序启动会通过-m或--socket-mem参数指定应用程序使用的内存大小，之后DPDK应用程序就会reserve相应大小的内存，然后整个程序运行期间调用rte_malloc()或rte_memzone_reserve()等内存分配结构都是从这个reserve内存池中进行分配的，超过reserve内存的大小将无法分配。

但在动态内存模式下，应用程序可以不再需要通过-m或--socket-mem来预留内存，应用程序启动是完全可以不占用什么内存，当调用rte_malloc()或rte_memzone_reserve()等接口时动态的从系统中分配内存，并注册到DPDK的内存管理中，同样在调用释放内存接口时也会动态的将内存进行释放（从DPDK内存管理中删除）。这样应用程序可以不需要事先估算需要的内存大小，而采用**按需分配**，更加灵活（不过对于使用hugepage时，系统还是需要预留足够的hugepage）。



## 其他

### 动态内存模式不感知numa
在函数rte_eal_memseg_init中，有如下代码：
```bash
#ifndef RTE_EAL_NUMA_AWARE_HUGEPAGES
    if (!internal_conf->legacy_mem && rte_socket_count() > 1) {
        RTE_LOG(WARNING, EAL, "DPDK is running on a NUMA system, but is compiled without NUMA support.\n");
        RTE_LOG(WARNING, EAL, "This will have adverse consequences for performance and usability.\n");
        RTE_LOG(WARNING, EAL, "Please use --"OPT_LEGACY_MEM" option, or recompile with NUMA support.\n");
    }
#endif
```

可以看到，如果没有使用legacy的内存管理方式，则内存是无法感知numa的，因此编译指定NUMA_AWARE也是无效的，如果希望内存感知numa就需要用老的legacy模式管理内存。



# 动态内存模式的问题以及解决
## 问题
在动态内存模式下，DPDK应用程序会根据实际应用的内存需求，动态增加和减少大页内存的使用。通过rte_malloc、rte_memzone_reserve或其它内存分配接口会自动从系统中保留更多的内存，并在不再需要时将内存释放回给系统。

存在一个问题，比如，业务的配置查询，需要先申请一个内存空间，然后进行释放。
如果在 通过 rte_malloc 申请一个内存时，剩余的内存不足，则需要从系统中申请一个新的大页，并且进行大页的初始化，然后从这个大页中获取需要的内存。业务操作完成，进行释放时，还需要将这个内存还给DPDK程序，DPDK程序发现剩余了一个完整的大页，可能还会将大页还给系统。

那么，DPDK程序不断的从系统中申请大页，归还大页，比较耗费性能。

## 场景
内存持有时间短（即申请后，很快就释放了）的场景：
（1）配置查询
（2）中间状态对应的内存
添加一个存在大量rs的使用conhash的vs，其实每次添加一个RS，conhash调度算法都存在一个中间的状态的内存的申请以及释放。

## 解决
对于业务逻辑中，内存持有时间短（即申请后，很快就释放了），那么应该考虑使用 动态模式中的预留内存。


# 其他
## 相关问题
**（1）大页如何获取并初始化、还有释放？**

通过`/proc/mounts`记录了 `hugetlbfs` 的挂载路径，在该路径上创建一个文件并`mmap`即可创建一个大页，`munmap`即可释放

**（2）单个大页是否能够保持连续，怎么做到的**

无论大页小页，单个页面内的物理地址在内核本身就是连续地。


**（3）相邻的两个页面之间的内存（物理和虚拟）是否连续？连续是怎么做到的**
相同的`numa node`中的两个相邻页面在legacy模式下是连续的，例如一个numa下存在8个大页，那么最终会得到一段8G大小的连续内存段。不同的numa则不连续。
而动态模式下两个页面之间是没有明确保障的。

**（4）一次malloc分配能否跨numa分配内存**
malloc以malloc_heap为一个单位，而malloc_heap在legacy模式下只会绑定连续内存。

**（5）如果进程本地numa没有内存页怎么办**
无需系统调用即可转向其他numa进行获取

## 系统中各numa剩余的大页内存的查看
```bash
numastat -m
```

![](attachments/Pasted%20image%2020240410152055.png)

## Hugepage Worker Stacks
通过`--huge-worker-stack[=size]`参数可以将DPDK线程的栈内存从hugepage中进行分配，还可以通过可选的size参数设置线程的栈大小，如果不指定size就使用系统的默认设置。

## 外部内存
在DPDK新版本中可以支持将外部内存注册给DPDK内存管理，比如自己应用通过非DPDK API malloc的内存或者mmap的内存。将这部分内存注册进DPDK的内存管理中，同样可以使用DPDK的内存API进行访问，详细使用方法可以参考DPDK代码中的 ./app/test/test_external_mem.c中的例子。

# 参考
```bash
DPDK 22.11内存管理变化解析 （+++++++++）
(http://blog.chinaunix.net/uid-28541347-id-5877488.html)


# DPDK内存管理中动态模式和legacy模式
https://xisme.cn/post/494103e2/
```