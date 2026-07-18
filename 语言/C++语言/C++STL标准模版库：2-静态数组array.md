```table-of-contents
```
一、引言
在C++编程中，数组是一种非常基础且重要的数据结构，它可以存储一组相同类型的元素。在C++11标准之前，开发者通常使用C风格数组来处理数组数据，但C风格数组存在一些问题，比如无法直接进行对象赋值、无法直接拷贝、数组名会退化为指针等，这些问题给开发带来了一定的困扰。为了解决这些问题，C++11标准引入了`<array>`头文件，提供了std::array容器，它是对C风格数组的改进和封装，具有更多的功能和安全性。本文将详细介绍std::array的相关知识，帮助小白从入门到精通。

# std::array 简介

## 定义

`std::array`是C++11标准库提供的一个固定大小的容器，用于存储特定类型的元素序列。它定义在`<array>`头文件中，是一个模板类，其基本定义如下：
```C++
template <typename T, std::size_t N> 
class array;
```

其中，`T`表示数组中元素的类型，`N`表示数组的大小（即元素的个数）。

## 特点

- **固定大小**：`std::array`的大小在创建时就已经确定，并且之后不能再改变。这种固定大小的特性使得`std::array`在内存使用上是高效的，因为它不需要动态分配内存或管理内存大小的变化。同时，由于其大小在编译时就已知，编译器可以进行一些优化，提高代码的执行效率。

- **类型安全**：`std::array`强制类型检查，避免了C语言数组的类型不安全问题。它提供了`at()`函数来访问元素，同时提供边界检查以避免越界错误。

- **内存连续**：`std::array`的元素在内存中是连续存储的，这使得它可以高效地访问元素。

- **标准容器接口**：`std::array`提供了与`std::vector`类似的接口，如`size()`、`at()`、`front()`、`back()`等，使得对数组的操作更加便捷和安全。

## 与C风格数组的区别

|比较项|std::array|C风格数组|
|---|---|---|
|大小|固定大小，在编译时确定|大小可以是动态的，在运行时可以根据需要分配和调整|
|内存分配|对象通常在栈上分配，生命周期与创建它的作用域相同|可以在栈上或堆上分配，取决于数组的声明方式|
|安全性和边界检查|提供边界检查，使用`at()`函数访问元素时，如果索引超出范围，程序会抛出异常|不提供边界检查，程序员必须自己确保不会访问数组的非法区域|
|使用便捷性|提供了许多便利的成员函数和迭代器，操作更加简单和直观|操作相对较为基础，没有提供这些便利的函数和迭代器|
|与STL算法的兼容性|作为STL容器，与STL算法完美兼容|虽然也可以与一些STL算法结合使用，但通常需要额外的处理或转换|

# std::array的基本操作

## 包含头文件

要使用`std::array`，需要包含头文件`<array>`：
```C++
#include <array>
```

## 声明与初始化

### 声明

声明一个`std::array`对象时，需要指定数组中的元素类型和数组的大小。声明的基本语法如下：
```C++
std::array<T, N> array_name;
```
其中，`T`是数组中元素的类型，`N`是数组的大小，必须是一个非负整数。

### 初始化

- **默认值初始化**：如果不提供任何初始值，`std::array`中的元素将使用其类型的默认值进行初始化。
```C++
std::array<int, 5> myArray; // 所有元素将被初始化为随机值std::array<int, 5> myArray1 = {1, 2, 3, 4, 5};
std::array<int, 5> myArray2 {1, 2, 3, 4, 5}; std::array<int, 5> myArray;
std::fill(myArray.begin(), myArray.end(), 0); // 所有元素设置为0
```

- **列表初始化**：可以使用花括号`{}`内的元素列表来初始化`std::array`。元素的数量必须与数组大小相匹配。

- **使用`std::fill`**：可以使用`std::fill`函数来设置`std::array`中所有元素的值。

## 访问元素

可以使用下标操作符`[]`或者`at()`成员函数来访问`std::array`中的元素。`at()`函数会进行边界检查，如果索引越界，会抛出`std::out_of_range`异常。

```C++
#include <iostream>
#include <array>

int main() {
    std::array<int, 5> arr = {1, 2, 3, 4, 5};
    std::cout << "arr[0] = " << arr[0] << std::endl; // 使用下标操作符
    std::cout << "arr.at(2) = " << arr.at(2) << std::endl; // 使用at()函数

    try {
        std::cout << arr.at(5) << std::endl; // 超出范围，抛出异常
    } catch (const std::out_of_range& e) {
        std::cout << "Exception: " << e.what() << std::endl;
    }

    return 0;
}
```

