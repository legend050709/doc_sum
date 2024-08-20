```table-of-contents
```

# 介绍
# 应用
## 多个分隔符
### 方法一：多次awk
使用多次awk，第一次awk将想要输出的字段进行输出，第二个awk再次使用其他的分隔符进行分隔。
好处：每个awk就一个分隔符，这样每个要输出的字段`($n)`，非常清晰。如果在一个awk中使用多个分隔符，要输出的字段不清晰`($n 很容易表示错误）`;


```bash
date +%T.%N; ipvsadm -ln -u 118.121.194.195:4666 --fstats | grep "\-> 10" | awk '{print $2, $3}' | awk -F'[: ]' '{print $1,$3}' | awk 'BEGIN{sum[$1]=0}{sum[$1] += $2}END{for(i in sum) if(sum[i]!=0){printf "%-20s %d\n",i,sum[i]}}'; sleep 300; ipvsadm -ln -u 118.121.194.195:4666 --fstats | grep "\-> 10" | awk '{print $2, $3}' | awk -F'[: ]' '{print $1,$3}' | awk 'BEGIN{sum[$1]=0}{sum[$1] += $2}END{for(i in sum) if(sum[i]!=0){printf "%-20s %d\n",i,sum[i]}}'
```

### 方法二：


## awk使用shell变量 和 shell中使用awk变量

在写shell脚本时，经常会使用到awk程序。但是有些复杂的逻辑，可能需要在awk中使用在shell中定义的变量，而且awk程序处理之后，产生的中间变量，还需要在shell中继续处理。

### awk使用shell变量
#### 方法一
使用"'把shell变量包起来，即"'$var'"；注意是“双引号+单引号+shell变量+单引号+双引号”的格式。

这种写法大家无需改变用'括起awk程序的习惯,是老外常用的写法；这种写法其实际是双括号变为单括号的常量,传递给了awk.  
```bash
例如：
var="abc"
awk 'BEGIN{print "'$var'"}'
```

#### 方法二
**方法二：** 和方法一类似，但使用"'"把shell变量包起来，即"'"$var"'"；注意是“双引号+单引号+双引号+shell变量+双引号+单引号+双引号”的格式。

如果变量的值中包含空格，为了shell不把空格作为分隔符，则应使用方法二。
```bash
例如：  
var="this a test"
awk 'BEGIN{print "'"$var"'"}'
```

#### 方法三：export变量，awk使用使用环境变量

export变量，然后在awk中使用`ENVIRON["var"]`形式获取环境变量的值
```bash
例如：
var="this a test"; export var;  
awk 'BEGIN{print ENVIRON["var"]}'
```

#### 方法四：使用awk -v选项

当变量不是很多时，建议使用这种方式。  
```bash
例如：
var="this a test"  
awk -v awkVar="$var" 'BEGIN{print awkVar}'
```


#### 范例
**一个检测磁盘空间使用情况的脚本的例子**

```bash
#!/bin/sh
#文件系统名字
FILE_SYSTEM_NAME="rootfs"

#文件系统挂在的目录
MOUNTED_ON="/"

# shell命令使用awk中定义的变量spaceSize
eval $(df -P | awk '$1=="'"$FILE_SYSTEM_NAME"'" && $6=="'$MOUNTED_ON'" {printf("spaceSize=%s;",$5)}')
echo "主磁盘的使用空间为$spaceSize"

spaceSize=`echo $spaceSize | cut -d% -f1`
if [ aa$spaceSize = "aa" ]; then
    spaceSize=-1
fi

if [ $spaceSize -le 85 ]; then
    echo '主磁盘的使用空间充足'
elif [ $spaceSize -eq -1 ]; then
    echo '没有找到主磁盘使用空间，请检查脚本'
else
    echo '主磁盘的使用空间超过阈值'
fi
```

### awk使用shell中定义的数组


### shell中使用awk变量
#### awk 传递变量处理给shell
“由awk向shell传递变量”，其思想无非是用awk(sed/perl等也是一样)输出若干条shell命令，然后再用shell去执行这些命令。

```bash
例如：

eval $(awk 'BEGIN{print "var1='str1';var2='str2'"}')
或者eval $(awk '{printf("var1=%s; var2=%s; var3=%s;",$1,$2,$3)}' abc.txt)

之后可以在当前shell中使用var1,var2等变量了。
echo "var1=$var1 ----- var2=$var2"

```
#### awk 传递数组出来给shell
首先申明一下数组 `declare -A xxxx`；然后再使用awk 输出若干条给shell数组赋值的命令；再用shell去执行这些命令。

##### 范例一：awk 多行求和

