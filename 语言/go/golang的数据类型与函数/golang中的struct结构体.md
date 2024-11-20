```table-of-contents
```

# golang实现面向对象
## 背景
go中没有面向对象(oop)的说法，它不像其它面向对象编程语言一样，有类、继承、对象、构造函数、析构函数这些概念，但是可以使用结构体(struct)来实现面向对象的特性和功能，它类似于其它编程语言中的类，不过和传统的面向对象有很大的区别.

## golang实现面向对象的三大特性
知道面向对象三大特性：封装、继承、多态，在golang中都可以通过struct结构体来实现

### 封装
把数据存储到对象的内部，隐藏内部细节，对外可以可见或不可见。
按标识符规则：大写开头表示外部（包外）可见，小写则外部（包外）不可见。

### 继承

把公共的字段和方法提取出来复用。
通过在结构体嵌入匿名结构体字段方式实现，这种方式也叫**结构体组合(composite)**。

### 多态

鸭子类型，通过接口interface实现，接口是方法的类型，只要实现接口中的方法即可。
接口是go中的重要的特性，通过接口（interface）关联，耦合性低，非常灵活，所以也叫面向接口编程。


# 结构体标签Struct Tag
## 介绍
在 Go 语言中，struct 是一种常见的数据类型，它可以用来表示复杂的数据结构。在 struct 中，我们可以定义多个字段，每个字段可以有不同的类型和名称。

除了这些基本信息之外，Go 还提供了 struct tags，也称为结构体注释（Struct Annotation），它可以用来指定 struct 中每个字段的元信息。

## 使用

struct tags 使用还是很广泛的，特别是在 json 序列化，或者是数据库 ORM 映射方面。

![](attachments/Pasted%20image%2020241023101611.png)

### Tag可选的字段

- “-”：表示不需要解析这个字段
- “omitempty”：当字段为空(默认值)时，不要解析这个字段；比如是false、0、nil或者长度为0的array、map、slice、string等。


### 使用反引号包裹
在声明 struct tag 时，使用反引号 `` ` `` 包围 tag 的值，可以防止转义字符的影响，使 tag 更容易读取和理解。

结构体标记使用`key:"value"`的格式来定义；
**其中key是标记的名称（即tag名称），value是该标记的值(即tag值)**。
key 一般指的是要使用的包名，比如这里的json表示这个Name字段会被 `encoding/json`包使用和处理。

```go
type User struct {
    Name string `json:"name"`
    Age int `json:"age"`
}
```
### 避免使用空格
在 struct tag 中，应该避免使用空格，特别是在 tag 名称和 tag 值之间。使用空格可能会导致编码或解码错误，并使代码更难以维护。

例如：
```go
// 不规范的写法
type User struct {
    ID    int    `json: "id" db: "id"`
    Name  string `json: "name" db: "name"`
    Email string `json: "email" db: "email"`
}

// 规范的写法
type User struct {
    ID    int    `json:"id" db:"id"`
    Name  string `json:"name" db:"name"`
    Email string `json:"email" db:"email"`
}
```


### 一个字段含有多个标记
一个结构体字段可以有多个标记，每个标记之间使用空格分隔。
```go
type User struct {
    Name string `json:"name" xml:"name"`
}

type User2 struct {
    ID    int    `json:"id" db:"id"`
    Name  string `json:"name" db:"name"`
    Email string `json:"email" db:"email"`
}

```

### 避免重复

在 struct 中，应该避免重复使用同一个 tag 名称(即 tag的value值)。如果重复使用同一个 tag 名称，编译器可能会无法识别 tag，从而导致编码或解码错误。
例如：
```go
// 不规范的写法
type User struct {
    ID    int    `json:"id" db:"id"`
    Name  string `json:"name" db:"name"`
    Email string `json:"email" db:"name"`
}

// 规范的写法
type User struct {
    ID    int    `json:"id" db:"id"`
    Name  string `json:"name" db:"name"`
    Email string `json:"email" db:"email"`
}

```

### 使用标准化的 tag 名称

为了使 struct tag 更加标准化和易于维护，应该使用一些标准化的 tag 名称。

例如，对于序列化和反序列化，可以使用 `json`、`xml`、`yaml` 等；对于数据库操作，可以使用 `db`。

```go
type User struct {
    ID       int    `json:"id" db:"id"`
    Name     string `json:"name" db:"name"`
    Password string `json:"-" db:"password"` // 忽略该字段
    Email    string `json:"email" db:"email"`
}

