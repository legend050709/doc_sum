```table-of-contents
```
# 介绍
# 安装和编译
```bash
wget https://github.com/linux-rdma/perftest

yum install -y automake libtool pciutils-devel

cd perftest/

./autogen.sh

./configure

make -j 9

当前 目录下输出查看：
# ll ib_*
-rwxr-xr-x 1 root root 1042776 Mar 14 16:56 ib_atomic_bw
-rwxr-xr-x 1 root root 1036728 Mar 14 16:56 ib_atomic_lat
-rwxr-xr-x 1 root root 1042664 Mar 14 16:56 ib_read_bw
-rwxr-xr-x 1 root root 1035248 Mar 14 16:56 ib_read_lat
-rwxr-xr-x 1 root root 1088552 Mar 14 16:56 ib_send_bw
-rwxr-xr-x 1 root root 1086496 Mar 14 16:56 ib_send_lat
-rwxr-xr-x 1 root root 1044656 Mar 14 16:56 ib_write_bw
-rwxr-xr-x 1 root root 1036568 Mar 14 16:56 ib_write_lat
```

# 工具集介绍
```bash
ib_send_lat   latency test with send transactions
ib_send_bw    bandwidth test with send transactions
ib_write_lat  latency test with RDMA write transactions
ib_write_bw   bandwidth test with RDMA write transactions
ib_read_lat   latency test with RDMA read transactions
ib_read_bw    bandwidth test with RDMA read transactions
ib_atomic_lat latency test with atomic transactions
ib_atomic_bw  bandwidth test with atomic transactions
```


# 调优
## NUMA调优
### 设备信息
记录一次 numa 的调优，服务端的机器情况如下
```bash
CPU 为 AMD EPYC 7763 64-Core Processor
网卡为 Infiniband 200G 网卡, 网卡归属在 CPU1
```
CPU 信息如下，一个 CPU 有 64 个核心，2个CPU，有128个core：
```bash
# lscpu 
 Architecture:                       x86_64
 CPU op-mode(s):                     32-bit, 64-bit
 Byte Order:                         Little Endian
 Address sizes:                      48 bits physical, 48 bits virtual
 CPU(s):                             128
 On-line CPU(s) list:                0-127
 Thread(s) per core:                 1
 Core(s) per socket:                 64
 Socket(s):                          2
 NUMA node(s):                       8
 Vendor ID:                          AuthenticAMD
 CPU family:                         25
 Model:                              1
 Model name:                         AMD EPYC 7763 64-Core Processor
 Stepping:                           1
 Frequency boost:                    enabled
 CPU MHz:                            1795.906
 CPU max MHz:                        2450.0000
 CPU min MHz:                        1500.0000
 BogoMIPS:                           4900.31
 Virtualization:                     AMD-V
 L1d cache:                          4 MiB
 L1i cache:                          4 MiB
 L2 cache:                           64 MiB
 L3 cache:                           512 MiB
 NUMA node0 CPU(s):                  0-15
 NUMA node1 CPU(s):                  16-31
 NUMA node2 CPU(s):                  32-47
 NUMA node3 CPU(s):                  48-63
 NUMA node4 CPU(s):                  64-79
 NUMA node5 CPU(s):                  80-95
 NUMA node6 CPU(s):                  96-111
 NUMA node7 CPU(s):                  112-127
 Vulnerability Gather data sampling: Not affected
 Vulnerability Itlb multihit:        Not affected
 Vulnerability L1tf:                 Not affected
 Vulnerability Mds:                  Not affected
 Vulnerability Meltdown:             Not affected
 Vulnerability Mmio stale data:      Not affected
 Vulnerability Retbleed:             Not affected
 Vulnerability Spec store bypass:    Mitigation; Speculative Store Bypass disabled via prctl and seccomp
 Vulnerability Spectre v1:           Mitigation; usercopy/swapgs barriers and __user pointer sanitization
 Vulnerability Spectre v2:           Mitigation; Retpolines, IBPB conditional, IBRS_FW, STIBP disabled, RSB filling, PBRSB-eIBRS Not affected
 Vulnerability Srbds:                Not affected
 Vulnerability Tsx async abort:      Not affected
 Flags:                              fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx mmxext fxsr_opt pdpe1gb rdtscp lm constan
                                     t_tsc rep_good nopl nonstop_tsc cpuid extd_apicid aperfmperf pni pclmulqdq monitor ssse3 fma cx16 pcid sse4_1 sse4_2 movbe popcnt aes xsave avx f16c rdra
                                     nd lahf_lm cmp_legacy svm extapic cr8_legacy abm sse4a misalignsse 3dnowprefetch osvw ibs skinit wdt tce topoext perfctr_core perfctr_nb bpext perfctr_ll
                                     c mwaitx cpb cat_l3 cdp_l3 invpcid_single hw_pstate ssbd mba ibrs ibpb stibp vmmcall fsgsbase bmi1 avx2 smep bmi2 invpcid cqm rdt_a rdseed adx smap clflu
                                     shopt clwb sha_ni xsaveopt xsavec xgetbv1 xsaves cqm_llc cqm_occup_llc cqm_mbm_total cqm_mbm_local clzero irperf xsaveerptr wbnoinvd arat npt lbrv svm_lo
                                     ck nrip_save tsc_scale vmcb_clean flushbyasid decodeassists pausefilter pfthreshold v_vmsave_vmload vgif umip pku ospke vaes vpclmulqdq rdpid overflow_re
                                     cov succor smca
```


