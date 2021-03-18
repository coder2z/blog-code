+++
tags = ["golang"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang踩坑笔记"
date = "2020-09-15T09:35:21+08:00"
+++

# golang中的一些陷阱

## int和float64类型不匹配

Go类型系统不允许在整数和浮点变量之间进行任何数学运算。

**比如**
```go
package main

import "fmt"

func main() {
	var x, y = 13, 3.5
	fmt.Println(x / y)
}

```

上面的程序将显示错误

```shell
invalid operation: x / y (mismatched types int and float64)
```

正确的写法应该是：

***使用float64将x的类型转换为float64***

```go
package main

import "fmt"

func main() {
	var x, y = 13, 3.5
	fmt.Println(float64(x) / y)
}

```

那我们看看下面这个代码：

```go
package main

import "fmt"

func main() {
	var x = 13 / 3.5
	fmt.Println(x)
}

```

按理说这里的 13 与 3.5 类型也是不一样的，为什么就没有报错呢？

![](https://gitee.com/coder2m/pic/raw/master/img/20200915113134.png)

## 计算

数值运算的操作数必须具有相同的类型，除非该运算涉及移位或未类型化的常量。

我们看看下面这段代码：
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	var timeout = 3
	fmt.Println(timeout)
	fmt.Println(timeout * time.Millisecond)
}

```

这段代码会报错：
```
invalid operation: timeout * time.Millisecond (mismatched types int and time.Duration)
```

那么下面这段代码呢？
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	const timeout = 10
	fmt.Println(timeout)
	fmt.Println(timeout * time.Millisecond)
}

```

毫秒的基础类型是int64，编译器知道如何将其转换为int64。除非明确声明类型，否则在使用它们之前，不对它们进行转换处理。在此示例中，timeout是未类型化的常量。其类型隐式转换为time.Millisecond。

该程序将打印输出

```
10
10ms
```

那么上面的报错代码就可以修改为：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	var timeout time.Duration
	timeout = 10
	fmt.Println(timeout * time.Millisecond)
}

```

## 字符串字节长度与字符长度

看看下面这个代码：

```go
package main

import "fmt"

func main() {
	data := "♥Go"
	fmt.Println(len(data))
}

```

该程序将打印输出：

```
5
```

在Go Strings中，字符串是UTF-8编码的。在这里，字符♥占用3个字节，因此的字符串字节总长度为5。

如果我们需要获取到字符长度，可以这样：

```go
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	data := "♥Go"
	fmt.Println(utf8.RuneCountInString(data))
}

```

如果要获取字符串中的符文数，则可以使用 unicode/utf8 包。该 RuneCountInString 方法将一个字符串返回符文的数量。

以上程序将打印:

```
3
```

## 浮点乘法

浮点算术被许多人视为深奥的话题。

我们看看下面这段代码的输出：
```go
package main

import (
	"fmt"
)

func main() {
	 var m = 1.39
	 fmt.Println(m * m)
	 
	 const n = 1.39
	 fmt.Println(n * n)
}

```

以上程序将打印:

```
1.9320999999999997
1.9321
```

## 字符串类型转换

查看下面的代码输出的是什么：
```go
package main

import "fmt"

func main() {
	 i := 105
	 s := string(i)
	 fmt.Println(s)	 
}

```
以上程序将打印:

```
i
```

使用string（）把int进行转换的时候会把被转换的数据当作字符。所以这里输出的是```i```

我们只需要使用 strconv 或者 fmt 进行转换就行：

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	i := 105
	s := strconv.Itoa(i)
	fmt.Println(s)
	
	s = fmt.Sprintf("%d", i)
	fmt.Println(s)
}

```