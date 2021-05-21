---
layout:    post
title:    "Go 每日一库之 gotalk"
subtitle: 	"每天学习一个 Go 库"
date:		2021-05-20T07:25:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/05/20/godailylib/gotalk"
categories: [
	"Go"
]
---

## 简介

[`gotalk`](https://github.com/rsms/gotalk)专注于进程间的通信，致力于简化通信协议和流程。同时它：

* 提供简洁、清晰的 API；
* 支持 TCP，WebSocket 等协议；
* 采用非常简单而又高效的传输协议格式，便于抓包调试；
* 内置了 JavaScript 文件`gotalk.js`，方便开发基于 Web 网页的客户端程序；
* 内含丰富的示例可供学习参考。

那么，让我们来玩一下吧~

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir gotalk && cd gotalk
$ go mod init github.com/darjun/go-daily-lib/gotalk
```

安装`gotalk`库：

```cmd
$ go get -u github.com/rsms/gotalk
```

接下来让我们来编写一个简单的 echo 程序，服务端直接返回收到的客户端信息，不做任何处理。首先是服务端：

```golang
// get-started/server/server.go
package main

import (
  "log"

  "github.com/rsms/gotalk"
)

func main() {
  gotalk.Handle("echo", func(in string) (string, error) {
    return in, nil
  })
  if err := gotalk.Serve("tcp", ":8080", nil); err != nil {
    log.Fatal(err)
  }
}
```

通过`gotalk.Handle()`注册消息处理，它接受两个参数。第一个参数为消息名，字符串类型，保证唯一且可辨识即可。第二个参数为处理函数，收到对应名称的消息，调用该函数处理。处理函数接受一个参数，返回两个值。正常处理完成通过第一个返回值传递处理结果，出错时通过第二个返回值表示错误类型。

这里的处理器函数比较简单，接受一个字符串参数，直接原样返回。

然后，调用`gotalk.Serve()`启动服务器，监听端口。它接受 3 个参数，协议类型、监听地址、处理器对象。此处我们使用 TCP 协议，监听本地`8080`端口，使用默认处理器对象，传入`nil`即可。

服务器内部一直循环处理请求。

然后是客户端：

```golang
func main() {
  s, err := gotalk.Connect("tcp", ":8080")
  if err != nil {
    log.Fatal(err)
  }

  for i := 0; i < 5; i++ {
    var echo string
    if err := s.Request("echo", "hello", &echo); err != nil {
      log.Fatal(err)
    }

    fmt.Println(echo)
  }

  s.Close()
}
```

客户端首先调用`gotalk.Connect()`连接服务器，它接受两个参数：协议和地址（IP + 端口）。我们使用与服务器一致的协议和地址即可。连接成功会返回一个连接对象。调用连接对象的`Request()`方法，即可向服务器发送消息。`Request()`方法接受 3 个参数。第一个参数为消息名，这对应于服务器注册的消息名，请求一个不存在的消息名会返回错误。第二个参数是传给服务器的参数，有且只能有一个参数，对应处理器函数的入参。第三个参数为返回值的指针，用于接受服务器返回的结果。

如果请求失败，返回错误`err`。使用完成之后不要忘记关闭连接对象。

先运行服务器：

```cmd
$ go run server.go
```

在开启一个命令行，运行客户端：

```golang
$ go run client.go
hello
hello
hello
hello
hello
```

实际上如果了解标准库`net/http`，你应该就会发现，使用`gotalk`的服务端代码与使用`net/http`编写 Web 服务器非常相似。都非常简单，清晰：

```golang
// get-started/http/main.go
package main

import (
  "fmt"
  "log"
  "net/http"
)

func index(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, "hello world")
}

func main() {
  http.HandleFunc("/", index)

  if err := http.ListenAndServe(":8888", nil); err != nil {
    log.Fatal(err)
  }
}
```

运行：

```cmd
$ go run main.go

