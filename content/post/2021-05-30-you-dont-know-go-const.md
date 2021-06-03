---
layout:    post
title:    "你不知道的 Go 之 const"
subtitle: 	"深入理解背后的知识和误区"
date:		2021-05-30T16:38:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/05/30/youdontknowgo/const"
categories: [
	"Go"
]
---

## 简介

常量可以说在每个代码文件中都存在，使用常量有很多好处：

* 避免魔法字面量，即直接出现在代码中的数字，字符串等。阅读代码的时候无法一眼看出它的含义。另外可以避免使用字面量可能出现的不一致，当它们的值需要修改时，常量只需修改一处，而字面量要修改多处，容易遗漏造成不一致；
* 相对于变量，常量可以执行编译期优化。

Go 语言也提供了常量的语法支持，与其他语言提供的常量基本一致。但是 Go 中的常量有几个有用的特性值得了解一下。

## 常量基础

Go 语言中使用`const`关键字定义常量：

```golang
package main

import "fmt"

const PI float64 = 3.1415926
const MaxAge int = 150
const Greeting string = "hello world"

func main() {
  fmt.Println(PI)
  fmt.Println(MaxAge)
  fmt.Println(Greeting)
}
```

多个常量定义可以合并在一起，如上面的几个常量定义可以写成下面的形式：

```golang
const (
  PI       float64 = 3.1415926
  MaxAge   int     = 150
  Greeting string  = "hello world"
)
```

不过通常建议将相同类型的，相关联的常量定义在一个组里面。

Go 语言中常量有一个很大的限制：只能定义基本类型的常量，即布尔类型（`bool`），整数（无符号`uint/uint8/uint16/uint32/uint64/uintptr`，有符号`int/int8/int16/int32/int64`），浮点数（单精度`float32`，双精度`float64`），或者底层类型是这些基本类型的类型。不能定义切片，数组，指针，结构体等这些类型的常量。例如，`byte`底层类型为`uint8`，`rune`底层类型为`int32`，见 Go 源码`builtin.go`：

```golang
// src/builtin/builtin.go
type byte = uint8
type rune = int32
```

故可以定义类为`byte`或`rune`的常量：

```golang
const b byte = 128
const r rune = 'c'
```

定义其他类型的变量会在编译期报错：

```golang
type User struct {
  Name string
  Age  int
}

const u User = User{} // invalid const type User

var i int = 1
const p *int = &i // invalid const type *int
```

## `iota`

Go 语言的代码中常量定义经常使用`iota`，下面看几个 Go 的源码。

标准库`time`源码：

```golang
// src/time/time.go
type Month int

const (
  January Month = 1 + iota
  February
  March
  April
  May
  June
  July
  August
  September
  October
  November
  December
)

type Weekday int

const (
  Sunday Weekday = iota
  Monday
  Tuesday
  Wednesday
  Thursday
  Friday
  Saturday
)
```

标准库`net/http`源码：

```golang
// src/net/http/server.go
type ConnState int

const (
  StateNew ConnState = iota
  StateActive
  StateIdle
  StateHijacked
  StateClosed
)
```

`iota`是方便我们定义常量的一个机制。简单来说，`iota`独立作用于每一个常量定义组中（单独出现的每个`const`语句都算作一个组），`iota`出现在用于初始化常量值的常量表达式中，**`iota`的值为它在常量组中的第几行（从 0 开始）**。使用`iota`定义的常量下面可以省略类型和初始化表达式，这时会沿用上一个定义的类型和初始化表达式。我们看几组例子：

```golang
const (
  One int = iota + 1
  Two
  Three
  Four
  Five
)
```

这个也是最常使用的方式，`iota`出现在第几行，它的值就是多少。上面常量定义组中，`One`在第 0 行（注意从 0 开始计数），`iota`为 0，所以`One = 0 + 1 = 1`。
下一行`Two`省略了类型和初始化表达式，因此`Two`沿用上面的类型`int`，初始化表达式也是`iota + 1`。但是此时是定义组中的第 1 行，`iota`的值为 1，所以`Two = 1 + 1 = 2`。
再下一行`Three`也省略了类型和初始化表达式，因此`Three`沿用了`Two`进而沿用了`One`的类型`int`，初始化表达式也是`iota + 1`，但是此时是定义的第 2 行，所以`Three = 2 + 1 = 3`。以此类推。

我们可以在非常复杂的初始化表达式中使用`iota`：

```golang
const (
  Mask1 int = 1<<(iota+1) - 1
  Mask2
  Mask3
  Mask4
)
```

按照上面的分析`Mask1~4`依次为 1, 3, 7, 15。

另外还有奇数，偶数：

```golang
const (
  Odd1 = 2*iota + 1
  Odd2
  Odd3
)

const (
  Even1 = 2 * (iota + 1)
  Even2
  Even3
)
```

在一个组中，`iota`不一定出现在第 0 行，**但是它出现在第几行，值就为多少**：

```golang
const (
  A int = 1
  B int = 2
  C int = iota + 1
  D
  E
)
```

上面`iota`出现在第 2 行（从 0 开始），`C`的值为`2 + 1 = 3`。`D`和`E`分别为 4, 5。

**一定要注意`iota`的值等于它出现在组中的第几行，而非它的第几次出现。**

可以通过赋值给空标识符来忽略值：

```golang
const (
  _ int = iota
  A // 1
  B // 2
  C // 3
  D // 4
  E // 5
)
```

