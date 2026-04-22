```table-of-contents
```
# 概述

信号是异步的，它会在程序的任何地方发生。由此程序正常的执行路径被打破，去执行信号处理函数。**一般情况下，进程正在执行某个系统调用，那么在该系统调用返回前信号是不会被传递的。但慢系统调用除外，这些系统调用都会返回-1，errno置为EINTR**。当系统调用被中断时，我们可以选择使用循环再次调用，或者设置重启该系统调用。

`man 7 signal`, 如下所示：
![](attachments/Pasted%20image%2020250525142850.png)

# 系统调用分类

为了支持系统调用被信号打断而不再执行的特性，将系统调用分为两类：低速系统调用和其他系统调用。

当信号处理函数返回时，有可能发生以下的情况：
- 如果信号处理函数是用`signal`注册的，系统调用会自动重启，系统调用函数不会返回。
- 如果信号处理函数是用`sigaction`注册的
    - 默认情况下，系统调用不会自动重启，函数将返回失败，同时`errno`被置为`EINTR`;
    - 只有中断信号的`SA_RESTART`标志有效时，系统调用才会自动重启。

## 慢系统调用（slow system call）
慢系统调用（低速系统调用）「`slow system call`」指的是那些可能被永远阻塞的系统调用。永远阻塞的系统调用是指调用永远无法返回，多数**网络支持函数**都属于这一类，比如：若没有客户连接到服务器上，那么服务器的**accept**调用就会一直阻塞。

==注：对于非阻塞的`fd`的系统调用，不存在慢系统调用，因为会直接返回==。

### 分类
慢系统调用可以被永久阻塞，包括以下几个类别：

（1）读写‘慢’设备（包括pipe，终端设备，网络连接等）。读时，数据不存在，需要等待；写时，缓冲区满或其他原因，需要等待。
（2）当打开某些特殊文件时，需要等待某些条件，才能打开。例如：打开中断设备时，需要等到连接设备的modem响应才能完成。
（3）pause和wait函数。pause函数使调用进程睡眠，直到捕获到一个信号。wait等待子进程终止。
（4）某些ioctl操作。
（5）某些IPC操作。

## 其他系统调用

### 系统调用的自动重启动
#### 简介
当系统调用被信号中断时，并不返回，而是继续执行。
比如：`read()`阻塞等待，此时进程接受到信号时，并不将`read`返回，而是继续阻塞等待。


# `EINTR` 类型的errno错误


## `EINTR`错误产生的原因
总体来说，就是慢系统调用被信号中断。具体又分为2种情况：

### 原因一：慢系统调用被含有信号处理函数的信号中断
如果进程在一个**慢系统调用(`slow system call`)** 中阻塞时，当捕获到某个信号且相应信号处理函数返回时，这个系统调用不再阻塞而是被中断，就会调用返回错误（一般为`-1`）&& 设置`errno`为`EINTR`（相应的错误描述为`Interrupted system call`）。

### 原因二：慢系统调用在程序收到停止信号后又收到`SIGCONT`信号
在 Linux 上，即使没有信号处理器函数，某些阻塞的系统调用也会产生 `EINTR` 错误。如果系统调用遭到阻塞，并且进程因信号（`SIGSTOP、SIGTSTP、SIGTTIN 或 SIGTTOU`）而停止，之后又因收到 `SIGCONT` 信号而恢复执行时，就会发生这种情况。

以下系统调用和函数具有这一行为：`epoll_pwait()`、`epoll_wait()`、对 `inotify` 文件描述符执行的 `read()`调用、`semop()`、`semtimedop()`、`sigtimedwait()`和 `sigwaitinfo()`。
这种行为的结果是，如果程序可能因为信号而停止和重启，那么系统添加代码来重新启动这些系统调用，即便该程序并未为停止信号设置处理器函数。


## 解决办法
既然系统调用会被中断，那么别忘了要处理被中断的系统调用。有三种处理方式：

