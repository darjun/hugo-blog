---
layout:    post
title:    "Go 每日一库之 net/http（基础和中间件）"
subtitle: "每天学习一个 Go 语言库"
date:     2021-07-13T06:12:35
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/07/13/in-post/godailylib/nethttp"
categories: [
  "Go"
]
---

## 简介

几乎所有的编程语言都以`Hello World`作为入门程序的示例，其中有一部分以编写一个 Web 服务器作为实战案例的开始。每种编程语言都有很多用于编写 Web 服务器的库，或以标准库，或通过第三方库的方式提供。Go 语言也不例外。本文及后续的文章就去探索 Go 语言中的各个Web 编程框架，它们的基本使用，阅读它们的源码，比较它们优缺点。让我们先从 Go 语言的标准库`net/http`开始。标准库`net/http`让编写 Web 服务器的工作变得非常简单。我们一起探索如何使用`net/http`库实现一些常见的功能或模块，了解这些对我们学习其他的库或框架将会很有帮助。

## Hello World

使用`net/http`编写一个简单的 Web 服务器非常简单：

```golang
package main

import (
  "fmt"
  "net/http"
)

func index(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, "Hello World")
}

func main() {
  http.HandleFunc("/", index)
  http.ListenAndServe(":8080", nil)
}
```

首先，我们调用`http.HandleFunc("/", index)`注册路径处理函数，这里将路径`/`的处理函数设置为`index`。处理函数的类型必须是：

```golang
func (http.ResponseWriter, *http.Request)
```

其中`*http.Request`表示 HTTP 请求对象，该对象包含请求的所有信息，如 URL、首部、表单内容、请求的其他内容等。

`http.ResponseWriter`是一个接口类型：

```golang
// net/http/server.go
type ResponseWriter interface {
  Header() Header
  Write([]byte) (int, error)
  WriteHeader(statusCode int)
}
```

用于向客户端发送响应，实现了`ResponseWriter`接口的类型显然也实现了`io.Writer`接口。所以在处理函数`index`中，可以调用`fmt.Fprintln()`向`ResponseWriter`写入响应信息。

仔细阅读`net/http`包中`HandleFunc()`函数的源码：

```golang
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
  DefaultServeMux.HandleFunc(pattern, handler)
}
```

我们发现它直接调用了一个名为`DefaultServeMux`对象的`HandleFunc()`方法。`DefaultServeMux`是`ServeMux`类型的实例：

```golang
type ServeMux struct {
  mu    sync.RWMutex
  m     map[string]muxEntry
  es    []muxEntry // slice of entries sorted from longest to shortest.
  hosts bool       // whether any patterns contain hostnames
}

var DefaultServeMux = &defaultServeMux
var defaultServeMux ServeMux
```

**像这种提供默认类型实例的用法在 Go 语言的各个库中非常常见，在默认参数就已经足够的场景中使用默认实现很方便**。`ServeMux`保存了注册的所有路径和处理函数的对应关系。`ServeMux.HandleFunc()`方法如下：

```golang
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
  mux.Handle(pattern, HandlerFunc(handler))
}
```

这里将处理函数`handler`转为`HandlerFunc`类型，然后调用`ServeMux.Handle()`方法注册。注意这里的`HandlerFunc(handler)`是类型转换，而非函数调用，类型`HandlerFunc`的定义如下：

```golang
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
  f(w, r)
}
```

`HandlerFunc`实际上是以函数类型`func(ResponseWriter, *Request)`为底层类型，为`HandlerFunc`类型定义了方法`ServeHTTP`。是的，Go 语言允许为（基于）函数的类型定义方法。`Serve.Handle()`方法只接受类型为接口`Handler`的参数：

```golang
type Handler interface {
  ServeHTTP(ResponseWriter, *Request)
}

func (mux *ServeMux) Handle(pattern string, handler Handler) {
  if mux.m == nil {
    mux.m = make(map[string]muxEntry)
  }
  e := muxEntry{h: handler, pattern: pattern}
  if pattern[len(pattern)-1] == '/' {
    mux.es = appendSorted(mux.es, e)
  }
  mux.m[pattern] = e
}
```

