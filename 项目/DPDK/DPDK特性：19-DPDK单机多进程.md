```table-of-contents
```
# 多个 DPDK 进程绑定同一个 PCIe 网卡，会不会冲突？
## 结论
在未开启多进程共享 PMD（multi-process / Primary + Secondary）模式的情况下，多个独立 DPDK 程序不能同时绑定同一块网卡的同一个 PCIe 设备。绝大多数驱动中会直接冲突，初始化失败。

多个独立 DPDK 程序如果尝试绑定同一 NIC：
- **第一个进程成功初始化驱动**
- **第二个进程会失败**，典型错误如：
```bash
EAL: Cannot reserve memory
ethdev: device already probed

```

## DPDK 中会自动分配不同队列给不同进程吗？
**不会。**

DPDK PMD 没办法做到“队列级别独占绑定”，因为：
- 驱动 probe PCIe 设备是设备级，不是队列级
- **一个进程初始化 NIC 必然初始化整个 device**
- 队列数是在 device init 时统一配置的（`rte_eth_dev_configure`）



## 原因
DPDK 的 PMD 默认是 **进程私有** 驱动：
- 每个进程要重新 probe PCIe 设备
- 多个 PMD 同时 probe 会导致硬件资源重复申请
- 队列配置、寄存器访问都是针对同一设备，无法并发

因此 **同一 NIC 不能被多个独立 DPDK 程序初始化**。




# 多个应用需要共享同一张 DPDK 网卡怎么办？
## 方案 1：DPDK Multi-process（Primary + Secondary）

**只有 primary 进程会初始化 PCIe 设备**
secondary 进程：
- 共享同一块 hugepage 内存
- 共享同一个 ethdev 结构
- 不会重复初始化 PCIe 设备

此模式下：
- 多个进程能共享同一张卡  
- 队列、内存池、ring 可分别管理

### 使用

- 使用 EAL 的 `--proc-type=primary/secondary`
- 共享 hugepage 和 file-prefix

```bash
./app1 --proc-type=primary   --file-prefix=dpdk0
./app2 --proc-type=secondary --file-prefix=dpdk0
```


## 方案 2：SR-IOV，将网卡切成多个 VF

每个 VF：
- 有独立队列
- 具有独立 PCI 设备号（0000:xx:xx.x）
- 各自可被独立 DPDK 程序绑定


比如：
```bash
PF → 创建 8 个 VF → 8 个 DPDK 程序独立使用
```

# 其他
## 基于libibverbs的RDMA程序，可以在单个机器上启动多个RDMA程序使用同一个RNIC

RDMA NIC（如 Mellanox ConnectX）本身就是：
- 多 QP、多 SQ/RQ、多 CQ 的硬件
- 设计目标就是支撑 **多个应用同时使用**;

每个进程都可以：
- 创建自己的 `Protection Domain（PD）`
- 创建自己的 `Queue Pair（QP）`
- 创建自己的 `Completion Queue（CQ）`
- 注册自己的 `Memory Region（MR）`

这些资源在NIC 中是天然隔离的。


# 参考
```bash

```