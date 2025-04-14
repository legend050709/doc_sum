```table-of-contents
```

# 胶水记录
## 背景
### 循环依赖范例
我们使用 `dig` 命令来查询`weread.qq.com` 域名。
![](../attachments/Pasted%20image%2020240123110356.png)
整个解析过程大致的流程图如下，接下来我们对整个流程涉及到的步骤做详细解释。
![](../attachments/Pasted%20image%2020240123110404.png)

详细流程如下所示：
1. 客户端先去指定的 DNS 服务器（`8.8.8.8`，如果不指定，则使用系统配置的localdns）获取根服务器，`8.8.8.8`返回根服务器的域名列表（`a.root-servers.net`、`b.root-servers.net`等13个域名）。上图中的 1、2 步骤分别是获取根服务器的请求和响应。
2. 客户端从第2 步响应的根服务器列表中选取一个根服务器，向根服务器发送请求，请求内容为**查询域名`weread.qq.com` 的 A 记录**。根服务器当然没有这条记录（否则数据量会大到无法承受），但是根服务器知道 `com` 的DNS服务器（而且它只知道这些顶级域名的DNS服务器），于是返回给客户端`com` 的DNS服务器列表（`a.gtld-servers.net`、`b.gtld-servers.net`等13个域名），让客户端去`com` 的DNS服务器问一下。这就是上图中的 3、4 步骤。
3. 客户端从第 4 步的响应中选取一个 DNS 服务器（`c.gtld-servers.net`），然后向其询问**域名`weread.qq.com` 的 A 记录**。`com` 的DNS服务器也没有这条记录，于是返回 `qq.com` 的 DNS 服务器列表（列表为`ns1.qq.com`、`ns2.qq.com`等4个；即 **`qq.com` 的 DNS 服务器为其子域名**），让客户端去`qq.com` 的 DNS 服务器询问一下。这对应了上图的5、6 步骤。
4. 最后客户端拿着第6步中返回的DNS服务器中的一个（`ns4.qq.com`）向其发起请求，询问**域名`weread.qq.com` 的 A 记录**。而`ns4.qq.com`就是域名`qq.com`的权威DNS，它上面有记录`weread.qq.com`的A记录，因此它直接返回了期望域名的A记录值`43.159.233.225`。

这里面有个问题，第 6 步中因为 `com` DNS 服务器 `c.gtld-servers.net` 返回的 `qq.com` 的 DNS 服务器是域名列表（`ns4.qq.com`、`ns3.qq.com` 等；**`qq.com` 的 DNS 服务器为其子域名**）。
我们知道，互联网寻址是靠 IP 的，那就必须要把选中的 `ns4.qq.com` 服务器域名解析 IP，那此时流程就需要进入到解析 `ns4.qq.com` 的 IP 上了。
流程还是上面的流程，只是 `weread.qq.com` 换成 `ns4.qq.com`。按照迭代流程走到第6步， `com` DNS 服务器 `c.gtld-servers.net` 返回 `qq.com` DNS 服务器列表，为`ns4.qq.com`、`ns3.qq.com` 等 4 个域名，然后从中选择一个使用，比如选中 `ns1.qq.com` ，因为也是域名，需要解析为 IP，然后再次迭代查询。有没有发现已经死循环了，不管你循环多少次，都是在解析`ns1.qq.com`、`ns2.qq.com` 等 4 个域名，而且还得不到结果。

#### 解决死循环的胶水记录
我们先看一下第 6 步中，`c.gtld-servers.net` 的响应内容。
![](../attachments/Pasted%20image%2020240123111258.png)

从抓包中可以看到响应中还返回了`Additional records`，这些响应是对`Authoritative nameservers` 中的域名做补充，主要是补充了这些域名的IP地址，这样在第 7 步发起之前，就不需要再次解析 `ns4.qq.com` 的获取IP，从而避免了死循环的发生，而这些记录就是胶水记录。
![](../attachments/Pasted%20image%2020240123111347.png)

### 自建DNS时域名的注册
在讲这个概念之前，我们先理清一下域名的注册。

