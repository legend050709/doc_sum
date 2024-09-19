```table-of-contents
```
# 前言
## zone文件基础
zone 文件用来存储zone的资源记录。在master上，zone文件一般是外部创建，为了维护的方便和灵活性，一般使用text格式。

**master的zone的维护**
zone的记录维护：可以通过nsupdate动态更新内存中的zone数据，然后回写到zone文件中；也可以通过更改zone文件，然后reload方式写入到内存中。


**slave的zone的维护**
slave中的zone数据一般来自于master的zone传输同步，然后将这些zone数据以zone文件的形式保存到磁盘中，这样slave重启时，就可以从磁盘中读取zone配置，而不是从master中接收配置。

> 注：其实slave也可以通过nsupdate进行动态更新，或者通过更改zone文件，然后reload的方式进行，不过一般不这样使用。

## zone文件格式

![](attachments/Pasted%20image%2020240422190128.png)


zone文件的格式：
- text 格式
- raw 格式
- map 格式

### text 格式
默认情况下， master 中的 zone 文件格式是 text 格式。

#### 优缺点 
**优点**
人类可读，读取方便。

**缺点**
每次存储（写文件）都需要编码（encode）；每次加载（读文件）都需要解码（decode）。
如果 zone 文件中的记录非常多（比如：上百万条）将会花费大量的时间进行编码和解码。

### raw 格式
raw format, 又称之为 "wire format"，也是 master 到 slave之间传递 的 zone的格式。
默认情况下， slave 中的 zone 文件格式是  raw 格式。

#### 优缺点
**缺点**
人类不可取。
需要特别的工具（ `named-compilezone` ）来读取文件内容。
```bash
# 输出到标准输出
named-compilezone -f raw -o -  zone_name  zone文件

# 输出到指定文件(/tmp/zone.out)
named-compilezone -f raw -o -  zone_name  zone文件

比如：
# named-compilezone  -o  /tmp/aa.out  longshuai.com  /var/named/db.longshuai.com  
```

### map格式

BIND 9.10 及更高版本支持zone文件的map格式。这种格式编解码速度更快。它可以在主服务器或辅助服务器上使用，但在主服务器上使用文本格式是最简单的。

#### 介绍
当BIND运行时，它会将区域信息保留在计算机的内存中。解码区域文件意味着从磁盘读取文件，找到相关信息，并将这些信息存储在内存中。

如果使用一种称为内存映射（memory mapping）的技术，**将区域信息数据（zone data）按其在内存中的相同格式写回磁盘，那么在使用时就不需要对其进行解码**。
BIND可以直接链接到这个文件，就像它在内存中一样，操作系统会很快确保文件被加载到内存中。

####  配置
```bash
masterfile-format map;

比如：
zone third.example.com {
         type secondary;
         file "sec/third.example.com";
         masterfile-format map;
         primaries {192.168.1.100;};
    };
```

#### 优缺点
（1）优点
映射格式（map format）比原始格式(raw format)快得多，因为区域文件在使用前不需要进行处理，文件中的区域数据直接可用。

它到底有多快取决于主机操作系统的许多细节和硬件性能。
访问映射格式的区域文件的速度依赖于硬盘速度、磁盘缓存的大小和速度、操作系统内存管理算法、硬件内存总线速度以及计算机主板上的其他活动。ISC已经观察到映射格式相对于原始格式的速度提升了50倍，但也观察到远低于这个速度提升的情况。

（2）缺点
映射格式不适用于不同系统之间或不同版本的BIND之间。

**映射格式的区域文件与其创建时的计算机、操作系统和BIND版本紧密相关**。
- 你不应该尝试在一个计算机上创建映射格式的区域文件然后在另一个计算机上使用。
- 如果你正在升级BIND或升级支持BIND的操作系统，你应该删除所有映射格式的区域文件，并在BIND启动后通过区域传输让它们重新创建。


#### zone文件是否使用map格式

![](attachments/Pasted%20image%2020240422222950.png)

**在slave上使用map格式的zone文件**：

在slave上使用map格式的zone文件风险比较小。slave的内存中的zone信息是通过zone 传输获取的，然后通过named 写入到zone文件，最后可能在重启时再从 zone 文件中读取。

在正常运行中，通常不需要人工检查存储在slave上的区域zone文件，但如果确实需要检查，可以使用`named-compilezone`程序对map格式的区域文件进行编码和解码。
> 注意：如果升级了BIND、操作系统或更换了服务器硬件，你应该删除区域文件，并让它们作为BIND正常主次同步的一部分重新创建。

**在master上使用map格式的zone文件**：

你可以在master上使用映射格式，这将使named的启动或重启更快（map格式主要用于文件读取的解析）。
但是，如果这样做，你必须有办法在BIND之外创建和编辑这些map格式的zone文件。
你可以创建一个空的映射格式区域文件，然后通过动态更新来管理它，或者使用转换工具在文本和映射格式之间进行转换。
软件工具`named-compilezone`可以在映射格式(map format)和文本格式(text format)之间进行转换。

在master上使用map格式的zone文件否过于麻烦，你需要自己权衡。如果你选择这样做，必须确保使用与你的BIND(`named`)版本配套的`named-compilezone`版本，它们必须完全匹配。

### 小结

可以在master上面使用 text 格式的 zone文件，在slave中使用 map格式的zone文件；map格式的zone文件的读取速度比 raw格式以及text格式更快。

另外，注意 slave的 named 版本升级时候 删除旧的map格式zone文件，创建新的 map格式 zone文件（通过zone 传输）。

# zone传输的定义
所谓区域传输(zone transfer)，就是**内存中的域名记录**发生了变更，辅助域名服务器与主域名服务器通信，并同步 RR 资源的过程。这样做的目的是为了保证多台服务器保证内容同步。

**如何定义变更**
即：SOA记录中的 serial number 发生了递增的变更，则说明zone发生了变更。

# 特性

## TCP协议进行zone transfer

**区域传输使用 TCP 而不是 UDP**

如果使用UDP，限制传输数据 512B以内（DNS的UDP响应就是限制在512B以内）。由于数据同步传送的数据量比一个 DNS 请求和响应报文的数据量要多得多，因此使用TCP。

![](attachments/Pasted%20image%2020240104133505.png)

因此：
**UDP 用于 client 和 server 的查询和响应**；
```bash
man dig
       +tcp, +notcp
        This option indicates whether to use TCP when querying name servers.  The default behavior is to use UDP unless a type any or ixfr=N query is requested, in which case the default is TCP. AXFR queries always use TCP.

如上所示：并不是所有的dns请求都是 使用 udp；
如果 dig 查询的type = any 或者  ixfr=N or  AXFR 的查询，是使用 TCP的。
```

**TCP 用于主从 server 之间的Zone传送**。
> 注： notify的请求响应，soa的请求响应还是UDP；只是 zone transfer 消息（XFER 消息）是通过tcp的。




## master自动同步变更给slave
RFC 标准协议通过 MASTER-SLAVE 架构，**NOTIFY请求响应（UDP） + SOA请求响应（UDP） + XFR请求响应（TCP）** 机制实现数据自动同步，用户只需要在主服务器上更改域名，更改信息便可自动同步到从服务器 。

