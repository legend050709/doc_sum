```table-of-contents
```

# ps命令选项解析

了解ps命令首先需要从ps命令的选项格式入手。像其他很多linux shell命令一样，ps命令的选项也有长格式和短格式的区别。短选项中也可以带中横线、也可以不带中横线。

## 选项格式

根据选项长短和是否有横线的情况，ps命令的选项可以分为以下3类：

- BSD风格语法，必须不能以中横线开头；
    
- SYSV风格语法，必须仅一个中横线开头；
    
- GNU风格语法，必须以两个中横线开头；


![](attachments/Pasted%20image%2020240508193623.png)


不过linux ps命令的长选项并不多，而且几乎每个长选项都有一个功能完全相同的短选项对应。在centos7环境运行如下命令可以见。

![](attachments/Pasted%20image%2020240508193645.png)

在本文中我们将主要介绍BSD和SYSV两种风格的ps命令选项。如果大家有对GNU风格的长选项使用的需求，那么可以参考对应的短选项语法即可。需要注意的是GNU风格选项都是带参数值的，例如--sid 1。

各风格的ps命令选项可以**混合使用**，比如：

![](attachments/Pasted%20image%2020240508193729.png)


Linux ps命令解析SYSV和BSD风格选项时，会分别将每组字符串都解析成单独的字母。以下三个实例，拆分前后的命令都是等价的。

![](attachments/Pasted%20image%2020240508193755.png)

从示例中可以看出，当SYSV风格语法一个中横线之后有多个字母选项时，拆分后需要给每一个字母前都加上一个中横线。也就是说-elL转换为-e -l -L，而不是转换为-e l L。

从上面例子中也可以看出，ps命令选项除了有是否加中横线的区别，字母大小写也表现为不同的选项含义。英文字母一共26个，SYSV风格选项-A到-Z和-a到-z共52个，BSD风格选项A到Z和a到z共52个。于是ps命令就有一共104个命令选项可能性。

不同版本的ps命令选项的使用可能略有出入，本文主要使用主流的centos7上的procps-ng version 3.3.10版本来说明。在这104个命令选项中，未启用的或曾经使用过现在废弃的命令选项有如下40个，分别是A、B、C、D、E、F、G、I、J、K、P、Q、R、W、Y、b、d、i、y、z、-B、-D、-E、-I、-J、-K、-Q、-R、-S、-W、-X、-Y、-b、-h、-i、-k、-r、-v、-x和-z。


# Linux ps命令的记录类选项

## 背景

差不多每一个工程师使用ps命令时应该都有这样的疑问， 使用ps aux时输出结果中记录行数要远大于只使用ps命令时（如下所示）。这其实会让很多工程师在使用ps命令查找需要的进程时心里很忐忑，会不会由于命令的选项使用不当导致ps没有列出所需要的进程信息。正是这个原因，我们首先需要搞懂ps命令影响记录行数的那些选项。

![](attachments/Pasted%20image%2020240508194056.png)

## all_processes选项

Linux ps命令的记录类选项大概有20几个之多。有些可以列出所有的进程信息，有些按某种规则筛选显示部分进程信息。如今操作系统中awk、sed和grep这些shell文本处理命令的功能都十分强大，我们重点还是掌握ps命令中那些显示所有进程信息记录的选项，其他ps命令过滤选项都可以通过shell文本处理命令（awk、sed和grep）间接实现。

**Linux ps命令显示所有进程信息的选项只有2个，即SYSV风格的-e和-A。相比之下，-e更容易记忆和书写，请大家牢记这个-e选项**。


大家知道，ps命令的所有信息都是linux kernel生成，并通过/proc/目录输出给用户空间的。在/proc/目录下，每一个以数字开头的目录，就对应一个进程信息。既然如此，通过如下命令便可一目了然。

![](attachments/Pasted%20image%2020240508194233.png)

参数-e和-A显示的进程记录数确实和proc目录下的所有进程目录数一致。

