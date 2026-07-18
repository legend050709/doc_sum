```table-of-contents
```

# 背景

在C/C++中，变量、函数和类都是大量存在的，这些变量、函数和类的名称都将作用于全局作用域中，可能会导致很多命名冲突。  
使用命名空间的目的就是**对标识符和名称进行本地化**，以**避免命名冲突或名字污染**，namespace关键字的出现就是针对这种问题的。

## C语言中没有命名空间而存在的问题
在C语言中，所有的全局标识符（包括函数名、变量名等）都位于同一个全局作用域中。这导致了几个显著的问题：

**命名冲突**：当多个库或模块包含相同名称的函数或变量时，这些标识符之间会发生冲突。开发者需要手动修改名称以避免冲突，这既繁琐又容易出错。

**代码可读性差**：随着项目规模的增大，全局作用域中的标识符数量急剧增加，使得代码的阅读和维护变得困难。开发者需要花费更多时间来理解每个标识符的来源和用途。

**模块间耦合度高**：由于所有全局标识符都可见，模块间的依赖关系可能变得复杂且难以管理。这增加了代码重构和模块化的难度。


### 问题范例
在C语言中，如果定义一个rand全局变量，刚开始，可以正常打印；
![](attachments/Pasted%20image%2020260720223409.png)

然后，包含了`<stdlib.h>`头文件之后，就报错了，因为在stdlib头文件中，rand是函数，这里我们又定义了以rand全局变量，就产生了命名冲突
![](attachments/Pasted%20image%2020260720223424.png)

## C++引入了命名空间解决的问题

C++通过引入命名空间（namespace）机制来解决上述问题：

**解决命名冲突**：命名空间允许开发者将相关的标识符组织在一起，并通过命名空间名称作为前缀来访问这些标识符。这样，即使不同的库或模块包含相同名称的标识符，只要它们位于不同的命名空间中，就不会发生冲突。

**提高代码可读性**：命名空间为代码提供了一种自然的分组方式，使得相关的标识符能够按照逻辑或功能进行组织。这有助于开发者快速理解代码的结构和每个标识符的用途。

**降低模块间耦合度**：通过限制命名空间成员的可见性，C++可以减少模块间的依赖关系。开发者可以更加灵活地重构和模块化代码，而无需担心意外的命名冲突或依赖问题。

### 范例

还是上面的例子，在C++中，将rand全局变量放在了命名空间中后，就不会与头文件中rand函数发生冲突

![](attachments/Pasted%20image%2020260720223546.png)


# C++命名空间的重要性

C++命名空间的重要性体现在以下几个方面：

**支持大型项目**：对于大型项目而言，命名空间是组织和管理代码的关键工具。它有助于减少命名冲突、提高代码可读性和可维护性。

**促进模块化编程**：命名空间鼓励开发者将代码划分为独立的模块或库，并通过命名空间来区分这些模块或库中的标识符。这有助于实现更加清晰和灵活的模块化编程。

**与标准库集成**：C++标准库中的所有内容都定义在std命名空间中。通过使用命名空间，标准库能够与用户代码和谐共存，而不会引发命名冲突。

**增强代码复用性**：命名空间使得库和框架的开发者能够更容易地提供可复用的代码。通过定义清晰的命名空间，他们可以避免命名冲突，并确保库或框架中的标识符在与其他代码集成时保持清晰和一致。


**总之，C++命名空间是一种强大的代码组织工具，它有助于解决命名冲突、提高代码的可读性和可维护性。通过合理使用命名空间，你可以创建出更加清晰、模块化和可复用的C++代码**。

# 命名空间的定义

定义命名空间，需要使用到 namespace 关键字，后面跟命名空间的名字，然后接一对{}即可，{}中即为命名空间的成员。
命名空间中可以定义变量/函数/类型等。



## 命名空间的普通定义

```C++
//命名空间的普通定义
namespace N1 //N1为命名空间的名称
{
	//在命名空间中，既可以定义变量，也可以定义函数
	int a;
	int Add(int x, int y)
	{
		return x + y;
	}
}

```

