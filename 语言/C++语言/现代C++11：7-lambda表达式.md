```table-of-contents
```

# lambda 本质
在 C++ 中，Lambda 表达式并不是一个普通的函数指针，它本质上是一个**匿名函数对象**（仿函数）。


## 底层 `operator()` 的特性


Lambda 闭包类型内部的核心是 `operator()` 成员函数，它的行为受以下规则限制：

- **默认是 `const` 的**：默认情况下，`operator()` 带有 `const` 限定符。这意味着在 Lambda 内部，**无法修改按值捕获的外部变量**（因为它们是 `const` 成员变量）14。
    
- **`mutable` 关键字**：如果在 Lambda 声明中使用了 `mutable` 关键字，`operator()` 的 `const` 属性会被取消。此时你可以修改按值捕获的变量副本，但这依然不会影响外部的原始变量。




# lambda原理
## 函数对象(仿函数)
###  函数对象：`operator()`运算符的重载

# 参考
```bash
# 一文读懂C++11的Lambda表达式的原理与使用场景
https://zhuanlan.zhihu.com/p/717554993

# 《C++11》Lambda 匿名函数从入门到进阶 & 优缺点分析 & 示例
https://lizhuo.blog.csdn.net/article/details/144970127
```