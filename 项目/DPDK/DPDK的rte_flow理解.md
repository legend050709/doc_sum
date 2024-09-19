```table-of-contents
```

# rte_flow的应用场景
## mark的使用
rte_flow规则中设置mark 进行下发， 那么在 rte_mbuf 匹配了这个规则，rte_mbuf的 meta data中会携带 mark，就可以基于 mark 得到有用的信息, 比如 这个 mbuf 匹配的 listen socket id, worker id 等信息，便于后续的查找。
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
### 基于dip:dport_range的rte_flow导流

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