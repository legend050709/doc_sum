```table-of-contents
```
# 多线程处理

## 相关命令
多线程调试的基本命令。 

**info threads** 显示当前可调试的所有线程，每个线程会有一个GDB为其分配的ID，后面操作线程的时候会用到这个ID。 前面有*的是当前调试的线程。 

**thread ID** 切换当前调试的线程为指定ID的线程。 

**break thread_test.c:123 thread all** 在所有线程中相应的行上设置断点（watch也可以指定thread）

**thread apply ID1 ID2 command** 让一个或者多个线程执行GDB命令command。 

**thread apply all command** 让所有被调试线程执行GDB命令command。 

**set scheduler-locking off|on|step** 估计是实际使用过多线程调试的人都可以发现，在使用step或者continue命令调试当前被调试线程的时候，其他线程也是同时执行的，怎么只让被调试 程序执行呢？

（注：step是进入内部，next是外部过一下）

通过这个命令就可以实现这个需求。off 不锁定任何线程，也就是所有线程都执行，这是默认值。 on 只有当前被调试程序会执行。 step 在单步的时候，除了next过一个函数的情况(熟悉情况的人可能知道，这其实是一个设置断点然后continue的行为)以外，只有当前线程会执行。

# gdb与进程
## gdb 启动程序
### 带有参数的二进制
```bash
gdb --args <可执行文件> <参数1> <参数2> ...
```

### 运行
```bash

```


## 父子进程处理
### set follow-fork-mode child
#### 问题
在调试程序时，如果目标程序调用了 `fork()` 函数。
但是在gdb中启动程序，然后设置断点，发现 一直不触发中断。

#### 分析
很多“断点不生效”，其实是**断点打在 A 进程，但是 你在看 B 进程跑**

在调试程序时，如果目标程序调用了 `fork()` 函数，GDB 需要决定是在父进程还是子进程中继续调试。
`set follow-fork-mode` 用来指定 GDB 在 `fork()` 发生后跟踪哪一方。

- **`parent`（默认值）**： GDB 在 `fork()` 调用后继续调试父进程。
- **`child`**： GDB 在 `fork()` 调用后停止调试父进程，转而调试子进程。


#### 含义
- 如果设置为 `child`，GDB 会自动切换到子进程，并忽略父进程。
- 如果调试的目标程序会生成子进程（例如，通过 `fork()` 或类似函数），使用该选项可以深入调试子进程的行为。

#### 使用场景
- 调试多进程程序时： 如果程序调用了 `fork()`，你想要调试子进程的逻辑，而不是父进程。
- 调试守护进程（Daemon）： 通常守护进程在启动时会通过 `fork()` 创建一个子进程作为真正的服务进程，使用 `set follow-fork-mode child` 可以直接跟踪子进程。

### detach-on-fork
```bash
set detach-on-fork off: 
	父子都留在 gdb 里
	
set detach-on-fork on: 默认值
	gdb 只管一个，另一个放飞
```

### 查看
```bash
//1 查看一下进程和线程的信息
info inferiors; 
	在GDB 语境下，`inferiors` 指的是：被 GDB 控制和调试的进程实例；
	也可以更通俗地理解为：GDB 正在调试的各个进程 / fork 之后产生的父/子进程对象

info threads;

//2 或者切换进程和线程的信息
inferior <infer number>;
thread <thread number>;

//3 查看对应的模式
show follow-fork-mode
show detach-on-fork

```

### 推荐使用
```bash
set follow-fork-mode child
set detach-on-fork off
```

![](attachments/Pasted%20image%2020260210110653.png)

# gdb查看
## 源码查看
### 源码路径查看
```bash
`info sources`：查看编译二进制时所有源码文件路径

`list <func>`：查看函数源码

`directory <path>`：添加源码搜索路径
```
### 查看函数源码
```bash
list FUNC
```

#### 函数显示过短问题
gdb 默认只显示函数前几行源码，如果函数很长，默认一次只显示 10 行，很难快速看到整个函数的全部热点区域。

**解决**：

（1）多次enter：
默认 `list` 会显示 10 行，你可以连续按 **Enter** 来显示下一段 10 行

（2）设置 listsize
可以通过修改行数参数：
```bash
set listsize 50 
list kucl_epoll_wait
```
`sets listsize` 改变默认显示行数（比如一次显示 50 行）， 后续的 `list` 命令都会按照这个数量显示

（3）gdb 没有直接命令一次性打印整个函数，但可以结合 `start` 和 `end` 行, 显示整个函数
```bash
(1) 得到函数的开始，结束位置：
(gdb) info line kucl_epoll_wait
No line number information available for address 0x42f3aa <kucl_epoll_wait>
Line 330 of "poll/epoll.c" starts at address 0x69e2b0 <kucl_epoll_wait> and ends at 0x69e2d3 <kucl_epoll_wait+35>.

（2）计算行数
>>> 0x69e2d3 - 0x69e2b0
35

（3）展示：list N,M   # 用结束行号
比如：
```

![](attachments/Pasted%20image%2020250918123402.png)

## 查看出现coredump的程序的map映射

如果程序正在运行，则可以通过`/proc/PID/maps` 来查看当前的进程的内存的映射信息。如果程序出现了coredump，也可以通过 gdb 中的 `info proc mappings` 来查看到 挂掉的程序的 maps信息。 如下所示：

