```table-of-contents
```

# 参数默认值

## ${2:-} 含义
在shell脚本中，`${2:-}`是参数扩展的一种形式，用于提供默认值。这里的数字`2`表示传递给脚本的第二个参数。如果第二个参数在脚本执行时未定义或为空，那么`-`后面的部分（即默认值）将会被使用。

例如，假设你有一个脚本如下：
```bash
#!/bin/bash
echo "Your name is: ${2:-John Doe}"
```

当你不提供第二个参数时，脚本会输出`Your name is: John Doe`。如果提供了第二个参数，比如`./script.sh arg1 Alice`，它将输出`Your name is: Alice`。

### 作用

这种语法非常有用，因为它允许你在不知道用户是否提供所有必需参数的情况下，为脚本提供合理的默认行为。


### 范例

```bash
#!/bin/bash
if [ -n "${2:-}" ]; then
    echo "Second argument is provided and non-empty: \$2"
else
    echo "No second argument provided or it's empty."
fi
```

结果：
```bash
如果运行`./script.sh arg1`，
它将输出`No second argument provided or it's empty.`。
如果运行`./script.sh arg1 some_value`，它将输出`Second argument is provided and non-empty: some_value`。
```

分析：
```bash
在shell脚本中，`if [ -n "${2:-}" ]`是一个条件测试，用于检查第二个命令行参数（`\$2`）是否非空。如果第二个参数未定义、为空或者未提供，`${2:-}`会使用空字符串作为默认值。`-n`标志用于测试其后的字符串是否非空。如果字符串长度大于零，`-n`测试返回真（true），否则返回假（false）。

所以，这个条件语句的意义是：

1. 如果用户在运行脚本时提供了第二个参数，那么检查这个参数的值是否非空。
2. 如果用户没有提供第二个参数，那么使用空字符串作为默认值，并检查这个默认值是否非空。
```

# 参考