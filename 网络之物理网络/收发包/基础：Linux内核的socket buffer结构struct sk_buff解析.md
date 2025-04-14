```table-of-contents
```
# sk_buff介绍
sk_buff的意思是socket buffer，这是Linux网络子系统中的核心数据结构。它表示接收或发送数据包的包头信息，并包含很多成员变量供网络代码中的各子系统使用。

## 作用
内核中sk_buff结构体在各层协议之间传输不是用拷贝sk_buff结构体，而是通过增加协议头和移动指针来操作的。
这个结构被网络的不同层(MAC或者其他二层链路协议，三层的IP，四层的TCP或UDP等)使用，并且其中的成员变量在结构从一层向另一层传递时改变。
L4向L3传递前会添加一个L4的头部，同样，L3向L2传递前，会添加一个L3的头部。添加头部比在不同层之间拷贝数据的效率更高。

# struct sk_buff 结构体
## 结构体定义
```cpp
/*  include/linux/skbuff.h */
struct sk_buff {
    union {
        struct {
            /* These two members must be first. 
这两个域是用来连接相关的skb的（如果有分片的话，可以通过它们将分片链接到一起），sk_buff是双链表结构。
              */
            struct sk_buff      *next;  /*链表中的下一个skb*/
            struct sk_buff      *prev; /*链表中的上一个skb*/

            union {
                ktime_t     tstamp; /*记录接受或者传输报文的时间戳*/
                struct skb_mstamp skb_mstamp;
            };
        };
        struct rb_node      rbnode; /* 红黑树，used in netem, ip4 defrag, and tcp stack */
    };

    union {
        struct sock     *sk; /*指向报文所属的套接字指针*/
        int         ip_defrag_offset;
    };

    struct net_device   *dev; /*记录接受或发送报文的网络设备*/

    /*
     * This is the control buffer. It is free to use for every
     * layer. Please put your private variables there. If you
     * want to keep them across layers you have to do a skb_clone()
     * first. This is owned by whoever has the skb queued ATM.
     */
    char            cb[48] __aligned(8); /*保存与协议相关的控制信息，每个协议可能独立使用这些信息*/

    unsigned long       _skb_refdst; /*主要用于路由子系统，保存路由相关的东西*/
    void            (*destructor)(struct sk_buff *skb);
#ifdef CONFIG_XFRM
    struct  sec_path    *sp;
#endif
#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
    struct nf_conntrack *nfct;
#endif
#if IS_ENABLED(CONFIG_BRIDGE_NETFILTER)
    struct nf_bridge_info   *nf_bridge;
#endif
    unsigned int        len,   
/*整个数据区域的长度，
这里的len = length(实际线性数据，不包括头空间和尾空间) + length(非线性数据)
len = (tail - data) + data_len
这个len中数据区长度是个有效长度,因为不删除协议头,
所以只计算有效协议头和包内容。如：当在L3时，不会计算L2的协议头长度。*/
                data_len; /*非线性数据，length(实际线性数据 = skb->len - skb->data_len)*/
    __u16           mac_len, /*mac层报头的长度*/
                hdr_len; /*用于clone时，表示clone的skb的头长度*/

    /* Following fields are _not_ copied in __copy_skb_header()
     * Note that queue_mapping is here mostly to fill a hole.
     */
    kmemcheck_bitfield_begin(flags1);
    __u16           queue_mapping;

/* if you move cloned around you also must adapt those constants */
#ifdef __BIG_ENDIAN_BITFIELD
#define CLONED_MASK (1 << 7)
#else
#define CLONED_MASK 1
#endif
#define CLONED_OFFSET()     offsetof(struct sk_buff, __cloned_offset)

    __u8            __cloned_offset[0];
    __u8            cloned:1,
                nohdr:1,
                fclone:2,
                peeked:1,
                head_frag:1,
                xmit_more:1,
                pfmemalloc:1;
    kmemcheck_bitfield_end(flags1);

    /* fields enclosed in headers_start/headers_end are copied
     * using a single memcpy() in __copy_skb_header()
     */
    /* private: */
    __u32           headers_start[0];
    /* public: */

/* if you move pkt_type around you also must adapt those constants */
#ifdef __BIG_ENDIAN_BITFIELD
#define PKT_TYPE_MAX    (7 << 5)
#else
#define PKT_TYPE_MAX    7
#endif
#define PKT_TYPE_OFFSET()   offsetof(struct sk_buff, __pkt_type_offset)

    __u8            __pkt_type_offset[0];
    __u8            pkt_type:3; /*标记帧的类型*/
    __u8            ignore_df:1;
    __u8            nfctinfo:3;
    __u8            nf_trace:1;

    __u8            ip_summed:2;
    __u8            ooo_okay:1;
    __u8            l4_hash:1;
    __u8            sw_hash:1;
    __u8            wifi_acked_valid:1;
    __u8            wifi_acked:1;
    __u8            no_fcs:1;

    /* Indicates the inner headers are valid in the skbuff. */
    __u8            encapsulation:1;
    __u8            encap_hdr_csum:1;
    __u8            csum_valid:1;
    __u8            csum_complete_sw:1;
    __u8            csum_level:2;
    __u8            csum_bad:1;
#ifdef CONFIG_IPV6_NDISC_NODETYPE
    __u8            ndisc_nodetype:2;
#endif
    __u8            ipvs_property:1;

    __u8            inner_protocol_type:1;
    __u8            remcsum_offload:1;
#ifdef CONFIG_NET_SWITCHDEV
    __u8            offload_fwd_mark:1;
#endif
    /* 2, 4 or 5 bit hole */

#ifdef CONFIG_NET_SCHED
    __u16           tc_index;   /* traffic control index */
#ifdef CONFIG_NET_CLS_ACT
    __u16           tc_verd;    /* traffic control verdict */
#endif
#endif

    union {
        __wsum      csum;
        struct {
            __u16   csum_start;
            __u16   csum_offset;
        };
    };
    __u32           priority; /*优先级，主要用于QOS*/
    int         skb_iif; /*接收设备的index*/
    __u32           hash;
    __be16          vlan_proto;
    __u16           vlan_tci;
#if defined(CONFIG_NET_RX_BUSY_POLL) || defined(CONFIG_XPS)
    union {
        unsigned int    napi_id;
        unsigned int    sender_cpu;
    };
#endif
#ifdef CONFIG_NETWORK_SECMARK
    __u32       secmark;
#endif

    union {
        __u32       mark;
        __u32       reserved_tailroom;
    };

    union {
        __be16      inner_protocol;
        __u8        inner_ipproto;
    };

    __u16           inner_transport_header; 
    __u16           inner_network_header; 
    __u16           inner_mac_header; 

    __be16          protocol; /*协议类型*/
    __u16           transport_header;
    __u16           network_header;
    __u16           mac_header;

    /* private: */
    __u32           headers_end[0];
    /* public: */

    /* These elements must be at the end, see alloc_skb() for details.  */
    sk_buff_data_t      tail; /*指向数据区中实际数据结束的位置*/
    sk_buff_data_t      end; /*指向数据区中结束的位置（非实际数据区域结束位置）*/
     
    unsigned char       *head, /* 指向数据区中开始的位置（非实际数据区域开始位置）*/
                *data; /*指向数据区中实际数据开始的位置*/
    unsigned int        truesize; /*缓冲区总长度*/
    atomic_t        users; 
};
```

