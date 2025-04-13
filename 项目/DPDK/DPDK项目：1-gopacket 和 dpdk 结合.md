```table-of-contents
```

# 介绍
## DPDK
PDK（Data Plane Development Kit）是一个开源的软件项目，旨在提供快速的数据包处理能力，特别适用于高性能网络应用。DPDK通过优化Linux环境下数据包的接收和发送过程，显著提高了网络性能，降低了延迟。

DPDK的设计初衷正是为了解决x86架构在处理网络高吞吐量任务时遇到的瓶颈问题，通过一系列技术创新，显著提升了x86平台在网络数据包处理方面的性能，使其更适合于对吞吐量和延迟有严格要求的应用场景。

DPDK被广泛应用于需要高吞吐量、低延迟的网络环境中，通过使用DPDK，开发者可以构建出比传统方法更快、更高效的网络应用程序，这对于满足日益增长的数据流量需求至关重要。

## gopacket

gopacket 是一个用于Go语言的库，它提供了对网络数据包的低层次访问。**基于著名的C库 libpcap** ，gopacket **利用了Go语言的特性，如协程（并发性）、垃圾回收和类型安全性**，使得处理网络数据包更加高效和易于使用。
gopacket 常被用于网络安全分析、网络监控、性能分析等领域。例如，开发人员可以使用它来创建网络入侵检测系统（NIDS）、实时网络流量分析工具等。

### 主要功能
- **数据包捕获**：可以用来监听并捕获流经指定网络接口的数据包。
- **数据包解码**：支持多种协议（如以太网、IPv4、IPv6、TCP、UDP等）的数据包自动解码。

# 代码
代码在这里 `https://github.com/njcx/gopacket_dpdk` 。

# 流程
## DPDK的安装
```bash
CentOS
#  yum install -y libpcap-devel gcc gcc-c++ make meson ninja  numactl-devel  numactl  net-tools pciutils
#  yum install -y kernel-devel-$(uname -r) kernel-headers-$(uname -r)

Debian+Ubuntu
# apt install -y libpcap-dev gcc g++ make meson ninja-build libnuma-dev numactl net-tools pciutils
# apt install -y linux-headers-$(uname -r)


#  wget http://fast.dpdk.org/rel/dpdk-20.11.10.tar.xz
#  tar -Jxvf dpdk-20.11.10.tar.xz
#  cd dpdk-stable-20.11.10 && meson build && cd build && ninja && ninja install
#  export LD_LIBRARY_PATH=/usr/local/lib64:$LD_LIBRARY_PATH
#  git clone git://dpdk.org/dpdk-kmods && cd  dpdk-kmods/linux/igb_uio
#  make
#  modprobe uio  &&  insmod igb_uio.ko
#  dpdk-devbind.py --status
#  ifconfig ens38 down   ## 填写实际网卡
#  dpdk-devbind.py -b igb_uio 0000:03:00.0(pci-addr)  ## 根据实际填写

#  echo "vm.nr_hugepages=1024" | tee -a /etc/sysctl.conf
#  sysctl -p
```

## gopacket 的实现
### go调用c

### DPDK的初始化

## 范例


# 参考
```bash
# 把gopacket 和 dpdk 绑定一起
https://mp.weixin.qq.com/s/Iy8ynT3mAu6ugde8Zk5HBw
```