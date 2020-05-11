---
layout:    post
title:    "Go 每日一库之 jsonrpc"
subtitle: 	"每天学习一个 Go 库"
date:		2020-05-10T19:30:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/05/10/godailylib/jsonrpc"
categories: [
	"Go"
]
---

## 简介

在[上一篇文章](https://darjun.github.io/2020/05/08/godailylib/rpc)中我们介绍了 Go 标准库`net/rpc`的用法。在默认情况下，`rpc`库内部使用`gob`格式传输数据。我们仿造`gob`的编解码器实现了一个`json`格式的。实际上标准库`net/rpc/jsonrcp`中已有实现。本文是对上一篇文章的补充。

## 快速使用

标准库无需安装。

首先是服务端，使用`net/rpc/jsonrpc`之后，我们就不用自己去编写`json`的编解码器了：

```golang
package main

import (
  "log"
  "net"
  "net/rpc"
  "net/rpc/jsonrpc"
)

type Args struct {
  A, B int
}

type Arith int

func (t *Arith) Multiply(args *Args, reply *int) error {
  *reply = args.A * args.B
  return nil
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

    // 注意这一行
    go rpc.ServeCodec(jsonrpc.NewServerCodec(conn))
  }
}
```

直接调用`jsonrpc.NewServerCodec(conn)`创建一个服务端的`codec`。客户端也是类似的：

```golang
func main() {
  conn, err := net.Dial("tcp", ":1234")
  if err != nil {
    log.Fatal("dial error:", err)
  }

  // 这里，这里😁
  client := rpc.NewClientWithCodec(jsonrpc.NewClientCodec(conn))

  args := &Args{7, 8}
  var reply int
  err = client.Call("Arith.Multiply", args, &reply)
  if err != nil {
    log.Fatal("Multiply error:", err)
  }
  fmt.Printf("Multiply: %d*%d=%d\n", args.A, args.B, reply)
}
```

先运行服务端程序：

```golang
$ go run main.go
```

然后在一个新的控制台中运行客户端程序：

```golang
$ go run client.go
Multiply: 7*8=56
```

下面这段代码基本上每个使用`jsonrpc`的程序都要编写：

```golang
conn, err := net.Dial("tcp", ":1234")
if err != nil {
  log.Fatal("dial error:", err)
}

client := rpc.NewClientWithCodec(jsonrpc.NewClientCodec(conn))
```

因此`jsonrpc`为了方便直接提供了一个`Dial`方法。使用`Dial`简化上面的客户端程序：

```golang
func main() {
  client, err := jsonrpc.Dial("tcp", ":1234")
  if err != nil {
    log.Fatal("dial error:", err)
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

效果是一样的。

## JSON-RPC 标准

JSON-RPC 1.0 标准在 2005 年发布，经过数年演化，于 2010 年发布了 2.0 版本。JSON-RPC 标准的内容可在[https://www.jsonrpc.org/specification](https://www.jsonrpc.org/specification)查看。Go 标准库`net/rpc/jsonrpc`实现了 1.0 版本。关于 2.0 版本的实现可以在`pkg.go.dev`上搜索`json-rpc+2.0`。本文以 1.0 版本为基础进行介绍。

JSON-RPC 传输的是单一的对象，序列化为 JSON 格式。请求对象包含以下 3 个属性：

* `method`：请求调用的方法；
* `params`：一个数组表示传给方法的各个参数；
* `id`：请求 ID。ID 可以是任何类型，在收到响应时根据这个属性判断对应哪个请求。

响应对象包含以下 3 个属性：

* `result`：方法返回的对象，如果`error`非空时，该属性必须为`null`；
* `error`：表示调用是否出错；
* `id`：对应请求的 ID。

另外标准还定义了一种通知类型，除了`id`属性为`null`之外，通知对象的属性与请求对象完全一样。

调用`client.Call("echo", "Hello JSON-RPC", &reply)`时：

```golang
请求：{ "method": "echo", "params": ["Hello JSON-RPC"], "id": 1}
响应：{ "result": "Hello JSON-RPC", "error": null, "id": 1}
```

## 使用 zookeeper 实现简单的负载均衡

下面我们使用`zookeeper`实现一个简单的**客户端**侧的负载均衡。`zookeeper`中记录所有的可提供服务的服务器，客户端每次请求时都随机挑选一个。**我们的示例中，请求必须是无状态的**。首先，我们改造一下服务端程序，将监听地址提取出来，通过`flag`指定：

```golang
package main

import (
  "flag"
  "log"
  "net"
  "net/rpc"
  "net/rpc/jsonrpc"
)

var (
  addr *string
)

type Args struct {
  A, B int
}

type Arith int

func (t *Arith) Multiply(args *Args, reply *int) error {
  *reply = args.A * args.B
  return nil
}

func init() {
  addr = flag.String("addr", ":1111", "addr to listen")
}

func main() {
  flag.Parse()

  l, err := net.Listen("tcp", *addr)
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

    go rpc.ServeCodec(jsonrpc.NewServerCodec(conn))
  }
}
```

关于有哪些服务器可用，我们存储在`zookeeper`中。

首先要启动一个`zookeeper`的程序。在 Apache Zookeeper 官网可以下载能直接运行的 Windows 程序。下载之后解压，将`conf`文件夹中的样板配置`zoo_sample.cfg`复制一份，文件名改为`zoo.cfg`。在编辑器中打开`zoo.cfg`，将`dataDir`改为一个已存在的目录，或创建一个新目录。我在`bin`同级目录中创建了一个`data`目录，然后设置`dataDir=../data`。切换到`bin`目录下执行`zkServer.bat`，`zookeeper`程序就运行起来了。使用`zkClient.bat`连接上这个`zookeeper`，增加一个节点，设置数据：

```golang
$ create /rpcserver
$ set /rpcserver 127.0.0.1:1111,127.0.0.1:1112,127.0.0.1:1113
```

我们用`,`分隔多个服务器地址。

准备工作完成后，接下来就开始编写客户端代码了。我们实现一个代理类，负责监听`zookeeper`的数据变化，根据`zookeeper`中新的地址创建到服务器的连接，删除老的连接，将调用请求随机转发到一个服务器处理：

```golang
type Proxy struct {
  zookeeper     string
  clients       map[string]*rpc.Client
  events        <-chan zk.Event
  zookeeperConn *zk.Conn
  mutex         sync.Mutex
}

