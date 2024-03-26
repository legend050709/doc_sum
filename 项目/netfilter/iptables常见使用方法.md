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


# 扩展模块
## iprange扩展模块
### 介绍
在不使用任何扩展模块的情况下，使用-s选项或者-d选项即可匹配报文的源地址与目标地址，而且在指定IP地址时，可以同时指定多个IP地址，每个IP用”逗号”隔开，但是，-s选项与-d选项并不能一次性的指定一段连续的IP地址范围，如果我们需要指定一段连续的IP地址范围，可以使用iprange扩展模块。

使用iprange扩展模块可以指定”一段连续的IP地址范围”，用于匹配报文的源地址或者目标地址。
iprange扩展模块中有两个扩展匹配条件可以使用
–src-range
–dst-range

### 范例
![](attachments/Pasted%20image%2020240325111604.png)
上例表示如果报文的源IP地址如果在192.168.1.127到192.168.1.146之间，则丢弃报文，IP段的始末IP使用”横杠”连接，–src-range与–dst-range和其他匹配条件一样，能够使用”!”取反。

### 其他
也可以使用 `ipset` 达到和 `iprange` 一样的效果。
但是ipset比iprange更加灵活，可以在不更改 iptables规则的情况下，更改ipset的内容，来达到更新网段的作用。

### 小结
包含的扩展匹配条件如下
–src-range：指定连续的源地址范围
–dst-range：指定连续的目标地址范围
```bash
#示例
iptables -t filter -I INPUT -m iprange --src-range 192.168.1.127-192.168.1.146 -j DROP
iptables -t filter -I OUTPUT -m iprange --dst-range 192.168.1.127-192.168.1.146 -j DROP
iptables -t filter -I INPUT -m iprange ! --src-range 192.168.1.127-192.168.1.146 -j DROP
```

## limit限速模块
### 介绍
`limit`模块主要是用于限制一个时间段内进入系统的数据包，主要的用途是减少泛洪攻击的影响。

我们能控制某条规则在一段时间内的匹配次数（也就是能匹配的包的数量），这样就能够减少 `DoS`、`syn flood`攻击的影响。这是他的主要作用，当然，更有非常多其他作用（注：比如，对于某些不常用的服务能限制连接数量，以免影响其他服务）。

`limit`模块引入了**令牌桶**的机制，使用`--limit-burst`指定令牌桶的大小，当令牌桶中有令牌时，则可以匹配到数据包，如果没有了令牌，则需要等到`--limit`指定的时间单位间隔向令牌桶加入令牌，才能继续匹配数据包。

### 理解
limit match的工作方式就像一个单位大门口的保安，当有人要进入时，需要找他办理通行证。早上上班时，保安手里有一定数量的通行证，来一个人，就签发一个，当通行证用完后，再来人就进不去了，但他们不会等，而是到别的地方去（在iptables里，这相当于一个包不符合某条规则，就会由后面的规则来处理，如果都不符合，就由缺省的策略处理）。
但有个规定，每隔一段时间保安就要签发一个新的通行证。这样，后面来的人如果恰巧赶上，也就能进去了。如果没有人来，那通行证就保留下来，以备来的人用。如果一直没人来，可用的通行证的数量就增加了，但不是无限增大的，最多也就是刚开始时保安手里有的那个数量。也就是说，刚开始时，通行证的数量是有限的，但每隔一段时间就有新的通行证可用。
`limit match`有两个参数就对应这种情况：
`--limit-burst`：指定刚开始时有多少通行证可用；其实`--limit-burst`就是指定令牌桶的最大容量。
`--limit`：指定要隔多长时间才能签发一个新的通行证；其实`--limit`表示向令牌桶放入令牌的速率。

要注意的是，我这里强调的是“签发一个新的通行证”，这是以iptables的角度考虑的。在你自己写规则时，就要从这个角度考虑。比如，你指定了--limit 3/minute --limit-burst 5 ，意思是开始时有5个通行证，用完之后每20秒增加一个（这就是从iptables的角度看的，要是以用户的角度看，说法就是每一分钟增加三个或每分钟只能过三个）。你要是想每20分钟过一个，只能写成--limit 3/hour --limit-burst 5，也就是说你要把时间单位凑成整的。


###  使用
`--limit`和`--limit-burst`规则匹配是iptables对数据包众多的匹配的方式中的一种，使用之前需要用"-m limit"，表示使用`limit`模块。

`--limit`和`--limit-burst`是配合使用的，如果`iptables`规则只使用了`--limit`，则表示使用`--limit-burst`的默认值为5。如果只指定了`--limit-burst`，则`--limit`默认值`3/hour`。

`limit match`也能用英文感叹号取反，如：`-m limit ! --limit 5/s`表示在数量超过限定值后，所有的包都会被匹配。

