# mount
## 介绍
Linux系统中我们经常将一个块设备上的文件系统挂载到某个目录下，才能访问这个文件系统下的文件。
根文件系统之外的其他文件要想能够被访问，都必须通过“关联”至根文件系统上的某个目录来实现，此关联操作即为“**挂载**”，此目录即为“**挂载点**”,解除此关联关系的过程称之为“**卸载**”

mount命令的功能是用于将文件系统挂载到目录，文件系统指的是被格式化过的硬盘或分区设备，进行挂载操作后，用户便可以在挂载目录中使用硬盘资源了。  比如：将指定分区、光盘、U盘或移动硬盘挂载到Linux系统的目录下。

## 使用
**挂载方法**：mount DECE MOUNT_POINT
mount：通过查看/etc/mtab（通过 strace -T -tt mount 可以看到打开了 /etc/mtab文件， **cat /proc/mounts** ）文件显示当前系统已挂载的所有设备。
**命令使用格式：mount [-fnrsvw] [-t vfstype] [-o options] device dir**

- device：指明要挂载的设备；（被挂载的设备可用以下四种之一表示）
(1) **设备文件**：例如/dev/sda5
(2) **卷标**：-L 'LABEL', 例如 -L 'MYDATA'
(3) **UUID**, -U 'UUID'：例如 -U '0c50523c-43f1-45e7-85c0-a126711d406e'
(4) **伪文件系统名称**：proc, sysfs, devtmpfs, configfs

- dir：挂载点
    注：**目录需要事先存在**；
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
### 注意事项
关于这个文件的描述说明如下:  
注意：
1.  根目录必须优先于其他挂载点
2.  挂载点必须为已经存在的目录
3.  卸载时必须保证当前磁盘没有发生读写操作

### 文件说明
```c
磁盘的UUID查看
# blkid
/dev/sda1: UUID="3c8d3ccd-df8e-4229-9c5e-89675b4dece9" TYPE="ext4"
/dev/sda2: UUID="eaac01aa-14cb-4d30-b397-abae98b3879d" TYPE="ext4"
/dev/sda3: UUID="c68922fc-1e83-42e4-93d2-14c00d0294cd" TYPE="ext4"
/dev/sdb: UUID="bdd589ee-a90a-4c41-beb2-e87932b4486a" TYPE="xfs"


fstab文件查看
# cat /etc/fstab
#
# /etc/fstab
# Created by anaconda on Thu Jun  7 05:44:31 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=3c8d3ccd-df8e-4229-9c5e-89675b4dece9 /boot ext4 defaults,noatime,nofail 0 0
UUID=eaac01aa-14cb-4d30-b397-abae98b3879d / ext4 defaults,noatime,nofail 0 0
UUID=c68922fc-1e83-42e4-93d2-14c00d0294cd /home ext4 defaults,noatime,nofail 0 0
UUID=bdd589ee-a90a-4c41-beb2-e87932b4486a /media/disk1 xfs defaults,noatime,nofail 0 0
nodev /mnt/huge_1GB hugetlbfs pagesize=1GB 0 0
```
注：通过将信息写入etc/fstab中进行自动化挂载云硬盘操作时，建议不要使用盘符以及分区id，建议使用文件系统的UUID，因为当云硬盘涉及到挂载和卸载操作时盘符会产生改变或者漂移。


/etc/fstab里面每列大概意思为：
第一列为设备号或该设备的卷标，即需要挂载的文件系统或存储设备；  
第二列为挂载点  
第三列为文件系统或分区的类型  
第四列为文件系统参数，即挂载选项，详细参考man mount.命令，defaults就没有问题，除非你有特殊需求。
第五列为dump选项，设置是否让备份程序dump备份文件系统。0：不备份，1：备份，2：备份(但比1重要性小)。设置了该参数后，Linux中使用dump命令备份系统的时候就可以备份相应设置的挂载点了。  
第六列为是否在系统启动的时候，用fsck检验分区,告诉fsck程序以什么顺序检查文件系统。因为有些挂载点是不需要检验的，比如：虚拟内存swap、/proc等。0：不检验，1：要检验，2要检验(但比1晚检验)，一般根目录设置为1，其他设置为2就可以了。


修改完/etc/fstab文件后，运行  mount -a 命令验证一下配置是否正确。如果配置不正确可能会导致系统无法正常启动。

