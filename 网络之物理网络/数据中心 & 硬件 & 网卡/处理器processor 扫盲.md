```table-of-contents
```

# scalar and vector processor
标量处理器和向量处理器是两种计算机处理器的类型，它们之间的主要区别在于它们处理数据的方式。
A processor is an essential component of a computer system, responsible for carrying out instructions in order to facilitate various computer operations. 
Traditionally, processors have been either vector processors or scalar processors, both of which have their own unique set of benefits and drawbacks. 
Vector processors are designed to complete multiple data operations in one instruction and can provide higher performance than scalar processors.
Conversely, scalar processors are designed to carry out one instruction at a time and are more efficient for certain types of operations. In this article, we will discuss the differences between vector processors and scalar processors and how they are used in modern computing.

## vector processor
**Vector—矢量或者向量**：有大小有方向。
向量处理器是一种处理器，它可以同时处理多个数据元素。
也就是说，它可以对一组向量数据进行操作，例如一组浮点数或整数。
向量处理器在同一时刻可以执行多个指令，每个指令可以操作多个数据元素。
向量处理器通常有一个非常长的数据寄存器，用于存储向量数据。向量处理器在操作过程中可以在寄存器中读取一组数据，进行并行处理后再将结果写回到寄存器中。

![](attachments/Pasted%20image%2020230920163416.png)
## scalar processor
**Scalar—标量**：标量其实就很简单了，它只有大小没有方向。

标量处理器是一种处理器，它一次只能处理一个数据元素。
也就是说，它每次只能处理一个单个的标量值，例如整数、浮点数或字符。
标量处理器主要用于处理单个数据的运算和控制流程。标量处理器在操作过程中需要不断地从存储器中读取数据，进行处理后再将结果写回存储器中。
![](attachments/Pasted%20image%2020230920163633.png)
## 区别与联系
因此，标量处理器和向量处理器的最大区别在于它们的数据处理方式。标量处理器一次只能处理一个标量数据，而向量处理器可以同时处理多个数据元素。

需要注意的是，现代的处理器通常是复合型的，也就是说它们既包含标量处理器，又包含向量处理器。
例如，Intel的Xeon和Core i7处理器中都包含有SSE向量单元，可以用于处理浮点向量运算。此外，还有一些专门的向量处理器如GPU，它们专门用于图像处理和计算密集型应用，可以大大加速计算速度。
![](attachments/Pasted%20image%2020230920170031.png)
# 参考
```c
https://www.geeksforgeeks.org/vector-processor-vs-scalar-processor/
```
