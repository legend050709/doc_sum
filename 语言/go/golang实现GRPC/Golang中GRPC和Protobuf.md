```table-of-contents
```
# RPC
## RPC介绍 

RPC (Remote Procedure Call)从字面上理解，就是调用一个方法，但是这个方法不是运行在本地，而是运行在远端的服务器上。也就是说，客户端应用可以像调用本地函数一样，直接调用运行在远端服务器上的方法。


## RPC框架
本地的同一个进程中执行该方法：
![](attachments/Pasted%20image%2020241219152402.png)

远程的服务器的进程执行该方法：
![](attachments/Pasted%20image%2020241219152407.png)

下图中的绿色背景部分，就是 RPC 框架需要做的事情。
![](attachments/Pasted%20image%2020241219153017.png)

对于应用程序来说，Client 端代理就相当于是服务的“本地代理人”，至于这个代理人是怎么来处理如何通信、如何数据打包、如何通用、灵活的远程调用等几个问题、然后从真正的服务器上得到结果，这就不需要应用程序来关心了。

结合下图，从应用程序的角度看，它只是执行了一个函数调用(步骤1)，然后就立刻得到了结果(步骤10)，这中间的所有步骤(2-9)，全部是 RPC 框架来处理，而且能够灵活的处理各种不同的请求、响应数据。
![](attachments/Pasted%20image%2020241219152439.png)


RPC 远程调用框架的最终的目的：
> 1. 服务器端利用这个框架，在网络上提供服务；
> 2. 客户端利用这个框架，远程调用位于服务器上的函数；


## PRC通信
对于单独部署，独立运行的微服务实例而言，在业务需要时，需要与其他服务进行通信，这种通信方式是进程之间的通讯方式（inter-process communication，简称IPC）。

前文已经描述过，IPC有两种实现方式，分别为：**同步过程调用、异步消息响应**。
> （1）**同步过程调用**：同步过程调用是指一个进程（客户端）在调用另一个进程（服务器）的服务时，会阻塞等待服务器处理完成并返回结果。
> （2）**异步消息响应**：比如消息队列(message queue)。异步消息调用是指一个进程发送请求到另一个进程后，不会等待结果的返回，而是继续执行后续操作。服务器在处理完请求后，可以通过回调、事件或消息队列将结果发送回客户端。

在同步过程调用的具体实现中，有一种实现方式为RPC通信方式，远程过程调用（英语：Remote Procedure Call，缩写为 RPC）。远程过程调用是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一台计算机的子程序，而程序员无需额外地为这个交互作用编程。如果涉及的软件采用面向对象编程，那么远程过程调用亦可称作远程调用或远程方法调用。

**简单地说RPC就是能使应用像调用本地方法一样的调用远程的过程或服务。** 
很显然，这是一种`client-server`的交互形式，调用者(`caller`)是`client`,执行者(`executor`)是`server`。典型的实现方式就是`request–response`通讯机制。

## RPC实现步骤

![](attachments/Pasted%20image%2020241219152215.png)

一个正常的RPC过程可以分为一下几个步骤：

- 1、client调用client stub，这是一次本地过程调用。
    
- 2、client stub将参数打包成一个消息，然后发送这个消息。打包过程也叫做marshalling。
    
- 3、client所在的系统将消息发送给server。
    
- 4、server的的系统将收到的包传给server stub。
    
- 5、server stub解包得到参数。 解包也被称作 unmarshalling。
    
- 6、server stub调用服务过程。返回结果按照相反的步骤传给client。

## RPC实现需要解决的问题
### Call ID映射
#### 问题
远端进程中间可以包含定义的多个函数，本地客户端该如何告知远端进程程序调用特定的某个函数呢？

#### 思路
在RPC调用过程中，所有的函数都需要有一个自己的ID。
开发者在客户端（调用端）和服务端（被调用端）分别维护一个{函数<-->`Call ID`}的对应表。两者的表不一定完全相同，但是相同的函数对应的`Call ID`必须相同。

当客户端需要进行远程调用时，调用者通过映射表查询想要调用的函数的名称，找到对应的`Call ID`，然后传递给服务端；
服务端也通过查表，来确定客户端所需要调用的函数，然后执行相应函数的代码。

### 序列化和反序列化
#### 问题
客户端如何把参数传递给远程调用的函数呢？
在本地调用中，我们只需要把参数压到栈里，然后让函数自己去栈里读就行。
但是在远程过程调用时，客户端跟服务端是不同的进程，不能通过内存来传递参数。甚至有时候客户端和服务端使用的都不是同一种语言（比如服务端用C++，客户端用Java或者Python）。

#### 介绍
**思路**：
这时候就需要客户端把参数先转成一个字节流，传给服务端后，再把字节流转成自己能读取的格式。这个过程叫序列化和反序列化。

**序列化**：
将结构数据或对象转换成能够被存储和传输（例如网络传输）的格式，同时应当要保证这个序列化结果在之后（可能在另一个计算环境中）能够被重建回原来的结构数据或对象。

#### 解决方法
利用 `protobuf`，可以解决了序列化的问题。

### 网络传输
####  介绍
远程调用往往用在网络上，客户端和服务端是通过网络连接的。所有的数据都需要通过网络传输，因此就需要有一个网络传输层。
网络传输层需要把`Call ID`和序列化后的参数字节流传递给服务端，然后在把序列化后的调用结果传回给客户端，完成这种数据传递功能的被成为传输层。大部分的网络传输成都使用TCP协议，属于长连接。

#### 实现
网络传输部分，使用的是 HTTP 协议。
 
# protobuf
## protobuf 介绍
`protobuf` 即 `Protocol Buffers`，是一种轻便高效的结构化数据存储格式，**语言无关、平台无关、可扩展的序列化结构数据格式。**。

`protobuf` 性能和效率大幅度优于 `JSON`、`XML` 等其他的结构化数据格式。`protobuf` 是**以二进制方式存储**的，**占用空间小**，但也带来了**可读性差**的缺点。
`protobuf` 在通信协议和数据存储等领域应用广泛。例如著名的分布式缓存工具 `Memcached` 的 Go 语言版本`groupcache` 就使用了 `protobuf` 作为其 `RPC` 数据格式。

`Protobuf` 在 `.proto` 定义需要处理的结构化数据，可以通过 `protoc` 工具，将 `.proto` 文件转换为 `C++`、`Golang`、`Java`、`Python` 等多种语言的代码，**兼容性好**，易于使用。

## protobuf 在GRPC中的应用
gRPC默认使用`protocol buffers`作为交换数据序列化的机制，即gRPC底层依赖`protobuf`。
在gRPC框架中，PB主要有三个作用：
1）可以用来定义（消息）数据结构；
2）可以用来定义服务接口；
3）可以通过protobuf序列化和反序列化，提升传输效率。


## protobuf的优点
### 性能好/效率高
- **时间维度**：
采用XML格式对数据进行序列化时，时间消耗上性能尚可；
对于使用XML格式对数据进行反序列化时的时间花费上，耗时长，性能差。

- **空间维度**：
XML格式为了保持较好的可读性，引入了一些冗余的文本信息。
所以在使用XML格式进行存储数据时，也会消耗空间。

整体而言，Protobuf**以二进制方式存储**，比XML小3到10倍，快20到100倍。

### 代码生成机制
#### 代码生成机制的含义
在Go语言中，可以通过定义结构体封装描述一个对象，并定义一个新的结构体对象。比如定义Person结构体，并存放于`Person.go`文件：
```go
type Person struct{
    Name string
    Age int
    Sex int
}
```

在分布式系统中，因为程序代码是分开部署，比如分别为A、B两个系统。
A系统在调用B系统时，无法直接采用代码的方式进行调用，因为A系统中不存在B系统中的代码。
因此，A系统只负责将调用和通信的数据以二进制数据包的形式传递给B系统，由B系统根据获取到的数据包，自己构建出对应的数据对象。
B 利用编译器，提前根据`proto`数据文件自动生成结构体定义和相关方法的文件的机制被称作代码生成机制。

#### 代码生成机制的优点

代码生成机制能够极大解放开发者编写数据协议解析过程的时间，提高工作效率；
其次，易于开发者维护和迭代，当需求发生变更时，开发者只需要修改对应的数据传输文件内容即可完成所有的修改。

### 支持"向后兼容"和"向前兼容"
支持前后兼容是非常重要的一个特点，在庞大的系统开发中，往往不可能统一完成所有模块的升级，为了保证系统功能正常不受影响，应最大限度保证通讯协议的向前兼容和向后兼容。

#### 向后兼容
在软件开发迭代和升级过程中，"后"可以理解为新版本，越新的版本越靠后；而“前”意味着早起的版本或者先前的版本。
向“后”兼容即是说当系统升级迭代以后，仍然可以处理老版本的数据业务逻辑。

#### 向前兼容
向前兼容即是系统代码未升级，但是接受到了新的数据，此时老版本生成的系统代码可以处理接收到新类型的数据。


### 支持多种编程语言
 `Protobuf`不仅仅`Google`开源的一个数据协议，还有很多种语言的开源项目实现。在`Google`官方发布的`Protobuf`的源代码中包含了`C++`、`Java`、`Python`、`GO`等语言。


## Protobuf 缺点

### 缺乏自描述
二进制数据本身不携带字段名信息，离开 `.proto` 文件数据就失去了语义，不利于长期存档或跨系统数据流转。

诸如`XML`语言是一种自描述的标记语言，即字段标记的同时就表达了内容对应的含义。而`Protobuf`协议不是自描述的，`Protobuf`是通过二进制格式进行数据传输，开发者面对二进制格式的`Protobuf`，没有办法知道所对应的真实的数据结构。
因此在使用`Protobuf`协议传输时，必须配备对应的`proto`配置文件。

### 可读性与调试性差
为了提高性能，Protobuf采用了二进制格式进行编码。
Protobuf 序列化后是二进制格式，人眼无法直接阅读，无法直接用文本编辑器、抓包工具阅读，调试时需要借助额外工具，排查问题成本高。


### 强依赖 Schema（.proto 文件）
- 双方必须共享同一份 `.proto` 文件才能通信；
- Schema 变更需要重新编译生成代码，跨团队协作摩擦大

### 不适合动态结构（如不确定字段的业务数据）
Protobuf 的强项是**「字段固定、结构稳定」的内部服务通信；一旦业务要求「字段由运行时数据决定」**，它的强 Schema 就从优点变成了枷锁。

#### 范例
场景：电商平台的「商品属性」系统
不同类型的商品，属性字段完全不同：

- 手机：CPU、内存、屏幕尺寸、电池容量
- 衣服：颜色、尺码、材质
- 食品：保质期、配料、净重

用 Protobuf 来表达，会很痛苦。

**方案一：穷举所有字段（字段爆炸）**
```protobuf
message Product {
  string name = 1;

  // 手机属性
  string cpu = 2;
  int32  ram_gb = 3;
  float  screen_inch = 4;
  int32  battery_mah = 5;

  // 衣服属性
  string color = 6;
  string size = 7;
  string material = 8;

  // 食品属性
  string ingredients = 9;
  int32  shelf_days = 10;
  float  weight_g = 11;

  // 明天产品经理又加了新品类，继续加字段...
  // 每次新增都要改 proto、重新编译、重新部署所有服务
}

问题：
一件衬衫的数据里，cpu / battery_mah 全是空的，大量字段永远是零值
每新增品类，所有服务都要跟着改 .proto 并重新部署，牵一发动全身
```


**方案二：用 `map<string, string>` 凑合**
```protobuf
message Product {
  string name = 1;
  map<string, string> attributes = 2; // 把所有属性塞进 map
}

问题：

类型全丢失，battery_mah 本来是 int，现在存成字符串 "4000"，消费方要自己猜类型、手动转换
没有任何 Schema 约束，拼写错 "baterry_mah" 编译期完全不报错
Protobuf 的 map 底层是 repeated message，性能比原生 map 差
```

**方案三：用 Any / oneof**
```protobuf
message PhoneAttrs  { string cpu = 1; int32 ram = 2; }
message ClothAttrs  { string color = 1; string size = 2; }

message Product {
  string name = 1;
  oneof attrs {
    PhoneAttrs  phone = 2;
    ClothAttrs  cloth = 3;
    // 每新增品类还是要改 proto...
  }
}

问题：
本质上还是要穷举，动态扩展性没有解决
Any 类型需要手动 pack/unpack，代码繁琐且容易出错
```


### 小数据场景优势不明显，反而更重
- 序列化 / 反序列化有固定开销
- 字段极少时，体积和性能不如简单文本格式


## 安装`protobuf`包
使用protobuf需要先安装对应的包。
```go
go get -u github.com/golang/protobuf
```

## Protobuf序列化原理

- **Serialization（序列化）**：
是将数据结构或对象转换为可存储或传输的格式的过程，例如将对象转换为字节流或字符串。

- **Deserialization（反序列化）**：
是将序列化后的数据转换回原始数据结构或对象的过程。


Protobuf的message中有很多字段，每个字段的格式为：**修饰符 字段类型 字段名 = 标识号; **


# `.proto`文件讲解
## syntax
```pb
syntax = "proto3";

option go_package = "./";
package main;
// this is a comment
message Student {
  string name = 1;
  bool male = 2;
  repeated int32 scores = 3;
}
```
`protobuf` 有2个版本，默认版本是 proto2，如果需要 proto3，则需要在非空非注释第一行使用 `syntax = "proto3"` 标明版本。

## go_package 和 package

