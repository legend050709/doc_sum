```table-of-contents
```

# lspci查看
## 查看GPU设备
```bash
# lspci | grep -i nvidia
27:00.0 3D controller: NVIDIA Corporation Device 2322 (rev a1)
b8:00.0 3D controller: NVIDIA Corporation Device 2322 (rev a1)

# lspci | grep -i eth
16:00.0 Ethernet controller: Broadcom Inc. and subsidiaries BCM57508 NetXtreme-E 10Gb/25Gb/40Gb/50Gb/100Gb/200Gb Ethernet (rev 12)
16:00.1 Ethernet controller: Broadcom Inc. and subsidiaries BCM57508 NetXtreme-E 10Gb/25Gb/40Gb/50Gb/100Gb/200Gb Ethernet (rev 12)
38:00.0 Ethernet controller: Mellanox Technologies MT2910 Family [ConnectX-7]
98:00.0 Ethernet controller: Mellanox Technologies MT2910 Family [ConnectX-7]
```

# nvidia-smi
## GPU拓扑
```bash
nvidia-smi topo -m

或者 

lspci -t 
```
![](../集合通信/attachments/Pasted%20image%2020260423152444.png)


# 参考
```bash

```