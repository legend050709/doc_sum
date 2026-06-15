（xxx）杂项：

（1）RDMA

（1.1）RDMA时延性能：

RDMA - inline 内联提高小包性能-降低时延(减少两个 PCIe 往返延迟)
https://zhuanlan.zhihu.com/p/703645611


（1.2）RDMA 的 Tag Matching and Rendezvous
RDMA 的 Tag Matching and Rendezvous Offload 特性：
https://zhuanlan.zhihu.com/p/567720023

软件 tag matching 和 硬件 tag matching：
Tag Matching  和 Rendezvous 的联系以及配合使用；
    Tag Matching 的核心是"命中就直通，未命中就暂存等待"。
    Tag Matching 是基础设施，Rendezvous 是上层协议，两者正交但后者依赖前者。
    当消息走 Rendezvous 路径时，你同样需要 Tag Matching——接收方 NIC 收到 RTS 后，正是通过 Tag Matching 在 TML 里找到对应的用户缓冲区，才能把地址放进 CTS 回给发送方。
    所以 Rendezvous 是"借助 Tag Matching 完成的大消息传输"，不是替代关系。


Rendezvous中使用RDMA Write 和 RDMA Read的区别：


(1.3) RDMA 多路径：

    1》多QP方式：单个fd多个qp
    2》多TOS方式：一个fd一个qp 或 多个 fd 多个qp，但是 qp 的 不同的wr 可以设置 不同的 TOS，交换机可以基于五元组+TOS进行hash选路吗？

```
单 QP (RC)
   │
   │  WR_0: ah_attr.grh.flow_label = F0  ─► Spine ECMP ─► Path 0
   │  WR_1: ah_attr.grh.flow_label = F1  ─► Spine ECMP ─► Path 1
   │  WR_2: ah_attr.grh.flow_label = F2  ─► Spine ECMP ─► Path 2
   ▼
  注意：对 RC，QP 的 AV 在 RTR 阶段就固定了，per-WR 改不了
        ➜ 必须用 mlx5dv_qp 的 raw send + flow_label segment
        ➜ 或者用 UD QP（AH 是 per-WR 的）
```
    3》DCT/DCI: 多个DCI的方式

```
发送端:                       接收端:
┌─────────────────┐          ┌─────────────────┐
│  DCI Pool       │          │  DCT (Target)   │
│  ├── DCI_0  ────┼──path0──►│  (KUCL 进程)    │
│  ├── DCI_1  ────┼──path1──►│  共享 RQ + SRQ  │
│  ├── DCI_2  ────┼──path2──►│                 │
│  └── DCI_3  ────┼──path3──►│                 │
└─────────────────┘          └─────────────────┘

每个 DCI 可以指向不同 dgid / sl / flow_label → 多路径
1 个 DCT 接收多个 DCI 的数据
```



（1.4）SRQ流控：
场景：单连接突发大流量、多租户混部、接收应用慢消费、维护期/抖动

方法：可以考虑在UCX的流控的基础上，给每个QP超额分配信用+ 设置SRQ低水位的水线，一旦低于低水位的水线，就需要发送类似于pause的消息。


-------------------



--------------

（3）libfabric
libfabric的理解：
https://cloud.tencent.com/developer/article/2531002


libfabric / UCX / MPI / NCCL / libibverbs 的 通信的层次和关联：

```
                应用层
────────────────────────────────
PyTorch / TensorFlow / HPC App
────────────────────────────────

MPI             NCCL
(HPC)            (AI Collectives)

────────────────────────────────
UCX         libfabric

(communication middleware/runtime)

────────────────────────────────
libibverbs     CUDA Driver
rdma-core      GPUDirect

────────────────────────────────
mlx5 / efa / cxi / hfi drivers

────────────────────────────────
RNIC / GPU / Switch / NVLink

```

--------------------

（4）UCX
ucx的理解：


（5）其他：

pin住内存 和 mlock的区别和对比；