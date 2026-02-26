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


# 单机中跑多个DPDK程序

在单机运行多个进程时，必须处理好以下资源的冲突：

|**资源**|**冲突后果**|**解决办法**|
|---|---|---|
|**大页内存**|进程启动失败或 OOM|使用不同的 `--file-prefix` (独立模式) 或共享内存池 (主从模式)|
|**CPU 核心**|严重的性能下降 (100% 占用抢占)|严格使用 `-l` 参数分配不重叠的核心|
|**网卡 (NIC)**|初始化冲突|使用网卡分出来的 **SR-IOV VF**，或者通过 `-a` 指定不同的 BDF 号|
|**日志/运行时文件**|无法创建 PID 文件|必须使用 `--file-prefix` 区分 `/var/run/dpdk/` 下的目录|


## 多个应用绑定不同的DPDK网卡(独立进程模式)
如果你希望单机上的多个 DPDK 进程完全隔离，互不干扰（例如运行两个完全不同的 DPDK 应用），则需要进行**资源彻底隔离**。

- **方法关键点**：
    1. **内存隔离**：使用不同的 `--file-prefix`。这样每个进程会创建自己独立的大页内存映射文件。
    2. **核心隔离**：使用 `--lcores` 或 `-c` 确保不同进程运行在不同的物理核上，避免上下文切换抖动。
    3. **设备隔离**：使用 `-a` (allow) 或 `-b` (block) 参数。例如进程 A 只接管 `0000:01:00.0`，进程 B 只接管 `0000:01:00.1`。

比如：
```bash
进程 A: ./app1 --file-prefix=p1 -a 01:00.0 -l 1-2
进程 B: ./app2 --file-prefix=p2 -a 01:00.1 -l 3-4
```

### 内存隔离
使用不同的 `--file-prefix`。

大页内存是通过mmap将某个大页文件映射到程序内存中；
如果每个进程存在不同的`--file-prefix`，意味着mmap映射不同的文件，这样就可以做到了内存的隔离。

### 设备隔离
设备隔离 其实本质上还是 **Doorbell 寄存器**独占。

不同的应用程序绑定不同的网卡，就可以做到 `Doorbell 寄存器`独占。确保了两个进程的 DMA 操作互不干扰。


## 多个应用共享同一张 DPDK 网卡时
### 方案 1：DPDK Multi-process（Primary + Secondary）


DPDK 原生支持 **Primary / Secondary 进程模型**：多个进程共享同一份大页内存（Hugepages）和网卡资源。

**Primary process**：负责初始化共享资源（内存、内存池、网卡）。必须先启动。
- 初始化 EAL
- 初始化 PCI 设备
- 创建 memzone / ring / mbuf pool

**secondary 进程**：通过挂载到主进程创建的共享内存来运行。可以有多个。
- 不会重复初始化 PCIe 设备，直接 attach 已存在的资源。比如：共享同一块 hugepage 内存

**注意**：所有参与的进程必须使用相同的 `--file-prefix` 参数，以便它们能找到同一个大页内存配置文件。hugepage 只分配一次。

#### 使用

- 使用 EAL 的 `--proc-type=primary/secondary`
- 共享 hugepage 和 file-prefix

```bash
# Primary
./app_primary \
  -l 0-3 \
  -n 4 \
  --file-prefix=dpdk0 \
  --proc-type=primary

# Secondary
./app_secondary \
  -l 4-5 \
  -n 4 \
  --file-prefix=dpdk0 \
  --proc-type=secondary

```


### 方案 2：SR-IOV，将网卡切成多个 VF

每个 VF：
- 有独立队列
- 具有独立 PCI 设备号（0000:xx:xx.x）
- 各自可被独立 DPDK 程序绑定


比如：
```bash
PF → 创建 8 个 VF → 8 个 DPDK 程序独立使用
```


#### 基于容器的隔离 (Docker / K8s)

在现代云原生环境下，通常在不同的容器中运行 DPDK。

- **实现方法**：
    1. **挂载大页**：将主机的 `/dev/hugepages` 挂载到容器内部。
    2. **特权模式**：通常需要 `--privileged` 权限来访问 PCI 设备。
    3. **资源分配**：利用 Cgroups 限制每个容器可见的 CPU 核心。
    4. **设备分配**：通过 `vfio-pci` 将不同的网卡 VF（虚拟功能）透传给不同的容器。