## 布局layout
用一张图来表示sk_buff和数据区的关系：
![](attachments/Pasted%20image%2020231024141113.png)
![](attachments/Pasted%20image%2020231024140838.png)


### 控制信息
sk_buff结构体中的都是sk_buff的控制信息，是网络数据包的一些配置，真正储存数据的是sk_buff结构体中几个指针指向的数据区中。

### 线性区域
- head 到data 之间，称为headroom.
- data 到tail 之间，存放包的数据。
- tail 到end 之间，称为tailroom.

1）线性数据：head - end
2）实际线性数据：data - tail，不包含线性数据中的头空间和尾空间。

线性数据区是用来存放各层协议头部和应用层发下来的数据。各层协议头部相关信息放在线性数据区中。
线性数据区的大小 = (skb->end - skb->head)，对于每个数据包来说这个大小都是固定不变的。在传输过程中skb->end和skb->head所指向的地址都是不变的。
实际数据指针为data和tail，data指向实际数据开始的地方，tail指向实际数据结束的地方。

#### 初始化sk_buff
sk_buff结构数据区刚被申请好，此时 head 指针、data 指针、tail 指针都是指向同一个地方。记住前面讲过的：head 指针和 end 指针指向的位置一直都不变，而对于数据的变化和协议信息的添加都是通过 data 指针和 tail 指针的改变来表现的。
![](attachments/Pasted%20image%2020231024142343.png)

