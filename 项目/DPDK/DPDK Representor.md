```table-of-contents
```
# 背景
## DPU中的快慢路径
### 为什么会有DPU

网络数据的转发需求非常大，以我们常用的K8S网络，它有这么这么多的需求需要满足：

- L2转发。
- L3转发。
- NAT。需要实现Service的能力，还需要实现Pod出网的能力，所以既要DNAT，又要SNAT，由于要实现Pod出网，在一堆Pod共享一个出网IP的情况下，conntrack也得支持了。
- ACL。需要实现NetworkPolicy的能力，单单这个ACL的需求就非常多了，毕竟有那么多的协议、字段需要支持。
- ARP代答。在大二层的网络里，尤其是BGP EVPN的方案中，需要ARP代答来控制泛洪。
- Tunnel封装、解封装。比如VXLAN、GRE等，在BGP EVPN或者VRF跑VPC的场景里，不打封装根本没办法玩。
- …

可以看到，网络的需求有这么多，但是**硬件的能力就那么点**，所以让一个硬件实现完整的功能支持并不现实，即便是FPGA也不能这样既要又要，因此，**把硬件和软件转发打包在一起组成一个DPU，就可以让它去包办所有网络相关的工作，Host就只需要关注业务了**。


### DPU如何加速数据传输
这里我拿容器中落地比较多的SR-IOV方案来阐述。

容器场景（Kubernetes）和虚拟机有点不太一样，他们虽然都算是基础设施平台，但是交付的资源的类型和对业务的感知程度有些许差别。在容器场景中，平台能够直接感知业务Pod的状态，可以说离业务更近了，业务可靠性通过多副本实现，因此可以一定程度容忍Pod失败。因此，容器场景下，不需要通过热迁移来保证Pod高可用，SR-IOV非常适合。

但是在虚拟机场景下，虚拟机要能够支持热迁移，不能引入硬件依赖，因此SR-IOV无法胜任，vDPA反而是最佳选择。

在SR-IOV加速的方案中，DPU会有两个职能部分（如下图）

![](attachments/Pasted%20image%2020240708113204.png)

（1）在通用芯片（比如 CPU）中运行的OVS-DPDK，来作为慢速路径。

（2）在ASIC/FPGA中的快速路径（图中为offloading path，卸载路径）。通过首包后下发的硬件规则（switching rule）将数据包从VF到VF直接转发，可以非常轻松达到线速，大幅提高性能。

这样加速之后，就形成了一个效果，一条流（flow）的首包发出之后，只要知道它该去哪里了，我就把这个路径告诉DPU，这样DPU每次在查找匹配到这条流之后，就直接从另外端口发出（或者做一些其他操作）就好了，就不用再来问问慢速路径该怎么走了，毕竟慢速路径转发也是靠通用芯片，效率肯定比不上ASIC/FPGA。通过这种抄近路的办法，DPU就可以把数据传输加速了。

总结：**即首包上送通用芯片，软件转发，然后下发规则到硬件，后续包匹配硬件规则转发作为快路径**。



### 为什么要区分快速、慢速路径

首先我们要知道，这两条路径是它们两个间相对的。

快速路径指的是可编程芯片（FPGA）或者专用芯片（ASIC）等中的转发路径，慢速路径指的是通用芯片（x86、ARM等）中的转发路径。慢速路径可以和快速路径打包在一起组成DPU，也可以转移到Host上，原理上区别不大。
> 注：慢路径可以是DPU中的CPU，也可以是 Host中的  CPU，两者区别不大。

快速路径虽然很快，但是它只能实现简单的操作，**硬件级别Match/Action的能力有限**，没有办法像慢速路径的通用芯片那样可以依靠**多条指令**处理非常复杂的转发，因此快速路径没有办法包揽业务的全部需求，需要慢速路径出面，对快速路径中硬件解决不了的转发需求进行补盲，共同支撑DPU的功能。


### DPU和智能网卡对比

此处，我拿一张来自Netronome的胶片，来一起看看智能网卡会怎么处理。

如下图，图片名称为OVS Datapath Hooks。

![](attachments/Pasted%20image%2020240708164314.png)

如下图，图片名称为OVS-TC。
![](attachments/Pasted%20image%2020240708164327.png)

