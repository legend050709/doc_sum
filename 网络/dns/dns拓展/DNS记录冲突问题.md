```table-of-contents
```
# 记录冲突问题
在进行递归解析查询时，各记录类型之间是有优先级的，所以在主机记录相同、解析线路相同的情况下，有几种记录类型不能共存使用，否则会给用户造成配置风险，导致业务不可用的情况发生。

在DNS解析中，CNAME记录与其他记录往往是互斥的。最常见的是CNAME记录和MX记录的互斥。例如我们在`http://example.com`部署官网，通过CNAME解析到后端网关的IP地址。但是`http://example.com`往往也是我们的邮件地址，需要添加MX解析记录和SPF-TXT记录。如果有CNAME记录的存在，可能会导致他们失效（有时候也不会，要看实际访问的主机是否缓存了CNAME记录）。

# 记录冲突的规则
## 主机记录为@
在主机记录相同、解析线路相同的情况下，这几种不同类型的解析记录不能共存：
1、❌：冲突，在相同的主机记录情况下，同一条解析线路下，该两种类型的解析记录不允许共存。如：已经设置了 `dnswork.top` 的 A 记录，则不允许再设置 `dnswork.top` 的 CNAME 记录；

2、✅：不冲突，在相同的主机记录情况下，同一条解析线路下，该两种类型的解析记录可以共存。如：已经设置了 `dnswork.top` 的 A 记录，则还可以再设置 `dnswork.top` 的 MX 记录；

3、↔️：可重复，指在同一类型下，同一条线路下，可设置相同的多条记录值。如：已经设置了 `dnswork.top` 的 A 记录，还可以继续再设置 `dnswork.top` 的 A 记录。

![](attachments/Pasted%20image%2020240122102606.png)

## 主机记录为非@
在主机记录相同、解析线路相同的情况下，这几种不同类型的解析记录不能共存：
1、❌：冲突，在相同的主机记录情况下，同一条解析线路下，该两种类型的解析记录不允许共存。如：已经设置了 `www.dnswork.top` 的 A 记录，则不允许再设置 `www.dnswork.top` 的 CNAME 记录；

2、✅：不冲突，在相同的主机记录情况下，同一条解析线路下，该两种类型的解析记录可以共存。如：已经设置了 `www.dnswork.top` 的 A 记录，则还可以再设置 `www.dnswork.top` 的 MX 记录；

3、↔️：可重复，指在同一类型下，同一条线路下，可设置相同的多条记录值。如：已经设置了 `www.dnswork.top` 的 A 记录，还可以继续再设置 `www.dnswork.top` 的 A 记录。

![](attachments/Pasted%20image%2020240122102834.png)

# cname和其他记录冲突
## 冲突原因
CNAME 记录和 MX、TXT 记录冲突的根本原因在于 CNAME (Canonical NAME) 记录的特殊性。根据 `RFC 1034` 的规定，根域名不能设置 CNAME 记录，这是由 DNS 服务本身的固有限制决定的。或许你可以在一些 DNS 服务商那里为根域名添加 CNAME 记录，但这些都是不符合 DNS 规范的。如果要将根域名设置为另一个域名的别名，需要设置 ALIAS 记录。

> 根域名：即zone文件对应的域名，也是`@`代表的域名。

## Cname冲突的记录类型
如果根域名设置了 CNAME 记录，会和其他所有的记录相冲突，而最常见的冲突情形就是 MX 记录。对于同一个根域名，CNAME 记录和 A 记录、NS 记录、SOA 记录、TXT 记录等都会冲突。

我们以同时在根域名设置 CNAME 记录和 MX 记录为例。向该域名的域名邮箱发信且使用 DNS 寻址时，如果先寻到了 CNAME 记录，就无法再获取到该域名对应的 MX 记录。这就会导致使用该域名搭建的域名邮箱在收件时会经常丢信漏信。同时，CNAME 记录不仅与 MX 记录冲突，也会与 TXT 记录冲突，这就会导致为根域名设置的 SPF-TXT 记录无法生效，因此发信时更容易进垃圾箱。

