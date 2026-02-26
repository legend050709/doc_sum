```table-of-contents
```
# IBA架构(Infiniband Architecture)
# Infiniband协议
## 数据包类型
## 数据包格式
![](attachments/Pasted%20image%2020250801115101.png)

通常，一个InfiniBand数据包的结构包含以下几个部分（从包头到包尾的顺序）：

（1）本地路由报头 (Local Routing Header, LRH)： 所有数据包都必须包含本地路由报头。LRH包含子网内路由所需的信息，例如源LID (SLID) 和目标LID (DLID)。
    
（2）全局路由报头 (Global Routing Header, GRH)： 并非所有数据包都包含GRH。以下情况下需要GRH：
- 需要路由到不同InfiniBand子网的数据包。
	
- 所有多播 (Multicast) 数据包，无论目的地是否在同一子网。
	
- 除了子网管理数据包 (SMP) 之外的任何数据包都可以选择包含GRH。GRH包含全局路由信息，例如源GID (SGID) 和目标GID (DGID)。
        
（3）传输报头 (Transport Header, TH)： 只有IBA数据包才包含传输报头。TH包含与传输协议相关的信息，例如QP编号 (QPN)、操作码 (Opcode) 和数据包序号等。
    
（4）有效载荷 (Payload)： 包含实际的应用数据。
    
（5）循环冗余校验 (CRC)： 用于检测数据包在传输过程中是否发生错误。InfiniBand使用两种类型的CRC：

- 不变CRC (Invariant CRC)： 对于IBA数据包，在有效载荷之后有一个固定长度的不变CRC。
    
- 可变CRC (Variant CRC)： 所有数据包（包括IBA数据包和原始数据包）的末尾都有一个可变CRC。可变CRC的长度取决于链路速度。

### 数据包结构总结

|数据包类型|报头结构|CRC类型|
|---|---|---|
|IBA数据包|LRH [GRH] TH Payload 不变CRC 可变CRC|双CRC|
|原始数据包|LRH [GRH] Payload 可变CRC|单CRC|


![](attachments/Pasted%20image%2020250801115239.png)

## 链路层
### LRH
![](attachments/Pasted%20image%2020250801121258.png)

本地路由头 (LRH) 是一个 8 字节的报头，包含在InfiniBand子网内进行本地路由所需的字段。LRH位于每个InfiniBand数据包的开头，数据包的末尾是可变循环冗余校验 ( Variable CRC, VCRC)。

![](attachments/Pasted%20image%2020250801115306.png)

### 虚拟通道 (Virtual Lanes, VL) 机制
#### 特殊虚拟通道：VL15 (Special Virtual Lane: VL15)
#### 数据虚拟通道 (Data Virtual Lanes)

### 服务等级 (Service Levels, SL)

### 链路级流控 (Link-Level Flow Control)


## 网络层
### 交换与路由
IBA支持两级拓扑划分：

（1）子网 (Subnet)：
子网是IBA网络的基本组成单元。数据包在子网内部的转发称为交换 (Switching) ，由IBA交换机完成。数据包在子网内的传输路径由其注入到网络中的位置（由本地路由头 (LRH) 中的源本地标识符 (SLID) 字段标识）以及目标端口的本地标识符 (DLID) 和服务等级 (SL) 字段共同决定。
    
（2）全局网络 (Global Network)： 
多个子网可以通过路由器互连形成更大的全局网络。数据包在不同子网之间的转发称为路由 ( Routing) ，由路由器完成。路由器可以是符合IBA规范的路由器，也可以是支持其他协议（如IP）的路由器。IBA规范并未定义数据包在全局网络中经过的一系列子网的完整路径，但全局路由头 ( GRH) 中提供了一些关键字段，例如源全局标识符 (SGID)、目标全局标识符 (DGID)、流量类别 (TClass) 和流标签 (FlowLabel) ，供路由器进行路由决策。此外，路由器还可以使用其他报头中的字段，例如LRH中的SL字段，来确定到TClass的映射关系。无论采用何种转发机制，IBA都要求路径相对于SGID和DGID对称。也就是说，如果存在从SGID到DGID的有效路径，则交换SGID和DGID的值后也必须存在有效的反向路径。

#### 路由
路由过程：
- 当数据包在子网内传输时，其SLID和DLID字段保持不变。
- 当数据包在子网之间传输时（即通过路由器），路由器会将SLID更新为其自身的LID，并将DLID更新为下一个路由器或最终目标的LID。



### 全局网络特征 (Global Network Characteristics)
本节描述了完全通过IBA路由器互连的全局网络的特征。与非IBA技术互连的全局网络可能表现出部分或全部这些特征，但超出了IBA规范的范围。

**子网特征继承**： 除了VL15子网管理数据包（这些数据包只在子网级别传输，不会通过路由器），全局网络继承了所有子网的数据包传递特征。

**端到端数据完整性**： IBA规范为除原始数据包 (Raw Packet) 之外的所有IBA数据包定义了一个不变循环冗余校验 (Invariant CRC, ICRC)。通过在数据包传输整个全局网络时保持ICRC不变，可以提供端到端的数据完整性保证。

**SL和VL支持**： 整个全局网络都支持服务等级 (SL) 和虚拟通道 (VL) 机制。这是通过将SL映射到GRH中的TClass来实现的。

### 全局多路径 (Global Multipath)

在子网内和子网之间路由数据包所需的信息分别包含在数据包的LRH和GRH中。与许多网络协议不同，IBA并不强制要求所有数据包都包含GRH，只有当数据包的目标设备不在同一子网中（例外：基于全局标识符 ( GID) 的寻址）或数据包是多播数据包时，才需要GRH。然而，除了子网管理数据包外，任何数据包都可以选择包含GRH。

**子网内多路径**： 在子网内部，两个通道适配器 (CA) 之间的多条路径由多个本地标识符 (LID) 标识。可以使用LID/LMC（本地管理控制）组合有效地==为一个端口分配多个LID==。源CA通过选择分配给目标端口的其中一个LID来选择路径。

**子网间多路径**： 类似地，CA可以选择支持，==为一个端口分配多个GID==。在跨子网的全局路由中，LID指示在子网内使用的有效路径（即交换机转发），而GID指示在子网之间使用的有效路径（即路由器转发）。

通过这种方式，源CA可以选择通过网络的路径，拥有两个自由度：选择LID决定了通过源子网到第一个路由器的路由；选择GID决定了到达第一个路由器后采用的路由。路径上的每个路由器都可以通过选择下一个路由器（或最终目标）的LID来选择通过子网到下一个路由器（或最终目标）的路径。此外，由于DLID可能包含多路径数据的LMC位，因此路由器也可以使用DLID作为其路由确定算法的一部分。

IBA规范没有指定路由器转发数据包的具体决策过程；但是，路由器可以依赖`目标GID`、`源GID`、`SL`、`TClass`和`FlowLabel`等字段的各种组合以及其他因素来确定转发路径和必须按顺序传递的流量。`CA`和/或入口路由器可以使用`GRH`中的`FlowLabel`来标记预期按顺序传递的数据包流。==虽然IBA路由器主要利用`LID`和`GID`来确定路径，但非IBA路由器也可以使用`FlowLabel`来确定路径==。


### GRH
![](attachments/Pasted%20image%2020250801123813.png)

**补充说明**
1. GRH是可选的，只有当需要跨子网路由或使用多播功能时才需要。
2. 通过结合使用SGID、DGID、TCLASS、FLOWLABEL等字段，IBA路由器可以实现复杂的路由和QoS策略。
3. GRH的设计借鉴了IPv6的一些概念，例如IPVER、TCLASS、FLOWLABEL和HOPLMT等字段，这有助于IBA网络与IP网络的互操作性。


![](attachments/Pasted%20image%2020250801115318.png)

#### 字段说明

全局路由头 (GRH) 并非所有数据包都必需的。本地路由头 (LRH) 中的“链路下一报头 (LNH) ”字段会指示是否存在GRH。GRH用于在InfiniBand (IBA) 全局网络中进行跨子网的路由。

以下详细描述GRH中的每个字段：
**IP 版本 (IP Version, IPVER) - 4 位**
指示GRH的版本。根据IBA规范，此字段始终设置为6，表示该报头格式类似于IPv6。

**流量类别 (Traffic Class, TCLASS) - 8 位**
此字段用于在端到端（即跨越多个子网）传递服务等级 (SL) 信息。它提供了类似IPv6的流量类别功能。
映射： 特定SL值到特定TCLASS值的映射没有在IBA规范中明确定义，可以根据具体实现而有所不同。这允许网络管理员根据需要自定义QoS策略。

**流标签 (Flow Label, FLOWLABEL) - 20 位**
此字段可用于标识属于同一流的一系列数据包，并要求这些数据包按照发送顺序进行传递。这对于对顺序敏感的应用程序（例如流媒体）非常重要。

**有效载荷长度 (Payload Length, PAYLEN) - 16 位**
- 对于IBA数据包：此字段指定从GRH之后的第一个字节开始，到不变CRC (ICRC) 的最后一个字节（包含ICRC）的字节数。
- 对于原始IPv6数据报：此字段指定从GRH之后的第一个字节开始，到可变CRC (VCRC) 或4字节数据包长度倍数填充之前的最后一个字节的字节数。这使得IBA能够封装原始IPv6数据包。

**下一报头 (Next Header, NXTHDR) - 8 位**
此字段指示紧随GRH之后的报头类型。例如，它可以指示基本传输报头 (BTH) 或其他扩展报头。此字段的值与IPv6中的“下一报头”字段类似。

**跳数限制 (Hop Limit, HOPLMT) - 8 位**
此字段指示数据包在被丢弃之前允许经过的路由器数量（即跳数）。它类似于IPv6中的“跳数限制”字段或IPv4中的“生存时间 ( TTL)”字段。
- 防止循环： HOPLMT的主要目的是防止由于路由循环而导致数据包在网络中无限期循环。每当数据包经过一个路由器时，HOPLMT的值就会减1。当HOPLMT的值变为0时，路由器会丢弃该数据包。
    
- 限制本地子网： 将此值设置为0或1将确保数据包不会转发到本地子网之外。


**源全局标识符 (Source Global Identifier, SGID) - 128 位**

此字段标识将数据包注入到全局IBA网络的端口的全局唯一标识符。SGID用于跨子网的路由。

**目标全局标识符 (Destination Global Identifier, DGID) - 128 位**

此字段标识数据包的最终目标端口的全局唯一标识符，或者如果数据包是多播数据包，则DGID表示要将数据包传递到的一组端口的多播组标识符。DGID也用于跨子网的路由。

#### 全局路由头GRH的修改
 IBA 路由器在子网之间转发数据包时可以和必须对 GRH 进行的修改。需要注意的是，修改这些字段意味着需要重新计算并更新可变 CRC (VCRC)。这些更改不会影响不变 CRC (ICRC)。

**IP版本 (IP Version, IPVer)**： IBA 路由器禁止修改此字段。该字段应保持原始值6不变，以确保GRH的版本一致性。

**流量类别 (Traffic Class, TClass)**： 此字段用于端到端（即跨越多个子网）传递服务等级 (SL) 信息。路由器利用此字段来确定在下一个子网上转发数据包时应使用的适当SL。具体的TClass到SL的映射规则并未在IBA规范中定义，允许各实现根据需要进行自定义。
> 如果TClass字段的值不为零 ，则IBA路由器禁止修改该字段。
> 当TClass字段的值为零时，IBA规范未定义路由器应如何处理该字段。
    
**流标签 (FlowLabel)**： 此字段可用于标识必须按顺序传递的一系列数据包。IBA规范不强制要求使用此字段。
> 如果未使用FlowLabel（即其值为0），则路由器应保持该字段的值不变。
> 如果使用FlowLabel（即其值为非零），路由器可以 更改FlowLabel的值；但是，路由器必须确保所有需要按顺序传递的数据包（包括任何给定的两个队列对 (QP) 之间的所有流量）都使用相同的FlowLabel值。这确保了属于同一流的数据包能够按照正确的顺序到达目的地。

**有效载荷长度 (Payload Length, PayLen)**： IBA路由器禁止修改PayLen字段的内容。该字段反映了数据包的实际有效载荷长度，修改会导致数据包解析错误。
    
**下一报头 (Next Header, NxtHdr)**： IBA路由器禁止修改NxtHdr字段的内容。该字段指示GRH之后跟随的报头类型，修改会导致数据包解析错误。
    
**跳数限制 (Hop Limit, HopLmt)**： IBA路由器应执行以下操作：
> 如果HopLmt字段中包含的值为1或0，则路由器应丢弃该数据包，以防止数据包在网络中无限循环。
> 否则（即HopLmt的值大于1），IBA路由器应将HopLmt字段的值递减1 ，然后继续转发数据包。

**源全局标识符 (Source Global Identifier, SGID)**： IBA路由器禁止修改SGID字段的内容。SGID标识了数据包的原始发送者，修改会导致回复或其他相关操作出现问题。
    
**目标全局标识符 (Destination Global Identifier, DGID)**： IBA路由器禁止 修改DGID字段的内容。DGID标识了数据包的最终目标，修改会导致数据包无法正确送达目的地。

## 传输层

### 各个服务类型的属性

![](attachments/Pasted%20image%2020250801180703.png)
![](attachments/Pasted%20image%2020250801180719.png)




### BTH
所有 IBA 传输服务都包含基础传输头 (BTH) 中的字段。本地路由头 (LRH) 中的“链路下一报头 (LNH) ”字段用于指示BTH的存在。原始数据包不使用IBA传输服务，因此不需要BTH。
![](attachments/Pasted%20image%2020250801141430.png)
![](attachments/Pasted%20image%2020250801115912.png)
![](attachments/Pasted%20image%2020250801115916.png)

#### 字段
##### 操作码 (OPCODE)

OpCode 字段定义了其余报头和有效负载字节

