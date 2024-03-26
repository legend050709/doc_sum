```table-of-contents
```

# 定义
所谓区域传输(zone transfer)，就是辅助域名服务器与主域名服务器通信，并同步 RR 资源的过程。这样做的目的是为了保证多台服务器保证内容同步。

# 传输协议
**区域传输使用 TCP 而不是 UDP**

如果使用UDP，限制传输数据 512B以内（DNS的UDP响应就是限制在512B以内）。由于数据同步传送的数据量比一个 DNS 请求和响应报文的数据量要多得多，因此使用TCP。
![](attachments/Pasted%20image%2020240104133505.png)

因此，**UDP 用于 client 和 server 的查询和响应**，**TCP 用于主从 server 之间的Zone传送**。

# 特性
**更改自动同步**
RFC 标准协议通过 MASTER-SLAVE 架构，NOTIFY + XFR 机制实现数据自动同步，用户只需要在主服务器上更改域名，更改信息便可自动同步到从服务器 。
![](attachments/Pasted%20image%2020240104162335.png)

# zone同步时机
## 时机一：启动同步
（1）slave重启
当辅助域名服务器启动时，slave主动去master上获取区域数据。。

（2）master重启
此时master会发送notify消息给slave。但要求区域数据文件中定义了slave的ns记录及其A记录，否则找不到slave，也就联系不上slave。

## 时机二：定时检查
### 从服务器的定时检查
辅域名服务器会定时向主域名服务器进行查询以便了解区域是否有变动。如有变动，则会执行一次区域传输。

一旦启动区域传输，就会存在两种传输方式：
1. 全量传输(AXFR)：即传输整个区域的消息，全量传输会传输整个区域的消息。
2. 增量传输(IXFR)：增量传输就是传输一部分消息，增量传输使用的消息。

### 主服务器的定时notify
如果主DNS服务配置了`heartbeat-interval`为5；
则 master每5分钟会给全部slave发送notify消息。并且是给每个view的每个zone发送一条notify消息。没有做打散处理。发送notify消息的时候，会影响master的性能。影响变更生效的速度。

> 注：可以关闭 master的 定时全量更新通知。将 master 的`heartbeat-interval` 设置为0 即可。


### AXFR
![](attachments/Pasted%20image%2020231225111500.png)
全区域传输（AXFR：Full Zone Transfer）：
主 DNS 服务器通知辅助 DNS 服务器已对**特定区域**进行了更改，辅助 DNS 与主 DNS 联系以检查发生更改的区域的 SOA 记录中的序列号。如果主 DNS 上的序列号大于该区域的辅助 DNS 服务器的序列号，则整个区域文件将从主 DNS 服务器复制到辅助 DNS 服务器。
> 注：AXFR传输指的是对某个Zone的全部RR记录进行传输，而不是所有的Zone的所有记录进行传输。

**axfr query**
![](attachments/Pasted%20image%2020240115203521.png)

**axfr response**
![](attachments/Pasted%20image%2020240115204308.png)
对于response，axfr结果放在`answers section`内，开头和结尾的SOA记录表示左括号和右括号，当中的内容表示这个zone的所有记录。

### IXFR
![](attachments/Pasted%20image%2020231225112241.png)
增量区域传输（IXFR：Incremental Zone Transfer）：
主 DNS 服务器通知辅助 DNS 服务器已对**特定区域**进行了更改，辅助 DNS 与主 DNS 联系以检查发生更改的区域的 SOA 记录中的序列号。如果主 DNS 上的序列号大于该区域的辅助 DNS 服务器的序列号，则辅助 DNS 服务器会将上次更改与现有版本进行比较，并仅从主 DNS 复制更改的记录。

**ixfr query**
![](attachments/Pasted%20image%2020240115204422.png)
与axfr不同的是，ixfr除`query type`与axfr不同，还会额外在第三section也就是表示权威服务器（author ns）区域携带一条slave当前的SOA记录，由master根据slave的序列号（上图红框），判别增量更新信息，并返回给slave。

**ixfr response**
![](attachments/Pasted%20image%2020240115204444.png)
对于应答，上图包含4个SOA，同样在`answer section`内，开始和结尾的soa含义仍旧和axfr保持一致，代表左括号和右括号。
第二个SOA为老的序列号，后面跟的是需要删除掉的RRs；第三个SOA为较新的序列号，表示需要增加的RRs。

下面截取rfc1995中的部分内容，多个版本更新可以顺次将更新串起来，也可以直接经过运算，得到最终增量结果。
![](attachments/Pasted%20image%2020240115205057.png)


**ixfr的返回分类**：
```
1. ixfr获取增量失败，增量信息不完整或已丢失，直接返回全量结果，answer section同axfr （SOA、records&#8230;..、SOA)
2. ixfr请求序列号和最新序列号一致，answer区域仅返回一条SOA记录
3. ixfr请求序列号小于最新序列号，但无更新RR，直接返回两条SOA(SOA、SOA)
4. ixfr请求序列号大于master最新序列号，异常，返回axfr
```


### 范例
常规情况下，zone每发生一次变化，序列号加1，通过序列号标识版本，获取增量更新信息。
![](attachments/Pasted%20image%2020240115203238.png)

如上，忽略tcp的建连过程，
1，2号报文为notify的通告和响应；通告即zone发生变化，master通告slave。
3，4号报文为SOA的查询和响应；slave发起，请求master最新的序列号
10，12号报文为ixfr更新请求和响应；slave发起，请求更新zone信息（axfr同理，也是这样）；

## 时机三：DNS NOTIFY
但是使用轮询这种方式有一些弊端，因为从服务器会定期检查主服务器上内容是否更新，这是一种资源浪费，因为绝大多数情况下都是一次无效检查，所以为了改善这种情况，DNS 设计了 `DNS NOTIFY` 机制，`DNS NOTIFY` 允许修改区域内容后主服务器通知从服务器内容需要更新，应该启动区域传输。


**`master`发送`notify`消息的作用**：
这将激活辅域名服务器中的对域的序列号的检验。
因此，其实监控到了`master`和`slave`的`zone`的配置不一致，其实也可以其他的服务器，甚至`slave`自身，给`slave`发送一条`notify`消息，这样也可以触发`slave`服务器对域的序列号的检查。

### 变更的同步过程
主从权威的同步过程如下：
（1）主节点 Zone 配置变更，向从节点发送 NOTIFY 通知
（2）从节点返回 NOTIFY Respons，并向主节点发起 SOA 查询
（3）主节点返回 SOA Respons
（4）从节点对比 SOA Respons 中的序列号是否比自身序列号大，仅当 SOA Respons 序列号大于自身序列号时才发起 `Zone transfer request`，并利用 **TCP 53 端口**进行数据传输。
（5）主节点收到 `Zone transfer request`，进行响应。

因此 Zone 配置变更后必须增大序列号，否则会导致主从节点数据不一致。
# zone文件更新
## 静态域维护
前提条件：
在区域配置文件中 添加 allow-update { none; }; 表示不允许动态更新。

