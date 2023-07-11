# 概述
shell 识别三种基本命令：内建命令，shell 函数以及外部命令。

Linux系统为了提高系统运行效率，将经常使用的轻量的命令在系统启动时一并加载这些命令到内存中供Shell随时调用，这部分命令即为内部命令。

反之，系统层调用的较重的命令只有当被调用时才会硬盘加载的这部分命令即为外部命令。


# 内置命令

## 特点
内部命令嵌入在Shell程序中，并不单独以磁盘文件的形式存在于磁盘上。内部命令是写在bash源码里面的，其执行速度比外部命令快，因为解析内部命令shell不需要创建子进程，它们都运行在 Shell 进程当中。
## type判断某个命令是否为内置命令
```c
[root@localhost ~]# type cd  
cd is a Shell builtin  

[root@localhost ~]# type ifconfig  
ifconfig is /sbin/ifconfig
```
## 常见内置命令
SHELL中的内置命令约有60个，通过内置的`enable`命令即可查看所有的内部命令。
```c
[root@VM-8-8-centos ~]# enable
enable .
enable :
enable [
enable alias
enable bg
enable bind
enable break
enable builtin
enable caller
enable cd
enable command
enable compgen
enable complete
enable compopt
enable continue
enable declare
enable dirs
enable disown
enable echo
enable enable
enable eval
enable exec
enable exit
enable export
enable false
enable fc
enable fg
enable getopts
enable hash
enable help
enable history
enable jobs
enable kill
enable let
enable local
enable logout
enable mapfile
enable popd
enable printf
enable pushd
enable pwd
enable read
enable readarray
enable readonly
enable return
enable set
enable shift
enable shopt
enable source
enable suspend
enable test
enable times
enable trap
enable true
enable type
enable typeset
enable ulimit
enable umask
enable unalias
enable unset
enable wait
```

![](attachments/Pasted%20image%2020230727133455.png)
## 查看内置命令的源码
查看系统当前使用的shell：
![](attachments/Pasted%20image%2020230727133521.png)

查看bash的版本:
![](attachments/Pasted%20image%2020230727133534.png)

bash源码路径：[http://ftp.gnu.org/gnu/bash/](https://link.zhihu.com/?target=http%3A//ftp.gnu.org/gnu/bash/)，从该路径中下载指定版本下来。使用VS Code打开，源码均是使用C语言编写。
# 外部命令
## 执行原理
外部命令就是由Shell副本（新的进程）所执行的命令，基本的过程如下：
- a. 建立一个新的进程。此进程即为Shell的一个副本。
- b. 在新的进程里，在PATH变量内所列出的目录中，寻找特定的命令。  
    `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin:`为PATH变量典型的默认值。 当命令名称包含有斜杠（/）符号时，将略过路径查找步骤。
- c. 在新的进程里，以所找到的新程序取代执行中的Shell程序并执行。
- d. 程序完成后，最初的Shell会接着从终端读取下一条命令，和执行脚本里的下一条命令。



- 小结：
当外部命令被调用时，本质就是调用了另外一个程序，首先 Shell 会创建子进程，然后在子进程当中运行该程序。Shell程序管理外部命令执行的路径查找、加载存放，并控制命令的执行。

## 查看外部命令的源码
怎么查看外部命令的源码呢？方法如下：
1、查看命令工具所在绝对路径。
```c
比如：
# which ip
/sbin/ip
# rpm -qf /sbin/ip
iproute-4.11.0-25.el7_7.2.x86_64
```
2、搜索工具所属包。
3、下载工具源码包。
如果是GUN的软件包可以直接到GUN官网查找相关软件包：
[http://www.gnu.org/software/](https://link.zhihu.com/?target=http%3A//www.gnu.org/software/)

找到需要的软件包，点进去即可找到源码下载命令。

# 区别
## 原理的区别
通常来说，内建命令会比外部命令执行得更快，执行外部命令时不但会触发磁盘 I/O，还需要 fork 出一个单独的进程来执行，执行完成后再退出。而执行内建命令相当于调用当前 Shell 进程的一个函数。

## 执行顺序

 **一个内建命令通常会与一个系统命令（外部命令）同名，但是Bash在内部重新实现了这些命令。比如，Bash的echo命令与/bin/echo就不尽相同，虽然它们的行为在绝大多数情况下都是一样的。**
 
查找的顺序是**内部命令->外部命令**。
shell命令解释器在执行命令时，先尝试按照内部命令来执行，如果要执行的命令不是内部命令，则按照外部命令去查找对应的执行文件所在的目录，并执行。当要执行的命令不是内部命令时（例如ls），如果有两个ls指令分别在不同的目录中（例如/usr/local/bin/ls和/bin/ls），shell命令解释器就根据PATH里面哪个目录先被查询到，则那个目录下的命令就先被执行。