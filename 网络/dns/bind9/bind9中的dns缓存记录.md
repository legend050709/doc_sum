```table-of-contents
```
# 介绍

# 配置

```bash
optoins{
...
[ max-ncache-ttl number; ]
[ max-cache-ttl number; ]

[ cleaning-interval number; ]
...
# [ additional-from-auth yes_or_no ; ]
[ additional-from-cache yes_or_no ; ]
[ max-cache-size size_spec ; ]
...
};
```

**max-cache-size**

服务器缓冲使用的最大内存量，用比特表示。但在缓存数据的量达到这个界限，服务器将会使记录提早过期这样限制就不会被突破。在多视图的服务器中，限制分别使用于每个视图的缓存。默认值没有限制，意味着只有当总的限制被突破的时候记录才会被缓存清除。


**cleaning-interval**
服务器将每隔`cleaning-interval`的时间从缓存中清除过期的资源记录。默认为60分钟，如果设置为0，就不会有周期性清理。

**max-ncache-ttl**
为降低网络流量和提升服务器存储否定回答的性能。`max-ncache-ttl`以秒为单位设定这些回答的保存时间。默认`max-ncache-ttl`是10800秒(3小时)。`max-ncache-ttl`不能超过7天，如果设成一个更大的值，则将会被自动减为7天。

**max-cache-ttl**
`max-cache-ttl`设定了服务器储存普通(肯定)答案的最大时间。默认值一周(7 天)。



# 查看缓存
首先我们来看看如何查看所有缓存的域名解析：
```bash
# rndc dumpdb -cache
```

上面的命令将把bind的缓存转储到`/var/cache/bind/named_dump.db`中。如果执行上述命令后找不到该文件，请检查服务器的配置文件以显示缓存转储文件的位置。

要查看缓存的 dns 记录，只需 `cat` 或 `grep` 生成的转储文件。例如：
```bash
# grep gnu.org /var/named/data/cache_dump.db
gnu.org.                86358   NS      ns1.gnu.org.
                        86358   NS      ns2.gnu.org.
                        86358   NS      ns3.gnu.org.
ns1.gnu.org.            86358   A       208.118.235.164
ns2.gnu.org.            86358   A       87.98.253.102
ns3.gnu.org.            86358   A       46.43.37.70
```

# 清除缓存
```bash
rndc
  flush 	Flushes all of the server's caches.
  flush [view]	Flushes the server's cache for a view.
  flushname name [view]
		Flush the given name from the server's cache(s)
  flushtree name [view]
		Flush all names under the given name from the server's cache(s)
```
如果您希望清除 Bind 服务器的缓存，以下 Linux 命令将为您提供帮助。首先，刷新所有缓存条目
```bash
# rndc flush
```
完成后，重新加载绑定：
```bash
# rndc reload
```
# 参考
```bash# bind9的配置文件中的配置解释 
https://chengqian90.com/DNS/DNS%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B9%8BBIND9.html

# BIND配置文件详解（二）（++++++++++++）
https://developer.aliyun.com/article/471229

```