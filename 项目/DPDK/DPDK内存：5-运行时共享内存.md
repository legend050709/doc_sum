```table-of-contents
```
# DPDK运行时共享内存
DPDK主从进程通信，以及使用dpdk-pdump抓包，或者使用dpdk-telemetry查看进程信息，这些都和DPDK的运行时共享内存相关。
站在用户的视角，DPDK运行时共享内存就是一个文件目录，目录里面有各类和具体功能相关的文件。

## 运行时共享内存目录
经常接触DPDK的同学一定对参数`file-prefix`不陌生，通过这个参数允许非合作的进程拥有不同的运行时内存区域，如下为官方文档的引用。
```bash
--file-prefix: to allow processes that do not want to co-operate to have different memory regions
```

![](attachments/Pasted%20image%2020250402163010.png)

`file-prefix`的默认文件路径是`/var/run/dpdk/rte`. 
**主从进程运行，`dpdk-pdump`，`dpdk-telemetry`等功能都依赖这个内存区域**。



### 目录下的文件

目录下的内容如下所示，不同的机器环境和运行参数会导致看到的目录内容有差异。
![](attachments/Pasted%20image%2020250402163213.png)


**（1）config文件**
是DPDK初始化时调用函数`rte_config_init`创建的，存放的内容为内存配置相关，使用`dpdk-pdump`抓包需要用到此文件，如果运行时指定了参数`no_shconf`，则不会创建`config`。config文件大小为`sizeof(struct rte_mem_config)`，具体信息可以查看源码`struct rte_mem_config`。

**(2) dpdk_telemetry.v2文件**
是一个套接字文件，初始化时调用函数`rte_telemetry_init`创建，使用`dpdk-telemetry.py`脚本查看进程信息时需要指定此文件的路径。可查看DPDK进程的信息包括`ethdev stats`，`ethdev`端口列表，`mempool`和`ring`等。

**(3) fbarray_memseg-xxk-x-x文件**
2048k表示使用的大页为2M，第一个x表示机器上的numa号，第二个x为大页号，即`segmet list`。

**(4) fbarray_memzone文件**
是DPDK初始化时调用函数`rte_eal_memzone_init`创建，存放的是DPDK可以使用`memzone`的信息，文件大小为`RTE_MAX_MEMZONE*sizeof(struct rte_memzone)`。DPDK可以使用的`memzone`的个数在初始化时就会规定好，上限为`RTE_MAX_MEMZONE`，该值在`dpdk 22.11`里面为`2560`。

**(5) hugepage_info文件**
是DPDK初始化调用`eal_hugepage_info_init`时创建，存放的是DPDK使用的大页相关信息，文件大小为`sizeof(struct hugepage_info)`，具体信息可以查看源码`struct hugepage info`。需要说明的是，系统可能有多种大页，但是DPDK只使用最大的。

**(6)mp_socket文件**
是`primary`和`secondary`进程之间的通信文件。`mp: 即 multi process`。

# dpdk-pdump抓包
dpdk-pdump抓包框架从DPDK v16.07版本开始引入，整个抓包框架由pdump库和pdump工具两部分组成，如下所示。

![](attachments/Pasted%20image%2020250402164309.png)

`dpdk-pdump`作为`secondary进程`和`dpdk primary进程`以`domain socket`的方式通信，进行消息传递来完成抓包。

## 流程
`dpdk primary`进程和`dpdk-pdump`进程的抓包流程如图3所示，省略了一些细节。

![](attachments/Pasted%20image%2020250402171113.png)

1）首先`dpdk primary`进程调用`rte_eal_init`进行初始化，创建`config文件`，并进行大页内存的映射，在`rte_eal_init`还会调用`rte_mp_channel_init`创建`domain socket server线程`，另外`dpdk primay`还需要调用`rte_pdump_init`注册回掉函数。

2）`dpdk-pdump进程`也调用`rte_eal_init`进程进行初始化，并`attach config文件`，接着`dpdk-pdump进程`从大页上申请内存创建`mempool和rte_ring`。需要注意的是这个`mempool和ring`，`dpdk-pdump和dpdk primary进程`是都可以访问的，因为它们共享大页内存，映射的基地址在`primary`和`dpdk-pdump`进程中是相同的。同样在`rte_eal_init`里面也会调用`rte_mp_channle_init`创建`domain socket client线程`。

3）`dpdk-pdump`进程构造请求消息，请求消息里面的`rte_mempool和rte_ring`指针指向在2）中创建并初始化好的`mempool和ring地址`。然后通过`domain socket`将请求消息发送给`dpdk primary`进程。

```c
struct pdump_request {
    uint16_t ver;
    uint16_t op;
    uint32_t flags;
    char device[RTE_DEV_NAME_MAX_LEN];
    uint16_t queue;
    struct rte_ring *ring;
    struct rte_mempool *mp;

    const struct rte_bpf_prm *prm;
    uint32_t snaplen;
};
```

4）`dpdk primary`进程`domain socket server`线程接收请求消息并解析，然后调用`rte_pktmbuf_copy`从`dpdk-pdump`进程创建的`mempool`中申请新的`rte_mbuf`，并将收到的`rte_mbuf copy`到新申请的`rte_mbuf`，再将`copy`出来的的`rte_mbuf`以指针数组的形式`enqueue`到`ring`里面。

5）`dpdk-pdump`进程从`ring`里面`dequeue rte_mbuf`，并根据需求分`tx`和`rx`写到`pcap`文件。


# 参考
```bash
# 6、DPDK运行时共享内存
https://zhuanlan.zhihu.com/p/665639460
```