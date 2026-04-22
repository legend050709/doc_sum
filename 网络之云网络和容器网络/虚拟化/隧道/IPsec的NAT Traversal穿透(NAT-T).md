```table-of-contents
```
# IPSec NAT穿越的场景

设备作为NAT网关部署在建立IPsec隧道的两个设备间的场景，又称为IPsec NAT穿越场景，如下所示。

**分支作为协商发起方位于一个私网内部**，协商报文通过NAT网关及Internet到达对端响应方，协商并建立IPsec隧道。为保证协商报文能够顺利通过NAT网关，需要在IPsec隧道两端的设备（DeviceA和DeviceC）上开启NAT穿越功能。

![](attachments/Pasted%20image%2020241126113626.png)

# IPsec 和 NAT的不兼容

以一个TCP报文为例来看看在不同IPsec的不同模式(Transport和Tunnel)和协议(AH和ESP)下，这种不兼容是如何发生的。

## Transport模式

![](attachments/Pasted%20image%2020241126115127.png)

对AH协议，由于其Authenticate范围是整个IP报文，所以如果两个IPsec之间存在NAT设备，修改了报文IP Header中的地址，就会导致接收方的Authenticate失败。

对ESP协议，其Authenticate返回不包括IP Header，所以接收方的Authenticate会通过，但如果中间的NAT设备修改了IP Header中的地址，理论上后面的TCP/UDP checksum也会随之修改，但这部分在ESP协议中是 加密的，NAT设备没有办法修改，所以接收端在TCP/UDP 接收时会出现checksum校验失败。

### 小结 

因此，==对于传输（Transport）模式，无论是AH还是ESP的传输模式，都是和 NAT不兼容的==。

## Tunnel模式

![](attachments/Pasted%20image%2020241126115224.png)

对AH协议, AH认证整个IP头部，包括新生成的IP头；Tunnel模式和Transport模式没什么不同，Authenticate范围包含了外层IP Header，因此同样会造成接收方Authenticate失败。

对ESP协议，ESP的认证范围是从ESP头部到ESP尾部。与Transport模式不同的是，经过NAT设备。内层IP Header并不会改变，所以TCP checksum也不会变化，接收方不会出现checksum校验失败。

这样看起来，ESP-Tunnel似乎成为了在有NAT设备环境下，唯一可行的协议-模式组合。但即使是这种组合也是有缺点的。
> 它只能支持一对一的NAT(NAT设备后面只有一台内网主机)。在很多组网中，NAT设备通常作为网关使用，其背后可能有很多台主机。这时地址转换就不够了，它还需要端口转换，显然，NAT设备对ESP-Tunnel的报文是无能为力的，因为TCP部分已经被加密了，已经没有端口字段了。 所以，IPsec需要想办法能绕开NAT设备的影响，也就是进行NAT穿越(NAT-Tranversal)。

### 小结

AH的隧道模式，不兼容NAT；
==ESP的隧道模式，兼容一对一的ip-address 的 NAT，不支持ip + port的NAT==。

## AH 和 NAT 的不兼容

虽然AH提供了非常强大的数据包内容保护，因为它覆盖了一切可能被视为不可变的内容，但是这种保护是有代价的：AH与NAT（网络地址转换，Network Address Translation）不兼容。

![](attachments/Pasted%20image%2020241125211212.png)

### nat的更改字段

NAT被用于将一个范围的私有地址（例如，192.168.1.X）映射到（通常是）较小的一组公共地址，并因此减少了对可路由的公共 IP 空间的需求。
在这个过程中，NAT设备会在传输过程中实时修改 IP 头，改变源 IP 地址和/或目标 IP 地址。

### ICV 包含 ip地址，不包含TTL和 IP头的csum

当更改源或目标IP地址时，会强制重新计算标头校验和。这个过程必须进行，因为NAT设备通常在源和目标之间作为路径中的一个"跃点（hop）"，这需要减少TTL（存活时间:Time To Live）字段。

