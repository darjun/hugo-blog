---
layout:    post
title:    "Go 每日一库之 reflect"
subtitle: 	"每天学习一个 Go 库"
date:		2021-05-27T06:45:00
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/05/27/godailylib/reflect"
categories: [
	"Go"
]
---

## 简介

反射是一种机制，在编译时不知道具体类型的情况下，可以透视结构的组成、更新值。使用反射，可以让我们编写出能统一处理所有类型的代码。甚至是编写这部分代码时还不存在的类型。一个具体的例子就是`fmt.Println()`方法，可以打印出我们自定义的结构类型。

虽然，一般来说都不建议在代码中使用反射。反射影响性能、不易阅读、将编译时就能检查出来的类型问题推迟到运行时以 panic 形式表现出来，这些都是反射的缺点。但是，我认为反射是一定要掌握的，原因如下：

* 很多标准库和第三方库都用到了反射，虽然暴露的接口做了封装，不需要了解反射。但是如果要深入研究这些库，了解实现，阅读源码， 反射是绕不过去的。例如`encoding/json`，`encoding/xml`等；
* 如果有一个需求，编写一个可以处理所有类型的函数或方法，我们就必须会用到反射。因为 Go 的类型数量是无限的，而且可以自定义类型，所以使用类型断言是无法达成目标的。

Go 语言标准库`reflect`提供了反射功能。

## 接口

反射是建立在 Go 的类型系统之上的，并且与接口密切相关。

首先简单介绍一下接口。Go 语言中的接口约定了一组方法集合，任何定义了这组方法的类型（也称为实现了接口）的变量都可以赋值给该接口的变量。

```golang
package main

import "fmt"

type Animal interface {
  Speak()
}

type Cat struct {
}

func (c Cat) Speak() {
  fmt.Println("Meow")
}

type Dog struct {
}

func (d Dog) Speak() {
  fmt.Println("Bark")
}

func main() {
  var a Animal

  a = Cat{}
  a.Speak()

  a = Dog{}
  a.Speak()
}
```

上面代码中，我们定义了一个`Animal`接口，它约定了一个方法`Speak()`。而后定义了两个结构类型`Cat`和`Dog`，都定义了这个方法。这样，我们就可以将`Cat`和`Dog`对象赋值给`Animal`类型的变量了。

接口变量包含两部分：类型和值，即`(type, value)`。类型就是赋值给接口变量的值的类型，值就是赋值给接口变量的值。如果知道接口中存储的变量类型，我们也可以使用类型断言通过接口变量获取具体类型的值：

```golang
type Animal interface {
  Speak()
}

type Cat struct {
  Name string
}

func (c Cat) Speak() {
  fmt.Println("Meow")
}

func main() {
  var a Animal

  a = Cat{Name: "kitty"}
  a.Speak()

  c := a.(Cat)
  fmt.Println(c.Name)
}
```

上面代码中，我们知道接口`a`中保存的是`Cat`对象，直接使用类型断言`a.(Cat)`获取`Cat`对象。**但是，如果类型断言的类型与实际存储的类型不符，会直接 panic。所以实际开发中，通常使用另一种类型断言形式`c, ok := a.(Cat)`。如果类型不符，这种形式不会 panic，而是通过将第二个返回值置为 false 来表明这种情况**。

有时候，一个类型定义了很多方法，而不只是接口约定的方法。通过接口，我们只能调用接口中约定的方法。当然我们也可以将其类型断言为另一个接口，然后调用这个接口约定的方法，前提是原对象实现了这个接口：

```golang
var r io.Reader
r = new(bytes.Buffer)
w = r.(io.Writer)
```

`io.Reader`和`io.Writer`是标准库中使用最为频繁的两个接口：

```golang
// src/io/io.go
type Reader interface {
  Read(p []byte) (n int, err error)
}

type Writer interface {
  Write(p []byte) (n int, err error)
}
```

`bytes.Buffer`同时实现了这两个接口，所以`byte.Buffer`对象可以赋值给`io.Reader`变量`r`，然后`r`可以断言为`io.Writer`，因为接口`io.Reader`中存储的值也实现了`io.Writer`接口。

