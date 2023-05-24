# 概述

我们可以发现命令Date后面有一个数字Date(1),这个数字1代表一般用户可使用的命令。查询命令后面的数字是有意义的。
![](attachments/Pasted%20image%2020230525120814.png)
这张表的内容可以用man  man的方式来取得。
![](attachments/Pasted%20image%2020230525121018.png)
```c
man手册9册内容的侧重点
section     名称                         说明
1           可执行程序或shell命令			用户可操作的命令 (用户命令)
2			系统调用						内核提供的函数（查头文件）
3			库调用						常用的函数库
4			特殊文件						在/dev下的设备文件
5			文件格式和约定				对一些文件进行解释，如passpd,就会说明这个文件中各个字段的含义
6			游戏程序						游戏程序
7			杂项							包括宏包和约定等
8			系统管理员使用的管理命令		通常只有系统管理员root可以使用（比如，yum, ip 命令， iptable命令）
9			内核相关						Linux内核相关文件		
n           内置命令                     内置命令，如cd
```

对于**内置命令**，还可以使用help命令查看帮助手册，例如：  
help cd

判断命令是什么类型可使用type命令，例如：  
type cd   
cd is a shell builtin


# 原则
**man对一个命令手册页的查找是从1开始的，默认依次查找每个部分，直到找到并显示。**
比如：对于像open,kill这种既有命令,又有系统调用的来说,man open则显示的是open(1),也就是从最前面的section开始,如果想查看open系统调用的话,就得man 2 open。
>使用篇章号的原因在于，有的命令、系统调用、库调用、文件等，可能存在重名的问题，这个时候就要加上数字，指定查看哪个命令。

比如 `passwd` 既可以表示设置密码的shell命令，又可以表示 `/etc/passwd` 文件
- **查看 `passwd` 命令，可以使用 `man passwd` 或 `man 1 passwd`。**
![](attachments/Pasted%20image%2020230525174151.png)
![](attachments/Pasted%20image%2020230525174248.png)

# 查看
## 参数
**man对一个命令手册页的查找是从1开始的，默认依次查找每个部分，直到找到并显示。**
但是，如果不知道命令的类型如何处理？
也就是，有时我们不知道，一个"命令"是属于shell命令、系统调用、库调用，还是文件、宏...；可以借助-a参数。
### man -f：查询命令的所有section
```c
       -f, --whatis
              Equivalent to whatis.  Display a short description from the manual page, if available. See whatis(1) for details.
              
       man -f smail
           Lookup the manual pages referenced by smail and print out the short descriptions of any found.  Equivalent to whatis -r smail.
```

```c
# man -f read
read (n)             - Read from a channel
read (3p)            - read from a file
read (2)             - read from a file descriptor
read (1p)            - read a line from standard input

# man -f passwd
sslpasswd (1ssl)     - compute password hashes
passwd (1)           - update user's authentication tokens
passwd (5)           - password file

# man -f ip
ip (8)               - show / manipulate routing, devices, policy routing and tunnels
ip (7)               - Linux IPv4 protocol implementation
```
### man -a：查询命令的所有section
>使用 `man -a command` 可以查看所有类型中包含该命令的帮助文档。这样，可以去查找是否属于自己想查看的命令帮助。

如下，查看`man -a passwd`，再列出一个帮助文档后，可以根据是否是自己需要的，决定查看下一个，跳过，还是退出！
![](attachments/Pasted%20image%2020230525174836.png)

### man -k/-K：查找包含关键字的手册
```c
       -k, --apropos
              Equivalent to apropos.  Search the short manual page descriptions for keywords and display any matches.  See apropos(1) for details.

       -K, --global-apropos
              Search for text in all manual pages.  This is a brute-force search, and is likely to take some time; if you can, you should specify a section to reduce the number of pages that need to be searched.  Search terms may be simple strings (the default),  or  regular
              expressions if the --regex option is used.
```

有时候我们需要查看包含某些关键字的手册，但是又不知道具体是那个手册，这个时候可以使用下面的方式：  
man -k touch  #查找包含touch关键字的手册（模糊匹配）

man -K XX 存在多个查找时，操作如下所示：
![](attachments/Pasted%20image%2020230525180853.png)

## 输出内容
man page大致分为一下部分：
NAME:   简单命令、数据名称说明
SYNOPSIS:   简短的命令语法（sysntax）简介
DESCRIPTION:    较为完整的说明，需要认真阅读
OPTION： 针对SYNOPSIS中列举的所有可用选项说明
COMMANDS：   当这个软件在执行的时候，可用在此软件中使用命令
FILES:  这个软件或数据所使用或参考或链接到的文件
SEE ALSE:   可以参考的，与这个命令有关的其他说明
EXAMPLE:    一些可以参考的范例，这个最好用
BUGS:   是否有相关的bug

![](attachments/Pasted%20image%2020230525181338.png)


## 未指定章节时查找顺序

简单说明一下manpath.config中的SECTION，它指定了优先输出的手册顺序。例如：