对于刚刚通过alloc_skb() 方法申请出来的skb，head,tail,data 三个指针都指向同一位置，而tail 和end 之间有一段根据alloc_skb(len, flag) 方法的参数申请出来的空间。
![](attachments/Pasted%20image%2020231024145816.png)

#### 初始定位skb_reserve(m)
开始准备存储应用层下发过来的数据，通过调用函数 skb_reserve(m) 来使 data 指针和 tail 指针同时向下移动，空出一部分空间来为后期添加协议信息。
> 注：m一般为最大协议头长度，内核中定义。
![](attachments/Pasted%20image%2020231024142919.png)

为了给协议头预留空间，可以使用skb_reserve(skb, head_len)方法，该方法会根据参数将data 指针后移，扩展headroom.
![](attachments/Pasted%20image%2020231024145843.png)
#### 储存应用层数据skb_put()
开始存储数据了，通过调用函数 skb_put() 来使 tail 指针向下移动空出空间来添加数据，此时 skb->data 和 skb->tail 之间存放的都是数据信息，无协议信息。
![](attachments/Pasted%20image%2020231024143044.png)


可以通过skb_put(skb, data_len) 方法移动tail 指针，扩展用户数据空间。该方法同时会增加skb->len.
![](attachments/Pasted%20image%2020231024145932.png)
这些空间都是从tailroom “挤”出来的，因此需要保证tailroom 有足够的空间。另外要注意skb_put() 只能在没有page 的数据的情况下调用。

#### 添加协议头
为了添加协议头的内容，需要调用skb_push()方法，这个方法和skb_put()类似，但它是从headroom 挤出空间，data 指针会往前移动，它同样会增加skb->len.
![](attachments/Pasted%20image%2020231024143153.png)

添加一个四层头：
![](attachments/Pasted%20image%2020231024150038.png)
再添加一个三层头：
![](attachments/Pasted%20image%2020231024150046.png)

以上是对于没有page buffer，只有线性buffer 时的操作，对于比较大的包，还需要用到线性buffer 以外的部分。
对于部分驱动来讲，有一个copybreak 的字段，当包的大小大于copybreak 时，只将起始一部分数据放入skb->data（协议头等），而剩余部分会存放于page buffer. 后续添加数据时应该不再调用skb_put()方法，否则数据顺序是有问题的。

### 非线性数据区
当线性数据区不够用的时候就会启用非线性数据区作为数据区域的扩展，skb中用skb_shared_info分片结构体来配置非线性数据。

skb_shared_info结构体是和skb中的线性数据区一体的，所以在skb的各种操作时都会把这两个结构看作是一个结构来操作。如：
1. 当sk_buff结构的线性数据区申请和释放空间时，分片结构会跟着数据区一起分配和释放。
2. 克隆skb时，sk_buff的线性数据区和分片结构都由分片结构中的dataref成员字段来标识是否被引用。

![](attachments/Pasted%20image%2020231024143356.png)

从上图中可以看出来非线性数据区接到skb->end的位置后，skb->end的下一个字节就作为非线性数据区的开始。
end指针的下个字节可以作为分片结构的开始，获取end指针的位置要强行转成分片结构，内核中有定义好的宏：
```cpp
#define skb_shinfo(SKB) ((struct skb_shared_info *)(skb_end_pointer(SKB)))
```

