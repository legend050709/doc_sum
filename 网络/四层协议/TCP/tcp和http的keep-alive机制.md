```table-of-contents
```
# tcp的keep-alive
## 原理
定义一个时间段，在这个时间段内，如果没有任何连接相关的活动（即：没有数据发送），TCP 保活机制会开始作用，每隔一个时间间隔，发送一个探测报文。
该探测报文包含的数据非常少，如果连续几个探测报文都没有得到响应，则认为当前的 TCP 连接已经死亡，系统内核将错误信息通知给上层应用程序。

## keep-alive的设置
### 系统的Keepalive参数
**如果程序中的sock使能了`SO_KEEPALIVE`，但是没有设置具体的 参数，则使用系统默认的参数。如果程序中没有设置，那么就无法使用 TCP 保活机制**。

```bash
net.ipv4.tcp_keepalive_time=7200
net.ipv4.tcp_keepalive_intvl=75  
net.ipv4.tcp_keepalive_probes=9
```

- tcp_keepalive_time=7200：表示保活时间是 7200 秒（2小时），也就 2 小时内如果没有任何连接相关的活动，则会启动保活机制
- tcp_keepalive_intvl=75：表示每次检测间隔 75 秒；
- tcp_keepalive_probes=9：表示检测 9 次无响应，认为对方是不可达的，从而中断本次的连接。

也就是说在 Linux 系统中，最少需要经过 2 小时 11 分 15 秒才可以发现一个「死亡」连接。
![](attachments/Pasted%20image%2020240327201058.png)

### 程序设置
```bash
    // 启用TCP Keepalive选项
    int keepalive = 1;
    if (setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &keepalive, sizeof(keepalive)) == -1) {
        perror("setsockopt");
        exit(1);
    }
    
    // 设置Keepalive参数
    int keepidle = 15; // 空闲时间
    int keepinterval = 3;   // 探测间隔
    int keepcount = 4;      // 探测次数
    if (setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE, &keepidle, sizeof(keepidle)) == -1) {
        perror("setsockopt");
    }
    if (setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &keepinterval, sizeof(keepinterval)) == -1) {
        perror("setsockopt");
    }
    if (setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT, &keepcount, sizeof(keepcount)) == -1) {
        perror("setsockopt");
    }
```



## keep-alive的报文的特点
其实 keep-alive报文不携带数据，且 keep-alive报文 的 seq_num = 对方最后发送的ack报文的 ack_num -1 , 这样对方收到keep-alive报文，继续回复正确的 ack。

![](attachments/Pasted%20image%2020240327202126.png)

## keep-alive的问题以及解决
TCP keepalive 是 **TCP 层（内核态）** 实现的，它是给所有基于 TCP 传输协议的程序一个兜底的方案。

实际上，我们应用层可以自己实现一套探测机制，可以在较短的时间内，探测到对方是否存活。

比如，web 服务软件一般都会提供 `keepalive_timeout` 参数，用来指定 HTTP 长连接的超时时间。如果设置了 HTTP 长连接的超时时间是 60 秒，web 服务软件就会**启动一个定时器**，如果客户端在完后一个 HTTP 请求后，在 60 秒内都没有再发起新的请求，**定时器的时间一到，就会触发回调函数来释放该连接。**

![](attachments/Pasted%20image%2020240327202239.png)


# 其他
## 没有keep-alive的established的tcp的超时时间
### 问题
三次握手之后，tcp就处于established 状态。对应到用户态的程序，则此时 epoll 监听到 listen_fd 可读， server端是可以从全连接队列accept 一个 new_sock。 然后将 new_sock 加入到 epoll 中，但是client 一直没有数据的发送，并且client也不进行 close 或者shutdown调用（即：client 也不会发送 fin 或者 rst），那么 server上处于 established 状态的  new_sock 可以持续多久呢？

###  分析
1》如果 server 程序在编写的时候，对 new_sock 设置了  keep-alive；
那么server上每个 established 状态的 sock 会定期的 对外发送 keep-alive报文，client 如果正常，就会回复一个ack。一个client 不正常 或者网络不正常，那么就keep-alive 失败，达到一定次数之后，这个established 状态的 sock 就就会关闭。

