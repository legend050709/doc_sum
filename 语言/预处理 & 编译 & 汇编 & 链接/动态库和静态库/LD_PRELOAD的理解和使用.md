```table-of-contents
```
# 前言

LD_PRELOAD 是 Linux 系统中的一个环境变量，它可以影响程序的运行时的链接（Runtime linker），它允许你定义**在程序运行前优先加载的动态链接库**。
如果你是个 Web 狗，你肯定知道 LD_PRELOAD，并且网上关于 LD_PRELOAD 的文章基本都是绕过 disable_functions，都快被写烂了。

今天我们就从浅入深完整的学习一下什么是 LD_PRELOAD，LD_PRELOAD 有什么作用，我们可以如何利用 LD_PRELOAD。

## 什么是链接
程序的链接主要有以下三种：
- 静态链接：
在程序运行之前先将各个目标模块以及所需要的库函数链接成一个完整的可执行程序，之后不再拆开。

- 加载时动态链接（编译时指定了`-l`动态库，启动/`ldd`可执行文件时，自动加载动态库。）：
源程序编译后所得到的一组目标模块，在装入内存时，边装入边链接。

- 运行时动态链接（通过dlopen, dlclose 引入动态库中的函数）：
原程序编译后得到的目标模块，在程序执行过程中需要用到时才对它进行链接。

对于动态链接来说，需要一个动态链接库，其作用在于当动态库中的函数发生变化对于可执行程序来说时透明的，可执行程序无需重新编译，方便程序的发布/维护/更新。

# LD_PRELOAD 
## 介绍
LD_PRELOAD 是 Linux 系统中的一个环境变量，它可以影响程序的运行时的链接（Runtime linker），它允许你定义**在程序运行前优先加载的动态链接库**。

这个功能主要就是用来有选择性的载入不同动态链接库中的相同函数。
通过这个环境变量，我们可以在主程序和其动态链接库的中间加载别的动态链接库，甚至覆盖正常的函数库。
一方面，我们可以以此功能来使用自己的或是更好的函数（无需别人的源码），
而另一方面，我们也可以以向别人的程序注入程序，从而达到特定的目的。

```bash
LD_PRELOAD is an optional environmental variable containing one or more paths to shared libraries, or shared objects, that the loader will load before any other shared library including the C runtime library (libc.so) This is called preloading a library.
```
`LD_PRELOAD`用于指定提前加载一些动态库，这些动态库比`libc.so`等库装载更早，它们提供的函数能够屏蔽后加载的动态库中的函数。这个特性可以方便地用来截获库函数调用。

我们知道，Linux的用的都是`glibc`，有一个叫`libc.so.6`的文件，这是几乎所有Linux下命令的动态链接中，其中有标准C的各种函数。对于GCC而言，默认情况下，所编译的程序中对标准C函数的链接，都是通过动态链接方式来链接`libc.so.6`这个函数库的。

`loader`在进行动态链接的时候，会将有相同符号名的符号覆盖成`LD_PRELOAD`指定的so文件中的符号。换句话说，可以用我们自己的so库中的函数替换原来库里有的函数，从而达到`hook`的目的。

## 原理
Linux操作系统的动态链接库在加载过程中，动态链接器会先读取`LD_PRELOAD`环境变量和默认配置文件`/etc/ld.so.preload`，并将读取到的动态链接库文件进行预加载。
**即使程序不依赖这些动态链接库，`LD_PRELOAD`环境变量和`/etc/ld.so.preload`配置文件中指定的动态链接库依然会被加载，因为它们的优先级比`LD_LIBRARY_PATH`环境变量所定义的链接库查找路径的文件优先级要高**，所以能够提前于用户调用的动态库载入。

程序执行，加载动态库的查找顺序：
```bash
一般情况下，其加载顺序为:
LD_PRELOAD > LD_LIBRARY_PATH > /etc/ld.so.cache > /lib > /usr/lib
```

LD_PRELOAD的工作原理是：**当程序需要调用某个符号时，系统会先在程序自身的符号表中查找，如果找不到，则会在LD_PRELOAD指定的共享库中查找**。如果在LD_PRELOAD指定的共享库中找到了该符号，则使用该符号中的代码。

