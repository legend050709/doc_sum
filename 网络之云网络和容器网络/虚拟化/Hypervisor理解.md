```table-of-contents
```
# 什么是 Hypervisor
Hypervisor（虚拟机监控器，VMM，Virtual Machine Monitor）是**运行在物理硬件上的一层软件**，主要作用是：
- **抽象并虚拟化硬件资源**（CPU、内存、硬盘、网络等），
- **允许在同一台物理机上运行多个虚拟机（VM, Virtual Machine）**，
- **隔离和调度**不同虚拟机之间的资源。

换句话说，hypervisor 就是“制造虚拟机的底层工具”。

## Hypervisor 的作用
- **资源隔离**：每个 VM 有独立的 CPU、内存、磁盘、网卡等“虚拟硬件”。
- **多租户支持**：在云计算里（比如 AWS EC2、阿里云 ECS），hypervisor 让一台物理机能安全地跑多个租户的 VM。
- **高可用与迁移**：支持 VM 快照、迁移（如 vMotion），运维方便。

# Hypervisor 的类型
通常分为两类：
**Type 1（裸机型）**  
直接运行在物理硬件之上，不依赖宿主操作系统。  
例子：
- VMware ESXi
- Microsoft Hyper-V (裸机模式)
- KVM（严格来说 Linux 内核+KVM模块可以看作 Type 1）
- Xen

**Type 2（托管型）**  
运行在宿主操作系统之上，就像一个应用程序。  
例子：
- VMware Workstation / Fusion
- Oracle VirtualBox
- Parallels Desktop

# Hypervisor和容器/虚拟机的关系
## 和虚拟机（VM）的关系
- **虚拟机是 hypervisor 创建出来的产物**。
- 虚拟机运行的是完整的操作系统（Guest OS），和物理机上装的操作系统一样，只是跑在虚拟硬件上。
所以关系是：  
物理硬件 → hypervisor → 虚拟机

## 和容器（Container）的关系
容器和虚拟机经常被比较，但它们并不一样：
**虚拟机**
- 需要 hypervisor 来虚拟化硬件
	
- 每个 VM 都有自己的 **完整操作系统内核**（Guest OS）
	
- 开销大，启动慢，但隔离性强（接近物理机）
        
**容器**
- 基于宿主机的操作系统内核（Linux namespaces + cgroups）
- 没有额外的 Guest OS（共享宿主机内核）
- 启动快，资源开销小，但隔离性不如 VM
        

简单类比：
- VM：相当于在一台电脑里装了多台虚拟电脑
- Container：相当于在同一个操作系统里开了多个独立的“隔离环境”


**容器 是操作系统层面的隔离技术，不依赖 Hypervisor**（但容器可以运行在虚拟机里）


# 参考
```bash

```