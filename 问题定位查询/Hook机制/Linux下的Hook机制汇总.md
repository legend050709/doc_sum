```table-of-contents
```
# 背景
大家有没有想过，一些系统监控软件是如何得知我们所进行的操作的？外挂又是如何读取到游戏的内部数据的？
这些功能的实现，基本都有`API`的`HOOK`存在。

# 介绍
大致意思是将指定函数替换成自己的，然后再去执行这个指定函数。从而偷梁换柱，在原有的函数上添加自己想要的功能；从设计模式上说倒是和装饰器模式非常相似；hook 的核心就是让系统放弃原有代码，转而运行你的代码。

# Linux下的Hook机制概述
Linux hook 有很多中实现方式，从 ring0 到 ring3 都有对应的实现方式。
![](attachments/Pasted%20image%2020240312154155.png)

Intel的CPU将特权级别分为4个级别：RING0,RING1,RING2,RING3；
ing0是最高级别，ring1次之，ring2更次之，以此类推。
拿Linux+x86来说， 操作系统（内核）的代码运行在最高运行级别ring0上，可以使用特权指令，控制中断、修改页表、访问设备等等。  
应用程序的代码运行在最低运行级别上ring3上，不能做受控操作。
如果要做，比如要访问磁盘，写文件，那就要通过执行系统调用（函数），执行系统调用的时候，CPU的运行级别会发生从ring3到ring0的切换，并跳转到系统调用对应的内核代码位置执行，这样内核就为你完成了设备访问，完成之后再从ring0返回ring3。这个过程也称作用户态和内核态的切换。

![](attachments/Pasted%20image%2020240312153838.png)

**Linux的hook一般发生在ring0和ring3层，其中ring0层通常是hook Linux系统调用表中的系统调用，而ring3层则一般hook的动态链接库中的函数。**

# ring3的hook
## 概述
- `LD_PRELOAD`劫持动态库
- `ptrace API`调试技术`Hook`
- PLT(`Procedure Linkage Table`，程序链接表)劫持

## inject
当我们需要`hook`某个应用程序时，我们需要往目标程序中添加自己的代码，这种添加代码的方式就被称之为`inject`，`inject`方式又一般分为两种：
动态注入  和 静态注入。

### 静态注入
静态注入就是在程序还未运行时，去修改其so文件，以达到注入函数的目的，由于没具体研究过，所以此处仅提一下。
### 动态注入
动态注入一般是使用`ptrace`来实现的，一般流程为：
1> `attach`上目标进程。  
2> 查找到目标进程的`dlopen`函数，调用`dlopen`加载要注入的so到目标进程空间。  
3> 查找到目标进程的`dlsym`函数，调用`dlsym`查找到新so中的目标函数地址，并调用目标函数。  
4> 在目标函数中实现自己的代码逻辑。

## hook
### PLT重定向劫持Hook
利用ELF文件的，GOT和PLT的方式解决地址无关的链接.so文件的机制.


在`ring3` 层的`hook`，一般是指 `PLT/GOT hook`，也就是对程序的got表进行替换。

`PLT (Problogcedure Linkage Table)` 和 `GOT (Global Offset Table)` 是 GCC 中生成`shared library(.so动态库)`的重要元素。

为何一定要这两个表?
众所周知`Linux`对外部函数的引用是采用动态链接的，也就是说，在用到某个函数时，才会具体的去定位其在内存中的位置，之所以这么做是为了程序能更快的启动，否则如果程序启动时，就去加载所有引用函数，会让程序启动的很慢。

PLT（Procedure Linkage Table）是一个函数指针表，用于在动态链接过程中找到函数的真正地址。GOT（Global Offset Table）是一个全局偏移表，用于存储函数在内存中的真正地址。

### LD_PRELOAD HOOK

### ptrace hook
ptrace可以实现调试程序、跟踪；
> 注：一个进程只能被一个进程跟踪。所以无法在gdb或者其他程序调试的时候去ptrace一个程序，同样也无法在ptrace一个进程的时候，再去gdb调试。

总体思路
```
ptrace attach目标进程
保存rip
控制跳转到mmap分配一段rwx内存
将一段机器码copy进去
控制跳转到机器码（可以以bin文件的形式）
恢复执行。
```

# ring0的 hook

## 利用0x80劫持system_call
**原理**：
系统调用，会引入0x80处的软中断，从而引入系统调用处理程序，跳转到其入口system_call，system_call会根据系统调用号来确定调用服务，然后根据内核符号表来进行特定跳转。所以，修改这里内核符号表，就可以跳转到自己编写的函数中，完成hook。


**流程**：
```
针对系统调用的hook
    首先获得sys_call_table的基地址
    修改指定偏移对应的系统调用
利用sys函数的嵌套实现hook调用的子函数
修改系统调用的前几个字节为jmp之类的指令
```

# ptrace hook详解

## 总体思路
```
ptrace attach目标进程
保存rip
控制跳转到mmap分配一段rwx内存
将一段机器码copy进去
控制跳转到机器码（可以以bin文件的形式）
恢复执行。
```

## 范例
(1) 获取目标进程中的要替换函数的地址
首先需要知道一些函数在目标进程的地址，下面是已知pid获取libc基地址（读取**/proc/pid/maps**），和函数地址(**dlsym**)
```c
size_t getLibcbase(int pid)
{
    size_t libcAddr;
    char* buf;
    char* end;
    char* mapfile[0x18];
    sprintf(mapfile, Mapfile, pid);
    FILE* fd = fopen(mapfile, "r");
    if(!fd)
    {
        printf("open maps error!");
        exit(1);
    }
    //search the libc-.....
    buf = (char*) malloc(0x100);
    do{
        fgets(buf, 0x100, fd);
    } while(!strstr(buf, "libc-"));
    end = strchr(buf, '-');
    libcAddr = strtol(buf, &end, 16);
    printf("The process %d's libcbase is: 0x%lx\n", pid, libcAddr);
    fclose(fd);
    return libcAddr;
}
size_t getFuncAddr(int pid, char* funcName)
{
    size_t funcAddr;
    char* buf;
    char* end;
    char* mapfile[0x18];
    sprintf(mapfile, Mapfile, pid);

    //get function offset from self process, the shared libc.so
    funcAddr = (size_t)dlsym(0, funcName);
    funcAddr -= getLibcbase(getpid());
    funcAddr += libc_addr;
    printf("function %s address is: 0x%lx\n", funcName, funcAddr);
    return funcAddr;
}
```





# 参考
```c
# Linux动态链接为什么要用PLT和GOT表？[知乎]
https://www.zhihu.com/question/21249496


# linux的系统调用hook
https://zhuanlan.zhihu.com/p/638774761

# Elf文件格式与hook
https://pitechan.com/ELF%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E4%B8%8Ehook/

# hook 的妙用
https://weakyon.com/2022/09/12/magical-effect-of-hook.html

### Linux下Hook方式汇总
https://xz.aliyun.com/t/6961?time__1311=n4%2BxnD0DRDyD9i8tDsAohrFei%3DQiteDBi3D&alichlgref=https%3A%2F%2Fwww.google.com.hk%2F
```