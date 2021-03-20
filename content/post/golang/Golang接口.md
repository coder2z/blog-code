---
title: "Go interface 一个需要注意的地方"
date: 2021-03-20T10:38:00+08:00
Description: "Golang 在使用 if inf == nil 的时候需要注意..."
Tags: ["golang"]
Categories: ["golang"]
DisableComments: false
url: /post/golang/interface/
---

# 序

在前段时间码代码的时候，发现一个有趣的 interface 问题。之前都没注意到，以至于...

![](https://oss.myxy99.cn/images/2021/03/20210320104846.jpg)


# 问题复现

## 例子一

请看这段代码：

```go
func main() {
    var v interface{}
    v = (*int)(nil)
    fmt.Println(v == nil)
}
```

猜测一下会输出啥？

```
false
```

这里是不是就开始很迷惑了，这是为什么呢？这里以及把v强行设置为`nil`了...就很蒙蔽...

![](https://oss.myxy99.cn/images/2021/03/20210320105505.gif)

## 例子二

不着急，我们再看看这个例子

```go
func main() {
    var str *string
    var inf interface{}
    fmt.Println(str == nil)
    fmt.Println(inf == nil)
    inf = str
    fmt.Println(inf == nil)
}
```

猜测一下这又会输出啥？

```
true
true
false
```

![](https://oss.myxy99.cn/images/2021/03/20210320105927.gif)


wtf？？？这就很奇怪，这里声明出来的`str`与`inf`开始判断都是`nil`，但是把变量`str`赋值给`inf`，这里明显`inf`的值还是`nil`，但是判断缺变成了`false`

与上面的例子基本上一样的道理。。。

# 为什么会这样？

![](https://oss.myxy99.cn/images/2021/03/20210320112357.jpg)

这里我去看了一下interface的底层代码，interface 并不是一个指针类型，它是一个结构体。众所周知，interface 和 空interface 是两种不同的数据结构。分别是：

```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type iface struct {
    tab  *itab
    data unsafe.Pointer
}
```

* `eface` 就是空 interface，不包含任何方法的空接口，也称为 empty interface。 
* `iface` 就是正常的 interface，包含方法的接口。

我们上面例子中的`inf`就是一个`eface`。

我们看到上面 interface 的数据结构，就会发现，interface 是把类型与值进行分开存储的。所以我们之前判断是否为 `nil`，**必须需要类型与值同时都为 `nil` 的情况下，interface 的 `nil` 判断才会为 `true`**。

所以例子二中，通过`inf = str`赋值以后，这时候的`inf`虽然值是`nil`，但是类型已经不为空了。这时候判断就是`false`，但是输出是只输出值，就为`nil`

![](https://oss.myxy99.cn/images/2021/03/20210320112721.gif)

# 怎么办呢？

这里我想的只能通过反射来操作：

```go
func IsNilInf(i interface{}) bool {
    v := reflect.ValueOf(i)
    if v.Kind() == reflect.Ptr {
        return v.IsNil()
    }
    return false
}
```

# ...

这个坑，也算是平时没注意的地方，而且还经常碰到...主要是平时使用太多 `if err != nil` 就很容易顺手就踩进去了。

![](https://oss.myxy99.cn/images/2021/03/20210320113255.gif)


