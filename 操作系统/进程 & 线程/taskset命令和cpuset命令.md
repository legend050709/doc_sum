```table-of-contents
```
# 背景知识
## 超线程

**超线程技术(Hyper-Threading)：** 就是利用特殊的硬件指令，**把两个逻辑内核(CPU core)模拟成两个物理芯片，**（一个物理核模拟出两个逻辑核。）

### 背景
尽管提高CPU的时钟频率和增加缓存容量后的确可以改善CPU性能，但这样的CPU性能提高在技术上存在较大的难度。

实际上在应用中基于很多原因，**CPU的执行单元都没有被充分使用**。
如果CPU不能正常读取数据(总线/内存的瓶颈)，其执行单元利用率会明显下降。另外就是目前大多数执行线程缺乏ILP(Instruction-Level Parallelism，多种指令同时执行)支持。这些都造成了目前CPU的性能没有得到全部的发挥。

### 超线程技术
因此，Intel则采用另一个思路去提高CPU的性能，让CPU可以同时执行多重线程，就能够让CPU发挥更大效率，即所谓“超线程(Hyper-Threading，简称“HT”)”技术。超线程技术就是利用特殊的硬件指令，把两个逻辑内核模拟成两个物理芯片，让单个处理器都能使用线程级并行计算，进而兼容多线程操作系统和软件，减少了CPU的闲置时间，提高的CPU的运行效率。

让**单个处理器都能使用线程级并行计算**，进而兼容多线程操作系统和软件，减少了CPU的闲置时间，提高的CPU的运行效率。

我们常听到的双核四线程/四核八线程指的就是支持超线程技术的CPU.

### 其他
#### CPU个数
主板插槽上 cpu芯片的个数

#### 物理核
嵌在cpu芯片上的处理器，一个cpu可以有多个内核，其id都不一样.

#### 逻辑核
通过超线程技术，能将一个物理核分成多个逻辑核,也就是代码层面的多线程技术

一般情况，我们认为一颗CPU可以有多核，加上intel的超线程技术(HT), 可以在逻辑上再分一倍数量的CPU core出来；
逻辑CPU数量 = 物理CPU数量 x CPU cores x 2(如果支持并开启HT)

### 查看

#### linux上查看CPU相关信息
CPU的信息主要都在/proc/cupinfo中

```bash
# 查看物理CPU个数
cat /proc/cpuinfo|grep "physical id"|sort -u|wc -l

# 查看每个物理CPU中core的个数(即核数)
cat /proc/cpuinfo|grep "cpu cores"|uniq

# 查看逻辑CPU的个数
cat /proc/cpuinfo|grep "processor"|wc -l

# 查看CPU的名称型号
cat /proc/cpuinfo|grep "name"|cut -f2 -d:|uniq
```

#### 查看某个进程运行在哪个逻辑CPU上

```bash
ps -eo pid,args,psr

#参数的含义：
pid  - 进程ID
args - 该进程执行时传入的命令行参数
psr  - 分配给进程的逻辑CPU
```

## CPU亲和性
### cpu绑定
`CPU`绑定指的是在多核`CPU`的系统中将进程或线程绑定到指定的`CPU`核上去执行。在`Linux`中，我们可以利用`CPU affinity`属性把进程绑定到一个或多个`CPU`核上。



### cpu亲和性

`CPU Affinity`是进程的一个调度属性(scheduler property)，这个属性指明了进程调度器能够把这个进程调度到哪些`CPU`上。该属性要求进程在某个指定的`CPU`上尽量长时间地运行而不被迁移到其他处理器。

在SMP(Symmetric Multi-Processing对称多处理)架构下，**Linux调度器(scheduler)会根据CPU affinity的设置让指定的进程运行在"绑定"的CPU上,而不会在别的CPU上运行.**

Linux调度器同样支持自然CPU亲和性(natural CPU affinity): **调度器会试图保持进程在相同的CPU上运行**, 这意味着进程通常不会在处理器之间频繁迁移



### 注意

绑核（即设置 亲和性）以后，Linux调度器就会让这个进程/线程只在所绑定的核上面去运行。但**并不是说该进程/线程就独占这个CPU的核，其他的进程/线程还是可以在这个核上面运行的**。如果想要实现某个进程/线程独占某个核，就要使用`cpuset`命令去实现。

### 表示方法

CPU affinity 使用位掩码(bitmask)表示, 每一位都表示一个CPU, 置1表示"绑定".

