```table-of-contents
```
#  Linux中的进程和服务

计算机中，一个正在执行的程序或命令，被叫做“进程”（process）。
启动之后一直存在、常驻内存的进程，一般被称作“服务”（service）。

# systemd服务介绍
## 历史
历史上，Linux 的启动一直采用`init`进程。
下面的命令用来启动服务。
```bash
$ sudo /etc/init.d/apache2 start
# 或者
$ service apache2 start
```

这种方法有两个缺点。
一是**启动时间长。`init`进程是串行启动，只有前一个进程启动完，才会启动下一个进程**。
二是启动脚本复杂。`init`进程只是执行启动脚本，不管其他事情。脚本需要自己处理各种情况，这往往使得脚本变得很长。

 注：centos7版本以后：==系统的第一个进程由`initd`变成了`systemd`，`systemd`就像是一个小型操作系统一样，接管了系统初始化启动和进程服务管理等==。

## 介绍
**Systemd**：Systemd 就是为了解决这些问题而诞生的。它的设计目标是，为**系统的启动和管理**提供一套完整的解决方案。

因此，systemd 是 ==系统启动和服务器守护进程管理器==，负责在系统启动或运行时，激活系统资源和服务器进程、以及其它进程。

使用了 Systemd，就不需要再用`init`了。Systemd 取代了`initd`，成为系统的第一个进程（PID 等于 1），其他进程都是它的子进程。

## Systemd 的 优缺点
`Systemd` 的优点是功能强大，使用方便。
缺点是体系庞大，非常复杂。事实上，现在还有很多人反对使用 Systemd，理由就是它过于复杂，与操作系统的其他部分强耦合，违反"keep simple, keep stupid"的Unix 哲学。

## system 工具集

![](attachments/Pasted%20image%2020231218115433.png)

Systemd 并不是一个命令，而是一组命令，涉及到系统管理的方方面面。
- `systemctl`是 Systemd 的主命令，用于管理系统。
- `systemd-analyze`：显示此次系统启动时运行每个服务所消耗的时间，可以用于分析系统启动过程中的性能瓶颈。
- `hostnamectl`：用于查看和修改系统的主机名和主机信息
```bash
# 显示当前主机的信息
$ hostnamectl

# 设置主机名。
$ sudo hostnamectl set-hostname levonfly
```

- `journalctl`：用于查看系统、内核、各类应用服务日志
- `systemd-notify`：Systemd 的内部工具，用于通知服务的状态变化
- `systemd-path`：Systemd 的内部工具，用于显示系统上下文中的各种路径配置。

## 用户进程和systemd的关系
操作系统使用`systemd`后，所有用户进程都是`systemd`的后代进程。如下所示：
```bash
$ pstree -p  
systemd(1)─┬─agetty(1056)  
           ├─auditd(737)───{auditd}(738)  
           ├─crond(810)  
           ├─dbus-daemon(761)  
           ├─dhclient(966)  
           ├─irqbalance(764)  
           ├─lvmetad(573)  
           ├─master(1140)─┬─pickup(1141)  
           │              └─qmgr(1142)  
           ├─mysqld(1068)─┬─{mysqld}(1161)  
           │              └─......  
           ├─polkitd(763)─┬─{polkitd}(814)  
           │              ├─......  
           ├─rpcbind(762)  
           ├─rsyslogd(1047)─┬─{rsyslogd}(1093)  
           │                └─{rsyslogd}(1094)  
           ├─sshd(1042)───sshd(1110)─┬─bash(1143)───pstree(2240)  
           │                         └─bash(1492)  
           ├─systemd-journal(545)  
           ├─systemd-logind(773)  
           ├─systemd-udevd(576)  
           └─tuned(1024)─┬─{tuned}(1377)  
                         ├─{tuned}(1378)  
                         ├─{tuned}(1380)  
                         └─{tuned}(1381)
```
> 注意：虽然从进程树关系来看，所有进程都直接或间接地受到systemd的管理。
> 但是，并非所有`systemd`的子进程都受`Systemd Unit`管理单元的管理。只有那些由`systemd`方式启动的服务进程(比如`systemctl`命令启动)才受到`Systemd Unit`管理单元的监控和管理。

比如，用户可以通过下面两种方式启动Nginx服务进程：
```bash
nginx                    # (1)  
systemctl start nginx    # (2)
```
但`systemd`只能监控、管理第(2)种方式启动的`nginx`服务。比如第一种方式启动的`nginx`，无法使用`systemctl stop nginx`来停止。

所以，**systemd下的直系子进程可分为两类：受systemd管理的子进程和不受systemd管理的子进程**。
![](attachments/Pasted%20image%2020231219161753.png)

![](attachments/Pasted%20image%2020231219162018.png)
![](attachments/Pasted%20image%2020231219162056.png)

## systemd-analyze
`systemd-analyze`是一个**分析启动性能**的工具，用于**分析启动时服务时间消耗**。

它将列出有关每个服务启动所需时间的信息，包括启动时内核和用户空间所花费的时间。



### 使用
`systemd-analyze`可使用的命令：
```bash
- systemd-analyze [OPTIONS…] [time]
- systemd-analyze [OPTIONS…] blame
- systemd-analyze [OPTIONS…] critical-chain [UNIT…]
- systemd-analyze [OPTIONS…] plot [> file.svg]
- systemd-analyze [OPTIONS…] dot [PATTERN…] [> file.dot]
- systemd-analyze [OPTIONS…] dump
- systemd-analyze [OPTIONS…] set-log-level LEVEL
- systemd-analyze [OPTIONS…] set-log-target TARGET
- systemd-analyze [OPTIONS…] get-log-level
- systemd-analyze [OPTIONS…] get-log-target
- systemd-analyze [OPTIONS…] syscall-filter [SET…]
- systemd-analyze [OPTIONS…] verify [FILES…]
```

```bash                           
# 查看启动耗时
$ systemd-analyze   

# 查看每个服务的启动耗时
$ systemd-analyze blame

# 显示瀑布状的启动过程流
$ systemd-analyze critical-chain

# 显示指定服务的启动流
$ systemd-analyze critical-chain atd.service
```


### 范例
使用`systemd-analyze blame`可以查看每个单元的启动时间。如下所示：
```c
  5.923s dev-sda1.device
  5.543s dev-loop14.device
  5.478s dev-loop15.device
  5.121s dev-loop13.device
  2.442s snapd.service
  1.163s snapd.seeded.service
  1.036s udisks2.service
   964ms networkd-dispatcher.service
   930ms fwupd.service
   883ms apparmor.service
   758ms ModemManager.service
   677ms accounts-daemon.service
   581ms systemd-udev-trigger.service
   538ms systemd-journald.service
   530ms NetworkManager-wait-online.service
   528ms grub-common.service
   514ms speech-dispatcher.service
   499ms dev-loop4.device
   493ms networking.service
   481ms dev-loop7.device
   474ms avahi-daemon.service
   444ms dev-loop2.device
   433ms dev-loop0.device
   418ms NetworkManager.service
   415ms dev-loop1.device
   415ms systemd-modules-load.service
   414ms dev-loop5.device
   396ms dev-loop6.device
   393ms dev-loop3.device
   363ms dev-loop8.device
   344ms systemd-random-seed.service
```
使用`systemd-analyze plot > boot.svg`生成一张启动详细信息矢量图，然后用图像浏览器或者网页浏览器打开查看 。
![](attachments/Pasted%20image%2020240102141221.png)


# systemd的Unit
Systemd 可以管理所有系统资源。不同的资源统称为 Unit（单位）。
简单说，单元就是 Systemd 的最小功能单位，是单个进程的描述。一个个小的单元互相调用和依赖，组成一个庞大的任务管理系统，这就是 Systemd 的基本思想。

## unit 分类
由于 Systemd 要做的事情太多，导致单元有很多不同的种类，大概一共有12种。举例来说，Service 单元负责后台服务，Timer 单元负责定时器，Slice 单元负责资源的分配。