```bash
(gdb) info proc mappings
Mapped address spaces:

          Start Addr           End Addr       Size     Offset objfile
......
        0x2200200000       0x2200400000   0x200000        0x0 /anon_hugepage (deleted)
        0x2200400000       0x2200600000   0x200000        0x0 /anon_hugepage (deleted)
        0x2200600000       0x2200800000   0x200000        0x0 /anon_hugepage (deleted)
        0x2200800000       0x2200a00000   0x200000        0x0 /anon_hugepage (deleted)
        0x2200a00000       0x2200c00000   0x200000        0x0 /anon_hugepage (deleted)
        0x2200c00000       0x2200e00000   0x200000        0x0 /anon_hugepage (deleted)
        0x2200e00000       0x2201000000   0x200000        0x0 /anon_hugepage (deleted)
        0x2201000000       0x2201200000   0x200000        0x0 /anon_hugepage (deleted)
        0x2201200000       0x2201400000   0x200000        0x0 /anon_hugepage (deleted)
        0x2201400000       0x2201600000   0x200000        0x0 /anon_hugepage (deleted)
        0x2201600000       0x2201800000   0x200000        0x0 /anon_hugepage (deleted)
        0x2201800000       0x2201a00000   0x200000        0x0 /anon_hugepage (deleted)
        0x2201a00000       0x2201c00000   0x200000        0x0 /anon_hugepage (deleted)
......
......
        0x3221800000       0x3221a00000   0x200000        0x0 /anon_hugepage (deleted)
        0x3221a00000       0x3221c00000   0x200000        0x0 /anon_hugepage (deleted)
        0x3221c00000       0x3221e00000   0x200000        0x0 /anon_hugepage (deleted)
        0x3221e00000       0x3222000000   0x200000        0x0 /anon_hugepage (deleted)
        0x3222000000       0x3222200000   0x200000        0x0 /anon_hugepage (deleted)
        0x3222200000       0x3222400000   0x200000        0x0 /anon_hugepage (deleted)
        0x3222400000       0x3222600000   0x200000        0x0 /anon_hugepage (deleted)
        0x3222600000       0x3222800000   0x200000        0x0 /anon_hugepage (deleted)
        0x3222800000       0x3222a00000   0x200000        0x0 /anon_hugepage (deleted)
      0x200000200000     0x200000400000   0x200000        0x0 /dev/hugepages/spdk_pid62656map_0 (deleted)
      0x200003a00000     0x200003c00000   0x200000        0x0 /dev/hugepages/spdk_pid62656map_28 (deleted)
      0x200003c00000     0x200003e00000   0x200000        0x0 /dev/hugepages/spdk_pid62656map_29 (deleted)
      0x200014c00000     0x200014e00000   0x200000        0x0 /dev/hugepages/spdk_pid62656map_165 (deleted)
      0x7fc18069e000     0x7fc18879e000  0x8100000    0x25000 /dev/shm/spdkchunkstore_perf_trace.pid62656 (deleted)
      0x7fc18c3a4000     0x7fc18c404000    0x60000        0x0 /usr/lib64/libpcre.so.1.2.0
      0x7fc18c404000     0x7fc18c604000   0x200000    0x60000 /usr/lib64/libpcre.so.1.2.0
      0x7fc18c604000     0x7fc18c605000     0x1000    0x60000 /usr/lib64/libpcre.so.1.2.0
      0x7fc18c605000     0x7fc18c606000     0x1000    0x61000 /usr/lib64/libpcre.so.1.2.0
      0x7fc18c606000     0x7fc18c62a000    0x24000        0x0 /usr/lib64/libselinux.so.1
      0x7fc18c62a000     0x7fc18c829000   0x1ff000    0x24000 /usr/lib64/libselinux.so.1
      0x7fc18c829000     0x7fc18c82a000     0x1000    0x23000 /usr/lib64/libselinux.so.1
      0x7fc18c82a000     0x7fc18c82b000     0x1000    0x24000 /usr/lib64/libselinux.so.1
      0x7fc18c82d000     0x7fc18c843000    0x16000        0x0 /usr/lib64/libresolv-2.17.so
      0x7fc18c843000     0x7fc18ca43000   0x200000    0x16000 /usr/lib64/libresolv-2.17.so
      0x7fc18ca43000     0x7fc18ca44000     0x1000    0x16000 /usr/lib64/libresolv-2.17.so
      0x7fc18ca44000     0x7fc18ca45000     0x1000    0x17000 /usr/lib64/libresolv-2.17.so
      0x7fc18ca47000     0x7fc18ca4a000     0x3000        0x0 /usr/lib64/libkeyutils.so.1.5
      0x7fc18ca4a000     0x7fc18cc49000   0x1ff000     0x3000 /usr/lib64/libkeyutils.so.1.5
      0x7fc18cc49000     0x7fc18cc4a000     0x1000     0x2000 /usr/lib64/libkeyutils.so.1.5
      0x7fc18cc4a000     0x7fc18cc4b000     0x1000     0x3000 /usr/lib64/libkeyutils.so.1.5
      0x7fc18cc4b000     0x7fc18cc59000     0xe000        0x0 /usr/lib64/libkrb5support.so.0.1
      0x7fc18cc59000     0x7fc18ce59000   0x200000     0xe000 /usr/lib64/libkrb5support.so.0.1
      0x7fc18ce59000     0x7fc18ce5a000     0x1000     0xe000 /usr/lib64/libkrb5support.so.0.1
      0x7fc18ce5a000     0x7fc18ce5b000     0x1000     0xf000 /usr/lib64/libkrb5support.so.0.1
      0x7fc18ce5b000     0x7fc18ce8c000    0x31000        0x0 /usr/lib64/libk5crypto.so.3.1
      0x7fc18ce8c000     0x7fc18d08b000   0x1ff000    0x31000 /usr/lib64/libk5crypto.so.3.1
      0x7fc18d08b000     0x7fc18d08d000     0x2000    0x30000 /usr/lib64/libk5crypto.so.3.1
      0x7fc18d08d000     0x7fc18d08e000     0x1000    0x32000 /usr/lib64/libk5crypto.so.3.1
      0x7fc18d08e000     0x7fc18d091000     0x3000        0x0 /usr/lib64/libcom_err.so.2.1
      0x7fc18d091000     0x7fc18d290000   0x1ff000     0x3000 /usr/lib64/libcom_err.so.2.1
      0x7fc18d290000     0x7fc18d291000     0x1000     0x2000 /usr/lib64/libcom_err.so.2.1
      0x7fc18d291000     0x7fc18d292000     0x1000     0x3000 /usr/lib64/libcom_err.so.2.1
      0x7fc18d292000     0x7fc18d36b000    0xd9000        0x0 /usr/lib64/libkrb5.so.3.3
      0x7fc18d36b000     0x7fc18d56a000   0x1ff000    0xd9000 /usr/lib64/libkrb5.so.3.3
      0x7fc18d56a000     0x7fc18d578000     0xe000    0xd8000 /usr/lib64/libkrb5.so.3.3
      0x7fc18d578000     0x7fc18d57b000     0x3000    0xe6000 /usr/lib64/libkrb5.so.3.3
      0x7fc18d57b000     0x7fc18d5c5000    0x4a000        0x0 /usr/lib64/libgssapi_krb5.so.2.2
      0x7fc18d5c5000     0x7fc18d7c5000   0x200000    0x4a000 /usr/lib64/libgssapi_krb5.so.2.2
      0x7fc18d7c5000     0x7fc18d7c6000     0x1000    0x4a000 /usr/lib64/libgssapi_krb5.so.2.2
      0x7fc18d7c6000     0x7fc18d7c8000     0x2000    0x4b000 /usr/lib64/libgssapi_krb5.so.2.2
      0x7fc18d7c8000     0x7fc18d828000    0x60000        0x0 /usr/lib64/libmlx5.so.1.19.35.0
      0x7fc18d828000     0x7fc18da28000   0x200000    0x60000 /usr/lib64/libmlx5.so.1.19.35.0
      0x7fc18da28000     0x7fc18da29000     0x1000    0x60000 /usr/lib64/libmlx5.so.1.19.35.0
      0x7fc18da29000     0x7fc18da2a000     0x1000    0x61000 /usr/lib64/libmlx5.so.1.19.35.0
      0x7fc18da2c000     0x7fc18da49000    0x1d000        0x0 /usr/lib64/libibverbs.so.1.14.35.0
      0x7fc18da49000     0x7fc18dc49000   0x200000    0x1d000 /usr/lib64/libibverbs.so.1.14.35.0
      0x7fc18dc49000     0x7fc18dc4a000     0x1000    0x1d000 /usr/lib64/libibverbs.so.1.14.35.0
      0x7fc18dc4a000     0x7fc18dc4b000     0x1000    0x1e000 /usr/lib64/libibverbs.so.1.14.35.0
      0x7fc18dc4b000     0x7fc18de03000   0x1b8000        0x0 /usr/lib64/libc-2.17.so
      0x7fc18de03000     0x7fc18e003000   0x200000   0x1b8000 /usr/lib64/libc-2.17.so
      0x7fc18e003000     0x7fc18e007000     0x4000   0x1b8000 /usr/lib64/libc-2.17.so
      0x7fc18e007000     0x7fc18e009000     0x2000   0x1bc000 /usr/lib64/libc-2.17.so
      0x7fc18e00e000     0x7fc18e023000    0x15000        0x0 /usr/lib64/libgcc_s-4.8.5-20150702.so.1
      0x7fc18e023000     0x7fc18e222000   0x1ff000    0x15000 /usr/lib64/libgcc_s-4.8.5-20150702.so.1
      0x7fc18e222000     0x7fc18e223000     0x1000    0x14000 /usr/lib64/libgcc_s-4.8.5-20150702.so.1
      0x7fc18e223000     0x7fc18e224000     0x1000    0x15000 /usr/lib64/libgcc_s-4.8.5-20150702.so.1
      0x7fc18e224000     0x7fc18e30d000    0xe9000        0x0 /usr/lib64/libstdc++.so.6.0.19
      0x7fc18e30d000     0x7fc18e50c000   0x1ff000    0xe9000 /usr/lib64/libstdc++.so.6.0.19
      0x7fc18e50c000     0x7fc18e514000     0x8000    0xe8000 /usr/lib64/libstdc++.so.6.0.19
      0x7fc18e514000     0x7fc18e516000     0x2000    0xf0000 /usr/lib64/libstdc++.so.6.0.19
      0x7fc18e52b000     0x7fc18e52f000     0x4000        0x0 /usr/lib64/libuuid.so.1.3.0
      0x7fc18e52f000     0x7fc18e72e000   0x1ff000     0x4000 /usr/lib64/libuuid.so.1.3.0
      0x7fc18e72e000     0x7fc18e72f000     0x1000     0x3000 /usr/lib64/libuuid.so.1.3.0
      0x7fc18e72f000     0x7fc18e730000     0x1000     0x4000 /usr/lib64/libuuid.so.1.3.0
      0x7fc18e730000     0x7fc18e737000     0x7000        0x0 /usr/lib64/librt-2.17.so
      0x7fc18e737000     0x7fc18e936000   0x1ff000     0x7000 /usr/lib64/librt-2.17.so
      0x7fc18e936000     0x7fc18e937000     0x1000     0x6000 /usr/lib64/librt-2.17.so
      0x7fc18e937000     0x7fc18e938000     0x1000     0x7000 /usr/lib64/librt-2.17.so
      0x7fc18e938000     0x7fc18e99f000    0x67000        0x0 /usr/lib64/libssl.so.1.0.2k
      0x7fc18e99f000     0x7fc18eb9f000   0x200000    0x67000 /usr/lib64/libssl.so.1.0.2k
      0x7fc18eb9f000     0x7fc18eba3000     0x4000    0x67000 /usr/lib64/libssl.so.1.0.2k
      0x7fc18eba3000     0x7fc18ebaa000     0x7000    0x6b000 /usr/lib64/libssl.so.1.0.2k
      0x7fc18ebaa000     0x7fc18ebbf000    0x15000        0x0 /usr/lib64/libz.so.1.2.7
      0x7fc18ebbf000     0x7fc18edbe000   0x1ff000    0x15000 /usr/lib64/libz.so.1.2.7
      0x7fc18edbe000     0x7fc18edbf000     0x1000    0x14000 /usr/lib64/libz.so.1.2.7
      0x7fc18edbf000     0x7fc18edc0000     0x1000    0x15000 /usr/lib64/libz.so.1.2.7
      0x7fc18edc0000     0x7fc18eff4000   0x234000        0x0 /usr/lib64/libcrypto.so.1.0.2k
      0x7fc18eff4000     0x7fc18f1f4000   0x200000   0x234000 /usr/lib64/libcrypto.so.1.0.2k
      0x7fc18f1f4000     0x7fc18f210000    0x1c000   0x234000 /usr/lib64/libcrypto.so.1.0.2k
      0x7fc18f210000     0x7fc18f21d000     0xd000   0x250000 /usr/lib64/libcrypto.so.1.0.2k
      0x7fc18f221000     0x7fc18f267000    0x46000        0x0 /usr/lib64/libtcmalloc.so.4.4.5
      0x7fc18f267000     0x7fc18f466000   0x1ff000    0x46000 /usr/lib64/libtcmalloc.so.4.4.5
      0x7fc18f466000     0x7fc18f468000     0x2000    0x45000 /usr/lib64/libtcmalloc.so.4.4.5
      0x7fc18f468000     0x7fc18f46a000     0x2000    0x47000 /usr/lib64/libtcmalloc.so.4.4.5
      0x7fc18f616000     0x7fc18f61b000     0x5000        0x0 /usr/lib64/libsnappy.so.1.1.4
      0x7fc18f61b000     0x7fc18f81a000   0x1ff000     0x5000 /usr/lib64/libsnappy.so.1.1.4
      0x7fc18f81a000     0x7fc18f81b000     0x1000     0x4000 /usr/lib64/libsnappy.so.1.1.4
      0x7fc18f81b000     0x7fc18f81c000     0x1000     0x5000 /usr/lib64/libsnappy.so.1.1.4
      0x7fc18f81c000     0x7fc18f91d000   0x101000        0x0 /usr/lib64/libm-2.17.so
      0x7fc18f91d000     0x7fc18fb1c000   0x1ff000   0x101000 /usr/lib64/libm-2.17.so
      0x7fc18fb1c000     0x7fc18fb1d000     0x1000   0x100000 /usr/lib64/libm-2.17.so
      0x7fc18fb1d000     0x7fc18fb1e000     0x1000   0x101000 /usr/lib64/libm-2.17.so
      0x7fc18fb1e000     0x7fc18fb28000     0xa000        0x0 /usr/lib64/libnuma.so.1.0.0
      0x7fc18fb28000     0x7fc18fd28000   0x200000     0xa000 /usr/lib64/libnuma.so.1.0.0
      0x7fc18fd28000     0x7fc18fd29000     0x1000     0xa000 /usr/lib64/libnuma.so.1.0.0
      0x7fc18fd29000     0x7fc18fd2a000     0x1000     0xb000 /usr/lib64/libnuma.so.1.0.0
      0x7fc18fd2a000     0x7fc18fd2c000     0x2000        0x0 /usr/lib64/libdl-2.17.so
      0x7fc18fd2c000     0x7fc18ff2c000   0x200000     0x2000 /usr/lib64/libdl-2.17.so
      0x7fc18ff2c000     0x7fc18ff2d000     0x1000     0x2000 /usr/lib64/libdl-2.17.so
      0x7fc18ff2d000     0x7fc18ff2e000     0x1000     0x3000 /usr/lib64/libdl-2.17.so
      0x7fc18ff2e000     0x7fc18ff6c000    0x3e000        0x0 /usr/lib64/libpcap.so.1.5.3
      0x7fc18ff6c000     0x7fc19016b000   0x1ff000    0x3e000 /usr/lib64/libpcap.so.1.5.3
      0x7fc19016b000     0x7fc19016d000     0x2000    0x3d000 /usr/lib64/libpcap.so.1.5.3
      0x7fc19016d000     0x7fc19016e000     0x1000    0x3f000 /usr/lib64/libpcap.so.1.5.3
      0x7fc19016f000     0x7fc1901d3000    0x64000        0x0 /usr/lib64/libnl-route-3.so.200.23.0
      0x7fc1901d3000     0x7fc1903d2000   0x1ff000    0x64000 /usr/lib64/libnl-route-3.so.200.23.0
      0x7fc1903d2000     0x7fc1903d5000     0x3000    0x63000 /usr/lib64/libnl-route-3.so.200.23.0
      0x7fc1903d5000     0x7fc1903da000     0x5000    0x66000 /usr/lib64/libnl-route-3.so.200.23.0
      0x7fc1903dc000     0x7fc1903fa000    0x1e000        0x0 /usr/lib64/libnl-3.so.200.23.0
      0x7fc1903fa000     0x7fc1905fa000   0x200000    0x1e000 /usr/lib64/libnl-3.so.200.23.0
      0x7fc1905fa000     0x7fc1905fc000     0x2000    0x1e000 /usr/lib64/libnl-3.so.200.23.0
      0x7fc1905fc000     0x7fc1905fd000     0x1000    0x20000 /usr/lib64/libnl-3.so.200.23.0
      0x7fc1905fd000     0x7fc190605000     0x8000        0x0 /usr/lib64/libprotobuf-c.so.1.0.0
      0x7fc190605000     0x7fc190804000   0x1ff000     0x8000 /usr/lib64/libprotobuf-c.so.1.0.0
      0x7fc190804000     0x7fc190805000     0x1000     0x7000 /usr/lib64/libprotobuf-c.so.1.0.0
      0x7fc190805000     0x7fc190806000     0x1000     0x8000 /usr/lib64/libprotobuf-c.so.1.0.0
      0x7fc190806000     0x7fc19081a000    0x14000        0x0 /usr/lib64/libkucl.so
      0x7fc19081a000     0x7fc190a4f000   0x235000    0x14000 /usr/lib64/libkucl.so
      0x7fc190a4f000     0x7fc190ad5000    0x86000   0x249000 /usr/lib64/libkucl.so
      0x7fc190ad5000     0x7fc190ad6000     0x1000   0x2cf000 /usr/lib64/libkucl.so
      0x7fc190ad6000     0x7fc190adc000     0x6000   0x2cf000 /usr/lib64/libkucl.so
      0x7fc190adc000     0x7fc190b5a000    0x7e000   0x2d5000 /usr/lib64/libkucl.so
      0x7fc19119a000     0x7fc19119b000     0x1000        0x0 /usr/lib64/libaio.so.1.0.1
      0x7fc19119b000     0x7fc19139a000   0x1ff000     0x1000 /usr/lib64/libaio.so.1.0.1
      0x7fc19139a000     0x7fc19139b000     0x1000        0x0 /usr/lib64/libaio.so.1.0.1
      0x7fc19139b000     0x7fc19139c000     0x1000     0x1000 /usr/lib64/libaio.so.1.0.1
      0x7fc19139c000     0x7fc1913b3000    0x17000        0x0 /usr/lib64/libpthread-2.17.so
      0x7fc1913b3000     0x7fc1915b2000   0x1ff000    0x17000 /usr/lib64/libpthread-2.17.so
      0x7fc1915b2000     0x7fc1915b3000     0x1000    0x16000 /usr/lib64/libpthread-2.17.so
      0x7fc1915b3000     0x7fc1915b4000     0x1000    0x17000 /usr/lib64/libpthread-2.17.so
      0x7fc1915b8000     0x7fc1915d9000    0x21000        0x0 /usr/lib64/ld-2.17.so
      0x7fc191766000     0x7fc191767000     0x1000   0x700000 /dev/infiniband/uverbs0
      0x7fc191767000     0x7fc191768000     0x1000   0x500000 /dev/infiniband/uverbs0
      0x7fc191768000     0x7fc191769000     0x1000     0x7000 /dev/infiniband/uverbs0
      0x7fc191769000     0x7fc19176a000     0x1000     0x6000 /dev/infiniband/uverbs0
      0x7fc19176a000     0x7fc19176b000     0x1000     0x5000 /dev/infiniband/uverbs0
      0x7fc19176b000     0x7fc19176c000     0x1000     0x4000 /dev/infiniband/uverbs0
      0x7fc19176c000     0x7fc19176d000     0x1000     0x3000 /dev/infiniband/uverbs0
      0x7fc19176d000     0x7fc19176e000     0x1000     0x2000 /dev/infiniband/uverbs0
      0x7fc19176e000     0x7fc19176f000     0x1000     0x1000 /dev/infiniband/uverbs0
      0x7fc19176f000     0x7fc191770000     0x1000        0x0 /dev/infiniband/uverbs0
      0x7fc191774000     0x7fc191775000     0x1000   0x600000 /dev/infiniband/uverbs0
      0x7fc1917ca000     0x7fc1917cb000     0x1000   0x700000 /dev/infiniband/uverbs0
      0x7fc1917cb000     0x7fc1917cc000     0x1000   0x500000 /dev/infiniband/uverbs0
      0x7fc1917cc000     0x7fc1917cd000     0x1000   0x307000 /dev/infiniband/uverbs0
      0x7fc1917cd000     0x7fc1917ce000     0x1000   0x306000 /dev/infiniband/uverbs0
      0x7fc1917ce000     0x7fc1917cf000     0x1000   0x305000 /dev/infiniband/uverbs0
      0x7fc1917cf000     0x7fc1917d0000     0x1000   0x304000 /dev/infiniband/uverbs0
      0x7fc1917d0000     0x7fc1917d1000     0x1000   0x303000 /dev/infiniband/uverbs0
      0x7fc1917d1000     0x7fc1917d2000     0x1000   0x302000 /dev/infiniband/uverbs0
      0x7fc1917d2000     0x7fc1917d3000     0x1000   0x301000 /dev/infiniband/uverbs0
      0x7fc1917d3000     0x7fc1917d4000     0x1000   0x300000 /dev/infiniband/uverbs0
      0x7fc1917d9000     0x7fc1917da000     0x1000    0x21000 /usr/lib64/ld-2.17.so
      0x7fc1917da000     0x7fc1917db000     0x1000    0x22000 /usr/lib64/ld-2.17.so
```

