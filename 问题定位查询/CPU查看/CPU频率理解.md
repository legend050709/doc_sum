```table-of-contents
```
# CPU介绍
## 硬件组成
![](attachments/Pasted%20image%2020240131171017.png)

从上图可以看出 `CPU` 是安装在主板(mainboard)上的，主板上安装 `CPU` 的插槽就称之为 `socket` 。一个主板一般会有一个或多个 `socket` ，每个 `socket` 只可以插入一个 `CPU` 。

其次，每个物理 `CPU` 内部又会有一个或多少物理核心，这个核心切切实实存在的硬件芯片。
![](attachments/Pasted%20image%2020240131171044.png)

在实际开发中，我们接触的都是 ****逻辑核**** 的概念，简单的地说，如果 `CPU` 支持 `Hyper-Threading` 那么启用 `Hyper-Threading` 后，每个物理核都会划分成 `n` 个逻辑核， `n` 是 `Hyper-Threading` 的数量。

- `socket` ：主板上的 `CPU` 插槽
- `CPU（Central Processing Unit）` ：完整的 `CPU` 硬件，可以插入主板的硬件。
- `物理核（Physical core/processor）` ： `CPU` 内部存在的硬件物理核心，可以看的到的，有独立的电路元件以及L1,L2缓存，可以独立地执行指令。
- `逻辑核（Logical core/processor）` ： 在同一个物理核内，逻辑层面的核。
- `超线程（Hyper-Threading）` ： 超线程技术就是让一个核模拟出两个核的技术。

最后整体看一下 `socket` 、 `CPU` 、 `physical core` 、 `logical core` 之间的关系：
![](attachments/Pasted%20image%2020240131171132.png)

# CPU频率定义
## CPU 频率
单位时间1s内所产生的脉冲个数称之为频率，频率的最基本计量单位就是赫兹Hz。
**时钟频率就是每秒钟产生的脉冲信号数量，是评定CPU性能的重要指标。**

## 时钟周期
**时钟频率（f）与周期（T）两者互为倒数：f=1/T**

以Intel Core i3-8350k为例，它的默频是4GHz，意味着它内部时钟频率为4GHz，一秒钟可以产生40亿个脉冲信号，换句话说每一个脉冲信号仅仅用时0.25ns（时钟周期）。

时钟周期作为CPU操作的最小时间单位，内部的所有操作都是以这个时钟周期作为基准。一般来说CPU都是以时钟脉冲的上升沿作为执行指令的基准，频率越高，CPU执行的指令数越多，工作速度越快。

## CPU主频
CPU的主频，即CPU内核工作的主时钟频率，表示在CPU内数字脉冲信号震荡的速度。比如我们平时看到的4.0GHZ、3.0GHZ等指的就是CPU主频，即每秒可以产生40亿、30亿个脉冲信号。

## 外频
外频是CPU与外设（例如主板）之间同步运行的速度，所以也可以说**CPU的外频决定着整块主板的运行速度**。目前的CPU设计的外频都相当低，只有100MHz。

自从CPU诞生后，为了追求更高的性能，更快的速度，各大巨头就开始频率大战，虽然主频提升了，但外部的主板芯片组、内存、外部接口（PCIe、Sata）可还是处于旧有标准，而且这些设备的运行频率早就固定下来了，并且远低于CPU工作频率。就好比一个王者带一群青铜，实在是带不起来呀。

### 前端总线频率和外频
总是有很多网友将前端总线频率和外频混为一谈，其实他们不太一样。

在以前有北桥的时代，前端总线是CPU总线接口单元和北桥芯片之间的数据交换通道，曾经在AMD雷鸟系列、Intel奔腾 4处理器以前，前端总线与外频是一致的，但后来有了四倍数据传输率技术或者是八倍数据传输率技术，前端总线频率就极大地提高了。

举个例子，如果一个处理器的频率是2GHz，外频为100MHz，使用四倍数据传输率技术时，前端总线频率就变成400MHz；如果是八倍，那么就是800MHz。前端总线频率越大, 代表着CPU与北桥芯片之间的数据传输能力越大, 更能充分发挥出CPU的功能。

## 倍频
CPU和外设的频率不一致，这样一来CPU就无法很好与外设交流，Intel就机智地提出了倍频的概念。

