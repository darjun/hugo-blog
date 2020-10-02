---
layout:    post
title:    "Go 每日一库之 air"
subtitle: 	"每天学习一个 Go 库"
date:		2020-09-27T22:24:52
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/09/27/godailylib/air"
categories: [
	"Go"
]
---

## 简介

[`air`](github.com/cosmtrek/air)是 Go 语言的热加载工具，它可以监听文件或目录的变化，自动编译，重启程序。大大提高开发期的工作效率。

## 快速使用

本文代码使用 Go Modules，在 Mac 上运行。

先创建目录并初始化：

```cmd
$ mkdir air && cd air
$ go mod init github.com/darjun/go-daily-lib/air
```

执行下面的命令安装`air`工具：

```cmd
$ go get -u github.com/cosmtrek/air
```

上面的命令会在`$GOPATH/bin`目录下生成`air`命令。我一般会将`$GOPATH/bin`加入系统`PATH`中，所以可以方便地在任何地方执行`air`命令。

下面我们使用标准库`net/http`编写一个简单的 Web 服务器：

```golang
package main

import (
  "fmt"
  "log"
  "net/http"
)

func index(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, "Hello, world!")
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", index)

  server := &http.Server{
    Handler: mux,
    Addr:    ":8080",
  }

  log.Fatal(server.ListenAndServe())
}
```

运行程序：

```cmd
$ go run main.go
```

在浏览器中访问`localhost:8080`即可看到`Hello, world!`。

如果现在要求把`Hello, world!`改为`Hello, dj!`，如果不用`air`只能修改代码，执行`go build`编译，然后重启。

使用`air`，我们只需要执行下面的命令就可以运行程序：

```cmd
$ air
```

`air`会自动编译，启动程序，并监听当前目录中的文件修改：

![](/img/in-post/godailylib/air1.png#center)

当我们将`Hello, world!`改为`Hello, dj!`时，`air`监听到了这个修改，会自动编译，并且重启程序：

![](/img/in-post/godailylib/air2.png#center)

这时，我们在浏览器中访问`localhost:8080`，文本会变为`Hello, dj!`，是不是很方便？

## 配置

直接执行`air`命令，使用的就是默认的配置。一般建议将`air`项目中提供的[`air_example.toml`](https://github.com/cosmtrek/air/blob/master/air_example.toml)配置文件复制一份，根据自己的需求做修改和定制：

```toml
root = "."
tmp_dir = "tmp"

[build]
cmd = "go build -o ./tmp/main ."
bin = "tmp/main"
full_bin = "APP_ENV=dev APP_USER=air ./tmp/main"
include_ext = ["go", "tpl", "tmpl", "html"]
exclude_dir = ["assets", "tmp", "vendor", "frontend/node_modules"]
include_dir = []
exclude_file = []
log = "air.log"
delay = 1000 # ms
stop_on_error = true
send_interrupt = false
kill_delay = 500 # ms

[log]
time = false

[color]
main = "magenta"
watcher = "cyan"
build = "yellow"
runner = "green"

[misc]
clean_on_exit = true
```

可以配置项目根目录，临时文件目录，编译和执行的命令，监听文件目录，监听后缀名，甚至控制台日志颜色都可以配置。

## 调试模式

如果想查看`air`更详细的执行流程，可以使用`-d`选项。

![](/img/in-post/godailylib/air3.png#center)

使用`-d`选项，`air`会输出非常详细的信息，可以帮助排查问题。

## 总结

在开发期，使用`air`可以避免频繁地编译，重启。把这些都自动化了，大大地提升了开发效率。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. air GitHub：[https://github.com/cosmtrek/air](https://github.com/cosmtrek/air)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)