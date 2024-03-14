```table-of-contents
```
# meminfo
# buddyinfo
## 介绍
`/proc/buddyinfo` 文件可以**查看Linux机器上可用的内存块（多个连续的pagesize）的个数**。
可以查看每个节点（node），不同区域（zone）的每个`order`大小的块的可用数量。
如下所示：
![](attachments/Pasted%20image%2020240314145642.png)
## 输出说明
在`/proc/buddyinfo`中每一行都代表了一个内存块的信息，包含空闲内存块的数量、大小。
每行的每一列表示**指定大小的内存块的个数**。分别是2的0次方(即1)个页面的个数，2的1次方(即2)个页面的个数，2的2次方(即4)个页面的个数。

注：一般默认情况下，页面的大小为4K。可以通过 getconf PAGESIZE 来查看。
那么从左到右的指定大小就是4K(`1*4K`)，8K(`2*4K`)，16K(`4*4K`)，32K(`8*4K`)... 的大小。
## 应用
如果业务偶然出现响应时间过长或者系统调用时间过长，系统的`sys`指标也会相应的增高。查看`/proc/buddyinfo`文件内容，确定每种大小的内存块的数量。如果某种大小的内存块非常少或者不可用，说明系统的内存碎片比较严重，需要进行相应的优化。

### 范例
实例内部署的业务偶然出现响应时间过长或者系统调用时间过长，系统的`sys`指标也会相应的增高。伙伴系统会缺少高阶内存（`order`大于3的内存）。例如，运行cat /proc/buddyinfo命令的返回结果如下所示，其中从第4列开始，每一列对应伙伴系统的不同`order`空闲内存。

![](attachments/Pasted%20image%2020240314150505.png)

#### 分析
**Linux系统在长时间运行后，连续的大块物理内存会被分解为小块的物理内存**。
此时系统中部署的业务如果需要连续的大块内存，则系统会先进入**耗时较长**的内存规整流程（compact 压实pages），进而会引起系统性能抖动。

一般情况下，内存碎片化会产生类似于如下所示的内核堆栈信息。
![](attachments/Pasted%20image%2020240314150757.png)

#### 解决
应对Linux内存碎片化，您可以采取如下措施：
**调整`min`水位线**
多数情况下阿里云建议您将min水位线设置为总内存的1%~3%。推荐您设置为总内存的2%，当内存资源紧张时，提前进入异步回收。
调整min水位线的命令如下：
```shell
sysctl -w vm.min_free_kbytes = memtotal_kbytes * 2%

其中，变量 memtotal_kbytes * 2% 表示当前实例内总内存的2%对应的内存大小。
```

**调整`min`水位线和`low`水位线之间的差值**
可以通过内核的`watermark_scale_factor`调整min水位线和low水位线之间的差值，以应对业务突发申请内存的情况。`watermark_scale_factor`的默认值为总内存的0.1%，最小值（即min水位线和low水位线之间的最小差值）为`0.5*min水位线`。
调整`watermark_scale_factor`的命令如下：
```shell
sysctl -w vm.watermark_scale_factor = value

其中，变量value为您手动设置的min水位线和low水位线之间的差值。
```

**定期进行内存规整(compact)**:
可以在业务空闲时段，主动触发异步内存规整。触发命令如下：
```shell
echo 1 > /proc/sys/vm/compact_memory

注：内存规整的开销过大。
```

**定期手动释放缓存**:
以上措施均不能有效应对内存碎片化时，您还可以在业务空闲时段执行释放缓存（`drop cache`）的操作，然后内存会重新分配。释放缓存是避免内存碎片化的有效措施，但**在执行释放缓存时会出现短时间的系统性能抖动**。

手动释放缓存的命令如下：
```shell
echo 3 > /proc/sys/vm/drop_caches
```

## 其他

### 内存规整（compact)
#### 流程
**思路**
从选中的 `zone` 底部扫描已分配的迁移类型为 `MIGRATE_MOVABLE` 的页面，再从此 `zone` 的顶部扫描空闲页面，把底部的可移动页移到顶部的空闲页，在底部形成连续的空闲页。

**图解**
（1）选中的碎片化 zone：
![](attachments/Pasted%20image%2020240314173216.png)

（2）扫描可移动的页面
![](attachments/Pasted%20image%2020240314173241.png)

（3）扫描空闲的页面
![](attachments/Pasted%20image%2020240314173321.png)


（4）规整完成
![](attachments/Pasted%20image%2020240314173404.png)

