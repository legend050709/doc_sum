```table-of-contents
```

# rte_flow的应用场景
## mark的使用
rte_flow规则中设置mark 进行下发， 那么在 rte_mbuf 匹配了这个规则，rte_mbuf的 meta data中会携带 mark，就可以基于 mark 得到有用的信息。
比如 在用户态tcp协议栈libtpa项目中，使用mark 来携带这个 mbuf 匹配的 listen socket id, worker id 等信息，便于后续的查找。
具体：可以参考 libtpa 中使用 Mellanox网卡，下发的 rte_flow 带有 mark（通过worker id 以及 tcp socket id 封装而成，u32类型）的FDIR规则 ，rte_mbuf 匹配到 flow 规则之后，rte_mbuf 中的 hash.fdir.hi  即为 mark。

```bash
参见：make_flow_mark、parse_flow_mark
```

```c
struct rte_mbuf {
	RTE_MARKER cacheline0;
	...
	uint64_t ol_flags; /**< Offload features. */
	...
    RTE_STD_C11
    union {
        union {
            uint32_t rss;     /**< RSS hash result if RSS enabled */
            struct {
                union {
                    struct {
                        uint16_t hash;
                        uint16_t id;
                    };
                    uint32_t lo;
                    /**< Second 4 flexible bytes */
                };
                uint32_t hi;
                /**< First 4 flexible bytes or FD ID, dependent
                 * on RTE_MBUF_F_RX_FDIR_* flag in ol_flags.
                 */
            } fdir; /**< Filter identifier if FDIR enabled */
            struct rte_mbuf_sched sched;
            /**< Hierarchical scheduler : 8 bytes */
            struct {
                uint32_t reserved1;
                uint16_t reserved2;
                uint16_t txq;
                /**< The event eth Tx adapter uses this field
                 * to store Tx queue id.
                 * @see rte_event_eth_tx_adapter_txq_set()
                 */
            } txadapter; /**< Eventdev ethdev Tx adapter */
            uint32_t usr;
            /**< User defined tags. See rte_distributor_process() */
        } hash;                   /**< hash information */
    };
	...
};
```


## dpdk多线程同一个流的正反向包导流到同一个核
**背景**
为了保证同一条流的正方向包 都交给 LB的同一个 worker 线程进行处理。

拿 LB 进行分析：

![](attachments/Pasted%20image%2020240914173345.png)


### 基于dip的rte_flow 导流

![](attachments/Pasted%20image%2020240914173122.png)

如上所示，美团的 LB 使用的是 RTC 模式，每个worker 分配不同的 LIP，下发的 rte_flow 规则，基于 dip=lip ，action为 lip 所在的 worker 对应的队列。

### 基于dip:dport_index 的rte_flow导流

在 dpvs 项目中，是基于 dip:dport_index 来设置 rte_flow，具体就是 LB的  LIP:Lport_index
来保证同一条流的正反向的报文交给同一个core来处理，因为session是per-core的。

**配置**：
(1)每个core的配置：
保证每个core的 LIP:Lport对不一样，那么如果LIP共享，就需要每个core的Lport不一样，不重叠。
比如 存在4个转发core（4是2的n次方）：
```text

将 port & 3 == 0的port分配给core0，port & 3 == 1的port分配给core1，
port & 3 == 2的port分配给core2，port & 3 == 3的port分配给core3;

注：3 为 port 的mask，是 转发核个数-1，最终为2的n次方-1；

core0:  1024, 1028,1032, ...
core1:  1025, 1029,1033,...
core2:  1026, 1030,1034,...
core3:  1027, 1031,1035,...
```

(2) 网卡的RSS配置：
网卡设置RSS配置，比如tcp、udp协议的包，基于四元组hash，然后
mod 队列的个数，得到入向网卡队列的 index；

(3) 网卡的FDIR配置：
将dip为lip，dport & 3 的 规则，设置 action为 对应LIp:Lport 所在的 线程的 index（该线程的index == 该线程所在的core 绑定的网卡的接收队列的 index ）。

**正向流匹配**：
正向流量，dip为vip，dport 为vport，无法匹配到 FDIR规则（默认情况下，FDIR规则优先级高于RSS规则）；如果是TCP、UDP协议就可以匹配到 RSS规则，就基于hash得到一个入向的网卡的队列。

在转发core从网卡队列中取包之后，建立或者匹配session，然后FULLNAT转发，发出去时
`sip:sport==lip:lport, dip:dport==rip:rport`；

**反向流匹配**：
RS的回包，`dip:dport==lip:lport`, 此时可以匹配到 FDIR规则，将流量导流到 对应的队列，保证了正反向同一个core中处理。



### 基于dip:dport_range的rte_flow导流

**下发的规则**：
dip为本端的ip，dport为 本worker分配的 port段的起始port，port的掩码(port_mask)为段长度(64)取反后所对应的掩码。动作为：指定队列的id。
每个worker线程分配属于自身的 port段，worker之间的port段互不重叠， 且和 内核的 ip-port-range 也不重叠。
每个worker在每个网卡上对应一个接收队列。


**使用场景**：
libtpa作为client，连接server，需要分配local-ip、local-port；为了保证同一个流的正反向都交给同一个worker来处理。
在当前worker中选择属于当前worker自身的local port之后，需要保证回包还交给这个workrer来处理，这样才可以在这个worker中查找到 tcp socket。



**规则匹配**： 
回包的`dip:dport == lip:lport`. 如果每个lip:lport 都下发一个FDIR规则，那么存在大量的FDIR规则。
回包的 dip为lip，dport & port_mask 就可以得到 port段的start port，然后基于 lip: start_port 就可以匹配到这样的规则, 得到网卡的队列。


```
参见：libtpa 中的 do_port_block_alloc

struct port_block {
    __le16 start;   // block-port 起始port; 一般是 64的整数倍;
    __le16 end;     // block-port 结束port;
    uint16_t size;    // block-port 大小，一般是 64;
    uint16_t mask;  // size-1; 一般是 63;
    int refcnt;

    __le16 port_mask; // 0xffc0, 即 1111 1111 1100 0000（低n位为0，其他的16-n位为1,2的n次方为64，n位6）;
    struct tpa_worker *worker; // 所在的 worker;

    struct offload_list offload_list; 
        /* 链表：每个成员为struct offload 类型;
            保存所有的 block port 的 offload 规则，可以作为展示和删除; bonding 模式，每个口下发了一个规则，2个rule链接为 链表 */
    struct timer timer;
};

struct offload {
    int port; // offload 规则对应的 port;
    struct rte_flow *flow; // offload 规则 的  rte_flow;

    TAILQ_ENTRY(offload) node;
};
```

# rte_flow的限制

## fdir 的限制

因为intel上的卡有个缺陷，下发了local ip的filter（ip + port粒度）以后，input set就固定了，ip粒度就不生效，为了向下兼容，就多加规则，把kni ip 的端口范围`[0-65535]`覆盖到，起到了kn ip粒度导流的作用。

![](attachments/Pasted%20image%2020240913152249.png)

注：ethtool 下发可能也有问题，即下发了一个ethtool 的 FDIR 规则，rule 的match flow type 是tcp，就不允许在再发一个FDIR规则 flow type 是 IP4了。



# 参考
```bash
# DPDK开发者手册流分类rte_flow
https://blog.csdn.net/legend050709/article/details/123569011
```