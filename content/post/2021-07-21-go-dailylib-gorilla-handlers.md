---
layout:    post
title:    "Go 每日一库之 gorilla/handlers"
subtitle: "每天学习一个 Go 库"
date:     2021-07-21T06:30:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/07/21/godailylib/gorilla/handlers"
categories: [
  "Go"
]
---

## 简介

上一篇文章中，我们介绍了 gorilla web 开发工具包中的路由管理库[`gorilla/mux`](https://github.com/gorilla/mux)，在文章最后我们介绍了如何使用中间件处理通用的逻辑。在日常 Go Web 开发中，开发者遇到了很多相同的中间件需求，[gorilla/handlers](https://github.com/gorilla/handlers)（后文简称为`handlers`）收集了一些比较常用的中间件。一起来看看吧~

关于中间件，前面几篇文章已经介绍的很多了。这里就不赘述了。`handlers`库提供的中间件可用于标准库`net/http`和所有支持`http.Handler`接口的框架。由于`gorilla/mux`也支持`http.Handler`接口，所以也可以与`handlers`库结合使用。**这就是兼容标准的好处**。

## 项目初始化&安装

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir gorilla/handlers && cd gorilla/handlers
$ go mod init github.com/darjun/go-daily-lib/gorilla/handlers
```

安装`gorilla/handlers`库：

```cmd
$ go get -u github.com/gorilla/handlers
```

下面依次介绍各个中间件和相应的源码。

## 日志

`handlers`提供了两个日志中间件：

* `LoggingHandler`：以 Apache 的`Common Log Format`日志格式记录 HTTP 请求日志；
* `CombinedLoggingHandler`：以 Apache的`Combined Log Format`日志格式记录 HTTP 请求日志，Apache 和 Nginx 默认都使用这种日志格式。

两种日志格式差别很小，`Common Log Format`格式如下：

```golang
%h %l %u %t "%r" %>s %b
```

各个指示符含义如下：

* `%h`：客户端的 IP 地址或主机名；
* `%l`：`RFC 1413`定义的客户端标识，由客户端机器上的`identd`程序生成。如果不存在，则该字段为`-`；
* `%u`：已验证的用户名。如果不存在，该字段为`-`；
* `%t`：时间，格式为`day/month/year:hour:minute:second zone`，其中：
  * `day`： 2位数字；
  * `month`：月份缩写，3个字母，如`Jan`；
  * `year`：4位数字；
  * `hour`：2位数字；
  * `minute`：2位数字；
  * `second`：2位数字；
  * `zone`：`+`或`-`后跟4位数字；
  * 例如：`21/Jul/2021:06:27:33 +0800`
* `%r`：包含 HTTP 请求行信息，例`GET /index.html HTTP/1.1`；
* `%>s`：服务器发送给客户端的状态码，例如`200`；
* `%b`：响应长度（字节数）。

`Combined Log Format`格式如下：

```golang
%h %l %u %t "%r" %>s %b "%{Referer}i" "%{User-Agent}i"
```

可见相比`Common Log Format`只是多了：

* `%{Referer}i`：HTTP 首部中的`Referer`信息；
* `%{User-Agent}i`：HTTP 首部中的`User-Agent`信息。

对中间件，我们可以让它作用于全局，即全部处理器，也可以让它只对某些处理器生效。如果要对所有处理器生效，可以调用`Use()`方法。如果只需要作用于特定的处理器，在注册时用中间件将处理器包装一层：

```golang
func index(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, "Hello World")
}

type greeting string

func (g greeting) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "Welcome, %s", g)
}

