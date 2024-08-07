```table-of-contents
```
# 简介
Bind是`Berkeley Internet Name Domain Service`的简写，它是一款实现DNS 服务器的开放源码软件。已经成为世界上使用最为广泛的DNS服务器软件，目前Internet上半数以上的DNS服务器有都是用Bind来架设的，已经成为DNS中事实上的标准。

bind 目前在 ISC(网络系统联盟)组织下。
> `Internet Systems Consortium`(ISC）是一个非营利组织，致力于开发和支持**网络基础设施技术**，例如**DNS软件BIND、DHCP服务器**软件等。ISC在互联网技术的发展中扮演了重要的角色，提供了许多被广泛使用的开源软件。


# BIND概述
## bind相关包介绍
- 包名：bind
- 进程：named
- 协议：dns
- 使用端口：53（tcp，udp）

**常用命令**
```bash
#检查配置文件
named-checkconf

#启动bind9
systemctl start named

#检查bind9服务状态
systemctl status named
```

查看文件所属的包
```bash
# rpm -qf /sbin/named
bind-9.11.4-16.P2.el7_8.3.x86_64
```

**相关包**：
- bind : 提供的DNS Server主软件程序包
- bind-chroot：选装，将named进程的活动范围限定在chroot目录，保证安全性。让named运行于一个jail（沙箱）模式下。
- bind-devel：与开发相关的头文件和库文件（编译安装bind时所需）
- bind-libs：被bind和bind-utils包中的程序共同用到的文件。
- bind-utils ：bind客户端程序集，包含了如dig,host,nslookup等工具。
- bind-export：
- bind-license：

## bind的工作端口
### UDP 53
查询单条记录时，使用UDP 53

### TCP 53
如果DNS查询的响应传送的数据太大，其会以TCP模式尝试发起DNS请求以及响应。

> 注：区域传输的时候 需要使用TCP 53 和 UDP 53。

### TCP  953
`rndc`的连接端口

## 二进制文件
**程序文件**：
```c
/usr/sbin/named              # dns 主程序
/usr/sbin/named-checkconf    # 检测/etc/named.conf文件语法错误
/usr/sbin/named-checkzone    # 检测zone和对应zone文件的语法错误
/usr/sbin/rndc               # 远程dns管理工具
/usr/sbin/rndc-confgen       # 生成rndc密钥

全部的程序文件如下所示：
（1）主程序
- named

（2）管理工具
- nsupdate
- rndc
- named-checkconf
- named-checkzone
- rndc-confgen

- dnssec-keygen
- dnssec-makekeyset
- dnssec-signkey
- dnssec-signzone

（3）诊断工具
- nslookup
- dig
- host
```

**主程序目录**：/var/named
```c
/var/named/named.ca          # 根解析库
/var/named/named.localhost   # 本地主机解析库
/var/named/slaves            # 从ns服务器文件夹
```

### 对配置文件进行语法检查

**named-checkconf工具：检查主配置文件**

```bash
named-checkconf [主配置文件]       //检查主配置文件的语法错误
named-checkconf -z [主配置文件]    //加载主配置文件中对应的区域数据库文件
```

**named-checkzone工具：检查区域数据库文件**

```bash
named-checkzone  <域名>  [区域数据库文件]
例：
named-checkzone  yuji.com  yuji.com.zone
```

**named-compilezone**：对区域数据文件进行编译，并输出编译后的结果。

```
named-compilezone  -o  OUTFILE  <域名>  [区域数据库文件]
```

比如：
```bash
[root@xuexi named]# named-compilezone  -o  -  longshuai.com  /var/named/db.longshuai.com   
zone longshuai.com/IN: loaded serial 1  
longshuai.com.            21600 IN SOA    dnsserver.longshuai.com. xyz.longshuai.com. 1 10800 3600 604800 3600  
longshuai.com.            21600 IN NS     dnsserver.longshuai.com.  
dnsserver.longshuai.com.  21600 IN A      172.16.10.15  
ftp.longshuai.com.        21600 IN A      172.16.10.17  
mydb.longshuai.com.       21600 IN A      172.16.10.18  
www.longshuai.com.        21600 IN A      172.16.10.16  
www1.longshuai.com.       21600 IN CNAME  www.longshuai.com.  
OK
```
## 配置文件
**配置文件**：
- 主配置文件：  **/etc/named.conf**
- 根域配置文件： **/var/named/named.ca**
- 区域配置文件： **/etc/named.rfc1912.zones**
- 保存DNS解析记录的数据文件（区域数据库文件）位于： **/var/named/目录下**

### 根域配置文件
**根域配置文件位于： /var/named/named.ca**
- 根域配置文件设定根域的域名数据库，包括根域中13台DNS服务器的信息。
- 利用该文件可以让DNS服务器找到根DNS服务器，并初始化DNS的缓冲区。当DNS服务器接到客户端主机的查询请求时，如果在Cache中找不到相应的数据，就会通过根服务器进行逐级查询 。
- 用户不需要进行修改该文件 。
### 主配置文件（全局配置文件）
**BIND服务的主配置文件位于： /etc/named.conf**。
- 设置DNS服务器的全局参数
- 包括监听地址和端口、区域数据文件存放的目录等
- 使用 options{......}; 的配置段

### 区域配置文件
**区域配置文件位于：/etc/named.rfc1912.zones。**
- 设置本服务器提供域名解析的特定DNS区域
- 包括域名、服务器角色、数据文件名等
- 使用 zone "区域名" IN{......}; 的配置段

### 区域数据库文件（zone文件）
BIND服务的区域数据库文件位于/var/named/ 目录下，具体文件名由管理员定义。一般格式为/var/named/域名.zone，例如：/var/named/yuji.com.zone。
#### 资源记录
主要包含以下三部分：
**1）全局TTL配置项及SOA记录**
- $TTL（Time To Live，生存时间）： 表示DNS记录在DNS服务器上的缓存时间，默认单位秒。
- @：表示当前域名。
- IN：表示使用 Internet 协议。
- SOA（Start Of Authority，授权信息开始）：表示解析方式。
- 分号 ";" 开始的部分表示注释信息

**2）正向解析记录**
- NS记录：域名服务器记录（Name Server）。
- MX记录：邮件交换记录（Mail Exchange）。
- A记录：地址记录（Address）。用来指定主机名（或域名）对应的IP地址记录。用于正向解析。
- CNANE：别名记录。 这种记录允许您将多个名字映射到同一台计算机。
```bash
 NS  master                                //当前区域的DNS服务器名称    
 master  IN    A     192.168.72.10        //记录DNS服务器master的IP地址
             MX 10   mail.yuji.com        //MX为邮件交换记录，数字越大优先级越低
             MX 20   mail2.yuji.com       //MX为邮件交换记录，数字越大优先级越低
 mail    IN    A     192.168.72.103       //记录正向解析mail.yuji.com对应的IP
 mail2   IN    A     192.168.72.104       //记录正向解析mail2.yuji.com对应的IP
 www     IN    A     192.168.72.101       //记录正向解析www.yuji.com对应的IP
 ftp     IN    A     192.168.72.102       //记录正向解析ftp.yuji.com对应的IP
 web     IN   CNAME   WWW                 //CNAME使用别名，web是www的别名
 *       IN    A     192.168.72.100      //泛域名解析，*表示任意主机名。泛域名有优先级，从上而下匹配。
```

**3）反向解析记录**
PTR： 指针记录 (Pointer Record) ，用来指定IP地址对应的域名。用于反向解析。
记录的如一列指定IP地址中的主机地址部分
```bash
NS        master          //当前区域的DNS服务器名称
 master  IN     A      192.168.72.10     //记录DNS服务器的IP地址
 200     IN    PTR     www.nan.com       //记录反向解析192.168.72.200对应的IP地址
 201     IN    PTR     ftp.nan.com       //记录反向解析192.168.72.201对应的IP地址
```

**注意**
- TTL可从全局继承
- 使用 "@" 符号可用于引用当前区域的域名
- 同一个名字可以通过多条记录定义多个不同的值；此时DNS服务器会以轮询方式响应。
- 同一个值也可能有多个不同的定义名字；通过多个不同的名字指向同一个值进行定义；此仅表示通过多个不同的名字可以找到同一个主机。

#### 区域数据库文件的特殊应用
- **基于域名解析的负载均衡**
同一域名对应到多个IP地址

- **泛域名解析**
找不到精确对应的A记录时，使用星号（*）进行匹配


## BIND服务控制
- systemctl [status|start|stop|restart] named.service
- rndc reload 重新加载DNS服务

## 权限相关
**bind权限相关**：安装完named会自动创建用户named系统用户

## named启动脚本
`/etc/init.d/named` 内容如下：
```bash
#!/bin/bash 
# named a network name service. 
# chkconfig: 345 35 75 
# description: a name server

if [ `id -u` -ne 0 ];then
   echo "ERROR:For bind to port 53,must run as root." 
   exit 1
fi


case "$1" in
start)
        if [ -x /usr/sbin/named ]; then
  /usr/sbin/named -c /etc/bind/named.conf -u named && echo . && echo 'BIND9.10 server started' 
        fi
      ;;
 
stop)
       kill `cat /etc/named/var/named.pid` && echo . && echo 'BIND9.10 server stopped' 
       ;;

restart)
       echo . 
       echo "Restart BIND9.10 server" 
       $0 stop
       sleep 10
       $0 start
       ;;

reload)
      /usr/sbin/rndc reload
      ;;

status)
     /usr/sbin/rndc status
     ;;

*)
     echo "$0 start | stop | restart |reload |status" 
     ;;
esac
```

操作：
```bash
chmod  755  /etc/init.d/named
chkconfig --add named
chkconfig named on

/etc/init.d/named start 
/etc/init.d/named status
```


日常管理
```bash
/etc/init.d/named start 启动服务  
rndc status  查看服务运行情况  
rndc reload  重新加载区域文件  
rndc stop    停止DNS服务
```

# bind服务器
## 分类
 bind服务器配置角色：
- **主服务器（Primary Authoritative Name Server）**：
负责至少解析一个域内（zone）的域名，维护所负责解析的域数据库（zone文件），**可对负责的域数据进行读写操作**
-  **辅服务器（Secondary Authoritative Name Server）**：
负责从主服务器或其他辅服务器中复制相关解析库，为主服务器缓解解析压力
- **缓存服务器（Caching Name Servers）**：
不负责域名解析，仅仅作为缓存，加速解析速度
- **转发服务器（Forwarding Name Servers）**：
发现非本机负责的请求后，不再向根发起请求，而是直接转发给指定的一台或多台服务器，自身并不保存查询

## 应用
不同角色配置起到不同的功能，在实际环境中，主服务器和辅助服务器 同时也是缓存服务器和转发服务器，可以把配置参数进行结合启用，那么就相当于一台服务器多种角色。

## DNS服务器返回结果
### 答案类别
- `有查询结果`（肯定答案）
- `不存在查询结果`（否定答案）
### 有查询结果分类
- `权威回答`
    - DNS服务器自己直接负责的域返回的答案

- `非权威回答`
    - DNS服务器未负责的域，由缓存或者查询到的记录返回的答案

## 从服务器
DNS从服务器也叫辅服DNS服务器，如果网络上某个节点只有一台DNS服务器的话，首先服务器的抗压能力是有限的，当压力达到一定的程度，服务器就可能会宕机罢工。
其次如果这台服务器出现了硬件故障那么服务器管理的区域的域名将无法访问。
为了解决这些问题，最好的办法就是使用多个DNS服务器同时工作，  并实现数据的同步，这样两台服务器就都可以实现域名解析操作。

### 从服务器要点
```
1、应该为一台独立的名称服务器
2、主DNS服务器的区域解析库文件中必须有一条NS记录指向从服务器
3、从DNS服务器只需要定义区域，而无须提供解析库文件；解析库文件自动生成文件放置于/var/named/slaves/目录中，并且有写权限。
4、主DNS和辅助DNS服务器设置区域复制权限。防止使用 dig -t axfr hunk.tech 抓取信息。
5、主DNS和辅助DNS服务器时间应该同步，可通过ntp进行；
6、bind程序的版本应该保持一致；否则，应该辅助DNS版本高，主DNS版本低
```

> 既然slave也是域数据解析的负责人(尽管它是从的)，因此在master上的区域数据文件中应该添加slave的ns记录，否则外部没法找到slave这个不完整的dns服务器来帮忙解析，且master也无法找到slave。


```
定义从区域的方法：
zone "ZONE_NAME" IN {
	type slave;
	masters { 主DNS服务器IP; };
	file "slaves/ZONE_NAME.zone";
};
```

**很重要的一点：主从服务器时间不同步的话，则会导致各种意想不到的问题发生。因此，先配置NTP时间同步**

### 从服务器什么时候进行zone同步
```
A.把辅助DNS的区域文件删除了
    1.重启辅助DNS的named服务后可以恢复
    2.主DNS服务器版本更新并重新加载时，辅助DNS会同步生成文件
    3.主服务器上执行 `rndc retransfer 区域名称`
    如：rndc retransfer 4.168.192.in-addr.arpa
       rndc retransfer hunk.tech
B.版本更新后，只要reload，会立即发生同步 
```

### 从服务器上的zone文件查看

辅助DNS辅助器生成的区域文件，Centos 6 可以使用`cat`等文本工具查看；
Centos 7 已经使用 raw 格式存放，需要使用这个命令配合参数查看:
```bash
# 输出到标准输出
named-compilezone -f raw -o -  zone_name  zone文件

# 输出到指定文件(/tmp/zone.out)
named-compilezone -f raw -o -  zone_name  zone文件
```

范例如下所示：
```bash
# named-compilezone  -o  /tmp/aa.out  longshuai.com  /var/named/db.longshuai.com   
zone longshuai.com/IN: loaded serial 1
dump zone to /tmp/aa.out...done
OK

# cat /tmp/aa.out
longshuai.com.            21600 IN SOA    dnsserver.longshuai.com. xyz.longshuai.com. 1 10800 3600 604800 3600  
longshuai.com.            21600 IN NS     dnsserver.longshuai.com.  
dnsserver.longshuai.com.  21600 IN A      172.16.10.15  
ftp.longshuai.com.        21600 IN A      172.16.10.17  
mydb.longshuai.com.       21600 IN A      172.16.10.18  
www.longshuai.com.        21600 IN A      172.16.10.16  
www1.longshuai.com.       21600 IN CNAME  www.longshuai.com.  
```

### 从服务器也可以是其他从服务器的主
还可以配置slave从另一个slave上复制区域数据，这种情况下，master是slave1的主，slave1是slave2的主。如下图：
![](attachments/Pasted%20image%2020240117153926.png)

配置方式一样很简单，只需将slave2主机的named.conf中的zone区域配置成如下格式：
```bash
zone "domain" IN {  
    type slave;  
    masters { slave1_ip };  
};
```
当然，master的区域数据文件中也应该制定slave2的ns记录和A记录。

## 转发(forward)服务器
### 介绍
转发功能可以用来在一些服务器上产生一个大的缓存，从而减少到外部服务器链路上的流量。它可以使用在和internet没有直接连接的内部网络进行域名服务器上，用来提供对外部域名的查询。

其中，转发设备称之为转发者，被转发设备称之为转发器（转发选项所指定的机器称为转发器）。
![](attachments/Pasted%20image%2020240117144840.png)
> **转发者转发给转发器的查询是递归查询，转发器必须要亲自回复转发者**，也就是说转发者其实就相当于一个客户端。

### 配置
内网DNS服务器转发内网主机对于外网的域名查询。
只有当服务器对查询记录是非授权的，并且缓存中没有相关记录时，才会进行转发。
> 注意：被转发的服务器需要能够为请求者做递归，否则转发请求不予进行。
```bash
recursion yes
```

关闭`dnssec`功能：
```
dnssec-enable no;
dnssec-validation no;
```


### 作用范围
#### 全局转发
对非本机所负责解析区域的请求，全转发给指定的服务器。
```bash
语法格式：
    Options { 
        forward first 或 only; 
        forwarders { ip;可以有多个，用;号隔开};
    };

范例：
    forward first;
    forwarders { 192.168.4.204; };
```

- first：默认值。服务器首先将请求转发列表中的设定的DNS主机 ，如果转发列表中的DNS主机不应答，该主机将从根DNS开始找应答。
- only：**仅转发**；只会请求转发列表中的设定的DNS主机 ，如果转发列表中的DNS主机不应答，也不会自己去查询。

#### 转发区
仅转发对特定的区域的请求，比全局转发优先级高。

```bash
语法格式：
    zone "ZONE_NAME" IN {
        type forward;
        forward first 或 only;
        forwarders { ip;可以有多个，用;号隔开};
    };
```

范例：
```bash
	zone "longshuai.com" IN {  
	    type forward;  
	    forwarders { 172.16.10.15; };  
	};
```
只有查询”longshuai.com”结尾的记录都将转发到`172.16.10.15`上。
但要注意，是`longshuai.com`结尾，而不是限制其域为”longshuai.com”，所以如果发出`png.img.longshuai.com`的查询请求，也会被转发，但实际上这是`longshuai.com`的子域`img.longshuai.com`中的主机。

#### 禁转发区
![](attachments/Pasted%20image%2020240117145714.png)

```bash
options {  
    directory "/var/named";  
    forwarders { 172.16.10.15; };  
};  
  
zone "longshuai.com" IN {  
    type master;  
    forwarders {};  
};  
  
zone "." IN {  
    type hint;  
    file "named.ca";  
};  
  
include /etc/named.rfc1912.zones;
```

### 分类
#### 转发优先
默认转发：如果转发器无法查询到结果，也就是说转发者获取不到来自转发器的答案，则转发者自己也会去查询。
”forward first”，先转发，转发失败时自行查询，这是默认值。
#### 仅转发
仅转发：它只转发，即使获取不到转发器的回复，也不会自己去查询。
要配置成”仅转发”dns服务器，只需加上”forward only”指令即可。

几乎不会去配置仅转发dns服务器。实际上，操作系统中设置dns指向的地方，就是在设置仅转发功能，例如`/etc/resolv.conf`文件中所设置的`nameserver`，这就是仅转发功能。


# bind配置文件
## 配置总体概述
`/var/named/` : 资源记录文档的存放位置
`/var/named/named.ca` ：13个根服务器存放文件
`/var/named/named.empty`：
`/var/named/named.localhost`：
`/var/named/named.loopback`：

`/etc/logrotate.d/named` ：日志切割配置文件

`/etc/named.conf`： 主配置文件
`/etc/named.rfc1912.zones`： 区域配置文件（即zone文件，用include指令包含在主配置文件）
`/etc/named.root.key`： 根区域的key文件以实现事务签名；
`/etc/rndc.key`： `rndc`加密密钥

`/etc/rndc.conf`： `rndc`（远程名称服务器控制器）配置文件

### 配置格式
下面是 BIND 9.2.4 支持的 named.conf 选项的摘要。
```bash
options  {
        blackhole { <address_match_element>; ... };
        coresize <size>;
        datasize <size>;
        deallocate-on-exit <boolean>; // obsolete
        directory <quoted_string>;
        dump-file <quoted_string>;
        fake-iquery <boolean>; // obsolete
        files <size>;
        has-old-clients <boolean>; // obsolete
        heartbeat-interval <integer>;
        host-statistics <boolean>; // not implemented
        host-statistics-max <integer>; // not implemented
        interface-interval <integer>;
        listen-on [ port <integer> ] { <address_match_element>; ... };
        listen-on-v6 [ port <integer> ] { <address_match_element>; ... };
        match-mapped-addresses <boolean>;
        memstatistics-file <quoted_string>; // not implemented
        multiple-cnames <boolean>; // obsolete
        named-xfer <quoted_string>; // obsolete
        pid-file <quoted_string>;
        port <integer>;
        random-device <quoted_string>;
        recursive-clients <integer>;
        rrset-order { [ class <string> ] [ type <string> ] [ name
            <quoted_string> ] <string> <string>; ... }; // not implemented
        serial-queries <integer>; // obsolete
        serial-query-rate <integer>;
        stacksize <size>;
        statistics-file <quoted_string>;
        statistics-interval <integer>; // not yet implemented
        tcp-clients <integer>;
        tkey-dhkey <quoted_string> <integer>;
        tkey-gssapi-credential <quoted_string>;
        tkey-domain <quoted_string>;
        transfers-per-ns <integer>;
        transfers-in <integer>;
        transfers-out <integer>;
        treat-cr-as-space <boolean>; // obsolete
        use-id-pool <boolean>; // obsolete
        use-ixfr <boolean>;
        version <quoted_string>;
        allow-recursion { <address_match_element>; ... };
        allow-v6-synthesis { <address_match_element>; ... };
        sortlist { <address_match_element>; ... };
        topology { <address_match_element>; ... }; // not implemented
        auth-nxdomain <boolean>; // default changed
        minimal-responses <boolean>;
        recursion <boolean>;
        provide-ixfr <boolean>;
        request-ixfr <boolean>;
        fetch-glue <boolean>; // obsolete
        rfc2308-type1 <boolean>; // not yet implemented
        additional-from-auth <boolean>;
        additional-from-cache <boolean>;
        query-source <querysource4>;
        query-source-v6 <querysource6>;
        cleaning-interval <integer>;
        min-roots <integer>; // not implemented
        lame-ttl <integer>;
        max-ncache-ttl <integer>;
        max-cache-ttl <integer>;
        transfer-format ( many-answers | one-answer );
        max-cache-size <size_no_default>;
        check-names <string> <string>; // not implemented
        cache-file <quoted_string>;
        allow-query { <address_match_element>; ... };
        allow-transfer { <address_match_element>; ... };
        allow-update-forwarding { <address_match_element>; ... };
        allow-notify { <address_match_element>; ... };
        notify <notifytype>;
        notify-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ];
        notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ];
        also-notify [ port <integer> ] { ( <ipv4_address> | <ipv6_address>
            ) [ port <integer> ]; ... };
        dialup <dialuptype>;
        forward ( first | only );
        forwarders [ port <integer> ] { ( <ipv4_address> | <ipv6_address> )
            [ port <integer> ]; ... };
        maintain-ixfr-base <boolean>; // obsolete
        max-ixfr-log-size <size>; // obsolete
        transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ];
        transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ];
        max-transfer-time-in <integer>;
        max-transfer-time-out <integer>;
        max-transfer-idle-in <integer>;
        max-transfer-idle-out <integer>;
        max-retry-time <integer>;
        min-retry-time <integer>;
        max-refresh-time <integer>;
        min-refresh-time <integer>;
        sig-validity-interval <integer>;
        zone-statistics <boolean>;
};

controls {
        inet ( <ipv4_address> | <ipv6_address> | * ) [ port ( <integer> | *
            ) ] allow { <address_match_element>; ... } [ keys { <string>; ... } ];
        unix <unsupported>; // not implemented
};

acl <string> { <address_match_element>; ... };

logging {
        channel <string> {
                file <logfile>;
                syslog <optional_facility>;
                null;
                stderr;
                severity <logseverity>;
                print-time <boolean>;
                print-severity <boolean>;
                print-category <boolean>;
        };
        category <string> { <string>; ... };
};

view <string> <optional_class> {
        match-clients { <address_match_element>; ... };
        match-destinations { <address_match_element>; ... };
        match-recursive-only <boolean>;
        key <string> {
                algorithm <string>;
                secret <string>;
        };
        zone <string> <optional_class> {
                type ( master | slave | stub | hint | forward );
                allow-update { <address_match_element>; ... };
                file <quoted_string>;
                ixfr-base <quoted_string>; // obsolete
                ixfr-tmp-file <quoted_string>; // obsolete
                masters [ port <integer> ] { ( <ipv4_address> |
                    <ipv6_address> ) [ port <integer> ] [ key <string> ]; ... };
                pubkey <integer> <integer> <integer> <quoted_string>; //
                    obsolete
                update-policy { ( grant | deny ) <string> ( name |
                    subdomain | wildcard | self ) <string> <rrtypelist>; ... };
                database <string>;
                check-names <string>; // not implemented
                allow-query { <address_match_element>; ... };
                allow-transfer { <address_match_element>; ... };
                allow-update-forwarding { <address_match_element>; ... };
                allow-notify { <address_match_element>; ... };
                notify <notifytype>;
                notify-source ( <ipv4_address> | * ) [ port ( <integer> | *
                    ) ];
                notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer>
                    | * ) ];
                also-notify [ port <integer> ] { ( <ipv4_address> |
                    <ipv6_address> ) [ port <integer> ]; ... };
                dialup <dialuptype>;
                forward ( first | only );
                forwarders [ port <integer> ] { ( <ipv4_address> |
                    <ipv6_address> ) [ port <integer> ]; ... };
                maintain-ixfr-base <boolean>; // obsolete
                max-ixfr-log-size <size>; // obsolete
                transfer-source ( <ipv4_address> | * ) [ port ( <integer> |
                    * ) ];
                transfer-source-v6 ( <ipv6_address> | * ) [ port (
                    <integer> | * ) ];
                max-transfer-time-in <integer>;
                max-transfer-time-out <integer>;
                max-transfer-idle-in <integer>;
                max-transfer-idle-out <integer>;
                max-retry-time <integer>;
                min-retry-time <integer>;
                max-refresh-time <integer>;
                min-refresh-time <integer>;
                sig-validity-interval <integer>;
                zone-statistics <boolean>;
        };
        server {
                bogus <boolean>;
                provide-ixfr <boolean>;
                request-ixfr <boolean>;
                support-ixfr <boolean>; // obsolete
                transfers <integer>;
                transfer-format ( many-answers | one-answer );
                keys <server_key>;
                edns <boolean>;
        };
        trusted-keys { <string> <integer> <integer> <integer>
            <quoted_string>; ... };
        allow-recursion { <address_match_element>; ... };
        allow-v6-synthesis { <address_match_element>; ... };
        sortlist { <address_match_element>; ... };
        topology { <address_match_element>; ... }; // not implemented
        auth-nxdomain <boolean>; // default changed
        minimal-responses <boolean>;
        recursion <boolean>;
        provide-ixfr <boolean>;
        request-ixfr <boolean>;
        fetch-glue <boolean>; // obsolete
        rfc2308-type1 <boolean>; // not yet implemented
        additional-from-auth <boolean>;
        additional-from-cache <boolean>;
        query-source <querysource4>;
        query-source-v6 <querysource6>;
        cleaning-interval <integer>;
        min-roots <integer>; // not implemented
        lame-ttl <integer>;
        max-ncache-ttl <integer>;
        max-cache-ttl <integer>;
        transfer-format ( many-answers | one-answer );
        max-cache-size <size_no_default>;
        check-names <string> <string>; // not implemented
        cache-file <quoted_string>;
        allow-query { <address_match_element>; ... };
        allow-transfer { <address_match_element>; ... };
        allow-update-forwarding { <address_match_element>; ... };
        allow-notify { <address_match_element>; ... };
        notify <notifytype>;
        notify-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ];
        notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ];
        also-notify [ port <integer> ] { ( <ipv4_address> | <ipv6_address>
            ) [ port <integer> ]; ... };
        dialup <dialuptype>;
        forward ( first | only );
        forwarders [ port <integer> ] { ( <ipv4_address> | <ipv6_address> )
            [ port <integer> ]; ... };
        maintain-ixfr-base <boolean>; // obsolete
        max-ixfr-log-size <size>; // obsolete
        transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ];
        transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ];
        max-transfer-time-in <integer>;
        max-transfer-time-out <integer>;
        max-transfer-idle-in <integer>;
        max-transfer-idle-out <integer>;
        max-retry-time <integer>;
        min-retry-time <integer>;
        max-refresh-time <integer>;
        min-refresh-time <integer>;
        sig-validity-interval <integer>;
        zone-statistics <boolean>;
};

lwres {
        listen-on [ port <integer> ] { ( <ipv4_address> | <ipv6_address> )
            [ port <integer> ]; ... };
        view <string> <optional_class>;
        search { <string>; ... };
        ndots <integer>;
};

key <string> {
        algorithm <string>;
        secret <string>;
};

zone <string> <optional_class> {
        type ( master | slave | stub | hint | forward );
        allow-update { <address_match_element>; ... };
        file <quoted_string>;
        ixfr-base <quoted_string>; // obsolete
        ixfr-tmp-file <quoted_string>; // obsolete
        masters [ port <integer> ] { ( <ipv4_address> | <ipv6_address> ) [
            port <integer> ] [ key <string> ]; ... };
        pubkey <integer> <integer> <integer> <quoted_string>; // obsolete
        update-policy { ( grant | deny ) <string> ( name | subdomain |
            wildcard | self ) <string> <rrtypelist>; ... };
        database <string>;
        check-names <string>; // not implemented
        allow-query { <address_match_element>; ... };
        allow-transfer { <address_match_element>; ... };
        allow-update-forwarding { <address_match_element>; ... };
        allow-notify { <address_match_element>; ... };
        notify <notifytype>;
        notify-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ];
        notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ];
        also-notify [ port <integer> ] { ( <ipv4_address> | <ipv6_address>
            ) [ port <integer> ]; ... };
        dialup <dialuptype>;
        forward ( first | only );
        forwarders [ port <integer> ] { ( <ipv4_address> | <ipv6_address> )
            [ port <integer> ]; ... };
        maintain-ixfr-base <boolean>; // obsolete
        max-ixfr-log-size <size>; // obsolete
        transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ];
        transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ];
        max-transfer-time-in <integer>;
        max-transfer-time-out <integer>;
        max-transfer-idle-in <integer>;
        max-transfer-idle-out <integer>;
        max-retry-time <integer>;
        min-retry-time <integer>;
        max-refresh-time <integer>;
        min-refresh-time <integer>;
        sig-validity-interval <integer>;
        zone-statistics <boolean>;
};

server {
        bogus <boolean>;
        provide-ixfr <boolean>;
        request-ixfr <boolean>;
        support-ixfr <boolean>; // obsolete
        transfers <integer>;
        transfer-format ( many-answers | one-answer );
        keys <server_key>;
        edns <boolean>;
};

trusted-keys { <string> <integer> <integer> <integer> <quoted_string>; ... };
```
## 主配置文件`named.conf`
named.conf是named默认加载的配置文件，该配置文件中使用`#`或`/**/`或`//`作为注释符号，每个非注释语句都必须使用分号`;`结束。

BIND 9 配置由**块、语句和注释**组成。
- 注释
- 语句
named.conf，每个语句都要使用分号结尾
- 块
此中的块，就是模块。一个模块中包含多个语句。


|named.conf配置文件所有的配置语句|功能说明|
|---|---|
|acl|定义一个主机匹配列表，用户访问控制权限|
|controls|定义rndc命令使用的控制通道，若省略，则只允许经过rndc.key认证的127.0.0.1的rndc控制|
|include|include filename; 引入外部局部配置文件|
|dnssec-policy|dnssec 安全策略，一般默认不启用dnssec功能|
|key|使用 TSIG 指定用于身份验证和授权的密钥信息，一般rndc的授权密钥信息|
|logging|日志记录|
|options|定义全局选项|
|statistics-channels|声明通信的通道，用于访问bind的统计信息数据|
|tls|指定 TLS 连接的配置信息|
|http|指定 HTTP 连接的配置信息|
|trust-anchors|定义 DNSSEC 密钥信息|
|zone|定义一个区域声明|
|view|定义域名空间的一个视图|
|server|可以出现在配置文件的顶级，可以在一个view中，定义对特定的服务器设置参数|
|primaries|定义一个命名的主服务器列表，一般包含在存根区或者辅区的primaries或者also-notify列表中。|
|parental-agents|定义要由主要和次要区域使用的委派代理列表|

重点说明下：acl、view、logging、options、zone等模块

### 范例
```json
# named.conf主配置文件参数详解 

//全局配置段options {
        listen-on port 53 { 192.168.31.113; }; #设置通信的网段，这里建议使用本机IP，并非127.0.0.1
        listen-on-v6 port 53 { ::1; }; #监听bind端口
        directory       "/var/named";  #指定区域文件存放路径
        dump-file       "/var/named/data/cache_dump.db"; #设置当执行rndc dumpdb命令后导出文件存放路径
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt"; #服务器输出的内存使用统计文件名
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };  #允许查询来源(这里建议修改为IP地址，localhost代表只允许本机查询) any代表所有网段
        allow-transfer { none; } #允许查询的网段
        recursion yes; #是否开启递归查询
        dnssec-enable yes; #是否支持DNSEEC开关
        dnssec-validation yes; #是否开启dnsec确认开关
        bindkeys-file "/etc/named.root.key";
        managed-keys-directory "/var/named/dynamic";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};  

//日志配置段
logging {
        channel default_debug {         
                file "data/named.run";
                severity dynamic;
        };
  #本段参数解释，将日志写入工作目录下的named.run文件。注意：如果服务器用-f参数启动，则named.run会被stderr所代替，severity 按照服务器当前Debug级别记录日志
  #bind日志可以写到很多地方，具体写入方式可以参考https://blog.csdn.net/zhu_tianwei/article/details/45103455
}; 

//区域配置段
zone "." IN {             #.代表根域
        type hint;       #代表根服务器，除hint还有master 代表主域名服务器，slave代表辅助域名服务器，forward 代表转发服务器
        file "named.ca";  #域信息源数据库信息文件名
}; 

include "/etc/named.rfc1912.zones";   #区域管理文件(包含资源记录、宏定义和注释)
                                      #通常定义在/var/named目录下且以.zone结尾
include "/etc/named.root.key";
```
### 注释
C 语言的注释风格，分别是`/* xxx */` 以及 `//`。 如下所示：
```bash
// : 表示注释一行
/* */ : 表示注释多行的内容
```
![](attachments/Pasted%20image%2020231221152917.png)
![](attachments/Pasted%20image%2020231221152953.png)

>说明：zone区域配置文件可以使用分号 `;` 字符注释，但是在主配置文件中`;`不是注释，而是配置文件的行末尾结束符。

## 主配置文件中的options 块
参考： [Bind中的 options 语句定义和用法](http://www.hangdaowangluo.com/archives/1665)

`options` 的参数设置会影响整个 BIND9 DNS环境的配置，具体各部分常用到的配置参数如下：
- `listen-on`： 用于配置监听的端口以及IPv4地址，默认的监听端口为：53；
- `listen-on-v6`：用于监听 IPv6 地址以及端口；
- `tls-port`：接收和发送 DNS-over-TLS 协议流量的 TCP 端口号。默认值为 853；
- `http-port`：通过 HTTP 接收和发送未加密的 DNS 流量的 TCP 端口号；
- `https-port`：接收和发送 DNS-over-HTTPS 协议流量的 TCP 端口号。默认值为 443；
- `directory`: 用于指定读取DNS数据文件的文件夹，默认的文件夹的路径为：`/var/cache/bind`；
- `dump-file`：用来设置域名缓存数据库文件的位置，可以自己定义。默认的存储文件为：`named_dump.db`；
- `statistics-file`：用来设置状态统计文件的位置，可以自己定义。；
- `memstatistics-file` ：用来设置服务器输出的内存使用统计信息。默认保存在 `/var/named/data` 目录下，文件名为 `named.memstats`；
- `allow-query`：用来设置允许DNS查询的客户端地址，默认值为`localhost`, 可以设置为某个网段、任意地址、具体的某台主机三种情况。例如，要修改为任意地址，就在括号内的加入 `any`，也可以引用之前创建的 `acl` 内的所有地址；

- `recursion`：用于设置递归查询，一般客户机和服务器之间属于递归查询，即当客户机向DNS服务器发出查询请求后，若DNS服务器本身不能解析，则会向另外的DNS服务器发出查询请求，得到结果后转交给客户机。此选项有`yes`和`no`两个值。这个选项用于设置 Failover 非常有用；
- `dnssec-enable`： 选项用来设置是否启用`DNSSEC`支持，`DNSSEC`可以用来验证`DNS数据`的有效性，该选项有`yes`和`no`两个值，默认值为`yes`。
- `dnssec-validation`：选项用来设置是否启用DNSSEC确认，默认值为`yes`，可以选择 `auto`。
- `bindkeys-file` ： 用来设置内置信任的密钥文件，其默认值为 `/etc/named/iscdlv.key`；
- `managed-keys-directory`： 选项用于指定目录中的文件存储位置，跟踪管理 `DNSSEC` 密钥
- `forwarders`：DNS转发器。用于设定该DNS解析服务器无法进行当前域名解析的情况下，进行转发解析的DNS地址，其中 `8.8.8.8` 和 `8.8.4.4` 是谷歌的免费DNS服务器的网络地址；`233.5.5.5` 和 `233.6.6.6` 是阿里云的免费DNS地址。当设置了 `forwarder` 的转发器之后，所有的非本域的和在缓存中无法查找到的域名查询都转发都设置的DNS转发器，由DNS转发器 完成转发操作。因此这台转发器的缓存中就记录了丰富的域名信息。因此如果遇到非本域的查询，转发器的缓存就可以做到查询，从而减少了向外部的查询流量。
- `rrset-order`：  
    在 BIND 9 提供的负载均衡策略建立在一个名称（域名 - Name）使用多个资源记录 ( Records ) 的情况下，其实现的轮询机制并不是传统的负载均衡服务器实现的轮询机制 - 即追踪和记录每一次应答的资源顺序；  
    BIND 9 实现了一个类似 List 的数据结构，将所有的资源记录填入到 一个顺序表中，这个填入的次序随机，或者根据设定的参数随机；  
    格式：[class _class_name_] [type _type_name_] [name “_domain_name_”] order _ordering_  
    如果参数没有被赋值，那么默认的赋值为： class: ANY type: ANY Name: *  
    参数：
    - `fixed` ： 根据 zone 文件定义资源记录的顺序按照顺序逐个进行解析；
    - `random`： 根据 zone 文件资源记录随机返回解析记录；
    - `cyclic`： 创建一个循环，循环输出资源记录；
    - `none`： 完全随机的资源返回形式；

- `sortlist`：基于查询的client的范围，返回响应的DNS记录的顺序。可在全局`option`以及具体的 `view`中进行配置。

### recursion 递归
一般客户机和服务器之间属递归查询，当客户机向本地DNS服务器发出请求后,若DNS服务器本身不能解析,则会向另外的DNS服务器发出查询请求，得到结果后转交给客户机。
主机向本地域名服务器的查询一般都是采用递归查询。

#### 配置参数

- **recursion**：默认值是yes。

![](attachments/Pasted%20image%2020240416182654.png)

如果是yes，并且一个client的 DNS询问要求递归，那么服务器将会做所有能够回答查询请求的工作。如果recursion是off的，并且服务器不知道答案，它将会返回一个推荐（referral）响应。
注意把recursion设为no，不会阻止用户从服务器的缓存中得到数据，它仅仅阻止新数据作为查询的结果被缓存。服务器的内部操作还是可以影响本地的缓存内容，如NOTIFY地址查询。

- **allow-recursion**

![](attachments/Pasted%20image%2020240416182859.png)

设定哪台`Client`主机可以进行递归查询。如果没有设定，缺省是允许所有主机进行递归查询。
注意禁止一台主机的递归查询，并不能阻止这台主机查询已经存在于服务器缓存中的数据。
权威服务器`不应允许`递归查询，防止服务器被用于`DNS放大分布式拒绝服务攻击`，并更好地保护其免受缓存中毒攻击。

- **recursive-clients**：默认值1000.

![](attachments/Pasted%20image%2020240416183606.png)


服务器同时为用户执行的递归查询的最大数量。默认值1000，因为每个递归用户使用许多内存，一般为20KB，主机上的recursive-clients选项值必须根据实际内存大小调整。

recursive-clients 为待处理的递归客户端定义“硬配额”限制；当挂起的客户端数量超过此数量时，将不接受新的传入请求，并且对于每个传入请求，都会删除先前的挂起请求。

还设置了“软配额”。当超出这一较低配额时，传入请求将被接受，但对于每个请求，都会删除待处理的请求。如果 recursive-clients 大于 1000，则软配额设置为 recursive-clients 减去 100；否则它被设置为 90% 的递归客户端。

- **resolver-query-timeout**

![](attachments/Pasted%20image%2020240416183113.png)

指定 dns 解析器进行递归查询的超时时间。

- **max-clients-per-query** 和 **clients-per-query**
 
![](attachments/Pasted%20image%2020240416191518.png)

注：max-clients-per-query 和 clients-per-query 限制的是对于特定的外网域名(同样的name，同样的type的查询) 的递归查询的并发请求个数，而 recursive-clients 针对的 任意外网域名的的并发请求个数。


- **fetches-per-zone**

![](attachments/Pasted%20image%2020240416193137.png)

max-clients-per-query 指示的是对于外网的某个特定的递归查询（多个并发查询有相同的请求name和type）的并发请求限制。
如果对外网的多个并发请求不同，但是属于一个zone或者其下的子zone，那么 max-clients-per-query 将无法进行限制。

即 多个client 对于特定域名以及类型的多个并发查询，被归拢为一个 fetch，只迭代发送一个递归请求给外网dns服务器。

隶属于同一个zone及其下的子zone的不同类型以及域名的查询，属于不同的fetch，fetch的个数受限于 fetches-per-zone。

超过阈值之后的动作：
1> 直接丢弃。（默认动作）
2> 给client 回复 SERVFAIL。

- **fetches-per-server**

![](attachments/Pasted%20image%2020240417105557.png)

单个外网服务器(即：reslove 服务器的转发服务器) 可以接收的并发的 fetch 的最大数量。

 对于特定域名以及类型的多个并发查询，被归拢为一个 fetch，只迭代发送一个递归请求给外网dns服务器。如果 reslover 服务器 并发的转发多个外网请求给 外网的权威dns服务器，那么限制转发给外网服务器的 fetch的数量。

注：fetches-per-server 是动态调整的， 需要和 fetch-quota-params 配合使用。
如下所示：
```none
fetches-per-server 200 fail;
fetch-quota-params 100 0.1 0.3 0.7;
```

参考：https://bind9.readthedocs.io/en/v9.18.25/reference.html

#### 配置格式
```bash
recursion yes | no;
recursive-clients number;
allow-recursion { address_match_list | any | none;};
```
范例：
```bash
recursion      yes;
allow-recursion   { any; };
或
allow-recursion { 10.0/16; };
recursive-clients 25;
```

**配置建议**
- 如果你要建立一个 授权域名服务器服务器, 那么不要开启recursion（递归）功能。
- 如果你要建立一个 递归 DNS 服务器, 那么需要开启recursion 功能。
- 如果你的递归DNS服务器有公网IP地址, 你必须开启访问控制功能，只有那些合法用户才可以发询问. 如果不这么做的话，那么你的服服务就会受到DNS 放大攻击。实现BCP38将有效抵御这类攻击。

使用非递归查询服务器需要注意:
1. 保证该非递归服务器不出现在客户机的`/etc/resolv.conf` 的 `server`中；
2. 保证该非递归服务器不被其他 name server 当成转发器 （forwarder）；

注：内网中的`Client`的 `/etc/resolv.conf` 的 `server` 为内网DNS服务器的地址；内网 DNS 服务器可以解析内网的域名、主机名等，同时也可以转发外网的域名查询。

此时内网的DNS服务器可以开启递归查询，这样对于外网的域名查询，得到响应后，也可以进行缓存。
否则，不开启递归查询，那么收到client对于外网的域名查询，本地不存在记录，则给client返回 一个推荐（referral）响应， client 后续进行迭代查询。

正常情况下，内网DNS服务器配置的转发器为知名的DNS服务器，比如`8.8.8.8`，对于`8.8.8.8`外网DNS服务器上，正常情况下也是开启了递归查询。因为外网DNS服务器也需要递归查询，才可以缓存记录，以及减轻访问其的Client的压力。


### allow-query
**作用域**：
全局option的配置，以及 zone中的配置。一般配置在**主和从dns服务器**。

如果是zone中的配置，`allow-query` 中定义的任何主机都可以查询此 zone。

### also-notify
**作用域**：
全局option的配置，以及 zone中的配置。一般配置在**主dns服务器**。

```bash
also-notify { 192.168.10.223; }; 
        # 主动通知从域名服务器（辅助DNS）进行更新，在主域名服务器进行更新后，而不需要在等规定的时间后才通知从域名服务器进行更新。
```

### allow-transfer
**作用域**：
全局option的配置，以及 zone中的配置。一般配置在**主dns服务器**。

控制`区域转移`；区域转移允许指定的服务器获取您 `区域中所有数据的转储`；
区域转移应该受到限制，以使潜在的攻击者更难执行一个DNS查询来快速获取您区域中的所有资源记录。
`主服务器必须配置允许转移`，以允许您的`从服务器`执行区域转移。您应该禁止其他主机执行区域传输。您可能允许`localhost`执行区域传输以帮助进行故障排除。

```bash
allow-transfer { 192.168.10.223; }; 
        # 允许本区域传输至特定的从DNS服务器。

```


## 主配置文件中的区域zone配置
除了option，还必需有区域的配置。zone关键字后面接的是域和类，域是自定义的域名，IN是internet的简称，是bind 9中的默认类，所以可以省略。

### Zone引导配置
正向解析域名的引导文件，通过以下的格式进行定义：
```json
zone "<YOUR DNS Domain Name >" {
    <Configurations>
}
```
反向解析域名的引导文件，通过以下的格式进行定义：
```json
zone "<YOUR IP ADDRESS>-addr.arpa" {
    <Configurations>
}
```
#### 字段说明

`named.rfc1912.zones`主配置文件的区域配置
```c
zone "ZONE_NAME" IN {
    type {master|slave|hint|forward};
    file "ZONE_NAME.zone";
};
```

```json
zone <string> [ <class> ] {
	in-view <string>;
};
zone <string> [ <class> ] {
	type delegation-only;
};
zone <string> [ <class> ] {
	type forward;
	delegation-only <boolean>;
	forward ( first | only );
	forwarders [ port <integer> ] [ dscp <integer> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ dscp <integer> ]; ... };
};
zone <string> [ <class> ] {
	type hint;
	check-names ( fail | warn | ignore );
	delegation-only <boolean>;
	file <quoted_string>;
};
zone <string> [ <class> ] {
	type mirror;
	allow-notify { <address_match_element>; ... };
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	allow-transfer [ port <integer> ] [ transport <string> ] { <address_match_element>; ... };
	allow-update-forwarding { <address_match_element>; ... };
	also-notify [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	alt-transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	alt-transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	check-names ( fail | warn | ignore );
	database <string>;
	file <quoted_string>;
	ixfr-from-differences <boolean>;
	journal <quoted_string>;
	masterfile-format ( raw | text );
	masterfile-style ( full | relative );
	max-ixfr-ratio ( unlimited | <percentage> );
	max-journal-size ( default | unlimited | <sizeval> );
	max-records <integer>;
	max-refresh-time <integer>;
	max-retry-time <integer>;
	max-transfer-idle-in <integer>;
	max-transfer-idle-out <integer>;
	max-transfer-time-in <integer>;
	max-transfer-time-out <integer>;
	min-refresh-time <integer>;
	min-retry-time <integer>;
	multi-master <boolean>;
	notify ( explicit | master-only | primary-only | <boolean> );
	notify-delay <integer>;
	notify-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	primaries [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	request-expire <boolean>;
	request-ixfr <boolean>;
	transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	try-tcp-refresh <boolean>;
	use-alt-transfer-source <boolean>;
	zero-no-soa-ttl <boolean>;
	zone-statistics ( full | terse | none | <boolean> );
};
zone <string> [ <class> ] {
	type primary;
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	allow-transfer [ port <integer> ] [ transport <string> ] { <address_match_element>; ... };
	allow-update { <address_match_element>; ... };
	also-notify [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	alt-transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	alt-transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	auto-dnssec ( allow | maintain | off );
	check-dup-records ( fail | warn | ignore );
	check-integrity <boolean>;
	check-mx ( fail | warn | ignore );
	check-mx-cname ( fail | warn | ignore );
	check-names ( fail | warn | ignore );
	check-sibling <boolean>;
	check-spf ( warn | ignore );
	check-srv-cname ( fail | warn | ignore );
	check-wildcard <boolean>;
	database <string>;
	dialup ( notify | notify-passive | passive | refresh | <boolean> );
	dlz <string>;
	dnskey-sig-validity <integer>;
	dnssec-dnskey-kskonly <boolean>;
	dnssec-loadkeys-interval <integer>;
	dnssec-policy <string>;
	dnssec-secure-to-insecure <boolean>;
	dnssec-update-mode ( maintain | no-resign );
	file <quoted_string>;
	forward ( first | only );
	forwarders [ port <integer> ] [ dscp <integer> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ dscp <integer> ]; ... };
	inline-signing <boolean>;
	ixfr-from-differences <boolean>;
	journal <quoted_string>;
	key-directory <quoted_string>;
	masterfile-format ( raw | text );
	masterfile-style ( full | relative );
	max-ixfr-ratio ( unlimited | <percentage> );
	max-journal-size ( default | unlimited | <sizeval> );
	max-records <integer>;
	max-transfer-idle-out <integer>;
	max-transfer-time-out <integer>;
	max-zone-ttl ( unlimited | <duration> );
	notify ( explicit | master-only | primary-only | <boolean> );
	notify-delay <integer>;
	notify-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	notify-to-soa <boolean>;
	nsec3-test-zone <boolean>; // test only
	parental-agents [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	parental-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	parental-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	serial-update-method ( date | increment | unixtime );
	sig-signing-nodes <integer>;
	sig-signing-signatures <integer>;
	sig-signing-type <integer>;
	sig-validity-interval <integer> [ <integer> ];
	update-check-ksk <boolean>;
	update-policy ( local | { ( deny | grant ) <string> ( 6to4-self | external | krb5-self | krb5-selfsub | krb5-subdomain | krb5-subdomain-self-rhs | ms-self | ms-selfsub | ms-subdomain | ms-subdomain-self-rhs | name | self | selfsub | selfwild | subdomain | tcp-self | wildcard | zonesub ) [ <string> ] <rrtypelist>; ... } );
	zero-no-soa-ttl <boolean>;
	zone-statistics ( full | terse | none | <boolean> );
};
zone <string> [ <class> ] {
	type redirect;
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	dlz <string>;
	file <quoted_string>;
	masterfile-format ( raw | text );
	masterfile-style ( full | relative );
	max-records <integer>;
	max-zone-ttl ( unlimited | <duration> );
	primaries [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	zone-statistics ( full | terse | none | <boolean> );
};
zone <string> [ <class> ] {
	type secondary;
	allow-notify { <address_match_element>; ... };
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	allow-transfer [ port <integer> ] [ transport <string> ] { <address_match_element>; ... };
	allow-update-forwarding { <address_match_element>; ... };
	also-notify [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	alt-transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	alt-transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	auto-dnssec ( allow | maintain | off );
	check-names ( fail | warn | ignore );
	database <string>;
	dialup ( notify | notify-passive | passive | refresh | <boolean> );
	dlz <string>;
	dnskey-sig-validity <integer>;
	dnssec-dnskey-kskonly <boolean>;
	dnssec-loadkeys-interval <integer>;
	dnssec-policy <string>;
	dnssec-update-mode ( maintain | no-resign );
	file <quoted_string>;
	forward ( first | only );
	forwarders [ port <integer> ] [ dscp <integer> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ dscp <integer> ]; ... };
	inline-signing <boolean>;
	ixfr-from-differences <boolean>;
	journal <quoted_string>;
	key-directory <quoted_string>;
	masterfile-format ( raw | text );
	masterfile-style ( full | relative );
	max-ixfr-ratio ( unlimited | <percentage> );
	max-journal-size ( default | unlimited | <sizeval> );
	max-records <integer>;
	max-refresh-time <integer>;
	max-retry-time <integer>;
	max-transfer-idle-in <integer>;
	max-transfer-idle-out <integer>;
	max-transfer-time-in <integer>;
	max-transfer-time-out <integer>;
	min-refresh-time <integer>;
	min-retry-time <integer>;
	multi-master <boolean>;
	notify ( explicit | master-only | primary-only | <boolean> );
	notify-delay <integer>;
	notify-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	notify-to-soa <boolean>;
	nsec3-test-zone <boolean>; // test only
	parental-agents [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	parental-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	parental-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	primaries [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	request-expire <boolean>;
	request-ixfr <boolean>;
	sig-signing-nodes <integer>;
	sig-signing-signatures <integer>;
	sig-signing-type <integer>;
	sig-validity-interval <integer> [ <integer> ];
	transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	try-tcp-refresh <boolean>;
	update-check-ksk <boolean>;
	use-alt-transfer-source <boolean>;
	zero-no-soa-ttl <boolean>;
	zone-statistics ( full | terse | none | <boolean> );
};
zone <string> [ <class> ] {
	type static-stub;
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	forward ( first | only );
	forwarders [ port <integer> ] [ dscp <integer> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ dscp <integer> ]; ... };
	max-records <integer>;
	server-addresses { ( <ipv4_address> | <ipv6_address> ); ... };
	server-names { <string>; ... };
	zone-statistics ( full | terse | none | <boolean> );
};
zone <string> [ <class> ] {
	type stub;
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	check-names ( fail | warn | ignore );
	database <string>;
	delegation-only <boolean>;
	dialup ( notify | notify-passive | passive | refresh | <boolean> );
	file <quoted_string>;
	forward ( first | only );
	forwarders [ port <integer> ] [ dscp <integer> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ dscp <integer> ]; ... };
	masterfile-format ( raw | text );
	masterfile-style ( full | relative );
	max-records <integer>;
	max-refresh-time <integer>;
	max-retry-time <integer>;
	max-transfer-idle-in <integer>;
	max-transfer-time-in <integer>;
	min-refresh-time <integer>;
	min-retry-time <integer>;
	multi-master <boolean>;
	primaries [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	use-alt-transfer-source <boolean>;
	zone-statistics ( full | terse | none | <boolean> );
};

```

##### class
上面的 class 一般默认指**IN**类型。

##### `zone ZONE_NAME`
定义解析库名字。通常和解析库文件前缀对应起来。

##### type
type定义该域的类型是`master|slave|stub|hint|forward`中的哪种。

**master**
指的是主dns解析。

**slave**
指的是从dns解析
> 注：slave服务器不需要配置具体的`zone`文件以及`zone`引导配置。因为slave启动成功后，会自动同步master里面的RR记录，并且表明同步来的zone记录的type是slave类型，以及 master的`ip`地址。

**hint**
指的是根服务器。
![](attachments/Pasted%20image%2020240116161732.png)
![](attachments/Pasted%20image%2020240116162134.png)
为什么要根服务器列表？
> 因为所有域名的 DNS 记录查询入口都在这里，就好像你要进入一个多级的目录，你得一级一级地进入，而根服务器就是入口。


所有递归服务器上都会内置这张表。

```bash
zone "." IN {  
    type hint;  
    file "named.ca";  
};
```

```bash
# cat /var/named/named.ca

; <<>> DiG 9.11.3-RedHat-9.11.3-3.fc27 <<>> +bufsize=1200 +norec @a.root-servers.net
; (2 servers found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46900
;; flags: qr aa; QUERY: 1, ANSWER: 13, AUTHORITY: 0, ADDITIONAL: 27

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1472
;; QUESTION SECTION:
;.				IN	NS

;; ANSWER SECTION:
.			518400	IN	NS	a.root-servers.net.
.			518400	IN	NS	b.root-servers.net.
.			518400	IN	NS	c.root-servers.net.
.			518400	IN	NS	d.root-servers.net.
.			518400	IN	NS	e.root-servers.net.
.			518400	IN	NS	f.root-servers.net.
.			518400	IN	NS	g.root-servers.net.
.			518400	IN	NS	h.root-servers.net.
.			518400	IN	NS	i.root-servers.net.
.			518400	IN	NS	j.root-servers.net.
.			518400	IN	NS	k.root-servers.net.
.			518400	IN	NS	l.root-servers.net.
.			518400	IN	NS	m.root-servers.net.

;; ADDITIONAL SECTION:
a.root-servers.net.	518400	IN	A	198.41.0.4
b.root-servers.net.	518400	IN	A	199.9.14.201
c.root-servers.net.	518400	IN	A	192.33.4.12
d.root-servers.net.	518400	IN	A	199.7.91.13
e.root-servers.net.	518400	IN	A	192.203.230.10
f.root-servers.net.	518400	IN	A	192.5.5.241
g.root-servers.net.	518400	IN	A	192.112.36.4
h.root-servers.net.	518400	IN	A	198.97.190.53
i.root-servers.net.	518400	IN	A	192.36.148.17
j.root-servers.net.	518400	IN	A	192.58.128.30
k.root-servers.net.	518400	IN	A	193.0.14.129
l.root-servers.net.	518400	IN	A	199.7.83.42
m.root-servers.net.	518400	IN	A	202.12.27.33
a.root-servers.net.	518400	IN	AAAA	2001:503:ba3e::2:30
b.root-servers.net.	518400	IN	AAAA	2001:500:200::b
c.root-servers.net.	518400	IN	AAAA	2001:500:2::c
d.root-servers.net.	518400	IN	AAAA	2001:500:2d::d
e.root-servers.net.	518400	IN	AAAA	2001:500:a8::e
f.root-servers.net.	518400	IN	AAAA	2001:500:2f::f
g.root-servers.net.	518400	IN	AAAA	2001:500:12::d0d
h.root-servers.net.	518400	IN	AAAA	2001:500:1::53
i.root-servers.net.	518400	IN	AAAA	2001:7fe::53
j.root-servers.net.	518400	IN	AAAA	2001:503:c27::2:30
k.root-servers.net.	518400	IN	AAAA	2001:7fd::1
l.root-servers.net.	518400	IN	AAAA	2001:500:9f::42
m.root-servers.net.	518400	IN	AAAA	2001:dc3::35

;; Query time: 24 msec
;; SERVER: 198.41.0.4#53(198.41.0.4)
;; WHEN: Thu Apr 05 15:57:34 CEST 2018
;; MSG SIZE  rcvd: 811
```


**forward**
指的是转发（取值为`first` or `only`），需要配置`forwarders`字段。
> 注：`forward `字段只有当forwarders 列表中有内容的时候才有意义。当值是 first，默认情况下，使服务器先查询设置的forwarders，如果它没有得到回答，服务器就会查询全局options转发器寻找答案。如果设定的是 only，服务器就只会把请求转发到指定的服务器上去。

##### allow-update
是否允许客户主机或服务器自行更新dns记录。


##### file
指定存放zone的数据库文件名称；一般是相对路径，这个路径相对于您在 `options` 语句中的目录中创建的 `directory`（默认为：`/var/named`） 相对。

`file`的前缀通常和`zone`的名字通常对应起来，然后加一个.zone的后缀。

##### in-view
多个view共享 zone配置。

> BIND9.10+ only. "in-view", was added that lets multiple views refer to the same in-memory instance of a zone. Allows a zone clause within one view to be used by another view. This allows both views to use the same zone without the overhead of loading it more than once.
> The view-name must refer to a valid view which contains a zone of the same name and the view containing the zone must have been previously defined (only backward references to views are allowed, not forward references).

```json
格式：
in-view "view-name";

范例：
in-view "internal";
```


#### 特殊域名的配置
##### 根域名”.”的区域配置
```bash
zone "." IN {  
    type hint;  
    file named.ca;  
}
```
type hint表示该区域”.”类型为hint。
回顾dns解析流程，在客户端让dns服务器迭代查询时，迭代查询的第一步就是让dns服务器去找根域名服务器。但是dns服务器如何知道根域名服务器在哪里？这就是hint类型的作用，它提示dns服务器根据其区域数据文件named.ca中的内容去获取根域名地址，并将这些数据缓存起来，下次需要根域名地址时直接查找缓存即可。
> 因此，只有根区域”.”才会设置为hint类型。

##### ”localhost”域名
”localhost”域名(用于解析localhost为127.0.0.1)和127.0.0.1的方向查找区域。
```bash
zone "localhost" IN {  
        type master;  
        file "named.localhost";  
        allow-update { none; };  
};  
   
zone "1.0.0.127.in-addr.arpa" IN {  
        type master;  
        file "named.loopback";  
        allow-update { none; };  
};
```

当然，反向查找区域可以定义为域而不是直接定义成主机。但这样的话，就需要相对应地修改/var/named/named.loopback文件。例如：
```bash
zone "0.0.127.in-addr.arpa" IN {  
        type master;  
        file "named.loopback";  
        allow-update { none; };  
};
```

#### 范例

范例如下所示：
```json
 zone example.com {
  // shed load of statements
  // type may be master, slave etc.
  ...
 };
};

view "gondor" {
 ...
 zone example.com {
  in-view "mordor";
  // valid back-reference to previous defined view
  // a forward reference to "khand" would fail
  // forward and forwarders statements are allowed - nothing else
 };
};

view "khand" {
 ...
 zone example.com {
  in-view "mordor";
 };
};
```


>注：自定义的配置解析库文件（Zone files）,一般是在/var/named下写，文件名格式一般写为ZONE_NAME.zone。


### zone数据库配置(Zone文件)
zone文件：保存 RR (Record Resource) 信息的文件。
zone文件包括正向Zone文件和反向Zone文件。
DNS正解: 域名解析->返回IP；
DNS反解: IP解析->域名；
> 注：在 zone file 中，注释的符号是`;`。

#### 区域文件的写法
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

#### 正向区域文件
```json
[root@linuxmaster named]# vi /var/named/aliyun.com.zone
$TTL 300;
@   IN SOA  linuxmaster.aliyun.com. admin.aliyun.com. (
                    2017051720      ;serial
                    1H          ;refresh
                    5M          ;retry
                    7D          ;expiry
                    3D )        ;minimum
    IN  NS  linuxmaster
    IN  MX 20 MX
linuxmaster IN  A   172.24.8.10
www         IN  A   172.24.8.30
mirrors     IN  A   172.24.8.30
ftp         IN  CNAME   www
```

在 zone file （区域数据库文件）中：注释的符号是：`;`

比如：
```josn
;
; BIND data file for local loopback interface
;
; Import ZSK / KSK
;
;
$ORIGIN domain.com.
; 我们已经定义了一个区域，那么在定义 SOA 的时候可以进行两种定义方式
@ IN SOA ns.domain.com. admin.domain.com. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
; 或者我们不需要 at-sign - @ 符号，直接引用ORIGIN的名字
；在这里这两条配置代表的含义是一样的
domain.com. ns.domain.com. admin.domain.com. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
```

##### `$`符号
`$`符号：定义宏。
最常见的是`$TTL`、`$ORIGIN`。origin的的意思是起点，可以自行定义`$ORIGIN`的值。

**`$ORIGIN [REGION NAME]`**
 设定了当前的解析域名区域；
`$ORIGIN longshuai.com.`表示`$ORIGIN`的值为”longshuai.com.”，未定义`$ORIGIN`时，值为`zone domain-name`的 `domain-name` 部分。

```josn
;
; BIND data file for local loopback interface
;
; Import ZSK / KSK
;
;
$ORIGIN domain.com.
; 我们已经定义了一个区域，那么在定义 SOA 的时候可以进行两种定义方式
@ IN SOA ns.domain.com. admin.domain.com. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
; 或者我们不需要 at-sign - @ 符号，直接引用ORIGIN的名字
；在这里这两条配置代表的含义是一样的
domain.com. ns.domain.com. admin.domain.com. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
```


注： **可以在zone文件中多次定义 $ORIGIN  的值(类似于一个zone下的子zone，但其zone引导配置中不存在这些子zone，引导配置中都是这个zone。)**。
如下所示：

zone 引导配置如下所示：
```bash
zone "testdns.internal" {
	type master;
	file "bjfs/db.testdns.internal"
}
```

zone 配置文件(`bjfs/db.testdns.internal`)如下所示：
```bash
$ORIGIN .
@ IN SOA master-dns-test-1.testdns.internal. admin.domain. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )
    NS  master-dns-test-1.testdns.internal.
    NS  slave1-dns-test-1.testdns.internal.
    NS  slave2-dns-test-1.testdns.internal.

master-dns-test-1  A  1.1.1.2
slave1-dns-test-1  A  1.1.1.3
slave2-dns-test-1  A  1.1.1.4

$ORIGIN test2.testdns.internal
t1 A  2.2.2.4
t2 A  2.2.2.5
t3 A  2.2.2.6

$ORIGIN test3.testdns.internal
t1 A  3.2.2.4
t2 A  3.2.2.5
t3 A  3.2.2.6

$ORIGIN test4.testdns.internal
t1 A  4.2.2.4
t2 A  4.2.2.5
t3 A  4.2.2.6
```

##### fqdn自动补齐
在区域数据文件中，没有使用点号”.”结尾的，在实际使用的时候都会自动补上域名(即`$ORIGIN`)，使其变为fqdn。
例如区域”longshuai.com.”，以下是完全格式的资源记录：
```bash
dnsserver.longshuai.com.    IN  A   172.16.10.15
```
可以缩写为：
```bash
dnsserver    IN    A   172.16.10.15
```
因为`dnsserver`后没有点，所以会补齐整个域名”longshuai.com.”。
实际上，自动补齐的部分是`$ORIGIN`的值，只不过默认没定义`$ORIGIN`时，`$ORIGIN`的值为zone定义的域名，所以默认是自动补齐域名。

##### `@`符号
可以使用@符号来缩写`$ORIGIN`的值。
由于自定义的区域数据文件中，一般不会主动定义`$ORIGIN`的值，而第一个资源记录一般都是SOA记录，所以此时SOA记录中的第一列就可以使用`@`符号，其它地方只要值为`$ORIGIN`，都可以使用@符号缩写。例如：

```bash
@     IN  SOA  dnsserver  xyz. ( 1 3h 1h 1w 1h )  
@     IN  NS   dnsserver
```
##### 重复最近一个名称
**区域数据文件中的第一列可以使用空格或制表符使该列继承上一行的第一列的值。**

例如第一行定义的是SOA记录，第一列是”longshuai.com.”，那么第二行定义的NS记录中，其第一列就可以留空来继承第一行第一列的”longshuai.com.”。不止第一行和第二行，第三行也可以继承第二行的第一列，第n+1行也可以继承第n行的第一列，只要它们的值一样即可。
```bash
# vim /var/named/db.longshuai.com 
$TTL 6h
@         IN  SOA    dnsserver   xyz ( 1 3h 1h 1w 1h )
@         IN  NS     dnsserver
 
dnsserver IN  A      172.16.10.15
www       IN  A      172.16.10.16
ftp       IN  A      172.16.10.17
mydb      IN  A      172.16.10.18
 
www1      IN  CNAME  www
```
##### TTL
TTL：定义区域中数据文件里面的各项记录的默认TTL值，单位为秒；
> 对于  Negative RR 即 Nxdomain 记录的 TTL 则是权威服务器的SOA 的 minimum 作为超时时间。

RR都会被保存在DNS的解析服务器的cache上，有一个失效的时间，TTL就是控制这个失效时间的一个参数。
在区域数据文件中，`$TTL`的定义表示其后的记录都以此TTL为准，直到遇到下一个`$TTL`。也就是说，两个`$TTL`之间的所有记录都以前面的`$TTL`为准。不过大多数时候，一个区域数据文件中只会在第一行定义一个`$TTL`值，表示该文件中所有记录都使用该缓存周期值。

这个参数可以单独进行设定，也可以在 SOA 设定中进行配置。
- 单独设定： `$TTL [TIME]`
- 在 SOA 中进行设定： `SOA - Negative Cache TTL`

##### SOA记录
SOA：SOA记录，@代表相应的域名，每个区域数据文件只能有一个SOA，其中参数如下：

![](attachments/Pasted%20image%2020240416123049.png)

- serial：表示配置文件的修改版本，格式为年月日加上修改的次数；
- refresh：设定辅助dns和主dns进行同步的间隔时间；
```bash
slave服务器间隔一段时间查询master中的zone的serial number是否发生增加。
目前master每次进行zone的更新之后，都会给slave发送对应的 notify通知slave存在zone更新。

This is time(in seconds) when the slave DNS server will refresh from the master. This value represents how often a secondary will poll the primary server to see if the serial number for the zone has increased (so it knows to request a new copy of the data for the zone).

```
- retry：如果辅助dns进行更新失败后，间隔多久进行重试；
```bash
slave服务器连接不上master服务器（即master没有响应），间隔多久进行再次尝试请求SOA，单位为秒；

取值：retry通常小于 refresh。

Now assume that a slave tried to contact the master server and failed to contact it because it was down. The Retry value (time in seconds) will tell it when to get back. This value is not very important and can be a fraction of the refresh value.
```
- expiry：设定辅助dns与主dns同步失败后，多长时间后清除对应的记录；
```bash
当slave服务器连接不上master时，slave可以在多长时间内认为其zone是有效的，并且供用户进行查询。单位为秒；超时之后，还是无法连接，slave将删除这份zone。

取值：expire 必须大于 REFRESH_和 RETRY 的和。

This is the time (in seconds) that a slave server will keep a cached zone file as valid, if it can’t contact the primary server. If this value were set to say 2 weeks ( in seconds), what it means is that a slave would still be able to give out domain information from its cached zone file for 2 weeks, without anyone knowing the difference. The recommended value is between 2 to 4 weeks.
```
- minimum ：
使用的场景如下所示：
```
- 定义了当该域无记录时，缓存的时长，即 Nodomain 记录的缓存时间。（区别于有记录时缓存的时长 `TTL`）
Signed 32 bit value in seconds. RFC 2308 redefined this value to be the negative caching time - the time a NAME ERROR = NXDOMAIN record is cached.

- 默认的 TTL 值。（zone file中 无 TTL 变量时使用该值作为dns响应中的RR记录的TTL值）

```

**zone file中的TTL 和 minimum区别**
```bash
$ORIGIN .
$TTL 86400  ; 1 day
internal    IN SOA  master-aaa-kdns-test-k-41.internal. admin.internal. (
        1712866142 ; serial
        10800      ; refresh (3 hours)
        900        ; retry (15 minutes)
        604800     ; expire (1 week)
        86400      ; minimum (1 day)
        )
      NS  master-aaa-kdns-test-k-41.internal.
      NS  slave-aaa-kdns-test-k42.internal.
      NS  slave-aaa-kdns-test-k43.internal.
      NS  slave-aaa-kdns-test-k-40.internal.
slave-aaa-kdns-test-k-40 A  10.108.164.22
slave-aaa-kdns-test-k42 A 10.108.164.24
slave-aaa-kdns-test-k43 A 10.108.164.25
sms-api     A 10.20.253.166
      A 10.20.254.105
```

![](attachments/Pasted%20image%2020240416121939.png)


- **注意**：
>在所有的配置中，`ns.domain.com != ns.domain.com.` ，必须注意在 zone file 中的配置文件的最后 `.` 必须不能省略；
>如果不写最后一个的 `.` 那么该域名就是一个 **相对名** ，结果就是在解析的过程中，这条资源就被当成 `ns.domain.com.domain.com`

##### NS记录
Name Server Records 定义了在当前 区域的 DNS服务器 ；
每个NS必须有对应的 IP地址，即在每一个 zone file 中必须指定 主/从 域名解析器的IP地址（`A` 记录/`AAAA`记录）。

```json
; 记录 NS 记录
@               IN          NS          ns.domain.com.
; 记录 NS 记录对应的 IP 地址信息
ns.domain.com.          IN          A           192.168.1.1
```
##### A记录
Address Records 记录了 域名 与 IP 地址的对应关系；
```json
ns.domain.com. IN A 192.168.1.1
```

##### Cname记录
CName 将 单个昵称或者别名映射到一个可能存在在区域外的真实的区域，在一个域名下存在多个子域名。
如果需要更改映射之前的子域名，那么只需要更改映射的域名地址就可以了；
```json
; 
$TTL 2d
$ORIGIN domain.com.
...
server1     IN      A       192.168.1.1
www             IN      CNAME       Server1
```


#### 反向区域文件
反向查找是根据ip地址查找其对应的主机名。
在/etc/named.conf中，需要定义`zone  *.in-addr.arpa`，其中`*`是点分十进制ip的反写，可以是反写ip后的任意一段长度，例如`127.0.0.1`反写后是`1.0.0.127`，所以zone所定义的可以是”1.0.0.127”、”0.0.127”、”0.127”，甚至是”127”，长度位数不同，在区域数据文件中需要补全的数值就不同。
另外，在zone文件中，反解域也是要有SOA记录的，在反解域中ns记录就不用在写A记录了，因为反解区域文件只可以有PTR，不可以有A记录。反向查找区域的各种缓存时间可以都设置长一点，因为用的不多。

##### 范例
***定义区域引导配置**
```bash
zone "16.172.in-addr.arpa" IN {
    type master;
    file "172.16.zone";
};
```

**创建区域数据文件**
```bash
(1) 修改权限：
# chown :named 172.16.zone 
# chmod o-r 172.16.zone

(2) 编辑内容
$ORIGIN 16.172.in-addr.arpa.
$TTL 86400
@   IN  SOA ns1.qhdlink.com. admin.qhdlink.com. (
                            2017081001;serial
                            1H;refresh
                            15M;retry
                            1W;expire
                            1D);TTL

    IN  NS ns1.qhdlink.com.
1.72  IN  PTR     ns1.qhdlink.com.
1.100 IN  PTR     www.qhdlink.com.
2.100 IN  PTR     www.qhdlink.com.
3.100 IN  PTR     www.qhdlink.com.
4.100 IN  PTR     mx1.qhdlink.com.
```

**重载区域和配置文件**
```bash
# named-checkconf -----------检测named.conf语法
# named-checkzone "ZONE_NAME" FILE_NAME ---------检测域名、区域数据库文件语法
# rndc reload ------------重载配置文件
# systemctl reload named.service
```

#### slave和master中的zone配置差异
**master(primary)中的zone配置**：
```json
# zone引导配置
zone "abc.com" IN {
    type primary;
    file "/usr/local/bind/etc/named.abc.com";  // RR记录文件名称可以自定义 最好取有意义名称
};


# zone配置文件
# cat /usr/local/bind/etc/named.abc.com
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

slave(secondary)不需要配置对应的zone配置，因为secondary启动成功后，会自动同步primary里面的RR记录。在named.conf中的配置也不一样，如下所示：
```json
zone "abc.com" IN {
    type secondary;
    file "/usr/local/bind/etc/named.abc.com";
    primarys {192.168.10.200;};
};
```

### 正向区域文件的配置范例
这里给出一个自定义的总的区域定义文件，新加一个区域文件的定义：
```c
zone "zhangqifei.top" IN {
    type master;
    file "zhangqifei.top.zone";
};
```

/var/named/ZONE_NAME.zone区域配置文件：这里给出我一个自定义的区域文件：zhangqifei.top.zone
```json
$TTL 1D
@    IN  SOA  ns1.zhangqifei.top. me.zhangqifei.top ( 0 1H 10M 1D 3H)
     IN  NS   ns1.zhangqifei.top
     IN  NS   ns2
         MX 10 mail1
         MX 20 mail2
ns1.zhangqifei.top  IN  A  192.168.111.254
ns2  IN  A  192.168.111.253  
db1   A    192.168.111.100
db2   A    192.168.111.111
web1  A    192.168.111.200
web2  A    192.168.111.222
mail1 A    192.168.111.10
mail2 A    192.168.111.20
www   CNAME   web1
```
说明：
```c
@指的就是本域 zhangqifei.top
IN ns2.zhangqifei.top可以省略写成IN ns2
IN都可以省略不写,比如直接写MX mail


设置.zone文件权限（参照/var/named/name.xxxx的权限来设置，这里xxx为任意字符）
chmod 640 zhangqifei.top.zone
chown :named zhangqifei.top.zone

配置完毕后，启动或重启(已启动的话）服务。
service named start|restart
```

修改文件属性：
```c
将新创建的zone文件修改为和/var/named下的其他的zone文件相同的属性：

chown named:named /var/named/zhangqifei.top.zone
chmod 640 /var/named/zhangqifei.top.zone
```
检查配置文件：
```c
named-checkzone  zhangqifei.top  /var/named/zhangqifei.top.zone
```

测试：
```c
dig ns2.zhangqifei.top
```
### 反向区域文件的配置范例

**范例一**：
编辑`/etc/named.rfc1912.zones`，添加一个域名
```c

zone "111.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.111.zone";
};
配置/var/named/ZONE_NAME.zone
不需要MX、A、AAAA，要有SOA、NS记录，PTR记录。
```
然后配置192.168.111.zone：
```c
$TTL 1D
@   IN  SOA ns1.magedu.com. me.zhangqifei.top (
                    20170001
                    1H
                    5M
                    7D
                    1D)
    IN NS ns1.zhangqifei.top.
    IN NS ns2.zhangqifei.top.
254 IN PTR ns1.zhangqifei.top.
253 IN PTR ns2.zhangqifei.top.
100 IN PTR db1.magedu.com.
111 IN PTR db2.magedu.com.
200 IN PTR web1.magedu.com.
222 IN PTR web2.magedu.com.
10  IN PTR mail1.magedu.com.
20  IN PTR mail2.magedu.com.
```

检查区域配置文件是否正常：
```c
named-checkzone 111.168.192.in-addr.arpa /var/named/192.168.111.zone
```


**范例二**：
编辑`/etc/named.rfc1912.zones`，添加一个域名：
```json
zone "31.168.192.in-addr.arpa" IN {
        type master;
        file "31.168.192.in-addr.arpa.zone";
        allow-update { 192.168.31.113; };};

#检查配置文件
[root@dns01-113 ~]# named-checkconf
```

在`/var/named/`下创建反解域区域数据库文件：
```json
[root@dns01-113 ~]# vim /var/named/31.168.192.in-addr.arpa.zone     #名称需要根据上面定义的来创建
$TTL  60
@         IN  SOA    dns.host.com.    604419314.qq.com. (
                     20200818
                     10800
                     900
                     64800
                     86400
                     )
              NS     dns.host.com.

$ORIGIN  31.168.192.in-addr.arpa.
$TTL 60 
113          PTR     dns01-113.host.com.
114          PTR     dns02-114.host.com. 
#####################
提示: 如果我们$ORIGIN 31.168.192.in-addr.arpa.不写全，也是可以的。例如 
$ORIGIN in-addr.arpa.
$TTL 60
113.31.168.192   PTR   dns01-113.host.com.
#上面的写法也是可以的
```

说明: 反解域也是要有SOA记录的，在反解域中ns记录就不用在写A记录了，因为反解区域文件只可以有PTR，不可以有A记录

检查区域配置文件是否正常：
```json
#31.168.192.in-addr.arpa为我们反向解析的域名 
[root@dns01-113 ~]# named-checkzone 31.168.192.in-addr.arpa. /var/named/31.168.192.in-addr.arpa.zone
zone 31.168.192.in-addr.arpa/IN: loaded serial 20200818
OK
```

重启`named`：
```json
systemctl restart named
```

测试：
```bash
#第一种方法
# dig -t PTR 113.31.168.192.in-addr.arpa. @192.168.31.113  +short
dns01-113.host.com. 

#第二种方法
# dig -x 192.168.31.114 @192.168.31.113 +short
dns02-114.host.com. 

#dig 后面ip为需要解析的ip，@后面ip为dns服务器ip
```

## 主配置文件中的acl 
一般来说，ACL模块用来控制哪些主机可以访问域名解析服务器，使用`ACL`访问控制列表可以使配置简单而清晰，一次定义之后可以在多处使用，不会使配置文件因为大量的 IP 地址而变得混乱。
采用这个配置可以有效防范`DOS`以及`Spoofing`攻击。

ACL匹配客户端是否能够接入到域名服务器基于三个基本的特征:
- 客户端的IPv4或者IPv6地址
- 用于签署请求的 TSIG 和 SIG(0) 密钥
- 在DNS客户端子网选项中编码的前缀地址

### ACL格式
```json
acl ACL_NAME {
    <address_match_element>;
    ...
};
```

范例如下所示：
```json
# 范例一：
acl innet {
    10.189.9.0/24;
    127.0.0.1/8;
    192.168.0.0/24;
};

options {
    directory "/var/named";
    allow-recursion { innet; };
};

# 范例二：
acl "someips" {                               //定义一个名为someips的ACL  
  10.0.0.1; 192.168.23.1; 192.168.23.15;      //包含3个单个IP  
};
 
acl "complex" {             //定义一个名为complex的ACL  
  "someips";                //可以嵌套包含其他ACL  
  10.0.15.0/24;             //包含10.0.15.0子网中的所有IP  
  !10.0.16.1/24;            //非10.0.16.1子网的IP  
  {10.0.17.1;10.0.18.2;};   //包含了一个IP组  
  localhost；               //本地网络接口IP（含实际接口IP和127.0.0.1）  
};
```

**named的内置ACL**：
- `none` : 没有一个主机
- `any` : 任意
- `local/localhost` : 本机
- `localnet` : 本机IP所属的网络
### EDNS中ACL的使用
#### 匹配client的ACL方式
![](attachments/Pasted%20image%2020240120113126.png)
- Client的 IP
- TSIG Key
- ECS

#### 注意
鉴于bind acl并非是最精确匹配，只是线性匹配；
配置的时候必须要注意 View的配置顺序。
> 注：ACL的定义和VIew的定义是并行的，并不是ACL在View中定义。



### ACL 配置语句
定义了 ACL 之后，可以在任何可以使用ACL匹配的子句中使用。
下面简单介绍常用的子句：

|子句|模块|说明|
|---|---|---|
|allow-query|options,zone|指定哪些主机或网络可以查询本服务器的区域，默认的是允许所有主机进行查询。|
|allow-transfer|options,zone|指定哪些主机允许和本地服务器进行域传输，默认值是允许和所有主机进行域传输。|
|allow-recursion|options|指定哪些主机可以进行递归查询。如果没有设定，缺省是允许所有主机进行递归查询的。注意禁止一台主机的递归查询，并不能阻止这台主机查询已经存在于服务器缓存中的数据。|
|allow-update|zone|指定哪些主机允许为主域名服务器提交动态 DNS 更新。默认为拒绝任何主机进行更新。|
|blackhole|options|指定不接收来自哪些主机的查询请求和地址解析。默认值是 none 。|

注意：上面列出的一些配置子句既可以出现在全局配置 **options** 语句里，又可以出现在 **zone** 声明语句里，当在两处同时出现时，**zone** 声明语句中的配置将会覆盖全局配置 **options** 语句中的配置。

## 主配置文件中的view
`view`语句定义了视图功能。视图是`BIND9`提供的强大的新功能，允许`DNS`服务器根据客户端的不同，有区别地回答`DNS`查询。
比如，不同机房/不同运营商的用户查询某个域名，得到不同的A记录。

**视图语句的顺序是很重要的**。**一位用户的请求将会在它所匹配的第一个视图中被解答**。
### View 格式

在`view语`句中定义的`zone`只对匹配视图的用户是可用的。
通过在多个视图中用相同名称定义一个域(但是对应的不同的zone文件)，不同域数据可以传给不同的用户。

视图的定义格式，如下所示：
```json
view <string> <optional_class> {
        match-clients { <address_match_element>; ... };
        match-destinations { <address_match_element>; ... };
        match-recursive-only <boolean>;
        key <string> {
                algorithm <string>;
                secret <string>;
        };
        zone <string> <optional_class> {
                type ( master | slave | stub | hint | forward );
                allow-update { <address_match_element>; ... };
                file <quoted_string>;
                ixfr-base <quoted_string>; // obsolete
                ixfr-tmp-file <quoted_string>; // obsolete
                masters [ port <integer> ] { ( <ipv4_address> |
                    <ipv6_address> ) [ port <integer> ] [ key <string> ]; ... };
                pubkey <integer> <integer> <integer> <quoted_string>; //
                    obsolete
                update-policy { ( grant | deny ) <string> ( name |
                    subdomain | wildcard | self ) <string> <rrtypelist>; ... };
                database <string>;
                check-names <string>; // not implemented
                allow-query { <address_match_element>; ... };
                allow-transfer { <address_match_element>; ... };
                allow-update-forwarding { <address_match_element>; ... };
                allow-notify { <address_match_element>; ... };
                notify <notifytype>;
                notify-source ( <ipv4_address> | * ) [ port ( <integer> | *
                    ) ];
                notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer>
                    | * ) ];
                also-notify [ port <integer> ] { ( <ipv4_address> |
                    <ipv6_address> ) [ port <integer> ]; ... };
                dialup <dialuptype>;
                forward ( first | only );
                forwarders [ port <integer> ] { ( <ipv4_address> |
                    <ipv6_address> ) [ port <integer> ]; ... };
                maintain-ixfr-base <boolean>; // obsolete
                max-ixfr-log-size <size>; // obsolete
                transfer-source ( <ipv4_address> | * ) [ port ( <integer> |
                    * ) ];
                transfer-source-v6 ( <ipv6_address> | * ) [ port (
                    <integer> | * ) ];
                max-transfer-time-in <integer>;
                max-transfer-time-out <integer>;
                max-transfer-idle-in <integer>;
                max-transfer-idle-out <integer>;
                max-retry-time <integer>;
                min-retry-time <integer>;
                max-refresh-time <integer>;
                min-refresh-time <integer>;
                sig-validity-interval <integer>;
                zone-statistics <boolean>;
        };
        server {
                bogus <boolean>;
                provide-ixfr <boolean>;
                request-ixfr <boolean>;
                support-ixfr <boolean>; // obsolete
                transfers <integer>;
                transfer-format ( many-answers | one-answer );
                keys <server_key>;
                edns <boolean>;
        };
        trusted-keys { <string> <integer> <integer> <integer>
            <quoted_string>; ... };
        allow-recursion { <address_match_element>; ... };
        allow-v6-synthesis { <address_match_element>; ... };
        sortlist { <address_match_element>; ... };
        topology { <address_match_element>; ... }; // not implemented
        auth-nxdomain <boolean>; // default changed
        minimal-responses <boolean>;
        recursion <boolean>;
        provide-ixfr <boolean>;
        request-ixfr <boolean>;
        fetch-glue <boolean>; // obsolete
        rfc2308-type1 <boolean>; // not yet implemented
        additional-from-auth <boolean>;
        additional-from-cache <boolean>;
        query-source <querysource4>;
        query-source-v6 <querysource6>;
        cleaning-interval <integer>;
        min-roots <integer>; // not implemented
        lame-ttl <integer>;
        max-ncache-ttl <integer>;
        max-cache-ttl <integer>;
        transfer-format ( many-answers | one-answer );
        max-cache-size <size_no_default>;
        check-names <string> <string>; // not implemented
        cache-file <quoted_string>;
        allow-query { <address_match_element>; ... };
        allow-transfer { <address_match_element>; ... };
        allow-update-forwarding { <address_match_element>; ... };
        allow-notify { <address_match_element>; ... };
        notify <notifytype>;
        notify-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ];
        notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ];
        also-notify [ port <integer> ] { ( <ipv4_address> | <ipv6_address>
            ) [ port <integer> ]; ... };
        dialup <dialuptype>;
        forward ( first | only );
        forwarders [ port <integer> ] { ( <ipv4_address> | <ipv6_address> )
            [ port <integer> ]; ... };
        maintain-ixfr-base <boolean>; // obsolete
        max-ixfr-log-size <size>; // obsolete
        transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ];
        transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ];
        max-transfer-time-in <integer>;
        max-transfer-time-out <integer>;
        max-transfer-idle-in <integer>;
        max-transfer-idle-out <integer>;
        max-retry-time <integer>;
        min-retry-time <integer>;
        max-refresh-time <integer>;
        min-refresh-time <integer>;
        sig-validity-interval <integer>;
        zone-statistics <boolean>;
};
```

**`match_clients`和`matach-destinations`**：
match_clients 可以设置为 TSIG，ACL名称（client的sip，或 ECS网段）。
一般情况下，想要测试某个View，要么选择特定的Client-ip的测试机器，或者选择 ECS方式（需要bind9编译的时候就支持ECS），或者在任意一个测试机器上，dig测试时指定TSIG。相比较之下，指定TSIG的方式更加通用。如下所示：
```bash
dig @1.1.1.1 t1999.dhb.test11.internal -y hmac-sha256:bjfs_idc_key:"xxxxxxxxxx" +short
```


**`match-recursive-only`**
一个视图也可以做为match-recursive-only来指定，意思是来自匹配用户的递归请求将会匹配该视图。

**`class`**
视图精确到类。如果没有给定任何类，就假设为IN类。如果在配置文件中没有view语句，在IN类中就会自动产生一个默认视图匹配于任何用户，任何指定在配置文件的zone配置被看作是此默认视图的一部分。

### 注意实现
(1).所有的`zone`都必须要定义在`view`中，尽管默认的`named.conf`根本没有定义`view`，但是此时所有的`zone`都定义在了一个隐含的默认的视图中。

(2). view中`match-clients`指令的匹配方式是**从前向后匹配**的，如果在第一个view中匹配了，则后面定义的view将不会生效，所以**定义的view的先后顺序是很重要的**。

(3).绝大多数`named.conf`中的指令都能写在`view`中，只有很少量的指令不允许。
例如`acl`指令。对于本该封装在`options`中的指令，如果想定义在`view`中，则不应该在`view`中使用`options`，因为`options`定义的是全局默认值，配置文件中只能出现一次，所以可以直接在view中写指令，这样会覆盖全局options。

(4).不同的view中定义的相同的zone，它们使用的区域文件一般不同（并非必须不同），否则就没有自定义view的必要。

### View实现智能DNS
ACL及视图的配合使用，可实现智能DNS的实现：

**主配置文件中的内容**：
```json
# cat /etc/named.conf
    acl telecom {
        10.189.9.0/24;    
    }; 
                #此ACL定义了电信IP的列表
    acl unicom {
        10.189.8.0/24;
    };
                #此ACL定义了联通IP的列表
    options {
        directory "/var/named";
        allow-recursion {   localnet; };   #定义了允许递归请求的主机为所在网络的所有主机
        allow-query { any; };
    };

    view  telecom {
        match-clients { telecom; };
        zone "mageedua.com" IN {
            type master;
            file "telecom.mageedua.com.zone";
        };
    };

    view unicom {
        match-clients { unicom; };
        zone "mageedua.com" IN {
            type master;
            file "unicom.mageedua.com.zone";
        };
    };
