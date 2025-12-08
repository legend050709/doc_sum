```table-of-contents
```
# 介绍
# 使用
```c
# -T 打印系统调用花费的时间  
# -tt 打印系统调用的时间点  
# -s 输出的最大长度，默认32，对于调用参数较长的场景，建议加大  
# -f 是否追踪fork出来子进程的系统调用，由于服务端服务普通使用线程池，建议加上  
# -p 指定追踪的进程pid  
# -o 指定追踪日志输出到哪个文件，不指定则直接输出到终端  

$ strace -T -tt -f -s 10000 -p 87 -o strace.log

strace -T -tt -f -s 10000 -p 87 |& tee strace.log
```
## 存在子进程的命令追踪
```c
strace -T -tt -f CMD
```

## 多线程追踪
```
strace -T -tt -fp PID

```

## 指定线程追踪
一个一个进程存在多个线程，那么只追踪指定的线程，如下所示：
```bash
strace -T -tt -fp TID
```

## 追踪所有信息
```bash
strace -T -tt -e trace=all -fp PID
```



# 范例
## 场景一
**问题**
在进行bind9的监控过程中，发现监控出现了**断点**的情况。那么断点是bind9程序有问题呢，还是 bind9的监控有问题，还是falcon有问题。


**查询**
如下：查询是否是falcon本身的问题，还是bind9的监控执行的问题。如果falcon本身没有问题，那么应该是每隔30s执行一下 execve，execve中调用监控脚本。
如果间隔时间超过了30s，那么就应该是 falcon自身的问题。

```bash
strace -tt -T -s 1000 -e execve -fp `pidof falcon-agent` 2>&1 |grep 30_bind9_zones_maintaince.py

注：falcon-agent 进程一直是不退出的。
```

## 谁杀了我的进程


# 其他
大多数进程基本都会使用基础c库，而不是系统调用，如Linux上的glibc，Windows上的msvc，所以还有一个工具ltrace，可以用来追踪库调用，如下：
```c
ltrace -T -tt -f -s 10000 -p 87 -o ltrace.log
```
基本用法和strace一样，一般来说，使用strace就够了。
# 参考
```c
# 我的进程去哪儿了，谁杀了我的进程
https://www.cnblogs.com/xybaby/p/8098229.html
```