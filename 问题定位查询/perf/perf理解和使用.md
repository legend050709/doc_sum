```table-of-contents
```
# 背景
日常的工作中，会收到一堆CPU使用率过高的告警邮件，遇到某台服务的`CPU被占满了`，这时候我们就要去查看是什么进程将服务器的CPU资源占用满了。通常我们会通过`top`或者`htop`来快速的查看占据CPU最高的那个进程，如下图：
![](attachments/Pasted%20image%2020240130152550.png)
当然你可能遇到一个服务器上运行有多个服务，想快速知道占用率最高的那几个进程的话，你可以使用以下命令:
```bash
ps aux|head -1;ps -aux | sort -k3nr | head -n 10 //查看前10个最占用CPU的进程
ps aux|head -1;ps -aux | sort -k4nr | head -n 10 //查看前10个最占用内存的进程
```
通过以上的方法获取到服务器占用资源的进程之后，还是`不知道CPU使用究竟耗时在哪里`,不清楚瓶颈在哪里，此时就可以通过`Linux`系统的性能分析工具`perf`分析，分析其返回的正在消耗CPU的函数以及调用栈。

# 介绍
Perf全名是`Performance`，是在Linux内建的性能分析工具。
perf通过对监测对象进行采样，根据采样点的分布来推断整个程序的行为。
通过`perf list`命令我们可以看到`perf`支持很多的采样事件，比如`branch-misse`s、`cpu-clock`等等。

## 原理
Perf 可以对程序进行**函数级别的采样**，从而了解程序的性能瓶颈在哪里。
其基本原理是：每隔一个固定时间，就是CPU上产生一个中断，看当前是哪个进程、哪个函数，然后给对应的进程和函数加一个统计值，这样就知道CPU有多少时间在某个进程或某个函数上了。


# 安装

# 使用
![](attachments/Pasted%20image%2020240415104949.png)
![](attachments/Pasted%20image%2020240415104851.png)

## 总览
**perf top**：适合监控整个系统的性能。
类似于系统中的`top`命令。某些时候我们可能并不知道是哪个程序影响了系统性能，这时候就可以用perf top来查找可疑的程序。

**perf stat**：比较适合单个程序的性能分析。


**perf record/report**：更适合对程序进行更细粒度的分析

## perf top
```bash
- 进程级：perf top -p <pid>
- 线程级：perf top -t <tid>
线程tid可以通过 pidstat -t -p <pid>获取 或 ps -T -p xxx
```

我们可以通过-e参数指定统计其他的事件，比如`perf top -e context-switches`可以查看进程切换最多的`top N`个进程。

## perf stat
`perf stat`命令提供常见性能事件的总体统计信息，包括执行的指令和消耗的时钟周期。选项可用于选择默认测量事件以外的事件。

## perf record
perf 的使用方法也很丰富，目前只要会用 `perf record` 和 `perf report` 就能够进行大部分的性能分析了。
```bash
(1) perf record
perf record -F 99 -C 1 --call-graph dwarf -o xxxx.data -- sleep 30
    -F: (freq) 采集的评率
    -p: 指定进程; 
    -C：指定某个cpu；
    --call-graph: 栈回溯方式。
	    fp, dwarf, lbr；
	    dwarf较为常用；
    sleep: 采集的时间，以s为单位；
    -o: output file, 不指定时默认保存到当前目录新建立的 perf.data 文件中；

(2) perf report
perf report  -i FILE
-i: input file; 
```
![](attachments/image.png)


### 范例
```bash
root@master:~# sudo perf record -F 99 -p 25633 -g -- sleep 30
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.039 MB perf.data (120 samples) ]

上面的命令中:
	`perf record`表示记录
	`-F 99`表示每秒99次
	`-p 25633`是进程号，即对哪个进程进行分析
	`-g`表示记录调用栈
	`sleep 30`则是持续30秒，参数信息可以视情况调整。
生成的数据采集文件在当前目录下，名称为`perf.data`。
```
这个命令会产生一个大的数据文件，取决与你采集的进程与CPU的配置，如果一台服务器有16个 CPU，每秒抽样99次，持续30秒，就得到 47,520 (99*30*16)个调用栈，长达几十万甚至上百万行。

## perf report

## perf trace
此命令执行与 `strace` 工具类似的函数。它监控指定线程或进程使用的系统调用，以及该应用收到的所有信号。

## perf script



# 参考
```bash

```