```c
- Service unit：系统服务
- Target unit：多个 Unit 构成的一个组
- Device Unit：硬件设备
- Mount Unit：文件系统的挂载点
- Automount Unit：自动挂载点
- Path Unit：文件或路径
- Scope Unit：不是由 Systemd 启动的外部进程
- Slice Unit：进程组
- Snapshot Unit：Systemd 快照，可以切回某个快照
- Socket Unit：进程间通信的 socket
- Swap Unit：swap 文件
- Timer Unit：定时器
```

常见查看：
```bash
# 列出正在运行的 Unit
$ systemctl list-units

# 查看所有正在运行的 Service 单元
$ systemctl list-unit-files --type service

# 查看所有正在运行的 Timer 单元
$ systemctl list-unit-files --type timer

# 列出所有Unit，包括没有找到配置文件的或者启动失败的
$ systemctl list-units --all

# 列出所有没有运行的 Unit
$ systemctl list-units --all --state=inactive

# 列出所有加载失败的 Unit
$ systemctl list-units --failed
```
## Unit 的状态
`systemctl status`命令用于查看系统状态和单个 Unit 的状态。
```bash
# 显示系统状态
$ systemctl status

# 显示单个 Unit 的状态
$ sysystemctl status bluetooth.service

# 显示远程主机的某个 Unit 的状态
$ systemctl -H root@rhel7.example.com status httpd.service
```

除了`status`命令，`systemctl`还提供了三个查询状态的简单方法，主要供脚本内部的判断语句使用。
```bash
# 显示某个 Unit 是否正在运行
$ systemctl is-active application.service

# 显示某个 Unit 是否处于启动失败状态
$ systemctl is-failed application.service

# 显示某个 Unit 服务是否建立了启动链接（开机启动）
$ systemctl is-enabled application.service
```
## unit 的相关文件
每个单元都有一个单元描述文件，它们分散在三个目录。
- `/lib/systemd/system`：系统默认的单元文件
- `/etc/systemd/system`：用户安装的软件的单元文件
- `/usr/lib/systemd/system`：用户自己定义的单元文件

## Unit 管理
最常用的是下面这些命令，用于启动和停止 Unit（主要是 service）。
```bash
# 开机启动
sudo systemctl enable httpd
注：设置开机启动以后，软件并不会立即启动，必须等到下一次开机。如果想现在就运行该软件，那么要执行`systemctl start`命令。

# 立即启动一个服务
$ sudo systemctl start apache.service

# 立即停止一个服务
$ sudo systemctl stop apache.service

# 查看服务是否开机自启动  
systemctl is-enabled Service_Name

# 查看所有开机自启动的服务  
systemctl list-unit-files --type service | grep 'enabled'

# 重启一个服务
$ sudo systemctl restart apache.service

# 杀死一个服务的所有子进程
$ sudo systemctl kill apache.service

# 重新加载一个服务的配置文件
$ sudo systemctl reload apache.service

# 重载所有修改过的配置文件
$ sudo systemctl daemon-reload

# 显示某个 Unit 的所有底层参数
$ systemctl show httpd.service

# 显示某个 Unit 的指定属性的值
$ systemctl show -p CPUShares httpd.service

# 设置某个 Unit 的指定属性
$ sudo systemctl set-property httpd.service CPUShares=500
```
### enable 开机启动
对于那些支持 Systemd 的软件，安装的时候，会自动在`/usr/lib/systemd/system`目录添加一个配置文件。
如果你想让该软件开机启动，就执行下面的命令（以`httpd.service`为例）。
```bash
 sudo systemctl enable httpd
```
上面的命令相当于在`/etc/systemd/system`目录添加一个符号链接，指向`/usr/lib/systemd/system`里面的`httpd.service`文件。

这是因为开机时，`Systemd`只执行`/etc/systemd/system`目录里面的配置文件。这也意味着，如果把修改后的配置文件放在该目录，就可以达到覆盖原始配置的效果。

#### disable 禁止开机启动

执行`systemctl disable xxx`后，会禁用这个服务。它实现的方法是将服务对应的软连接从`/etc/systemd/system`中删除。

```bash
 sudo systemctl disable httpd
```

#### 范例
```bash
[root@NameNode01 system]# systemctl enable NetworkManager
Created symlink from /etc/systemd/system/multi-user.target.wants/NetworkManager.service to /usr/lib/systemd/system/NetworkManager.service.
Created symlink from /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service to /usr/lib/systemd/system/NetworkManager-dispatcher.service.
Created symlink from /etc/systemd/system/network-online.target.wants/NetworkManager-wait-online.service to /usr/lib/systemd/system/NetworkManager-wait-online.service.
```

这个命令会在`/etc/systemd/system/`目录下创建需要的符号链接，表示服务需要进行启动。通过stdout输出的信息可以看到，软连接实际指向的文件为`/usr/lib/systemd/system/`目录中的文件，实际起作用的也是这个目录中的文件。

```bash
[root@NameNode01 system]# systemctl disable NetworkManager
Removed symlink /etc/systemd/system/multi-user.target.wants/NetworkManager.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.NetworkManager.service.
Removed symlink /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service.
Removed symlink /etc/systemd/system/network-online.target.wants/NetworkManager-wait-online.service.
```
在执行`systemctl disable xxx`的时候，实际只是删除了软连接，并不会产生其他影响。

### mask 屏蔽服务

`systemctl mask` 命令用于屏蔽服务，使其无法被 `systemctl` 启动。屏蔽服务的情况通常包括：

- **安全性要求：** 一些敏感服务可能需要被完全禁用，以防止潜在的安全威胁。
- **系统定制：** 在定制化系统中，可能需要屏蔽一些默认启用的服务，以符合特定需求。
- **依赖关系：** 在某些情况下，为了确保系统的正确运行，可能需要屏蔽某些服务的启动。

```bash
systemctl mask servicename
```

#### 注意
在执行了mask禁用服务后，强行start服务会报
```text
Failed to start <service-name>: Unit is masked.
```


在执行了mask禁用服务后，强行enable开机自启会报
```text
Failed to execute operation: Cannot send after transport endpoint shutdown
```

#### unmask 取消屏蔽

```bash
systemctl unmask servicename
```


#### mask和 disable的区别

执行 `systemctl mask xxx`会`屏蔽`这个服务。它和`systemctl disable xxx`的区别在于，前者只是删除了符号链接，后者会建立一个指向`/dev/null`的符号链接，这样，即使有其他服务要启动被`mask`的服务，仍然无法执行成功。

- **`systemctl disable` 命令：** 用于禁用一个服务的开机启动。当你禁用一个服务后，系统将不再开机启动该服务，但仍然允许手动启动。

- **`systemctl mask` 命令：** 用于屏蔽一个服务，将其单元文件链接到 `/dev/null`，使其无法被 `systemctl` 启动。屏蔽后，即使尝试手动启动服务，也将无法成功。


#### 范例

```bash
[root@NameNode01 system]# systemctl mask NetworkManager 
Created symlink from /etc/systemd/system/NetworkManager.service to /dev/null.
```

### start 启动服务
设置开机启动以后，软件并不会立即启动，必须等到下一次开机。如果想现在就运行该软件，那么要执行`systemctl start`命令。
```bash
sudo systemctl start httpd
```


