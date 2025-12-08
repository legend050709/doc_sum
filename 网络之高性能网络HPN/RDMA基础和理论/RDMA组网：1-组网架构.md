```table-of-contents
```

# Scale-up与Scale-out

在计算领域，我们经常会听到Scale这个词，中文大多数翻译成”扩展“，其含义是通过对现有系统的”扩展“来处理更大规模的工作。
然而，对于扩展来说，我们有两个不同的思路。一个是Scale-up，另一个是Scale-out。对于Scale-up，比较主流的翻译是向上扩展，垂直扩展；对于Scale-out，通常翻译成横向扩展，水平扩展等。

## Scale-up
Scale-up（向上/垂直扩展）：通过增加单个系统的资源（如处理器速度、内存或存储容量）来**提升单个系统的性能**。这意味着让一个单一的系统变得更加强大。

```text
To scale vertically (or scale up/down) means to add resources to (or remove resources from) a single node in a system, typically involving the addition of CPUs or memory to a single computer.  
Such vertical scaling of existing systems also enables them to use virtualization technology more effectively, as it provides more resources for the hosted set of operating system and application modules to share. Taking advantage of such resources can also be called "scaling up", such as expanding the number of Apache daemon processes currently running. Application scalability refers to the improved performance of running applications on a scaled-up version of the system
```

`Scale-up`, 即 `Scale vertically`，纵向扩展，向上扩展。
称为单节点系统，指系统中只包括一个有效节点（如果需要HA时，可以将两个单节点以System Replication形式构成单节点的HA架构）。这种架构的系统只具有垂直扩展能力，当需要扩展系统时，通过在节点上增加更多的CPU、内存和硬盘来扩大系统的能力。

`Scale-up`通过购买性能更好的硬件提升系统的并发处理能力；比如：我们向原有的机器增加CPU、内存数。

## Scale-out
Scale-out（横向/水平扩展）：通过增加更多的相同或相似配置的系统来分散工作负载。这意味着添加更多的独立系统来共同完成任务。


```text
To scale horizontally (or scale out/in) means to add more nodes to (or remove nodes from) a system, such as adding a new computer to a distributed software application. An example might involve scaling out from one Web server system to three.  
As computer prices have dropped and performance continues to increase, high-performance computing applications such as seismic analysis and biotechnology workloads have adopted low-cost "commodity" systems for tasks that once would have required supercomputers. System architects may configure hundreds of small computers in a cluster to obtain aggregate computing power that often exceeds that of computers based on a single traditional processor. The development of high-performance interconnects such as [Gigabit Ethernet](https://zhida.zhihu.com/search?content_id=234689243&content_type=Article&match_order=1&q=Gigabit+Ethernet&zhida_source=entity), [InfiniBand](https://zhida.zhihu.com/search?content_id=234689243&content_type=Article&match_order=1&q=InfiniBand&zhida_source=entity) and [Myrinet](https://zhida.zhihu.com/search?content_id=234689243&content_type=Article&match_order=1&q=Myrinet&zhida_source=entity) further fueled this model.  
Such growth has led to demand for software that allows efficient management and maintenance of multiple nodes, as well as hardware such as shared data storage with much higher I/O performance. Size scalability is the maximum number of processors that a system can accommodate.
```

`Scale-out` 即`Scale horizontally`，横向扩展，向外扩展 。称为集群系统。指由多个节点组成的系统，这种系统的扩展主要以水平扩展方式（指增加节点的方式）来进行。
Scale-out 通过将多个低性能的机器组成一个分布式集群来共同抵御高并发流量的冲击。
比如向原有的web、邮件系统添加一个新机器。

## 范例

举例来说，在构建网络时：
如果使用机框式交换机，并通过增加更多的板卡来提升网络接入能力，这就属于Scale-up模式。
![](attachments/Pasted%20image%2020250312103612.png)

如果采用多台相同接口密度的盒式交换机，并通过CLOS架构或其他分布式网络设计来增加网络接入能力，这就属于Scale-out模式。

![](attachments/Pasted%20image%2020250312103639.png)


当然很多时候Scale-up和Scale-out也可以配合实现更大规模（或更高效）的组网。
![](attachments/Pasted%20image%2020250312103753.png)