**倍频系数是指CPU的核心工作频率与外频之间存在着一个比值关系**，这个比值就是倍频系数，简称倍频。**CPU主频=外频×倍频系数**

以前没有倍频概念，外频（系统总线频率）就相当于CPU的主频，两者速度一样。随着CPU的速度越来越快，倍频技术也就随之出现。即使外频（系统总线频率）工作速度相对较低，CPU速度仍然可以通过倍频来提升。

但由于CPU与系统之间数据传输速度有限，如果一味追求高倍频CPU会出现明显的瓶颈效应，即CPU从系统中得到数据的极限速度不能够满足CPU运算的速度。所以一般Intel的CPU都锁了倍频（工程样板可能除外），AMD之前也因为不锁倍频而很受电脑发烧友的追捧。

利用倍频技术, 较为完美地解决了CPU和内存等数据中转站的异步运行问题。为CPU后来向更高频率方向发展打下了扎实的基础。 

倍频系数是指CPU主频与外频之间的相对比例关系。在相同的外频下，倍频越高CPU的频率也越高。但实际上，在相同外频的前提下，高倍频的CPU本身意义并不大。这是因为CPU与系统之间数据传输速度是有限的，一味追求高主频而得到高倍频的CPU就会出现明显的“瓶颈”效应－CPU从系统中得到数据的极限速度不能够满足CPU运算的速度。

# CPU频率分类
## 默频
默频 即默认基础频率，是 CPU 标出的主频。
默频 是CPU制造商为其设计的正常运行频率，也称为基本频率或标称频率。例如，一块CPU可能被设计为在2.5 GHz的主频下运行。

## Intel 睿频Trubo Boost
睿频 是采用 Intel 睿频加速技术可达到的更高频率，可以理解为自动超频。

睿频是一种超频技术，使CPU可以在超过其主频的频率下运行。通过提高电压和时钟速度，睿频可以提高CPU的性能。
因此，睿频是一种性能提升技术，但也会增加CPU的功耗和温度。通常情况下，超频会导致CPU更快地磨损或损坏，所以需要谨慎使用。

如下面第九代 Intel core 处理器的介绍，已经明确标注默频、睿频能达到的频率，以及是未锁频，支持超频的也写在介绍中。
![](attachments/Pasted%20image%2020240131160038.png)

### 为什么要开启睿频？

以目前主流民用CPU i7 10700k为例，基础频率为3.8Ghz，全核睿频为4.7Ghz，同一CPU的性能跟频率高低成正比。开启睿频前，8个物理核心总频率为30.4Ghz，开启后为37.6Ghz，开启前后差距23.7%，如果不开启睿频，则意味着有接近23.7%的性能浪费。

### 开启和关闭
一般禁用睿频加速将限制您的 CPU 速度，以防止运行超过基本频率。 它对于节省电量并防止计算机过热很有用。 作为选择，用户还可以设置恒定的 CPU 速度，以获得最低功耗或最高性能。如今大多数机器在 BIOS 页面中都有选项来启用/禁用甚至更改 CPU 频率，但它并不灵活。

对于 Linux，内核有一个名为 cpupower 的工具可以从命令行完成这项工作，用户可以通过 sysfs 打开/关闭 Turbo boost。

#### windows系统设置

在bois彻底关闭睿频加速方法（需要关机重启）
不同型号电脑关闭的方式不一样，这里我只说一下我这个型号的关闭方式，此方法适合想彻底关闭睿频加速，让电脑保持冷静的朋友。  
**来自戴尔客服的方案：**

![](attachments/Pasted%20image%2020240807142409.png)

#### linux 下 设置
```bash
### intel 关闭睿频
echo 1 >  /sys/devices/system/cpu/intel_pstate/no_turbo

如果上面的第一个命令输出“No such file or directory”，则说明正在使用其他缩放驱动程序。在这种情况下，运行命令来检查 sysfs：

cat /sys/devices/system/cpu/cpufreq/boost
```

