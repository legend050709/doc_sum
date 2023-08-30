# 定位
## DPDK定位
- DPDK 的定位是基础组件, 提供的是基于X86平台的高效包转发方案.
DPDK包含一些最重要的底层组件,PMD/内存管理/初始化/Lib库等等.这些组件大多是为了使能硬件,优化硬件的用例.

## VPP定位
- VPP 的定位是解决方案，更接近面向最终产品化的解决方案。
VPP 并不依赖特定的包转发框架, 当然在X86平台下, VPP集成DPDK的包转发组件是一个明智选择. VPP对于对各类协议的支持更为完善，调试框架,控制平面的接口都更加完备.
> VPP 有Honeycomb这样的控制平面接口, 支持netconf/yang. 这个对于产品化是非常重要的.DPDK 没有对应的组件.
# 区别
## 模式差异
DPDK支持 Run To Completation/Pipe Line 两种模型。
VPP 只支持 Run To Completation.
# 参考
```c
https://github.com/iqiyi/dpvs
https://zhuanlan.zhihu.com/p/533431428
https://www.bilibili.com/video/BV1ir4y1j7DE/?vd_source=e56378e669d648d0ad1a01aee17b88c4
```