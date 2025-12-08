```table-of-contents
```
# 传统SRIOV的缺陷
**VF 灵活性不足**：
即VF不支持动态分配和按需创建，使用过SRIOV的都清楚VF初始化必须echo一个具体sriov_num的设备个数，如果开始初始化的是2个，后面你想用3个，那就必须把当前两个释放掉，重启分配初始化为3个，再重新创建新的三个VF。这意味着 VF 数只能在宿主机启动阶段配置。预留过多 VF 也不可行，因为每个 VF 占用63 条虚拟队列、如果每个队列desc深度设置为存储 5000个 MTU 尺寸的数据包，就需要总计消耗 2.4 GB 内存。（PS：个人觉得，这里的例子有点极端了，实际上安全容器很少需要这么多队列，即使需要，队列深度也不需要这么大，不过内存开销确实是一个问题）

**VFIO 需要ping住所有runD的内存（GPA）**：
RunD 运行容器（也包括虚拟机）使用设备直通的时候，一般需要将容器的内存都pin住，导致RunD启动时间比较长。
原因是：设备直通通过DMA代替传统半虚拟化的memcpy方式，为了保证虚拟机间的内存隔离，虚拟机hypervisors必须为直通设备设置合适的I/O页面表（IOPT，IO page tables），以限制其直接内存访问（DMA）。而当前的设备大多是不支持page fault的处理，即不支持IOPF，所以一般hypervisors就会把容器/虚拟机整个内存pin住。因为对于hypervisor，整个虚拟机的内存都是潜在发生DMA的内存区域。

# 参考
```bash
# Stellar: 新一代 RDMA 网络
https://mp.weixin.qq.com/s/XTN2m59zNcCWkNdpE17l2Q

# FreeFlow: 基于软件的虚拟RDMA容器云网络（上）
https://www.infoq.cn/article/2ljyuw*4qvgce94dqb64
https://www.usenix.org/system/files/nsdi19spring_kim_prepub.pdf

# RDMA NIC虚拟化
https://zhuanlan.zhihu.com/p/651023182

```