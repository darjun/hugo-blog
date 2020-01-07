---
layout:		post
title:		"Go Web 编程之 响应"
subtitle: 	"入门 Go Web 编程"
date:		2019-12-18T21:41:43
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go Web 编程
URL: "2019/12/18/goweb/response"
categories: [
	"Go"
]
---

## 概述

上一篇文章中，我们介绍了请求的结构与处理。本文将详细介绍如何响应客户端的请求。其实在前面几篇文章中，我们已经使用过响应的功能——通过`http.ResponseWriter`发送字符串给客户端。
但是这种方式仅限于发送字符串。本文我们将介绍如何定制响应的参数。

## `ResponseWriter`接口

如果你看了我前面几篇文章，应该对处理器和处理器函数都非常熟悉了。处理器函数即拥有以下签名的函数：

```golang
func (w http.ResponseWriter, r *http.Request)
```

这里的`ResponseWriter`其实是定义在`net/http`包中的一个接口：

```golang
// src/net/http/
type ReponseWriter interface {
    Header() Header
    Write([]byte) (int, error)
    WriteHeader(statusCode int)
}
```

我们响应客户端请求都是通过该接口的 3 个方法进行的。例如之前`fmt.Fprintln(w, "Hello World")`其实底层调用了`Write`方法。

收到请求后，多路复用器会自动创建一个`http.response`对象，它实现了`http.ResponseWriter`接口，然后将该对象和请求对象作为参数传给处理器。那为什么请求对象使用的时结构指针`*http.Request`，而响应要使用接口呢？

实际上，请求对象使用指针是为了能在处理逻辑中方便地获取请求信息。而响应使用接口来操作，一方面底层也是对象指针，可以保存修改。另一方面，我认为是为了扩展性。可以很方便地用新的实现替换而不用修改应用层代码，即处理器接口不用修改。例如，Go 标准库提供了一个测试 HTTP 请求的工具包`net/http/httptest`。它定义了一个`ResponseRecorder`结构，该结构实现了接口`http.ResponseWriter`。这个结构不将写入的数据发送给客户端，**而是将数据记录下来，方便测试断言**。

接口`ResponseWriter`有 3 个方法，下面依次来介绍如何使用：

* `Write`；
* `WriteHeader`；
* `Header`。

### `Write`方法

由于接口`ResponseWriter`拥有方法`Write([]byte) (int, error)`，所以实现了`ResponseWriter`接口的结构也实现了`io.Writer`接口：

```golang
// src/io/io.go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

这也是为什么`http.ResponseWriter`类型的变量`w`能在下面代码中使用的原因（`fmt.Fprintln`的第一个参数接收一个`io.Writer`接口）：

```golang
fmt.Fprintln(w, "Hello World")
```

我们也可以直接调用`Write`方法来向响应中写入数据：

```golang

func writeHandler(w http.ResponseWriter, r *http.Request) {
    str := `<html>
<head><title>Go Web 编程之 响应</title></head>
<body><h1>直接使用 Write 方法<h1></body>
</html>`
    w.Write([]byte(str))
}

mux.HandleFunc("/write", writeHandler)
```

下面，我们介绍一个工具`curl`来测试我们的 Web 应用。由于浏览器只会展示响应中主体的内容，其它元信息需要进行一些操作才能查看，不够直观。`curl`是一个 Linux 命令行程序，可用来发起 HTTP 请求，功能非常强大，如设置首部/请求体，展示响应首部等。

通常 Linux 系统会自带`curl`命令。简单介绍几种 Windows 上安装`curl`的方式。

* 直接在[curl官网](https://curl.haxx.se/download.html)下载可执行程序，下载完成后放在`PATH`目录中即可在`Cmd`或`Powershell`界面中使用；

* Windows 提供了一个软件包管理工具[chocolatey](https://chocolatey.org/)，可以安装/更新/删除 Windows 软件。安装`chocolatey`后，直接在`Cmd`或`Powershell`界面执行以下命令即可安装`curl`，也比较方便：

```
choco install curl
```

* 我想作为程序员，每个人都应该熟悉`git`。安装[git for windows](https://git-scm.com/download/win)后，就可以直接在`Git Bash`中使用`curl`命令。实际上，`git for windows`使用了`mingw`来在 Windows 上模拟 Linux 环境。它提供了很多 Linux 命令的 Windows 版本，非常推荐使用。

启动服务器，使用下面命令测试`Write`方法：

```shell
curl -i localhost:8080/write
```

选项`-i`的作用是显示响应首部。该命令返回：

```
HTTP/1.1 200 OK
Date: Thu, 19 Dec 2019 13:36:32 GMT
Content-Length: 113
Content-Type: text/html; charset=utf-8