```
在`named.conf`配置文件中，配置`view`功能，并在视图 `view` 区域定义`match-clients `参数，让`match-clients`引用`acl`列表，`acl`可以为电信IP列表，或者联通IP列表。
最后在`view`视图中定义需要智能DNS的区域。一般情况，可以将区域划分三类： 
1、内网视图  
2、电信视图 
3、联通视图

**不同视图的zone文件配置**：
电信视图的zone文件：
```json
# cat telecom.mageedua.com.zone
    $TTL 86400
    @   IN  SOA ns1.mageedua.com.   admin.mageedua.com. (
            2015061101
            1D
            1H
            7D
            1D )
        IN  NS  ns1
        IN  MX  10    mail
    ns1 IN  A   10.189.9.110
    mail    IN  A   10.189.9.111
    www IN  A   10.189.9.112   #电信www服务器为112的地址
```

联通视图的zone文件：
```json
# cat unicom.mageedua.com.zone
    $TTL 86400
    @   IN  SOA ns1.mageedua.com.   admin.mageedua.com. (
            2015061101
            1D
            1H
            7D
            1D )
        IN  NS  ns1
        IN  MX  10    mail
    ns1 IN  A   10.189.9.110
    mail    IN  A   10.189.9.111
    www IN  A   10.189.9.113  #联通www服务器为113的地址
