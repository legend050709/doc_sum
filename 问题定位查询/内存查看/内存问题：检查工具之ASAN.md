```table-of-contents
```

# ASAN概述

首先，先介绍一下 [Sanitizer](https://github.com/google/sanitizers)项目，该项目是谷歌出品的一个开源项目，该项目包含了 `ASAN`、`LSAN`、`MSAN`、`TSAN`等内存、线程错误的检测工具，这里简单介绍一下这几个工具的作用：

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


### 内存写穿(buffer-overflow)
### UAF(use-after-free)

### stack-use-after-scope
```c
//clang -O -g -fsanitize=address hollk.cpp -o hollk_cpp
volatile int *p = 0;

int main() {
  {
    int hollk = 0;
    p = &hollk;
  }
  *p = 5;
  return 0;
}
```
### stack-use-after-return
```c
//clang -O -g -fsanitize=address hollk.cpp -o hollk_cpp
//RUN:ASAN_OPTIONS=detect_stack_use_after_return=1 ./hollk_cpp
int *ptr;
__attribute__((noinline))
void FunctionA() {
  int hollk[100];
  ptr = &hollk[0];
}

int main(int argc, char **argv) {
  FunctionA();
  return ptr[argc];
}
```

## 原理

### 红区 (Red Zone) 和  影子内存 (Shadow Memory)

大多数的内存检测工具采用以下步骤进行检测，比如通过在被保护的堆，栈、全局变量周围建立标记为中毒状态的红区（redzones），如下图1所示，在绿色区域表示可访问的进程内存，红色区域表示在它周边插入的红区。

![](attachments/Pasted%20image%2020260503084733.png)

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
1. 8 字节的数据可读写，则 shadow memory 的值为 0
2. 8 字节的数据不可读写，则 shadow memory 的值为**负数**，如 `0xfa` 表示堆左边的 redzone、`0xf1` 表示栈左边的 redzone. ASan 也根据这个值在报错的时候输出对应的错误类型，如区分 `heap-buffer-underflow`/`stack-buffer-underflow`
3. 前 k 个字节可读写，后 8 - k 个字节不可读写，则 shadow memory 的值为 k，k 的取值范围为 `[1, 7]`
```


这种映射机制可以构建高效的查找方式，通过原始指针的值除以8，再添加offset，也就是在内存影子的位置添加。计算公式是 
```bash
ShadowAddr = (Addr >> 3) + offset
```


### 编译器插桩
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
1. 当访问 `a[0]`、 `a[1]` 的时候，发现对应的 shadow memory 为 0x00, 判定为安全
2. 当访问 `a[2]`、 `a[3]` 的时候，发现对应的 shadow memory 为 0x00, 判定为安全
3. 当访问 `a[4]`、 `a[5]` 的时候，发现对应的 shadow memory 为 0x00, 判定为安全
4. 当访问 `a[6]` 的时候，发现对应的 shadow memory 不为 0x00, 那么访问这块内存可能是不安全的，需要更详细的判断才能确定访问是否安全
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

5. 打开了调试标志`-g`，这是因为当发现内存错误时调试符号可以帮助错误报告更准确的告知错误发生位置的堆栈信息，如果错误报告中的堆栈信息看起来不太正确，请尝试使用`-fno-omit-frame-pointer`来改善堆栈信息的生成情况。
6. 如果构建代码时，**编译**和**链接**阶段分开执行，则必须在编译和链接阶段都添加`-fsanitize=address`选项。

# DPDK ASAN

## 背景
### DPDK的内存管理默认不支持ASAN
Glibc是linux 系统中最底层的API，ASan是基于glibc 实现，但向系统申请的内存DPDK和glibc采用不同的管理方式，对外接口完全不一样。所以动态运行库无法挂住DPDK的内存申请接口，就无法设置红区，并无法将其加入影子内存。

对内存是否可访问是编译器在插桩阶段完成，DPDK常用的编译器GCC 和CLANG 默认可以支持ASan，只要编译时添加ASan编译选项即可。

### 现有DPDK内存检查的缺陷 
现有的DPDK已经有内存管理的机制，是通过检测可用内存前后区域是否被修改来判断是否越界，但是现有的DPDK实现**只有在rte_free的时候才做内存检测**，甚至发生内存问题的时候, 非常少且有效的调试信息输出来帮助定位问题，如果问题带有一定的随机性且很难重现，让人很崩溃。


## 作用
### 内存写穿(buffer overflow)
```c
// 错误代码：申请 9 字节但访问第 10 字节
char *p = rte_zmalloc(NULL, 9, 0);
p[9] = 'a';  // 越界写入
```
### UAF(user-after-free)
```c
// 错误代码：释放后继续访问
char *p = rte_zmalloc(NULL, 9, 0);
rte_free(p);
*p = 'a';  // 释放后写入
```

### 重复释放(double free)



# 参考
```bash
# 工欲善其事必先利其器——AddressSanitizer
https://zhuanlan.zhihu.com/p/382994002

```
