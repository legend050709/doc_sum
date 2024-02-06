ctrl-d 不是发送信号，而是表示一个特殊的二进制值，表示 EOF
# 定义
EOF 是 end of file 的缩写，表示”文字流”（stream）的结尾。这里的”文字流”，可以是文件（file），也可以是标准输入（stdin）。

# EOF原理

## 范例
以前在学习 C 语言文件操作的时候，一直记得 EOF 就是一个标记，通过它可以判断程序是否读取到文件的末尾了，例如下面的这段代码就是将一个文本文件中的字符输出到标准输出中，并通过 EOF 来判断程序是否读到了文件末尾：
```c
include <stdio.h>

#define FILENAME "gdb_test.c"

int main()
{
    FILE* fd;
    int c;

    fd = fopen(FILENAME, "r");

    while ((c = fgetc(fd)) != EOF)
    {
        fputc(c, stdout);
    }
    fclose(fd);
    return 0;
}
```
所以，我一直很好奇 `EOF` 到底是什么个什么东西，它只是一个简单的常量还是一个特殊的字符呢？

**EOF** 定义在 _/usr/include/stdio.h_ 文件中：
```c
/* End of file character.
Some things throughout the library rely on this being -1.  */
#ifndef EOF
    #define EOF (-1)
#endif
```
从上面 `EOF` 的定义我们可以看出 EOF 本质上就是一个值为`-1`的常量！

## 怎么通过 EOF 来判断程序是否读取到了文件末尾
Linux 系统一个非常重要的思想就是：**一切皆文件。**不管是标准输入，文件系统中的普通文本文件，还是网络流都可以看做是文件，都可以通过 read/write 函数进行读写操作。因此，不同的文件类型，判断是否读取到文件末尾的方式也就有所不同，下面就按照

- 普通文本文件
- 标准输入文件（stdin）
- socket 流文件

这三类文件来介绍判断它们是否读取到文件末尾的方法。
### 普通文本文件
这里的普通文件指的是我们平时在通过文件管理器所能看到那些文本文件，它***存在于 Linux 中的文件系统中，并且文件的大小是固定的。***

对于这种文件，Linux 系统判断普通文本文件是否读取到文件末尾的方法是：
**read 函数会对所打开的文件维护一个读取指针，然后根据这个指针跟文件开始位置的指针值相减得到一个相对于文件开始位置的偏移字节数，最后通过这样一个偏移字节数和文件本身的大小进行一个比较，如果相对于文件开始位置的偏移字节数大于文件本身的大小，那么就返回一个 EOF 常量，说明此时已经读取到文件末尾了。**

### 标准输入流文件
标准输入文件（stdin）它对应的是外设键盘输入，而在 Linux 系统中它被抽象成一个文件，准确地说是一个流文件。
这种文件和上面普通文本文件最大的区别：
***就是它的文件大小是不固定的，它就像是一个水管的进水端，可以在任何时候都可以接收输入。***

正是因为标准输入文件这种流式的特点，决定了无法通过前面提到那种比较文件大小方法来判断是否读取到了文件末尾。因此，Linux 系统判断标准输入文件是否读取到文件末尾的方法是：**设置一个特殊的输入标记来表示文件末尾，而在Linux 系统中这个标记就是组合键`Ctrl+D`，当系统捕获到这个组合键时，就让  read 函数返回一个 EOF 常量，告知程序已经读取到标准文件的末尾了。**

```c
ctrl-c 发送 SIGINT 信号给前台进程组中的所有进程。常用于终止正在运行的程序。  
ctrl-z 发送 SIGTSTP 信号给前台进程组中的所有进程，常用于挂起一个进程。  
ctrl-d 不是发送信号，而是表示一个特殊的二进制值，表示 EOF。  
ctrl-/ 发送 SIGQUIT 信号给前台进程组中的所有进程，终止前台进程并生成 core 文件。

Key Function  
Ctrl-c Kill foreground process  
Ctrl-z Suspend foreground process  
Ctrl-d Terminate input, or exit shell  
Ctrl-s Suspend output  
Ctrl-q Resume output  
Ctrl-o Discard output  
Ctrl-l Clear screen
```

### socket 流文件
socket 流文件和标准输入文件类似都是流式文件，并且它是从网络上进行数据读取，所以上面两种判断文件是否读取到末尾的方法都不适用于 socket 流文件。

**那么客户端进程怎么判断服务端进程是否已经写完所有数据？** 
在 socket 流文件中，当客户端进程通过 read 函数读取远程服务端进程发送过来的数据时，使用的是阻塞I/O的方式进行读取的。只要客户端和服务端之间的连接没有断开，如果服务端没有向 socket 写入数据，那么客户端的读操作就会阻塞，直到服务端中写入了新的数据。

**如果服务端进程关闭了socket连接，那么客户端会接收到服务端发送过来的一个 TCP 协议的 FIN 数据包，然后客户端进程中原本阻塞着等待接收服务端进程数据的 read函数此时就会被唤醒，返回一个值 0。**
这跟我们前面提到两种文件读到文件末尾返回 EOF（值为-1）的情况有点差别，所以在程序中从 socket 进行读取操作时，判断数据流结束的标志不是 -1 而是 0。

所以，一个简单的从 socket 文件读取数据的样例代码，通常是下面这样的：
```c
char recvline[MAX_LINE_LENGTH];
int read_count;
while ((read_count = read(sock_fd, recvline, MAX_LINE_LENGTH)) > 0)
{
    printf("%s\n", "String received from server: ");
    fputs(recvline, stdout);
}
```
# 小结
**EOF 是一个常量而不是一个字符！**

# 参考
```c
https://flyflypeng.tech/linux/2016/07/08/Linux%E4%B8%AD%E7%9A%84EOF%E5%88%B0%E5%BA%95%E6%98%AF%E4%BB%80%E4%B9%88.html
```