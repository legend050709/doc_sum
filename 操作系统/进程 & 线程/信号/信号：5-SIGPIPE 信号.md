```table-of-contents
```
# 介绍
`SIGPIPE`信号对应的数值为 13。
该信号意味着管道错误，通常在进程间通信产生，比如采用`FIFO`(管道)通信的两个进程，==读管道没打开或者意读管道关闭了仍继续通过写管道写==，写进程就会收到 `SIGPIPE` 信号。

# 管道的原理
**(1) Linux管道(pipe)的基本工作原理**：
- 管道是一种单向通信机制
- 创建管道时会返回两个文件描述符：一个用于读取，一个用于写入
- 当所有读取端关闭后，向管道写入数据会触发`SIGPIPE`信号，默认行为是终止进程
- 如果`SIGPIPE`被忽略，`write()`系统调用会返回`EPIPE`错误(`errno=32`)

**（2）管道的生命周期**：
管道只有在两端都打开时才能正常工作，任何一端关闭都会影响另一端的操作。

# TCP-socket通信`SIGPIPE`信号的产生
在 TCP 通信双方中，为了描述方便，以下将通信双方用 A 和 B 代替。当 A “关闭”连接时，若 B 继续给 A 发数据，根据 TCP 协议的规定，B 会收到 A 的一个 RST 报文响应，如 B 继续再往这个服务器发送数据，系统会产生一个 **SIGPIPE** 信号给该 B 进程，告诉该进程这个连接已经断开了，不要再写了。系统对 **SIGPIPE** 信号的默认处理行为是让 B 进程退出。

![](attachments/Pasted%20image%2020250527194748.png)

上图是 TCP 通信四次挥手的示意图，TCP 通信是全双工的信道，可以看作两条单工信道， TCP 连接两端的两个端点各负责一条。
当对端“关闭”时, 虽然本意是关闭整个两条信道，但本端只是收到 FIN 包。
按照 TCP 协议规定的语义，表示对端只是关闭了其所负责的那一条单工信道，虽然不再发送数据，但仍然可以继续接收数据。
也就是说，因为 TCP 协议的限制，==收到 FIN 包时，通信一方无法获知对端的 socket 是调用了 `close` 还是 `shutdown`==。

```c
int shutdown(int socket, int how);

shutdown 函数的参数 how 可以设置为 SHUT_RD、SHUT_WR 或 SHUT_RDWR;
分别用于表示关闭收通道、通道发通道或者同时关闭收发通道。
```

# 问题范例
## 背景
在前几天，我们在灰度上线时遇到了一个服务程序闪退的问题。最后排查的结果是因为一个小小的网络 SIGPIPE 信号导致的这个严重问题。
该新服务在线上遇到了崩溃的问题。而且崩溃还不是因为它自己，而是它依赖的另一个业务进程热升级的时候出现的。只要对该依赖热升级，就会导致该新服务崩溃退出，进而导致线上 SLA 出现较为严重的下降。

**初步分析**
遇到这种问题，大家第一反应是看日志。但不幸的是在业务日志中没有找到任何线索。然后我的思路是找 coredump 文件单步调试一下，看看崩溃发生在代码的哪一行，结果发现这次崩溃连 core 文件都没有留下，悄无声息的就消失了。

## 概述
接下来的文章我分三大部分给大家讲解：
- SIGPIPE 信号是如何发生的，带大家看看为什么连接异常会导致 SIGPIPE 的发生
- 内核 SIGPIPE 信号处理流程，带大家看看为什么内核默认遇到 SIGPIPE 时会将应用给杀死
- 应用层该如何应对 SIGPIPE，带大家看语言运行时以及我们自己的程序如何规避该问题

## SIGPIPE 信号如何发生
### 基础知识
我们知道，TCP 三次握手成功后会在内核中生成一个 socket 内核对象（open file description），通过该内核对象来表示一条 TCP 连接(file descriptor)。
但内核对象是不允许我们随便访问的。我们平时在用户态程序中看到的 socket fd 其实只是一个句柄(file descriptor)而已，并不是真正的 socket 对象（open file description）。

### 问题
![](attachments/Pasted%20image%2020250527200747.png)

假如由于网络、对端重启等问题这条 TCP 连接断开了。此时我们的用户态程序可能根本就是**不知情或者还没有来得及知情**。本端依然在调用 `send、write` 等系统调用往 `socket` 里面发送数据。

### 分析
如果对端彻底关闭，当本端第二次调用 `write/send` 方法时，当数据包发送过程走到内核中的时候，内核是知道这个 `socket` 已经断开了的「第一次调用 `write/send` 方法成功之后，然后收到了对端的`RST`报文」，然后就会给当前进程发送一个 SIGPIPE 信号。

