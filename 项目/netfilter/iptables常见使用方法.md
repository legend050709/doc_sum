```table-of-contents
```
# 匹配条件
## ip层
### TTL匹配
- 匹配TTL **大于、等于、小于** n（如 2） 的报文
```c
iptables -t mangle -A POSTROUTING -m ttl --ttl-gt 2 -j DROP
iptables -t mangle -A POSTROUTING -m ttl --ttl-eq 2 -j DROP
iptables -t mangle -A POSTROUTING -m ttl --ttl-lt 2 -j DROP
```

- IPv6 hop limit 匹配
```c
ip6tables -t mangle -A PREROUTING -i eth1 -m hl --hl-eq 2 -j DROP
```
### 指定IP
匹配取反条件
```c
iptables -t filter -A INPUT ! -s 192.168.23.246 -j ACCEPT
```

### 指定网段
指定特定网段
```c
iptables -I INPUT -s 192.168.23.0/24 -J DROP
```


## tcp层
## udp层
## 应用层
### 内容匹配
# 动作
## 丢弃
## 打日志
## 打fwmark
## 设置ttl
**ipv4 ttl**
- 将TTL值 **设定** 为 n （如 2）
```c
iptables -t mangle -A PREROUTING -i eth0 -j TTL --ttl-set 2
```

- 将TTL值 **减小** n
```c
iptables -t mangle -A PREROUTING -i eth0 -j TTL --ttl-dec 2
```

- 将TTL值 **增大** n（一般情况下不要去增大TTL值）
```c
iptables -t mangle -A PREROUTING -i eth0 -j TTL --ttl-inc 2
```

**ipv6 hop limit**
```c
ip6tables -t mangle -A PREROUTING -i eth1 -j HL --hl-dec 2
```
## dnat
## snat

# 参考
```c

```