## Scale In 和 Scale down
`Scale In「Scale out 的反方向」`，`Scale down「Scale up 的反方向」`;

无论是`Scale Out`，`Scale Up`，`Scale In「Scale out 的反方向」`，`Scale down「Scale up 的反方向」`；实际上就是一种架构的概念，这些概念用在存储上可以，用在数据库上，网络上一样可以。

## Scale-up和Scale-out如何选择
简单来说，Scale-up通过对现有资源的规模扩大来实现扩展，而Scale-out将通过增加相同规模的现有资源来实现扩展。

例如，某物流公司有一艘轮船用于海运，可以一次性运送600个集装箱或者尺寸小于100x25x13米大小的整体货物。如果随着业务的拓展，需要对货运能力进行扩展，那么就会有两种思路，一种是将这艘船换成更大规模的货船，这就是Scale-up方案；另一种是再增加一艘相同规模的货船，这就是Scale-out方案；

![](attachments/Pasted%20image%2020250312110237.png)

### `Scale-up`的优缺点

一般系统设计初期会考虑使用`Scale-up`，因为足够简单，堆砌硬件解决即可，但当系统并发超过单机的极限时，就要使用`Scale-out`了。

#### 优点
通常来说，Scale-up在扩展速度，简易程度，以及成本上相对Scale-out方案会更具优势。

以个人电脑和服务器为例，为了确保内存容量的Scale-up能力，个人电脑和服务器的生产商会预留扩展内存DIMMs，这些DIMMs在一开始出厂的时候并没有安插内存条，而是到了用户可能出现的Scale-up需求后，由客户自行购买和安装。除了内存条容量的扩展，其他的例如PCIe外设种类扩展，原本你的PC可能用的是集成显卡，但最近你希望能够玩一些大型的3D游戏，集成显卡的性能有限，此时如果主板上还有空闲的PCIe槽位，那么扩展显卡性能最快的方案就是买一张独立显卡，插入到PCIe槽位中即可实现。而对于处理器扩展，如果是服务器，有额外的处理器座，那么扩展方式与独立显卡扩展方式相同。如果没有额外的处理器座，那么只能更换一个性能更好的处理器，例如更高的频率，更多的CPU核。其他的扩展，例如存储扩展，网络带宽扩展等等，都大同小异。对于这些扩展进入系统的硬件，系统软件基本不用做什么配置就能够直接使用这些扩展硬件。

但相对Scale-up来说，Scale-out可能没有现成的软件能够把多台系统的能力有效的利用起来，可能需要花费更多的时间才能完成扩展步骤。

#### 缺点
Scale-up有没有什么不足呢？

（1）首先是大多数Scale-up扩展硬件时需要让系统下电。
当然也有部分硬件支持热插拔功能，在系统不下电的情况下扩展。然而对于Scale-out来说，大多数情况下都不需要对原本的系统进行下电操作。

（2）Scale-up的规模相比Scale-out而言要小许多。
例如目前NVLink所组的Scale-up域，其部署规模还没有超过1K。反观通过以太网，IB或者RoCE组成的Scale-out数据中心，其组网规模要大上不少。

![](attachments/Pasted%20image%2020250312110740.png)


（3）故障的隔离较麻烦。
Scale-up方式虽然能够使得系统通过一些相对简单的方式，获得更大的容量，更高的性能，但是由于硬件系统耦合相较Scale-out要紧密的多，对于一些故障可能会出现隔离不充分，甚至导致整个系统瘫痪的可能。这在Scale-out的情况下则有效的避免了这个问题。就好比有两艘货轮，任意一艘出现问题，都不影响另一艘货轮的使用。

### `Scale-out`的优缺点
`Scale-out`虽能突破单机限制，但也会引入一些复杂问题。比如，
1> 若某节点故障，如何保证 HA？
2> 当多个节点有状态需要同步时如何保证状态信息在不同节点的一致性？
3> 如何做到使用方无感知的增加和删除节点？
这些问题的存在与解决也伴随着分布式系统不断完善发展。

### 如何选择
究竟我们应该怎么选择呢？这可能要具体看我们的应用特点。
某些应用，例如HPC或者是AI大模型计算，其中部分计算和通讯量较大的过程，可能需要Scale-up方式提供扩展能力。而对于计算和通讯耦合不大的过程，可以使用Scale-out的方式进行扩展。这也是Nvidia在其DGX GH200集群中，Scale-up域采用了NVLink，而Scale-out域采用了IB的原因。