## simple_select选项

Linux ps命令的simple_select选项一共5个，具体包括-a、-d、a、g和x。

他们包括2个SYSV风格和3个BSD风格选项。2个SYSV风格和3个BSD风格的选项不能同时使用，否则会报错。2个SYSV风格或3个BSD风格内部可以组合使用，具体的组合可能性有-ad、ga、ax、gx、agx。

这里值得注意的是这种字母组合选项绝对不是单字母选项筛选规则的简单组合，ps命令给这几种组合赋予了新的筛选规则。

Linux环境下的ps命令，会对BSD风格simple_select选项部分做2个特殊处理：

- 在原来的BSD风格simple_select情况下，再额外增加一个g选项；
    
- 如果已经有a、g和x三个选项都出现了，那么就直接替换为-e选项；

按照这2个特殊处理规则，ps aux选项组合等价于ps auxg，等价于ps agx u，等价于ps -e u。


总结下来，**ps命令simple_select选项只有6种组合情况-a、-d、-ad、g、ga、gx**。

## selection_list选项

这类选项比较容易理解，都是根据进程的某个属性值对进程进行筛选。

此类选项一共13个，主要分为如下几组：

- 进程ID选项，查询PID值为一个或几个PID值范围的进程信息。

![](attachments/Pasted%20image%2020240508194734.png)


- 进程会话(session)ID选项

![](attachments/Pasted%20image%2020240508194754.png)


- 进程名称选项，显示符合当前进程名称参数的进程。这里需要注意，当进程名参数值字符串长度大于15时，只是用其前15位作为匹配条件

![](attachments/Pasted%20image%2020240508194824.png)

- 用户ID选项

![](attachments/Pasted%20image%2020240508194847.png)

- 用户组ID选项

![](attachments/Pasted%20image%2020240508194858.png)


- 进程终端(tty)选项

![](attachments/Pasted%20image%2020240508195017.png)



这些选项不但可以单独使用，还可以组合使用（如下所示）。需要注意的是这些选项之间的组合是**逻辑或**的关系，即或者符合-u选项条件或者符合-p选项条件。

![](attachments/Pasted%20image%2020240508194951.png)



## 特殊选择选项

- 当不希望结果中出现标题页头这一行信息时，h选项可以隐藏ps输出结果中的标题栏。

![](attachments/Pasted%20image%2020240508195042.png)

- 如果我们只希望列出运行中的R状态和D状态的进程，r 选项选中时将只显示其他筛选条件过滤后的结果集中的R和D状态进程。帮助手册上写只筛选R状态是不正确的，这里也包括D状态进程的筛选

![](attachments/Pasted%20image%2020240508195113.png)



# Linux ps命令输出结果排序选项

如果我们需要对输出结果进行排序，那么ps命令也给我们提供了3个选项，分别是k、f和-H。

## 字段排序选项

选项k可以让我们以某个字段为条件对输出结果进行排序，并且还可以使用+-符号设置升序排序还是降序排序。

![](attachments/Pasted%20image%2020240508195324.png)

选项k还可以使用多个字段同时对结果集排序，从输出结果可以看到，先按ppid进行升序排序，ppid值相同时，再按rss值进行升序排序。

![](attachments/Pasted%20image%2020240508195344.png)

## 树形排序选项

**每一个进程都有一个父进程，所有用户空间的进程的最终父进程都是1号进程，所有内核空间的线程的最终父线程都是2号线程**。

这样所有进程按照父子进程的关系就可以构成2个树形结构。选项f和-H就是实现这个树形排序功能的2个选项。

![](attachments/Pasted%20image%2020240508195445.png)


从上面的结果中不难看出，选项f是使用ACSII码对父子进程进行关联，选项-H是使用tab空格对父子进程进行关联。

# Linux ps命令线程展开选项

同一个进程有时候还会起多个线程，同样内核也在/proc/目录下显示了进程的线程信息，如下所示。