可以看到，智能网卡和DPU一样，能够对特定的flow进行加速（卸载），上边的两幅图中都是硬件加速，只是实现的方式不同而已，但是最终都只是实现了快速路径，慢速路径的工作仍然还留在Host上。
而DPU，可以把快速路径和慢速路径都转移到自己身上，这可能就是和智能网卡的最大差别吧。

### 总结

整体上来讲，我觉得DPU之所以能够被叫做DPU而不是智能网卡，是因为DPU不仅可以和智能网卡一样加速多种流，同时还整合了通用芯片来运行慢速路径，这两条路径协同起来，整体上看起来DPU才是真正一个独立的、可以处理一切流量的网络处理单元，才能从Host上解放了网络处理的工作，整体上提高网络的性能，让Host更加专注业务本身。

## SR-IOV介绍

SR-IOV技术最开始来源于虚拟机场景中，为了能够实现更高的虚拟化效率，让网卡“分身”成为PF和众多的VF，就可以把VF通过**硬件直通（Pcie直通)** 的方式直通到虚拟机中，这样一来，就不再需要通过软件来实现虚拟交换机了，就可以大幅提升网络性能，并且减少虚拟化宿主CPU的压力。

![](attachments/Pasted%20image%2020240708113859.png)

在SR-IOV这个过程中，共诞生出来了两个产物，一个叫做PF，一个叫做VF。

其中，PF本身也是个接口，可以在Linux操作系统中用于数据传输，相对于VF的区别在于PF接口提供了额外的权限，可以用于管理整个网卡的功能，比如VF的数量。如下的代码块，就是PF设备的驱动提供的额外权限，可以用于创建VF，仅仅PF接口的驱动才会有这个文件。

```bash
echo "10" > /sys/class/net/ens6/device/sriov_numvfs
```

VF接口就是个实打实的接口了，可以把它理解为一个轻量的PCIe设备，仅仅只包含了最基础的传输数据的功能。

在这个模型中，**网卡更多还充当了一个硬交换机的角色（Embedded Switch）**，众多的VF实际上就是接在这么个交换机上，再通过网卡的物理接口，和外部网络接通。

# VF Representor
其实最开始我是觉得它应该归属到SR-IOV的范畴，但是之所以拆出来讲，是随着了解的深入，我觉得它更多算是SR-IOV的副产物，**它并不算是SR-IOV这个硬件体系下的概念，只是各家软件系统在为了对齐标准的网络栈因而创造的概念**。

VF Representor（后文简称VF Rep，VF代表），在DPDK中也叫做Port Representor，是DPDK在管理PF时由DPDK PMD驱动创建的接口，本质是个ethdev。
**在大多数情况下，switching rule（在DPDK中叫做traffic steering rule）是没有办法预先确定的，因此快速路径必须要慢速路径告知，通过rte_flow接口将整一条flow下发到快速路径，才能完成所谓的硬件加速**。

那么实现这样的协同处理，网卡就必须要提供switch rule的other end选项，能够**将失配规则的流通过某一种方式送回软件处理**，这就是Representor诞生的意义，翻译成中文就是“代表”，也就可以理解成VF转不动的包，就让VF代表来，还是挺有意思的。

在有的智能网卡中，这个VF Rep也会被驱动支持，在Linux中呈现单独的接口，其目的和DPDK中的一样，都是为了能够实现标准的网络栈，能够通过VF Rep上送fallback的流量而已。


## Representor如何收包
再一起来看看DPDK里是如何处理VF Rep接口的收包的。

此处我拿Intel的ixgbe驱动来举例，VF Rep的初始化流程大概为：

```bash
- `rte_eal_init()`
    - `rte_bus_probe()`
        - `bus->probe()` 这里会到驱动的`eth_ixgbe_pci_probe`函数
            - `rte_eth_dev_create()` 创建PF的ethdev
            - `rte_eth_dev_create()` 创建VF Rep的ethdev
                - `ixgbe_vf_representor_init()` 配置队列的数量、链路信息等
                
- `rte_eth_dev_start()`
    - `(*dev->dev_ops->dev_start)(dev)` 这里会到驱动的`ixgbe_dev_start`函数
        - `ixgbe_dev_rx_init()` 每个VF Rep会分配独立的mbuf到rx ring中，最后通过`IXGBE_WRITE_REG`函数写入地址到网卡寄存器

```

