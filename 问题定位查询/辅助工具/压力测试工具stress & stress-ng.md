```table-of-contents
```
# stress
## 介绍
stress 命令主要用来模拟系统负载较高时的场景,
可以对**cpu、memory、IO以及磁盘**进行压力测试。


## 安装
```c
yum install stress -y
```

## 使用

![](attachments/Pasted%20image%2020231031112349.png)

```text
-c, --cpu N      产生 N 个进程，每个进程都反复不停的计算随机数的平方根
-i, --io N       产生 N 个进程，每个进程反复调用 sync() 将内存上的内容写到硬盘上
-m, --vm N       产生 N 个进程，每个进程不断分配和释放内存
--vm-bytes B     指定分配内存的大小
--vm-stride B    不断的给部分内存赋值，让 COW(Copy On Write)发生
--vm-hang N      指示每个消耗内存的进程在分配到内存后转入睡眠状态 N 秒，然后释放内存，一直重复执行这个过程
--vm-keep        一直占用内存，区别于不断的释放和重新分配(默认是不断释放并重新分配内存)
-d, --hadd N     产生 N 个不断执行 write 和 unlink 函数的进程(创建文件，写入内容，删除文件)
--hadd-bytes B   指定文件大小
-t, --timeout N  在 N 秒后结束程序        
--backoff N      等待N微妙后开始运行
-q, --quiet      程序在运行的过程中不输出信息
-n, --dry-run    输出程序会做什么而并不实际执行相关的操作
--version        显示版本号
-v, --verbose    显示详细的信息
```

### 模拟CPU压力

#### 之前的CPU压力测试
```python
while true;do echo "压力测试" ; done
```

#### stress cpu压力测试

使用`stress --cpu/-c`选项可以进行CPU压力测试。可以指定要使用的CPU核心数量。

```bash
# 使用以下命令模拟4个CPU核心的满载负载
stress --cpu/-c  4


```

stress 消耗 CPU 资源的方式是通过调用 sqrt 函数计算由 rand 函数产生的随机数的平方根实现的。

### 模拟内存压力

产生N个子进程，每个进程分配一定量的内存资源。

**--vm-keep**  
一直占用内存，区别于不断的释放和重新分配(默认是不断释放并重新分配内存)。  

**--vm-hang N**  
指示每个消耗内存的进程在分配到内存后转入睡眠状态 N 秒，然后释放内存，一直重复执行这个过程。


```c
### 同时进行CPU和内存测试
并发10个进程，8个CPU负载，2个内存负载（128M*2），持续10秒。
stress --cpu 8 --vm 2 --vm-bytes 128M --timeout 10s
```


```c
启动2个消耗内存的进程，每个进程占用200M内存
# stress -m 2 --vm-bytes 200M
stress: info: [4364] dispatching hogs: 0 cpu, 0 io, 2 vm, 0 hdd
```



```c
用pidstat 查看内存的占用情况
# pidstat -r | grep stress
13时32分48秒   UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
13时33分38秒     0      4364      0.04      0.00    7948    1044   0.03  stress
13时33分38秒     0      4365   8748.16      0.00  212752   56072   1.46  stress
13时33分38秒     0      4366   9156.42      0.00  212752   91712   2.38  stress
```

### 模拟IO压力

#### 磁盘IO

-i,--io：表示产生 N 个进程，每个进程都反复调用调用sync()，它表示通过系统调用 sync() 来模拟 I/O 的问题；


```c
stress --io 2 --timeout 60s
开启2个IO进程，执行sync系统调用，刷新内存缓冲区到磁盘， 60s结束。

使用以下命令模拟4个线程同时进行磁盘读写操作：
stress --io 4 --io-size 1G
```

![](attachments/Pasted%20image%2020231031113030.png)

使用stress无法模拟iowait升高，但sys升高。
stress -i参数表示通过系统调用sync来模拟IO问题，但sync是刷新内存缓冲区数据到磁盘中，以确保同步。如果内存缓冲区内没多少数据，读写到磁盘中的数据也就不多，没法产生IO压力。
使用SSD磁盘的环境中尤为明显，iowait一直为0，但因为大量系统调用，导致系统CPU使用率sys升高。

