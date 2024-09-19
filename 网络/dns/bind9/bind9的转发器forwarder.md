```table-of-contents
```
# 转发器介绍
当`bind9` 设置了`forwarders`转发器后，所有非本域的和在缓存中无法找到的域名查询都将转发到设置的DNS转发器上，由这台DNS来完成解析工作并做缓存，因此这台转发器的缓存中记录了丰富的域名信息。因而对非本域的查询，很可能转发器就可以在缓存中找到答案，避免了再次向外部发送查询，减少了流量。

# 多个转发器

在 named.conf.options 配置多个转发器；如下所示：
```bash
    forwarders {
            223.5.5.5;
            119.29.29.29;
    };

223.5.5.5:   阿里的public pod;
119.29.29.29: 腾讯的public pod;
```


## 无效授权

**delegation授权**
授权是指将部分工作交给或转移到别处的行为。在 DNS 中，授权的具体含义是将查询重定向到其他的DNS服务器，比如迭代查询中，查找`example.com`,根服务器返回`com.`域名的NS服务器信息，让迭代器 查询`com.`域名的NS服务器。

![](attachments/Pasted%20image%2020240702182028.png)

在域名系统 (DNS) 中，“无效授权(Lame delegation)”或“无效响应”是指已委派一个或多个名称服务器为某个域提供权威 DNS 信息，但这些服务器未能提供这些信息。

### 无效授权的原因

当负责为某个域提供权威答案的名称服务器未能对 DNS 查询作出响应或以某种方式作出错误响应时，就会发生无效授权。造成无效授权的原因可能是：

- **不可用**。当名称服务器无法访问、未正常运行或名称服务器的 IP 地址无法路由时，可能会出现无效授权。
- **配置错误**。当名称服务器配置错误或其 DNS 软件未正常工作时，它们将无法对查询作出响应。
- **无法连接**。网络连接问题或防火墙限制可能导致名称服务器无法访问。
- **不一致**。如果多个所委派的名称服务器的 DNS 配置不一致，则可能会导致无效授权。当 DNS 区域数据不一致或区域传输不正确时，可能会导致配置不一致。
- **不存在**。如果某个域的指定名称服务器不存在，或者在未更新委派记录的情况下被停用，那么该名称服务器将无法正确地响应 DNS 查询。
- **缺少更新**。在进行新的委派后，如果特定的名称服务器尚未进行配置或更新，它将无法对 DNS 查询返回正确的响应。

### 无效授权的影响

无效授权可能会产生各种不利影响。

- **DNS 查找时间变长**。在发生无效授权时，DNS 请求可能需要更长的时间来解析，这可能会导致网页加载和 Web 资源访问出现延迟。
- **解析失败**。当授权无效时，对域名的 DNS 解析可能会失败，导致用户和设备无法访问该域。这可能会引发功能缺失和停机。
- **用户体验降级**。查找时间变长和解析失败将不可避免地导致用户体验不佳，因为用户可能会遇到延迟、超时或错误等问题。
- **安全威胁**。攻击者可能会利用无响应或配置错误的名称服务器发起一系列 DNS 攻击，如 DNS 劫持、缓存中毒或放大攻击。

### 如何才能防止发生无效授权

IT 和安全团队可以遵循一些最佳实践，以防止发生无效授权并确保 DNS 服务能够继续高效运行。

- 定期验证名称服务器的配置可确保名称服务器设置正确并能够正确地响应 DNS 查询。
- 将 DNS 委派分发给位于不同位置和网络中的多个名称服务器有助于提高冗余性，并确保系统在某个服务器无响应时仍然可以继续处理查询。
- 通过主动监控名称服务器的运行状况和响应速度，可以及早发现并解决问题。
- 定期执行 DNS 审计有助于验证所有权威 DNS 服务器上的委派信息是否是最新、准确且一致。
- 启用 DNS 安全扩展 (DNSSEC) 可以提高 DNS 解析的安全性和完整性，确保 DNS 响应的真实性并降低发生 DNS 欺骗的风险。