### status 查看服务的状态
执行上面的`systemctl start`命令以后，有可能启动失败，因此要用`systemctl status`命令查看一下该服务的状态。
```bash
$ sudo systemctl status httpd

httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled)
   Active: active (running) since 金 2014-12-05 12:18:22 JST; 7min ago
 Main PID: 4349 (httpd)
   Status: "Total requests: 1; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├─4349 /usr/sbin/httpd -DFOREGROUND
           ├─4350 /usr/sbin/httpd -DFOREGROUND
           ├─4351 /usr/sbin/httpd -DFOREGROUND
           ├─4352 /usr/sbin/httpd -DFOREGROUND
           ├─4353 /usr/sbin/httpd -DFOREGROUND
           └─4354 /usr/sbin/httpd -DFOREGROUND

12月 05 12:18:22 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
12月 05 12:18:22 localhost.localdomain systemd[1]: Started The Apache HTTP Server.
12月 05 12:22:40 localhost.localdomain systemd[1]: Started The Apache HTTP Server.
```
上面的输出结果含义如下。
```text
- `Loaded`行：配置文件的位置，是否设为开机启动
- `Active`行：表示正在运行
- `Main PID`行：主进程ID
- `Status`行：由应用本身（这里是 httpd ）提供的软件当前状态
- `CGroup`块：应用的所有子进程
- 日志块：应用的日志
```


#### 范例

![](attachments/Pasted%20image%2020241113114041.png)

### list-unit-files 查看服务开机状态

![](attachments/Pasted%20image%2020241113114131.png)

### stop 停止服务
终止正在运行的服务，需要执行`systemctl stop`命令。
```bash
sudo systemctl stop httpd.service
```
有时候，该命令可能没有响应，服务停不下来。这时候就不得不"杀进程"了，向正在运行的进程发出`kill`信号。
```bash
sudo systemctl kill httpd.service
```

此外，重启服务要执行`systemctl restart`命令。
```bash
sudo systemctl restart httpd.service
```



## unit的依赖关系
Unit 之间存在依赖关系：A 依赖于 B，就意味着 Systemd 在启动 A 的时候，同时会去启动 B。
`systemctl list-dependencies`命令列出一个 Unit 的所有依赖。
```bash
systemctl list-dependencies nginx.service
```
上面命令的输出结果之中，有些依赖是 Target 类型（详见下文），默认不会展开显示。如果要展开 Target，就需要使用`--all`参数。
```bash
systemctl list-dependencies --all nginx.service
```

## Unit 的配置文件
每一个 `Unit` 都有一个配置文件(形如：`xxx.service`)，告诉 `Systemd `怎么启动这个 `Unit` 。

Systemd 默认从目录`/etc/systemd/system/`读取配置文件。但是，里面存放的大部分文件都是符号链接，指向目录`/usr/lib/systemd/system/`，真正的配置文件存放在那个目录。

> 注：`systemctl enable`命令用于在上面两个目录之间，建立符号链接关系。
```bash
$ sudo systemctl enable clamd@scan.service
# 等同于
$ sudo ln -s '/usr/lib/systemd/system/clamd@scan.service' '/etc/systemd/system/multi-user.target.wants/
```
如果配置文件里面设置了开机启动，`systemctl enable`命令相当于激活开机启动。
与之对应的，`systemctl disable`命令用于在两个目录之间，撤销符号链接关系，相当于撤销开机启动。

配置文件的后缀名，就是该 Unit 的种类，比如`sshd.socket`。如果省略，Systemd 默认后缀名为`.service`，所以`sshd`会被理解成`sshd.service`。


### 列出配置文件
`systemctl list-unit-files`命令用于列出所有配置文件。
```bash
# 列出所有配置文件
$ systemctl list-unit-files

# 列出指定类型的配置文件
$ systemctl list-unit-files --type=service
```

### 配置文件的状态
这个命令会输出一个列表。
```bash
$ systemctl list-unit-files

UNIT FILE              STATE
chronyd.service        enabled
clamd@.service         static
clamd@scan.service     disabled
```

这个列表显示每个配置文件的状态，一共有四种。
```bash
- enabled：已建立启动链接（开启自启动）
- disabled：没建立启动链接（开机不启动）
- static：该配置文件没有`[Install]`部分（无法执行），只能作为其他配置文件的依赖
- masked：该配置文件被禁止建立启动链接
```
> 注：从配置文件的状态无法看出，该 Unit 是否正在运行。这必须执行前面提到的`systemctl status`命令。如下所示：
```bash
systemctl status bluetooth.service
```

### 查看配置文件
前面说过，配置文件主要放在`/usr/lib/systemd/system`目录，也可能在`/etc/systemd/system`目录。找到配置文件以后，使用文本编辑器打开即可。

`systemctl cat`命令也可以用来查看配置文件，下面以`sshd.service`文件为例，它的作用是启动一个 SSH 服务器，供其他用户以 SSH 方式登录。
```bash
$ systemctl cat sshd.service

[Unit]
Description=OpenSSH server daemon
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target sshd-keygen.service
Wants=sshd-keygen.service

[Service]
EnvironmentFile=/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D $OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
Type=simple
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

### 修改配置文件
一旦修改配置文件，就要让 SystemD 重新加载配置文件，然后重新启动，否则修改不会生效。
```bash
 # 重载所有修改过的配置文件
$ sudo systemctl daemon-reload

# 重新加载一个服务的配置文件
systemctl reload nginx

$ sudo systemctl restart httpd.service
```
### 配置文件的格式
配置文件就是普通的文本文件，可以用文本编辑器打开。
`systemctl cat`命令可以查看配置文件的内容。
```bash
$ systemctl cat atd.service

[Unit]
Description=ATD daemon

[Service]
Type=forking
ExecStart=/usr/bin/atd

[Install]
WantedBy=multi-user.target
```
从上面的输出可以看到，配置文件分成几个区块。每个区块的第一行，是用方括号表示的区块名，比如`[Unit]`。每个区块内部是一些等号连接的键值对。
```bash
[Section]
Directive1=value
Directive2=value

. . .
```
> 注意：**配置文件的区块名和字段名，都是大小写敏感的。另外，键值对的等号两侧不能有空格**。

## 配置文件的区块详解
### `[Unit]`区块

**`[Unit]`区块通常是配置文件的第一个区块，用来定义 Unit 的元数据，配置与其他 Unit 的关系**。

它的主要字段如下。
```text
Description：简短描述
Documentation：文档地址
Requires：表示"强依赖"关系；当前 Unit 依赖的其他 Unit，如果它们没有运行，当前 Unit 会启动失败
Wants：表示"弱依赖"关系；与当前 Unit 配合的其他 Unit，如果它们没有运行，当前 Unit 不会启动失败
BindsTo：与Requires类似，它指定的 Unit 如果退出，会导致当前 Unit 停止运行
Before：如果该字段指定的 Unit 也要启动，那么当前 Unit 必须在他们之前启动
After：如果该字段指定的 Unit 也要启动，那么当前 Unit 在他们之后启动
Conflicts：这里指定的 Unit 不能与当前 Unit 同时运行
Condition...：当前 Unit 运行必须满足的条件，否则不会运行
Assert...：当前 Unit 运行必须满足的条件，否则会报启动失败
```

>注：`After`和`Before`字段只涉及启动顺序，不涉及依赖关系。设置依赖关系，需要使用`Wants`字段和`Requires`字段。`Wants`字段与`Requires`字段只涉及依赖关系，与启动顺序无关，默认情况下是同时启动的。

#### 范例说明
举例来说：
某 `Web` 应用需要 `postgresql` 数据库储存数据。在配置文件中，它只定义要在 `postgresql` 之后启动，而没有定义依赖 `postgresql` 。上线后，由于某种原因，`postgresql` 需要重新启动，在停止服务期间，该 Web 应用就会无法建立数据库连接。

`Wants`字段：表示`sshd.service`与`sshd-keygen.service`之间存在"弱依赖"关系，即如果 `sshd-keygen.service`  启动失败或停止运行，不影响`sshd.service`继续执行。  

`Requires`字段则表示"强依赖"关系，即如果该服务启动失败或异常退出，那么`sshd.service`也必须退出。


### `[Service]`区块
**`Service`区块定义如何启动当前服务**。

`Type`：定义启动时的进程行为。它有以下几种值。
```text
- `Type=simple`：默认值，`ExecStart`字段启动的进程为主进程。  
服务进程不会 fork，如果该服务要启动其他服务，不要使用此类型启动。