### 应用场景
DPDK程序的`double free`问题。
比如：如果将spdk的大页内存申请好了之后，然后通过外部内存的方式注册到`dpdk`中，通过外部的`heap`来进行管理。如果程序退出的时候，先清理`spdk`的内存空间，即通过`munmap`取消映射，如果在dpdk中还存在操作这块内存空间，就会出现`double free`的问题。

为了验证是否真的是存在double free, 可以通过上面的方法进行验证。
```bash

gdb中 info proc mappings；

看 fault 地址：
如果这个地址已经不在映射区间,则是 hugepage 被 munmap；
```


# gdb 输出

## gdb输出到文件

```bash
(gdb)set logging file <文件名>

(gdb)set logging on

(gdb)thread apply all bt

(gdb)set logging off

(gdb)quit

```

## set pretty print on
`set pretty-print on` 是 GDB 中的一个设置，用于启用更友好的格式化输出（pretty-print）。当启用时，GDB 会以更加易读的方式显示复杂数据结构（如数组、结构体、类和 STL 容器等）。

### 范例
```c
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};
    std::cout << "Debugging example!" << std::endl;
    return 0;
}
```

(1) 默认设置（pretty-print off）：
```bash
(gdb) print v
$1 = {<std::_Vector_base<int, std::allocator<int> > > = {<__gnu_cxx::new_allocator<int>> = {...}}, _M_impl = {...}}

```

