# 简介
![](attachments/Pasted%20image%2020230906201859.png)
![](attachments/Pasted%20image%2020230906201925.png)
# 建连快
数据的发送和接收，要想保证安全和可靠，一定是需要连接的。TCP 需要，QUIC 也同样需要。连接到底是什么？连接是一个通道，是在一个客户端和一个服务端之间的唯一一条可信的通道，主要是为了安全考虑，建立了连接，也就是建立了可信通道，服务器对这个客户端“很放心”，对于服务器来说：你想跟我进行通信，得先让我认识一下你，我得先确认一下你是好人，是有资格跟我通信的。那么这个确认对方身份的过程，就是建立连接的过程。

## TCP+TLS建连慢
![](attachments/Pasted%20image%2020230906141217.png)
传统基于 TCP 的 HTTPS 的建连过程为什么如此慢？它需要 TCP 和 TLS 两个建连过程。如图 1 所示（传统 HTTPS 请求流程图）：
![](attachments/Pasted%20image%2020230906115033.png)

>对于一个小请求（用户数据量较小）而言，传输数据只需要 1 个 RTT，但是光建连就花掉了 3 个 RTT，这是非常不划算的，这里建连包括两个过程：TCP 建连需要 1 个 RTT，TLS 建连需要 2 个 RTT。RTT：Round Trip Time，数据包在网络上一个来回的时间。


- **TCP 和 TLS 没办法合并**
**为什么需要两个过程？**可恶就可恶在这个地方，TCP 和 TLS 没办法合并，因为 TCP 是在内核里完成的，TLS 是在用户态。
也许有人会说把干掉内核里的 TCP，把 TCP 挪出来放到用户态，然后就可以和 TLS 一起处理了。首先，你干不掉内核里的 TCP，TCP 太古老了，全世界的服务器的 TCP 都固化在内核里了。

- **思路**
所以，既然干不掉 TCP，那我不用它了，我再自创一个传输层协议，放到用户态，然后再结合 TLS，这样不就可以把两个建连过程合二为一了吗？是的，这就是 QUIC。

## TCP的拥塞控制之慢启动
TCP 由于具有「拥塞控制」的特性，所以刚建立连接的 TCP 会有个「慢启动」的过程，它会对 TCP 连接产生“减速”效果。



## QUIC 的 1-RTT 建连
如图 2 所示，是 QUIC 的连接建立过程：初次建连只需要 1 个 RTT 即可完成建连。后续再次建连就可以使用 0-RTT 特性。
![](attachments/Pasted%20image%2020230906115540.png)

- **QUIC 的 1-RTT 建连：**
客户端与服务端初次建连（之前从未进行通信过），或者长时间没有通信过（0-RTT 过期了），只能进行 1-RTT 建连。只有先进行一次完整的 1-RTT 建连，后续一段时间内的通信才可以进行 0-RTT 建连。

如图 3 所示：QUIC 的 1-RTT 建连可以分成两个部分。QUIC 连接信息部分和 TLS1.3 握手部分。
![](attachments/Pasted%20image%2020230906202057.png)

**QUIC 连接**：协商 QUIC 版本号、协商 quic 传输参数、生成连接 ID、确定 Packet Number 等信息，类似于 TCP 的 SYN 报文；保证通信的两端确认过彼此，是对的人。

**TLS1.3 握手**：标准协议，非对称加密，目的是为了协商出 对称密钥，然后后续传输的数据使用这个对称密钥进行加密和解密，保护数据不被窃取。

### TLS1.3握手
重点看 QUIC 的 TLS1.3 握手过程。
![](attachments/Pasted%20image%2020230906202331.png)
通过图 4 可以看到，整个握手过程需要 2 次握手（第三次握手是带了数据的），所以整个握手过程只需要 1-RTT（RTT 是指数据包在网络上的一个来回）的时间。

1-RTT 的握手主要包含两个过程：
1. 客户端发送 Client Hello 给服务端；
2. 服务端回复 Server Hello 给客户端；

我们通过下图中图 5 和图 6 来看 Client Hello 和 Server Hello 具体都做了啥：

（1）**第一次握手（Client Hello 报文）**
![](attachments/Pasted%20image%2020230906202504.png)
Client Hello 在扩展字段里标明了支持的 TLS 版本（Supported Version：TLS1.3）。值得注意的是 Version 字段必须要是 TLS1.2，这是因为 TLS1.2 已经在互联网上存在了 10 年。网络中大量的网络中间设备都十分老旧，这些网络设备会识别中间的 TLS 握手头部，所以 TLS1.3 的出现如果引入了未知的 TLS Version 必然会存在大量的握手失败。

