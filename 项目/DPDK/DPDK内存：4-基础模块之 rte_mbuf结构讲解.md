```table-of-contents
```
# 介绍
DPDK网络功能中使用的rte_mbuf作用类似于内核态网络中的sk_buff，它是对接网络驱动和协议栈的接口。

# 设计思想
mbuf主要由元信息和数据两部分组成。

对于数据报文的存储（包括协议头），可以考虑如下两种方式：
 - 1、将网络帧元数据（metadata）和帧本身存放在固定大小的同一段缓存中；
	  用单独的内存缓存结构哦存储元数据，后面跟着固定大小的内存区域去保存报文数据；
 - 2、将元数据和网络帧分开存放在两段缓存里；
 
 两种方式的优缺点：
 - 第一种 ：对缓存的申请及释放均只需一条指令，缺点是因为缓存长度固定而网络帧大小不一，大部分帧只能使用填0（padding）的方式填满整个缓存，较为耗费内存空间。
 - 第二种 ：帧数据的大小可以任意，同时对元数据和网络帧的缓存可以分开申请及释放；缺点是低效，因为无法保证数据存在一个Cache Line中，可以造成HIT Miss；

为了保持包处理的效率，DPDK采用第一种方式。
网络帧元数据的一部分内容由DPDK网卡驱动写入。这些内容包括VLAN标签、RSS哈希值、网络入口端口号以及巨型帧所占的Mbuf个数等。

# rte_mbuf 详解
## pktmbuf pool
Mbuf由缓冲池rte_mempool（元素大小固定的内存池rte_mempool）管理，rte_mempool在初始化时一次申请多个mbuf，申请的mbuf个数和长度都由用户指定。
申请pktmbuf pool对外开放的API如下：
```c
struct rte_mempool * rte_pktmbuf_pool_create_by_ops(const char *name, unsigned int n,
    unsigned int cache_size, uint16_t priv_size, uint16_t data_room_size,
    int socket_id, const char *ops_name);

struct rte_mempool * rte_pktmbuf_pool_create(const char *name, unsigned int n,
    unsigned int cache_size, uint16_t priv_size, uint16_t data_room_size,
    int socket_id);

struct rte_mempool * rte_pktmbuf_pool_create_extbuf(const char *name, unsigned int n,
  unsigned int cache_size, uint16_t priv_size,
  uint16_t data_room_size, int socket_id,
  const struct rte_pktmbuf_extmem *ext_mem,
  unsigned int ext_num);
```
rte_pktmbuf_pool_create_by_ops()和rte_pktmbuf_pool_create()的差异在于前者指定了rte_mempool_ops的名字。
如果上层应用自己没有实现 rte_mempool_ops，或者在eal层初始化时没有指定，则用rte_pktmbuf_pool_create()创建时，默认会使用 ring_mp_mc(支持多生产者多消费者)。

从这两个接口，可以看出，创建pktmbuf pool时，需要指定：mempool的name，mbuf个数，核的local cache大小，mbuf中私有数据的大小，mbuf中data_room的大小以及从哪个socket_id上申请。

![](attachments/Pasted%20image%2020231025113346.png)
## rte_mbuf的内存结构
从rte_pktmbuf_pool_create_by_ops()的接口中的如下代码，可以看出rte_mbuf的内存结构主要由三部分构成：rte_mbuf结构体，私有数据和data room。
```c
    elt_size = sizeof(struct rte_mbuf) + (unsigned)priv_size +
        (unsigned)data_room_size;
    memset(&mbp_priv, 0, sizeof(mbp_priv));
    mbp_priv.mbuf_data_room_size = data_room_size;
    mbp_priv.mbuf_priv_size = priv_size;
```

