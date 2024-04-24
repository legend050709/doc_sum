```table-of-contents
```
# history介绍

![](attachments/Pasted%20image%2020240513114930.png)

我们都对 `history` 命令很熟悉。它将终端上 bash 执行过的所有命令存储到 `.bash_history` 文件中，来帮助我们复查用户之前执行过的命令。

## 原理

Linux 命令的历史记录，会持久化存储，默认位置是**当前用户家目录的 `.bash_history` 文件**。

**启动shell时读取文件到缓冲区**
当 Linux 系统启动一个 Shell 时，Shell 会从 `.bash_history` 文件中，读取历史记录，存储在相应内存的缓冲区中。

**运行中保存到缓冲区**
我们平时所操作的 Linux 命令，都会记录在**内存的缓冲区**中。包括 `history` 命令所执行的历史命令管理，都是在操作内存的缓冲区，而不是直接操作`.bash_history` 文件。

**退出shell时写缓存区到bash_history文件**
当我们退出 Shell，比如按下 `Ctrl+D` 时，Shell 进程会把历史记录缓冲区的内容，写回到`.bash_history` 文件中去。




```bash
# cat ~/.bash_history  | head -n 30
#1711020759
cat /proc/sys/net/ipv4/conf/all/forwarding
#1711020769
ifconfig
#1711020770
ip ro
#1711020779
ip ro add 0.0.0.0/0 via 2.3.4.1
#1711020782
ifconfig
#1711020791
ip ro
#1711020800
ifconfig s3 2.3.4.5/24
#1711020802
ip ro add 0.0.0.0/0 via 2.3.4.1
#1711020826
ifconfig
#1711020832
tcpdump -i s3 -nnne
#1711020877
ifconfig
#1711020881
nginx
#1711020885
less /etc/nginx/nginx.conf
#1711020890
netstat -lntp
#1711021495
ifconfig
```

```bash
linux下时间戳转化为时间：

# date -d @1711020759
Thu Mar 21 19:32:39 CST 2024

# date -d @1711020770
Thu Mar 21 19:32:50 CST 2024
```

## 使用
### 相关环境变量

合适使用几个相关的环境变量，让你的 Linux 系统更安全：

- `HISTSIZE`：控制缓冲区历史记录的最大个数
    
- `HISTFILESIZE`：控制历史记录文件中的最大个数
    
- `HISTIGNORE`：设置哪些命令不记录到历史记录
    
- `HISTTIMEFORMAT`：设置历史命令显示的时间格式
    
- `HISTCONTROL`：扩展的控制选项


```bash
export HISTCONTROL=erasedups    # 清除整个命令历史中的重复条目
export HISTCONTROL=ignoredups   # 忽略记录命令历史中连续重复的命令
export HISTCONTROL=ignorespace  # 忽略记录空格开始的命令
export HISTCONTROL=ignoreboth   # 等价于ignoredups和ignorespace
```



### 重复执行命令

如果要重复执行一些命令，可以使用 `!` 来快速执行重复的命令。

举个例子，重复执行第 1024 历史命令，可以执行如下命令

```bash
# !1024 

`1024` 这个编号可以通过 `history` 查看
```

重复执行上一条命令
```bash
$ !!
```

### 交互式搜索历史命令

在 Linux 搜索历史命令，还可以通过交互式的搜索方式，简直高效直接。在命令行输入 `Ctrl+R` 后，进入交互界面，键入需要搜索的关键字，如果匹配到多条命令，可以多次键入 `Ctrl+R` 来切换上一条匹配的命令。

### 控制历史记录总数

默认情况下，Linux 系统最多存储 1000 条历史记录，可以通过 `HISTSIZE` 环境变量查看

对于需要做审计的场景，1000 条历史记录可能会太少了，我们可以修改为合适的值.
```bash
export HISTSIZE=10000
```

**注意**:
`HISTSIZE` 变量只能控制缓冲区中的历史记录数量，如果需要控制 `.bash_history` 文件存储的最大记录数，可以通过 `HISTFILESIZE` 进行控制

上述命令行修改只在当前 Shell 环境生效，如果需要永久生效，需要写入配置文件
```bash
echo "export HISTSIZE=10000" >> ~/.bash_profile
echo "export HISTFILESIZE=200000" >> ~/.bash_profile
source ~/.bash_profile
```


