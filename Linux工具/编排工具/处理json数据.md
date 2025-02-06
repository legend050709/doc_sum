```table-of-contents
```
# jq：格式化为json格式
## 介绍
如果说要给Linux文本三剑客(grep、sed、awk)添加一员的话，我觉得应该是jq命令，因为jq命令是用来处理json数据的工具，而现如今json几乎无所不在！

网上的jq命令分享文章也不少，但大多介绍得非常浅，jq的强大之处完全没有介绍出来.
![](attachments/Pasted%20image%2020231030165555.png)
## 安装
```c
yum install -y jq
```
## 使用
### 格式化
```bash
# jq默认的格式化输出
$ echo -n '{"id":1, "name":"zhangsan", "score":[75, 85, 90]}'| jq .
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

### 属性提取
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

#### json文件中提取某个list字段

json格式的数据，如下所示：
```json
# cat data.json
{
    "code": 0,
    "message": "",
    "poolInfos": [
        {
            "count": 52061,
            "eltSize": 2432,
            "flags": 16,
            "hdrSize": 64,
            "name": "eth_mbuf_pool_0_0",
            "privSize": 64,
            "size": 65536,
            "tailSize": 0
        },
        {
            "count": 51948,
            "eltSize": 2432,
            "flags": 16,
            "hdrSize": 64,
            "name": "eth_mbuf_pool_0_1",
            "privSize": 64,
            "size": 65536,
            "tailSize": 0
        },
        {
            "count": 63223,
            "eltSize": 2432,
            "flags": 16,
            "hdrSize": 64,
            "name": "kni_mbuf_pool_0",
            "privSize": 64,
            "size": 65536,
            "tailSize": 0
        }
    ],
    "ringInfos": [],
    "segInfos": [],
    "zoneInfos": []
}
```

**目标**：
想要对某个list 字段，进行提取，然后转换为更加可读的格式，比如`table`格式。
那么，首先要提取该字段，提取该字段的方法，如下所示：
```bash
# cat data.json | jq ".poolInfos"
[
  {
    "count": 52061,
    "eltSize": 2432,
    "flags": 16,
    "hdrSize": 64,
    "name": "eth_mbuf_pool_0_0",
    "privSize": 64,
    "size": 65536,
    "tailSize": 0
  },
  {
    "count": 51948,
    "eltSize": 2432,
    "flags": 16,
    "hdrSize": 64,
    "name": "eth_mbuf_pool_0_1",
    "privSize": 64,
    "size": 65536,
    "tailSize": 0
  },
  {
    "count": 63223,
    "eltSize": 2432,
    "flags": 16,
    "hdrSize": 64,
    "name": "kni_mbuf_pool_0",
    "privSize": 64,
    "size": 65536,
    "tailSize": 0
  }
]
```

进一步优化为：
```bash
cat data.json | jq ".poolInfos" | python3 json_to_table.py - | column -t 
```

# json格式转换为更加可读的表格
## 背景

## 将json转换为 Pandas DataFrame
### 背景
`json`格式进行格式化很好，但是可读性并不好。如果`json`文件中存在较多的数据,并不能够一目了然的进行读取。

如下所示，json的数据就不太好读取。
```json
# cat data.json
{
    "poolInfos": [
        {
            "count": 52061,
            "eltSize": 2432,
            "flags": 16,
            "hdrSize": 64,
            "name": "eth_mbuf_pool_0_0",
            "privSize": 64,
            "size": 65536,
            "tailSize": 0
        },
        {
            "count": 51948,
            "eltSize": 2432,
            "flags": 16,
            "hdrSize": 64,
            "name": "eth_mbuf_pool_0_1",
            "privSize": 64,
            "size": 65536,
            "tailSize": 0
        },
        {
            "count": 63223,
            "eltSize": 2432,
            "flags": 16,
            "hdrSize": 64,
            "name": "kni_mbuf_pool_0",
            "privSize": 64,
            "size": 65536,
            "tailSize": 0
        },
        {
            "count": 63223,
            "eltSize": 2432,
            "flags": 16,
            "hdrSize": 64,
            "name": "kni_mbuf_pool_1",
            "privSize": 64,
            "size": 65536,
            "tailSize": 0
        },
        {
            "count": 1024,
            "eltSize": 1408,
            "flags": 16,
            "hdrSize": 64,
            "name": "cdev_mbuf_pool_0",
            "privSize": 64,
            "size": 1024,
            "tailSize": 0
        },
        {
            "count": 1009,
            "eltSize": 1408,
            "flags": 16,
            "hdrSize": 64,
            "name": "cdev_mbuf_pool_1",
            "privSize": 64,
            "size": 1024,
            "tailSize": 0
        },
        {
            "count": 65536,
            "eltSize": 2432,
            "flags": 16,
            "hdrSize": 64,
            "name": "lcore_pool_name_17",
            "privSize": 64,
            "size": 65536,
            "tailSize": 0
        },
        {
            "count": 1023,
            "eltSize": 65920,
            "flags": 16,
            "hdrSize": 64,
            "name": "sa_pool_17",
            "privSize": 0,
            "size": 1024,
            "tailSize": 0
        },
        {
            "count": 4096,
            "eltSize": 8200,
            "flags": 16,
            "hdrSize": 64,
            "name": "sa_grp_pool_17",
            "privSize": 0,
            "size": 4096,
            "tailSize": 120
        }
  ]
}
```

### 分析
如果可以将`json`格式转换为其他的格式(比如，一个表格的格式)。
使用  `pandas` 工具，可以将 `json 的 list` 中的每个元素输出在一行。

### 执行

```python
# cc.py
# -*- coding: utf-8 -*-

import pandas as pd
import json
import sys

if sys.stdin.isatty():
    # 如果没有标准输入，则从文件读取
    with open(sys.argv[1], 'r', encoding='utf-8') as f:
        data = json.load(f)
else:
    # 从标准输入读取 JSON 数据
    data = json.load(sys.stdin)

# 转换为 DataFrame
df = pd.DataFrame(data)

# 打印为表格格式
print(df.to_string(index=False))

```

```bash
pip3 install pandas
```

解析脚本：
```python3
# cat json_to_table.py
import pandas as pd
import json
import sys


if sys.stdin.isatty():
	# 如果返回 `True`：表示没有通过管道或重定向提供输入，脚本将尝试从命令行参数中读取文件名（即 `sys.argv[1]`）
    # 如果没有标准输入，则从文件读取
    with open(sys.argv[1], 'r', encoding='utf-8') as f:
        data = json.load(f)
else:
    # 从标准输入读取 JSON 数据
    data = json.load(sys.stdin)

# 转换为 DataFrame
df = pd.DataFrame(data)

# 打印为表格格式
print(df.to_string(index=False))
```


执行的效果，如下所示：
```bash
# python3 json_to_table.py data.json
```

![](attachments/Pasted%20image%2020250124112713.png)

```bash
# python3 json_to_table.py data.json | column -t
```
![](attachments/Pasted%20image%2020250124112814.png)

### 小结

如果是 curl的输出是json格式，那么一条命令，将需要的字段，转换为更加可读的格式。如下所示：
```bash
curl -s -X POST http://localhost:6667/manage/GetDpdkMemInfo -d '{"name": "pool"}' | jq ".poolInfos" | python3  json_to_table.py  - | column -t
```

# 参考
```c
https://juejin.cn/post/7103145719141236743
```