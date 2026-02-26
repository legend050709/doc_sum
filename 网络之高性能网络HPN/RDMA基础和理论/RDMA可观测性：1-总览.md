```table-of-contents
```
# 发展轨迹
## 第一阶段：传统网络监控（Device-Centric）

### 核心特点
- SNMP
- NetFlow/sFlow
- 设备级 counters
- 端口丢包、带宽利用率

### 问题
- 粒度粗（端口级）
- 延迟大（分钟级）
- 无法定位微突发
- 无法感知应用级异常
- RDMA 场景基本失效（RoCE 无 TCP 可见性）

## 第二阶段：Telemetry + 主动探测

##  第三阶段：端网协同

### 更精细
#### 端侧：基于eBPF 驱动的内核级可观测性

网络监控一定会重度使用：
```bash
- eBPF tracepoint
- kprobe
- XDP
- tc-bpf
- bpf ringbuffer
```

因为：
```bash
- 内核态可见性
- 无侵入
- 低开销
- 可以实时上传异常
```

比如：无需修改内核即可动态注入探针，监控：
- TCP连接状态（`tcp_connect`, `tcp_retransmit`）
- Socket缓冲区水位
- 应用层协议解析（HTTP/GRPC）



## 后续发展
### 智能化：实时跨层关联分析
其实，可观测性，从宏观上看，应该是一个系统工程，涉及到很多的产品。用户从发起请求到获取响应，端到端的整个链路涉及到哪些节点和设备等。有这样的一个业务画像，出现问题时，需要先找到问题节点，再在问题节点上找到具体的原因。

比如：把下面的监控，现象统一到一个因果模型里。这是因果链，而不是简单报警。
```bash
- NIC metrics
- switch telemetry    
- RDMA QP状态
- CPU调度
- 应用RT：（应用 RT = 用户请求从发出到完成的端到端耗时）

比如：

RT升高 （Response Time（响应时间））
  ↓
某rack queue depth上升
  ↓
PFC pause增加
  ↓
某节点 softirq backlog上升
  ↓
QP retry增加
```


### 自治系统/自愈系统
未来方向不是“监控系统”，而是 网络自反馈控制系统（Closed-loop Network OS）；监控系统不仅是“眼睛”，更是“手”。网络从 `passive` 变成 `self-aware + self-optimizing`; 包括：
```bash
- 自动调节 ECN
- 自动限速
- 自动 reroute
- 自动 QP 迁移
- 自动负载均衡
```

比如：
- 检测到某链路 ECN 标记激增 → 自动调整 RDMA DCQCN 参数
- 发现微服务 A 调用 B 延迟突增 → 触发 Service Mesh 限流或切换备用实例

# RDMA 场景下的未来演进
## 当前 RDMA 可观测性的核心痛点
RDMA（Remote Direct Memory Access）技术在高性能计算（HPC）、大规模 AI 模型训练以及分布式存储中已经成为核心组件。然而，由于 RDMA 技术的特性是 **绕过内核（Kernel Bypass）** 和 **零拷贝（Zero Copy）** ，这使得传统的基于操作系统内核的网络监控工具（如 tcpdump, netstat 等）在 RDMA 场景下几乎失效，导致网络变成了“黑盒”。

## 数据采集层面：从“采样”走向“全量精确遥测”
传统的 SNMP 或 sFlow 采样对于 RDMA 这种对延迟极度敏感（微秒级甚至纳秒级）的协议来说，颗粒度太粗。

- **带内网络遥测（INT/IFA）的普及：**  
    未来的交换机和网卡将更广泛地支持 In-band Network Telemetry (INT) 或 IFA (In-band Flow Analyzer)。这允许网络设备在每个 RDMA 数据包中插入元数据（如入队时间、出队时间、队列深度、具体转发路径）。这意味着运维人员可以追踪每一个数据包在整个网络中的精确行为，彻底打开网络黑盒。
- **芯片级遥测技术：**  
    交换机芯片和 SmartNIC/DPU 芯片将内置更强大的遥测逻辑。不仅关注丢包，还能监控 **ECN（拥塞通知）标记率**、**PFC（优先级流控）帧的触发频率**、**Head-of-Line (HOL) 阻塞**等微观拥塞指标，并以流式数据（Streaming Telemetry）毫秒级推送到分析通过平台。

## 观测视角层面：端网协同的全链路视图
目前 RDMA 的监控往往是割裂的：网卡看网卡的，交换机看交换机的。未来的演进将强调 **“端网协同”** 。

- **SmartNIC/DPU 成为观测重心：**  
    由于 RDMA 协议栈卸载到了网卡上，网卡（NIC）实际上是 RDMA 通信的“大脑”。未来的可观测性将通过 DPU/SmartNIC 深度暴露 RDMA 动词（Verbs）的状态、QP（Queue Pair）的状态机转换、以及重传（Retransmission）和乱序（Out-of-Order）的具体原因。
- **全路径拓扑映射：**  
    系统将能够自动绘制特定 RDMA Flow（流）的端到端物理路径。当发生性能抖动时，可以直接定位到是哪一台服务器的网卡配置错误，还是中间哪一台交换机的 Buffer 被打满。

## 分析诊断层面：从“人工排查”走向“AIOps 智能诊断”
RDMA 网络产生的数据量极大，且问题往往是瞬态的（Micro-burst 微突发）。依靠人工盯着 Dashboard 已不现实。

- **PFC 风暴与死锁的自动检测与自愈：**  
    RoCEv2 网络中最头疼的问题是 PFC 风暴导致的拥塞扩散。未来的可观测性平台将结合 AI 算法，实时识别 PFC 触发模式，预测死锁风险，并联动控制器自动调整水线（Watermark）或隔离异常节点，实现故障自愈。
- **Root Cause Analysis (RCA) 自动化：**  
    当 AI 训练任务变慢时，系统能直接给出结论：例如“GPU 0 到 GPU 8 的通信延迟增加了 200us，原因是交换机 SW-3 的 端口 4 发生了拥塞丢包（ECN 未及时响应）”，而不是丢给运维一堆图表。

##  技术栈融合：eBPF 与 RDMA 的结合

虽然 RDMA 绕过内核，但控制面（Control Plane）和应用层交互依然在 OS 层面。

- **eBPF 增强观测：**  
    使用 eBPF（Extended Berkeley Packet Filter）技术挂载到 RDMA 用户态驱动（User-space Driver）或应用层库（如 NCCL, MPI）的调用点。这样可以在不修改应用代码的情况下，关联应用层的行为（如“正在进行 AllReduce 操作”）和底层的网络指标，实现业务视角的网络观测。


# 参考
```bash

```