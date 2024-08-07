```table-of-contents
```
# 概述
# 场景
# 特性
## `sed` 不可能同时打开一个文件，一边读一边写

因为边读边写的话，文件会被 truncate。
比如执行一下 `cat foo.txt > foo.txt`，文件会被直接清空；

## `sed` 也不可能将文件读入内存，处理，然后写入原文件

因为 `sed` 最基本的设计就是一个**行处理器**，要一行一行 streaming 处理，读入一部分，处理，然后写入一部分，用很少的内存就够了；

所以要实现文件原地替换的话，就需要有一个临时文件，sed 先把结果写入到这个文件，最后将文件 rename 到原来的地方；


# 使用方法
# 操作
## 提取
### 从指定的字符串中提取多个内容
将目标用()括起来，然后输出的时候使用\n来代替进行输出。

```c
# ss -tman | grep skmem
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o136,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb787968,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb4194304,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb332800,f4096,w0,o136,bl0)
	 skmem:(r0,rb369280,t0,tb374272,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb46080,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t1408,tb87040,f3968,w4224,o0,bl0)
	 skmem:(r0,rb369280,t0,tb678912,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb2491109,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb369280,t0,tb217600,f0,w0,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o136,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f4096,w0,o0,bl0)
	 skmem:(r0,rb366080,t0,tb46080,f4096,w0,o136,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb978736,t0,tb46080,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f4096,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
```
比如想要提取每一个数字。
则如下所示：
```c
# ss -tman | grep skmem | sed -r "s/skmem:\(r([0-9]+),rb([0-9]+),t([0-9]+),tb([0-9]+),f([0-9]+),w([0-9]+),o([0-9]+),bl([0-9]+)\)/\1  \2  \3  \4  \5  \6   \7  \8/g"
```
![](attachments/Pasted%20image%2020231116195238.png)

```c
# echo "libgcc-4.8.5-4.h5.x86_64.rpm" | sed -r "s/libgcc-([0-9]+\.[0-9]+.*)\.rpm/\1/g"
4.8.5-4.h5.x86_64

如果是使用grep，则如下所示：
# echo "libgcc-4.8.5-4.h5.x86_64.rpm" | grep -Eo "[0-9]+\.[0-9]+.*x86_64"
4.8.5-4.h5.x86_64
```
```c
grep参数说明：

   -E, --extended-regexp
          Interpret PATTERN as an extended regular expression (ERE, see below).  (-E is specified by POSIX.)

   -o, --only-matching
          Print only the matched (non-empty) parts of a matching line, with each such part on a separate output line.

   -e PATTERN, --regexp=PATTERN
          Use PATTERN as the pattern.  This can be used to specify multiple search patterns, or to protect a pattern beginning with a hyphen (-).  (-e is specified by POSIX.)
```
## 删除
### 删除指定范围的行
```bash
for a in `seq 25 39`; do sed -i '18,145d' 192.22.2.${a}_tcp_8080.conf; done
```
### 删除匹配指定字符串的行
```bash
sed -i '/hello/d' aaaa
sed -i '/^hello/d' aaaa
```

### 删除指定字符串前后的n行

```bash
命令：sed -i '/AISchang/,+9d'   aischang.zone 

会删除文件 aischang.zone 中包含AISchang的这一行以及下面9行的数据

```

![](attachments/Pasted%20image%2020240521151429.png)



## 添加
### 指定字符串之后行添加
- 在指定字符串的下一行插入一行
```c
sed -i '/snat_log_enable/a\        snat_log_stop 1' /etc/dpvs.conf
或者
sed -i "/snat_log_enable/a\\        snat_log_stop 1" /etc/dpvs.conf
```

