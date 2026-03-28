```table-of-contents
```
# pause Frame

## 作用
PAUSE帧位于网络报文协议中的数据链路层（详细点讲应该是**数据链路层中的MAC控制子层**）。

- 发送Pause
交换控制电路要防止缓冲区溢出，可以利用MAC控制子层来控制以太网介质访问控制子层的操作。当已用缓冲区容量达到一个预先设定的阈值时，端口向全双工链路对方发出停止发送数据的请求，这个请求通过MAC控制子层产生的控制帧实现。

- 接受Pause
同样，端口可以接收由其他站点MAC控制子层产生的控制帧，控制帧夹在客户数据帧流中发送，接收方会根据帧的内容将控制帧分离出来，提交到MAC控制子层中的流量控制模块，流量控制模块解析控制帧的内容，提取帧中的控制参数，根据控制参数决定暂停发送的时间。

## 格式
PAUSE帧的帧长为64字节，PAUSE帧格式如下：
![](attachments/Pasted%20image%2020230802145033.png)

MAC控制帧在网络上的发送和接收与数据帧类似，除了前导码和帧开始符外，**长度为以太网帧的最小帧长度(64字节)**。
MAC控制帧的数据域内，前两个字节标识了MAC控制的操作码，表示帧请求的控制功能。目前协议只定义了一种操作代码，即PAUSE操作，操作码为0x0001。
操作码后是操作所需的参数，参数只用了数据字段的2个字节，数据字段中其余位将填充0。

PAUSE帧各个字段的定义如下：

**目的地址**：协议规定PAUSE的目的地址为保留的组播地址0x01-80-C2-00-00-01。

**源地址**：发送PAUSE帧端口的48位MAC地址。

**类型**：MAC控制帧（PAUSE帧）是符合IEEE802.3协议的以太网帧，可以通过其唯一的类型域标识符(0x8808)识别。

**操作码**：恒为0x0001。其实，PAUSE帧是MAC控制帧的一种，其他类型的MAC控制帧使用不同的opcode值，此处不做详细说明。后面会谈到**和PAUSE类似的PFC帧**，PFC帧中该域的取值是0x0101。

**操作参数**：2字节的暂停时间参数。它是PAUSE发送方请求对方停止发送数据帧的时间长度，通常为0xFFFF，**时间度量单位是以当前传输速率传输512位数据所用的时间**，接收方实际暂停的时间为操作参数字段内容与以当前传输速率传输512位数据所用时间的乘积。

**校验序列（FCS：Frame Check Sequence，帧校验序列，俗称帧尾，即计算机网络数据链路层的协议数据单元（帧）的尾部字段）**：4个字节的循环冗余校验（CRC）字段。CRC 是一种数学算法，MAC发送侧（MAC_TX）创建每个帧时都将运行它。MAC接收侧（MAC_RX）接收到帧时以preamble, SFD, DA, SA, Length/Type, DATA and Pading,作为输入数据进行 CRC 计算，计算结果与接收帧的FCS字段比较，结果相同表示帧有效，结果不同接收方认为发生了错误，进而将帧丢弃。

## PAUSE帧处理流程
如图所示，左侧为本端芯片，右侧为对端芯片。
![](attachments/Pasted%20image%2020230802150758.png)

MAC0和MAC1都包含发送侧tx和接收侧rx。左侧芯片内部mac上游模块A与mac0发送侧有流控信号fc_rdy。信号高表示模块A无法及时处理输入数据，需要进行流控。为了方便突出重点，图中省略了PCS以及serdes等模块。


