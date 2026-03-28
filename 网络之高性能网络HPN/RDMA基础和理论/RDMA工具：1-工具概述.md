```table-of-contents
```
# 总体介绍
RDMA 有一堆的管理工具和测试工具，慢慢都归集于`linux-rdma/rdma-core`项目下，甚至这些工具都有 man 手册，比如`ibv_devices`,可以说非常友好了，你只需要知晓有哪些工具，大概是干什么的，具体使用的时候查询它的 man 手册就好了。

根据 Red Hat/CentOS 系统中不同的包所包含的工具分类进行介绍。
```bash
# RHEL/CentOS
yum install libibverbs-utils infiniband-diags rdma-core perftest librdmacm-utils
```

这些工具主要用于：
- RDMA 设备状态检查
- 网络连接测试
- 性能测试和基准测试
- 故障排除
- 网络拓扑发现

# 各个工具包
## libibverbs-utils 包
### 包内容
```bash
# rpm -qf /bin/ibv_devinfo
libibverbs-utils-50mlnx1-1.50218.x86_64

# rpm -ql libibverbs-utils
/usr/bin/ibv_asyncwatch
/usr/bin/ibv_devices
/usr/bin/ibv_devinfo
/usr/bin/ibv_rc_pingpong
/usr/bin/ibv_srq_pingpong
/usr/bin/ibv_uc_pingpong
/usr/bin/ibv_ud_pingpong
/usr/bin/ibv_xsrq_pingpong
/usr/share/man/man1/ibv_asyncwatch.1.gz
/usr/share/man/man1/ibv_devices.1.gz
/usr/share/man/man1/ibv_devinfo.1.gz
/usr/share/man/man1/ibv_rc_pingpong.1.gz
/usr/share/man/man1/ibv_srq_pingpong.1.gz
/usr/share/man/man1/ibv_uc_pingpong.1.gz
/usr/share/man/man1/ibv_ud_pingpong.1.gz
/usr/share/man/man1/ibv_xsrq_pingpong.1.gz
```

### 工具

#### ibv_devices

- **ibv_devices**：列出系统中所有的 IB 设备

#### ibv_devinfo

- **ibv_devinfo**：显示 IB 设备的详细信息

##### 重要字段
```bash
# ibv_devinfo -vv
hca_id:	mlx5_bond_0
	transport:			InfiniBand (0)
	fw_ver:				22.36.1010
	node_guid:			08c0:eb03:00f0:28b2
	sys_image_guid:			08c0:eb03:00f0:28b2
	vendor_id:			0x02c9
	vendor_part_id:			4125
	hw_ver:				0x0
	board_id:			MT_0000000359
	phys_port_cnt:			1
	max_mr_size:			0xffffffffffffffff
	page_size_cap:			0xfffffffffffff000
	max_qp:				131072
	max_qp_wr:			32768
	device_cap_flags:		0xed721c36
					BAD_PKEY_CNTR
					BAD_QKEY_CNTR
					AUTO_PATH_MIG
					CHANGE_PHY_PORT
					PORT_ACTIVE_EVENT
					SYS_IMAGE_GUID
					RC_RNR_NAK_GEN
					MEM_WINDOW
					XRC
					MEM_MGT_EXTENSIONS
					MEM_WINDOW_TYPE_2B
					RAW_IP_CSUM
					MANAGED_FLOW_STEERING
					Unknown flags: 0xC8400000
	max_sge:			30
	max_sge_rd:			30
	max_cq:				16777216
	max_cqe:			4194303
	max_mr:				16777216
	max_pd:				8388608
	max_qp_rd_atom:			16
	max_ee_rd_atom:			0
	max_res_rd_atom:		2097152
	max_qp_init_rd_atom:		16
	max_ee_init_rd_atom:		0
	atomic_cap:			ATOMIC_HCA (1)
	max_ee:				0
	max_rdd:			0
	max_mw:				16777216
	max_raw_ipv6_qp:		0
	max_raw_ethy_qp:		0
	max_mcast_grp:			2097152
	max_mcast_qp_attach:		240
	max_total_mcast_qp_attach:	503316480
	max_ah:				2147483647
	max_fmr:			0
	max_srq:			8388608
	max_srq_wr:			32767
	max_srq_sge:			31
	max_pkeys:			128
	local_ca_ack_delay:		16
	general_odp_caps:
					ODP_SUPPORT
					ODP_SUPPORT_IMPLICIT
	rc_odp_caps:
					SUPPORT_SEND
					SUPPORT_RECV
					SUPPORT_WRITE
					SUPPORT_READ
					SUPPORT_SRQ
	uc_odp_caps:
					NO SUPPORT
	ud_odp_caps:
					SUPPORT_SEND
	xrc_odp_caps:
					SUPPORT_SEND
					SUPPORT_WRITE
					SUPPORT_READ
					SUPPORT_SRQ
	completion timestamp_mask:			0x7fffffffffffffff
	hca_core_clock:			1000000kHZ
	raw packet caps:
					C-VLAN stripping offload
					Scatter FCS offload
					IP csum offload
					Delay drop
	device_cap_flags_ex:		0x30000055ED721C36
					RAW_SCATTER_FCS
					PCI_WRITE_END_PADDING
					Unknown flags: 0x3000004100000000
	tso_caps:
		max_tso:			262144
		supported_qp:
					SUPPORT_RAW_PACKET
	rss_caps:
		max_rwq_indirection_tables:			524288
		max_rwq_indirection_table_size:			2048
		rx_hash_function:				0x1
		rx_hash_fields_mask:				0x800000FF
		supported_qp:
					SUPPORT_RAW_PACKET
	max_wq_type_rq:			8388608
	packet_pacing_caps:
		qp_rate_limit_min:	1kbps
		qp_rate_limit_max:	100000000kbps
		supported_qp:
					SUPPORT_RAW_PACKET
	tag matching not supported

	cq moderation caps:
		max_cq_count:	65535
		max_cq_period:	4095 us

	maximum available device memory:	131072Bytes

	num_comp_vectors:		63
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			0
			port_lid:		0
			port_lmc:		0x00
			link_layer:		Ethernet
			max_msg_sz:		0x40000000
			port_cap_flags:		0x04010000
			port_cap_flags2:	0x0000
			max_vl_num:		invalid value (0)
			bad_pkey_cntr:		0x0
			qkey_viol_cntr:		0x0
			sm_sl:			0
			pkey_tbl_len:		1
			gid_tbl_len:		255
			subnet_timeout:		0
			init_type_reply:	0
			active_width:		2X (16)
			active_speed:		50.0 Gbps (64)
			phys_state:		LINK_UP (5)
			GID[  0]:		fe80:0000:0000:0000:0ac0:ebff:fef0:28b2, RoCE v1
			GID[  1]:		fe80::ac0:ebff:fef0:28b2, RoCE v2
			GID[  2]:		0000:0000:0000:0000:0000:ffff:1b69:b303, RoCE v1
			GID[  3]:		::ffff:27.105.179.3, RoCE v2
```


```bash
max_qp_wr:
max_sge:
max_cqe:
max_srq_wr:
max_srq_sge:
active_mtu:
max_msg_sz:     0x40000000; 一次WR发送，对端最多接受1G的消息; 
link_layer:		Ethernet; 
```

##### 原理

`ibv_devinfo` 既然是 `ibv_`开头，就意味着程序内部会调用 `libibverbs`库的接口，它本质上是对 **libibverbs + provider 驱动** 的一次 `demo` 级封装。

`ibv_devinfo` 的执行流程概括如下：

```bash
libibverbs 扫描 RDMA 设备目录
            ↓
加载匹配的 provider（mlx5、rxe、hfi1……）
            ↓
使用 provider 回调获取设备属性
            ↓
格式化打印设备信息
```

`ibv_devinfo` 的原理就是：

让 libibverbs 枚举 NIC → 为每个 NIC 打开 `verbs context` → 调用 provider 的 query_device/query_port → 打印结果。

###### 扫描 RDMA 设备
`ibv_devinfo` 调用：
```c
ibv_get_device_list(&num_devices);
```

扫描 `/sys/class/infiniband/`，找到所有 RNIC 设备 ,（如 `mlx5_0`, `mlx5_1`, `rxe0`, `hns_0`…）


###### 加载 provider（驱动插件）
`libibverbs` 使用 **插件机制**：插件目录位于 `/etc/libibverbs.d/`；**libibverbs 会动态加载对应 provider**（根据设备 `vendor_id/device_id` ），provider 注册对应的 `verbs` ops，例如：
```c
struct verbs_context_ops {
    query_device;
    query_port;
    alloc_pd;
    create_cq;
    ...
};
```
这些函数最终调到内核 ibverbs 驱动（mlx5、rxe、hfi1 等）对应的 ioctl。

```bash
# ll /lib64/libibverbs*
-rw-r--r-- 1 root root 2675104 Nov 30 18:39 /lib64/libibverbs.a
lrwxrwxrwx 1 root root      15 Oct 24 18:51 /lib64/libibverbs.so -> libibverbs.so.1
lrwxrwxrwx 1 root root      23 Dec  3 13:07 /lib64/libibverbs.so.1 -> libibverbs.so.1.14.35.0
-rwxr-xr-x 1 root root  131384 Nov 13  2022 /lib64/libibverbs.so.1.14.35.0

/lib64/libibverbs:
total 2112
-rwxr-xr-x 1 root root 246032 Nov 29 13:52 libbnxt_re-rdmav34.so
-rwxr-xr-x 1 root root 270816 Nov 29 13:52 libcxgb4-rdmav34.so
lrwxrwxrwx 1 root root     21 Nov 29 13:52 libefa-rdmav34.so -> ../libefa.so.1.3.51.0
-rwxr-xr-x 1 root root 144848 Nov 29 13:52 liberdma-rdmav34.so
-rwxr-xr-x 1 root root  95600 Nov 29 13:52 libhfi1verbs-rdmav34.so
lrwxrwxrwx 1 root root     21 Nov 29 13:52 libhns-rdmav34.so -> ../libhns.so.1.0.51.0
-rwxr-xr-x 1 root root  95872 Nov 29 13:52 libipathverbs-rdmav34.so
-rwxr-xr-x 1 root root 214440 Nov 29 13:52 libirdma-rdmav34.so
lrwxrwxrwx 1 root root     22 Nov 29 13:52 libmana-rdmav34.so -> ../libmana.so.1.0.51.0
lrwxrwxrwx 1 root root     22 Nov 29 13:52 libmlx4-rdmav34.so -> ../libmlx4.so.1.0.51.0
lrwxrwxrwx 1 root root     27 Dec  3 13:28 libmlx5-rdmav34.so -> /lib64/libmlx5.so.1.19.35.0
-rwxr-xr-x 1 root root 265640 Nov 29 13:52 libmthca-rdmav34.so
-rwxr-xr-x 1 root root 160752 Nov 29 13:52 libocrdma-rdmav34.so
-rwxr-xr-x 1 root root 223240 Nov 29 13:52 libqedr-rdmav34.so
-rwxr-xr-x 1 root root 145576 Nov 29 13:52 librxe-rdmav34.so
-rwxr-xr-x 1 root root 106296 Nov 29 13:52 libsiw-rdmav34.so
-rwxr-xr-x 1 root root 166728 Nov 29 13:52 libvmw_pvrdma-rdmav34.so

# ll /etc/libibverbs.d/
total 8
-rw-r--r-- 1 root root 12 Dec  2 14:46 hrn3.driver
-rw-r--r-- 1 root root 12 Nov 13  2022 mlx5.driver
```

###### 打开设备获取详细信息
对每个设备：
```c
struct ibv_context *ctx = ibv_open_device(dev);

```

`ctx` 内部是 provider wrap 的上下文，用于调用 HW。

###### 查询设备属性
`ibv_devinfo` 会调用以下 API：

|动作|libibverbs API|最终来源|
|---|---|---|
|查询设备能力|`ibv_query_device(ctx, &dev_attr)`|provider → 内核 → NIC|
|查询端口属性|`ibv_query_port(ctx, port, &port_attr)`|provider → 内核|
|查询 GID 表|`ibv_query_gid(ctx, port, gid_idx, &gid)`|sysfs + 内核|
|查询 P_Key|`ibv_query_pkey(ctx, port, idx, &pkey)`|内核|
|查询 link layer|通过 sysfs `/sys/class/infiniband/<dev>/ports/X/link_layer`|内核驱动|

所以你看到的 MTU、支持的 transport、最大 QP 数、最大 WR 数等全部来自 **verbs 驱动向内核查询的硬件能力**。

```c
/**
 * ibv_query_device - Get device properties
 */
int ibv_query_device(struct ibv_context *context,
		     struct ibv_device_attr *device_attr);

/**
 * ibv_query_port - Get port properties
 */
int ibv_query_port(struct ibv_context *context, uint8_t port_num,
		   struct _compat_ibv_port_attr *port_attr);
```


##### `ibv_devinfo` 为什么有时信息不一致？

因为：
- 部分字段来自 **sysfs**（如 link_layer, state）
- 部分来自 **ioctl**（如 max_qp, max_wr）
- 部分来自 **provider 自定义的扩展属性**


不同 provider（mlx5/rxe/hns）实现略有差异。


#### ibv_rc_pingpong

`rdma-core`中`libibverbs-utils`提供了针对`verbs`接口的`pingpong`检测工具，可以用于检测链路的带宽和延时。
针对`rdma`的操作类型如`RC`，`UD`，`UC`等分别提供了对应的工具。`ibv_rc_pingpong`,`ibv_srq_pingpong`,  `ibv_uc_pingpong`,  `ibv_ud_pingpong`,  `ibv_xsrq_pingpong`。

