```table-of-contents
```

# 介绍
mmap函数是一种内存映射文件的方法，它可以将一个文件或设备映射到进程的地址空间中，使得进程可以像访问内存一样访问文件或设备。
mmap可以分为：文件映射和匿名映射。



# mmap 函数
```c
#include <sys/mman.h>  
void* mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset);  
  
// 内核文件：/arch/x86/kernel/sys_x86_64.c  
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,  
  unsigned long, prot, unsigned long, flags,  
  unsigned long, fd, unsigned long, off)
```

![](attachments/5aa10c882fc425ff5750515fce39fd9e.jpg)

## 参数
### addr 和 length

![](attachments/Pasted%20image%2020250806154731.png)

- addr ： 表示我们要映射的这段虚拟内存区域在进程虚拟内存空间中的起始地址（虚拟内存地址），但是这个参数只是给内核的一个暗示，内核并非一定得从我们指定的 addr 虚拟内存地址上划分虚拟内存区域，内核只不过在划分虚拟内存区域的时候会优先考虑我们指定的 addr，如果这个虚拟地址已经被使用或者是一个无效的地址，那么内核则会自动选取一个合适的地址来划分虚拟内存区域。我们一般会将 addr 设置为 NULL，意思就是完全交由内核来帮我们决定虚拟映射区的起始地址。

- length ：从进程虚拟内存空间中的什么位置开始划分虚拟内存区域的问题解决了，那么我们要申请的这段虚拟内存有多大呢 ？ 这个就是 length 参数的作用了，如果是匿名映射，length 参数决定了我们要映射的匿名物理内存有多大，如果是文件映射，length 参数决定了我们要映射的文件区域有多大。

>注： ==addr，length 必须要按照 PAGE_SIZE（4K） 对齐==。

### prot
指定映射区域的保护方式。可以是以下几种值的组合：  
```c
#define PROT_READ 0x1  /* page can be read */  
#define PROT_WRITE 0x2  /* page can be written */  
#define PROT_EXEC 0x4  /* page can be executed */  
#define PROT_NONE 0x0  /* page can not be accessed */
```
**PROT_READ** 表示该虚拟内存区域背后映射的物理内存是可读的。
    
**PROT_WRITE** 表示该虚拟内存区域背后映射的物理内存是可写的。
    
**PROT_EXEC** 表示该虚拟内存区域背后映射的物理内存所存储的内容是可以被执行的，该内存区域内往往存储的是执行程序的机器码，比如进程虚拟内存空间中的代码段，以及动态链接库通过文件映射的方式加载进文件映射与匿名映射区里的代码段，这些 VMA 的权限就是 PROT_EXEC 。
    
**PROT_NONE** 表示这段虚拟内存区域是不能被访问的，既不可读写，也不可执行。用于实现防范攻击的 guard page。如果攻击者访问了某个 guard page，就会触发 SIGSEV 段错误。除此之外，指定 PROT_NONE 还可以为进程预先保留这部分虚拟内存区域，虽然不能被访问，但是当后面进程需要的时候，可以通过 mprotect 系统调用修改这部分虚拟内存区域的权限。

### flags
指定映射区域的标志。可以是以下几种值的组合：  
```c
#define MAP_FIXED   0x10        /* Interpret addr exactly */  
#define MAP_ANONYMOUS   0x20        /* don't use a file */  
  
#define MAP_SHARED  0x01        /* Share changes */  
#define MAP_PRIVATE 0x02        /* Changes are private */

#define MAP_LOCKED 0x2000  /* pages are locked */  
#define MAP_POPULATE  0x008000 /* populate (prefault) pagetables */  
#define MAP_HUGETLB  0x040000 /* create a huge page mapping */
```
**MAP_SHARED**：与其他进程共享映射区域。  
**MAP_PRIVATE**：不与其他进程共享映射区域。  
**MAP_FIXED**：指定映射区域的起始地址。如果指定了这个标志，则 addr 参数必须为非 NULL。  
**MAP_ANONYMOUS**：不映射任何文件，而是映射一段匿名的内存区域。


### offset 和 fd
当我们将 mmap 系统调用参数 flags 指定为 `MAP_ANONYMOUS` 时，表示我们需要进行匿名映射，既然是匿名映射，fd 和 offset 这两个参数也就没有了意义，fd 参数需要被设置为 -1 。

当我们进行文件映射的时候，只需要指定 fd 和 offset 参数就可以了。

如果我们通过 mmap 映射的是磁盘上的一个文件，那么就需要通过参数 fd 来指定要映射文件的描述符（file descriptor），通过参数 offset 来指定文件映射区域在文件中偏移。