**package**：用于描述 `pb`文件,在被其他的`pb`文件引用时起作用;  
**option go_package**：用于指定生成的`.pb.go`文件的`package`的路径以及名称，对于确保生成的代码可以被其他 Go 代码正确引用非常重要。

### package
**package**是pb文件的包名，每个pb文件都要属于一个package，多个pb文件可以属于同一个package。（==很类似于go文件和go中的package==）。


#### 范例

一个 `aa.proto`文件中引入了其他的`proto`文件。如下：
```text
import "pfoo/foo.proto"

注：这里`pfoo/foo.proto`是相对路径, 取决于使用protoc -I  (大写i)传入时的搜索路径。
```
假设这个`foo.proto` 声明了
```text
package demo
```
那么我们在其他包就要通过`demo.xxx`的方式来引用`foo.proto`这个文件(`package demo`这个包)所声明的类型。

#### 小结


### option go_package

`go_package` 是  `.proto` 文件中的一个选项，用于指定生成的 `.pb.go`文件中的包名。

我们来看看`google\protobuf\descriptor.proto`中给`go_package`的定义:
```text
// Sets the Go package where structs generated from this .proto will be
// placed. If omitted, the Go package will be derived from the following:
//   - The basename of the package import path, if provided.
//   - Otherwise, the package statement in the .proto file, if present.
//   - Otherwise, the basename of the .proto file, without extension.
optional string go_package = 11;

```

格式如下所示：
```bash
option go_package = "{out_path};out_go_package";

注：这里指定的 out_path 并不是绝对路径，只是相对路径或者说只是路径的一部分，
和 protoc 的 `--go_out` 拼接后才是完整的路径。

这里是逗号（；），使用分号将包路径和包名分开。  
`;`的前一个参数out_path：用于指定生成的`.go`文件的位置；
后一个参数out_go_package：指定生成的 `.go `文件的 package。
```

`go_package` 如果设置了，如果其他 `go`文件`import` 了这个`pb`文件生成的`.pb.go`文件对应的 `package`时，那么他们就会使用逗号（；）前面的作为`go package`的路径。
如果没有设置，那则由`.proto`文件的`package`语句或`.proto`文件的名称来决定。

#### `go_package` 的作用

 **指定生成的go代码的包名**：
 `go_package` 选项可以用来定义生成的 Go 代码所在的包名。这对于确保生成的代码可以被其他 Go 代码正确引用非常重要。
    
**避免命名冲突**：
在大型项目中，可能会有多个 `.proto` 文件生成相同的 Go 结构体或类型。通过指定不同的 `go_package`，可以避免这些命名冲突。
    
 **提高可读性**：
 使用 `go_package` 可以使生成的代码更具可读性，并且与 Go 的包管理方式一致。


#### 范例
范例一：
```text
syntax = "proto3";

option go_package = "github.com/TripleCGame/apis/api;api";
import "google/protobuf/struct.proto";

message Response {
  int32 code = 1;
  google.protobuf.Struct data = 2;
  string msg = 3; 
}
```

生成的`pb.go`文件：
![](attachments/Pasted%20image%2020241219104710.png)

其他`go`文件`import`该`.pb.go`文件时：
![](attachments/Pasted%20image%2020241219104830.png)


#### 小结
`go_package` 指定的是 `.proto` 文件生成的 `.pb.go` 文件中的包名。
多个  `.proto` 文件 可以有相同的 `go_package`，就意味着多个`.proto` 文件生成的 `.pb.go` 文件属于同一个 `package`。

## 消息(message)
一个 `.proto` 文件中可以写多个消息类型，即对应多个结构体(struct)。

### 成员类型
#### 基本类型
![](attachments/Pasted%20image%2020241218160012.png)


**空值**
- 对于strings,默认是一个空string
- 对于bytes,默认是一个空bytes
- 对于bools,默认false
- 对于数值类型,默认0
- 对于枚举类型,默认是第一个定义的枚举值,必须为0；

#### oneof

当你有多个字段，但在某个时刻只需要其中一个字段时，使用 `oneof` 是非常合适的。
**`oneof`** 关键字用于定义一组互斥的字段。
换句话说，在一个 `oneof` 中，最多只能设置一个字段。使用 `oneof` 可以有效地节省存储空间，因为它只会存储一个字段的值。

#### map类型
##### 背景
我们在Go语言开发中，最常用的就是切片类型和map类型了。
切片类型在ProtoBuf中对应的就是repeated类型，前面我们已经介绍过了。
再重点介绍一下map类型，ProtoBuf也是支持map类型的：

##### 语法
proto3支持map类型声明:

```
map<key_type, value_type> map_field = N;

message Project {...}
map<string, Project> projects = 1;


message 消息名 {
   map<key, value> name = n;
}
```

**（1）`key_type`**
可以是内置的标量类型(除浮点类型和`bytes`)。另外，请注意，枚举不是有效的`key_type`。

**（2）`value_type`**
可以是除map以外的任意类型。

 **（3）`map`字段不支持`repeated`属性**
**（4）不要依赖`map`类型的字段顺序**

##### 范例

![](attachments/Pasted%20image%2020241231105610.png)

```go
syntax = "proto3";

package map;
option go_package = "./;score";

message Student{
  int64              id    = 1; //id
  string             name  = 2; //学生姓名
  map<string, int32> score = 3;  //学科 分数的map
}
```

![](attachments/Pasted%20image%2020241231105905.png)


#### any类型
**`Any`** 是一个特殊的消息类型，允许你在 `protobuf` 消息中嵌入任意类型的消息。它可以用来实现动态类型的消息，允许在一个字段中存储不同类型的数据。
比如，将`Any` 用于需要在运行时处理不同类型的消息的场景，例如 `RPC` 系统中需要传递多种类型的请求或响应。


##### 范例
```text
syntax = "proto3";

package messages;

import "google/protobuf/any.proto";

// 定义一个 Message 消息，使用 Any 来表示任意类型的内容
message Message {
    string id = 1;
    google.protobuf.Any content = 2; // 任意类型的内容
}

```

使用 `protoc` 生成 Go 代码：
```go
protoc --go_out=. messages.proto
```

生成的 Go 代码：
```go
package messages

import (
    proto "github.com/golang/protobuf/proto"
    any "google.golang.org/protobuf/types/known/anypb"
)

// Message 消息
type Message struct {
    Id      string        `protobuf:"bytes,1,opt,name=id,proto3" json:"id,omitempty"`
    Content *any.Any      `protobuf:"bytes,2,opt,name=content,proto3" json:"content,omitempty"`
}
```

在 Go 代码中使用 `Any`:
```go
package main

import (
    "fmt"
    "github.com/example/project/messages" // 导入生成的包
    "google.golang.org/protobuf/types/known/anypb"
)

func main() {
    // 创建一个具体的消息
    circle := &shapes.Circle{Radius: 5.0}

    // 将 Circle 包装到 Any 中
    anyCircle, err := anypb.New(circle)
    if err != nil {
        fmt.Println("Error creating Any:", err)
        return
    }

    // 创建一个 Message
    msg := &messages.Message{
        Id:      "msg1",
        Content: anyCircle,
    }

    // 打印消息内容
    fmt.Printf("Message ID: %s\n", msg.Id)
    fmt.Printf("Content Type: %s\n", msg.Content.TypeUrl)
}

```

### 成员修饰符
#### singular
每个字段的修饰符默认是 singular （Singular：单数的意思），一般省略不写。
即 表示该字段不重复。

#### repeated
消息体中该字段 可以重复任意多次（包括0次）。重复的值的顺序会被保留。
即用来表示 Go 语言中的 slice 类型。

#### required
消息体中该字段是必须要设置的。

【注意】：使用`required`弊多于利；在实际开发中更应该使用`optional`和`repeated`而不是`required`。


#### optional
消息体中该字段可以有0个或1个值（不超过1个）。
消息体中该字段为空时，`optional`的字段可以根据`defalut`设置默认值。

在 proto3 的早期版本中，确实没有 `optional` 关键字。直到 2020 年，Google 在 proto3 的更新中引入了 `optional` 关键字，以便在处理某些场景时提供更好的语义和功能。

现在的  proto3 中，所有字段默认都是 **optional** 的。这意味着您不需要显式地声明字段为 `optional`。
如果消息体没有提供值，字段将被视为缺失。对于基本数据类型（如 `int32`、`string` 等），缺失的字段将会使用其默认值（例如，数字类型默认为 0，布尔类型默认为 false，字符串默认为空字符串）。

使用 `optional` 可以明确指示字段是可选的，并且可以通过 `has_<field_name>()` 方法来检查消息体中该字段是否被设置。

#### proto2 和 proto3的修饰符的变化
在 `proto2` 中，您必须显式声明字段为 `optional` 或 `required`。
在 `proto3` 中，所有字段默认都是 `optional`，并且不再支持 `required` 关键字，这简化了字段的定义。并且可以通过 `has_<field_name>()` 方法来检查字段是否被设置。

### 成员标识号(域号：field_number)
在消息Message 的定义中，每个字段等号(`=`)后面都有唯一的标识号(域号：field_number)，
**用于在反序列化过程中识别各个字段的，一旦开始使用就不能改变 **。

标识号从整数1开始，依次递增，每次增加1，标识号的范围为`1~2^29 - 1`，其中`[19000-19999]`为`Protobuf`协议预留字段，开发者不建议使用该范围的标识号；一旦使用，在编译时`Protoc`编译器会报出警告。


```pb
syntax = "proto3";

option go_package = "./";
package main;
// this is a comment
message Student {
  string name = 1; // 标识符1
  bool male = 2; // 标识符2
  repeated int32 scores = 3; // 标识符3
}
```

### message 中使用其他message类型
```text
message SearchResponse {
  repeated Result results = 1; 
}
message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```
`Result`是另一个消息类型，在 `SearchReponse` 作为一个消息字段类型使用。

### message 嵌套
上面已经介绍过可以嵌套使用另一个`message`作为字段类型，其实还可以在一个`message`内部定义另一个`message`类型，作为子级`message`。

示例：`Result`类型在`SearchResponse`类型中定义并直接使用，作为`results`字段的类型
```text
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

内部声明的`message`类型名称只可在内部直接使用，在外部引用需要前置父级`message`名称. 
```text
message SomeOtherMessage {
    SearchResponse.Result result = 1;
}
```

### message 的更新规则
 message定义以后如果需要进行修改，为了保证之前的序列化和反序列化能够兼容新的message；
 message的修改需要满足以下规则：

- 不可以修改已存在域中的标识号。
    
- 所有新增添的域必须是 optional 或者 repeated。
    
- 非required域可以被删除。但是这些被删除域的标识号不可以再次被使用。
    
- 非required域可以被转化，转化时可能发生扩展或者截断，此时标识号和名称都是不变的。
    
- sint32和sint64是相互兼容的。
    
- fixed32兼容sfixed32。 fixed64兼容sfixed64。
    
- optional兼容repeated。发送端发送repeated域，用户使用optional域读取，将会读取repeated域的最后一个元素。

### message中optional字段的空值(缺失值)

现在的  proto3 中，所有字段默认都是 **optional** 的。这意味着您不需要显式地声明字段为 `optional`。
如果消息体没有为某个字段提供值，字段将被视为缺失。对于基本数据类型（如 `int32`、`string` 等），缺失的字段将会使用其默认值（例如，数字类型默认为 0，布尔类型默认为 false，字符串默认为空字符串）。


#### 区分Protobuf 3中缺失值和默认值
##### `has_xxx（）`方法来区分

使用 `optional` 可以明确指示字段是可选的，并且可以通过 `has_<field_name>()` 方法来检查消息体中该字段是否被设置。

##### 增加标识字段

众所周知，在Go中数字类型的默认值为`0`（这里仅以数字类型举例），这在某些场景下往往会引起一定的歧义。

例如：增加一个`has_show_field`字段标识`is_show`是否为有效值。如果`has_show_field`为`true`则`is_show`为有效值，否则认为`is_show`未设置值。

此方案虽然直白，但每次设置`is_show`的值时还需设置`has_show_field`的值，甚是麻烦故笔者十分不推荐。

##### 字段实际取值和默认值区分

字段的实际趋势和默认值区分，即不使用对应类型的默认值作为该字段的有效值。
接着前面的例子继续描述，`is_show`为`1`时表示展示，`is_show`为`2`时表示不展示，其他情况则认为`is_show`未设置值。


## 枚举(enum)
枚举类型适用于提供一组预定义的值，选择其中一个。例如我们将性别定义为枚举类型。

```pb
message Student {
  string name = 1;
  enum Gender {
    FEMALE = 0;
    MALE = 1;
  }
  Gender gender = 2;
  repeated int32 scores = 3;
}
```

### 特性

- 枚举类型的第一个选项的标识符必须是0，这也是枚举类型的默认值。
- 别名（Alias），允许为不同的枚举值赋予相同的标识符，称之为别名，需要打开`allow_alias`选项。

```text
message EnumAllowAlias {
  enum Status {
    option allow_alias = true;
    UNKOWN = 0;
    STARTED = 1;
    RUNNING = 1;
  }
}
```

## 服务(service)


## import 其他pb文件
`import`，如果`proto`文件需要使用在其他`proto`文件中已经定义的结构，可以使用`import`来导入其他的`proto`文件。

```text
import "myproject/other_protos.proto";
```

导入后则通过**被导入pb文件的包名.结构体名**使用。

### 范例 
目录结构：
```bash
lixd@17x:~/17x/projects/grpc-go-example$ tree
├── protobuf
│   │ 
│   └── import
│       ├── compoent.proto
│       └── computer.proto
└── README.md

