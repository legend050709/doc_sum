```table-of-contents
```

# ipset设置
## ipset设置ip
## ipset设置net
## ipset设置port
## ipset 设置ipv6

# iptables使用ipset的方法
## `-m set`使用ipset
```bash
[ ! ] --match-set setname flag [,flag] ...
```
 flag可以通过指定为src或dst，flag最多数量为6个，中间使用逗号隔开。

```bash
# 创建名称为test的ipset
 
ipset create test hash:ip,port
 
ipset add test 192.168.1.105,tcp:80
 
# 丢弃来自源IP192.168.1.105，源端口80的包
 
iptables -A INPUT -m set --match-set test src,src -j DROP
 
# 丢弃来自源IP192.168.1.105，目的端口80的包
 
iptables -A INPUT -m set --match-set test src,dst -j DROP
 
# 丢弃访问目的IP192.168.1.105，目的端口80的包
 
iptables -A INPUT -m set --match-set test dst,dst -j DROP
 
 
 
# 创建名称为test1的ipset
 
ipset create test1 hash:ip,mac
 
ipset add test1 192.168.1.105,3C:7C:3F:D7:94:AF
 
# 丢弃来自源IP192.168.1.105，源MAC 3C:7C:3F:D7:94:AF的包
 
iptables -A INPUT -m set --match-set test1 src,src -j DROP
 
其他情况同test一样
```

## `--return-nomatch`选项
匹配被设置了`nomatch`标志的条目。

```bash
# 创建类型为hash:net的test3 ipset
ipset create test3 hash:net
 
# 添加nomatch, 匹配时跳过此条目
ipset add test3 192.168.1.0/24 nomatch
 
# iptables匹配test3中，设置为nomatch的条目的网络段的icmp包丢弃
iptables -I INPUT -m set --match-set test3 src --return-nomatch -p icmp -j DROP
```
## counters选项
**[!] --packets-eq value**
如果包匹配了ipset中的一个条目，并且包的数量等于（！不等于）设置的value值时，iptables的动作将作用于此条目。

**--packets-lt value**
用法同`--packets-eq value`，但是是当包的数量小于value

**--packets-gt value**
用法同`--packets-eq value`，但是是当包的数量大于value

**[!] -bytes-eq value; --bytes-lt value; --bytes-gt value**
用法同`--packets-eq`，`--packets-lt`，`--packets-gt`。只是匹配的是字节数。


```bash
# 创建test ipset，记得要指定counters选项
ipset create test hash:ip counters
 
# 添加192.168.1.105到test ipset
ipset add test 192.168.1.105 
 
# 匹配到小于3个icmp包的ipset条目丢弃
iptables -I INPUT -m set --match-set test src --packets-lt 3 -p icmp -j DROP
 
# ipse list命令可看到当前包的数量统计和字节统计
[root@192 ~]# ipset list
Name: test
Type: hash:ip
Revision: 4
Header: family inet hashsize 1024 maxelem 65536 counters
Size in memory: 248
References: 1
Number of entries: 1
Members:
192.168.1.105 packets 4 bytes 240
 
```
如果指定了counters选项创建一个IP set时，匹配的条目的包和字节的数量会增加，如下所示。
```bash
[root@192 ~]# ipset list
Name: test
Type: hash:ip
Revision: 4
Header: family inet hashsize 1024 maxelem 65536 counters
Size in memory: 248
References: 1
Number of entries: 1
Members:
192.168.1.105 packets 4 bytes 240
 
[root@192 ~]# ipset list
Name: test
Type: hash:ip
Revision: 4
Header: family inet hashsize 1024 maxelem 65536 counters
Size in memory: 248
References: 1
Number of entries: 1
Members:
192.168.1.105 packets 6 bytes 360
```
如果指定`！--update-counters`，`IP set`条目后的`packets`和`bytes`匹配到了包，数量不会增加。

## 小结
使用方法如下：
- iptables如果使用ipset，需要指定"-m set"。
- iptables可以用"--nomatch"选项，专门匹配ipset中带nomatch的条目。
- iptables要根据IP set中条目包数量动作时，需要ipset创建IP set时，要指定counters选项。

## 范例

# 参考
```c
# ipset 理解
https://liupeng0518.github.io/2018/12/24/linux/%E5%AE%89%E5%85%A8/ipset/

# iptables和ipset配合使用
https://blog.csdn.net/to_be_better_wen/article/details/127416638
```