起初，我以为网卡会为每个数据包在头端中添加额外的字段来标识这个数据包来自于哪个VF，随着分析DPDK的代码发现，其实并没有这些逻辑，网卡本身只需要匹配switching rule然后丢到合适的队列就好了，失配的就成other end，丢到VF Rep的队列，等慢速路径的PMD拿走处理就好，整个环节似乎没有理由再去添加专用的头端域。

在这个过程中，网卡非常依赖大量的队列，然而这些队列又都是通过DMA映射的物理内存，考虑到VF与VF间的隔离，我相信这里投递的disc，其内部的data也一定是从一个buf中复制到了另外的buf，应该不太可能能全部ZeroCpoy（吧？）。

因此，是不是可以得出一个结论，内存性能会非常影响网络性能（仅在慢速路径中）？


## Datapath

为了能够更直观一些，我画了个图来说明datapath，如下图。

![](attachments/Pasted%20image%2020240708163914.png)

图中共两条路径A和B。

路径A为快速路径，即已经通过rte_flow下发到硬件的flow信息，将会由硬件从VF的tx队列中Match到，然后直接Action转发到所选择的VF的rx队列中。

路径B为慢速路径，即当数据包在VF的tx队列中Match时失配，通过VF Rep上送到OVS-DPDK，在UIO的OVS-DPDK中完整查表后，找到目的的VF Rep口并发出，最终就会被送回到关联的VF口。最后，如果这一条流能够被快速路径加速，OVS-DPDK会再通过UIO驱动程序操作PF的寄存器，将这一条流的信息下发到快速路径中，后续就可以实现硬件加速。

## 快速路径的瓶颈

如果说Representor本质上只是系统或者软件的概念，那么意味着所有能够**支持SR-IOV并且能够支持fallback上送路径**的网卡，都可以作为快速路径使用（推测，改天找机会测试一下）。

这里，我就拿一张Intel 82599为例，看看它能在快速路径中发挥哪些作用。

在datasheet中，参考**8.2 Device Registers — PF**的**Flow Programming Registers**部分，可支持编程的字段不多，如下图。

![](attachments/Pasted%20image%2020240708164118.png)


可能硬件与硬件间的区别，就在于能够支持可编程的Match/Action的字段有多少了吧。

可编程的Match/Action字段越多，就支持更复杂的流的加速，相对而言就能分担更多慢速路径的负担，提高整体的网络性能。毕竟，在快速路径上打满线速还是比较容易的，而在慢速路径上，大带宽的小包满线速就非常难了。


## Representor 的理解

在DPDK中，representor是一种抽象的设备，用于表示 物理或虚拟设备的逻辑端口。
在DPDK中，使用者可以通过抽象的representor来访问和操作底层设备，而不需要关心底层硬件细节。这些底层设备可以是物理网卡、虚拟机或者容器等。

dpdk中的 vf representor，简单理解，是**给控制面准备的vf的分身**。
DPDK representor提供了一套统一的接口和抽象，使得开发者可以方便地管理和操作底层设备的逻辑端口。通过DPDK representor，开发者可以进行逻辑端口的配置、状态查询、数据包收发等操作，从而满足不同应用场景的需求。


## Representor 的 优点
使用DPDK Representor可以实现以下好处：

1.数据平面性能得到显著提升：绕过vSwitch的处理，减少了数据包在处理链路上的延迟，提高了数据传输的速度和效率。

2.减少CPU开销：将数据平面的处理工作从主机CPU转移到专用的网卡硬件上，释放了CPU的计算资源，提高了主机的整体性能。3.简化虚拟网卡的操作：虚拟机直接连接到物理网卡上，不再需要复杂的虚拟化网络配置，降低了管理成本和配置复杂度。

然而，需要注意的是，使用DPDK Representor需要支持SR—IOV技术的硬件和驱动，同时还需要进行适当的配置和优化。此外，DPDKRepresentor在一些特定场景下可能会降低虚拟化环境的灵活性和可管理性。因此，在实际应用中，需要综合考虑系统需求和实际情况，权衡利弊。