#### 调试
```bash
cd /sys/kernel/debug/tracing/events/compaction
```
![](attachments/Pasted%20image%2020240314180307.png)

## 使用
### 更友好的输出脚本
```python
#!/usr/bin/env python
# vim: tabstop=4 expandtab shiftwidth=4 softtabstop=4 textwidth=79 autoindent

"""
Python source code
Last modified: 15 Feb 2014 - 13:38
Last author: lmwangi at gmail  com
Displays the available memory fragments
by querying /proc/buddyinfo
Example:
# python buddyinfo.py
"""
import optparse
import os
import re
from collections import defaultdict
import logging


class Logger:
    def __init__(self, log_level):
        self.log_level = log_level

    def get_formatter(self):
        return logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')

    def get_handler(self):
        return logging.StreamHandler()

    def get_logger(self):
        """Returns a Logger instance for the specified module_name"""
        logger = logging.getLogger('main')
        logger.setLevel(self.log_level)
        log_handler = self.get_handler()
        log_handler.setFormatter(self.get_formatter())
        logger.addHandler(log_handler)
        return logger


class BuddyInfo(object):
    """BuddyInfo DAO"""
    def __init__(self, logger):
        super(BuddyInfo, self).__init__()
        self.log = logger
        self.buddyinfo = self.load_buddyinfo()

    def parse_line(self, line):
        line = line.strip()
        self.log.debug("Parsing line: %s" % line)
        parsed_line = re.match("Node\s+(?P<numa_node>\d+).*zone\s+(?P<zone>\w+)\s+(?P<nr_free>.*)", line).groupdict()
        self.log.debug("Parsed line: %s" % parsed_line)
        return parsed_line

    def read_buddyinfo(self):
        buddyhash = defaultdict(list)
        buddyinfo = open("/proc/buddyinfo").readlines()
        for line in map(self.parse_line, buddyinfo):
            numa_node =  int(line["numa_node"])
            zone = line["zone"]
            free_fragments = map(int, line["nr_free"].split())
            max_order = len(free_fragments)
            fragment_sizes = self.get_order_sizes(max_order)
            usage_in_bytes =  [block[0] * block[1] for block in zip(free_fragments, fragment_sizes)]
            buddyhash[numa_node].append({
                "zone": zone,
                "nr_free": free_fragments,
                "sz_fragment": fragment_sizes,
                "usage": usage_in_bytes })
        return buddyhash

    def load_buddyinfo(self):
        buddyhash = self.read_buddyinfo()
        self.log.info(buddyhash)
        return buddyhash

    def page_size(self):
        return os.sysconf("SC_PAGE_SIZE")

    def get_order_sizes(self, max_order):
        return [self.page_size() * 2**order for order in range(0, max_order)]

    def __str__(self):
        ret_string = ""
        width = 20
        for node in self.buddyinfo:
            ret_string += "Node: %s\n" % node
            for zoneinfo in self.buddyinfo.get(node):
                ret_string += " Zone: %s\n" % zoneinfo.get("zone")
                ret_string += " Free KiB in zone: %.2f\n" % (sum(zoneinfo.get("usage")) / (1024.0))
                ret_string += '\t{0:{align}{width}} {1:{align}{width}} {2:{align}{width}}\n'.format(
                        "Fragment size", "Free fragments", "Total available KiB",
                        width=width,
                        align="<")
                for idx in range(len(zoneinfo.get("sz_fragment"))):
                    ret_string += '\t{order:{align}{width}} {nr:{align}{width}} {usage:{align}{width}}\n'.format(
                        width=width,
                        align="<",
                        order = zoneinfo.get("sz_fragment")[idx],
                        nr = zoneinfo.get("nr_free")[idx],
                        usage = zoneinfo.get("usage")[idx] / 1024.0)

        return ret_string

def main():
    """Main function. Called when this file is a shell script"""
    usage = "usage: %prog [options]"
    parser = optparse.OptionParser(usage)
    parser.add_option("-s", "--size", dest="size", choices=["B","K","M"],
                      action="store", type="choice", help="Return results in bytes, kib, mib")

    (options, args) = parser.parse_args()
    logger = Logger(logging.DEBUG).get_logger()
    logger.info("Starting....")
    logger.info("Parsed options: %s" % options)
    print logger
    buddy = BuddyInfo(logger)
    print buddy

if __name__ == '__main__':
    main()
```

执行的结果：
![](attachments/Pasted%20image%2020240314145158.png)
# slabinfo
# zoneinfo

# 其他
## pagesize查看
```bash
getconf PAGESIZE
```