```

测试（ #电信用户查询结果，如下所示）：
```
dig -t A www.mageedua.com @10.189.9.110  

    ; <<>> DiG 9.9.4-RedHat-9.9.4-14.el7_0.1 <<>> -t A www.mageedua.com @10.189.9.110
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58349
    ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2
    ;; WARNING: recursion requested but not available

    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 4096
    ;; QUESTION SECTION:
    ;www.mageedua.com.      IN  A

    ;; ANSWER SECTION:
    www.mageedua.com.   86400   IN  A   10.189.9.112

    ;; AUTHORITY SECTION:
    mageedua.com.       86400   IN  NS  ns1.mageedua.com.

    ;; ADDITIONAL SECTION:
    ns1.mageedua.com.   86400   IN  A   10.189.9.110

    ;; Query time: 1 msec
    ;; SERVER: 10.189.9.110#53(10.189.9.110)
    ;; WHEN: 四 6月 11 10:26:21 CST 2015
    ;; MSG SIZE  rcvd: 95
```

## 主配置文件中的logging 块
### 背景
在默认情况下，bind 9把日志消息写到`/var/log/messages`文件中，而这些日志消息是非常少的，主要就是启动，关闭的日志记录和一些严重错误的消息，所以要详细记录服务器的运行状况，需要自己配置服务器的日志行为，也就是要在配置文件`named.conf`中使用`logging`语句来定制自己所需要的日志记录。

### 介绍
`logging` 部分的配置为DNS解析服务器提供了日志记录的功能，DNS服务器上的所有日志记录被存储到了指定的文件中。其通用的配置文件为:
```json
logging {
      category <string> { <channel_name_string>; ... };
      channel <string> {
              buffered <boolean>;
              file <quoted_string> [ versions ( unlimited | <integer> ) ]
                  [ size <size> ] [ suffix ( increment | timestamp ) ];
              null;
              print-category <boolean>;
              print-severity <boolean>;
              print-time ( iso8601 | iso8601-utc | local | <boolean> );
              severity <log_severity>;
              stderr;
              syslog [ <syslog_facility> ];
      };
};
```
从上边的通用配置格式可以看出来，`logging` 模块分为两个部分，`category` 和 `channel`。
- `channel`
`channel`的作用是指定输出的文件、日志格式和事件的严重性。
每一个`channel`可以指定一个 `category` 来指定记录的事件类型。

- `category`
 `category` 定义哪些类的信息需要写入日志。
 用来区分不同的类别或者场景，比如：客户端请求（`client request`）、配置文件解析处理（`Configuration file parsing and processing`）。

每种类别的数据可以被发送给一个或多个通道。例如统计类别的信息发送到系统日志通道且发送到自定义日志文件通道，而查询类别只发送到自定义日志文件通道。
![](attachments/Pasted%20image%2020240117165623.png)

```bash
logging {  
    channel my_syslog {          /* 定义日志通道my_syslog */   
        syslog daemon;           /* my_syslog通道的日志写入到系统日志syslog，并指定使用daemon工具记录 */   
        severity warning;        /* 该通道只记录严重级别为warning及以上的信息 */   
    };  
    channel my_file {           /* 定义第二个日志通道my_file */   
        file "log.msgs";        /* my_file通道的日志写入到自定义的文件log.msgs中 */   
        severity info;          /* 该通道记录日志级别为info 及以上的信息 */   
    };  
    category statistics {my_syslog;my_file;};  /* 统计类别statistics的信息使用my_syslog和my_file通道记录 */   
    category queries {my_file;};               /* 查询类别queries的信息记录使用my_file通道记录 */   
    category default {null;};                  /* 除了上述两个自定义的类别，其余绝大部分的类别的信息都丢弃 */     
//category default { my_file; };           /* 或者不丢弃，将其记录到my_file通道指定的文件也可以 */   
};
```

### 原理
首先需要创建一个 `channel` 来规定输出日志流的格式还以及日志文件名、文件版本。
每一个 `channel` 可以被多个 `category` 调用使用，每一个 `category` 相当于一个 `BIND9` 内嵌的服务模块，服务模块去调用日志配置模最后输出格式化日志。

### `channel` 的配置
**`channel`配置规则**：
`channel` 的配置规则：
- 所有的日志输出都需要 channel 来指定输出格式，BIND9 对于创建 channel 的数量没有限制。
- 每一个 channel 都需要为该通道的日志信息指定一个 `destination clause` - 目的句柄；
 
#### 通道的目的位置
通道的有四种位置可选
>1）输出到具体的文件的名字 - `file`；
file:使用自定义的文件路径。可以指定该日志文件有几个版本(versions)和多大(size)时就进行轮替，不指定大小限制时日志文件将无限制增长。
例如以下定义方式：每个日志文件增长到10M大小就创建新的文件来记录新的日志，最多记录3个版本的日志文件，当第三个日志文件my_logs2也到10M后将删除最早的日志文件继续记录。
```bash
channel my_file {  
    file my_logs versions 3 size 10M;  
    severity info;  
};
```

>2）输出到具体的系统日志工具中（syslog/syslogd）- `syslog`；
syslog:使用系统日志来记录，同时需要指定使用哪种工具(facility)来记录。
有kern、user、mail、daemon、auth、syslog、lpr、news、uucp、cron、authpriv、ftp、local0、local1、local2、local3、local4、local5、local6 和local7这么多种工具可选。默认是daemon，也建议使用daemon。

>3）输出到终端显示- 标准错误流(standard error stream)；
使用stderr方式的通道将把使用该通道来记录的信息输出到stderr。

>4）或者该错误消息直接被丢弃 - `null`。
使用null方式的通道将把使用该通道来记录的信息全部丢弃。

#### 通道的其他配置
channel 的配置可以规定每一个错误日志消息的响应级别，默认的响应级别是`info`。
channel 还可以控制输出错误日志消息的格式，可以包含：响应时间戳、category名字、严重等级等（`print-time boolean`;`print-severity boolean`和`print-category boolean`）。

#### `channel` 的配置参数
- `buffered`: 用来规定是否刷新错误日志的文件，其参数值为`<boolean>`，在 BIND9 中 `<boolean>` 值的参数值为 `yes` / `no`，如果设置成为 `yes` 那么日志消息流(一般每一个错误日志消息都是一个 Log Entry)就不会刷新，而是被保存在缓冲区中了，不会刷新到文件中。
    
- `file`： 类似于Linux的通道概念，file 将日志输出流通过通道直接输出给文件，从上边的通用配置可以看出来可以为 file 指定文本文件的大小 - `size` ；指定 log 文件的版本号 - `version`；指定用于命名备份版本的格式 - `suffix`；
> - `size` ：用来限制log文件的大小，如果log文件的大小设置超过了设定的阈值，那么系统会自动停止输出内容到文件中；   
> - `versions`： 用于指定新创建的 log文件数存储到本地的上限值，默认的参数值为`unlimited`，当指定的文件的大小超过设定的`size`值得时候，如果没有指定 `versions`，那么系统就不会继续写进log；如果制定了`versions`，那么就会继续写入；比如指定5个版本（version 5），当access.log达到size的大小，则会保存为query.log.0，以此类推一直保存到access.log.4。


> - `suffix` ：设定用来命名`log`文件的方式；

- `syslog`：将通道定向到系统的日志文件流中； 常用的支持日志文件服务为：`dameon`、`syslog`、`local6`、`local7`；
- `severity`：用来指定记录消息的级别，相当于 `syslog` - `priorities`。
比如说定义了日志的严重级别为 `Debug`，那么会输出日志事件 `Debug` 以上的错误到文件中。按照严重性递减的顺序，如下所示：
```c
1. critical
2. error
3. warning
4. notice
5. info
6. debug 
7. dynamic


