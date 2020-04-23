---
layout:    post
title:    "Go 每日一库之 go-app"
subtitle: 	"每天学习一个 Go 库"
date:		2020-04-22T21:30:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/04/22/godailylib/go-app"
categories: [
	"Go"
]
---

## 简介

[`go-app`](https://github.com/maxence-charriere/go-app)是一个使用 Go + WebAssembly 技术编写**渐进式 Web 应用**的库。WebAssembly 是一种可以运行在现代浏览器中的新式代码。近两年来，WebAssembly 技术取得了较大的发展。我们现在已经可以使用 C/C++/Rust/Go 等高级语言编写 WebAssembly 代码。本来就来介绍`go-app`这个可以方便地使用 Go 语言来编写 WebAssembly 代码的库。

## 快速使用

`go-app`对 Go 语言版本有较高的要求（**Go 1.14+**），而且必须使用`Go module`。先创建一个目录并初始化`Go Module`（Win10 + Git Bash）：

```golang
$ mkdir go-app && cd go-app
$ go mod init
```

然后下载安装`go-app`包：

```golang
$ go get -u -v github.com/maxence-charriere/go-app/v6
```

至于`Go module`的详细使用，去看煎鱼大佬的[Go Modules 终极入门](https://juejin.im/post/5e57537cf265da57584da62b)。

首先，我们要编写 WebAssembly 程序：

```golang
package main

import "github.com/maxence-charriere/go-app/v6/pkg/app"

type Greeting struct {
  app.Compo
  name string
}

func (g *Greeting) Render() app.UI {
  return app.Div().Body(
    app.Main().Body(
      app.H1().Body(
        app.Text("Hello, "),
        app.If(g.name != "",
          app.Text(g.name),
        ).Else(
          app.Text("World"),
        ),
      ),
    ),
    app.Input().
      Value(g.name).
      Placeholder("What is your name?").
      AutoFocus(true).
      OnChange(g.OnInputChange),
  )
}

func (g *Greeting) OnInputChange(src app.Value, e app.Event) {
  g.name = src.Get("value").String()
  g.Update()
}

func main() {
  app.Route("/", &Greeting{})
  app.Run()
}
```

在`go-app`中使用**组件**来划分功能模块，每个组件结构中必须内嵌`app.Compo`。组件要实现`Render()`方法，在需要显示该组件时会调用此方法返回显示的页面。`go-app`使用**声明式语法**，完全使用 Go 就可以编写 HTML 页面，上面绘制 HTML 的部分比较好理解。上面代码中还实现了一个输入框的功能，并为它添加了一个监听器。每当输入框内容有修改，`OnInputChange`方法就会调用，`g.Update()`会使该组件重新渲染显示。

最后将该组件挂载到路径`/`上。

编写 WebAssembly 程序之后，需要使用交叉编译的方式将它编译为`.wasm`文件：

```golang
$ GOARCH=wasm GOOS=js go build -o app.wasm
```

如果编译出现错误，使用`go version`命令检查 Go 是否是 1.14 或更新的版本。

接下来，我们需要编写一个 Go Web 程序使用这个`app.wasm`：

```golang
package main

import (
  "log"
  "net/http"

  "github.com/maxence-charriere/go-app/v6/pkg/app"
)

func main() {
  h := &app.Handler{
    Title:  "Go-App",
    Author: "dj",
  }

  if err := http.ListenAndServe(":8080", h); err != nil {
    log.Fatal(err)
  }
}
```

`go-app`提供了一个`app.Handler`结构，它会自动查找同目录下的`app.wasm`（这也是为什么将目标文件设置为`app.wasm`的原因）。然后我们将前面编译生成的`app.wasm`放到同一目录下，执行该程序：

```golang
$ go run main.go
```

默认显示`"Hello World"`：

![](/img/in-post/godailylib/go-app1.png#center)

在输入框中输入内容之后，显示会随之变化：

![](/img/in-post/godailylib/go-app2.png#center)

可以看到，`go-app`为我们设置了一些基本的样式，网页图标等。

## 简单原理

GitHub 上这张图很好地说明了 HTTP 请求的执行流程：

![](/img/in-post/godailylib/go-app3.png#center)

用户请求先到`app.Handler`层，它会去`app.wasm`中执行相关的路由逻辑、去磁盘上查找静态文件。响应经由`app.Handler`中转返回给用户。用户就看到了`app.wasm`渲染的页面。实际上，在本文中我们只需要编写一个 Go Web 程序，每次编写新的 WebAssembly 之后，将新编译生成的 app.wasm 文件拷贝到 Go Web 目录下重新运行程序即可。注意，**如果页面未能及时刷新，可能是缓存导致的，可尝试清理浏览器缓存**。

## 组件

自定义一个组件很简单，只需要将`app.Compo`内嵌到结构中即可。实现`Render()`方法可定义组件的外观，实际上`app.Compo`有一个默认的外观，我们可以这样来查看：

```golang
func main() {
  app.Route("/app", &app.Compo{})
  app.Run()
}
```

编译生成`app.wasm`之后，一开始的 Go Web 程序不需要修改，直接运行，打开浏览器查看：

![](/img/in-post/godailylib/go-app5.png#center)

### 事件处理

在**快速开始**中，我们还介绍了如何使用事件。使用声明式语法`app.Input().OnChange(handler)`即可监听内容变化。事件处理函数必须为`func (src app.Value, e app.Event)`类型，`app.Value`是触发对象，`app.Event`是事件的内容。通过`app.Value`我们可以得到输入框内容、选择框的选项等信息，通过`app.Event`可以得到事件的信息，是鼠标事件、键盘事件还是其它事件：

```golang
type ShowSelect struct {
  app.Compo
  option string
}

func (s *ShowSelect) Render() app.UI {
  return app.Div().Body(
    app.Main().Body(
      app.H1().Body(
        app.If(s.option == "",
          app.Text("Please select!"),
        ).Else(
          app.Text("You've selected "+s.option),
        ),
      ),
    ),
    app.Select().Body(
      app.Option().Body(
        app.Text("apple"),
      ),
      app.Option().Body(
        app.Text("orange"),
      ),
      app.Option().Body(
        app.Text("banana"),
      ),
    ).
      OnChange(s.OnSelectChange),
  )
}

func (s *ShowSelect) OnSelectChange(src app.Value, e app.Event) {
  s.option = src.Get("value").String()
  s.Update()
}

func main() {
  app.Route("/", &ShowSelect{})
  app.Run()
}
```

上面代码显示一个选择框，当选项改变时上面显示的文字会做相应的改变。初始时：

![](/img/in-post/godailylib/go-app6.png#center)

选择后：

![](/img/in-post/godailylib/go-app7.png#center)

### 嵌套组件

组件可以嵌套使用，即在一个组件中使用另一个组件。渲染时将内部的组件表现为外部组件的一部分：

```golang
type Greeting struct {
  app.Compo
}

func (g *Greeting) Render() app.UI {
  return app.P().Body(
    app.Text("Hello, "),
    &Name{name: "dj"},
  )
}

type Name struct {
  app.Compo
  name string
}

func (n *Name) Render() app.UI {
  return app.Text(n.name)
}

func main() {
  app.Route("/", &Greeting{})
  app.Run()
}
```

上面代码在组件`Greeting`中内嵌了一个`Name`组件，运行显示：

![](/img/in-post/godailylib/go-app8.png#center)

### 生命周期

`go-app`提供了组件的 3 个生命周期的钩子函数：

* `OnMount`：当组件插入到 DOM 时调用；
* `OnNav`：当一个组件所在页面被加载、刷新时调用；
* `OnDismount`：当一个组件从页面中移除时调用。

例如：

```golang
type Foo struct {
  app.Compo
}

func (*Foo) Render() app.UI {
  return app.P().Body(
    app.Text("Hello World"),
  )
}

func (*Foo) OnMount() {
  fmt.Println("component mounted")
}

func (*Foo) OnNav(u *url.URL) {
  fmt.Println("component navigated:", u)
}

func (*Foo) OnDismount() {
  fmt.Println("component dismounted")
}

func main() {
  app.Route("/", &Foo{})
  app.Run()
}
```

编译运行，在浏览器中打开页面，打开**浏览器控制台**观察输出：

```golang
component mounted
component navigated: http://localhost:8080/
```

## 编写 HTML

在前面的例子中我们已经看到了如何使用**声明式语法**编写 HTML 页面。`go-app`为所有标准的 HTML 元素都提供了相关的类型。创建这些对象的方法名也比较好记，就是元素名的首字母大写。如`app.Div()`创建一个`div`元素，`app.P()`创建一个`p`元素，`app.H1()`创建一个`h1`元素等等。在`go-app`中，这些结构都是暴露出对应的接口供开发者使用的，如`div`对应`HTMLDiv`接口：

```golang
type HTMLDiv interface {
  Body(nodes ...Node) HTMLDiv
  Class(v string) HTMLDiv
  ID(v string) HTMLDiv
  Style(k, v string) HTMLDiv

  OnClick(h EventHandler) HTMLDiv
  OnKeyPress(h EventHandler) HTMLDiv
  OnMouseOver(h EventHandler) HTMLDiv
}
```

可以看到每个方法都返回该`HTMLDiv`自身，所以支持**链式调用**。调用这些方法可以设置元素的各方面属性：

* `Class`：添加 CSS Class；
* `ID`：设置 ID 属性；
* `Style`：设置内置样式；
* `Body`：设置元素内容，可以随意嵌套。`div`中包含`h1`和`p`，`p`中包含`img`等；

和设置事件监听：

* `OnClick`：点击事件；
* `OnKeyPress`：按键事件；
* `OnMouseOver`：鼠标移过事件。

例如下面代码：

```golang
app.Div().Body(
  app.H1().Body(
    app.Text("Title"),
  ),
  app.P().ID("id").
    Class("content").Body(
      app.Text("something interesting"),
    ),
)
```

相当于 HTML 代码：

```html
<div>
  <h1>title</h1>
  <p id="id" class="content">
    something interesting
  </p>
</div>
```

### 原生元素

我们可以在`app.Raw()`中直接写 HTML 代码，`app.Raw()`会生成对应的`app.UI`返回：

```golang
svg := app.Raw(`
<svg width="100" height="100">
    <circle cx="50" cy="50" r="40" stroke="green" stroke-width="4" fill="yellow" />
</svg>
`)
```

但是这种写法是不安全的，因为没有检查 HTML 的结构。

### 条件

我们在最开始的例子中就已经用到了条件语句，条件语句对应 3 个方法：`If()/ElseIf()/Else()`。

`If`和`ElseIf`接收两个参数，第一个参数为`bool`值。如果为`true`，则显示第二个参数（类型为`app.UI`），否则不显示。

`Else`必须在`If`或`ElseIf`后使用，如果前面的条件都不满足，则显示传入`Else`方法的`app.UI`：

```golang
type ScoreUI struct {
  app.Compo
  score int
}

func (c *ScoreUI) Render() app.UI {
  return app.Div().Body(
    app.If(c.score >= 90,
      app.H1().
        Style("color", "green").
        Body(
          app.Text("Good!"),
        ),
    ).ElseIf(c.score >= 60,
      app.H1().
        Style("color", "orange").
        Body(
          app.Text("Pass!"),
        ),
    ).Else(
      app.H1().
        Style("color", "red").
        Body(
          app.Text("fail!"),
        ),
    ),
    app.Input().
      Value(c.score).
      Placeholder("Input your score?").
      AutoFocus(true).
      OnChange(c.OnInputChange),
  )
}

func (c *ScoreUI) OnInputChange(src app.Value, e app.Event) {
  score, _ := strconv.ParseUint(src.Get("value").String(), 10, 32)
  c.score = int(score)
  c.Update()
}

func main() {
  app.Route("/", &ScoreUI{})
  app.Run()
}
```

上面我们根据输入的分数显示对应的文字，`90`及以上显示绿色的`Good!`，`60-90`之间显示橙色的`Pass!`，小于`60`显示红色的`Fail!`。下面是运行结果：

![](/img/in-post/godailylib/go-app10.png#center)

![](/img/in-post/godailylib/go-app11.png#center)

![](/img/in-post/godailylib/go-app12.png#center)

### `Range`

假设我们要编写一个 HTML 列表，当前有一个字符串的切片。如果一个个写就太繁琐了，而且不够灵活，且容易出错。这时就可以使用`Range()`方法了：

```golang
type RangeUI struct {
  app.Compo
  name string
}

func (*RangeUI) Render() app.UI {
  langs := []string{"Go", "JavaScript", "Python", "C"}
  return app.Ul().Body(
    app.Range(langs).Slice(func(i int) app.UI {
      return app.Li().Body(
        app.Text(langs[i]),
      )
    }),
  )
}

func main() {
  app.Route("/", &RangeUI{})
  app.Run()
}
```

`Range()`可以对切片或`map`中每一项生成一个`app.UI`，然后平铺在某个元素的`Body()`方法中。

运行结果：

![](/img/in-post/godailylib/go-app13.png#center)

### 上下文菜单

在`go-app`中，我们可以很方便的自定义右键弹出的菜单，并且为菜单项编写响应：

```golang
type ContextMenuUI struct {
  app.Compo
  name string
}

func (c *ContextMenuUI) Render() app.UI {
  return app.Div().Body(
    app.Text("Hello, World"),
  ).OnContextMenu(c.OnContextMenu)
}

func (*ContextMenuUI) OnContextMenu(src app.Value, event app.Event) {
  event.PreventDefault()

  app.NewContextMenu(
    app.MenuItem().
      Label("item 1").
      OnClick(func(src app.Value, e app.Event) {
        fmt.Println("item 1 clicked")
      }),
    app.MenuItem().Separator(),
    app.MenuItem().
      Label("item 2").
      OnClick(func(src app.Value, e app.Event) {
        fmt.Println("item 2 clicked")
      }),
  )
}

func main() {
  app.Route("/", &ContextMenuUI{})
  app.Run()
}
```

我们在`OnContextMenu`中调用了`event.PreventDefault()`阻止默认菜单的弹出。看运行结果：

![](/img/in-post/godailylib/go-app14.png#center)

点击菜单项，观察控制台输出~

## `app.Handler`

上面我们都是使用`go-app`内置的`app.Handler`处理客户端的请求。我们只设置了简单的两个属性`Author`和`Title`。`app.Handler`还有其它很多字段可以定制：

```golang
type Handler struct {
  Author string
  BackgroundColor string
  CacheableResources []string
  Description string
  Env Environment
  Icon Icon
  Keywords []string
  LoadingLabel string
  Name string
  RawHeaders []string
  RootDir string
  Scripts []string
  ShortName string
  Styles []string
  ThemeColor string
  Title string

  UseMinimalDefaultStyles bool
  Version string
}
```

* `Icon`：设置应用图标；
* `Styles`：CSS 样式文件；
* `Scripts`：JS 脚本文件。

CSS 和 JS 文件必须在`app.Handler`中声明。下面是一个示例`app.Handler`：

```golang
h := &app.Handler{
  Name:        "Luck",
  Author:      "Maxence Charriere",
  Description: "Lottery numbers generator.",
  Icon: app.Icon{
    Default: "/web/icon.png",
  },
  Keywords: []string{
    "EuroMillions",
    "MEGA Millions",
    "Powerball",
  },
  ThemeColor:      "#000000",
  BackgroundColor: "#000000",
  Styles: []string{
    "/web/luck.css",
  },
  Version: "wIKiverSiON",
}
```

## 本文代码

本文中 WebAssembly 代码都在各自的目录中。Go Web 演示代码在 web 目录中。先进入某个目录，使用下面的命令编译：

```golang
$ GOARCH=wasm GOOS=js go build -o app.wasm
```

然后将生成的`app.wasm`拷贝到`web`目录：

```golang
$ cp app.wasm ../web/
```

切换到 web 目录，启动服务器：

```golang
$ cd ../web/
$ go run main.go
```

## 总结

本文介绍如何使用`go-app`编写基于 WebAssembly 的 Web 应用程序。可能有人会觉得，`go-app`编写 HTML 的方式有点繁琐。但是我们可以写一个转换程序将普通的 HTML 代码转为`go-app`代码，感兴趣可以自己实现一下。WebAssembly 技术非常值得关注一波~

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. go-app GitHub：[https://github.com/maxence-charriere/go-app](https://github.com/maxence-charriere/go-app)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)