- **ibv_rc_pingpong**：RC（Reliable Connection）模式的 ping-pong 测试工具
    
- **ibv_uc_pingpong**：UC（Unreliable Connection）模式测试工具
    
- **ibv_srq_pingpong**：Shared Receive Queue 测试工具

##### 使用方法
```bash

server端：ibv_rc_pingpong -g 3 -d mlx5_bond_0
client端：ibv_rc_pingpong -g 3 -d mlx5_bond_0  10.17.70.166

show_gids: 查看具体的gid
```


![](attachments/Pasted%20image%2020251031203818.png)

如上所示，得到的 RTT是 `15.34us`，但是添加`-t`之后，得到的`cycles` 是 `2392`。进行换算。需要查看`RNIC`网卡的 评率`ibv_devinfo -vv`。

![](attachments/Pasted%20image%2020251031203956.png)

经过换算之后的时间，平均为15.44us，和上面的平均时间 `15.34us`是接近的。


##  infiniband-diags 包
### 包内容
```bash
# rpm -qf /sbin/ibstat
infiniband-diags-50mlnx1-1.50218.x86_64


# rpm -ql infiniband-diags
/etc/infiniband-diags/error_thresholds
/etc/infiniband-diags/ibdiag.conf
/usr/lib64/libibmad.so.5
/usr/lib64/libibmad.so.5.3.28.0
/usr/lib64/libibnetdisc.so.5
/usr/lib64/libibnetdisc.so.5.0.28.0
/usr/sbin/check_lft_balance.pl
/usr/sbin/dump_fts
/usr/sbin/dump_lfts.sh
/usr/sbin/dump_mfts.sh
/usr/sbin/ibaddr
/usr/sbin/ibcacheedit
/usr/sbin/ibccconfig
/usr/sbin/ibccquery
/usr/sbin/ibclearcounters
/usr/sbin/ibclearerrors
/usr/sbin/ibfindnodesusing.pl
/usr/sbin/ibhosts
/usr/sbin/ibidsverify.pl
/usr/sbin/iblinkinfo
/usr/sbin/ibnetdiscover
/usr/sbin/ibnodes
/usr/sbin/ibping
/usr/sbin/ibportstate
/usr/sbin/ibqueryerrors
/usr/sbin/ibroute
/usr/sbin/ibrouters
/usr/sbin/ibstat
/usr/sbin/ibstatus
/usr/sbin/ibswitches
/usr/sbin/ibsysstat
/usr/sbin/ibtracert
/usr/sbin/perfquery
/usr/sbin/saquery
/usr/sbin/sminfo
/usr/sbin/smpdump
/usr/sbin/smpquery
/usr/sbin/vendstat
/usr/share/man/man8/dump_lfts.8.gz
/usr/share/man/man8/dump_mfts.8.gz
/usr/share/man/man8/ibclearcounters.8.gz
/usr/share/man/man8/ibclearerrors.8.gz
/usr/share/perl5/vendor_perl/IBswcountlimits.pm
```

### 工具
- **ibstat**：显示 IB 适配器和端口状态
    
- **ibstatus**：显示 IB 设备状态信息
    
- **ibhosts**：显示 IB 子网中的主机
    
- **ibnetdiscover**：发现并显示 IB 网络拓扑
    
- **iblinkinfo**：显示 IB 网络链路信息
    