- `Type=forking`：`ExecStart`字段将以`fork()`方式从父进程创建子进程启动，创建后父进程会立即退出，子进程成为主进程。通常需要指定`PIDFile`字段，以便 `Systemd` 能够跟踪服务的主进程。对于常规的守护进程（daemon），除非你确定此启动方式无法满足需求，使用此类型启动即可。

- `Type=oneshot`：只执行一次，Systemd 会等当前服务退出，再继续往下执行, 适用于只执行一项任务、随后立即退出的服务。通常需要指定`RemainAfterExit=yes`字段，使得 Systemd 在服务进程退出之后仍然认为服务处于激活状态。

- `Type=dbus`：类似于`simple`，但会等待 D-Bus 信号后启动

- `Type=notify`：当前服务启动完毕会发出通知信号，通知 Systemd，然后 Systemd 再启动其他服务。

- `Type=idle`：Systemd 会等到其他任务都执行完，才会启动该服务。一种使用场合是：让该服务的输出，不与其他服务的输出相混合。
```

启动，启动前，启动后的相关命令。
```
ExecStart：启动当前服务的命令
ExecStartPre：启动当前服务之前执行的命令
ExecStartPost：启动当前服务之后执行的命令
ExecReload：重启当前服务时执行的命令
ExecStop：停止当前服务时执行的命令
ExecStopPost：停止当其服务之后执行的命令
RestartSec：自动重启当前服务间隔的秒数
Restart：定义何种情况 Systemd 会自动重启当前服务，可能的值包括always（总是重启）、on-success、on-failure、on-abnormal、on-abort、on-watchdog
TimeoutSec：定义 Systemd 停止当前服务之前等待的秒数
Environment：指定环境变量

RemainAfterExit：该字段为yes，表示进程退出以后，服务仍然保持执行。
KillMode：定义 Systemd 如何停止服务。
```

注：定义的时候，所有路径都要写成绝对路径，比如`bash`要写成`/bin/bash`，否则 Systemd 会找不到。


#### 启动命令
##### 环境参数
许多软件都有自己的环境参数文件，该文件可以用`EnvironmentFile`字段设置。
`EnvironmentFile`字段：指定当前服务的环境参数文件。该文件内部的`key=value`键值对，可以用`$key`的形式，在当前配置文件中获取。
上面的例子中，sshd 的环境参数文件是`/etc/sysconfig/sshd`。

##### 启动命令字段
配置文件里面最重要的字段是`ExecStart`。
`ExecStart`字段：定义启动进程时执行的命令。
上面的例子中，启动`sshd`，执行的命令是`/usr/sbin/sshd -D $OPTIONS`，其中的变量`$OPTIONS`就来自`EnvironmentFile`字段指定的环境参数文件。
```text
# systemctl cat sshd, 输出如下所示：
# /usr/lib/systemd/system/sshd.service
[Unit]
Description=OpenSSH server daemon
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target sshd-keygen.service
Wants=sshd-keygen.service

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/sshd
ExecStart=/usr/sbin/sshd -D $OPTIONS  # OPTIONS 来自 EnvironmentFile字段指定的环境变量文件。
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target

# cat /etc/sysconfig/sshd, 输出如下所示：
# Configuration file for the sshd service.

# The server keys are automatically generated if they are missing.
# To change the automatic creation uncomment and change the appropriate
# line. Accepted key types are: DSA RSA ECDSA ED25519.
# The default is "RSA ECDSA ED25519"

# AUTOCREATE_SERVER_KEYS=""
# AUTOCREATE_SERVER_KEYS="RSA ECDSA ED25519"

# Do not change this option unless you have hardware random
# generator and you REALLY know what you are doing

SSH_USE_STRONG_RNG=0
# SSH_USE_STRONG_RNG=1
```

注：所有的启动设置之前，都可以加上一个连词号（`-`），表示"抑制错误"，即发生错误的时候，不影响其他命令的执行。比如，`EnvironmentFile=-/etc/sysconfig/sshd`（注意等号后面的那个连词号），就表示即使`/etc/sysconfig/sshd`文件不存在，也不会抛出错误。

#### 停止行为
**停止行为**
`KillMode`字段：定义 `Systemd` 如何停止 服务。
`KillMode`字段可以设置的值如下。
```text
control-group（默认值）：当前控制组里面的所有子进程，都会被杀掉
process：只杀主进程
mixed：主进程将收到 SIGTERM 信号，子进程收到 SIGKILL 信号
none：没有进程会被杀掉，只是执行服务的 stop 命令。
```

上面这个例子中，将`KillMode`设为`process`，表示只停止主进程，不停止任何`sshd` 子进程，即子进程打开的 `SSH session` 仍然保持连接。这个设置不太常见，但对 sshd 很重要，否则你停止服务的时候，会连自己打开的 `SSH session` 一起杀掉。


#### 重启行为
**重启行为**
`Restart`字段：定义了 服务 退出后，Systemd 的重启方式。
`Restart`字段可以设置的值如下。
```text
no（默认值）：退出后不会重启
on-success：只有正常退出时（退出状态码为0），才会重启
on-failure：非正常退出时（退出状态码非0），包括被信号终止和超时，才会重启
on-abnormal：只有被信号终止和超时，才会重启
on-abort：只有在收到没有捕捉到的信号终止时，才会重启
on-watchdog：超时退出，才会重启
always：不管是什么退出原因，总是重启
```
上面的例子中，`Restart`设为`on-failure`，表示任何意外的失败，就将重启sshd。如果 sshd 正常停止（比如执行`systemctl stop`命令），它就不会重启。
对于守护进程，推荐设为`on-failure`。对于那些允许发生错误退出的服务，可以设为`on-abnormal`。

>另外，`RestartSec`字段：表示 Systemd 重启服务之前，需要等待的秒数。上面的例子设为等待42秒。



#### 范例
```text
[Service]
ExecStart=/bin/echo execstart1
ExecStart=
ExecStart=/bin/echo execstart2
ExecStartPost=/bin/echo post1
ExecStartPost=/bin/echo post2
```
上面这个配置文件，第二行`ExecStart`设为空值，等于取消了第一行的设置，运行结果如下。
```bash
execstart2
post1
post2
```

下面是一个`oneshot`的例子，笔记本电脑启动时，要把触摸板关掉，配置文件可以这样写。
```bash
[Unit]
Description=Switch-off Touchpad

[Service]
Type=oneshot
ExecStart=/usr/bin/touchpad-off

[Install]
WantedBy=multi-user.target
```
上面的配置文件，启动类型设为`oneshot`，就表明这个服务只要运行一次就够了，不需要长期运行。

如果关闭以后，将来某个时候还想打开，配置文件修改如下。
```bash
[Unit]
Description=Switch-off Touchpad

[Service]
Type=oneshot
ExecStart=/usr/bin/touchpad-off start
ExecStop=/usr/bin/touchpad-off stop
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
上面配置文件中，`RemainAfterExit`字段设为`yes`，表示进程退出以后，服务仍然保持执行。这样的话，一旦使用`systemctl stop`命令停止服务，`ExecStop`指定的命令就会执行，从而重新开启触摸板。


### `[Install]`区块

`[Install]`通常是配置文件的最后一个区块，定义如何安装这个配置文件，用来定义如何启动，以及是否开机启动。它的主要字段如下。

```text
WantedBy：表示该服务所在的 Target。它的值是一个或多个 Target，当前 Unit 激活时（enable）符号链接会放入/etc/systemd/system目录下面以 Target 名 + .wants后缀构成的子目录中。

RequiredBy：它的值是一个或多个 Target，当前 Unit 激活时，符号链接会放入/etc/systemd/system目录下面以 Target 名 + .required后缀构成的子目录中。

Alias：当前 Unit 可用于启动的别名。

Also：当前 Unit 激活（enable）时，会被同时激活的其他 Unit
```