如果一个接口`A`包含另一个接口`B`的所有方法，那么接口`A`的变量可以直接赋值给`B`的变量，因为`A`中存储的值一定实现了`A`约定的所有方法，那么肯定也实现了`B`。此时，无须类型断言。例如标准库`io`中还定义了一个`io.ReadCloser`接口，此接口变量可以直接赋值给`io.Reader`：

```golang
// src/io/io.go
type ReadCloser interface {
  Reader
  Closer
}
```

**空接口`interface{}`是比较特殊的一个接口，它没有约定任何方法。所有类型值都可以赋值给空接口类型的变量，因为它没有任何方法限制**。

有一点特别重要，**接口变量之间类型断言也好，直接赋值也好，其内部存储的`(type, value)`类型-值对是没有变化的。只是通过不同的接口能调用的方法有所不同而已**。也是由于这个原因，**接口变量中存储的值一定不是接口类型**。

有了这些接口的基础知识，下面我们介绍反射。

## 反射基础

Go 语言中的反射功能由`reflect`包提供。`reflect`包定义了一个接口`reflect.Type`和一个结构体`reflect.Value`，它们定义了大量的方法用于获取类型信息，设置值等。在`reflect`包内部，只有类型描述符实现了`reflect.Type`接口。由于类型描述符是未导出类型，我们只能通过`reflect.TypeOf()`方法获取`reflect.Type`类型的值：

```golang
package main

import (
  "fmt"
  "reflect"
)

type Cat struct {
  Name string
}

func main() {
  var f float64 = 3.5
  t1 := reflect.TypeOf(f)
  fmt.Println(t1.String())

  c := Cat{Name: "kitty"}
  t2 := reflect.TypeOf(c)
  fmt.Println(t2.String())
}
```

输出：

```cmd
float64
main.Cat
```

Go 语言是静态类型的，每个变量在编译期有且只能有一个确定的、已知的类型，即变量的静态类型。静态类型在变量声明的时候就已经确定了，无法修改。一个接口变量，它的静态类型就是该接口类型。虽然在运行时可以将不同类型的值赋值给它，改变的也只是它内部的动态类型和动态值。它的静态类型始终没有改变。

`reflect.TypeOf()`方法就是用来取出接口中的动态类型部分，以`reflect.Type`返回。等等！上面代码好像并没有接口类型啊？

我们看下`reflect.TypeOf()`的定义：

```golang
// src/reflect/type.go
func TypeOf(i interface{}) Type {
  eface := *(*emptyInterface)(unsafe.Pointer(&i))
  return toType(eface.typ)
}
```

它接受一个`interface{}`类型的参数，所以上面的`float64`和`Cat`变量会先转为`interface{}`再传给方法，`reflect.TypeOf()`方法获取的就是这个`interface{}`中的类型部分。

相应地，`reflect.ValueOf()`方法自然就是获取接口中的值部分，返回值为`reflect.Value`类型。在上例基础上添加下面代码：

```golang
v1 := reflect.ValueOf(f)
fmt.Println(v1)
fmt.Println(v1.String())

v2 := reflect.ValueOf(c)
fmt.Println(v2)
fmt.Println(v2.String())
```

运行输出：

```cmd
3.5
<float64 Value>
{kitty}
<main.Cat Value>
```

由于`fmt.Println()`会对`reflect.Value`类型做特殊处理，打印其内部的值，所以上面显示调用了`reflect.Value.String()`方法获取更多信息。

获取类型如此常见，`fmt`提供了格式化符号`%T`输出参数类型：

```golang
fmt.Printf("%T\n", 3) // int
```

Go 语言中类型是无限的，而且可以通过`type`定义新的类型。但是类型的种类是有限的，`reflect`包中定义了所有种类的枚举：

```golang
// src/reflect/type.go
type Kind uint

const (
  Invalid Kind = iota
  Bool
  Int
  Int8
  Int16
  Int32
  Int64
  Uint
  Uint8
  Uint16
  Uint32
  Uint64
  Uintptr
  Float32
  Float64
  Complex64
  Complex128
  Array
  Chan
  Func
  Interface
  Map
  Ptr
  Slice
  String
  Struct
  UnsafePointer
)
```

