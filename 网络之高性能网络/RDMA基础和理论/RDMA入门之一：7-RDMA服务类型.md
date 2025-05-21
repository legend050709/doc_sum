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
### 适用场景
某个client 和 某个server端存在多个连接，多个连接复用一个QP。
如果每个 client 和 server 都只是存在一个连接，都将消耗一个QP，那么就无法多个连接共享QP。

## DCT
RDMA DCT（Dynamic Connected Transport）是一种用于提升RDMA（远程直接内存访问）连接可扩展性和资源效率的动态连接管理机制。
### 核心概念

### 适用场景


# 参考
```bash
# RDMA(5)服务类型：RC向左，UD向右
https://mp.weixin.qq.com/s/Auj5wY_ucKFGAxpF1zeCow
```