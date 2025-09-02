```table-of-contents
```
# 介绍
`debuginfo` 是包含**调试信息**的特殊文件（通常是 `.debug` 文件或打包在 `.rpm/.deb` 包中）。
这些信息在编译源代码时生成（使用 `-g` 编译器标志），但通常不包含在可执行文件、共享库或内核映像的`release版本`中；主要原因有：
**体积巨大**：调试信息（包含变量名、函数名、源代码行号、数据结构定义等）会使二进制文件体积膨胀数倍甚至数十倍。
**安全性和逆向工程**：这些信息会暴露程序的内部结构和逻辑，可能被恶意利用。
**运行时不需要**：程序在正常运行时不需要这些信息。

`debuginfo` 文件将调试信息从主二进制文件中分离出来，单独存储。
当使用调试器（如 `gdb`）或分析工具（如 `perf`, `systemtap`, `crash`）时，可以按需加载这些文件，提供丰富的调试和分析能力。

# 作用
`debuginfo` 是 Linux 系统中存储调试符号和源代码的映射信息的特殊软件包，用于在分析内核或用户态程序的崩溃转储文件（如 `vmcore`、`coredump`）时，提供机器码和源代码的映射关系。

## debuginfo 的作用
### 调试符号（Debug Symbols）
将二进制文件中的地址映射到源码中的函数、变量名。
允许调试工具（如 `crash`、`gdb`）显示有意义的堆栈跟踪和数据结构。
```bash
案例：
没有调试符号时，地址 `0xffffffff810e3d20` 显示为匿名函数；有调试符号时，显示为 `panic()` 函数。
```

### 源码关联
部分 `debuginfo` 包中包含与二进制对应的源代码片段，可直接在调试工具中查看代码上下文。

### 内核与用户态支持
- **内核级**：用于分析 `vmcore`（如 `kernel-debuginfo`）。
- **用户态**：用于分析应用程序的 `coredump`（如 `nginx-debuginfo`）。



# 分类
`debuginfo` 可以根据其来源和作用范围进行分类：
## 按层次分类
- **（1）应用程序 debuginfo：** 为用户空间应用程序及其依赖的特定库提供调试信息,  比如 `nginx` 的 `debuginfo`。
- **（2）系统库 debuginfo：** 为核心系统库提供调试信息，例如：
	- **glibc debuginfo：** 这是最重要的系统库之一。它包含 C 标准库（`libc.so.6`, `ld-linux.so`）、数学库（`libm.so.6`）、线程库（`libpthread.so.0`）等的调试符号。调试内存分配错误（`malloc`/`free`）、线程问题、系统调用封装等都需要它。
	- **其他核心库 debuginfo：** 如 `libgcc`, `libstdc++` (C++ 标准库), `openssl`, `zlib` 等。
- **（3）内核 debuginfo：** 为 Linux 内核本身（`vmlinux`）以及内核模块（`.ko` 文件）提供调试信息。这是调试内核崩溃（oops/panic）、分析内核性能、理解内核内部机制的关键。通常名为 `kernel-debuginfo` 或类似。


## 按格式分类（底层）

- **DWARF：** 这是 Linux 上最主流的调试信息格式。`gdb`, `perf`, `systemtap` 等工具都直接支持 DWARF。debuginfo 文件通常包含 DWARF 格式的数据。
- **CTF：** 紧凑型 C 类型格式，旨在比 DWARF 更小。在调试场景中不如 DWARF 普及，主要用于某些特定场景（如 BTF 用于 eBPF）。

# 安装和使用
## 安装
### 配置yum仓库
某些系统需手动启用 `debuginfo` 仓库：

![](attachments/Pasted%20image%2020250601134353.png)
```bash
sudo yum-config-manager --enable debuginfo
sudo yum clean all && sudo yum makecache
```

### 使用 `debuginfo-install`安装
```bash
# 安装 debuginfo-install 工具
（1）yum install yum-utils -y    # debuginfo-install 工具在此中

# 安装特定包的 debuginfo（包括依赖）
（2）debuginfo-install <package-name>
# 例如，安装正在运行的内核的 debuginfo 和 glibc 的 debuginfo
（2.1）debuginfo-install kernel-$(uname -r) glibc
```

### 验证安装成功
#### 检查内核符号文件
```bash
ls /usr/lib/debug/lib/modules/$(uname -r)/vmlinux
```
若存在此文件且大小合理（通常数百MB），则安装成功。

