```table-of-contents
```
# 背景
NAT设备一般根据报文4元组(IP, 端口)进行转换，但是在IPSec的报文中AH\ESP头部中根本没有端口的概念，因此无法穿越NAT设备。

后来IPSec专门修改了在NAT下的报文封装格式：在IP头与AH/ESP头部之间增加一个UDP头部，专门用来做NAT转换；NAT下不再使用IP地址作为标识，改用其他参数作为标识。
通过这样的封装可以通过NAT设备了，不过在IPSec协议栈中，增加了很多有关NAT的处理逻辑，导致IPSec的实现更加复杂。
![](attachments/Pasted%20image%2020231212192932.png)
> SSL/TLS协议对NAT完全透明，因为它位于TCP协议之上，与NAT没有半毛钱关系。因此SSL与NAT兼容性非常友好。

# IPsec 和 NAT的不兼容
以一个TCP报文为例来看看在不同IPsec的不同模式(Transport和Tunnel)和协议(AH和ESP)下，这种不兼容是如何发生的。

- **Transport模式**：
![](attachments/Pasted%20image%2020231212193436.png)

对AH协议，由于其Authenticate范围是整个IP报文，所以如果两个IPsec之间存在NAT设备，修改了报文IP Header中的地址，就会导致接收方的Authenticate失败。

对ESP协议，其Authenticate返回不包括IP Header,所以接收方的Authenticate会通过，但如果中间的NAT设备修改了IP Header中的地址，理论上后面的TCP checksum也会随之修改，但这部分在ESP协议中是 加密的，NAT设备没有办法修改，所以接收端在TCP接收时会出现checksum校验失败。

- **Tunnel模式**：
![](attachments/Pasted%20image%2020231212193551.png)
对AH协议, Tunnel模式和Transport模式没什么不同，Authenticate范围包含了外层IP Header，因此同样会造成接收方Authenticate失败。
对ESP协议，与Transport模式不同的是，经过NAT设备。内层IP Header并不会改变，所以TCP checksum也不会变化，接收方不会出现checksum校验失败。

这样看起来，ESP-Tunnel似乎成为了在有NAT设备环境下，唯一可行的协议-模式组合。但即使是这种组合也是有缺点的。
> 它只能支持一对一的NAT(NAT设备后面只有一台内网主机)。在很多组网中，NAT设备通常作为网关使用，其背后可能有很多台主机。这时地址转换就不够了，它还需要端口转换，显然，NAT设备对ESP-Tunnel的报文是无能为力的，因为TCP部分已经被加密了，已经没有端口字段了。 所以，IPsec需要想办法能绕开NAT设备的影响，也就是进行NAT穿越(NAT-Tranversal)。


# 参考
```c
# IPsec与NAT Traversal(NAT-T)
https://switch-router.gitee.io/blog/IPsec-nat-t/

# NAT-T下的端口浮动
https://blog.csdn.net/s2603898260/article/details/105214411

# IPSec的 NAT-T 系列
https://blog.csdn.net/s2603898260/category_9780533.html
```