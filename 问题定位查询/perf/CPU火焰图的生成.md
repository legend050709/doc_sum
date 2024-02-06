```table-of-contents
```
# 介绍
![](attachments/image%20(1).png)
火焰图是基于`stack`信息生成的`SVG` 图片, 用来展示 CPU 的调用栈。
说明：
y 轴表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶部就是正在执行的函数，下方都是它的父函数。
x 轴表示抽样数，如果一个函数在 x 轴占据的宽度越宽，就表示它被抽到的次数多，即执行的时间长。
火焰图就是看顶层的哪个函数占据的宽度最大。只要有"平顶"(plateaus)，就表示该函数可能存在性能问题。颜色没有特殊含义, 因为火焰图表示的是 CPU 的繁忙程度, 所以一般选择暖色调.

# 火焰图类型
火焰图常见的类型有 `On-CPU`, `Off-CPU`, 还有 `Memory`, `Hot/Cold`, [Differential](http://www.brendangregg.com/blog/2014-11-09/differential-flame-graphs.html "Differential") 等等. 
`on-CPU`/`off-cpu` 的区别就是一个是用于CPU是性能瓶颈，一个是IO是性能瓶颈.

另外一种情况就是如果无法确定当前的系统瓶颈, 可以通过压测工具来确认 : 通过压测工具看看能否让CPU使用率趋于饱和, 如果能那么使用 `On-CPU` 火焰图, 如果不管怎么压, CPU 使用率始终上不来, 那么多半说明程序被 `IO` 或锁卡住了, 此时适合使用 `Off-CPU` 火焰图. 你可以通过压测工具进行测试


# 制作火焰图
Github上有`Brendan D. Gregg` 的 `Flame Graph` 工程实现了一套生成火焰图的脚本.我们可以直接克隆下来直接用。
```bash
git clone https://github.com/brendangregg/FlameGraph.git
```
## 流程
生成火焰图，我们一般都遵循以下流程
![](attachments/Pasted%20image%2020240130153434.png)

- `捕获堆栈`: 使用`perf`捕捉进程运行堆栈信息
- `折叠堆栈`: 对抓取的系统和程序运行每一时刻的堆栈信息进行分析组合, 将重复的堆栈累计在一起, 从而体现出负载和关键路径，通过`stackcollapse`脚本完成
- `生成火焰图`：分析 stackcollapse 输出的堆栈信息渲染成火焰图

### 获取堆栈
```bash
perf record -F 99 -C 1 --call-graph dwarf -- sleep 30

默认在当前路径下生成一个 perf.data 文件
```
### 折叠堆栈
`Flame Graph`中提供了抓取不同信息的脚本，可以按需使用。下面我们需要对捕获到的进程堆栈信息`perf.data`进行折叠，生成折叠的堆栈信息:
```bash
 perf script -i /root/perf.data &> /root/perf.unfold
```

用 `stackcollapse-perf.pl` 将 perf 解析出的内容 `perf.unfold` 中的符号进行折叠
```bash
 ./stackcollapse-perf.pl /root/perf.unfold &> /root/perf.folded
```
最后就是生成火焰图
```bash
./flamegraph.pl /root/perf.folded > /root/perf.svg
```

最后在谷歌浏览器上打开该火焰图文件（perf.svg）：
![](attachments/Pasted%20image%2020240130153922.png)


# 参考
```bash
# [火焰图：全局视野的Linux性能剖析]
(https://segmentfault.com/a/1190000023103508)


```