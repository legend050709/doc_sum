```table-of-contents
```
# 绑核方法
通过 `taskset` 命令、`numactl` 命令、修改已经启动进程的 `smp_affinity_list`文件 等方式，对要执行的程序，或者已经启动的进程，进行绑定CPU。

一般而言，`numactl` 命令 只能设置要启动程序要绑定的NUMA，CPU以及内存，无法设置已经启动的程序的绑定。

# taskset 工具
## 查看
查看CPU的亲和性
```bash
# cat /proc/569/status | grep Cpu
Cpus_allowed:   01 
Cpus_allowed_list:    0
```
## 设置

## 范例
```bash
taskset -pc 0 578 //将 pid 为 578 的进程绑定到 0 号 CPU
```

**将进程PID绑定在CPU编号为0和1的核心上运行：**
```bash
taskset -cp 0,1 PID
-c参数指定要绑定的CPU核心编号，可以使用逗号分隔多个核心编号；

-p参数可以用来确认绑定的结果。
```

# numactl命令

# smp_affinity_list文件设置
```bash
echo $cpuNumber > /proc/irq/$irq/smp_affinity_list 

例子：echo 0-4 > /proc/irq/78/smp_affinity_list 

```
# 其他
## numastat命令

# 参考
```c

```