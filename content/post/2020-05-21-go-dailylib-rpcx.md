---
layout:    post
title:    "Go 每日一库之 rpcx"
subtitle: 	"每天学习一个 Go 库"
date:		2020-05-21T23:04:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/05/21/godailylib/rpcx"
categories: [
	"Go"
]
---

## 简介

在之前的两篇文章[`rpc`](https://darjun.github.io/2020/05/08/godailylib/rpc)和[`json-rpc`](https://darjun.github.io/2020/05/10/godailylib/jsonrpc)中，我们介绍了 Go 标准库提供的`rpc`实现。在实际开发中，`rpc`库的功能还是有所欠缺。今天我们介绍一个非常优秀的 Go RPC 库——`rpcx`。`rpcx`是一位国人大牛开发的，详细开发历程可以在[`rpcx`官方博客](https://blog.rpcx.io/)了解。`rpcx`拥有媲美，甚至某种程度上超越`gRPC`的性能，有完善的中文文档，提供服务发现和治理的插件。

## 快速使用

本文示例使用`go modules`。

首先是安装：

```golang
$ go get -v -tags "reuseport quic kcp zookeeper etcd consul ping" github.com/smallnest/rpcx/...
```

可以看出`rpcx`的安装有点特殊。使用`go get -v github.com/smallnest/rpcx/...`命令只会安装`rpcx`的基础功能。扩展功能都是通过`build tags`指定。为了使用方便，一般安装所有的`tags`，如上面命令所示。这也是官方推荐的安装方式。

我们先编写服务端程序，实际上这个程序与用`rpc`标准库编写的程序几乎一模一样：

```golang
package main

import (
  "context"
  "errors"

  "github.com/smallnest/rpcx/server"
)

type Args struct {
  A, B int
}

type Quotient struct {
  Quo, Rem int
}

type Arith int

func (t *Arith) Mul(cxt context.Context, args *Args, reply *int) error {
  *reply = args.A * args.B
  return nil
}

func (t *Arith) Div(cxt context.Context, args *Args, quo *Quotient) error {
  if args.B == 0 {
    return errors.New("divide by 0")
  }

  quo.Quo = args.A / args.B
  quo.Rem = args.A % args.B
  return nil
}

func main() {
  s := server.NewServer()
  s.RegisterName("Arith", new(Arith), "")
  s.Serve("tcp", ":8972")
}
```

首先创建一个`Server`对象，调用它的`RegisterName()`方法在服务路径`Arith`下注册`Mul`和`Div`方法。与标准库相比，`rpcx`要求注册方法的第一个参数必须为`context.Context`类型。最后调用`s.Serve("tcp", ":8972")`监听 TCP 端口 8972。是不是很简单？启动服务器：

```cmd
$ go run main.go
```

然后是客户端程序：

```golang
package main

import (
  "context"
  "flag"
  "log"

  "github.com/smallnest/rpcx/client"
)

var (
  addr = flag.String("addr", ":8972", "service address")
)

func main() {
  flag.Parse()

  d := client.NewPeer2PeerDiscovery("tcp@"+*addr, "")
  xclient := client.NewXClient("Arith", client.Failtry, client.RandomSelect, d, client.DefaultOption)
  defer xclient.Close()

  args := &Args{A:10, B:20}
  var reply int

  err :=xclient.Call(context.Background(), "Mul", args, &reply)
  if err != nil {
    log.Fatalf("failed to call: %v", err)
  }

  fmt.Printf("%d * %d = %d\n", args.A, args.B, reply)

  args = &Args{50, 20}
  var quo Quotient
  err = xclient.Call(context.Background(), "Div", args, &quo)
  if err != nil {
    log.Fatalf("failed to call: %v", err)
  }

  fmt.Printf("%d * %d = %d...%d\n", args.A, args.B, quo.Quo, quo.Rem)
}
```

`rpcx`支持多种服务发现的方式让客户端找到服务器。上面代码中我们使用的是最简单的点到点的方式，也就是**直连**。要调用服务端的方法，必须先创建一个`Client`对象。使用`Client`对象来调用远程方法。运行客户端：

```cmd
$ go run main.go
10 * 20 = 200
50 * 20 = 2...10
```

注意到，创建`Client`对象的参数有`client.Failtry`和`client.RandomSelect`。这两个参数分别为**失败模式**和**如何选择服务器**。

## 传输

`rpcx`支持多种传输协议：

* `TCP`：TCP 协议，网络名称为`tcp`；
* `HTTP`：HTTP 协议，网络名称为`http`；
* `UnixDomain`：unix 域协议，网络名称为`unix`；
* `QUIC`：是 Quick UDP Internet Connections 的缩写，意为**快速UDP网络连接**。HTTP/3 底层就是 QUIC 协议，Google 出品。网络名称为`quic`；
* `KCP`：快速并且可靠的 ARQ 协议，网络名称为`kcp`。

`rpcx`对这些协议做了非常好的封装。除了在创建服务器和客户端连接时需要指定协议名称，其它时候的使用基本是透明的。我们将上面的例子改装成使用`http`协议的：

服务端改动：

```golang
s.Serve("http", ":8972")
```

客户端改动：

```golang
d := client.NewPeer2PeerDiscovery("http@"+*addr, "")
```

`QUIC`和`KCP`的使用有点特殊，`QUIC`必须与 TLS 一起使用，`KCP`也需要做传输加密。使用 Go 语言我们能很方便地生成一个证书和私钥：

```golang
package main

import (
  "crypto/rand"
  "crypto/rsa"
  "crypto/x509"
  "crypto/x509/pkix"
  "encoding/pem"
  "math/big"
  "net"
  "os"
  "time"
)

func main() {
  max := new(big.Int).Lsh(big.NewInt(1), 128)
  serialNumber, _ := rand.Int(rand.Reader, max)
  subject := pkix.Name{
    Organization:       []string{"Go Daily Lib"},
    OrganizationalUnit: []string{"TechBlog"},
    CommonName:         "go daily lib",
  }

  template := x509.Certificate{
    SerialNumber: serialNumber,
    Subject:      subject,
    NotBefore:    time.Now(),
    NotAfter:     time.Now().Add(365 * 24 * time.Hour),
    KeyUsage:     x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature,
    ExtKeyUsage:  []x509.ExtKeyUsage{x509.ExtKeyUsageServerAuth},
    IPAddresses:  []net.IP{net.ParseIP("127.0.0.1")},
  }

  pk, _ := rsa.GenerateKey(rand.Reader, 2048)

  derBytes, _ := x509.CreateCertificate(rand.Reader, &template, &template, &pk.PublicKey, pk)
  certOut, _ := os.Create("server.pem")
  pem.Encode(certOut, &pem.Block{Type: "CERTIFICATE", Bytes: derBytes})
  certOut.Close()

  keyOut, _ := os.Create("server.key")
  pem.Encode(keyOut, &pem.Block{Type: "RSA PRIVATE KEY", Bytes: x509.MarshalPKCS1PrivateKey(pk)})
  keyOut.Close()
}
```

上面代码生成了一个证书和私钥，有效期为 1 年。运行程序，得到两个文件`server.pem`和`server.key`。然后我们就可以编写使用`QUIC`协议的程序了。服务端：

```golang
func main() {
  cert, _ := tls.LoadX509KeyPair("server.pem", "server.key")
  config := &tls.Config{Certificates: []tls.Certificate{cert}}

  s := server.NewServer(server.WithTLSConfig(config))
  s.RegisterName("Arith", new(Arith), "")
  s.Serve("quic", "localhost:8972")
}
```

实际上就是加载证书和密钥，然后在创建`Server`对象时作为选项传入。客户端改动：

```golang
conf := &tls.Config{
  InsecureSkipVerify: true,
}

option := client.DefaultOption
option.TLSConfig = conf
d := client.NewPeer2PeerDiscovery("quic@"+*addr, "")
xclient := client.NewXClient("Arith", client.Failtry, client.RandomSelect, d, option)
defer xclient.Close()
```

客户端也需要配置 TLS。

有一点需要注意，`rpcx`对`quic/kcp`这些协议的支持是通过`build tags`实现的。默认不会编译`quic/kcp`相关文件。如果要使用，必须自己手动指定`tags`。先启动服务端程序：

```
$ go run -tags quic main.go
```

然后切换到客户端程序目录，执行下面命令：

```golang
$ go run -tags quic main.go
```

还有一点需要注意，在使用`tcp`和`http`（底层也是`tcp`）协议的时候，我们可以将地址简写为`:8972`，因为默认就是本地地址。但是`quic`不行，必须把地址写完整：

```golang
// 服务端
s.Serve("quic", "localhost:8972")
// 客户端
addr = flag.String("addr", "localhost:8972", "service address")
```

## 注册函数

上面的例子都是调用对象的方法，我们也可以调用函数。函数的类型与对象方法相比只是没有**接收者**。注册函数需要指定一个服务路径。服务端：

```golang
type Args struct {
  A, B int
}

type Quotient struct {
  Quo, Rem int
}


func Mul(cxt context.Context, args *Args, reply *int) error {
  *reply = args.A * args.B
  return nil
}

func Div(cxt context.Context, args *Args, quo *Quotient) error {
  if args.B == 0 {
    return errors.New("divide by 0")
  }

  quo.Quo = args.A / args.B
  quo.Rem = args.A % args.B
  return nil
}

func main() {
  s := server.NewServer()
  s.RegisterFunction("function", Mul, "")
  s.RegisterFunction("function", Div, "")
  s.Serve("tcp", ":8972")
}
```

只是注册方法由`RegisterName`变为了`RegisterFunction`，参数由一个对象变为一个函数。我们需要为注册的函数指定一个服务路径，客户端调用时会根据这个路径查找对应方法。客户端：

```golang
func main() {
  flag.Parse()

  d := client.NewPeer2PeerDiscovery("tcp@"+*addr, "")
  xclient := client.NewXClient("function", client.Failtry, client.RandomSelect, d, client.DefaultOption)
  defer xclient.Close()

  args := &Args{A: 10, B: 20}
  var reply int

  err := xclient.Call(context.Background(), "Mul", args, &reply)
  if err != nil {
    log.Fatalf("failed to call: %v", err)
  }

  fmt.Printf("%d * %d = %d\n", args.A, args.B, reply)

  args = &Args{50, 20}
  var quo Quotient
  err = xclient.Call(context.Background(), "Div", args, &quo)
  if err != nil {
    log.Fatalf("failed to call: %v", err)
  }

  fmt.Printf("%d * %d = %d...%d\n", args.A, args.B, quo.Quo, quo.Rem)
}
```

## 注册中心

`rpcx`支持多种注册中心：

* 点对点：其实就是直连，没有注册中心；
* 点对多：可以配置多个服务器；
* `zookeeper`：常用的注册中心；
* `Etcd`：Go 语言编写的注册中心；
* 进程内调用：方便调试功能，在同一个进程内查找服务；
* `Consul/mDNS`等。

我们之前演示的都是**点对点**的连接，接下来我们介绍如何使用`zookeeper`作为注册中心。在`rpcx`中，注册中心是通过插件的方式集成的。使用`ZooKeeperRegisterPlugin`这个插件来集成`Zookeeper`。服务端代码：

```golang
type Args struct {
  A, B int
}

type Quotient struct {
  Quo, Rem int
}

var (
  addr     = flag.String("addr", ":8972", "service address")
  zkAddr   = flag.String("zkAddr", "127.0.0.1:2181", "zookeeper address")
  basePath = flag.String("basePath", "/services/math", "service base path")
)

type Arith int

func (t *Arith) Mul(cxt context.Context, args *Args, reply *int) error {
  fmt.Println("Mul on", *addr)
  *reply = args.A * args.B
  return nil
}

func (t *Arith) Div(cxt context.Context, args *Args, quo *Quotient) error {
  fmt.Println("Div on", *addr)
  if args.B == 0 {
    return errors.New("divide by 0")
  }

  quo.Quo = args.A / args.B
  quo.Rem = args.A % args.B
  return nil
}

func main() {
  flag.Parse()

  p := &serverplugin.ZooKeeperRegisterPlugin{
    ServiceAddress:   "tcp@" + *addr,
    ZooKeeperServers: []string{*zkAddr},
    BasePath:         *basePath,
    Metrics:          metrics.NewRegistry(),
    UpdateInterval:   time.Minute,
  }
  if err := p.Start(); err != nil {
    log.Fatal(err)
  }

  s := server.NewServer()
  s.Plugins.Add(p)

  s.RegisterName("Arith", new(Arith), "")
  s.Serve("tcp", *addr)
}
```

在`ZooKeeperRegisterPlugin`中，我们指定了本服务地址，zookeeper 集群地址（可以是多个），起始路径等。服务器启动时自动向 zookeeper 注册本服务的信息，客户端可直接从 zookeeper 拉取可用的服务列表。

首先启动 zookeeper 服务器，zookeeper 的安装与启动可以参考我的上一篇文章。分别在 3 个控制台中启动 3 个服务器，指定不同的端口（注意需要指定`-tags zookeeper`）：

```cmd
// 控制台1
$ go run -tags zookeeper main.go -addr 127.0.0.1:8971
// 控制台2
$ go run -tags zookeeper main.go -addr 127.0.0.1:8972
// 控制台3
$ go run -tags zookeeper main.go -addr 127.0.0.1:8973
```

启动之后，我们观察 zookeeper 路径`/services/math`中的内容：

![](/img/in-post/godailylib/rpcx1.png#center)

非常棒，可用的服务地址不用我们手动维护了！

接下来是客户端：

```golang
var (
  zkAddr   = flag.String("zkAddr", "127.0.0.1:2181", "zookeeper address")
  basePath = flag.String("basePath", "/services/math", "service base path")
)

func main() {
  flag.Parse()

  d := client.NewZookeeperDiscovery(*basePath, "Arith", []string{*zkAddr}, nil)
  xclient := client.NewXClient("Arith", client.Failtry, client.RandomSelect, d, client.DefaultOption)
  defer xclient.Close()

  args := &Args{A: 10, B: 20}
  var reply int

  err := xclient.Call(context.Background(), "Mul", args, &reply)
  if err != nil {
    log.Fatalf("failed to call: %v", err)
  }

  fmt.Printf("%d * %d = %d\n", args.A, args.B, reply)

  args = &Args{50, 20}
  var quo Quotient
  err = xclient.Call(context.Background(), "Div", args, &quo)
  if err != nil {
    log.Fatalf("failed to call: %v", err)
  }

  fmt.Printf("%d * %d = %d...%d\n", args.A, args.B, quo.Quo, quo.Rem)
}
```

我们通过 zookeeper 读取可用的`Arith`服务列表，然后随机选择一个服务发送请求：

```cmd
$ go run -tags zookeeper main.go
2020/05/26 23:03:40 Connected to 127.0.0.1:2181
2020/05/26 23:03:40 authenticated: id=72057658440744975, timeout=10000
2020/05/26 23:03:40 re-submitting `0` credentials after reconnect
10 * 20 = 200
50 * 20 = 2...10
```

我们的客户端发送了两条请求。由于使用了`client.RandomSelect`策略，所以这两个请求随机发送到某个服务端。我在`Mul`和`Div`方法中增加了一个打印，可以观察一下各个控制台的输出！

如果我们关闭了某个服务器，对应的服务地址会从 zookeeper 中移除。我关闭了服务器 1，zookeeper 服务列表变为：

![](/img/in-post/godailylib/rpcx2.png#center)

相比上一篇文章中需要手动维护 zookeeper 的内容，`rpcx`的自动注册和维护明显要方便太多了！

## 总结

`rpcx`是 Go 语言中首屈一指的 rpc 库，功能丰富，性能出众，文档丰富，已经被不少公司和个人采用。推荐深入学习！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. rpcx GitHub：[https://github.com/smallnest/rpcx](https://github.com/jordan-wright/rpcx)
2. rpcx 博客：[https://blog.rpcx.io/](https://blog.rpcx.io/)
3. rpcx 官网：[https://rpcx.io/](https://rpcx.io/)
4. rpcx 文档：[https://doc.rpcx.io/](https://doc.rpcx.io/)
5. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)