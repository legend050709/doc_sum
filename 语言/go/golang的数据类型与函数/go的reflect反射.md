```table-of-contents
```
# 背景
空接口`interface{}` 虽然能保存任意的值，但也带来了一个问题：一个空的接口会隐藏值对应的类型，以及其中的所有的公开的方法。
因此只有我们知道具体的动态类型才能使用类型断言来访问内部的值；如果我们事先不知道空接口指向的值的具体类型，我们可能就束手无策了。

这个时候我们想要知道一个接口类型的变量具体是什么（什么类型），有什么能力（有哪些方法），就需要一面“镜子”能够反射（`reflect`）出这个变量的具体内容。在Go语言中也正好有这样的工具——`reflect`。

# 参考
```c
# 深入理解 interface和reflect
https://turling.me/2019/10/13/Golang-series-interface-and-reflect/

# 图解go反射实现原理
https://i6448038.github.io/2020/02/15/golang-reflection/

# 深入理解Golang的reflect原理
https://kunkkawu.com/archives/shen-ru-li-jie-golang-de-reflect-yuan-li
http://liangjf.top/2020/09/22/161.go-reflect%E5%8F%8D%E5%B0%84/

# 【Golang】反射的三大laws
http://www.randyfield.cn/post/2022-07-08-the-laws-of-reflect/
https://juejin.cn/post/7183132625580605498

# 深入理解 go reflect - 反射为什么慢
https://juejin.cn/post/7186859098661453884
```