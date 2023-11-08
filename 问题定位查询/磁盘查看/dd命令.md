```table-of-contents
```
# 介绍
`dd` 命令在 Linux 系统中用来拷贝和转换文件。
dd在实际运用场景中可以用来**备份数据，测试磁盘速度，制作U盘启动盘、文本内容的大小写转换、擦除硬盘数据** 等操作。
# 使用方法
![](attachments/Pasted%20image%2020231102110823.png)
**参数说明:**
```c
- if=文件名：输入文件名，缺省为标准输入。即指定源文件。
- of=文件名：输出文件名，缺省为标准输出。即指定目的文件。
- ibs=bytes：一次读入bytes个字节，即指定一个块大小为bytes个字节。  
    obs=bytes：一次输出bytes个字节，即指定一个块大小为bytes个字节。  
    bs=bytes：同时设置读入/输出的块大小为bytes个字节。
- cbs=bytes：一次转换bytes个字节，即指定转换缓冲区大小。
- skip=blocks：从输入文件开头跳过blocks个块后再开始复制。
- seek=blocks：从输出文件开头跳过blocks个块后再开始复制。
- count=blocks：仅拷贝blocks个块，块大小等于ibs指定的字节数。
- conv=<关键字>，关键字可以有以下11种：

- conversion：用指定的参数转换文件。
- ascii：转换ebcdic为ascii
- ebcdic：转换ascii为ebcdic
- ibm：转换ascii为alternate ebcdic
- block：把每一行转换为长度为cbs，不足部分用空格填充
- unblock：使每一行的长度都为cbs，不足部分用空格填充
- lcase：把大写字符转换为小写字符
- ucase：把小写字符转换为大写字符
- swab：交换输入的每对字节
- noerror：出错时不停止
- notrunc：不截断输出文件
- sync：将每个输入块填充到ibs个字节，不足部分用空（NUL）字符补齐。
- --help：显示帮助信息
```
# 使用场景
## 产生固定大小的文件
可以指定bs，以及count，产生指定大小的文件。
比如，产生一个大文件，通过curl上传，测试上传的速率。那么就可以通过dd来产生一个指定大小的大文件。

```c
dd if=/dev/zero of=xxx bs=1M count=xxxx
```

随机生成100百万个1K的小文件
```text
seq 1000000 | xargs -i dd if=/dev/zero of={}.dat bs=1024 count=1  #随机生成指定大小
```

## 测试磁盘的写速率

查看文件所在的磁盘
```c
# df /data/logs/
Filesystem      1K-blocks      Used  Available Use% Mounted on
/dev/sdb       1952560720 757117016 1195443704  39% /media/disk1
```

测试磁盘的写速率
```c
date +%T.%N; sync; dd if=/dev/zero of=/data/logs/tempfile3 bs=1M count=1024; sync; date +%T.%N
```

说明：因为/dev//zero是一个伪设备，它只产生空字符流，对它不会产生IO。
所以，IO都会集中在of文件中，of文件只用于写，所以这个命令相当于测试磁盘的写能力。命令结尾添加oflag=direct将跳过内存缓存，添加oflag=sync将跳过hdd缓存。

## 测试磁盘的读速率
```c
date +%T.%N; dd if=/data/logs/tempfile3 of=/dev/null bs=1M count=1024 iflag=direct;  date +%T.%N
```
说明：/dev/null是伪设备，相当于黑洞，of到该设备不会产生IO。所以，这个命令的IO只发生在/data/logs/tempfile3文件所在的磁盘上，也相当于测试磁盘的读能力。

执行读写测试时，建议加上oflag=direct/iflag=direct参数，因为没有这个参数，dd 命令有时会显示从内存中传输数据的结果速度，而不是从硬盘，无法测试出真实速度。

清除 cache，准确的测试真实的读速度：
```c
sysctl -w vm.drop_caches=3
date +%T.%N; dd if=/data/logs/tempfile3 of=/dev/null bs=1M count=1024 iflag=direct;  date +%T.%N
```

## 创建随机数文件
```
dd if=/dev/urandom of=/path/to/random.file bs=1M count=10
```

## 文件内容拷贝
dd命令和 类似功能的 `cp` 命令还是存在一些区别。比如：

- `cp` 命令只能操作文件或目录，但 `dd` 命令可以直接操作存储设备。
- `cp` 同时可以拷贝多个文件或文件夹，但 `dd` 命令一次只能操作一个文件对象。
- `cp` 命令以字节为单位读取文件，`dd` 命令以「块」为单位读取文件。


将 /tmp/test 中内容拷贝到 /var/test中。
```c
dd if=/tmp/test of=/var/test bs=64k
```

完整备份磁盘 _/dev/sda_ 到 _/dev/sdb_ 上：
```shell
dd if=/dev/hda of=/dev/hdb
```

**擦除硬盘数据**
```c
dd if=/dev/zero of=/dev/sda bs=4096
```

## 同时测试磁盘的读写速率
可以通过文件内容拷贝，来同时测试读速率以及写速率。
```c
# df /tmp
Filesystem     1K-blocks     Used Available Use% Mounted on
/dev/sda2       49371796 30891608  15949168  66% /

# date +%T.%N; dd if=/dev/sda2 of=/tmp/test_dd.out bs=1M count=1000; sync; date +%T.%N;
10:51:35.400075276
1000+0 records in
1000+0 records out
1048576000 bytes (1.0 GB) copied, 7.51776 s, 139 MB/s
10:51:43.222289350
```
在这个命令下，一个是物理分区，一个是实际的文件，对它们的读写都会产生IO（对/dev/sda2是读，对/tmp/test_dd.out是写），假设它们都在一个磁盘中，这个命令就相当于测试磁盘的同时读写能力。

## 格式转换
- 大小写转换
```text
[root@knode1 letter]# echo "Hello World" > hello.txt
[root@knode1 letter]# cat hello.txt
Hello World
[root@knode1 letter]# dd if=hello.txt of=hello1.txt conv=ucase
记录了0+1 的读入
记录了0+1 的写出
12字节(12 B)已复制，0.000166877 秒，71.9 kB/秒
[root@knode1 letter]# cat hello1.txt
HELLO WORLD
```

## 制作U盘启动盘
dd命令还有个很实用的功能就是制作U盘启动盘，将U盘插入linux主机，然后使用命令 fdisk -l 查看U盘挂载的设备路径（比如是/dev/sdb）。

然后使用 dd 命令制作U盘启动盘：
```text
 dd if=/***.iso of=/dev/xxx 
# if后接镜像文件路径
# of后接写入U盘的路径

#将上传的镜像文件上传在linux目录后，执行如下命令制作U盘启动盘
dd if=/root/Kylin-Desktop-V10-Release-2107-arm64.iso of=/dev/sdb
```
写入完成后就可以拿着U盘设置U盘启动，来安装操作系统了


## 小结
if=/dev/zero 不产生读的 IO，可以用来测试纯写速度；
of=/dev/null 不产生写 IO，可以用来测试纯读速度；

# 参考
```c

```