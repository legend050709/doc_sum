```table-of-contents
```
# 是什么
## chroot介绍
**chroot命令** 用来在指定的根目录下运行指令，即：`change root directory （更改 root 目录）`。**使用 chroot，可以更改当前正在运行的进程及其子进程的根目录**。在 linux 系统中，系统默认的目录结构都是以`/`，即是以根 (root) 开始的；
而在使用 chroot 之后，系统的目录结构将以指定的位置作为`/`位置；
在经过 chroot 命令之后，系统读取到的目录和文件将不在是旧系统根下的而是新根下（即被指定的新的位置）的目录结构和文件；*

![](attachments/Pasted%20image%2020240222111741.png)
**简单来说：**
**一个正在运行的进程经过 chroot 操作后，其根目录将被显式映射为某个指定目录，它将不能够对该指定目录之外的文件进行访问动作；**
这是一种非常简单的资源隔离化操作，类似于现在 Linux 的 Mount Namespace 功能；

## chroot监狱(jail)
chroot 监狱这个术语源自这样一个概念：chroot 环境内的进程及其子进程无法访问或查看基本文件系统，并且受限于chroot所预定的资源。

chroot 监狱（chroot jail）实质上是一个目录，其中包含了程序正常运行所需的所有资源、文件、二进制文件和其他依赖项。

然而，与常规的 Linux 环境不同，chroot监狱的环境受到严格限制，程序无法访问外部或额外的文件和系统资源。

例如，要在 chroot 监狱中运行 Bash shell，你需要将 Bash 二进制文件及其所有依赖项复制到chroot目录中。


# 为什么使用(作用)
`chroot` 的主要用途是锁定系统进程，以便这些进程中的任何安全漏洞都不会影响系统的其余部分。

- **增加系统的安全性，限制用户的权力；**
在经过 chroot 之后，在新根下将访问不到旧系统的根目录结构和文件，这样就增强了系统的安全性；
这个一般是在登录 (login) 前使用 chroot，以此达到用户不能访问一些特定的文件；
- **建立一个与原系统隔离的系统目录结构，方便用户的开发；**
使用 chroot 后，系统读取的是新根下的目录和文件，这是一个与原系统根下文件不相关的目录结构；
在这个新的环境中，可以用来测试软件的静态编译以及一些与系统不相关的独立开发；

# 怎样使用
## 语法
```bash
chroot [OPTION] NEWROOT [COMMAND [ARGS]...]

# 例如：
chroot /path/to/new/root command
# 或
chroot /path/to/new/root /path/to/server
# 或
chroot [options] /path/to/new/root /path/to/server

# 选项
--userspec=USER:GROUP  # 使用指定的 用户 和 组 (ID 或 名称)
--groups=G_LIST        # 指定补充组 g1,g2,..,gN 
--help     # 显示帮助并退出
--version  # 显示版本信息并退出
```
- 目录(dir)：指定新的根目录；
- 指令(command)：指定要执行的指令；

COMMAND 指的是切换 root 目录后需要执行的命令，如果没有指定，默认是 `${SHELL} -i`，大部分情况是 `/bin/bash`；

此外，执行 `chroot(8)` 需要使用 root 权限；

## 将进程发送到监狱
要在受限目录中打开 shell，您可以运行：
```bash
sudo chroot /jail
```

但是，此命令将因新创建的 `/jail` 目录而失败，因为 `chroot` 将尝试从 `/jail/bin/bash` 加载 bash。这个文件不存在，这是 `chroot` 的第一个问题——你必须自己构建 jail。

对于某些事情，使用 `cp` 复制它们就足够了：
```bash
cp -a /bin/bash /jail/bin/bash
```
但这只会复制 bash 可执行文件，而不是它的所有依赖项，这些依赖项还不存在于我们的监狱中。您可以使用 `ldd` 命令列出 bash 的依赖项：
```bash
ldd $(which bash)
	linux-vdso.so.1 (0x00007ffd079a1000)
	libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007f339096f000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f339076b000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f339037a000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f3390eb3000)
```
您可以手动复制它们：
```bash
cp /lib/x86_64-linux-gnu/libtinfo.so.5 /jail/lib/x86_64-linux-gnu/
cp /lib/x86_64-linux-gnu/libdl.so.2 /jail/lib/x86_64-linux-gnu/
cp /lib/x86_64-linux-gnu/libc.so.6 /jail/lib/x86_64-linux-gnu/
cp /lib64/ld-linux-x86-64.so.2 /jail/lib64/
```
但这对于您可能希望在 `chroot` 下运行的每个命令来说都是一个主要的麻烦。如果您不关心您的 `chroot` 访问您的实际 `lib` 和 `bin` 目录（无法访问系统的其余部分），那么您可以使用 `mount --bind` 在你的 jail 中提供一个链接：
```bash
mount --bind /bin /jail/bin
mount --bind /lib /jail/lib
mount --bind /lib64 /jail/lib64
```
您也可以只复制整个 `/bin` 和 `/lib` 目录，这会占用更多空间，但对于安全性来说可能更好一些，尤其是当您使用 `chroot` 以运行您不希望弄乱系统文件夹的不安全进程。

如果您正在通过 chroot bash 运行进程，您可以使用 `exit` 或 Control+D 退出 shell，这将停止正在运行的进程。在 jail 中运行的进程在它们自己的环境中运行，并且无法访问系统上的其他进程。