## lame server

![](attachments/Pasted%20image%2020240702203239.png)

lame server（糟糕的服务器） 的 判定：
（1）referral 过程中（即迭代过程中），某个NS无法给出下一步迭代的信息，则该NS被标记为 lame server。

（2）迭代过程中，reslover和某个NS之间的网络出现问题，这个NS可能被标记为 Lame。
（3）迭代过程中，某个NS负责该zone，但是没配置要查询的域名，则这个NS被标记为 Lame。


在 BIND中，以上的几种情况，只有情况1才被标记为 Lame Server。

因此，lame server 指的是 迭代查询过程中，不可以提供下一步信息的NS。lame server 能否用来标识  forwarder 呢?? 应该也是可以标识的。如下所示：

![](attachments/Pasted%20image%2020240905121159.png)

###  lame cache

在Bind 中，lame nameservers 和 valid nameservers 是分开缓存的，lame nameservers 的缓存被用来避免reslover 在一段时间内（`lame-ttl`）向这些Server进行后续的同类型域名(Qname/Type)的查询。

lame cache的作用是：如果权威服务器以特定的破损方式（比如 time out 或者 ServeFail ）响应解析器(reslover)的查询，则来自于客户端的对于同一 <QNAME, QTYPE> 的查询，在可配置的时间内不会触发对同一forwarder的进一步查询。
通过在 named.conf 中将 lame-ttl 选项设置为大于 0 的值来启用 lame cache。默认配置中该选项设置为 lame-ttl 600，这意味着脆弱缓存默认是启用的。

> 理解：我的理解是：lame cache 以 <QNAME, QTYPE> 为key，以 server-ip 为value；在对于某个域名的查询 选择 forwarder的时候，可能会先排除掉 lame cache中该域名对应的 sever。


#### lame cache 的问题

![](attachments/Pasted%20image%2020240906165334.png)

![](attachments/Pasted%20image%2020240906165915.png)

> 注：`lame-ttl` 在 9.11.4中默认值为600s，在 9.18.3版本中，默认中为0，且无法配置为其他的非0的值。这个是因为 lame server 的 缓存存在bug，非0值，可能导致  lame server 的 缓存无限制的增长，导致clinet查询经过reslover时查询时间过长进而超时。如下所示，通过设置 `lame-ttl = 0` 来规避该问题。

![](attachments/Pasted%20image%2020240703102516.png)


## 转发器的质量相关日志

### 背景

比如 存在多个转发器时，对于某个外网域名的解析超时。那么不知道是哪个转发器的质量不太好。可以通过查看日志，看到哪个转发器对于该域名的解析质量问题。

如下的日志配置：

查询错误日志 ：
```bash
category query-errors {query-errors_log; };

channel query-errors_log {
	file "/var/named/logs/query-errors.log" versions 5 size 20m;
	print-time yes;
	print-category yes;
	print-severity yes;
	severity dynamic;
};

# 注：severity 最好不要设置为 dynamic，设置为 xxx 可能更好呢。

```



```bash
category resolver { auth_servers_log; default_debug; };
category cname { auth_servers_log; default_debug; };
category delegation-only { auth_servers_log; default_debug; };
category lame-servers { auth_servers_log; default_debug; };
category edns-disabled { auth_servers_log; default_debug; };

# resolver 日志
channel auth_servers_log {
	file "/var/named/logs/auth_servers.log" versions 100 size 20m;
	print-time yes;
	print-category yes;
	print-severity yes;
	severity info;
};

# debug channel, 仅仅在 debug level 不为 0 时输出
channel default_debug {
	file "/var/named/logs/debug.log" versions 3 size 100m;
	print-time yes;
	print-category yes;
	print-severity yes;
	severity dynamic;
};
```


### 分析

错误日志如下所示：

![](attachments/Pasted%20image%2020240628121850.png)