- 1、确定好公司域名，如`example.com.cn`，并向域名注册商注册
- 2、自建DNS服务器，可提供`example.com.cn`域的DNS解析服务，并对公网发布DNS服务器的地址（DNS服务器地址为`111.1.1.1`）
- 3、在域名注册商平台上配置权威DNS，也就是向外界通告，`example.com.cn`的域名，应由谁来做解析。（配置为111.1.1.1）
- 4、在自建的DNS服务器上，配置公司业务域名如www、mail等解析

通过以上4步，公司的域名解析服务即可生效。
但第3点有个问题：在配置权威DNS指向时，即向外界通告权威DNS服务器时，只能配置域名，而不能配置`111.1.1.1`。
如果是使用第三方的解析服务，那这个问题就不存在，域名解析服务商会告诉你，这个地方应该填哪几个域名。
对于自建DNS的公司来讲，这就存在一个鸡生蛋还是蛋生鸡的问题。假如我们在注册商那配置权威服务器为`ns.example.com.cn`，
并且在我们自己DNS服务器上也加一个A记录：`ns.example.com.cn` `111.1.1.1`

貌似就可以了，但仔细一看不然。因为此时此刻我们的DNS服务器还没有正式对外通告提供服务（即外界不知道我们的DNS服务器的地址），在公网上解析不出来`ns.example.com.cn`的地址，DNS服务器上有配置A记录也无法生效。但是注册商那又必须配置域名，而域名DNS服务器又无法工作，陷入死循环。

这个时候胶水记录就发挥了作用：在注册商那，创建一条胶水记录：
`ns.example.com.cn` `111.1.1.1`

这样，即使我们自有DNS服务器还未生效，外界也知道`ns.example.com.cn`指向的地址是`111.1.1.1`。胶水记录的名字也由此而来：相当于是用胶水把这个关联关系粘起来。

## 定义
![](../attachments/Pasted%20image%2020240123103914.png)
胶水记录的英文叫`Glue Record`。当前，大多数企业使用云解析，无需自建DNS服务器，很少会碰到这个概念。但如果你是自建DNS，那就必须掌握这个概念。
胶水记录是指在为域名配置NS记录时，**如果使用本zone下的域名来做解析服务器，必须在上级域指示出这个服务器对应的ip，如果使用其他地方的，则只要保证那个域名可以正常解析就好了。**


**方式方法**
通过在**域名上一级DNS服务器中登记域名NS以及其解析IP**，并在解析过程中通过`Additional records`返回，避免迭代查询陷入死循环。

**为啥要有胶水记录**？
胶水记录最主要的作用是为了解决域名解析过程中的一个循环依赖的问题。
因为DNS是从根节点开始逐层解析的，如果NS是它的子域名，不配置NS的A地址的话，解析过程便会陷入死循环。

**举例说明**：
在.com域下，想指示`xxx.com`这个域使用`a.xxx.com`作为dns服务器的话，必须写两条记录：
```bash
xxx IN NS a.xxx.com
a.xxx IN A 1.2.3.4
```

这样才能让客户端找到`a.xxx.com`这台机器，否则，会陷入死循环：
```text
告诉我谁负责xxx.com?   去找a.xxx.com!
a.xxx.com去是多少？    去找xxx.com的负责解析服务器！
谁负责xxx.com?   去找a.xxx.com!
……
```


# 理解胶水记录
## DNS查询过程
以查询 `jd.com` 这个域名的 A 记录为例，使用 `dig` 命令来分析整个查询过程。
我们会以递归服务器的工作方式来逐级查询 DNS 记录。

### 一、获取根服务器列表
为什么要根服务器列表？因为所有域名的 DNS 记录查询入口都在这里，就好像你要进入一个多级的目录，你得一级一级地进入，而根服务器就是入口。

全球共有 13 组公认的 DNS 服务器，所有递归服务器上都会内置这张表。

### 二、向根服务器发送查询
老衲随机选取一个根服务器进行查询，这里老衲选了 `m.root-servers.net`，它的IP地址是 `202.12.27.33`。

