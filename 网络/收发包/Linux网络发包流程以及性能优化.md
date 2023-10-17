# 发包流程
1.应用程序的数据包，在TCP层增加TCP报文头，形成可传输的数据包。 
2.在IP层增加IP报头，形成IP报文。 
3.经过数据网卡驱动程序将IP包再添加14字节的MAC头，构成frame（暂⽆CRC），frame（暂⽆CRC）中含有发送端和接收端的MAC地址。 
4.驱动程序将frame（暂⽆CRC）拷贝到网卡的缓冲区，由网卡处理。 
5.⽹卡为frame（暂⽆CRC）添加头部同步信息和CRC校验，将其封装为可以发送的packet，然后再发送到网线上，这样说就完成了一个IP报文的发送了，所有连接到这个网线上的网卡都可以看到该packet。



# 统计计数
# 性能调优参数
# 参考
```c
# [转][译]Linux 网络栈监控和调优：发送数据（2017）
https://colobu.com/2019/12/09/monitoring-tuning-linux-networking-stack-sending-data/

# 说明的很清楚
https://simonzgx.github.io/2020/08/17/Linux%E7%BD%91%E7%BB%9C%E6%95%B0%E6%8D%AE%E5%8C%85%E6%8E%A5%E5%8F%97%E8%BF%87%E7%A8%8B/

发包流程：
https://www.ithome.com/0/644/289.htm

发包优化：
https://mp.weixin.qq.com/s/JR-qqjNG9ClHCYoRiFg-CQ


```