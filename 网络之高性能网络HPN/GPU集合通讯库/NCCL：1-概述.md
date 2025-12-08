```table-of-contents
```
# 介绍
NVIDIA 的 集合通信库（NCCL, NVIDIA Collective Communications Library） 是一个专为多GPU和分布式计算环境设计的高效通信库。它提供了一系列优化的通信操作，能够在多GPU或多节点之间进行快速的数据交换。NCCL 库基于 NVIDIA 的硬件加速技术，特别是针对 GPU 之间的高速通信进行了优化，并且支持多种通信模式（如点对点、广播、规约等），以提高大规模并行计算的效率。


# 应用场景
常见应用场景
**深度学习训练**：
在分布式训练中，不同的GPU需要共享参数和梯度。NCCL 提供了多种集合通信操作来协助模型参数同步，减少训练时间。

**大规模数据并行计算**：
当训练模型需要使用多个 GPU 时，NCCL 可以用来实现高效的数据分发、同步和合并。

**分布式计算框架**：
NCCL 作为深度学习框架（如 PyTorch、TensorFlow）中的重要组件，提供了底层的通信支持，使得分布式训练更加高效。

**高性能计算（HPC）**：
在需要大规模并行计算的科学模拟、图像处理等领域，NCCL 也能提供加速的集合通信操作。



# 参考
```bash
# NCCL概述和NCCL-Test分析
https://mp.weixin.qq.com/s/3lrLSYe7JhalTcXjpFQ0dA
```