## `collectl` 工具
### 介绍
Linux系统管理员经常需要监测cpu,内存,磁盘,网络等系统信息。Linux上已有`iotop`,`top`,`free`,`htop`,`sar`等丰富的常规工具来实现监测功能。
`Collectl` **是一个轻量级的性能监控工具，可监控包括`CPU`、磁盘、带宽、内存、网络、`NFS`、进程等等信息**。与大多数其他监控工具不同，**`collectl`**不关注有限数量的系统指标，相反，它可以收集许多不同类型的系统资源的信息，例如 cpu、磁盘、内存、网络、套接字、`tcp`、`inode`、`infiniband` 、集群、内存、`nfs`、进程、二次曲线、平板和 `buddyinfo`。
![](attachments/Pasted%20image%2020240314143644.png)

注：`collectl` 有些类似于 `busybox`。都是将多个工具集成到一个可执行文件中，`collectl` 添加各个参数，就类似于之前的 某个工具的作用。
如下所示：
![](attachments/Pasted%20image%2020240314144452.png)

### 收集功能

- 它可以交互运行，作为守护进程运行，或两者兼而有之。
- 它可以以多种格式显示输出。
- 它能够监控几乎任何子系统。
- 它可以扮演许多其他实用程序的角色，例如 ps、top、iotop 和 vmstat。
- 它具有记录和回放捕获数据的能力。
- 它可以以各种文件格式导出数据。（当您想使用外部工具分析数据时，这非常有用）。
- 它可以作为服务运行以监控远程机器或整个服务器集群。
- 它可以在终端显示数据，并写入文件或套接字。

子系统是可检测到的不同系统资源类型。像CPU,内存,带宽等等都可构成一个子系统。
只运行collectl命令将以批处理模式输出CPU,磁盘和网络子系统信息;
![](attachments/Pasted%20image%2020240314143225.png)

### 使用
安装：
```bash
 yum install -y collectl
```

### 范例
#### 查看buddyinfo
`collectl` 工具可以实时监控相关剩余内存(buddyinfo)的变化

使用：
```Bash
#collectl -sB -oT  或者  collectl -s B -o T
waiting for 1 second sample...

# MEMORY FRAGMENTATION (4K pages)
#Time   Node   Zone   1Pg  2Pgs  4Pgs  8Pgs 16Pgs 32Pgs   64Pgs  128Pgs  256Pgs  512Pgs 1024Pgs
12:50:41   0    DMA   126    93    71    64    38    13       4       2       0       0       0
12:50:41   0  DMA32   441   349    94    26    14     0       0       0       0       0       0
12:50:42   0    DMA   126    93    71    64    38    13       4       2       0       0       0
12:50:42   0  DMA32   441   349    94    26    14     0       0       0       0       0       0
12:50:43   0    DMA   126    93    71    64    38    13       4       2       0       0       0
12:50:43   0  DMA32   441   349    94    26    14     0       0       0       0       0       0
12:50:44   0    DMA   126    93    71    64    38    13       4       2       0       0       0
12:50:44   0  DMA32   441   349    94    26    14     0       0       0       0       0       0
12:50:45   0    DMA   126    93    71    64    38    13       4       2       0       0       0
12:50:45   0  DMA32   441   349    94    26    14     0       0       0       0       0       0
12:50:46   0    DMA   126    93    71    64    38    13       4       2       0       0       0
12:50:46   0  DMA32   441   349    94    26    14     0       0       0       0       0       0
12:50:47   0    DMA   126    94    71    66    40    13       4       2       0       0       0
12:50:47   0  DMA32   452   354    97    40    19     0       0       0       0       0       0
```

#### 查看内存
![](attachments/Pasted%20image%2020240314143922.png)

#### 查看CPU
![](attachments/Pasted%20image%2020240314144006.png)’


**将collectl**作为**top**实用程序非常容易，只需在终端中运行以下命令，您将在**top**工具中看到类似的输出：
```text
collectl --top：
```
![](attachments/Pasted%20image%2020240314144634.png)

要将**collectl**实用程序用作**ps**工具，请在终端中运行以下命令。
```text
collectl -c1 -sZ -i:1
```
![](attachments/Pasted%20image%2020240314144901.png)
# 参考
```bash
# Linux内存碎片化的应对措施
https://help.aliyun.com/zh/alinux/support/solutions-to-memory-fragmentation-in-linux-operating-systems

# /proc/buddyinfo文件
https://garlicspace.com/2020/06/06/proc-buddyinfo%E6%96%87%E4%BB%B6/
```