显然`HandlerFunc`实现了接口`Handler`。`HandlerFunc`类型只是为了方便注册函数类型的处理器。我们当然可以直接定义一个实现`Handler`接口的类型，然后注册该类型的实例：

```golang
type greeting string

func (g greeting) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, g)
}

http.Handle("/greeting", greeting("Welcome, dj"))
```

我们基于`string`类型定义了一个新类型`greeting`，然后为它定义一个方法`ServeHTTP()`（实现接口`Handler`），最后调用`http.Handle()`方法注册该处理器。

为了便于区分，我们将通过`HandleFunc()`注册的称为处理函数，将通过`Handle()`注册的称为处理器。通过上面的源码分析不难看出，它们在底层本质上是一回事。

注册了处理逻辑后，调用`http.ListenAndServe(":8080", nil)`监听本地计算机的 8080 端口，开始处理请求。下面看源码的处理：

```golang
func ListenAndServe(addr string, handler Handler) error {
  server := &Server{Addr: addr, Handler: handler}
  return server.ListenAndServe()
}
```

`ListenAndServe`创建了一个`Server`类型的对象：

```golang
type Server struct {
  Addr string
  Handler Handler
  TLSConfig *tls.Config
  ReadTimeout time.Duration
  ReadHeaderTimeout time.Duration
  WriteTimeout time.Duration
  IdleTimeout time.Duration
}
```

`Server`结构体有比较多的字段，我们可以使用这些字段来调节 Web 服务器的参数，如上面的`ReadTimeout/ReadHeaderTimeout/WriteTimeout/IdleTimeout`用于控制读写和空闲超时。在该方法中，先调用`net.Listen()`监听端口，将返回的`net.Listener`作为参数调用`Server.Serve()`方法：

```golang
func (srv *Server) ListenAndServe() error {
  addr := srv.Addr
  ln, err := net.Listen("tcp", addr)
  if err != nil {
    return err
  }
  return srv.Serve(ln)
}
```

在`Server.Serve()`方法中，使用一个无限的`for`循环，不停地调用`Listener.Accept()`方法接受新连接，开启新 goroutine 处理新连接：

```golang
func (srv *Server) Serve(l net.Listener) error {
  var tempDelay time.Duration // how long to sleep on accept failure
  for {
    rw, err := l.Accept()
    if err != nil {
      if ne, ok := err.(net.Error); ok && ne.Temporary() {
        if tempDelay == 0 {
          tempDelay = 5 * time.Millisecond
        } else {
          tempDelay *= 2
        }
        if max := 1 * time.Second; tempDelay > max {
          tempDelay = max
        }
        srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
        time.Sleep(tempDelay)
        continue
      }
      return err
    }
    tempDelay = 0
    c := srv.newConn(rw)
    go c.serve(connCtx)
  }
}
```

这里有一个**指数退避策略**的用法。如果`l.Accept()`调用返回错误，我们判断该错误是不是临时性地（`ne.Temporary()`）。如果是临时性错误，`Sleep`一小段时间后重试，每发生一次临时性错误，`Sleep`的时间翻倍，最多`Sleep` 1s。获得新连接后，将其封装成一个`conn`对象（`srv.newConn(rw)`），创建一个 goroutine 运行其`serve()`方法。省略无关逻辑的代码如下：

```golang
func (c *conn) serve(ctx context.Context) {
  for {
    w, err := c.readRequest(ctx)
    serverHandler{c.server}.ServeHTTP(w, w.req)
    w.finishRequest()
  }
}
```

`serve()`方法其实就是不停地读取客户端发送地请求，创建`serverHandler`对象调用其`ServeHTTP()`方法去处理请求，然后做一些清理工作。`serverHandler`只是一个中间的辅助结构，代码如下：

```golang
type serverHandler struct {
  srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
  handler := sh.srv.Handler
  if handler == nil {
    handler = DefaultServeMux
  }
  handler.ServeHTTP(rw, req)
}
```

