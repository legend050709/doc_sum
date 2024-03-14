```table-of-contents
```
# Linux下so动态链接库介绍
## 什么是库
库从本质上来说是一种可执行代码的二进制格式，可以被载入内存中执行。库分静态库和动态库两种。

## 静态库
静态库：这类库的名字一般是libxxx.a，xxx为库的名字。利用静态函数库编译成的文件比较大，因为整个函数库的所有数据都会被整合进目标代码中，他的优点就显而易见了，即编译后的执行程序不需要外部的函数库支持，因为所有使用的函数都已经被编译进去了。当然这也会成为他的缺点，因为如果静态函数库改变了，那么你的程序必须重新编译。

## 动态库
Linux下动态库文件的文件名形如 **libxxx.so**，其中so是 **Shared Object** 的缩写，即可以共享的目标文件。

动态库：这类库的名字一般是libxxx.M.N.so，同样的xxx为库的名字，M是库的主版本号，N是库的副版本号。当然也可以不要版本号，但名字必须有。

相对于静态函数库，动态函数库在编译的时候并没有被编译进目标代码中，**你的程序执行到相关函数时才调用该函数库里的相应函数**，因此动态函数库所产生的可执行文件比较小。
由于函数库没有被整合进你的程序，而是程序运行时动态的申请并调用，所以程序的运行环境中必须提供相应的库。动态函数库的改变并不影响你的程序，所以动态函数库的升级比较方便。linux系统有几个重要的目录存放相应的函数库，如/lib和/usr/lib。



# 动态库的使用
共享文件（`*.so`）也称为动态库文件，它包含了代码和数据（这些数据是在连接时候被连接器ld和运行时动态连接器使用的）。动态连接器可能称为`ld.so.1`，`libc.so.1`或者 `ld-linux.so.1`。我的`CentOS6.0`系统中该文件为：`/lib/ld-2.12.so`

在链接动态库生成可执行文件时，并不会把动态库的代码复制到执行文件中，而是在执行文件中记录对动态库的引用。  
程序执行时，再去加载动态库文件。如果动态库已经加载，则不必重复加载，从而能节省内存空间。

**程序动态链接的优点**是

1. 减少依赖相同动态库的多个进程同时运行时的内存的占用（不用每一个进程都加载一份动态库
2. 可扩展性：在程序不用重启的情况下，动态的加载所需要的动态库，可实现对程序的扩展
3. 程序版本更新与动态链接库的分离。

## 查看程序依赖的动态库
基本上每一个linux 程序都会使用动态库，查看某个程序使用了那些动态库，可以使用ldd命令：
```bash
# ldd /bin/ls  
linux-vdso.so.1 => (0x00007fff597ff000)  
libselinux.so.1 => /lib64/libselinux.so.1 (0x00000036c2e00000)  
librt.so.1 => /lib64/librt.so.1 (0x00000036c2200000)  
libcap.so.2 => /lib64/libcap.so.2 (0x00000036c4a00000)  
libacl.so.1 => /lib64/libacl.so.1 (0x00000036d0600000)  
libc.so.6 => /lib64/libc.so.6 (0x00000036c1200000)  
libdl.so.2 => /lib64/libdl.so.2 (0x00000036c1600000)  
/lib64/ld-linux-x86-64.so.2 (0x00000036c0e00000)  
libpthread.so.0 => /lib64/libpthread.so.0 (0x00000036c1a00000)  
libattr.so.1 => /lib64/libattr.so.1 (0x00000036cf600000)
```

注意：使用`dlopen`加载的动态库通过`ldd`无法查看到。

## 编译生成动态库
test.c 文件如下
```c
#include <stdio.h>
#include <unistd.h>
#include "test.h"

void test()
{
    printf("I am test；hello，编程珠玑\n");
}

void test1(void) {
    printf("This is do test1\n");
    sleep(10);
    printf("End of test1\n");
}
 
void test2(void) {
    printf("This is do test2\n");
    sleep(10);
    printf("End of test2\n");
}
```

test.h代码如下：
```bash
#include <stdio.h>  
void test1(void);
void test2(void);
void test(void);
```

