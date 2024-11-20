```table-of-contents
```
# go doc
## 语法
```bash
go doc
go doc <pkg>
go doc <sym>[.<method>]
go doc [<pkg>].<sym>[.<method>]
go doc <pkg> <sym>[.<method>]
```
## 查看包

### 范例

#### go doc fmt

![](attachments/Pasted%20image%2020241028101736.png)



## 查看包函数

### 范例
#### go doc fmt.Println

![](attachments/Pasted%20image%2020241028101851.png)


## 构建本地的网页版官方手册
还可以构建本地的网页版官方手册，在断网的时候可以访问：
```bash
godoc -http=:6060
```
然后就可以在浏览器中通过`http://localhost:6060/`访问官方手册。

# 官方文档
官方文档地址：[golang pkg 官方文档](https://pkg.go.dev/)

如下所示，输入要查询的包或者包函数。
![](attachments/Pasted%20image%2020241028101357.png)

# 参考
```bash
```