![](attachments/Pasted%20image%2020240508195710.png)

Linux ps同样提供了一组选项可以将每个进程的线程信息详细展现，这组选项包括H、-L、-T、M、m和-m。

在讲解这些选项之前，我们先看一个小测试。

![](attachments/Pasted%20image%2020240508195740.png)

```text
说明：

同样为了统计的准确，用h选项去掉标题栏信息。其中最后一个486的值是ps -e h的记录数，说明当前系统有486个进程。
非常巧的是486恰巧等于1217减去731的值。

从这里我们可以了解到H、-L和-T这3个选项记录数都是731，M、m和-m三个选项记录数都是1217。

```

找一个起了多线程的进程查看下具体输出内容。

![](attachments/Pasted%20image%2020240508195906.png)

选项-L的输出可以看到一共4行输出结果，第一行PID等于LWP（线程ID）的值，说明是线程组的主线程（即进程）。其余三个线程ID各不相同，但PID值都和主线程的PID值一样，说明是同一线程组的普通线程。

第二组三个选项单纯的显示不便识别，我们这里先引入一个后面讲解的O选项，额外增加一个输出值LWP。

![](attachments/Pasted%20image%2020240508200017.png)

可以看到一共5行输出结果，对照上面的输出我们可以判断出，第二组选项除了把线程组中的4个线程分别显示之外，又额外增加了一行内容专门用于显示这个线程组（即进程）的信息。我们再回头看前面的1271减去731等于486应该就很容易明白了。


# Linux ps命令的字段选项组合

## 字段组合类通用选项

很多人在使用ps命令时都会注意到，在我们输入不同的命令组合时，ps命令输出结果中列的数据项并不统一。比如下面2个命令。

![](attachments/Pasted%20image%2020240508200302.png)

Linux ps命令的aux选项组合输出PID、%CPU、%MEM、RSS、TIME等数据项，ps命令的-el选项组合输出PID、PPID、WCHAN、TIME、CMD等数据项。

首先一个问题就是：
**ps命令一共有多少数据项可以输出**。
这个问题很好回答，通过L选项很容易获取，一共有168个数据输出项。

![](attachments/Pasted%20image%2020240508200322.png)

**是什么决定了ps aux命令输出结果中恰恰包含USER、PID、%CPU、%MEM、VSZ、RSS、TTY、STAT、START、TIME和COMMAND这11个数据项呢**。

原因是ps命令中有一些选项用来对数据字段进行固定组合的作用。其中aux中的u选项就固定包含了以上11个数据输出项，并且他们的显示顺序也已经固化在代码中。

Linux ps命令这种字段组合类选项一共15个。其中6个选项用途比较广泛，其余9个选项都主要适合在查询某一种问题时使用。

本小节先介绍6个通用选项：
- 面向用户角度来显示进程状况，其中的%CPU、%MEM、VSZ和RSS字段都是非常常用的信息。

![](attachments/Pasted%20image%2020240508200439.png)


- 采用详细格式显示进程状况，此类选项所显示字段主要为一些常用字段信息。

![](attachments/Pasted%20image%2020240508200506.png)

- 采用完整格式显示进程状况，此类选项所显示字段同样为一些常用字段信息。

![](attachments/Pasted%20image%2020240508200517.png)


## 字段组合类专用选项

本小节先介绍9个适合特殊用途的专用选项：

- 采用作业(job)控制的格式显示进程状况，字段PPID、PID、PGID、SID和TPGID都是此选项的关键信息。

![](attachments/Pasted%20image%2020240508200549.png)



- 采用旧式的linux i386寄存器格式显示进程状况，很明显此选项特点是STACKP、ESP和EIP这些寄存器信息。

![](attachments/Pasted%20image%2020240508200559.png)


- 采用虚拟内存的角度显示进程状况，此选项特色字段包括MAJFL、TRS、DRS、RSS和%MEM。

