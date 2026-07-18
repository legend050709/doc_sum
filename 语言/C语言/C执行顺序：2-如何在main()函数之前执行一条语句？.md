```table-of-contents
```
# 如何让一段程序在main函数之前执行
## C++中全局变量的构造函数，会在main之前执行
```c++
#include <iostream>
using namespace std;

class app
{
public:
    //构造函数
    app()
    {
        cout<<"First"<<endl;
    }
};

app a;  // 申明一个全局变量

int main()
{
    cout<<"Second"<<endl;
    return 0;
}
```
## C++中全局变量的赋值函数，会在main之前执行
注：C中好像不允许通过函数给全局变量赋值。
C语言从语法上规定全局变量只能用**常量表达式**来初始化，因此下面这种全局变量初始化是不合法的。
```c
int minute = 360; 
int hour = minute / 60;
```
范例如下：
```c++
#include <iostream>
using namespace std;

int f(){
    printf("before");
    return 0;
}

int _ = f();

int main(){
    return 0;
}
```
##  Linux C中的 __attribute__((constructor))
如果是`GNUC`的编译器（`gcc，clang`），就在你要执行的方法前加上 `__attribute__((constructor))`
```c++
#include<stdio.h>

__attribute__((constructor)) void func()
{
    printf("hello world\n");
}

int main()
{
    printf("main\n"); //从运行结果来看，并没有执行main函数
}
```
同理，如果想要在`main`函数结束之后运行，可加上`__attribute__((destructor))`.
```c++
#include<stdio.h>

void func()
{
    printf("hello world\n");
    //exit(0);
    return 0;
}

__attribute((constructor))void before()
{
    printf("before\n");
    func();
}


__attribute((destructor))void after()
{
    printf("after\n");

}

int main()
{
    printf("main\n"); //从运行结果来看，并没有执行main函数
}
```

# main函数是如何被执行的
![](attachments/Pasted%20image%2020240311170902.png)

运行一个程序，经过加载和动态链接后，从`kernel`回到`userspace`时的`entrypoint`取决于平台的ABI，在`x86_64`的`linux`下是`_start`，经过一系列的初始化、`libc code`后调用`main`开始主程序的运行，具有`constructor`的函数正是在这一阶段得到执行。

#  __attribute__介绍

`__attribute__` 可以设置函数属性(`Function Attribute`)、变量属性(`Variable Attribute`)和类型属性(`Type Attribute`)。`__attribute__`前后都有两个下划线，并且后面会紧跟一对原括弧，括弧里面是相应的`__attribute__`参数。

`__attribute__`语法格式为：
```c
__attribute__ ( ( attribute-list ) )
```

## `__attribute__((constructor))`介绍
**用constructor属性指定的函数，会在目标文件加载的时候自动执行，发生在main函数执行以前**，常常用来隐形得做一些初始化工作。

如果函数被设定为`constructor`属性，则该函数会在`main（）`函数执行之前被自动的执行；
```c
void __init() __attribute__((constructor));

也可以是静态函数：
static void __init() __attribute__((constructor));

注： `__init` 只是函数名，没任何含义，也可以是其他的名称。
```
若函数被设定为`destructor`属性，则该函数会在`main（）`函数执行之后或者`exit（）`被调用后被自动的执行。

注意：`attribute((constructor))`属性是`GCC`、`Clang`等编译器提供的扩展功能，不属于标准`C/C++`语言规范，因此使用时需要注意兼容性和可移植性问题。

### 作用

## 动态库（.so）中的构造（constructor）
共享库(shared library)是一种可以被多个程序共享的动态链接库(Dynamic Linking Library)。共享库通常包含一些可被其他程序调用的函数和数据。为了确保共享库在被调用前被正确初始化，我们可以使用`__attribute__((constructor))`来声明一个构造函数。这个构造函数会在共享库被加载时自动被执行。

当程序加载共享库时，动态链接器会查找共享库中是否存在构造函数，如果存在，就会自动执行这个构造函数。执行构造函数可以完成一些必要的初始化操作。

在编写共享库时，我们可以使用`__attribute__((constructor))`来声明一个构造函数，并在其中完成必要的初始化操作。这样可以确保共享库在被调用前被正确初始化，从而保证程序的正常运行。

## 多个构造的执行顺序
### 多个动态库中的构造的执行顺序
目标可执行文件在执行的时候有时会加载不同的动态库，比如动态库libA.so 和libB.so。 动态库中各自包含`constructor`属性修饰的函数，当动态库被加载时，该函数会自动执行。而且执行顺序与动态库的加载顺序恰好相反。

```bash
The invocation of _init() functions follows in the reverse order of library loading (for example, the first library loaded is the last to call _init()). This reversed ordering is necessary because libraries appearing earlier on the link line might depend on functions used in libraries appearing later
```

**范例**：
A.c文本中：
```c
void __attribute__((constructor)) __initA() {
    printf("initA\n");
}
```

B.c文本中：
```c
void __attribute__((constructor)) __initB() {
    printf("initB\n");
}
```

