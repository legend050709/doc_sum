```table-of-contents
```

# CPU和GPU的内存互访
![](attachments/Pasted%20image%2020260423173805.png)

## 三种机制

### ① PCIe BAR 映射（CPU 直接 load/store）

GPU 在 PCIe 配置空间里注册若干 **BAR（Base Address Register）**，操作系统将其映射到 CPU 的物理地址空间。CPU 用普通 `mov` 指令访问，但每次读写都发起一次 PCIe transaction，走的是 **Non-Posted Write / Completion-based Read**，必须等 PCIe round-trip。

关键细节：这段内存默认是 **WC（Write Combining）** 属性，CPU cache 不缓存，写操作在硬件合并后才发到总线。历史上 BAR 窗口 ≤ 256 MB（32-bit BAR 限制）；现代系统通过 **Resizable BAR / Above-4G Decoding** 可以把整块 GPU VRAM 暴露出来（但随机访问依然很慢）。

### ② DMA 引擎传输（cudaMemcpy 底层）

GPU 内部有专用的 **Copy Engine**，独立于 SM 运行。`cudaMemcpy` 的本质是：

1. CPU 准备好锁页内存（`cudaMallocHost` 或 `mlock`），让 kernel 固定物理页，使其不被换出
2. 向 Copy Engine 写入描述符（src PA、dst PA、length、方向）
3. Copy Engine 在 PCIe 上连续发 DMA Read/Write，完全不经过 CPU
4. 完成后通过 MSI-X 中断或 polling doorbell 通知 Host

Host 内存**必须锁页**的原因：DMA 引擎使用物理地址，如果 OS 在传输过程中换出页面，物理地址失效会造成数据错乱。

### ③ 统一内存（cudaMallocManaged）

这是由 **GPU MMU + IOMMU + CUDA driver + OS** 联合实现的按需页迁移机制：

- CPU 访问 GPU 页 → CPU MMU 产生缺页异常 → CUDA driver 介入 → 调用 `cuMemMigrateRanges` 把整页从 VRAM 搬到 Host DRAM → 更新 CPU 页表，放行访问
- GPU 访问 Host 页 → GPU MMU 产生缺页 → 反向迁移

在支持 **ATS（Address Translation Services）+ NVLink** 的系统（如 Grace-Hopper）上，GPU 可以通过 NVLink 直接 remote access CPU 内存，不触发迁移，延迟更低。

### 小结

|机制|CPU 如何"动手"|数据走的路|适合场景|
|---|---|---|---|
|BAR|CPU 亲自 LD/ST|CPU → PCIe → GPU VRAM|寄存器级小量访问|
|DMA|CPU 委托 DMA engine|DMA engine 自主走 PCIe|大块数据批量搬运|
|统一内存|CPU 正常访问，缺页触发迁移|driver 整页搬运，透明|编程简单，容忍冷启动开销|



## GPU的Copy Engine

**Copy Engine(CE) 就在 GPU 芯片内部**，是 GPU die 上一个独立的硬件功能单元，和 SM（流式多处理器）、内存控制器等并列存在。

Copy Engine它的身份：PCIe Bus Master。关键点在于它的角色不是"被动接收者"，而是 **PCIe 总线的主设备（Bus Master）**。流程是这样的：

1. CPU 执行 `cudaMemcpy`，本质是往 GPU 的 **Command Queue** 里写一条描述符（src物理地址、dst物理地址、长度、方向），这一步 CPU 确实参与了，但只是"下单"；
2. Copy Engine 从 Command Queue 取出描述符，**自己**在 PCIe 总线上发起 `MRd`（Memory Read）事务，主动从 Host DRAM 拉数据；
3. 数据经过 PCIe Interface，直接落入 GPU VRAM；
4. CPU 全程不碰数据，只在最后收一个完成信号。

# 参考
```bash
# GPU驱动软件栈介绍
https://zhuanlan.zhihu.com/p/1974237076372349203
```