# 作用
`LD_PRELOAD`的主要功能就是用来有选择性的载入不同动态链接库中的相同函数。通过这个环境变量，我们可以在主程序和其动态链接库的中间加载别的动态链接库，甚至覆盖正常的函数库。一方面，我们可以以此功能来使用自己的或是更好的函数（无需别人的源码），而另一方面，我们也可以以向别人的程序注入恶意程序，从而达到那不可告人的罪恶的目的。

使用`LD_PRELOAD`可以实现一些特殊的功能，例如：

**动态库劫持**：
可以用LD_PRELOAD来劫持程序中的函数，替换为自己编写的函数，实现一些特殊的功能。
例如，可以用LD_PRELOAD来Hook系统调用，实现一些系统级别的监控或控制。

**程序调试**：
可以用LD_PRELOAD来替换程序中的函数，增加一些调试信息，例如，在程序中调用printf函数时，可以用LD_PRELOAD来替换为自己编写的函数，输出调试信息。

**库版本控制**：
可以用LD_PRELOAD来强制程序使用指定版本的共享库，以避免程序在不同版本的环境中产生兼容性问题。

**代码注入**
使用LD_PRELOAD可以向程序中注入一些代码，实现一些与程序无关的功能，例如统计程序使用情况、监控程序运行状态等。

## 动态库函数劫持
### 概念
动态库劫持是指通过替换程序中的动态链接库（共享库）中的函数，改变程序的行为的一种技术。在程序启动时，操作系统会在预定义的路径中查找所需要的共享库，加载到内存中，并建立程序和共享库之间的链接关系。在程序运行时，如果程序需要用到共享库中的某个函数，就会通过链接关系在共享库中查找该函数，并调用它。动态库劫持的技术就是通过替换共享库中的函数，来改变程序的行为。
### 实现原理
动态库劫持的实现原理是利用了Linux系统中的一个环境变量LD_PRELOAD。LD_PRELOAD是一个环境变量，可以用于在程序运行时动态加载指定的共享库。

### 步骤
动态库劫持的实现步骤如下：

1. 编写共享库中的替代函数，并编译成共享库。
2. 设置LD_PRELOAD环境变量，指定要加载的共享库。
3. 运行需要Hook的程序。

当程序需要调用被替换的函数时，会**先在程序自身的符号表中查找，如果找不到，则会在LD_PRELOAD指定的共享库中查找**。
如果在LD_PRELOAD指定的共享库中找到了该函数，则使用该函数中的代码，从而实现了动态库劫持。
# 使用
## 基本使用方法
```bash
（1）申明环境变量
gcc -shared -fPIC unrandom.c -o unrandom.so
export LD_PRELOAD=xxxx.so

`-shared`：是生成共享库格式, 即 so动态库，而不是.a静态库。
    
`-fPIC`：作用于编译阶段，告诉编译器产生与位置无关代码（Position-Independent Code）；这样一来，产生的代码中就没有绝对地址了，全部使用相对地址，所以代码可以被加载器加载到内存的任意位置，都可以正确的执行。这正是共享库所要求的，共享库被加载时，在内存的位置不是固定的。 

（2）取消申明
unset LD_PRELOAD
或者
export -n LD_PRELOAD=xxx.so

（3）启动程序
方式一：
先 export ，然后直接执行可执行文件。

比如：
export LD_PRELOAD=$PWD/unrandom.so
./random_num

方式二：
直接在执行可执行文件前，加上`LD_PRELOAD=xxxx.so`
比如：
LD_PRELOAD=$PWD/unrandom.so ./random_num

(4) ldd查看可执行文件

比如：
LD_PRELOAD=$PWD/unrandom.so ldd random_nums
```

Linux C中获取环境变量，以及取消环境变量。
```bash
#include <stdlib.h>
char *getenv(const char *name);
char *secure_getenv(const char *name);

int setenv(const char *name, const char *value, int overwrite);
int unsetenv(const char *name);
```
范例如下：
```c
    if(getenv("LD_PRELOAD") == NULL) 
	    return;      //getenv是获取环境变量 当他为空时我们已经不需要执行了
    
	unsetenv("LD_PRELOAD");                       
		//unsetenv删除环境变量的函数 调用一次就可以直接删除了
```

