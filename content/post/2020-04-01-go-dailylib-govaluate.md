---
layout:    post
title:    "Go 每日一库之 govaluate"
subtitle: 	"每天学习一个 Go 库"
date:		2020-04-01T14:06:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/04/01/godailylib/govaluate"
categories: [
	"Go"
]
---

## 简介

今天我们介绍一个比较好玩的库[`govaluate`](https://github.com/Knetic/govaluate)。`govaluate`与 JavaScript 中的`eval`功能类似，用于计算任意表达式的值。此类功能函数在 JavaScript/Python 等动态语言中比较常见。`govaluate`让 Go 这个编译型语言也有了这个能力！

## 快速使用

先安装：

```golang
$ go get github.com/Knetic/govaluate
```

后使用：

```golang
package main

import (
  "fmt"
  "log"

  "github.com/Knetic/govaluate"
)

func main() {
  expr, err := govaluate.NewEvaluableExpression("10 > 0")
  if err != nil {
    log.Fatal("syntax error:", err)
  }

  result, err := expr.Evaluate(nil)
  if err != nil {
    log.Fatal("evaluate error:", err)
  }

  fmt.Println(result)
}
```

使用`govaluate`计算表达式只需要两步：

* 调用`NewEvaluableExpression()`将表达式转为一个**表达式对象**；
* 调用表达式对象的`Evaluate`方法，传入参数，返回表达式的值。

上面演示了一个很简单的例子，我们使用`govaluate`计算`10 > 0`的值，该表达式不需要参数，故传给`Evaluate()`方法`nil`值。当然，这个例子并不实用，显然我们直接在代码中计算`10 > 0`更简单。但问题是，有些时候我们并不知道需要计算的表达式的所有信息，甚至我们都不知道表达式的结构。这时`govaluate`的作用就体现出来了。

## 参数

`govaluate`支持在表达式中使用参数，调用表达式对象的`Evaluate()`方法时通过`map[string]interface{}`类型将参数传入计算。其中`map`的键为参数名，值为参数值。例如：

```golang
func main() {
  expr, _ := govaluate.NewEvaluableExpression("foo > 0")
  parameters := make(map[string]interface{})
  parameters["foo"] = -1
  result, _ := expr.Evaluate(parameters)
  fmt.Println(result)

  expr, _ = govaluate.NewEvaluableExpression("(requests_made * requests_succeeded / 100) >= 90")
  parameters = make(map[string]interface{})
  parameters["requests_made"] = 100
  parameters["requests_succeeded"] = 80
  result, _ = expr.Evaluate(parameters)
  fmt.Println(result)

  expr, _ = govaluate.NewEvaluableExpression("(mem_used / total_mem) * 100")
  parameters = make(map[string]interface{})
  parameters["total_mem"] = 1024
  parameters["mem_used"] = 512
  result, _ = expr.Evaluate(parameters)
  fmt.Println(result)
}
```

第一个表达式中，我们想要计算`foo > 0`的结果，在传入参数中将`foo`设置为 -1，最终输出`false`。

第二个表达式中，我们想要计算`(requests_made * requests_succeeded / 100) >= 90`的值，在参数中设置`requests_made`为 100，`requests_succeeded`为 80，结果为`true`。

上面两个表达式都返回`bool`结果，第三个表达式返回一个浮点数。`(mem_used / total_mem) * 100`根据传入的总内存`total_mem`和当前使用内存`mem_used`，返回内存占用百分比，结果为 50。

## 命名

使用`govaluate`与直接编写 Go 代码不同，在 Go 代码中标识符中不能出现`-`、`+`、`$`等符号。`govaluate`可以通过转义使用这些符号。有两种转义方式：

* 将名称用`[`和`]`包裹起来，例如`[response-time]`；
* 使用`\`将紧接着**下一个**的字符转义。

例如：

```golang
func main() {
  expr, _ := govaluate.NewEvaluableExpression("[response-time] < 100")
  parameters := make(map[string]interface{})
  parameters["response-time"] = 80
  result, _ := expr.Evaluate(parameters)
  fmt.Println(result)

  expr, _ = govaluate.NewEvaluableExpression("response\\-time < 100")
  parameters = make(map[string]interface{})
  parameters["response-time"] = 80
  result, _ = expr.Evaluate(parameters)
  fmt.Println(result)
}
```

注意一点，因为在字符串中`\`本身就是需要转义的，所以在第二个表达式中要使用`\\`。或者可以使用

```golang
`response\-time` < 100
```

## 一次“编译”多次运行

使用带参数的表达式，我们可以实现一个表达式的一次“编译”，多次运行。只需要使用编译返回的表达式对象即可，可多次调用其`Evaluate()`方法：

```golang
func main() {
  expr, _ := govaluate.NewEvaluableExpression("a + b")
  parameters := make(map[string]interface{})
  parameters["a"] = 1
  parameters["b"] = 2
  result, _ := expr.Evaluate(parameters)
  fmt.Println(result)

  parameters = make(map[string]interface{})
  parameters["a"] = 10
  parameters["b"] = 20
  result, _ = expr.Evaluate(parameters)
  fmt.Println(result)
}
```

第一次运行，传入参数`a = 1, b = 2`得到结果 3；第二次运行，传入参数`a = 10, b = 20`得到结果 30。

## 函数

如果仅仅能进行常规的算数和逻辑运算，`govaluate`的功能会大打折扣。`govaluate`提供了自定义函数的功能。所有自定义函数需要先定义好，存入一个`map[string]govaluate.ExpressionFunction`变量中，然后调用`govaluate.NewEvaluableExpressionWithFunctions()`生成表达式，此表达式中就可以使用这些函数了。自定义函数类型为`func (args ...interface{}) (interface{}, error)`，如果函数返回错误，则这个表达式求值返回错误。

```golang
func main() {
  functions := map[string]govaluate.ExpressionFunction{
    "strlen": func(args ...interface{}) (interface{}, error) {
      length := len(args[0].(string))
      return length, nil
    },
  }

  exprString := "strlen('teststring')"
  expr, _ := govaluate.NewEvaluableExpressionWithFunctions(exprString, functions)
  result, _ := expr.Evaluate(nil)
  fmt.Println(result)
}
```

上面例子中，我们定义一个函数`strlen`计算第一个参数的字符串长度。表达式`strlen('teststring')`调用`strlen`函数返回字符串`teststring`的长度。

函数可以接受任意数量的参数，而且可以处理嵌套函数调用的问题。所以可以写出类似下面这种复杂的表达式：

```golang
sqrt(x1 ** y1, x2 ** y2)

max(someValue, abs(anotherValue), 10 * lastValue)
```

## 访问器

在 Go 语言中，访问器（`Accessors`）就是通过`.`操作访问结构中的字段。如果传入的参数中有结构体类型，`govaluate`也支持使用`.`访问其内部字段或调用它们的方法：

```golang
type User struct {
  FirstName string
  LastName  string
  Age       int
}

func (u User) Fullname() string {
  return u.FirstName + " " + u.LastName
}

func main() {
  u := User{FirstName: "li", LastName: "dajun", Age: 18}
  parameters := make(map[string]interface{})
  parameters["u"] = u

  expr, _ := govaluate.NewEvaluableExpression("u.Fullname()")
  result, _ := expr.Evaluate(parameters)
  fmt.Println("user", result)

  expr, _ = govaluate.NewEvaluableExpression("u.Age > 18")
  result, _ = expr.Evaluate(parameters)
  fmt.Println("age > 18?", result)
}
```

在上面代码中，我们定义了一个`User`结构，并为它编写了一个`Fullname()`方法。第一个表达式中，我们调用`u.Fullname()`返回全名，第二个表达式比较年龄是否大于 18。

需要注意的一点是，我们不能使用`foo.SomeMap['key']`的方式访问`map`的值。由于访问器涉及到很多反射，所以它一般比直接使用参数慢 4 倍左右。如果能使用参数的形式，尽量使用参数。在上面的例子中，我们可以直接调用`u.Fullname()`，将结果作为参数传给表达式求值。涉及到复杂的计算可以通过自定义函数来解决。我们还可以实现`govaluate.Parameter`接口，对于表达式中使用的未知参数，`govaluate`会自动调用其`Get()`方法获取：

```golang
// src/github.com/Knetic/govaluate/parameters.go
type Parameters interface {
  Get(name string) (interface{}, error)
}
```

例如，我们可以让`User`实现`Parameter`接口：

```golang
type User struct {
  FirstName string
  LastName  string
  Age       int
}

func (u User) Get(name string) (interface{}, error) {
  if name == "FullName" {
    return u.FirstName + " " + u.LastName, nil
  }

  return nil, errors.New("unsupported field " + name)
}

func main() {
  u := User{FirstName: "li", LastName: "dajun", Age: 18}
  expr, _ := govaluate.NewEvaluableExpression("FullName")
  result, _ := expr.Eval(u)
  fmt.Println("user", result)
}
```

表达式对象实际上有两个方法，一个是我们前面用的`Evaluate()`，这个方法接受一个`map[string]interface{}`参数。另一个就是我们在这个例子中使用的`Eval()`方法，该方法接受一个`Parameter`接口。实际上，在`Evaluate()`实现内部也是调用的`Eval()`方法：

```golang
// src/github.com/Knetic/govaluate/EvaluableExpression.go
func (this EvaluableExpression) Evaluate(parameters map[string]interface{}) (interface{}, error) {
  if parameters == nil {
    return this.Eval(nil)
  }
  return this.Eval(MapParameters(parameters))
}
```

在表达式计算时，未知的参数都需要调用`Parameter`的`Get()`方法获取。上面的例子中我们直接使用`FullName`就可以调用`u.Get()`方法返回全名。

## 支持的操作和类型

`govaluate`支持的操作和类型与 Go 语言有些不同。一方面`govaluate`中的类型和操作不如 Go 丰富，另一方面`govaluate`也对一些操作进行了扩展。

算数、比较和逻辑运算：

* `+` `-` `/` `*` `&` `|` `^` `**` `%` `>>` `<<`：加减乘除，按位与，按位或，异或，乘方，取模，左移和右移；
* `>` `>=` `<` `<=` `==` `!=` `=~` `!~`：`=~`为正则匹配，`!~`为正则不匹配；
* `||` `&&`：逻辑或和逻辑与。

常量：

* 数字常量，`govaluate`中将数字都作为 64 位浮点数处理；
* 字符串常量，注意在`govaluate`中，字符串用单引号`'`；
* 日期时间常量，格式与字符串相同，**`govaluate`会尝试自动解析字符串是否是日期**，只支持 RFC3339、ISO8601等有限的格式；
* 布尔常量：`true`、`false`。

其他：

* 圆括号可以改变计算优先级；
* 数组定义在`()`中，每个元素之间用`,`分隔，可以支持任意的元素类型，如`(1, 2, 'foo')`。实际上在`govaluate`中数组是用`[]interface{}`来表示的；
* 三目运算符：`? :`。

在下面代码中，`govaluate`会先将`2014-01-02`和`2014-01-01 23:59:59`转为`time.Time`类型，然后再比较大小：

```golang
func main() {
  expr, _ := govaluate.NewEvaluableExpression("'2014-01-02' > '2014-01-01 23:59:59'")
  result, _ := expr.Evaluate(nil)
  fmt.Println(result)
}
```

## 错误处理

在上面的例子中，我们刻意忽略了错误处理。实际上，`govaluate`在创建表达式对象和表达式求值这两个操作中都可能产生错误。在生成表达式对象时，如果表达式有语法错误，则返回错误。表达式求值，如果传入的参数不合法，或者某些参数缺失，或者访问结构体中不存在的字段都会报错。

```golang
func main() {
  exprString := `>>>`
  expr, err := govaluate.NewEvaluableExpression(exprString)
  if err != nil {
    log.Fatal("syntax error:", err)
  }
  result, err := expr.Evaluate(nil)
  if err != nil {
    log.Fatal("evaluate error:", err)
  }
  fmt.Println(result)
}
```

我们可以依次修改表达式字符串，验证各种错误，首先是`>>>`：

```golang
2020/04/01 22:31:59 syntax error:Invalid token: '>>>'
```

然后我们将其修改为`foo > 0`，但是我们没有传入参数`foo`，执行失败：

```golang
2020/04/01 22:33:07 evaluate error:No parameter 'foo' found.
```

其他错误可以自行验证。

## 总结

`govaluate`虽然支持的操作和类型有限，也能实现比较有意思的功能。例如，可以写一个 Web 服务，由用户自己编写表达式，设置参数，服务器算出结果。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. govaluate GitHub：[https://github.com/Knetic/govaluate](https://github.com/Knetic/govaluate)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)