```bash
date +%T.%N; declare -A sum1; eval $(ipvsadm -ln -u 118.121.194.195:4666 --fstats | grep "\-> 10" | awk '{print $2, $3}' | awk -F'[: ]' '{print $1,$3}' | awk 'BEGIN{sum[$1]=0}{sum[$1] += $2}END{for(i in sum) if(sum[i]!=0){printf "sum1[%s]=%d;",i,sum[i]}}'); sleep 300; declare -A sum2; eval $(ipvsadm -ln -u 118.121.194.195:4666 --fstats | grep "\-> 10" | awk '{print $2, $3}' | awk -F'[: ]' '{print $1,$3}' | awk 'BEGIN{sum[$1]=0}{sum[$1] += $2}END{for(i in sum) if(sum[i]!=0){printf "sum2[%s]=%d;",i,sum[i]}}'); for bb1 in "${!sum1[@]}"; do echo $bb1 ${sum1[$bb1]}; done; date; for bb1 in "${!sum2[@]}"; do echo $bb1 ${sum2[$bb1]}; done; for bb in "${!sum1[@]}"; do echo $bb diff=$((${sum2[$bb]} - ${sum1[$bb]})); done;

注意：上面的 eval() 中 awk的 `printf` 输出变量赋值中使用了双引号，即 "xxx=xxx;"
```

说明：
```bash
for bb1 in "${!sum1[@]}" // 遍历索引数组的每一个下标；
declare -A sum1 // 申明一个数组
```

##### 范例二：ethtool统计累计值的差值
正常输出的累计值如下所示：
```bash
# ethtool -S eth02 | grep -v ": 0" | grep -i -e xdp -e drop -e disc -e out -e err -e full -e pci -e wake -e xsk -e stop -e pause | awk -F: '{print $1,$2}' | awk '{print $1,$2}'
rx_xdp_drop 26835566565
rx_xdp_redirect 19825095281
tx_queue_stopped 8479939
tx_queue_wake 8479939
rx_xsk_xdp_redirect 2839393509
rx_xsk_buff_alloc_err 163580331
tx_xsk_xmit 2838931268
tx_xsk_full 303171057
tx_xsk_cqes 48351037
rx_out_of_buffer 2396568440
tx_pause_ctrl_phy 1224554
rx_discards_phy 6510045870
tx_pci_signal_integrity 7
outbound_pci_stalled_wr_events 362
rx_prio0_discards 6510045870
tx_global_pause 1224554
tx_global_pause_duration 657121418
rx0_xdp_drop 17042264150
rx0_xdp_redirect 4179441741
rx0_xsk_xdp_redirect 2839393509
rx0_xsk_buff_alloc_err 163580331
rx1_xdp_drop 987107407
rx1_xdp_redirect 1489697536
rx2_xdp_drop 855849518
rx2_xdp_redirect 1585203229
rx3_xdp_drop 889268809
rx3_xdp_redirect 1721918804
rx4_xdp_drop 866127425
rx4_xdp_redirect 1675922651
rx5_xdp_drop 862589327
rx5_xdp_redirect 1568384560
rx6_xdp_drop 678926129
rx6_xdp_redirect 1375556544
rx7_xdp_drop 772185628
rx7_xdp_redirect 1301017364
rx8_xdp_drop 611932527
rx8_xdp_redirect 828089089
rx9_xdp_drop 444358330
rx9_xdp_redirect 820054091
rx10_xdp_drop 839127444
rx10_xdp_redirect 730749083
rx11_xdp_drop 651551365
rx11_xdp_redirect 835357700
rx12_xdp_drop 406896929
rx12_xdp_redirect 924071263
rx13_xdp_drop 927382630
rx13_xdp_redirect 789631719
tx0_stopped 34839
tx0_wake 34839
tx1_stopped 5
tx1_wake 5
tx3_stopped 3169312
tx3_wake 3169312
tx4_stopped 3689271
tx4_wake 3689271
tx5_stopped 776691
tx5_wake 776691
tx6_stopped 703902
tx6_wake 703902
tx7_stopped 79106
tx7_wake 79106
tx10_stopped 137
tx10_wake 137
tx11_stopped 7942
tx11_wake 7942
tx12_stopped 1745
tx12_wake 1745
tx13_stopped 16989
tx13_wake 16989
tx0_xsk_xmit 2838931268
tx0_xsk_full 303171057
tx0_xsk_cqes 48351037
```



计算Mellanox网卡的一些异常统计的pps，如下所示：
```bash
sleep_time=1
declare -A allinfo1
declare -A allinfo2
while true
do
	echo ------------------$(date)----------------------
	eval $(ethtool -S eth02 | grep -v ": 0" | grep -i -e xdp -e drop -e disc -e out -e err -e full -e pci -e wake -e xsk -e stop -e pause | awk -F: '{print $1,$2}' | awk '{print $1,$2}' | awk  '{printf "allinfo1[%s]=%d;",$1,$2'})
       	sleep ${sleep_time}
       	for key1 in "${!allinfo1[@]}"
       	do
		diff=$((allinfo1[$key1] - allinfo2[$key1]))
		echo $key1 : ${diff}
		allinfo2[$key1]=${allinfo1[$key1]}
       	done
done
```