```bash
gcc -fPIC -shared -o libtest.so test.c

参数含义：  
-fPIC/-fpic: Compiler directive to output position independent code, a characteristic required by shared libraries. 创建共享库必须的参数  

-shared: Produce a shared object which can then be linked with other objects to form an executable.
```

## 查看动态库中的信息
```
readelf
	--dyn-syms：显示文件动态符号表中的入口。
	-d：显示文件动态段内容。
```
## 编译程序时链接动态库
```bash
#include <stdio.h>
#include "test.h" 

int main()
{
    test1();
    test2(); 
    return 0;
}
```

```bash
动态库链接： 生成二进制程序main2
gcc -o main2 main2.c -L . -ltest 

参数含义：  
-L 指定动态库目录为当前目录  
-l 指定动态库名test，不要写libtest.so
```

可以通过ldd命令查看其依赖的动态库：
```bash
$ ldd main2  
	linux-vdso.so.1 =>  (0x00007ffd57757000)  
	libtest.so => not found  
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f84c13f6000)  
	/lib64/ld-linux-x86-64.so.2 (0x00007f84c17c0000)
```

## 运行时动态库的查找
运行：
```bash
$ ./main  
./main: error while loading shared libraries: libtest.so: cannot open shared object file: No such file or directory
```

它并不能找到这个动态库，因为它会默认从系统库的路径去查找这个库，但是我们并没有把这个库放到系统路径下，因此会找不到了。

程序执行，加载动态库的查找顺序：
一般情况下，其加载顺序为:
`LD_PRELOAD > rpath > LD_LIBRARY_PATH > /etc/ld.so.cache > /lib > /usr/lib`
```bash
动态库的搜索顺序如下：
- LD_PRELOAD环境变量指定库路径
- -rpath链接时指定路径
- LD_LIBRARY_PATH环境变量设置路径
- /etc/ld.so.conf配置文件指定路径
- 默认共享库路径，/usr/lib，lib
```


`rpath`是在编译可执行文件时指定的运行时库要搜索的路径。查看可执行ELF文件的，`rpath`配置，如下所示：
![](attachments/Pasted%20image%2020240314112628.png)

> 注：通过`strace`可以看到程序加载了哪些动态库文件，可以看到程序通过`dlopen`加载的动态库文件。

# 动态库的加载时机
在Linux下，**动态库（也称为共享库）的加载是在程序运行时进行的。当程序启动时，并不会立即加载所有的动态库，而是在需要使用到库中的函数时才会进行加载**。

当程序启动时，操作系统会加载可执行文件（二进制文件），并将其映射到进程的地址空间中。因此，程序启动时，并没有加载任何动态库。只有当程序在运行时需要调用动态库中的函数时，动态链接器才会根据需要动态地加载相应的动态库，并将其映射到进程的地址空间中。

这种动态加载的机制使得程序只在需要时才加载动态库，从而**减小了程序的启动时间和内存占用**。这也为程序提供了更大的灵活性，可以根据需要加载不同的动态库，实现动态的功能扩展和插件机制。

## 动态库的链接流程
动态库的链接以及加载，是在程序运行时进行的，而不是在程序启动时。**链接是指将程序中的函数调用与实际的函数地址进行关联，以便程序能够正确地调用动态库中的函数**。

动态链接器在程序运行时负责完成动态库的链接工作，它会进行以下步骤：
1. 符号解析：当程序中出现对动态库中函数的调用时，动态链接器首先会解析函数名，确定需要调用的函数。
    
2. 动态库加载：动态链接器根据一定的搜索顺序，如之前所述的顺序，开始搜索并加载需要的动态库。加载动态库是将库文件映射到进程的虚拟地址空间中。
    
3. 符号绑定：一旦动态库被加载到内存中，动态链接器会解析库中的符号表，将函数名与实际的函数地址进行绑定，以建立函数调用的关联关系。
    
4. 重定位（如果需要）：在某些情况下，动态库的加载可能涉及到地址重定位的过程。这是因为库文件中的函数和全局变量的地址可能与程序的地址空间不匹配，需要进行调整以确保正确的访问。

