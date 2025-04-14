```table-of-contents
```
# 介绍
**RSocket**（RDMA Socket）是一个面向 RDMA（Remote Direct Memory Access）技术的高性能用户空间通信接口，它简化了 RDMA 应用开发，并为开发者提供了类似于 BSD 套接字的编程体验，但具有 RDMA 的高性能特点。它利用 RDMA 的高效数据传输能力，同时屏蔽了一些 RDMA Verbs 的复杂性。

rsocket是基于 RDMA 的协议，支持应用程序的套接字级 API（socket-API）。rsocket API 是 旨在匹配相应套接字调用的行为，除非另有说明。rsocket 函数匹配套接字调用的名称和函数签名，其中 所有函数调用都以 'r' 为前缀的异常。

# rsocket 接口
```c
int rsocket(int domain, int type, int protocol);  
int rbind(int socket, const struct sockaddr *addr, socklen_t addrlen);  
int rlisten(int socket, int backlog);  
int raccept(int socket, struct sockaddr *addr, socklen_t *addrlen);  
int rconnect(int socket, const struct sockaddr *addr, socklen_t addrlen);  
int rshutdown(int socket, int how);  
int rclose(int socket);  
  
ssize_t rrecv(int socket, void *buf, size_t len, int flags);  
ssize_t rrecvfrom(int socket, void *buf, size_t len, int flags,  
    struct sockaddr *src_addr, socklen_t *addrlen);  
ssize_t rrecvmsg(int socket, struct msghdr *msg, int flags);  
ssize_t rsend(int socket, const void *buf, size_t len, int flags);  
ssize_t rsendto(int socket, const void *buf, size_t len, int flags,  
  const struct sockaddr *dest_addr, socklen_t addrlen);  
ssize_t rsendmsg(int socket, const struct msghdr *msg, int flags);  
ssize_t rread(int socket, void *buf, size_t count);  
ssize_t rreadv(int socket, const struct iovec *iov, int iovcnt);  
ssize_t rwrite(int socket, const void *buf, size_t count);  
ssize_t rwritev(int socket, const struct iovec *iov, int iovcnt);  
  
int rpoll(struct pollfd *fds, nfds_t nfds, int timeout);  
int rselect(int nfds, fd_set *readfds, fd_set *writefds,  
     fd_set *exceptfds, struct timeval *timeout);  
  
int rgetpeername(int socket, struct sockaddr *addr, socklen_t *addrlen);  
int rgetsockname(int socket, struct sockaddr *addr, socklen_t *addrlen);  
  
#define SOL_RDMA 0x10000  
enum {  
 RDMA_SQSIZE,  
 RDMA_RQSIZE,  
 RDMA_INLINE,  
 RDMA_IOMAPSIZE,  
 RDMA_ROUTE  
};  
  
int rsetsockopt(int socket, int level, int optname,  
  const void *optval, socklen_t optlen);  
int rgetsockopt(int socket, int level, int optname,  
  void *optval, socklen_t *optlen);  
int rfcntl(int socket, int cmd, ... /* arg */ );  
  
off_t riomap(int socket, void *buf, size_t len, int prot, int flags, off_t offset);  
int riounmap(int socket, void *buf, size_t len);  
size_t riowrite(int socket, const void *buf, size_t count, off_t offset, int flags);
```

# rsocket 和 RDMAcm
## rdma_cm
RDMAcm，全称为`RDMA communication manager`，是对verbs API的一个封装调用库。是 RDMA（Remote Direct Memory Access）编程模型中一个关键的通信管理组件，主要负责管理 RDMA 连接的创建、配置和拆除。

`rdma_cm` 封装了底层复杂的地址解析和连接管理逻辑，降低了开发难度。支持多种 RDMA 技术（如 InfiniBand、RoCE、iWARP），提供统一的接口。

## rsocket 和 RDMAcm
在 rdma_cm 基础上，一些公司和社区又推出了 rsocket。
rsocket 由 Mellanox（现为 NVIDIA）主导设计，并通过 Linux 社区和 OpenFabrics Alliance 进行开源支持和标准化。

rsocket的API近似于socket API，如`rdma_connect，rdma_listen，rdma_accept`等，只是作为部分verbs API的一个简化使用方案而已。

**rsocket是包含在librdmacm中的一个工具，可以把RDMA使用封装为socket接口，替代系统默认的socket**。从而使得应用程序在不修改代码的情况下使用RDMA。

# go使用rsocket
为什么不把这个接口引入到 Go 生态圈呢?