|Code[7:5]|Code[4:0]|Description|Packet Contents following the Base Transport headera|
|---|---|---|---|
|000<br><br>  <br><br>Reliable Connection (RC)|00000|SEND First|PayLd|
||00001|SEND Middle|PayLd|
||00010|SEND Last|PayLd|
||00011|SEND Last with Immediate|ImmDt, PayLd|
||00100|SEND Only|PayLd|
||00101|SEND Only with Immediate|ImmDt, PayLd|
||00110|RDMA WRITE First|RETH, PayLd|
||00111|RDMA WRITE Middle|PayLd|
||01000|RDMA WRITE Last|PayLd|
||01001|RDMA WRITE Last with Immediate|ImmDt, PayLd|
||01010|RDMA WRITE Only|RETH, PayLd|
||01011|RDMA WRITE Only with Immediate|RETH, ImmDt, PayLd|
||01100|RDMA READ Request|RETH|
||01101|RDMA READ response First|AETH, PayLd|
||01110|RDMA READ response Middle|PayLd|
||01111|RDMA READ response Last|AETH, PayLd|
||10000|RDMA READ response Only|AETH, PayLd|
||10001|Acknowledge|AETH|
||10010|ATOMIC Acknowledge|AETH, AtomicAckETH|
||10011|CmpSwap|AtomicETH|
||10100|FetchAdd|AtomicETH|
||10101|Reserved|Undefined|
||10110|SEND Last with Invalidate|IETH, PayLd|
||10111|SEND Only with Invalidate|IETH, PayLd|
||11000-11111|Reserved|undefined|
|001<br><br>  <br><br>Unreliable Connection (UC)|00000|SEND First|PayLd|
||00001|SEND Middle|PayLd|
||00010|SEND Last|PayLd|
||00011|SEND Last with Immediate|ImmDt, PayLd|
||00100|SEND Only|PayLd|
||00101|SEND Only with Immediate|ImmDt, PayLd|
||00110|RDMA WRITE First|RETH, PayLd|
||00111|RDMA WRITE Middle|PayLd|
||01000|RDMA WRITE Last|PayLd|
||01001|RDMA WRITE Last with Immediate|ImmDt, PayLd|
||01010|RDMA WRITE Only|RETH, PayLd|
||01011|RDMA WRITE Only with Immediate|RETH, ImmDt, PayLd|
||01100-11111|Reserved|undefined|
|010<br><br>  <br><br>Reliable Datagram (RD)|00000|SEND First|RDETH, DETH, PayLd|
||00001|SEND Middle|RDETH, DETH, PayLd|
||00010|SEND Last|RDETH, DETH, PayLd|
||00011|SEND Last with Immediate|RDETH, DETH, ImmDt, PayLd|
||00100|SEND Only|RDETH, DETH, PayLd|
||00101|SEND Only with Immediate|RDETH, DETH, ImmDt, PayLd|
||00110|RDMA WRITE First|RDETH, DETH, RETH, PayLd|
||00111|RDMA WRITE Middle|RDETH, DETH, PayLd|
||01000|RDMA WRITE Last|RDETH, DETH, PayLd|
||01001|RDMA WRITE Last with Immediate|RDETH, DETH, ImmDt, PayLd|
||01010|RDMA WRITE Only|RDETH, DETH, RETH, PayLd|
||01011|RDMA WRITE Only with Immediate|RDETH, DETH, RETH, ImmDt, PayLd|
||01100|RDMA READ Request|RDETH, DETH, RETH|
||01101|RDMA READ response First|RDETH, AETH, PayLd|
||01110|RDMA READ response Middle|RDETH, PayLd|
||01111|RDMA READ response Last|RDETH, AETH, PayLd|
||10000|RDMA READ response Only|RDETH, AETH, PayLd|
||10001|Acknowledge|RDETH, AETH|
||10010|ATOMIC Acknowledge|RDETH, AETH, AtomicAckETH|
||10011|CmpSwap|RDETH, DETH, AtomicETH|
||10100|FetchAdd|RDETH, DETH, AtomicETH|
||10101|RESYNC|RDETH, DETH|
||10110-11111|Reserved|undefined|
|011<br><br>  <br><br>Unreliable Datagram (UD)|00000-00011|Reserved|undefined|
||00100|SEND only|DETH, PayLd|
||00101|SEND only with Immediate|DETH, ImmDt, PayLd|
||00110-11111|Reserved|undefined|
|100<br><br>  <br><br>CNP|00000|CNP|none|
||00001-11111|Reserved|undefined|
|101<br><br>  <br><br>Extended Reliable Connection (XRC)|00000|SEND First|XRCETH, PayLd|
||00001|SEND Middle|XRCETH, PayLd|
||00010|SEND Last|XRCETH, PayLd|
||00011|SEND Last with Immediate|XRCETH, ImmDt, PayLd|
||00100|SEND Only|XRCETH, PayLd|
||00101|SEND Only with Immediate|XRCETH, ImmDt, PayLd|
||00110|RDMA WRITE First|XRCETH, RETH, PayLd|
||00111|RDMA WRITE Middle|XRCETH, PayLd|
||01000|RDMA WRITE Last|XRCETH, PayLd|
||01001|RDMA WRITE Last with Immediate|XRCETH, ImmDt, PayLd|
||01010|RDMA WRITE Only|XRCETH, RETH, PayLd|
||01011|RDMA WRITE Only with Immediate|XRCETH, RETH, ImmDt, PayLd|
||01100|RDMA READ Request|XRCETH, RETH|
||01101|RDMA READ response First|AETH, PayLd|
||01110|RDMA READ response Middle|PayLd|
||01111|RDMA READ response Last|AETH, PayLd|
||10000|RDMA READ response Only|AETH, PayLd|
||10001|Acknowledge|AETH|
||10010|ATOMIC Acknowledge|AETH, AtomicAckETH|
||10011|CmpSwap|XRCETH, AtomicETH|
||10100|FetchAdd|XRCETH, AtomicETH|
||10101|Reserved|Undefined|
||10110|SEND Last with Invalidate|XRCETH, IETH, PayLd|
||10111|SEND Only with Invalidate|XRCETH, IETH, PayLd|
||11000-11111|Reserved|undefined|
|110 - 111|00000-11111|Manufacturer Specific OpCodes|undefined|


##### 请求事件 (Solicited Event, SE) - 1 位

发送方（请求方）将此位设置为1，表示接收方（响应方）应在接收到该数据包后调用相应的完成队列 (CQ) 事件处理程序，以通知应用程序数据传输的完成。

- 使用场景： SE位应仅在SEND、SEND with Immediate或RDMA Write with Immediate操作的最后一个或唯一数据包中设置。
    
- SEND with Invalidate： 除了上述操作外，SE位也可以用于SEND with Invalidate操作。在这种情况下，SE位同样应仅在SEND with Invalidate操作的最后一个或唯一数据包中设置。SE位在SEND with Invalidate操作中的使用规则与其他SEND操作相同。
    
- 重要性： SE位不属于数据包头部验证的一部分，即接收方即使接收到设置了SE位但不满足调用要求的（例如，不是操作的最后一个数据包）数据包，也不会因此产生否定应答 ( NAK)。


##### 分区密钥 (Partition Key, P_KEY) - 16 位

P_KEY用于标识目标队列对 (QP)（RC、UC、UD、XRC类型）或EE上下文 (RD类型) 所属的分区。分区提供了一种隔离不同用户或应用程序流量的机制。

##### 目标队列对 (Destination Queue Pair, DESTQP) - 24 位

此字段指定目标QP的标识符。QP是IBA中进行数据传输的基本单元。

##### 前向显式拥塞通知/保留1 (Forward Explicit Congestion Notification/Reserved 1, F/RES1) - 1 位

- F (FECN)：
    - 0：表示数据包在传输过程中未遇到拥塞点。
        
    - 1：表示数据包经过了拥塞点，网络正在经历拥塞。
        
- Res1： 保留位，应设置为0，接收方在接收时应忽略此位。此字段不包含在ICRC中。
    
##### 后向显式拥塞通知/保留1 (Backward Explicit Congestion Notification/Reserved 1, B/RES1) - 1 位

- B (BECN)：
    - 0：表示数据包未经过拥塞点，或者经过了拥塞点但未被标记为拥塞。
        
    - 1：表示此报头指示的数据包受到转发拥塞的影响。B位通常在确认 (ACK) 或拥塞通知 (CN) BTH中设置，用于向发送方反馈拥塞信息。
        
- Res1： 保留位，应设置为0，接收方在接收时应忽略此位。此字段不包含在ICRC中。

##### 保留6 (Reserved 6, RESV6) - 6 位

保留位，应设置为0，接收方在接收时应忽略此字段。此字段不包含在ICRC中。

> 注：生成数据包时，发送方应将Resv6、F/Res1和B/Res1字段设置为零。


##### 确认请求 (Acknowledge Request, A) - 1 位

此位用于指示接收方是否需要在关联的QP上发送ACK回复。如果设置为1，则接收方需要发送ACK；如果设置为0，则不需要发送ACK。

##### 数据包序列号 (Packet Sequence Number, PSN) - 24 位

此字段用于标识数据包在一系列数据包中的位置。接收方可以根据传输服务类型和/或实现要求，通过验证PSN来检测丢失的数据包，从而实现可靠传输。



### 扩展传输头：ETH
#### 可靠数据报扩展传输头：RDETH(Reliable Datagram Eth)

![](attachments/Pasted%20image%2020250801173543.png)
![](attachments/Pasted%20image%2020250801120014.png)

RDETH用于可靠数据报 (RD) 传输服务，包含端到端 (EE： End-to-End) 上下文标识符。

- **保留字段 (Reserved)**: 8位。如果CA实现了可靠数据报功能，发送方在生成数据包时应将此字段设置为0x0。接收方应忽略此字段。
    
- **端到端 (EE) 上下文 (End-to-End Context)**: 24位。此字段指示用于该数据包的EE上下文。EE上下文是唯一的终端节点标识符，用于在任意两个终端节点之间复用/解复用可靠数据报数据包。EE上下文为可靠传输状态提供了一个上下文，类似于用于可靠连接的上下文，但应用于无连接的可靠数据报服务。


#### 数据报扩展传输头：DETH(Datagram Eth)

![](attachments/Pasted%20image%2020250801173438.png)
![](attachments/Pasted%20image%2020250801120019.png)


DETH用于可靠和不可靠数据报服务，包含附加的传输字段。

- **Q_KEY**: 32位。此字段是授权访问目标队列对 (QP) 所必需的。接收方（响应方）将此字段与目标QP的Q_Key进行比较，以进行访问控制。
    
- **保留字段 (Reserved)**: 8位。发送方在生成数据包时应将此字段设置为0x0。接收方应忽略此字段。
    
- **源QP (Source QP, SRCQP)**: 24位。此字段指定源QP的标识符，用作回复数据包的目标QP。

#### XRCETH
![](attachments/Pasted%20image%2020250801120240.png)

#### RDMA 扩展传输头：RETH（RdmaETH）
RETH用于RDMA操作，包含RDMA操作所需的附加传输字段。

![](attachments/Pasted%20image%2020250801173314.png)
![](attachments/Pasted%20image%2020250801120112.png)

**虚拟地址 (Virtual Address, VA)**: 64位。此字段包含目标缓冲区在远程内存中的起始虚拟地址。RDMA VA可以从任何字节边界开始。

**R_KEY (Remote Key)**: 32位。R_Key具有以下重要属性：

- 访问控制： R_Key充当保护密钥，用于控制对给定操作指定内存地址和范围的访问。这是一种保护机制，用于确保正确访问目标内存。接收方将R_Key与其本地保护机制相关联，以验证发送方的访问权限。
    
- 密钥分发： R_Key必须由目标端（内存所有者）发送给请求端。
    
- 授权访问类型： R_Key授予对RDMA读取、RDMA写入和原子操作的任何组合的访问权限。
    
- 内存区域/窗口关联： 每个内存区域 (Memory Region) 或内存窗口 (Memory Window) 在任何给定时刻都有一个有效的R_Key。虚拟连续的内存位置范围可以同时具有与其关联的多个区域或窗口，每个区域或窗口都有一个关联的R_Key。
    
- 使用范围： R_Key仅用于RDMA和原子操作，包含在数据包头中。
    
- 验证： 支持RDMA和/或原子操作的接收方应验证R_Key、关联的访问权限和指定的虚拟地址。接收方还必须执行边界检查（即验证引用的数据长度是否未超过关联的内存起始地址和结束地址）。任何违规都必须导致数据包被丢弃，并且对于可靠服务，必须生成否定应答 ( NAK)。
    

**DMA长度 (DMA Length, DMALEN)**:
32位。此字段指示远程DMA操作的长度（以字节为单位）。对于执行RDMA操作的HCA，DMALen字段中指定的最小长度为0，最大长度为231。

#### AtomicETH

![](attachments/Pasted%20image%2020250801120220.png)

#### ACK 扩展传输头：AETH(AckETH)
![](attachments/Pasted%20image%2020250801173633.png)
![](attachments/Pasted%20image%2020250801120251.png)


AETH包含ACK数据包的附加传输字段。所有ACK以及RDMA READ response消息的第一个和最后一个数据包都包含AETH。

**症状代码 (Syndrome)**:
此字段用于指示数据包是确认 (ACK) 还是否定应答 (NAK)。
- 如果是ACK且QP与可靠连接（RC）传输服务相关联，则症状代码还提供限制序列号 (LSN: Limit SN)，用于端到端（消息级）流控。
- 如果是NAK，则此字段指示错误代码。对于RNR（Receive Not Ready）NAK，此字段指示在重新传输请求之前要使用的定时器值。


**消息序列号 (Message Sequence Number, MSN)**:

响应方完成的最后一条消息的序列号。此字段用于优化请求方的完成处理。

##### AETH Sydrome字段的值

|bit 7|bits 6:5|bits 4:0|Definition|
|---|---|---|---|
|0|00|C CCCC|ACK (C CCCC = credit count)|
|0|01|T TTTT|RNR NAK (T TTTT = timer value)|
|0|10|X XXXX|reserved|
|0|11|N NNNN|NAK (N NNNN = NAK code)|

```bash
C CCCC = encoded end-to-end flow control credits 
注：If a CA implements Reliable Datagram service, the C CCCC bits are set to zero, since end to end credits are not defined for RD service.

T TTTT = RNR NAK Timer Field
```


###### 其他NAK类型

|NAK Code (AETH bits 4:0)|Definition|
|---|---|
|0 0000|PSN Sequence Error|
|0 0001|Invalid Request|
|0 0010|Remote Access Error|
|0 0011|Remote Operational Error|
|0 0100|Invalid RD Request|
|0 0101 - 1 1111|reserved|

###### RNR NAK

