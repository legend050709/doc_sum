```table-of-contents
```
# 背景
​ 一般而言，DNS服务器是繁忙的 ，一台公网DNS会维护成千上万个zone，named服务不会轻易被重启，登陆dns服务器进行维护也会有极大的风险。所以需要对named进行远程管理。

# 简介
rndc（Remote Name Domain Controllerr）是一个远程管理bind的工具，通过这个工具可以在本地或者远程了解当前服务器的运行状况，也可以对服务器进行关闭、重载、刷新缓存、增加删除zone等操作。
rndc 可以运行在其他计算机上，通过网络与DNS服务器进行连接，然后根据管理员的指令对named进程进行远程控制，此时，管理员不需要DNS服务器的根用户权限。

# 流程
`rndc`  通过一个 TCP 连接与DNS服务器实行连接时，需要通过数字证书进行认证，而不是传统的用户名/密码方式。
`rndc` 在连接通道中发送命令时，必须使用经过服务器认可的密钥加密。
因此，使用`rndc`管理`bind`前需要使用`rndc`生成一对密钥文件，一个保存于`rndc`的配置文件中，另一个保存于 `bind` 主配置文件中。
> `rndc`的配置文件为`/etc/rndc.conf`，在CentOSL中，`rndc`的密钥保存在`/etc/rndc.key`文件中。
> 为了生成双方都认可的密钥，可以使用`rndc-confgen`命令产生密钥和相应的配置，再把这些配置分别放入`named.conf`和`rndc`的配置文件`rndc.conf`中。

**`rndc`默认监听在953号端口（TCP）**，其实在bind9中`rndc`默认就是可以使用，不需要配置密钥文件。

# 语法
```bash
# rndc --help
rndc: invalid argument --
Usage: rndc [-b address] [-c config] [-s server] [-p port]
	[-k key-file ] [-y key] [-r] [-V] command

command is one of the following:

  addzone zone [class [view]] { zone-options }
		Add zone to given view. Requires allow-new-zones option.
  delzone [-clean] zone [class [view]]
		Removes zone from given view.
  dnstap -reopen
		Close, truncate and re-open the DNSTAP output file.
  dnstap -roll count
		Close, rename and re-open the DNSTAP output file(s).
  dumpdb [-all|-cache|-zones|-adb|-bad|-fail] [view ...]
		Dump cache(s) to the dump file (named_dump.db).
  flush 	Flushes all of the server's caches.
  flush [view]	Flushes the server's cache for a view.
  flushname name [view]
		Flush the given name from the server's cache(s)
  flushtree name [view]
		Flush all names under the given name from the server's cache(s)
  freeze	Suspend updates to all dynamic zones.
  freeze zone [class [view]]
		Suspend updates to a dynamic zone.
  halt		Stop the server without saving pending updates.
  halt -p	Stop the server without saving pending updates reporting
		process id.
  loadkeys zone [class [view]]
		Update keys without signing immediately.
  managed-keys refresh [class [view]]
		Check trust anchor for RFC 5011 key changes
  managed-keys status [class [view]]
		Display RFC 5011 managed keys information
  managed-keys sync [class [view]]
		Write RFC 5011 managed keys to disk
  modzone zone [class [view]] { zone-options }
		Modify a zone's configuration.
		Requires allow-new-zones option.
  notify zone [class [view]]
		Resend NOTIFY messages for the zone.
  notrace	Set debugging level to 0.
  nta -dump
		List all negative trust anchors.
  nta [-lifetime duration] [-force] domain [view]
		Set a negative trust anchor, disabling DNSSEC validation
		for the given domain.
		Using -lifetime specifies the duration of the NTA, up
		to one week.
		Using -force prevents the NTA from expiring before its
		full lifetime, even if the domain can validate sooner.
  nta -remove domain [view]
		Remove a negative trust anchor, re-enabling validation
		for the given domain.
  querylog [ on | off ]
		Enable / disable query logging.
  reconfig	Reload configuration file and new zones only.
  recursing	Dump the queries that are currently recursing (named.recursing)
  refresh zone [class [view]]
		Schedule immediate maintenance for a zone.
  reload	Reload configuration file and zones.
  reload zone [class [view]]
		Reload a single zone.
  retransfer zone [class [view]]
		Retransfer a single zone without checking serial number.
  scan		Scan available network interfaces for changes.
  secroots [view ...]
		Write security roots to the secroots file.
  showzone zone [class [view]]
		Print a zone's configuration.
  sign zone [class [view]]
		Update zone keys, and sign as needed.
  signing -clear all zone [class [view]]
		Remove the private records for all keys that have
		finished signing the given zone.
  signing -clear <keyid>/<algorithm> zone [class [view]]
		Remove the private record that indicating the given key
		has finished signing the given zone.
  signing -list zone [class [view]]
		List the private records showing the state of DNSSEC
		signing in the given zone.
  signing -nsec3param hash flags iterations salt zone [class [view]]
		Add NSEC3 chain to zone if already signed.
		Prime zone with NSEC3 chain if not yet signed.
  signing -nsec3param none zone [class [view]]
		Remove NSEC3 chains from zone.
  signing -serial <value> zone [class [view]]
		Set the zones's serial to <value>.
  stats		Write server statistics to the statistics file.
  status	Display status of the server.
  stop		Save pending updates to master files and stop the server.
  stop -p	Save pending updates to master files and stop the server
		reporting process id.
  sync [-clean]	Dump changes to all dynamic zones to disk, and optionally
		remove their journal files.
  sync [-clean] zone [class [view]]
		Dump a single zone's changes to disk, and optionally
		remove its journal file.
  thaw		Enable updates to all dynamic zones and reload them.
  thaw zone [class [view]]
		Enable updates to a frozen dynamic zone and reload it.
  trace		Increment debugging level by one.
  trace level	Change the debugging level.
  tsig-delete keyname [view]
		Delete a TKEY-negotiated TSIG key.
  tsig-list	List all currently active TSIG keys, including both statically
		configured and TKEY-negotiated keys.
  validation [ yes | no | status ] [view]
		Enable / disable DNSSEC validation.
  zonestatus zone [class [view]]
		Display the current status of a zone.
```

