
# traceroute的难点
## 流量经过LB的转换

### 问题描述
一条流量经过LB进行了源目IP,Port的转换后，报文转换后在设备上生成ICMP差错报文，LB如何将ICMP差错报文透传给TraceRoute发起者？
如下所示：

### 问题解决


## Overlay封装
### 问题描述
TraceRoute发起者发起的TraceRoute是对于某条underlay流的trace，经过中间设备之后，underlay流量被封装为Overlay（虽然外层Overlay也可以）。那么TraceRoute发起者发出trace后，经过overlay封装，此时是无法知道overlay数据包所经过的路径，因为overlay封装之后，即使产生了icmp差错报文也是给了其他的设备，而不是TraceRoute发起者。
另外一个问题是，vxlan封装之后，外层IP头的TTL和内层不一样的问题。

### 问题解决
思路：
可以基于外层Sport的计算方法，基于内层的五元组信息，