---------

# cat compoent.proto
syntax = "proto3";
option go_package = "protobuf/import;proto";
package import;

message CPU {
  string Name = 1;
  int64 Frequency = 2;
}
message Memory {
  string Name = 1;
  int64 Cap = 2;
}

----------
# cat computer.proto
syntax = "proto3";
option go_package = "protobuf/import;proto";
package import;
import "protobuf/import/component.proto";

message Computer {
  string name = 1;
  import.CPU cpu = 2;
  import.Memory memory = 3;
}

```

protoc 编译：在`项目根路径(grpc-go-example)`下进行编译
```bash
lixd@17x:~/17x/projects/grpc-go-example$ protoc --proto_path=. --go_out=. ./protobuf/import/*.proto 

```
参数：

**1）`–proto_path=.`**
指定在当前目录(`grpc-go-example`)寻找 `import` 的文件（默认值也是当前目录）；然后 `protobuf` 文件中的 `import` 路径如下：
```bash
import "protobuf/import/component.proto";
```

所以最终会去找 `grpc-go-example/protobuf/import/component.proto`。
`--proto_path`和`import`是可以互相调整的，只需要能找到就行。

==建议：protoc参数 –proto_path 指定为根目录，proto文件中的import 则从根目录次一级目录开始==。

**2）`–go_out=.`**

指定将生成文件放在当前目录( `grpc-go-example`)，同时因为 `proto` 文件中也指定了目录为`protobuf/import`,具体如下：
```bash
option go_package = "protobuf/import;proto";
```
所以最终生成目录为`--go_out+go_package = grpc-go-example/protobuf/import`。

可以通过参数 `--go_opt=paths=source_relative` 来指定使用绝对路径，从而忽略掉 `proto` 文件中的 `go_package` 路径，直接生成在 `–go_out` 指定的路径。

**3) `./protobuf/import/*.proto`**

编译 指定 目录下的所有`proto` 文件。当然也可以一个一个编译，只要把相关文件都编译好即可。

## 注释
`.proto` 文件可以写注释，单行注释 `//`，多行注释 `/* ... */`

## protobuf 的 推荐书写风格

### 消息和字段(Messages & Fields)

- **消息名**使用**首字母大写驼峰风格**(`CamelCase`)，例如`message StudentRequest { ... }`
- **消息的字段名**使用**小写下划线**的风格，例如 `string status_code = 1`
- 枚举类型，**枚举名**使用**首字母大写驼峰风格**，例如 `enum FooBar`，枚举值使用全大写下划线隔开的风格(`CAPITALS_WITH_UNDERSCORES` )，例如 FOO_DEFAULT=1

### 服务(service)

- **服务名和方法名**，均使用**首字母大写驼峰风格**，例如`service FooService{ rpc GetSomething() }`


# 编译go代码所需的工具

以`go`为例，下面是所需工具。

|工具|介绍|安装|
|---|---|---|
|[protobuf](https://github.com/protocolbuffers/protobuf)|protoc 可执行文件|[Install](http://google.github.io/proto-lens/installing-protoc.html)|
|[protoc-gen-go](https://github.com/golang/protobuf/tree/master/protoc-gen-go)|从 proto 文件，生成 .go 文件|[Install](https://grpc.io/docs/languages/go/quickstart/)|
|[protoc-gen-go-grpc](https://github.com/grpc/grpc-go)|从 proto 文件，生成 GRPC 相关的 .go 文件|[Install](https://grpc.io/docs/languages/go/quickstart/)|
|[protoc-gen-grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)|从 proto 文件，生成 grpc-gateway 相关的 .go 文件|[Install](https://github.com/grpc-ecosystem/grpc-gateway#installation)|
|[protoc-gen-openapiv2](https://github.com/grpc-ecosystem/grpc-gateway)|从 proto 文件，生成 swagger 所需的json文件|[Install](https://github.com/grpc-ecosystem/grpc-gateway#installation)|

# `protobuf`的编译器：`protoc`

protoc(Protocol Compiler： protobuf 的 编译器)。

![](attachments/Pasted%20image%2020241217201208.png)


## `protoc`的安装
```bash
# 下载安装包 
$ wget https://github.com/protocolbuffers/protobuf/releases/download/v3.11.2/protoc-3.11.2-linux-x86_64.zip 

# 解压到 /usr/local 目录下 
$ unzip protoc-3.11.2-linux-x86_64.zip -d /usr/local

$ protoc --version 
libprotoc 3.11.2
```



### `golang`下`protoc`的插件工具`protoc-gen-go`的安装
在 `Golang` 中使用 `protobuf`，还需要安装 `protoc`的插件工具`protoc-gen-go`，这个工具用来将 `.proto` 文件转换为 `Golang` 代码。
如果是其他的语言，比如`Java`, 安装 `protoc`即可，就可以将`pb`文件转换为 `Java` 代码，不需要插件工具。

```bash
// 最新版本
go install github.com/golang/protobuf/protoc-gen-go@latest

// 指定版本
go install github.com/golang/protobuf/protoc-gen-go@v1.5.2
```

`protoc-gen-go` 将自动安装到 `$GOPATH/bin` 目录下，也需要将这个目录加入到`PATH`环境变量中。


## 将`.proto`文件编译成指定语言类库

`protoc`编译器支持将`proto文件`编译成多种语言版本的代码。
```bash
protoc [OPTION] PROTO_FILES
```

OPTION是命令的选项, PROTO_FILES是我们要编译的proto消息定义文件，支持多个。

常用的OPTION选项：
```c
  --go_out=OUT_DIR            指定代码生成目录，生成 GO 代码
  --cpp_out=OUT_DIR           指定代码生成目录，生成 C++ 代码
  --csharp_out=OUT_DIR        指定代码生成目录，生成 C# 代码
  --java_out=OUT_DIR          指定代码生成目录，生成 java 代码
  --js_out=OUT_DIR            指定代码生成目录，生成 javascript 代码
  --objc_out=OUT_DIR          指定代码生成目录，生成 Objective C 代码
  --php_out=OUT_DIR           指定代码生成目录，生成 php 代码
  --python_out=OUT_DIR        指定代码生成目录，生成 python 代码
  --ruby_out=OUT_DIR          指定代码生成目录，生成 ruby 代码
```

## 使用

### `--proto_path`
```bash
protoc --proto_path=$GOPATH/src --proto_path=. --go_out=. ./*.proto
```

`--proto_path`：（protobuf path）用于表示要编译的`proto`文件所依赖的其他`proto`文件的查找位置，可以使用`-I`来替代。如果没有指定则从当前目录中查找。(类似于编译`.c`文件时，`.c`文件中`include`了`.h`文件时设置`.h`文件的查找路径)

==建议：`protoc`参数 `--proto_path` 指定为根目录(即`.`，在根目录下执行protoc )，`proto`文件中的`import` 则从根目录下的次一级目录开始。==。

那么，`proto`文件中 `import` 其他的 `proto`文件时；
==查找其他`proto`文件的路径就是：`--proto_path`+ `pb 文件中 import 的路径`== （ 和`C`语言中查找头文件是一样的，`-I`的路径 + `include 指定的文件路径`）。

### `--go_out`
```bash
protoc --proto_path=$GOPATH/src --proto_path=. --go_out=. ./*.proto
```
`--go_out`有两层含义：
一层是输出的是`go`语言对应的文件(如果是`java`语言，就是`--java_out`)；
一层是指定生成的`go`文件的存放位置。

#### 生成的 `pb.go`文件的位置
**（1）不存在`--go_opt=paths=source_relative`参数时**：
最终生成的`go`文件的目录为`--go_out` + `pb文件中的 go_package 指定的路径`

如果不加`--go_opt=paths=source_relative`，效果等同于加了`--go_opt=paths=import`，会在生成的`.pb.go`文件前加上`proto`文件内`option go_package`的内容，最终的路径就变成了:
```go
${--go_out}/${option go_package}/${proto文件名}.pb.go
```

**（2）存在`--go_opt=paths=source_relative`参数时**：
通过参数 `--go_opt=paths=source_relative` 来指定使用绝对路径，从而忽略掉 `proto` 文件中的 `go_package` 路径；
也就是直接在`--go_out=paths`指定目录下生成`.pb.go`文件，最终的文件会生成在：
```bash
${--go_out}/${proto文件名}.pb.go
```

### `--go_opt`

### `--experimental_allow_proto3_optional`
在 proto3 的早期版本中，确实没有 `optional` 关键字。直到 2020 年，Google 在 proto3 的更新中引入了 `optional` 关键字，以便在处理某些场景时提供更好的语义和功能。

现在的  proto3 中，所有字段默认都是 **optional** 的。这意味着您不需要显式地声明字段为 `optional`。
这需要使用新版本的`protoc`在将`pb`文件编译为`go`文件时必须使用`--experimental_allow_proto3_optional`开启此特性。


## 范例
###  范例一
文件名：`score_server/score_info.proto`
```pb
syntax = "proto3";

package score_server;
// package 的name和 pb文件所在的目录名称相同

// 基本的积分消息
message base_score_info_t{
  int32       win_count = 1;                  // 玩家胜局局数
    int32       lose_count = 2;                 // 玩家负局局数
    int32       exception_count = 3;            // 玩家异常局局数
    int32       kill_count = 4;                 // 总人头数
    int32       death_count = 5;                // 总死亡数
    int32       assist_count = 6;               // 总总助攻数
    int64       rating = 7;                     // 评价积分
}

/*
proto文件的 message 名称以及成员的名称，在编译为 go代码之后，在go代码中会发生变化：
 下划线命名法变为了大驼峰命名法（首字母大写）
 首字母大写：是为了go中的可见性，其他的包可以访问自动生成的go中的结构体以及成员。
*/
```

编译proto文件，生成go代码
```bash
cd score_server 
protoc --go_out=. score_info.proto
```

测试代码：
```go
package main

import (
  "fmt"
   // 导入protobuf依赖包
  "github.com/golang/protobuf/proto"

  // demo 应该是go.mod的 module path，score_server是根目录(go.mod所在目录)下的 score_server 文件夹（score_server文件夹下的package name就是score_server）
  "demo/score_server" 
)

func main() {
/*
proto文件的 message 名称以及成员的名称，在编译为 go代码之后，在go代码中会发生变化：
下划线命名法变为了大驼峰命名法（首字母大写）；
 首字母大写：是为了go中的可见性，其他的包可以访问自动生成的go中的结构体以及成员。
*/
  score_info := &score_server.BaseScoreInfoT{}
  score_info.WinCount = 10
  score_info.LoseCount = 1
  score_info.ExceptionCount = 2
  score_info.KillCount = 2
  score_info.DeathCount = 1
  score_info.AssistCount = 3
  score_info.Rating = 120

  // 以字符串的形式打印消息
  fmt.Println(score_info.String())

  // encode, 转换成二进制数据
  data, err := proto.Marshal(score_info)
  if err != nil {
    panic(err)
  }

  // decode, 将二进制数据转换成struct对象
  new_score_info := score_server.BaseScoreInfoT{}
  err = proto.Unmarshal(data, &new_score_info)
  if err != nil {
    panic(err)
  }

  // 以字符串的形式打印消息
  fmt.Println(new_score_info.String())
}
```

## protoc 代码生成插件机制
由于 Protobuf 是跨语言的, 所以在使用的时候需要为目标语言生成代码, 这些生成代码的工具就是 protoc(Protocol Compiler) 的插件. 
用一句话概括: 插件就是一个命令行工具, 负责根据标准输入读取的 `CodeGeneratorRequest` 消息生成目标代码, 并且序列化成 `CodeGeneratorResponse` 格式写入标准输出(两种消息类型都是 protobuf 消息类型).

插件可执行文件需要命名成 `protoc-gen-$NAME` 的形式, 并且需要放在 PATH 里直接可以调用, 会在命令中有 `--${NAME}_out` 参数时被调用.

（1）`--${NAME}_out` 参数：
控制传递给**插件的参数**和**生成文件输出目录**, 并且是以 `${OPTION},${OPTION}:${OUT_PATH}` 的形式。

（2）`--${NAME}_opt` ：专门来负责参数传递。

 `--${NAME}_out=${OPTION},${OPTION}:${OUT_PATH}` 等于 `--${NAME}_out=${OUT_PATH} --${NAME}_opt=${OPTION},${OPTION}` 这种新形式.


范例如下：
```bash
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    ./pb/origin-hello.proto