```
其中，`Password` 字段后面的 `-` 表示忽略该字段，也就是说该字段不会被序列化或反序列化。


### 多个 tag 值
如果一个字段需要指定多个 tag 值，可以使用 `,` 将多个 tag 值分隔开。
例如：
```go
type User struct {
    ID        int    `json:"id" db:"id"`
    Name      string `json:"name" db:"name"`
    Email     string `json:"email,omitempty" db:"email,omitempty"`
}

```
其中 `omitempty` 表示如果该字段值为空，则不序列化该字段。




## 原理
Go 的反射库提供了一些方法，可以让我们在程序运行时获取和解析结构体标签。
介绍这些方法之前，先来看看 `reflect.StructField` ，它是描述结构体字段的数据类型。定义如下：
```go
type StructField struct {
    Name      string      // 字段名
    Type      Type        // 字段类型
    Tag       StructTag   // 字段标签
}

```

在结构体的反射中，我们经常使用 `reflect.TypeOf` 获取类型信息，然后使用 `Type.Field` 或 `Type.FieldByName()` 获取结构体字段的 `reflect.StructField`，然后根据 `StructField` 中的信息做进一步处理。

### 获取tag
```go
package main

import (
    "fmt"
    "reflect"
)

type User struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

type Manager struct {
    Title string `json:"title"`
    User
}

func main() {
    m := Manager{Title: "Manager", User: User{Name: "Alice", Age: 25}}

    mt := reflect.TypeOf(m)

    // 获取 User 字段的 reflect.StructField
    userField, _ := mt.FieldByName("User")
    fmt.Println("Field 'User' exists:", userField.Name, userField.Type)

    // 获取 User.Name 字段的 reflect.StructField
    nameField, _ := userField.Type.FieldByName("Name")
    tag := nameField.Tag.Get("json")
    fmt.Println("User.Name tag:", tag)
}

结果：
Field 'User' exists: User {string int}
User.Name tag: "name"