### 解决方法1：人为重启被中断的系统调用
**（1）如何理解“重启”？**
一些`IO`系统调用执行时，如 `read` 等待输入期间，如果收到一个信号，系统将中断`read`， 转而执行信号处理函数；当信号处理返回后， 系统遇到了一个问题： 是重新开始这个系统调用? 还是让系统调用失败?
早期UNIX系统的做法是：中断系统调用，并让系统调用失败， 比如`read`返回 `-1`， 同时设置 `errno` 为`EINTR`；
中断了的系统调用是没有完成的调用，它的失败是临时性的，如果再次调用则可能成功，这并不是真正的失败，所以要对这种情况进行处理， 典型的方式为“重启”。


当碰到`EINTR`错误的时候，有一些可以重启的系统调用要进行重启，而对于有一些系统调用是不能够重启的。
例如：
（1）`accept`、`read`、`write`、`select`、和`open`之类的函数来说，是可以进行重启的。
（2）不过对于套接字编程中的`connect`函数是不能重启的。
若`connect`函数返回一个`EINTR`错误的时候，我们不能再次调用它，否则将立即返回一个错误。针对`connect`不能重启的处理方法是，必须调用`select`来等待连接完成。
```c
again:
    if ((n = read(fd， buf， BUFFSIZE)) < 0) {
       if (errno == EINTR)
            goto again;     /* just an interrupted system call */
      /* handle other errors */
    }
```

如果需要频繁使用上面代码，那么定义成如下宏会比较方便：
```c
#define NO_EINTR(stmt) while((stmt) == -1 && errno == EINTR);

NO_EINTR(cnt = read(fd, buf, BUS_SIZE));
if(cnt == -1) {   // read()failed with other than EINTR
   exit(EXIT_FAILUER);
}
```

`GNU C` 库提供了一个（非标准）宏，其作用与定义于`<unistd.h>`中的 `NO_EINTR()`相同。该宏名为 `TEMP_FAILURE_RETRY()`，定义特性测试宏`_GNU_SOURCE` 后即可使用。



#### 重启`connect`的问题

当`connect`遇到`EINTR`错误时，不能向上面那样重新进入循环处理。
原因是，`connect`的请求已经发送向对方，正在等待对方回应。此时如果重新调用`connect`，而对方已经接受了上次的`connect`请求，这一次的`connect`就会被拒绝「此时新的`connect`应该会发送相同五元组的`Syn`?」。

##### 处理
因此，需要使用`select`或`poll`调用来检查`socket`的状态。
如果`socket`的状态就绪，则`connect`已经成功；
否则，视错误原因，做对应的处理。

```c
#include "poll.h"
 
int check_conn_is_ok(socket_t sock) {
	struct pollfd fd;
	int ret = 0;
	socklen_t len = 0;
 
	fd.fd = sock;
	fd.events = POLLOUT;
 
	while ( poll (&fd, 1, -1) == -1 ) {
		if( errno != EINTR ){
			perror("poll");
			return -1;
		}
	}
 
	len = sizeof(ret);
	if ( getsockopt (sock, SOL_SOCKET, SO_ERROR,
                     &ret,
                     &len) == -1 ) {
    	        perror("getsockopt");
		return -1;
	}
 
	if(ret != 0) {
		fprintf (stderr, "socket %d connect failed: %s\n",
                 sock, strerror (ret));
		return -1;
	}
 
	return 0;
}
```


```c
#include "erron.h"
 
....
if(connnect()) {
    if(errno == EINTR) {
        if(check_conn_is_ok() < 0) {
              perror();
              return -1;
        }
        else {
             printf("connect is success!\n");
        }
    }
    else {
         perror("connect");
         return -1;
    }
}
```

#### 范例
采用`accept`函数为例子，代码如下：
```c
ACCEPT:
    clifd = accept(srvfd, (struct sockaddr*)&cliaddr, &cliaddrlen);
    if (clifd == -1) {
        if (errno == EINTR) {
            goto ACCEPT;
        } else {
            fprintf(stderr, "accept fail,error:%s\n", strerror(errno));
            return -1;
        }
    }
```