![](attachments/Pasted%20image%2020240508200623.png)


- Linux或sunos操作系统中会额外增加PSR字段的显示，PSR字段是指当前进程被调度到的CPU核序号。

![](attachments/Pasted%20image%2020240508200720.png)


## 自定义字段选项


我们可以通过-o或o选项来实现自定义数据项的输出功能。比如我们对ps j这个命令字段组合的输出信息不满意，我们自定义他的输出。

![](attachments/Pasted%20image%2020240508200809.png)

前文提到过，ps命令一共可以输出168个字段，ps L命令可以显示这168个字段的详细情况。第一列小写字母是-o选项的参数，可以通过逗号隔开。第二列大写字母是ps命令输出的结果集标题栏名称。

![](attachments/Pasted%20image%2020240509104218.png)


有些说明符还提供缩写，下表是ps命令有缩写的说明符和缩写的对应关系表，一共15个。

![](attachments/Pasted%20image%2020240509104239.png)

有了说明符的缩写之后，可以对自定义字段的输出字段之间添加自定义分隔符。区别于以往ps命令各输出字段都是使用空格作为分割，使用自定义分隔符之后将更方便使用shell数据处理命令进行解析。

![](attachments/Pasted%20image%2020240509104423.png)

前文提到所有字段组合选项都默认包含4个或5字段。如果想在自定义字段组合时也默认添加一些常用字段，而同时又省去-o选项参数的输入过程，那么可以使用O或-O选项。

![](attachments/Pasted%20image%2020240509104511.png)

这2个选项O或-O，会在自定义字段之前默认增加pid字段，在自定义字段之后默认增加state、tname、time和command字段。

# Linux ps命令字段修饰选项

本节前面的选项都是决定输出结果中字段的数量和顺序，本小节将介绍几个只对输出结果中某个字段进行修饰的选项。首先来看-w和w选项。

![](attachments/Pasted%20image%2020240509105641.png)

这个实例说明，当屏幕不是很宽时，如果进程命令很长，默认情况下，会将命令超出屏幕的部分截取掉，这样势必会影响系统管理员调查问题，使用w或-w选项，就会将完整的进程命令信息显示，多出的部分换行显示。有的时候为了效果好一点，建议我们可以多使用几次w选项，比如ww、www或wwww。



# 使用
## 选项
- 选项 - L：通过此选项可以把多线程的进程展开每个线程的细颗粒度。
- 选项 - o：或选项 o，通过此选项可以自定义输出符合自己需求的字段信息。
- 选项 u：此选项可以列出 cpu 使用率、mem 使用率、rss 内存等字段信息。
- 选项 k：通过此选项可以实现对输出结果的排序。
- 选项 h：选中时可以隐藏输出结果的标题栏信息，在一些自动化脚本中使用此参数可以去除页头信息。
- 选项 - e：显示所有进程的记录，记住这个参数就可以保证把当前系统的所有进程都输出。需要筛选进程时，可以结合 grep 等文本处理命令实现目的

## 应用场景
### 查看进程的启动时间(lstart/etime/start)
一个进程启动的精确时间和进程启动后所流逝的时间。
```bash
       lstart      STARTED   time the command started.  See also bsdstart, start, start_time, and stime.
       etime       ELAPSED   elapsed time since the process was started, in the form [[DD-]hh:]mm:ss.
```

![](attachments/Pasted%20image%2020240903145759.png)

==注：正常情况下，我们只需要用到2个时间的输出，即 `lstart` 和 `etime` 即可==。


查看 nginx 进程启动的精确时间和启动后所流逝的时间：
```bash
ps -eo pid,lstart,etime,cmd | grep nginx
```

![](attachments/Pasted%20image%2020240903145619.png)

```
分析：
如上所示：
lstart的输出：
	Fri Mar 4 16:04:27 2016

etime的输出：
	41-21:14:04
```