func main() {
  r := mux.NewRouter()
  r.Handle("/", handlers.LoggingHandler(os.Stdout, http.HandlerFunc(index)))
  r.Handle("/greeting", handlers.CombinedLoggingHandler(os.Stdout, greeting("dj")))

  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

上面代码中`LoggingHandler`只作用于处理函数`index`，`CombinedLoggingHandler`只作用于处理器`greeting("dj")`。

运行代码，通过浏览器访问`localhost:8080`和`localhost:8080/greeting`：

```golang
::1 - - [21/Jul/2021:06:39:45 +0800] "GET / HTTP/1.1" 200 12
::1 - - [21/Jul/2021:06:39:54 +0800] "GET /greeting HTTP/1.1" 200 11 "" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.164 Safari/537.36"
```

对照前面分析的指示符，很容易看出各个部分。

由于`*mux.Router`的`Use()`方法接受类型为`MiddlewareFunc`的中间件：

```golang
type MiddlewareFunc func(http.Handler) http.Handler
```

而`handlers.LoggingHandler/CombinedLoggingHandler`并不满足，所以还需要包装一层才能传给`Use()`方法：

```golang
func Logging(handler http.Handler) http.Handler {
  return handlers.CombinedLoggingHandler(os.Stdout, handler)
}

func main() {
  r := mux.NewRouter()
  r.Use(Logging)
  r.HandleFunc("/", index)
  r.Handle("/greeting/", greeting("dj"))

  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

另外`handlers`还提供了`CustomLoggingHandler`，我们可以利用它定义自己的日志中间件：

```golang
func CustomLoggingHandler(out io.Writer, h http.Handler, f LogFormatter) http.Handler
```

最关键的`LogFormatter`类型定义：

```golang
type LogFormatterParams struct {
  Request    *http.Request
  URL        url.URL
  TimeStamp  time.Time
  StatusCode int
  Size       int
}

type LogFormatter func(writer io.Writer, params LogFormatterParams)
```

我们实现一个简单的`LogFormatter`，记录时间 + 请求行 + 响应码：

```golang
func myLogFormatter(writer io.Writer, params handlers.LogFormatterParams) {
  var buf bytes.Buffer
  buf.WriteString(time.Now().Format("2006-01-02 15:04:05 -0700"))
  buf.WriteString(fmt.Sprintf(` "%s %s %s" `, params.Request.Method, params.URL.Path, params.Request.Proto))
  buf.WriteString(strconv.Itoa(params.StatusCode))
  buf.WriteByte('\n')

  writer.Write(buf.Bytes())
}

func Logging(handler http.Handler) http.Handler {
  return handlers.CustomLoggingHandler(os.Stdout, handler, myLogFormatter)
}
```

使用：

```golang
func main() {
  r := mux.NewRouter()
  r.Use(Logging)
  r.HandleFunc("/", index)
  r.Handle("/greeting/", greeting("dj"))

  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

现在记录的日志是下面这种格式：

```golang
2021-07-21 07:03:18 +0800 "GET /greeting/ HTTP/1.1" 200
```

翻看源码，我们可以发现`LoggingHandler/CombinedLoggingHandler/CustomLoggingHandler`都是基于底层的`loggingHandler`实现的，不同的是`LoggingHandler`使用了预定义的`writeLog`作为`LogFormatter`，`CombinedLoggingHandler`使用了预定义的`writeCombinedLog`作为`LogFormatter`，而`CustomLoggingHandler`使用我们自己定义的`LogFormatter`：

```golang
func CombinedLoggingHandler(out io.Writer, h http.Handler) http.Handler {
  return loggingHandler{out, h, writeCombinedLog}
}

func LoggingHandler(out io.Writer, h http.Handler) http.Handler {
  return loggingHandler{out, h, writeLog}
}

func CustomLoggingHandler(out io.Writer, h http.Handler, f LogFormatter) http.Handler {
  return loggingHandler{out, h, f}
}
```

预定义的`writeLog/writeCombinedLog`实现如下：

```golang
func writeLog(writer io.Writer, params LogFormatterParams) {
  buf := buildCommonLogLine(params.Request, params.URL, params.TimeStamp, params.StatusCode, params.Size)
  buf = append(buf, '\n')
  writer.Write(buf)
}

func writeCombinedLog(writer io.Writer, params LogFormatterParams) {
  buf := buildCommonLogLine(params.Request, params.URL, params.TimeStamp, params.StatusCode, params.Size)
  buf = append(buf, ` "`...)
  buf = appendQuoted(buf, params.Request.Referer())
  buf = append(buf, `" "`...)
  buf = appendQuoted(buf, params.Request.UserAgent())
  buf = append(buf, '"', '\n')
  writer.Write(buf)
}
```

它们都是基于`buildCommonLogLine`构造基本信息，`writeCombinedLog`还分别调用`http.Request.Referer()`和`http.Request.UserAgent`获取了`Referer`和`User-Agent`信息。

`loggingHandler`定义如下：

```golang
type loggingHandler struct {
  writer    io.Writer
  handler   http.Handler
  formatter LogFormatter
}
```

`loggingHandler`实现有一个比较巧妙的地方：为了记录响应码和响应大小，定义了一个类型`responseLogger`包装原来的`http.ResponseWriter`，在写入时记录信息：

```golang
type responseLogger struct {
  w      http.ResponseWriter
  status int
  size   int
}

func (l *responseLogger) Write(b []byte) (int, error) {
  size, err := l.w.Write(b)
  l.size += size
  return size, err
}

func (l *responseLogger) WriteHeader(s int) {
  l.w.WriteHeader(s)
  l.status = s
}

func (l *responseLogger) Status() int {
  return l.status
}

func (l *responseLogger) Size() int {
  return l.size
}
```

`loggingHandler`的关键方法`ServeHTTP()`：

```golang
func (h loggingHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  t := time.Now()
  logger, w := makeLogger(w)
  url := *req.URL

  h.handler.ServeHTTP(w, req)
  if req.MultipartForm != nil {
    req.MultipartForm.RemoveAll()
  }

  params := LogFormatterParams{
    Request:    req,
    URL:        url,
    TimeStamp:  t,
    StatusCode: logger.Status(),
    Size:       logger.Size(),
  }

  h.formatter(h.writer, params)
}
```

构造`LogFormatterParams`对象，调用对应的`LogFormatter`函数。

## 压缩

如果客户端请求中有`Accept-Encoding`首部，服务器可以使用该首部指示的算法将响应压缩，以节省网络流量。`handlers.CompressHandler`中间件启用压缩功能。还有一个`CompressHandlerLevel`可以指定压缩级别。实际上`CompressHandler`就是使用`gzip.DefaultCompression`调用的`CompressHandlerLevel`：

```golang
func CompressHandler(h http.Handler) http.Handler {
  return CompressHandlerLevel(h, gzip.DefaultCompression)
}
```

看代码：

```golang
func index(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, "Hello World")
}

