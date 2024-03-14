```table-of-contents
```
# 概述
写了一个简单的脚本，输入域名可以生成使用iptables封禁dns请求的规则，使用于内部递归dns的一些基本防护。

# 脚本
```bash
#!/bin/bash
create_iptables(){
    name=$1
    string=$(  echo $name|awk -F '.' '{for(i=1;i&lt;=NF;i++){printf("|%02x|%s",length($i),$i)}}')
    string="$string|00|"
    echo  -e "rule for \e[1;32m $name \e[mquery:"
    echo  "iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "$string" --algo bm -j LOG --log-prefix "drop an dns query ""
    echo  "iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "$string" --algo bm -j DROP"
}
create_iptables_ex(){
    name=$1
    declare -A types
    types["A"]="0001|"
    types["MX"]="000f|"
    types["CNAME"]="0005|"
    types["ANY"]="00ff|"
    types["AAAA"]="001c|"
    types["NS"]="0002|"
    types["SOA"]="0006|"
    for t in ${!types[@]}
    do
        string=$(  echo $name|awk -F '.' '{for(i=1;i&lt;=NF;i++){printf("|%02x|%s",length($i),$i)}}')
        string="$string|00${types[$t]}"
        echo  -e "rule for \e[1;32m $name \e \e[1;31m $t \e[mquery:"
        echo  "iptables -t raw -A PREROUTING  -p udp --dport 53 -m string --hex-string "$string" --algo bm -j LOG --log-prefix "drop an dns query ""
        echo  "iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "$string" --algo bm -j DROP"
    done
}
if [ $# -lt 1 ];then
    echo -e "usage:\n -t $0 www.abc.com"
fi

for domain in $@
do
    echo "DROP $domain rule:"
    create_iptables $domain
    echo "DROP $domain TYPE rule:"
    create_iptables_ex $domain
done
```

# 范例
```bash
DROP baidu.com rule:
rule for  baidu.com query:
iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|00|" --algo bm -j LOG --log-prefix "drop an dns query "
iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|00|" --algo bm -j DROP
DROP baidu.com TYPE rule:
rule for  baidu.com   A query:
iptables -t raw -A PREROUTING  -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|000001|" --algo bm -j LOG --log-prefix "drop an dns query "
iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|000001|" --algo bm -j DROP
rule for  baidu.com   SOA query:
iptables -t raw -A PREROUTING  -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|000006|" --algo bm -j LOG --log-prefix "drop an dns query "
iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|000006|" --algo bm -j DROP
rule for  baidu.com   NS query:
iptables -t raw -A PREROUTING  -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|000002|" --algo bm -j LOG --log-prefix "drop an dns query "
iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|000002|" --algo bm -j DROP
rule for  baidu.com   CNAME query:
iptables -t raw -A PREROUTING  -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|000005|" --algo bm -j LOG --log-prefix "drop an dns query "
iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|000005|" --algo bm -j DROP
rule for  baidu.com   MX query:
iptables -t raw -A PREROUTING  -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|00000f|" --algo bm -j LOG --log-prefix "drop an dns query "
iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|00000f|" --algo bm -j DROP
rule for  baidu.com   AAAA query:
iptables -t raw -A PREROUTING  -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|00001c|" --algo bm -j LOG --log-prefix "drop an dns query "
iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|00001c|" --algo bm -j DROP
rule for  baidu.com   ANY query:
iptables -t raw -A PREROUTING  -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|0000ff|" --algo bm -j LOG --log-prefix "drop an dns query "
iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|0000ff|" --algo bm -j DROP
```

如果只是想封禁AAAA查询，可以使用–hex-string “|05|baidu|03|com|00000f|”，如果任意类型都想封禁就直接使用第一条
```bash
iptables -t raw -A PREROUTING -p udp --dport 53 -m string --hex-string "|05|baidu|03|com|00|" --algo bm -j DROP
```

原文里很多都是简单做正则匹配，会有误伤。本文通过在生成的规则里批评了最后的”.”所有只会封”baidu.com.”,不会封其他的baidu.com.xxx.com

# 参考
```bash
# 使用iptables过滤特定域名请求
https://blog.gnuers.org/?p=1060
```