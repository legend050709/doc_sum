```table-of-contents
```
# RNIC微架构
![](attachments/Pasted%20image%2020250318152739.png)
![](attachments/Pasted%20image%2020250318152748.png)

RNIC属于智能网卡的一类，是一种专门设计用于支持RDMA技术的网卡。它在硬件层面实现RDMA的功能，并提供用于数据传输的高性能网络接口。

## RNIC组成
除了数据包缓冲区（TX/RX Buffer），RNIC还具有多个处理单元（PU,Processing Unit）和各种类型的内部高速缓冲区，这些不同的高速缓冲区存储特定类型的数据。如ICM Cache，用来存储QP的上下文（QPC）；内存转换表（MTT）内存保护表（MPT）；WQE Cache，存储预取的SQ和RQ队列中的WQE。

### ICM Cache(内部上下文内存缓存)
**ICM Cache**（Internal Context Memory Cache）是RNIC（RDMA Network Interface Card）内部用于缓存关键控制数据结构的高速存储区域，旨在加速对频繁访问的元数据（如QPC、地址转换表等）的访问，从而降低延迟并提升性能。

ICM 是 NIC 用来存储各种 RDMA 对象上下文（context）的“逻辑内存空间”；包括：
```bash
- QP context（QPC）
- CQ context（CQC）
- SRQ context（SRQC）
```

#### ICM Cache的硬件实现
1. **存储介质**
    
    - 通常由RNIC芯片上的SRAM（静态随机存取存储器）实现，提供纳秒级访问速度。
    - 容量较小（几MB级），需通过高效缓存算法管理热点数据。
2. **缓存管理策略**
    
    - **LRU（最近最少使用）**：淘汰长时间未访问的条目。
    - **预取（Prefetching）**：根据访问模式提前加载可能需要的元数据。
    - **一致性机制**：确保缓存与主存中元数据的同步（如写回/写穿策略）。
3. **性能影响**
    
    - **命中（Hit）**：直接从ICM Cache获取数据，延迟通常为数十纳秒。
    - **未命中（Miss）**：需从主存或片上内存加载数据，延迟增加至数百纳秒甚至微秒级。

### 流程
以一次RDMA write操作为例：

Step 0：CPU通过Doorbell机制（敲门铃，一种通信机制）通知RNIC上的处理单元（PU）有请求待处理。

Step 1：为了获取和处理该请求（WQE），RNIC需要先从各个高速缓冲区中获取相应的元数据（例如：从ICM中获取对应QP的QPC从而定位WQE，检查WQE Cache中是否预存需要的WQE）；

Step 2：发生缓存未命中时RNIC通过PCIe总线从主机DRAM中获取相应的元数据（图3中红线）。取得后RNIC处理该WQE，从中获得主机待发送数据的DRAM地址和对端放置数据的地址等相关信息；

Step 3：RNIC发出DMA请求读取主机DRAM中的数据，然后将数据处理成网络包放到TX队列发出。

为了简单起见，未示出对称接收端的操作。

# RDMA内存管理概述
## MR


远端想要访问本地内存，首先需要本地的“同意”，即本地仅仅暴露想要暴露的内存，其他内存远端则不可访问。
该“同意”操作，我们一般称为 Memory Region （简称 MR） 注册，操作系统在收到该请求时会**锁住（pin）** 该段申请内存，防止内存被 swap 到硬盘上，同时将这个注册信息告知 RDMA 专用网卡，RDMA 网卡会保存虚拟地址到物理地址的映射。

![](attachments/Pasted%20image%2020260401102708.png)


经过此番操作，由 MR 代表的内存暴露出来了，远端可以对其进行访问。
出于安全的考虑，只有被允许的远端才可以访问，这些远端持有远端访问密钥，即Remote Key（简称 RKey），只有带有正确 RKey 的请求才能够访问成功。

### MR和PD、QP、ib_ctx的关系

```bash
struct ibv_context *ibv_open_device(struct ibv_device *device);
struct ibv_pd *ibv_alloc_pd(struct ibv_context *context);


struct ibv_mr *ibv_reg_mr(struct ibv_pd *pd, void *addr, size_t length,
                       int access);

struct ibv_qp *ibv_create_qp(struct ibv_pd *pd,
                          struct ibv_qp_init_attr *qp_init_attr);


struct ibv_srq *ibv_create_srq(struct ibv_pd *pd,
                            struct ibv_srq_init_attr *srq_init_attr);


struct ibv_cq *ibv_create_cq(struct ibv_context *context, int cqe,
                          void *cq_context,
                          struct ibv_comp_channel *channel,
                          int comp_vector);
---------------------------------------------------------

                ┌──────────────────────────┐
                │       ib_context         │
                │   (一个 RDMA 设备)       │
                └──────────┬───────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────────┐        ┌────────┐        ┌────────┐
   │   PD   │        │   CQ   │        │   CQ   │
   │(隔离域)│        │完成队列│        │完成队列│
   └───┬────┘        └────────┘        └────────┘
       │
       │
 ┌─────┼──────────────────────────────┐
 │     │                              │
 │  ┌──────┐                    ┌────────┐
 │  │  QP  │                    │  SRQ   │
 │  │队列对│                    │共享RQ  │
 │  └──┬───┘                    └──┬─────┘
 │     │                           │
 │     │                           │
 │     └──────────────┬────────────┘
 │                    │
 │               ┌────────┐
 │               │   MR   │
 │               │内存区域│
 │               └────────┘
 │
 └────────────────────────────────────
 
```

