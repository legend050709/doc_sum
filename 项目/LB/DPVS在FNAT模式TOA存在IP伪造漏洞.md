```table-of-contents
```
# 漏洞描述

四层负载均衡后业务需要使用用户原始IP，这是常见的功能需求。在大型IDC内部，一般会在RS节点上部署TOA（TCP Option Address）内核模块，用来获取TCP Option中的原始IP。

开源的L4LB DPVS 4层负载均衡产品，在FNAT模式下，这些产品技术实现上，未能很好的清除TCP Option中恶意构造的TOA信息，将恶意数据透传至RS服务器，导致业务服务器取到伪造IP。

# 漏洞影响

IP在很多Web防火墙、反爬虫系统、防刷系统（薅羊毛）是用于做策略控制的基础强依赖，IP的伪造将导致这些系统完全失效，造成极大的风险损失。

IP同样被用于后台系统的ACL的网络边界，此漏洞也依旧成为可以突破的入口。攻击者可伪造IDC公网IP、内网IP等，实现特定IP加白，甚至直接放行等等。

# 触发条件

1. L4LB产品使用FNAT模式传递客户端IP
    
2. 服务器主机启用TOA模块获取IP
# 修复方案
```c
Index: src/ipvs/ip_vs_proto_tcp.c  
IDEA additional info:  
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP  
<+>UTF-8  
===================================================================  
diff --git a/src/ipvs/ip_vs_proto_tcp.c b/src/ipvs/ip_vs_proto_tcp.c  
--- a/src/ipvs/ip_vs_proto_tcp.c (revision 30e558898060f33a2595c6545253eddd692ecd20)  
+++ b/src/ipvs/ip_vs_proto_tcp.c (revision 669c54d067cfd4dbceae3304c1d1aa5a6031a7f7)  
@@ -305,6 +305,48 @@  
     }  
 }  
   
+/* use NOP option to replace TCP_OLEN_IP4_ADDR and TCP_OLEN_IP6_ADDR opt */  
+static void tcp_in_remove_toa(struct tcphdr *tcph, int af)  
+{  
+    unsigned char *ptr;  
+    int len, i;  
+    uint32_t tcp_opt_len = af == AF_INET ? TCP_OLEN_IP4_ADDR : TCP_OLEN_IP6_ADDR;  
+  
+    ptr = (unsigned char *)(tcph + 1);  
+    len = (tcph->doff << 2) - sizeof(struct tcphdr);  
+  
+    while (len > 0) {  
+        int opcode = *ptr++;  
+        int opsize;  
+  
+        switch (opcode) {  
+        case TCP_OPT_EOL:  
+            return;  
+        case TCP_OPT_NOP:  
+            len--;  
+            continue;  
+        default:  
+            opsize = *ptr++;  
+            if (opsize < 2)    /* silly options */  
+                return;  
+            if (opsize > len)  
+                return;    /* partial options */  
+            if ((opcode == TCP_OPT_ADDR) && (opsize == tcp_opt_len)) {  
+                for (i = 0; i < tcp_opt_len; i++) {  
+                    *(ptr - 2 + i) = TCP_OPT_NOP;  
+                }  
+                /* DON'T RETURN  
+                 * keep search other TCP_OPT_ADDR ,and clear them.  
+                 * See https://github.com/iqiyi/dpvs/pull/925 for more detail. */  
+            }  
+  
+            ptr += opsize - 2;  
+            len -= opsize;  
+            break;  
+        }  
+    }  
+}  
+  
 static inline int tcp_in_add_toa(struct dp_vs_conn *conn, struct rte_mbuf *mbuf,  
                           struct tcphdr *tcph)  
 {  
@@ -719,14 +761,25 @@  
      */  
     if (th->syn && !th->ack) {  
         tcp_in_remove_ts(th);  
+  
         tcp_in_init_seq(conn, mbuf, th);  
-        tcp_in_add_toa(conn, mbuf, th);  
+  
+        /* Only clear when adding TOA fails to reduce invocation frequency and improve performance.  
+         * See https://github.com/iqiyi/dpvs/pull/925 for more detail. */  
+        if (unlikely(tcp_in_add_toa(conn, mbuf, th) != EDPVS_OK)) {  
+            tcp_in_remove_toa(th, af);  
+        }  
     }  
   
     /* add toa to first data packet */  
     if (ntohl(th->ack_seq) == conn->fnat_seq.fdata_seq  
-            && !th->syn && !th->rst /*&& !th->fin*/)  
-        tcp_in_add_toa(conn, mbuf, th);  
+            && !th->syn && !th->rst /*&& !th->fin*/) {  
+        /* Only clear when adding TOA fails to reduce invocation frequency and improve performance.  
+         * See https://github.com/iqiyi/dpvs/pull/925 for more detail. */  
+        if (unlikely(tcp_in_add_toa(conn, mbuf, th) != EDPVS_OK)) {  
+            tcp_in_remove_toa(th, af);  
+        }  
+    }  
   
     tcp_in_adjust_seq(conn, th);
```

> 注：DPVS目前不支持串联获取Client IP。即目前不支持： Clinet--->LB---->LB---->RS， 然后在RS获取原始client的ip。
> 因为在DPVS中toa option都是插入到TCP option的最前面，在TOA内核模块中，也是从开始的TCP option中查询toa信息，查找到直接返回了。

# 影响范围

## DPVS 产品

DPVS官方社区显示，国内小米、中国移动、网易、快手等公司在该软件，可能造成一定影响。笔者未一一验证。

## LVS产品

从开源社区的代码来看，LVS也存在这个漏洞，笔者暂时无时间验证，请参考修复。

## 危害扩展

如前面所述，形成此漏洞的条件有两个，若你的产品是4层FNAT，RS使用TOA获取IP，都受到影响。包括不限于各家自定义toa结构体长度，即`OPSIZE`，自定义`OPCODE` kind ID等。从网上公开的信息来看，包括阿里云、阿里巴巴、腾讯云、华为云、字节等都有类似方案的痕迹。

除了TOA，比如UDP的IP传递方式`UOA`，甚至基于IP Option的传递方式，也有类似的风险，请自检。
# 参考
```c
https://mp.weixin.qq.com/s/UCwTc_Znv55m-omyUdKatw
```