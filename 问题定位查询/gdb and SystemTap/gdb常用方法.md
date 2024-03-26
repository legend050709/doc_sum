```table-of-contents
```
# 其他
## 非交互式设置与打印

**设置**
```bash

```

**打印**
```bash
gdb -p `pidof dpvs` -ex 'p /x dpvs_estats' -ex 'detach' -ex 'quit';

gdb -q --batch -ex 'p /x dpvs_estats' -p `pidof dpvs`
```
# 参考
```bash

```