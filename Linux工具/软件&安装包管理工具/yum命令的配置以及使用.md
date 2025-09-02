```table-of-contents
```
# yum仓库
## 查看
### 查看当前的仓库
#### `yum repolist`
```bash
yum repolist

# 查看启用仓库
yum repolist enabled

# 查看系统中所有的yum仓库
yum repolist all
```

## 管理仓库
```bash
# which yum-config-manager
/bin/yum-config-manager

# rpm -qf /bin/yum-config-manager
yum-utils-1.1.31-54.el7_8.noarch

# yum install yum-utils -y

```



### 关闭、启动仓库

#### `yum-config-manager --enable/--disable`

```bash
1. 启用一个yum仓库：
	yum-config-manager --enable xxxx
	
1. 关闭一个yum仓库：
	yum-config-manager --disable  xxxx
```

### 查看仓库的软件列表
#### `yum list`

![](attachments/Pasted%20image%2020250619195339.png)

```bash
yum list

# 仓库是否存在某个软件
yum list | grep xxxx
```



## 缓存
更新了 仓库的信息，则需要清空缓存，重新建立缓存，然后再次下载。

1、需要在`/etc/yum.repos.d`目录下配置`yum`源地址
2、清空缓存建立新的缓存
3、安装软件(自动解决依赖关系)

### 清空缓存
```bash
yum clean all
```
### 建立缓存
```bash
yum makecache
```


# yum 使用
## yum install

### 指定 rpm名称安装
```bash
yum install -y xxx
```
### 指定url路径安装
```bash
yum install https://example.com/path/to/package.rpm
```
在这个命令中，将 `https://example.com/path/to/package.rpm` 替换为实际的 HTTPS 链接，指向要安装的软件包的位置。这样可以直接从指定的 HTTPS 地址下载并安装软件包。

## yum remove
```bash
yum remove -y xxx
```
## yum update
```bash
yum update xxx
yum reinstall xxx
```
## 查看
### 查看是否安装
```bash
yum list installed | grep xxxx
```
### 查询仓库是否存在某个rpm
```bash
yum search xxx

# 仓库是否存在某个软件
yum list | grep xxxx
```

![](attachments/Pasted%20image%2020240827165200.png)

### 查看rpm的默认版本以及仓库信息
```bash
yum info xxx
```

![](attachments/Pasted%20image%2020240827165118.png)


### 范例
```bash
yum list available rdma-core
yum list available rdma-core --showduplicates
yum list installed |grep rdma-core
```
![](attachments/Pasted%20image%2020250619200230.png)

# 其他
## repoquery 命令
### 介绍

### 安装
```bash
# which repoquery
/usr/bin/repoquery

# rpm -qf /usr/bin/repoquery
yum-utils-1.1.31-40.el7.noarch

# yum install yum-utils -y
```


### `repoquery --whatrequires`
### `repoquery --whatprovides`

# 参考
```bash

```