```table-of-contents
```
# 介绍
# 范例
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
# 其他
大多数进程基本都会使用基础c库，而不是系统调用，如Linux上的glibc，Windows上的msvc，所以还有一个工具ltrace，可以用来追踪库调用，如下：
```c
ltrace -T -tt -f -s 10000 -p 87 -o ltrace.log
```
基本用法和strace一样，一般来说，使用strace就够了。
# 参考
```c

```