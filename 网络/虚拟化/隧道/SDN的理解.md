# 前言
云数据中心对网络提出了**灵活、按需、动态和隔离**的需求，***SDN的集中控制、控制与转发分离、应用可编程***这三个特点正巧能够较好的匹配以上需求。

# 为什么引入SDN
云计算数据中心利用自身所拥有的计算、存储、网络、软件平台等资源向租户提供Iaas虚拟资源出租。典型的IaaS资源包含云主机（虚拟机）、对象存储、块存储、VPC专用网络（VPC，Virtual Private Network虚拟私有云）、公网IP、带宽、防火墙、负载均衡等产品。

在网络层面，假设暂不考虑公网IP、带宽等衍生网络产品，仅是云主机，网络上最基本的技术要求就是可迁移性和隔离性。
- 可迁移性，通常是指云主机在数据中心具备自动恢复能力。当云主机所在宿主机（物理服务器）出现宕机时，云主机能够自动迁移至另一台正常运行的物理服务器上且IP保持不变。
- 隔离性通常可以分为两个层面，一是不同租户间的网络隔离，鉴于安全考虑，不同租户间内部网络不可达；二是同一租户内部不同子网（vlan）间的隔离，为业务规模较大的租户提供的多层组网能力。

因云主机迁移IP不能变，进而要求网络需处于二层环境中，早期云数据中心在组网上通常是采用大二层技术。大二层技术，简单理解就是整个数据中心是一个大二层环境，云主机网关都位于核心设备上。

一想到二层环境，肯定离不开广播风暴，也离不开遏制广播风暴的生成树协议。全网都用生成树协议，势必会阻塞较多的网络链路，导致网络链路利用率不足。为了解决利用率不足的问题，思科VPC（这个跟上文的VPC不一样，virtual port channel虚拟端口转发）技术和华为华三的IRF堆叠技术应运而出。简单的理解，上面两种技术都是对生成树协议的欺骗，最终使被生成树协议阻塞链路转变为可使用状态，提升链路使用率。

结合大二层技术使用的租户隔离方式有两种常用的，一个是vlan隔离，一个是VRF（Virtual Routing Forwarding虚拟路由转发）隔离。
- 若是采用vlan隔离，通常需要把云主机网关终结在防火墙上，这样才能满足租户间安全隔离的需求。这种模式下，一般是一个租户对应一个vlan；针对同一租户有多子网的需求，则需要在网关设备防火墙上通过较为复杂策略来实现。
- 若是采用VRF隔离的方式，通常是把云主机网关终结在高端交换机或者路由器上，一个租户对应一个VRF。针对同租户有多子网的需求，则是一个VRF+多个vlan的模式。

>受限于vlan/VRF规模，无论是“大二层+vlan”还是“大二层+VRF”，都存在云数据中心租户数量不超过4096个的限制，同时也不允许租户间的IP地址段冲突。

#  SDN在云数据中心的系统架构
SDN的3+2架构模型，从上到下分为应用层、控制层和转发层。以控制层为基准点定义了两个外部接口，其中，向上为应用提供自定义业务功能的API称为北向接口，向下控制使用底层网络资源的API称为南向接口。常用的北向接口标准是Restful，常用的南向接口标准是Openflow。

SDN的3+2架构模型相信大家都不陌生。SDN在云数据中心跟云管理平台（以Openstack为例）整体融合考虑时，比较常见的系统架构如下所示。针对下图进行几个说明，说说为什么常用这种模式：

![](attachments/Pasted%20image%2020230506203149.png)

## 关于系统层级的划分
推荐的SDN系统层次划分中，云数据中心运营管理平台和Openstak统一被定义为应用层，独立的SDN控制器设备构成控制层，底层网络设备构成了转发层。

在业界关于Openstack数据中心系统层级，除了图中所示，还有另外一种划分方式。在另一种划分方式中，是把Openstack的Neutorn当成是控制层，由neutron直接对接底层网络设备。在服务器规模较大的云数据中心，尤其是采用虚拟交换机时，控制器跟网络设备（虚拟交换机）之间的交互流量是比较大的。在实际部署中，通常会引用商业版的SDN控制器。商业版的SDN控制器与Neutron的对接，主要体现在于Neutron插件上。



# 参考
```c
https://www.sdnlab.com/19236.html

```