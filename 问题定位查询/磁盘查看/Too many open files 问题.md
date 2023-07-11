# 问题
- Too many open files有四种可能:
1》单个进程打开文件句柄数过多  
2》操作系统打开的文件句柄数过多  
3》systemd对该进程进行了限制  

# 不同层级
## 系统级别
### fs.file-max
参数 fs.file-max 控制了**整个系统所有进程**能够打开的文件的最大数，该参数是由kernel在内核层面限制的，适用于所有用户所有进程。

### fs.file-nr
file-nr，展示了系统级别的，当前已经使用的句柄数量和总的句柄数量。可以拿来做监控。

```c
# cat /proc/sys/fs/file-max
15803154

# cat /proc/sys/fs/file-nr
4720	0	15803154
```
fs.file-nr：
	第一个数表示当前系统已分配的文件描述符数（文件句柄数）；
	第二个数为分配后已释放的文件描述符数（当前不再使用的文件描述符数）；
	第三个数为最大文件描述符数，等于file-max；

### 查看与配置
- 永久配置
```c
**# vim /etc/sysctl.conf**
在文件尾添加如下内容（假设目标大小为65535）：
**fs.file-max=65535**

**确保配置立即生效，执行以下命令**
**# sysctl -p**
```
- 临时配置(**重启后就失效了**)
```c
**# echo 65530 > /proc/sys/fs/file-max**
**# sysctl fs.file-max** **#查看**
fs.file-max = 65530
或者
**# sysctl -w fs.file-max=65531**
```
可以通过如下命令查看全局级别的限制：sysctl fs.file-max；
也可以通过如下命令查看全局级别的限制：cat /proc/sys/fs/file-max；
可以通过如下命令修改全局级别的限制：sysctl -w fs.file-max=xxx；

## 用户级别
### ulimit 
可以在用户级别，配置某个特定用户，可以同时打开的最大文件句柄数，此时有以下几点需要注意：
- 该参数实质是控制某个特定用户下每一个进程所能够同时打开的最大文件句柄数（每个用户的每个进程能够打开的最大文件数）；

- 最大句柄数包括软限制和硬限制（soft limit and hard limit)，且软限制小于等于硬限制，普通用户能够调整自己的 soft limit，而 hard limit只有root用户才能调整；
    
- 临时修改最大文件句柄数限制，可以使用命令 ulimit -n 65535：该命令对该会话下后续新启动的所有进程都会立即生效，但系统重启后修改会丢失（ulimit -Hn/ulimit -Sn）;
    
- 永久修改文件句柄数，需要修改配置文件 /etc/security/limits.conf或/etc/security/limits.d/20-nproc.conf：该修改在系统重启后不会丢失，但用户需要重新登录才能使用这里的修改值；
    
- 更改参数前已经存在的进程，以及这些已经存在的进程 fork 出的新进程，其底层实际生效的 nproc，仍然是参数更改前的值;
    
- systemd 控制的 service unit, 是不受 /etc/security/limits.conf 和 /etc/security/limits.d/20-nproc.conf 中的配置参数影响的，对于他们需要在配置文件中配置 LimitNOFILE（推荐配置 /etc/systemd/system/.d/override.conf，而不是 /usr/lib/systemd/system/.service，因为通过rpm等包管理器升级时后者会被覆盖掉）;
    
- 查看当前会话下用户级别的限制，可以使用命令 ulimit -a；
    
- The ulimit is for filehandles，it applies to files, directories, sockets, pipes epolls, eventfds, timerfds etc etc. The parameter is set at user level, but applied for each process；
    
- You need to relogin to use the changed parms in /etc/security/limits.conf;
    
- Old process's forked sub-process will inherit and use old value;
    
- A duplicted session from an old session may not be using the new ulimit settings, you need use ulimit -a to check;
    
- ulimit can also be set by environment variables in /etc/profile, etc.
    
- you can use prlimit to dynamically modify limit settings for a certain process;
    
- systemd service units will completely igrone ulimit settings.


- 注意：
只有在同一个shell中启动的进程，ulimit -n 的设置才会生效。你打开另外一个shell，或者重启机器，ulimit的改动都会丢失。

### fs.nr_open
限制单个进程上可打开的文件数：具体参数是`softnofile`和`fs.nr_open`。
它们两个区别是`soft nofile`可以针对不同用户配置不同的值，而`fs.nr_open`在一台`Linux`上只能配置一次。


#### 查看
使用ulimit 命令可以分别查看软限制和硬限制，方法实在查看的参数前加 S 或 H。例如，查看打开文件数限制
```c
`ulimit -Sn` 查看的是软限制

`ulimit -Hn` 查看的是硬限制
```