下面是和这台根服务器的对话过程：
“嗨，请问你这有 jd.com 吗？”
“没有，但是你可以找管理 .com 域名的这些服务器问问。哦，对了，他们的 IP 地址我这刚好有，直接给你吧！”

```bash
# dig @202.12.27.33 jd.com

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> @202.12.27.33 jd.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29264
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 13, ADDITIONAL: 27
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;jd.com.                                IN      A

;; AUTHORITY SECTION:
com.                    172800  IN      NS      d.gtld-servers.net.
com.                    172800  IN      NS      f.gtld-servers.net.
com.                    172800  IN      NS      e.gtld-servers.net.
com.                    172800  IN      NS      g.gtld-servers.net.
com.                    172800  IN      NS      k.gtld-servers.net.
com.                    172800  IN      NS      a.gtld-servers.net.
com.                    172800  IN      NS      c.gtld-servers.net.
com.                    172800  IN      NS      j.gtld-servers.net.
com.                    172800  IN      NS      l.gtld-servers.net.
com.                    172800  IN      NS      b.gtld-servers.net.
com.                    172800  IN      NS      m.gtld-servers.net.
com.                    172800  IN      NS      i.gtld-servers.net.
com.                    172800  IN      NS      h.gtld-servers.net.

;; ADDITIONAL SECTION:
a.gtld-servers.net.     172800  IN      A       192.5.6.30
b.gtld-servers.net.     172800  IN      A       192.33.14.30
c.gtld-servers.net.     172800  IN      A       192.26.92.30
d.gtld-servers.net.     172800  IN      A       192.31.80.30
e.gtld-servers.net.     172800  IN      A       192.12.94.30
f.gtld-servers.net.     172800  IN      A       192.35.51.30
g.gtld-servers.net.     172800  IN      A       192.42.93.30
h.gtld-servers.net.     172800  IN      A       192.54.112.30
i.gtld-servers.net.     172800  IN      A       192.43.172.30
j.gtld-servers.net.     172800  IN      A       192.48.79.30
k.gtld-servers.net.     172800  IN      A       192.52.178.30
l.gtld-servers.net.     172800  IN      A       192.41.162.30
m.gtld-servers.net.     172800  IN      A       192.55.83.30
a.gtld-servers.net.     172800  IN      AAAA    2001:503:a83e::2:30
b.gtld-servers.net.     172800  IN      AAAA    2001:503:231d::2:30
c.gtld-servers.net.     172800  IN      AAAA    2001:503:83eb::30
d.gtld-servers.net.     172800  IN      AAAA    2001:500:856e::30
e.gtld-servers.net.     172800  IN      AAAA    2001:502:1ca1::30
f.gtld-servers.net.     172800  IN      AAAA    2001:503:d414::30
g.gtld-servers.net.     172800  IN      AAAA    2001:503:eea3::30
h.gtld-servers.net.     172800  IN      AAAA    2001:502:8cc::30
i.gtld-servers.net.     172800  IN      AAAA    2001:503:39c1::30
j.gtld-servers.net.     172800  IN      AAAA    2001:502:7094::30
k.gtld-servers.net.     172800  IN      AAAA    2001:503:d2d::30
l.gtld-servers.net.     172800  IN      AAAA    2001:500:d937::30
m.gtld-servers.net.     172800  IN      AAAA    2001:501:b1f9::30

;; Query time: 211 msec
;; SERVER: 202.12.27.33#53(202.12.27.33)
;; WHEN: Thu Aug 01 18:13:03 CST 2019
;; MSG SIZE  rcvd: 831
```

根服务器告诉老衲，`.com` 由 `[a-m].gtld-servers.net` 这组服务器管理，这组服务器实际上就是 `.com` 域名的权威服务器，并且还顺便告诉了老衲他们的 IP 地址。

### 三：向 .com 权威服务器发送查询
老衲随机选取一个 `.com` 权威服务器进行查询，这里老衲选了 `a.gtld-servers.net`，它的IP地址是 `192.5.6.30`。