(2) 启用 pretty-print（pretty-print on）：
```bash
(gdb) set pretty-print on
(gdb) print v
$1 = std::vector of length 5, capacity 5 = {1, 2, 3, 4, 5}
```

## set pagination off

GDB 默认会分页显示输出，也就是说，如果输出的内容较多，GDB 会暂停显示，并提示你按下回车键或空格键来查看更多内容。

- **`on`（默认值）**： 输出内容分页显示。
- **`off`**： 输出内容不分页，直接显示所有内容。

### 范例
假设你打印了一个大数组：
```bash
(gdb) print my_array
```
**分页开启（默认）：** GDB 会暂停输出，并提示类似以下内容：
```bash
--Type <RET> for more, q to quit, c to continue without paging--
```

**分页关闭：** GDB 会直接显示完整的数组内容，无需用户输入。

### 作用

设置`set pagination off`，禁用分页后，GDB 输出内容时不会暂停，可以快速查看大量信息。对于调试过程中频繁输出的大量信息（例如打印大量变量、内存、堆栈等），禁用分页可以避免频繁按键。

#### 使用场景
- 调试输出较多的程序时，避免中断和手动翻页。
- 在自动化调试脚本中使用 GDB，确保输出不会被分页影响。


# gdb中主动调用某个函数

