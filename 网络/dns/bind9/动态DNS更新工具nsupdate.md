```table-of-contents
```
# 介绍
 nsupdate是一个动态DNS更新工具，可以向主DNS服务器提交更新记录的请求，它可以从区文件中添加或删除资源记录，而不需要手动进行编辑zone文件。

​ 使用`nsupdate` 不会更改区域数据库文件，而是产生了一个`jnl`的数据文件，不能使用文本编辑器打开，只能使用完全区域数据传送查看。
此时添加进入的DNS记录是能够正常提供服务的(即 `named`的内存中存在更改后的记录)，如果需要实时更新到区域文件中，需要使用`rndc sync`且需要注意区域文件的文件权限。**一般情况下在15分钟内，Bind会将`jnl`文件转储到区域文件中**。


注：jnl文件（journal文件）是BIND9**动态更新**的时候记录更新内容所生成的日志文件。

# update消息
使用 `TCP 53`端口，进行 DNS的`update`更新，如下所示：

## 范例
```bash
说明：
10.108.164.23 是 dns服务器；
10.110.166.146 是 执行 nsupdate的机器；
```
**dns更新请求**
![](attachments/Pasted%20image%2020240318204321.png)


**dns更新响应**
![](attachments/Pasted%20image%2020240318204442.png)

# 使用
## 语法
```bash
语法：
    nsupdate [-dD] [-L level] [-l][-g | -o | -y keyname:secret | -k keyfile] [-v] [filename]

-d 调试模式。
-k 从keyfile文件中读取密钥信息。
-y keyname是密钥的名称,secret是以base64编码的密钥。
-v 使用TCP协议进行nsupdate.默认是使用UDP协议。
filename:可以从终端或文件中读取命令.每个命令一行；一个空行或一个”send”命令,则会将先前输入的命令发送到DNS服务器上
```


**必须的条件**：
指定的zone语句块或全局中添加：allow-update { any; }; 或 allow-update { IP范围; };
> 如果 "allow-update" 为 none，表示不允许动态更新。

```
（1）指定密钥
# nsupdate -k /etc/named/dns-key

（2）添加删除命令
update delete|add {domain-name} [ttl] [class] [type [data...]]

说明：
	class不指定的话，默认是IN
	ttl可以单独写一行，后面的行继承此值
```

**更新一条记录**
不支持直接更新，需要先执行删除，再新增。

## 使用小结
nsupdate使用小结：
- 优点
    - 不用手动变更SOA的serial序列号，自动滚动
    - 不需要重启/重载BIND9服务/配置，生效快
    - 可以通过配置acl实现远程管理

- 缺点
    - jnl文件无法使用文本文件的方式打开
    - 只能依赖完全区域传送查看所有区域的记录
    - 更新操作复杂，先删再增
    - 远程管理有安全隐患，需要加强审计
    - 动态域在rndc管理上多一步

# 范例

```
zone "hunk.tech" IN {
        type slave;                     #type类型修改为master
        masters { 192.168.4.200; }       #定义主DNS服务器IP
        file "slaves/named.hunk.tech";  #定义辅助DNS数据库文件
};

```

具体的zone文件。
```
$ORIGIN .
$TTL 600        ; 10 minutes
hunk.tech               IN SOA  6-DNS-1.hunk.tech. admin.hunk.tech. (
                                0          ; serial
                                7200       ; refresh (2 hours)
                                600        ; retry (10 minutes)
                                86400      ; expire (1 day)
                                10800      ; minimum (3 hours)
                                )
                        NS      6-DNS-1.hunk.tech.
                        NS      6-DNS-2.hunk.tech.
                        MX      10 mail.hunk.tech.
$ORIGIN hunk.tech.
*                       A       192.168.0.206
6-DNS-1                 A       192.168.4.200
6-DNS-2                 A       192.168.4.201
6-WEB-1                 A       192.168.4.205
                        A       192.168.4.206
7-WEB-2                 A       192.168.4.206
mail                    A       192.168.4.205
www                     CNAME   6-WEB-1
```
> 本来应该是给出主服务器的 zone配置，以及zone文件。此中为了方便，给出的是从服务上的信息。

范例如下所示：
```
#nsupdate
> server 192.168.7.254     > 指定动态更新的DNS服务器。`192.168.7`是主服务器的IP
> zone hunk.tech            > 指定更新的zone
> update add hunk.tech 600 IN A 192.168.7.204 >添加A记录
> update delete hunk.tech A 192.168.7.204     >删除A记录
> send                  > 发送更新指令到DNS服务器
> quit
```