其中：
`-c config-file`：指定`rndc`的配置文件，若不显式指定，则默认为`/etc/rndc.conf`。
`-k key-file`：指定`rndc`的`key`文件，若不显式指定，则默认为`/etc/rndc.key`。
`-s server`: 指定远程DNS服务器的地址。若不显示置顶，默认为`127.0.0.1`
`-p port`：指定`rndc`连接远程的 Port，若不显式指定，则默认为`953`。

```bash
语法格式：
rndc --> rndc (953/tcp)
rndc COMMAND

COMMAND:
    reload:             重载主配置文件和区域解析库文件
    reload zonename:    重载区域解析库文件
    retransfer zonename: 手动启动区域传送，而不管序列号是否增加
    notify zonename:    重新对区域传送发通知
    reconfig:           重载主配置文件
    querylog:           开启或关闭查询日志文件/var/log/message.可以详细到DNS查询的细节。生产中不建议打开，除非用于排错。
    trace:              递增debug一个级别
	    trace LEVEL:    指定使用的级别
    notrace：           将调试级别设置为 0
    flush：             清空DNS服务器的所有缓存记录
    freeze              关闭动态更新zone（主要是nsupdate动态更新）
    thaw                启用动态更新zone（主要是nsupdate动态更新）

比如：
	更新了配置`view`下的`zone`引导以及库文件配置的时候，不允许其他设备通过`nsupdate`远程动态更新某个`zone`。
```

# 生成rndc的key
## 原理
在当前版本下，rndc和named都只支持HMAC-MD5认证算法，在通信两端使用共享密钥。
它为命令请求和named的响应提供 TSIG类型的认证。所有经由通道发送的命令都必须被一个服务器所知道的 key_id 签名。
为了生成双方都认可的密钥，可以使用`rndc-confgen`命令产生密钥和相应的配置，再把这些配置分别放入`named.conf`和`rndc`的配置文件`rndc.conf`中。

