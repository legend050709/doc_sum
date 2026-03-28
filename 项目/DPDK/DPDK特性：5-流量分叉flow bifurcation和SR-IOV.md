```table-of-contents
```

# SR-IOV
## 背景

为了能够实现更高的虚拟化效率，让网卡“分身”成为PF和众多的VF，就可以把VF通过硬件直通的方式直通到虚拟机中，这样一来，就不再需要通过软件来实现虚拟交换机了，就可以大幅提升网络性能，并且减少虚拟化宿主CPU的压力。

![](attachments/Pasted%20image%2020240704200531.png)

在SR-IOV这个过程中，共诞生出来了两个产物，一个叫做PF，一个叫做VF。

其中，PF本身也是个接口，可以在Linux操作系统中用于数据传输，相对于VF的区别在于PF接口提供了额外的权限，可以用于管理整个网卡的功能，比如VF的数量。

### PCI直通

在讲述SR-IOV之前我们先讲述一下PCI直通技术。

什么是PCI直通？

PCI直通：不使用hypervisor也可以向虚拟机提供完整的网卡。虚拟机认为自己与物理网卡直接相连。如下图所示，有两个NIC卡和两个VNF，每个都独占访问其中一个NIC卡。

![](attachments/Pasted%20image%2020240620172754.png)

PCI直通缺点：
由于上面的两个物理网卡被VNF1和VNF3独占。并且没有第三个专用网卡分配给VNF2使用。

如果使用 SR-IOV，将当个网卡虚拟为多个虚拟网卡；通过创建PCIe设备的VF，每个VF可以分配给单个VM/VNF，从而消除由于网卡不够的问题。

![](attachments/Pasted%20image%2020240620172911.png)


## 介绍
SR-IOV全称 single root input/output virtualization，单根虚拟化：定义了一种用于虚拟化PCIe设备的机制。这种机制可以将单个PCIe以太网控制器虚拟化为多个PCIe设备。

Intel在早期为了支持虚拟化环境，在CPU和PCI总线上提供了三层虚拟化技术，它们分别是：
- 基于处理器的虚拟化技术VT-x
- 基于PCI总线实现的IO虚拟化技术VT-d
- 基于网络的虚拟化技术VT-c

从SRIOV的中文字面不难理解，它属于VT-d技术的一个分支，要实现SRIOV功能，前提条件就是你的网卡首先要支持SRIOV，你的主板要支持VT-d技术（支持VT-d自然也就支持SRIOV）。

SR-IOV 标准将一个PCIe的网络控制器虚拟化成多个PCIe设备，即多个PCI虚拟网卡，这些虚拟网卡不仅可以给虚拟机使用，也可以直接给操作系统使用，也可以给物理机上的DPDK使用。
>  ==SR-IOV 标准允许在虚拟机之间高效共享 PCIe==（Peripheral Component Interconnect Express，快速外设组件互连）设备，并且它是在硬件中实现的，可以获得能够与本机性能媲美的 I/O 性能。


==启用了 SR-IOV 并且具有适当的硬件和 OS 支持的 PCIe 设备（例如以太网端口）可以显示为多个单独的物理设备，每个都具有自己的 PCIe 配置空间==。


## PF 和 VF
SR-IOV 中的两种新功能类型是：
PF （Phycics function） ——物理功能
VF （Virutal Funtion）——虚拟功能。

相对于VF的区别在于PF接口提供了额外的权限，可以用于管理整个网卡的功能，比如VF的数量。VF接口就是个实打实的接口了，可以把它理解为一个**轻量**的PCIe设备，仅仅只包含了最基础的传输数据的功能。

在这个模型中，**网卡更多还充当了一个硬交换机的角色（Embedded Switch），众多的VF实际上就是接在这么个交换机上，再通过网卡的物理接口，和外部网络接通**。

![](attachments/Pasted%20image%2020240617154136.png)


###  PF
**物理功能 (Physical Function, PF)**
PF用于支持 SR-IOV 功能的 PCI 功能，如 SR-IOV 规范中定义，PF 包含 SR-IOV 功能配置结构体，用于管理 SR-IOV 功能。PF 是==全功能的 PCIe 功能==，可以像其他任何 PCIe 设备一样进行发现、管理和处理。PF 拥有完全配置资源，可以用于配置或控制 PCIe 设备。可以把PF理解为一个物理的PCIe设备。

### VF
**虚拟功能 (Virtual Function, VF)**
VF是与PF关联的一种功能，是一种==轻量级 PCIe== 功能，可以与物理功能以及与同一物理功能关联的其他 VF 共享一个或多个物理资源。VF 仅允许拥有用于其自身行为的配置资源。

