```table-of-contents
```
# 查看
## /proc/net/sockstat

## `SO_ERROR`
### `so_error`变量
当一个套接字`socket`上发生错误时，内核协议中的协议模块将此套接字的名为`so_error`的变量设为标准的`Unix Exxx`值中的一个，我们称它为该套接字的待处理错误（`pending error`），`so_error`为0时表示没有错误发生。

进程然后可以通过访问套接字选项`SO_ERROR`获取`so_error`的值。由`getsockopt`返回的整数值就是该套接字的待处理错误。`so_error`随后由内核复位为0。



### `so_error` 和 `errno`

 当进程调用`read`且没有数据返回时，如果`so_error`为非0值，那么`read`返回`-1`且`errno`被置为`so_error`的值。`so_error`随后被复位为0。
 如果该套接字上有数据在排队等待读取，那么`read`返回那些数据而不是返回错误条件。
如果在进程调用`write`时`so_error`为非0值，那么`write`返回`-1`且`errno`被设为`so_error`的值。`so_error`随后被复位为0。

注：这是我们遇到的第一个可以获取但不能设置的套接字选项。

### 使用场景
#### 非阻塞 connect 的结果判定
当你使用非阻塞（Non-blocking）的 `connect` 时，函数会立即返回 `-1` 且 `errno == EINPROGRESS`。此时，你通常会用 `epoll` 或 `select` 监听该 FD 的**写事件**。

**妙用之处：** 当 `epoll` 提醒你该 FD 可写时，并不一定代表连接成功了。也有可能是连接失败（比如目标端口没开，收到 RST）。
 **判定逻辑：** 此时必须调用 `getsockopt(..., SO_ERROR, &err, ...)`。
- 如果 `err == 0`：连接真正成功。
- 如果 `err != 0`：连接失败，`err` 的值就是失败的原因（如 `ECONNREFUSED`）。

#### 区分“本地错误”与“远端错误”

- **本地错误**：通常由系统调用直接返回（比如 `EBADF` 无效句柄，`EFAULT` 内存错误）。
- **异步（远端）错误**：比如对端发送了 RST，或者中间路由器返回了不可达。这些错误是异步到达内核的。

**妙用之处：** `SO_ERROR` 专门负责捕获这些**异步产生的远端错误**。它可以帮你区分：到底是你的代码写错了（本地错误），还是网络/对端出问题了（异步错误）。

#### 配合边缘触发（ET）模式进行容错

在 `epoll` 的 ET 模式下，如果事件触发了，但你处理时出错了。

**妙用之处：** 有些高性能服务器在 `epoll_wait` 返回 `EPOLLERR` 或 `EPOLLHUP` 时，并不会直接 `close`，而是先用 `SO_ERROR` 记录一下到底发生了什么错误。这对于线上故障排查和监控告警极其有用——你可以精确统计出有多少连接是因为“超时”断开，有多少是因为“被重置（RST）”断开。

#### ICMP不可达错误

当你的 TCP 连接在运行过程中收到 ICMP 不可达报文时，`getsockopt` 配合 `SO_ERROR` 会根据 ICMP 报文的具体类型（Type）和代码（Code），将其转换为标准的 C 语言 `errno` 错误码。

|**ICMP 报文内容 (Type 3)**|**对应 SO_ERROR 获取的值**|**场景描述**|
|---|---|---|
|**Network Unreachable** (Code 0)|**`ENETUNREACH`**|路由器找不到通往目标网络的路由。|
|**Host Unreachable** (Code 1)|**`EHOSTUNREACH`**|路由存在，但在目标网段内找不到该主机（如 ARP 失败）。|
|**Port Unreachable** (Code 3)|**`ECONNREFUSED`**|目标主机存在，但目标端口没有程序在监听（通常对 UDP 常见）。|
|**Fragmentation Needed** (Code 4)|**`EMSGSIZE`**|报文超过了路径 MTU，且设置了 DF（不分片）位。|
|**Admin Prohibited** (Code 13)|**`EACCES`** 或 **`EPERM`**|路径上的防火墙（ACL）拦截了该报文。|

##### `write` 可能不报错，而 `SO_ERROR` 能拿到错误

TCP 协议栈处理 ICMP 是**异步**的：
1. **发送端**：你调用 `write`，内核只是把数据放进了发送缓冲区（Send Buffer），此时返回成功。
2. **网络中**：数据包飞到一半，某个路由器发现路断了，回了一个 ICMP Host Unreachable。
3. **内核态**：内核收到 ICMP 包，发现它对应你这个 `new_fd` 的序列号，于是把 `EHOSTUNREACH` 记录在套接字的“待处理错误”变量里。
4. **用户态**：你下次通过 `epoll_wait` 会收到一个 `EPOLLERR` 事件。
5. **此时调用 `getsockopt(..., SO_ERROR, ...)`**：你就能立刻拿到 `EHOSTUNREACH`。



# 参考
```c

```