总结起来，动态库的链接是在程序运行时进行的，动态链接器负责根据程序的需要，在运行时加载动态库并解析符号，将函数调用与实际的函数地址进行绑定。这种动态链接的机制使得程序能够在运行时动态地加载和链接不同的动态库，提供了灵活性和可扩展性。

# 动态库的调用

在Linux平台下，动态库的调用分为显式和隐式两种方式。

## 隐式调用
隐式调用又叫隐式加载或载入时加载。

通过在编译时设置包含动态库的头文件（`-Idir... -include file`），并指明动态库的位置和名称（`-Ldir... -lnamespec(libnamespec.so)`）。

隐式调用由系统完成加载，对编程者透明。隐式调用必须包含动态库中的头文件。


## 显式调用详解
在运行时加载，在程序运行过程中通过`dlopen`指定动态链接库文件，然后通过`dlsym`获取动态库里函数的地址。

相比隐式调用，显式调用需要在程序中实现，写明要加载的动态库以及调用函数的名称。

注意：使用`dlopen`链接并不是在进程启动时候加载映射的，而是当进程运行到使用`dlopen`的地方才加载。因此如果一个程序进行显式调用，但并没有附带显式调用加载的动态库文件，该程序仍能正常启动。而且当运行逻辑没有触发调用`dlopen`的代码时，即使缺少动态库文件程序仍然可以正常运行。

### 作用
显式调用时，由编程者实现进行加载并负责卸载。这样好比对动态申请内存空间，需要使用时申请，不使用时即可释放。因此显式调用内存使用更为合理，大型项目应使用显式调用。

### 函数原型
```c
#include <dlfcn.h>
void *dlopen(const char *filename, int flags);
void *dlsym(void *handle, const char *symbol);
int dlclose(void *handle);
char *dlerror(void);

```
#### `dlopen`
函数原型：`void *dlopen(const char *libname,int flag)`;
功能描述：`dlopen`必须在`dlerror`，`dlsym`和`dlclose`之前调用，表示要将库装载到内存，准备使用。
如果要装载的库依赖于其它库，必须首先装载依赖库。如果`dlopen`操作失败，返回`NULL`值；如果库已经被装载过，则`dlopen`会返回同样的句柄。
参数中的`libname`一般是库的全路径，这样`dlopen`会直接装载该文件；如果只是指定了库名称，在`dlopen`会按照下面的机制去搜寻：
a.根据环境变量`LD_LIBRARY_PATH`查找
b.根据`/etc/ld.so.cache`查找
c.查找依次在`/lib`和`/usr/lib`目录查找。
flag参数：表示处理未定义函数的方式，可以使用`RTLD_LAZY`或`RTLD_NOW`。
`RTLD_LAZY`表示暂时不去处理未定义函数，先把库装载到内 存，等用到没定义的函数再说；`RTLD_NOW`表示马上检查是否存在未定义的函数，若存在，则`dlopen`以失败告终。


#### `dlsym`
用于从动态库中查找需要使用的函数；
函数原型：`void *dlsym(void *handle,const char *symbol)`;  
1)handle：表示具体的句柄信息，也就是dlopen函数的返回值；
2)symbol：字符串形式的符号/标识符，通常指“函数名”；

功能描述：在`dlopen`之后，库被装载到内存。`dlsym`可以获得指定函数(`symbol`)在内存中的位置(指针)。  
如果找不到指定函数，则`dlsym`会返回`NULL`值。但判断函数是否存在最好的方法是使用`dlerror`函数，

![](attachments/Pasted%20image%2020240312134006.png)

#### `dlclose`
用于卸载已加载的动态库；
函数原型：`int dlclose(void *)`;  
功能描述：将已经装载的库句柄减一，如果句柄减至零，则该库会被卸载。如果存在析构函数，则在`dlclose`之后，析构函数会被调用。

#### `dlerror`
函数原型：`char *dlerror(void)`;  
功能描述：`dlerror`可以获得最近一次`dlopen`,`dlsym`或`dlclose`操作的错误信息，返回`NULL`表示无错误。`dlerror`在返回错误信息的同时，也会清除错误信息。