又例如另一些应用，其I/O相关的比重较高，仅通过Scale-up方式进行扩展，其性能瓶颈并不能有效解决。这种情况下通过Scale-out可能是最佳的解决。

# 通算网络和智算网络

 数据中心网络总体上可分为两大类。
 一类是**通算网络**，即传统的数据中心网络，主要用于支持传统的计算任务和应用，如企业的IT系统、网站托管、电子邮件服务等。
 
 另一类是**智算网络**，专门用于支持人工智能（AI）和机器学习（ML）任务。这类网络需要更高的计算能力和更低的延迟，以处理大量的数据并执行复杂的计算任务。

# 智算网络中的 Scale-up和Scale-out网络
 目前主流的用于ML的智算网络和通算网络，在架构上存在重大的差异，即，**通算网络一般来说只有一张网**，而**智算网络，会存在两张网**，如下图所示。
 
 ![](attachments/Pasted%20image%2020250312111407.png)
 
两张网络的搭配使用，才造就了当今的AIGC大模型。在智算中心的两张网中；
一张是通过ETH/IB实现GPU之间的RDMA功能的网络，即所谓的前端网络，通常称作**Scale-out**网络。
一张是GPU之间高速互连（比如通过NVLink），可以实现POD内跨GPU之间的内存的读写，即所谓的后端网络，即通常说的**scale-up**网络。


## 基于以太网构建智算网络中更大规模的Scale out集群
网络并不是简单地将GPU互联，组成更大规模，就达成“集群化算力”的效果。网络连接好比高速公路，并不是高速通了就可以畅通无阻，规划不合理、车道不足、调度不合理，都会出现拥堵，节假日高峰出行就让人不省心，网络也是如此，在这么庞大的GPU互联中，带宽大小、拓扑设计、负载均衡、任务排布等等，都会影响GPU并行计算中的通信性能。

**更重要的是，今天的大模型训练是基于并行计算范式，一个训练任务是计算-通信-计算这种周期性迭代的过程，所有GPU 在一轮计算迭代后都必须同步参数和梯度才能进行下一轮的计算，集群中任何一处有网络拥塞或者故障都会影响整体训练的性能，具有很强的木桶短板效应**。 所以稳定的高性能网络互联成为智算集群的最核心诉求。

==为传统CPU业务设计的数据中心网络架构针对的是大规模分布式计算，已经不能适应大规模并行任务的智算集群==。
为此，阿里云在去年设计了HPN7.0架构，其论文被顶会SIGCOMM录取，成为网络顶会历史上首篇AI智算网络架构论文，成为业界标杆，为Scale out的以太网技术路线树立旗帜。目前基于以太网来构建大规模智算集群，基本上成为业界的共识，北美的meta、xAI都相继发布了基于以太网的10w级别集群。

## 智算网络中Scale up网络的发展
GPU集群演进的另外一个热点话题是Scale up。各大GPU 厂商相继发布了AI rack级产品路标，Scale up范围由目前的8卡增加到64、72卡，甚至将来到576或者更多，所以Scale up网络怎么做，基于什么协议来做，是封闭还是开放，大家都非常关心，这也是UEC和UAL备受关注的原因。

### GPU Scale up
#### 什么是 GPU Scale up 
**不少人以为Scale up是机内互联，这是一种误解。**
在8卡系统的时代，因为8卡在一个OS内部，所以确实是机内互联。
**当NVL36、72这种AI rack的形态出现后，NV link 就不再是“机内互联”**，而是一种新型的节点间网络互联，为了和目前的 RDMA 高性能 Scale out 网络区分，行业内还继续采用 “Scale up” 这个叫法。

阿里云给出了一个定义：Scale up就是在一定范围内、在成本和互联技术约束下实现的超高带宽互联。这个超高带宽互联的范围固定，并且带宽是Scale out的数倍以上，可以在协议层面优化来支持内存语义。**以NVL72为例，实际上是18台服务器通过9台Scale up交换机连在一起的网络域，只不过是在这个域内的带宽9倍于Scale out的大的带宽（7.2Tbps vs 800Gbps）**，此外还支持了内存操作语义，为了区分，我们依旧称其为Scale up，但实际上是一种更大带宽的新型 scale out 网络。

