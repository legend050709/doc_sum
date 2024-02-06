```table-of-contents
```
# sync.map的值的修改问题
如果我们需要修改一个 sync.Map 中的元素，该怎么办呢？不能像普通的 map 类型那样直接通过下标来对值进行修改。我们来看一下 Go 官方文档对于修改的说法：
```bash
It must not be copied after first use.

To avoid ownership issues, values stored in the Map Should not be modified.
```
文档中指出，sync.Map 中的值不应该被修改。这是因为 map 是一种引用类型，如果我们修改了它，那么可能会影响到其他协程，从而导致竞争条件和数据不一致问题。

## 解决方法
最后，我们需要总结一下：
1. sync.Map 类型的值不能被修改，如果要更新一个值，我们应该通过 Range 方法获取到该值，然后重新写入一个新的值。
2. 在使用 sync.Map 时，一定要注意并发处理的问题，防止数据不一致等问题。

# 参考
```bash
# map：如何实现线程安全的map类型？
https://github.com/CodeFish-xiao/go_concurrent_notes/blob/master/1.%E5%9F%BA%E6%9C%AC%E5%B9%B6%E5%8F%91%E5%8E%9F%E8%AF%AD/1.09%EF%BC%9Amap%EF%BC%9A%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84map%E7%B1%BB%E5%9E%8B%EF%BC%9F/09.00-map%EF%BC%9A%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84map%E7%B1%BB%E5%9E%8B%EF%BC%9F.md
```