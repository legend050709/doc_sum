```table-of-contents
```
# 介绍
RAS指的是Reliability可靠性、Availability可用性、Serviceability可服务性这三个属性，主要用于衡量计算机系统的稳定性和可维护性。
从单机到智算中心，算力每跃升一个量级，硬件故障率就几乎线性翻倍。若缺乏系统化的RAS设计，每一次微小失效都可能触发checkpoint丢失、作业重排、GPU空转，从而浪费算力投资。

RAS目标是使系统尽可能长期可靠的运行而不停机，减少系统downtime；提供硬件检测上报机制，以便在硬件错误引起数据丢失或宕机之前能够通知管理员及时更换硬件；提供硬件错误恢复机制，并尽可能纠正错误，使系统可持续可靠的运行。

RAS涉及的硬件包括且不限于：CPU、Memory、IO、PCIe、硬盘和其他外设

- CPU – detect errors at instruction execution and at L1/L2/L3 caches;
- Memory – add error correction logic ECC to detect and correct errors;
- I/O – add CRC checksums for tranfered data;
- Storage – RAID，journal file systems, checksums, Self-Monitoring, Analysis and Reporting Technology (SMART).

通常来说，硬件错误分为CE、UE、Fatal Error、Non-fatal Error，定义如下

• **Correctable Error (CE)**- the error detection mechanism detected and corrected the error. Such errors are usually not fatal, although some Kernel mechanisms allow the system administrator to consider them as fatal.

• **Uncorrected Error (UE)**- the amount of errors happened above the error correction threshold, and the system was unable to auto-correct.

• **Fatal Error**- when an UE error happens on a critical component of the system (for example, a piece of the Kernel got corrupted by an UE), the only reliable way to avoid data corruption is to hang or reboot the machine.

• **Non-fatal Error**- when an UE error happens on an unused component, like a CPU in power down state or an unused memory bank, the system may still run, eventually replacing the affected hardware by a hot spare, if available.

. **Defferred Error（DE）** - The error was detected, was not corrected, and was deferred. The error has not been silently propagated. The error might be latent in the system. It is IMPLEMENTATION DEFINED whether the error continues to infect the state of the node or whether it has been deferred to the consumer. The node continues to operate. If the error might have been silently propagated, it must be reported as an Uncorrected error.

# 框架

![](attachments/Pasted%20image%2020260702112937.png)

RAS基本流程框图如上，硬件发生故障后，通过硬件RAS能力触发中断或异常，通知到Firmware/OS，软件收到通知后采取相应的策略，比如`Panic`、执行`Recover actions`或者通知到用户。随着RAS功能不断更新迭代以及架构不同，RAS体系开始呈现多样性，因不同使用场景所有不同，体现在：

**1.通知方式多样**
通知方式细分下来包括`IRQ、Exception、Poll、SEA、SDEI、GPIO`等方式。

**2.Mode多样**
硬件故障先通知到Firmware，然后Firmware带外处理或再通知到OS的方式，称为**Firmware First Mode**；
硬件故障通知到OS，OS处理硬件故障的方式，称为**Kernel First Mode**；
这两种方式还可以支持混合使用，各有优劣，要学会因地制宜。比如对于CE来说，服务器经常发生大量CE事件，就会产生`CE Irq`风暴，CPU长时间在处理这些Irq，就会导致其他任务得不到调度，影响整体性能。

**3.芯片架构、硬件多样性**
随着近些年芯片行业发展，芯片架构越来越多样性，包括`Intel、AMD、ARM、RISC`等，不同芯片架构下硬件组成也有些许差异。

**4.软件多样性**
对于Linux驱动来说，包括`mce`驱动、`apei`驱动、`edac`驱动等；
对于用户态RAS服务来说，包括`mcelog、rasdaemon、perf event`通知等；
总体来说，RAS是一个复杂的体系，不同芯片架构、不同硬件RAS功能各不相同，作为RAS开发要根据不同业务场景采取对应的RAS方案。



# RAS故障处理流程

![](attachments/Pasted%20image%2020260702113243.png)

以Intel服务器为例：
（1） Intel服务器内存发生CE故障后，硬件触发CMCI中断，执行OS注册的中断处理函数；
（2）该函数调用EDAC驱动代码，读取MCA状态寄存器来获取硬件故障信息，比如故障级别、故障硬件位置、故障地址等等。EDAC驱动会将信息保存在/dev/mcelog；
（3）Mcelog是一个用户态的服务程序，通过解析/dev/mcelog信息，将其保存在/var/log/mcelog。用户可以通过查看该文件了解此服务器是否发生过硬件故障以及故障发生的时间、硬件信息、是否恢复等关键信息；



# 参考
```bash
# RAS（一）介绍
https://zhuanlan.zhihu.com/p/646145842
```