```table-of-contents
```
# 网卡的收发包
![](attachments/Pasted%20image%2020230809121546.png)

# MRRS
```c
PCIe Max Read Request determines the maximal PCIe read request allowed. A PCIe device usually keeps track of the number of pending read requests due to having to prepare buffers for an incoming response. The size of the PCIe max read request may affect the number of pending requests (when using data fetch larger than the PCIe MTU). Again, use the command **lspci** in order to query for the Max Read Request value:
```
为什么有MRRS。MRRS决定了最大的读取数据量，因为对于每个请求的响应，接收者需要准备缓冲区。
# 其他
## MPS & MRRS &RCB的区别
MPS: posted 类型的内存写
MRRS：内存读，发起多少个请求
RCB：内存读，发起多少个complete.
# 参考
```c
公众号、知乎
https://mp.weixin.qq.com/s/l6tgZyn7CXvnqGsaEjlrtw
https://mp.weixin.qq.com/s/KBkkI42pb8jyMlZjKTw2Ew
http://blog.chinaaet.com/justlxy/p/5100062236
https://zhuanlan.zhihu.com/p/645335755

PCIe的开销以及效率：
https://mp.weixin.qq.com/s/fFS5tkeJxKD0Z7If7BXRDQ
【计算传输效率的例子比较好】
https://chipdebug.com/forum-post/40241.html

MPS对于性能的影响：
https://mp.weixin.qq.com/s/NTE062fX67ZXMEzeujLdrQ
https://aijishu.com/a/1060000000287490#item-2-3

英文文档
http://trac.gateworks.com/wiki/PCI
https://docs.xilinx.com/v/u/en-US/wp350

RCB的原理：
https://www.cnblogs.com/readdad/p/pcie-rcb-design-original-intention-zjxrua.html


```