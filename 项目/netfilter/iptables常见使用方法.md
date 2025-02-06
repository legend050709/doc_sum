```table-of-contents
```
# 操作命令command
```text
-t 指定配置表
-A, ––append 将规则添加到链中（最后）。
-I, ––insert 将规则添加到给定位置的链中。
-C, ––check 寻找符合链条要求的规则。
-D, ––delete 从链中删除指定的规则。
-R, --replace 更改
-F, ––flush 删除对应表的所有规则，慎重使用。
-L, ––list 连锁显示所有规则。
-v, ––verbose 使用列表选项时显示更多信息。
-P, --policy 设置链的默认策略(policy)
-N, --new 创建用户自定义链
-X, --delete-chain 删除用户自定义链
-E, --rename-chain 重命名用户自定义链
-j target 决定符合条件的包到何处去，target模式很多
-Z或者--zero //将指定链中的所有规则的包字节计数器清零。
```

常见的操作：
```bash
iptables [-t table] {-A|-C|-D} chain rule-specification
iptables [-t table] -I chain [rulenum] rule-specification
iptables [-t table] -R chain rulenum rule-specification
iptables [-t table] -D chain rulenum
iptables [-t table] -S [chain [rulenum]]
iptables [-t table] {-F|-L|-Z} [chain [rulenum]] [options...]
iptables [-t table] -N chain
iptables [-t table] -X [chain]
iptables [-t table] -P chain target
iptables [-t table] -E old-chain-name new-chain-name
rule-specification = [matches...] [target]
match = -m matchname [per-match-options]
target = -j targetname [per-target-options] //target 即 action
```

记忆：
```bash
iptables [-t table] {-A|-C|-D|-F|-L} chain MATCH-RULE [-j target]

MATCH-RULE: 匹配规则
-j target： 动作
```


iptables的版本：
```bash
# iptables -V
iptables v1.4.21
```
# 匹配match

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
### 设置mss 
```bash
# ipv4 设置mss
vip4SetName="vip4List"
ipset create ${vip4SetName} hash:ip family inet
ipset add ${vip4SetName} 1.1.1.1
iptables -t mangle -I OUTPUT -m set --match-set ${vip4SetName} src -p tcp --tcp-flags SYN,ACK SYN,ACK -j TCPMSS --set-mss ${synAckMss}

# 查看



```
## udp层

# 动作
常用的ACTION，如下所示：
```bash
DROP：悄悄丢弃，一般我们多用DROP来隐藏我们的身份，以及隐藏我们的链表
REJECT：明示拒绝
ACCEPT：接受 custom_chain：转向一个自定义的链
DNAT：
SNAT：
MASQUERADE：源地址伪装
REDIRECT：重定向：主要用于实现端口重定向
MARK：打防火墙标记的
RETURN：返回，在自定义链执行完毕后使用返回，来返回原规则链。
```

## 丢弃
## 打日志
## DSCP/TOS打标
### 背景
在使用专线，传输跨机房的流量时，需要对不同的业务流量打不同的TOS标签进行QOS管理。TOS一般分为金牌，银牌，铜牌。

金牌：一般是比较重要的业务，比如支付，商业化，重要的基础服务，比如数据库，安全网关access-proxy等。
银牌：默认流量，即不存在tos值。
铜牌：不太重要的流量，比如离线流量。

### 处理
```bash
iptables -t mangle -A POSTROUTING -p tcp -j DSCP --set-dscp 0x22
```
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





# iptables规则的顺序
## 生效的顺序
iptables规则是按照从上到下的顺序逐一匹配的，iptables 多条规则有冲突的时候，排在上面的规则优先。因此，规则的排列顺序直接影响了数据包的处理流程。

比如我们已经设置了 `iptables -A INPUT -p udp –dport 53 -j REJECT`
那么如果再执行`iptables -A INPUT -p udp –dport 53 -s 180.169.223.10 -j ACCEPT` ，则不会生效
`-A`参数是`append`，添加的规则会放在追后面，而前面已经有`REJECT` 该端口所有的访问了，那么这条`ACCEPT`就不会生效。

所以这里`-A`要改成`-I`，也就是`insert`的意思，插入一条记录，那么这条就会放在最前面，就在那条`REJECT`前面了，这样就能生效。
```bash
iptables -I INPUT -p udp --dport 53 -s 180.169.223.10 -j ACCEPT
```
这样我们就能在拒绝所有地址访问我们的udp 53端口之后，指定给180.169.223.10能访问了。
那如果我不想把新的规则加入到最前面，也不想加在最后，我要放到一个中间指定的地方，怎么做呢？ 使用-I 的同时，加入编号就可以了。
```bash
iptables -I INPUT 3 -p tcp --dport 80 -s 180.168.233.10 -j ACCEPT
```

##  查询当前iptables的规则number
```bash
iptables -L -n --line-numbers    ##所有链的规则number

iptables -L INPUT --line-numbers ## 查看INPUT的

iptables -L OUTPUT --line-numbers ## 查看OUTPUT的

iptables -L FORWARD --line-numbers ##查看FORWARD的
```

## 基于ID删除以及更改rule
![](attachments/Pasted%20image%2020231227151624.png)
```
iptables -t nat -D PREROUTING 1
iptables -t nat -D POSTROUTING 2
```

# 其他
## ipv6的iptables规则

# 参考
```c

```