## 使用多个库
**可以在_LD_PRELOAD_变量**中指定多个库。为此，我们可以提供一个库列表，使用冒号或空格分隔：

```bash
$ LD_PRELOAD="/data/preload/lib/malloc_interpose.so:/data/preload/lib/free_interpose.so" ls -lh
malloc(20000)  call number: 223
malloc(32816)  call number: 226
total 2,8G
free((nil))  call number: 174
free((nil))  call number: 175
free((nil))  call number: 178
-rw-rw-r-- 1 blogdemo_user blogdemo_user 3,6K Jul  4 20:16 BVF_Density.ipynb
-rwxrwxr-x 1 blogdemo_user blogdemo_user 2,8G Jul  5 15:59 cuda_11.0.1_450.36.06_linux.run
drwxrwxr-x 2 blogdemo_user blogdemo_user   25 Jul  8 16:12 h5store
drwxrwxr-x 2 blogdemo_user blogdemo_user   30 Jul  5 18:47 pandas_processing
drwxrwxr-x 4 blogdemo_user blogdemo_user  142 Jul 19 15:59 preload
free((nil))  call number: 180
```

# 应用
我们可以通过`readelf`命令查看某个命令调用了哪些外部链接库，然后找到其中某个库，编写同名函数进行劫持，然后编译成共享对象文件，接着使用`LD_PRELOAD`环境变量指定生成的对象，达到命令执行的目的。
```bash
(1) readelf -s xxx

(2) 获取要hook的函数，man 该函数，查看参数以及返回值.

(3) 编写以及编译劫持动态库

```


## 误删基础库的应用

比如：误删 libc.so.6， 实际上 libc.so.6 是一个软链接。
```bash
# ll /lib64/libc.so.6
lrwxrwxrwx 1 root root 12 Jun 19  2018 /lib64/libc.so.6 -> libc-2.17.so
```

该软链接被删除之后，影响是，linux下的任何命令依赖于 libc库都无法使用，比如常见的 ls/cp/mv 等等；如下所示，此时无法恢复。

![](attachments/Pasted%20image%2020240620200829.png)


**解决方法**
（1）如果本机安装有 busybox 也可以。 因为 busybox 中的 工具是不依赖于动态库的，都是静态编译。通过busybox 的 ln 进行软链接。

（2）LD_PRELOAD 提前使用真实库中的符号，忽略掉了  libc.so.6 软链接。

```bash
LD_PRELOAD=/lib64/libc-2.17.so ln -s /lib64/libc-2.17.so /lib64/libc.so.6
```

# 优缺点
## 优点
LD_PRELOAD 的优势主要包括以下几个方面：

**灵活性**
使用LD_PRELOAD可以在运行时动态加载指定的共享库，灵活性非常高，可以根据需要替换程序中的任何函数。

**易用性**
使用LD_PRELOAD非常简单，只需要设置LD_PRELOAD环境变量即可，不需要修改程序本身的代码。

**可移植性**
使用LD_PRELOAD不依赖于具体的编程语言和操作系统，可以在不同的平台上使用。

**高效性**
使用LD_PRELOAD可以减少代码修改和重编译的时间，提高程序的开发效率。

## 缺点
### LD_PRELOAD只能hook动态链接库的函数
LD_PRELOAD有个限制：只能hook动态链接的库，对静态链接的库无效，因为静态链接的代码都写到可执行文件里了嘛，没有坑让你填。

### LD_PRELOAD的影响范围
可以使用`LD_PRELOAD`来劫持动态链接库并对目标进程注入一些修改。但是，这将影响整个进程，包括所有共享的目标文件。但是，可以通过在劫持的代码中实现条件来限制劫持的作用范围。

以下是示例代码，`LD_PRELOAD`库将仅影响主要可执行文件：
main.c文件
```c
#include <unistd.h> 

int main(void) 
{ 
    printf("This is the main program.\n"); 
    sleep(10); 
    printf("Exiting main program.\n"); 
    return 0; 
}
```
hijack.c文件
```c
#define _GNU_SOURCE 
#include <dlfcn.h> 
#include <stdio.h> 

typedef int (*orig_main_function_t)(void); 

int main(void) 
{ 
    printf("This is the hijack program.\n"); 

    orig_main_function_t orig_main_function = (orig_main_function_t)dlsym(RTLD_NEXT, "main"); 
    printf("Calling original main function...\n"); 
    int ret = orig_main_function(); 
    printf("Returned from original main function with status code %d\n", ret); 

    return 0; 
}
```
![](attachments/Pasted%20image%2020240312134020.png)