## 遍历数组

`std::array`支持基于范围的for循环和迭代器，使得遍历元素变得非常方便。

### 基于范围的for循环

```C++
#include <iostream>
#include <array>

int main() {
    std::array<int, 5> arr = {1, 2, 3, 4, 5};
    for (const auto& elem : arr) {
        std::cout << elem << " ";
    }
    std::cout << std::endl;
    return 0;
}
```

### 迭代器遍历

```C++
#include <iostream>
#include <array>

int main() {
    std::array<int, 5> arr = {1, 2, 3, 4, 5};
    for (auto it = arr.begin(); it != arr.end(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;
    return 0;
}
```

## std::array的常用成员函数

### `size()`

返回数组的大小，即元素的数量。

```C++
#include <iostream>
#include <array>

int main() {
    std::array<int, 5> arr = {1, 2, 3, 4, 5};
    std::cout << "Array size: " << arr.size() << std::endl;
    return 0;
}
```

### `front()`和`back()`

`front()`返回数组的第一个元素的引用，`back()`返回数组的最后一个元素的引用。

```C++
#include <iostream>
#include <array>

int main() {
    std::array<int, 5> arr = {1, 2, 3, 4, 5};
    std::cout << "First element: " << arr.front() << std::endl;
    std::cout << "Last element: " << arr.back() << std::endl;
    return 0;
}
```

### `data()`

返回一个指向数组首个元素的指针。利用该指针，可以实现复制容器中所有元素等类似功能。
```C++
#include <iostream>
#include <array>

int main() {
    std::array<int, 5> arr = {1, 2, 3, 4, 5};
    int* ptr = arr.data();
    for (int i = 0; i < arr.size(); ++i) {
        std::cout << *(ptr + i) << " ";
    }
    std::cout << std::endl;
    return 0;
}
```


### `fill(const T& value)`

将`value`这个值赋值给容器中的每个元素。

```C++
#include <iostream>
#include <array>

int main() {
    std::array<int, 5> arr;
    arr.fill(10);
    for (const auto& elem : arr) {
        std::cout << elem << " ";
    }
    std::cout << std::endl;
    return 0;
}
```

## std::array的高级应用

### 与STL算法结合使用

由于`std::array`完全兼容STL（标准模板库），可以与其他STL算法和容器类无缝协作。例如，可以使用`std::sort`对数组进行排序。

```C++
#include <iostream>
#include <array>
#include <algorithm>

int main() {
    std::array<int, 5> arr = {5, 3, 1, 4, 2};
    std::sort(arr.begin(), arr.end());
    for (const auto& elem : arr) {
        std::cout << elem << " ";
    }
    std::cout << std::endl;
    return 0;
}
```

### 多维数组

可以使用`std::array`创建多维数组。例如，创建一个二维数组：

```C++
#include <iostream>
#include <array>

int main() {
    std::array<std::array<int, 3>, 2> arr = {
        { {1, 2, 3}, {4, 5, 6} }
    };
    for (const auto& row : arr) {
        for (const auto& elem : row) {
            std::cout << elem << " ";
        }
        std::cout << std::endl;
    }
    return 0;
}
```

# std::array的优缺点和适用场景
## 优点

- **类型安全**：`std::array`的大小是类型的一部分，因此不能创建大小不匹配的`std::array`对象，避免了一些潜在的错误。
- **性能**：由于大小在编译时已知，编译器可以对`std::array`进行更多的优化，提高代码的执行效率。
- **接口丰富**：`std::array`提供了许多有用的成员函数，如`fill()`、`swap()`等，这些函数让数组的操作更加便捷。

## 缺点

- **固定大小**：`std::array`的大小在编译时就已确定，无法动态增加或减少元素，不适合需要动态调整数组大小的场景。
- **初始化**：如果`std::array`在全局或命名空间作用域中定义，并且没有显式初始化，则所有元素都将被零初始化，这在某些情况下可能是不必要的或不方便的。

## 适用场景

当需要使用固定大小的数组，并且对性能和安全性有较高要求时，`std::array`是一个很好的选择。例如，在数学和科学计算中，表示向量和矩阵；在图形编程中，表示像素数据、纹理和颜色等。

# 总结
`std::array`是C++11引入的一个非常有用的容器，它结合了C风格数组的性能和可访问性以及容器的优点，提供了一种更安全、更灵活的方式来处理固定大小的数组。

# 参考
```bash
# C++11 <array>从入门到精通
https://cloud.tencent.com/developer/article/2533739
```