## 理解
### gdb 主动调用某个函数，函数内存在局部变量，是否影响堆栈

在 GDB 中主动调用一个函数时，该函数的局部变量会影响堆栈，因为它们会在堆栈上分配空间。虽然这不会对程序的整体执行造成影响，但在调试过程中，也要注意堆栈溢出的风险。

#### 函数调用的基本原理

当你在 GDB 中使用 `call` 命令主动调用一个函数时，GDB 会像正常程序执行一样，在当前堆栈帧上执行该函数。这意味着：

- **局部变量的分配**：函数的局部变量会在堆栈上分配空间。即使你是在调试器中手动调用这个函数，这些局部变量仍然会被创建并占用堆栈空间。
- **堆栈帧的创建**：每次调用函数时，都会创建一个新的堆栈帧，包含该函数的参数和局部变量。

#### 调用函数的影响

当你在 GDB 中调用一个函数时，可能会遇到以下情况：
- **堆栈溢出**：如果你连续调用函数，尤其是递归函数，可能会导致堆栈溢出，因为每次调用都会消耗堆栈空间。

## 范例
```bash
# ps -ef |grep named
root      9817 17904  0 21:01 pts/0    00:00:00 grep --color=auto named
named    27245     1  4  2024 ?        1-05:33:44 /usr/sbin/named -u named -c /etc/named.conf -t /var/named/chroot

# ./gdb-11.2 -D share_gdb/ -p 27245 # 使用高版本的gdb，低版本的gdb可能存在问题。

(gdb) call malloc(16)
'malloc' has unknown return type; cast the call to its declared return type

(gdb) call (void*)malloc(16)
$1 = (void *) 0x55ec16cc7410

(gdb) set $dst = $0 #将上个结果的返回值赋值给dst

(gdb) set $src = "hello, gdb"

(gdb) call strncpy($dst, $src, 16)
'__strncpy_sse2_unaligned' has unknown return type; cast the call to its declared return type

(gdb) call (char*)strncpy($dst, $src, 16)
$3 = 0x55ec16cc7430 "hello, gdb"

(gdb) x/s $dst
0x55ec16cc7430: "hello, gdb"
```

