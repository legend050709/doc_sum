```table-of-contents
```
# 背景
因为使用read()将数据读到**不连续的内存**、使用write()将不连续的内存发送出去，要经过**多次的调用read、write。**

如果要从文件中读一片连续的数据至进程的**不同**区域，有两种方案：

- 使用read()一次将它们读至一个较大的缓冲区中，然后将它们分成若干部分复制到不同的区域；
- 调用read()若干次分批将它们读至不同区域。

但是多次系统调用+拷贝会带来较大的开销，所以UNIX提供了另外两个函数—readv()和writev()，它们**只需一次系统调用**就可以实现在文件和进程的**多个缓冲区之间**传送数据，免除了多次系统调用或复制数据的开销。

## 为什么要用 writev / readv
```bash
对比 write / read

(1) write
write(fd, buf1, len1);
write(fd, buf2, len2);
write(fd, buf3, len3);

(2) writev
struct iovec iov[3];
writev(fd, iov, 3);

```

**优势：**
- 减少系统调用次数
- 避免拼接大 buffer（零拷贝友好）
- 对 socket / pipe / 文件都有效
- 非常适合：
    - 协议头 + payload
    - 日志模块


# 基础
在 Linux C 网络编程中，`writev` 和 `readv` 被称为**分散/聚集 I/O (Scatter/Gather I/O)**。
readv函数将数据从文件描述符读到分散的内存块中，即==分散读==；
writev函数则将多块分散的内存数据一并写入文件描述符中，即==集中写==。
它们允许你在一次系统调用中处理多个不连续的内存缓冲区，这比多次调用 `write` 或 `read` 效率更高，因为==减少系统调用，以及用户态与内核态之间的上下文切换==。

![](attachments/Pasted%20image%2020251223174942.png)


## 核心数据结构：`struct iovec`
```c
#include <sys/uio.h>
struct iovec {
    void  *iov_base;    /* Starting address */
    size_t iov_len;     /* Number of bytes to transfer */
};
```

struct iovec定义了一个向量元素。通常，这个结构用作一个多元素的数组。对于每一个传输的元素，指针成员iov_base指向一个缓冲区，这个缓冲区是存放的是readv所接收的数据或是writev将要发送的数据。成员iov_len在各种情况下分别确定了接收的最大长度以及实际写入的长度。

## 核心函数
readv和writev函数用于在一次函数调用中读、写多个非连续缓冲区。有时也将这两个函数称为散布读（scatter read）和聚集写（gather write）。

```c
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```

(1) 参数：readv和writev的第一个参数fd是个**文件描述符**，第二个参数是指向**iovec**数据结构的一个指针，其中iov_base为缓冲区首地址，iov_len为缓冲区长度，参数iovcnt指定了iovec的个数。  
(2) 返回值：**函数调用成功时返回读、写的总字节数**；失败时返回`-1`并设置相应的`errno`。

![](attachments/Pasted%20image%2020251224103511.png)


# writev

`writev` 将多个缓冲区中的数据“聚集”起来，按顺序写入文件描述符。
readv 和 writev 这两个函数可以用于任何描述符，而不仅限于套接字。
另外 writev 是一个原子操作，意味着对于类似 UDP 协议，一次 writev 调用只产生单个 UDP 数据报。

**典型场景**：在 Web 服务器中，你需要发送一个 HTTP 头部（在内存 A 中）和一个文件内容（在内存 B 中）。使用 `writev` 可以一次性发送，而不需要先将它们拷贝到一个连续的大内存块中。
之前我们说过 TCP_NODELAY 套接字选项，一个 4 字节的 write 和一个 396 字节的 write 可能触发 Nagle 算法，首选方法之一是针对这两个缓冲区调用 writev。

```c
#include <sys/uio.h>

char *header = "HTTP/1.1 200 OK\r\nContent-Length: 5\r\n\r\n";
char *body = "Hello";

struct iovec iov[2];
iov[0].iov_base = header;
iov[0].iov_len = strlen(header);
iov[1].iov_base = body;
iov[1].iov_len = strlen(body);

ssize_t nwritten = writev(fd, iov, 2);
```

## writev和write对比

我们应当用尽量少的系统调用次数来完成任务。如果我们只写少量的数据，将会发现自己复制数据然后使用一次 write 会比用 writev 更合算。但也可能发现，我们管理自己的分段缓冲区会增加程序额外的复杂性成本，所以从性能成本的角度来看不合算。



# readv