- 在指定字符串的下面插入多行
```bash
比如，要在keepalived.conf中的 “cluster_tag test-fnat“ 字符串后面插入：
        vxlan {
            vni 1
            local_vtep_ip 192.22.2.100
            local_vtep_mac 0C:42:A1:90:F4:43
            vtep_ip 192.20.144.100
            vtep_port 4789
            vtep_mac 0c:42:a1:78:4f:a1
        }


# sed -i "/cluster_tag test-fnat/a\\        vxlan {\n            vni 1\n            local_vtep_ip 192.22.2.100\n            local_vtep_mac 0C:42:A1:90:F4:43\n            vtep_ip 192.20.144.100\n            vtep_port 4789\n            vtep_mac 0c:42:a1:78:4f:a1\n        }\n" keepalived.conf
```

### 注释指定行
```c
文件中，不是以#开头的行，在开头添加#
sed -i "s/^[^#].*$/#&/g" ${outFile}
```
![](attachments/Pasted%20image%2020230720123333.png)
## 替换

**替换操作**
替换操作：s 命令
```
sed 's/books/BOOKS/' ./test.js
```

**直接编辑文件 选项-i**
接编辑文件 选项-i ，会匹配 file 文件中每一行的所有 book 替换为 books：
```
sed -i 's/book/books/g' file
```


**全面替换**
全面替换标记 g
使用后缀 /g 标记会替换每一行中的所有匹配：
```bash
sed 's/book/books/g' file
```

当需要从第 N 处匹配开始替换时，可以使用 /Ng：
```
echo sksksksksksk | sed 's/sk/SK/2g'
skSKSKSKSKSK

echo sksksksksksk | sed 's/sk/SK/3g'
skskSKSKSKSK

echo sksksksksksk | sed 's/sk/SK/4g'
skskskSKSKSK
```


#### 定界符
以上命令中字符 / 在 sed 中作为定界符使用，也可以使用任意的定界符：
```
sed 's:test:TEXT:g'
sed 's|test|TEXT|g'
```
定界符出现在样式内部时，需要进行转义：
```
sed 's/\/bin/\/usr\/local\/bin/g'
```

### 替换为常量
```c
ls *tcp_8080.conf | xargs sed -i 's/192.20.1/192.20.45/g'
```
### 替换为变量
如下所示，变量用'${xx}' 。
```c
for a in `seq 25 39`; do sed -i 's/192.22.2.'${a}'_tcp_80/192.22.2.'${a}'_tcp_8080/g' 192.22.2.${a}_tcp_8080.conf; done
```

### 替换包含关键字的整行

```bash
查找关键字 user10 所在的行，替换整行内容为aaaaaaaaaa

#sed -i "s/^.user10.*$/aaaaaaaaaa/" useradd.txt

说明：^.* 即以任意开头，包含 user10;
     .*$ 即以任意结尾。整体就是，包含 user10的行。
 
```

![](attachments/Pasted%20image%2020240730111050.png)


### 替换整个目录下文件的指定字符串
```bash
find /path/to/directory -type f -name "*.txt" -exec sed -i '' 's/old_string/new_string/g' {} \;

或者 

sed -i "s/oldstring/newstring/g" `grep "oldstring" -rl /opt/zxq/sentinel-dashboard/src`
```

# 注意
## sed 替换符号链接文件内容的问题
### 问题描述

今天遇到的一个小问题：`foo.txt` 是一个常规文件，`link-1` 是一个**符号连接**（有些地方又叫软连接，**symbol link**），指向的是 `foo.txt`。

我们使用 `sed -i 's/foo/bar/g'` 对 `link-1` 的内容进行替换，替换完成之后，发现 `link-1` 从一个符号连接变成了一个常规的文件。

实验流程如下，我们对 `link-1` 进行原地替换，`-i`，执行之后，`foo.txt` 的内容没有改变，但是 `link-1` 从符号连接变成了常规文件。