#### struct skb_shared_info
```cpp
/*  include/linux/skbuff.h */
struct skb_shared_info {
    unsigned char   nr_frags; /*表示有多少分片结构*/
    __u8        tx_flags;
    unsigned short  gso_size;
    /* Warning: this field is not always filled in (UFO)! */
    unsigned short  gso_segs;
    unsigned short  gso_type;
    struct sk_buff  *frag_list; /*一种类型的分配数据*/
    struct skb_shared_hwtstamps hwtstamps;
    u32     tskey;
    __be32          ip6_frag_id;

    /*
     * Warning : all fields before dataref are cleared in __alloc_skb()
     */
    atomic_t    dataref; /*用于引用计数，克隆一个skb结构体时会增加一个引用计数*/

    /* Intermediate layers must ensure that destructor_arg
     * remains valid until skb destructor */
    void *      destructor_arg;

    /* must be last field, see pskb_expand_head() */
    skb_frag_t  frags[MAX_SKB_FRAGS];  /*保存分页数据，skb->data_len = 所有数组数据长度之和*/

};
```


在sk_buff的数据缓冲区的末尾，即end指针所指向的地址起紧跟着有一个skb_shared_info结构，保存了数据块的附加信息。

非线性数据区有两种不同的构成数据的方式：
- **用数组存储的分片数据区**
采用是是结构体中的frags[MAX_SKB_FRAGS]  
对于frags[]一般用在当数据比较多，在线性数据区装不下的时候，skb_frag_t中是一页一页的数据，skb_frag_struct结构体如下：
```cpp
/*  include/linux/skbuff.h */
typedef struct skb_frag_struct skb_frag_t;

struct skb_frag_struct {
    struct {
        struct page *p; /*指向分片数据区的指针，类似于sk_buff中的data指针*/
    } page;
#if (BITS_PER_LONG > 32) || (PAGE_SIZE >= 65536)
    __u32 page_offset;
    __u32 size;
#else
    __u16 page_offset; /*偏移量，表示相对开始位置的页偏移量*/
    __u16 size; /*page中的数据长度*/
#endif
};
```
  
下图显示了frags是怎么分配分片数据的：
![](attachments/Pasted%20image%2020231024145116.png)

- **frag_list指针来指向的分片数据**
![](attachments/Pasted%20image%2020231024145214.png)

#### 是否存在非线性buffer
```c
static inline bool skb_is_nonlinear(const struct sk_buff *skb)
{
	return skb->data_len;
}
```
skb_is_nonlinear() 方法可以帮助判断是否存在page buffer，对于有page buffer 的skb 来讲，之前已经提到skb->data_len 用于指示这部分数据的大小，而之前刚刚提到的通过skb_put() 和skb_push() 方法添加进去的数据的大小就是skb->len-skb->data_len，也就是skb_headlen()。

## 字段含义
### struct sock *sk

这个指针在网络包由本机发出或者由本机进程接收时有效，因为插口socket相关的信息被L4(TCP或 UDP)或者用户空间程序使用。如果sk_buff只在转发中使用(这意味着，源地址和目的地址都不是本机地址)，这个指针是NULL

### sk_buff->tstamp
这个变量只对接收到的包有意义。它代表包接收时的时间戳，或者有时代表包准备发出时的时间戳。它在netif_rx里面由函数net_timestamp设置，而netif_rx是设备驱动收到一个包后调用的函数。
net_enable_timestamp() 和net_disable_timestamp() 函数可用于启用或禁用时间戳。在用户态，可以通过socket 选项SIOCGSTAMP 管理。

### sk_buff->users
这是一个引用计数，用于计算有多少实体引用了这个sk_buff缓冲区。它的主要用途是防止释放sk_buff后，还有其他实体引用这个sk_buff。
因此，每个引用这个缓冲区的实体都必须在适当的时候增加或减小这个变量。这个计数器只保护sk_buff结构本身，而缓冲区的数据部分由类似的计数器 (dataref)来保护.
有时可以用atomic_inc和atomic_dec函数来直接增加或减小users，但是，通常还是使用函数 skb_get和kfree_skb来操作这个变量。

### 长度相关


```c
unsigned int len,  
data_len;  
__u16 mac_len,  
hdr_len;
```

- **skb->data_len**: 
skb中的分片（fragment）数据（即非线性数据）的长度。  
- **skb->len**: 
skb中的数据块的总长度，数据块包括实际线性数据和非线性数据，非线性数据为data_len，所以skb->len= (data - tail) + data_len。
- **skb->truesize**: 
skb的总长度，包括sk_buff结构和数据部分，skb=sk_buff控制信息 + 线性数据（包括头空间和尾空间） + skb_shared_info控制信息 + 非线性数据。
所以：skb->truesize = sizeof(struct sk_buff) + (head - end) + sizeof(struct skb_shared_info) + data_len。