func NewProxy(addr string) *Proxy {
  return &Proxy{
    zookeeper: addr,
    clients:   make(map[string]*rpc.Client),
  }
}
```

这里我们使用了`go-zookeeper`这个库，需要额外安装：

```cmd
$ go get github.com/samuel/go-zookeeper/zk
```

程序启动时，代理对象从`zookeeper`中获取服务端地址，创建连接：

```golang
func (p *Proxy) Connect() {
  c, _, err := zk.Connect([]string{p.zookeeper}, time.Second) //*10)
  if err != nil {
    panic(err)
  }

  data, _, event, err := c.GetW("/rpcserver")
  if err != nil {
    panic(err)
  }

  p.events = event
  p.zookeeperConn = c

  p.CreateClients(string(data))
}

func (p *Proxy) CreateClients(server string) {
  p.mutex.Lock()
  defer p.mutex.Unlock()

  addrs := strings.Split(server, ",")
  allAddr := make(map[string]struct{})
  for _, addr := range addrs {
    allAddr[addr] = struct{}{}
    if _, exist := p.clients[addr]; exist {
      continue
    }

    client, err := jsonrpc.Dial("tcp", addr)
    if err != nil {
      log.Println("jsonrpc Dial error:", err)
      continue
    }

    p.clients[addr] = client
    log.Println("new addr:", addr)
  }

  for addr := range p.clients {
    if _, exist := allAddr[addr]; !exist {
	  // 不在 zookeeper 中的地址，删除对应连接
	  oldClient.Close()
	  delete(p.clients, addr)

      log.Println("delete addr", addr)
    }
  }
}
```

同时，需要监听`zookeeper`中的数据变化，当新增或删除某个服务端地址时，`Proxy`要及时更新连接：

```golang
func (p *Proxy) Run() {
  for {
    select {
    case event := <-p.events:
      if event.Type == zk.EventNodeDataChanged {
        data, _, err := p.zookeeperConn.Get("/rpcserver")
        if err != nil {
          log.Println("get zookeeper data failed:", err)
          continue
        }

        p.CreateClients(string(data))
      }
    }
  }
}
```

客户端主体程序使用`Proxy`结构非常方便：

```golang
package main

