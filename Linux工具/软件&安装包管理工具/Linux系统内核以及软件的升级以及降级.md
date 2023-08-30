# 背景
CentOS 的更新方式和其他 Linux 发行版本不同。首先，每个大版本会有一系列小版本。如 CentOS 6 是大版本，CentOS 6.1、CentOS 6.2 是小版本。当新的小版本发布后，CentOS 将不再继续更新前序小版本。

# 概念
## yum repo仓库
Yum 的仓库配置文件放在文件夹 `/etc/yum.repos.d` 中，以 `.repo` 结尾。  
一个文件中可以配置多个仓库，也可以将不同仓库放在不同文件中。Yum 会扫描所有以 `.repo` 结尾的文件确认所有可用的仓库。

仓库格式如下：
![](attachments/Pasted%20image%2020230817173257.png)
`[repoid]`: repo ID。包括在中括号中，用以标志仓库，不能与其他仓库冲突。如：[base]、[extras]等。  
`name`: 仓库的描述信息，长短不限，可以有空格，但是必不可少。  
`baseurl`：仓库位置。可以是网站（`http://`），ftp 服务器(`ftp://`)，或者本地文件（`file:///`）。目录下一定要有一个 repodata 的文件夹存放包的元数据信息。  
`gpgcheck`: 下载 rpm 包之前是否需要自动进行来源（签名）合法性检测，`1` 表示要检查。  
`gpgkey`：如果启用 gpg 检测，则需要指定 gpgkey 的路径，即使导入过 gpgkey。  
`enabled`: 是否启用这个仓库，`0` 表示不启用，`1` 表示启用，默认是启用的。

