```table-of-contents
```
# 背景
大家应该都知道用第三方库，就少不了要使用到第三方的头文件和库文件。我们在编译、链接的时候，必须要指定这些头文件和库文件的位置。对于一个比较大第三方库，其头文件和库文件的数量是比较多的。如果我们一个个手动地写，那将是相当麻烦的。所以，pkg-config就产生了。

# 介绍
`pkg-config`是一个用来帮助我们添加编译时头文件和链接时库文件的标志的工具。
当我们在**开发使用某个库的应用程序时**，通常需要指定这个库的头文件和库文件的路径。`pkg-config`通过读取特定的`.pc`（Package Config）文件来提供这些信息。

# pkg-congfig的使用
## 查看
### 查看所有pkg-config管理的库
```bash
pkg-config --list-all
```
### 查看某个库是否被pkg-config管理
```bash
sudo yum install pkg-config # CentOS/RHEL
```

![](attachments/Pasted%20image%2020241206155932.png)

### 查看指定库的编译选项(头文件)
```bash
pkg-config --cflags libdpdk
```

### 查看指定库的链接选项(lib库)
```bash
pkg-config  --libs libdpdk
pkg-config --static --libs libdpdk
```
![](attachments/Pasted%20image%2020241206104615.png)

### 查看指定库的版本
```bash
# pkg-config --modversion libdpdk
22.11.0
```

### 查看库的特定变量的值
#### 库支持的变量
```bash
# pkg-config --print-variables libdpdk
includedir
libdir
pcfiledir
prefix

```

#### 查看库的lib文件的目录
要使用 `pkg-config` 获取库的路径，可以使用 `--variable=libdir` 选项。这个选项可以返回指定库的安装路径。

```bash
pkg-config --variable=libdir <package-name>
```

比如：
```bash

# pkg-config --variable=prefix libdpdk
/usr/local/lib/dpdklib-22.11

# pkg-config --variable=libdir libdpdk
/usr/local/lib/dpdklib-22.11/lib64


```

## 编译和链接
```bash
gcc $(pkg-config --cflags libfoo) -o myprogram myprogram.c $(pkg-config --libs libfoo)
```
在这里，`pkg-config --cflags libfoo`和`pkg-config --libs libfoo`分别会输出编译和链接应用程序时需要的标志。

# pkg-config的配置文件（.pc文件）

![](attachments/Pasted%20image%2020241206105026.png)

因为pkg-config也只是一个命令，所以不是你安装了一个第三方的库，pkg-config就能知道第三方库的头文件和库文件所在的位置。pkg-config命令是通过查询XXX.pc文件而知道这些的。
## 配置文件的组成
```bash
[root@localhost pkgconfig]# cat libevent.pc 
#libevent pkg-config source file

prefix=/usr/local
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include

Name: libevent
Description: libevent is an asynchronous notification event loop library
Version: 2.0.22-stable
Requires:
Conflicts:
Libs: -L${libdir} -levent
Libs.private: 
Cflags: -I${includedir}
```


### 基础字段
- `Name`: 库的名称。
- `Description`: 库的简短描述。
- `Version`: 库的版本号。
### 编译和链接字段

- `Cflags`: 编译时需要的选项，通常包括头文件路径。
- `Libs`: 链接时需要的选项，通常包括库文件路径。

### 其他字段
- Requires: 本库所依赖的其他库文件。所依赖的库文件的版本号可以通过使用如下比较操作符指定：=,<,>,<=,>=
- Requires.private: 本库所依赖的一些私有库文件，但是这些私有库文件并不需要暴露给应用程序。这些私有库文件的版本指定方式与Requires中描述的类似。
- Conflicts: 是一个可选字段，其主要用于描述与本package所冲突的其他package。版本号的描述也与Requires中的描述类似。本字段也可以取值为同一个package的多个不同版本实例。例如: Conflicts: bar < 1.2.3, bar >= 1.3.0

### 范例

```bash
Name: opencv
Description:OpenCV pc file
Version: 2.4
Cflags:-I/usr/local/include
Libs:-L/usr/local/lib –lxxx –lxxx
```

## 配置文件的查询路径
通常，`pkg-config`的配置文件（.pc文件）存放在`/usr/lib/pkgconfig`、`/usr/share/pkgconfig`或`/usr/local/lib/pkgconfig`等目录中。我们也可以通过设置`PKG_CONFIG_PATH`环境变量来指定额外的目录：

```bash
export PKG_CONFIG_PATH=/your/custom/path/pkgconfig
```

## 创建pc文件
在动态库开发中，可以为库创建一个`.pc`文件，并将其放入`pkgconfig`目录中。这样，其他开发者就可以通过`pkg-config`轻松地使用这个库。

一个简单的`libfoo.pc`文件可能如下：
```bash
prefix=/usr/local
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include

Name: libfoo
Description: The Foo library
Version: 1.0.0
Cflags: -I${includedir}/foo
Libs: -L${libdir} -lfoo
```
## 验证pc文件
```bash
# 验证示例
pkg-config --validate mylibrary.pc
```