要查看不同用户的硬值和软值，你可以`su`切换用户查看比较。比如：
```c
# su rumenz  
$ ulimit -Sn  
1024
```

#### 设置
编辑 /etc/security/limits.conf 文件：
```c
格式：
<domain>        <type>  <item>  <value>

这是为用户设置软限制和硬限制的示例 `rumenz`用户：
## Example hard limit for max opened files  
rumenz        hard nofile 4096  
## Example soft limit for max opened files  
rumenz        soft nofile 1024
```
注：修改  /etc/security/limits.conf 文件 的方式，也要求你需要打开一个新的shell进行操作。在当前修改的shell里或者修改之前的shell里，同样不生效。


### systemd对进程的限制
被systemd管理的进程（也就是可以通过systemctl来控制的进程）通过修改该进程的service文件（通常在/etc/systemd/system/目录下），在“[Service]”下面添加“LimitNOFILE=20480000”来实现，修改完成之后需要执行"systemctl daemon-reload"来使该配置生效。

## 特定进程的限制
要看到这些改变是否已经对进程生效，可以查看进程的内存映射文件。比如`cat /proc/180323/limits`，其中会有详细的展示。

用户级别对最大句柄数的调整，对某个进程是否生效，还取决于该进程隶属于哪个会话，该进程的父子进程链如何，以及该进程是否受 systemd 管控的影响。

- 对于某个正在运行的进程，其实际生效的句柄数限制，可以通过如下命令查看：cat /proc/$pid/limits | grep "open files"；
    
- 对于某个正在运行的进程，其已经打开的文件句柄数，可以通过如下命令行获得：ls /proc/$pid/fd |wc -l；


### lsof 查看
最后，我们经常使用 lsof 查看某个进程或当前整个系统所打开的文件数，比如：

- 通过如下命令查看某个进程打开的所有文件：lsof -p $pid |grep / | awk '{print $9}'|sort | uniq;
    
- 通过如下命令查看当前系统打开的所有文件的句柄数：sudo lsof | wc -l；

> 注意：lsof 和 fs.file-nr统计的口径并不一致：lsof count is more than fs.file-nr count， and the reason is, file-nr ignores some of the directories which are considered as files by lsof;
![](attachments/Pasted%20image%2020230725161816.png)

#### lsof 查看的缺陷
CentOS 7中的lsof是按PID/TID/file的组合显示结果的。
如果一个进程含有多个线程，同一个线程可能在每个线程中都显示了一遍，导致含有大量的重复。
因此查看一个进程下的fd的数目的正确方法为：
```c
ls -l /proc/<pid>/fd | wc -l

或

lsof -p PID | wc
```

- 其他
```c
查看一个进程的线程数:
cat /proc/PID/status | grep -i  Threads
或
ll /proc/PID/task/ | wc
【对于一个进程中创建的每个线程，在 `/proc/<pid>/task` 中会创建一个相应的目录，命名为其线程 ID。由此在 `/proc/<pid>/task` 中目录的总数表示在进程中线程的数目。】
或
pstree -p PID

或
ps -hH -p <pid>|wc-l
```


### 应用
- 查看系统中打开fd最多的进程
```c
find /proc -print | grep -P '/proc/\d+/fd/'| awk -F '/' '{print $3}' | uniq -c | sort -rn | head

注：以下方法可能存在缺陷「因为多线程原因」
lsof -n | awk '{print $2}' | sort | uniq -c | sort -nr
```

### prlimit 设置/获取某进程的资源限制
- 背景
假设有个场景，数据库或者其它中间件的运行时文件句柄等参数设置过低，导致服务不可用或者间歇性不可用。
但是重启服务的代价可能很大，那么我们也可以不重启进程，做到修改某个进程的 limits范围。
这里可以使用 prlimit 命令来实现。
prlimit: process resource limit. 提供了进程粒度的资源限制的能力.
【bash 的 ulimit 限制当前 user/ shell 及其子进程的资源使用】
 
- 使用
![](attachments/Pasted%20image%2020230725172004.png)
![](attachments/Pasted%20image%2020230725172041.png)

- 范例
```c
  $ cat /proc/1204/limits | grep file   # 看到文件句柄限制为10240了
  Max file size             unlimited            unlimited            bytes     
  Max core file size        0                    unlimited            bytes     
  Max open files            10240                10240                files     
  Max file locks            unlimited            unlimited            locks     

  # 用prlimit 搞一下
  $ prlimit --nofile=65535:65535 --pid 1204

  $ cat /proc/1204/limits | grep file        # 再次查看，可以看到已经变成 65535了 
  Max file size             unlimited            unlimited            bytes     
  Max core file size        0                    unlimited            bytes     
  Max open files            65535                65535                files     
  Max file locks            unlimited            unlimited            locks  
```

