```table-of-contents
```
# 转发器介绍
当`bind9` 设置了`forwarders`转发器后，所有非本域的和在缓存中无法找到的域名查询都将转发到设置的DNS转发器上，由这台DNS来完成解析工作并做缓存，因此这台转发器的缓存中记录了丰富的域名信息。因而对非本域的查询，很可能转发器就可以在缓存中找到答案，避免了再次向外部发送查询，减少了流量。

# `SRTT` DNS服务器选择算法
大家都知道BIND在作为递归服务器时在向权威DNS请求时会使用优选策略. 
无论是 多个`forwarder`的选择，还是某个域存在多个 NS服务器时NS服务器的选择，都使用 `SRTT` 算法。
下面介绍下这个优选策略。
![](attachments/Pasted%20image%2020240313160646.png)



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

在`/etc/named.conf`中配置，可以在`options`中做全局配置，在`zone`语句中可以为特定`zone`设置特定的`forwarder`。

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

## 转发区

**针对特定域名配置的转发器**：
```bash
zone zone_name [class] {
    type forward;
    [ forward (only|first) ; ]
    [ forwarders { [ ip_addr [port ip_port] ; ... ] }; ]
    [ delegation-only yes_or_no ; ]
};
```
在BIND8.2以后引入了一个新的特性：**转发区（forward zone）**，它允许把DNS配置成只有查找特定域名的时候才使用转发器。（ BIND 9从9.1.0才开始有转发区功能 ）。
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