```table-of-contents
```
#  Host 上 RDMA 网络协议栈的可观测性
## 背景
为了更好的调试故障，需要有一个细粒度的遥测工具来捕获端到端路径中每个组件的行为。虽然许多现有工具可以用于诊断交换机和链路故障，但这些工具都不能提供端点 Host 上 RDMA 网络协议栈的良好可观测性。

## 思路
受 TCP 诊断工具的启发，为了诊断网络和主机的性能问题我们开发了 RDMA Estats（Extended Statistics）工具。当 RDMA 应用性能不佳时，RDMA Estats 可以让我们判断出性能瓶颈是在发送方、接收方还是传输网络中。

为了实现这一目标，RDMA Estats 除了收集诸如发送/接收的字节数和 NACK 数等常规计数外，还为每个 RDMA 操作提供细粒度的延迟分解。
当 WQE 穿过传输 pipeline 时，请求方 RNIC 会在一个或多个观测点记录时间戳。
当收到响应（ACK 或读取响应）时，RNIC 会在接收 pipeline 沿线的测量点记录附加时间戳。

## 方法
在 Azure 中实施 RDMA Estats 需要提供以下测量点：
![](attachments/Pasted%20image%2020250319164751.png)

T1：WQE posting：WQE 发布到提交队列时的 Host 处理时间戳。
T5：CQE generation：NIC 中生成 CQE 时的 NIC 时间戳。
T6：CQE polling：软件轮询 CQE 时的 Host 时间戳。

在 Azure 中，NIC 驱动通过上述时间戳上报各种时延统计。
例如，T6 - T1 是 RDMA 消费者看到的操作时延，而 T5 - T1 是 NIC 看到的时延。

## 可观测性展示
用户态 agent 按连接、操作类型和（成功/失败）状态对时延样本进行分组，以便为每个分组创建时延直方图。直方图的默认时延采集间隔为一分钟。每个直方图的分位数和汇总统计都会输入到 Azure 的遥测 pipeline 中。
随着诊断技术的发展，在高时延事件中我们为用户态 agent 添加了收集和上传 NIC 和 QP 状态 dump 的功能。最后，为了以防非特定于 RDMA 事件的服务操作影响连接，我们扩展了用户态 agent 的事件触发的数据收集范围，以包括 NIC 统计和状态 dump。

时延样本的收集增加了 WQE 发布和处理代码的路径开销，此开销主要是为了保持 NIC 和主机时间戳的同步。为了减少该开销，我们开发了一种时钟同步程序，在保持较低的偏差的同时最大限度地降低读取 NIC 时钟寄存器的频率。

# 参考
```bash
# 使用 RDMA 提升微软 Azure 云的存储性能
https://zhuanlan.zhihu.com/p/679493965
```