如下所示，可以知道。对于 `qq.com`这个zone下的外网域名解析的错误，主要是 "119.29.29.29" 这个forwarder导致的。

![](attachments/Pasted%20image%2020240628122724.png)

```
另外：
   query-errors.log 中的 query failed (timed out) 和  query failed (SERVFAIL) 应该不一样。
   前者应该是向 forwarder 发送请求，但是没有在规定时间内得到响应。
   后者 应该是都没有向 forwarder 发送请求，比如由于限速直接给client回复 SERVFAIL 或者 向转发器请求超时后给client回复的 SERVFAIL 。
   
```

## 选择更优的 forwarder

### 问题
经常会遇到这种问题：
在 bind9 的配置文件中配置了 多个 forwarder，防止单点问题。常见的forwarder有 ：
```bash
223.5.5.5:   阿里的public pod;
119.29.29.29: 腾讯的public pod;
```
### 原因
但是不同的 forwarder 对于不同的域名其服务质量不一样。比如，在实际的线上环境中，发现 `119.29.29.29`的服务质量并不太好，比如，对于阿里系的域名的服务质量就不太好，也有可能是对于某个域名存在限速配置，导致服务质量不好。
目前看，223.5.5.5 和 223.6.6.6 对于dns请求还没有限速，后期就会有限速。

![](attachments/Pasted%20image%2020240906174501.png)


### 思路
对于这种问题的常见的解决思路：
1》不同的 forwarder 设置不同的优先级、权重
2》forwarder 设置 active-backup模式
3》针对特定的zone，设置特定的forwarder
4》某个forwarder对于某个zone查询失败，则设置标记，且存在一定的缓存时间
5》提前预取 + cache中过期的记录继续保存并返回
> 提前预取：比如某个cache 中的记录即将过期，还有1s就过期，此时收到一个client的查询，那么解析器继续向转发器发送请求，提前预取更新cache中的记录。
> 过期记录继续缓存：某个过期的记录可以继续返回给client，然后向forwarder发送请求更新。

注：在bind9的实现功能中，目前看着好像只有 3和5 有这样的 功能，其他的目前看着还没有该功能。
至于设置标记，应该就是lame-ttl 这个，但是看着新版本9.18及其以上，甚至当下最新的 9.21.0 也是强制设置 lame-ttl = 0.

### 解决

#### 转发区域
参见：下面的forward zone
#### 预取+老化记录缓存与返回
参考：stale cache 。


# 多个forwarder的选择策略

## `SRTT` DNS服务器选择算法
大家都知道BIND在作为递归服务器时在向权威DNS请求时会使用优选策略. 
无论是 多个`forwarder`的选择，还是某个域存在多个 NS服务器时NS服务器的选择，都使用 `SRTT` 算法。
下面介绍下这个优选策略。
![](attachments/Pasted%20image%2020240313160646.png)


BIND 使用了一种名为“平滑往返时间”（SRTT）的机制。基本上，它会选择响应最快的服务器，并优先使用该服务器。BIND 会定期查询其他服务器，以更新 SRTT 值，这样可以让服务器“赶上”，但也降低了较慢服务器成为主要转发器的机会。**这也意味着，无论如何，您的一小部分查询将会使用到最慢的服务器**。

如果某个服务器没有响应，BIND 会尝试另一个服务器，并且未响应服务器的 SRTT 值会被递增。


## BIND9.8及之前版本的SRTT策略
小编针对BIND9.8的SRTT计算过程描述如下：

(1)、首先BIND在第一次计算SRTT时为所有的NS记录一个初始化的值，赋值方法是：

```
isc_random_get(&r);
e->srtt = (r & 0x1f) + 1;
e->expires = 0;
```

注释：这个值为随机1-32us，由于这个值非常小，远小于正常的SRTT，因此可以认为在初始化的时候，所有的NS都会得到一个很小的近乎为零的SRTT，因此所有的NS都有机会去被第一次优选。