>其中 **data room包括 headroom 和报文数据区域构成**。即 data_room_size 一般已经包含了 headroom。
>headroom大小由RTE_PKTMBUF_HEADROOM宏控制，默认为128。
>priv_size：就看在创建mbuf的 mempool的时候是否指定了 priv_size 参数为非0 值。
```c
/**
 * Some NICs need at least 2KB buffer to RX standard Ethernet frame without
 * splitting it into multiple segments.
 * So, for mbufs that planned to be involved into RX/TX, the recommended
 * minimal buffer length is 2KB + RTE_PKTMBUF_HEADROOM.
 */
 
#define RTE_MBUF_DEFAULT_DATAROOM   2048
#define RTE_MBUF_DEFAULT_BUF_SIZE   \
    (RTE_MBUF_DEFAULT_DATAROOM + RTE_PKTMBUF_HEADROOM)
```


再根据pktmbuf初始化函数rte_pktmbuf_init()中的如下代码，即可得到rte_mbuf的内存结构。
```c
void rte_pktmbuf_init(struct rte_mempool *mp,
         __rte_unused void *opaque_arg,
         void *_m,
         __rte_unused unsigned i)
{
    struct rte_mbuf *m = _m;
    uint32_t mbuf_size, buf_len, priv_size;

    priv_size = rte_pktmbuf_priv_size(mp);
    mbuf_size = sizeof(struct rte_mbuf) + priv_size;
    buf_len = rte_pktmbuf_data_room_size(mp);

    RTE_ASSERT(RTE_ALIGN(priv_size, RTE_MBUF_PRIV_ALIGN) == priv_size);
    RTE_ASSERT(mp->elt_size >= mbuf_size);
    RTE_ASSERT(buf_len <= UINT16_MAX);

    memset(m, 0, mbuf_size);
    /* start of buffer is after mbuf structure and priv data */
    m->priv_size = priv_size;
    m->buf_addr = (char *)m + mbuf_size;
    m->buf_iova = rte_mempool_virt2iova(m) + mbuf_size;
    m->buf_len = (uint16_t)buf_len;

    /* keep some headroom between start of buffer and data */
    m->data_off = RTE_MIN(RTE_PKTMBUF_HEADROOM, (uint16_t)m->buf_len);

    /* init some constant fields */
    m->pool = mp;
    m->nb_segs = 1;
    m->port = RTE_MBUF_PORT_INVALID;
    rte_mbuf_refcnt_set(m, 1);
    m->next = NULL;
}
```

rte_mbuf的内存结构如下图：
![](attachments/Pasted%20image%2020231025111714.png)

如果没有priv_size, 则内存结构如下所示：
![](attachments/Pasted%20image%2020231025112307.png)

mbuf元信息是需要被频繁访问的部分, 它位于mbuf头部, 且被设计地足够小, 目前占用两个cache lines, 其中访问最频繁的信息位于第一个cache line. mbuf元信息由数据结构rte_mbuf表示. 
如果用户还需要存储业务相关的其他数据, 可以放在mbuf的 priv_data 或者 headroom中, 它是 rte_mbuf与数据之间的一块内存区域.

## 结构体成员
Mbuf结构报头经过精心设计，原先仅占1个Cache Line。随着Mbuf头部携带的信息越来越多，现在Mbuf头部已经调整成两个Cache Line，原则上讲基础性、频繁访问的数据放在一个Cache Line字节，而将功能性扩展的数据放在第二个Cache Line字节。
因此，rte_mbuf结构体由两个cache line构成(见下的 cacheline0 以及 cacheline1， 这2个不是成员，不占用内存空间， 只是一个标识)，其中有很多成员。
```c
typedef void    *MARKER[0];   /**< generic marker for a point in a structure */
```
这是一个柔性数组。长度为0，所以这里在编译时是不占用内存滴。只是一个标记。