#### VF 基本功能
参考：[Linux Base Driver for Intel(R) I40e driver Ethernet Adaptive Virtual Function](https://docs.kernel.org/networking/device_drivers/ethernet/intel/iavf.html)

I40e 驱动网卡的 VF的基本功能：
![](attachments/Pasted%20image%2020240626152339.png)



### PF和VF联系

1. PF就是物理网卡所支持的一项PCI功能，PF可以扩展出若干个VF
2. VF是支持SRIOV的物理网卡所虚拟出的一个“网卡”或者说虚出来的一个实例，它会以一个独立网卡的形式呈现出来，每一个VF有它自己独享的PCI配置区域，并且可能与其他VF共享着同一个物理资源（公用同一个物理网口）

**联系**
一旦在 PF 中启用了 SR-IOV，就可以通过 PF 的总线、设备和功能编号（路由 ID）访问各个 VF 的 PCI 配置空间。每个 VF 都具有一个 PCI 内存空间，用于映射其寄存器集。VF 设备驱动程序对寄存器集进行操作以启用其功能，并且显示为实际存在的 PCI 设备。
此功能使得虚拟功能可以共享物理设备，并在没有 CPU 和虚拟机管理程序软件开销的情况下执行 I/O。

### 获取PF和VF信息脚本
使用如下脚本可以获取到SR-IOV中，Physical Functions和Virtual Functions之间，清晰的关系。
 包括接口名称，domain、bus、slot、function的编号信息，接口mac地址信息，以及给VF是否被绑给了vm。
 
```bash
#!/bin/bash
function pf_vf(){
  echo "<=============>PF:$1<=============="
  echo "`lspci|grep $(ls -l /sys/class/net/$1/ |grep device |cut -d '/' -f 4|sed 's/0000://g')` --> $1"
  echo " VF:"
  eth_dev=`ls /sys/class/net/$1/device/virtfn* -l | cut -d ">" -f 2 |cut -d "/" -f 2`
  for i in $eth_dev; 
  do 
    vf_name=`ls /sys/bus/pci/devices/$i/net 2>&1`;
    if  [ $? -eq 0 ]; then
      echo " |_ `lspci|grep $(echo $i|sed 's/0000://g')` --> $vf_name"; 
      echo "   |_ _ $(ip link show |grep -w $vf_name -A1 |grep -v $vf_name|awk '{print $2}')";
    else
      echo "$i has bound by vm !"
    fi
  done
}
for i in $(ip link show |grep mq|cut -d ':' -f 2);do pf_vf $i;done
```


测试效果如下：
![](attachments/Pasted%20image%2020240626181451.png)



### 注意
**注意**
==缺省情况下，SR-IOV 功能处于禁用状态，PF 充当传统 PCIe 设备==。
所有的PF和VF共用一个物理的PCIe端口，通过Routing决定数据流的方向，所以**PF和各VF的数据带宽总和不超过实际物理的PCIe端口的带宽**。

## SR-IOV 的优缺点
**SR-IOV 架构** 

![](attachments/Pasted%20image%2020240621113501.png)


### 优点
使用 SR-IOV VF 而不是模拟设备的主要优点是：

- 提高的性能
- 减少主机 CPU 和内存资源使用量

例如，作为 vNIC 附加到虚拟机的 VF 性能几乎与物理 NIC 相同，并且优于半虚拟化或模拟的 NIC。特别是，当多个 VF 在单个主机上同时使用时，其性能优势可能非常显著。


### 缺点
在SR-IOV passthrough的场景下，虚拟机（VM）可以获得与裸金属主机上相比拟的网络性能。但是，仍然存在以下限制：


(1)要修改 PF 的配置，您必须首先将 PF 公开的 VF 数量改为零。

因此，您还需要将这些 VF 提供的设备从分配给虚拟机的设备中删除。

（2）SR-IOV VF passthrough到VM后，VM的迁移性会受限
主要原因在于SR-IOV这种passthrough I/O借助了Intel CPU VT-d（Virtualization Technology for Directed I/O）或AMD的IOMMU（I/O Memory Management Unit）技术，在VM上VF网卡初始化的时候，建立了Guest虚拟地址到Host物理地址的映射表，所以这种“有状态”的映射表在热迁移的过程中会丢失。

（3）
由于SR-IOV VF passthrough到VM，而SR-IOV PF直接连接到TOR上，在这种部署环境中虚拟机（VM）对外的网络需要自定义，如需要像OVS-DPDK那样自动开通网络，则需要将TOR加入SDN控制器的管理范畴，由SDN控制器统一管控，这样做通常会使网络运营变的非常复杂。


## Linux的 IAVF
IAVF (Intel Ethernet Adaptive Virtual Function)：自适应虚拟功能，不同英特尔以太网控制器上具有相同设备 ID (8086:1889)。

SR-IOV 标准将一个PCIe的网络控制器虚拟化成多个PCIe设备，即多个PCI虚拟网卡，**这些虚拟网卡不仅可以给虚拟机使用，也可以直接给操作系统使用，也可以给物理机上的DPDK使用**。

### IAVF驱动

IAVF驱动 是Linux 内核管理下的 创建生成的 VF 网口的 驱动 。也有可能是 ixgbevf 驱动、igbvf驱动。

如果使用DPDK来绑定VF，应该不需要 IAVF驱动，而是需要 vfio-pci 驱动。

### 范例

参考：[# testpmd / SR-IOV RX packets, but TX-errors](https://mails.dpdk.org/archives/dev/2019-October/147285.html)

主要的流程如下所示：

（1）2个物理口，每个物理口创建一个 vf；
（2）设置vf的mac地址
（3）设置物理口下的vf 为 trust vf
（4）绑定 vf 口 到 vfio-pci 驱动
（5）testpmd 指定 vf 的 pci 进行启动


![](attachments/Pasted%20image%2020240619105025.png)

![](attachments/Pasted%20image%2020240619105259.png)


## DPDK下的 VFIO

参考：[vfio-pci 驱动 ](https://doc.dpdk.org/guides-20.11/linux_gsg/linux_drivers.html#bifurcated-driver)

不同的 PMD 可能需要不同的内核驱动程序（比如： igb_uio, vfio-pci）才能正常工作。根据所使用的 PMD，应加载相应的内核驱动程序，并将网络端口绑定（dpdk-devbind.py）到该驱动程序。

### 介绍

![](attachments/Pasted%20image%2020240620112341.png)

VFIO 是一个依赖 IOMMU 保护的强大且安全的驱动程序。要使用 VFIO，必须加载 vfio-pci 模块。
```bash
sudo modprobe vfio-pci
```

### 限制条件

Linux 内核 、BIOS 需要支持 SR-IOV；网卡需要支持 SR-IOV

![](attachments/Pasted%20image%2020240620142113.png)

### 相关问题


![](attachments/Pasted%20image%2020240620141916.png)

## 操作与查看

### VF 的 设置与查看

#### 查看某个网卡是否支持SR-IOV

```bash
lspci -vv -s 5e:00.0 | grep -i iov
```

![](attachments/Pasted%20image%2020240619113948.png)

#### 查看PF支持的最大VF个数

![](attachments/Pasted%20image%2020240621114745.png)

```bash

sriov_totalvfs : PF 支持的 最大的 VF个数；
sriov_numvfs： 设置实际 PF的VF个数；

```

#### 设置 VF的个数
```
echo 3 > /sys/class/net/eth04/device/sriov_numvfs
```

![](attachments/Pasted%20image%2020240621110120.png)

![](attachments/Pasted%20image%2020240621110229.png)


#### VF的设置

```bash
(1) ip 命令 设置
ip link set { dev DEVICE | group DEVGROUP } [ { up | down } ]
...
[ vf NUM [ mac LLADDR ] [ vlan VLANID [ qos VLAN-QOS ] ]
...
[ spoofchk { on | off} ] ]
...

（2）sysfs 设置
sysfs configuration (ConnectX-4):
 
/sys/class/net/enp8s0f0/device/sriov/[VF]
 
+-- [VF]
| +-- config
| +-- link_state
| +-- mac
| +-- mac_list
| +-- max_tx_rate
| +-- min_tx_rate
| +-- spoofcheck
| +-- stats
| +-- trunk
| +-- trust
| +-- vlan

```


#### VF的统计

可以通过sysfs查询虚函数统计信息：
```bash
cat /sys/class/infiniband/mlx5_2/device/sriov/2/stats tx_packets : 5011
tx_bytes : 4450870
tx_dropped : 0
rx_packets : 5003
rx_bytes : 4450222
rx_broadcast : 0
rx_multicast : 0
tx_broadcast : 0
tx_multicast : 8
rx_dropped : 0

```

也可以通过 ethtool -S VF口 ，来查看VF的每个队列收到、发送的包的统计信息。如下所示：
```bash
# ethtool -S enp94s10
NIC statistics:
     rx_bytes: 12130
     rx_unicast: 134
     rx_multicast: 0
     rx_broadcast: 1
     rx_discards: 0
     rx_unknown_protocol: 0
     tx_bytes: 14278
     tx_unicast: 134
     tx_multicast: 47
     tx_broadcast: 0
     tx_discards: 0
     tx_errors: 0
     tx-0.packets: 134
     tx-0.bytes: 10988
     rx-0.packets: 0
     rx-0.bytes: 0
     tx-1.packets: 53
     tx-1.bytes: 3710
     rx-1.packets: 0
     rx-1.bytes: 0
     tx-2.packets: 7
     tx-2.bytes: 626
     rx-2.packets: 142
     rx-2.bytes: 12052
     tx-3.packets: 0
     tx-3.bytes: 0
     rx-3.packets: 0
     rx-3.bytes: 0

注：
	 rx-N.rx_packets : 第N个队列收的包的个数；
	 fdir_sb_match： sb (Sideband Perfect Filters) 匹配的 个数；
```


### SR-IOV MAC 反欺骗
#### pf下vf 的 mac地址设置

```bash
echo 1 > /sys/class/net/ethx/device/sriov_numvfs
```

如上所示，创建了一个VF，默认创建了VF之后，通过 `ip link `显示的 PF下的VF的mac地址为`00:00:00:00:00:00`; 一旦设置了 vf 口 up之后，自动填充了 PF下的 VF的mac地址。

![](attachments/Pasted%20image%2020240621153028.png)

```
也可以手动设置 VF的 mac地址：

ip link set ethx VF x mac xxxxxxxx

注：vf的mac地址不可以通过 ip link set VF mac xxxxxx 来设置，而是如上，通过 PF vf xxx mac xxxxxxx来设置。
```


#### 介绍
通常，MAC 地址是分配给网络接口的唯一标识符，并且是无法更改的固定地址。MAC 地址欺骗是一种更改 MAC 地址以达到不同目的的技术。更改 MAC 地址的某些情况可能是合法的，而另一些情况可能是非法的，并且会滥用安全机制或伪装可能的攻击者。

SR-IOV MAC 地址反欺骗功能（也称为 MAC 欺骗检查）可防止恶意虚拟机 MAC 地址伪造。如果网络管理员将 MAC 地址分配给 VF（通过虚拟机管理程序）并对其启用欺骗检查，则这将限制最终用户仅从该 VF 的分配的 MAC 地址发送流量。

####  MAC 防欺骗配置

默认情况下，MAC 反欺骗功能处于禁用状态。

在下面的配置示例中，虚拟机位于 VF-0 上，具有以下 MAC 地址：11:22:33:44:55:66。  
有两种方法可以启用或禁用 MAC 反欺骗：
```bash
启用 MAC 反欺骗
ip link set ens785f1 vf 0 spoofchk on

禁用：
ip link set ens785f1 vf 0 spoofchk off
```

### 每个VF的限制和带宽共享

此功能可在 SR-IOV 模式下对每个 VF 的流量进行速率限制。
在 /sys/class/net//device/sriov/<vf_num>/max_tx_rate 下.

![](attachments/Pasted%20image%2020240621111453.png)

### Trusted VF

如果恶意驱动程序在其中一个 VF 上运行，并且 VF 的权限不受限制，则可能会打开安全漏洞。例如，如果允许所有 VF 而不是特定 VF 进入混杂模式作为特权，这将使恶意用户能够嗅探和监视整个物理端口的传入流量，包括针对其他 VF 的流量，这被认为是严重的安全漏洞。

因此 VF的权限默认情况下会受到限制。然而，VF 可以被标记为可信，因此可以接收物理功能特权或许可的排他子集。


![](attachments/Pasted%20image%2020240619161709.png)

```bash
启用对特定 VF 的信任:
ip link set ens785f1 vf 0 trust on

```


#### VF 混杂 模式

![](attachments/Pasted%20image%2020240625121554.png)

VF 可以进入混杂模式，除了最初针对 VF 的流量之外，还可以接收不匹配的流量以及到达物理端口的所有组播流量。

> 注：不匹配的流量是与任何 VF 或 PF 的 MAC 地址不匹配的任何流量的 DMAC。
> 注意：**只有特权/受信任的 VF 才能进入 VF 混杂模式**。


**VM中接口设置混杂模式**
```bash
# ip link set <ethX> promisc on
Where <ethX> is a VF interface in the VM

# ip link set <ethX> allmulticast on
Where <ethX> is a VF interface in the VM
```

```bash
ethtool --set-priv-flags <ethX> vf-true-promisc-support on
```


### Probed VFs

启用 SR-IOV 后探测虚拟功能 (VF) 可能会消耗适配器卡的资源。因此，当不需要监控VM时，建议不要启用VF探测。

```bash
cat /sys/class/net/eth04/device/sriov_drivers_autoprobe
```

#### MDD
Malicious Driver Detection (MDD)

![](attachments/Pasted%20image%2020240625121639.png)



### 更改 PF 的VF 个数

比如： 之前 PF下存在3个VF，期望更改为 1个 VF。
不可以直接更改，而是需要先更改为0，然后再更改为1。如果直接更改，则会出现下列的错误。

```bash
# echo 1 > /sys/class/net/eth04/device/sriov_numvfs
-bash: echo: write error: Device or resource busy
```

## 支持 SR-IOV的网卡

[# SR-IOV 相关问题](https://www.intel.com/content/www/us/en/support/articles/000005722/ethernet-products.html)


![](attachments/Pasted%20image%2020240626170248.png)

### intel 网卡 

- **Intel® Ethernet 800 Series**
    - Intel® Ethernet Network Adapter E810 (All Product SKUs)
- **Intel® Ethernet Network Adapter X722 Series**
    - Intel® Ethernet Network Adapter X722-DA2
    - Intel® Ethernet Network Adapter X722-DA4
- **Intel® Ethernet Converged Network Adapter XL710 Series**
    - Intel® Ethernet Converged Network Adapter XL710-QDA1
    - Intel® Ethernet Converged Network Adapter XL710-QDA2
    - Intel® Ethernet Converged Network Adapter XL710-QDA1 for OCP
    - Intel® Ethernet Converged Network Adapter XL710-QDA2 for OCP
- **Intel® Ethernet Network Adapter XXV710 Series**
    - Intel® Ethernet Network Adapter XXV710-DA1
    - Intel® Ethernet Network Adapter XXV710-DA2
    - Intel® Ethernet Network Adapter XXV710-DA1 for OCP
    - Intel® Ethernet Network Adapter XXV710-DA2 for OCP
- **Intel® Ethernet Converged Network Adapter X710 Series**
    - Intel® Ethernet Converged Network Adapter X710-DA2
    - Intel® Ethernet Converged Network Adapter X710-DA4
    - Intel® Ethernet Converged Network Adapter X710-T4
    - Intel® Ethernet Controller X710/X557-AT 10GBASE-T
- **Intel® Ethernet Connection X722**
    - Intel® Ethernet Connection X722 for 10GBASE-T
    - Intel® Ethernet Connection X722 for 10GbE backplane
    - Intel® Ethernet Connection X722 for 10GbE QSFP+
    - Intel® Ethernet Connection X722 for 10GbE SFP+
- **Intel® Ethernet Converged Network Adapter X550**
    - Intel® Ethernet Converged Network Adapter X550-T1
    - Intel® Ethernet Converged Network Adapter X550-T2
- **Intel® Ethernet Converged Network Adapter X540**
    - Intel® Ethernet Converged Network Adapter X540-T1
    - Intel® Ethernet Converged Network Adapter X540-T2
- **Intel® 82599 10 Gigabit Ethernet Controller**
    - Intel® Ethernet 82599EB 10 Gigabit Ethernet Controller
    - Intel® Ethernet 82599ES 10 Gigabit Ethernet Controller
    - Intel® Ethernet 82599EN 10 Gigabit Ethernet Controller
- **Intel® Ethernet Converged Network Adapter X520**
    - Intel® Ethernet Converged Network Adapter X520-DA2
    - Intel® Ethernet Converged Network Adapter X520-SR1
    - Intel® Ethernet Converged Network Adapter X520-SR2
    - Intel® Ethernet Converged Network Adapter X520-LR1
    - Intel® Ethernet Converged Network Adapter X520-T2
- **Intel® Ethernet Controller I350**
    - Intel® Ethernet Controller I350-AM4
    - Intel® Ethernet Controller I350-AM2
    - Intel® Ethernet Controller I350-BT2
- **Intel® Ethernet Server Adapter I350**
    - Intel® Ethernet Server Adapter I350-T2
    - Intel® Ethernet Server Adapter I350-T4
    - Intel® Ethernet Server Adapter I350-F2
    - Intel® Ethernet Server Adapter I350-F4


#### VF参数
**X710/XL710  系列：i40e驱动**

![](attachments/Pasted%20image%2020240619145913.png)

X710/XL710 每个物理NIC最多支持 128个 VF，每个VF最多支持 16个 queue pair。


**82599  系列：ixgbe驱动**
![](attachments/Pasted%20image%2020240619150504.png)

82599 10G网卡，每个物理NIC有一个PF，最多支持63个VF（此时每个VF应该就是一个queue pair，即一个发送队列，一个接受队列）。如果VF个数配置较少（1-32个），则最多每个VF有4个  queue pair。

![](attachments/Pasted%20image%2020240619151159.png)


#### 配置
##### 修改Bios enable SR-IOV
![](attachments/Pasted%20image%2020240617142230.png)


##### 修改启动参数
![](attachments/Pasted%20image%2020240619115155.png)


```bash
- 要使 SR-IOV 设备分配正常工作，必须在主机 BIOS 和内核中启用 IOMMU 功能。要做到这一点：
    
    - 在 Intel 主机上启用 VT-d：
        
        1. 使用 `intel_iommu=on` 和 `iommu=pt` 参数重新生成 GRUB 配置：
            
            ```none
            # grubby --args="intel_iommu=on iommu=pt" --update-kernel=ALL
            ```
            
        2. 重启主机。
            
    - 在 AMD 主机上启用 AMD-Vi：
        
        1. 使用 `iommu=pt` 参数重新生成 GRUB 配置：
            
            ```none
            # grubby --args="iommu=pt" --update-kernel=ALL
            ```
            
        2. 重启主机。
```


![](attachments/Pasted%20image%2020240624163834.png)


```bash
 
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=vg00/Root rhgb quiet intel_iommu=on iommu=pt igb.max_vfs=1"

注：
igb.max_vfs 指igb驱动的网卡创建VF的个数。如果是其他驱动一样修改，如ixgbe.max_vfs。

将 max_vfs 放入到内核参数中，那么每次开机启动，就给nic创建了VF，不需要单独创建了？

```



##### 设置vf网卡mac地址，权限
```bash
ip link set enp5s0f1 vf 0 mac a0:36:9f:aa:64:9d
ip link set dev enp5s0f1 vf 0 trust on
ip link set dev enp5s0f1 vf 0 spoof off
ip link set enp5s0f1 allmulticast on

注：
这里有一个坑点，VF默认是拒绝multicast组播报文的。以至于ipv6 neighbor solicitation报文被过滤，VF网卡无法被邻居发现。查询了一圈资料，更换了多次驱动未果，终于发现PF开启allmulticast，VF才能收到左右组播报文，当然ipv6 nd也就成功了。
```

```bash
[root@localhost ~]# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 00:1b:21:bb:3d:00 brd ff:ff:ff:ff:ff:ff
3: enp5s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether a0:36:9f:09:63:9c brd ff:ff:ff:ff:ff:ff
    vf 0 MAC b2:6e:0e:68:0f:2b, spoof checking on, link-state auto, trust off
4: enp5s0f1: <BROADCAST,MULTICAST,ALLMULTI,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether a0:36:9f:09:63:9d brd ff:ff:ff:ff:ff:ff
    vf 0 MAC a0:36:9f:aa:64:9d, spoof checking off, link-state auto, trust on
```


##### 切换网卡驱动

DPDK支持SR-IOV的驱动只有 **igb_uio和vfio-pci，uio_pci_generic是无法接管VF的**。这里使用vfio-pci驱动。
```bash
modprobe vfio-pci
```

查看驱动的方式可以用DPDK工具dpdk-devbind.sh。
```bash
# dpdk-devbind.py -s
 
Network devices using DPDK-compatible driver
============================================
0000:05:10.0 'I350 Ethernet Controller Virtual Function' drv=vfio-pci unused=igbvf,uio_pci_generic
0000:05:10.1 'I350 Ethernet Controller Virtual Function' drv=vfio-pci unused=igbvf,uio_pci_generic
 
Network devices using kernel driver
===================================
0000:01:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection' if=enp1s0 drv=ixgbe unused=vfio-pci,uio_pci_generic 
0000:05:00.0 'I350 Gigabit Network Connection' if=enp5s0f0 drv=igb unused=vfio-pci,uio_pci_generic 
0000:05:00.1 'I350 Gigabit Network Connection' if=enp5s0f1 drv=igb unused=vfio-pci,uio_pci_generic 
0000:0a:00.0 'I210 Gigabit Network Connection' if=eno1 drv=igb unused=vfio-pci,uio_pci_generic 
0000:0b:00.0 'I210 Gigabit Network Connection' if=eno2 drv=igb unused=vfio-pci,uio_pci_generic 

```

![](attachments/Pasted%20image%2020240617145155.png)

![](attachments/Pasted%20image%2020240617145448.png)



##### 创建的 VF 持久化

通过为您用于创建 VF 的网络接口创建一个 udev 规则，使创建的 VF 持久化。例如，对于 _eth1_ 接口，创建 `/etc/udev/rules.d/eth1.rules` 文件，并添加以下行：

```bash
ACTION=="add", SUBSYSTEM=="net", ENV{ID_NET_DRIVER}=="ixgbe", ATTR{device/sriov_numvfs}="2"
```

这样可确保使用 `ixgbe` 驱动程序的两个 VF 在主机启动时可自动对 `eth1` 接口可用。如果不需要持久性 SR-IOV 设备，请跳过这一步。



#### 范例

**82599 10G网卡配置VF**
Linux 下  ixgbe 驱动使用：
```bash
rmmod ixgbe (To remove the ixgbe module)
insmod ixgbe max_vfs=2,2 (To enable two Virtual Functions per port)

```


Linux 下 DPDK使用
```bash
启动参数：
Kernel Params: iommu=pt, intel_iommu=on


驱动：
modprobe uio
insmod igb_uio
./dpdk-devbind.py -b igb_uio bb:ss.f
echo 2 > /sys/bus/pci/devices/0000\:bb\:ss.f/max_vfs (To enable two VFs on a specific PCI device)
```

##  SR-IOV 的数据包分发机制

参考：[](https://cloud.tencent.com/developer/article/2035806?from_column=20421&from=20421)

![](attachments/Pasted%20image%2020240709113327.png)



![](attachments/Pasted%20image%2020240619144248.png)

![](attachments/Pasted%20image%2020240627184739.png)

![](attachments/Pasted%20image%2020240627185124.png)

![](attachments/Pasted%20image%2020240627185150.png)

![](attachments/Pasted%20image%2020240627185501.png)

参考：[# SR-IOV, PCI Passthrough, and OVS-DPDK](https://study-ccnp.com/sr-iov-pci-passthrough-ovs-dpdk/)


其实，从逻辑上可以认为启用了 SR-IOV 技术后的物理网卡内置了一个特别的 Switch，将所有的 PF 和 VF 端口连接起来，通过 VF 和 PF 的 MAC 地址以及 VLAN ID 来进行数据包分发。

- **在 Ingress 上（从外部进入网卡）**：如果数据包的目的 MAC 地址和 VLAN ID 都匹配某一个 VF，那么数据包会分发到该 VF，否则数据包会进入 PF；如果数据包的目的 MAC 地址是广播地址，那么数据包会在同一个 VLAN 内广播，所有 VLAN ID 一致的 VF 都会收到该数据包。
    
- **在 Egress 上（从 PF 或者 VF 发出）**：如果数据包的 MAC 地址不匹配同一 VLAN 内的任何端口（VF 或 PF），那么数据包会向网卡外部转发，否则会直接在内部转发给对应的端口；如果数据包的 MAC 地址为广播地址，那么数据包会在同一个 VLAN 内以及向网卡外部广播。

**NOTE**：所有未设置 VLAN ID 的 VF 和 PF，可以认为是在同一个 LAN 中，不带 VLAN 的数据包在该 LAN 中按照上述规则进行处理。此外，设置了 VLAN 的 VF，发出数据包时，会自动给数据包加上 VLAN，在接收到数据包时，可以设置是否由硬件剥离 VLAN 头部。


如下所示：intel E810 系列收包处理流水线：

![](attachments/Pasted%20image%2020240703120453.png)


### 注意

我的理解是：上面仅仅是 配置了 SR-IOV，没有配置FDIR规则，然后在VF上配置Mac地址，就可以简单的实现流量分发到PF还是 特定的 VF。

实际在应用中，可以通过 设置 FDIR 规则（比如：指定dstport）将特定的流量导入到VF的队列，其他的流量继续走 PF的队列。


## SR-IOV性能

使用SR_IOV技术和纯物理机，以及用户态的ovs性能对比如下：

![](attachments/Pasted%20image%2020240619112436.png)




## DPDK vs SR-IOV

数据中心中存在东西向流量与南北向流量。

### 东西向流量
![](attachments/Pasted%20image%2020240617145925.png)

在同一个服务器内的东西向流量，DPDK性能优于SR-IOV。
这很容易理解:如果流量是在服务器内部路由/交换的，而不是到NIC。SR-IOV没有任何优势。相反，SR-IOV会成为一个瓶颈(流量路径会变长，网卡资源会被占用)。


### 南北向流量

在南北流量(也包括从一个服务器到另一个服务器的东西流量)的场景中，SR-IOV性能要优于DPDK。

![](attachments/Pasted%20image%2020240617150045.png)


## 使用范例

### intel E810网卡

参考：[dpdk with intel E810 网卡](https://edc.intel.com/content/www/us/en/design/products/ethernet/config-guide-e810-dpdk/virtual-function-vf-setup-with-dpdk/)

![](attachments/Pasted%20image%2020240619104339.png)

### 博通网卡使用 SR-IOV

参考：[# 配置BroadCOM网卡的SR-IOV功能](https://zhiliao.h3c.com/theme/details/24770)

(1) **首先在BIOS中开启网卡的SR-IOV的支持**

服务器开机自检按ESC或DEL进入BIOS Setup，点击Advanced -> 选中530FLR网卡。 默认Multi-Function Mode为SF，这里改成SR-IOV

![](attachments/Pasted%20image%2020240619114404.png)

(2) **操作系统中开启IOMMU支持**

执行dmesg | grep -i iommu看操作系统是否开启了IOMMU支持，如果没开启，则编辑如下
```bash
# vi /etc/default/grub

...

GRUB_CMDLINE_LINUX="nofb splash=quiet cOnsole=tty0 intel_iommu=on pci=realloc

...

注：在kernel中再加入一个参数pci=realloc，否则 在 设置vf个数时，echo 8 > /sys/class/net/ens9f0/device/sriov_numvfs， 可能会如下错误。

[  641.704649] bnx2x 0000:03:00.0: not enough MMIO resources for SR-IOV
[  641.704656] [bnx2x_enable_sriov:2514(ens9f0)]pci_enable_sriov failed with -12

```

![](attachments/Pasted%20image%2020240619121022.png)

参考：[# Unable to enable SR-IOV and receiving the message "not enough MMIO resources for SR-IOV"](https://access.redhat.com/solutions/37376)


(3) **开启网卡的VF端口**

注意：首先要确保端口是up状态
```bash
#ifup ens9f0
查看sriov的端口数量

# cat /sys/class/net/ens9f0/device/sriov_numvfs
0
如果返回结果是0，表示没有VF接口

# 设置
echo 8 > /sys/class/net/ens9f0/device/sriov_numvfs


# 查看
lspci | grep –i ethernet
ip addr show

如下所示，但是所有的mac地址都是00:00:00:00:00:00；
根据Broadcom bnx2x driver的readme描述，这属于正常情况
```

![](attachments/Pasted%20image%2020240619120830.png)

![](attachments/Pasted%20image%2020240619120843.png)


(4) **设置VF的mac地址**

### dpdk-testpmd 使用 SR-IOV

参考：[dpdk-testpmd 使用范例](https://doc.dpdk.org/guides-20.11/testpmd_app_ug/index.html)

参考：[dpdk i40e SR-IOV](https://doc.dpdk.org/guides-20.11/nics/i40e.html#sr-iov-prerequisites-and-sample-application-notes)

参考：[# VF daemon (VFd)](https://doc.dpdk.org/guides-21.11/howto/vfd.html)

查看其中的 VF相关范例。

![](attachments/Pasted%20image%2020240619155656.png)



# 流量分叉 flow bifurcation
## 介绍

![](attachments/Pasted%20image%2020240617165041.png)

![](attachments/Pasted%20image%2020240617165058.png)

![](attachments/Pasted%20image%2020240617165110.png)

## 优缺点

使用分岔驱动程序的PMDs与设备内核驱动程序共存。在这样的模型中，NIC由内核控制，而数据路径则由设备上的PMD直接执行。

**这种模式有以下好处**:

- 它是安全且健壮的.  
    因为内存管理和隔离是由内核完成的。
- 它允许用户在相同的网络端口上运行DPDK应用程序时使用传统的linux工具，如ethtool或ifconfig。
- 它允许DPDK应用程序只过滤部分流量，而其余的流量将由内核驱动程序定向和处理；流分岔由NIC硬件完成，例如，使用流隔离模式 [dpdk rte_flow isolated mode](http://doc.dpdk.org/guides/prog_guide/rte_flow.html#flow-isolated-mode) 可以严格选择DPDK中接收到的内容。

![](attachments/Pasted%20image%2020240617165141.png)

![](attachments/Pasted%20image%2020240617165317.png)


**总结**
- 更好的性能  
    流量分叉是硬件特性，不需要CPU的参与。可以提供更好的性能。
- 和 kni 对比  
    kni 的话，需要在DPDK中实现具体的代码来进行流量从DPDK应用到内核协议栈。流量分叉只需要通过软件给硬件配置对应的规则即可。


## 实现流量分叉的方式

参考：[# Flow Bifurcation How-to Guide](https://doc.dpdk.org/guides-20.11/howto/flow_bifurcation.html)

![](attachments/Pasted%20image%2020240619160641.png)

 实现流量分叉的方式
SR-IOV +  Flow filter(rte_flow/ fdir)

或者 Mellanox 网卡 +  Flow  director
> 注：感觉 Mellanox 网卡的流分叉， 也是在网卡内部默认启动了SR-IOV，匹配`rte_flow`规则的流量走了`某个VF（感觉像是底层默认创建的）`下的队列，给DPDK应用程序；不匹配的流量则给了PF的队列，给内核协议栈；

## 流量分叉的理解

### SR-IOV 和  FDIR的关系
我的理解是：上面仅仅是 配置了 SR-IOV，没有配置FDIR规则，然后在VF上配置Mac地址，就可以简单的实现流量分发到PF还是 特定的 VF。

实际在应用中，可以通过 设置 FDIR 规则（比如：指定dstport）将特定的流量导入到VF的队列，其他的流量继续走 PF的队列。

### FDIR 规则，如何配置呢
我认为应该是直接通过 ethtool 在 PF上进行配置，将指定的流量导流到指定的VF；
只有某个VF下的多个队列中的流量分发规则，则是在DPDK程序中设定。
具体就是：DPDK应用程序通过VFIO驱动/Igb_uio驱动绑定到VF上，在DPDK中，可以具体在VF的多个队列上设置规则，比如 RSS规则。因为 VF 如果被 DPDK绑定了之后，在Linux上可以看不到VF口了，因此在DPDK程序中设置VF的多个队列的相关规则。


### ethtool 设置导流到 VF

ethtool 通过下面的命令 设置过滤规则以及动作：
```bash
-N -U --config-nfc --config-ntuple
-n -u --show-nfc --show-ntuple
-k --show-features --show-offload
-K --features --offload


范例
（1）查看网卡是否支持 ntuple 过滤
ethtool -k eth1

（2）开启、关闭 nutple 过滤
ethtool -K eth0 ntuple on


（3）设置过滤规则

ethtool --config-ntuple eth0  rx-flow-hash tcp4 sdfn
设置 RSS规则，其中 sdfn 是一个字符串，可以通过 man ethtool 来查看

ethtool -U eth0 flow-type tcp4 dst-port 80 action 2
添加 ntuple 过滤规则，让目标端口80的TCP流发送到RX队列2

（4）查看过滤规则
ethtool --show-ntuple eth0 rx-flow-hash tcp4
```



**man ethtool **

![](attachments/Pasted%20image%2020240620174531.png)


#### ethtool 范例

![](attachments/Pasted%20image%2020240620175605.png)


### ethtool 设置N元祖导流到VF不生效问题

#### 方式一：ethtool 设置 VF

![](attachments/Xnip2024-06-23_17-26-54%201.jpg)

如上所示，ethtool 可以设置VF，但是好像是没法设置VF的具体的哪个queue。
对于较低版本的 ethtool ，都无法设置VF。那么VF的设置是通过其他的方式来进行设置的。

```bash
If you are using ethtool version 4.11 and later, we would suggest to use "vf" and "queue" parameters instead of the "action' parameter stated in the readme file of the driver.
```


#### 方式二：不支持设置VF的ethtool

![](attachments/Pasted%20image%2020240623174053.png)

如上，不支持 ethtool 设置 VF，VF的队列的设置是通过action来进行设置的。

![](attachments/Pasted%20image%2020240624113335.png)

如上所示， action的 值的设置。


如下所示：

![](attachments/Pasted%20image%2020240623174221.png)

```bash
$queue_index_in_VFn: 
	Bits 39:32 of the variable defines VF id + 1; 
	the lower 32 bits indicates the queue index of the VF. Thus:

因此：
	 $queue_index_in_VF0 = (0x1 & 0xFF) << 32 + [queue index]
	 $queue_index_in_VF1 = (0x2 & 0xFF) << 32 + [queue index]

```




范例：
```bash
比如： VF0的某个队列的索引：
$queue_index_in_VF0 = (0x1 && 0xFF) << 32 + [queue_index]

ethtool -N p3p4 flow-type udp4 src-ip 66.110.22.99 dst-ip 192.168.50.108 action 4294967296 

or

ethtool -N p3p4 flow-type udp4 src-ip 66.110.22.99 dst-ip 192.168.50.108 user-def 0xffffffff00000000 action 1 

注：4294967296 = 2^32 = 0x100000000

ethtool -N eth1 flow-type udp4 src-ip 192.0.2.2 dst-ip 198.51.100.2 \
        action $queue_index_in_VF0
        
ethtool -N eth1 flow-type udp4 src-ip 198.51.100.2 dst-ip 192.0.2.2 \
        action $queue_index_in_VF1
        
```




参考：[fdir to a VF in X710 NIC not work](https://community.intel.com/t5/Ethernet-Products/Flow-director-configuration-not-working-Need-help-in-identifying/m-p/725353)

参考：[dpdk19.11- flow_bifurcation ](https://doc.dpdk.org/guides-19.11/howto/flow_bifurcation.html)



## 不同网卡的流量分叉


### mellanox

![](attachments/Pasted%20image%2020240617165421.png)

```bash
receiving VXLAN 42 in 4 queues of the DPDK port 0, while all other packets go to the kernel:

testpmd> flow isolate 0 true
testpmd> flow create 0 ingress pattern eth / ipv4 / udp / vxlan vni is 42 / end actions rss queues 0 1 2 3 end / end
```

Mellanox Cx系列网卡**天然支持流量分叉**，不需要在配置SR-IOV PF/VF 进行流量分叉。
> 注：感觉像是Mellanox Cx系列网卡，在DPDK使用该网卡时，自己在内部创建了VF，配置了FDIR规则，匹配的流量进入了该VF的队列，给了DPDK应用程序；不匹配的流量走了PF的队列，给了内核协议栈；

Mellanox Cx系列流量分叉的好处有：
- 更好的性能，DPDK应用直接处理数据面的流量。
- 网卡依然可以被内核控制。
- Linux kernel 的控制工具/命令依然可以使用。比如，ethtool


#### DPDK 使用  Mellanox 网卡

参考：[Mellanox Bifurcated DPDK PMD](https://www.dpdk.org/wp-content/uploads/sites/35/2016/10/Day02-Session04-RonyEfraim-Userspace2016.pdf)

**单个设备上运行单个DPDK程序**

![](attachments/Pasted%20image%2020240620162742.png)

![](attachments/Pasted%20image%2020240620163416.png)

Mellanox PMD 可以理解为 使用 Mellnaox 网卡的 DPDK 应用程序；
在 PMD中通过 API的方式来更新、获取MTU，以及设置流控的规则。
通过 DPDK filter 来过滤需要的流量到 PMD中，剩余的 流量走 linux 内核协议栈。

**单个设备上运行多个DPDK程序**

![](attachments/Pasted%20image%2020240620163219.png)

#### DPDK 使用 Mellanox 网卡的特殊之处

DPDK通过 `-w/-a PCIe` 来使用  Mellanox 网卡，存在一些特殊的地方。

**（0）不需要安装类似于igb_uio这样的驱动**
DPDK程序 通过 `-w/-a PCIe`来使用 Mellanox网卡，不像使用 intel 等其他网卡，需要解绑原始的网卡驱动，然后将网卡绑定到 igb_uio 驱动上。


**（1）mellanox物理网卡内核可见、linux工具可用**
DPDK程序 通过 `-w/-a PCIe`来使用 Mellanox网卡之后，物理网卡依然在内核中可见，通过 `ip a` 依然可以看到。另外，ethtool/ ifconfig 等命令依然可见。


**（2）流量分叉**
可以在DPDK程序中通过 `rte_flow_isolate` + 下发指定的 `RSS/FDIR` 规则，将匹配到规则的流量导流到 DPDK程序，其他的控制流量还是PASS 给内核，进而做到了流量分叉，这样即使DPDK程序挂掉，也不会导致系统的不可用。

注：在 DPDK中通过 `rte_flow` 创建的规则，通过 `ethtool -n ethx` 也是无法看到的。==感觉 Mellanox 网卡实现流量分叉，驱动里可能也是封装了PF/VF，DPDK绑定Mellanox网卡，可能就是在内层新建了 VF，DPDK中设置的rte_flow 作用于这个VF，这个VF又不对外暴漏==。DPDK程序退出时，这样的VF及其规则都自动销毁了。

**（3）抓包**
DPDK 使用 Mellnaox 网卡，Mellanox 网卡使能 sniffer 特性，那么就可以在Linux中使用 tcpdump 抓到 DPDK程序收到的 数据包。

![](attachments/Pasted%20image%2020240912131700.png)

> 注：低版本的 MLNX_OFED（5.1以下），要用ethtool --set-priv-flags ethxx sniffer on开启dump报文，之后就可以正常用tcpdump抓包了；
>  对于高版本的 MLNX_OFED，没有sniffer选项了。需要**高版本的tcpdump**才能抓包。使用方式是==./tcpdump -i mlx5_xx -nnn==需要将==`mlx5_xx`替换成`ibdev2netdev`看到的网卡的名称==。


### 博通系列网卡
参考：[# DPDK-22.11-BNXT Poll Mode Driver](https://doc.dpdk.org/guides-19.11/nics/bnxt.html)

![](attachments/Pasted%20image%2020240627115619.png)



### intel系列网卡

其他系列的网卡，比如Intel 网卡不是天然的流量分叉。需要配置SO-IOV +  FDIR 规则来实现流量分叉。

![](attachments/Pasted%20image%2020240620164344.png)

支持流量分叉的网卡（主要是Mellanox网卡），他们的驱动不可以从linux 内核中解绑定；
对于不支持流量分叉的网卡，如果DPDK程序使用它，则需要将他们从linux 内核中解绑定，然后绑定到 `vfio-pci`, `uio_pci_generic`, or `igb_uio` 驱动上。


参考：
[# DPDK-19.11-I40E Poll Mode Driver](https://doc.dpdk.org/guides-19.11/nics/i40e.html)
[# DPDK-22.11-ICE Poll Mode Driver](https://doc.dpdk.org/guides-22.11/nics/ice.html)
[# DPDK-19.11-IXGBE PMD Driver](https://doc.dpdk.org/guides-19.11/nics/ixgbe.html)


#### intel ixgbe 驱动的网卡

The driver is compatible with devices based on the following:

> - Intel(R) Ethernet Controller 82598
> - Intel(R) Ethernet Controller 82599
> - Intel(R) Ethernet Controller X520
> - Intel(R) Ethernet Controller X540
> - Intel(R) Ethernet Controller x550
> - Intel(R) Ethernet Controller X552
> - Intel(R) Ethernet Controller X553


参考：[intel ixgbe 驱动系列](https://www.kernel.org/doc/html/v4.20/networking/ixgbe.html)

比如： 在 82599 网卡上实现流量分叉，可以通过SR-IOV +  flow director. 

![](attachments/Pasted%20image%2020240623182054.png)

> 注意：此中使用 的内核版本在 4.2及其以上，ethtool 版本在 3.18及其以上，否则可能会有问题。


 ixgbe 驱动的 patch 如下所示：
 
![](attachments/Pasted%20image%2020240624103323.png)

![](attachments/Pasted%20image%2020240624104236.png)

> 在内核 4.2 版本中已经打上了 上诉的 ixgbe 以及 ethtool.h 的 patch。


##### max_vfs

![](attachments/Pasted%20image%2020240624145509.png)

##### FDIR规则

![](attachments/Pasted%20image%2020240624145850.png)

Intel 网卡 设置的 FDIR 规则 中的掩码和  子网掩码的规则，是相反的。
比如：
```bash
src-ip 172.4.1.2 m 255.0.0.0 实际上得到的网段是 ：

172.4.1.2 & （^ 255.0.0.0） = 0.4.1.2
```

注：如果使用  `ethtool -K ethX ntuple off` 关闭 了n元祖过滤规则，那么配置的所有的规则都将失效。需要重新设置。

###### atr
atr:   Application Targeted Routing   

![](attachments/Pasted%20image%2020240624154826.png)

查看是否开启：

![](attachments/Pasted%20image%2020240624154522.png)

###### Sideband Perfect Filters

![](attachments/Pasted%20image%2020240624150711.png)

注意：看上诉的最后一段，FDIR 应该是不会打破内部的流量规则。如果流量之前没有被发送给指定的VF，FDIR也不会导流给VF。
```text
Note that these filters will not break internal routing rules, and will not route traffic that otherwise would not have been sent to the specified Virtual Function.
```

#### intel i40e 驱动 网卡

##### ethtool 的  user-def 参数

参考：[#  Intel(R) Ethernet Controller 700 Series](https://www.kernel.org/doc/html/v4.20/networking/i40e.html)


![](attachments/Pasted%20image%2020240624110300.png)

##### 相关限制

（1）flow-type 相同的多个过滤规则，需要有相同类型的匹配条件。

![](attachments/Pasted%20image%2020240624112406.png)

注：用户定义（user-defined）的灵活偏移也被视为输入集（input set）的一部分，不能为同一类型(flow-type )的多个滤波器单独编程。然而，灵活数据不是输入集的一部分，并且多个过滤器可以使用相同的偏移量但匹配不同的数据。


##### i40e 的 ethtool 设置规则

![](attachments/Pasted%20image%2020240624120745.png)

![](attachments/Pasted%20image%2020240624120917.png)

```
如上所示：
(1) user-def 一共是 64bit；
如果高32bit是 0xffffffff，那么过滤器规则就是当做 L3 VEB filter, 即 非隧道包。
如果高32bit不是 0xffffffff，那么 过滤器规则就是当做 Cloud Filter，高32bit 携带 vni；

（2） user-def 的 低32位：
user-def 的 低32位 指定的是 VF，如果 低32位的值 >= max_vfs, 那么 代表的是PF;
action 字段指定具体的  queue。


(3) Cloud Filter：
dst 表示的是 外层；
src 表示的 内层；

```




 **范例**

![](attachments/Pasted%20image%2020240624121926.png)

![](attachments/Pasted%20image%2020240624121951.png)


##### VF的QP个数
参考: [## x710 open sr-iov, vf increase max queues](https://community.intel.com/t5/Ethernet-Products/x710-open-sr-iov-vf-increase-max-queues/td-p/596100)


**问题**

![](attachments/Pasted%20image%2020240627182015.png)

即： x710 网卡开启 SR-IOV，每个VF只有4个QP，期望增加每个VF的QP数量。更改驱动代码之后，从4个更改为16个，可以看到队列数为16个，但是中断到达4个队列中。




**分析**
```bash
I40E_DEFAULT_QUEUES_PER_VF
I40EVF_MAX_REQ_QUEUES
```

参考：[x710的特性矩阵图](https://www.intel.com/content/dam/www/public/us/en/documents/release-notes/xl710-ethernet-controller-feature-matrix.pdf
)

![](attachments/Pasted%20image%2020240627182413.png)
![](attachments/Pasted%20image%2020240627182516.png)

![](attachments/Pasted%20image%2020240627182646.png)

```bash
注：x 表示支持的意思；--- 表示不支持；
```

![](attachments/Pasted%20image%2020240627182729.png)

即： 如果是内核协议栈的中断方式，每个VF支持4个QP；如果是 DPDK方式，每个VF支持16个QP。

![](attachments/Pasted%20image%2020240627182949.png)



####  intel ice驱动系列网卡

参考：[# Linux Base Driver for the Intel(R) Ethernet Controller 800 Series](https://docs.kernel.org/networking/device_drivers/ethernet/intel/ice.html)

参考：[intel ice 驱动网卡](https://www.kernel.org/doc/html/v4.20/networking/ice.html)

参考：[dpdk 20.11 intel-ice 驱动 适配](https://www.intel.com/content/www/us/en/content-details/633514/intel-ethernet-controller-e810-data-plane-development-kit-dpdk-20-11-configuration-guide.html?wapkw=E800)


#####  ice驱动网卡在DPDK中使用范例

> 注：使用 igb_uio 或者  vfio-pci 驱动 来 进行 VF的 接管。

![](attachments/Pasted%20image%2020240624161426.png)

![](attachments/Pasted%20image%2020240624162043.png)



## 流量分叉的应用场景

### 减少网卡的数量

正常情况下，在一个机器上部署DPDK程序，需要2块网卡：

> - 管理口：  
>     登陆机器，管理机器。
> - 业务口：  
>     一个单卡双口的网卡作为DPDK程序转发流量的业务口。

如果使用基于SR-IOV的 flow bifurcation，只需要一块卡即可。利用网卡的SR-IOV，存在一个PF以及多个VF。  
在网卡上配置管理IP，以及一些flow filter。  
将DPDK的流量，交给VF对应的队列，进而给VF处理。  
将其他流量交给PF处理，对设备进行管理。这样即使DPDK程序退出，不影响设备的管理。


### DPDK程序中控制流量和业务流量分离

目前的DPDK程序，比如DPVS，控制流量（健康检查、bgp流量）也可能和业务流量混在一起（比如交给了相同的接收队列）。在业务流量很大的情况下，有可能导致控制流量丢包，进而导致BGP保活失败，VIP路由发送失败，流量偶发断连的情况。  
如果可以将控制流量和业务流量分发到不同的队列中，做到互不影响。


# intel 的E810网卡的ADQ特性

参考：
```bash
# SPDK总体技术介绍 
https://spdk.io/cn/articles/

其中包含多个ADQ的介绍，如下所示：
# # 在SPDK中使能E810网卡ADQ 特性

https://mp.weixin.qq.com/s?__biz=MzI3NDA4ODY4MA==&mid=2653337354&idx=1&sn=2bccc7b8681f5bde5a9e82b58d9f7e35&chksm=f0cb408dc7bcc99b37d4d77cd572de5c3ff092a4c0cb80766e3841050bec620c298d710842c0&scene=0&xtrack=1&exportkey=Aw4ff5FCzsijjG81i0a45J4%3D&pass_ticket=w%2BZq2BsgB7kRtHy74eCg%2Bg3OzQ1%2BagIyRNE8HTReHBBzgY%2BPoBJ2KHsJWJVus9X2&wx_header=0#rd

# CVL网卡的ADQ特性在SPDK的NVMF测试中的应用实例 - 上篇
# CVL网卡的ADQ特性在SPDK的NVMF测试中的应用实例 - 下篇
# 在SPDK中体验一下E810网卡ADQ直通车

# # TC queue based filtering 
https://docs.kernel.org/networking/tc-queue-filters.html
```


# 参考
```bash
# DPDK中的流量分叉(flow bifurcation)
https://blog.csdn.net/legend050709/article/details/124872468

# Frequently Asked Questions for SR-IOV on Intel® Ethernet Server Adapters
https://www.intel.com/content/www/us/en/support/articles/000005722/ethernet-products.html


# Intel® Ethernet Adaptive Virtual Function (AVF) Hardware Architecture Specification (HAS)
https://www.intel.com/content/dam/www/public/us/en/documents/product-specifications/ethernet-adaptive-virtual-function-hardware-spec.pdf

# 官网 中  SR-IOV相关
https://doc.dpdk.org/guides-20.11/nics/index.html
https://doc.dpdk.org/guides-20.11/nics/bnx2x.html#sr-iov-prerequisites-and-sample-application-notes
（博通网卡的SR-IOV）

https://doc.dpdk.org/guides-20.11/nics/intel_vf.html#sr-iov-mode-utilization-in-a-dpdk-environment
(intel 的 SR-IOV的介绍)

https://doc.dpdk.org/guides-20.11/nics/i40e.html#sr-iov-prerequisites-and-sample-application-notes
(intel i40e 驱动网卡的 SR-IOV)


```