![](attachments/Pasted%20image%2020250806154722.png)

### 注意
在内存管理系统中，物理内存是按照内存页为单位组织的；在文件系统中，磁盘中的文件是按照磁盘块为单位组织的，内存页和磁盘块大小一般情况下都是 4K 大小，所以这里的 offset 也必须是按照 4K 对齐的。

# 内核中实现
## VMA

![](attachments/df13c9ad4dc3c851ce441c83304c097d.jpg)

# mmap映射分类
## 匿名映射和文件映射
操作系统对于物理内存的管理是按照内存页为单位进行的，而内存页的类型有两种：一种是匿名页，另一种是文件页。根据内存页类型的不同，内存映射也自然分为两种：一种是虚拟内存对匿名物理内存页的映射，另一种是虚拟内存对文件页的也映射，也就是我们常提到的匿名映射和文件映射。
## 共享映射和私有映射
根据 mmap 创建出的这片虚拟内存区域背后所映射的**物理内存**能否在多进程之间共享，又分为了两种内存映射方式：

- `MAP_SHARED` 表示共享映射，通过 mmap 映射出的这片内存区域在多进程之间是共享的，一个进程修改了共享映射的内存区域，其他进程是可以看到的，用于多进程之间的通信。
    
- `MAP_PRIVATE` 表示私有映射，通过 mmap 映射出的这片内存区域是进程私有的，其他进程是看不到的。如果是私有文件映射，那么多进程针对同一映射文件的修改将不会回写到磁盘文件上

## 四种组合

![](attachments/Pasted%20image%2020250806155829.png)

![](attachments/Pasted%20image%2020250806155854.png)

### 私有匿名映射
`MAP_PRIVATE | MAP_ANONYMOUS` 表示私有匿名映射，

#### 使用场景
##### 为进程申请虚拟内存
我们常常利用这种映射方式来申请虚拟内存，比如，我们使用 glibc 库里封装的 malloc 函数进行虚拟内存申请时，当申请的内存大于 128K 的时候，malloc 就会调用 mmap 采用私有匿名映射的方式来申请堆内存。因为它是私有的，所以申请到的内存是进程独占的，多进程之间不能共享。

##### 用在 execve 系统调用中
私有匿名映射还会应用在 execve 系统调用中，execve 用于在当前进程中加载并执行一个新的二进制执行文件：
```c
#include <unistd.h>  
  
int execve(const char* filename, const char* argv[], const char* envp[])
```


# mmap的使用场景
`mmap`是 Linux系统的一种内存映射机制，允许**将文件或设备直接映射到进程的虚拟地址空间**。进程通过操作内存指针即可读写文件，无需调用 `read`/`write` 等系统调用。mmap内存映射在Linux系统中广泛被使用，如：内存分配、零拷贝网络传输、进程间通信、大文件处理等。

## 内存分配
## 进程间通信
## 大文件处理


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


# 进程之间通过mmap共享内存
## 通过 `mmap` 将同一个文件或共享内存区域映射到多个进程的地址空间中
这种方式创建的共享内存具有以下特点：
- 各进程看到的是**同一段物理内存**：两个进程共享同一个文件映射的内存
- **内存改动可见于其他进程**
- 需要一定的同步机制（如加锁）来避免数据竞争

## 2个进程通过mmap共享，不加锁，可不可以？
如果为了高性能，2个进程通过mmap共享，不加锁，可不可以？比如其中一个进程作为只是作为统计进程，读取统计信息；另外一个进程是主体进程。

**可以，但要小心使用**。如果设计得当，**两个进程通过 `mmap` 共享内存而不加锁是可行的**，特别是当：
- 一个进程只读（比如统计进程）
- 一个进程只写（比如数据生产者）
- 对数据一致性的要求**不是特别严格**
- 或者你设计了**无锁通信协议**


### 合理情况下无锁 mmap 是可行的
场景如下：
> **进程 A：写入统计数据（例如计数器）**  
> **进程 B：周期性读取这些数据进行分析或打印**

你可以这么设计：
- 进程 A 定期更新结构体中的统计字段
- 进程 B 只读这些字段，不尝试修改
- 允许进程 B 读到“正在写的值”，即 **可能读取到中间状态或旧值**

#### 注意
##### 内存访问一致性问题

如果写入的值跨越多个字节（比如 `int64_t`），而 CPU 没有原子地写入它：
- 读者进程**可能看到一个被撕裂的值**（例如，低位是新值，高位是旧值）

解决方式：
- 对共享变量使用原子类型，如 GCC 的 `__atomic_store` / `__atomic_load`
- 或仅共享小于等于 `word size` 的字段（如 32 位系统中共享 `int`）

