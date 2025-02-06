```table-of-contents
```
# 背景
将某个进程的多个线程绑定到指定的Core上，然后进行CPU的隔离，期望其他的进程不会占用该进程绑定的CPU，但是实际过程中可能会出现其他的进程占用了这些CPU，进而导致被绑定的进程的性能受到了影响，比如丢包。

那么如何查询到是哪个进程占用了指定的CPU的呢？

# kernel的debug-trace方式

## 打开开关

```bash
cd /sys/kernel/debug/tracing/events/sched/sched_switch

echo "prev_pid == 1200 || next_pid == 1200" > filter
# 注： 1200 是指定进程(比如dpvs)的某个线程的 thread_id;

echo 1 > enable
```


## 查看
### 清空之前的 trace 记录
```bash
# 清空当前的跟踪输出
echo > /sys/kernel/debug/tracing/trace
```

### 查看
```bash
# 查看
tailf /sys/kernel/debug/tracing/trace
或
cat /sys/kernel/debug/tracing/trace

```
![](attachments/Pasted%20image%2020240130104052.png)

### 字段说明

![](attachments/Pasted%20image%2020241225181627.png)

整行输出表示的是一个调度事件，记录了进程 `named`（PID 31192）在 CPU 5 上被调度出去的情况，它处于不可中断的等待状态，并且被切换到空闲进程 `swapper/5`（PID 0）。这个信息对于分析系统的调度行为、性能调优以及故障排查非常有用。

**`named-31192`**:
- **named**: 这是进程的名称（`comm`），表示当前正在执行的进程。
- **31192**: 这是进程的 ID（PID），即该进程的唯一标识符。
    
**`[005]`**:
- 这是一个数字，表示该事件发生时的 CPU 核心编号。在这个例子中，事件发生在 CPU 5 上。

**`d...`**:
- 这个部分表示进程的调度状态。`d` 表示该进程处于“等待”状态（`TASK_UNINTERRUPTIBLE`），后面的省略号可能表示其他状态的简略表示或当前状态的更多信息。
    
**`67645572.119615`**:
    - 这是时间戳，表示事件发生的时间。通常是从**系统启动以来的秒数**（包括小数部分），在这个例子中是67645572.119615秒。
    
 **sched_switch**:
    - 这是事件的类型，表示调度器正在进行进程切换。`sched_switch` 是 Linux 内核中的一个事件，表示当前正在进行的进程上下文切换。
    
**prev_comm=named**:
    - `prev_comm` 表示前一个（即被切换出去的）进程的名称，这里是 `named`。

 **prev_pid=31192**:
    - `prev_pid` 表示前一个进程的 PID，这里是 `31192`。

 **prev_prio=120**:
    - `prev_prio` 表示前一个进程的优先级，这里是 `120`。优先级的数值越小，优先级越高。
    
**prev_state=D**:
    - `prev_state` 表示前一个进程的状态，这里是 `D`，表示该进程处于不可中断的等待状态（`TASK_UNINTERRUPTIBLE`）。

 **==>**:
    - 这个符号表示进程切换的方向，指示前一个进程被切换出去，接下来是下一个进程的信息。

**next_comm=swapper/5**:
    - `next_comm` 表示下一个（即被切换进来的）进程的名称，这里是 `swapper/5`，通常表示系统的空闲进程或调度程序。

**next_pid=0**:
    - `next_pid` 表示下一个进程的 PID，这里是 `0`，通常指的是空闲进程。

 **next_prio=120**:
    - `next_prio` 表示下一个进程的优先级，这里也是 `120`。


## 关闭开关
```bash
cd /sys/kernel/debug/tracing/events/sched/sched_switch
echo 0 > enable
echo 0 > filter

查看：
# cat filter
none

# cat enable
0
```

## 其他
### trace 的 buffer size调整
如果trace 发现不输出，或者输出不全。可能是 buffer 空间不足导致存在丢包的情况。
```bash
# ls /sys/kernel/debug/tracing/buffer_*
-rw-r--r-- 1 root root 0 Nov  3  2022 buffer_size_kb
-r--r--r-- 1 root root 0 Nov  3  2022 buffer_total_size_kb

```

`buffer_size_kb` 记录 `CPU buffer` 的大小，单位为 `KB` 。
`per_cpu/cpuX/buffer_size_kb` 记录 `每个CPU buffer` 大小，单位为 `KB` 。可通过写 `buffer_size_kb` 来改变 `CPU buffer` 的大小。

`buffer_total_size_kb` 记录所有 `CPU buffer` 的总大小，即所有 CPU buffer 大小总和。`buffer_total_size_kb` 文件是只读的。

```bash
echo 28160 > buffer_size_kb

# cat buffer_size_kb
28160
# cat buffer_total_size_kb
1802240
```

### 获取trace的系统时间
如上所示，trace打印的是 开机启动之后的秒数。那么如何转换为系统时间呢。
通过如下的脚本：
```bash
# cat test11.sh
# 假设你的 trace 输出的时间戳是67645572.119615
timestamp=67645572.119615

#btime 获取系统启动时间(以s为单位)
btime=$(awk '/btime/ {print $2}' /proc/stat)

# 计算可读时间
readable_time=$(date -d "@$(echo "$btime + $timestamp" | bc)" +"%Y-%m-%d %H:%M:%S.%N")
echo $readable_time
```


### 多线程的进程的调度切换
```bash
cd /sys/kernel/debug/tracing/events/sched/sched_switch
echo "(prev_pid >= 31167 && prev_pid <= 31201) || (next_pid >= 31167 && next_pid <= 31201)" > filter
```

# ftrace 方式


# perf 方式

# 其他方式
假设 dpvs 进程绑定的 CPU是 `2-9,18-26`,查看还有其他的哪些进程抢占了这些Core。通过下面的脚本。

```bash
while true; do out=`ps -T -eo psr,%cpu,stat,pid,tid,args:150 | awk -F" " '{if(($1 <= 25 && $1 >=18) || ($1 <= 9 && $1 >=2))   print $0}' | grep -v "\[" | grep -v dpvs`; if [[ -n "$out" ]]; then date; echo ---------------------------------; echo $out; fi; done >> /tmp/cpuinfo3
```


# 其他
## 查看一个进程的所有线程号
```bash
ps -T -p PID

or

ll /proc/PID/task

```

# 参考
```bash
# Linux内核调试的方式以及工具集锦
https://github.com/gatieme/LDD-LinuxDeviceDrivers/blob/master/study/debug/README.md


https://www.toutiao.com/article/6643950886622069261/

## ftrace 简介
https://abcdxyzk.github.io/blog/2016/03/28/debug-ftrace/
```