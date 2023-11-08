```table-of-contents
```
# 介绍
chmod(英文全拼：change mode),用来变更文件或目录的权限.
Linux/Unix 的文件调用权限分为三级 : 文件所有者（Owner）、用户组（Group）、其它用户（Other Users）。
![](attachments/Pasted%20image%2020231106193301.png)
只有文件所有者和超级用户可以修改文件或目录的权限。可以使用绝对模式（八进制数字模式），符号模式指定文件的权限。
![](attachments/Pasted%20image%2020231106193329.png)
# 使用
```c
chmod  [选项]   [权限] [文件或目录]
chmod [OPTION] [MODE]  FILE
chmod [OPTION] [MODE]  DIRECETORY
```

## 参数说明
mode : 权限设定字串，格式如下 :
```c
u  #用户user，文件或目录的所有者。
g  #用户组group，文件或目录所属组
o  #其它用户others
a  #所有用户all，系统默认
+  #添加权限
-  #取消权限
=  #配置文件的权限为指定的权限
r  #可读权限
w  #可写权限
x  #可执行权限
-  #没有权限
x  #设置可执行权限
s  #设置suid和sgid，可以使用“u+s”，“g+s”的方式来设置
t  #只有目录或文件的所有者才可以删除目录下的文件
-c    #效果类似“-v”参数
-f    #操作过程中不显示任何错误信息；
-R    #递归处理，将指定目录下的所有文件及子目录一并处理；
-v或——verbose    #显示命令运行时的详细执行过程；
--reference=<参考文件或目录> #把指定文件或目录的所属组权限设成参考文件或目录的所属组一样；
<权限范围>+<权限设置> #开启权限范围的文件或目录的该选项权限设置；
<权限范围>-<权限设置> #关闭权限范围的文件或目录的该选项权限设置；
<权限范围>=<权限设置> #指定权限范围的文件或目录的该选项权限设置；
--help     #显示帮助信息
--version  #显示版本信息
```

## 符号模式
使用符号模式可以设置多个项目：who（用户类型），operator（操作符）和 permission（权限），每个项目的设置可以用逗号隔开。

**who 的符号模式表所示:**

|who|用户类型|说明|
|----|----|----|
|u|user|文件所有者|
|g|group|文件所有者所在组|
|o|others|所有其他用户|
|a|all|所有用户, 相当于 _ugo_|

**operator 的符号模式表:**

|Operator|说明|
|---|---|
|+|为指定的用户类型增加权限|
|-|去除指定用户类型的权限|
|=|设置指定用户权限的设置，即将用户类型的所有权限重新设置|

**permission 的符号模式表:**

|模式|名字|说明|
|----|----|----|
|r|读|设置为可读权限|
|w|写|设置为可写权限|
|x|执行权限|设置为可执行权限|
|X|特殊执行权限|只有当文件为目录文件，或者其他类型的用户有可执行权限时，才将文件权限设置可执行|
|s|setuid/gid|当文件被执行时，根据who参数指定的用户类型设置文件的setuid或者setgid权限|
|t|粘贴位|设置粘贴位，只有超级用户可以设置该位，只有文件所有者u可以使用该位|

## 八进制语法
chmod命令可以使用八进制数来指定权限。文件或目录的权限位是由9个权限位来控制，每三位为一组，它们分别是文件所有者（User）的读、写、执行，用户组（Group）的读、写、执行以及其它用户（Other）的读、写、执行。

|#|权限|rwx|二进制|
|---|---|---|---|
|7|读 + 写 + 执行|rwx|111|
|6|读 + 写|rw-|110|
|5|读 + 执行|r-x|101|
|4|只读|r--|100|
|3|写 + 执行|-wx|011|
|2|只写|-w-|010|
|1|只执行|--x|001|
|0|无|---|000|

例如， 765 将这样解释：

- 所有者的权限用数字表达：属主的那三个权限位的数字加起来的总和。如 rwx ，也就是 4+2+1 ，应该是 7。
- 用户组的权限用数字表达：属组的那个权限位数字的相加的总和。如 rw- ，也就是 4+2+0 ，应该是 6。
- 其它用户的权限数字表达：其它用户权限位的数字相加的总和。如 r-x ，也就是 4+0+1 ，应该是 5。


## 特殊权限说明
### SET位权限
Linux系统除了正常的读写操作权限外，还有Linux特殊权限。包括SET位权限（suid，sgid）.

