# 背景
编写网络应用程序时，我们一般都是在网络状况良好的局域网甚至是本机内进行测试调试。
有没有办法在网络状况良好的内网环境中，在不改动程序自身代码的前提下，为应用程序模拟复杂的外网环境——尤其是网络延迟呢？

# 介绍

netem(Network Emulator)可以用来对==网卡发出==的数据包进行增加延迟、丢包、重复、乱序等处理，来模拟复杂网络环境。用来在性能良好的局域网中，模拟出复杂的互联网传输性能。
netem的设置依赖tc命令，tc是Linux内核提供的流量控制工具。关于更多功能以及参数的详细解释可以参阅 `tc-netem` 的 man page。

包从进入tc开始，分为多个qd(qdisc)。每个qd可以包含多个子qd，qd彼此连接形成一颗树。每个qd上可以附加filter，选择进入哪个child。如果都没命中，那就看本身规则。

**注意：tc控制的是发包动作，不能控制收包动作。它直接对物理接口生效，如果控制了物理的eth0，那么逻辑网卡（比如eth0:1）也会受到影响，反之则不行，控制逻辑网卡是无效的。**

> 注：此中的tc netem 延迟丢包是在软件层面的机制，并不是硬件层面，具体应该是在linux协议栈中。在本机上设置延迟100ms，通过tcpdump抓包看时间戳其实看不到这个延迟，因为tcpdump抓包在驱动和协议栈之间。

# tc内核流程
## tc 和 netfilter 对比
Netfilter 作用在**内核网络协议栈**上，通过在各个枢纽设立关卡，对网络包(`sk_buff` 数据结构, skb)进行检查，并实施 ACCEPT、DROP、MASQUERADE 等策略。
相比之下，TC 是绑定在**网络设备**上实施的。提供 `enqueue`, `dequeue` 两个核心函数，也是作为关卡对到达网络设备的网络包实施策略。
要说核心的不同之处，Netfilter 是流式地处理网络包，先到的网络包一定先出（也可能是被丢弃）；TC 的处理方式就依照策略，类比块设备的随机读/随机写了。


## 网络发包流程
网络包的发送由应用层通过系统调用 `sendmsg` (类似的系统调用还有 `send`, `sendto` 等) 发起，后陷入内核态，由内核代码实现网络协议栈逐层封装，逐层下发的工作。

```plain
 0)               |              sock_sendmsg() {       /* 系统调用 sendmsg 对应的内核函数 */
 0)               |                inet_sendmsg() {     /* 使用 IPv4 (not IPv6) 实现的 sendmsg */
 0)               |                  tcp_sendmsg() {    /* 传输层，TCP 协议的处理 */
 0)               |                      sk_stream_alloc_skb() {    /* 创建 sk_buff(skb) 数据结构，网络包在内核中存在的形式 */
 0)               |                        __alloc_skb() {
 0)   5.273 us    |                        }
 0)   0.056 us    |                        sk_forced_mem_schedule();
 0)   6.026 us    |                      }
 0)               |                      tcp_push() {
 0)               |                        __tcp_push_pending_frames() {
 0)               |                          tcp_write_xmit() {
 0)   0.132 us    |                            tcp_tso_segs();
 0)   0.058 us    |                            tcp_init_tso_segs();
 0)               |                            tcp_transmit_skb() { /* 下发 skb ，交由网络层继续 */
 0)   0.046 us    |                              skb_push();
 0)               |                              ip_queue_xmit() {  /* 网络层，IP 协议的处理 */
 0)               |                                __sk_dst_check() {
 0)   0.069 us    |                                  ipv4_dst_check();
 0)   0.429 us    |                                }
 0)   0.049 us    |                                skb_push();
 0)               |                                  ip_output() {
 0)               |                                    ip_finish_output() {
 0)               |                                      ip_finish_output2() {
 0)   0.044 us    |                                        skb_push();
 0)               |                                        dev_queue_xmit() {   /* 数据链路层，由网卡设备（内核根据物理设备抽象出来的概念）继续处理 */
 0)               |                                          __dev_queue_xmit() {
 0)   0.151 us    |                                            netdev_pick_tx();
 0)   0.046 us    |                                            _raw_spin_lock();
 0)               |                                            /* some important things omitted */
 0) + 39.674 us   |                                          }
 0) + 40.053 us   |                                        }
 0) + 41.656 us   |                                      }
 0) + 43.041 us   |                                    }
 0) + 47.121 us   |                                  }
 0) + 68.614 us   |                                }
 0) + 70.558 us   |                              }
 0) + 75.995 us   |                            }
 0) + 85.718 us   |                          }
 0) + 86.418 us   |                        }
 0) + 87.275 us   |                      }
 0) ! 101.636 us  |                    }
 0) ! 105.024 us  |                  }
 0) ! 105.516 us  |                }
 0) ! 107.861 us  |              }
 0) ! 108.230 us  |            }
 0) ! 108.653 us  |          }
 0) ! 109.066 us  |        }
 0) ! 112.023 us  |      }
 0) ! 113.078 us  |    }
 0) ! 113.374 us  |  }
```

