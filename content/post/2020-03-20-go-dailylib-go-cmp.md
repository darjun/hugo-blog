---
layout:    post
title:    "Go 每日一库之 go-cmp"
subtitle: 	"每天学习一个 Go 库"
date:		2020-03-20T20:49:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/03/20/godailylib/go-cmp"
categories: [
	"Go"
]
---

## 简介

我们时常有比较两个值是否相等的需求，最直接的方式就是使用`==`操作符，其实`==`的细节远比你想象的多，我在[深入理解 Go 之`==`](https://darjun.github.io/2019/08/20/golang-equal/)中有详细介绍，有兴趣去看看。但是直接用`==`，一个最明显的弊端就是对于指针，只有两个指针指向同一个对象时，它们才相等，不能进行递归比较。为此，`reflect`包提供了一个`DeepEqual`，它可以进行递归比较。但是相对的，`reflect.DeepEqual`不够灵活，无法提供选项实现我们想要的行为，例如允许浮点数误差。所以今天的主角[`go-cmp`](https://github.com/google/go-cmp)登场了。`go-cmp`是 Google 开源的比较库，它提供了丰富的选项。**最初定位是用在测试中。**

感谢[thinkgos](https://github.com/thinkgos)的推荐！

## 快速使用

先安装：

```golang
$ go get github.com/com/google/go-cmp/cmp
```

后使用：

```golang
package main

import (
  "fmt"

  "github.com/google/go-cmp/cmp"
)

type Contact struct {
  Phone string
  Email string
}

type User struct {
  Name    string
  Age     int
  Contact *Contact
}

func main() {
  u1 := User{Name: "dj", Age: 18}
  u2 := User{Name: "dj", Age: 18}

  fmt.Println("u1 == u2?", u1 == u2)
  fmt.Println("u1 equals u2?", cmp.Equal(u1, u2))

  c1 := &Contact{Phone: "123456789", Email: "dj@example.com"}
  c2 := &Contact{Phone: "123456789", Email: "dj@example.com"}

  u1.Contact = c1
  u2.Contact = c1
  fmt.Println("u1 == u2 with same pointer?", u1 == u2)
  fmt.Println("u1 equals u2 with same pointer?", cmp.Equal(u1, u2))

  u2.Contact = c2
  fmt.Println("u1 == u2 with different pointer?", u1 == u2)
  fmt.Println("u1 equals u2 with different pointer?", cmp.Equal(u1, u2))
}
```

上面的例子中，我们将`==`与`cmp.Equal`放在一起做个比较：

* 在指针类型的字段`Contact`未设置时，`u1 == u2`和`cmp.Equal(u1, u2)`都返回`true`；
* 两个结构的`Contact`字段都指向同一个对象时，`u1 == u2`和`cmp.Equal(u1, u2)`都返回`true`；
* 两个结构的`Contact`字段指向不同的对象时，尽管这两个对象包含相同的内容，`u1 == u2`也返回了`false`。而`cmp.Equal(u1, u2)`可以比较指针指向的内容，从而返回`true`。

以下是运行结果：

```golang
u1 == u2? true
u1 equals u2? true
u1 == u2 with same pointer? true
u1 equals u2 with same pointer? true
u1 == u2 with different pointer? false
u1 equals u2 with different pointer? true
```

## 高级选项

### 未导出字段

默认情况下，`cmp.Equal()`函数不会比较未导出字段（即字段名首字母小写的字段）。遇到未导出字段，`cmp.Equal()`直接`panic`。这一点与`reflect.DeepEqual()`有所不同，后者也会比较未导出的字段。

我们可以使用`cmdopts.IgnoreUnexported`选项忽略未导出字段，也可以使用`cmdopts.AllowUnexported`选项指定某些类型的未导出字段需要比较。

```golang
package main

import (
  "fmt"

  "github.com/google/go-cmp/cmp"
)

type Contact struct {
  Phone string
  Email string
}

type User struct {
  Name    string
  Age     int
  contact *Contact
}

func main() {
  c1 := &Contact{Phone: "123456789", Email: "dj@example.com"}
  c2 := &Contact{Phone: "123456789", Email: "dj@example.com"}

  u1 := User{"dj", 18, c1}
  u2 := User{"dj", 18, c2}

  fmt.Println("u1 equals u2?", cmp.Equal(u1, u2))
}
```

运行上面的代码会`panic`，因为`cmd.Equal()`比较的类型中有未导出字段`contact`。我们先使用`cmdopts.IngoreUnexported`忽略未导出字段：

```golang
fmt.Println("u1 equals u2?", cmp.Equal(u1, u2, cmpopts.IgnoreUnexported(User{})))
```

我们在`cmp.Equal()`的调用中添加了选项`cmpopts.IgnoreUnexported`，选项参数传入`User{}`，**表示忽略`User`的直接未导出字段。导出字段中的未导出字段是不会被忽略的，除非显示指定该类型。**如果我们将`User`稍作修改：

```golang
type Address struct {
  Province string
  city     string
}

type User struct {
  Name    string
  Age     int
  Address Address
}

func main() {
  u1 := User{"dj", 18, Address{}}
  u2 := User{"dj", 18, Address{}}

  fmt.Println("u1 equals u2?", cmp.Equal(u1, u2, cmpopts.IgnoreUnexported(User{})))
}
```

注意，`city`字段未导出，这种情况下，使用`cmpopts.IngoreUnexported(User{})`还是会`panic`，因为`city`是`Address`中的未导出字段，而非`User`的直接字段。

我们也可以使用`cmdopts.AllowUnexported(User{})`表示需要比较`User`的未导出字段：

```golang
fmt.Println("u1 equals u2?", cmp.Equal(u1, u2, cmp.AllowUnexported(User{})))
```

### 浮点数比较

我们知道，计算机中浮点数的表示是不精确的，如果涉及到运算，可能会产生误差累计。此外，还有一个特殊的浮点数`NaN`（Not a Number），**它与任何浮点数都不等，包括它自己**。这样，有时候会出现一些反直觉的结果：

```golang
package main

import (
  "fmt"
  "math"

  "github.com/google/go-cmp/cmp"
)

type FloatPair struct {
  X float64
  Y float64
}

func main() {
  p1 := FloatPair{X: math.NaN()}
  p2 := FloatPair{X: math.NaN()}
  fmt.Println("p1 equals p2?", cmp.Equal(p1, p2))

  f1 := 0.1
  f2 := 0.2
  f3 := 0.3
  p3 := FloatPair{X: f1 + f2}
  p4 := FloatPair{X: f3}
  fmt.Println("p3 equals p4?", cmp.Equal(p3, p4))

  p5 := FloatPair{X: 0.1 + 0.2}
  p6 := FloatPair{X: 0.3}
  fmt.Println("p5 equals p6?", cmp.Equal(p5, p6))
}
```

运行程序，输出：

```golang
p1 equals p2? false
p3 equals p4? false
p5 equals p6? true
```

是不是很反直觉？`NaN`不等于`NaN`，`0.1 + 0.2`竟然不等于`0.3`！前者是由于标准的规定，后者是浮点数的表示不精确导致的计算误差。

奇怪的是第三组表示，为什么直接用字面量运算就不会导致误差呢？实际上，在 Go 语言中这些字面量的运算直接是在编译器完成的，可以做到精确。如果先赋值给浮点类型的变量，就像第 2 组所示，受限于变量的存储空间，就会存在误差。

关于这一点，我这里再顺带介绍一个知识点。我们都知道使用`const`定义常量时可以不指定类型，这种常量被称为无类型的常量，它的值可以超出正常数值的表示范围，可以相互进行的运算。**只是不能赋值给超过其类型表示范围的普通变量**：

```golang
package main

import "fmt"

const (
  _  = 1 << (10 * iota)
  KB // 1024
  MB // 1048576
  GB // 1073741824
  TB // ‭1099511627776‬
  PB // ‭1125899906842624‬
  EB // ‭1152921504606846976‬
  ZB // ‭1180591620717411303424‬
  YB // ‭1208925819614629174706176‬
)

func main() {
  // constant ‭1180591620717411303424‬ overflows int
  // fmt.Println(ZB)

  // constant 1208925819614629174706176 overflows uint64
  // var mem uint64 = YB

  fmt.Println(YB / ZB)
}
```

后面`ZB`和`YB`都已经超出了`uint64`的表示范围。直接使用时，如`fmt.Println(ZB)`编译器会自动将其转为`int`类型，但是它的值超出了`int`的表示范围，所以编译报错。赋值时也是如此。

`go-cmp`提供比较浮点数的选项，我们希望两个`NaN`的比较返回`true`，两个浮点数相差不超过一定范围就认为它们相等：

* `cmpopts.EquateNaNs()`：两个`NaN`比较，返回`true`；
* `cmpopts.EquateApprox(fraction, margin)`：这个选项有两个参数，第二个参数比较好理解，如果两个浮点数的差的绝对值小于`margin`则认为它们相等。第一个参数的含义是取两个数绝对值的较小者，乘以`fraction`，如果两个数的差的绝对值小于这个数即`|x-y| ≤ max(fraction*min(|x|, |y|), margin)`，则认为它们相等。如果`fraction`和`margin`同时设置，**只需要满足一个就行了**。

例如：
```golang
type FloatPair struct {
  X float64
  Y float64
}

func main() {
  p1 := FloatPair{X: math.NaN()}
  p2 := FloatPair{X: math.NaN()}
  fmt.Println("p1 equals p2?", cmp.Equal(p1, p2, cmpopts.EquateNaNs()))

  f1 := 0.1
  f2 := 0.2
  f3 := 0.3
  p3 := FloatPair{X: f1 + f2}
  p4 := FloatPair{X: f3}
  fmt.Println("p3 equals p4?", cmp.Equal(p3, p4, cmpopts.EquateApprox(0.1, 0.001)))
}
```

运行输出：

```golang
p1 equals p2? true
p3 equals p4? true
```

### Nil

默认情况下，如果一个切片变量值为`nil`，另一个是使用`make`创建的长度为 0 的切片，那么`go-cmp`认为它们是不等的。同样的，一个`map`变量值为`nil`，另一个是使用`make`创建的长度为 0 的`map`，那么`go-cmp`也认为它们不等。我们可以指定`cmpopts.EquateEmpty`选项，让`go-cmp`认为它们相等：

```golang
func main() {
  var s1 []int
  var s2 = make([]int, 0)

  var m1 map[int]int
  var m2 = make(map[int]int)

  fmt.Println("s1 equals s2?", cmp.Equal(s1, s2))
  fmt.Println("m1 equals m2?", cmp.Equal(m1, m2))

  fmt.Println("s1 equals s2 with option?", cmp.Equal(s1, s2, cmpopts.EquateEmpty()))
  fmt.Println("m1 equals m2 with option?", cmp.Equal(m1, m2, cmpopts.EquateEmpty()))
}
```

### 切片

默认情况下，两个切片只有当长度相同，且对应位置上的元素都相等时，`go-cmp`才认为它们相等。如果，我们想要实现无序切片的比较（即只要两个切片包含相同的值就认为它们相等），可以使用`cmpopts.SortedSlice`选项先对切片进行排序，然后再进行比较：

```golang
func main() {
  s1 := []int{1, 2, 3, 4}
  s2 := []int{4, 3, 2, 1}
  fmt.Println("s1 equals s2?", cmp.Equal(s1, s2))
  fmt.Println("s1 equals s2 with option?", cmp.Equal(s1, s2, cmpopts.SortSlices(func(i, j int) bool { return i < j })))

  m1 := map[int]int{1: 10, 2: 20, 3: 30}
  m2 := map[int]int{1: 10, 2: 20, 3: 30}
  fmt.Println("m1 equals m2?", cmp.Equal(m1, m2))
  fmt.Println("m1 equals m2 with option?", cmp.Equal(m1, m2, cmpopts.SortMaps(func(i, j int) bool { return i < j })))
}
```

对于`map`来说，由于本身就是无序的，所以`map`比较差不多是下面这种形式。没有上面的顺序问题：

```golang
func compareMap(m1, m2 map[int]int) bool {
  if len(m1) != len(m2) {
    return false
  }

  for k, v := range m1 {
    if v != m2[k] {
      return false
    }
  }

  return true
}
```

`cmpopts.SortMaps`会将`map[K]V`类型按照键排序，生成一个`[]struct{K, V}`的切片，然后逐个比较。

`SortSlices`和`SortMaps`都需要提供一个比较函数`less`，函数必须是`func(T, T) bool`这种形式，切片的元素类型必须可以赋值给`T`类型，`map`的键也必须可以赋值给`T`类型。

## 自定义`Equal`方法

对于有些类型来说，`go-cmp`内置的比较结果不符合我们的要求，这时我们可以自定义`Equal`方法来比较该类型。例如我们想要表示`IP`地址的字符串比较时`127.0.0.1`与`localhost`相等：

```golang
package main

type NetAddr struct {
  IP   string
  Port int
}

func (a NetAddr) Equal(b NetAddr) bool {
  if a.Port != b.Port {
    return false
  }

  if a.IP != b.IP {
    if a.IP == "127.0.0.1" && b.IP == "localhost" {
      return true
    }

    if a.IP == "localhost" && b.IP == "127.0.0.1" {
      return true
    }

    return false
  }

  return true
}

func main() {
	a1 := NetAddr{"127.0.0.1", 5000}
	a2 := NetAddr{"localhost", 5000}
	a3 := NetAddr{"192.168.1.1", 5000}

	fmt.Println("a1 equals a2?", cmp.Equal(a1, a2))
	fmt.Println("a1 equals a3?", cmp.Equal(a1, a3))
}
```

很简单，只需要给想要自定义比较操作的类型提供一个`Equal()`方法即可，方法接受该类型的参数，返回一个`bool`表示是否相等。如果我们将上面的`Equal()`方法注释掉，那么比较输出都是`false`。

## 自定义比较器

如果`go-cmp`默认的行为无法满足我们的需求，我们可以针对某些类型自定义比较器。我们使用`cmp.Comparer()`传入比较函数，比较函数必须是`func (T, T) bool`这种形式。所有能转为`T`类型的值，都会调用该函数进行比较。所以如果`T`是接口类型，那么可能传给比较函数的参数的实际类型并不相同，只是它们都实现了`T`接口。我们使用`Comparer()`重构一下上面的程序：

```golang
type NetAddr struct {
  IP   string
  Port int
}

func compareNetAddr(a, b NetAddr) bool {
  if a.Port != b.Port {
    return false
  }

  if a.IP != b.IP {
    if a.IP == "127.0.0.1" && b.IP == "localhost" {
      return true
    }

    if a.IP == "localhost" && b.IP == "127.0.0.1" {
      return true
    }

    return false
  }

  return true
}

func main() {
  a1 := NetAddr{"127.0.0.1", 5000}
  a2 := NetAddr{"localhost", 5000}

  fmt.Println("a1 equals a2?", cmp.Equal(a1, a2))
  fmt.Println("a1 equals a2 with comparer?", cmp.Equal(a1, a2, cmp.Comparer(compareNetAddr)))
}
```

这种方式与上面介绍的自定义`Equal()`方法有些类似，但更灵活。有时，我们要自定义比较操作的类型定义在第三方包中，这样就无法给它定义`Equal`方法。这时，我们就可以采用自定义`Comparer`的方式。

## `Exporter`

从前面的介绍我们知道默认情况下，未导出字段会导致`cmp.Equal()`直接`panic`。前面也介绍过两种方式处理未导出字段，这里再介绍一种方式——`cmp.Exporter`。通过传入一个函数`func (t reflec.Type) bool`，返回传入的类型是否比较其未导出字段。例如，下面代码中，我们指定需要比较类型`User`的未导出字段：

```golang
type Contact struct {
  Phone string
  Email string
}

type User struct {
  Name    string
  Age     int
  contact Contact
}

func allowUnExportedInType(t reflect.Type) bool {
  if t.Name() == "User" {
    return true
  }

  return false
}

func main() {
  c1 := Contact{Phone: "123456789", Email: "dj@example.com"}
  c2 := Contact{Phone: "123456789", Email: "dj@example.com"}

  u1 := User{"dj", 18, c1}
  u2 := User{"dj", 18, c2}

  fmt.Println("u1 equals u2?", cmp.Equal(u1, u2, cmp.Exporter(allowType)))
}
```

`cmp.Exporter`的使用不多，且可以通过`AllowUnexported`选项来实现。

## 转换器

转换器可以将特定类型的值转为另一种类型的值。转换器有很多用法，下面介绍两种。

### 忽略字段

如果我们想忽略结构中的某些字段，我们可以定义转换，返回一个不设置这些字段的对象：

```golang
type User struct {
  Name string
  Age  int
}

func omitAge(u User) string {
  return u.Name
}

type User2 struct {
  Name    string
  Age     int
  Email   string
  Address string
}

func omitAge2(u User2) User2 {
  return User2{u.Name, 0, u.Email, u.Address}
}

func main() {
  u1 := User{Name: "dj", Age: 18}
  u2 := User{Name: "dj", Age: 28}

  fmt.Println("u1 equals u2?", cmp.Equal(u1, u2, cmp.Transformer("omitAge", omitAge)))

  u3 := User2{Name: "dj", Age: 18, Email: "dj@example.com"}
  u4 := User2{Name: "dj", Age: 28, Email: "dj@example.com"}

  fmt.Println("u3 equals u4?", cmp.Equal(u3, u4, cmp.Transformer("omitAge", omitAge2)))
}
```

如果一个类型，我们只关心一个字段，忽略其它字段，那么直接返回这个字段就行了，如上面的`omitAge`。如果该类型有多个字段，我们只忽略很少的字段，我们要返回一个同样的类型，不设置忽略的字段即可，如上面的`omitAge2`。

### 转换值

上面我们介绍了如何使用自定义`Equal()`方法和`Comparer`比较器的方式来实现 IP 地址的比较。实际上转换器也可以实现同样的效果，我们可以将`localhost`转换为`127.0.0.1`：

```golang
type NetAddr struct {
  IP   string
  Port int
}

func transformLocalhost(a NetAddr) NetAddr {
  if a.IP == "localhost" {
    return NetAddr{IP: "127.0.0.1", Port: a.Port}
  }

  return a
}

func main() {
  a1 := NetAddr{"127.0.0.1", 5000}
  a2 := NetAddr{"localhost", 5000}

  fmt.Println("a1 equals a2?", cmp.Equal(a1, a2, cmp.Transformer("localhost", transformLocalhost)))
}
```

遇到`IP`为`localhost`的对象，将其转换为`IP`为`127.0.0.1`的对象。

## `Diff`

除了能比较两个值是否相等，`go-cmp`还能汇总两个值的不同之处，方便我们查看。上面介绍的选项都可以用在`Diff`中：

```golang
type Contact struct {
  Phone string
  Email string
}

type User struct {
  Name    string
  Age     int
  Contact *Contact
}

func main() {
  c1 := &Contact{Phone: "123456789", Email: "dj@example.com"}
  c2 := &Contact{Phone: "123456879", Email: "dj2@example.com"}
  u1 := User{Name: "dj", Age: 18, Contact: c1}
  u2 := User{Name: "dj2", Age: 18, Contact: c2}

  fmt.Println(cmp.Diff(u1, u2))
}
```

我们着重介绍一下输出的格式：

```golang
  main.User{
-  Name: "dj",
+  Name: "dj2",
   Age:  18,
   Contact: &main.Contact{
-    Phone: "123456789",
+    Phone: "123456879",
-    Email: "dj@example.com",
+    Email: "dj2@example.com",
   },
  }
```

相信使用过 SVN 或对 Linux 的`diff`命令熟悉的童鞋对上面的格式应该不会陌生。我们可以这样认为，第一个对象为原来的版本，第二个对象为新的版本。这样上面的输出我们可以想象成如何将对象从原来的版本变为新版本。没有前缀的行不需要改变，前缀为`-`的行表示新版本删除了这一行，前缀`+`表示新版本增加了这一行。

## 总结

`go-cmp`库大大地方便两个值的比较操作。源码中大量使用我们之前介绍过的**选项模式**，提供给使用者简洁、一致的接口。这种设计思想也值得我们学习、借鉴。本文介绍了这是`go-cmp`的一部分内容，还有一些特性如**过滤器**感兴趣可自行探索。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. go-cmp GitHub：[https://github.com/google/go-cmp](https://github.com/google/go-cmp)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)