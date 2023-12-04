```table-of-contents
```
# 使用
# 场景
# 问题
## Wireshark分析包中TCP包的 Len 远远大于 MSS
**问题**
正常每个TCP数据包的Len 不应该大于 MSS，但我们通过 tcpdump 抓包出来后，通过 wireshark 查看的话，有些 Len 远远大于了 MSS值。如下所示：
![](attachments/Pasted%20image%2020231130155051.png)

**分析**
这是由于服务器网卡开启了 TCP Segment Offload, TSO 选项导致，在支持 TSO 的网卡上，为了降低 CPU 的负载，提高网络的出口带宽，TSO 提供一些较大的缓冲区来缓存 TCP 发送的包，然后由网卡负责TCP的分段。而我们通过 tcpdump 抓包工具抓取的是从内核到网卡路径上的数据包，此时还没有到网卡，还没有进行TCP的分段。
![](attachments/Pasted%20image%2020231130155234.png)

**解决**
关闭网卡的TSO：
```c
ethtool -K eth04 tso off
```
上面设置后，我们再进行实验，抓包分析如下：
![](attachments/Pasted%20image%2020231130155343.png)

再次抓包后，发现Len没有大于MSS的了，这里为1488，之所以是1488bytes，是因为 Len = 1500(MTU) - 20(IP Header)- 20(TCP Header) -12(TCP 选项： timestamp 选项 + padding ) = 1448 bytes。
# 参考
```c
```