### 显示加载动态库的流程

显示加载动态库，可以大致分为以下几个步骤：
- 使用dlopen打开动态库
- 使用dlsym找到需要使用的符号
- 调用动态库中的函数
- dlopen关闭（卸载）动态库
```
注意：
显示加载动态链接库的时候，在编译动态库时，需要添加`-dl`：


比如：
gcc who.c -o who.so -fPIC -shared -ldl -D_GNU_SOURCE

-ldl： 显示加载（dynamic load）动态库，可能会调用dlopen、dlsym、dlclose、dlerror.
-D_GNU_SOURCE： 以GNU规范标准编译，如果不加上这个参数会报RTLD_NEXT未定义的错误
```

这种方式有以下好处：
- 编译时无需链接需要的动态库，我们注意到第二种方式编译时没有加-ltest
- 如果程序的某些场景不需要动态库的函数，那么它就不会去加载该动态库

### 范例
```c
#include<stdio.h>
#include <dlfcn.h>
#include"test.h"
int main(void)
{
    printf("start to call test\n");
    char *error = NULL;
    /*打开动态库*/
    void *handle = dlopen("libtest.so",RTLD_LAZY);
    if(NULL == handle)
    {
        error = dlerror();
        printf("open error:%s\n",error);
        return -1;
    }
    /*返回类型为函数指针*/
    void (*fun)() = dlsym(handle,"test1");
    if(NULL == fun)
    {
        error = dlerror();
        printf("open error:%s\n",error);
        dlclose(handle);
        return -1;
    }
    /*调用函数*/
    (*fun)();
    
    /*关闭*/
    dlclose(handle);
    printf("end to call test\n");
    return 0;
}
```

编译以及运行
```
$ gcc  main.c -ldl -L . -o main  #需要链接libdl.so库
-ldl： 显示方式加载动态库，可能会调用dlopen、dlsym、dlclose、dlerror

$ ./main 
start to call test
open error:libtest.so: cannot open shared object file: No such file or directory
```
运行时，我们发现并没有如预期的那样。但是可以看到，程序已经打印了`start to call test`，然后才报错，说明程序是在运行起来之后再尝试去从动态库中查找`test`符号的。
当然了，至于问题原因，我们在前面已经提到了，是由于没有设置动态库搜索路径或者在系统默认库路径下没有我们需要的libtest.so。


## 显式调用和隐式调用的对比
**对比**：
- 显式调用是在**源代码**中明确指定使用动态库的函数或变量，需要在编译时指定动态库的名称或链接选项（`-dl`）。

- 隐式调用是在程序运行时自动加载和链接动态库，无需在源代码中显式指定调用。
只是在编译时，通过`-I`指定库的头文件，以及`-L 和 -l` 指定库的路径以及名称。在代码中不需要指定具体的代码库的路径，直接调用即可。

**加载方式**：
程序运行时，需要用到库中 函数时，一个是通过 `dlopen`加载库 和 一个是自动加载库。

**加载时机**：
显式调用和隐式调用是动态库的使用方式，并不影响动态库的加载时机。动态库的加载仍然是在程序运行时进行的，即调用到动态库中的函数时加载动态库。

# 动态库更新
## 背景
用户总是希望服务进程能保持稳定。如果可以 `7*24` 小时的工作，那就永远不要重启它。但是，软件产品的功能总是在不断的丰富。当用户发现一些新的功能正是他所需要的，他也许会主动要求进行一次升级。而当严重的安全问题出现时，用户就不得不接受强制的升级了。

不停机升级，也被称为热升级。通常实现热升级，需要用户部署两套业务系统。至少，被升级的关键模块是两块以上的。

Linux 环境中的应用依赖相当数量的共享库。通常，为了达到软件模块化的目的，开发人员会把逻辑上紧密相关的功能集中在一起，编译到共享库中。这样做，既有利于代码的管理，也便于模块的复用。同时，共享库的方式也有利于应用升级。许多时候，仅仅更新数个共享库就可以完成整个应用的升级，降低了升级时的开销。

