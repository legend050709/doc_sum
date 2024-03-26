```table-of-contents
```
# 介绍
# 使用
# 范例
## 列出某个进程打开的所有文件
```
lsof -p 1190
```
## 列出某个用户打开的文件
```
sudo lsof -u cizixs
```
## 列出某个文件被哪些进程打开
```
lsof /path/to/file
```
## 列出访问某个目录的所有进程
```
lsof +d /path/to/dir/
```
这个命令并不会递归地去访问子目录，如果想做到这一点，可以使用 `+D`：
```
ls +D /var/log/apache/
```

## 列出某个命令使用的文件信息
```
lsof -c nginx
```

# 参考
```c

```