![](attachments/Pasted%20image%2020230906202544.png)
其次，ClientHello 中包含了非常重要的 key_share 扩展：客户端在发送之前，会自己根据 DHE 算法生成一个公私钥对。发送 Client Hello 报文的时候会把这个公钥发过去，那么这个公钥就存在于 key_share 中，key_share 还包含了客户端所选择的曲线 X25519。总之，key_share 是客户端提前生成好的公钥信息。
最后，Client Hello 里还包括了：客户端支持的算法套、客户端所支持的椭圆曲线以及签名算法、psk 的模式等等，一起发给服务端。
![](attachments/Pasted%20image%2020230906202634.png)

（2）**第二次握手：（Server Hello 报文）**
![](attachments/Pasted%20image%2020230906202746.png)
服务端自己根据 DHE 算法也生成了一个公私钥对，同样的，Key_share 扩展信息中也包含了 服务端的公钥信息。服务端通过 ServerHello 报文将这些信息发送给客户端。

至此为止，双方（客户端服务端）都拿到了对方的公钥信息，然后结合自己的私钥信息，生成 pre-master key，在这里官方的叫法是（client_handshake_traffic_secret 和server_handshake_traffic_secret），然后根据以下算法进行算出 key 和 iv，使用 key 和 iv 对 Server Hello 之后所有的握手消息进行加密。

>_注意：在握手完成之后，服务端会发送一个 New Session Ticket 报文给客户端，这个包非常重要，这是 0-RTT 实现的基础。_

![](attachments/Pasted%20image%2020230906202844.png)


## QUIC 的 0-RTT 握手
这个功能类似于 TLS1.2 的会话复用，或者说 0-RTT 是基于会话复用功能的。
![](attachments/Pasted%20image%2020230906203115.png)

通过上图  我们可以看到，client 和 server 在建连时，仍然需要两次握手，仍然需要 1 个 rtt，但是为什么我们说这是 0-rtt 呢，是因为 client 在发送第一个包 client hello 时，就带上了数据（HTTP 请求），从什么时候开始发送数据这个角度上来看，的确是 0-RTT。
我们通过抓包来看 0-RTT 的过程：
![](attachments/Pasted%20image%2020230906203202.png)

所以真正在实现 0-RTT 的时候，请求的数据并不会跟 Initial 报文（内含 Client Hello）一起发送，而是单独一个数据包（0-RTT 包）进行发送。即一共发了2个包，数据包只不过是跟 Initial 包同时发出来而已。
![](attachments/Pasted%20image%2020230906203422.png)
我们单独看 Initial 报文发现，除了 pre_share_key、early-data 标识等信息与 1-RTT 时不同，其他并无区别。

## QUIC 建连需要注意的问题
第一，QUIC 实现的时候，必须缓存收到的乱序加密帧，这个缓存至少要大于 4096 字节。当然可以选择缓存更多的数据，更大的缓存上限意味着可以交换更大的密钥或证书。终端的缓存区大小不必在整个连接生命周期内保持不变。这里记住：乱序帧一定要缓存下来。如果不缓存，会导致连接失败。如果终端的缓存区不够用了，则其可以通过暂时扩大缓存空间确保握手完成。如果终端不扩大其缓存，则其必须以错误码 CRYPTO_BUFFER_EXCEEDED 关闭连接。

第二，0-RTT 存在前向安全问题，请慎用！

# 不存在队头阻塞问题
QUIC依赖于UDP，UDP不存在队头阻塞问题。

## HTTP2依赖的TCP的队头阻塞
在HTTP/2中，多个请求是跑在一个TCP管道中的。但当出现了丢包时，HTTP/2 的表现反倒不如 HTTP/1 了。因为TCP为了保证可靠传输，有个特别的“丢包重传”机制，丢失的包必须要等待重新传输确认，HTTP/2出现丢包时，整个 TCP 都要开始等待重传，那么就会阻塞该TCP连接中的所有请求（如下图）。而对于 HTTP/1.1 来说，可以开启多个 TCP 连接，出现这种情况反到只会影响其中一个连接，剩余的 TCP 连接还可以正常传输数据。
# 参考
```c
https://mp.weixin.qq.com/s/3GwoY7wGPybcPsR9--dLyQ
```