一共 26 种，我们可以分类如下：

* 基础类型`Bool`、`String`以及各种数值类型（有符号整数`Int/Int8/Int16/Int32/Int64`，无符号整数`Uint/Uint8/Uint16/Uint32/Uint64/Uintptr`，浮点数`Float32/Float64`，复数`Complex64/Complex128`）
* 复合（聚合）类型`Array`和`Struct`
* 引用类型`Chan`、`Func`、`Ptr`、`Slice`和`Map`（值类型和引用类型区分不明显，这里不引战，大家理解意思就行）
* 接口类型`Interface`
* 非法类型`Invalid`，表示它还没有任何值（`reflect.Value`的零值就是`Invalid`类型）

Go 中所有的类型（包括自定义的类型），都是上面这些类型或它们的组合。

例如：

```golang
type MyInt int

func main() {
  var i int
  var j MyInt

  i = int(j) // 必须强转

  ti := reflect.TypeOf(i)
  fmt.Println("type of i:", ti.String())

  tj := reflect.TypeOf(j)
  fmt.Println("type of j:", tj.String())

  fmt.Println("kind of i:", ti.Kind())
  fmt.Println("kind of j:", tj.Kind())
}
```

上面两个变量的静态类型分别为`int`和`MyInt`，是不同的。虽然`MyInt`的底层类型（underlying type）也是`int`。它们之间的赋值必须要强制类型转换。但是它们的种类是一样的，都是`int`。

代码输出如下：

```cmd
type of i: int
type of j: main.MyInt
kind of i: int
kind of j: int
```

## 反射用法

由于反射的内容和 API 非常多，我们结合具体用法来介绍。

### 透视数据组成

透视结构体组成，需要以下方法：

* `reflect.ValueOf()`：获取反射值对象；
* `reflect.Value.NumField()`：从结构体的反射值对象中获取它的字段个数；
* `reflect.Value.Field(i)`：从结构体的反射值对象中获取第`i`个字段的反射值对象；
* `reflect.Kind()`：从反射值对象中获取种类；
* `reflect.Int()/reflect.Uint()/reflect.String()/reflect.Bool()`：这些方法从反射值对象做取出具体类型。

示例：

```golang
type User struct {
  Name    string
  Age     int
  Married bool
}

func inspectStruct(u interface{}) {
  v := reflect.ValueOf(u)
  for i := 0; i < v.NumField(); i++ {
    field := v.Field(i)
    switch field.Kind() {
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
      fmt.Printf("field:%d type:%s value:%d\n", i, field.Type().Name(), field.Int())

    case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
      fmt.Printf("field:%d type:%s value:%d\n", i, field.Type().Name(), field.Uint())

    case reflect.Bool:
      fmt.Printf("field:%d type:%s value:%t\n", i, field.Type().Name(), field.Bool())

    case reflect.String:
      fmt.Printf("field:%d type:%s value:%q\n", i, field.Type().Name(), field.String())

    default:
      fmt.Printf("field:%d unhandled kind:%s\n", i, field.Kind())
    }
  }
}

func main() {
  u := User{
    Name:    "dj",
    Age:     18,
    Married: true,
  }

  inspectStruct(u)
}
```

结合使用`reflect.Value`的`NumField()`和`Field()`方法可以遍历结构体的每个字段。然后针对每个字段的`Kind`做相应的处理。

有些方法只有在原对象是某种特定类型时，才能调用。例如`NumField()`和`Field()`方法只有原对象是结构体时才能调用，否则会`panic`。

识别出具体类型后，可以调用反射值对象的对应类型方法获取具体类型的值，例如上面的`field.Int()/field.Uint()/field.Bool()/field.String()`。但是为了减轻处理的负担，`Int()/Uint()`方法对种类做了合并处理，它们只返回相应的最大范围的类型，`Int()`返回`Int64`类型，`Uint()`返回`Uint64`类型。而`Int()/Uint()`内部会对相应的有符号或无符号种类做处理，转为`Int64/Uint64`返回。下面是`reflect.Value.Int()`方法的实现：