```

## struct tags 的应用

使用 struct tag 的主要优势之一是可以在**运行时通过反射来访问和操作 struct 中的字段**。

### 应用场景

#### 携带额外信息

比如在 Go Web 开发中，常常需要将 HTTP 请求中的参数绑定到一个 struct 中。这时，我们可以使用 struct tag 指定每个字段对应的参数名称、验证规则等信息。
在接收到 HTTP 请求时，就可以使用反射机制读取这些信息，并根据信息来验证参数是否合法。

#### 序列化和反序列化

另外，在将 struct 序列化为 JSON 或者其他格式时，我们也可以使用 struct tag 来指定每个字段在序列化时的名称和规则。

#### 提高代码的可读性和可维护性
此外，使用 struct tag 还可以提高代码的**可读性**和**可维护性**。

在一个大型的项目中，struct 中的字段通常会包含很多不同的元信息，比如数据库中的表名、字段名、索引、验证规则等等。

如果没有 struct tag，我们可能需要将这些元信息放在注释中或者在代码中进行硬编码。这样会让代码变得难以维护和修改。
而**使用 struct tag 可以将这些元信息与 struct 字段紧密关联起来**，使代码更加清晰和易于维护。

## 常用的 struct tags

在 Go 的官方 wiki 中，有一个常用的 struct tags 的库的列表，我复制在下面了，感兴趣的同学可以看看源码，再继续深入学习。

|Tag|Documentation|
|---|---|
|xml|[https://pkg.go.dev/encoding/xml](https://pkg.go.dev/encoding/xml)|
|json|[https://pkg.go.dev/encoding/json](https://pkg.go.dev/encoding/json)|
|asn1|[https://pkg.go.dev/encoding/asn1](https://pkg.go.dev/encoding/asn1)|
|reform|[https://pkg.go.dev/gopkg.in/reform.v1](https://pkg.go.dev/gopkg.in/reform.v1)|
|dynamodb|[https://docs.aws.amazon.com/sdk-for-go/api/service/dynamodb/dynamodbattribute/#Marshal](https://docs.aws.amazon.com/sdk-for-go/api/service/dynamodb/dynamodbattribute/#Marshal)|
|bigquery|[https://pkg.go.dev/cloud.google.com/go/bigquery](https://pkg.go.dev/cloud.google.com/go/bigquery)|
|datastore|[https://pkg.go.dev/cloud.google.com/go/datastore](https://pkg.go.dev/cloud.google.com/go/datastore)|
|spanner|[https://pkg.go.dev/cloud.google.com/go/spanner](https://pkg.go.dev/cloud.google.com/go/spanner)|
|bson|[https://pkg.go.dev/labix.org/v2/mgo/bson](https://pkg.go.dev/labix.org/v2/mgo/bson), [https://pkg.go.dev/go.mongodb.org/mongo-driver/bson/bsoncodec](https://pkg.go.dev/go.mongodb.org/mongo-driver/bson/bsoncodec)|
|gorm|[https://pkg.go.dev/github.com/jinzhu/gorm](https://pkg.go.dev/github.com/jinzhu/gorm)|
|yaml|[https://pkg.go.dev/gopkg.in/yaml.v2](https://pkg.go.dev/gopkg.in/yaml.v2)|
|toml|[https://pkg.go.dev/github.com/pelletier/go-toml](https://pkg.go.dev/github.com/pelletier/go-toml)|
|validate|[https://github.com/go-playground/validator](https://github.com/go-playground/validator)|
|mapstructure|[https://pkg.go.dev/github.com/mitchellh/mapstructure](https://pkg.go.dev/github.com/mitchellh/mapstructure)|
|parser|[https://pkg.go.dev/github.com/alecthomas/participle](https://pkg.go.dev/github.com/alecthomas/participle)|
|protobuf|[https://github.com/golang/protobuf](https://github.com/golang/protobuf)|
|db|[https://github.com/jmoiron/sqlx](https://github.com/jmoiron/sqlx)|
|url|[https://github.com/google/go-querystring](https://github.com/google/go-querystring)|
|feature|[https://github.com/nikolaydubina/go-featureprocessing](https://github.com/nikolaydubina/go-featureprocessing)|

## 范例
```go
type User struct {
    Name string `json:"name"`
    Age int `json:"age"`
}
```
注意如上结构体中反引号引起来的内容就是Golang中的Struct Tag，接下来看一下它的作用,如果输出json格式
```go
u := &User{Name: "xiaohong", Age: "18"}
j, _ := json.Marshal(u)
fmt.Println(string(j))

输出如下内容:
{"name": "xiaohong","age": 18}
```

如果去掉StructTag会输出什么呢，看如下例子:
```go
type User struct {
  Name string
  Age int
}
u := &User{Name: "xiaohong", Age: "18"}
j, _ := json.Marshal(u)
fmt.Println(string(j))

输出如下内容
{"Name": "xiaohong","Age": 18}
```

# struct的导出和暴露
## struct 导出原则
struct的属性是否被导出，也遵循大小写的原则：首字母大写的被导出，首字母小写的不被导出。

所以：

1. **如果struct名称首字母是小写的，这个struct不会被导出。连同它里面的字段也不会导出，即使有首字母大写的字段名**。
2. **如果struct名称首字母大写，则struct会被导出，但只会导出它内部首字母大写的字段，那些小写首字母的字段不会被导出**。

也就是说，struct的导出情况是混合的。

注意：**如果struct嵌套了，那么即使被嵌套在内部的struct名称首字母小写，也能访问到它里面首字母大写的字段**。

### 范例

```go
type animal struct{
    name string
    Speak string
}
type Horse struct {
    animal
    sound string
}
```

Horse中嵌套的animal是小写字母开头的，但Horse是能被导出的，所以能在其它包中使用Horse struct，其他包也能访问到animal中的Speak属性。


## struct不对外暴露
### 背景
很多时候，不应该将某包(如包abc)中的struct(如animal)直接暴露给其它包，暴露意味着打开了那个"黑匣子"，所以struct会以小写字母开头，不将其导出。

这时在外界其它包中构建包abc的animal，就没法直接通过以下几种方式实现：
```text
var xxx abc.animal
new(abc.animal)
&abc.animal{...}
abc.animal{...}
```

例如，下面的是错误的：
```go
// abc/abc.go文件内容：
package abc