```c
struct rte_mbuf {
    RTE_MARKER cacheline0;

    void *buf_addr;           /**< Virtual address of segment buffer. */
    /**
     * Physical address of segment buffer.
     * Force alignment to 8-bytes, so as to ensure we have the exact
     * same mbuf cacheline0 layout for 32-bit and 64-bit. This makes
     * working on vector drivers easier.
     */
    rte_iova_t buf_iova __rte_aligned(sizeof(rte_iova_t));

    /* next 8 bytes are initialised on RX descriptor rearm */
    RTE_MARKER64 rearm_data;
    uint16_t data_off;

    /**
     * Reference counter. Its size should at least equal to the size
     * of port field (16 bits), to support zero-copy broadcast.
     * It should only be accessed using the following functions:
     * rte_mbuf_refcnt_update(), rte_mbuf_refcnt_read(), and
     * rte_mbuf_refcnt_set(). The functionality of these functions (atomic,
     * or non-atomic) is controlled by the RTE_MBUF_REFCNT_ATOMIC flag.
     */
    uint16_t refcnt;
    uint16_t nb_segs;         /**< Number of segments. */

    /** Input port (16 bits to support more than 256 virtual ports).
     * The event eth Tx adapter uses this field to specify the output port.
     */
    uint16_t port;

    uint64_t ol_flags;        /**< Offload features. */

    /* remaining bytes are set on RX when pulling packet from descriptor */
    RTE_MARKER rx_descriptor_fields1;

    /*
     * The packet type, which is the combination of outer/inner L2, L3, L4
     * and tunnel types. The packet_type is about data really present in the
     * mbuf. Example: if vlan stripping is enabled, a received vlan packet
     * would have RTE_PTYPE_L2_ETHER and not RTE_PTYPE_L2_VLAN because the
     * vlan is stripped from the data.
     */
    RTE_STD_C11
    union {
        uint32_t packet_type; /**< L2/L3/L4 and tunnel information. */
        __extension__
        struct {
            uint8_t l2_type:4;   /**< (Outer) L2 type. */
            uint8_t l3_type:4;   /**< (Outer) L3 type. */
            uint8_t l4_type:4;   /**< (Outer) L4 type. */
            uint8_t tun_type:4;  /**< Tunnel type. */
            RTE_STD_C11
            union {
                uint8_t inner_esp_next_proto;
                /**< ESP next protocol type, valid if
                 * RTE_PTYPE_TUNNEL_ESP tunnel type is set
                 * on both Tx and Rx.
                 */
                __extension__
                struct {
                    uint8_t inner_l2_type:4;
                    /**< Inner L2 type. */
                    uint8_t inner_l3_type:4;
                    /**< Inner L3 type. */
                };
            };
            uint8_t inner_l4_type:4; /**< Inner L4 type. */
        };
    };

    uint32_t pkt_len;         /**< Total pkt len: sum of all segments. */
    uint16_t data_len;        /**< Amount of data in segment buffer. */
    /** VLAN TCI (CPU order), valid if PKT_RX_VLAN is set. */
    uint16_t vlan_tci;

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
                 * on PKT_RX_FDIR_* flag in ol_flags.
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
            /**< User defined tags. See rte_distributor_process() */
            uint32_t usr;
        } hash;                   /**< hash information */
    };

    /** Outer VLAN TCI (CPU order), valid if PKT_RX_QINQ is set. */
    uint16_t vlan_tci_outer;

    uint16_t buf_len;         /**< Length of segment buffer. */

    struct rte_mempool *pool; /**< Pool from which mbuf was allocated. */

    /* second cache line - fields only used in slow path or on TX */
    RTE_MARKER cacheline1 __rte_cache_min_aligned;

    struct rte_mbuf *next;    /**< Next segment of scattered packet. */

    /* fields to support TX offloads */
    RTE_STD_C11
    union {
        uint64_t tx_offload;       /**< combined for easy fetch */
        __extension__
        struct {
            uint64_t l2_len:RTE_MBUF_L2_LEN_BITS;
            /**< L2 (MAC) Header Length for non-tunneling pkt.
             * Outer_L4_len + ... + Inner_L2_len for tunneling pkt.
             */
            uint64_t l3_len:RTE_MBUF_L3_LEN_BITS;
            /**< L3 (IP) Header Length. */
            uint64_t l4_len:RTE_MBUF_L4_LEN_BITS;
            /**< L4 (TCP/UDP) Header Length. */
            uint64_t tso_segsz:RTE_MBUF_TSO_SEGSZ_BITS;
            /**< TCP TSO segment size */

            /*
             * Fields for Tx offloading of tunnels.
             * These are undefined for packets which don't request
             * any tunnel offloads (outer IP or UDP checksum,
             * tunnel TSO).
             *
             * PMDs should not use these fields unconditionally
             * when calculating offsets.
             *
             * Applications are expected to set appropriate tunnel
             * offload flags when they fill in these fields.
             */
            uint64_t outer_l3_len:RTE_MBUF_OUTL3_LEN_BITS;
            /**< Outer L3 (IP) Hdr Length. */
            uint64_t outer_l2_len:RTE_MBUF_OUTL2_LEN_BITS;
            /**< Outer L2 (MAC) Hdr Length. */

            /* uint64_t unused:RTE_MBUF_TXOFLD_UNUSED_BITS; */
        };
    };

    /** Shared data for external buffer attached to mbuf. See
     * rte_pktmbuf_attach_extbuf().
     */
    struct rte_mbuf_ext_shared_info *shinfo;

    /** Size of the application private data. In case of an indirect
     * mbuf, it stores the direct mbuf private data size.
     */
    uint16_t priv_size;

    /** Timesync flags for use with IEEE1588. */
    uint16_t timesync;

    uint32_t dynfield1[9]; /**< Reserved for dynamic fields. */
} __rte_cache_aligned;
```
### 长度相关成员
| 成员| 含义|
| ---- | ---- |
| m->buf_addr	| headroom起始地址 |
|m->data_off	| data起始地址相对于buf_addr的偏移|
|m->buf_len	|mbuf和priv之后内存的长度，包含headroom|
|m->pkt_len	|整个mbuf链的data总长度，包括所有分片的长度|
|m->data_len	|实际data的长度。当前的数据长度。<br>如果没有分片，pkt_len与data_len数值应该是相同的|
|m->buf_addr+m->data_off	|实际data的起始地址|
|rte_pktmbuf_mtod(m)	|同上|
|rte_pktmbuf_data_len(m)	|同m->data_len|
|rte_pktmbuf_pkt_len	|同m->pkt_len|
|rte_pktmbuf_data_room_size	|同m->buf_len|
|rte_pktmbuf_headroom	|headroom长度|
|rte_pktmbuf_tailroom	|尾部剩余空间长度|

