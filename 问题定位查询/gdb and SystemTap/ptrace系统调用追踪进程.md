```table-of-contents
```
# 介绍
`ptrace`，即`process trace`，这个系统调用是Linux程序调试工具`gdb`以及系统调用追踪器`strace`的基础。此外，`ptrace`系统调用也常被用在逆向工程里面。
`ptrace()`系统调用为一个进程提供了**观察**和**控制**另一个进程的执行过程的能力，同时也提供**检查**和**改变**另一个进程的内存值以及相关注册信息。其中，被控制的进程被称为`tracee`，控制进程被称为`tracer`。

## 函数
### 原型
```c
#include <sys/ptrace.h>       
long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);

request：要执行的操作类型；
pid：被追踪的目标进程ID；
addr：被监控的目标内存地址；
data：保存读取出或者要写入的数据。

request的取值：
#define PTRACE_TRACEME         0
	tracee表明自己想要被追踪，这会自动与父进程建立追踪关系，这也是唯一能被tracee使用的request，其他的request都由tracer指定。
#define PTRACE_PEEKTEXT        1
#define PTRACE_PEEKDATA        2
	PTRACE_PEEKDATA读取进程虚拟地址空间的任意数据。
#define PTRACE_PEEKUSR         3
	PTRACE_PEEKUSR读取单个寄存器。
#define PTRACE_POKETEXT        4
#define PTRACE_POKEDATA        5
	PTRACE_POKEDATA设置进程虚拟地址空间的任意数据。
#define PTRACE_POKEUSR         6
	PTRACE_POKEUSR修改单个寄存器。
#define PTRACE_CONT        7
	PTRACE_CONT使已经被调试器暂停掉或者断掉的进程继续执行。
#define PTRACE_KILL        8
#define PTRACE_SINGLESTEP      9
	PTRACE_SINGLESTEP设置进程的标志寄存器为单步模式并让被调试进程继续执行。被调试进程执行完一条指令后，触发int1异常，并发信号给控制进程，把控制权交给主进程。
	
#define PTRACE_ATTACH         16
	PTRACE_ATTACH是调试进程让被调试进程进入trace模式。
	tracer用来附着一个进程tracee，以建立追踪关系，并向其发送SIGSTOP信号使其暂停。
#define PTRACE_DETACH         17
	解除追踪关系，tracee将继续运行。
#define PTRACE_SYSCALL        24
	PTRACE_SYSCALL设置被调试进程在进入/退出系统调用时被断掉，即进程暂停运行并发送信号给调试主进程。

#define PTRACE_SEIZE 0x4206
像PTRACE_ATTACH附着进程，但它不会让tracee暂停，addr参数须为0，data参数指定一位ptrace选项。
```



### 内核实现
```bash
SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr,
unsigned long, data);
```

```bash
SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr,
        unsigned long, data)
{
    struct task_struct *child;
    long ret;

    if (request == PTRACE_TRACEME) {
        ret = ptrace_traceme();
        if (!ret)
            arch_ptrace_attach(current);
        goto out;
    }

    child = find_get_task_by_vpid(pid);
    if (!child) {
        ret = -ESRCH;
        goto out;
    }

    if (request == PTRACE_ATTACH || request == PTRACE_SEIZE) {
        ret = ptrace_attach(child, request, addr, data);
        /*
         * Some architectures need to do book-keeping after
         * a ptrace attach.
         */
        if (!ret)
            arch_ptrace_attach(child);
        goto out_put_task_struct;
    }

    ret = ptrace_check_attach(child, request == PTRACE_KILL ||
                  request == PTRACE_INTERRUPT);
    if (ret < 0)
        goto out_put_task_struct;

    ret = arch_ptrace(child, request, addr, data);
    if (ret || request != PTRACE_DETACH)
        ptrace_unfreeze_traced(child);

 out_put_task_struct:
    put_task_struct(child);
 out:
    return ret;
}
```

