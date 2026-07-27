```table-of-contents
```
# 左值和右值
## 左值（Lvalue）
左值是一个表示数据的表达式，如变量名或解引用的指针。 
- 左值可以被取地址，也可以被修改（const修饰的左值除外）。

左值：有名字、有固定内存地址、**可以取地址**（`&`）的对象。它代表一个**持久的存储位置**。

## 右值（Rvalue）
右值也是一个表示数据的表达式，如字母常量、表达式的返回值、函数的返回值（不能是左值引用返回）等等。

右值通常是**临时对象、常量、值返回的函数调用**。右值不能被取地址。比如：`&42` 或 `&(a+b)` 确实会报错。
> 这些临时变量和常量值并没有被实际存储起来，这也就是为什么右值不能被取地址的原因，因为只有被存储起来后才有地址。
```C++
#include <iostream>
using namespace std;

int main() {
    // 字面量是右值，不能取地址
    // int* p = &42;        // ❌ 编译错误：不能对右值取地址
    
    // 表达式结果也是右值
    int a = 10, b = 20;
    // int* p2 = &(a + b);  // ❌ 编译错误：不能对右值取地址
    
    // 临时对象也是右值
    // int* p3 = &std::string("Hello");  // ❌ 编译错误
    
    return 0;
}
```


**右值本质就是一个临时变量或常量值**，比如代码中的10就是常量值，表达式x+y和只返回的函数fmin的返回值就是临时变量，这些都叫做右值。
> 因为传值返回的函数在返回对象时返回的是对象的拷贝，这个拷贝出来的对象就是一个临时变量。

对于左值引用返回的函数来说，这些函数返回的是左值。比如string类实现的`[]`运算符重载函数。
```C++
namespace cl
{
	//模拟实现string类
	class string
	{
	public:
		//[]运算符重载（可读可写）
		char& operator[](size_t i)
		{
			assert(i < _size); //检测下标的合法性
			return _str[i]; //返回对应字符
		}
		//...
	private:
		char* _str;       //存储字符串
		size_t _size;     //记录字符串当前的有效长度
		//...
	};
}
int main()
{
	cl::string s("hello");
	s[3] = 'x';    //引用返回，支持外部修改
	return 0;
}


```

说明：
这里的`[]`运算符重载函数返回的是一个字符的引用，因为它需要支持外部对该位置的字符进行修改，所以必须采用左值引用返回。之所以说这里返回的是一个左值，是因为这个返回的字符是被存储 起来了的，是存储在string对象的_str对象当中的，因此这个字符是可以被取到地址的。


### 右值的场景
#### 字面量：除字符串字面量外的字面量，都是右值
```C++
// ✅ 右值：整数字面量
42
3.14
true
'c'

// ❌ 不是右值：字符串字面量（是左值，有地址）
"Hello"  // 这是const左值，存储在只读数据段, 可以取地址。&"Hello"

// 验证
int&& r1 = 42;        // ✅ 可以
double&& r2 = 3.14;   // ✅ 可以
const char* p = "Hello";  // "Hello" 是左值，不能绑定到 char*&&
```

#### 临时对象（Temporary Objects）

临时对象（Temporary Objects）：通过构造函数（但是没有赋值给新的变量）或类型转换创建的临时对象。
```C++
// 2.1 直接创建临时对象
std::string("Hello");        // ✅ 右值
MyClass(10);                 // ✅ 右值

// 2.2 类型转换产生的临时对象
static_cast<double>(10);     // ✅ 右值（临时 double）
(int)3.14;                   // ✅ 右值（C 风格转换）

// 2.3 隐式类型转换产生的临时对象
void foo(const std::string& s);
foo("Hello");  // "Hello" → std::string 临时对象（右值）

// 验证
std::string&& r3 = std::string("Hello");  // ✅ 可以
int&& r4 = static_cast<int>(3.14);        // ✅ 可以
```

