```table-of-contents
```
# 介绍
**刷新 DNS 缓存：是指清除缓存中的所有 IP 地址和 DNS 记录**。
这有助于解决安全、网络连接和其他问题。
例如，当我们浏览器的地址栏中首次输入 `https://www.sysgeek.cn/` 时，浏览器必须向 DNS 服务器询问该网站的 IP 位置。一旦获取了这些信息，浏览器就可以将其存储在本地缓存中。当下一次再输入该网址时，浏览器将首先在本地缓存中查找其 DNS 信息，以便更快地访问该网站。

问题在于，有时可能会缓存不安全的 IP 地址或已经失效的 IP 结果，这时就需要将其删除。DNS 缓存还可能影响您连接到 Internet 的能力或引起其他问题。无论出于什么原因，所有主要操作系统都允许您强制清除此缓存的过程，也就是「刷新 DNS 缓存」。

重要的是要了解，DNS 缓存条目也会定期自动清除，无需干预。这是因为 DNS 缓存除了保存所有与识别和查询域名相关的信息外，还保存了一个称为 TTL（生存时间）的值。
![](../attachments/Pasted%20image%2020240313191919.png)
TTL 定义了 DNS 记录「保持有效」的时间段（以秒为单位）。在此期间，所有查询请求将从本地缓存中得到回答，而无需再次查询 DNS 服务器。一旦 TTL 到期，该条目将自动从缓存中删除。
**有时我们要强制刷新 DNS，而不是等待所有条目 TTL 自动到期。**

# DNS的缓存机制

## 缓存的对象


如果dns服务器上存在某个权威记录，那么在这个DNS服务器上就不存在这个记录的缓存，这些权威记录都是存储在DB数据库中，而不是缓存中。
不存在某个记录的时候，才会向转发器发起请求，获取响应之后，缓存下来。

> 注：缓存和权威记录是2个概念，缓存针对的是本地没有的权威记录。


## 缓存响应的TTL

DNS缓存中的记录的TTL是一直在变少的，直到超时，才会再次向权威服务器发起请求。
因此，DNS服务器中存在缓存时，其响应给Client的记录的TTL也是一直在变小的。

如下所示，2次查询时，响应是从缓存中获取的。Cname的TTL为1049，A记录的TTL为207;
![](attachments/Pasted%20image%2020240624194859.png)

10s之后的4次查询，响应也是从缓存中获取的。Cname的TTL为1039，A记录的TTL为197;
![](attachments/Pasted%20image%2020240624195027.png)


查看named的缓存文件，也可以看到 缓存的TTL是不断变少的，如下所示；
第一次查询，没有缓存，会向转发器发起请求，获得响应之后，进行缓存；后续的查询，缓存中存在，响应较快。
![](attachments/Pasted%20image%2020240624195810.png)

## 缓存中的 cname和A的TTL不一致时

如果缓存中一个域名的Cname和Cname的A记录的TTL不一致时。
比如：cname的缓存时间较短，那么缓存中的cname超时之后，再次发起请求，查询不到Cname，就会再次向权威发起请求，不用等待A记录超时才向权威发起请求。


## 指定域名(qtype, qname)带有不同TTL值的记录的返回结果

配置如下所示：
```
# cat 20_ip_of_domain.out
server 127.0.0.1
key hmac-sha256:default_key xxxxx
zone internal
update add a.b.t66666rr.dhb.yyyy.internal 10000 A 2.2.2.11
update add a.b.t66666rr.dhb.yyyy.internal 3000 A 2.2.2.33
update add a.b.t66666rr.dhb.yyyy.internal 4000 A 2.2.2.44
update add a.b.t66666rr.dhb.yyyy.internal 500 A 2.2.2.55
update add a.b.t66666rr.dhb.yyyy.internal 60 A 2.2.2.66
update add a.b.t66666rr.dhb.yyyy.internal 10000 AAAA 2002::2:2:2:2
send
quit
```