如果`rndc-confgen`命令执行时卡住，是因为系统的熵值不足了。因为`rndc-confgen`命令默认会去`/dev/random`和`/dev/urandom`读取随机数生成密钥，第一顺序是`/dev/random`。

- `/dev/random`：从熵池中取随机数，如果熵池中的随机数被用尽，则阻塞相关进程
- `/dev/urandom`：从熵池中取随机数，如果熵池中的随机数被用尽，则用软件生成伪随机数

```bash
rndc-confgen -r /dev/urandom >/etc/rndc.conf
```

## 范例
执行命令`rndc-confgen`，生成rndc的key，内容如下：
```bash
[root@k8s-dns ~]# rndc-confgen -r /dev/urandom
# Start of rndc.conf
key "rndc-key" {
        algorithm hmac-md5;
        secret "PmY9ozjj3+pkKJ4NXLpIlQ==";
};

options {
        default-key "rndc-key";
        default-server 127.0.0.1;
        default-port 953;
};
# End of rndc.conf

# Use with the following in named.conf, adjusting the allow list as needed:
# key "rndc-key" {
#       algorithm hmac-md5;
#       secret "PmY9ozjj3+pkKJ4NXLpIlQ==";
# };
# 
# controls {
#       inet 127.0.0.1 port 953
#               allow { 127.0.0.1; } keys { "rndc-key"; };
# };
# End of named.conf
```

### 更新控制器设备上的 `rndc.conf` 配置文件
然后更新 `rndc.conf` 配置文件，将`rndc-confgen`生成的如下部分复制到`rndc.conf`文件中：
```bash
[root@k8s-dns ~]# cat /etc/rndc.conf 
# Start of rndc.conf
key "rndc-key" {
        algorithm hmac-md5;
        secret "waEmuOMU3OnrTsvdOBDQdQ==";
};

options {
        default-key "rndc-key";
        default-server 10.1.1.250;
        default-port 953;
};
```
**options段**
用于配置默认项。可以配置的指令包括default-server、default-port、default-key，表示当没有任何地方指定这些项的值时将采用这些默认值。

**server段**
用于配置待控制的dns服务器。
server关键字后接的是dns服务器的主机名或ip地址。在此段落内，可以配置的指令包括：key、port、addresses。
其中key表示连接此server时将使用该key进行配对；
port表示要连接的dns服务器的rndc端口号；
addresses指定要连接的dns服务器地址；当使用了该指令时将替代server关键字后的主机名或ip地址，addresses后可以紧跟着端口号。

**key段**
定义key值。只有两个指令，一个是algorithm，目前只支持hmac-md5，另一个指令是secret，表示该key段所使用的key。secret段加密的key可以使用rndc-confgen生成，只需使用不同的随机数即可。

以下是在172.16.10.16上设置的rndc.conf，用于控制172.16.10.9和172.16.10.15这两台dns服务器。
```bash
key "rndc-key" {  
        algorithm hmac-md5;  
        secret "QDCyDaU8El7quzv3vB3z9A==";  
};  
  
options {  
        default-key "rndc-key";  
        default-server 127.0.0.1;  
        default-port 953;  
};  
  
server localhost {  
    key    "rndc-key";  
};  
  
server 172.16.10.9 {  
    key  "rndc-key";  
    port 953;  
};  
  
server 172.16.10.15 {  
    key "rndc-key";  
    port 953;  
};   
  
# Use with the following in named.conf, adjusting the allow list as needed:  
# key "rndc-key" {  
#       algorithm hmac-md5;  
#       secret "QDCyDaU8El7quzv3vB3z9A==";  
# };  
#   
# controls {  
#       inet 127.0.0.1 port 953  
#        allow { 127.0.0.1; } keys { "rndc-key"; };  
# };  
# End of named.conf
```

### 更新受控制DNS服务器上的 `rndc.key` 配置文件
再然后更新DNS服务器上的`rndc.key`配置文件，将`1rndc-confgen`生成的如下部分复制到`rndc.key`文件中：
> 注：服务器中的 `/etc/named.conf` 配置中 include了 `rndc.key` 配置文件 。


