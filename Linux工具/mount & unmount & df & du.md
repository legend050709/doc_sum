# df命令
## 说明
df是**disk free**的缩写，在Linux中df 命令的功能是用来检查linux服务器的文件系统的磁盘空间占用情况。
![](attachments/Pasted%20image%2020230524153827.png)
## 查看磁盘使用情况
![](attachments/Pasted%20image%2020230524154016.png)
>linux中df命令的输出清单的第1列是代表文件系统对应的设备文件的路径名（一般是硬盘上的分区）；
>Mounted on列表示文件系统的挂载点。

![](attachments/Pasted%20image%2020230524154142.png)

- 以inode模式来显示磁盘使用情况
![](attachments/Pasted%20image%2020230524154340.png)

- 显示文件系统的类型
![](attachments/Pasted%20image%2020230524154629.png)

## 查看某个目录属于哪个挂载点
很多情况下，查找到了某个磁盘下占用空间最大的文件，不知道这个文件属于哪个磁盘，以及挂载点。通过下面的方式。
![](attachments/Pasted%20image%2020230524154923.png)
![](attachments/Pasted%20image%2020230524154850.png)

# du命令
## 说明
du 命令，全称是 disk usage，用来展示磁盘使用量的统计信息。
du 和 df 算是一对同门师兄弟，du 侧重在文件夹和文件的磁盘占用方面，而 df 则侧重在文件系统级别的磁盘占用方面。
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

# mount
## 介绍
Linux系统中我们经常将一个块设备上的文件系统挂载到某个目录下才能访问这个文件系统下的文件。根文件系统之外的其他文件要想能够被访问，都必须通过“关联”至根文件系统上的某个目录来实现，此关联操作即为“**挂载**”，此目录即为“**挂载点**”,解除此关联关系的过程称之为“**卸载**”

mount命令的功能是用于将文件系统挂载到目录，文件系统指的是被格式化过的硬盘或分区设备，进行挂载操作后，用户便可以在挂载目录中使用硬盘资源了。  比如：将指定分区、光盘、U盘或移动硬盘挂载到Linux系统的目录下。

## 使用
**挂载方法**：mount DECE MOUNT_POINT
mount：通过查看/etc/mtab（通过 strace -T -tt mount 可以看到打开了 /etc/mtab文件， **cat /proc/mounts** ）文件显示当前系统已挂载的所有设备。
**命令使用格式：mount [-fnrsvw] [-t vfstype] [-o options] device dir**

device：指明要挂载的设备；（被挂载的设备可用以下四种之一表示）
(1) **设备文件**：例如/dev/sda5
(2) **卷标**：-L 'LABEL', 例如 -L 'MYDATA'
(3) **UUID**, -U 'UUID'：例如 -U '0c50523c-43f1-45e7-85c0-a126711d406e'
(4) **伪文件系统名称**：proc, sysfs, devtmpfs, configfs

dir：挂载点
    目录需要事先存在；
    建议使用空目录；(若目录非空则挂载后原文件会被隐藏，卸载后方能显示出来)

```c

### 查看当前系统中已有的文件系统信息：  
# mount

### 通过/etc/fstab文件记录的信息，挂载该文件中所有的磁盘分区
# mount -a


### 以只读方式挂载/dev/sda5磁盘分区到/mnt/www目录
# mount -t ext4 -o ro /dev/sda5 /mnt/www

```



## 开机自动挂载
linux开机之后自动加载挂载的分区，这块，涉及到的文件是/etc/fstab文件  
关于这个文件的描述说明如下:  
注意：
1.  根目录必须优先于其他挂载点
2.  挂载点必须为已经存在的目录
3.  卸载时必须保证当前磁盘没有发生读写操作

/etc/fstab里面每列大概意思为：
第一列为设备号或该设备的卷标，即需要挂载的文件系统或存储设备；  
第二列为挂载点  
第三列为文件系统或分区的类型  
第四列为文件系统参数，即挂载选项，详细参考man mount.命令，defaults就没有问题，除非你有特殊需求。
第五列为dump选项，设置是否让备份程序dump备份文件系统。0：不备份，1：备份，2：备份(但比1重要性小)。设置了该参数后，Linux中使用dump命令备份系统的时候就可以备份相应设置的挂载点了。  
第六列为是否在系统启动的时候，用fsck检验分区,告诉fsck程序以什么顺序检查文件系统。因为有些挂载点是不需要检验的，比如：虚拟内存swap、/proc等。0：不检验，1：要检验，2要检验(但比1晚检验)，一般根目录设置为1，其他设置为2就可以了。

![](attachments/Pasted%20image%2020230524165520.png)

> 注：通过将信息写入etc/fstab中进行自动化挂载云硬盘操作时，建议不要使用盘符以及分区id，建议使用文件系统的UUID，因为当云硬盘涉及到挂载和卸载操作时盘符会产生改变或者漂移。


修改完/etc/fstab文件后，运行  mount -a 命令验证一下配置是否正确。如果配置不正确可能会导致系统无法正常启动。


# umount
**umount命令** 用于卸载已经加载的文件系统。利用设备名或挂载点都能umount文件系统，不过最好还是通过挂载点卸载，以免使用绑定挂载（一个设备，多个挂载点）时产生混乱。

- 语法
umount [选项] [设备|挂载目录]
```c
-a：卸除/etc/mtab中记录的所有文件系统；
-h：显示帮助；
-n：卸除时不要将信息存入/etc/mtab文件中；
-r：若无法成功卸除，则尝试以只读的方式重新挂入文件系统；
-t<文件系统类型>：仅卸除选项中所指定的文件系统；
-v：执行时显示详细的信息；
-V：显示版本信息。

```

- 使用
```c

### 卸载磁盘分区/dev/sda5文件系统
[root@rhel ～]# umount /dev/sda5

### 卸载/mnt/www目录所在的磁盘分区文件系统
[root@rhel ～]# umount /mnt/www


```
注：如果设备正忙，卸载即告失败。
卸载失败的常见原因是，某个打开的shell当前目录为挂载点里的某个目录。

有时，导致设备忙的原因并不好找。碰到这种情况时，可以用lsof列出已打开文件，然后搜索列表查找待卸载的挂载点：

```shell
lsof | grep mymount         查找mymount分区里打开的文件
bash   9341  francois  cwd   DIR   8,1   1024    2 /mnt/mymount
```
对付系统文件正忙的另一种方法是执行延迟卸载：
```shell
umount -l /mnt/mymount/     执行延迟卸载
```


# 参考
```c
http://c.biancheng.net/linux/du.html

```