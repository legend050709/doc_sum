```table-of-contents
```
# 左值和右值
## 左值（Lvalue）
## 右值（Rvalue）
右值通常是**临时对象、常量**。

## 区分左值和右值
可以通过以下几个方面来区分左值和右值：

- **（1）可寻址性（可取地址）**：
左值对应具体的内存地址，可通过取地址操作符（&）获取其地址；而右值不能取地址。
例如：
```C++
int x = 10; // x 是左值，可取其地址 
int *p = &x; 

// &1; // 非法，1 是右值，无地址int a = 5; 

a = 20; // 合法，a 是左值 

const int b = 10; // 
b = 30; // 非法，b 是 const 左值，不可修改
```

- **（2）可修改性（除非被 const 限定）**：
左值通常可被赋值，除非被声明为 const；右值不能作为赋值目标。例如：

- **（3）生命周期**：
左值代表的对象的生命周期超出其所在的表达式；右值的生命周期通常仅限于当前表达式。

# 左值引用 和右值引用
参考：[C++11 右值引用：从入门到精通](https://cloud.tencent.com/developer/article/2528826)
参考：[《C++11》深入解析引用限定符：掌握左值与右值的关键技巧](https://cloud.tencent.com/developer/article/2486860)

## 左值引用限定符(`&`)
### 定义和语法
基本语法为：**类型 & 引用名 = 左值**。

范例：
```C++
int x = 10; 
int &ref = x; // ref 是 x 的别名，x 是左值；ref是左值引用
ref = 20; // 修改 ref 即修改 x 的值
```

#### const 左值引用
基本语法为：**const 类型 & 引用名 = 左值**。

### 左值引用变量
### 左值引用成员函数
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
### 语法
基本语法为：==类型 && 引用名 = 右值==。

### 右值引用变量
### 右值引用成员函数
```C++
class MyClass {
    public:
        void bar() && { // 右值引用限定符
            // ...
        }
};
```

## 右值引用使用场景

## QA
### 左值引用可以引用右值吗？
**（1）左值引用不能引用右值**：
左值引用是对左值的引用，左值是可以改变的以及取地址的 ；右值是不可以取地址以及不可以改变的，因此左值引用不能引用右值。

**（2）const左值引用可以引用右值**：
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
- (1) 右值引用只能引用右值，不能引用左值。
- (2) 但是右值引用可以引用move以后的左值。

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