```
[root@k8s-dns ~]# cat /etc/rndc.key 
key "rndc-key" {
      algorithm hmac-md5;
      secret "waEmuOMU3OnrTsvdOBDQdQ==";
};

controls {
      inet 10.1.1.250 port 953
              allow { 10.1.1.250;10.1.1.254; } keys { "rndc-key"; };
};
```
**注意**：这里要配置一下controls段的acl，限定好哪些主机可以使用`rndc`远程管理DNS服务。
# rndc使用

## 常用命令
>说明：`rndc`命令后面可以跟”-s”和”-p”选项连接到远程DNS服务器，以便对远程DNS服务器进行管理，但此时双方的密钥要一致才能正常连接。
>在设置`rndc.conf`时一定要注意`key`的名称和预共享密钥一定要和`named.conf`相同，否则`rndc`工具无法正常工作。

**选项**
```shell

-b source-address   绑定rndc客户端使用的源地址，因为一个网卡可有多个地址。
-c config-file      指定连接时使用的配置文件，而不是默认的/etc/rndc.conf。
-s server           指定要连接的服务器的IP地址。
-p port             指定要连接的服务器的端口。
-k key-file         指定连接时使用的密钥文件，而不是默认的/etc/rndc.key。
-y key-id           指定要使用的密钥标识，必须与服务器一致。
-v                  输出详细的日志信息。
```


**命令功能**
```bash
rndc status                    #查询DNS服务器状态
rndc reload                    #重新加载named.conf和新的域，但不会重新加载已存的域文件。
rndc reload zone_name          #重新加载指定区域  
```
```shell
reconfig                       #重读配置文件并加载新增的区域
querylog                       #关闭或开启查询日志， 查询日志会输出到  /var/log/message, 繁忙时，可能会瞬间增大 message 

dumpdb                          #将高速缓存转储到转储文件 (named_dump.db)
freeze                          #暂停更新所有动态zone
freeze zone [class [view]]      #暂停更新一个动态zone
flush   [view]                  #刷新服务器的所有高速缓存
flushname name                  #为某一视图刷新服务器的高速缓存
stats                           #将服务器统计信息写入统计文件中    /var/named/data/named_stats.txt
status                              #显示服务器状态。
stop                                #将暂挂更新保存到主文件并停止服务器
halt                                #停止服务器，但不保存暂挂更新
trace                               #打开debug, debug有级别的概念，每执行一次提升一次级别
trace LEVEL                         #指定 debug 的级别, trace 0 表示关闭debug
notrace                                 #将调试级别设置为 0
restart                                 #重新启动服务器（尚未实现）
addzone zone  [class [view]]   { zone-options } #增加一个zone
delzone  zone  [class [view]]                   #删除一个zone
```

范例：
```shell
# rndc  -s 192.168.10.11 reload   base07.com
```



## `rndc status`检查rndc管理状态
```shell
rndc status
```
## rndc 管理域

named 命令允许动态更新。一个区域可以设置成动态或静态。缺省值为静态。
要使一个区域成为动态区域，必须将关键字 allow-update 添加到bind配置文件中的该区域的部分中。allow-update 关键字指定一个因特网地址匹配列表，该列表定义允许提交更新的主机。

简单的理解，在zone配置中如果allow-update的值不是none，那么这个zone就是一个动态zone；反之，如果没有填写allow-update或者值为none，那么这个zone为静态static。

静态修改，即直接修改 zone文件。
动态修改，则是通过 nsupdate 修改，其实修改的是named中的内存，zone文件并没有修改。

###  `rndc reload`管理静态域
使用：
```bash
reload  重新载入配置文件和区文件  
用法：  
	rndc reload  
  
reload zone [class [view]]  
重新载入指定的区文件
```

在静态域修改区域数据库文件(zone文件)后（包含zone中的`serial number`），使用以下命令重新加载区域数据库配置。

```bash
zone "od.com" IN {  
	type master;  
	file "od.com.zone";  
	allow-update { none; };  
};
```
静态域zone文件,增、删、改一条记录后；
```bash
# rndc reload od.com  
zone reload up-to-date
```


