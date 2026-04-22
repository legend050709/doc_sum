```table-of-contents
```
# 概述
DNS是内网中非常重要的基础设施，影响着几乎全部的服务。为了保证服务的稳定可靠，我们使用了bind9进行集群部署，对外发布`10.10.10.10`的`anycast`地址，并且提供了10倍以上的容量冗余。
# 问题
虽然提供了10倍以上的容量冗余，但是线上还是偶尔的出现丢包。

# 分析
对于TCP服务而言，偶发的丢包是没有影响，因为TCP协议层存在重传。但是DNS大部分使用的是UDP协议，大部分进行DNS请求的Client没有很好的重传机制。一次丢包，就可能导致业务超时。

==DNS丢包的问题，大部分场景是微突发导致的==。
由于我们优化的目标就是请求不失败，可以从2个方面下手：

**1》增加socket缓冲区的接收缓存大小**
```bash
DNS使用的是开源的bind9程序，基本上是半黑盒的，目前更多的是运维（比如：调参数，设置参数， 写插件，主体代码的开发基本不涉及），开发改造成本高。

目前的DNS程序独占服务器，可以直接修改内核参数提升socket缓冲区的大小。

注：经过分析，将socket的接收缓冲区设置为5M时，在最高负载的情况下，会给每个请求增加30ms的时延，在可接受范围内。
```

**2》优化性能**
DNS会将请求记录到日志文件，由于多线程写同一个文件存在加锁，并且flush到文件也会影响性能。在记录文件的情况下，单机的性能大概是8wQPS；
本次优化将DNS记录日志更改为异步操作（通过插件的形式单独设置一个日志线程写日志，其他线程将信息传递给日志线程），去掉每次写文件的flush操作，大大提升了DNS写日志的性能，从8wQPS提升到50wQPS。

**3》成果**
优化之前，某些DNS集群，大概有0.06%的请求失败；优化后，失败率降到0;

# 具体优化
## DNS日志性能优化
### 背景
DNS日志开启之后，会影响性能。但是日常又需要基于日志（A/4A的DNS请求时间，响应结果，响应时间，状态）进行定位问题。
当前bind日志是写文件，多线程写日志有锁，并且每次会flush，所以性能比较差。
### 目标
1. 默认记录全部请求日志
2. 单独记录公网请求日志
3. QPS超过阈值之后，对日志进行限速，但是不关闭日志。
### 方案
#### 插件形式的独立写日志线程
可以用插件（即：自己实现一个querylog.so，在bind启动的时候加载，在so中会启动一个写日志的线程）记录日志，可以减少对DNS本身代码的修改。
要记录的日志可以放到队列里，然后用一个单独的线程写日志。如果写文件阻塞，会导致队列满，请求处理过程中就可以丢掉这个日志，保证请求处理的性能。
![](attachments/image%20(36)%201.png)

#### 日志线程绑核
为了防止日志记录对于dns查询的影响「丢包以及时延变大」，设置CPU核独占的query-log线程。

## 主从同步性能优化
优化集群中master和slave之间的zone变更同步时间，且同步期间不影响dns的域名查询。
### 背景
当前的dns在配置下发时可能存在配置生效时间长的问题。个别业务存在频繁的域名记录更新（比如：KDB要求2000+域名5s生效），需要较高的时效性，因此需要对dns的配置下发以及配置主从同步进行优化。

### 目标
目标暂定：

主从配置同步期间，不影响slave的dns查询解析（比如：10Mqps下无查询解析的丢包）；
主从配置同步期间，不影响后续增量配置的变更（比如：后续一次性对所有view下「线上40个view」的6000条RR记录的变更在10秒内生效）。
主从配置的数据一致，即主从配置可以达到同步，即使某次不同步，可以在30s内进行报警以及自动同步。
### 分析
#### zone传输场景
![](attachments/image%20(32)%201.png)

如上所示，主从之间的zone传输的场景有：

（1）named的启动、重启
- master或者slave的重启

（2）slave上的zone refresh超时
- slave上的zone存在refresh时间（一般3h），超时之后会给master发送soa请求，检查是否存在zone的更新。

（3）master上的zone存在更新
- 更改zone文件，然后reload的方式
- nsupdate动态更新的方式

