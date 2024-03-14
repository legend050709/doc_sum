```table-of-contents
```
# 介绍
如果说要给Linux文本三剑客(grep、sed、awk)添加一员的话，我觉得应该是jq命令，因为jq命令是用来处理json数据的工具，而现如今json几乎无所不在！

网上的jq命令分享文章也不少，但大多介绍得非常浅，jq的强大之处完全没有介绍出来.
![](attachments/Pasted%20image%2020231030165555.png)
# 安装
```c
yum install -y jq
```
# 使用
## 格式化
```bash
# jq默认的格式化输出
$ echo -n '{"id":1, "name":"zhangsan", "score":[75, 85, 90]}'|jq .
{
  "id": 1,
  "name": "zhangsan",
  "score": [
    75,
    85,
    90
  ]
}

# -c选项则是压缩到1行输出
$ jq -c . <<eof
{
  "id": 1,
  "name": "zhangsan",
  "score": [
    75,
    85,
    90
  ]
}
eof
{"id":1,"name":"zhangsan","score":[75,85,90]}

```

## 属性提取
```bash
# 获取id字段
$ echo -n '{"id":1, "name":"zhangsan", "score":[75, 85, 90]}'|jq '.id'
1
# 获取name字段
$ echo -n '{"id":1, "name":"zhangsan", "score":[75, 85, 90]}'|jq '.name'
"zhangsan"

# 获取name字段，-r 解开字符串引号
$ echo -n '{"id":1, "name":"zhangsan", "score":[75, 85, 90]}'|jq -r '.name'
zhangsan

# 多层属性值获取
$ echo -n '{"id":1, "name":"zhangsan", "attr":{"height":1.78,"weight":"60kg"}}'|jq '.attr.height'
1.78

# 获取数组中的值
$ echo -n '{"id":1, "name":"zhangsan", "score":[75, 85, 90]}'|jq -r '.score[0]'
75

$ echo -n '[75, 85, 90]'|jq -r '.[0]'
75

# 数组截取
$ echo -n '[75, 85, 90]'|jq -r '.[1:3]'
[
  85,
  90
]

# []展开数组
$ echo -n '[75, 85, 90]'|jq '.[]'
75
85
90

# ..展开所有结构
$ echo -n '{"id":1, "name":"zhangsan", "score":[75, 85, 90]}'|jq -c '..'
{"id":1,"name":"zhangsan","score":[75,85,90]}
1
"zhangsan"
[75,85,90]
75
85
90

# 从非对象类型中提取字段，会报错
$ echo -n '{"id":1, "name":"zhangsan", "attr":{"height":1.78,"weight":"60kg"}}'|jq '.name.alias'
jq: error (at <stdin>:0): Cannot index string with string "alias"

# 使用?号可以避免这种报错
$ echo -n '{"id":1, "name":"zhangsan", "attr":{"height":1.78,"weight":"60kg"}}'|jq '.name.alias?'

# //符号用于，当前面的表达式取不到值时，执行后面的表达式
$ echo -n '{"id":1, "name":"zhangsan", "attr":{"height":1.78,"weight":"60kg"}}'|jq '.alias//.name'
"zhangsan"

```


# 参考
```c
https://juejin.cn/post/7103145719141236743
```