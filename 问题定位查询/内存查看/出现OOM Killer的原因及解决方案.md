```table-of-contents
```
# 概述
Linux操作系统内存不足时，会先触发内存回收机制释放内存，并将这部分被释放的内存分配给其他进程。如果内存回收机制不能处理系统内存不足的情况，则系统会触发`OOM Killer（Out of Memory Killer）`强制释放进程占用的内存，达到给系统解压的目的。

# 问题现象
`Alibaba Cloud Linux`操作系统出现`OOM Killer`的部分日志信息示例如下，表示`test`进程引发了`OOM Killer`。
```shell
565 [六 9月 11 12:24:42 2021] test invoked oom-killer: gfp_mask=0x62****(GFP_HIGHUSER_MOVABLE|__GFP_ZERO), nodemask=(null), order=0, oom_score_adj=0
566 [六 9月 11 12:24:42 2021] test cpuset=/ mems_allowed=0
567 [六 9月 11 12:24:42 2021] CPU: 1 PID: 29748 Comm: test Kdump: loaded Not tainted 4.19.91-24.1.al7.x86_64 #1
568 [六 9月 11 12:24:42 2021] Hardware name: Alibaba Cloud Alibaba Cloud ECS, BIOS e62**** 04/01/2014
```

# 可能原因
系统出现`OOM Killer`表示内存不足，内存不足可以分为实例全局内存不足和实例内`cgroup`的内存不足。目前常见的出现`OOM Killer`的场景及原因说明如下：

## 实例内cgroup的内存
在日志中搜索，`OOM`以及`Kill`, 以及`limit`。
![](attachments/Pasted%20image%2020240314170641.png)
## 全局内存不足
![](attachments/Pasted%20image%2020240314171519.png)

注：Node x Normal free 的内存大小 = Node x 下buddyInfo 中 Normal  的 内存大小之和。
如下所示：`Node 1 Normal free:43636kB` 等于 下一行的 `= 47560kB`。
```shell
[Sat Sep 11 09:46:24 2021] main invoked oom-killer: gfp_mask=0x62****(GFP_HIGHUSER_MOVABLE|__GFP_ZERO), nodemask=(null), order=0, oom_score_adj=0
[Sat Sep 11 09:46:24 2021] main cpuset=mm_cpuset mems_allowed=1
[Sat Sep 11 09:46:24 2021] Task in / killed as a result of limit of host
[Sat Sep 11 09:46:24 2021] Mem-Info:
[Sat Sep 11 09:46:24 2021] active_anon:172 inactive_anon:4518735 isolated_anon:
    free:4111496 free_pcp:1 free_cma:0
[Sat Sep 11 09:46:24 2021] Node 1 Normal free:43636kB min:45148kB low:441424kB high:837700kB
[Sat Sep 11 09:46:24 2021] Node 1 Normal: 856*4kB (UME) 375*8kB (UME) 183*16kB (UME) 184*32kB (UME) 87*64kB (ME) 45*128kB (UME) 16*256kB (UME) 5*512kB (UE) 14*1024kB (UME) 0     *2048kB 0*4096kB = 47560kB
```
# 解决方案
## **子cgroup或**父cgroup**内存不足**
根据内存实际的提升情况，手动调整`cgroup`的内存上限。

```shell
sudo bash -c 'echo <value> > /sys/fs/cgroup/memory/<cgroup_name>/memory.limit_in_bytes'

其中，`<value>`为您为cgroup设置的内存上限、`<cgroup_name>`为您实际的cgroup名称，请根据实际情况替换。
```


## **系统全局内存不足**
**查看`slab_unreclaimable`内存使用情况**
```shell
cat /proc/meminfo | grep "SUnreclaim"
```
`slab_unreclaimable`内存为系统不可回收的内存，当占用总内存的`10%`以上时，表示系统可能存在`slab`内存泄露。如果存在内存泄露问题，您可以手动排查并解决


**查看systemd内存使用情况**
```shell
cat /proc/1/status | grep "RssAnon"
```
内核发生`OOM Killer`时，会跳过系统的1号进程。
此时您查看`systemd`内存使用情况时，一般不会超过`200 MB`。如果出现异常，您可以尝试自行更新`systemd`工具的版本。

**查看透明大页THP的性能**
开启THP会出现内存膨胀（`memory bloating`），从而导致`OOM Killer`.

## **内存节点（Node）的内存不足**
内存节点（Node）的内存不足导致的`OOM Killer`，您需要重新配置`cgroup`的`cpuset.mems`接口的值，使`cgroup`能够合理使用内存节点的内存。

**确定系统中内存节点（Node）的数量信息**
```shell
cat /proc/buddyinfo
```

**配置`cgroup`的`cpuset.mems`接口**:
```shell
sudo bash -c 'echo <value> > /sys/fs/cgroup/cpuset/<cgroup_name>/cpuset.mems'

其中，`<value>`为对应的内存节点号、`<cgroup_name>`为您实际的cgroup名称，请根据实际情况替换。
例如，系统中存在三个Node，分别为Node 0、Node 1、Node 2。您需要让cgroup使用Node 0和Node 2两个节点的内存。则`<value>`取值为`0,2`。
```
## **内存碎片化时伙伴系统内存不足**
内存碎片化时导致的OOM Killer，建议您**定期在业务空闲时间段，进行内存整理**。
开启内存整理功能的命令为：
```shell
sudo bash -c 'echo 1 > /proc/sys/vm/compact_memory'
```
# 参考
```bash
# 出现OOM Killer的原因及解决方案
https://help.aliyun.com/zh/alinux/support/causes-of-and-solutions-to-the-issue-of-oom-killer-being-triggered
```