```bash
# nsupdate < 20_ip_of_domain.out

# dig @127.0.0.1 a.b.t66666rr.dhb.yyyy.internal -y hmac-sha256:default_key:"xxxxx"
...
;; ANSWER SECTION:
a.b.t66666rr.dhb.yyyy.internal. 60 IN A	2.2.2.55
a.b.t66666rr.dhb.yyyy.internal. 60 IN A	2.2.2.66
a.b.t66666rr.dhb.yyyy.internal. 60 IN A	2.2.2.44
a.b.t66666rr.dhb.yyyy.internal. 60 IN A	2.2.2.33
a.b.t66666rr.dhb.yyyy.internal. 60 IN A	2.2.2.11
...
```
如上所示，并不是每个记录都有一个自己的TTL，**==相同的域名(qtype, qname)的多个A记录响应时存在相同的TTL值==**。

**原因解析**：

正常情况下，一个域名存在多个A记录，每次返回的时候多个A记录的顺序是  round-robin，Clinet一般选择的是第一个A记录，这样可以做到服务在多个A记录上的负载均衡。

多个A记录，返回相同的TTL，这样可以确保在互联网上的某个解析 DNS 服务器（reslover dns）上，多个记录同时被清除缓存，并且在下次请求该 A 记录时可以重新获取多个记录。

如果不这样做，那么较短的记录 TTL 会先被清除，较长的 TTL 记录则会留在缓存中, 这样一来，你在远方的服务就失去了冗余，也无法做到多个IP的负载均衡，这最终会导致不稳定的行为，可能很难进行故障排除。

![](attachments/Pasted%20image%2020240913111701.png)



# 为什么要刷新 DNS 缓存
有几个主要原因可能会需要清空 DNS 缓存，这些原因可能与安全、技术问题或数据隐私有关。
## 防止 DNS 欺骗
DNS 欺骗，也称为 DNS 缓存投毒，是一种攻击方式。
恶意攻击者通过访问 DNS 缓存并篡改其中的信息，将您重定向到错误的网站。有时，他们会将您重定向到钓鱼网站，以便窃取敏感信息，例如网银登录信息。建议对 DNS 缓存定期清除，以防止此类攻击。

![](../attachments/Pasted%20image%2020240313192533.png)

##  遇到 404 错误

假设已经缓存了一个网站的 DNS 信息，但该网站已经更换了新的 IP 地址。在这种情况下，计算机上的 DNS 信息可能不会立即更新，导致尝试访问时会看到 404 错误或旧版本的网站。尽管这些信息最终会在 DNS 缓存中更新，但也可以不用等待，随时都可以手动清除 DNS 缓存。

## 保护访问隐私

当您想保护访问隐私时，通常会想到 cookie，但 DNS 缓存同样会泄露访问历史记录。因为 DNS 缓存像一个虚拟地址簿一样存储您经常访问的网站信息。为了避免数据收集者或网络上的不良行为者获取这些信息，定期刷新 DNS 缓存是一种好的习惯。

## 无法访问网站
如果无法访问某个网站，应首先尝试其他步骤，例如清除 Web 浏览器的临时文件和 cookies，调整浏览器设置，以关闭弹窗拦截功能并允许网站保存和读取 cookies。如果这些步骤都无效，还可以尝试清除 DNS 缓存记录并向服务器发出新的请求。



# 缓存的优缺点
## 缓存的优点
本地缓存DNS解析信息，提供解析速度。这消除了对远程 DNS 服务器重复查询的需要，并允许你的 OS 或浏览器快速解析网站的 URL。
DNS服务挂了也没有问题，在缓存服务时间范围内，解析依旧正常。

注：此中的本地其实是 DNS服务器，有些Server将自身作为DNS服务器。

## 缓存的缺点
DNS解析信息的需要传递到Client会滞后，如域名解析更改需要手动刷新缓存。对于依赖DNS切换的服务，建议不要开启DNS缓存。
```bash
You've changed your DNS provider to AdGuard DNS. If the user has changed their DNS, it may take some time to see the result because of the cache.

You regularly get a 404 error. 
For example, the website has been transferred to another server, and its IP address has changed. To make the browser open the website from the new IP address, you need to remove the cached IP from the DNS cache.
```


刷新dns缓存非常简单，任何时候都以进行。但是不同的系统，Windows、Mac OS和Linux上的方法是不一样的。
![](../attachments/Pasted%20image%2020231017173054.png)