## 静态替换
针对未被程序加载的SO，利用复制命令（cp new.so old.so）即可直接完成静态替换，新SO在下次加载时生效。

对于已经被程序加载的原SO，不能直接使用`cp`进行覆盖，否则源程序可能段错误`Segmentation fault (段错误)`。
```text
程序崩溃的原因是复制替换操作会破坏系统访问原SO的索引节点inode，导致系统找不到原SO。系统为每个加载到内存中的文件创建对应的inode，用来管理该文件，inode包含了文件的元信息，如文件字节数、拥有者ID、读写执行权限等。

系统以inode标识程 序加载的SO，不再关心文件名，修改SO名称并未改变对应inode，因此程序可以继续正常运行；删除SO只是无法查看，系统直到程序释放SO后才真正删除SO和inode，因此程序也可以继续正常运行；但是在直接复制替换时，新SO将会继承原SO的inode，程序无法继续访问原SO，从而导致程序崩溃。
```

正确的做法是：
进入动态库所在的文件夹（比如：/usr/lib64），先删除(rm)原来库文件或修改原SO名称（mv old.so oldx.so）后，再放进去（cp）新库文件。替换之后，执行`ldconfig`命令。新SO同样在下次加载时生效(即旧程序退出，下次新程序启动时自动加载新的so)。
```text
cp from to，则被覆盖文件 to的inode依旧不变（属性也不变），内容变为from的；
mv from to，则to的inode变为from的，相应的，to的属性也成了from的；rm类似；
```


## 动态替换
针对已经被程序加载的SO，为了实现不停止程序，替换后的SO立即生效的目的，可以采用动态替换。
动态替换的对象既可以是SO整体，也可以是SO中的特定函数。两者的区别主要是整体替换需要在特定函数替换的基础上再增加SO加载及输出函数重定位等过程。

SO特定函数动态替换主要包括三个关键过程：控制目标进程，构造替换内容和确定替换地址。实际上依次解决的就是利用什么替换、替换什么内容和替换到哪里的问题。

**详细的动态替换的流程查看：`Linux利用动态库进行软件的热升级`**

# 动态库的查找路径
执行`main2`程序发现报错：
```bash
error while loading shared libraries: libtest.so: cannot open shared object file: No such file or directory
```

原因是共享库不在系统默认的路径里面，可以在shell执行`export LD_LIBRARY_PATH=./` 添加当前路径或者在` /etc/ld.so.conf` 增加路径并`ldconfig`生效。


# 其他
## 位置无关代码PIC
### 简介
系统使用共享库的主要目的是允许多个进程共享内存中相同的库代码，从而节约内存空间。

位置无关代码（Position-Independent Code）技术可以加载无需重定位的代码，`gcc`使用-`fpic`选项可生成`PIC`代码，共享库则必须使用该项.

无论在内存中何处加载一个目标模块，其数据段和代码段的距离总是不变的，也就是说，**​ 代码段中任意指令和数据段中任意变量之间的距离是一个运行时常量。**

编译器在数据段开头的地方创建全局偏移量表GOT（Global Offset Table），在GOT中，每个被目标模块引用的全局数据目标都有8字节的条目，编译器还会为GOT每一个条目生成重定位记录，在加载时，动态链接器会重定位GOT中的每个条目，使之得到正确地址。
**简单来说，就是对于共享库中的变量，是通过指令和变量之间距离常量来找的**。

### 范例
```c
// demolib.h
void add();
int g_num=0;

// demolib.c
// gcc -lazy -shared -fpic -o demolib.so demolib.c
#include<stdio.h>
#include "demolib.h"
void add(){
        g_num++;
        return;
}


// demo.c
// gcc -lazy -o demo demo.c ./demolib.so
#include <stdio.h>
#include "demolib.h"
int main(){
        printf("%d\n",g_num);
        add();
        printf("%d\n",g_num);
        add();
        printf("%d\n",g_num);
}

```
功能很简单，就是去访问共享库的变量和方法
通过`gdb`打开该程序，查看访问变量`g_num`的指令：
![](attachments/Pasted%20image%2020240312190358.png)
可以看到，这里使用当前指令地址rip加上一个固定偏移`0x2e9d`来访问数据.

