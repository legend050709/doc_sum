```table-of-contents
```
# 背景

## 普通双口 NIC 的限制
一般而言，单卡双口的网卡只能在一个NUMA上，那么该网卡进行收发数据包，进行内存访问的时候，可能就涉及到跨NUMA访问内存。


- 单芯片双口卡 → 两个口共享一个 PCIe function，只能挂在一个 NUMA 节点。    
- 即使是多芯片双口卡，如果没有板载 PCIe switch 来分流到不同 root complex，那么也还是归属于单个 NUMA 节点。

```bash
有些高端双口卡，其实是 **两块 NIC 芯片 + 一个板载 PCIe switch**。
这种情况下，每个芯片可以挂到不同的 PCIe root complex（即不同 NUMA 节点）。
在 `lspci` 下会看到两条独立的 NIC 设备号，每个口对应一个 NUMA 节点。

常见于 Mellanox ConnectX 系列的部分高端双口卡，或者 OEM 服务器定制 NIC。
```

# Multihost NIC
**Multihost NIC** 的设计就是：
- 一块网卡上有 **多个 PCIe 接口**，分别连到不同主机 / NUMA 节点。
- 每个主机看到的都是一个独立的 PCIe function，相当于“独占”一部分网卡资源。
- 在 NUMA 系统里，这就可以让 **端口 A 属于 NUMA0，端口 B 属于 NUMA1**。

举个例子：
 Mellanox **ConnectX-5/6 Multihost** 支持 2/4-host 模式。
 一块 200Gbps 网卡，可以分成 2 个 100G，分别通过 PCIe x16 接口连到 CPU0 和 CPU1。
 这样 CPU0 的 NUMA 下看到 eth0，CPU1 的 NUMA 下看到 eth1。

# 参考
```c

```