## 冲突范例
假设为dnswork.top配置如下两条记录
![](attachments/Pasted%20image%2020240122142012.png)

按照RFC标准协议CNAME优先级较高。
```text
If a CNAME RR is present at a node, no other data should be present; this ensures that the data for a canonical name and its aliases cannot be different.


就是说如果CNAME资源记录出现在一个域名中，为了确保不会出现不同的解析结果，这个域名下将不再接受其他记录值。
```
所以在解析请求过程中，会优先返回CNAME解析记录结果，这样设置的结果导致用户无法请求到MX记录，直接对客户的邮箱业务造成使用影响。所以对于这类情况，云解析DNS会通过**记录冲突**的提示方式，来帮助用户避免这种配置风险。



## Cname记录冲突的解决
### 删除其中的一条解析记录
既然发生冲突了，那么最简单的方式就是二选一，保留一条，删除一条，这样就可以恢复正常。

如果实在不愿意删除，那么可以尝试把其中一条 CNAME 的解析记录更换为 A 记录，指定到一个 IP 地址上。这样会有一个隐患发生，那就是当 IP 地址更换了以后，你的解析会中断，需要手动变更 A 记录才能恢复。不是很推荐这种方法。

### 使用更现代的ALIAS记录
使用 ALIAS 记录代替 CNAME 记录是目前国际上最主流的设置办法了，它能起到与 CNAME 记录完全一样的效果，又不会和其他记录产生冲突。

ALIAS 记录，又称`CNAME Flattening` 记录，中文为“别名”记录，是一种 CNAME 记录的替代型记录。它能够起到和 CNAME 记录完全一样的效果，即将一个域名设置为另一个域名的别名，而唯一的差别就是 ALIAS 记录不会与其他记录发生冲突。

因此，我们只需要在 DNS 服务商那里为根域名设置 ALIAS 或者 `CNAME Flattening` 记录就可以了，它的设置方法与 CNAME 记录完全相同。
通过设置 ALIAS 记录，我们就能够完美解决网站根域名的 CDN 接入与域名邮箱共存问题。如果您的 DNS 服务商目前不支持 ALIAS 记录，您可以使用市面上很多免费的 DNS 服务，比如 `Cloudflare`、`he.net`、`dnsimple.com`、`Route53` (这个不免费)、`cloudns.net` 等等。这些 DNS 服务商都支持设置 ALIAS 记录。大部分国际域名注册商，比如 `Godaddy`、`Porkbun`、`Namesilo`、`Namecheap`、`Gandi`、`Google Domains` 等等，也都支持设置 ALIAS 记录。


### 使用二级域名进行解析
刚才也说到了，一个域名节点只能有一个 CNAME 的解析记录，那么就可以启用二级域名，这样就不是同一个域名节点了。
大致操作方法如下：
![](attachments/Pasted%20image%2020240122142850.png)

一般来说为根域名设置 CNAME 记录的情况都是由于网站需要接入 CDN。如果您可以接受网站采用 `www.example.com` 这样的网址而不是`example.com`，那么您完全可以使用 `www.example.com` 域名接入 CDN。由于 `www.example.com` 不是根域名了，因此它的 CNAME 记录不会和根域名的 MX、TXT 记录冲突，这样就解决了网站的 CDN 接入与域名邮箱共存的问题。
> 这种方法的有点在于最为简单，但缺点是必须使用二级域名。