```bash
ls /sys/devices/system/cpu/intel_pstate
hwp_dynamic_boost  max_perf_pct  min_perf_pct  no_turbo  num_pstates  status  turbo_pct

说明：
max_perf_pct： 最大频率的百分比，例如4GHZ 4GHZ*90% = 3.6GHZ,可利用此控制最大频率
min_perf_pct： 最小比列，p-state能够调到最小频率的比例，这个比例最节能
no_turbo： 是否开启睿频， 0：开启 1： 关闭
num_pstates : 处理器支持的p状态数量。这个值在0-255之前。包含了睿频和非睿频p-state。此属性是只读的, 这是所支持的p状态与cpu电压和频率的一张表

```

**查看**
```bash
# cpupower frequency-info
```

![](attachments/Pasted%20image%2020240807152816.png)


### 范例

![](attachments/Pasted%20image%2020240807143737.png)

![](attachments/Pasted%20image%2020240807143902.png)

如上所示，开启了Turbo ， 单核的CPU的频率会更高。标频为2.7GHz，开启了Turbo之后，睿频为3.3GHz。
目前主流的民用CPU INTEL的 10700k，基础频率3.8Ghz, 睿频单核最高可以达到5.1Ghz。

## 超频
超频 是为了实现超过额定频率性能，人为调整各种指标（如电压、散热、外频、电源、BIOS等），属于手动超频。

但由于强行超频对系统和硬件会产生负面影响，所以大厂们在CPU出厂时将其倍频锁定在一个固定的数值，使其倍频系数不能再任何变动，即锁频。

### 睿频跟超频的区别

睿频：CPU根据实际运行程序的需求，动态的增加处理器的运行频率来提高处理器的性能，同时保持处理器继续稳定运行在规定的功耗、电流、电压和温度范围内，如果CPU出现故障，是享受质保的。**睿频是CPU自动实现的，无需人工设置，并且CPU运行稳定**。处理器商可以保证cpu达到睿频最大频率，并对cpu的寿命无影响。

超频：**用户强制**CPU所有内核运行比额定频率高的频率上，功耗、电流、电压和温度都可能超出规定范围，如果出现损坏，那么有可能无法享受质保。超频需要调整各种指标，比如电压、散热、外频、电源、BIOS等等，并且容易出现系统不稳定的情况。处理器商不保证CPU可以达到的最大频率，并且会对cpu的寿命进行影响。


## 频率之间的关系
睿频和超频很像，都可以提升频率，提高性能，但两者还是有本质区别。

**超频是人为提升频率**，调整各项指标，一般会超过处理器的规划规格，导致功耗的大幅度增加，虽然超频可以带来明显的性能提升，但是准入门槛高，风险很大，对CPU和系统都可能造成严重损坏。
而且，服务器的CPU是不允许超频的，如果服务器CPU超频，改变了外频，会产生异步运行，造成整个服务器系统的不稳定。

**睿频是采用睿频加速技术**（Intel的睿频技术为Turbo Boost Technology；AMD的睿频技术为TurboCore），**依靠处理器的智能自主处理，使 CPU 主频可以在某一范围内根据处理数据需要自动调整主频。**
通过分析当前CPU的负载情况，智能地完全关闭一些用不上的核心，把能源留给正在使用的核心，并使它们更高频率运行，进一步提升性能；
比如我们需要运行一个复杂程序，处理器会在原来的运行速度上自动提升 10%~20% 以保证程序流畅运行。
相反，需要多个核心时，动态开启相应的核心，智能调整频率。这样，在不影响CPU的TDP情况下，能把核心工作频率调得更高。


## CPU支持的最大频率和最小频率
# 工具
```bash
yum install -y util-linux kernel-tools
```

![](attachments/Pasted%20image%2020240807143119.png)


## lscpu命令
```bash
  [root@xeon ~]# lscpu
  Architecture:          x86_64
  CPU op-mode(s):        32-bit, 64-bit
  Byte Order:            Little Endian
  CPU(s):                40
  On-line CPU(s) list:   0-39
  Thread(s) per core:    2
  Core(s) per socket:    10
  Socket(s):             2
  NUMA node(s):          2
  Vendor ID:             GenuineIntel
  CPU family:            6
  Model:                 62
  Model name:            Intel(R) Xeon(R) CPU E5-2690 v2 @ 3.00GHz
  Stepping:              4
  CPU MHz:               1202.819
  BogoMIPS:              6017.30
  Virtualization:        VT-x
  L1d cache:             32K
  L1i cache:             32K
  L2 cache:              256K
  L3 cache:              25600K
  NUMA node0 CPU(s):     0-9,20-29
  NUMA node1 CPU(s):     10-19,30-39
```