```bash
vagrant@foobarhost:~/test$ cat foo.txt
foo
1112
hello
world
vagrant@foobarhost:~/test$ ln -s foo.txt link-1

vagrant@foobarhost:~/test$ ls -l
total 4
-rw-r--r-- 1 vagrant vagrant 21 Oct 27 07:44 foo.txt
lrwxrwxrwx 1 vagrant vagrant  7 Oct 27 07:47 link-1 -> foo.txt

vagrant@foobarhost:~/test$ sed -i 's/foo/bar/g' link-1
vagrant@foobarhost:~/test$ ls -l
total 8
-rw-r--r-- 1 vagrant vagrant 21 Oct 27 07:44 foo.txt
-rw-r--r-- 1 vagrant vagrant 21 Oct 27 07:47 link-1

vagrant@foobarhost:~/test$ cat foo.txt
foo
1112
hello
world

vagrant@foobarhost:~/test$ cat link-1
bar
1112
hello
world
```

### 原因分析

根据上面的 sed 的  特性。

可以用 `strace` 来验证我们的推测：
```bash
strace sed -i 's/foo/bar/g' link-1 > strace.log 2>&1
```

可以看到，`sed` 就是打开了一个临时文件，然后读-处理-写，最后进行 `rename(2)`。

![](attachments/Pasted%20image%2020240514102404.png)

因为创建的临时文件是常规文件，最后再对临时文件进行 rename，因此就会导致导致最终的链接文件 变成了临时文件。

### 解决方法

那有没有不改变文件性质的方法呢？如果解析出来符号连接指向的目标就好了。

很多 Unix 命令，比如 `cp`, `ls` 都有设置 是否 follow 符号连接的选项。

`sed` 应该也有。看了下 `man sed`，发现果然有。
```bash
       --follow-symlinks

              follow symlinks when processing in place
```

再用 `strace` 追踪了一下过程，发现 `rename` 的时候，就会指向符号连接的 target 而不是符号连接本身了。

![](attachments/Pasted%20image%2020240514102714.png)


**注**：其实还有个 -c 选项，–follow-symlinks 针对软链接是适用的，对硬链接是不适用的；-c 对软硬链接都适用。按照 man 手册说法，如果不涉及硬链接，一般 –follow-symlinks 够了，性能较好。

```bash
-c, --copy
     use copy instead of rename when shuffling files in -i mode
```


# 范例
## 查找是否存在，存在则替换，不存在则在指定位置插入
```bash

```

## 特定字符串后进行复杂的追加以及复制

**需求**
```bash
比如存在下面的 文件内容：
aa hello
bb hello
cc hello

aa world
bb world
cc world

aa nihao
bb nihao
cc nihao


如何执行下面的操作：
（1）在cc hello后面的行中添加
dd hello
ee hello
ff hello

（2）在cc world后面的行中添加
dd world
ee world
ff world

（3）在cc nihao后面的行中添加
dd nihao
ee nihao
ff nihao

最终的效果为：
aa hello
bb hello
cc hello
dd hello
ee hello
ff hello

aa world
bb world
cc world
dd world
ee world
ff world

aa nihao
bb nihao
cc nihao
dd nihao
ee nihao
ff nihao
```


**方法**：
python读取文件，然后处理。

**具体**：
python中的str.format的函数的替换功能，以及python的优势。


```python3
# cat python_handle_slave_ip.py
with open('named.conf.views', 'r') as file:
    lines = file.readlines()

new_content = "        192.21.45.5 {num} \n        192.21.45.6 {num} \n        192.21.45.7 {num} \n        192.21.45.8 {num} \n        192.21.45.9 {num} \n        192.21.45.10 {num} \n"

output = []
for line in lines:
    if "10.108.164.22" in line:
        key = line.split()[1]
        key_value = line.split()[2]
        key = key + " " + key_value
        output.extend(new_content.format(num=key))
        output.append(line)
    else:
        output.append(line)

with open('named.conf.views', 'w') as file:
    file.writelines(output)
```

# 参考

```bash
# sed 原地替换和符号连接的一个小坑
https://www.kawabangga.com/posts/5462
```
