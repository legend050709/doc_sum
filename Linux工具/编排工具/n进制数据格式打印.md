```table-of-contents
```
# echo命令
## 介绍
**echo命令** 用于在shell中打印shell变量的值，或者直接输出指定的字符串。

## 使用
```bash
echo(选项)(参数)

-e：启用转义字符。 
-E: 不启用转义字符（默认） 
-n: 结尾不换行
```


在echo命令中使用了`-e`选项，则表示开启了转义。
```bash
	`\\`：打印反斜杠字符
	`\n`：打印换行。
	`\r`：打印回车。
	`\b`：退格键，也就是向左删除键。
	`\e`：escapse 键；
	`\0nnn`：按照八进制输出字符。其中0为数字零，nnn为三位八进制数。
	`\xhh`：按照十六进制输出字符。其中hh是2位十六进制数。
```

 
## 范例

十六进制数输出：
```bash
# echo -e -n "\x41\x42\x43\x44\x61" > text

# hexdump text
0000000 4241 4443 0061
0000005

# cat text
ABCDa
```

八进制数字输出：
```bash
[root@localhost ~ ] # echo -e "\0141\t\0142\t\0143\n\0144\t\0145\t\0146"
a  b  c
d  e  f
```

`-n` 参数演示：
```bash
[root@localhost ~]# echo "666 888"
666 888
[root@localhost ~]# echo -n "666 888"
666 888[root@localhost ~]#
```

# od 命令
## 介绍
`od - dump files in octal and other formats`
od命令 用于输出文件的八进制、十六进制或其它格式编码的字节，通常用于显示或查看文件中不能直接显示在终端的字符。

## 使用
```bash
- A/--address-radix=RADIX   # 设置偏移量的编码单位；指定地址基数，包括：  
	d 十进制  
	o 八进制（系统默认值）  
	x 十六进制

-t<TYPE>，--format=TYPE：指定输出格式，格式包括a、c、d、f、o、u和x，各含义如下：  
  c：可打印的字符或者反斜杠；  
  d[SIZE]：十进制，正负数都包含，SIZE字节组成一个十进制整数；  
  f[SIZE]：浮点，SIZE字节组成一个浮点数；  
  o[SIZE]：八进制，SIZE字节组成一个八进制数；  
  u[SIZE]：无符号十进制，只包含正数，SIZE字节组成一个无符号十进制整数；  
  x[SIZE]：十六进制，SIZE字节为单位以十六进制输出，即输出时一列包含SIZE字节。

-w, --width[=BYTES]         # 设置每一行的最大字数
-v, --output-duplicates     # 显示重复的数据
```


## 范例
```bash
[sogrey@bogon newDir3]$ cat test2.txt
123
23
212
[sogrey@bogon newDir3]$ od test2.txt # 以八进制显示
0000000 031061 005063 031462 031012 031061 000012
0000013
[sogrey@bogon newDir3]$ od -t c test2.txt # 以字符方式显示
0000000   1   2   3  \n   2   3  \n   2   1   2  \n
0000013
[sogrey@bogon newDir3]$ od -b test2.txt # 使用单字节八进制解释进行输出，注意左侧的默认地址格式为八字节
0000000 061 062 063 012 062 063 012 062 061 062 012
0000013
[sogrey@bogon newDir3]$ od -c test2.txt # 使用ASCII码进行输出，注意其中包括转义字符
0000000   1   2   3  \n   2   3  \n   2   1   2  \n
0000013
[sogrey@bogon newDir3]$ od -t d1 test2.txt # 使用单字节十进制进行解释
0000000   49   50   51   10   50   51   10   50   49   50   10
0000013
[sogrey@bogon newDir3]$ od -A d -c test2.txt # 设置地址格式为十进制
0000000   1   2   3  \n   2   3  \n   2   1   2  \n
0000011
[sogrey@bogon newDir3]$ od -A x -c test2.txt # 设置地址格式为十六进制
000000   1   2   3  \n   2   3  \n   2   1   2  \n
00000b
[sogrey@bogon newDir3]$ od -j 2 -c test2.txt # 跳过开始的两个字节
0000002   3  \n   2   3  \n   2   1   2  \n
0000013
[sogrey@bogon newDir3]$ od -N 2 -j 2 -c test2.txt # 跳过开始的两个字节，并且仅输出两个字节
0000002   3  \n
0000004
[sogrey@bogon newDir3]$ od -w1 -c test2.txt # 每行仅输出1个字节
0000000   1
0000001   2
0000002   3
0000003  \n
0000004   2
0000005   3
0000006  \n
0000007   2
0000010   1
0000011   2
0000012  \n
0000013
[sogrey@bogon newDir3]$ od -w2 -c test2.txt # 每行仅输出2个字节
0000000   1   2
0000002   3  \n
0000004   2   3
0000006  \n   2
0000010   1   2
0000012  \n
0000013
[sogrey@bogon newDir3]$ od -w3 -c test2.txt # 每行仅输出3个字节
0000000   1   2   3
0000003  \n   2   3
0000006  \n   2   1
0000011   2  \n
0000013
```