#### 匿名对象（Anonymous Objects）：
匿名对象：没有名字的临时对象
```C++
class Widget {
public:
    Widget() { std::cout << "构造\n"; }
};

// 3.1 无名对象
Widget();                    // ✅ 右值（临时 Widget）
std::vector<int>{1,2,3};     // ✅ 右值

// 3.2 返回值优化中的匿名对象
Widget CreateWidget() {
    return Widget();  // 这个临时 Widget 是右值
}

// 验证
Widget&& r5 = Widget();                    // ✅ 可以
std::vector<int>&& r6 = std::vector<int>{1,2,3};  // ✅ 可以
```

#### 表达式的值（Expression Results）

运算符和表达式产生的结果
```C++
int a = 10, b = 20;

// 4.1 算术运算结果
a + b                      // ✅ 右值（临时 int）
a * b                      // ✅ 右值
a - b                      // ✅ 右值

// 4.2 比较运算结果
a > b                      // ✅ 右值（临时 bool）
a == b                     // ✅ 右值

// 4.3 逻辑运算结果
a && b                     // ✅ 右值（临时 bool）
!(a > b)                   // ✅ 右值

// 4.4 位运算结果
a & b                      // ✅ 右值（临时 int）
a << 2                     // ✅ 右值

// 4.5 取地址（不是右值）
&a                         // ✅ 这是右值（临时指针）
int*&& r7 = &a;            // ✅ 可以
// &a 是右值，因为不能再取地址。

// 验证
int&& r8 = a + b;          // ✅ 可以
bool&& r9 = a > b;         // ✅ 可以
```

#### 值返回的函数调用
函数返回非引用类型时，返回值是右值。

```C++
// 5.1 返回基本类型
int GetInt() { return 42; }
int&& r10 = GetInt();          // ✅ 可以

// 5.2 返回自定义类型
std::string GetString() { 
    return std::string("Hello"); 
}
std::string&& r11 = GetString();  // ✅ 可以

// 5.3 返回 STL 容器
std::vector<int> GetVector() {
    return std::vector<int>{1,2,3};
}
std::vector<int>&& r12 = GetVector();  // ✅ 可以

// 5.4 返回对象（触发移动）
MyString CreateMyString() {
    MyString temp("Hello");
    return temp;  // temp 是左值，但 return 语句把它当作右值
}
MyString&& r13 = CreateMyString();  // ✅ 可以（延长生命周期）
```

#### std::move 转换的结果
std::move 把左值强制转成右值

```C++
std::string s1("Hello");
std::string&& r14 = std::move(s1);  // ✅ 可以

std::vector<int> v{1,2,3};
std::vector<int>&& r15 = std::move(v);  // ✅ 可以

// 注意：std::move 本身就是函数调用，返回右值引用
int a = 10;
decltype(std::move(a)) r16 = std::move(a);  // r16 是 int&&
```

#### lambda 表达式（本身是右值）
在 C++ 中，Lambda 表达式并不是一个普通的函数指针，它本质上是一个**匿名函数对象**（仿函数）。

```C++
// lambda 是临时对象（右值）
auto&& r25 = [](){ return 42; };  // lambda 是右值

// 立即调用 lambda 表达式的结果
int&& r26 = [](){ return 42; }();  // 返回的 42 是右值
```

#### 后置递增/递减的结果
```C++
int a = 10;
int&& r22 = a++;  // ✅ 可以（a++ 返回旧的临时值）
int&& r23 = a--;  // ✅ 可以

// 注意：前置递增/递减是左值
int&& r24 = ++a;  // ❌ 错误！++a 返回左值
```


## 区分左值和右值
- **左值**：有名字、有固定内存地址、**可以取地址**（`&`）的对象。它代表一个**持久的存储位置**。
- **右值**：没有名字、通常是临时的、**不能取地址**的值。它代表一个**即将销毁的临时数据**或**纯数值**。
> 注：“有名字”意味着这个数据在内存中有一个明确的、持久的存储位置，并且你可以通过一个标识符（变量名）反复找到它。


### 快速区分

