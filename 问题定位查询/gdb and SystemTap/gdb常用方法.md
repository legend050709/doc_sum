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

# 父子进程处理
## set follow-fork-mode child
### 背景
在调试程序时，如果目标程序调用了 `fork()` 函数，GDB 需要决定是在父进程还是子进程中继续调试。`set follow-fork-mode` 用来指定 GDB 在 `fork()` 发生后跟踪哪一方。

- **`parent`（默认值）**： GDB 在 `fork()` 调用后继续调试父进程。
- **`child`**： GDB 在 `fork()` 调用后停止调试父进程，转而调试子进程。

### 含义
- 如果设置为 `child`，GDB 会自动切换到子进程，并忽略父进程。
- 如果调试的目标程序会生成子进程（例如，通过 `fork()` 或类似函数），使用该选项可以深入调试子进程的行为。

### 使用场景
- 调试多进程程序时： 如果程序调用了 `fork()`，你想要调试子进程的逻辑，而不是父进程。
- 调试守护进程（Daemon）： 通常守护进程在启动时会通过 `fork()` 创建一个子进程作为真正的服务进程，使用 `set follow-fork-mode child` 可以直接跟踪子进程。



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


# 其他
## 非交互式设置与打印

**设置**
```bash

```

**打印**
```bash
gdb -p `pidof dpvs` --batch -ex 'set print pretty on'  -ex 'p /x dpvs_estats' -ex 'detach' -ex 'quit';

```


# 参考
```bash
# 调试多线程 & 查死锁的bug & gcore命令 & gdb对多线程的调试 & gcore & pstack & 调试常用命令

https://www.cnblogs.com/charlesblc/p/6256912.html


```