#### 查看软件包信息
```bash
# RPM 系
rpm -qa | grep debuginfo
```

#### 测试 crash 工具
启动 `crash` 并验证符号加载：
```bash
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /path/to/vmcore
crash> mod  # 应显示内核模块列表而非错误
```

### 安装后效果
####  `glibc`的`debuginfo`：gdb查看
 如果 `glibc`的`debuginfo` 包已正确安装，并且文件安装在标准路径 (`/usr/lib/debug`)，`gdb` 通常能自动找到并加载对应的 `debuginfo`。运行 `show debug-file-directory` 查看搜索路径。`gdb` 在调试用户程序时就能自动定位到 `glibc` 内部的函数（如 `malloc`, `free`, `pthread` 相关函数），显示有意义的堆栈回溯和参数信息。

##### 验证
使用 `info address <function>` 或 `info functions <regex>` 检查是否能看到函数名和地址。加载符号后，`bt` (backtrace) 会显示函数名和源代码行号。
安装 `glibc-debuginfo` 后，当程序执行到 `malloc`、`free` 或 `pthread_mutex_lock` 等内部函数时，`gdb` 就能显示这些函数的源码和参数了。


#### `kernel的debuginfo`：crash查看
`crash` 是专门用于分析 Linux **内核崩溃转储** (vmcore) 的工具。它强烈依赖 `kernel-debuginfo` 包中的 `vmlinux` 文件来理解内核数据结构、解析堆栈回溯等。启动时必须指定：`crash /path/to/vmlinux /path/to/vmcore`。

#### perf 查看
**记录性能数据：** 
`perf record -g <command>` 或 `perf record -g -p <pid>`。

**报告分析：** 
`perf report`。如果对应的 debuginfo 已安装，`perf` 就能将采样到的地址解析为**函数名**，并显示**调用图**。没有 debuginfo，就只能看到十六进制地址。

**内核符号：** 
`perf` 依赖内核的 `vmlinux` 文件来解析内核函数。确保 `kernel-debuginfo` 已安装且 `vmlinux` 位于标准路径。


## `glibc`的`debuginfo`
`glibc`及它的`debuginfo`包为：
```bash
[yunkai@fedora t]$ rpm -qa | grep glibc
glibc-2.18-12.fc20.x86_64
glibc-debuginfo-2.18-12.fc20.x86_64
...
```
我不禁有如下这些疑问：
```bash
- `glibc-debuginfo`中包含了什么信息？
- `glibc-debuginfo`是如何创建出来的？
- `gdb`或`systemtap`，是如何把`glibc源码`与`glibc-debuginfo`关联起来的？
```

### debuginfo中包含了什么信息？
让我们来看看`glibc-debuginfo`中，包含有什么内容：
```bash
[yunkai@fedora t]$ rpm -ql glibc-debuginfo-2.18-12.fc20.x86_64
/usr/lib/debug
/usr/lib/debug/.build-id
/usr/lib/debug/.build-id/00
/usr/lib/debug/.build-id/00/a32f1b9405f5fcd41a7618f3c2c895ee4aab09
/usr/lib/debug/.build-id/00/a32f1b9405f5fcd41a7618f3c2c895ee4aab09.debug
...

/usr/lib/debug/lib64/libthread_db.so.1.debug
/usr/lib/debug/lib64/libutil-2.18.so.debug
/usr/lib/debug/lib64/libc-2.18.so.debug
...

/usr/src/debug/glibc-2.18/wcsmbs/wcwidth.h
/usr/src/debug/glibc-2.18/wcsmbs/wmemchr.c
/usr/src/debug/glibc-2.18/wcsmbs/wmemcmp.c
...

```

由上可见，`glibc-debuginfo`大致有三类文件：
(1) 存放在`/usr/lib/debug/`下的：`.build-id/nn/nnn...nnn.debug`文件，文件名是`hash key`。
(2) 存放在`/usr/lib/debug/`下的其它`*.debug`文件，其文件名，是库文件名`+.debug`后缀。
(3) `glibc`的源代码

### 如何关联`glibc源码`与`glibc-debuginfo`
当使用`gdb`调试时，需要在机器码与源代码之间，建立起映射关系。这就需要三个信息：
**（1）机器码**：可执行文件、动态链接库，例如：`/lib64/libc-2.18.so`
**（2）源代码**：显然就是`glibc-debuginfo`中，包含的`*.c`和`*.h`等源文件。
**（3）映射关系**：你应该猜到了，它们就保存在`*.debug`文件中。

