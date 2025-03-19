```table-of-contents
```


# QP
## Work queue
## SQ
## RQ

# QP连接
## 建立QP连接
### 作用

## 

# CQ
## QP和CQ的关系

![](attachments/Pasted%20image%2020250318144055.png)

一个QP包含一个Send Queue(SQ)，一个Receive Queue(RQ)以及对应的Send Completion Queue(SCQ)和Receive Completion Queue(RCQ)。
用户发送请求的时候，把请求封装为一个Work Queue Element(WQE)发送到SQ里面，然后RDMA网卡会把这个WQE发送出去，当这个WQE完成的时候，对应的SCQ里面会被放一个Completion Queue Element(CQE)，然后用户可以从SCQ里面Poll这个CQE并通过检查状态来确认对应的WQE是否成功完成。
需要指出的是，**不同的QP可以共用CQ来减少SRAM的存储消耗**。



# 参考
```bash
# RDMA cq event机制-ibv_req_notify_cq
https://zhuanlan.zhihu.com/p/688269158


```