区别：
uint32_t pkt_len ：表示总的报文大小的长度，包含所有seg分段报文的报文长度  
uint16_t data_len ：表示当前mbuf的报文数据长度  
uint16_t buf_len ：表示当前mbuf的整个buf的长度，包含headroom的长度+data_len

###  rte_mbuf->packet_type
报文类型就是DPDK中所谓的packet type, 当mbuf用于收包时, 它是网卡硬件解析的报文协议类型, 不同的网卡解析的能力不同.
```c
    /*
     * The packet type, which is the combination of outer/inner L2, L3, L4
     * and tunnel types. The packet_type is about data really present in the
     * mbuf. Example: if vlan stripping is enabled, a received vlan packet
     * would have RTE_PTYPE_L2_ETHER and not RTE_PTYPE_L2_VLAN because the
     * vlan is stripped from the data.
     */
    RTE_STD_C11
    union {
        uint32_t packet_type; /**< L2/L3/L4 and tunnel information. */
        __extension__
        struct {
            uint8_t l2_type:4;   /**< (Outer) L2 type. */
            uint8_t l3_type:4;   /**< (Outer) L3 type. */
            uint8_t l4_type:4;   /**< (Outer) L4 type. */
            uint8_t tun_type:4;  /**< Tunnel type. */
            RTE_STD_C11
            union {
                uint8_t inner_esp_next_proto;
                /**< ESP next protocol type, valid if
                 * RTE_PTYPE_TUNNEL_ESP tunnel type is set
                 * on both Tx and Rx.
                 */
                __extension__
                struct {
                    uint8_t inner_l2_type:4;
                    /**< Inner L2 type. */
                    uint8_t inner_l3_type:4;
                    /**< Inner L3 type. */
                };
            };
            uint8_t inner_l4_type:4; /**< Inner L4 type. */
        };
    };
```
32bit的packet type构成如下所示:
```c
0               4               8               12              16
+---------------+---------------+---------------+---------------+
| outer_L2_type | outer_L3_type | outer_L4_type |  tunnel_type  |
+---------------+---------------+---------------+---------------+
| inner_L2_type | inner_L3_type | inner_L4_type |               |
+---------------+---------------+---------------+---------------+
```
为了方便, 这32bit可以使用packet_type成员来一次性访问. 不同网卡对同一个报文的报文类型的识别结果是不同的.

