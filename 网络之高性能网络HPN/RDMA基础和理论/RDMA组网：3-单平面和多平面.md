```table-of-contents
```

# 单平面和多平面

一般是TCP平面和RDMA平面，分别跑TCP流量和RDMA流量。

## 双平面
![](attachments/image%20(25).png)

（1）参数平面
8个GPU（8轨）+ `8*400G（Mellanox Cx7）`

（2）数据平面
2张DPU（每张DPU是：`2*200G`的 NV BF3 supernic bond。 ）

总带宽是：`8*400G + 2*2*200G`

# 参考
```bash


```