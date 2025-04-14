```table-of-contents
```
# UD
## UD下send 和 recv
## UD和数据分段、IP分片
# RC
## RC下read 和 write
## RC下send 和 recv
# XRC
**XRC（eXtended Reliable Connection）** 是 RDMA 的一种传输服务类型，专为**多对多通信**设计。
## 背景
**传统 RC（Reliable Connection）的瓶颈**：
    - 每个通信对端需创建独立的 QP（Queue Pair），当节点数量增加时，QP 数量呈平方级增长（例如 1000 节点需 100 万 QP），严重消耗资源。

- **XRC 的改进**：
    - 通过共享接收资源（XRC SRQ）和复用发送端 QP，将 QP 数量从 O(N2)O(N2) 降低到 O(N)O(N)，显著提升扩展性。

## XRC domain

# 复用QP
## 多个连接共享QP
## DCT
DCT（Dynamic Connected Transport）

# 参考
```bash
# RDMA(5)服务类型：RC向左，UD向右
https://mp.weixin.qq.com/s/Auj5wY_ucKFGAxpF1zeCow
```