### rte_mbuf->rss
RSS是现代多队列网卡的一个重要功能, 可以对收到的包计算hash值, 然后分到多个队列. 然后用户程序就可以让CPU的多个核心分别处理这些队列, 利用多核CPU优势提升处理性能. 

rte_mbuf结构体使用 hash.rss 来存放网卡计算的RSS hash值, 它是一个32bit数. 注意要配置网卡参数, 启用RSS功能后这个值才有效.

### rte_mbuf->next
Mbuf报头包含包处理所需的所有数据，对于单个Mbuf存放不下的巨型帧（Jumbo Frame），Mbuf还有指向下一个Mbuf结构的指针来形成帧链表结构。
因此，rte_mbuf中有个next域段，如果一个报文只有一个mbuf，则mbuf中的next为NULL;如果一个报文由多个mbuf构成，则mbuf的next被用来指向下一个mbuf。
![](attachments/Pasted%20image%2020231025111907.png)

# 外部内存
## 背景
比如：存储场景中，在一个程序中，网络层面使用DPDK收到数据，然后直接零拷贝的将数据通过SPDK写入到磁盘。
那么就可以考虑，通过spdk的接口开辟一块内存空间，然后在网络层面DPDK中，将这块内存进行注册，作为网卡收发包的内存，网卡DMA直接访问这块内存。

## 操作
```c
(1) 方式一：
// 外部内存申请
void *mmap(void *addr, size_t length, int prot, int flags,
		  int fd, off_t offset);

// 外部内存管理
int rte_malloc_heap_create(const char *heap_name);

int rte_malloc_heap_memory_add(const char *heap_name, void *va_addr, size_t len,
    rte_iova_t iova_addrs[], unsigned int n_pages, size_t page_sz);

int rte_malloc_heap_get_socket(const char *name);


// 外部内存关联 mbuf pool;
const struct rte_memzone *rte_memzone_reserve(const char *name, size_t len, int socket_id,
        unsigned flags);

struct rte_mempool * rte_pktmbuf_pool_create_extbuf(const char *name, unsigned int n,
  unsigned int cache_size, uint16_t priv_size,
  uint16_t data_room_size, int socket_id,
  const struct rte_pktmbuf_extmem *ext_mem,
  unsigned int ext_num);

（2）方式二：
rte_extmem_register
rte_dev_dma_map


(3) 考虑：
收方向的零拷贝；
发方向的零拷贝；
两者的实现是不是不太一样。
```


# rte_mbuf 操作
## rte_pktmbuf_alloc
Mbuf由缓冲池rte_mempool管理，rte_mempool在初始化时一次申请多个mbuf，申请的mbuf个数和长度都由用户指定。
用下面函数向rte_mempool申请一个mbuf：
```c
struct rte_mbuf *rte_pktmbuf_alloc(struct rte_mempool *mp);
```
## rte_pktmbuf_clone
dpdk接收报文并把报文上送上层应用的过程中，报文传输是“零拷贝”，即不需要拷贝报文内容，只需要传送mbuf地址。然而在一个报文上送给多个应用时，仍然需要对报文做拷贝并送给不同的应用。
Librte_mbuf采用“复制rte_mbuf，共享data数据域”的方式实现报文的拷贝。
函数原型如下：
```c
struct rte_mbuf * rte_pktmbuf_clone(struct rte_mbuf *md, struct rte_mempool *mp)
{
    struct rte_mbuf *mc, *mi, **prev;
    uint32_t pktlen;
    uint16_t nseg;

    mc = rte_pktmbuf_alloc(mp);
    if (unlikely(mc == NULL))
        return NULL;

    mi = mc;
    prev = &mi->next;
    pktlen = md->pkt_len;
    nseg = 0;

    do {
        nseg++;
        rte_pktmbuf_attach(mi, md);
        *prev = mi;
        prev = &mi->next;
    } while ((md = md->next) != NULL &&
        (mi = rte_pktmbuf_alloc(mp)) != NULL);

    *prev = NULL;
    mc->nb_segs = nseg;
    mc->pkt_len = pktlen;

    /* Allocation of new indirect segment failed */
    if (unlikely(mi == NULL)) {
        rte_pktmbuf_free(mc);
        return NULL;
    }

    __rte_mbuf_sanity_check(mc, 1);
    return mc;
}
```
rte_pktmbuf_clone()函数首先申请一个新的rte_mbuf，我们称这个mbuf为indirect buffer，用mi表示，参数md称为direct buffer。
函数将md的各结构体成员（引用计数refcnt除外）一一复制给mi，同时将md的引用计数refcnt增1。此时，mi->pkt.data指向md的data数据域。

