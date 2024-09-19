```table-of-contents
```
# 应用
```bash
bpftool prog dump xlated id <prog_id>： 将某个id的BPF程序指令转换为汇编指令打印出来


```

![](attachments/Pasted%20image%2020240913144448.png)

如果基于`BTF`编译, 那么输出会包含从`BTF`中获取的源代码信息

![](attachments/Pasted%20image%2020240913144505.png)

```bash
`bpftool prog dump jited id <prog_id>`: 显示经过`JIT`编译之后的机器码

`bpftool btf dump id 5`: 可以打印`BTF`的`ID`
```

![](attachments/Pasted%20image%2020240913144550.png)

# 参考
```bash

```