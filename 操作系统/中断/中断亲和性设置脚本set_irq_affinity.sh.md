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
通过下面的脚本设置网卡队列的CPU亲和性。
注意：==设置之前停掉 irqbalance== .
```bash
# 查看当前运行情况
service irqbalance status
 
# 终止服务
service irqbalance stop
 
取消开机启动：
 chkconfig irqbalance off
```

`set_irq_affinity  CPUx-CPUy ethx`
```
说明：
CPUx-CPUy： 表示一个cpu的范围，即网卡队列绑定的CPU。
ethx：要设置的网卡
```

范例：
```bash
ethtool -L eth0 combined 2
./set_irq_affinity 0-1 eth0
```

![](attachments/Pasted%20image%2020241024142250.png)

# 原理
```bash
ll /usr/sbin/common_irq_affinity.sh
ll /usr/sbin/set_irq_affinity_cpulist.sh
ll /usr/sbin/set_irq_affinity.sh
ll /usr/sbin/set_irq_affinity_bynode.sh
```

# 参考
```bash
(https://www.kernel.org/doc/html/latest/core-api/irq/irq-affinity.html)
```