```
上面的命令使用了 `protoc-gen-go` 和 `protoc-gen-go-grpc` 两个插件, 传递给 `protoc-gen-go` 的参数为 `paths=source_relative`, 传递给 `protoc-gen-go-grpc` 参数也是 `paths=source_relative`.

## protoc的插件: protoc-gen-go
### 介绍
pb文件导出指定的语言的文件时需要是使用工具 `protoc`，而平时你可能听说过或者在`.pb.go`文件中看到过 `protoc-gen-go` 这个工具的名字。如下所示 ，某个`.pb.go`文件的 部分内容：

![](attachments/Pasted%20image%2020241219101016.png)

==其实 `protoc-gen-go` 是`protoc`用于生成 Go 代码的插件==，
它是 Protocol Buffers 编译器 `protoc` 的一部分。
当您使用 `protoc` 编译 `.proto` 文件时，可以通过指定 `--go_out` 标志来指定要使用的 `protoc-gen-go` 插件，从而生成对应的 `Go` 代码文件（即：指定了 `--go_out`就说明了 `protoc` 要使用  `protoc-gen-go`插件）。

这个插件会根据 `.proto` 文件中定义的消息和服务等内容，生成与之对应的 Go 结构体、接口和方法等代码。因此，`protoc-gen-go` 和 `protoc` 是紧密相关的，它们共同用于将 Protocol Buffers 文件编译为 Go 语言中的数据结构和服务定义。

### 安装
```go
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
```
安装好了之后, 在`$GOPATH/bin`下面会找到`protoc-gen-go`。
将`$GOPATH/bin`路径添加到`PATH`环境变量中。


### protobuf 到 go 的转换
这里使用一个测试文件对照说明常用结构的protobuf到golang的转换。只说明关键部分代码。

#### package(go_package) 的转换
在`proto`文件中使用`package`关键字声明proto文件的包名，默认转换成`go`中的包名与此一致；如果需要指定不一样的包名，可以使用`go_package`选项来指定。

```pb
package test;
option go_package="test";
```

#### message 的转换

proto中的`message`对应go中的`struct`，全部使用驼峰命名规则。
嵌套定义的`message`，`enum`转换为go之后，名称变为`Parent_Child`结构。

proto如下所示：
```
// Test 测试
message Test {
    int32 age = 1;
    int64 count = 2;
    double money = 3;
    float score = 4;
    string name = 5;
    bool fat = 6;
    bytes char = 7;
    // Status 枚举状态
    enum Status {
        OK = 0;
        FAIL = 1;
    }
    Status status = 8;
    // Child 子结构
    message Child {
        string sex = 1;
    }
    Child child = 9;
    map<string, string> dict = 10;
}
```

转换后的 go 代码：
```
// Status 枚举状态
type Test_Status int32

const (
    Test_OK   Test_Status = 0
    Test_FAIL Test_Status = 1
)

// Test 测试
type Test struct {
    Age    int32       `protobuf:"varint,1,opt,name=age" json:"age,omitempty"`
    Count  int64       `protobuf:"varint,2,opt,name=count" json:"count,omitempty"`
    Money  float64     `protobuf:"fixed64,3,opt,name=money" json:"money,omitempty"`
    Score  float32     `protobuf:"fixed32,4,opt,name=score" json:"score,omitempty"`
    Name   string      `protobuf:"bytes,5,opt,name=name" json:"name,omitempty"`
    Fat    bool        `protobuf:"varint,6,opt,name=fat" json:"fat,omitempty"`
    Char   []byte      `protobuf:"bytes,7,opt,name=char,proto3" json:"char,omitempty"`
    Status Test_Status `protobuf:"varint,8,opt,name=status,enum=test.Test_Status" json:"status,omitempty"`
    Child  *Test_Child `protobuf:"bytes,9,opt,name=child" json:"child,omitempty"`
    Dict   map[string]string `protobuf:"bytes,10,rep,name=dict" json:"dict,omitempty" protobuf_key:"bytes,1,opt,name=key" protobuf_val:"bytes,2,opt,name=value"`
}

// Child 子结构
type Test_Child struct {
    Sex string `protobuf:"bytes,1,opt,name=sex" json:"sex,omitempty"`
}
```


除了会生成对应的结构外，还会有些工具方法，如字段的`getter`:

```go
func (m *Test) GetAge() int32 {
    if m != nil {
        return m.Age
    }
    return 0
}
```

枚举类型会生成对应名称的常量，同时会有两个`map`方便使用：
```go
var Test_Status_name = map[int32]string{
    0: "OK",
    1: "FAIL",
}
var Test_Status_value = map[string]int32{
    "OK":   0,
    "FAIL": 1,
}
```

#### service 的转换

定义一个简单的Service，`TestService`有一个方法`Test`，接收一个`Request`参数，返回`Response`：

```pb
// TestService 测试服务
service TestService {
    // Test 测试方法
    rpc Test(Request) returns (Response) {};
}

// Request 请求结构
message Request {
    string name = 1;
}

// Response 响应结构
message Response {
    string message = 1;
}
```

转换结果：

```go
// 客户端接口
type TestServiceClient interface {
    // Test 测试方法
    Test(ctx context.Context, in *Request, opts ...grpc.CallOption) (*Response, error)
}

// 服务端接口
type TestServiceServer interface {
    // Test 测试方法
    Test(context.Context, *Request) (*Response, error)
}
```
生成的`go`代码中包含该`Service`定义的接口，客户端接口已经自动实现了，直接供客户端使用者调用，服务端接口需要由服务提供方实现。


##  protoc的插件: protoc-gen-optional-dft

### 背景

如果消息体没有为某个`optional`字段提供值，字段将被视为缺失，使用缺失值（空值）。但是又不想使用该字段类型的空值，而是额外给该字段设置一个默认值。

那么，如何获取到默认值呢？

### 思路
**（1）方式一**：
在项目的程序代码中进行设置。
通过 `has_<field_name>()` 方法来检查消息体中该字段是否被设置。
如果没有设置，直接给给其赋值一个默认值。这个默认值的设置是放在了 项目代码逻辑中。

**（2）方式二**：
在 `pb` 文件的`Message`的字段后面添加`// dft = xxx` 的注释的方式。
这种方式，默认值的设置是放在了 `pb` 文件中，但是需要将 `pb` 文件的 `Message`的字段的 默认值注释 给解析为 对应的 代码。
`protoc-gen-optional-dft` 插件就是这样的作用，其将`pb`文件中含有 默认值注释  的 `message`的字段 生成一个设置默认值的函数。
> 注：该函数的作用是：如果消息体中含有该字段，则取消息体中的字段值；如果不存在该字段，则取 `pb` 中设置的默认值。

比如：`aa.proto` 文件中含有设置了 默认值注释 的 `message`字段， 经过 `protoc-gen-optional-dft` 插件处理后，会生成一个 `aa.pb.dft.go`文件。

### 方式二的实现

实现过程中，会用到参考下面的 pb插件：
```go
参考一：
https://github.com/j2gg0s/protoc-gen-dft
https://github.com/linka-cloud/protoc-gen-defaults
https://pkg.go.dev/github.com/protoc-extensions/protoc-gen-go-defaults#section-readme

```

# grpc

## grpc 介绍
gRPC 是一个基于 HTTP/2 的"高性能、开源和通用的 RPC 框架". 

### grpc 和 protobuf 的关系

gRPC 默认使用 Protobuf 作为接口定义语言和数据传输格式.
gRPC、 Protobuf 和 Go 都是由 Google 开发的, 这三者结合使用具有套装效果.

## pb文件中定义 gRPC 服务和方法

定义一个服务（service）, 并指定其可以被远程调用的方法, 以及方法的参数和返回值类型. 这即是 gRPc 定义服务（service）的思想.

```go
syntax = "proto3"; // 指定版本

option go_package = "example/proto"; // 指定 Go 代码生成的包名

package artwork; // 包名, 用于区分不同的 proto 文件. 注意此包名并不是 Go 中的包名

// 定义消息类型
message ArtworkInfo {
    uint64 artworkID = 1; 
    string title = 2;
    SourceName source = 3; // 引用枚举类型
    repeated string tags = 4; // repeated 表示为数组
    bool r18 = 5;
    repeated PictureInfo pictures = 6; // 引用嵌套消息类型

    // 定义枚举类型
    enum SourceName {
        Pixiv = 0;
        Twitter = 1;
        // ...
    }

    // 定义嵌套消息类型
    message PictureInfo {
        uint64 pictureID = 1;
        uint64 width = 2;
        uint64 height = 3;
    }
}

message SetArtworkInfoReq {
  string ID = 1;
  ArtworkInfo info = 2;
}

message SetArtworkInfoReply {
  string ID = 1;
  int32 code = 2;
  string message = 3;
}

message GetArtworkInfoRequest {
    string ID = 1; 
    uint64 artworkID = 2;
} 

message GetArtworkInfoReply { 
    string ID = 1;
    int32 code = 2;
    string message = 3;
    repeated ArtworkInfo info = 4;
} 


// 定义服务
service ArtworkService { 
    // 定义方法1
    rpc GetArtworkInfo (GetArtworkInfoRequest) returns (GetArtworkInfoReply);
    // 定义方法2
    rpc SetArtworkInfo (SetArtworkInfoReq) returns (SetArtworkInfoReply);
} 
```

## gRPC 的服务方法类型
gRPC 有四种方法类型:

1. 简单方法(普通的请求-响应方法)
2. 服务端流式方法
3. 客户端流式方法
4. 双向流式方法

> 注意：存在关键字 stream（无论是返回值还是函数参数），说明其为一个流方法。

**服务端流式 RPC**：
即客户端发送一个请求给服务端，可获取一个数据流用来读取一系列消息。客户端从返回的数据流里一直读取直到没有更多消息为止。
```go
rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse){ }
```


**客户端流式 RPC**：
即客户端用提供的一个数据流写入并发送一系列消息给服务端。一旦客户端完成消息写入，就等待服务端读取这些消息并返回应答。
```go
rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse) { }
```

**双向流式 RPC**：
即两边都可以分别通过一个读写数据流来发送一系列消息。
这两个数据流操作是相互独立的，所以客户端和服务端能按其希望的任意顺序读写。
例如：服务端可以在写应答前等待所有的客户端消息，或者它可以先读一个消息再写一个消息，或者是读写相结合的其他方式。每个数据流里消息的顺序会被保持。
```go
rpc BidiHello(stream HelloRequest) returns (stream HelloResponse){ }
```

## protoc-gen-go-grpc
### 安装
```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
export PATH="$PATH:$(go env GOPATH)/bin" # 把 GOBIN 加入 PATH
```
### 介绍
`protoc-gen-go-grpc` 是 `protoc` 生成 go 代码 中的 `GRPC API function`(基于 `pb` 文件中的 `service` 生成 )的插件；
类似于 `protoc-gen-go` 是 `protoc` 生成 go 代码 中的 结构体的插件（基于 `pb` 文件中的 `messsage` 生成 ）。

当您使用 `protoc` 编译 `.proto` 文件时，可以通过指定 `--go-grpc_out` 标志来指定要使用的 `protoc-gen-go-grpc` 插件，从而生成对应的 `Go` 代码文件（即：指定了 `--go-grpc_out`就说明了 `protoc` 要使用  `protoc-gen-go-grpc`插件）。

### 使用
生成`_grpc.pb.go`文件
```bash

```
#### `--go-grpc_out`
#### `--go-grpc_opt`

### `pb`文件生成的`.pb.go`和`_grpc.pb.go`的区别
一个 `pb`文件中定义了消息(mesage)，也定义了服务（service）。
那么，消息(`mesage`)相关的信息借助于 `protoc-gen-go`插件，生成的文件为`.pb.go`文件，
服务（`service`）相关的信息，借助于`protoc-gen-go-grpc`插件，生成的文件为`_grpc.pb.go`文件。

#### 注意

`pb`文件的服务（`service`）在 生成的`_grpc.pb.go`文件中对应的类型是 2个`interface` 接口类型，分别是 `Client`和 `Server` 的  接口类型 。

client 和 server 的 `pb`文件相同，都需要基于 `pb` 文件 生成对应的  `.pb.go`和`_grpc.pb.go`文件。

**（1）对于`client`而言**：
不需要在项目中实现 client 的`interface` 接口中的函数。

**（2）对于`server`而言**：
在项目中，导入`_grpc.pb.go`文件对应的`package`之后，需要实现 服务端的`interface` 接口中的函数。

## grpc的范例
### pb文件
```pb
syntax = "proto3"; // 指定版本

option go_package = "example/proto"; // 指定 Go 代码生成的包名

package artwork; // 包名, 用于区分不同的 proto 文件. 注意此包名并不是 Go 中的包名

// 定义消息类型
message ArtworkInfo {
    uint64 artworkID = 1; 
    string title = 2;
    SourceName source = 3; // 引用枚举类型
    repeated string tags = 4; // repeated 表示为数组
    bool r18 = 5;
    repeated PictureInfo pictures = 6; // 引用嵌套消息类型

    // 定义枚举类型
    enum SourceName {
        Pixiv = 0;
        Twitter = 1;
        // ...
    }

    // 定义嵌套消息类型
    message PictureInfo {
        uint64 pictureID = 1;
        uint64 width = 2;
        uint64 height = 3;
    }
}

message GetArtworkInfoRequest {
    uint64 artworkID = 1;
}

// 定义服务
service ArtworkService {
    // 定义方法
    rpc GetArtworkInfo (GetArtworkInfoRequest) returns (ArtworkInfo);
    rpc GetArtworkInfoList (GetArtworkInfoRequest) returns (stream ArtworkInfo); // 定义流式方法 //[!code ++]
}
```
### 生成`_grpc.pb.go`文件
```bash
protoc --go_out=. --go-grpc_out=. example.proto
```
`protoc` 会在当前目录下生成 `proto/example.pb.go` 和 `proto/example_grpc.pb.go` 两个文件, 前者包含了 `example.proto` 中定义的消息类型, 后者包含了 `example.proto` 中定义的服务和方法.

### 服务端实现
实现上文定义的 `ArtworkService` 服务, 需要做两部分工作:

1. 实现 `ArtworkService` 服务中定义的所有方法. 即实现 `example_grpc.pb.go` 中的 `ArtworkServiceServer` 接口.
2. 运行一个 gRPC 服务器, 监听来自客户端的请求并返回响应.

#### 实现服务方法