```bash
# ps -eo pid,start,lstart,%cpu,start_time,etime,cmd | head -n 1 && ps -eo pid,start,lstart,%cpu,start_time,etime,cmd  | grep nginx
```

![](attachments/Pasted%20image%2020240903145954.png)

```
分析：
如上所示：
start 的输出：
	20:37:01

lstart 的输出：
	Fri Jul 21 20:37:01 2023

start_time 的输出：
	20:37

etime 的输出：
	01:00:17
	或
	22-09:14:44
```

#### 使用范例
```bash
ps -Leo pid,ppid,psr,tid,state,lstart,etimes,bsdtime,pcpu,vsz,rss,cmd
```

# 常用字段

 linux ps 一共最多可以输出 168 个字段，通过 `ps L` 命令可获取详情。通过字段相关选项可以获取符合用途的字段组合。
 为了让大家对 ps 命令的理解更加深入，本节会深入介绍一些常用的输出字段的含义。
 
## 进程ID类字段

进程ID类字段是ps命令字段中最基础的一类。为了能更加形象的说明这些ID的关系和含义，请大家按照如下命令顺序操作。

![](attachments/Pasted%20image%2020240509105038.png)

**对以上输出结果的字段逐条说明**：

- 字段tid表示进程的线程ID，可以看出每个线程的tid都不相同。

- 字段nlwp表示当前线程组中的线程个数，以上命令都是单线程进程，因此此值均为1。

- 字段pid表示进程ID，也可以看出每个进程的PID都不相同。

- 字段pgid表示进程组ID
上面的例子中除了和setsid结合的vmstat命令，其余三组通过shell管道连接起来的命令的pgid都相同。比如tail、awk和nl命令的pgid都为1384，且pgid值为组内第一个命令tail的pid值；iostat、sed和fold命令的pgid都为1388，且pgid值为组内第一个命令iostat的pid值。


- 字段sid表示会话ID。
上面的例子中最后一行是第三个登录终端的shell。第一个登录终端上的所有进程sid都相同，且为登录shell的pid值1351；除了和setsid结合的vmstat命令，第二个终端上的所有进程sid都相同，且为登录shell的pid值1394。

- 字段tpgid表示进程连接到的tty(终端)所在的前台进程组的ID。
除了vmstat进程之外，第二个终端上的所有进程tpgid也都等于登录shell的pid值1394。但是第一个终端上的所有进程tpgid却都等于第一个终端上又启动的那个shell的进程id值1370。充分说明了tpgid值是链接着终端的前台进程组ID值。

- 字段ppid表示父进程ID。

最后我们再来解释和vmstat结合的setsid，setsid就是使和它结合的vmstat脱离原来的会话，脱离之后pgid和sid都等于了vmstat进程的pid，同时父进程也由1号进程托管。此时也没有了所依附的终端，tpgid统一等于-1。Linux上的所有守护进程的tpgid值都是-1。


**进程ID类字段的别名情况**：
字段spid和字段lwp是字段tid的别名，字段tgid是字段pid的别名，字段pgrp是字段pgid的别名，字段sess和字段session是字段sid的别名。

## 命令名字段

命令名相关的字段一共有 3 组，如下所示。
```bash
ps -eo fname,ucmd,cmd
```

![](attachments/Pasted%20image%2020240508184526.png)

命令名字段的别名情况：字段 comm 和字段 ucomm 是字段 ucmd 的别名，字段 args 和字段 command 是字段 cmd 的别名。

建议大家掌握 ucmd 和 cmd 这 2 个字段，cmd 为长命令名字段，ucmd 为短命令名字段，可以理解为 unadorned cmd(未加修饰的命令名)。

如果程序名称长度超过 15 位，ps 命令的短命令名无法完整显示 16 位及以上的部分。下面看一个小例子来说明这个问题。

![](attachments/Pasted%20image%2020240508184553.png)