- Architecture: CPU的架构信息，上面信息表示这是一个x86架构的64位CPU。
- CPU op-mode(s): CPU支持的模式 32位、64位。
- Byte Order：CPU字节序，上面信息表示这是个小端机器。
- CPU(s): CPU ****逻辑核**** 数量，上面信息表示这台机器有40个逻辑核。
- On-line CPU(s) list: 在线CPU列表。
- Thread(s) per core：每个 ****物理核**** 开启的超线程数量，如果不支持或没有开启超线程，该值为1。上面信息表示每个物理核开启2个超线程。
- Core(s) per socket: 每个CPU插槽有多少个 ****物理核**** ，上面信息表示这台机器每个 `socket` 有10个物理核。
- Socket(s): 主板上有多少个CPU插槽。上面信息表示这台机器有2个CPU插槽。
- NUMA node(s): 有多少个NUMA节点。上面信息表示这台机器有2个NUMA节点。
- Vendor ID: CPU制造商标识。
- CPU family: CPU产品系列标识
- Model: 第几代CPU标识
- Model name: CPU型号名称。
- Stepping: CPU版本号。
- CPU MHz: 当前运行频率。
- CPU max MHz: CPU最大运行频率。
- CPU min MHz: CPU最低运行频率。
- BogoMIPS: CPU 每秒钟什么都不能做的百万指令次数。上面信息表示这台机器CPU每秒可以执行60亿零1千7百三十万次指令。
- Virtualization: 支持虚拟化。
- L1d cache: CPU数据一级缓存大小。
- L1i cache: CPU指令一级缓存大小。
- L2 cache：CPU二级缓存大小。
- L3 cache: CPU三级缓存大小。
- NUMA node0 CPU(s): 位于NUMA node0的CPU ****逻辑核**** 编号。
- NUMA node1 CPU(s): 位于NUMA node1的CPU ****逻辑核**** 编号。


## /proc/cpuinfo信息
```bash
[root@xeon ~]# cat /proc/cpuinfo | head -27
  processor       : 0
  vendor_id       : GenuineIntel
  cpu family      : 6
  model           : 62
  model name      : Intel(R) Xeon(R) CPU E5-2690 v2 @ 3.00GHz
  stepping        : 4
  microcode       : 0x416
  cpu MHz         : 1199.890
  cache size      : 25600 KB
  physical id     : 0
  siblings        : 20
  core id         : 0
  cpu cores       : 10
  apicid          : 0
  initial apicid  : 0
  fpu             : yes
  fpu_exception   : yes
  cpuid level     : 13
  wp              : yes
  flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm epb tpr_shadow vnmi flexpriority ept vpid fsgsbase smep erms xsaveopt dtherm ida arat pln pts
  bugs            :
  bogomips        : 5999.64
  clflush size    : 64
  cache_alignment : 64
  address sizes   : 46 bits physical, 48 bits virtual
  power management:
```

`/proc/cpuinfo` 这个文件里记录的是每个 ****逻辑核**** 的信息。

- processor: 当前 ****逻辑核**** 编号
- physical id: ****物理CPU**** 编号（注意不是物理核，是完整的CPU），该值表明当前逻辑核位于哪颗 `CPU` 上。不同的 `physical id` 值，表示不同的CPU。这个编号不一定连续。
- siblings: 表示与当前逻辑核所在的 `CPU` 中有一共有少个逻辑核。
- core id: 当前逻辑核所在的 ****物理核**** 编号，不同的 `CPU` 可以有相同的 ****物理核**** 编号，即该编号只在同一个CPU内唯一，这个编号不一定连续。
- cpu cores: 当前 ****逻辑核**** 所在的 `CPU` 有多个个 ****物理核**** 。如果当前逻辑核的 `siblings` 的值和 `cpu cores` 值不相等，说明当前机器开启了超线程（ `Hyper-Threading` ）。