> 比如：github.com/smallnest/rsocket

rsocket 的接口简单，所以很容易通过 CGO 封装成 Go 库，这个库smallnest/rsocket[6]就是一个封装库。 它提供了大部分常用的函数比如`Socket`、`Bind`、`Listen`、`Accept`、`Connect`、`Read`、`Write`。

## 范例
使用传统 socket 的概念，你也不用知道 CQ、SQ、CQ、GID、LID、协商等各种概念。
### server端
```go
package main

import (
 "fmt"
 "log"
 "net"
 "syscall"

 "github.com/smallnest/rsocket"
)

func main() {
 // 创建RDMA socket
 fd, err := rsocket.Socket(rsocket.AF_INET, rsocket.SOCK_STREAM, 0)
 if err != nil {
  log.Fatal(err)
 }
 defer rsocket.Close(fd)

 // 绑定到地址
 addr := &net.TCPAddr{IP: net.ParseIP("0.0.0.0"), Port: 8000}
 sa := &syscall.SockaddrInet4{
  Port: addr.Port,
 }
 copy(sa.Addr[:], addr.IP.To4())

 if err := rsocket.Bind(fd, sa); err != nil {
  log.Fatal(err)
 }

 // 监听连接
 if err := rsocket.Listen(fd, 128); err != nil {
  log.Fatal(err)
 }
 fmt.Printf("服务器正在监听 :%d\n", addr.Port)

 for {
  // 接受新的连接
  clientFd, clientAddr, err := rsocket.Accept(fd)
  if err != nil {
   log.Printf("接受连接失败: %v\n", err)
   continue
  }

  // 处理新的连接
  go handleConnection(clientFd, clientAddr)
 }
}

func handleConnection(clientFd int, clientAddr syscall.Sockaddr) {
 defer rsocket.Close(clientFd)

 // 将clientAddr转换为更易读的格式
 addr, ok := clientAddr.(*syscall.SockaddrInet4)
 if ok {
  ip := net.IPv4(addr.Addr[0], addr.Addr[1], addr.Addr[2], addr.Addr[3])
  fmt.Printf("新的客户端连接: %s:%d\n", ip.String(), addr.Port)
 }

 // 读取客户端数据
 buffer := make([]byte, 1024)
 n, err := rsocket.Read(clientFd, buffer)
 if err != nil {
  log.Printf("读取数据失败: %v\n", err)
  return
 }
 fmt.Printf("收到客户端消息: %s\n", buffer[:n])

 // 发送响应
 response := []byte("Server received your message!")
 n, err = rsocket.Write(clientFd, response)
 if err != nil {
  log.Printf("发送响应失败: %v\n", err)
  return
 }
 fmt.Printf("发送了 %d 字节的响应\n", n)
}
```
###  client端
```go
package main

import (
 "fmt"
 "log"
 "net"
 "syscall"

 "github.com/smallnest/rsocket"
)

func main() {
 // 创建RDMA socket
 fd, err := rsocket.Socket(rsocket.AF_INET, rsocket.SOCK_STREAM, 0)
 if err != nil {
  log.Fatal(err)
 }
 defer rsocket.Close(fd)
 // Create RDMA socket
 // 准备服务器地址
 serverAddr := &net.TCPAddr{IP: net.ParseIP("127.0.0.1"), Port: 8000}
 sa := &syscall.SockaddrInet4{
  Port: serverAddr.Port,
 }
 copy(sa.Addr[:], serverAddr.IP.To4())

 // 连接到服务器
 if err := rsocket.Connect(fd, sa); err != nil {
  log.Fatal("连接失败:", err)
 }
 fmt.Println("成功连接到服务器")

 // 发送数据
 message := []byte("Hello, RDMA Server!")
 n, err := rsocket.Write(fd, message)
 if err != nil {
  log.Fatal("发送数据失败:", err)
 }
 fmt.Printf("发送了 %d 字节的数据\n", n)

 // 接收响应
 buffer := make([]byte, 1024)
 n, err = rsocket.Read(fd, buffer)
 if err != nil {
  log.Fatal("接收数据失败:", err)
 }
 fmt.Printf("收到服务器响应: %s\n", buffer[:n])
}
```
# 参考
```bash
# 官方介绍
https://github.com/linux-rdma/rdma-core/blob/master/librdmacm/docs/rsocket

# rsocket：RDMA编程的傻瓜库
https://mp.weixin.qq.com/s/3jmYRsARNdqTxXF_kZw1eg
```