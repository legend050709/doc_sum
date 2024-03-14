```table-of-contents
```
# plugin插件
## 介绍
插件是通过动态库的方式来扩展`named`的功能。这样`named`的主体代码就比较的简单和清洁，对于一些复杂的、定制化的特性，可以通过插件的形式来进行扩展。
当前的插件还比较少，正在开发更多的插件；
**当前仅仅支持`query`类型的插件，`query`类型的插件更改了 dns 域名查询的逻辑**。
未来可能会支持其他类型的插件。
当前的插件也只有1个，即`filter-aaaa.so`，`filter-aaaa.so`这个插件是将之前 `named`主体代码中的`filter-aaaa`特性给移除了，然后放入到插件中。

![](attachments/Pasted%20image%2020240124150625.png)
## 插件开发
![](attachments/Pasted%20image%2020240124152222.png)

# `filter-aaaa.so`插件理解
参考：[github isc-projects/bind9](https://github.com/isc-projects/bind9) 下的`bind9/bin/plugins` 目录下的 插件。如下所示：
![](attachments/Pasted%20image%2020240124161620.png)

## 介绍
![](attachments/Pasted%20image%2020240124162126.png)

## 配置
**配置位置**：
将插件的配置配置在`view`或者最顶层中。多数情况下，配置在`view`中。

**Grammar:** `plugin ( query ) <string> [ { <unspecified-text> } ]; // may occur multiple times`
**Blocks:** topmost, view
**Tags:** server

# 参考
```bash
# plugin query "filter-aaaa.so"
https://docs.oracle.com/cd/E88353_01/html/E72487/filter-aaaa-8.html

# plugin官方说明
https://bind9.readthedocs.io/en/v9.18.24/chapter4.html#plugins
```
