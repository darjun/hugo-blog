---
layout:		post
title:		"Go Web 编程之 程序结构"
subtitle: 	"入门 Go Web 编程"
date:		2019-12-05T21:35:01
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go Web 编程
URL: "2019/12/05/goweb/structure"
categories: [
	"Go"
]
---

## 概述

一个典型的 Go Web 程序结构如下，摘自《Go Web 编程》：

![](/img/in-post/goweb/structure1.png)

* 客户端发送请求；
* 服务器中的多路复用器收到请求；
* 多路复用器根据请求的 URL 找到注册的处理器，将请求交由处理器处理；
* 处理器执行程序逻辑，必要时与数据库进行交互，得到处理结果；
* 处理器调用模板引擎将指定的模板和上一步得到的结果渲染成客户端可识别的数据格式（通常是 HTML）；
* 最后将数据通过响应返回给客户端；
* 客户端拿到数据，执行对应的操作，如渲染出来呈现给用户。

本文介绍如何创建多路复用器，如何注册处理器，最后再简单介绍一下 URL 匹配。我们以上一篇文章中的"Hello World"程序作为基础。

```golang
package main

import (
    "fmt"
    "log"
    "net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello World")
}

func main() {
    http.HandleFunc("/", hello)
    if err := http.ListenAndServe(":8080", nil); err != nil {
        log.Fatal(err)
    }
}
```

## 多路复用器

### 默认多路复用器

net/http 包为了方便我们使用，内置了一个默认的多路复用器`DefaultServeMux`。定义如下：

```golang
// src/net/http/server.go

// DefaultServeMux is the default ServeMux used by Serve.
var DefaultServeMux = &defaultServeMux

var defaultServeMux ServeMux
```

这里给大家介绍一下 Go 标准库代码的组织方式，便于大家对照。

* Windows上，Go 语言的默认安装目录为`C:\Go`，即`GOROOT`；
* `GOROOT`下有一个 src 目录，标库库的代码都在这个目录中；
* 每个包有一个单独的目录，例如 fmt 包在`src/fmt`目录中；
* 子包在其父包的子目录中，例如 net/http 包在`src/net/http`目录中。

net/http 包中很多方法都在内部调用`DefaultServeMux`的对应方法，如`HandleFunc`。我们知道，`HandleFunc`是为指定的 URL 注册一个处理器（准确来说，`hello`是处理器函数，见下文）。其内部实现如下：

```golang
// src/net/http/server.go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

实际上，`http.HandleFunc`方法是将处理器注册到`DefaultServeMux`中的。

另外，我们使用 ":8080" 和 `nil` 作为参数调用`http.ListenAndServe`时，会创建一个默认的服务器：

```golang
// src/net/http/server.go
func ListenAndServe(addr string, handler Handler) {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```

这个服务器默认使用`DefaultServeMux`来处理器请求：

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

服务器收到的每个请求会调用对应多路复用器（即`ServeMux`）的`ServeHTTP`方法。在`ServeMux`的`ServeHTTP`方法中，根据 URL 查找我们注册的处理器，然后将请求交由它处理。

虽然默认的多路复用器使用起来很方便，但是在生产环境中不建议使用。由于`DefaultServeMux`是一个全局变量，所有代码，**包括第三方代码**都可以修改它。
有些第三方代码会在`DefaultServeMux`注册一些处理器，这可能与我们注册的处理器冲突。

比较推荐的做法是自己创建多路复用器。

### 创建多路复用器

创建多路复用器也比较简单，直接调用`http.NewServeMux`方法即可。然后，在新创建的多路复用器上注册处理器：

```golang
package main

import (
	"fmt"
	"log"
	"net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World")
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", hello)

	server := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```

上面代码的功能与 "Hello World" 程序相同。**这里我们还自己创建了服务器对象**。通过指定服务器的参数，我们可以创建定制化的服务器。

```golang
server := &http.Server{
	Addr:           ":8080",
	Handler:        mux,
	ReadTimeout:    1 * time.Second,
	WriteTimeout:   1 * time.Second,
}
```

在上面代码，我们创建了一个读超时和写超时均为 1s 的服务器。

## 处理器和处理器函数

上文中提到，服务器收到请求后，会根据其 URL 将请求交给相应的处理器处理。处理器是实现了`Handler`接口的结构，`Handler`接口定义在 net/http 包中：

```golang
// src/net/http/server.go
type Handler interface {
    func ServeHTTP(w Response.Writer, r *Request)
}
```

我们可以定义一个实现该接口的结构，注册这个结构类型的对象到多路复用器中：

```golang
package main

import (
    "fmt"
    "log"
    "net/http"
)

type GreetingHandler struct {
    Language string
}

func (h GreetingHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "%s", h.Language)
}