## 区别
在`Linux`上能打开多少个文件，有两种限制：

- 第一种：进程级别，限制的是单个进程上可打开的文件数。具体参数是`softnofile`和`fs.nr_open`。它们两个区别是`soft nofile`可以针对不同用户配置不同的值，而`fs.nr_open`在一台`Linux`上只能配置一次。
    
- 第二种：系统级别，整个系统上可以打开的最大文件数，具体参数是`fs.file-mas`，但这个参数不限制`root`用户。
- 
## 小结
![](attachments/Pasted%20image%2020230725170125.png)

Linux进程的连接的上限，受到单进程文件句柄数量和操作系统文件句柄数量的限制，也就是ulimit和file-max。

为了能够将参数修改持久化，我们倾向于将改动写入到文件里。
进程的文件句柄限制，可以放在`/etc/security/limits.conf`中，它的上限受到`fs.nr_open`的制约；
操作系统的文件句柄限制，可以放到`/etc/sysctl.conf`文件中。
最后，别忘了在`/proc/$id/limits`文件中，确认修改是否对进程生效了。


另外这几个参数之间还有耦合关系，因此还要注意以下三点：

- 1、如果你想加大 soft nofile,  那么 hard nofile 也需要一起调整。因为如果 hard nofile 设置的低， 你的 soft nofile 设置的再高都没用，实际生效的值会按二者里最低的来。
    
- 2、如果你加大了 hard nofile，那么 fs.nr_open 也都需要跟着一起调整。**如果不小心把 hard nofile 设置的比 fs.nr_open 大了，后果比较严重。会导致该用户无法登陆。如果设置的是 * 的话，那么所有的用户都无法登陆。**
    
- 3、还要注意如果你加大了 fs.nr_open，但是用的是 echo "xx" > ../fs/nr_open 的方式，刚改完你可能觉得没问题。只要机器一重启你的 fs.nr_open 设置就会失效，还是会无法登陆。


假如你想让你的进程可以打开 100 万个文件描述符，我觉得比较稳妥点的修改方法是干脆都直接用 conf 文件的方式来改。这样比较统一，也比较安全。
```c
 vi /etc/sysctl.conf  
fs.nr_open=1100000  //要比 hard nofile 大一点  
fs.file-max=1100000 //多留点buffer  
# sysctl -p  
# vi /etc/security/limits.conf  
*  soft  nofile  1000000  
*  hard  nofile  1000000
```

# 原理
以`socket`为例，来讲解上面`softnofile`和`fs.nr_open`、`fs.file-mas`三个参数的关系。
![](attachments/Pasted%20image%2020230725170855.png)

![](attachments/Pasted%20image%2020230725170925.png)

```c
int get_unused_fd_flags(unsigned flags)  
{  
//RLIMIT_NOFILE是limits.conf中配置的nofile  
return __alloc_fd(current->files, 0, rlimit(RLIMIT_NOFILE), flags);  
}  
EXPORT_SYMBOL(get_unused_fd_flags);

//file: include/linux/sched/signal.h  
static inline unsigned long rlimit(unsigned int limit)  
{  
return task_rlimit(current, limit);  
}  
  
static inline unsigned long task_rlimit(const struct task_struct *tsk,  
unsigned int limit)  
{  
return READ_ONCE(tsk->signal->rlim[limit].rlim_cur);  
}
```
其中`rlimit(RLIMIT_NOFILE)`这个就是读取`limits.conf`中配置的`soft nofile`；通过当前进程描述符访问到`rlim[RLIMIT_NOFILE]`对象、这个对象`rlim_cur`就是`soft nofile`

