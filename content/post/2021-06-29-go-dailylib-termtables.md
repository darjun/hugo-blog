---
layout:    post
title:    "Go 每日一库之 termtables"
subtitle: "每天学习一个 Go 库"
date:     2021-06-29T22:18:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/06/29/godailylib/termtables"
categories: [
  "Go"
]
---

## 简介

今天学个简单点的😀，[`termtables`](github.com/scylladb/termtables)处理表格形式数据的输出。适用于随时随地的输出一些状态或统计数据，便于观察和调试。是一个很小巧的工具库。我在学习[`dateparse`](https://darjun.github.io/2021/06/24/godailylib/dateparse/)库时偶尔见遇到了这个库。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir termtables && cd termtables
$ go mod init github.com/darjun/go-daily-lib/termtables
```

安装`termtables`库：

```cmd
$ go get -u github.com/scylladb/termtables
```

最原始的`termtables`库为`github.com/apcera/termtables`，然后原始仓库已经被删除了。目前使用的都是其他人 fork 的仓库。

使用：

```golang
package main

import (
  "fmt"
  "github.com/scylladb/termtables"
)

func main() {
  t := termtables.CreateTable()
  t.AddHeaders("User", "Age")
  t.AddRow("dj", 18)
  t.AddRow("darjun", 30)
  fmt.Println(t.Render())
}
```

运行：

```golang
$ go run main.go
+--------+-----+
| User   | Age |
+--------+-----+
| dj     | 18  |
| darjun | 30  |
+--------+-----+
```

使用很方便，首先调用`termtables.CreateTable()`创建一个表格对象，调用该对象的`AddHeader()`方法添加头部信息，然后调用`AddRow()`逐行添加数据。最后调用`Render()`返回渲染后的表格字符串。

## 模式

处理普通的文本表格，`termtables`还支持输出 HTML 和 Markdown 格式的表格。只需要调用表格对象的`SetModeHTML()/SetModeMarkdown()`方法设置一些模式即可 。

```golang
func main() {
  t := termtables.CreateTable()
  t.AddHeaders("User", "Age")
  t.AddRow("dj", 18)
  t.AddRow("darjun", 30)
  fmt.Println("HTML:")
  t.SetModeHTML()
  fmt.Println(t.Render())

  fmt.Println("Markdown:")
  t.SetModeMarkdown()
  fmt.Println(t.Render())
}
```

运行：

```golang
$ go run main.go
HTML:
<table class="termtable">
<thead>
<tr><th>User</th><th>Age</th></tr>
</thead>
<tbody>
<tr><td>dj</td><td>18</td></tr>
<tr><td>darjun</td><td>30</td></tr>
</tbody>
</table>

Markdown:
| User   | Age |
| ------ | --- |
| dj     | 18  |
| darjun | 30  |
```

输出的格式可以直接用在 Markdown/HTML 文件中。

## 总结

今天轻松一下，了解了一个小巧的工具库`termtables`。虽然自己实现一个类似的也不复杂，`termtables`库额外帮我们处理了编码、字宽等比较繁琐的细节。有需要在写示例程序中打印类似表格之类的数据不妨试一试`termtables`。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)
2. termtables GitHub：[github.com/scylladb/termtables](github.com/scylladb/termtables)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~