import (
  "flag"
  "fmt"
  "math/rand"
)

var (
  zookeeperAddr *string
)

func init() {
  zookeeperAddr = flag.String("addr", ":2181", "zookeeper address")
}

type Args struct {
  A, B int
}

func main() {
  flag.Parse()

  fmt.Println(*zookeeperAddr)
  p := NewProxy(*zookeeperAddr)
  p.Connect()

  go p.Run()

  for i := 0; i < 10; i++ {
    var reply int
    args := &Args{rand.Intn(1000), rand.Intn(1000)}
    p.Call("Arith.Multiply", args, &reply)
    fmt.Printf("%d*%d=%d\n", args.A, args.B, reply)
  }

  // sleep 过程中可以修改 zookeeper 中的数据
  time.Sleep(1 * time.Minute)

  // 使用新的地址做随机
  for i := 0; i < 100; i++ {
    var reply int
    args := &Args{rand.Intn(1000), rand.Intn(1000)}
    p.Call("Arith.Multiply", args, &reply)
    fmt.Printf("%d*%d=%d\n", args.A, args.B, reply)
  }
}
```

创建一个代理对象，在一个新的 goroutine 中监听`zookeeper`事件。然后通过`Proxy`的`Call`调用远程服务端的方法：

```golang
func (p *Proxy) Call(method string, args interface{}, reply interface{}) error {
  var client *rpc.Client
  var addr string
  idx := rand.Int31n(int32(len(p.clients)))
  var i int32
  p.mutex.Lock()
  for a, c := range p.clients {
    if i == idx {
      client = c
      addr = a
      break
    }
    i++
  }
  p.mutex.Unlock()

  fmt.Println("use", addr)
  return client.Call(method, args, reply)
}
```

首先我们要启动 3 个服务端程序，分别监听端口 1111、1112、1113，需要 3 个控制台：

控制台 1：

```golang
$ go run main.go -addr :1111

```

控制台 2：

```golang
$ go run main.go -addr :1112

```

控制台 3：

```golang
$ go run main.go -addr :1113

```

客户端在一个新的控制台启动，指定`zookeeper`地址：

```golang
$ go run . -addr=127.0.0.1:2181
```

在输出中，我们可以看到是怎么随机挑选服务器的。

我们可以尝试在客户端程序运行的过程中，将某个服务器地址从`zookeeper`中删除。我特意在程序中加了一个 1 分钟的延迟。在`sleep`过程中，通过`zkClient.cmd`将`127.0.0.1:1113`这个地址从`zookeeper`中删除：

```cmd
$ set /rpcserver 127.0.0.1:1111,127.0.0.1:1112
```

控制台输出:

```cmd
$ 2020/05/10 23:47:47 delete addr 127.0.0.1:1113
```

并且后续的请求不会再发到`127.0.0.1:1113`这个服务器了。

其实，在实际的项目中，`Proxy`一般是一个独立的服务器，而不是放在客户端侧。上面示例这样处理只是为了方便。

## 总结

RPC 底层可以使用各种协议传输数据，JSON/XML/Protobuf 都可以。对 rpc 感兴趣的建议看看`rpcx`这个库，[https://github.com/smallnest/rpcx](https://github.com/smallnest/rpcx)。非常强大！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. jsonrpc GitHub：[https://golang.org/pkg/net/rpc/jsonrpc/](https://golang.org/pkg/net/rpc/jsonrpc/)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)