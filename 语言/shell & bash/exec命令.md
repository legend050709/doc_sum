# 定义
exec是个bash builtin内置命令.
exec命令来自英文单词execute的缩写，其功能是用于调用并执行指定的命令，  或者进行文件的重定向。

# 作用
![](attachments/Pasted%20image%2020230705144942.png)

## exec执行命令 
***exec执行命令时，不会启用新的shell进程，且exec命令后的其他命令将不再执行***

因为exec是内建命令，将并不启动新的shell，而是用要被执行命令替换当前的shell进程，并且将老进程的环境清理掉，而且exec命令后的其它命令将不再执行。

### Linux C的 exec系列函数

常用的场景是：先fork后exec！
一个进程主要包括以下几个方面的内容:
1. 一个可以执行的程序
2. 与进程相关联的全部数据(包括变量，内存，缓冲区)
3. 程序上下文(程序计数器PC,保存程序执行的位置)

我们知道，在fork()建立新进程之后，父进各与子进程共享代码段，但数据空间是分开的，但父进程会把自己数据空间的内容copy到子进程中去，还有上下文也会copy到子进程中去。而为了提高效率，采用一种写时copy的策略，即创建子进程的时候，并不copy父进程的地址空间，父子进程拥有共同的地址空间，只有当子进程需要写入数据时(如向缓冲区写入数据),这时候会复制地址空间，复制缓冲区到子进程中去。从而父子进程拥有独立的地址空间。而对于fork()之后执行exec后，这种策略能够很好的提高效率，如果一开始就copy,那么exec之后，子进程的数据会被放弃，被新的进程所代替。

对于exec系列函数来说，它的作用就是用另一个可执行程序替换当前的进程。
在Linux中，创建新的进程使用fork系统调用，创建之后再调用exec执行新进程的代码。
当进程A正在执行时，通过调用exec函数，使得另一个可执行程序B的数据段、代码段和堆栈段取代当前进程A的数据段、代码段和堆栈段，当前的进程开始执行程序B，这一过程不会创建新的进程，而且PID也没有改变。

- fork 创建子进程
fork是linux的系统调用，用来创建子进程（child process）。子进程是父进程(parent process)的一个副本，从父进程那里获得一定的资源分配以及继承父进程的环境。***子进程与父进程唯一不同的地方在于pid（process id）***。环境变量（传给子进程的变量，遗传性是本地变量和环境变量的根本区别）只能单向从父进程传给子进程。不管子进程的环境变量如何变化，都不会影响父进程的环境变量。

- exec执行
系统调用exec是以新的进程去代替原来的进程，但进程的PID保持不变。
因此，可以这样认为，exec系统调用并没有创建新的进程，只是替换了原来进程上下文的内容。原进程的代码段，数据段，堆栈段被新的进程所代替。

- exec函数族
![](attachments/Pasted%20image%2020230705123626.png)
```c
可以看到这个函数组的函数名都是有exec和添加l、v、p、e这四个字母的组合而来。

字母l代表list，表示该函数取一个参数表，即要求将新程序的每个命令行参数都说明为一个单独的参数，必须以(char*)0结尾；
字母v代表vector，表示该函数取一个argv[]矢量，即要求先构造一个指向各个参数的数组指针，然后将该数组地址作为这个exec函数的参数；
字母p代表path，表示该函数取filename作为参数，并且使用PATH环境变量寻找可执行文件；
字母e代表environ，表示该函数取一个envp[]数组，而不使用当前环境。
这里，字母l和v互斥，即不能同时出现在exec函数中。
```

![](attachments/Pasted%20image%2020230705142518.png)
![](attachments/Pasted%20image%2020230705142707.png)
````c
#include <stdio.h>
#include <unistd.h>
#include <errno.h>

int main(void) {
    execl("/usr/bin/ls", "ls", "-alh", NULL);
    /* if error */
    perror("execl");
    return errno;
}

执行效果：