|RNR Time|Delay in milliseconds|RNR Time|Delay in milliseconds|
|---|---|---|---|
|00000|655.36|10000|2.56|
|00001|0.01|10001|3.84|
|00010|0.02|10010|5.12|
|00011|0.03|10011|7.68|
|00100|0.04|10100|10.24|
|00101|0.06|10101|15.36|
|00110|0.08|10110|20.48|
|00111|0.12|10111|30.72|
|01000|0.16|11000|40.96|
|01001|0.24|11001|61.44|
|01010|0.32|11010|81.92|
|01011|0.48|11011|122.88|
|01100|0.64|11100|163.84|
|01101|0.96|11101|245.76|
|01110|1.28|11110|327.68|
|01111|1.92|11111|491.52|




##### MSN(Message Sequence Number）
**MSN 的作用**： 告知请求方响应方已完成的消息数量，帮助请求方完成 WQE。
    
**MSN 的位置**： 包含在 AETH 的三个最低有效字节中。

**SSN（Send Sequence Number） 的概念**： 请求方逻辑上为每个 WQE 分配一个 SSN，与响应方返回的 MSN 一一对应。这只是一个逻辑概念，实际实现中不一定需要完全按照此方式实现。

**MSN 的初始化和递增**： 响应方将 MSN 初始化为 0，并在完成每个入站请求消息后递增。

![](attachments/Pasted%20image%2020250804162155.png)


**MSN 初始化**： HCA 响应方将其 MSN 值初始化为零。

**MSN 递增时机**： 仅当成功完成处理 新的、有效的 请求消息时才递增 MSN。重复的请求不会导致 MSN 递增。

**MSN 返回位置**： 对于 RDMA READ 或 Atomic 响应，递增后的 MSN 应在 Last 或 Only 数据包 中返回。 对于 RDMA READ 请求，响应方 可以在 完成请求验证后、开始传输数据前递增 MSN，并将其放在 第一个响应数据包的 AETH 中返回。这提供了一种优化的可能性，允许更早地通知请求方。

**MSN 递增次数**： 对于任何给定的请求消息，MSN 只能递增一次，确保了 MSN 和请求的一一对应关系。

![](attachments/Pasted%20image%2020250804162503.png)

![](attachments/Pasted%20image%2020250804162508.png)

#### AtomicAckEth
![](attachments/Pasted%20image%2020250801120257.png)

#### 立即数扩展传输头：ImmDtEth (immediate data Eth)
![](attachments/Pasted%20image%2020250801173907.png)
![](attachments/Pasted%20image%2020250801120313.png)

IMMDT包含放置在接收完成队列元素 (CQE) 中的立即数数据 (ImmDt)。ImmDt仅允许在带有立即数数据的SEND或RDMA WRITE数据包中使用。


#### 失效扩展传输头：IETH(Invalidate Eth)

![](attachments/Pasted%20image%2020250801174115.png)
![](attachments/Pasted%20image%2020250801120404.png)

IETH用于`SEND with Invalidate`操作，包含R_Key字段。
接收方使用此`R_Key`在接收并执行`SEND with Invalidate`请求后使指定的内存区域或内存窗口失效, 后续发送方就不能再访问这个内存区域了。


### 传输操作

IBA 支持多种传输功能，根据可靠性和连接类型的不同，可以分为以下几类：

|传输功能|可靠连接 (RC)|不可靠连接 (UC)|XRC|可靠数据报 (RD)|不可靠数据报 (UD)|原始数据报 (Raw)|
|---|---|---|---|---|---|---|
|SEND|支持|支持|支持|支持|支持|不适用|
|RESYNC|不支持|不支持|不支持|支持|不支持|不支持|
|RDMA WRITE|支持|支持|支持|支持|不支持|不适用|
|RDMA READ|支持|不支持|支持|支持|不支持|不适用|
|原子操作 (ATOMIC)|可选支持|不支持|可选支持|可选支持|不支持|不适用|

#### send 操作

![](attachments/Pasted%20image%2020250804111038.png)


SEND 操作用于发送数据，根据消息大小和PMTU (Path Maximum Transmission Unit，路径最大传输单元) 的关系，有不同的处理方式：

**消息长度小于等于 PMTU**： 使用 BTH 操作码“SEND Only”或“SEND Only with imm”（带有立即数）。
    
**消息长度为零**： 同样使用 BTH 操作码“SEND Only”或“SEND Only with imm”。在这种情况下，没有数据有效载荷，但所有其他字段仍然存在。
    
**消息长度大于 PMTU**： 需要将消息分段传输：
- 第一个数据包的 BTH 操作码为“SEND First”。
	
- 最后一个数据包的 BTH 操作码为“SEND Last”或“SEND Last with imm”。
	
- 第一个和最后一个数据包之间的所有数据包的 BTH 操作码为“SEND Middle”。
	
**数据包长度**： 除了操作码为“SEND Only”、“SEND Only with imm”、“SEND Last”或“SEND Last with imm”的数据包外，消息中所有其他数据包的数据字段长度都应为 PMTU。
    
**消息长度的确定**： 接收节点（发送操作的目标）只有在接收到带有“SEND Last”或“SEND Last with imm”操作码的最后一个数据包后才能知道发送消息的最终长度。
    
**SEND 消息拆分为多数据包发送的约束**： 对于某个请求节点的指定QP，一旦启动多数据包SEND操作，在“SEND Last”或“SEND Last with imm”数据包之前，不得生成其他请求数据包。多数据包的消息（一个消息被拆分为多个数据包）的send不得与同一 SEND 队列上的其他操作交错。

**立即数数据**： 并非所有 SEND 消息都带有立即数数据。如果带有立即数数据，则会在消息的最后一个或唯一数据包中包含一个特殊的立即数扩展传输头 (IMMDT)。BTH 中的特殊操作码“SEND Last with imm”或“SEND only with imm”用于指示 IMMDT 的存在。

#### RDMA WRITE 操作

![](attachments/Pasted%20image%2020250804111838.png)

RDMA WRITE 操作用于将数据直接写入远程内存，类似 SEND 操作，根据消息大小和PMTU的关系，有不同的处理方式：

**消息长度为零**： 使用 BTH 操作码“RDMA WRITE Only”或“RDMA WRITE Only with imm”。在这种情况下，没有数据有效载荷，但所有其他字段仍然存在。
    
**消息长度小于等于 PMTU**： 使用 BTH 操作码“RDMA WRITE Only”或“RDMA WRITE Only with imm”。
    
**消息长度大于 PMTU**： 需要将消息分段传输：
- 第一个数据包的 BTH 操作码为“RDMA WRITE First”。
	
- 最后一个数据包的 BTH 操作码为“RDMA WRITE Last”或“RDMA WRITE Last with imm”。
	
- 第一个和最后一个数据包之间的所有数据包的 BTH 操作码为“RDMA WRITE Middle”。
        
**数据包长度**： 每个在 RDMA WRITE 消息中的数据包，如果其操作码不是 ”RDMA WRITE Only”、”RDMA WRITE Only with Immediate”、”RDMA WRITE Last” 或 ”RDMA WRITE Last with Immediate”，都会有一个长度为 PMTU 的数据字段。

**RETH 报头**： RDMA 扩展传输头 (RETH) 出现在消息的第一个数据包中。它包含目标缓冲区的虚拟地址 (VA)、远程密钥 (R_Key) 和消息长度 (DMALEN) 字段。
    
**多数据包 RDMA WRITE 操作的约束**： 对于请求节点的给定QP，一旦启动多数据包 RDMA WRITE 操作，在发送“RDMA WRITE Last”或“RDMA WRITE Last with imm”数据包之前，不得生成其他请求数据包。多数据包的消息（一个消息被拆分为多个数据包）的write 不得与同一 SEND 队列上的其他操作交错。

**RDMA WRITE 响应**： 生成 RDMA WRITE 响应时，响应方应在每个响应数据包中至少包含以下报头和字段：LRH、BTH、AETH、ICRC、VCRC。

#### RDMA READ 操作

![](attachments/Pasted%20image%2020250804113244.png)

RDMA READ 操作用于从远程内存读取数据：

**BTH 操作码**： BTH 操作码字段用于将数据包标识为 RDMA READ 请求或响应，并确定是否存在任何扩展传输头。
    
**请求零字节传输**： 如果 RDMA READ 请求消息请求零字节（即：响应为零字节）传输，则使用 BTH 操作码“RDMA READ Response Only”。所有其他字段保持不变。

**响应消息长度小于等于 PMTU**： 使用 BTH 操作码“RDMA READ Response Only”。
    
**响应消息长度大于 PMTU**： 需要将消息分段传输：
- 第一个数据包的 BTH 操作码为“RDMA READ Response First”。
- 最后一个数据包的 BTH 操作码为“RDMA READ Response Last”。
        
**数据包序列号 (PSN)**： PSN 字段用于检测乱序或丢失的响应数据包。
    
**未完成请求**： 在发送 RDMA READ 请求数据包后，请求节点可以发送其他请求数据包，而无需等待响应数据包返回。QP 在任何时候未完成的 RDMA READ 请求的最大数量可以在连接建立时协商。响应方可以将连接限制为每个 QP 只有一个未完成的 RDMA READ 请求。
    
**立即数数据**： RDMA READ 数据包从不携带立即数数据。
- 重试机制： 如果请求方没有收到正确的响应，则会重试 RDMA READ 请求。重试的 RDMA READ 请求不需要从相同的地址开始，也不需要与原始 RDMA READ 具有相同的长度。重试的请求只能重新READ第一次未成功响应的部分。响应方验证重试请求的 R_Key 和 RDMA 读取虚拟地址。


### 可靠服务(RELIABLE SERVICE)

可靠服务旨在保证消息按发送顺序无损地送达接收方。实现可靠性的关键要素包括：

**循环冗余校验 (CRC)**： 用于检测数据传输过程中发生的错误。
    
**确认 (ACK) 机制**： 接收方通过发送ACK来告知发送方已成功接收数据。
    
**丢失数据检测（PSN）**： 发送方和接收方通过序列号等机制来检测丢失的数据包。并允许请求方将收到的响应与最初的请求关联起来（相关性）。
    
**超时重传机制（Timer）**： 发送方在超时后重传未被确认的数据包。

#### 可靠服务的类型 (Types)
- **可靠连接 (Reliable Connection, RC)**
- **可靠数据报 (Reliable Datagram, RD)**
- **扩展可靠连接 (Extended Reliable Connection)**


#### PSN
PSN包含在所有数据包的基础传输头 (BTH) 中。它用于检测丢失或乱序的数据包，并且在可靠服务中，用于将ACK与特定的请求数据包关联起来。

对于使用可靠连接服务 (RC) 的请求方，其生成的每个请求数据包都应包含一个PSN。响应方在生成每个响应数据包时也应包含一个PSN。 除了`RDMA READ Response`这种特殊情况外，请求数据包中的PSN与其对应的响应数据包中的PSN之间存在一对一的关系。

**PSN的生成和使用**：

![](attachments/Pasted%20image%2020250804120750.png)

- (1)请求方： 
请求方会计算即将生成的下一个请求数据包的PSN，称为“Next PSN / 下一个 PSN”。当请求方生成新的请求数据包时，“Next PSN / 下一个 PSN”的值会被复制到BTH的PSN字段中，成为当前数据包的PSN。然后，请求方会计算一个新的“Next PSN / 下一个 PSN”值。

- (2)响应方： 
为了检测丢失或乱序的数据包，响应方也会计算它期望在下一个收到的请求数据包中找到的PSN，称为“Expected PSN / 预期 PSN”。

- (3)确认合并「ack合并」的影响： 
由于确认合并机制的存在，请求方不一定能够准确预测下一个响应数据包中可能出现的PSN范围。


#### ACK/NAK 协议

ACK/NAK协议与PSN一起，构成了可靠服务的基础。该协议适用于可靠连接服务（RC）、XRC服务和可靠数据报服务(RD)。

**协议目标**： 
ACK/NAK协议的目的是使请求方能够确切地知道响应方是否正确接收了请求数据包。此外，该协议还提供了一些机制来确保完整消息的正确接收。这是通过结合使用PSN和数据包操作码（指示First/Middle/Last/Only）来实现的。

由于响应数据包可能在网络中丢失，ACK/NAK协议要求请求方实现一个定时器来检测丢失的响应数据包，并在超时后进行重传请求。

##### 术语
**(1)确认**
确认一词用于表示否定确认 (NAK： negative ACK) 或肯定确认 (ACK)。

**(2) 响应**
响应是一个通用术语，用于描述接收方（响应方）返回给发送方（请求方）的确认。响应包含在一个或多个确认数据包中，并且可能包含ACK数据包、NAK数据包、`RDMA READ Response`或`ATOMIC操作响应`，具体取决于原始请求消息的类型。


##### 管理ACK/NAK协议的规则摘要
```bash
The following is a summary of the rules governing the ACK/NAK protocol: 
• Each request packet received on a reliable service shall be acknowledged.

• Each RDMA READ request requires an explicit response. A RDMA
READ response, with a properly formed ACK Extended Transport Header (AETH) is considered a valid response. The ACK Extended Transport Header appears in the first packet and last packet (or only packet) of a RDMA READ response. 

• Each ATOMIC Request requires an explicit response. An acknowledge packet, with a properly formed ACK Extended Transport Header (AETH) and an ATOMIC ACK Extended Transport Header (AtomicAckETH) is considered to be a valid response.

• Acknowledges may be coalesced; that is, a single acknowledge packet can serve as acknowledgment for one or more previous request packets.

• Acknowledge packets shall be returned in the PSN order in which the original request packet was received, including RDMA READ responses.

• A RDMA READ response consists of one or more packets; all other responses consist of exactly one packet.
```

**请求确认**： 在可靠服务上接收到的每个请求数据包都应得到确认。
    
**RDMA READ请求的响应**： 每个RDMA READ请求都需要一个显式响应。具有正确格式的ACK扩展传输报头 (AETH) 的RDMA READ Response被视为有效响应。AETH出现在RDMA READ Response的第一个数据包和 最后一个数据包（或Only数据包） 中。
    
**ATOMIC请求的响应**： 每个ATOMIC请求都需要一个显式响应。具有正确格式的AETH和ATOMIC ACK扩展传输报头 (AtomicAckETH) 的确认数据包被视为有效响应。
    
**确认合并**： ACK可以合并；也就是说，单个确认数据包可以作为对一个或多个先前请求数据包的确认。
    
**确认顺序**： 确认数据包应按照原始请求数据包接收到的PSN顺序返回，包括RDMA READ Response。
    