### IB 网卡测试(不设置 NUMA)
不设置 NUMA，直接启动 IB 网卡测试：

（1）启动 IB 测试服务端
```bash
# ib_write_bw -a  
   
 ************************************  
 * Waiting for client to connect... *  
 ************************************
```


（2）检查进程所在 CPU (可以看到进程在 CPU0 的核心上)
```bash
# ps -eo cmd,psr | grep ib_write_bw  
 ib_write_bw -a               49
```

(3) 启动 IB 测试客户端
```bash
# ib_write_bw -a x.x.x.x
 ---------------------------------------------------------------------------------------
                     RDMA_Write BW Test
  Dual-port       : OFF      Device         : mlx5_0
  Number of qps   : 1        Transport type : IB
  Connection type : RC       Using SRQ      : OFF
  PCIe relax order: ON
  ibv_wr* API     : ON
  TX depth        : 128
  CQ Moderation   : 100
  Mtu             : 4096[B]
  Link type       : IB
  Max inline data : 0[B]
  rdma_cm QPs     : OFF
  Data ex. method : Ethernet
 ---------------------------------------------------------------------------------------
  local address: LID 0x1f9 QPN 0x0180 PSN 0xbee21d RKey 0x006dc5 VAddr 0x007f5bd95e2000
  remote address: LID 0x1f7 QPN 0x5864 PSN 0x7a35c6 RKey 0x00624f VAddr 0x007f746bc41000
 ---------------------------------------------------------------------------------------
  #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 Conflicting CPU frequency values detected: 1496.133000 != 1529.788000. CPU Frequency is not max.
  2          5000             4.26               3.88           2.036842
 Conflicting CPU frequency values detected: 1472.079000 != 1367.578000. CPU Frequency is not max.
  4          5000             8.52               8.49           2.226588
 Conflicting CPU frequency values detected: 1492.201000 != 2599.097000. CPU Frequency is not max.
  8          5000             16.99              16.96          2.222746
 Conflicting CPU frequency values detected: 1493.384000 != 2599.755000. CPU Frequency is not max.
  16         5000             34.39              34.35          2.251356
 Conflicting CPU frequency values detected: 1522.760000 != 1489.507000. CPU Frequency is not max.
  32         5000             68.72              68.71          2.251540
 Conflicting CPU frequency values detected: 1493.744000 != 2599.238000. CPU Frequency is not max.
  64         5000             137.19             126.25         2.068451
 Conflicting CPU frequency values detected: 1495.813000 != 2599.455000. CPU Frequency is not max.
  128        5000             274.12             273.70         2.242156
 Conflicting CPU frequency values detected: 1480.229000 != 2599.223000. CPU Frequency is not max.
  256        5000             499.29             496.87         2.035180
 Conflicting CPU frequency values detected: 1498.459000 != 2598.913000. CPU Frequency is not max.
  512        5000             1007.82            1007.51        2.063390
 Conflicting CPU frequency values detected: 1511.641000 != 1471.462000. CPU Frequency is not max.
  1024       5000             1861.92            1861.32        1.905990
 Conflicting CPU frequency values detected: 1493.094000 != 2599.077000. CPU Frequency is not max.
  2048       5000             2286.24            1975.36        1.011383
 Conflicting CPU frequency values detected: 1466.259000 != 1496.885000. CPU Frequency is not max.
  4096       5000             9513.14            2561.96        0.655862
 Conflicting CPU frequency values detected: 1496.754000 != 2599.334000. CPU Frequency is not max.
  8192       5000             10105.87            3496.79           0.447589
 Conflicting CPU frequency values detected: 1511.467000 != 2599.230000. CPU Frequency is not max.
  16384      5000             10586.62            5599.69           0.358380
 Conflicting CPU frequency values detected: 1499.808000 != 2599.680000. CPU Frequency is not max.
  32768      5000             10742.48            7679.79           0.245753
 Conflicting CPU frequency values detected: 1497.353000 != 2599.184000. CPU Frequency is not max.
  65536      5000             10765.91            8902.73           0.142444
 Conflicting CPU frequency values detected: 1466.773000 != 1499.932000. CPU Frequency is not max.
  131072     5000             10771.95            9920.94           0.079368
 Conflicting CPU frequency values detected: 1493.656000 != 2598.997000. CPU Frequency is not max.
  262144     5000             10774.35            10303.05          0.041212
 Conflicting CPU frequency values detected: 1518.926000 != 1488.449000. CPU Frequency is not max.
  524288     5000             10770.42            10572.72          0.021145
 Conflicting CPU frequency values detected: 1498.174000 != 2599.424000. CPU Frequency is not max.
  1048576    5000             10754.23            10669.84          0.010670
 Conflicting CPU frequency values detected: 1501.071000 != 1569.932000. CPU Frequency is not max.
  2097152    5000             10743.48            10728.88          0.005364
 Conflicting CPU frequency values detected: 1494.374000 != 1401.108000. CPU Frequency is not max.
  4194304    5000             10763.32            10760.67          0.002690
 Conflicting CPU frequency values detected: 1520.651000 != 1475.312000. CPU Frequency is not max.
  8388608    5000             10788.20            10787.86          0.001348
 ---------------------------------------------------------------------------------------
```