编译：
```c
$ gcc -o main main.c 
$ gcc -shared -o hijack.so -fPIC hijack.c -ldl 
```
运行：
```c
LD_PRELOAD=./hijack.so ./main

在此示例中，LD_PRELOAD仅影响main可执行文件，而不是共享的库。
```

#### 解决方法
可以将程序的启动放入到脚本中。
然后脚本中程序的启动的方式为：
```bash
LD_PRELOAD=xxxx.so ./cmd
```
这样 `LD_PRELOAD` 环境变量的作用范围，就有限了。仅限于这个程序。

### 编译器的优化
通过`LD_PRELOAD`的方式，尝试对`strcmp` 函数进行`hack`时，并不能`hack`成功。

通过查阅资料，原来编译器对很多函数进行了内联优化，并不会调用到 `so` 库中的函数，
因而通过优先加载自定义动态库的方式不可行。
不过，可以在编译测试程序时，添加 `-fno-builtin-strcmp`，关闭 `strcmp` 函数的优化 `gcc -o main main.c -fno-builtin-strcmp`。然后再以相同的方式运行测试程序： `LD_PRELOAD=./libmyhook.so ./main`


### 死循环问题
以`python`为例，通过命令`readelf -s /usr/bin/python`列出`python`程序调用的系统函数，可以筛选出`get`型的函数
![](attachments/Pasted%20image%2020240312145223.png)

尝试劫持`getpid()`函数
首先通过 man 命令查看`getpid()`函数的实现
![](attachments/Pasted%20image%2020240312145254.png)

然后重写`getpid()`函数
```c
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void){
    system("echo 'pwned by getpid!'");
    return 0;
}
```

**注意：**因为通过设置`preload`劫持了比较底层的函数，而派发出的新进程如果用到该函数也会一并被劫持，也就是说如果没有及时`unsetenv("LD_PRELOAD")`则会导致不断循环，一旦操作敏感就会比较危险，所以一定要及时删除这个环境变量。

改进版如下：
```c
#include <sys/types.h>
#include <unistd.h>

void payload(void){
    system("echo 'pwned by getpid!'");
}

pid_t getpid(void){
    if (getenv("LD_PRELOAD") == NULL){
        return 0;
    }

    unsetenv("LD_PRELOAD");
    payload();

    return 0;
}
```

接着编译共享对象，`-shared`表示生成共享库，`-fPIC`表示使用地址无关代码
```shell
gcc -shared -fPIC getpid.c -o getpid.so
```

`LD_PRELOAD`设置加载 so 文件，运行 python，可以看到函数被成功劫持
![](attachments/Pasted%20image%2020240312145453.png)



# 拓展
## 劫持动态库函数到劫持动态库

使用 `LD_PRELOAD` 进行动态库中的某个函数劫持，有一定限制。
因为需要确保该函数被目标程序调用。
GCC 有个 C 语言扩展修饰符 `__attribute__((constructor))`，可以让由它修饰的函数在 `main()` 之前执行，若它出现在动态链接库中，那么一旦动态链接库被系统加载，将立即执行` __attribute__((constructor))` 修饰的函数。
这样，我们就**不用局限于仅劫持某一函数，而应考虑劫持动态链接库了，也可以说是劫持了一个新进程**。

```string
__attribute__((constructor))
    constructor参数让系统执行main()函数之前调用函数(被__attribute__((constructor))修饰的函数)


__attribute__((destructor))
    destructor参数让系统在main()函数退出或者调用了exit()之后,(被__attribute__((destructor))修饰的函数)
```

当函数属性被设置为`constructor`时，该函数会在可执行文件（或共享对象）加载时被调用，同理当设置属性为`destructor`时会在对象`unload`时调用，也就是说设置为这两个属性时，会在`main()`函数执行之前或者`return()`执行之后被调用，我们就可以借助这个扩展修饰符，当加载 `so` 文件时自动执行恶意函数，这样就不局限于某个特定函数，使用面大大扩展了。