最低位表示第一个**逻辑CPU**, 最高位表示最后一个**逻辑CPU**.


```bash
0x00000001
    is processor #0

0x00000003
    is processors #0 and #1

0xFFFFFFFF
    is all processors (#0 through #31)
```

# taskset

## 介绍

taskset 可以查询或设置进程（线程）绑定CPU（亲和性） ; 
通过 taskset 命令可将某个进程与某个CPU核心绑定，使得其仅在与之绑定的CPU核心上运行。
线程是最小的内核执行调度单元，因此，准确地说是将某个线程与某个CPU核心绑定，而非某个进程。  
taskset 是依据 **线程PID（TID）查询或设置线程的CPU亲和性（与哪个CPU核心绑定）**.



## 参数

使用方法：
```bash
       taskset [options] mask command [arg]...
       taskset [options] -p [mask] pid
```

![](attachments/Pasted%20image%2020240515193701.png)

```bash
-a, --all-tasks 操作所有的任务线程
-p, --pid 操作已存在的pid
-c, --cpu-list 通过列表显示方式设置CPU（逗号相隔）
-V, --version           输出版本信息
```

### 绑核方式
设置绑核有两种方法：

- 列表形式
- 掩码形式

#### 指定 CPU 列表 绑核**

`cpu-list`是数字化的cpu列表，从0开始。多个不连续的cpu可用逗号连接，连续的可用短现连接，比如0,2,5-11等。


```bash
taskset -pc cpu-list pid

范例：
（1）新进程启动使用列表形式：
taskset -c 0,4,7-11 /usr/local/bin/my_embedded_process


（2）已经存在进程使用列表形式：

将进程9865绑定到`0`、`2`、`5~11`号核上面
taskset -pc 0,2,5-11 9865


```

**指定CPU掩码进行绑核**：
```bash
taskset 0xF0 /usr/local/bin/my_embedded_process
```


## 使用
### 指定pid绑核
```bash
(1) 掩码方式
taskset -p MASK PID

指示 PID 为 7013 的进程仅在 CPU 0 上运行。
# taskset -p 1 7013

(2) CPU列表形式
taskset -pc 新的CPU号 pid

```

### 新程序启动时绑核
```bash
taskset -c 0,4,7-11 /usr/local/bin/my_embedded_process
```

### 显示已经运行的进程的CPU亲和性
#### taskset 方法
查出进程pid现在的绑核情况。

(1) **掩码方式查询**：
```bash
taskset -p pid

比如：
# taskset -p 2641
pid 2641's current affinity mask: ffffffff

# taskset -p 2676
pid 2676's current affinity mask: 100  （100 即 core 8）

```


（2）**列表形式查询**：
```bash
taskset -pc pid

比如：
# taskset -pc 2676
pid 2676's current affinity list: 8
```

#### 其他方法
(1) **查看进程或线程的亲和性**：

```bash
进程：
cat /proc/PID/status | grep -i cpu

线程：
cat /proc/TID/status | grep -i cpu

```

```bash

# cat /proc/18149/status | grep -i cpu
Cpus_allowed:	ffffffff
Cpus_allowed_list:	0-31
```

(2) **查看线程的亲和性**：


```bash
cat /proc/TID/status | grep -i cpu

or

cat /proc/PID/task/TID/status | grep -i cpu
```

```bash
# cat /proc/18149/task/18183/status | grep -i cpu
Cpus_allowed:	80000000
Cpus_allowed_list:	31

or

# cat /proc/18183/status | grep -i cpu
Cpus_allowed:	80000000
Cpus_allowed_list:	31
```

### 线程绑核以及查询线程的绑核

将上面的 PID 替换为 TID(即现场ID) 即可。其他的 操作都是相同的。


#### 获取多进程下所有线程的亲和性
```bash
taskset -a -p PID
taskset -a -pc PID
```

