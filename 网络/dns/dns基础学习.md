```table-of-contents
```
# DNS介绍
![](attachments/Pasted%20image%2020240104113047.png)
DNS 的全称是 _Domain Name Systems_，它是一个由分层的 DNS 服务器实现的分布式数据库；它还是一个使得主机能够查询分布式数据库的应用层协议。DNS 协议运行在 UDP 协议上，使用 53 端口。

与 HTTP、FTP 和 SMTP 一样，DNS 协议也是一种应用层的协议，DNS 采用 client/server 模式，DNS client 发出查询请求，DNS server 响应请求，并通过 UDP 协议来传输 DNS 报文。
![](attachments/Pasted%20image%2020240104121432.png)

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

## 主从服务器的关系
![](attachments/Pasted%20image%2020240104121221.png)

设置域名服务器时，服务器管理员可以选择将域名服务器指定为主服务器还是辅助服务器(也称为从服务器)。

主域名服务器负责维护一个区域的所有域名信息，是特定的所有信息的权威信息源，数据可以修改。主服务器直接从本地文件获取此信息。只能在主服务器上更改区域的 DNS 记录，然后主服务器才能更新辅助服务器。

辅助域名服务器作为主域名服务器的备份提供域名解析服务，可以缓解主域名服务器的压力负载。辅助域名服务器中的区域文件中的数据是从主域名服务器中复制过来的，无法自行修改。
其实就是主从的概念，各位应该也都比较熟悉了。

**主域名服务器用来写，辅助域名服务器用来读，提供负载均衡的能力，缓解主域名服务器的压力。**

## 区域传输(zone transfer)
### 定义
那么所谓区域传输(zone transfer)呢，就是辅助域名服务器与主域名服务器通信，并同步 RR 资源的过程。这样做的目的是为了保证多台服务器保证内容同步。

### 传输协议
**区域传输使用 TCP 而不是 UDP**

如果使用UDP，限制传输数据 512B以内（DNS的UDP响应就是限制在512B以内）。由于数据同步传送的数据量比一个 DNS 请求和响应报文的数据量要多得多，因此使用TCP。
![](attachments/Pasted%20image%2020240104133505.png)

因此，**UDP 用于 client 和 server 的查询和响应**，**TCP 用于主从 server 之间的Zone传送**。

### 特性
**更改自动同步**
RFC 标准协议通过 MASTER-SLAVE 架构，NOTIFY + XFR 机制实现数据自动同步，用户只需要在主服务器上更改域名，更改信息便可自动同步到从服务器 。
![](attachments/Pasted%20image%2020240104162335.png)

### zone同步机制
#### 启动同步
当辅助域名服务器启动时，将从主域名服务器执行区域传送。
#### 定时检查
辅域名服务器会定时向主域名服务器进行查询以便了解区域是否有变动。如有变动，则会执行一次区域传输。

一旦启动区域传输，就会存在两种传输方式：
1. 全量传输：即传输整个区域的消息，全量传输会传输整个区域（使用 AXFR）的消息。
2. 增量传输：增量传输就是传输一部分消息，增量传输使用（使用 DNS IXFR）的消息。

#### DNS NOTIFY
但是使用轮询这种方式有一些弊端，因为从服务器会定期检查主服务器上内容是否更新，这是一种资源浪费，因为绝大多数情况下都是一次无效检查，所以为了改善这种情况，DNS 设计了 `DNS NOTIFY` 机制，`DNS NOTIFY` 允许修改区域内容后主服务器通知从服务器内容需要更新，应该启动区域传输。

### 具体流程
![](attachments/Pasted%20image%2020240104162652.png)

（1）用户在 MASTER 上动态修改域名解析记录（如 NSUPDATE），修改成功后，域名所在 ZONE 的版本号加 1。

`test.com`初始配置：
![](attachments/Pasted%20image%2020240104162738.png)

初始 SOA 序列号：
![](attachments/Pasted%20image%2020240104162805.png)

NSUPDTA 新增记录：
![](attachments/Pasted%20image%2020240104162815.png)

