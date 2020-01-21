---
layout:		post
title:		"Go Web 编程之 模板（二）"
subtitle: 	"入门 Go Web 编程"
date:		2020-01-09T21:31:43
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go Web 编程
URL: "2020/01/09/goweb/template2"
categories: [
	"Go"
]
---

## 概述

在[上一篇文章](https://darjun.github.io/2019/12/31/goweb/template1/)中，我们介绍了 Go 模板库`text/template`。
`text/template`库用于生成文本输出。在 Web 开发中，涉及到很多安全方面的问题。有些数据是用户输入的，不能直接替换到模板中，否则可能导致注入攻击。
Go 提供了`html/template`库处理这些问题。`html/template`提供了与`text/template`一样的接口。
我们通常使用`html/template`生成 HTML 输出。

由于[上一篇文章](https://darjun.github.io/2019/12/31/goweb/template1/)已经详细介绍了 Go 模板的基本概念，
本文主要从使用的层面来介绍`html/template`库。中间会有安全方面的内容。那就开始吧！

## 初体验

`html/template`库的使用与`text/template`基本一样：

```golang
package main

import (
	"fmt"
	"html/template"
	"log"
	"net/http"
)

func indexHandler(w http.ResponseWriter, r *http.Request) {
	t, err := template.ParseFiles("hello.html")
	if err != nil {
		w.WriteHeader(500)
		fmt.Fprint(w, err)
		return
	}

	t.Execute(w, "Hello World")
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", indexHandler)

	server := &http.Server {
		Addr:    ":8080",
		Handler: mux,
	}
	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```

模板文件`hello.html`：

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<meta http-equiv="X-UA-Compatible" content="ie=edge">
	<title>Go Web 编程之 模板（二）</title>
</head>
<body>
	{{ . }}
</body>
</html>
```

模板中的`{{ . }}`会被替换为传入的数据"Hello World"，程序将模板执行后生成的文本通过`ResponseWriter`传回客户端。

编译，运行程序（我的环境 Win10 + Git Bash）：

```
$ go build -o main.exe main.go
$ ./main.exe
```

打开浏览器，输入`localhost:8080`，即可看到"Hello World"页面。

我们在介绍`text/template`库的时候，提到了**空白问题**。在生成 HTML 时一般不需要考虑这个问题，因为浏览器渲染的时候会自动去掉多余的空白。

为了编写示例代码的便利，在解析时不进行错误处理，`html/template`库提供了`Must`方法。
它接受两个参数，一个模板对象指针，一个错误。如果错误参数不为`nil`，直接 panic，否则返回模板对象指针。
使用`Must`方法简化上面的处理器：

```golang
func indexHandler(w http.ResponseWriter, r *http.Request) {
    t := template.Must(template.ParseFiles("hello.html"))
    t.Execute(w, "Hello World")
}
```

## 动作

接下来，我们通过示例再过一遍几种动作。

### 条件动作

```golang
func conditionHandler(w http.ResponseWriter, r *http.Request) {
	age, err := strconv.ParseInt(r.URL.Query().Get("age"), 10, 64)
	if err != nil {
		fmt.Fprint(w, err)
		return
	}

	t := template.Must(template.ParseFiles("condition.html"))
	t.Execute(w, age)
}

mux.HandleFunc("/condition", conditionHandler)
```

模板文件 condition.html 只有 body 部分不同：

```html
<p>Your age is: {{ . }}</p>
{{ if gt . 60 }}
<p>Old People!</p>
{{ else if gt . 40 }}
<p>Middle Aged!</p>
{{ else }}
<p>Young!</p>
{{ end }}
```

模板逻辑很简单，使用内置函数`gt`判断传入的年龄处于哪个区间，显示对应的文本。

编译、运行程序，打开浏览器，输入`localhost:8080/condition?age=10`。

### 迭代动作

迭代动作一般用于生成一个列表。

```golang
type Item struct {
	Name	string
	Price	int
}

func iterateHandler(w http.ResponseWriter, r *http.Request) {
	t := template.Must(template.ParseFiles("iterate.html"))

	items := []Item {
		{ "iPhone", 5499 },
		{ "iPad", 6331 },
		{ "iWatch", 1499 },
		{ "MacBook", 8250 },
	}
	t.Execute(w, items)
}

mux.HandleFunc("/iterate", iterateHandler)
```

模板文件`iterate.html`：

```html
<h1>Apple Products</h1>
<ul>
{{ range . }}
<li>{{ .Name }}: ￥{{ .Price }}</li>
{{ end }}
</ul>
```

再次提醒，在`{{ range }}`中，`.`会被替换为当前遍历的元素值。

### 设置动作

设置动作允许用户在指定范围内为`.`设置值。

```golang
type User struct {
	Name	string
	Age		int
}

type Pet struct {
	Name	string
	Age		int
	Owner	User
}

func setHandler(w http.ResponseWriter, r *http.Request) {
	t := template.Must(template.ParseFiles("set.html"))

	pet := Pet {
		Name:	"Orange",
		Age:	2,
		Owner:	User {
			Name:	"dj",
			Age:	28,
		},
	}
	t.Execute(w, pet)
}

mux.HandleFunc("/set", setHandler)
```

模板文件`set.html`：

```html
<h1>Pet Info</h1>
<p>Name: {{ .Name }}</p>
<p>Age: {{ .Age }}</p>
<p>Owner:</p>
{{ with .Owner }}
<p>Name: {{ .Name }}</p>
<p>Age: {{ .Age }}</p>
{{ end }}
```

在`{{ with .Owner }}`和`{{ end }}`之间，可以直接通过`{{ .Name }}`和`{{ .Age }}`访问宠物主人的信息。

### 包含动作

包含动作允许用户在一个模板里面包含另一个模板，从而构造出嵌套的模板。

```golang
func includeHandler(w http.ResponseWriter, r *http.Request) {
	t := template.Must(template.ParseFiles("include1.html", "include2.html"))
	t.Execute(w, "Hello World!")
}
```

模板`include1.html`：

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<meta http-equiv="X-UA-Compatible" content="ie=edge">
	<title>Go Web 编程之 模板（二）</title>
</head>
<body>
	<div>This is in template include1.html</div>
	<p>The value of dot is {{ . }}</p>
	<hr/>
	<p>Don't pass argument to include2.html:</p>
	{{ template "include2.html" }}
	<hr/>
	<p>Pass dot to include2.html</p>
	{{ template "include2.html" . }}
	<hr/>
</body>
</html>
```

模板`include2.html`：

```html
<p>Get dot of value [{{ . }}]</p>
```

建议自己动手运行一下程序，观察输出。

`{{ template "include2.html" }}`未传入参数给模板`include2.html`，`{{ template "include2.html" . }}`将模板`include1.html`的参数传给了`include2.html`。

## 其它元素

### 管道

管道我们可以理解为数据的流向，在数据流向输出的每个阶段进行特定的处理。

```golang
func pipelineHandler(w http.ResponseWriter, r *http.Request) {
	t := template.Must(template.ParseFiles("pipeline.html"))
	t.Execute(w, rand.Float64())
}

mux.HandleFunc("/pipeline", pipelineHandler)
```

模板文件`pipeline.html`：

```html
<p>{{ . | printf "%.2f" }}</p>
```

该程序实现的功能非常简单，将传入的浮点数格式化为只保留小数点后两位。`|`是管道符号，前面的输出将作为后面的输入（如果是函数或方法调用，前面的输出将作为最后一个参数）。
实际上，`{{ . | printf "%.2f" }}`的输出`fmt.Sprintf("%.2f", .表示的数据)`的返回字符串相同。

### 函数

Go 模板库内置了一些基础的函数，如果要实现更为复杂的功能，可以自定义函数。

```golang
func formateDate(t time.Time) string {
	return t.Format("2006-01-02")
}

func funcsHandler(w http.ResponseWriter, r *http.Request) {
	funcMap := template.FuncMap{ "fdate": formateDate }
	t := template.Must(template.New("funcs.html").Funcs(funcMap).ParseFiles("funcs.html"))
	t.Execute(w, time.Now())
}

mux.HandleFunc("/funcs", funcsHandler)
```

模板文件`funcs.html`：

```html
<div>Today is {{ . | fdate }}</div>
```

自定义函数可以接受任意多个参数，但是只能返回一个值，或者返回一个值和一个错误。
上面代码中，我们必须先通过`template.New`创建模板，然后调用`Funcs`设置自定义函数，最后再解析模板文件。
因为模板文件中使用了`fdate`，未设置之前会解析失败。

## 上下文感知

上下文感知是`html/template`库的一个非常有趣的特性。根据需要替换的文本在文档中所处的位置，模板在显示这些内容的时候会对其进行相应的修改。
上下文感知的一个常见用途就是对内容进行转义。如果需要显示的是 HTML 的内容，那么进行 HTML 转义。如果显示的是 JavaScript 内容，那么进行 JavaScript 转义。
Go 模板引擎还能识别出内容中的 URL 或 CSS，可以对它们实施正确的转义。

```golang
func contextAwareHandler(w http.ResponseWriter, r *http.Request) {
	t := template.Must(template.ParseFiles("context-aware.html"))
	t.Execute(w, `He saied: <i>"She's alone?"</i>`)
}

mux.HandleFunc("/contextAware", contextAwareHandler)
```

模板文件`context-aware.html`：

```html
<div>{{ . }}</div>
<div><a href="/{{ . }}">Path</a></div>
<div><a href="/?q={{ . }}">Query</a></div>
<div><a onclick="f('{{ . }}')">JavaScript</a></div>
```

编译、运行程序，使用 curl 访问`localhost:8080/contextAware`，得到下面的内容：

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Go Web 编程之 模板（二）</title>
</head>
<body>
  <div>He saied: &lt;i&gt;&#34;She&#39;s alone?&#34;&lt;/i&gt;</div>
  <div><a href="/He%20saied:%20%3ci%3e%22She%27s%20alone?%22%3c/i%3e">Path</a></div>
  <div><a href="/?q=He%20saied%3a%20%3ci%3e%22She%27s%20alone%3f%22%3c%2fi%3e">Query</a></div>
  <div><a onclick="f('He saied: \x3ci\x3e\x22She\x27s alone?\x22\x3c\/i\x3e')">JavaScript</a></div>
</body>
</html>
```

我们依次来看，需要呈现的数据是`He saied: <i>"She's alone?"</i>`：

* 第一个`div`中，直接在页面中显示，其中 HTML 标签`<i>`和单、双引号都被转义了；
* 第二个`div`中，数据出现在 URL 的路径中，所有非法的路径字符都被转义了，包括空格、尖括号、单双引号；
* 第三个`div`中，数据出现在查询字符串中，除了 URL 路径中非法的字符，还有冒号（`:`）、问号（`?`）和斜杠也被转义了；
* 第四个`div`中，数据出现在 OnClick 代码中，单双引号和斜杠都被转义了。

这四种转义方式又有所不同，第一种转义为 HTML 字符实体，第二、三种转义为 URL 转义字符（`%`后跟字符编码的十六进制表示） ，第四种转义为 Go 中的十六进制字符表示。

## 防御 XSS 攻击

XSS 是一种常见的攻击形式。在论坛之类的可以接受用户输入的网站，攻击者可以内容中添加`<scrpit>`标签。如果网站未对输入的内容进行处理，
其他用户浏览该页面时，`<script>`标签中的内容就会被执行，泄露用户的私密信息或利用用户的权限做破坏。

```golang
func xssHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method == "POST" {
		t := template.Must(template.ParseFiles("xss-display.html"))
		t.Execute(w, r.FormValue("comment"))
	} else {
		t := template.Must(template.ParseFiles("xss-form.html"))
		t.Execute(w, nil)
	}
}

mux.HandleFunc("/xss", xssHandler)
```

模板文件`xss-form.html`：

```html
<form action="/xss" method="post">
	Comment: <input name="comment" type="text">
	<hr/>
	<button id="submit">Submit</button>
</form>
```

模板文件`xss-display.html`：

```html
{{ . }}
```

处理器中我们根据请求方法的不同进行不同的处理。GET 请求返回一个表单页面，POST 请求显示用户输入的评论信息。

编译、运行程序，打开浏览器输入`localhost:8080/xss`，然后在输入框中输入`<script>alert("Boom!")</script>`：

![](/img/in-post/goweb/template1.png#center)

点击 Submit 按钮：

![](/img/in-post/goweb/template2.png#center)

`alert`代码并没有执行，为什么？

我们之前说过 Go 模板有上下文感知的功能，它检测到在 HTML 页面中，所以输入数据会被转义。查看网页源码可以看到转义后的结果：

![](/img/in-post/goweb/template3.png#center)

那么如何才能不转义呢？`html/template`提供了`HTML`类型，Go 模板不会对该类型的变量进行转义。如果我们把上面的处理器修改为：

```golang
func xssHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method == "POST" {
		t := template.Must(template.ParseFiles("xss-display.html"))
		t.Execute(w, template.HTML(r.FormValue("comment")))
	} else {
		t := template.Must(template.ParseFiles("xss-form.html"))
		t.Execute(w, nil)
	}
}
```

再次实验，提交上面的内容，在 Chrome 浏览器中显示：

![](/img/in-post/goweb/template4.png#center)

这意味着攻击者可以让网站上的其他用户执行任意可能的攻击代码。

## 嵌套模板

上一篇文章中说过，嵌套模板在定义网页布局时非常有用。

```golang
type NestInfo struct {
	Name string
	Todos []string
}