### 使用 A 记录
如果您无法接受网站采用 二级域名(如：`www.` 域名），那么您也可以将根域名采用 A 记录的方式接入 CDN。
使用 A 记录时，您还可以自行设定线路，或者设置轮询。根域名的 A 记录不会和 MX 记录冲突，这样就解决了网站的 CDN 接入与域名邮箱共存的问题。

一般来说，这种情况比较适用于网站使用自行搭建的 CDN 系统，因为商用 CDN 系统的 IP 地址有时会发生变动，造成 A 记录解析失效。

### 使用隐式URL进行解析
在不同的域名服务商可能有不同的叫法，比如阿里云叫 显示/隐式 URL，CloudXNS的叫做 LINK 记录。

现在我们有域名`example.com`
![](attachments/Pasted%20image%2020240122154650.png)
需要将其以CNAME解析到`cn.to.com`,同时需要添加MX记录且主机记录为@，此时CNAME会与MX冲突。

通过配置隐性URL解决冲突:
通过二级域名`www.example.com`以CNAME解析到`cn.to.com`。
![](attachments/Pasted%20image%2020240122154807.png)

添加隐性URL记录，主机记录为@ ，记录值为`www.example.com`。
![](attachments/Pasted%20image%2020240122154838.png)

此时访问`example.com`都会转发到`www.example.com`，而`www.example.com`又以CNAME解析到`cn.to.com`。  这样CNAME就没有用到`@`，MX就可以使用@作为主机记录。
![](attachments/Pasted%20image%2020240122154951.png)

# 其他
## URL转发
URL转发，就是通过在服务器的设置，将一个域名指向另外一个已经存在的站点。

举个实例，小米的网站之前域名为`http://xiaomi.com`，2014年小米公司启动了全新的域名`mi`，原有域名`xiaomi`已跳转至`mi`。小米科技CEO雷军表示，新域名有利于小米国际化战略。
小米就是通过设置URL转发实现跳转的，当用户访问`http://www.xiaomi.com`，浏览器地址栏里将显示的是`http://www.mi.com`。

### 分类
URL转发分为两种：隐性URL转发、显性URL转发。
#### 隐性URL转发
隐性转发：效果为浏览器地址栏输入`http://a.com` 回车，打开网站内容是目标地址 `http://www.quansucloud.com/` 的网站内容，但地址栏显示当前地址`http://a.com` 。

```text
注意：
(1) 目标地址不允许被嵌套时，则不能使用隐性转发（例如 QQ 空间，不能使用隐性转发）。 
(2) 目标地址不支持添加 IP 地址 或 IP 地址 + 端口号 转发方式。
	URL 转发记录转发前，地址仅支持 HTTP、不支持 HTTPS；
	转发后地址支持 HTTP 及 HTTPS ；
(3) 添加 URL 转发记录时，转发后域名需在工信部完成备案（任意接入商）。
```

#### 显性URL转发
使用网宿dns云解析+301 重定向技术，效果为浏览器地址栏输入`http://a.com` 回车，打开网站内容是目标地址 `http://www.quansucloud.com/`的网站内容，且地址栏显示目标地址 `http://www.quansucloud.com/`。

### 访问流程
（1）用户在浏览器中输入`http://abc.com`

（2）浏览器做DNS解析,返回`nginx`代理服务器地址，浏览器访问`nginx`代理服务器上的`http://www.quansucloud.com`

（2.1 ）显性URL转发：客户端收到云解析服务器返回的信息，浏览器直接跳转到`www.quansucloud.com`;

（2.2） 隐性URL转发：网宿云解析中转收到隐性转发请求，发起代理访问，直接代理访问目标源站资源，并将访问的结果返回给客户端。

### URL转发的限制
URL转发的功能在国内各域名注册商被列为“违禁”功能之一。
根据相关规定，本功能受限使用。URL转发功能其实并不神秘，它的最基本用途是将新域名和已有网站进行关联。被禁的原因，是因为许多域名注册者将这个功能用于转发至非法网站，大量的“高仿域名”转发至非法网站，从而榨取利润。

使用此功能有两个途径：
第一个途径是在国外域名注册商注册，管理的域名可以实现URL转发功能；
第二个可以将域名托管（或更改NX记录）至支持转发功能的域名管理系统。

# 参考
```c
# 解析记录冲突规则
https://help.aliyun.com/document_detail/39787.html


# 也来谈谈关于 CNAME 和 MX 冲突的一些事
https://blog.skk.moe/post/about-cname-and-mx-conflicts/
```