![](attachments/Pasted%20image%2020240104162335.png)

## slave发起zone传输
zone transfer的请求发起者，永远是 slave服务器。
slave服务器发现自身zone的 serial number 小于 master 的zone的 serial number，就会发起 zone transfer（XFER）的请求。

# zone transfer 

## 同步时机
![](attachments/Pasted%20image%2020240408103444.png)

同步时机有：
1. 服务（named）重启；
2. rndc强制transfer zone
3. slave的zone的refresh time到期
4. slave收到notfiy消息

### 时机一：启动同步
（1）slave 重启：

当slave的named启动时，slave主动去master上获取区域数据。
(slave重启，此时slave将zone的refresh time设置为now，就可以发起SOA请求，检查serial number；不一致，则会进行XFR请求。)

（2）master重启：

master的named重启后，master会发送notify消息给slave。但要求区域数据文件中定义了slave的ns记录及其A记录，否则找不到slave，也就联系不上slave。
（master重启，则master发送notfiy消息给slave。）

### 时机二：定时检查

#### 主服务器的定时notify

如果主DNS服务配置了`heartbeat-interval`为5；
则 master每5分钟会给全部slave发送notify消息。并且是给每个view的每个zone发送一条notify消息，无论该zone是否存在记录变更。没有做打散处理。发送notify消息的时候，会影响master的性能。影响增量配置变更生效的速度。

> 注：可以关闭 master的 定时全量更新通知。将 master 的`heartbeat-interval` 设置为0 即可。

#### 从服务器的定时检查

master 服务器会定时(SOA中的 refresh time)向主域名服务器进行查询（SOA请求）以便了解区域是否有变动(serial number 是否增加)。如有增加，则会执行一次区域传输请求（XFER消息）。

一旦启动区域传输，就会存在两种传输方式：
1. 全量传输(AXFR)：即传输整个区域的消息，全量传输会传输整个区域（zone）的消息。
2. 增量传输(IXFR)：增量传输就是传输一部分消息，增量传输使用的消息。

注：**slave收到notify，其实就是相当于将 refresh time 设置为 now，所以slave会立刻向 master 发送SOA请求(zone refresh)**。


### 时机三：DNS NOTIFY

但是使用轮询（refresh timer）这种方式有一些弊端，因为从服务器会定期检查主服务器上内容是否更新，这是一种资源浪费，因为绝大多数情况下都是一次无效检查，所以为了改善这种情况，DNS 设计了 `DNS NOTIFY` 机制，`DNS NOTIFY` 允许修改区域内容后主服务器通知从服务器内容需要更新，应该启动区域传输。


#### 变更的同步过程

![](attachments/Pasted%20image%2020240826201532.png)
```bash
10.16.128.209: master ip;
10.16.131.18: slave ip;
```

主从权威的同步过程如下：
（1）主节点 Zone 配置变更，向从节点发送 NOTIFY 通知（基于UDP）
（2）从节点返回 NOTIFY Respons
（3）slave向master发起 SOA 查询（基于UDP），主节点返回 SOA Respons
（4）从节点对比 SOA Respons 中的序列号是否比自身序列号大，仅当 SOA Respons 序列号大于自身序列号时才发起 `Zone transfer request(IXFR)`，并利用 **TCP 53 端口**进行数据传输。
（5）主节点收到 `Zone transfer request(IXFR)`，进行响应。

因此 Zone 配置变更后必须增大序列号，否则会导致主从节点数据不一致。


> 注： notify 消息中，可能带有 serial numer 的SOA信息，也可能没有。
如果 notify 中 没有 serial number 消息，则会有 日志：
```bash
general: info: zone 23.172.in-addr.arpa/IN/base: notify from 10.108.164.23#40210: no serial
```

#### 范例
常规情况下，zone每发生一次变化，序列号加1，通过序列号标识版本，获取增量更新信息。

![](attachments/Pasted%20image%2020240115203238.png)

如上，忽略tcp的建连过程，
1，2号报文为notify的通告和响应；通告即zone发生变化，master通告slave。

3，4号报文为SOA的查询和响应；slave发起，请求master最新的序列号；

10，12号报文为ixfr更新请求和响应；slave发起，请求更新zone信息（axfr同理，也是这样）；


### 时机四：rndc强制重传
```bash
rdnc 
	retransfer zone [class [view]]
		Retransfer a single zone without checking serial number.
```
![](attachments/Pasted%20image%2020240408101945.png)

## 相关问题

### master`发送`notify`消息的作用
这将激活辅域名服务器中的zone的**`next refresh` time to now**，然后检查序列号是否变化。（即：slave向master发送 SOA请求，从SOA响应中获取 serial num ）。
因此，其实监控到了`master`和`slave`的`zone`的配置不一致，其实也可以其他的服务器，甚至`slave`自身，给`slave`发送一条`notify`消息，这样也可以触发`slave`服务器对向master发送 SOA请求获取 serial number 。

### 哪些服务器可以给slave发送notify消息

1. zone 的  NS记录中的服务器：
2. allow-notify 配置的服务器：一般在从服务器中配置该项。
3. primary list 中的服务器：一般在从服务器中的 zone的 masters中设置。
```bash
A notify is deemed **valid** if the sender is one of the servers in the NS RRset for the zone, has been explicitly allowed using an "allow-notify" clause, or is from an address listed in the primary servers clause.
```

### 从服务器上的zone文件查看

辅助DNS辅助器生成的区域文件，Centos 6 可以使用`cat`等文本工具查看；
Centos 7 已经使用 raw 格式存放，需要使用这个命令配合参数查看:
**named-compilezone**：对区域数据文件进行编译，并输出编译后的结果。
```bash
# 输出到标准输出
named-compilezone -f raw -o -  zone_name  zone文件

# 输出到指定文件(/tmp/zone.out)
named-compilezone -f raw -o -  zone_name  zone文件
```

范例如下所示：
```bash
# named-compilezone  -o  /tmp/aa.out  longshuai.com  /var/named/db.longshuai.com   
zone longshuai.com/IN: loaded serial 1
dump zone to /tmp/aa.out...done
OK

# cat /tmp/aa.out
longshuai.com.            21600 IN SOA    dnsserver.longshuai.com. xyz.longshuai.com. 1 10800 3600 604800 3600  
longshuai.com.            21600 IN NS     dnsserver.longshuai.com.  
dnsserver.longshuai.com.  21600 IN A      172.16.10.15  
ftp.longshuai.com.        21600 IN A      172.16.10.17  
mydb.longshuai.com.       21600 IN A      172.16.10.18  
www.longshuai.com.        21600 IN A      172.16.10.16  
www1.longshuai.com.       21600 IN CNAME  www.longshuai.com.  
```

### 其他问题

![](attachments/Pasted%20image%2020240409173946.png)

**(1)slave正在进行zone的refresh或排队(queue)进行refresh，此时收到notify，怎么办?**
正在进行（in progress）zone refresh：因为zone refresh(soa的发送和接收)是2个包，可能刚发送了soa请求，还没有收到回复。

排队(queue)进行zone refresh：有多个zone进行refresh(发送soa)，正在发送了某个zone的请求，那么后续zone的refresh就需要进行排队(queue)。