```c
stress --io 2 --hdd 2 --timeout 60s
开启2个IO进程，2个磁盘IO进程
```

![](attachments/Pasted%20image%2020231031113130.png)


#### 问题

但这种方法实际上并不可靠，因为 sync() 的本意是刷新内存缓冲区的数据到磁盘中，以确保同步。
如果缓冲区内本来就没多少数据，那读写到磁盘中的数据也就不多，也就没法产生 I/O 压力。
这一点，在使用 SSD 磁盘的环境中尤为明显，很可能你的 iowait 总是 0，却单纯因为大量的系统调用，导致了系统CPU使用率 sys 升高。
这种情况，推荐使用 stress-ng 来代替 stress。


### 模拟生成多个进程

```text
stress -c N

# 起2个吃CPU的后台进程 
$ stress -c 2 &

```


## 缺陷

由于stress的压力模型非常简单，所以无法模拟任何复杂的场景。

举个例子，在stress压测过程中，如果用top命令去观察，会发现所有的cpu压力都在用户态，内核态没有任何压力：

![](attachments/Pasted%20image%2020240605161706.png)


# stress-ng

## 介绍

stress-ng是一个功能强大的Linux性能测试工具，它可以在Linux系统上模拟各种负载情况，对CPU、内存、磁盘I/O、网络等方面进行全面而深入的性能测试

。**stress-ng是stress工具的增强版，完全兼容stress, 并且在此基础上通过几百个参数，可以产生各种复杂的压力**。



## 功能与特点

### 1. 多维度测试

stress-ng支持对CPU、内存、磁盘I/O、网络等多个方面进行测试，可以模拟各种实际工作负载，帮助用户全面评估系统的性能表现。

### 2. 灵活的测试参数

stress-ng提供了丰富的测试参数，用户可以根据需要自定义测试类型、持续时间、并发数等，以满足不同的测试需求。

### 3. 支持多核测试

stress-ng支持多核测试，可以同时模拟多个CPU核心的工作负载，以评估系统在多核环境下的性能表现。

### 4. 详细的测试报告

stress-ng在测试过程中会生成详细的测试报告，包括各种性能指标、错误信息等，帮助用户分析系统性能瓶颈和优化方向。



## 安装

```bash
yum install stress-ng

```

## 使用

### CPU压力测试

```bash
 -c, --cpu N: 产生 N 个进程，每个进程都反复不停的计算随机数的平方根
每个进程会占用一个cpu，当超出cpu个数时，进程间会互相争用cpu
```


通过stress-ng的CPU测试，您可以**模拟多核CPU的高负载情况**。

```bash
模拟4个CPU核心的满负荷运行，持续60秒。
stress-ng --cpu 4 --timeout 60s

产生2个worker做圆周率算法压力：
stress-ng --cpu 4 --cpu-method pi

产生2个worker做30多种不同的压力算法，比如 pi, crc16, fft 等
stress-ng --cpu 4 --cpu-method all

```




#### cpu 任务绑核

```
# 启动3个任务，并且绑核
stress-ng --taskset 0,2-3 --cpu 3

```


###  系统调用的压力

```bash
产生2个worker调用socket相关函数产生压力
stress-ng --sock 2

产生2个worker读取tsc产生压力
stress-ng --tsc 2

将压力指定到cpu 0,2,3,6：
stress-ng --sock 4 --taskset 0,2-3,6

```


### 内存压力测试
stress-ng同样可以用于**内存压力测试**，**模拟内存分配和释放**的过程。