# 作用
Ptrace 可以让父进程控制子进程运行，并可以检查和改变子进程的核心image的功能（Peek and poke 在系统编程中是很知名的叫法，指的是直接读写内存内容）。ptrace主要跟踪的是进程运行时的状态，直到收到一个终止信号结束进程，这里的信号如果是我们给程序设置的断点，则进程被中止，并且通知其父进程，在进程中止的状态下，进程的内存空间可以被读写。当然父进程还可以使子进程继续执行，并选择是否忽略引起中止的信号。

ptrace()系统调用函数提供了一个进程（the “tracer”）**监听**和**控制**另一个进程（the “tracee”）的方法。并且可以**检查和改变“tracee”进程的内存和寄存器里的数据**。它可以用来实现**断点**调试和**系统调用跟踪**。strace和gdb工具就是基于ptrace编写的！

`ptrace` 的一些主要功能包括：
1. 追踪进程：`ptrace` 允许一个进程（称为跟踪器）追踪另一个进程（称为被跟踪进程）的执行。跟踪器可以获取被跟踪进程的状态信息，例如寄存器的值、内存内容等。
    
2. 控制进程：`ptrace` 允许跟踪器对被跟踪进程进行控制操作，例如暂停、继续执行、单步执行等。
    
3. 修改进程状态：`ptrace` 允许跟踪器修改被跟踪进程的状态，例如修改寄存器的值、改变内存中的数据等。
    
4. 监视系统调用：`ptrace` 可以用于监视被跟踪进程的系统调用的发生，并在系统调用执行前后进行处理。


# 使用
## 使用场景
### 调试程序
典型使用场景，就是`gdb`.

`ptrace` 是实现调试器的基础。它可以让调试器监视和控制正在执行的程序，获取程序的状态信息（如寄存器、内存内容），并允许单步执行、设置断点、观察变量等。通过 `ptrace`，调试器可以在程序执行过程中获取关键的调试信息，帮助开发人员定位和修复错误。

### 进程监控
典型使用场景，就是`strace`.

`ptrace` 可以用于监控进程的行为。通过追踪进程，可以收集进程的运行时信息，如系统调用、信号、异常等。这对于系统监控、性能分析、安全审计等方面都非常有用。

## 使用方法
使用 `ptrace` 的一般步骤如下：

1. 使用 `fork` 创建一个新进程，其中一个是跟踪器，另一个是被跟踪进程。
2. 在跟踪器进程中，使用 `ptrace(PTRACE_ATTACH, pid, 0, 0)` 将跟踪器附加到被跟踪进程上，其中 `pid` 是被跟踪进程的进程 ID。
3. 在跟踪器进程中，使用 `ptrace(PTRACE_CONT, pid, 0, 0)` 继续执行被跟踪进程。
4. 可以使用 `ptrace` 提供的其他函数和选项来获取被跟踪进程的状态信息、修改进程状态、监视系统调用等。
5. 当完成对被跟踪进程的操作后，使用 `ptrace(PTRACE_DETACH, pid, 0, 0)` 将跟踪器从被跟踪进程上分离。

> 注意：`ptrace` 的使用需要特权权限（通常是 root 用户或具有相应权限的用户）。

## 应用
### gdb
ptrace可以对被调试进程进行一系列控制动作：可以让被调试进程在进入/退出系统调用时断点，可以对被调试进程的任何位置插入调试断点，可以控制被调试进程单步执行，可以读取/写入被调试进程的寄存器，可以读取/写入被调试进程的堆栈内容。

gdb利用ptrace的这些特性实现了对进程的调试功能。

### strace
strace也是一个常用的调试工具，strace的功能是追踪程序的系统调用。

