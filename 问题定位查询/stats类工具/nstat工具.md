```table-of-contents
```

# nstat
## 介绍
nstat 是一个简单的监视内核的 SNMP 计数器和网络接口状态的实用工具 。

nstat 可以使用通配符指定一个或多个要过滤的内核的 SNMP（Simple Network Management Protocol） 计数器名称。
**其实查看的是 `/proc/net/netstat、/proc/net/snmp6、/proc/net/snmp` 文件中的内容**。主要是这几个文件，读取不方便。因此有了nstat 命令。

```bash
nstat and rtacct are simple tools to monitor kernel snmp counters and network interface statistics.
```

**差值的原理**
其实是在/tmp下创建了一个临时文件(比如：/tmp/.nstat.u0)，记录上一次的输出，再次执行该命令从 `/proc/net/netstat、/proc/net/snmp6、/proc/net/snmp`中读取结果后，和上一次结果进行diff，得到差值。

## 使用

### 显示计数器的绝对值
```
nstat -az
```

### json格式打印结果
```
nstat -j -az
nstat -j

如果不支持json格式，可能是 iproute包的版本过低，
yum -y update iproute
```

### 指定计数器名称查询
```bash
nstat IpInReceives
nstat IpInReceives -a
nstat IpInReceives -a -j
```
## 使用场景

# rtacct

# 参考
```c
# Nstat Linux 命令
https://0xzx.com/2023020800033150675.html

## Linux 命令（219）—— nstat 命令
https://cloud.tencent.com/developer/article/2195776
```