![](attachments/deepseek_mermaid_20260319_e9d21a%201.png)


**关系说明：**

- ib_ctx: 代表一个 RDMA 设备/HCA
- 设备下可以创建多个 **CQ**（图中粉色）和多个 **PD**（图中蓝色）。
- 每个 PD 内部可以创建 **QP**、**SRQ**、**MR**。
- QP 需要关联到 CQ。
- SRQ 可以被同 PD 内的多个 QP 共享。
- MR： MR 注册的内存，只允许同一个 PD 里的 QP/SRQ 访问


#### PD
`PD` 是一个**设备内的逻辑隔离单元**。
同一个设备上可以创建多个 `PD`，每个 `PD` 内部的 `MR` 只能被同 `PD` 内的 `QP` 访问。
这种隔离机制允许不同应用在同一设备上安全地共享硬件资源，而不会互相干扰。因此，创建 `PD` 时必须指定它属于哪个设备（即 `ib context`），因为 `PD` 本身是设备的一个子资源。

比如：
- 在 PD-1 中注册的 MR1，其 LKey 可能是 `0x123`。
- 在 PD-2 中注册的 MR1，其 LKey 也可以是 `0x123`。
- 硬件在执行 DMA 时，会将 ==`PD_ID + LKey`== 组合作为唯一标识去查表。如果 QP (属于 PD-1) 拿着 PD-2 的 Key 来访问，硬件会直接报错 (Access Error)。


### MR的属性
MR 包括以下属性：
（1）context：上下文
（2）addr：MR 的 Buffer 首地址
（3）length：MR 的 Buffer 长度
（4）lkey：MR 的 local key，唯一标识，在本端执行 RDMA API 操作中用来验证访问权限。
（5）rkey：MR 的 remote key，唯一标识，在远端执行 RDMA API 操作中用来验证访问权限。

#### 内存秘钥Keys

在RDMA通信中，内存区域（Memory Region，MR）的访问是通过两种密钥来控制的：本地访问密钥（local access key，L_Key）和远程访问密钥（remote access key，R_Key）。这两个密钥作为MR的标识符，并在MR注册过程中生成。

L_Key 总是在注册MR时生成，并用于控制本地用户对MR的读写访问。本地用户在这里指的是在同一台主机上运行的进程或者应用程序，它们可以使用L_Key来直接访问内存。  

R_Key 用于控制远程用户对MR的读写访问，这个密钥在注册MR时必须明确请求。远程用户指的是通过网络进行通信的其他主机上的进程或应用程序。R_Key对于RDMA通信至关重要，因为没有它，远程用户将无法执行RDMA操作，即无法直接从远程主机读取或写入内存区域。


### 限制
MR注册，会创建两个key (local和remote)指向需要操作的内存区域，注册的keys是数据传输请求的一部分。

RDMA硬件对用来做数据传输的内存是有特殊要求的。
（1）**在数据传输过程中，应用程序不能修改数据所在的 内存**。
（2）**操作系统不能对数据所在的内存进行swap out操作**；
即物理地址和虚拟地址的映射必须是固定不变的。

### MR注册的流程
RDMA要求应用程序在使用内存前显式注册内存区域（Memory Region, MR），此时驱动会：
 a. Pin内存：锁定物理内存，阻止操作系统换出（swap)。
 b. 构建地址映射表（如MTT）：将虚拟地址映射到物理地址，供网卡直接访问。

### 什么是swap out?

因为**物理内存是有限的**，所以操作系统通过换页机制来暂时把某个进程不用的内存内容保存到硬盘中。当该进程需要使用时，再通过**缺页中断**把硬盘中的内容搬移回内存，这个就是swap out。

影响：`swap out`几乎必然导致`VA-PA`的映射关系发生改变。

### 什么是pin内存(锁定VA-PA的映射关系)？

pin内存也叫**锁页**，就是固定虚拟地址和物理地址的映射，**防止物理页面被swap**。

注：==只要涉及IO地址翻译，无论是IOMMU还是MTT都需要pin内存==。


由于HCA经常会绕过CPU对用户提供的VA所指向的物理内存区域进行读写，如果前后的`VA-PA`映射关系发生改变，那么我们在前文提到的`VA->PA`映射表将失去意义，HCA将无法找到正确的物理地址。

为了防止换页所导致的`VA-PA`映射关系发生改变，注册MR时会"Pin"住这块内存（亦称“锁页”），即锁定`VA-PA`的映射关系。也就是说，MR这块内存区域会长期存在于物理内存中不被换页，直到完成通信之后，用户主动注销这片MR。

### 使用RDMA的时候为什么需要pin内存？
使用RDMA的时候为什么要pin内存？
因为网卡要直接内存访问（DMA），RDMA操作(read/write等)得到的是虚拟地址，需要MTT（Memory Translation Table）进行地址翻译。
准确的说是虚拟地址（VA）到物理地址的翻译（PA），而**==pin内存是为了不让这个物理地址被其他的程序使用==**；

