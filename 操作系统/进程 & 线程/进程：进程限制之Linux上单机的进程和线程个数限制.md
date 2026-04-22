```table-of-contents
```
# 概述
 Linux 系统中可以设置关于资源的使用限制，比如：进程数量，文件句柄数，连接数等等。
 
# 最大进程个数
如何查看linux系统默认的最大进程数，这里以centos7(x64)作为例子:
```c
(1) 系统级别
[root@es1 ~]# cat /proc/sys/kernel/pid_max
131072

类似的还有 /proc/sys/fs/file-max

（2）用户级别
[root@es1 ~]# ulimit -a | grep processes
max user processes              (-u) 15012

类似的还有： ulimit -a | grep "open files"

（3）进程级别
[root@es1 ~]# cat /proc/1/limits |grep processes
Max processes             15012                15012                processes 

```
注意第一种才是内核级别的配置，后面的设置不能超过内核级别设置的限制，这个值是可以具体的情况修改的。

## 系统允许的最大进程数
```c
cat  /proc/sys/kernel/pid_max
```

## 当前用户允许打开的最大进程数
查看 max user processes  ：即系统限制某用户下最多可以运行多少进程或线程。如下图所示：
![](attachments/Pasted%20image%2020231120153212.png)

# 最大线程个数
其实最大线程数量也可以配置无限大，在资源充足的情况下，但一般都有会默认限制，主要影响线程的参数如下：

```c
ulimit -s  栈大小设置
ulimit -i  阻塞的引号量
ulimit -u  最大的线程/进程数
/proc/sys/kernel/threads-max 最大线程数量
/proc/sys/vm/max_map_count  限制一个进程可以拥有的VMA(虚拟内存区域)的数量
/proc/sys/kernel/pid_max  最大进程数量
```

## 系统允许的最大线程数
```c
cat /proc/sys/kernel/threads-max
```
## 单个进程允许的最大线程数
Linux无法直接控制单个进程可拥有的线程数，但有参考公式`max = VM/stack_size`，默认
stack为8k。用 ulimit -s 可以查看默认的线程栈大小，一般情况下，这个值是8M=8192KB。
在32位系统中，可以最多创建381个线程，这个值和理论完全相符，因为 **32 位 linux 下的进程用户空间是 3G 的大小(32位一共是4G，其中内核空间1G，用户空间3G)**，也就是 3072M，用3072M/8M（stack大小）=3843072M/8M=384，但是实际上代码段和数据段等还要占用一些空间，这个值应该向下取整到 383，再减去主线程，得到 382。

- **提升单个进程的线程个数**
可通过降低stack大小或增加虚拟内存来调大每个进程可拥有的最大线程数；
为了突破内存的限制，可以有两种方法
- 用ulimit -s 1024减小默认的栈大小。
- 调用pthread_create的时候用pthread_attr_getstacksize设置一个较小的栈大小。

# 其他
## 最大文件描述符个数
- **文件描述符定义**
文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符

关于文件描述符的最大数量，其实是可以无限大的，但考虑到每一个文件描述符都需要一定数量的内存和磁盘维护，所以还是有限制的

原因有两方面：
（1）系统本身的资源有限；
（2）比如一个机器有多个用户，如果没有限制，某一个用户起了无限多的进程和无休止的创建文件描述符，就直接有可能导致整台机器挂掉，影响了其他正常的用户的使用，所以还是有必要给不同的用户根据所需限制文件描述的数量。


- **相关配置**
```c
[root@es1 ~]# cat /proc/sys/fs/file-max
379804

[root@es1 ~]# ulimit -n
65536

[root@es1 ~]# lsof | wc -l
2201

```
第一个命令代表：当前系统允许创建的最大文件描述符的数量
第二个命令代表：当前会话session允许创建的最大文件描述符，即每个进程允许打开的最大文件描述符数量。
第三个命令代表：统计当前所有进程的占用的文件描述符的总量.

```c
查看每个进程打开的文件描述符的数量，并按打开的数量降序排序：
【第一列是文件描述符数量，第二列是进程id】
lsof -n | awk '{print $2}' | sort | uniq -c  |sort -nr

```

- **nginx服务器的配置参考**
```c
ulimit -HSn 1024000; 
sysctl -w fs.file-max=13025552; 
sysctl -w net.ipv4.ip_local_port_range='1024 64000'
```

### 系统最大打开文件描述符数
`/proc/sys/fs/file-max`中指定了系统范围内所有进程可打开的文件句柄的数量限制(系统级别, kernel-level)。

### 用户级别最大打开文件描述符数
```bash
ulimit -a | grep "open files"
```

### 单个进程可分配的最大文件数
`/proc/sys/fs/nr_open`  即一个进程最多使用的file handle数
> the maximum number of files that can be opened by process。


# limit相关配置
## ulimit命令
## 配置文件/etc/security/limits.conf
```c
soft   xxx  : 代表警告的设定，可以超过这个设定值，但是超过后会有警告。
hard  xxx   : 代表严格的设定，不允许超过这个设定的值。
如：soft 设为1024，hard设为2048 ，则当你使用数在1~1024之间时可以随便使用，1024~2048时会出现警告信息，大于2048时，就会报错。

nproc  : 是操作系统级别对每个用户创建的进程数的限制
nofile : 是每个进程可以打开的文件数的限制
```

设置限制数量，第一列表示用户，* 表示所有用户
soft nproc ：单个用户可用的最大进程数量(超过会警告);  
hard nproc：单个用户可用的最大进程数量(超过会报错);  
soft nofile ：每个进程可打开的文件描述符的最大数(超过会警告);  
hard nofile ：每个进程可打开的文件描述符的最大数(超过会报错);

```text
注：
	①一般soft的值会比hard小，也可相等。
	②/etc/security/limits.d/里面配置会覆盖/etc/security/limits.conf的配置
	③只有root用户才有权限修改/etc/security/limits.conf
	④如果limits.conf没有做设定，则默认值是1024
```
## 代码级别的函数
```c
struct rlimit {
   rlim_t rlim_cur;  /* Soft limit */
   rlim_t rlim_max;  /* Hard limit (ceiling for rlim_cur) */
};

int getrlimit(int resource, struct rlimit *rlim);

int setrlimit(int resource, const struct rlimit *rlim);
```
# 参考
```c
https://zhuanlan.zhihu.com/p/259301109
```