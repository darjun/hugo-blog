---
layout:    post
title:    "Go 每日一库之 gomacro"
subtitle: "每天学习一个 Go 库"
date:     2022-08-19T20:30:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2022/08/19/godailylib/gomacro"
categories: [
  "Go"
]
---

## 简介

上一篇文章给大家介绍了一个好玩的 Go 语言的 REPL（read-eval-print-loop）工具[gore](https://darjun.github.io/2022/07/18/godailylib/gore/)。在使用过程中我发现 gore 启动速度较慢，其它命令的执行速度也很慢。前两天 github 给我推了[`gomacro`](https://github.com/cosmos72/gomacro)这个库。我试用了一下，感觉还不错。除了 REPL 之外，gomacro 可以用作一个库，从而可以在代码中执行动态的脚本。

## 快速使用

gomacro 是一个命令行工具，需要配合 Go Module 安装。Go 环境安装完成之后，执行下面的命令安装 gomacro：

```cmd
$ go install github.com/cosmos72/gomacro
```

go install 命令会将会在`$GOPATH/bin`目录生成的 gomacro 可执行文件。强烈推荐大家把`$GOPATH/bin`加入到系统的可执行文件搜索目录（即`$PATH`）中。

执行下面的命令即可进入 Go 的 REPL：

```cmd
$ gomacro
```
## 总结

总体来说 gomacro 是一个比较好玩的工具，期待项目发展壮大！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gomacro GitHub：[github.com/cosmos72/gomacro](https://github.com/cosmos72/gomacro)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)