从上面的例子可以看出，当程序名称超过 15 位时，确实短命令名无法显示完整的程序名称，只显示了 15 位。进一步查看 / proc/8040 / 目录，可以发现如下信息。

![](attachments/Pasted%20image%2020240508184615.png)

查询内核代码，可以发现 comm 值取自内核 struct task_struct 结构体的 comm 属性字段。
![](attachments/Pasted%20image%2020240508184642.png)

这就告诉我们通过 ps 命令短命令字段无论如何都无法输入超过 15 位的程序名称，原因是内核数据结构原生就只支持 15 个字符长度的程序名称。


除此之外上面的例子还给我们另外一个启示，如果通过使用 SYSV 风格的短命令名就可以满足使用要求（如 ps -el），那就尽量不要使用 BSD 风格的长命令名（如 ps -e u，即 ps aux）。长命令名需要依赖内核中健康的文件系统，而当文件系统工作不正常时，往往短命令名却可以不受影响。所以我们在实际生产中偶尔会发现系统中有大量 ps aux 进程 D 住的情况。


## 进程状态字段

进程状态类字段一共有三个，分别是 s、state 和 stat，如下所示。
```bash
ps -eo s.state,stat
```
![](attachments/Pasted%20image%2020240508183950.png)

字段 s 和 state 互为别名，值为单字节进程状态。stat 为多字节进程状态。


重点介绍一下 stat 选项的多字节进程状态，查看一下 ps 命令关于这个多字节进程状态的 c 语言代码。

![](attachments/Pasted%20image%2020240508184357.png)

根据以上源代码，我们来逐条解释：

- 字符'<'表示 nice 值小于 0，nice 值最小为 - 20。进程特性 nice 值允许进程间接的影响内核调度算法。所谓 nice 就是指对其他进程的谦让程度，显然比 0 越小就越不谦让，比 0 越大就越谦让。顾名思义，越谦让的君子在 CPU 调度过程中占用 CPU 的时间会越少，反之不谦让的进程相比较可能占用更多的 CPU 时间。因此字符'<'表示此进程可能在调度过程中获得优势。
    
- 字符'N'表示 nice 值大于 0，nice 值最大为 19。因此字符'N'表示此进程可能在调度过程中不能获得优势。
    
- 字符'L'表示进程 vm_lock 值为真，即此进程有内存页被锁在内存中，这些内存页不能通过换页换出。
    
- 字符's'表示进程的 tgid(pid) 值等于进程的 session(sid) 值，这说明当前进程是会话的 leader，参考 8.1 小节。
    
- 字符'l'表示进程中的线程数量大于 1，这说明当前进程是一个多线程程序。
    
- 字符'+'表示进程的 pgrp(pgid) 值等于进程的 tpgid 前台进程组 ID，这表示当前进程在前台进程组中。
    
## 时钟 (系统) 时间类字段

时钟时间（系统时间）类字段，记录了进程开始时间点和执行的时长信息，这类字段一共有 6 组。其中 4 个记录进程开始时间点，2 个记录进程执行时长信息。

```bash
ps -ao lstart,start,bsdstart,stime,etimes,etime
```

![](attachments/Pasted%20image%2020240508184929.png)

从自动化运维脚本的角度，lstart 字段的输出信息格式更加规范便于解析，etimes 字段作为一个正整数也可以直接使用。字段 start_time 是字段 stime 的别名。

## CPU 时间和使用率字段

CPU 时间和使用率类字段一共有 5 组，记录了进程所消耗的 CPU 时间片和 CPU 使用率信息，示例如下:
```bash
ps -eo bsdtime,cputime,pcpu,cp,c k pcpu

```


![](attachments/Pasted%20image%2020240508185219.png)

字段 bsdtime 的输出相比较 cputime 更加方便转换为正整数的秒数。字段 cp 的单位是千分比，不能超过 999。字段 c 是百分比，不能超过 99。

