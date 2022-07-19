---
layout:    post
title:    "Go 每日一库之 gore"
subtitle: "每天学习一个 Go 库"
date:     2022-07-18T22:30:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2022/07/18/godailylib/gore"
categories: [
  "Go"
]
---

## 简介

周末闲逛 GitHub 的时候发现一个很好玩的 Go 语言的 REPL（read-eval-print-loop）工具。本文和大家分享一下这个工具：[`gore`](https://github.com/x-motemen/gore)。

首先放一张 GitHub 的动图感受一下：

![](/img/in-post/godailylib/gore1.gif#center)

## 快速使用

gore 是一个命令行工具，需要配合 Go Module 安装。Go 环境安装完成之后，执行下面的命令安装 gore：

```cmd
$ go install github.com/x-motemen/gore/cmd/gore@latest
```

go install 命令会将会在`$GOPATH/bin`目录生成的 gore 可执行文件。强烈推荐大家把`$GOPATH/bin`加入到系统的可执行文件搜索目录（即`$PATH`）中。

执行下面的命令即可进入 Go 的 REPL：

```cmd
$ gore
```

## 命令

目前支持的命令还不多。我将命令分为两类，一种是基础命令，一种是代码相关命令。

注意，在 gore 中所有的命令必须以`:`开始。非`:`开始的行会被识别为代码。

### 基础命令

* help：显示命令列表

![](/img/in-post/godailylib/gore2.png#center)

* doc：我觉得这个命令挺有意思，它可以显示一个包、结构体或接口的文档。如果要查看标准库的文档，例如`math/bits`，可以直接键入命令`:doc bits`。

![](/img/in-post/godailylib/gore3.png#center)

注意，要查看文档的包必须先使用`:import`命令导入。并且传给`:doc`命令的参数必须是包去掉路径的部分，例如我们不能使用`:doc math/bits`，必须使用`:doc bits`。

doc 也可以用来查看第三方库的文档。也是先导入后查看：

![](/img/in-post/godailylib/gore4.png#center)

* quit：退出 REPL。

### 代码命令

* import：导入包的命令。既可以导入标准库的包，也可以导入第三方库的包，gore 会自动调用 go get 去下载第三方库

![](/img/in-post/godailylib/gore5.png#center)

* type：输出表达式会变量的类型

![](/img/in-post/godailylib/gore6.png#center)

* print：我们在 gore 中输入的代码都会存放在一个临时文件中。print 命令会打印这个临时文件的内容。

![](/img/in-post/godailylib/gore7.png#center)

* clear：清空临时文件。执行 clear 命令之后 print 将会打印一个空的 main 函数。

![](/img/in-post/godailylib/gore8.png#center)

* write `[<filename>]`：将临时文件保存到指定的路径中。

## 特性

* 行编辑：通过左右键在行内移动光标，退格键删除光标前的内容，重新输入内容
* 命令历史：gore 会保存之前输入的**命令**历史，可以通过上下键找出之前执行的命令。
* 多行输入：如果 gore 检测到输入还没有结束就换行了，它会等待输入结束再执行这条语句。例如 if 语句
* 自动补全：使用 TAB 键可以自动补全命令，**不能补全代码**
* 代码补全：这个我没试过，文档介绍说需要 gocode 配合
* 自动导入：使用选项`-autoimport`启动可以自动导入要使用的包

![](/img/in-post/godailylib/gore9.png#center)

## 缺点

体验下来，我觉得有几个缺点：

* 启动速度较慢。键入 gore 命令按下回车要等好几秒。其他命令的执行速度也不快

* doc 命令的限制有点奇怪。为什么包不能加路径？go doc 是可以加路径的。也有可能我使用的姿势不对，有知道的可以指点一二😀

![](/img/in-post/godailylib/gore10.png#center)

* 多行输入有点反直觉。如果我没有输入完整的代码，它会一直等着我输入。可是我已经不想输入了。有一次我键入 type 命令时忘记加`:`了，就变成这样了：

![](/img/in-post/godailylib/gore11.png#center)

当然，可以通过`Ctrl + C`终止输入，这个让我摸索了好一会儿。我个人使用其他软件的经验是连续几个空行就可以终止了。这一点严格不算缺点，只是不符合我的习惯。

## 总结

总体来说 gore 是一个比较好玩的工具，期待项目发展壮大！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gore GitHub：[github.com/x-motemen/gore](https://github.com/x-motemen/gore)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)