```table-of-contents
```
# cpio命令
# rpm2cpio 命令
# rpm 命令

## 查看某个二进制文件所在的包
```bash
# which ethtool
/sbin/ethtool

# rpm -qf /sbin/ethtool
ethtool-4.8-1.el7.x86_64
```
## 查看包中的文件：`rpm -ql`
```bash
# rpm -qpl uoa-dkms-1.0.3-1.x86_64.rpm
/usr/src/uoa-1.0.3/Makefile
/usr/src/uoa-1.0.3/dkms.conf
/usr/src/uoa-1.0.3/uoa.c
/usr/src/uoa-1.0.3/uoa.h
/usr/src/uoa-1.0.3/uoa_extra.h
```
## 查看包关联的脚本：`rpm -q --scripts`
要查看某个 RPM 包中关联的脚本，可以使用 `rpm` 命令的 `-q` 选项与 `--scripts` 参数。这个命令可以显示与指定 RPM 包相关的所有脚本，包括安装、卸载、升级等脚本。
```bash
rpm -q --scripts <package-name>

<package-name>: 为某个已经安装的包的名称。
```

```bash
# yum install -y ./uoa-dkms-1.0.3-1.x86_64.rpm
# rpm -q --scripts uoa-dkms
postinstall scriptlet (using /bin/sh):
#!/bin/sh
set -e

DKMS_NAME=uoa
DKMS_PACKAGE_NAME=$DKMS_NAME-dkms
DKMS_VERSION=1.0.3

postinst_found=0

DKMS_POSTINST="/usr/lib/dkms/common.postinst"

if [ -f $DKMS_POSTINST ]; then
    $DKMS_POSTINST $DKMS_NAME $DKMS_VERSION /usr/share/$DKMS_PACKAGE_NAME "" ""
    postinst_found=1
fi

if [ "$postinst_found" -eq 0 ]; then
    echo "ERROR: DKMS version is too old and $DKMS_PACKAGE_NAME was not"
    echo "built with legacy DKMS support."
    echo "You must either rebuild $DKMS_PACKAGE_NAME with legacy postinst"
    echo "support or upgrade DKMS to a more current version."
    exit 1
fi
preuninstall scriptlet (using /bin/sh):
#!/bin/sh
set -e

DKMS_NAME=uoa
DKMS_VERSION=1.0.3

if [  "$(dkms status -m $DKMS_NAME -v $DKMS_VERSION)" ]; then
    dkms remove -m $DKMS_NAME -v $DKMS_VERSION --all
fi
```

## 查看安装包的依赖
### 已安装包的前向依赖和反向依赖
查看一个包依赖哪些其他包（前向依赖）：  
查看一个包被哪些其他包依赖（反向依赖）：  

```bash
# 查看前向依赖 
rpm -qR <installed-package-name> 
注意：`rpm -qR`, 需要该包已经安装。

如果未安装，可以先下载rpm文件然后使用下面的命令，查看前向依赖；
rpm -qpR <package_file>.rpm

# 查看反向依赖 
rpm -q --whatrequires <installed-package-name>
注：同样，这只能查询已安装的包。如果要查询仓库中的包，需要结合其他工具（如repoquery）。
```


### 未安装包的前向依赖和反向依赖
对于未安装的包，我们通常使用`yum`工具（需要配置好仓库）：
```bash
前向依赖：
yum deplist <package>

反向依赖：
yum install yum-utils -y
repoquery --whatrequires <package>
```

### 依赖递归树
```bash
# 递归查看所有依赖
repoquery --tree-requires <package-name>

# 递归查看被依赖关系
repoquery --tree-whatrequires <package-name>
```

# 查看命令的源码
##  查看二进制文件/命令的位置
```c
#whereis ls
/usr/bin/ls /usr/share/man/man1/ls.1.gz /usr/share/man/man1p/ls.1p.gz

```

## 查看对应的rpm包：`rpm -qf`
```c
#rpm -qf /usr/bin/ls
coreutils-8.22-24.el7_9.2.x86_64
```
## 下载rpm包源码
- yumdownloaders 方式下载rpm包源码
```c
#yum install yum-utils

#yumdownloader --source coreutils-8.22-24.el7_9.2.x86_64
```

- 手动下载rpm包源码
```c
https://pkgs.org/search/?q=coreutils
```

如上两种方式下载之后，得到 带有源码的 rpm文件：ccoreutils-8.22-24.el7.src.rpm。


## 查询源码的文件
```c
#rpm -qpl xxxx.rpm
```

## 提取rpm中的文件

### 提取文件到指定 目录

正常的查看`rpm`中的文件为：
`rpm -qpl xxxx.rpm`

正常的提取`rpm`中的文件为：
`rpm2cpio xxxx.rpm | cpio -div`

上诉方式，提取的文件，会直接在执行 `rpm2cpio`的路径的基础上，放入 `rpm -qpl xxxx.rpm` 中展示的路径文件。不太好查看。

如下所示，`rpm2cpio xxxx.rpm | cpio -div` 方式提取的文件，直接放入到了 `/usr/share/bnxt_en` 和  `/usr/src/bnxt_en-1.10.3.230.0.132.0` 目录。

![](attachments/Pasted%20image%2020240813111409.png)

如果想要提取文件到指定的目录，可以通过`-D dir`的方式，如下的命令：
```bash
make rpm_files_dir; 
cd rpm_files_dir
rpm2cpio bnxt_en-1.10.3.230.0.132.0-1dkms.noarch.rpm | cpio -div 
```

### 提取源码tar文件
```c
# rpm -qpl ccoreutils-8.22-24.el7.src.rpm | grep tar
coreutils-8.22.tar.xz

# rpm2cpio coreutils-8.22-24.el7.src.rpm |cpio -idv coreutils-8.22.tar.xz

# mkdir -p new_dir; tar xvf coreutils-8.22.tar.xz -C new_dir

```

# 其他
## libxxx.rpm 和  libxxx-devel.rpm 和 libxxx.src.rpm的关系

![](attachments/Pasted%20image%2020240719115019.png)

![](attachments/Pasted%20image%2020240719115221.png)

libxxx.rpm 和  libxxx-devel.rpm 结合使用，提供给外部开发者。其中提供了 lib库(一般还会存在libxxx.pc 文件，供 pkgconfig 发现，比如：/usr/lib64/pkgconfig/libbpf.pc )，以及.h 头文件，不存在.c的文件。

libxxx.src.rpm 则是存在 .h、.c 以及 makefile 的文件，可以看到源码实现；但是需要编译成lib库 + 头文件的形式，提供给外部使用。

# 参考
```c
https://juejin.cn/post/6981019721197944845
```