如下所示：
```shell
[root@dns1 ~]# cat /etc/named.rfc1912.zones 
zone "host.com" IN {
        type  master;
        file  "host.com.zone";
        allow-update { none; };
};
```

每次配置更改区域数据库文件(zone文件)，需要手动前滚`serial number`，通知辅DNS更新。
```shell
[root@dns1 ~]# cat /var/named/host.com.zone
$TTL 600        ;10 minutes
@                       IN SOA  dns.host.com. test.qq.com. (
                                2021070705 ; serial number  需要手动+1
                                10800      ; refresh (3 hours)
                                900        ; retry (15 minutes)
                                604800     ; expire (1 week)
                                86400      ; minimum (1 day)
                                )
                        NS   dns.host.com.
$ORIGIN host.com.
$TTL 60 ; 1 minute
dns1                    A    192.168.10.222
dns2                    A    192.168.10.223
test                    A    192.168.10.55
dns                     A    192.168.10.222
```

## nsupdate进行动态域维护
前提条件：在区域配置文件中 添加 allow-update { acl; }; 表示根据acl指定策略进行动态更新。可填写ip地址。

```shell
[root@dns1 ~]# cat /etc/named.rfc1912.zones 
zone "host.com" IN {
        type  master;
        file  "host.com.zone";
        allow-update { 192.168.10.222; };
};
```
每次配置更改区域数据库文件，不需要手动前滚`serial number`，自动通知辅DNS更新。

### nsupdate
`nsupdate`是一个动态`DNS`更新工具，可以向DNS服务器提交更新记录的请求，它可以从区文件中添加或删除资源记录，而不需要手动进行编辑区文件。
使用 `nsupdate` 等工具进行动态配置。​ 使用`nsupdate` 不会更改区域数据库文件，而是产生了一个`jnl`的数据文件，不能使用文本编辑器打开，只能使用完全区域数据传送查看。
注：`jnl`文件（`journal`文件）是`BIND9`动态更新的时候记录更新内容所生成的日志文件。

**优缺点**
- 优点
	- 命令简单，便于记忆
	- 不用手动变更SOA的serial序列号，自动滚动
	- 不需要重启/重载BIND9服务/配置，生效快
	- 可以通过配置acl实现远程管理


- 缺点
	- jnl文件无法使用文本文件的方式打开
	- 只能依赖完全区域传送查看所有区域的记录
	- 更新操作复杂，先删再增
	- 远程管理有安全隐患，需要加强审计
	- 动态域在rndc管理上多一步


**使用方法**
```shell
#发送请求到servername服务器的port端口.如果不指定servername,nsupdate将把请求发送给当前去的主DNS服务器.
server servername [ port ]

# 添加一条资源记录
update add domain-name ttl [ class ] type data…

# 删除domain-name的资源记录.如果指定了type和data,仅删除匹配的记录。
update delete domain-name [ ttl ] [ class ] [ type [ data... ] ]


# 将要求信息和更新请求发送到DNS服务器.等同于输入一个空行
send

# 退出nsupdate工具
quit
```



**范例**
```shell
# 前提条件：允许设备动态修改配置，将以下参数修改为any或者指定ip。
allow-update { any; };

添加记录

[root@dns1 ~]# nsupdate
> server 192.168.10.222
> update add aaa.host.com 600 IN A 192.168.10.224
> send
> quit 
```
查看测试结果：
```shell
[root@dns2 ~]# dig -t AXFR host.com @192.168.10.222

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.5 <<>> -t AXFR host.com @192.168.10.222
;; global options: +cmd
host.com.               600     IN      SOA     dns.host.com. test.qq.com. 2021070706 10800 900 604800 86400
host.com.               600     IN      NS      dns.host.com.
aaa.host.com.           600     IN      A       192.168.10.224
dns.host.com.           60      IN      A       192.168.10.222
dns1.host.com.          60      IN      A       192.168.10.222
dns2.host.com.          60      IN      A       192.168.10.223
test.host.com.          60      IN      A       192.168.10.55
host.com.               600     IN      SOA     dns.host.com. test.qq.com. 2021070706 10800 900 604800 86400
;; Query time: 0 msec
;; SERVER: 192.168.10.222#53(192.168.10.222)
;; WHEN: Fri Jul 16 10:09:54 CST 2021
;; XFR size: 8 records (messages 1, bytes 234)
```

# DNS NOTIFY机制
主DNS服务器可以通知从DNS服务器进行区域传送。
![](attachments/Pasted%20image%2020231225113423.png)

## 同步流程
- master发送`notify`信息给`slave`
- `slave`去查询主服务器的`SOA`记录
- `master`将SOA记录发送给`slave`
- `slave`根据SOA记录去检查`serial number`是否有递增更新
- 如果有的话，`slave`向`master`发起`zone transfer`请求，然后`master`返回响应结果，`slave`更新记录。如果没有的话就说明不需要更新。

## 区分主从服务器
它是这样判断哪些是主DNS服务器哪些是从DNS服务器的：
**找到区域数据文件中的所有NS记录，并排除本机以及SOA记录中MNAME的那个服务器(SOA记录的MNAME列即IN关键字后的一列，一般就是主DNS服务器)，剩余的都是从DNS服务器。**

主DNS为每个自定义的区都发送notify声明给所有从服务器，告知其哪些区改变了；
slave服务器接收到notify声明后响应主DNS服务器告知它已经收到了通知；
然后slave服务器向主DNS服务器发起查询，以确定notify声明中所通告的区的SOA记录是否真的发生了改变，如果SOA发生了改变，则进行区域传送，如果没有改变则不进行传送。