下面是和这台 `.com` 权威服务器的对话过程：
“嗨，请问你这有 jd.com 吗？”
“没有，但是你可以找管理 jd.com 域名的这些服务器问问。哦，对了，他们的 IP 地址我这刚好有，直接给你吧！”

```bash
# dig @192.5.6.30 jd.com  

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> @192.5.6.30 jd.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16955
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 4, ADDITIONAL: 5
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;jd.com.                                IN      A

;; AUTHORITY SECTION:
jd.com.                 172800  IN      NS      ns1.jdcache.com.
jd.com.                 172800  IN      NS      ns2.jdcache.com.
jd.com.                 172800  IN      NS      ns3.jdcache.com.
jd.com.                 172800  IN      NS      ns4.jdcache.com.

;; ADDITIONAL SECTION:
ns1.jdcache.com.        172800  IN      A       111.13.28.10
ns2.jdcache.com.        172800  IN      A       111.206.226.10
ns3.jdcache.com.        172800  IN      A       120.52.149.254
ns4.jdcache.com.        172800  IN      A       106.39.177.32

;; Query time: 3 msec
;; SERVER: 192.5.6.30#53(192.5.6.30)
;; WHEN: Thu Aug 01 18:20:21 CST 2019
;; MSG SIZE  rcvd: 179
```

`.com` 权威服务器告诉老衲，`jd.com` 由 `ns[1-4].jdcache.com` 这组服务器管理，并且还顺便告诉了老衲他们的 IP 地址。

### 第四步：向 jd.com 域名权威服务器发送查询
老衲随机选取一个 `jd.com` 权威服务器进行查询，这里老衲选了 `ns1.jdcache.com`，它的IP地址是 `111.13.28.10`。

下面是和这台 `jd.com` 权威服务器的对话过程：
“嗨，请问你这有 jd.com 吗？”
“有的，就在我这里，给你吧！”

```bash
# dig @111.13.28.10 jd.com

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> @111.13.28.10 jd.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53607
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 8, ADDITIONAL: 9
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;jd.com.                                IN      A

;; ANSWER SECTION:
jd.com.                 120     IN      A       120.52.148.118

;; AUTHORITY SECTION:
jd.com.                 120     IN      NS      ns3.jd.com.
jd.com.                 120     IN      NS      ns3.jdcache.com.
jd.com.                 120     IN      NS      ns1.jdcache.com.
jd.com.                 120     IN      NS      ns1.jd.com.
jd.com.                 120     IN      NS      ns2.jdcache.com.
jd.com.                 120     IN      NS      ns2.jd.com.
jd.com.                 120     IN      NS      ns4.jdcache.com.
jd.com.                 120     IN      NS      ns4.jd.com.

;; ADDITIONAL SECTION:
ns1.jd.com.             120     IN      A       111.13.28.10
ns1.jdcache.com.        720     IN      A       111.13.28.10
ns2.jd.com.             120     IN      A       111.206.226.10
ns2.jdcache.com.        720     IN      A       111.206.226.10
ns3.jd.com.             120     IN      A       120.52.149.254
ns3.jdcache.com.        720     IN      A       120.52.149.254
ns4.jd.com.             120     IN      A       106.39.177.32
ns4.jdcache.com.        720     IN      A       106.39.177.32

;; Query time: 201 msec
;; SERVER: 111.13.28.10#53(111.13.28.10)
;; WHEN: Thu Aug 01 18:23:53 CST 2019
;; MSG SIZE  rcvd: 331
```
最终老衲得到了 `jd.com` 的 IP 地址为 `120.52.148.118`。

## 胶水记录在哪
在上面这部分查询过程中，其实有多台服务器返回了胶水记录：

1. 在第二步向根服务器查询返回的结果里，根服务器顺便告诉老衲的包含 `*.gtld-servers.net` 这组服务器的 IP 地址的记录，就是胶水记录。
2. 在第三步向 `.com` 权威服务器查询返回的结果里，`.com` 权威服务器顺便告诉老衲的包含 `ns*.jdcache.com` 这组服务器的 IP 地址的记录，就是胶水记录。

