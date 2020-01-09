---
layout:		post
title:		"Go Web 编程之 请求"
subtitle: 	"入门 Go Web 编程"
date:		2019-12-13T22:05:01
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go Web 编程
URL: "2019/12/09/goweb/request"
categories: [
	"Go"
]
---

## 概述

前面我们学习了处理器和处理器函数，如何编写和注册处理器。本文我们将学习如何从请求中获取信息。

## 请求的结构

通过前面的学习，我们知道处理器函数需要符合下面的签名：

```golang
func (w http.ResponseWriter, r *http.Request)
```

其中，`http.Request`就是请求的类型。客户端传递的数据都可以通过这个结构来获取。结构`Request`定义在包 net/http 中：

```golang
// src/net/http/request.go

type Request struct {
    Method          string    
    URL             *url.URL
    Proto           string
    ProtoMajor      int
    ProtoMinor      int
    Header          Header
    Body            io.ReadCloser
    ContentLength   int
    // 省略一些字段...
}
```

我们来看一下几个重要的字段。

### `Method`

请求中的`Method`字段表示客户端想要调用服务器的哪个方法。在[第一篇文章](https://darjun.github.io)中，我们提到过 HTTP 协议方法。其取值有`GET/POST/PUT/DELETE`等。服务器根据请求方法的不同会进行不同的处理，例如`GET`方法只是获取信息（用户基本信息，商品信息等），`POST`方法创建新的资源（注册新用户，上架新商品等）。

### `URL`

Tim Berners-Lee 在创建万维网的同时，也引入了使用字符串来表示互联网资源的概念。他称该字符串为**统一资源标识符**（URI，Uniform Resource Identifier）。URI 由两部分组成。一部分表示资源的名称，即**统一资源名称**（URN，Uniform Resource Name）。另一部分表示资源的位置，即**统一资源定位符**（URL，Uniform Resource Location）。

在 HTTP 请求中，使用 URL 来对要操作的资源位置进行描述。URL 的一般格式为：

```
[scheme:][//[userinfo@]host][/]path[?query][#fragment]
```

* `scheme`：协议名，常见的有`httphttpsftp`；
* `userInfo`：若有，则表示用户信息，如用户名和密码可写作`dj:password`；
* `host`：表示主机域名或地址，和一个可选的端口信息。若端口未指定，则默认为 80。例如`www.example.com`，`www.example.com:8080`，`127.0.0.1:8080`；
* `path`：资源在主机上的路径，以`/`分隔，如`/posts`；
* `query`：可选的查询字符串，客户端传输过来的键值对参数，键值直接用`=`，多个键值对之间用`&`连接，如`page=1&count=10`；
* `fragment`：片段，又叫锚点。表示一个页面中的位置信息。由浏览器发起的请求 URL 中，通常没有这部分信息。但是可以通过`ajax`等代码的方式发送这个数据；

我们来看一个完整的 URL：

```
http://dj:password@www.example.com/posts?page=1&count=10#fmt
```

Go 中的 URL 结构定义在`net/url`包中：

```golang
// net/url/url.go
type URL struct {
    Scheme      string
    Opaque      string
    User        *Userinfo
    Host        string
    Path        string
    RawPath     string
    RawQuery    string
    Fragment    string
}
```

可以通过请求对象中的`URL`字段获取这些信息。接下来，我们编写一个程序来具体看看（使用上一篇文章讲的 Web 程序基本结构，只需要增加处理器函数和注册即可）：

```golang
func urlHandler(w http.ResponseWriter, r *http.Request) {
    URL := r.URL
    
    fmt.Fprintf(w, "Scheme: %s\n", URL.Scheme)
    fmt.Fprintf(w, "Host: %s\n", URL.Host)
    fmt.Fprintf(w, "Path: %s\n", URL.Path)
    fmt.Fprintf(w, "RawPath: %s\n", URL.RawPath)
    fmt.Fprintf(w, "RawQuery: %s\n", URL.RawQuery)
    fmt.Fprintf(w, "Fragment: %s\n", URL.Fragment)
}

// 注册
mux.HandleFunc("/url", urlHandler)
```

运行服务器，通过浏览器访问`localhost:8080/url/posts?page=1&count=10#main`：

```
Scheme: 
Host: 
Path: /url/posts
RawPath: 
RawQuery: page=1&count=10
Fragment:
```

为什么会出现空字段？注意到源码`Request`结构中`URL`字段上有一段注释：

```
// URL specifies either the URI being requested (for server
// requests) or the URL to access (for client requests).
//
// For server requests, the URL is parsed from the URI
// supplied on the Request-Line as stored in RequestURI.  For
// most requests, fields other than Path and RawQuery will be
// empty. (See RFC 7230, Section 5.3)
//
// For client requests, the URL's Host specifies the server to
// connect to, while the Request's Host field optionally
// specifies the Host header value to send in the HTTP
// request.
```

大意是作为服务器收到的请求时，`URL`中除了`Path`和`RawQuery`，其它字段大多为空。对于这个问题，Go 的 Github 仓库上[Issue 28940](https://github.com/golang/go/issues/28940)有过讨论。

我们还可以通过`URL`结构得到一个 URL 字符串：

```golang
URL := &net.URL {
    Scheme:     "http",
    Host:       "example.com",
    Path:       "/posts",
    RawQuery:   "page=1&count=10",
    Fragment:   "main",
}
fmt.Println(URL.String())
```

上面程序运行输出字符串：

```
http://example.com/posts?page=1&count=10#main
```

### `Proto/ProtoMajor/ProtoMinor`

`Proto`表示 HTTP 协议版本，如`HTTP/1.1`，`ProtoMajor`表示大版本，`ProtoMinor`表示小版本。

```golang
func protoFunc(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Proto: %s\n", r.Proto)
    fmt.Fprintf(w, "ProtoMajor: %d\n", r.ProtoMajor)
    fmt.Fprintf(w, "ProtoMinor: %d\n", r.ProtoMinor)
}

mux.HandleFunc("/proto", protoFunc)
```

启动服务器，浏览器请求`localhost:8080`返回：

```
Proto: HTTP/1.1
ProtoMajor: 1
ProtoMinor: 1
```

当前 HTTP/1.1 是主流的版本。

### `Header`

`Header`中存放的客户端发送过来的首部信息，键-值对的形式。`Header`类型底层其实是`map[string][]string`：

```golang
// src/net/http/header.go
type Header map[string][]string
```

每个首部的键和值都是字符串，可以设置多个相同的键。注意到`Header`值为`[]string`类型，存放相同的键的多个值。浏览器发起 HTTP 请求的时候，会自动添加一些首部。我们编写一个程序来看看：

```golang
func headerHandler(w http.ResponseWriter, r *http.Request) {
    for key, value := range r.Header {
        fmt.Fprintf(w, "%s: %v\n", key, value)
    }
}

mux.HandleFunc("/header", headerHandler)
```

启动服务器，浏览器请求`localhost:8080/header`返回：

```
Accept-Encoding: [gzip, deflate, br]
Sec-Fetch-Site: [none]
Sec-Fetch-Mode: [navigate]
Connection: [keep-alive]
Upgrade-Insecure-Requests: [1]
User-Agent: [Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36]
Sec-Fetch-User: [?1]
Accept: [text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3]
Accept-Language: [zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7]
```

我使用的是 Chrome 浏览器，不同的浏览器添加的首部不完全相同。

常见的首部有：

* `Accept`：客户端想要服务器发送的内容类型；
* `Accept-Charset`：表示客户端能接受的字符编码；
* `Content-Length`：请求主体的**字节**长度，一般在 POST/PUT 请求中较多；
* `Content-Type`：当包含请求主体的时候，这个首部用于记录主体内容的类型。在发送 POST 或 PUT 请求时，内容的类型默认为`x-www-form-urlecoded`。但是在上传文件时，应该设置类型为`multipart/form-data`。
* `User-Agent`：用于描述发起请求的客户端信息，如什么浏览器。

## `Content-Length/Body`

`Content-Length`表示请求体的**字节**长度，请求体的内容可以从`Body`字段中读取。细心的朋友可能发现了`Body`字段是一个`io.ReadCloser`接口。**在读取之后要关闭它，否则会有资源泄露。可以使用`defer`简化代码编写**。

```golang
func bodyHandler(w http.ResponseWriter, r *http.Request) {
    data := make([]byte, r.ContentLength)
    r.Body.Read(data) // 忽略错误处理
    defer r.Body.Close()
    
    fmt.Fprintln(w, string(data))
}

mux.HandleFunc("/body", bodyHandler)
```

上面代码将客户端传来的请求体内容回传给客户端。还可以使用`io/ioutil`包简化读取操作：

```golang
data, _ := ioutil.ReadAll(r.Body)
```

直接在浏览器中输入 URL 发起的是`GET`请求，无法携带请求体。有很多种方式可以发起带请求体的请求，下面介绍两种：

### 使用表单

通过 HTML 的表单我们可以向服务器发送 POST 请求，将表单中的内容作为请求体发送。

```golang
func indexHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, `
<html>
    <head>
        <title>Go Web 编程之 request</title>
    </head>
    <body>
        <form method="post" action="/body">
            <label for="username">用户名：</label>
            <input type="text" id="username" name="username">
            <label for="email">邮箱：</label>
            <input type="text" id="email" name="email">
            <button type="submit">提交</button>
        </form>
    </body>
</html>
`)
}

mux.HandleFunc("/", indexHandler)
```

在 HTML 中使用`form`来显示一个表单。点击提交按钮后，浏览器会发送一个 POST 请求到路径`/body`上，将用户名和邮箱作为请求包体。

启动服务器，进入主页`localhost:8080/`，显示表单。填写信息，点击提交：

![](/img/in-post/goweb/request1.png#center)

浏览器向服务器发送 POST 请求，URL 为`/body`，`bodyHandler`处理完成后将包体回传给客户端。最后客户端显示：

![](/img/in-post/goweb/request2.png#center)

上面的数据使用了`x-www-form-urlencoded`编码，这是表单的默认编码。后文还有详述。

### 使用 Postman

[Postman](https://www.getpostman.com/) 是一款功能非常强大的 API 测试工具。

* 支持 HTTP 协议的所有方法请求（`GET/POST/PUT/DELETE`）。
* 可以在请求中携带首部信息，请求体的内容；
* 支持`json/xml/http`等各种格式的内容；
* 界面友好。

接下来我们看看如何使用 PostMan 测试我们的`bodyHandler`。

![](/img/in-post/goweb/request3.png#center)

* 黑色部分：选择 HTTP 协议方法，这里选择 POST 以便可以携带请求体；
* 绿色部分：请求的 URL；
* 蓝色部分：可以设置请求的首部，请求体；
* 淡红色部分：请求体支持多种格式，这里选择原始格式；
* 灰色部分：请求体的具体内容；
* 红色部分：发送之后显示的响应信息，可以查看响应首部，Cookie，响应体等。可以看到是原样返回。

## 获取请求参数

上面我们分析了 Go 中 HTTP 请求的常见字段。在实际开发中，客户端通常需要在请求中传递一些参数。参数传递的方式一般有两种方式：

* URL 中的键值对，又叫查询字符串，即 query string；
* 表单。

下面依次来介绍。

### URL 键值对

前文中介绍 URL 的一般格式时提到过，URL 的后面可以跟一个可选的查询字符串，以`?`与路径分隔，形如`key1=value1&key2=value2`。

URL 结构中有一个`RawQuery`字段。这个字段就是查询字符串。

```
func queryHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, r.URL.RawQuery)
}

mux.HandleFunc("/query", queryHandler)
```

如果我们以`localhost:8080/query?name=dj&age=20`请求，查询字符串`name=dj&age=20`会原样传回客户端。但是`RawQuery`是字符串类型的，使用字符串方法解析也能用，但是太麻烦了！！！

![](/img/in-post/goweb/request4.png)

### 表单

表单狭义上说是通过表单发送请求，广义上说可以将数据放在请求体中发送到服务器。接下来我们简单编写一个 HTML 页面，通过页面表单发送 HTTP 请求：

```html
<html>
    <head>
        <title>Go Web 编程之 request</title>
    </head>
    
    <body>
        <form action="/form?lang=cpp&name=dj" method="post" enctype="application/x-www-form-urlencoded">
            <label>Form:</label>
            <input type="text" name="lang" />
            <input type="text" name="age" />
            <button type="submit">提交</button>
        </form>
    </body>
</html>
```

* `action`表示提交表单时请求的 URL，`method`表示请求的方法。**如果使用`GET`请求，由于`GET`方法没有请求体，参数将会拼接到 URL 尾部**；
* `enctype`指定请求体的编码方式，默认为`application/x-www-form-urlencoded`。如果需要发送文件，必须指定为`multipart/form-data`；

我们介绍一下什么是`urlencoded`编码。RFC 3986 中定义了 URL 中的保留字以及非保留字，所有保留字符都需要进行 URL 编码。URL 编码会把字符转换成它在 ASCII 编码中对应的字节值，接着把这个字节值表示为一个两位长的十六进制数字，最后在这个数字前面加上一个百分号（%）。例如空格的 ASCII 编码为 32，十六进制为 20，故 URL 编码为`%20`。

#### `Form`字段

使用`x-www-form-urlencoded`编码的请求体，在处理时首先调用请求的`ParseForm`方法解析，然后从`Form`字段中取数据：

```golang
func formHandler(w http.ResponseWriter, r *http.Request) {
    r.ParseForm()
    fmt.Fprintln(w, r.Form)
}

mux.HandleFunc("/form", formHandler)
```

运行程序，验证结果：

![](/img/in-post/goweb/request5.png#center)

![](/img/in-post/goweb/request6.png#center)

`Form`字段的类型`url.Values`底层实际上是`map[string][]string`。调用`ParseForm`方法之后，可以使用`url.Values`的方法操作数据。

使用`ParseForm`还能解析查询字符串，将上面的表单改为：

```html
<html>
    <head>
        <title>Go Web 编程之 request</title>
    </head>
    
    <body>
        <form action="/form?lang=cpp&name=dj" method="post" enctype="application/x-www-form-urlencoded">
            <label>Form:</label>
            <input type="text" name="lang" />
            <input type="text" name="age" />
            <button type="submit">提交</button>
        </form>
    </body>
</html>
```

请求结果：

![](/img/in-post/goweb/request7.png#center)

![](/img/in-post/goweb/request8.png#center)

可以看出，查询字符串中的键值对和表单中解析处理的合并到一起了。同一个键下，表单值总是排在前面，如`[golang cpp]`。

#### `PostForm`字段

如果一个请求，同时有 URL 键值对和表单数据，而用户只想获取表单数据，可以使用`PostForm`字段。

使用`PostForm`只会返回表单数据，不包括 URL 键值。如果把上面的程序中，`r.Form`改为`r.PostForm`，那么程序将显示以下结果：

![](/img/in-post/goweb/request9.png#center)

![](/img/in-post/goweb/request10.png#center)

#### `MultipartForm`字段

如果要处理上传的文件，那么就必须使用`multipart/form-data`编码。与之前的`Form/PostForm`类似，处理`multipart/form-data`编码的请求时，也需要先解析后使用。只不过使用的方法不同，解析使用`ParseMultipartForm`，之后从`MultipartForm`字段取值。

```html
<form action="/multipartform?lang=cpp&name=dj" method="post" enctype="multipart/form-data">
	<label>MultipartForm:</label>
    <input type="text" name="lang" />
    <input type="text" name="age" />
    <input type="file" name="uploaded" />
    <button type="submit">提交</button>
</form>
```

```golang
func multipartFormHandler(w http.ResponseWriter, r *http.Request) {
    r.ParseMultipartForm(1024)
    fmt.Fprintln(w, r.MultipartForm)
    
    fileHeader := r.MultipartForm.File["uploaded"][0]
	file, err := fileHeader.Open()
	if err != nil {
		fmt.Println("Open failed: ", err)
		return
	}

	data, err := ioutil.ReadAll(file)
	if err == nil {
		fmt.Fprintln(w, string(data))
	}
}

mux.HandleFunc("/multipartform", multipartFormHandler)
```

运行程序：

![](/img/in-post/goweb/request11.png#center)

![](/img/in-post/goweb/request12.png#center)

`MultipartForm`包含两个`map`类型的字段，一个表示表单键值对，另一个为上传的文件信息。

使用表单中文件控件名获取`MultipartForm.File`得到通过该控件上传的文件，可以是多个。得到的是`multipart.FileHeader`类型，通过该类型可以获取文件的各个属性。

需要注意的是，这种方式用来处理文件。为了安全，`ParseMultipartForm`方法需要传一个参数，表示最大使用内存，避免上传的文件占用空间过大。

#### `FormValue/PostFormValue`

为了方便地获取值，`net/http`包提供了`FormValue/PostFormValue`方法。它们在需要时会自动调用`ParseForm/ParseMultipartForm`方法。

`FormValue`方法返回请求的`Form`字段中指定键的值。**如果同一个键对应多个值，那么返回第一个**。如果需要获取全部值，直接使用`Form`字段。下面代码将返回`hello`对应的第一个值：

```golang
fmt.Fprintln(w, r.FormValue("hello"))
```

`PostFormValue`方法返回请求的`PostForm`字段中指定键的值。**如果同一个键对应多个值，那么返回第一个**。如果需要获取全部值，直接使用`PostForm`字段

注意：
当编码被指定为`multipart/form-data`时，`FormValue/PostFormValue`将不会返回任何值，它们读取的是`Form/PostForm`字段，而`ParseMultipartForm`将数据写入`MultipartForm`字段。

#### 其他格式

通过 AJAX 之类的技术可以发送其它格式的数据，例如`application/json`等。这种情况下：

* 首先通过首部`Content-Type`来获知具体是什么格式；
* 通过`r.Body`读取字节流；
* 解码使用。
 
## 总结

本文介绍了`net/http`包中请求的各方面内容。从`Request`结构到如何传递参数，最后介绍各种编码的请求如何处理。

## 参考

1. [Go Web 编程](https://book.douban.com/subject/27204133/)
2. [net/http](https://golang.org/pkg/net/http/)标准库文档

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)