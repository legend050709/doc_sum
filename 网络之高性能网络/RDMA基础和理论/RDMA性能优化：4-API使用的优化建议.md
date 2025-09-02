```table-of-contents
```

# 优化建议
## 通用建议
## 提升带宽的建议
## 减少时延的建议
## 减少内存使用的建议
## 减少CPU消耗的建议


# 性能优化工具

## libibprof
`libibverbs`唯一的性能分析工具是`libibprof`，它是`Mellanox HPC` 软件工具包的一部分.
参考：[mellanox-hpc/libibprof](https://github.com/mellanox-hpc/libibprof)

### 介绍
**`libibprof`**（InfiniBand Profiler Library）是由 **Mellanox（现为 NVIDIA Networking）** 提供的一个 **轻量级、可插拔的性能分析库**，用于对基于 RDMA 的应用程序进行性能分析。它通过拦截底层 API 调用来帮助开发者、调试人员或运维人员了解和优化 RDMA 应用的性能瓶颈。
可用于 **收集和分析使用 Mellanox 驱动的 RDMA 应用程序** 中的一些关键指标，如：
- 应用在调用 RDMA verbs（例如 `ibv_post_send`, `ibv_poll_cq` 等）时的行为
- API 调用的频率与延迟
- 时间戳、调用栈信息（可选）
- 应用中使用的 Mellanox 设备（HCA）性能数据

它主要适用于使用 Mellanox OFED 驱动、`libibverbs` 接口以及 InfiniBand、RoCE 网络的 RDMA 程序。


#### 使用范例

![](attachments/Pasted%20image%2020250716192009.png)

![](attachments/Pasted%20image%2020250717161125.png)

### 主要作用

|功能点|描述|
|---|---|
|🎯 API 监控|拦截 RDMA verbs（如 `ibv_post_send`, `ibv_poll_cq`）的调用信息|
|⏱️ 性能分析|统计每个调用的执行时间、频率、延迟分布等|
|🧩 模块化|支持多种分析模块：verbs、memcpy、malloc、pthread 等|
|📈 输出分析报告|自动或手动输出结构化的性能报告（可输出到文件、终端）|
|🧠 调用栈追踪（可选）|可记录调用栈信息（需启用对应配置）用于分析复杂调用路径|
|📦 零侵入式分析|无需修改原程序，仅通过 `LD_PRELOAD` 实现拦截分析|


### 使用场景
==使用了`libibverbs`库中`RDMA API`接口的的程序的性能分析与调优==：

- **HPC 和 AI 应用优化**：如 OpenMPI、NCCL 中 RDMA 通信性能瓶颈分析
- **存储系统调优**：分析 Ceph、Lustre 等使用 RDMA 的存储组件性能
- **微服务或 RPC 框架分析**：如基于 RDMA 实现的 gRPC 调用性能监控
- **RoCE 网络问题定位**：识别高延迟调用、丢包或网络拥塞产生的位置


### 工作机制
**（1）LD_PRELOAD 机制拦截函数调用**
- 通过在运行程序时使用 `LD_PRELOAD=libibprof.so` 来拦截目标 API（如 `ibv_post_send`）
- 插桩并记录调用统计信息

**（2）采样数据**
- 默认情况下记录调用次数、平均耗时
- 可以通过环境变量配置采样频率、开启调用栈追踪等
        
**（3）输出**
- 会在程序退出时输出统计数据（到 stderr 或文件）
- 或者通过 API 在运行中获取数据


### 使用方法

#### 安装
```bash
$ git clone https://github.com/mellanox-hpc/libibprof.git
# cd libibprof
$ ./autogen.sh
$ ./configure --prefix=<path to install>
比如：
./configure --prefix=../libibprof_prefix
$ make -j20
$ make install
```

查看编译输出：
```bash
$ tree ../libibprof_prefix
../libibprof_prefix
├── include
│   └── ibprof_api.h
├── lib
│   ├── libibprof.a
│   ├── libibprof.la
│   ├── libibprof.so -> libibprof.so.0.0.0
│   ├── libibprof.so.0 -> libibprof.so.0.0.0
│   ├── libibprof.so.0.0.0
│   └── pkgconfig
│       └── libibprof.pc
└── README
```

#### 使用
![](attachments/Pasted%20image%2020250716192643.png)

#### API调用的次数以及时延测试
可以查看到最大时延，最小时延，平均时延；基于最大时延，最小时延，可知是否存在抖动，是否存在长尾。

![](attachments/Pasted%20image%2020250716195346.png)

#### 故障注入

#### 代码段的运行时间测试
如上，编译输出的产出中`libprof_api.h`,提供了接口`ibprof_interval_start`和`ibprof_interval_end`，测量一段代码的运行时长，并输出，进行代码段的实验分析。

![](attachments/Pasted%20image%2020250716195153.png)


### 问题
#### output_file中没有输出
可能是编译`libibprof`时，`libibverbs` 版本不支持。
如下所示：在 `./configure --prefix=xxx` 报错如下： 
![](attachments/Pasted%20image%2020250717165011.png)

#### 主要问题
`libibprof` 是 Mellanox（NVIDIA）开发的用于分析 RDMA 接口调用开销的工具，但它的维护确实已经停滞多年。==随着 `libibverbs` / `rdma-core` 的更新，`libibprof` 已经出现兼容性和功能上的不足， 比如：`libibprof`  无法在集成到ofed中的 `libibverbs`进行分析，只能对原生的很老版本的`libibverbs`「比如: libibverbs 1.2及其之前的」进行分析，而原生的老版本的`libibverbs`版本目前基本不在使用了。 ==。


注：可以借鉴学习`libibprof`的实现，为日后自身实现分析`libibverbs`的工具打下基础。

### 小结

|项目|内容|
|---|---|
|名称|`libibprof`|
|类型|动态性能分析工具|
|适用平台|RDMA、InfiniBand、RoCE，基于 Mellanox 驱动|
|使用方式|动态链接拦截（LD_PRELOAD）|
|优势|零代码侵入，模块化，可定制，支持详细分析|
|局限|仅适用于 Mellanox 环境，可能影响极端高性能测试场景|


# 参考
```bash
# Tips and tricks to optimize your RDMA code
https://www.rdmamojo.com/2013/06/08/tips-and-tricks-to-optimize-your-rdma-code/
```