所以，胶水记录就是域管理者向上级域管理者提供的一组主机名和IP的映射表：

1. `.com` 域管理者（也就是 `.com` 域名注册局，VeriSign）向根域管理者（也就是 IANA）提供了：`.com` 的权威 DNS 服务器 `*.gtld-servers.net` 以及 对应的 IP 地址。
2. `jd.com` 域管理者向`.com` 域管理者（VeriSign）提供了：`jd.com` 的权威 DNS 服务器 `ns*.jdcache.com`以及 对应的 IP 地址

# 设置胶水记录
## 什么时候需要设置胶水记录
那什么时候需要设置胶水记录呢？
域名的DNS服务器为该域名的子域名。则需要在该域名的父域名的注册局设置该域名的NS记录以及NS对应的A记录。

举例来说，如果你想将 `example.com` 的 DNS 服务器修改为`ns1.example.com` 和 `ns2.example.com` 那么你需要在 example.com 的注册商的自定义DNS处（一般为`com`对应的域名服务器）添加`ns1.example.com` 和 `ns2.example.com` 及其对应的 IP，否则会修改不成功。

## 胶水记录生效的前提
胶水记录设置后，能够减少递归查询次数。
但是一个生效的前提：
**“域名的父级域”和“域名的 DNS 服务器域名的父级域”使用同一组权威服务器。**

比如 `jd.com` 的权威服务器为 `ns*.jdcache.com`，而且 `jd.com` 和 `ns*.jdcache.com` 的父级域都是 `.com`，权威服务器都是 `*.gtld-servers.net`。

这样在向 `.com` 权威服务器 `*.gtld-servers.net` 查询 `jd.com` 时，服务器才能在返回 `ns*.jdcache.com` 的同时，顺便返回登记在案的 `ns*.jdcache.com` 的 IP 地址。

### 范例
**如果没有满足这个前提条件，胶水记录还会生效吗？**

来看看 `laona.dev`，老衲将它托管在 DNSPod 解析：
- 域名的DNS服务器：`f1g1ns1.dnspod.net`、`f1g1ns2.dnspod.net`
- 域名的父级域：`.dev`，权威服务器为 `ns-tld*.charlestonroadregistry.com`
- DNS服务器域名的父级域：`.net`，权威服务器为 `*.gtld-servers.net`

由上可知，两组权威服务器不同（即域名的父域名的权威服务器，以及域名的DNS服务器的父域名的权威服务器）。
老衲用 `dig @ns-tld1.charlestonroadregistry.com laona.dev` 查查看是否返回了 DNSPod 的胶水记录：
> 即：使用域名的父域名的权威服务器查询域名。

```bash
# dig @ns-tld1.charlestonroadregistry.com laona.dev

; <<>> DiG 9.11.5-P1-1ubuntu2.5-Ubuntu <<>> @ns-tld1.charlestonroadregistry.com laona.dev
; (2 servers found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4730
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 2, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;laona.dev.                     IN      A

;; AUTHORITY SECTION:
laona.dev.              10800   IN      NS      f1g1ns1.dnspod.net.
laona.dev.              10800   IN      NS      f1g1ns2.dnspod.net.

;; Query time: 61 msec
;; SERVER: 216.239.32.105#53(216.239.32.105)
;; WHEN: Thu Aug 01 19:21:42 CST 2019
;; MSG SIZE  rcvd: 92
```

如同所料，`.dev` 的权威服务器 `ns-tld*.charlestonroadregistry.com` 中并没有保存 `dnspod.net` 的胶水记录，因此只返回了负责解析 `laona.dev` 的权威服务器名称，没有返回这组权威服务器的 IP 地址。

那么怎么得到这组权威服务器的 IP 地址呢？通过查询对应的 A 记录获得。
## 什么时候需要修改胶水记录
由于胶水记录绑定的是权威DNS服务器的域名和IP的关系，因此如果权威DNS服务器的域名或者IP有变化时，需要在上级服务器上删除或者修改胶水记录。

即：**域名的NS记录或者对应的IP地址存在变化时，也要考虑修改对应的胶水记录**。