结果如下：
```bash
------------------Wed Aug 7 08:47:10 PM CST 2024----------------------
rx8_xdp_redirect : 0
tx0_wake : 20
rx_xdp_drop : 5328032
tx0_xsk_xmit : 0
tx0_xsk_full : 0
rx_xsk_buff_alloc_err : 0
tx3_wake : 0
rx11_xdp_redirect : 0
rx0_xdp_redirect : 272319
rx4_xdp_drop : 0
tx10_stopped : 0
rx12_xdp_redirect : 0
rx3_xdp_redirect : 0
rx1_xdp_drop : 0
tx_global_pause : 0
rx1_xdp_redirect : 0
tx7_wake : 0
tx13_wake : 0
tx1_wake : 0
tx10_wake : 0
tx_pci_signal_integrity : 0
tx6_wake : 0
tx11_wake : 0
rx2_xdp_redirect : 0
tx13_stopped : 0
tx4_wake : 0
tx12_wake : 0
rx0_xdp_drop : 5329752
outbound_pci_stalled_wr_events : 0
rx13_xdp_drop : 0
tx_queue_stopped : 20
rx_xdp_redirect : 272576
tx1_stopped : 0
rx13_xdp_redirect : 0
rx3_xdp_drop : 0
rx7_xdp_redirect : 0
tx0_xsk_cqes : 0
tx_pause_ctrl_phy : 0
rx9_xdp_redirect : 0
tx_queue_wake : 20
tx_xsk_xmit : 0
rx10_xdp_redirect : 0
rx6_xdp_drop : 0
rx11_xdp_drop : 0
rx7_xdp_drop : 0
rx10_xdp_drop : 0
rx_out_of_buffer : 2917461
rx5_xdp_redirect : 0
tx_global_pause_duration : 0
rx12_xdp_drop : 0
rx_discards_phy : 0
tx11_stopped : 0
tx6_stopped : 0
rx0_xsk_xdp_redirect : 0
rx2_xdp_drop : 0
tx0_stopped : 20
rx9_xdp_drop : 0
rx5_xdp_drop : 0
tx_xsk_full : 0
tx5_wake : 0
tx12_stopped : 0
tx3_stopped : 0
rx8_xdp_drop : 0
rx0_xsk_buff_alloc_err : 0
tx_xsk_cqes : 0
tx7_stopped : 0
rx6_xdp_redirect : 0
tx5_stopped : 0
tx4_stopped : 0
rx4_xdp_redirect : 0
rx_prio0_discards : 0
rx_xsk_xdp_redirect : 0
------------------Wed Aug 7 08:47:11 PM CST 2024----------------------
rx8_xdp_redirect : 0
tx0_wake : 20
rx_xdp_drop : 5326916
tx0_xsk_xmit : 0
tx0_xsk_full : 0
rx_xsk_buff_alloc_err : 0
tx3_wake : 0
rx11_xdp_redirect : 0
rx0_xdp_redirect : 272416
rx4_xdp_drop : 0
tx10_stopped : 0
rx12_xdp_redirect : 0
rx3_xdp_redirect : 0
rx1_xdp_drop : 0
tx_global_pause : 0
rx1_xdp_redirect : 0
tx7_wake : 0
tx13_wake : 0
tx1_wake : 0
tx10_wake : 0
tx_pci_signal_integrity : 0
tx6_wake : 0
tx11_wake : 0
rx2_xdp_redirect : 0
tx13_stopped : 0
tx4_wake : 0
tx12_wake : 0
rx0_xdp_drop : 5325880
outbound_pci_stalled_wr_events : 0
rx13_xdp_drop : 0
tx_queue_stopped : 20
rx_xdp_redirect : 272178
tx1_stopped : 0
rx13_xdp_redirect : 0
rx3_xdp_drop : 0
rx7_xdp_redirect : 0
tx0_xsk_cqes : 0
tx_pause_ctrl_phy : 0
rx9_xdp_redirect : 0
tx_queue_wake : 20
tx_xsk_xmit : 0
rx10_xdp_redirect : 0
rx6_xdp_drop : 0
rx11_xdp_drop : 0
rx7_xdp_drop : 0
rx10_xdp_drop : 0
rx_out_of_buffer : 2916515
rx5_xdp_redirect : 0
tx_global_pause_duration : 0
rx12_xdp_drop : 0
rx_discards_phy : 0
tx11_stopped : 0
tx6_stopped : 0
rx0_xsk_xdp_redirect : 0
rx2_xdp_drop : 0
tx0_stopped : 20
rx9_xdp_drop : 0
rx5_xdp_drop : 0
tx_xsk_full : 0
tx5_wake : 0
tx12_stopped : 0
tx3_stopped : 0
rx8_xdp_drop : 0
rx0_xsk_buff_alloc_err : 0
tx_xsk_cqes : 0
tx7_stopped : 0
rx6_xdp_redirect : 0
tx5_stopped : 0
tx4_stopped : 0
rx4_xdp_redirect : 0
rx_prio0_discards : 0
rx_xsk_xdp_redirect : 0
```


# 参考
```bash

```