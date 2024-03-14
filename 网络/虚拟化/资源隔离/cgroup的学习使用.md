```table-of-contents
```
# 问题
## cgroup引起的应用性能抖动
系统通过`cgroup`(控制群组：`control group`)可以对系统内的资源进行分配、管理、监控等操作。不合理的`cgroup`层级或数量可能引起系统中应用性能的不稳定。
### 问题现象
在容器相关的业务场景下，系统中的应用偶然会出现请求延时增大，并且容器所属宿主机的CPU使用率中，sys指标（内核空间占用CPU的百分比）达到`30%`及以上。例如，通过`top`命令查看Linux系统性能数据时，CPU的`sy`指标达到`30%`.
![](attachments/Pasted%20image%2020240314174148.png)

### 可能原因
例如，运行`cat /proc/cgroups` 查看当前所有控制群组的状态，发现`memory`对应的`cgroup`数目高达`2040`。
![](attachments/Pasted%20image%2020240314174300.png)

### 详细分析
可以通过perf工具动态分析，定位出现问题的原因。
（1）对系统的进程进行采样、分析
```shell
perf record -a -g sleep 10
```

（2）查看分析结果
```shell
perf report
```
![](attachments/Pasted%20image%2020240314174404.png)

从分析结果中，可得出Linux内核运行时间大多集中在`memcg_stat_show`函数，这是由于memory对应的cgroup数量过多，系统在遍历这些cgroup所导致的内核长时间运行。

除了`memory`之外，`cpuacct`、`cpu`对应的`cgroup`过多还可能使`CFS`、`load-balance`的性能受影响。

### 解决方案
对Linux实例运维时，可参考以下两条建议，避免因cgroup引起的应用性能抖动。
- cgroup的层级建议不超过`10`层。
- cgroup的数量上限建议不超过`1000`，且应当尽可能地减少`cgroup`的数量。

# 参考
```bash

```