2》如果 server 程序在编写的时候，对 new_sock 没有设置  keep-alive；
client 也不发送任何数据包，也不进行close （即没有发送fin，rst 包）。
server 端由于new_conn没有可读事件，也不会对 new_conn 执行 read（即使 read 发现 非阻塞的 fd 返回0，返回0的情况下，server 如果不调用 close ），也不进行close。
那么 server 上的这个 established 状态的 new_sock 就 一直存在，永不超时。

###  结论
Linux 内核对于  established 状态的 sock  没有设置超时时间。如果没有以下两种情况，则  established 状态的 sock 已知存在。
1> sock 的 状态发生了改变
比如：server端执行了 close/shutdown 系统调用（发送fin/rst 报文)， 改变了 sock的状态。
或者 client 发送 了fin/rst， 改变了 sock的状态

2> sock开启了 tcp的keep-alive，且 keep-alive不正常。

## 拔掉网线后， 原本的 TCP 连接还存在吗？
### 问题
**拔掉网线几秒，再插回去，原本的 TCP 连接还存在吗？**

可能有的同学会说，网线都被拔掉了，那说明物理层被断开了，那在上层的传输层理应也会断开，所以原本的 TCP 连接就不会存在的了。就好像， 我们拨打有线电话的时候，如果某一方的电话线被拔了，那么本次通话就彻底断了。

真的是这样吗？

上面这个逻辑就有问题。问题在于，错误的认为拔掉网线这个动作会影响传输层，事实上并不会影响。

实际上，TCP 连接在 Linux 内核中是一个名为 `struct socket` 的结构体，该结构体的内容包含 TCP 连接的状态等信息。当拔掉网线的时候，操作系统并不会变更该结构体的任何内容，所以 TCP 连接的状态也不会发生改变。


### 分析
我在我的电脑上做了个小实验，我用 ssh 终端连接了我的云服务器，然后我通过断开 wifi 的方式来模拟拔掉网线的场景，此时查看 TCP 连接的状态没有发生变化，还是处于 ESTABLISHED 状态。
![](attachments/Pasted%20image%2020240327194827.png)

拔掉网线这个动作并不会影响 TCP 连接的状态。
接下来，要看拔掉网线后，双方做了什么动作，要分场景来讨论：
- 拔掉网线后，有数据传输；
- 拔掉网线后，没有数据传输

### 拔掉网线后，有数据传输
在客户端拔掉网线后，服务端向客户端发送的数据报文会得不到任何的响应，在等待一定时长后，服务端就会触发**超时重传**机制，重传未得到响应的数据报文。

**（1）如果在服务端重传报文的过程中，客户端刚好把网线插回去了**
由于拔掉网线并不会改变客户端的 TCP 连接状态，并且还是处于 ESTABLISHED 状态，所以这时客户端是可以正常接收服务端发来的数据报文的，然后客户端就会回 ACK 响应报文。
此时，客户端和服务端的 TCP 连接依然存在的，就感觉什么事情都没有发生。

**（2）如果如果在服务端重传报文的过程中，客户端一直没有将网线插回去**
服务端超时重传报文的次数达到一定阈值后，内核就会判定出该 TCP 有问题，然后通过 Socket 接口告诉应用程序该 TCP 连接出问题了，于是服务端的 TCP 连接就会断开。

而等客户端插回网线后：
（2.1）如果客户端向服务端发送了数据
由于服务端已经没有与客户端相同四元祖的 TCP 连接了，因此服务端内核就会回复 RST 报文，客户端收到后就会释放该 TCP 连接。

（2.2）如果客户端不给服务器发送数据，且不存在 keep-alive
那么 客户端 的连接还是处于 ESTABLISHED 状态，一直不消失。

#### TCP 建立连接后无响应的retry
```bash
tcp retry相关的所有配置:
# sysctl -a |grep -i retr | grep tcp
net.ipv4.tcp_early_retrans = 3
net.ipv4.tcp_orphan_retries = 0
net.ipv4.tcp_retrans_collapse = 1
net.ipv4.tcp_retries1 = 3
net.ipv4.tcp_retries2 = 15
net.ipv4.tcp_syn_retries = 6
net.ipv4.tcp_synack_retries = 5
```
在 Linux 系统中，提供了一个叫 tcp_retries2 配置项，默认值是 15，如下图：
![](attachments/Pasted%20image%2020240327195557.png)
这个内核参数是控制，在 TCP 连接建立的情况下，超时重传的最大次数。