# 特性
## **不存在刷新DNS缓存的超时时间**
每次都是
即：Client进行某个域名的DNS查询，如果Client本地存在缓存，或者本地不存在缓存，在LocalDNS服务器上存在缓存，在缓存中查询到结果，然后进行响应，此时不会对DNS缓存的超时时间进行刷新。


# 刷新缓存
此中的刷新DNS缓存，不是指的是刷新DNS缓存的超时时间。而是指的是，dns域名的记录发生了变更，进行DNS缓存的刷新。即，**记录发生了变更，能否在一定时间范围内保证缓存中的记录和权威服务器上的记录的一致**。

分为2个方面：
（1）刷新Client本地的DNS缓存。
（2）刷新dns缓存服务器（比如：本地的内网域名的dns服务器上缓存的外网域名的记录）上，缓存的非权威的记录。



# Client本地的dns缓存
## 系统级别刷新
### windows系统
要在 Windows 10 和 Windows 8 中清除 DNS 缓存，请执行以下步骤：

1. 在 Windows 搜索栏中键入 cmd 。
    
2. 右键单击 “命令提示符”，然后右击 “以管理员身份运行”。这将打开 “命令提示符” 窗口。
    
3. 在命令行上，键入以下行，然后按回车：
```c
ipconfig /flushdns
```

### macos系统
![](../attachments/Pasted%20image%2020240313175909.png)

### linux系统
在 Linux 上，除非已安装并运行诸如 `Systemd-Resolved`，`DNSMasq` 或 `Nscd` 之类的缓存服务，否则没有操作系统级 `DNS` 缓存。

#### nscd
`NSCD(name service cache daemon)`是一个缓存守护程序，是大多数基于RedHat的Linux操作系统的首选DNS缓存系统，比如`OpenSUSE Linux`、`CentOS`就用此方式来刷新DNS。

如果是清除 nscd 上的 Cache，可重新启动 nscd 服务来达成清除 DNS Cache 的效果：
```

1》 查看是否启动了nscd
systemctl status nscd.service

2》运行systemctl restart命令，它将重新加载服务并自动清除DNS缓存
systemctl restart nscd
```
如果 nscd 服务不存在，先安装 nscd，命令如下：
```c
sudo yum install nscd
```
#### systemd-resolve
大多数Linux用户正在运行一个内置`Systemd init`系统的操作系统，每个人都知道有一件事：`Systemd`使得操作系统级别的深度调整和维护比以往更加容易，清除DNS缓存时尤其如此。
`Systemd`以称为`systemd resolved`的方式处理DNS缓存，它是一个标准实用程序.


使用 `resolvectl` 命令刷新 DNS 缓存：
```c
# 查看是否启动 Systemd Resolved
systemctl status systemd-resolved.service

# Step 1. 查看 DNS 缓存状况  
sudo resolvectl --statistics  
  
# Step 2. 清除 DNS 缓存  
systemd-resolved --flush-caches

如果上诉方法不行，则 systemctl restart systemd-resolved.service。
  
# Step 3. 正在查看验证结果 (sysin)  
sudo resolvectl --statistics
```
如果 resolvectl 无法执行，先安装 systemd-resolved，命令如下：
```c
yum install systemd-resolved
```


# 浏览器级别刷新
大多数现代的 Web 浏览器都有一个内置的 DNS 客户端，以防止每次访问该网站时重复查询。

## 谷歌浏览器 Chrome
要清除 Google Chrome 的 DNS 缓存，请执行以下步骤：

1. 打开一个新标签，然后在地址栏输入 
```bash
chrome://net-internals/#dnsChrome

or

chrome://net-internals/#dns
```

2. 点击 “清除主机缓存” 按钮。

此方法适用于所有基于 Chrome 的浏览器，包括 Chromium，Vivaldi 和 Opera。

## 清除 Microsoft Edge 的 DNS 缓存
Microsoft Edge 在 2020 年切换到了 Chromium 内核以提高其稳定性和性能。由于它使用与 Chrome 相同的浏览器引擎，因此清除 DNS 缓存的步骤类似：