# 查看
## 当前的系统版本
```c
查看内核版本
# uname -r
4.18.0-2.4.3.3.x86_64


查看centos 版本
# cat /etc/redhat-release
CentOS Linux release 7.4.2003 (Core)


```
## centos版本和内核版本的对应关系
![](attachments/Pasted%20image%2020230817174148.png)
参考：[# Red Hat Enterprise Linux Release Dates](https://access.redhat.com/articles/3078)

## 查看可升级的内核版本
```c
[root@localhost ~]# yum list kernel --showduplicates    //查看yum可升级的内核版本
...
Installed Packages: 已安装的软件包
kernel.x86_64                           2.6.32-642.el6                                 @anaconda-CentOS-201605220104.x86_64/6.8
Available Packages: 可安装的软件包
kernel.x86_64                           2.6.32-754.el6                                 base
kernel.x86_64                           2.6.32-754.2.1.el6                             updates
kernel.x86_64                           2.6.32-754.3.5.el6                             updates
kernel.x86_64                           2.6.32-754.6.3.el6                             updates
kernel.x86_64                           2.6.32-754.9.1.el6                             updates
kernel.x86_64                           2.6.32-754.10.1.el6                            updates
kernel.x86_64                           2.6.32-754.11.1.el6                            updates
kernel.x86_64                           2.6.32-754.12.1.el6                            updates
kernel.x86_64                           2.6.32-754.14.2.el6                            updates
kernel.x86_64                           2.6.32-754.15.3.el6                            updates
kernel.x86_64                           2.6.32-754.17.1.el6                            updates
kernel.x86_64                           2.6.32-754.18.2.el6                            updates
kernel.x86_64                           2.6.32-754.22.1.el6                            updates
kernel.x86_64                           2.6.32-754.23.1.el6                            updates
kernel.x86_64                           2.6.32-754.24.2.el6                            updates
kernel.x86_64                           2.6.32-754.24.3.el6                            updates
kernel.x86_64                           2.6.32-754.25.1.el6                            updates
kernel.x86_64                           2.6.32-754.27.1.el6                            updates
kernel.x86_64                           2.6.32-754.28.1.el6                            updates
kernel.x86_64                           2.6.32-754.29.1.el6                            updates
kernel.x86_64                           2.6.32-754.29.2.el6                            updates
```

```c
yum update kernel-2.6.32-754.el6.x86_64 //直接执行update升级内核
reboot //重启系统
之后就可以看到系统内核升级到指定版本了。
```

# 问题
希望将Centos 版本从7.4提升到7.9，或者降级到7.2

# 系统&软件的升级与降级
## 安装其他版本软件源
```c
安装 centos-release
# yum install centos-release -y

查看当前所有软件源/仓库信息
# yum repolist all
或者：
# cat /etc/yum.repos.d/CentOS-Vault.repo
```
![](attachments/Pasted%20image%2020230817173619.png)
yum repolist all 显示的是 /etc/yum.repos.d 目录下 .repo文件中的仓库的名称。

### 注意
如果此时 yum repolist all  输出的repo中没有目标版本的 centos, 则需要在  /etc/yum.repos.d 目录下新建.repo文件，创建这样的 repo。（可以将其他.repo文件改名备份）

## 清除 yum 缓存
```c
删除 yum 缓存
# yum clean all

删除 yum 缓存目录
# rm -rf /var/cache/yum
```
## 将系统或软件升级到指定版本

### 将整个系统升级到指定版本

```
# yum --disablerepo='*' --enablerepo='C7.6*' update

即：禁止其他的repo仓库，只开启某些具体repo，然后进行更新
```

（补充：这里以将整个系统升级到 CentOS Linux 7.6 版本为例）

（注意：系统中其他所有软件都会升级到 CentOS Linux 7.6 系统版本中的最新版本）

或者：

```
# yum --disablerepo='*' --enablerepo='C7.6*' upgrade

即：禁止其他的repo仓库，只开启某些具体repo，然后进行更新
```

（补充：这里以将整个系统升级到 CentOS Linux 7.6 版本为例）

（注意：系统中其他所有软件都会升级到 CentOS Linux 7.6 系统版本中的最新版本）

### 将系统内核升级到指定版本

```
# yum --disablerepo='*' --enablerepo='C7.6*' update kernel
```

（补充：这里以只将系统内核 kernel 升级到 CentOS Linux 7.6 系统版本里的最新版本为例）

或者：

```
# yum --disablerepo='*' --enablerepo='C7.6*' upgrade kernel
```

（补充：这里以只将系统内核 kernel 升级到 CentOS Linux 7.6 系统版本里的最新版本为例）

### 升级单个软件
```
# yum --disablerepo='*' --enablerepo='C7.6*' update openssh
```

（补充：这里以将 openssh 软件升级到 CentOS Linux 7.6 系统版本里的最新版本为例为例）

或者：

```
# yum --disablerepo='*' --enablerepo='C7.6*' upgrade openssh
```

（补充：这里以将 openssh 软件升级到 CentOS Linux 7.6 系统版本里的最新版本为例为例）

### 软件降级到某个版本
```
# yum --disablerepo='*' --enablerepo='C7.6*' downgrade openssh
```

（补充：这里以将 openssh 软件降级到 CentOS Linux 7.6 系统版本里的最新版本为例为例）


## 禁止内核更新
```c
[root@spgpu ~]# vim /etc/yum.conf
在[main]部分加上：
exclude=kernel* centos-release
```
这样，在yum update的时候就不会更新内核了。

# 建议
## 安装新内核 or 直接升级内核
使用安装新内核而不是直接升级内核，安装新内核不会覆盖旧内核，而升级会导致新内核直接替换旧内核，可能会导致系统无法启动，安装也可以让我们在升级后有回滚的选择。

一般地，对于大多数 Linux 分发版，使用 yum/dnf 和分发版布官方的存储库来升级内核，这种方式只能升级到该分发版的存储库提供的最新版本，而不是 Linux 内核组织发布的最新内核。

# 安装新内核
## 安装包安装
下载指定版本 kernel： http://rpm.pbone.net/index.php3?stat=3&limit=1&srodzaj=3&dl=40&search=kernel
下载指定版本 kernel-devel：http://rpm.pbone.net/index.php3?stat=3&limit=1&srodzaj=3&dl=40&search=kernel-devel
官方 Centos 7: http://elrepo.org/linux/kernel/el7/x86_64/RPMS/

将rpm包下载上传到服务器上，使用下面的命令安装即可：
```c
yum -y install kernel-ml-devel-4.12.4-1.el7.elrepo.x86_64.rpm
yum -y install kernel-ml-4.12.4-1.el7.elrepo.x86_64.rpm

注：
长期维护版本lt（long-term）
最新主线稳定版ml()
```
## 修改grub中默认启动的内核版本
期望使用最新的 新内核(5.2.8) 。
```c
1> 查看启动顺序
# awk -F\' '$1=="menuentry " { print $2}' /etc/grub2.cfg
CentOS Linux (5.2.8-1.el7.elrepo.x86_64) 7 (Core)
CentOS Linux (4.12.10-1.el7.elrepo.x86_64) 7 (Core)
CentOS Linux (0-rescue-07dc8a29b0184efc8aa87b7c4ea82b45) 7 (Core)
CentOS Linux (0-rescue-bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb) 7 (Core)
如上，新内核(5.2.8)目前位置在0，原来的内核(4.12.10)目前位置在1。

2> 更爱 grub 文件中的 GRUB_DEFAULT
# vi /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
#GRUB_DEFAULT=saved
GRUB_DEFAULT=0
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto consoleblank=0 vga=0x305"
GRUB_DISABLE_RECOVERY="true"

3> 重新创建grub内核配置
# grub2-mkconfig -o /boot/grub2/grub.cfg

4>重启
# reboot
```
# 参考
```c
https://eternalcenter.com/software-specified-version-upgarde-downgrade-centos-linux-7/
https://eternalcenter.com/system-specified-version-upgarde-centos-linux-7/
https://docs.azure.cn/zh-cn/articles/azure-operations-guide/virtual-machines/linux/aog-virtual-machines-linux-centos-howto-upgrade-to-specified-minor-version
```