说了这么多`iota`的用法，那么为什么要用`iota`呢？换句话说，`iota`有什么优点？我觉得有两点：

* 方便定义，在模式比较固定的时候，我们可以只写出第一个，后面的常量不需要写出类型和初始化表达式了；
* 方便调整顺序，增加和删除常量定义，如果我们定义了一组常量之后，想要调整顺序，使用`iota`的定义，只需要调整位置即可，不需要修改初始化式，因为就没有写。增加和删除也是一样的，如果我们一个个写出了初始化式，删除中间某个，后续的值就必须做调整。

例如，`net/http`中的源码：

```golang
type ConnState int

const (
  StateNew ConnState = iota
  StateActive
  StateIdle
  StateHijacked
  StateClosed
)
```

如果我们需要增加一个常量，表示正在关闭的状态。现在只需要写出新增的状态名：

```golang
type ConnState int

const (
  StateNew ConnState = iota
  StateActive
  StateIdle
  StateHijacked
  StateClosing // 新增的状态
  StateClosed
)
```

如果是显式写出初始化式：

```golang
type ConnState int

const (
  StateNew ConnState = 0
  StateActive ConnState = 1
  StateIdle ConnState = 2
  StateHijacked ConnState = 3
  StateClosed ConnState = 4
)
```

这时新增需要改动后续的值。另外需要键入的字符也多了不少😊：

```golang
const (
  StateNew ConnState = 0
  StateActive ConnState = 1
  StateIdle ConnState = 2
  StateHijacked ConnState = 3
  StateClosing ConnState = 4
  StateClosed ConnState = 5
)
```

## 无类型常量

Go 语言中有一种特殊的常量，即无类型常量。即在定义时，我们不显式指定类型。这种常量可以存储超过常规的类型范围的值：

```golang
package main

import (
  "fmt"
  "math"
  "reflect"
)

const (
  Integer1 = 1000
  Integer2 = math.MaxUint64 + 1
  Float1   = 1.23
  Float2   = 1e100
  Float3   = 1e400
)

func main() {
  fmt.Println("integer1=", Integer1, "type", reflect.TypeOf(Integer1).Name())
  // 编译错误
  // fmt.Println("integer2=", Integer2, "type", reflect.TypeOf(Integer2).Name())
  fmt.Println("integer2/10=", Integer2/10, "type", reflect.TypeOf(Integer2/10).Name())

  fmt.Println("float1=", Float1, "type", reflect.TypeOf(Float1).Name())
  fmt.Println("float2=", Float2, "type", reflect.TypeOf(Float2).Name())
  // 编译错误
  // fmt.Println("float3=", Float3, "type", reflect.TypeOf(Float3).Name())
  fmt.Println("float3/float2=", Float3/Float2, "type", reflect.TypeOf(Float3/Float2).Name())
}
```

虽然无类型常量可以存储超出正常类型范围的值，并且可以相互之间做算术运算，但是它在使用时（赋值给变量，作为参数传递）还是需要转回正常类型。如果值超过正常类型的范围，编译就会报错。每个无类型常量都有一个默认类型，整数的默认类型为`int`，浮点数（有小数点或者使用科学计数法表示的都被当成浮点数）的默认类型为`float64`。所以上面例子中，我们定义`Integer2`为无类型常量，值为`uint64`的最大值 + 1，这是允许的。但是如果我们直接输出`Integer2`的值，就会导致编译报错，因为`Integer2`默认会转为`int`类型，而它存储的值超过了`int`的范围了。另一方面，我们可以用`Integer2`做运算，例如除以 10，得到的值在`int`范围内，可以输出。（我使用的是 64 位机器）

下面的浮点数类型也是类似的，`Float3`超出了`float64`的表示范围，故不能直接输出。但是`Float3/Float2`的结果在`float64`的范围内，可以使用。

上面程序输出：

```golang
integer1= 1000 type int
integer2/10= 1844674407370955161 type int
float1= 1.23 type float64
float2= 1e+100 type float64
float3/float2= 1e+300 type float64
```

由输出也可以看出整数和浮点的默认类型分别为`int`和`float64`。

结合`iota`和无类型常量我们可以定义一组存储单位：

```golang
package main

import "fmt"

const (
  _  = iota
  KB = 1 << (10 * iota)
  MB // 2 ^ 20
  GB // 2 ^ 30
  TB // 2 ^ 40
  PB // 2 ^ 50
  EB // 2 ^ 60
  ZB // 2 ^ 70，1180591620717411303424
  YB // 2 ^ 80
)

func main() {
  fmt.Println(YB / ZB)
  fmt.Println("1180591620717411303424 B = ", 1180591620717411303424/ZB, "ZB")
}
```

`ZB`实际上已经达到 1180591620717411303424，超过了`int`的表示范围了，但是我们仍然可以定义`ZB`和`YB`，还能在使用时对他们进行运算，只要最终要使用的值在正常类型的范围内即可。

## 总结

本文介绍了常量的相关知识，记住两个要点即可：

* `iota`的值等于它出现在常量定义组的第几行（从 0 开始）；
* 无类型常量可以定义超过存储范围的值，但是使用时必须能转回正常类型的范围，否则会报错。

利用无类型常量，我们可以在编译期对大数进行算术运算。

## 参考

1. 《Go 程序设计语言》
2. 你不知道的Go GitHub：[https://github.com/darjun/you-dont-know-go](https://github.com/darjun/you-dont-know-go)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~