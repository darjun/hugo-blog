---
layout:    post
title:    "Go 每日一库之 gorilla/sessions"
subtitle: "每天学习一个 Go 库"
date:     2021-07-25T12:30:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/07/25/godailylib/gorilla/sessions"
categories: [
  "Go"
]
---

## 简介

上一篇文章[《Go 每日一库之 securecookie》](https://darjun.github.io/2021/07/23/godailylib/gorilla/securecookie/)中，我们介绍了 cookie。同时提到 cookie 有两个缺点，一是数据不宜过大，二是安全问题。session 是服务器端的存储方案，可以存储大量的数据，而且不需要向客户端传输，从而解决了这两个问题。但是 session 需要一个能唯一标识用户的 ID，这个 ID 一般存放在 cookie 中发送到客户端保存，随每次请求一起发送到服务器。cookie 和 session 通常配套使用。

[`gorilla/sessions`](https://github.com/gorilla/sessions)是 gorilla web 开发工具包中管理 session 的库。它提供了基于 cookie 和本地文件系统的 session。同时预留扩展接口，可以使用其它的后端存储 session 数据。

本文先介绍`sessions`提供的两种 session 存储方式，然后通过第三方扩展介绍在多个 Web 服务器实例间如何保持登录状态。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir gorilla/sessions && cd gorilla/sessions
$ go mod init github.com/darjun/go-daily-lib/gorilla/sessions
```

安装`gorilla/sessions`库：

```cmd
$ go get -u github.com/valyala/gorilla/sessions
```

现在我们实现在服务器端通过 session 存储一些信息的功能：

```golang
package main

import (
  "fmt"
  "github.com/gorilla/mux"
  "github.com/gorilla/sessions"
  "log"
  "net/http"
  "os"
)

var (
  store = sessions.NewFilesystemStore("./", securecookie.GenerateRandomKey(32), securecookie.GenerateRandomKey(32))
)

func set(w http.ResponseWriter, r *http.Request) {
  session, _ := store.Get(r, "user")
  session.Values["name"] = "dj"
  session.Values["age"] = 18
  err := sessions.Save(r, w)
  if err != nil {
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
  }
  fmt.Fprintln(w, "Hello World")
}

func read(w http.ResponseWriter, r *http.Request) {
  session, _ := store.Get(r, "user")
  fmt.Fprintf(w, "name:%s age:%d\n", session.Values["name"], session.Values["age"])
}

func main() {
  r := mux.NewRouter()
  r.HandleFunc("/set", set)
  r.HandleFunc("/read", read)
  log.Fatal(http.ListenAndServe(":8080", r))
}
```

整个程序逻辑比较清晰，分别在`/set`和`/read`路径下挂上设置和读取的处理函数。重点是变量`store`。我们调用`session.NewFilesystemStore()`方法创建了一个`*sessions.FilesystemStore`类型的对象，它会将我们的 session 内容存储到文件系统（即本地磁盘上）。我们需要给`NewFilesytemStore()`方法传入至少 2 个参数，第一个参数指定 session 存储的本地磁盘路径。后续参数依次指定`hashKey`和`blockKey`（可省略），前者用于验证，后者用于加密，我们可以使用`securecookie`生成足够随机的 key，详情见前一篇介绍`securecookie`的文章。

`sessions`为所有的 session 存储抽象了一个接口`Store`：

```golang
type Store interface {
  Get(r *http.Request, name string) (*Session, error)
  New(r *http.Request, name string) (*Session, error)
  Save(r *http.Request, w http.ResponseWriter, s *Session) error
}
```

实现这个接口可以自定义我们存储 session 的位置和格式。

在`set`处理函数中，我们调用`store.Get(r, "user")`获取名为`user`的 session，如果 session 不存在，则创建一个新的。`sessions`库支持为同一个用户创建多个 session，`store.Get()`方法的第二个参数指定名字。获取到的`*Session`结构如下：

```golang
type Session struct {
  ID string
  Values  map[interface{}]interface{}
  Options *Options
  IsNew   bool
  store   Store
  name    string
}
```

数据直接存放在`Session.Values`字段中，这是一个类型为`map[interface{}]interface{}`的字段，几乎能保存任何类型的数据（之所以我这里要说几乎，因为还要考虑序列化到存储的限制，有些数据类型无法序列化为字节流保存，如`chan`）。

在`set`处理函数中，我们直接操作`Values`字段，最后我们调用`store.Save(r, w, session)`将 session数据保存到对应的存储中。

在`get`处理函数中，同样地我们先调用`store.Get(r, "user")`获取`*Session`对象，然后读取里面的`name`和`age`值。

运行：

```golang
$ go run main.go

```

首先访问`localhost:8080/set`，通过浏览器的开发者工具`Application`页签查看 cookie：

![](/img/in-post/godailylib/sessions1.png#center)

我们发现 session 的名字会作为 cookie 名发送到客户端，session ID 被保存为 cookie 的值。

然后我们访问`localhost:8080/read`，读取到 session 保存的数据：

![](/img/in-post/godailylib/sessions2.png#center)

另前面说过`FilesystemStore`数据是存储在本地硬盘上的，在运行程序的本地目录我们看到有以 session 开头的文件，文件名 session 后面的部分就是 session ID：

![](/img/in-post/godailylib/sessions3.png#center)

## cookie 存储

除了默认的将本地文件系统作为存储外，`sessions`还支持将 cookie 作为存储，也就是将 session 的数据直接通过 cookie 在客户端和服务器之间传输。cookie 存储的创建方式与文件系统存储的创建方式类似：

```golang
var store = sessions.NewCookieStore(securecookie.GenerateRandomKey(32), securecookie.GenerateRandomKey(32))
```

`sessions.NewCookieStore()`方法的第一个参数为 hashKey 用于验证，第二个参数为 blockKey 用于加密，与`sessions.NewFilesystemStore()`一样。

其他部分的代码完全不用修改，运行程序的结果与上面的一致。session 数据保存在 cookie 中，随每次请求由客户端传给服务器。这种方式其实就是之前文章中介绍的 cookie 用法。

## 记录登录状态

之前我们介绍`gorilla/mux`时介绍过使用 cookie 保存登录状态。当时将用户名和密码经过简单的 Base64 编码后就直接存放在 cookie 中了，基本处于“裸露”状态。只要有意，很容易就能窃取用户名和密码。现在我们将用户关键信息存储在 session 中，cookie 中只存储一个 session ID。

首先，我们设计 3 个页面，登录页面，主页面，授权才能访问的 secret 页面。登录页面只需要用户名&密码的输入框和登录按钮即可：

```golang
// login.tpl
<form action="/login" method="post">
  <label>Username:</label>
  <input name="username"><br>
  <label>Password:</label>
  <input name="password" type="password"><br>
  <button type="submit">登录</button>
</form>
```

登录请求根据方法不同需要执行不同的操作，GET 方法表示请求登录的页面，POST 方法表示执行登录操作。我们使用`handlers.MethodHandler`这个中间件来处理同一个路径的不同方法的请求：

```golang
r.Handle("/login", handlers.MethodHandler{
  "GET":  http.HandlerFunc(Login),
  "POST": http.HandlerFunc(DoLogin),
})
```

`Login`处理函数很简单，只是展示页面：

```golang
func Login(w http.ResponseWriter, r *http.Request) {
  ptTemplate.ExecuteTemplate(w, "login.tpl", nil)
}
```

这里我使用 Go 标准库`html/template`模版库来加载和管理各个页面的模板：

```golang
var (
  ptTemplate *template.Template
)

func init() {
  template.Must(template.New("").ParseGlob("./tpls/*.tpl"))
}
```

`DoLogin`处理函数，需要验证登录请求，然后创建`User`对象，保存在 session 中，接着重定向到主页面：

```golang
func DoLogin(w http.ResponseWriter, r *http.Request) {
  r.ParseForm()
  username := r.Form.Get("username")
  password := r.Form.Get("password")
  if username != "darjun" || password != "handsome" {
    http.Redirect(w, r, "/login", http.StatusFound)
    return
  }

  SaveSessionUser(w, r, &User{Username: username})
  http.Redirect(w, r, "/", http.StatusFound)
}
```

下面是主页面的处理，我们可以从 session 中取出保存的`User`对象，根据是否有`User`对象显示不同的页面：

```golang
// home.tpl
{% if . %}
<p>Hi, {% .Username %}</p><br>
<a href="/secret">Goto secret?</a>
{% else %}
<p>Hi, stranger</p><br>
<a href="/login">Goto login?</a>
{% end %}
```

`HomeHandler`代码如下：

```golang
func HomeHandler(w http.ResponseWriter, r *http.Request) {
  u := GetSessionUser(r)
  ptTemplate.ExecuteTemplate(w, "home.tpl", u)
}
```

最后是 secret 页面：

```golang
// secret.tpl
<p>
Lorem ipsum dolor sit amet consectetur adipisicing elit.
Inventore a cumque sunt pariatur nihil doloremque tempore,
consectetur ipsum sapiente id excepturi enim velit,
quis nisi esse doloribus aliquid. Incidunt, dolore.
</p>
<p>You have visited this page {% .Count %} times.</p>
```

显示访问了该页面多少次。

`SecretHandler`如下：

```golang
func SecretHandler(w http.ResponseWriter, r *http.Request) {
  u := GetSessionUser(r)
  if u == nil {
    http.Redirect(w, r, "/login", http.StatusFound)
    return
  }
  u.Count++
  SaveSessionUser(w, r, u)
  ptTemplate.ExecuteTemplate(w, "secret.tpl", u)
}
```

如果没有 session，则重定向到登录页面。反之显示该页面。这里每次成功访问 secret 页面，都会增加计数器，保存在 session 中。

上面代码中需要注意一点，由于 session 内容的序列化使用了标准库中的`encoding/gob`，所以不支持直接序列化结构体，我封装了两个函数，将`User`对象序列化为 JSON，然后保存到 session 中和从 session 中取出字符串反序列化为`User`对象：

```golang
func GetSessionUser(r *http.Request) *User {
  session, _ := store.Get(r, "user")
  s, ok := session.Values["user"]
  if !ok {
    return nil
  }
  u := &User{}
  json.Unmarshal([]byte(s.(string)), u)
  return u
}

func SaveSessionUser(w http.ResponseWriter, r *http.Request, u *User) {
  session, _ := store.Get(r, "user")
  data, _ := json.Marshal(u)
  session.Values["user"] = string(data)
  store.Save(r, w, session)
}
```

现在运行我们的程序，首先访问`localhost:8080`，由于没有登录，显示`欢迎陌生人，去登录`：

![](/img/in-post/godailylib/sessions4.png#center)

点击去登录，跳转到登录界面，输入用户名和密码：

![](/img/in-post/godailylib/sessions5.png#center)

点击登录，跳转到主页，这时由于记录了登录状态，会显示`欢迎 darjun`：

![](/img/in-post/godailylib/sessions6.png#center)

点击去隐秘链接：

![](/img/in-post/godailylib/sessions7.png#center)

不停刷新页面，发现访问次数一直累加。

如果未登录时，直接访问`localhost:8080/secret`，会直接重定向到登录界面。

上面程序有一个缺点，程序重启启动后，就需要重新登录。因为每次启动我们都重新随机 hashKey 和 blockKey，只需要固定这两个值即可实现重启也能保存登录状态。

登录验证类的功能非常适合放在中间件中处理，之前的文章已经介绍过如何编写中间件了，这里就不赘述了。

## 第三方后端存储

将 session 存储在本地文件系统，不利于水平扩展。一般稍微上点规模的网站，Web 服务器都会部署很多个实例，请求通过 Nginx 之类的反向代理转发到一个后端实例处理。不能保证后面的请求与之前的请求在同一个实例中处理，故 session 一般需要存储在一个公共的地方，例如 redis。

`sessions`提供了扩展接口，方便扩展使用其他的后端存储 session 内容。目前 GitHub 上已经有很多的第三方后端扩展了，详细 list 见`sessions`库的 GitHub 首页：

![](/img/in-post/godailylib/sessions8.png#center)

我们只介绍基于 redis 的后端存储，其他的扩展感兴趣可自行研究。首先安装扩展：

```golang
$ go get gopkg.in/boj/redistore.v1
```

创建一个 redistore 的实例：

```golang
store, _ = redistore.NewRediStore(10, "tcp", ":6379", "", []byte("redis-key"))
```

参数依次为：

* `size`：最大空闲连接数；
* `network`：连接类型，一般是 TCP；
* `addr`：网络地址+端口；
* `password`：redis 的密码，如果未启用，填空；
* `keyPairs`：依次是 hashKey 和 blockKey（可省略），不再赘述。

为了验证，我们开启多个服务器，所以将端口通过命令行参数传入，使用标准库`flag`：

```golang
port = flag.Int("port", 8080, "port to listen")

func init() {
  flag.Parse()
}

func main() {
  // ...
  log.Fatal(http.ListenAndServe(fmt.Sprintf(":%d", *port), nil))
}
```

为了运行服务器，我们需要先开启一个 redis-server。redis 的安装就不多说了，在 windows 下，建议使用 chocolatey 安装，chocolatey 类似于 Ubutnu 的 apt-get，Mac 的 brew，非常方便，强烈推荐。

![](/img/in-post/godailylib/sessions9.png#center)

为了演示反向代理的效果，即通过一个地址可以随机访问部署的多个 Web 服务器，我们开启 3 个 Web 服务器。终端1：

```golang
$ go build 
$ ./redis -port 8080

```

终端2:

```golang
$ ./redis -port 8081

```

终端3:

```golang
$ ./redis -port 8082

```

可以使用`nginx`做反向代理，安装 nginx，配置：

```golang
upstream mysvr {
  server localhost:8080;
  server localhost:8081;
  server localhost:8082;
}

server {
  listen       80;
  server_name  localhost;

  location / {
    proxy_pass http://mysvr;
  }
}
```

这里表示将`localhost`随机转发到`mysvr`这个组中的 3 个服务器上，启动 nginx：

```golang
$ nginx -c nginx.conf
```

万事俱备，现在使用浏览器访问`localhost`，通过控制台日志发现是 server3 处理了这个请求：

![](/img/in-post/godailylib/sessions10.png#center)

点击去登录，server1 处理了展示页面的请求：

![](/img/in-post/godailylib/sessions11.png#center)

点击登录，server3 处理了 POST 类型的登录请求：

![](/img/in-post/godailylib/sessions12.png#center)

登录成功之后，重定向到主界面的请求又是 server1 处理的：

![](/img/in-post/godailylib/sessions13.png#center)

点击私密链接，展示页面的请求是 server2 处理的：

![](/img/in-post/godailylib/sessions14.png#center)

虽然每次处理的 server 不同，但是登录状态一直保存着。因为我们使用了 redis 保存 session。

注意，我这里每次都是随机一个 server 去处理，你运行的结果不一定一样。

## 总结

session 为了解决存储用户大量数据和安全性的问题。`sessions`库为 Go Web 开发中处理 session 提供了简单，灵活的方法。它依赖较少，可以即插即用，非常方便。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gorilla/sessions GitHub：[github.com/gorilla/sessions](https://github.com/gorilla/sessions)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)