#### 范例
对于ssdh的配置文件中，`WantedBy=multi-user.target`指的是，sshd 所在的 Target 是`multi-user.target`。
这个设置非常重要，因为执行`systemctl enable sshd.service`命令时，`sshd.service`的一个符号链接，就会放在`/etc/systemd/system`目录下面的`multi-user.target.wants`子目录之中。
Systemd 有默认的启动 Target。
```bash
$ systemctl get-default
multi-user.target
```
上面的结果表示，默认的启动 Target 是`multi-user.target`。在这个组里的所有服务，都将开机启动。这就是为什么`systemctl enable`命令能设置开机启动的原因。

#### 其他
一般来说，常用的 Target 有两个：
一个是`multi-user.target`，表示多用户命令行状态；
另一个是`graphical.target`，表示图形用户状态，它依赖于`multi-user.target`。
官方文档有一张非常清晰的 [Target 依赖关系图](https://www.freedesktop.org/software/systemd/man/bootup.html#System%20Manager%20Bootup)。

```bash
# 查看 multi-user.target 包含的所有服务
$ systemctl list-dependencies multi-user.target

# 切换到另一个 target
# shutdown.target 就是关机状态
$ sudo systemctl isolate shutdown.target
```

## 自定义Service 单元
前面说过，Service 单元就是所要执行的任务。比如发送邮件就是一种 Service。
新建 Service 非常简单，就是在`/usr/lib/systemd/system`目录里面新建一个文件，比如`mytimer.service`文件，你可以写入下面的内容。

```bash
[Unit]
Description=MyTimer

[Service]
ExecStart=/bin/bash /path/to/mail.sh
```
可以看到，这个 Service 单元文件分成两个部分。

`[Unit]`部分介绍本单元的基本信息（即元数据），`Description`字段给出这个单元的简单介绍（名字叫做`MyTimer`）。

`[Service]`部分用来定制行为，Systemd 提供许多字段。
现在，启动这个 Service。
```bash
sudo systemctl start mytimer.service
```

## 自定义Timer 单元
Service 单元只是定义了如何执行任务，要定时执行这个 Service，还必须定义 Timer 单元。

`/usr/lib/systemd/system`目录里面，新建一个`mytimer.timer`文件，写入下面的内容。
```bash
[Unit]
Description=Runs mytimer every hour

[Timer]
OnUnitActiveSec=1h
Unit=mytimer.service

[Install]
WantedBy=multi-user.target
```
这个 Timer 单元文件分成几个部分。
`[Unit]`部分定义元数据。
`[Timer]`部分定制定时器。Systemd 提供以下一些字段。

```text
OnActiveSec：定时器生效后，多少时间开始执行任务
OnBootSec：系统启动后，多少时间开始执行任务
OnStartupSec：Systemd 进程启动后，多少时间开始执行任务
OnUnitActiveSec：该单元上次执行后，等多少时间再次执行
OnUnitInactiveSec： 定时器上次关闭后多少时间，再次执行
OnCalendar：基于绝对时间，而不是相对时间执行
AccuracySec：如果因为各种原因，任务必须推迟执行，推迟的最大秒数，默认是60秒
Unit：真正要执行的任务，默认是同名的带有.service后缀的单元
Persistent：如果设置了该字段，即使定时器到时没有启动，也会自动执行相应的单元
WakeSystem：如果系统休眠，是否自动唤醒系统
```
上面的脚本里面，`OnUnitActiveSec=1h`表示一小时执行一次任务。其他的写法还有`OnUnitActiveSec=*-*-* 02:00:00`表示每天凌晨两点执行，`OnUnitActiveSec=Mon *-*-* 02:00:00`表示每周一凌晨两点执行。

`mytimer.timer`文件里面，还有一个`[Install]`部分，定义开机自启动（`systemctl enable`）和关闭开机自启动（`systemctl disable`）这个单元时，所要执行的命令。

上面脚本中，`[Install]`部分只写了一个字段，即`WantedBy=multi-user.target`。它的意思是，如果执行了`systemctl enable mytimer.timer`（只要开机，定时器自动生效），那么该定时器归属于`multi-user.target`。
它背后的操作其实很简单，执行`systemctl enable mytimer.timer`命令时，就会在`multi-user.target.wants`目录里面创建一个符号链接，指向`mytimer.timer`。

### 相关命令
启动刚刚新建的这个定时器。你应该立刻就会收到邮件，然后每个小时都会收到同样邮件。
```bash
sudo systemctl start mytimer.timer
```

查看这个定时器的状态。
```bash
systemctl status mytimer.timer
```

查看所有正在运行的定时器。
```bash
systemctl list-timers
```

关闭这个定时器。
```bash
sudo systemctl stop myscript.timer
```

下次开机，自动运行这个定时器。
```bash
sudo systemctl enable myscript.timer
```

关闭定时器的开机自启动。
```bash
sudo systemctl disable myscript.timer
```

# systemd的 Target
## 介绍
启动计算机的时候，需要启动大量的 `Unit`。如果每一次启动，都要一一写明本次启动需要哪些 `Unit`，显然非常不方便。`Systemd` 的解决方案就是 `Target`。

简单说：
**`Target` 就是一个 `Unit` 组，包含许多相关的 Unit 。启动某个 Target 的时候，Systemd 就会启动里面所有的 Unit**。
从这个意义上说，`Target` 这个概念类似于"状态点"，启动某个 `Target` 就好比启动到某种状态。


## Target 的配置文件
Target 也有自己的配置文件。
```bash
$ systemctl cat multi-user.target

[Unit]
Description=Multi-User System
Documentation=man:systemd.special(7)
Requires=basic.target
Conflicts=rescue.service rescue.target
After=basic.target rescue.service rescue.target
AllowIsolate=yes
```
注意，Target 配置文件里面没有启动命令。
上面输出结果中，主要字段含义如下。
`Requires`字段：要求`basic.target`一起运行。
`Conflicts`字段：冲突字段。如果`rescue.service`或`rescue.target`正在运行，`multi-user.target`就不能运行，反之亦然。
`After`：表示`multi-user.target`在`basic.target` 、 `rescue.service`、 `rescue.target`之后启动，如果它们有启动的话。
`AllowIsolate`：允许使用`systemctl isolate`命令切换到`multi-user.target`。


## 操作
```bash
# 查看当前系统的所有 Target
$ systemctl list-unit-files --type=target

# 查看一个 Target 包含的所有 Unit
$ systemctl list-dependencies multi-user.target

# 查看启动时的默认 Target
$ systemctl get-default

# 设置启动时的默认 Target
$ sudo systemctl set-default multi-user.target

# 切换 Target 时，默认不关闭前一个 Target 启动的进程，
# systemctl isolate 命令改变这种行为，
# 关闭前一个 Target 里面所有不属于后一个 Target 的进程
$ sudo systemctl isolate multi-user.target
```

## `Target` 和 `init`进程比较
所谓 `Target` 指的是一组相关进程，有点像 `init` 进程模式下面的启动级别。
启动某个`Target` 的时候，属于这个 `Target` 的所有进程都会全部启动。

不同的是，`RunLevel` 是互斥的，不可能多个 `RunLevel` 同时启动，但是多个 `Target` 可以同时启动。

`Target` 与 传统 `RunLevel` 的对应关系如下。
```bash
Traditional runlevel      New target name     Symbolically linked to...

Runlevel 0           |    runlevel0.target -> poweroff.target
Runlevel 1           |    runlevel1.target -> rescue.target
Runlevel 2           |    runlevel2.target -> multi-user.target
Runlevel 3           |    runlevel3.target -> multi-user.target
Runlevel 4           |    runlevel4.target -> multi-user.target
Runlevel 5           |    runlevel5.target -> graphical.target
Runlevel 6           |    runlevel6.target -> reboot.target
```

它与`init`进程的主要差别如下。
**（1）默认的 RunLevel**（在`/etc/inittab`文件设置）现在被默认的 Target 取代，位置是`/etc/systemd/system/default.target`，通常符号链接到`graphical.target`（图形界面）或者`multi-user.target`（多用户命令行）。

**（2）启动脚本的位置**，以前是`/etc/init.d`目录，符号链接到不同的 RunLevel 目录 （比如`/etc/rc3.d`、`/etc/rc5.d`等），现在则存放在`/lib/systemd/system`和`/etc/systemd/system`目录。

**（3）配置文件的位置**，以前`init`进程的配置文件是`/etc/inittab`，各种服务的配置文件存放在`/etc/sysconfig`目录。现在的配置文件主要存放在`/lib/systemd`目录，在`/etc/systemd`目录里面的修改可以覆盖原始设置。


# systemd的日志管理
## 介绍
`Systemd` 统一管理所有 `Unit` 的启动日志。

**日志收集**：`systemd-journald`进程收集来**自系统、用户空间服务和内核**的日志消息。

**二进制日志格式**：与传统的文本日志不同，`journald` 使用二进制格式存储日志，这使得日志的读取和处理更加高效，同时也支持结构化数据(比如，以`json`格式输出)。

 **持久性和临时存储**：`journald` 可以配置为将日志存储在内存中（临时）或持久化到磁盘上。持久化日志存储可以通过配置 `/var/log/journal` 目录来实现。

**日志轮换和清理**：`journald` 具有自动轮换和清理日志的功能，可以根据配置的大小限制和时间限制自动删除旧日志，以节省存储空间。

**支持日志的元数据**：每条日志消息都可以附带元数据，例如时间戳、进程ID、用户ID、会话ID等，这些信息有助于更好地理解和分析日志。

**实时日志查看**：`journald` 提供了实时日志查看功能，可以使用 `journalctl` 命令来查看和过滤日志消息。这使得管理员能够快速定位问题。

**日志的配置文件**：`/etc/systemd/journald.conf`。

## 操作
`journalctl`功能强大，用法非常多。
```bash
# 查看所有日志（默认情况下 ，只保存本次启动的日志）
$ sudo journalctl

# 查看 mytimer.timer 的日志
$ sudo journalctl -u mytimer.timer

# 查看 mytimer.timer 和 mytimer.service 的日志
$ sudo journalctl -u mytimer


# 查看内核日志（不显示应用日志）
$ sudo journalctl -k

# 查看系统本次启动的日志
$ sudo journalctl -b
$ sudo journalctl -b -0

# 查看上一次启动的日志（需更改设置）
$ sudo journalctl -b -1

# 查看指定时间的日志
$ sudo journalctl --since="2012-10-30 18:17:16"
$ sudo journalctl --since "20 min ago"
$ sudo journalctl --since yesterday
$ sudo journalctl --since "2015-01-10" --until "2015-01-11 03:00"
$ sudo journalctl --since 09:00 --until "1 hour ago"

# 显示尾部的最新10行日志
$ sudo journalctl -n

# 显示尾部指定行数的日志
$ sudo journalctl -n 20

# 实时滚动显示最新日志
$ sudo journalctl -f

# 查看指定服务的日志
$ sudo journalctl /usr/lib/systemd/systemd

# 查看指定进程的日志
$ sudo journalctl _PID=1

# 查看某个路径的脚本的日志
$ sudo journalctl /usr/bin/bash

# 查看指定用户的日志
$ sudo journalctl _UID=33 --since today

# 查看某个 Unit 的日志
$ sudo journalctl -u nginx.service
$ sudo journalctl -u nginx.service --since today

# 实时滚动显示某个 Unit 的最新日志
$ sudo journalctl -u nginx.service -f

# 合并显示多个 Unit 的日志
$ journalctl -u nginx.service -u php-fpm.service --since today

# 查看指定优先级（及其以上级别）的日志，共有8级
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug
$ sudo journalctl -p err -b

# 日志默认分页输出，--no-pager 改为正常的标准输出
$ sudo journalctl --no-pager

# 以 JSON 格式（单行）输出
$ sudo journalctl -b -u nginx.service -o json

# 以 JSON 格式（多行）输出，可读性更好
$ sudo journalctl -b -u nginx.service -o json-pretty

# 显示日志占据的硬盘空间
$ sudo journalctl --disk-usage

# 指定日志文件占据的最大空间
$ sudo journalctl --vacuum-size=1G

# 指定日志文件保存多久
$ sudo journalctl --vacuum-time=1years
```

# systemd 的 notify
## 介绍
`systemd sd_notify`是什么？ 
传统的服务化进程管理，我们只是能知道他活着，他挂了.  
`sd_notify`可以做到一个运行程序跟`systemd`交互通信。 **服务主动告诉`systemd`我现在的状态和事件**。

## 配置
在 `systemd` 中，服务的配置通常在 `.service` 文件中进行。为了使用 `Notify` 类型，您需要在服务文件中设置 `Type` 字段为 `notify`。
```text
[Unit]
Description=My Example Service

[Service]
Type=notify
ExecStart=/usr/bin/my_service
# 其他配置项...

[Install]
WantedBy=multi-user.target

```
1. 当 `systemd` 启动服务时，它会等待服务进程发送通知。
2. 服务进程在启动完成后调用 `sd_notify` 来发送状态消息。
3. `systemd` 接收到通知后，会更新服务的状态，并根据通知内容执行相应的操作（如启动依赖服务、记录日志等）。
4. 如果在指定的时间内`systemd`没有收到 `READY=1` 的消息，`systemd` 就会认为启动失败。并终止该服务。

## 作用
在 `systemd` 中，`Notify` 类型的主要作用是允许服务在启动、运行和停止过程中向 `systemd` 发送状态更新。这种机制提供了更精细的进程管理和监控能力，具体作用包括如下。

### 状态报告

- **服务准备就绪**: 
服务可以在完成初始化后，通过发送 `READY=1` 通知 `systemd`，表明它已经准备好接受请求。这使得 `systemd` 可以有效地管理服务的依赖关系，例如在服务启动时自动启动其他依赖服务

- **状态更新**: 
服务可以通过 `STATUS=...` 发送当前状态信息，帮助系统管理员或监控工具更好地理解服务的运行状况。

- **监控和重启**: 
通过使用 `Notify`，服务可以在运行时报告其健康状态，`systemd` 可以根据这些状态信息决定是否需要重启服务或采取其他恢复措施。

### 进程管理

- **动态监控**：
通过发送通知，`systemd` 可以实时监控服务的状态。这使得管理者能够在服务运行时获得更多的上下文信息，例如服务是否正在重新加载配置，或是否正在停止。

- **失败处理**:
如果服务在启动过程中未能在预定时间内发送通知，`systemd` 会将其视为启动失败，并采取相应的措施（如重启服务或记录错误）。

### 资源管理

- **依赖管理**: 
`systemd` 可以根据收到的通知动态管理服务之间的依赖关系。例如，如果一个服务依赖于另一个服务的状态，`systemd` 可以确保在依赖服务准备好之前，不会启动依赖它的服务。

- **通知其他服务或系统组件**: 
服务可以通过 `sd_notify` 向 `systemd` 发送各种类型的事件通知，例如它正在重新加载配置或即将停止。这种机制可以帮助其他依赖于该服务的组件做出相应的调整。



## 原理
`systemd` 的 `Notify` 类型配置允许服务在启动时向 `systemd` 通知其状态，从而实现更好的进程管理和监控。通过使用 `sd_notify` 接口，服务可以灵活地报告其状态和事件，帮助 `systemd` 做出相应的管理决策。

 `sd_notify`函数 接口中，服务其实是通过环境变量 `NOTIFY_SOCKET`指定的`unix socket`的路径， 直接与 `systemd` 进行通信。

## sd_notify 函数

![](attachments/Pasted%20image%2020241221152756.png)

`sd_notify()`是`systemd`提供的一个函数，用于向`systemd`发送通知消息。它可以用于指示服务启动、停止、重启、状态变化等操作。当`systemd`收到通知消息时，它会更新服务状态，并向其他应用程序广播这些通知。

### 函数原型
```c
#include <systemd/sd-daemon.h>

int sd_notify(int unset_environment, const char *state);
```

如果 `unset_environment` 参数非零，`sd_notify()` 在返回之前将会取消设置 `$NOTIFY_SOCKET` 环境变量（无论函数调用本身是否成功）。后续对 `sd_notify()` 的调用将会失败，但该变量不再被子进程继承。

state：表示当前服务的状态，可以是以下值之一：
- READY=1：表示服务已经启动完成，可以接受客户端请求。
- STATUS=""：表示服务状态发生了变化，是一个字符串，说明服务的当前状态。
- RELOADING=1：表示服务正在重新加载配置。
- STOPPING=1：表示服务正在停止。
- WATCHDOG=1：表示服务正在进行自我监控。


### 使用
`sd_notify()`函数通常在服务启动时使用，在进入主循环之前调用，告知`systemd`服务已成功启动，可以接受请求。这样，`systemd`就知道了服务的状态，并可以监控服务的运行状态。
#### `Type=notify`的服务 使用`sd_notify()`函数
如上所诉。

#### `Type=forking`的服务 使用`sd_notify()`函数
在服务的配置文件（`xxx.service`）中配置的`service`的`Type=forking`的`systemd`管理的服务中，通常需要使用`sd_notify()`函数来通知`systemd`服务已经启动成功，并且获取主进程的`PID`，以用于`systemd`监控该服务的状态。
这是因为，使用`Type=forking`类型启动的服务，`systemd`无法直接得知服务的真正进程`ID`，而必须由服务本身通过`sd_notify()`函数通知`systemd`。

虽然在`Type=forking`的`systemd`管理的服务中可以不使用`sd_notify()`函数，但是这样会导致`systemd`无法监控到服务的启动状态，如果服务意外停止，`systemd`无法自动重启服务，从而增加了服务管理的难度。

因此，一般建议在`Type=forking`的`systemd`管理的服务中使用`sd_notify()`函数通知`systemd`进程已经启动完成。可以在服务启动时调用`sd_notify(0, "READY=1")`函数，将`READY`状态通知给`systemd`。如果服务需要重新加载配置，则可以在重新加载配置时使用`sd_notify(0, "RELOADING=1"`)通知`systemd`。

