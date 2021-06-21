---
layout:    post
title:    "Go 每日一库之 ozzo-validation"
subtitle: "每天学习一个 Go 库"
date:     2021-06-20T15:18:30
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/06/20/godailylib/ozzo-validation"
categories: [
  "Go"
]
---

## 简介

[`ozzo-validation`](https://github.com/go-ozzo/ozzo-validation)是一个非常强大的，灵活的数据校验库。与其他基于 struct tag 的数据校验库不同，`ozzo-validation`认为 struct tag 在使用过程中比较容易出错。因为 struct tag 本质上就是字符串，完全基于字符串的解析，无法利用语言的静态检查机制，很容易在不知不觉中写错而不易察觉，实际代码中出现错误也很难排查。

`ozzo-validation`提倡用代码指定规则来进行校验。实际上`ozzo`是辅助开发 Web 应用程序的一套框架，包括 ORM 库[`ozzo-dbx`](https://github.com/go-ozzo/ozzo-dbx)，路由库[`ozzo-routing`](https://github.com/go-ozzo/ozzo-routing)，日志库[`ozzo-log`](https://github.com/go-ozzo/ozzo-log)，配置库[`ozzo-config`](https://github.com/go-ozzo/ozzo-config)以及最出名的，使用最为广泛的数据校验库[`ozzo-validation`](https://github.com/go-ozzo/ozzo-validation)。作者甚至还搞出了一个开发 Web 应用程序的模版[`go-rest-api`](https://github.com/qiangxue/go-rest-api)。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir ozzo-validation && cd ozzo-validation
$ go mod init github.com/darjun/go-daily-lib/ozzo-validation
```

安装`ozzo-validation`库：

```cmd
$ go get -u github.com/go-ozzo/ozzo-validation/v4
```

`ozzo-validation`的程序写起来都比较直观：

```golang
package main

import (
  "fmt"

  "github.com/go-ozzo/ozzo-validation/v4/is"
  "github.com/go-ozzo/ozzo-validation/v4"
)

func main() {
  name := "darjun"

  err := validation.Validate(name,
    validation.Required,
    validation.Length(2, 10),
    is.URL)
  fmt.Println(err)
}
```

`ozzo-validation`使用函数`Validate()`对基本类型值进行校验，传入的第一个参数就是要校验的数据，后面以可变参数传入一个或多个校验规则。上例中对一个字符串做校验。我们用代码来表达规则：

* `validation.Required`：表示值必须设置，对于字符串来说就是不能为空；
* `validation.Length(2, 10)`：指定长度的范围；
* `is.URL`：`is`子包中内置了大量的辅助方法，`is.URL`限制值必须是 URL 格式。

`Validate()`函数根据传入的规则按顺序依次对数据进行校验，直到遇到某个规则校验失败，或所有规则都校验成功。如果一个规则返回失败，则跳过后面的规则直接返回错误。如果数据通过了所有规则，则返回一个`nil`。

运行上面程序输出：

```golang
must be a valid URL
```

因为字符串"darjun"明显不是一个合法的 URL。如果去掉`is.URL`规则，则运行输出`nil`。

## 结构体

使用`ValidateStruct()`函数可以对一个结构体对象进行校验。我们需要依次指定结构体中各个字段的校验规则：

```golang
type User struct {
  Name  string
  Age   int
  Email string
}

func validateUser(u *User) error {
  err := validation.ValidateStruct(u,
    validation.Field(&u.Name, validation.Required, validation.Length(2, 10)),
    validation.Field(&u.Age, validation.Required, validation.Min(1), validation.Max(200)),
    validation.Field(&u.Email, validation.Required, validation.Length(10, 50), is.Email))

  return err
}
```

`ValidateStruct()`接受一个结构体的指针作为第一个参数，后面依次指定各个字段的规则。字段规则使用`validation.Field()`函数指定，该函数接受一个指向具体字段的指针，后跟一个或多个规则。上面我们限制，名字长度在[2, 10]之间，年龄在[1, 200]之间（姑且认为现在人类最多能活 200 年），电子邮箱长度在[10, 50]之间，并且使用`is.Email`限制它必须是一个合法的邮箱地址。同时这 3 个字段都是必填的（用`validation.Required`限制的）。

然后我们构造一个合法的`User`对象和一个非法的`User`对象，分别校验：

```golang
func main() {
  u1 := &User {
    Name: "darjun",
    Age: 18,
    Email: "darjun@126.com",
  }
  fmt.Println("user1:", validateUser(u1))

  u2 := &User {
    Name: "lidajun12345",
    Age: 201,
    Email: "lidajun's email",
  }
  fmt.Println("user2:", validateUser(u2))
}
```

程序运行输出：

```golang
user1: <nil>
user2: Age: must be no greater than 200; Email: must be a valid email address; Name: the length must be between 2 and 10.
```

对于结构体来说，`validation`依次对每个字段检验传入的规则。对于某个字段，如果一条规则校验失败了，则跳过后面的规则，**继续校验下一个字段**。如果某个字段校验失败，会在结果中包含关于该字段的错误信息，如上例。

## Map

有时数据保存在一个`map`中，而非一个结构体中。这时可以使用`validation.Map()`指定校验`map`的规则，`validation.Map()`规则中需要使用`validation.Key()`依次指定各个键对应的一个或多个规则。最后将`map`类型的数据和`validation.Map()`规则传给`validation.Validate()`函数校验：

```golang
func validateUser(u map[string]interface{}) error {
  err := validation.Validate(u, validation.Map(
    validation.Key("name", validation.Required, validation.Length(2, 10)),
    validation.Key("age", validation.Required, validation.Min(1), validation.Max(200)),
    validation.Key("email", validation.Required, validation.Length(10, 50), is.Email),
  ))

  return err
}

func main() {
  u1 := map[string]interface{} {
    "name": "darjun",
    "age": 18,
    "email": "darjun@126.com",
  }
  fmt.Println("user1:", validateUser(u1))

  u2 := map[string]interface{} {
    "name": "lidajun12345",
    "age": 201,
    "email": "lidajun's email",
  }
  fmt.Println("user2:", validateUser(u2))
}
```

我们改造了上面的例子，改用`map[string]interface{}`存储`User`信息。`map`的校验与结构体类似，根据`validation.Map()`中指定的键的顺序依次校验。如果某个键校验失败，记录错误信息。最终汇总所有键的错误信息返回。运行程序：

```golang
user1: <nil>
user2: age: must be no greater than 200; email: must be a valid email address; name: the length must be between 2 and 10.
```

## 可校验类型

`ozzo-validation`库提供了一个接口`Validatable`：

```golang
type Validatable interface {
  // Validate validates the data and returns an error if validation fails.
  Validate() error
}
```

凡是实现了`Validatable`接口的类型都是可校验的类型。`validation.Validate()`函数在校验某个类型的数据时，先校验传入该函数的所有规则。如果这些规则都通过了，那么`Validate()`函数判断该类型有没有实现`Validatbale`接口。如果实现了，则调用其`Validate()`方法进行校验。我们让上例中`User`类型实现`Validatable`接口：

```golang
type User struct {
  Name   string
  Age    int
  Gender string
  Email  string
}

func (u *User) Validate() error {
  err := validation.ValidateStruct(u,
    validation.Field(&u.Name, validation.Required, validation.Length(2, 10)),
    validation.Field(&u.Age, validation.Required, validation.Min(1), validation.Max(200)),
    validation.Field(&u.Gender, validation.Required, validation.In("male", "female")),
    validation.Field(&u.Email, validation.Required, validation.Length(10, 50), is.Email))

  return err
}
```

由于`User`实现了`Validatable`接口，我们可以直接调用`Validate()`函数校验：

```golang
func main() {
  u1 := &User{
    Name:   "darjun",
    Age:    18,
    Gender: "male",
    Email:  "darjun@126.com",
  }
  fmt.Println("user1:", validation.Validate(u1, validation.NotNil))

  u2 := &User{
    Name:  "lidajun12345",
    Age:   201,
    Email: "lidajun's email",
  }
  fmt.Println("user2:", validation.Validate(u2, validation.NotNil))
}
```

在通过了`NotNil`校验后，`Validate()`函数还会调用`User.Validate()`方法进行校验。

需要注意的是，在实现了`Validatable`接口的类型的`Validate()`方法内部，不能直接对该类型的值调用`validation.Validate()`函数，这会导致无限递归：

```golang
type UserName string

func (n UserName) Validate() error {
  return validation.Validate(n,
    validation.Required, validation.Length(2, 10))
}

func main() {
  var n1, n2 UserName = "dj", "lidajun12345"

  fmt.Println("username1:", validation.Validate(n1))
  fmt.Println("username2:", validation.Validate(n2))
}
```

我们基于`string`定义了一个新类型`UserName`，规定`UserName`非空，并且长度在[2, 10]范围内。但是上面的`Validate()`方法中将`UserName`类型的变量`n`传入了函数`validation.Validate()`。该函数内部检查发现`UserName`实现了`Validatable`接口，又会调用它的`Validate()`方法，导致无限递归。

我们只需要简单地将`n`转为`string`类型即可：

```golang
func (n UserName) Validate() error {
  return validation.Validate(string(n),
    validation.Required, validation.Length(2, 10))
}
```

### 可校验类型的集合

`Validate()`函数对元素为可校验类型（即实现了`Validatable`接口）的集合（切片/数组/map等）进行校验时，会依次调用其元素的`Validate()`方法，最后校验返回一个`validation.Errors`类型。这实际上是一个`map[string]error`类型。键为元素的键（对于切片和数组就是索引，对于`map`就是键），值为错误值。例：

```golang
func main() {
  u1 := &User{
    Name:   "darjun",
    Age:    18,
    Gender: "male",
    Email:  "darjun@126.com",
  }
  u2 := &User{
    Name:  "lidajun12345",
    Age:   201,
    Email: "lidajun's email",
  }

  userSlice := []*User{u1, u2}
  userMap := map[string]*User{
    "user1": u1,
    "user2": u2,
  }

  fmt.Println("user slice:", validation.Validate(userSlice))
  fmt.Println("user map:", validation.Validate(userMap))
}
```

`userSlice`切片中第二个元素的校验错误会在结果的键`1`（索引）中返回，`userMap`中键`user2`校验的错误会在结果的键`user2`中返回。运行结果：

```golang
user slice: 1: (Age: must be no greater than 200; Email: must be a valid email address; Gender: cannot be blank; Name: the length must be between 2 and 10.).
user map: user2: (Age: must be no greater than 200; Email: must be a valid email address; Gender: cannot be blank; Name: the length must be between 2 and 10.).
```

如果需要集合中每个元素都满足某些规则，我们可以使用`validation.Each()`函数。例如，我们的`User`对象有多个邮箱，要求每个邮箱地址的格式都合法：

```golang
type User struct {
  Name   string
  Age    int
  Emails []string
}

func (u *User) Validate() error {
  return validation.ValidateStruct(u,
    validation.Field(&u.Emails, validation.Each(is.Email)))
}

func main() {
  u := &User{
    Name: "dj",
    Age:  18,
    Emails: []string{
      "darjun@126.com",
      "don't know",
    },
  }
  fmt.Println(validation.Validate(u))
}
```

错误消息中会指出哪个位置数据不合法了：

```golang
Emails: (1: must be a valid email address.).
```

## 条件规则

我们可以根据某个字段的值来给另一个字段设置规则。例如我们的`User`对象有两个字段：布尔值`Student`表示是否还是学生，字符串`School`表示学校。在`Student`为`true`时，字段`School`必须存在并且长度在[10, 20]范围内：

```golang
type User struct {
  Name    string
  Age     int
  Student bool
  School  string
}

func (u *User) Validate() error {
  return validation.ValidateStruct(u,
    validation.Field(&u.Name, validation.Required, validation.Length(2, 10)),
    validation.Field(&u.Age, validation.Required, validation.Min(1), validation.Max(200)),
    validation.Field(&u.School, validation.When(u.Student, validation.Required, validation.Length(10, 20))))
}

func main() {
  u1 := &User{
    Name:    "dj",
    Age:     18,
    Student: true,
  }

  u2 := &User{
    Name: "lidajun",
    Age:  31,
  }

  fmt.Println("user1:", validation.Validate(u1))
  fmt.Println("user2:", validation.Validate(u2))
}
```

我们使用`validation.When()`函数，该函数接受一个布尔值作为第一个参数，一个或多个规则作为后面的可变参数。只有在第一个参数为`true`时才执行后面的规则校验。

`u1`因为设置了字段`Student`为`true`，所以`School`字段不能为空。`u2`因为`Student=false`，`School`字段可有可无。运行：

```golang
user1: School: cannot be blank.
user2: <nil>
```

在检查注册用户信息时，我们确保用户必须设置了邮箱或手机号也可以用条件规则：

```golang
type User struct {
  Email string
  Phone string
}

func (u *User) Validate() error {
  return validation.ValidateStruct(u,
    validation.Field(&u.Email, validation.When(u.Phone == "", validation.Required.Error("Either email or phone is required."), is.Email)),
    validation.Field(&u.Phone, validation.When(u.Email == "", validation.Required.Error("Either email or phone is required."), is.Alphanumeric)))
}

func main() {
  u1 := &User{}

  u2 := &User{
    Email: "darjun@126.com",
  }

  u3 := &User{
    Phone: "17301251652",
  }

  u4 := &User{
    Email: "darjun@126.com",
    Phone: "17301251652",
  }

  fmt.Println("user1:", validation.Validate(u1))
  fmt.Println("user2:", validation.Validate(u2))
  fmt.Println("user3:", validation.Validate(u3))
  fmt.Println("user4:", validation.Validate(u4))
}
```

如果`Phone`字段为空，`Email`必须设置。反之，如果`Email`字段为空，`Phone`必须设置。所有的规则都可以调用`Error()`方法设置自定义错误信息。运行输出：

```golang
user1: Email: Either email or phone is required.; Phone: Either email or phone is required..
user2: <nil>
user3: <nil>
user4: <nil>
```

## 自定义规则

除了库提供的规则之外，我们还可以定义自己的规则。规则实现为一个如下类型的函数：

```golang
func Validate(value interface{}) error
```

下面我们实现一个检查 IP 地址是否合法的函数。这里我们介绍一个库[`commonregex`](https://github.com/mingrammer/commonregex)。这个库收录了绝大部分常用的正则表达式。我之前也写过一篇文章介绍这个库的使用，[Go 每日一库之 commonregex](https://darjun.github.io/2020/09/05/godailylib/commonregex/)，感兴趣可以过去看看。

```golang
func checkIP(value interface{}) error {
  ip, ok := value.(string)
  if !ok {
    return errors.New("ip must be string")
  }

  ipList := commonregex.IPs(ip)
  if len(ipList) != 1 || ipList[0] != ip {
    return errors.New("invalid ip format")
  }

  return nil
}
```

然后定义一个网络地址结构及校验方法，通过`validation.By()`函数使用自定义的校验函数：

```golang
type Addr struct {
  IP   string
  Port int
}

func (a *Addr) Validate() error {
  return validation.ValidateStruct(a,
    validation.Field(&a.IP, validation.Required, validation.By(checkIP)),
    validation.Field(&a.Port, validation.Min(1024), validation.Max(65536)))
}
```

验证：

```golang
func main() {
  a1 := &Addr{
    IP:   "127.0.0.1",
    Port: 6666,
  }

  a2 := &Addr{
    IP:   "xxx.yyy.zzz.hhh",
    Port: 7777,
  }

  fmt.Println("addr1:", validation.Validate(a1))
  fmt.Println("addr2:", validation.Validate(a2))
}
```

运行：

```golang
addr1: <nil>
addr2: IP: invalid ip format.
```

## 规则组

每次指定规则都一个一个地来指定有点不方便，这时我们可以将常用的校验规则组成一个规则组，需要时直接使用这个组即可。例如，我们项目中约定合法的用户名必须是 ASCII 字母加数字，长度为 10-20，用户名肯定不能为空。规则组没什么特殊的，它只是一个规则的切片：

```golang
var NameRule = []validation.Rule{
  validation.Required,
  is.Alphanumeric,
  validation.Length(10, 20),
}

func main() {
  name1 := "lidajun12345"
  name2 := "lidajun@!#$%"
  name3 := "short"
  name4 := "looooooooooooooooooong"

  fmt.Println("name1:", validation.Validate(name1, NameRule...))
  fmt.Println("name2:", validation.Validate(name2, NameRule...))
  fmt.Println("name3:", validation.Validate(name3, NameRule...))
  fmt.Println("name4:", validation.Validate(name4, NameRule...))
}
```

运行：

```golang
name1: <nil>
name2: must contain English letters and digits only
name3: the length must be between 10 and 20
name4: the length must be between 10 and 20
```

## 总结

`ozzo-validation`提倡以代码指定规则代替容易出错的`struct tag`，并提供了大量的内置规则。使用`ozzo-validation`编写的代码清晰，易读，而且对编译器友好（很多错误都暴露在编译期）。本文介绍了`ozzo-validation`库的基本使用，核心就两个函数`Validate()`和`ValidateStruct()`，前者用于校验基本类型或可校验的类型，后者用于校验结构体。实际编码过程中，一般都会让结构体实现`Validatbale`接口将它变为可校验类型，再调用`Validate()`函数校验。

`ozzo-validation`还可以对集合进行校验，可以自定义校验规则，可以定义通用的校验组。除此之外，`ozzo-validation`还有很多高级特性，如自定义错误，基于`context.Context`的校验，使用正则表达式定义规则等，感兴趣可自行探索。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. ozzo-validation GitHub：[github.com/go-ozzo/ozzo-validation](https://github.com/go-ozzo/ozzo-validation)
2. go-rest-api GitHub：[github.com/qiangxue/go-rest-api](https://github.com/qiangxue/go-rest-api)
3. Go 每日一库之 commonregex：[https://darjun.github.io/2020/09/05/godailylib/commonregex/](https://darjun.github.io/2020/09/05/godailylib/commonregex/)
4. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~