```bash

启动8个进程，占用80%的可用内存，因此每个占用 10%的可用内存
stress-ng --vm 8 --vm-bytes 80% -t 1h


模拟分配和释放1GB内存的过程，持续60秒。
stress-ng --vm 1 --vm-bytes 1G --timeout 60s


创建10个进程，共分配180M内存，不断释放并重新分配
stress-ng --vm 10 --vm-bytes 180M


创建10个进程，共分配180M内存，并一直占用
stress-ng --vm 10 --vm-bytes 180M --vm-keep


创建10个进程，共分配180M内存，内存分配后睡眠60s，然后释放
stress-ng --vm 10 --vm-bytes 180M --vm-hang 60
```

```bash

使用2颗CPU
[root@nginx ~]#   stress --cpu 2 --timeout 600

[root@nginx ~]# uptime
10:33:44 up 28 min,  4 users,  load average: 1.99, 1.39, 0.81

[root@nginx ~]# mpstat -P ALL 5 1
Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all   50.05    0.00    0.08    0.00    0.00    0.00    0.00    0.00    0.00   49.87
Average:       0    0.07    0.00    0.17    0.00    0.00    0.01    0.00    0.00    0.00   99.75
Average:       1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
Average:       2  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
Average:       3    0.08    0.00    0.15    0.01    0.00    0.01    0.00    0.00    0.00   99.76

[root@nginx sysstat-12.1.5]# pidstat -u 5

1.通过uptime可以观察到，系统平均负载很高，通过mpstat观察到2个CPU使用率很高，平均负载也很高，而iowait为0，说明进程是CPU密集型的；
2.是由进程使用CPU密集导致系统平均负载变高、CPU使用率变高; 
3.可以通过pidstat查看是哪个进程导致CPU使用率较高

```


### 磁盘压力测试

**对IO进行压测(使用stress观测到的iowait指标可能为0，所以使用stress-ng)**。

通过stress-ng的磁盘测试，您可以**模拟磁盘读写操作**，评估磁盘性能。


```bash
-i,--io：表示调用sync()，它表示通过系统调用 sync() 来模拟 I/O 的问题；
但这种方法实际上并不可靠，因为 sync() 的本意是刷新内存缓冲区的数据到磁盘中，以确保同步。

如果缓冲区内本来就没多少数据，那读写到磁盘中的数据也就不多，也就没法产生 I/O 压力。
```

```bash
对IO进行压测(使用stress观测到的iowait指标可能为0，所以使用stress-ng)
[root@nginx ~]# stress-ng -i 4 --hdd 1 --timeout 600

[root@nginx ~]# uptime
11:11:12 up  1:05,  4 users,  load average: 4.35, 4.11, 3.65

[root@nginx ~]# mpstat -P ALL 5
Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all    0.20    0.00   13.04   38.70    0.00    1.33    0.00    0.00    0.00   46.73
Average:       0    0.07    0.00    6.63   40.96    0.00    3.72    0.00    0.00    0.00   48.62
Average:       1    0.19    0.00   20.14   26.77    0.00    0.04    0.00    0.00    0.00   52.85
Average:       2    0.27    0.00   13.81   45.15    0.00    0.88    0.00    0.00    0.00   39.89
Average:       3    0.27    0.00   11.22   42.20    0.00    0.80    0.00    0.00    0.00   45.51

[root@nginx sysstat-12.1.5]# pidstat -u 5

1.可以通过uptime观察到，系统平均负载很高，通过mpstat观察到CPU使用很低，iowait很高，一直在等待IO处理，说明此进程是IO密集型的；
2.是由进程频繁的进行IO操作，导致系统平均负载很高而CPU使用率不高的情况；
```





### 网络压力测试

stress-ng还提供了网络压力测试功能，可以**模拟网络负载和网络延迟**。

```bash
模拟4个网络连接，每个连接具有100毫秒的延迟，持续60秒。
stress-ng --network 4 --network-delay 100ms --timeout 60s
```

### 模拟大量进程

主要是测试：等待CPU的进程->进程间会争抢CPU。