## 命名空间的嵌套定义

![](attachments/Pasted%20image%2020260720224158.png)

## 命名空间的成员合并

同一个工程中**多文件定义同名的命名空间，它们会被当做是同一个命名空间，自动合并到一起**。
所以我们**不能在相同名称的命名空间中定义两个相同名称的成员**。

![](attachments/Pasted%20image%2020260720224254.png)
![](attachments/Pasted%20image%2020260720224258.png)


## 小结
- namespace本质是定义出⼀个域，这个域跟全局域各⾃独⽴，不同的域可以定义同名变量。
-  **namespace只能定义在全局，当然他还可以嵌套定义**。
- 项⽬⼯程中多⽂件中定义的同名namespace会认为是⼀个namespace，不会冲突。
- C++标准库都放在⼀个叫std(standard)的命名空间中。

# 命名空间的使用

编译查找⼀个变量的声明/定义时，默认只会在局部或者全局查找，不会到命名空间⾥⾯去查找。

所以我们要使⽤命名空间中定义的变量/函数，有三种⽅式：
（1）指定命名空间访问：项⽬中推荐这种⽅式。
（2）using将命名空间中某个成员展开：项⽬中经常访问的不存在冲突的成员推荐这种⽅式。
（3）展开命名空间中全部成员：项⽬不推荐，冲突⻛险很⼤，⽇常⼩练习程序为了⽅便推荐使⽤。


## 指定命名空间访问：`命名空间名称::命名空间成员`
==符号“::”在C++中叫做**作用域限定符**==，我们通过“命名空间名称::命名空间成员”便可以访问到命名空间中相应的成员。

```C++
//加命名空间名称及作用域限定符
#include <stdio.h>
namespace N
{
	int a;
	double b;
}
int main()
{
	N::a = 10;//将命名空间中的成员a赋值为10
	printf("%d\n", N::a);//打印命名空间中的成员a
	return 0;
}

```

![](attachments/Pasted%20image%2020260720224609.png)

## using将命名空间中某个成员展开：`using 命名空间名称::命名空间成员`

我们还可以通过“using 命名空间名称::命名空间成员”的方式将命名空间中指定的成员引入。这样一来，在该语句之后的代码中就可以直接使用引入的成员变量了。

```C++
//使用using将命名空间中的成员引入
#include <stdio.h>
namespace N
{
	int a;
	double b;
}

using N::a;//将命名空间中的成员a引入
int main()
{
	a = 10;//将命名空间中的成员a赋值为10
	printf("%d\n", a);//打印命名空间中的成员a
	return 0;
}
```

![](attachments/Pasted%20image%2020260720224631.png)

## 展开命名空间中全部成员：using namespace 命名空间

通过”using namespace 命名空间名称“将命名空间中的全部成员引入。这样一来，在该语句之后的代码中就可以直接使用该命名空间内的全部成员了。

```C++
//使用using namespace 命名空间名称引入
#include <stdio.h>
namespace N
{
	int a;
	double b;
}
using namespace N;//将命名空间N的所有成员引入

int main()
{
	a = 10;//将命名空间中的成员a赋值为10
	printf("%d\n", a);//打印命名空间中的成员a
	return 0;
}

```

![](attachments/Pasted%20image%2020260720224658.png)


# using关键字

## 作用

|用途|作用|示例|
|---|---|---|
|引入整个命名空间|引入命名空间或其中的成员|`using namespace std;`|
|引入命名空间单个成员|将命名空间中单个成员注入到当前作用域的机制，使得在当前作用域下访问另一个作用域（命名空间）下的成员时无需使用限定符 `::`|`using 命名空间名::成员名`|
|类型别名|给类型起别名|`using int_ptr = int*;`|
|模板别名|给模板类型起别名|`template<class T> using Vec = vector<T>;`|

## 命名空间引入
引入命名空间，无论是引入整个命名空间，还是命名空间的单个成员，都是在**指定的作用域** 后续访问命名空间的成员时，可以不用写作用域限定符（`::`）。