```golang
// src/reflect/value.go
func (v Value) Int() int64 {
  k := v.kind()
  p := v.ptr
  switch k {
  case Int:
    return int64(*(*int)(p))
  case Int8:
    return int64(*(*int8)(p))
  case Int16:
    return int64(*(*int16)(p))
  case Int32:
    return int64(*(*int32)(p))
  case Int64:
    return *(*int64)(p)
  }
  panic(&ValueError{"reflect.Value.Int", v.kind()})
}
```

上面代码，我们只处理了少部分种类。在实际开发中，完善的处理需要破费一番功夫，特别是字段是其他复杂类型，甚至包含循环引用的时候。

另外，我们也可以透视标准库中的结构体，**并且可以透视其中的未导出字段**。使用上面定义的`inspectStruct()`方法：

```golang
inspectStruct(bytes.Buffer{})
```

`bytes.Buffer`的结构如下：

```golang
type Buffer struct {
  buf      []byte
  off      int   
  lastRead readOp
}
```

都是未导出的字段，程序输出：

```cmd
field:0 unhandled kind:slice
field:1 type:int value:0
field:2 type:readOp value:0
```

透视`map`组成，需要以下方法：

* `reflect.Value.MapKeys()`：将每个键的`reflect.Value`对象组成一个切片返回；
* `reflect.Value.MapIndex(k)`：传入键的`reflect.Value`对象，返回值的`reflect.Value`；
* 然后可以对键和值的`reflect.Value`进行和上面一样的处理。

示例：

```golang
func inspectMap(m interface{}) {
  v := reflect.ValueOf(m)
  for _, k := range v.MapKeys() {
    field := v.MapIndex(k)

    fmt.Printf("%v => %v\n", k.Interface(), field.Interface())
  }
}

func main() {
  inspectMap(map[uint32]uint32{
    1: 2,
    3: 4,
  })
}
```

我这里偷懒了，没有针对每个`Kind`去做处理，直接调用键-值`reflect.Value`的`Interface()`方法。该方法以空接口的形式返回内部包含的值。程序输出：

```cmd
1 => 2
3 => 4
```

同样地，`MapKeys()`和`MapIndex(k)`方法只能在原对象是`map`类型时才能调用，否则会`panic`。

透视切片或数组组成，需要以下方法：

* `reflect.Value.Len()`：返回数组或切片的长度；
* `reflect.Value.Index(i)`：返回第`i`个元素的`reflect.Value`值；
* 然后对这个`reflect.Value`判断`Kind()`进行处理。

示例：

```golang
func inspectSliceArray(sa interface{}) {
  v := reflect.ValueOf(sa)

  fmt.Printf("%c", '[')
  for i := 0; i < v.Len(); i++ {
    elem := v.Index(i)
    fmt.Printf("%v ", elem.Interface())
  }
  fmt.Printf("%c\n", ']')
}

func main() {
  inspectSliceArray([]int{1, 2, 3})
  inspectSliceArray([3]int{4, 5, 6})
}
```

程序输出：

```cmd
[1 2 3 ]
[4 5 6 ]
```

同样地`Len()`和`Index(i)`方法只能在原对象是切片，数组或字符串时才能调用，其他类型会`panic`。

透视函数类型，需要以下方法：

* `reflect.Type.NumIn()`：获取函数参数个数；
* `reflect.Type.In(i)`：获取第`i`个参数的`reflect.Type`；
* `reflect.Type.NumOut()`：获取函数返回值个数；
* `reflect.Type.Out(i)`：获取第`i`个返回值的`reflect.Type`。

示例：

```golang
func Add(a, b int) int {
  return a + b
}

func Greeting(name string) string {
  return "hello " + name
}

func inspectFunc(name string, f interface{}) {
  t := reflect.TypeOf(f)
  fmt.Println(name, "input:")
  for i := 0; i < t.NumIn(); i++ {
    t := t.In(i)
    fmt.Print(t.Name())
    fmt.Print(" ")
  }
  fmt.Println()

  fmt.Println("output:")
  for i := 0; i < t.NumOut(); i++ {
    t := t.Out(i)
    fmt.Print(t.Name())
    fmt.Print(" ")
  }
  fmt.Println("\n===========")
}

func main() {
  inspectFunc("Add", Add)
  inspectFunc("Greeting", Greeting)
}
```

