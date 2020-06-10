---
layout:    post
title:    "Go 每日一库之 twirp"
subtitle: 	"每天学习一个 Go 库"
date:		2020-06-07T17:18:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/06/07/godailylib/twirp"
categories: [
	"Go"
]
---

## 简介

[twirp](https://github.com/twitchtv/twirp)是一个基于 Google Protobuf 的 RPC 框架。`twirp`通过在`.proto`文件中定义服务，然后自动生产服务器和客户端的代码。让我们可以将更多的精力放在业务逻辑上。咦？这不就是 gRPC 吗？gRPC 自己实现了一套 HTTP 服务器和网络传输层，twirp 使用标准库`net/http`。另外 gRPC 只支持 HTTP/2 协议，twirp 还可以运行在 HTTP 1.1 之上。同时 twirp 还可以使用 JSON 格式交互。当然并不是说 twirp 比 gRPC 好，只是多了解一种框架也就多了一个选择😊

## 快速使用

首先需要安装 twirp 的代码生成插件：

```golang
$ go get github.com/twitchtv/twirp/protoc-gen-twirp
```

上面命令会在`$GOPATH/bin`目录下生成可执行程序`protoc-gen-twirp`。我的习惯是将`$GOPATH/bin`放到 PATH 中，所以可在任何地方执行该命令。

接下来安装 protobuf 编译器，直接到 GitHub 上[https://github.com/protocolbuffers/protobuf/releases](https://github.com/protocolbuffers/protobuf/releases)下载编译好的二进制程序放到 PATH 目录即可。

最后是 Go 语言的 protobuf 生成插件：

```golang
$ go get github.com/golang/protobuf/protoc-gen-go
```

同样地，命令`protoc-gen-go`会安装到`$GOPATH/bin`目录中。

本文代码采用**Go Modules**。先创建目录，然后初始化：

```cmd
$ mkdir twirp && cd twirp
$ go mod init github.com/darjun/go-daily-lib/twirp
```

接下来，我们开始代码编写。先编写`.proto`文件：

```proto
syntax = "proto3";
option go_package = "proto";

service Echo {
  rpc Say(Request) returns (Response);
}

message Request {
  string text = 1;
}

message Response {
  string text = 2;
}
```

我们定义一个`service`实现**echo**功能，即发送什么就返回什么。切换到`echo.proto`所在目录，使用`protoc`命令生成代码：

```cmd
$ protoc --twirp_out=. --go_out=. ./echo.proto
```

上面命令会生成`echo.pb.go`和`echo.twirp.go`两个文件。前一个是 Go Protobuf 文件，后一个文件中包含了`twirp`相关组件。

然后我们就可以编写服务器和客户端程序了。服务器：

```golang
package main

import (
  "context"
  "net/http"

  "github.com/darjun/go-daily-lib/twirp/get-started/proto"
)

type Server struct{}

func (s *Server) Say(ctx context.Context, request *proto.Request) (*proto.Response, error) {
  return &proto.Response{Text: request.GetText()}, nil
}

func main() {
  server := &Server{}
  twirpHandler := proto.NewEchoServer(server, nil)

  http.ListenAndServe(":8080", twirpHandler)
}
```

使用自动生成的代码，我们只需要 3 步即可完成一个服务器：

1. 定义一个结构，可以存储一些状态。让它实现我们定义的`service`接口；
2. 创建一个该结构的对象，调用生成的`New{{ServiceName}}Server`方法创建`net/http`需要的处理器，这里的`ServiceName`为我们的服务名；
3. 监听端口。

客户端：

```golang
package main

import (
  "context"
  "fmt"
  "log"
  "net/http"

  "github.com/darjun/go-daily-lib/twirp/get-started/proto"
)

func main() {
  client := proto.NewEchoProtobufClient("http://localhost:8080", &http.Client{})

  response, err := client.Say(context.Background(), &proto.Request{Text: "Hello World"})
  if err != nil {
    log.Fatal(err)
  }
  fmt.Printf("response:%s\n", response.GetText())
}
```

`twirp`也生成了客户端相关代码，直接调用`NewEchoProtobufClient`连接对应的服务器上，然后调用`rpc`请求。

开启两个控制台，分别运行服务器和客户端程序。服务器：

```cmd
$ cd server && go run main.go
```

客户端：
```cmd
$ cd client && go run main.go
```

正确返回结果：

```cmd
response:Hello World
```

为了便于对照，下面列出该程序的目录结构。也可以去我的 GitHub 上查看示例代码：

```golang
get-started
├── client
│   └── main.go
├── proto
│   ├── echo.pb.go
│   ├── echo.proto
│   └── echo.twirp.go
└── server
    └── main.go
```

## JSON 客户端

除了使用 Protobuf，`twirp`还支持 JSON 格式的请求。使用也非常简单，只需要在创建`Client`时将`NewEchoProtobufClient`改为`NewEchoJSONClient`即可：

```golang
func main() {
  client := proto.NewEchoJSONClient("http://localhost:8080", &http.Client{})

  response, err := client.Say(context.Background(), &proto.Request{Text: "Hello World"})
  if err != nil {
    log.Fatal(err)
  }
  fmt.Printf("response:%s\n", response.GetText())
}
```

Protobuf Client 发送的请求带有`Content-Type: application/protobuf`的`Header`，JSON Client 则设置`Content-Type`为`application/json`。服务器收到请求时根据`Content-Type`来区分请求类型：

```golang
// proto/echo.twirp.go
unc (s *echoServer) serveSay(ctx context.Context, resp http.ResponseWriter, req *http.Request) {
  header := req.Header.Get("Content-Type")
  i := strings.Index(header, ";")
  if i == -1 {
    i = len(header)
  }
  switch strings.TrimSpace(strings.ToLower(header[:i])) {
  case "application/json":
    s.serveSayJSON(ctx, resp, req)
  case "application/protobuf":
    s.serveSayProtobuf(ctx, resp, req)
  default:
    msg := fmt.Sprintf("unexpected Content-Type: %q", req.Header.Get("Content-Type"))
    twerr := badRouteError(msg, req.Method, req.URL.Path)
    s.writeError(ctx, resp, twerr)
  }
}
```

## 提供其他 HTTP 服务

实际上，`twirpHandler`只是一个`http`的处理器，正如其他千千万万的处理器一样，没什么特殊的。我们当然可以挂载我们自己的处理器或处理器函数（概念有不清楚的可以参见我的《Go Web 编程》系列文章，[https://darjun.github.io/tags/go-web-%E7%BC%96%E7%A8%8B/](https://darjun.github.io/tags/go-web-%E7%BC%96%E7%A8%8B/）：

```golang
type Server struct{}

func (s *Server) Say(ctx context.Context, request *proto.Request) (*proto.Response, error) {
  return &proto.Response{Text: request.GetText()}, nil
}

func greeting(w http.ResponseWriter, r *http.Request) {
  name := r.FormValue("name")
  if name == "" {
    name = "world"
  }

  w.Write([]byte("hi," + name))
}

func main() {
  server := &Server{}
  twirpHandler := proto.NewEchoServer(server, nil)

  mux := http.NewServeMux()
  mux.Handle(twirpHandler.PathPrefix(), twirpHandler)
  mux.HandleFunc("/greeting", greeting)

  err := http.ListenAndServe(":8080", mux)
  if err != nil {
    log.Fatal(err)
  }
}
```

上面程序挂载了一个简单的`/greeting`请求，可以通过浏览器来请求地址`http://localhost:8080/greeting`。`twirp`的请求会挂载到路径`twirp/{{ServiceName}}`这个路径下，其中`ServiceName`为服务名。上面程序中的`PathPrefix()`会返回`/twirp/Echo`。

客户端：

```golang
func main() {
  client := proto.NewEchoProtobufClient("http://localhost:8080", &http.Client{})

  response, _ := client.Say(context.Background(), &proto.Request{Text: "Hello World"})
  fmt.Println("echo:", response.GetText())

  httpResp, _ := http.Get("http://localhost:8080/greeting")
  data, _ := ioutil.ReadAll(httpResp.Body)
  httpResp.Body.Close()
  fmt.Println("greeting:", string(data))

  httpResp, _ = http.Get("http://localhost:8080/greeting?name=dj")
  data, _ = ioutil.ReadAll(httpResp.Body)
  httpResp.Body.Close()
  fmt.Println("greeting:", string(data))
}
```

先运行服务器，然后执行客户端程序：

```cmd
$ go run main.go
echo: Hello World
greeting: hi,world
greeting: hi,dj
```

## 发送自定义的 Header

默认情况下，`twirp`实现会发送一些 Header。例如上面介绍的，使用`Content-Type`辨别客户端使用的协议格式。有时候我们可能需要发送一些自定义的 Header，例如`token`。`twirp`提供了`WithHTTPRequestHeaders`方法实现这个功能，该方法返回一个`context.Context`。发送时会将其中保存的 Header 一并发送。同样地，服务器要发送自定义 Header，使用`WithHTTPResponseHeaders`。

由于`twirp`封装了`net/http`，导致外层拿不到原始的`http.Request`和`http.Response`对象，所以 Header 的读取有点麻烦。在服务器端，由于`NewEchoServer`返回的是一个`http.Handler`，我们加一层中间件读取`http.Request`。看下面代码：

```golang
type Server struct{}

func (s *Server) Say(ctx context.Context, request *proto.Request) (*proto.Response, error) {
  token := ctx.Value("token").(string)
  fmt.Println("token:", token)

  err := twirp.SetHTTPResponseHeader(ctx, "Token-Lifecycle", "60")
  if err != nil {
    return nil, twirp.InternalErrorWith(err)
  }
  return &proto.Response{Text: request.GetText()}, nil
}

func WithTwirpToken(h http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    token := r.Header.Get("Twirp-Token")
    ctx = context.WithValue(ctx, "token", token)
    r = r.WithContext(ctx)

    h.ServeHTTP(w, r)
  })
}

func main() {
  server := &Server{}
  twirpHandler := proto.NewEchoServer(server, nil)
  wrapped := WithTwirpToken(twirpHandler)

  http.ListenAndServe(":8080", wrapped)
}
```

上面程序给客户端返回了一个名为`Token-Lifecycle`的 Header。客户端代码：

```golang
func main() {
  client := proto.NewEchoProtobufClient("http://localhost:8080", &http.Client{})

  header := make(http.Header)
  header.Set("Twirp-Token", "test-twirp-token")

  ctx := context.Background()
  ctx, err := twirp.WithHTTPRequestHeaders(ctx, header)
  if err != nil {
    log.Fatalf("twirp error setting headers: %v", err)
  }

  response, err := client.Say(ctx, &proto.Request{Text: "Hello World"})
  if err != nil {
    log.Fatalf("call say failed: %v", err)
  }
  fmt.Printf("response:%s\n", response.GetText())
}
```

运行程序，服务器正确获取客户端传过来的 token。

## 请求路由

我们前面已经介绍过了，`twirp`的`Server`实际上也就是一个`http.Handler`，如果我们知道了它的挂载路径，完全可以通过浏览器或者`curl`之类的工具去请求。我们启动`get-started`的服务器，然后用`curl`命令行工具去请求：

```cmd
$ curl --request "POST" \
  --location "http://localhost:8080/twirp/Echo/Say" \
  --header "Content-Type:application/json" \
  --data '{"text":"hello world"}'\
  --verbose
{"text":"hello world"}
```

这在调试的时候非常有用。

## 总结

本文介绍了 Go 的一个基于 Protobuf 生成代码的 RPC 框架，非常简单，小巧，实用。可以作为 gRPC 等的备选方案考虑。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. twirp GitHub：[https://github.com/twitchtv/twirp](https://github.com/twitchtv/twirp)
2. twirp 官方文档：[https://twitchtv.github.io/twirp/docs/intro.html](https://twitchtv.github.io/twirp/docs/intro.html)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)