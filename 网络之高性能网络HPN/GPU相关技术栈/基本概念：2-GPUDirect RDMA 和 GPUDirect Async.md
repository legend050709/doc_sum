```table-of-contents
```
# GPUDirect演进历史
## GPUDirect v1 (2010) - Shared Pinned Memory
## GPUDirect v2 (2011) - Peer-to-Peer (P2P)
## GPUDirect RDMA (2013)
## GPUDirect Storage (2020)
## GPUDirect Async (2021+)

# GPUDirect RDMA
##  介绍
**GPUDirect RDMA(GDR)**  是NVIDIA 提供的一项技术，允许第三方PCIe设备（主要是 RDMA NIC）直接通过PCIe总线与GPU的内存进行数据交换，而无需经过CPU或主机内存。

```bash
GPUDirect RDMA enables third-party devices (like RDMA NICs) to directly access GPU memory over PCIe, bypassing the CPU and host memory, enabling low-latency and high-bandwidth data transfers between GPUs across nodes.
```

核心点：
- **RDMA NIC 通过PCIe 直接访问 GPU memory**
- **绕过 CPU 和  host memory**
- **降低 latency，提高 bandwidth**

在传统的数据传输过程中，如果网卡要从GPU读取数据，数据通常需要先拷贝到主机内存，再由CPU转发到GPU内存。这增加了延迟并占用CPU资源。  GPUDirect RDMA 技术绕过了这个中间环节，允许支持RDMA（远程直接内存访问）的设备直接读写GPU显存。



## GDR数据传输路径
要清晰地理解GPUDirect RDMA（GDR）的价值，最好的方式就是对比“有GDR”和“没有GDR”时，数据从一台服务器的GPU到另一台服务器GPU的完整旅程。

我们以Client Server架构为例，假设Client端的GPU要将一批训练好的模型梯度数据，发送给Server端的GPU进行聚合。

### 传统方式 (无 GDR)

```bash
Client Node
───────────
GPU Memory
    │
    │ PCIe DMA
    ▼
Host Memory
    │
    │ RDMA DMA
    ▼
NIC
    │
    │ Network
    ▼
NIC
    │
    │ RDMA DMA
    ▼
Host Memory
    │
    │ PCIe DMA
    ▼
GPU Memory
───────────
Server Node
```

1. **准备 (Client端)**：GPU内的数据需要发送，CPU介入，将数据从GPU显存拷贝到主机内存的一个临时缓冲区。
2. **装车 (Client端)**：CPU通知网卡，数据已在主机内存中。网卡通过DMA方式，将数据从主机内存拷贝到自己的网卡缓冲区。
3. **运输**：网卡将数据打包，通过网络发送到Server端。
4. **卸货 (Server端)**：Server端的网卡收到数据，同样通过DMA方式，将数据放入主机内存的临时缓冲区。
5. **入库 (Server端)**：CPU再次介入，将数据从主机内存拷贝到目标GPU的显存中，供GPU计算使用。

可以看到，数据在整个过程中被拷贝了四次，CPU在这两端都承担了繁重的搬运和协调工作。

### 有GDR
```bash
Client Node
───────────
GPU Memory
    │
    │ RDMA DMA (PCIe P2P)
    ▼
NIC
    │
    │ Network
    ▼
NIC
    │
    │ RDMA DMA (PCIe P2P)
    ▼
GPU Memory
───────────
Server Node
```
启用GPUDirect RDMA后，一切都变得简单高效。

1. **直装 (Client端)**：CPU只负责“告诉”网卡：“GPU显存里的数据已经准备好了，地址是XXX，你去取吧。”随后，支持GDR的网卡通过PCIe总线，直接发起一个RDMA读操作，将数据从GPU显存拷贝到自己的网卡缓冲区。
2. **直运**：网卡将数据打包，通过网络发送到Server端。
3. **直卸 (Server端)**：Server端的网卡收到数据包后，识别出这是一个RDMA操作。它根据数据包里的目标地址信息，通过PCIe总线，直接将数据写入目标GPU的显存中。

