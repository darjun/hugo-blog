---
layout:    post
title:    "Go 每日一库之 rpc"
subtitle: 	"每天学习一个 Go 库"
date:		2020-05-08T21:12:22
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/05/08/godailylib/rpc"
categories: [
	"Go"
]
---

## 简介

RPC（Remote Procedure Call）是远程方法调用的缩写，它可以通过网络调用远程对象的方法。Go 标准库`net/rpc`提供了一个**简单、强大且高性能**的 RPC 实现。仅需编写很少的代码就能实现 RPC 服务。本文就来介绍一下这个库。

## 快速使用

标准库无需安装。

由于是网络程序，我们需要编写服务端和客户端两个程序。首先是服务端程序：

```golang
package main

import (
  "errors"
  "log"
  "net"
  "net/http"
  "net/rpc"
)

type Args struct {
  A, B int
}

type Quotient struct {
  Quo, Rem int
}

type Arith int

func (t *Arith) Multiply(args *Args, reply *int) error {
  *reply = args.A * args.B
  return nil
}

func (t *Arith) Divide(args *Args, quo *Quotient) error {
  if args.B == 0 {
    return errors.New("divide by 0")
  }

  quo.Quo = args.A / args.B
  quo.Rem = args.A % args.B
  return nil
}

func main() {
  arith := new(Arith)
  rpc.Register(arith)
  rpc.HandleHTTP()
  if err := http.ListenAndServe(":1234", nil); err != nil {
    log.Fatal("serve error:", err)
  }
}
```

我们定义了一个`Arith`类型，为它编写了两个方法`Multiply`和`Divide`。创建`Arith`类型的对象`arith`，调用`rpc.Register(arith)`会注册这两个方法。`rpc`库对注册的方法有一定的限制，方法必须满足签名`func (t *T) MethodName(argType T1, replyType *T2) error`：

* 首先，方法必须是导出的（名字首字母大写）；
* 其次，方法接受两个参数，必须是导出的或内置类型。第一个参数表示客户端传递过来的请求参数，第二个是需要返回给客户端的响应。第二个参数必须为指针类型（需要修改）；
* 最后，方法必须返回一个`error`类型的值。返回非`nil`的值，表示调用出错。

`rpc.HandleHTTP()`注册 HTTP 路由。`http.ListenAndServe(":1234", nil)`在端口`1234`上启动一个 HTTP 服务，请求 rpc 方法会交给`rpc`内部路由处理。这样我们就可以通过客户端调用这两个方法了：

```golang
package main

import (
  "fmt"
  "log"
  "net/rpc"
)

type Args struct {
  A, B int
}

type Quotient struct {
  Quo, Rem int
}

func main() {
  client, err := rpc.DialHTTP("tcp", ":1234")
  if err != nil {
    log.Fatal("dialing:", err)
  }

  args := &Args{7, 8}
  var reply int
  err = client.Call("Arith.Multiply", args, &reply)
  if err != nil {
    log.Fatal("Multiply error:", err)
  }
  fmt.Printf("Multiply: %d*%d=%d\n", args.A, args.B, reply)

  args = &Args{15, 6}
  var quo Quotient
  err = client.Call("Arith.Divide", args, &quo)
  if err != nil {
    log.Fatal("Divide error:", err)
  }
  fmt.Printf("Divide: %d/%d=%d...%d\n", args.A, args.B, quo.Quo, quo.Rem)
}
```

客户端比服务端稍微简单一点，我们使用`rpc.DialHTTP("tcp", ":1234")`连接到服务端的监听地址，返回一个 rpc 的客户端对象。后续就可以调用该对象的`Call()`方法调用服务端对象的对应方法，依次传入方法名（需要加上类型限定）、参数、一个指针（用于接收返回值）。首先运行服务端程序：

```golang
$ go run main.go
```

然后在一个新的控制台中运行客户端程序，输出：

```golang
$ go run client.go
Multiply: 7*8=56
Divide: 15/6=2...3
```

对`net/http`包不熟悉的童鞋可能会觉得奇怪，`rpc.HandleHTTP()`与`http.ListenAndServer(":1234", nil)`是怎么联系起来的？我们简单看一下源码：