- mac_len
mac 头大小。


### 指针相关
![](attachments/Pasted%20image%2020231024140846.png)
> 注意：head 指针和 end 指针指向的位置一直都不变，而对于数据的变化和协议信息的添加都是通过 data 指针和 tail 指针的改变来表现的。


- head和end指向数据缓存区域的start和end.
- data和tail指向实际协议数据区域的start和end.
- `mac_header` 指向MAC头的start, `network_header` 和 `transport_header` 分别指向network和transport层的数据头.

### pkt_type
这个变量表示帧的类型，分类是由L2的目的地址来决定的。
这个值在网卡驱动程序中由函数eth_type_trans通过判断目的以太网地址来确定。
如果目的地址是FF:FF:FF:FF:FF:FF，则为广播地址，pkt_type = PACKET_BROADCAST；
如果最高位为1,则为组播地址，pkt_type = PACKET_MULTICAST；
如果目的mac地址跟本机mac地址不相等，则不是发给本机的数据报，pkt_type = PACKET_OTHERHOST；
否则就是缺省值PACKET_HOST。

### sk_buff->dst
这个变量在路由子系统中使用.

### sk_buff->protocol
这个变量是高层协议从二层设备的角度所看到的协议。典型的协议包括IP，IPV6和ARP。完整的列表在 include/linux/if_ether.h中。
由于每个协议都有自己的协议处理函数来处理接收到的包，因此，这个域被设备驱动用于通知上层调用哪个协议处理函数。每个网络驱动都调用netif_rx来通知上层网络协议的协议处理函数，因此protocol变量必须在这些协议处理函数调用之前初始化。
### sk_buff->cloned
一个布尔标记，当被设置时，表示这个结构是另一个sk_buff的克隆

### sk_buff->cb[48]
这是一个“control buffer”，或者说是一个私有信息的存储空间，由每一层自己维护并使用。
它在分配sk_buff结构时分配(它目前的大小是48字节，已经足够为每一层存储必要的私有信息了)。在每一层中，访问这个变量的代码通常用宏实现以增强代码的可读性。例如，TCP用这个变量存储tcp_skb_cb结构。

### sk_buff->input_dev
这是收到包的网络设备的指针。如果包是本地生成的，这个值为NULL。对以太网设备来说，这个值由eth_type_trans初始化,它主要被流量控制代码使用。
### sk_buff->dev
这个变量的类型是net_device，net_device它代表一个网络设备。dev的作用与这个包是准备发出的包还是刚接收的包有关。
当收到一个包时，设备驱动会把sk_buff的dev指针指向收到这个包的网络设备；
当一个包被发送时，这个变量代表将要发送这个包的设备。
在发送网络包时设置这个值的代码要比接收网络包时设置这个值的代码复杂。
有些网络功能可以把多个网络设备组成一个虚拟的网络设备(也就是说，这些设备没有和物理设备直接关联)，并由一个虚拟网络设备驱动管理。当虚拟设备被使用时，dev指针指向虚拟设备的net_device结构。
而虚拟设备驱动会在一组设备中选择一个设备并把dev指针修改为这个设备的net_device结构。因此，在某些情况下，指向传输设备的指针会在包处理过程中被改变。

### sk_buff->sp
这个变量被IPSec协议用于跟踪传输的信息。
### sk_buff->priority
这个变量描述发送或转发包的QoS类别。
如果包是本地生成的，socket层会设置priority变量。如果包是将要被转发的， rt_tos2priority函数会根据ip头中的Tos域来计算赋给这个变量的值。这个变量的值与DSCP(DiffServ CodePoint)没有任何关系。

## 其他结构体定义
### struct sk_buff_head

```c
struct sk_buff_head {
    /* These two members must be first. */
    struct sk_buff  *next;
    struct sk_buff  *prev;
    __u32       qlen;
    spinlock_t  lock;
};


```
- `qlen`：链表长度
- `lock`：用来控制sk_buff链表并发操作的自旋锁