### 内核代码流程
我们来看下具体的源码。内核的发送会走到 `do_tcp_sendpages` 函数，在这里内核如果发现该 `socket` 已经 在这种情况下，会调用 `sk_stream_error` 函数。
```c
//file:net/core/stream.c  
ssize_t do_tcp_sendpages(struct sock *sk, struct page *page, int offset,  
    size_t size, int flags)  
{  
 ......  
 err = -EPIPE;  
 if (sk->sk_err || (sk->sk_shutdown & SEND_SHUTDOWN))  
  goto out_err;  
out_err:  
 return sk_stream_error(sk, flags, err);  
}
```

`sk_stream_error` 函数主要工作就是给正在 `current`（发送数据的进程）发送一个 `SIGPIPE` 信号。
```c
int sk_stream_error(struct sock *sk, int flags, int err)  
{  
 ......  
 if (err == -EPIPE && !(flags & MSG_NOSIGNAL))  
  send_sig(SIGPIPE, current, 0);  
 return err;  
}
```

### 原因小结
对一个对端已经**彻底关闭**的`socket`调用两次`write`, 第一次可能会收到`rst`报文，第二次将会生成`SIGPIPE`信号, 该信号默认结束进程。
具体就是：如果对端彻底关闭，本端第一次调用 `write/send` 方法时，如果发送缓冲没问题，会返回正确写入（即 `write/send` 函数返回值大于 0），但发送的报文会导致对端回应 RST 报文。然后再次尝试调用 `write/send` 函数时，如果`SIGPIPE`没有被忽略，会因产生 **SIGPIPE** 信号导致进程退出。


## 内核 SIGPIPE 信号处理流程
上一节我们看到如果遇到网络连接异常断开，内核会给当前进程发送一个 SIGPIPE 信号。那么为啥这个信号就能把服务程序给搞崩而且没留下 coredump 文件呢？
简单来说，这是 Linux 内核对 SIGPIPE 信号处理的默认行为。

### 分析
目标进程每当从内核态返回用户态的过程中，会检测是否有挂起的信号。如果有信号存在，就会进入到信号的处理过程中，会执行到 `do_notify_resume`，然后再进到核心函数 `do_signal`。

```c
//file:arch/x86/kernel/signal.c  
static void do_signal(struct pt_regs *regs)  
{  
 struct ksignal ksig;  
 ...  
 if (get_signal(&ksig)) {  
  /* Whee!  Actually deliver the signal.  */  
  handle_signal(&ksig, regs);  
  return;  
 }  
 ...  
}
```
在 `do_signal` 主要包含 `get_signal` 和 `handle_signal` 两个操作。

内核在 `get_signal` 中是获取一个信号。值得注意的是，内核获取到信号后，还会判断信号的关联行为。
(1) 如果发现这个信号内核可以处理，内核直接就操作了。
(2) 如果内核发现获得到的信号内核需要交接给用户态程序处理，才会在 `get_signal` 函数中返回。接着再把信号交给 `handle_signal` 函数，由该函数来为用户空间准备好处理信号的环境，进行后面的处理。

服务程序在收到 SIGPIPE 会导致进程崩溃的关键就藏在这个 `get_signal` 函数里。
```c
//file:kernel/signal.c  
bool get_signal(struct ksignal *ksig)  
{  
 ...  
 for (;;) {  
  // 1.取出信号  
  signr = dequeue_synchronous_signal(&ksig->info);  
  if (!signr)  
   signr = dequeue_signal(current, &current->blocked,  
           &ksig->info, &type);  
  
  // 2.判断用户进程是否为信号配置了 handler  
  // 2.1 如果是 SIG_IGN(ignore的缩写)，就跳过  
  if (ka->sa.sa_handler == SIG_IGN)   
   continue;  
  
  // 2.3 判断如果不是 SIG_DFL(default的缩写)，  
  //     则证明用户定义了处理函数，break 退出循环后返回信号对象  
  if (ka->sa.sa_handler != SIG_DFL) {  
   ksig->ka = *ka;  
   ...  
   break;   
  }  
  
  // 3.接下来就是内核的默认行为了  
  ......  
 }  
out:  
 ksig->sig = signr;   
 return ksig->sig > 0;  
}
```

在 `get_signal` 函数里主要做了三件事。
- (1) 通过 `dequeue_xxx` 函数来获取一个信号
- (2) 判断下用户进程是否为信号配置了 handler。如果用户配置的是 `SIG_IGN` 直接跳过就行了，如果配置了处理函数，`get_signal` 就会将信号返回交给后面的流程交给用户态程序执行。
- (3) 如果用户没配置 `handler`，则会进入到内核默认行为中。