<html>
<head><title>Go Web 编程之 响应</title></head>
<body><h1>直接使用 Write 方法<h1></body>
</html>
```

可以看出很清晰地看出响应的各个部分。也可以继续使用浏览器来测试：

![](/img/in-post/goweb/response1.png#center)

但是如果要查看首部，状态码等信息就必须使用浏览器的开发者工具了。Chrome 的开发者工具可以通过 F12 唤出，然后切换到`Network`标签，点击刚刚发送的请求：

![](/img/in-post/goweb/response2.png#center)

![](/img/in-post/goweb/response3.png#center)

我们看到上面红色的两个部分为响应的元信息，下面的绿色部分为请求的基本信息。

注意到，如果我们没有设置响应码，则响应码默认为**200**。
而且我们也没有设置内容类型，但是返回的首部中有`Content-Type: text/html; charset=utf-8`，说明`net/http`会自动推断。`net/http`包是通过读取响应体中前面的若干个字节来推断的，并不是百分百准确的。

如何设置状态码和响应内容的类型呢？这就是`WriteHeader`和`Header()`两个方法的作用。

### `WriteHeader`方法

`WriteHeader`方法的名字带有一点误导性，它并不能用于设置响应首部。`WriteHeader`接收一个整数，并将这个整数作为 HTTP 响应的状态码返回。调用这个返回之后，可以继续对`ResponseWriter`进行写入，但是**不能对响应的首部进行任何修改操作**。如果用户在调用`Write`方法之前没有执行过`WriteHeader`方法，那么程序默认会使用 200 作为响应的状态码。

如果，我们定义了一个 API，还未定义其实现。那么请求这个 API 时，可以返回一个 501 Not Implemented 作为状态码。

```golang
func writeHeaderHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(501)
    fmt.Fprintln(w, "This API not implemented!!!")
}

mux.HandleFunc("/writeheader", writeHeaderHandler)
```

使用`curl`来测试刚刚编写的处理器：

```
curl -i localhost:8080/writeheader
```

返回：

```
HTTP/1.1 501 Not Implemented
Date: Thu, 19 Dec 2019 14:15:16 GMT
Content-Length: 28
Content-Type: text/plain; charset=utf-8

This API not implemented!!!
```

### `Header`方法

`Header`方法其实返回的是一个`http.Header`类型，该类型的底层类型为`map[string][]string`：

```golang
// src/net/http/header.go
type Header map[string][]string
```

类型`Header`定义了 CRUD 方法，可以通过这些方法操作首部。

```golang
func headerHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Location", "http://baidu.com")
    w.WriteHeader(302)
}
```

通过第一篇文章我们知道 302 表示重定向，浏览器收到该状态码时会再发起一个请求到首部中`Location`指向的地址。使用`curl`测试：

```
curl -i localhost:8080/header
```

返回：

```
HTTP/1.1 302 Found
Location: http://baidu.com
Date: Thu, 19 Dec 2019 14:17:49 GMT
Content-Length: 0
```

如何在浏览器中打开`localhost:8080/header`，网页会重定向到[百度首页](http://baidu.com)。

接下来，我们看看如何设置自定义的内容类型。通过`Header.Set`方法设置响应的首部`Contet-Type`即可。我们编写一个返回 JSON 数据的处理器：

```golang
type User struct {
    FirstName   string      `json:"first_name"`
    LastName    string      `json:"last_name"`
    Age         int         `json:"age"`
    Hobbies     []string    `json:"hobbies"`
}

func jsonHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    u := &User {
        FirstName:  "lee",
        LastName:   "darjun",
        Age:        18,
        Hobbies:    []string{"coding", "math"},
    }
    data, _ := json.Marshal(u)
    w.Write(data)
}

mux.HandleFunc("/json", jsonHandler)
```

通过`curl`发送请求：

```
curl -i localhost:8080/json
```

返回：

```
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 19 Dec 2019 14:31:03 GMT
Content-Length: 78