> 注：sk_buff和sk_buff_head的前两个元素是一样的：next和prev指针。这使得它们可以放到同一个链表中，尽管 sk_buff_head要比sk_buff小得多。
> 

### 结构体之间的关系
![](attachments/Pasted%20image%2020231024112442.png)

# 操作函数
## 缓冲区操作函数
### skb_reserve
从空白缓冲区中分配len字节的数据区，通过减少尾部空间，增加一个空&sk_buff的首部空间，将data指针和tail指针同时下移。这个操作在存储空间的头部预留len长度的空隙。

### skb_put
在缓冲区的尾部空间扩充len字节数据区，将tail指针下移，并增加skb的len值。
data和tail之间的空间就是可以存放网络报文的空间。这个操作增加了可以存储网络报文的空间，但是增加不能使 tail的值大于end的值，skb的len值大于truesize 的值。

### skb_push
在缓冲区的头部空间扩充len字节的数据区。将data指针上移，并增加skb的len值。
这个操作在存储空间的头部增加了一段可以存储网络报文的空间，但是增加不能使data的值小于 head的值，skb的len值大于truesize的值。

### skb_pull
从缓冲区的数据区删除len字节，把腾出的内存归还给头部空间。将data指针下移，并减小skb的len值。这个操作使data指针指向下一层网络报文的头部。

## 缓冲区分配、克隆和释放函数
### alloc_skb
alloc_skb是net/core/skbuff.c里面定义的，用于分配缓冲区的函数。
我们已经知道，数据缓冲区和缓冲区的描述结构(sk_buff结构)是两种不同的实体，这就意味着，在分配一个缓冲区时，需要分配两块内存(一个是缓冲区，一个是缓冲区的描述结构sk_buff)。

### skb_clone
如果一个缓冲区需要被不同的用户独立地操作，而这些用户可能会修改sk_buff中某些变量的值，内核没有必要为每个用户复制一份完整的 sk_buff以及相应的缓冲区。
克隆过程只复制sk_buff结构，同时修改缓冲区的引用计数以避免共享的数据被提前释放。克隆缓冲区使用skb_clone函数。
> 注：clone的意思就是只复制skb而不复制data域。

被克隆的sk_buff不会放在任何链表中，同时也不会有到socket的引用。
原始的和克隆的sk_buff中的skb->cloned值都被置为 1。克隆包的skb->users值被置为1，这样，在释放时，可以先释放sk_buff结构。同时，缓冲区的引用计数(dataref)增加1 (因为有多个sk_buff结构指向它)。克隆缓冲区的结构如下图所示。
![](attachments/Pasted%20image%2020231024152808.png)

### kfree_skb
kfree_skb函数释放缓冲区，并把它返回给缓冲池(缓存)。只有在skb->users为1的情况下才释放内存(没有人引用这个结构)。否则，它只是简单地减小skb->users。

### skb_copy && pskb_copy
当一个skb被clone之后，这个skb的数据区是不能被修改的。
这就意味着，我们存取数据不需要任何锁。可是有时我们需要修改数据区，这个时候会有两个选择：
一个是我们只修改线性区域，也就是head和end之间的区域，可以使用pskb_copy来复制这部分数据。
一种是还要修改切片数据，也就是skb_shared_info，就必须使用skb_copy。

- **pskb_copy**
先alloc一个新的skb，然后调用skb_copy_from_linear_data来复制线性区的数据，并更新相关域，最后复制切片数据的指针。如下图所示：
![](attachments/Pasted%20image%2020231024153229.png)

- **skb_copy函数**
先alloc一个新的skb，然后复制skb的所有数据段，包括切片数据。如下图所示：
![](attachments/Pasted%20image%2020231024153236.png)

# 参考
```c

# Linux内核中sk_buff结构详解
https://www.jianshu.com/p/3738da62f5f6

http://vger.kernel.org/~davem/skb_data.html

https://blog.csdn.net/weixin_43564241/article/details/123591833

https://www.jianshu.com/p/3738da62f5f6

# socket buffer结构解析 [文章特别好，从代码层面介绍；这个作者的整个系列文章都不错；+++++]
https://wiki.dreamrunner.org/public_html/Linux/Networks/sk_buff-structure-analysis.html

```