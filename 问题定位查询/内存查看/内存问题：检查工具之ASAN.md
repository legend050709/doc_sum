```table-of-contents
```

# ASAN概述

首先，先介绍一下 [Sanitizer](https://github.com/google/sanitizers)项目，该项目是谷歌出品的一个开源项目。
它通过==编译期插桩 + 运行期影子内存==的组合，能够精确检测多类内存访问错误，并在发生错误时给出带完整调用栈的报告。
该项目包含了 `ASAN`、`LSAN`、`MSAN`、`TSAN`等内存、线程错误的检测工具，这里简单介绍一下这几个工具的作用：

- **ASAN**: 内存错误检测工具，在编译命令中添加`-fsanitize=address`启用
- **LSAN**: 内存泄漏检测工具，已经集成到 ASAN 中，可以通过设置环境变量`ASAN_OPTIONS=detect_leaks=0`来关闭`ASAN`上的`LSAN`，也可以使用`-fsanitize=leak`编译选项代替`-fsanitize=address`来关闭ASAN的内存错误检测，只开启内存泄漏检查。
- **MSAN**: 对程序中未初始化内存读取的检测工具，可以在编译命令中添加`-fsanitize=memory -fPIE -pie`启用，还可以添加`-fsanitize-memory-track-origins`选项来追溯到创建内存的位置
- **TSAN**: 对线程间数据竞争的检测工具，在编译命令中添加`-fsanitize=thread`启用  

# 基于Linux内核的标准ASAN

`ASAN`，全称 AddressSanitizer，可以用来检测内存问题，例如缓冲区溢出或对悬空指针的非法访问等。


ASan可以检测到：`UAF`、`Heap buffer overflow`、`Stack buffer overflow`、`Global buffer overflow`、`Use after return`、`Use after scpoe`、`Initialization order bugs`、`Memory leak`。

## ASAN的作用

• （1）缓冲区溢出
```bash
① 堆内存溢出  
② 栈上内存溢出  
③ 全局区缓存溢出
```

• （2）悬空指针（引用）
```bash
① 使用释放后的堆上内存  
② 使用返回的栈上内存  
③ 使用退出作用域的变量


- `stack-use-after-scope`, 栈上变量在作用域失效后仍被使用
- `stack-use-after-return`, 栈上变量在函数返回后仍被使用（默认未开启）
```

• （3）非法释放
```bash
① 重复释放  
② 无效释放
```

• （4）内存泄漏

• （5）初始化顺序导致的问题

### 栈上对象检测
ASan 针对栈上对象主要提供 3 种检测手段，分别是

**（1）`stack-buffer-overflow`/`stack-buffer-underflow`, 针对栈上变量 out-of-bound 的检测**
```c
- `stack-buffer-overflow`  
    含义：访问超过栈上数组或缓冲区边界的内存。

- `stack-buffer-underflow`  
    含义：访问缓冲区起始地址之前的内存（负方向越界）。

比如：
char buf[8];
buf[8] = 'a';   // overflow
buf[-1] = 'a';  // underflow
```

**（2）`stack-use-after-scope`, 栈上变量在作用域失效后仍被使用**
含义：局部变量已经离开其生命周期（scope），但仍被访问。
```c
比如：
int *p;
{
    int x = 10;
    p = &x;
}
printf("%d\n", *p); // use-after-scope
这里 `x` 已经离开作用域。
```

**（3）`stack-use-after-return`, 栈上变量在函数返回后仍被使用（默认未开启）**
含义：函数已经返回，但外部仍然使用该函数栈帧中的局部变量地址。
```c
int* foo() {
    int x = 10;
    return &x;
}

int *p = foo();
printf("%d\n", *p); // use-after-return
```

### 动态分配对象的检测

ASan 针对堆上对象主要提供 3 种检测手段，分别是:

**(1) `heap-buffer-overflow`, 针对堆上对象 out-of-bound 的检测**

```c
int main() {
    char* p = new char[13];
    p[-1] = 'a';
}
```

**(2) `heap-use-after-free`, 针对堆上对象被释放后仍被使用**
```c
int main() {
    char* p = new char[13];
    delete[] p;
    p[0] = 0;
}
```

**(3) `double-free`, 针对堆上对象的重复释放**
```cpp
int main() {
    char* p = new char[13];
    delete[] p;
    delete[] p;
}
```
除此之外，ASan 还会在 alloc/dealloc 方法不匹配时报错。

### 非对齐内存访问越界

```cpp
int main() {
    char* buf[8];
    int* p = (int*)(buf + 6);
    *p = 1; // [6, 9]
}
```



## 概念

### 红区 (Red Zone) 和  影子内存 (Shadow Memory)

**影子内存 (Shadow Memory)**：
每 8 字节真实内存对应 1 字节影子内存。影子值编码含义：
- 0x00：该 8 字节全部可访问
- 0x01~0x07：前 N 字节可访问，后 (8-N) 字节不可访问
- 0xfa：红区 (Red Zone) 不可访问
- 0xfd：已释放 (Freed) 内存不可访问

**红区 (Red Zone)**：
在每次分配前后插入不可访问区域，越界访问会被拦截。

![](attachments/Pasted%20image%2020260503084733.png)

#### 红区 (Red Zone)
大多数的内存检测工具采用以下步骤进行检测，比如通过在被保护的堆，栈、全局变量周围建立标记为中毒状态的红区（redzones），如下图1所示，在绿色区域表示可访问的进程内存，红色区域表示在它周边插入的红区。

#### 影子内存 (Shadow Memory)

通过影子内存的状态值来存储真实内存中的每个字节的访问是否安全，这样的影子内存需要一大块连续的虚拟地址空间，如图1所示蓝色的影子区域（shadow region），在程序初始化的时候申请好这片空间。当然了查找影子内存需要非常快才行，一般会构建查找表(lookup table)来记录快速查找每块区域的访问权限。这个查找表并未真正分配，而是在进程启动的时候保存，在需要的时候访问。编译的时候，在每一次内存访问前编译器会帮我们采用代码插桩技术来针对应用程序的每次load和store对影子内存进行检查，它会影响每一次内存访问，并在代码前缀加上检测，检测对应的影子区域的状态值是否是可访问状态。

动态库可以在程序运行的时候做实时检测，如果影子内存的状态值是在可访问范围内, 则允许你继续，如果是在不可访问范围内, 则内存中毒，它会跟踪程序，并生成诊断报告。

AddressSanitizer(ASan)是google开发的内存检测工具，检测方法与上述类似，突出的是它使用非常高效的映射机制和紧凑的影子编码方法来提高效率。

注意到malloc函数的返回地址基本是以8字节对齐的, 所以对于堆上的8字节内存空间, 只有9种状态, 而9种状态只需要1个字节就可以表示, 也就是说只使用1/8的虚拟地址空间的影子内存就可以描述所有的地址。如图2所示，左边是进程内存，右边是影子区域内存，绿色表示可以访问，红色表示不可以访问。 

![](attachments/Pasted%20image%2020260503085103.png)

（1）当进程中的8个字节都是绿色的，也就是全部可以访问，那在影子区域存储的状态值是0；

（2）如果是进程中的8个字节都是红色，如最后一行所示，它是都不能访问的，那么它就是负值。采用不同的负值来表示不同类型的不可访问地址，程序中是以“f”开头表示负数，比如“fa”是堆，“f1”是栈；

（3）还有部分区域可以访问，部分不可以访问，也就是图2所示的中间部分，前N个字节可以访问，那么它的影子状态值是N，剩下的8-N就是不可以访问的。

```bash
而 shadow memory 的值也可以分成 3 种状态来讨论:
2. 8 字节的数据可读写，则 shadow memory 的值为 0
3. 8 字节的数据不可读写，则 shadow memory 的值为**负数**，如 `0xfa` 表示堆左边的 redzone、`0xf1` 表示栈左边的 redzone. ASan 也根据这个值在报错的时候输出对应的错误类型，如区分 `heap-buffer-underflow`/`stack-buffer-underflow`
4. 前 k 个字节可读写，后 8 - k 个字节不可读写，则 shadow memory 的值为 k，k 的取值范围为 `[1, 7]`
```

这种映射机制可以构建高效的查找方式，通过原始指针的值除以8，再添加offset，也就是在内存影子的位置添加。

==实际地址到影子内存地址的映射公式：`Shadow = (Addr >> 3) + ShadowOffset`==

### 释放后隔离区(quarantine) 
释放的内存不会立即回收，而是进入 quarantine 区域，标记为 0xfd，防止被快速复用导致 use-after-free 难以检测。

## 原理

`ASAN`的内存检测方法与`Valgrind`的`AddrCheck`工具很像，都是使用`shadow内存`来记录应用程序的每个字节是否可以被安全的访问，在访问内存时都对其映射的`shadow内存`进行检查。

但是，`ASAN`使用一个更具效率的`shadow内存`映射机制和更加紧凑的内存编码来实现，并且除了堆内存外还能检测栈和全局对象中的错误访问，且比`AddrCheck`快一个数量级。

`ASAN`由两部分组成：**代码插桩模块**和**运行时库**。
- 代码插桩模块会修改代码使其在访问内存时检查每块内存访问状态，称为`shadow 状态`，以及在内存两侧创建`redzone`的内存区域。
- 运行时库则提供一组接口用来替代`malloc`和`free`以及相关的函数，使得在分配堆空间时在其周围创建`redzone`，并在内存出错时报告错误。

### 编译器插桩（代码插桩模块）
编译器在每次内存访问前插入影子内存检查代码。
前面介绍的在每一次内存访问前编译器会帮我们在代码前缀加上检测。

针对每一次**内存读写**，编译器都会插桩（插入判断逻辑），判断地址是否被**投毒**(poisoned)。
在插桩前，代码是这样的:
```c
*addr = ...; // or ... = *addr;
```
在插桩后，代码就变成了这样:
```c
if (IsPoisoned(addr)) {
  ReportError(addr, kAccessSize, kIsWrite);
}
*addr = ...; // or ... = *addr;
```



#### 不同内存大小访问的问题

下面是8 bytes内存访问代码。
原始代码如蓝色所示，红色代码是编译器插入的指令。既然是访问8 bytes，必须要保证对应的影子内存的状态值必须是0，否则肯定是有问题。
那么如果访问的是1,2 或4 bytes，该如何检查呢？如图所示，只需要修改一下if判断条件即可，这里不做过多算法推算。
```bash
if (*shadow && *shadow < ((unsigned long)addr & 7) + N); //N = 1,2,4
```
![](attachments/Pasted%20image%2020260503085411.png)

考虑默认情况下 `scale` 为 3 的场景，`IsPoisoned()` 会遇到 2 种情况:
（1）部分访问，即实际访问的字节数小于 8 字节
（2）完全访问，即实际访问的字节数大于等于 8 字节


**在第 1 种情况下**，访问的内存可能处于最末端抑或是内存有效长度小于当前的访问字节数的大小。
![](attachments/Pasted%20image%2020260503090818.png)

```bash
5. 当访问 `a[0]`、 `a[1]` 的时候，发现对应的 shadow memory 为 0x00, 判定为安全
6. 当访问 `a[2]`、 `a[3]` 的时候，发现对应的 shadow memory 为 0x00, 判定为安全
7. 当访问 `a[4]`、 `a[5]` 的时候，发现对应的 shadow memory 为 0x00, 判定为安全
8. 当访问 `a[6]` 的时候，发现对应的 shadow memory 不为 0x00, 那么访问这块内存可能是不安全的，需要更详细的判断才能确定访问是否安全
```

已知当前访问的地址为 `addr`、访问字节数为 `size`, 且 `memToShadow(addr)` 不为 0. 显然，当前的访问操作涉及的地址范围为 `[addr, addr + size)`，而它实际安全的访问范围为 `[p, p + memToShadow(addr))`, 其中 `p` 为 `addr & ~0x7` 表示当前以 8 字节为单元的起始地址，`memToShadow(addr)`表示 `addr` 地址对应 shadow memory 的内存值。显然 `p <= addr`， `addr - p == (addr & 0x7)`.

![](attachments/Pasted%20image%2020260503090955.png)


即只要满足 `addr + size - 1 >= p + memToShadow(addr)` 就能够说明当前的访问已经越界了。将这个表达式化简可以得到 `(addr & 0x7) + size - 1 >= memToShadow(addr)`. 此外也可以证明，在访问到 redzone 时 `memToShadow(addr)` 为负数，表达式恒成立。


**在第 2 种情况下**，由于访问的字节数已经大于等于 8 了，所以可以直接检测对应的 `memToShadow(addr)` 的值，如果不为 0 那么一定是有问题的。

综上，可以用如下的伪代码来描述 `IsPoisoned()` 的逻辑:
```c
const uint8_t s = memToShadow(addr);
if (size < 8) {
    if (s != 0) {
        if ((addr & 0x7) + size - 1 >= s) {
            ReportError(...);
        }
    }
} else {
    if (s != 0) {
        ReportError(...);
    }
}
```

### 动态运行库/运行时库(libasan.so)

在应用程序启动时，整个影子空间将会被分配，以保证程序其他部分无法使用它。每种平台，定义了推荐的默认的常数补偿，在linux x86_64上，shadow的offset是`0x00007fff8000`。

运行时，会对glibc中的函数进行替换以便检测内存错误，即==运行时插桩，malloc和free将会被特定的实现替换掉，malloc会在返回的内存区域周围，设置红区，红区将会被标记为不可访问的或者说是有毒的(poisoned)==。

Free函数会对整个内存区域染毒，同时将它放到隔离区(quarantine)。

默认情况下，malloc和free会记录当前的调用栈以为问题发生的时候提供更多日志信息，它可以快速定位到内存是什么时候怎么样被malloc或free的。

当发现内存出错，会使用启发法，来关联错误地址到有效地址的对象，记录并汇报错误信息。




### 整体流程

（1）在应用程序启动时，整个影子空间将会被分配，以保证程序其他部分无法使用它。
每种平台，定义了推荐的默认的常数补偿，在linux x86_64上，shadow的offset是`0x00007fff8000`。

（2）运行时，会对glibc中的函数进行替换以便检测内存错误，即运行时插桩。
malloc和free将会被特定的实现替换掉，malloc会在返回的内存区域周围，设置红区，红区将会被标记为不可访问的或者说是有毒的(poisoned)。

Free函数会对整个内存区域染毒，同时将它放到隔离区(quarantine)。

默认情况下，malloc和free会记录当前的调用栈以为问题发生的时候提供更多日志信息，它可以快速定位到内存是什么时候怎么样被malloc或free的。

当发现内存出错，会使用启发法，来关联错误地址到有效地址的对象，记录并汇报错误信息。


## ASAN和Valgrind对比
根据检测结果显示，ASAN可能导致性能降低`2`倍左右，比`Valgrind`（官方给的数据大概是降低`10-50`倍）快了一个数量级。

而且相比于`Valgrind`只能检查到堆内存的越界访问和悬空指针的访问，`ASAN` 不仅可以检测到堆内存的越界和悬空指针的访问，还能检测到栈和全局对象的越界访问。

![](attachments/Pasted%20image%2020260503090255.png)


## 如何使用ASAN
基于Linux C/C++的程序，使用基于glibc提供的 `malloc/free`内存申请释放函数（非DPDK的内存申请释放），只需要在编译命令中加上`-fsanitize=address`检测选项就可以让`ASAN`在你的项目中大展神通。

9. 打开了调试标志`-g`，这是因为当发现内存错误时调试符号可以帮助错误报告更准确的告知错误发生位置的堆栈信息，如果错误报告中的堆栈信息看起来不太正确，请尝试使用`-fno-omit-frame-pointer`来改善堆栈信息的生成情况。
10. 如果构建代码时，**编译**和**链接**阶段分开执行，则必须在编译和链接阶段都添加`-fsanitize=address`选项。

# DPDK ASAN

## 背景
### DPDK的内存管理默认不支持ASAN
Glibc是linux 系统中最底层的API，ASan是基于glibc 实现，==标准 ASAN 通过 hook malloc/free== 来感知内存状态。DPDK 绕过 libc 自行管理 hugepage 内存，分配路径为：
```c
rte_malloc / rte_zmalloc
    └─ malloc_heap_alloc
           └─ heap_alloc
                  └─ malloc_elem_alloc   ← 不经过 libc malloc
```
ASAN 对这段路径完全无感知。如果没有适配，DPDK 分配的内存对 ASAN 来说全是"可访问"的（影子值 0x00），越界写也不会被检测到。
因此 DPDK 引入 RTE_MALLOC_ASAN，在关键操作点手动维护影子内存，"教会" ASAN 理解 DPDK 内存的边界。


### 现有DPDK内存检查的缺陷 
现有的DPDK已经有内存管理的机制，是通过检测可用内存前后区域是否被修改来判断是否越界，但是现有的DPDK实现**只有在rte_free的时候才做内存检测**，甚至发生内存问题的时候, 非常少且有效的调试信息输出来帮助定位问题，如果问题带有一定的随机性且很难重现，让人很崩溃。


## 作用

|错误类型|英文名|DPDK 是否支持|
|---|---|---|
|堆缓冲区溢出|heap-buffer-overflow|✅ 支持（通过 redzone）|
|使用已释放内存|heap-use-after-free|✅ 支持（通过 freezone）|
|栈缓冲区溢出|stack-buffer-overflow|✅ 支持（编译器原生）|
|全局缓冲区溢出|global-buffer-overflow|✅ 支持（编译器原生）|
|重复释放|double-free|✅ 支持（编译器原生）|
|内存泄漏|memory-leak|⚠️ 需配合 LeakSanitizer|
|野指针访问|wild pointer|⚠️ 可检测但报告不完整|

### heap-buffer-overflow（堆缓冲区溢出检测）

#### 核心机制：Red Zone

在用户数据两侧插入不可访问的红区（影子值 0xfa），任何超出 `[ptr, ptr+user_size)` 范围的访问都会命中红区。

#### 设置时机

|调用路径|代码位置|触发时机|
|---|---|---|
|rte_malloc → malloc_socket → malloc_heap_alloc → heap_alloc|malloc_heap.c:251|普通分配完成后|
|rte_zmalloc → 同上路径|malloc_heap.c:251|零初始化分配完成后|
|heap_alloc_biggest|malloc_heap.c:272|最大块分配完成后|
|rte_realloc_socket（原地扩展成功）|rte_malloc.c:217|原地 resize 成功后|

调用形式：asan_set_redzone(elem, user_size)

注意：asan_set_redzone 在 malloc_elem_alloc（内存分割）之后调用，此时 elem 已经精确指向用户分配的块，user_size 是未经对齐的原始请求大小。

#### 清除时机

|调用路径|代码位置|触发时机|
|---|---|---|
|rte_free → malloc_heap_free|malloc_heap.c:874|释放操作最开始，在 spinlock 之前|

调用形式：asan_clear_redzone(elem)

为什么先清除红区：释放时，内存会被合并到空闲链表、可能被 unmap，这些操作都需要访问 elem 头尾区域。如果红区还标记着 0xfa，ASAN 会对这些内部操作误报。

#### 检测时机（ASAN 运行时）

ASAN 编译器插桩在每次内存访问前自动生成检查代码，伪码如下：

```
// 编译器在每条 store/load 前插入：
shadow = (addr >> 3) + ASAN_SHADOW_OFFSET;
if (*shadow != 0) {
    // 该粒度有限制
    if (*shadow == 0xfa)
        __asan_report_heap_buffer_overflow(addr);
    // 或：if (addr & 7) >= *shadow → 超出了可用前N字节
    if ((addr & 0x7) >= *shadow)
        __asan_report_heap_buffer_overflow(addr);
}
```

即：每次访问地址 addr 前，ASAN 查其影子值，若 != 0 则进一步判断是否真的越界。

#### 范例
```c
// 错误代码：申请 9 字节但访问第 10 字节
char *p = rte_zmalloc(NULL, 9, 0);
p[9] = 'a';  // 越界写入
```


### heap-use-after-free（释放后使用检测）

#### 核心机制：Free Zone

释放内存时将整个数据区的影子标记为 0xfd，后续任何访问都会报 use-after-free。

#### 设置时机

|调用路径|代码位置|触发时机|
|---|---|---|
|rte_free → malloc_heap_free|malloc_heap.c:1042|释放完成后（free_unlock 标签处）|

```
// malloc_heap.c:1041~1042
free_unlock:
    asan_set_freezone(asan_ptr, asan_data_len);
    // asan_ptr = elem + HEADER_LEN + pad（用户数据起始）
    // asan_data_len = elem->size - OVERHEAD - pad（用户数据长度）
```

时序注意：asan_set_freezone 在 malloc_elem_free（真正合并到空闲链表）之后、rte_spinlock_unlock 之前调用。此时内存已经合并但锁还在持有，保证原子性。

#### 清除时机（重分配该内存块时）

当空闲块再次被分配时，通过 asan_clear_alloczone 清除 0xfd 标记：

|调用路径|代码位置|触发时机|
|---|---|---|
|malloc_elem_alloc（无分割，仅 pad）|malloc_elem.c:458|分配后|
|malloc_elem_alloc（有分割，新 elem）|malloc_elem.c:482|分割后|
|malloc_elem_hide_region（hide 区域）|malloc_elem.c:655|隐藏操作|
|malloc_elem_resize（缩小 size）|malloc_elem.c:671|resize 完成|
|malloc_elem_resize（扩大后再分割）|malloc_elem.c:699|resize 分割完|

#### 检测时机

同 heap-buffer-overflow，编译器插桩在每次内存访问前检查：

```c
if (*shadow == 0xfd)
    __asan_report_heap_use_after_free(addr);
```

#### 范例
```c
// 错误代码：释放后继续访问
char *p = rte_zmalloc(NULL, 9, 0);
rte_free(p);
*p = 'a';  // 释放后写入
```

### 重复释放(double free)



## 在DPDK中启用ASan工具检测内存

Meson 方式编译DPDK代码的时候，添加“-Db_sanitize=address”编译选项便可开启ASan工具。
由于ASan集成到CLANG编译器时，要求链接的代码允许加上`undefined symbols`，因此CLANG要求加上“-Db_lundef=false”选项, 如果采用GCC编译方式可以不加这个选项。
`-Dbuildtype=debug`选项可以帮助输出更多的日志信息帮助定位调试，建议加上这个选项。 
总而言之，meson编译DPDK代码时，加上`-Dbuildtype=debug -Db_lundef=false -Db_sanitize=address`编译选项，便可启用`ASan`工具。

```bash
- gcc: meson setup -Db_sanitize=address <build_dir>
- clang: meson setup -Db_sanitize=address -Db_lundef=false <build_dir>

- The libasan package must be installed when compiling with gcc in Centos/RHEL.
```

## ASAN的性能影响

|指标|典型值|说明|
|---|---|---|
|CPU 开销|~2x 慢|每次内存访问前额外检查影子内存|
|内存开销|~3x 多|影子内存(1/8) + redzone + trailer(64B/块) + asan_cookie(24B/块)|
|TRAILER_LEN|64B (ASAN) vs 0B (普通)|每个分配块额外 64B|
|结构体开销|+24B/块|user_size(8B) + `asan_cookie[2](16B)`|

在 DPDK 高性能转发场景中，大量小包频繁分配/释放时（如 mbuf），这些开销会导致吞吐量大幅下降，不适合在性能测试中使用。

## DPDK ASAN的不足
### 只覆盖 rte_malloc 系列，不覆盖 mempool/memzone

|分配接口|ASAN 保护|
|---|---|
|rte_malloc / rte_zmalloc / rte_calloc|✅ 完整保护|
|rte_realloc|✅ 完整保护|
|rte_memzone_reserve|❌ 无保护|
|rte_mempool_create + rte_mempool_get|❌ 无保护|
|rte_ring 内部对象|❌ 无保护|
|直接操作 hugepage（mmap）|❌ 无保护|

实际上，DPDK 高性能场景的核心路径（mempool 对象分配）根本没有 ASAN 保护，这是最大的盲区。

### 缺少 memory-leak 检测

DPDK ASAN 没有集成 LSan（LeakSanitizer）。标准 ASAN 可以在进程退出时报告未释放的内存，但 DPDK 的 hugepage 内存在 rte_eal_cleanup 之前不会释放，容易产生大量误报，且目前没有针对性集成。


# 参考
```bash
# 工欲善其事必先利其器——AddressSanitizer
https://zhuanlan.zhihu.com/p/382994002

```
