```table-of-contents
```
# 介绍
当前主流高性能的I/O模型的实现本质是**进程/线程/协程和 I/O多路复用的一个封装**，让用户只要聚焦业务处理，屏蔽网络请求处理的细节。

reactor模型是一种**事件驱动**的**应用层**I/O模型，基于**分而治之**和**事件驱动**的思想，致力于构建一个高性能的可伸缩的I/O模型。

# 基本组件
reactor模型无论何种实现，都会包含几个基本组件：

- **reactor**: 负责I/O多路复用系统调用以及请求的分派(dispatch)。dispatch主要是根据event类型交给特定的handler处理
- **acceptor**: 应用层没有accept的请求交给acceptor先进行accept系统调用。
- **handler**: 各种类型的handler，处理对应的event

# reactor单线程模型

## 应用


# 参考
```bash
# 高性能网络编程之 Reactor 网络模型（彻底搞懂）
https://juejin.cn/post/7092436770519777311

# 高性能网络模式：Reactor 和 Proactor
https://xiaolincoding.com/os/8_network_system/reactor.html#%E6%BC%94%E8%BF%9B

# Linux I/O模型：从阻塞调用到io_uring
https://www.iserica.com/posts/linux-io-from-blocking-to-uring/
```