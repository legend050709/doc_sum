```table-of-contents
```
# auto 
## auto关键字
**（1）C语言**： 
auto被解释为一个自动存储变量的关键字，也就是申明一块临时的变量内存
```C
auto double a=3.7;
```

**（2）C++ 98标准/C++03标准**：
同C语言的意思完全一样：auto被解释为一个自动存储变量的关键字，也就是申明一块临时的变量内存。
在C++98/03中，auto作为存储类说明符，用于标识变量的自动存储周期（默认行为，极少显式使用），几乎处于废弃状态。

**(3) C++ 11标准**:
当我们面对复杂的类型的时候，之前的C++程序必须明确的指明变量的类型才能继续编码，会降低开发效率，使得代码的可读性下降。
C++11 对 `auto` 关键字进行了重新定义，用于**类型自动推导**，简化代码书写。==它让**编译器**根据变量初始化表达式自动推导出变量的类 型，这一特性极大简化了复杂类型声明（如STL容器]迭代器、模板类型），提升了代码可读性与维护性，同时避免了手动指定类型可能导致的错误==。auto的自动类型推断发生在编译期，所以使用auto并不会造成程序运行时效率的降低。

```C++
auto x = 10;         // 推导为 int
auto y = 3.14;       // 推导为 double
auto z = "hello";    // 推导为 const char*
```

## auto（自动类型推导）原理