```bash
模拟16个进程，本机是4核
[root@nginx ~]# stress -c 16 --timeout 600

[root@nginx ~]# uptime
11:23:24 up  1:18,  4 users,  load average: 15.10, 8.98, 6.04

[root@nginx ~]# mpstat -P ALL 5
Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all   99.92    0.00    0.08    0.00    0.00    0.00    0.00    0.00    0.00    0.00
Average:       0   99.87    0.00    0.13    0.00    0.00    0.00    0.00    0.00    0.00    0.00
Average:       1   99.96    0.00    0.04    0.00    0.00    0.00    0.00    0.00    0.00    0.00
Average:       2   99.90    0.00    0.10    0.00    0.00    0.00    0.00    0.00    0.00    0.00
Average:       3   99.93    0.00    0.07    0.00    0.00    0.00    0.00    0.00    0.00    0.00

[root@nginx sysstat-12.1.5]# pidstat -u 5 1
Linux 3.10.0-957.21.3.el7.x86_64 (nginx)     07/10/2019  _x86_64_    (4 CPU)

11:23:07 AM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
11:23:12 AM     0     23613   25.15    0.00    0.00   75.25   25.15     1  stress
11:23:12 AM     0     23614   24.95    0.00    0.00   75.45   24.95     0  stress
11:23:12 AM     0     23615   25.15    0.00    0.00   75.25   25.15     0  stress
11:23:12 AM     0     23616   24.95    0.00    0.00   74.65   24.95     0  stress
11:23:12 AM     0     23617   25.15    0.00    0.00   74.85   25.15     1  stress
11:23:12 AM     0     23618   24.75    0.00    0.00   75.25   24.75     1  stress
11:23:12 AM     0     23619   24.75    0.00    0.00   75.85   24.75     2  stress
11:23:12 AM     0     23620   24.55    0.00    0.00   75.65   24.55     2  stress
11:23:12 AM     0     23621   25.35    0.00    0.00   74.85   25.35     3  stress
11:23:12 AM     0     23622   25.35    0.00    0.00   74.45   25.35     3  stress
11:23:12 AM     0     23623   25.15    0.00    0.00   75.65   25.15     1  stress
11:23:12 AM     0     23624   25.35    0.00    0.00   74.45   25.35     3  stress
11:23:12 AM     0     23625   24.55    0.00    0.00   75.45   24.55     2  stress
11:23:12 AM     0     23626   24.95    0.00    0.00   75.45   24.95     0  stress
11:23:12 AM     0     23627   24.75    0.00    0.00   75.65   24.75     3  stress
11:23:12 AM     0     23628   24.55    0.00    0.00   75.05   24.55     2  stress
11:23:12 AM     0     23803    0.20    0.40    0.00    0.80    0.60     2  watch
11:23:12 AM     0     24022    0.00    0.20    0.00    0.00    0.20     2  pidstat

1.通过uptime观察到系统平均负载很高，通过mpstat观察到CPU使用率也很高，iowait为0，说明此进程是CPU密集型的，或者在进行CPU的争用；
2.通过pidstat -u观察到wait指标很高，则说明进程间存在CPU争用的情况，可以判断系统中存在大量的进程在等待使用CPU；
3.大量的进程，超出了CPU的计算能力，导致的系统的平均负载很高；

```

### 混合使用

```bash
cpu、磁盘io、内存混合使用

stress-ng --cpu 4 --io 2 --vm 1 --vm-bytes 1G --timeout 60s


```

# 单进程多线程压力测试

主要是测试大量线程造成上下文切换，从而造成系统负载升高。

## Sysbench
### 介绍

Sysbench 是一个基于 LuaJIT 的可编写脚本的多线程基准测试工具，可以执行 CPU、内存、线程、IO、数据库等方面的性能测试，**常用于评估测试各种不同系统参数下的数据库负载情况，不需要修改源码，通过自定义 Lua 脚本就可以实现不同业务类型的测试**。



### 测试