func main() {
    mux := http.NewServeMux()
    mux.Handle("/chinese", GreetingHandler{Language: "你好"})
    mux.Handle("/english", GreetingHandler{Language: "Hello"})
    
    server := &http.Server {
        Addr:   ":8080",
        Handler: mux,
    }
    
    if err := server.ListenAndServe(); err != nil {
        log.Fatal(err)
    }
}
```

与前面的代码有所不同，上段代码中，定义了一个实现`Handler`接口的结构`GreetingHandler`。然后，创建该结构的两个对象，分别将它注册到多路复用器的`/hello`和`/world`路径上。注意，这里注册使用的是`Handle`方法，注意与`HandleFunc`方法对比。

启动服务器之后，在浏览器的地址栏中输入`localhost:8080/chinese`，浏览器中将显示`你好`，输入`localhost:8080/english`将显示`Hello`。

虽然，自定义处理器这种方式比较灵活，强大，但是需要定义一个新的结构，实现`ServeHTTP`方法，还是比较繁琐的。为了方便使用，net/http 包提供了以函数的方式注册处理器，即使用`HandleFunc`注册。函数必须满足签名：`func (w http.ResponseWriter, r *http.Request)`。
我们称这个函数为处理器函数。我们的 "Hello World" 程序中使用的就是这种方式。`HandleFunc`方法内部，会将传入的处理器函数转换为`HandlerFunc`类型。

```golang
// src/net/http/server.go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
```

`HandlerFunc`是底层类型为`func (w ResponseWriter, r *Request)`的新类型，它可以自定义其方法。由于`HandlerFunc`类型实现了`Handler`接口，所以它也是一个处理器类型，最终使用`Handle`注册。

```golang
// src/net/http/server.go
type HandlerFunc func(w *ResponseWriter, r *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

注意，这几个接口和方法名很容易混淆，这里再强调一下：

* `Handler`：处理器接口，定义在 net/http 包中。实现该接口的类型，其对象可以注册到多路复用器中；
* `Handle`：注册处理器的方法；
* `HandleFunc`：注册处理器函数的方法；
* `HandlerFunc`：底层类型为`func (w ResponseWriter, r *Request)`的新类型，实现了`Handler`接口。它连接了**处理器函数**与**处理器**。

## URL 匹配

一般的 Web 服务器有非常多的 URL 绑定，不同的 URL 对应不同的处理器。但是服务器是怎么决定使用哪个处理器的呢？例如，我们现在绑定了 3 个 URL，`/`和`/hello`和`/hello/world`。

显然，如果请求的 URL 为`/`，则调用`/`对应的处理器。如果请求的 URL 为`/hello`，则调用`/hello`对应的处理器。如果请求的 URL 为`/hello/world`，则调用`/hello/world`对应的处理器。
但是，如果请求的是`/hello/others`，那么使用哪一个处理器呢？ 匹配遵循以下规则：

* 首先，精确匹配。即查找是否有`/hello/others`对应的处理器。如果有，则查找结束。如果没有，执行下一步；
* 将路径中最后一个部分去掉，再次查找。即查找`/hello/`对应的处理器。如果有，则查找结束。如果没有，继续执行这一步。即查找`/`对应的处理器。

这里有一个注意点，**如果注册的 URL 不是以`/`结尾的，那么它只能精确匹配请求的 URL**。反之，即使请求的 URL 只有**前缀**与被绑定的 URL 相同，`ServeMux`也认为它们是匹配的。

这也是为什么上面步骤进行到`/hello/`时，不能匹配`/hello`的原因。因为`/hello`不以`/`结尾，必须要精确匹配。
如果，我们绑定的 URL 为`/hello/`，那么当服务器找不到与`/hello/others`完全匹配的处理器时，就会退而求其次，开始寻找能够与`/hello/`匹配的处理器。

看下面的代码：

```golang
package main

import (
	"fmt"
	"log"
	"net/http"
)

func indexHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "This is the index page")
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "This is the hello page")
}

func worldHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "This is the world page")
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", indexHandler)
	mux.HandleFunc("/hello", helloHandler)
	mux.HandleFunc("/hello/world", worldHandler)

	server := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```

* 浏览器请求`localhost:8080/`将返回`"This is the index page"`，因为`/`精确匹配；

![](/img/in-post/goweb/structure2.png#center)

* 浏览器请求`localhost:8080/hello`将返回`"This is the hello page"`，因为`/hello`精确匹配；

![](/img/in-post/goweb/structure3.png#center)

* 浏览器请求`localhost:8080/hello/`将返回`"This is the index page"`。**注意这里不是`hello`，因为绑定的`/hello`需要精确匹配，而请求的`/hello/`不能与之精确匹配。故而向上查找到`/`**；

![](/img/in-post/goweb/structure4.png#center)

* 浏览器请求`localhost:8080/hello/world`将返回`"This is the world page"`，因为`/hello/world`精确匹配；

![](/img/in-post/goweb/structure5.png#center)

* 浏览器请求`localhost:8080/hello/world/`将返回`"This is the index page"`，**查找步骤为`/hello/world/`（不能与`/hello/world`精确匹配）-> `/hello/`（不能与`/hello/`精确匹配）-> `/`**；

![](/img/in-post/goweb/structure7.png#center)

* 浏览器请求`localhost:8080/hello/other`将返回`"This is the index page"`，**查找步骤为`/hello/others` -> `/hello/`（不能与`/hello`精确匹配）-> `/`**；

![](/img/in-post/goweb/structure6.png#center)


如果注册时，将`/hello`改为`/hello/`，那么请求`localhost:8080/hello/`和`localhost:8080/hello/world/`都将返回`"This is the hello page"`。自己试试吧！

思考：
使用`/hello/`注册处理器时，`localhost:8080/hello/`返回什么？

## 参考资料

1. [Go Web 编程](https://book.douban.com/subject/27204133/)