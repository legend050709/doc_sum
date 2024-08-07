```table-of-contents
```
# 介绍
ldconfig命令的用途主要是在默认搜寻目录/lib和/usr/lib以及动态库配置文件/etc/ld.so.conf内所列的目录下，搜索出可共享的动态链接库（格式如lib_.so_）,进而创建出动态装入程序(ld.so)所需的连接和缓存文件。

# 应用场景
ldconfig通常在系统启动时运行，而当用户安装了一个新的动态链接库时，就需要手工运行这个命令。

(1) 往/lib和/usr/lib里面加东西，是不用修改/etc/ld.so.conf的，但是完了之后要调一下ldconfig，不然这个library会找不到。

(2) 想往上面两个目录以外加东西的时候，一定要修改/etc/ld.so.conf，然后再调用ldconfig，不然也会找不到。
```text
比如安装了一个mysql到/usr/local/mysql，mysql有一大堆library在/usr/local/mysql/lib下面，这时就需要在/etc/ld.so.conf下面加一行/usr/local/mysql/lib，保存过后ldconfig一下，新的library才能在程序运行时被找到。
```

(3) 如果想在这两个目录以外放lib，但是又不想在/etc/ld.so.conf中加东西（或者是没有权限加东西）。那也可以，就是export一个全局变量LD_LIBRARY_PATH，然后运行程序的时候就会去这个目录中找library。
> 一般来讲这只是一种临时的解决方案，在没有权限或临时需要的时候使用,因为这样的export 只对当前shell有效，当另开一个shell时候，又要重新设置，当然可以把export LD_LIBRARY_PATH=xxx 语句写到 ~/.bashrc中，这样就对当前用户有效了，写到/etc/bashrc中就对所有用户有效了。

# 查找路径
## 编译程序链接时动态库的查找路径
我们都知道，在编译成可执行文件前，链接器链接动态库也是需要查找动态库路径的，否则怎么链接上指定的动态库呢？那么这个顺序又是怎样的呢？
### 范例
```c
// test.c  
//来源：公众号【编程珠玑】  
#include <stdio.h>  
#include "test.h"  
#include "test1.h"  
void test()  
{  
    printf("I am test；hello，编程珠玑\n");  
    test1();  
}  
  
// test.h  
void test();  
  
  
//test1.c  
#include <stdio.h>  
#include "test1.h"  
void test1()  
{  
    printf("test1,needed by test\n");  
}  
// test1.h  
void test1();
```
分别制作动态库libtest.so和libtest1.so，这在后面的示例中会用到：
```c
$ gcc test1.c -fPIC -shared -o libtest1.so  
$ gcc test.c -fPIC -shared -o libtest.so -L. -ltest1
```
这样你在当前目录下就会看到有一个libtest.so和libtest1.so文件生成了，其中litest.so依赖libtest.so
注意，由于libtest.so依赖libtest1.so，这里用-L指定了要链接的test1的路径，因此我们看到：
```bash
$ ldd libtest.so  
	linux-vdso.so.1 (0x00007ffd1bbca000)  
	libtest1.so => not found  
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f9f1d0ae000)  
	/lib64/ld-linux-x86-64.so.2 (0x00007f9f1d6a1000)
```

### 分析

首先会查找的会是编译时链接的路径。修改前面的main.c，让它调用libtest.so中的test函数：
```c
// 来源：公众号【编程珠玑】  
#include <stdio.h>  
#include "test.h"  
int main()  
{  
    test(); // 调用libtest.so中的test函数  
    return 0;  
}
```

编译链接：
```bash
gcc -o main main.c -I ./ -L./ -ltest -ltest1
```
完美编译过。除此之外，如果我们把libtest.so和libtest1.so都移到/usr/lib下面，我们发现，即便不用-L也能编译过了:
```bash
gcc -o main main.c -I ./  -ltest -ltest1
```
这里需要说明的是，我们通过-L./来指定搜索库的路径，由于libtest.so依赖libtest1.so，因此在编译链接时，也需要链接上test1。

### 小结
从上面的内容可以看到，在链接时，我们通过-L参数搜索要链接的库路径。
但是这个路径信息不会写到ELF文件中，因此你会通过ldd命令看到`not found`，而通过`-rpath`可以指定链接时的搜索路径，这个信息会写入到ELF文件中，最终看到的结果是，由于libtest.so依赖libtest1.so，所以其他程序依赖libtest.so时，会自动从写入ELF的rpath中搜索它依赖的其他库，因此只需要链接libtest即可。

在制作库libtest1.so时，加上-rpath-link选项：
```bash
gcc test.c -fPIC -shared -o libtest.so -L. -ltest1 -Wl,-rpath-link $(pwd)
```

在编译`main`的时候，就不需要特意指定链接`libtest1.so`
```bash
gcc -o main main.c -L ./ -ltest
```
只需要链接`libtest.so`，其依赖的`libtest1.so`也链接进来了。

当然了，如果-L指定的路径没有呢,它还会去查找其他地方，否则依赖的系统库怎么找到呢？总结大致顺序如下：
- `-L`指定链接路径
- 对于依赖库中依赖的搜索顺序通过`-rpath-link`或`-rpath`选项查找（后面会提到）
- `gcc`默认链接路径（`gcc --print-search-dir | grep libraries` 查看）
- 链接器配置的查找路径（`ld --verbose | grep SEARCH_DIR`查看）