# pkg-config的工作流程 
- **（1）查询库信息**: 当你执行`pkg-config`命令时，它首先会在预定义的目录（通常是`/usr/lib/pkgconfig/`或`/usr/share/pkgconfig/`）中查找与指定库相关的`.pc`文件。  
    
- **（2）读取`.pc`文件**: 找到`.pc`文件后，`pkg-config`会解析其中的字段，这些字段包括但不限于`Cflags`（编译选项）和`Libs`（链接选项）。  
    
- **（3）输出信息**: 根据你的命令选项（如`--cflags`或`--libs`），`pkg-config`会输出相应的信息，这些信息可以直接用于编译和链接。

# 环境变量
## PKG_CONFIG_PATH

环境变量PKG_CONFIG_PATH是用来设置.pc文件的搜索路径的，pkg-config按照设置路径的先后顺序进行搜索，直到找到指定的.pc 文件为止。这样，库的头文件的搜索路径的设置实际上就变成了对.pc文件搜索路径的设置。
**在安装完一个需要使用的库后，比如Glib，
一是将相应的.pc文件，如glib-2.0.pc拷贝到/usr/lib/pkgconfig目录下；
二是通过设置环境变量PKG_CONFIG_PATH添加glib-2.0.pc文件的搜索路径。**

添加环境变量PKG_CONFIG_PATH，在bash中应该进行如下设置：
```bash
$ export PKG_CONFIG_PATH=/opt/gtk/lib/pkgconfig:$PKG_CONFIG_PATH
```

# pkg-config与LD_LIBRARY_PATH
pkg-config与LD_LIBRARY_PATH在使用时有些类似，都可以帮助找到对应的库（静态库和共享库）。

## 区别
我们知道一个程序从源代码，然后编译连接，最后再执行这一基本过程。
### 工作阶段不同
这里我们列出pkg-config与LD_LIBRARY_PATH的主要工作阶段：
- pkg-config: 编译时、 链接时
- LD_LIBRARY_PATH: 链接时、 运行时

pkg-config主要是在编译时会用到其来查找对应的头文件、链接库等；而LD_LIBRARY_PATH环境变量则在 链接时 和 运行时 会用到。程序编译出来之后，在程序加载执行时也会通过LD_LIBRARY_PATH环境变量来查询所需要的库文件。

# 其他
## 库文件在程序执行时的搜索
库文件在链接（静态库和共享库）和运行（仅限于使用共享库的程序）时被使用，其搜索路径是在系统中进行设置的。
一般Linux系统把/lib和/usr/lib这两个目录作为默认的库搜索路径，所以使用这两个目录中的库时不需要进行设置搜索路径即可直接使用。对于处于默认库搜索路径之外的库，需要将库的位置添加到库的搜索路径之中。
### 运行时链接库的搜索顺序
Linux程序在运行时对动态链接库的搜索顺序如下：

1） 在编译目标代码时所传递的动态库搜索路径（注意，这里指的是通过`-Wl,rpath=<path1>:<path2>`或`-R`选项传递的运行时动态库搜索路径，而不是通过`-L`选项传递的）
```bash
# gcc -Wl,-rpath,/home/arc/test,-rpath,/lib/,-rpath,/usr/lib/,-rpath,/usr/local/lib test.c
或者
# gcc -Wl,-rpath=/home/arc/test:/lib/:/usr/lib/:/usr/local/lib test.c
```

2） 环境变量`LD_LIBRARY_PATH`指定的动态库搜索路径；

3） 配置文件_/etc/ld.so.conf_中所指定的动态库搜索路径(更改_/etc/ld.so.conf_之后，一定要执行命令ldconfig，该命令会将_/etc/ld.so.conf_文件中所有路径下的库载入内存）;

4） 默认的动态库搜索路径`/lib`；

5） 默认的动态库搜索路径`/usr/lib`;


### 设置库文件的搜索路径

处理前面说的，编译时通过 `-rpath` 设置运行时的搜索路径之外，
设置库文件的搜索路径有下列方式，可任选其中一种使用：
- 在环境变量LD_LIBRARY_PATH中指明库的搜索路径
- 在/etc/ld.so.conf文件中添加库的搜索路径

#### /etc/ld.so.conf文件中添加库的搜索路径
将自己可能存放库文件的路径都加入到/etc/ld.so.conf中是明智的选择。添加方法也及其简单，将库文件的绝对路径直接写进去就OK了，一行一个。比如：
```bash
/usr/X11R6/lib 
/usr/local/lib 
/opt/lib
```
#### 程序执行时查找库的缓存 /etc/ld.so.cache
在 /etc/ld.so.conf中 添加库的路径，对于程序链接时的库（包括共享库和静态库）的定位已经足够了。但是对于使用了程序的执行时查找库还是不够的。
这是因为为了加快程序执行时对共享库的定位速度，避免使用搜索路径查找共享库的低效率，所以是直接读取库列表文件/etc/ld.so.cache的方式从中进行搜索。

/etc/ld.so.cache是一个非文本的数据文件，不能直接编辑，它是根据/etc/ld.so.conf中设置的搜索路径由/sbin/ldconfig命令将这些搜索路径下的共享库文件集中在一起而生成的（ldconfig命令要以root权限执行）。

因此为了保证程序执行时对库的定位，==在/etc/ld.so.conf中进行了库搜索路径的设置之后，还必须要运行/sbin/ldconfig命令更新/etc/ld.so.cache文件之后才可以==。

#### ldconfig命令

ldconfig，简单的说，它的作用就是将/etc/ld.so.conf列出的路径下的库文件缓存到/etc/ld.so.cache以供使用。
因此当安装完一些库文件（例如刚安装好glib)，或者修改ld.so.conf增加新的库路径之后，需要运行一下/sbin/ldconfig使所有的库文件都被缓存到ld.so.cache中。如果没有这样做，即使库文件明明就在/usr/lib下的，也是不会被使用的，结果在编译过程中报错。