## 小结

- 如果你需要**进程间通信（如共享 Ring）**：请选择 `Primary/Secondary` 模式。
- 如果你需要**业务完全解耦**：请选择 `Independent` 模式(即：不同进程绑定不同的网卡)并配合不同的 `--file-prefix`。
- 如果你需要**最大化利用单网卡**：请开启网卡的 `SR-IOV`，将不同的 `VF` 分配给不同的独立进程。
- `Mellanox`网卡具有其特殊性，可以直接在单机上启动多个`DPDK`程序，或者是`RDMA`程序。

# Mellanox网卡的特殊性

## Mellanox网卡单机启动多个DPDK程序

### 现象
为什么使用Mellanox网卡在单机上运行多个DPDK程序的时候，不使用`primary/secondry`, 以及不使用`PF/VF`也可以，在单机上运行多个DPDK程序？


### 范例
基于`libtpa`用户态协议栈的的`DPDK`应用程序，在存在`Mellanox`网卡的单个机器上跑`Client`和`Server`是没有问题的。
首先`client`和`server`都会下发不同的`Port`的`Flow director`规则给网卡, 由于`client`和`server`的`port`的使用不会交叉和重叠， 收到流量时网卡进行`Flow director`规则匹配知道将流量送给哪个队列，进而给哪个进程的哪个线程。不会造成流量错乱。

`client`和`server`的`port`的使用不会交叉和重叠的保证：用户态每次占用一个`port`的时候，都会通过系统调用`bind`来占住该端口；如果bind失败，就会换其他的端口。

### 分析

传统的 Intel 网卡（使用 `igb_uio` 或 `vfio-pci`）在 DPDK 接管后，网卡会完全从内核态“断开”，由 DPDK 独占。这种模式下，如果不分 VF，第二个进程就无法再访问该硬件。
而 Mellanox 使用的是 **`mlx5_core`** 驱动，它基于 Linux 标准的 **RDMA 子系统（ibverbs）**：
- **硬件共用**：内核驱动 `mlx5_core` 始终控制硬件资源。
- **用户态映射**：DPDK 通过 `libibverbs` 和 `libmlx5` 将硬件队列直接映射到用户态。
- **多上下文支持**：内核驱动允许创建多个独立的 `ibv_context`。每个 DPDK 进程都可以创建一个属于自己的上下文，直接向网卡申请独立的硬件队列对（Queue Pairs）。

```bash
NIC FW
 ├── Protection Domain (PD)
 ├── Memory Region (MR)
 ├── Completion Queue (CQ)
 ├── Queue Pair (QP)
 ├── Flow Table

这些对象全部是 per-process / per-context 的
```

当你启动一个 DPDK 程序，每个进程：
- 打开自己的 `ibv_context`
- 分配自己的 PD
- 注册自己的 MR
- 创建自己的 QP / CQ / flow
    
**完全没有全局寄存器锁的概念**



### 注意事项
- **流量冲突**：如果两个进程都配置了 `promiscuous`（混杂模式），网卡可能无法确定包该给谁（通常会根据规则匹配优先级）。
在单机上同时启动两个进程，并使用 `rte_flow` 给它们设置不同的目的地（例如根据不同的 `dst_port`），你会发现 Mellanox 可以在不开启 `SR-IOV` 的情况下，在硬件层面完美实现流量分流。

- **资源占用**：网卡的硬件队列总数是有限的（如 1024 个），如果启动进程过多，可能会耗尽硬件队列。

### 实现

在 Mellanox 网卡上，你可以启动两个完全独立的进程（使用不同的 `--file-prefix`），它们都 `-a` 绑定同一个 PCI 地址：

1. **资源分配**：当进程 A 启动时，它向内核申请了一组 Rx/Tx 队列；当进程 B 启动时，它同样向内核申请了另一组独立的 Rx/Tx 队列。
    
2. **隔离性**：虽然物理 PCI 地址相同，但硬件内部通过不同的 **Doorbell 寄存器页面** 和 **内存保护域（Protection Domain）** 确保了两个进程的 DMA 操作互不干扰。
    