(4) server端的结果
```bash
# ib_write_bw -a
 
 ************************************
 * Waiting for client to connect... *
 ************************************
 ---------------------------------------------------------------------------------------
                     RDMA_Write BW Test
  Dual-port       : OFF      Device         : mlx5_0
  Number of qps   : 1        Transport type : IB
  Connection type : RC       Using SRQ      : OFF
  PCIe relax order: ON
  ibv_wr* API     : ON
  CQ Moderation   : 100
  Mtu             : 4096[B]
  Link type       : IB
  Max inline data : 0[B]
  rdma_cm QPs     : OFF
  Data ex. method : Ethernet
 ---------------------------------------------------------------------------------------
  local address: LID 0x1f7 QPN 0x5864 PSN 0x7a35c6 RKey 0x00624f VAddr 0x007f746bc41000
  remote address: LID 0x1f9 QPN 0x0180 PSN 0xbee21d RKey 0x006dc5 VAddr 0x007f5bd95e2000
 ---------------------------------------------------------------------------------------
  #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
  8388608    5000             10788.20            10787.86          0.001348
 ---------------------------------------------------------------------------------------
```

### IB 网卡测试 (设置 numa)
(1) 启动 IB 网卡测试服务端 (设置 numa，将进程绑定在 CPU1 的核心上)
```bash
# numactl -N 7 ib_write_bw -a
 
 ************************************
 * Waiting for client to connect... *
 ************************************
```

（2）查看进程所在 CPU (可以看到进程在 CPU1 的核心上)
```bash
# ps -eo cmd,psr | grep ib_write_bw  
 ib_write_bw -a              113
```