#### AI rack和于小型机、大型框式交换机/路由的区别
**类似NVL72这种"AI rack"本质上是多台服务器组成的一个小型集群，而不是一台服务器**。
不同于小型机、大型框式交换机/路由等，都是运行一个主控OS，由于系统复杂，故障率高，已经退出了历史舞台。其中核心组件一旦出现故障，整个 rack 系统都会fail，也因为这个原因（外加成本，运维复杂度等）行业内在很多年前就走向了开放解耦架构，采用更小的 x86 服务器 or 白盒交换机 Scale out。

![](attachments/Pasted%20image%2020250312113236.png)

如上图所示，NVL72 并不是一台大服务器，实际上是为了提供更大带宽互联的一个小型化浓缩的集群，由18个服务器和9 个交换机通过高速铜线互联而成，其中任何一个计算或网络节点出现问题，都不会影响其他服务器节点，整个NVL72的其他部分依然会正常运行，这一点也是类似NVL72这种“AI rack”与其他小型机、大型框式交换机/路由等的本质区别。

历史上出现的小型机、大型框式交换机/路由等，都是运行一个主控OS，其中核心组件一旦出现故障，整个系统都会宕机，再加上封闭系统和高昂的成本，行业内很多年前就抛弃了这个方向，走向了开放解耦架构，采用更小的 x86 服务器 or 白盒交换机，通过分布式集群的方法来构建系统。历史不会倒退，类似NVL72的AI rack必然采用分布式方法，成为一个小集群而不是一台服务器。

#### GPU Scale up的未来发展

随着大模型训练和推理对算力性能需求的持续提升，以及性价比的持续驱动，Scale up域会越来越大，也就是说 Scale up 集群的规模会越来越大，从单 rack 到双rack，再到跨多个rack将成为必然趋势，当 Scale up 集群规模达到千卡级别，和传统 Scale out集群就已经具备很多共同点了，这个时候如何设计 GPU 互联架构，需要智算网络的下一轮革新。

Scale up网络大体上可以分成2个技术方向。一个是封闭的私有技术方向，典型代表比如NV、Google（NVLink和TPU互联）。另外一个是基于Ethernet的开放技术方向，这个方向以各大互联网和云计算公司自研GPU（微软、Meta、Tesla等）为代表，包括一些大的GPU芯片公司。最近大家都知道的消息是，某GPU芯片大厂，在谨慎评估后选择了Ethernet作为其下一代GPU Scale up的路线，通过一层互联即可以做到256 GPU的Scale up域。

说起 GPU Scale up 的行业生态，必然会提 UAL，UAL 联盟也已经成立有段时间，据说内部也调整了好几次，从最开始的采用 PCIE 交换机作为 Scale up switch 到转向 Ethernet 作为网络底层，联盟核心成员也有调整，网络芯片龙头老大博通退出，而一向不加入开源组织的 AWS 反而加入，让 UAL 蒙了一层神秘的面纱，标准制定道路漫长，但是众多GPU芯片公司却等不及了，采用可规模落地的 Ethernet 已经成为首选，包括上面说的某GPU芯片大厂都开始转向Ethernet 了。

Ethernet有超大带宽技术和强大的生态支撑，目前UEC、高通量以太网等开放组织还在针对Scale up进行协议的改进来实现低时延、在网计算等核心功能，以及针对内存语义进行优化，所以众多GPU芯片公司都选择了以太网作为Scale up网络的首选技术路线，同时，基于 Ethernet 的Scale up 方案为未来的数据中心网络持续演进，为 Scale up 和 Scale out 二网融合奠定了重要基础。


## cluster和superpod
根据Nvidia的信息，在这里我们澄清一个概念，即，cluster和superpod。
superpod可以视为一个GPU之间通过NVLink高速总线进行互连的一个域，即，在这个superpod域内的所有的GPU是全带宽互连，即scale-up网络。

cluster则是由所有GPU服务器组成的一个网络群的总称，一个cluster可以有多个superpod组成，怎么组成呢，那就是通过scale-out网络来连接。

