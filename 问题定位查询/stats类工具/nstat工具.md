```table-of-contents
```

# nstat
## 介绍
nstat 是一个简单的监视内核的 SNMP 计数器和网络接口状态的实用工具 。

nstat 可以使用通配符指定一个或多个要过滤的内核的 SNMP（Simple Network Management Protocol） 计数器名称。
**其实查看的是 `/proc/net/netstat、/proc/net/snmp6、/proc/net/snmp` 文件中的内容**。主要是这几个文件，读取不方便。因此有了nstat 命令。

```bash
nstat and rtacct are simple tools to monitor kernel snmp counters and network interface statistics.
```

**差值的原理**
其实是在/tmp下创建了一个临时文件(比如：/tmp/.nstat.u0)，记录上一次的输出，再次执行该命令从 `/proc/net/netstat、/proc/net/snmp6、/proc/net/snmp`中读取结果后，和上一次结果进行diff，得到差值。

## 使用

### 显示计数器的绝对值
```
# nstat -az
#kernel
IpInReceives                    469406576          0.0
IpInHdrErrors                   0                  0.0
IpInAddrErrors                  0                  0.0
IpForwDatagrams                 0                  0.0
IpInUnknownProtos               0                  0.0
IpInDiscards                    0                  0.0
IpInDelivers                    469406573          0.0
IpOutRequests                   490427517          0.0
IpOutDiscards                   0                  0.0
IpOutNoRoutes                   22                 0.0
IpReasmTimeout                  0                  0.0
IpReasmReqds                    0                  0.0
IpReasmOKs                      0                  0.0
IpReasmFails                    0                  0.0
IpFragOKs                       0                  0.0
IpFragFails                     0                  0.0
IpFragCreates                   0                  0.0
IcmpInMsgs                      0                  0.0
IcmpInErrors                    649433             0.0
IcmpInCsumErrors                0                  0.0
IcmpInDestUnreachs              0                  0.0
IcmpInTimeExcds                 41161              0.0
IcmpInParmProbs                 0                  0.0
IcmpInSrcQuenchs                0                  0.0
IcmpInRedirects                 0                  0.0
IcmpInEchos                     0                  0.0
IcmpInEchoReps                  608272             0.0
IcmpInTimestamps                0                  0.0
IcmpInTimestampReps             0                  0.0
IcmpInAddrMasks                 0                  0.0
IcmpInAddrMaskReps              0                  0.0
IcmpOutMsgs                     0                  0.0
IcmpOutErrors                   685414             0.0
IcmpOutDestUnreachs             0                  0.0
IcmpOutTimeExcds                77142              0.0
IcmpOutParmProbs                0                  0.0
IcmpOutSrcQuenchs               0                  0.0
IcmpOutRedirects                0                  0.0
IcmpOutEchos                    0                  0.0
IcmpOutEchoReps                 0                  0.0
IcmpOutTimestamps               608272             0.0
IcmpOutTimestampReps            0                  0.0
IcmpOutAddrMasks                0                  0.0
IcmpOutAddrMaskReps             0                  0.0
IcmpMsgInType3                  41161              0.0
IcmpMsgInType8                  608272             0.0
IcmpMsgOutType0                 608272             0.0
IcmpMsgOutType3                 77142              0.0
TcpActiveOpens                  21727611           0.0
TcpPassiveOpens                 2348412            0.0
TcpAttemptFails                 17185953           0.0
TcpEstabResets                  1938348            0.0
TcpInSegs                       92831131           0.0
TcpOutSegs                      119724480          0.0
TcpRetransSegs                  67576              0.0
TcpInErrs                       0                  0.0
TcpOutRsts                      18846742           0.0
TcpInCsumErrors                 0                  0.0
UdpInDatagrams                  376741215          0.0
UdpNoPorts                      80457              0.0
UdpInErrors                     7347               0.0
UdpOutDatagrams                 389014861          0.0
UdpRcvbufErrors                 7347               0.0
UdpSndbufErrors                 0                  0.0
UdpInCsumErrors                 0                  0.0
UdpIgnoredMulti                 0                  0.0
UdpLiteInDatagrams              0                  0.0
UdpLiteNoPorts                  0                  0.0
UdpLiteInErrors                 0                  0.0
UdpLiteOutDatagrams             0                  0.0
UdpLiteRcvbufErrors             0                  0.0
UdpLiteSndbufErrors             0                  0.0
UdpLiteInCsumErrors             0                  0.0
UdpLiteIgnoredMulti             0                  0.0
Ip6InReceives                   913929             0.0
Ip6InHdrErrors                  0                  0.0
Ip6InTooBigErrors               0                  0.0
Ip6InNoRoutes                   0                  0.0
Ip6InAddrErrors                 0                  0.0
Ip6InUnknownProtos              0                  0.0
Ip6InTruncatedPkts              0                  0.0
Ip6InDiscards                   0                  0.0
Ip6InDelivers                   913929             0.0
Ip6OutForwDatagrams             0                  0.0
Ip6OutRequests                  917648             0.0
Ip6OutDiscards                  0                  0.0
Ip6OutNoRoutes                  0                  0.0
Ip6ReasmTimeout                 0                  0.0
Ip6ReasmReqds                   0                  0.0
Ip6ReasmOKs                     0                  0.0
Ip6ReasmFails                   0                  0.0
Ip6FragOKs                      0                  0.0
Ip6FragFails                    0                  0.0
Ip6FragCreates                  0                  0.0
Ip6InMcastPkts                  0                  0.0
Ip6OutMcastPkts                 3719               0.0
Ip6InOctets                     60364825           0.0
Ip6OutOctets                    60543145           0.0
Ip6InMcastOctets                0                  0.0
Ip6OutMcastOctets               208672             0.0
Ip6InBcastOctets                0                  0.0
Ip6OutBcastOctets               0                  0.0
Ip6InNoECTPkts                  913929             0.0
Ip6InECT1Pkts                   0                  0.0
Ip6InECT0Pkts                   0                  0.0
Ip6InCEPkts                     0                  0.0
Icmp6InMsgs                     7588               0.0
Icmp6InErrors                   0                  0.0
Icmp6OutMsgs                    11307              0.0
Icmp6OutErrors                  0                  0.0
Icmp6InCsumErrors               0                  0.0
Icmp6InDestUnreachs             0                  0.0
Icmp6InPktTooBigs               0                  0.0
Icmp6InTimeExcds                0                  0.0
Icmp6InParmProblems             0                  0.0
Icmp6InEchos                    0                  0.0
Icmp6InEchoReplies              0                  0.0
Icmp6InGroupMembQueries         0                  0.0
Icmp6InGroupMembResponses       0                  0.0
Icmp6InGroupMembReductions      0                  0.0
Icmp6InRouterSolicits           0                  0.0
Icmp6InRouterAdvertisements     0                  0.0
Icmp6InNeighborSolicits         3794               0.0
Icmp6InNeighborAdvertisements   3794               0.0
Icmp6InRedirects                0                  0.0
Icmp6InMLDv2Reports             0                  0.0
Icmp6OutDestUnreachs            0                  0.0
Icmp6OutPktTooBigs              0                  0.0
Icmp6OutTimeExcds               0                  0.0
Icmp6OutParmProblems            0                  0.0
Icmp6OutEchos                   0                  0.0
Icmp6OutEchoReplies             0                  0.0
Icmp6OutGroupMembQueries        0                  0.0
Icmp6OutGroupMembResponses      0                  0.0
Icmp6OutGroupMembReductions     0                  0.0
Icmp6OutRouterSolicits          3698               0.0
Icmp6OutRouterAdvertisements    0                  0.0
Icmp6OutNeighborSolicits        3797               0.0
Icmp6OutNeighborAdvertisements  3794               0.0
Icmp6OutRedirects               0                  0.0
Icmp6OutMLDv2Reports            18                 0.0
Icmp6InType135                  3794               0.0
Icmp6InType136                  3794               0.0
Icmp6OutType133                 3698               0.0
Icmp6OutType135                 3797               0.0
Icmp6OutType136                 3794               0.0
Icmp6OutType143                 18                 0.0
Udp6InDatagrams                 1                  0.0
Udp6NoPorts                     0                  0.0
Udp6InErrors                    0                  0.0
Udp6OutDatagrams                1                  0.0
Udp6RcvbufErrors                0                  0.0
Udp6SndbufErrors                0                  0.0
Udp6InCsumErrors                0                  0.0
Udp6IgnoredMulti                0                  0.0
UdpLite6InDatagrams             0                  0.0
UdpLite6NoPorts                 0                  0.0
UdpLite6InErrors                0                  0.0
UdpLite6OutDatagrams            0                  0.0
UdpLite6RcvbufErrors            0                  0.0
UdpLite6SndbufErrors            0                  0.0
UdpLite6InCsumErrors            0                  0.0
TcpExtSyncookiesSent            0                  0.0
TcpExtSyncookiesRecv            0                  0.0
TcpExtSyncookiesFailed          0                  0.0
TcpExtEmbryonicRsts             0                  0.0
TcpExtPruneCalled               0                  0.0
TcpExtRcvPruned                 0                  0.0
TcpExtOfoPruned                 0                  0.0
TcpExtOutOfWindowIcmps          0                  0.0
TcpExtLockDroppedIcmps          0                  0.0
TcpExtArpFilter                 0                  0.0
TcpExtTW                        2537710            0.0
TcpExtTWRecycled                0                  0.0
TcpExtTWKilled                  0                  0.0
TcpExtPAWSActive                0                  0.0
TcpExtPAWSEstab                 0                  0.0
TcpExtDelayedACKs               1964927            0.0
TcpExtDelayedACKLocked          9                  0.0
TcpExtDelayedACKLost            390                0.0
TcpExtListenOverflows           0                  0.0
TcpExtListenDrops               0                  0.0
TcpExtTCPHPHits                 11365261           0.0
TcpExtTCPPureAcks               11259717           0.0
TcpExtTCPHPAcks                 11706535           0.0
TcpExtTCPRenoRecovery           0                  0.0
TcpExtTCPSackRecovery           56                 0.0
TcpExtTCPSACKReneging           0                  0.0
TcpExtTCPSACKReorder            67                 0.0
TcpExtTCPRenoReorder            0                  0.0
TcpExtTCPTSReorder              3                  0.0
TcpExtTCPFullUndo               1                  0.0
TcpExtTCPPartialUndo            3                  0.0
TcpExtTCPDSACKUndo              6                  0.0
TcpExtTCPLossUndo               66                 0.0
TcpExtTCPLostRetransmit         40465              0.0
TcpExtTCPRenoFailures           0                  0.0
TcpExtTCPSackFailures           1                  0.0
TcpExtTCPLossFailures           0                  0.0
TcpExtTCPFastRetrans            796                0.0
TcpExtTCPSlowStartRetrans       1                  0.0
TcpExtTCPTimeouts               5947               0.0
TcpExtTCPLossProbes             22438              0.0
TcpExtTCPLossProbeRecovery      18                 0.0
TcpExtTCPRenoRecoveryFail       0                  0.0
TcpExtTCPSackRecoveryFail       0                  0.0
TcpExtTCPRcvCollapsed           0                  0.0
TcpExtTCPDSACKOldSent           395                0.0
TcpExtTCPDSACKOfoSent           0                  0.0
TcpExtTCPDSACKRecv              14636              0.0
TcpExtTCPDSACKOfoRecv           0                  0.0
TcpExtTCPAbortOnData            1655204            0.0
TcpExtTCPAbortOnClose           0                  0.0
TcpExtTCPAbortOnMemory          0                  0.0
TcpExtTCPAbortOnTimeout         5720               0.0
TcpExtTCPAbortOnLinger          0                  0.0
TcpExtTCPAbortFailed            0                  0.0
TcpExtTCPMemoryPressures        0                  0.0
TcpExtTCPMemoryPressuresChrono  0                  0.0
TcpExtTCPSACKDiscard            0                  0.0
TcpExtTCPDSACKIgnoredOld        19                 0.0
TcpExtTCPDSACKIgnoredNoUndo     5511               0.0
TcpExtTCPSpuriousRTOs           0                  0.0
TcpExtTCPMD5NotFound            0                  0.0
TcpExtTCPMD5Unexpected          0                  0.0
TcpExtTCPMD5Failure             0                  0.0
TcpExtTCPSackShifted            80                 0.0
TcpExtTCPSackMerged             59                 0.0
TcpExtTCPSackShiftFallback      194                0.0
TcpExtTCPBacklogDrop            0                  0.0
TcpExtPFMemallocDrop            0                  0.0
TcpExtTCPMinTTLDrop             0                  0.0
TcpExtTCPDeferAcceptDrop        0                  0.0
TcpExtIPReversePathFilter       0                  0.0
TcpExtTCPTimeWaitOverflow       0                  0.0
TcpExtTCPReqQFullDoCookies      0                  0.0
TcpExtTCPReqQFullDrop           0                  0.0
TcpExtTCPRetransFail            0                  0.0
TcpExtTCPRcvCoalesce            1399098            0.0
TcpExtTCPOFOQueue               531                0.0
TcpExtTCPOFODrop                0                  0.0
TcpExtTCPOFOMerge               0                  0.0
TcpExtTCPChallengeACK           0                  0.0
TcpExtTCPSYNChallenge           0                  0.0
TcpExtTCPFastOpenActive         0                  0.0
TcpExtTCPFastOpenActiveFail     0                  0.0
TcpExtTCPFastOpenPassive        0                  0.0
TcpExtTCPFastOpenPassiveFail    0                  0.0
TcpExtTCPFastOpenListenOverflow 0                  0.0
TcpExtTCPFastOpenCookieReqd     0                  0.0
TcpExtTCPFastOpenBlackhole      0                  0.0
TcpExtTCPSpuriousRtxHostQueues  107                0.0
TcpExtBusyPollRxPackets         0                  0.0
TcpExtTCPAutoCorking            1678488            0.0
TcpExtTCPFromZeroWindowAdv      2                  0.0
TcpExtTCPToZeroWindowAdv        2                  0.0
TcpExtTCPWantZeroWindowAdv      11                 0.0
TcpExtTCPSynRetrans             647                0.0
TcpExtTCPOrigDataSent           50369635           0.0
TcpExtTCPHystartTrainDetect     116                0.0
TcpExtTCPHystartTrainCwnd       5494               0.0
TcpExtTCPHystartDelayDetect     0                  0.0
TcpExtTCPHystartDelayCwnd       0                  0.0
TcpExtTCPACKSkippedSynRecv      0                  0.0
TcpExtTCPACKSkippedPAWS         0                  0.0
TcpExtTCPACKSkippedSeq          6                  0.0
TcpExtTCPACKSkippedFinWait2     0                  0.0
TcpExtTCPACKSkippedTimeWait     0                  0.0
TcpExtTCPACKSkippedChallenge    0                  0.0
TcpExtTCPWinProbe               0                  0.0
TcpExtTCPKeepAlive              899311             0.0
TcpExtTCPMTUPFail               0                  0.0
TcpExtTCPMTUPSuccess            0                  0.0
TcpExtTCPDelivered              54919689           0.0
TcpExtTCPDeliveredCE            0                  0.0
TcpExtTCPAckCompressed          88                 0.0
TcpExtTCPWqueueTooBig           0                  0.0
IpExtInNoRoutes                 3                  0.0
IpExtInTruncatedPkts            0                  0.0
IpExtInMcastPkts                0                  0.0
IpExtOutMcastPkts               0                  0.0
IpExtInBcastPkts                0                  0.0
IpExtOutBcastPkts               0                  0.0
IpExtInOctets                   79681676414        0.0
IpExtOutOctets                  110216190575       0.0
IpExtInMcastOctets              0                  0.0
IpExtOutMcastOctets             0                  0.0
IpExtInBcastOctets              0                  0.0
IpExtOutBcastOctets             0                  0.0
IpExtInCsumErrors               0                  0.0
IpExtInNoECTPkts                470036308          0.0
IpExtInECT1Pkts                 0                  0.0
IpExtInECT0Pkts                 0                  0.0
IpExtInCEPkts                   0                  0.0
IpExtReasmOverlaps              0                  0.0
```