# hexdump 命令
## 介绍
`hexdump`命令是一个十六进制转储工具，它可以将文件或数据以十六进制和 ASCII 码的形式打印出来。
## 使用
![](attachments/Pasted%20image%2020240223144854.png)


以十六进制和ASCII格式查看指定文件内容：  
```bash
# cat test
"3DUf

# hexdump -C test
00000000  11 22 33 44 55 66                                 |."3DUf|
00000006
```

```bash
# hexdump -c test
0000000 021   "   3   D   U   f
0000006
```

# xxd命令
## 介绍
`xxd - make a hexdump or do the reverse.`
xxd命令可以为给定的标准输入或者文件做一次十六进制的输出，它也可以将十六进制输出转换为原来的二进制格式，即将任意文件转换为十六进制或二进制形式。

注：电脑上的文件本质上还是以二进制进行保存的。但是我们查看的时候，一般是通过16进制进行查看。
## 使用
```bash
$ xxd [options] [infile [outfile]]
```
![](attachments/Pasted%20image%2020240223151419.png)

## 范例
**查看文件的十六进制码**：
```bash
 gackle@machine:~$ echo 1111111 > 1.txt

 gackle@machine:~$ cat 1.txt
 1111111

 gackle@machine:~$ xxd 1.txt
 00000000: 3131 3131 3131 310a                      1111111.
```

**查看文件的二进制码**：
```bash
# xxd -b 1.txt
0000000: 00110001 00110001 00110001 00110001 00110001 00110001  111111
0000006: 00110001 00001010
```

**设置显示字节数**：
```bash
xdd -l 0x30 /path/to/test.txt
00000000: 7878 6420 6372 6561 7465 7320 6120 6865  xxd creates a he
00000010: 7820 6475 6d70 206f 6620 6120 6769 7665  x dump of a give
00000020: 6e20 6669 6c65 206f 7220 7374 616e 6461  n file or standa
```


# 进制转换

## printf 命令
### 16进制转换成10进制
```bash
printf %d 0xF
15
或者
echo $((16#F))
15

# 十六进制转十进制 
echo $[16#ff] 
echo $((16#FF))
在本例中，$((16#FF))表示将FF解释为十六进制，并输出对应的十进制值255。

printf %d 0xac
```

### 10进制转换成16进制
```bash
printf %x 15
f
或者
echo "obase=16;15"|bc
F
```

![](attachments/Pasted%20image%2020240808170152.png)

### 10进制转换成8进制
```bash
printf %o 9  
11

八进制转十进制
echo $[8#100]
```

### 二进制转换成10进制  
```bash
echo $((2#111))  
7

# 二进制转十进制 
echo $[2#1100] 
echo $((2#1100))
```

## bc命令
### 介绍
`bc - An arbitrary precision calculator language`   一个任意精度的计算器语言。通常在linux下当计算器用。通过命令行选项可以使用标准数学库。
支持的运算包括：
- + 加法
- - 减法
- * 乘法
- / 除法
- ^ 指数
- % 余数

### 使用
#### 与管道符结合进行计算
```bash
# echo “sqrt(100)” |bc  
10  

# echo “3^3” |bc  
27

# echo "15+5" | bc
20

# echo "1.212*3" | bc 
3.636
```

```shell
abc=11000000
echo "obase=10;ibase=2;$abc" | bc
执行结果为：192
```

#### 进制转换
格式为：echo "obase=16 ; ibase=2 ; number" | bc ，其中obase代表输出进制，ibase代表输入进制，number表示ibase进制对应的数字。
注意：为10时可不设置ibase obase的值，obase要尽量放在ibase前，因为ibase设置后，后面的数字都是以ibase的进制来换算的。同时16进制字母必须大写。
```bash
16进制转换为二进制；
# echo "ibase=16;obase=2;FFEE" | bc
1111111111101110

八进制转换为二进制：
# echo "ibase=8;obase=2;67"  |  bc
110111

二进制转化为十六进制：
# echo "obase=16;base=2;11001111"  |  bc
A7DD17 

# 同时转换2个数字
# echo "obase=16;ibase=2;11001111;0101100111001111"  |  bc
CF
59CF  

十进制转化为十六进制：
# echo "obase=16;ibase=10;192" | bc
C0
```

# 参考
```bash

```