**(2) 为什么有时候需要 对zone refresh进行排队(queue)?**

由于资源限制，named 可能 正在处理一个zone refresh。另外的 zone refresh 则需要排队。

**(3) 收到master的notify之后为什么不是立刻对这个master发送zone refresh?**

存在多个master的时候，收到 notify，不太可能立刻进行 zone refresh，而是需要选择 master来发送 soa请求。
另外一个就是，如果一个zone的2个记录更新，可能会发送2个notify，那么延迟发送soa请求，就可以少发 一个soa请求。



# zone文件更新
## 静态更新
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

## nsupdate进行动态更新

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
使用 `nsupdate` 等工具进行动态配置。​ 使用`nsupdate` 不会更改区域数据库文件，而是产生了一个`jnl`的数据文件，**不能使用文本编辑器打开，只能使用完全区域数据传送（AXFR 请求以及响应）查看**。

注：`jnl`文件（`journal`文件）是`BIND9`动态更新的时候记录更新内容所生成的日志文件。

#### 优缺点
- 优点
	- 命令简单，便于记忆
	- 不用手动变更SOA的serial序列号，自动滚动
	- 不需要重启/重载BIND9服务/配置，生效快
	- 可以通过配置acl实现远程管理


- 缺点
	- jnl文件无法使用文本文件的方式打开
	- 只能依赖完全区域传送查看所有区域的记录（ AXFR 请求以及响应）
	- 更新操作复杂，先删再增
	- 远程管理有安全隐患，需要加强审计
	- 动态域在rndc管理上多一步

#### 查看 slave上的zone配置
对于 jnl文件无法 查看，只能看 AXFR 的问题。

```bash
比如：
在 slave中对于指定的 zone 进行axfr的查询
dig -t AXFR xxxx @127.0.0.1

比如：
dig -t axfr internal @127.0.0.1

or

dig -t axfr hunk.tech -k /etc/named/dns-key @192.168.7.254    
-k 指定密钥文件

or
dig -t axfr hunk.tech -y [hmac:]keyname:secret @192.168.7.254
-y：指定密钥
比如：
dig  @127.0.0.1 internal SOA +norecurse +noadflag +aaflag +opcode=notify  -y hmac-sha256:kwai_default_key:"U2LTw11jcgl5Lc2pm3/P8GVHV10DTz/1fc1yrXdAVcA="

前提条件：
即在 internal 的 zone 中配置了 allow-transfer 

zone "internal" {
    type slave;
	...
    allow-transfer { 127.0.0.1; };
    ....
};

注意：
	如果没有上面的 zone中的配置 allow-transfer { 127.0.0.1; }; 
则执行 dig -t axfr internal @127.0.0.1 会出现 refused。
将其配置在 option中，而不是zone中， 也会出现 refused。

```


#### 使用方法

```shell
#发送请求到servername服务器的port端口.
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



#### 范例

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
- master发送`notify`信息给`slave`，slave给出 响应。
- `slave`去查询主服务器的`SOA`记录，master给出SOA记录响应。
- `slave`根据SOA记录去检查`serial number`是否有递增更新
	- 如果有的话，`slave`向`master`发起`zone transfer`请求，然后`master`返回响应结果，`slave`更新记录。
	-  如果没有的话就说明不需要更新。

## 工作流程
notify是这样工作的：

**当主DNS服务器重启了DNS服务或者通过NSUPDATE动态修改了域名解析记录时，则通知所有slave DNS服务器来更新区域数据。**(有些地方模糊地说SOA序列号发生改变也会发送notify，其实不然，因为区域数据文件需要编译加载到内存，简单的修改Zone数据库文件，没有加载到内存中，则是无效的）

![](attachments/Pasted%20image%2020240104162652.png)

（1）用户在 MASTER 上动态修改域名解析记录（如 NSUPDATE），修改成功后，域名所在 ZONE 的版本号加 1。

如下所示，`test.com`初始配置：

![](attachments/Pasted%20image%2020240104162738.png)

初始 SOA 序列号：

![](attachments/Pasted%20image%2020240104162805.png)

NSUPDTA 新增记录：

![](attachments/Pasted%20image%2020240104162815.png)

最新 SOA 序列号：

![](attachments/Pasted%20image%2020240104162831.png)

（2）MASTER 向其配置的 SLAVE 节点发送 NOTIFY（一般是 UDP 报文）

（3）SLAVE 在收到 NOTIFY 消息后，进行以下操作：

- SLAVE 在收到 NOTIFY 消息后会给 MASTER 发送一个响应表示收到了 NOTIFY;
- SLAVE 给Master 发送 SOA请求，并获取到SOA响应。
- SLAVE 比较 SOA响应 中的 ZONE 的版本号（serial number）和本地的 ZONE 的版本号。
	- 如果本地的版本号不低于 SOA响应 中的版本号，SLAVE 不做任何操作;
	- 如果 SLAVE 本地的版本号低于 SOA响应 中的版本号，表示本地的 ZONE 数据已经落后。SLAVE 向 MASTER 发送 IXFR 请求。
		- 如果没有获取到IXFR响应，则SLAVE 根据 REFRESH（定义在 ZONE 的 SOA 记录中）定时向 MASTER 发送 IXFR 请求，作为当 IXFR 响应的报文因为某些原因无法发送到 SLAVE 时的一种补偿机制。
		- 如果 IXFR 失败，会转向 AXFR;

（4）MASTER 根据 SLAVE 请求的 XFR 类型返回对应的数据
- IXFR 返回格式和结果：
![](attachments/Pasted%20image%2020240104163411.png)
![](attachments/Pasted%20image%2020240104163415.png)

- AXFR 返回结果（即 zone下的所有RR记录）：
![](attachments/Pasted%20image%2020240104163511.png)


## 报文解析
**整体流程**
![](attachments/Pasted%20image%2020240411104357.png)

如上所示，包含了基于UDP的notify请求以及响应，基于UDP的SOA的请求以及响应，基于TCP的IXFR的请求以及响应。
```bash
10.108.164.25: slave
10.108.164.23: master
```

**notify请求报文（基于UDP）**

![](attachments/Pasted%20image%2020240411104745.png)

即：某个view(bjx)的zone(internal)中的RR记录发生了改变，则master给slave发送notify请求。

**notify响应报文（基于UDP）**
![](attachments/Pasted%20image%2020240411104933.png)
即：slave收到notify，给master 回复表示我收到了notify。

**soa请求报文（基于UDP）**
![](attachments/Pasted%20image%2020240411105129.png)
即： salve收到notify，并响应之后。给master发送soa请求，为了获取master中对应zone的serial number，查看zone是否发生了改变。

**soa响应报文（基于UDP）**
![](attachments/Pasted%20image%2020240411105413.png)
即： master收到soa请求之后的soa响应。此中master中的zone的serial number 为 1705367117.

**ixfr请求报文（基于TCP）**
![](attachments/Pasted%20image%2020240411105714.png)

即：slave发现master的zone的serial number 比自身的大，slave给master发送IXFR的请求，附上自身的zone的serial number 为 1705367116.


**ixfr响应报文（基于TCP）**
![](attachments/Pasted%20image%2020240411110009.png)


## AXFR
### 定义
![](attachments/Pasted%20image%2020231225111500.png)

**全区域传输（AXFR：Full Zone Transfer）**：

主 DNS 服务器通知辅助 DNS 服务器已对**特定区域**进行了更改，辅助 DNS 与主 DNS 联系以检查发生更改的区域的 SOA 记录中的序列号。如果主 DNS 上的序列号大于该区域的辅助 DNS 服务器的序列号，则默认情况下，整个区域文件（AXFR）将从主 DNS 服务器复制到辅助 DNS 服务器。

> 注：AXFR传输指的是对某个Zone的全部RR记录进行传输，而不是所有的Zone的所有记录进行传输。
> 另外， AXFR以及 IXFR的 zone transfer 都是使用TCP协议进行传输

### 报文介绍
**axfr query**

![](attachments/Pasted%20image%2020240115203521.png)

如上所示，query type 为 AXFR；

**axfr response**

![](attachments/Pasted%20image%2020240115204308.png)
对于response，axfr结果放在`answers section`内，开头和结尾的SOA记录表示左括号和右括号，当中的内容表示这个zone的所有记录。

## IXFR
### 定义

![](attachments/Pasted%20image%2020231225112241.png)

增量区域传输（IXFR：Incremental Zone Transfer）：
主 DNS 服务器通知辅助 DNS 服务器已对**特定区域**进行了更改，辅助 DNS 与主 DNS 联系以检查发生更改的区域的 SOA 记录中的序列号。如果主 DNS 上的序列号大于该区域的辅助 DNS 服务器的序列号，则辅助 DNS 服务器会将上次更改与现有版本进行比较，并仅从主 DNS 复制更改的记录。

### 报文介绍
**ixfr query**
![](attachments/Pasted%20image%2020240115204422.png)

如上所示，query type 为 IXFR；

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


## 其他

### 区分主从服务器

**它是这样判断哪些是主DNS服务器哪些是从DNS服务器的？**

找到区域数据文件（zone文件）中的所有NS记录，并排除本机以及SOA记录中MNAME的那个服务器(SOA记录的MNAME列即IN关键字后的一列，一般就是主DNS服务器)，剩余的都是从DNS服务器。

主DNS为每个自定义的区都发送notify声明给所有从服务器，告知其哪些区改变了；
slave服务器接收到notify声明后响应主DNS服务器告知它已经收到了通知；
然后slave服务器向主DNS服务器发起SOA查询，以确定notify声明中所通告的区的SOA记录的serial number 是否真的发生了改变；如果SOA发生了改变，则进行区域传送的请求，如果没有改变则不进行传送。

### `notify`声明后的SOA记录请求原因

**为什么从DNS服务器接收到`notify`声明后还要再次查询主服务器上SOA记录来确认呢？**
第一是因为要比较序列号，决定是否要传送，以及要完全传送还是增量区域传送；

第二是因为有些人可能会发送假冒的notify声明给从DNS服务器，从而导致多余的区域传送。

# AXFR传输的安全限制
## 背景
我们在本地一台电脑上使用一个命令：
```bash
dig @115.29.32.62 liumapp.com axfr
```
不出意外，应该能够得到`liumapp.com`在`115.29.32.62`这台`DNS server`上的所有解析记录。
![](attachments/Pasted%20image%2020240114220534.png)
但是从安全角度来讲，我肯定不希望这样的事情发生，所以就要用到传输限制。

## `allow-transfer`限制措施

![](attachments/Pasted%20image%2020240410143804.png)

默认情况下，`allow-transfer`的值为any，表示允许任何人都可以给该主机发送区域传送请求。`allow-transfer` 可以配置在 option、view，以及zone的引导配置中。

实际上，**应该设置主dns服务器只允许slave服务器来区域传送，并设置slave服务器不允许任何人区域传送**，这样就最大程度保证了区域数据不泄漏。

**注：此中的  `allow-transfer` 指的是发给本机的 区域传输(zone transfer)的请求。默认情况下，只有 master 会接收到 slave的  AXFR请求。slave不会收到任何人的   AXFR请求**。

### 基于主机的访问控制

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


### 事务签名

通过密钥对数据进行加密。  比如`TSIG`（对称方式）或 `SIGO`（非对称方式）。
```bash
allow-transfer : {key keyfile} 
	(key及key的文件位置); 事务签名的key
