# df命令
## 说明
df是**disk free**的缩写，在Linux中df 命令的功能是用来检查linux服务器的文件系统的磁盘空间占用情况。
![](attachments/Pasted%20image%2020230524153827.png)

## 查看磁盘使用情况
![](attachments/Pasted%20image%2020230524154016.png)
>linux中df命令的输出清单的第1列是代表文件系统对应的设备文件的路径名（一般是硬盘上的分区）；
>Mounted on列表示文件系统的挂载点。

![](attachments/Pasted%20image%2020230524154142.png)

### inode使用情况
![](attachments/Pasted%20image%2020230919104941.png)

### 文件系统类型
![](attachments/Pasted%20image%2020230524154629.png)

## 查看某个目录属于哪个挂载点
很多情况下，查找到了某个磁盘下占用空间最大的文件，不知道这个文件属于哪个磁盘，以及挂载点。通过下面的方式。
![](attachments/Pasted%20image%2020230524154923.png)
![](attachments/Pasted%20image%2020230524154850.png)

## 其他
### inode说明
inode译成中文就是**索引节点（index node）**，每个存储设备（例如硬盘）或存储设备的分区被格式化为文件系统后，应该有两部份，一部份是inode，另一部份是Block。

- Block
Block是用来存储数据用的。文件储存在硬盘上，硬盘的最小存储单位叫做"扇区"（Sector）。每个扇区储存 512 字节（相当于0.5KB）。
操作系统读取硬盘的时候，不会一个个扇区地读取，这样效率太低，而是一次性连续读取多个扇区，即一次性读取一个"块"（block）。
这种由多个扇区组成的"块"，是文件存取的最小单位。“块"的大小，最常见的是 4KB，即连续八个 sector 组成一个 block。文件数据都储存在"块"中。

- Inode
Inode 就是用来存储这些数据的信息，这些信息包括文件大小、属主、归属的用户组、读写权限等。
inode为每个文件进行信息索引，所以就有了inode的数值。操作系统根据指令，能通过inode值最快的找到相对应的文件。

inode 包含文件的元信息，具体来说有以下内容：
```c
* 文件名
* 文件的字节数   
* 文件拥有者的 User ID
* 文件的 Group ID
* 文件的读、写、执行权限

* 文件的时间戳，共有三个：
	* ctime 指 inode 上一次变动的时间，
	* mtime 指文件内容上一次变动的时间，
	* atime 指文件上一次打开的时间。

* 链接数，即有多少文件名指向这个 inode
* 文件数据 block 的位置
```

可以用 stat 命令，查看某个文件或者文件夹的 inode 信息，第一行则包含文件名，具体如下图所示：

![](attachments/Pasted%20image%2020230919111357.png)

inode 也会消耗硬盘空间，每个 inode 节点的大小，一般是 128 字节或 256 字节。
inode 节点的总数，在硬盘格式化时就给定，一般是每 1KB 或每 2KB 就设置一个 inode。
假定在一块 1GB 的硬盘中，每个 inode 节点的大小为 128 字节，每 1KB 就设置一个 inode，那么 inode table 的大小就会达到 128MB，占整块硬盘的 1/8 空间（12.5%）。

- inode 号码
每个 inode 都有一个号码，操作系统用 inode 号码来识别不同的文件。Unix/linux 系统内部不使用文件名，而使用 inode 号码来识别文件或者文件夹。

对于系统来说，文件名只是 inode 号码便于识别的别称或者绰号。
表面上，用户通过文件名，打开文件。实际上，系统内部这个过程分成三步：
首先，系统找到这个文件名对应的inode 号码；
其次，通过 inode 号码，获取 inode 信息；
最后，根据 inode 信息，找到文件数据所在的 block，读出数据。



## 问题
### inode不足的查看以及解决
#### 不足的查看
使用``"df -h"``命令发现磁盘使用率没有占满，但是无法写入文件，提示``"no space left on device"``!。这个应该就是inode不足导致的。

通过 `df -iT` 来查看每个磁盘下的inode数量以及使用情况。
#### 解决方法一:删除多余文件
(0) df查看具体的哪个磁盘占用inode多
```bash
df -iT 先查看磁盘的inode

查找到磁盘挂载目录之后，查看该目录下的各个子目录的文件的数量。
比如：查看 根目录下的各个子目录的文件的数量。
for i in /*; do echo $i; find $i |wc -l; done

查看某个目录所在的磁盘(比如：查看/etc 所在的磁盘)：
df /etc
```