## 应用
### 查看某个已经运行进程的某个socket的缓冲区的大小

#### 背景
比如对于`udp`的程序，`named`，其出现了 `snmp`的 `recvBuffError` 或者 `sendBuffErr` 报警。
此时，查看系统当前的 `net.core.rmem_max` 以及 `net.core.wmem_max`，但是程序启动时候的  `net.core.rmem_max` 以及 `net.core.wmem_max` 可能不是当前值，那么如何获取到程序启动时候的   `net.core.rmem_max` 以及 `net.core.wmem_max`的值呢。

#### 思路
##### 思路一：
```bash
ss -nump : 查看udp的socket
ss -ntmp : 查看tcp的socket
```

```bash
# ss -nump
Recv-Q Send-Q                                                      Local Address:Port                                                                     Peer Address:Port
0      0                                                           10.52.145.147:34953                                                                       223.5.5.5:53
	 skmem:(r0,rb16000000,t0,tb262144000,f0,w0,o0,bl0)
0      0                                                           10.52.145.147:58997                                                                       223.5.5.5:53
	 skmem:(r0,rb16000000,t0,tb262144000,f0,w0,o0,bl0)
```

rb16000000：表示这个socket的接受缓冲区的大小为：16000000B.
tb262144000: 表示这个socket的发送缓冲区的大小为：262144000B.

##### 思路二：
一个已经运行的进程，查看某个socket fd的 option信息。一般情况下，
`cat /proc/net/udp` 或者 `netstat -apn` 或者 `cat /proc/PID/net/udp` 或者
`ll /proc/PID/fd/`等等都是无法查看的。

也无法都是在外面编写一个进程，来查看，因为fd表是基于进程的，每个进程的fd表都是隔离的。

但是可以通过`gdb attach PID`的方式来查看某个已经运行进程的信息，然后在 `gdb`中 通过主动`call getsockopt`的方式，来查看某个`fd`的`option`信息。

#### 查看

**（1）`getsockopt` 函数原型**：
```c
   int getsockopt(int sockfd, int level, int optname,
				  void *optval, socklen_t *optlen);
```

**（2）进程的信息查看**：
```bash
# ps -ef |grep named
root     19037 20684  0 14:53 pts/0    00:00:00 grep --color=auto named
named    27245     1  4  2024 ?        1-06:24:03 /usr/sbin/named -u named -c /etc/named.conf -t /var/named/chroot


# netstat -apn |grep udp
udp        0      0 10.108.164.22:53        0.0.0.0:*                           27245/named
udp        0      0 127.0.0.1:53            0.0.0.0:*                           27245/named
udp        0      0 127.0.0.1:323           0.0.0.0:*                           993/chronyd
udp        0      0 0.0.0.0:33687           0.0.0.0:*                           991/rsyslogd
udp6       0      0 ::1:323                 :::*                                993/chronyd
udp6       0      0 :::6607                 :::*                                2434/java

# cat /proc/27245/net/udp | column -t
sl      local_address  rem_address    st  tx_queue           rx_queue     tr        tm->when  retrnsmt  uid        timeout  inode             ref  pointer  drops
26050:  16A46C0A:0035  00000000:0000  07  00000000:00000000  00:00000000  00000000  25        0         137470918  2        00000000ecba7c52  0
26050:  0100007F:0035  00000000:0000  07  00000000:00000000  00:00000000  00000000  25        0         137470915  2        00000000379db82c  0
26320:  0100007F:0143  00000000:0000  07  00000000:00000000  00:00000000  00000000  0         0         29857      2        0000000064f38ee9  0
26916:  00000000:8397  00000000:0000  07  00000000:00000000  00:00000000  00000000  0         0         17838      2        000000000425f297  0

# cat /proc/net/udp
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode ref pointer drops
26050: 16A46C0A:0035 00000000:0000 07 00000000:00000000 00:00000000 00000000    25        0 137470918 2 00000000ecba7c52 0
26050: 0100007F:0035 00000000:0000 07 00000000:00000000 00:00000000 00000000    25        0 137470915 2 00000000379db82c 0
26320: 0100007F:0143 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 29857 2 0000000064f38ee9 0
26916: 00000000:8397 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 17838 2 000000000425f297 0


# ls -l /proc/27245/fd
total 0
lrwx------ 1 named named 64 Dec 30 02:08 0 -> /dev/null
lrwx------ 1 named named 64 Dec 30 02:08 1 -> /dev/null
l-wx------ 1 named named 64 Dec 30 02:08 10 -> pipe:[137506306]
lrwx------ 1 named named 64 Dec 30 02:08 11 -> anon_inode:[eventpoll]
lr-x------ 1 named named 64 Dec 30 02:08 12 -> /var/named/chroot/dev/random
l-wx------ 1 named named 64 Dec 30 02:08 13 -> /var/named/chroot/var/named/logs/queries.log
l-wx------ 1 named named 64 Dec 30 02:08 14 -> /var/named/chroot/var/named/logs/zone_transfers.log
lrwx------ 1 named named 64 Dec 30 02:08 2 -> /dev/null
lrwx------ 1 named named 64 Dec 30 02:08 20 -> socket:[137470904]
lrwx------ 1 named named 64 Dec 30 02:08 21 -> socket:[137470905]
lrwx------ 1 named named 64 Dec 30 02:08 22 -> socket:[137470917]
lrwx------ 1 named named 64 Dec 30 02:08 23 -> socket:[137470919]
lrwx------ 1 named named 64 Dec 30 02:08 24 -> socket:[137470922]
lrwx------ 1 named named 64 Dec 30 02:08 3 -> socket:[137495073]
lr-x------ 1 named named 64 Dec 30 02:08 4 -> /var/lib/sss/mc/initgroups (deleted)
lrwx------ 1 named named 64 Dec 30 02:08 5 -> socket:[137495078]
lrwx------ 1 named named 64 Dec 30 02:08 512 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 513 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 514 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 515 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 516 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 517 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 518 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 519 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 520 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 521 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 522 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 523 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 524 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 525 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 526 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 527 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 528 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 529 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 530 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 531 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 532 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 533 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 534 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 535 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 536 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 537 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 538 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 539 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 540 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 541 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 542 -> socket:[137470915]
lrwx------ 1 named named 64 Dec 30 02:08 543 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 544 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 545 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 546 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 547 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 548 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 549 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 550 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 551 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 552 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 553 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 554 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 555 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 556 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 557 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 558 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 559 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 560 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 561 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 562 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 563 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 564 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 565 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 566 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 567 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 568 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 569 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 570 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 571 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 572 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 573 -> socket:[137470918]
lrwx------ 1 named named 64 Dec 30 02:08 6 -> /dev/null
l-wx------ 1 named named 64 Dec 30 02:08 7 -> /var/named/chroot/var/named/logs/default.log
lr-x------ 1 named named 64 Dec 30 02:08 8 -> pipe:[137506306]
l-wx------ 1 named named 64 Dec 30 02:08 9 -> /var/named/chroot/var/named/logs/debug.log
```

