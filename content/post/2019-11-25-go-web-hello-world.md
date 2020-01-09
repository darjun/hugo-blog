---
layout:		post
title:		"Go Web 编程之 Hello World"
subtitle: 	"入门 Go Web 编程"
date:		2019-11-25T21:30:00
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go Web 编程
URL: "2019/11/25/goweb/hello-world"
categories: [
	"Go"
]
---

## 概述

计划写一个讲 Go Web 编程的系列文章。从基于 net/http 包编写 Go Web 程序开始，讲述处理器，请求，响应等基础知识。然后到框架的使用。中间会穿插一些源码的分析。最后做一个实战项目。

目前 Go 社区已经有非常多关于 Web 开发的库或框架。大而全的有[beego](https://github.com/astaxie/beego)，[revel](https://github.com/revel/revel)。超高性能的有[echo](https://github.com/labstack/echo)，[fasthttp](https://github.com/valyala/fasthttp)，[gin](https://github.com/gin-gonic/gin)（目前 GitHub 星标最多）。还有不少专注于具体某个方面的，最多要属路由了，例如：[mux](https://github.com/gorilla/mux)/[httprouter](https://github.com/julienschmidt/httprouter)。

那为什么还要从最原始的 net/http 包开始学起？因为这些库/框架大多是基于 net/http 包做了包装，提供易于使用的功能，如路由参数（`/:name/:age`）/路由分组等。熟练掌握了基础知识和 net/http，学习其他框架必然能有事半功倍的效果。不管是快速上手使用库和框架，还是深入阅读源码，都能得心应手。

## HTTP

HTTP 协议是整个互联网的基石。不管技术，产品还是运营，甚至是非互联网行业的人，每天都在与 HTTP 协议打交道。我们每天浏览网页都在使用 HTTP。现在很多 APP 也都在内部使用 HTTP 与服务器交互。所以学习 Web 编程，HTTP 协议是必须要掌握的。

HTTP 是一个无状态的，基于文本的协议。它灵活，稳定，强大。自 1991 年发布以来只进行了几次修订。下面是 HTTP 发展简史：

* 1991 年，HTTP 的第一个版本 0.9 由 Tim Berners-Lee 创建。最初只有一个方法 GET，而且规定服务器返回的只能是 HTML 格式的数据。
* 1996 年，HTTP 1.0 发布，支持了 POST 和 HEAD 方法。
* 1999 年，HTTP 1.1 发布，添加了 PUT/DELETE/OPTIONS/TRACE/CONNECT 这 5 个方法，**并允许开发者自行添加更多方法**。
* 2015 年，HTTP 2.0 发布，为提升性能做出了不少修改，如采用二进制格式，完全多路复用。

HTTP 是一种请求——响应模式的协议，所有操作以一个请求开始，以一个响应结束。

### HTTP 请求

HTTP 请求的格式非常简单。一个请求由以下 4 个部分组成：

* 请求行（request-line）；
* 零个或多个首部（header）；
* 一个空行；
* 一个可选的报文主体（body）。

第一个重要的部分为请求行，其格式如下：

```
Method Path Version
```

* `Method`：请求的方法，表示对资源进行的操作。常用的方法有`GET/POST/PUT/DELETE`；
* `Path`：请求资源的路径，如`/user/info.html`；
* `Version`：即 HTTP 的版本号，1.1 版本写做`HTTP/1.1`。

第二个部分为请求的首部，每个首部占一行。首部使用由冒号（`:`）分隔的键值对表示，如`Content-Type: x-www-form-urlencoded`。

第三个部分为一个空行。注意，**即使没有首部，后面的空行也不能省略**。

最后为一个可选的报文主体。如果有主体，服务器会根据首部中`Content-Type`指定的格式来解析这部分内容。

### HTTP 响应

HTTP 响应的格式与请求非常相似。一个响应由以下 4 个部分组成：

* 响应行（response line）；
* 零个或多个首部（header）；
* 一个空行；
* 一个可选的报文主体（body）。

响应行的格式为：

```
StatusCode Description
```

* `StatusCode`：状态码，表示请求状态；
* `Description`：对状态码的简短描述。

响应的首部与主体和请求的一样，这里就不多说了。 

这里简单介绍一下状态码。HTTP 将状态码分为了 5 大类，`1XX/2XX/3XX/4XX/5XX`。

* `1XX`：情报状态码，又叫做信息状态码。服务器通过这类状态码告知客户端，自己已经收到了客户端发送的请求。几个常见的状态码如下：
    * `100 Continue`：表示服务器到目前为止收到的内容都正常，客户端应该继续请求。如果已经完成，则忽略它。
    * `101 Switching Protocol`：这个状态码是响应客户端的`Upgrade`首部发送的，并且指示服务器也正在切换的协议。如客户端请求切换协议，服务器将协议切换至 Websocket，就会发送该状态码给客户端，并且在`Upgrade`首部中填上 Websocket。

* `2XX`：成功状态码。表示服务器已经收到了客户端的请求，并成功对请求进行了处理。几个常见的状态码如下：
    * `200 OK`：这最常见的状态码了，表示请求成功。
    * `201 Created`：请求已成功，并因此创建了一个新的资源。

* `3XX`：重定向状态码。服务器收到了请求，但是为了完整地处理该请求，客户端还需要执行指定的动作。一般用于 URL 重定向。几个常见的状态码如下：
    * `300 Multiple Choice`：被请求的资源有一系列可供选择的回馈信息，每个都有自己特定的地址。用户或浏览器可自行选择一个地址进行重定向。
    * `302 Moved Permanently`：被请求的资源已经永久移动到新的位置了。

* `4XX`：客户端错误状态码。表示客户端发送的请求中有错误，如格式不对。常见的状态码如下：
    * `404 Not Found`：最常见的状态码了，表示页面不存在。
    * `405 Method Not Allowed`：请求的方法不允许。

* `5XX`：服务器错误状态码。表示服务器由于某些原因无法正确处理请求。常见的状态码如下：
    * `500 Internal Server Error`：服务器遇到了不知道如何处理的情况。
    * `501 Not Implemented`：此请求方法不被服务器支持。

## 第一个 Go Web 程序

> Talk is cheap, show me the Code!

接下来，我们来编写一个 Web 版本的 "Hello World" 程序。我们将使用 Go 语言提供的 net/http 包。该包的功能十分强大，使用起来也非常方便。

* 首先，在`$GOPATH`目录下创建一个项目目录`go-web-example`；
* 在`go-web-example`目录下创建一个`1-hello_world`程序目录；
* 创建`server.go`文件，输入下面内容：

```golang
package main

import (
  "fmt"
  "net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "Hello, World!")
}

func main() {
    http.HandleFunc("/", hello)
    http.ListenAndServe(":8080", nil)
}
```

* 打开命令行，进入`$GOPATH/go-web-example/1-hello_world`目录，输入命令：`go run server.go`，我们的第一个服务器程序就跑起来了。

![](/img/in-post/goweb/helloworld1.png)

* 打开浏览器，输入网址`localhost:8080`，"Hello, World"就在网页上显示出来了！

![](/img/in-post/goweb/helloworld3.png)

我们来解析一下该程序。

`http.HandleFunc`将`hello`函数注册到根路径`/`上，`hello`函数我们也叫做处理器。它接收两个参数，第一个参数为一个类型为`http.ResponseWriter`的接口，响应就是通过它发送给客户端的。第二个参数是一个类型为`http.Request`的结构指针，客户端发送的信息都可以通过这个结构获取。

然后，`http.ListenAndServe`将在 8080 端口上监听请求，然后交由`hello`处理。

由于 net/http 包为我们封装了很多细节，所以我们的使用如此简单。

## 总结

本文简单介绍了 HTTP 的发展简史、HTTP 请求和响应的格式，并且编写了第一个 Go Web 程序。作为整个互联网的基石，HTTP 协议的重要性怎么形容都不为过，是每个开发人员都必须掌握的知识。Go 语言的 net/http 为 Web 程序的开发封装了很多细节。使用它来开发 Web 程序非常简单。最后，为了能加深学习的印象，我画了一个脑图。期望学完之后能形成一个完整的知识体系。

![](/img/in-post/goweb/helloworld4.png)

接下来，我们来深入学习 HTTP 请求的内容。

## 参考

1. [Go Web 编程](https://book.douban.com/subject/27204133/)
2. [HTTP 响应码](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)