**从文件读取指令**
```
#nsupdate update_dns.txt
文件中的指令与交互式是一样的，一行一条指令
```


**查看区域数据库文件**:
`/var/named`  产生了一个`jnl`的数据文件，不能使用文本编辑器打开。jnl文件（journal文件）是BIND9动态更新的时候记录更新内容所生成的日志文件。


# 多个view的时候使用nsupdate更新记录
## 背景
经常使用bind的时候是划分不同的view的，因为每个view的zone需要单独修改，所以人肉修改是比较麻烦的。这个时候可以使用nsupdate进行批量的操作。只要注意每个view使用正确的记录就行。

## 方法
使用`nsupdate`需要给每个`view`都创建一个key，每个view指定允许对应的这个key能更新。  

**views.key文件**：
```bash
key "default" {  
    algorithm hmac-md5;  
    secret "GkbQ6Q2WtVqu9pk8WzPDOA==";  
};  
key "test1" {  
    algorithm hmac-md5;  
    secret "4qEjC+NgFmRvGdt8DuCRDA==";  
};  
key "test2" {  
    algorithm hmac-md5;  
    secret "88PUPwk66CbQacWCgFG0kw==";  
};
```

**named.conf文件**：
```bash
controls {
    inet 127.0.0.1 port 953
    allow { 127.0.0.1; } keys { "rndc-key"; };
};
//
acl test1 {
    10.0.0.0/8;
};
acl test2 {
    192.0.0.0/8;
};
acl slavedns {  
        192.18.208.31; //ztt dns1

        127.0.0.1;
};
options {
     listen-on port 53 { any; };
     listen-on-v6  { none; };
     directory      "/opt/bind/etc/";
     dump-file      "/opt/bind/var/named/data/cache_dump.db";
     statistics-file "/opt/bind/var/named/data/named_stats.txt";
     memstatistics-file "/opt/bind/var/named/data/named_mem_stats.txt";
     zone-statistics yes;
     allow-query     { any; };
# recursion config
     recursion yes;
     max-ncache-ttl 60;
     recursive-clients 2000;
# dnssec config
     dnssec-enable yes;
     dnssec-validation yes;
     dnssec-lookaside auto;
# rrt config
     rate-limit {
        responses-per-second 20;
        qps-scale  1000;
        window 4;
        slip 2;
        ipv4-prefix-length 32;
    };
# rpz config
    response-policy {
        zone "rpz.zone"  policy given;
   };
# log query
      querylog yes;
# define version
      version "GNUer's dns 2.0";
# transfer config
      notify explicit;
      tcp-clients 2000;
      transfers-out 100;
      allow-transfer {  slavedns; 127.0.0.1;};
      also-notify {
                192.18.208.31;

    };
     /* Path to ISC DLV key */
     #bindkeys-file "/opt/bind/etc/named.iscdlv.key";
};

logging {
  channel default_syslog { file "/opt/bind/var/log/named.syslog" versions 5 size 100m; severity dynamic; print-time yes;};
  channel default_debug { file "/opt/bind/var/log/named.run" versions 5 size 100m; severity dynamic; print-time yes;};
  channel default_stderr { stderr; severity info; };
  channel null { null; };
  channel general_debug { file "/opt/bind/var/log/named.general" versions 3 size 100m; severity dynamic; print-time yes;};
  channel database_debug { file "/opt/bind/var/log/named.database" versions 3 size 100m; severity dynamic; print-time yes;};
  channel query_log { file "/opt/bind/var/log/named.query" versions 3 size 100m; severity dynamic; print-time yes;print-severity yes; print-category yes;};
  channel resolver_log { file "/opt/bind/var/log/named.resolver" versions 3 size 100m; severity dynamic; print-time yes;};
  channel security_log { file "/opt/bind/var/log/named.security" versions 3 size 100m; severity dynamic; print-time yes;};
  channel notify_log { file "/opt/bind/var/log/named.notify" versions 3 size 100m; severity dynamic; print-time yes;};
  channel rrt_log { file "/opt/bind/var/log/named.rrt" versions 3 size 100m; severity dynamic; print-time yes;};
  channel rpz_log { file "/opt/bind/var/log/named.rpz" versions 3 size 100m; severity dynamic; print-time yes;};
  category default {null; };
  category queries { query_log; };
  category resolver { resolver_log; };
  category security { security_log; };
  category notify { notify_log; };
  category xfer-in { notify_log; };
  category xfer-out { notify_log; };
  category update { notify_log; };
  category unmatched {default_syslog; };
  category rate-limit {rrt_log;};
  category rpz {rpz_log;};
};
view "test1" {
    recursion yes;
    allow-query { any; };
    match-clients {test1; key test1;};
    allow-update { key test1; };
    zone "test.org" {
        type master;
        file "master/test.org.view1";
    };      
    zone "rpz.zone" {
       type master;
       file "master/rpz.zone";
       allow-update {none;};
   };
   zone "."{
     type hint;
     file "named.root";
   };
};

view "test2" {
    recursion yes;
    allow-query { any; };
    match-clients {test2; key test2;};
    allow-update { key test2; };
    zone "test.org" {
        type master;
        file "master/test.org.view2";
    };      
    zone "rpz.zone" {
       type master;
       file "master/rpz.zone";
       allow-update {none;};
   };
   zone "."{
     type hint;
     file "named.root";
   };
};
view "default" {
    recursion yes;
    allow-query { any; };
    match-clients {any;key default; };
    allow-update { key default; };
    zone "test.org" {
        type master;
        file "master/test.org.default";
    };      
    zone "rpz.zone" {
       type master;
       file "master/rpz.zone";
       allow-update {none;};
   };
   zone "."{
     type hint;
     file "named.root";
   };
};
```


