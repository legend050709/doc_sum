```table-of-contents
```
# 简介
## UEC
超以太网联盟 (`UEC: Ultra Ethernet Consortium /kənˈsɔːrtiəm/`)是在 Linux 基金会的牵头下由多家全球头部科技企业联合成立，推动以太网技术在高性能计算（HPC）、人工智能（AI）和云计算等领域发展的行业组织。

![](attachments/Pasted%20image%2020250723202310.png)


## UEC的目标

![](attachments/Pasted%20image%2020250723202425.png)

## UET
UET: 超级以太传输协议

## Tail latency
尾部延迟，（以通信阶段最后一条消息的到达时间为衡量标准）是系统性能的关键指标。


# 背景
## RoCE用于未来AI/HPC网络的局限性

![](attachments/Pasted%20image%2020250723201939.png)

# 超以太网协议栈
## 协议栈概览
▣ 物理层与传统以太网完全兼容，可选支持FEC（前向纠错）统计功能

▣ 链路层 可选支持链路层重传（LLR），并支持包头压缩，为此扩展了LLDP的协商能力

▣ 网络层 依然是IP协议，没有变化

▣ 传输层 是全新的，作为UEC协议栈的核心数据包传输子层（Packet Delivery）和消息语义子层（Message Semantics）。包传输子层实现新一代拥塞控制、灵活的包顺序等功能，消息语义子层支持xCCL和MPI等消息。可选支持安全传输。另外，[在网集合通信]（In Network Collective，INC）也在这一层实现

▣ 软件API层。提供UEC扩展的Libfabrics 2.0


![](attachments/Pasted%20image%2020250723202646.png)


## 物理层

## 链路层

## 传输层(UET, 新一代协议栈的核心)

## 软件层


# UEC（UET）和 Roce的比较




# 参考
```bash
# 【网络】超以太网联盟 UEC|Ultra Ethernet|下一代 “RoCE” 协议--编辑中
https://blog.csdn.net/bandaoyu/article/details/144647522

# 锐捷网络加入超以太网联盟UEC，助力智算网络持续升级
https://zhuanlan.zhihu.com/p/673251084

# UEC2023白皮书：超以太网联盟规范
https://seclee.com/post/202408_uec2023/
```