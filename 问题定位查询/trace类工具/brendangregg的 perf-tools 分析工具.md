```table-of-contents
```
# 背景
在实际工作中，偶尔会遇到系统的CPU使用率和系统平均负载很高，但却找不到高CPU的应用；
产生这个问题的原因：进程有可能在不断的崩溃、重启。即==存在一些瞬时进程==。

# 新进程创建探测工具：execsnoop
## 介绍
对于传统的CPU分析工具，比如说top。其==对于运行时间十分短的进程是无法做到检测的==，在这种场景下就可以考虑使用`execsnoop`来捕获这些进程。

`execsnoop`-专门用于为追踪短时进程（瞬时进程）设计的工具；
它通过 `ftrace` 实时监控进程的 `exec()` 函数，并输出短时进程的基本信息，包括进程 PID、父进程 PID、命令行参数以及执行的结果。
## 安装
要安装execsnoop，只要安装Perf-Tools，Perf-Tools 是基于 perf_events (perf) 和 ftrace 的Linux性能分析调优工具集。Perf-Tools 依赖库少，使用简单。

```bash
# 下载Perf-Tools
https://github.com/brendangregg/perf-tools


# 安装及使用execsnoop
unzip perf-tools-master.zip  
cd perf-tools-master/
```

```bash
# 下载:
https://github.com/brendangregg/perf-tools/blob/master/execsnoop

# 下载或者拷贝文件内容 写到 /usr/bin/execsnoop ，
并执行 chmod +x /usr/bin/execsnoop

```
## 使用

# 进程退出探测工具：exitsnoop
## 介绍
exitsnoop工具来自BCC工具集，其可以跟踪进程退出事件，打印出进程的总运行时常和退出原因。

## 安装
## 使用
# CPU队列长度工具：runqlen
## 介绍
runqlen是一个基于BCC和bpftrace的工具，用来采样CPU运行队列的长度信息，可以统计有多少线程正在等待运行，并以线性直方图的方式输出。
## 安装
## 使用

# 系统调用统计工具：syscount
## 介绍
syscount是一个bcc和bpftrace工具，用于统计一段时间内系统中系统调用的频次信息。

## 使用
```text
# syscount
# -T TOP：仅打印调用频率最高的N个结果
# -L：打印系统调用的总耗时
# -P：每个进程打印一个直方图
# -p pid：仅测量给定的进程
./syscount -T 5 -L

```

下面是在笔者电脑上开启syscout统计系统调用的频次信息，这里我们可以看到排名最高的poll这个系统调用，其调用63次的开销和select居然是差不多的，这也说明了poll的系统调用性能比较高。

```bash
[root@bogon ik]# syscount -T 5 -L
Tracing syscalls, printing top 5... Ctrl+C to quit.
[06:17:31]
SYSCALL                   COUNT        TIME (us)
poll                         63     16442734.871
select                       10     13211296.193
futex                       596     12234961.939
epoll_wait                  139      5257698.941
ppoll                         8      4225473.030

```


# 参考
```bash
# BPF——CPU分析工具

https://blog.csdn.net/shenmingxueIT/article/details/130996808
```