### 引入命名空间的单个成员
引入命名空间中的单个成员，之后可以直接使用该成员，不用再加作用域限定符(`::`)。
将命名空间中单个名字注入到当前作用域的机制，使得在当前作用域下访问另一个作用域（命名空间）下的成员时无需使用限定符 `::`

using 声明将其它 namespace 的成员引入本命名空间的 **当前作用域** (包括其嵌套作用域)  。
```C++
#include <iostream>
 
int main() {
    // 没用 using 之前，必须写 std::
    std::cout << "Hello" << std::endl;
    
    // 用 using 声明后，可以直接写 cout 和 endl
    using std::cout;
    using std::endl;
    
    cout << "Hello" << endl;  // 不需要 std::
    
    // 其他没引入的成员，仍然需要 std::
    int a;
    std::cin >> a;  // cin 没引入，还是要写 std::
    
    return 0;
}
```
**说明**：`using std::cout;` 的意思是“把 `std` 命名空间里的 `cout` 引入当前作用域，以后直接用 `cout` 就行”。只引入这一个，其他成员不受影响。


利用 using 声明，可以改变派生类对父类成员的访问控制:
```C++
class Base{
protected:
    int bn1;
    int bn2;
};
 
class Derived: private Base{
public:
    using Base::bn1;
};
 
class DerivedAgain: public Derived{
};
 
int main(){
    Derived d;
    DerivedAgain da; 
    d.bn1 = 1;
    d.bn2 = 2; //error, 'bn2' is a private member of 'Base'
    da.bn1 = 3;  //ok
    std::cout<<d.bn1<<std::endl;
 
    return 0;
}
```
尽管 Derived 对 base 是私有继承，但通过 using 声明，我们还是可以在 Derived 中访问其成员，且后续的继承同样不受 private 限定的影响。


### 引入整个命名空间
引入整个命名空间，之后所有成员都可以直接使用。是使一个命名空间中的 **所有** 名字都在该作用域中可见的机制。这是最常用的方式了。需要注意的是命名冲突问题。



```C++
#include <iostream>
using namespace std;  // 引入整个 std 命名空间
 
int main() {
    cout << "Hello" << endl;  // 直接使用
    int a;
    cin >> a;
    return 0;
}
```

Notice: 尽管 using指示很方便，但在实际工作中应该尽量避免：它一下子将另一个 namespace 中的成员全部引入了，一不小心就会出现命名空间污染问题。

```C++
#include <iostream>
namespace n1{ 
    int n1_member = 10; 
    int m = 11; 
}
 
int m = 12; 
 
int main(){
    using namespace n1; 
    std::cout<<n1_member<<std::endl;
    //std::cout<<m<<std::endl;  //error 命名冲突
    std::cout<<::m<<std::endl;
 
    int m = 13; //ok, 局部变量屏蔽命名空间变量
    std::cout<<m<<std::endl;
 
    return 0;
}
```

### 注意事项：避免二义性
```C++
namespace A {
    void func() { cout << "A" << endl; }
}
 
namespace B {
    void func() { cout << "B" << endl; }
}
 
using namespace A;
using namespace B;
 
int main() {
    func();  // ❌ 错误：不知道调用 A::func 还是 B::func
}
```
#### 解决方法
**(1)方法1：不用 using，直接用完整名字**
```C++
A::func();  // 明确调用 A 的 func
B::func();  // 明确调用 B 的 func
```
说明：最安全的方式，不会有任何歧义，推荐在头文件或大型项目 中使用。

**(2)方法2：用 using 声明，只引入不冲突的成员**
```C++
using A::func;  // 只引入 A 的 func
func();         // 调用 A::func
B::func();      // B 的 func 还是写全名
```
说明：只引入需要的成员，减少命名空间污染，同时避免冲突。

**(3)方法3：在局部作用域使用**
```C++
{
    using namespace A;
    func();  // 在这个大括号里，func 指 A::func
}
{
    using namespace B;
    func();  // 在这个大括号里，func 指 B::func
}
```
说明：将 `using namespace` 限制在局部作用域，不同区域可以分别使用不同的命名空间，互不影响。