不过 tcp_retries2 设置了 15 次，并不代表 TCP 超时重传了 15 次才会通知应用程序终止该 TCP 连接，**内核会根据 tcp_retries2 设置的值，计算出一个 timeout**（_如果 tcp_retries2 =15，那么计算得到的 timeout = 924600 ms_），**如果重传间隔超过这个 timeout，则认为超过了阈值，就会停止重传，然后就会断开 TCP 连接**。

在发生超时重传的过程中，每一轮的超时时间（RTO）都是**倍数增长**的，比如如果第一轮 RTO 是 200 毫秒，那么第二轮 RTO 是 400 毫秒，第三轮 RTO 是 800 毫秒，以此类推。

而 RTO 是基于 RTT（一个包的往返时间） 来计算的，如果 RTT 较大，那么计算出来的 RTO 就越大，那么经过几轮重传后，很快就达到了上面的 timeout 值了。

举个例子，如果 tcp_retries2 =15，那么计算得到的 timeout = 924600 ms，如果重传总间隔时长达到了 timeout 就会停止重传，然后就会断开 TCP 连接：

- 如果 RTT 比较小，那么 RTO 初始值就约等于下限 200ms，也就是第一轮的超时时间是 200 毫秒，由于 timeout 总时长是 924600 ms，表现出来的现象刚好就是重传了 15 次，超过了 timeout 值，从而断开 TCP 连接
- 如果 RTT 比较大，假设 RTO 初始值计算得到的是 1000 ms，也就是第一轮的超时时间是 1 秒，那么根本不需要重传 15 次，重传总间隔就会超过 924600 ms。

最小 RTO 和最大 RTO 是在 Linux 内核中定义好了：
```c
#define TCP_RTO_MAX ((unsigned)(120*HZ))
#define TCP_RTO_MIN ((unsigned)(HZ/5))
```
Linux 2.6+ 使用 1000 毫秒的 HZ，因此`TCP_RTO_MIN`约为 200 毫秒，`TCP_RTO_MAX`约为 120 秒。

如果`tcp_retries`设置为`15`，且 RTT 比较小，那么 RTO 初始值就约等于下限 200ms，这意味着**它需要 924.6 秒**才能将断开的 TCP 连接通知给上层（即应用程序），每一轮的 RTO 增长关系如下表格：
![](attachments/Pasted%20image%2020240327195937.png)


### 拔掉网线后，没有数据传输
**针对拔掉网线后，没有数据传输的场景，还得看是否开启了 TCP keepalive 机制 （TCP 保活机制）。**

（1）如果**没有开启** TCP keepalive 机制
在客户端拔掉网线后，并且双方都没有进行数据传输，那么客户端和服务端的 TCP 连接将会一直保持存在。


（2）如果**开启**了 TCP keepalive 机制
在客户端拔掉网线后，即使双方都没有进行数据传输，在持续一段时间后，TCP 就会发送探测报文：
- 如果**对端是正常工作**的。当 TCP 保活的探测报文发送给对端, 对端会正常响应，这样 **TCP 保活时间会被重置**，等待下一个 TCP 保活时间的到来。
- 如果**对端主机宕机**（_注意不是进程崩溃，进程崩溃后操作系统在回收进程资源的时候，会发送 FIN 报文，而主机宕机则是无法感知的，所以需要 TCP 保活机制来探测对方是不是发生了主机宕机_），或对端由于其他原因（比如：拔掉网线）导致报文不可达。当 TCP 保活的探测报文发送给对端后，石沉大海，没有响应，连续几次，达到保活探测次数后，**TCP 会报告该 TCP 连接已经死亡**。


### 总结
![](attachments/Pasted%20image%2020240327202554.png)

# http的keep-alive
## 介绍
由于 HTTP 是基于 TCP 传输协议实现的，客户端与服务端要进行 HTTP 通信前，需要先建立 TCP 连接，然后客户端发送 HTTP 请求，服务端收到后就返回响应，至此「请求-应答」的模式就完成了，随后就会释放 TCP 连接。
![](attachments/Pasted%20image%2020240327202737.png)