Rte_pktmbuf_clone()要求参数md必须是direct buffer，我们可以通过判断md->buf_addr – sizeof(struct rte_mbuf) == md 是否为真，确定md是否为direct buffer，该功能由宏RTE_MBUF_DIRECT(mb)实现。

> 注意：rte_pktmbuf_clone()提供的拷贝机制在某些场景不一定适用，如多个应用竞争data数据域。为避免竞争的发生，使用者可以通过拷贝data数据域实现自己的clone()。
## rte_pktmbuf_copy
**mbuf克隆属于浅度拷贝, mbuf拷贝是深度拷贝**，接口如下：
```c
struct rte_mbuf * rte_pktmbuf_copy(const struct rte_mbuf *m, struct rte_mempool *mp, uint32_t off, uint32_t len)
```
rte_pktmbuf_copy支持拷贝多段的mbuf，但是mbuf的私有数据不会被拷贝。

## rte_pktmbuf_free
用下面函数释放一个mbuf，释放过程即把mbuf归还到rte_mempool中：
```c
void rte_pktmbuf_free(struct rte_mbuf *m);
```
根据m的引用计数和m的indirect/direct类型，rte_pktmbuf_free()分以下方式释放m：

如果m的引用计数大于1，则只将m的引用计数减1，函数返回；

如果m的引用计数是1且m是direct类型，则将m的引用计数置0，然后把m归还mempool，函数返回；

如果m的引用计数是1且m是indirect类型，则rte_pktmbuf_free()将m引用计数置0，同时将m对应的direct buffer的引用计数减1(减1后引用计数为0则把direct buffer归还mempool)，把m归还mempool，函数返回；

`Rte_pktmbuf_free()`通过宏`RTE_MBUF_FROM_BADDR(m->buf_addr)`找到m对应的direct buffer，宏实现如下：
```c
#define RTE_MBUF_FROM_BADDR(ba) (((struct rte_mbuf *)(ba)) - 1)
```
`Rte_pktmbuf_free()`通过判断 `m != RTE_MBUF_FROM_BADDR(m->buf_addr)`是否为真判断m的` indirect/direct `类型。

## 封装与解封装操作
Rte_mbuf的结构与linux内核协议栈的skb_buf相似，在保存报文的内存块前后分别保留headroom和tailroom，以方便应用解封报文。
Headroom默认128字节，可以通过宏RTE_PKTMBUF_HEADROOM调整。

我们可以通过`m->pkt.data – m->buf_addr`计算出headroom长度，通过`m->buf_len – m->pkt.data_len – headroom_size`计算出tailroom长度。这些计算过程都由以下函数实现：
```c
uint16_t rte_pktmbuf_headroom(const struct rte_mbuf *m)

uint16_t rte_pktmbuf_tailroom(const struct rte_mbuf *m)
```