## 类型重定义，取代 typedef
C++11 引入了 `using` 作为类型别名的新语法，比传统的 `typedef` 更直观。

### typedef 的用法

语法格式：
```C++
typedef 旧类型  新类型;
```

范例如下：
```C++
typedef unsigned int u_int;      // 无符号整型
typedef char* pChar;              // 字符指针
typedef int Array[10];            // 数组
typedef void (*pFun)(int, int);   // 函数指针
```

### using 类型别名
语法：
```C++
using 类型别名 = 旧的类型;
```

范例：
```C++
using u_int = unsigned int;       // 无符号整型
using pChar = char*;              // 字符指针
using Array = int[10];            // 数组
using pFun = void(*)(int, int);   // 函数指针
```

```C++
using value_type = int;
using pointer = int*;
using const_pointer = const int*;
using reference = int&;
using const_reference = const int&;
```

### typedef 和 using 对比

![](attachments/Pasted%20image%2020260722224902.png)

**using 的优势**：语法更直观，尤其是函数指针，`using pFun = void(*)(int, int);` 比 `typedef` 更容易理解。


## 模版别名

`using` 最大的优势是**支持模板别名**，而 `typedef` 不能。

### 定义模板别名
```C++
template<class T, size_t N>
using Array = T[N];
```
**说明**：`Array` 是一个模板别名，`T` 是元素类型，`N` 是数组大小。使用时只需指定这两个参数。

```C++
int main() {
    Array<int, 5> ar;      // int ar[5]
    Array<int, 5> br;      // int br[5]
    
    Array<double, 5> dr;   // double dr[5]
    
    Array<int*, 10> par;   // int* par[10]
    
    return 0;
}
```
**说明**：一次定义，多次使用。不同大小、不同类型的数组都可以用同一个模板别名。

#### 为什么 typedef 做不到？
```C++
// typedef 做不到！必须为每个类型单独写
typedef int Array_int_5[5];
typedef double Array_double_5[5];
// 无法模板化
```
**原因**：`typedef` 只能给**具体的、已经确定的类型**起别名。如果需要多种类型或大小的数组，只能重复写，无法模板化。

#### 实际应用场景
```C++
// 简化 STL 容器写法
template<class T>
using Vec = std::vector<T>;
 
int main() {
    Vec<int> v;        // 等价于 std::vector<int>
    Vec<double> d;     // 等价于 std::vector<double>
    Vec<string> s;     // 等价于 std::vector<string>
    return 0;
}
```
**说明**：每次写 `std::vector<int>` 太长了，用 `Vec<int>` 代替，代码更短。而且 `Vec` 是模板，可以生成任意类型的 vector。

#### 范例
```C++
#include <iostream>
#include <vector>
using namespace std;
 
// 1. 类型别名：简化复杂类型
using UInt = unsigned int;
using PFunc = void(*)(int);  // 函数指针类型
 
// 2. 模板别名：简化嵌套容器
template<class T>
using Vec2D = vector<vector<T>>;  // 二维数组
 
void printInt(int x) {
    cout << x << endl;
}
 
int main() {
    // 类型别名使用
    UInt a = 10;           // unsigned int
    PFunc f = printInt;    // 函数指针
    f(a);
    
    // 不用模板别名：写法冗长
    vector<vector<int>> m1 = {{1, 2}, {3, 4}};
    
    // 用模板别名：写法简洁
    Vec2D<int> m2 = {{1, 2}, {3, 4}};  // 等价于 vector<vector<int>>
    
    for (auto& row : m2) {
        for (auto x : row) {
            cout << x << " ";
        }
        cout << endl;
    }
    
    return 0;
}
```

**示例说明**：
- 类型别名：`UInt` 比 `unsigned int` 短，`PFunc` 让函数指针更易读
- 模板别名：`Vec2D<int>` 比 `vector<vector<int>>` 短很多，嵌套层次越深优势越明显

# 双冒号(`::`)的含义和使用场景

