```table-of-contents
```
# 背景
# 方案
## 方案一：wan口作为client，lan口作为 server

wan口单独再配置一个IP，和wan-ip同网段。
lan口单独再配置一个IP，和lan-ip同网段，作为RS-ip。
> 如果不是同一个网段，就可能需要在两端都用bird进行路由发布了。
> 另外，需dpip也将对应的ip添加到lan/wan口，因为给本机的报文需要查询本机路由才可以收到。


lan/wan口发业务流量包，经过dpvs上送到交换机之后，经过路由查询，再次回到DPVS。


## 方案二：使用tap/tun在一个机器上启动DPVS以及client和Server
# 参考
```bash
# DPVS构建线上模拟测试环境
https://blog.csdn.net/legend050709/article/details/125995267
```