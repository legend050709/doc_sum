```table-of-contents
```

# 回调函数
## eth相关的回调函数
参考：[dpdk-22.11 ethdev api](https://doc.dpdk.org/api-22.11/rte__ethdev_8h.html)
在其中搜索 `callback` ，如下所示：

```c
typedef uint16_t(*  rte_rx_callback_fn) (uint16_t port_id, uint16_t queue, struct rte_mbuf *pkts[], uint16_t nb_pkts, uint16_t max_pkts, void *user_param)
 
typedef uint16_t(*  rte_tx_callback_fn) (uint16_t port_id, uint16_t queue, struct rte_mbuf *pkts[], uint16_t nb_pkts, void *user_param)
 
typedef int(*   rte_eth_dev_cb_fn) (uint16_t port_id, enum rte_eth_event_type event, void *cb_arg, void *ret_param)
```

![](attachments/Pasted%20image%2020250126123643.png)

![](attachments/Pasted%20image%2020250126123736.png)

### eth收发包的回调函数
```c
const struct rte_eth_rxtx_callback* rte_eth_add_rx_callback(uint16_t port_id, uint16_t queue_id, rte_rx_callback_fn fn, void *user_param);

const struct rte_eth_rxtx_callback* rte_eth_add_tx_callback(uint16_t port_id, uint16_t queue_id, rte_tx_callback_fn fn, void *user_param);

/**
 * @internal
 * Structure used to hold information about the callbacks to be called for a
 * queue on Rx and Tx.
 */
struct rte_eth_rxtx_callback {
    struct rte_eth_rxtx_callback *next;
    union{
        rte_rx_callback_fn rx;
        rte_tx_callback_fn tx;
    } fn;
    void *param;
};


/**
 * Function type used for Rx packet processing packet callbacks.
 *
 * The callback function is called on Rx with a burst of packets that have
 * been received on the given port and queue.
 *
 * @param port_id
 *   The Ethernet port on which Rx is being performed.
 * @param queue
 *   The queue on the Ethernet port which is being used to receive the packets.
 * @param pkts
 *   The burst of packets that have just been received.
 * @param nb_pkts
 *   The number of packets in the burst pointed to by "pkts".
 * @param max_pkts
 *   The max number of packets that can be stored in the "pkts" array.
 * @param user_param
 *   The arbitrary user parameter passed in by the application when the callback
 *   was originally configured.
 * @return
 *   The number of packets returned to the user.
 */
typedef uint16_t (*rte_rx_callback_fn)(uint16_t port_id, uint16_t queue, struct rte_mbuf *pkts[], uint16_t nb_pkts, uint16_t max_pkts, void *user_param);


/**
 * Function type used for Rx packet processing packet callbacks.
 *
 * The callback function is called on Rx with a burst of packets that have
 * been received on the given port and queue.
 *
 * @param port_id
 *   The Ethernet port on which Rx is being performed.
 * @param queue
 *   The queue on the Ethernet port which is being used to receive the packets.
 * @param pkts
 *   The burst of packets that have just been received.
 * @param nb_pkts
 *   The number of packets in the burst pointed to by "pkts".
 * @param max_pkts
 *   The max number of packets that can be stored in the "pkts" array.
 * @param user_param
 *   The arbitrary user parameter passed in by the application when the callback
 *   was originally configured.
 * @return
 *   The number of packets returned to the user.
 */
typedef uint16_t (*rte_rx_callback_fn)(uint16_t port_id, uint16_t queue,
    struct rte_mbuf *pkts[], uint16_t nb_pkts, uint16_t max_pkts,
    void *user_param);

/**
 * Function type used for Tx packet processing packet callbacks.
 *
 * The callback function is called on Tx with a burst of packets immediately
 * before the packets are put onto the hardware queue for transmission.
 *
 * @param port_id
 *   The Ethernet port on which Tx is being performed.
 * @param queue
 *   The queue on the Ethernet port which is being used to transmit the packets.
 * @param pkts
 *   The burst of packets that are about to be transmitted.
 * @param nb_pkts
 *   The number of packets in the burst pointed to by "pkts".
 * @param user_param
 *   The arbitrary user parameter passed in by the application when the callback
 *   was originally configured.
 * @return
 *   The number of packets to be written to the NIC.
 */
typedef uint16_t (*rte_tx_callback_fn)(uint16_t port_id, uint16_t queue, struct rte_mbuf *pkts[], uint16_t nb_pkts, void *user_param);

```
## mempool相关的回调函数

参考：[dpdk-22.11 mempool api](https://doc.dpdk.org/api-22.11/rte__mempool_8h.html)
在其中搜索 `callback or cb` ，如下所示：

![](attachments/Pasted%20image%2020250126124129.png)


### 详细介绍
```c
struct rte_mempool * rte_mempool_create(const char *name, unsigned n, unsigned elt_size,
    unsigned cache_size, unsigned private_data_size,
    rte_mempool_ctor_t *mp_init, void *mp_init_arg,
    rte_mempool_obj_cb_t *obj_init, void *obj_init_arg,
    int socket_id, unsigned flags);


/**
 * A mempool constructor callback function.
 *
 * Arguments are the mempool and the opaque pointer given by the user in
 * rte_mempool_create().
 */
typedef void (rte_mempool_ctor_t)(struct rte_mempool *, void *);



/**
 * An object callback function for mempool.
 *
 * Used by rte_mempool_create() and rte_mempool_obj_iter().
 */
typedef void (rte_mempool_obj_cb_t)(struct rte_mempool *mp, void *opaque, void *obj, unsigned obj_idx);
typedef rte_mempool_obj_cb_t rte_mempool_obj_ctor_t; /* compat */



/* call obj_cb() for each mempool element */
uint32_t rte_mempool_obj_iter(struct rte_mempool *mp, rte_mempool_obj_cb_t *obj_cb, void *obj_cb_arg)



/* call mem_cb() for each mempool memory chunk */
uint32_t rte_mempool_mem_iter(struct rte_mempool *mp, rte_mempool_mem_cb_t *mem_cb, void *mem_cb_arg)

/**
 * A memory callback function for mempool.
 *
 * Used by rte_mempool_mem_iter().
 */
typedef void (rte_mempool_mem_cb_t)(struct rte_mempool *mp, void *opaque, struct rte_mempool_memhdr *memhdr, unsigned mem_idx);
```

## mbuf 相关回调函数

目前来看，mbuf中目前没有什么回调函数。

![](attachments/Pasted%20image%2020250126124936.png)


# 参考
```bash
```