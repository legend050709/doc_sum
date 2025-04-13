```table-of-contents
```

# DPDK支持的加密引擎
在 `supported Hardwares ---> Crypto Engines` 中找到如下所示：
![](attachments/Pasted%20image%2020241127114232.png)
![](attachments/Pasted%20image%2020241127114022.png)

# 硬件加解密
## intel qat 的加解密
参考：[# Intel(R) QuickAssist (QAT) Crypto Poll Mode Driver](https://doc.dpdk.org/guides/cryptodevs/qat.html)

[# 【QAT】英特尔QAT加速技术|Intel QuickAssist Technology](https://blog.csdn.net/bandaoyu/article/details/119671880)

QAT (Quick Assist Technology) 是 Intel 提供的一种硬件加速技术，用于加强加密和解密操作的性能。

### QAT的集成形式
QAT 的实现可以集成在不同的硬件组件中，可以集成到网卡中，也可以集成到CPU中。

#### 集成到 CPU 中

一些 Intel Xeon 处理器（例如，部分 Xeon Scalable 处理器`# Second Generation Intel® Xeon® Scalable Processors`）内置了 QAT 功能。
这种集成允许 CPU 在执行其他计算任务的同时，利用 QAT 加速器处理加密和压缩任务，从而提高整体性能。
这种集成方式使得 QAT 在数据中心和云计算环境中非常受欢迎，因为它可以减少对外部硬件的依赖。

#### 集成到 NIC 中

一些网络适配器（网卡）也集成了 QAT 功能。这种设计允许在网络数据传输时进行实时的加密和解密处理，从而减轻 CPU 的负担。

这种形式适合于网络密集型应用，能够在数据传输的同时提升安全性和性能。

#### 独立的Pcie加速卡

Intel 提供了专门的 QAT 加速卡（Pcie加速卡），例如 Intel QAT 设备（如 Intel QAT C62x 系列加速卡），这些卡可以插入到服务器的 PCIe 插槽中。

![](attachments/Pasted%20image%2020241127182358.png)

> 注：目前来看，主要还是独立的Pcie的QAT加速卡。

### QAT硬件的优势在于非对称加密

QAT硬件加速提供了对称与非对称两类密码算法的支持，主要包括：
1. 非对称加密算法：RSA, ECDSA, ECDHE
2. 对称加密算法：AES-GCM(128,192,256) 
    
注：QAT加速的优势主要体现在非对称加密上，从官方的整体性能数据看，非对称算法性能提升1.6~2倍，对称算法性能提升10%~15%

### QAT engine
#### QAT engine的两种加速方式
QAT engine支持两种加速：
**（1）硬件加速**：
使用QAT硬件加速卡，将密码算法的计算从 CPU 转移到硬件加速卡。

注：==QAT硬件加速主要是加速非对称加密，对于对称加密的提升有限==。
硬件加速核心是将TLS中的非对称加解密操作剥离出来，放到硬件加速卡里计算，即解放了CPU，同时专用的硬件加速卡也提供了更高的加解密性能，这是典型的硬件OffLoad技术方案。

**（2）软件加速**：
使用 Intel 的多缓冲 (multi-buffer) 技术，对密码算法进行并行处理优化。这种技术主要利用了 Intel 的 AVX-512 指令集等。

##### QAT engine的使用
QAT Engine 是作为一个中间层位于应用程序和硬件之间，负责在用户应用程序和硬件卡之间传递加密解密操作的输入输出数据。
QAT Engine 以 OpenSSL 插件的形式提供给用户，使得开发者可以利用标准的 OpenSSL API 来实现对 TLS（传输层安全协议）的加速，而不需要修改现有的代码。



## mellanox Cx5 crypto的加解密
参考：[# NVIDIA MLX5 Crypto Driver](https://doc.dpdk.org/guides/cryptodevs/mlx5.html)


# 软件cpu的加解密

软件加速，主要就是**用CPU的SIMD指令提高CPU处理时的并行能力，这个是intel的方案，称之为multi-buffer crypto**。  
对应着两个开源工程，可以把他们组合到qatengine里，从而完成软件加速。  
对称加密：[https://github.com/intel/intel-ipsec-mb](https://github.com/intel/intel-ipsec-mb)  
非对称加密：[https://github.com/intel/ipp-crypto/tree/develop/sources/ippcp/crypto_mb](https://github.com/intel/ipp-crypto/tree/develop/sources/ippcp/crypto_mb)

## aesni-gcm aead算法
参考：[# AES-NI GCM Crypto Poll Mode Driver](https://doc.dpdk.org/guides/cryptodevs/aesni_gcm.html)
[# AES-NI Multi Buffer Crypto Poll Mode Driver](https://doc.dpdk.org/guides/cryptodevs/aesni_mb.html)

### 介绍
在 AES-NI GCM 中，"NI" 指的是 "New Instructions"（新指令）。AES-NI 是一组由英特尔和AMD等处理器制造商引入的指令集扩展，旨在加速高级加密标准（AES）的加密和解密操作。
因此，AES-NI 指令集通过硬件加速 AES 算法，使得加密和解密过程更加高效，特别是在处理大数据量时。

## zuc aead算法
参考：[# ZUC Crypto Poll Mode Driver](https://doc.dpdk.org/guides/cryptodevs/zuc.html)

# intel 多缓存 ipsec 加密库
## intel CPU的 Crypto Acceleration
### 背景
Intel认为网络通信安全对于未来互联网业务具有十分重要的意义，其中设计的加减密操作，通常消耗大量cpu资源，如作为网络安全基石的TLS企业技术的实现，在大规模微服务场景下可能成为瓶颈。

intel在最新的第三代至强可扩展处理器引入了Crypto Acceleration技术和架构创新，大幅度提高了一些加解密操作的性能指标。这些技术包括：
Public-Key Cryptography 、Symmetric Encryption 、Hashing 、Function stitching 、Multi-Buffer技术。

![](attachments/Pasted%20image%2020241127122840.png)

##  Intel Multi-Buffer 技术

Intel Multi-buffer 基本原理就是使用CPU的SIMD机制，通过 AVX-512 指令集并行处理数据，来提升对称加密/非对称加密算法性能。

Multi-Buffer使用AVX-512指令同时处理多个独立的缓冲区，既可以在一个执行周期内同时执行多个加解密操作，加解密的执行效率便会得到成倍的提升。


## Intel IPsec MB介绍
为使用`Multi-Buffer`技术，intel提供了软件库。
`Intel IPsec MB`是一个专为加速包处理应用设计的**软件加密库**，其支持包括IPsec、TLS、无线通信（RAN）等多种场景。

这个库在GitHub上开源，并被集成到 DPDK、Intel(R) QAT Engine和FD.io等框架中，为用户提供**灵活的软件加密解决方案**。

### Intel IPsec MB软件库在DPDK中的集成
dpdk中的  `intel-ipsec-mb` 库 `ipsec_mb` ，`ipsec_mb` 下还有多个pmd，对应不同的算法。不同pmd类型相关的数据会保存到 ipsec_mb_pmds 变量中，公共函数根据当前的pmd类型获取对应pmd的数据和函数处理数据。实现不同pmd的分离。


# 参考
```bash
# 使用阿里云服务网格 ASM 和 Intel Multi-Buffer 技术实现更快的应用服务间加密通信
https://developer.aliyun.com/article/1090597
```