当你看到一个表达式时，问自己两个问题：
1. **我能对它取地址吗？** (`&expr`)
    - 能 ：**左值**
    - 不能 ：**右值**

2. **是否有名字**
    - 有 ：**左值**
    - 没有 ：**右值**

### 特殊：字符串字面量 `"hello"`是const左值, 不是右值

**字符串字面量的类型是 `const char[N]`**，其中 `N` 是字符个数 + 1（末尾的空字符）；不是`char[N]`, 因为字符串字面量在C++中是不可修改的。

```C++
&"hello"; // 可以取地址，左值（const char[6]）
```


|写法|类型|左值/右值|
|---|---|---|
|`"Hello"`|`const char[6]`|**左值**（存储在静态存储区，有固定地址）|
|`const char* p = "Hello"` 中的 `p`|`const char*`|**左值**（有变量名，可取地址）|
|`"Hello"[0]`|`const char&`（实际上是引用）|**左值**（数组元素是内存中的具体对象）|
|`std::string("Hello")`|`std::string`|**右值**（临时对象）|

```C++
const char* p = "Hello";   // p 的类型是 const char*
auto p2 = "Hello";         // p2 的类型是 const char* （注意！auto 会退化数组为指针）
auto& p3 = "Hello";        // p3 的类型是 const char (&)[6] （引用保留数组类型）

测试如下：

#include <iostream>
#include <typeinfo>

int main() {
    const char* p = "Hello";
    auto p2 = "Hello";
    auto& p3 = "Hello";
    
    std::cout << typeid(p).name() << std::endl;   // const char* (MSVC) 或 PKc (GCC)
    std::cout << typeid(p2).name() << std::endl;  // const char*
    std::cout << typeid(p3).name() << std::endl;  // const char [6]
    return 0;
}
```


### 特殊：临时对象（匿名对象）都是右值
```C++
string("test");
Point{1,2}; 
```

### 特殊：一个变量取地址后（`&a`）整体是右值
```C++
int a = 10;
&a                         // ✅ 这是右值（临时指针）
int*&& r7 = &a;            // ✅ 可以

// a 是左值，但是 &a 是右值，因为不能再取地址。
```


