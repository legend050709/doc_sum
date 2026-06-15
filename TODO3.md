（一）NCCL:

  

```bash

  

NCCL的内容理解：

  

(1) GPU节点感知，拓扑选择（ring/tree/collnet allreduce等）

  

NVLink、PCIe、RDMA：NCCL 到底让 GPU 数据走哪条路

https://mp.weixin.qq.com/s/ki2wvWLhlHTy-suFioTbMw

  

GPU拓扑检测（GPU Topology Detection）

https://mp.weixin.qq.com/s/hVWsWDUtBSrvAB6UahgqCg

  

---

  

(2) 传输选择(nvlink/RDMA/GDR/Pcie P2P)

  
  

----

  

(3) 协议选择(带宽or 时延： simple/LL/LL128 等)

  
  

------

  

(4) 并行通道: channel(比如多个ring并行)

不同 Channel 各跑各的 chunk，每条 Channel 内部执行流程跟单条 Ring 一样——先 Reduce-Scatter（RS，求和轮），再 All-Gather（AG，同步轮）

  

------

  

(5) 实现机制：计算和传输overlap （GPU Kernel vs Proxy Thread）

选好了拓扑、确定了 Channel 数，接下来：谁负责把数据从内存搬到网卡？

NCCL 内部每个 Channel 有两条并行路径——GPU Kernel 负责计算，Proxy Thread 负责网络传输

  
  

--------------------------------

  

(1) AllReduce

AllReduce算法详解（AllReduce Algorithm） // 文章系列 ++++++++++++

https://mp.weixin.qq.com/s/bcAsPsD0L7Xkn9xjsmAvkg

  
  
  

(2) 数据并行(DP) / 张量并行(TP) / 流水线并行（PP）下的 AllReduce：

  
  
  
  

```

  
  
  

(二) CUDA

  

```bash

CUDA的理解：

  

（1）CUDA的内存域(MD: memory domain)、

  

CPU内存、GPU内存、Manged memory

  
  

（2）CUDA 传输层：IB/Roce/RDMA/TCP/NVLINK/SHM

  

```

  
  
  

参考资料：

  

```bash

  

(1) PyTorch-->NCCL--->RDMA网卡：

  

从PyTorch到RDMA网卡：自顶向下（一）训练为什么卡？

https://mp.weixin.qq.com/s/sI6Bo7uG63jEWX2R7hel8w

  
  

从PyTorch到RDMA网卡：自顶向下（二）NCCL怎么组织通信

https://mp.weixin.qq.com/s/1rgx7rnYEiW2xmSu6P8EOg

  

从PyTorch到RDMA网卡：自顶向下（三）Proxy 如何把 GPU 数据交给网卡

https://mp.weixin.qq.com/s/xMKR8PqcevKPpTI_l8BVgA

  
  

(2)NCCL-->RNIC:

  

多轨不是定轨：NCCL 到底是怎么选择网卡的？

https://mp.weixin.qq.com/s/tp01VWUoUOHD9G09ttSqAA

  
  

```

  
  
  

（三）GPU

  

```bash

(0) PCIe P2P / GPU Direct P2P

  

(1) GDR

GPUDirect RDMA（GPUDirect RDMA）

https://mp.weixin.qq.com/s/jgK1iQK9F7N45H92J7JALw

  

(2) GDA

GPUDirect异步操作（GPUDirect Async）

https://mp.weixin.qq.com/s/uWEyUkxmTt8WRPiPE-_E-Q

  

(3)

```

  
  
  

（四）RDMA



RNIC上的WQE的cache：WQE的生命周期；

  
  
  

（五）debug工具

  

```bash

（1）uftrace + Perfetto UI 函数调用时延分析方案

这是一种用户态函数级 tracing + 可视化分析的组合方案。不需要改代码，只需要编译时加 -pg（或者用 -finstrument-functions），

就可以把每个函数的进入/退出时间戳全部录下来，再喂给浏览器里的火焰图/时序图查看。

  

uftrace：一个开源的 用户态函数 tracer（GitHub: namhyung/uftrace），定位类似内核态的 ftrace，但是抓的是用户态进程的每一次函数调用。

Perfetto UI：https://ui.perfetto.dev/ 是 Google 开源的 trace 可视化平台（前身就是 Chrome 的 chrome://tracing / about:tracing）。

```