```

### 测试
可以手动使用dig命令强制区域传送，只需使用-t指定区域传送的类型即可，如下：

```bash
dig -t AXFR  xxxx @IP-Address

dig -t ixfr=N
在指定增量区域传送时，需要指定序列号，只有比N大的序列号才会传送。
```

### 范例

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
## serial number
- `serial` : 序列号，即主DNS数据库的版本号。
主服务器数据库内容发生变化时，其版本号需要递增，从服务器会对比与主服务器的数据库版本号，一样的版本号就不需要更新，否则需要更新。

## refresh
- `refresh` : 从服务器每多久到主服务器检查序列号的变化(发送SOA请求)

## retry
- `retry` : 从服务器到服务器请求同步解析库失败时，再次发起解析请求的时间间隔，这个时间需短时刷新时间。

## expire

- `expire` : 从服务器始终联系不到主服务器时，多久之后放弃从主服务器同步数据，超过此时间后，从服务器也将停止解析。

## negative answer

- `negative answer` : 否定答案的缓存时长。


# 同步的相关参数配置

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
辅域名服务器将会定时查询主域名服务器，来确定域的串号(serial number)是否改变。每个查询将会占用一些辅域名服务器网络带宽。为限制占用的带宽，BIND9 可以限制每个查询发送的频率。serial-query-rate 的值是一个整数，就是每秒能发送的最大查询数。默认值为20。

**transfer-format**
域传输可以用两种不同格式，one-answer 和 many-answer。

transfer-format 选项使用在主域名服务器上，用来确定发送哪种格式。

one-answer 在每个资源记录传输中使用一个DNS 消息。

many-answer 则将尽可能多的资源记录集中在一个消息中。many-answer 是更加有效的，但只有相对比较新的辅域名服务器才支持它，如 BIND9、BIND8.x 和打了补丁的 BIND4.9.5。默认的设置为 many-answer。使用 server 语句中的相关选项，可以替代全局选项中的 transfer-format 设置。

**transfers-in**
slave上配置，收到的 zone-transfer (xfer-in) 的 并发数的最大值。
可以同时运行的进入的域传输的最大值。默认值为 10。增加 transfers-in 的值，可以加速辅域的收敛速度，但也可能增加本地系统的负载。

**transfers-out**
可以同时运行的zone transfer 发出的最大值。超过限定的域传输请求将会被拒绝。默认值为10。
![](attachments/Pasted%20image%2020240408163737.png)

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

### Dialup

![](attachments/Pasted%20image%2020240410120403.png)

> 注：默认值为no，后续该配置将会被移除。

如果是yes，那么服务器将会象在通过一条按需拨号的链路进行域传送一样，对待所有的域（按需拨号就是在服务器有流量的时候，链路才连通）。

根据域类型的不同它有不同的作用，并将集中域的维护操作，这样所有有关的操作都会集中在一段很短的时间内完成，每个heartbeat-interval一次，一般是在一次调用之中完成。它也禁止一些正常的域维护的流量。

dialup选项也可以定义在view和zone语句中，这样就会代替了全局设置中dialup的选项。


通过下列的设置，可以实现更好的控制。
1. notify只发送NOTIFY信息。
2. notify-passive发送NOTIFY信息，并禁止普通的刷新（refresh）请求。
3. refresh禁止普通的刷新处理，当heartbeat-interval过期时才发送刷新请求。
4. passive只用于关闭普通的刷新处理。



### heartbeat-interval

服务器将会为所有标记dialup的域运行维护任务，无论它的间隔在何时到期。默认为60分钟，合理值不超过1天(1440 分钟)。如果设定为0,不会为这些域产生域维护。

![](attachments/Pasted%20image%2020240410113620.png)

注：该参数后续将会被移除。


### notify

![](attachments/Pasted%20image%2020240410121440.png)

默认值为yes。Notify 选项也可能设定在 zone 语句中，这样它就替代了 options 中的 notify 语句。

**如果是 yes（默认）**
当一个授权的服务器修改了一个域后，**DNS NOTIFY** 信息被发送出去。此信息将会发给列在域 NS 记录上的服务器（除了由 SOA MNAME 标示的主域名服务器）和任何列在 also-notify 选项中的服务器。

**如果是 explicit**：
notify 将只发给列在 also-notify 中的服务器。

**如果是 no**：
就不会发出任何报文。

#### 范例

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
但是可以使用`allow-notify`字句定义可以接受其他从服务器的notify信息。例如a是b和c的主，如果在c上定义`allow-notify { b_IP; };`，那么它也会接受b的notify信息。


# 主从同步范例
**环境准备：**
```c
192.168.10.222 dns1.host.com 主dns服务器
192.168.10.223 dns2.host.com 辅dns服务器
```

**配置要点：**
- 辅助DNS的Bind版本必须小于主DNS的软件版本。
- 主DNS named.conf里配置allow-transfer和also-notify选项
- 辅助DNS主配置文件中option段，masterfile-format text；
- 辅助DNS的配置文件里 type:slave
- 启动辅助DNS时，检查完全区域传送：dig -t axfr @192.168.10.222
- 辅助DNS不可修改主DNS配置。

## 配置主DNS
配置主配置文件，添加以下字段：

- **allow-transfer { 192.168.10.223; };**

指定从服务器信息。
允许本区域传输至特定的从DNS服务器，防止未授权的区域复制。`192.168.10.223` 为从服务器IP地址。
> 注：一般在从服务器的主配置文件中，`allow-transfer { none; };`，禁止从某个从服务器向外作区域传送。


- **also-notify { 192.168.10.223; };**

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
[root@dns1 ~]# dig dns.host.com @192.168.10.223 +short +timeout=1
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
2. 每个view内用 allow-update 设置只允许响应的key进行更新。  
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
服务器之间数据配置文件传输的安全性，比如主从服务器**同步数据（zone transfer）**，动态域名更新（nsupdate），防止数据配置文件传输过程中遭到篡改。

## 介绍
`Transaction signatures`(**TSIG：事务签名**) 通常是一种确保DNS消息安全，并提供安全的服务器与服务器之间通讯的机制。

TSIG可以保护以下类型的DNS服务器：**Zone区域传送(zone transfer: ixfr/axfr)、Notify、动态更新(nsupdate)、递归查询邮件**。

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

注：**TSIG 只有一组密码，并无公开/私密金钥之分**。
如上，2个文件中的Key是相同的。


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


## 测试TSIG
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


## 配置zone同步key
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

# zone传输带来的潜在问题
## 潜在问题总结
![](attachments/Pasted%20image%2020231225161101.png)
如果存在大量的zone transfer，则可能导致其他的潜在问题：
1. 当前海量zone的更新在slave生效慢，导致slave继续使用旧数据对外服务。
2. 当前海量zone的传输，影响后续增量的zone的更新；
3. zone更新的同时影响对于client的dns请求；
4. zone的 refresh 或者 transfer 失败，导致master和slave的zone配置不一致。


## zone transfer的异常日志
日志传输相关的日志类别主要有：
**general**、**xfer-in** 、 **xfer-out**、**client**、**notify**等。

```bash
Some messages are logged from category **general** , and others from categories **xfer-in** and **xfer-out** .  On the server providing the zone transfers you might also see some relevant messages logged from category **client.**
```

注：**xfer-in** 代表收到xfer，一般代表的是 slave 机器；
**xfer-out** 代表发送xfer，一般代表的是 master 机器；

### slave中的异常日志
#### connect time out 日志

![](attachments/Pasted%20image%2020240408140632.png)

**日志**
```bash
16-Nov-2011 16:31:07.044 xfer-in: error: transfer of 'testzone.example.com/IN' from 192.0.2.1#53: failed to connect: timed out
```

**原因**
(1) slave的tcp连接达到阈值。
> 使用tcp连接存在2种情况：
> - client使用tcp进行dns请求
> - master和slave之间的 zone transfer(xfer)

(2) slave和master之间存在网络问题
比如：路由问题，或者存在防火墙。


**影响**
（1）zone refresh失败，直到下次 retry 到期 或 收到 master的 notify 才会再次尝试 zone refresh。

（2）master 被标记为 `unreachable`, 10min之后或者再次从master 收到 notify，否则不会再次进行 refresh 。
> 收到master的notify消息，就可以将master 的 `unreachable` 标记去除。

**master 标记为 unreachable 的场景**
![](attachments/Pasted%20image%2020240408142952.png)

即 TCP 连接失败，才会导致master 被标记为 unreachable。存在两种情况：
(1) SOA的udp请求失败，转换为TCP SOA请求，直到最后一次TCP SOA请求还是失败，
(2) UDP的SOA请求失败，TCP的 Zone transfer 失败。


#### connection reset日志
**日志**
日志一：
```bash
17-Nov-2011 21:50:14.762 xfer-in: info: transfer of 'testzone.com/IN' from 192.0.2.1#53: connected using 192.0.2.4#47296
17-Nov-2011 21:50:14.762 xfer-in: error: transfer of 'testzone.com/IN' from 192.0.2.1#53: failed while receiving responses: connection reset
17-Nov-2011 21:50:14.762 xfer-in: info: transfer of 'testzone.com/IN' from 192.0.2.1#53: Transfer completed: 0 messages, 0 records, 0 bytes, 0.001 secs (0 bytes/sec)
```

日志二：
```bash
17-Nov-2011 21:50:15.712 xfer-in: error: transfer of 'testzone.com/IN' from 192.0.2.1#53: failed sending request length prefix: connection reset
```

日志三：
```bash
17-Nov-2011 21:50:28.394 xfer-in: error: transfer of 'testzone.com/IN' from 192.0.2.1#53: failed sending request data: connection reset
```


日志四：
```bash
13-Jan-2012 18:45:51.036 client: warning: client 192.0.2.4#42229: no more TCP clients: quota reached.
```

**原因**
```bash
(1) tcp-listen-queue
server的半连接/全连接队列小了。