##### 编译器/CPU 重排序问题
即使你写的是：
```c
counter++;
flag = 1;
```

编译器或 CPU 可能会将其重排序为：
```c
flag = 1;
counter++;
```

导致读者看到 `flag == 1`，但 `counter` 还没更新。

解决方式：
- 使用原子操作（如 `__atomic_store`、内存屏障等）来控制顺序

##### 可能读到旧值或不一致值
对于统计类数据（如计数器），即使读取时存在短暂不一致，也不会影响准确性太大，可以接受
```c
// 主进程周期性更新共享数据结构 stats
stats.requests += 1;
stats.errors += 1;

// 统计进程周期性读取 stats，打印日志
printf("Req: %d, Errors: %d\n", stats.requests, stats.errors);

```

只要写入频率不远高于读取频率，读取“延迟”几次更新是可以接受的。

#### 高级优化技巧
##### 加版本号 + 再读确认一致
这是经典 lock-free 技巧：
```c
struct Stats {
    uint64_t version;
    uint64_t req_count;
    uint64_t err_count;
    uint64_t version_check;
};

```

(1) 写入时顺序写入：
```bash
- `version += 1`
- 写入字段
- `version_check = version`
```

(2)  读取操作：
```c
do {
    uint64_t v1 = stats->version_check;
    req = stats->req_count;
    err = stats->err_count;
    uint64_t v2 = stats->version;
} while (v1 != v2);

```
这个模式称为 “versioned snapshot”，在共享内存读取一致状态时非常有效，无需加锁，也不会丢数据。

##### 双缓冲区（Double Buffering 的 lock-free 技巧）
注：双缓冲适合状态快照，对于累计类数据需要注意，需要读取上一个 buffer 值进行累加。

```c
struct Stats {
    int request_count;
};

struct Shared {
    struct Stats buffer[2];
    int current_index;
};


1> 写操作：
int old_index = shared->current_index;
int new_index = !old_index;

shared->buffer[new_index].request_count = shared->buffer[old_index].request_count + 1;

shared->current_index = new_index;


2> 读操作：
int idx = shared->current_index;
printf("Reader: %d\n", shared->buffer[idx].request_count);
```


##### 原子类型
使用 C11 或 GCC 提供的原子变量：
```c
#include <stdatomic.h>

typedef struct {
    atomic_int requests;
    atomic_int errors;
} Stats;


1> 写入
atomic_fetch_add(&stats->requests, 1);


2> 读取
int req = atomic_load(&stats->requests);
```
也可以用 `__atomic_store_n(&counter, value, __ATOMIC_RELEASE);` 等


# 其他
## mmap和 Direct IO对比

内存映射和 Direct IO 都是用来提高文件读写性能的技术，但它们之间有一些不同。

首先，内存映射是将文件映射到进程的地址空间中，而 Direct IO 是直接使用文件描述符进行读写操作。因此，内存映射可以充分利用虚拟内存系统的优势，而 Direct IO 则可以避免缓存的影响。

其次，内存映射可以实现文件的共享访问，而 Direct IO 则不行。这是因为 Direct IO 会绕过文件系统缓存，而文件系统缓存是用来实现文件共享访问的。

最后在内存映射中，修改过的数据会被缓存在内存中，并不会立即写回到磁盘中。如果需要将数据写回到磁盘中，可以使用 msync() 函数或者 munmap() 函数来实现。而 Direct IO 是可以直接将数据写入磁盘的。

# 参考
```bash
# 3 万字 + 40 张图 ｜ 拆解 mmap 内存映射的本质！
https://mp.weixin.qq.com/s/sLoiOevTxIonrgLa7yWJkw

# 万字图文拆解零拷贝：从DMA到mmap，彻底搞懂高性能IO
https://mp.weixin.qq.com/s/IHwuaiggixTzpataKMLl1g

# 同样是mmap崩溃，为什么有时是SIGSEGV，有时是SIGBUS？99%的开发者都分不清
https://mp.weixin.qq.com/s/fUO4hu1QrSbCVwYCfmGnLQ

# mmap图解
https://mp.weixin.qq.com/s/W7dcBy3IuMwKFjMScOVWgg
https://mp.weixin.qq.com/s/ZSaZggYjYHYtVTof0Dmz4A

# Linux Mmap映射：优化文件访问和共享内存
https://zhuanlan.zhihu.com/p/685848279

# 大疆二面追问：如何通过内存映射提高文件读写性能？
https://mp.weixin.qq.com/s/tzRcbH1fWta4hSkUqoWCEQ


# mmap的使用
https://mp.weixin.qq.com/s/hmazpJTjd6TxZaV1KvoXkA
```