从`Server`对象中获取`Handler`，这个`Handler`就是调用`http.ListenAndServe()`时传入的第二个参数。在`Hello World`的示例代码中，我们传入了`nil`。所以这里`handler`会取默认值`DefaultServeMux`。调用`DefaultServeMux.ServeHTTP()`方法处理请求：

```golang
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
  h, _ := mux.Handler(r)
  h.ServeHTTP(w, r)
}
```

`mux.Handler(r)`通过请求的路径信息查找处理器，然后调用处理器的`ServeHTTP()`方法处理请求：

```golang
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
  host := stripHostPort(r.Host)
  return mux.handler(host, r.URL.Path)
}

func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
  h, pattern = mux.match(path)
  return
}

func (mux *ServeMux) match(path string) (h Handler, pattern string) {
  v, ok := mux.m[path]
  if ok {
    return v.h, v.pattern
  }

  for _, e := range mux.es {
    if strings.HasPrefix(path, e.pattern) {
      return e.h, e.pattern
    }
  }
  return nil, ""
}
```

上面的代码省略了大量的无关代码，在`match`方法中，首先会检查路径是否精确匹配`mux.m[path]`。如果不能精确匹配，后面的`for`循环会匹配路径的最长前缀。**只要注册了`/`根路径处理，所有未匹配到的路径最终都会交给`/`路径处理**。为了保证最长前缀优先，在注册时，会对路径进行排序。所以`mux.es`中存放的是按路径排序的处理列表：

```golang
func appendSorted(es []muxEntry, e muxEntry) []muxEntry {
  n := len(es)
  i := sort.Search(n, func(i int) bool {
    return len(es[i].pattern) < len(e.pattern)
  })
  if i == n {
    return append(es, e)
  }
  es = append(es, muxEntry{})
  copy(es[i+1:], es[i:])
  es[i] = e
  return es
}
```

运行，在浏览器中键入网址`localhost:8080`，可以看到网页显示`Hello World`。键入网址`localhost:8080/greeting`，看到网页显示`Welcome, dj`。

思考题：
根据最长前缀的逻辑，如果键入`localhost:8080/greeting/a/b/c`，应该会匹配`/greeting`路径。
如果键入`localhost:8080/a/b/c`，应该会匹配`/`路径。是这样么？答案放在后面😀。

### 创建`ServeMux`

调用`http.HandleFunc()/http.Handle()`都是将处理器/函数注册到`ServeMux`的默认对象`DefaultServeMux`上。使用默认对象有一个问题：不可控。

一来`Server`参数都使用了默认值，二来第三方库也可能使用这个默认对象注册一些处理，容易冲突。更严重的是，我们在不知情中调用`http.ListenAndServe()`开启 Web 服务，那么第三方库注册的处理逻辑就可以通过网络访问到，有极大的安全隐患。所以，除非在示例程序中，否则建议不要使用默认对象。