`readv` 将从文件描述符读到的数据“分散”存储到多个预先指定的缓冲区中。
这里的读取虽然说是“向量化”，但实际上，缓冲区是按数组顺序处理的，也就是说，只有在`iov[0]`被填满之后，才会去填充`iov[1]`。
即：readv 总是先填满一个缓冲区，然后再填写下一个。关于文件偏移的更新，`readv`和`read`一样，在结束之后会更新文件偏移。

**典型场景**：当你已知输入数据的格式（例如前 4 字节是长度，后面是负载），你可以直接通过 `readv` 将它们分别读入不同的变量中。

```c
char hdr[4];
char payload[1024];

struct iovec iov[2];
iov[0].iov_base = hdr;
iov[0].iov_len = sizeof(hdr);
iov[1].iov_base = payload;
iov[1].iov_len = sizeof(payload);

ssize_t nread = readv(fd, iov, 2);
```

## 范例



# 注意
## 内存的释放

这是开发者最容易出错的地方。**`writev` 和 `readv` 都是阻塞/同步操作（除非 fd 设置为非阻塞）。**



### 对于 `readv` (接收数据)

- **调用返回后**：数据已经从内核拷贝到了你提供的 `iov_base` 内存中。
    
- **释放时机**：在处理完这些读到的业务数据之后，再释放内存。

### 对于 `writev` (发送数据)

- **调用返回后**：当 `writev` 返回时，意味着数据已经**从用户态缓冲区拷贝到了内核的发送缓冲区**。
    
- **释放时机**：只要 `writev` 返回，你就可以立即释放 `iovec` 数组以及其中 `iov_base` 指向的所有内存块。
    
- **注意**：返回并不代表数据已经到达对方机器，只代表内核“接管”了这些数据。
    

#### 释放时机
清理操作必须覆盖 **3 个点**：
- 发送完毕后主动释放。
- `writev` 系统调用返回非 `EAGAIN` 错误时释放。
- `epoll_wait` 捕获到 `EPOLLHUP` 或 `EPOLLERR` 时释放。



#### 特殊情况：部分写（Partial Write）

如果你在 `epoll` 边缘触发（ET）模式下使用 `writev`，情况会复杂得多：
如果 `writev` 返回 `EAGAIN` 或实际返回的写入的字节数少于要写入的字节数（==部分写==）。
此时**不能释放内存**。需要保存当前的进度（哪个 `iovec` 写了一半，还剩多少），等待 `epoll` 再次触发 `EPOLLOUT` 事件时，继续发送剩余部分。
==只有在所有数据完全发送完毕后，才能释放内存==。
因此，==始终检查返回值==。如果返回值小于所有 `iov_len` 的总和，说明发生了部分写，需要手动处理剩下的字节。

##### 场景
当你尝试通过 `writev` 发送 1000 字节，但内核缓冲区只剩 300 字节空间时，`writev` 会返回 300。此时：

- 第一个 `iovec` 可能发完了。
    
- 第二个 `iovec` 可能发了一半。
    
- 剩下的 `iovec` 完全没动。
    
- **你不能释放这些内存，且必须记录这复杂的断点。**

##### 思路
针对 `epoll` + `non-blocking` + `writev` 场景下的“部分写入（Partial Write）”问题，所谓的“完美解决方法”必须包含一个**状态机**或**进度追踪器**，以确保在缓冲区满时能原地挂起，并在内核缓冲区可用时从断点继续。

##### 范例一：带断点重传的发送逻辑（简单版，不太完美）

```c
#include <sys/uio.h>
#include <errno.h>
#include <stdlib.h>

// 用于追踪 writev 进度的结构体
typedef struct {
    struct iovec *iov; // 原始 iovec 数组
    int iov_cnt;       // 剩余未完全发送的 iov 数量
    int current_idx;   // 当前正在发送哪个 iov
} write_session_t;

// 处理 writev 逻辑的函数
// 返回 1 表示发送完毕，0 表示阻塞（等待下一次 EPOLLOUT），-1 表示出错
int do_writev_persistence(int fd, write_session_t *session) {
    while (session->current_idx < session->iov_cnt) {
        // 关键点：调用 writev 时，传入的是当前剩余部分的起始地址
        ssize_t nwritten = writev(fd, &session->iov[session->current_idx], 
                                  session->iov_cnt - session->current_idx);

        if (nwritten > 0) {
            // 更新进度：遍历 iovec 数组，看看发掉了多少
            while (session->current_idx < session->iov_cnt && nwritten > 0) {
                struct iovec *cur = &session->iov[session->current_idx];
                if (nwritten >= cur->iov_len) {
                    // 这个块完全发完了
                    nwritten -= cur->iov_len;
                    session->current_idx++;
                } else {
                    // 这个块发了一部分，修改 iov_base 和 iov_len 以备下次重试
                    cur->iov_base = (char *)cur->iov_base + nwritten;
                    cur->iov_len -= nwritten;
                    nwritten = 0; // 这一次系统调用的配额用完了
                }
            }
        } else if (nwritten == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 内核缓冲区满了，保持当前 session 状态，退出等待 epoll 再次触发
                return 0; 
            }
            return -1; // 真正出错（如连接断开）
        }
    }
    return 1; // 全部发送完毕
}
```



