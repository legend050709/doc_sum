```table-of-contents
```
# 背景
昨天瞎逛，看到一个开源项目：[udp2raw-tunnel](https://link.segmentfault.com/?enc=XvG784R21mvZQUgzPbG8Eg%3D%3D.THsqIzzY%2FNPu9ZudQzlbNjOs8r3XvztW5sWb2lxXaX14Z0r1RIUTsjrM7A%2F4lWUh)，他实现的是将一个IP报文伪装成TCP报文，目的是穿过网络中UDP防火墙.

哈？！这难道是`TCP-In-TCP`？ 这玩意儿不是不可用吗？
![](attachments/Pasted%20image%2020231128174311.png)

# 结论
很早以前就有人说过了：`TCP-In-TCP`不是一个好主意..
[Why TCP Over TCP Is A Bad Idea](https://rockhoppervpn.sourceforge.net/techdoc_tcpotcp.html)
![](attachments/Pasted%20image%2020231128174624.png)

# 分析
![](attachments/Pasted%20image%2020231128174643.png)
图中主机A和主机D通过隧道进行End to End的通信，而设备B对原始报文进行加密和封装，设备C做解封装和解密。当然B和C不一定是单独的设备，它们可以就集成在主机A和主机D中。

`TCP-In-TCP`的问题是: 当隧道报文在网络中丢失时，B上的TCP连接`<B-C>`显然会对报文进行重传，但要知道A上也有一个TCP连接`<A-D>`，所以A如果超时也会进行重传，而从整个网络的角度，这个重传是不必要的。但A不会理会，因为它是TCP。

TCP的设计原则是**下层协议和介质是不可靠的，所以我需要自己保证**。所以, `TCP-In-TCP`这样的双保险完全是没有必要的。不仅没必要，还有可能造成网络中重传报文过多而拥塞。

# 其他
本文开头提到的那个开源项目是怎么回事呢？ 仔细看了看它的介绍，又下载了它的代码。噢，原来它真的只是**伪装**成了TCP报文，实际上隧道报文的外层TCP封装都是在用户态自己填充的TCP首部，然后通过`raw socket`发送；而在接收端，同样使用`raw socket`(关于这个，详见[inet socket 与 packet socket](https://segmentfault.com/a/1190000020103410))，所以报文会提前进入`raw_local_deliver`上送，而不会由TCP去接收，这样也就没有了TCP状态机那一大堆复杂的东西。

# 参考
```c
https://segmentfault.com/a/1190000020458221
```