(2) tcp-clients：tcp 连接个数达到设置的阈值，则出现 no more TCP clients: quota reached。
使用tcp连接的情况：
1》client的tcp的dns请求
2》zone transfer
```

#### `refresh: retry limit for master  x.x.x.x#53 exceeded`

**范例日志**
如下所示：
```text
26-Aug-2024 17:30:11.454 notify: info: client @0x7f2d4070b390 10.16.128.209#10510/key kwai_base_key: view base: received notify for zone '94.10.in-addr.arpa': TSIG 'kwai_base_key'
26-Aug-2024 17:30:11.454 general: info: zone 94.10.in-addr.arpa/IN/base: notify from 10.16.128.209#10510: serial 1720604391
26-Aug-2024 17:30:11.461 notify: info: client @0x7f2d4070b390 10.16.128.209#10510/key kwai_base_key: view base: received notify for zone 'kwaidc.com': TSIG 'kwai_base_key'
26-Aug-2024 17:30:11.461 general: info: zone kwaidc.com/IN/base: notify from 10.16.128.209#10510: serial 1720608895
26-Aug-2024 17:30:11.864 general: info: zone 4.10.in-addr.arpa/IN/base: refresh: retry limit for master 10.16.128.209#53 exceeded (source 0.0.0.0#0)
26-Aug-2024 17:30:11.864 general: info: zone 4.10.in-addr.arpa/IN/base: Transfer started.
26-Aug-2024 17:30:11.864 xfer-in: info: transfer of '4.10.in-addr.arpa/IN/base' from 10.16.128.209#53: connected using 10.16.131.81#53097 TSIG kwai_base_key
26-Aug-2024 17:30:11.865 xfer-in: info: transfer of '4.10.in-addr.arpa/IN/base' from 10.16.128.209#53: Transfer status: up to date
26-Aug-2024 17:30:11.865 xfer-in: info: transfer of '4.10.in-addr.arpa/IN/base' from 10.16.128.209#53: Transfer completed: 0 messages, 1 records, 0 bytes, 0.001 secs (0 bytes/sec)
```