```golang
// src/net/rpc/server.go
const (
  // Defaults used by HandleHTTP
  DefaultRPCPath   = "/_goRPC_"
  DefaultDebugPath = "/debug/rpc"
)

func (server *Server) HandleHTTP(rpcPath, debugPath string) {
  http.Handle(rpcPath, server)
  http.Handle(debugPath, debugHTTP{server})
}

func HandleHTTP() {
  DefaultServer.HandleHTTP(DefaultRPCPath, DefaultDebugPath)
}
```

实际上，`rpc.HandleHTTP()`会调用`http.Handle()`在预定义的路径上（`/_goRPC_`）注册处理器。这个处理器最终被添加到`net/http`包中的默认多路复用器上：

```golang
// src/net/http/server.go
func Handle(pattern string, handler Handler) {
  DefaultServeMux.Handle(pattern, handler)
}
```

而`http.ListenAndServer()`第二个参数传入`nil`时也是使用默认的多路复用器。具体可以看看我之前的文章[Go Web 编程之 程序结构](https://darjun.github.io/2019/12/05/goweb/structure/)。

细心的朋友可能发现了，除了默认的路径`/_goRPC_`用来处理 RPC 请求，`rpc.HandleHTTP()`方法还注册了一个调试路径`/debug/rpc`。我们可以直接在浏览器中访问这个网址（需要服务端程序开启。如果服务端在远程，需要相应地修改地址）[localhost:1234](http://localhost:1234/debug/prc)，直观的查看各个方法的调用情况：

![](/img/in-post/godailylib/rpc1.png#center)

## 异步调用

上面的例子中，我们在客户端使用了同步的调用方式，即一直等待服务端的响应或出错。在等待的过程中，客户端就不能处理其它的任务了。当然，我们也可以采用异步的调用方式：

```golang
func main() {
  client, err := rpc.DialHTTP("tcp", ":1234")
  if err != nil {
    log.Fatal("dialing:", err)
  }

  args1 := &Args{7, 8}
  var reply int
  multiplyReply := client.Go("Arith.Multiply", args1, &reply, nil)

  args2 := &Args{15, 6}
  var quo Quotient
  divideReply := client.Go("Arith.Divide", args2, &quo, nil)

  ticker := time.NewTicker(time.Millisecond)
  defer ticker.Stop()

  var multiplyReplied, divideReplied bool
  for !multiplyReplied || !divideReplied {
    select {
    case replyCall := <-multiplyReply.Done:
      if err := replyCall.Error; err != nil {
        fmt.Println("Multiply error:", err)
      } else {
        fmt.Printf("Multiply: %d*%d=%d\n", args1.A, args1.B, reply)
      }
      multiplyReplied = true
    case replyCall := <-divideReply.Done:
      if err := replyCall.Error; err != nil {
        fmt.Println("Divide error:", err)
      } else {
        fmt.Printf("Divide: %d/%d=%d...%d\n", args2.A, args2.B, quo.Quo, quo.Rem)
      }
      divideReplied = true
    case <-ticker.C:
      fmt.Println("tick")
    }
  }
}
```

异步调用使用`client.Go()`方法，参数与同步调用基本一样。它返回一个`rpc.Call`对象：

```golang
// src/net/rpc/client.go
type Call struct {
  ServiceMethod string     
  Args          interface{}
  Reply         interface{}
  Error         error      
  Done          chan *Call 
}
```

我们可以通过该对象获取此次调用的信息，如方法名、参数、返回值和错误。我们通过监听通道`Done`是否有值判断调用是否完成。上面代码中使用一个`select`语句轮询两次调用的状态。注意一点，**如果多个通道都有值，`select`执行哪个`case`是随机的**。所以可能先输出`divide`的信息：

```golang
$ go run client.go 
Divide: 15/6=2...3
Multiply: 7*8=56
```

服务端可以继续使用一开始的。

## 定制方法名

默认情况下，`rpc.Register()`将方法接收者（`receiver`）的类型名作为方法名前缀。我们也可以自己设置。这时需要调用`RegisterName(name string, rcvr interface{}) error`方法：

```golang
func main() {
  arith := new(Arith)
  rpc.RegisterName("math", arith)
  rpc.HandleHTTP()
  if err := http.ListenAndServe(":1234", nil); err != nil {
    log.Fatal("serve error:", err)
  }
}
```

上面我们将注册的方法名前缀改为`math`了，客户端调用时传入的方法名也需要相应的修改：

```golang
func main() {
  client, err := rpc.DialHTTP("tcp", ":1234")
  if err != nil {
    log.Fatal("dialing:", err)
  }

  args := &Args{7, 8}
  var reply int
  err = client.Call("math.Multiply", args, &reply)
  if err != nil {
    log.Fatal("Multiply error:", err)
  }
  fmt.Printf("Multiply: %d*%d=%d\n", args.A, args.B, reply)
}
```

## TCP

上面我们都是使用 HTTP 协议来实现 rpc 服务的，`rpc`库也支持直接使用 TCP 协议。首先，服务端先调用`net.Listen("tcp", ":1234")`创建一个监听某个 TCP 端口的监听器（Accepter），然后使用`rpc.Accept(l)`在此监听器上接受连接并处理：

```golang
func main() {
  l, err := net.Listen("tcp", ":1234")
  if err != nil {
    log.Fatal("listen error:", err)
  }

  arith := new(Arith)
  rpc.Register(arith)
  rpc.Accept(l)
}
```

然后，客户端调用`rpc.Dial()`以 TCP 协议连接到服务端：

```golang
func main() {
  client, err := rpc.Dial("tcp", ":1234")
  if err != nil {
    log.Fatal("dialing:", err)
  }

  args := &Args{7, 8}
  var reply int
  err = client.Call("Arith.Multiply", args, &reply)
  if err != nil {
    log.Fatal("Multiply error:", err)
  }
  fmt.Printf("Multiply: %d*%d=%d\n", args.A, args.B, reply)
}
```

## 自己接收连接

我们可以自己接受连接，然后在此连接上应用 rpc 协议：

```golang
func main() {
  l, err := net.Listen("tcp", ":1234")
  if err != nil {
    log.Fatal("listen error:", err)
  }

  arith := new(Arith)
  rpc.Register(arith)

  for {
    conn, err := l.Accept()
    if err != nil {
      log.Fatal("accept error:", err)
    }

    go rpc.ServeConn(conn)
  }
}
```

这个客户端与上面 TCP 的客户端一样，不用修改。

## 自定义编码格式

默认客户端与服务端之间的数据使用`gob`编码，我们可以使用其它的格式来编码。在服务端，我们要实现`rpc.ServerCodec`接口：

```golang
// src/net/rpc/server.go
type ServerCodec interface {
  ReadRequestHeader(*Request) error
  ReadRequestBody(interface{}) error
  WriteResponse(*Response, interface{}) error

  Close() error
}
```

实际上不用这么麻烦，我们查看源码看看`gobServerCodec`是怎么实现的，然后仿造实现一个就行了。下面我实现了一个 JSON 格式的编解码器：

```golang
type JsonServerCodec struct {
  rwc    io.ReadWriteCloser
  dec    *json.Decoder
  enc    *json.Encoder
  encBuf *bufio.Writer
  closed bool
}

func NewJsonServerCodec(conn io.ReadWriteCloser) *JsonServerCodec {
  buf := bufio.NewWriter(conn)
  return &JsonServerCodec{conn, json.NewDecoder(conn), json.NewEncoder(buf), buf, false}
}

func (c *JsonServerCodec) ReadRequestHeader(r *rpc.Request) error {
  return c.dec.Decode(r)
}

func (c *JsonServerCodec) ReadRequestBody(body interface{}) error {
  return c.dec.Decode(body)
}

func (c *JsonServerCodec) WriteResponse(r *rpc.Response, body interface{}) (err error) {
  if err = c.enc.Encode(r); err != nil {
    if c.encBuf.Flush() == nil {
      log.Println("rpc: json error encoding response:", err)
      c.Close()
    }
    return
  }
  if err = c.enc.Encode(body); err != nil {
    if c.encBuf.Flush() == nil {
      log.Println("rpc: json error encoding body:", err)
      c.Close()
    }
    return
  }
  return c.encBuf.Flush()
}

func (c *JsonServerCodec) Close() error {
  if c.closed {
    return nil
  }
  c.closed = true
  return c.rwc.Close()
}

func main() {
  l, err := net.Listen("tcp", ":1234")
  if err != nil {
    log.Fatal("listen error:", err)
  }

  arith := new(Arith)
  rpc.Register(arith)

  for {
    conn, err := l.Accept()
    if err != nil {
      log.Fatal("accept error:", err)
    }

    go rpc.ServeCodec(NewJsonServerCodec(conn))
  }
}
```

在`for`循环中需要创建编解码器`JsonServerCodec`传给`ServeCodec`方法。同样的，客户端要实现`rpc.ClientCodec`接口，也是仿造`gobClientCodec`的实现：

```golang
type JsonClientCodec struct {
  rwc    io.ReadWriteCloser
  dec    *json.Decoder
  enc    *json.Encoder
  encBuf *bufio.Writer
}

func NewJsonClientCodec(conn io.ReadWriteCloser) *JsonClientCodec {
  encBuf := bufio.NewWriter(conn)
  return &JsonClientCodec{conn, json.NewDecoder(conn), json.NewEncoder(encBuf), encBuf}
}

func (c *JsonClientCodec) WriteRequest(r *rpc.Request, body interface{}) (err error) {
  if err = c.enc.Encode(r); err != nil {
    return
  }
  if err = c.enc.Encode(body); err != nil {
    return
  }
  return c.encBuf.Flush()
}

func (c *JsonClientCodec) ReadResponseHeader(r *rpc.Response) error {
  return c.dec.Decode(r)
}

func (c *JsonClientCodec) ReadResponseBody(body interface{}) error {
  return c.dec.Decode(body)
}

func (c *JsonClientCodec) Close() error {
  return c.rwc.Close()
}

func main() {
  conn, err := net.Dial("tcp", ":1234")
  if err != nil {
    log.Fatal("dial error:", err)
  }

  client := rpc.NewClientWithCodec(NewJsonClientCodec(conn))

  args := &Args{7, 8}
  var reply int
  err = client.Call("Arith.Multiply", args, &reply)
  if err != nil {
    log.Fatal("Multiply error:", err)
  }
  fmt.Printf("Multiply: %d*%d=%d\n", args.A, args.B, reply)
}
```

要使用`NewClientWithCodec`以指定的编解码器创建客户端。

## 自定义服务器

实际上，上面我们调用的方法`rpc.Register`，`rpc.RegisterName`，`rpc.ServeConn`，`rpc.ServeCodec`都是转而去调用默认`DefaultServer`的相关方法：

```golang
// src/net/rpc/server.go
var DefaultServer = NewServer()

func Register(rcvr interface{}) error { return DefaultServer.Register(rcvr) }

func RegisterName(name string, rcvr interface{}) error {
  return DefaultServer.RegisterName(name, rcvr)
}

func ServeConn(conn io.ReadWriteCloser) {
  DefaultServer.ServeConn(conn)
}

func ServeCodec(codec ServerCodec) {
  DefaultServer.ServeCodec(codec)
}
```

但是因为`DefaultServer`是全局共享的，如果有第三方库使用了相关方法，并且注册了一些对象的方法，我们引用这个第三方库之后，就出现两个问题。第一，可能与我们注册的方法冲突；第二，带来额外的安全隐患（库中方法直接`panic`？）。故而推荐做法是自己`NewServer`：

```golang
func main() {
  arith := new(Arith)
  server := rpc.NewServer()
  server.RegisterName("math", arith)
  server.HandleHTTP(rpc.DefaultRPCPath, rpc.DefaultDebugPath)

  if err := http.ListenAndServe(":1234", nil); err != nil {
    log.Fatal("serve error:", err)
  }
}
```

这其实是一个套路，很多库会提供一个默认的实现直接使用，如`log`、`net/http`这些库。但是也提供了创建和自定义的方法。一般测试时为了方便可以使用默认实现，实践中最好自己创建相应的对象，避免干扰和安全问题。

## 总结

本文介绍了 Go 标准库中的`rpc`，它使用非常简单，性能异常强大。很多`rpc`的第三方库都是对`rpc`的封装，早期版本的[`rpcx`](https://rpcx.io/)就是基于`rpc`做的封装。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. rpc 官方文档[https://golang.org/pkg/net/rpc/](https://golang.org/pkg/net/rpc/)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)