SECTION 1 n l 8 3 2 3posix 3pm 3perl 5 4 9 6 7  
这里它最先显示的是1，即shell命令的帮助手册，其次是n，即内置命令的帮助手册。以此类推。当然，前提是这些手册都有。


# 安装
```c
yum install -y man man-db man-pages
 
yum list installed |grep -i man
```

# 指定section章节查询
一般的命令或者库函数的帮助手册都很好查看，但是如果你想查看write函数的帮助手册，使用下面的命令是看不到的：  
man write

因为它既是一个用户命令也是一个系统调用名称，按照前面所设置的顺序，它会优先显示用户命令的帮助手册。因此，如果我们想直接查看作为系统调用的write的帮助手册，直接使用下面的方式即可：  
```c
man 2 write  #2表明从系统调用手册中查找
```

## man 1

man 1 COMMAND ： 主要是查看用户命令的使用。
```c
# man 1 read
```
![](attachments/Pasted%20image%2020230525181231.png)

![](attachments/Pasted%20image%2020230525201550.png)

## man 2
man 2 SYSCALL ： 主要是查看系统调用的使用。
![](attachments/Pasted%20image%2020230525181444.png)

## man 3
man 2 LIBCALL ： 主要是查看库函数的使用。
> 注：含有P标记，是posix的意思。

![](attachments/Pasted%20image%2020230525181824.png)

## man 5
查看文件的格式说明。

比如：
```c
# cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
...
```

man 5 passwd 如下所示：
![](attachments/Pasted%20image%2020230525182120.png)

## man 7
一些杂项说明，包括惯例与协议,例如Linux文件系统、网络协议（socket/四层/三层/二层， /proc内核参数）、ASCII code等的说明。
```c
比如：
man ASCII
man 7 arp
man 7 ip
man 7 ipv6
man 7 icmp
man 7 tcp
man 7 udp
man 7 socket


其他socket类型：
raw socket:  man 7 raw
unix socket：  man 7 unix
packet socket: man 7 packet
```
### ipv4协议
- ipv4 相关的数据结构
![](attachments/Pasted%20image%2020230525182510.png)

- ipv4 socket相关sockopt
![](attachments/Pasted%20image%2020230525182712.png)
> IP_PKTINFO 用在udp socket 获取目的地址。可以用在主机多ip情况下，udp socket的发送的sip和接收的dip相同的保证。

- ipv4 socket 的错误返回
![](attachments/Pasted%20image%2020230525183924.png)

- ipv4 的内核参数
![](attachments/Pasted%20image%2020230525182915.png)

- 其他
![](attachments/Pasted%20image%2020230525183156.png)

### tcp协议
- 内核参数
![](attachments/Pasted%20image%2020230525184229.png)
![](attachments/Pasted%20image%2020230525184512.png)

tcp相关的/proc下的内核参数：
![](attachments/Pasted%20image%2020230525184347.png)

- tcp socket option
![](attachments/Pasted%20image%2020230525184707.png)

- tcp socket的error
![](attachments/Pasted%20image%2020230525184844.png)

### udp 协议
- udp 内核参数
![](attachments/Pasted%20image%2020230525194243.png)

### socket
- 介绍
![](attachments/Pasted%20image%2020230525194746.png)
- socket相关内核参数
![](attachments/Pasted%20image%2020230525194507.png)

### 查看数据结构
比如 想要查看 sockaddr_in 数据结构的定义。

- 方法1：man 7 ip
该方法需要事先知道相关结构体在哪，以sockaddr_in为例
man 7 ip。

- 方法2：-k/-K查询关键字
![](attachments/Pasted%20image%2020230525195259.png)

最终在 man 7 ip中查询到。

- 方法3：google 查询
![](attachments/Pasted%20image%2020230525195500.png)


## man 8
man 8 COMMAND:  用来查看一些系统命令（root下才可以执行的系统命令）

### iptables 相关命令：man iptables
![](attachments/Pasted%20image%2020230525195800.png)
![](attachments/Pasted%20image%2020230525195931.png)
### ip 相关命令
![](attachments/Pasted%20image%2020230525200747.png)
![](attachments/Pasted%20image%2020230525200825.png)

```c
最后的 iproute2 : 表示 ip 命令属于哪个包。
如下所示：
# which ip
/sbin/ip

# rpm -qf /sbin/ip
iproute-4.11.0-25.el7_7.2.x86_64
```
#### ip address 命令: man ip address
```c
# man ip address
或者
# man ip-address
```
![](attachments/Pasted%20image%2020230525200115.png)

#### ip tunnel 命令: man ip tunnel
![](attachments/Pasted%20image%2020230525200301.png)

#### ip link 命令：man ip link
![](attachments/Pasted%20image%2020230525200356.png)

#### ip route 命令：man ip route
![](attachments/Pasted%20image%2020230525200553.png)

#### ip neigher 命令： man ip neighbour
![](attachments/Pasted%20image%2020230525200643.png)