进程 CPU 时间类字段别名：字段 atime 和字段 time 是字段 cputime 的别名；字段 util 是字段 c 的别名；字段 %cpu 是字段 pcpu 的别名，但是 % 字符在 crontab 中使用并不友好，推荐使用 pcpu。

![](attachments/Pasted%20image%2020240508185504.png)

从这个命令运行的结果可以看出 bsdtime 除以 etimes 的值转换为百分比后，基本和 pcpu 的值相等。这就足以说明 ps 命令的 CPU 利用率字段指标是指从进程开始运行以来进程所耗费的 CPU 时间片占时钟时间的百分比。有时候这个值大于 100%，那是因为进程启用了多线程，很多时候有多核在同时使用 CPU 时间片。

看过 top 命令源码可以知道，top 命令默认是取最近 3 秒钟进程所耗费的 CPU 时间片除以 3 秒钟的百分比值。我们可以设想一种场景，如果一个进程已经运行了 1 年以上，平时都很稳定。但是刚刚就在十几分钟前突然运行大量线程，占用大量 CPU 资源。结果你在你刚刚登陆系统之前 10 秒钟这些运行的线程都结束了。那么你不论是通过 top 命令的 CPU 利用率指标，还是 ps 命令的 CPU 利用率指标都无法发现刚才作怪的这个线程的迹象。

## 进程内存相关字段

进程内存相关字段也 ps 命令字段中非常重要的一类，主要分为如下 9 组，示例如下。
```bash
ps -eo drs,vsz,size,sz,rss,trs,pmem,minflt,majflt k -rss | column -t
```

![](attachments/Pasted%20image%2020240508185613.png)

对以上输出结果的字段逐条说明：

- 字段 vsz（virtual memory size）表示进程所申请的虚拟地址空间的内存大小，单位 KB。在 64 位系统中，每个进程都有 128Tb 大小的堆内存虚拟地址空间的内存空间大小。vsz 值并不反映进程占用的真正物理内存大小。
    
- 字段 rss（resident set size）表示进程真正占用了的物理内存大小，单位 KB。
    
- 字段 pmem 表示进程占用的物理内存大小 (rss) 占本机总物理内存大小百分比。
    
- 字段 trs（text resident set size）表示用于可执行代码的物理内存大小，约等于进程的程序尺寸大小。
    
- 字段 drs（data resident set size）表示可执行代码之外的内存大小，实际基本等于 vsz 减去 trs 的值。
    
- 字段 size 表示如果进程交换到磁盘所需的交换空间大小。
    
- 字段 sz 表示进程在物理页面中的核心镜像的大小。
    
- 字段 minflt 表示此进程中发生的次缺页异常的数量，下面详细介绍。
    
- 字段 majflt 表示此进程中发生的主缺页异常的数量。


进程内存相关字段别名：
```text
字段 m_drs 和字段 dsiz 是字段 drs 的别名，
字段 vsize 是字段 vsz 的别名，
字段 m_size 是字段 sz 的别名，
字段 rssize、字段 rsz 和字段 sgi_rss 是字段 rss 的别名，字段 m_trs、字段 trss 和字段 tsiz 是字段 trs 的别名，字段 %mem 是字段 pmem 的别名，
字段 min_flt 是字段 minflt 的别名，
字段 maj_flt 和字段 pagein 是字段 majflt 的别名。
```

下面通过一个例子来加深对缺页异常的理解。
```bash
# ps -e h -o minflt,rss | awk '{if($1>10000 && $2>10000) {div=sprintf("%.2f", $2/$1); print $1,$2,div}}' | sort -k 3nr | head | column -t
```
![](attachments/Pasted%20image%2020240508190316.png)