type animal struct{
    name string
    Speak string
}

// test.go内容：
package main

import "./abc"

func main() {
    // 全都错误
    var t1 abc.animal
    t2 := new(abc.animal)
    t3 := &abc.animal{}
    t4 := abc.animal{}
}
```

### 设置可导出的函数

#### 构建函数

那么如何在外界构建隐藏起来的struct实例？这时可以在abc包中写一个可导出的函数，通过这个函数来构建struct实例。

```go
// abc/abc.go文件内容：
package abc

type animal struct{
    name string
    Speak string
}

func NewAnimal() *animal{
    a := new(animal)
    return a
}


// test.go内容：
package main

import (
    "fmt"
    "./abc"
)

func main() {
    t1 := abc.NewAnimal()
//  t1.name = "haha"    // 无法访问name属性
    t1.Speak = "hhhh"
    fmt.Println(t1.Speak)
}
```

上面的代码一切正常，在main包中可以通过NewAnimal()构建出abc包中未导出的animal struct。注意，上面NewAnimal()中是使用new()函数构造实例的，它返回的是实例的指针，至于如何构造实例，完全可以根据自己的需求，但对于struct类型来说，一般都是使用指针的，也就是完全可以将new()通用化。

#### set函数

写一个专门的可导出方法来设置实例的name属性。
```go
func NewAnimal() *animal{
    a := new(animal)
    return a
}

func (a *animal) SetName(name string){
    a.name = name
}

```

需要注意的是，上面的setter类方法`SetName()`不能同时被2个或多个线程修改，否则值被覆盖，出现线程安全问题，可以使用sync包或者goroutine和channel来解决这个问题。

#### get函数

在abc包中继续写一个可导出的方法，该方法用于获取实例的name属性：
```go
// abc/abc.go中添加：
func (a *animal) GetName() string {
    return a.name
}


```

## 嵌套struct中的方法导出问题
### 原则
当内部struct（即匿名struct 结构体成员）嵌套进外部struct（含有匿名struc成员的外部struct）时，内部struct（匿名struct 结构体成员）的方法也会被嵌套，也就是说外部struct拥有了内部struct的方法。

### 注意

需要注意方法的首字母大小写问题。
内、外struct的定义在同一包内时：
**（1）如果直接在该包内构建外部struct实例**：
外部struct实例是可以直接访问内部struct（匿名struct 结构体成员）的所有方法的。

**（2）如果在其它包内构建外部struct实例**：
该实例将无法访问内部struct（匿名struct 结构体成员）中首字母小写的方法。

### 范例

**(1) 以下是在同一个包内测试**
外部实例可以直接调用内部struct的方法
```go
package main

import (
    "fmt"
)

type person struct {
    name string
    age  int
}

// 未导出方法
func (p *person) speak() {
    fmt.Println("speak in person")
}

// 导出的方法
func (p *person) Sing() {
    fmt.Println("Sing in person")
}

// Admin exported
type Admin struct {
    person
    salary int
}

func main() {
    a := new(Admin)
    a.speak()  // 正常输出
    a.Sing()   // 正常输出
}


执行结果时`a.speak()`和`a.Sing()`都正常输出。
```


**(2) 以下是不同包内测试**
struct定义在abc/abc.go文件中，main在test.go中，它们的目录结构如下：

```go
$ tree .
.
├── abc
│   └── abc.go
├── test.go


$ cat abc/abc.go
package abc

import "fmt"

// 未导出的person
type person struct {
    name string
    age  int
}

// 未导出的方法
func (p *person) speak() {
    fmt.Println("speak in person")
}

// 导出的方法
func (p *person) Sing() {
    fmt.Println("Sing in person")
}

// Admin exported
type Admin struct {
    person
    salary int
}


$ cat test.go
package main

import "./abc"

func main() {
    a := new(abc.Admin)

    // 下面报错
//  a.speak()

    // 下面正常
    a.Sing()
}
```

# 参考
```
# go基础系列：结构struct
https://www.cnblogs.com/f-ck-need-u/p/9834459.html

# Go基础系列：struct和嵌套struct
https://www.cnblogs.com/f-ck-need-u/p/9882315.html

# Go基础系列：struct的导出和暴露问题
https://www.cnblogs.com/f-ck-need-u/p/9887233.html


```