```

使用 curl 验证：

```cmd
$ curl localhost:8888
hello world
```

## WebSocket

除了 TCP，`gotalk`还支持基于 WebSocket 协议的通信。下面我们使用  WebSocket 重写上面的服务端程序，然后编写一个简单 Web 页面与之通信。

服务端：

```golang
func main() {
  gotalk.Handle("echo", func(in string) (string, error) {
    return in, nil
  })

  http.Handle("/gotalk/", gotalk.WebSocketHandler())
  http.Handle("/", http.FileServer(http.Dir(".")))
  if err := http.ListenAndServe(":8080", nil); err != nil {
    log.Fatal(err)
  }
}
```

`gotalk`消息处理函数的注册还是与前面的一样。不同的是这里将 HTTP 路径`/gotalk/`的请求交由`gotalk.WebSocketHandler()`处理，这个处理器负责 WebSocket 请求。同时，在当前工作目录开启一个文件服务器，挂载到 HTTP 路径`/`上。文件服务器是为了客户端方便地请求`index.html`页面。最后调用`http.ListenAndServe()`开启 Web 服务器，监听端口 8080。

然后是客户端，`gotalk`为了方便 Web 程序的编写，将 WebSocket 通信细节封装在一个 JavaScript 文件`gotalk.js`中。可以直接从仓库中的 js 目录下获取使用。接着我们编写页面`index.html`，引入`gotalk.js`：

```html
<!DOCTYPE HTML>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <script type="text/javascript" src="gotalk/gotalk.js"></script>
  </head>
  <body>
    <input id="txt">
    <button id="snd">send</button><br>
    <script>
    let c = gotalk.connection()
      .on('open', () => log(`connection opened`))
      .on('close', reason => log(`connection closed (reason: ${reason})`))
    let btn = document.querySelector("#snd")
    let txt = document.querySelector("#txt")
    btn.onclick = async () => {
      let content = txt.value
      if (content.length === 0) {
        alert("no message")
        return
      }
      let res = await c.requestp('echo', content)
      log(`reply: ${JSON.stringify(res, null, 2)}`)
      return false
    }
    function log(message) {
      document.body.appendChild(document.createTextNode(message))
      document.body.appendChild(document.createElement("br"))
    }
    </script>
  </body>