**响应数据包数量**： RDMA READ Response由一个或多个数据包组成；所有其他响应都只包含一个数据包。

#### 请求方：生成请求数据包

请求方负责生成发送给响应方的请求数据包。生成过程包括确定和设置数据包序列号 (PSN)、操作码 (OpCode) 和有效负载。

##### 生成包序列号 (PSN)
对于可靠连接服务 (RC)，请求方必须在每个请求数据包的 BTH:PSN 字段中设置一个值，该值称为当前 PSN。

**初始 PSN**： 在连接建立期间，传输层的客户端会将“下一个 PSN”初始化为一个介于 `0~16777215「1的24次方-1」` 之间的任意值。为了确保正常通信，响应方的初始“预期 PSN”值必须与请求方的“下一个 PSN”初始值相同。

**后续 PSN 计算**： 连接建立后，请求方会根据正在执行的操作（SEND、RDMA READ 等）和数据有效负载的大小来计算“下一个 PSN”。


**RDMA READ 请求的特殊处理**：
为了正确处理RDMA READ请求的响应，紧跟在RDMA READ请求消息之后的请求数据包的PSN应比最后一个预期的RDMA READ Response数据包的PSN大1。这种方式会在请求数据包的PSN序列中预留一个“空洞”，确保所有READ Response数据包都具有单调递增的PSN，方便接收方进行排序和处理。
具体计算公式如下：
```bash
设 `curr_PSN` = RDMA READ 请求的 PSN
设 `next_PSN` = 紧跟在 RDMA READ 请求之后的请求的 PSN
设 `n` = 预期的 RDMA READ Response 数据包的数量
则 `next_PSN = (curr_PSN + n) mod 2^24`
```

![](attachments/Pasted%20image%2020250804145642.png)


##### FMLO标记
**FMLO 标记：** 在每个数据包的BTH头部中，有一个控制字段，用于标记该包在整个Message中的位置：
    - **First：** 属于Message的第一个包。
    - **Middle：** 属于Message的中间包。
    - **Last：** 属于Message的最后一个包。
    - **Only：** Message只有一个包。
- **作用：** 接收端通过FMLO标记来识别消息的边界，并将属于同一Message的包按顺序重新组装成完整的Message。

以下表格总结了根据前一个数据包的操作码，当前数据包的有效操作码：

|前一个数据包 OpCode|当前数据包的有效 OpCode|
|---|---|
|无（例如，连接建立后的第一个数据包）|“First” 数据包<br><br>  <br><br>“Only” 数据包|
|“First” 数据包|“Middle” 数据包（消息包含 3 个或更多数据包）<br><br>  <br><br>“Last” 数据包（消息恰好包含 2 个数据包）<br><br>  <br><br>当前操作的类型必须与前一个 OpCode 的类型匹配|
|“Middle” 数据包|“Middle” 数据包<br><br>  <br><br>“Last” 数据包<br><br>  <br><br>当前操作的类型必须与前一个 OpCode 的类型匹配|
|“Last” 数据包|“First” 数据包（新消息的第一个数据包）<br><br>  <br><br>“Only” 数据包（新的单个数据包消息）|
|“Only” 数据包|“First” 数据包<br><br>  <br><br>“Only” 数据包|


##### 生成有效负载﻿

请求方应根据操作码生成有效负载长度，规则如下：

**“First” 或 “Middle” 数据包**： 数据包有效负载长度必须是完整的路径最大传输单元 (PMTU) 大小。
    
**“Only” 数据包**： 数据包有效负载长度必须介于 0 到 PMTU 字节之间。因此，创建零字节长度传输的唯一方法是使用单个“Only”数据包消息。
    
**“Last” 数据包**： 数据包有效负载长度必须介于 1 到 PMTU 字节之间。
    
**RDMA WRITE 请求**： 如果操作码指定 RDMA WRITE 请求，则 RETH 的 DMALen 字段中指定的长度应不小于 0，且不大于 `2的31次方`字节。

#### 响应方
##### 响应方：收包后进行包的验证
响应方收到请求之后，先进行包的验证。

![](attachments/Pasted%20image%2020250804150226.png)

##### 响应方：生成响应数据包详解

###### 响应数据包序号生成﻿﻿

![](attachments/Pasted%20image%2020250804151256.png)

**SEND 和 RDMA WRITE 请求的响应**： 响应方会在响应数据包的 PSN 字段中填写被确认的最新请求数据包的 PSN。这表明响应方已经成功接收到并处理了这个请求。

**RDMA READ 请求的响应**： RDMA READ 请求的响应数据包的 PSN 需要满足连续递增的原则。即，每个响应数据包的 PSN 要比前一个响应数据包的 PSN 大 1，且从原始请求数据包的 PSN 开始。这种方式保证了响应数据包的顺序性，便于请求方进行组装。

###### 确认合并 (ACK Coalescing)﻿

为了提高效率，响应方可以将多个请求的确认信息合并到一个响应数据包中。这种机制称为确认合并。
由于合并确认的规则，连续响应数据包的 PSN 不一定是顺序的。

###### ACK 规则
当响应方发送ACK数据包时，会按照以下规则对之前的未完成请求进行确认：

**RDMA READ 或 ATOMIC 操作**： 如果有未完成的 RDMA READ 或 ATOMIC 操作请求的 PSN 小于当前 ACK 的 PSN，则认为这些请求是失败的，ACK 会隐式地表示对最早的一个失败请求发送 NAK。在此请求后续的请求会视作未确认状态。

**SEND 或 RDMA WRITE 操作**： 如果有未完成的 SEND 或 RDMA WRITE 请求的 PSN 小于当前 ACK 的 PSN 且不在上述失败的 RDMA READ 或 ATOMIC 操作请求之后，则认为这些请求是成功的，ACK 会隐式地对这些请求发送 ACK。

**RDMA READ Response 的特殊处理**：
- RDMA READ Response 的 First 数据包会隐式确认之前的所有未完成请求。
- RDMA READ Response 的 Last 数据包只显式确认当前的 RDMA READ 请求。

![](attachments/Pasted%20image%2020250804152341.png)

```bash
示例： 对于ACK数据包 a4：
ACK 数据包 a4 隐式确认了 SEND 请求 r1 和 r2，因为它们的 PSN 小于 a4 的 PSN 且不在任何失败的 RDMA READ 或 ATOMIC 操作之后。
ACK 数据包 a4 隐式地对 RDMA READ 请求 r3 发送了 NAK，因为 r3 的 PSN 小于 a4 的 PSN，并且是最早的未完成 RDMA READ 或 ATOMIC 操作。
SEND 请求 r4 并未被确认，因为它在失败的 r3 之后。
因此上述过程仅对请求 r1 和 r2 进行了合并确认，请求方必须重新发送 r3 和 r4。
```

##### 响应方生成 RDMA READ Response 详解
###### RDMA READ Response 的特殊性

RDMA READ Response 与一般的 SEND 或 RDMA WRITE 响应最大的不同在于它包含了实际的数据Payload。这使得 RDMA READ Response 的处理和确认机制变得更加复杂。

###### RDMA READ Response 的构成与确认
**数据包构成**： 一个 RDMA READ Response 可以包含多个数据包。
- 第一个数据包： 操作码为 "RDMA READ Response First"，包含 AETH，隐式确认之前所有未完成的请求。
	
- 中间数据包： 操作码为 "RDMA READ Response Middle"，不包含任何确认信息。
	
- 最后一个数据包： 操作码为 "RDMA READ Response Last" 或 "RDMA READ Response Only"（当响应只有一个数据包时），包含 AETH，显式确认当前的 RDMA READ 请求。
	
- AETH： ACK 扩展传输头，用于携带确认信息。


**确认机制**：
- 隐式确认： 第一个数据包的 AETH 会隐式确认之前所有未完成的请求，包括 SEND、RDMA WRITE 和 RDMA READ。
	
- 显式确认： 最后一个数据包的 AETH 会显式确认当前的 RDMA READ 请求。
	
- 错误处理： 如果在发送 RDMA READ Response 的过程中发生错误，会强制将出错的数据包的操作码设置为 "acknowledge"，提前终止 Response 并插入 NAK 信息。


**RDMA READ Response 数据包长度: **
- "RDMA READ Response Only"： 长度在 0 到 PMTU 字节之间，用于传输零长度数据或单个小数据包。
- "RDMA READ Response First" 或 "RDMA READ Response Middle"： 长度恰好为 PMTU 字节，用于传输大数据包的一部分。
- "RDMA READ Response Last"： 长度在 1 到 PMTU 字节之间，用于传输大数据包的最后一部分。

**零长度 RDMA READ 请求的处理**
对于零长度的 RDMA READ 请求，响应方只需要发送一个操作码为 "RDMA READ Response Only" 的空数据包即可。


![](attachments/Pasted%20image%2020250804154124.png)

##### 响应方处理重复请求详解

![](attachments/Pasted%20image%2020250804154221.png)


对于不同类型重复请求：
**SEND、RESYNC 或 RDMA WRITE 请求**
- 不重新执行： 对于重复的 SEND、RESYNC 或 RDMA WRITE 请求，响应方不会再次执行请求的操作（如写入数据到内存），而只是生成相应的响应。
- 静默丢弃错误： 如果在生成响应的过程中出现错误，响应方会直接丢弃该请求，避免与后续请求的 NAK 混淆
- 保持预期 PSN 不变： 响应方不会更新其预期 PSN，继续等待与之前预期 PSN 匹配的新请求。

![](attachments/Pasted%20image%2020250804154622.png)



**RDMA READ 请求**
对于重复的 RDMA READ 请求，响应方需要重新生成响应数据。
PSN 规则：
- 第一个数据包的 PSN： 与重复请求的 PSN 相同。
- 后续数据包的 PSN： 根据原始 RDMA READ 请求的 PSN 规则递增。

错误处理：
- 返回第一个数据包之前发生错误： 静默丢弃重复请求。
- 返回部分数据包后发生错误： 中止响应，不再发送后续数据包。

如果重复请求的 PSN 范围与原始请求不重叠，则视为无效请求，直接丢弃。
响应方会按照 PSN 的顺序处理重复请求，先处理 PSN 最小的请求。
如果在处理一个重复请求时，又收到 PSN 更小的重复请求，则中断当前处理，转而先处理这个 PSN 更小的新请求。

![](attachments/Pasted%20image%2020250804154819.png)

##### 响应ACK格式

###### 普通确认数据包

==普通确认数据包（用于 SEND、RESYNC 和 RDMA WRITE）不携带有效负载字段==。

![](attachments/Pasted%20image%2020250804155841.png)


###### RDMA READ 和 ATOMIC 操作的响应

==RDMA READ 和 ATOMIC 操作的响应都携带有效负载字段==。
![](attachments/Pasted%20image%2020250804155903.png)

###### 确认数据包包含的信息
确认数据包包含以下信息：
- 用于通知请求方给定请求消息成功或失败的Symdrome字段（AETH中）。
- 请求方用于将确认消息与其未完成请求列表相关联的 PSN 值（BTH中），
- 响应方用于通知请求方请求消息已完成的消息序列号MSN(AETH中)，
- 可选的端到端流控制credits(AETH中)，
- RDMA READ 响应或 ATOMIC 操作响应中的有效负载数据。



##### 响应方生成NAK
###### 普通 NAK
- PSN：包含响应方的预期 PSN。
- 操作码：设置为 'Acknowledge'。
- AETH Sydrome：设置为 'NAK'。 用于通知请求方，收到的数据包的 PSN 与预期不符。
        
###### RNR NAK (Receive Not Ready NAK)
- PSN：指向被 RNR NAK 的数据包的 PSN。
- 操作码：设置为 'Acknowledge'。
- AETH Sydrome：设置为 'RNR NAK'。 用于通知请求方，收到一个可以接受但需要重新发送的数据包。

###### 响应方处理 NAK 后的行为
- 忽略新请求： 一旦发送了 NAK，响应方应该忽略所有新的请求（重复请求除外），直到收到一个 PSN 与预期 PSN 匹配的有效请求。

- 继续处理重复请求： 对于重复请求，响应方应该继续处理，并生成相应的响应。

- 不发送额外 NAK： 在等待有效请求的过程中，响应方不应再发送其他 NAK。

##### 响应方生成 RNR NAK
==RNR NAK (Receive Not Ready NAK) ==：是一种特殊的 NAK，它告诉请求方，接收方目前无法处理请求，但不久后可能会恢复。这通常是因为接收方的资源暂时不足，比如接收队列没有可用的 Work Queue Element (WQE)。

###### RNR NAK 的生成和发送时机
- 接收队列满： 当接收队列中的 WQE 都被占用时，响应方会发送 RNR NAK。
- 其他资源不足： 除了接收队列满，还有其他可能导致发送 RNR NAK 的情况，比如内存不足、处理器负载过高等等。

###### 请求方的处理流程

1. 接收 RNR NAK： 请求方收到 RNR NAK 后，会记录下 RNR NAK 中指定的重试间隔和当前的重试计数器(3 bit)。
2. 等待重试间隔： 请求方会等待 RNR NAK 中指定的重试间隔。
3. 检查重试计数器： 如果重试计数器不为 0，则进行重试。
4. 递减重试计数器： 每次重试后，请求方会将重试计数器减 1。
5. 重试计数器为 0： 如果重试计数器减为 0，则向上报告错误，表示重试失败。


#### 请求方： 接收响应数据包﻿

##### 接收数据包验证
接收到数据包后，请求方需要进行一系列验证以确保数据包的有效性：
- 数据包完整性验证： 检查数据包头部是否正确，任何无效的数据包都应静默丢弃。
- PSN 检查（可靠连接服务）： 验证数据包的 PSN 是否在预期的范围内，以保证数据包的顺序性。
- ACK/NAK 解析： 解析数据包中的 ACK/NAK 信息，以确认请求是否成功或需要重试。
- 目标 QPN 检查： 验证数据包的目标队列对 (QPN) 是否与本地队列对匹配。
- 操作码检查： 验证数据包的操作码是否正确。

##### 请求方对NAK消息的响应﻿
根据 NAK 类型的不同，请求方的响应也不同：

**NAK-Sequence 错误**：
- 触发自动重试。NAK 中包含的 PSN 指示了响应方期望收到的 PSN，请求方应重试发送该 PSN 的数据包。
    
- 重试计数器：为了防止无限重试，请求方维护一个 3 位重试计数器。
    
