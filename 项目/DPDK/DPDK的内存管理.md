```table-of-contents
```
# 预留内存
## 没有预留内存的问题
现在新版本的 DPDK默认都是按需申请大页，比如系统启动时，一共分配了100个大页，但是DPDK程序启动的时候，不会将100个大页全部占用，而是基于实际需要多少个大页，就从系统中申请多少个大页。

如果DPDK程序存在频繁的配置变更，每次配置下发都需要进行内存申请，然后进行释放。
正常是先从DPDK管理的内存中申请，如果DPDK管理的内存不足，则从系统中申请一个大页；使用完毕之后，DPDK应用将内存还给DPDK管理，DPDK管理时发现管理的内存超过一个大页且连续，可能又将大页还给了系统，即DPDK是按需管理内存。

如果存在大量的配置变更，DPDK需要频繁的从系统中申请大页，以及将大页还给系统。这个是比较耗时的。

在老版本的DPDK，系统申请100个大页，在DPDK系统启动的时候，直接从系统中申请100个大页全部给占用了，实际运行中，不存在频繁的从系统中申请大页以及释放大页给系统。

## DPDK中的预留内存
```bash
`--socket-mem`: Memory to allocate from hugepages on specific sockets. In dynamic memory mode, this memory will also be pinned (i.e. not released back to the system until application closes).



```

# 参考
```bash
# DPDK 22.11内存管理变化解析
http://blog.chinaunix.net/uid-28541347-id-5877488.html

--socket-mem 参数说明
https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html
https://doc.dpdk.org/guides/linux_gsg/build_sample_apps.html

```