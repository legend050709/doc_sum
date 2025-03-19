```table-of-contents
```
# 背景
如果存在大量 ACL规则，那么ACL 规则的编译的时间就会比较长。
在DPDK应用程序中，存在多个转发线程需要进行ACL的查询；存在控制线程，进行ACL的增删改的操作。

# 问题
- acl 如果 per-core，会占用大量的内存
- acl 的 编译会耗费大量的时间  
如果在 ctl线程中，则会影响配置下发、查询；在转发线程中，则会影响数据转发。

使用DPDK acl存在2个问题：
- 不支持增量变更  
    每次增删ACL后，都需要全量编译。
- 大量规则的编译速度慢。



# 思路

在已经存在大量配置的情况下，增删少量的ACL规则，不需要每次增删都进行编译，只需要合并变更，编译一次就可以了。
因此 acl 配置为 shared，并且单独设置一个 acl 线程来进行编译处理。

![](attachments/Pasted%20image%2020250219172511.png)

```bash
注：  
match ：dpvs程序中的配置结构，多线程共享；  
acl ：dpdk提供的单独的结构，多线程共享；  
svc：dpvs程序中的配置结构，per-core。
```

![](attachments/Pasted%20image%2020250219172610.png)

## 梳理

### match
match：只是在ctl线程中写，在acl线程中读，在fwd中不会访问（在转发线程中访问的是其关联的pre-core的svc）；多个core共享。

#### match的update
ctl线程：直接在ctl中对原始的match 写操作，可能会影响acl线程中的读到的match，但是问题不大。
因为每次match增删，最终都会在acl线程中重新读取所有的match，构建 acl_ctx。如果担心影响，ctl线程中update时，申请新的结构，同时将旧的添加到释放链表中延迟释放，也是可以的。

#### match的delete
ctl线程：ctl中逻辑删除（从hash表中删除，该hash表用来查重），同时添加到释放链表 `g_free_list` 中（头插法，cas方式更新链表头）;
acl线程：`cas`方式夺取整个 `g_free_list` 链表（夺取后`g_free_list`链表为空），获取要释放的节点，将 match 以及 svc 加入到 新的释放链表中，待 ctl中的释放链表来释放。（因为svc会被fwd来访问，所以延迟释放）

关键词：RCU（RCU方式读写match的链表）、CAS（cas方式更新释放链表）、延迟释放。

### g_acl_ctx
g_acl_ctx： 在 acl线程中写，在fwd线程中读。多个core共享。

#### g_acl_ctx 的update
acl线程：每次都是创建新的 new_acl_ctx，而不是在原有的 g_acl_ctx 中通过增删rule来更新。创建并且build 完 new_acl_ctx后，将 new_acl_ctx 赋值给 g_acl_ctx。

#### g_acl_ctx的delete
acl线程：原有的旧的 g_acl_ctx 的释放是等待所有的FWD轮询一轮以后，就不会有FWD线程对旧的 g_acl_ctx的读取，下一轮FWD读取的都是 新的 g_acl_ctx。

### svc
svc：在 ctl 中写，在 fwd中读（通过`all_svc[cid][idx]` 获取到）。配置 `per-core`。

#### svc的update
svc的update 即match的update，因为多个core下的相同svc关联一份match。

#### svc的delete
svc在ctl中写是因为match在ctl中写，如果match在ctl中不存在写了，那么svc也不会存在写操作。
match的释放是在acl中释放的，那么svc也可以在acl中进行移除操作（`all_svc[cid][idx] = NULL` && svc 加入到释放链表），然后在ctl中通过释放链表延迟释放（延迟：等待所有的FWD轮询一轮）。

# 其他
## 大量ACL编译时间长的问题

存在大量的ACL规则，每次增删少数的ACL规则，都需要重新进行编译。编译时间过长的问题。

### 思路
如果大量的ACL规则中，大部分单个IP粒度的ACL规则，少量为`IP-range`的ACL规则。
那么可以用hash表来保存单个IP粒度的规则，对于`IP-range`的ACL规则，继续通过DPDK的`acl ctx`前缀树来组织。

#### ACL的插入、删除
大部分的单IP粒度的ACL直接放入到了Hash表中，不存在编译。
ACl Ctx前缀树中的IP-range粒度的ACL规则较少，编译时间较小。

#### 流量的匹配
流量到来时，基于包的信息，先在Hash表中进行查询。然后在Acl Ctx前缀树中进行查询。
如果两者都查询到，则再比较查询到的实体的优先级，选择优先级高的。


# LPM
`LPM`和`ACL`类似， 都使用一个前缀树。
在转发线程中查询 LPM，在控制线程中的 `lpm_ctx(struct rte_lpm * 类型)`下增删 `lpm_entry`。

为了不加锁，那么也是每次生成一个新的 `lpm_ctx(struct rte_lpm * 类型). `，而不是在原有的基础上，增量的增删 `lpm_entry`。
> 注：每次名称不一样，可以加一个计数器，每次要生成新的，计数器加一，计数器作为名称的一部分，就可以保证不重复
可以用一个额外的线程(非控制线程，非转发线程)去做这样的更新`lpm_ctx`的事情。

至于原有的`lpm_ctx(struct rte_lpm * 类型)`的释放， 则是等待所有的转发线程执行一轮之后，这样就不会再次被访问了，才考虑释放。


# 参考
```bash

```