3. **流量分发（关键）**：
    
    - **默认行为**：如果不做特殊处理，流量通常只会去往其中一个进程，或者根据网卡的默认流控规则分发。
        
    - **流量分流**：由于 Mellanox 支持 **Flow Isolation** 和特殊的流量引导规则，你可以让进程 A 接收 VLAN 10 的流量，进程 B 接收 VLAN 20 的流量，即使它们使用的是同一个物理网口。



### 小结
在 Mellanox 网卡上，你“看似”在跑多个 DPDK 程序控制同一 PF，实际上是 `mlx5/FW` 在替你做硬件对象级隔离，而不是 DPDK 在管理多进程。

在 Mellanox 网卡（mlx5 PMD）上：**多个 DPDK 进程之所以能同时跑，并不是 DPDK 多进程模型在起作用，而是 mlx5 把“硬件独占”问题下沉到了内核 / 固件 / RDMA verbs 层。**
换句话说：**你跑的不是“多个 DPDK 控制同一网卡”，而是“多个进程通过 verbs 各自创建硬件对象”。**

## Mellanox网卡单机启动多个RDMA程序

RDMA NIC（如 Mellanox ConnectX）本身就是：
- 多 QP、多 SQ/RQ、多 CQ 的硬件
- 设计目标就是支撑 **多个应用同时使用**;

每个进程都可以：
- 创建自己的 `Protection Domain（PD）`
- 创建自己的 `Queue Pair（QP）`
- 创建自己的 `Completion Queue（CQ）`
- 注册自己的 `Memory Region（MR）`

这些资源在NIC 中是天然隔离的。

## Mellanox网卡和其他网卡的对比

### Intel NIC vs Mellanox NIC 架构对比

#### Intel NIC：寄存器独占
```bash
User App
 └── DPDK PMD (i40e/ixgbe)
      ├── MMIO 寄存器
      ├── 全局 RX/TX Queue 配置
      ├── 全局 RSS
      ├── 全局 Flow Director
      └── PF reset / link state

```

##### 关键特征

- RX/TX queue 是 **端口级全局资源**
    
- Queue enable / disable 会写寄存器
    
- PF reset 会清掉所有队列
    
- PMD 假设：**我就是唯一控制者**
    

**多进程必须靠 DPDK 自己做协调**
- primary / secondary
- PF / VF

#### Mellanox NIC 
```bash
User App
 └── DPDK mlx5 PMD
      └── libibverbs
           └── mlx5 kernel driver
                └── NIC FW
                     ├── PD (Protection Domain)
                     ├── MR (Memory Region)
                     ├── CQ
                     ├── QP
                     ├── Flow Table

```


##### 关键特征

- **没有“启用队列寄存器”**
- QP / CQ 是“硬件对象”
- 对象绑定到 PD
- PD 绑定到 process context


### 小结
`dpdk mlx5 PMD`:  mlx5 PMD 更像是“verbs 适配器”，而不是“传统 DPDK PMD”
这也是为什么：
- 它行为像 RDMA
- 多进程“看起来能跑”

## Mellanox网卡和PF/VF虚拟化
### PF/VF 在 Intel NIC 里的作用

- PF/VF = **硬件级队列/寄存器隔离**

- 防止多个进程互相踩寄存器

### mlx5 本身就“天然虚拟化”
ConnectX 网卡：
- 每个 QP / CQ 都是硬件对象
- 每个对象都属于某个 PD
- PD 绑定到某个进程上下文


本质上： **你已经在用“隐式 VF”**

这也是为什么：
- RDMA 本身就支持多进程
- DPDK mlx5 只是“借用”了这套机制

#### 共享的资源信息

下面这些是**共享的、会互相影响的**：

##### 1️、端口级资源（全局）

- Link up/down
- MTU
- Port speed
- PFC / ECN
- RoCE 参数
    

任何一个进程修改：`rte_eth_dev_set_mtu()`，其他进程全部受影响

##### 2、Flow steering 资源（rte_flow规则）
- TCAM / flow table 是全局的
- 多进程下容易：
    - flow 插入失败        
    - 优先级冲突
    - 性能下降


##### 3、Reset / error recovery

某个进程触发：
- device reset
- FW fatal  

所有进程一起死；



# 参考
```bash

```