在`__alloc_fd`中会判断要分配的句柄号是否超过了limits.conf中的nofile的限制。fd是当前进程相关的，是一个从0开始的整数。如果超限，就报错`EMFILE 24 /* Too many open files */`。
函数最后会进入`expand_files`:
```c
/*  
* Expand files.  
* This function will expand the file structures, if the requested size exceeds  
* the current capacity and there is room for expansion.  
* Return <0 error code on error; 0 when nothing done; 1 when files were  
* expanded and execution may have blocked.  
* The files->file_lock should be held on entry, and will be held on exit.  
*/  
static int expand_files(struct files_struct *files, unsigned int nr)  
__releases(files->file_lock)  
__acquires(files->file_lock)  
{  
struct fdtable *fdt;  
int expanded = 0;  
  
repeat:  
fdt = files_fdtable(files);  
  
/* Do we need to expand? */  
if (nr < fdt->max_fds)  
return expanded;  
  
/* Can we expand? */  
if (nr >= sysctl_nr_open)  
return -EMFILE;  
  
if (unlikely(files->resize_in_progress)) {  
spin_unlock(&files->file_lock);  
expanded = 1;  
wait_event(files->resize_wait, !files->resize_in_progress);  
spin_lock(&files->file_lock);  
goto repeat;  
}  
  
/* All good, so we try */  
files->resize_in_progress = true;  
expanded = expand_fdtable(files, nr);  
files->resize_in_progress = false;  
  
wake_up_all(&files->resize_wait);  
return expanded;  
}
```
# 一台Linux服务器最多能支撑多少个TCP连接？
## 背景
>"TCP连接四元组是源IP地址、源端口、目的IP地址和目的端口。任意一个元素发生了改变，那么就代表的是一条完全不同的连接了。拿我的Nginx举例，它的端口是固定使用80。另外我的IP也是固定的，这样目的IP地址、目的端口都是固定的。剩下源IP地址、源端口是可变的。所以理论上我的Nginx上最多可以建立2的32次方（ip数）×2的16次方（port数）个连接。这是两百多万亿的一个大数字！！"

![](attachments/Pasted%20image%2020230725174016.png)

## 限制
- 打开文件的限制
>"进程每打开一个文件（linux下一切皆文件，包括socket），都会消耗一定的内存资源。如果有不怀好心的人启动一个进程来无限地创建和打开新的文件，会让服务器崩溃。所以linux系统出于安全角度的考虑，在多个位置都限制了可打开的文件描述符的数量，包括系统级、用户级、进程级。这三个限制的含义和修改方式如下："

- 系统级：当前系统可打开的最大数量，通过fs.file-max参数可修改
    
- 用户级：指定用户可打开的最大数量，修改/etc/security/limits.conf
    
- 进程级：单个进程可打开的最大数量，通过fs.nr_open参数可修改

- 内存的限制
![](attachments/Pasted%20image%2020230725174159.png)

>"我的接收缓存区大小是可以配置的，通过sysctl命令就可以查看。"
```c
$ sysctl -a | grep rmem  
net.ipv4.tcp_rmem = 4096 87380 8388608  
net.core.rmem_default = 212992  
net.core.rmem_max = 8388608
```
>"其中在tcp_rmem"中的第一个值是为你们的TCP连接所需分配的最少字节数。该值默认是4K，最大的话8MB之多。也就是说你们有数据发送的时候我需要至少为对应的socket再分配4K内存，甚至可能更大。"

![](attachments/Pasted%20image%2020230725174302.png)
> "TCP分配发送缓存区的大小受参数net.ipv4.tcp_wmem配置影响。"
```c
$ sysctl -a | grep wmem  
net.ipv4.tcp_wmem = 4096 65536 8388608  
net.core.wmem_default = 212992  
net.core.wmem_max = 8388608
```
>"在net.ipv4.tcp_wmem"中的第一个值是发送缓存区的最小值，默认也是4K。当然了如果数据很大的话，该缓存区实际分配的也会比默认值大。"
![](attachments/Pasted%20image%2020230725174405.png)

## 解决方法
服务端百万连接达成记：
![](attachments/Pasted%20image%2020230725174526.png)

>“准备啥呢，还记得前面说过Linux对最大文件对象数量有限制，所以要想完成这个实验，得在用户级、系统级、进程级等位置把这个上限加大。我们实验目的是100W，这里都设置成110W，这个很重要！因为得保证做实验的时候其它基础命令例如ps，vi等是可用的。“

![](attachments/Pasted%20image%2020230725174622.png)
![](attachments/Pasted%20image%2020230725174639.png)

```c
活动连接数量确实达到了100W：
$ ss -n | grep ESTAB | wc -l    
1000024

当前机器内存总共是3.9GB，其中内核Slab占用了3.2GB之多。MemFree和Buffers加起来也只剩下100多MB了：
$ cat /proc/meminfo  
MemTotal:        3922956 kB  
MemFree:           96652 kB  
MemAvailable:       6448 kB  
Buffers:           44396 kB  
......  
Slab:          3241244KB kB

```
通过slabtop命令可以查看到densty、flip、sock_inode_cache、TCP四个内核对象都分别有100W个：
![](attachments/Pasted%20image%2020230725174819.png)


![](attachments/Pasted%20image%2020230725174902.png)
![](attachments/Pasted%20image%2020230725174908.png)

# 参考
```c
https://mp.weixin.qq.com/s/VFNvEw7-K8VevWyQG2Xesw
https://mp.weixin.qq.com/s/GBn94vdL4xUL80WYrGdUWQ
https://mp.weixin.qq.com/s/4lJ6Zt04AfmmuBrdHr90-w
```