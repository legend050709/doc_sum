```table-of-contents
```
# 包装器
# function 包装器
## 介绍
function是一种函数包装器，也叫做适配器。它可以**对可调用对象进行包装**，C++中的==function本质就是一个类 模板==。
## function类模版
function类模板的原型如下：
```C++
template <class T> function;     // undefined
template <class Ret, class... Args>
class function<Ret(Args...)>; 
// Ret: 返回值类型；
// (Args...): 参数类型

```

模板参数说明：
- `Ret`：被包装的可调用对象的返回值类型。
- `Args...`：被包装的可调用对象的形参类型。

## function包装器的可调用对象
function包装器可以对可调用对象进行包装。
可调用对象包括：函数指针（函数名）、仿函数（函数对象）、lambda表达式、类的成员函数。

```C++
int f(int a, int b)
{
	return a + b;
}
struct Functor
{
public:
	int operator()(int a, int b)
	{
		return a + b;
	}
};
class Plus
{
public:
	static int plusi(int a, int b)
	{
		return a + b;
	}
	double plusd(double a, double b)
	{
		return a + b;
	}
};
int main()
{
	//1、包装函数指针（函数名）
	function<int(int, int)> func1 = f;
	cout << func1(1, 2) << endl;

	//2、包装仿函数（函数对象）
	function<int(int, int)> func2 = Functor();
	cout << func2(1, 2) << endl;

	//3、包装lambda表达式
	function<int(int, int)> func3 = [](int a, int b){return a + b; };
	cout << func3(1, 2) << endl;

	//4、类的静态成员函数
	//function<int(int, int)> func4 = Plus::plusi;
	function<int(int, int)> func4 = &Plus::plusi; //&可省略
	cout << func4(1, 2) << endl;

	//5、类的非静态成员函数
	function<double(Plus, double, double)> func5 = &Plus::plusd; //&不可省略
	cout << func5(Plus(), 1.1, 2.2) << endl;
	return 0;
}

```

### 注意事项

(1) 包装时指明返回值类型和各形参类型，然后将可调用对象赋值给function包装器即可，包装后function对象就可以像普通函数一样使用了。

(2) 取静态成员函数的地址可以不用取地址运算符“&”，但取非静态成员函数的地址必须使用取地址运算符“&”。

(3) 包装非静态的成员函数时需要注意，非静态成员函数的第一个参数是隐藏this指针，因此在包装时需要指明第一个形参的类型为类的类型。

## 通过function包装器来统一类型：function包装器对象作为函数模版的实参
### 范例
对于以下函数模板useF：

- 传入该函数模板的第一个参数可以是任意的可调用对象，比如函数指针、仿函数、lambda表达式等。
- useF中定义了静态变量count，并在每次调用时将count的值和地址进行了打印，可判断多次调用时调用的是否是同一个useF函数。

```C++
template<class F, class T>
T useF(F f, T x)
{
	static int count = 0;
	cout << "count: " << ++count << endl;
	cout << "count: " << &count << endl;

	return f(x);
}

```

在传入第二个参数类型相同的情况下，如果传入的可调用对象的类型是不同的，那么在编译阶段该函数模板就会被实例化多次。比如：
```C++
double f(double i)
{
	return i / 2;
}
struct Functor
{
	double operator()(double d)
	{
		return d / 3;
	}
};
int main()
{
	//函数指针
	cout << useF(f, 11.11) << endl;

	//仿函数
	cout << useF(Functor(), 11.11) << endl;

	//lambda表达式
	cout << useF([](double d)->double{return d / 4; }, 11.11) << endl;
	return 0;
}

```

### 分析

（1）由于函数指针、仿函数、lambda表达式是不同的类型，因此useF函数会被实例化出三份，三次调用useF函数所打印count的地址也是不同的。

（2）实际这里根本没有必要实例化出三份useF函数，因为三次调用useF函数时传入的可调用对象虽然是不同类型的，但这三个可调用对象的返回值和形参类型都是相同的。

### 通过funciton封装器来统一类型
这时就可以用包装器分别对着三个可调用对象进行包装，然后再用这三个包装后的可调用对象来调用useF函数，这时就只会实例化出一份useF函数。

根本原因就是因为包装后，这三个可调用对象都是相同的function类型，因此最终只会实例化出一份useF函数，该函数的第一个模板参数的类型就是function类型的。