同样地，只有在原对象是函数类型的时候才能调用`NumIn()/In()/NumOut()/Out()`这些方法，其他类型会`panic`。

程序输出：

```golang
Add input:
int int
output:
int
===========
Greeting input:
string
output:
string
===========
```

透视结构体中定义的方法，需要以下方法：

* `reflect.Type.NumMethod()`：返回结构体定义的方法个数；
* `reflect.Type.Method(i)`：返回第`i`个方法的`reflect.Method`对象；

示例：

```golang
func inspectMethod(o interface{}) {
  t := reflect.TypeOf(o)

  for i := 0; i < t.NumMethod(); i++ {
    m := t.Method(i)

    fmt.Println(m)
  }
}

type User struct {
  Name    string
  Age     int
}

func (u *User) SetName(n string) {
  u.Name = n
}

func (u *User) SetAge(a int) {
  u.Age = a
}

func main() {
  u := User{
    Name:    "dj",
    Age:     18,
  }
  inspectMethod(&u)
}
```

`reflect.Method`定义如下：

```golang
// src/reflect/type.go
type Method struct {
  Name    string // 方法名
  PkgPath string

  Type  Type  // 方法类型（即函数类型）
  Func  Value // 方法值（以接收器作为第一个参数）
  Index int   // 是结构体中的第几个方法
}
```

事实上，`reflect.Value`也定义了`NumMethod()/Method(i)`这些方法。区别在于：`reflect.Type.Method(i)`返回的是一个`reflect.Method`对象，可以获取方法名、类型、是结构体中的第几个方法等信息。如果要通过这个`reflect.Method`调用方法，必须使用`Func`字段，而且要传入接收器的`reflect.Value`作为第一个参数：

```golang
m.Func.Call(v, ...args)
```

但是`reflect.Value.Method(i)`返回一个`reflect.Value`对象，它总是以调用`Method(i)`方法的`reflect.Value`作为接收器对象，不需要额外传入。而且直接使用`Call()`发起方法调用：

```golang
m.Call(...args)
```

`reflect.Type`和`reflect.Value`有不少同名方法，使用时需要注意甄别。

### 调用函数或方法

调用函数，需要以下方法：

* `reflect.Value.Call()`：使用`reflect.ValueOf()`生成每个参数的反射值对象，然后组成切片传给`Call()`方法。`Call()`方法执行函数调用，返回`[]reflect.Value`。其中每个元素都是原返回值的反射值对象。

示例：

```golang
func Add(a, b int) int {
  return a + b
}

func Greeting(name string) string {
  return "hello " + name
}

func invoke(f interface{}, args ...interface{}) {
  v := reflect.ValueOf(f)

  argsV := make([]reflect.Value, 0, len(args))
  for _, arg := range args {
    argsV = append(argsV, reflect.ValueOf(arg))
  }

  rets := v.Call(argsV)

  fmt.Println("ret:")
  for _, ret := range rets {
    fmt.Println(ret.Interface())
  }
}

func main() {
  invoke(Add, 1, 2)
  invoke(Greeting, "dj")
}
```

我们封装一个`invoke()`方法，以`interface{}`空接口接收函数对象，以`interface{}`可变参数接收函数调用的参数。函数内部首先调用`reflect.ValueOf()`方法获得函数对象的反射值对象。然后依次对每个参数调用`reflect.ValueOf()`，生成参数的反射值对象切片。最后调用函数反射值对象的`Call()`方法，输出返回值。

程序运行结果：

```cmd
ret:
3
ret:
hello dj
```

方法的调用也是类似的：

```golang
type M struct {
  a, b int
  op   rune
}

func (m M) Op() int {
  switch m.op {
  case '+':
    return m.a + m.b

  case '-':
    return m.a - m.b

  case '*':
    return m.a * m.b

  case '/':
    return m.a / m.b

  default:
    panic("invalid op")
  }
}

func main() {
  m1 := M{1, 2, '+'}
  m2 := M{3, 4, '-'}
  m3 := M{5, 6, '*'}
  m4 := M{8, 2, '/'}
  invoke(m1.Op)
  invoke(m2.Op)
  invoke(m3.Op)
  invoke(m4.Op)
}
```