在这个流程中，数据从源GPU显存直达目标GPU显存，主机内存被完全绕过，CPU仅在初始阶段下达指令，之后便不再参与。

### 对比

![](attachments/deepseek_mermaid_20260316_7a06c7.png)

|传输阶段|无 GPUDirect RDMA 的传统路径|有 GPUDirect RDMA 的优化路径|
|---|---|---|
|**路径总览**|**GPU显存 → 主机内存 → 网卡 → ... → 主机内存 → GPU显存**|**GPU显存 → 网卡 → ... → 网卡 → GPU显存**|
|**数据拷贝次数**|发送端2次 + 接收端2次 = **共4次**|发送端1次 + 接收端1次 = **共2次**|
|**CPU参与度**|**全程深度参与**。CPU负责从GPU拷贝数据、通知网卡、处理中断等，负担很重。|**基本不参与**。数据的读写完全由网卡和GPU通过PCIe总线直接完成，CPU只在最初协调。|
|**关键瓶颈**|**主机内存带宽**和**CPU处理能力**成为数据流动的新瓶颈，限制了GPU和网络的性能发挥。|**PCIe带宽**是主要瓶颈，但这是GPU和网卡通信的“直连通道”，效率远高于经过CPU。|


# GPUDirect Async
## 介绍
**GPUDirect Async (GDA)** 是一项允许GPU直接触发与第三方设备（如网卡）通信操作的技术，即 允许 GPU kernel 直接触发网络或 IO 操作，而不需要 CPU 参与。

```bash
GPUDirect Async is a technology that enables direct synchronization and communication triggering between a GPU and a third-party device, such as a network interface card (NIC) without CPU intervention.
```

## 背景
在GDA出现之前，即使有了GDR，通信流程依然存在瓶颈。GDR解决了**数据路径**的问题（数据不用经过CPU内存），但**控制路径**（谁负责发号施令、检查任务是否完成）依然由CPU主导。

**传统 GPU 网络通信**，通信流程如下：
```bash
1. GPU 计算完成，需要将数据发送给其他节点。
2. GPU 通知 CPU：通过中断或轮询告知 CPU“数据准备好了”。
3. CPU 介入控制：CPU 收到信号后，执行上下文切换，运行通信库（如 MPI, NCCL），构建网络描述符（Work Queue Entries, WQEs）。
4. CPU 触发网卡：CPU 将描述符写入网卡（NIC）的门铃寄存器（Doorbell），命令网卡开始传输。
5. 数据传输：网卡通过 DMA 直接从 GPU 显存读取数据并发送（这一步是 GDR 做的，很快）。

```

问题所在：虽然数据搬运（步骤 5）很快，但**步骤 2、3、4 完全依赖 CPU**。在高并发、低延迟的大模型训练场景中，频繁的 CPU 中断、上下文切换和队列管理成为了巨大的延迟来源，且占用了宝贵的 CPU 算力。

```bash
GPU kernel 完成计算
        │
        ▼
CPU 检测 GPU event
        │
        ▼
CPU post RDMA request
        │
        ▼
NIC 发包

流程图：
GPU kernel
    │
    ▼
CPU polling / interrupt
    │
    ▼
CPU post RDMA WQE
    │
    ▼
NIC send
```

在这个流程中，CPU就像一个频繁介入的“调度员”。对于需要频繁、细粒度通信的现代应用（如大规模分布式训练、实时数据分析），这种CPU的频繁介入带来了两大弊端：
- **增加延迟**：每一次通信都需要CPU参与，增加了往返时间。
- **消耗CPU资源**：CPU需要不断地轮询和同步，占用了宝贵的CPU核心周期，这些周期本可以用于运行其他应用或操作系统服务

## GDA解决的问题
GDA 的核心目标：**让 GPU 直接驱动 NIC**。 即 GPU 可以`GPU kernel → NIC doorbell`，而不需要 `GPU → CPU → NIC`，这样GPU可以直接启动 RDMA。

