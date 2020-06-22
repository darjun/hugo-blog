---
layout:    post
title:    "Go 每日一库之 negroni"
subtitle: 	"每天学习一个 Go 库"
date:		2020-06-19T08:39:17
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/06/19/godailylib/negroni"
categories: [
	"Go"
]
---

## 简介

[`negroni`](https://github.com/urfave/negroni)是一个专注于 HTTP 中间件的库。它小巧，无侵入，鼓励使用标准库`net/http`的处理器（`Handler`）。本文就来介绍一下这个库。

为什么要使用中间件？有一些逻辑代码，如统计、日志、调试等，每一个处理器中都需要，如果一个个去添加太繁琐了、容易出错、容易遗漏。如果我们要统计处理器耗时，可以在每个处理器中添加代码统计耗时：

```golang
package main

import (
  "fmt"
  "net/http"
  "time"
)

func index(w http.ResponseWriter, r *http.Request) {
  start := time.Now()
  fmt.Fprintf(w, "home page")
  fmt.Printf("index elasped:%fs", time.Since(start).Seconds())
}

func greeting(w http.ResponseWriter, r *http.Request) {
  start := time.Now()
  name := r.FormValue("name")
  if name == "" {
    name = "world"
  }

  fmt.Fprintf(w, "hello %s", name)
  fmt.Printf("greeting elasped:%fs\n", time.Since(start).Seconds())
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", index)
  mux.HandleFunc("/greeting", greeting)

  http.ListenAndServe(":8000", mux)
}
```

但是这个做法非常不灵活：

* 每增加一个处理器，都需要添加这部分代码。而这些代码与实际的处理器逻辑并没有什么关系。编写处理器时比较容易遗忘，特别是要考虑所有的返回路径。增加了编码负担；
* 不利于修改：如果统计代码有错误或者需要调整，必须要改动所有的处理器；
* 添加麻烦：要添加其他的统计逻辑也需要改动所有的处理器代码。

利用 Go 语言的闭包，我们可以将实际的处理器代码封装到一个函数中，在这个函数中执行额外的逻辑：

```golang
func elasped(h func(w http.ResponseWriter, r *http.Request)) http.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request) {
    path := r.URL.Path
    start := time.Now()
    h(w, r)
    fmt.Printf("path:%s elasped:%fs\n", path, time.Since(start).Seconds())
  }
}

func index(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "home page")
}

func greeting(w http.ResponseWriter, r *http.Request) {
  name := r.FormValue("name")
  if name == "" {
    name = "world"
  }

  fmt.Fprintf(w, "hello %s", name)
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", elasped(index))
  mux.HandleFunc("/greeting", elasped(greeting))

  http.ListenAndServe(":8000", mux)
}
```

我们将额外的与处理器无关的代码放在另外的函数中。注册处理器函数时，我们不直接使用原始的处理器函数，而是用`elasped`函数封装一层。实际上`elasped`这样的函数就是中间件。它封装原始的处理器函数，返回一个新的处理器函数。从而能很方便在实际的处理逻辑前后插入代码，便于添加、修改和维护。

## 快速使用

先安装：

```cmd
$ go get github.com/urfave/negroni
```

后使用：

```golang
package main

import (
  "fmt"
  "net/http"

  "github.com/urfave/negroni"
)

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello World!")
  })

  n := negroni.Classic()
  n.UseHandler(mux)

  http.ListenAndServe(":3000", n)
}
```

`negroni`的使用非常简单，它可以很方便的与`http.Handler`一起使用。`negroni.Classic()`提供了几个常用的中间件：

* `negroni.Recovery`：恢复`panic`，处理器代码中有`panic`会被这个中间件捕获，程序不会退出；
* `negroni.Logger`：日志，记录请求和响应的基本信息；
* `negroni.Static`：在`public`目录提供静态文件服务。

调用`n.UseHandler(mux)`，将这些中间件应用到多路复用器上。运行，在浏览器中输入`localhost:3000`，查看控制台输出：

```cmd
$ go run main.go 
[negroni] 2020-06-22T06:48:53+08:00 | 200 |      20.9966ms | localhost:3000 | GET /
[negroni] 2020-06-22T06:48:54+08:00 | 200 |      0s | localhost:3000 | GET /favicon.ico

```

## `negroni.Handler`

接口`negroni.Handler`让我们对中间件的执行流程有更灵活的控制：

```golang
type Handler interface {
  ServeHTTP(w http.ResponseWriter, r *http.Request, next http.HandlerFunc)
}
```

我们编写的中间件签名必须是`func(http.ResponseWriter,*http.Request,http.HandlerFunc)`，或者实现`negroni.Handler`接口：

```golang
func RandomMiddleware(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
  if rand.Int31n(100) <= 50 {
    fmt.Fprintf(w, "hello from RandomMiddleware")
  } else {
    next(w, r)
  }
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello World!")
  })

  n := negroni.New()
  n.Use(negroni.HandlerFunc(RandomMiddleware))
  n.UseHandler(mux)

  http.ListenAndServe(":3000", n)
}
```

上面代码中实现了一个随机的中间件，有一半的概率直接从`RandomMiddleware`这个中间件返回，一半的概率执行实际的处理器函数。运行程序，在浏览器中不停地刷新页面`localhost:3000`看看效果。

注意，实际上`func(w http.ResponseWriter, r *http.Request, next http.HandlerFunc)`只是一个方便的写法。在调用`n.Use`时使用了`negroni.HandlerFunc`做了一层封装，而`negroni.HandlerFunc`实现了`negroni.Handler`接口：

```golang
// src/github.com/urfave/negroni/negroni.go
type HandlerFunc func(rw http.ResponseWriter, r *http.Request, next http.HandlerFunc)

func (h HandlerFunc) ServeHTTP(rw http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
  h(rw, r, next)
}
```

`net/http`中也有类似的代码，通过`http.HandlerFunc`封装`func(http.ResponseWriter,*http.Request)`从而实现接口`http.Handler`。

## `negroni.With`

如果有多个中间件，每个都需要`n.Use()`有些繁琐。`negroni`提供了一个`With()`方法，它接受一个或多个`negroni.Handler`参数，返回一个新的对象：

```golang
func Middleware1(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
  fmt.Println("Middleware1 begin")
  next(w, r)
  fmt.Println("Middleware1 end")
}

func Middleware2(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
  fmt.Println("Middleware2 begin")
  next(w, r)
  fmt.Println("Middleware2 end")
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello World!")
  })

  n := negroni.New()
  n = n.With(
    negroni.HandlerFunc(Middleware1),
    negroni.HandlerFunc(Middleware2),
  )
  n.UseHandler(mux)

  http.ListenAndServe(":3000", n)
}
```

## `Run`

`Negroni`对象提供了一个方便的`Run()`方法来运行服务器程序。它接受与`http.ListenAndServe()`一样的地址（`Addr`）参数：

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello World!")
  })

  n := negroni.New()
  n.UseHandler(mux)
  n.Run(":3000")
}
```

如果未指定端口，那么尝试使用`PORT`环境变量。如果`PORT`环境变量也未设置，那么使用默认的端口`:8080`。

## 作为`http.Handler`使用

`negroni`很容易在`net/http`程序中使用，`negroni.Negroni`对象可直接作为`http.Handler`传给相应的方法：

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello World!")
  })

  n := negroni.Classic()
  n.UseHandler(mux)

  s := &http.Server{
    Addr:           ":8080",
    Handler:        n,
    ReadTimeout:    10 * time.Second,
    WriteTimeout:   10 * time.Second,
    MaxHeaderBytes: 1 << 20,
  }
  s.ListenAndServe()
}
```

