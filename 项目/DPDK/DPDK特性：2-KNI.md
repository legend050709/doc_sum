```table-of-contents
```
# DPDK 23.11移除KNI

目前来看，DPDK 22.11 还是存在 rte_kni lib 库的，到了 DPDK 23.11 不存在 rte_kni lib库。 

![](attachments/Pasted%20image%2020241206202304.png)

参考：[ kni: remove deprecated kernel network interface](https://inbox.dpdk.org/dev/20230801160500.67480-3-stephen@networkplumber.org/)

# 替代方案：

# dpdk-kmods
## 介绍
从主线DPDK 代码仓库中移出来之后，单独放入到一个代码仓库中`[dpdk-kmods](https://git.dpdk.org/dpdk-kmods/ "dpdk-kmods")`，`https://git.dpdk.org/dpdk-kmods/`。
比如：KNI 模块从 DPDK 22.11 中移出来之后，`igb_uio` 也从主线的 DPDK 代码中移出来了。

# 其他
## 查看当前的DPDK版本
```bash
pkg-config --modversion libdpdk
```
![](attachments/Pasted%20image%2020241206151112.png)


其他方法：
```bash
查找 libdpdk.pc 文件；感觉文件中的信息得到dpdk的版本
# find / -type f -name libdpdk.pc

各个pc文件，一般都是在 pkgconfig 目录下。
find / -type d -name pkgconfig
```
![](attachments/Pasted%20image%2020241206145734.png)

# 参考
```bash

```