## 为什么cp的方式更新运行中进程的so，程序会coredump

### 背景
**怎样在不停止程序的情况下替换so文件，并且保证程序不会崩溃**
### 问题
我们的公共组件绝大部分都支持so形式的自定义插件，在不停进程使用cp命令更新so的时候，往往会产生coredump。

### 范例
动态库的源码：
```c
# cat test.c
#include<stdio.h>
 
void test1(void){
    int j=0;
    printf("test1:j=%d\n", j);
    return ;
}
 
void test2(void){
    int j=1;
    printf("test2:j=%d\n", j);
    return ;
}
```
生成动态库：
```bash
gcc -fPIC -shared -o libtest.so test.c -g 生成共享库文件
```


main的源码
```c
# cat main.c
#include <stdio.h>
#include <dlfcn.h> /* 必须加这个头文件 */
 
int main()
{
    void *lib_handle;
    void (*fn1)(void);
    void (*fn2)(void);
    char *error;
 
    lib_handle = dlopen("libtest.so", RTLD_LAZY);
    if (!lib_handle)
    {
        fprintf(stderr, "%s\n", dlerror());
        return 1;
    }
 
    fn1 = dlsym(lib_handle, "test1");
    if ((error = dlerror()) != NULL)
    {
        fprintf(stderr, "%s\n", error);
        return 1;
    }
 
    printf("fn1:0x%x\n", fn1);
 
    fn1();
 
    fn2 = dlsym(lib_handle, "test2");
    if ((error = dlerror()) != NULL)
    {
      fprintf(stderr, "%s\n", error);
      return 1;
    }
 
    printf("fn2:0x%x\n", fn2);
 
    fn2();
 
    dlclose(lib_handle);
 
    return 0;
}

```

生成可执行文件：
```bash
执行gcc -o main main.c -ldl -g 
生成二进制文件main
```

用gdb加载main进行调试：
```bash
gdb -q main
Reading symbols from /root/so/main...done.
(gdb) b 27 //在main.c第27行设置断点
Breakpoint 1 at 0x80485fc: file main.c, line 27.
(gdb) l 27 //显示代码
22              return 1;
23          }
24
25          [printf](http://www.opengroup.org/onlinepubs/009695399/functions/printf.html)("fn1:0x%x\n", fn1);
26             
27          fn1();
28
29          fn2 = dlsym(lib_handle, "test2");
30          if ((error = dlerror()) != NULL)  
31          {
(gdb) r  //运行程序
Starting program: /root/so/main 
fn1:0x2c1450
 
Breakpoint 1, main () at main.c:27 //中断在我们预设的27行
27          fn1();
Missing separate debuginfos, use: debuginfo-install glibc-2.12-1.7.el6.i686
//在另外一个终端里用cp将libtest2.so(仅仅是libtest.so的拷贝而已)覆盖libtest.so
(gdb) s  //单步跟入函数fn1()的实现
test1 () at test.c:4
4           int j=0;   
(gdb) s
5           [printf](http://www.opengroup.org/onlinepubs/009695399/functions/printf.html)("test1:j=%d\n", j);  //执行test.c第4行   int j=0;  并没有问题，因为没有引入未外部符号。
(gdb) s
 
Program received signal SIGSEGV, Segmentation fault.  //执行到test.c第5行printf("test1:j=%d\n", j);出现问题，因为printf是外部符号
0x0000035a in ?? ()
(gdb) bt  //打印堆栈信息
#0  0x0000035a in ?? ()
#1  0x002c147e in test1 () at test.c:5  //test.c第5行是printf("test1:j=%d\n", j);
#2  0x08048602 in main () at main.c:27
(gdb)
```
### 分析
先看一下用cp的方式更新so的时候发生了什么事情
```bash
strace cp libnew.so libold.so 2>&1 |grep open.*lib.*.so

open("libnew.so", O_RDONLY|O_LARGEFILE) = 3

open("libold.so", O_WRONLY|O_TRUNC|O_LARGEFILE) = 4
```