{"first_name":"lee","last_name":"darjun","age":18,"hobbies":["coding","math"]}
```

可以看到响应首部中类型`Content-Type`被设置成了`application/json`。类似的格式还有 xml（`application/xml`）/pdf（`application/pdf`）/png（`image/png`）等等。

## cookie

### 概念

什么是 cookie？

cookie 的出现是为了解决 HTTP 协议的无状态性的。客户端通过 HTTP 协议与服务器通信，多次请求之间无法记录状态。服务器可以在响应中设置 cookie，客户端保存这些 cookie。然后每次请求时都带上这些 cookie，服务器就可以通过这些 cookie 记录状态，辨别用户身份等。


### 重要性

> 整个计算机行业的收入都建立在 cookie 机制之上，广告领域更是如此。

上面的说法虽然有些夸张，但是可见 cookie 的重要性。

我们知道广告是互联网最常见的盈利方式。其中有一个很厉害的广告模式，叫做**联盟广告**。你有没有这样一种经历，刚刚在百度上搜索了某个关键字，然后打开淘宝或京东后发现相关的商品已经被推荐到首页或边栏了。这是由于这些网站组成了广告联盟，只要加入它们，就可以共享用户浏览器的 cookie 数据。

### 使用

Go 中 cookie 使用`http.Cookie`结构表示，在`net/http`包中定义：

```golang
// src/net/http/cookie.go
type Cookie struct {
	Name        string
	Value       string
	Path        string
	Domain      string
	Expires     time.Time
	RawExpires  string
	MaxAge      int
	Secure      bool
	HttpOnly    bool
	SameSite    SameSite
	Raw         string
	Unparsed    []string
}
```

* `Name/Value`：cookie 的键值对，都是字符串类型；
* 没有设置`Expires`字段的 cookie 被称为**会话 cookie** 或**临时 cookie**，这种 cookie 在浏览器关闭时就会自动删除。设置了`Expires`字段的 cookie 称为**持久 cookie**，这种 cookie 会一直存在，直到指定的时间来临或手动删除；
* `HttpOnly`字段设置为`true`时，该 cookie 只能通过 HTTP 访问，不能使用其它方式操作，如 JavaScript。提高安全性；

注意：

`Expires`和`MaxAge`都可以用于设置 cookie 的过期时间。`Expires`字段设置的是 cookie 在什么**时间点**过期，而`MaxAge`字段表示 cookie 自创建之后能够存活多少秒。虽然 HTTP 1.1 中废弃了`Expires`，推荐使用`MaxAge`代替。但是几乎所有的浏览器都仍然支持`Expires`；而且，微软的 IE6/IE7/IE8 都不支持 `MaxAge`。所以为了更好的可移植性，可以只使用`Expires`或同时使用这两个字段。

cookie 需要通过响应的首部发送给客户端。浏览器收到`Set-Cookie`首部时，会将其中的值解析成 cookie 格式保存在浏览器中。下面我们来具体看看如何设置 cookie：

```golang
func setCookie(w http.ResponseWriter, r *http.Request) {
    c1 := &http.Cookie {
        Name:       "name",
        Value:      "darjun",
        HttpOnly:   true,
    }
    c2 := &http.Cookie {
        Name:       "age",
        Value:      18,
        HttpOnly:   true,
    }
    w.Header().Set("Set-Cookie", c1.String())
    w.Header().Add("Set-Cookie", c2.String())
}

mux.HandleFunc("/set_cookie", setCookie)
```

运行程序，打开浏览器输入`localhost:8080/set_cookie`，浏览器中什么都没有显示，我们需要通过开发者工具查看 cookie。在 chrome 浏览器（其它浏览器类似）按下 F12，切换到 Application（应用）标签，在左侧 Cookies 下点击测试的 URL，右侧即可显示我们刚刚设置的 cookie：

![](/img/in-post/goweb/cookie1.png)

当然，我们也可以使用`curl`测试。但是`curl`返回的结果就只是响应中的`Set-Cookie`首部：

```
curl -i localhost:8080/set_cookie
```

```
HTTP/1.1 200 OK
Set-Cookie: name=darjun; HttpOnly
Set-Cookie: age=18; HttpOnly
Date: Fri, 20 Dec 2019 14:08:01 GMT
Content-Length: 0
```

上面构造 cookie 的代码中，有几点需要注意：

* 首部名称为`Set-Cookie`；
* 首部的值需要是字符串，所以调用了`Cookie`类型的`String`方法将其转为字符串再设置；
* 设置第一个 cookie 调用`Header`类型的`Set`方法，添加第二个 cookie 时调用`Add`方法。`Set`会将同名的键覆盖掉。如果第二个也调用`Set`方法，那么第一个 cookie 将会被覆盖。

为了使用的便捷，`net/http`包还提供了`SetCookie`方法。用法如下：

```golang
func setCookie2(w http.ResponseWriter, r *http.Request) {
    c1 := &http.Cookie {
        Name:       "name",
        Value:      "darjun",
        HttpOnly:   true,
    }
    c2 := &http.Cookie {
        Name:       "age",
        Value:      "18",
        HttpOnly:   true,
    }
    http.SetCookie(w, c1)
    http.SetCookie(w, c2)
}

