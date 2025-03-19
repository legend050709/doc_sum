```table-of-contents
```
# ib网络状态探测
在 InfiniBand 网络中，Host Channel Adapter（HCA）是关键组件，了解其状态和配置对于网络管理和故障排查至关重要。以下是一些常用的命令，用于查询和管理 HCA 的状态和配置。
## ibstatus
## ibstat

### 介绍
显示 HCA 的基本状态信息，包括设备状态、端口状态、链路速度等。
ibstat 输出：包括 HCA 的名称、固件版本、端口状态（如 `PORT_ACTIVE`）、最大和活动 MTU 等。


### 安装
```bash
# which ibstat
/sbin/ibstat

# rpm -qf /sbin/ibstat
infiniband-diags-51mlnx1-1.51237.x86_64
```


## ibportstate
## ibv_devinfo
## iblinkinfo
## ibqueryerrors
## ibtracert

# 网卡统计计数
## mellanox的RNIC网卡的统计
### 参考
```bash
# ib out_of_buffer问题分析
https://storage.blog.csdn.net/article/details/145187153
```
# 参考
```bash
# ib网络状态探测
https://storage.blog.csdn.net/article/details/145699299
```