**（3）udp listen socket的 fd查看**：
![](attachments/Pasted%20image%2020250123151808.png)

如上所示，感觉通过 `cat /proc/27245/net/udp` 查看的 udp 的 inode，然后再到 `ll /proc/PID/fd` 下基于 inode 查找 fd ，感觉不太准确。
直接通过 `ss -apnie  | grep -i named` 来查看 listen socket的 fd 以及 inode 更加准确。

**（4）gdb 调用 getsockopt查看fd的信息**：
```bash
# ./gdb-11.2 -D share_gdb/ -p 27245
GNU gdb (GDB) 11.2
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
0x00007fcbec5cc572 in sigsuspend () from /lib64/libc.so.6
(gdb) set $optval = malloc(4)
'malloc' has unknown return type; cast the call to its declared return type
(gdb) set $optval = (void*)malloc(4)
(gdb) set $optlen = (void*)malloc(4)
(gdb) set *((int *)$optlen) = 4
(gdb) set $sockfd = 23
(gdb) set $level = 1
(gdb) set $optname = 8
(gdb) call getsockopt($sockfd, $level, $optname, $optval, $optlen)
'getsockopt' has unknown return type; cast the call to its declared return type
(gdb) (int)call getsockopt($sockfd, $level, $optname, $optval, $optlen)
Undefined command: "".  Try "help".
(gdb) call (int)getsockopt($sockfd, $level, $optname, $optval, $optlen)
$1 = 0
(gdb) print *((int *)$optval)
$2 = 87380
(gdb) call (void)free($optval)
(gdb) call (void)free($optlen)
(gdb) quit
```

### 查看大量信息或更优雅的查看
`GDB`中可以通过 `print` 来查看信息，但是`Print`一次只能查看一个变量，并且`print`查看的都是每个字节的数据，如果期望以字符串形式查看。

如果是一片连续内存，有些字段需要查看，有些字段不想查看，那么用 `Print` 就不太方便。

#### 查看Mbuf的二三四层头信息
查看某个`Mbuf`的内容，比如，二层头，三层头，四层头信息。那么 `Print` 就很不方便。

那么，就可以通过实现一个函数，传入`mbuf`的地址，然后打印对应的头的信息。在`GDB`中主动调用该函数，就比较方便了。


# 信号 signal 

## 信号 Signals

信号是一种软中断，应用程序一般都会处理信号，如==程序异常退出等会发出信号==。  
UNIX下的部分信号：

![](attachments/Pasted%20image%2020240527153041.png)


|SIGINT|表示中断字符信号，也就是Ctrl+C的信号|
|---|---|
|SIGBUS|表示硬件故障的信号|
|SIGCHLD|表示子进程状态改变信号|
|SIGKILL|表示终止程序运行的信号|


## GDB中处理信号

GDB调试器可以自动捕获C、C++程序中出现的信号，并根据事先约定好的方式处理它，默认收到任何信号都会停住正在运行的程序，以供你进行调试。

![](attachments/Pasted%20image%2020240527154603.png)

### 发送信号

给程序发信号有2种方法
```bash

shell终端：kill [ -s signame | -n signum ] pid  
gdb终端：signal [ signame | signum ]
```

使用`signal`命令和在shell环境使用`kill`命令给程序发送信号的区别在于：
在shell环境使用`kill`命令给程序发送信号，gdb会根据当前的设置决定是否把信号发送给进程，而使用`signal`命令则直接把信号发给进程。


注：其中 `signal 0` 使用：如果当前的程序由于某个信号而暂停，可以通过`signal 信号`命令让程序忽略该信号，继续运行。

![](attachments/Pasted%20image%2020240527161348.png)



### 控制GDB收到信号的处理方式

```bash

handle SIGNAL [ACTIONS]  ：配置收到指定信号的处理方式

如果没有指定 action，表示忽略对应的信号的处理。
```

使用handle SIG32 noprint nostop忽略系统信号，让GDB只停在我们自己的断点处。

keywords 可以是以下几种关键字的一个或多个：


```bash
actions 可以是如下 的动作：
"stop", "nostop", "print", "noprint",
"pass", "nopass", "ignore", or "noignore".

（1）Stop means reenter debugger if this signal happens (implies print).
（2）Print means print a message if this signal happens.
（3）Pass means let program see this signal; otherwise program doesn't know.
（4）Ignore is a synonym for nopass and noignore is a synonym for pass.
Pass and Stop may be combined.

```


- **nostop**  
    当被调试的程序收到信号时，GDB不会停住程序的运行，但会打出消息告诉你收到这种信号。
- **stop**  
    当被调试的程序收到信号时，GDB会停住你的程序。
- **print**  
    当被调试的程序收到信号时，GDB会显示出一条信息。
- **noprint**  
    当被调试的程序收到信号时，GDB不会告诉你收到信号的信息。
- **pass（Pass to program）** 、**noignore**  
    当被调试的程序收到信号时，GDB不处理信号。这表示，GDB会把这个信号交给被调试程序会处理。
- **nopass**、**ignore**  
    当被调试的程序收到信号时，GDB不会让被调试程序来处理这个信号


#### linux c中控制信号的处理

```c
#include <stdio.h>
#include <signal.h>

void signal_handler(int signum)
{
    // 什么也不做
}

int main()
{
    // 设置信号处理程序为SIG_IGN
    signal(SIGINT, SIG_IGN);

    int i = 0;
    while (1) {
        printf("i = %d\n", i);
        i++;
        sleep(1);
    }

    return 0;
}
```

## 信号处理常用命令

|handle SIG32 noprint nostop|遇到SIG32不停止不打印，从而不影响GDB过程|
|---|---|
|info signals|查看GDB可以处理的信号种类，以及各个信号的具体处理方式|
|info handle|查看有哪些信号在被GDB检测中|


# 数据类型查看
## ptype

![](attachments/Pasted%20image%2020240527162414.png)

### 查看结构体以及结构体的成员类型


### 范例

在某些平台上（比如Linux）使用gdb调试程序，当有信号发生时，gdb在把信号丢给程序之前，可以通过`$_siginfo`变量读取一些额外的有关当前信号的信息，这些信息是`kernel`传给信号处理函数的。