### 管理动态态域
**前提条件**：
在区域配置文件中 添加 allow-update { acl; }; 表示根据acl指定策略进行动态更新。可填写ip地址。

**原理**
使用nsupdate等工具进行动态配置，不需要手动前滚serial number，自动通知辅DNS更新。



```bash
freeze [zone[class[view]]]   
	冻结一个动态更新区的更新.如果没有指定的区，那么就冻结所有区的动态更新.
	
	这就允许对一个动态更新的区进行手工编辑.它也会导致日志文件中的变化被同步到主服务器,然后删除日志文件.在区被冻结时，所有的动态更新尝试都会拒绝.  
  
thaw [zone[class[view]]]  
	解冻一个被冻结的动态更新区.
```

在动态域修改区域数据库文件后，使用以下命令冻结区域数据库文件配置。
```shell
[root@localhost ~]# rndc  freeze host.com
rndc: 'freeze' failed: permission denied
Flushing the zone updates to disk failed.
```
加载失败 检查权限问题，发现所属主和所属组都是`root`，应该修改权限为`named`。
```shell
[root@dns1 ~]# ll /var/named
total 1084
-rw-r--r-- 1 root  root   328 Jul 11 17:53 10.168.192.in-addr.arpa.zone
drwxrwx--- 2 named named   75 Jul 18 03:16 data
drwxrwx--- 2 named named   60 Jul 18 16:04 dynamic
-rw-r--r-- 1 root  root   425 Jul 15 15:42 host.com.zone
-rw-r--r-- 1 named named  698 Jul 16 10:02 host.com.zone.jnl
```
修改配置文件权限。
```shell
[root@dns1 ~]# chown named.named /var/named/host.com.zone
[root@dns1 ~]# chmod 660 /var/named/host.com.zone
```

重新冻结区域数据库文件；
```shell
[root@localhost ~]# rndc  freeze host.com
You have new mail in /var/spool/mail/root
```

重新加载区域配置文件；
```shell
[root@localhost ~]# rndc thaw host.com  
The zone reload and thaw was successful.
You have new mail in /var/spool/mail/root
```
#### 范例
```bash
zone "host.com" IN {  
	type master;  
	file "host.com.zone";  
	allow-update { 10.4.7.11; };  
};
```
动态（nsupdate）增、删、改一条记录后。
```bash
#rndc reload host.com  
rndc: 'reload' failed: dynamic zone
```
直接`reload`会报错，需要先`freeze`再`thaw`才行;
```bash
#rndc freeze host.com  
#rndc thaw host.com  
The zone reload and thaw was successful.
```

#### 使用规范
1》使用 `nsupdate` 进行 动态更新。
  （可选）使用 `rndc sync` 将内存中的信息保存到zone文件中。
2》`rndc freeze` 冻结动态更新。
3》手动更改 zone文件。
4》 `rndc thaw`：解冻动态更新，并且 reload 配置。

#### 其他

#### 问题和解决
**问题**
今天想在不关闭bind的情况下更新一下zone文件，用了`rndc reload`命令也都返回reload成功但是利用dig命令检测发现解析并没有被更改。后来用了 `rndc reload xxx.top` 提示
`rndc: reload failed: dynamic zone`。

**解决**
```bash
先暂停动态区域的更新以便进行reload
rndc freeze xxx.top

进行reload
rndc reload xxx.top

启用动态区域的更新并重新加载区域文件
rndc thaw xxx.top
```

## `rndc sync`将动态更新落盘到zone文件
Bind9支持区域记录的动态更新，使用nsupdate命令可以动态更新区域内的记录。

例如我们在test.com域中有如下内容：
```bash
$TTL 1D
@       IN SOA  dns.test.com. admin.test.com. (
                                        4       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      dns
        NS      dns2
ms      NS      dns2
tomcat  A       192.168.100.90
dns2    A       192.168.100.60
dns     A       192.168.100.50
Linux   A       192.168.200.30
www     A       192.168.100.20
web.test.com.   A       192.168.100.10
Nginx   CNAME   web.test.com.
```