否则RDMA要访问虚拟地址A，本来对应的是物理地址a；如果不pin内存，这个时候操作系统把物理地址a进行swap，然后物理地址a又被另外一个进程拿去用了，这个时候RDMA再去访问虚拟地址A，经过MTT转换还是往a这个物理地址写，那就写了其他进程的内存了。

### 非RDMA场景应用也有DMA操作，为什么不用pin内存？
#### 内核协议栈
平时写程序用的都是虚拟内存，都要经过MMU地址翻译成物理内存，怎么我们写普通程序就不用pin内存？

传统TCP通信通过操作系统内核协议栈处理数据：用户态数据 → 内核态Socket Buffer → 网卡DMA缓冲区。
==内核自身负责分配和固定（pin）用于DMA的内核缓冲区（如sk_buff的数据部分，其实就是page），用户空间内存无需pin==。

因为数据会被复制到内核的固定区域。而 ==内核中网卡驱动分配出的内存本身就是物理地址固定的==。 并且通过内核的DMA映射接口（如dma_map_single()）临时建立DMA映射，完成后立即解除映射。

#### DPDK用户态协议栈
如果是用户态协议栈，那用户态协议栈本身要不要pin内存呢。一般用户态协议栈都基于DPDK，而DPDK一般使用hugepage，**hugepage是不支持swap的，因此不显式pin也是没有问题的**。
那如果就是用普通的匿名内存呢？这种方式还是需要pin的，否则可能导致IOMMU缺页。

### 为什么MMU不需要pin内存
因为**要地址翻译就要pin内存**。那CPU最常用的MMU解决进程VA到PA的翻译怎么不需要进程都pin内存呢，MMU缺页是很正常的现象。

这是因为内核处理了MMU的缺页中断（Page Fault），在其中进行了页面的重映射。

而**IOMMU一般不支持动态缺页处理，当IO设备尝试访问未映射的虚拟地址时，IOMMU不会触发类似CPU的缺页中断，而是直接导致DMA错误（如PCIe错误或系统崩溃）**。

不过也有一些新硬件设备支持触发的缺页机制（如ARM SMMUv3的STALL模型），允许IOMMU暂停DMA操作并通知操作系统重新映射内存，或者一些intel新CPU支持再IOMMU缺页时产生MMU-notifier提供内核处理，然而，当然这些是依赖硬件支持的。

### 小结

**Memory Region (MR)：**
用户通过`ibv_reg_mr`向对端暴露一块内存(`iova+va+len+key`)：
```bash
iova是对端来访问所用地址，
va是本地访问所用地址
```
**iova 和 va** 都需在硬件里翻译为**PA**，所以注册MR涉及建立**MTT**地址翻译表。
同时也需为MR指定访问权限如可读可写，这涉及建立**MPT表**。

MR所涵盖的物理内存需要在注册时**PIN住**，以避免DMA访问`swapped out`的页。
注册MR可以指定**MR的访问权限**(`local/remote read/write`)。

MR注册好之后会返回**LKEY和RKEY**，LKEY用于自己访问自己，RKEY用于别人访问自己。

**一片内存区可以多次注册MR**，每次可以设置不同的访问权限，每次都会返回不同的LKEY和RKEY。

## FMR
### 介绍
**FMR（Fast Memory Region/Registration）** 是RDMA中用于**优化内存注册/注销性能**的技术。
传统的内存注册（通过 `ibv_reg_mr`）涉及操作系统和硬件的交互，开销较大（尤其是频繁注册/注销的场景）。
FMR 通过 **预分配和复用内存注册资源** 来减少开销，==适用于需要频繁注册/注销内存的场景==（例如短生命周期的小块内存操作）。


## PA-MR
**PA-MR** 是 **Physical Address Memory Region（物理地址内存区域）** 的缩写，它是RDMA内存注册（Memory Registration）的一种特殊模式，直接基于物理地址（而非虚拟地址）注册内存。

### 核心概念
#### 物理地址直接暴露
PA-MR在注册内存时，直接使用**物理地址（Physical Address, PA）**而非虚拟地址（Virtual Address），从而绕过虚拟地址到物理地址的转换（Address Translation）过程。

- 传统MR（基于虚拟地址）：需要维护虚拟地址与物理地址的映射表（如页表），依赖RNIC的地址转换服务（ATS）或IOMMU。
- PA-MR：直接通过物理地址访问内存，省去地址转换开销，但需确保内存的物理连续性和权限控制。

### 使用场景

- **内核驱动或特权级应用**：需直接操作物理内存的场景（如内核态驱动、DPDK等高性能网络框架）。
- **低延迟需求**：避免地址转换延迟，适用于超低延迟通信（如高频交易、实时控制系统）。
- **物理内存预分配**：在虚拟化或容器化环境中，预先分配物理连续内存（如HugePages），确保PA-MR的高效访问。

### PA-MR的实现与限制
#### 实现方式
在注册MR时，通过RDMA API（如`ibv_reg_mr`）指定**IBV_ACCESS_PHYSICAL_ADDR**标志，显式声明使用物理地址。

#### 限制与注意事项
 **物理内存管理**：需确保内存的物理连续性（如通过`mmap`分配HugePages或预留内存）。
**安全性风险**：直接暴露物理地址可能被恶意利用（需结合PD保护域和密钥隔离）。
**平台依赖**：部分RNIC硬件或操作系统可能不支持PA-MR模式。