在 C++ 中，双冒号 `::` 被称为**作用域解析运算符**（Scope Resolution Operator） 或者 作用域符/ 作用域限定符。它的核心作用是**指明某个标识符（如变量、函数、类）所属的命名空间或类，从而消除歧义或访问特定范围内的成员。**

简单来说，它就像“所属关系”的冒号，告诉编译器：“请去这个特定的范围内查找这个名字”。

## 背景

C++中域有函数局部域，全局域，命名空间域，类域。域影响的是编译时语法查找⼀个变量/函数/ 类型出处(声明或定义)的逻辑，所有有了域隔离，名字冲突就解决了。

局部域和全局域除了会影响 编译查找逻辑，还会影响变量的生命周期；命名空间域和类域不影响变量生命周期，只影响作用域。

因此：**在C++中，作用域限定符（Scope Resolvers）主要用于访问特定作用域中的成员**，特别是在处理类、命名空间（Namespace）等复杂结构时非常有用。它们帮助编译器确定某个标识符（如变量名、函数名等）的精确作用域，从而避免命名冲突和歧义。

## 含义

`::` 左边的部分代表一个**作用域**（命名空间或类），右边代表该作用域内的**成员**（变量、函数、类型等）。如果左边为空（即 `::` 开头），则代表**全局作用域**。

## 使用场景
`::`是运算符中**等级最高的**，它分为三种:

1)global scope(全局作用域符），用法（::name)。
2)class scope(类作用域符），用法(class::name)。
3)namespace scope(命名空间作用域符），用法(namespace::name)。

他们都是左关联（left-associativity)，他们的作用都是为了更明确的调用你想要的变量。
1》如在程序中的某一处你想调用全局变量a，那么就写成::a；（也可以是全局函数）
2》如果想调用class A中的静态成员变量a，那么就写成A::a；
3》另外一个如果想调用namespace std中的cout成员，你就写成std::cout（相当于using namespace std；再单独调用 cout）意思是在这里我想用cout对象是命名空间std中的cout（即就是标准库里边的cout）；

### 变量/函数冲突时：访问全局变量或者全局函数(`::x`)

在函数内部，如果局部变量与全局变量同名，局部变量的作用域会覆盖全局变量的作用域。
此时，如果要访问全局变量，就需要使用`::`操作符。不过，通常不推荐在函数内部使用全局变量，因为这会增加代码的耦合度和复杂度。

但为了演示作用域限定符的用法，这里给出一个例子：
```C++
#include <iostream>  
  
int x = 5; // 全局变量  
  
void func() {  
    int x = 10; // 局部变量  
    std::cout << ::x << std::endl; // 使用::访问全局变量x，输出5  
    std::cout << x << std::endl; // 访问局部变量x，输出10  
}  
  
int main() {  
    func();  
    return 0;  
}

```

### 访问命名空间中的成员

命名空间是C++中用于组织代码的一种方式，可以避免全局命名冲突。在访问命名空间中的成员时，可以使用`::`操作符来指定命名空间。

```C++
#include <iostream>  
  
namespace Math {  
    int add(int a, int b) {  
        return a + b;  
    }  
}  
  
int main() {  
    std::cout << Math::add(2, 3) << std::endl; // 访问Math命名空间中的add函数  
    return 0;  
}

```


#### 命名空间的嵌套

当命名空间或类嵌套时，可以通过连续使用`::`操作符来访问深层的成员。

```C++
namespace Outer {  
    namespace Inner {  
        int value = 10;  
    }  
}  

class OuterClass {  
public:  
    class InnerClass {  
    public:  
        static int value = 20;  
    };  
};  

int main() {  
    std::cout << Outer::Inner::value << std::endl; // 访问嵌套命名空间中的value  
    std::cout << OuterClass::InnerClass::value << std::endl; // 访问嵌套类中的静态成员变量  
    return 0;  
}
```

### 类的静态成员（变量和函数）访问

类的静态成员（包括静态变量和静态成员函数）属于类本身，而不是类的某个具体对象。因此，在访问这些静态成员时，可以使用类名和作用域限定符`::`。

