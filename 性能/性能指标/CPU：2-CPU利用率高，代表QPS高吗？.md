```table-of-contents
```

# CPU和QPS的关系

# IPC
## 定义
## IPC如何影响程序性能(QPS)
## 为什么IPC有高有低？
## 查看
### 查看perf可用的事件

**（1）事件可用性**：

==不同 CPU 架构支持的事件不同，可通过 `perf list` 查看完整列表==：

```c
perf list | grep -E 'cycles|instructions|cache-misses|branch'
```

```bash
# perf list | grep -E 'cycles|instructions|cache-misses|branch'
  branch-instructions OR branches                    [Hardware event]
  branch-misses                                      [Hardware event]
  bus-cycles                                         [Hardware event]
  cache-misses                                       [Hardware event]
  cpu-cycles OR cycles                               [Hardware event]
  instructions                                       [Hardware event]
  ref-cycles                                         [Hardware event]
  branch-load-misses                                 [Hardware cache event]
  branch-loads                                       [Hardware cache event]
  branch-instructions OR cpu/branch-instructions/    [Kernel PMU event]
  branch-misses OR cpu/branch-misses/                [Kernel PMU event]
  bus-cycles OR cpu/bus-cycles/                      [Kernel PMU event]
  cache-misses OR cpu/cache-misses/                  [Kernel PMU event]
  cpu-cycles OR cpu/cpu-cycles/                      [Kernel PMU event]
  cycles-ct OR cpu/cycles-ct/                        [Kernel PMU event]
  cycles-t OR cpu/cycles-t/                          [Kernel PMU event]
  instructions OR cpu/instructions/                  [Kernel PMU event]
  ref-cycles OR cpu/ref-cycles/                      [Kernel PMU event]
```


**（2）常用事件列表**：

|事件名称|描述|
|---|---|
|`cycles`|CPU 周期数|
|`instructions`|已完成的指令数|
|`cache-misses`|缓存未命中总次数|
|`branch-misses`|分支预测失败次数|
|`L1-dcache-load-misses`|L1 数据缓存加载未命中|
|`LLC-load-misses`|最后一级缓存（如 L3）加载未命中|




### 查看ipc、branch-miss、cache-miss 的百分比
```bash
perf stat -p `pidof xxx` -e cycles,instructions,cache-misses,branch-misses --output perf.log

perf stat -p `pidof xxx` -e cycles,instructions,cache-references,cache-misses,branches,branch-misses -I 2000 --output perf.log

比如：`-I 2000`：每 2000 毫秒（2 秒）输出一次统计。
```
执行过程中，若未指定 `sleep`，需手动按 `Ctrl+C` 终止监控。
默认统计的是整个监控时间内的累计值，若需实时跟踪，可使用 `-I <毫秒>` 参数分时段输出。

输出结果1，如下所示：
```bash
# perf stat -p `pidof xxx` -e cycles,instructions,cache-misses,branch-misses --output perf.log
# cat perf.log
# started on Wed May 28 10:59:37 2025


 Performance counter stats for process id '25581':

    30,494,983,348      cycles
    57,855,473,931      instructions              #    1.90  insn per cycle
           397,222      cache-misses
        45,786,664      branch-misses

      10.539923395 seconds time elapsed
```

输出结果2，如下所示：
```bash
# perf stat -p `pidof xxx` -e cycles,instructions,cache-references,cache-misses,branches,branch-misses -I 2000 --output perf.log
# cat perf.log
# started on Wed May 28 11:17:15 2025

#           time             counts unit events
     2.000191619      5,786,733,324      cycles
     2.000191619     11,364,043,013      instructions              #    1.96  insn per cycle
     2.000191619          6,108,039      cache-references
     2.000191619            140,236      cache-misses              #    2.296 % of all cache refs
     2.000191619      2,162,083,568      branches
     2.000191619          5,877,888      branch-misses             #    0.27% of all branches
     4.000334077      5,786,795,378      cycles
     4.000334077     11,349,397,735      instructions              #    1.96  insn per cycle
     4.000334077          6,148,261      cache-references
     4.000334077            151,784      cache-misses              #    2.477 % of all cache refs
     4.000334077      2,159,357,287      branches
     4.000334077          5,943,654      branch-misses             #    0.28% of all branches
     6.000442478      5,786,731,890      cycles
     6.000442478     11,348,074,040      instructions              #    1.96  insn per cycle
     6.000442478          6,103,975      cache-references
     6.000442478            126,835      cache-misses              #    2.072 % of all cache refs
     6.000442478      2,159,191,620      branches
     6.000442478          6,085,460      branch-misses             #    0.28% of all branches
     7.378540595      3,987,066,636      cycles
     7.378540595      7,814,851,450      instructions              #    1.46  insn per cycle
     7.378540595          4,201,402      cache-references
     7.378540595             90,284      cache-misses              #    1.601 % of all cache refs
     7.378540595      1,486,930,433      branches
     7.378540595          4,248,230      branch-misses             #    0.21% of all branches
```

#### 计算未命中百分比
需要通过 `perf` 收集 **分母和分子** 的事件：
- **IPC 计算**：`instructions/cycles`，比如：值为`0.85`，表明每周期执行约 `0.85` 条指令，可能存在内存或分支瓶颈。
- **缓存未命中率** = `cache-misses / cache-references`
- **分支预测失败率** = `branch-misses / branches`

|事件名称|描述|
|---|---|
|`cache-references`|缓存引用总次数（分母）|
|`cache-misses`|缓存未命中次数（分子）|
|`branches`|分支指令总数（分母）|
|`branch-misses`|分支预测失败次数（分子）|

```bash
sudo perf stat -p <PID> \ -e cycles,instructions,cache-references,cache-misses,branches,branch-misses -- sleep 10 # 监控10秒后自动停止（可选）

perf stat -p `pidof xxxx` -e cycles,instructions,cache-references,cache-misses,branches,branch-misses --output perf.log
```
输出结果，如下所示：
```bash
# cat perf.log
# started on Wed May 28 11:12:09 2025


 Performance counter stats for process id '43623':

    22,090,383,323      cycles
    43,485,866,266      instructions              #    1.97  insn per cycle
        23,085,426      cache-references
           948,340      cache-misses              #    4.108 % of all cache refs
     8,273,916,745      branches
        22,863,497      branch-misses             #    0.28% of all branches

      10.645870435 seconds time elapsed
```

### 特定缓存层级的未命中率
#### L1 数据缓存未命中率
```bash
perf stat -e L1-dcache-loads,L1-dcache-load-misses -p PID

计算：`L1-dcache-load-misses / L1-dcache-loads`
```
#### 最后一级缓存（LLC，如 L3）未命中率
```bash
perf stat -e LLC-loads,LLC-load-misses -p PID

计算：`LLC-load-misses / LLC-loads`
```
### 小结
通过 `perf stat -p <PID>`，您可以实时监控运行中进程的 CPU 性能指标，快速诊断以下问题：
- **低 IPC** → 内存延迟（缓存未命中）或分支预测失败。
- **高缓存未命中率** → 优化数据局部性（如调整数据结构或预取）。
- **高分支预测失败率** → 简化条件逻辑或使用无分支编程。


# 数据依赖和流水线并行



# 参考
```bash

```