### API接口
参考: [# ibv_alloc_dm](https://man7.org/linux/man-pages/man3/ibv_alloc_dm.3.html)
```C
(1) 申请设备内存，并注册MR
ibv_alloc_dm
ibv_reg_dm_mr

（2）释放设备内存，取消注册MR
ibv_free_dm
ibv_dereg_mr

（3）设备内存和程序的虚拟地址之间传递数据
ibv_memcpy_from_dm
ibv_memcpy_to_dm
```

## MR 的 prefetch


## MW
为了内存管理的细粒度化，RDMA 还提供了 Memory Window（简称 MW），一个 MR 上可以分列出多块 MW，并且每一块 MW 上都可以自定义访问权限。

**Memory window (MW)：**
MW允许用户更灵活的控制远端对本地内存的访问：动态的授予和收回远端对MR的访问权限，给不同的远端以不同的访问权限，为MR内的不同range的小块内存授予不同的访问权限。注册MR时需要同时建立地址映射表和安全保护表，但注册MW仅需建立安全保护表，所以建立MW可以直接在用户态与硬件通信完成而不需要经过内核。 ibv_bind_mw用于建立MW，它其实也是向QP里post了一个请求。


## PD

由于用户可能需要与许多不同的目的地通信，但并不希望所有这些目的地都能够同等地访问其注册的内存，因此IBA提供了PD机制。
PD确保Memory Region和QP在相同的PD内才能被相互访问。PD实际上提供了一个更细粒度的访问控制。
例如，可以控制哪些内存区域可以被远程节点读取或写入，以及哪些QP可以处理这些请求。同时，PD简化了资源管理。在一个PD内部，可以轻松地管理和配置多个内存区域和QP，而不需要为每个资源单独设置访问权限。

在用户分配QP或注册内存之前，可以创建一个或多个PD。将之后的QP和Memory Region关联到特定PD。内存区域的`L_Keys`和`R_Keys`只有在相同PD中的QP上才有效。

## MR、MW和PD以及QP的关系

![](attachments/deepseek_mermaid_20260319_e9d21a%202.png)

除了上述中的 MR 和 MW，RDMA 中的内存管理还和 Protect Domain（简称 PD） 和 Queue Pair （简称 QP） 相关，这里不详细阐述这两个概念。下图详细介绍了，这些概念之间的依赖关系：

![](attachments/Pasted%20image%2020250317103530.png)

现有的 RDMA 开发接口，即 InfiniBand Verbs 接口（简称 IBV 接口）并没有显式地展现这种依赖关系，但**在实际使用中，任何不按规定顺序的资源释放都会造成错误**，而用户找到问题的根本原因则非常困难。
更进一步，当 MR 或者 MW 中的任何内存段被使用时，对应的 MR 和 MW 都不应该被释放或注销，这些在原有的 IBV 接口中也很难规范化。

## RNIC的片上内存和主机内存
### RNIC的片上内存
RNIC存在片上内存：可以理解为主要是作为缓存，类似于CPU上也是存在缓存。
大小大概是几M。

#### 存储内容
主要存储的是 QP的元数据(metadata), QPC、MTT缓存、CCS(拥塞控制状态), MPT缓存，WQE、CQE、统计和计数等。

##### QP元数据的主要信息

QP是RDMA通信的最小单元，其元数据定义了通信端点的行为和规则，主要包括以下内容：

4. **QP基础信息**
    
    - **QP编号（QP Number）**：唯一标识一个QP。
    - **QP状态（QP State）**：包括初始化（Reset）、就绪（Ready）、错误（Error）等状态。
    - **QP类型（QP Type）**：可靠连接（RC）、不可靠连接（UC）、不可靠数据报（UD）或扩展类型（XRC）。
5. **队列配置**
    
    - **发送队列（SQ）和接收队列（RQ）深度**：队列中可缓存的WQE数量。
    - **关联的CQ（Completion Queue）**：用于上报该QP的完成事件。
6. **权限与密钥**
    
    - **L_Key和R_Key**：用于本地和远程内存访问的权限验证。
    - **PD（Protection Domain）**：限制QP可访问的内存区域范围。
7. **网络与传输层参数**
    
    - **端口号（Port）**：QP绑定的物理端口（如InfiniBand的子网端口）。
    - **服务类型（Service Level, SL）**：定义数据包优先级和路由策略。
    - **地址句柄（Address Handle, AH）**：目标端的网络地址和路径信息。
8. **可靠传输参数（仅RC/UC类型）**
    
    - **序列号（PSN, Packet Sequence Number）**：用于数据包顺序校验和重传。
    - **超时与重传策略**：ACK等待时间、最大重传次数等。
    - **流控（Flow Control）**：窗口大小、信用值（Credits）管理。
9. **原子操作与内存序控制**
    
    - 支持原子操作的标志位（如Compare-and-Swap、Fetch-and-Add）。
    - 内存一致性模型配置（如宽松内存序/严格内存序）。

```bash
QP元数据示例：
{
  QP Number: 0x1234,
  State: Ready,
  Type: Reliable Connected (RC),
  SQ Depth: 128,
  RQ Depth: 64,
  CQ: 0x5678,
  PD: 0x9ABC,
  L_Key: 0xDEADBEEF,
  R_Key: 0xCAFEBABE,
  Port: 1,
  SL: 3,
  PSN: 0x0000FFFF,
  Retry Timeout: 100ms,
  AH: {目标IP、GID、LID等}
}
```

### QP数量上升性能下降的原因

conn是有状态的，这些状态会缓存在网卡的SRAM中，但是SRAM是有限的，混存的conn数量有限，当待请求的数据地址在网卡SRAM中的MTT/MPT没有命中的时候，网卡需要通过PCIe去在内存中的MTT和MPT进行查找，这是一个耗时的操作。尤其是当我们需要 `high fan-out（高扇出：指数据被大量独立的请求者或服务频繁访问）`、`fine-grained（细粒度: 指数据访问的操作粒度非常小）`的数据访问时，这个开销会尤为的明显。这或许就是性能下降的原因.

## MTT 和 MPT
RDMA使能硬件直接访问用户态应用软件分配的内存从而实现零拷贝和绕过内核以达到低延迟和高带宽。

**MTT和MPT被存储在内存中，但是RNIC的SRAM中会进行缓存**。
当RNIC接收到来自用户的READ/WRITE请求的时候，首先在SRAM中的缓存中查找用户请求的目标地址对应的物理地址以及这块地址对应的访问权限，如果缓存命中了，就直接基于DMA进行操作，如果没有命中，就得通过PCIe发送请求，在内存的MTT和MPT中进行查找

![](attachments/Pasted%20image%2020250318142525.png)

### RDMA工作流程
#### 通信前
应用软件必须向驱动软件注册内存，提供内存的起始虚拟地址、长度和访问权限（比如读、写）等信息；

驱动软件根据虚拟地址和长度，会构建内存翻译表（MTT），每一个MTT表项对应注册内存的一个物理页起始地址，MTT的大小就是注册内存的物理页个数。

驱动软件进一步向硬件注册内存。硬件会为注册内存构建内存保护表（MPT），MPT包含内存的起始虚拟地址、长度、访问权限和MTT地址等信息，并返回本端密钥（LKEY「 Local Key」）和远端密钥（RKEY「Remote Key」）给应用软件。LKEY/RKEY可以被硬件用来索引MPT。

![](attachments/Pasted%20image%2020250318113313.png)

#### 通信时

通信时，应用软件如果要发送或者接收数据，可将发送或者接收内存的虚拟地址、长度和LKEY告诉硬件，硬件会根据LKEY找到MPT，权限检查通过后，硬件根据MTT翻译虚拟地址成物理地址，从而读写发送或者接收内存的数据。应用软件如果要读写远端内存，可以将远端内存的虚拟地址、长度和RKEY提供给硬件，硬件会将其嵌入发送报文中，远端的RDMA硬件收到报文后根据RKEY找到MPT，权限检查通过后，根据MTT翻译虚拟地址成物理地址，从而读写目的内存的数据。

![](attachments/Pasted%20image%2020250318113329.png)

### MTT
#### 介绍
  MTT（Memory Translation Table）是RDMA硬件（如Mellanox InfiniBand网卡）内部的专用机制，用于将RDMA的虚拟内存地址（例如QP的虚拟地址）映射到物理内存页。

#### MTT作用
Memory Translation Table（内存转换表）；

- **作用**：
    - **虚拟地址到物理地址的转换**：当应用程序通过RDMA访问远程内存时，MTT用于将用户空间的**虚拟地址（VA）**转换为实际的**物理地址（PA）**。
    - **内存注册**：在RDMA中，应用程序需要预先将内存区域（Memory Region）注册到网卡，网卡通过MTT记录这些内存区域的虚拟地址与物理地址的映射关系。

- **典型场景**：
    - 当网卡需要直接访问某块注册内存时，若目标地址不在网卡SRAM缓存的MTT条目中，需通过PCIe到主机内存中查询完整的MTT，导致额外延迟。

#### MTT和 ATS
虽然有些高级网卡可以支持ATS（Address Translation Services，地址转换服务），但是也存在cache miss的情况。因此位置不同决定了MTT的访问性能更好；

当高级网卡支持ATS时，它可以通过缓存地址转换结果，减少对IOMMU（Input-Output Memory Management Unit）的频繁访问，从而提升数据传输性能。

#### MTT和IOMMU

同样是IO地址翻译，为什么不直接用IOMMU呢。这要从以IOMMU和MTT下几个方面的不同说起：

##### MTT和IOMMU的位置不同

OMMU是位于CPU RC中，每次地址反应是要经过CPU RC的，所以普通使用IOMMU的DMA不需要CPU参与，只是不需要CPU的计算core参与，但是还是要经过CPU uncore的，是root port下的设备共享的。
而MTT是位于网卡设备上的，因此MTT的访问会更加高效。

##### 通用和专用

IOMMU是设备通用的地址转换，而MTT是RDMA网卡内部的专用表结构，由驱动直接管理（绕过操作系统），支持更快的地址转换，而且不同网卡厂商可以针对RDMA场景定制优化。

举个例子，例如MTT 是位于主机内存（DDR）上的，如果硬件每次做地址转换都到主机内存查询一遍表，肯定会影响数据传输的效率。所以实际方案中还需要考虑一些机制来解决这种问题，比如有些网卡（如海思）引入了MR HEM，来先让硬件快速找到MR对应的数据结构，然后根据MR查找MTT，将数据缓存的前两个内存页的物理地址保存到MR HEM 表的对象中，如果应用程序要求传输的数据量比较少，都位于前两个内存页，就可以直接从MR HEM 表的对象中获取内存页的物理地址，从而简化查询MTT表的步骤。另外还可以将最近使用的MR HEM/MTT 表中的表项暂存到硬件内部，避免每次都到DDR 中查表。

此外，IOMMU通过Domain ID（如PCIe PASID）隔离设备内存访问，但无法感知RDMA的MR粒度。MTT提供了对每个MR的精确控制，避免IOMMU的过度泛化。

##### MTT和IOMMU的协同

对于RDMA来说有了MTT是不是就不需要IOMMU了呢？要看什么场景，比如在物理机没有虚拟化的时候，确实使用RDMA也没必要开启IOMMU。但是在虚拟化场景，MTT虽然完成了地址翻译，但是还是需要IOMMU进行内存校验的，以确保MTT翻译的地址在合法范围内，避免DMA到其他虚拟机的内存。


### MPT
Memory Protection Table（内存保护表）。
- **作用**：
    - **权限控制**：MPT记录每个注册内存区域的访问权限（如读、写、原子操作），确保只有授权的RDMA队列对（QP）可以访问该内存。
    - **安全性**：防止未授权的QP或节点篡改或读取其他节点的内存。
- **典型场景**：
    - 若网卡在处理数据包时发现当前QP没有目标内存区域的访问权限（MPT未命中缓存），需通过PCIe查询主机内存中的完整MPT，验证权限后才能继续操作。

### 为什么需要MTT/MPT？
RDMA的核心目标是**绕过CPU和操作系统**，直接通过网卡访问远程内存。为实现这一目标，必须满足：

10. **地址转换**：网卡需自主完成虚拟地址到物理地址的转换（传统上由CPU的MMU完成）。
11. **权限控制**：网卡需自主验证访问权限（传统上由操作系统内核管理）。 MTT和MPT正是网卡实现这两大功能的核心机制。

### MTT/MPT在RDMA中的工作流程

**RNIC 需要通过 rkey 找到相应的 MPT、然后基于 MPT 的信息再找到相应的 MTT，进而完成 VA 到 PA 的转换**。

![](attachments/Pasted%20image%2020250323231610.png)

1. **内存注册**：
    - 应用程序调用`ibv_reg_mr()`注册内存，内核驱动为此内存区域生成MTT（地址映射）和MPT（权限）。
2. **数据传输**：
    - 当网卡收到RDMA请求时，先检查本地SRAM中的MTT/MPT缓存：
        - **命中**：直接转换地址并验证权限，执行数据传输。
        - **未命中**：通过PCIe访问主机内存中的MTT/MPT，更新缓存后再执行操作。


### 性能瓶颈：MTT/MPT未命中

- **网卡SRAM缓存限制**： 网卡的SRAM容量有限，只能缓存部分MTT/MPT条目。若请求的内存地址未命中缓存，网卡需通过PCIe总线访问主机内存中的完整MTT/MPT表。
- **PCIe访问的代价**：
    - **延迟**：PCIe访问的延迟通常在数百纳秒到微秒级，远高于网卡SRAM的纳秒级延迟。
    - **带宽竞争**：频繁的PCIe访问会占用带宽，影响数据传输效率。

### 优化方法

#### （1）提高缓存命中率

- **增大内存注册块（Memory Region）**： 使用更大的连续内存块注册，减少MTT/MPT条目数量，使更多条目可被网卡SRAM缓存。
- **复用已注册的内存区域**： 避免频繁注册/注销内存，尽量复用已有的内存区域。

#### （2）硬件辅助

- **使用支持缓存扩展的网卡**： 部分高端网卡（如NVIDIA ConnectX系列）支持更大的片上缓存或缓存预取算法。
- **启用地址转换缓存（ATC）**： 某些网卡支持ATC（Address Translation Cache），自动缓存热点地址的转换结果。

#### （3）软件优化

- **预取策略**： 在预期访问前，通过软件提示网卡预加载相关MTT/MPT条目到SRAM。
- **减少细粒度访问**： 合并多个小数据操作成批量操作，减少MTT/MPT查询次数。


# None-Pin RDMA
## PINNED RDMA的问题
1，注册MR时PIN住物理内存这使得可注册的内存空间受限于物理内存大小。
2，应用程序必须有锁住内存的权限。
3，注册本身需要在HCA建立地址映射表，而get_user_pages以及填写页表都是耗时操作，这导致**注册MR很慢**。
4，持续的在软硬件之间同步地址翻译表是一个很耗时的操作。这是因为malloc/mmap/stack都会导致页表变化。

## 计算内存区和通信内存区

![](attachments/Pasted%20image%2020260401150250.png)

很多程序并不知道未来的数据会放在哪个VA范围，也不知道未来需要使用的内存会有多大，这就导致它不能提前注册MR，只能在快要用RDMA发送数据时才注册MR，而注册MR很慢又会阻塞计算。
应用也不能将整个进程的虚拟地址空间注册为MR，因为没有这么多的物理内存可以PIN住。

这种矛盾就产生了计算内存区和通信内存区的区分：计算内存区是程序运行时动态扩张的，其大小和范围无法提前预测，而通信内存区是提前注册好的MR，其物理内存是PIN住的。

应用使用RDMA发送数据时要先将数据从计算内存区copy至通信内存区。这是一个很大的开销。

## 方案一：ODP MR（按需分页MR）

参考：[# Understanding On Demand Paging (ODP)](https://enterprise-support.nvidia.com/s/article/understanding-on-demand-paging--odp-x)

ODP（On-Demand Paging）：注册MR时指定`IBV_ACCESS_ON_DEMAND`标识则创建`ODP MR`，其初始地址翻译表里VA对应的物理页并不存在，因此设备首次访问`MR VA`时会产生`IO page fault(IOPF)`，HCA驱动处理此`IOPF`并换入所需物理页，更新HCA里的地址翻译表，则下次设备DMA时不再发生IOPF。
若操作系统决定`swap out` VA对应的物理页，也会由HCA驱动更新地址翻译表将VA对应entry置为`page none-present`。

### 背景
借助ODP技术，应用程序无需确定用户注册的虚拟地址位于物理页的实际位置。
在RDMA杂谈[对MR的介绍](https://zhuanlan.zhihu.com/p/156975042)中我们知道，**传统情况下本地的HCA是需要维护`VA->PA`的映射表的**，如下图所示，且**注册MR时需要通过锁页保证`VA->PA`的映射不会更改**。

而**使用ODP后HCA不再需要维护映射表**，而是**当页面不存在时向操作系统请求新的`VA->PA`的转化**。

### 分类：隐式ODP和显式ODP
ODP（On-Demand Paging）分为 隐式ODP 「 iODP：Implicit ODP」和 显式ODP  「Explicit ODP」。
显示方式是针对对特定缓冲区，而隐式方式则是注册程序的整个虚地址空间：
```bash
access_flags |= IBV_ACCESS_ON_DEMAND;  
  
// 隐式  
ctx->mr = ibv_reg_mr(ctx->pd, NULL, SIZE_MAX, access_flags);  
  
// 显示  
ibv_reg_mr(ctx->pd, ctx->buf, size, access_flags);
```


## DM作为MR（比如：GDR）
DM(device memory): register the allocated device memory as a memory region.

设备内存作为MR，省去了RNIC通过DMA从主机内存中读写数据。
但是涉及到，应用程序从设备内存中拿数据的问题。

# UMR

## 介绍
**User-mode memory registration (UMR)**： 是 Mellanox/NVIDIA RDMA 网卡（ConnectX-4 及以上）提供的一种高级内存注册机制，允许在**不重新注册物理内存**的前提下，**动态地重新定义一个 MR 的内存布局**。

## 核心数据结构

```bash
┌─────────────────────────────────────────────┐
│            UMR (klm_mkey / indirect MR)      │
│  lkey/rkey: 0xABCD                           │
│  virtual_addr: 0x1000_0000 (虚拟起始地址)    │
│  length: len(Body1) + len(Body2) + len(Body3) │
│                                               │
│  KLM List (Key-Len-Mkey entries):            │
│  ┌──────────────────────────────────────┐    │
│  │ entry[0]: mkey=MR1, offset=header1_sz, len=body1_sz │
│  │ entry[1]: mkey=MR2, offset=header2_sz, len=body2_sz │
│  │ entry[2]: mkey=MR3, offset=header3_sz, len=body3_sz │
│  └──────────────────────────────────────┘    │
└─────────────────────────────────────────────┘


```

UMR 通过 KLM（Key-Length-Mkey） 或 MTT（Memory Translation Table） 条目描述底层物理内存的映射，每个条目指向已有 MR 的某个偏移区域。

## 特性
### 将多块非连续的MR拼接成一个VA连续的MR
**VA连续**：==对于应用程序来说，VA连续就可以（不关注底层是否PA连续）==，这样应用可以一次性处理一大块数据（比如：存储中的大块压缩），就不需要拷贝来凑成一大块。


![](attachments/Pasted%20image%2020251230154011.png)

如上图所示，我们之前创建了3个常规得MR：MR1(green), MR2(purple), MR3(red)。
现在我们想**从这三个MR中各抽取一部分拼接起来形成一个新的逻辑上连续的MR**：
第一块是MR1(v0-v1)部分，第二块是MR2(v2-v3)部分，第三块是MR3(v4-v5)部分。

这个新的MR有一个新的base VA地址，长度是3个小块的长度之和。这样虽然内部是不连续的，但在外部访问者（基于CPU的应用程序）看来这个MR是连续的。


### 将一个MR内有规律非连续的块拼接成一个VA连续的MR

![](attachments/Pasted%20image%2020251230154220.png)

即：**将一个矩阵的列（VA地址不连续，物理地址不连续）给提出来，组成一个VA地址连续的MR**。

如上图所示，当我们做一个矩阵的转置时需要把一列的元素拼成新的行，这个行就成了新的连续的MR。
老矩阵的列元素一般可以用`<基地址(base address), 元素间距(stride)，元素长度(block size)，元素数量(repeat count)>`来描述。

### 将多个MR拼接成新的相互交织的VA连续MR

![](attachments/Pasted%20image%2020251230154339.png)
如上图所示，2个老矩阵的列相互交织形成新的列，这是一个新的VA连续的MR，有它自己的新的base address和length。


## 应用场景
### 场景一：减少MR的数量（进程中多个MR时，减少lkey以及rkey的查找）
如果每 NUMA 一个巨型 MR，未来若支持用户自定义内存（kucl_extmem_register，比如为每个GPU显存存在一个MR，单机存在多个GPU），那么进程中就存在多个MR，每个RDMA 读写需要设置地址的 lkey 或者 rkey ，都需要查找 lkey 和 rkey。
通过 UMR（Indirect MR ）将多个物理不连续的 MR 聚合成一个虚拟连续的 MR，这样就只有一个 lkey 和 rkey。

注：为了防止跨NUMA通信等，实际上需要设置线程/CPU/内存的亲和性。

### 场景二：gather多个MR的部分内存为一块逻辑地址连续的内存
#### 具体场景
**场景**：
底层基于libibverbs rdma，client的多个连接，分别和不同的server端进行RPC通信（RPC请求和响应）。
最终希望将多个server的RPC响应的数据体，聚合到一块逻辑地址连续的内存上，整个过程零拷贝。 
但是有一个问题，每个server响应的数据的格式：rpc响应头+响应消息体；
最终期望的是多个响应消息体放在一大块逻辑地址连续的内存空间中（业务就可以操作这个逻辑地址连续的内存空间，比如数据压缩等，避免拷贝）。 
如何实现呢？


**问题建模**

![](attachments/problem_model.svg)

```bash
Server A 响应:  [RPC Header A (32B)] [Message Body A (N bytes)]
Server B 响应:  [RPC Header B (32B)] [Message Body B (M bytes)]
Server C 响应:  [RPC Header C (32B)] [Message Body C (K bytes)]

目标内存:       [Body A][Body B][Body C]  ← 逻辑连续，零拷贝
```

#### 解决一：RDMA Write with Immediate + 分段两次写（RPC协议改造，无 UMR 要求）
修改 RPC 协议，让 Server 做两次 RDMA Write：
```bash
Write 1: 写rpc Header → Client 的 header_recv_buf（per-connection 小缓冲区）
Write 2 with IMM: 写 rpc Body → Client 的聚合 Buffer + offset
```

**缺点**：
但这增加了 RTT（需要两个 Write）且需要修改 Server 侧逻辑，不推荐。

**优点**：
这个不仅可以做到多个消息体的逻辑地址连续，也可以做到多个消息体的物理地址连续。

#### 解决二：UMR（Indirect MR）

**UMR（User-Mode Memory Registration / Indirect MKey）** 是 mlx5 硬件的一项能力，允许你在用户态创建一个"虚拟 MKey"，这个 MKey 的虚拟地址空间映射到若干段不连续的物理内存区域。对远端 server 来说 以及本地的应用程序来说，它看到的是一块连续的目标地址。

##### 方案一：RPC header 长度固定：

![](attachments/umr_concept.svg)

server 看到的是一块连续的 RDMA 目标地址（由 UMR rkey 描述），server 自己不需要做任何特殊处理；硬件在 DMA 时根据 UMR 散列表，自动把 header 字节投放到 scratch 区，把 body 字节投放到聚合 buf 的正确 slot。

![](attachments/umr_full_flow.svg)

从多个server获取数据的流程：
![](attachments/scheme1_memory_layout.svg)

##### 方案二：PRC header长度不固定：

方案二有一个关键细节：动态重配的 UMR 不是用来"再次接收"数据，而是用来把已经落在 staging buf 里的 body 部分**重新映射成一个虚拟连续的视图**，供消费者通过 rkey 访问，全程无 CPU 拷贝。

流程：
server 先写入 staging，client 读 header 拿到偏移，再动态重配 UMR 暴露虚拟连续的 body 视图。

![](attachments/scheme2_flow.svg)


##### 对比

![](attachments/two_scheme_overview.svg)

实际工程中，方案二有一个更实用的变体：**header 有上界但不固定**。此时用方案一的架构，但把 `hdr_size` 改为 `max_hdr_size`（对齐到 cache line，如 256 字节），server 填充不足的部分用 0 补齐，UMR 散列点固定在 `max_hdr_size` 偏移处。


#### 解决三：RDMA Send/Recv 操作 + Scatter Receive（分散接收）
当接收方 post 一个带有多个 SGE 的 recv WQE 时，硬件会按顺序将入站数据分散填充到各个 SGE 所指向的内存：
但是，这个**只适用于 RDMA Send/Recv ，RDMA Write 由于绕过 RQ，直接写入到指定地址，是不可行的**。

```bash
Server 发送:  [  RPC Header (32B)  |    Message Body (N bytes)   ]
                    ↓ 硬件自动分散
Client WQE:   SGE[0] → header_buf      (32B)
              SGE[1] → agg_buf+offset  (N bytes)
```



# 参考
```bash
# RDMA(7)内存：探索内存管理的艺术，在数据的海洋中航行
https://mp.weixin.qq.com/s/bQy9BQyeLYj12kjZqIeObg

# 在Rust中管理RDMA内存
https://mp.weixin.qq.com/s/xYAHl3eN-vdez5hojiSuLw

# RDMA 高级
https://zhuanlan.zhihu.com/p/567720023

# Notes about RDMA UMR(User-Mode Memory Registration)
https://liujunming.top/2024/10/13/Notes-about-RDMA-User-Mode-Memory-Registration-UMR/
```