打开 `example_grpc.pb.go` 文件, 可以看到 `ArtworkServiceServer` 接口的定义;

```go
// ArtworkServiceServer is the server API for ArtworkService service.
// All implementations must embed UnimplementedArtworkServiceServer
// for forward compatibility
type ArtworkServiceServer interface {
 GetArtworkInfo(context.Context, *GetArtworkInfoRequest) (*ArtworkInfo, error)
 GetArtworkInfoList(*GetArtworkInfoRequest, ArtworkService_GetArtworkInfoListServer) error
 mustEmbedUnimplementedArtworkServiceServer()
}
```

在项目中导入 `example/proto` 包, 并实现 `ArtworkServiceServer` 接口:
```go
package main

import (
    "context"
    "log"
    "net"

    "example/proto"
    "google.golang.org/grpc"
)

type ArtworkServiceServer struct {
    proto.UnimplementedArtworkServiceServer // 为了实现向前兼容, 需要嵌入此结构体
}

func (s *ArtworkServiceServer) GetArtworkInfo(ctx context.Context, req *proto.GetArtworkInfoRequest) (*proto.ArtworkInfo, error) {
    // 实现
    return &proto.ArtworkInfo{}, nil
}

func (s *ArtworkServiceServer) GetArtworkInfoList(req *proto.GetArtworkInfoRequest, stream proto.ArtworkService_GetArtworkInfoListServer) error {
    // 实现
    for i := 0; i < 10; i++ {
        err := stream.Send(&proto.ArtworkInfo{})
        if err != nil {
            return err
        }
    }
    return nil
}

var server = &ArtworkServiceServer{}

func main() {
}
```

可以看到, 流式方法和一元方法的区别在于, 流式方法的参数或返回值是一个流, 需要向流中写入或读取消息.
```go
func (s *ArtworkServiceServer) GetArtworkInfoList(req *proto.GetArtworkInfoRequest, stream proto.ArtworkService_GetArtworkInfoListServer) error {
    // 实现
    for i := 0; i < 10; i++ {
        err := stream.Send(&proto.ArtworkInfo{})
        if err != nil {
            return err
        }
    }
    return nil
}
```

#### 运行 gRPC 服务器
注：下面并使用任何安全措施。
```go
func main() {
    lis, err := net.Listen("tcp", ":8080")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    s := grpc.NewServer()
    proto.RegisterArtworkServiceServer(s, server)
    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

### 客户端调用
在客户端调用 gRPC 服务, 只需要初始化连接, 并创建客户端, 然后直接调用定义的方法.
```go
package main