# DPDK Representor 的理解

## 基础知识
### ethernet hub集线器

ethernet hub 只负责从一个端口上收到的 ethernet 包广播到 其他的所有的端口（所谓的广播，只是hub将该ethernet 包发送到其他的所有的端口，并不是指的是hub将包更改为广播报）。
即：在hub中不存在二层的mac表项，无法基于mac地址进行转发。


## 为什么需要 Representor

无法提前知道流量的流向，因此无法提前向硬件下发规则进行流量的 offload 转发，因此首包会上送到软件来进行处理。因此，在应用程序中需要具备接收以及将流量注入到各种各样的终端（比如，PF、VF等）的能力

如下所示，

![](attachments/Pasted%20image%2020240709143853.png)

DPDK程序绑定了PF，此时DPDK程序中是无法感知到 VF1 和VF2的存在，那么在软件中又期望将流量导流给 VF1或者VF2，因此 需要 在DPDK绑定PF的时候，创建对应的VF Representor （一个 VF Representor 在DPDK程序中也表现为一个PCIe的网口）
```bash
struct rte_eth_dev_info {
    ...
    uint32_t dev_flags; /**< Device flags */
    ...
};
DPDK 程序区分 VF Representor 和普通的 PF或者VF 是通过 检查 dev_flags 是否存在 RTE_ETH_DEV_REPRESENTOR 标记位。
```

## VF Representor 使用