## 工作流程
notify是这样工作的：
**当主DNS服务器重启了DNS服务或者通过NSUPDATE动态修改了域名解析记录时，则通知所有slave DNS服务器来更新区域数据。**(有些地方模糊地说SOA序列号发生改变也会发送notify，其实不然，因为区域数据文件需要编译加载到内存，简单的修改Zone数据库文件是无效的）

![](attachments/Pasted%20image%2020240104162652.png)

（1）用户在 MASTER 上动态修改域名解析记录（如 NSUPDATE），修改成功后，域名所在 ZONE 的版本号加 1。
`test.com`初始配置：
![](attachments/Pasted%20image%2020240104162738.png)

初始 SOA 序列号：
![](attachments/Pasted%20image%2020240104162805.png)

NSUPDTA 新增记录：
![](attachments/Pasted%20image%2020240104162815.png)

最新 SOA 序列号
![](attachments/Pasted%20image%2020240104162831.png)

（2）MASTER 向其配置的 SLAVE 节点发送 NOTIFY（一般是 UDP 报文），NOTIFY 信息中包含了修改域名所在的 ZONE 和该 ZONE 最新的版本号。

NOTIFY 消息：
![](attachments/Pasted%20image%2020240104162903.png)

（3）SLAVE 在收到 NOTIFY 消息后，进行以下操作：
- SLAVE 在收到 NOTIFY 消息后会给 MASTER 发送一个响应表示收到了 NOTIFY;
- SLAVE 比较 NOTIFY 中的 ZONE 的版本号和本地的 ZONE 的版本号，如果本地的版本号不低于 NOTIFY 中的版本号，SLAVE 不做任何操作;
- 如果 SLAVE 本地的版本号低于 NOTIFY 中的版本号，表示本地的 ZONE 数据已经落后，SLAVE 向 MASTER 发送 IXFR 请求; SLAVE 根据 REFRESH（定义在 ZONE 的 SOA 记录中）定时向 MASTER 发送 IXFR 请求，作为当 NOTIFY 的报文因为某些原因无法发送到 SLAVE 时的一种补偿机制。
- 如果 IXFR 失败，会转向 AXFR;

（4）MASTER 根据 SLAVE 请求的 XFR 类型返回对应的数据
IXFR 返回格式和结果：
![](attachments/Pasted%20image%2020240104163411.png)
![](attachments/Pasted%20image%2020240104163415.png)

AXFR 返回结果：
![](attachments/Pasted%20image%2020240104163511.png)

## 配置
bind 9中，notify默认是打开的，注意notify是写在主DNS服务器的named.conf中的，它的作用对象默认是所有的从服务器。使用下面的语句可以关闭：
```bash
options {  
    notify no;  
};
```

也可以将notify的开和关字句写在某个区域中，这样该设置会覆盖全局配置。
```bash
zone "fx.movie.edu" {  
    type master;  
    file "db.fx.movie.edu";  
    notify no;  
};
```

还可以使用also-notify来定义notify列表，这样除了会发送notify通知给从服务器还会发送给列表中定义的机器。also-notify字句可以写在某个区中，也可以写在全局配置options字句中。
```bash
zone "fx.movie.edu" {  
    type slave;  
    file "bak.fx.movie.edu";  
    notify yes;  
    also-notify { 15.255.152.4; };  
};
```
默认从服务器只接受来自其主DNS服务器的notify信息，非主DNS服务器的信息都会忽略，
但是可以使用allow-notify字句定义可以接受其他从服务器的notify信息。例如a是b和c的主，如果在c上定义`allow-notify { b_IP; };`，那么它也会接受b的notify信息。

## 其他
**为什么从DNS服务器接收到`notify`声明后还要再次查询主服务器上SOA记录来确认呢？**
第一是因为要比较序列号，决定是否要传送，以及要完全传送还是增量区域传送；
第二是因为有些人可能会发送假冒的notify声明给从DNS服务器，从而导致多余的区域传送。

# 区域传输限制
## 背景
我们在本地一台电脑上使用一个命令：
```bash
dig @115.29.32.62 liumapp.com axfr
```
不出意外，应该能够得到`liumapp.com`在`115.29.32.62`这台`DNS server`上的所有解析记录。
![](attachments/Pasted%20image%2020240114220534.png)
但是从安全角度来讲，我肯定不希望这样的事情发生，所以就要用到传输限制。

## 限制措施
默认情况下，`allow-transfer`的值为any，表示允许任何人都可以从此主机上执行区域传送。实际上，**应该设置主dns服务器只允许slave服务器来区域传送，并设置slave服务器不允许任何人区域传送**，这样就最大程度保证了区域数据不泄漏。

（1）基于主机的访问控制
通过主机IP来限制访问。
`allow-transfer : {address_list | none}` , 允许域传输的机器列表。

范例如下：
``` bash
zone "liumapp.com" {
 type  master;
 notify  yes;
 also-notify {106.14.212.41;};
 allow-transfer {106.14.212.41;};
 file "liumapp.com.zone";
};

zone "movie.edu" {  
     type master;  
     file "db.movie.edu";  
     allow-transfer { 192.249.249.1; 192.253.253.1; 192.249.249.9; 192.253.253.9; };  
};
```


（2）事务签名
通过密钥对数据进行加密。  比如`TSIG`（对称方式）或 `SIGO`（非对称方式）。
```bash
allow-transfer : {key keyfile} 
	(key及key的文件位置); 事务签名的key
```

### 测试
可以手动使用dig命令强制区域传送，只需使用-t指定区域传送的类型即可，如下：
```bash
dig -t AXFR  

dig -t ixfr=N
在指定增量区域传送时，需要指定序列号，只有比N大的序列号才会传送。
```
## 范例
（1）通过主机IP来限制访问。
在主服务器（`115.29.32.62`）上配置如下的配置：
``` bash
zone "liumapp.com" {
 type  master;
 notify  yes;
 also-notify {106.14.212.41;};
 allow-transfer{106.14.212.41;};
 file "liumapp.com.zone";
};
```
重启Bind之后，回到本地电脑上，继续使用命令：
```bash
dig @115.29.32.62  liumapp.com axfr

```
![](attachments/Pasted%20image%2020240114221026.png)

但是通过`106.14.212.41`是可以获取数据的：
```bash
dig @106.14.212.41  liumapp.com axfr
注：辅助服务器 `106.14.212.41` 中没有 allow-transfer 的限制，因此可以成功。
```
![](attachments/Pasted%20image%2020240114221511.png)

# 主从DNS服务器的数据同步的SOA参数
- `serial` : 序列号，即主DNS数据库的版本号。
主服务器数据库内容发生变化时，其版本号需要递增，从服务器会对比与主服务器的数据库版本号，一样的版本号就不需要更新，否则需要更新。

- `refresh` : 从服务器每多久到主服务器检查序列号的变化

- `retry` : 从服务器到服务器请求同步解析库失败时，再次发起解析请求的时间间隔，这个时间需短时刷新时间。

- `expire` : 从服务器始终联系不到主服务器时，多久之后放弃从主服务器同步数据，超过此时间后，从服务器也将停止解析。

- `negative answer` : 否定答案的缓存时长。

注意：以上选项时间单位都支持`W`,`D`,`H`,`M`,这参数定义在资源记录的文件中，位置为SOA的后面，以（）包含，其括号前后都有空格。

# 相关配置

## options中域传输的相关配置
BIND 有适当的机制来简化域传输，并限定系统传输的负载量。
**also-notify**
定义一个用于全局的域名服务器 IP 地址列表。无论何时，当一个新的域文件被调入系统，域名服务器都会向这些地址，还有这些域中的 NS 记录发送 NOTIFY 信息。

这有助于更新的域文件极快的在相关的域名服务器上收敛同步。如果一个 also-notify 列表配置在一个 zone 语句中，全局 options 中的 also-notify 语句就会在这里失效。

当一个zone-notify 语句被设定为 no，系统就不会向在全局中 also-notify 列表中的 IP 地址发送NOTIFY 消息。缺省状态为空表(没有全局通知列表)。

**max-transfer-time-in**
比设定时间更长的进入的域传输将会被终止。默认值是 120 分钟(2 小时)。

**max-transfer-idle-in**
在设定时间下没有任何进展的进入域传输将会被终止。默认为 60 分钟(1 小时)。

**max-transfer-time-out**
运行时间比设定的时间长的发出的域传输将会被终止。默认为 120 分钟(2 小时).

**max-transfer-idle-out**
在设定时间下没有任何进展的发出的域传输将会被终止。默认为 60 分钟(1 小时)。

**serial-query-rate**
辅域名服务器将会定时查询主域名服务器，来确定域的串号是否改变。每个查询将会占用一些辅域名服务器网络带宽。为限制占用的带宽，BIND9 可以限制每个查询发送的频率。serial-query-rate 的值是一个整数，就是每秒能发送的最大查询数。默认值为20。

**Serial-queries**
在 BIND8 中, serial-queries 选项设定了在任何时候允许达到的最大的并发查询数。

BIND9 不限制串号查询的数量并忽略了 serial-queries 选项。它会使用 serial-query-rate选项来限制查询的频率。

**transfer-format**
域传输可以用两种不同格式，one-answer 和 many-answer。

transfer-format 选项使用在主域名服务器上，用来确定发送哪种格式。

one-answer 在每个资源记录传输中使用一个DNS 消息。

many-answer 则将尽可能多的资源记录集中在一个消息中。many-answer 是更加有效的，但只有相对比较新的辅域名服务器才支持它，如 BIND9、BIND8.x 和打了补丁的 BIND4.9.5。默认的设置为 many-answer。使用 server 语句中的相关选项，可以替代全局选项中的 transfer-format 设置。

**transfers-in**
可以同时运行的进入的域传输的最大值。默认值为 10。增加 transfers-in 的值，可以加速辅域的收敛速度，但也可能增加本地系统的负载。

**transfers-out**
可以同时运行的发出的传输的最大值。超过限定的域传输请求将会被拒绝。默认值为10。

**transfers-per-ns**
从一台指定的远程域名服务器，同时进行的进入的域传输的最大值。默认值 2。

增加transfers-per-ns 的值，会加速辅域的收敛速度，但也可能增加远程系统的负载。使用server 语句中的 transfer 短语可以替代全局选项中的 transfers-per-ns。

**transfer-source**
transfer-source 决定在从外部域名服务器上得到域传送数据时，选哪个本地的 ip 地址使用在 IPV4 的 TCP 连接中。它可以选定 IPV4 的源地址，和可选的 UDP 端口，用于更新的查询和转发的动态更新。不过不做设置，它会缺省挑选一个系统中的地址(常常是最靠近远终端服务器的接口地址)。但这个地址必须已经配置在远终端的 allow-tranfer选项中，才能进行域传送。此语句为所有的域设定了 transfer-source，但如果 view 或 zone中也使用了 transfer-source 语句，则全局选项中的配置就在这里失效了。

**transfer-source-v6**
和 transfer-source 一样，只是域传输是通过 IPV6 执行的。

**notify-source**
notify-source 确定使用哪些本地的源地址和可选的 UDP 端口，用于发送 NOTIFY 消息。

这个地址必须在辅域名服务器的 master 域或在 allow-notify 中设置。它会为所有域设定notify-source, 但如果 view 或 zone 中也使用了 notify-source 语句，则全局选项中的配置就在这里失效了。

**notify-source-v6**
与 notify-source 类似，但应用于 ipv6 地址的 notify 报文的发送。


## options中周期性任务间隔
**heartbeat-interval**

服务器将会为所有标记dialup的域运行维护任务，无论它的间隔在何时到期。默认为60分钟，合理值不超过1天(1440 分钟)。如果设定为0,不会为这些域产生域维护。

## 其他
**Dialup**

如果是yes，那么服务器将会象在通过一条按需拨号的链路进行域传送一样，对待所有的域（按需拨号就是在服务器有流量的时候，链路才连通）。根据域类型的不同它有不同的作用，并将集中域的维护操作，这样所有有关的操作都会集中在一段很短的时间内完成，每个heartbeat-interval一次，一般是在一次调用之中完成。它也禁止一些

正常的域维护的流量。默认值是no。

dialup选项也可以定义在view和zone语句中，这样就会代替了全局设置中dialup的选项。

**如果域是一个主域，服务器就会对所有辅域发送NOTIFY请求**。这将激活辅域名服务器中的对域的序列号的检验。这样当建立一个连接时，辅域名服务器才能确认这个域的传输合法性。

如果这个域是一个辅域或是末梢域（stub zone），那么服务器将会禁止通常的“zone up to date”（refresh）请求，为了能发送NOTIFY请求，只有在heartbeat-interval过期之后才执行。

通过下列的设置，可以实现更好的控制。

1. notify只发送NOTIFY信息。
2. notify-passive发送NOTIFY信息，并禁止普通的刷新（refresh）请求。
3. refresh禁止普通的刷新处理，当heartbeat-interval过期时才发送刷新请求。
4. passive只用于关闭普通的刷新处理。


**notify**

如果是 yes（默认），当一个授权的服务器修改了一个域后，**DNS NOTIFY** 信息被发送出去。此信息将会发给列在域 NS 记录上的服务器（除了由 SOA MNAME 标示的主域名服务器）和任何列在 also-notify 选项中的服务器。

如果是 explicit，则 notify 将只发给列在 also-notify 中的服务器。

如果是 no，就不会发出任何报文。

Notify 选项也可能设定在 zone 语句中，这样它就替代了 options 中的 notify 语句。

**如果 notify 会使得辅域名服务器崩溃，就需要将此选项关闭。**

# 主从同步范例
**环境准备：**
```c
192.168.10.222 dns1.host.com 主dns服务器
192.168.10.223 dns2.host.com 辅dns服务器
```

**配置要点：**
```c
- 辅助DNS的Bind版本必须小于主DNS的软件版本。
- 主DNS named.conf里配置allow-transfer和also-notify选项
- 辅助DNS主配置文件中option段，masterfile-format text；
- 辅助DNS的配置文件里 type:slave
- 启动辅助DNS时，检查完全区域传送：dig -t axfr @192.168.10.222
- 辅助DNS不可修改主DNS配置。
```


## 配置主DNS
配置主配置文件，添加以下字段：
- `allow-transfer { 192.168.10.223; };` 
指定从服务器信息。
允许本区域传输至特定的从DNS服务器，防止未授权的区域复制。`192.168.10.223` 为从服务器IP地址。
> 注：一般在从服务器的主配置文件中，`allow-transfer { none; };`，禁止从某个从服务器向外作区域传送。


- `also-notify { 192.168.10.223; };`
主动通知从域名服务器（辅助DNS）进行更新，在主域名服务器进行更新后，而不需要在等规定的时间后才通知从域名服务器进行更新。

主配置文件中主要修改以下字段：
```json
[root@dns1 ~]# cat /etc/named.conf 
options {
        listen-on port 53 { 192.168.10.222; };
        allow-query     { any; };
        allow-transfer { 192.168.10.223; };
        also-notify { 192.168.10.223; };       
};
```

在zone引导配置中添加正解域和反解域：
```json
[root@dns1 ~]# vi /etc/named.rfc1912.zones 
vi /etc/named.rfc1912.zones
zone "host.com" IN {
        type  master;
        file  "host.com.zone";
        allow-update { none; };
};
zone "10.168.192.in-addr.arpa" IN {
        type master;
        file "10.168.192.in-addr.arpa.zone";
        allow-update { none; };
};
```

配置区域数据库文件：
```json
[root@dns1 ~]# cd /var/named/
[root@dns1 named]# cat host.com.zone 
$TTL 600        ;10 minutes
@                       IN SOA  dns.host.com. test.qq.com. (
                                2021070705 ; serial
                                10800      ; refresh (3 hours)
                                900        ; retry (15 minutes)
                                604800     ; expire (1 week)
                                86400      ; minimum (1 day)
                                )
                        NS   dns.host.com.
$ORIGIN host.com.
$TTL 60 ; 1 minute
dns1                    A    192.168.10.222
dns2                    A    192.168.10.223
test                    A    192.168.10.55
dns                     A    192.168.10.222

[root@dns1 named]# cat 10.168.192.in-addr.arpa.zone 
$TTL 600 ;10min
@       IN      SOA     dns.host.com    17614902580@163.com (
                        2021071101      ;serial number
                        10600           ;refresh 3 hours
                        900             ;retry 15 minites
                        604800          ;expire 1 week
                        86400           ;minimum 1 day
                        )
                ns      dns.host.com.
$ORIGIN 10.168.192.in-addr.arpa.
$TTL 60
222     PTR     dns1.host.com.
223     PTR     dns2.host.com.
224     PTR     dns3.host.com.
```

## 配置辅助DNS
修改主配置文件 /etc/named.conf，修改以下三个位置：
```json
[root@dns2 ~]# cat /etc/named.conf
options {
        listen-on port 53 { 192.168.10.223; };
        allow-query     { any; };
        masterfile-format text;
};
```

辅助dns检查主dns完全区域数据传送，解析列表如下:
```json
[root@dns2 slaves]#  dig -t AXFR host.com @192.168.10.222

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.5 <<>> -t AXFR host.com @192.168.10.222
;; global options: +cmd
host.com.               600     IN      SOA     dns.host.com. test.qq.com. 2021070705 10800 900 604800 86400
host.com.               600     IN      NS      dns.host.com.
dns.host.com.           60      IN      A       192.168.10.222
dns1.host.com.          60      IN      A       192.168.10.222
dns2.host.com.          60      IN      A       192.168.10.223
test.host.com.          60      IN      A       192.168.10.55
host.com.               600     IN      SOA     dns.host.com. test.qq.com. 2021070705 10800 900 604800 86400
;; Query time: 0 msec
;; SERVER: 192.168.10.222#53(192.168.10.222)
;; WHEN: Thu Jul 15 16:13:37 CST 2021
;; XFR size: 7 records (messages 1, bytes 214)
```

配置辅助dns的zone引导配置，添加正解域和反解域：
```json
[root@dns2 slaves]# cat /etc/named.rfc1912.zones

zone "host.com" IN {
        type slave ;
        masters {192.168.10.222 ;} ;
        file "slaves/host.com.zone" ;
};
zone "10.168.192.in-addr.arpa" IN {
        type slave;
        masters {192.168.10.222 ;} ;
        file "slaves/10.168.192.in-addr.arpa.zone";
};

```
**注意：我们只有在/etc/named.rfc1912.zone中添加了需要同步的域名，辅助dns才会进行同步。不添加的域名dns是不会进行同步**。

**从服务器zone的file配置**：
实际上slave是可以不要区域数据文件的，它从master上传送区域数据后会将其缓存下来，并从缓存中提供查询解析服务。**如果在slave区域内指定file指令，则表示在区域传送时还将备份一份数据到file指定的文件中**，所以该文件对named组要求有写权限。在安装bind后，在/var/named目录下自动生成了一个`/var/named/slaves`目录，其属组和权限已经设置好，正适合放置区域传送的备份文件。

重启辅DNS服务器：
```shell
# systemctl restart named
```

查看是否有区域数据库文件传输到辅助DNS slaves文件夹下:
```shell
[root@dns2 ~]# ls /var/named/slaves/
10.168.192.in-addr.arpa.zone  host.com.zone
```

在主DNS测试辅DNS是否可以解析。如果可以解析则说明主辅配同步完成：
```shell
[root@dns1 ~]# dig dns.host.com @192.168.10.223 +short
192.168.10.222
```
# 多个view的主备同步
多个view的主备同步主要是是主备之间每个view都使用共享key进行消息的签名。
## 配置范例
### master的配置
```bash
include "/opt/bind/etc/rndc.key";
include "/opt/bind/etc/views.key";
//
controls {
    inet 127.0.0.1 port 953
    allow { 127.0.0.1; } keys { "rndc-key"; };
};
//
acl test1 {
    10.201.0.0/16;
};
acl test2 {
    192.0.0.0/8;
};
acl slavedns {  
        10.144.149.61;
        127.0.0.1;
};
options {
     listen-on port 53 { any; };
     listen-on-v6  { none; };
     directory      "/opt/bind/etc/";
     dump-file      "/opt/bind/var/named/data/cache_dump.db";
     statistics-file "/opt/bind/var/named/data/named_stats.txt";
     memstatistics-file "/opt/bind/var/named/data/named_mem_stats.txt";
     zone-statistics yes;
     allow-query     { any; };
# recursion config
     recursion yes;
     max-ncache-ttl 60;
     recursive-clients 2000;
# dnssec config
     dnssec-enable yes;
     dnssec-validation yes;
     dnssec-lookaside auto;
# rrt config
     rate-limit {
        responses-per-second 20;
        qps-scale  1000;
        window 4;
        slip 2;
        ipv4-prefix-length 32;
    };
# rpz config
    response-policy {
        zone "rpz.zone"  policy given;
   };
# log query
      querylog yes;
#define version
      version "GNUer's dns 2.0";
## transfer config
      notify explicit;
      tcp-clients 2000;
      transfers-out 100;
      allow-transfer {  slavedns; 127.0.0.1;};
      also-notify { 10.144.149.61; };
     /* Path to ISC DLV key */
     #bindkeys-file "/opt/bind/etc/named.iscdlv.key";
};

logging {
  channel default_syslog { file "/opt/bind/var/log/named.syslog" versions 5 size 100m; severity dynamic; print-time yes;};
  channel default_debug { file "/opt/bind/var/log/named.run" versions 5 size 100m; severity dynamic; print-time yes;};
  channel default_stderr { stderr; severity info; };
  channel null { null; };
  channel general_debug { file "/opt/bind/var/log/named.general" versions 3 size 100m; severity dynamic; print-time yes;};
  channel database_debug { file "/opt/bind/var/log/named.database" versions 3 size 100m; severity dynamic; print-time yes;};
  channel query_log { file "/opt/bind/var/log/named.query" versions 3 size 100m; severity dynamic; print-time yes;print-severity yes; print-category yes;};
  channel resolver_log { file "/opt/bind/var/log/named.resolver" versions 3 size 100m; severity dynamic; print-time yes;};
  channel security_log { file "/opt/bind/var/log/named.security" versions 3 size 100m; severity dynamic; print-time yes;};
  channel notify_log { file "/opt/bind/var/log/named.notify" versions 3 size 100m; severity dynamic; print-time yes;};
  channel rrt_log { file "/opt/bind/var/log/named.rrt" versions 3 size 100m; severity dynamic; print-time yes;};
  channel rpz_log { file "/opt/bind/var/log/named.rpz" versions 3 size 100m; severity dynamic; print-time yes;};
  category default {null; };
  category queries { query_log; };
  category resolver { resolver_log; };
  category security { security_log; };
  category notify { notify_log; };
  category xfer-in { notify_log; };
  category xfer-out { notify_log; };
  category update { notify_log; };
  category unmatched {default_syslog; };
  category rate-limit {rrt_log;};
  category rpz {rpz_log;};
};
view "test1" {
    recursion yes;
    allow-query { any; };
    match-clients {test1; key test1;};
    allow-update { key test1; };
    server 10.144.149.61 {keys  test1;};
  //  also-notify { 10.144.149.61; };
    zone "test.org" {
        type master;
        file "master/test.org.view1";
    };      
    zone "rpz.zone" {
       type master;
       file "master/rpz.zone";
       allow-update {none;};
   };
   zone "."{
     type hint;
     file "named.root";
   };
};

view "test2" {
    recursion yes;
    allow-query { any; };
    server 10.144.149.61 {keys  test2;};
    match-clients {test2; key test2;};
    allow-update { key test2; };
   // also-notify { 10.144.149.61; };
    zone "test.org" {
        type master;
        file "master/test.org.view2";
    };      
    zone "rpz.zone" {
       type master;
       file "master/rpz.zone";
       allow-update {none;};
   };
   zone "."{
     type hint;
     file "named.root";
   };
};
view "default" {
    recursion yes;
    allow-query { any; };
    server 10.144.149.61 {keys  default;};
    match-clients {any;key default; };
    allow-update { key default; };
   // also-notify { 10.144.149.61; };
    zone "test.org" {
        type master;
        file "master/test.org.default";
    };      
    zone "rpz.zone" {
       type master;
       file "master/rpz.zone";
       allow-update {none;};
   };
   zone "."{
     type hint;
     file "named.root";
   };
};
```
master中的注意事项是：  
1. also-notify 可以不用每个view都写一遍，在options里把slave都写全也行（也得跟进实际的安全需求来）  
2. 每个view内用allow-update设置只允许响应的key进行更新。  
3. 需要使用server来指定和对端机器通信的共享密钥。

### slave的配置
```bash
include "/opt/bind/etc/rndc.key";
include "/opt/bind/etc/views.key";
//
controls {
    inet 127.0.0.1 port 953
    allow { 127.0.0.1; } keys { "rndc-key"; };
};
//
acl test1 {
    10.161.65.8;
};
acl test2 {
    192.0.0.0/8;
};

options {
     listen-on port 53 { any; };
     listen-on-v6  { none; };
     directory      "/opt/bind/etc/";
     dump-file      "/opt/bind/var/named/data/cache_dump.db";
     statistics-file "/opt/bind/var/named/data/named_stats.txt";
     memstatistics-file "/opt/bind/var/named/data/named_mem_stats.txt";
     masterfile-format text;
     zone-statistics yes;
     allow-query     { any; };
# recursion config
     recursion yes;
     max-ncache-ttl 60;
     recursive-clients 2000;
# dnssec config
     dnssec-enable yes;
     dnssec-validation yes;
     dnssec-lookaside auto;
# rrt config
     rate-limit {
        responses-per-second 20;
        qps-scale  1000;
        window 4;
        slip 2;
        ipv4-prefix-length 32;
    };
# rpz config
    response-policy {
        zone "rpz.zone"  policy given;
   };
# log query
      querylog yes;
#define version
      version "GNUer's dns 2.0";
## transfer config
      notify explicit;
      tcp-clients 2000;
      transfers-out 100;

     /* Path to ISC DLV key */
     #bindkeys-file "/opt/bind/etc/named.iscdlv.key";
};

logging {
  channel default_syslog { file "/opt/bind/var/log/named.syslog" versions 5 size 100m; severity dynamic; print-time yes;};
  channel default_debug { file "/opt/bind/var/log/named.run" versions 5 size 100m; severity dynamic; print-time yes;};
  channel default_stderr { stderr; severity info; };
  channel null { null; };
  channel general_debug { file "/opt/bind/var/log/named.general" versions 3 size 100m; severity dynamic; print-time yes;};
  channel database_debug { file "/opt/bind/var/log/named.database" versions 3 size 100m; severity dynamic; print-time yes;};
  channel query_log { file "/opt/bind/var/log/named.query" versions 3 size 100m; severity dynamic; print-time yes;print-severity yes; print-category yes;};
  channel resolver_log { file "/opt/bind/var/log/named.resolver" versions 3 size 100m; severity dynamic; print-time yes;};
  channel security_log { file "/opt/bind/var/log/named.security" versions 3 size 100m; severity dynamic; print-time yes;};
  channel notify_log { file "/opt/bind/var/log/named.notify" versions 3 size 100m; severity dynamic; print-time yes;};
  channel rrt_log { file "/opt/bind/var/log/named.rrt" versions 3 size 100m; severity dynamic; print-time yes;};
  channel rpz_log { file "/opt/bind/var/log/named.rpz" versions 3 size 100m; severity dynamic; print-time yes;};
  category default {null; };
  category queries { query_log; };
  category resolver { resolver_log; };
  category security { security_log; };
  category notify { notify_log; };
  category xfer-in { notify_log; };
  category xfer-out { notify_log; };
  category update { notify_log; };
  category unmatched {default_syslog; };
  category rate-limit {rrt_log;};
  category rpz {rpz_log;};
};
view "test1" {
    recursion yes;
    server 10.161.64.97 {keys test1; };
    allow-query { any; };
    match-clients {test1; key test1;};
    allow-update { key test1; };
    zone "test.org" {
        type slave;
        file "master/test.org.view1";
    masters { 10.161.64.97; } ;
    };      
    zone "rpz.zone" {
       type master;
       file "master/rpz.zone";
       allow-update {none;};
   };
   zone "."{
     type hint;
     file "named.root";
   };
};

view "test2" {
    recursion yes;
    allow-query { any; };
    match-clients {test2; key test2;};
    server 10.161.64.97 {keys test2; };
    allow-update { key test2; };
    zone "test.org" {
        type slave;
    file "master/test.org.view2";
    masters { 10.161.64.97; } ;
    };      
    zone "rpz.zone" {
       type master;
       file "master/rpz.zone";
       allow-update {none;};
   };
   zone "."{
     type hint;
     file "named.root";
   };
};
view "default" {
    recursion yes;
    allow-query { any; };
    server 10.161.64.97 {keys default; };
    match-clients {any;key default; };
    allow-update { key default; };
    zone "test.org" {
        type slave;
        file "master/test.org.default";
    masters { 10.161.64.97; } ;
    };      
    zone "rpz.zone" {
       type master;
       file "master/rpz.zone";
       allow-update {none;};
   };
   zone "."{
     type hint;
     file "named.root";
   };
};
```

slave的配置注意项也是每个view要使用server定义master通信时使用的key，然后限制特定的key才能更新。另外需要注意的是slave和master的IP不要在任何的acl里。

# DNS BIND主辅同步之TSIG加密
## 背景
服务器之间数据配置文件传输的安全性，比如主从服务器**同步数据**，动态域名更新，防止数据配置文件传输过程中遭到篡改。

## 介绍
`Transaction signatures`(TSIG：事务签名)通常是一种确保DNS消息安全，并提供安全的服务器与服务器之间通讯的机制。

TSIG可以保护以下类型的DNS服务器：**Zone区域传送、Notify、动态更新(nsupdate)、递归查询邮件**。

TSIG使用**共享秘密**和单向散列函数来验证DNS信息。`TSIG` 可确认 DNS 之信息是由某特定 `DNS Server` 所提供。通常`TSIG` 应用于域名服务器间的区带传输，确保数据不会被篡改或产生 `dns spoofing`。

整体逻辑：主服务器中生成公钥和秘钥，从服务器中只有提供正确的秘钥，才可以从主服务器中备份数据。

## 流程
### 生成`TSIG`
#### `dnsssec-kengen`工具
使用bind提供的工具`dnsssec-kengen`生成共享密钥。
```text
dnssec-keygen  
DNSSEC 密钥生成工具

-a 选择加密算法
    对于DNSSEC 值必须是 RSAMD5, RSASHA1(强制实现), DSA(推荐), NSEC3RSASHA1, NSEC3DSA, RSASHA256, RSASHA512, ECCGOST
    对于TSIG/TKEY, 值必须是DH (Diffie Hellman), HMAC-MD5(强制实现),HMAC-SHA1, HMAC-SHA224, HMAC-SHA256, HMAC-SHA384, HMAC-SHA512

-b 指定密钥中的位数。
    密钥大小的选择取决于使用的算法。
    RSAMD5 和 RSASHA1 密钥必须在 512 和 2048 位之间。
    Diffie-Hellman 密钥必须在 128 和 4096 位之间。
    DSA 密钥必须在 512 和 1024 位之间，并且必须是 64 的整数倍。
    HMAC-MD5 密钥必须在 1 位和 512 位之间。

-f  在 KEY/DNSKEY 记录的标志字段中设置指定的标志。唯一识别的标志是 KSK（Key Signing Key，密钥签名密钥）DNSKEY。

-h 列出 dnssec-keygen 的选项和参数的简短摘要

-n 指定密钥的所有者类型,可以选择ZONE或者HOST。
    nametype 的值必须是 ZONE（对于 DNSSEC 区域密钥 (KEY/DNSKEY)）、HOST 或 ENTITY（对于与主机相关的密钥 (KEY)）、USER（对于与用户相关的密钥 (KEY)）或 OTHER (DNSKEY).
    这些值不区分大小写。缺省值是 ZONE（用于生成 DNSKEY）

-r 指定随机源，有助与生成速度。
    如果操作系统不提供 /dev/random 或等效设备，则缺省的随机源是键盘输入.

-K（大写） <directory>: 设置要写入的密钥文件的目录
```
#### 在主DNS服务器中生成密钥
```bash
#dnssec-keygen -a HMAC-SHA512 -b 512 -n HOST -K /root/dnskey/ -r /dev/urandom hunk-tech-key

选项解读：
    -a HMAC-SHA512  :采用HMAC-SHA512加密算法
    -b 512          :生成的密钥长度为512位
    -n HOST         :指定密钥的所有者类型为主机类型
    -K /root/dnskey/:指定生成密钥的目录
    -r /dev/urandom :指定生成密钥使用的随机数来源，否则将会让你在键盘上敲入随机字符，导致会非常慢。
    hunk-tech-key   :密钥的名称
```

之后，会在指定的目录/root/dnskey/生成2个文件。
```c
Khunk-tech-key.+165+40008.key     # 公钥
Khunk-tech-key.+165+40008.private # 私钥

内容类似如下：
#cat Khunk-tech-key.+165+40008.key 
hunk-tech-key. IN KEY 512 3 165 MmQEQV+fSKe/uEKfxcpMa4avCFPTY3ipmcg+JqaPU2dV9yYx9rOdXesP aVnUyv6XarzJ3ml1H2gCgR0cDf3TGg==

#cat Khunk-tech-key.+165+40008.private 
Private-key-format: v1.3
Algorithm: 165 (HMAC_SHA512)
Key: MmQEQV+fSKe/uEKfxcpMa4avCFPTY3ipmcg+JqaPU2dV9yYx9rOdXesP aVnUyv6XarzJ3ml1H2gCgR0cDf3TGg==
Bits: AAA=
Created: 20180206083046
Publish: 20180206083046
Activate: 20180206083046

```

注：TSIG 只有一组密码，并无公开/私密金钥之分。如上，2个文件中的Key是相同的。


#### 在主DNS服务器上创建密钥验证文件
```bash
#vim /etc/named/dns-key

key "hunk-tech-key" {   > 这个双引号内填写的字符串可以是任意的。这个字符串主从必须要一致。这个例子使用dnssec-keygen生成时指定的密钥的名称
        algorithm HMAC-SHA512;   > 这个加密算法填写的是dnssec-keygen生成时指定的加密算法
        secret "MmQEQV+fSKe/uEKfxcpMa4avCFPTY3ipmcg+JqaPU2dV9yYx9rOdXesPaVnUyv6XarzJ3ml1H2gCgR0cDf3TGg==";  > 这里填写的生成密钥中K*.private文件中的key值。注意双引号和分号
};
```

修改密钥验证文件所有者与权限
```bash
#chown root:named /etc/named/dns-key
#chmod 640 /etc/named/dns-key
```

#### 修改主DNS服务器的主配置文件
```bash
# vim /etc/named.conf
include "/etc/named/dns-key"; # 加载秘钥验证文件
options {
    allow-transfer { key hunk-tech-key; };       > 定义有key的主机才能同步。
    notify yes;
    ....
}
```

或者 

```bash
# vim /etc/named.conf

include "/etc/named/dns-key"; # 加载秘钥验证文件

options {
    .....
    dnssec-enable yes;
    dnssec-validation yes;
    allow-update { localhost;192.168.7.253; };      > 定义仅有本机和从DNS才可以动态更新
    allow-transfer { localhost;192.168.7.253; };    > 定义只允许本机和从DNS主机才能使用区域传送
    notify yes;
    .....
}

以下行不在全局定义的范围内，不要误写入options的{ }中
server 192.168.7.253 { keys hunk-tech-key; };       > 定义与从dns服务器使用密钥通讯
```


#### 从DNS服务器创建密钥认证文件
方法一：从主服务器导入密钥验证文件
```bash
为了确保传输的文件没有被破坏，请使用md5sum之类的哈希算法进行校验
#md5sum /etc/named/dns-key > /etc/named/md5sum
#scp /etc/named/* 192.168.7.253:/etc/named/
```

方法二：在从DNS服务器上面创建完全相同内容的密钥认证文件


然后，修改密钥验证文件所有者与权限。
```
#chown root:named /etc/named/dns-key
#chmod 640 /etc/named/dns-key
```
#### 修改从DNS服务器的主配置文件
```bash
# vim /etc/named.conf

include "/etc/named/dns-key"; # 加载秘钥验证文件

options {
    .....
    dnssec-enable yes;
    dnssec-validation yes;
    allow-update { none; };             > 不允许客户端动态更新
    allow-transfer { localhost; };      > 定义只允许本机才能使用区域传送
    ......
}

如果只想要在某个zone中使用密钥传送，按以下写法即可
    zone "hunk.tech" {
            type slave;
            masters { 192.168.7.254 key hunk-tech-key; };
            ...
    }

如果有多个zone中需要使用密钥传送，保持zone的设置不更改，只需要定义一个全局的server配置项即可
    server 192.168.7.254 { keys hunk-tech-key; };
```

#### 在主DNS服务以及从服务器上生效配置
分别在主服务器，从服务器上执行下面的命令。
```bash
#named-checkconf
#rndc reload
```


#### 测试TSIG
在从DNS服务器
```
#dig -t axfr hunk.tech -k /etc/named/dns-key @192.168.7.254    > -k 指定密钥
```
使用专用的动态更新工具来测试
```
#nsupdate -k /etc/named/dns-key 
> server 192.168.7.254
> zone hunk.tech
> update add 9.hunk.tech 600 A 9.9.9.9
> send
> quit
```

在主DNS服务器日志中可以看到
```
client 192.168.7.254#42738: view net_192: signer "hunk-tech-key" approved
client 192.168.7.254#42738: view net_192: updating zone 'hunk.tech/IN': adding an RR at '9.hunk.tech' A
```

在从DNS服务器日志中可以看到
```
transfer of 'hunk.tech/IN/net_192' from 192.168.7.254#53: connected using 192.168.7.253#34324
zone hunk.tech/IN/net_192: transferred serial 61: TSIG 'hunk-tech-key'
transfer of 'hunk.tech/IN/net_192' from 192.168.7.254#53: Transfer completed: 1 messages, 11 records, 428 bytes, 0.006 secs (71333 bytes/sec)
```

#### 配置zone同步key
由于bind的主辅同步可以控制到具体的zone，所以TSIG可以对不同的zone，配置不同的TSIG，不过要通过view配置。

如主服务器：
```bash
view "tisg"{
    match-clients{
        key "tisg";
        192.168.36.0/24;
    };
    allow-transfer { key xxx; };
    zone "."{
        type hint;
        file "named.root";
    };
    zone "test.com"{
        type master;
        also-notify{
            192.168.36.189;
        };
        file "tisg/test.com.zone";
    };
};
```

如辅服务器：
```bash
view "tisg"{
    match-clients{
        key "tisg";
        192.168.36.0/24;
    };
    allow-transfer { key xxx; };
    zone "."{
        type hint;
        file "/var/named/named.root";
    };
    zone "test.com"{
        type slave;
        masters{
            192.168.36.54;
        };
        file "tisg/test.com.zone";
    };
};
```

# 区域传输tune调优
参考:  [zone transfer tune](https://kb.isc.org/docs/aa-00726)
**潜在问题**：
![](attachments/Pasted%20image%2020231225161101.png)
即：zone更新的延迟生效、zone更新的同时影响对于client的dns请求等。

**master 服务器调优*：
![](attachments/Pasted%20image%2020231225161905.png)
![](attachments/Pasted%20image%2020231225162526.png)

**slave服务器调优**：
![](attachments/Pasted%20image%2020231225163005.png)

## bind主备同步的问题
最近遇到几次DNS的主备同步问题。

1. 每分钟动态生成反解，然后发现有的slave服务器上不更新。  
通过日志看到每次都是最后一个slave机器收到notify消息去master上请求传输zone时有报错，提示master服务不可用。后来发现是master上的transfers-out没有单独指定，这个值默认是10，所有可能比较多slave机器请求时就失败了。把这个配置根据实际的情况调整后解决的。

2. 海量的域名同步时，slave查询soa都查不过来。
到底多大是海量？这个自己看看自己机器上的域名总数能占到中国所以域名的几个百分点以上吧。当数量大了后确实是各种问题都出来了，这个我是观察了一下有个serial-query-rate 可以设置soa查询速度的，默认是20/S，对于有海量域名的DNS来说这样显然是跟不上节奏的，直接把这个调整到5W/S，查询速度飕飕的。对于master的压力其实也还好，就当时收到那么点请求。相应地tcp-clients和transfers-in，transfers-per-ns也要做好调整。

3. SOA的TTL设置问题。
这个其实也不算是什么问题。就是nxdomain的缓存时间是有SOA TTL决定的（如果本地LDNS没有单独设置的话）。
有人先把1个域名删除了，接着又有人去解析一下这个被删除的域名，然后之前的人把域名又加上去。。结果所有人在办公网访问不了这个新增的域名。其实这个就是NXDOMAIN的缓存问题。我只有直接把SOA TTL缩短一下。
很多性能上的问题我们需要根据日志来看到底存在的瓶颈是在哪里，然后再去考虑如何优化。没有目的的优化是瞎折腾。

## bind主备同步的关键配置
DNS系统中，在大家的直观印象下bind主备的同步都是“实时”的。实际上主备同步的速度有诸多的瓶颈。

**对于master而言**：  
1. 是否有delay notify消息，这个配置是 notify-delay,默认是5s，有必要的话是需要缩短的。  
2. transfers-out 限制同时允许区传输的数量，默认是10，如果slave多，zone多需要调大。  
3. serial-query-rate 对于master而言会限制master给slave发送notify的速度，默认是20,需要调大。

**对于slave而言**：  
1. transfers-per-ns限制了从单个master同步的并发，默认也是10,需要调大。  
2. transfers-in 限制了同时从master（可能有多个）同步的总数，默认是10,需要调大。  
3. serial-query-rate 在slave中会限制slave向mastetr做SOA查询的频率。默认是20,需要调大。

对于单个同步的case，可以从master域名更新、master触发notify，slave收到notify，slave开始同步，slave完成同步几个关键的时间点，查看时间到底消耗到哪里。

## ixfr-from-differences的功效
### 背景
常规情况下bind的主备同步是自动增量同步的。但是有些场景下是全量同步，比如自己手动改的zone文件，重新加载进去。  
一般内部的反解信息是根据所有的zone自动生成的，就会存在PTR记录每次全量同步的量非常大。
### 测试
测试了可以通过打开ixfr-from-differences，在master上自动计算差异，slave就可以做增量同步了。
```bash
ixfr-from-differences yes;
```
![](attachments/Pasted%20image%2020240122165222.png)
上图中可以看到之前没有打开ixfr-from-differences时同步1.9W条记录需要2.6s，开启之后每次增量同步只需要0.02s。
开启ixfr-from-differences 时会增加master的CPU、内存开销，所以需要根据实际的情况衡量是否需要打开。
# 参考
```bash
# Tuning your BIND configuration effectively for zone transfers
https://kb.isc.org/docs/aa-00726

# bind9的配置文件中的配置解释 （++++++++++++）
https://chengqian90.com/DNS/DNS%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B9%8BBIND9.html
```