```bash
# gcc --print-search-dir | grep libraries
libraries: =/usr/lib/gcc/x86_64-redhat-linux/4.8.5/:/usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../../x86_64-redhat-linux/lib/x86_64-redhat-linux/4.8.5/:/usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../../x86_64-redhat-linux/lib/../lib64/:/usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../x86_64-redhat-linux/4.8.5/:/usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../../lib64/:/lib/x86_64-redhat-linux/4.8.5/:/lib/../lib64/:/usr/lib/x86_64-redhat-linux/4.8.5/:/usr/lib/../lib64/:/usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../../x86_64-redhat-linux/lib/:/usr/lib/gcc/x86_64-redhat-linux/4.8.5/../../../:/lib/:/usr/lib/

# ld --verbose | grep SEARCH_DIR
SEARCH_DIR("/usr/x86_64-redhat-linux/lib64"); SEARCH_DIR("/usr/lib64"); SEARCH_DIR("/usr/local/lib64"); SEARCH_DIR("/lib64"); SEARCH_DIR("/usr/x86_64-redhat-linux/lib"); SEARCH_DIR("/usr/local/lib"); SEARCH_DIR("/lib"); SEARCH_DIR("/usr/lib");
```

## 程序加载时对于so动态库的查找顺序
程序执行时按照下列顺序依次装载或者查找共享对象:  
0）`LD_PRELOAD`环境变量指定库路径
这种方式也最好仅仅只是用于测试，因为它的优先级非常高，并且影响全局。使用也很简单：
```bash
$ export LD_PRELOAD=./libtest.so
$ ./main

为了避免影响后面的验证，这里取消设置该环境变量：
unset LD_PREALOD
```
1）如果在编译时通过`-rpath`选项指定了路径，便会优先搜索这个路径
![](attachments/Pasted%20image%2020240305113518.png)
2）由环境变量 `LD_LIBRARY_PATH`指定的路径  
> 其中`LD_LIBRARY_PATH`是一个环境变量，当指定某个程序的`LD_LIBRARY_PATH`时  
动态链接器在查找共享库的时候，**会首先从指定的路径开始查找**；
3）由路径缓存文件`/etc/ld.so.cache`或者 `/etc/ld.so.conf`指定的路径  
4）默认共享目录 `/lib和/usr/lib`  


# LD_LIBRARY_PATH 环境变量

## 使用
### 临时生效
```bash
LD_LIBRARY_PATH=xxxx CMD
```
这样 LD_LIBRARY_PATH 只是在执行 CMD的时候生效，执行完成之后，就不在生效；


### 当前终端生效
```bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:xxxx
```
如上所示，设置环境变量的方式，这是在当前终端的后续命令的执行过程中生效。在其他的终端中执行命令其实是不生效的。

### 永久生效
即每个终端都生效。


### 其他
LD_PRELOAD环境变量 的使用同理。


# so动态库使用的常见问题
## cp命令拷贝一个新的so去覆盖旧的so时，如果有进程正在使用这个so，有可能导致该进程coredump

### 场景
出现问题的场景是升级，在升级流程的脚本中需要升级各个业务进程使用的so，但是有一个so文件是两个业务进程都在同时使用。比如有业务进程A、B、C，升级的过程是:A->B->C。其中有so是A和B都依赖，在升级A的过程中，先停掉A进程，升级其需要的so，这个时候升级so，使用的命令是cp，升级完so后，升级A进程使用的二进制，然后拉起A进程。在这一系列的过程中发现B进程coredump了，主要是没有考虑到B进程也在使用那个so。

注：此中的升级，应该指的是进程重启升级。即停止进程，替换so，重新拉起进程。
### 原因
**cp与mv/rm的区别：**

cp from to，则被覆盖文件 to的inode依旧不变（属性也不变），内容变为from的；

mv from to，则to的inode变为from的，相应的，to的属性也成了from的；rm类似


### 解决方法
方法一：  
先删除旧的so，然后再把新的so拷贝过去，即：  
rm oldlib.so 然后 cp newlib.so oldlib.so

方法二：  
mv oldlib.so oldlib.so_bak 然后 cp newlib.so oldlib.so

## 多个进程都在使用同一个so，但这个so的路径和版本均不同，那么使用ldconfig命令可能导致另一个进程出错

### 场景
与我们服务共同部署在同一个Linux服务器的其他服务也使用了zk服务，需要用到zk的动态链接库，我们的业务进程也需要用到zk的动态链接库。
本来最初相安无事，但是在执行一个脚本之后，发现他们的服务挂了，经定位发现是因为so使用有问题，用到了我们服务进程的路径下的zk的动态链接库。在那个shell脚本中，直接用了“ldconfig + 路径”的方式搜索指定路径的so，随后导致他们的服务链接到我们的zk动态链接库了，而这动态链接库是有区别的，最终导致他们的服务挂掉。

### 解决方法
方法一：检查使用ldconfig的地方，在多种服务共同使用的服务器上，不能直接用“ldconfig + 路径”的方式随意添加一些常用的so路径（诸如zk这样常用的服务），诸如多种服务共同部署的时候，要注意避免这种情况；如果要调用脚本使用，可以通过export LD_LIBRARY_PATH的方式临时添加。

方法二（推荐）：在编译的时候通过`gcc -rpath` 就指定动态库路径，这样就可以避免被其他路径下的不同的版本的so干扰。
> 运行程序时需要export `LD_LIBRARY_PATH`，不过使用它存在一些弊端，可能会影响到其它程序的运行。更好的是使用`gcc -rpath` 。



# 参考
```c
## 程序是如何找到动态库的？[守望]
https://www.yanbinghu.com/2021/03/13/33445.html
```