由于TTL和头部校验和字段总是在传输过程中被修改，因此 AH 知道将它们排除在覆盖范围之外，但这不适用于 IP 地址。

IP 地址将包含在完整性校验值（ICV）中，任何修改都将导致接收方在验证时失败。由于ICV包含了一个对中间方不知道的私有密钥，NAT路由器无法重新计算ICV。

同样的困难也适用于PAT（端口地址转换：Port Address Translation），它将多个私有IP地址映射到一个外部IP地址。不仅要实时修改IP地址，还要修改UDP和TCP端口号（甚至可能包括有效负载）。这需要 NAT 设备具有更高的智能性，并对整个 IP 数据报进行更广泛的修改。

### 小结
AH协议，无论是隧道模式还是传输模式下，AH 都与 NAT 完全不兼容。只有当源和目标网络可以无需转换即可到达时，才能使用 AH。

## 总结

AH协议而言，无论是传输模式，还是隧道模式，都不兼容NAT；
==ESP协议，传输模式不兼容NAT；隧道模式支持一对一的NAT，不支持Port的NAT==。


# IPsec 实现 NAT-T

## 背景

如上，已知：
```text
AH协议而言，无论是传输模式，还是隧道模式，都不兼容NAT；
ESP协议，传输模式不兼容NAT；隧道模式支持一对一的NAT，不支持Port的NAT。
```
如果NAT设备期望进行Port 进行转换。那么如何处理呢？


### AH协议的问题

AH协议对IP报文的整性检查范围涵盖了整个IP报文，对IP报文头的任何修改将导致AH的完整性校验失败，而地址转换会改变IP地址，因此使用AH协议保护的IPsec隧道不能穿越NAT设备。

### ESP协议的问题

ESP协议对数据进行的完整性检查不包括外部的IP头，若只进行了地址转换，则使用ESP协议保护的IPsec隧道可以穿越NAT设备。但是由于ESP协议是三层协议，无法设置端口，因此当进行允许端口转换的NAT时采用ESP协议保护的IPsec隧道也存在问题。

## 方案：NAT网关为ESP协议报文的IP头后增加UDP头的方式

![](attachments/Pasted%20image%2020241126115909.png)

NAT穿越采用了：NAT网关**为ESP协议报文的IP头后增加UDP头的方式**
当ESP隧道报文穿越NAT网关时，NAT对该报文的外层IP和增加的UDP报文头进行地址和端口的转换。
转换后到达对端，与普通IPsec处理方式相同，但在发送响应报文时也要在IP头和ESP头之间增加一个UPD报文头。

**（1）ESP传输模式下**
NAT穿越在原报文的IP头和ESP头间增加一个标准的UDP报头；

**（2）ESP隧道模式下**
NAT穿越在新IP头和ESP头间增加一个标准的UDP报头。

这样，当ESP报文穿越NAT设备时，NAT设备对该报文的外层IP头和增加的UDP报头进行地址和端口号转换；转换后的报文到达IPsec隧道对端后，与普通IPsec处理方式相同。


## IKE与NAT穿越

### IPSEC网关如何知道自己是否支持NAT-T

决定双方是否支持NAT-T和判断peers之间是否有NAT存在的任务在IKEv1的第一阶段完成，NAT-T能力检测在IKE协商的前两个消息中交换完成，通过在消息中插入一个标识NAT-T能力的Vendor ID载荷来告诉对方自己对该能力的支持。

如果双方都在各自的消息中包含了该载荷，说明双方对NAT-T都是支持的。只有双方同时支持NAT-T能力，才能继续进行其他协商。

![](attachments/Pasted%20image%2020241126122940.png)

![](attachments/Pasted%20image%2020241126123008.png)

### IPSEC网关如何判定经过NAT的设备

当存在NAT设备时必须使用UDP传输，所以在IKEv2中的第一阶段协商中必须先探测是否存在NAT设备，也就是NAT探测。通过发送NAT-D载荷来实现NAT探测是目前比较流行的方法。

#### 判定依据