对于逻辑核比较多情况，直接查看 `/proc/cpuinfo` 数据量太多， 可以根据需要通过不同的命令来筛选：

1. 统计当前机器有多少个 ****逻辑核**** ： `grep processor /proc/cpuinfo | wc -l`
2. 统计当前机器有多少个 ****CPU**** ： `grep "physical id" /proc/cpuinfo | sort -u | wc -l`


### 范例
![](attachments/Pasted%20image%2020240131171612.png)

我把 `/proc/cpuinfo` 内容中部分字段整理成表格形式，这样我们就可以直观的解读每个 ****逻辑核**** 信息，以 `27` 号 ****逻辑核**** 为例：

- 该 ****逻辑核**** 位于 `0` 号 `CPU` 中 （ `physical id == 0` ）
- 该 ****逻辑核**** 所在 `CPU` 共有 20 个 ****逻辑核**** （ `siblings == 20` ）
- 该 ****逻辑核**** 所在 ****物理核**** 编号是 `10` ( `core id == 10` )
- 该 ****逻辑核**** 所有 `CPU` 共有 10 个 ****物理核**** ( `cpu cores == 10` )

## cpupower命令
### CPU频率策略
#### 静态策略
**performance**：将CPU频率固定工作在其支持的最高运行频率上，不动态调节，可以获取到最大的性能。

**powersave**: 将 CPU 频率设置为最低的所谓 “省电” 模式，CPU 会固定工作在其支持的最低运行频率上。

因此这两种 `governors` 都属于静态 `governor`，即在使用它们时 CPU 的运行频率不会根据系统运行时负载的变化动态作出调整。

这两种 `governors` 对应的是两种极端的应用场景，使用 `performance governor` 是对系统高性能的最大追求，而使用 `powersave governor` 则是对系统低功耗的最大追求。
### 动态策略
**ondemand**：按需快速动态调整 CPU 频率， 一有 cpu 计算量的任务，就会立即达到最大频率运行，等执行完毕就立即回到最低频率；

**conservative**: 与 ondemand 不同，平滑地调整 CPU 频率，频率的升降是渐变式的, 会自动在频率上下限调整，和 ondemand 的区别在于它会按需分配频率，而不是一味追求最高频率；

### 查看
#### 查看当前cpu可用的策略
```bash
# cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
conservative userspace powersave ondemand performance
```

#### 查看当前cpu生效的策略
```bash
# cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
conservative
```
#### 查看当前CPU频率
```bash
# cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
1000000

or 

# lscpu | grep -i "CPU MHz"
CPU MHz:               2800.102

or

# cat /proc/cpuinfo | grep -i "CPU MHz"


```

