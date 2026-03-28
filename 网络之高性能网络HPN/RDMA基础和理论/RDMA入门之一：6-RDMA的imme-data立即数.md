```table-of-contents
``` 

# 介绍
```bash
optional) A 32 bits number, in network order, in an SEND or RDMA WRITE opcodes that is being sent along with the payload to the remote side and placed in a Receive Work Completion and not in a remote memory buffer
```

# 零字节的消息
参考：[# Zero byte messages](https://www.rdmamojo.com/2013/09/20/zero-byte-messages/)

# 其他
## RDMA的字节序问题

![](attachments/Pasted%20image%2020250714152831.png)

如上所示:
(1) RDMA报文中的数据都是网络字节序发送的。
(2) 控制路径中调用`libibverbs API`不需要考虑字节序。
(3) RDMA数据路径发送数据时，如果是字节流发送数据（即：不需要关注一连串数据中的某个具体的数值），那么就应该没有字节序的问题，都是网络字节序发和收。
如果是在发送的数据中取一个具体的数值(比如：imm_data) ，那么在发的时候就需要将其转换为网络字节序了，收到之后使用之前需要明确自己收到的是网络字节序。

# 参考
```bash
# RDMA与外卖小哥
https://mp.weixin.qq.com/s/GvLeHAVXerQBQlLrJfcbFQ


```