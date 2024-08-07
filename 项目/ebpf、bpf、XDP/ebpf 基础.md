```table-of-contents
```
# 概述
# bpf架构
## 指令集
## helper辅助函数
## ebpf map
### ebpf map类型
#### BPF_MAP_TYPE_XSKMAP
#### BPF_MAP_TYPE_PERCPU_ARRAY

## object pin钉住对象
## 尾调用tail calls
## bpf to bpf calls
## 即时编译器JIT
## 加固hardening
## offload
## BTF


# 工具链
## 开发环境
## LLVM
## iproutes包

### 使用范例
#### xdp的使用

![](attachments/Pasted%20image%2020240719162129.png)

## bpftool
## bpf sysctls




# 参考
```bash
# [译] Cilium：BPF 和 XDP 参考指南（2021）
https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/

# 极客时间，运行原理：eBPF 是一个新的虚拟机吗？
https://time.geekbang.org/column/article/481889

# [译] [论文] XDP (eXpress Data Path)：在操作系统内核中实现快速、可编程包处理（ACM，2018）
https://arthurchiao.art/blog/xdp-paper-acm-2018-zh/

# [译] Cilium：BPF 和 XDP 参考指南（2021）
https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/

# [译] 基于 BPF/XDP 实现 K8s Service 负载均衡 (LPC, 2020)
https://arthurchiao.art/blog/cilium-k8s-service-lb-zh/


# [EBPF学习计划](https://davidlovezoe.club/wordpress/archives/862)
https://davidlovezoe.club/wordpress/archives/862

```