重新写一个函数，使用 `__attribute__((constructor))`修饰
```c
#include <unistd.h>

void payload(void){
    system("echo 'pwned!'");
}

__attribute__ ((__constructor__)) void exec(void){
    if (getenv("LD_PRELOAD") == NULL){
        return;
    }

    unsetenv("LD_PRELOAD");
    payload();

    return;
}
```
重新编译并加载，成功执行
![](attachments/Pasted%20image%2020240312150755.png)

## `LD_PRELOAD`失效的方法
在我们编程时，我们要随时警惕着`LD_PRELOAD`。不可否认，`LD_PRELOAD`是一个很难缠的问题。目前来说，要解决这个问题，只能想方设法让`LD_PRELOAD`失效。目前而言，有以下面两种方法可以让`LD_PRELOAD`失效。

1）通过静态链接。
使用`gcc`的`-static`参数可以把`libc.so.6`静态链入执行程序中。但这也就意味着你的程序不再支持动态链接。

2）通过设置执行文件的`setgid` / `setuid`标志。
在有`SUID`权限的执行文件，系统会忽略`LD_PRELOAD`环境变量。也就是说，如果你有以`root`方式运行的程序，最好设置上`SUID`权限。（如：`chmod 4755 daemon`）

# 范例
## malloc 库函数替换
例如，有一个已经编译好的程序使用malloc分配内存，你想使用Google开发的tcmalloc来提升效率，[使用LD_PRELOAD可以实现这个目的](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)。

```c
#include <stdio.h>  
#include <stdlib.h>  
int main(){  
    printf("start.\n");  
    malloc(10);  
    printf("end.\n");  
    return 0;  
}
```
首先，使用普通方法运行。
```bash
gcc test.c -o test  
./test
输出：
start.  
end.
```

接下来，`malloc.c`（如下所示），编译成动态链接库
```c
#include <stdio.h>  
typedef unsigned long size_t;  
void *malloc(size_t size){  
    printf("malloc(%ld)\n", size);  
    return NULL;  
}
```
编译，运行如下所示：
```
gcc -fPIC -shared malloc.c -o malloc.so  
LD_PRELOAD=./malloc.so ./test
输出：
start.  
malloc(10)  
end.
```

## 打开文件显示路径
编写`inspect_open.c`：
```c
#define _GNU_SOURCE
#include <dlfcn.h>
#include <stdio.h>


typedef int (*orig_open_f_type) (const char *pathname, int flags);
// open的钩子
int open(const char *pathname, int flags, ...)
{
    // remember to include stdio.h!
    printf("open():%s\n", pathname);
    /* Some evil injected code goes here. */
    orig_open_f_type orig_open;
    orig_open = (orig_open_f_type) dlsym(RTLD_NEXT, "open");
    return orig_open(pathname, flags);
}

// fopen的钩子
FILE *fopen(const char *path, const char *mode) {
    printf("fopen():%s\n", path);
    FILE* (*original_fopen) (const char*, const char*); // 定义函数指针变量；
    original_fopen = dlsym(RTLD_NEXT, "fopen");
    return (*original_fopen)(path, mode);
}

```
编译为动态库：
```bash
gcc -shared -fPIC inspect_open.c -o inspect_open.so -ldl
```
编写读取文件的例子`test.c`：
```c
#include<stdlib.h>
#include<stdio.h>

int main()
{
    FILE*fp;
    fp=fopen("file01.txt","r");
    if(fp==NULL)
    {
        printf("Can not openthe file!\n");
        exit(0);
    }
    fclose(fp);
    return 0;
}
```

```bash
编译：gcc test.c -o test

加载库运行：LD_PRELOAD=$PWD/inspect_open.so ./test
```

# 参考
```bash
# 掌握LD_PRELOAD轻松进行程序修改和优化的绝佳方法！
https://blog.csdn.net/Long_xu/article/details/128897509

利用 LD_PRELOAD 绕过 DISBALE_FUNCTIONS
https://kylingit.com/blog/%E5%88%A9%E7%94%A8ld_preload%E7%BB%95%E8%BF%87disbale_functions/
```