func nestHandler(w http.ResponseWriter, r *http.Request) {
	t := template.Must(template.ParseFiles("layout.html"))

	data := NestInfo {
		"dj", []string{"Homework", "Game", "Cleaning"},
	}
	t.ExecuteTemplate(w, "layout", data)
}
```

模板文件`layout.html`：

```html
{{ define "layout" }}
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Go Web 编程之 模板（二）</title>
</head>
<body>
  {{ template "header" . }}

  {{ template "content" . }}

  {{ template "footer" . }}
</body>
</html>
{{ end }}
 
{{ define "header" }}
<h1>Hi, {{ .Name }}!</h1>
{{ end }}

{{ define "content" }}
Todo:
<ul>
  {{ range .Todos }}
  <li>{{ . }}</li>
  {{ end }}
</ul>
{{ end }}

{{ define "footer" }}
Copyright © 2020 darjun.
{{ end }}
```

我们将网页主体拆为三个部分：header、content和footer。header 一般显示导航栏，欢迎信息。content 展示主体内容。footer 中显示版权，联系方式等信息。

解析之后，我们需要调用`ExecuteTemplate`执行`layout`模板，不能直接使用`Execute`，因为主模板中只是定义了一些模板，没有具体的内容。

### 使用块动作定义默认模板

在上面的`layout.html`文件中，我们定义了模板`layout`，`layout`中使用了在本模板文件中定义的三个模板`header/content/footer`。其实这几个模板的定义和使用可以通过`block`动作合并在一起：

```html
{{ define "layout" }}
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Go Web 编程之 模板（二）</title>
</head>
<body>
  {{ block "header" . }}
  <h1>Hi, {{ .Name }}!</h1>
  {{ end }}

  {{ block "content" . }}
  Todo:
  <ul>
    {{ range .Todos }}
    <li>{{ . }}</li>
    {{ end }}
  </ul>
  {{ end }}
	
  {{ block "footer" . }}
  Copyright © 2020 darjun.
  {{ end }}
</body>
</html>
```

`block`相当于定义模板后立即使用。

```
{{ block "name" arg }}
...
{{ end }}
```

等价于：

```
{{ define "name" }}
...
{{ end }}

{{ template "name" arg }}
```

## 总结

本文详细介绍了`html/template`库，更多地通过案例来介绍，关注如何使用。模板在 Web 开发中是非常重要的部件，需要我们牢牢掌握。

## 参考

1. [Go Web 编程](https://book.douban.com/subject/27204133/)
2. [html/template](https://golang.org/pkg/html/template/)文档

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)