(2)、在所有的NS中选择SRTT最小的一个NS服务器发起解析请求，如得到应答则记录这次请求的RTT，并重新计算这个NS的SRTT，计算方法是：

```
new_srtt = (addr->entry->srtt / 10 * factor)+ (rtt / 10 * (10 - factor));
```

注释：这里的factor定义如下：

```
#define DNS_ADB_RTTADJDEFAULT           7       /*%< default scale */
#define DNS_ADB_RTTADJREPLACE           0       /*%< replace with our rtt */
#define DNS_ADB_RTTADJAGE               10      /*%< age this rtt */
```

因此，在正常收到应答的情况：

```
        factor = DNS_ADB_RTTADJDEFAULT;
```

所以在正常的请求中，factor的值为7，所以这个新的NS的SRTT计算方法如下，也就是说这次请求的RTT在新的SRTT值的计算中权重占30%：`old_srtt * 0.7 + curr_rtt * 0.3`。

(3)、在这次请求中计算了请求的NS的同时，还需要对其他的NS进行衰减计算，计算方法如下：

```
if (factor == DNS_ADB_RTTADJAGE)
     new_srtt = addr->entry->srtt * 98 / 100;
```

注释：即所有的SRTT赋值为原来的98%

(4)、如果本次NS请求以失败告终，即发出请求并没有得到应答的情况，这里就要对这个NS进行惩罚，计算方法如下：

```
INSIST(no_response);
     rtt = query->addrinfo->srtt + 200000;
     if (rtt > 10000000)
     rtt = 10000000;
```

注释：直接给SRTT加上200ms，且SRTT最大值不能超过10s

(5)、1800s后，所有的SRTT清零，重复以上的计算  
这个1800来自源码的宏定义:

```
#define ADB_ENTRY_WINDOW        1800    /*%< seconds */
```

## BIND9.9及以后版本的SRTT策略
1、首先BIND在第一次计算SRTT时为所有的NS记录一个初始化的值，用样的赋值方法，随机1-32us。

2、在所有的NS中选择SRTT最小的一个NS服务器发起解析请求，如得到应答则记录这次请求的RTT，并重新计算这个NS的SRTT，同样的计算方法`old_srtt * 0.7 + curr_rtt * 0.3`。

3、其他NS的计算方法如下：
```
if (addr->entry->lastage != now) {
       new_srtt = addr->entry->srtt;
       new_srtt <<= 9;
       new_srtt -= addr->entry->srtt;
       new_srtt >>= 9;
       addr->entry->lastage = now;
```

注释：大概值为“SRTT = ((SRTT<<9)-SRTT)>>9”，即赋值为原来的SRTT的511/512，大概99.8%，这是BIND9.9和之前版本在计算SRTT中的一个最重要的差别。

4、如果本次NS请求以失败告终，则惩罚方式如下：

```
INSIST(no_response);
rtt = query->addrinfo->srtt + 200000;
if (rtt > MAX_SINGLE_QUERY_TIMEOUT_US)
       rtt = MAX_SINGLE_QUERY_TIMEOUT_US;
```

5、如果本次NS请求以失败告终，则惩罚方式如下：

```
INSIST(no_response);
rtt = query->addrinfo->srtt + 200000;
if (rtt > MAX_SINGLE_QUERY_TIMEOUT_US)
       rtt = MAX_SINGLE_QUERY_TIMEOUT_US;
```

注释：这里MAX_SINGLE_QUERY_TIMEOUT_US为宏定义，定义为

```
#define MAX_SINGLE_QUERY_TIMEOUT 9U
#define MAX_SINGLE_QUERY_TIMEOUT_US (MAX_SINGLE_QUERY_TIMEOUT*US_PER_SEC)
```

共9s，也就是SRTT的最大值降低了1s。值得说明的是，在BIND9.11中，这里的惩罚逻辑又有了变化，计算方法如下：

