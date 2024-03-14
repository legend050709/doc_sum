```table-of-contents
```
# DNS介绍
![](attachments/Pasted%20image%2020240104113047.png)
DNS 的全称是 `Domain Name Systems`，DNS是一个分层级 （hierarchical ），分布式（decentralized）的网络数据库，完成主机名称和IP地址之间的相互映射。DNS 协议运行在 UDP 协议上，使用 53 端口。

与 HTTP、FTP 和 SMTP 一样，DNS 协议也是一种应用层的协议，DNS 采用 client/server 模式，DNS client 发出查询请求，DNS server 响应请求，并通过 UDP 协议来传输 DNS 报文。
![](attachments/Pasted%20image%2020240104121432.png)


DNS名称空间包含一个树状结构，如下所示：
![](attachments/Pasted%20image%2020240110160858.png)

除了提供主机名到 IP 地址的转换，DNS 还支持了下面几种重要的功能：
- `主机别名(host aliasing)`，有着复杂主机名的主机能够拥有一个或多个其他别名，比如说一台名为 relay1.west-coast.enterprise.com 的主机，同时会拥有 enterprise.com 和 [www.enterprise.com](http://www.enterprise.com/) 的两个主机别名，在这种情况下，relay1.west-coast.enterprise.com 也称为**规范主机名**，而主机别名要比规范主机名更加容易记忆。应用程序可以调用 DNS 来获得主机别名对应的规范主机名以及主机的 IP地址。
    
- `邮件服务器别名(mail server aliasing)`，同样的，电子邮件的应用程序也可以调用 DNS 对提供的主机名进行解析。
    
- `负载分配(load distribution)`，DNS 也用于冗余的服务器之间进行负载分配。繁忙的站点例如 .cn .com 被冗余分布在多台服务器上，每台服务器运行在不同的端系统，每个都有着不同的 IP 地址。由于这些冗余的 Web 服务器，一个 IP 地址集合因此与同一个规范主机名联系。DNS 数据库中存储着这些 IP 地址的集合。由于客户端每次都会发起 HTTP 请求，所以 DNS 就会在所有这些冗余的 Web 服务器之间循环分配了负载。

# DNS服务器
## 服务器分类
- **主服务器（Primary Authoritative Name Server）**：
负责至少解析一个域内（zone）的域名，维护所负责解析的域数据库（zone文件），**可对负责的域数据进行读写操作**
-  **辅服务器（Secondary Authoritative Name Server）**：
负责从主服务器或其他辅服务器中复制相关解析库，为主服务器缓解解析压力
- **缓存服务器（Caching Name Servers）**：
不负责域名解析，仅仅作为缓存，加速解析速度
- **转发服务器（Forwarding Name Servers）**：
发现非本机负责的请求后，不再向根发起请求，而是直接转发给指定的一台或多台服务器，自身并不保存查询

其他：
dns解析器（resolver）、dns权威服务器；
![](attachments/Pasted%20image%2020240304123409.png)

## DNS 解析器
一般指本地域名服务器(Local DNS Server)，它是DNS查找中的第一站，是负责处理发出初始请求的DNS服务器。
运行商ISP分配的DNS如电信的`114.114.114.114`和谷歌的`8.8.8.8`等都属于`DNS Resolver`。

解析器（resloving dns server）在不同的使用场景下常被称为各种令人困惑的名字，如缓存名称服务器(cache dns server)、递归名称服务器(recursive dns server)、转发解析器(forwarder dns server)等。
> 和 DNS 解析器相对应的 就是 权威服务器（Authoritative DNS server）。 权威服务器是实际持有并负责特定域资源记录的服务器，通常是解析器查找 IP 地址过程中的最后一步，拥有这些域的最终解释权。


## 主从服务器的关系
![](attachments/Pasted%20image%2020240104121221.png)

设置域名服务器时，服务器管理员可以选择将域名服务器指定为主服务器还是辅助服务器(也称为从服务器)。

主域名服务器负责维护一个区域的所有域名信息，是特定的所有信息的权威信息源，数据可以修改。主服务器直接从本地文件获取此信息。只能在主服务器上更改区域的 DNS 记录，然后主服务器才能更新辅助服务器。

辅助域名服务器作为主域名服务器的备份提供域名解析服务，可以缓解主域名服务器的压力负载。辅助域名服务器中的区域文件中的数据是从主域名服务器中复制过来的，无法自行修改。
其实就是主从的概念，各位应该也都比较熟悉了。

**主域名服务器用来写，辅助域名服务器用来读，提供负载均衡的能力，缓解主域名服务器的压力。**

# 域名
域名由一系列字母（a～z，**不区分大小写**）、数字（0～9）、连接符（`-`）以及点号(`.`)分隔符组成，总长度不大于255。分隔符隔出的每段相当于一个层次的域名，级别低的在左，级别高的在右，每段长度不大于63。
如域名`dailyupdate.wangwang.taobao.com`，三段域名分别为`dailyupdate`、`wangwang`、`taobao`、`com`，其中`com`的级别最高。

## 域名结构
**DNS 域**的本质是一种管理范围的划分，最大的域是根域 ，向下可以划分为顶级域 、二级域 、三级域 、四级域 等。相对应的域名是根域名 、顶级域名 、二级域名 、三级域名 等。不同等级的域名使用**点号**分隔，级别最低的域名写在最左边，而级别最高的域名写在最右边。

![](attachments/Pasted%20image%2020231225195047.png)

举个栗子：网站域名 `www.tsinghua.edu.cn`中，从右到左开始，cn 是顶级域名，代表中国，edu 是二级域名，代表教育机构，tsinghua 是三级域名，表示清华大学，www 则表示三级域名中的主机，并提供了 web 服务。
![](attachments/Pasted%20image%2020240104121647.png)

### 根域
​ 根（root）域就是`.`；它是由Inetnet名字注册授权机构管理，该机构把域名空间各部分的管理责任分配连接到Internet的各个组织 。

**域名空间**结构像是一棵倒过来的树，也叫做树形结构 。**根域名**就是树根（ root ），用点号 表示，往下是这棵树的各层枝叶。
![](attachments/Pasted%20image%2020240104122016.png)

### 一级域
根域名的下一层叫**顶级域名**。顶级域(TLD：Top Level Domain）又称一级域。

其中顶级域名分为：国家顶级域名（ccTLD，比如 .cn 和 .us）、通用顶级域名（gTLD，比如 .com 和 .net）、反向域名。
顶级域名由国际域名管理机构 ICANN（互联网名称与数字地址分配机构：Internet Corporation for Assigned Names and Numbers） 控制，它委托商业公司管理 gTLD，委托各国管理自己的国别域名。


|顶级域|示例|
|---|---|
|国家顶级域名|中国:cn， 美国:us，英国uk…|
|通用顶级域名|com公司企业，edu教育机构，gov政府部门，int国际组织，mil军事部门 ，net网络，org非盈利组织…|
|反向域名|arpa，用于PTR查询（IP地址转换为域名）|


### 二级域
顶级域名下面是**二级域名**。
二级域（SLD：Second Level Domain）注册到个人、组织或公司的名称。这些名称基于相应的顶级域，二级域下可以包括主机和子域。

![](attachments/Pasted%20image%2020240104122344.png)

二级域名下面是**三级域名**、**四级域名**等。**命名树上任何一个节点的域名就是从这个节点到最高层的域名串起来，中间以 “ . ” 分隔。**
![](attachments/Pasted%20image%2020240104122452.png)

在域名结构中，节点在所属域中的**主机名**标识可以相同 ，但是**域名必须不同**。比如：清华大学和新浪公司下都有一台主机的标识是 mail ，但是两者的域名却是不同的，前者为 **mail.tsinghua.edu.cn**，而后者为 **mail.sina.com.cn**。
![](attachments/Pasted%20image%2020240104122631.png)
## 域的分层授权
域是从上到下授权的，每一层都只负责自己的直辖下层，而不负责下下层。
例如根域给顶级域授权，顶级域给普通域授权，但是根域不会给普通域授权。和现实中的行政管理不一样，域的授权和管理绝对不会向下越级，因为它根本不知道下下级的域名是否存在。
## 子域（Sub Domain）
DNS 有层次结构。顶级域”.com”是根域”.”的子域，”baidu.com.”又是”.com.”的子域。
TLD 下面可以有多个域名。例如，com 下面有 google.com 和 ubuntu.com。
子域是相对父域来说的，指域名中的每一个段。各子域之间用小数点分隔开。放在域名最后的子域称为最高级子域，或称为一级域，在它前面的子域称为二级域。
```bash
比如：
www.nansan.sz.hunk.tech.

	hunk.tech为父域
	sz为hunk.tech的子域，为一级子域
	nansan为hunk.tech的子域，为二级子域
	www是主机
```

注：zone分为主机名zone和域名zone。其中，**主机名zone就不存在子域了。域名zone才可能存在子域**。

## 子域管理
### 子域的区域数据文件
**不管是不是子域，只要它是一个域，就必须要有dns服务器来负责该域的解析**。
域的灵魂在于其区域数据文件，只有区域数据文件中才提供了域所需的所有数据，要让dns服务器能够正常工作，域数据必须要完整且正确。
**完整的域数据至少要存储SOA记录，ns记录以及ns对应的a记录**。
（1）如果没有ns记录，则表示该域缺少dns服务器，这是不可能的。
（2）缺少ns对应的a记录则能知道dns服务器的存在以及其主机名，却找不到dns服务器，因为没有它的ip地址。
（3）缺少了soa记录后，它指定不了哪个dns服务器是主dns服务器，以及一系列的附加属性(序列号、各种重试缓存时间等)。更关键的是soa是起始授权机构，缺少了soa记录表明父域没有对该域授权，该域没有自主权，它只是父域下的一部分(可能只是主机名上多了一截，例如wuda.video.longshuai.com)，所有的解析工作都需要由父域来完成，只有有了soa记录，才表明父域授权了该域，该域可以享受起始授权的权利，也就是具有自主权，可以独立完成解析工作。子域，同样是域的概念，因此它的区域数据文件也一样要满足这些不是条件的条件。

### 子域在父域中的记录
再回顾下dns解析流程，客户端向dns服务器发送递归查询后，dns服务器会查询根域，根域会告诉dns服务器负责解析顶级域的服务器地址，dns服务器再向顶级域服务器发起查询，顶级域服务器再告诉dns服务器再下一层次域的服务器地址，依次下去，直到找到最终主机地址并返回给客户端。
在这个解析流程中，**父域总是将子域dns服务器的地址告诉dns服务器**。所以，父域是知道子域dns服务器的ip地址的，因此**在父域的区域数据文件中，必然要指定子域dns服务器的A记录**。

另外，父域如何区分其内主机是普通的主机还是子域dns服务器？总不能让子域dns服务器被父域当成普通主机吧？区分的方法是**在父域的区域数据文件中使用NS记录来存储子域dns服务器信息**。这样一来，子域dns服务器不仅在父域的区域数据文件中有了NS记录，还有了A记录，父域就能将子域信息返回给查询发起者。

### 划分子域
比如，那种有子公司的公司，使用子域来管理的。
在主DNS服务器`/etc/named.rfc1912.zones`独立一个zone区域，并生成文件，交由子域管理员管理此文件。

> 注：子域的服务器还是和父域一个服务器。只不过子域下还有一些域名，这些域名单独划分一个域（即子域）来管理。

比如：`sz`分公司。`hunk.tech`下划分`sz.hunk.tech`子域。
```bash
zone "sz.hunk.tech" IN {
        type master;
        file "named.sz.hunk.tech";
        allow-update { none; };
};
```
zone区域文件`named.sz.hunk.tech`内容如下：
```bash
$ORIGIN .
$TTL 600    ; 10 minutes
sz.hunk.tech        IN SOA  6-DNS-1.sz.hunk.tech. admin.sz.hunk.tech. (
                21         ; serial
                7200       ; refresh (2 hours)
                600        ; retry (10 minutes)
                86400      ; expire (1 day)
                10800      ; minimum (3 hours)
                )
            NS  6-DNS-1.sz.hunk.tech.
$ORIGIN sz.hunk.tech.
6-DNS-1         A   192.168.4.200
7-WEB-2         A   7.7.7.7
www         CNAME   7-WEB-2
```

### 子域授权新服务器
子域授权：在原有的域上再划分出一个小的区域并指定新DNS服务器。
对子域进行授权，只需在父域的区域文件中直接授权即可。在子域的NS服务器上，直接创建子域的区域文件，管理资源记录。

在这个小的区域中如果有客户端请求解析，则只要找新的子DNS服务器。这样的做的好处可以减轻主DNS的压力，也有利于管理。
> 注：如果子域设置了主从DNS，那么，在委派的时候，也是需要把管理子域的主、从DNS服务器同时添加记录，否则可能会单点故障。

#### 范例
比如：bj 分公司。`hunk.tech`下划分`bj.hunk.tech`子域。
```json
zone "bj.hunk.tech" IN {
        type master;
        file "named.bj.hunk.tech";
        allow-update { none; };
};

```

zone区域文件`named.bj.hunk.tech`内容如下:
```json
$ORIGIN .
$TTL 600        ; 10 minutes
bj.hunk.tech            IN SOA  bj-dns.bj.hunk.tech. admin.bj.hunk.tech. (
                                23         ; serial
                                7200       ; refresh (2 hours)
                                600        ; retry (10 minutes)
                                86400      ; expire (1 day)
                                10800      ; minimum (3 hours)
                                )
                        NS      bj-dns.bj.hunk.tech.
$ORIGIN bj.hunk.tech.
bj-dns                  A       192.168.4.204
bj-WEB-2                A       8.8.8.8
www
```

另外，在父域的DNS服务器上面的zone文件要定义委派的子域DNS的权威DNS服务器:
```json
bj                      NS      bj-dns                     #bj子域 的NS 记录为bj.hunk.tech中定义的DNS服务器
bj-dns                  A       192.168.4.204               # 对应子域权威DNS的A记录
```
### 子域和区域的一部分的区别

子域的区域数据文件中是可以没有SOA记录的，只不过此时子域没有得到父域授权，也就没有自主权，无法提供解析功能。这意味着这不是一个子域，而是父域的一部分，就相当于在父域的区域数据文件中使用`$INCLUDE`一样。

可以认为授权了的子域是父域将自己的孩子送到了它们该到的地方，它们自己能够独挡一面，没有授权的子域实际上只是住在了父域的隔壁，它没有独立解析的能力，一切问题仍然需要父域来负责解答。如下图：
![](attachments/Pasted%20image%2020240117164324.png)

### 小结
根据上述分析，应该就能明白在互联网上申请域名并向外界提供解析时，实际上是向申请机构的区域数据文件中添加NS记录和NS对应的A记录，仅此而已。
由此也知，根域名和顶级域名的区域数据文件是无比巨大的，在其中存放了无数的NS记录和A记录。

## 主机名、域名、FQDN
### 域名
不论是`www.baidu.com`还是`tieba.baidu.com`，它们的域名都是`baidu.com`，严格地说是`baidu.com.`。这是百度所购买的com域下的一个子域名。
### 主机名
对于`www.baidu.com`来说，主机名是`www`，对于`tieba.baidu.com`来说，主机名是`tieba`。其实严格来说，`www.baidu.com`和`tieba.baidu.com`才是主机名，它们都是`baidu.com`域下的主机。一个域下可以定义很多主机，只需配置好它的主机名和对应主机的`IP`地址即可。
### FQDN
FQDN(Fully Qualified Domain Name)：**完全合格域名/全程域名**/完全限定域名/绝对域名。**完全合格域名/全程域名**用的相对较多。
即域名可以通过DNS进行解析，其公式 **FQDN = HostName + Domain，即全程域名=主机名+域**。
FQDN是指包含了所有域的主机名，其中包括根域。FQDN可以说是主机名的一种完全表示形式，它从逻辑上准确地表示出主机在什么地方。例如`www.baidu.com`的FQDN是`www.baidu.com.`，com后面还有个点，这是根域。
#### PQDN
与 FQDN 对应的，系统中的默认域名是**非合格域名/部分限定域名**(PQDN: Partially Qualified Domain Name)，会把当前的区域域名添加到尾部。例如，tsinghua 域内的主机上查找 mail ，本地解析器就会将这个名称转换为 FQDN ，即 `mail.tsinghua.edu.cn.`，然后解析出 IP 地址。
![](attachments/Pasted%20image%2020240104121831.png)

**FQDN**的完整格式是以点结尾的域名。
> DNS 系统中的域名可以是相对的，所以可能是模糊的。FQDN 是一个绝对名称，表示了它相对于域名系统中绝对根目录的位置。

#### 范例
zone 引导文件，如下所示：
```json
zone "abc.com" IN {
    type primary;
    file "/usr/local/bind/etc/named.abc.com";  // RR记录文件名称可以自定义 最好取有意义名称
};
```
**FQDN的写法：**
```json
$TTL 600
@                    IN  SOA     primary.abc.com. admin.abc.com. ( 2022120802 3H 15M 1W 1D )
@                    IN  NS      primary.abc.com.
primary.abc.com.     IN  A       192.168.10.200
@                    IN  MX  10  www.abc.com.
www.abc.com.         IN  A       192.168.10.121
bbs.abc.com.         IN  CNAME   www.abc.com.
ftp.abc.com.         IN  CNAME   www.abc.com.
linux.abc.com.       IN  CNAME   www.abc.com.
secondary.abc.com.   IN  A       192.168.10.201
122.abc.com.         IN  A       192.168.10.122


# 其中2022120802 3H 15M 1W 1D,分别是serial,refresh,retry,expire,Minimum

```

**简写：**
```json
$TTL 600
@               IN  SOA     primary.abc.com. admin.abc.com.( 2022120802 3H 15M 1W 1D )
@               IN  NS      primary
primary         IN  A       192.168.10.200
@               IN  MX  10  www
www             IN  A       192.168.10.121
bbs             IN  CNAME   www
ftp             IN  CNAME   www
linux           IN  CNAME   www
secondary       IN  A       192.168.10.201
122             IN  A       192.168.10.122
```

简写不太容易看明白，而FQDN的写法，又太啰嗦，而且要注意.（点号），所以我个人偏好喜欢这样的写法。
```json
$TTL 600
@               IN  SOA     primary.abc.com. admin.abc.com.( 2022120802 3H 15M 1W 1D )
@               IN  NS      primary.abc.com.
master          IN  A       192.168.10.200
@               IN  MX  10  www.abc.com.
www             IN  A       192.168.10.121
bbs             IN  CNAME   www.abc.com.
ftp             IN  CNAME   www.abc.com.
linux           IN  CNAME   www.abc.com.
secondary       IN  A       192.168.10.201
122             IN  A       192.168.10.122
```

## 域名服务器
域名是分层结构，域名服务器也是对应的层级结构。有了域名结构，还需要遍及全世界的域名服务器去解析域名。

每一级域都有对应的服务器负责，如下所示：
![](attachments/Pasted%20image%2020240104113557.png)
还有一种 DNS 服务器是`本地域名服务器(local name server)`，本地域名服务器并不属于上图要求的域名服务器层次，但是它对域名系统非常重要。当我们在使用 DNS 查询请求时，我们发出的查询请求并不会直接传输到权威域 DNS 服务器，而是会发送给本地域名服务器。本地域名服务器然后查询本地是否有资源记录，如果没有，再进行递归查询。

### 特征
每一个域内都有至少一个专门的DNS服务器，这些服务器负责接收并响应本域内的主机名的解析请求。
> 注：一个域名服务器所负责的范围，或者说有管理权限的范围，就称为区（zone）

- 每个层的域名上都有自己的域名服务器，最顶层的是根域名服务器。
- 每一级域名服务器都知道下级域名服务器的IP地址。
- 为了容灾, 每一级至少设置两个或以上的域名服务器。

### 分类
DNS 服务器可以分成四种：

|分类|作用|
|---|---|
|根域名服务器|最高层次的域名服务器，本地域名服务器解析不了的域名就会向其求助，从根域名服务器进行域名解析。|
|顶级域名服务器|负责管理在该顶级域名服务器下注册的二级域名。|
|权限（权威）域名服务器|负责一个区的域名解析工作。|
|本地域名服务器|当一个主机发出DNS查询请求时，这个查询请求首先发给本地域名服务器。一般本地DNS服务器又是递归DNS服务器|


它们的关系如下图。
![](attachments/Pasted%20image%2020240129114310.png)

#### 根域名服务器
根域名服务器全世界一共有13台（都是服务器集群）。它们的域名和 IP 地址如下。
![](attachments/Pasted%20image%2020240129114513.png)

根域名服务器的 IP 地址是不变的，集成在操作系统里面。
操作系统会选其中一台，查询 TLD 服务器的 IP 地址。
```bash
dig @192.33.4.12 es6.ruanyifeng.com
```
上面示例中，我们选择`192.33.4.12`，向它发出查询，询问`es6.ruanyifeng.com`的 TLD 服务器的 IP 地址。
dig 命令的输出结果如下。
![](attachments/Pasted%20image%2020240129114600.png)
因为它给不了 `es6.ruanyifeng.com` 的 IP 地址，所以输出结果中没有 `ANSWER SECTION`，只有一个 `AUTHORITY SECTION`，给出了`com.`的13台 TLD 服务器的域名。

下面还有一个 `ADDITIONAL SECTION`，给出了这13台 TLD 服务器的 IP 地址（包含 IPv4 和 IPv6 两个地址）。

#### TLD 服务器
有了 TLD 服务器的 IP 地址以后，我们再选一台接着查询。
```bash
dig @192.41.162.30 es6.ruanyifeng.com
```
上面示例中，192.41.162.30 是随便选的一台 .com 的 TLD 服务器，我们向它询问 `es6.ruanyifeng.com` 的 IP 地址。
![](attachments/Pasted%20image%2020240129114657.png)

它依然没有 `ANSWER SECTION` 的部分，只有 `AUTHORITY SECTION`，给出了一级域名 `ruanyifeng.com` 的两台 DNS 服务器。

下面的`ADDITIONAL SECTION` 就是这两台 DNS 服务器对应的 IP 地址。

#### 二级域名的 DNS 服务器
再向二级域名的 DNS 服务器查询二级域名的 IP 地址。
```bash
dig @172.64.32.123 es6.ruanyifeng.com
```
返回结果如下。
![](attachments/Pasted%20image%2020240129114802.png)

### 范例
**根域的域名服务器**：
```bash
# dig -t NS .

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-16.P2.el7_8.3 <<>> -t NS .
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45095
;; flags: qr rd ra; QUERY: 1, ANSWER: 13, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;.				IN	NS

;; ANSWER SECTION:
.			2992	IN	NS	f.root-servers.net.
.			2992	IN	NS	j.root-servers.net.
.			2992	IN	NS	e.root-servers.net.
.			2992	IN	NS	g.root-servers.net.
.			2992	IN	NS	k.root-servers.net.
.			2992	IN	NS	c.root-servers.net.
.			2992	IN	NS	b.root-servers.net.
.			2992	IN	NS	l.root-servers.net.
.			2992	IN	NS	d.root-servers.net.
.			2992	IN	NS	i.root-servers.net.
.			2992	IN	NS	m.root-servers.net.
.			2992	IN	NS	h.root-servers.net.
.			2992	IN	NS	a.root-servers.net.

;; Query time: 5 msec
;; SERVER: 10.6.6.6#53(10.6.6.6)
;; WHEN: Mon Dec 25 19:31:47 CST 2023
;; MSG SIZE  rcvd: 239
```
这次终于有了 `ANSWER SECTION`，得到了最终域名的 IP 地址。

# DNS解析流程
将域名转换为对应的 IP 地址 的过程叫做**域名解析**。在域名解析过程中，DNS client 的主机调用解析器 （ Resolver ），向 DNS server 发出请求，DNS server 完成域名解析。
![](attachments/Pasted%20image%2020240112144506.png)


**域名解析**是按照 DNS 分层结构的特点，自顶向下进行的。但是如果每一个域名解析都从根域名服务器开始，那么根域名服务器有可能无法承载海量的流量。在实际应用中，大多数域名解析都是在**本地域名服务器**完成。通过合理设置本地域名服务器，由本地域名服务器负责大部分的域名解析请求，提高域名解析效率。

范例如下：
![](attachments/Pasted%20image%2020240104123104.png)
DNS 客户端进行域名 www.tsinghua.edu.cn的解析过程如下：

1. DNS 客户端 向本地域名服务器发送请求，查询 www.tsinghua.edu.cn 主机的 IP 地址；
2. 本地域名服务器 查询数据库，发现没有域名为 www.tsinghua.edu.cn 的主机，于是将请求发送给根域名服务器；
3. 根域名服务器 查询数据库，发现没有这个主机域名记录，但是根域名服务器知道 cn 域名服务器可以解析这个域名，于是将 cn 域名服务器的 IP 地址返回给本地域名服务器；
4. 本地域名服务器 向 cn 域名服务器查询 www.tsinghua.edu.cn 主机的 IP 地址；
5. cn 域名服务器 查询数据库，也没有相关记录，但是知道 edu.cn 域名服务器可以解析这个域名，于是将 edu.cn 域名服务器的 IP 地址返回给本地域名服务器；
6. 本地域名服务器 再向 edu.cn 域名服务器查询 www.tsinghua.edu.cn 主机 IP 地址；
7. edu.cn 域名服务器 查询数据库，也没有相关记录，但是知道 tsinghua.edu.cn 域名服务器可以解析这个域名，于是将 tsinghua.edu.cn 的域名服务器 IP 地址返回给本地域名服务器；
8. 本地域名服务器 向 tsinghua.edu.cn 域名服务器查询 www.tsinghua.edu.cn 主机的 IP 地址；
9. tsinghua.edu.cn 域名服务器 查询数据库，发现有主机域名记录，于是给本地域名服务器返回 www.tsinghua.edu.cn 对应的 IP 地址；
10. 最后 本地域名服务器 将 www.tsinghua.edu.cn 的 IP 地址返回给客户端，整个解析过程完成。

## DNS 解析器
从应用程序的角度看，访问 DNS 是通过一个叫**解析器**（ Resolver ）的应用程序来完成的。发送一个 TCP 或 UDP 数据包之前，解析器必须将域名（主机名）转换为 IP 地址。一个解析器至少要注册一个域名服务器的 IP 地址。通常，它至少包括本地域名服务器的 IP 地址。
![](attachments/Pasted%20image%2020240104122945.png)

## DNS 域名服务器
DNS 域名空间的层次结构，允许不同的域名服务器管理域名空间的不同部分。**域名服务器**是指管理域名的主机及软件，它可以管理所在分层的域。其所管理的分层叫做**区域**（ zone ）。一个 zone 是 DNS 域名空间的一棵子树，它可以单独管理而不受其它 zone 影响。**每层都设有一个域名服务器。**
![](attachments/Pasted%20image%2020240104123022.png)

## 解析分类
域名解析分为**动态域名解析**和**静态域名解析**。在解析域名时，首先采用静态域名解析，如果静态解析不成功，再采用动态域名解析。
### 静态域名解析
**静态域名解析**是通过静态域名解析表进行的，手动建立域名和 IP 地址之间的对应关系表，该表的作用类似于 Windows 操作系统下的 hosts 文件 ，可以将一些常用的域名放入表中。当 DNS client 需要域名所对应的 IP 地址时，即到静态域名解析表中去查找指定的域名，从而获得所对应的 IP 地址，提高域名解析的效率。
![](attachments/Pasted%20image%2020240104134946.png)

### 动态域名解析
**动态域名解析**需要专用的域名服务器（ DNS server ）运行域名解析服务器程序，提供从域名到 IP 地址的映射关系，负责接收客户端（ DNS client）提出的域名解析请求。

# DNS查询
DNS解析流程分为**递归查询**和**迭代查询**。
递归查询是以本地名称服务器为中心查询， 递归查询是默认方式；
迭代查询是以DNS客户端，也就是客户机器为中心查询。
**其实DNS客户端到本地名称服务器是递归，而本地名称服务器和其他名称服务器之间是迭代。**

![](attachments/Pasted%20image%2020240122190915.png)

##  递归查询（recursion）
![](attachments/Pasted%20image%2020231225120053.png)
### 定义
“递归解析”（或叫“递归查询”，其实意思是一样的）是最常见，**也是默认的解析方式**。在这种解析方式中，如果客户端配置的本地名称服务器，**（又称Local DNS， 可以是默认的运营商提供的Local DNS 或者自己设置的DNS）** 不能解析的话，则后面的查询全由本地名称服务器代替DNS客户端进行查询，直到本地名称服务器从权威名称服务器得到了正确的解析结果，然后由本地名称服务器告诉DNS客户端查询的结果。

> 即：客户端发起一个DNS解析请求，本地域名服务器不知道被查询的域名的IP地址，那么本地域名服务器就以DNS客户的身份，向其它根域名服务器继续发出查询请求报文(即本地域名服务器替主机继续查询)，而不是让主机自己进行下一步查询。直到有服务器响应回答了该请求后，将请求结果返回给客户端。

### 特征
在这个查询过程中，一直是以本地名称服务器（Local DNS）为中心的，DNS客户端只是发出原始的域名查询请求报文，然后就一直处于等待状态的，直到本地名称服务器发来了最终的查询结果。此时的本地名称服务器就相当于中介代理的作用。

### 流程
完整的递归DNS 查询流程需要 DNS 服务器从根域名 “.” 服务器，顶级域名服务器（例如“.com”），一级域名服务器（例如“example.com”）等一级一级递归查询，直到最终找到权威服务器取得结果，并返回给客户。同时，递归服务器根据域名 TTL，缓存查询结果，便于相同域名重复查询。  
![](attachments/Pasted%20image%2020231225200009.png)
递归解析的基本流程如下：
（1）客户端向本机配置的本地名称服务器（在此仅以首选DNS服务器为例进行介绍，所配置其它备用DNS服务器的解析流程完全一样）发出DNS域名查询请求。

（2）本地名称服务器收到请求后，先查询本地的缓存，如果有该域名的记录项，则本地名称服务器就直接把查询的结果返回给客户端；如果本地缓存中没有该域名的记录，则本地名称服务器再以DNS客户端的角色发送与前面一样的DNS域名查询请求发给**根名称服务器**。

（3）根名称服务器收到DNS请求后，把所查询得到的所请求的DNS域名中**顶级域名所对应的顶级名称服务器**地址返回给本地名称服务器。  
（4）本地名称服务器根据根名称服务器所返回的顶级名称服务器地址，向对应的顶级名称服务器发送与前面一样的DNS域名查询请求。

（5）对应的顶级名称服务器在收到DNS查询请求后，也是先查询自己的缓存，如果有所请求的DNS域名的记录项，则相接把对应的记录项返回给本地名称服务器，然后再由本地名称服务器返回给DNS客户端，否则向本地名称服务器返回所请求的DNS域名中的二级域名所对应的二级名称服务器地址。

然后本地名称服务器继续按照前面介绍的方法一次次地向三级、四级名称服务器查询，直到最终的对应域名所在区域的**权威名称服务器**返回到最终的记录给本地名称服务器。然后再由本地名称服务器返回给DNS客户，同时本地名称服务器会缓存本次查询得到的记录项。

### 应用
**递归 DNS 服务器大多数在运营商端，负责网络接入终端的 DNS 查询，即网络访问设备上配置的 DNS 服务器 IP。**
递归 DNS 的访问过程如下图所示（递归 DNS 在图中表示为 Local DNS）：
![](attachments/Pasted%20image%2020231225140931.png)
## 迭代查询（iteration）
![](attachments/Pasted%20image%2020231225120103.png)
### 定义
客户端(下级服务器)发起DNS解析请求后，若上级DNS服务器并不能直接提供该DNS的解析结果，则该上级DNS服务器会告知客户端(下级服务器)另一个可能查询到该DNS解析结果的DNS服务器IP。客户端(下级服务器)再次请求这个DNS服务器，如此类推，直到查询到对应的结果为止。

> 注：上面讲的是客户端的查询方式是迭代，实际上一般客户端的默认查询方式是递归查询。

### 迭代查询的条件
**DNS递归名称解析**： 在DNS递归名称解析中，当所配置的本地名称服务器解析不了时，后面的查询工作是由本地名称服务器替代DNS客户端进行的（以“本地名称服务器”为中心），只需要本地名称服务器向DNS客户端返回最终的查询结果即可。

**DNS迭代名称解析**：（或者叫“迭代查询”）的所有查询工作全部是DNS客户端自己进行（以“DNS客户端”自己为中心）。
在以下条件之一满足时就会采用迭代名称解析方式：
- 客户端配置：
在查询本地名称服务器时，如果客户端的请求报文中没有申请使用递归查询，即在DNS请求报头部的RD字段没有置1。相当于说“你都没有主动要求我为你进行递归查询，我当然不会为你工作了”。

- 本地DNS服务器配置：
客户端在DNS请求报文中申请使用的是递归查询（也就是RD字段置1了），但在所配置的本地名称服务器上是禁用递归查询（DNS服务器一般默认支持递归查询的），即在应答DNS报文头部的RA字段置0。

### 递归查询和迭代查询对比
**一般情况下，DNS客户端和本地名称服务器是递归，而本地名称服务器和其他名称服务器之间是迭代**。
![](attachments/Pasted%20image%2020240104134248.png)
**递归**：客户端只发一次请求，要求对方给出最终结果。
**迭代**：客户端发出一次请求，对方如果没有授权回答（权威回答），它就会返回一个能解答这个查询的其它名称服务器列表，客户端会再向返回的列表中发出请求，直到找到最终负责所查域名的名称服务器，从它得到最终结果。

例如A主机要查询C域中的一个主机，A所指向的DNS服务器为B，递归和迭代查询的方式是这样的：
递归查询：`A --> B --> C --> B --> A`
迭代查询：`A --> B A --> C --> A`

## 反向查询
### 定义
在 DNS 查询中，客户端希望知道域名对应的 IP 地址，这种查询称为**正向查询**。大部分的 DNS 查询都是正向查询。与正向查询对应的，是**反向查询**。它允许 DNS 客户端通过 IP 地址查找对应的域名。
![](attachments/Pasted%20image%2020240104134440.png)

### 原理
为实现反向查询，在 DNS 标准中定义了特色域 in-addr.arpa 域，并保留在域名空间中，以便执行反向查询。为创建反向域名空间，in-addr.arpa 域中的子域是按照 IP 地址相反的顺序 构造的。
举个栗子：`www.tsinghua.edu.cn`的 IP 地址是 `166.111.4.100` ，那么在 in-addr.arpa 域中对应的节点就是 `100.4.111.166` 。
![](attachments/Pasted%20image%2020240104134548.png)


## 非递归查询
在某些情况下，并不希望NS服务器额外地为递归查询寻找答案，或是建立数据缓存。
root名称服务器就是个例子。root名称服务器非常繁忙，不能再浪费额外的时间为递归查询寻找答案。

oot名称服务器仅依据其拥有的权威数据作出应答。该应答可能包含着答案，但更有可能包含着到其他名称服务器的指引。
由于root服务器不支持递归查询，所以它们不用为非权威数据建立缓存，否则它们的缓存将会非常巨大。

## 查询优先级
## 分级查询
# DNS应答
## 返回结果答案类别
- `有查询结果`（肯定答案）
- `不存在查询结果`（否定答案）

### 否定答案
否定答案 「NXDOMAIN（Non-Existent Domain，rcode=3）」是一种特定的应答类型，表示查询的域名在DNS数据库中不存在。在诸多DNS错误应答类型中，NXDOMAIN是最为常见的一种，其在所有DNS错误应答类型中占比超90%。

DNS递归解析器也会对`NXDOMAIN`应答的查询域名进行缓存 ([RFC 2308](https://datatracker.ietf.org/doc/html/rfc2308?ref=blog.xlab.qianxin.com))，并且有缓存时间。

例如某个Client请求`51cto.com`域下的ftp主机，但是实际上`51cto.com`下面可能根本没有这个ftp主机，那么`51cto.com`就会给否定答案，为了防止Client不死心的访问ftp搞破坏，`51cto.com`这个域负责解析的DNS服务器有必要给Client指定否定答案的缓存时间。

### 肯定答案
#### 权威应答
DNS服务器自己直接负责的域返回的答案
#### 非权威应答
DNS服务器未负责的域，由缓存或者查询到的记录返回的答案


# DNS的泛域名解析

## 定义
泛解析（泛域名：Wildcard DNS）是指将多个子域名解析到同一个IP地址；例如域名 `cloud-example.com`，设置泛解析 `*.cloud-example.com` ，则该域名下所有的子域名（如 `a.cloud-example.com`，`b.cloud-example.com`，`c.cloud-example.com`等）都将指向与 `*.cloud-example.com` 相同的IP地址。

## 场景  
在网站运营中，域名持有者为了避免因为错误输入，而造成用户流失，就会使用泛域名解析。  它可以将没有明确设置的子域名一律解析到一个IP地址上。这样，即使用户输入错误的子域名，也可以访问到域名持有者指定的IP地址。

## 分类
### 泛解析
泛解析：使用通配符 `*` 来匹配所有的子域名。通配符记录以星号 (`*`) 字符作为域名的最左侧标签。例如，`*.example.com`。域名中其他位置的`*`仅作为普通字符使用。例如， `new.*.example.com` 不是有效的通配符 DNS 记录。

当您需要解析的多个子域名对应为同一个 IP 时，可通过以下两种方式进行添加：
- 普通添加方式：当您有多个子域名时，需添加多条记录进行解析。如下图所示：
![](attachments/Pasted%20image%2020240122104808.png)

- 泛解析添加方式：当您有多个子域名时，只需添加一条记录，即可对多个子域名进行解析。如下图所示：
![](attachments/Pasted%20image%2020240122104820.png)
```bash
333.demo.com	A	6.6.6.6
*.demo.com		A	6.6.6.6

# 第一个解析记录优先级，大于，第二个泛解析记录
```

### 混合泛解析
混合泛解析：在泛解析的基础上，增加一个限制，使记录可以按照需求进行分类。

混合泛解析可通过在`*`前添加字符或者在`*`后添加字符。且仅支持添加3个字符。例如：`aaa*`或者`*aaa`。

- 普通添加方式：当您有多个子域名时，需添加多条记录进行解析。如下图所示：
![](attachments/Pasted%20image%2020240122104849.png)

- 混合泛解析添加方式：当您有多个子域名时，只需按照分组添加记录，即可对多个子域名进行解析。如下图所示：
![](attachments/Pasted%20image%2020240122104907.png)

```bash
原解析记录，如下：
111.demo.com	A	6.6.6.6
112.demo.com	A	6.6.6.6
113.demo.com	A	6.6.6.6

771.demo.com	A	7.7.7.7
772.demo.com	A	7.7.7.7
773.demo.com	A	7.7.7.7

转为泛解析，如下：
1*.demo.com		A	6.6.6.6
7*.demo.com		A	7.7.7.7
```

## 泛解析查询规则
- DNS查询请求优先进行线路（view）匹配查询，其次进行域名匹配查询；
- 同一线路下，精确域名匹配查询优先级高于泛域名查询；

即：同一个view的zone下的精确匹配高于泛解析。

## 泛解析的实现方式
除NS类型和SOA之外，其他类型的记录集均支持泛解析。

泛解析的常用实现方式主要有两种：CNAME方式和A记录方式。
CNAME方式是指将一个域名指向另一个域名，这种方式可以方便地管理大量的子域名。
A记录方式是指将多个域名都解析到同一个IP地址，这种方式应用于对性能要求较高的场景。

## 范例
DNS的泛域名解析，就是都匹配不到的时候，此时就会匹配到`*`的地址。
```bash
# cat /var/named/test.cn.zone
$TTL 1D
@       IN SOA  @ rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
test.cn. NS     tk
tk      A       10.0.0.200
www     A       10.0.0.201
www     A       10.0.0.202
*       A       1.1.1.1
```

我们进行测试
```bash
[root@gitlab ~]# nslookup abc.test.cn
Server:         10.0.0.200
Address:        10.0.0.200#53
Name:   abc.test.cn
Address: 1.1.1.1

[root@gitlab ~]# nslookup www.test.cn
Server:         10.0.0.200
Address:        10.0.0.200#53
Name:   www.test.cn
Address: 10.0.0.201
Name:   www.test.cn
Address: 10.0.0.202
```

## 建议
慎用泛域名解析。
泛域名解析是指利用通配符 * 将所有的子域名都指向相同的解析记录，实现灵活配置。然而当某个子域名需要独立配置时，容易忽略泛域名的配置，引发故障。

举个例子(以真实故障为蓝本)，方便大家理解：
1> 业务上线初期为了方便配置使用泛域名解析: `*.example.com CNAME xxxcdn.com`
2> 发展一段时间后 `a.example.com` 有了新需求，需要加个 TXT 做验证
3> 运维同学添加解析 `a.example.com TXT xxx`
4> 此时因为 `a.example.com` 只有 TXT 记录，没有 `A/AAAA` 或 `CNAME`，直接导致 a 站点无法访问。

# 资源记录RR
当一个解析器把域名传递给DNS时，DNS所返回的是与该域名相关联的资源记录；DNS的主要功能就是将域名映射到资源记录上。

## 定义
- 为了将名字解析为IP地址，服务器查询它们的区(DNS数据库文件)。区中包含组成相关DNS域资源信息的资源记录（RR）。
- 某些资源记录不仅包括DNS域中服务器的信息，还可以用于定义域，即指定每台服务器授权了哪些域，这些资源记录就是SOA和NS资源记录。
- 数据库中的每一个条目称作是一个资源记录(Resource Record，RR)，它是一个五元组，可以用以下格式表示：
```c
domain_name Time_to_live Class Type Value  

说明：
# Domain_name：	指出这条记录适用于哪个域名  
# Time_to_live：用来表明记录的生存周期，也就是说最多可以缓存该记录多长时间，可省略  
# Class：	    一般总是IN；对应的是internet  
# Type：	    资源记录的类型  
# Value：	    记录的值，如果是A记录，则value是一个IPv4地址
```


## 格式
资源记录是一个包含了下列字段的 4 元组；
```c
(Name, Value, Type, TTL)
```

资源记录(Resource Record)格式
```c
name [ttl(缓存时间)] IN 资源记录类型 (RRtype) Value
 
宏定义以$开头
TTL        #缓存时间
ORIGIN .   #源地址，ORIGIN . 意思代表根域
```

一个资源记录包括5个部分：

|资源记录|描述|
|---|---|
|域名|Domain Name|
|生存周期|Time to Live|
|类别|Class|
|类型|Type|
|值|Value|

- 域名：
指出这条记录适用于哪个域
通常，每个域有多条记录，而数据库则保存了多个域的信息  
域名字段是匹配查询条件的主要关键字  
记录在数据库中的顺序是无关紧要的

- 生存时间
指示该条记录的稳定程度
极稳定的信息会被分配一个很大的值，如 86400 （一天时间的秒数）  
非常不稳定的信息会被分配一个很小的值，如60（1分钟）

- 类别
对于互联网信息，它总是 IN

- 类型
指出了这是什么类型的记录。
RR 会有不同的类型，下面是不同类型的 RR 汇总表。

|DNS RR 类型|解释|
|---|---|
|A 记录|IPv4 主机记录，用于将域名映射到 IPv4 地址|
|AAAA 记录|IPv6 主机记录，用于将域名映射到 IPv6 地址|
|CNAME 记录|别名记录，用于映射 DNS 域名的别名|
|MX 记录|邮件交换器，用于将 DNS 域名映射到邮件服务器|
|PTR 记录|指针，用于反向查找（IP地址到域名解析）|
|SRV 记录|SRV记录，用于映射可用服务。|



## SOA记录
### 定义
SOA( Start of Authority)，起始授权机构记录：用来表示被标记成在众多NS记录中哪一台是主服务器。SOA必须是区域数据库文件第一条记录，并且一个zone文件只有一个SOA记录。

SOA记录表明了DNS服务器之间的关系。SOA记录表明了谁是这个区域的所有者。
比如`51CTO.COM`这个区域。一个DNS服务器安装后，需要创建一个区域，以后这个区域的查询解析，都是通过DNS服务器来完成的。
**所有者**: 这里所说的所有者，就是谁对这个区域有修改权利。
>常见的DNS服务器只能创建一个标准区域，然后可以创建很多个辅助区域。（有的DNS服务器也可以创建多个标准区域）
>标准区域是可以读写修改的。而辅助区域只能通过标准区域复制来完成，不能在辅助区域中进行修改。
>而创建标准区域的DNS就会有SOA记录，或者准确说SOA记录中的主机地址一定是这个标准区域的服务器IP地址。

### 特性
SOA 作为所有区域文件的强制性记录，他必须是 ZONE 文件中的RR记录的第一个。

SOA记录的作用在于提供DNS区域的权威信息。因此在进行DNS解析时，当要查询的域名在所有递归解析服务器没有域名的解析缓存时，递归DNS服务器首先会查询该域名的SOA记录，以确定该域名的权威DNS服务器，并从权威DNS服务器上获取相应的DNS记录。

### 内容说明
SOA记录负责说明哪个DNS服务器是主服务器，以及主服务器和辅助服务器之间的一些关联参数。
SOA记录中包含了该DNS区域的管理员、该DNS区域的刷新时间、重试时间、过期时间等信息。SOA记录中的序列号字段还可以用来追踪DNS区域文件的更新历史。

在域名配置中，Zone文件中的 **SOA记录格式**：
```text
[LOCATION NAME] IN SOA  [PRIMARY_DNS_SERVER_NAME] [EMAIL_ADDRESS_NAME] (
  Serial_NUM ;序列号
  Refresh    ;刷新时间
  Retry      ;重试时间
  Expire     ;过期时间
  Min_TTL    ;最小TTL时间
)

总结即：
格式为：[zone] IN SOA [主机名] [管理员email] （[五组更新时间参数])

注：
ZONE文件中使用`;`表示注释符号。
@在ZONE文件中具有特殊含义，代表当前区域。
```

- `Location Name`： 区域的名称，或者用 `@` 进行代替；
-  `PRIMARY_DNS_SERVER_NAME`： 用于规定解析当前域名的主服务器
这个服务器的IP地址以及详细资源需要在后边被规定；
- `EMAIL_ADDRESS_NAME` 指定了管理员的 Email 地址；
比如： `admin.domain.com. 实际上等价于 admin@domain.com.`
- `serial number`（序列号）：
是域名记录的版本，每更改一次域名的任何DNS记录，版本号就会自动加一。
在配置了区域复制时，辅助DNS服 务器会间歇的查询主服务器上DNS区域的序列号，如果主服务器上DNS区域的序列号大于自己的序列号，则辅助DNS服务器向主服务器发起区域复制。建议的格式为YYYYMMDDnn 其中nn为修订号。

- `refresh`（刷新时间，单位秒）：
辅助DNS服务器(`secondary dns`)查询主服务器(`primary dns`)以进行区域更新前等待的时间。当刷新时间到期时，辅助DNS服务器从主服务器上获取主DNS区域的SOA记录， 然后和本地辅助DNS区域的SOA记录相比较，如果记录中的`serial number`跟`secondary dns`已有的序列号不一样，则会向`primary dns`请求传送域名的当前的DNS记录。

- `retry`（重试时间，单位秒）：
如果想`secondary dns`请求传送域名当前的DNS记录失败后，则会间隔重试时间（retry）后再次重试请求。一般来说，retry小于refresh。

- `expire`（过期时间，单位秒）：
在过期时间之前，`secondary dns`会继续请求传送DNS记录，并且在此时间里，`secondary dns`会根据已有的记录应答相关的DNS查询。
如果到了过期时间后，`secondary dns`会停止应答该域名的DNS查询。

- `min TTL`（最小TTL，单位秒）：
域名所有记录的最小生存时间值。当用户DNS查询到记录后，将存在缓存中，直到至少过了这个时间才将缓存刷新重新查询。

- `Negative caching TTL`
有的DNS服务器还会有`Negative caching TTL`，就是当用户DNS查询到无此域名记录（`NXDOMAIN`）时，将把这个“没有此域名的记录”的声明保存在缓存中的时间。

### 注意
```json
domain.com. ns.domain.com. admin.domain.com. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
```
在所有的配置中，`ns.domain.com != ns.domain.com.` ，必须注意在 zone file 中的配置文件的最后 `.` 必须不能省略；

如果不写最后一个的 `.` ，那么该域名就是一个 **相对名** ，结果就是在解析的过程中，这条资源就被当成 `ns.domain.com.domain.com`；

### 范例
```text
@   IN  SOA     nameserver.place.dom.  postmaster.place.dom. (  
    1            ; serial number  
    3600         ; refresh   [1h]  
    600          ; retry     [10m]  
    86400        ; expire    [1d]  
    3600 )       ; min TTL   [1h]


说明：
如上所示，负责人参数为 postmaster.place.dom. 看起来像个主机的完全合格域名，其实意思是
postmaster@place.dom.，是一个邮箱地址。那么为什么负责人这个参数不直接写成邮箱地址呢？
因为@已经被作为特殊代码(zone)，所以就用小数点来取代。因此我们被迫把邮件地址写成了完全合格域名的格式。
```

使用“dig”时的返回格式为：
```c
nameserver.place.dom.              7200   IN      SOA     ns1.he.net. postmaster.place.dom. 1 3600 600 68400 3600
```

## NS记录
NS(Name Server)，域名服务器：用于确定哪些服务器（注意可能不是单个服务器）为一个局域网传递DNS信息以及确定域名由哪个服务器进行解析。
即：NS记录表明谁对某个区域有解释权，即哪些是权威DNS。

一般NS配置在 BIND9 中的 db 文件中进行配置，在 SOA 配置之后。
**NS记录 和 SOA记录是任何一个DNS区域都不能或缺的两条记录**。NS记录可以有多条，每一个NS记录，必须对应一个A记录。
在 BIND 的配置中，可以在 Zone 文件中指定多个 Name Server，一般来说 Name Server 需要搭配 SOA 进行配置；

格式为：
```
[zone] IN NS [主机名称]
```
意思是：这个zone的查询请向后面这部主机请求。如果此zone有两部以上的DNS服务器负责时，就必须写两个NS，而NS后面接的主机名称必须要有ip的对应，这时需要A记录。

范例：
```
longshuai.com.  IN  NS  dnsserver.longshuai.com.
```
前三列仍然是声明性语句，表示`longshuai.com.`域内的DNS服务器(name server)为第四列值所表示的`dnsserver.longshuai.com.`主机。


如果一个域内有多个dns服务器，则必然有主次之分，即master和slave之分。但在NS记录上并不能体现主次关系。例如：
```bash
longshuai.com.    IN  NS  dnsserver1.longshuai.com.  
longshuai.com.    IN  NS  dnsserver2.longshuai.com.
```
表示主机”dnsserver1.longshuai.com.”和主机”dnsserver2.longshuai.com.”都是域”longshuai.com.”内的dns服务器，但没有区分出主次dns服务器。



## A记录
A记录：address，存储的是域内主机名所对应的ip地址。
格式如下：
```json
[hostname] IN A [IP]

范例：
dnsserver.longshuai.com.    IN  A   172.16.10.15
```
客户端之所以能够解析到主机名对应的ip地址，就是因为dns服务器中的有A记录存储了主机名和ip的对应关系。  
AAAA记录存储的是主机名和ipv6地址的对应关系。

## PTR记录
PTR记录：pointer，和A记录相反，存储的是ip地址对应的主机名，该记录只存在于反向解析的区域数据文件中(并非一定)。格式如下：
```bash
16.10.16.172.in-addr.arpa.  IN  PTR  www.longshuai.com.
```
表示解析`172.16.10.16`地址时得到主机名`www.longshuai.com.`

## Cname记录
canonical name，表示规范名的意思，其所代表的记录常称为别名记录。
之所以如此称呼，就是因为为规范名起了一个别名。
什么是规范名？可以简单认为是fqdn。

**格式如下**：
```bash
www1.longshuai.com.     IN  CNAME  www.longshuai.com.
```
最后一列就是规范名，而第一列是规范名即最后一列的别名。
当查询”www1.longshuai.com.”，dns服务器会找到它的规范名”www.longshuai.com."，然后再查询规范名的A记录，也就获得了对应的IP地址并返回给客户端。

**应用场景**：
当需要多个域名指向同一个IP，此时可以将其中一个域名做A记录指向这个IP，然后将其他的域名做这个存在IP的域名的别名。
当服务器的IP地址变更时，就不需要每个域名都进行更改，因为其他的域名都是别名，可以不更改，只需要更改这个A记录的IP地址即可。
正常情况下，业务访问的都是别名。如下所示，`www.baidu.com.` 是 `www.a.shifen.com.`的别名。可以很方面的更改`www.a.shifen.com.`下的A记录的IP地址，而用户是无感知的。
```bash
# dig www.baidu.com

; <<>> DiG 9.9.4-RedHat-9.9.4-51.el7 <<>> www.baidu.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4849
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.baidu.com.			IN	A

;; ANSWER SECTION:
www.baidu.com.		19	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	42	IN	A	110.242.68.4
www.a.shifen.com.	42	IN	A	110.242.68.3

;; Query time: 7 msec
;; SERVER: 10.6.6.6#53(10.6.6.6)
;; WHEN: Mon Jan 15 16:01:47 CST 2024
;; MSG SIZE  rcvd: 101
```

**范例**
一个服务器的域名地址 `nyc3.example.com` 这个主机域名可能会提供不同的服务；
比如说域名 `nyc3.example.com` 指向的主机既要去提供 `web`服务，也要提供 `ftp` 服务，那么这个时候就需要一个别名域名来指向原来的域名，这个时候就可以使用一个 A 记录来管理多个域名，为DNS后期修改配置提供方便。


**使用**
```bash
别名-----> 规范名
规范名-----> A记录A1/A2/A3...

场景：
如果服务器升级，则需要更换服务器的IP地址，即A记录。
用户一般访问的是别名，用户要访问的域名(别名)不进行更改；
只需要将规范名---->A记录映射中的 IP地址更改为新的服务器地址即可。
```
举个例子：有一台主机名为`host.example.com`的服务器，其对外ip为`10.110.72.29`；服务器提供了门户网站和邮箱两个服务，我们希望用户通过地址`www.example.com`和`mail.example.com`分别访问两个服务，那么DNS应该这样记录：
```bash
+------------------+-------+------------------+
| host.example.com | A     | 10.110.72.29     |
+------------------+-------+------------------+
| www.example.com  | CNAME | host.example.com |
+------------------+-------+------------------+
| mail.example.com | CNAME | host.example.com |
+------------------+-------+------------------+
```
这样的话，www和mail服务其实都是指向了同一个ip，当主机的ip地址变更时，只需更改A记录即可。

### CNAME记录的常用用途
CNAME记录是DNS记录的一种类型，有两个字段，即规范名称和别名。它被用来。

- (1) 将同一实体或组织所拥有的一组网站引向该实体的主网站.
- (2) 为不同的网络服务如电子邮件或`FTP`映射多个主机名，每个主机名都指向父域.
- (3) 将CDN的地址作为你的网站源服务器的CNAME记录，从而确保试图访问服务器上的资源的用户被重定向到CDN。
> 比如我要给 `www.vpsss.net` 这个域名加速，去阿里云 CDN 中添加` www.vpsss.net`，会生成 `www.vpsss.net.w.kunlungr.com` 这个 CNAME 地址，然后去阿里云解析 DNS 中给 `www.vpsss.net` 添加一个 CNAME 记录，记录值就是 `www.vpsss.net.w.kunlungr.com` 这个地址，立即生效。这样凡是访问 `www.vpsss.net` 的请求都会被指向这个 CNAME 地址，系统会自行把访问引导到距离最近的 CDN 节点，这样就实现了一个完整的加速过程。

### CNAME 记录的限制
#### CNAME冲突
CNAME 记录将一个主机名映射到另一个主机名，但不允许同一主机名上的其他 DNS 记录（如 MX 记录、A记录、Txt记录）；但 DNSSEC 记录（如 RRSIG 和 NSEC）除外。

> 因为 如果一个域名添加了CNAME后，继承了CNAME值中域名的全部记录，那么这个域名本身也就没有必要存在除CNAME以外的其他记录了，只要一个CNAME记录就全都包括了。
#### zone的CNAME
你可能想把一个zone的DNS解析转发到另一个zone的DNS解析，比如这样配置
```bash
old.taobao.com. IN CNAME new.taobao.com.
```
实际上上面的CNAME意图是错误的，因为`old.taobao.com`已经有了SOA和NS的记录。如果你为`old.taobao.com`配置了CNAME， 那么`old.taobao.com`在CNAME链中的角色是一个别名**alias**，在SOA和NS的角度看它的角色是一个权威名**canonical name**, 一个domain是不能同时承担这两种角色的。
所以，正确的做法应该是为zone下面的domain设置CNAME，像下面这样：
```bash
    img01.old.taobao.com.    IN   CNAME   img.new.taobao.com.
    img02.old.taobao.com.    IN   CNAME   img.new.taobao.com.
```

即：MX 和 NS 记录绝不能指向 CNAME 别名。

#### 指向CNAME的CNAME
CNAME 记录**可以**指向其他 CNAME 记录，但这不是一个好的做法，因为它效率低下。

标准DNS协议是不鼓励指向CNAME的CNAME的，因为这样会导致cname loop，同时会增加解析时间。我遇到的一个DNS服务器 就因为没有做CNAME loop的检查，不断向系统申请资源从而导致内存暴增直至宕机。
如果你决定你的DNS服务器不遵循标准DNS协议，支持多层CNAME的话，那么对于CNAME链的长度限制和CNAME loop的检查是 十分有必要的。

#### 多个CNAME值
一个domain name或许会有对应的对个CNAME，像下面这样：
```bash
img.taobao.com.    IN   CNAME   img01.taobaocdn.com.
img.taobao.com.    IN   CNAME   img02.taobaocdn.com.
img.taobao.com.    IN   CNAME   img03.taobaocdn.com.
```
这看起来好像是一个不可思议的配置。但是你可以这样实现你的DNS服务器来做基于CNAME的负载均衡(sounds pathologicall), 方法就是你的DNS服务器每次会随机返回上面三个CNAME中的一个(当然也可以是你设计的任何选择策略)。
值得一体的是，BIND9不支持这种多值的CNAME。

### CNAME 记录的 DNS 解析过程

详细流程如下所示：
- DNS 客户端（例如浏览器或网络设备）请求地址 www.example.com ，并创建 DNS 请求。
- DNS 解析器接收请求并找到权威名称服务器，该服务器保存带有“example.com”域的 DNS 记录的 DNS 区域文件。
- DNS请求被解析，CNAME记录返回给客户端。
- 客户端发现 www.example.com 只是真实地址“example.com”的别名（CNAME），并为“example.com”发出新的 DNS 查询
- 重复该过程，解析器返回“example.com”的 A 记录，其中包含 IP 地址。
- DNS 客户端现在使用其 IP 地址连接到“example.com”。

### 解析超时问题
这几天有开发同学反馈说是线上的应用dns解析总是失败，我自己测试了连续dig 1000次都是正常的。今天也把合作方的同学一起叫上了。因为之前是看对方有的CNAME设置的TTL是0,造成每次需要重新解析，dns服务器没有办法做cache。

今天排除了很久，后来看了线上的日志才发现问题的本质是业务量非常小，每天就几十笔调用，即便对方把TTL改成60后，实际每次应用服务器查询dns的时候，dns服务器都是需要重新递归一次（每次两三秒），所有可能没有解析出来应用都已经报错了。

这个也没有啥好的解决的方式，要么应用把这个超时时间增大，要么自己另外跑个脚本周期性地访问dns缓存住这样的域名。

### ALIAS 和 CNAME 的区别
ALIAS 记录与 CNAME 一样，也将一个主机名映射到另一个主机名。
但是，ALIAS 记录可以在同一主机名上拥有其他 DNS 记录，而 CNAME 则不然。
此外，ALIAS 的性能 比 CNAME 更好，因为它不需要 DNS 客户端解析另一个主机名，它直接返回一个 IP。然而，ALIAS 记录也需要在幕后进行递归查找，这会影响性能。

### A记录、CNAME和URL区别
**区别**
它们间区别如下：
- A记录 —— 映射域名到一个或多个IP。
- CNAME——映射域名到另一个域名（子域名），再由另一个域名提供ip地址。
- URL转发——重定向一个域名到另一个URL地址，使用HTTP 301状态码。
> URL转发，是通过服务器的特殊设置，将访问您当前域名的用户引导到您指定的另一个网络地址。


（1）**A 记录和 CNAME 属于标准的 DNS 记录，而 URL 转发则实际上只是个简单的重定向。因为 CNAME 是基于 ip 的，而 URL 转发是基于网址。**
（2）**URL 转发可以转发到某一个目录下，甚至某一个文件上。而 CNAME 是不可以，这就是 URL 转发和 CNAME 的主要区别所在。**
（3）CNAME 可以随意设，但 URL 转发在一些缺少网络自由的国家是被禁止的，因为 URL 转发还分显示和隐式，很容易造成误解。

**应用**
了解以上区别，在应用方面：
- A记录——适应于独立主机、有固定IP地址
- CNAME——适应于虚拟主机、变动IP地址主机
- URL转发——适应于更换域名又不想抛弃老用户

**URL隐式转发**
隐性转发：用的是`iframe`框架技术，非重定向技术;效果为浏览器地址栏输入`http://a.com`回车，打开网站内容是目标地址`http://www.dnspod.cn`的网站内容，但地址栏显示当前地址`http://a.com`。
![](attachments/Pasted%20image%2020240122154239.png)

**URL显性转发**
用的是301重定向技术;效果为浏览器地址栏输入`http://a.com`回车，打开网站内容是目标地址`http://www.dnspod.cn`的网站内容，且地址栏显示目标地址`http://www.dnspod.cn`。


## MX记录
MX记录：mail exchanger，邮件交换记录。负责转发或处理该域名内的邮件。和邮件服务器有关，且话题较大，所以不多做叙述，如有深入的必要，请查看《dns & bind》中”Chapter 5. DNS and Electronic Mail”。

```json
#优先级:0-99，数字越小，级别越高，
 
@ 600 IN MX 10 mail
@ 600 IN MX 20 smtp
```
## TXT记录
TXT（Text）记录：TXT记录是一种DNS记录类型，它允许域名的所有者在域名系统中存储文本信息。被用来标记存储在DNS中的不同类型的信息。

### 作用
如果希望对域名进行标识和说明，可以使用 TXT 记录，它们的主要用途包括电子邮件验证（如SPF和DKIM）、网站所有权验证、信息发布等。

**SPF验证**
SPF（Sender Policy Framework）用于登记某个域名拥有的用来**外发邮件的所有ip地址**。
主要作用是**反垃圾邮件**，主要针对那些发信人伪造域名的垃圾邮件。通过在域名的 DNS TXT 记录中设置 SPF 记录，域名所有者可以指定哪些邮件服务器被允许发送来自他们域名的电子邮件。这有助于减少垃圾邮件和电子邮件欺诈。

- SPF的TXT记录和MX记录区别

MX记录的作用是给寄信者指明某个域名的邮件服务器有哪些；
SPF格式的TXT记录的作用跟MX记录相反，它向收信者表明，哪些邮件服务器是经过某个域名认可发送邮件的。


- 范例
![](attachments/Pasted%20image%2020240120130944.png)

大部分时间，TXT 记录是用来做 SPF 反垃圾邮件的。
最典型的 SPF 格式的 TXT 记录例子为 “v=spf1 a mx ~all”，表示只有这个域名的 A 记录和 MX 记录中的 IP 地址有权限使用这个域名发送邮件。


**DKIM验证**
DKIM（DomainKeys Identified Mail） 是另一种电子邮件验证技术，它使用公钥加密来验证电子邮件的来源。
域名所有者可以在 DNS TXT 记录中存储 DKIM 密钥，以便邮件接收者可以验证电子邮件的真实性和完整性。

**验证网站所有权**
某些服务提供商要求域名所有者在其 DNS 记录中添加特定的 TXT 记录，以证明他们拥有该域名。这可以用于验证域名的所有权，例如在 SSL 证书申请过程中。

**信息发布**
除了验证功能外，域名所有者还可以在 DNS TXT 记录中存储各种信息，例如公司联系信息、服务公告、加密密钥或任何其他文本信息。这些信息可以被其他人访问和利用，因此需要小心处理。


**语法和格式**
通常采用双引号括起来的文本字符串表示。
每个 TXT 记录可以包含一个或多个文本字符串，每个字符串之间用空格或分号分隔。

**检测TXT记录**
![](attachments/Pasted%20image%2020240120130145.png)
## SRV记录
SRV (Service)记录是从 RFC2052 中对 SRV资源进行了定义。SRV 被用来记录服务器提供什么样的服务。

## 其他
### SOA记录与NS记录的区别
NS记录和SOA记录是任何一个DNS区域都不可或缺的两条记录。
NS记录仅仅只是声明该域内哪台主机是dns服务器，用来提供名称解析服务，NS记录不会区分哪台dns服务器是master哪台dns服务器是slave。；
SOA记录则用于指定哪个NS记录对应的主机是master dns服务器，也就是从多个dns服务器中挑选一台任命其为该域内的master dns服务器，其他的都是slave，都需要从master上获取域相关数据。

假设`hexun.com`区域有两个DNS服务器负责解析，`ns1.hexun.com`是主服务器，`ns2.hexun.com`是辅助服务器，`ns1.hexun.com`的ip是`202.99.16.1`，`ns2.hexun.com`的ip是`202.99.16.2`。那么我们应该创建两条NS记录，当然，NS记录依赖A记录的解析，我们首先应该为`ns1.hexun.com`和`ns2.hexun.com`创建两条A记录。

NS记录说明了有两个DNS服务器（`ns1.hexun.com` 和 `ns2.hexun.com`）负责`hexun.com`的域名解析，但哪个是主服务器呢？NS记录并没有说明，这个任务由SOA记录来完成。

### 记录的优先级
**优先级**
- 单独设置的域名解析优先级高于泛域名解析

![](attachments/Pasted%20image%2020240122104226.png)
以上示例，访问主机记录为 `blog` 时，将解析至 `2.2.2.2` 主机记录。其他未指定的主机记录将解析至 `1.1.1.1` 主机记录。


- NS记录优先于A记录。
即，如果一个主机地址同时存在NS记录和A记录，则A记录不生效。这里的NS记录只对子域名生效。

- A记录优先于CNAME记录。
即如果一个主机地址同时存在A记录和CNAME记录，则CNAME记录不生效。

- MX记录可以通过设置优先级实现主辅服务器设置
“优先级”中的数字越小表示级别越高。也可以使用相同优先级达到负载均衡的目的


# DNS代理
在使用了 **DNS 代理**（ DNS proxy ）功能的组网中，DNS client 将 DNS 请求报文直接发送给 DNS proxy 。DNS proxy 会先查找本地域名解析表，如果未查询到对应的解析表项，会将 DNS 请求报文转发给 DNS Server ，并在收到 DNS server 的应答报文后将其返回给 DNS client ，从而实现域名解析。
![](attachments/Pasted%20image%2020240104135118.png)
因此，当 DNS server 的地址发生变化时，只需改变 DNS proxy 上的配置，无需逐一改变局域网内每个 DNS client 的配置，从而简化了网络管理。

# DNS缓存
`DNS 缓存(DNS caching)` 有时也叫做 `DNS 解析器缓存`，它是**由操作系统维护的临时数据库**，它包含有**最近的网站和其他 Internet 域的访问记录**。
也就是说， DNS 缓存只是计算机为了满足快速的响应速度而把已加载过的资源缓存起来，再次访问时可以直接快速引用的一项技术和手段。

## DNS 缓存的工作流程
在浏览器向外部发出请求之前，计算机会拦截每个请求并在 DNS 缓存数据库中查找域名，该数据库包含有最近的域名列表，以及 DNS 首次发出请求时 DNS 为它们计算的地址。

## DNS 缓存方式
DNS 数据可缓存到各种不同的位置上，每个位置均将存储 DNS 记录，它的生存时间由 TTL(DNS 字段) 来决定。

从在浏览器的搜索框中输入URL。它会先后访问**浏览器缓存**、**操作系统的缓存**`/etc/hosts`、**最近的DNS服务器缓存**。如果都找不到，才是到根域，顶级（一级）域，二级域等DNS服务器进行查询请求。
![](attachments/Pasted%20image%2020240122191032.png)

于是请求过程就成了下图这样。可以看到上面提到的好几有缓存的地方我都加了个绿色的小文件图标，优先在缓存里做查询。
![](attachments/Pasted%20image%2020240122191110.png)

由于缓存了上面树状结构的信息，最近的DNS服务器也**不再需要每次都从根域开始查起**。比如在缓存里能找到`baidu.com`的服务器IP，就直接跳到二级域服务器上做查找就好了。
正因为**多级缓存**的存在，每一层实际接收到的请求都大大减少了。并且每个人日常访问的网站也就那么几个，所以大部分时候都能命中缓存直接返回IP地址。

### 浏览器缓存
现如今的 Web 浏览器设计默认将 DNS 记录缓存一段时间。因为越靠近 Web 浏览器进行 DNS 缓存，为检查缓存并向 IP 地址发出请求的次数就越少。发出对 DNS 记录的请求时，浏览器缓存是针对所请求的记录而检查的第一个位置。

在 `chrome` 浏览器中，你可以使用 `chrome://net-internals/#dns` 查看 DNS 缓存的状态。这是基于 `Windows` 下查询的，我的 Mac 电脑输入上面 url 后无法查看 DNS ，只能 `clear host cache`。
![](attachments/Pasted%20image%2020231222105944.png)

### 操作系统内核缓存
在浏览器缓存查询后，会进行操作系统级 DNS 解析器的查询，操作系统级 DNS 解析器是 DNS 查询离开你的计算机前的第二站，也是本地查询的最后一个步骤。

## 其他
### 递归查询和缓存
dns服务器接收到递归查询请求时，它需要帮忙去找答案，并亲自回复请求者。
如果收到的是迭代查询请求，则将自己知道的消息(一般是自己负责的域信息，所以是权威消息)告诉请求者，让请求者亲自去查询。

由于dns解析器发起的查询都是递归查询，所以一般客户端配置DNS指向谁就表示找谁帮忙做递归查询。
**允许递归查询的服务器，由于要帮忙查询，所以在递归查询服务器上总是缓存了一些非权威数据。**

**如果是非递归查询服务器，则不用缓存任何数据，只需返回其负责的域的权威数据即可。**

### 缓存的非权威性
要访问的主机IP可能会改变，所有使用缓存得到的答案不一定是对的，因此缓存给的答案是非权威的。缓存给的非权威答案应该设定缓存时间，这个缓存时间的长短由权威者指定。

### 否定答案的缓存
访问某个域下根本不存在的主机，这个域的DNS服务器也会给出答案，但是这是否定答案「NXDOMAIN（Non-Existent Domain）」；
DNS递归解析器也会对NXDOMAIN应答的查询域名进行缓存 ([RFC 2308](https://datatracker.ietf.org/doc/html/rfc2308?ref=blog.xlab.qianxin.com))，并且有缓存时间。

例如某个Client请求`51cto.com`域下的ftp主机，但是实际上`51cto.com`下面可能根本没有这个ftp主机，那么`51cto.com`就会给否定答案，为了防止Client不死心的访问ftp搞破坏，`51cto.com`这个域负责解析的DNS服务器有必要给Client指定否定答案的缓存时间。

### 缓存的容量限制
当处理针对单一域名的大量查询时，服务器能够高效利用缓存机制。一旦域名解析结果被缓存，对该域名的后续查询就能快速响应，从而避免重复解析。然而，在面对众多不同域名的大量查询时，缓存效率显著降低，因为服务器需要为更广泛的域名进行频繁的缓存更新和维护。

比如在缓存数据的量达到这个设定的阈值时，服务 器将会使记录提早过期这样限制就不会被突破。

### 如何判断是缓存回复还是解析后回复
dig测试时，如何区分是否是由缓存给答案还是解析后给答案，有些时候不是很好判断。

例如，dig一下www1.baidu.com的别名记录。
```bash
[root@xuexi ~]# dig -t cname www1.baidu.com @172.16.10.15  
  
; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.62.rc1.el6 <<>> -t cname www1.baidu.com @172.16.10.15  
;; global options: +cmd  
;; Got answer:  
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43817  
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 5, ADDITIONAL: 5  
  
;; QUESTION SECTION:  
;www1.baidu.com.                        IN      CNAME  
  
;; ANSWER SECTION:  
www1.baidu.com.     7200    IN   CNAME   www.baidu.com.  
  
;; AUTHORITY SECTION:  
baidu.com.         86400   IN   NS  ns2.baidu.com.  
baidu.com.         86400   IN   NS  dns.baidu.com.  
baidu.com.         86400   IN   NS  ns3.baidu.com.  
baidu.com.         86400   IN   NS  ns4.baidu.com.  
baidu.com.         86400   IN   NS  ns7.baidu.com.  
  
;; ADDITIONAL SECTION:  
ns4.baidu.com.     172800  IN   A   220.181.38.10  
ns3.baidu.com.     172800  IN   A   220.181.37.10  
ns2.baidu.com.     172800  IN   A   61.135.165.235  
dns.baidu.com.     172800  IN   A   202.108.22.220  
ns7.baidu.com.     172800  IN   A   119.75.219.82  
  
;; Query time: 527 msec  
;; SERVER: 172.16.10.15#53(172.16.10.15)  
;; WHEN: Thu Aug 10 04:53:07 2017  
;; MSG SIZE  rcvd: 220
```
查询消耗了527毫秒。再次执行上面的dig，发现查询时间一定是0毫秒，因为是缓存给的答案。
> 可以简单通过 查询消耗的时间来简单的进行判断。

## 小结
DNS是非常优秀的高并发分布式系统，**通过层次结构将服务进行拆分**，流量分散到多个服务器中。又通过加入**多级缓存**，让每个层级实际接收到的缓存大大减小，因此大大提高了系统的性能。这两点在做业务开发的过程中是可以借鉴的。


# DNS ANYCAST
Anycast 是一种为一组端点提供多个路由路径的技术，每个端点都分配有相同的 IP 地址。 组中的每个设备在网络上通告相同的地址，路由协议用于选择最佳目的地。

这里需要注意两个点：
- IPv4协议本身是不支持Anycast技术的
	- IPv4 anycast 通过BGP来实现。
- IPv6协议原生支持Anycast技术，但是普及率要远低于IPv4

## anycast 是什么
Bgp+anycast是多个主机使用相同ip地址的一种技术，当报文发给该地址时，根据路由协议，选择最近（跳数最少）的主机服务。
所以，在服务器数量足够多的前提下，anycast天然支持负载均衡和抵抗ddos攻击。
简而言之，anycast就是不同服务器用了相同的ip地址，利用BGP，达到就近访问的效果。

## 多播&单播&任播
**Multicast（多播）**
它是指网络中一个节点发出的信息被多个节点收到。实际上，在数据链路层和网络层都有Multicast，通常所说的Multicast大多是针对IP的。这种技术用于多媒体应用、多用户交互(如聊天室)、软件分发等，相比与传统的Unicast可以大大提高效率。在子网内实现
Multicast 较为简单，跨越子网时需要路由器、网关等设备的支持。

**Unicast(单播)**
单播方式下，只有一个发送方和一个接收方。与之比较，组播是指单个发送方对应一组选定接收方。

 **Anycast（任意播）**
 Anycast 中文称为任意播。集Multicast和Unicast的特性于一身。
从宏观上来说，Anycast类似于Multicast，同一种类型的数据流同时存在多个接收者。
从微观上来说，Anycast又有着Unicast的唯一性。

## anycast 会有地址冲突问题么？
同一个IP地址配置在不同的主机上，这不是地址冲突了吗？

我们知道，IP地址存在的目的就是为了指挥路由器选路，最终将数据包路由到目的地，那么IP地址冲突的结果是什么？
**IP地址冲突不是问题，路由冲突才是！
IP地址冲突只有导致路由器的路由冲突(be confusing)的时候才有问题**。

在路由器看来，它们并不知道不同指向的下一跳最终将数据包导向不同的目的地，它们只是认为这只是通往同一个目的地的不同路径罢了！
简单点说， Anycast之所以得以部署和实现，就是利用了IP协议逐跳寻址的特性！
事实上，Anycast的结果是，相同的IP地址位于不同的主机，因此，它的弊端也是显而易见的。
由于 逐跳的路由收敛 和 端到端的五元组连接 之间并没有同步，因此Anycast并不适合基于端到端连接的TCP应用。
> 因为TCP是带有状态的，某一台设备的Anycast IP出现路由收敛时，已经建连的TCP报文发送给了其他的设备，导致TCP链接断开。

### BGP的结合实践
使用BGP，可实现ip不冲突;
(1)设置多个服务器IP为相同IP，如1.1.1.1
 >每一个服务器主机处在不同的地理位置，他们之间不在同一个广播域内。所以把所有主机配置成相同的IP地址并不会引起我们日常所见的IP地址冲突。
(2)通过各个站点的BGP对互联网宣告1.1.1.0/24的网段
(3)以上步骤完成以后，互联网路由表针对1.1.1.1/24会有三个不同的出口路由器，分别是北京，上海，广州（举例）
(4)不同地区的用户根据就近原则，选择相应的主机。

## anycast的优缺点
### 优点
- 负载均衡
Anycast可以零成本实现负载均衡，无视流量大小。
- 高可用
当任意目的主机接入的网络出现故障，导致该目的主机不可达时，客户端请求可以在无人为干预的情况下自动被路由到目前可达的最近目的主机，在一定程度上为目标主机提供了冗余性;
- 低时延
anycast + BGP选路，做到不同地区的用户根据就近原则，选择相应的主机。
### 缺点
- **使用Anycast中的共享单播地址不能作为客户端发起请求**
因为请求的响应不一定能返回到发起的Anycast单播地址。因此，目前Anycast仅适合一些特定的上层协议，从目前的实际应用来看， Anycast最广泛的应用是DNS的部署。

- **Anycast严重依赖于BGP的选路原则**
不同地区的用户根据就近原则，选择相应的主机。

## anycast 应用
Anycast实质上是一种网络技术，它借助于网络中动态路由协议实现服务的负载均衡和冗余。
从实现类型上分，可以分为`subnet Anycast`和`Global Anycast`:
- `subnet Anycast`
指所有目的主机都位于同一网段，此方式仅提供负载均衡和冗余，对安全度提升没有实质效果。

- `Global Anycast`
指目的主机处于不同网段，可能处于不同城市，甚至分布在全球各地，在实际应用中`Global Anycast`中目标主机的部署除地理位置的考虑外，多接入不同自治域的网络中。

AnyCast主要应用于大范围的DNS部署，CDN数据缓存，数据中心等。
### 基于IP Anycast＋BGP的DNS部署
**背景**：
假设部署三个DNS服务器站点，地点分别在北京、上海、广州，且服务于全国的DNS解析。

**常规方案**：
为了实现三个DNS服务器负载均衡，通常会考虑到使用硬件负载均衡设备，例如常见的F5负载均衡设备等。

**AnyCast方案**：
方案优点：

**负载均衡**
通过AnyCast技术，无需要借助任何第三方负载均衡器，就可以轻松达到负载均衡的效果，同时还提供了冗余和高可靠性。
![](attachments/Pasted%20image%2020240110203003.png)
通过配置三个DNS站点的服务器IP为相同IP，例如1.1.1.1/32。然后通过各个站点的BGP对互联网宣告1.1.1.0/24的网段。
以上步骤完成以后，互联网路由表针对1.1.1.1/24会有三个不同的出口路由器，分别是北京，上海，广州。

**就近访问**
假设现有用户都使用1.1.1.1作为DNS服务器，依据就近原则，若用户地域为东北，则会优先采用北京DNS服务器进行解析。
同理，贵阳的宽带路由器通过查看BGP路由，发现1.1.1.1出口最优路由是在广州，那么贵阳用户的DNS数据包将被发送给广州的DNS服务器。而江南一带的则是上海DNS服务器负责提供解析服务。

**故障容灾**
若三台DNS服务器中某一台出现故障，如广东DNS服务宕机，BGP协议会立即停止通告此1.1.1.0/24的网段，Internet路由表将会只有北京和上海的DNS可供选择。

### 防范DDOS攻击
####  DDoS简述
DoS攻击是指故意的攻击网络协议实现的缺陷或直接通过野蛮手段残忍地耗尽被攻击对象的资源，目的是让目标计算机或网络无法提供正常的服务或资源访问，使目标系统服务系统停止响应甚至崩溃。
DDoS（分布式拒绝服务）指借助于客户/服务器技术，将多个计算机联合起来作为攻击平台，同时对一个或多个目标发动DoS攻击。
####  DDoS分类
DDOS攻击主要分为三类：流量型攻击；连接型攻击；特殊协议缺陷。
#### 范例
案例参考：以NTP协议为例，NTP协议基于C/S模式，客户发起NTP时间查询申请，服务器响应NTP查询。假设有成千上万的僵尸主机纷纷伪造如下数据包并不断连续发送给全球NTP服务器：

- 伪造源地址：1.2.3.4               # 此地址为真正需要攻击的地址
- 目标地址：全球各个NTP服务器地址     # 大批量提供响应的节点

当全球各地的NTP服务器收到此查询以后，它会把查询结果发送回给真正的被攻击者1.2.3.4，此时IP地址为1.2.3.4的受害者收到全世界的NTP服务器发过来的数据包时，其有限的带宽链路就很容易产生拥塞并造成服务中断。
受到的DDoS攻击流量=虚假数据包发送数量x全世界NTP服务器的数量。

#### AnyCast防范DDOS攻击
DDOS攻击最关键是需要把所有地理位置分散的小流量最终汇集为一个巨大的流量，从而发起攻击。

在AnyCast环境下，由于多个地理位置不同的主机同时使用同一个IP地址。因此，DDOS攻击流量在穿越运营商路由器时，路由器会根据地理位置远近把数据包路由到距离源地址最近的受害者主机站点，从而间接又再次分散了整个DDOS流量。

如上案例，假设IP地址为1.2.3.4的受害者采用了AnyCast协议部署网络，其服务器分布在全国各地。当DDOS洪流攻击时，不同的NTP服务器根据路由选择，把流量发送到距离NTP服务器最近的受害者服务器上。最终，大流量的攻击被逐步分解。

### 大型服务的CDN部署
AnyCast在大型企业中也常用于CDN部署，采用了Anycast技术为用户提供距离用户最近的Cache服务器，可大大提高了用户的服务体验。在全球建设了多个数据中心，凭借于AnyCast的高冗余性，任何一个数据中心出现网络、系统故障。均不会影响客户体验度，所有当地的客户流量会自动路由到其他就近的数据中心。相对传统企业网络面对网络节点故障的脆弱性，Anycast具有很强的优势。
![](attachments/Pasted%20image%2020240110203709.png)


## IPv6 和 Anycast
在网一个接口上配置了一个IPv6地址之后，会进行DAD 检测，如果发现IPv6地址冲突，则 通过`ip addr`可以看到 `dadfailed` 的标记，后续主动发起请求内核无法使用该地址。
![](attachments/Pasted%20image%2020240110202524.png)
但是对于IPv4而言，发送免费ARP，即使存在冲突，也不会有任何影响。

### 约束
IPv6对Anycast进行了标准化，首先在RFC3513中，它对Anycast提出了两点约束：
> An anycast address must not be used as the source address of an IPv6 packet.
> An anycast address must not be assigned to an IPv6 host, that is, it may be assigned to an IPv6 router only.

- IPv6 anycast IP不可以作为数据包的SIP地址
- IPv6 anycast IP 不可以被配置到端上，可以存在于路由中。

即：**在IPv6中，Anycast不是用来通信的，而是用来路由寻址的**。
紧接着，RFC3513要求 **所有的路由器的所有含有IPv6地址的接口** 都必须配置一个 必选的Anycast地址。
```text
 2.6.1 Required Anycast Address
  The Subnet-Router anycast address is predefined. Its format is as
 follows:
  | n bits | 128-n bits |
 ±———————————————–±—————+
 | subnet prefix | 00000000000000 |
 ±———————————————–±—————+
  The “subnet prefix” in an anycast address is the prefix which identifies a specific link. 
  This anycast address is syntactically the same as a unicast address for an interface on the link with the interface identifier set to zero.
  Packets sent to the Subnet-Router anycast address will be delivered to one router on the subnet. 
  All routers are required to support the Subnet-Router anycast addresses for the subnets to which they have interfaces.
```
比方说，路由R有两个接口，分别配置了两个IP地址：
```bash
e0: 240e:909:2001::4e3/64
e1: 240e:101:4004::111/64
```
那么根据RFC的要求，这个路由器上将会生成下面的Anycast地址：
```bash
e0 Anycast:  240e:909:2001::/64
e1 Anycast:  240e:101:4004::/64
```

### 设置与查看
(1) 开启IPv6的转发
```bash
sysctl -w net.ipv6.conf.all.forwarding=1
```

(2) 查看anycast IP地址：` /proc/net/anycast6` 文件
```bash
[root@localhost src]# cat /proc/net/anycast6
3    enp0s8          fe800000000000000000000000000000     1
4    enp0s9          fe800000000000000000000000000000     1
4    enp0s9          240e0918800300000000000000000000     1

我在enp0s9上配置了如下的IPv6地址：
[root@localhost src]# ip -6 addr ls dev enp0s9
4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qlen 1000
    inet6 240e:918:8003::3f5/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::eb7c:b2da:7088:3c38/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```
所以说，什么都不用干，Linux内核自动生成了其对应的Anycast地址，对应RFC3513的2.6.1 Required Anycast Address格式：`240e:918:8003::`。

（3）查看组播地址：` /proc/net/igmp6`文件
我们知道，这个 `240e:918:8003::` anycast IP 是可以被邻居发现而解析的，而我们知道，IPv6的邻居发现使用的是组播地址，其组播构成规则详见：
对应组播地址： FF02::1:FF00:0000/104 Solicited-Node Address `RFC3513 -RFC4291`对其进行了增强更新。
```text
指定节点的组播地址：
 Solicited-Node Address: FF02:0:0:0:0:1:FFXX:XXXX
  Solicited-node multicast address are computed as a function of a node’s unicast and anycast addresses. 
  A solicited-node multicast address is formed by taking the low-order 24 bits of an address(unicast or anycast) and appending those bits to the prefix
 FF02:0:0:0:0:1:FF00::/104 resulting in a multicast address in the range FF02:0:0:0:0:1:FF00:0000 to FF02:0:0:0:0:1:FFFF:FFFF
```
```bash
[root@localhost src]# cat /proc/net/igmp6
1    lo              ff020000000000000000000000000001     1 0000000C 0
1    lo              ff010000000000000000000000000001     1 00000008 0
...
# 下面这个便是！
4    enp0s9          ff0200000000000000000001ff000000     2 00000004 0
...
```

将Anycast地址作为默认网关的下一跳发送数据，最终邻居解析的时候，只要发送到组播地址 `ff02::1:FF00::` 就可以解析出该网段上的Anycast地址的MAC地址信息。

## LB后端挂载多个DNS服务器
![](attachments/Pasted%20image%2020240110191937.png)

LB后端挂载DNS服务器，本身只是实现了负载均衡；没有实现就近访问。

# CDN 和 DNS
下图是加入了CDN节点的资源请求过程图。
![](attachments/Pasted%20image%2020240117165204.png)

大致过程如下：
(1).客户端向A公司总部的DNS服务器发起`www.a.com`的查询请求。

(2).DNS服务器对`www.a.com`设置了CNAME记录指向CDN节点`www.a.cdn.com`，于是客户端再去查找`www.a.cdn.com`的IP地址。

(3).根据`www.a.cdn.com`的域名，客户端最终会查找到CDN的DNS服务器上。该DNS服务器通过智能DNS(例如BIND的视图功能)为A公司按网络类型(电信、网通)、地理位置远近设置了不同的区域数据文件，每个区域数据文件中设置了对应地区、网络类型的一条或多条A记录指向A公司部署在该地区的缓存服务器。

(4).客户端根据CDN-DNS返回的A记录IP地址，直接进行访问。这时访问的一般是缓存服务器，如果缓存未命中，则缓存服务器负责从总部服务器中请求数据并返回给客户端，同时缓存下来。

甚至，A公司总部的DNS服务器上，还可以直接智能选择，并CNAME到不同区域的的服务器节点上，如下图，注意区域配置文件的设置内容包括CNAME和A记录。这时就无需向服务商购买CDN服务。
![](attachments/Pasted%20image%2020240117165348.png)

# DNS安全
几乎所有的网络请求都会经过 DNS 查询，而且 DNS 和许多其他的 Internet 协议一样，系统设计时并未考虑到安全性，并且存在一些设计限制，这为 DNS 攻击创造了机会。

## DNS攻击
DNS 攻击主要有下面这几种方式：

- 第一种是 Dos 攻击
这种攻击的主要形式是使重要的 DNS 服务器比如 TLD 服务器或者根域名服务器过载，从而无法响应权威服务器的请求，使 DNS 查询不起作用。

- 第二种攻击形式是 DNS 欺骗
通过改变 DNS 资源内容，比如伪装一个官方的 DNS 服务器，回复假的资源记录，从而导致主机在尝试与另一台机器连接时，连接至错误的 IP 地址。

- 第三种攻击形式是 DNS 隧道
这种攻击使用其他网络协议通过 DNS 查询和响应建立隧道。攻击者可以使用 SSH、TCP 或者 HTTP 将恶意软件或者被盗信息传递到 DNS 查询中，这种方式使防火墙无法检测到，从而形成 DNS 攻击。

- 第四种攻击形式是 DNS 劫持
在 DNS 劫持中，攻击者将查询重定向到其他域名服务器。这可以通过恶意软件或未经授权的 DNS 服务器修改来完成。尽管结果类似于 DNS 欺骗，但这是完全不同的攻击，因为它的目标是假的名称服务器，返回假的结果。而DNS欺骗是伪造DNS服务器，回复伪造的DNS资源记录。

- 第五章攻击形式是 DDoS 攻击
也叫做分布式拒绝服务带宽洪泛攻击，这种攻击形式相当于是 Dos 攻击的升级版

## DNS防护
### DNSSEC
DNSSEC 又叫做 DNS 安全扩展，DNSSEC 通过对数据进行数字签名来保护其有效性，从而防止受到攻击。它是由 IETF 提供的一系列 DNS 安全认证的机制。
DNSSEC 不会对数据进行加密，它只会验证你所访问的站点地址是否有效。

### DNS防火墙
有一些攻击是针对服务器进行的，这就需要 DNS 防火墙的登场了，DNS 防火墙是一种可以为 DNS 服务器提供许多安全和性能服务的工具。
DNS 防火墙位于用户的 DNS 解析器和他们尝试访问的网站或服务的权威名称服务器之间。防火墙提供限速访问，以关闭试图淹没服务器的攻击者。如果服务器确实由于攻击或任何其他原因而导致停机，则 DNS 防火墙可以通过提供来自缓存的 DNS 响应来使操作员的站点或服务正常运行。

## DNS 为什么同时使用 TCP 和 UDP
### 使用UDP进行传输
使用UDP用户服务器端口53发送消息。首选UDP协议，因为它速度快且开销低。
由UDP携带的DNS报文长度被限制在512字节之内，其中不包括IP首部或UDP首部。较长的DNS报文被截断，TCP字段在首部中被设置为1。

UDP是DNS的Internet标准查询的推荐方式，但不包括区域传输。使用UDP发送的查询可能丢失，因此需要考虑重传策略。查询或查询的应答可能由网络重新排序，或者经DNS服务器处理过，因此解析程序不能依赖按顺序返回的应答。

### 使用TCP进行传输
通过TCP传输的DNS报文使用两个字节长度字段做前缀。这个长度字段给出报文长度，计算长度不包括这个长度字段。该长度字段使得在开始解析报文之前，底层处理能够组装好完整的报文。

#### 为什么使用TCP
因为 DNS 响应报文中有一个**截断标志位**，用 TC 表示。当响应报文使用 **UDP 封装**，且报文长度大于 **512 字节**(其中不包括IP首部或UDP首部)时，那么服务器只返回前 512 字节，同时 TC 标志位置位，表示报文进行了截断。当客户端收到 TC 置位的响应报文后，将采用 **TCP 封装**查询请求。DNS 服务器返回的响应报文长度大于 512 字节。
![](attachments/Pasted%20image%2020240104133159.png)

当请求体和响应的大小比较小时，通过 TCP 协议进行传输不仅需要传输更多的数据，还会消耗更多的资源，多次通信以及信息传输带来的时间成本在 DNS 查询较小时是无法被忽视的，TCP 连接带来的可靠性在 DNS 的场景中没能发挥太大的作用。
在 DNS 中存储较多的内容时，TCP 三次握手以及协议头带来的额外开销就不是关键因素了。不过我们 TCP 三次握手带来的三次网络传输耗时还是没有办法避免的，这也是我们在目前的场景下不得不接受的问题。
- 当 DNS 数据包大小为 500 字节时，TCP 协议的额外开销为 ~41.2%；
- 当 DNS 数据包大小为 1100 字节时，TCP 协议的额外开销为 ~20.7%；
- 当 DNS 数据包大小为 2300 字节时，TCP 协议的额外开销为 ~10.3%；
- 当 DNS 数据包大小为 4800 字节时，TCP 协议的额外开销为 ~5.0%；
![](attachments/Pasted%20image%2020240108151643.png)

### 小结
重新回顾一下 DNS 查询选择 UDP 或者 TCP 两种不同协议时的主要原因：
- UDP 协议
    - DNS 查询的数据包较小、机制简单；
    - UDP 协议的额外开销小、有着更好的性能表现；
- TCP 协议
    - DNS 查询由于 DNSSEC 和 IPv6 的引入迅速膨胀，导致 DNS 响应经常超过 MTU 造成数据的分片和丢失，我们需要依靠更加可靠的 TCP 协议完成数据的传输；
    - 随着 DNS 查询中包含的数据不断增加，TCP 协议头以及三次握手带来的额外开销比例逐渐降低，不再是占据总传输数据大小的主要部分；

 DNS **查询**在刚设计时主要使用 UDP 协议进行通信，而 TCP 协议也是在 DNS 的演进和发展中被加入到规范的：
1. DNS 在设计之初就在区域传输中引入了 TCP 协议，在查询中使用 UDP 协议；
2. 当 DNS 超过了 512 字节的限制，我们第一次在 DNS 协议中明确了『当 DNS 查询被截断时，应该使用 TCP 协议进行重试』这一规范；
3. 随后引入的 EDNS 机制允许我们使用 UDP 最多传输 4096 字节的数据，但是由于 MTU 的限制导致的数据分片以及丢失，使得这一特性不够可靠；
4. 在最近的几年，我们重新规定了 DNS 应该同时支持 UDP 和 TCP 协议，TCP 协议也不再只是重试时的选择；

无论是选择 UDP 还是 TCP，最核心的矛盾就在于需要传输的数据包大小，如果数据包小到一定程度，UDP 协议绝对最佳的选择，但是当数据包逐渐增大直到突破 512 字节以及 MTU 1500 字节的限制时，我们也只能选择使用更可靠的 TCP 协议来传输 DNS 查询和响应。

参考：[为什么 DNS 使用 UDP 协议](https://draveness.me/whys-the-design-dns-udp-tcp/)
## 什么时候用TCP进行传送
DNS使用的通信方式，有UDP和TCP两种。一般情况下使用的是UDP进行DNS域名查询。但是，在以下两种情况会使用TCP进行域名查询：
![](attachments/Pasted%20image%2020231107195700.png)

1. 若客户端事先知道 DNS 响应报文的长度会大于 512 字节（其中不包括IP首部或UDP首部），则应当直接使用 TCP 建立连接
2. 若客户端事先不知道 DNS 响应报文的长度，一般会先使用 UDP 协议发送 DNS 查询报文，若 DNS 服务器发现 DNS 响应报文的长度大于 512 字节，则多出来的部分会被 UDP 抛弃(截断 TrunCation)，那么服务器会把这个部分被抛弃的 DNS 报文首部中的 TC 标志位置为 1，以通知客户端该 DNS 报文已经被截断。客户端收到之后会重新发起一次 TCP 请求，从而使得它将来能够从 DNS 服务器收到完整的响应报文。
> 当然了，在域名解析的时候，一般返回的 DNS 响应报文都不会超过 512 字节，用 UDP 传输即可。事实上，很多 DNS 服务器进行配置的时候，也仅支持 UDP 查询包。
4. 区域传输的过程，而在进行区域传输的时候 DNS 会强制使用 TCP 协议。


## DNS使用udp的响应数据不可以超过512B?
### 说明
阅读RFC1035，有这段描述：
```c
Messages carried by UDP are restricted to 512 bytes (not counting the IP or UDP headers). Longer messages are truncated and the TC bit is set in the header.
```
**512字节具体指UDP数据部分，没有UDP头部**。
需要注意UDP协议中的`Length`字段长度是包括UDP头部的。

### 流程
当使用UDP传输时，若响应数据超过DNS标准限制（超过512B），数据包便会发生截断，超出部分被丢弃，此时该flag位被置1。

当客户端发现TC位被置1的响应数据包时应该选择使用TCP重新发送查询。因为TCP DNS报文不受512字节限制。

小结：
DNS协议从UDP切换到TCP的过程如下：
```c
1、客户端向服务器发起UDP DNS请求；
2、如果服务器发现DNS响应数据超过512字节，则返回UDP DNS响应中置truncated位，告知客户端改用TCP进行重新请求；
3、客户端向服务器发起TCP DNS请求；
4、服务器返回TCP DNS响应。
```
### 原因
对于传输层，即 UDP 数据包本身来说，Length 字段为 16 位，理论限制为 65535 字节（2^16 - 1），那么能传输的数据为 65535 - IPHeader(20) - UDPHeader(8) = 65507 字节。

其次，对于网络层，以太网规定 MTU 上限为 1500 字节综合权衡的结果，如果按照 MTU = 1500 计算，那么 UDP 能传输的数据包上限为 MTU(1500) - IPHeader(20) - UDPHeader(8) = 1472 字节。
因此一般建议将 UDP 数据包限制在 1472 字节以下。但是呢，这个限制是在普通局域网（Ethernet v2）环境下的，在非局域网（X.25）环境下则有所不同。
> 因为 Internet 上的路由器可能会将 MTU 设为不同的值。如果我们假定 MTU 为 1500 字节来发送数据的，而途经的某个网络的 MTU 值小于 1500 字节，那么系统将会使用一系列的机制来调整 MTU 值 或者分片重组，使数据报能够顺利到达目的地，这样就会做许多不必要的操作。


鉴于互联网上物理链路的最小传输单元（MTU） = 576 字节，所以建议在进行 Internet 的 UDP 编程时，最好将 UDP 的数据长度控件在 548 字节（MTU(576) - IPHeader(20) - UDPHeader(8)）以内。
但这也还是 548 字节，并不是 512 字节。于是又搜索了一番，发现 StackOverflow 上有一个[回答](https://taifua.com/go/?url=aHR0cHM6Ly9zdGFja292ZXJmbG93LmNvbS9xdWVzdGlvbnMvMTA5ODg5Ny93aGF0LWlzLXRoZS1sYXJnZXN0LXNhZmUtdWRwLXBhY2tldC1zaXplLW9uLXRoZS1pbnRlcm5ldA%3D%3D)提到：
> 典型的 IPv4 头部是 20 字节，而 UDP 头部是 8 字节。然而，可以包括 IP 选项，该选项可以将 IP头部的大小增加到多达 60 字节。
> 此外，有时中间节点需要将数据报封装在另一种协议（如IPsec（用于VPN等））中，以便将数据包路由到其目的地。
> 因此，如果不知道特定网络路径上的 MTU，最好为可能没有预料到的其他头部信息留出合理的余量。512字节的 UDP 有效载荷通常被认为可以做到这一点，尽管即使这样也没有为最大尺寸的 IP报头留下足够的空间。

因此，通过这个回答我们可以知道，**UDP 数据包的最大安全负载应该是 508 字节**（MTU(576) - IPHeader(60) - UDPHeader(8)），因为 IP 头部最大时为 60 字节。512 也是一个综合考虑的结果。
**总的来说，这些数值的限制就是各层之间综合权衡的结果，以在整体上达到最优传输效率。当然，还得提一下，上述讨论针对的是 IPV4，而非 IPV6。**

### 解决方法
最大UDP DNS数据包设置为了512字节，而现如今许多查询的结果往往远超这一限制，有两种方法解决这一问题：
- 一个是TCP DNS
> RFC1035。当封装的DNS响应的长度超过512字节时，协议应采用TCP传输，而不是UDP。由于历史原因，TCP DNS在很多地方支持得并不算好。
- 另一个采用EDNS0扩展。
>  RFC 6891

### 范例
利用dig发送  
`dig @127.0.0.1 www.test.com +bufsize=4096 AAAA`
![](attachments/Pasted%20image%2020231108103519.png)

## DNS为什么查询根域名服务器只返回13个IP地址

## 什么是智能DNS
## 什么是EDNS

# 参考
```c
# DNS服务
https://linuxgeeks.github.io/2016/03/25/212131-DNS%E6%9C%8D%E5%8A%A1/

## 万字长文爆肝 DNS 协议！
https://mp.weixin.qq.com/s?__biz=MzI0ODk2NDIyMQ==&mid=2247487880&idx=1&sn=fd38ce30ae82fa7d08e5f83fabb9d497&chksm=e999e49adeee6d8c1adacbfe27dc59097e4cb9d39c6a04802b0fe61877653330e75721cbde0b&scene=21#wechat_redirect

# DNS体系架构最详解(图文)
https://wenku.baidu.com/view/f35c35d5240c844769eaee3a?pcf=2&bfetype=new&bfetype=new&_wkts_=1703214684668

# DNS 协议
https://github.com/crisxuan/bestJavaer/blob/master/computer-network/network-dns.md

# 36 张图详解 DNS ：网络世界的导航
https://www.sohu.com/a/705284462_121124376

# DNS多点部署IP Anycast+BGP实战分析
https://www.linuxidc.com/Linux/2014-08/105816.htm

# anycast 技术浅析
https://www.cnblogs.com/itzgr/p/10192799.html#_label3

# 阿里云中的DNS软件学习系列（++++++）
https://developer.aliyun.com/group/dns/softwares/article?spm=a2c6h.27925324.detail.196.3b23217cBHpSWs&pageNum=2
```