代码如下：
```C++
int main()
{
	//函数名
	function<double(double)> func1 = func;
	cout << useF(func1, 11.11) << endl;

	//函数对象
	function<double(double)> func2 = Functor();
	cout << useF(func2, 11.11) << endl;

	//lambda表达式
	function<double(double)> func3 = [](double d)->double{return d / 4; };
	cout << useF(func3, 11.11) << endl;
	return 0;
}

```

这时三次调用useF函数所打印count的地址就是相同的，并且count在三次调用后会被累加到3，表示这一个useF函数被调用了三次。

## 范例：function包装器简化代码

求解逆波兰表达式的步骤如下：

- 定义一个栈，依次遍历所给字符串。
- 如果遍历到的字符串是数字则直接入栈。
- 如果遍历到的字符串是加减乘除运算符，则从栈定抛出两个数字进行对应的运算，并将运算后得到的结果压入栈中。
- 所给字符串遍历完毕后，栈顶的数字就是逆波兰表达式的计算结果。

### 原始代码

```C++
class Solution {
public:
	int evalRPN(vector<string>& tokens) {
		stack<int> st;
		for (const auto& str : tokens)
		{
			int left, right;
			if (str == "+" || str == "-" || str == "*" || str == "/")
			{
				right = st.top();
				st.pop();
				left = st.top();
				st.pop();
				switch (str[0])
				{
				case '+':
					st.push(left + right);
					break;
				case '-':
					st.push(left - right);
					break;
				case '*':
					st.push(left * right);
					break;
				case '/':
					st.push(left / right);
					break;
				default:
					break;
				}
			}
			else
			{
				st.push(stoi(str));
			}
		}
		return st.top();
	}
};

```

在上述代码中，我们通过switch语句来判断本次需要进行哪种运算，如果运算类型增加了，比如增加了求余、幂、对数等运算，那么就需要在switch语句的后面中继续增加case语句。

### function包装器精简后的代码

这种情况可以用包装器来简化代码。

- 建立各个运算符与其对应需要执行的函数之间的映射关系，当需要执行某一运算时就可以直接通过运算符找到对应的函数进行执行。
- 当运算类型增加时，就只需要建立新增运算符与其对应函数之间的映射关系即可。

代码如下：
```C++
class Solution {
public:
	int evalRPN(vector<string>& tokens) {
		stack<int> st;
		unordered_map<string, function<int(int, int)>> opMap = {
			{ "+", [](int a, int b){return a + b; } },
			{ "-", [](int a, int b){return a - b; } },
			{ "*", [](int a, int b){return a * b; } },
			{ "/", [](int a, int b){return a / b; } }
		};
		for (const auto& str : tokens)
		{
			int left, right;
			if (str == "+" || str == "-" || str == "*" || str == "/")
			{
				right = st.top();
				st.pop();
				left = st.top();
				st.pop();
				st.push(opMap[str](left, right));
			}
			else
			{
				st.push(stoi(str));
			}
		}
		return st.top();
	}
};

```

需要注意的是，这里建立的是运算符与function类型之间的映射关系，因此无论是函数指针、仿函数还是lambda表达式都可以在包装后与对应的运算符进行绑定。

## function包装器的意义

- 将可调用对象的类型进行统一，便于我们对其进行统一化管理。
- 包装后明确了可调用对象的返回值和形参类型，更加方便使用者使用。



# bind 包装器
bind也是一种函数包装器，也叫做适配器。
它可以**接受一个可调用对象，生成一个新的可调用对象来“适应”原对象的参数列表**，C++中的==bind本质是一个函数模板==。

## 原型
bind函数模板的原型如下：
```C++
template <class Fn, class... Args>
/* unspecified */ bind(Fn&& fn, Args&&... args);

template <class Ret, class Fn, class... Args>
/* unspecified */ bind(Fn&& fn, Args&&... args);

```

模板参数说明：
- `fn`：可调用对象。
- `args...`：要绑定的参数列表：值或占位符。

## 调用bind的一般形式

调用bind的一般形式为：`auto newCallable = bind(callable, arg_list);`

# 可调用对象总结
可调用对象包括：函数指针（函数名）、仿函数（函数对象）、lambda表达式、类的成员函数、被包装器包装后的可调用对象等。

# 参考
```bash
# C++11：从源码探究std::function的原理与使用
https://zhuanlan.zhihu.com/p/718427256
```