***具体流程处理如下：***
1~2步：对端mac1发送数据给mac0接收侧，进行发送到模块A
3步：模块A无法即使处理输入的数据，需要减少数据输入，从而将fc_rdy拉高。
4步：mac0发送侧tx发现流控信号fc_rdy为高，产生pause帧，发送给mac1接收侧。只要fc_rdy为高，mac0发送侧tx**每隔一段时间**发送一个pause帧，间隔时间由配置寄存器控制。间隔时长计算由计数器counting计算。Pause帧内停止发送数据的时间由另外一个配置寄存器控制。只要fc_rdy为高期间，mac0发送侧不发送正常数据。
5步：mac1接收侧rx接受到pause报文后，提取pause帧内包含的暂停时间，控制发送侧tx停止发送数据.
6~7~8：mac1停止发送数据后，模块A处理完之前的数据后将fc_rdy拉低，表示mac1可以继续发送数据了。
9:步：第9步分2种情况。
- 情况1：fc_rdy拉低，并且counting在计数没有到一个间隔周期，此时发送pause帧，但是帧内暂停时间为0. Mac1接受到pause帧后，控制tx控制立即开始发送数据。
- 情况2：fc_rdy拉低的同时，counting正好计数到一个间隔周期，此时不发送pause帧。等到上一个pause帧的暂停时间到达后，mac1发送侧tx继续发送数据。



***pause帧处理协议强制要求：***

- **pause的产生发送过程不能中断一个完整的数据报文。**即在第4步中，fc_rdy拉高后，首先mac0 tx侧需要判断当前是否正常数据报文在传输。如果有，则需要在当前数据报文传输完成后才能发送pause帧。也就是说**在发送过程中，只能在完整数据报文的间隙插入pause帧。**

- **新的pause报文暂停时间会覆盖上一个暂停时间。**对mac1来说，当mac1接收到新的pause帧后，暂停时间以最新时间为准。

# 网卡流控的开启、关闭
## 网卡开启关闭流控
```c
# 设置是否产生 pause frame
ethtool -A eth04 rx off tx off
ethtool -A eth03 rx off tx off

注：接受发送pause需要是开启的。如果将rx/tx pause关闭。则ethtool -S 的统计中不会有pause的增长。

# 查看网卡配置
查看：
ethtool --show-pause  eth03
or
ethtool -a eth03

比如：
# ethtool -a eth03
Pause parameters for eth03:
Autonegotiate:	off
RX:		on
TX:		on
```

## ethtool 查看Pause的统计

```c
1> Mellanox 网卡
# ethtool -S eth03 | grep -i -e pause -e flow_
     rx_pause_ctrl_phy: 0
     tx_pause_ctrl_phy: 0
     rx_global_pause: 0
     rx_global_pause_duration: 0
     tx_global_pause: 0
     tx_global_pause_duration: 0
     rx_global_pause_transition: 0
     tx_pause_storm_warning_events: 0
     tx_pause_storm_error_events: 0

2> intel ixgbe 10G网卡
# ethtool -S eth03 | grep -i -e pause -e control -e priori
     tx_flow_control_xon: 0
     rx_flow_control_xon: 0
     tx_flow_control_xoff: 0
     rx_flow_control_xoff: 0

3> intel ice E810系列网卡：
# ethtool -S eth03 | grep -i -e pause -e control -e priori
     tx_priority_0_xon.nic: 0
     tx_priority_0_xoff.nic: 0
     tx_priority_1_xon.nic: 0
     tx_priority_1_xoff.nic: 0
     tx_priority_2_xon.nic: 0
     tx_priority_2_xoff.nic: 0
     tx_priority_3_xon.nic: 0
     tx_priority_3_xoff.nic: 0
     tx_priority_4_xon.nic: 0
     tx_priority_4_xoff.nic: 0
     tx_priority_5_xon.nic: 0
     tx_priority_5_xoff.nic: 0
     tx_priority_6_xon.nic: 0
     tx_priority_6_xoff.nic: 0
     tx_priority_7_xon.nic: 0
     tx_priority_7_xoff.nic: 0
     rx_priority_0_xon.nic: 0
     rx_priority_0_xoff.nic: 0
     rx_priority_1_xon.nic: 0
     rx_priority_1_xoff.nic: 0
     rx_priority_2_xon.nic: 0
     rx_priority_2_xoff.nic: 0
     rx_priority_3_xon.nic: 0
     rx_priority_3_xoff.nic: 0
     rx_priority_4_xon.nic: 0
     rx_priority_4_xoff.nic: 0
     rx_priority_5_xon.nic: 0
     rx_priority_5_xoff.nic: 0
     rx_priority_6_xon.nic: 0
     rx_priority_6_xoff.nic: 0
     rx_priority_7_xon.nic: 0
     rx_priority_7_xoff.nic: 0

```

# Pause Frame的应用

# 参考
