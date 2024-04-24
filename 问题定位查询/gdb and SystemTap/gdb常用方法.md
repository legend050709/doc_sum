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




# gdb输出到文件

```bash
(gdb)set logging file <文件名>

(gdb)set logging on

(gdb)thread apply all bt

(gdb)set logging off

(gdb)quit

```

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