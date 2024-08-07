# rpm
# 查看命令的源码
##  查看二进制文件/命令的位置
```c
#whereis ls
/usr/bin/ls /usr/share/man/man1/ls.1.gz /usr/share/man/man1p/ls.1p.gz

```

## 查看对应的rpm包
```c
#rpm -qf /usr/bin/ls
coreutils-8.22-24.el7_9.2.x86_64
```
## 下载rpm包源码
- yumdownloaders 方式下载rpm包源码
```c
#yum install yum-utils

#yumdownloader --source coreutils-8.22-24.el7_9.2.x86_64
```

- 手动下载rpm包源码
```c
https://pkgs.org/search/?q=coreutils
```

如上两种方式下载之后，得到 带有源码的 rpm文件：ccoreutils-8.22-24.el7.src.rpm。

## 查询源码的文件
```c
#rpm -qpl ccoreutils-8.22-24.el7.src.rpm | grep tar

coreutils-8.22.tar.xz
```

## 提取源码文件
```c
#rpm2cpio coreutils-8.22-24.el7.src.rpm |cpio -idv coreutils-8.22.tar.xz

# mkdir -p new_dir; tar xvf coreutils-8.22.tar.xz -C new_dir

```


# 其他
## libxxx.rpm 和  libxxx-devel.rpm 和 libxxx.src.rpm的关系

![](attachments/Pasted%20image%2020240719115019.png)

![](attachments/Pasted%20image%2020240719115221.png)

libxxx.rpm 和  libxxx-devel.rpm 结合使用，提供给外部开发者。其中提供了 lib库(一般还会存在libxxx.pc 文件，供 pkgconfig 发现，比如：/usr/lib64/pkgconfig/libbpf.pc )，以及.h 头文件，不存在.c的文件。

libxxx.src.rpm 则是存在 .h、.c 以及 makefile 的文件，可以看到源码实现；但是需要编译成lib库 + 头文件的形式，提供给外部使用。

# 参考
```c
https://juejin.cn/post/6981019721197944845
```