运行结果：

```cmd
ret:
3
ret:
-1
ret:
30
ret:
4
```

以上是在编译期明确知道方法名的情况下发起调用。如果只给一个结构体对象，通过参数指定具体调用哪个方法该怎么做呢？这需要以下方法：

* `reflect.Value.MethodByName(name)`：获取结构体中定义的名为`name`的方法的`reflect.Value`对象，这个方法默认有接收器参数，即调用`MethodByName()`方法的`reflect.Value`。

示例：

```golang
type Math struct {
  a, b int
}

func (m Math) Add() int {
  return m.a + m.b
}

func (m Math) Sub() int {
  return m.a - m.b
}

func (m Math) Mul() int {
  return m.a * m.b
}

func (m Math) Div() int {
  return m.a / m.b
}

func invokeMethod(obj interface{}, name string, args ...interface{}) {
  v := reflect.ValueOf(obj)
  m := v.MethodByName(name)

  argsV := make([]reflect.Value, 0, len(args))
  for _, arg := range args {
    argsV = append(argsV, reflect.ValueOf(arg))
  }

  rets := m.Call(argsV)

  fmt.Println("ret:")
  for _, ret := range rets {
    fmt.Println(ret.Interface())
  }
}

func main() {
  m := Math{a: 10, b: 2}
  invokeMethod(m, "Add")
  invokeMethod(m, "Sub")
  invokeMethod(m, "Mul")
  invokeMethod(m, "Div")
}
```

我们可以在结构体的反射值对象上使用`NumMethod()`和`Method()`遍历它定义的所有方法。

### 实战案例

使用前面介绍的方法，我们很容易实现一个简单的、基于 HTTP 的 RPC 调用。约定格式：路径名`/obj/method/arg1/arg2`调用`obj.method(arg1, arg2)`方法。

首先定义两个结构体，并为它们定义方法，我们约定可导出的方法会注册为 RPC 方法。并且方法必须返回两个值：一个结果，一个错误。

```golang
type StringObject struct{}

func (StringObject) Concat(s1, s2 string) (string, error) {
  return s1 + s2, nil
}

func (StringObject) ToUpper(s string) (string, error) {
  return strings.ToUpper(s), nil
}

func (StringObject) ToLower(s string) (string, error) {
  return strings.ToLower(s), nil
}

type MathObject struct{}

func (MathObject) Add(a, b int) (int, error) {
  return a + b, nil
}

func (MathObject) Sub(a, b int) (int, error) {
  return a - b, nil
}

func (MathObject) Mul(a, b int) (int, error) {
  return a * b, nil
}

func (MathObject) Div(a, b int) (int, error) {
  if b == 0 {
    return 0, errors.New("divided by zero")
  }
  return a / b, nil
}
```

接下来我们定义一个结构表示可以调用的 RPC 方法：

```golang
type RpcMethod struct {
  method reflect.Value
  args   []reflect.Type
}
```

其中`method`是方法的反射值对象，`args`是各个参数的类型。我们定义一个函数从对象中提取可以 RPC 调用的方法：

```golang
var (
  mapObjMethods map[string]map[string]RpcMethod
)

func init() {
  mapObjMethods = make(map[string]map[string]RpcMethod)
}

func registerMethods(objName string, o interface{}) {
  v := reflect.ValueOf(o)

  mapObjMethods[objName] = make(map[string]RpcMethod)
  for i := 0; i < v.NumMethod(); i++ {
    m := v.Method(i)

    if m.Type().NumOut() != 2 {
      // 排除不是两个返回值的
      continue
    }

    if m.Type().Out(1).Name() != "error" {
      // 排除第二个返回值不是 error 的
      continue
    }

    t := v.Type().Method(i)
    methodName := t.Name
    if len(methodName) <= 1 || strings.ToUpper(methodName[0:1]) != methodName[0:1] {
      // 排除非导出方法
      continue
    }

    types := make([]reflect.Type, 0, 1)
    for j := 0; j < m.Type().NumIn(); j++ {
      types = append(types, m.Type().In(j))
    }

    mapObjMethods[objName][methodName] = RpcMethod{
      m, types,
    }
  }
}
```

