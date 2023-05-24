# 背景
POST请求的数据主体放在body中，服务端根据请求头中的Content-Type字段来获取body的编码方式，进而对数据进行解析。
# 基础介绍
Http请求消息体的格式为：request-line（请求行）、header（头部）、blank-line（空格行）和body（消息体）。
Request-line声明了请求地址、方法和版本号，header部分声明请求的附加信息，Blank-line（空格行）用于分隔header和body，body部分是请求的消息体，可以为空。
![](attachments/Pasted%20image%2020230619110703.png)
Body不是必须的，也没有格式要求。通常Get类型的请求，body部分可以为空，参数在header中以key-value格式传输即可；Post类型的请求，通过body传输数据。由于body可以传输任意格式的数据格式，为了让接收方知道怎么解析数据，发送方需要在header中，使用Content-Type字段声明body部分的数据格式。
# Content-Type常见的取值

## application/x-www-form-urlencoded
最常见的 POST 提交数据的方式，原生Form表单，如果不设置 enctype 属性，默认为application/x-www-form-urlencoded 。


Content-Type被指定为 application/x-www-form-urlencoded，表单数据会转换为键值对并按照 key1=val1&key2=val2 的方式进行编码，key 和 val 会进行 URL 转码。大部分服务端语言都对这种方式有很好的支持。

发送如下：
```c
<form name="form" action="test-form" method="post">
   <input type="text" name="key1" value="value1">
   <input type="text" name="key2" value="value2">
   <input type="submit" value="Submit">
</form>
```
点击提交按钮，表单发起的http请求header里自动带着“Content-Type: application/x-www-form-urlencoded”，Body中的数据以“key1=value1&key2=value2”的文本格式传输。
## application/json
Content-Type: application/json 作为响应头比较常见。实际上，现在越来越多的人把它作为请求头，用来告诉服务端消息主体是序列化后的 JSON 字符串，其中一个好处就是JSON 格式支持比键值对复杂得多的结构化数据。

##  multipart/form-data
Form 表单的 enctype 设置为multipart/form-data，它会将表单的数据处理为一条消息，以标签为单元，用分隔符（这就是boundary的作用）分开。由于这种方式将数据有很多部分，它既可以上传键值对，也可以上传文件，甚至多个文件。

# 参考
```c
https://blog.oonne.com/site/blog?id=47
```