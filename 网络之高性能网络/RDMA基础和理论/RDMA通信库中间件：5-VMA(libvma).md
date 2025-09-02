```table-of-contents
```
# 介绍
如果能使用我们熟悉的 socket 编程接口进行 RDMA 数据传输多好呀!

Mellanox 的 VMA（Voltaire Messaging Accelerator）就是这么一种用户空间的高性能网络库，旨在加速基于以太网和 InfiniBand 的网络通信。
VMA 主要用于低延迟、高吞吐量的应用场景，特别是在金融服务（如高频交易）、电信和大规模分布式系统中。
它通过绕过内核网络协议栈，直接与网卡硬件交互，从而减少网络延迟并提高应用程序性能。**VMA 支持标准的 BSD 套接字 API**，使现有应用程序能够无缝迁移，无需对代码进行大规模修改。

# 使用
## 安装
查看设备上是否存在该库。
```bash
# find / -type f -name "libvma*"
/root/MLNX_OFED_SRC-5.8-6.0.4.2/SRPMS/libvma-9.7.2-1.src.rpm
/root/MLNX_OFED_LINUX-5.8-6.0.4.2-rhel7.4-x86_64/RPMS/libvma-9.7.2-1.x86_64.rpm
/root/MLNX_OFED_LINUX-5.8-6.0.4.2-rhel7.4-x86_64/RPMS/libvma-devel-9.7.2-1.x86_64.rpm
/root/MLNX_OFED_LINUX-5.8-6.0.4.2-rhel7.4-x86_64/RPMS/libvma-utils-9.7.2-1.x86_64.rpm
```

安装
```bash
yum install -y /root/MLNX_OFED_LINUX-5.8-6.0.4.2-rhel7.4-x86_64/RPMS/libvma-9.7.2-1.x86_64.rpm /root/MLNX_OFED_LINUX-5.8-6.0.4.2-rhel7.4-x86_64/RPMS/libvma-devel-9.7.2-1.x86_64.rpm /root/MLNX_OFED_LINUX-5.8-6.0.4.2-rhel7.4-x86_64/RPMS/libvma-utils-9.7.2-1.x86_64.rpm
```

安装后，再次查看
```bash
# find / -type f -name "libvma*"
/usr/lib64/libvma-debug.so
/usr/lib64/libvma.so.9.7.2
/root/MLNX_OFED_SRC-5.8-6.0.4.2/SRPMS/libvma-9.7.2-1.src.rpm
/root/MLNX_OFED_LINUX-5.8-6.0.4.2-rhel7.4-x86_64/RPMS/libvma-9.7.2-1.x86_64.rpm
/root/MLNX_OFED_LINUX-5.8-6.0.4.2-rhel7.4-x86_64/RPMS/libvma-devel-9.7.2-1.x86_64.rpm
/root/MLNX_OFED_LINUX-5.8-6.0.4.2-rhel7.4-x86_64/RPMS/libvma-utils-9.7.2-1.x86_64.rpm
/etc/libvma.conf
```

```bash
# rpm -qf /usr/lib64/libvma.so.9.7.2
libvma-9.7.2-1.x86_64

# rpm -qf /usr/lib64/libvma-debug.so
libvma-devel-9.7.2-1.x86_64

# rpm -qf /etc/libvma.conf
libvma-9.7.2-1.x86_64
```

```bash
# rpm -ql libvma-9.7.2-1.x86_64
/etc/libvma.conf
/usr/lib/systemd/system/vma.service
/usr/lib64/libvma.so
/usr/lib64/libvma.so.9
/usr/lib64/libvma.so.9.7.2
/usr/sbin/vmad
/usr/share/doc/libvma-9.7.2
/usr/share/doc/libvma-9.7.2/CHANGES
/usr/share/doc/libvma-9.7.2/README
/usr/share/licenses/libvma-9.7.2
/usr/share/licenses/libvma-9.7.2/LICENSE
/usr/share/man/man7/vma.7.gz
/usr/share/man/man8/vmad.8.gz

# rpm -ql libvma-devel-9.7.2-1.x86_64
/usr/include/mellanox
/usr/include/mellanox/vma_extra.h
/usr/lib64/libvma-debug.so

# rpm -ql libvma-utils-9.7.2-1.x86_64
/usr/bin/vma_stats
/usr/share/man/man8/vma_stats.8.gz
```

## 程序使用
只需要在正常的加上`LD_PRELOAD=libvma.so`即可让你的程序支持 RDMA:
```bash
# 服务端  
LD_PRELOAD=libvma.so VMA_STATS_FD_NUM=500 ./src/redis-server --bind 192.168.3.11 --port 6379 --protected-mode no --save  
# 客户端  
LD_PRELOAD=libvma.so VMA_STATS_FD_NUM=500 ./src/redis-benchmark -h 192.168.3.11 -p 6379 -n 1000000 -t set -c 100 -d 3000
```

VMA 通过 `LD_PRELOAD` 的方式拦截应用程序中的标准 `socket` 调用，将其重定向到用户空间库中处理。这种设计允许 VMA 优化部分或全部网络通信，而无需对应用程序代码进行显著修改。

## 限制
VMA 与 Mellanox 网卡（如 ConnectX 系列）深度集成，充分利用网卡的硬件加速功能。
 **vma 主要和 Mellanox 网卡集成，对 Intel 或其他厂商的网卡的支持非常有限**，
 通常不作为官方支持的目标。

## 其他
### 源码查看
#### 方式一：github
```bash
https://github.com/Mellanox/libvma
```

#### 方式二：libvma的 src.rpm提取
```bash
mkdir -p /tmp/vma_dir
cp /root/MLNX_OFED_SRC-5.8-6.0.4.2/SRPMS/libvma-9.7.2-1.src.rpm /tmp/vma_dir
cd /tmp/vma_dir

# rpm2cpio libvma-9.7.2-1.src.rpm | cpio -div
libvma-9.7.2.tar.gz
libvma.spec
3114 blocks

# tar xf libvma-9.7.2.tar.gz

# tree -L 1 libvma-9.7.2
libvma-9.7.2
├── aclocal.m4
├── CHANGES
├── config
├── config.h.in
├── configure
├── configure.ac
├── contrib
├── debian
├── docs
├── LICENSE
├── Makefile.am
├── Makefile.in
├── README
├── src
├── tests
└── tools

```
# 参考
```bash

```