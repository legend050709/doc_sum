```table-of-contents
```
# 介绍
一些网络设备会带有设置中断亲和性的脚本，
```bash
# find / -type f -name  set_irq_affinity.sh
/usr/src/ofa_kernel/default/ofed_scripts/set_irq_affinity.sh
/usr/src/ofa_kernel-5.1/ofed_scripts/set_irq_affinity.sh
/usr/sbin/set_irq_affinity.sh
```


# 使用
## 查看
### 查看网卡的中断号
```bash
set_irq_affinity.sh  eth02
```
![](attachments/Pasted%20image%2020240724164102.png)

### 查看中断的亲和性
```bash
for a in `seq 1227 1234`; do cat /proc/irq/${a}/smp_affinity_list; done

for a in `seq 1227 1234`; do cat /proc/irq/${a}/smp_affinity; done
```

![](attachments/Pasted%20image%2020240724164243.png)

## 设置

```bash
ethtool -L eth0 combined 2
./set_irq_affinity 0-1 eth0
```

# 参考
```bash
(https://www.kernel.org/doc/html/latest/core-api/irq/irq-affinity.html)
```