## 编译时与运行时动态库查找的比较

(1) 编译时查找的是静态库或动态库， 而运行时，查找的是动态库；

(2） 编译时可以用`-L`指定查找路径，或者用环境变量`LIBRARY_PATH`， 而运行时可以用`-Wl,rpath`或者`-R`选项，或者修改`/etc/ld.so.conf`，或者设置环境变量`LD_LIBRARY_PATH`;

> 说明： -Wl,rpath选项虽然是在编译时传递的，但是其实是工作在运行时。其本身其实也不算是gcc的一个选项，而是ld的选项，gcc只不过是一个包装器而已。我们可以执行man ld来进一步了解相关信息

(3) 编译时用的链接器是`ld`，而运行时用的链接器是`/lib/ld-linux.so.2`

(4) 编译时与运行时都会查找默认路径`/lib`、`/usr/lib`

 (5) 编译时还有一个默认路径`/usr/local/lib`，而运行时不会默认查找该路径；

## gcc使用-Wl,-rpath
### -Wl,-rpath
加上`-Wl,-rpath`选项的作用就是指定`程序运行时`的库搜索目录，是一个链接选项，生效于设置的环境变量之前(LD_LIBRARY_PATH)。

#### 范例
```c
// add.h
int add(int i, int j);
 
// add.c
#include "add.h"
 
int add(int i, int j)
{
    return i + j;
}
 
// main.c
#include <stdio.h>
#include <stdlib.h>
#include "add.h"
 
int main(int argc, char *argv[]) 
{
    printf("1 + 2 = %d\n", add(1, 2));
    return 0;
}


add.h和add.c用于生成一个so库，实现了一个简单的加法，main.c中引用共享库计算1 + 2：
```

编译共享库
```bash
gcc add.c -fPIC -shared -o libadd.so
# 编译主程序
gcc main.o -L. -ladd -o app
```

编译好后, 运行依赖库
```bash
# ldd app
linux-vdso.so.1 (0x00007ffeb23ab000)
libadd.so => not found
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007febb7dd0000)
/lib64/ld-linux-x86-64.so.2 (0x00007febb83d0000
# ./app
./app: error while loading shared libraries: libadd.so: cannot open shared object file: No such file or directory

```

可以看到， `libadd.so`这个库没有找到，程序也无法运行，要运行它必须要把当前目录添加到环境变量或者搜索路径中去。
但是如果在链接时加上`-Wl,rpath`选项之后：依赖库的查找路径就找到了，程序能正常运行。
```bash
# gcc -c -o main.o main.c
# gcc -Wl,-rpath=`pwd` main.o -L. -ladd -o app
# ldd app
linux-vdso.so.1 (0x00007fff8f4e3000)
libadd.so => /data/code/c/1-sys/solib/libadd.so (0x00007faef8428000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007faef8030000)
/lib64/ld-linux-x86-64.so.2 (0x00007faef8838000)
# ./app
1 + 2 = 3
```


下面我们再来看一下生成的可执行文件`app`，执行如下命令：可以看到是在编译后的程序中包含了库的搜索路径。
```bash
#  readelf app -d

Dynamic section at offset 0xe08 contains 26 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libadd.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000f (RPATH)              Library rpath: [/root/test]
 0x000000000000000c (INIT)               0x400578
 0x000000000000000d (FINI)               0x400784
 0x0000000000000019 (INIT_ARRAY)         0x600df0
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x600df8
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x400298
 0x0000000000000005 (STRTAB)             0x400408
 0x0000000000000006 (SYMTAB)             0x4002d0
 0x000000000000000a (STRSZ)              189 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x601000
 0x0000000000000002 (PLTRELSZ)           96 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x400518
 0x0000000000000007 (RELA)               0x400500
 0x0000000000000008 (RELASZ)             24 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffe (VERNEED)            0x4004e0
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x4004c6
 0x0000000000000000 (NULL)               0x0
```
### -Wl,rpath-link

# 参考
```bash
# 【Linux 库管理工具】深入解析pkg-config与CMake的集成与应用
https://developer.aliyun.com/article/1468226

# Linux中pkg-config的使用
https://ivanzz1001.github.io/records/post/linux/2017/09/08/linux-pkg-config#63-%E8%A1%A5%E5%85%85gcc%E4%BD%BF%E7%94%A8-wl-rpath
```