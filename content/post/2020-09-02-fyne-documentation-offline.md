---
layout:    post
title:    "在本地运行 fyne 官网"
subtitle: 	"学以致用"
date:		2020-09-02T20:07:55
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/09/02/godailylib/fyne3"
categories: [
	"Go"
]
---

## 简介

要深入学习和理解一个框架，官方文档是必须要仔细阅读的。fyne 官网有非常系统和详尽的文档。官方网站：[https://fyne.io/](https://fyne.io/)。有时候我们会有这样一个需求——离线查看文档。我经常乘坐高铁来往杭州、上海两地，地铁、高铁上通常网络比较差，甚至没有网络。为此我特地去研究了一番怎样搭建 fyne 离线文档。

首先，我找到了 fyne 官方网站的 GitHub 仓库，网址为[https://github.com/fyne-io/developer.fyne.io](https://github.com/fyne-io/developer.fyne.io)。很快我发现 fyne 官网是采用 jekyll 构建的。jekyll 是采用 ruby 语言编写的**静态网页工具**。jekyll 常用于搭建个人博客。它支持使用 markdown 语法编写文章，然后自动生成相应的静态页面托管在远程主机上供用户访问。为了能本地运行文档，我们必须先安装 ruby + jekyll 环境。

## Windows

在 Windows 平台上，我们可以从[https://rubyinstaller.org/downloads/](https://rubyinstaller.org/downloads/)下载 RubyInstaller 直接双击安装。这里我们下载 Ruby+Devkit 2.6.6-1(x64)。

![](/img/in-post/godailylib/fyne-website1.png#center)

这会同时安装 ruby 基本环境和 MSYS2 开发环境（用来编写和编译 C 扩展）。

默认会将可执行程序所在目录加入 PATH 中：

![](/img/in-post/godailylib/fyne-website2.png#center)

MSYS2 开发环境默认也是安装的：

![](/img/in-post/godailylib/fyne-website3.png#center)

ruby 安装完成之后会使用 ridk 安装 MSYS2 开发环境：

![](/img/in-post/godailylib/fyne-website4.png#center)

安装完成之后，打开 cmd，输入`ruby -v`。如果输出正确的 ruby 版本信息，说明安装成功。如果提示命令找不到，则未安装成功，或环境变量设置不正确：

![](/img/in-post/godailylib/fyne-website5.png#center)

成熟的编译语言通常都有相应的包管理工具，用于下载和管理依赖。正如 node 有 npm，python 有 pip，rust 有 cargo，ruby 也有它的 gem。gem 需要独立下载安装。下载地址[https://rubygems.org/pages/download](https://rubygems.org/pages/download)。我们可以直接下载压缩包 TGZ/ZIP，或者 GEM 文件，或者使用 git 从 GitHub 仓库克隆。

1. 下载压缩包之后，解压；
2. cd 到解压之后的目录；
3. 执行 ruby setup.rb 安装。

安装完成之后，打开 cmd，输入`gem -v`。如果输出正确的 gem 版本信息，说明安装成功。如果提示命令找不到，则安装失败，或环境变量设置不正确：

![](/img/in-post/godailylib/fyne-website6.png#center)

## Mac

在 Mac 上可以直接使用 brew 安装 ruby 和 gem。

## 安装 jekyll

gem 安装完成之后，安装 jekyll 就很简单了。只需要执行`gem install jekyll`等待安装完成。

![](/img/in-post/godailylib/fyne-website7.png#center)

## clone 官网仓库

我们使用 git 将官网仓库 clone 到本地计算机上：

```cmd
$ git clone git@github.com:fyne-io/developer.fyne.io.git
```

![](/img/in-post/godailylib/fyne-website8.png#center)

## 安装依赖

**cd**到`developer.fyne.io`目录，使用`gem`安装该网站的所有依赖：

```cmd
$ gem install -g
```

gem 安装依赖的速度取决于你的网速，耐心等待~

![](/img/in-post/godailylib/fyne-website9.png#center)

## 本地运行网站

一切准备就绪，接下来只需要输入下面的指令网站就在本地运行起来了：

```cmd
$ jekyll serve
```

一般会出现下面的错误：

![](/img/in-post/godailylib/fyne-website10.png#center)

这是应该有个依赖的版本问题，我们可以使用错误提示中的命令`bundle`启动：

```cmd
$ bundle exec jekyll serve
```

运行成功：

![](/img/in-post/godailylib/fyne-website11.png#center)

这时，我们就可以在浏览器中输入：`http://localhost:4000`就可以在本地随意浏览官网了。

![](/img/in-post/godailylib/fyne-website12.png#center)

## 总结

本文介绍如何搭建 fyne 离线文档，大家可以触类旁通~

## 参考

1. fyne.developer.io GitHub：[https://github.com/fyne-io/developer.fyne.io](https://github.com/fyne-io/developer.fyne.io)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)