## 内置中间件

`negroni`内置了一些常用的中间件，可直接使用。

### `Static`

`negroni.Static`可在指定目录中提供文件服务：

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello world")
  })

  n := negroni.New()
  n.Use(negroni.NewStatic(http.Dir("./public")))
  n.UseHandler(mux)

  http.ListenAndServe(":3000", n)
}
```

在程序运行目录下创建`public`目录，然后放入一些文件`1.txt`，`2.jpg`。程序运行之后，就能通过浏览器`localhost:3000/1.txt`和`localhost:3000/2.jpg`请求这些文件了。

另外需要特别注意一点，如果找不到对应的文件，`Static`会将请求传给下一个中间件或处理器函数。在上面的例子中就是`hello world`。在浏览器中输入`localhost:3000/none-exist.txt`看看效果。

### `Logger`

在快速开始中，我们通过`negroni.Classic()`已经使用过这个中间件了。我们也可以单独使用，它可以记录请求的信息。我们还可以调用`SetFormat()`方法设置日志的格式：

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello world")
  })

  n := negroni.New()
  logger := negroni.NewLogger()
  logger.SetFormat("[{{.Status}} {{.Duration}}] - {{.Request.UserAgent}}")
  n.Use(logger)
  n.UseHandler(mux)

  http.ListenAndServe(":3000", n)
}
```