```text
$ gcc -Wall -Wextra test_exec.c -o test_exec
$ ./test_exec
total 76K
drwxrwxr-x  2 xinlin xinlin 4.0K 7月   7 11:06 .
drwxr-xr-x 26 xinlin xinlin 4.0K 7月   7 11:06 ..
-rwxrwxr-x  1 xinlin xinlin  17K 7月   7 11:06 test_exec
-rw-rw-r--  1 xinlin xinlin  182 7月   7 11:06 test_exec.c

注：`ls`总是出现两次！第2次出现表示0号参数。
````

- 对比
bash提供的exec命令，就像直接执行exec系统调用，没有用fork创建新进程，也没有新的PID，当前的shell进程就被替换了。

#### exec与system系统调用的区别
exec是直接用新的进程去代替原来的程序运行，运行完毕之后不回到原先的程序中去。
system是调用shell执行你的命令，system=fork+exec+waitpid,执行完毕之后，回到原先的程序中去。继续执行下面的部分。

总之，如果你用exec调用，首先应该fork一个新的进程，然后exec. 而system不需要你fork新进程，已经封装好了。


### exec命令 和  source 的区别
source和 .也不会启用新的shell进程，在当前shell中执行，设定的局部变量在执行完命令后仍然有效。  
bash或sh执行时，会另起一个子shell进程，其继承父shell进程的环境变量，其子shell进程的变量执行完后不影响父shell进程的变量。  
exec是用被执行的命令行替换掉当前的shell进程，且exec命令后的其他命令将不再执行。

> 注：exec和source都属于bash内部命令（builtins commands），在bash下输入man exec或man source可以查看所有的内部命令信息。
![](attachments/Pasted%20image%2020230705144008.png)
![](attachments/Pasted%20image%2020230705144210.png)

### 注意
**注意：exec命令不是shell，因此shell中定义的function和alias等，exec在执行时都找不到。**

### 范例
在当前shell中执行 exec ls  表示执行ls这条命令来替换当前的shell ，即为执行完后会退出当前shell。因为这个shell进程已被替换为仅仅执行ls命令的一个进程，执行结束自然也就退出了。

为了避免这个结果的影响，一般***将exec命令放到一个shell脚本中***，用主脚本调用这个脚本，调用点处可以用bash a.sh，（a.sh就是存放该命令的脚本），这样会为a.sh建立一个sub shell去执行，当执行到exec后，该子脚本进程就被替换成了相应的exec的命令。

```bash
cat test_exec.sh
####################
!/bin/bash  
echo"Hello mysql"
exec echo"Hello oracle"
echo"Hello sqlserver"
```
![](attachments/Pasted%20image%2020230705122115.png)

## exec操作文件描述符设置重定向

exec后面不跟command，而是设定一个整个shell都有效的输入输出重定向。这种用法，exec命令就不会销毁当前的shell进程，只是将所有的输出，重定向到文件。

注：一定记住exec是内置命令，如果在当前的bash或者脚本下执行了 exec > FILE, 由于是内置命令，对当前进程有效。并且当前bash/脚本的子进程会继承其fd，所以当前进程的子进程也会将标准输出重定向到 FILE 中。

### 应用
在脚本中这个技巧比较有用，可以统一将输出进行重定向，而不用在每一条命令上设置重定向，减少很多键盘工作。

#### exec 输入重定向
exec <filename命令会将stdin重定向到文件中. 从这句开始, 所有的stdin就都来自于这个文件了, 而不是标准输入(通常都是键盘输入). 这样就提供了一种按行读取文件的方法, 并且可以使用sed和/或awk来对每一行进行分析.
```bash
#!/bin/bash
# 使用'exec'重定向stdin.
 
exec 6<&0          # 将文件描述符#6与stdin链接起来.
                   # 保存stdin.
 
exec < data-file   # stdin被文件"data-file"所代替.
 
read a1            # 读取文件"data-file"的第一行.
read a2            # 读取文件"data-file"的第二行.
 
echo
echo "Following lines read from file."
echo "-------------------------------"
echo $a1
echo $a2
 
echo; echo; echo
 
exec 0<&6 6<&-
#  现在将stdin从fd #6中恢复, 因为刚才我们把stdin重定向到#6了,
#+ 然后关闭fd #6 ( 6<&- ), 好让这个描述符继续被其他进程所使用.
#
# <&6 6<&-    这么做也可以.
 
echo -n "Enter data  "
read b1  # 现在"read"已经恢复正常了, 就是能够正常的从stdin中读取.
echo "Input read from stdin."
echo "----------------------"
echo "b1 = $b1"
 
echo
 
exit 0
```

#### exec 输出输出重定向
```bash
#!/bin/bash
# upperconv.sh
# 将一个指定的输入文件转换为大写.
 
E_FILE_ACCESS=70
E_WRONG_ARGS=71
 
if [ ! -r "$1" ]     # 判断指定的输入文件是否可读?
then
  echo "Can't read from input file!"
  echo "Usage: $0 input-file output-file"
  exit $E_FILE_ACCESS
fi                   #  即使输入文件($1)没被指定
                     #+ 也还是会以相同的错误退出(为什么?).
 