由于我们的服务程序没对 SIG_PIPE 信号配过任何处理逻辑，所以 `get_signal` 在遇到 `SIG_PIPE` 时会进入到第三步 -- 内核默认行为处理。
我们来继续看看，内核的默认行为究竟是啥样的。
```c
//file:kernel/signal.c  
bool get_signal(struct ksignal *ksig)  
{  
 ...  
 for (;;) {  
  // 1.取出信号  
  ......  
  
  // 2.判断信号是否配置了 handler  
  ......  
  
  // 3.接下来就是内核的默认行为了  
  // 3.1 如果是可以忽略的信号，直接跳过  
  if (sig_kernel_ignore(signr)) /* Default is nothing. */  
   continue;  
  
  // 3.2 判断是否是暂停执行信号，是则暂停其运行  
  if (sig_kernel_stop(signr)) {  
   do_signal_stop(ksig->info.si_signo)  
  }  
  
  fatal:  
  // 3.3 判断是否需要 coredump  
  //     coredump 会杀死进程下的所有线程，并生成 coredump 文件  
  if (sig_kernel_coredump(signr)) {  
   do_coredump(&ksig->info);  
  }  
  
  // 3.4 对于非以上情形的信号  
  //     直接让进程下所有线程退出，并且不生成coredump  
  do_group_exit(ksig->info.si_signo);  
 }  
 ......  
}
```

#### 内核中信号的默认处理
内核默认行为大概是分成四种。
**（1）第一种是默认要忽略的信号**。
从内核源码里可以看到 `SIGCONT、SIGCHLD、SIGWINCH` 和 `SIGURG`，这几个信号内核都是默认忽略的。
```c
//file: include/linux/signal.h  
#define sig_kernel_ignore(sig)  siginmask(sig, SIG_KERNEL_IGNORE_MASK)  
#define SIG_KERNEL_IGNORE_MASK (\  
        rt_sigmask(SIGCONT)   |  rt_sigmask(SIGCHLD)   | \  
 rt_sigmask(SIGWINCH)  |  rt_sigmask(SIGURG)    )
```


**（2）第二种是暂停信号**。
内核对 `SIGSTOP、SIGTSTP、SIGTTIN、SIGTTOU` 这几个信号的默认行为是暂停进程运行。

各个 `IDE` 中集成的代码断点调试器就是使用 `SIGSTOP` 信号来工作的。调试器给被调试进程发送 `SIGSTOP` 信号，让其进入停止状态。等到需要继续运行的时候，再发送 `SIGCONT` 信号让被调试进程继续运行。调试器通过 `SIGSTOP` 和 `SIGCONT` 等信号将被调试进程玩弄于股掌之间！
```c
//file: include/linux/signal.h  
#define sig_kernel_stop(sig)  siginmask(sig, SIG_KERNEL_STOP_MASK)  
#define SIG_KERNEL_STOP_MASK (\  
 rt_sigmask(SIGSTOP)   |  rt_sigmask(SIGTSTP)   | \  
 rt_sigmask(SIGTTIN)   |  rt_sigmask(SIGTTOU)   )
```

**（3）第三种是需要终止程序运行，并生成 coredump 文件的信号**
通过源码我们可以看到 `SIGQUIT、SIGILL、SIGTRAP、SIGABRT、SIGABRT、SIGFPE、SIGSEGV、SIGBUS、SIGSYS、SIGXCPU、SIGXFSZ` 这些信号的默认行为走这个逻辑。

我们以 `SIGSEGV` 为例，当应用程序试图访问空指针、数组越界访问等无效的内存操作时，内核会给当前进程发送 `SIGSEGV` 信号。
内核对于这些信号的默认行为就是会调用 `do_coredump` 内核函数。这个函数会杀死目标程序所有线程的运行，并生成 `coredump` 文件。
我们线上遇到的绝大部分程序崩溃都是这一类。
```c
//file: include/linux/signal.h  
#define sig_kernel_coredump(sig) siginmask(sig, SIG_KERNEL_COREDUMP_MASK)  
#define SIG_KERNEL_COREDUMP_MASK (\  
        rt_sigmask(SIGQUIT)   |  rt_sigmask(SIGILL)    | \  
 rt_sigmask(SIGTRAP)   |  rt_sigmask(SIGABRT)   | \  
        rt_sigmask(SIGFPE)    |  rt_sigmask(SIGSEGV)   | \  
 rt_sigmask(SIGBUS)    |  rt_sigmask(SIGSYS)    | \  
        rt_sigmask(SIGXCPU)   |  rt_sigmask(SIGXFSZ)   | \  
 SIGEMT_MASK           )
```