### 更改历史记录文件名

有时，为了方便管理和备份，需要更改历史记录文件的路径和名称。简单，同样可以通过环境变量 `HISTFILE` 更改它的文件名称

```bash
echo "export HISTFILE=/data/backup/chopin.bash_history" >> ~/.bash_profile
souce ~/.bash_profile
```

### 禁用历史记录

处于某种特殊环境，我们需要禁用历史记录
```bash
echo "export HISTSIZE=0" >> ~/.bash_profile
echo "export HISTFILESIZE=0" >> ~/.bash_profile
source ~/.bash_profile
```

## 注意
正常情况下，只有在 Shell 正常退出时，才会将缓冲区内容保存到文件。如果你想主动保存缓冲区的历史记录，执行 `-w` 选项即可
```bash
history -w
```

当然，如果你执行了一些敏感的命令操作，可以执行 `-c` 将缓冲区内容直接删除
```bash
history -c
```




# 基础知识

## who命令

who 可以 查看当前登录用户信息
 who缺省输出包括用户名、终端类型、登陆日期以及远程主机。

```bash
who /var/log/wtmp
		可以查看自从wtmp文件创建以来的每一次登陆情况  
（1）-b：查看系统最近一次启动时间  
（2）-H：打印每列的标题
```

## last命令
last命令 查看用户登录历史；
此命令会读取 /var/log/wtmp文件；/var/log/btmp可以显示远程登陆信息。  

![](attachments/Pasted%20image%2020240513113949.png)

last默认打印所有用户的登陆信息。  
如果想打印某个用户的登陆信息，可以使用  
```bash
last 用户名
```

## lastlog
查看 /etc/passwd 下存在的 所有用户最近一次登录历史。命令将读取/var/log/lastlog文件；用户排列顺序按照/etc/passwd中的顺序。

![](attachments/Pasted%20image%2020240513114141.png)

选项：  
（1） -u：查看某个用户的最后一次登陆历史  
例如： lastlog -u test  
查看用户test的登陆历史  
（2） -t：查看最近几天之内的用户登录历史  
例如： lastlog -t 1  
查看最近1天之内的登陆历史  
（3） -b：查看指定天数之前的用户登录历史  
例如： lastlog -b 60  
查看60天之前的用户登录历史


## 使用pkill强制退出登录的用户

使用pkill可以结束当前登录用户的进程，从而强制退出用户登录，具体使用可以结合w命令；

首先：使用w查看当前登录的用户，注意TTY所示登录进程终端号

```bash
$ w
23:04:27 up 29 days,  7:51,  3 users,  load average: 0.04, 0.06, 0.02
USER     TTY      FROM              LOGIN@   IDLE   JCPU   PCPU WHAT
ramesh   pts/0    10.1.80.56        22:57    8.00s  0.05s  0.01s sshd: ramesh [priv]
jason    pts/1    10.20.48          23:01    2:53   0.01s  0.01s -bash
john     pts/2    10.1.80.7         23:04    0.00s  0.00s  0.00s w
```

