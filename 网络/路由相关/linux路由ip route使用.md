```table-of-contents
```
# 配置ECMP
配置如下：
```
ip route add 100.0.0.0/16 nexthop via 1.1.1.1 dev vnet0 weight 1  nexthop via 2.2.2.1 dev vnet1 weight 1 
```

说明：
- `nexhop` 指定下一跳
- `dev` 指定接口
- `weight` 指定权重

查看配置：
```
$ ip route show 100.0.0.0/16
100.0.0.0/16 proto zebra  metric 20
  nexthop via 1.1.1.1 dev vnet0 weight 1 onlink
  nexthop via 2.2.2.1 dev vnet0 weight 1 onlink
```
# 参考
```c

```