```bash
# cpupower monitor -m Mperf
              |Mperf
PKG |CORE|CPU | C0   | Cx   | Freq
   0|   0|   0|  0.05| 99.95|  2896
   0|   0|  28|  0.00|100.00|  2890
   0|   1|   2|  0.01| 99.99|  2931
   0|   1|  30|  0.02| 99.98|  2900
   0|   2|   4|  0.01| 99.99|  2901
   0|   2|  32|  0.01| 99.99|  2892
   0|   3|   6|  0.01| 99.99|  2895
   0|   3|  34|  0.04| 99.96|  2898
   0|   4|   8|  0.07| 99.93|  2898
   0|   4|  36|  0.11| 99.89|  2897
   0|   5|  10|  0.01| 99.99|  2910
   0|   5|  38|  0.00|100.00|  2893
   0|   6|  12|  0.01| 99.99|  2909
   0|   6|  40|  0.00|100.00|  2886
   0|   8|  14|  0.01| 99.99|  2919
   0|   8|  42|  0.01| 99.99|  2893
   0|   9|  16|  0.13| 99.87|  2898
   0|   9|  44|  0.00|100.00|  2867
   0|  10|  18|  0.00|100.00|  2934
   0|  10|  46|  0.00|100.00|  2875
   0|  11|  20|  0.01| 99.99|  2914
   0|  11|  48|  0.05| 99.95|  2901
   0|  12|  22|  0.02| 99.98|  2901
   0|  12|  50|  0.00|100.00|  2875
   0|  13|  24|  0.01| 99.99|  2898
   0|  13|  52|  0.01| 99.99|  2893
   0|  14|  26|  0.01| 99.99|  2898
   0|  14|  54|  0.01| 99.99|  2900
   1|   0|   1|  0.02| 99.98|  2905
   1|   0|  29|  0.11| 99.89|  2899
   1|   1|   3|  0.01| 99.99|  2903
   1|   1|  31|  0.02| 99.98|  2898
   1|   2|   5|  0.01| 99.99|  2911
   1|   2|  33|  0.02| 99.98|  2896
   1|   3|   7|  0.01| 99.99|  2900
   1|   3|  35|  0.01| 99.99|  2890
   1|   4|   9|  1.51| 98.49|  2898
   1|   4|  37|  0.02| 99.98|  2891
   1|   5|  11|  0.02| 99.98|  2895
   1|   5|  39|  0.17| 99.83|  2899
   1|   6|  13|  0.03| 99.97|  2898
   1|   6|  41|  0.06| 99.94|  2899
   1|   8|  15|  0.01| 99.99|  2891
   1|   8|  43|  0.03| 99.97|  2899
   1|   9|  17|  0.00|100.00|  2939
   1|   9|  45|  0.00|100.00|  2889
   1|  10|  19|  0.01| 99.99|  2907
   1|  10|  47|  0.01| 99.99|  2932
   1|  11|  21|  0.00|100.00|  2912
   1|  11|  49|  0.00|100.00|  2909
   1|  12|  23|  0.08| 99.92|  2898
   1|  12|  51|  0.01| 99.99|  2895
   1|  13|  25|  0.01| 99.99|  2884
   1|  13|  53|  0.04| 99.96|  2899
   1|  14|  27|  0.01| 99.99|  2876
   1|  14|  55|  0.56| 99.44|  2898
```

```bash
# 查看当前服务器是否开启了睿频(0:enable 1:diable)
[root@node1 ~]# cat /sys/devices/system/cpu/intel_pstate/no_turbo

# 关闭CPU环保模式，更改为性能模式 
[root@node1 ~]# cpupower -c all frequency-set -g performance
```

#### 查看当前所有CPU的信息
```bash
[root@CENTOS ~]# cpupower -c all frequency-info
analyzing CPU 0:
  driver: acpi-cpufreq
  CPUs which run at the same hardware frequency: 0
  CPUs which need to have their frequency coordinated by software: 0
  maximum transition latency: 10.0 us
  hardware limits: 1000 MHz - 1.90 GHz
  available frequency steps:  1.90 GHz, 1.90 GHz, 1.80 GHz, 1.70 GHz, 1.60 GHz, 1.50 GHz, 1.40 GHz, 1.30 GHz, 1.20 GHz, 1.10 GHz, 1000 MHz
  available cpufreq governors: conservative userspace powersave ondemand performance
  current policy: frequency should be within 1000 MHz and 1.90 GHz.
                  The governor "conservative" may decide which speed to use
                  within this range.
  current CPU frequency: 1000 MHz (asserted by call to hardware)
  boost state support:
    Supported: yes
    Active: yes
```

查看某个CPU的信息
```bash
cpupower -c 0 frequency-info        #查看CPU0的信息
cpupower -c 1 frequency-info        #查看CPU1的信息
cpupower -c 2 frequency-info        #查看CPU2的信息
```

### 设置所有CPU的模式
```bash
cpupower -c all frequency-set -g powersave
cpupower -c all frequency-set -g "conservative"
```
# CPU驱动器
通过`cpupower`命令可以查看系统当前使用的驱动。

## intel_pstate驱动
intel_pstate 驱动是 Linux内核中的一个组件，专门用于管理现代 Intel 处理器的电源管理和性能状态。它提供了一种机制，允许操作系统更精细地控制 CPU 频率和功耗，以适应不同的工作负载和性能需求。

### 工作原理
intel_pstate 驱动利用了 Intel 处理器的硬件特性，如 Performance-Scale Technology (P-state) 和 Power-Saving-Mode (P-state)。这些特性允许 CPU 根据当前负载和电源策略动态调整其运行频率和功耗状态。