探测的方式就计算本端IP地址和端口的HASH1值和对端IP地址和端口的HASH2值，然后将这两个哈希值同时以NAT-D负载的方式发送给对方设备，对方收到后通过计算接收到的报文中源和目的的IP地址和端口的哈希值与NAT-D负载中的哈希值进行比较：

![](attachments/Pasted%20image%2020241126140521.png)

![](attachments/Pasted%20image%2020241126140615.png)


判断依据：

 （1）计算对端的IP地址和端口的哈希值hash1, 然后与报文中NAT-D中对端计算的哈希值HASH1进行比较，如果不同，说明对端的IP地址或者端口发生了变化，因此对端设备是位于NAT设备之后的。
 （2）计算本端的IP地址和端口的哈希值hash2, 然后和报文中NAT-D负载中对端的计算结果HASH2进行比较，如果不同，则说明本端的IP地址或者端口发生了变化，因此本端设备是位于NAT设备之后的。
 （3）由于响应端在第三个报文时便可以知道链路上的NAT情况，但是发送端还不清楚，因此响应端需要将本端和对端的哈希值计算后填充到第四个报文的NAT-D负载中发送给发起端。

#### NAT-T的端口浮动

![](attachments/Pasted%20image%2020241126140844.png)

##### IKE端口浮动的原因
端口浮动的原因在于有些NAT设备对于500端口的报文不做NAT转换，从而导致NAT穿越失败。至于都包括哪些设备，暂不清楚。因此将端口浮动到4500后，方便NAT设备进行映射转换，从而实现NAT-T穿越。

##### KE端口浮动是必须的吗？

首先说明，端口浮动不是必须的，但是现在通常情况下是进行端口浮动的：
即如果有NAT-T存在，则IKE端口会从500切换到4500。

##### IKE端口浮动一定是浮动到4500吗

IKE端口浮动肯定是将端口由500浮动到4500的（包括源端口和目的端口），但是中间的NAT设备如果支持端口映射的话，那么一般是将源端口做一个映射, 源端口做映射对于IPSEC影响并不大，但是要求IPsec能够响应来自任意端口的报文（下图中的X便是经过NAPT映射后的IPsec报文）。NAT设备做端口映射的目的主要为了为了实现多路分解和复用。

![](attachments/Pasted%20image%2020241126141137.png)

### NAT-T的启用

在第一阶段协商完成之后，协商双方均已经明确是否存在NAT，以及NAT的位置。

至于是否启用NAT穿越，则由快速模式协商决定。
NAT穿越的启用协商在快速模式的SA载荷中进行。

传输模式下，协商双方可向对端发送IPSec报文的原始地址，从而使对端有可能在NAT转换之后，对TCP/IP进行校验和修正。


### NAT-T 的 keepalive 机制

由于NAT设备上的NAT会话表项有一定的存活时间，如果IPsec隧道建立后长时间没有报文进行NAT穿越，则NAT设备会删除该NAT会话表项，这将导致在NAT设备外网侧的对等体无法继续传输数据。

为防止NAT表项老化，NAT设备内网侧的IKE SA会以一定的时间间隔向对端发送NAT Keepalive报文，以维持NAT会话的存活。


# ipsec vpn和ssl vpn的nat兼容性

SSL/TLS协议对NAT完全透明，因为它位于TCP协议之上，与NAT没有半毛钱关系。因此SSL与NAT兼容性非常友好。

## ipsec 和 SSL 对比

![](attachments/Pasted%20image%2020241202105031.png)

# 参考

```c
# IPsec与NAT Traversal(NAT-T)
https://switch-router.gitee.io/blog/IPsec-nat-t/

# NAT-T下的端口浮动
https://blog.csdn.net/s2603898260/article/details/105214411

# IPSec的 NAT-T 系列
https://blog.csdn.net/s2603898260/category_9780533.html

## IPSec VPN的NAT穿越(NAT-T)原理
https://cshihong.github.io/2019/04/17/IPSec%20VPN%E7%9A%84NAT%E7%A9%BF%E8%B6%8A-NAT-T-%E5%8E%9F%E7%90%86/
```