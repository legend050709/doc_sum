```table-of-contents
```
# 队列（Queues）
## 整体流程
实际上，队列在整个日志的生命周期中都存在，它是Rsyslog的核心，一般情况下，我们感觉不到它的存在；然而，从日志的产生到被处理的过程，都必须经过两个队列，一个是主[消息队列](https://cloud.tencent.com/product/cmq?from_column=20065&from=20065)（main message queue），另一个是动作队列（action queue）。

![](attachments/Pasted%20image%2020231030141415.png)
![](attachments/Pasted%20image%2020231030141419.png)

从上图中可以看到：
- 日志产生后，先经过预处理器然后就被压入main message queue等待后续的处理
- 在进入action queue之前，日志被解析器和过滤器处理，它们的作用是读取rsyslog.conf配置文件中设置的规则，和日志中的内容进行对比，然后发送到合适的action queue
>一旦日志进入到这个action queue之后，就会从主消息队列中删除。
- 日志真正被处理的阶段发生在进入action queue之后，action processor（动作处理器）会从action queue中获取最先进入队列的日志进行处理，根据规则进行日志的输出，例如写入文件，录入数据库、发送到远程服务器，甚至是把它们丢弃。

## 基本概念


## **主消息队列（main message queue）**
rsyslog中只有一个主消息队列，任何消息都要先进入这个队列，然后直到进入到动作队列之后消息才会从这个队列中删除。
通常，我们都不会太过在意主消息队列的设置，因为默认的设置已经工作得很好；往往rsyslog中的关于队列的配置，都是针对动作队列的，这也是我们接下来要说的。

## **动作队列（action queue）**
消息经过主消息队列之后，就被rule processor解析和处理，然后根据预先配置的规则压入各自的动作队列。进入动作队列之后消息最终被消费掉。例如输出到指定的地方。
正是如此，往往我们都会根据实际情况对动作队列的行为作出一些适当的调整。

## **队列的种类**
队列的类型划分为下面4种：

- Direct queue
- Disk queue
- In-memory queue （LinkedList/FixedArray）
- Disk-Assisted In-memory queue


设置队列类型的语法如下：
```javascript
$<Object>QueueType  <QueueTypeValue>  #Object可以代表MainMsg或Action
```





###  Direct Queue
Direct queue是Action Queue 默认的行为。
实际上是一个 `non-queuing` queue。它直接把消息从 producer 传递给 customer。
这种模式适合简单的把日志写入本地, 效率非常高。

![](attachments/Pasted%20image%2020231030153333.png)

> Direct queue是唯一一个会把执行结果（成功/失败）从消费者（action processor）返回给生产者的队列。action processor正是通过这个返回值提醒action queue，让action queue取回这些处理失败的消息，如此循环，直到消息处理成功。

**就是因为 direct queue 实际上没有队列，而是直接从生产者传递日志到消费者。所以，消费者可以将日志消费的结果（成功/失败）返回给生产者。**

>在direct queue下，同一条日志如果被多个动作处理器消费，这个时候，同一条日志会被复制到各个动作队列中，那么可能会造成的现象是，当你使用discard丢弃日志的时候，会发现discard指令没有生效，原因是：discard指令丢弃的是原始日志的副本，而原始的日志会继续活动在原来的工作流中。

### Disk Queue
Disk queue使用硬盘作为消息缓冲设备，而不会使用任何内存作为缓冲。因此，它的最大好处是可靠，缺点是，它的写入速度是最慢的。如果不是必须，不推荐使用这种队列。

当Disk queue写入文件的时候，它是以块的方式接收消息的，一个数据块是一个文件，每一个数据块默认是10m，文件有一个前缀，可以通过`$<Object>QueueFileName`配置。
文件的默认大小可以使用`$<Object>QueueMaxFileSize`设置。
每一个队列可以使用不同的位置保存数据，通过`$WorkDirectory`指令设置，这个指令要在队列创建之前配置。

### In-memory Queue
这种类型的队列把所有的消息都保存在内存中，因此它的处理速度非常快，缺点是当电脑关闭或死机的时候，所有未被处理的消息都会丢失。如果希望电脑关机的时候保存这些消息，可以使用`$<Object>QueueSaveOnShutdown`设置。

有两种类型的内存队列，它们是：
- FixedArray queue
- LinkedList queue
它们使用内存作为缓存，所以效率非常高，但是相对的它们的可靠性没有Disk Queue好。

**main message queue默认是FixedArray Queue， main message queue的默认上限是10000个消息。**
FixedArray队列预先分配一定的内存来保存这些消息，它的缺点是，无论你的日志有多少，它都需要完全占用这些内存；好处是当数据量不大的时候，它的性能是最好的。

LinkedList队列和FixedArray队列不同，它的内存是运行时动态分配的，会根据数据量的不同而作出调整，好处是内存利用率高，LinkedList队列适合使用在一些突发数据量大的场景。

### Disk-Assisted In-memory Queue（DA队列）
它集成了In-Memory 和 Disk 这两种队列的优点。
这种队列实际上是以内存队列为主，Disk Queue为辅的队列。在正常情况下，不会使用辅助的Disk queue，但当内存队列被填满，或者主机关闭的时候，Disk Queue就会被激活，数据被写入硬盘。结合两者使用，可以同时满足速度和数据的可靠性。

这种类型的队列创建指令是（以action queue为例子）：

```javascript
$ActionQueueType LinkedList
$ActionQueueFileName fileName

也就是说，在建立一个普通的内存队列之后，再使用指令设置一个保存文件，两者组合在一起之后，就成了Disk-Assisted In-memeory Queue。
```
> DA队列实际是两个队列，一个普通的memory队列 (called the "primary queue")和一个disk队列(called the "DA queue")。当达到一定条件后，DA队列就会被激活。

#### 配置
这里给出一份Action使用DA队列的配置:
```c

# 这两个是全局的 
$ActionResumeRetryCount                  3
$ActionResumeInterval                    10
# 以下是每个ActionQueue自己的配置
$ActionQueueType                         LinkedList
$ActionQueueFileName                     da_queue
$ActionQueueMaxFileSize                  100M         # 设置单个disk文件的大小
$ActionQueueMaxDiskSpace                 10G          # 设置最大占用空间
$ActionQueueDisacdSeverity               3            # 设置忽略的等级
$ActionQueueLowWaterMark                 5000         # 默认是2000
$ActionQueueHighWatermark                15000        # 默认是8000
$ActionQueueDiscardMark                  30000        # 默认是9750
$ActionQueueSize                         80000        # 文档没写，测试发现默认是1000
$ActionQueueSaveOnShutdown               on
```
- QueueHighWatermark的作用是：
当队列中的数据超过这个设置的值的时候，要么把数据保存，要么把数据丢弃，如果是Disk-Assisted In-memory Queue，队列中的数据超过这个值，Disk Queue就会被激活。

- QueueLowWatermark的作用：
和上面的相反，这是一个低水位设置，当数据小于这个值的时候，就停止相关的操作，如果是Disk-Assisted In-memory Queue，数据低于这个值，Disk Queue就会被取消激活状态。

### 小结
main message queue默认是FixedArray Queue（In-memory Queue）， main message queue的默认上限是10000个消息。
Action Queue 默认的行为是 Direct queue。

# 队列丢消息
## 限制队列的容量
对于容量限制的指令有两个，它们是：
```c
 $<Object>QueueSize  <number>
 $<Object>QueueHighWaterMark <number>
```

两者之间有细微的差别：
`$<Object>QueueSize`用于设置队列的总容量，即队列可容纳的消息数量。
而`$<Object>QueueHighWaterMark`只用于`disk-assisted`类型的队列，当队列中的消息数量达到这个值之后，消息就会被写入到硬盘。但是这种行为是有依赖性的，仅当日志的输出目标无法到达的时候（数据库无法访问，远程服务器离线等），它才会发生。

## 丢弃消息（Discarding Messages）
控制这个行为的指令是`$<Object>QueueDiscardMark`，当队列中的消息达到这个指定的值时，消息就会被丢弃。
至于丢弃哪一种消息，则由`$<Object>QueueDiscardSeverity`指令控制，这个指令接受以文字表示的等级或以数字表示的等级。
具体的等级和对应的数字如下：

```javascript
Numerical Code        Severity
 
     0                Emergency: system is unusable
     1                Alert: action must be taken immediately
     2                Critical: critical conditions
     3                Error: error conditions
     4                Warning: warning conditions
     5                Notice: normal but significant condition
     6                Informational: informational messages
     7                Debug: debug-level messages
```

## 队列的终止
我们不能控制队列的终止，只有在系统被关闭（或者程序被杀死）的那一刻，队列才会结束。
当队列终止的时候，可能会遇到这样的情况：队列中依然有数据尝试进入。这种情况rsyslog会试图处理这些数据。
如果希望控制这些数据的处理时间，可以使用这个指令：`$<Object>QueueTimeoutShutdown <milliseconds>`。当时间超过这个值，队列中的所有数据被丢弃。如下图：
![](attachments/Pasted%20image%2020231030161646.png)

另一种情况是，当超时后，依然希望队列处理完当前正在被处理的数据再关闭，那么可以使用`$<Object>QueueTimeoutActionCompletion`指令，它设置了处理当前数据的时间，也就是说除了当前正在被处理的消息外，其他任何的消息都被丢弃。
![](attachments/Pasted%20image%2020231030161719.png)

和图一不同的是，图二保留了当前正在被处理的消息（队列最前绿色）。

> 注：如果不希望丢弃任何消息，可以使用`$<Object>QueueSaveOnShutdown`指令。这个指令要求队列是Disk Queue或者Disk-assisted Queue。

# 为什么需要队列
把这个问题放到最后才说，是有原因的，因为到这个位置，才比较清楚队列都干了什么。知道它的作用，才能回答这个问题。
从上面几种队列可以看到，队列的作用无非是两种，一种是加速，另一种是可靠。

那么，现在再回答一个问题，**为什么默认Action Queue是Direct Queue（不进入队列）？**
> 原因很简单：比起其他操作，例如写入数据库或者是通过网络协议传输日志，直接写入硬盘速度是最快的，也是最可靠的，因此，它使用的是Direct Queue。参考：[rsyslog performance: main and action queue workers](https://cloud.tencent.com/developer/tools/blog-entry?target=http%3A%2F%2Fblog.gerhards.net%2F2013%2F06%2Frsyslog-performance-main-and-action.html)

另一个问题是：**我们什么时候使用队列？**
> 这个问题其实可以根据上面的解释回答，也就是在一些慢操作和可靠性不高的场景（写入数据库、网络传输）。


![](attachments/Pasted%20image%2020231030162039.png)


# 注意事项
如果消息最终不能被消费（输出到指定位置），那么这些消息就会停留在先前的队列中。
这就有可能会导致队列被填满，一旦队列填满，后续的输入消息就不能再进入消息队列，最终造成某些服务无法进行日志记录，最坏的后果是导致该服务无法正常提供服务。

比如某次故障时, CPU/Disk/Mem 都正常, 但某些进程异常卡顿, 排查2天, 竟然发现是 rsyslog 的锅: 因为 ES 挂了, rsyslog-elasticsearch 队列被打满且阻塞, 然而 rsyslog 默认主队列是 direct, 也就是必须等待elasticsearch子列队超时才返回, 导致 sshd, sudo , rsync , php.syslog等凡是用到 syslog 的地方被阻塞, 后果很严重!!!
**所以合理配置主队列和动作队列的类型及丢弃策略非常必要!!!**

# 参考
```c
# rsyslog queue队列权威指南
https://cloud.tencent.com/developer/article/1683274

# 官方文档
https://www.rsyslog.com/category/guides-for-rsyslog/
https://www.rsyslog.com/doc/v8-stable/concepts/queues.html

```