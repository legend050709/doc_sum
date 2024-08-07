```table-of-contents
```
# 问题
## `Binary file xxx matches`问题
**问题**：
```
grep test app.log 
Binary file app.log matches


比如：
在 keepalived中 grep xxx
出现：Binary file keepalived.log matches 的 错误，实际无任何输出。

但是 less keepalived.log 后，在文件内部进行查找xxx, 是可以查找到指定的字符串的。
```

**原因**
grep 默认情况下不喜欢输出二进制数据（例如，输出二进制数据很可能会弄乱终端）;
有可能你的文件的确是一个text文件，但是某一部分文件内容由于数据损坏，而 是二进制格式。

![](attachments/Pasted%20image%2020240714154853.png)


**解决**
（1）方法一：
```bash
grep -a test XXX.log

- -a, --text equivalent to --binary-files=text，即让二进制文件等价于文本。
```

（2）方法二：
```bash
strings data | grep -i whatever
```

# 参考
```bash

```