### 个数or速率限制
```bash
# --limit
iptables -A INPUT -m limit --limit 3/hour
# 为limit match设置最大平均匹配速率，也就是单位时间内limit match能匹配几个包。
他的形式是个数值加一个时间单位，能是/second /minute /hour /day 。
默认值是每小时3次（用户角度），即3/hour ，也就是每20分钟一次（iptables角度）。
```

```bash
iptables -A INPUT -p icmp -m limit --limit 6/m --limit-burst 5 -j ACCEPT
iptables -P INPUT DROP
# 然后从另一部主机上ping这部主机，就会发生如下的现象：
# 首先我们能看到前四个包的回应都非常正常，然后从第五个包开始，我们每10秒能收到一个正常的回应。# 这是因为我们设定了单位时间(在这里是每分钟)内允许通过的数据包的个数是每分钟6个，也即每10秒钟一个；# 其次我们又设定了事件触发阀值为5，所以我们的前四个包都是正常的，只是从第五个包开始，限制规则开始生效，故只能每10秒收到一个正常回应。
# 假设我们停止ping，30秒后又开始ping，这时的现象是：
# 前两个包是正常的，从第三个包开始丢包，这是因为在这里我的允许一个包通过的周期是10秒，如果在一个周期内系统没有收到符合条件的包，系统的触发值就会恢复1# 所以如果我们30秒内没有符合条件的包通过，系统的触发值就会恢复到3，如果5个周期内都没有符合条件的包通过，系统都触发值就会完全恢复。
```

### 小结
常用的扩展匹配条件如下
–limit-burst：类比”令牌桶”算法，此选项用于指定令牌桶中令牌的最大数量。
–limit：类比”令牌桶”算法，此选项用于指定令牌桶中生成新令牌的频率，可用时间单位有second、minute 、hour、day。
```bash
#示例 #注意，如下两条规则需配合使用，具体原因上文已经解释过，忘记了可以回顾。
iptables -t filter -I INPUT -p icmp -m limit --limit-burst 3 --limit 10/minute -j ACCEPT
iptables -t filter -A INPUT -p icmp -j REJECT
```

## string扩展模块
### 介绍
使用string扩展模块，可以指定要匹配的字符串，如果报文中包含对应的字符串，则符合匹配条件。

### 范例
比如，如果报文中包含字符”OOXX”，我们就丢弃当前报文。
首先，我们在IP为146的主机上启动http服务，然后在默认的页面目录中添加两个页面，页面中的内容分别为”OOXX”和”Hello World”，如下图所示，在没有配置任何规则时，126主机可以正常访问146主机上的这两个页面。
![](attachments/Pasted%20image%2020240325105820.png)

那么，我们想要达到的目的是，如果报文中包含”OOXX”字符，我们就拒绝报文进入本机，所以，我们可以在126上进行如下配置。

![](attachments/Pasted%20image%2020240325105833.png)

上图中，’-m string’表示使用string模块，’--algo bm’表示使用bm算法去匹配指定的字符串，’ --string “OOXX” ‘则表示我们想要匹配的字符串为”OOXX”

设置完上图中的规则后，由于index.html中包含”OOXX”字符串，所以，146的回应报文无法通过126的INPUT链，所以无法获取到页面对应的内容。

### 小结
总结一下string模块的常用选项：
–algo：指定对应的匹配算法，可用算法为bm、kmp，此选项为必需选项。
–string：指定需要匹配的字符串
```bash
#示例
iptables -t filter -I INPUT -p tcp --sport 80 -m string --algo bm --string "OOXX" -j REJECT
iptables -t filter -I INPUT -p tcp --sport 80 -m string --algo bm --string "OOXX" -j REJECT
```

## time扩展模块
### 介绍
我们可以通过time扩展模块，根据时间段区匹配报文，如果报文到达的时间在指定的时间范围以内，则符合匹配条件。

### 范例
比如，”我想要自我约束，每天早上9点到下午6点不能看网页”，擦，多么残忍的规定，如果你想要这样定义，可以尝试使用如下规则。
![](attachments/Pasted%20image%2020240325110101.png)
上图中”-m time”表示使用time扩展模块，`--timestart`选项用于指定起始时间，`--timestop`选项用于指定结束时间。

如果你想要换一种约束方法，只有周六日不能看网页，那么可以使用如下规则。
![](attachments/Pasted%20image%2020240325110144.png)
使用–weekdays选项可以指定每个星期的具体哪一天，可以同时指定多个，用逗号隔开，除了能够数字表示”星期几”,还能用缩写表示，例如：Mon, Tue, Wed, Thu, Fri, Sat, Sun；

当然，你也可以将上述几个选项结合起来使用，比如指定只有周六日的早上9点到下午6点不能浏览网页。
![](attachments/Pasted%20image%2020240325110209.png)