我们可以使用`http.NewServeMux()`创建一个新的`ServeMux`对象，然后创建`http.Server`对象定制参数，用`ServeMux`对象初始化`Server`的`Handler`字段，最后调用`Server.ListenAndServe()`方法开启 Web 服务：

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", index)
  mux.Handle("/greeting", greeting("Welcome to go web frameworks"))

  server := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  20 * time.Second,
    WriteTimeout: 20 * time.Second,
  }
  server.ListenAndServe()
}
```

这个程序与上面的`Hello World`功能基本相同，我们还额外设置了读写超时。

为了便于理解，我画了两幅图，其实整理下来整个流程也不复杂：

![](/img/in-post/godailylib/nethttp1.png#center)

![](/img/in-post/godailylib/nethttp2.png#center)

## 中间件

有时候需要在请求处理代码中增加一些通用的逻辑，如统计处理耗时、记录日志、捕获宕机等等。如果在每个请求处理函数中添加这些逻辑，代码很快就会变得不可维护，添加新的处理函数也会变得非常繁琐。所以就有了中间件的需求。

中间件有点像面向切面的编程思想，但是与 Java 语言不同。在 Java 中，通用的处理逻辑（也可以称为切面）可以通过反射插入到正常逻辑的处理流程中，在 Go 语言中基本不这样做。

在 Go 中，中间件是通过函数闭包来实现的。Go 语言中的函数是第一类值，既可以作为参数传给其他函数，也可以作为返回值从其他函数返回。我们前面介绍了处理器/函数的使用和实现。那么可以利用闭包封装已有的处理函数。

首先，基于函数类型`func(http.Handler) http.Handler`定义一个中间件类型：

```golang
type Middleware func(http.Handler) http.Handler
```

接下来我们来编写中间件，最简单的中间件就是在请求前后各输出一条日志：

```golang
func WithLogger(handler http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    logger.Printf("path:%s process start...\n", r.URL.Path)
    defer func() {
      logger.Printf("path:%s process end...\n", r.URL.Path)
    }()
    handler.ServeHTTP(w, r)
  })
}
```

实现很简单，通过中间件封装原来的处理器对象，然后返回一个新的处理函数。在新的处理函数中，先输出开始处理的日志，然后用`defer`语句在函数结束后输出处理结束的日志。接着调用原处理器对象的`ServeHTTP()`方法执行原处理逻辑。

类似地，我们再来实现一个统计处理耗时的中间件：

```golang
func Metric(handler http.Handler) http.HandlerFunc {
  return func (w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    defer func() {
      logger.Printf("path:%s elapsed:%fs\n", r.URL.Path, time.Since(start).Seconds())
    }()
    time.Sleep(1 * time.Second)
    handler.ServeHTTP(w, r)
  }
}
```

`Metric`中间件封装原处理器对象，开始执行前记录时间，执行完成后输出耗时。为了能方便看到结果，我在上面代码中添加了一个`time.Sleep()`调用。

最后，由于请求的处理逻辑都是由功能开发人员（而非库作者）自己编写的，所以为了 Web 服务器的稳定，我们需要捕获可能出现的 panic。`PanicRecover`中间件如下：

```golang
func PanicRecover(handler http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    defer func() {
      if err := recover(); err != nil {
        logger.Println(string(debug.Stack()))
      }
    }()

    handler.ServeHTTP(w, r)
  })
}
```

调用`recover()`函数捕获 panic，输出堆栈信息，为了防止程序异常退出。实际上，在`conn.serve()`方法中也有`recover()`，程序一般不会异常退出。但是自定义的中间件可以添加我们自己的定制逻辑。

现在我们可以这样来注册处理函数：

```golang
mux.Handle("/", PanicRecover(WithLogger(Metric(http.HandlerFunc(index)))))
mux.Handle("/greeting", PanicRecover(WithLogger(Metric(greeting("welcome, dj")))))
```

这种方式略显繁琐，我们可以编写一个帮助函数，它接受原始的处理器对象，和可变的多个中间件。对处理器对象应用这些中间件，返回新的处理器对象：

```golang
func applyMiddlewares(handler http.Handler, middlewares ...Middleware) http.Handler {
  for i := len(middlewares)-1; i >= 0; i-- {
    handler = middlewares[i](handler)
  }

  return handler
}
```

注意应用顺序是**从右到左**的，即**右结合**，越靠近原处理器的越晚执行。

利用帮助函数，注册可以简化为：

```golang
middlewares := []Middleware{
  PanicRecover,
  WithLogger,
  Metric,
}
mux.Handle("/", applyMiddlewares(http.HandlerFunc(index), middlewares...))
mux.Handle("/greeting", applyMiddlewares(greeting("welcome, dj"), middlewares...))
```

上面每次注册处理逻辑都需要调用一次`applyMiddlewares()`函数，还是略显繁琐。我们可以这样来优化，封装一个自己的`ServeMux`结构，然后定义一个方法`Use()`将中间件保存下来，重写`Handle/HandleFunc`将传入的`http.HandlerFunc/http.Handler`处理器包装中间件之后再传给底层的`ServeMux.Handle()`方法：

```golang
type MyMux struct {
  *http.ServeMux
  middlewares []Middleware
}

