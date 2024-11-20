```table-of-contents
```

# 包的命名
# main包
# 导入包
## 远程导入包
## 导入包重命名
## 导入包但不使用该包
## 包的init函数

# golang包管理的发展历程
## gopath
## govendor
## go mod
## go list
```bash
go list -m all 
查看当前项目正在使用的 package 版本
然后执行 `go get xxx/xxx` 来更新指定的 package
```
# go mod详解
## go mod原理
## go mod常用命令
### go mod init
### go mod tidy

# 参考
```bash
# Golang 工程管理与业务实践
https://wenzhiquan.github.io/2021/05/16/2021-05-16-golang-dependency/

# Go语言实战笔记（一）| Go包管理
https://www.flysnow.org/2017/03/04/go-in-action-go-package
```