- 自动路径迁移：如果重试计数器递减到零后仍然收到 NAK-Sequence 错误，且 CA 支持自动路径迁移，则应尝试迁移路径并重新开始重试过程。
    
- 错误上报：如果多次重试后仍然无法成功发送请求，则应向上层报告错误。
    

**其他 NAK**：
可以选择重试相同的请求数据包，或者将当前 WQE 标记为完成错误。


##### 请求方没有收到确认消息的原因

请求方可能因为以下原因没有收到确认消息：
1. 响应方生成的确认消息在网络中丢失。
2. 响应方发生故障。
3. 原始请求消息在到达响应方之前丢失。

##### 请求方定时器

为了检测丢失的确认消息，请求方使用可靠连接服务时，每个发送队列都需要实现一个传输定时器。

**定时器测量**： 定时器测量以下两者中较小的值：
- 自请求方发送设置了 AckReq 位的包以来的时间。
- 自上次有效确认数据包到达以来的时间。
    

**定时器启动条件**： 只要定时器当前未运行，且满足以下条件之一，请求方就启动定时器：
-  请求方在 Send 或 RDMA WRITE 请求中设置 AckReq 位。
- 请求方生成 RDMA READ 请求。
- 请求方生成 ATOMIC Operation 请求。


**定时器重启**： 只要还有未完成的预期响应，请求方每次收到新的入站确认数据包时都会重新启动定时器。

**定时器停止**： 当没有未完成的预期响应时，定时器停止。

**超时处理**： 一旦检测到给定请求数据包的超时，请求方可以重试该请求。


#### RC
##### FMLO 标记
**FMLO 标记：** 在每个数据包的BTH头部中，有一个控制字段，用于标记该包在整个Message中的位置：
    - **First：** 属于Message的第一个包。
    - **Middle：** 属于Message的中间包。
    - **Last：** 属于Message的最后一个包。
    - **Only：** Message只有一个包。
- **作用：** 接收端通过FMLO标记来识别消息的边界，并将属于同一Message的包按顺序重新组装成完整的Message。


##### MSN（Message Sequence Number i）
对于RC服务，消息序列号 (Message Sequence Number, MSN)是由响应方返回给请求方的一个数字，指示**响应方已完成的消息数量**。MSN 包含在 AETH（确认扩展传输头）的最低有效三字节中。

向请求方提供 MSN 的作用：协助它完成 WQE（工作队列元素），通过告知请求方响应方已完成了哪些消息。

使用服务的 HCA 响应端应将其 MSN 值初始化为零。每当响应端**成功完成**处理一个新的、有效的请求消息时，它应递增其 MSN。对于重复的请求，MSN 不应递增。递增后的 MSN 应在 **RDMA READ 或 ATOMIC 响应的最后一个或仅有的数据包中返回**。对于 RDMA READ 请求，响应方可以在验证请求完成后、开始传输任何请求数据之前递增其 MSN，并可以在第一个响应数据包的 AETH 中返回递增后的 MSN。对于任何给定的请求消息，MSN 只能递增一次。

> 即： MSN 仅在响应端成功处理了一个新的、有效的请求消息后才递增。初始化为0.

由于响应端可能选择合并确认（coalesce acknowledges），一个响应包实际上可能确认了多个请求消息的完成。因此，请求端接收到递增的 MSN 后，可以批量完成其发送队列中**早于**该 MSN 的 WQE，而无需为每个 WQE 等待单独的确认，从而优化了完成处理效率。

###### SSN
从逻辑上讲，请求方会给提交到发送队列的每个 WQE 分配一个连续的**发送序列号 (Send Sequence Number, SSN)**。SSN 与响应方在每个响应数据包中返回的 MSN 之间存在一对一的关系。因此，当请求方收到响应时，它将 MSN 解释为**响应方完成的最新的请求的 SSN**，以此来确定哪些发送 WQE 可以被完成。

> **注意：** 上述描述的 SSN 仅是一个逻辑概念，用于表述 MSN 的应用方式；实现上不需要按此描述实现 SSN。

初始化后，第一个提交到发送队列的 WQE 被分配 SSN 为 1。响应方将其 MSN 计数器初始化为零。此后，响应方在完成执行一个入站请求消息时，会将其 24 位 MSN 值递增。


###### 请求方接收新 MSN 时的行为
如上所述，响应数据包中存在新的 MSN 值，可被请求方用作完成提交到其发送队列的某些 WQE的信号。

注：对于使用RC服务的 HCA 响应方，无论响应是肯定确认、否定确认还是重复确认，**MSN 计数器都应插入到 AETH 中**。


##### 消息级别的端到端流控(END-TO-END (MESSAGE LEVEL) FLOW CONTROL)

IBA（InfiniBand 架构）为RC服务提供了一种**端到端（或消息级别）的流控制能力**，响应方可利用此能力来优化其接收资源的使用。
本质上，请求方必须拥有适当的**信用 (credits)** 才能发送请求消息。

###### 信用的定义和作用

每个信用代表接收一个入站请求消息所需的接收资源。具体来说，**一个信用代表一个提交到接收队列的 WQE（工作队列元素）**。

> 注意：尽管存在接收信用，但这不一定意味着有足够的**物理内存**来接收整个入站消息。即使信用充足，仍可能遇到内存不足的情况。

**SRQ 禁用：** 
由于共享接收队列 (SRQ) 允许一组接收队列从一个公共 WQE 池中提取资源，单个接收队列无法准确估计公共 WQE 池中的 WQE 数量，因此：**任何关联了 SRQ 的 QP 都将禁用端到端流控制机制。**


###### 流控的特性与要求
- (1) 该端到端信用机制**仅适用于可靠连接 (Reliable Connected) 服务**。
    
- (2) 端到端信用由**响应方的接收队列生成**，并由**请求方的发送队列消耗**。
信用的存在与否限制了发送端传输**将消耗接收端 WQE** 的请求（即 **SEND 请求**或带有 **Immediate Data 的 RDMA WRITE 请求**）的能力。
    
- (3) HCA（主机通道适配器）的接收队列**必须**生成端到端信用（除非 QP 关联了 SRQ）
> 对于未关联 SRQ 的 QP，HCA 接收队列必须生成端到端流控制信用。如果 QP **关联了 SRQ**，HCA 接收队列不得生成端到端流控制信用。

- (4) 信用是**按每条消息**发放的，**与消息的大小无关**。
    
- (5) 端到端信用作为**一个编码后的 5 位字段**携带在 AETH 中。
    
- (6) 响应方可以使用**非请求确认包 (Unsolicited Acknowledge Packet)** 向请求方异步发送信用（通过重发最近的确认包）


###### 信用从响应方到请求方的传输 (TRANSFERRING CREDITS)
传输信用的机制有两种：
1. **附带信用 (Piggybacked Credits)：**
    
    - 在**正常的确认包**的 AETH 字段中传输信用。
        
    - 信用携带在 AETH Syndrome[4:0]`字段中。
        
    - 只有当 AETH 中的 **MSN 字段也有效**时，才能附带信用。


**非请求确认包 (Unsolicited Acknowledge Packet)：**

- 从 PSN（数据包序列号）角度来看，非请求确认消息对请求方来说像是**最近一次肯定确认消息的重复**。
    
- 即使最近的肯定确认是 `RDMA READ` 或 `Atomic` 响应，它也始终使用 **Acknowledge 操作码**。
    
- 响应方可随时发送非请求确认包。
    
- 它**仅用于传输信用**，对请求方的其他状态没有影响（因为它是重复响应）。
    
-  非请求确认包的 MSN 字段必须有效。

###### 连接协商：初始信用 (NEGOTIATING CONNECTIONS: INITIAL CREDITS)

对于建立的每个连接，**端到端流控**的使用（或不使用）是**针对每个方向单独确定的**。连接的**接收队列**的能力决定了该半连接的流控特性。


如果对端的接收队列发出信号表明它期望生成信用，则本端相应的发送队列必须遵守端到端流控规则。反之，如果接收队列发出信号表明它不会生成端到端流控信用，则相应的发送队列可以随意传输请求消息，无需考虑信用。

如果 CA 支持 SRQ（共享接收队列），并且 QP（队列对）提供RC服务并与 SRQ 关联，则该 QP 必须不生成端到端信用，并应在 `AETH Syndrome[4:0]` 字段中放置值 `5b11111`，以表明信用字段无效。

当接收队列处于 RESET（重置）状态时，传输层应将初始信用计数设置为零。一旦队列对转换到 `INITIALIZED（初始化）、RTR（接收就绪）、SQD（发送队列排水）或 RTS（就绪发送）状态`，每提交一个接收 `WQE（工作队列入口）`，它应增加其信用计数。一旦处于 RTR、SQD 或 RTS 状态，响应端可以利用非请求确认（unsolicited acknowledges）将这些信用传输给请求端。通常，非请求确认是通过重发最近发送的肯定确认包并更新信用字段来创建的。然而，在**初始化时**，由于尚未发送任何确认包，因此无法使用创建非请求确认的常规方法。因此，在初始化时，非请求确认是通过将初始 PSN 减去 “1”来创建的。因此，如果接收队列处于 RESET 状态时 PSN 初始化为 `0x000000`，那么初始非请求确认的 PSN 应为 `0xFFFFFF`。

![](attachments/Pasted%20image%2020251117144811.png)

###### 响应方计算信用的算法
对于在**未关联 SRQ** 的 QP 上使用RC服务的 HCA:
**递增 (Increment)：** 每提交一个 WQE 到接收队列，信用计数递增。
**递减 (Decrement)：** 每接收到一个消耗 WQE 的入站请求消息时，信用计数递减。
**不调整：** 接收到 `RDMA READ`、`不带 Immediate Data 的 RDMA WRITE` 或 `ATOMIC Operation` 请求时，信用计数**不调整**（因为它们不消耗接收 WQE）。

注：如果接收队列与 SRQ 关联，无论可能有多少 WQE 提交到接收队列，响应端都不调整其信用计数。
对于生成的每个确认消息（无论是普通确认消息还是非请求确认消息），接收队列应将当前编码后的信用计数（如第 378 页表 50 所示）插入到 `AETH Syndrome[4:0]`字段中。例如，如果接收队列有五个可用信用，它应在 AETH 中插入 5 位值 `b00100`。它还包括其当前 MSN 值。如果 QP 与 SRQ关联，接收队列应在 AETH 中插入 5 位值 `5b11111`。


###### 请求端行为
信用的存在与否限制了发送端传输**将消耗接收 WQE** 的请求（即 `SEND 请求`或带有 `Immediate Data 的 RDMA WRITE 请求`）的能力。
（1）当发送队列没有可用信用时，其行为应按照下面的“请求端行为 - 受限发送 WQE”的规定执行。
（2）请求端始终可以发送不消耗接收 WQE 的请求（不带 Immediate Data 的 RDMA WRITE 请求、RDMA READ 请求或 ATOMIC Operation 请求），无需考虑信用。
（3）特别是，请求端**不得在发送队列中搜索不消耗接收 WQE 的请求并打乱顺序进行传输**，也不得违反有关 `fenced WQEs`的规则。

![](attachments/Pasted%20image%2020251117144523.png)

可用信用被编码并承载在 `AETH Syndrome[4:0]` 中；`MSN` 承载在 AETH 的最低 3 个字节中。下表显示了每个有效编码信用所代表的实际信用数量。

![](attachments/Pasted%20image%2020251117151914.png)

从逻辑上讲，请求端将一个连续的发送序列号（SSN：Send Sequence Number）与提交到发送队列的每个 WQE 相关联。SSN 与响应端在每个响应包中返回的 MSN具有一一对应关系。因此，请求端将 MSN 解释为代表响应端完成的最新请求的 SSN。

![](attachments/Pasted%20image%2020251117152955.png)

请求端**每次收到包含有效信用的确认包**时，都会计算一个新的 LSN。请求端还会**动态调整 LSN**：对于它希望发送的**不消耗接收 WQE** 的每个请求（RDMA READ 请求、不带 Immediate Data 的 RDMA WRITE 请求或 ATOMIC Operation 请求），它会**将 LSN 加一**。这种调整机制允许请求端发送不消耗接收 WQE 的请求。

从逻辑上讲，MSN 加上信用计数的总和是请求端的 **限制序列号（LSN：Limit Sequence Number）**。
（1）请求端可以自由传输任何 **SSN 小于或等于**计算出的 **LSN** 的请求。
LSN 计算：LSN = MSN + Credit Value
（2）任何 **SSN 大于当前计算出的 LSN** 的请求被称为**受限（limited）**。
（3）如果响应端在 AETH 中返回“无效”代码而非信用计数，则请求端可以随意传输请求。
响应端使用“无效”代码来表明 AETH 信用计数字段无效，原因在于响应端不生成端到端信用。即使是生成端到端信用的响应端，也可以选择在 AETH 中发送“无效”代码。然而，一旦请求端从响应端收到“无效”代码，请求端可以选择忽略该连接上未来所有事务的 AETH 信用计数字段。因此，如果响应端在发出“无效”信号后又恢复返回有效信用，结果可能是不可预测的。

###### 请求端行为 - 受限发送 WQE(LIMITED SEND WQES)
下面的文字RDMA 规范中关于请求端（Requester）在信用不足时如何处理受限（Limited）WQE 的详细规则，这些规则确保了流控的遵守和事务的有序性。

当请求端在其发送队列中遇到一个**没有可用信用**的 WQE 时，该 WQE 被称为**受限 WQE**。发送队列在遇到受限 WQE 时的行为应如下：
（1）对于不消耗 WQE 的请求：
如果受限请求 WQE 是 RDMA READ 请求、不带 Immediate Data 的 RDMA WRITE 请求或 ATOMIC Operation 请求，它可以正常发送，无需考虑信用的可用性。正常的请求排序规则仍然适用（即，发送队列不得通过搜索已提交 WQE 列表来尝试查找非受限 WQE并打乱顺序发送）。
请求方发送不消耗 WQE 的请求后，会动态地将 LSN 递增 1，以允许发送下一个不消耗信用的请求。

（2）对于 SEND 请求：
发送队列最多只能传输该请求消息的一个数据包，然后必须停止并等待确认包。为确保响应方响应，请求方必须在该数据包中设置 AckReq 位。

（3）对于带 Immediate Data 的 WRITE 请求：
请求端可以在停止传输并等待确认包之前，传输整个请求消息。这是允许的，因为实际消耗接收 WQE 的是请求中包含 `Immediate Data` 的那个单个数据包。为确保响应端会生成响应，请求端应在请求消息的最后一个数据包中设置 AckReq 位。

##### 数据包的有序性保障：PSN 与 ACK/NAK
这是在RDMA传输层（Transport Layer）保证RDMA数据包正确交付的机制。

 **PSN (Packet Sequence Number) **：
- 每个 QP 有一个 Send PSN。发送每个 packet 时递增。
- 接收端维护 Expected PSN，每收到一个按序包，Expected PSN++。
接收端只接受expected PSN 的包，确保包按序。任意乱序/丢包 → NAK → 发送端重传。

 **ACK/NAK**： 接收端会对 packet 回复

|类型|含义|
|---|---|
|**ACK**|packet 收到并按序|
|**NAK: PSN sequence error**|出现乱序或丢包|
|**NAK: RNR**|receiver 没有 RECV WQE|
|**NAK: Remotely Aborted**|对端错误/qp reset|


即：`PSN + ACK/NAK = 包级别的有序性和可靠性`


整体流程：
```bash
（1）发送方（Sender）
- 硬件自动分配 PSN：PSN = qp->sq_psn++（每个 packet +1）
- 分片：若数据 > MTU → 拆分为多个 packet
    - 第一个：First
    - 中间：Middle
    - 最后一个：Last（或 Only）


