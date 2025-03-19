```table-of-contents
```
# 使用
## 基于包内容进行过滤

**dns报文过滤**
如果您想要过滤出特定的域名相关的数据包，可以使用` Wireshark` 中的`dns`过滤器来实现。具体操作如下：
首先，启动` Wireshark` 并开始捕获网络数据包。
在 Wireshark 的过滤器栏中，输入`dns`并按下回车键，可以看到所有与 `DNS` 协议相关的数据包都被过滤出来了。
在过滤器栏中输入`dns.qry.name == 域名`，其中`域名`就是你要过滤的特定域名。例如，如果您要过滤出所有访问 `www.baidu.com` 的数据包，可以在过滤器栏中输入`dns.qry.name == www.baidu.com`。
按下回车键，`Wireshark` 就会过滤出所有与该特定域名相关的数据包了。


# 分析

## wireshark 查看TCP的时序图

Wireshark 可以用时序图的方式显示数据包交互的过程，从菜单栏中，点击 ：

**统计 (Statistics) -> 流量图 (Flow Graph)，然后，在弹出的界面中的「流量类型」选择 「TCP Flows」**；

你可以更清晰的看到，整个过程中 TCP 流的执行过程：

![](../../网络/四层协议/TCP/attachments/Pasted%20image%2020240513195500.png)


**时序图展示TCP重传**：

![](../../网络/四层协议/TCP/attachments/Pasted%20image%2020240513195523.png)

## I/O Graph 
### 介绍
在 Statistics 下拉菜单中：
![](attachments/Pasted%20image%2020240514171209.png)

点击 I/O Graph 后，Wireshark 会弹出一个趋势图，其 X 轴是时间，Y 轴是性能指标：

![](attachments/Pasted%20image%2020240514171221.png)

注意，这时的图不是我们需要的，需要做一些调整。

### 查看 BPS

既然我们关注传输速度，所以就选择 All Bytes 这个指标项（也是默认选中的那项）作为 Y 轴，然后修改它的计量单位。很可能你的 Wireshark 默认选中的是 AVG(Y Field)，而这并不是我们要关注的字节数。我们可以双击 AVG(Y Field) 进入编辑模式，把它改为 Bytes：

![](attachments/Pasted%20image%2020240514171307.png)

这时我们就能清晰地看到，Wireshark 帮助我们计算出来的分时的速度趋势柱状图了，差不多速度在 480KB/s 上下：

![](attachments/Pasted%20image%2020240514171317.png)

你可能也注意到了，在 X 时间轴上看，一开始几秒速度比较低，第 7 秒才达到 400KB/s 以上。为什么一开始速度这么低呢？其实，这正是 TCP 慢启动的一个缩影：初始阶段，速度是特别低的，但是会很快爬高。



## TCP Stream Graph 
### 介绍
如果说 I/O Graph 展示的速度值很容易理解，那么 TCP Stream Graphs 展示的信息就需要一点 TCP 的知识来辅助理解了。

还是到 Statics 下拉菜单，选择 TCP Stream Graphs，在子菜单中选择 Time sequence (Stevens)。

> 补充：Stevens 这个名字你应该是熟悉的，他就是鼎鼎大名的 TCP/IP 三卷和 Unix 网络编程两卷等名著的作者，Richard Stevens。他把网络协议的圣火带到人间，却又早早离开了人间。这个工具以他的名字命名，也代表了大家对他的深深的怀念。

![](attachments/Pasted%20image%2020240514171448.png)

时间为 X 轴、TCP 序列号为 Y 轴。

### 计算 BPS

你应该知道，序列号其实就等于字节数，那么显然，这里**这条线的斜率也就是传输速率了**。

![](attachments/Pasted%20image%2020240514171554.png)

我们可以自己计算这个斜率（速率） 是多少。比如可以计算 10 秒和 40 秒两处的序列号的差距，再除以 (40-10) 秒，就是速率了。10 秒处的序列号是 2800092，40 秒处是 16480292，那么速率就是 (16480292-2800092)/(40-10)=456KB/s。

![](attachments/Pasted%20image%2020240514171655.png)

![](attachments/Pasted%20image%2020240514171658.png)

#### 和 I/O Graph  的 bps的差异

你可能发现了，两个 Graph 算出来的速度怎么有点差异？
一个是 480KB/s 左右，一个是 456KB/s，相差有 5% 左右。

其实，这是正常的。因为 I/O Graph 统计的 Bytes 是二层帧的大小，而 TCP Stream Graphs 关注的是四层 TCP 段的大小。后者比前者少了二层到四层的头部。严格来说，TCP Stream Graphs 的斜率，只是 TCP payload 的速率；而 I/O Graph 展示的，才是我们一般谈论的传输速度。当然，在定性的讨论中，这点差异是可以忽略的。



# 场景
# 问题
## Wireshark分析包中TCP包的 Len 远远大于 MSS
**问题**
正常每个TCP数据包的Len 不应该大于 MSS，但我们通过 tcpdump 抓包出来后，通过 wireshark 查看的话，有些 Len 远远大于了 MSS值。如下所示：
![](attachments/Pasted%20image%2020231130155051.png)

**分析**
这是由于服务器网卡开启了 TCP Segment Offload, TSO 选项导致，在支持 TSO 的网卡上，为了降低 CPU 的负载，提高网络的出口带宽，TSO 提供一些较大的缓冲区来缓存 TCP 发送的包，然后由网卡负责TCP的分段。而我们通过 tcpdump 抓包工具抓取的是从内核到网卡路径上的数据包，此时还没有到网卡，还没有进行TCP的分段。
![](attachments/Pasted%20image%2020231130155234.png)

**解决**
关闭网卡的TSO：
```c
ethtool -K eth04 tso off
```
上面设置后，我们再进行实验，抓包分析如下：
![](attachments/Pasted%20image%2020231130155343.png)

再次抓包后，发现Len没有大于MSS的了，这里为1488，之所以是1488bytes，是因为 Len = 1500(MTU) - 20(IP Header)- 20(TCP Header) -12(TCP 选项： timestamp 选项 + padding ) = 1448 bytes。
# 参考
```c
# Wireshark的抓包和分析，看这篇就够了！
https://www.wangan.com/p/7fy7fg901377ae58

# Wireshark抓包分析TCP重传
https://chegva.com/3572.html


```