### 进程可以越狱吗？
 **任何需要 root 权限操作的程序(特权进程)，对其 chroot 是没意义的，因为通常 root 用户都能脱离 chroot；**

因此，特权进程可以轻松逃脱监狱。
非特权进程有可能通过方法 `chdir(\..\)` 和对 `chroot` 的另一个调用完全中断。如果您真的专注于安全性，您应该放弃对 `chroot(2)` 系统调用的访问，或者使用 fork `jchroot`，它会自动执行此额外的安全功能。

## chroot后挂载之前根目录下的某个目录
### 背景
chroot之后，之前的根目录下的目录，在chroot中就无法再访问到了。
但是如果期望再次访问之前的某个目录。


## 范例
创建对应的新的根目录：
```bash
$ J=$HOME/jail
$ mkdir -p $J
$ mkdir -p $J/{bin,lib/x86_64-linux-gnu,lib64,etc,var}
```

将几个必要的命令工具 copy 到 `bin/` 下：
```bash
sudo cp -vf /bin/{bash,ls} $J/bin
```

将步骤 2 中可执行命令依赖的动态库 copy 到 `jail/` 下：
```bash
$ list=`ldd /bin/ls | egrep -o '/lib.*\.[0-9]'`
$ for i in $list; do sudo cp -vf $i $J/$i; done

$ list=`ldd /bin/bash | egrep -o '/lib.*\.[0-9]'`
$ for i in $list; do sudo cp $i -vf $J/$i; done
```

执行 chroot 命令：
```bash
$ sudo chroot $J /bin/bash

bash-4.3# ls
bin  etc  lib  lib64  var
bash-4.3# cd /
bash-4.3# ls
bin  etc  lib  lib64  var
bash-4.3# cd ..
bash-4.3# ls
bin  etc  lib  lib64  var
```

可以看到无论我们如何改变目录，其根目录都被隔离在 `$J` 中；
**执行 `exit` 命令可退出这一环境；**


# 应用
## chroot搭建新的内核系统
### 目的
通过chroot，在一个Linux操作系统中安装另一个操作系统。
### 流程
chroot环境可以用来在一个完整的大文件系统上运行另外一个虚拟的文件系统。可使用它来实现许多功能，比如生成虚拟共享主机账号。每个用户的账号与一个chroot环境一一对应，用户可使用此chroot内安装的Linux发行版的整个文件系统，但是不能触及外层的大文件系统。（译者使用chroot调试程序）

1.新建一个chroot的目录，例如：
```bash
mkdir -p /var/jail/chroot
```

2.要搭建chroot环境，首先需要初始化rpm数据库
```bash
mkdir -p /var/jail/chroot/var/lib/rpm
rpm --rebuilddb --root=/var/jail/chroot
```

3.手动下载CentOS的发行包，使用rpm命令安装
```bash
wget http://mirror.centos.org/centos/6/os/i386/Packages/centos-release-6-0.el6.centos.5.i686.rpm (或者你想使用的任何版本)
rpm -i --root=/var/jail/chroot --nodeps centos-release-6-0.el6.centos.5.i686.rpm
```

4.使用YUM工具安装CentOS发行版的其余包到虚拟的chroot环境
```bash
yum --installroot=/var/jail/chroot install -y rpm-build yum
```

5.初始化chroot，尝试新系统
```bash
chroot /var/jail/chroot
```

如果一切正常，你已经有了一个相对简单的可运行的chroot环境。但是，如果你想实际使用此环境，还需要其它一些重要的文件系统必要组件，比如/proc和/dev.
```bash
proc文件加载脚本，判断proc文件是否已经加载，未加载调用mount：

mount -l | grep "/var/jail/chroot/proc" > /dev/null
if [ $? != 0 ]
then
   sudo mount -t proc chroot_proc /var/jail/chroot/proc/
fi
-----------

#存放驱动信息
sudo mount --bind /sys /mnt/debian10_amd64/sys

#存放设备节点信息
sudo mount --bind /dev /mnt/debian10_amd64/dev

#存放系统信息，例如内存信息，cpu信息等
sudo mount --bind /proc /mnt/debian10_amd64/proc
```

# 其他
## chroot(2)系统调用
### 介绍
`chroot()` 将调用进程及其子进程的根目录指定为 path；
```bash
#include <unistd.h>

int chroot(const char *path);
```

### 范例
```c
#include <stdio.h>
#include <error.h>
#include <unistd.h>
#include <stdlib.h>

char *const path = "/root/jail"; // 如上文实验所述目录
char *const argv[] = {"/bin/bash", NULL};

int main(void) {
    if (chroot(path) != 0) {
        perror("chroot error");
          exit(1);
    }
    chdir("/");                 // 忽略返回值
    execvp("/bin/bash", argv);  // 忽略返回值
    return 0;
}

编译和运行代码：
$ gcc test_chroot.c -o test_chroot

$ ./test_chroot # 非 root 用户执行命令
chroot error: Operation not permitted

$ sudo ./test_chroot
bash-4.3#
```
# 参考
```c
# Linux中的chroot命令
https://jasonkayzk.github.io/2021/06/26/Linux%E4%B8%AD%E7%9A%84chroot%E5%91%BD%E4%BB%A4/

https://zyy.rs/post/chroot-mechanism/
```