type greeting string

func (g greeting) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "Welcome, %s", g)
}

func main() {
  r := mux.NewRouter()
  r.Use(handlers.CompressHandler)
  r.HandleFunc("/", index)
  r.Handle("/greeting/", greeting("dj"))

  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

运行，请求`localhost:8080`，通过 Chrome 开发者工具的 Network 页签可以看到响应采用了 gzip 压缩：

![](/img/in-post/godailylib/handlers1.png#center)

忽略一些细节处理，`CompressHandlerLevel`函数代码如下：

```golang
func CompressHandlerLevel(h http.Handler, level int) http.Handler {
  const (
    gzipEncoding  = "gzip"
    flateEncoding = "deflate"
  )

  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    var encoding string
    for _, curEnc := range strings.Split(r.Header.Get(acceptEncoding), ",") {
      curEnc = strings.TrimSpace(curEnc)
      if curEnc == gzipEncoding || curEnc == flateEncoding {
        encoding = curEnc
        break
      }
    }

    if encoding == "" {
      h.ServeHTTP(w, r)
      return
    }

    if r.Header.Get("Upgrade") != "" {
      h.ServeHTTP(w, r)
      return
    }

    var encWriter io.WriteCloser
    if encoding == gzipEncoding {
      encWriter, _ = gzip.NewWriterLevel(w, level)
    } else if encoding == flateEncoding {
      encWriter, _ = flate.NewWriter(w, level)
    }
    defer encWriter.Close()

    w.Header().Set("Content-Encoding", encoding)
    r.Header.Del(acceptEncoding)

    cw := &compressResponseWriter{
      w:          w,
      compressor: encWriter,
    }

    w = httpsnoop.Wrap(w, httpsnoop.Hooks{
      Write: func(httpsnoop.WriteFunc) httpsnoop.WriteFunc {
        return cw.Write
      },
      WriteHeader: func(httpsnoop.WriteHeaderFunc) httpsnoop.WriteHeaderFunc {
        return cw.WriteHeader
      },
      Flush: func(httpsnoop.FlushFunc) httpsnoop.FlushFunc {
        return cw.Flush
      },
      ReadFrom: func(rff httpsnoop.ReadFromFunc) httpsnoop.ReadFromFunc {
        return cw.ReadFrom
      },
    })

    h.ServeHTTP(w, r)
  })
}
```

从请求`Accept-Encoding`首部中获取客户端指示的压缩算法。如果客户端未指定，或请求首部中有`Upgrade`，则不压缩。反之，则压缩。根据识别的压缩算法，创建对应`gzip`或`flate`的`io.Writer`实现对象。

与前面的日志中间件一样，为了压缩写入的内容，新增类型`compressResponseWriter`封装`http.ResponseWriter`，重写`Write()`方法，将写入的字节流传入前面创建的`io.Writer`实现压缩：

```golang
type compressResponseWriter struct {
  compressor io.Writer
  w          http.ResponseWriter
}

func (cw *compressResponseWriter) Write(b []byte) (int, error) {
  h := cw.w.Header()
  if h.Get("Content-Type") == "" {
    h.Set("Content-Type", http.DetectContentType(b))
  }
  h.Del("Content-Length")

  return cw.compressor.Write(b)
}
```

## 内容类型

我们可以通过`handler.ContentTypeHandler`指定请求的`Content-Type`必须在我们给出的类型中，只对`POST/PUT/PATCH`方法生效。例如我们限制登录请求必须通过`application/x-www-form-urlencoded`的形式发送：

```golang
func main() {
  r := mux.NewRouter()
  r.HandleFunc("/", index)
  r.Methods("GET").Path("/login").HandlerFunc(login)
  r.Methods("POST").Path("/login").
    Handler(handlers.ContentTypeHandler(http.HandlerFunc(dologin), "application/x-www-form-urlencoded"))

  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

这样，只要请求`/login`的`Content-Type`不是`application/x-www-form-urlencoded`就会返回 415 错误。我们可以故意写错，再请求看看表现：

```golang
Unsupported content type "application/x-www-form-urlencoded"; expected one of ["application/x-www-from-urlencoded"]
```

`ContentTypeHandler`的实现非常简单：

```golang
func ContentTypeHandler(h http.Handler, contentTypes ...string) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if !(r.Method == "PUT" || r.Method == "POST" || r.Method == "PATCH") {
      h.ServeHTTP(w, r)
      return
    }

    for _, ct := range contentTypes {
      if isContentType(r.Header, ct) {
        h.ServeHTTP(w, r)
        return
      }
    }
    http.Error(w, fmt.Sprintf("Unsupported content type %q; expected one of %q", r.Header.Get("Content-Type"), contentTypes), http.StatusUnsupportedMediaType)
  })
}
```

就是读取`Content-Type`首部，判断是否在我们指定的类型中。

## 方法分发器

在上面的例子中，我们注册路径`/login`的`GET`和`POST`方法处理采用`r.Methods("GET").Path("/login").HandlerFunc(login)`这种冗长的写法。`handlers.MethodHandler`可以简化这种写法：

```golang
func main() {
  r := mux.NewRouter()
  r.HandleFunc("/", index)
  r.Handle("/login", handlers.MethodHandler{
    "GET":  http.HandlerFunc(login),
    "POST": http.HandlerFunc(dologin),
  })

  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

`MethodHandler`底层是一个`map[string]http.Handler`类型，它的`ServeHTTP()`方法根据请求的 Method 调用不同的处理：

```golang
type MethodHandler map[string]http.Handler

func (h MethodHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  if handler, ok := h[req.Method]; ok {
    handler.ServeHTTP(w, req)
  } else {
    allow := []string{}
    for k := range h {
      allow = append(allow, k)
    }
    sort.Strings(allow)
    w.Header().Set("Allow", strings.Join(allow, ", "))
    if req.Method == "OPTIONS" {
      w.WriteHeader(http.StatusOK)
    } else {
      http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
  }
}
```

方法如果未注册，则返回`405 Method Not Allowed`。有一个方法除外，`OPTIONS`。该方法通过`Allow`首部返回支持哪些方法。

## 重定向

`handlers.CanonicalHost`可以将请求重定向到指定的域名，同时指定重定向响应码。在同一个服务器对应多个域名时比较有用：

```golang
func index(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, "hello world")
}

func main() {
  r := mux.NewRouter()
  r.Use(handlers.CanonicalHost("http://www.gorillatoolkit.org", 302))
  r.HandleFunc("/", index)
  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

上面将所有请求以 302 重定向到`http://www.gorillatoolkit.org`。

`CanonicalHost`的实现也很简单：

```golang
func CanonicalHost(domain string, code int) func(h http.Handler) http.Handler {
  fn := func(h http.Handler) http.Handler {
    return canonical{h, domain, code}
  }

  return fn
}
```

关键类型`canonical`：

```golang
type canonical struct {
  h      http.Handler
  domain string
  code   int
}
```

核心方法：

```golang
func (c canonical) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  dest, err := url.Parse(c.domain)
  if err != nil {
    c.h.ServeHTTP(w, r)
    return
  }

  if dest.Scheme == "" || dest.Host == "" {
    c.h.ServeHTTP(w, r)
    return
  }

  if !strings.EqualFold(cleanHost(r.Host), dest.Host) {
    dest := dest.Scheme + "://" + dest.Host + r.URL.Path
    if r.URL.RawQuery != "" {
      dest += "?" + r.URL.RawQuery
    }
    http.Redirect(w, r, dest, c.code)
    return
  }

  c.h.ServeHTTP(w, r)
}
```

由源码可知，域名不合法或未指定协议（Scheme）或域名（Host）的请求下不转发。

## Recovery

之前我们自己实现了`PanicRecover`中间件，避免请求处理时 panic。`handlers`提供了一个`RecoveryHandler`可以直接使用：

```golang
func PANIC(w http.ResponseWriter, r *http.Request) {
  panic(errors.New("unexpected error"))
}

func main() {
  r := mux.NewRouter()
  r.Use(handlers.RecoveryHandler(handlers.PrintRecoveryStack(true)))
  r.HandleFunc("/", PANIC)
  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

选项`PrintRecoveryStack`表示 panic 时输出堆栈信息。

`RecoveryHandler`的实现与之前我们自己编写的基本一样：

```golang
type recoveryHandler struct {
  handler    http.Handler
  logger     RecoveryHandlerLogger
  printStack bool
}

func (h recoveryHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  defer func() {
    if err := recover(); err != nil {
      w.WriteHeader(http.StatusInternalServerError)
      h.log(err)
    }
  }()

  h.handler.ServeHTTP(w, req)
}
```


## 总结

GitHub 上有很多开源的 Go Web 中间件实现，可以直接拿来使用，避免重复造轮子。`handlers`很轻量，容易与标准库`net/http`和 gorilla 路由库`mux`结合使用。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gorilla/handlers GitHub：[github.com/gorilla/handlers](https://github.com/gorilla/handlers)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)