参考：[dpdk Port Representor Tests](https://doc.dpdk.org/dts/test_plans/port_representor_test_plan.html)


![](attachments/Pasted%20image%2020240709145621.png)

```bash
./x86_64-native-linuxapp-gcc/app/dpdk-testpmd --lcores 1,2 -n 4 -a af:00.0,representor=0-1 --socket-mem 1024,1024 \
        --proc-type auto --file-prefix testpmd-pf -- -i --port-topology=chained

./x86_64-native-linuxapp-gcc/app/dpdk-testpmd --lcores 3,4 -n 4 -a af:02.0 --socket-mem 1024,1024 --proc-type auto --file-prefix testpmd-vf0 -- -i

./x86_64-native-linuxapp-gcc/app/dpdk-testpmd --lcores 5,6 -n 4 -a af:02.1 --socket-mem 1024,1024 --proc-type auto --file-prefix testpmd-vf1 -- -i
```

![](attachments/Pasted%20image%2020240709145715.png)

如上所示，DPDK程序1绑定了PF，同时创建了2个 VF Representor；那么就可以在DPDK1中通过 VF Representor 设置 VF的特性，以及导流到对应的 VF。


## interconnection

SR-IOV 中的 interconnection 是一个逻辑上的概念，可以理解为 网卡硬件的导流规则，通过该导流 规则，PF和多个VF之间可以互通、导流。其是DPDK应用程序通过 rte_flow 进行下发的。


# 应用
## intel E810的 DCF

参见：[dpdk-20.11-ICE Poll Mode Driver](https://doc.dpdk.org/guides-20.11/nics/ice.html)

### DDP 介绍

参见：[intel E810 DDP ](https://cdrdv2.intel.com/v1/dl/getContent/617015)

![](attachments/Pasted%20image%2020240704174037.png)

![](attachments/Pasted%20image%2020240709152342.png)

Dynamic Device Personalization (DDP)  可以理解为：

E810系列的网卡的驱动是 ice 驱动，然后固件fireware(可以理解为网卡硬件中的规则)支持的功能、规则、协议等 确实由 DDP来决定以及下发的。即，给网卡下发规则，则是 DDP来进行的。**通过DDP，可以下发 更加丰富的 RSS、FDIR 规则给网卡**。


如下所示：

![](attachments/Pasted%20image%2020240709152130.png)


注：DDP包不需要单独安装，Linux接管的设备，加载 ice驱动时会自动查询DDP包(ice.pkg)；对于DPDK接管的设备，启动时也会自动查找DDP包(ice.pkg)。

![](attachments/Pasted%20image%2020240709152701.png)

### DCF 介绍

![](attachments/Pasted%20image%2020240709153237.png)

我的理解：
即 被设置了 trust on 的 id =0 的 VF，如果含有DCF标记，就会被授予了更多的PF的功能，可以理解为一个PF了。
然后 通过在 **switch filter** 中配置规则，导流给指定的 VF 口。


DCF 使用范例：
```bash
1. Create a number of VFs (for example, four).
#echo 4 > /sys/bus/pci/devices/0000\:18\:00.0/sriov_numvfs

2. Enable VF0 as the trusted VF:
#ip link set dev enp24s0f0 vf 0 trust on

3. Bind VF0 by running testpmd (or DPDK application) with 'cap=dcf' devarg:
#testpmd -l 22-25 -n 4 -w 18:01.0,cap=dcf -- -i

下面可能需要配置 rte_flow 规则，然后将流量导流到其他的VF，然后通过在其他的VF上抓包可能可以抓到包。
```


#### 拓展

参考：[# ICE DCF Switch Filter Tests](https://doc.dpdk.org/dts/test_plans/ice_dcf_switch_filter_test_plan.html)

![](attachments/Pasted%20image%2020240709154308.png)

如上所示，物理口创建了4个VF，其中VF0设置为trust模式，启动 DPDK的时候，VF0含有DCF标记，此时被认为是PF，然后创建 VF1和VF2的 representor。另外，DPDK也会接管VF1和VF2。此时 VF0的驱动为 net_ice_dcf， VF1和VF2的驱动是  vfio-pci。


**add existing rules but with different vfs**
```bash
(1) DPDK 绑定 VF0, VF1, VF2; VF0 使用 DCF模式，设置VF1，vF2 rep。

./x86_64-native-linuxapp-gcc/app/dpdk-testpmd -c 0xf -n 4 -a 0000:18:01.0,cap=dcf,representor=[1,2] -a 0000:18:01.1 -a 0000:18:01.2 -- -i
testpmd> set portlist 0,3,4  # 设置绑定给的 VF0,VF1, VF2的port id为 0，3，4？那么 VF1 representor,VF2 representor 就是 1 和 2 ??
还是说，VF0和 VF1 rep，vF2 rep 的 port id是 0，3，4?? 
感觉像是后者。

注：representor=[1,2] 中的 1, 2 应该是 VF的 id. 由于 绑定的 trust 的 VF0，因此 representor 列表中不可以有0； 如果绑定的是 PF，那么 representor 列表中是可以有0的。



testpmd> set promisc 0 off
testpmd> set fwd rxonly
testpmd> set verbose 1
testpmd> start


（2）创建 含有同样的 匹配条件，action 为不同VF的规则： 


testpmd> flow create 0 ingress pattern eth dst is 68:05:ca:8d:ed:a8 / ipv4 src is 192.168.0.1 dst is 192.168.0.2 tos is 4 ttl is 3 / udp src is 25 dst is 23 / end actions port_representor port_id 0 / end

上面是将流量导流给 VF0？

testpmd> flow create 0 ingress pattern eth dst is 68:05:ca:8d:ed:a8 / ipv4 src is 192.168.0.1 dst is 192.168.0.2 tos is 4 ttl is 3 / udp src is 25 dst is 23 / end actions represented_port ethdev_port_id 1 / end

上面是将流量导流给 VF1 representor。

testpmd> flow create 0 ingress pattern eth dst is 68:05:ca:8d:ed:a8 / ipv4 src is 192.168.0.1 dst is 192.168.0.2 tos is 4 ttl is 3 / udp src is 25 dst is 23 / end actions represented_port ethdev_port_id 2 / end

testpmd> flow list 0

check both rules exist in the list.


(3) 发送数据包
sendp([Ether(dst="68:05:ca:8d:ed:a8")/IP(src="192.168.0.1",dst="192.168.0.2",tos=4,ttl=3)/UDP(sport=25,dport=23)/("X"*480)], iface="ens786f0", count=1)

检查到 port 0, port 3 and 4 收到了数据包。

```


#####  rte_flow 中的 `PORT_REPRESENTOR` 和 `REPRESENTED_PORT`

![](attachments/Pasted%20image%2020240709182444.png)

**匹配条件**：
![](attachments/Pasted%20image%2020240709163642.png)

如上所示，`port representor`  和 `represented port`  应该是代表了2个不同的流量方向。

**动作**：
动作，是站在 embedded switch的角度，将流量导流给哪一边。
![](attachments/Pasted%20image%2020240709164625.png)


**理解**：
`port representor（端口代表者）` 应该是 DPDK绑定的 PF，即设置了含有 VF representor的设备。
`represented port（被代表的端口）` 应该是 连接到 embedded switch 中的实体，位于通向ethdev 的“线路”的另一端。

> 注意： REPRESENTED_PORT 和 `PORT_REPRESENTOR` 在 DPDK 21.11 才存在。20.11 都不存在，最好是使用 更新的 DPDK，比如 DPDK 22.11 来进行测试。



#### 小结

如上  intel的 DPDK 的 DCF-PMD 的介绍，允许PF创建多个VF，然后设置VF0为 DCF  VF, 此时可以将其当做 PF，然后配置 rte_flow 规则 （比如: 三层规则，四层规则「如dstport 为 80」）将流量 导流到 其他的 VF中。 **拓展了 之前  E810 网卡的 硬件 switch filter 只能基于dst-mac 转发的 问题**。


### 应用

#### 需求

Intel E810 系列网卡， 如果期望流量分叉，即DPDK绑定VF，引流特定的流量（比如：dstport 80）的流量到DPDK，其他的流量还是继续通过 PF到内核协议栈。
那么就不太容易搞定，因为 SR-IOV中PF-->VF的导流默认是通过 dst-mac  （==即上图中的switch== ，至于后续的RSS/FDIR 都是在 switch 之后了）进行导流的，而正常 VF 和 PF的 mac 地址是不一样的，而上诉的期望是通过 dst-port 进行导流。
> 即： 报文的 dst mac = vf的mac，或者 mac为广播地址，才会导流给指定的VF。


#### 方法一：VF和PF配置同网段的不同IP

VF和PF配置同网段的不同IP地址，DPDK中实现VF的IP地址的ARP协议栈，VF和PF的流量隔离是通过IP来隔离的。其实通过IP来隔离，在底层也是反应到 dst-mac来隔离。


##### 优缺点

**缺点**：
配置了2个IP，业务流量一个  IP，本机控制流量一个IP，不太友好。

#### 方法二：intel的 turst VF设置DCF

如上所示，介绍。


## 博通网卡

参考：[DPDK 23.11 -# BNXT Poll Mode Driver ](https://doc.dpdk.org/guides-23.11/nics/bnxt.html)

![](attachments/Pasted%20image%2020240709180302.png)

![](attachments/Pasted%20image%2020240709175804.png)


如上所示，博通网卡也支持设置 Trust VF，最好是在DPDK的应用程序绑定VF之前，将VF设置为trust 模式。
另外，也支持将 trust VF 视为一个 PF，然后设置 vf representor.
```bash
-a DBDF,representor=[0,1,4]

注：DBDF 即 PCIe号。
```

# 其他
## DBDF
首先在x86系统中PCIe支持256个Bus， 每条Bus支持32个Device， 每个Device支持8个Function，所以PCIe设备关键信息组成为：

DBDF(Domain,Bus,Deivce,Function)。

### PCIe的拓扑和Linux的PCIe ID

```bash
lspci –vt
```

![](attachments/Pasted%20image%2020240708170000.png)

所以使用 `lspci  -vt`  可以查看PCIe拓扑；根据拓扑就可以看出，接入几个板卡，每个板卡下面接入了多少设备.


```bash
dmidecode –t slot 命令查看PCIE Slot的信息
```

# 参考
```bash
# DPU如何加速云原生容器网络？
https://blog.xuegaogg.com/posts/1955/

https://blog.csdn.net/leiyanjie8995/article/details/121341828


# BF2 swithdev representor 方案介绍
https://blog.csdn.net/leoufung/article/details/121046338?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ECtr-1-121046338-blog-121341828.235%5Ev43%5Epc_blog_bottom_relevance_base1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ECtr-1-121046338-blog-121341828.235%5Ev43%5Epc_blog_bottom_relevance_base1&utm_relevant_index=2

# Switch Representation within DPDK Applications
https://doc.dpdk.org/guides-22.11/prog_guide/switch_representation.html
```