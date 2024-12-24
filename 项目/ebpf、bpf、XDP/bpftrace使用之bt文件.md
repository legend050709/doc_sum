```table-of-contents
```

# 范例
## 查看socket层的sendmsg 和 dev层的 dev_hard_start_xmit 函数
作用：在应用层调用`tcp`的`send`函数，以及dev层的发包之间设置`trace`，查看 是否都有对应的 `trace`，进而确定是否在协议栈的`tcp`层或者`ip`层出现问题。
比如：如果 `tcp_sendmsg` 之后的数据没有进入到 `dev`层，说明问题出现在 `tcp`层或者`ip`层。

注：并且通过下面的`delay`的输出，可以看出来相邻两次调用`sendmsg`的`delay`。

```bt
#!/usr/bin/bpftrace

#include <linux/socket.h>
#include <linux/tcp.h>
#include <net/sock.h>

// 不能指定pid，因为有部分调用是在时间中断中执行的，
// 这部分pid 是不确定的

BEGIN {
 // $ptr= (struct sock *) 0;
  @addr[0] = ntop(0);
  @pids[0] = 0;
  @time[0] = 0;
  @port[0] = 0;
}


kprobe:__tcp_transmit_skb {
  $sk = (struct sock *) arg0;
  $tp = (struct tcp_sock *) arg0;
  $inet_family = $sk->__sk_common.skc_family;

  if ($inet_family == AF_INET || $inet_family == AF_INET6) {
    $now = nsecs;

    if (@time[arg0]) {
      $gap = $now - @time[arg0];
      if ( $gap > 100000000 ) {
        printf("%s.%-9d sendmsg pid:%-8d dst:%14s:%-6d delay:%-10d\n",
                strftime("%H:%M:%S", $now), $now % 1000000000,
                @pids[arg0], @addr[arg0], @port[arg0], $gap);
      }
    }

    $daddr = ntop(0);
    if ($inet_family == AF_INET) {
      $daddr = ntop($sk->__sk_common.skc_daddr);
    } else {
      $daddr = ntop($sk->__sk_common.skc_v6_daddr.in6_u.u6_addr8);
    }
    $dport = $sk->__sk_common.skc_dport;
    $dport = ($dport >> 8) | (($dport << 8) & 0x00FF00);

    @addr[arg0]=$daddr;
    @port[arg0]=$dport;
    @pids[arg0]=pid;
    @time[arg0]=nsecs;
  }
}

kprobe:dev_hard_start_xmit {
  $skb = (struct sk_buff*)arg0;
  $sk = (uint64) ($skb->sk);

  if (@time[$sk]) {
    delete(@addr[$sk]);
    delete(@time[$sk]);
    delete(@pids[$sk]);
    delete(@port[$sk]);
  }
}

END {
   clear(@addr);
   clear(@pids);
   clear(@time);
   clear(@port);
}
```

## 查看tcp层的 tcp_sync_mss 函数
```c
# cat see_tcp_mss_sync.bt
#!/bin/bpftrace

#include <linux/socket.h>
#include <linux/tcp.h>
#include <net/sock.h>
#include <net/route.h>

/*
//kprobe:tcp_sync_mss  / pid == $1 / {
kprobe:tcp_sync_mss  {
  @["call", arg1] = count();
}

kretprobe:tcp_sync_mss  {
  @["ret", retval] = count();
}
*/

kprobe:tcp_sync_mss  / pid == $1 / {
  $sk = (struct sock *) arg0;
  $tp = (struct tcp_sock *) arg0;
  $inet_family = $sk->__sk_common.skc_family;
  $dst = (struct dst_entry *) $sk->sk_dst_cache;

  if ($inet_family == AF_INET || $inet_family == AF_INET6) {
    $now = nsecs;

    $daddr = ntop(0);
    $saddr = ntop(0);
    if ($inet_family == AF_INET) {
      $daddr = ntop($sk->__sk_common.skc_daddr);
      $saddr = ntop($sk->__sk_common.skc_rcv_saddr);
    } else {
      $daddr = ntop($sk->__sk_common.skc_v6_daddr.in6_u.u6_addr8);
      $saddr = ntop($sk->__sk_common.skc_v6_rcv_saddr.in6_u.u6_addr8)
    }
    $dport = $sk->__sk_common.skc_dport;
    $dport = ($dport >> 8) | (($dport << 8) & 0x00FF00);

    $dev = $dst->dev->name;
    $devmtu = $dst->dev->mtu;
    $rt_pmtu = ((struct rtable*)$dst)->rt_pmtu;

    printf("%s.%09d %14s %14s:%-6d pmtu:%d dev:%s devmtu:%d %lx %lx\n",
           strftime("%H:%M:%S", $now), $now % 1000000000,
           $saddr, $daddr, $dport, arg1, $dev, $devmtu, $sk, $sk->sk_dst_cache);
  }
}

kprobe:tcp_retransmit_timer {
  $sk = (struct sock *) arg0;
  $tp = (struct tcp_sock *) arg0;
  $inet_family = $sk->__sk_common.skc_family;
  $dst = (struct dst_entry *) $sk->sk_dst_cache;

  if ($inet_family == AF_INET || $inet_family == AF_INET6) {
    $now = nsecs;

    $daddr = ntop(0);
    $saddr = ntop(0);
    if ($inet_family == AF_INET) {
      $daddr = ntop($sk->__sk_common.skc_daddr);
      $saddr = ntop($sk->__sk_common.skc_rcv_saddr);
    } else {
      $daddr = ntop($sk->__sk_common.skc_v6_daddr.in6_u.u6_addr8);
      $saddr = ntop($sk->__sk_common.skc_v6_rcv_saddr.in6_u.u6_addr8)
    }
    $dport = $sk->__sk_common.skc_dport;
    $dport = ($dport >> 8) | (($dport << 8) & 0x00FF00);

    $dev = $dst->dev->name;
    $devmtu = $dst->dev->mtu;
    $rt_pmtu = ((struct rtable*)$dst)->rt_pmtu;

    printf("re %s.%09d %14s %14s:%-6d pmtu:%d dev:%s devmtu:%d %lx %lx\n",
           strftime("%H:%M:%S", $now), $now % 1000000000,
           $saddr, $daddr, $dport, arg1, $dev, $devmtu, $sk, $sk->sk_dst_cache);
  }
}
```


![](attachments/Pasted%20image%2020241216103314.png)





# 参考
```bash
```