实现strace，查看 HelloWorld 程序执行过程中的系统调用：
```c
    switch(pid = fork())
    {
    case -1:
        return -1;
    case 0: //子进程
        ptrace(PTRACE_TRACEME,0,NULL,NULL);
        execl("./HelloWorld", "HelloWorld", NULL);
    default: //父进程
        wait(&val); //等待并记录execve
        if(WIFEXITED(val))
            return 0;
        syscallID=ptrace(PTRACE_PEEKUSER, pid, ORIG_EAX*4, NULL);
        printf("Process executed system call ID = %ld/n",syscallID);
        ptrace(PTRACE_SYSCALL,pid,NULL,NULL);
        while(1)
        {
            wait(&val); //等待信号
            if(WIFEXITED(val)) //判断子进程是否退出
                return 0;
            if(flag==0) //第一次(进入系统调用)，获取系统调用的参数
            {
                syscallID=ptrace(PTRACE_PEEKUSER, pid, ORIG_EAX*4, NULL);
                printf("Process executed system call ID = %ld ",syscallID);
                flag=1;
            }
            else //第二次(退出系统调用)，获取系统调用的返回值
            {
                returnValue=ptrace(PTRACE_PEEKUSER, pid, EAX*4, NULL);
                printf("with return value= %ld/n", returnValue);
                flag=0;
            }
            ptrace(PTRACE_SYSCALL,pid,NULL,NULL);
        }
    }
```


在上面的程序中，`fork`出的子进程先调用了`ptrace(PTRACE_TRACEME)`表示子进程让父进程跟踪自己。然后子进程调用`execl`加载执行了 `HelloWorld`。
而在父进程中则使用`wait`系统调用等待子进程的状态改变。子进程因为设置了`PTRACE_TRACEME`而在执行系统调用被系统停止(设置为`TASK_TRACED`)，这时父进程被唤醒，使用`ptrace(PTRACE_PEEKUSER,pid,…)`分别去读取子进程执行的系统调用ID(放在`ORIG_EAX`中)以及系统调用返回时的值(放在`EAX`中)。然后使用 `ptrace(PTRACE_SYSCALL,pid,…)`指示子进程运行到下一次执行系统调用的时候(进入或者退出)，直到子进程退出为止。

程序的执行结果如下:
```bash
Process executed system call ID = 11
Process executed system call ID = 45 with return value= 134520832
Process executed system call ID = 192 with return value= -1208934400
Process executed system call ID = 33 with return value= -2
Process executed system call ID = 5 with return value= -2

其中，11号系统调用就是execve，45号是brk, 192是mmap2, 33是access, 5是open…经过比对可以发现，和strace的输出结果一样。当然strace进行了更详尽和完善的处理，我们这里只是揭示其原理.
```
### ltrace
ltrace用来最终程序运行过程中对库函数的调用。
ltrace其实也是基于ptrace。我们知道，ptrace能够主要是用来跟踪系统调用，那么它是如何跟踪库函数呢？

首先ltrace打开elf文件，对其进行分析。在elf文件中，出于动态连接的需要，需要在elf文件中保存函数的符号，供连接器使用。具体格式，大家可以参考elf文件的格式。这样ltrace就能够获得该文件中，所有系统调用的符号，以及对应的执行指令。然后，ltrace将该指令所对应的4个字节，替换成断点。这样在进程执行到相应的库函数后，就可以通知到了ltrace，ltrace将对应的库函数打印出来之后，继续执行子进程。实际上ltrace与strace使用的技术大体相同，但ltrace在对支持fork和clone方面，不如strace。strace在收到frok和clone等系统调用后，做了相应的处理，而ltrace没有。
### pstack
pstack用来显示运行中函数的堆栈调用情况。
其是实质上也是用ptrace来实现的，首先用PTRACE_ATTACH停住被查看程序，然后尝试从”/proc/pid/exe”中解析出程序elf中的符号表，再通过PTRACE_PEEKUSER读出程序的堆栈指针，通过PTRACE_PEEKDATA读出堆栈的数据，根据堆栈数据在符号表中查询，解析出程序的整个堆栈调用关系。随后PTRACE_DETACH恢复程序的运行。

# 范例
## 为ptrace子进程设置LD_PRELOAD环境变量
要为`ptrace`子进程设置`LD_PRELOAD`环境变量，可以使用`execv`函数来执行子进程，并在`execv`调用之前设置`LD_PRELOAD`环境变量。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>