范例：
```bash
# taskset -a -p 2641
pid 2641's current affinity mask: e
pid 2642's current affinity mask: ffffffff
pid 2643's current affinity mask: ffffffff
pid 2644's current affinity mask: ffffffff
pid 2645's current affinity mask: ffffffff
pid 2646's current affinity mask: ffffffff
pid 2647's current affinity mask: ffffffff
pid 2648's current affinity mask: ffffffff
pid 2650's current affinity mask: ffffffff
pid 2651's current affinity mask: ffffffff
pid 2652's current affinity mask: ffffffff
pid 2653's current affinity mask: ffffffff
pid 2654's current affinity mask: ffffffff
pid 2655's current affinity mask: ffffffff
pid 2656's current affinity mask: ffffffff
pid 2657's current affinity mask: ffffffff
pid 2658's current affinity mask: ffffffff
pid 2659's current affinity mask: ffffffff
pid 2660's current affinity mask: ffffffff
pid 2661's current affinity mask: ffffffff
pid 2662's current affinity mask: ffffffff
pid 2663's current affinity mask: ffffffff
pid 2664's current affinity mask: ffffffff
pid 2665's current affinity mask: ffffffff
pid 2666's current affinity mask: ffffffff
pid 2667's current affinity mask: ffffffff
pid 2668's current affinity mask: ffffffff
pid 2669's current affinity mask: ffffffff
pid 2670's current affinity mask: 200
pid 2671's current affinity mask: ffffffff
pid 2672's current affinity mask: ffffffff
pid 2673's current affinity mask: ffffffff
pid 2674's current affinity mask: ffffffff
pid 2675's current affinity mask: ffffffff
pid 2676's current affinity mask: 100
```


#### 设置多进程下所有线程的亲和性
##### 所有线程有相同的亲和性
```bash

taskset -a -p MASK PID
```

范例如下所示：
```bash
# taskset -a -p 0xf0ff00 2641
```

![](attachments/Pasted%20image%2020240516160444.png)

##### 不同线程不同的亲和性
```bash
# 获取线程ID
ps -T -p <进程ID>

# numa情况查看
numactl --hardware


# 更改绑核情况, 假如 12830-12863 为 named 进程生成的线程的 线程的id；
cnt=1;for a in `seq 12830 12863`; do taskset -pc $cnt $a; cnt=$((cnt+1)); if [[ $cnt -eq 31 ]]; then cnt=1; fi; done

# 理论生效信息: 查看进程下的线程的绑核情况
taskset -a -pc PID


# 实际生效信息验证：进程的线程实际运行的core的信息
ps -L -p PID  -o pid,ppid,lwp,nlwp,psr,state,lstart,etimes,bsdtime,pcpu,vsz,rss,pmem,minflt,majflt,wchan:25,ucmd | sort -k5 -n

```

### 注意

#### taskset 给 进程的线程绑核不会立刻生效

![](attachments/image%20(18).png)

如上所示，通过taskset 给named的线程设置了绑核之后，然后通过ps查看，大概需要等待一段时间（分钟级别，3分钟左右），绑核才可以生效。

# cpuset

# linux C中CPU亲和性
## 背景
在`Linux`中，用结构体`cpu_set_t`来表示`CPU Affinity`掩码，同时定义了一系列的宏来用于操作进程的可调度`CPU`集合：
```c
#define _GNU_SOURCE 
#include <sched.h>
void CPU_ZERO(cpu_set_t *set);
void CPU_SET(int cpu, cpu_set_t *set);
void CPU_CLR(int cpu, cpu_set_t *set);
int CPU_ISSET(int cpu, cpu_set_t *set);
int CPU_COUNT(cpu_set_t *set);



CPU_ZERO()：清除集合的内容，让其不包含任何CPU。
CPU_SET()：添加cpu到集合中。
CPU_CLR()：从集合中移除cpu
CPU_ISSET() ：测试cpu是否在集合中。
CPU_COUNT()：返回集合中包含的CPU数量。
```

## 将进程与CPU核进行绑核

在`Linux`中，可以使用以下两个函数设置和获取进程的`CPU Affinity`属性：

```c

#define _GNU_SOURCE 
#include <sched.h>
int sched_setaffinity(pid_t pid, size_t cpusetsize,const cpu_set_t *mask);
int sched_getaffinity(pid_t pid, size_t cpusetsize,cpu_set_t *mask);

```

另外可以通过下面的函数获知当前进程运行在哪个`CPU`上：
```c
int sched_getcpu(void);
```