### 注意
需要注意的是，`sd_notify()`函数只有在`systemd`的控制下启动的服务中才有效，对于其他任意进程调用`sd_notify()`函数是没有任何作用的。


# systemd 的 path
## 介绍
systemd 的 path工具提供了**监控文件、目录变化并触发执行指定操作**的功能。
有时候这种监控功能是非常实用的，比如监控到`/etc/nginx/nginx.conf`或`/etc/nginx/conf.d/`发生变化后，立即 `reload nginx`。虽然，用户也可以使用**inotify**类的工具来监控，但远不如**systemd path**更方便、更简单且更易于观察监控效果和调试。
> 注意：其实，`systemd path`的底层使用的是`inotify`，所以受限于`inotify`的缺陷。

注：`systemd path`只能监控本地文件系统，而无法监控网络文件系统。

## 监控项
`systemd path`暴露的监控功能并不多，它能监控的动作包括：
![](attachments/Pasted%20image%2020231219162602.png)
> 注：这些指令监控的路径必须是绝对路径。

## 使用
要使用`systemd path`的功能，需至少编写两个文件，一个`.path`文件和一个`.service`文件，这两个文件的前缀名称通常保持一致，但并非必须。

这两个文件可以位于以下路径：
- `/usr/lib/systemd/system/`
- `/etc/systemd/system/`
- `~/.config/systemd/user/`：用户级监控，只在该用户登录后才监控，该用户所有会话都退出后停止监控。