**（4）第四种是需要退出程序运行，但是不生成 coredump文件**
看了这么多信号名了，还是找不到我们开篇提到的 SIGPIPE，好气！！！
最后仔细看完代码以后，发现对于非上面提到的信号外，对于其它的所有信号包括 `SIGPIPE` 的默认行为都是调用 `do_group_exit`。这个内核函数的行为也是杀死进程下的所有线程，但**不生成 coredump 文件！！！**

## 应用层如何应对 SIGPIPE
### 程序退出的原因小结
看完前面两节，我们彻底弄明白了为什么我们的应用程序会崩溃了。
事故大体逻辑是这样的：
- 1.服务依赖的程序热升级的时候有连接异常断开
- 2.服务并不知道连接异常，还是正常向连接里发送数据
- 3.内核在处理数据发送时发现，该连接已经异常中断了，直接给应用程序发送一个 `SIGPIPE` 信号
- 4.服务程序会进入到信号处理流程中
- 5.由于应用程序未对 `SIGPIPE` 定义处理逻辑，所以走的是内核默认行为
- 6.内核对于 `SIGPIPE` 的默认行为是终止程序运行，但不生成 `coredump` 文件

### 解决方法
#### 方法一：忽略 `SIGPIPE`信号
```c
signal(SIGPIPE, SIG_IGN);
```
==这样设置后，第二次调用 `write/send` 方法时，会返回 `-1`，同时 `errno` 错误码被置为 `EPIPE`==，程序便能知道对端已经关闭。

#### 方法二：`send/write`添加`MSG_NOSIGNAL`的flag
如下所示，`sk_stream_error` 中，有如下的判断逻辑。
```c
int sk_stream_error(struct sock *sk, int flags, int err)  
{  
 ......  
 if (err == -EPIPE && !(flags & MSG_NOSIGNAL))  
  send_sig(SIGPIPE, current, 0);  
 return err;  
}
```

##### 范例
```c
/* send "n" bytes to a descriptor */
ssize_t send_n(int fd, const void *vptr, size_t n, int flags)
{
    size_t nleft;
    ssize_t nwritten;
    const char *ptr;

    ptr = vptr;
    nleft = n;

    while (nleft > 0) {
        if ((nwritten = send(fd, ptr, nleft, flags)) <= 0) {
            if (nwritten < 0 && (errno == EINTR || errno == EAGAIN))
                nwritten = 0;       /* and call send() again */
            else
                return (-1);        /* error */
        }
        nleft -= nwritten;
        ptr += nwritten;
    }

    return (n);
}

#define MSG_NOSIGNAL    0x80000         /* do not generate SIGPIPE on EOF */

static inline int sockopt_msg_send(int clt_fd,
        const struct dpvs_sock_msg *hdr,
        const char *data, int data_len)
{
    int len, res;

    if (!hdr) {
        fprintf(stderr, "[%s] empty socket msg header\n", __func__);
        return -ESOCKOPT_INVAL;
    }

    len = sizeof(struct dpvs_sock_msg);
    res = send_n(clt_fd, hdr, len, MSG_NOSIGNAL);
    if (len != res) {
        fprintf(stderr, "[%s] socket msg header send error -- %d/%d sent\n",
                __func__, res, len);
        return -ESOCKOPT_IO;
    }

    if (data && data_len) {
        res = send_n(clt_fd, data, data_len, MSG_NOSIGNAL);
        if (data_len != res) {
            fprintf(stderr, "[%s] scoket msg body send error -- %d/%d sent\n",
                    __func__, res, data_len);
            return -ESOCKOPT_IO;
        }
    }

    return 0;
}
```

## 小结
（1）没`coredump`是怎么定位的这个问题的呢？
在测试环境可以复现该问题，通过`strace`命令发现了`SIGPIPE`信号；
`strace`命令不光可以跟踪系统调用，也可以跟踪信号的。用法是`strace -e signal`;
注：用`perf trace`替代`strace` ，性能开销小，对程序影响小，还能够把堆栈打印出来。

（2）==进程退出问题基本都是和各种信号有关==。


# 参考
```bash
# 我的服务程序被 SIGPIPE 信号给搞崩了！
https://mp.weixin.qq.com/s/WpYW0E_b-8ktsFBpiR_ZzQ

# 【常见报错】"Broken pipe"错误：你的Shell程序为何突然崩溃？资深运维来揭秘！
https://mp.weixin.qq.com/s/peDLhcEsVeeK4auFBFV4yA
```