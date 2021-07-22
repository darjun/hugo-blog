---
layout:    post
title:    "Go 每日一库之 gorilla/schema"
subtitle: "每天学习一个 Go 库"
date:     2021-07-22T07:48:33
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/07/22/godailylib/gorilla/schema"
categories: [
  "Go"
]
---

## 简介

[`gorilla/schema`](https://github.com/gorilla/schema) 是 gorilla 开发工具包中用于处理表单的库。它提供了一个简单的方式，可以很方便地将表单数据转为结构体对象，或者将结构体对象转为表单数据。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir gorilla/schema && cd gorilla/schema
$ go mod init github.com/darjun/go-daily-lib/gorilla/schema
```

安装`gorilla/schema`库：

```cmd
$ go get -u github.com/gorilla/schema
```

我们还是拿前面登录的例子：

```golang
func index(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, "Hello World")
}

func login(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, `<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Login</title>
</head>
<body>
<form action="/login" method="post">
  <label>Username:</label>
  <input name="username"><br>
  <label>Password:</label>
  <input name="password" type="password"><br>
  <button type="submit">登录</button>
</form>
</body>
</html>`)
}

type User struct {
  Username string `schema:"username"`
  Password string `schema:"password"`
}

var (
  decoder = schema.NewDecoder()
)

func dologin(w http.ResponseWriter, r *http.Request) {
  r.ParseForm()
  u := User{}
  decoder.Decode(&u, r.PostForm)
  if u.Username == "dj" && u.Password == "handsome" {
    http.Redirect(w, r, "/", 301)
    return
  }

  http.Redirect(w, r, "/login", 301)
}

func main() {
  r := mux.NewRouter()
  r.HandleFunc("/", index)
  r.Handle("/login", handlers.MethodHandler{
    "GET":  http.HandlerFunc(login),
    "POST": http.HandlerFunc(dologin),
  })
  log.Fatal(http.ListenAndServe(":8080", r))
}
```

首先调用`schema.NewDecoder()`方法创建一个解码器`decoder`。在处理器中，先调用`r.ParseForm()`解析表单数据，然后创建用户对象`u`，调用`decoder.Decode(&u, r.PostForm)`通过表单数据填充该对象。

`schema`使用反射来对应表单和结构体字段，我们可以通过结构体标签来指定表单数据和字段的对应关系。

**上面我们将解码器作为一个包内的全局变量，因为`decoder`中会缓存一些结构体的元数据，并且它是并发安全的**。

在`main`函数中，我们创建了`gorilla/mux`路由，注册`/`根处理函数，使用中间件`handlers.MethodHandler`分别注册路径`/login`的 GET 和 POST 方法的处理器。然后调用`http.Handle("/", r)`将所有请求交由`gorilla/mux`路由处理。最后启动 Web 服务器接受请求。

## 编码

除了用于服务器解码表单数据外，`schema`还可用于客户端，将结构体对象编码到表单数据中发送给服务器。我们编写一个程序登录上面的服务器：

```golang
var (
  encoder = schema.NewEncoder()
)

func main() {
  client := &http.Client{}
  form := url.Values{}

  u := &User{
    Username: "dj",
    Password: "handsome",
  }
  encoder.Encode(u, form)

  res, _ := client.PostForm("http://localhost:8080/login", form)
  data, _ := ioutil.ReadAll(res.Body)
  fmt.Println(string(data))
  res.Body.Close()
}
```

与解码器的用法类似，首先调用`schema.NewEncoder()`创建一个编码器`encoder`，创建一个`User`类型的对象`u`和表单数据对象`form`，调用`encoder.Encode(u, form)`将`u`编码到`form`中。然后使用`http.Client`的`PostForm`方法发送请求。读取响应。

## 自定义类型转换

目前`schema`支持以下类型：

* 布尔类型：`bool`
* 浮点数：`float32/float64`
* 有符号整数：`int/int8/int32/int64`
* 无符号整数：`uint/uint8/uint32/uint64`
* 字符串：`string`
* 结构体：由以上类型组成的结构体
* 指针：指向以上类型的指针
* 切片：元素为以上类型的切片，或指向切片的指针

有时候客户端会将一个切片拼成一个字符串传到服务器，服务器收到之后需要解析成切片：

```golang
type Person struct {
  Name    string   `schema:"name"`
  Age     int      `schema:"age"`
  Hobbies []string `schema:"hobbies"`
}

var (
  decoder = schema.NewDecoder()
)

func init() {
  decoder.RegisterConverter([]string{}, func(s string) reflect.Value {
    return reflect.ValueOf(strings.Split(s, ","))
  })
}

func doinfo(w http.ResponseWriter, r *http.Request) {
  r.ParseForm()
  p := Person{}
  decoder.Decode(&p, r.PostForm)

  fmt.Println(p)
  fmt.Fprintf(w, "Name:%s Age:%d Hobbies:%v", p.Name, p.Age, p.Hobbies)
}
```

调用`decoder.RegisterConverter()`注册对应类型的转换函数，转换函数类型为：

```golang
func(s string) reflect.Value
```

即将请求中的字符串值转为满足我们格式的值。


客户端请求：

```golang
type Person struct {
  Name    string `schema:"name"`
  Age     int    `schema:"age"`
  Hobbies string `schema:"hobbies"`
}

var (
  encoder = schema.NewEncoder()
)

func main() {
  client := &http.Client{}
  form := url.Values{}

  p := &Person{
    Name:    "dj",
    Age:     18,
    Hobbies: "Game,Programming",
  }
  encoder.Encode(p, form)

  res, _ := client.PostForm("http://localhost:8080/info", form)
  data, _ := ioutil.ReadAll(res.Body)
  fmt.Println(string(data))
  res.Body.Close()
}
```

客户端故意将`Hobbies`字段设置为字符串，发送请求。服务器将使用注册的`func (s string) reflect.Value`函数将该字符串切割为`[]string`返回。

## 总结

`schema`提供了一个简单的获取表单数据的方式，通过将数据填充到结构体对象中，我们可以很方便的进行后续操作。`schema`库比较小巧，对特性没太多要求的可以试试~

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gorilla/schema GitHub：[github.com/gorilla/schema](https://github.com/gorilla/schema)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)