使用Ctrl + T快捷键打开一个新标签页——在地址栏中执行`edge://net-internals/#dns`打开清理页面。
# DNS缓存服务器上的缓存刷新
## dnsmasq
`Dnsmasq` 是轻量级的 DHCP 和 DNS 缓存名称服务器。它可以应用在内部网和 Internet 连接的时候的 IP 地址 NAT 转换，也可以用做小型网络的 DNS 服务。

### dnsmasq的缓存时长
在`dnsmasq`的配置文件（`dnsmasq.conf`）中，可以通过设置`cache-size`和`cache-min-ttl`参数来控制DNS缓存时间。

man dnsmasq 如下所示：
```bash
 --max-cache-ttl=<time>
        Set a maximum TTL value for entries in the cache. 
        the maximum time a DNS record is kept in the cache, regardless of the TTL provided by the authoritative server;

 --min-cache-ttl=<time>
        Extend short TTL values to the time given when caching them. Note that artificially extending TTL values is in general a bad idea, do not do it unless you have a good reason,  and  understand
        what you are doing.  Dnsmasq limits the value of this option to one hour, unless recompiled.
        
 -c, --cache-size=<cachesize>
        Set the size of dnsmasq's cache. The default is 150 names. Setting the cache size to zero disables caching.

 -N, --no-negcache
        Disable negative caching. Negative caching allows dnsmasq to remember "no such domain" answers from upstream nameservers and answer identical queries without forwarding them again.
```

`cache-size`参数控制缓存的最大DNS记录数量，`min-cache-ttl`参数控制缓存最小的TTL值。`max-cache-ttl` 控制缓存的最大ttl值。
例如：

```bash
cache-size=1000    // 缓存最多1000条DNS记录
min-cache-ttl=3600 // 缓存最小TTL值为1小时
```

### 操作查看dnsmasq
如果你的 DNS 服务器是用 dnsmasq 实现的，用下面这个命令:
```
1》 查看是否运行
systemctl status dnsmasq.service

2》可以使用systemctl restart命令立即清除DNSMasq的DNS缓存
systemctl restart dnsmasq
```
如果 dnsmasq 服务不存在，先安装 dnsmasq，命令如下：
```c
yum install dnsmasq
```

## bind之named
**设置**
在`BIND`的配置文件（`named.conf`）中，可以通过设置`options`中的`max-cache-ttl`和`max-ncache-ttl`参数来控制DNS缓存时间。
`max-cache-ttl`参数控制正常DNS记录的缓存时间，
`max-ncache-ttl`参数控制负向DNS记录（即不存在的域名）的缓存时间。
例如：
```bash
options {
    max-cache-ttl 86400;   // 缓存正常DNS记录1天
    max-ncache-ttl 3600;   // 缓存负向DNS记录1小时
};
```

**清除**
如果是清除 `BIND（Berkeley Internet Name Domain）` 服务器上的 CACHE，用这个命令:
```c
1> 查看DNS Cache
rndc dumpdb -cache

2> 清空DNS Cache:
rndc flush
or 
重启 named进程：systemctl restart named
or
重启 rndc: rndc restart

3> 清楚特定的域名缓存：
rndc flushname 2daygeek.com

```
如果 `rndc` 无法执行，先安装 bind，命令如下：
```c
yum install bind
```
## unboud
**unbound**：一个高性能的递归 DNS 解析器，可以替代系统默认的解析器。安装 unbound 并进行相应配置后，它会负责处理 DNS 查询并缓存结果。
unbound 使用 unbound-control 命令来管理 DNS 缓存：
```c
# 刷新所有缓存  
unbound-control flush all  
# 更多命令查看帮助  
unbound-control -h
```

如果 unbound-control 无法执行，先安装 unbound，命令如下：
```c
yum install unbound
```

# 参考
```c
# 如何清除与刷新 DNS 缓存，完全指南
https://www.sysgeek.cn/flush-dns-cache/

# Linux中如何清除DNS缓存
https://leokongwq.github.io/2017/08/30/linux-clean-dns-cache.html

# 如何有效的清除 DNS 缓存
https://www.hi-linux.com/posts/56208.html
```
