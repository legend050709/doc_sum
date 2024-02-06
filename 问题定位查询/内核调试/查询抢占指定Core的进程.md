```table-of-contents
```
# 背景
将某个进程的多个线程绑定到指定的Core上，然后进行CPU的隔离，期望其他的进程不会占用该进程绑定的CPU，但是实际过程中可能会出现其他的进程占用了这些CPU，进而导致被绑定的进程的性能受到了影响，比如丢包。

那么如何查询到是哪个进程占用了指定的CPU的呢？

# debug查询
（1）打开开关

```bash
cd /sys/kernel/debug/tracing/events/sched/sched_switch

echo "prev_pid == 1200 || next_pid == 1200" > filter
# 注： 1200 是指定进程(比如dpvs)的某个线程的 thread_id;

echo 1 > enable
```


（2）查看
```bash
cd /sys/kernel/debug/tracing
tailf trace
```
![](attachments/Pasted%20image%2020240130104052.png)

（3）关闭开关
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


# 其他方式
假设 dpvs 进程绑定的 CPU是 `2-9,18-26`,查看还有其他的哪些进程抢占了这些Core。通过下面的脚本。

```bash
while true; do out=`ps -T -eo psr,%cpu,stat,pid,tid,args:150 | awk -F" " '{if(($1 <= 25 && $1 >=18) || ($1 <= 9 && $1 >=2))   print $0}' | grep -v "\[" | grep -v dpvs`; if [[ -n "$out" ]]; then date; echo ---------------------------------; echo $out; fi; done >> /tmp/cpuinfo3
```

# 参考
```bash
# Linux内核调试的方式以及工具集锦
https://github.com/gatieme/LDD-LinuxDeviceDrivers/blob/master/study/debug/README.md


https://www.toutiao.com/article/6643950886622069261/

## ftrace 简介
https://abcdxyzk.github.io/blog/2016/03/28/debug-ftrace/
```