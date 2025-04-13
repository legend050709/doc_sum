```table-of-contents
```

# epoll事件模式
在 Linux 的 epoll 机制中，有边缘触发（Edge-Triggered, ET）模式与水平触发（Level-Triggered, LT）模式。核心区别在于事件通知的时机。

## LT 模式
只要文件描述符的状态符合条件（比如可读或可写），`epoll_wait()` 每次调用都会返回该事件。　
如果 `read()` 读取数据后，仍然有数据可读，`epoll` 会 继续通知。

## ET 模式
只在文件描述符的状态发生变化时通知一次，例如从不可读变为可读。　
如果 `read()` 读取数据后仍然有数据，但没有再次触发事件，`epoll` 不会再通知；应用程序必须自行处理。


## 阻塞 I/O 在 ET 模式下的致命问题

# epoll的结构以及函数
## 使用范例
## 数据结构
```c
   #include <sys/epoll.h>

   struct epoll_event {
       uint32_t      events;  /* Epoll events */
       epoll_data_t  data;    /* User data variable */
   };

   union epoll_data {
       void     *ptr;
       int       fd;
       uint32_t  u32;
       uint64_t  u64;
   };

   typedef union epoll_data  epoll_data_t;
```

说明：
```bash
 The data member of the epoll_event structure specifies data that
       the kernel should save and then return (via epoll_wait(2)) when
       this file descriptor becomes ready.
```
## 函数
```c
int epoll_create(int size);  
int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);  
```
### epoll_create
### epoll_create1
### epoll_ctl
### epoll_wait

## 事件
```c
/* Epoll event masks */
#define EPOLLIN		(__force __poll_t)0x00000001
#define EPOLLPRI	(__force __poll_t)0x00000002
#define EPOLLOUT	(__force __poll_t)0x00000004
#define EPOLLERR	(__force __poll_t)0x00000008
#define EPOLLHUP	(__force __poll_t)0x00000010
#define EPOLLNVAL	(__force __poll_t)0x00000020
#define EPOLLRDNORM	(__force __poll_t)0x00000040
#define EPOLLRDBAND	(__force __poll_t)0x00000080
#define EPOLLWRNORM	(__force __poll_t)0x00000100
#define EPOLLWRBAND	(__force __poll_t)0x00000200
#define EPOLLMSG	(__force __poll_t)0x00000400
#define EPOLLRDHUP	(__force __poll_t)0x00002000
```
参考：[man epoll_ctrl 说明](https://man7.org/linux/man-pages/man2/epoll_ctl.2.html)
### EPOLLIN
### EPOLLOUT
### EPOLLERR
### EPOLLRDHUP
# 一个进程中的epfd 和普通的fd的关系
## 两者是否会重叠
# 参考
```bash
# 如果这篇文章说不清epoll的本质，那就过来掐死我吧！ （1）
https://zhuanlan.zhihu.com/p/63179839

# 如果这篇文章说不清epoll的本质，那就过来掐死我吧！（2）
https://zhuanlan.zhihu.com/p/64138532

# 如果这篇文章说不清epoll的本质，那就过来掐死我吧！ （3）
https://zhuanlan.zhihu.com/p/64746509

# 揭开epoll面纱：Nginx，Redis等都在用的多路复用，到底是什么？
https://mp.weixin.qq.com/s/yvH5SaweAUnym6bQS3Y8Sg

# I/O 多路复用：select/poll/epoll 【小林coding】
https://www.xiaolincoding.com/os/8_network_system/selete_poll_epoll.html#%E6%80%BB%E7%BB%93

```