##### 范例二：epoll+ ET边缘触发+nonBlock + 部分写处理

在 **Edge Triggered (ET)** 模式下，`writev` 的处理必须非常严谨。因为 ET 模式下 `epoll_wait` 只会在缓冲区从“不可写”变为“可写”时触发一次。如果 `writev` 没发完且你没有把剩下的数据处理好，或者没有继续尝试发送直到返回 `EAGAIN`，你的程序就会“卡死”在该连接上，直到下一次对方发来数据。

我们需要一个能够记录 `iovec` 数组当前“断点”的结构。
```c
#include <sys/uio.h>
#include <errno.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

typedef struct {
    struct iovec *iov;   // 原始 iovec 数组的指针
    int iov_cnt;         // 初始总块数
    int current_idx;     // 当前处理到第几块
    size_t offset;       // 当前块内已发送的偏移量
} write_context_t;
```

下面的这个函数模拟了在 ET 模式下，如何处理 `writev` 的各种返回值。
```c
/**
 * @return  1: 全部发送完毕
 * 0: 缓冲区满 (EAGAIN), 需等待下次 EPOLLOUT
 * -1: 严重错误 (Connection reset, etc.)
 */
int do_writev_et(int fd, write_context_t *ctx) {
    while (ctx->current_idx < ctx->iov_cnt) {
        
        // 1. 构造“剩余部分”的临时 iovec 数组
        // 因为 writev 不支持偏移量，我们要手动调整第一个待发块的指针
        struct iovec active_iov[ctx->iov_cnt - ctx->current_idx];
        int active_cnt = ctx->iov_cnt - ctx->current_idx;
        
        for (int i = 0; i < active_cnt; i++) {
            active_iov[i] = ctx->iov[ctx->current_idx + i];
        }
        
        // 修正第一个块的起始位置（应用偏移量）
        active_iov[0].iov_base = (char *)active_iov[0].iov_base + ctx->offset;
        active_iov[0].iov_len -= ctx->offset;

        // 2. 发起系统调用
        ssize_t n = writev(fd, active_iov, active_cnt);

        if (n > 0) {
            // 3. 成功写入，更新进度
            size_t bytes_left = (size_t)n;
            while (bytes_left > 0 && ctx->current_idx < ctx->iov_cnt) {
                size_t current_block_remains = ctx->iov[ctx->current_idx].iov_len - ctx->offset;
                
                if (bytes_left >= current_block_remains) {
                    // 当前块完全发完
                    bytes_left -= current_block_remains;
                    ctx->current_idx++;
                    ctx->offset = 0; // 重置下一块的偏移量
                } else {
                    // 当前块只发了一部分
                    ctx->offset += bytes_left;
                    bytes_left = 0;
                }
            }
            // 继续 loop，尝试把剩下的发完，直到 EAGAIN
        } else if (n == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 内核缓冲区满了。在 ET 模式下，我们停止并等待下一次 EPOLLOUT。
                return 0;
            } else if (errno == EINTR) {
                // 被信号中断，继续尝试
                continue;
            } else {
                // 真正的错误
                perror("writev error");
                return -1;
            }
        } else {
            // n == 0 通常意味着对端关闭
            return -1;
        }
    }
    return 1; // 所有 iovec 块都发完了
}
```

```c
// 假设在某个事件处理逻辑中
if (events[i].events & EPOLLOUT) {
    write_context_t *ctx = (write_context_t *)events[i].data.ptr;
    int res = do_writev_et(fd, ctx);
    
    if (res == 1) {
        // 【重要】发完后，记得释放内存
        for(int i=0; i < ctx->iov_cnt; i++) {
            free(ctx->iov[i].iov_base);
        }
        free(ctx->iov);
        free(ctx);
        // 如果后续没有数据要发，记得把 EPOLLOUT 监听关掉，否则会触发 Busy Loop
        modify_epoll_event(epoll_fd, fd, EPOLLIN); 
    } else if (res == -1) {
        // 清理并关闭连接
        close_connection(fd, ctx);
    }
    // res == 0 的情况：什么都不做，继续等下次 EPOLLOUT
}
```