## rdma-core 包
### 包内容
```bash
# rpm -ql rdma-core
/etc/rdma
/etc/rdma/mlx4.conf
/etc/rdma/rdma.conf
/etc/rdma/sriov-vfs
/etc/udev/rules.d/70-persistent-ipoib.rules
/usr/bin/rxe_cfg
/usr/lib/dracut/modules.d/05rdma
/usr/lib/dracut/modules.d/05rdma/module-setup.sh
/usr/lib/modprobe.d/libmlx4.conf
/usr/lib/systemd/system/rdma-ndd.service
/usr/lib/udev/rdma_rename
/usr/lib/udev/rules.d/60-rdma-ndd.rules
/usr/lib/udev/rules.d/60-rdma-persistent-naming.rules
/usr/lib/udev/rules.d/75-rdma-description.rules
/usr/lib/udev/rules.d/90-rdma-umad.rules
/usr/libexec/mlx4-setup.sh
/usr/libexec/rdma-set-sriov-vf
/usr/sbin/rdma-ndd
/usr/share/doc/rdma-core-50mlnx1
/usr/share/doc/rdma-core-50mlnx1/COPYING.BSD_FB
/usr/share/doc/rdma-core-50mlnx1/COPYING.BSD_MIT
/usr/share/doc/rdma-core-50mlnx1/COPYING.GPL2
/usr/share/doc/rdma-core-50mlnx1/COPYING.md
/usr/share/doc/rdma-core-50mlnx1/README.md
/usr/share/doc/rdma-core-50mlnx1/rxe.md
/usr/share/doc/rdma-core-50mlnx1/tag_matching.md
/usr/share/doc/rdma-core-50mlnx1/udev.md
/usr/share/man/man7/rxe.7.gz
/usr/share/man/man8/rdma-ndd.8.gz
/usr/share/man/man8/rxe_cfg.8.gz



# rpm -ql rdma-core-devel
/usr/include/infiniband
/usr/include/infiniband/acm.h
/usr/include/infiniband/acm_prov.h
/usr/include/infiniband/arch.h
/usr/include/infiniband/ib.h
/usr/include/infiniband/ib_user_ioctl_verbs.h
/usr/include/infiniband/ibnetdisc.h
/usr/include/infiniband/ibnetdisc_osd.h
/usr/include/infiniband/mad.h
/usr/include/infiniband/mad_osd.h
/usr/include/infiniband/mlx4dv.h
/usr/include/infiniband/mlx5_api.h
/usr/include/infiniband/mlx5_user_ioctl_verbs.h
/usr/include/infiniband/mlx5dv.h
/usr/include/infiniband/opcode.h
/usr/include/infiniband/sa-kern-abi.h
/usr/include/infiniband/sa.h
/usr/include/infiniband/tm_types.h
/usr/include/infiniband/umad.h
/usr/include/infiniband/umad_cm.h
/usr/include/infiniband/umad_sa.h
/usr/include/infiniband/umad_sa_mcm.h
/usr/include/infiniband/umad_sm.h
/usr/include/infiniband/umad_str.h
/usr/include/infiniband/umad_types.h
/usr/include/infiniband/verbs.h
/usr/include/infiniband/verbs_api.h
/usr/include/rdma
/usr/include/rdma/rdma_cma.h
/usr/include/rdma/rdma_cma_abi.h
/usr/include/rdma/rdma_verbs.h
/usr/include/rdma/rsocket.h
/usr/lib64/libibmad.a
/usr/lib64/libibmad.so
/usr/lib64/libibnetdisc.a
/usr/lib64/libibnetdisc.so
/usr/lib64/libibumad.a
/usr/lib64/libibumad.so
/usr/lib64/libibverbs.a
/usr/lib64/libibverbs.so
/usr/lib64/libmlx4.a
/usr/lib64/libmlx4.so
/usr/lib64/libmlx5.a
/usr/lib64/libmlx5.so
/usr/lib64/librdmacm.a
/usr/lib64/librdmacm.so
/usr/lib64/librxe-rdmav25.a
/usr/lib64/pkgconfig/libibmad.pc
/usr/lib64/pkgconfig/libibnetdisc.pc
/usr/lib64/pkgconfig/libibumad.pc
/usr/lib64/pkgconfig/libibverbs.pc
/usr/lib64/pkgconfig/libmlx4.pc
/usr/lib64/pkgconfig/libmlx5.pc
/usr/lib64/pkgconfig/librdmacm.pc
/usr/share/doc/rdma-core-devel-50mlnx1
/usr/share/doc/rdma-core-devel-50mlnx1/MAINTAINERS
/usr/share/man/man3/ibnd_debug.3.gz
/usr/share/man/man3/ibnd_destroy_fabric.3.gz
/usr/share/man/man3/ibnd_discover_fabric.3.gz
/usr/share/man/man3/ibnd_find_node_dr.3.gz
/usr/share/man/man3/ibnd_find_node_guid.3.gz
/usr/share/man/man3/ibnd_iter_nodes.3.gz
/usr/share/man/man3/ibnd_iter_nodes_type.3.gz
/usr/share/man/man3/ibnd_set_max_smps_on_wire.3.gz
/usr/share/man/man3/ibnd_show_progress.3.gz
/usr/share/man/man3/ibv_ack_async_event.3.gz
/usr/share/man/man3/ibv_ack_cq_events.3.gz
/usr/share/man/man3/ibv_advise_mr.3.gz
/usr/share/man/man3/ibv_alloc_dm.3.gz
/usr/share/man/man3/ibv_alloc_mw.3.gz
/usr/share/man/man3/ibv_alloc_null_mr.3.gz
/usr/share/man/man3/ibv_alloc_parent_domain.3.gz
/usr/share/man/man3/ibv_alloc_pd.3.gz
/usr/share/man/man3/ibv_alloc_td.3.gz
/usr/share/man/man3/ibv_attach_counters_point_flow.3.gz
/usr/share/man/man3/ibv_attach_mcast.3.gz
/usr/share/man/man3/ibv_bind_mw.3.gz
/usr/share/man/man3/ibv_close_device.3.gz
/usr/share/man/man3/ibv_close_xrcd.3.gz
/usr/share/man/man3/ibv_create_ah.3.gz
/usr/share/man/man3/ibv_create_ah_from_wc.3.gz
/usr/share/man/man3/ibv_create_comp_channel.3.gz
/usr/share/man/man3/ibv_create_counters.3.gz
/usr/share/man/man3/ibv_create_cq.3.gz
/usr/share/man/man3/ibv_create_cq_ex.3.gz
/usr/share/man/man3/ibv_create_flow.3.gz
/usr/share/man/man3/ibv_create_flow_action.3.gz
/usr/share/man/man3/ibv_create_qp.3.gz
/usr/share/man/man3/ibv_create_qp_ex.3.gz
/usr/share/man/man3/ibv_create_rwq_ind_table.3.gz
/usr/share/man/man3/ibv_create_srq.3.gz
/usr/share/man/man3/ibv_create_srq_ex.3.gz
/usr/share/man/man3/ibv_create_wq.3.gz
/usr/share/man/man3/ibv_dealloc_mw.3.gz
/usr/share/man/man3/ibv_dealloc_pd.3.gz
/usr/share/man/man3/ibv_dealloc_td.3.gz
/usr/share/man/man3/ibv_dereg_mr.3.gz
/usr/share/man/man3/ibv_destroy_ah.3.gz
/usr/share/man/man3/ibv_destroy_comp_channel.3.gz
/usr/share/man/man3/ibv_destroy_counters.3.gz
/usr/share/man/man3/ibv_destroy_cq.3.gz
/usr/share/man/man3/ibv_destroy_flow.3.gz
/usr/share/man/man3/ibv_destroy_flow_action.3.gz
/usr/share/man/man3/ibv_destroy_qp.3.gz
/usr/share/man/man3/ibv_destroy_rwq_ind_table.3.gz
/usr/share/man/man3/ibv_destroy_srq.3.gz
/usr/share/man/man3/ibv_destroy_wq.3.gz
/usr/share/man/man3/ibv_detach_mcast.3.gz
/usr/share/man/man3/ibv_event_type_str.3.gz
/usr/share/man/man3/ibv_fork_init.3.gz
/usr/share/man/man3/ibv_free_device_list.3.gz
/usr/share/man/man3/ibv_free_dm.3.gz
/usr/share/man/man3/ibv_get_async_event.3.gz
/usr/share/man/man3/ibv_get_cq_event.3.gz
/usr/share/man/man3/ibv_get_device_guid.3.gz
/usr/share/man/man3/ibv_get_device_list.3.gz
/usr/share/man/man3/ibv_get_device_name.3.gz
/usr/share/man/man3/ibv_get_pkey_index.3.gz
/usr/share/man/man3/ibv_get_srq_num.3.gz
/usr/share/man/man3/ibv_inc_rkey.3.gz
/usr/share/man/man3/ibv_init_ah_from_wc.3.gz
/usr/share/man/man3/ibv_memcpy_from_dm.3.gz
/usr/share/man/man3/ibv_memcpy_to_dm.3.gz
/usr/share/man/man3/ibv_modify_cq.3.gz
/usr/share/man/man3/ibv_modify_flow_action.3.gz
/usr/share/man/man3/ibv_modify_qp.3.gz
/usr/share/man/man3/ibv_modify_qp_rate_limit.3.gz
/usr/share/man/man3/ibv_modify_srq.3.gz
/usr/share/man/man3/ibv_modify_wq.3.gz
/usr/share/man/man3/ibv_node_type_str.3.gz
/usr/share/man/man3/ibv_open_device.3.gz
/usr/share/man/man3/ibv_open_qp.3.gz
/usr/share/man/man3/ibv_open_xrcd.3.gz
/usr/share/man/man3/ibv_poll_cq.3.gz
/usr/share/man/man3/ibv_port_state_str.3.gz
/usr/share/man/man3/ibv_post_recv.3.gz
/usr/share/man/man3/ibv_post_send.3.gz
/usr/share/man/man3/ibv_post_srq_ops.3.gz
/usr/share/man/man3/ibv_post_srq_recv.3.gz
/usr/share/man/man3/ibv_query_device.3.gz
/usr/share/man/man3/ibv_query_device_ex.3.gz
/usr/share/man/man3/ibv_query_gid.3.gz
/usr/share/man/man3/ibv_query_pkey.3.gz
/usr/share/man/man3/ibv_query_port.3.gz
/usr/share/man/man3/ibv_query_qp.3.gz
/usr/share/man/man3/ibv_query_rt_values_ex.3.gz
/usr/share/man/man3/ibv_query_srq.3.gz
/usr/share/man/man3/ibv_rate_to_mbps.3.gz
/usr/share/man/man3/ibv_rate_to_mult.3.gz
/usr/share/man/man3/ibv_read_counters.3.gz
/usr/share/man/man3/ibv_reg_dm_mr.3.gz
/usr/share/man/man3/ibv_reg_mr.3.gz
/usr/share/man/man3/ibv_req_notify_cq.3.gz
/usr/share/man/man3/ibv_rereg_mr.3.gz
/usr/share/man/man3/ibv_resize_cq.3.gz
/usr/share/man/man3/ibv_wr_abort.3.gz
/usr/share/man/man3/ibv_wr_atomic_cmp_swp.3.gz
/usr/share/man/man3/ibv_wr_atomic_fetch_add.3.gz
/usr/share/man/man3/ibv_wr_bind_mw.3.gz
/usr/share/man/man3/ibv_wr_complete.3.gz
/usr/share/man/man3/ibv_wr_local_inv.3.gz
/usr/share/man/man3/ibv_wr_post.3.gz
/usr/share/man/man3/ibv_wr_rdma_read.3.gz
/usr/share/man/man3/ibv_wr_rdma_write.3.gz
/usr/share/man/man3/ibv_wr_rdma_write_imm.3.gz
/usr/share/man/man3/ibv_wr_send.3.gz
/usr/share/man/man3/ibv_wr_send_imm.3.gz
/usr/share/man/man3/ibv_wr_send_inv.3.gz
/usr/share/man/man3/ibv_wr_send_tso.3.gz
/usr/share/man/man3/ibv_wr_set_inline_data.3.gz
/usr/share/man/man3/ibv_wr_set_inline_data_list.3.gz
/usr/share/man/man3/ibv_wr_set_sge.3.gz
/usr/share/man/man3/ibv_wr_set_sge_list.3.gz
/usr/share/man/man3/ibv_wr_set_ud_addr.3.gz
/usr/share/man/man3/ibv_wr_set_xrc_srqn.3.gz
/usr/share/man/man3/ibv_wr_start.3.gz
/usr/share/man/man3/mbps_to_ibv_rate.3.gz
/usr/share/man/man3/mlx4dv_init_obj.3.gz
/usr/share/man/man3/mlx4dv_query_device.3.gz
/usr/share/man/man3/mlx4dv_set_context_attr.3.gz
/usr/share/man/man3/mlx5dv_alloc_dm.3.gz
/usr/share/man/man3/mlx5dv_alloc_var.3.gz
/usr/share/man/man3/mlx5dv_create_cq.3.gz
/usr/share/man/man3/mlx5dv_create_flow.3.gz
/usr/share/man/man3/mlx5dv_create_flow_action_modify_header.3.gz
/usr/share/man/man3/mlx5dv_create_flow_action_packet_reformat.3.gz
/usr/share/man/man3/mlx5dv_create_flow_matcher.3.gz
/usr/share/man/man3/mlx5dv_create_mkey.3.gz
/usr/share/man/man3/mlx5dv_create_qp.3.gz
/usr/share/man/man3/mlx5dv_destroy_mkey.3.gz
/usr/share/man/man3/mlx5dv_devx_alloc_uar.3.gz
/usr/share/man/man3/mlx5dv_devx_cq_modify.3.gz
/usr/share/man/man3/mlx5dv_devx_cq_query.3.gz
/usr/share/man/man3/mlx5dv_devx_create_cmd_comp.3.gz
/usr/share/man/man3/mlx5dv_devx_create_event_channel.3.gz
/usr/share/man/man3/mlx5dv_devx_destroy_cmd_comp.3.gz
/usr/share/man/man3/mlx5dv_devx_destroy_event_channel.3.gz
/usr/share/man/man3/mlx5dv_devx_free_uar.3.gz
/usr/share/man/man3/mlx5dv_devx_general_cmd.3.gz
/usr/share/man/man3/mlx5dv_devx_get_async_cmd_comp.3.gz
/usr/share/man/man3/mlx5dv_devx_get_event.3.gz
/usr/share/man/man3/mlx5dv_devx_ind_tbl_modify.3.gz
/usr/share/man/man3/mlx5dv_devx_ind_tbl_query.3.gz
/usr/share/man/man3/mlx5dv_devx_obj_create.3.gz
/usr/share/man/man3/mlx5dv_devx_obj_destroy.3.gz
/usr/share/man/man3/mlx5dv_devx_obj_modify.3.gz
/usr/share/man/man3/mlx5dv_devx_obj_query.3.gz
/usr/share/man/man3/mlx5dv_devx_obj_query_async.3.gz
/usr/share/man/man3/mlx5dv_devx_qp_modify.3.gz
/usr/share/man/man3/mlx5dv_devx_qp_query.3.gz
/usr/share/man/man3/mlx5dv_devx_query_eqn.3.gz
/usr/share/man/man3/mlx5dv_devx_srq_modify.3.gz
/usr/share/man/man3/mlx5dv_devx_srq_query.3.gz
/usr/share/man/man3/mlx5dv_devx_subscribe_devx_event.3.gz
/usr/share/man/man3/mlx5dv_devx_subscribe_devx_event_fd.3.gz
/usr/share/man/man3/mlx5dv_devx_umem_dereg.3.gz
/usr/share/man/man3/mlx5dv_devx_umem_reg.3.gz
/usr/share/man/man3/mlx5dv_devx_wq_modify.3.gz
/usr/share/man/man3/mlx5dv_devx_wq_query.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_dest_devx_tir.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_dest_ib_port.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_dest_ibv_qp.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_dest_table.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_dest_vport.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_drop.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_flow_counter.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_flow_meter.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_modify_header.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_packet_reformat.3.gz
/usr/share/man/man3/mlx5dv_dr_action_create_tag.3.gz
/usr/share/man/man3/mlx5dv_dr_action_destroy.3.gz
/usr/share/man/man3/mlx5dv_dr_action_modify_flow_meter.3.gz
/usr/share/man/man3/mlx5dv_dr_domain_create.3.gz
/usr/share/man/man3/mlx5dv_dr_domain_destroy.3.gz
/usr/share/man/man3/mlx5dv_dr_domain_sync.3.gz
/usr/share/man/man3/mlx5dv_dr_flow.3.gz
/usr/share/man/man3/mlx5dv_dr_matcher_create.3.gz
/usr/share/man/man3/mlx5dv_dr_matcher_destroy.3.gz
/usr/share/man/man3/mlx5dv_dr_rule_create.3.gz
/usr/share/man/man3/mlx5dv_dr_rule_destroy.3.gz
/usr/share/man/man3/mlx5dv_dr_table_create.3.gz
/usr/share/man/man3/mlx5dv_dr_table_destroy.3.gz
/usr/share/man/man3/mlx5dv_flow_action_esp.3.gz
/usr/share/man/man3/mlx5dv_free_var.3.gz
/usr/share/man/man3/mlx5dv_get_clock_info.3.gz
/usr/share/man/man3/mlx5dv_init_obj.3.gz
/usr/share/man/man3/mlx5dv_is_supported.3.gz
/usr/share/man/man3/mlx5dv_open_device.3.gz
/usr/share/man/man3/mlx5dv_qp_ex_from_ibv_qp_ex.3.gz
/usr/share/man/man3/mlx5dv_query_device.3.gz
/usr/share/man/man3/mlx5dv_ts_to_ns.3.gz
/usr/share/man/man3/mlx5dv_wr_post.3.gz
/usr/share/man/man3/mlx5dv_wr_set_dc_addr.3.gz
/usr/share/man/man3/mult_to_ibv_rate.3.gz
/usr/share/man/man3/rdma_accept.3.gz
/usr/share/man/man3/rdma_ack_cm_event.3.gz
/usr/share/man/man3/rdma_bind_addr.3.gz
/usr/share/man/man3/rdma_connect.3.gz
/usr/share/man/man3/rdma_create_ep.3.gz
/usr/share/man/man3/rdma_create_event_channel.3.gz
/usr/share/man/man3/rdma_create_id.3.gz
/usr/share/man/man3/rdma_create_qp.3.gz
/usr/share/man/man3/rdma_create_srq.3.gz
/usr/share/man/man3/rdma_dereg_mr.3.gz
/usr/share/man/man3/rdma_destroy_ep.3.gz
/usr/share/man/man3/rdma_destroy_event_channel.3.gz
/usr/share/man/man3/rdma_destroy_id.3.gz
/usr/share/man/man3/rdma_destroy_qp.3.gz
/usr/share/man/man3/rdma_destroy_srq.3.gz
/usr/share/man/man3/rdma_disconnect.3.gz
/usr/share/man/man3/rdma_establish.3.gz
/usr/share/man/man3/rdma_event_str.3.gz
/usr/share/man/man3/rdma_free_devices.3.gz
/usr/share/man/man3/rdma_get_cm_event.3.gz
/usr/share/man/man3/rdma_get_devices.3.gz
/usr/share/man/man3/rdma_get_dst_port.3.gz
/usr/share/man/man3/rdma_get_local_addr.3.gz
/usr/share/man/man3/rdma_get_peer_addr.3.gz
/usr/share/man/man3/rdma_get_recv_comp.3.gz
/usr/share/man/man3/rdma_get_request.3.gz
/usr/share/man/man3/rdma_get_send_comp.3.gz
/usr/share/man/man3/rdma_get_src_port.3.gz
/usr/share/man/man3/rdma_getaddrinfo.3.gz
/usr/share/man/man3/rdma_init_qp_attr.3.gz
/usr/share/man/man3/rdma_join_multicast.3.gz
/usr/share/man/man3/rdma_join_multicast_ex.3.gz
/usr/share/man/man3/rdma_leave_multicast.3.gz
/usr/share/man/man3/rdma_listen.3.gz
/usr/share/man/man3/rdma_migrate_id.3.gz
/usr/share/man/man3/rdma_notify.3.gz
/usr/share/man/man3/rdma_post_read.3.gz
/usr/share/man/man3/rdma_post_readv.3.gz
/usr/share/man/man3/rdma_post_recv.3.gz
/usr/share/man/man3/rdma_post_recvv.3.gz
/usr/share/man/man3/rdma_post_send.3.gz
/usr/share/man/man3/rdma_post_sendv.3.gz
/usr/share/man/man3/rdma_post_ud_send.3.gz
/usr/share/man/man3/rdma_post_write.3.gz
/usr/share/man/man3/rdma_post_writev.3.gz
/usr/share/man/man3/rdma_reg_msgs.3.gz
/usr/share/man/man3/rdma_reg_read.3.gz
/usr/share/man/man3/rdma_reg_write.3.gz
/usr/share/man/man3/rdma_reject.3.gz
/usr/share/man/man3/rdma_resolve_addr.3.gz
/usr/share/man/man3/rdma_resolve_route.3.gz
/usr/share/man/man3/rdma_set_option.3.gz
/usr/share/man/man3/umad_addr_dump.3.gz
/usr/share/man/man3/umad_alloc.3.gz
/usr/share/man/man3/umad_attribute_str.3.gz
/usr/share/man/man3/umad_class_str.3.gz
/usr/share/man/man3/umad_close_port.3.gz
/usr/share/man/man3/umad_debug.3.gz
/usr/share/man/man3/umad_done.3.gz
/usr/share/man/man3/umad_dump.3.gz
/usr/share/man/man3/umad_free.3.gz
/usr/share/man/man3/umad_get_ca.3.gz
/usr/share/man/man3/umad_get_ca_portguids.3.gz
/usr/share/man/man3/umad_get_cas_names.3.gz
/usr/share/man/man3/umad_get_fd.3.gz
/usr/share/man/man3/umad_get_issm_path.3.gz
/usr/share/man/man3/umad_get_mad.3.gz
/usr/share/man/man3/umad_get_mad_addr.3.gz
/usr/share/man/man3/umad_get_pkey.3.gz
/usr/share/man/man3/umad_get_port.3.gz
/usr/share/man/man3/umad_init.3.gz
/usr/share/man/man3/umad_mad_status_str.3.gz
/usr/share/man/man3/umad_method_str.3.gz
/usr/share/man/man3/umad_open_port.3.gz
/usr/share/man/man3/umad_poll.3.gz
/usr/share/man/man3/umad_recv.3.gz
/usr/share/man/man3/umad_register.3.gz
/usr/share/man/man3/umad_register2.3.gz
/usr/share/man/man3/umad_register_oui.3.gz
/usr/share/man/man3/umad_release_ca.3.gz
/usr/share/man/man3/umad_release_port.3.gz
/usr/share/man/man3/umad_send.3.gz
/usr/share/man/man3/umad_set_addr.3.gz
/usr/share/man/man3/umad_set_addr_net.3.gz
/usr/share/man/man3/umad_set_grh.3.gz
/usr/share/man/man3/umad_set_grh_net.3.gz
/usr/share/man/man3/umad_set_pkey.3.gz
/usr/share/man/man3/umad_size.3.gz
/usr/share/man/man3/umad_status.3.gz
/usr/share/man/man3/umad_unregister.3.gz
/usr/share/man/man7/mlx4dv.7.gz
/usr/share/man/man7/mlx5dv.7.gz
/usr/share/man/man7/rdma_cm.7.gz

## 查询某个函数, 比如
# man ibv_get_device_list
```