## 范例

例如：
```bash
/usr/lib/systemd/system/test.path  
/usr/lib/systemd/system/test.service  
  
/etc/systemd/system/test.path  
/etc/systemd/system/test.service  
  
~/.config/systemd/user/test.path  
~/.config/systemd/user/test.service
```


有以下监控需求：

1. 监控/tmp/foo目录下的所有文件修改、创建、删除等操作
2. 如果被监控目录/tmp/foo不存在，则创建
3. 监控/tmp/a.log文件的更改
4. 监控/tmp/file.lock锁文件是否存在

为了简化，这些监控触发的事件都执行同一个操作：向/tmp/path.log中写入一行信息。
此处将path_test.path文件和path_test.service文件放在/etc/systemd/system/目录下。

path_test.path内容如下：
```bash
$ cat /etc/systemd/system/path_test.path  
[Unit]  
Description = monitor some files  
  
[Path]  
PathChanged = /tmp/foo  
PathModified = /tmp/a.log  
PathExists = /tmp/file.lock  
MakeDirectory = yes  
Unit = path_test.service  
  
# 如果不需要开机后就自动启动监控的话，可省略下面这段  
# 如果开机就监控，则加上这段，并执行systemctl enable path_test.path  
[Install]  
WantedBy = multi-user.target
```
其中MakeDirectory指令默认为no，当设置为yes时表示如果监控的目录不存在，则自动创建目录，但该指令对PathExists指令无效。

Unit指令表示该sysmted path实例监控到符合条件的事件时启动的服务单元，即要执行的对应操作。通常省略该指令，这时启动的服务名称和path实例的名称一致(除了后缀)，例如`path_test.path`默认启动的是`path_test.service`服务。

path_test.service内容如下：
```bash
$ cat /etc/systemd/system/path_test.service  
[Unit]  
Description = path_test.service  
  
[Service]  
ExecStart = /bin/bash -c 'echo file changed >>/tmp/path.log'
```

然后执行如下操作启动该systemd path实例：
```bash
systemctl daemon-reload  
systemctl start path_test.path
```

使用如下命令可以列出当前已启动的所有systemd path实例
```bash
$ systemctl --type=path list-units --no-pager  
UNIT                               LOAD   ACTIVE SUB     DESCRIPTION                                
systemd-ask-password-console.path  loaded active waiting Dispatch Password Requests to Console  
systemd-ask-password-wall.path     loaded active waiting Forward Password Requests to Wall Dir  
path_test.path                     loaded active waiting monitor some files
```

测试该systemd path能否如愿工作。
```bash
$ touch /tmp/foo/a  
$ touch /tmp/foo/a  
$ touch /tmp/a.log  
$ echo 'hello world' >>/tmp/a.log  
$ rm -rf /tmp/a.log  
...
```

如果想观察触发情况，可使用journalctl。例如：
```bash
$ journalctl -u path_test.service  
Jul 05 16:09:43 junmajinlong.com systemd[1]: Started path_test.service.  
Jul 05 16:09:45 junmajinlong.com systemd[1]: Started path_test.service.  
Jul 05 16:09:47 junmajinlong.com systemd[1]: Started path_test.service.  
Jul 05 16:09:49 junmajinlong.com systemd[1]: Started path_test.service.  
Jul 05 16:09:51 junmajinlong.com systemd[1]: Started path_test.service.  
Jul 05 16:09:55 junmajinlong.com systemd[1]: Started path_test.service.
```
## 资源控制
`systemd path`触发的任务可能会消耗大量资源，比如执行`rsync`的定时任务、执行数据库备份的定时任务，等等，它们可能会消耗网络带宽，消耗IO带宽，消耗CPU等资源。

想要控制这些定时任务的资源使用量也非常简单，因为真正执行任务的是`.service`，而`Service`配置文件中可以轻松地配置一些资源控制指令或直接使用`Slice`定义的`CGroup`。这些资源控制类的指令可参考`man systemd.resource-control`。

例如，直接在`[Service]`中定义资源控制指令：
```bash
[Service]  
Type=simple  
MemoryLimit=20M  
ExecStart=/usr/bin/backup.sh
```