上面代码中将日志格式设置为`[{{.Status}} {{.Duration}}] - {{.Request.UserAgent}}`，即响应状态、耗时和`UserAgent`。

使用 Chrome 浏览器请求：

```cmd
[negroni] [200 26.0029ms] - Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.106 Safari/537.36
```

### `Recovery`

`negroni.Recovery`可以捕获后续的中间件或处理器函数中出现的`panic`，返回一个`500`的响应码：

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    panic("internal server error")
  })

  n := negroni.New()
  n.Use(negroni.NewRecovery())
  n.UseHandler(mux)

  http.ListenAndServe(":3000", n)
}
```

请求时`panic`的堆栈会显示在浏览器中：

![](/img/in-post/godailylib/negroni1.png#center)

这在开发环境比较有用，但是生成环境中不能泄露这个信息。这时可以设置`PrintStack`字段为`false`：

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    panic("internal server error")
  })

  n := negroni.New()
  r := negroni.NewRecovery()
  r.PrintStack = false
  n.Use(r)
  n.UseHandler(mux)

  http.ListenAndServe(":3000", n)
}
```

除了在控制台和浏览器中输出`panic`信息，`Recovery`还提供了钩子函数，可以向其他服务上报`panic`，如`Sentry/Airbrake`。当然上报的代码要自己写😄。

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    panic("internal server error")
  })

  n := negroni.New()
  r := negroni.NewRecovery()
  r.PanicHandlerFunc = reportToSentry
  n.Use(r)
  n.UseHandler(mux)

  http.ListenAndServe(":3000", n)
}

func reportToSentry(info *negroni.PanicInformation) {
  fmt.Println("sent to sentry")
}
```

设置`PanicHandlerFunc`之后，发生`panic`就会调用此函数。

我们还可以对输出的格式进行设置，设置`Formatter`字段为`negroni.HTMLPanicFormatter`能让输出更好地在浏览器中呈现：

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    panic("internal server error")
  })

  n := negroni.New()
  r := negroni.NewRecovery()
  r.Formatter = &negroni.HTMLPanicFormatter{}
  n.Use(r)
  n.UseHandler(mux)

  http.ListenAndServe(":3000", n)
}
```

效果：

![](/img/in-post/godailylib/negroni2.png#center)

### 第三方中间件

除了内置中间件外，`negroni`还有很多第三方的中间件。完整列表看这里：[https://github.com/urfave/negroni#third-party-middleware](https://github.com/urfave/negroni#third-party-middleware)。

我们只介绍一个[`xrequestid`](https://github.com/gravityblast/xrequestid)，它在每个请求中增加一个随机的`Header`：`X-Request-Id`。

安装`xrequestid`：

```cmd
$ go get github.com/pilu/xrequestid
```

使用：

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "X-Request-Id is `%s`", r.Header.Get("X-Request-Id"))
  })

  n := negroni.New()
  n.Use(xrequestid.New(16))
  n.UseHandler(mux)
  n.Run(":3000")
}
```

给每个请求增加一个 16 字节的`X-Request-Id`，处理器函数中将这个`X-Request-Id`写入响应中，最后呈现在浏览器中。运行程序，在浏览器中输入`localhost:3000`查看效果。

## 总结

`negroni`专注于中间件，没有很多花哨的功能。无侵入性使得它很容易与标准库`net/http`和其他的 Web 库（如`gorilla/mux`）一起使用。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. negroni GitHub：[https://github.com/urfave/negroni](https://github.com/urfave/negroni)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)