待到网络包(skb)到达数据链路层，`dev_queue_xmit` 标志着 skb 终于交付给网络设备，下面就是 TC 策略的运转了。
Qdisc (queueing discipline) ，是整个 TC 的基本模型。所有需要通过网卡接口发送的数据包，都会进入接口绑定的 Qdisc 等待队列(enqueue)。
再由 ksoftirq 内核线程读取接口 Qdisc 中的数据包(dequeue)，尽最大能力发送出去。

# 安装
查看 
```c
tc qdisc show
```

安装
```c
yum install kernel-modules-extra
```
# 模拟出向包的网络质量
在下面的例子中 `add` 表示为网卡添加 netem 配置，`change` 表示修改已经存在的 netem 配置到新的值，如果要删除网卡上的配置可以使用 `del`：
```
# tc qdisc del dev eth0 root
```
## 模拟时延
- 对eth0网卡发出的数据包延迟500毫秒发送
```c
$ tc qdisc add dev eth0 root netem delay 500ms
```
这里的qdisc是tc中的一个基本概念，dev eth0表示操作的网卡，
root表示出队列的根（egress），与之相对的是ingress(入队列)。
netem表示启用netem，delay 500ms表示延迟500毫秒。其他命令类似。

## 模拟概率丢包
- 对eth0网卡发出的数据包随机丢包5%
```
$ tc qdisc add dev eth0 root netem loss 5%
```
## 模拟重复数据包

# 模拟入向包的网络质量


# 其他
## tcconfig
tc提供的非常强大的功能，同时也非常难以理解使用。通过tcconfig可以方便地配置netem的功能。 tcconfig可以通过pip或pip3安装，详细参见：[官方文档](https://tcconfig.readthedocs.io/en/latest/)

如对发往1.1.1.1的数据包增加延迟500ms，只需要一个命令即可。tcconfig会自动根据需要调用tc命令进行配置。
```
tcset eth0 --delay 500ms --dst-address 1.1.1.1 --direction outgoing 
```

对1.1.1.1的80端口的出入数据包增加随机丢包5%。

```
tcset eth0 --loss 5% --dst-address 1.1.1.1 --dst-port 80 --direction outgoing

tcset eth0 --loss 5% --src-address 1.1.1.1 --src-port 80 --direction incoming
```


通过tcshow命令可以查看现有的规则
```
tcshow eth0
```

通过tcdel命令删除现有规则
```
tcdel eth0 --all
```

# 参考
```c
发包以及收包的模拟
https://www.qlee.in/%E6%97%A5%E5%B8%B8%E8%AE%B0%E5%BD%95/2019/12/29/linux-tc-netem/

netem介绍以及测试案例
https://cizixs.com/2017/10/23/tc-netem-for-terrible-network/

https://developer.aliyun.com/article/553750

https://www.haxi.cc/archives/Linux%E6%A8%A1%E6%8B%9F%E5%A4%8D%E6%9D%82%E7%BD%91%E7%BB%9C%E7%8E%AF%E5%A2%83%E4%B8%8B%E7%9A%84%E4%BC%A0%E8%BE%93-netem%E5%92%8Ctc.html

tc的原理：
https://www.ffutop.com/posts/2019-08-23-traffic-control/

```