```
INSIST(no_response);
isc_random_get(&value);
if (query->addrinfo->srtt > 800000)
       mask = 0x3fff;
else if (query->addrinfo->srtt > 400000)
       mask = 0x7fff;
else if (query->addrinfo->srtt > 200000)
       mask = 0xffff;
else if (query->addrinfo->srtt > 100000)
       mask = 0x1ffff;
else if (query->addrinfo->srtt > 50000)
       mask = 0x3ffff;
else if (query->addrinfo->srtt > 25000)
       mask = 0x7ffff;
else
       mask = 0xfffff;
……
rtt = query->addrinfo->srtt + (value & mask);
```

注释：这里面根据当前SRTT值的不同，重新定义了一个随机数，而且是如果当前值的SRTT越小则惩罚的度量越大。


6、同样的1800s后，所有的SRTT清零，重复以上的计算SRTT策略&DNS解析质量。所以BIND的SRTT整个过程如下：
![](attachments/Pasted%20image%2020240313154955.png)

## 查看
SRTT的查看，如下所示：
![](attachments/Pasted%20image%2020240313151141.png)

## 小结
SRTT从设计上来说即兼顾了DNS异常依赖的优选以及容灾措施，在所有NS的存活的情况下能够保持绝大部分的递归请求可以优选最好的NS，同时在个别NS挂掉的情况下又能容灾切换至其他的NS。

同时，根据BIND版本演进中的衰减/惩罚机制变化来看， BIND在保障容灾的前提下尽可能更加选择优选（衰减策略从原来BIND9.8版本的98%变更至BIND9.9版本的99.8%），因此对于被优选NS的质量也提出了更高要求。

在此小编假设一种场景，对于BIND9.11版本的递归来讲如果一直优选的那个NS因为异常原因发生了丢包从而被递归惩罚，将使用更长的时间和次数来为这个NS进行衰减，从而有更长的时间/更多的递归次数不能被优选（比如一个原本20ms的NS因为一次丢包导致SRTT增加至220ms，那么需要2300次的衰减/或者等1800s过期才能使SRTT重新恢复至20ms），这对于递归的性能有本质上的影响。

# 配置
当设置了`forwarders`转发器后，所有非本域的和在缓存中无法找到的域名查询都将转发到设置的`DNS`转发器上，由DNS转发器来完成解析工作并做缓存。

在`/etc/named.conf`中配置，可以在`options`中做全局配置，**在`zone`语句中可以为特定`zone`设置特定的`forwarder`**。

## 全局配置

```bash
options {  
            forwarders { 192.168.10.35; 192.168.10.36; };  
            forward first | only;
};
```

介绍：
```bash
# forward first: 优先使用forwarders DNS服务器做域名解析，如果查询不到本机再做迭代查询。
# forward only: 只使用forwarders DNS服务器做域名解析，如果查询不到则返回DNS客户端查询失败。
# forwarders：全局配置的转发器
```

## 转发区 forward zone

![](attachments/Pasted%20image%2020240905115231.png)

type为 forward 的 zone 来改变options中配置的 全局转发forwarder 的行为（比如，从“优先转发（forward first）”更改为“仅转发（forward only）”，或反之）。

**针对特定域名配置的转发器**：
```bash
在 某个view的 zone配置中，设置如下：

zone <string> [ <class> ] {
		type forward;
		forward ( first | only );
		forwarders [ port <integer> ] [ tls <string> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ tls <string> ]; ... };
	};
```

在BIND8.2以后引入了一个新的特性：**转发区（forward zone）**，它允许把DNS配置成只有查找特定域名的时候才使用转发器。

（ BIND 9从9.1.0才开始有转发区功能 ）。
例如，你可以使你的服务器将所有对 kevin.cn 结尾的域名查询都转发给 kevin.cn 的两台名字服务器：
```bash
zone "kevin.cn" {  
          type forward;  
          forwarders { 110.50.80.208; 110.50.80.209; };  
};
```


# 参考
```bash
# “SRTT” DNS服务器选择算法介绍
https://alidns.com/articles/6018321800a44d0e45e90d70
```