##### 范例三：范例二的基础上+无效free的问题

```c
#include <sys/uio.h>
#include <stdlib.h>
#include <errno.h>
#include <stdio.h>
#include <sys/epoll.h>
#include <unistd.h>

// 定义每一个待发送的 Buffer 单元
typedef struct {
    void *ptr;       // 实际数据内存块
    size_t len;      // 该块的总长度
    int free_it;     // 标志位：发完后是否需要 free()
} msg_block_t;

// 发送会话上下文
typedef struct {
    msg_block_t *blocks; // 消息块数组
    int block_cnt;       // 总块数
    int current_idx;     // 当前处理到第几块
    size_t offset;       // 当前块已发送的偏移量
} write_context_t;
```


```c
/**
 * 返回值说明:
 * 1: 全部数据发送完毕
 * 0: 缓冲区满 (EAGAIN)，等待下次 EPOLLOUT
 * -1: 发生错误（对端关闭或网络异常）
 */
int do_writev_et(int fd, write_context_t *ctx) {
    while (ctx->current_idx < ctx->block_cnt) {
        // 1. 构建本次系统调用的 iovec 数组
        int remaining_blocks = ctx->block_cnt - ctx->current_idx;
        struct iovec iov[remaining_blocks];
        
        for (int i = 0; i < remaining_blocks; i++) {
            msg_block_t *mb = &ctx->blocks[ctx->current_idx + i];
            if (i == 0) {
                // 第一块需要考虑之前的偏移量
                iov[i].iov_base = (char *)mb->ptr + ctx->offset;
                iov[i].iov_len = mb->len - ctx->offset;
            } else {
                iov[i].iov_base = mb->ptr;
                iov[i].iov_len = mb->len;
            }
        }

        // 2. 尝试写入
        ssize_t n = writev(fd, iov, remaining_blocks);

        if (n > 0) {
            // 3. 更新进度
            size_t bytes_sent = (size_t)n;
            while (bytes_sent > 0 && ctx->current_idx < ctx->block_cnt) {
                size_t current_block_remains = ctx->blocks[ctx->current_idx].len - ctx->offset;
                if (bytes_sent >= current_block_remains) {
                    bytes_sent -= current_block_remains;
                    ctx->current_idx++;
                    ctx->offset = 0;
                } else {
                    ctx->offset += bytes_sent;
                    bytes_sent = 0;
                }
            }
            // 继续循环，直到全部发完或遇到 EAGAIN
        } else if (n == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) return 0;
            if (errno == EINTR) continue;
            return -1; // 真正的错误
        } else {
            return -1; // n == 0
        }
    }
    return 1;
}
```

```c
/**
 * 释放会话资源
 */
void destroy_write_context(write_context_t *ctx) {
    if (!ctx) return;
    
    if (ctx->blocks) {
        for (int i = 0; i < ctx->block_cnt; i++) {
            // 只有标记了 free_it 的块才释放（防止释放了静态内存或栈内存）
            if (ctx->blocks[i].free_it && ctx->blocks[i].ptr) {
                free(ctx->blocks[i].ptr);
                ctx->blocks[i].ptr = NULL;
            }
        }
        free(ctx->blocks);
    }
    free(ctx);
    printf("Write context cleaned up successfully.\n");
}

void handle_event(int epoll_fd, struct epoll_event *ev) {
    int fd = ev->data.fd;
    write_context_t *ctx = (write_context_t *)ev->data.ptr;

    // A. 错误处理：收到 HUP 或 ERR
    if (ev->events & (EPOLLHUP | EPOLLERR)) {
        printf("Connection error on fd %d\n", fd);
        epoll_ctl(epoll_fd, EPOLL_CTL_DEL, fd, NULL);
        destroy_write_context(ctx);
        close(fd);
        return;
    }

    // B. 写就绪处理
    if (ev->events & EPOLLOUT) {
        int res = do_writev_et(fd, ctx);
        
        if (res == 1) {
            // 全部发完：清理资源，修改 epoll 移除写监听（防止忙循环）
            destroy_write_context(ctx);
            // 假设此时切回只听读事件
            struct epoll_event new_ev;
            new_ev.events = EPOLLIN | EPOLLET;
            new_ev.data.fd = fd;
            epoll_ctl(epoll_fd, EPOLL_CTL_MOD, fd, &new_ev);
        } else if (res == -1) {
            // 发送时报错：清理资源并关闭
            epoll_ctl(epoll_fd, EPOLL_CTL_DEL, fd, NULL);
            destroy_write_context(ctx);
            close(fd);
        }
        // res == 0：继续保持 EPOLLOUT 状态，等待下次触发
    }
}
```

