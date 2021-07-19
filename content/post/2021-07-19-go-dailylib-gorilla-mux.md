---
layout:    post
title:    "Go 每日一库之 gorilla/mux"
subtitle: "每天学习一个 Go 库"
date:     2021-07-19T20:22:34
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/07/19/godailylib/gorilla/mux"
categories: [
  "Go"
]
---

## 简介

[`gorilla/mux`](https://github.com/gorilla/mux)是 gorilla Web 开发工具包中的路由管理库。gorilla Web 开发包是 Go 语言中辅助开发 Web 服务器的工具包。它包括 Web 服务器开发的各个方面，有表单数据处理包[`gorilla/schema`](https://github.com/gorilla/schema)，有 websocket 通信包[`gorilla/websocket`](https://github.com/gorilla/websocket)，有各种中间件的包[`gorilla/handlers`](https://github.com/gorilla/handlers)，有 session 管理包[`gorilla/sessions`](https://github.com/gorilla/sessions)，有安全的 cookie 包[`gorilla/securecookie`](https://github.com/gorilla/securecookie)。本文先介绍`gorilla/mux`（下文简称`mux`），后续文章会依次介绍上面列举的 gorilla 包。

`mux`有以下优势：

* 实现了标准的`http.Handler`接口，所以可以与`net/http`标准库结合使用，非常轻量；
* 可以根据请求的主机名、路径、路径前缀、协议、HTTP 首部、查询字符串和 HTTP 方法匹配处理器，还可以自定义匹配逻辑；
* 可以在主机名、路径和请求参数中使用变量，还可以为之指定一个正则表达式；
* 可以传入参数给指定的处理器让其构造出完整的 URL；
* 支持路由分组，方便管理和维护。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir -p gorilla/mux && cd gorilla/mux
$ go mod init github.com/darjun/go-daily-lib/gorilla/mux
```

安装`gorilla/mux`库：

```cmd
$ go get -u github.com/gorilla/gorilla/mux
```

我现在身边有几本 Go 语言的经典著作：

![](/img/in-post/godailylib/mux1.png#center)

下面我们编写一个管理图书信息的 Web 服务。图书由 ISBN 唯一标识，ISBN 意为国际标准图书编号（International Standard Book Number）。

首先定义图书的结构：

```golang
type Book struct {
  ISBN        string   `json:"isbn"`
  Name        string   `json:"name"`
  Authors     []string `json:"authors"`
  Press       string   `json:"press"`
  PublishedAt string   `json:"published_at"`
}

var (
  mapBooks map[string]*Book
  slcBooks []*Book
)
```

定义`init()`函数，从文件中加载数据：

```golang
func init() {
  mapBooks = make(map[string]*Book)
  slcBooks = make([]*Book, 0, 1)

  data, err := ioutil.ReadFile("../data/books.json")
  if err != nil {
    log.Fatalf("failed to read book.json:%v", err)
  }

  err = json.Unmarshal(data, &slcBooks)
  if err != nil {
    log.Fatalf("failed to unmarshal books:%v", err)
  }

  for _, book := range slcBooks {
    mapBooks[book.ISBN] = book
  }
}
```

然后是两个处理函数，分别用于返回整个列表和某一本具体的图书：

```golang
func BooksHandler(w http.ResponseWriter, r *http.Request) {
  enc := json.NewEncoder(w)
  enc.Encode(slcBooks)
}

func BookHandler(w http.ResponseWriter, r *http.Request) {
  book, ok := mapBooks[mux.Vars(r)["isbn"]]
  if !ok {
    http.NotFound(w, r)
    return
  }

  enc := json.NewEncoder(w)
  enc.Encode(book)
}
```

注册处理器：

```golang
func main() {
  r := mux.NewRouter()
  r.HandleFunc("/", BooksHandler)
  r.HandleFunc("/books/{isbn}", BookHandler)
  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

`mux`的使用与`net/http`非常类似。首先调用`mux.NewRouter()`创建一个类型为`*mux.Router`的路由对象，该路由对象注册处理器的方式与标准库的`*http.ServeMux`完全相同，即调用`HandleFunc()`方法注册类型为`func(http.ResponseWriter, *http.Request)`的处理函数，调用`Handle()`方法注册实现了`http.Handler`接口的处理器对象。上面注册了两个处理函数，一个是显示图书信息列表，一个显示具体某本书的信息。

注意到路径`/books/{isbn}`使用了变量，在`{}`中间指定变量名，它可以匹配路径中的特定部分。在处理函数中通过`mux.Vars(r)`获取请求`r`的路由变量，返回`map[string]string`，后续可以用变量名访问。如上面的`BookHandler`中对变量`isbn`的访问。

由于`*mux.Router`也实现了`http.Handler`接口，所以可以直接将它作为`http.Handle("/", r)`的处理器对象参数注册。这里注册的是根路径`/`，相当于把所有请求的处理都托管给了`*mux.Router`。

最后还是`http.ListenAndServe(":8080", nil)`开启一个 Web 服务器，等待接收请求。

运行，在浏览器中键入`localhost:8080`，显示书籍列表：

![](/img/in-post/godailylib/mux2.png#center)

键入`localhost:8080/books/978-7-111-55842-2`，显示图书《Go 程序设计语言》的详细信息：

![](/img/in-post/godailylib/mux3.png#center)

从上面的使用过程中可以看出，`mux`库非常轻量，能很好的与标准库`net/http`结合使用。

我们还可以使用正则表达式限定变量的模式。ISBN 有固定的模式，现在使用的模式大概是这样：`978-7-111-55842-2`（这就是《Go 程序设计语言》一书的 ISBN），即 3个数字-1个数字-3个数字-5个数字-1个数字，用正则表达式表示为`\d{3}-\d-\d{3}-\d{5}-\d`。在变量名后添加一个`:`分隔变量和正则表达式：

```golang
r.HandleFunc("/books/{isbn:\\d{3}-\\d-\\d{3}-\\d{5}-\\d}", BookHandler)
```

## 灵活的匹配方式

`mux`提供了丰富的匹配请求的方式。相比之下，`net/http`只能指定具体的路径，稍显笨拙。

我们可以指定路由的域名或子域名：

```golang
r.Host("github.io")
r.Host("{subdomain:[a-zA-Z0-9]+}.github.io")
```

上面的路由只接受域名`github.io`或其子域名的请求，例如我的博客地址`darjun.github.io`就是它的一个子域名。指定域名时可以使用正则表达式，上面第二行代码限制子域名的第一部分必须是若干个字母或数字。

指定路径前缀：

```golang
// 只处理路径前缀为`/books/`的请求
r.PathPrefix("/books/")
```

指定请求的方法：

```golang
// 只处理 GET/POST 请求
r.Methods("GET", "POST")
```

使用的协议（`HTTP/HTTPS`）：

```golang
// 只处理 https 的请求
r.Schemes("https")
```

首部：

```golang
// 只处理首部 X-Requested-With 的值为 XMLHTTPRequest 的请求
r.Headers("X-Requested-With", "XMLHTTPRequest")
```

查询参数（即 URL 中`?`后的部分）：

```golang
// 只处理查询参数包含key=value的请求
r.Queries("key", "value")
```

最后我们可以组合这些条件：

```golang
r.HandleFunc("/", HomeHandler)
 .Host("bookstore.com")
 .Methods("GET")
 .Schemes("http")
```

除此之外，`mux`还允许自定义匹配器。自定义的匹配器就是一个类型为`func(r *http.Request, rm *RouteMatch) bool`的函数，根据请求`r`中的信息判断是否能否匹配成功。`http.Request`结构中包含了非常多的信息：HTTP 方法、HTTP 版本号、URL、首部等。例如，如果我们要求只处理 HTTP/1.1 的请求可以这么写：

```golang
r.MatchrFunc(func(r *http.Request, rm *RouteMatch) bool {
  return r.ProtoMajor == 1 && r.ProtoMinor == 1
})
```

**需要注意的是，`mux`会根据路由注册的顺序依次匹配。所以，通常是将特殊的路由放在前面，一般的路由放在后面**。如果反过来了，特殊的路由就不会被匹配到了：

```golang
r.HandleFunc("/specific", specificHandler)
r.PathPrefix("/").Handler(catchAllHandler)
```

## 子路由

有时候对路由进行分组管理，能让程序模块更清晰，更易于维护。现在网站扩展业务，加入了电影相关信息。我们可以定义两个子路由分别管理：

```golang
r := mux.NewRouter()
bs := r.PathPrefix("/books").Subrouter()
bs.HandleFunc("/", BooksHandler)
bs.HandleFunc("/{isbn}", BookHandler)

ms := r.PathPrefix("/movies").Subrouter()
ms.HandleFunc("/", MoviesHandler)
ms.HandleFunc("/{imdb}", MovieHandler)
```

子路由一般通过路径前缀来限定，`r.PathPrefix()`会返回一个`*mux.Route`对象，调用它的`Subrouter()`方法创建一个子路由对象`*mux.Router`，然后通过该对象的`HandleFunc/Handle`方法注册处理函数。

电影没有类似图书的 ISBN 国际统一标准，只有一个民间“准标准”：IMDB。我们采用豆瓣电影中的信息：

![](/img/in-post/godailylib/mux4.png#center)

定义电影的结构：

```golang
type Movie struct {
  IMDB        string `json:"imdb"`
  Name        string `json:"name"`
  PublishedAt string `json:"published_at"`
  Duration    uint32 `json:"duration"`
  Lang        string `json:"lang"`
}
```

加载：

```golang
var (
  mapMovies map[string]*Movie
  slcMovies []*Movie
)

func init() {
  mapMovies = make(map[string]*Movie)
  slcMovies = make([]*Movie, 0, 1)

  data,  := ioutil.ReadFile("../../data/movies.json")
  json.Unmarshal(data, &slcMovies)
  for _, movie := range slcMovies {
    mapMovies[movie.IMDB] = movie
  }
}
```

使用子路由的方式，还可以将各个部分的路由分散到各自的模块去加载，在文件`book.go`中定义一个`InitBooksRouter()`方法负责注册图书相关的路由：

```golang
func InitBooksRouter(r *mux.Router) {
  bs := r.PathPrefix("/books").Subrouter()
  bs.HandleFunc("/", BooksHandler)
  bs.HandleFunc("/{isbn}", BookHandler)
}
```

在文件`movie.go`中定义一个`InitMoviesRouter()`方法负责注册电影相关的路由：

```golang
func InitMoviesRouter(r *mux.Router) {
  ms := r.PathPrefix("/movies").Subrouter()
  ms.HandleFunc("/", MoviesHandler)
  ms.HandleFunc("/{imdb}", MovieHandler)
}
```

在`main.go`的主函数中：

```golang
func main() {
  r := mux.NewRouter()
  InitBooksRouter(r)
  InitMoviesRouter(r)

  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

需要注意的是，子路由匹配是需要包含路径前缀的，也就是说`/books/`才能匹配`BooksHandler`。

## 构造路由 URL

我们可以为一个路由起一个名字，例如：

```golang
r.HandleFunc("/books/{isbn}", BookHandler).Name("book")
```

上面的路由中有参数，我们可以传入参数值来构造一个完整的路径：

```golang
fmt.Println(r.Get("book").URL("isbn", "978-7-111-55842-2"))
// /books/978-7-111-55842-2 <nil>
```

返回的是一个`*url.URL`对象，其路径部分为`/books/978-7-111-55842-2`。这同样适用于主机名和查询参数：

```golang
r := mux.Router()
r.Host("{name}.github.io").
 Path("/books/{isbn}").
 HandlerFunc(BookHandler).
 Name("book")

url, err := r.Get("book").URL("name", "darjun", "isbn", "978-7-111-55842-2")
```

路径中所有的参数都需要指定，并且值需要满足指定的正则表达式（如果有的话）。运行输出：

```golang
$ go run main.go
http://darjun.github.io/books/978-7-111-55842-2
```

可以调用`URLHost()`只生成主机名部分，`URLPath()`只生成路径部分。

## 中间件

`mux`定义了中间件类型`MiddlewareFunc`：

```golang
type MiddlewareFunc func(http.Handler) http.Handler
```

所有满足该类型的函数都可以作为`mux`的中间件使用，通过调用路由对象`*mux.Router`的`Use()`方法应用中间件。如果看过我上一篇文章[《Go 每日一库之 net/http（基础和中间件）》](https://darjun.github.io/2021/07/13/in-post/godailylib/nethttp/)应该对这种中间件不陌生了。编写中间件一般会将原处理器传入，中间件中会手动调用原处理函数，然后在前后增加通用处理逻辑：

```golang
func loggingMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    log.Println(r.RequestURI)
    next.ServeHTTP(w, r)
  })
}
```

我在上篇文章中写的 3 个中间件可以直接使用，这就是兼容`net/http`的好处：

```golang
func main() {
  logger = log.New(os.Stdout, "[goweb]", log.Lshortfile|log.LstdFlags)

  r := mux.NewRouter()
  // 直接使用上一篇文章中定义的中间件
  r.Use(PanicRecover, WithLogger, Metric)
  InitBooksRouter(r)
  InitMoviesRouter(r)

  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

如果不手动调用原处理函数，那么原处理函数就不会执行，这可以用来在校验不通过时直接返回错误。例如，网站需要登录才能访问，而 HTTP 是一个无状态的协议。所以发明了 Cookie 机制用于在客户端和服务器之间记录一些信息。

我们在登录成功之后生成一个键为`token`的 Cookie 表示已登录成功，我们可以编写一个中间件来出来这块逻辑，如果 Cookie 不存在或者非法，则重定向到登录界面：

```golang
func login(w http.ResponseWriter, r *http.Request) {
  ptTemplate.ExecuteTemplate(w, "login.tpl", nil)
}

func doLogin(w http.ResponseWriter, r *http.Request) {
  r.ParseForm()
  username := r.Form.Get("username")
  password := r.Form.Get("password")
  if username != "darjun" || password != "handsome" {
    http.Redirect(w, r, "/login", http.StatusFound)
    return
  }

  token := fmt.Sprintf("username=%s&password=%s", username, password)
  data := base64.StdEncoding.EncodeToString([]byte(token))
  http.SetCookie(w, &http.Cookie{
    Name:     "token",
    Value:    data,
    Path:     "/",
    HttpOnly: true,
    Expires:  time.Now().Add(24 * time.Hour),
  })
  http.Redirect(w, r, "/", http.StatusFound)
}
```

上面为了记录登录状态，我将登录的用户名和密码组合成`username=xxx&password=xxx`形式的字符串，对这个字符串进行`base64`编码，然后设置到 Cookie 中。Cookie 有效期为 24 小时。同时为了安全只允许 HTTP 访问此 Cookie（JS 脚本不可访问）。**当然这种方式安全性很低，这里只是为了演示**。登录成功之后重定向到`/`。

为了展示登录界面，我创建了几个`template`模板文件，使用`html/template`解析：

登录展示页面：

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

主页面
```golang
<ul>
  <li><a href="/books/">图书</a></li>
  <li><a href="/movies/">电影</a></li>
</ul>
```

同时也创建了图书和电影的页面：

```golang
// movies.tpl
<ol>
  {{ range . }}
  <li>
    <p>书名: <a href="/movies/{{ .IMDB }}">{{ .Name }}</a></p>
    <p>上映日期: {{ .PublishedAt }}</p>
    <p>时长: {{ .Duration }}分</p>
    <p>语言: {{ .Lang }}</p>
  </li>
  {{ end }}
</ol>
```

```golang
// movie.tpl
<p>IMDB: {{ .IMDB }}</p>
<p>电影名: {{ .Name }}</p>
<p>上映日期: {{ .PublishedAt }}</p>
<p>时长: {{ .Duration }}分</p>
<p>语言: {{ .Lang }}</p>
```

图书页面类似。接下来要解析模板：

```golang
var (
  ptTemplate *template.Template
)

func init() {
  var err error
  ptTemplate, err = template.New("").ParseGlob("./tpls/*.tpl")
  if err != nil {
    log.Fatalf("load templates failed:%v", err)
  }
}
```

访问对应的页面逻辑：

```golang
func MoviesHandler(w http.ResponseWriter, r *http.Request) {
  ptTemplate.ExecuteTemplate(w, "movies.tpl", slcMovies)
}

func MovieHandler(w http.ResponseWriter, r *http.Request) {
  movie, ok := mapMovies[mux.Vars(r)["imdb"]]
  if !ok {
    http.NotFound(w, r)
    return
  }

  ptTemplate.ExecuteTemplate(w, "movie.tpl", movie)
}
```

执行对应的模板，传入电影列表或某个具体的电影信息即可。现在页面没有限制访问，我们来编写一个中间件限制只有登录用户才能访问，未登录用户访问时跳转到登录界面：

```golang
func authenticateMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    cookie, err := r.Cookie("token")
    if err != nil {
      // no cookie
      http.Redirect(w, r, "/login", http.StatusFound)
      return
    }

    data, _ := base64.StdEncoding.DecodeString(cookie.Value)
    values, _ := url.ParseQuery(string(data))
    if values.Get("username") != "dj" && values.Get("password") != "handsome" {
      // failed
      http.Redirect(w, r, "/login", http.StatusFound)
      return
    }

    next.ServeHTTP(w, r)
  })
}
```

再次强调，这里只是为了演示，这种验证方式安全性很低。

然后，我们让`books`和`movies`子路由应用中间件`authenticateMiddleware`（需要登录验证），而`login`子路由不用：

```golang
func InitBooksRouter(r *mux.Router) {
  bs := r.PathPrefix("/books").Subrouter()
  // 这里
  bs.Use(authenticateMiddleware)
  bs.HandleFunc("/", BooksHandler)
  bs.HandleFunc("/{isbn}", BookHandler)
}

func InitMoviesRouter(r *mux.Router) {
  ms := r.PathPrefix("/movies").Subrouter()
  // 这里
  ms.Use(authenticateMiddleware)
  ms.HandleFunc("/", MoviesHandler)
  ms.HandleFunc("/{id}", MovieHandler)
}

func InitLoginRouter(r *mux.Router) {
  ls := r.PathPrefix("/login").Subrouter()
  ls.Methods("GET").HandlerFunc(login)
  ls.Methods("POST").HandlerFunc(doLogin)
}
```

运行程序（注意多文件程序运行方式）：

```golang
$ go run .
```

访问`localhost:8080/movies/`，会重定向到`localhost:8080/login`。输入用户名`darjun`，密码`handsome`，登录成功显示主页面。后面的请求都不需要验证了，请随意点击点击吧😀

## 总结

本文介绍了轻量级的，功能强大的路由库`gorilla/mux`。它支持丰富的请求匹配方法，子路由能极大地方便我们管理路由。由于兼容标准库`net/http`，所以可以无缝集成到使用`net/http`的程序中，利用为`net/http`编写的中间件资源。下一篇我们介绍`gorilla/handlers`——一些常用的中间件。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gorilla/mux GitHub：[github.com/gorilla/gorilla/mux](https://github.com/gorilla/mux)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~