**影响**
有可能出现master存在配置更新，给slave发送notify，正常情况下，slave需要给master发送SOA(即  zone refresh 动作)，如果SN不一致，则会有后续的向master的IXFR请求。
但是，由于上诉的`# Zone refresh error: refresh: retry limit for master a.b.c.d#53 exceeded` 错误，有可能是 slave 给 master 发送 soa请求，但是无法收到 soa响应。这样会影响master的zone变更时， master 和 slave 之间的 zone的同步。

**怀疑点**
可能是 slave 和 master 之间的 refresh(SOA请求) 存在防火墙、存在限速、存在丢包 等。

```bash
try-tcp-refresh：is a compatibility setting that defaults to yes.  When set, if all prior attempts to obtain the SOA record over UDP have failed, then before giving up, there will be one more try, this time using TCP.  It would be unusual to change this setting.
```

## 其他日志
![](attachments/Pasted%20image%2020240408160017.png)

  `xfer-in` 达到配置阈值(**transfers-in** 或 **transfers-per-ns**): 则 zone transfer 被延迟发送。
  

**相关QA**
![](attachments/Pasted%20image%2020240408155646.png)

zone refresh的过程中或者等待 zone refresh的过程中，收到了notify消息，则比较 serial num；不相等，则 排队(queued) 一个 zone refresh。