> 注：类的非静态成员（包括成员变量和成员函数）通常不能直接通过作用域限定符（`::`）来访问，因为非静态成员是依赖于类的具体对象的。

```C++
#include <iostream>  
  
class MyClass {  
public:  
    static int count; // 静态成员变量  
    static void printCount() { // 静态成员函数  
        std::cout << "Count: " << count << std::endl;  
    }  
};  
  
int MyClass::count = 0; // 静态成员变量初始化  
  
int main() {  
    MyClass::count = 5; // 访问并修改静态成员变量  
    MyClass::printCount(); // 调用静态成员函数，输出：Count: 5  
    return 0;  
}

```

### 在类外部定义成员函数

当你在 `.cpp` 文件中实现类体内声明的成员函数时，必须用 `::` 告诉编译器这个函数属于哪个类。

```C++
// MyClass.h
class MyClass {
public:
    void doSomething();
};

// MyClass.cpp
#include "MyClass.h"
// 必须用 ClassName:: 前缀，否则编译器会当成普通全局函数
void MyClass::doSomething() { 
    // 实现代码
}
```




### 容器实例(类模版实例化「具体的类型」)的类型成员

```C++
vector<int> v = {1,2,3,4};
vector<int>::iterator it = v.begin();
```

### 类的类型成员
标准库中string的size函数的返回值是一个`string::size_type类型`；这是一个无符号类型的值，而且足够存放下任何string对象的大小。

```C++
string::size_type len = s.size();
```

### 在派生类中访问基类的被覆盖成员

当子类重写了父类的函数，但你想在子类中调用父类的原始版本时，用 `基类名::` 来显式指明。

(1) 在子类中直接访问父类的同名的public的成员
```C++
class Base {
public:
    void print() { std::cout << "Base"; }
};

class Derived : public Base {
public:
    void print() { 
        Base::print();   // 调用父类的 print，避免无限递归
        std::cout << "Derived"; 
    }
};
```

(2) 在子类外（子类实例中）直接访问父类的同名的public的成员
```C++
#include <iostream>
#include <string>
using namespace std;

//父类
class Person
{
public:
	void fun(int x)
	{
		cout << x << endl;
	}
};

//子类
class Student : public Person
{
public:
	void fun(double x)
	{
		cout << x << endl;
	}
};

int main()
{
	Student s;
	s.fun(3.14);       //直接调用子类当中的成员函数fun
	s.Person::fun(20); //指定调用父类当中的成员函数fun
	return 0;
}
```


### 访问嵌套类或枚举
如果在一个类或命名空间内部定义了另一个类或枚举，外部访问时需要层层解析。

```C++
class Outer {
public:
    enum Color { RED, GREEN, BLUE };
    class Inner {
    public:
        static void hello() {}
    };
};

int main() {
    Outer::Color c = Outer::RED;
    Outer::Inner::hello();
    return 0;
}

```


## `::`和`.`以及`->`的对比

|运算符|名称|作用对象|示例|
|:--|:--|:--|:--|
|`::`|作用域解析|类、命名空间、全局作用域|`std::cout`, `MyClass::func()`|
|`.`|成员访问|对象实例、引用|`obj.func()`, `ref.member`|
|`->`|指针成员访问|指针|`ptr->func()`, `ptr->member`|

|运算符|左边操作数|使用场景|
|---|---|---|
|`::`|**类名** 或 **命名空间名**|访问静态成员、命名空间成员、类外定义函数|
|`.`|**对象** 或 **引用**|访问对象的非静态成员|
|`->`|**指针**|指针指向对象的成员，等价于 `(*ptr).`|

总结：只要你想访问的东西**不是**通过一个具体的“对象实例”或“指针”去点的，而是通过“名字”（类名、命名空间名）去定位的，或者你想强制回到全局作用域，就用 `::`。


# 参考
```bash
# 【C++指南】命名空间
https://blog.csdn.net/2302_78391795/article/details/140417082?spm=1001.2014.3001.5502
```