（2）接收方：
- 按 PSN 顺序重组（硬件强制）
- First/Middle/Last 标识 message 边界
- 发送 ACK：
    - Acknowledge PSN = 最高连续收到的 PSN
    - 若缺失 → 发送 NAK + 缺失 PSN 范围
```

|机制|作用|
|---|---|
|**PSN 严格递增**|硬件拒绝乱序 packet|
|**ACK 携带最新 PSN**|发送方只在前一个 packet ACK 后发下一个|
|**Go-Back-N 重传**|NAK 触发从缺失 PSN 开始重传|


##### 操作/消息(Message)的完成：MSN 与 FMLO

**(1)消息/操作与包的关系：FMLO**
- Message (消息/操作)： 应用程序提交的一个完整的工作请求WR（如一次 `SEND` 或 `RDMA WRITE`）。
- 包 (Packet)： Message 被分割成的数据传输单位（不超过 MTU）。
- FMLO 标记 (First/Middle/Last/Only)：位于数据包头部（通常是 BTH 头部中的控制位），用于标记该包在整个 Message 中的边界和位置。
- 作用： 接收端通过 FMLO 标记将具有相同逻辑 Message 的所有包重新组装起来。

**（2）MSN (Message Sequence Number)**
- 定义：每个完整 Message/操作的序列号。
- 完成确认：只有当 Message 的所有包（包括带有 `Last` 或 `Only` 标记的包）都已根据 PSN 有序接收，并且 Message 被成功组装和处理后，响应端才会递增其 MSN，并携带新的 MSN 返回给请求端。
```bash
Message/操作完成的条件：
1. 所有包按序到达（由 PSN 保证）
2. 最后一个包的标志为 Last 或 Only
3. 接收端 MSN++
4. 完整消息被 RNIC 写入用户 buffer（对 recv 操作）
5. 若该 WQE 配置了 completion → CQE 产生
```

- 作用： 请求端利用 MSN 来确认其发送队列（Send Queue）中对应的 WQE（Work Queue Entry）已在远程端完成执行。

结论： MSN 和 FMLO 共同确保了一个高层操作的完整性、有序性，并用于通知操作的完成状态。


**(3) 流控与序列号的协作：SSN 与 信用/LSN**
这是将可靠性与流控限制结合起来，控制发送速率的机制。

**(3.1)SSN (Send Sequence Number)**
- 定义：请求端逻辑上为发送队列中的每个 WQE 分配的序列号。
- 关系：SSN 与 MSN 是一一对应的。
```bash
MSN_received​ 实际上就是 SSN_completed​。
```

**(3.2)信用 (Credit) 与 LSN (Limit Sequence Number)**
- 信用：响应端当前可用的接收 WQE 数量。
- LSN (限制序列号)： 请求端被允许发送的最高 SSN值，用于限制消耗 WQE的请求的发送。
- 计算关系： 请求端每次收到有效的 ACK 报文时，都会根据其中的 MSN 和信用计数来计算 LSN：
```bash
LSN=MSN_acked​+Credit_Count_received​
```
- 流控限制：请求端只能发送 SSN≤LSN 的 WQE（SEND 请求或带 Immediate Data 的 WRITE 请求）。


|**机制**|**作用对象**|**字段/位置**|**解决的问题**|
|---|---|---|---|
|**PSN**|**包 (Packet)**|BTH 头部|包的有序性、丢包检测|
|**ACK/NAK**|**包 (Packet)**|AETH 头部|包的可靠交付与错误恢复|
|**FMLO**|**包与消息边界**|BTH 头部|消息的开始、中间和结束识别|
|**MSN**|**消息 (Message)**|AETH 头部|远端操作的完成、操作的逻辑有序性|
|**SSN**|**发送 WQE**|(逻辑概念)|本地发送队列中的操作顺序|
|**信用/LSN**|**发送 WQE 数量**|AETH 头部|接收端缓冲区（WQE）的流控限制|


##### RC服务基于信用的流控机制是不是意味着RC服务基本不会有RNR NAK的发生？
###### 对 RNR NAK 的影响：极大地减少，但不能消除

 **（1）为什么 RC 信用流控能减少 RNR NAK？ **
- RNR NAK 的主要原因之一： 是接收端没有可用的 WQE 来处理入站请求。
- 信用流控的作用： RC 信用流控（LSN 机制）就是直接针对 WQE 资源的预防机制。它保证了发送方发送的消耗 WQE 的请求数量，永远不会超过接收方当前已提交的 WQE 数量。
因此，理论上，只要信用流控机制完美运行，就不会因为“WQE 不足”而触发 RNR NAK。


 **（2）为什么 RNR NAK 仍然可能发生？ **
 
正如我们之前讨论的，RNR NAK 意味着 "Receiver Not Ready"，这不仅指 WQE 数量不足，还包括更高层和状态的临时性错误，这些是信用流控无法预防的：
1》内存注册问题 (Memory Registration Issues):
    - 当发起 `RDMA WRITE` 或 `RDMA READ` 请求时，如果目标内存地址未被正确注册（MR）到 RNIC，或者注册的权限或长度不匹配，RNIC 无法将数据放置到位，此时会返回 RNR NAK。
    - 信用流控只管 WQE 的数量，不管目标内存是否有效。
 
2》QP 状态转换：
    - 接收方的 Queue Pair 处于临时转换状态（例如从 RTR 到 Error 或 Reset 的过渡），无法处理数据包。
    - 信用流控只在 QP 状态稳定时才有效。
        
3》其他资源竞争/内部处理延迟：
    - 尽管 WQE 数量足够，但接收方 RNIC 内部的其他关键资源（例如保护键、上下文、处理引擎）临时处于锁定或等待状态，导致无法立即处理数据。

**总结：** RC 信用流控消除了 "资源数量" 导致的 RNR NAK；但 "资源状态" 或 "高层错误"导致的 RNR NAK 依然需要 RNR NAK 机制来处理和恢复。

###### RC基于消息的流控的缺点

（1）如果对端RQ中WQE充足一直能接收QP数据，但上层应用不读取，会导致接收内存一直增加。
现在主要是KUCL不存在一个基于conn的软件层的流控，即没有基于连接的接收缓冲区和发送缓冲区。比如没有接收缓冲区，那么只要接收端的RQ的深度足够大，
就可以一直接收数据，放入到缓存链表中；如果应用层一直不从这个缓存链表中取数据，则这个链表会一直增长，直到，可能单个conn就会将整个进程的skb给耗尽。

类比：内核中每个`TCP socket`都有一个发送缓冲区和接收缓冲区，发送缓冲区和接收缓冲区的大小可配置。应用层调用`read`会从接收缓冲区中读取数据。

（2）QP粗粒度级别：在VRC、VQP这种共享QP的模式下，QP粒度的流控会导致连接不公平的问题，某一个连接过度消耗RQ中的WQE会导致所有的连接受影响。

（3）SRQ不支持。一是因为SRQ中的WQE为共享资源，无法向共享SRQ的每个SQ分配一个准确的credit值。二是因为SRQ中并不维护对端SQ的信息，缺乏向SQ主动发送credit的机制。

（4）队头阻塞问题：受限制的SEND操作会阻塞后续的WRITE、READ操作。即使WRITE/READ不消耗WQE，也无法绕过前面受限的SEND。


###### 小结
有了 RC 消息级信用流控，确实会**极大的减少** RNR NAK 的发生，但不能完全消除 RNR NAK，并且仍然可能产生其他类型的 NAK（PSN error、remote access error 等）。

```bash
(1) 序列号错误 NAK (Sequence Error NAK):
发送端发送的数据包在网络中丢失、乱序，导致接收端收到的 PSN 不连续。

(2) 访问错误 NAK (Access Error NAK):
接收端收到合法的 RDMA 请求（如 `READ` 或 `WRITE`），但发现请求的 内存访问权限（RKEY/Protection Key）是错误的或无效的。