</html>
```

首先调用`gotalk.connection()`连接服务端，返回一个连接对象。调用此对象的`on()`方法，分别注册连接建立和断开的回调。然后给按钮添加回调，每次点击将输入框中的内容发送给服务端。调用连接对象的`requestp()`方法发送请求，第一个参数为消息名，对应在服务端使用`gotalk.Handle()`注册的名字。第二个即为处理参数，会一并发送给服务端。这里使用 Promise 处理异步请求和响应，为了编写方便和易于理解使用`async-await`同步的写法。响应的内容直接显示在页面上：

![](/img/in-post/godailylib/gotalk1.png#center)

注意，`gotalk.js`文件需要放在服务器运行目录的`gotalk`目录下。

## 协议格式

`gotalk`采用基于 ASCII 的协议格式，设计为方便人类阅读且灵活的。每条传输的消息都分为几个部分：类型标识、请求ID、操作、消息内容。

* 类型标识：只用一个字节，用来表示消息的类型，是请求消息还是响应消息，流式消息还是非流式的，错误、心跳和通知也都有其特定的类型标识。
* 请求 ID：用 4 个字节表示，方便匹配响应。由于`gotalk`可以同时发送任意个请求并接收之前请求的响应。所以需要有一个 ID 来标识接收到的响应对应之前发送的哪条请求。
* 操作：即为我们上面定义的消息名，例如"echo"。
* 消息内容：使用长度 + 实际内容格式。

看一个官方请求的示例：

```
+------------------ SingleRequest
|   +---------------- requestID   "0001"
|   |      +--------- operation   "echo" (text3Size 4, text3Value "echo")
|   |      |       +- payloadSize 25
|   |      |       |
r0001004echo00000019{"message":"Hello World"}
```

* `r`：表示这是一个单条请求。
* `0001`：请求 ID 为 1，这里采用十六进制编码。
* `004echo`：这部分表示操作为"echo"，在实际字符串内容前需要指定长度，否则接收方不知道内容在哪里结束。`004`指示"echo"长度为 4，同样采用十六进制编码。
* `00000019{"message":"Hello World"}`：这部分是消息的内容。同样需要指定长度，十六进制`00000019`表示长度为 25。

详细格式可以查看官方文档。

使用这种可阅读的格式给问题排查带来了极大的便利。但是在实际使用中，可能需要考虑安全和隐私的问题。

## 聊天室

`examples`内置一个基于 WebSocket 的聊天室示例程序。特性如下：

* 可以创建房间，默认创建 3 个房间`animals/jokes/golang`；
* 在房间聊天（基本功能）；
* 一个简单的 Web 页面。

运行：

```golang
$ go run server.go
```

打开浏览器，输入"localhost:1235"，显示如下：

![](/img/in-post/go-daily-lib/gotalk2.png#center)


接下来就可以创建房间，在房间聊天了。

整个实现的有几个要点：

其一，`gotalk.WebSocketHandler()`创建的 WebSocket 处理器可以设置连接回调：

```golang
gh := gotalk.WebSocketHandler()
gh.OnConnect = onConnect
```

在回调中设置随机用户名，并将当前连接的`gotalk.Sock`存储下来，方便消息广播：

```golang
func onConnect(s *gotalk.WebSocket) {
  socksmu.Lock()
  defer socksmu.Unlock()
  socks[s] = 1

  username := randomName()
  s.UserData = username
}
```

其二，`gotalk`设置处理器函数可以有两个参数，第一个表示当前连接，第二个才是实际接收到的消息参数。

其三，`enableGracefulShutdown()`函数实现了 Web 服务器的优雅关闭，非常值得学习。接收到`SIGINT`信号，先关闭所有的连接，再退出程序。注意监听信号和运行 HTTP 服务器并不是同一个 goroutine，看它们是如何协作的：

```golang
func enableGracefulShutdown(server *http.Server, timeout time.Duration) chan struct{} {
  server.RegisterOnShutdown(func() {
    // close all connected sockets
    fmt.Printf("graceful shutdown: closing sockets\n")
    socksmu.RLock()
    defer socksmu.RUnlock()
    for s := range socks {
      s.CloseHandler = nil // avoid deadlock on socksmu (also not needed)
      s.Close()
    }
  })
  done := make(chan struct{})
  quit := make(chan os.Signal, 1)
  signal.Notify(quit, syscall.SIGINT)
  go func() {
    <-quit // wait for signal

    fmt.Printf("graceful shutdown initiated\n")
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    server.SetKeepAlivesEnabled(false)
    if err := server.Shutdown(ctx); err != nil {
      fmt.Printf("server.Shutdown error: %s\n", err)
    }

    fmt.Printf("graceful shutdown complete\n")
    close(done)
  }()
  return done
}
```

接收到`SIGINT`信号后`done`通道关闭，`server.ListenAndServe()`返回`http.ErrServerClosed`错误，退出循环：

```golang
done := enableGracefulShutdown(server, 5*time.Second)

// Start server
fmt.Printf("Listening on http://%s/\n", server.Addr)
if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
  panic(err)
}

<- done
```

整个聊天室功能比较简单，代码也比较短，建议深入理解。在此基础之上做扩展也比较简单。

## 总结

`gotalk`实现了一个简单、易用的通信库。并且提供了 JavaScript 文件`gotalk.js`，方便 Web 程序的开发。协议格式清晰，易调试。内置丰富的示例。整个库的代码也不长，建议深入了解。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gotalk GitHub：[https://github.com/rsms/gotalk](https://github.com/rsms/gotalk)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~