最新 SOA 序列号
![](attachments/Pasted%20image%2020240104162831.png)

（2）MASTER 向其配置的 SLAVE 节点发送 NOTIFY（一般是 UDP 报文），NOTIFY 信息中包含了修改域名所在的 ZONE 和该 ZONE 最新的版本号。

NOTIFY 消息：
![](attachments/Pasted%20image%2020240104162903.png)

（3）SLAVE 在收到 NOTIFY 消息后，进行以下操作：
- SLAVE 在收到 NOTIFY 消息后会给 MASTER 发送一个响应表示收到了 NOTIFY;
- SLAVE 比较 NOTIFY 中的 ZONE 的版本号和本地的 ZONE 的版本号，如果本地的版本号不低于 NOTIFY 中的版本号，SLAVE 不做任何操作;
- 如果 SLAVE 本地的版本号低于 NOTIFY 中的版本号，表示本地的 ZONE 数据已经落后，SLAVE 向 MASTER 发送 IXFR 请求; SLAVE 根据 REFRESH（定义在 ZONE 的 SOA 记录中）定时向 MASTER 发送 IXFR 请求，作为当 NOTIFY 的报文因为某些原因无法发送到 SLAVE 时的一种补偿机制。
- 如果 IXFR 失败，会转向 AXFR;

（4）MASTER 根据 SLAVE 请求的 XFR 类型返回对应的数据
IXFR 返回格式和结果：
![](attachments/Pasted%20image%2020240104163411.png)
![](attachments/Pasted%20image%2020240104163415.png)

AXFR 返回结果：
![](attachments/Pasted%20image%2020240104163511.png)

# 域名
域名由一系列字母（a～z，**不区分大小写**）、数字（0～9）、连接符（`-`）以及点号(`.`)分隔符组成，总长度不大于255。分隔符隔出的每段相当于一个层次的域名，级别低的在左，级别高的在右，每段长度不大于63。
如域名dailyupdate.wangwang.taobao.com，三段域名分别为dailyupdate、wangwang、taobao、com，其中com的级别最高。
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

其中顶级域名分为：国家顶级域名、通用顶级域名、反向域名。

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

## 子域名（Sub Domain Name）
DNS 有层次结构，TLD 下面可以有多个域名。例如，com 下面有 google.com 和 ubuntu.com。
”子域名” 是指作为较高层级域名的一部分。所以说，ubuntu.com 可以说是 com 的子域名，
每个域名可以控制它下面的子域名。


## FQDN
FQDN(Fully Qualified Domain Name)：完全合格域名/全程域名/完全限定域名/绝对域名。
即域名可以通过DNS进行解析，其公式 **FQDN = HostName + Domain，即全程域名=主机名+域**。
与 FQDN 对应的，系统中的默认域名是**非合格域名**，会把当前的区域域名添加到尾部。例如，tsinghua 域内的主机上查找 mail ，本地解析器就会将这个名称转换为 FQDN ，即 **mail.tsinghua.edu.cn**，然后解析出 IP 地址。
![](attachments/Pasted%20image%2020240104121831.png)

**FQDN**的完整格式是以点结尾的域名。
> DNS 系统中的域名可以是相对的，所以可能是模糊的。FQDN 是一个绝对名称，表示了它相对于域名系统中绝对根目录的位置。

这门技术解决了**一个域多个主机**的问题。
一个网站或者服务器集群一般都是有多个主机一起协作的。
比如说包括正向代理服务器、反向代理服务器、Web服务器、Email服务器、OA服务器、FTP服务器等等，这个时候就涉及需不需要为每一个主机申请一个域名。
有了这个技术之后每一个主机都可以自己申请一个 `Hostname` 来区别于其他的主机，这个时候就只需要一个域名就可以做到管理所有的主机。

比如我申请了一个域名: `doheras.com`
现在我有两个服务器需要用到这个域名，一个 FTP服务器，一个Web服务器，这两个服务器都需要用到 `doheras.com`这个域名，根据公式，我们知道可以采用 `hostname` 的方式来访问不同的主机：
Web 服务器: `web.doheras.com`
FTP 服务器: `ftp.doheras.com`

