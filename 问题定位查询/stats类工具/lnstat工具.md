```table-of-contents
```
# 概述
**lnstat命令**用来显示linux系统的网路状态。
lnstat命令实际上是读取系统“/proc”中目录“/proc/net/stat”下面的文件，来显示当前主机的网络状态的。lnstat命令是rtstat命令的更新替代命令，功能更完善。
![](attachments/Pasted%20image%2020231129202547.png)
> 如果开启了 contrack, 则会有 /proc/net/stat/nf_conntrack 文件。

# 使用
## 常用选项
```c
-c	指定显示网络状态的次数，每隔一定时间显示一次网络状态
-d	显示可用的文件或关键字
-i	指定两次显示网络状的间隔秒数
-k	只显示给定的关键字
-s	是否显示标题头
-w	指定每个字段所占的宽度
-h	显示帮助信息
-v	显示指令版本信息

```
## 范例
**显示命令支持的统计文件**
```c
[root@localhost ~]# lnstat -d
/proc/net/stat/nf_conntrack:
	 1: entries
	 2: searched
	 3: found
	 4: new
	 5: invalid
	 6: ignore
	 7: delete
	 8: delete_list
	 9: insert
	10: insert_failed
	11: drop
	12: early_drop
	13: icmp_error
	14: expect_new
	15: expect_create
	16: expect_delete
	17: search_restart
/proc/net/stat/ndisc_cache:
	 1: entries
	 2: allocs
	 3: destroys
	 4: hash_grows
	 5: lookups
	 6: hits
	 7: res_failed
	 8: rcv_probes_mcast
	 9: rcv_probes_ucast
	10: periodic_gc_runs
	11: forced_gc_runs
	12: unresolved_discards
	13: table_fulls
/proc/net/stat/arp_cache:
	 1: entries
	 2: allocs
	 3: destroys
	 4: hash_grows
	 5: lookups
	 6: hits
	 7: res_failed
	 8: rcv_probes_mcast
	 9: rcv_probes_ucast
	10: periodic_gc_runs
	11: forced_gc_runs
	12: unresolved_discards
	13: table_fulls
/proc/net/stat/rt_cache:
	 1: entries
	 2: in_hit
	 3: in_slow_tot
	 4: in_slow_mc
	 5: in_no_route
	 6: in_brd
	 7: in_martian_dst
	 8: in_martian_src
	 9: out_hit
	10: out_slow_tot
	11: out_slow_mc
	12: gc_total
	13: gc_ignored
	14: gc_goal_miss
	15: gc_dst_overflow
	16: in_hlist_search
	17: out_hlist_search
```



```c
lnstat -s1 -i1 -c-1 -f rt_cache
```

# 参考
```c

https://blog.csdn.net/dengjin20104042056/article/details/100126590

```