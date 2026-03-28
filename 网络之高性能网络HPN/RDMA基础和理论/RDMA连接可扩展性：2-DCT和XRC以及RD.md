```table-of-contents
```
# RD
## 为什么没有实现RD服务类型

# DCT



## 优缺点
### 优点

### 缺点
DCT 是一个 server scalability 技术，而不是 client scalability 技术。其 可以减少 server端的QP的个数，但是不可以减少Client的QP的个数。

## perftest使用DCT
```bash
# 使用 DC（Dynamic Connection）模式  
ib_write_bw -d mlx5_0 -c DC -q 8 192.168.1.1
```

![](attachments/Pasted%20image%2020260327203038.png)

## 其他
### perftest使用大页
```bash
# 配置大页  
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages  
  
# 在测试时使用大页  
ib_write_bw -d mlx5_0 -x 0 --use_huge
```

# XRC

# 软件层自实现的连接可扩展性
## per-thread的VRC
## per-procoss的VQP

# 参考
```bash

```