import (
    "context"
    "log"

    "example/proto"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    conn, err := grpc.Dial("localhost:8080", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()
    client := proto.NewArtworkServiceClient(conn)
    resp, err := client.GetArtworkInfo(context.Background(), &proto.GetArtworkInfoRequest{ArtworkID: 1})
    if err != nil {
        log.Fatalf("could not get artwork info: %v", err)
    }
    log.Println(resp)
}
```

而对于流式方法, 客户端需要从流中读取响应消息, 直到没有消息为止.
```go
package main

import (
    "context"
    "log"
    "io"

    "example/proto"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    conn, err := grpc.Dial("localhost:8080", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()
    client := proto.NewArtworkServiceClient(conn)
    stream, err := client.GetArtworkInfoList(context.Background(), &proto.GetArtworkInfoRequest{ArtworkID: 1})
    if err != nil {
        log.Fatalf("could not get artwork info: %v", err)
    }
    for {
        resp, err := stream.Recv()
        if err == io.EOF { // 判断是否已结束
            break
        }
        if err != nil {
            log.Fatalf("could not get artwork info: %v", err)
        }
        log.Println(resp)
    }
}
```

# grpc 支持TLS

参考：[Go 的 gRPC 和 Protocol Buffers---TLS 认证](https://krau.top/posts/grpc-and-protobuf-in-go-tls)

## GRPC的连接类型
**gRPC 中的连接类型一共有以下3种**：

- 1）`insecure connection`：不使用TLS加密
- 2）`server-side TLS`：仅服务端TLS加密
- 3）`mutual TLS`：客户端、服务端都使用TLS加密

之前案例中使用的都是 `insecure connection`
```go
conn, err := grpc.Dial(addr,grpc.WithInsecure())
```

### server-side TLS
服务端 TLS 具体包含以下几个步骤：
- 1）制作证书，包含服务端证书（server.crt）和 CA 证书(ca.crt)；
- 2）服务端启动时加载证书；
- 3）客户端连接时使用CA 证书校验服务端证书有效性。

注： 也可以不使用 CA证书，即服务端证书自签名。




## GRPC使用TLS的背景
为了保证 gRPC 服务不被第三方监听和调用, 防止通信被篡改或伪造, 需要对 gRPC 服务添加身份验证机

**SSL/TLS**: gRPC 集成了 SSL/TLS, 提倡使用 SSL/TLS 对服务器/客户端进行身份验证, 并对客户端与服务器之间交换的所有数据进行加密。



## 基本概念

### CA
CA (证书授权机构：Certificate Authority), 即证书颁发机构, 用于管理和签发证书. 

#### CA的作用
(1) 检查证书持有者的合法性,
(2) 颁发证书, 防止证书被伪造.

#### CA根证书
**CA 根证书（Certificate Authority Root Certificate）**
CA 自己也需要有证书, 称为根证书(`ca.crt`)，即证书颁发机构的根证书。
主要用于建立和验证信任链，可以用它用于验证其他证书的有效性。

> 注：CA 根证书(ca.crt)是信任链的顶端，所有其他证书（包括中间证书和最终用户证书）都是通过它进行验证的。

**一般情况下, 根证书是自签名的, 即自己给自己签发证书**. 
根证书是信任链的起点, 有了根证书之后 CA 才能给其他人签发证书.

#### 小结
**ca.crt**：这是 CA 的根证书文件。它包含了 CA 的公钥和相关信息，用于验证由该 CA 签发的所有证书的有效性。
客户端和服务器通常会使用这个文件（ca.crt）来验证证书(比如：server.crt,  client.crt )的合法性。


### PEM格式
#### 介绍
**PEM全称**: 
Privacy-Enhanced Mail（隐私增强邮件）。尽管 PEM 最初的目的是确保电子邮件传输的安全，但现在广泛用于**证书文件(xx.csr、xxx.crt、xxx.key)**，以实现跨不同平台的**加密通信**。

**PEM格式**: 
PEM是一种文件格式，用于存储和传输加密数据，包括证书和私钥。
PEM格式的文件通常以 **`ASCII`文本形式存储**，使用 **`Base64`编码** 来表示二进制数据，并以特定的标头和尾部标识。

#### 特点
PEM 文件有几个特点，使其成为安全通信的理想选择。下面是一些决定性的特征：

- **纯文本格式**：
PEM 文件是纯文本文件(ASCII)，便于使用标准文本编辑器打开。

- **Base64 编码**：
PEM 文件的编码内容以**base64 ASCII 格式**存储，可作为可读文本处理。

- **页眉和页脚标记**：
PEM 文件有明确的页眉标记，如 **—–BEGIN CERTIFICATE—–**；
页脚，如 **—–END CERTIFICATE—–**

- **扩展名多种多样**：
虽然 **.pem** 是最常见的扩展名，但 PEM 文件也可以有 `.crt`、`.cer` 和`.key` 扩展名，具体取决于具体内容（如证书、私钥）。

这些功能确保 PEM 文件**易于访问**和**编辑**，并与各种安全工具**兼容**，例如 **OpenSSL**.



#### PEM格式的结构

PEM 文件**结构**包括页眉和页脚，通常如下所示：
```text
-----BEGIN CERTIFICATE-----
（Base64编码的数据）
-----END CERTIFICATE-----
```

不同类型的PEM文件有不同的标头，例如：

- 证书(certificate)：`-----BEGIN CERTIFICATE-----`
- 私钥（private key）：`-----BEGIN PRIVATE KEY-----` 或 `-----BEGIN RSA PRIVATE KEY-----`
- 公钥（public key）：`-----BEGIN PUBLIC KEY-----`
- CSR（证书签名请求）：`-----BEGIN CERTIFICATE REQUEST-----`



#### PEM 文件的工作原理
PEM 文件的工作原理是将加密数据编码为可读的 ASCII 格式，从而轻松安全地存储和传输敏感信息。

下面是它们的工作原理：
**编码和结构**：
PEM 文件中的`base64` 编码将二进制数据转换为 `ASCII` 文本。
这种编码包括独特的页眉和页脚，如 **—–BEGIN CERTIFICATE—–** 和 **—–END CERTIFICATE—–**，它们是文件内容类型的信号。

**纯文本可访问性**：
由于 PEM 文件基于文本，因此可以用任何标准文本编辑器打开。不过，文件本身的内容仍被编码，因此只有支持加密标准（如 OpenSSL）的工具或软件才能正确解释和使用文件数据。

**公钥基础设施 (PKI)：**
PEM 文件是公钥基础设施的组成部分，是安全数据交换的基础。
PKI 依赖于一套加密规则和协议，包括公钥和私钥的分配。PEM 文件持有这些密钥，可在系统间进行安全、经过验证的通信。

PEM 文件的**编码**、**结构**和**安全协议**使其既易于访问又安全可靠，可作为**加密证书**和**密钥存储的**可靠格式。

#### PEM 与其他证书格式的区别
##### PEM 与 DER
- **编码**：
PEM 文件是**base64 编码的文本文件**，因此可以用任何文本编辑器阅读。而**DER**（区分编码规则）是一种**二进制格式**，不能用纯文本阅读。

- **使用方法**：
PEM 文件在**网络服务器**和**公钥基础设施**（PKI）设置中较为常见，而 DER 文件则主要用于**Java**平台。

## 生成证书
要想在 gRPC 中使用 TLS, 需要准备三组证书:

- CA 根证书
- 服务端证书
- 客户端证书

###  生成CA根证书
#### ca.conf 文件
在任意目录下创建 `ca.conf` 文件:
```text
[ req ]
default_bits       = 4096
distinguished_name = moe

[ moe ]
countryName                 = GB
countryName_default         = BeiJing
stateOrProvinceName         = State or Province Name (full name)
stateOrProvinceName_default = BeiJing
localityName                = Locality Name (eg, city)
localityName_default        = NanJing
organizationName            = Organization Name (eg, company)
organizationName_default    = Kompany
commonName                  = krau.top
commonName_max              = 64
commonName_default          = krau.top
```

说明如下：
##### 证书请求（req） 
`[ req ]` ：Request (证书请求)，这一部分定义了生成证书请求时的参数和选项。用于配置证书请求的整体设置。

- **default_bits**: 这是生成密钥时使用的默认位数。在这个例子中，设置为 4096 位，表示生成的密钥将是 4096 位长，这通常用于增强安全性。
    
- **distinguished_name**: 这一项指定了在证书请求中使用的主题名称部分。

##### 主题名称 (Distinguished Name)
Distinguished Name (主题名称) 部分的自定义配置。包含了生成证书请求时所需的详细信息。通过这种方式，用户可以自定义证书请求的主题信息，以满足特定的需求。

`moe` 是一个自定义名称，可以是任何符合命名规则的标识符。它包含了生成证书请求时需要的各个主题字段（如国家、州/省、市、组织名称和通用名称等）的定义和默认值。

- **countryName**: 表示国家名称，通常是两个字母的 ISO 代码。在这个例子中，设置为 `GB`，表示国家是英国。
    
- **countryName_default**: 这是 `countryName` 字段的默认值。在这个例子中，默认值为 `BeiJing`，但这并不是 ISO 代码，通常应该是国家代码。可能是一个错误或特定配置的需求。
    
- **stateOrProvinceName**: 表示州或省的名称。这里的值是 `State or Province Name (full name)`，提示用户在生成请求时需要提供完整的州或省名称。
    
- **stateOrProvinceName_default**: 这是 `stateOrProvinceName` 字段的默认值，设置为 `BeiJing`，表示省份或州的默认值为北京市。
    
- **localityName**: 是一个在 X.509 证书中使用的属性之一，它用于描述证书持有者的地理位置。具体来说，`localityName` 通常表示一个城市或城镇的名称。这个属性是证书主题（Subject）的一部分，帮助提供有关证书持有者的地理信息。这里的值是 `Locality Name (eg, city)`，提示用户需要提供城市的名称。
    
- **localityName_default**: 这是 `localityName` 字段的默认值，设置为 `NanJing`，表示默认城市为南京。
    
- **organizationName**: 表示组织或公司的名称。这里的值是 `Organization Name (eg, company)`，提示用户需要提供组织的名称。
    
- **organizationName_default**: 这是 `organizationName` 字段的默认值，设置为 `Kompany`，表示默认组织名称为 "Kompany"。
    
- **commonName**：`commonName`（通常缩写为 `CN`）是 `X.509` 证书中一个重要的字段，通常用于标识证书所对应的**主体名称**。对于 `SSL/TLS` 证书，它通常是一个域名（例如 `www.example.com`）或 IP 地址。在 `SSL/TLS` 证书中，`commonName` 用来验证客户端与服务器之间的安全连接。浏览器会检查访问的 `URL` 是否与证书中的 `commonName` 相匹配，以确保连接的安全性。在同一证书中`CN`应唯一，且应与 SAN 中的名称一致。
在这个例子中，设置为 `krau.top`，表示证书将用于这个域名。
    
- **commonName_max**: `commonName` 的值通常是一个有效的域名或 IP 地址。对于域名，必须符合 DNS 规范。一般建议 `commonName` 的长度不超过 64 字符。
    
- **commonName_default**: 这是 `commonName` 字段的默认值，设置为 `krau.top`，表示如果没有提供其他值，默认使用这个域名。

#### ca.key
生成 CA 私钥:：
```bash
openssl genrsa -out ca.key 4096
```

CA收到客户端和服务器的 CSR， 使用其私钥（ca.key）生成相应的证书（client.crt 和 server.crt），并将 CA 证书（ca.crt）分发给需要验证这些证书的客户端和服务器。

#### ca.crt
**自签名** 生成 CA 证书（即CA根证书，ca.crt）:
```bash
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt -config ca.conf
```
根据提示输入各种信息, 最后会生成 `ca.crt` 文件.

### 生成服务端证书
#### server.conf
```text
[ req ]
default_bits       = 2048
distinguished_name = moe

[ moe ]
countryName                 = Country Name (2 letter code)
countryName_default         = CN
stateOrProvinceName         = State or Province Name (full name)
stateOrProvinceName_default = JiangSu
localityName                = Locality Name (eg, city)
localityName_default        = NanJing
organizationName            = Organization Name (eg, company)
organizationName_default    = ovocom
commonName                  = CommonName (e.g. server FQDN or YOUR name)
commonName_max              = 64
commonName_default          = localhost # 此值应该包含在 [alt_names] 中
[ req_ext ]
subjectAltName = @alt_names
[alt_names]
DNS.1   = localhost
IP      = 127.0.0.1
```

此中的 `server.conf` 文件是 服务器的 OpenSSL 的配置文件；
`server.conf` 文件的主要作用是提供一个结构化的方式来定义和生成 `SSL/TLS` 证书请求文件（`server.csr`），确保生成的证书(`server.crt`)符合安全和合规要求，并支持多种使用场景。通过合理配置该文件，用户可以高效地管理和生成所需的证书。

##### `server.conf` 的 CN 和  `ca.conf` 的 CN 不同
- **用途不同**：
`server.conf` 的 CN 是为了标识具体的服务器或域名，而 `ca.conf` 的 CN 是为了标识证书颁发机构。

- **安全性和信任**：
服务器的 CN 需要与客户端实际访问的域名匹配，以确保 SSL/TLS 连接的安全性。而 CA 的 CN 则是为了建立信任链，通常是组织名称或 CA 的标识。


##### 主题备用名称(SAN)
在 Go 1.15 版本之后废弃了 `CommonName`, 而是要使用 `主题备用名称：Subject Alternative Name` (SAN) 来验证证书的有效性.

- **定义**: SAN 是一个扩展字段，用于在同一证书中指定多个备用名称。它可以包括多个域名、子域名、IP 地址，甚至电子邮件地址。

- **用途**: SAN 提供了更大的灵活性，允许一个证书保护多个域名或子域名。现代的 SSL/TLS 证书通常使用 SAN 字段。例如，一个证书可以同时包含 `www.example.com`、`api.example.com` 和 `example.org` 等多个名称。

##### CN 和 SAN
**（1）`commonName` (CN)**:
- 定义: `commonName` 是证书中的一个字段，通常用于指定证书的主要主体名称，通常是一个域名或 IP 地址。
- 用途: 这是在 SSL/TLS 证书中最基本的标识；客户端在与服务器建立安全连接时，会检查请求的主机名(URL)是否与证书中的 `commonName` 匹配。例如，如果证书的 `commonName` 是 `www.example.com`，那么客户端访问 `https://www.example.com` 时，证书将被认为是有效的。

**（2）区别**:
- 数量: `commonName` 通常只能有一个值，而 SAN 可以包含多个值。
- 用途: `commonName` 是主要的标识符，而 SAN 用于扩展和补充 `commonName`，提供更多的名称选项。
- 验证方式: 在某些情况下，如果 SAN 存在，现代浏览器和客户端会优先验证 SAN 中的名称，而不是 `commonName`。

**(3)联系**:
两者共同作用于证书的验证，确保客户端能够正确识别和信任服务器的身份。
在许多情况下，`commonName` 和 SAN 可以包含相同的值。为了确保**兼容性**，通常建议在 SAN 中包含 `commonName` 的值。
例如，如果 `commonName` 是 `www.example.com`，那么在 SAN 中也可以包含 `www.example.com`，以确保无论通过哪个字段进行验证，证书都能被正确识别。

#### server.key
生成`server`的非对称加密的私钥(`server.key`)
```bash
openssl genrsa -out server.key 2048
```

**KEY（Private Key）**
`server.key`：这是服务器的私钥文件。它与 CSR 中的公钥配对，私钥用于解密信息。

**server的私钥的作用**：
client向server通信时，client使用server的公钥进行加密，server收到包之后使用私钥进行解密。

> 注：server向client发包时，使用的是client的公钥进行加密；client收到包之后，使用自身的私钥进行解密。

#### server.csr
**CSR（证书签名请求：Certificate Signing Request）**

**server.csr**：这是一个证书签名请求文件。
它包含了关于申请者的信息（如域名、组织名称、地点等）以及公钥。CSR文件 用于向证书颁发机构（CA）请求数字证书。
用户生成 CSR 后，会将其发送给 CA，CA 根据 CSR 中的信息签发相应的证书（crt文件，比如：server.crt）。


生成CSR，如下所示：
```bash
openssl req -new -key server.key -out server.csr -config server.conf
```


#### server.crt

**CRT（证书：Certificate）**
**server.crt**：这是一个数字证书文件。它是由 CA 签发的，包含了公钥和有关持有者的信息，以及 CA 的数字签名。这个文件用于客户端验证服务器的身份，并用于加密通信。


**请求 CA 签发证书，这里和生成 CA证书类似，不过最后一步由 CA 证书进行签名，而不是自签名**。如下所示：
```bash
openssl x509 -req -days 3650 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -extensions req_ext -extfile server.conf
```

 **CA 使用其私钥（ca.key）签署客户端和服务器的 CSR，生成相应的证书（client.crt 和 server.crt），并将 CA 证书（ca.crt）分发给需要验证这些证书的客户端和服务器**。

#### server.pem
`server.pem 「Server Certificate File (PEM format)」`有时候指的就是 `server.crt（服务器的证书文件）` 。

有时候 `server.pem` 包含了私钥和证书。
将私钥和证书合并到一个PEM文件中，形成`server.pem`。可以使用以下命令：
```bash
cat server.crt server.key > server.pem
```
这样，`server.pem`文件就包含了服务器的公钥证书和私钥。


### 生成客户端证书

#### client.key

**client.key**：客户端私钥

生成私钥:
```bash
openssl genrsa -out client.key 2048
```

#### client.csr
**client.csr**：客户端证书请求（Certificate Signing Request）；
**作用**：包含客户端的公钥和身份信息，向 CA 请求签署证书。

客户端不需要配置文件, 直接生成CSR，如下所示：
```bash
openssl req -new -key client.key -out client.csr
```

#### client.crt(client.pem)
**client.csr**：客户端证书（Certificate）；
- **作用**：用于在 TLS 连接中证明客户端的身份。
- **使用时机**：在客户端与服务器建立连接时，在需要客户端身份验证的情况下，客户端客户端会向服务器提供此证书以进行身份验证。

请求 CA 签发证书（生成 client.crt）:
```bash
openssl x509 -req -days 3650 -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt
```

### 小结

现在已经准备好证书了. 服务端需要 `server.crt` 和 `server.key`, 客户端需要 `client.crt` 和 `client.key`. 此外, 两者都需要 `ca.crt`.

## 服务端实现

```go
package main

import (
	/*
		tls 和 x509:
		用于处理 TLS 连接和证书。
	*/
	"crypto/tls"
	"crypto/x509"
	"net"
	"os"
    "fmt"

	/*
		grpc 和 credentials：
		gRPC 库和用于 TLS 的凭据管理。
	*/
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	/*
		example/proto： pb文件中指定的 go_package: 即生成的go的package的路径。
	*/
    "example/proto"
)

func main() {

	/*
	使用 `tls.LoadX509KeyPair` 加载服务器的证书和私钥。
	`server.crt` 是公钥证书，`server.key` 是私钥。
	*/
	pair, err := tls.LoadX509KeyPair("./server.crt", "./server.key")
	if err != nil {
		fmt.Printf("Failed to load certificates: %s", err)
		return
	}

	/*
		在使用 gRPC 和 TLS 的服务器与客户端通信中，根证书的读取和证书池的设置是至关重要的。
		CA根证书（ca.crt）用于验证客户端证书的合法性。
		服务器通过读取根证书来建立一个信任链，以确保它能够验证客户端提供的证书是否由该根证书签发。
		
	*/
	certPool := x509.NewCertPool()
	ca, err := os.ReadFile("./ca.crt") // 读取根证书
	if err != nil {
		fmt.Printf("Failed to read ca certificate: %s", err)
		return
	}
	/*
	证书池（`CertPool`）是一个用于存储受信任的根证书的集合。它允许服务器信任多个根证书。
	使用 `AppendCertsFromPEM` 方法将读取到的根证书添加到证书池中。
	这样，服务器就可以在 TLS 握手期间使用这些证书来验证客户端证书。
	*/
	if ok := certPool.AppendCertsFromPEM(ca); !ok {
		fmt.Println("Failed to append ca certificate")
		return
	}
	/*
	 **验证客户端证书**：
	    服务器使用 `certPool` 中的根证书来验证客户端证书是否有效。
	    验证过程包括：
	    - 确认客户端证书是否由信任的 CA 签发。
	    - 检查客户端证书是否在有效期内。
	    - 检查证书的撤销状态（如果实现了 CRL 或 OCSP）。
    */
    /*
		创建一个新的 TLS 凭证，设置证书、客户端认证要求和信任的 CA 列表。
    */
	creds := credentials.NewTLS(&tls.Config{
		/*
		`Certificates` 包含服务器的证书和私钥。
		*/
		Certificates: []tls.Certificate{pair},
		/*
		`ClientAuth` 字段用于指定服务端对客户端证书的验证要求。它的值可以是 `tls.NoClientCert`、`tls.RequestClientCert`、`tls.RequireAnyClientCert`、`tls.RequireAndVerifyClientCert` 等。
		
		`tls.RequireAndVerifyClientCert`：表示服务端必须要求客户端提供证书，并对其进行验证。如果客户端没有提供有效的证书，连接将被拒绝。
		`tls.RequireAnyClientCert`: 允许客户端提供证书，但不强制要求。即使客户端没有提供证书，连接仍然可以建立。服务端会接受任何客户端证书（如果提供的话），但不会验证其有效性。
		*/
		ClientAuth:   tls.RequireAndVerifyClientCert,
		/*
		`ClientCAs` : 包含ca根证书(ca.crt)的证书池，为了验证client的证书。
		*/
		ClientCAs:    certPool,
	})
	
	/* 创建一个新的 gRPC 服务器实例，并将 TLS 凭据传递给它。 */
	s := grpc.NewServer(grpc.Creds(creds))

	/* 注册实现了 `ArtworkService` interface 接口的服务器实例。 */
	proto.RegisterArtworkServiceServer(s, &ArtworkServiceServer{})
	lis, err := net.Listen("tcp", ":39010")
    if err != nil {
        fmt.Printf("Failed to listen: %s", err)
        return
    }
    /* 启动服务器 */
	if err := s.Serve(lis); err != nil {
		fmt.Printf("Failed to serve: %s", err)
		return
	}
}
```

## 客户端实现

```go
package main

import (
	/*
		`crypto/tls` 和 `crypto/x509` 用于处理 TLS 证书
	*/
	"crypto/tls"
	"crypto/x509"
	"os"

	"example/proto"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
)


func main() {
	/* 加载客户端证书和私钥。 */
	pair, err := tls.LoadX509KeyPair("./client.crt", "./client.key")
	if err != nil {
		fmt.Printf("Failed to load certificates: %s", err)
		return
	}
	/*
		创建一个新的证书池。读取 CA 证书文件（ca.crt）加入到证书池中。
		为了使对 server.crt（服务器的证书）进行验证。
	*/
	certPool := x509.NewCertPool()
	ca, err := os.ReadFile("./ca.crt")
	if err != nil {
        fmt.Printf("Failed to read ca certificate: %s", err)
		return
	}
	if ok := certPool.AppendCertsFromPEM(ca); !ok {
		fmt.Println("Failed to append ca certificate")  
		return
	}
	/*
		创建一个新的 TLS 凭证。
		客户端的 `NewTLS` 配置主要关注服务器的身份验证和连接的安全性。
	*/
	cred := credentials.NewTLS(&tls.Config{
		/*
		`Certificates` 包含客户端 的证书和私钥。
		因为服务器的实现在创建 TLS 凭证时，需要对于Client进行验证。
		*/
		Certificates: []tls.Certificate{pair},
		/*
		指定服务器的主机名，TLS 连接时会验证服务器证书的 `CommonName` 或 `Subject Alternative Name` 是否和此中的设置匹配。
		*/
		ServerName:   "localhost", // 服务端生成证书时的 commonName
		/*
		将之前创建的证书池（`certPool`）设置为信任的根 CA，这样客户端在验证服务器证书时会使用这个池。
		*/
		RootCAs:      certPool,
	})
	/*
	建立与 gRPC 服务器的连接:
	`grpc.Dial` 函数尝试连接到指定的地址和端口（`127.0.0.1:39010`），并使用之前创建的 TLS 凭证（`cred`）进行安全连接。
	*/
	conn, err := grpc.Dial("127.0.0.1:39010", grpc.WithTransportCredentials(cred))
	if err != nil {
		fmt.Printf("Failed to dial: %s", err)
		return
	}
	defer conn.Close()
	/*
		创建 gRPC 客户端实例。
		使用生成的 gRPC 代码（`xx_grpc.pb.go`）创建一个 `ArtworkServiceClient` 实例，后续可以通过该客户端调用服务端的方法。
	*/
    client := proto.NewArtworkServiceClient(conn)
    
    // TODO: grpc 调用 server的 方法。
}
```
# gRPC拦截器（interceptor）

## 介绍
gRPC 提供了 `Interceptor` 功能，包括客户端拦截器和服务端拦截器。==拦截器就相当于是一个包装的中间件==。

在常规的 `HTTP Server`中，我们可以设置有一个中间件将我们的处理程序进行包装。中间件可以在拦截到发送给 `handler` 的请求，且可以拦截 `handler` 返回给客户端的响应。

![](attachments/Pasted%20image%2020241222203545.png)

gRPC 不同，它允许在服务器和客户端都使用拦截器。

## 使用场景

**验证**：
在自定义Token认证的示例中，认证信息是由每个服务中的方法处理并认证的，如果有大量的接口方法，这种姿势就太不优雅了，每个接口实现都要先处理认证信息。
这个时候interceptor就可以用来解决了这个问题，在请求被转到具体接口之前处理认证信息，一处认证，到处无忧。

**日志**：
在客户端，我们增加一个请求日志，记录请求相关的参数和耗时等等。

## 拦截器分类


###  角色分类
gRPC 拦截器主要分为两种：客户端拦截器（ClientInterceptor），服务端拦截器（ServerInterceptor）。
顾名思义，分别于请求的两端执行相应的前拦截处理。


### 类型分类
在 gRPC 中，根据拦截的方法类型不同可以分为拦截 Unary 方法的**一元拦截器（grpc.UnaryInterceptor）**，和作用于 Stream 方法的**流拦截器（grpc.StreamInterceptor）
**。

同时还分为**服务端拦截器**和**客户端拦截器**，所以一共有以下4种类型:
```go
- grpc.UnaryServerInterceptor
- grpc.StreamServerInterceptor
- grpc.UnaryClientInterceptor
- grpc.StreamClientInterceptor
```


### 客户端拦截器
使用客户端拦截器 只需要在 `Dial`的时候指定相应的 `DialOption` 即可。

#### Unary Interceptor
客户端一元拦截器类型为 `grpc.UnaryClientInterceptor`，具体如下：

```go
type UnaryClientInterceptor func(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error

```

可以看到，所谓的**拦截器其实就是一个函数**，可以分为`预处理(pre-processing)`、`调用RPC方法(invoking RPC method)`、`后处理(post-processing)`三个阶段。

参数含义如下:

- ctx：Go语言中的上下文，一般和 Goroutine 配合使用，起到超时控制的效果
- method：当前调用的 RPC 方法名
- req：本次请求的参数，只有在`处理前`阶段修改才有效
- reply：本次请求响应，需要在`处理后`阶段才能获取到
- cc：gRPC 连接信息
- invoker：可以看做是当前 RPC 方法，一般在拦截器中调用 invoker 能达到调用 RPC 方法的效果，当然底层也是 gRPC 在处理。
- opts：本次调用指定的 options 信息

作为一个客户端拦截器，可以在`处理前`检查 req 看看本次请求带没带 token 之类的鉴权数据，没有的话就可以在拦截器中加上。

#### Stream Interceptor
```go
type StreamClientInterceptor func(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, streamer Streamer, opts ...CallOption) (ClientStream, error)

```

由于 StreamAPI 和 UnaryAPI有所不同，因此拦截器方面也有所区别，比如 req 参数变成了 streamer 。同时其拦截过程也有所不同，不在是处理 req resp，而是对 streamer 这个流对象进行包装，比如说实现自己的 SendMsg 和 RecvMsg 方法。

然后在这些方法中的`预处理(pre-processing)`、`调用RPC方法(invoking RPC method)`、`后处理(post-processing)`各个阶段加入自己的逻辑。

### 服务端拦截器

#### Unary Interceptor
```go
type UnaryServerInterceptor func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)

```

参数具体含义如下：
```text
- ctx context.Context：请求上下文
- req interface{}：RPC 方法的请求参数
- info *UnaryServerInfo：RPC 方法的所有信息
- handler UnaryHandler：RPC 方法真正执行的逻辑
```


#### Stream Interceptor

```go
type StreamServerInterceptor func(srv interface{}, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
```


### UnaryInterceptor

一元拦截器可以分为3个阶段：

- 1）预处理(pre-processing)
- 2）调用RPC方法(invoking RPC method)
- 3）后处理(post-processing)


#### Client
```go
// unaryInterceptor 一个简单的 unary interceptor 示例。
func myUnaryInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
	// pre-processing
	start := time.Now()
	err := invoker(ctx, method, req, reply, cc, opts...) // invoking RPC method
	// post-processing
	end := time.Now()
	logger("RPC: %s, req:%v start time: %s, end time: %s, err: %v", method, req, start.Format(time.RFC3339), end.Format(time.RFC3339), err)
	return err
}

```

`invoker(ctx, method, req, reply, cc, opts...)` 是真正调用 `RPC` 方法。因此我们可以在调用前后增加自己的逻辑：比如调用前检查一下参数之类的，调用后记录一下本次请求处理耗时等。

建立连接时通过 `grpc.WithUnaryInterceptor` 指定要加载的一元拦截器即可。
```go
func main() {
	flag.Parse()

	creds, err := credentials.NewClientTLSFromFile(data.Path("x509/server.crt"), "www.lixueduan.com")
	if err != nil {
		log.Fatalf("failed to load credentials: %v", err)
	}

	/*
	建立连接时指定要加载的拦截器.
	myUnaryInterceptor: 自定义的一元拦截器实现
	*/
	conn, err := grpc.Dial(*addr, grpc.WithTransportCredentials(creds), grpc.WithUnaryInterceptor(myUnaryInterceptor))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()

	client := ecpb.NewEchoClient(conn)
	callUnaryEcho(client, "hello world")
}

```

#### server
服务端的一元拦截器和客户端类似：
```go
func unaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	start := time.Now()
	m, err := handler(ctx, req)
	end := time.Now()
	// 记录请求参数 耗时 错误信息等数据
	logger("RPC: %s,req:%v start time: %s, end time: %s, err: %v", info.FullMethod, req, start.Format(time.RFC3339), end.Format(time.RFC3339), err)
	return m, err
}

```

服务端则是在 `NewServer` 时指定拦截器：
```go
func main() {
	flag.Parse()

	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	creds, err := credentials.NewServerTLSFromFile(data.Path("x509/server.crt"), data.Path("x509/server.key"))
	if err != nil {
		log.Fatalf("failed to create credentials: %v", err)
	}

	s := grpc.NewServer(grpc.Creds(creds), grpc.UnaryInterceptor(unaryInterceptor))

	pb.RegisterEchoServer(s, &server{})
	log.Println("Server gRPC on 0.0.0.0" + fmt.Sprintf(":%d", *port))
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

#### test
Server 端：
```text
2021/01/24 19:18:09 Server gRPC on 0.0.0.0:50051
unary echoing message "hello world"
LOG:    RPC: /echo.Echo/UnaryEcho,req:message:"hello world" start time: 2021-01-24T19:18:10+08:00, end time: 2021-01-24T19:18:10+08:00, err: <nil>
```

client端：
```text
LOG:    RPC: /echo.Echo/UnaryEcho, req:message:"hello world" start time: 2021-01-24T19:18:10+08:00, end time: 2021-01-24T19:18:10+08:00, err: <nil>
UnaryEcho:  hello world

```

### StreamInterceptor
流拦截器过程和一元拦截器有所不同，不过同样可以分为3个阶段：

- 1）预处理(pre-processing)
- 2）调用RPC方法(invoking RPC method)
- 3）后处理(post-processing)

预处理阶段和一元拦截器类似，但是调用RPC方法和后处理这两个阶段则完全不同。

`StreamAPI` 的请求和响应都是通过 `Stream` 进行传递的，更进一步是通过 `Streamer` 调用 `SendMsg` 和 `RecvMsg` 这两个方法获取的。

然后 `Streamer` 又是调用`RPC`方法来获取的，所以在流拦截器中我们可以 **对 Streamer 进行包装，然后实现 SendMsg 和 RecvMsg 这两个方法**。

#### Client
本例中通过结构体嵌入的方式，对 Streamer 进行包装，在 SendMsg 和 RecvMsg 之前打印出具体的值。

```go
// wrappedStream  用于包装 grpc.ClientStream 结构体并拦截其对应的方法。
type wrappedStream struct {
	grpc.ClientStream
}

func newWrappedStream(s grpc.ClientStream) grpc.ClientStream {
	return &wrappedStream{s}
}

func (w *wrappedStream) RecvMsg(m interface{}) error {
	logger("Receive a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ClientStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m interface{}) error {
	logger("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ClientStream.SendMsg(m)
}

// streamInterceptor 一个简单的 stream interceptor 示例。
func streamInterceptor(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
	s, err := streamer(ctx, desc, cc, method, opts...)
	if err != nil {
		return nil, err
	}
    // 返回的是自定义的封装过的 stream
	return newWrappedStream(s), nil
}

```

连接时则通过 `grpc.WithStreamInterceptor` 指定要加载的拦截器。
```go
func main() {
	flag.Parse()

	creds, err := credentials.NewClientTLSFromFile(data.Path("x509/server.crt"), "www.lixueduan.com")
	if err != nil {
		log.Fatalf("failed to load credentials: %v", err)
	}

	// 建立连接时指定要加载的拦截器
	conn, err := grpc.Dial(*addr, grpc.WithTransportCredentials(creds), grpc.WithStreamInterceptor(streamInterceptor))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()

	client := ecpb.NewEchoClient(conn)
	// callUnaryEcho(client, "hello world")
	callBidiStreamingEcho(client)
}

```

#### Server
和客户端类似。

```go
type wrappedStream struct {
	grpc.ServerStream
}

func newWrappedStream(s grpc.ServerStream) grpc.ServerStream {
	return &wrappedStream{s}
}

func (w *wrappedStream) RecvMsg(m interface{}) error {
	logger("Receive a message (Type: %T) at %s", m, time.Now().Format(time.RFC3339))
	return w.ServerStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m interface{}) error {
	logger("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ServerStream.SendMsg(m)
}

func streamInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
	// 包装 grpc.ServerStream 以替换 RecvMsg SendMsg这两个方法。
	err := handler(srv, newWrappedStream(ss))
	if err != nil {
		logger("RPC failed with error %v", err)
	}
	return err
}

```
相似的，通过`grpc.StreamInterceptor`指定要加载的拦截器。

```go
func main() {
	flag.Parse()

	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	creds, err := credentials.NewServerTLSFromFile(data.Path("x509/server.crt"), data.Path("x509/server.key"))
	if err != nil {
		log.Fatalf("failed to create credentials: %v", err)
	}

	s := grpc.NewServer(grpc.Creds(creds), grpc.StreamInterceptor(streamInterceptor))

	pb.RegisterEchoServer(s, &server{})
	log.Println("Server gRPC on 0.0.0.0" + fmt.Sprintf(":%d", *port))
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```

#### Test
Server 端：
```go
lixd@17x:~/17x/projects/grpc-go-example/features/interceptor/server$ go run main.go 2021/01/24 19:58:12 Server gRPC on 0.0.0.0:50051 LOG: Receive a message (Type: *echo.EchoRequest) at 2021-01-24T19:58:14+08:00 bidi echoing message "Request 1" LOG: Send a message (Type: *echo.EchoResponse) at 2021-01-24T19:58:14+08:00 LOG: Receive a message (Type: *echo.EchoRequest) at 2021-01-24T19:58:14+08:00 bidi echoing message "Request 2" LOG: Send a message (Type: *echo.EchoResponse) at 2021-01-24T19:58:14+08:00 LOG: Receive a message (Type: *echo.EchoRequest) at 2021-01-24T19:58:14+08:00
```

Client 端：
```go
lixd@17x:~/17x/projects/grpc-go-example/features/interceptor/client$ go run main.go 
LOG:    Send a message (Type: *echo.EchoRequest) at 2021-01-24T19:58:14+08:00
LOG:    Send a message (Type: *echo.EchoRequest) at 2021-01-24T19:58:14+08:00
LOG:    Receive a message (Type: *echo.EchoResponse) at 2021-01-24T19:58:14+08:00
BidiStreaming Echo:  Request 1
LOG:    Receive a message (Type: *echo.EchoResponse) at 2021-01-24T19:58:14+08:00
BidiStreaming Echo:  Request 2
LOG:    Receive a message (Type: *echo.EchoResponse) at 2021-01-24T19:58:14+08:00

```


### 小结
**（1）拦截器分类与定义**
gRPC 拦截器可以分为：一元拦截器和流拦截器，服务端拦截器和客户端拦截器。

一共有以下4种类型:
```go
- grpc.UnaryServerInterceptor
- grpc.StreamServerInterceptor
- grpc.UnaryClientInterceptor
- grpc.StreamClientInterceptor
```

拦截器本质上就是一个特定类型的函数，所以实现拦截器只需要实现对应类型方法（**方法签名相同**）即可。


**（2）拦截器执行过程**

**(2.1) 一元拦截器**
```go
- 1）预处理
- 2）调用RPC方法
- 3）后处理
```

**(2.2) 流拦截器**
```go
- 1）预处理
- 2）调用RPC方法 获取 Streamer
- 3）后处理
    - 调用 SendMsg 、RecvMsg 之前
    - 调用 SendMsg 、RecvMsg
    - 调用 SendMsg 、RecvMsg 之后
```


**(3) 多拦截器的使用及执行顺序**

配置多个拦截器时，会按照参数传入顺序依次执行
所以，如果想配置一个 `Recovery` 拦截器，则必须放在第一个，放在最后则无法捕获前面执行的拦截器中触发的 `panic`。

同时也可以将 一元拦截器和流拦截器一起配置，gRPC 会根据不同方法选择对应类型的拦截器执行。

最后推荐一下这个 [go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware)，该仓库提供了多种常用拦截器。

## 实现范例
### 目录结构
```text
|—— hello_interceptor/
    |—— client/
        |—— main.go   // 客户端
    |—— server/
        |—— main.go   // 服务端
|—— keys/             // 证书目录
    |—— server.key
    |—— server.pem
|—— proto/
    |—— hello/
        |—— hello.proto   // proto描述文件
        |—— hello.pb.go   // proto编译后文件
```

### 服务端interceptor
```go
# cat hello_interceptor/server/main.go
package main

import (
    "fmt"
    "net"

    pb "github.com/jergoo/go-grpc-example/proto/hello"

    "golang.org/x/net/context"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"       // grpc 响应状态码
    "google.golang.org/grpc/credentials" // grpc认证包
    "google.golang.org/grpc/grpclog"
    "google.golang.org/grpc/metadata" // grpc metadata包
)

const (
    // Address gRPC服务地址
    Address = "127.0.0.1:50052"
)

// 定义helloService并实现约定的接口
type helloService struct{}

// HelloService Hello服务
var HelloService = helloService{}

// SayHello 实现Hello服务接口
func (h helloService) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloResponse, error) {
    resp := new(pb.HelloResponse)
    resp.Message = fmt.Sprintf("Hello %s.", in.Name)

    return resp, nil
}

func main() {
    listen, err := net.Listen("tcp", Address)
    if err != nil {
        grpclog.Fatalf("Failed to listen: %v", err)
    }

    var opts []grpc.ServerOption

    // TLS认证
    creds, err := credentials.NewServerTLSFromFile("../../keys/server.pem", "../../keys/server.key")
    if err != nil {
        grpclog.Fatalf("Failed to generate credentials %v", err)
    }

    opts = append(opts, grpc.Creds(creds))

    // 注册interceptor
    opts = append(opts, grpc.UnaryInterceptor(myInterceptor))

    // 实例化grpc Server
    s := grpc.NewServer(opts...)

    // 注册HelloService
    pb.RegisterHelloServer(s, HelloService)

    grpclog.Println("Listen on " + Address + " with TLS + Token + Interceptor")

    s.Serve(listen)
}

// auth 验证Token
func auth(ctx context.Context) error {
    md, ok := metadata.FromContext(ctx)
    if !ok {
        return grpc.Errorf(codes.Unauthenticated, "无Token认证信息")
    }

    var (
        appid  string
        appkey string
    )

    if val, ok := md["appid"]; ok {
        appid = val[0]
    }

    if val, ok := md["appkey"]; ok {
        appkey = val[0]
    }

    if appid != "101010" || appkey != "i am key" {
        return grpc.Errorf(codes.Unauthenticated, "Token认证信息无效: appid=%s, appkey=%s", appid, appkey)
    }

    return nil
}

// interceptor 拦截器
func myInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	// 预处理
    err := auth(ctx)
    if err != nil {
        return nil, err
    }
    // 继续处理请求
    return handler(ctx, req)
    
}
```

### 客户端interceptor

```go
# cat hello_intercepror/client/main.go
package main

import (
    "time"

    pb "github.com/jergoo/go-grpc-example/proto/hello" // 引入proto包

    "golang.org/x/net/context"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials" // 引入grpc认证包
    "google.golang.org/grpc/grpclog"
)

const (
    // Address gRPC服务地址
    Address = "127.0.0.1:50052"

    // OpenTLS 是否开启TLS认证
    OpenTLS = true
)

// customCredential 自定义认证
type customCredential struct{}

// GetRequestMetadata 实现自定义认证接口
func (c customCredential) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
    return map[string]string{
        "appid":  "101010",
        "appkey": "i am key",
    }, nil
}

// RequireTransportSecurity 自定义认证是否开启TLS
func (c customCredential) RequireTransportSecurity() bool {
    return OpenTLS
}

func main() {
    var err error
    var opts []grpc.DialOption

    if OpenTLS {
        // TLS连接
        creds, err := credentials.NewClientTLSFromFile("../../keys/server.pem", "server name")
        if err != nil {
            grpclog.Fatalf("Failed to create TLS credentials %v", err)
        }
        opts = append(opts, grpc.WithTransportCredentials(creds))
    } else {
        opts = append(opts, grpc.WithInsecure())
    }

    // 指定自定义认证
    opts = append(opts, grpc.WithPerRPCCredentials(new(customCredential)))
    // 指定客户端interceptor
    opts = append(opts, grpc.WithUnaryInterceptor(myClientInterceptor))

    conn, err := grpc.Dial(Address, opts...)
    if err != nil {
        grpclog.Fatalln(err)
    }
    defer conn.Close()

    // 初始化客户端
    c := pb.NewHelloClient(conn)

    // 调用方法
    req := &pb.HelloRequest{Name: "gRPC"}
    res, err := c.SayHello(context.Background(), req)
    if err != nil {
        grpclog.Fatalln(err)
    }

    grpclog.Println(res.Message)
}

// interceptor 客户端拦截器
func myClientInterceptor(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    start := time.Now()
    err := invoker(ctx, method, req, reply, cc, opts...)
    grpclog.Printf("method=%s req=%v rep=%v duration=%s error=%v\n", method, req, reply, time.Since(start), err)
    return err
}
```

### 运行结果
服务端：
```
$ cd hello_inteceptor/server && go run main.go
Listen on 127.0.0.1:50052 with TLS + Token + Interceptor
```

客户端：
```
$ cd hello_inteceptor/client && go run main.go
method=/hello.Hello/SayHello req=name:"gRPC"  rep=message:"Hello gRPC."  duration=33.879699ms error=<nil>

Hello gRPC.
```

## 多拦截器

采用开源项目 [go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware) 可以实现多个拦截器。
> `go-grpc-middleware` 中也提供了多种常用 `interceptor` ，可以直接使用

gRPC-go 在 `v1.28.0`版本增加了`多 interceptor` 支持，可以在不借助第三方库（`go-grpc-middleware`）的情况下添加多个 `interceptor` 了。


# 使用 buf 替代 protoc 生成和管理代码 
参考：[# 如何优化 Go 项目中的 Protocol Buffers 开发工作流？Buf操作困难吗？](https://insight.xiaoduoai.com/intelligent-frontiers/tech/what-are-the-protobuf-optimization-tools-is-it-difficult-to-operate-buf.html)

## 背景
`buf`出现之前，我们只能使用`protoc`，配合一堆参数来生成代码。

```bash
cuiwei@weideMacBook-Pro protobuf % protoc --go_out=. --go-grpc_out=require_unimplemented_servers=false:. ./helloworld.proto 

cuiwei@weideMacBook-Pro protobuf % protoc -I . --grpc-gateway_out ../gen \
--grpc-gateway_opt logtostderr=true \
--grpc-gateway_opt paths=source_relative \
--grpc-gateway_opt generate_unbound_methods=true \
helloworld.proto
```

可以看到使用protoc的时候，当使用的插件逐渐变多，插件参数逐渐变多时，命令行执行并不是很方便和直观。

其次依赖某些外部的`protobuf`文件时，只能通过拷贝到本地的方式，也不够方便。

## 介绍
**Buf** 是一个专注于**优化和改进 Protocol Buffers（protobuf）开发体验**的工具，旨在解决传统 `protoc` 编译器在处理 `.proto` 文件时遇到的一些痛点，如**重复代码生成、版本控制缺乏、linting（代码规范检查）不足**等问题。

Buf 提供了对 `.proto` 文件的高效管理、代码生成、错误检测、和依赖管理等功能，简化了整个 protobuf 工作流，尤其适合于**大型项目和跨团队协作**。

## buf的功能

buf 的亮点：
1. 为 protobuf 提供依赖管理
2. 使用 yaml 配置简化代码生成命令
3. 提供 lint 静态检查工具
4. 提供 breaking change 静态检查工具
5. 提供 format 格式化工具
6. 自己实现 compiler 取代 protoc

### Linting（代码风格检查）

Buf 内置了强大的 linting 功能，它会检查 `.proto` 文件中的编码风格、结构和规范，确保代码符合最佳实践。
这在协作开发中尤为重要，能帮助开发团队保持代码风格的一致性，避免潜在的错误。

- Buf 提供了一套默认的规则，你也可以根据项目需求自定义规则。
- 支持对项目中的 `.proto` 文件执行静态分析，提示潜在问题或不符合规范的地方。

例如，运行 lint 命令来检查项目中的 `.proto` 文件：
```go
buf lint
```

### 远程仓库托管 protobuf 
Buf 支持通过远程仓库托管 protobuf 模块，例如将 `.proto` 文件发布到一个中央仓库，方便不同项目共享同一套 `.proto` 定义。

## buf 和 其他工具的关系
### 和 protoc的关系
`protoc` 是 protobuf 的官方编译器，Buf 可以被看作是对 `protoc` 的增强和替代。
Buf 使用与 `protoc` 相同的基础编译机制，但在其上增加了模块化、linting、版本控制等功能。
对于复杂的项目或团队协作，Buf 提供了更高级的功能，使得 protobuf 的管理更加简便和高效。

### 和 protoc-gen-go的关系

`protoc-gen-go` 是 Go 语言专用的 Protocol Buffers 插件，它负责将 `.proto` 文件生成 Go 代码。Buf 在其代码生成过程中会自动调用 `protoc-gen-go` 插件，因此 Buf 本质上是封装了 `protoc` 及其插件的生成过程，使整个工作流程更加自动化和可配置化。

## 安装

```bash
curl -sSL https://github.com/bufbuild/buf/releases/download/v1.8.0/buf-Linux-x86_64 -o /usr/local/bin/buf 
chmod +x /usr/local/bin/buf

或者

go install github.com/bufbuild/buf/cmd/buf@v1.34.0
```


检查是否安装成功：
```bash
buf --version
1.34.0

```
## 使用
### 初始化模块
在pb文件的根目录执行，为这个`pb`目录创建一个`buf`的模块。此后便可以使用`buf`的各种命令来管理这个buf模块了。
```bash
# buf mod init
```
这个命令会生成 `buf.yaml` 和 `buf.lock` 文件，这些文件定义了项目中 protobuf 文件的根目录和依赖管理。
```yaml
# buf.yaml
version: v1
lint:
  use:
    - DEFAULT
```
### 生成代码
插件：和使用`protoc`一样，该装的插件一样要装

#### 插件模版：buf.gen.yaml
创建 `buf.gen.yaml`文件,该文件替换了`protoc`所配置的插件等,即用 文件的形式 替代命令行的形式。
```yaml
# buf.gen.yaml
version: v1
plugins:
  - plugin: go
    out: ecommerce
    opt:
      - paths=source_relative
  - plugin: go-grpc
    out: ecommerce
    opt:
      - paths=source_relative
  - name: grpc-gateway
    out: ecommerce
    opt:
      - paths=source_relative
      - generate_unbound_methods=true
  - name: openapiv2
    out: doc
    opt:
      - logtostderr=true

```

#### 代码的生成
```go
# buf generate 
```

`buf generate` 命令将会
- 搜索每一个`buf.yaml`配置里的所有`protobuf`文件
- 复制所有`protobuf`文件到内存
- 编译所有`protobuf`文件
- 执行模版文件里的每一个插件

# grpc 的 keepalive
参考：[# gRPC之grpc keepalive](https://blog.csdn.net/qq_30614345/article/details/134585507)


# gRPC的metadata

参考：[# gRPC之metadata](https://blog.csdn.net/qq_30614345/article/details/134470773)



# 参考
```bash
# protobuf3 官方文档：
https://protobuf.dev/programming-guides/proto3/

# grpc 官方文档：
https://grpc.io/docs/languages/

# gRPC(Go)教程(五)---拦截器Interceptor【系列文章很好】
https://www.lixueduan.com/posts/grpc/05-interceptor/

# GO-grpc 实践指南【整个系列都挺好的】
https://jergoo.gitbooks.io/go-grpc-practice-guide/content/chapter3/gateway.html

# Go 的 gRPC 和 Protocol Buffers---Quick Start
https://krau.top/posts/grpc-and-protobuf-in-go-start
# Go 的 gRPC 和 Protocol Buffers---TLS 认证
https://krau.top/posts/grpc-and-protobuf-in-go-tls

# ProtoBuf 入门教程[包含入门，定义消息、数据类型、各种类型等]
https://www.tizi365.com/archives/367.html

# 长文图解Google的protobuf思考、设计、应用
https://www.eet-china.com/mp/a63366.html

# golang 生成 .pb.go 文件
https://juejin.cn/post/7373645539457482793

# 在Golang中使用Protobuf
https://blog.csdn.net/kevin_tech/article/details/104093914


```