# zone传输tune调优
参考:  [zone transfer tune](https://kb.isc.org/docs/aa-00726)

DNS系统中，在大家的直观印象下bind主备的同步都是“实时”的。实际上主备同步的速度有诸多的瓶颈。

**对于master而言**：  
1. 是否有delay notify消息，这个配置是 notify-delay,默认是5s，有必要的话是需要缩短的。  
2. transfers-out 限制同时允许区传输的数量，默认是10，如果slave多，zone多需要调大。  
3. notify-rate 、startup-notify-rate，对于master而言会限制master给slave发送notify的速度，默认是20,需要调大。

**对于slave而言**：  
1. transfers-per-ns限制了从单个master同步的并发，默认也是10,需要调大。  
2. transfers-in ：slave上配置，收到的 zone-transfer (xfer-in) 的 并发数的最大值。限制了同时从master（可能有多个）同步的总数，默认是10,需要调大。  
3. serial-query-rate 在slave中会限制slave向mastetr做SOA查询的频率。默认是20,需要调大。

对于单个同步的case，可以从master域名更新、master触发notify，slave收到notify，slave开始同步，slave完成同步几个关键的时间点，查看时间到底消耗到哪里。



## master 服务器调优
![](attachments/Pasted%20image%2020231225161905.png)
![](attachments/Pasted%20image%2020231225162526.png)

zone transfer时，在master上，影响的配置参数有：
（1）master到slave的notify的速率
```bash
serial-query-rate 
notify-delay
```

(2) zone transfer 并发个数限制
```bash
transfers-out
tcp-clients
tcp-listen-queue
reserved-sockets
```

(3) 其他的性能参数
```bash
max-transfer-time-out
max-transfer-idle-out
transfer-format
```



注：比较重要的参数有：**transfers-out, notify-delay， serial-query-rate, tcp-clients and tcp-listen-queue**.

### transfers-out

![](attachments/Pasted%20image%2020240408163744.png)

**含义**
master上设置，可以同时运行的发出的传输的最大值。超过限定的来自slave的域传输请求将会被拒绝。默认值为 10 。

**建议**

![](attachments/Pasted%20image%2020240408164039.png)


### tcp-clients
![](attachments/Pasted%20image%2020240408164427.png)

**含义**
本机收到的TCP连接的个数的最大值，默认值为100. 包含基于TCP的dns请求，以及 zone 传输。
- master 收到 slave的 tcp的 ixfr的 tcp 请求；
- slave 收到 client 的 基于tcp的 dns请求；

**查看**
rndc status 可以查看 tcp连接的个数。

### tcp-listen-queue

**含义**
即： 影响tcp ==全连接队列==的最大值，用在了 `listen()`函数中了。
```bash
int listen(int sockfd, int backlog);
```

半连接队列(lisen队列) 以及  全连接队列(accept队列)的大小：
![](attachments/Pasted%20image%2020240408165736.png)
```bash
- tcp_max_syn_backlog：`net.core.ipv4.tcp_max_syn_backlog` 来设置其值
- somaxconn： Linux 内核的参数，默认值是 128，可以通过 `net.core.somaxconn` 来设置其值；
- backlog： `listen(int sockfd, int backlog)` 函数中的 backlog 大小；
```

注：如果`tcp-listen-queue` 设置为0，则取系统的 
`somaxconn` 的值。


### serial-query-rate
**含义**
```bash
**serial-query-rate** (default 20) is a rate-limiter that has been used for a long time to control both the rate of notifies and of zone refresh (`SOA` queries).

Note that older versions of BIND managed notifies and SOA refresh queries in a single queue (which sometimes caused problems for slaves that are also masters for other servers). 

To ensure that notifies and refreshes were not competing with each other, in BIND versions 9.6-ESV-R11, 9.8.7, 9.9.5 and 9.10.0 we introduced independent queues - both still with serial-query-rate controlling them.

即：在 9.10.0 中，使用不同的queue来分区处理 notifies 和 soa请求(refresh)，如果slave不再是另外slave的master，也不会出现notify和soa请求公用queue的情况，  但是这些队列还是使用 serial-query-rate  来控制；

```
从bind9.11.1之后，使用**serial-query-rate** 、**notify-rate** 、**startup-notify-rate** 来控制。
```bash
From BIND 9.9.7-S1 (and this change will also be found in BIND 9.11.1) there are three separate rate-limiting controls: **serial-query-rate** ; **notify-rate** and **startup-notify-rate** .


**serial-query-rate** continues to control the rate at which SOA refresh queries are issued by secondary servers. 

**notify-rate** takes over as a configuration option for normal notifies (those sent out when a zone has been updated).  

startup-notify-rate allows the administrator to configure independently the rate at which notifies are sent out after restarting or reloading.

For all three of these, the default remains at 20, which may be too low for many production environments.  Administrators however are encouraged to increase the values of **serial-query-rate** and **notify-rate** gradually to find the levels that meet their production  
environment's requirements.
```

![](attachments/Pasted%20image%2020240409102841.png)

**serial-query-rate**：控制slave发起soa refresh查询的速率。默认值20. 比如，slave重启，则也会给master 发送大量的 soa请求。

**notify-rate** （标准notify通知）：zone发生变更时，master发送notify的频率。默认值20.
```bash
This specifies the rate at which NOTIFY requests are sent during normal zone maintenance operations. (NOTIFY requests due to initial zone loading
are subject to a separate rate limit; see below.) The default is 20
per second. The lowest possible rate is one per second; when set to
zero, it is silently raised to one.
```

**startup-notify-rate**（启动notify通知） ：named重启或者reload时，发起notify的速率。默认值20。
```bash
This is the rate at which NOTIFY requests are sent when the name server
is first starting up, or when zones have been newly added to the
name server. The default is 20 per second. The lowest possible rate is
one per second; when set to zero, it is silently raised to one.
```

注：标准notify通知的优先级高于启动notify通知。

**问题**
如果 startup-notify-rate 设置的非常大，那么 master 重启或者reload，那么就会短时间发送 大量的 notify给所有的slave的所有的zone。接下来，salve就会发送大量的 soa refresh请求。

![](attachments/Pasted%20image%2020240408222415.png)

### notify-delay
![](attachments/Pasted%20image%2020240409102306.png)

**含义**：
控制单个zone的变更发送notify的延迟时间。
默认值为5s，也就是意味着即使某个zone一直在不断的update，也会延迟5s之后才会给slave发送 notify，这样可以防止 zone的频繁更新造成的 notfiy 的 风暴(storm)。

### 其他参数
**transfer-format**
![](attachments/Pasted%20image%2020240409103332.png)


## slave服务器调优
![](attachments/Pasted%20image%2020231225163005.png)

zone transfer时，在slave上，影响的配置参数有：
（1）zone transfer的速率：
```bash
transfers-in
transfers-per-ns
transfers
```

(2)  refresh (SOA) queries：即soa查询的速率
```bash
serial-query-rate
min-refresh-time
max-refresh-time
min-retry-time
max-retry-time
```

(3) 其他的性能参数
```bash
try-tcp-refresh
max-transfer-time-in
max-transfer-idle-in
```

注：比较重要的参数有：**transfers-in**、 **transfers-per-ns**。

### transfers-in
```bash
Only used by slave zones. **transfer-in** determines the number of concurrent inbound zone transfers. Default is 10.
```

![](attachments/Pasted%20image%2020240409105151.png)

**含义**
slave上配置，收到的 zone-transfer (xfer-in) 的 并发数的最大值。

**理解**
```bash
If you make this value too large on a secondary  server with many zones that are frequently updated, you may find that your server is too busy handling zone transfers to handle queries effectively.  Depending also on your secondary zone configuration and zone data propagation strategy, each inbound zone transfer completion may cause onward notifications to other servers along with inbound SOA queries that also increase the workload.

```
不可以设置太大，否则如果zone变更很频发的话，可能影响本slave对于dns查询的处理。
如果本slave又是其他slave的master，那么本slave的zone传输完成之后，还需要给其他的slave发送notify，以及收取其他slave的soa请求。

### transfers-per-ns
![](attachments/Pasted%20image%2020240409105939.png)

**含义**
per-ns:  即 per primary server.  即其实可以配置多个 primary server（即 master）, 多数情况下，为了简单，只是配置了一个。

 transfers-per-ns 限制了每个 master 发送给本slave的 xfer-in 的 并发数最大值。如果存在多个master，通过限制每一个master给本slave的 xfer-in 的 并发数最大值，可以减轻每个master对于zone transfer的压力。


### serial-query-rate

![](attachments/Pasted%20image%2020240409110537.png)

![](attachments/Pasted%20image%2020240409114632.png)

**含义**
serial-query-rate 控制 slave 给 master 发送 SOA请求(zone refresh)的频率。其控制的是发送zone refresh的 动作的个数(zone refresh：发送SOA请求，以及后续的IXFR请求)，而不是包的个数。比如，一个zone refresh 可能存在多个包，比如没有收到 soa请求的响应，进行了重传，则认为还是一个 动作。
在slave进行重启之后，即使已经从zone文件中读取了配置，也会给master发送 大量的soa请求。


**建议**
serial-query-rate 如果设置的比较大，那么slave重启之后，发送大量的soa请求，可能给master造成压力。


### 其他参数
![](attachments/Pasted%20image%2020240409115616.png)

## ixfr-from-differences的功效

### 背景

常规情况下bind的主备同步是自动增量同步的。但是有些场景下是全量同步，比如自己手动改的zone文件，重新加载进去。  

一般内部的反解信息是根据所有的zone自动生成的，就会存在PTR记录每次全量同步的量非常大。


### 说明

![](attachments/Pasted%20image%2020240409120714.png)

### 测试
测试了可以通过打开 `ixfr-from-differences`，在master上自动计算差异，slave就可以做增量同步了。
```bash
ixfr-from-differences yes;
```

![](attachments/Pasted%20image%2020240122165222.png)

上图中可以看到之前没有打开`ixfr-from-differences`时同步1.9W条记录需要2.6s，开启之后每次增量同步只需要0.02s。
开启`ixfr-from-differences` 时会增加`master`的CPU、内存开销，所以需要根据实际的情况衡量是否需要打开。

## 其他的调优思路
### zone transfer失败的补救措施
####  heartbeat-interval 定期全量notify

**说明**
master 上配置 heartbeat-interval， 每隔一段间隔，就给所有的slave的所有的view下的zone发送notify消息。然后slave给master发送soa请求，如果存在zone的不一致，则会同步一致。

**缺点**
master每隔一段时间，都发送notify。无论zone是否发生变更，都是给所有的slave发送所有zone的notify。对于master来说，压力比较大。如果此时正好存在增量的zone的配置变更，则增量zone的配置变更生效时间很慢，因为master在忙着全量的notify。

#### 监控脚本检测master和slave的zone是否一致

**说明**
通过slave上的监控脚本，监控master和slave的zone是否一致。如果不一致，则slave给自身发送一个notify消息，促使自身给master发送soa请求，进而继续后续的zone 传输来保证同步。
```bash
# 本机配置，允许自身给自身发送notify消息：
options {
    ######省略了一些配置######
    allow-notify { 127.0.0.1; };
    ######省略了一些配置######
};

# 自身给自身发送notify消息的命令；
dig  @127.0.0.1 <zone name> SOA -t SOA  +norecurse +noadflag +aaflag +opcode=notify -y hmac-sha256:<view key name>:<view key>

+norecurse: 没有递归flag
+noadflag: noad flag(Non-authenticated data)
+aaflag: Set AA flag in query
+opcode=notify: 设置 opcode 为 notify

比如：
dig  @127.0.0.1 internal SOA +norecurse +noadflag +aaflag +opcode=notify  -y hmac-sha256:default_key:"default_view_key"

```


**缺点**
相比 master的 heartbeat-interval，可以减轻master的压力。
因为 slave给master发送 soa请求，是各个slave独自发送的，可以大概给打散了。防止master集中收到各个slave的大量的 soa请求。


### zone传输影响slave的dns查询处理
**背景**
一般而言，master不提供dns查询，只是动态接受zone的变更，然后同步给slave。slave提供client的 dns查询。
slave通过 bird 发布 anycast ip，这样多个slave就可以提供dns的集群服务。但是 配置变更之后，master需要和slave之间进行 zone的同步，会影响slave对于client的dns的请求。

**说明**
大多数情况下，client的dns请求都是使用的  UDP进行dns请求(TCP的情况很少)，对于client来说，dns的请求和响应也就是一问一答，一共2个包（如果是外网的dns请求，reslover和权威之间可能还会有迭代查询，但是对于client来说，就是2个包）。
那么，就可以通过控制 bird 的 anycast IP 的 path，对要进行配置下发的slave服务器，进行查询流量的摘除。摘除了之后，再对外提供dns查询服务。

> 即：不再是mster和slave之间进行配置同步，而是直接nsupdate给slave下发配置，给slave下发配置前，本slave不再对外提供dns查询服务，集群中的其他的slave对外提供dns查询服务，配置下发完成之后，本slave再次对外提供查询服务。


**缺点**
如果 client 是使用的 tcp 进行dns查询，那么建立连接以及查询、响应、以及断开连接是多个包。给slave配置下发的过程中，可能会导致查询异常。


# 测试

## nsupdate批量更改多个view下 的zone
```bash
# cat gen_nsupdate_multi_ops_cmd.sh
#!/bin/bash

batch=500
count=2000
IP=1.2.3.4
echo server 127.0.0.1
for v in $(awk -F '[ "]+' '/^view/{print $2}' /etc/named/named.conf.views |grep -v base);do
    key=$(awk -F '[ ";]+' '/kwai_'${v}'_key/{getline;getline;print $3}' /etc/named/named.conf.keys)
    echo key hmac-sha256:kwai_${v}_key ${key}
    echo zone test11.domain
    j=0
    for((i=0;i<count;i++));do
        echo update delete t${i}.dhb.test11.domain A
        echo update delete t${i}.dhb.test11.domain CNAME
        echo update add t${i}.dhb.test11.domain 10000 A ${IP}
        if [[ $j -eq $batch ]];then
            j=0
            echo send
        else
            ((j++))
        fi
    done
    if [[ $j -ne 0 ]];then
        echo send
    fi
done
echo quit
```

使用：
```bash
sh gen_nsupdate_multi_ops_cmd.sh > nsupdate_multi_ops_cmd.out
```


## 查看zone同步的生效时间
执行nsupdate的同时，同时查看配置变更是否在各个slave中生效。
```bash
date +%T.%N; nsupdate -v < nsupdate_multi_ops_cmd.out & while true;do echo $(date +%T.%N) $(for i in {2..9};do dig -b 10.44.79.146 t1999.dhb.test11.domain @192.21.45.$i +short +timeout=1;done);done

```

```bash
说明：
（1）nsupdate -v ： 表示使用 tcp 进行nsupdate 更新。
master 是本机，即 server 127.0.0.1

（2）nsupdate -v < nsupdate_multi_ops_cmd.out

< 表示标准输入；

（3）nsupdate -v < nsupdate_multi_ops_cmd.out &  
& 表示后台运行。这样 nsupdate的更新和 对于slave的dig查询可以同时进行。

```

改进测试：
```bash
所有slave生效之后自动停止，此中是3台slave：

batch_nsupdate_cmd_3.3.3.5.out： 多个view下的某个zone的多个记录进行更改，将记录更改为 3.3.3.5；
t1999.dhb.test11.internal 为 zone中更改的最后一个记录;
bjfs_idc：是最后一个更改的 view。
如果最后一个view中，该zone的最后一个记录生效了，那么其他所有view的zone的所有记录都生效了。

date +%T.%N; nsupdate -v < batch_nsupdate_cmd_3.3.3.5.out &  while true;do cnt=0; echo ""; echo -n "$(date +%T.%N) "; for i in 2 4 5;do rep=`dig t1999.dhb.test11.internal -y hmac-sha256:bjfs_idc:"xxxxx" @10.108.164.2${i} +timeout=1 +short`; if [[ "${rep}" == "3.3.3.5" ]]; then cnt=$((cnt+1));fi; echo -n "${rep} "; done; if [[ $cnt -eq 3 ]]; then break; fi; done

date +%T.%N; nsupdate -v < batch_nsupdate_cmd_2.2.4.5.out &  while true;do cnt=0; echo ""; echo -n "$(date +%T.%N) "; for i in 2 4 5;do rep=`dig t1999.dhb.test11.internal -y hmac-sha256:bjfs_idc:"xxxxx" @10.108.164.2${i} +timeout=1 +short`; if [[ "${rep}" == "2.2.4.5" ]]; then cnt=$((cnt+1));fi; echo -n "${rep} "; done; if [[ $cnt -eq 3 ]]; then break; fi; done
```


效果查看（生效时间查看）：
![](attachments/image%20(7).png)

如上所示，最开始slave中的A记录是 1.2.3.1； 后来多个slave都生效为 1.2.3.6。
开始时间为 16:15:03.02658, 所有slave都同步成功的时间为：16:15:04.25815。

## dig查询指定view下的zone记录

view中的match_clients字段用于基于clinet的信息选择view。
可在match_clients可以设置为 TSIG，ACL名称（client的sip，或 ECS网段）。

一般情况下，想要测试某个View，要么选择特定网段（匹配Client-ip）的测试机器，或者选择 ECS方式（需要bind9编译的时候就支持ECS），或者在任意一个测试机器上，dig测试时指定TSIG。

相比较之下，通过指定TSIG的方式，匹配到指定的view更加通用。如下所示：

```bash
dig @1.1.1.1 t1999.dhb.test11.internal -y hmac-sha256:bjfs_idc_key:"xxxxxxxxxx" +short
```

# 参考
```bash
# Tuning your BIND configuration effectively for zone transfers
https://kb.isc.org/docs/aa-00726

# bind9的配置文件中的配置解释 （++++++++++++）
https://chengqian90.com/DNS/DNS%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B9%8BBIND9.html

# Tuning DNS for TCP queries
https://ant.isi.edu/diiner/tcp/index.html

# # [DNS & bind从基础到深入](https://www.cnblogs.com/sandshell/p/11674957.html)
```