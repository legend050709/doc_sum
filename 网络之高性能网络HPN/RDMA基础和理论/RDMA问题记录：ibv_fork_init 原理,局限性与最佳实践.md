```table-of-contents
```
# 概述
本文介绍RDMA verbs函数: `ibv_fork_init()` 的出现原因、作用原理、局限性，并提出RDMA编程中关于fork场景的最佳实践。

# 背景
## RDMA内存读写原理

1. 应用申请MR内存块
2. 调用ibv_reg_mr，在RNIC中注册内存块
3. RNIC对内存块的读写采用DMA方式，读写过程应用进程无感知

具体过程不多赘述。

# fork 带来的问题
## Copy-On-Write (COW)

COW是对fork创建子进程过程的一个性能优化手段。

当fork刚刚创建好子进程时，父子进程用的是相同的物理空间（内存区），子进程的代码段、数据段、堆栈都是指向父进程的物理空间，也就是说，两者的虚拟空间不同，但其对应的物理空间是同一个。

同时，相关内存页被标记为只读，无论父子进程对其进行的首次写入操作，都会触发page fault，进而会重新分配内存页，进行内存拷贝，然后谁触发的写操作，就更新谁的进程页表 (见: [Does parent process lose write ability during copy on write?](https://stackoverflow.com/questions/49246735/does-parent-process-lose-write-ability-during-copy-on-write))。

## 问题

考察一个典型的场景：

某进程P注册的MR对应内存页记为R。P调用fork创建子进程C，C调用exec系列函数加载新的程序。此时存在如下2种可能的时序：

1. C执行exec系列函数完成之前，P没有对R的写入（例如，P主动block直到C运行结束）： 此后，C执行exec完，替换自身页表，P会一直保留原有的内存页R，不会触发COW，不会出问题
    
2. 在C执行完成exec之前，P对R进行写入操作： 此时，P触发COW，系统复制了一份R的副本R’，并用R’更新了P的页表，如图：

![](attachments/Pasted%20image%2020251208195806.png)

但是，RNIC仍持有原先的R做DMA(硬件RNIC中的MTT??)，而P由于页表(软件程序的页表)改变，错误的从R’ 读写RDMA数据，导致与RNIC不一致，通信数据出现错误。

> 注意，虽然rdma mr的内存受`mlock`保护，也不会改变这种行为。`mlock`只是保证这些内存页不会被换到swap当中，即所谓的pinned内存。

# `ibv_fork_init` 原理

为了解决上述问题，RDMA开发人员向内核贡献了`MADV_DOFORK`/`MADV_DONTFORK` 两个madvise的flag ([madvise MADV_DONTFORK/MADV_DOFORK](https://lwn.net/Articles/171956/))，并且新增了`ibv_fork_init`函数 ([Make fork() work for verbs consumers](https://github.com/linux-rdma/rdma-core/commit/aa953da156333c6f06cfa93b4c747d838e7e693d))。

> 注意：这些措施仍然没有完美解决问题，见下一节


## `ibv_fork_init`

该函数需要在使用RDMA功能之前开启。开启后，基本只会影响`ibv_reg_mr`和`ibv_dereg_mr`两个函数。

![](attachments/Pasted%20image%2020251208201204.png)

### 原理

开启`ibv_fork_init`后，调用`ibv_reg_mr`时，会对相关内存，调用`madvise` 打标记`MADV_DONTFORK`。

由于`madvise`需要对整页进行做标记，但是mr内存不一定是整页，所以verbs库内部对所有注册的MR内存地址段进行了记录：
建立红黑树，记录全部mr，以及mr所在的内存页
    
(1) 每一次调用`ibv_reg_mr`时，判断当前mr所在的内存页的起止地址，检查是否红黑树中存在记录
如果不存在，则加入红黑树并调用`madvise`打标记`MADV_DONTFORK`；

(2) 每一次调用`ibv_dereg_mr`，使用当前mr地址查询红黑树；
如果该mr所在内存页没有包含其他已注册的mr，则调用`madvise`打标记`MADV_DOFORK`；


## `MADV_DONTFORK`标记

`madvise` 通过将某内存页标记为`MADV_DONTFORK`，强迫fork场景下该页不会被复制


### `MADV_DONTFORK` 标记的效果

通过对所有已注册的MR所在内存页打`MADV_DONTFORK`标记，创建子进程后，MR所在内存页不会触发COW拷贝，避免了前面所说的COW带来网卡DMA内存地址不一致的问题。

**`MADV_DONTFORK`** 标志是 Linux 内核提供的一种机制，专门用于通知内核在 `fork()` 操作中，**不要**将该段虚拟内存区域（VMA）复制到子进程中。

#### 默认 `fork` 行为 (COW)

- **页表复制：** 父进程页表中的 VMA 被复制到子进程。
    
- **物理页共享：** 父子进程的页表都指向相同的物理页。
    
- **权限设置：** 共享的物理页被设置为 **只读**。
    
- **目的：** 实现 **写时复制 (COW)**。父进程或子进程的第一次写入操作会触发页故障，内核复制物理页，从而保证各自的内存独立性。
    

#### `MADV_DONTFORK` 后的 `fork` 行为

当 MR 内存页被标记为 `MADV_DONTFORK` 后，`fork()` 的行为是：

- **不复制 VMA (页表项)：** 内核在为子进程创建内存空间时，会 **跳过** 包含 `MADV_DONTFORK` 标记的这个 VMA。
    
- **子进程中 VMA 缺失：** 子进程的虚拟地址空间中将 **没有** 对应的 MR 地址映射。
    
- **结果：** MR 所在的内存页及其内容**不会**被复制到子进程，也不会以 COW 方式共享。子进程对这块虚拟地址是 **不可见的** 或 **无法访问的**。


##### 父进程在 `fork` 后的写操作

如果父进程在 `fork` 之后，对 MR 内存进行了写操作，会发生什么？

对父进程的影响：

- **正常写入：** 父进程写入 MR 内存是**正常且允许**的。
    
- **不触发 COW：** 由于这块内存已经被父进程（通常是 RDMA 应用程序本身）在 `fork` 之前注册并锁住（可能还涉及 `mlock`），它已经是父进程可读写的私有内存。**写入操作不会触发 COW**，因为它已经不是 COW 共享页。
    
- **数据一致性：** 父进程对 MR 内存的修改会立即生效。

##### 对子进程的影响：**访问异常**

子进程无法访问这块内存区域。

- **访问行为：** 如果子进程尝试访问 MR 所在的虚拟地址：
    - **如果是合法的 VMA 地址，但页表缺失：** 会立即触发 **页故障 (Page Fault)**，重新分配物理内存。
    - **如果是已分配但被 `MADV_DONTFORK` 标记的区域：** 子进程会遇到 **Segmentation Fault (段错误)**。这是因为该虚拟地址在子进程的地址空间中找不到有效的映射。


### `MADV_DONTFORK` 标记根本目的
`ibv_fork_init` 的目的正是为了避免在 `fork` 场景下，**RDMA 硬件** 和 **OS 内存** 之间的地址不一致问题：

通过移除子进程的映射，`ibv_fork_init` 确保了 MR 对应的物理页**永远不会被复制**，从而保证了父进程的 RDMA 硬件（它记录了旧物理页的 DMA 地址）所引用的内存地址在程序生命周期内保持稳定和有效。


## fork之后，子进程立马进行exec，则不需要提前`ibv_fork_init`


# `ibv_fork_init`的局限性

`ibv_fork_init`仍然存在问题，根源是设计上的一个矛盾：`ibv_reg_mr`允许任意内存地址段，而`madvise`只能对整个内存页进行标记。

通过上面对`ibv_fork_init`的行为描述，可以很容易看出，如果mr的地址段不是整内存页，则`ibv_reg_mr`会对mr之外的内存做标记，如图：

![](attachments/Pasted%20image%2020251208202022.png)

蓝色部分是注册的MR，红色部分是其他内存，但是红色部分也会被标记为`MADV_DONTFORK`

如果是fork-exec的用法，子进程不会使用父进程的任何内存，则多标记这一部分内存不会造成影响。

如果是单纯fork，并且子进程运行时访问到了红色部分的内存，则实际上子进程访问的是父进程的内存，就会出现内存访问的问题 (见 [A Cursed Bug](https://blog.nelhage.com/post/a-cursed-bug/) 、[rdma-core: ibv_dontfork_range should not round up to page boundaries](https://lore.kernel.org/linux-rdma/CAPSG9dZ-dkWPcbXECQeZyvOHu7M+vfrX+jJDe+fxY6_iSnQyKw@mail.gmail.com/))。

## 相关解决方案

### MR整页限制

最新的rdma-core版本中合入了一个提交 ( [verbs: Allow aligned address & size only for fork init #1222](https://github.com/linux-rdma/rdma-core/pull/1222)) ：在开启`ibv_fork_init`后，强迫所有mr的起始地址必须是内存页对齐，长度必须是内存页整数倍，其实就是强迫只能按照整页去注册mr，修复了`ibv_fork_init`的局限性

### copy-on-fork

最新的内核代码包含了copy-on-fork的功能 (https://github.com/torvalds/linux/commit/70e806e4e645019102d0e09d4933654fb5fb58ce 、 https://lore.kernel.org/linux-rdma/20210418121025.66849-1-galpress@amazon.com/) ，在fork时，DMA内存不再执行COW策略，而是直接复制到子进程，因此没有必要再做`MADV_DONTFORK`的标记。

相应的，为了使用该内核功能，verbs提供了一个新的功能 （见[Report when ibv_fork_init() is not needed #975](https://github.com/linux-rdma/rdma-core/pull/975)） ：

引入函数：`ibv_is_fork_initialized()`，如果返回的是`IBV_FORK_UNNEEDED`，说明当前内核支持上述copy-on-fork特性，不必开启`ibv_fork_init()`。

# 使用总结

- 如果确定应用不存在任何fork/popen等创建子进程的行为，则不应该开启`ibv_fork_init`，因为根据前面所述，开启后，verbs会建立红黑树管理全部MR，如果注册MR非常频繁，这部分操作会引起性能下降 (见：[prov/verbs misleading use of ibv_fork_init() #4974](https://github.com/ofiwg/libfabric/issues/4974))
    
- 如果需要进行创建子进程的操作，可以wait到子进程执行结束，也可以使用`posix_spawn`(主进程阻塞直到exec执行完毕)
    
- 如果主进程不能等待：
    
    - 较新版本verbs库，则应该首先调用`ibv_is_fork_initialized`，如果返回值是`IBV_FORK_UNNEEDED`，则不需要调用`ibv_fork_init`；如果返回值是`IBV_FORK_DISABLED`，则需要开启`ibv_fork_init`
        
    - 旧版本verbs库，需要执行`ibv_fork_init`，并且在设计上规避频繁注册MR的行为，避免性能下降


# 参考
```bash
# ibv_fork_init 原理,局限性与最佳实践
http://jiangzhuti.me/posts/ibv_fork_init%E5%8E%9F%E7%90%86,%E5%B1%80%E9%99%90%E6%80%A7%E4%B8%8E%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5
```