```

##### RC中基于消息的流控和PFC、DCQCN、ECN、以及IB链路层基于信用的流控的关系

|**机制名称**|**传输协议**|**工作层次**|**解决的问题焦点**|**流控对象/单位**|**机制类型**|**粒度**|
|---|---|---|---|---|---|---|
|**1. RC 消息级信用流控 (End-to-End)**|RoCEv2/IB (RC)|**传输层 (RDMA/L4)**|**接收端资源溢出**（WQE/缓冲区）。|**WQE/Message 数量**|**预防性**流控 (端到端)|**粒度是操作/消息 (WQE)：** 它关心的是**请求的数量**，即接收端能处理多少个新的操作。|
|**2. PFC (Priority Flow Control)**|Ethernet (L2)|**数据链路层 (L2)**|**链路层队列溢出**（交换机端口）。|**整个端口/优先级队列**|**阻塞性**流控 (逐跳)|**粒度是优先级队列的缓冲区：** 关心的是**流量的物理承载能力**，单位是帧/包的物理尺寸。|
|**3. DCQCN/ECN**|RoCEv2/IP (L3/L4)|**网络层/传输层 (L3/L4)**|**网络拥塞**（交换机队列排队）。|**发送速率 (Rate)**|**速率控制** (端到端)|**粒度是发送速率：** 它关心的是**每秒发送的字节数**（例如 10 Gbps）。|
|**4. IB 链路层信用流控**|InfiniBand (L2)|**数据链路层 (L2)**|**IB 链路缓冲资源**（交换机缓冲区）。|**虚拟通道 (VL) 缓冲区**|**预防性**流控 (逐跳)|**粒度是虚拟通道 (VL)** 缓冲区。关心的是**流量的物理承载能力**，单位是帧/包的物理尺寸。|


|层级|机制|属于哪种网络|控制粒度|触发条件|主要作用|是否必须|与 RC 消息级流控的关系|
|---|---|---|---|---|---|---|---|
|**L1 物理/链路层**|**IB 链路层 Credit-based Flow Control**|仅 InfiniBand|**每个 packet / FLIT**|发送前检查 VL credit|**防止链路 buffer 溢出**（无损）|IB 必须|**完全独立**，IB 天然无损，RC 消息级流控只是额外一层|
|**L2 无损以太层**|**PFC（Priority Flow Control）**|RoCEv2|**每个优先级（通常 TC 3）**|交换机队列超过高水位 → 发送 Pause/Resume|**防止 RoCE 队列溢出，实现物理无损**|RoCEv2 生产必开|**必要前提**：没有 PFC，DCQCN 和 RC 消息级流控都会频繁触发重传|
|**L3 拥塞控制层**|**ECN 标记 + DCQCN（CNP + 速率调整）**|RoCEv2（推荐） IB 可选（FECN/BECN）|**流（QP）级别速率**|交换机队列超过 ECN 阈值 → 标记 IP header|**端到端拥塞避免**，防止长时间排队|RoCEv2 生产必开|**互补**：DCQCN 控制发送速率，RC 消息级流控控制发送窗口，二者一起决定还能发多少|
|**L4 RDMA 传输层**|**RC 消息级流控（MSN + Credit）**|RC QP（IB 和 RoCEv2 都支持）|**每个 Message（WR）**|接收方 RQ 可用 WR 数（credit）|**防止接收 RQ 耗尽**，保证接收方能处理完整个 message|RC 必须|**最高层**，最终决定还能 post_send 多少个 WR|


###### 消息级信用流控 (RC End-to-End Flow Control)
- **机制核心：** 基于 MSN/LSN 和 WQE 信用。
- **目标：** 确保发送方发送的消耗 WQE 的请求数量（如 `SEND`）不超过接收方当前可用的 WQE 数量。
- **工作方式：** 接收方通过 ACK 报文返回 (MSN + 信用计数)，发送方计算 LSN。一旦请求的 SSN 超过 LSN，发送即受限。
- **关系：**
    - 与 PFC/ECN **独立**：它不关心网络链路是否拥塞，只关心终端主机的资源。
    - 与 IB 链路层信用流控**独立**：L4 信用解决的是主机 WQE 资源，L2 信用解决的是链路缓冲区资源。


###### PFC (Priority Flow Control)

- **机制核心：** IEEE 802.1Qbb 协议。
    
- **目标：** 在拥塞发生时，通过发送 **PAUSE 帧**，在 L2 层面暂停**特定优先级**的流量，从而**防止交换机缓存溢出**。
    
- **工作方式：** **逐跳（Hop-by-Hop）阻塞。** 交换机发现队列满，向**上游设备**发送 PAUSE 帧，使其停止发送该优先级的流量。
    
- **关系：** PFC 是**阻塞性的**、**逐跳的** L2 机制，用于**解决 RoCEv2 网络的丢包问题**。它与端到端 L4 信用流控的目的完全不同，但两者可以并存。


###### DCQCN/ECN (Quantized Congestion Notification)

- **机制核心：** 基于 **ECN 标记**和 **CNP 报文**。
    
- **目标：** 解决**网络拥塞**（交换机排队延迟）问题，**动态控制发送速率**。
    
- **工作方式：**
    
    1. **交换机 (ECN):** 队列排队超过阈值时，对数据包设置 ECN 标记。
        
    2. **接收端 (CNP):** 收到 ECN 标记包后，生成 CNP 报文反馈给发送端。
        
    3. **发送端 (速率控制):** 收到 CNP 后，乘法减小发送速率。
        
- **关系：** DCQCN 是一种**端到端**的**拥塞控制机制**，它与 RC 消息级信用流控**独立**。前者控制**速率**，后者控制**数量**。


###### IB 链路层信用流控 (InfiniBand Link Layer Flow Control)

- **机制核心：** 基于 VL（Virtual Lane，虚拟通道）缓冲区信用。
    
- **目标：** 确保 IB 交换机端口的输入缓冲区不被淹没，防止 L2 链路丢包。
    
- **工作方式：** 逐跳（Hop-by-Hop）预防。交换机在转发数据前，必须确保下游端口有足够的信用。当下游收到数据并释放缓冲区后，会返回 **Credit Update** 报文给上游，以补充信用。
    
- **关系：** 这是 **InfiniBand** 独有的**无损网络**实现机制，与 RoCEv2 中使用的 PFC 作用类似（都是 L2 逐跳流控），但实现方式不同。它与 RC 消息级信用流控（L4）是分层独立协作的关系。


###### 小结

这四种机制在 RDMA 网络中扮演了**分层管理**的角色：

3. **链路层 (L2)：** **PFC 或 IB 链路信用** 负责**逐跳**防止**链路和交换机缓冲区溢出**，是实现无损通信（低延迟 RDMA 的基础）的关键。
    
4. **传输层流控 (L4 End-to-End)：** **RC 消息级信用流控** 负责**端到端**防止**终端主机（接收方）的 WQE 资源溢出**。
    
5. **拥塞控制 (L3/L4)：** **DCQCN/ECN** 负责**端到端**调整**发送速率**，以适应**网络拥塞**（队列排队）。


所有机制必须协同工作才能确保 RDMA 的高性能：
6. 当发送方要发送一个消耗 WQE 的请求时，它必须首先满足 **RC 消息级信用流控** (SSN≤LSN) 的要求（L4）。
7. 请求包进入网络后，在通过每个交换机时，必须遵守 **PFC 或 IB 链路层信用流控** 的要求，以确保无损传输（L2）。
8. 在整个传输过程中，如果交换机队列开始积压，**ECN** 会标记拥塞。接收端将反馈 **CNP**，使发送端降低发送**速率**（DCQCN），从而减缓所有包的流入（L3/L4）。


##### 小结
所有 packet 按 PSN 顺序到达，message 按 MSN 顺序完成；
数据包的有序保证：PSN 严格递增 + ACK/NAK 确认；
操作/message完成 = 最后一个 packet 的 ACK 返回 + CQE 生成；
Message = 多个 packet：一个 WQE 会产生一个 message，一个 Message（WR）可拆分为多个 Packet，每个 Packet 有 PSN；
MSN（Message Level）：MSN 是针对完整的 RDMA 操作/消息的序列号，与 PSN（Packet Level，包序列号）是分开的。


#### RD


#### XRC




### 不可靠服务
**两种类型**： 不可靠连接（UC:  SEND, RDMA WRITE操作）和不可靠数据报（UD: 仅 SEND操作）。

#### 特点
**无确认**： 请求方不接收消息送达的确认。这意味着请求方无法知道消息是否成功送达。
    
**无序性**： 不保证数据包的传输顺序。接收方可能以与发送方发送顺序不同的顺序接收数据包。
    
**基本验证**： 响应方仍然会对传入的数据包进行基本的验证，例如头部字段和 CRC 校验。
    
**错误处理**： 即使检测到错误（如丢包或乱序），响应方也会继续接收后续的数据包，而不会停止。
    
**响应方完成标准**： 响应方在按正确顺序接收到完整消息、有效性检查完成后，认为操作完成。
    
**请求方完成标准**： 请求方在“Last”或“Only”的数据包发出后，认为操作完成。


![](attachments/Pasted%20image%2020250804164645.png)

#### 不可靠数据报服务（UD）

**通信方式**： UD 允许一个源 QP 将消息发送到多个目标 QP 中的任何一个，这些目标 QP 可以位于相同或不同的目标终端节点上。这提供了一种灵活的广播或多播通信方式。

**消息大小限制**： UD 消息必须限制在单个数据包内。

**PMTU 限制**： 该数据包的大小不能超过源和目标之间的 PMTU，否则会被丢弃。PMTU 是指在网络路径上可以传输的最大数据包大小，超过此大小的数据包需要分片，而 UD 不支持分片。

##### 请求方﻿

**PSN 生成**： 除 QP0 和 QP1 外，每个请求消息的 PSN 递增 1（模 2的24次方 ）。初始 PSN 值在发送队列初始化时由客户端加载。PSN 仅在发送队列处于“就绪发送”状态时更新。

**消息发送完成**： 当最后一个字节（VCRC 字段）提交到线路且未检测到本地错误，或检测到本地错误并终止发送时，认为消息发送完成。


##### 响应方﻿

**PSN 验证**： 响应方 可以 忽略 PSN 字段。虽然可以实现 PSN 验证以检测乱序数据包，但这不是强制要求。
    
**长度验证**： 必须验证 LRH 的“数据包长度”和 BTH 的 PadCnt，以确保接收缓冲区有足够的空间，并且数据包大小在 PMTU 范围内。长度无效的数据包会被静默丢弃。
    
**OpCode 验证**： 必须验证 BTH:OpCode 是否受接收队列支持且不是无效的reserved值。OpCode 无效的请求会被静默丢弃。
    
**本地操作验证**： 本地错误（如内存转换错误）可能导致请求失败。
    
**消息接收完成**： 当消息有效负载无错误地提交到本地，并且所有有效性检查（包括 CRC）都成功完成时，认为消息接收完成。错误可能导致 WQE 以错误状态完成，也可能不消耗任何 WQE。


### 错误检测和处理机制
IBA 采用分层错误管理架构 (LEMA)，每一层负责检测和管理自身层的错误。传输层将错误报告给其客户端（对于 HCA 来说，通过 CQE 在 CQ 上报告）。TCA 可以自行决定是否报告错误。


错误报告方式： HCA 通过 verbs 层报告三种错误：立即错误、完成错误和异步错误。传输层只能报告完成错误或异步错误。完成错误又分为接口检查和处理错误。

#### 错误分类
##### 请求方侧错误
**本地检测到的错误**： 由请求方自身检测到的错误，例如本地内存访问错误、超时或过度重试。请求方会停止受影响 QP 的传输，存储错误状态，完成之前的 WQE，并根据错误类型将 QP 置于错误状态。
    
**远程检测到的错误**： 由响应方检测到并通过 NAK 报告给请求方的错误，仅适用于可靠服务。响应方通过 NAK 代码报告远程错误。NAK 序列错误和 NAK-RNR 指示请求方应自动重试的操作。其他 NAK 代码指示必须立即报告给请求方客户端且无法重试的故障。

##### 响应方侧错误
响应方侧只有本地检测到的错误。响应方根据服务类型和具体错误，决定是否将错误报告给请求方或本地客户端。

#### 请求方的错误处理
**A 类错误**：可恢复的错误。
- 包括：数据包序列错误、隐含 NAK 序列错误、本地 Ack 超时错误和 RNR NAK。
    
- 处理方式：通过重试机制恢复。每次重试递减相应的重试计数器（一个用于数据包序列错误和本地 Ack 超时，另一个用于 RNR NAK）。只有当重试计数器过期时，才会将错误报告给客户端。

**B 类错误**：需要重置连接或 EE 上下文的错误。
- 处理方式：对于可靠数据报以外的服务，以错误状态完成当前 WQE，将发送队列置于错误状态，并将所有后续 WQE 标记为flushed。对于可靠数据报服务，执行 RESYNC 过程，如果仍然出错，则执行与非可靠数据报服务相同的操作。HCA 将错误发布为“Completion - 处理类型”，并带有相应的错误类型。失败的 WQE 之后的所有 WQE 也以“Completed - Flushed in Errors”状态完成。处于错误状态的发送队列会静默丢弃所有到达的确认消息。
    

**C 类错误**：无法与特定 WQE 关联的错误。
- 处理方式：如果可以与 QP 关联，则将发送队列置于错误状态，所有未完成的 WQE 都以“Completed - Flushed in Errors”状态完成。如果可以与 EE 上下文关联（仅限可靠数据报），则其发送侧置于错误状态，对于 HCA，发布的错误是“Affiliated Asynchronous错误”。如果无法与任何资源关联，则可能无法报告该错误。
    

**D 类错误**：仅发生在可靠数据报服务中的错误。
- 处理方式：将请求方的 EE 上下文置于错误状态，终止当前消息传输，向当前调度的发送队列发出错误信号，并取消其排队。EE 上下文继续将其他请求服务的发送队列置于错误状态。每个发送队列 (QP) 的行为都像 B 类错误。
    

**E 类错误**：收到 PSN 不匹配的确认消息的错误。
- 处理方式：丢弃确认消息。例外情况是，对于可靠连接服务，如果 PSN 比预期 PSN 小 1，则需要恢复端到端流控制credits并丢弃消息的其余部分（“未经请求的确认”）。此类错误不会报告给上层。
    

**F 类错误**：CQ 无法访问或已满导致的错误。
- 处理方式：受影响的 QP 移动到错误状态，并生成Affiliated Asynchronous错误。当前 WQE 和所有后续 WQE 处于未知状态。




#### 响应方的错误处理

**错误分类和处理依据**： Responder 端的错误处理依据包括是否向本地客户端报告错误、是否通过 NAK 代码向请求方报告错误，以及是否从接收队列中消耗 WQE。

**错误类别（A-M）**： 针对不同类型的错误，Responder 采取不同的应对措施。下表对这些类别进行了总结：

|错误类别|描述|是否报告给客户端|是否发送 NAK|是否消耗 WQE|影响其他|备注|
|---|---|---|---|---|---|---|
|A|与接收 QP 相关的错误（WQE 格式错误等）|是|是|是|共享 SRQ|对于共享 SRQ，会影响其他 QP 的 WQE。|
|B|需要报告给请求方但不报告给本地客户端的错误|否|是|否|无|仅限可靠服务。|
|C|可以与 QP 关联但无法与特定 WQE 关联的错误|是|是|是|无|如果无法与 WQE 关联，则作为异步错误报告；否则，QP 进入错误状态，并作为完成错误报告。|
|D|静默丢弃数据包的错误|否|否|否|无||
|D1|仅在不可靠连接模式下发生的错误|否|否|否|无|如果 BTH 操作码为“first”或“only”，则视为新消息的开始。|
|E|不影响接收队列继续接收消息的错误|否|否|否|共享 SRQ|对于共享 SRQ，会影响同一 QP 已获取的 WQE。|
|F|针对 RD 服务的错误，既向请求方返回 NAK 代码，又以错误状态完成 WQE。|是|是|是|无||
|G|CQ 无法访问或已满时发生的错误|是|否|否|无|受影响的 QP 进入错误状态。|
|H|与接收 EEC 相关的本地错误|是|否|是|无||
|J|仅针对 RC QP 发生的本地检测到的错误，与 SEND with Invalidate 相关。|是|否|是|无|先执行 SEND 操作，再处理 Invalidate 错误。|
|K|针对 HCA XRC TGT QP 的错误。|是|是|是|XRC SRQ|影响相同或不同 XRC SRQ 上的其他 WQE。|
|L|当 CQ 无法访问或已满，且尝试在 XRC SRQ 上完成 WQE 时发生的错误。|是|否|否|XRC SRQ|影响 XRC SRQ 上的 WQE 和其他 XRC SRQ 上的 WQE。|
|M|仅针对 XRC TGT QP 发生的本地检测到的错误，与 SEND with Invalidate 相关。|是|否|是|XRC SRQ|先执行 SEND 操作，再处理 Invalidate 错误，影响不同 XRC SRQ 上的其他 WQE。|


**共享接收队列 (SRQ) 的影响**： 一些错误类别（如 A、E、K、L、M）的处理会涉及到共享接收队列，并可能影响其他 QP 或 WQE。
    
**错误报告方式**： 对于 HCA 来说，错误通常通过 verbs 层以“完成错误”或“关联的异步错误”的方式报告。
    
**错误优先级**： 某些错误（如 J、M）存在优先级关系，例如与 SEND with Invalidate 相关的错误，会优先处理 SEND 操作相关的错误。



# RoCE和RoCEv2协议

## RoCE协议(RoCEv1)
基于==融合以太网的 RDMA== (RoCE) 是 InfiniBand 贸易协会标准，旨在以太网网络上提供 InfiniBand 传输服务。

==RoCE(RoCEv1)保留了 InfiniBand Verbs 语义及其传输和网络协议，并用以太网的链路层和物理层取代了 InfiniBand 的链路层和物理层==。

![](attachments/Pasted%20image%2020250801142320.png)

### RoCE 的局限性
由于 RoCE 数据包不包含 IP 标头，因此无法通过传统的 IP 路由器在不同的以太网 L2 子网之间进行路由。这意味着 RoCE 只能在同一个 L2 广播域内工作。

## RoCEv2
RoCEv2 是 RoCE 的扩展，解决了 RoCE 无法跨子网路由的问题：

![](attachments/Pasted%20image%2020250801142738.png)

- 使用 IP 标头进行 L3 路由。
    
- 使用 UDP 标头作为 RDMA 传输协议数据包的无状态封装。
    
- 使用固定的 UDP 目标端口 (dport) 来区分 RoCEv2 数据包。
    
- UDP 源端口 (sport) 用作流标识符，用于网络基础设施的包转发优化（例如 ECMP）。


### IP 报头规范

```bash
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |Version|  IHL  |Type of Service|          Total Length         |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |         Identification        |Flags|      Fragment Offset    |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |  Time to Live | Protocol 41   |         Header Checksum       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                       Source Address                          |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                    Destination Address                        |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                    Options                    |    Padding    |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |            IPv6 header and payload ...              /
     +-------+-------+-------+-------+-------+------+------+
