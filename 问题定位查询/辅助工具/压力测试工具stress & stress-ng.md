```table-of-contents
```
# 介绍
stress 命令主要用来模拟系统负载较高时的场景,
可以对cpu、memory、IO以及磁盘进行压力测试。
# 操作
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
# 模拟CPU压力
stress 消耗 CPU 资源的方式是通过调用 sqrt 函数计算由 rand 函数产生的随机数的平方根实现的。
# 模拟内存压力
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
# 模拟IO压力
产生 N 个进程，每个进程都反复调用 sync 函数，刷新内存缓冲区到磁盘上。

```c
stress --io 2 --timeout 60s
开启2个IO进程，执行sync系统调用，刷新内存缓冲区到磁盘， 60s结束。
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
# 模拟生成多个进程
```text
stress -c N
```
# stress-ng
stress-ng完全兼容stress, 并且在stress基础上增加数百个选项参数，支持产生各种复杂的压力。
# 参考
```c
https://juejin.cn/post/7040095840089669639
https://juejin.cn/post/7063709470173429768
https://zhuanlan.zhihu.com/p/457147071
https://www.cnblogs.com/sparkdev/p/10354947.html
```