`registerMethods()`函数使用`reflect.Value.NumMethod()`和`reflect.Method(i)`从对象中遍历方法，排除掉不是两个返回值的、第二个返回值不是 error 的或者非导出的方法。

然后定义一个 http 处理器：

```golang
func handler(w http.ResponseWriter, r *http.Request) {
  parts := strings.Split(r.URL.Path[1:], "/")
  if len(parts) < 2 {
    handleError(w, errors.New("invalid request"))
    return
  }

  m := lookupMethod(parts[0], parts[1])
  if m.method.IsZero() {
    handleError(w, fmt.Errorf("no such method:%s in object:%s", parts[0], parts[1]))
    return
  }

  argSs := parts[2:]
  if len(m.args) != len(argSs) {
    handleError(w, errors.New("inconsistant args num"))
    return
  }

  argVs := make([]reflect.Value, 0, 1)
  for i, t := range m.args {
    switch t.Kind() {
    case reflect.Int:
      value, _ := strconv.Atoi(argSs[i])
      argVs = append(argVs, reflect.ValueOf(value))

    case reflect.String:
      argVs = append(argVs, reflect.ValueOf(argSs[i]))

    default:
      handleError(w, fmt.Errorf("invalid arg type:%s", t.Kind()))
      return
    }
  }

  ret := m.method.Call(argVs)
  err := ret[1].Interface()
  if err != nil {
    handleError(w, err.(error))
    return
  }

  response(w, ret[0].Interface())
}
```

我们将路径分割得到一个切片，第一个元素为对象名（即`math`或`string`），第二个元素为方法名（即`Add/Sub/Mul/Div`等），后面的都是参数。接着，我们查找要调用的方法，根据注册时记录的各个参数的类型将路径中的字符串转换为对应类型。然后调用，检查第二个返回值是否为`nil`可以获知方法调用是否出错。成功调用则返回结果。

最后我们只需要启动一个 http 服务器即可：

```golang
func main() {
  registerMethods("math", MathObject{})
  registerMethods("string", StringObject{})

  mux := http.NewServeMux()
  mux.HandleFunc("/", handler)

  server := &http.Server{
    Addr:    ":8080",
    Handler: mux,
  }

  if err := server.ListenAndServe(); err != nil {
    log.Fatal(err)
  }
}
```

完整代码在 Github 仓库中。运行：

```cmd
$ go run main.go

```

使用 curl 来验证：

```cmd
$ curl localhost:8080/math/Add/1/2
{"data":3}
$ curl localhost:8080/math/Sub/10/2
{"data":8}
$ curl localhost:8080/math/Div/10/2
{"data":5}
$ curl localhost:8080/math/Div/10/0
{"error":"divided by zero"}
$ curl localhost:8080/string/Concat/abc/def
{"data":"abcdef"}
```

当然，这只是一个简单的实现，还有很多错误处理没有考虑，方法参数的类型目前只支持`int`和`string`，感兴趣可以去完善一下。

### 设置值

首先介绍一个概念：可寻址。可寻址是可以通过反射获得其地址的能力。可寻址与指针紧密相关。所有通过`reflect.ValueOf()`得到的`reflect.Value`都不可寻址。因为它们只保存了自身的值，对自身的地址一无所知。例如指针`p *int`保存了另一个`int`数据在内存中的地址，但是它自身的地址无法通过自身获取到，因为在将它传给`reflect.ValueOf()`时，其自身地址信息就丢失了。我们可以通过`reflect.Value.CanAddr()`判断是否可寻址：

```golang
func main() {
  x := 2

  a := reflect.ValueOf(2)
  b := reflect.ValueOf(x)
  c := reflect.ValueOf(&x)
  fmt.Println(a.CanAddr()) // false
  fmt.Println(b.CanAddr()) // false
  fmt.Println(c.CanAddr()) // false
}
```

虽然指针不可寻址，但是我们可以在其反射对象上调用`Elem()`获取它指向的元素的`reflect.Value`。这个`reflect.Value`就可以寻址了，因为是通过`reflect.Value.Elem()`获取的值，可以记录这个获取路径。因而得到的`reflect.Value`中保存了它的地址：