```bash
Program received signal SIGHUP, Hangup.
0x00000034e42accc0 in __nanosleep_nocancel () from /lib64/libc.so.6
Missing separate debuginfos, use: debuginfo-install glibc-2.12-1.132.el6.x86_64
(gdb) ptype $_siginfo
type = struct {
    int si_signo;
    int si_errno;
    int si_code;
    union {
        int _pad[28];
        struct {...} _kill;
        struct {...} _timer;
        struct {...} _rt;
        struct {...} _sigchld;
        struct {...} _sigfault;
        struct {...} _sigpoll;
    } _sifields;
}
(gdb) ptype $_siginfo._sifields._sigfault
type = struct {
    void *si_addr;
}
(gdb) p $_siginfo._sifields._sigfault.si_addr
$1 = (void *) 0x850e
```



# 反汇编

# 寄存器
## 查看寄存器的值
### 查看所有寄存器的值
`info registers` 可以查看所有寄存器的值，如下所示：

```bash
(gdb) info registers
rax            0x450640            4523584
rbx            0x7ffd2afda9e0      140725324720608
rcx            0x0                 0
rdx            0x0                 0
rsi            0x80                128
rdi            0x0                 0
rbp            0x1401b3d10         0x1401b3d10
rsp            0x7ffd2afda7c0      0x7ffd2afda7c0
r8             0x0                 0
r9             0x7ffd2afda5f0      140725324719600
r10            0x0                 0
r11            0x7f2441dcfab0      139793700551344
r12            0xb                 11
r13            0x1401b3cd0         5370494160
r14            0x0                 0
r15            0xffffffff          4294967295
rip            0x41614a            0x41614a <rdma_fcntl+634>
eflags         0x10206             [ PF IF RF ]
cs             0x33                51
ss             0x2b                43
ds             0x0                 0
es             0x0                 0
fs             0x0                 0
gs             0x0                 0
k0             0x12012001          302063617
k1             0x1                 1
k2             0xfefe0000          4278059008
k3             0x0                 0
k4             0xfffffffb          4294967291
k5             0x0                 0
k6             0x0                 0
k7             0x0                 0
```

### 查看指定寄存器的值
`info registers [xxx] [yyy] ...` 可以查看寄存器 xxx, yyy的值。
```bash
(gdb) info registers rdi rsi
rdi            0x0                 0
rsi            0x80                128
(gdb) info registers rbp rbx
rbp            0x1401b3d10         0x1401b3d10
rbx            0x7ffd2afda9e0      14072532472060
```

或者 通过`p/x $xxx`的方式，查看某个寄存器的值。
```bash
(gdb) p/x $rdi
$5 = 0x0
(gdb) p/x $rsi
$6 = 0x80
(gdb) p/x $rbp
$7 = 0x1401b3d10
(gdb) p/x $rbx
$8 = 0x7ffd2afda9e0
```

## `optimized out`变量优化
### 范例
```c
(gdb) bt
#0  0x00007f2441dcfaca in ibv_create_cq () from /usr/lib64/libibverbs.so.1
#1  0x000000000041614a in rdma_create_comp_channel (cntl=<optimized out>, conn=0x1401b3d10)
    at rdma/rdma_cq.c:61
#2  rdma_comp_channel_ctl (cntl=0x7ffd2afda9e0, conn=0x1401b3d10) at rdma/rdma_cq.c:86
#3  rdma_fcntl (data=0x1401b3d10, type=<optimized out>, cntl=0x7ffd2afda9e0) at rdma/rdma.c:188
#4  0x0000000000412ea6 in kucl_epoll_create (size=100) at socket/poll.c:56
#5  main (argc=<optimized out>, argv=<optimized out>) at ibv_test_server.c:172

```
如上所示，f1中的 cntl 被优化了，其实根据`f2`可知道，其值为 `0x7ffd2afda9e0`，那么假装不知道，来进行分析。

实际的函数原型为：`int rdma_comp_channel_ctl(struct kucl_rdma_conn *conn, kucl_cntl_data *cntl);`
函数参数的位置和`bt`中的函数参数的顺序好像不太对应（应该是编译器的优化导致）。

#### 分析
在x86_64中，前六个整数或指针参数依次通过`rdi、rsi、rdx、rcx、r8、r9`传递。
函数可能在进入时已经将参数存到了其他寄存器或栈上，所以可能需要查看函数入口处的汇编代码，确认参数是否被移动到了其他位置。用户可以使用`disassemble`命令反汇编函数（重点查看，是否存在反汇编开头的 mov指令，比如`mov %rdi,%rax` 或`mov %rsi,%rbx`），看看`rdi`的值是否被保存到其他地方，比如rax或其他寄存器，然后查看对应的寄存器。

#### 查看
(1) 查看反汇编：
```bash
(1) disassemble 函数名
比如：
disassemble kucl_epoll_real_polling

(2) disassemble /m 函数名
比如：
disassemble /m kucl_epoll_real_polling
```
![](attachments/Pasted%20image%2020250422153714.png)

(2) 查看寄存器的值
```bash
(gdb) info registers rdi rsi
rdi            0x0                 0
rsi            0x80                128
(gdb) info registers rbp rbx
rbp            0x1401b3d10         0x1401b3d10
rbx            0x7ffd2afda9e0      140725324720608
(gdb) p *(kucl_cntl_data *)0x7ffd2afda9e0
$4 = {
  op = 1,
  epfd = 11,
  data = 0x1401b3cd0,
  events = 2147483653
}
```


# 其他
## 非交互式设置与打印

**设置**
```bash

```

**打印**
```bash
gdb -p `pidof dpvs` --batch -ex 'set print pretty on'  -ex 'p /x dpvs_estats' -ex 'detach' -ex 'quit';

```

# gdb常见问题
## 无法设置断点
程序未加 `-g` 编译，重新编译时添加 `-g`。

##  变量显示 `optimized out`

编译时优化级别过高（如 `-O3`），调试时用 `-O0` 或 `-O1`。
## 无法生成 core 文件
未开启 `ulimit -c unlimited`，或目录无写权限。

# 参考
```bash
# 调试多线程 & 查死锁的bug & gcore命令 & gdb对多线程的调试 & gcore & pstack & 调试常用命令
https://www.cnblogs.com/charlesblc/p/6256912.html

# Linux | 调试器GDB的详细教程【纯命令行调试】【+++++++全流程执行范例】
https://juejin.cn/post/7206723874506899512


```