其次：使用pkill –9 -t pts/1 结束pts/1进程所对应用户登录(可根据FROM的IP地址或主机号来判断）



# history显示时间

默认情况下 `history` 命令直接显示用户执行的命令而不会输出运行命令时的日期和时间，即使 `history` 命令记录了这个时间。

运行 `history` 命令时，它会检查一个叫做 `HISTTIMEFORMAT` 的环境变量，这个环境变量指明了如何格式化输出 `history` 命令中记录的这个时间。

若该值为 null 或者根本没有设置，则它跟大多数系统默认显示的一样，不会显示日期和时间。


`HISTTIMEFORMAT` 使用 `strftime` 来格式化显示时间（`strftime` - 将日期和时间转换为字符串）。`history` 命令输出日期和时间能够帮你更容易地追踪问题。

```bash
- `%T`： 替换为时间（`%H:%M:%S`）。
- `%F`： 等同于 `%Y-%m-%d` （ISO 8601:2000 标准日期格式）。
```

## 方法

根据需求，有三种不同的设置环境变量的方法。

- 临时设置当前用户的环境变量
- 永久设置当前/其他用户的环境变量
- 永久设置所有用户的环境变量

**注意：** 不要忘了在最后那个单引号前加上空格，否则输出会很混乱的。

### 方法一：环境变量
```bash
# env | grep -i HIST
HISTSIZE=1000
HISTIGNORE=*kdb*:*mysql*:*redis-cli*:*mongo*:*clickhouse*:*git clone*:*password*:*passwd*:*Secret*:*secret*:*token*:*Token*:*authorization*:*Authorization*:*AUTHORIZATION*:*Authentication*:*authentication*:*cookie*:*Cookie*:*session*:*Session*:*login*:*AccessKey*:*accesskey*:*kubeconfig*
HISTCONTROL=ignoredups
HISTTIMEFORMAT=%F %T
```

运行下面命令为为当前用户临时设置 `HISTTIMEFORMAT` 变量。这会一直生效到下次重启。

```bash
export HISTTIMEFORMAT='%F %T '
```

如下所示：

![](attachments/Pasted%20image%2020240510162927.png)



### 方法二：修改.bash.rc文件

将 `HISTTIMEFORMAT` 变量加到 `~/.bashrc` 或 `~/.bash_profile` 文件中，让它对当前的用户永久生效。

```bash
# echo 'HISTTIMEFORMAT="%F %T "' >> ~/.bashrc
或
# echo 'HISTTIMEFORMAT="%F %T "' >> ~/.bash_profile
```

运行下面命令来让文件中的修改生效。
```bash
# source ~/.bashrc
或
# source ~/.bash_profile

```

### 方法三：修改 /etc/profile 文件

将 `HISTTIMEFORMAT` 变量加入 `/etc/profile` 文件中，让它对所有用户永久生效。
```bash
echo 'HISTTIMEFORMAT="%F %T "' >> /etc/profile
```

运行下面命令来让文件中的修改生效。
```bash
source /etc/profile
```


## 拓展

其实这些对于审计需求，还不够，可以加上更详细的信息：

```bash
#  export HISTTIMEFORMAT="%F %T `who -u am i 2>/dev/null| awk '{print $NF}'|sed \-e 's/[()]//g'` `whoami` "

 6  2021-04-18 16:07:48 113.200.44.237 root ls
 7  2021-04-18 16:07:59 113.200.44.237 root pwd
 8  2021-04-18 16:08:14 113.200.44.237 root history

```

# history显示所有用户的历史命令
## 背景
每个用户都有一份命令历史记录  
查看当前用户的历史命令。
```bash
$HOME/.bash_history  
或者在终端输入： history
```

在linux系统的环境下，不管是root用户还是其它的用户只有登陆系统后用进入操作我们都可以通过命令history来查看历史记录，可是假如一台服务器多人登陆，一天因为某人误操作了删除了重要的数据。
这时候通过查看历史记录（命令：history）是没有什么意义了（因为history只针对登录用户下执行有效，即使root用户也无法得到其它用户histotry历史）。

那有没有什么办法实现通过记录登陆后的IP地址和某用户名所操作的历史记录呢？答案：有的。

## 方法

### 方法一：逐个用户查看

要查看所有用户的`history`命令记录，可以使用以下步骤：

1. 切换到`root`用户或具有管理员权限的用户。
2. 打开`/etc/passwd`文件，查找所有的用户账号。
3. 依次切换到每个用户，使用`history`命令查看该用户的历史命令记录。命令如下：
```bash
su - <用户名>
history
```
4. 每个用户的历史命令记录都会显示在屏幕上，可以使用`more`或`less`命令进行分页查看。


**注意**：
> 这种方法只能查看每个用户使用**当前Shell**窗口执行的历史命令记录，如果用户使用了其他终端或Shell窗口执行过命令（用户还没关闭那个终端窗口），这些命令就无法在当前窗口中查看。
> 另外，历史命令记录也可能会被用户删除或清空，所以这种方法也并不完全可靠。


### 方法二：修改/etc/profile脚本

通过在/etc/profile里面加入以下代码就可以实现：
```bash
PS1="`whoami`@`hostname`:"'[$PWD]'
history
USER_IP=`who -u am i 2>/dev/null| awk '{print $NF}'|sed -e 's/[()]//g'`
if [ "$USER_IP" = "" ]
then
USER_IP=`hostname`
fi
if [ ! -d /tmp/dbasky ]
then
mkdir /tmp/dbasky
chmod 777 /tmp/dbasky
fi
if [ ! -d /tmp/dbasky/${LOGNAME} ]
then
mkdir /tmp/dbasky/${LOGNAME}
chmod 300 /tmp/dbasky/${LOGNAME}
fi
export HISTSIZE=4096
DT=`date "+%Y-%m-%d_%H:%M:%S"`
export HISTFILE="/tmp/dbasky/${LOGNAME}/${USER_IP} dbasky.$DT"
chmod 600 /tmp/dbasky/${LOGNAME}/*dbasky* 2>/dev/null
```


source /etc/profile 使用脚本生效

退出用户，重新登录
上面脚本在系统的/tmp新建个dbasky目录，记录所有登陆过系统的用户和IP地址（文件名），每当用户登录/退出会创建相应的文件，该文件保存这段用户登录时期内操作历史，可以用这个方法来监测系统的安全性。

### 方法三：使用账户审计工具

Linux系统中有一些账户审计工具可以用于监控和记录用户的行为，包括执行的命令。常见的账户审计工具有`auditd`和`acct`等。可以使用这些工具来监控记录用户的命令执行情况，并生成相应的报告。

# Linux 中多终端同步 history 记录
## 背景
Linux 默认配置是当打开一个 shell 终端后，执行的所有命令均不会写入到`~/.bash_history`文件中，只有当前用户退出后才会写入，这期间发生的所有命令其它终端是感知不到的。

## 问题场景

在网络上看到 2 个问题，有点意思：

假若之前`history`命令记录为 c0，用户先打开了 shell 终端 a，执行了一部分命令 c1，又打开了一个 shell 终端 b，又执行了一部分命令 c2。

- **问题1：**终端 a 执行的这部分命令终端 b 上看不到。
- **问题2：**终端 a 正常退出，相关命令会写入到`~/.bash_history`文件中（c1 命令也会写入，即 c0+c1），等到终端 b 正常退出后，相关命令也会写入到`~/.bash_history`文件中，注意这个时候终端 b 写入的内容为 c0+c2，也即 c1 记录会丢失！！！

问题 1 的确会出现！
但是问题 2 貌似不会出现；个人在 CentOS 7 中测试了一下，发现终端 a 正常退出，相关命令的确会写入到`~/.bash_history`文件中，即 c0+c1；但终端 b 也正常退出后，终端 b 的相关命令是会自动**追加**到`~/.bash_history`文件，这时候`~/.bash_history`的文件内容 = c0 + c1 + c2！

## 需求

如果在多个打开的终端中实时同步 history（例如，如果我 ls 在一个终端中，切换到另一个已经运行的终端，然后按向上，`ls`出现）的确也是有一定的使用需求。

我们增加一个需求：当打开一个 shell 终端后，不管是正常退出还是非正常退出，执行的所有命令均**实时追加**到`~/.bash_history`文件中，但当前终端不会实时同步其他终端的 history，除非我重新开启了一个新终端。
>  即：当前终端没有退出时，也将 执行的 命令追加到 `~/.bash_history`文件中。其他的已经开启的终端，其history 不受到 `~/.bash_history`文件 变更的 影响。

## 解决方法
### 实时同步多个终端的 history 记录

```bash
# Avoid duplicates
export HISTCONTROL=ignoredups:erasedups

# When the shell exits, append to the history file instead of overwriting it
shopt -s histappend

# After each command, append to the history file and reread it
export PROMPT_COMMAND="${PROMPT_COMMAND:+$PROMPT_COMMAND$'\n'}history -a; history -c; history -r"
```

### 多个终端执行的命令均实时追加到`~/.bash_history`文件中

```bash
shopt -s histappend 
PROMPT_COMMAND="history -a"
```


# 参考
```bash
# 让 history 命令显示日期和时间

https://linux.cn/article-9253-1.html

# [谁动了我的Linux？原来history这么强大！](https://segmentfault.com/a/1190000039858269)

```