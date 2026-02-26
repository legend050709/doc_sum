```table-of-contents
```
# 介绍
## 为什么不用直接 `sudo` / root 跑程序？

|问题|说明|
|---|---|
|权限过大|一旦漏洞 = 系统沦陷|
|运维风险|误删文件、误操作|
|安全审计差|很难限制|


# 常见使用

# 使用场景
## `setcap cap_net_raw+ep your_program`
在一个多接口的设备上，选择指定的接口作为出口；
```bash
SO_BINDTODEVICE
   ↓
需要 CAP_NET_RAW
   ↓
setcap cap_net_raw+ep your_program
   ↓
非 root 也能精确控制出接口

```
# 参考
```bash
# 权限细分工具——Linux Capabilities 和 setcap 简介
https://zhuanlan.zhihu.com/p/693896673
```