### 效果查看
```c
df查看：

# df -ahT
Filesystem     Type         Size  Used Avail Use% Mounted on
sysfs          sysfs           0     0     0    - /sys
proc           proc            0     0     0    - /proc
devtmpfs       devtmpfs      76G     0   76G   0% /dev
securityfs     securityfs      0     0     0    - /sys/kernel/security
tmpfs          tmpfs        126G  292K  126G   1% /dev/shm
devpts         devpts          0     0     0    - /dev/pts
tmpfs          tmpfs        126G  4.0G  122G   4% /run
tmpfs          tmpfs        126G     0  126G   0% /sys/fs/cgroup
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/systemd
pstore         pstore          0     0     0    - /sys/fs/pstore
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/net_cls,net_prio
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/hugetlb
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/cpu,cpuacct
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/perf_event
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/blkio
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/pids
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/cpuset
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/rdma
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/freezer
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/memory
cgroup         cgroup          0     0     0    - /sys/fs/cgroup/devices
configfs       configfs        0     0     0    - /sys/kernel/config
/dev/sda2      ext4          93G   14G   75G  16% /
systemd-1      -               -     -     -    - /proc/sys/fs/binfmt_misc
mqueue         mqueue          0     0     0    - /dev/mqueue
debugfs        debugfs         0     0     0    - /sys/kernel/debug
hugetlbfs      hugetlbfs       0     0     0    - /dev/hugepages
nodev          hugetlbfs       0     0     0    - /mnt/huge_1GB
/dev/sda1      ext4         453M  275M  151M  65% /boot
/dev/sda3      ext4         126G   13G  107G  11% /home
/dev/sdb       xfs          1.9T   25G  1.8T   2% /media/disk1
tracefs        tracefs         0     0     0    - /sys/kernel/debug/tracing
binfmt_misc    binfmt_misc     0     0     0    - /proc/sys/fs/binfmt_misc
tmpfs          tmpfs         26G     0   26G   0% /run/user/12949
tmpfs          tmpfs         26G     0   26G   0% /run/user/1101
tmpfs          tmpfs         26G     0   26G   0% /run/user/0



mount查看：
# mount
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
devtmpfs on /dev type devtmpfs (rw,nosuid,size=79190696k,nr_inodes=19797674,mode=755)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,mode=755)
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
pstore on /sys/fs/pstore type pstore (rw,nosuid,nodev,noexec,relatime)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
configfs on /sys/kernel/config type configfs (rw,relatime)
/dev/sda2 on / type ext4 (rw,noatime)
systemd-1 on /proc/sys/fs/binfmt_misc type autofs (rw,relatime,fd=22,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=56375)
mqueue on /dev/mqueue type mqueue (rw,relatime)
debugfs on /sys/kernel/debug type debugfs (rw,relatime)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,pagesize=1024M)
nodev on /mnt/huge_1GB type hugetlbfs (rw,relatime,pagesize=1024M)
/dev/sda1 on /boot type ext4 (rw,noatime,stripe=4)
/dev/sda3 on /home type ext4 (rw,noatime)
/dev/sdb on /media/disk1 type xfs (rw,noatime,attr2,inode64,noquota)
tracefs on /sys/kernel/debug/tracing type tracefs (rw,relatime)
binfmt_misc on /proc/sys/fs/binfmt_misc type binfmt_misc (rw,relatime)
tmpfs on /run/user/12949 type tmpfs (rw,nosuid,nodev,relatime,size=26326572k,mode=700,uid=12949,gid=12949)
tmpfs on /run/user/1101 type tmpfs (rw,nosuid,nodev,relatime,size=26326572k,mode=700,uid=1101,gid=1101)
tmpfs on /run/user/0 type tmpfs (rw,nosuid,nodev,relatime,size=26326572k,mode=700)
```


### nodev选项
```c
nodev: Do not interpret character or block special devices on the file system
```
添加了nodev选项，那么系统不会把该文件系统里面的 block/character 文件当作是 block/character 文件来处理。
即：禁止文件系统中执行任何设备文件，这可以提高系统的安全性，防止在文件系统中恶意执行代码或访问设备。

举个例子，如果您在使用 mount 命令挂载一个 U 盘或 CD-ROM，如果没有指定 nodev 选项，则设备文件将可执行，如果设备文件被篡改，攻击者就有可能在您的系统中执行恶意代码，导致系统被攻击或受到破坏。

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