mux.HandleFunc("/set_cookie2", setCookie2)
```

如果收到的响应中有 cookie 信息，浏览器会将这些 cookie 保存下来。只有没有过期，在向同一个主机发送请求时都会带上这些 cookie。在服务端，我们可以从请求的`Header`字段读取`Cookie`属性来获得 cookie：

```golang
func getCookie(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Host:", r.Host)
	fmt.Fprintln(w, "Cookies:", r.Header["Cookie"])
}

mux.HandleFunc("/get_cookie", getCookie)
```

第一次启动服务器，请求`localhost:8080/get_cookie`时，结果如下，没有 cookie 信息：

![](/img/in-post/goweb/cookie2.png#center)

先请求一次`localhost:8080/set_cookie`，然后再次请求`localhost:8080/get_cookie`，结果如下，浏览器将 cookie 传过来了：

![](/img/in-post/goweb/cookie3.png#center)

`r.Header["Cookie"]`返回一个切片，这个切片又包含了一个字符串，而这个字符串又包含了客户端发送的任意多个 cookie。如果想要取得单个键值对格式的 cookie，就需要解析这个字符串。
为此，`net/http`包在`http.Request`上提供了一些方法使我们更容易地获取 cookie：

```golang
func getCookie2(w http.ResponseWriter, r *http.Request) {
    name, err := r.Cookie("name")
    if err != nil {
        fmt.Fprintln(w, "cannot get cookie of name")
    }
    
    cookies := r.Cookies()
    fmt.Fprintln(w, c1)
    fmt.Fprintln(w, cookies)
}

mux.HandleFunc("/get_cookies", getCookies2)
```

* `Cookie`方法返回以传入参数为键的 cookie，如果该 cookie 不存在，则返回一个错误；
* `Cookies`方法返回客户端传过来的所有 cookie。

测试新的 URL `get_cookie2`：

![](/img/in-post/goweb/cookie5.png#center)

有一点需要注意，cookie 是与主机名绑定的，不考虑端口。我们上面查看 cookie 的图中有一列`Domain`表示的就是主机名。可以这样来验证一下，创建两个服务器，一个绑定在 8080 端口，一个绑定在 8081 端口，先请求`localhost:8080/set_cookie`设置 cookie，然后请求`localhost:8081/get_cookie`：

```golang
func main() {
	mux1 := http.NewServeMux()
	mux1.HandleFunc("/set_cookie", setCookie)
	mux1.HandleFunc("/get_cookie", getCookie)

	server1 := &http.Server{
		Addr:    ":8080",
		Handler: mux1,
	}

	mux2 := http.NewServeMux()
	mux2.HandleFunc("/get_cookie", getCookie)

	server2 := &http.Server {
		Addr: 	 ":8081",
		Handler: mux2,
	}
	
	wg := sync.WaitGroup{}
	wg.Add(2)

	go func () {
		defer wg.Done()

		if err := server1.ListenAndServe(); err != nil {
			log.Fatal(err)
		}
	}()

	
	go func() {
		defer wg.Done()

		if err := server2.ListenAndServe(); err != nil {
			log.Fatal(err)
		}
	}()

	wg.Wait()
}
```

发送给端口 8081 的请求同样可以获取 cookie：

![](/img/in-post/goweb/cookie4.png#center)

建议自己尝试一下，(*^_^*)

上面代码中，不能直接在主 goroutine 中依次`ListenAndServe`两个服务器。因为`ListenAndServe`只有在出错或关闭时才会返回。在此之前，第二个服务器永远得不到机会运行。所以，我创建两个 goroutine 各自运行一个服务器，并且使用`sync.WaitGroup`来同步。否则，主 goroutine 运行结束之后，整个程序就退出了。

## 总结

本文介绍了如何响应客户端的请求和 cookie 的相关知识。相关代码在[Github](https://github.com/darjun/go-web-example/tree/master/4-response)上，非常建议大家自己编写运行一遍以便加深印象。