我需要添加一条A记录进入该区域。可以使用nsupdate命令行工具。首先在主配置文件中允许某个主机动态更新区域内容。
```bash
zone "test.com" IN {
        type master;
        file "test.com.zones";
        allow-transfer { key test.com-key; };
        allow-update { 192.168.100.60; };    //配置allow-update项
};
```

在客户端使用nsupdate命令进行更新。
```bash
[root@dns2 ~]# nsupdate
> server 192.168.100.50
> zone test.com.
> update add LDAP.test.com. 86400 A 192.168.100.40
> send
```


查看zone文件所在的目录变化：
```bash
[root@dns1 named]# ll
······
-rw-r--r--  1 named named  499 Jun 16 04:59 test.com.zones
-rw-r--r--  1 named named  705 Jun 16 04:29 test.com.zones.jnl
-rw-r--r--  1 named named  499 Jun 16 04:51 tmp-28cGJKIgsY
······
```

直接查看原有的区域文件，发现内部并没有被改动，实际上的改动结果被储存在tmp开头的文件中，改动的信息被储存在`test.com.zones.jnl`文件（二进制文件不宜修改）中。**一般情况下在15分钟内，Bind会将`jnl`文件转储到区域文件中**。

此时添加进入的DNS记录是能够正常提供服务的(即 `named`的内存中存在更改后的记录)，如果需要实时更新到区域文件中，需要使用`rndc sync`且需要注意区域文件的文件权限。

注：执行完 `rndc sync`之后，区域日志文件（`test.ZONE_NAME.jnl` 文件）比如（`test.com.zones.jnl`文件）就没有必要要了，就可以删除了。`rndc sync clean`完成上诉一些列动作。

## 查询缓存
```bash
rndc dumpdb
```
## 清空DNS缓存
```bash
rndc
	  flush 	Flushes all of the server's caches.
	  flush [view]	Flushes the server's cache for a view.
	  flushname name [view]
			Flush the given name from the server's cache(s)
	  flushtree name [view]
			Flush all names under the given name from the server's cache(s)
```
线上难免会遇到刷新缓存的需求，如果直接用 rndc flush 刷新全量缓存，在有客户端缓存的情况下，在每一次客户端缓存过期的时间都可能会产生极高的 QPS 。

因此，尽量使用 flushname 或 flushtree 来刷新指定域名或 Zone。
# 范例
## 查询DNS服务状态（可以取值做监控）
```bash
# rndc 和 named 服务在一个机器上
rndc -c /etc/rndc.conf status
or 
rndc status
```

```bash
[root@k8s-dns ~]# rndc status
WARNING: key file (/etc/rndc.key) exists, but using default configuration file (/etc/rndc.conf)
version: 9.9.4-RedHat-9.9.4-72.el7 <id:8f9657aa>
CPUs found: 1
worker threads: 1
UDP listeners per interface: 1
number of zones: 3
debug level: 0
xfers running: 0
xfers deferred: 0
soa queries in progress: 0
query logging is OFF
recursive clients: 0/0/1000
tcp clients: 0/100
server is up and running
```

## 管理静态域(allow-update { none; };)
```json
zone "boy.com" IN {
    type master;
    file "boy.com.zone";
    allow-update { none; };
};
```

增、删、改一条记录后：
```bash
[root@k8s-dns ~]# rndc reload boy.com
zone reload up-to-date
```

## 管理动态域（allow-update { 10.1.1.250; };）
```json
zone "boysec.cn" IN {
    type master;
    file "boysec.cn.zone";
    allow-update { 10.1.1.250; };
};
```

增、删、改一条记录后：
```bash
[root@k8s-dns ~]# rndc reload boysec.cn
rndc: 'reload' failed: dynamic zone
```
直接`reload`会报错，需要先`freeze`再`thaw`才行
```bash
[root@k8s-dns ~]# rndc freeze boysec.cn
[root@k8s-dns ~]# rndc thaw boysec.cn
The zone reload and thaw was successful.
```
# 参考

```c
# Bind之rndc介绍及使用
http://www.hangdaowangluo.com/archives/1603

# Linux运维之bind9（DNS）远程管理rndc
https://cloud.tencent.com/developer/article/2272120
```