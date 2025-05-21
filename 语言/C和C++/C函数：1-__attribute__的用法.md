```table-of-contents
```
# `__attribute__((__packed__))`
在C或C++中，使用 `__attribute__((__packed__))` 修饰一个结构体的成员变量是没有作用的。
`__attribute__((__packed__))` 只能应用于整个结构体或联合体，而不能单独用于结构体的某个成员。

## 特性
###  `__packed__` 的作用范围：整体结构体
当你将整个结构体声明为 `__packed__` 时，结构体内的所有成员（包括嵌套的结构体）都将被紧凑排列，不会有额外的填充。
```c
struct __attribute__((__packed__)) MyStruct {
    char a;    // 1 byte
    int b;     // 4 bytes
    short c;   // 2 bytes
};

```
在上面的例子中，`MyStruct` 的内存布局将是紧凑的，`a`、`b`、`c` 将依次排列，没有填充。

### 对结构体的单个成员的无效性
如果你尝试将 `__attribute__((__packed__))` 应用于结构体的单个成员，它不会有任何效果，因为这个属性只影响结构体的整体布局，而不会影响单个成员的对齐。
```c
struct MyStruct {
    char a;  // 1 byte
    int b __attribute__((__packed__));  // 无效
    short c; // 2 bytes
};

```
在这个例子中，`b` 的 `__packed__` 属性是无效的，`MyStruct` 的整体布局仍然会根据 `int` 和 `short` 的对齐要求进行填充。

#### 如何实现成员的紧凑排列
如果你确实需要某个成员的紧凑排列，通常的做法是将该成员放在一个单独的结构体中，并对该结构体使用 `__packed__` 属性。比如：
```c
struct Inner {
    int b;  // 4 bytes
} __attribute__((__packed__));

struct MyStruct {
    char a;  // 1 byte
    struct Inner inner; // 4 bytes, no padding
    short c; // 2 bytes
};
```
在这个例子中，`Inner` 结构体是紧凑的，因此 `MyStruct` 中的 `inner` 成员也会按照紧凑的方式排列。

## 范例
```c
$ cat bbb.c
#include <stdio.h>

struct Inner {
    char a;
    int b;
    int c;
};

struct Inner2 {
    char a;
    int b;
    int c;
}__attribute__((__packed__));

struct Outer {
    short x;
    __attribute__((__packed__)) struct Inner inner;
    long y;
} __attribute__((__packed__));

struct Outer2 {
    short x;
    struct Inner inner;
    long y;
} __attribute__((__packed__));

struct Outer3 {
    short x;
    struct Inner2 inner;
    long y;
} __attribute__((__packed__));


struct Outer4 {
    short x;
    __attribute__((__packed__)) struct Inner inner;
    long y;
};

int main() {
  int size_inner = sizeof(struct Inner);
  int size_inner2 = sizeof(struct Inner2);
  int size_outer = sizeof(struct Outer);
  int size_outer2 = sizeof(struct Outer2);
  int size_outer3 = sizeof(struct Outer3);
  int size_outer4 = sizeof(struct Outer4);
  printf("size inner:%d, size inner2:%d, size outer:%d, size outer2:%d size outer3:%d, size outer4:%d \n",
       size_inner, size_inner2, size_outer, size_outer2, size_outer3, size_outer4);
  return 0;
}
```

测试结果：
```bash
$ ./bbb
size inner:12, size inner2:9, size outer:22, size outer2:22 size outer3:19, size outer4:24
```

# 参考
```bash
```