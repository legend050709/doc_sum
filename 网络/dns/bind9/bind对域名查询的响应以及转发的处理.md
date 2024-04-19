```table-of-contents
```
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
# 参考
```bash

```