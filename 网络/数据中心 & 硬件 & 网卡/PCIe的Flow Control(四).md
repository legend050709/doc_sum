```table-of-contents
```
# 概述
## 可靠传输
可靠传输，可以分为两个方面

- 确保PCIe Link两端的数据传输不会因为SerDes的误码率而出错，主要利用CRC校验和重传（Replay）的机制。
- 在数据传送正确的基础上，确保Link两端在发送TLP时候，确保对方有能力接收数据，防止对方因为没有空间来接收数据导致数据的丢失。这就是流量控制（Flow Control）

从这个分层结构可以看出，Flow Control的目的是为了确保对方有能力接收TLP。TLP在Transaction Layer, TLP的发送首先达到Data Link Layer，**所以Flow Control的机制存在于Data Link Layer, 是直接用来管控TLP流向Data Link Layer的“红绿灯”**

# Flow Control
## 背景
对于大部分的串行传输协议而言，发送方能够有效地将数据发送至接收方的前提是，接收方有足够的接收Buffer来接收数据。在PCI总线中，发送方在发送前并不知道接收法是否有足够的Buffer来接收数据（即接收方是否就绪），因此经常需要一些Disconnects和Retries的操作，这将会严重地影响到总线的传输效率（性能）。

PCIe总线为了解决这一问题，提出了Flow Control的概念，如下图所示。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731220244.png)
>注：采用Flow Control机制的PCIe总线，相对于PCI总线获得了更高的总线利用率。虽然增加了Flow Control DLLP，但是这些DLLP对带宽的占用极小，几乎对总线利用率没有什么影响。

PCIe Spec规定，PCIe设备的每一个端口（Ports）都必须支持Flow Control机制，在发送TLP之前，Flow Control必须先检查接收端口是否有足够的Buffer空间来接收这个TLP。当PCIe设备支持多个VC（Virtual Channel）时，Flow Control机制可以显著地提高总线的传输效率。

PCIe Spec规定，每个PCIe端口最多支持8个VC，并且每个VC的Flow Control Buffer是完全独立的。也就是说，某一个VC的Flow Control Buffer满了，并不会影响其他的VC的通信。

## 目的
PCIe总线采用Flow Control的目的是，保证发送端的PCIe设备永远不会发送接收端的PCIe设备不能接收的TLP（事务层包）。也就是说，发送端在发送前可以通过Flow Control机制知道接收端能否接收即将发送的TLP。

# 参考
```c
https://zhuanlan.zhihu.com/p/352533416

http://blog.chinaaet.com/justlxy/p/5100053464
```
