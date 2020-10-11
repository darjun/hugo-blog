---
layout:    post
title:    "Go 每日一库之 quicktemplate"
subtitle: 	"每天学习一个 Go 库"
date:		2020-09-26T12:34:04
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/09/26/godailylib/quicktemplate"
categories: [
	"Go"
]
---

## 简介

最近在整理我们项目代码的时候，发现有很多活动的代码在结构和提供的功能上都非常相似。为了方便今后的开发，我花了一点时间编写了一个生成代码框架的工具，最大程度地降低重复劳动。代码本身并不复杂，且与项目代码关联性较大，这里就不展开介绍了。在这个过程中，我发现 Go 标准的模板库`text/template`和`html/template`使用起来比较束手束脚，很不方便。我从 GitHub 了解到[`quicktemplate`](https://github.com/valyala/quicktemplate)这个第三方模板库，功能强大，语法简单，使用方便。今天我们就来介绍一下`quicktemplate`。

## 快速使用

本文代码使用 Go Modules。

先创建代码目录并初始化：

```cmd
$ mkdir quicktemplate && cd quicktemplate
$ go mod init github.com/darjun/go-daily-lib/quicktemplate
```

`quicktemplate`会将我们编写的模板代码转换为 Go 语言代码。因此我们需要安装`quicktemplate`包和一个名为`qtc`的编译器：

```cmd
$ go get -u github.com/valyala/quicktemplate
$ go get -u github.com/valyala/quicktemplate/qtc
```

首先，我们需要编写`quicktemplate`格式的模板文件，模板文件默认以`.qtpl`作为扩展名。下面我编写了一个简单的模板文件`greeting.qtpl`：

```template
All text outside function is treated as comments.

{% func Greeting(name string, count int) %}
  {% for i := 0; i < count; i++ %}
    Hello, {%s name %}
  {% endfor %}
{% endfunc %}
```

模板语法非常简单，我们只需要简单了解以下 2 点：

* 模板以函数为单位，函数可以接受任意类型和数量的参数，这些参数可以在函数中使用。所有函数外的文本都是注释，`qtc`编译时会忽视注释；
* 函数内的内容，除了语法结构，其他都会原样输出到渲染后的文本中，**包括空格和换行**。

将`greeting.qtpl`保存到`templates`目录，然后执行`qtc`命令。该命令会生成对应的 Go 文件`greeting.qtpl.go`，包名为`templates`。现在，我们就可以使用这个模板了：

```golang
package main

import (
  "fmt"

  "github.com/darjun/go-daily-lib/quicktemplate/get-started/templates"
)

func main() {
  fmt.Println(templates.Greeting("dj", 5))
}
```

调用模板函数，传入参数，返回渲染后的文本：

```cmd
$ go run .


    Hello, dj

    Hello, dj

    Hello, dj

    Hello, dj

    Hello, dj

```

`{%s name %}`执行文本替换，`{% for %}`循环生成重复文本。输出中出现多个空格和换行，这是因为**函数内除了语法结构，其他内容都会原样保留，包括空格和换行**。

需要注意的是，由于`quicktemplate`是将模板转换为 Go 代码使用的，所以**如果模板有修改，必须先执行`qtc`命令重新生成 Go 代码，否则修改不生效**。

## 语法结构

`quicktemplate`支持 Go 常见的语法结构，`if/for/func/import/return`。而且写法与直接写 Go 代码没太大的区别，几乎没有学习成本。只是在模板中使用这些语法时，需要使用`{%`和`%}`包裹起来，而且`if`和`for`等需要添加`endif/endfor`明确表示结束。

### 变量

上面我们已经看到如何渲染传入的参数`name`，使用`{%s name %}`。由于`name`是 string 类型，所以在`{%`后使用`s`指定类型。`quicktemplate`还支持其他类型的值：

* 整型：`{%d int %}`，`{%dl int64 %}`，`{%dul uint64 %}`；
* 浮点数：`{%f float %}`。还可以设置输出的精度，使用`{%f.precision float %}`。例如`{%f.2 1.2345 %}`输出`1.23`；
* 字节切片（`[]byte`）：`{%z bytes %}`；
* 字符串：`{%q str %}`或字节切片：`{%qz bytes %}`，引号转义为`&quot;`；
* 字符串：`{%j str %}`或字节切片：`{%jz bytes %}`，没有引号；
* URL 编码：`{%u str %}`，`{%uz bytes %}`；
* `{%v anything %}`：输出等同于`fmt.Sprintf("%v", anything)`。

先编写模板：

```template
{% func Types(a int, b float64, c []byte, d string) %}
  int: {%d a %}, float64: {%f.2 b %}, bytes: {%z c %}, string with quotes: {%q d %}, string without quotes: {%j d %}.
{% endfunc %}
```

然后使用：

```golang
func main() {
  fmt.Println(templates.Types(1, 5.75, []byte{'a', 'b', 'c'}, "hello"))
}
```

运行：

```cmd
$ go run .

  int: 1, float64: 5.75, bytes: abc, string with quotes: &quot;hello&quot;, string without quotes: hello.

```

### 调用函数

`quicktemplate`支持在模板中调用模板函数、标准库的函数。由于`qtc`会直接生成 Go 代码，**我们甚至还可以在同目录下编写自己的函数给模板调用**，模板 A 中也可以调用模板 B 中定义的函数。

我们先在`templates`目录下编写一个文件`rank.go`，定义一个`Rank`函数，传入分数，返回评级：

```golang
package templates

func Rank(score int) string {
  if score >= 90 {
    return "A"
  } else if score >= 80 {
    return "B"
  } else if score >= 70 {
    return "C"
  } else if score >= 60 {
    return "D"
  } else {
    return "E"
  }
}
```

然后我们可以在模板中调用这个函数：

```template
{% import "fmt" %}
{% func ScoreList(name2score map[string]int) %}
  {% for name, score := range name2score %}
    {%s fmt.Sprintf("%s: score-%d rank-%s", name, score, Rank(score)) %}
  {% endfor %}
{% endfunc %}
```

编译模板：

```cmd
$ qtc
```

编写程序：

```golang
func main() {
  name2score := make(map[string]int)
  name2score["dj"] = 85
  name2score["lizi"] = 96
  name2score["hjw"] = 52

  fmt.Println(templates.ScoreList(name2score))
}
```

运行程序输出：

```cmd
$ go run .


    dj: score-85 rank-B

    lizi: score-96 rank-A

    hjw: score-52 rank-E


```

由于我们在模板中用到`fmt`包，需要先使用`{% import %}`将该包导入。

在模板中调用另一个模板的函数也是类似的，因为模板最终都会转为 Go 代码。Go 代码中有同样签名的函数。

## Web

`quicktemplate`常用来编写 HTML 页面的模板：

```template
{% func Index(name string) %}
<html>
  <head>
    <title>Awesome Web</title>
  </head>
  <body>
    <h1>Hi, {%s name %}
    <p>Welcome to the awesome web!!!</p>
  </body>
</html>
{% endfunc %}
```

下面编写一个简单的 Web 服务器：

```golang
func index(w http.ResponseWriter, r *http.Request) {
  templates.WriteIndex(w, r.FormValue("name"))
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

`qtc`会生成一个`Write*`的方法，它接受一个`io.Writer`的参数。将模板渲染的结果写入这个`io.Writer`中，我们可以直接将`http.ResponseWriter`作为参数传入，非常便捷。

运行：

```cmd
$ qtc
$ go run .
```

浏览器输入`localhost:8080?name=dj`查看结果。

## 总结

`quicktemplate`至少有下面 3 个优势：

* 语法与 Go 语言非常类似，几乎没有学习成本；
* 会先转换为 Go，渲染速度非常快，比标准库`html/template`快 20 倍以上；
* 为了安全考虑，会执行一些编码，避免受到攻击。

从我个人的实际使用情况来看，确实很方便，很实用。感兴趣的还可以去看看`qtc`生成的 Go 代码。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. quicktemplate GitHub：[https://github.com/valyala/quicktemplate](https://github.com/valyala/quicktemplate)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)