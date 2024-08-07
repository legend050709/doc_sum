```table-of-contents
```
# 背景

你是否有过这样的需求：在公司内部的DNS解析服务器上封禁一些域名、劫持一些外网域名，最好支持域名通配符设定，并且可以动态增删记录。

# 介绍

![](attachments/Pasted%20image%2020240118194459.png)

**响应策略区域(RPZ)**（Response policy zones）
```bash
A response policy zone (RPZ) is a mechanism to introduce a customized policy in Domain Name System servers, 
so that recursive resolvers return possibly modified results.
```

RPZ是一种在DNS服务器上自定义策略的机制，它可以**让递归解析返回被修改后的结果**。
在[RPZ官网](https://dnsrpz.info/)，更是提及到它的另一个名字——DNS Firewall。

目前RPZ机制仍作为[IETF RPZ草案](https://datatracker.ietf.org/doc/html/draft-vixie-dnsop-dns-rpz-00#section-4)存在，但RPZ官网显示其已被BIND 9、PowerDNS、Recursor 4+等多款DNS软件支持。

# RPZ的作用
RPZ的主要作用是用于保护用户免受互联网上已知恶意域名、IP、nameserver等的不良影响。例如：
- 劫持某个域名
- 丢弃指定域名的请求
- 指定域名解析到NXDOMAIN
- 禁用某个IP的DNS请求
- 封禁执行某个IP的域名解析

## 应用

1. 把部分做反射攻击的域名封掉。  
2. 把内部域名屏蔽掉,直接返回NXDOMAIN。  
3. 屏蔽部分暴力,黄色,诈骗网站。

## 范例

```bash
1. 劫持某个域名(劫持www.google.com)  
./client add www.google.com.rpz.zone. A 8.8.8.8

2. 丢弃某个域名的解析请求（丢弃对www.google.com的解析请求）  
./client add blog.gnuers.org.rpz.zone. CNAME rpz-drop.

3. 将某个域名解析到NXDOMAIN（www.google.com指向NXDOMAIN）  
./client add www.google.com.rpz.zone. CNAME .

4. 禁用掉某个IP的DNS请求(丢弃8.8.8.8/32的DNS 请求)  
./client add 32.8.8.8.8.rpz-client-ip.rpz.zone. CNAME rpz-drop.

5. 封禁某个NS上的域名解析 （将ns1.google.com.返回的DNS查询置NXDOMAIN或者指向127.0.0.1）  
./client add ns1.google.com.rpz-nsdname.rpz.zone. CNAME .  
./client add ns1.google.com.rpz-nsdname.rpz.zone. A 127.0.0.1

6. 封禁某个NSIP的域名解析（将NS IP 匹配8.8.8.8/32的DNS响应置位NXDOMAI或者劫持到127.0.0.1）  
./client add 32.8.8.8.8.rpz-nsip.rpz.zone. CNAME .  
或  
./client add 32.8.8.8.8.rpz-nsip.rpz.zone. A 127.0.0.1

7. 封禁指向某个IP的域名解析  
./client add 32.8.8.8.8.rpz-ip.rpz.zone. CNAME .  
或  
./client add 32.8.8.8.8.rpz-ip.rpz.zone. A 127.0.0.8
```

# 工作原理

RPZ工作原理的总体思路是：
可以为特定查询（或响应）创建策略(策略=匹配条件+动作)，然后将这些策略存储在DNS服务器上的特别`zone`中。
注：还可以通过将这些区域从DNS服务器传输到（另外的）DNS服务器来共享这些`zone`。

# RPZ策略
RPZ中的记录也是由`owner name`，type和 记录值（`rdata`）组成。

只是RPZ区域（即zone）不会用于接收用户发起的请求，只在用户发起的请求时会匹配RPZ定义的规则。RPZ规则中`owner name`用于定义触发器，`rdata`用于定义动作，即满足触发器的请求会按照策略执行相应的动作。

![](attachments/Pasted%20image%2020240119111959.png)

配置如下所示：
```bash
response-policy {
    zone zone_name
    [ policy (given | disabled | passthru | drop |
              nxdomain | nodata | cname domain) ]
    [ recursive-only yes_or_no ]
    [ max-policy-ttl number ]
    ; [...]
} [ recursive-only yes_or_no ]
  [ max-policy-ttl number ]
  [ break-dnssec yes_or_no ]
  [ min-ns-dots number ]
  [ qname-wait-recurse yes_or_no ]
  [ automatic-interface-scan yes_or_no ]
;
```

## RPZ匹配条件



![](attachments/Pasted%20image%2020240119113603.png)

**存在5种匹配条件**：
- 发起请求的client的ip为匹配条件进行劫持
- 以要查询的域名为匹配条件，进行劫持
- 以外网应答的应答记录中的的IP地址为匹配条件，进行劫持
- 以DNS资源记录的数据路径中出现的nameserver为匹配条件，进行劫持
- 以DNS资源记录的数据路径中出现的nameserver的IP为匹配条件，进行劫持


### Client IP Address
owner name：以.rpz-client-ip结尾，用于匹配发起请求的客户端的IP地址，双栈均可支持；即以 发起请求的client的ip为匹配条件进行劫持。

定义`IPv4`地址段的客户端时，如`B1.B2.B3.B4/prefix`，在RPZ策略规则的书写规则中应当写成`prefix.B4.B3.B2.B1`；
定义`IPv6`地址段的客户端时，如`B1:B2:B3:B4:B5:B6:B7:B8/prefix`，在RPZ策略规则的书写格式中应当写成`prefix.B8.B7.B6.B5.B4.B3.B2.B1`。
如同`IPv6`地址可以使用双冒号缩写，此处的`IPv6`地址可以相应的使用`zz`替代。如`2001::6:180/128`可写成`128.180.6.zz.2001`。

![](attachments/Pasted%20image%2020240118153157.png)

### QDNAME
owner name ：正常域名，可带通配符。用于匹配发起请求包或应答包中请求域名字段的域名。即以要查询的域名为匹配条件，进行劫持。

![](attachments/Pasted%20image%2020240118153331.png)


### Response IP Address
owner name：以.rpz-ip结尾。用于匹配应答记录中的的IP地址，双栈均可支持。
IP地址段的编写格式与`Client IP Address`中描述的一致。

![](attachments/Pasted%20image%2020240118153410.png)

`www.baidu.com`正常情况下会返回如下记录
![](attachments/Pasted%20image%2020240118153427.png)
匹配到180.101.49.0/24IP段后，`www.baidu.com`将会返回`NXDOMAIN`。

### NSDNAME
owner name ：以.rpz-nsdname结尾。用于匹配应答记录中NS的名字，可以在回答部分，授权部分。即：匹配DNS资源记录的数据路径中出现的nameserver。
> 需要注意的是，nameserver触发条件不一定会如预期进行，因为DNS Server不总是会以Trace的方式从`root NS`开始查询。

![](attachments/Pasted%20image%2020240118153604.png)

### NSIP
owner name ：以.rpz-nsip结尾。用于匹配应答记录中NS名字对应的IP地址（A或AAAA的数据），可以在回答部分，附加部分。

![](attachments/Pasted%20image%2020240118153637.png)

## RPZ动作

![](attachments/Pasted%20image%2020240119112149.png)

**RPZ规则存在6种动作**：


### NXDOMAIN

==data为：`.` ，动作为回复**NXDOMAIN类型**的应答==。

当RPZ存在一个域名CNAME记录指向根域(.)的话，recursor不会向上游DNS进行查询，直接返回NXDOMAIN，即域名不存在。

![](attachments/Pasted%20image%2020240118154305.png)

### NODATA

==data为：`*.` 。动作为回复NODATA类型（**`rcode`为`noerror`但是`answer`个数为0**）的应答==。

当RPZ存在一个域名CNAME记录指向通配符顶级域名，recursor不会向上游DNS进行查询，直接返回NODATA，即空返回。

![](attachments/Pasted%20image%2020240118154401.png)

### PASSTHRU

rdata为：`rpz-passthru.` 。动作为透传，即RPZ白名单，不走RPZ策略规则。

`ok.example.com` CNAME记录指向`rpz-passthru.`，会正常返回查询结果，尽管`example.com`子域下其他域名会返回`NXDOMAIN`。

![](attachments/Pasted%20image%2020240118154504.png)


### DROP
radata为：`rpz-drop.`。动作为丢弃，即不做应答。

当RPZ存在一个域名CNAME指向`rpz-drop.`，recursor会直接丢弃查询请求，查询客户端无法得到正确响应。
![](attachments/Pasted%20image%2020240118154539.png)

### TCP-ACTION
rdata为：`rpz-tcp-only.` 。动作为将应答中的TC标志置1，强制用户发起TCP的DNS请求，用于攻击防护。
当RPZ存在一个域名CNAME指向`rpz-tcp-only.`，recursor会引导客户的重新以TCP协议再次发起域名查询。
![](attachments/Pasted%20image%2020240118154615.png)

### LOCAL DATA
rdata为：正常域名 。动作为回复指定预先配置好的解析结果。
![](attachments/Pasted%20image%2020240118154643.png)


## 其他

### rpz策略调用的相关日志

测试如下所示：

![](attachments/Pasted%20image%2020240626103832.png)


rpz的日志如下：

测试用例中 RPZ的触发条件都是QNAME，==通过 以**QNAME+RPZ配置的zone的名称** 这个zone下的记录的动作（RPZ的action） 作为 QNAME 的结果==。

![](attachments/Pasted%20image%2020240626104017.png)



### NODATA 和 NXDOMAIN 的区别
`NODATA`意味着该域存在，但没有关于该域的信息与该域关联的指定类型（如A记录）。 如果域本身不存在，将会看到`NXDOMAIN`。
`NXDOMAIN（代表Rcode=3）`是 `Rcode`响应码的一种；
但是 `there isn’t an RCODE associated with NODATA.`, 即 `Nodata` 并不是`Rcode`响应码的一种；对于`Nodata`，意味着`rcode`为`noerror`但是`answer`个数为0。
```bash
dig represents NODATA by displaying NOERROR with an ANSWER of zero. So what does NOERROR with an ANSWER of 0 actually represent? It means one or more resource records exist for this domain but there isn’t a record matching the resource record _type_ (A, AAAA, MX, etc.).
```



### qname-wait-recurse 字段

#### 背景

rpz的触发条件是 query name时，在正常操作中，仅当任何查询的结果可用时（当查询过程完成时 ，无论成功或不成功），才会调用RPZ策略处理。

这样的 后果就是 存在出外网的流量，并且 client得到结果的时间较长。

#### 解决

rpz的触发条件是 **query name** 或者 **client-ip**时，需要设置==`qname-wait-recurse no`==；才能使得rpz策略的调用是在向外网查询之前进行触发。

> 注：RPZ的其他触发条件，想要让RPZ策略动作不迭代查询而提前调用，则无法通过`qname-wait-recurse no`实现。
> 比如：如果触发条件是响应的IP，NSIP，NSDNAME，则必须进行迭代查询，因为这些触发条件都是在迭代查询中得到的。而**query name** 和 **client-ip**作为触发条件，则不需要递归查询，就可以得到这些条件。

![](attachments/Pasted%20image%2020240119114340.png)


#### 范例
Before:
```
response-policy { zone "rpz"; };
```

After:
```
response-policy { zone "rpz"; } qname-wait-recurse no;
```


具体在view中的配置，如下所示：
```bash
view xxxx {
   ...
    response-policy {
        zone "yyyyy";
    } qname-wait-recurse no;   
};
```

**测试**
更改完，上诉配置之后， rndc reload 配置即可生效。经过测试，上诉配置在 9.11.4 以及 9.18.3上都是生效的。


### recursive-only 字段 

#### 背景

dig查询时，reslover收到的请求的RD(recursion desired)默认都是开启的。对于外网域名查询，然后reslover向外转发查询时，RD也都是默认开启的。
如下所示：

![](attachments/Pasted%20image%2020240626114009.png)

转发查询如下所示：

![](attachments/Pasted%20image%2020240626114055.png)

#### 介绍

![](attachments/Pasted%20image%2020240626105126.png)

`recursive-only` 为 Yes，即只有 DNS查询的 RD=1时，rpz策略才会调用。
`recursive-only` 为 No，则不关注  DNS查询的 RD是否为1，rpz策略都可以调用；





# RPZ配置

==RPZ的配置，可以为全局的，也可以是基于View的==。
其实本质上还是基于View的，因为 Zone其实也是基于View而存在的。

## RPZ的配置流程
RPZ的配置，分为3步：
- view中配置`RPZ zone`引导配置
- 配置rpz的zone对应的数据库文件配置
- 使能 RPZ

## 语法

![](attachments/Pasted%20image%2020240626104846.png)

语法如下：
```bash
response-policy { zone zone-name 
   [ policy (given|disabled|passthru|drop|nxdomain|nodata|tcp-only| cname domain-name)
   [ recursive-only yes_or_no ] 
   [ max-policy-ttl seconds ] 
   [ log yes_or_no]
   ; ... } 
   [ recursive-only yes_or_no ] 
   [ max-policy-ttl seconds ]
   [ break-dnssec yes_or_no ]
   [ min-ns-dots number ]
   [ qname-wait-recurse yes_or_no ]
   [ nsip-wait-recurse yes_or_no] ;


注：上诉的`...` 表示可以在 response-policy {} 中配置多个 zone。

比如：
response-policy {zone "dontlike" ; zone "likeless" policy passthru;} recursive-only yes;
```


范例如下：
```bash
# example
options {
	  ...
	  response-policy {
		zone "evenless" ;
	    zone "noway"; 
	  };
	  ...
};
...
zone "evenless" {
	  type master;
	  file "master/evenless.map";
		...
	  };
view "mordor" {
	...
	zone "noway" {
	  type slave;
	  file "slave/noway.map";
	  ...
	  };
};
```



## 实验测试

### 全局配置
**`/etc/bind/named.conf.options` 添加 `response-policy`**
```bash
options {
    ...
    response-policy {
        zone "rpz.local";
    };
    ...
}
```

**创建 `rpz zone`引导配置**
```bash
zone "rpz.local" {
    type master;
    file "rpz.local";
    allow-query { localhost; 192.0.2.0/24; 2001:db8:1::/64; };
    allow-transfer { none; };
};
```
验证 `/etc/named.conf` 文件的语法：`named-checkconf`；如果命令没有显示输出，则语法为正确的。


**创建 rpz日志(可选)**
```bash
channel rpzlog {
    file “/var/log/named/rpz.log” versions unlimited size 100m;
    print-time yes;
    print-category yes;
    print-severity yes;
    severity info;
};
category rpz { rpzlog; };
```

**创建`rpz zone`文件**
`/var/named/rpz.local` 文件
```bash
$TTL 10m
@ IN SOA ns1.example.com. hostmaster.example.com. (
                          2022070601 ; serial number
                          1h         ; refresh period
                          1m         ; retry period
                          3d         ; expire time
                          1m )       ; minimum TTL

                 IN NS    ns1.example.com.

example.org      IN CNAME .
*.example.org    IN CNAME .
example.net      IN CNAME rpz-drop.
*.example.net    IN CNAME rpz-drop.
```
- 将查询的 `NXDOMAIN` 错误返回给该域中的 `example.org` 和主机。
- 将查询丢弃至此域中的 `example.net` 和主机。

验证 `/var/named/rpz.local` 文件的语法：
```bash
# named-checkzone rpz.local /var/named/rpz.local
zone _rpz.local/IN_: loaded serial _2022070601_
OK
```

### view中配置
同上类似。

```bash
options {
     .....
     .....
}

acl internal {
    10.10.10.0/24;
};

acl guest { 
   10.10.20.0/24;
};

view "internal" {
  match-clients { internal; };

  //enable the response policy zone.
  response-policy {
     zone "rpz.local";
  };

  zone "rpz.local" {
    type master;
    file "/etc/bind/db.rpz.local";
    allow-query { localhost; };
    allow-transfer { 12.34.56.78; };
  };

};

view guest {
     match-clients { guest; };
     allow-recursion { any; };

    zone "." {
        type hint;
        file "/usr/share/dns/root.hints";
    };

    zone "localhost" {
        type master;
        file "/etc/bind/db.local";
    };

    .....
    .....
};

```

### 重新载入 BIND

```bash
systemctl reload named

注：如果在 change-root 环境中运行 BIND，请使用 `systemctl reload named-chroot` 命令来重新加载该服务。
```

### 验证

**验证**
尝试解析 `example.org` 中的主机，该主机在 RPZ 中配置，以返回 `NXDOMAIN` 错误：
```bash
# dig @localhost www.example.org
...
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 30286
...
```

注意：==尽管  zone 文件（/var/named/rpz.local）实际的 域名理解上 带有 
`rpz.local`， 比如，`www.example.org.rpz.local`, `example.net.rpz.local`；
但是client实际访问的时候，还是访问的是 `www.example.org`,  `example.net.rpz`
就可以实现了匹配，然后拦截替换==。



尝试解析 `example.net` 域中的主机，该域在 RPZ 中配置以丢弃查询：
```bash
# dig @localhost www.example.net
...
;; connection timed out; no servers could be reached
...
```







# 范例
## 范例一
(1) `/etc/named/named.conf`
![](attachments/Pasted%20image%2020240118154825.png)

(2) `rpz`日志
![](attachments/Pasted%20image%2020240118154834.png)

(3) `RPZ zone` 引导配置
![](attachments/Pasted%20image%2020240118154904.png)

(4) RPZ 区域主配置文件
```bahs
$TTL 10M
@         IN      SOA     kylin.zabbix.com.      dnsadmin.zabbix.com. (                        1   ;序列号                        1H  ;1小时后刷新                        5M  ;15分钟后重试                        7D  ;1星期后过期                        1D );否定缓存TTL为1天          IN      NS     kylin
www.jd.com IN A  192.168.100.161
*.google.com IN A  192.168.100.161
www.souhu.com   CNAME  www.baidu.com.
www.3399.com    CNAME  .
```

(5) dig 测试
![](attachments/Pasted%20image%2020240118154955.png)
![](attachments/Pasted%20image%2020240118154959.png)
![](attachments/Pasted%20image%2020240118155003.png)
![](attachments/Pasted%20image%2020240118155007.png)

(6) 查看日志
```bash
tail -n 10 /var/named/data/rpz.log
```


# 参考
```c
Bind 响应策略区域 Response Policy Zones (RPZ)
https://cloud.tencent.com/developer/article/2168926

# bind9的官方网站
https://bind9.readthedocs.io/en/latest/reference.html

# bind9 RPZ 的官方配置
https://www.zytrax.com/books/dns/ch7/rpz.html


# 腾讯云的DNS的文章大全
https://cloud.tencent.com/developer/tag/10707

# bind rpz使用注意事项
https://blog.gnuers.org/?p=1062
```