```golang
d := c.Elem()
fmt.Println(d.CanAddr())
```

另外通过切片反射对象的`Index(i)`方法得到的`reflect.Value`也是可寻址的，我们总是可以通过切片得到某个索引的地址。通过结构体的指针获取到的字段也是可寻址的：

```golang
type User struct {
  Name string
  Age  int
}

s := []int{1, 2, 3}
sv := reflect.ValueOf(s)
e := sv.Index(1)
fmt.Println(e.CanAddr()) // true

u := &User{Name: "dj", Age: 18}
uv := reflect.ValueOf(u)
f := uv.Elem().Field(0)
fmt.Println(f.CanAddr()) // true
```

如果一个`reflect.Value`可寻址，我们可以调用其`Addr()`方法返回一个`reflect.Value`，包含一个指向原始数据的指针。然后在这个`reflect.Value`上调用`Interface{}`方法，会返回一个包含这个指针的`interface{}`值。如果我们知道类型，可以使用类型断言将其转为一个普通指针。通过普通指针来更新值：

```golang
func main() {
  x := 2
  d := reflect.ValueOf(&x).Elem()
  px := d.Addr().Interface().(*int)
  *px = 3
  fmt.Println(x) // 3
}
```

这样的更新方法有点麻烦，我们可以直接通过可寻址的`reflect.Value`调用`Set()`方法来更新，不用通过指针：

```golang
d.Set(reflect.ValueOf(4))
fmt.Println(x) // 4
```

如果传入的类型不匹配，会 panic。`reflect.Value`为基本类型提供特殊的`Set`方法：`SetInt`、`SetUint`、`SetFloat`等：

```golang
d.SetInt(5)
fmt.Println(x) // 5
```

**反射可以读取未导出结构字段的值，但是不能更新这些值。一个可寻址的`reflect.Value`会记录它是否是通过遍历一个未导出字段来获得的，如果是则不允许修改。所以在更新前使用`CanAddr()`判断并不保险。`CanSet()`可以正确判断一个值是否可以修改。**

`CanSet()`判断的是**可设置性**，它是比可寻址性更严格的性质。如果一个`reflect.Value`是可设置的，它一定是可寻址的。反之则不然：

```golang
type User struct {
  Name string
  age  int
}

u := &User{Name: "dj", age: 18}
uv := reflect.ValueOf(u)
name := uv.Elem().Field(0)
fmt.Println(name.CanAddr(), name.CanSet()) // true true
age := uv.Elem().Field(1)
fmt.Println(age.CanAddr(), age.CanSet()) // true false

name.SetString("lidajun")
fmt.Println(u) // &{lidajun 18}
// 报错
// age.SetInt(20)
```

### `StructTag`

在定义结构体时，可以为每个字段指定一个标签，我们可以使用反射读取这些标签：

```golang
type User struct {
  Name string `json:"name"`
  Age  int    `json:"age"`
}

func main() {
  u := &User{Name: "dj", Age: 18}
  t := reflect.TypeOf(u).Elem()
  for i := 0; i < t.NumField(); i++ {
    f := t.Field(i)
    fmt.Println(f.Tag)
  }
}
```

标签就是一个普通的字符串，上面程序输出：

```cmd
json:"name"
json:"age"
```

`StructTag`定义在`reflect/type.go`文件中：

```golang
type StructTag string
```

一般惯例是将各个键值对，使用空格分开，键值之间使用`:`。例如：

```golang
`json:"name" xml:"age"`
```

`StructTag`提供`Get()`方法获取键对应的值。

## 总结

本文系统地介绍了 Go 语言中的反射机制，从类型、接口到反射用法。还使用反射实现了一个简单的基于 HTTP 的 RPC 库。反射虽然在平时开发中不建议使用，但是阅读源码，自己编写库的时候需要频繁用到反射知识。熟练掌握反射可以使源码阅读事半功倍。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. Rob Pike, laws of reflection: [https://golang.org/doc/articles/laws_of_reflection.html](https://golang.org/doc/articles/laws_of_reflection.html)
2. Go 程序设计语言，第 12 章：反射
3. reflect 官方文档，[https://pkg.go.dev/reflect](https://pkg.go.dev/reflect)
4. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~