```table-of-contents
```
# tcp socket
# udp socket
## udp socket的发送缓冲区
### udp socket是否有发送缓冲区
#### 误解
以前我以为：

> "每个UDP socket都有一个接收缓冲区，没有发送缓冲区，从概念上来说就是只要有数据就发，不管对方是否可以正确接收，所以不缓冲，不需要发送缓冲区。"

#### 正解
UDP socket 也是 socket，一个socket 就是会有收和发两个缓冲区，跟用什么协议关系不大。有没有是一回事，用不用又是一回事。

### UDP socket不用发送缓冲区？
事实上，UDP不仅有发送缓冲区，也用发送缓冲区。
**一般正常情况下，会把数据直接拷到发送缓冲区后直接发送**。

还有一种情况，是在发送数据的时候，设置一个 `MSG_MORE` 的标记。
```bash
ssize_t send(int sock, const void *buf, size_t len, int flags); 
// flag 置为 MSG_MORE
```

大概的意思是告诉内核，待会还有其他**更多消息**要一起发，先别着急发出去。此时内核就会把这份数据先用**发送缓冲区**缓存起来，待会应用层说ok了，再一起发。

我们可以看下源码。
```bash
int udp_sendmsg()  
{  
    // corkreq 为 true 表示是 MSG_MORE 的方式，仅仅组织报文，不发送；  
    int corkreq = up->corkflag || msg->msg_flags&MSG_MORE；  
  
    //  将要发送的数据，按照MTU大小分割，每个片段一个skb；并且这些  
    //  skb会放入到套接字的发送缓冲区中；该函数只是组织数据包，并不执行发送动作。  
    err = ip_append_data(sk, fl4, getfrag, msg->msg_iov, ulen,  
                 sizeof(struct udphdr), &ipc, &rt,  
                 corkreq ? msg->msg_flags|MSG_MORE : msg->msg_flags);  
  
    // 没有启用 MSG_MORE 特性，那么直接将发送队列中的数据发送给IP。   
    if (!corkreq)  
        err = udp_push_pending_frames(sk);  
  
}
```

因此，不管是不是 `MSG_MORE`， 都会先把数据放到发送队列中，然后根据实际情况再考虑是不是立刻发送。

而我们大部分情况下，都不会用  `MSG_MORE`，也就是来一个数据包就直接发一个数据包。从这个行为上来说，**虽然UDP用上了发送缓冲区，但实际上并没有起到"缓冲"的作用。**

### 小结
`UDP socket` 也存在发送缓冲区。
我们大部分情况下，都不会用  `MSG_MORE`，也就是来一个数据包就直接发一个数据包。从这个行为上来说，**虽然UDP用上了发送缓冲区，但实际上并没有起到"缓冲"的作用。**

# 参考
```c
# 动画图解 socket 缓冲区的那些事儿
https://mp.weixin.qq.com/s/yImrTDVCsVsbZicj-ncn4Q
```