## 两张网中Scale-up和scale-out的网络规模
对于N系的GPU服务器来说，目前的Scale-up的网络规模一般可以认为是scale-out网络的十倍。从下图中 GB200 超级芯片的接口上可以看出，NVLink、InfiniBand、Ethernet 三种网络的容量配比为，NVLink 网络 14.4Tb/s，InfiniBand 网络 1.6Tb/s，Ethernet 网络 400Gb/s。三种网络的端口带宽之比为 NVLink : InfiniBand : Ethernet = 36 : 4 : 1。

![](attachments/Pasted%20image%2020250312111744.png)

其中，scale-up的Nvlink网络是RDMA的IB的带宽近10倍。

> 注：需要额外说明的是，一般主要用于CPU之间进行数据存储智能网卡的网域，在实际的应用中，也归于scale-out网络。

## 智算网络中为何要区分Scale-out与Scale-up网络
Scale-out与Scale-up网络的设计初衷和应用场景大相径庭，这正是我们需要区分它们的原因。

首先，随着AI大模型的兴起，模型规模日益庞大，几乎“上不封顶”。这对计算资源提出了前所未有的挑战。单台GPU服务器已难以承载如此庞大的数据量，因此，并行处理成为必然选择。但并行处理并非没有代价，它带来了通信开销、分割复杂性和编程难度的增加。特别是对于像Transformer这样的模型，其注意力机制和前馈网络对内存和计算资源的需求极高。

为了应对这些挑战，我们设想了一种理想情况：拥有一个超级强大的GPU芯片，能够单独处理整个大模型，从而省去并行切割的麻烦。然而，这仅是理想状态，现实中并不可行。因此，我们采取了折衷方案：将大模型分解为两部分。一部分是高频数据交互的部分，如张量并行和专家并行，这些部分需要高速、低延迟的互连网络来处理，以减少通信成本。**这就是Scale-up网络，也称为Load-Store或内存语义网络，它追求极致的性能**。

另一部分则是相对独立的并行数据，如流水线并行和数据并行，这些可以通过更经济、更灵活的方式来实现，即Scale-out网络。它利用现有的技术体系，如以太网，并进行适当优化，以低成本满足性能需求。在Scale-out网络中，RDMA（如RoCE）技术扮演了重要角色，尽管它在某些方面模拟了内存访问模式，但在**处理大量小内存读写时效率并不理想**，因此不被视为纯粹的**内存语义网络**。

通过这样的网络划分，我们既保证了AI大模型训练的性能需求，又有效控制了成本。Scale-up网络专注于高性能，而Scale-out网络则侧重于经济性和灵活性，两者相辅相成，共同推动了AI技术的进步。

## 智算网络中Scale-out与Scale-up网络的区别
在大规模模型训练中，两者虽都负责GPU间的数据传输，但时延表现却大相径庭。

### 动态时延和静态时延

网络时延，简单来说，就是数据在网络上传输所需的时间。它可以细分为静态时延和动态时延。

静态时延，像是网络硬件的“天生”属性，包括互联、转发和交换等过程，它相对稳定，主要由物理布局和设备性能决定。

动态时延则像是个“情绪化”的孩子，会随着网络负载、带宽利用率等因素波动，随时可能变化。例如，通过UEC优化以太网，主要就是减少了这种“情绪化”的时延。

### scale-up：追求纳秒级极致

Scale-up网络，或称总线域网络，是追求极致性能的典范。在这里，GPU能像访问本地内存一样直接读写其他GPU的存储器，这要求时延必须极低。想象一下，如果GPU的主频超过1GHz，每个时钟周期都不到1纳秒，那么内存访问的时延自然也要跟上这个节奏。因此，Scale-up网络的时延目标通常是微秒级以下，甚至更低。

为了实现这一目标，Scale-up网络设计时需要紧密贴合特定业务需求，摒弃传统网络中的传输层和网络层，采用信用机制和链路层重传来确保可靠性。同时，面对高速SerDes技术的挑战，如PAM4调制和DSP架构下的112Gbps、224Gbps技术，静态时延的控制变得尤为关键。现有的RS(544, 514) FEC方案在高速下可能不再适用，需要探索新的FEC方案来进一步降低时延。

### scale-out：毫秒级时延的灵活之选

 Scale-out网络更加灵活多变，它借鉴了传统网络的分层架构，如OSI模型，具有清晰定义的**传输层和网络层**，以支持多样化的通讯和数据传输需求。这种灵活性虽然带来了时延上的妥协，但也确保了网络能够适应更多样化的应用场景。

