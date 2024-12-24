```table-of-contents
```
# 概述
3.6版本内核移除了FIB查询前的路由缓存，取而代之的是下一跳缓存，这在[路由缓存的前世今生](https://switch-router.gitee.io/blog/routecache/) 中已经说过了。
本文要说的是在该版本中引入的另一个概念：`FIB Nexthop Exception`，用于记录==下一跳的例外情形==。

# FIB Nexthop Exception  作用
我们知道，==FIB表项来源于路由配置(用户手动或者动态路由进程计算)，说到底，它们都来源于用户空间的设置。这些表项基本是稳定的==。
但内核实际发包时，还有两个情况需要考虑

- 收到过==`ICMP REDIRECT`报文==，表示之前发送的报文绕路了，之后的报文应该修改报文的下一跳。
- 收到过==`ICMP FRAGNEEDED`报文==，表示之前的报文太大了，路径上的一些设备不接受，需要源端进行分片。

这两种情况是==针对单一目的地址==的，什么意思呢？
已PMTU为例，在下面的网络拓扑中，我在主机A上配置了下面一条路由
![](attachments/Pasted%20image%2020231212150134.png)
```
设备A上的路由：
ip route add 4.4.0.0/16 via 4.4.4.1
```

当A发送一个长度为1500的IP报文给C时 ，中间的一台网络设备B觉得这个报文太大了，因此它向A发送`ICMP FRAGNEEDED`报文,说我只能转发1300以下的报文，请将报文分片。

A收到该报文后怎么办呢？总不能以后命中这条路由的报文全部按1300发送吧，因为并不是所有报文的路径都会包含B。
这时`FIB Nexthop Exception`就派上用场了，他可以记录下这个例外情况。当发送报文命中这条路由时，如果目的地址不是C，那么按1500进行分片，如果是C，则按1300进行分片。

## 小结
即配置的FIB是一个大的网段，到达某个具体的目的的报文被中间设备发送了  `ICMP FRAGNEEDED` 或者 `ICMP REDIRECT`给发送端。
以`ICMP FRAGNEEDED`为例，那么发送端的路由表中就会生成一个到达该**指定目的IP(/32网段，不是中间设备的IP)的`Nexthop exception`路由，其MTU为中间设备发送的MTU**。
这样，就存在了2条FIB，一个是粗粒度的 FIB网段的路由，一个是细粒度(/32网段)的`Nexthop exception` 路由。

# 实现
## 数据结构
内核中使用`fib_nh_exception`表示这种例外表项：
```c
(include/net/ip_fib.h)

struct fib_nh_exception {
	struct fib_nh_exception __rcu	*fnhe_next;  /*  冲突链上的下个fib_nh_exception结构 */
	int             fnhe_genid;
	__be32				fnhe_daddr;              /*  例外的目标地址                     （即：exception ip）*/
	u32				  fnhe_pmtu;                 /*  收到的ICMP FRAGNEEDED通告的PMTU    */
	__be32				fnhe_gw;                 /*  收到的ICMP REDIRECT通告的网关      */         
	unsigned long			fnhe_expires;        /*  该例外表项的过期时间                */
	struct rtable __rcu     *fnhe_rth_input; /*  关联的路由缓存                    (route cache) */
    struct rtable __rcu     *fnhe_rth_output; /*  关联的路由缓存                    (route cache) */
	unsigned long			fnhe_stamp; /*  exception的创建时刻 */
};
```

每一个下一跳结构`fib_nh`上有一个指针指向`fnhe_hash_bucket`哈希桶的指针:

```c
struct fib_nh {
    struct net_device   *nh_dev;
    struct hlist_node   nh_hash;
    struct fib_info     *nh_parent;
    unsigned int        nh_flags;
    unsigned char       nh_scope;
#ifdef CONFIG_IP_ROUTE_MULTIPATH
    int         nh_weight;
    int         nh_power;
#endif
#ifdef CONFIG_IP_ROUTE_CLASSID
    __u32           nh_tclassid;
#endif
    int         nh_oif;
    __be32          nh_gw;
    __be32          nh_saddr;
    int         nh_saddr_genid;
    struct rtable __rcu * __percpu *nh_pcpu_rth_output; /* output route cache: __percpu的，每个CPU中该成员不一样么？即使整个 struct fib_nh 是共享的??*/
    struct rtable __rcu *nh_rth_input; /* input route cache */
    struct fnhe_hash_bucket *nh_exceptions; /* exception 的 hash表 */
};

struct fnhe_hash_bucket {
    struct fib_nh_exception __rcu   *chain;
};

```

哈希桶在`update_or_create_fnhe`中创建，每个哈希桶包含2048条冲突链，每条冲突链可以存5个`fib_nh_exception`。

## 收到ICMP FRAGNEEDED的处理
以`PMTU`为例，在收到网络设备返回的`ICMP FRAGNEEDED`报文后，会调用下列函数==将通告的`pmtu`值记录到`fib_nh_exception`上，也会记录到`nexthop exception`绑定的路由缓存`rtable`上==。

```
static void __ip_rt_update_pmtu(struct rtable *rt, struct flowi4 *fl4, u32 mtu)
{
    
	if (fib_lookup(dev_net(dst->dev), fl4, &res) == 0) {
		/* fib 查找到下一跳 */
		struct fib_nh *nh = &FIB_RES_NH(res);

		update_or_create_fnhe(nh, fl4->daddr, 0, mtu,
				      jiffies + ip_rt_mtu_expires);
	}
}

static void fill_route_from_fnhe(struct rtable *rt, struct fib_nh_exception *fnhe)
{
    /* 填充route cache */
    rt->rt_pmtu = fnhe->fnhe_pmtu;
    rt->rt_mtu_locked = fnhe->fnhe_mtu_locked;
    rt->dst.expires = fnhe->fnhe_expires;

    if (fnhe->fnhe_gw) {
        rt->rt_flags |= RTCF_REDIRECTED;
        rt->rt_gateway = fnhe->fnhe_gw;
        rt->rt_uses_gateway = 1;
    }
}


static void update_or_create_fnhe(struct fib_nh *nh, __be32 daddr, __be32 gw,
                  u32 pmtu, unsigned long expires)
{
    struct fnhe_hash_bucket *hash;
    struct fib_nh_exception *fnhe;
    struct rtable *rt;
    unsigned int i;
    int depth;
    u32 hval = fnhe_hashfun(daddr);

    spin_lock_bh(&fnhe_lock);

    hash = nh->nh_exceptions; 
    /* 下一跳 nexthop 上 exceoption fib 的hash表*/
    if (!hash) {
        hash = kzalloc(FNHE_HASH_SIZE * sizeof(*hash), GFP_ATOMIC);
        if (!hash)
            goto out_unlock;
        nh->nh_exceptions = hash;
    }

    hash += hval;

    depth = 0;
    for (fnhe = rcu_dereference(hash->chain); fnhe;
         fnhe = rcu_dereference(fnhe->fnhe_next)) {
         /* 查找 nexthop 的 exception 节点 */
        if (fnhe->fnhe_daddr == daddr)
            break;
        depth++;
    }

    if (fnhe) {
        /* 在hash表中 查找到 nexthop 的 exception 节点 */
        if (gw)
            fnhe->fnhe_gw = gw;
        if (pmtu) {
            fnhe->fnhe_pmtu = pmtu;
            fnhe->fnhe_expires = max(1UL, expires);
        }
        /* Update all cached dsts too */
        /* 更新 exception 节点对应的 route cache */
        rt = rcu_dereference(fnhe->fnhe_rth_input);
        if (rt)
            fill_route_from_fnhe(rt, fnhe);
        rt = rcu_dereference(fnhe->fnhe_rth_output);
        if (rt)
            fill_route_from_fnhe(rt, fnhe);
    } else {
        if (depth > FNHE_RECLAIM_DEPTH)
            fnhe = fnhe_oldest(hash);
        else {
            /* 申请 exception 节点 */
            fnhe = kzalloc(sizeof(*fnhe), GFP_ATOMIC);
            if (!fnhe)
                goto out_unlock;

            fnhe->fnhe_next = hash->chain;
            rcu_assign_pointer(hash->chain, fnhe);
        }
        fnhe->fnhe_genid = fnhe_genid(dev_net(nh->nh_dev));
        fnhe->fnhe_daddr = daddr;
        fnhe->fnhe_gw = gw;
        fnhe->fnhe_pmtu = pmtu;
        fnhe->fnhe_expires = expires;

        /* Exception created; mark the cached routes for the nexthop
         * stale, so anyone caching it rechecks if this exception
         * applies to them.
           创建了 nexthop 的 exception之后，标识一下 nexthop上 的 route cache 过时了，
           使得新建的 nexthop 的 exception 生效；
           标记 nexthop上 的 route cache 过时了之后，
           使用该 nexthop时，不应该使用它的 route cache了，应该重新查询下最新的 exception；
         */
        rt = rcu_dereference(nh->nh_rth_input);
        /*
            #define DST_OBSOLETE_NONE   0 // 未知含义
            #define DST_OBSOLETE_DEAD   2 // 已经被处理过，等待引用计数=0之后被清理
            #define DST_OBSOLETE_FORCE_CHK  -1  // 默认值
            #define DST_OBSOLETE_KILL   -2 // 该路由上有exception，标记为等待过期时间到达之后再处理
        */
        if (rt)
            rt->dst.obsolete = DST_OBSOLETE_KILL; /* OBSOLETE: 过期的，废弃的*/

        for_each_possible_cpu(i) {
            struct rtable __rcu **prt;
            prt = per_cpu_ptr(nh->nh_pcpu_rth_output, i);
            rt = rcu_dereference(*prt);
            if (rt)
                rt->dst.obsolete = DST_OBSOLETE_KILL;
        }
    }

    fnhe->fnhe_stamp = jiffies;

out_unlock:
    spin_unlock_bh(&fnhe_lock);
    return;
}


```

## 再次发包处理
==在发包流程查询FIB之后，会首先看是否存在以目标地址为KEY的例外表项(`fib_nh_exception`)==；

如果有，就使用其绑定的路由缓存（即：`struct fib_nh_exception`  中的 `fnhe_rth_input` 或 `fnhe_rth_output` ）；
如果没有就使用下一跳上的缓存（即：`struct fib_nh`结构中的`nh_rth_input` 或 `nh_pcpu_rth_output`）。


**(1) 查找exception**:
基于`dip`，在`nexthop`的`exception hash`表中查找` exception`（即：`struct fib_nh` 结构中的 `nh_exceptions hash`表）;
如果查找到 exception ，再考虑是否使用 `exception` 中的 `route cache`（即：`struct fib_nh_exception`  中的 `fnhe_rth_input` 或 `fnhe_rth_output` ）



```
static struct rtable *__mkroute_output(const struct fib_result *res,
				       const struct flowi4 *fl4, int orig_oif,
				       struct net_device *dev_out,
				       unsigned int flags)
{
    /* code omitted */
    struct fib_nh_exception *fnhe;

    if (fi) {
		struct rtable __rcu prth;
		struct fib_nh *nh = &FIB_RES_NH(*res);

		/* 基于 fl4->daddr 查找是否存在 fib_nh_exception */
		fnhe = find_exception(nh, fl4->daddr);
		if (fnhe)
			/*如果有，直接使用 excepton上 绑定的路由缓存 */
			prth = &fnhe->fnhe_rth_output; 
		else {
			if (unlikely(fl4->flowi4_flags &
				     FLOWI_FLAG_KNOWN_NH &&
				     !(nh->nh_gw &&
				       nh->nh_scope == RT_SCOPE_LINK))) {
				do_cache = false;
				goto add;
			}
			/*  如果没有，使用下一跳上缓存的路由缓存 */
			prth = __this_cpu_ptr(nh->nh_pcpu_rth_output); 
		}
		rth = rcu_dereference(*prth);
		if (rt_cache_valid(rth)) {
			dst_hold(&rth->dst);
			return rth;
		}
	}
}
```

## exception 的超时

如下所示，exception 看着没有超时机制，而是通过RCU来保证最多多少个，这样数量也不会无限增长。看样子exception 的hash表的每个链上最多是5个表项。

```c
#define FNHE_RECLAIM_DEPTH  5

static void fnhe_flush_routes(struct fib_nh_exception *fnhe)
{
  struct rtable *rt;

  rt = rcu_dereference(fnhe->fnhe_rth_input);
  if (rt) {
    RCU_INIT_POINTER(fnhe->fnhe_rth_input, NULL);
    dst_dev_put(&rt->dst);
    dst_release(&rt->dst);
  }
  rt = rcu_dereference(fnhe->fnhe_rth_output);
  if (rt) {
    RCU_INIT_POINTER(fnhe->fnhe_rth_output, NULL);
    dst_dev_put(&rt->dst);
    dst_release(&rt->dst);
  }
}

static struct fib_nh_exception *fnhe_oldest(struct fnhe_hash_bucket *hash)
{
  struct fib_nh_exception *fnhe, *oldest;

  oldest = rcu_dereference(hash->chain);
  for (fnhe = rcu_dereference(oldest->fnhe_next); fnhe;
       fnhe = rcu_dereference(fnhe->fnhe_next)) {
    if (time_before(fnhe->fnhe_stamp, oldest->fnhe_stamp))
      oldest = fnhe;
  }
  fnhe_flush_routes(oldest);
  return oldest;
}

static void update_or_create_fnhe(struct fib_nh *nh, __be32 daddr, __be32 gw,
          u32 pmtu, bool lock, unsigned long expires)
{
  struct fnhe_hash_bucket *hash;
  struct fib_nh_exception *fnhe;
  struct rtable *rt;
  u32 genid, hval;
  unsigned int i;
  int depth;

  genid = fnhe_genid(dev_net(nh->nh_dev));
  hval = fnhe_hashfun(daddr);

  spin_lock_bh(&fnhe_lock);

  hash = rcu_dereference(nh->nh_exceptions);
  if (!hash) {
    hash = kcalloc(FNHE_HASH_SIZE, sizeof(*hash), GFP_ATOMIC);
    if (!hash)
      goto out_unlock;
    rcu_assign_pointer(nh->nh_exceptions, hash);
  }

  hash += hval;

  depth = 0;
  for (fnhe = rcu_dereference(hash->chain); fnhe;
       fnhe = rcu_dereference(fnhe->fnhe_next)) {
    if (fnhe->fnhe_daddr == daddr)
      break;
    depth++;
  }

  if (fnhe) {
    if (fnhe->fnhe_genid != genid)
      fnhe->fnhe_genid = genid;
    if (gw)
      fnhe->fnhe_gw = gw;
    if (pmtu) {
      fnhe->fnhe_pmtu = pmtu;
      fnhe->fnhe_mtu_locked = lock;
    }
    fnhe->fnhe_expires = max(1UL, expires);
    /* Update all cached dsts too */
    rt = rcu_dereference(fnhe->fnhe_rth_input);
    if (rt)
      fill_route_from_fnhe(rt, fnhe);
    rt = rcu_dereference(fnhe->fnhe_rth_output);
    if (rt)
      fill_route_from_fnhe(rt, fnhe);
  } else {
    if (depth > FNHE_RECLAIM_DEPTH)
      fnhe = fnhe_oldest(hash);
    else {
      fnhe = kzalloc(sizeof(*fnhe), GFP_ATOMIC);
      if (!fnhe)
        goto out_unlock;

      fnhe->fnhe_next = hash->chain;
      rcu_assign_pointer(hash->chain, fnhe);
    }
    fnhe->fnhe_genid = genid;
    fnhe->fnhe_daddr = daddr;
    fnhe->fnhe_gw = gw;
    fnhe->fnhe_pmtu = pmtu;
    fnhe->fnhe_mtu_locked = lock;
    fnhe->fnhe_expires = max(1UL, expires);

    /* Exception created; mark the cached routes for the nexthop
     * stale, so anyone caching it rechecks if this exception
     * applies to them.
     */
    rt = rcu_dereference(nh->nh_rth_input);
    if (rt)
      rt->dst.obsolete = DST_OBSOLETE_KILL;

    for_each_possible_cpu(i) {
      struct rtable __rcu **prt;
      prt = per_cpu_ptr(nh->nh_pcpu_rth_output, i);
      rt = rcu_dereference(*prt);
      if (rt)
        rt->dst.obsolete = DST_OBSOLETE_KILL;
    }
  }

  fnhe->fnhe_stamp = jiffies;

out_unlock:
  spin_unlock_bh(&fnhe_lock);
}


```

## 查看 exception 表项

```bash
ip route show table cache
或
ip route get x.x.x.x
```


# 小结
（1）下一跳以及下一跳的exception 上都存在路由缓存。

（2）查找顺序
对于每个要发送的数据包，都需要查询fib。
查找到`fib`之后，得到下一跳。
如果下一跳上存在`exception hash`表，还有基于`dstip`查询`excecption`, 查找到，则使用 `excecption`的`route cache`。
如果在下一跳的 基于`dstip`查询不到`excecption`，则使用下一跳自己的 `route cache`。

（3）`ip route show table cache` 查看
`ip route show table cache` 查看的是`route cache`, 应该是 `nexthop 的 exception`的 `route cache`。
而 `nexthop exception` 就是由于收到了 `icmp needfrag` 或者是 `icmp redirect`报文才会生成的。



# 参考
```c
# FIB nexthop Exception是什么
https://switch-router.gitee.io/blog/fib_nh_exception/
```