intel_pstate 驱动通过与处理器的硬件接口交互，读取和设置 CPU 性能状态。它使用硬件提供的 P-state 转换表来确定 CPU 频率和电压设置。驱动程序还与操作系统的调度器和电源管理策略协同工作，以确保 CPU 状态的变化与系统需求相匹配。

### 配置开启intel_pstate
要配置`intel_pstate=enable`，通常需要通过修改Linux系统的内核启动参数来实现。以下是一般步骤，但请注意，具体步骤可能会根据您使用的Linux发行版和内核版本有所不同。

**(1) 编辑GRUB1配置文件**：
打开GRUB配置文件/etc/default/grub，找到GRUB_CMDLINE_LINUX_DEFAULT或GRUB_CMDLINE_LINUX这一行。在这个参数中添加intel_pstate=enable。例如，如果原来的配置是这样的：
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```
应该修改为：
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_pstate=enable"
```

**(2)更新GRUB配置**
```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

**(3)重启系统**：
为了使更改生效，需要重启系统。在重启后，`intel_pstate`驱动应该已经启用。

**(4)验证**：通过cpupower命令可以查看。

![](attachments/Pasted%20image%2020240807154207.png)

## acpi-cpufreq驱动

针对AMD处理器的服务器，当前配置显示使用的是acpi-cpufreq驱动，在这里也扩展一下知识。

acpi-cpufreq驱动是Linux内核中的一个CPU频率缩放驱动程序，它基于高级配置和电源接口（ACPI）规范来实现CPU性能状态的管理。ACPI是一个开放标准，被所有主流操作系统所支持，用于操作系统和硬件之间的电源管理通信。

### 工作原理

acpi-cpufreq驱动通过与系统的ACPI接口交互来调整CPU的频率和电压。它利用ACPI提供的接口和方法来查询和设置CPU的性能状态（P-states），这些状态定义了CPU的频率和电压水平。驱动程序可以响应系统负载的变化，通过调整CPU频率来优化性能和功耗。

![](attachments/Pasted%20image%2020240807154552.png)

### 驱动对比

如上的2个截图，对比如下：

**性能模式**：
- intel_pstate：只支持powersave模式和performance模式；
- acpi-cpufreq：支持很多种模式`conservative userspace powersave ondemand performance`多种模式；


**频率范围**：
- intel_pstate驱动支持CPU的动态频率调整，包括Turbo Boost。
CPU频率范围为1.20GHz至3.00GHz(尽管CPU规格为2.00GHz)，如果我们检查CPU规格页面，它显示2.00GHz是基本频率，而Max Turbo频率为3.00GHz。

- acpi-cpufreq驱动通常只使用基本频率
acpi-cpufreq频率为了实现更好的能效，会控制在1.20 GHz和2.00 GHz以内。



# 其他
## CPU性能和CPU的频率关系
CPU单核频率高不一定代表性能好，因为CPU的性能不仅取决于其频率，还与许多其他因素相关。

CPU的性能与其架构、缓存、指令集、制造工艺、功耗管理等因素密切相关。单核频率虽然是衡量CPU性能的一个指标，但不能单纯地以此作为衡量标准。例如，同样是3GHz的CPU，如果一个CPU的架构设计得好，缓存、指令集等方面也做得不错，而另一个CPU的架构设计不佳，缓存较小，指令集较少，那么前者的性能就会比后者更好。
### cpu主频越高的影响
并不是说提高主频就可以无限提高CPU的运算速度。因为提高主频也会带来一些问题和挑战：
（1）提高主频会增加CPU的功耗和发热量，需要更好的散热和供电系统来保证CPU的稳定运行
![](attachments/Pasted%20image%2020240131153023.png)

（2）提高主频会增加CPU与其他硬件之间的同步难度，需要更高速的总线和内存来匹配CPU的速度。
![](attachments/Pasted%20image%2020240131153040.png)

（3）提高主频会受到制造工艺和物理极限的限制，需要更精密和更先进的技术来制造更小更密集的晶体管和导线。
### cpu主频的设置调教
在实际应用中，不同厂商和不同架构的CPU会根据自己的特点和优势来平衡主频和其他性能指标，以达到最佳的效率和效果。
英特尔公司的`Pentium 4`系列CPU以高主频著称，最高可达3.8GHz，但由于其流水线过长、缓存过小、指令集过多等原因，其实际运算速度并不如同主频那么高。

AMD公司的Athlon FX系列CPU以低主频高性能著称，最高只有2.8GHz，但由于其流水线较短、缓存较大、指令集较少等原因，其实际运算速度却能超过英特尔公司的Pentium 4系列CPU。
### 小结
因此，CPU主频与CPU的运算速度有一定的关系，但并不是一个简单的线性关系。CPU的运算速度还要看CPU的流水线、缓存、指令集、位数等各方面的性能指标。也就是说，主频仅仅是CPU性能表现的一个方面，而不代表CPU的整体性能。
在实际应用中，还需要综合考虑CPU的频率、功耗、散热等因素。

综上所述，我们可以看出，CPU主频并不是衡量CPU性能的唯一标准，也不是越高越好。我们在选择或评价CPU时，还要考虑其他方面的因素，如核心数、线程数、缓存大小、架构设计等。只有综合考虑这些因素，才能找到适合自己需求和预算的最佳CPU。

## 为什么我们现在CPU频率基本还停留在4GHZ平台呢
CPU处理器中有一条金科玉律，那就大名鼎鼎的摩尔定律，它阐述了晶体管数目与性能提升的关系，之于它究竟是还活着，还是像死了般活着还很难说。但是我们今天要讲的是另一条不太出名的定律——登纳德缩放比例（Dennard Scaling）。

1974年内存之父罗伯特登纳德在其论文中表示，晶体管面积的缩小使得其所消耗的电压以及电流会以差不多相同的比例缩小，这个就是登纳德缩放比例定律。

我们先了解晶体管功耗是如何计算的，静态功耗的就是常规的电压乘以电流，W=V x I。而晶体管在做 1和 0的相互转换时会根据转换频率的高低产生动态功耗，W=V2 x F。显然，频率越高，功耗就越大。

根据登纳德缩放比例，工艺的提升，可以让晶体管们做的更小，导通电压更低，显然就弥补了频率提升带来功耗增加问题。但是我们的工艺并不是无休止境地提升，很快就会进入了一个长期的技术平台期，7nm以后路将会十分艰辛。

而且晶体管尺寸缩小以后，静态功耗不减反增，带来了很大的热能转换，加之晶体管之间的积热十分严重，让CPU散热问题成为亟待解决的问题。散热做不好，CPU寿命大大下降，而且目前普遍存在的动态频率技术，过热会让CPU处于最低工作频率，高频只是个装饰、是个笑话。

**单纯提高CPU时钟频率因为随之而来的散热问题而变得不再现实，毕竟我们不会无时无刻地使用液氮为CPU降温，所以Intel、AMD都很识趣地停止了高频芯片的研发，转而向低频多核的架构开始研究。**

## 四核CPU频率3.2Ghz是指每个核心频率都是3.2Ghz还是指总共四个核心加起来才3.2ghz？

CPU的核心频率指标，指的都是单个核心的运行频率，而不是所有核心的频率和(总的频率，依旧是1.7，不能简单的相加)。

1. 现在的CPU很多都是多核产品，每个核心的工作频率都是一样的，所以标识CPU频率时，都是以单个核心的频率来标识的。

2. CPU并没有所谓的总频率一说，只有主频一个指标，主频可以代表每个核心运算的频率，但并不是所有核心的频率总和。

3. 多核心处理器在处理多任务时会发挥更好的性能优势，如果处理单一任务，一般来说主频越高速度越快。主频跟核心数多少没有关系。  
而且，需要注意，现在CPU的运行频率已经不是衡量CPU性能的主要指标，不能以频率来判断CPU的性能优劣。

## CPU是多核心还是要高频率？
核心越多，处理器的并行处理能力越强，换句话说，就是能够同时处理任务的数量多。
主频越高，说明在处理单个任务的时候更快。

你可以把核心数量看作“手”的数量——数量越多，同时搬起的东西就越多；而主频就相当于“手”的力量——力量越大，就能胜任更繁重的工作。


# 参考
```bash
# cpu主频越高越好吗 cpu主频参数介绍
https://www.160.com/article/4725.html

# [CPU信息解读]
https://leenzhu.com/posts/cpu.html
```