使用–monthdays选项可以具体指定的每个月的哪一天，比如，如下图设置表示指明每月的22号，23号。
![](attachments/Pasted%20image%2020240325110233.png)

前文已经总结过，**当一条规则中同时存在多个条件时，多个条件之间默认存在”与”的关系**。
下图中的设置表示匹配的时间必须为星期5，并且这个”星期5″同时还需要是每个月的22号到28号之间的一天，所以，下图中的设置表示每个月的第4个星期5：
![](attachments/Pasted%20image%2020240325110317.png)


除了使用–weekdays选项与–monthdays选项，还可以使用–datestart 选项与-datestop选项，指定具体的日期范围，如下。
![](attachments/Pasted%20image%2020240325110340.png)

### 小结
常用扩展匹配条件如下
–timestart：用于指定时间范围的开始时间，不可取反
–timestop：用于指定时间范围的结束时间，不可取反
–weekdays：用于指定”星期几”，可取反
–monthdays：用于指定”几号”，可取反
–datestart：用于指定日期范围的开始日期，不可取反
–datestop：用于指定日期范围的结束时间，不可取反
```bash
#示例
iptables -t filter -I OUTPUT -p tcp --dport 80 -m time --timestart 09:00:00 --timestop 19:00:00 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 443 -m time --timestart 09:00:00 --timestop 19:00:00 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --weekdays 6,7 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --monthdays 22,23 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time ! --monthdays 22,23 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --timestart 09:00:00 --timestop 18:00:00 --weekdays 6,7 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --weekdays 5 --monthdays 22,23,24,25,26,27,28 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --datestart 2017-12-24 --datestop 2017-12-27 -j REJECT
```

## connlimit扩展模块
### 介绍
使用connlimit扩展模块，可以限制每个IP地址同时链接到server端的链接数量，注意：我们不用指定IP，其默认就是针对”每个客户端IP”，即对单IP的并发连接数限制。

### 范例
我们想要限制，每个IP地址最多只能占用两个ssh链接远程到server端，我们则可以进行如下限制。
![](attachments/Pasted%20image%2020240325110737.png)

上例中，使用”-m connlimit”指定使用connlimit扩展，使用”–connlimit-above 2″表示限制每个IP的链接数量上限为2，再配合-p tcp –dport 22，即表示限制每个客户端IP的ssh并发链接数量不能高于2。

**centos6**中，我们可以对–connlimit-above选项进行取反，没错，老规矩，使用”!”对此条件进行取反，示例如下
![](attachments/Pasted%20image%2020240325110754.png)
上例表示，每个客户端IP的ssh链接数量只要不超过两个，则允许链接。
上例的规则**并不能表示**：每个客户端IP的ssh链接数量超过两个则拒绝链接。
也就是说，即使我们配置了上例中的规则，也不能达到”限制”的目的，所以我们通常并不会对此选项取反，因为既然使用了此选项，我们的目的通常就是”限制”连接数量。

> 注：centos7中iptables为我们提供了一个新的选项，`--connlimit-upto`，这个选项的含义与`! –commlimit-above`的含义相同，即链接数量未达到指定的连接数量之意，所以综上所述，`--connlimit-upto`选项也不常用。


刚才说过，–connlimit-above默认表示限制”每个IP”的链接数量，其实，我们还可以配合–connlimit-mask选项，去限制”某类网段”的链接数量，示例如下：
![](attachments/Pasted%20image%2020240325111019.png)
上例中，”–connlimit-mask 24″表示某个C类网段，没错，mask为掩码之意，所以将24转换成点分十进制就表示255.255.255.0，所以，上图示例的规则表示，一个最多包含254个IP的C类网络中，同时最多只能有2个ssh客户端连接到当前服务器，看来资源很紧俏啊！254个IP才有2个名额，如果一个IP同时把两个连接名额都占用了，那么剩下的253个IP连一个连接名额都没有了。

即：在不使用–connlimit-mask的情况下，连接数量的限制是针对”每个IP”而言的，当使用了–connlimit-mask选项以后，则可以针对”某类IP段内的一定数量的IP”进行连接数量的限制

### 小结
常用的扩展匹配条件如下
–connlimit-above：单独使用此选项时，表示限制每个IP的链接数量。
–connlimit-mask：此选项不能单独使用，在使用–connlimit-above选项时，配合此选项，则可以针对”某类IP段内的一定数量的IP”进行连接数量的限制，如果不明白可以参考上文的详细解释。
```bash
#示例
iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 2 -j REJECT
iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 20 --connlimit-mask 24 -j REJECT
iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 10 --connlimit-mask 27 -j REJECT
```

# 其他
## ipv6的iptables规则

# 参考
```c

```