```table-of-contents
```
# 背景
`shell` 的输出作为 python的输入，通过 简单的`python`编程 对数据进行处理。

# 范例
如下所示，bash命令的输出是 json格式，想取`json`中的某一个字段的值，通过bash还是不太方便的，则通过 python 命令的方式获取该值。

```bash
dpip -s link show cpu --json | python -c 'import sys, json; json_obj=json.load(sys.stdin); print(json_obj[-1]["fpktburst"])'

```

查看的范例：
```bash
while true; do imiss_num=$(dpip -x link show wan  | grep rx_missed_errors | awk -F ":" '{print $2}'); cpu_outinfo=$(dpip -s link show cpu --json); fpkts=$(echo ${cpu_outinfo} |  python -c 'import sys, json; json_obj=json.load(sys.stdin); print(json_obj[-1]["fpktburst"])'); ipackets=$(echo ${cpu_outinfo} |  python -c 'import sys, json; json_obj=json.load(sys.stdin); print(json_obj[-1]["ipackets"])'); diff_imiss=$((imiss_num - pre_imiss_num)); diff_fpkts=$((fpkts - pre_fpkts)); diff_ipkts=$((ipackets - pre_ipackets)); echo "$(date +%T.%N), ipkts: ${diff_ipkts}; fpkts:${diff_fpkts};  imiss: ${diff_imiss}"; pre_imiss_num=${imiss_num}; pre_fpkts=${fpkts}; pre_ipackets=${ipackets}; sleep 1;  done
```
# 参考
```bash

```