执行分析：
```bash
bryli@diag-bld-07-g1:/ws-bryli/code$ gcc -Wall -fPIC -c A.c B.c
bryli@diag-bld-07-g1:/ws-bryli/code$ gcc -shared -o libA.so A.o
bryli@diag-bld-07-g1:/ws-bryli/code$ gcc -shared -o libB.so B.o
bryli@diag-bld-07-g1:/ws-bryli/code$ gcc -L/ws-bryli/code -Wall main.c -o main -lA -lB

bryli@diag-bld-07-g1:/ws-bryli/code$ ldd main
linux-vdso.so.1 (0x00007ffd27494000)
libA.so => /ws-bryli/code/libA.so (0x00007fe33b24e000)
libB.so => /ws-bryli/code/libB.so (0x00007fe33b04c000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fe33ac5b000)
/lib64/ld-linux-x86-64.so.2 (0x00007fe33b652000)

bryli@diag-bld-07-g1:/ws-bryli/code$ ./main
initB
initA
```
可执行文件先加载A后加载B，但是libB.so中的init先执行，libA.so中的init后执行，刚好验证了前面的论述。

在实际工作中，常常会有很多层动态库，有的动态库是底层函数的封装，有的动态库负责上层函数的封装。上层的动态库往往会依赖于下层的动态库，所以我们编译的时候会先加载上层的的动态库，后加载下层的动态库，保证编译正确。在这种情形下，`constructor`属性函数的动态库，会先执行下层动态库中的`constructor`中的函数，后执行上层的。

### constructor中设置优先级
如果你需要多个函数来完成顺序完成初始化，还可以添加优先级。
**优先级数值越小，越早执行**。不过这种优先级只在单个目标文件中有作用，在不同的目标文件，比如不同动态库文件之间，是没有作用的。

加入优先级的函数声明：
```c
__attribute__((constructor(priority)))

范例：
void __init() __attribute__((constructor(100)));
```

####  默认优先级
未显式设置优先级时，所有构造函数的**默认优先级是0**（实际实现中可能等同于65535）。
此时执行顺序**依赖编译单元（.c文件）在链接时的顺序**，先链接的源文件中的构造函数先执行。

#### 优先级规则
在C语言中，使用`__attribute__((constructor))`可以定义在main函数之前执行的函数。
如果有多个构造函数，它们的执行顺序取决于两个因素：优先级和编译单元（.c文件）的链接顺序。

 **（1）优先级数字越小，越先执行**
 即优先级为100的函数比优先级为200的函数先执行。
 
 **（2）相同优先级的函数**
 执行顺序与编译单元（.c文件）在链接时的顺序有关。链接器按照目标文件（.o）出现的顺序处理，所以先链接的文件的构造函数（相同优先级）会先执行，后链接的后执行。

#### 多个`.c`文件设置优先级来保序
如果你有多个.c文件，每个文件中都有构造函数，并且你需要控制它们的执行顺序，有两种方法：
**（1）显式设置优先级**：给每个构造函数设置不同的优先级，确保你希望先执行的函数优先级数值更小（更优先）。
**（2）控制链接顺序**：调整链接时目标文件的顺序。相同优先级的构造函数，会按照链接顺序执行。但是这种方法不够灵活，尤其是当项目很大时，链接顺序可能由构建系统自动管理，难以精确控制。

##### 范例
**(1) 方法一：显示设置优先级**
```c
// file1.c
__attribute__((constructor(100))) void init1() { 
    /* 最先执行（优先级数值最小） */ 
}

// file2.c
__attribute__((constructor(200))) void init2() { 
    /* 在 init1 后执行 */ 
}
```

**(2) 方法二：控制链接顺序**
```c
# 确保 file1.o 在 file2.o 之前链接 → init_a 先于 init_b 执行 
gcc -o program file1.o file2.o main.c
```

## 相关问题
### 问题1：C++中动态库（.so）中的构造（constructor）中使用静态对象导致程序崩溃
#### 描述
linux下动态链接库（.so）的构造（`constructor`）中使用静态对象导致程序崩溃。
最近按项目的要求封装了一个动态库。本来没啥毛病，中间为了方便调试，加了日志功能（使用了static对象作为单例），并在`constructor`中执行了日志相关设置。编译ok，调用的时候在`dlopen`报错崩溃。

|**坑的描述**|**linux下动态链接库（.so）的constructor中调用了static对象的相关功能，导致调用库时崩溃**|
|---|---|
|**根本原因**|**constructor是在dlopen返回之前调用的，此时可能还未对全局或静态对象执行构造**|
|**填坑进度**|**已解决**|

#### 分析
![](attachments/Pasted%20image%2020240311163919.png)
#### 解决
C++中，不要在使用`constructor`标记的函数中，调用全局或者静态对象；或者将静态对象的初始化在优先级更高的构造函数中执行。

# 参考
```bash
# GDB调试LD_PRELOAD动态链接库
https://zhuanlan.zhihu.com/p/622554160


# Linux X86 程序启动 - main函数是如何被执行的？
https://zhuanlan.zhihu.com/p/52054044

http://dbp-consulting.com/tutorials/debugging/linuxProgramStartup.html (英文版本)
```