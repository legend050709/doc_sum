```table-of-contents
```
# MSS的背景
# Linux 内核关于 MSS 的实现
## 结构体
## mss_cache的设置
### 主动端设置
### 被动端设置
### icmp差错报文设置
### 通信双方的mss_cache设置流程
# MSS的设置
## 整体流程
## sockopt设置
## 路由设置
## 接口mtu设置
# 参考
```c
# Linux内核协议栈中一些关于 TCP MSS 的细节
https://switch-router.gitee.io/blog/tcp-mss/

# TCP最大报文段MSS源码分析
https://www.cnblogs.com/wanpengcoder/p/11751292.html
```