### 主机（Host）
你可以在一个域名下面定义其它主机。比如说，通过 api 主机(api.example.com) 允许 API 访问，通过 ftp 主机或者 files 主机(ftp.example.com 或者 files.example.com）允许 ftp 访问。主机名可以任意指定，只要它们在该域名下是唯一的。

主机名和子域名之间的区别是主机定义计算机或资源，而子域名扩展父域。

### 范例
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

|分类|作用|
|---|---|
|根域名服务器|最高层次的域名服务器，本地域名服务器解析不了的域名就会向其求助，从根域名服务器进行域名解析。|
|顶级域名服务器|负责管理在该顶级域名服务器下注册的二级域名。|
|权限（权威）域名服务器|负责一个区的域名解析工作。|
|本地域名服务器|当一个主机发出DNS查询请求时，这个查询请求首先发给本地域名服务器。|

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

# DNS解析流程
将域名转换为对应的 IP 地址 的过程叫做**域名解析**。在域名解析过程中，DNS client 的主机调用解析器 （ Resolver ），向 DNS server 发出请求，DNS server 完成域名解析。
![](attachments/Pasted%20image%2020240104122830.png)


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


## 反向查询
### 定义
在 DNS 查询中，客户端希望知道域名对应的 IP 地址，这种查询称为**正向查询**。大部分的 DNS 查询都是正向查询。与正向查询对应的，是**反向查询**。它允许 DNS 客户端通过 IP 地址查找对应的域名。
![](attachments/Pasted%20image%2020240104134440.png)

### 原理
为实现反向查询，在 DNS 标准中定义了特色域 in-addr.arpa 域，并保留在域名空间中，以便执行反向查询。为创建反向域名空间，in-addr.arpa 域中的子域是按照 IP 地址相反的顺序 构造的。
举个栗子：`www.tsinghua.edu.cn`的 IP 地址是 `166.111.4.100` ，那么在 in-addr.arpa 域中对应的节点就是 `100.4.111.166` 。
![](attachments/Pasted%20image%2020240104134548.png)

## 查询优先级
## 分级查询
# DNS应答
## 返回结果答案类别
- `有查询结果`（肯定答案）
- `不存在查询结果`（否定答案）

## 肯定答案分类
### 权威应答
- DNS服务器自己直接负责的域返回的答案
### 非权威应答
- DNS服务器未负责的域，由缓存或者查询到的记录返回的答案

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
SOA( Start of Authority)，起始授权机构记录：用来表示被标记成在众多NS记录中哪一台是主服务器。

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
**NS记录 和 SOA记录是任何一个DNS区域都不能或缺的两条记录**。
在 BIND 的配置中，可以在 Zone 文件中指定多个 Name Server，一般来说 Name Server 需要搭配 SOA 进行配置；

格式为：
```
[zone] IN NS [主机名称]
```
意思是：这个zone的查询请向后面这部主机请求。如果此zone有两部以上的DNS服务器负责时，就必须写两个NS，而NS后面接的主机名称必须要有ip的对应，这时需要A记录。

## A记录
格式为：
```json
[hostname] IN A [IP]
```

## AAAA记录

## PTR记录
## MX记录
```json
#优先级:0-99，数字越小，级别越高，
 
@ 600 IN MX 10 mail
@ 600 IN MX 20 smtp
```
## Cname记录
CNAME-records ( Canonical name for an alias )是域名的别名。
一个服务器的域名地址 nyc3.example.com 这个主机域名可能会提供不同的服务；
比如说域名 nyc3.example.com 指向的主机既要去提供 web服务，也要提供 ftp 服务，那么这个时候就需要一个别名域名来指向原来的域名，这个时候就可以使用一个 A 记录来管理多个域名，为DNS后期修改配置提供方便。

## SRV记录
SRV (Service)记录是从 RFC2052 中对 SRV资源进行了定义。SRV 被用来记录服务器提供什么样的服务。

## 其他
### SOA记录与NS记录的区别
NS记录和SOA记录是任何一个DNS区域都不可或缺的两条记录，NS记录表示域名服务器记录，用来指定该域名由哪些DNS服务器来进行解析；
SOA记录负责说明哪个DNS服务器是主服务器，以及主服务器和辅助服务器之间的一些关联参数。

假设hexun.com区域有两个DNS服务器负责解析，ns1.hexun.com是主服务器，ns2.hexun.com是辅助服务器，ns1.hexun.com的ip是202.99.16.1，ns2.hexun.com的ip是202.99.16.2。那么我们应该创建两条NS记录，当然，NS记录依赖A记录的解析，我们首先应该为ns1.hexun.com和ns2.hexun.com创建两条A记录。

NS记录说明了有两个DNS服务器（ns1.hexun.com 和 ns2.hexun.com）负责hexun.com的域名解析，但哪个是主服务器呢？NS记录并没有说明，这个任务由SOA记录来完成。

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

### 浏览器缓存
现如今的 Web 浏览器设计默认将 DNS 记录缓存一段时间。因为越靠近 Web 浏览器进行 DNS 缓存，为检查缓存并向 IP 地址发出请求的次数就越少。发出对 DNS 记录的请求时，浏览器缓存是针对所请求的记录而检查的第一个位置。

在 `chrome` 浏览器中，你可以使用 `chrome://net-internals/#dns` 查看 DNS 缓存的状态。这是基于 `Windows` 下查询的，我的 Mac 电脑输入上面 url 后无法查看 DNS ，只能 `clear host cache`。
![](attachments/Pasted%20image%2020231222105944.png)

### 操作系统内核缓存
在浏览器缓存查询后，会进行操作系统级 DNS 解析器的查询，操作系统级 DNS 解析器是 DNS 查询离开你的计算机前的第二站，也是本地查询的最后一个步骤。
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

# QA
## DNS 为什么同时使用 TCP 和 UDP
DNS 域名服务器使用的端口号是 53 ，并且同时支持 UDP 和 TCP 协议 。为什么同时使用两种协议呢？

因为 DNS 响应报文中有一个**截断标志位**，用 TC 表示。当响应报文使用 **UDP 封装**，且报文长度大于 **512 字节**时，那么服务器只返回前 512 字节，同时 TC 标志位置位，表示报文进行了截断。当客户端收到 TC 置位的响应报文后，将采用 **TCP 封装**查询请求。DNS 服务器返回的响应报文长度大于 512 字节。
![](attachments/Pasted%20image%2020240104133159.png)

## 什么时候用TCP进行传送
DNS使用的通信方式，有UDP和TCP两种。一般情况下使用的是UDP进行DNS域名查询。但是，在以下两种情况会使用TCP进行域名查询：
![](attachments/Pasted%20image%2020231107195700.png)

1. 若客户端事先知道 DNS 响应报文的长度会大于 512 字节，则应当直接使用 TCP 建立连接
2. 若客户端事先不知道 DNS 响应报文的长度，一般会先使用 UDP 协议发送 DNS 查询报文，若 DNS 服务器发现 DNS 响应报文的长度大于 512 字节，则多出来的部分会被 UDP 抛弃(截断 TrunCation)，那么服务器会把这个部分被抛弃的 DNS 报文首部中的 TC 标志位置为 1，以通知客户端该 DNS 报文已经被截断。客户端收到之后会重新发起一次 TCP 请求，从而使得它将来能够从 DNS 服务器收到完整的响应报文。
> 当然了，在域名解析的时候，一般返回的 DNS 响应报文都不会超过 512 字节，用 UDP 传输即可。事实上，很多 DNS 服务器进行配置的时候，也仅支持 UDP 查询包。
4. 区域传输的过程，而在进行区域传输的时候 DNS 会强制使用 TCP 协议。


## DNS使用udp的响应数据不可以超过512B?
### 说明
阅读RFC1035，有这段描述：
```c
Messages carried by UDP are restricted to 512 bytes (not counting the IP or UDP headers). Longer messages are truncated and the TC bit is set in the header.
```
512字节具体指UDP数据部分，没有UDP头部。需要注意UDP协议中的Length字段长度是包括UDP头部的。
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
```