### 工具

- **rdma**：RDMA 子系统管理工具
- **rxe_cfg**：软件 RDMA 配置工具，弃用，逐步被 rdma 命令替代
    

## perftest 包
### 包内容
```bash
# yum install -y perftest

# rpm -ql perftest
/usr/bin/ib_atomic_bw
/usr/bin/ib_atomic_lat
/usr/bin/ib_read_bw
/usr/bin/ib_read_lat
/usr/bin/ib_send_bw
/usr/bin/ib_send_lat
/usr/bin/ib_write_bw
/usr/bin/ib_write_lat
/usr/bin/raw_ethernet_bw
/usr/bin/raw_ethernet_lat
/usr/share/doc/perftest-4.2
/usr/share/doc/perftest-4.2/COPYING
/usr/share/doc/perftest-4.2/README
/usr/share/man/man1/ib_atomic_bw.1.gz
/usr/share/man/man1/ib_atomic_lat.1.gz
/usr/share/man/man1/ib_read_bw.1.gz
/usr/share/man/man1/ib_read_lat.1.gz
/usr/share/man/man1/ib_send_bw.1.gz
/usr/share/man/man1/ib_send_lat.1.gz
/usr/share/man/man1/ib_write_bw.1.gz
/usr/share/man/man1/ib_write_lat.1.gz
/usr/share/man/man1/raw_ethernet_bw.1.gz
/usr/share/man/man1/raw_ethernet_lat.1.gz
```

### 工具
- **ib_send_bw**：带宽测试工具
    
- **ib_read_bw**：RDMA read 带宽测试
    
- **ib_write_bw**：RDMA write 带宽测试
    
- **ib_send_lat**：延迟测试工具
    
- **ib_read_lat**：RDMA read 延迟测试
    
- **ib_write_lat**：RDMA write 延迟测试

## qperf
### 介绍
网络性能主要有两个指标是带宽和延时。`qperf`和`iperf/netperf`一样可以评测两个节点之间的带宽和延时。可以在测试`tcp/ip`协议和`RDMA`传输。不过相比`netperf`和`iperf`，支持`RDMA`是`qperf`工具的独有特性。



### 包内容
```bash
# yum install -y qperf

# rpm -ql qperf
/usr/bin/qperf
/usr/share/doc/qperf-0.4.9
/usr/share/doc/qperf-0.4.9/COPYING
/usr/share/man/man1/qperf.1.gz


# rpm -qR qperf
libc.so.6()(64bit)
libc.so.6(GLIBC_2.14)(64bit)
libc.so.6(GLIBC_2.15)(64bit)
libc.so.6(GLIBC_2.2.5)(64bit)
libc.so.6(GLIBC_2.3)(64bit)
libc.so.6(GLIBC_2.3.4)(64bit)
libc.so.6(GLIBC_2.4)(64bit)
libc.so.6(GLIBC_2.8)(64bit)
libibverbs.so.1()(64bit)
libibverbs.so.1(IBVERBS_1.0)(64bit)
libibverbs.so.1(IBVERBS_1.1)(64bit)
librdmacm.so.1()(64bit)
librdmacm.so.1(RDMACM_1.0)(64bit)
rpmlib(CompressedFileNames) <= 3.0.4-1
rpmlib(FileDigests) <= 4.6.0-1
rpmlib(PayloadFilesHavePrefix) <= 4.0-1
rtld(GNU_HASH)
rpmlib(PayloadIsXz) <= 5.2-1

如上所示，安装qperf，同时会安装两个依赖包(libibverbs, librdmacm),是直接和rdma功能相关的，不然无法启动rdma功能。
```

### 使用
`qperf`是测试两个节点之间的带宽和延时的，为此需要一个当作服务端，一个当作客户端。

其中服务端直接运行`qperf`, 无需任何参数。
```bash
#qperf  
默认开启端口号：19765  
通过netstat查看，如下:  
#netstat –tunlup  
tcp 0 0 0.0.0.0:19765 0.0.0.0:* LISTEN 53755/qperf
```

客户端运行获取带宽、延时情况，运行过程中不需要指定端口号，只要指定主机名或者`ip`地址即可。

![](attachments/Pasted%20image%2020250720124903.png)

```bash
RDMA Send/Receive
    rc_bi_bw                RC streaming two way bandwidth
    rc_bw                   RC streaming one way bandwidth
    rc_lat                  RC one way latency
    uc_bi_bw                UC streaming two way bandwidth
    uc_bw                   UC streaming one way bandwidth
    uc_lat                  UC one way latency
    ud_bi_bw                UD streaming two way bandwidth
    ud_bw                   UD streaming one way bandwidth
    ud_lat                  UD one way latency
    xrc_bi_bw               XRC streaming two way bandwidth
    xrc_bw                  XRC streaming one way bandwidth
    xrc_lat                 XRC one way latency
RDMA  write/read
    rc_rdma_read_bw         RC RDMA read streaming one way bandwidth
    rc_rdma_read_lat        RC RDMA read one way latency
    rc_rdma_write_bw        RC RDMA write streaming one way bandwidth
    rc_rdma_write_lat       RC RDMA write one way latency
    rc_rdma_write_poll_lat  RC RDMA write one way polling latency
    uc_rdma_write_bw        UC RDMA write streaming one way bandwidth
    uc_rdma_write_lat       UC RDMA write one way latency
    uc_rdma_write_poll_lat  UC RDMA write one way polling latency
InfiniBand Atomics
    rc_compare_swap_mr      RC compare and swap messaging rate
    rc_fetch_add_mr         RC fetch and add messaging rate
Verification
    ver_rc_compare_swap     Verify RC compare and swap
    ver_rc_fetch_add        Verify RC fetch and add
```

![](attachments/Pasted%20image%2020250720125455.png)

![](attachments/Pasted%20image%2020250720131043.png)

#### RDMA测试
如果网卡支持RDMA功能，例如IB卡，那么可以进行RDMA性能测试：
为了使用 RoCE 运行 qperf，应该在客户端添加 -cm1 标志。(mellonx)
```bash
-cm, --use_cm OnOff
	  Use the RDMA Connection Manager (CM) if OnOff is non-zero.  It is necessary to  use  the  CM
	  for  iWARP  devices.  The default is to establish the connection without using the CM.  This
	  only works for the tests that use the RC transport.

-cm1   Use RDMA Connection Manager.
```


```bash
服务端：
`qperf`

客户端
send/receive
`qperf -cm1 172.17.31.51 rc_lat`
`qperf -cm1 172.17.31.51 rc_bw`

write/read
`qperf -cm1 172.17.31.51`rc_rdma_write`_lat`
`qperf -cm1 172.17.31.51`rc_rdma_write_bw

数据包size： -m 
qperf -cm1 172.17.31.51 -m 1M rc_bw

数据包数量： -n
qperf -cm1 172.17.31.51 -m 1M -n 1000 rc_bw

polling或者wait event模式, -cq 1: 开始polling模式。
qperf -cm1 172.17.31.51 -m 1M -cp 0  rc_bw

设置持续时间：-t 默认是s， 加m、h、d 可表示分钟、小时、天
qperf  -cm1 172.17.31.51 -m 1M -t 20 rc_bw #20s
qperf  -cm1 172.17.31.51 -m 1M -t 20m rc_bw #20分钟
```