(1) **删除大小为0的文件**
查找文件大小为 0 的空文件，可以使用如下命令查找：
```c
find PATH -name "*" -type f -size 0c
比如：find /home -type f -size 0 

查找并删除大小为0的文件
find /home -type f -size 0 -exec rm {} \;

查找大小在某个范围内的文件使用-size参数，-size +n表示大于n单位的范围，-size –n表示小于n单位的范围。
find . -type f -mtime -1 -size +100k -size-400k
（查找大于100k且小于400k的文件）

注意：
使用 `-size` 参数时，不要用 `-size 1k`，这个表示占用空间为 1KB，而不是文件大小为 1KB，应该使用 `-size 1024c` 才表示文件大小为 1KB。

```
![](attachments/Pasted%20image%2020230919105708.png)

（2）**删除无用的临时文件 或者 很久之前的待删除的文件**
```c
查找某个目录下一个月或两个月之前的文件，然后删除
# find . -type f -mtime +30 |wc -l
# find . -type f -mtime +60 |wc -l
# find . -type f -mtime +30 -exec rm -f {} \;
# find . -type f -mtime +60 -exec rm -f {} \;


除无用的临时文件，释放inode。比如/tmp下有很多临时文件
# ls -lt /tmp | wc -l
# find /tmp -type f -exec rm {} \;
```

```c
大量小文件分布有两种可能：
a）一是只有一个或少量目录下存在大量小文件，这种情况可以使用如下命令来找出这个异常目录：
# find / -type d -size +10M 
即找出大小大于10M的目录（目录大小越大，表示目录下的文件越多）。
   
b）大量的小文件分布在大量的目录下，这时候上面的命令可能找不出异常的目录，需要以下命令：
# cd /
# find */ ! -type l | cut -d / -f 1 | uniq -c
此命令作用是找出目录下文件总数，可能需要执行多次，直到找出具体的目录。比如上面的命令找出了/data目录下存在大量的小文件，
但/data/目录还有很多目录，这时候我们还需要继续执行：
# cd /data
# find */ ! -type l | cut -d / -f 1 | uniq -c
直到找出具体的目录。
```

（3）查看已经删除没有释放的文件

当磁盘空间被占满之后，删除了某些日志却发现空间并没有释放，或者释放的空间没有删掉的日志大，原因是删掉的文件正好有服务在调用，而此时删掉的文件是不会释放空间的。
查看已经删除没有释放的文件
```bash
lsof | grep delete
```

确认了是哪个服务在占用之后重启相应服务就可以解决了。