```bash
模拟10个线程，对系统进行基准测试
[root@nginx ~]# sysbench --threads=10 --time=300 threads run

可以看到平均1分钟的系统再升高
[root@nginx ~]# uptime
16:43:41 up  6:38,  4 users,  load average: 4.82, 2.10, 0.84

可以看到sys(内核态)对CPU的使用率比较高，iowait无（表示没有进程间的争用）
[root@nginx ~]# mpstat -P ALL 5
Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all   23.92    0.00   68.92    0.00    0.00    0.00    0.00    0.00    0.00    7.16
Average:       0   24.11    0.00   68.77    0.00    0.00    0.00    0.00    0.00    0.00    7.12
Average:       1   24.02    0.00   68.41    0.00    0.00    0.00    0.00    0.00    0.00    7.56
Average:       2   23.74    0.00   69.40    0.00    0.00    0.01    0.00    0.00    0.00    6.85
Average:       3   23.79    0.00   69.10    0.00    0.00    0.00    0.00    0.00    0.00    7.10

可以看到无进程间的上下文切换（默认是进程间的）
[root@nginx ~]# pidstat -w  3
04:45:46 PM   UID       PID   cswch/s nvcswch/s  Command
04:45:49 PM     0         9      2.67      0.00  rcu_sched
04:45:49 PM     0        11      0.33      0.00  watchdog/0
04:45:49 PM     0        12      0.33      0.00  watchdog/1
04:45:49 PM     0        14      0.67      0.00  ksoftirqd/1
04:45:49 PM     0        17      0.33      0.00  watchdog/2
04:45:49 PM     0        22      0.33      0.00  watchdog/3
04:45:49 PM     0       556     19.67      0.00  xfsaild/dm-0
04:45:49 PM     0     21287      0.33      0.00  sshd
04:45:49 PM     0     26834      1.00      0.00  kworker/u256:0
04:45:49 PM     0     30955      1.00      0.00  kworker/2:2
04:45:49 PM     0     31207      1.67      0.00  kworker/0:2
04:45:49 PM     0     31778      2.00      0.00  kworker/3:1
04:45:49 PM     0     32262      0.33      0.00  pidstat
04:45:49 PM     0     32263      1.00      0.00  kworker/1:1
04:45:49 PM     0     32350      0.33      0.00  kworker/3:0

可以看到存在大量的非自愿上下文切换（表示线程间争用引起的上下文切换，造成系统负载升高）
[root@nginx ~]# pidstat -w -t 3
04:48:35 PM   UID      TGID       TID   cswch/s nvcswch/s  Command
04:48:41 PM     0     32597         -      1.67      0.33  sysbench
04:48:41 PM     0         -     32597      1.67      0.33  |__sysbench
04:48:41 PM     0         -     32598   8932.67  63606.33  |__sysbench
04:48:41 PM     0         -     32599  10554.00  52275.33  |__sysbench
04:48:41 PM     0         -     32600  10941.00  49976.67  |__sysbench
04:48:41 PM     0         -     32601   9393.67  50796.33  |__sysbench
04:48:41 PM     0         -     32602  10196.67  50815.33  |__sysbench
04:48:41 PM     0         -     32603   9538.67  54755.00  |__sysbench
04:48:41 PM     0         -     32604  10112.33  50476.67  |__sysbench
04:48:41 PM     0         -     32605   9135.67  53922.00  |__sysbench
04:48:41 PM     0         -     32606  10506.33  55677.00  |__sysbench
04:48:41 PM     0         -     32607  10346.33  55691.67  |__sysbench
```



# 参考
```c
## 压力测试神器stress-ng
https://cloud.tencent.com/developer/article/1513544

# Linux命令拾遗-top中的%nice是啥？
https://juejin.cn/post/7040095840089669639

# Linux命令拾遗-%iowait指标代表了什么？
https://juejin.cn/post/7063709470173429768

# Linux环境下使用stress进行压力测试
https://zhuanlan.zhihu.com/p/457147071

# stress 命令
https://www.cnblogs.com/sparkdev/p/10354947.html


# 如何使用stress工具用于 Linux 系统的压力测试？
https://www.linuxprobe.com/stress-linux.html
```