在Scale-out网络中，端到端的时延通常控制在1至10毫秒之间，以确保用户感受到系统的即时响应。这个上限是基于人的感知能力，超出此范围可能会让用户感觉到系统迟缓或不响应。

对于AI/HPC等计算密集型业务来说，虽然不要求极致低时延，但稳定的低时延表现仍然是提升性能的关键。Scale-out网络通过借用传统网络的产业链资源，如交换机和光模块，并在此基础上进行性能优化，如UEC和GSE等，以降低动态时延。然而，由于网络架构的复杂性，静态时延仍然相对较高。



### 小结

Scale-up网络和Scale-out网络在时延方面的追求截然不同。Scale-up网络致力于将RTT从亚毫秒级降低到亚微秒级，实现极致的低时延性能；而Scale-out网络则更注重灵活性和成本效益，以毫秒级时延满足多样化的业务需求。这种时延上的差异，正是两者在AI/HPC等领域中扮演不同角色的根本原因。

> 注：亚毫秒级别，指的是小于1毫秒（1 ms = 1000微秒）的时间范围；通常在0到999微秒之间。
> 亚微秒级别指的是小于1微秒（1 μs = 1000纳秒）的时间范围；通常在0到999纳秒之间。

### 将scale-out与scale-up网络合二为一的困难

scale-out与scale-up网络，作为两种截然不同的技术路径，它们在设计理念、应用目标以及技术实现上都有着根本性的区别，因此，将它们合成为一张网络并不现实。

scale-out网络，根植于传统数据中心网络，其设计初衷在于连接分布在各个地理位置的节点，以实现远程的、高效的通讯和信息交换。它擅长处理长距离传输、异构设备互联以及多样化的业务通信需求，与上层业务和应用的关系相对灵活。

而scale-up网络，则是一种全新的技术理念，它追求的是通过提升单一设备的性能来增强整个系统的能力。这种网络设计得更紧凑，旨在在有限的物理空间内集成更多资源，从而大幅提升系统性能，并且与业务逻辑紧密耦合。

在人工智能（AI）和通用人工智能（AGI）时代，智算网络的需求日益增长，但无论是**简单地增强传统数据中心网络的load-store能力，还是试图通过load-store技术来扩展网络**，都无法满足scale-up网络的独特需求。这是因为两者在设计之初的出发点就存在本质的不同，导致它们在技术实现、性能表现和成本效益上都有着显著的差异。

从业务逻辑的角度看，**scale-up网络（如NVLink）更接近于load-store语义网络，强调直接、快速的内存访问**；而**scale-out网络（如InfiniBand）则基于消息语义，注重灵活性和可扩展性**。尽管在某些技术指标上（如224G的传输速率），两者可能看起来相似，但这仅仅是一个巧合，并不能说明它们可以融合或互相替代。

因此，我们可以得出结论：scale-out与scale-up网络，由于它们在技术理念、应用目标以及业务逻辑上的根本差异，不会也不应该被合成为一张网络。它们各自在特定的场景下发挥着不可替代的作用，共同推动着计算网络技术的发展（但scale-up的物理层、链路层有可能和scale-out一样选择以太，这个咱们后面有机会单起文章聊）。

## 构想未来网络的融合架构
如果未来更大规模的Scale up选择Ethernet作为路线后，就可以实现Scale up和Scale out的融合，如下图所示，**做到效率更高、成本更低的架构。**

![](attachments/Pasted%20image%2020250312114343.png)

Scale up范围内进行大带宽的TP、EP、CP等通信，多个Scale up域通过Scale out互联，进行DP、PP等通信，跨Scale up实现合理的带宽收敛即可。同时，独立Scale out网卡+网络的成本也不容小觑，如果将 Scale up 和Scale out 的以太网融合为一张网，通过将不同的Scale up域进行Scale out互联组网，不但少了一张网络和网卡的投入，在运维、扩展上也将更加统一高效。



# 参考
```bash
# 大规模 RDMA AI 组网技术创新：算法和可编程硬件的深度融合 【系列文章】
https://mp.weixin.qq.com/s/u6JiJWRruVWR4wnRgilZtg


```