（4）**查找文件个数多的文件夹**
这里为什么要循环/var/*？这是根据个人经验吧！
如下所示，查看/var目录下的各个子目录的文件数量。
```text
for i in /var/*; do echo $i; find $i |wc -l; done 

注： find xxx ： 查找以 xxx开头的文件or 文件夹。
# find /var/tmp
/var/tmp
/var/tmp/net_mlx5_85
/var/tmp/net_mlx5_103
/var/tmp/systemd-private-7fa2e63fc47b4ff59c92752702536558-kernel-server.service-Z4REdp
/var/tmp/systemd-private-7fa2e63fc47b4ff59c92752702536558-kernel-server.service-Z4REdp/tmp
/var/tmp/net_mlx5_123
/var/tmp/net_mlx5_136
/var/tmp/net_mlx5_55
/var/tmp/systemd-private-7fa2e63fc47b4ff59c92752702536558-chronyd.service-ZPZDBD
/var/tmp/systemd-private-7fa2e63fc47b4ff59c92752702536558-chronyd.service-ZPZDBD/tmp
/var/tmp/host_0
```
![](attachments/Pasted%20image%2020231018103713.png)

一般情况都是crond导致的。
问题成因：crond在执行脚本时会将脚本输出信息以邮件的形式发送给系统用户，所以必然要调用sendmail，而sendmail又会调用postdrop发送邮件，但是如果系统的postfix服务没有正常运行，那么邮件就会发送不成功，导致持续写入日志到日志文件，造成sendmail、postdrop、crond进程就无法正常退出，形成大量的僵尸进程


**处理**：
```bash
(1) 杀进程
ps -ef | egrep "sendmail|postdrop" | grep -v grep |xargs kill 
或者
killall postdrop

​ （2）删除占用inode的文件
find /var/spool/postfix/maildrop/ -type f |xargs rm -rf

（3）修改crond
为防以后postfix挂了再出现类似问题，可以进行如下配置，将crond的邮件通知关闭：
将/etc/crontab和/etc/cron.d/0hourly里的MAILTO=root修改为MAILTO=""

```

（5）**查找某目录下inode最多的子目录**
```bash
find /data -xdev -printf '%h\n' | sort | uniq -c | sort -k 1 -n
## /data 目录根据需求进行替换

%h: 表示子目录名称；
\n: 换行；
```

![](attachments/Pasted%20image%2020240409170450.png)

注：该命令和上诉的 `for i in /var/*; do echo $i; find $i |wc -l; done ` 类似。

#### 增大inode总数
如果不允许清理磁盘中的文件，或者清理后inode使用率仍然较高，则需要通过如下步骤，增加inode节点数量。

注：**inode的调整需要重新格式化磁盘，请确保数据已经得到有效备份后，再进行以下操作。**

```bash
(1) 卸载系统文件
umount /home

(2) 重新建立文件系统，指定inode节点数
mkfs.ext3 /dev/xvdb -N 1638400


(3) 修改fstab文件


(4) 查看修改后的inode节点数
dumpe2fs -h /dev/xvdb | grep node
or
df -iT
```

# du命令
## 说明
du 命令，全称是 disk usage，用来展示磁盘使用量的统计信息。
**du 和 df 算是一对同门师兄弟，du 侧重在文件夹和文件的磁盘占用方面，而 df 则侧重在文件系统级别的磁盘占用方面。**
![](attachments/Pasted%20image%2020230524155341.png)



>`-s`选项，是 --summarize 的缩写形式，其作用是对 du 的每一个给定参数计算其磁盘使用量
>`-c`选项，是 --total 的缩写形式，它表示的是针对输出的各个对象来计算其磁盘使用量的总和。
> `-a`选项让 du 输出包括文件夹和文件在内的完整统计信息。默认情况下，du 命令只会关心文件夹，输出的都是文件夹的空间使用量，而不会关注单个文件。
> 注：-s和 -a不可以同时使用。


## 查看目录的磁盘占用大小
ls 命令一般只能查看文件的占用大小，不方便查看目录的占用大小。通过du可以查看一个目录下的子目录以及文件的占用磁盘的大小。
![](attachments/Pasted%20image%2020230524160142.png)
![](attachments/Pasted%20image%2020230524160622.png)

```c
du -sh * | sort -rh
du -sh DIR: 查看目录的大小
du -sh DIR/* :查看目录下子目录以及文件的大小
du -ach DIR: 查看目录及其子目录的大小
```
## 用 --max-depth /-d 选项控制深度

文件夹是可以嵌套的，有的时候，我们只想展示第一级或第二级子文件夹的信息，而不希望 du 统计的层次太深，那么我们可以用 --max-depth 选项来进行控制。
我们绘制了一个示意图，movies 文件中存储了中美两国 2016 年和 2017 年的一些电影大片，而且是按照类型来分的，包括探险片、爱情片、动作片。从图 1 中可以看出，movies 文件夹中共有 3 级子文件夹。
![](attachments/Pasted%20image%2020230524161356.png)
从图 1 中，我们可以很清晰地看到，当 --max-depth 是 0、1、2 时，du 分别对应哪一目录层级。  
我们通过下面的例子，来实际看一下 --max-depth 的效果。
```c
#我们模拟了和图8完全一致的目录结构
[roc@roclinux movies]$ tree
.
|-- China
|   |-- 2016
|   |   |-- Action
|   |   |   `-- yip_man.avi
|   |   |-- Adventure
|   |   |   `-- hero.avi
|   |   `-- Romance
|   |       `-- rose.avi
|   `-- 2017
|       |-- Action
|       |   `-- drunken_master.avi
|       |-- Adventure
|       |   `-- treatment.avi
|       `-- Romance
|           `-- banquet.avi
`-- USA
    |-- 2016
    |   |-- Action
    |   |   `-- gian.avi
    |   |-- Adventure
    |   |   `-- patton.avi
    |   `-- Romance
    |       `-- god_father.avi
    `-- 2017
        |-- Action
        |   `-- african_queen.avi
        |-- Adventure
        |   `-- star_wars.avi
        `-- Romance
            `-- citizen_kane.avi
 
 
#当--max-depth设定为0时, 只显示当前文件夹总大小
#可见, --max-depth=0的作用, 相当于-s
[roc@roclinux movies]$ du --max-depth=0 -h .
5.2G
 
#当--max-depth设定为1时, 则增加显示了第一级的文件夹大小
[roc@roclinux movies]$ du --max-depth=1 -h .
2.7G     ./China
2.5G     ./USA
5.2G     .
 
#当--max-depth设定为2时, 则会继续增加显示下一级子文件夹
[roc@roclinux movies]$ du --max-depth=2 -h .
1.4G     ./China/2017
1.3G     ./China/2016
2.7G     ./China
1.2G     ./USA/2017
1.3G     ./USA/2016
2.5G     ./USA
5.2G     .
```

## 磁盘占满问题
磁盘被占满，是 Linux 工程师经常遇到的问题，如果能够熟练使用 du 和 sort 形成组合拳，那么找到元凶并非难事。

```c
#只想看当前文件夹下第一级的大小排序
[roc@roclinux ruanjian]$ du -sh *|sort -nr
41M     soft
6.8M    wordpress-4.4.1.tar.gz
3.4M    curl-7.34.0.tar.gz
 
#想看当前文件夹和其子文件夹下的大排序
[roc@roclinux ruanjian]$ du -ah .|sort -hr
51M     .
41M     ./soft
40M     ./soft/go1.1.2.Linux-amd64.tar.gz
6.8M    ./wordpress-4.4.1.tar.gz
3.4M    ./curl-7.34.0.tar.gz
980K    ./soft/redis-2.6.16.tar.gz
```
  
这里补充一个 sort 的知识点，那就是`-h`选项和`-n`选项的区别：
-   `-n`选项，按数值进行比较，只会傻傻地比较数字，它会认为 98 K大于 2G。
-   `-h`选项，会更加聪明，先优先比较单位（G>M>K），然后再对数值进行比较。

![](attachments/Pasted%20image%2020230524161717.png)

## du的单位
首先，要明确的是，du 的默认单位是 KB，也就是 1024bytes。我们来看一个例子。
```c
#这里的单位就是KB, 按1KB=1024bytes计算
[roc@roclinux ruanjian]$ du curl-7.34.0.tar.gz
3448    curl-7.34.0.tar.gz
 
#而这里可以很清楚地看到是MB
[roc@roclinux ruanjian]$ du -h curl-7.34.0.tar.gz
3.4M    curl-7.34.0.tar.gz
```
但 du 的单位，其实并没有这么简单，有不少只幕后黑手都可能会控制它，我们来一一曝光它们：

1.  如果你通过 --block-size 选项设置了块大小，那么，这就会成为你 du 输出信息的单位。
2.  假如上一条没满足，且你设置了环境变量 DU_BLOCK_SIZE，则这会成为你 du 输出信息的单位。
3.  假如上两条都没满足，且你设置了环境变量 BLOCK_SIZE，则这会成为你 du 输出信息的单位。
4.  假如前三条都没满足，且你设置了环境变量 BLOCKSIZE，则这会成为你 du 输出信息的单位。
5.  假如前四条都没满足，且你开启了环境变量 POSIXLY_CORRECT，则 du 输出信息的单位会是 512 bytes。
6.  假如前面的五条都没满足，那么 du 的输出信息的单位就是 1024 bytes，也就是 KB。


## 为什么 du 和 ls 输出的值不同
如果我告诉你说 du 和 ls 针对同一个文件，展示的大小是不一样的，你会不会很惊讶呢？

```c
#有一个文件, 里面只输入了a、b两个英文字母
[roc@roclinux ruanjian]$ cat myword
ab
 
#用下面的方法, 我们可以把文件中的控制字符也展示出来, 发现除了a、b外还包括了一个结尾符
[roc@roclinux ruanjian]$ sed -n l myword
ab$
 
#用ls来查看大小, 发现展示的是3字节
[roc@roclinux ruanjian]$ ls -l myword
-rw-rw-r-- 1 roc roc 3 2月  18 15:53 myword
 
#用du来查看大小, 竟然展示的是4KB字节
[roc@roclinux ruanjian]$ du myword
4       myword
# du 的默认单位是 KB，也就是 1024bytes。
```

其实，du 和 ls 在展示文件大小时，是存在着本质区别的：
-   **du 展示的是磁盘空间占用量**。
-   **ls 展示的是文件内容的大小**。
举一个形象的例子。中秋节时，中国人走亲访友时都会购买月饼礼盒，月饼的体积可以认为是文件内容大小，而加上包装礼盒的总体积可以认为是磁盘空间使用量。
 Linux 文件系统进驻磁盘之初，就会将磁盘按照固定数据块（block）大小进行分隔切块，通常情况下每一个固定数据块大小会被设定为 4096bytes，也就是 4KB。

与此同时，大部分文件系统规定：
1.  一个数据块中最多存放一个文件的内容，当没存满时，剩余的空间不得被其他文件使用。
2.  当一个文件的内容较大时，则可以存储到多个数据块中。



# 参考
```c
http://c.biancheng.net/linux/du.html

inode的讲解不错：
https://cloud.tencent.com/developer/article/1768547
```