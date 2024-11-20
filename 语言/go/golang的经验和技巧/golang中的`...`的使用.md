```table-of-contents
```

# 处理变长参数
用于函数有多个不定参数的情况，可以接受多个不确定数量的参数。

## 定义变长参数函数

当定义一个函数时，可以使用 `...` 来表示该函数可以接受可变数量的参数。变长参数在函数内部被当作切片处理。

### 范例
```go
func sum(nums ...int) int {
    total := 0
    for _, num := range nums {
        total += num
    }
    return total
}

func main() {
    fmt.Println(sum(1, 2, 3)) // 输出 6
    fmt.Println(sum(1, 2, 3, 4, 5)) // 输出 15
}
```

## 传递变长参数

当调用一个变长参数函数时，可以直接传递多个参数，也可以传递一个切片并使用 `...` 展开切片。

```go
func main() {
    nums := []int{1, 2, 3, 4, 5}
    fmt.Println(sum(nums...)) // 输出 15
}
// 其中 nums 切片内部的元素数量可以是任意个，sum 函数都能够接受。
```

# slice 被打散进行传递

```go
    var strss= []string{
        "qwr",
        "234",
        "yui",

    }
    var strss2= []string{
        "qqq",
        "aaa",
        "zzz",
        "zzz",
    }
strss=append(strss,strss2...) //strss2的元素被打散一个个append进strss
fmt.Println(strss)

结果：
    [qwr 234 yui qqq aaa zzz zzz]

说明：
	如果没有’…’，面对上面的情况，无疑会增加代码量，有了’…’，是不是感觉简洁了许多
```



# 标识数组元素个数



```go
//这里，...意味着数组的元素个数：
stooges1 := [...]string{"Moe", "Larry", "Curly"} // len(stooges1) == 3

//上面的例子与下面的等效
stooges2 := [3]string{"Moe", "Larry", "Curly"} // len(stooges2) == 3
```

# Go命令行中的通配符

描述包文件的通配符。  
在这个例子中，会单元测试当前目录和所有子目录的所有包：
```go
go test ./...
```

# 参考
```bash
```