### json格式打印结果
```
nstat -j -az
nstat -j

如果不支持json格式，可能是 iproute包的版本过低，
yum -y update iproute
```

```bash
# nstat -a | grep -i tcp
TcpActiveOpens                  21729638           0.0
TcpPassiveOpens                 2348627            0.0
TcpAttemptFails                 17187544           0.0
TcpEstabResets                  1938525            0.0
TcpInSegs                       92841388           0.0
TcpOutSegs                      119737414          0.0
TcpRetransSegs                  67577              0.0
TcpOutRsts                      18848489           0.0
TcpExtTW                        2537944            0.0
TcpExtDelayedACKs               1965115            0.0
TcpExtDelayedACKLocked          9                  0.0
TcpExtDelayedACKLost            390                0.0
TcpExtTCPHPHits                 11366527           0.0
TcpExtTCPPureAcks               11261055           0.0
TcpExtTCPHPAcks                 11708620           0.0
TcpExtTCPSackRecovery           56                 0.0
TcpExtTCPSACKReorder            67                 0.0
TcpExtTCPTSReorder              3                  0.0
TcpExtTCPFullUndo               1                  0.0
TcpExtTCPPartialUndo            3                  0.0
TcpExtTCPDSACKUndo              6                  0.0
TcpExtTCPLossUndo               66                 0.0
TcpExtTCPLostRetransmit         40465              0.0
TcpExtTCPSackFailures           1                  0.0
TcpExtTCPFastRetrans            796                0.0
TcpExtTCPSlowStartRetrans       1                  0.0
TcpExtTCPTimeouts               5947               0.0
TcpExtTCPLossProbes             22440              0.0
TcpExtTCPLossProbeRecovery      18                 0.0
TcpExtTCPDSACKOldSent           395                0.0
TcpExtTCPDSACKRecv              14637              0.0
TcpExtTCPAbortOnData            1655358            0.0
TcpExtTCPAbortOnTimeout         5720               0.0
TcpExtTCPDSACKIgnoredOld        19                 0.0
TcpExtTCPDSACKIgnoredNoUndo     5511               0.0
TcpExtTCPSackShifted            80                 0.0
TcpExtTCPSackMerged             59                 0.0
TcpExtTCPSackShiftFallback      194                0.0
TcpExtTCPRcvCoalesce            1399221            0.0
TcpExtTCPOFOQueue               531                0.0
TcpExtTCPSpuriousRtxHostQueues  107                0.0
TcpExtTCPAutoCorking            1679122            0.0
TcpExtTCPFromZeroWindowAdv      2                  0.0
TcpExtTCPToZeroWindowAdv        2                  0.0
TcpExtTCPWantZeroWindowAdv      11                 0.0
TcpExtTCPSynRetrans             647                0.0
TcpExtTCPOrigDataSent           50376067           0.0
TcpExtTCPHystartTrainDetect     117                0.0
TcpExtTCPHystartTrainCwnd       5512               0.0
TcpExtTCPACKSkippedSeq          6                  0.0
TcpExtTCPKeepAlive              899398             0.0
TcpExtTCPDelivered              54926558           0.0
TcpExtTCPAckCompressed          88                 0.0
```


### 指定计数器名称查询
```bash
nstat IpInReceives
nstat IpInReceives -a
nstat IpInReceives -a -j

范例如下：
# nstat -az UdpSndbufErrors UdpRcvbufErrors
#kernel
UdpRcvbufErrors                 0                  0.0
UdpSndbufErrors                 0                  0.0
```
## 使用场景

## nstat 和 netstat 对比
nstat 也是读取 `/proc/net/snmp` 、`/proc/net/snmp6` 、`/proc/net/netstat` 的文件。
但是`nstat`的输出可以更方便的进行格式化。
# rtacct

# 参考
```c
# Nstat Linux 命令
https://0xzx.com/2023020800033150675.html

## Linux 命令（219）—— nstat 命令
https://cloud.tencent.com/developer/article/2195776
```