suid/sgid是为了使“没有取得特权用户要完成一项必须要有特权才可以执行的任务”而产生的。 一般用于给可执行的程序或脚本文件进行设置。
其中SUID表示对属主用户增加SET位权限，SGID表示对属组内用户增加SET位权限。
执行文件被设置了SUID、SGID权限后，任何用户执行该文件时，将获得该文件属主、属组账号对应的身份。
在许多环境中，suid 和 sgid 很管用，但是不恰当地使用这些位可能使系统的安全遭到破坏。所以应该尽量避免使用SET位权限程序。（passwd 命令是为数不多的必须使用“suid”的命令之一）。
```c
ls -al /usr/bin/passwd -rwsr-xr-x 1 pythontab pythontab 32988 2018-03-16 14:25 /usr/bin/passwd
```
- suid(set User ID,set UID)的意思是进程执行一个文件时通常保持进程拥有者的UID。然而，如果设置了可执行文件的suid位，进程就获得了该文件拥有者的UID。
- sgid(set Group ID,set GID)意思也是一样，只是把上面的进程拥有者改成进程组就好了。


**作用**：
s权限： 设置使文件在执行阶段具有文件所有者的权限，相当于临时拥有文件所有者的身份.

> 使用场景：比如，在任何用户都拥有对于特定的文件的root权限。

**范例**：
```c
chmod u+s filename  #设置suid位
chmod u-s filename  #去掉suid设置
chmod g+s filename  #设置sgid位
chmod g-s filename  #去掉sgid设置
```
如果一个文件被设置了suid或sgid，在其所有者或所属组权限的可执行位上有明显的标记，如果文件设置了suid且也设置了x（执行）权限，则在其执行权限位上会显示一个字母s(小写)。但是，如果没有设置x权限，则显示为字母S(大写)。如下：
```c
-rwsr-xr-x #设置了suid，且文件所有者也配置了可执行权限
-rwSr--r-- #设置了suid，但文件所有者没有配置可执行权限
-rwxr-sr-x #设置了guid，且所属组也配置了可执行权限
-rw-r-Sr-- #设置了guid，但所属组没有配置可执行权限
```
### 粘滞位权限（sticky）

## reference
根据其他文件/目录的权限，设置权限。

```c
# 根据其他文件的权限设置文件权限。 
chmod --reference=./1.log ./test.log

# 根据其他目录的权限设置目录的权限。 
chmod /var/named/acl/ --reference /var/named/

chmod -R /var/named/acl/* --reference=/etc/named.conf
```

# 实例
将文件 file1.txt 设为所有人皆可读取 :
```c
chmod ugo+r file1.txt

or

chmod a+r file1.txt
```

为 ex1.py 文件拥有者增加可执行权限:
```c
chmod u+x ex1.py
```

将目前目录下的所有文件与子目录皆设为任何人可读取 :
```c
chmod -R a+r *
```
# 应用场景
## 使用普通用户管理DNS服务器
### 背景
使用 Bind 提供的 DNS 服务器，要想配置和管理，默认情况下需要求以 `root` 身份进行。如果是多人维护的情况，`root` 用户权限过高，这导致如果有人做了误操作将会产生十分严重的后果。并且一旦 DNS 服务器被入侵，黑客将有可能直接获取到 `root` 用户权限，安全代价太高。
Linux 系统规定非 `root` 用户一般无法启动小于 1024 的端口，而 DNS 服务器使用 named 进程默认监听于 udp 协议的 53 号端口，如果我们将其服务的端口改为非默认的大于 1024 的，就需要修改大量的配置文件，并且影响很大。

### 思路
普通用户用 `passwd` 命令修改自己的密码，实际上最终更改的是 `/etc/passwd` 文件，此文件是用户管理配置文件，并且只有 `root` 用户才能更改。既然是 `root` 用户才拥有此权限，为什么我们可以通过 `passwd` 命令来修改密码呢，那这就要归功于 `passwd` 设置了 SUID 权限位了。

因此，我们完全可以对启用 DNS 服务的命令 `/usr/sbin/named` 也添加一个 SUID 权限，这样普通用户就能实现启动一个只有 `root` 才能启用小于 1024 的端口了。

### 操作
```c
[root@bogon ~]# useradd user1  
[root@bogon ~]# echo 'user1' | passwd user1 --stdin
Changing password for user user1.
passwd: all authentication tokens updated successfully.
[root@bogon ~]# chmod u+s /usr/sbin/named
[root@bogon ~]# su - user1
```

创建好普通用户和密码，对 named 命令添加 s 位权限后，就使用普通用户启动一个进程。
```c
[user1@bogon ~]$ /usr/sbin/named -c /var/named/named.conf          
[user1@bogon ~]$ ps -ef | grep named
root       7320      1  4 11:33 ?        00:00:00 /usr/sbin/named -c /var/named/named.conf
user1      7325   7237  0 11:33 pts/1    00:00:00 grep --color=auto named
```
一旦配置文件更新了，普通用户也可以使用 `pkill -1 PID` 来重载对应的进程了。
```c
[user1@bogon ~]$ kill -1 7320
```
# 参考
```c
https://www.runoob.com/linux/linux-comm-chmod.html
```