# 胶水记录的作用
## 主要作用
 如果：没有设置这些胶水记录；那么：这些服务器就无法**顺便**返回给老衲这些DNS服务器IP；结果：老衲要想得到这些DNS服务器的IP，就得额外查询。
 
所以胶水记录的作用就是：
- **解决域名解析过程中的一个循环依赖的问题**；
- **减少递归查询次数，加快 DNS 递归查询**。


# 如何查询胶水记录
有多种方法可以查询域名是否设置了胶水记录，以及其对应的 IP。

## dig查询
通过上面抓包我们可以看到，胶水记录就放在响应的 `Additional records` 中，我们只需要从里面提取即可，好在 `dig` 命令已经实现了此能力，以刚刚的`qq.com` 域名为例。
```shell
dig @m.gtld-servers.net. qq.com
```
![](../attachments/Pasted%20image%2020240123112813.png)

因为胶水记录是由上一级 DNS 服务器返回的，因此 `dig` 时使用的 DNS 服务器需要是上一级的。`qq.com` 的上一级是注册局是`m.gtld-servers.net`，因此需要指定注册局的 DNS 服务器地址`m.gtld-servers.net`。

# 注意
## 胶水记录的IP和域名A记录的区别
NS记录用以授权**子域**，和NS记录一起用以授权的还有glue记录，glue记录用以指定子域名称服务器的地址，但它不是这个域本身的权威数据。

需要注意的是域名设置的胶水记录IP 值与该域名解析出来的 IP 有可能不同，因为两个的数据来源不同，但是建议最好设置为相同。
## 范例
### dns glue引起的异常
近期内部开发反馈某些合作方的域名无法解析。团内同事分析发现这些域名都是托管在相同的一个域名厂商上,而且都是刷新`cache`后刚开始能解析，过段时间不能解析。如下的2个域名：
```bash
efly.cc  
bhc888.net
```

测试如下：
```bash
# dig efly.cc
; &lt;&lt;&gt;&gt; DiG 9.9.5-3ubuntu0.5-Ubuntu &lt;&lt;&gt;&gt; efly.cc
;; global options: +cmd
;; Got answer:
;; -&gt;&gt;HEADER&lt;&lt;- opcode: QUERY, status: NOERROR, id: 7761
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;efly.cc.           IN  A

;; ANSWER SECTION:
efly.cc.        600 IN  A   121.9.13.185

;; AUTHORITY SECTION:
efly.cc.        168802  IN  NS  ns2.eflydns.net.
efly.cc.        168802  IN  NS  ns1.eflydns.net.

;; Query time: 1356 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Sun Nov 29 19:00:23 CST 20
```

