---
layout:		post
title:		"Go Web 编程之 静态文件"
subtitle: 	"入门 Go Web 编程"
date:		2020-01-13T06:15:36
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go Web 编程
URL: "2020/01/13/goweb/fileserver"
categories: [
	"Go"
]
---

## 概述

在 Web 开发中，需要处理很多静态资源文件，如 css/js 和图片文件等。本文将介绍在 Go 语言中如何处理文件请求。
接下来，我们将介绍两种处理文件请求的方式：原始方式和`http.FileServer`方法。

## 原始方式

原始方式比较简单粗暴，直接读取文件，然后返回给客户端。

```golang
func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/static/", fileHandler)

  server := &http.Server {
    Addr:    ":8080",
    Handler: mux,
  }
  if err := server.ListenAndServe(); err != nil {
    log.Fatal(err)
  }
}
```

上面我们创建了一个文件处理器，将它挂载到路径`/static/`上。一般地，静态文件的路径有一个共同的前缀，以便与其它路径区分。如这里的`/static/`，还有一些常用的，例如`/public/`等。
代码的其它部分与[程序模板](https://darjun.github.io/2019/12/05/goweb/structure/#%E6%80%BB%E7%BB%93)没什么不同，这里就不赘述了。

另外需要注意的是，这里的注册路径`/static/`最后的`/`不能省略。我们在前面的文章[程序结构](https://darjun.github.io/2019/12/05/goweb/structure/)中介绍过，如果请求的路径没有精确匹配的处理，会逐步去掉路径最后部分再次查找。
静态文件的请求路径一般为`/static/hello.html`这种形式。没有精确匹配的路径，继而查找`/static/`，这个路径与`/static`是不能匹配的。

接下来，我们看看文件处理器的实现：

```golang
func fileHandler(w http.ResponseWriter, r *http.Request) {
  path := "." + r.URL.Path
  fmt.Println(path)

  f, err := os.Open(path)
  if err != nil {
    Error(w, toHTTPError(err))
    return
  }
  defer f.Close()

  d, err := f.Stat()
  if err != nil {
    Error(w, toHTTPError(err))
    return
  }

  if d.IsDir() {
    DirList(w, r, f)
    return
  }

  data, err := ioutil.ReadAll(f)
  if err != nil {
    Error(w, toHTTPError(err))
    return
  }

  ext := filepath.Ext(path)
  if contentType := extensionToContentType[ext]; contentType != "" {
    w.Header().Set("Content-Type", contentType)
  }

  w.Header().Set("Content-Length", strconv.FormatInt(d.Size(), 10))
  w.Write(data)
}
```

首先我们读出请求路径，再加上相对可执行文件的路径。一般地，`static`目录与可执行文件在同一个目录下。然后打开该路径，查看信息。
如果该路径表示的是一个文件，那么根据文件的后缀设置`Content-Type`，读取文件的内容并返回。代码中简单列举了几个后缀对应的`Content-Type`：

```golang
var extensionToContentType = map[string]string {
  ".html": "text/html; charset=utf-8",
  ".css": "text/css; charset=utf-8",
  ".js": "application/javascript",
  ".xml": "text/xml; charset=utf-8",
  ".jpg":  "image/jpeg",
}
```

如果该路径表示的是一个目录，那么返回目录下所有文件与目录的列表：

```golang
func DirList(w http.ResponseWriter, r *http.Request, f http.File) {
  dirs, err := f.Readdir(-1)
  if err != nil {
    Error(w, http.StatusInternalServerError)
    return
  }
  sort.Slice(dirs, func(i, j int) bool { return dirs[i].Name() < dirs[j].Name() })

  w.Header().Set("Content-Type", "text/html; charset=utf-8")
  fmt.Fprintf(w, "<pre>\n")
  for _, d := range dirs {
    name := d.Name()
    if d.IsDir() {
      name += "/"
    }
    url := url.URL{Path: name}
    fmt.Fprintf(w, "<a href=\"%s\">%s</a>\n", url.String(), name)
  }
  fmt.Fprintf(w, "</pre>\n")
}
```

上面的函数先读取目录下第一层的文件和目录，然后按照名字排序。最后拼装成包含超链接的 HTML 返回。用户可以点击超链接访问对应的文件或目录。

如何上述过程中出现错误，我们使用`toHTTPError`函数将错误转成对应的响应码，然后通过`Error`回复给客户端。

```golang
func toHTTPError(err error) int {
  if os.IsNotExist(err) {
    return http.StatusNotFound
  }
  if os.IsPermission(err) {
    return http.StatusForbidden
  }
  return http.StatusInternalServerError
}

func Error(w http.ResponseWriter, code int) {
  w.WriteHeader(code)
}
```

同级目录下`static`目录内容：

```
static
├── folder
│   ├── file1.txt
│   └── file2.txt
│   └── file3.txt
├── hello.css
├── hello.html
├── hello.js
└── hello.txt
```

运行程序看看效果：

```
$ go run main.go
```

打开浏览器，请求`localhost:8080/static/hello.html`：

![](/img/in-post/goweb/static1.png#center)

可以看到页面`hello.html`已经呈现了：

```html
<!-- hello.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Go Web 编程之 静态文件</title>
  <link rel="stylesheet" href="/static/hello.css">
</head>
<body>
  <p class="greeting">Hello World!</p>
  <script src="/static/hello.js"></script>
</body>
</html>
```

html 使用的 css 和 js 文件也是通过`/static/`路径请求的，两个文件都比较简单：

```css
.greeting {
  font-family: sans-serif;
  font-size: 15px;
  font-style: italic;
  font-weight: bold;
}
```

```js
console.log("Hello World!")
```

"Hello World!"字体显示为 css 设置的样式，通过观察控制台也能看到 js 打印的信息。

再来看看文件目录浏览，在浏览器中请求`localhost:8080/static/`：

![](/img/in-post/goweb/static2.png#center)

可以依次点击列表中的文件查看其内容。

点击`hello.css`：
![](/img/in-post/goweb/static3.png#center)

点击`hello.js`：
![](/img/in-post/goweb/static4.png#center)

依次点击`folder`和`file1.txt`：

![](/img/in-post/goweb/static5.png#center)

![](/img/in-post/goweb/static6.png#center)

静态文件的请求路径也会输出到运行服务器的控制台中：

```
$ go run main.go 
./static/
./static/hello.css
./static/hello.js
./static/folder/
./static/folder/file1.txt

```

原始方式的实现有一个缺点，实现逻辑复杂。上面的代码尽管我们已经忽略很多情况的处理了，代码量还是不小。自己编写很繁琐，而且容易产生 BUG。
静态文件服务的逻辑其实比较一致，应该通过库的形式来提供。为此，Go 语言提供了`http.FileServer`方法。

## `http.FileServer`

先来看看如何使用：

```golang
package main

import (
  "log"
  "net/http"
)

func main() {
  mux := http.NewServeMux()
  mux.Handle("/static/", http.FileServer(http.Dir("")))


  server := &http.Server {
    Addr: ":8080",
    Handler: mux,
  }

  if err := server.ListenAndServe(); err != nil {
    log.Fatal(err)
  }
}
```

上面的代码使用`http.Server`方法，几行代码就实现了与原始方式相同的效果，是不是很简单？这就是使用库的好处！

`http.FileServer`接受一个`http.FileSystem`接口类型的变量：

```golang
// src/net/http/fs.go
type FileSystem interface {
  Open(name string) (File, error)
}
```

传入`http.Dir`类型变量，注意`http.Dir`是一个类型，其底层类型为`string`，并不是方法。因而`http.Dir("")`只是一个类型转换，而非方法调用：

```golang
// src/net/http/fs.go
type Dir string
```

`http.Dir`表示文件的起始路径，空即为当前路径。调用`Open`方法时，传入的参数需要在前面拼接上该起始路径得到实际文件路径。

`http.FileServer`的返回值类型是`http.Handler`，所以需要使用`Handle`方法注册处理器。`http.FileServer`将收到的请求路径传给`http.Dir`的`Open`方法打开对应的文件或目录进行处理。
在上面的程序中，如果请求路径为`/static/hello.html`，那么拼接`http.Dir`的起始路径`.`，最终会读取路径为`./static/hello.html`的文件。

有时候，我们想要处理器的注册路径和`http.Dir`的起始路径不相同。有些工具在打包时会将静态文件输出到`public`目录中。
这时需要使用`http.StripPrefix`方法，该方法会将请求路径中特定的前缀去掉，然后再进行处理：

```golang
package main

import (
  "log"
  "net/http"
)

func main() {
  mux := http.NewServeMux()
  mux.Handle("/static/", http.StripPrefix("/static", http.FileServer(http.Dir("./public"))))


  server := &http.Server {
    Addr: ":8080",
    Handler: mux,
  }

  if err := server.ListenAndServe(); err != nil {
    log.Fatal(err)
  }
}
```

这时，请求`localhost:8080/static/hello.html`将会返回`./public/hello.html`文件。
路径`/static/index.html`经过处理器`http.StripPrefix`去掉了前缀`/static`得到`/index.html`，然后又加上了`http.Dir`的起始目录`./public`得到文件最终路径`./public/hello.html`。

除此之外，`http.FileServer`还会根据请求文件的后缀推断内容类型，更全面：

```golang
// src/mime/type.go
var builtinTypesLower = map[string]string{
  ".css":  "text/css; charset=utf-8",
  ".gif":  "image/gif",
  ".htm":  "text/html; charset=utf-8",
  ".html": "text/html; charset=utf-8",
  ".jpeg": "image/jpeg",
  ".jpg":  "image/jpeg",
  ".js":   "application/javascript",
  ".mjs":  "application/javascript",
  ".pdf":  "application/pdf",
  ".png":  "image/png",
  ".svg":  "image/svg+xml",
  ".wasm": "application/wasm",
  ".webp": "image/webp",
  ".xml":  "text/xml; charset=utf-8",
}
```

如果文件后缀无法推断，`http.FileServer`将读取文件的前 512 个字节，根据内容来推断内容类型。感兴趣可以看一下源码`src/net/http/sniff.go`。

## `http.ServeContent`

除了直接使用`http.FileServer`之外，`net/http`库还暴露了`ServeContent`方法。这个方法可以用在处理器需要返回一个文件内容的时候，非常易用。

例如下面的程序，根据 URL 中的`file`参数返回对应的文件内容：

```golang
package main

import (
  "fmt"
  "log"
  "net/http"
  "os"
  "time"
)

func ServeFileContent(w http.ResponseWriter, r *http.Request, name string, modTime time.Time) {
  f, err := os.Open(name)
  if err != nil {
    w.WriteHeader(500)
    fmt.Fprint(w, "open file error:", err)
    return
  }
  defer f.Close()

  fi, err := f.Stat()
  if err != nil {
    w.WriteHeader(500)
    fmt.Fprint(w, "call stat error:", err)
    return
  }

  if fi.IsDir() {
    w.WriteHeader(400)
    fmt.Fprint(w, "no such file:", name)
    return
  }

  http.ServeContent(w, r, name, fi.ModTime(), f)
}

func fileHandler(w http.ResponseWriter, r *http.Request) {
  query := r.URL.Query()
  filename := query.Get("file")

  if filename == "" {
    w.WriteHeader(400)
    fmt.Fprint(w, "filename is empty")
    return
  }

  ServeFileContent(w, r, filename, time.Time{})
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/show", fileHandler)

  server := &http.Server {
    Addr:    ":8080",
    Handler: mux,
  }

  if err := server.ListenAndServe(); err != nil {
    log.Fatal(err)
  }
}
```

`http.ServeContent`除了接受参数`http.ResponseWriter`和`http.Request`，还需要文件名`name`，修改时间`modTime`和`io.ReadSeeker`接口类型的参数。

`modTime`参数是为了设置响应的`Last-Modified`首部。如果请求中携带了`If-Modified-Since`首部，`ServeContent`方法会根据`modTime`判断是否需要发送内容。
如果需要发送内容，`ServeContent`方法从`io.ReadSeeker`接口重读取内容。`*os.File`实现了接口`io.ReadSeeker`。

## 使用场景

Web 开发中的静态资源都可以使用`http.FileServer`来处理。除此之外，`http.FileServer`还可以用于实现一个简单的文件服务器，浏览或下载文件：

```golang
package main

import (
  "flag"
  "log"
  "net/http"
)

var (
  ServeDir string
)

func init() {
  flag.StringVar(&ServeDir, "sd", "./", "the directory to serve")
}

func main() {
  flag.Parse()

  mux := http.NewServeMux()
  mux.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.Dir(ServeDir))))


  server := &http.Server {
    Addr:    ":8080",
    Handler: mux,
  }

  if err := server.ListenAndServe(); err != nil {
    log.Fatal(err)
  }
}
```

在上面的代码中，我们构建了一个简单的文件服务器。编译之后，将想浏览的目录作为参数传给命令行选项，就可以浏览和下载该目录下的文件了：

```
$ ./main.exe -sd D:/code/golang
```

可以将端口也作为命令行选项，这样做出一个通用的文件服务器，编译之后就可以在其它机器上使用了😀。

## 总结

本文介绍了如何处理静态文件，依次介绍了原始方式、`http.FileServer`和`http.ServeContent`。最后使用`http.FileServer`实现了一个简单的文件服务器，可供日常使用。

## 参考

1. [Go Web 编程](https://book.douban.com/subject/27204133/)
2. [net/http](https://golang.org/pkg/net/http/)文档

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)