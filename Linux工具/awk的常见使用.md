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

```bash
date +%T.%N; declare -A sum1; eval $(ipvsadm -ln -u 118.121.194.195:4666 --fstats | grep "\-> 10" | awk '{print $2, $3}' | awk -F'[: ]' '{print $1,$3}' | awk 'BEGIN{sum[$1]=0}{sum[$1] += $2}END{for(i in sum) if(sum[i]!=0){printf "sum1[%s]=%d;",i,sum[i]}}'); sleep 300; declare -A sum2; eval $(ipvsadm -ln -u 118.121.194.195:4666 --fstats | grep "\-> 10" | awk '{print $2, $3}' | awk -F'[: ]' '{print $1,$3}' | awk 'BEGIN{sum[$1]=0}{sum[$1] += $2}END{for(i in sum) if(sum[i]!=0){printf "sum2[%s]=%d;",i,sum[i]}}'); for bb1 in "${!sum1[@]}"; do echo $bb1 ${sum1[$bb1]}; done; date; for bb1 in "${!sum2[@]}"; do echo $bb1 ${sum2[$bb1]}; done; for bb in "${!sum1[@]}"; do echo $bb diff=$((${sum2[$bb]} - ${sum1[$bb]})); done;
```

说明：
```bash
for bb1 in "${!sum1[@]}" // 遍历索引数组的每一个下标；
declare -A sum1 // 申明一个数组
```


# 参考
```bash

```