假设`m->pkt.data`指向报文的二层首地址，我们可以通过以下一系列操作剥去报文的二层头部：
```c
m->pkt.data += 14;
m->pkt.data_len -= 14;
m->pkt.pkt_len -= 14;
```
这些操作已经由`rte_pktmbuf_adj()`实现，函数原型如下：
```c
char *rte_pktmbuf_adj(struct rte_mbuf *m, uint16_t len)
```

我们可以通过以下一系列操作为IP报文封装二层头部：
```c
m->pkt.data -= 14;
m->pkt.data_len += 14;
m->pkt.pkt_len += 14;
```
这些操作由`rte_pktmbuf_prepend()`实现，函数原型如下：
```c
char *rte_pktmbuf_prepend(struct rte_mbuf *m, uint16_t len)
```

如果需要在tailroom 中加入N个字节数据，我们可以通过以下操作完成：
```c
tail = m->pkt.data + m->pkt.data_len; // tail记录tailroom首地址
m->pkt.data_len += N;
m->pkt.pkt_len += N;
```
这些操作由`rte_pktmbuf_append()`实现，函数原型如下：
```c
char *rte_pktmbuf_append(struct rte_mbuf *m, uint16_t len)
```

librte_mbuf还提供了`rte_pktmbuf_trim()`函数，用来移除mbuf中data数据域的最后N个字节，函数实现如下：
```c
m->pkt.data_len -= N;
m->pkt.pkt_len -= N;
```
函数原型如下：
```c
int rte_pktmbuf_trim(struct rte_mbuf *m, uint16_t len)
```

## rte_mbuf读取操作
读rte_mbuf报文内容:  
```c
void *rte_pktmbuf_read(const struct rte_mbuf *m, uint32_t off, uint32_t len, void *buf)  
```
从报文长度偏移off位置读取len个字节的报文内容到buf中。  

支持dump mbuf的头信息和报文内容到文件:  
```c
void rte_pktmbuf_dump(FILE *f, const struct rte_mbuf *m, unsigned dump_len)  
```
dump_len：表示要dump的报文长度。

## 其他操作
当我们从mbuf pool alloc一块mbuf过来的时候，都会reset一下mbuf的变量。`rte_pktmbuf_reset` 
```c
/**
 * Reset the fields of a packet mbuf to their default values.
 *
 * The given mbuf must have only one segment.
 *
 * @param m
 *   The packet mbuf to be reset.
 */
static inline void rte_pktmbuf_reset(struct rte_mbuf *m)
{
    m->next = NULL;
    m->pkt_len = 0;
    m->tx_offload = 0;
    m->vlan_tci = 0;
    m->vlan_tci_outer = 0;
    m->nb_segs = 1;
    m->port = RTE_MBUF_PORT_INVALID;

    m->ol_flags &= EXT_ATTACHED_MBUF;
    m->packet_type = 0;
    rte_pktmbuf_reset_headroom(m);

    m->data_len = 0;
    __rte_mbuf_sanity_check(m, 1);
}
```

# 其他
## Direct与Indirect mbuf
为了避免某些场景(如复制报文, IP分片等)下的内存拷贝, DPDK引入了indirect mbuf的概念.
Direct mbuf就是普通的mbuf, 实际持有数据;
 而indirect mbuf不实际持有数据, 而是附着(attach)在direct mbuf上, 它的数据指针指向direct mbuf的数据. 当direct mbuf被attach时, 它的引用计数+1; 反之当被detach时, 引用计数-1, 当减为0时, direct mbuf被释放.
 Indirect mbuf不能实质上attach到另一个indirect mbuf, 这么做最终会attach到后者attach的direct mbuf. 所以indirect mbuf的引用计数只能是1.
 Indirect mbuf也不能重新attach到direct mbuf, 除非先detach.
使用mbuf的attach/detach接口可以执行相应操作, 但推荐使用clone接口, 因为它可以正确处理indirect mbuf的初始化, 以及mbuf chain.

由于indirect mbuf不实际持有数据, 因此可以调整存储它的内存池的参数, 如元素大小, 以便减少内存占用.
# 参考
```c
http://m.blog.chinaunix.net/uid-70024505-id-5870762.html
https://www.cnblogs.com/ziding/p/4214499.html
```