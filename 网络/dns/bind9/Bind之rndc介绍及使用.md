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

`rndc`默认监听在953号端口（TCP），其实在bind9中`rndc`默认就是可以使用，不需要配置密钥文件。
# 使用
## 生成rndc的key
### 原理
在当前版本下，rndc和named都只支持HMAC-MD5认证算法，在通信两端使用共享密钥。
它为命令请求和named的响应提供 TSIG类型的认证。所有经由通道发送的命令都必须被一个服务器所知道的 key_id 签名。
为了生成双方都认可的密钥，可以使用`rndc-confgen`命令产生密钥和相应的配置，再把这些配置分别放入`named.conf`和`rndc`的配置文件`rndc.conf`中。

如果`rndc-confgen`命令执行时卡住，是因为系统的熵值不足了。因为`rndc-confgen`命令默认会去`/dev/random`和`/dev/urandom`读取随机数生成密钥，第一顺序是`/dev/random`。

- `/dev/random`：从熵池中取随机数，如果熵池中的随机数被用尽，则阻塞相关进程
- `/dev/urandom`：从熵池中取随机数，如果熵池中的随机数被用尽，则用软件生成伪随机数

```bash
rndc-confgen -r /dev/urandom >/etc/rndc.conf
```

### 范例
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

- **更新控制器设备上的 `rndc.conf` 配置文件**
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

- **更新DNS服务器上的 `rndc.key` 配置文件**
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
## rndc使用
### 语法
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

### 常用命令
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



#### 检查rndc管理状态
```shell
rndc status
```
####  rndc 管理静态域
在静态域修改区域数据库文件(zone文件)后（包含zone中的`serial number`），使用以下命令重新加载区域数据库配置。
如下，修改了`whbblog.cn`对应的zone文件之后，使用下列方式进行加载。
```shell
[root@localhost ~]# rndc reload whbblog.cn
zone reload up-to-date
You have new mail in /var/spool/mail/root
```

#### rndc 管理动态态域
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

### 范例
#### 查询DNS服务状态（可以取值做监控）
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

#### 管理静态域(allow-update { none; };)
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

#### 管理动态域（allow-update { 10.1.1.250; };）
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