# 左值引用 和右值引用
参考：[C++11 右值引用：从入门到精通](https://cloud.tencent.com/developer/article/2528826)
参考：[《C++11》深入解析引用限定符：掌握左值与右值的关键技巧](https://cloud.tencent.com/developer/article/2486860)


## 左值引用限定符(`&`)
左值引用就是对左值的引用，给左值取别名，通过“&”来声明。比如：

### 定义和语法
基本语法为：**类型 & 引用名 = 左值**。

范例：
```C++
int x = 10; 
int &ref = x; // ref 是 x 的别名，x 是左值；ref是左值引用
ref = 20; // 修改 ref 即修改 x 的值

int main()
{
	//以下的p、b、c、*p都是左值
	int* p = new int(0);
	int b = 1;
	const int c = 2;

	//以下几个是对上面左值的左值引用
	int*& rp = p;
	int& rb = b;
	const int& rc = c;
	int& pvalue = *p;
	return 0;
}
```

#### const 左值引用
基本语法为：**const 类型 & 引用名 = 左值**。

### 左值引用变量
同上，参考参数是左值引用。

### 左值引用成员函数
`&`写在函数后：`&` 限定表示**只能被左值对象调用**；

```C++
class MyClass {
    public:
        void foo() & { // 左值引用限定符
            // ...
        }
};
```


## 左值引用使用场景
### 函数参数使用(左值)引用：避免值传递（拷贝构造）
如果函数的形参是自定义类型的**值传递**，会存在**拷贝构造函数**的调用。
如果函数的形参是自定义类型的**引用传递**，则==不会==调用**拷贝构造函数**。

### 函数返回值使用(左值)引用：避免值传递（拷贝构造）
如果函数的返回值是自定义类型的**值传递**，会存在**拷贝构造函数**的调用。
如果函数的返回值是自定义类型的**引用传递**，则==不会==调用**拷贝构造函数**。

### 范例
```C++
namespace cl
{
	class string
	{
	public:
		typedef char* iterator;
		iterator begin()
		{
			return _str; //返回字符串中第一个字符的地址
		}
		iterator end()
		{
			return _str + _size; //返回字符串中最后一个字符的后一个字符的地址
		}
		//构造函数
		string(const char* str = "")
		{
			_size = strlen(str); //初始时，字符串大小设置为字符串长度
			_capacity = _size; //初始时，字符串容量设置为字符串长度
			_str = new char[_capacity + 1]; //为存储字符串开辟空间（多开一个用于存放'\0'）
			strcpy(_str, str); //将C字符串拷贝到已开好的空间
		}
		//交换两个对象的数据
		void swap(string& s)
		{
			//调用库里的swap
			::swap(_str, s._str); //交换两个对象的C字符串
			::swap(_size, s._size); //交换两个对象的大小
			::swap(_capacity, s._capacity); //交换两个对象的容量
		}
		//拷贝构造函数（现代写法）
		string(const string& s)
			:_str(nullptr)
			, _size(0)
			, _capacity(0)
		{
			cout << "string(const string& s) -- 深拷贝" << endl;

			string tmp(s._str); //调用构造函数，构造出一个C字符串为s._str的对象
			swap(tmp); //交换这两个对象
		}
		//赋值运算符重载（现代写法）
		string& operator=(const string& s)
		{
			cout << "string& operator=(const string& s) -- 深拷贝" << endl;

			string tmp(s); //用s拷贝构造出对象tmp
			swap(tmp); //交换这两个对象
			return *this; //返回左值（支持连续赋值）
		}
		//析构函数
		~string()
		{
			delete[] _str;  //释放_str指向的空间
			_str = nullptr; //及时置空，防止非法访问
			_size = 0;      //大小置0
			_capacity = 0;  //容量置0
		}
		//[]运算符重载
		char& operator[](size_t i)
		{
			assert(i < _size); //检测下标的合法性
			return _str[i]; //返回对应字符
		}
		//改变容量，大小不变
		void reserve(size_t n)
		{
			if (n > _capacity) //当n大于对象当前容量时才需执行操作
			{
				char* tmp = new char[n + 1]; //多开一个空间用于存放'\0'
				strncpy(tmp, _str, _size + 1); //将对象原本的C字符串拷贝过来（包括'\0'）
				delete[] _str; //释放对象原本的空间
				_str = tmp; //将新开辟的空间交给_str
				_capacity = n; //容量跟着改变
			}
		}
		//尾插字符
		void push_back(char ch)
		{
			if (_size == _capacity) //判断是否需要增容
			{
				reserve(_capacity == 0 ? 4 : _capacity * 2); //将容量扩大为原来的两倍
			}
			_str[_size] = ch; //将字符尾插到字符串
			_str[_size + 1] = '\0'; //字符串后面放上'\0'
			_size++; //字符串的大小加一
		}
		
		//+=运算符重载, 返回引用
		string& operator+=(char ch)
		{
			push_back(ch); //尾插字符串
			return *this; //返回左值（支持连续+=）
		}
		//返回C类型的字符串
		const char* c_str()const
		{
			return _str;
		}
	private:
		char* _str;
		size_t _size;
		size_t _capacity;
	};
}


void func1(cl::string s)
{}
void func2(const cl::string& s)
{}
int main()
{
	cl::string s("hello world");
	func1(s);  //值传参；会调用拷贝构造。
	func2(s);  //左值引用传参

	s += 'X';  //左值引用返回
	return 0;
}

```

分析：
因为我们模拟实现，在string类的拷贝构造函数当中打印了提示语句，因此运行代码后通过程序运行结果就知道，**值传参时调用了string的拷贝构造函数**。
此外，因为string的`+=`运算符重载函数是**左值引用**返回的，因此在返回`+=`后的对象时不会调用拷贝构造函数，但如果将`+=`运算符重载函数改为**传值**返回，那么重新运行代码后你就会发现多了一次拷贝构造函数的调用。

我们都知道string的拷贝是深拷贝，深拷贝的代价是比较高的，我们应该尽量避免不必要的深拷贝操作，因此这里左值引用起到的作用还是很明显的。

## 左值引用的短板：引用作为返回值，不可以返回局部变量的引用

左值引用做返回值，如果函数返回的对象是一个局部变量，该变量出了函数作用域就被销毁了，这种情况下不能用左值引用作为返回值，只能**以传值的方式返回，这样就会调用拷贝构造函数**，这就是左值引用的短板。

```C++
namespace cl
{
	cl::string to_string(int value)
	{
		bool flag = true;
		if (value < 0)
		{
			flag = false;
			value = 0 - value;
		}
		cl::string str;
		while (value > 0)
		{
			int x = value % 10;
			value /= 10;
			str += (x + '0');
		}
		if (flag == false)
		{
			str += '-';
		}
		std::reverse(str.begin(), str.end());
		return str;
	}
}

int main()
{
	cl::string s = cl::to_string(1234); 
	// 此时调用to_string函数返回时，就一定会调用string的拷贝构造函数。
	return 0;
}

```

C++11提出**右值引用**就是为了解决左值引用的这个短板的，但解决方式并不是简单的将右值引用作为函数的返回值。

## 右值引用限定符（`&&`）
右值引用就是对右值的引用，给右值取别名，通过“&&”来声明。

### 语法
基本语法为：==类型 && 引用名 = 右值==。

```C++
int main()
{
	double x = 1.1, y = 2.2;
	
	//以下几个都是常见的右值
	10;
	x + y;
	fmin(x, y);

	//以下几个都是对右值的右值引用
	int&& rr1 = 10;
	double&& rr2 = x + y;
	double rr3 = fmin(x, y);
	return 0;
}

```

### 右值引用变量
同上，参考参数是右值引用。

### 右值引用限定符的成员函数

`&&`写在函数后：`&&` 限定表示**只能被右值（临时）对象调用**。

```C++
class MyClass {
    public:
        void bar() && { // 右值引用限定符
            // ...
        }
};
```

### 右值引用（指向栈上的临时对象）可以取地址

**右值本身不能取地址，但绑定到右值引用后，引用变量指向栈上的临时对象（临时对象在栈上开辟了空间），所以可以对右值引用变量进行取地址和修改（除非被 `const` 修饰）。**
> 注：绝大多数临时对象都在**栈区**。如果右值是**字符串字面量**（如 `"Hello"`），它存储在**静态存储区**。给 `"Hello"` 绑定引用后，取地址指向的是静态区，而非栈区。

**逻辑上**：引用是别名，没有独立空间（这是C++标准规定的语义）。
**物理上**：为了在内存中存储右值，编译器**必须**在==栈上==开辟一块内存。这块内存属于**临时对象本身**，而不是属于“引用变量”。引用变量只是指向这块内存的“标签”。

- **临时对象**（右值实体）：**有**独立的内存空间（在栈上）。
- **引用变量**（如 `int&& rref`）：**没有**独立的空间，它只是临时对象的别名。


总结：==右值引用本身只是别名（无空间），但其绑定的右值会被物化为有独立空间的对象（通常在栈上），因此通过这个引用变量，可以取地址和修改。也就是这个右值引用可以作为左值传递了。==
右值引用变量本身没有空间，但它所引用的那个临时对象有空间。取 `&rref` 时，取的是这块临时内存的地址，而不是“引用变量”的地址。
因为，==引用虽然是别名，其本质是指针，指向一段内存空间，右值开引用，就相当于给右值存储到了独立空间（通常在栈上），就可以取地址了（右值引用变量指向了这个空间）==。

```C++
#include <iostream>
using namespace std;

class MyClass {
public:
    int value;
    MyClass(int v) : value(v) {}
};

int main() {
    MyClass&& rref = MyClass(10);  // rref 是右值引用类型
    
    // ===== 关键：rref 本身是左值 =====
    MyClass* p = &rref;  // ✅ 可以取地址（只有左值才能取地址）
    cout << "rref is at: " << p << endl;
    
    // ===== rref 可以修改 =====
    rref.value = 20;  // ✅ 可以修改（rref本身是左值）
    cout << "Modified: " << rref.value << endl;
    
    // ===== 但是，std::move(rref) 是右值 =====
    MyClass&& rref2 = std::move(rref);  // std::move(rref) 是右值表达式
    
    return 0;
}
```




## 右值引用使用场景

## QA
### 左值引用可以引用右值吗？
**（1）左值引用不能引用右值**：
左值引用是对左值的引用，左值是可以改变的以及取地址的 ；右值是不可以取地址以及不可以改变的，因此左值引用不能引用右值。

**（2）const左值引用可以引用右值，也可以引用左值**：
因为const左值引用能够保证被引用的数据不会被修改。**因此const左值引用既可以引用左值，也可以引用右值**。
```C++
template<class T>
void func(const T& val)
{
	cout << val << endl;
}
int main()
{
	string s("hello");
	func(s);       //s为左值

	func("world"); //"world"为右值
	return 0;
}

```

### 右值引用可以引用左值吗？
(1) 右值引用只能引用右值，不能引用左值。
(2) 但是右值引用可以引用move语义后的左值。

```C++
int main()
{
	int a = 10;

	//int&& r1 = a;     //右值引用不能引用左值
	int&& r2 = move(a); //右值引用可以引用move以后的左值
	return 0;
}

```




# 移动语义(`std::move`)
在传统的 C++ 编程中，对象的复制和赋值可能会导致性能问题，特别是当对象包含大量数据或资源时。
为了解决这个问题，C++11 引入了移动语义，它允许我们“移动”对象而不是复制它们。

## 移动构造
### 移动构造和拷贝构造的区别

## 移动赋值
### 移动赋值和`=`赋值运算符重载

## move函数
右值引用虽然不能引用左值，但也不是完全不可以，当需要用右值引用引用一个左值时，可以通过move函数将左值转化为右值。

move函数的名字具有迷惑性，move函数实际并不能搬移任何东西，该函数唯一的功能就是**将一个左值强制转化为右值引用**，然后实现移动语义。

### 函数原型

```C++
template<class _Ty>
inline typename remove_reference<_Ty>::type&& move(_Ty&& _Arg) _NOEXCEPT
{
	//forward _Arg as movable
	return ((typename remove_reference<_Ty>::type&&)_Arg);
}

```

**说明一下：**

- move函数中_Arg参数的类型不是右值引用，而是==**万能引用**==。万能引用跟右值引用的形式一样，但是右值引用需要是确定的类型。
- 一个左值被move以后，它的资源可能就被转移给别人了，因此要慎用一个被move后的左值。

# 万能引用

模板中的&&不代表右值引用，而是万能引用，其既能接收左值又能接收右值。比如：
```C++
template<class T>
void PerfectForward(T&& t)
{
	//...
}
```

## 万能引用和右值引用的对比

右值引用需要是**确定的类型**，而万能引用是**根据传入实参的类型进行推导**。
如果传入的实参是一个左值，那么这里的形参t就是左值引用，如果传入的实参是一个右值，那么这里的形参t就是右值引用。

# 完美转发(`std::forward`)


# 参考
```bash
# C++11 ——— 右值引用和移动语义
https://blog.csdn.net/chenlong_cxy/article/details/126747523

# 《C++11》深入解析引用限定符：掌握左值与右值的关键技巧
https://lizhuo.blog.csdn.net/article/details/145142377

# 《C++11》移动构造函数的功能和用法：让你的代码更高效
https://lizhuo.blog.csdn.net/article/details/145041548

# 《C++11》右值引用深度解析：性能优化的秘密武器
https://lizhuo.blog.csdn.net/article/details/145015901

# C++11 Move Constructors and Move Assignment Operators 从入门到精通
https://lizhuo.blog.csdn.net/article/details/148480297
```