（4） master定期同步
- master配置了heartbeat心跳，会定期的发送notify检查主从的zone配置是否一致

### 优化手段：

可以**关闭master的定时全量更新通知**，设置heartbeat-interval为0 。优化slave的监控，如果有配置不同步的情况，主动给slave（127.0.0.1）发送notify消息。只针对不同步的view的zone发送notify即可。

优点：
- 避免master集中发送大量notify影响各种性能和时效性指标（knp给master的增量配置更新）。
- notify在slave上发送，各个slave可以打散，避免集中触发更新。
- slave的监控是30s检查一次。在不同步的情况下，可以更快的自动恢复。

发送notify方式：slave需要增加配置，允许接受本机发送的notify; 否则如下所示，出现refused.

`/etc/named/named.conf.options`配置文件的`options`增加`allow-notify { 127.0.0.1; };`
```bash
options {
    directory "/var/named";
    pid-file "/var/run/named/named.pid";
    ######省略了一些配置######
    allow-notify { 127.0.0.1; };
    ######省略了一些配置######
};

添加如上记录之后， kbind reload 即可生效。
```
![](attachments/image%20(33)%201.png)

用dig发送notify。
需要从配置文件获取不同的zone的view的key
```bash
dig  @127.0.0.1 <zone name> SOA -t SOA  +norecurse +noadflag +aaflag +opcode=notify -y hmac-sha256:<view key name>:<view key>

+norecurse: 没有递归flag
+noadflag: noad flag(Non-authenticated data)
+aaflag: Set AA flag in query
+opcode=notify: 设置 opcode 为 notify

比如：
dig  @127.0.0.1 internal SOA +norecurse +noadflag +aaflag +opcode=notify  -y hmac-sha256:kwai_default_key:"U2LTw11jcgl5Lc2pm3/P8GVHV10DTz/1fc1yrXdAVcA="

```
![](attachments/image%20(34)%201.png)


### 结论
named.conf.options 中的配置参数更改，如下所示：
```bash
（1）优化后的配置如下：
    	# 以下配置，在master和slave的 named.conf.options 均配置；
			dialup						no;
      heartbeat-interval  0;
      
      notify-rate	 2000;
      notify-delay	1;
      
      tcp-clients     2000;
      tcp-listen-queue	200;
      
      transfers-in 200;
      transfers-per-ns 200;
      transfers-out 2000;
      serial-query-rate  200;
      
      # 以下配置，slave 的 named.conf.options 配置，master 不配置；
      allow-notify { 127.0.0.1; };  
      
（2）平台的配置情况：
	2.1）已开放的配置
  		dialup	no;    					# 之前配置为 notify
      heartbeat-interval	0;  # 之前配置为 5
      notify-rate	2000;				# 之前配置为 100
      transfers-in	200;			# 之前配置为 50
      transfers-out	2000;			# 之前配置为 50
      serial-query-rate	200;	# 之前配置为 50

      
  2.2）需要添加的配置
  		notify-delay	1; 							# 之前未配置，程序默认值为 5
      tcp-clients     2000;					# 之前平台默认下发为 200；之前在限速配置里？
      tcp-listen-queue	200;				# 之前平台默认下发为 30；之前在限速配置里？
      transfers-per-ns 200;					# 之前未配置，程序默认值为 2
      allow-notify { 127.0.0.1; };	# 之前未配置，程序默认值为 {none;}
```

|   |   |   |
|---|---|---|
|测试项|配置更改前|配置更改后|
|存在40个view，4000个zone时更改40个view下某个zone的6000个记录 + dnsperf打流<br><br>「master 9.11，8个slave 9.11」|从配置下发到所有的slave中的zone生效大概需要5.5s-10.5s「相差了一个notify-delay：5s」；<br><br>最差的情况下(heartbeat-interval)时，大概需要84s。<br><br>同时使用dnsperf打流，10wQPS多数场景下无丢包，最差有0.01%的丢包率。|从配置下发到所有的slave中的zone生效大概需要的总体时间在6.5-7.2s之间。<br><br>同时使用dnsperf打流，10wQPS无丢包。|
|测试项目同上<br><br>「master 9.11，4个slave 9.11，4个slave 9.18」|基本同上|基本同上<br><br>「只测试了10WQPS查看是否丢包，实际可能更高」|
|测试项目同上<br><br>「master 9.18，8个slave 9.11」|基本同上|基本同上|
|测试项目同上<br><br>「master 9.18，8个slave 9.18」|基本同上|基本同上|
|测试项目同上<br><br>「master 9.18，4个slave 9.11，4个slave 9.11」|基本同上|基本同上|


