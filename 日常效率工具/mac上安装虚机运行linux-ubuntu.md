```table-of-contents
```
# 下载
## 安装虚拟机virtual box
[Downloads – Oracle VM VirtualBox](https://link.zhihu.com/?target=https%3A//www.virtualbox.org/wiki/Downloads)，在这个网址下载相应的virtual box，然后安装好就行，还需要下载扩展，也就是[Oracle VM VirtualBox Extension Pack](https://zhida.zhihu.com/search?content_id=239544112&content_type=Article&match_order=1&q=Oracle+VM+VirtualBox+Extension+Pack&zhida_source=entity) [Downloads – Oracle VM VirtualBox](https://link.zhihu.com/?target=https%3A//www.virtualbox.org/wiki/Downloads)，也在这个下载页面上。

安装好virtual box以后，打开，然后双击下载的Oracle_VM_VirtualBox_Extension_Pack-7.0.14.[vbox-extpack](https://zhida.zhihu.com/search?content_id=239544112&content_type=Article&match_order=1&q=vbox-extpack&zhida_source=entity)，安装好扩展。

## ubuntu的iso镜像下载
注意：由于mac的cpu芯片是M系列，基于arm的，那么就需要下载arm版本的ubuntu的iso镜像，不要下载其他版本的。
```bash
比如：
选择：ubuntu-25.10-live-server-arm64.iso 
	下载地址：https://mirrors.tuna.tsinghua.edu.cn/ubuntu-cdimage/ubuntu/releases/25.10/release/
	清华大学的镜像，国内下载比较快；
	
而不是：[ubuntu-25.10-live-server-amd64.iso](https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/25.10/ubuntu-25.10-live-server-amd64.iso "ubuntu-25.10-live-server-amd64.iso")
```

# 安装

下载好iso以后，就开始安装的。在virtual box的软件上面，同时按Ctrl+N的，或者点击右侧的新建按钮，名称配置到Ubuntu，虚拟光盘选择刚刚下载的*.iso文件，然后分配内存6G（多则多分配，少则少分配）、分配磁盘的60G（看实际情况来分配）。配置用户名和密码等，最后点击完成。

设置完：用户名密码，内存大小，cpu个数，磁盘大小，然后可以启动无人值守安装。



# 参考
```bash

```