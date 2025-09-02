```table-of-contents
```

# 为什么进行IB数据包的分析
![](attachments/Pasted%20image%2020250721110214.png)


# ibdump
## ibdump包
```bash
# which ibdump
/bin/ibdump

# rpm -qf /bin/ibdump
ibdump-6.0.0-1.54310.x86_64

# rpm -ql ibdump-6.0.0-1.54310.x86_64
/usr/bin/ibdump
/usr/bin/vpi_tcpdump
```

![](attachments/Pasted%20image%2020250623152059.png)


## 特性

![](attachments/Pasted%20image%2020250721110055.png)

![](attachments/Pasted%20image%2020250721110115.png)

## 使用限制

![](attachments/Pasted%20image%2020250721110524.png)

# tcpdump-rdma


# 参考
```bash
# Analyzing InfiniBand Packets
https://openfabrics.org/images/eventpresos/workshops2015/UGWorkshop/Thursday/thursday_09.pdf

# RDMA测试杂谈
https://mp.weixin.qq.com/s/qDjOKj4ISMdZXlhN54vVkQ

# RoCEv2智能流量分析
https://support.huawei.com/enterprise/zh/doc/EDOC1100198802/5d4312fb


#【RDMA】LRH和GRH InfiniBand标头（LRH and GRH InfiniBand Headers）
https://blog.csdn.net/bandaoyu/article/details/117464053


# RDMA(4)协议栈：你追我赶，快快跑 【系列文章++++++】
https://mp.weixin.qq.com/s/0td4YuE8MBw5RtvOGOFylw  
```