参考：[C++11 auto关键字：原理解析与使用指南](https://cloud.tencent.com/developer/article/2539922)

C++11中auto的类型推导基于**模板参数推导规则**（C++标准ISO/IEC 14882:2011 §7.1.6.4），编译器通过初始化表达式的类型反向推导变量类型。
其核心规则可分为三种场景：

### 按值推导（auto）
当变量声明为`auto var = expr`时，推导行为类似于模板函数的按值参数传递：
- **忽略顶层const/volatile限定符**：若expr为`const int`，var推导为`int`；
- **忽略引用语义**：若expr为`int&`，var推导为`int`（复制引用对象的值）；
- **数组/函数退化为指针**：若expr为数组名或函数名，推导为对应类型的指针。

```C++
const int ci = 10;
int& ri = ci;
auto a = ci;       // a: int（顶层const被忽略）
auto b = ri;       // b: int（引用语义被忽略）
int arr[3] = {1,2,3};
auto c = arr;      // c: int*（数组退化为指针）
void func() {}
auto d = func;     // d: void(*)()（函数退化为函数指针）
```

### 引用/指针推导（auto& / auto*）

当变量声明为`auto& var = expr`或`auto* var = expr`时，推导行为类似于模板函数的引用/指针参数传递：

- **保留底层const/volatile限定符**：若expr为`const int`，`auto& var`推导为`const int&`；
- **必须匹配引用/指针类型**：`auto*`要求expr为指针类型，否则编译错误；
- **数组/函数不退化**：若expr为数组名且声明为`auto&`，推导为数组类型。

```C++
const int ci = 10;
auto& ra = ci;     // ra: const int&（保留底层const）
auto* pa = &ci;    // pa: const int*（指针类型匹配）
int arr[3] = {1,2,3};
auto& arr_ref = arr; // arr_ref: int(&)[3]（数组类型，未退化）
```

用auto声明指针类型时，用auto和`auto*`没有任何区别，但用auto声明引用类型时必须加&。
> **注意：用auto声明引用时必须加&，否则创建的只是与实体类型相同的普通变量。**

```C++
#include <iostream>
using namespace std;
int main()
{
	int a = 10;
	auto b = &a;   //自动推导出b的类型为int*
	auto* c = &a;  //自动推导出c的类型为int*
	auto& d = a;   //自动推导出d的类型为int
	//打印变量b,c,d的类型
	cout << typeid(b).name() << endl;//打印结果为int*
	cout << typeid(c).name() << endl;//打印结果为int*
	cout << typeid(d).name() << endl;//打印结果为int
	return 0;
}
```


### 万能引用推导（auto&&）

C++11引入的`auto&&`（万能引用）结合了左值引用与右值引用的特性，推导规则遵循**引用折叠**：

- 若expr为**左值**（如变量名、左值引用），`auto&&`推导为**左值引用**；
- 若expr为**右值**（如字面量、临时对象），`auto&&`推导为**右值引用**。

```C++
int x = 5;
auto&& r1 = x;     // r1: int&（x为左值，推导为左值引用）
auto&& r2 = 10;    // r2: int&&（10为右值，推导为右值引用）
auto&& r3 = r1;    // r3: int&（r1为左值引用，折叠为左值引用）
```

### 多变量声明的一致性要求

同一声明语句中，多个auto变量的推导类型必须一致，否则编译错误：
```C++
auto a = 1, b = 2;       // 正确：a和b均为int
auto c = 1, d = 2.0;     // 错误：c推导为int，d推导为double，类型冲突
```

## 使用场景
auto的价值在处理**复杂类型**和**泛型编程**时尤为突出，以下是典型应用场景：

### 简化STL容器迭代器声明
传统迭代器声明冗长（如`std::vector<std::map<int, std::string>>::iterator`），auto可大幅精简代码：

```C++
std::vector<std::map<int, std::string>> data;
// 传统写法
std::vector<std::map<int, std::string>>::iterator it = data.begin();
// auto简化
auto it = data.begin(); // 推导为正确的迭代器类型
```

### 模板函数中依赖参数的类型推导
当模板函数的变量类型依赖于模板参数时，auto可避免手动推导复杂类型：
```C++
template <typename T, typename U>
void process(T t, U u) {
    auto result = t + u; // 推导t+u的类型（如int+double→double）
    // ... 使用result
}
```

### 与for-range循环结合

若是在C++98中我们要遍历一个数组，可以按照以下方式：
```C++
	int arr[] = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
	//将数组元素值全部乘以2
	for (int i = 0; i < sizeof(arr) / sizeof(arr[0]); i++)
	{
		arr[i] *= 2;
	}
	//打印数组中的所有元素
	for (int i = 0; i < sizeof(arr) / sizeof(arr[0]); i++)
	{
		cout << arr[i] << " ";
	}
	cout << endl;
```

以上方式也是我们C语言中所用的遍历数组的方式，但对于一个有范围的集合而言，循环是多余的，有时还容易犯错。因此C++11中引入了基于范围的for循环。for循环后的括号由冒号分为两部分：第一部分是范围内用于迭代的变量，第二部分则表示被迭代的范围。
```C++
	int arr[] = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
	//将数组元素值全部乘以2
	for (auto& e : arr)
	{
		e *= 2;
	}
	//打印数组中的所有元素
	for (auto e : arr)
	{
		cout << e << " ";
	}
	cout << endl;

```
注意：与普通循环类似，可用continue来结束本次循环，也可以用break来跳出整个循环。

**C++11范围for(for-range)循环中，auto可自动匹配容器元素类型，简化迭代**：
```C++
std::vector<int> nums = {1, 2, 3, 4};
for (auto num : nums) {       // num: int（值拷贝）
    std::cout << num << " ";
}
for (auto& num : nums) {      // num: int&（引用，可修改元素）
    num *= 2;
}
```

## 注意事项
尽管auto提升了开发效率，但滥用或误用可能导致隐蔽错误，需严格遵循以下规则：

### auto 变量必须在定义时初始化;  
auto变量的类型完全依赖初始化表达式，**未初始化的auto声明会直接编译错误**：
```C++
auto a; // 错误：无法推导类型（无初始化表达式）
auto b = 0; // 正确：b推导为int
```

### 避免丢失cv（const/volatile）限定符与引用语义

按值推导时，auto会忽略顶层const和引用，若需保留需显式声明：
```C++
const int ci = 10;
auto x = ci;       // x: int（丢失const）
const auto y = ci; // y: const int（显式保留const）
int& ri = x;
auto z = ri;       // z: int（丢失引用）
auto& rz = ri;     // rz: int&（显式保留引用）
```

### 数组与函数名的特殊处理

- **数组**：直接使用auto推导数组名时，结果为指针；若需保留数组类型，需声明为引用：
```C++
int arr3 = {1,2,3};
auto arr_ptr = arr; // arr_ptr: int*（数组退化）
auto& arr_ref = arr; // arr_ref: int(&)3（数组类型）
```

**auto不能直接用来声明数组**
```C++
int main()
{
	int a[] = { 1, 2, 3 };
	auto b[] = { 4, 5, 6 };//error
	return 0;
}
```


- **函数**：直接使用auto推导函数名时，结果为函数指针；声明为引用时为函数引用：
```C++
void func() {}
auto func_ptr = func; // func_ptr: void(*)()（函数指针）
auto& func_ref = func; // func_ref: void(&)()（函数引用）
```

### 禁止用于函数参数与模板参数

C++11中，auto**不能作为函数参数类型**或**模板参数类型**，仅允许用于变量声明。
auto不能作为形参类型，因为编译器无法对x的实际类型进行推导。

```C++
void func(auto x) {} // 错误：C++11不支持auto函数参数（C++20起支持）
template <auto T> struct A {}; // 错误：C++11不支持auto模板参数（C++17起支持）
```

### 类成员变量与数组声明限制
(1) auto**不能用于非静态类成员变量**
```C++
struct S {
    auto a = 10; // 错误：非静态成员变量不允许auto
    static const auto b = 20; // 正确：静态常量成员可结合const使用
};
```

 (2) auto**不能直接声明数组**：
 ```C++
 auto arr3 = {1,2,3}; // 错误：auto无法推导数组类型
 ```


## 常见错误与最佳实践
### 典型错误案例分析

|错误场景|代码示例|错误原因|修复方案|
|---|---|---|---|
|混合类型声明|`auto a = 1, b = 2.0;`|推导类型不一致（int与double）|分开声明或统一类型|
|非const引用绑定右值|`auto& x = 10;`|右值无法绑定非const左值引用|声明为`const auto& x = 10;`|
|依赖auto的类型可见性|`auto result = compute(); // 类型不明确`|读者无法直观判断result类型|注释说明或仅在类型冗长时使用auto|

### 最佳实践建议

**优先用于复杂类型**：如迭代器、lambda表达式类型（无法显式写出），避免基础类型（如`int`、`double`）滥用auto，影响可读性。
**显式控制cv与引用**：若需保留初始化表达式的const或引用属性，显式添加`const`、`&`或`&&`。
**结合IDE工具**：使用IDE的类型提示功能（如VS Code的悬停查看类型），辅助确认auto推导结果。
**遵循编码规范**：参考Google C++ Style Guide建议，仅在类型明显或冗长时使用auto，确保代码可维护性。

### C++11与C++14/17的扩展对比
尽管本文聚焦C++11，但需明确auto在后续标准中的扩展，避免混淆：

- **C++14**：支持函数返回类型推导（`auto func() { return 1; }`）和`decltype(auto)`；
- **C++17**：支持模板参数推导（`template <auto N> struct A {}`）和结构化绑定（`auto [x, y] = pair`）。

C++11中auto的能力有限，**仅支持变量类型推导**，上述扩展特性不可使用。

## 总结

C++11的auto关键字通过编译期类型推导，平衡了代码简洁性与类型安全性。其核心价值在于简化复杂类型声明，减少手动类型指定错误。然而，**开发者需深入理解其推导规则（按值/引用/万能引用），规避未初始化、类型丢失等陷阱**，并遵循“==复杂类型优先使用，基础类型谨慎使用==”的原则。合理运用auto，可显著提升现代C++代码的可读性与开发效率。

# decltype


# 参考
```bash

```