int main() {
    pid_t child;
    int status;

    child = fork();

    if (child == 0) {
        // 子进程
        char* const args[] = {"/path/to/child_process", NULL}; // 子进程可执行文件的路径
        char* const env[] = {"LD_PRELOAD=/path/to/preload_library.so", NULL}; // 预加载库的路径

        // 设置LD_PRELOAD环境变量
        if (execvpe(args[0], args, env) < 0) {
            perror("execvpe");
            exit(1);
        }
    } else if (child > 0) {
        // 父进程
        if (ptrace(PTRACE_ATTACH, child, NULL, NULL) < 0) {
            perror("ptrace");
            exit(1);
        }

        waitpid(child, &status, 0); // 等待子进程停止

        if (WIFEXITED(status)) {
            printf("Child process exited with status %d\n", WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("Child process terminated by signal %d\n", WTERMSIG(status));
        }

        ptrace(PTRACE_DETACH, child, NULL, NULL);
    } else {
        perror("fork");
        exit(1);
    }

    return 0;
}
```
在上面的示例代码中，我们使用`fork`函数创建子进程，并在子进程中使用`execvpe`函数来执行子进程的可执行文件。在`execvpe`函数调用之前，我们设置了`LD_PRELOAD`环境变量为预加载库的路径。

父进程使用`ptrace`函数将子进程附加到调试器，并使用`waitpid`函数等待子进程停止。然后，我们可以根据需要对子进程进行调试或其他操作。最后，父进程使用`ptrace`函数将子进程从调试器中分离。

## ptrace追踪某进程的某个系统调用
```c
#include<sys/wait.h>/*引入wait函数的头文件*/
#include<sys/reg.h>/* 对寄存器的常量值进行定义，如Eax，EBX....... */
#include<sys/user.h>/*gdb调试专用文件，里面有定义好的各种数据类型*/
#include<sys/ptrace.h>/*引入prtace头文件*/
#include<unistd.h>/*引入fork函数的头文件*/
#include<sys/syscall.h> /* SYS_write */
#include<stdio.h>
int main() {
    pid_t child;/*定义子进程变量*/
    long orig_rax;//定义rax寄存器的值的变量
    int status;/*定义进程状态变量*/
    int iscalling = 0;/*判断是否正在被调用*/
    struct user_regs_struct regs;/*定义寄存器结构体数据类型*/
    child = fork();/*利用fork函数创建子进程*/
    if(child == 0) 
    {
        ptrace(PTRACE_TRACEME, 0, 0);//发送信号给父进程表示已做好准备被跟踪（调试）
        execl("/bin/ls", "ls", "-l", "-h", NULL);/*执行命令ls -l -h,注意，这里函数参数必须要要以NULL结尾来终止参数列表*/
    }
    else
    {
        while(1)
        {
            wait(&status);//等待子进程发来信号或者子进程退出
            if(WIFEXITED(status))//WIFEXITED函数(宏)用来检查子进程是被ptrace暂停的还是准备退出
            {
                break;
            }
            orig_rax = ptrace(PTRACE_PEEKUSER, child, 8 * ORIG_RAX, 0);//获取rax值从而判断将要执行的系统调用号
            if(orig_rax == SYS_write)//如果系统调用是write
            {    
                ptrace(PTRACE_GETREGS, child, 0, &regs);
                if(!iscalling)
                {
                    iscalling = 1;
                    printf("SYS_write call with %lld, %lld, %lld\n",regs.rdi, regs.rsi, regs.rdx);//打印出系统调用write的各个参数内容
                }
                else
                {
                    printf("SYS_write call return %lld\n", regs.rax);//打印出系统调用write函数结果的返回值
                    iscalling = 0;
                }
            }

            ptrace(PTRACE_SYSCALL, child, 0, 0);//PTRACE_SYSCALL,其作用是使内核在子进程进入和退出系统调用时都将其暂停
            //得到处于本次调用之后下次调用之前的状态
        }
    }
    return 0;
}
```
运行结果如下：
![](attachments/Pasted%20image%2020240312111532.png)
# 参考
```bash
# 威力巨大的系统调用——ptrace （+++++）
https://zhuanlan.zhihu.com/p/653385264

# Linux沙箱入门——ptrace从0到1
https://www.anquanke.com/post/id/231078


# [monitor] 9. Linux ptrace(程序调试器原理)
https://www.cnblogs.com/pwl999/p/15535055.html
```