```table-of-contents
```
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

# 命令替换和组合
## 什么是命令替换
Linux中使用反引号（在波浪线的按键上）或者`$()`来执行命令替换。使用括号()来组合一系列命令。
```bash
[root@xuexi ~]# echo what date it is? $(date +%F)
what date it is? 2016-09-25

[root@xuexi tmp]# echo what date it is? `date +%F`  # 或者使用反引号
```

## 反引号`$()`两种命令替换对比
反引号和`$()`基本几乎等价，但尽量使用`$()`。
反引号有两点不方便之处：
(1)**命令替换嵌套**或者是包含引号的时候，反引号很麻烦，不如`$()`易读。
(2)反引号处理反斜线的转义规则比较不明确，但是`$()`中的反斜线会按正常的方式转义。


```bash
temp=$(seq $(ls -1 | wc -l))
 
echo $temp

说明：
$()支持嵌套，所以在$()内部的$()将先执行,如果是反引号的方式将无法支持如此复杂的嵌套语法；

执行过程：
内部的`ls -1 | wc -l` 会先执行;
接下来是seq命令的执行，参数则为上一步返回的标准输出，假定返回的是5；`$(seq 5)`
temp保存的是seq命令的标准输出（本身seq返回的是换行符组成的数字，保存到temp的时候变成了空格字符）
temp="1 2 3 4 5"
```

## 命令替换的特点
使用`$()`可以让括号里的命令提前于整个命令运行，然后将执行结果插入在命令替换符号处。
由于命令替换的结果经常交给外部命令，==不应该让结果有换行的行为，所以默认将所有的换行符替换为了空格(实际上所有的空白符都被压缩成了单个空格)==。

```bash
[root@xuexi ~]# echo -e "a\nb"
a
b

[root@xuexi ~]# echo `echo -e "a\nb\t   \tc"`
a b c

使用双引号引用可以保留空白符。
[root@xuexi ~]# echo "`echo -e "a\nb\t   \tc"`"
a
b               c
```

## 命令替换的流程
**命令替换分为两个过程**：
(1)开启子shell执行其中的命令
(2)将子shell中的输出结果打包插入在命令行中。但打包输出结果的过程是可以控制的(例如上面使用双引号)。

### 详细流程
子 Shell 的执行流程：

- 父 Shell 遇到 了命令替换`$(...)` 或 `` `...` ``。
    
- 父 Shell 使用 **`fork()`** 系统调用创建一个 **子 Shell 进程**。
    
- 子 Shell 进程执行括号内的 **`command`**。
    
- 子 Shell 捕获 `command` 的 **标准输出 (stdout)**。
    
- 子 Shell **退出**。
    
- 父 Shell 将子 Shell 捕获的输出作为命令替换的结果，并继续执行。



### 命令替换为什么需要子shell
子shell：即当前脚本/当前shell的子进程。

子 Shell（Sub-shell）是当前 Shell 进程的一个副本，它从父 Shell 继承了环境，但拥有独立的 **Shell 内部状态**。
- **环境隔离：** 子 Shell 会在执行完命令替换后立即退出，因此其对内部状态的任何修改都不会影响到父 Shell。
    - 例如，在命令替换中执行的 **`cd`** 命令只会改变子 Shell 的工作目录，而不会改变父 Shell 的工作目录。
    - 例如，在命令替换中设置的 Shell 变量（除非使用 `export` 且变量不是内部变量）在子 Shell 退出后就会消失。


### 命令替换中的外部命令

如果命令替换内部执行的是一个 **外部命令** (External Command)，例如 `$(ls -l)`：

- 子 Shell 进程 仍然会被创建。
    
- 这个 **子 Shell** 内部会再次使用 **`fork()` + `execve()`** 来执行外部程序 `/bin/ls`。
    

所以，在最常见的情况下，命令替换会涉及到 **一个子 Shell 进程**，并且如果其内部执行的是外部命令，还会再生成一个 **孙子进程** 来执行该外部命令。


### 范例


## 注意事项
命令替换，也称子命令替换，可以获取到命令的标准输出，注意：不能获取命令的标准错误。


## shell脚本中命令替换和外部命令的对比


# 内置命令和外部命令的区别
## 原理的区别
通常来说，内建命令会比外部命令执行得更快，执行外部命令时不但会触发磁盘 I/O，还需要 fork 出一个单独的进程来执行，执行完成后再退出。而执行内建命令相当于调用当前 Shell 进程的一个函数。

## 执行顺序

 **一个内建命令通常会与一个系统命令（外部命令）同名，但是Bash在内部重新实现了这些命令。比如，Bash的echo命令与/bin/echo就不尽相同，虽然它们的行为在绝大多数情况下都是一样的。**
 
查找的顺序是**内部命令->外部命令**。
shell命令解释器在执行命令时，先尝试按照内部命令来执行，如果要执行的命令不是内部命令，则按照外部命令去查找对应的执行文件所在的目录，并执行。当要执行的命令不是内部命令时（例如ls），如果有两个ls指令分别在不同的目录中（例如/usr/local/bin/ls和/bin/ls），shell命令解释器就根据PATH里面哪个目录先被查询到，则那个目录下的命令就先被执行。

# 参考
```bash

```