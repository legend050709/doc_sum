```table-of-contents
```
# 介绍

# dkms命令
## 安装
```bash
# which dkms
/sbin/dkms

# rpm -qf /sbin/dkms
dkms-2.8.1-4.20200214git5ca628c.el7.noarch

# yum install -y dkms
```
## 配置文件`dkms.conf`
### 范例一
```bash
（1）查看包内文件
# rpm -qpl /tmp/uoa-dkms-1.0.3-1.x86_64.rpm
/usr/src/uoa-1.0.3/Makefile
/usr/src/uoa-1.0.3/dkms.conf
/usr/src/uoa-1.0.3/uoa.c
/usr/src/uoa-1.0.3/uoa.h
/usr/src/uoa-1.0.3/uoa_extra.h

（2）查看包的安装、卸载脚本
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

（3）查看 dkms.conf
# cat dkms.conf
PACKAGE_NAME="uoa"
PACKAGE_VERSION="1.0.3"
BUILT_MODULE_NAME[0]="uoa"
MAKE="make"
CLEAN="make clean"
DEST_MODULE_LOCATION[0]="/updates/dkms"
AUTOINSTALL="YES"

（4）查看 makefile
# cat Makefile
obj-m	+= uoa.o

ifeq ($(KERNDIR), )
KDIR	:= /lib/modules/$(shell uname -r)/build
else
KDIR	:= $(KERNDIR)
endif
PWD	:= $(shell pwd)

ccflags-y := -I$(src)/../../include

ifeq ($(DEBUG), 1)
ccflags-y += -g -O0
endif

all:
	$(MAKE) -n -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) modules clean

install:
	if [ -d "$(INSDIR)" ]; then \
		install -m 664 uoa.ko $(INSDIR)/uoa.ko; \
	fi
```





## 操作命令
### dkms add
### dkms build
### dkms install
### dkms status
### dkms remove

## 使用流程
### 获取内核模块源代码
内核模块源代码目录中必需的 `dkms.conf` 文件将指示 `DKMS` 应该如何编译内核模块。
### 使用 dkms.conf 构建内核模块
如果内核模块源代码中不含 `dkms.conf` 文件，则必须手动创建该文件。
#### 创建或修改 `dkms.conf` 文件
在提取的源代码目录中创建或修改 `dkms.conf` 文件。
```bash
$EDITOR dkms.conf

MAKE="make -C src/ KERNELDIR=/lib/modules/${kernelver}/build"
CLEAN="make -C src/ clean"
BUILT_MODULE_NAME=custom_module
BUILT_MODULE_LOCATION=src/
PACKAGE_NAME=custom_module
PACKAGE_VERSION=1.0
DEST_MODULE_LOCATION=/kernel/drivers/other
AUTOINSTALL=yes
```
#### 将内核模块源代码复制到 `/usr/src/` 目录中
```bash
sudo mkdir /usr/src/<PACKAGE_NAME>-<PACKAGE_VERSION>
sudo cp -Rv . /usr/src/<PACKAGE_NAME>-<PACKAGE_VERSION>

注：<PACKAGE_NAME> 和 <PACKAGE_VERSION> 必须与 `dkms.conf` 文件中的条目匹配。
```
#### 将内核模块添加到 DKMS 树中
将内核模块添加到 DKMS 树中，以便由 DKMS 跟踪。
```bash
dkms add -m <MODULE-NAME>
```
#### 使用 DKMS 构建内核模块
如果构建时遇到错误，您可能需要编辑 `dkms.conf` 文件。
```bash
dkms build -m <MODULE-NAME> -v <MODULE-VERSION>
```
#### DKMS 安装内核模块
```bash
dkms install -m <MODULE-NAME> -v <MODULE-VERSION>
```

### 加载内核模块
默认情况下，DKMS 会将模块 “in-tree” 安装在 `/lib/modules` 下，以便可以使用 **modprobe** 命令加载它们。
```bash
使用 modprobe 命令加载已安装的模块。
sudo modprobe <MODULE-NAME>

验证内核模块是否已加载。
lsmod | grep <MODULE-NAME>
```

## GCC版本选择
### 方法一: 直接更改软链接
```bash
ll /usr/bin/gcc
ll /usr/bin/cc

/usr/bin/gcc -v
/usr/bin/cc -v

将上面的文件，软连接到新的版本的 gcc。如下所示：
source /opt/rh/devtoolset-9/enable
mv /usr/bin/gcc /usr/bin/bak-gcc-4.8.5
mv /usr/bin/cc /usr/bin/cc.bak
ln -s /opt/rh/devtoolset-9/root/usr/bin/gcc /usr/bin/gcc
ln -s /opt/rh/devtoolset-9/root/usr/bin/gcc /usr/bin/cc
```

### 将新的GCC的路径放在PATH前面
```bash
查看原有路径：
echo $PATH

更新路径：
export PATH=xxxx:$PATH

检查：
gcc -v
which gcc
```


# 参考
```bash

```