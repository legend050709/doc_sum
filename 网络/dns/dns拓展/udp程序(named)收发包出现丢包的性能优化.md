```table-of-contents
```
# 背景
# 收包丢包
## 统计查看
## 设置
```bash
sysctl -w net.core.rmem_max=8000000
```
## 注意
`rmem_max` 设置之前，程序已经启动，那么其实是不生效的。
设置 `rmem_max` 之后， 需要重启 named 程序 才可以生效。

## 丢包范例的分析流程

# 发包丢包
## 统计查看
## 设置
```bash
ip link set ethx txqueuelen 4000
```
## 注意
## 丢包范例的分析流程

#  参考
```bash

```