其中前5种级别和系统日志syslog系统相同，后两种(debug和dynamic) 是BIND独有，且debug还可以按照level进行细分。
但是写日志是一项非常消耗性能的操作，所以默认都是定义在info级别上。在正常使用环境中，除了调试时可能需要记录debug或者dynamic级别信息，其余至少都记录到info级别甚至更严格的级别。
```

- `stderr`：将通道指向服务器的标准错误流。这是为了在服务器作为前台进程运行时使用；
- `print-time`： `yes` / `no` / `local` / `iso8601` / `iso8061-utc` 可以设定不同的输出到日志文件的时间格式；
- `print-category`：打印日志消息配置`category` 的信息到你设定的日志文件中；
- `print-severity`： 打印日志的严重等级


### `category` 的配置
`category`词组配置规则：`category <config_string> { <channel_name_string>; ... };`

#### `category` 分类

|类别|说明|
|---|---|
|default|匹配所有未明确指定通道的类别，但是不匹配不属于任何类别的消息。这些不属于任何类别的消息属于下面列出的这些类别。|
|general|包括所有未明确分类的bind消息|
|client|处理客户端请求|
|database|同bind内部数据库相关的消息，用来存储区数据和缓存记录|
|dnssec|处理DNSSEC签名的响应|
|lame-servers|发现错误授权|
|network|网络操作|
|notify|异步区域变动通知|
|queries|查询日志|
|resolver|名字解析，包括对来自解析器的递归查询的处理|
|security|认可/非认可的请求|
|update|动态更新事件|
|xfer-in|从远程名称服务器到本地名称服务器的区传送|
|xfer-out|从本地名称服务器到远程名称服务器的区传送|


### 默认的channel和类别
当没有在`named.conf`中定义任何`logging`字句时，发现在`/var/log/messages`中还是记录了很多DNS相关的日志，为什么没定义还是有记录呢？
因为有BIND默认定义的日志记录方式。

在BIND中默认定义了4个channel，分别使用4种通道路径。这些channel你无法重定义它们，即使你不想要它们、不书写它们，BIND还是会自己创建它们。
只有一种方法：新添加通道并指定使用该通道的类别，使你想记录的日志不使用默认的定义。

以下是默认定义的通道：
```bash
channel default_syslog {  
    syslog daemon;  
    severity info;  
};  
channel default_debug {  
    file "named.run";  
    severity dynamic;  
};  
channel default_stderr {  
    stderr;  
severity info;  
};  
channel null {  
    null;  
};
```

在BIND 9 中所指定的默认类别语句如下:
```bash
category default { default_syslog;default_debug; };
```

也就是说，对于你没有指定的绝大多数类别的信息都使用通道`default_syslog`和通道`default_debug`来记录。由于默认定义了null通道，所以你想把`default`匹配的类别信息全部丢弃的话，可以直接使用`categroy default { null; };`来丢弃。

### 范例
```json
logging{
    channel default_logfile {
        file "/var/log/bind/example_1.log" versions 3 size 1m;
        print-category yes;
        print-severity yes;
        print-time yes;
        severity info;
    };

    channel default_querylogfile{
        file "/var/log/bind/query.log" versions 3 size 1m;
        print-category yes;
        print-severity yes;
        print-time yes;
        severity info;
    };

    channel default_databaselogfile{
        file "/var/cache/bind/data/database.log" versions 3 size 3m;
        print-time yes;
        print-severity yes;
        severity debug;
    };

    category default {
        default_logfile;
    };
    category queries {
        default_querylogfile;
    };
    category database {
        default_databaselogfile;
    };
    category unmatched { null; };
};
```


## 主配置文件中的 control 块
在/etc/named.conf中，可以使用controls指令来设置接收控制消息的通道，以及允许控制本机的控制者。
定义方式如下：
```bash
controls {  
    inet local_ip port PORT_NUM  allow { control_ip_list; } 
    keys { "rndc-key"; };  
};
```

范例：
```bash
# rndc controls
controls {
    inet * port 953 allow { any; }
    keys { kwai_rndc_key; rndc-key; };
};
```

在上述格式中：

**local_ip和PORT_NUM**
设置的是dns服务器开启的tcp通道，表示监听在本机某个IP地址local_ip上的PORT_NUM端口上。
其中local_ip可以使用`*`表示监听在本机所有地址上，port可以省略，默认监听端口为953。

**allow关键字**
定义允许连接本通道的主机列表，也就是限定谁能控制本机dns服务器。

**keys关键字**
定义的是连接本通道时需要进行密钥认证，只有认证通过的才能成功连接通道。这个key在后面介绍rndc时会说明。

## 安全性配置
### 背景
DNS 的设计起始于上世纪 80 年代，当时互联网规模小得多，安全性并非首要设计考虑因素。当DNS的递归解析器(DNS Resolver )向权威域名服务器发送查询请求的时候，解析器无法验证响应的真实性。解析器只是检查响应的IP地址与解析器发送初始查询的IP地址是否相同。
但是， 这种检查机制存在一个非常严重的问题：权威域名服务器的请求包的源地址非常容易被伪造和仿冒，攻击者很容易伪装成解析器最初查询的权威域名服务器，因此可以很轻松将用户重定向到其他的恶意网站。因为需要考虑到递归解析器可能被入侵的场景。

域名解析器有一种缓存的机制用来加速解析流程，其原理是从权威域名服务器和根解析器时候收到的上次DNS数据存储到本地来加速下一次应答，避免查询一个或者多个权威服务器引发的服务延迟。然而，依靠缓存存在一个缺点，如果攻击者发送的假冒的DNS权威域名服务器响应被接受，那么其缓存就会被一直缓存在递归解析服务器中，因此之后所有的查询DNS Server 的主机都会接收到欺诈性DNS数据。

### 方案
####  `allow-query`
首先， BIND9 的 `options` 提供的 `allow-query` 参数可以对访问DNS解析服务器的客户端地址和网段进行限制，从而达到对陌生客户端和恶意IP地址的封禁。

#### `DNSSEC`
BIND 提供了一种名叫 `DNSSEC` 的安全扩展服务以及配套的密钥生成工具供网络人员使用。

DNSSEC 采用基于公钥加密的数字签名技术来加强 DNS的验证强度。 DNSSEC 并非对DNS查询和响应的过程进行传输加密，而是数据所有者对 DNS 数据进行签名加密。

DNSSEC 在 DNS协议中新增加的两个安全的功能：
- 数据来源验证 ：验证收到的数据是否确实来自其认定的数据传送区域。
- 数据完整性保护：验证自从区域所有者使用私钥第一次进行数字签名，数据在传输过程中并未遭到修改。

每个zone都会发布一个公钥，递归解析器可以检索公钥以验证区域中的数据，区域的公钥也必须经过签名。
>每一个 DNS Zone 在配置的时候 BIND 服务会为其创建一个公钥和私钥的配对的秘钥对。 DNS zone 的所有者使用该区域的私钥对区域内的DNS数据进行加密，并为这些数据生成签名。 DNS 解析器生成的 公钥 对该区域内所有的 DNS数据验证真实性。如果有效，证明DNS数据合法。


# DNS测试工具
常见的DNS测试工具为 `dig`, `host`, `nslookup` 等。

# 参考
```c
# BIND9 - 最详细、最认真的从零开始的 BIND 9
https://www.cnblogs.com/doherasyang/p/14464999.html#11-dns

# DNS基础解析(3)
https://zhuanlan.zhihu.com/p/590032092

# Linux之DNS服务Bind
https://www.jianshu.com/p/149057c56344

# bind9 的官方英文文档
https://bind9.readthedocs.io/en/latest/

# bind9 
https://kb.isc.org/docs/aa-00726

# Centos DNS服务(二)-bind主从配置与基于TSIG加密的动态更新
https://developer.aliyun.com/article/486620

# DNS和bind从基础到深入
https://www.junmajinlong.com/linux/dns_bind/index.html


# Bind 之recursion递归
http://www.hangdaowangluo.com/archives/1631
```