又或者让`Service`使用定义好的`Slice`：
```bash
[Service]  
ExecStart=/usr/bin/backup.sh  
Slice=backup.slice
```
其中`backup.slice`的内容为：
```bash
$ cat /usr/lib/systemd/system/backup.slice  
[Unit]  
Description=Limited resources Slice  
DefaultDependencies=no  
Before=slices.target  
  
[Slice]  
CPUQuota=50%  
MemoryLimit=20M
```

## systemd path的`Bug`
注意，此中的`bug`是带引号的。

`systemd path`监控路径上所产生的事件是需要时间的，如果两个事件发生时的时间间隔太短，`systemd path`可能会丢失第二个甚至后续第三个第四个等等事件。

例如，使用`PathChanged`或`PathModified`监控路径`/tmp/foo`目录时，执行以下操作触发事件：
```bash
$ touch /tmp/foo/a && rm -rf /tmp/foo/a
```

期待的是`systemd path`能够捕获这两个事件并执行两次对应的操作，但实际上只会执行一次对应操作。换句话说，`systemd path`丢失了一次事件。

之所以会丢失事件，是因为`touch`产生的事件被`systemd path`捕获，`systemd path`立即启动对应`.service`服务做出对应操作，在本次操作还未执行完时，`rm`又立即产生了新的事件，于是`systemd path`再次启动服务，但此时老的服务尚未退出，所以本次启动的新的服务实际上什么事也不做。

所以，从结果上看去就像是`systemd path`丢失了事件，但实际上是因为服务尚未退出的情况下再次启动服务不会做任何事情。
可以加上一点休眠时间来耽搁一会：
```bash
$ touch /tmp/foo/a && sleep 0.1 && rm -rf /tmp/foo/a
```

上面的命令会成功执行两次对应操作。

再比如，将`.service`文件中的ExecStart设置为`/usr/bin/sleep 5`，那么在5秒内的所有操作，除了第一次触发的事件外，其它都会丢失。

systemd path的这个『bug』也有好处，因为可以让**瞬间产生的多个有关联关系的事件只执行单次任务**，从而避免了中间过程产生的事件也重复触发相关操作。

# Systemd服务环境变量缺失的问题
## 问题
在Linux系统运维中，我们可能会遇到在使用`systemd`管理的服务时无法获取系统环境变量，尤其是`PATH`变量，从而导致无法正确找到命令路径。这确实是一个常见的挑战。

参考：[# Linux: 解决Systemd服务环境变量缺失的问题](https://blog.csdn.net/qq_14829643/article/details/135613395)

## 原因
因为systemd启动的服务通常不会加载用户的环境变量。


## 解决方案
1. **通过systemd服务文件设置环境变量**
2. **使用脚本来设置环境并启动服务**
3. **全局设置环境变量**

### 服务文件中设置环境变量

1. 在 `xxx.service` 通过 `Environment="MY_VAR_1=VAR_1_VALUE"` 设置变量
2. 在 `xxx.service` 通过 `EnvironmentFile=/opt/workspace/my_env` 指定配置文件

**Environment方式**

编辑 `systemd` 的 `service` 文件，添加 `Environment=` 形如下：
```bash
[Service]
Environment="MY_VAR_1=VAR_1_VALUE"
Environment="MY_VAR_2=VAR_2_VALUE"
```
上述添加了两个环境变量：`MY_VAR_1=VAR_1_VALUE` 和 `MY_VAR_2=VAR_2_VALUE`
如需修改环境变量，即修改 service 文件，需要重新 reload， `systemctl daemon-reload`。
例如，如果我们知道需要的命令路径，可以直接在服务文件中设置`PATH`。
```bash
[Service]
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

```

这种方法的优点是直接且易于配置，但缺点是需要硬编码路径，这在路径不固定的情况下可能不理想。

**EnvironmentFile方式**
编辑 systemd 的 service 文件，添加 `EnvironmentFile=` 如下：
```bash
[Service]
EnvironmentFile=/opt/workspace/env_test.env
EnvironmentFile=-/opt/workspace/env_test_not_exist.env
```
上述指定了两个设置环境变量的文件：/opt/workspace/env_test.env 和 /opt/workspace/env_test_not_exist.env。
需要注意的是，第二个 EnvironmentFile 的路径前使用-，作用是忽略文件不存在。

创建 /opt/workspace/env_test.env 格式如下
```bash
MY_VAR_1=VAR_1_VALUE
MY_VAR_2=VAR_2_VALUE
```

### 使用脚本来设置环境并启动服务

另一种方法是编写一个包装脚本，在该脚本中设置所需的环境变量，然后启动服务。这样，当systemd启动服务时，它实际上是启动脚本。

创建一个脚本，例如`start-service.sh`：
```bash
#!/bin/bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
# 启动您的服务
exec /path/to/your/service
```

然后在systemd服务文件中引用这个脚本：
```bash
[Service]
ExecStart=/path/to/start-service.sh
```

### 全局设置环境变量

我们也可以考虑在系统级别设置环境变量，这样所有的服务和用户都可以访问这些变量。例如，可以在`/etc/environment`中设置`PATH`。
```bash
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```
这种方法的好处是它为整个系统提供了一致的环境，但可能会影响到所有用户和服务，有时这并不是所期望的。

### 加载/etc/profile合适吗

加载 /etc/profile 来为 systemd 服务设置环境变量是一个可行的解决方案，但需要小心处理。
**/etc/profile 是为交互式登录shell设计的，而不是为系统服务或非交互式shell环境设计的，是对于用户的SHELL初始化而言。因此，直接在 systemd 服务文件中加载 /etc/profile 可能不会按预期工作，也可能引入不必要的副作用**。

然而，如果我们确实需要从 /etc/profile 中提取特定的环境变量设置，可以创建一个包装脚本，该脚本首先加载 /etc/profile，然后启动我们的服务。这样做可以确保在启动服务之前设置了正确的环境变量。


#### 创建包装脚本
1. **创建脚本**：创建一个脚本，比如 `start-my-service.sh`。
2. **加载 `/etc/profile`**：在脚本中，首先执行 `source /etc/profile` 以加载环境变量。
3. **启动服务**：然后，执行服务启动命令。

```bash
#!/bin/bash
# 加载/etc/profile
source /etc/profile

# 启动您的服务
exec /path/to/your/service
```

#### 修改 systemd 服务文件
在systemd 服务文件中，将 `ExecStart` 指向前面的包装脚本。
```bash
[Service]
ExecStart=/path/to/start-my-service.sh
```

#### 注意事项

- 这种方法可能会比直接在服务文件中设置环境变量更复杂。
- 需要确保 `/etc/profile` 中的设置适用于我们的服务，并且不会干扰服务的正常运行。
- 某些在 `/etc/profile` 中设置的环境变量可能是为用户交互式会话设计的，不一定适合在后台服务中使用。


# chkconfig和systemctl命令对比

## systemctl 和 service、chkconfig的关系
**systemctl命令是系统服务管理器指令，它实际上将 service 和 chkconfig 这两个命令组合到一起。**
**systemctl是RHEL 7 的服务管理工具中主要的工具，它融合之前service和chkconfig的功能于一体。可以使用它永久性或只在当前会话中启用/禁用服务。**
**所以systemctl命令是service命令和chkconfig命令的集合和代替。**

例如：使用service启动服务实际上也是调用systemctl命令。
```c
[root@localhost ~]# service httpd start
Redirecting to /bin/systemctl start  httpd.service
```
![](attachments/Pasted%20image%2020230627121521.png)

# 参考
```c
# Systemd 定时器教程
https://www.ruanyifeng.com/blog/2018/03/systemd-timer.html

# Systemd 入门教程：实战篇
https://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-part-two.html

# Systemd 入门教程：命令篇
https://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html

# systemd服务配置文件编写(2)
https://www.junmajinlong.com/linux/systemd/service_2/
```