因此 ：不考虑从KNP变更zone的记录到数据库落库，再到触发nsupdate的时间。单单对master-dns机器上进行nsupdate变更到8个slave中配置生效，当前的named支持的不同view下的zone的RR记录变更的频率为：800个/s（使用6000/7.0s计算）；

## 支持外网域名解析黑名单

### bind9的PRZ
bind9支持RPZ功能[response policy zone(rpz) rewriting](https://bind9.readthedocs.io/en/v9_18_8/reference.html?highlight=rpz#response-policy-zone-rpz-rewriting)，可以针对单个域名修改返回的结果。

rpz和其他zone配置方式一致。

支持的封禁方式有：
(1)返回NXDOMAIN
- CNAME设置为.

(2) 返回NODATA
- CNAME设置为*.

(3) 返回指定IP
- A设置为指定IP

### 提前劫持
默认的RPZ的策略默认是reslover向外网发起请求，获取到结果之后，对结果进行替换，然后返回给client。
期望是不向外网发起请求， 直接向client响应结果。

`named.conf.views` 文件中：
之前的配置如下：
```bash
    response-policy {
        zone "blacklist";
    };
```

在 `response-policy {…}` 后添加`qname-wait-recurse no`配置, 即更改为：
```bash
    response-policy {
        zone "blacklist";
    } qname-wait-recurse no;
```

更改上诉配置之后，`kbind reload` 就可以生效；再进行 请求，就不会向外网发起请求，直接给client进行回复。

## DNS服务的可观测性增强
请求日志支持纪录请求协议、请求时间、响应状态码、响应时间及首个A纪录等信息。

### 目标
运营同学需要在dns的查询日志，qlog.log中打印域名的首个A记录或者4A记录。并且获取bind9解析花费的时间。
> qlog.log 和 qlog-ext.log;
> 一个是所有的查询日志，一个是外网的查询日志。

在上序日志的基础上：
（1）打印首个A记录/4A记录的信息。
（2）打印dns收到dns查询的时间
（3）bind9的解析时间
（4）响应状态码

注：在日志插件中进行日志的打印，而不是主体代码中打印，是为了不影响bind的dns解析的性能。

### 分析
在日志插件中，查询请求的时间，request_time的获取比较容易。现在的难点主要在于第一个A记录的获取上。
当前的日志打印的插件是定义在 NS_QUERY_DONE_SEND钩子点上，在`ns_query_done` 函数中调用该钩子点。但是该钩子点这已经是定义的最后一个钩子点，后续的处理函数中不会再有钩子调用。
如果一个域名存在多个A记录，此时还没有对A记录进行排序，因此在NS_QUERY_DONE_SEND钩子点上无法获取到第一个A记录。

### 方案一：定义其他的钩子点，在A记录排序之后的函数中调用
介绍：
在bind的主体代码逻辑中添加新的类型的钩子点。
当前bind主体代码中的钩子点不全，只能覆盖到解析结束。此时解析得到的RR记录是没有排序的，进而就无法获取到第一个RR记录。RR记录的排序逻辑是在要发送响应报文之前进行的。因此，需要在RR记录排序之后的主题代码逻辑中添加新的类型的钩子点。



## 小结
在DNS域名解析服务优化项目上，先后完成主从同步性能优化（从6.5s-10.5s「极端场景84s」优化为6.5s-7.2s、极端场景下0.01%的丢包优化为无丢包）
QueryLog性能优化（写日志速率从原来23.5万行/秒提升至26万行/秒）；
在可观测性（请求日志支持纪录请求协议、请求时间、响应状态码、响应时间及首个A纪录）及安全性增强上（支持外网域名解析黑名单）也有产出。

# 总结