```

![](attachments/Pasted%20image%2020250801145525.png)

|字段名称 (英文)|IPv4 取值|IPv6 取值|备注|
|---|---|---|---|
|Ethertype|0x0800|0x86DD||
|IHL|5|N/A||
|DSCP|取决于 Address Vector 中的 "Traffic Class"|取决于 Address Vector 中的 "Traffic Class"||
|ECN|'00' (不支持拥塞管理) 或 '01'/'10' (支持拥塞管理)|'00' (不支持拥塞管理) 或 '01'/'10' (支持拥塞管理)|用于拥塞控制|
|Total Length|包括 IPv4 报头和 ICRC 的完整数据包长度|N/A||
|Payload Length|N/A|从 BTH 到 ICRC 的长度||
|Flags|'010' (不分片)|N/A||
|Fragment Offset|0|N/A||
|TTL / Hop Limit|取决于 Address Vector 中的“Hop Limit”|取决于 Address Vector 中的“Hop Limit”||
|Protocol / Next Header|0x11 (UDP)|0x11 (UDP)||
|Source/Destination IP|取决于Address Vector 中的SGID/DGID|取决于Address Vector 中的SGID/DGID||


### UDP头规范

![](attachments/Pasted%20image%2020250801150013.png)

### RNIC对于RoCEv2报文和普通报文的识别和处理

RoCEv2 包是 UDP 封装的，因此 RNIC 通过识别特定 UDP 端口号（默认是 4791）来区分 RoCEv2 包和普通包。**
**（1）识别规则**：
- UDP 协议；
- 目标 UDP 端口为 4791（或配置的 RoCEv2 端口）；
- 可选检查 BTH 头是否合法（增强判断）；

**（2）识别后处理**：
一旦识别为 RoCEv2：
- RNIC 将 UDP/IP/Ethernet 层头部剥离；
- 按照 IB 协议解析 BTH 头；
- 检查 Queue Pair (QP)、PSN 等；
- 将 payload DMA 到预注册的内存区域；
- 触发 CQE（Completion Queue Entry）；

**（3）普通的包举例**：
- TCP 包
- UDP 非 4791 端口的包
- ICMP、ARP、其他协议

**（4）普通的包的处理**：
这些包会被 NIC 直接交给主机网络协议栈，不会触发 RNIC 的 RDMA 引擎处理。




##### ⛔ **不符合以上条件的包：**

- 会被当作普通以太网包；
    
- 交由内核网络栈处理（如 UDP/TCP/IP）。

### RoCEv2 中 ICRC 计算方法

CRC 提供端到端的数据完整性保护。

**ICRC 计算方法**： 基于RoCE和InfiniBand的ICRC计算方法，但有以下关键修改：
- 计算初始值为64位 1。
- 覆盖整个IP数据报，从IP报头开始到IB负载的最后一个字节（不包括ICRC本身）。
- 为了防止中间节点修改某些字段影响ICRC校验，计算ICRC时将以下字段替换为 1：
    IPv4：TTL、Header Checksum、Type of Service (DSCP 和 ECN)
    IPv6：Traffic Class (DSCP 和 ECN)、Flow Label、Hop Limit
- UDP Checksum字段在ICRC计算中也替换为 1。


### RoCEv2 L2/L3 地址
#### L3 地址 (网络层地址)﻿

- RoCEv2 使用 GID 来表示 L3 地址，这与 InfiniBand 的概念一致。
    
- 对于 IPv6 的 RoCEv2，SGID 和 DGID 直接对应于 IPv6 的 SIP 和 DIP。
    
- 对于 IPv4 的 RoCEv2，IPv4 地址被映射到 IPv6 地址格式（::ffff:<IPv4 地址>），然后用作 SGID 和 DGID。这种映射方式使得RoCEv2可以使用统一的GID寻址方式，无论底层是IPv4还是IPv6。


#### L2 地址 (链路层地址)﻿﻿

InfiniBand 规范中提到的 LRH（Local Route Header，本地路由报头）及其字段不适用于 RoCEv2 端口。因为RoCEv2运行在以太网上，使用以太网的报头格式，而不是InfiniBand的LRH。

### GID Table﻿

每个 RoCEv2 端口维护一个端口 GID 表，其中包含已配置给该端口的所有 L3 地址。GID 表中的地址类型可以是 IPv4、IPv6 或 IB GID，每个 GID 表条目都有一个“GID type”属性，明确指出地址类型， 出站数据包的协议选择取决于 GID type。 软件负责维护 GID 表，通常通过与操作系统交互完成。

### RoCEv2 中的不可靠数据报（UD）
在RoCEv2中，UD在完成队列条目 (CQE) 的结构以及 L3 报头在接收缓冲区中的存储方式上有所不同。

- CQE： RoCEv2 的 UD CQE 除了包含其他信息外，还包含源 L2 地址以及一个指示数据包类型的标志（IPv4、IPv6 或 RoCE）。
    
- L3 报头： 接收缓冲区的前 40 字节用于存储 L3 报头。 IPv6 报头占用全部 40 字节。 IPv4 报头占用后 20 字节（偏移量 20），前 20 字节内容未定义。

### 其他
**拥塞控制 (Congestion Control)**： InfiniBand 的拥塞控制机制不适用于 RoCEv2。RoCEv2 有其自身的拥塞管理机制。BTH 中的 BECN 和 FECN 位在 RoCEv2 中未使用并应被忽略。

**QoS (Quality of Service)**： InfiniBand 的 QoS 管理基于 InfiniBand 链路层功能，因此不适用于 RoCEv2。 RoCEv2 可以利用 IP/以太网的 QoS 机制。在融合网络中，QoS 对于 RoCEv2 尤其重要。QoS 的实现通过 Fabric 中的相关机制（例如 802.1Qaz 的 ETS）以及标准的 IP/以太网数据包/流识别方法（DSCP/802.1Q）和地址向量中的Traffic Class参数来控制。

**无损网络 (Lossless Network)**： RoCEv2 推荐使用无损网络以避免因 Fabric 争用导致的数据包丢失。这通常通过链路层流控（例如 802.1Qbb）实现。即使是“无损”网络也可能发生少量丢包，RoCEv2 自身有重传机制处理。RoCEv2 流量应与其他流量（例如 TCP）使用不同的优先级，以避免相互影响。

**拥塞管理 (Congestion Management)**： RoCEv2 拥塞管理 (RCM) 旨在避免拥塞热点并优化吞吐量。RCM 使用 RFC3168 (ECN) 进行拥塞信令。当网络设备检测到 RoCEv2 流量的拥塞时，会在 IP 报头中标记 ECN 字段。目标端节点会将此信息反馈给源端，源端则会降低发送速率。RCM 是可选的。

**RoCEv2 不支持 InfiniBand 的 RAW Datagram**

## RoCEv2 数据包的发送和接收
### 数据准备与传输发起
首先，用户需要在发送端执行以下步骤，为数据传输做好准备：

**内存注册**: 用户通过 Verbs API （如 `ibv_reg_mr()` ）注册一段内存区域，用于存放待发送的数据。
    
**创建完成队列 (CQ)**: 用户创建一个完成队列 (Completion Queue, CQ)，用于接收完成通知。
    
**创建队列对 (QP)**: 用户创建一个队列对 (Queue Pair, QP)，用于发送和接收数据。将QP转换为INIT状态。
    
**转换QP状态**: 将接收端QP状态从INIT转换为RTR（Ready to Receive）状态，设置QP Number、rq_psn等以便接收端能够接收数据。将发送端QP状态从INIT转换为RTS（Ready to Send）状态，设置sq_psn等以便发送数据。
    
**数据写入**: 用户将待发送的数据写入已注册的内存区域。
    
**工作请求 (WR) 创建**: 用户创建一个工作请求 (Work Request, WR)，其中包含操作类型（如 SEND、RDMA WRITE 或 RDMA READ）、数据长度、目标地址等信息。
    
**发送队列入队**: 用户调用 `ibv_post_send()` 函数，将创建的 WR 放入 QP 的 SQ 中。
    
**硬件处理**:
- Doorbell 通知: 软件向硬件发送一个 "doorbell" 通知，指示 SQ 中有新的 WR 需要处理。
	
- WQE 读取: 硬件通过直接内存访问 (DMA) 从 SQ 中读取工作队列项 (Work Queue Entry, WQE)。
	
- 数据读取: 硬件根据 WQE 中的信息，通过 DMA 从用户注册的内存区域读取待发送的数据。
	
- 数据封装: 硬件将读取的数据封装成 RoCEv2 数据包，其中包括必要的 RoCEv2 头部信息和数据载荷。




### RoCEv2 数据包结构

RoCEv2 数据包本质上是==在以太网帧中封装了 InfiniBand 数据包==，其主要结构如下：

**以太网帧头 (Ethernet Header)** ：标准以太网帧头，包含目标 MAC 地址、源 MAC 地址和以太网类型 (EtherType) 等信息。对于 RoCEv2，EtherType 通常设置为 0x0800。
    
**IP 头部 (IP Header)** ：包含源 IP 地址、目标 IP 地址和协议类型等信息。对于 RoCEv2，协议类型通常设置为 UDP。
    
**UDP 头部 (UDP Header)** ：包含源 UDP 端口和目标 UDP 端口。RoCEv2 通常使用 4791 端口。
    
**InfiniBand 基本传输头 (Base Transport Header, BTH)**： 包含数据包的基本传输信息。
    
**InfiniBand 扩展传输头 (Extended Transport Header, ETH)** ：根据操作类型，可能包含 RDETH, DETH, ImmDT, AETH 等不同的扩展头部。
    
**数据载荷 (Payload)** ：实际传输的数据内容。
    
**循环冗余校验 (Cyclic Redundancy Check, CRC)**：用于数据完整性校验。

以下是根据不同的操作类型 (SEND, WRITE, READ) 详细说明 BTH 和 ETH 的字段：

#### SEND 操作

![](attachments/Pasted%20image%2020250804180954.png)


##### SEND BTH 字段

**Opcode**: 指示操作类型，对于 SEND 操作，可能为 SEND Only、SEND First、SEND Middle、SEND Last 或 SEND Last with Immediate 等。

**SE (Solicited Event)**: 发送端在最后一个数据包中将其设置为 1，表示接收方应生成一个完成队列元素 (Completion Queue Entry, CQE)。

**M (Migration)**: 表示路径迁移请求，在 RoCEv2 中不使用，设置为 0。

**PADCNT (Padding Count)**: 用于填充数据包，确保数据载荷长度是 4 字节的倍数。

**TVER (Transport Version)**: 传输版本号，在 RoCEv2 中默认为 0。

**P_KEY (Partition Key)**: 分区密钥，用于区分不同的分区。

**DEST_QP (Destination Queue Pair)**: 目标 QP 的编号。

**FECN/RES1/BECN/RES2 (Forward Explicit Congestion Notification, Backward Explicit Congestion Notification)**: 拥塞控制字段，RoCEv2 不使用，设置为 0。

**ACK_REQ (Acknowledge Request)**: 指示是否需要接收方返回 ACK，根据数据包类型设置为 1 或 0。

**PSN (Packet Sequence Number)**: 数据包序列号，用于保证数据包的顺序。



##### SEND ETH 字段
**DETH (Datagram Extended Transport Header)** ：对于数据报服务（例如 Unreliable Datagram），在 BTH 之后需要加上 DETH，其中包含 Q_Key 和 SRCQP 等信息。

**ImmDt (Immediate Data)**: 对于带有 Immediate Data 的 SEND 操作（SEND Only with IMM 或 SEND Last with IMM），需要附加 4 字节的 Immediate Data。

#### RDMA WRITE 操作
![](attachments/Pasted%20image%2020250804193742.png)

##### WRITE BTH 字段
    
**Opcode**: 指示操作类型，对于 RDMA WRITE 操作，可能为 RDMA WRITE Only、RDMA WRITE First、RDMA WRITE Middle、RDMA WRITE Last 或 RDMA WRITE Last with Immediate 等。
	
**SE (Solicited Event)** ：发送端通常将其设置为 0，但在带有 Immediate Data 的 RDMA WRITE 操作（RDMA WRITE Last with Immediate）的最后一个数据包中设置为 1。
	
其他字段与 SEND BTH 类似。
        
##### WRITE ETH 字段
**RETH (RDMA Extended Transport Header)**: 在 RDMA WRITE 操作的首个数据包的 BTH 之后，需要加上 RETH。RETH 中包含远程内存地址的起始地址、用于访问控制的 R_Key、还有指示远程 DMA 操作的长度。
        
#####  ImmDt (Immediate Data)
对于带有 Immediate Data 的 RDMA WRITE 操作（RDMA WRITE Last with IMM），需要附加 4 字节的 Immediate Data。


#### RDMA READ 操作

![](attachments/Pasted%20image%2020250804193918.png)

##### READ Request BTH 和 RETH 字段
**READ Request BTH字段**：
- Opcode: 设置为 RDMA READ Request。
- SE: 发送端设置为 0。
- 其他字段与 SEND BTH 类似。

**RETH (RDMA Extended Transport Header)**: 
RETH 中包含远程内存地址的起始地址、用于访问控制的 R_Key、还有指示远程 DMA 操作的长度。

##### READ Response BTH 和 RETH 字段
**READ Response BTH 字段**
- Opcode: 设置为 RDMA READ Response Only、RDMA READ Response First、RDMA READ Response Middle 或 RDMA READ Response Last。
- SE: 设置为 0。
- 其他字段与 SEND BTH 类似。


**READ Response ETH 字段**
AETH (Acknowledge Extended Transport Header)： 在 RDMA READ Response First/Only/Last 数据包的 BTH 之后，需要加上 AETH。AETH 中包含了肯定或否定的应答信息，以及应答的消息序号（填入 READ Request 中的 PSN 号）。

### RoCEv2 数据包的传输与接收
**网络传输**: 硬件将封装好的 RoCEv2 数据包通过以太网发送到接收端。

**接收端处理**:
- 数据包解析: 接收端硬件解析收到的 RoCEv2 数据包，提取 BTH、ETH、数据载荷等信息。
- 数据写入: 硬件根据数据包中的信息，通过 DMA 将数据载荷写入接收端已注册的内存区域。
- CQE 生成: 如果请求了 Solicited Event (SE)，接收端硬件会在完成数据写入后，向完成队列 (Completion Queue, CQ) 中生成一个 CQE，通知用户操作已完成。

**应答**: 对于可靠传输的服务（Reliable Connection，RDMA Read等），响应方需要回复一个应答报文，用来确认报文的接收，完成此次数据传输。



# 参考
```bash
http://pnet-maning10-03.dev.kwaidc.com/rocev2.html#rocev2

# IB规范1.4
http://47.92.214.21:8888/rdma/IB%20Specification%20Vol%201-Release-1.4-2020-04-07_ib_spec_vol1.pdf
```