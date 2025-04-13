```table-of-contents
```
# 问题介绍
# imiss的统计
# 问题分析
# 小结
## Imiss可能原因
### 原因分类
**（1）程序的性能问题**
比如DPDK程序的设计、程序的性能存在优化的空间。

**（2）偶发因素**
	- 异常包的处理时间长
	- 微突发的流量
	- 其他的一次性的进程(执行完就退出)抢占了CPU

**（3）CPU降频**
如果存在CPU降频，则也是可能存在 Imiss 问题的。
另外不同型号的设备，有的产生了 Imiss，有的不存在 Imiss。考虑是否是不同型号的设备的CPU的实际频率不一致导致？

### 查询方法和应对手段
**针对微突发**：
- 查询：统计固定的短时间内的 ipackets 和 imiss 的 统计。查看短时间内的 imiss 是否和 ipackets 的增长有关系。
- 应对：调大网卡RXQ buffer提升扛突发能力.


**CPU频率**：
查看是否存在CPU的降频。

**CPU抢占**：
- 查询：通过查看指定的线程是否存在 `sched_switch`（通过内核的`debug/tracing`进行打印 ）。 
- 应对：

**异常包处理时间长**：
DPDK 应用程序中增加包处理的 trace 统计，并且可以设置阈值，将超过阈值的异常包给拷贝出来。查看 出现 imiss的时候，是否存在 处理时延的增加。


**其他方法**：
增大网卡的接收队列的个数、调大 ring_buffer的大小。

## 优化经验
### 字节的优化经验
![](attachments/Pasted%20image%2020240131181018.png)
![](attachments/Pasted%20image%2020240131180746.png)

总结即：
- 提升程序的性能，主要是快慢路径的性能。
- 网卡的队列的优化
- 网卡的队列个数的优化
- 网卡的队列的权重的动态调节
	- 基于这个队列的 ring buffer 是否发生了拥塞(是否出现了imiss)，决定后续这个队列的能力，达到动态调节。比如：某个队列发生了拥塞，则动态的将 权重调低，后续送给这个网卡的包减少。

# 参考
```c
# DPDK丢包那些事（+++++）
https://www.cnblogs.com/t-bar/p/17630652.html

# DPDK疑难杂症之网卡Imiss/ierror/i-nombuf问题
https://blog.csdn.net/legend050709/article/details/123655712


(https://zhuanlan.zhihu.com/p/73393629) (图文很形象；++++++++)

(https://blog.csdn.net/u013431916/article/details/81912266) (查看 pcie 的速率;)

(https://stackoverflow.com/questions/49044325/difference-between-rx-missed-rx-errors-and-rx-nombuf) (rx-errors, rx-nombuf, rx-missed 区别)

 # 突破性能瓶颈，火山引擎自研vSwitch技术实践揭秘
https://developer.volcengine.com/articles/7133167631380512782
```