if [ -z "$2" ]
then
  echo "Need to specify output file."
  echo "Usage: $0 input-file output-file"
  exit $E_WRONG_ARGS
fi
 
 
exec 4<&0
exec < $1            # 将会从输入文件中读取.
 
exec 7>&1
exec > $2            # 将写到输出文件中.
                     # 假设输出文件是可写的(添加检查?).
 
# -----------------------------------------------
    cat - | tr a-z A-Z   # 转换为大写.
#   ^^^^^                # 从stdin中读取.
#           ^^^^^^^^^^   # 写到stdout上.
# 然而, stdin和stdout都被重定向了.
# -----------------------------------------------
 
exec 1>&7 7>&-       # 恢复stout.
exec 0<&4 4<&-       # 恢复stdin.
 
# 恢复之后, 下边这行代码将会如预期的一样打印到stdout上.
echo "File \"$1\" written to \"$2\" as uppercase conversion."
 
exit 0
```

### exec 重定向和普通的命令重定向的区别

exec N > filename会影响整个脚本或当前shell. 对于这个指定PID的脚本或shell来说, 从这句命令执行之后, 就会重定向到这个文件中。

command N > filename只会影响command新fork出来的进程, 而不会影响整个脚本或shell。

### 特点
exec命令对文件描述符操作时，不会替换shell，且操作完成后还会继续执行后面的命令。

### 2>&1 说明
CMD >FILE 2>&1 运行一个命令并把它的标准输出和错误输出合并。
(严格的说是通过复制文件描述符 1 来建立文件描述符 2 ，但效果通常是合并了两个流。)

对 2>&1详细说明一下 ：2>&1 也就是 FD2＝FD1 ，这里并不是说FD2 的值 等于FD1的值，因为 > 是改变送出的数据信道，也就是说把 FD2 的 “数据输出通道” 改为 FD1 的 “数据输出通道”。类似于 dup2的作用。

### 重定向cmd >a 2>a 和 cmd >a 2>&1 为什么不同

cmd >a 2>a ：stdout和stderr都直接送往文件 a ，a文件会被打开两遍，由此导致stdout和stderr互相覆盖。

cmd >a 2>&1 ：stdout直接送往文件a ，stderr是继承了FD1的管道之后，再被送往文件a 。a文件只被打开一遍，就是FD1将其打开。

### 其他
tee 命令是在不影响原本 I/O 的情况下，将 stdout 复制一份到档案去。
![](attachments/Pasted%20image%2020230705153417.png)

### 范例
下面的代码，用exec命令打开FD5 for reading，FD5（File Descriptor）的内容来自文件：
```bash
$ cat output.txt
cs.pynote.net/about
$ exec 5< output.txt
$ read -u 5 aline
$ echo $aline
cs.pynote.net/about
$ exec 5<&-  # close the open read FD5
```

下面的代码，是一个打开FD7用来写的例子，注意 `>` 和 `>>` 都可以用：
```bash
$ exec 7> shuchu.txt  # >> means append
$ echo 'haha...' 1>&7
$ cat shuchu.txt
haha...
$ exec 7>&-  # close the open write FD7
```

打开一个FD，同时用于读写：
```
$ exec 18<> rw.txt  # open rw.txt for reading and writing as FD18
$ exec 18<>&-  # close FD18
```

**关闭exec设置的重定向**
```
$ exec >> a.txt
$ command and output
$ exec 1>/dev/tty  # 恢复monitor输出

注：`/dev/tty`，代表当前tty设备（Controlling Terminal），在当前的终端中输入：`sudo echo "hello" > /dev/tty`，都会直接显示在当前的终端中。
echo命令后面也可以换成别的终端设备文件，实现TTY相互之间发消息。
tty命令显示出来的dev，与/dev/tty等价。
```
```c
# tty
/dev/pts/5

# lsof -a -p $$ -d0,1,2
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
bash    34654 root    0u   CHR  136,5      0t0    8 /dev/pts/5
bash    34654 root    1u   CHR  136,5      0t0    8 /dev/pts/5
bash    34654 root    2u   CHR  136,5      0t0    8 /dev/pts/5

# bash -c 'lsof -a -p $$ -d0,1,2' | cat
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF       NODE NAME
lsof    35519 root    0u   CHR  136,5      0t0          8 /dev/pts/5
lsof    35519 root    1w  FIFO   0,13      0t0 1345369953 pipe
lsof    35519 root    2u   CHR  136,5      0t0          8 /dev/pts/5
```