（3）启动 IB 测试客户端
```bash
# ib_write_bw -a x.x.x.x
 ---------------------------------------------------------------------------------------
                     RDMA_Write BW Test
  Dual-port       : OFF      Device         : mlx5_0
  Number of qps   : 1        Transport type : IB
  Connection type : RC       Using SRQ      : OFF
  PCIe relax order: ON
  ibv_wr* API     : ON
  TX depth        : 128
  CQ Moderation   : 100
  Mtu             : 4096[B]
  Link type       : IB
  Max inline data : 0[B]
  rdma_cm QPs     : OFF
  Data ex. method : Ethernet
 ---------------------------------------------------------------------------------------
  local address: LID 0x1f9 QPN 0x0182 PSN 0xb97c74 RKey 0x006dc7 VAddr 0x007f1c732df000
  remote address: LID 0x1f7 QPN 0x5866 PSN 0xb96bc RKey 0x006251 VAddr 0x007fdda610c000
 ---------------------------------------------------------------------------------------
  #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 Conflicting CPU frequency values detected: 1500.927000 != 1379.045000. CPU Frequency is not max.
  2          5000             4.40               3.90           2.046438
 Conflicting CPU frequency values detected: 1495.980000 != 1369.271000. CPU Frequency is not max.
  4          5000             8.75               8.73           2.289451
 Conflicting CPU frequency values detected: 1499.375000 != 3249.769000. CPU Frequency is not max.
  8          5000             17.57              17.54          2.298830
 Conflicting CPU frequency values detected: 1471.789000 != 1501.354000. CPU Frequency is not max.
  16         5000             35.37              35.34          2.315900
 Conflicting CPU frequency values detected: 1499.840000 != 3249.723000. CPU Frequency is not max.
  32         5000             70.80              70.76          2.318764
 Conflicting CPU frequency values detected: 1485.819000 != 1525.796000. CPU Frequency is not max.
  64         5000             140.14             125.18         2.050997
 Conflicting CPU frequency values detected: 1508.828000 != 1476.094000. CPU Frequency is not max.
  128        5000             284.29             282.80         2.316675
 Conflicting CPU frequency values detected: 1494.598000 != 3249.678000. CPU Frequency is not max.
  256        5000             523.31             522.00         2.138095
 Conflicting CPU frequency values detected: 1496.531000 != 3249.598000. CPU Frequency is not max.
  512        5000             1026.86            1026.28        2.101823
 Conflicting CPU frequency values detected: 1498.270000 != 3249.644000. CPU Frequency is not max.
  1024       5000             1886.88            1885.12        1.930367
 Conflicting CPU frequency values detected: 1497.365000 != 3249.567000. CPU Frequency is not max.
  2048       5000             2527.77            2328.56        1.192221
 Conflicting CPU frequency values detected: 1463.888000 != 1497.118000. CPU Frequency is not max.
  4096       5000             9795.50            3115.68        0.797614
 Conflicting CPU frequency values detected: 1509.224000 != 1473.189000. CPU Frequency is not max.
  8192       5000             16819.48            4812.72           0.616029
 Conflicting CPU frequency values detected: 1498.535000 != 3249.652000. CPU Frequency is not max.
  16384      5000             21640.00            7135.11           0.456647
 Conflicting CPU frequency values detected: 1499.500000 != 3249.622000. CPU Frequency is not max.
  32768      5000             22108.58            10842.53          0.346961
 Conflicting CPU frequency values detected: 1493.685000 != 3249.586000. CPU Frequency is not max.
  65536      5000             22127.62            15517.96          0.248287
 Conflicting CPU frequency values detected: 1510.028000 != 3249.621000. CPU Frequency is not max.
  131072     5000             21923.26            18044.34          0.144355
 Conflicting CPU frequency values detected: 1478.508000 != 1516.561000. CPU Frequency is not max.
  262144     5000             22115.73            20066.64          0.080267
 Conflicting CPU frequency values detected: 1490.561000 != 3249.643000. CPU Frequency is not max.
  524288     5000             22155.65            21157.79          0.042316
 Conflicting CPU frequency values detected: 1511.055000 != 3249.697000. CPU Frequency is not max.
  1048576    5000             22225.34            21669.41          0.021669
 Conflicting CPU frequency values detected: 1495.998000 != 3249.732000. CPU Frequency is not max.
  2097152    5000             22206.97            21976.05          0.010988
 Conflicting CPU frequency values detected: 1499.201000 != 1370.976000. CPU Frequency is not max.
  4194304    5000             22194.49            22164.23          0.005541
 Conflicting CPU frequency values detected: 1499.356000 != 1356.002000. CPU Frequency is not max.
  8388608    5000             22292.78            22292.76          0.002787
 ---------------------------------------------------------------------------------------
```

(4) 查看server端的结果
```bash
# numactl -N 7 ib_write_bw -a

************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF      Device         : mlx5_0
 Number of qps   : 1        Transport type : IB
 Connection type : RC       Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 CQ Moderation   : 100
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x1f7 QPN 0x5866 PSN 0xb96bc RKey 0x006251 VAddr 0x007fdda610c000
 remote address: LID 0x1f9 QPN 0x0182 PSN 0xb97c74 RKey 0x006dc7 VAddr 0x007f1c732df000
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 8388608    5000             22292.78            22292.76          0.002787
---------------------------------------------------------------------------------------
```

### 小结
（1）未设置 numa 时，server端的写速度为 **10787.86 MB/sec**，设置 numa 后 (将进程绑定在 CPU1 的核心上)，速度为 **22292.76 MB/sec**。
（2）未设置 numa 的时候，server端的`ib_write_bw` 进程在 49 核心上，49 核心在 0 号 CPU 上，但 IB 网卡属于 CPU1 的资源，所以 IB 网卡未能发挥最好的效果。
（3）设置 numa 后，server端的`ib_write_bw` 进程在 113 核心上，113 核心在 1 号 CPU 上，IB 网卡属于 CPU1 的资源，能够发挥最好的性能。 
（4）进行了多次测试，CPU0 的 numa 节点为 0、1、2、3，CPU1 的 numa节点为 4、5、6、7；
当进程设置在 0、1、2、3 四个 numa 节点时，测试速度均为 **10000 MB/sec** 左右；
当进程设置在 4、5、6、7 四个 numa 节点时，测试速度均为 **22000 MB/sec** 左右

# 参考
```bash
# RDMA测试工具perftest简介
https://mp.weixin.qq.com/s/-KZPhboJP9lAzQDs8GoKCw

# 记录一次 NUMA 调优
https://mp.weixin.qq.com/s/0_L4Lb6soq1zLFKN_v3A0Q
```