GDA 解决上述**控制路径的瓶颈**，实现CPU与GPU的**控制流解耦**。具体来说，它解决了以下问题：
**（1）消除CPU调度延迟（Low Latency）**：允许GPU直接“敲门”（Ring Doorbell）通知网卡开始工作，不再需要CPU作为中间人转发指令。
**（2）降低CPU占用率 (CPU Offloading)**：将CPU从高频的通信轮询和同步操作中解放出来。CPU只需在初始设置时参与，之后的数据传输和同步由GPU和网卡自主完成。
**（3）GPU的计算和通信融合**：通信操作的发起由GPU控制，可以和与 GPU 的计算流（CUDA Stream）紧密绑定。GPU 可以在一个流中计算，同时在另一个流中发起通信，或者在计算指令后紧接着插入通信发起指令，实现真正的**计算与通信重叠 (Overlap)**，且无需等待 CPU 调度。

## 流量路径图
GDA改变的是 **控制路径**（control path），不改变数据路径，数据路径通常仍然是 **GDR**。

下面我们通过Client端发送数据给Server端的例子，通过两张流量路径图，来看看GDA带来的根本性变化。这实际上是GDR和GDA协同工作的完美状态。

### 场景一：仅有GDR，无GDA（控制路径依然经过CPU）
这张图展示了在有GDR但无GDA的情况下，Client端发送数据的完整流程。GDR虽然实现了数据直通，但整个通信的“指挥权”依然在CPU手里。

![](attachments/deepseek_mermaid_20260316_581b68.png)

```bash
GPU kernel
     │
     ▼
CPU poll event
     │
     ▼
CPU post RDMA WQE
     │
     ▼
NIC doorbell
```

### 场景二：GDR + GDA 协同工作（数据+控制全优化）
这张图展示了GDR和GDA协同工作时的完美状态。GDA接管了“指挥权”，让GPU能够自己管理整个通信流程。

```bash
Client Node
GPU kernel
    │
    │ trigger
    ▼
NIC
    │
    │ Network
    ▼
NIC
    │
    │ RDMA DMA
    ▼
GPU memory
Server Node
```

![](attachments/deepseek_mermaid_20260316_09959d.png)

## GDA 的技术原理
GDA 的核心思想是：将**通信控制的发起权**从 CPU 彻底移交给 GPU。即：将通信控制平面从CPU卸载到GPU，让GPU的流多处理器（SM）通过MMIO敲门铃(doorbell)，能够直接驱动网卡。即：**GDA 允许 GPU kernel 通过 MMIO 直接访问设备寄存器**，从而触发 RNIC 等设备执行DMA操作；也就是：`GPU SM → PCIe MMIO → Device doorbell`

```bash
GPU persistent kernel
      │
      │ write WQE
      ▼
Send Queue
      │
      │ MMIO
      ▼
NIC doorbell
      │
      ▼
NIC DMA data
```

在这个模型下，整个数据传输的控制流程完全由**GPU内核中的CUDA线程**主导。
![](attachments/deepseek_mermaid_20260316_2c6967.png)

## GDA的关键组件协同

### NVIDIA GPU (Hopper/Blackwell 及以后架构)
内置了专门的硬件逻辑或通过新的 CUDA API（如 `cudaGraph` 扩展或特定的 NVSHMEM/NVLink 增强接口），支持直接构造网络操作描述符。

### 支持 GDA 的网卡 (ConnectX-7/BlueField-3 及更新)
网卡固件和驱动程序经过特殊优化，能够识别并接受来自 GPU PCIe 总线直接写入的控制指令，而不仅仅接受来自 CPU 的指令。

### 通信库 (NCCL / MPI)
上层软件栈（如 NCCL）被重构，不再为每个小包去调用 CPU 系统调用，而是预先配置好通信图（Communication Graph），由 GPU 在运行时自动触发。

## 问题
### GDA情况下，Poll CQ是CPU来处理还是GPU？
#### 模式1：CPU poll（主流，今天最常见）

```bash
流程：

GPU → NIC   （doorbell / WQE via GDA）  
NIC → GPU   （DMA data）  
  
NIC → Host Memory（CQE）  ，CPU → poll CQ → 通知 GPU
```

即：GPU 发请求, CPU 收 completion

# 参考
```bash

```