---
layout:     post
title:      "搭建go开发环境"
subtitle:   "\"goto go\""
date:       2019-01-28T22:05:08
author:     "darjun"
image: "/img/background2.jpg"
tags:
    - go
URL: "/2019/01/24/golang-dev-env/"
categories: [ Tech ]
---

## 概述

最近发现 visual studio code 很好用。本文介绍在 windows 上基于 visual studio code 搭建一个 go 语言的基本开发环境。

## 基本软件安装

**step 1. 安装 visual studio code：**

这个没啥好说的，去[官网](https://code.visualstudio.com/)下载安装。

**step 2. 安装 git for windows：**

go get 工具使用 git 来获取远程代码包。故而需要安装 git，去[官网](https://git-scm.com/downloads)下载安装。

**step 3. 安装 vscode 的 go 插件：**
在 vscode 中点击扩展按钮，搜索 go，安装 go 插件。


## 基本环境配置

环境变量的配置这里就不赘述了，网上相关教程非常多的。

**step 1. 配置`GOPATH`环境变量：**

`GOPATH`是 go 语言的一个特色，代码存放在`GOPATH`下的`src`目录中。可根据个人需要配置，我配置的是`D:\code\golang`。

**step 2. 配置`PATH`环境变量：**

因为 go 代码编译之后的可执行文件默认存放在`GOPATH`下的`bin`目录中，在`PATH`环境变量中添加`%GOPATH%\bin`。

## golang.org/x相关包安装

在学习 go 语言的过程中,经常需要用到第三方编写的包。其中`golang.org/x`相关包是 go 团队开发的,使用最为广泛。然而, `golang.org/x`在 google 的服务器上，google 的服务器 ip 被强大的 GFW 阻隔，没有梯子是过不去的。
这时就能感受到 go team 的贴心之处了—— go team 在 github 上建立了这些包的镜像。在 github 上 golang 的[项目主页](https://github.com/golang)上可以看到。仓库描述中含有`[mirror]`的基本都是`golang.org/x`相关包的镜像。
搜索`mirror`关键字可查看所有的镜像包：

![golang.org/x镜像](/img/in-post/vscode-go-dev/golang-mirror.png)

其实，常用的也就是`tools`，`net`，`lint`，`image`这几个包。

打开`git bash`终端（可在开始菜单搜索），然后创建相应的目录：
```
$ mkdir -p $GOPATH/src/golang.org/x
```

使用`git clone`命令将对应包克隆到刚创建的目录下：
```
$ git clone https://github.com/golang/tools.git $GOPATH/src/golang.org/x/tools
$ git clone https://github.com/golang/net.git $GOPATH/src/golang.org/x/net
$ git clone https://github.com/golang/lint.git $GOPATH/src/golang.org/x/lint
$ git clone https://github.com/golang/image.git $GOPATH/src/golang.org/x/image
```

这样就可以使用`import golang.org/x/tools`使用相关工具类了。

## vscode go工具安装

要想更顺畅的编写 go 程序，需要安装以下工具：
```
gocode
gopkgs
go-outline
go-symbols
guru
gorename
dlv
gocode-gomod
godef
godef-gomod
goreturns
golint
gotests
gomodifytags
impl
fillstruct
goplay
```

这些都可以通过 vscode 很方便地安装。在 vscode 中按下`F1`或`Ctrl+Shift+P`，输入`Go:Install/Update Tools`回车。安装完成之后就可以编写 go 代码了。如果没有前面安装`golang/x`包的步骤，这里多半会报`golang.org/x/tools`等找不到的错误。

此时，vscode 可以：
1. 智能提示。
2. 保存时自动 import 对应包。
3. 错误检查。
4. 保存时自动格式化文件。
5. 等等等等。

Enjoy!

关于我：
[个人主页](https://darjun.github.io) [简书](https://www.jianshu.com/u/0aca4aa367c8) [掘金](https://juejin.im/user/5be15200e51d4505466ce2f4)