func NewMyMux() *MyMux {
  return &MyMux{
    ServeMux: http.NewServeMux(),
  }
}

func (m *MyMux) Use(middlewares ...Middleware) {
  m.middlewares = append(m.middlewares, middlewares...)
}

func (m *MyMux) Handle(pattern string, handler http.Handler) {
  handler = applyMiddlewares(handler, m.middlewares...)
  m.ServeMux.Handle(pattern, handler)
}

func (m *MyMux) HandleFunc(pattern string, handler http.HandlerFunc) {
  newHandler := applyMiddlewares(handler, m.middlewares...)
  m.ServeMux.Handle(pattern, newHandler)
}
```

注册时只需要创建`MyMux`对象，调用其`Use()`方法传入要应用的中间件即可：

```golang
middlewares := []Middleware{
  PanicRecover,
  WithLogger,
  Metric,
}
mux := NewMyMux()
mux.Use(middlewares...)
mux.HandleFunc("/", index)
mux.Handle("/greeting", greeting("welcome, dj"))
```

这种方式简单易用，但是也有它的问题，最大的问题是必须先设置好中间件，然后才能调用`Handle/HandleFunc`注册，后添加的中间件不会对之前注册的处理器/函数生效。

为了解决这个问题，我们可以改写`ServeHTTP`方法，在确定了处理器之后再应用中间件。这样后续添加的中间件也能生效。很多第三方库都是采用这种方式。`http.ServeMux`默认的`ServeHTTP()`方法如下：

```golang
func (m *ServeMux) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  if r.RequestURI == "*" {
    if r.ProtoAtLeast(1, 1) {
      w.Header().Set("Connection", "close")
    }
    w.WriteHeader(http.StatusBadRequest)
    return
  }
  h, _ := m.Handler(r)
  h.ServeHTTP(w, r)
}
```

改造这个方法定义`MyMux`类型的`ServeHTTP()`方法也很简单，只需要在`m.Handler(r)`获取处理器之后，应用当前的中间件即可：

```golang
func (m *MyMux) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  // ...
  h, _ := m.Handler(r)
  // 只需要加这一行即可
  h = applyMiddlewares(h, m.middlewares...)
  h.ServeHTTP(w, r)
}
```

后面我们分析其他 Web 框架的源码时会发现，很多都是类似的做法。为了测试宕机恢复，编写一个会触发 panic 的处理函数：

```golang
func panics(w http.ResponseWriter, r *http.Request) {
  panic("not implemented")
}

mux.HandleFunc("/panic", panics)
```

运行，在浏览器中请求`localhost:8080/`和`localhost:8080/greeting`，最后请求`localhost:8080/panic`触发 panic：

![](/img/in-post/godailylib/nethttp3.png#center)
![](/img/in-post/godailylib/nethttp4.png#center)

## 思考题

思考题：

这其实就是看阅读代码是不是仔细，最长前缀的排序列表在`ServeMux.Handle()`方法中生成：

```golang
func (mux *ServeMux) Handle(pattern string, handler Handler) {
  if pattern[len(pattern)-1] == '/' {
    mux.es = appendSorted(mux.es, e)
  }
}
```

这里明显有个限制条件，即注册路径最后必须以`/`结尾才会触发。所以`localhost:8080/greeting/a/b/c`和`localhost:8080/a/b/c`都只会匹配`/`路径。如果想要让`localhost:8080/greeting/a/b/c`匹配路径`/greeting`，注册路径需要改为`/greeting/`：

```golang
http.Handle("/greeting/", greeting("Welcome to go web frameworks"))
```

这时请求路径`/greeting`会自动重定向（301）到`/greeting/`。

## 总结

本文介绍了使用标准库`net/http`创建 Web 服务器的基本流程，一步步分析源码。然后介绍了如何使用中间件简化通用的处理逻辑。学习并理解了`net/http`库的内容对于学习其他的 Go Web 框架非常有帮助。第三方的 Go Web 框架大多也是基于`net/http`实现自己的`ServeMux`对象而已。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~