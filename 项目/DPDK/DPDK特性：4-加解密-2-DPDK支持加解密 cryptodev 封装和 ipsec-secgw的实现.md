```table-of-contents
```

# DPDK中加解密的各个lib库以及框架概述
分三个部分：
加解密框架（crypto lib）
加解密设备（crypto dev）
安全协议（Security Framework）

追加的部分，ipsec （ipsec lib）
这一部分主要是对前面三个部分的整合封装调用，用来专门处理ipsec报文


## 加解密框架：cryptodev lib
API，设计思路等，都在加解密框架里。
参见文档：[cyptodev lib](https://doc.dpdk.org/guides-22.11/prog_guide/cryptodev_lib.html)

```text
The cryptodev library provides a Crypto device framework for management and provisioning of hardware and software Crypto poll mode drivers, defining generic APIs which support a number of different Crypto operations. The framework currently only supports cipher, authentication, chained cipher/authentication and AEAD symmetric and asymmetric Crypto operations.

The cryptodev library follows the same basic principles as those used in DPDK’s Ethernet Device framework. The Crypto framework provides a generic Crypto device framework which supports both physical (hardware) and virtual (software) Crypto devices as well as a generic Crypto API which allows Crypto devices to be managed and configured and supports Crypto operations to be provisioned on Crypto poll mode driver.
```

`cryptodev` 库遵循与 `DPDK 的 eth` 以太网网卡 相同的基本原则。加密框架提供了一个通用的加密设备框架，支持物理（硬件）和虚拟（软件）加密设备，以及一个通用的加密 API，允许对加密设备进行管理和配置，并支持在加密轮询模式驱动程序上配置加密操作。


### 设备的管理
#### 设备创建
**(1)物理加密设备**
物理加密设备（Physical Crypto devices）在 DPDK 初始化时执行的 EAL 函数的 PCI 探测/枚举过程中被发现，基于它们的 PCI 设备标识符，即每个唯一的 PCI BDF（总线/桥接、设备、功能「`bus/bridge, device, function`」）。特定的物理加密设备可以像 DPDK 中的其他物理设备一样，通过 EAL 命令行选项列出。

**(2)虚拟加密设备**
虚拟设备可以通过两种机制创建，分别是使用 `EAL 命令行选项`或通过应用程序直接使用 `EAL API: rte_vdev_init`。

（2.1）通过命令行使用 `–vdev EAL` 选项：
```c
--vdev  'crypto_aesni_mb0,max_nb_queue_pairs=2,socket_id=0'
```
如果 DPDK 应用程序需要多个软件加密 PMD 设备，则需要添加相应数量的 `--vdev` 及其适当的库。
一个共享相同库的加密 PMD 实例的应用程序需要唯一的 ID。示例：
```bash
--vdev  'crypto_aesni_mb0' --vdev  'crypto_aesni_mb1'
```

（2.2）在应用程序代码中使用 `rte_vdev_init` API：
```c
rte_vdev_init("crypto_aesni_mb",
                  "max_nb_queue_pairs=2,socket_id=0")

所有虚拟加密设备支持以下初始化参数：
`max_nb_queue_pairs` - 设备支持的最大队列对数量。
`socket_id` - 分配设备资源所使用的套接字。
```

#### 设备的标识
每个设备，无论是虚拟还是物理，都由两个标识符唯一标识：

- 一个唯一的设备索引用于在 `cryptodev API` 导出的所有函数中标识加密设备。
- 一个设备名称用于在控制台消息中标识加密设备，以便进行管理或调试。为了方便使用，端口名称包含端口索引。


#### 设备的配置
每个加密设备的配置包括以下操作：

- 资源分配，比如，物理加密设备可能需要一些硬件资源。
- 将设备重置为已知的默认状态。
- 初始化统计计数器。

`rte_cryptodev_configure` API 用于配置加密设备。
```c
int rte_cryptodev_configure(uint8_t dev_id,
                            struct rte_cryptodev_config *config);


struct rte_cryptodev_config {
    int socket_id;            /**< 分配资源的套接字 */
    uint16_t nb_queue_pairs;  /**< 要在设备上配置的队列对数量 */
    uint64_t ff_disable;      /**< 要禁用的功能标志。仅允许禁用以下功能：
                                 - RTE_CRYPTODEV_FF_SYMMETRIC_CRYPTO
                                 - RTE_CRYPTODEV_FF_ASYMMETRIC_CRYPTO
                                 - RTE_CRYPTODEV_FF_SECURITY
                               */
};

```

#### QP(queue pair)的配置

每个加密设备的队列对通过 `rte_cryptodev_queue_pair_setup` API 单独配置。每个队列对的资源可以在指定的套接字上分配。
```c
int rte_cryptodev_queue_pair_setup(uint8_t dev_id, uint16_t queue_pair_id,
            const struct rte_cryptodev_qp_conf *qp_conf,
            int socket_id);

struct rte_cryptodev_qp_conf {
    uint32_t nb_descriptors; /**< 每个队列对的描述符数量 */
    struct rte_mempool *mp_session;
    /**< 用于在无会话模式下创建会话的内存池 */
};

```
字段 `mp_session` 用于在无会话模式下创建临时会话以处理加密操作。它们可以是相同的内存池或不同的内存池。请注意，并非所有的 `Cryptodev PMD` 都支持无会话模式。


### 逻辑核，内核和QP的关系
多个逻辑核心不应共享同一队列对（queue pair）进行加密设备上的入队或出队操作，因为这将需要全局锁并影响性能。
```text
Multiple logical cores should never share the same queue pair for enqueuing operations or dequeuing operations on the same Crypto device since this would require global locks and hinder performance.
```
某一个OP在某个逻辑核中入QP，在另外一个逻辑核从 QP中出是可以的。
这意味着，加密操作的突发入队/出队 API 在两个不同的逻辑核中操作是允许的。


### 设备的特性和能力
加密设备通过两种机制定义其功能：全局设备特性（global device features）和算法能力（algorithm capabilities）。
#### 特备特性(Device Features)
全局设备特性识别适用于整个设备的设备级别特性，例如设备是否具有硬件加速或支持对称和/或非对称加密操作。

目前定义了以下加密设备特性：

- 对称加密操作
- 非对称加密操作
- 对称加密操作的链式处理
- SSE 加速的 SIMD 向量操作
- AVX 加速的 SIMD 向量操作
- AVX2 加速的 SIMD 向量操作
- AESNI 加速的指令
- 硬件卸载处理

```text
Currently the following Crypto device features are defined:
- Symmetric Crypto operations
- Asymmetric Crypto operations
- Chaining of symmetric Crypto operations
- SSE accelerated SIMD vector operations
- AVX accelerated SIMD vector operations
- AVX2 accelerated SIMD vector operations
- AESNI accelerated instructions
- Hardware off-load processing
```



#### 加密能力
加密能力（capabilities）用于识别加密 PMD 支持的特定算法，例如特定的对称加密密码、身份验证操作或带有关联数据的认证加密（AEAD）操作。

下面是一个示例，展示了支持身份验证算法 SHA1_HMAC 和密码算法 AES_CBC 的 PMD 的能力。
```c
static const struct rte_cryptodev_capabilities pmd_capabilities[] = {
    {    /* SHA1 HMAC */
        .op = RTE_CRYPTO_OP_TYPE_SYMMETRIC,
        .sym = {
            .xform_type = RTE_CRYPTO_SYM_XFORM_AUTH,
            .auth = {
                .algo = RTE_CRYPTO_AUTH_SHA1_HMAC,
                .block_size = 64,
                .key_size = {
                    .min = 64,
                    .max = 64,
                    .increment = 0
                },
                .digest_size = {
                    .min = 12,
                    .max = 12,
                    .increment = 0
                },
                .aad_size = { 0 },
                .iv_size = { 0 }
            }
        }
    },
    {    /* AES CBC */
        .op = RTE_CRYPTO_OP_TYPE_SYMMETRIC,
        .sym = {
            .xform_type = RTE_CRYPTO_SYM_XFORM_CIPHER,
            .cipher = {
                .algo = RTE_CRYPTO_CIPHER_AES_CBC,
                .block_size = 16,
                .key_size = {
                    .min = 16,
                    .max = 32,
                    .increment = 8
                },
                .iv_size = {
                    .min = 16,
                    .max = 16,
                    .increment = 0
                }
            }
        }
    }
}
```

### op 处理

```text
Scheduling of Crypto operations on DPDK’s application data path is performed using a burst oriented asynchronous API set. A queue pair on a Crypto device accepts a burst of Crypto operations using enqueue burst API.

On physical Crypto devices the enqueue burst API will place the operations to be processed on the devices hardware input queue, for virtual devices the processing of the Crypto operations is usually completed during the enqueue call to the Crypto device.
```
在物理加密设备上，入队突发 API (enqueue burst API)会将待处理的操作放置在设备的硬件输入队列中。而对于虚拟设备，加密操作的处理通常在调用加密设备的入队时完成。

```text
The dequeue burst API will retrieve any processed operations available from the queue pair on the Crypto device, from physical devices this is usually directly from the devices processed queue, and for virtual device’s from a `rte_ring` where processed operations are placed after being processed on the enqueue call.
```
出队突发 API 将从加密设备的队列对中检索任何可用的已处理完成的OP。
对于物理设备，这通常直接来自设备的已处理队列；
对于虚拟设备，则来自 rte_ring，其中在入队调用后，已处理的操作被放置在 rte_ring 中。

#### 私有数据
```text
For session-based operations, the set and get API provides a mechanism for an application to store and retrieve the private user data information stored along with the crypto session.

For example, suppose an application is submitting a crypto operation with a session associated and wants to indicate private user data information which is required to be used after completion of the crypto operation. In this case, the application can use the set API to set the user data and retrieve it using get API.
```
对于基于会话的操作，应用程序能够存储和检索与加密会话一起存储的私有用户数据。

假设一个应用程序提交了一个与会话相关的加密操作，并希望在加密操作完成后获取要使用的私有用户数据信息。在这种情况下，应用程序可以使用设置 API 来设置用户数据，并通过获取 API 来检索它。
```c
int rte_cryptodev_sym_session_set_user_data(
        struct rte_cryptodev_sym_session *sess, void *data, uint16_t size);


void * rte_cryptodev_sym_session_get_user_data(
        struct rte_cryptodev_sym_session *sess);
```

请注意，传递给设置 API 的大小不能大于创建会话头内存池时预定义的 `user_data_sz`，否则该函数将返回错误。此外，当在创建会话头内存池时将 `user_data_sz` 定义为 0 时，获取 API 将始终返回 NULL。

对于无会话模式，私有用户数据信息可以与 `struct rte_crypto_op` 一起放置。`rte_crypto_op::private_data_offset` 指示私有数据信息的起始位置。该偏移量是从 `rte_crypto_op` 的起始位置开始计算的，包括其他加密信息，例如 IV（因为身份验证也可能有 IV）。

#### 用户自定义回调API（callback API）
一个用户回调函数，该函数将在给定加密设备队列对上接收/发送的每个加密操作的突发时被调用(`be called for each burst of crypto ops received/sent on a given crypto device queue pair`)。

返回值是一个指针，可以在后续使用中通过移除 API 来移除该回调。
应用程序需要注册一个类型为 `rte_cryptodev_callback_fn` 的回调函数。对于给定的队列对(QP)，可以添加多个回调函数，API 不限制回调的最大数量。

**回调函数的配置时机**
应用程序注册的回调在调用 `rte_cryptodev_configure` 时不会保留，因为该函数会重新初始化回调列表。用户有责任在调用 `rte_cryptodev_configure` 之前移除所有已安装的回调，以避免可能的内存泄漏。

因此，应用程序应该在调用 `rte_cryptodev_configure` 之后添加用户回调。这些回调也可以在运行时添加。

**回调函数的运行时机**
当调用 `rte_cryptodev_enqueue_burst` 或者`rte_cryptodev_dequeue_burst` 时，这些回调将被执行。

**配置回调函数**
```c
// 入qp的回调设置
struct rte_cryptodev_cb *
        rte_cryptodev_add_enq_callback(uint8_t dev_id, uint16_t qp_id,
                                       rte_cryptodev_callback_fn cb_fn,
                                       void *cb_arg);

// 出qp的回调设置
struct rte_cryptodev_cb *
        rte_cryptodev_add_deq_callback(uint8_t dev_id, uint16_t qp_id,
                                       rte_cryptodev_callback_fn cb_fn,
                                       void *cb_arg);

// 回调函数的定义
uint16_t (* rte_cryptodev_callback_fn)(uint16_t dev_id, uint16_t qp_id,
                                       struct rte_crypto_op **ops,
                                       uint16_t nb_ops, void *user_param);
```

**移除回调函数**
```c
// 入qp的回调移除
int rte_cryptodev_remove_enq_callback(uint8_t dev_id, uint16_t qp_id,
                                      struct rte_cryptodev_cb *cb);

// 出qp的回调移除
int rte_cryptodev_remove_deq_callback(uint8_t dev_id, uint16_t qp_id,
```

#### 入队/出队突发API(burst API)
**入队**
```text
The burst enqueue API uses a Crypto device identifier and a queue pair identifier to specify the Crypto device queue pair to schedule the processing on.

```

```c
uint16_t rte_cryptodev_enqueue_burst(uint8_t dev_id, uint16_t qp_id,
                                     struct rte_crypto_op **ops, uint16_t nb_ops)

`nb_ops` 参数是要处理的操作数量，这些操作以 `rte_crypto_op` 结构的数组形式提供。

返回值：入队函数返回实际入队处理的操作数量，返回值等于 `nb_ops` 表示所有数据包都已成功入队。
```


**出队**
```c
uint16_t rte_cryptodev_dequeue_burst(uint8_t dev_id, uint16_t qp_id,
                                     struct rte_crypto_op **ops, uint16_t nb_ops)
                                     
`nb_ops` 和 `ops` 参数现在用于指定用户希望检索的最大处理操作数量，以及存储它们的位置。
返回值：API 调用返回实际返回的处理操作数量，这个数量永远不会超过 `nb_ops`。
```

#### op 表示(Operation Representation)

![](attachments/Pasted%20image%2020250113140428.png)

加密操作(`Crypto operation`)由 `rte_crypto_op` 结构表示，该结构是一个通用元数据容器，包含特定加密设备在轮询模式下所需的所有必要信息。

```text
The operation structure includes the operation type, the operation status and the session type (session-based/less), a reference to the operation specific data, which can vary in size and content depending on the operation being provisioned.

Application software is responsible for specifying all the operation specific fields in the `rte_crypto_op` structure which are then used by the Crypto PMD to process the requested operation.
```
应用程序软件负责指定 `rte_crypto_op` 结构中的所有操作特定字段，这些字段在加密 PMD 处理请求的操作时会被使用到。

#### op的管理和申请
cryptodev 库提供了一套 API，用于管理加密操作（ Crypto operations），这些操作利用mempool来分配op。

```c
extern struct rte_mempool *
rte_crypto_op_pool_create(const char *name, enum rte_crypto_op_type type,
                          unsigned nb_elts, unsigned cache_size, uint16_t priv_size,
                          int socket_id);

```

`rte_crypto_op_alloc()` 和 `rte_crypto_op_bulk_alloc()` 用于从给定的加密操作内存池(mempool)中分配特定类型的加密操作。`__rte_crypto_op_reset()` 在每个操作被返回给用户之前被调用，以确保操作在应用程序使用之前始终处于良好的已知状态。

```c
struct rte_crypto_op *rte_crypto_op_alloc(struct rte_mempool *mempool,
                                          enum rte_crypto_op_type type)

unsigned rte_crypto_op_bulk_alloc(struct rte_mempool *mempool,
                                  enum rte_crypto_op_type type,
                                  struct rte_crypto_op **ops, uint16_t nb_ops)
```

`rte_crypto_op_free()` 由应用程序调用，以将操作返回到其分配的内存池。
```c
void rte_crypto_op_free(struct rte_crypto_op *op)
```



### 对称加密

cryptodev 库目前支持以下对称加密操作：密码、身份验证，包括这些操作的链式处理，同时也支持 AEAD 操作。

如下图，展示了DPDK中的CryptoDev库的信息，其中展示了 AES-GCM 提供了加密以及验证的功能。

![](../../网络之云网络和容器网络/虚拟化/隧道/attachments/Pasted%20image%2020241127100744.png)

#### AEAD 介绍
单一的看，数据保密性通常由加密算法提供；完整性可由两类hash算法提供，即摘要算法如md5, sha-1等，另一类则是消息认证码(MAC-message authentication code)；真实性通常也由MAC提供。

认证加密（Authenticated encryption，AE）和带有关联数据的认证加密（authenticated encryption with associated data，AEAD，AE的变种）是一种能够同时保证数据的保密性、 完整性和真实性的一种加密模式。这些属性都是在一个易于使用的编程接口下提供的。
即：
**AEAD的设计目标是在单个加密操作中完成数据的加密、认证和完整性检查，以提高安全性和效率。AEAD（Authenticated Encryption with Associated Data）是一种加密模式，它结合了加密和认证的功能**。

#### session和session管理
##### session
```text
Security Sessions are created to store the immutable fields of a particular Security Association for a particular protocol which is defined by a security session configuration structure which is used in the operation processing of a packet flow.

安全会话用于存储特定协议的特定安全关联的不可变字段，这些字段由安全会话配置结构定义，并在数据包流的操作处理中使用。

Sessions are used to manage protocol specific information as well as crypto parameters. Security sessions cache this immutable data in a optimal way for the underlying PMD and this allows further acceleration of the offload of Crypto workloads.
会话用于管理协议特定信息以及加密参数。安全会话以优化的方式缓存这些不可变数据，以适应底层的 PMD，从而进一步加速加密工作负载的卸载。

```
在对称加密处理（symmetric cryptographic processing ）中，会话用于存储在加密变换（cryptographic transform）中定义的不可变数据，这些数据在数据包流的操作处理中使用。

##### session的mempool
```text
The Security framework provides APIs to create and free sessions for crypto/ethernet devices, where sessions are mempool objects. It is the application’s responsibility to create and manage two session mempools - one for session and other for session private data.

安全框架提供了 API 来为加密/以太网设备创建和释放会话，其中会话是mempool内存池对象。应用程序负责创建和管理两个会话内存池——一个用于会话，另一个用于会话私有数据。



```

应用程序必须使用 `rte_cryptodev_sym_session_pool_create()` 来创建会话内存池头和私有数据，私有数据的大小由用户通过函数中的 `elt_size` 参数指定。会话私有数据供驱动程序在加密操作期间进行初始化和访问，因此 `elt_size` 应足够大，以适应所有共享该内存池的驱动程序。要获取加密设备的正确会话私有数据大小，用户可以调用 `rte_cryptodev_sym_get_private_session_size()` 函数。

一旦会话内存池创建完成，使用 `rte_cryptodev_sym_session_create()` 从给定内存池分配和初始化会话。

当会话不再使用时，用户必须调用 `rte_cryptodev_sym_session_free()` 来反初始化会话数据并将会话返回到其所属的内存池。

```c
void * rte_cryptodev_sym_session_create(uint8_t dev_id, struct rte_crypto_sym_xform *xforms, struct rte_mempool *mp)

int rte_cryptodev_sym_session_free(uint8_t dev_id, void *_sess);

/** Cryptodev symmetric crypto session
 * Each session is derived from a fixed xform chain. Therefore each session
 * has a fixed algo, key, op-type, digest_len etc.
 */
struct rte_cryptodev_sym_session {
    RTE_MARKER cacheline0;
    uint64_t opaque_data;
    /**< Can be used for external metadata */

    uint32_t sess_data_sz;
    /**< Pointer to the user data stored after sess data */

    uint16_t user_data_sz;
    /**< Session user data will be placed after sess data */

    uint8_t driver_id;
    /**< Driver id to get the session priv */

    rte_iova_t driver_priv_data_iova;
    /**< Session driver data IOVA address */

    RTE_MARKER cacheline1 __rte_cache_min_aligned;
    /**< Second cache line - start of the driver session data */

    uint8_t driver_priv_data[0];
    /**< Driver specific session data, variable size */
};


struct rte_cryptodev_sym_session_pool_private_data {
    uint16_t sess_data_sz;
    /**< driver session data size */
    
    uint16_t user_data_sz;
    /**< session user data will be placed after sess_data */
};
```

session用来存储加密过程中的key，以及分组信息等。
用来存储这些的叫做`private session data`

![](attachments/Pasted%20image%2020250113144344.png)



#### 变换与变换链（Transforms and Transform Chaining）

![](attachments/Pasted%20image%2020250113145402.png)

对称加密变换 (`Symmetric Crypto transforms：  rte_crypto_sym_xform`) 是用于指定加密操作（Crypto operation）详细信息的机制。对于对称操作的链式处理，例如密码加密（cipher encrypt）和身份验证生成（authentication generate,），next 指针允许变换链在一起。

```text
 Crypto devices which support chaining must publish the chaining of symmetric Crypto operations feature flag.

Allocation of the xform structure is in the application domain. To allow future API extensions in a backwardly compatible manner, e.g. addition of a new parameter, the application should zero the full xform struct before populating it.
```
支持链式处理的加密设备必须发布对称加密操作链式处理功能标志。`xform` 结构的分配在应用程序域中。为了以向后兼容的方式允许未来 API 扩展，例如添加新参数，应用程序应在填充 `xform` 结构之前将其全部置为零。

**有三种变换类型**：
密码(cipher)、身份验证(authentication)和 AEAD。
还需要注意的是，传递变换的顺序指示了链式处理的顺序。

```c

/** Crypto transformation types */
enum rte_crypto_sym_xform_type {
    RTE_CRYPTO_SYM_XFORM_NOT_SPECIFIED = 0, /**< No xform specified */
    RTE_CRYPTO_SYM_XFORM_AUTH,      /**< Authentication xform */
    RTE_CRYPTO_SYM_XFORM_CIPHER,        /**< Cipher xform  */
    RTE_CRYPTO_SYM_XFORM_AEAD       /**< AEAD xform  */
};



struct rte_crypto_aead_xform {
    enum rte_crypto_aead_operation op;
    /**< AEAD operation type */
    enum rte_crypto_aead_algorithm algo;
    /**< AEAD algorithm selection */

    struct {
        const uint8_t *data;    /**< pointer to key data */
        uint16_t length;    /**< key length in bytes */
    } key;

    struct {
        uint16_t offset;
        /**< Starting point for Initialisation Vector or Counter,
         * specified as number of bytes from start of crypto
         * operation (rte_crypto_op).
         *
         * - For CCM mode, the first byte is reserved, and the
         * nonce should be written starting at &iv[1] (to allow
         * space for the implementation to write in the flags
         * in the first byte). Note that a full 16 bytes should
         * be allocated, even though the length field will
         * have a value less than this.
         *
         * - For Chacha20-Poly1305 it is 96-bit nonce.
         * PMD sets initial counter for Poly1305 key generation
         * part to 0 and for Chacha20 encryption to 1 as per
         * rfc8439 2.8. AEAD construction.
         *
         * For optimum performance, the data pointed to SHOULD
         * be 8-byte aligned.
         */
        uint16_t length;
        /**< Length of valid IV data.
         *
         * - For GCM mode, this is either:
         * 1) Number greater or equal to one, which means that IV
         *    is used and J0 will be computed internally, a minimum
         *    of 16 bytes must be allocated.
         * 2) Zero, in which case data points to J0. In this case
         *    16 bytes of J0 should be passed where J0 is defined
         *    by NIST SP800-38D.
         *
         * - For CCM mode, this is the length of the nonce,
         * which can be in the range 7 to 13 inclusive.
         *
         * - For Chacha20-Poly1305 this field is always 12.
         */
    } iv;   /**< Initialisation vector parameters */

    uint16_t digest_length;

    uint16_t aad_length;
    /**< The length of the additional authenticated data (AAD) in bytes.
     * For CCM mode, this is the length of the actual AAD, even though
     * it is required to reserve 18 bytes before the AAD and padding
     * at the end of it, so a multiple of 16 bytes is allocated.
     */
};


/**
 * Symmetric crypto transform structure.
 *
 * This is used to specify the crypto transforms required, multiple transforms
 * can be chained together to specify a chain transforms such as authentication
 * then cipher, or cipher then authentication. Each transform structure can
 * hold a single transform, the type field is used to specify which transform
 * is contained within the union
 */
/* Structure rte_crypto_sym_xform 8< */
struct rte_crypto_sym_xform {
    struct rte_crypto_sym_xform *next;
    /**< next xform in chain */
    enum rte_crypto_sym_xform_type type
    ; /**< xform type */
    RTE_STD_C11
    union {
        struct rte_crypto_auth_xform auth;
        /**< Authentication / hash xform */
        struct rte_crypto_cipher_xform cipher;
        /**< Cipher xform */
        struct rte_crypto_aead_xform aead;
        /**< AEAD xform */
    };
};
```

**限制**
API 对可以链式处理的变换数量没有限制，但这将受到处理操作的底层加密设备轮询模式驱动程序的限制。

#### 对称操作
```text
The symmetric Crypto operation structure contains all the mutable data relating to performing symmetric cryptographic processing on a referenced mbuf data buffer. It is used for either cipher, authentication, AEAD and chained operations.

As a minimum the symmetric operation must have a source data buffer (`m_src`), a valid session (or transform chain if in session-less mode) and the minimum authentication/ cipher/ AEAD parameters required depending on the type of operation specified in the session or the transform chain.
```
对称加密操作结构包含在引用的 mbuf 数据缓冲区上执行对称加密处理相关的所有可变数据。它用于密码、身份验证、AEAD 和链式操作。

对称操作必须具有源数据缓冲区 (`m_src`)、有效的会话（或如果在无会话模式下，则是变换链）以及根据会话或变换链中指定的操作类型所需的最小身份验证/密码/AEAD 参数。

```c
/**
 * Symmetric Cryptographic Operation.
 *
 * This structure contains data relating to performing symmetric cryptographic
 * processing on a referenced mbuf data buffer.
 *
 * When a symmetric crypto operation is enqueued with the device for processing
 * it must have a valid *rte_mbuf* structure attached, via m_src parameter,
 * which contains the source data which the crypto operation is to be performed
 * on.
 * While the mbuf is in use by a crypto operation no part of the mbuf should be
 * changed by the application as the device may read or write to any part of the
 * mbuf. In the case of hardware crypto devices some or all of the mbuf
 * may be DMAed in and out of the device, so writing over the original data,
 * though only the part specified by the rte_crypto_sym_op for transformation
 * will be changed.
 * Out-of-place (OOP) operation, where the source mbuf is different to the
 * destination mbuf, is a special case. Data will be copied from m_src to m_dst.
 * The part copied includes all the parts of the source mbuf that will be
 * operated on, based on the cipher.data.offset+cipher.data.length and
 * auth.data.offset+auth.data.length values in the rte_crypto_sym_op. The part
 * indicated by the cipher parameters will be transformed, any extra data around
 * this indicated by the auth parameters will be copied unchanged from source to
 * destination mbuf.
 * Also in OOP operation the cipher.data.offset and auth.data.offset apply to
 * both source and destination mbufs. As these offsets are relative to the
 * data_off parameter in each mbuf this can result in the data written to the
 * destination buffer being at a different alignment, relative to buffer start,
 * to the data in the source buffer.
 */
/* Structure rte_crypto_sym_op 8< */
struct rte_crypto_sym_op {
    struct rte_mbuf *m_src; /**< source mbuf */
    struct rte_mbuf *m_dst; /**< destination mbuf */

    RTE_STD_C11
    union {
        void *session;
        /**< Handle for the initialised crypto/security session context */

        struct rte_crypto_sym_xform *xform;
        /**< Session-less API crypto operation parameters */
    };

    RTE_STD_C11
    union {
        struct {
            struct {
                uint32_t offset;
                 /**< Starting point for AEAD processing, specified as
                  * number of bytes from start of packet in source
                  * buffer.
                  */

                uint32_t length;
                 /**< The message length, in bytes, of the source buffer
                  * on which the cryptographic operation will be
                  * computed. This must be a multiple of the block size
                  */

            } data; /**< Data offsets and length for AEAD */

            struct {
                uint8_t *data;
                /**< This points to the location where the digest result
                 * should be inserted (in the case of digest generation)
                 * or where the purported digest exists (in the case of
                 * digest verification).
                 *
                 * At session creation time, the client specified the
                 * digest result length with the digest_length member
                 * of the @ref rte_crypto_auth_xform structure. For
                 * physical crypto devices the caller must allocate at
                 * least digest_length of physically contiguous memory
                 * at this location.
                 *
                 * For digest generation, the digest result will
                 * overwrite any data at this location.
                 *
                 * @note
                 * For GCM (@ref RTE_CRYPTO_AEAD_AES_GCM), for
                 * "digest result" read "authentication tag T".
                 */

                rte_iova_t phys_addr;
                /**< Physical address of digest */

            } digest; /**< Digest parameters */

            struct {
                uint8_t *data;
                /**< Pointer to Additional Authenticated Data (AAD)
                 * needed for authenticated cipher mechanisms (CCM and
                 * GCM)
                 *
                 * Specifically for CCM (@ref RTE_CRYPTO_AEAD_AES_CCM),
                 * the caller should setup this field as follows:
                 *
                 * - the additional authentication data itself should
                 * be written starting at an offset of 18 bytes into
                 * the array, leaving room for the first block (16 bytes)
                 * and the length encoding in the first two bytes of the
                 * second block.
                 *
                 * - the array should be big enough to hold the above
                 * fields, plus any padding to round this up to the
                 * nearest multiple of the block size (16 bytes).
                 * Padding will be added by the implementation.
                 *
                 * - Note that PMDs may modify the memory reserved
                 * (first 18 bytes and the final padding).
                 *
                 * Finally, for GCM (@ref RTE_CRYPTO_AEAD_AES_GCM), the
                 * caller should setup this field as follows:
                 *
                 * - the AAD is written in starting at byte 0
                 * - the array must be big enough to hold the AAD, plus
                 * any space to round this up to the nearest multiple
                 * of the block size (16 bytes).
                 *
                 */

                rte_iova_t phys_addr;   /**< physical address */
            } aad;
            /**< Additional authentication parameters */

        } aead;

        struct {
            struct {
                struct {
                    uint32_t offset;
                     /**< Starting point for cipher processing,
                      * specified as number of bytes from start
                      * of data in the source buffer.
                      * The result of the cipher operation will be
                      * written back into the output buffer
                      * starting at this location.
                      *
                      * @note
                      * For SNOW 3G @ RTE_CRYPTO_CIPHER_SNOW3G_UEA2,
                      * KASUMI @ RTE_CRYPTO_CIPHER_KASUMI_F8
                      * and ZUC @ RTE_CRYPTO_CIPHER_ZUC_EEA3,
                      * this field should be in bits. For
                      * digest-encrypted cases this must be
                      * an 8-bit multiple.
                      */

                    uint32_t length;
                     /**< The message length, in bytes, of the
                      * source buffer on which the cryptographic
                      * operation will be computed.
                      * This is also the same as the result length.
                      * This must be a multiple of the block size
                      * or a multiple of data-unit length
                      * as described in xform.
                      *
                      * @note
                      * For SNOW 3G @ RTE_CRYPTO_AUTH_SNOW3G_UEA2,
                      * KASUMI @ RTE_CRYPTO_CIPHER_KASUMI_F8
                      * and ZUC @ RTE_CRYPTO_CIPHER_ZUC_EEA3,
                      * this field should be in bits. For
                      * digest-encrypted cases this must be
                      * an 8-bit multiple.
                      */

                } data; /**< Data offsets and length for ciphering */
            } cipher;

            struct {
                struct {
                    uint32_t offset;
                     /**< Starting point for hash processing,
                      * specified as number of bytes from start of
                      * packet in source buffer.
                      *
                      * @note
                      * For SNOW 3G @ RTE_CRYPTO_AUTH_SNOW3G_UIA2,
                      * KASUMI @ RTE_CRYPTO_AUTH_KASUMI_F9
                      * and ZUC @ RTE_CRYPTO_AUTH_ZUC_EIA3,
                      * this field should be in bits. For
                      * digest-encrypted cases this must be
                      * an 8-bit multiple.
                      *
                      * @note
                      * For KASUMI @ RTE_CRYPTO_AUTH_KASUMI_F9,
                      * this offset should be such that
                      * data to authenticate starts at COUNT.
                      *
                      * @note
                      * For DOCSIS security protocol, this
                      * offset is the DOCSIS header length
                      * and, therefore, also the CRC offset
                      * i.e. the number of bytes into the
                      * packet at which CRC calculation
                      * should begin.
                      */

                    uint32_t length;
                     /**< The message length, in bytes, of the source
                      * buffer that the hash will be computed on.
                      *
                      * @note
                      * For SNOW 3G @ RTE_CRYPTO_AUTH_SNOW3G_UIA2,
                      * KASUMI @ RTE_CRYPTO_AUTH_KASUMI_F9
                      * and ZUC @ RTE_CRYPTO_AUTH_ZUC_EIA3,
                      * this field should be in bits. For
                      * digest-encrypted cases this must be
                      * an 8-bit multiple.
                      *
                      * @note
                      * For KASUMI @ RTE_CRYPTO_AUTH_KASUMI_F9,
                      * the length should include the COUNT,
                      * FRESH, message, direction bit and padding
                      * (to be multiple of 8 bits).
                      *
                      * @note
                      * For DOCSIS security protocol, this
                      * is the CRC length i.e. the number of
                      * bytes in the packet over which the
                      * CRC should be calculated
                      */

                } data;
                /**< Data offsets and length for authentication */

                struct {
                    uint8_t *data;
                    /**< This points to the location where
                     * the digest result should be inserted
                     * (in the case of digest generation)
                     * or where the purported digest exists
                     * (in the case of digest verification).
                     *
                     * At session creation time, the client
                     * specified the digest result length with
                     * the digest_length member of the
                     * @ref rte_crypto_auth_xform structure.
                     * For physical crypto devices the caller
                     * must allocate at least digest_length of
                     * physically contiguous memory at this
                     * location.
                     *
                     * For digest generation, the digest result
                     * will overwrite any data at this location.
                     *
                     * @note
                     * Digest-encrypted case.
                     * Digest can be generated, appended to
                     * the end of raw data and encrypted
                     * together using chained digest
                     * generation
                     * (@ref RTE_CRYPTO_AUTH_OP_GENERATE)
                     * and encryption
                     * (@ref RTE_CRYPTO_CIPHER_OP_ENCRYPT)
                     * xforms. Similarly, authentication
                     * of the raw data against appended,
                     * decrypted digest, can be performed
                     * using decryption
                     * (@ref RTE_CRYPTO_CIPHER_OP_DECRYPT)
                     * and digest verification
                     * (@ref RTE_CRYPTO_AUTH_OP_VERIFY)
                     * chained xforms.
                     * To perform those operations, a few
                     * additional conditions must be met:
                     * - caller must allocate at least
                     * digest_length of memory at the end of
                     * source and (in case of out-of-place
                     * operations) destination buffer; those
                     * buffers can be linear or split using
                     * scatter-gather lists,
                     * - digest data pointer must point to
                     * the end of source or (in case of
                     * out-of-place operations) destination
                     * data, which is pointer to the
                     * data buffer + auth.data.offset +
                     * auth.data.length,
                     * - cipher.data.offset +
                     * cipher.data.length must be greater
                     * than auth.data.offset +
                     * auth.data.length and is typically
                     * equal to auth.data.offset +
                     * auth.data.length + digest_length.
                     * - for wireless algorithms, i.e.
                     * SNOW 3G, KASUMI and ZUC, as the
                     * cipher.data.length,
                     * cipher.data.offset,
                     * auth.data.length and
                     * auth.data.offset are in bits, they
                     * must be 8-bit multiples.
                     *
                     * Note, that for security reasons, it
                     * is PMDs' responsibility to not
                     * leave an unencrypted digest in any
                     * buffer after performing auth-cipher
                     * operations.
                     *
                     */

                    rte_iova_t phys_addr;
                    /**< Physical address of digest */
                    
                } digest; /**< Digest parameters */
            } auth;
        };
    };
};
/* >8 End of structure rte_crypto_sym_op. */

```

### 同步模式(Synchronous mode)
一些 cryptodev 支持同步模式，同时也支持标准的异步模式。在同步模式下，操作是在调用 `rte_cryptodev_sym_cpu_crypto_process` 方法时直接执行，而不是在此之前进行入队和出队。此操作模式使得利用 CPU 加密加速的 `cryptodev` 在性能上相比标准异步方法有显著提升。支持同步模式的 `cryptodev` 会设置特性标志 `RTE_CRYPTODEV_FF_SYM_CPU_CRYPTO`。

要执行同步操作，必须调用 `rte_cryptodev_sym_cpu_crypto_process`


### 范例
#### 背景
有多种示例应用程序展示了如何使用 cryptodev 库，例如 L2fwd with Crypto 示例应用程序（L2fwd-crypto）和 IPsec 安全网关应用程序（ipsec-secgw）。

虽然这些应用程序演示了如何创建一个执行通用加密操作的应用程序，但所需的复杂性掩盖了使用 cryptodev API 的基本步骤。

以下示例代码展示了使用 DPDK 中可用的一个加密 PMD，通过 AES-CBC 加密多个缓冲区的基本步骤（尽管执行其他加密操作的过程类似）。

#### 主要代码
```c
/*
 * Simple example to encrypt several buffers with AES-CBC using
 * the Cryptodev APIs.
 */

#define MAX_SESSIONS         1024
#define NUM_MBUFS            1024
#define POOL_CACHE_SIZE      128
#define BURST_SIZE           32
#define BUFFER_SIZE          1024
#define AES_CBC_IV_LENGTH    16
#define AES_CBC_KEY_LENGTH   16
#define IV_OFFSET            (sizeof(struct rte_crypto_op) + \
                             sizeof(struct rte_crypto_sym_op))

struct rte_mempool *mbuf_pool, *crypto_op_pool;
struct rte_mempool *session_pool, *session_priv_pool;
unsigned int session_size;
int ret;

/* Initialize EAL. */
ret = rte_eal_init(argc, argv);
if (ret < 0)
    rte_exit(EXIT_FAILURE, "Invalid EAL arguments\n");

uint8_t socket_id = rte_socket_id();

/* Create the mbuf pool. 
    mbuf 的 mempool的创建
*/
mbuf_pool = rte_pktmbuf_pool_create("mbuf_pool",
                                NUM_MBUFS,
                                POOL_CACHE_SIZE,
                                0,
                                RTE_MBUF_DEFAULT_BUF_SIZE,
                                socket_id);
if (mbuf_pool == NULL)
    rte_exit(EXIT_FAILURE, "Cannot create mbuf pool\n");

/*
 * The IV is always placed after the crypto operation,
 * so some private data is required to be reserved.
 */
unsigned int crypto_op_private_data = AES_CBC_IV_LENGTH;

/* Create crypto operation pool.
    crypto op mempool的创建
 */
crypto_op_pool = rte_crypto_op_pool_create("crypto_op_pool",
                                        RTE_CRYPTO_OP_TYPE_SYMMETRIC,
                                        NUM_MBUFS,
                                        POOL_CACHE_SIZE,
                                        crypto_op_private_data,
                                        socket_id);
if (crypto_op_pool == NULL)
    rte_exit(EXIT_FAILURE, "Cannot create crypto op pool\n");

/* Create the virtual crypto device.
    虚拟设备的创建
 */
char args[128];
const char *crypto_name = "crypto_aesni_mb0";
snprintf(args, sizeof(args), "socket_id=%d", socket_id);
ret = rte_vdev_init(crypto_name, args);
if (ret != 0)
    rte_exit(EXIT_FAILURE, "Cannot create virtual device");

uint8_t cdev_id = rte_cryptodev_get_dev_id(crypto_name);

/* Get private session data size. */
session_size = rte_cryptodev_sym_get_private_session_size(cdev_id);

#ifdef USE_TWO_MEMPOOLS
/* session 以及 session priv data 使用2个 mempool */
/* Create session mempool for the session header.
    sym session mempool的创建
 */
session_pool = rte_cryptodev_sym_session_pool_create("session_pool",
                                MAX_SESSIONS,
                                0,
                                POOL_CACHE_SIZE,
                                0,
                                socket_id);

/*
 * Create session private data mempool for the
 * private session data for the crypto device.
  session priv data mempool的创建
 */
session_priv_pool = rte_mempool_create("session_pool",
                                MAX_SESSIONS,
                                session_size,
                                POOL_CACHE_SIZE,
                                0, NULL, NULL, NULL,
                                NULL, socket_id,
                                0);

#else
/* session 以及 session priv data 使用1个 mempool */
/* Use of the same mempool for session header and private data */

    session_pool = rte_cryptodev_sym_session_pool_create("session_pool",
                                MAX_SESSIONS * 2,
                                session_size,
                                POOL_CACHE_SIZE,
                                0,
                                socket_id);

    session_priv_pool = session_pool;

#endif

/* Configure the crypto device. */
struct rte_cryptodev_config conf = {
    .nb_queue_pairs = 1,
    .socket_id = socket_id
};

struct rte_cryptodev_qp_conf qp_conf = {
    .nb_descriptors = 2048,
    .mp_session = session_pool,
    .mp_session_private = session_priv_pool
};

if (rte_cryptodev_configure(cdev_id, &conf) < 0)
    rte_exit(EXIT_FAILURE, "Failed to configure cryptodev %u", cdev_id);

if (rte_cryptodev_queue_pair_setup(cdev_id, 0, &qp_conf, socket_id) < 0)
    rte_exit(EXIT_FAILURE, "Failed to setup queue pair\n");

if (rte_cryptodev_start(cdev_id) < 0)
    rte_exit(EXIT_FAILURE, "Failed to start device\n");

/* Create the crypto transform. */
uint8_t cipher_key[16] = {0};
struct rte_crypto_sym_xform cipher_xform = {
    .next = NULL,
    .type = RTE_CRYPTO_SYM_XFORM_CIPHER,
    .cipher = {
        .op = RTE_CRYPTO_CIPHER_OP_ENCRYPT,
        .algo = RTE_CRYPTO_CIPHER_AES_CBC,
        .key = {
            .data = cipher_key,
            .length = AES_CBC_KEY_LENGTH
        },
        .iv = {
            .offset = IV_OFFSET,
            .length = AES_CBC_IV_LENGTH
        }
    }
};

/* Create crypto session and initialize it for the crypto device. 
    获取一个 crypto session
*/
struct rte_cryptodev_sym_session *session;
session = rte_cryptodev_sym_session_create(cdev_id, &cipher_xform,
                session_pool);
if (session == NULL)
    rte_exit(EXIT_FAILURE, "Session could not be created\n");

/* Get a burst of crypto operations. 
    获取 多个 crypto ops
*/
struct rte_crypto_op *crypto_ops[BURST_SIZE];
if (rte_crypto_op_bulk_alloc(crypto_op_pool,
                        RTE_CRYPTO_OP_TYPE_SYMMETRIC,
                        crypto_ops, BURST_SIZE) == 0)
    rte_exit(EXIT_FAILURE, "Not enough crypto operations available\n");

/* Get a burst of mbufs. 
    获取多个mbuf
*/
struct rte_mbuf *mbufs[BURST_SIZE];
if (rte_pktmbuf_alloc_bulk(mbuf_pool, mbufs, BURST_SIZE) < 0)
    rte_exit(EXIT_FAILURE, "Not enough mbufs available");

/* Initialize the mbufs and append them to the crypto operations. 
    初始化 mbuf，并设置 cypto op的src为mbuf
*/
unsigned int i;
for (i = 0; i < BURST_SIZE; i++) {
    if (rte_pktmbuf_append(mbufs[i], BUFFER_SIZE) == NULL)
        rte_exit(EXIT_FAILURE, "Not enough room in the mbuf\n");
    crypto_ops[i]->sym->m_src = mbufs[i];
}

/* Set up the crypto operations.
    设置 crypto op的其他部分
 */
for (i = 0; i < BURST_SIZE; i++) {
    struct rte_crypto_op *op = crypto_ops[i];
    /* Modify bytes of the IV at the end of the crypto operation */
    uint8_t *iv_ptr = rte_crypto_op_ctod_offset(op, uint8_t *,
                                            IV_OFFSET);

    generate_random_bytes(iv_ptr, AES_CBC_IV_LENGTH);

    op->sym->cipher.data.offset = 0;
    op->sym->cipher.data.length = BUFFER_SIZE;

    /* Attach the crypto session to the operation */
    rte_crypto_op_attach_sym_session(op, session);
}

/* Enqueue the crypto operations in the crypto device.
    crypto op 进入到 queue pair
 */
uint16_t num_enqueued_ops = rte_cryptodev_enqueue_burst(cdev_id, 0,
                                        crypto_ops, BURST_SIZE);

/*
 * Dequeue the crypto operations until all the operations
 * are processed in the crypto device.
    crypto ops 从 queue pair 中出队
 */
uint16_t num_dequeued_ops, total_num_dequeued_ops = 0;
do {
    struct rte_crypto_op *dequeued_ops[BURST_SIZE];
    num_dequeued_ops = rte_cryptodev_dequeue_burst(cdev_id, 0,
                                    dequeued_ops, BURST_SIZE);
    total_num_dequeued_ops += num_dequeued_ops;

    /* Check if operation was processed successfully 
        检查 crypto op是否成功处理
    */
    for (i = 0; i < num_dequeued_ops; i++) {
        if (dequeued_ops[i]->status != RTE_CRYPTO_OP_STATUS_SUCCESS)
            rte_exit(EXIT_FAILURE,
                    "Some operations were not processed correctly");
    }

    rte_mempool_put_bulk(crypto_op_pool, (void **)dequeued_ops,
                                        num_dequeued_ops);
} while (total_num_dequeued_ops < num_enqueued_ops);
```

## 加解密设备：cryptodev
设备层的事情，加解密设备的分类，调度，主备等。
参见：
[cryptodev driver](https://doc.dpdk.org/guides-22.11/cryptodevs/index.html)

### 加密设备分类
所有的加解密设备大概分为以下几种：openssl，null，硬件架构相关的设备。
null为纯软件的最小实现，可以用来调试等。

硬件相关的主要包括，Intel，arm，NXP，AMD，QAT卡等。

#### 不同设备的特性(Feature Flags）

不同的设备，所支持的加解密算法也各有不同。如下所示：
参考：[ Crypto Device Supported Functionality Matrices](https://doc.dpdk.org/guides-22.11/cryptodevs/overview.html)



#### 不同设备的加密算法(Cipher Algorithms)

![](attachments/Pasted%20image%2020250113181135.png)

#### 不同设备的认证算法(Authentication Algorithms)

![](attachments/Pasted%20image%2020250113193259.png)

#### 不同设备的AEAD算法(AEAD Algorithms)

![](attachments/Pasted%20image%2020250113193333.png)

#### 不同设备的非对称加密算法(Asymmetric Algorithms)

![](attachments/Pasted%20image%2020250113193407.png)

### 调度设备（Cryptodev Scheduler）

![](attachments/Pasted%20image%2020250113110049.png)

在各设备的上层，还有一个调度设备，用于管理和协调多个加解密设备直接的数据流。叫做 `cryptodev scheduler PMD（librte_crypto_scheduler）`。

调度器 PMD（Scheduler PMD） 是一个软件加密 PMD，具有连接硬件和/或软件加密设备的能力，并以特定方式在它们之间分配入口加密操作。

加密设备调度器 PMD 库（librte_crypto_scheduler）充当软件加密 PMD，并共享 `librte_cryptodev` 提供的相同 API。该 PMD 支持将多个加密 PMD（软件或硬件）作为工作线程连接，并以特定行为将加密工作负载分配给它们。这些行为被分类为不同的“模式”。基本上，调度模式定义了将加密操作调度到其工作线程的特定动作。

`librte_crypto_scheduler` 库导出一个 `C API`，提供用于连接/断开工作线程、设置/获取调度模式以及启用/禁用加密操作重排序的 API。

#### 初始化
要在应用程序中使用 PMD，用户必须：
在应用程序中调用 `rte_vdev_init("crypto_scheduler")`。
在 EAL 选项中使用 `–vdev="crypto_scheduler"`，这将内部调用 `rte_vdev_init()`。

```bash
... --vdev "crypto_aesni_mb0,name=aesni_mb_1" --vdev "crypto_aesni_mb1,name=aesni_mb_2" --vdev "crypto_scheduler,worker=aesni_mb_1,worker=aesni_mb_2" ...
```

参数说明：
```
`worker`：如果某个加密设备已使用特定名称初始化，可以通过此参数将其连接到调度器，只需在此填写名称即可。可以通过多次提供此参数来初始连接多个加密设备。

`ordering`：指定加密操作重排序功能的状态。该参数的值可以为“enable”或“disable”。默认情况下，此功能是禁用的。
加密操作重排序功能需要使用每个待处理 mbuf 的 userdata 字段来存储临时数据。在处理结束时，该字段被设置为指向 NULL，任何之前存储在此字段中的值将会丢失。
```

### crypto dev PMD
#### AES-NI Multi Buffer crypto PMD
AES-NI 的全称是 **Advanced Encryption Standard New Instructions**。这是英特尔和其他处理器制造商为其处理器架构设计的一组指令，旨在加速 AES（高级加密标准）算法的加密和解密过程。通过使用这些指令，软件可以更高效地执行 AES 加密操作，从而提高性能和安全性。

AESNI MB PMD（librte_crypto_aesni_mb）提供了轮询模式（poll mode）加密驱动(crypto driver)程序的支持，以利用英特尔多缓冲库(Intel multi buffer library).

AESNI MB PMD 是一个虚拟加密设备的 PMD。

#####  Intel Multi-Buffer 技术

Intel Multi-buffer 基本原理就是使用CPU的`SIMD`机制，通过 `AVX-512` 指令集并行处理数据，来提升对称加密/非对称加密算法性能。

`Multi-Buffer`使用`AVX-512`指令同时处理多个独立的缓冲区，既可以在一个执行周期内同时执行多个加解密操作，加解密的执行效率便会得到成倍的提升。

##### Intel IPsec MB介绍
为使用`Multi-Buffer`技术，`intel`提供了软件库。
`Intel IPsec MB`是一个专为加速包处理应用设计的**软件加密库**，其支持包括`IPsec`、`TLS`、无线通信（`RAN`）等多种场景。

这个库在GitHub上开源，并被集成到 `DPDK`、`Intel(R) QAT Engine`和`FD.io`等框架中，为用户提供**灵活的软件加密解决方案**。

##### Intel IPsec MB软件库在DPDK中的集成
`dpdk`中的  `intel-ipsec-mb` 库 `ipsec_mb` ，`ipsec_mb` 下还有多个`pmd`，对应不同的算法。不同pmd类型相关的数据会保存到 `ipsec_mb_pmds` 变量中，公共函数根据当前的`pmd`类型获取对应`pmd`的数据和函数处理数据。实现不同pmd的分离。

##### 特性
AESNI MB PMD has support for:

**（1）Cipher algorithms**:
- RTE_CRYPTO_CIPHER_AES128_CBC
- RTE_CRYPTO_CIPHER_AES192_CBC
- RTE_CRYPTO_CIPHER_AES256_CBC
- RTE_CRYPTO_CIPHER_AES128_CTR
- RTE_CRYPTO_CIPHER_AES192_CTR
- RTE_CRYPTO_CIPHER_AES256_CTR
- RTE_CRYPTO_CIPHER_AES_DOCSISBPI
- RTE_CRYPTO_CIPHER_DES_CBC
- RTE_CRYPTO_CIPHER_3DES_CBC
- RTE_CRYPTO_CIPHER_DES_DOCSISBPI
- RTE_CRYPTO_CIPHER_AES128_ECB
- RTE_CRYPTO_CIPHER_AES192_ECB
- RTE_CRYPTO_CIPHER_AES256_ECB
- RTE_CRYPTO_CIPHER_ZUC_EEA3
- RTE_CRYPTO_CIPHER_SNOW3G_UEA2
- RTE_CRYPTO_CIPHER_KASUMI_F8
    

**（2）Hash algorithms**:
- RTE_CRYPTO_AUTH_MD5_HMAC
- RTE_CRYPTO_AUTH_SHA1_HMAC
- RTE_CRYPTO_AUTH_SHA224_HMAC
- RTE_CRYPTO_AUTH_SHA256_HMAC
- RTE_CRYPTO_AUTH_SHA384_HMAC
- RTE_CRYPTO_AUTH_SHA512_HMAC
- RTE_CRYPTO_AUTH_AES_XCBC_HMAC
- RTE_CRYPTO_AUTH_AES_CMAC
- RTE_CRYPTO_AUTH_AES_GMAC
- RTE_CRYPTO_AUTH_SHA1
- RTE_CRYPTO_AUTH_SHA224
- RTE_CRYPTO_AUTH_SHA256
- RTE_CRYPTO_AUTH_SHA384
- RTE_CRYPTO_AUTH_SHA512
- RTE_CRYPTO_AUTH_ZUC_EIA3
- RTE_CRYPTO_AUTH_SNOW3G_UIA2
- RTE_CRYPTO_AUTH_KASUMI_F9

**（3）AEAD algorithms**:
- RTE_CRYPTO_AEAD_AES_CCM
- RTE_CRYPTO_AEAD_AES_GCM
- RTE_CRYPTO_AEAD_CHACHA20_POLY1305
    

**（4）Protocol offloads**:
- RTE_SECURITY_PROTOCOL_DOCSIS

##### 安装
**multi-buffer library 安装**
要构建带有 AESNI_MB_PMD 的 DPDK，用户需要下载多缓冲库（multi-buffer library），并在构建 DPDK 之前在用户系统上编译它。
该 PMD （AESNI MB PMD）支持的最新版本库为 v1.3，可以从 [intel-ipsec-mb-v1.3.zip](https://github.com/01org/intel-ipsec-mb/archive/v1.3.zip) 下载。

```bash
make
make install

注：当 GCC 版本低于 5.0 且库版本小于等于 v0.53 时，多缓冲库的编译会出现问题。
```

**NASM安装**
该库需要 NASM 进行构建。根据库的版本，可能需要最低的 NASM 版本（例如，v0.54 至少需要 NASM 2.14）。

NASM 为不同的操作系统提供了打包。然而，在某些操作系统上，版本可能过旧，因此需要手动安装。在这种情况下，可以从 [NASM 网站](https://www.nasm.us/pub/nasm/releasebuilds/?C=M;O=D)下载 NASM。下载后，解压并按照以下步骤操作：
```
./configure
make
make install
```

##### DPDK版本和 multi-buffer lib的版本关系

![](attachments/Pasted%20image%2020250113195327.png)

##### 初始化

![](attachments/Pasted%20image%2020250113195643.png)

#### AES-NI GCM crypto PMD
AES-GCM的全称是“Advanced Encryption Standard Galois/Counter Mode”。它是一种对称加密算法，结合了高级加密标准（AES）和Galois/Counter模式（GCM）。

AES-NI GCM PMD（librte_crypto_aesni_gcm）提供了轮询模式加密驱动程序的支持，以利用 Intel 多缓冲库（有关更多信息，包括安装，请参见 AES-NI Multi-buffer PMD 文档）。

AES-NI GCM PMD 支持同步操作模式，通过 `rte_cryptodev_sym_cpu_crypto_process` 函数调用来处理 AES-GCM 和 GMAC，但 GMAC 的支持限制为每次操作一个段。

##### 特性
AESNI GCM PMD 支持以下功能：

**认证算法（Authentication algorithms）：**
- RTE_CRYPTO_AUTH_AES_GMAC

**AEAD 算法（AEAD algorithms）：**
- RTE_CRYPTO_AEAD_AES_GCM

##### 初始化

![](attachments/Pasted%20image%2020250113200113.png)

#### intel QAT crypto pmd 
QAT (Quick Assist Technology) 是 Intel 提供的一种硬件加速技术，用于加强加密和解密操作的性能。

参见：DPDK支持的软硬件加解密

#### MLX5 crypto driver
```text
The MLX5 crypto driver library (librte_crypto_mlx5) provides support for NVIDIA ConnectX-6 family adapters.


```
MLX5 加密驱动库（`librte_crypto_mlx5`）支持 `NVIDIA ConnectX-6` 系列适配器。

注：此中叫做 MLX5 crypto driver，而不是 MLX5 crypto PMD。


## 加解密协议：rte_security
 `ESP`，`PDCP`等网络加密协议相关的事情，主要功能是整合`NIC pmd`和`Crypto pmd`。
 参见：[rte_security](http://doc.dpdk.org/guides-22.11/prog_guide/rte_security.html)


安全库（ security library）提供了一个框架，用于管理和配置卸载到硬件设备上的安全协议操作（security protocol operations）。该库定义了通用 API，用于创建和释放安全会话（security sessions），这些会话可以支持完整的协议卸载以及与网络接口卡（NIC）或加密设备的内联加密操作（ inline crypto operation）。该框架目前仅支持 IPsec、PDCP 等协议及相关操作，未来将添加其他协议。

安全库为现有的加密设备（crypto device）和/或以太网设备（ethernet device.）提供了额外的卸载能力。

![](attachments/Pasted%20image%2020250113152902.png)

注：目前（DPDK 22.11），安全库不支持多进程的情况。未来的版本将对此进行更新。


### 内联加密(Inline Crypto)
#### 配置
```text
RTE_SECURITY_ACTION_TYPE_INLINE_CRYPTO: 
	The crypto processing for security protocol (e.g. IPsec) is processed inline during receive and transmission on NIC port. The flow based security action should be configured on the port.
```
安全协议（例如 IPsec）的加解密处理在 NIC 端口的接收和传输过程中以内联方式进行。应在端口上配置基于流的安全操作。

![](attachments/Pasted%20image%2020250113155215.png)

#### 收包

数据包在 RX 路径中被解密，并在 Rx 描述符中设置相关的加密状态。
经过成功的内联加密处理后，数据包作为常规的 Rx 数据包呈现给主机，但所有与安全协议相关的头部仍然附加在数据包上。
例如，在 IPsec 的情况下，IPsec 隧道头（如果有）、ESP/AH 头将保留在数据包中，但接收到的数据包包含解密后的数据，而在数据包到达时是加密的数据。
驱动程序的 Rx 路径检查描述符，并根据加密状态在 `rte_mbuf.ol_flags` 字段中设置附加标志。

**注意**
底层设备可能不支持对所有匹配特定流的入口数据包进行加密处理（例如，分片数据包），此类数据包将作为加密数据包传递。应用程序有责任使用其他加密驱动实例处理此类加密数据包。

#### 发包
 软件通过添加相关的安全协议头来准备出口数据包。只有数据不会被软件加密。驱动程序将相应地配置 tx 描述符。硬件设备将在发送数据包之前对数据进行加密。
 
**注意**
底层设备可能支持后加密的 TSO（TCP Segmentation Offload）。

### 内联协议卸载(Inline protocol offload)
#### 配置
```text
RTE_SECURITY_ACTION_TYPE_INLINE_PROTOCOL: 
	The crypto and protocol processing for security protocol (e.g. IPsec) is processed inline during receive and transmission. The flow based security action should be configured on the port.
```

安全协议（例如 IPsec）的加密和协议处理在接收和传输过程中以内联方式进行。应在端口上配置基于流的安全操作。

![](attachments/Pasted%20image%2020250113160244.png)

#### 收包
数据包在 RX 路径中被解密，并在 Rx 描述符中设置相关的加密状态。
经过成功的内联加密处理后，数据包作为常规的 Rx 数据包呈现给主机，但所有与安全协议相关的头部可以选择性地从数据包中移除。
例如，在 IPsec 的情况下，IPsec 隧道头（如果有）、ESP/AH 头将从数据包中移除，接收到的数据包将仅包含解密后的数据。
驱动程序的 Rx 路径检查描述符，并根据加密状态在 `rte_mbuf.ol_flags` 字段中设置附加标志。
驱动程序还将在 `RTE_SECURITY_DYNFIELD_NAME` 字段中设置特定于设备的元数据。这将允许应用程序识别数据包上执行的安全处理。

**注意**
在这种情况下，底层设备是有状态的。预计该设备将支持对所有匹配给定流的各种数据包进行加密处理，包括分片数据包（重新组装后）。
例如，在 IPsec 的情况下，设备可能会内部管理防重放等功能。它将提供防重放行为的配置选项，即丢弃数据包或将其传递给驱动程序，并在描述符中设置错误标志。

#### 发包
软件将发送没有任何安全协议头的数据包。驱动程序将配置 tx 描述符中的安全索引和其他要求。
硬件设备将对数据包进行安全处理，包括添加相关的协议头并在发送数据包之前对数据进行加密。
软件应确保缓冲区具有所需的头部和尾部空间，以便添加任何协议头。
如果预期生成的数据包将超过 MTU 大小，软件还应进行早期分片。
软件还应确保 L2 头部内容更新为最终的 L2 头部，预期在 IPsec 处理后，因为 IPsec 卸载将仅更新出口路径中的 L3 及以上。

**注意**
底层设备将管理出口处理所需的状态信息。例如，在 IPsec 的情况下，序列号将被添加到数据包中，但设备应提供指示，表明序列号即将溢出。底层设备可能支持后加密的 TSO。

### 旁路协议卸载(Lookaside protocol offload)
#### 配置
```text
RTE_SECURITY_ACTION_TYPE_LOOKASIDE_PROTOCOL: 
	This extends librte_cryptodev to support the programming of IPsec Security Association (SA) as part of a crypto session creation including the definition. In addition to standard crypto processing, as defined by the cryptodev, the security protocol processing is also offloaded to the crypto device.
```
这扩展了 `librte_cryptodev`，以支持在加密会话创建过程中编程 IPsec 安全关联（SA）。除了由 `cryptodev` 定义的标准**加密处理**外，安全**协议处理**也被卸载到加密设备上。类似于 内联协议卸载(Inline protocol offload)，不过是旁路的。

![](attachments/Pasted%20image%2020250113162036.png)

#### 解密

数据包被发送到加密设备进行安全协议处理。设备将解密数据包，并可选择性地从数据包中移除额外的安全头部。例如，在 IPsec 的情况下，IPsec 隧道头（如果有）和 ESP/AH 头将从数据包中移除，解密后的数据包可能仅包含明文数据。

**注意**
在 IPsec 的情况下，设备可能会内部管理防重放等功能。它将提供防重放行为的配置选项，即丢弃数据包或将其传递给驱动程序，并在描述符中设置错误标志。

#### 加密
软件将像往常一样将数据包提交给 `cryptodev` 进行加密，此时硬件设备还将添加相关的安全协议头，并对数据包进行加密。软件应确保缓冲区有足够的头部和尾部空间以便添加任何协议头。

**注意**
在 IPsec 的情况下，序列号将被添加到数据包中，并应提供指示以表明序列号即将溢出。

#### Lookaside Offload 和 Inline Offload 对比
`lookaside crypto offload` 是存在专门的加解密硬件，而不是内嵌到网卡中，`intel的 QAT` 就是这种类型。
而 `Inline crypto Offload` 是将 `crypto` 功能内嵌到网卡中。
一般而言，  `Inline crypto Offload` 由于内嵌到网卡中，适用于标准的`ipsec`协议。`lookaside crypto offload` 支持的功能更加完善，可以适用于自定义的加解密协议。

## 前面三个部分的封装整合调用：ipsec lib

![](attachments/Pasted%20image%2020250112154708.png)

这一部分主要是对前面三个部分的整合封装调用，用来专门处理ipsec报文。

参见：[ipsec_lib](https://doc.dpdk.org/guides-22.11/prog_guide/ipsec_lib.html)

DPDK 提供了一个用于 IPsec 数据路径处理的库。该库利用现有的 DPDK 加密设备（crypto-dev）和安全 API（security API），为应用程序提供透明且高性能的 IPsec 数据包处理 API。该库专注于数据路径协议处理（ESP 和 AH），而 IKE 协议的实现不在该库的范围之内。

### SA level API
```text
This API operates on the IPsec Security Association (SA) level. It provides functionality that allows user for given SA to process inbound and outbound IPsec packets.
```
该 API 在 IPsec 安全关联（SA）级别上操作。它提供了功能，允许用户针对给定的 SA 处理入站(inbound)和出站(outbound)的 IPsec 数据包。

![](attachments/Pasted%20image%2020250113170838.png)

**inbound**：即 收到的是解密的数据包，需要进行解密。

**outbound**：即收到的是加密前的数据包，需要进行加密。

```text
Due to the nature of the crypto-dev API (enqueue/dequeue model), the library introduces an asynchronous API for IPsec packets destined to be processed by the crypto-device.

由于加密设备 API 的特性（入队/出队模型），该库为目标处理的 IPsec 数据包引入了异步 API。
```

#### RTE_SECURITY ACTION TYPE
当前实现支持所有四种当前定义的 rte_security 类型：
```c
RTE_SECURITY_ACTION_TYPE_NONE
RTE_SECURITY_ACTION_TYPE_CPU_CRYPTO
RTE_SECURITY_ACTION_TYPE_INLINE_CRYPTO
RTE_SECURITY_ACTION_TYPE_INLINE_PROTOCOL
RTE_SECURITY_ACTION_TYPE_LOOKASIDE_PROTOCOL
```

##### RTE_SECURITY_ACTION_TYPE_NONE

![](attachments/Pasted%20image%2020250113171718.png)


在该模式下，库函数执行以下操作：

**(1)对于inbound数据包**：

- 检查 SQN
- 为每个输入数据包准备 `rte_crypto_op` 结构
- 验证由加密设备执行的完整性检查和解密是否成功完成
- 检查填充数据
- 移除外部 IP 头（隧道模式）/更新 IP 头（传输模式）
- 移除 ESP 头和尾、填充、IV 和 ICV 数据
- 更新 SA 重放窗口

**(2) 对于outbound数据包**：

- 生成 SQN 和 IV
- 添加外部 IP 头（隧道模式）/更新 IP 头（传输模式）
- 添加 ESP 头和尾、填充和 IV 数据
- 为每个输入数据包准备 `rte_crypto_op` 结构
- 验证加密设备操作（加密、ICV 生成）是否成功完成


##### RTE_SECURITY_ACTION_TYPE_CPU_CRYPTO
在该模式下，库函数执行与 RTE_SECURITY_ACTION_TYPE_NONE 相同的操作。唯一的区别是，`crypto operations` 使用 CPU 加密同步 API 执行。


##### RTE_SECURITY_ACTION_TYPE_INLINE_CRYPTO

![](attachments/Pasted%20image%2020250113172351.png)

在该模式下，库函数执行：

**(1)对于inbound数据包**：

- 验证由 `rte_security` 设备执行的完整性检查和解密是否成功完成
- 检查 SQN
- 检查填充数据
- 移除外部 IP 头（隧道模式）/更新 IP 头（传输模式）
- 移除 ESP 头和尾、填充、IV 和 ICV 数据
- 更新 SA 重放窗口

**(2) 对于outbound数据包**：

- 生成 SQN 和 IV
- 添加外部 IP 头（隧道模式）/更新 IP 头（传输模式）
- 添加 ESP 头和尾、填充和 IV 数据
- 更新 `struct rte_mbuf` 中的 `ol_flags`，以指示此数据包必须由硬件执行内联加密处理
- 调用特定于 `rte_security` 设备的 `set_pkt_metadata()`，以将安全设备特定数据与数据包关联

##### RTE_SECURITY_ACTION_TYPE_INLINE_PROTOCOL
![](attachments/Pasted%20image%2020250113172623.png)

在该模式下，库函数执行：

**(1)对于inbound数据包**：
- 验证由 `rte_security` 设备执行的完整性检查和解密是否成功完成

**(2) 对于outbound数据包**：
- 更新 `struct rte_mbuf` 中的 `ol_flags`，以指示此数据包必须由硬件执行内联加密处理
- 调用特定于 `rte_security` 设备的 `set_pkt_metadata()`，以将安全设备特定数据与数据包关联

##### RTE_SECURITY_ACTION_TYPE_LOOKASIDE_PROTOCOL

![](attachments/Pasted%20image%2020250113172802.png)

在该模式下，库函数执行：

**(1)对于inbound数据包**：

- 为每个输入数据包准备 `rte_crypto_op` 结构
- 验证由加密设备执行的完整性检查和解密是否成功完成

**(2) 对于outbound数据包**：

- 为每个输入数据包准备 `rte_crypto_op` 结构
- 验证加密设备操作（加密、ICV 生成）是否成功完成

#### SA的标识
根据 RFC4301，每个 SA 可以通过以下键唯一标识：
- 安全参数索引（SPI： security parameter index）
- 或 SPI 和目标 IP（DIP）
- 或 SPI、DIP 和源 IP（SIP）

在多个匹配的情况下，将返回最长匹配的键。

### SA database API

SA 数据库（SAD）是一个包含 <key, value> 对的表。
value 是一个不透明的用户提供的指针，指向用户定义的 SA 数据结构。

#### Create/destroy
要创建 SAD 表，用户必须指定所需的每种键类型的条目数量和 IP 协议类型（IPv4/IPv6）。例如：
```c
struct rte_ipsec_sad *sad;
struct rte_ipsec_sad_conf conf;

conf.socket_id = -1;
conf.max_sa[RTE_IPSEC_SAD_SPI_ONLY] = some_nb_rules_spi_only;
conf.max_sa[RTE_IPSEC_SAD_SPI_DIP] = some_nb_rules_spi_dip;
conf.max_sa[RTE_IPSEC_SAD_SPI_DIP_SIP] = some_nb_rules_spi_dip_sip;
conf.flags = RTE_IPSEC_SAD_FLAG_RW_CONCURRENCY;

sad = rte_ipsec_sad_create("test", &conf);
```
#### Add/delete rules

![](attachments/Pasted%20image%2020250113173549.png)

#### Lookup

![](attachments/Pasted%20image%2020250113173659.png)



### 支持的特性
```text
- ESP protocol tunnel mode both IPv4/IPv6.
- ESP protocol transport mode both IPv4/IPv6.
- ESN and replay window.
- NAT-T / UDP encapsulated ESP.
- TSO (only for inline crypto mode)
- algorithms: 3DES-CBC, AES-CBC, AES-CTR, AES-GCM, AES_CCM, CHACHA20_POLY1305, AES_GMAC, HMAC-SHA1, NULL.
```

- ESP 协议隧道模式，支持 IPv4/IPv6。
- ESP 协议传输模式，支持 IPv4/IPv6。
- ESN 和重放窗口。
- NAT-T / UDP 封装的 ESP。
- TSO（仅适用于内联加密模式）。
- 支持的算法：3DES-CBC、AES-CBC、AES-CTR、AES-GCM、AES_CCM、CHACHA20_POLY1305、AES_GMAC、HMAC-SHA1、NULL。

### 限制
```text
The following features are not properly supported in the current version:

- Hard/soft limit for SA lifetime (time interval/byte count).
```
当前版本不完全支持以下功能：
- SA 生命周期的硬限制/软限制（时间间隔/字节计数）。



## 实现范例
### ipsec-secgw
基于`ipsec lib` 实现了一个`ipsec gw` 应用程序.
参见：[ipsec_secgw](https://doc.dpdk.org/guides-22.11/sample_app_ug/ipsec_secgw.html)


## 其他

### ipsec-gw 和 ipsec-lib的test plan

参考：[# IPSec gateway and library test plan](https://doc.dpdk.org/dts/test_plans/ipsec_gw_and_library_test_plan.html)

[# 82599 Inline IPsec Tests](https://doc.dpdk.org/dts/test_plans/inline_ipsec_test_plan.html)

[ Cryptodev Performance Application Tests](https://doc.dpdk.org/dts/test_plans/crypto_perf_cryptodev_perf_test_plan.html)


# DPDK中 ipsec-secgw 安全网关的理解

## 概述

该应用程序展示了基于 RFC4301、RFC4303、RFC3602 和 RFC2404 的安全网关（不符合 IPsec 标准，详见下面的限制部分）的实现。

未实现互联网密钥交换（IKE：Internet Key Exchange），因此仅支持手动设置安全策略（SP:  Security Policies）和安全关联(SA: Security Associations)。

**(1)实现**
安全策略（SP）使用 ACL 规则实现，安全关联（SA）存储在表（SAD）中，路由使用 LPM 实现。

**(2)流量分类**
该应用程序将端口分类为受保护（Protected Port）和未受保护(unProtected Port )。因此，在未受保护或受保护端口上接收到的流量分别被视为 inbound 或 outbound。

**(3)ipsec协议卸载**
该应用程序还支持将完整的 IPsec 协议卸载到硬件（旁路加密加速器「(Look aside crypto accelerator」或使用以太网设备「ethernet device」）。在传输过程中，它还支持通过受支持的以太网设备进行内联 IPsec 处理( inline ipsec processing)。这些模式可以在 SA 创建配置期间进行选择。

在完全协议卸载的情况下，头部（ESP 和外部 IP 头部）的处理由硬件完成，应用程序在出站/入站处理期间无需添加/删除它们。

对于内联卸载（ inline offloaded）的出站流量（outbound traffic），应用程序不会进行 LPM 查找以进行路由，因为数据包需要转发的端口将是 SA 的一部分。安全参数将仅在该端口上配置，向其他端口发送数据包可能会导致未加密的数据包被发送出去。


## 概念

### Unprotected Port
取消保护端口：因此该接口收到的流量应该是加密的流量， 需要进行解密。

### Protected port
去保护的端口：因此该接口收到的是加密前的流量，需要进行加密。

### inbound
```text
The Path for IPsec Inbound traffic is:
- Read packets from the port.
- Classify packets between IPv4 and ESP.
- Perform Inbound SA lookup for ESP packets based on their SPI.
- Perform Verification/Decryption (Not needed in case of inline ipsec).
- Remove ESP and outer IP header (Not needed in case of protocol offload).
- Inbound SP check using ACL of decrypted packets and any other IPv4 packets.
- Routing.
- Write packet to port.
```

因此：inbound 就是收到的是加密的流量，需要进行解密。

### outbound
```text
The Path for the IPsec Outbound traffic is:
- Read packets from the port.
- Perform Outbound SP check using ACL of all IPv4 traffic.
- Perform Outbound SA lookup for packets that need IPsec protection.
- Add ESP and outer IP header (Not needed in case protocol offload).
- Perform Encryption/Digest (Not needed in case of inline ipsec).
- Routing.
- Write packet to port.
```
因此：outbound 就是收到的是待加密的流量，需要进行加密。

### 加密模式

参考：ipsec-secgw 中 加密模式的选择，如下所示。

![](attachments/Pasted%20image%2020250115105441.png)

使用**软件crypto设备**(比如 CPU驱动的 AESNI-GCM /AESNI-MB 虚拟加密设备)进行加解密，此时就选择使用**no-offload模式**，是**异步**的通过软件crypto进行加解密。

## 流程
### 设备初始化流程

![](attachments/Pasted%20image%2020250113102046.png)

### 设备操作流程

![](attachments/Pasted%20image%2020250113102123.png)


# DPDK自实现IPSec

## 需求
受限于内核版本的IPsec的转发性能不足，单核大概是1Gbps(bit per second), 
而DPDK官方给的
##  DPDK实现IPsec的理解
### 参考资料

### 思路
#### 加密隧道配置
#### 加密网段
#### 隧道的变更
#### 隧道的加解密匹配
##### inbound的业务流量
##### outbound的业务流量
##### inbound的探测流量
##### outbound的探测流量
#### 小结

# 参考
```bash
# DPDK：Cryptodev
https://fishmwei.github.io/2022/09/18/2022-20220918-cryptodev-weekly/
[系列：https://fishmwei.github.io/categories/%E6%8A%80%E6%9C%AF/dpdk/]

# dpdk cryptodev 官方说明
https://doc.dpdk.org/guides/prog_guide/cryptodev_lib.html
https://doc.dpdk.org/dts/test_plans/ipsec_gw_cryptodev_func_test_plan.html

# DPDK：ipsec-secgw（安全网关）
https://blog.csdn.net/hhd1988/article/details/124282717
https://blog.csdn.net/hhd1988/article/details/124400092
【系列文章：https://blog.csdn.net/hhd1988/category_11215757.html】

# dpdk ipsec-gateway 官方说明：
https://doc.dpdk.org/guides/prog_guide/rte_security.html
http://doc.dpdk.org/guides/prog_guide/ipsec_lib.html
https://doc.dpdk.org/guides-20.11/sample_app_ug/ipsec_secgw.html

# DPDK ipsec ZUC 算法(zuc 和 aes-gcm 应该是并列的 )：
https://fishmwei.github.io/2022/10/01/2022-20221001-zuc-weekly/

# dpdk加解密设备与IPSEC（dpdk中 ipsec_lib 和  cryptodev_lib 的关系）
https://www.cnblogs.com/hugetong/p/10469840.html

```