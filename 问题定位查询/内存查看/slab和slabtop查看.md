```table-of-contents
```
# slab的背景
# slab的简介
# slab占用的内存查看
## meminfo查看整体
## slabinfo查看具体
## slabtop查看具体
# 清理slab内存
# 问题
## slab_unreclaimable内存占用高的问题
`slab_unreclaimable`是指在`Linux`内存管理中由`slab`分配器分配的且被标记为不可回收（unreclaimable）的内存。当不可回收内存占用总内存的比例过高时，将会影响可用内存与系统性能。
### 现象
在Linux实例内运行`cat /proc/meminfo | grep "SUnreclaim"`命令查看`SUnreclaim(S:代表slab)`参数指标时，发现内存较大（例如`SUnreclaim: 6069340 kB`），当该内存超过系统总内存大小的10%时，表示`slab_unreclaimable`内存占用过高，系统可能会存在`slab`内存泄露。
### 可能原因
在Linux内存管理中，`slab`内存是**内核**用于高效分配小块内存的一种缓存机制。内核组件或驱动程序通过调用内存分配接口（**kmalloc**等）向`slab`分配器申请内存，但是内核组件或驱动程序又没有正确释放内存，这将导致不可用内存越来越多，可用内存越来越少。
### 排查步骤
排查使用`objects`或内存较多，且内存不可回收的`slab`内存对应的名称。

(1)查看使用`objects`或内存最多的slab内存信息。
```shell
slabtop -s -a
```
![](attachments/Pasted%20image%2020240314164007.png)

命令行返回结果中，您可以查看并记录`OBJ/SLAB`列数值较高的slab内存对应的名称（`NAME`列）。

(2) 确认slab内存是否为不可回收
```shell
cat /sys/kernel/slab/<slab NAME>/reclaim_account

命令中的`<slab NAME>`变量需要手动修改为上一步中获取到的`OBJ/SLAB`列数值较高的slab内存对应的名称。
```

例如，查看名称为`kmalloc-192`的slab内存是否为不可回收。
```shell
cat /sys/kernel/slab/kmalloc-192/reclaim_account
```
查询结果为0时，表示slab内存不可回收；查询结果为1时，表示slab内存可回收。

(3) 排查具体原因
您

### 工具排查具体原因
可以使用**crash工具进行静态分析**，也可以使用**perf工具进行动态分析**，排查造成slab内存泄露的原因。
本文提供的示例场景中，存在slab泄露的内存名称为`kmalloc-192`。

#### crash工具静态分析

（1）安装crash工具
```shell
sudo yum install crash -y
```

（2）安装内核调试工具kernel-debuginfo
```shell
sudo yum install -y kernel-debuginfo-<内核版本> 

or

sudo yum install kernel-debuginfo -y
```

（3）启动`crash`工具
```shell
sudo crash
```

(4) 在 crash工具内，查看`kmalloc-192`内存统计信息。
```shell
kmem -S kmalloc-192

内存统计信息较多时，您可以设置只显示最后几行（例如10行）信息
kmem -S kmalloc-192 | tail -n 10

命令行的返回结果示例如下：

	SLAB              MEMORY            NODE  TOTAL  ALLOCATED  FREE
  ffffea004c94e780  ffff88132539e000     0     42         29    13
  ffffea004cbef900  ffff88132fbe4000     0     42         40     2
  ffffea000a0e6280  ffff88028398a000     0     42         40     2
  ffffea004bfa8000  ffff8812fea00000     0     42         41     1
  ffffea006842b380  ffff881a10ace000     0     42         41     1
  ffffea0009e7dc80  ffff880279f72000     0     42         34     8
  ffffea004e67ae80  ffff881399eba000     0     42         40     2
  ffffea00b18d6f80  ffff882c635be000     0     42         42     0

以`ffff88028398a000`统计信息为例，空闲内存较少（`FREE`列），已分配的内存较多（`ALLOCATED`列）。
```

(5) 在crash工具内，查看`ffff88028398a000`内存信息。
```shell
rd ffff88028398a000 512 -S
```
命令行的返回信息较长，您可以根据命令行提示打印多个页面数据进行分析。例如：
返回信息中多次出现`put_cred_rcu`函数时，您可以自行查看Linux内核源码，搜索`put_cred_rcu`函数。
```shell
void __put_cred(struct cred *cred)
{
    call_rcu(&cred->rcu, put_cred_rcu);
}
```
查看到`put_cred_rcu`函数用于异步释放cred，且`put_cred_rcu`出现在cred的末尾，表示是内核中cred结构体出现了slab内存泄露


#### perf工具动态分析
(1) 使用perf工具动态获取`kmalloc-192`中没有释放的内存
```shell
sudo perf record -a -e kmem:kmalloc --filter 'bytes_alloc == 192' -e kmem:kfree --filter ' ptr != 0' sleep 200

动态获取数据的时间间隔为200秒.
```

(2) 将动态获取的数据打印至临时文件中
```shell
sudo perf script > testperf.txt
```

(3) 查看临时文件内容
```shell
cat testperf.txt
```
您需要手动排查没有空闲内存（`free`）的内存信息，然后在Linux内核源代码中手动查询产生`slab`内存泄露的函数。


#### 解决方案
通过crash和perf等工具确定了内存泄露的函数调用路径或者影响的内核数据结构后，建议在内核开发者或专业运维人员指导下确定内存泄露的具体源头，然后解决内存泄露问题。
以下是可能用到的一些解决方案，供您参考：
- 升级内核或补丁
- 调整内核参数
- 重启影响的服务或模块
- 优化应用程序或驱动
- 重启系统

# 参考
```c
https://blog.51cto.com/ghostwritten/5345133
https://www.cnblogs.com/architectforest/p/12575732.html

# 如何排查slab_unreclaimable内存占用高的原因？
https://help.aliyun.com/zh/alinux/support/identify-the-causes-of-high-percentage-of-the-slab-unreclaimable-memory
```