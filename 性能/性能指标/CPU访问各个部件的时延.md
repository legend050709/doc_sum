```table-of-contents
```

# CPU的cache访问的时延



## 时延数据

![](attachments/Pasted%20image%2020240701155622.png)

```
           0.5 ns - CPU L1 dCACHE reference
           1   ns - speed-of-light (a photon) travel a 1 ft (30.5cm) distance
           5   ns - CPU L1 iCACHE Branch mispredict
           7   ns - CPU L2  CACHE reference
          71   ns - CPU cross-QPI/NUMA best  case on XEON E5-46*
         100   ns - MUTEX lock/unlock
         100   ns - own DDR MEMORY reference
         135   ns - CPU cross-QPI/NUMA best  case on XEON E7-*
         202   ns - CPU cross-QPI/NUMA worst case on XEON E7-*
         325   ns - CPU cross-QPI/NUMA worst case on XEON E5-46*
      10,000   ns - Compress 1K bytes with Zippy PROCESS
      20,000   ns - Send 2K bytes over 1 Gbps NETWORK
     250,000   ns - Read 1 MB sequentially from MEMORY
     500,000   ns - Round trip within a same DataCenter
  10,000,000   ns - DISK seek
  10,000,000   ns - Read 1 MB sequentially from NETWORK
  30,000,000   ns - Read 1 MB sequentially from DISK
 150,000,000   ns - Send a NETWORK packet CA -> Netherlands
|   |   |   |
|   |   | ns|
|   | us|
| ms|
```

### 范例

intel I7 志强系统CPU的时延数据：

参考：[Performance Analysis Guide for Intel i7 processor](https://web.archive.org/web/20160315021718/https://software.intel.com/sites/products/collateral/hpc/vtune/performance_analysis_guide.pdf)


```
Core i7 Xeon 5500 Series Data Source Latency (approximate)               [Pg. 22]

local  L1 CACHE hit,                              ~4 cycles (   2.1 -  1.2 ns )
local  L2 CACHE hit,                             ~10 cycles (   5.3 -  3.0 ns )
local  L3 CACHE hit, line unshared               ~40 cycles (  21.4 - 12.0 ns )
local  L3 CACHE hit, shared line in another core ~65 cycles (  34.8 - 19.5 ns )
local  L3 CACHE hit, modified in another core    ~75 cycles (  40.2 - 22.5 ns )

remote L3 CACHE (Ref: Fig.1 [Pg. 5])        ~100-300 cycles ( 160.7 - 30.0 ns )

local  DRAM                                                   ~60 ns
remote DRAM                                                  ~100 ns
```

说明：像内存这样相对不算太慢的部件都基本稳定在100ns左右，其它外设就更不用说了。
根据CPU的主频不同，1个cycle代表的时间也不同，在1GHZ主频的CPU下，1个cycle也就是1ns。那么在至强5500的CPU上（按频率2GHZ多一点算，比如Xeon E5506 2.13GHz），L1 CACHE命中的情况下，4 cycles也就是约2ns。


## 应用

了解这些东西有什么用？
举个例子，如果要让一个主频为1GHZ的单核CPU做到万兆（10G）线速转发，我们可以算出每一个包只能占用：1GHZ/14.88Mpps=67.20ns，而访问一次内存就需要60ns（单核就没有本地内存、远程内存的概念了），所以要做到万兆(10G)线速就不能出现cache miss。

如果按上面的四核至强Xeon E5506 2.13GHz算，要做到万兆线速，则每个包可以有67*4*2.13=572.58ns，此时也只允许几次cache miss才能做到我们要求的万兆线速。


# 参考
```bash
# CPU访问各个部件的延时时长
https://blog.csdn.net/u014089131/article/details/59521367
```