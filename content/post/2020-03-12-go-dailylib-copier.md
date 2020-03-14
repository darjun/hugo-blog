---
layout:    post
title:    "Go 每日一库之 copier"
subtitle: 	"每天学习一个 Go 库"
date:		2020-03-13T09:40:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/03/13/godailylib/copier"
categories: [
	"Go"
]
---

## 简介

[上一篇文章](https://darjun.github.io/2020/03/11/godailylib/mergo)介绍了`mergo`库的使用，`mergo`是用来给结构体或`map`赋值的。`mergo`有一个明显的不足——它只能处理相同类型的结构！如果类型不同，即使字段名和类型完全相同，`mergo`也无能为力。今天我们要介绍的`copier`库就能处理不同类型之间的赋值。除此之外，`copier`还能：

* 调用同名方法为字段赋值；
* 以源对象字段为参数调用目标对象的方法，从而为目标对象赋值（当然也可以做其它的任何事情）；
* 将切片赋值给切片（可以是不同类型哦）；
* 将结构体追加到切片中。

感谢[@thinkgos](https://github.com/thinkgos)推荐。

顺带一提，作者是国人[jinzhu](https://github.com/jinzhu)大佬，如果你想找一个 Go 语言的 ORM 库，[gorm](https://github.com/jinzhu/gorm)你值得拥有！

## 快速使用

先安装：

```golang
$ go get github.com/jinzhu/copier
```

后使用：

```golang
package main

import (
  "fmt"

  "github.com/jinzhu/copier"
)

type User struct {
  Name string
  Age  int
}

type Employee struct {
  Name string
  Age  int
  Role string
}

func main() {
  user := User{Name: "dj", Age: 18}
  employee := Employee{}

  copier.Copy(&employee, &user)
  fmt.Printf("%#v\n", employee)
}
```

很好理解，就是将`user`对象中的字段赋值到`employee`的同名字段中。如果目标对象中没有同名的字段，则该字段被忽略。

### 高级特性

## 方法赋值

目标对象中的一些字段，源对象中没有，但是源对象有同名的方法。这时`Copy`会调用这个方法，将返回值赋值给目标对象中的字段：

```golang
type User struct {
  Name string
  Age  int
}

func (u *User) DoubleAge() int {
  return 2 * u.Age
}

type Employee struct {
  Name      string
  DoubleAge int
  Role      string
}

func main() {
  user := User{Name: "dj", Age: 18}
  employee := Employee{}

  copier.Copy(&employee, &user)
  fmt.Printf("%#v\n", employee)
}
```

我们给`User`添加一个`DoubleAge`方法。`Employee`结构有字段`DoubleAge`，`User`中没有，但是`User`有一个同名的方法，这时`Copy`调用`user`的`DoubleAge`方法为`employee`的`DoubleAge`赋值，得到 36。

## 调用目标方法

有时候源对象中的某个字段没有出现在目标对象中，但是目标对象有一个同名的方法，方法接受一个同类型的参数，这时`Copy`会以源对象的这个字段作为参数调用目标对象的该方法：

```golang
type User struct {
  Name string
  Age  int
  Role string
}

type Employee struct {
  Name      string
  Age       int
  SuperRole string
}

func (e *Employee) Role(role string) {
  e.SuperRole = "Super" + role
}

func main() {
  user := User{Name: "dj", Age: 18, Role: "Admin"}
  employee := Employee{}

  copier.Copy(&employee, &user)
  fmt.Printf("%#v\n", employee)
}
```

我们给`Employee`添加了一个`Role`方法，`User`的字段`Role`没有出现在`Employee`中，但是`Employee`有一个同名方法。`Copy`函数内部会以`user`对象的`Role`字段为参数调用`employee`的`Role`方法。最终，我们的`employee`对象的`SuperRole`值变为`SuperAdmin`。实际上，这个方法中可以执行任何操作，不一定是赋值。

### 切片赋值

使用一个切片来为另一个切片赋值。如果类型相同，那好办，直接`append`就行。如果类型不同呢？`copier`就派上大用场了：

```golang
type User struct {
  Name string
  Age  int
}

type Employee struct {
  Name string
  Age  int
  Role string
}

func main() {
  users := []User{
    {Name: "dj", Age: 18},
    {Name: "dj2", Age: 18},
  }
  employees := []Employee{}

  copier.Copy(&employees, &users)
  fmt.Printf("%#v\n", employees)
}
```

这个实际上就是将源切片中每个元素分别赋值到目标切片中。

### 将结构赋值到切片

这个不难，实际上就是根据源对象生成一个和目标切片类型相符合的对象，然后`append`到目标切片中：

```golang
type User struct {
  Name string
  Age  int
}

type Employee struct {
  Name string
  Age  int
  Role string
}

func main() {
  user := User{Name: "dj", Age: 18}
  employees := []Employee{}

  copier.Copy(&employees, &user)
  fmt.Printf("%#v\n", employees)
}
```

上面代码中，`Copy`先通过`user`生成一个`Employee`对象，然后`append`到切片`employees`中。

### 汇总

最后将所有的特性汇总在一个例子中，其实就是`Copier`的 GitHub 仓库首页的例子：

```golang
type User struct {
  Name string
  Age  int
  Role string
}

func (u *User) DoubleAge() int {
  return u.Age * 2
}

type Employee struct {
  Name      string
  Age       int
  SuperRole string
}

func (e *Employee) Role(role string) {
  e.SuperRole = "Super" + role
}

func main() {
  var (
    user  = User{Name: "dj", Age: 18}
    users = []User{
      {Name: "dj", Age: 18, Role: "Admin"},
      {Name: "dj2", Age: 18, Role: "Dev"},
    }
    employee  = Employee{}
    employees = []Employee{}
  )

  copier.Copy(&employee, &user)
  fmt.Printf("%#v\n", employee)

  copier.Copy(&employees, &user)
  fmt.Printf("%#v\n", employees)

  // employees = []Employee{}

  copier.Copy(&employees, &users)
  fmt.Printf("%#v\n", employees)
}
```

上面例子中，我故意把`employees = []Employee{}`这一行注释掉，最后输出的`employees`是 3 个元素，能更清楚的看出切片到切片是`append`的，目标切片原来的元素还是保留的。

## 总结

`copier`库的代码量很小，用了不到 200 行的代码就实现了如此实用的一个功能，非常值得一看！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. copier GitHub：[https://github.com/jinzhu/copier](https://github.com/jinzhu/copier)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)