# 数组类型
## 数组声明
### -a
### -A
## 命令生成数组
## 类型
### 索引为int类型
### 索引为string类型
# 添加
# 遍历
## 遍历数组的所有元素
## 遍历数组的所有下标
# 删除
## 删除数组
## 删除数组元素
# map的使用
## 声明
map在使用之前需要先声明，声明的方式如下
```bash
declare -A map_name
```
map需要先声明再使用。参数-A表示声明的变量是一个map。需要注意的是这里的A是大写的字母A。
## 使用方法
### 赋值
map的赋值有两种方式：
一种是直接给map赋值，如下：
```bash
map_name=(["foo"]="bar" ["hello"]="world")
```


另一种是使用下标给map添加`key-value`对
```bash
map_name["foo"]="bar"
map_name["hello"]="world"
```

### 遍历
输出所有的key：`echo ${!map_name[@]}`
输出所有的value：`echo ${map_name[@]}`
输出map长度：`echo ${#map_name[@]}`
遍历，根据key找到对应的value
```bash
for key in ${!map_name[*]};do
	echo ${map_name[$key]}
done
```


遍历所有的key
```bash
for key in ${!map_name[@]};do
	echo $key
done
```


遍历所有的value
```bash
for val in ${map_name[@]};do
	echo $val
done
```

# 实现二维数组
## 背景
shell不支持二维数组，但是还是可以通过简单的方式实现二维数组的功能 。

## 解决方法
### 方法一
思路就是用数组A1(行)里的值作为B系列(列)数组的变量名。
### 方法二
Linux shell 中二维数组定义二维数组格式是A=(‘element1 element2’ ‘element3 .... element4’)，以空格分隔。
遍历二维数组的方法就是利用两次一维数组的遍历方法。
## 范例
### 范例一
```bash
#!/bin/bash

A1=(B1 B2 B3)
B1=(B1v1 B1v2 B1v3 B1v4)
B2=(B2v1 B2v2 B2v3 B2v4)
B3=(B3v1 B3v2 B3v3 B3v4)
#循环方式输出B列数据
for A in ${A1[@]};do
  echo ${A}
  TMP=$A[@]   #这里的处理是关键
  echo "111:tmp:${TMP}:1111"
  TempB=(${!TMP})   #这里的处理是关键
  echo "222:tmpb:${TempB}:22222"
  for B in ${TempB[@]};do
    echo ${B}
  done
done

#下标方式输入B列数据

for A in ${A1[@]};do
  echo ${A}
  TMP=$A[@]   #这里的处理是关键
  echo "111:tmp:${TMP}:1111"
  TempB=(${!TMP})   #这里的处理是关键
  echo "222:tmpb:${TempB}:22222"
  echo ${TempB[0]} ${TempB[1]} ${TempB[2]} ${TempB[3]}
done
```
输出：
```text
B1
111:tmp:B1[@]:1111
222:tmpb:B1v1:22222
B1v1
B1v2
B1v3
B1v4
B2
111:tmp:B2[@]:1111
222:tmpb:B2v1:22222
B2v1
B2v2
B2v3
B2v4
B3
111:tmp:B3[@]:1111
222:tmpb:B3v1:22222
B3v1
B3v2
B3v3
B3v4
B1
111:tmp:B1[@]:1111
222:tmpb:B1v1:22222
B1v1 B1v2 B1v3 B1v4
B2
111:tmp:B2[@]:1111
222:tmpb:B2v1:22222
B2v1 B2v2 B2v3 B2v4
B3
111:tmp:B3[@]:1111
222:tmpb:B3v1:22222
B3v1 B3v2 B3v3 B3v4
```

### 范例二
```bash
array_2d=('1 2 3' '4 5 6' '7 8 9')

for array_1d in ${array_2d[@]}
do
    for var in ${array_1d[@]}
    do
        echo $var
    done
done
```

```bash
#!/bin/bash

# 思路：要读取二维数组，可以通过逐行读取并将每行分割为单个元素的方式来完成。

# 创建二维数组
my_array=(
  [0]="1 2 3"
  [1]="4 5 6"
  [2]="7 8 9"
)

# 读取二维数组
for ((i=0; i<${#my_array[@]}; i++)); do
  row=(${my_array[i]})

  for ((j=0; j<${#row[@]}; j++)); do
    echo "my_array[${i}][${j}] = ${row[j]}"
  done
done
```


```bash
#定义数组
v2=("11 22 33" "55 66 77")
 
##################################
 
#读取二维结果
tmpV2=(${v2[1]})  #读取第一维数据
echo +++++++${tmpV2[1]}
 
 
tmpV2[3]=ppp #添加一个元素
echo ———————${tmpV2[3]}
```
这里的关键就在于**tmpV2=(${v2[1]})**  
**${v2[1]}** 值为 **"55 66 77"**
而**(${v2[1]})**  加了括号相当于将**"55 66 77"**变成一维数组 **(55 66 77)**

# 参考
```c
# shell中map
http://imhuchao.com/2033.html
```
