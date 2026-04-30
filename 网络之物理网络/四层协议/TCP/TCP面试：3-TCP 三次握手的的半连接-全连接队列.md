```table-of-contents
```
# 思维导图
![](attachments/Pasted%20image%2020251209120757.png)


# 什么是 TCP 半连接队列和全连接队列？

在 TCP 三次握手的时候，Linux 内核会维护两个队列，分别是：

- 半连接队列，也称 SYN 队列；
- 全连接队列，也称 accepet 队列；

服务端收到客户端发起的 SYN 请求后，**内核会把该连接存储到半连接队列**，并向客户端响应 SYN+ACK，接着客户端会返回 ACK，服务端收到第三次握手的 ACK 后，**内核会把连接从半连接队列移除，然后创建新的完全的连接，并将其添加到 accept 队列，等待进程调用 accept 函数时把连接取出来。**

![](attachments/Pasted%20image%2020251205121158.png)


不管是半连接队列还是全连接队列，都有最大长度限制，超过限制时，内核会直接丢弃，或返回 RST 包。

## 三次握手的流程

- **第一次握手：** 客户端向服务端发送 **SYN** 包以发起连接。客户端进入 **SYN_SENT** 状态。
    
- **第二次握手：** 服务端收到客户端的 SYN 请求后，服务端进入 **SYN_RECV** 状态。此时，**内核会将此连接存储到 SYN 队列中**，并向客户端回复 **SYN+ACK** 包。
    
- **第三次握手：** 客户端收到服务端的 SYN+ACK 后，回复 **ACK** 包，并进入 **ESTABLISHED** 状态。
    
- **连接建立完成（服务端）：** 服务端收到客户端的 ACK 包后，**内核会将该连接从 SYN 队列中移除，并将其添加到 Accept 队列中**。服务端进入 **ESTABLISHED** 状态。
    
- **应用层接收连接：** 当服务端应用程序调用 **accept** 函数时，会将连接从 Accept 队列中取出。

# 查看
## ss命令

## netstat命令

# 实战 - TCP 全连接队列溢出

# 实战 - TCP 半连接队列溢出



# 参考
```c
# TCP 半连接队列和全连接队列满了会发生什么？又该如何应对？（+++++++++）
https://mp.weixin.qq.com/s/2qN0ulyBtO2I67NB_RnJbg


# TCP SYN Queue and Accept Queue Overflow Explained
https://www.alibabacloud.com/blog/tcp-syn-queue-and-accept-queue-overflow-explained_599203

【小林的文章】
https://xiaolincoding.com/network/3_tcp/tcp_queue.html
https://cloud.tencent.com/developer/article/1638042

# TCP Backlog
https://ylgrgyq.github.io/2017/05/18/tcp-backlog/

# backlog参数对TCP连接建立的影
https://switch-router.gitee.io/blog/TCP-Backlog/

# 全连接队列/半连接队列溢出的统计的代码 [如何正确查看线上半/全连接队列溢出情况？]
https://blog.51cto.com/u_14328839/4781507

# 如何查看linux的半连接队列，全连接队列
https://www.cnblogs.com/kumufengchun/p/15893977.html
```