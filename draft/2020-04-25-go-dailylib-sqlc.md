---
layout:    post
title:    "Go 每日一库之 sqlc"
subtitle: 	"每天学习一个 Go 库"
date:		2020-04-25T23:40:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/04/25/godailylib/sqlc"
categories: [
	"Go"
]
---

## 简介

在 Go 语言中编写数据库操作代码真的非常痛苦！`database/sql`标准库提供的都是比较底层的接口。我们需要编写大量重复的代码。大量的模板代码不仅写起来烦，而且还容易写错。有时候字段类型修改了一下，可能就需要改动很多地方；添加了一个新字段，之前使用`select *`查询语句的地方都要修改。如果有些地方有遗漏，可能就会造成运行时`panic`。即使使用 ORM 库，这些问题也不能完全解决！`sqlc`来了！`sqlc`可以根据我们编写的查询语句生成类型安全的、地道的 Go 接口代码，我们要做的只是调用这个方法。

## 快速使用

先安装：

```golang
$ go get github.com/kyleconroy/sqlc/cmd/sqlc
```

`sqlc`是一个命令行工具，上面代码会将可执行程序`sqlc`放到`$GOPATH/bin`目录下。我习惯把`$GOPATH/bin`目录加入到系统`PATH`中。所以可以执行使用这个命令。



## 总结

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. sqlc GitHub：[https://github.com/kyleconroy/sqlc](https://github.com/kyleconroy/sqlc)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)