### debuginfo是如何创建出来的
#### 为什么需要debuginfo
当我们使用`gcc`的`-g`选项编译程序时，机器码与源代码的映射关系，会被默认地与可执行程序或者动态链接库合并在一起。
把映射关系等调试信息，与可执行程序或者动态链接库合并在一起，会带来一个显著的问题：可执行文件或库的Size变得很大。这对于那些不关心调试信息的普通用户，是不必要的。

例如，Linux的内核，如果带上`Debuginfo`，会无谓的增加几百M的大小。如果一个`Linux`操作系统的所有库都带上各自的`Debuginfo`，那么光是一个干净的操作系统，就需要浪费掉几G甚至十几G的磁盘空间。如果是通过网络安装，还将浪费所有用户的带宽，并显著的拖慢安装的进度。正是了为解决这个问题，在`Linux`上的各种程序和库，在生成`RPM`时，就已经把`Debuginfo`单独的抽取出来，因此形成了独立的`debuginfo`包。

#### 如何分离出debuginfo
问题是，如何让程序生成分离的`debuginfo`呢？我们可以通过`objcopy`命令的`--only-keep-debug`选项来实现，下面的命令把调试信息从`a.out`中读取出来，写到`a.out.debug`文件中：
```bash
gcc -g main.c -o a.out
objcopy --only-keep-debug ./a.out a.out.debug
```
既然已经把调试信息，保存到了`a.out.debug`文件中，就可以通过`objcopy`的`--strip-debug`选项给`a.out`瘦身了（也可以使用`strip --strip-debug ./a.out`，效果一样）：

当把调试信息从`a.out`中清除后，使用`gdb`对`a.out`进行调试，会报`no debugging symbols found`：
```bash
[yunkai@fedora t]$ gdb ./a.out
GNU gdb (GDB) Fedora 7.6.50.20130731-19.fc20
...
Reading symbols from /home/yunkai/t/a.out...(no debugging symbols found)...done.
(gdb) 
```
如上所示，显然，`gdb`找不到调试信息了。

### 用户态程序分析（结合 GDB）
```bash
# 加载程序
gdb -q /usr/bin/nginx /path/to/coredump

(gdb) set debug-file-directory /usr/lib/debug
# 查看堆栈
(gdb) bt
# 查看变量值
(gdb) p variable_name
```

## 内核级的`debuginfo`
### kernel-debuginfo 文件结构
典型的 `kernel-debuginfo` 包包含以下内容：
```bash
/usr/lib/debug/lib/modules/5.4.0-80-generic/
├── vmlinux               # 内核的未压缩符号文件
├── kernel/
│   └── core.ko.debug     # 内核模块的调试符号
└── drivers/
    └── nvidia.ko.debug   # 硬件驱动的调试符号
```

### 注意事项
#### 版本严格匹配
`debuginfo` 必须与内核或程序版本 **完全一致**，否则调试工具无法解析符号。
```bash
uname -r  # 内核版本
nginx -v   # 程序版本
```

#### 存储空间
内核 `debuginfo` 包通常较大（1-2 GB），安装前确保磁盘空间充足。



### 内核崩溃分析（结合 crash）
```bash
# 加载 debuginfo 和 vmcore
crash /usr/lib/debug/lib/modules/5.4.0-80-generic/vmlinux /var/crash/vmcore
 
# 查看崩溃时的调用栈
crash> bt
# 检查进程状态
crash> ps
# 反汇编代码
crash> dis panic
```

## 常见问题
### “No debugging symbols found” 错误
**原因**：
未安装 `debuginfo` 或路径未正确指定。

**解决**：
```bash
# 指定 debuginfo 路径（GDB）
(gdb) set debug-file-directory /usr/lib/debug
# 或启动 crash 时显式指定 vmlinux
crash /path/to/vmlinux /path/to/vmcore
```

### “CRC mismatch” 错误
**原因**：`debuginfo` 与内核版本不匹配。

**解决**：重新安装正确版本的 `debuginfo`。

### Repo ‘debuginfo’ not found
**原因**：未启用 `debuginfo` 仓库。

**解决**：参考前文配置仓库。


# 参考
```bash
# 深入理解debuginfo
https://blog.csdn.net/chinainvent/article/details/24129311

# debuginfo详解
https://blog.csdn.net/ygq13572549874/article/details/147719849
```