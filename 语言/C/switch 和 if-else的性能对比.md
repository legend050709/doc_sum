```table-of-contents
```
# switch的语法
```c
switch (表达式)  
{  
    case 常量表达式1：    语句1  
    case 常量表达式2：    语句2  
       ┇  
    case 常量表达式n：    语句n  
    default:        语句n+1  
}
```
说明：  
1) switch 后面括号内的“表达式”必须是整数类型。也就是说可以是 int 型变量、char 型变量，也可以直接是整数或字符常量，哪怕是负数都可以。但绝对不可以是实数，float 型变量、double 型变量、小数常量通通不行，全部都是语法错误。  
  
2) switch 下的 case 和 default 必须用一对大括号`{}`括起来。
3) case 后面必须是“常量表达式”，不可以是变量。

# 原理
对于 switch 来说，他最终生成的字节码有两种形态：
一种是 tableswitch，
另一种是 lookupswitch。
决定最终生成的代码使用那种形态取决于 switch 的判断添加是否紧凑，例如到 case 是 1...2...3...4 这种依次递增的判断条件时，使用的是 tableswitch，
而像 case 是 1...33...55...22 这种非紧凑型的判断条件时则会使用 lookupswitch

## 性能对比
tableswitch 的存储结构类似于数组，是直接用索引获取元素的，所以整个查询的时间复杂度是 O(1)，这也意味着它的搜索速度非常快。

而执行 lookupswitch 时，会逐个进行分支比较或者使用二分法进行查询，因此查询时间复杂度是 O(log n)，**所以使用 lookupswitch 会比 tableswitch 慢**。

# 对比
## if-else  
只是单纯地一个接一个比较；if...else每个条件都计算一遍；

## switch  
使用了Binary Tree算法；绝大部分情况下switch会快一点，除非是if-else的第一个条件就为true。
编译器编译switch与编译if...else...不同。不管有多少case，都直接跳转，不需逐个比较查询；switch只计算一次值，然后都是test , jmp。
switch使用查找表的方式决定了case的条件必须是一个连续的常量。而if-else则可以灵活的多。

对于switch语句来说，起实际是使用一个跳转表实现分支结构，不需要一次进行比较每一个所需要的条件。进行比较的次数为1.
但是对于if…else语句来说：最少的比较次数为1。
跟switch相比，在时间方面，switch语句的执行速度比if else要快，但是在程序执行占用的空间方面，switch语句需要一张跳转表来维护。
这个跳转，表的本质是一个拥有标号的数组，需要额外的存储空间，if else语句的空间效率更好一点。
switch是一个很典型的空间换时间的例子。但是switch只能判断是一个指定值的数据，而不能对一个区间中的数据进行判断。这时候选择if…else语句是一个很好的选择。


## 小结
在选择分支较多时，选用switch…case结构可能会有更高的效率。
在case中的条件是连续数字或相隔不大时，编译器会使用表结构做优化，性能优于if-else。
但switch不足的地方在于case只能处理字符或者数字类型的常量，if…else结构更 加灵活一些。

# 测试
## switch-case
编译选项是O3, 汇编代码如下所示：
![](attachments/Pasted%20image%2020230918163929.png)
从图中可以看到，switch-case生成的汇编代码是使用的表结构，根据case里的1、2、3、4来拿到表结构的偏移量，进而拿到对应的值。这种使用表结构的switch-case效率高，但是有个问题，该switch-case使用表结构可能是因为case里的常量数字比较小，且连续，那如果是不连续的呢，假如有1、2、3、456、987，那还使用表结构岂不是非常浪费内存。

![](attachments/Pasted%20image%2020230918164011.png)
再看对应的汇编代码，完完全全变成了逐分支判断。

## if-else
![](attachments/Pasted%20image%2020230918164038.png)
可以看见，对应的汇编代码是逐分支判断。

再看条件是非连续随机数字的情况，如下图：
![](attachments/Pasted%20image%2020230918164119.png)
对应的汇编代码依旧是逐分支判断，这里可知，if-else可不管条件里面的数字是否连续，它就是不停的分支判断，没有任何优化。


# 参考
```c
https://juejin.cn/post/6844904151478960135#comment

https://www.zhihu.com/question/300975864/answer/524449404

https://bbs.huaweicloud.com/blogs/308576
```