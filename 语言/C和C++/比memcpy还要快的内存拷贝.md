```table-of-contents
```
# 前言
朋友们有想过居然还有比`memcpy`更快的内存拷贝吗？
讲道理，在这之前我没想到过，我也一直觉得`memcpy`、`rte_memcpy` 就是最快的内存拷贝方法了。

> 注：dpdk中的 `rte_memcpy` 其实和 `memcpy`的实现原理接近，只不过进一步进行了优化。


# SIMD技术简介
SIMD(Single Instruction Multiple Data，单指令多数据)。就是一条指令并发处理多条数据。
![](attachments/Pasted%20image%2020240314102549.png)

Scalar Operation就是指的SISD(Single Instruction Single Data，单指令单数据)，这种方式完成上图所有C[i]的计算需要串行执行八次，因为每个时间点，CPU的一条指令只能执行一份数据。

`SIMD`，就是一次运算就可以得到上述`SISD`的多次运算结果，即一条指令可以并发执行多份数据，因此`SIMD`也称为**向量化计算**。

到底是什么奇技淫巧使得`SIMD`具有并发执行多份数据的能力呢？

其实就是CPU增加了专门用于向量化计算的向量寄存器，这些寄存器跟普通的寄存器不太一样，它们的位宽都比较大，比如有`128bit`，`256bit`，甚至`512bit`，也就是说这些寄存器可以分别一次存储`16byte`，`32byte`，`64byte`的数据。比如上图的加法运算，`SISD`一条指令只能完成一次两个8byte数据的加法运算。但是`SIMD`，一条指令就可以完成`a[0:7] + b[0:7] = c[0:7]`，两组数据的加法运算。

CPU除了增加**向量寄存器**，还为向量寄存器配套了专门的**向量运算指令集**，比如`Intel`的`MMX`，`SSE`(`MMX`的升级版)，`AVX`(`SSE`的升级版)指令集。
CPU运算时，识别到指令集命令，就会采用指令集对应的`SIMD`计算方法完成并发运算。Intel指令集查询链接：[http://kntan.top/#!=undefined](http://kntan.top/#!=undefined)

# memcpy_fast方法
带着memcpy是否还可以继续优化的疑问，一通搜索，真找到了采用SIMD技术的memcpy方法：memcpy_fast，链接：[https://github.com/skywind3000/FastMemcpy](https://github.com/skywind3000/FastMemcpy)

## SSE指令集实现的fast拷贝
**实现方法**：

1、使用`_mm_loadu_si128`指令，从`src + 0`的位置取走`128bit`，即16字节，然后依次类推，`src + 1`，...，直至`src + 7`，一共取走`16byte * 8=128byte`，取出的内容分别储存到向量寄存器`c0，c1，...，c7`；

2、使用`_mm_prefetch`实现数据预取，提前把数据从内存加载到`cache`，保证CPU对数据的快速读取；

3、使用`_mm_store_si128`指令，将`c0，c1，...，c7`寄存器的内容分别存储至目的地址`dst + 0， dst + 1，...， dst + 7`的八个位置。

**总结**：
利用向量指令集、向量寄存器、数据预取技术实现了每次`16byte`的并发，`128byte`的批次拷贝。

![](attachments/Pasted%20image%2020240314103315.png)


## AVX指令集实现的fast拷贝

**实现方法**：
与SSE指令集实现内存拷贝逻辑一致。
1、由AVX指令集的`_mm256_loadu_si256`，实现每次256byte的数据加载；
2、由AVX指令集的`_mm256_storeu_si256`，实现每次256byte数据的存储。

可以预料，当然是寄存器位宽越大，性能会越好，也就是从理论上说使用AVX指令集会比SSE指令集更快。
![](attachments/Pasted%20image%2020240314103433.png)


## memcpy VS memcpy_fast
我们一起来看看`memcpy`与使用了`SIMD`技术的`memcpy_fast`的性能对比吧。
直接将`memcpy_fast`源码下载后编译即可:
```bash
　　SSE指令集编译命令：gcc -O3 -msse2 FastMemcpy.c -o FastMemcpy
　　AVX指令集编译命令：gcc -O3 -mavx FastMemcpy_Avx.c -o FastMemcpy_Avx
```
### SSE指令集下性能结果对比
绿色框里，即内存拷贝在1MB以下时，特别是拷贝长度在（1024 ~ 1048576）bytes时，拷贝性能有显著提升。但是靠拷贝长度超过1MB时，memcpy_fast居然比memcpy更慢了，发生了什么？
![](attachments/Pasted%20image%2020240314103614.png)

继续查阅源码，发现在大于`2MB`时，与`2MB`长度以下的拷贝相比，采用了不同的SIMD拷贝指令。即在拷贝长度小于等于 `cachesize = 0x200000` 时，使用 **`_mm_store_si128`**进行数据存储；在大于`0x200000` 时，使用**`_mm_stream_si128`**进行数据存储。
![](attachments/Pasted%20image%2020240314104032.png)

我把大长度数据拷贝由**`_mm_stream_si128`**替换为中等长度数据拷贝指令**`_mm_store_si128`**后，`memcpy_fast`无论是中等长度，还是大长度的数据拷贝性能都比`memcpy`要好。
![](attachments/Pasted%20image%2020240314104316.png)

### AVX指令集下性能结果对比
同样，将`AVX`大长度数据拷贝也进行优化，将指令**`_mm256_stream_si256`**替换为**`_mm256_storeu_si256`**，`AVX`指令集的性能测试结果如下图7所示。
简单总结为两点：

1、图6和图7进行了充分说明，相同长度的数据拷贝，`AVX`确实比`SSE`性能更高；

2、拷贝长度在（`512 ~ 8388608`）bytes，`memcpy_fast`都比`memcpy`要提升一倍不止，有的长度，内存拷贝性能甚至提升了4倍！

![](attachments/Pasted%20image%2020240314104443.png)

# 结语
这种内存拷贝的性能提升，有什么好处呢？
想到一个场景，比如生产环境的网关设备（FW，VPN等等），内存拷贝的性能提升可以降低网关设备的流量处理时延，提升网络质量，从而进一步提高用户使用体验。

# 参考
```bash
# 比memcpy还要快的内存拷贝，了解一下？
https://www.cnblogs.com/t-bar/p/17262147.html
```