### 源码
参考：[linux-rdma/qperf](https://github.com/linux-rdma/qperf)



## iproute2 包
iproute2是一个强大的网络管理工具集(ip, tc, rdma, devlink等)，其中包括许多用于配置和管理网络设备的工具。iproute2套件中的RDMA 工具为管理和配置 RDMA 设备提供了强大和丰富的功能。

### 包内容
```bash
# yum  list installed |grep iproute2
mlnx-iproute2.x86_64           5.19.0-1.58101           installed


#  rpm -ql mlnx-iproute2
/opt/mellanox/iproute2/etc/bpf_pinning
/opt/mellanox/iproute2/etc/ematch_map
/opt/mellanox/iproute2/etc/group
/opt/mellanox/iproute2/etc/nl_protos
/opt/mellanox/iproute2/etc/rt_dsfield
/opt/mellanox/iproute2/etc/rt_protos
/opt/mellanox/iproute2/etc/rt_realms
/opt/mellanox/iproute2/etc/rt_scopes
/opt/mellanox/iproute2/etc/rt_tables
/opt/mellanox/iproute2/include/iproute2
/opt/mellanox/iproute2/include/iproute2/bpf_elf.h
/opt/mellanox/iproute2/include/libnetlink.h
/opt/mellanox/iproute2/lib64/libnetlink.a
/opt/mellanox/iproute2/lib64/tc
/opt/mellanox/iproute2/lib64/tc/experimental.dist
/opt/mellanox/iproute2/lib64/tc/normal.dist
/opt/mellanox/iproute2/lib64/tc/pareto.dist
/opt/mellanox/iproute2/lib64/tc/paretonormal.dist
/opt/mellanox/iproute2/sbin/arpd
/opt/mellanox/iproute2/sbin/bridge
/opt/mellanox/iproute2/sbin/ctstat
/opt/mellanox/iproute2/sbin/devlink
/opt/mellanox/iproute2/sbin/genl
/opt/mellanox/iproute2/sbin/ifcfg
/opt/mellanox/iproute2/sbin/ifstat
/opt/mellanox/iproute2/sbin/ip
/opt/mellanox/iproute2/sbin/lnstat
/opt/mellanox/iproute2/sbin/nstat
/opt/mellanox/iproute2/sbin/rdma
/opt/mellanox/iproute2/sbin/routef
/opt/mellanox/iproute2/sbin/routel
/opt/mellanox/iproute2/sbin/rtacct
/opt/mellanox/iproute2/sbin/rtmon
/opt/mellanox/iproute2/sbin/rtpr
/opt/mellanox/iproute2/sbin/rtstat
/opt/mellanox/iproute2/sbin/ss
/opt/mellanox/iproute2/sbin/tc
/opt/mellanox/iproute2/sbin/tipc
/opt/mellanox/iproute2/share/bash-completion
/opt/mellanox/iproute2/share/bash-completion/completions
/opt/mellanox/iproute2/share/bash-completion/completions/tc
/opt/mellanox/iproute2/share/doc/mlnx-iproute2-5.4.0
/opt/mellanox/iproute2/share/doc/mlnx-iproute2-5.4.0/README
/opt/mellanox/iproute2/share/doc/mlnx-iproute2-5.4.0/README.devel
/opt/mellanox/iproute2/share/doc/mlnx-iproute2-5.4.0/actions
/opt/mellanox/iproute2/share/doc/mlnx-iproute2-5.4.0/actions/actions-general
/opt/mellanox/iproute2/share/doc/mlnx-iproute2-5.4.0/actions/gact-usage
/opt/mellanox/iproute2/share/doc/mlnx-iproute2-5.4.0/actions/ifb-README
/opt/mellanox/iproute2/share/doc/mlnx-iproute2-5.4.0/actions/mirred-usage
/opt/mellanox/iproute2/share/man
/opt/mellanox/iproute2/share/man/man3
/opt/mellanox/iproute2/share/man/man3/libnetlink.3
/opt/mellanox/iproute2/share/man/man7
/opt/mellanox/iproute2/share/man/man7/tc-hfsc.7
/opt/mellanox/iproute2/share/man/man8
/opt/mellanox/iproute2/share/man/man8/arpd.8
/opt/mellanox/iproute2/share/man/man8/bridge.8
/opt/mellanox/iproute2/share/man/man8/ctstat.8
/opt/mellanox/iproute2/share/man/man8/devlink-dev.8
/opt/mellanox/iproute2/share/man/man8/devlink-health.8
/opt/mellanox/iproute2/share/man/man8/devlink-monitor.8
/opt/mellanox/iproute2/share/man/man8/devlink-port.8
/opt/mellanox/iproute2/share/man/man8/devlink-region.8
/opt/mellanox/iproute2/share/man/man8/devlink-resource.8
/opt/mellanox/iproute2/share/man/man8/devlink-sb.8
/opt/mellanox/iproute2/share/man/man8/devlink-trap.8
/opt/mellanox/iproute2/share/man/man8/devlink.8
/opt/mellanox/iproute2/share/man/man8/genl.8
/opt/mellanox/iproute2/share/man/man8/ifcfg.8
/opt/mellanox/iproute2/share/man/man8/ifstat.8
/opt/mellanox/iproute2/share/man/man8/ip-address.8
/opt/mellanox/iproute2/share/man/man8/ip-addrlabel.8
/opt/mellanox/iproute2/share/man/man8/ip-fou.8
/opt/mellanox/iproute2/share/man/man8/ip-gue.8
/opt/mellanox/iproute2/share/man/man8/ip-l2tp.8
/opt/mellanox/iproute2/share/man/man8/ip-link.8
/opt/mellanox/iproute2/share/man/man8/ip-macsec.8
/opt/mellanox/iproute2/share/man/man8/ip-maddress.8
/opt/mellanox/iproute2/share/man/man8/ip-monitor.8
/opt/mellanox/iproute2/share/man/man8/ip-mroute.8
/opt/mellanox/iproute2/share/man/man8/ip-neighbour.8
/opt/mellanox/iproute2/share/man/man8/ip-netconf.8
/opt/mellanox/iproute2/share/man/man8/ip-netns.8
/opt/mellanox/iproute2/share/man/man8/ip-nexthop.8
/opt/mellanox/iproute2/share/man/man8/ip-ntable.8
/opt/mellanox/iproute2/share/man/man8/ip-route.8
/opt/mellanox/iproute2/share/man/man8/ip-rule.8
/opt/mellanox/iproute2/share/man/man8/ip-sr.8
/opt/mellanox/iproute2/share/man/man8/ip-tcp_metrics.8
/opt/mellanox/iproute2/share/man/man8/ip-token.8
/opt/mellanox/iproute2/share/man/man8/ip-tunnel.8
/opt/mellanox/iproute2/share/man/man8/ip-vrf.8
/opt/mellanox/iproute2/share/man/man8/ip-xfrm.8
/opt/mellanox/iproute2/share/man/man8/ip.8
/opt/mellanox/iproute2/share/man/man8/lnstat.8
/opt/mellanox/iproute2/share/man/man8/nstat.8
/opt/mellanox/iproute2/share/man/man8/rdma-dev.8
/opt/mellanox/iproute2/share/man/man8/rdma-link.8
/opt/mellanox/iproute2/share/man/man8/rdma-resource.8
/opt/mellanox/iproute2/share/man/man8/rdma-statistic.8
/opt/mellanox/iproute2/share/man/man8/rdma-system.8
/opt/mellanox/iproute2/share/man/man8/rdma.8
/opt/mellanox/iproute2/share/man/man8/routef.8
/opt/mellanox/iproute2/share/man/man8/routel.8
/opt/mellanox/iproute2/share/man/man8/rtacct.8
/opt/mellanox/iproute2/share/man/man8/rtmon.8
/opt/mellanox/iproute2/share/man/man8/rtpr.8
/opt/mellanox/iproute2/share/man/man8/rtstat.8
/opt/mellanox/iproute2/share/man/man8/ss.8
/opt/mellanox/iproute2/share/man/man8/tc-actions.8
/opt/mellanox/iproute2/share/man/man8/tc-basic.8
/opt/mellanox/iproute2/share/man/man8/tc-bfifo.8
/opt/mellanox/iproute2/share/man/man8/tc-bpf.8
/opt/mellanox/iproute2/share/man/man8/tc-cake.8
/opt/mellanox/iproute2/share/man/man8/tc-cbq-details.8
/opt/mellanox/iproute2/share/man/man8/tc-cbq.8
/opt/mellanox/iproute2/share/man/man8/tc-cbs.8
/opt/mellanox/iproute2/share/man/man8/tc-cgroup.8
/opt/mellanox/iproute2/share/man/man8/tc-choke.8
/opt/mellanox/iproute2/share/man/man8/tc-codel.8
/opt/mellanox/iproute2/share/man/man8/tc-connmark.8
/opt/mellanox/iproute2/share/man/man8/tc-csum.8
/opt/mellanox/iproute2/share/man/man8/tc-ctinfo.8
/opt/mellanox/iproute2/share/man/man8/tc-drr.8
/opt/mellanox/iproute2/share/man/man8/tc-ematch.8
/opt/mellanox/iproute2/share/man/man8/tc-etf.8
/opt/mellanox/iproute2/share/man/man8/tc-flow.8
/opt/mellanox/iproute2/share/man/man8/tc-flower.8
/opt/mellanox/iproute2/share/man/man8/tc-fq.8
/opt/mellanox/iproute2/share/man/man8/tc-fq_codel.8
/opt/mellanox/iproute2/share/man/man8/tc-fw.8
/opt/mellanox/iproute2/share/man/man8/tc-hfsc.8
/opt/mellanox/iproute2/share/man/man8/tc-htb.8
/opt/mellanox/iproute2/share/man/man8/tc-ife.8
/opt/mellanox/iproute2/share/man/man8/tc-matchall.8
/opt/mellanox/iproute2/share/man/man8/tc-mirred.8
/opt/mellanox/iproute2/share/man/man8/tc-mpls.8
/opt/mellanox/iproute2/share/man/man8/tc-mqprio.8
/opt/mellanox/iproute2/share/man/man8/tc-nat.8
/opt/mellanox/iproute2/share/man/man8/tc-netem.8
/opt/mellanox/iproute2/share/man/man8/tc-pedit.8
/opt/mellanox/iproute2/share/man/man8/tc-pfifo.8
/opt/mellanox/iproute2/share/man/man8/tc-pfifo_fast.8
/opt/mellanox/iproute2/share/man/man8/tc-pie.8
/opt/mellanox/iproute2/share/man/man8/tc-police.8
/opt/mellanox/iproute2/share/man/man8/tc-prio.8
/opt/mellanox/iproute2/share/man/man8/tc-red.8
/opt/mellanox/iproute2/share/man/man8/tc-route.8
/opt/mellanox/iproute2/share/man/man8/tc-sample.8
/opt/mellanox/iproute2/share/man/man8/tc-sfb.8
/opt/mellanox/iproute2/share/man/man8/tc-sfq.8
/opt/mellanox/iproute2/share/man/man8/tc-simple.8
/opt/mellanox/iproute2/share/man/man8/tc-skbedit.8
/opt/mellanox/iproute2/share/man/man8/tc-skbmod.8
/opt/mellanox/iproute2/share/man/man8/tc-skbprio.8
/opt/mellanox/iproute2/share/man/man8/tc-stab.8
/opt/mellanox/iproute2/share/man/man8/tc-taprio.8
/opt/mellanox/iproute2/share/man/man8/tc-tbf.8
/opt/mellanox/iproute2/share/man/man8/tc-tcindex.8
/opt/mellanox/iproute2/share/man/man8/tc-tunnel_key.8
/opt/mellanox/iproute2/share/man/man8/tc-u32.8
/opt/mellanox/iproute2/share/man/man8/tc-vlan.8
/opt/mellanox/iproute2/share/man/man8/tc-xt.8
/opt/mellanox/iproute2/share/man/man8/tc.8
/opt/mellanox/iproute2/share/man/man8/tipc-bearer.8
/opt/mellanox/iproute2/share/man/man8/tipc-link.8
/opt/mellanox/iproute2/share/man/man8/tipc-media.8
/opt/mellanox/iproute2/share/man/man8/tipc-nametable.8
/opt/mellanox/iproute2/share/man/man8/tipc-node.8
/opt/mellanox/iproute2/share/man/man8/tipc-peer.8
/opt/mellanox/iproute2/share/man/man8/tipc-socket.8
/opt/mellanox/iproute2/share/man/man8/tipc.8

## 直接通过man文件来查看某个工具
# man /opt/mellanox/iproute2/share/man/man8/rdma-dev.8
```

### 工具

```bash
# /opt/mellanox/iproute2/sbin/rdma -h
Usage: rdma [ OPTIONS ] OBJECT { COMMAND | help }
       rdma [ -f[orce] ] -b[atch] filename
where  OBJECT := { dev | link | resource | system | statistic | help }
       OPTIONS := { -V[ersion] | -d[etails] | -j[son] | -p[retty]}
```

#### rdma dev


```bash
man /opt/mellanox/iproute2/share/man/man8/rdma-dev.8
```
![](attachments/Pasted%20image%2020250403125939.png)

```bash
# /opt/mellanox/iproute2/sbin/rdma dev  help
Usage: rdma dev show [DEV]
       rdma dev set [DEV] name DEVNAME
       rdma dev set [DEV] netns NSNAME
       rdma dev set [DEV] adaptive-moderation [on|off]
```

```bash
用来查询和修改 RDMA 设备的属性
$ rdma dev
0: mlx5_0: node_type ca fw 2.8.9999 node_guid 5254:00c0:fe12:3457 sys_image_guid 5254:00c0:fe12:3457
1: mlx5_1: node_type ca fw 2.8.9999 node_guid 5254:00c0:fe12:3458 sys_image_guid 5254:00c0:fe12:3458

rdma设备重命名和ns设置
rdma dev set mlx5_3 name rdma_0 # 将设备 mlx5_3 重命名为 rdma_0。
rdma dev set mlx5_3 netns foo # 将 RDMA 设备的网络命名空间更改为 foo，其中 foo 是之前使用 iproute2 ip 命令创建的。
```

#### rdma link

![](attachments/Pasted%20image%2020250403130210.png)

```bash
# /opt/mellanox/iproute2/sbin/rdma link help
Usage: rdma link show [DEV/PORT_INDEX]
Usage: rdma link add NAME type TYPE netdev NETDEV
Usage: rdma link delete NAME
```


```bash
用于查询和修改 RDMA 链路的属性
$ rdma link
link mlx5_0/1: state ACTIVE physical_state LINK_UP netdev ens21f0
link mlx5_1/1: state ACTIVE physical_state LINK_UP netdev ens21f1


创建和删除rxe模拟RoCE设备
rdma link add rxe_eth0 type rxe netdev eth0：向网络设备 eth0 添加名为 rxe_eth0 的 RXE 链路。
rdma link del rxe_eth0：移除 RXE 链路 rxe_eth0。
```

#### rdma resource

这个命令用于显示RDMA资源跟踪信息。如cm_id | cq | mr | pd | qp | ctx | srq 这些资源。

```bash
# /opt/mellanox/iproute2/sbin/rdma resource help
Usage: rdma resource
          resource show [DEV]
          resource show [qp|cm_id|pd|mr|cq]
          resource show qp link [DEV/PORT]
          resource show qp link [DEV/PORT] [FILTER-NAME FILTER-VALUE]
          resource show cm_id link [DEV/PORT]
          resource show cm_id link [DEV/PORT] [FILTER-NAME FILTER-VALUE]
          resource show cq link [DEV/PORT]
          resource show cq link [DEV/PORT] [FILTER-NAME FILTER-VALUE]
          resource show pd dev [DEV]
          resource show pd dev [DEV] [FILTER-NAME FILTER-VALUE]
          resource show mr dev [DEV]
          resource show mr dev [DEV] [FILTER-NAME FILTER-VALUE]

使用：
/opt/mellanox/iproute2/sbin/rdma resource show
/opt/mellanox/iproute2/sbin/rdma resource show qp
/opt/mellanox/iproute2/sbin/rdma resource show cq
/opt/mellanox/iproute2/sbin/rdma resource show mr
/opt/mellanox/iproute2/sbin/rdma resource show pd

```

```bash
# /opt/mellanox/iproute2/sbin/rdma resource show qp
link mlx5_0/1 lqpn 1 type GSI state RTS sq-psn 0 comm ib_core
link mlx5_0/1 lqpn 261 type UD state RTS sq-psn 0 comm ib_core
link mlx5_1/1 lqpn 1 type GSI state RTS sq-psn 0 comm ib_core
link mlx5_1/1 lqpn 2309 type UD state RTS sq-psn 0 comm ib_core
link mlx5_2/1 lqpn 1 type GSI state RTS sq-psn 0 comm ib_core
link mlx5_2/1 lqpn 137 type UD state RTS sq-psn 0 comm ib_core
link mlx5_3/1 lqpn 1 type GSI state RTS sq-psn 0 comm ib_core
link mlx5_3/1 lqpn 441 type UD state RTS sq-psn 0 comm ib_core


# /opt/mellanox/iproute2/sbin/rdma resource show cq
ifindex 0 ifname mlx5_0 cqn 1 cqe 1023 users 4 poll-ctx UNBOUND_WORKQUEUE comm ib_core
ifindex 0 ifname mlx5_0 cqn 2 cqe 255 users 2 poll-ctx SOFTIRQ comm mlx5_ib
ifindex 1 ifname mlx5_1 cqn 1 cqe 1023 users 4 poll-ctx UNBOUND_WORKQUEUE comm ib_core
ifindex 1 ifname mlx5_1 cqn 2 cqe 255 users 2 poll-ctx SOFTIRQ comm mlx5_ib
ifindex 2 ifname mlx5_2 cqn 1 cqe 1023 users 4 poll-ctx UNBOUND_WORKQUEUE comm ib_core
ifindex 2 ifname mlx5_2 cqn 2 cqe 255 users 2 poll-ctx SOFTIRQ comm mlx5_ib
ifindex 3 ifname mlx5_3 cqn 1 cqe 1023 users 4 poll-ctx UNBOUND_WORKQUEUE comm ib_core
ifindex 3 ifname mlx5_3 cqn 2 cqe 255 users 2 poll-ctx SOFTIRQ comm mlx5_ib


```

#### rdma system
用于RDMA子系统的全局设置，包括网络命名空间模式和特权 qkey 参数。

```bash
rdma system show  
显示系统上 RDMA 子系统网络命名空间模式的状态以及特权 qkey 参数的状态。

rdma system set netns exclusive
将 RDMA 子系统设置为网络命名空间独占模式。在这种模式下，RDMA 设备仅在单个网络命名空间中可见。
```


#### rdma statistic
用于查询和配置 RDMA 设备的统计信息。

注：==`rdma statistic show` 其实读取的是`/sys/class/infiniband/xx/hw_counters「比如：/sys/class/infiniband/mlx5_0/ports/1/hw_counters/」`目录下的统计数据==。

```bash
# man 8 rdma-statistic
# /opt/mellanox/iproute2/sbin/rdma statistic help
Usage: rdma [ OPTIONS ] statistic { COMMAND | help }
       rdma statistic OBJECT show
       rdma statistic OBJECT show link [ DEV/PORT_INDEX ] [ FILTER-NAME FILTER-VALUE ]
       rdma statistic OBJECT mode
       rdma statistic OBJECT set COUNTER_SCOPE [DEV/PORT_INDEX] auto {CRITERIA | off}
       rdma statistic OBJECT bind COUNTER_SCOPE [DEV/PORT_INDEX] [OBJECT-ID] [COUNTER-ID]
       rdma statistic OBJECT unbind COUNTER_SCOPE [DEV/PORT_INDEX] [COUNTER-ID]
       rdma statistic show
       rdma statistic show link [ DEV/PORT_INDEX ]
where  OBJECT: = { qp }
       CRITERIA : = { type }
       COUNTER_SCOPE: = { link | dev }
Examples:
       rdma statistic qp show
       rdma statistic qp show link mlx5_2/1
       rdma statistic qp mode
       rdma statistic qp mode link mlx5_0
       rdma statistic qp set link mlx5_2/1 auto type on
       rdma statistic qp set link mlx5_2/1 auto off
       rdma statistic qp bind link mlx5_2/1 lqpn 178
       rdma statistic qp bind link mlx5_2/1 lqpn 178 cntn 4
       rdma statistic qp unbind link mlx5_2/1 cntn 4
       rdma statistic qp unbind link mlx5_2/1 cntn 4 lqpn 178
       rdma statistic show
       rdma statistic show link mlx5_2/1
```


范例如下所示：
```bash
# /opt/mellanox/iproute2/sbin/rdma statistic show
link mlx5_0/1 rx_write_requests 0 rx_read_requests 0 rx_atomic_requests 0 out_of_buffer 135 out_of_sequence 0 duplicate_request 0 rnr_nak_retry_err 0 packet_seq_err 0 implied_nak_seq_err 0 local_ack_timeout_err 0 rx_dct_connect 0 resp_local_length_error 0 resp_cqe_error 1009 req_cqe_error 22 req_remote_invalid_request 0 req_remote_access_errors 0 resp_remote_access_errors 0 resp_cqe_flush_error 998 req_cqe_flush_error 11 roce_adp_retrans 0 roce_adp_retrans_to 0 roce_slow_restart 0 roce_slow_restart_cnps 0 roce_slow_restart_trans 0 rp_cnp_ignored 0 rp_cnp_handled 0 np_ecn_marked_roce_packets 0 np_cnp_sent 0 rx_icrc_encapsulated 0
link mlx5_1/1 rx_write_requests 0 rx_read_requests 0 rx_atomic_requests 0 out_of_buffer 0 out_of_sequence 0 duplicate_request 0 rnr_nak_retry_err 0 packet_seq_err 0 implied_nak_seq_err 0 local_ack_timeout_err 0 rx_dct_connect 0 resp_local_length_error 0 resp_cqe_error 0 req_cqe_error 0 req_remote_invalid_request 0 req_remote_access_errors 0 resp_remote_access_errors 0 resp_cqe_flush_error 0 req_cqe_flush_error 0 roce_adp_retrans 0 roce_adp_retrans_to 0 roce_slow_restart 0 roce_slow_restart_cnps 0 roce_slow_restart_trans 0 rp_cnp_ignored 0 rp_cnp_handled 0 np_ecn_marked_roce_packets 0 np_cnp_sent 0 rx_icrc_encapsulated 0
link mlx5_2/1 rx_write_requests 0 rx_read_requests 0 rx_atomic_requests 0 out_of_buffer 0 out_of_sequence 0 duplicate_request 0 rnr_nak_retry_err 0 packet_seq_err 0 implied_nak_seq_err 0 local_ack_timeout_err 0 rx_dct_connect 0 resp_local_length_error 0 resp_cqe_error 0 req_cqe_error 0 req_remote_invalid_request 0 req_remote_access_errors 0 resp_remote_access_errors 0 resp_cqe_flush_error 0 req_cqe_flush_error 0 roce_adp_retrans 0 roce_adp_retrans_to 0 roce_slow_restart 0 roce_slow_restart_cnps 0 roce_slow_restart_trans 0 rp_cnp_ignored 0 rp_cnp_handled 0 np_ecn_marked_roce_packets 0 np_cnp_sent 0 rx_icrc_encapsulated 0
link mlx5_3/1 rx_write_requests 0 rx_read_requests 0 rx_atomic_requests 0 out_of_buffer 0 out_of_sequence 0 duplicate_request 0 rnr_nak_retry_err 0 packet_seq_err 0 implied_nak_seq_err 0 local_ack_timeout_err 0 rx_dct_connect 0 resp_local_length_error 0 resp_cqe_error 0 req_cqe_error 0 req_remote_invalid_request 0 req_remote_access_errors 0 resp_remote_access_errors 0 resp_cqe_flush_error 0 req_cqe_flush_error 0 roce_adp_retrans 0 roce_adp_retrans_to 0 roce_slow_restart 0 roce_slow_restart_cnps 0 roce_slow_restart_trans 0 rp_cnp_ignored 0 rp_cnp_handled 0 np_ecn_marked_roce_packets 0 np_cnp_sent 0 rx_icrc_encapsulated 0
```




## librdmacm-utils 包
### 包内容
```bash
# rpm -qf /bin/rping
librdmacm-utils-58mlnx43-1.58101.x86_64

# rpm -ql librdmacm-utils
/usr/bin/cmtime
/usr/bin/mckey
/usr/bin/rcopy
/usr/bin/rdma_client
/usr/bin/rdma_server
/usr/bin/rdma_xclient
/usr/bin/rdma_xserver
/usr/bin/riostream
/usr/bin/rping
/usr/bin/rstream
/usr/bin/ucmatose
/usr/bin/udaddy
/usr/bin/udpong
/usr/share/man/man1/cmtime.1.gz
/usr/share/man/man1/mckey.1.gz
/usr/share/man/man1/rcopy.1.gz
/usr/share/man/man1/rdma_client.1.gz
/usr/share/man/man1/rdma_server.1.gz
/usr/share/man/man1/rdma_xclient.1.gz
/usr/share/man/man1/rdma_xserver.1.gz
/usr/share/man/man1/riostream.1.gz
/usr/share/man/man1/rping.1.gz
/usr/share/man/man1/rstream.1.gz
/usr/share/man/man1/ucmatose.1.gz
/usr/share/man/man1/udaddy.1.gz
/usr/share/man/man1/udpong.1.gz
```
### 工具
#### rping
- **rping**：`rdma-core`中`librdmacm-utils`提供了针对`rdma cm`接口的连通性检测工具，可以用于检测链路的连通性。


#### ucmatose
- **ucmatose**：UC 连接测试工具

#### rdma_client/rdma_server
- **rdma_client/rdma_server**：RDMA 客户端/服务器测试程序


## mlnx工具包：mlnx-ofa_kernel 包或者 mlnx-tools包
```bash
(1) 设备一
# which show_gids
/sbin/show_gids

# rpm -qf /sbin/show_gids
mlnx-tools-5.2.0-0.54310.x86_64



（2）设备二
# which show_gids
/sbin/show_gids

# rpm -qf /sbin/show_gids
mlnx-ofa_kernel-5.1-OFED.5.1.2.3.7.1.rhel7u4.x86_64
```
### 包内容
![](attachments/Pasted%20image%2020250403113036.png)


```bash
# rpm -ql mlnx-tools-5.2.0-0.54310.x86_64
/lib/udev/mlnx_bf_udev
/sbin/mlnx-sf
/sbin/mlnx_bf_configure
/sbin/mlnx_bf_configure_ct
/sbin/sysctl_perf_tuning
/usr/bin/mlnx_dump_parser
/usr/bin/mlnx_perf
/usr/bin/mlnx_qos
/usr/bin/mlx_fs_dump
/usr/bin/tc_wrap.py
/usr/lib/python2.7/site-packages/dcbnetlink.py
/usr/lib/python2.7/site-packages/dcbnetlink.pyc
/usr/lib/python2.7/site-packages/dcbnetlink.pyo
/usr/lib/python2.7/site-packages/genetlink.py
/usr/lib/python2.7/site-packages/genetlink.pyc
/usr/lib/python2.7/site-packages/genetlink.pyo
/usr/lib/python2.7/site-packages/mlnx_tools-5.2.0-py2.7.egg-info
/usr/lib/python2.7/site-packages/netlink.py
/usr/lib/python2.7/site-packages/netlink.pyc
/usr/lib/python2.7/site-packages/netlink.pyo
/usr/sbin/cma_roce_mode
/usr/sbin/cma_roce_tos
/usr/sbin/common_irq_affinity.sh
/usr/sbin/compat_gid_gen
/usr/sbin/ib2ib_setup
/usr/sbin/mlnx_affinity
/usr/sbin/mlnx_tune
/usr/sbin/set_irq_affinity.sh
/usr/sbin/set_irq_affinity_bynode.sh
/usr/sbin/set_irq_affinity_cpulist.sh
/usr/sbin/show_counters
/usr/sbin/show_gids
/usr/sbin/show_irq_affinity.sh
/usr/sbin/show_irq_affinity_hints.sh
/usr/share/doc/mlnx-tools-5.2.0
/usr/share/doc/mlnx-tools-5.2.0/ib2ib_setup.txt
/usr/share/man/man8/ib2ib_setup.8.gz
```

### 工具
#### ibdev2netdev
`ibdev2netdev`：RDMA端口和以太网端口的对应关系。

```bash
# ibdev2netdev
mlx5_0 port 1 ==> eth01 (Up)
mlx5_1 port 1 ==> eth02 (Down)
mlx5_2 port 1 ==> eth03 (Up)
mlx5_3 port 1 ==> eth04 (Up)

这个表明，存在4个RDMA端口，mlx5_0、mlx5_1、mlx5_2、mlx5_3; 
其中 mlx5_0 对应以太网端口 eth01 (ifconfig 可以看出来)
```

#### show_gids
##### 介绍
```bash
# which show_gids
/sbin/show_gids

# file /sbin/show_gids
/sbin/show_gids: Bourne-Again shell script, ASCII text executable


```
##### 使用场景
```c
（1）如下所示，本端获取gid时，需要指定 port_num, gid index。
/**
 * ibv_query_gid - Get a GID table entry.
 * ibv_query_gid() : returns the value of an index in The GID table of an RDMA device port's.
 */
int ibv_query_gid(struct ibv_context *context, uint8_t port_num,
          int index, union ibv_gid *gid);




(2) 如下所示，设置RTR时，需要设置 sgid_index。
struct ibv_qp_attr attr_rtr = {
    .qp_state = IBV_QPS_RTR,
    .path_mtu = IBV_MTU_1024,
    .rq_psn = 0,
    .dest_qp_num = dst.qp_num,
    .max_dest_rd_atomic = 1,
    .min_rnr_timer = 12,
    .ah_attr = {
        .is_global = 1,
        .port_num = 1,
        .grh = {.sgid_index = 1, .dgid = dst.gid},
    }};

ret = ibv_modify_qp(ib_qp, &attr_rtr,
                    IBV_QP_STATE | IBV_QP_AV | IBV_QP_PATH_MTU |
                        IBV_QP_DEST_QPN | IBV_QP_RQ_PSN |
                        IBV_QP_MIN_RNR_TIMER | IBV_QP_MAX_DEST_RD_ATOMIC);
```

##### gid的选择
`gid`的选择，需要保持可路由性。
如下所示：对于设备一，则选择 index为2或者3均可「存在对外可路由的 ipv4 地址」。对于设备二，则 hrn3_bond_0 网卡选择 index 为 1.

**选择的原则**
（1）一个RDMA设备可能有多个端口（但通常我们使用端口1）。
（2）每个端口可能有多个GID（Global Identifier），每个GID对应一个IP地址（IPv4或IPv6）。
（3）在连接建立时，客户端和服务器端需要选择兼容的GID（例如，相同的IP版本和网络可达性）。
（4）常见的做法是选择与连接对端IP地址类型相匹配的GID（例如，如果对端是IPv4，则选择IPv4 GID；如果是IPv6，则选择IPv6 GID）。
（5）另外，我们可能希望选择与对端IP地址在同一个子网的GID，或者选择RoCEv2的GID（因为RoCEv2同时支持IPv4和IPv6，但实际上RoCEv2使用IPv4或IPv6作为网络层，但数据包格式是固定的）。


##### 范例
```bash
（1）设备一：
# show_gids
DEV	PORT	INDEX	GID					IPv4  		VER	DEV
---	----	-----	---					------------  	---	---
mlx5_bond_0	1	0	fe80:0000:0000:0000:0e42:a1ff:fead:479a			v1	bond0
mlx5_bond_0	1	1	fe80:0000:0000:0000:0e42:a1ff:fead:479a			v2	bond0
mlx5_bond_0	1	2	0000:0000:0000:0000:0000:ffff:0a60:531a	10.96.83.26  	v1	bond0
mlx5_bond_0	1	3	0000:0000:0000:0000:0000:ffff:0a60:531a	10.96.83.26  	v2	bond0
n_gids_found=4

注：如上所示, v1、v2应该是指的是 ROCEV1(基于ethernet), ROCEV2(基于ipv4/ipv6).

（2）设备二：
# show_gids
DEV	PORT	INDEX	GID					IPv4  		VER	DEV
---	----	-----	---					------------  	---	---
hrn3_bond_0	1	0	fe80:0000:0000:0000:aedc:caff:fe6c:9312			v2	bond0
hrn3_bond_0	1	1	0000:0000:0000:0000:0000:ffff:1b69:b403	27.105.180.3  	v2	bond0
mlx5_bond_0	1	0	fe80:0000:0000:0000:0ac0:ebff:fe8c:3b44			v1	bond1
mlx5_bond_0	1	1	fe80:0000:0000:0000:0ac0:ebff:fe8c:3b44			v2	bond1
n_gids_found=4
```

##### 华为网卡的：hiroce3 gids
```bash
# which hiroce3
/usr/local/bin/hiroce3

# file /usr/local/bin/hiroce3
/usr/local/bin/hiroce3: Bourne-Again shell script, ASCII text executable

# rpm -qf /usr/local/bin/hiroce3
hiroce3-17.7.7.2_4.18.0_2.4.3.3.kwai.x86_64-1.el7.centos.x86_64

# rpm -ql hiroce3
/etc/depmod.d/hiroce3.conf
/etc/libibverbs.d/hrn3.driver
/etc/modules-load.d/hiroce3-modules.conf
/lib/modules/4.18.0-2.4.3.3.kwai.x86_64/extra/hiroce3/hiroce3.ko
/usr/lib64/libhrn3-rdmav.so
/usr/lib64/libhrn3-rdmav34.so
/usr/local/bin/hiroce3  


# ethtool -i eth04
driver: hinic3
version: 17.7.7.2
firmware-version: 17.7.7.2
expansion-rom-version:
bus-info: 0000:17:00.1
supports-statistics: yes
supports-test: yes
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: yes

说明：
	hiroce3: 对应的包的名称
	bin/hiroce3: 华为网卡的工具；
	libhrn3-rdmav34.so: rdma-core/hns（用户态）下的代码就是华为的providers(提供 Verbs API 的实际实现), 感觉这个像是对应hns的代码呢。
	hiroce3.ko：用户态驱动??
	/etc/libibverbs.d/hrn3.driver: `libibverbs` 就是通过读取这个配置文件，得知需要动态加载 `libhrn3-rdmav34.so` 来发现并驱动 `hrn3` 设备??
	
注：此中都没有libhrn3-rdmav34的静态库。

	hinic3: 网卡驱动名称。
```

```bash
# hiroce3 -h
	Usage: hiroce3 <major_cmd> [option]
		-h, --help	show help info

	Major Commands:
		gids	show all gids

# hiroce3 gids
ib_dev    	net_dev   	port idx  gid                                      type     ip              link
----------	----------	---- ---  ---------------------------------------  -------  --------------  ----
hrn3_bond_0	bond0     	1    0    fe80:0000:0000:0000:aedc:caff:fe6c:9312  RoCE v2                  up
hrn3_bond_0	bond0     	1    1    0000:0000:0000:0000:0000:ffff:1b69:b403  RoCE v2  27.105.180.3    up
mlx5_0    	eth02     	1    0    fe80:0000:0000:0000:0ac0:ebff:fe8c:3b44  RoCE v1                  down
mlx5_0    	eth02     	1    1    fe80:0000:0000:0000:0ac0:ebff:fe8c:3b44  RoCE v2                  down
mlx5_1    	eth03     	1    0    fe80:0000:0000:0000:0ac0:ebff:fe8c:3b45  RoCE v1                  down
mlx5_1    	eth03     	1    1    fe80:0000:0000:0000:0ac0:ebff:fe8c:3b45  RoCE v2                  down
```
###### 动态加载各个厂商的ibverb实现的库

`libibverbs.so` (作为核心库) 不直接链接到所有厂商的驱动库。它会扫描 `/etc/libibverbs.d/` 目录，通过读取类似 **`hrn3.driver`** 这样的配置文件，在运行时使用 `dlopen()` **动态加载** 对应的 Provider 库 (`libhrn3-rdmav34.so`)。

这个动态加载机制解释了为什么在 `ib_write_lat` 的 `ldd` 输出中看不到 `libhrn3-rdmav34.so`，可以看到`libibverbs.so`；
但华为网卡仍能正常工作。

#### show_counters
```bash
(1) 脚本文件
# which show_counters
/sbin/show_counters

# file /sbin/show_counters
/sbin/show_counters: Bourne-Again shell script, ASCII text executable


（2）使用方法
# show_counters
Valid command is: show_rdma_counters <rdma_device_name>
Example:
show_rdma_counters mlx5_0


# show_counters mlx5_bond_0
Port 1 hw counters:
duplicate_request: 0
implied_nak_seq_err: 0
lifespan: 10
local_ack_timeout_err: 0
np_cnp_sent: 1052612505
np_ecn_marked_roce_packets: 21278189006
out_of_buffer: 0
out_of_sequence: 1811101
packet_seq_err: 0
req_cqe_error: 0
req_cqe_flush_error: 0
req_remote_access_errors: 0
req_remote_invalid_request: 0
resp_cqe_error: 3293504
resp_cqe_flush_error: 3211234
resp_local_length_error: 0
resp_remote_access_errors: 0
rnr_nak_retry_err: 0
roce_adp_retrans: 0
roce_adp_retrans_to: 0
roce_slow_restart: 0
roce_slow_restart_cnps: 0
roce_slow_restart_trans: 0
rp_cnp_handled: 0
rp_cnp_ignored: 0
rx_atomic_requests: 0
rx_dct_connect: 0
rx_icrc_encapsulated: 0
rx_read_requests: 0
rx_write_requests: 2402134794


（3）原理
读取下面的文件的内容：

# ll /sys/class/infiniband/mlx5_bond_0/ports/1/hw_counters/
total 0
-r--r--r-- 1 root root 4096 Jul  1 14:38 duplicate_request
-r--r--r-- 1 root root 4096 Jul  1 14:38 implied_nak_seq_err
-rw-r--r-- 1 root root 4096 Jul  1 14:38 lifespan
-r--r--r-- 1 root root 4096 Jul  1 14:38 local_ack_timeout_err
-r--r--r-- 1 root root 4096 Jul  1 14:38 np_cnp_sent
-r--r--r-- 1 root root 4096 Jul  1 14:38 np_ecn_marked_roce_packets
-r--r--r-- 1 root root 4096 Jul  1 14:38 out_of_buffer
-r--r--r-- 1 root root 4096 Jul  1 14:38 out_of_sequence
-r--r--r-- 1 root root 4096 Jul  1 14:38 packet_seq_err
-r--r--r-- 1 root root 4096 Jul  1 14:38 req_cqe_error
-r--r--r-- 1 root root 4096 Jul  1 14:38 req_cqe_flush_error
-r--r--r-- 1 root root 4096 Jul  1 14:38 req_remote_access_errors
-r--r--r-- 1 root root 4096 Jul  1 14:38 req_remote_invalid_request
-r--r--r-- 1 root root 4096 Jul  1 14:38 resp_cqe_error
-r--r--r-- 1 root root 4096 Jul  1 14:38 resp_cqe_flush_error
-r--r--r-- 1 root root 4096 Jul  1 14:38 resp_local_length_error
-r--r--r-- 1 root root 4096 Jul  1 14:38 resp_remote_access_errors
-r--r--r-- 1 root root 4096 Jul  1 14:38 rnr_nak_retry_err
-r--r--r-- 1 root root 4096 Jul  1 14:38 roce_adp_retrans
-r--r--r-- 1 root root 4096 Jul  1 14:38 roce_adp_retrans_to
-r--r--r-- 1 root root 4096 Jul  1 14:38 roce_slow_restart
-r--r--r-- 1 root root 4096 Jul  1 14:38 roce_slow_restart_cnps
-r--r--r-- 1 root root 4096 Jul  1 14:38 roce_slow_restart_trans
-r--r--r-- 1 root root 4096 Jul  1 14:38 rp_cnp_handled
-r--r--r-- 1 root root 4096 Jul  1 14:38 rp_cnp_ignored
-r--r--r-- 1 root root 4096 Jul  1 14:38 rx_atomic_requests
-r--r--r-- 1 root root 4096 Jul  1 14:38 rx_dct_connect
-r--r--r-- 1 root root 4096 Jul  1 14:38 rx_icrc_encapsulated
-r--r--r-- 1 root root 4096 Jul  1 14:38 rx_read_requests
-r--r--r-- 1 root root 4096 Jul  1 14:38 rx_write_requests
```



### 范例
（1） 基础信息
```bash
# lspci |grep -i eth
2f:00.0 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
2f:00.1 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
30:00.0 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]
30:00.1 Ethernet controller: Mellanox Technologies MT27710 Family [ConnectX-4 Lx]

==============================
# ethtool -i eth01
driver: mlx5_core
version: 5.0-2.1.8
firmware-version: 14.28.2006 (MT_2420110034)
expansion-rom-version:
bus-info: 0000:2f:00.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: yes


# ethtool -i eth02
driver: mlx5_core
version: 5.0-2.1.8
firmware-version: 14.28.2006 (MT_2420110034)
expansion-rom-version:
bus-info: 0000:2f:00.1
supports-statistics: yes
supports-test: yes
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: yes


# ethtool -i eth03
driver: mlx5_core
version: 5.0-2.1.8
firmware-version: 14.32.1010 (MT_2420110034)
expansion-rom-version:
bus-info: 0000:30:00.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: yes


# ethtool -i eth04
driver: mlx5_core
version: 5.0-2.1.8
firmware-version: 14.32.1010 (MT_2420110034)
expansion-rom-version:
bus-info: 0000:30:00.1
supports-statistics: yes
supports-test: yes
supports-eeprom-access: no
supports-register-dump: no
supports-priv-flags: yes

===========================================
# ip a
6: eth01: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 0c:42:a1:40:29:a0 brd ff:ff:ff:ff:ff:ff
    inet 10.108.164.36/26 brd 10.108.164.63 scope global eth01
       valid_lft forever preferred_lft forever
    inet6 fe80::e42:a1ff:fe40:29a0/64 scope link
       valid_lft forever preferred_lft forever
7: eth02: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 0c:42:a1:40:29:a1 brd ff:ff:ff:ff:ff:ff
8: eth03: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 0c:42:a1:75:32:72 brd ff:ff:ff:ff:ff:ff
    inet 192.20.28.1/24 scope global eth03
       valid_lft forever preferred_lft forever
    inet6 fe80::e42:a1ff:fe75:3272/64 scope link
       valid_lft forever preferred_lft forever
9: eth04: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 0c:42:a1:75:32:73 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::e42:a1ff:fe75:3273/64 scope link
       valid_lft forever preferred_lft forever

```


```bash
# ibdev2netdev
mlx5_0 port 1 ==> eth01 (Up)
mlx5_1 port 1 ==> eth02 (Down)
mlx5_2 port 1 ==> eth03 (Up)
mlx5_3 port 1 ==> eth04 (Up)

# show_gids
DEV	PORT	INDEX	GID					IPv4  		VER	DEV
---	----	-----	---					------------  	---	---
mlx5_0	1	0	fe80:0000:0000:0000:0e42:a1ff:fe40:29a0			v1	eth01
mlx5_0	1	1	fe80:0000:0000:0000:0e42:a1ff:fe40:29a0			v2	eth01
mlx5_0	1	2	0000:0000:0000:0000:0000:ffff:0a6c:a424	10.108.164.36  	v1	eth01
mlx5_0	1	3	0000:0000:0000:0000:0000:ffff:0a6c:a424	10.108.164.36  	v2	eth01
mlx5_1	1	0	fe80:0000:0000:0000:0e42:a1ff:fe40:29a1			v1	eth02
mlx5_1	1	1	fe80:0000:0000:0000:0e42:a1ff:fe40:29a1			v2	eth02
mlx5_2	1	0	fe80:0000:0000:0000:0e42:a1ff:fe75:3272			v1	eth03
mlx5_2	1	1	fe80:0000:0000:0000:0e42:a1ff:fe75:3272			v2	eth03
mlx5_2	1	2	0000:0000:0000:0000:0000:ffff:c014:1c01	192.20.28.1  	v1	eth03
mlx5_2	1	3	0000:0000:0000:0000:0000:ffff:c014:1c01	192.20.28.1  	v2	eth03
mlx5_3	1	0	fe80:0000:0000:0000:0e42:a1ff:fe75:3273			v1	eth04
mlx5_3	1	1	fe80:0000:0000:0000:0e42:a1ff:fe75:3273			v2	eth04
n_gids_found=12

# ibv_devices
    device          	   node GUID
    ------          	----------------
    mlx5_0          	0c42a103004029a0
    mlx5_1          	0c42a103004029a1
    mlx5_2          	0c42a10300753272
    mlx5_3          	0c42a10300753273


# show_counters mlx5_0
Port 1 hw counters:
duplicate_request: 0
implied_nak_seq_err: 0
lifespan: 10
local_ack_timeout_err: 0
np_cnp_sent: 0
np_ecn_marked_roce_packets: 0
out_of_buffer: 135
out_of_sequence: 0
packet_seq_err: 0
req_cqe_error: 22
req_cqe_flush_error: 11
req_remote_access_errors: 0
req_remote_invalid_request: 0
resp_cqe_error: 1009
resp_cqe_flush_error: 998
resp_local_length_error: 0
resp_remote_access_errors: 0
rnr_nak_retry_err: 0
roce_adp_retrans: 0
roce_adp_retrans_to: 0
roce_slow_restart: 0
roce_slow_restart_cnps: 0
roce_slow_restart_trans: 0
rp_cnp_handled: 0
rp_cnp_ignored: 0
rx_atomic_requests: 0
rx_dct_connect: 0
rx_icrc_encapsulated: 0
rx_read_requests: 0
rx_write_requests: 0


# ibv_devinfo
hca_id:	mlx5_0
	transport:			InfiniBand (0)
	fw_ver:				14.28.2006
	node_guid:			0c42:a103:0040:29a0
	sys_image_guid:			0c42:a103:0040:29a0
	vendor_id:			0x02c9
	vendor_part_id:			4117
	hw_ver:				0x0
	board_id:			MT_2420110034
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		1024 (3)
			sm_lid:			0
			port_lid:		0
			port_lmc:		0x00
			link_layer:		Ethernet

hca_id:	mlx5_1
	transport:			InfiniBand (0)
	fw_ver:				14.28.2006
	node_guid:			0c42:a103:0040:29a1
	sys_image_guid:			0c42:a103:0040:29a0
	vendor_id:			0x02c9
	vendor_part_id:			4117
	hw_ver:				0x0
	board_id:			MT_2420110034
	phys_port_cnt:			1
		port:	1
			state:			PORT_DOWN (1)
			max_mtu:		4096 (5)
			active_mtu:		1024 (3)
			sm_lid:			0
			port_lid:		0
			port_lmc:		0x00
			link_layer:		Ethernet

hca_id:	mlx5_2
	transport:			InfiniBand (0)
	fw_ver:				14.32.1010
	node_guid:			0c42:a103:0075:3272
	sys_image_guid:			0c42:a103:0075:3272
	vendor_id:			0x02c9
	vendor_part_id:			4117
	hw_ver:				0x0
	board_id:			MT_2420110034
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		1024 (3)
			sm_lid:			0
			port_lid:		0
			port_lmc:		0x00
			link_layer:		Ethernet

hca_id:	mlx5_3
	transport:			InfiniBand (0)
	fw_ver:				14.32.1010
	node_guid:			0c42:a103:0075:3273
	sys_image_guid:			0c42:a103:0075:3272
	vendor_id:			0x02c9
	vendor_part_id:			4117
	hw_ver:				0x0
	board_id:			MT_2420110034
	phys_port_cnt:			1
		port:	1
			state:			PORT_ACTIVE (4)
			max_mtu:		4096 (5)
			active_mtu:		1024 (3)
			sm_lid:			0
			port_lid:		0
			port_lmc:		0x00
			link_layer:		Ethernet
```


## mellanox工具包：mft包
### 介绍
Mellanox MFT（Mellanox Firmware Tools）。MFT (Mellanox Firmware Tools) 是 Mellanox/NVIDIA 硬件生态系统中的一个核心工具包，**专用于硬件级别的诊断、配置和维护**。

#### 核心功能
MFT 是一套用于 Mellanox/NVIDIA HCA、网卡和交换机设备管理和维护的命令行工具集。

1. **固件管理 (Firmware Management):** 升级、备份、查询设备固件版本。
2. **诊断与健康检查 (Diagnostics):** 查看硬件状态、温度、链路速度、错误计数器。
3. **配置与烧录 (Configuration):** 修改硬件配置参数，如 MAC 地址、VLAN ID、PCI 设备 ID，并将这些配置固化到设备闪存中。
4. **设备扫描：** 发现系统中的所有 Mellanox 设备和 PCI 设备。

### 工作原理

MFT 工具通常通过访问特定的内核驱动接口 (`/dev/mst/`) 或直接与 HCA 上的管理接口通信。它独立于上层通信协议栈（如 UCX 或 MPI），直接操作设备硬件。


### 包内容

```bash
# rpm -ql  mft
....
/usr/bin/dimax_init
/usr/bin/flint
/usr/bin/flint_ext
/usr/bin/fwtrace
/usr/bin/i2c
/usr/bin/itrace
/usr/bin/mcra
/usr/bin/mdevices_info
/usr/bin/mft_uninstall.sh
/usr/bin/mget_temp
/usr/bin/mget_temp_ext
/usr/bin/minit
/usr/bin/mlxburn
/usr/bin/mlxburn_old
/usr/bin/mlxcables
/usr/bin/mlxcables_ext
/usr/bin/mlxconfig
/usr/bin/mlxdump
/usr/bin/mlxdump_ext
/usr/bin/mlxfwmanager
/usr/bin/mlxfwreset
/usr/bin/mlxfwstress
/usr/bin/mlxfwstress_ext
/usr/bin/mlxgearbox
/usr/bin/mlxi2c
/usr/bin/mlxlink
/usr/bin/mlxlink_ext
/usr/bin/mlxmcg
/usr/bin/mlxmdio
/usr/bin/mlxpci
/usr/bin/mlxphyburn
/usr/bin/mlxprivhost
/usr/bin/mlxreg
/usr/bin/mlxreg_ext
/usr/bin/mlxtokengenerator
/usr/bin/mlxtrace
/usr/bin/mlxtrace_ext
/usr/bin/mlxuptime
/usr/bin/mlxvpd
/usr/bin/mremote
/usr/bin/mst
/usr/bin/mst_cable
/usr/bin/mst_ib_add
/usr/bin/mstdump
/usr/bin/mstop
/usr/bin/mtserver
/usr/bin/pckt_drop
/usr/bin/resourcedump
/usr/bin/resourceparse
/usr/bin/stedump
/usr/bin/wqdump
/usr/bin/wqdump_ext
....

```

### `mst` (Mellanox Status Tool)
`mst` 是 MFT 的核心设备管理工具，用于扫描系统中的 Mellanox 设备并建立设备映射。

|**命令**|**描述**|**用途**|
|---|---|---|
|**`mst start`**|**启动** MFT 设备访问驱动。这是运行其他 MFT 工具的前提。|必须在执行 `flint` 等命令前运行。|
|**`mst stop`**|**停止** MFT 设备访问驱动。|清理资源。|
|**`mst status`**|**显示**当前系统中所有 Mellanox/NVIDIA 设备的列表和状态。|确认 HCA 是否被 MFT 正确识别。|
|**`mst devices`**|**显示**可用的设备节点（如 `/dev/mst/mt4115_pciconf0`）。|用于确定 `flint` 等工具的操作目标。|


#### 范例

```bash
# mst status -v
MST modules:
------------
    MST PCI module is not loaded
    MST PCI configuration module is not loaded
PCI devices:
------------
DEVICE_TYPE             MST      PCI       RDMA            NET                       NUMA
ConnectX6DX(rev:0)      NA       17:00.1   mlx5_bond_1     net-bond1                 0

ConnectX4LX(rev:0)      NA       31:00.0   mlx5_bond_0     net-bond0                 0

ConnectX4LX(rev:0)      NA       31:00.1   mlx5_bond_0     net-bond0                 0

ConnectX6DX(rev:0)      NA       17:00.0   mlx5_bond_1     net-bond1                 0
```

![](attachments/Pasted%20image%2020251127120253.png)


### `flint` (Firmware Linux Tool)
`flint` 是 MFT 中用于**固件操作**的瑞士军刀，是处理设备固件（Firmware）最常用的工具。`flint` 是最底层的固件管理工具。
功能：
- 查看固件版本
- 备份固件
- 强制烧录固件（注意危险）

|**命令**|**描述**|**用途**|
|---|---|---|
|**`flint -d <device> q`**|**查询 (Query)** 设备的当前固件版本。|查看当前硬件上运行的固件版本号。|
|**`flint -d <device> i <file.bin>`**|**信息 (Image Info)**：查看固件映像文件 (`.bin`) 的详细信息。|验证要烧录的固件文件是否有效。|
|**`flint -d <device> r <file.bin>`**|**读取 (Read/Backup)** 设备的当前固件，并保存为文件。|在升级前备份当前固件。|
|**`flint -d <device> -i <file.bin> b`**|**烧录 (Burn)** 新的固件映像文件到 HCA。|执行固件升级操作。|
|**`flint -d <device> se <guid> w`**|**设置 E-SID**：设置或修改设备的全局唯一标识符 (GUID)。|用于网络配置或更换 GUID。|

### `mlxconfig`
`mlxconfig`：查询或设置网卡配置。
功能：

- 查看/设置卡的配置项，比如：
    - SR-IOV 开关
    - VPI 模式（ETH/IB）
    - RoCE（V1/V2）
    - LRO/MTU/TSO 相关项
    - NIC Mode / Device Mode / Steering Mode

### mlxlink
`mlxlink` 专注于**链路状态和物理层信息**。

- **功能：** 查看 HCA 端口的详细链路状态、光模块（Transceiver）信息、信号质量、错误计数器等。
- **用途：** 诊断物理链路问题，如光模块故障、线缆质量问题或协商速率不匹配。
- 

### MFT 和 MLNX_OFED 的关系

**MFT (Mellanox Firmware Tools)** 和 **MLNX_OFED (Mellanox OpenFabrics Enterprise Distribution)** 是两个完全不同、但又相互依赖的软件组件。它们的关系可以概括为：**MLNX_OFED 负责通信功能和驱动，而 MFT 负责硬件管理和诊断。**


（1）对比

|**特性**|**MFT (Mellanox Firmware Tools)**|**MLNX_OFED (Mellanox OFED)**|
|---|---|---|
|**全称**|Mellanox Firmware Tools|Mellanox OpenFabrics Enterprise Distribution|
|**主要作用**|**硬件管理、诊断、固件操作。**|**通信驱动、协议栈、性能优化。**|
|**功能领域**|**硬件层 (Firmware/HCA)**|**软件层 (Kernel/Userspace)**|
|**核心工具**|`flint`, `mst` (Mellanox Status Tool)|`ibdev2netdev`, `ibstatus`, `perftest` utilities|
|**安装对象**|HCA 卡、网卡、交换机固件|Linux 内核、操作系统|
|**依赖关系**|可以独立于 OFED 安装和使用，但需要操作系统支持。|**必须安装**才能启用 RDMA/IB/RoCE 通信功能。|
|**用途示例**|1. 升级 HCA 卡的 **固件 (Firmware)**。 2. 查看 HCA 的温度、速率、序列号。 3. 烧录硬件配置。|1. 提供 **Kernel Driver** (如 `mlx5_ib` 或 `mlx4_ib`)。 2. 提供用户空间库 (如 `libibverbs`, `librdmacm`)。 3. 提供 UCX、MPI 等通信库的基础驱动支持。|



（2）它们的关系 (依赖与协作)：


|**场景**|**必备组件**|**解释**|
|---|---|---|
|**日常通信**|**MLNX_OFED**|必须有驱动和库来运行 MPI/UCX 应用程序。MFT 不参与数据传输。|
|**固件升级**|**MFT**|需要 MFT 中的 `flint` 工具来将新的固件映像文件烧录到 HCA 硬件中。|
|**硬件诊断**|**MFT + MLNX_OFED**|你需要 MLNX_OFED 驱动来让 HCA 正常工作，但你需要 MFT 工具 (`mst`) 来查询 HCA 内部状态、温度、序列号等诊断信息。|
|**性能测试**|**MLNX_OFED**|需要 MLNX_OFED 提供的 `ib_write_bw` 或 `perftest` 工具，直接使用 RDMA 接口进行测试。|


## 华为工具包：hinicadm3
```bash
# which hinicadm3
/sbin/hinicadm3

# rpm -qf /sbin/hinicadm3
hinicadm3-17.7.7.2-1.x86_64
```
### 包内容
```bash
# rpm -ql hinicadm3-17.7.7.2-1.x86_64
/opt/hinic/driver/libhicsrdump.so
/usr/sbin/hinicadm3
```
### 工具
#### hinicadm3
```bash
hinicadm3 counter -i hinic0 -t 1 -x 4
```


# ib网络状态探测
在 InfiniBand 网络中，Host Channel Adapter（HCA）是关键组件，了解其状态和配置对于网络管理和故障排查至关重要。以下是一些常用的命令，用于查询和管理 HCA 的状态和配置。
## ibstatus
## ibstat

### 介绍
显示 HCA 的基本状态信息，包括设备状态、端口状态、链路速度等。
ibstat 输出：包括 HCA 的名称、固件版本、端口状态（如 `PORT_ACTIVE`）、最大和活动 MTU 等。


### 安装
```bash
# which ibstat
/sbin/ibstat

# rpm -qf /sbin/ibstat
infiniband-diags-51mlnx1-1.51237.x86_64
```


## ibportstate
## ibv_devinfo
## iblinkinfo
## ibqueryerrors
## ibtracert

# ping-pong测试工具
```bash
# ibv_
ibv_asyncwatch     ibv_devices        ibv_devinfo        ibv_rc_pingpong    ibv_srq_pingpong   ibv_uc_pingpong    ibv_ud_pingpong    ibv_xsrq_pingpong

# which ibv_rc_pingpong
/bin/ibv_rc_pingpong

# rpm -qf /bin/ibv_rc_pingpong
libibverbs-utils-58mlnx43-1.58604.x86_64
```
# 网卡统计计数
## mellanox的RNIC网卡的统计
### 参考
```bash
# ib out_of_buffer问题分析
https://storage.blog.csdn.net/article/details/145187153
```


# 参考
```bash
# ib网络状态探测
https://storage.blog.csdn.net/article/details/145699299

# RDMA网络编程：管理工具和测试工具
https://mp.weixin.qq.com/s/NzINx-Oa7iWqAKYc82EzSg

# RDMA(17)工具利器：技术讲得再好也不如这些工具来的实在
https://mp.weixin.qq.com/s/I2sSY-XI0twvFa3F0f_VRA
```