**nsupdate脚本**:
```bash
#!/bin/bash
TTL=600
declare -A views
views["test1"]="4qEjC+NgFmRvGdt8DuCRDA=="
views["test2"]="88PUPwk66CbQacWCgFG0kw=="
views["default"]="GkbQ6Q2WtVqu9pk8WzPDOA=="
usage(){
    echo "$0 view add/delete type domain record"
    echo "$0 view mod type1:type2 domain record1:record2"
    exit 1
}
if [ $# -ne 5 ];then
    usage

fi
view=$1
action=$2
dtype=$3
domain=$4
target=$5
case $2 in
add|delete)
    #echo "update $action $domain 600 $dtype $target"
    nsupdate -y "$view:${views[$view]}" &lt;&lt;-EOF
            server 127.0.0.1
            update $action $domain $TTL $dtype $target
            send
EOF
    if [ $? -eq 0 ];then
        echo -e "update $domain --&gt; $ntarget \e[1;32msuccessfull\e[m"
    else
        echo -e  "update $domain --&gt; $ntarget \e[1;31mfailed\e[m"

    fi
    ;;
mod)
    otype=$(echo $dtype |cut -d: -f1)
    ntype=$(echo $dtype |cut -d: -f2)
    otarget=$(echo $target|cut -d: -f1)
    ntarget=$(echo $target|cut -d: -f2)
    nsupdate -y "$view:${views[$view]}" &lt;&lt;-EOF
        server 127.0.0.1
        update delete $domain $TTL $otype $otarget
        update add $domain $TTL $ntype $ntarget
    send
EOF
    if [ $? -eq 0 ];then
        echo -e "update $domain --&gt; $ntarget \e[1;32msuccessfull\e[m"
    else
        echo -e  "update $domain --&gt; $ntarget \e[1;31mfailed\e[m"

    fi
    ;;
*)
    usage
    ;;
esac
```

## 范例
给ax3.test.org.新增A记录10.20.1.33
```bash
./nsupdate.sh test2 add A  ax3.test.org. 10.20.1.33
```

给ax3.test.org.删除A记录10.20.1.33
```bash
./nsupdate.sh test2 delete A  ax3.test.org. 10.20.1.33
```

把ax3.test.org.从A记录10.20.1.3修改为cname到www.baidu.com.
```bash
./nsupdate.sh test2 mod A:CNAME  ax3.test.org. 10.20.1.3:www.baidu.com.
```

把ax3.test.org.从cname到www.baidu.com.修改为A记录10.20.1.3
```bash
./nsupdate.sh test2 mod CNAME:A  ax3.test.org. www.baidu.com.:10.20.1.3
```
# 参考
```c
# bind主从配置与基于TSIG加密的动态更新
https://developer.aliyun.com/article/486620

# 构建企业级DNS系统（六）DNS动态更新
https://blog.csdn.net/u011288801/article/details/106870698
```