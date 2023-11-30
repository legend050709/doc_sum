```table-of-contents
```
# 介绍
基于策略的路由比传统路由在功能上更强大，使用更灵活，它使网络管理员不仅能够根据目的地址，而且能够根据报文大小、应用或IP源地址等属性来选择转发路径。
简单地来说，linux系统有多张路由表，而路由策略会根据一些条件，将路由请求转向不同的路由表。例如源地址在某些范围走路由表A，另外的数据包走路由表，类似这样的规则是有路由策略rule来控制。

![](attachments/Pasted%20image%2020231129175906.png)
在linux系统中，一条路由策略rule主要包含三个信息，即rule的优先级，条件，路由表。其中rule的优先级数字越小表示优先级越高，然后是满足什么条件下由指定的路由表来进行路由。

在linux系统启动时，内核会为路由策略数据库配置三条缺省的规则，即rule 0，rule 32766， rule 32767（数字是rule的优先级），具体含义如下：
（1）rule 0
匹配任何条件的数据包，查询路由表local（table id = 255）。rule 0非常特殊，不能被删除或者覆盖。 

（2）rule 32766
匹配任何条件的数据包，查询路由表main（table id = 254）。系统管理员可以删除或者使用另外的策略覆盖这条策略。

（3）rule 32767
匹配任何条件的数据包，查询路由表default（table id = 253）(ID 253) 。对于前面的缺省策略没有匹配到的数据包，系统使用这个策略进行处理。这个规则也可以删除。

# 路由匹配顺序
在linux系统中是按照rule的优先级顺序依次匹配。

假设系统中只有优先级为0，32766及32767这三条规则。那么系统首先会根据规则0在本地路由表里寻找路由，如果目的地址是本网络，或是广播地址的话，在这里就可以找到匹配的路由；
如果没有找到路由，就会匹配下一个不空的规则，在这里只有32766规则，那么将会在主路由表里寻找路由；
如果没有找到匹配的路由，就会依据32767规则，即寻找默认路由表；如果失败，路由将失败。

# 使用
## 查看
查看系统中所有的路由策略rule
```c
ip rule
ip rule list
```

查看指定路由表的内容
```c
ip route list table table_id
ip route list table table_name

ip route show table table_id
ip route show table table_name
```

> 注：可以通过命令ip rule或ip rule list来查看系统中所有的路由策略rule。另外使用ip rule命令配置的路由策略rule只在内存中有效，机器重启后，就会失效。可以将路由策略配置到文件/etc/sysconfig/network-scripts/rule-ethX中，这样机器重启后仍然有效。

# 其他
## table id
linux最多可以支持255张路由表，每张路由表有一个table id和table name。
查看 `/etc/iproute2/rt_tables` 可以看到 table name 和 table ID 的对应关系。
```c
# cat /etc/iproute2/rt_tables
#
# reserved values
#
255	local
254	main
253	default
0	unspec
#
# local
#
#1	inr.ruhep
```

其中有4张表是linux系统内置的：
（1）table id = 0
系统保留。

（2）table id = 255
称为本地路由表，表名为local。像本地接口地址，广播地址，以及NAT地址都放在这个表。该路由表由系统自动维护，管理员不能直接修改。

（3）table id = 254
称为主路由表，表名为main。如果没有指明路由所属的表，所有的路由都默认都放在这个表里。一般来说， 旧的路由工具(如route)所添加的路由都会加到这个表。main表中路由记录都是普通的路由记录。而且，使用ip route配置路由时，如果不明确制定要操作的路由表，默认情况下也是主路由表（表254）进行操作。
备注：我们使用ip route list 或 route -n 或 netstat -rn查看的路由记录，也都是main表中记录。

（4）table id = 253
称为默认路由表，表名为default。一般来说默认的路由都放在这张表。

备注：
A）系统管理员可以根据需要自己添加路由表，并向路由表中添加路由记录。
B）可以通过/etc/iproute2/rt_tables文件查看table id和table name的映射关系。

# 注意
路由策略rule指定满足一定条件的数据包有指定的路由表来路由，多个策略rule可以指向同一张路由表。
某些路由表可以没有策略指向它。值得注意的是，如果系统管理员删除了指向某个路由表的所有策略rule，那么这个路由表是没有用的，但它在系统中仍然存在，直到路由表中的所有路由记录被删除，它才会消失。

# 参考
```c

```