### 解决方法2：安装信号时设置 `SA_RESTART` 属性（该方法对有的系统调用无效）
```c
struct sigaction action;  
     
  action.sa_handler = handler_func;  
  sigemptyset(&action.sa_mask);  
  action.sa_flags = 0;  
  /* 设置SA_RESTART属性 */  
  action.sa_flags |= SA_RESTART;  
     
  sigaction(SIGALRM, &action, NULL);
```

![](attachments/Pasted%20image%2020250525141521.png)


#### 注意
不幸的是，并非所有的系统调用都可以通过指定 `SA_RESTART` 来达到自动重启的目的。

### 解决方法3： 忽略信号（让系统不产生信号中断）
```c
struct sigaction action;  
     
  action.sa_handler = SIG_IGN;  
  sigemptyset(&action.sa_mask);  
     
  sigaction(SIGALRM, &action, NULL);
```

## 测试代码
闹钟信号`SIGALRM`中断`read`系统调用。安装`SIGALRM`信号时如果不设置`SA_RESTART`属性，信号会中断`read`系统过调用。如果设置了`SA_RESTART`属性，`read`就能够自己恢复系统调用，不会产生`EINTR`错误。

### 测试代码一
```c

#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <error.h>
#include <string.h>
#include <unistd.h>

void sig_handler(int signum) {
    printf("in handler\n");
    sleep(1);
    printf("handler return\n");
}

int main(int argc, char *argv[]){
    char buf[100];
    int ret;
    struct sigaction action, old_action;

    action.sa_handler = sig_handler;
    sigemptyset(&action.sa_mask);
    action.sa_flags = 0;
    /* 版本1:不设置SA_RESTART属性
     * 版本2:设置SA_RESTART属性 */
   // action.sa_flags |= SA_RESTART;
    sigaction(SIGALRM, NULL, &old_action);
    if (old_action.sa_handler != SIG_IGN) {
        sigaction(SIGALRM, &action, NULL);
    }

    alarm(3);

    bzero(buf, 100);

    ret = read(0, buf, 100);
    if (ret == -1) {
        perror("read");
    }

    printf("read %d bytes:\n", ret);
    printf("%s\n", buf);
    return 0;
}
```

### 测试代码二
闹钟信号`SIGALRM`中断`msgrcv`系统调用。即使在插入信号时设置了`SA_RESTART`，也无效。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
 
void ding(int sig)
{
    printf("Ding!\n");
}
 
struct msgst
{
    long int msg_type;
    char buf[1];
};
 
int main()
{
    int nMsgID = -1;
 
    // 捕捉闹钟信息号
    struct sigaction action;
    action.sa_handler = ding;
    sigemptyset(&action.sa_mask);
    action.sa_flags = 0;
    // 版本1:不设置SA_RESTART属性
    // 版本2:设置SA_RESTART属性
    action.sa_flags |= SA_RESTART;
    sigaction(SIGALRM, &action, NULL);
   
    alarm(3);
    printf("waiting for alarm to go off\n");
 
    // 新建消息队列
    nMsgID = msgget(IPC_PRIVATE, 0666 | IPC_CREAT);
    if( nMsgID < 0 )
    {
        perror("msgget fail" );
        return;
    }
    printf("msgget success.\n");
 
    // 阻塞 等待消息队列
    //
    // msgrcv会因为进程收到了信号而中断。返回-1，errno被设置为EINTR。
    // 即使在插入信号时设置了SA_RESTART，也无效。man msgrcv就有说明。
    //
    struct msgst msg_st;
    if( -1 == msgrcv( nMsgID, (void*)&msg_st, 1, 0, 0 ) )
    {
        perror("msgrcv fail");
    }
 
    printf("done\n");
 
    exit(0);
}

```


# 参考
```bash
# linux系统中socket编程错误码：eintr和eagain的处理方法
https://zhuanlan.zhihu.com/p/159130182
```