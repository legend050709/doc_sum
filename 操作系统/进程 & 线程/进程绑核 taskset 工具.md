```table-of-contents
```
# 操作
## 查看
查看CPU的亲和性
```bash
# cat /proc/569/status | grep Cpu
Cpus_allowed:   01 
Cpus_allowed_list:    0
```
## 设置

# 范例
```bash
taskset -pc 0 578 //将 pid 为 578 的进程绑定到 0 号 CPU
```
# 参考
```c

```