可以看出字段 rss 和字段 minflt 的比值趋近于 4。操作系统管理内存的基本单元是 4096 字节大小的页框，当进程访问尚未有物理内存建立页表映射关系的虚拟内存地址值时，会产生一次缺页异常。在缺页异常处理过程中会为虚拟内存页分配一个物理内存页并建立映射。所以每一次缺页异常就会分配 4096（4kb）字节的物理内存，这样 rss 和 minflt 的比值当然就是 4 了。如果分配之后又有释放，后面再次分配，会使这个比值逐步小于 4。如果这个比值过于小，那我们就有充足理由怀疑用户程序代码在内存管理上存在重大问题。


## WCHAN 字段
WCHAN 类字段一共 3 个 nwchan、wchan 和 wname。WCHAN 就是 waiting channel 的意思，进程正在休眠的内核函数的函数符号名称。R 状态进程此字段值为 “-”。

![](attachments/Pasted%20image%2020240508183153.png)

```bash
ps -e -o pid,ppid,wchan:25,lstart,etime,stat,cmd

注：各个字段之间不要有空格。
```

字段 wchan 和 wname 都显示的是内核函数的函数符号名称信息，默认只显示 6 个字节。如果希望显示完整的函数名称，可以通过在字段名称后加冒号再加宽度数值的方式显示更丰富信息，即 wchan:25。

> 注： 可以通过查看 /proc/PID/wchan 以及  stack 来查看具体的睡眠/等待函数。



字段 nwchan 显示的是内核函数符号的指针地址数值信息。一个完整的 64 位的内核函数指针地址是一个 16 位的十六进制值，前 10 位固定为'ffffffff81'，因此 ps 命令的 nwchan 字段只显示出了后 6 位的十六进制值。比如指针地址是 ffffffff8124bb7e，那么 nwchan 显示 24bb7e。如果后 6 位的高位有 0，则省略掉 0 的显示。

# Linux ps命令选项容错机制

Linux ps命令所有的选项和大多数字段都解释过了，现在该说说文章开头那个报错的ps -axu了。

ps命令会提供一种选项容错机制。当用户输入的是一个SYSV风格参数组合后，如果参数解析失败，ps命令会继续尝试把同样的字母组合都转换为BSD风格再尝试进行一次解析。比如ps -aux解析失败后尝试按ps aux解析，ps -x解析失败后尝试按ps x解析。当然了，如果再次按照BSD风格尝试解析仍失败，那ps命令会最终失败报错。

事实上，能有机会被ps命令容错机制纠正的错误选项只有这7个，-S、-X、-h、-k、-v、-r和-x。因为这些字母虽然没有SYSV风格的选项，但是却都有BSD风格的选项。

最后说一下，没有将BSD风格到SYSV风格的容错机制，比如SYSV风格有-F选项，而BSD风格没有F选项。运行命令ps F还是会报错。

# 常见使用方法
## ps -auxf
## ps -lax
## ps -o 常用字段
```bash
ps -eo pid,ppid,psr,tid,state,lstart,etimes,bsdtime,pcpu,vsz,rss,pmem,minflt,majflt,wchan:25,ucmd | column -t

or

ps -o pid,ppid,psr,tid,state,lstart,etimes,bsdtime,pcpu,vsz,rss,pmem,minflt,majflt,wchan:25,ucmd | column -t
```

```bash
查看某个多线程的进程的信息：

ps -L -p 7063 -o pid,ppid,lwp,nlwp,psr,state,lstart,etimes,bsdtime,pcpu,vsz,rss,pmem,minflt,majflt,wchan:25,ucmd


lwp: 线程id；等价于 tid;
nlwp： 线程个数；等价于  thcount;
psr：使用的core；

```

## 展示进程的nice以及 pri
```bash
ps -eLo pid,ppid,psr,tid,nice,pri,start,lstart,%cpu,start_time,etime,cmd | sort -n -k3

或者 

 ps -eLo pid,ppid,psr,tid,nice,pri,start,lstart,%cpu,start_time,etime,cmd  |  sort -n -k3 | column -t
```

# 参考
```bash
# 史上最全 Linux ps 命令详解
https://juejin.cn/post/6844903938144075783
```