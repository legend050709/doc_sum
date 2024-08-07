```table-of-contents
```

# 转发器
## 发生时机

dns服务器不是某个查询的 权威服务器，且本地不存在该域名的缓存cache。那么就会将 请求转发给 转发器。


# 递归client请求的限速

## 背景

如果client查询外网域名，那么对于dns解析器(dns转发服务器)来说，收到client的请求，发现本地不存在记录 以及缓存，需要向外网dns服务器转发该client的请求（即递归查询），收到响应之后，将结果进行缓存，然后再响应给client。

**对于解析dns服务器来说，递归查询比较消耗资源，尤其是client 请求的外网域名的解析消耗较长时间 或者 不可以解析的情况下，此时会长时间占用资源**。因此，需要在解析服务器上限制递归查询的速率。

## 介绍


## 配置
```bash
（1） 配置一：
recursion yes | no;
recursive-clients number;
allow-recursion { address_match_list | any | none;};



配置实例：
recursion      yes;
allow-recursion   { any; };
或
allow-recursion { 10.0/16; };
recursive-clients 25;


（2）配置二：

max-recursion-queries
clients-per-query
max-clients-per-query

fetches-per-zone
fetches-per-server
fetch-quota-params
```

![](attachments/Pasted%20image%2020240416183113.png)

服务器同时为用户执行的递归查询的最大数量。默认值1000，因为每个递归用户使用许多内存，一般为20KB，主机上的recursive-clients选项值必须根据实际内存大小调整。

具体的 解释，可以参见：DNS-BIND理解。


参考：https://bind9.readthedocs.io/en/v9.18.25/reference.html

### clients-per-query 和 max-clients-per-query 区别

![](attachments/Pasted%20image%2020240416191518.png)


clients-per-query： 设置 对于特定外网域名（指定 type，name）的请求的 client的数量的**初始最大值**，该值是自我调节(self-tune)的，而 max-clients-per-query 是 可以自我调节到的最大值。clients-per-query 的 每次调节都会打印日志。

clients-per-query 的 值反应了在特定的域名从转发到外网到获取解析之前的这段时间内，dns迭代服务器收到了多少个这个域名的请求。

如果在获取特定域名的解析结果之前，client 对于这个特定域名的查询请求数量超过了 设定的 `clients-per-query`，那么 named 就认为是在查询一个没有响应的 zone 下的域名，因此会 丢弃掉额外对于这个特定域名的查询。
在丢弃了额外的查询之后，named又收到了这个特定域名的响应，其就会增加 估计值，即增加 `clients-per-query`（每次增加 5），最大只能到  `max-clients-per-query`。
如果  `clients-per-query` 20 分钟后保持不变，则降低估计值（每次减少1），最低值是最初设置的默认的10（即 减低 到 10 之后， `clients-per-query` 不会再继续减少）。

![](attachments/Pasted%20image%2020240607193053.png)

因此，异常情况下，当远程的区域服务器无法访问时，用于减轻负载的重要配置选项是 max-clients-per-query。但是，正常情况下，又需要 max-clients-per-query 设置的足够大，来解决存在多余特定域名的多个并发请求。

```text
大多数没有得到答案的客户端都会重试查询，如果 max-clients-per-query 设置的过小，应该也问题不大，不会导致 client的查询解析的延迟变大。

如果正在进行中的查询从外网服务器得到了解析结果，那么解析结果将会被加入到缓存中（解析结果也不会傻到TTL=0而不加入缓存的情况），那么client重试的查询可以立刻从缓存中得到结果。

如果外网服务器对于特定域名的查询没有响应，那么迭代服务器就会给client的查询返回 SERVFAIL，SERVFAIL 这个结果也会被加入到 缓存中。缓存的时间周期是 servfail-ttl 决定的（默认是1s）
```


注意： 如果 max-clients-per-query 设置为零（即没有上限），则除了`recursive-clients` 施加的上限之外，没有上限。如果`clients-per-query`设置为零，则`max-clients-per-query`不再适用，并且除了`recursive-clients`的上限之外，没有上限。
> 即： `recursive-clients` 和 `max-clients-per-query` 只要一个为0，只有 `recursive-clients`的 限制。