#### ip netns命令：man ip netns
![](attachments/Pasted%20image%2020230525201119.png)

#### ip rule命令： man ip rule
![](attachments/Pasted%20image%2020230525201024.png)

# 查看内核参数
## kernel-doc方式查看
### 安装kernel-doc
```c
yum install kernel-doc
```
### 查看kernel-doc软件包的文件安装位置
安装好kernel-doc软件包后，可以使用下面的命令查看它将文档安装在哪里了。
```c
可以简单通过来查看：（可能存在多个安装路径）
 rpm -ql kernel-doc | head
rpm -ql kernel-doc | tail 

或者 
find / -type d -name "kernel-doc*"

```
注：
### 进入安装路径进行查找
- 对于内核参数 tcp_max_syn_backlog 的查看。
![](attachments/Pasted%20image%2020230525114810.png)
![](attachments/Pasted%20image%2020230525114951.png)


- 内核参数 rp_filter 的查看
![](attachments/Pasted%20image%2020230612110735.png)
![](attachments/Pasted%20image%2020230612111042.png)

## man 方式查看
- 查看 tcp_max_syn_backlog
知道其属于tcp 协议栈的参数。
直接 man 7 tcp, 然后查找 tcp_max_syn_backlog。
![](attachments/Pasted%20image%2020230525202118.png)

# kernel-doc 说明
## 背景
使用linux是躲不开的kernel，但是kernel的内容实在是太多了，随便一个子系统，都可以出好几本书。
期望在遇到问题之前，将kernel的知识全部了解，是不实际，需要找到一种方法，能够快速地找到与问题相关的知识。
百度和google都是不靠谱的，因为通过搜索只能得到零碎、不一定正确的内容，构建起知识体系，加注第一手资料的索引才是王道。

Kernel源码中包含到文档很丰富、收集了kernel各个方面的内容： [Linux Kernel Documentation](https://www.kernel.org/doc/Documentation/ "linux kernel documentation")，也可以到[Github: linux kernel](https://github.com/torvalds/linux "Github: linux kernel")中的documents目录下查看，前者更适合在线阅读。

Redhat的产品文档是特别优秀的资料：[Product Documentation for Red Hat Enterprise Linux](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/ "Product Documentation for Red Hat Enterprise Linux")。

Redhat的知识库也是很靠谱的：[redhat knowledgebase](https://access.redhat.com/search/#/knowledgebase "redhat knowledgebase")。
## 00_INDEX
每个目录下通常都有一个名为`00_INDEX`的文件，对当前目录下的所有文件和和子目录做了说明，例如根目录下[Documentation 00-INDEX](https://www.kernel.org/doc/Documentation/00-INDEX "Documentation 00-INDEX")。
## sysctl
内核有非常多可调整参数，它们在/proc和/sys目录中以文件的形式暴露出来。

Kernnel的[Documentation/sysctl](https://www.kernel.org/doc/Documentation/sysctl/ "Documentation/sysctl")目录汇聚了所有内核参数。

## memory
[Documentation/vm](https://www.kernel.org/doc/Documentation/vm/ "Documentation/vm")
[Understanding the Linux Virtual Memory Manager](https://www.kernel.org/doc/gorman/html/understand/ "Understanding the Linux Virtual Memory Manager")
[Overview of Linux Memory Management Concepts: Slabs](http://www.secretmango.com/jimb/Whitepapers/slabs/slab.html "Overview of Linux Memory Management Concepts: Slabs")
可以用命令`slabtop`查看kernel memory的使用情况，或者直接查看`/proc/slabinfo`。

### numa
```c
linux-3.2.12\Documentation\vm\numa
linux-3.2.12\Documentation\vm\numa_memory_policy.txt
linux-3.2.12\Documentation\vm\page_migration
```

### Hugepages
```c
linux-3.2.12\Documentation\vm\hugetlbpage.txt
```
使用hugepages的目的是减少内存页面总数, 提高TLB缓存的命中率。
《深入理解计算机系统结构》中对内存的地址转换有非常详细的说明。
进程见到的内存地址只是内存的虚拟地址，不是真正的物理内存的地址，因此在访问内存的 内容时，需要将内存的虚拟地址转换成实际的物理地址。而内存的虚拟地址和物理地址的对 应关系是通过页表结构管理的。
为了提高地址转换的效率, CPU中存在TLB页表项缓存，但是缓存的大小是非常小的。因此如 果减少页表项的数目, 可以延长TLB中缓存的页表项的刷新周期，从而提高命中率。

## networking
网络相关的文档位于[Documentation/networking](https://www.kernel.org/doc/Documentation/networking/ "Documentation/networking")中。

# 参考
```c
https://juejin.cn/post/7023171019099602975
https://blog.csdn.net/legend050709/article/details/108793953
https://www.lijiaocn.com/%E6%96%B9%E6%B3%95/2017/11/13/howto-linux-kernel-doc.html
```