**为什么需要 `free_it` 标志？** 
在高性能服务器中，Header 可能是动态生成的（需要 free），而 Body 可能是 `mmap` 的文件或静态字符串（不需要 free）。区分对待能防止 `Invalid Free` 错误。


## 原子性

`writev` 在写入时是“原子”的（对于文件而言），即数据会按顺序连续写入，不会被其他进程的 `write` 插入中间。
对于`readv`而言，它读取的文件内容永远是连续的，也就是说不会因为文件偏移被别的线程改变而混乱。比如说，我们想将文件中的内容读入三块缓冲区中。如果我们是使用三次`read`，但是在第一次`read`结束之后，第二次`read`开始之前，另外一个线程对这个文件描述符的文件偏移进行了改变，那么接下来的两次`read`读出的数据与第一次`read`读出的数据是不连续的。但是，如果我们用`readv`，读出的数据一定是连续的。


## IOV_MAX
一次调用能处理的 `iovec` 数量有限制，Linux 上通常是 **1024**。
```bash
$ getconf -a| grep IOV
IOV_MAX                            1024
UIO_MAXIOV                         1024
_T_IOV_MAX
```

## 返回值处理
始终检查返回值。如果返回值小于所有 `iov_len` 的总和，说明发生了部分读写，你需要手动处理剩下的字节。

# pread和pwrite
```c
ssize_t preadv(int fd, const struct iovec *iov, int iovcnt,
			  off_t offset);

ssize_t pwritev(int fd, const struct iovec *iov, int iovcnt,
			   off_t offset);
```

read和pread是最基础的对文件读取的系统调用。read会从描述符为fd的文件中读取count个字节存入buf中，而pread则是从描述符为fd的文件中，从offset位置开始，读取count个字节存入buf中。如果读取成功，这两个系统调用都将返回读取的字节数。

## read和pread对比
### 从哪开始读
pread没有问题，就是从offset的位置开始读。而对于read，如果它读取的描述符对应的文件支持seek，那么它是从文件描述符中存储的文件偏移（file offset）处继续读。

对于支持seek的文件来说，read是从文件偏移的位置继续读，pread是从offset的位置开始读。

我们知道，更改文件偏移有单独的系统调用lseek，因此，如果我们要从某个特定的位置读取数据，可以lseek+read，也可以pread。但是，系统调用实际上是一个复杂的耗时操作，所以pread就用一次系统调用解决了两个系统调用的问题。


### 读多少

read和pread读取的字节数一定不大于count，但有可能小于count。假设说我们的二进制文件只有上述6个字节。那么，如果我们read了8个字节，后2个字节自然是无法被读取的。因此，只能读取到6个字节，read也将返回6。除此之外，还有很多可能会让read和pread读取的字节小于count。比如说，从一个终端读取（输入的字节小于其需求的字节），或者在读取时被某些信号中断。

此外，除了读的字节小于count之外，read和pread还有可能读取失败。此时的返回值将是-1。我们可以用errno查看其错误。
文件描述符不可读或无效（EBADF），buf不可使用（EFAULT），文件描述符是目录而非文件（EISDIR）等等，这些都有可能直接造成读取的错误。对于以非阻塞形式打开的文件，还可能返回EAGAIN或EWOULDBLOCK，详情请见open。

### 当read或pread读取结束后的工作
read会更新文件描述符中的文件偏移，它们读了多少字节，就向后移动多少字节。但是，值得注意的是，pread并不会更新文件偏移。pread不更新文件偏移这一点对于多线程的程序来说极其有用。我们知道，多条线程有可能共用同一个文件描述符，但文件偏移是存储在文件描述符中。如果我们在多线程中使用read，会导致文件偏移混乱；但是，如果我们使用pread，则会完满避免这个问题。

### 如何读
在Linux的哲学中，如何读并不是read和pread能决定的，而是由文件描述符本身决定的。文件描述符在创建的时候，就决定了它将被如何读取，比如说是否阻塞等等。

我们在上面提到，pread除了在多线程中发挥大作用之外，也可以将两次系统调用lseek+read化为一次系统调用。而这一节所讲的系统调用，则是更进一步。read和pread是将文件读取到一块连续内存中，那如果我们想要将文件读取到多块连续内存中（也就是说，有多块内存，内存内部连续，但内存之间不连续），就得多次使用这些系统调用，造成很大的开销。而**readv, preadv, preadv2**则是为了解决这样的问题。




# 参考
```bash

```