在 cp 使用“O_WRONLY|O_TRUNC” 打开目标文件时，原 so 文件的镜像被意外的破坏了。
即老的so被截断(`trunc`)了，这个过程发生的具体的事情是：  
  
      1.应用程序通过`dlopen`打开so的时候，kernel通过`mmap`把so加载到进程地址空间，对应于vma里的几个page.  
  
      2.在这个过程中loader会把so里面引用的外部符号例如`malloc`、 `printf`等解析成真正的虚存地址。  
  
      3.当so被cp覆盖时，确切地说是被trunc时，kernel会把so文件在虚拟内存的页清理（`purge`） 掉。  
  
      4.当运行到so里面的代码时，因为物理内存中不再有实际的数据（仅存在于虚存空间内），会产生一次缺页中断。  
  
      5.Kernel从so文件中copy一份到内存中去,
      a)但是这时的全局符号表并没有经过解析，当调用到时就产生`segment fault`;
      b)如果需要的文件偏移大于新的so的地址范围，就会产生`bus error`.


### 小结
cp覆盖so文件的时候由于不会改变inode号；
所以，如果用相同的so去覆盖 ：
A) 如果so 里面依赖了外部符号，coredump  
B) 如果so里面没有依赖外部符号，运气不错，不会coredump

### 解决
#### 方法一
采用“rm＋cp” 或“mv＋cp” 来替代直接“cp” 的操作方法。然后 执行`ldconfig`命令。

在用新的so文件 libnew.so 替换旧的so文件 libold.so 时，如果采用如下方法：
```
rm libold.so
cp libnew.so libold.so
```
采用这种方法，目标文件 libold.so 的 inode 其实已经改变了。
原来的 libold.so 文件虽然不能用　”ls”查看到，但其 inode 并没有被真正删除，直到内核释放对它的引用(即旧的程序退出)。同理，mv只是改变了文件名，其 inode 不变，新文件使用了新的 inode。**这样动态链接器 ld.so 仍然使用原来文件的 inode 访问旧的 so 文件。因而程序依然能正常运行**。

> 注：到这里，我们回想在上线操作中在替换可执行程序时，为什么直接使用“cp new old”这样的命令时，系统会禁止这样的操作，并且给出这样的提示“cp: cannot create regular file `old': Text file busy”。这时，我们采用的办法仍然是用“rm+cp”或者“mv+cp”来替代直接“cp”，这跟以上提到的so文件的替换有同样的道理。

但是，为什么系统会阻止 cp 覆盖可执行程序，而不阻止覆盖 so 文件呢？
这是因为 Linux 有个 Demand Paging 机制，所谓“Demand Paging”，简单的说，就是系统为了节约物理内存开销，并不会程序运行时就将所有页（page）都加载到内存中，而只有在系统有访问需求时才将其加载。  
“Demand Paging”要求正在运行中的程序镜像（注意，并非文件本身）不被意外修改，因此内核在启动程序后会锁定这个程序镜像的 inode。对于 so 文件，它是靠 ld.so 加载的，而ld.so毕竟也是用户态程序，没有权利去锁定inode，也不应与内核的文件系统底层实现耦合。

#### 方法二
所有问题的产生都是因为so被trunc了一把，所以如果不用turnc的方式就避免这个问题。Ok,该我们的`install` 上场了。
```bash
install new.so old.so
```
install 的方式跟cp不同，先unlink再creat，当unlink的时候，已经map的虚拟空间vma中的inode结点没有变，只有inode结点的引用计数为0是，kernel才把它干掉。  
也就是新的so和旧的so用的不是同一个inode结点，所以不会相互影响。
所以采用这种方式的话就可以避免先stop进程，更新so，再重启进程这样比较耗时的操作。

# 参考
```bash
# Linux下so动态链接库使用总结
https://www.a-programmer.top/2018/06/13/%E8%A6%86%E7%9B%96so%E5%AF%BC%E8%87%B4coredump%E9%97%AE%E9%A2%98%E6%80%BB%E7%BB%93/

# 如何进行Linux平台共享库替换
https://cloud.tencent.com/developer/article/1040842
```