### 范例
```text
#define _GNU_SOURCE
#include <sched.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>
 
int main(int argc, char *argv[])
{
    cpu_set_t set;
    int parentCPU, childCPU;
    int j;
    int cpu_num = -1;
    if (argc != 3) {
        fprintf(stderr, "Usage: %s parent-cpu child-cpu\n", argv[0]);
        exit(EXIT_FAILURE);
    }
    parentCPU = atoi(argv[1]);
    childCPU = atoi(argv[2]);
    CPU_ZERO(&set);
    
    switch (fork()) {
        case -1: { /* Error */
            fprintf(stderr, "fork error\n");
            exit(EXIT_FAILURE);
        }
        case 0: { /* Child */
            CPU_SET(childCPU, &set);
            if (sched_setaffinity(getpid(), sizeof(set), &set) == -1) {
                fprintf(stderr, "child sched_setaffinity error\n");
                exit(EXIT_FAILURE);
            }
            sleep(1);
            if (-1 != (cpu_num = sched_getcpu())) {
                fprintf(stdout, "The child process is running on cpu %d\n", cpu_num);
            }
            exit(EXIT_SUCCESS);
        }
        default: { /* Parent */
            CPU_SET(parentCPU, &set);
            if (sched_setaffinity(getpid(), sizeof(set), &set) == -1) {
                fprintf(stderr, "parent sched_setaffinity error\n");
                exit(EXIT_FAILURE);
            }
            if (-1 != (cpu_num = sched_getcpu())) {
                fprintf(stdout, "The parent process is running on cpu %d\n", cpu_num);
            }
            wait(NULL); /* Wait for child to terminate */
            exit(EXIT_SUCCESS);
        }
    }
}
```

程序首先用`CPU_ZERO`清空`CPU`集合，然后调用`fork()`函数创建一个子进程，并调用`sched_setaffinity()`函数给父进程和子进程分别设置`CPU Affinity`，输入参数`parentCPU`和`childCPU`分别指定父进程和子进程运行的`CPU`号。指定父进程和子进程运行的`CPU`为1和0，程序输出如下：

```text
# ./affinity_test 1 0
The parent process is running on cpu 1
The child process is running on cpu 0
```


## 将线程与CPU核进行绑核

前面介绍了进程与`CPU`的绑定，那么线程可不可以与`CPU`绑定呢？当然是可以的。在`Linux`中，可以使用以下两个函数设置和获取线程的`CPU Affinity`属性：

```c
#define _GNU_SOURCE 
#include <pthread.h>
int pthread_setaffinity_np(pthread_t thread, size_t cpusetsize, const cpu_set_t *cpuset);
int pthread_getaffinity_np(pthread_t thread, size_t cpusetsize, cpu_set_t *cpuset);
```

### 范例

```text
#define _GNU_SOURCE
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
 
static void *thread_start(void *arg) {
    ......
    struct thread_info *tinfo = arg;
    thread = tinfo->thread_id;
    CPU_ZERO(&cpuset);
    CPU_SET(tinfo->thread_num, &cpuset);
    s = pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset);
    if (s != 0) {
        handle_error_en(s, "pthread_setaffinity_np");
    } 
    CPU_ZERO(&cpuset);
    s = pthread_getaffinity_np(thread, sizeof(cpu_set_t), &cpuset);
    if (s != 0) {
        handle_error_en(s, "pthread_getaffinity_np");
    }
    
    for (j = 0; j < cpu_num; j++) {
        if (CPU_ISSET(j, &cpuset)) { //如果当前线程运行在CPU j上，则输出信息
            printf(" thread %d is running on cpu %d\n", tinfo->thread_num, j);
        }
    }
    pthread_exit(NULL);
}
 
int main(int argc, char *argv[])
{
    ......
    cpu_num = sysconf(_SC_NPROCESSORS_CONF); //获取系统的CPU数量
    tinfo = calloc(cpu_num, sizeof(struct thread_info));
    
    if (tinfo == NULL) {
        handle_error_en(0, "calloc");
    }
    
    for (j = 0; j < cpu_num; j++) { //有多少个CPU就创建多少个线程
        tinfo[j].thread_num = j;
        s = pthread_create(&tinfo[j].thread_id, NULL, thread_start, &tinfo[j]);
        if (s != 0) {
            handle_error_en(s, "pthread_create");
        }
    }
    
    for (j = 0; j < cpu_num; j++) {
        s = pthread_join(tinfo[j].thread_id, NULL);
        if (s != 0) {
            handle_error_en(s, "pthread_join");
        }
    }
    ......
}
```


程序首先获取当前系统的`CPU`数量`cpu_num`，然后根据`CPU`数量的数量创建线程，有多少个`CPU`就创建多少个线程，每个线程都运行在不同的`CPU`上。在4核的机器中运行结果如下：
```text
$ ./thread_affinity
 thread 1 is running on cpu 1
 thread 0 is running on cpu 0
 thread 3 is running on cpu 3
 thread 2 is running on cpu 2
```

# 参考
```bash
# 如何将进程、线程与CPU核进行绑定
https://zhuanlan.zhihu.com/p/432940336
```