```bash
# dig bhc888.net +trace +all

; &lt;&lt;&gt;&gt; DiG 9.9.5-3ubuntu0.5-Ubuntu &lt;&lt;&gt;&gt; bhc888.net +trace +all
;; global options: +cmd
;; Got answer:
;; -&gt;&gt;HEADER&lt;&lt;- opcode: QUERY, status: NOERROR, id: 24539
;; flags: qr ra; QUERY: 1, ANSWER: 14, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags: do; udp: 4096
;; QUESTION SECTION:
;.              IN  NS

;; ANSWER SECTION:
.           347738  IN  NS  m.root-servers.net.
.           347738  IN  NS  g.root-servers.net.
.           347738  IN  NS  h.root-servers.net.
.           347738  IN  NS  c.root-servers.net.
.           347738  IN  NS  e.root-servers.net.
.           347738  IN  NS  d.root-servers.net.
.           347738  IN  NS  k.root-servers.net.
.           347738  IN  NS  l.root-servers.net.
.           347738  IN  NS  a.root-servers.net.
.           347738  IN  NS  f.root-servers.net.
.           347738  IN  NS  b.root-servers.net.
.           347738  IN  NS  j.root-servers.net.
.           347738  IN  NS  i.root-servers.net.
.           518045  IN  RRSIG   NS 8 0 518400 20151209050000 20151129040000 62530 . EtQ9uRmWHEfzpE2KROfPA2LcYyde+z1YKDWRbfJBQebQ0S17h8FirKlu uaQFloFKfekxT+K6YsirfivvGlO2v4qcF6XvLMhsLinlJj/6+3DG7od/ ELN3wHTTUJOchLcQTkSW2BxalK5SWP0mRXhCo7TLro8S6C893n2uYWhK SzY=

;; Query time: 5 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Sun Nov 29 21:51:47 CST 2015
;; MSG SIZE  rcvd: 397

;; Got answer:
;; -&gt;&gt;HEADER&lt;&lt;- opcode: QUERY, status: NOERROR, id: 57915
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 15, ADDITIONAL: 16

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags: do; udp: 4096
;; QUESTION SECTION:
;bhc888.net.            IN  A

;; AUTHORITY SECTION:
net.            172800  IN  NS  a.gtld-servers.net.
net.            172800  IN  NS  b.gtld-servers.net.
net.            172800  IN  NS  c.gtld-servers.net.
net.            172800  IN  NS  d.gtld-servers.net.
net.            172800  IN  NS  e.gtld-servers.net.
net.            172800  IN  NS  f.gtld-servers.net.
net.            172800  IN  NS  g.gtld-servers.net.
net.            172800  IN  NS  h.gtld-servers.net.
net.            172800  IN  NS  i.gtld-servers.net.
net.            172800  IN  NS  j.gtld-servers.net.
net.            172800  IN  NS  k.gtld-servers.net.
net.            172800  IN  NS  l.gtld-servers.net.
net.            172800  IN  NS  m.gtld-servers.net.
net.            86400   IN  DS  35886 8 2 7862B27F5F516EBE19680444D4CE5E762981931842C465F00236401D 8BD973EE
net.            86400   IN  RRSIG   DS 8 1 86400 20151209050000 20151129040000 62530 . mu4PiPAwAMZ/X2wUCQTXZwwCiO9/hwlvB8sbg73q5a9jyaYnWPjpIMh2 1wJWzE2Xc+5+/VxE3uLzhALqfnvto0ACN4UlyXESJ2qiVc2k69PQ54hh 8PZO4b5CzkfG09bqccLJuGcyLuMacYSc4w1LmiSq329tk7OYZw09P2YG 0RU=

;; ADDITIONAL SECTION:
a.gtld-servers.net. 172800  IN  A   192.5.6.30
b.gtld-servers.net. 172800  IN  A   192.33.14.30
c.gtld-servers.net. 172800  IN  A   192.26.92.30
d.gtld-servers.net. 172800  IN  A   192.31.80.30
e.gtld-servers.net. 172800  IN  A   192.12.94.30
f.gtld-servers.net. 172800  IN  A   192.35.51.30
g.gtld-servers.net. 172800  IN  A   192.42.93.30
h.gtld-servers.net. 172800  IN  A   192.54.112.30
i.gtld-servers.net. 172800  IN  A   192.43.172.30
j.gtld-servers.net. 172800  IN  A   192.48.79.30
k.gtld-servers.net. 172800  IN  A   192.52.178.30
l.gtld-servers.net. 172800  IN  A   192.41.162.30
m.gtld-servers.net. 172800  IN  A   192.55.83.30
a.gtld-servers.net. 172800  IN  AAAA    2001:503:a83e::2:30
b.gtld-servers.net. 172800  IN  AAAA    2001:503:231d::2:30

;; Query time: 344 msec
;; SERVER: 128.63.2.53#53(128.63.2.53)
;; WHEN: Sun Nov 29 21:51:47 CST 2015
;; MSG SIZE  rcvd: 731

;; Got answer:
;; -&gt;&gt;HEADER&lt;&lt;- opcode: QUERY, status: NOERROR, id: 64484
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 6, ADDITIONAL: 5

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags: do; udp: 4096
;; QUESTION SECTION:
;bhc888.net.            IN  A

;; AUTHORITY SECTION:
bhc888.net.     172800  IN  NS  ns1.eflydns.net.
bhc888.net.     172800  IN  NS  ns2.eflydns.net.
A1RT98BS5QGC9NFI51S9HCI47ULJG6JH.net. 86400 IN NSEC3 1 1 0 - A1RUUFFJKCT2Q54P78F8EJGJ8JBK7I8B NS SOA RRSIG DNSKEY NSEC3PARAM
A1RT98BS5QGC9NFI51S9HCI47ULJG6JH.net. 86400 IN RRSIG NSEC3 8 2 86400 20151206063020 20151129052020 37703 net. QdTw71NidYfASViPME8hIX6IixUOqawLJgDF94/Z50pGN+V8mynVueuA 7sIYDinnSdZnkxIOUH284tZtfZRnUutLjocnd7YDb7hTqPSoP4QZij6A 8O7hGW+PRj/hRHJKhB7SN7aE6LN2zV+P6jLXLsTZmRnKBKAqzt+5/ZMe 23A=
K6E8QG8SUT2RJS20VQD9AQ0EQGOEVT99.net. 86400 IN NSEC3 1 1 0 - K6FGOS2E26R647F6LEEJI146DBAJE0PT NS DS RRSIG
K6E8QG8SUT2RJS20VQD9AQ0EQGOEVT99.net. 86400 IN RRSIG NSEC3 8 2 86400 20151206062959 20151129051959 37703 net. FxrolX/ogsqiCtZFd7KLBBfC9MibFkiFuIrTt9RTM+7RblfH6ZpgkxUD /oewDTkYarIMFNii+ABM+V9+fXDGszmSY4plFvTzfR7X5eiJWOVndvs2 ph8KubUiYd79+vCXkiHw86ILy1OEk3X79uhunpAO4lIaRwIq5TSQpjs+ KcY=

;; ADDITIONAL SECTION:
ns1.eflydns.net.    172800  IN  A   121.201.11.2
ns1.eflydns.net.    172800  IN  A   121.201.54.215
ns2.eflydns.net.    172800  IN  A   121.201.11.2
ns2.eflydns.net.    172800  IN  A   121.201.54.215

;; Query time: 201 msec
;; SERVER: 192.55.83.30#53(192.55.83.30)
;; WHEN: Sun Nov 29 21:51:48 CST 2015
;; MSG SIZE  rcvd: 632

;; Got answer:
;; -&gt;&gt;HEADER&lt;&lt;- opcode: QUERY, status: NOERROR, id: 33677
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 0
;; WARNING: recursion requested but not available
;; WARNING: Message has 8 extra bytes at end

;; QUESTION SECTION:
;bhc888.net.            IN  A

;; ANSWER SECTION:
bhc888.net.     600 IN  A   14.17.121.64

;; AUTHORITY SECTION:
bhc888.net.     600 IN  NS  ns1.eflydns.net.
bhc888.net.     600 IN  NS  ns2.eflydns.net.

;; Query time: 41 msec
;; SERVER: 121.201.12.66#53(121.201.12.66)
;; WHEN: Sun Nov 29 21:51:48 CST 2015
;; MSG SIZE  rcvd: 96
```

在trace内容中可以看到GLUE记录里的和实际的NS ip不一致。  
glue记录显示，如下所示：
```bash
;; ADDITIONAL SECTION:
ns1.eflydns.net.    172800  IN  A   121.201.11.2
ns1.eflydns.net.    172800  IN  A   121.201.54.215
ns2.eflydns.net.    172800  IN  A   121.201.11.2
ns2.eflydns.net.    172800  IN  A   121.201.54.215
```
但是由上所示，实际连接的DNS服务器的IP地址是`121.201.12.66`。
实际`ADDITIONAL`中的这2个IP都是不通的。很多人不清楚修改NS等需要同步改GLUE记录，就出现了这样的问题，去年当当网也出现过一次比较严重的故障。

# 参考
```bash
# 什么是胶水记录
https://kdefan.net/posts/what-is-glue-records.html

# 胶水记录（Glue Record）是什么？有什么作用？
https://laona.dev/post/glue-record/
```