如果每次请求都要经历这样的过程：建立 TCP -> 请求资源 -> 响应资源 -> 释放连接，那么此方式就是 **HTTP 短连接**；
这样实在太累人了，一次连接只能请求一次资源。
能不能在第一个 HTTP 请求完后，先不断开 TCP 连接，让后续的 HTTP 请求继续使用此连接？
当然可以，HTTP 的 Keep-Alive 就是实现了这个功能，可以使用同一个 TCP 连接来发送和接收多个 HTTP 请求/应答，避免了连接建立和释放的开销，这个方法称为 **HTTP 长连接**。
![](attachments/Pasted%20image%2020240327202813.png)

## http的keep-alive 和 tcp的keep-alive区别
- HTTP 的 Keep-Alive，是由**应用层（用户态）** 实现的，称为 HTTP 长连接；
- TCP 的 Keepalive，是由 **TCP 层（内核态）** 实现的，称为 TCP 保活机制；

## 使用 http的keep-alive
怎么才能使用 HTTP 的 Keep-Alive 功能？
### HTTP 1.0 中http的keep-alive
在 HTTP 1.0 中默认是关闭的，如果浏览器要开启 Keep-Alive，它必须在请求的包头中添加：

```
Connection: Keep-Alive
```

然后当服务器收到请求，作出回应的时候，它也添加一个头在响应中：

```
Connection: Keep-Alive
```

这样做，连接就不会中断，而是保持连接。当客户端发送另一个请求时，它会使用同一个连接。这一直继续到客户端或服务器端提出断开连接。

### HTTP 1.1 中http的keep-alive
**从 HTTP 1.1 开始， 就默认是开启了 Keep-Alive**，如果要关闭 Keep-Alive，需要在 HTTP 请求的包头里添加：
```
Connection:close
```

现在大多数浏览器都默认是使用 HTTP/1.1，所以 Keep-Alive 都是默认打开的。一旦客户端和服务端达成协议，那么长连接就建立好了。

## http的keep-alive的优点以及问题
HTTP 长连接不仅仅减少了 TCP 连接资源的开销，而且这给 **HTTP 流水线**技术提供了可实现的基础。

所谓的 HTTP 流水线，是**客户端可以先一次性发送多个请求，而在发送过程中不需先等待服务器的回应**，可以减少整体的响应时间。

举例来说，客户端需要请求两个资源。以前的做法是，在同一个 TCP 连接里面，先发送 A 请求，然后等待服务器做出回应，收到后再发出 B 请求。HTTP 流水线机制则允许客户端同时发出 A 请求和 B 请求。
![](attachments/Pasted%20image%2020240327203021.png)

但是**服务器还是按照顺序响应**，先回应 A 请求，完成后再回应 B 请求。
而且要等服务器响应完客户端第一批发送的请求后，客户端才能发出下一批的请求，也就说如果服务器响应的过程发生了阻塞，那么客户端就无法发出下一批的请求，此时就造成了「**队头阻塞**」的问题。

### keepalive_timeout
如果使用了 HTTP 长连接，如果客户端完成一个 HTTP 请求后，就不再发起新的请求，此时这个 TCP 连接一直占用着不是挺浪费资源的吗？

对没错，所以为了避免资源浪费的情况，web 服务软件一般都会提供 `keepalive_timeout` 参数，用来指定 HTTP 长连接的超时时间。

比如设置了 HTTP 长连接的超时时间是 60 秒，web 服务软件就会**启动一个定时器**，如果客户端在完后一个 HTTP 请求后，在 60 秒内都没有再发起新的请求，**定时器的时间一到，就会触发回调函数来释放该连接。**
![](attachments/Pasted%20image%2020240327203149.png)


# 参考
```c
小林：# TCP Keepalive 和 HTTP Keep-Alive 是一个东西吗？
https://xiaolincoding.com/network/3_tcp/tcp_http_keepalive.html


# 拔掉网线后， 原本的 TCP 连接还存在吗？
https://www.xiaolincoding.com/network/3_tcp/tcp_unplug_the_network_cable.html#%E6%8B%94%E6%8E%89%E7%BD%91%E7%BA%BF%E5%90%8E-%E6%9C%89%E6%95%B0%E6%8D%AE%E4%BC%A0%E8%BE%93
```