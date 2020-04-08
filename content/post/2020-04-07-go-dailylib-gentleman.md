---
layout:    post
title:    "Go 每日一库之 gentleman"
subtitle: 	"每天学习一个 Go 库"
date:		2020-04-07T21:30:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/04/07/godailylib/gentleman"
categories: [
	"Go"
]
---

## 简介

[`gentleman`](https://github.com/h2non/gentleman)是一个功能齐全、**插件**驱动的 HTTP 客户端。`gentleman`以扩展性为原则，可以基于内置的或第三方插件创建具有丰富特性的、可复用的 HTTP 客户端。相比标准库`net/http`，`gentleman`更灵活、易用。

## 快速使用

先安装：

```golang
$ go get gopkg.in/h2non/gentleman.v2
```

后使用：

```golang
package main

import (
  "fmt"

  "gopkg.in/h2non/gentleman.v2"
)

func main() {
  cli := gentleman.New()

  cli.URL("https://dog.ceo")

  req := cli.Request()

  req.Path("/api/breeds/image/random")

  req.SetHeader("Client", "gentleman")

  res, err := req.Send()

  if err != nil {
    fmt.Printf("Request error: %vn", err)
    return
  }

  if !res.Ok {
    fmt.Printf("Invalid server response: %dn", res.StatusCode)
    return
  }

  fmt.Printf("Body: %s", res.String())
}
```

`gentleman`目前有两个版本`v1`和`v2`，`v2`已经稳定，推荐使用，示例中使用的就是`v2`。`gentleman`的使用遵循下面的流程：

* 调用`gentleman.New()`创建一个 HTTP 客户端`cli`，**此`cli`对象可复用**；
* 调用`cli.URL()`设置要请求的 URL 基础地址；
* 调用`cli.Request()`创建一个请求对象`req`；
* 调用`req.Path()`设置请求的路径，基于前面设置的 URL；
* 调用`req.Header()`设置请求首部（`Header`），上面代码设置首部`Client`为`gentleman`；
* 调用`req.Send()`发送请求，获取响应对象`res`；
* 对响应对象`res`进行处理。

上面的测试 API 是我从[public-apis](https://github.com/public-apis/public-apis)找的。`public-apis`是 GitHub 上一个收集各种开放 API 的仓库。本文后面部分的 API 也来自于这个仓库。从`https://dog.ceo`我们可以获取各种和**狗**相关的信息，上面请求的路径`/api/breeds/image/random`将返回一个随机品种的狗的图片。运行结果：

```golang
Body: {"message":"https://images.dog.ceo/breeds/malamute/n02110063_10567.jpg","status":"success"}
```

由于是随机的，每次运行结果可能都不相同，`status`为`success`表示运行成功，`message`对应的值为图片的 URL。感兴趣自己在浏览器中打开返回的 URL，我获取的图片如下：

![](/img/in-post/godailylib/gentleman1.jpg#center)

## 插件

`gentleman`中的特性很多都是通过**插件**来实现的。`gentleman`内置了很多常用的插件。如果要实现的特性无法通过内置插件来完成，还有第三方插件可供选择，当然还可以自定义插件！`gentleman`的插件都是存放在`plugins`子目录中的，下面介绍几个常用的插件。

### `body`

客户端有时需要发送 JSON、XML 等格式的数据，`body`插件可以很好地完成这个任务：

```golang
package main

import (
  "fmt"
  "gopkg.in/h2non/gentleman.v2"
  "gopkg.in/h2non/gentleman.v2/plugins/body"
)

func main() {
  cli := gentleman.New()
  cli.URL("http://httpbin.org/post")

  data := map[string]string{"foo": "bar"}
  cli.Use(body.JSON(data))

  req := cli.Request()
  req.Method("POST")

  res, err := req.Send()
  if err != nil {
    fmt.Printf("Request error: %s\n", err)
    return
  }

  if !res.Ok {
    fmt.Printf("Invalid server response: %d\n", res.StatusCode)
    return
  }

  fmt.Printf("Status: %d\n", res.StatusCode)
  fmt.Printf("Body: %s", res.String())
}
```

注意插件的导入方式：`import "gopkg.in/h2non/gentleman.v2/plugins/body"`。

调用客户端对象`cli`或请求对象`req`的`Use()`方法使用插件。区别在于`cli.Use()`调用之后，所有通过该`cli`创建的请求对象都使用该插件，`req.Use()`只对该请求生效，在本例中使用`req.Use(body.JSON(data))`也是可以的。上面使用`body.JSON()`插件，每次发送请求时，都将`data`转为 JSON 设置到请求体中，并设置相应的首部（`Content-Type/Content-Length`）。`req.Method("POST")`设置使用 POST 方法。本次请求使用的 URL `http://httpbin.org/post`会回显请求的信息，看运行结果：

```golang
Status: 200
Body: {
  "args": {}, 
  "data": "{\"foo\":\"bar\"}\n", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept-Encoding": "gzip", 
    "Content-Length": "14", 
    "Content-Type": "application/json", 
    "Host": "httpbin.org", 
    "User-Agent": "gentleman/2.0.4", 
    "X-Amzn-Trace-Id": "Root=1-5e8dd0c7-ab423c10fb530deade846500"
  }, 
  "json": {
    "foo": "bar"
  }, 
  "origin": "124.77.254.163", 
  "url": "http://httpbin.org/post"
}
```

发送 XML 格式与上面的非常类似：

```golang
type User struct {
  Name string `xml:"name"`
  Age  int    `xml:"age"`
}

func main() {
  cli := gentleman.New()
  cli.URL("http://httpbin.org/post")

  req := cli.Request()
  req.Method("POST")

  u := User{Name: "dj", Age: 18}
  req.Use(body.XML(u))
  // ...
}
```

后半部分一样的代码我就省略了，运行结果：

```golang
Status: 200
Body: {
  "args": {}, 
  "data": "<User><name>dj</name><age>18</age></User>", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept-Encoding": "gzip", 
    "Content-Length": "41", 
    "Content-Type": "application/xml", 
    "Host": "httpbin.org", 
    "User-Agent": "gentleman/2.0.4", 
    "X-Amzn-Trace-Id": "Root=1-5e8dd339-830dba04536ceef247156746"
  }, 
  "json": null, 
  "origin": "222.64.16.70", 
  "url": "http://httpbin.org/post"
}
```

### `header`

`header`插件用于在发送请求前添加一些通用的首部，如 APIKey；或者删除一些自动加上的首部，如`User-Agent`。一般`header`插件应用在`cli`对象上：

```golang
package main

import (
  "fmt"
  "gopkg.in/h2non/gentleman.v2"
  "gopkg.in/h2non/gentleman.v2/plugins/headers"
)

func main() {
  cli := gentleman.New()
  cli.URL("https://api.thecatapi.com")

  cli.Use(headers.Set("x-api-key", "479ce48d-db30-46a4-b1a0-91ac4c1477b8"))
  cli.Use(headers.Del("User-Agent"))

  req := cli.Request()
  req.Path("/v1/breeds")
  res, err := req.Send()
  if err != nil {
    fmt.Printf("Request error: %s\n", err)
    return
  }
  if !res.Ok {
    fmt.Printf("Invalid server response: %d\n", res.StatusCode)
    return
  }

  fmt.Printf("Status: %d\n", res.StatusCode)
  fmt.Printf("Body: %s", res.String())
}
```

上面我们使用了`https://api.thecatapi.com`，这个 API 可以获取**猫**的品种信息，支持返回全部品种，搜索，分页等操作。API 使用需要申请 APIKey，我自己申请了一个`479ce48d-db30-46a4-b1a0-91ac4c1477b8`。`thecatapi`要求在请求首部中设置`x-api-key`为我们申请到的 APIKey。

`headers`可以很方便的实现这个功能，只需要在`cli`对象上设置一次即可。另外，`gentleman`会自动在请求中添加一个`User-Agent`首部，内容是`gentleman`的版本信息。细心的童鞋可能已经发现了，在上一节的输出中有`User-Agent: gentleman/2.0.4`这个首部。在本例中，我们使用`header.Del()`删除这个首部。

输出内容太多，我这里就不贴了。

### `query`

HTTP 请求通常会在 URL 的`?`后带上查询字符串（`query string`），`gentleman`的内置插件`query`可以很好的管理这个信息。我们可以基于上面代码，给请求带上参数`page`和`limit`使之分页返回：

```golang
package main

import (
  "fmt"

  "gopkg.in/h2non/gentleman.v2"
  "gopkg.in/h2non/gentleman.v2/plugins/headers"
  "gopkg.in/h2non/gentleman.v2/plugins/query"
)

func main() {
  cli := gentleman.New()
  cli.URL("https://api.thecatapi.com")

  cli.Use(headers.Set("x-api-key", "479ce48d-db30-46a4-b1a0-91ac4c1477b8"))
  cli.Use(query.Set("attach_breed", "beng"))
  cli.Use(query.Set("limit", "2"))
  cli.Use(headers.Del("User-Agent"))

  req := cli.Request()
  req.Path("/v1/breeds")
  req.Use(query.Set("page", "1"))
  res, err := req.Send()
  if err != nil {
    fmt.Printf("Request error: %s\n", err)
    return
  }
  if !res.Ok {
    fmt.Printf("Invalid server response: %d\n", res.StatusCode)
    return
  }

  fmt.Printf("Status: %d\n", res.StatusCode)
  fmt.Printf("Body: %s", res.String())
}
```

品种和每页显示数量最好还是在`cli`对象中设置，每个请求对象共用：

```golang
cli.Use(query.Set("attach_breed", "beng"))
cli.Use(query.Set("limit", "2"))
```

当前请求的页数在`req`对象上设置：

```golang
req.Use(query.Set("page", "1"))
```

其他的代码与上一个示例完全一样。除了设置`query string`，还可以通过`query.Del()`删除某个键值对。

### `url`

路径参数有些时候很有用，因为我们在开发中时常会碰到相似的路径，只是中间某个部分不一样，例如`/info/user/1`，`/info/book/1`等。重复写这些路径不仅很枯燥，而且容易出错。于是，偷懒的程序员发明了路径参数，形如`/info/:class/1`，我们可以传入参数`user`或`book`组成完整的路径。`gentleman`内置了插件`url`用来处理路径参数问题：

```golang
package main

import (
  "fmt"
  "os"

  "gopkg.in/h2non/gentleman.v2"
  "gopkg.in/h2non/gentleman.v2/plugins/headers"
  "gopkg.in/h2non/gentleman.v2/plugins/url"
)

func main() {
  cli := gentleman.New()
  cli.URL("https://api.thecatapi.com/")

  cli.Use(headers.Set("x-api-key", "479ce48d-db30-46a4-b1a0-91ac4c1477b8"))
  cli.Use(url.Path("/v1/:type"))

  for _, arg := range os.Args[1:] {
    req := cli.Request()
    req.Use(url.Param("type", arg))
    res, err := req.Send()
    if err != nil {
      fmt.Printf("Request error: %s\n", err)
      return
    }
    if !res.Ok {
      fmt.Printf("Invalid server response: %d\n", res.StatusCode)
      return
    }

    fmt.Printf("Status: %d\n", res.StatusCode)
    fmt.Printf("Body: %s\n", res.String())
  }
}
```

`thecatapi`除了可以获取猫的品种，还有用户投票、各种分类信息。它们的请求路径都差不多，`/v1/breeds`、`/v1/votes`、`/v1/categories`。我们使用`url`简化程序编写。上面程序在客户端对象`cli`上使用插件`url.Path("/v1/:type")`，调用`url.Param("type", arg)`用命令行中的参数分别替换`type`进行 HTTP 请求。运行程序：

```golang
$ go run main.go breeds votes categories
```

### 其他

`gentleman`内置了将近 20 个插件，有身份认证相关的`auth`、有`cookies`、有压缩相关的`compression`、有代理相关的`proxy`、有重定向相关的`redirect`、有超时相关的`timeout`、有重试的`retry`、有服务发现的`consul`等等等等。感兴趣可自行去探索。

### 自定义

如果内置的和第三方的插件都不能满足我们的需求，我们还可以自定义插件。自定义的插件需要实现下面的接口：

```golang
// src/gopkg.in/h2non/gentleman.v2/plugin/plugin.go
type Plugin interface {
  Enable()
  Disable()
  Disabled() bool
  Remove()
  Removed() bool
  Exec(string, *context.Context, context.Handler)
}
```

`Exec()`方法在 HTTP 请求的各个生命周期都会调用，可以在请求前添加一些首部、删除查询字符串，响应返回后进行一些处理等。

通过实现`Plugin`接口的方式实现插件比较繁琐，且很多插件往往只关注生命周期的某个点，不用处理所有的生命周期事件。`gentleman`提供了一个`Layer`结构，可以注册某个生命周期的方法，同时提供`NewRequestPlugin/NewResponsePlugin/NewErrorPlugin`等便捷函数。

我们现在来实现一个插件，在请求之前输出一行信息，收到响应之后输出一行信息：

```golang
package main

import (
  "fmt"

  "gopkg.in/h2non/gentleman.v2"
  c "gopkg.in/h2non/gentleman.v2/context"
  "gopkg.in/h2non/gentleman.v2/plugin"
)

func main() {
  cli := gentleman.New()
  cli.URL("https://httpbin.org")

  cli.Use(plugin.NewRequestPlugin(func(ctx *c.Context, h c.Handler) {
    fmt.Println("request")

    h.Next(ctx)
  }))

  cli.Use(plugin.NewResponsePlugin(func(ctx *c.Context, h c.Handler) {
    fmt.Println("response")

    h.Next(ctx)
  }))

  req := cli.Request()
  req.Path("/headers")
  res, err := req.Send()
  if err != nil {
    fmt.Printf("Request error: %s\n", err)
    return
  }
  if !res.Ok {
    fmt.Printf("Invalid server response: %d\n", res.StatusCode)
    return
  }

  fmt.Printf("Status: %d\n", res.StatusCode)
  fmt.Printf("Body: %s", res.String())
}
```

由于`NewRequestPlugin/NewResponsePlugin`这些便利函数，我们只需要实现一个类型为`func(ctx *c.Context, h c.Handler)`的函数即可，在`ctx`中有`Request`和`Response`等信息，可以在发起请求前对请求进行一些操作以及获得响应时对响应进行一些操作。上面只是简单地输出信息。

## 总结

使用`gentleman`可以实现灵活、便捷的 HTTP 客户端，它提供了丰富的插件，用起来吧~

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gentleman GitHub：[https://github.com/h2non/gentleman](https://github.com/h2non/gentleman)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)