参考：[# How does clients-per-query work](https://kb.isc.org/docs/aa-00463)

### clients-per-query 和 recursive-clients 的区别

![](attachments/Pasted%20image%2020240607200037.png)

recursive-clients: 限制的是 迭代服务器向外转发的 外网查询的个数。如果多个client 并发请求相同的外网域名，那么只对外发送一个迭代请求，占用一个 recursive-clients 限额。


max-clients-per-query 和 clients-per-query 限制的是对于特定的外网域名(同样的name，同样的type的查询) 的递归查询的并发client的请求个数，而 recursive-clients 针对的 迭代服务器向 外网转发的 任意外网域名的的并发请求个数。

即：一个限制的是 client 的并发请求个数，一个限制的是迭代服务器向外的并发请求个数。

### clients-per-query 和 fetches-per-zone 的区别

![](attachments/Pasted%20image%2020240416193137.png)

max-clients-per-query 指示的是对于外网的某个特定的递归查询（多个并发查询有相同的请求name和type）的并发请求限制。
如果对外网的多个并发请求不同，但是属于一个zone或者其下的子zone，那么 max-clients-per-query 将无法进行限制。
即 多个client 对于特定域名以及类型的多个并发查询，被归拢为一个 fetch，只迭代发送一个递归请求给外网dns服务器。
隶属于同一个zone及其下的子zone的不同类型以及域名的查询，属于不同的fetch，fetch的个数受限于 fetches-per-zone。

超过阈值之后的动作：
1> 直接丢弃。（默认动作）
2> 给client 回复 SERVFAIL。


### fetches-per-server 和  fetches-per-zone 的区别

![](attachments/Pasted%20image%2020240417105557.png)

fetches-per-server：单个外网服务器(即：reslove 服务器的转发服务器) 可以接收的并发的 fetch 的最大数量。

 对于特定域名以及类型的多个并发查询，被归拢为一个 fetch，只迭代发送一个递归请求给外网dns服务器。如果 reslover 服务器 并发的转发多个外网请求给 外网的权威dns服务器，那么限制转发给外网服务器的 fetch的数量。

注：fetches-per-server 是动态调整的， 需要和 fetch-quota-params 配合使用。
如下所示：
```none
fetches-per-server 200 fail;
fetch-quota-params 100 0.1 0.3 0.7;
```


### 查看
#### clients-per-query 日志

![](attachments/Pasted%20image%2020240618143705.png)


#### 由于 限速导致的丢包统计

```bash
Responses dropped by rate limits are included in the `RateDropped` and `QryDropped` statistics. 

Responses that are truncated by rate limits are included in `RateSlipped` and `RespTruncated`.

```

**QryDropped**

![](attachments/Pasted%20image%2020240607181002.png)

![](attachments/Pasted%20image%2020240607181330.png)

![](attachments/Pasted%20image%2020240607181413.png)

## 查看
打印当前正在递归的查询；执行下面的命令，会将结果打印到指定的文件中。
```bash
（1）配置
option中配置如下：
recursing-file <quoted_string>;

（2）查看
rndc recursing : 
	Dump the queries that are currently recursing (named.recursing)
```

## 其他
如果 解析器到外网权威服务器之间出现了 
```bash
no more recursive clients: quota reached
```
的问题。那么可能的原因是：
1. 查询的外网域名的响应时间长 （如：解析路径长、请求的外网域名不存在）
2. 大量的外网查询
3. 解析器和外网服务器之间的网络存在问题


# 响应限速
## 背景
### DNS反射攻击
DNS 使用UDP协议，UDP没有连接，很容易出现 UDP的反射攻击。DNS查询包很小，伪造DNS查询的源IP，DNS响应包很大。

![](attachments/Pasted%20image%2020240416152917.png)

攻击者为了攻击某个设备，一般是伪造某个sip，给dns权威服务器集群，发送DNS请求。
这样大量的DNS响应（DNS响应包大小远大于请求包大小）发送给了被攻击者。
即：攻击者伪造的sip大多数是同一个sip。这样才可以做到大量的响应发送给同一个设备，达到攻击的目的。

```bash
 The attacker sends out a large number of DNS queries that are forged to look like they were sent by the victim, so that the large response packets get sent to that victim.  This is the classic DNS DDoS. 
 
```

## 介绍
```bash
rate-limit {
	all-per-second <integer>;
	errors-per-second <integer>;
	exempt-clients { <address_match_element>; ... };
	ipv4-prefix-length <integer>;
	ipv6-prefix-length <integer>;
	log-only <boolean>;
	max-table-size <integer>;
	min-table-size <integer>;
	nodata-per-second <integer>;
	nxdomains-per-second <integer>;
	qps-scale <integer>;
	referrals-per-second <integer>;
	responses-per-second <integer>;
	slip <integer>;
	window <integer>;
};
```
参见：https://bind9.readthedocs.io/en/v9.18.25/reference.html#response-rate-limiting


Response Rate Limiting (RRL)：
DNS服务器无法基于DNS查询的sip判断哪些是攻击的DNS请求，哪些是正常的DNS请求。但是如果短期内大量的DNS请求都是用类似的 SIP地址，以及请求相同的内容，那么大概率是DNS攻击。

```bash
If one packet with a forged source address arrives at a DNS server, there is no way for the server to tell it is forged. 

If hundreds of packets per second arrive with very similar source addresses asking for similar or identical information, there is a very high probability of those packets, as a group, being part of an attack. 

The RRL software has two parts. It detects patterns in arriving queries, and when it finds a pattern that suggests abuse, it can reduce the rate at which the replies are sent.

```


**原理**
RRL 使用的是 令牌桶的 原理来进行限速。

![](attachments/Pasted%20image%2020240416155442.png)

## 配置

配置位置： 配置在 options 中 或 view中。
```bash
rate-limit {
      slip 2; // Every other response truncated
      window 15; // Seconds to bucket
      responses-per-second 5;// # of good responses per prefix-length/sec
      referrals-per-second 5; // referral responses
      nodata-per-second 5; // nodata responses
      nxdomains-per-second 5; // nxdomain responses
      errors-per-second 5; // error responses
      all-per-second 20; // When we drop all
      log-only no; // Debugging mode
      qps-scale 250; // x / 1000 * per-second
                    // = new drop limit
      exempt-clients { 127.0.0.1; 192.153.154.0/24; 192.160.238.0/24 };
      ipv4-prefix-length 24; // Define the IPv4 block size
      ipv6-prefix-length 56; // Define the IPv6 block size
      max-table-size 20000; // 40 bytes * this number = max memory
      min-table-size 500; // pre-allocate to speed startup
};
```

范例如下：

```bash
options {
               directory "/var/named";
               ...
               rate-limit {
                    responses-per-second 10;
                    log-only yes;
               };
          };
```
## 其他
**查询日志中丢弃查询的日志**

```bash
07-Jun-2013 12:44:44.868 queries: info: client 1.2.3.4#57114       (testhost.example.com): query: testhost.example.com IN A +ED (1.2.3.4)
07-Jun-2013      12:44:44.869 query-errors: info: client 1.2.3.4#57114       (testhost.example.com): rate limit drop response to 1.2.3.0/24 for       testhost.example.com IN A  (3ee9836b)
```

# 缓存

对于非权威记录的请求，dns需forward请求到外部的解析器。然后将结果缓存到本地。

## 过期缓存  stale cache
过期缓存：
- 过期缓存不删除，继续保存在cache中
- 允许将过期缓存响应给client


### 配置

![](attachments/Pasted%20image%2020240618144445.png)




```bash


（1）stale-cache-enable: 
缓存过期的cache记录。默认为no。

（2）stale-answer-enable： 
指定时间（stale-answer-client-timeout）内，收不到转发器的响应时，返回cache中过期的记录；可通过  rndc serve-stale on/off 开启和关闭；默认值为no；

（3）stale-answer-client-timeout：
收到clinet的请求，如果在 stale cache中查询到记录，依然会向转发器转发请求，在 指定时间（stale-answer-client-timeout）内获取到转发器的响应，则更新缓存中的记录，否则则向client回复 stale-cache 中的记录。推荐值为1.8s；如果该值配置为0，则立刻向client返回过期的缓存记录，同时也会向转发器发送请求。

（4）stale-answer-ttl：
返回的过期的 answer 的 ttl。

（5）stale-refresh-time：
如果转发器对于查询没有响应，则设置一个时间窗口，在该时间窗口内，新的client请求来查询该记录，在重新尝试请求转发器之前，named 将立即返回所请求的 RRSet 的“过时”缓存答案。默认值为30s，因为 RFC 8767 建议尝试cache 刷新（即向转发器请求）的频率不超过 30 秒。如果设置为0，即意味着每次都是先向转发器请求，请求失败之后才返回过期的缓存记录。

（6）max-stale-ttl ：
cache中的记录过期之后，如果转发器对于请求没有响应，那么缓存中保存过期记录的最大时间。 

（7）resolver-query-timeout：
解析器向转发器发送请求，超过一定时间，则认为查询失败。默认为10s


（8）servfail-ttl：
SERVFAIL 响应的 缓存时间。如果设置为0，则不缓存  SERVFAIL 结果。默认值为1s。如果查询设置了 CD（Checking Disabled，检查禁用）位，则不会查阅 SERVFAIL 缓存；这允许向转发器重试请求，而无需等待 SERVFAIL TTL 过期。

```

![](attachments/Pasted%20image%2020240618144435.png)



### 适用场景

比如，配置了多个  forwarder，但是某些forwarder 中的 某个zone的记录 存在限速，导致 client的请求 可能存在概率性的失败。但是为了高可用的考虑，又不太可能将这个 forwarder 给删除了。

那么可以考虑 缓存过期的 记录 或者 提前预取记录。


## 缓存过期前 预取 cache prefetch
### 背景

重新抓发请求到获取查询回复所需的时间可能比从缓存中返回查询所需的时间长 100 倍。
这就可能导致存在cache 和不存在 cache的响应时间的抖动。



### 原理

收到 client的 dns查询时，在缓存中查找到该请求对应的记录，如果 该记录的缓存剩余时间小于配置值，则在将缓存中的记录响应给 client 的同时，自动向转发器请求该记录。收到forwarder的响应后，如果记录值没有变更，则更新缓存记录的TTL即可，否则更新缓存记录。


> 注：自动向转发器发送请求，是事件触发的，即存在来自于client的查询 + 缓存记录的剩余时间小于配置值。


这样，对于那种频繁请求的记录，保证了其请求时都是存在缓存的，保证了client请求的响应时间的平滑、稳定、

### 配置

![](attachments/Pasted%20image%2020240618153629.png)

存在2个参数，其中第二个参数是可选的。
```bash

参数一：
	来自于client的请求到达时，缓存记录的过期的剩余时间 小于该记录值，则就会自动向转发器发起请求。

参数二：
	第二个参数是可选的，只有原始记录的TTL值 大于指定值的记录才可以被运行 prefetch.
```

```bash
options {
       ...
       prefetch 2 9;
    };
```


```bash
options {
       ...
       prefetch 0;
    };

prefetch: 配置为0，即关闭 prefetch。
```

## 查看缓存

```bash
(1) 生成缓存
rndc dumpdb -cache

(2) 缓存文件位置
/etc/named/named.conf.options 中的

options {
  dump-file "/var/named/stats/cache_dump.db";
};
```

查看记录：cache 记录 是以 view 为 基础进行组织的。

![](attachments/Pasted%20image%2020240618163413.png)


### 格式说明

![](attachments/Pasted%20image%2020240618163533.png)


```
与缓存内容相关的统计计数器，按view 进行维护。
“NXDOMAIN”计数器是已缓存为不存在的名称的数量。

以 RR 类型命名的计数器指示缓存数据库中每种类型的活动 RRset 数量。

（1）如果 RR 类型名称前面有感叹号 (!)，则表示缓存中的记录数，表明特定名称不存在该类型；这也称为“NXRRSET”。

（2）如果 RR 类型名称前面带有井号 (#)，则表示缓存中存在该类型但 TTL 已过期的 RRset 数量；仅当启用过时答案时才可使用这些 RRset。

（3）如果 RR 类型名称前面有波形符 (~)，则它表示缓存数据库中存在但标记为垃圾回收的该类型的 RRset 数量；无法使用这些 RRset。
```

# 参考
```bash
# How does clients-per-query work?
https://kb.isc.org/docs/aa-00463

# Early refresh of cache records (cache prefetch) in BIND
https://kb.isc.org/v1/docs/aa-01122
```