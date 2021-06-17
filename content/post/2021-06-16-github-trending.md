---
layout:    post
title:    "用 Go 实现一个 GitHub Trending API"
subtitle: "自己动手，丰衣足食"
date:     2021-06-16T07:35:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go
URL: "2021/06/16/github-trending-api"
categories: [
  "Go"
]
---

## 背景

上一篇文章[Go 每日一库之 bubbletea](https://darjun.github.io/2021/06/11/godailylib/bubbletea/)我们介绍了炫酷的 TUI 程序框架 — [`bubbletea`](https://github.com/charmbracelet/bubbletea)。最后实现了一个拉取 GitHub Trending 仓库，并显示在控制台的程序。由于 GitHub 没有提供官方的 Trending API，我们用`goquery`自己实现了一个。上篇文章由于篇幅关系，没有介绍如何实现。本文我整理了一下代码，并以单独的代码库形式开放出来。

## 先观察

首先，我们来观察一下 GitHub Trending 的结构：

![](/img/in-post/opensource/trending1.png#center)

左上角可以切换仓库（Repositories）和开发者（Developers）。右边可以选择语言（Spoken Language，本地语言，汉语、英文等）、语言（Language，编程语言，Golang、C++等）和时间范围（Date Range，支持 3 个维度，Today、This week、This month）。

然后下面是每个仓库的信息：

① 仓库作者和名字

② 仓库描述

③ 主要使用的编程语言（创建仓库时设置的），也可能没有

④ 星数

⑤ fork 数

⑥ 贡献者列表

⑦ 选定的时间范围内（Today、This week、This month）新增多少星数

开发者页面也是类似的，只不过信息少了很多：

![](/img/in-post/opensource/trending2.png#center)

① 作者信息

② 最火的仓库信息

注意到切换的开发者页面后，URL 变成为`github.com/trending/developers`。另外当我们选择本地语言为中文、开发语言为 Go 和时间范围为 Today 后，URL 变为`https://github.com/trending/go?since=daily&spoken_language_code=zh`，通过在 query-string 中增加相应的键值对表示这种选择。


## 准备

在 GitHub 上创建仓库`ghtrending`，clone 到本地，执行`go mod init`初始化：

```cmd
$ go mod init github.com/darjun/ghtrending
```

然后执行`go get`下载`goquery`库：

```cmd
$ go get github.com/PuerkitoBio/goquery
```

根据仓库和开发者的信息定义两个结构：

```golang
type Repository struct {
  Author  string
  Name    string
  Link    string
  Desc    string
  Lang    string
  Stars   int
  Forks   int
  Add     int
  BuiltBy []string
}

type Developer struct {
  Name        string
  Username    string
  PopularRepo string
  Desc        string
}
```

## 开爬

要想使用`goquery`获取相应的信息，我们首先要知道，对应的网页结构。按 F12 打开 chrome 开发者工具，选择`Elements`页签，即可看到网页结构：

![](/img/in-post/opensource/trending3.png#center)

使用左上角的按钮就可以很快速的查看网页上任何内容的结构，我们点击单个仓库条目：

![](/img/in-post/opensource/trending4.png#center)

右边`Elements`窗口显示每个仓库条目对应一个`article`元素：

![](/img/in-post/opensource/trending5.png#center)

可以使用标准库`net/http`获取整个网页的内容：

```golang
resp, err := http.Get("https://github.com/trending")
```

然后从`resp`对象中创建`goquery`文档结构：

```golang
doc, err := goquery.NewDocumentFromReader(resp.Body)
```

有了文档结构对象，我们可以调用其`Find()`方法，传入选择器，这里我选择`.Box .Box-row`。`.Box`是整个列表`div`的 class，`.Box-row`是仓库条目的 class。这样的选择更精准。`Find()`方法返回一个`*goquery.Selection`对象，我们可以调用其`Each()`方法对每个条目进行解析。`Each()`接收一个`func(int, *goquery.Selection)`类型的函数，第二个参数即为每个仓库条目在 goquery 中的结构：

```golang
doc.Find(".Box .Box-row").Each(func(i int, s *goquery.Selection) {
})
```

接下来我们看看如何提取各个部分。在`Elements`窗口中移动，可以很直观的看到每个元素对应页面的哪个部分：

![](/img/in-post/opensource/trending6.gif#center)

我们找到仓库名和作者对应的结构：

![](/img/in-post/opensource/trending7.png#center)

它被包在`article`元素下的`h1`元素下的`a`元素内，作者名在`span`元素内，仓库名直接在`a`下，另外仓库的 URL 链接是`a`元素的`href`属性。我们来获取它们：

```golang
titleSel := s.Find("h1 a")
repo.Author = strings.Trim(titleSel.Find("span").Text(), "/\n ")
repo.Name = strings.TrimSpace(titleSel.Contents().Last().Text())
relativeLink, _ := titleSel.Attr("href")
if len(relativeLink) > 0 {
  repo.Link = "https://github.com" + relativeLink
}
```

仓库描述在`article`元素内的`p`元素中：

```golang
repo.Desc = strings.TrimSpace(s.Find("p").Text())
```

编程语言，星数，fork 数，贡献者（`BuiltBy`）和新增星数都在`article`元素的最后一个`div`中。编程语言、`BuiltBy`和新增星数在`span`元素内，星数和 fork 数在`a`元素内。如果编程语言未设置，则少一个`span`元素：

```golang
var langIdx, addIdx, builtByIdx int
spanSel := s.Find("div>span")
if spanSel.Size() == 2 {
  // language not exist
  langIdx = -1
  addIdx = 1
} else {
  builtByIdx = 1
  addIdx = 2
}

// language
if langIdx >= 0 {
  repo.Lang = strings.TrimSpace(spanSel.Eq(langIdx).Text())
} else {
  repo.Lang = "unknown"
}

// add
addParts := strings.SplitN(strings.TrimSpace(spanSel.Eq(addIdx).Text()), " ", 2)
repo.Add, _ = strconv.Atoi(addParts[0])

// builtby
spanSel.Eq(builtByIdx).Find("a>img").Each(func(i int, img *goquery.Selection) {
  src, _ := img.Attr("src")
  repo.BuiltBy = append(repo.BuiltBy, src)
})
```

然后是星数和 fork 数：

```golang
aSel := s.Find("div>a")
starStr := strings.TrimSpace(aSel.Eq(-2).Text())
star, _ := strconv.Atoi(strings.Replace(starStr, ",", "", -1))
repo.Stars = star
forkStr := strings.TrimSpace(aSel.Eq(-1).Text())
fork, _ := strconv.Atoi(strings.Replace(forkStr, ",", "", -1))
repo.Forks = fork
```

Developers 也是类似的做法。这里就不赘述了。使用`goquery`有一点需要注意，因为网页层级结构比较复杂，**我们使用选择器的时候尽量多限定一些元素、class，以确保找到的确实是我们想要的那个结构**。另外网页上获取的内容有很多空格，需要使用`strings.TrimSpace()`移除。

## 接口设计

基本工作完成之后，我们来看看如何设计接口。我想提供一个类型和一个创建该类型对象的方法，然后调用对象的`FetchRepos()`和`FetchDevelopers()`方法就可以获取仓库和开发者列表。但是我不希望用户了解这个类型的细节。所以我定义了一个接口：

```golang
type Fetcher interface {
  FetchRepos() ([]*Repository, error)
  FetchDevelopers() ([]*Developer, error)
}
```

我们定义一个类型来实现这个接口：

```golang
type trending struct{}

func New() Fetcher {
  return &trending{}
}

func (t trending) FetchRepos() ([]*Repository, error) {
}

func (t trending) FetchDevelopers() ([]*Developer, error) {
}
```

我们上面介绍的爬取逻辑就是放在`FetchRepos()`和`FetchDevelopers()`方法中。

然后，我们就可以在其他地方使用了：

```golang
import "github.com/darjun/ghtrending"

t := ghtrending.New()
repos, err := t.FetchRepos()

developers, err := t.FetchDevelopers()
```

### 选项

前面也说过，GitHub Trending 支持选定本地语言、编程语言和时间范围等。我们希望把这些设置作为选项，使用 Go 语言常用的选项模式/函数式选项（functional option)。先定义选项结构：

```golang
type options struct {
  GitHubURL  string
  SpokenLang string
  Language   string // programming language
  DateRange  string
}

type option func(*options)
```

然后定义 3 个`DataRange`选项：

```golang
func WithDaily() option {
  return func(opt *options) {
    opt.DateRange = "daily"
  }
}

func WithWeekly() option {
  return func(opt *options) {
    opt.DateRange = "weekly"
  }
}

func WithMonthly() option {
  return func(opt *options) {
    opt.DateRange = "monthly"
  }
}
```

以后可能还有其他范围的时间，留一个通用一点的选项：

```golang
func WithDateRange(dr string) option {
  return func(opt *options) {
    opt.DateRange = dr
  }
}
```

编程语言选项：

```golang
func WithLanguage(lang string) option {
  return func(opt *options) {
    opt.Language = lang
  }
}
```

本地语言选项，国家和代码分开，例如 Chinese 的代码为 cn：

```golang
func WithSpokenLanguageCode(code string) option {
  return func(opt *options) {
    opt.SpokenLang = code
  }
}

func WithSpokenLanguageFull(lang string) option {
  return func(opt *options) {
    opt.SpokenLang = spokenLangCode[lang]
  }
}
```

`spokenLangCode`是 GitHub 支持的国家和代码的对照，我是从 GitHub Trending 页面爬取的。大概是这样的：

```golang
var (
  spokenLangCode map[string]string
)

func init() {
  spokenLangCode = map[string]string{
    "abkhazian":             "ab",
    "afar":                  "aa",
    "afrikaans":             "af",
    "akan":                  "ak",
    "albanian":              "sq",
    // ...
  }
}
```

最后我希望 GitHub 的 URL 也可以设置：

```golang
func WithURL(url string) option {
  return func(opt *options) {
    opt.GitHubURL = url
  }
}
```

我们在`trending`结构中增加`options`字段，然后改造一下`New()`方法，让它接受可变参数的选项。这样我们只需要设置我们想要设置的，其他的选项都可以采用默认值，例如`GitHubURL`：

```golang
type trending struct {
  opts options
}

func loadOptions(opts ...option) options {
  o := options{
    GitHubURL: "http://github.com",
  }
  for _, option := range opts {
    option(&o)
  }

  return o
}

func New(opts ...option) Fetcher {
  return &trending{
    opts: loadOptions(opts...),
  }
}
```

最后在`FetchRepos()`方法和`FetchDevelopers()`方法中根据选项拼接 URL：

```golang
fmt.Sprintf("%s/trending/%s?spoken_language_code=%s&since=%s", t.opts.GitHubURL, t.opts.Language, t.opts.SpokenLang, t.opts.DateRange)

fmt.Sprintf("%s/trending/developers?lanugage=%s&since=%s", t.opts.GitHubURL, t.opts.Language, t.opts.DateRange)
```

加入选项之后，如果我们要获取一周内的，Go 语言 Trending 列表，可以这样：

```golang
t := ghtrending.New(ghtrending.WithWeekly(), ghtreading.WithLanguage("Go"))
repos, _ := t.FetchRepos()
```

### 简单方法

另外，我们还提供一个不需要创建`trending`对象，直接调用接口获取仓库和开发者列表的方法（懒人专用）：

```golang
func TrendingRepositories(opts ...option) ([]*Repository, error) {
  return New(opts...).FetchRepos()
}

func TrendingDevelopers(opts ...option) ([]*Developer, error) {
  return New(opts...).FetchDevelopers()
}
```

### 使用效果

新建目录并初始化 Go Modules：

```cmd
$ mkdir -p demo/ghtrending && cd demo/ghtrending
$ go mod init github/darjun/demo/ghtrending
```

下载包：

![](/img/in-post/opensource/trending9.png#center)

编写代码：

```golang
package main

import (
  "fmt"
  "log"

  "github.com/darjun/ghtrending"
)

func main() {
  t := ghtrending.New()

  repos, err := t.FetchRepos()
  if err != nil {
    log.Fatal(err)
  }

  fmt.Printf("%d repos\n", len(repos))
  fmt.Printf("first repo:%#v\n", repos[0])

  developers, err := t.FetchDevelopers()
  if err != nil {
    log.Fatal(err)
  }

  fmt.Printf("%d developers\n", len(developers))
  fmt.Printf("first developer:%#v\n", developers[0])
}
```

运行效果：

![](/img/in-post/opensource/trending10.png#center)

### 文档

最后，我们加点文档：

![](/img/in-post/opensource/trending8.png#center)

一个小开源库就完成了。 


## 总结

本文介绍如何使用`goquery`爬取网页。着重介绍了`ghtrending`的接口设计。在编写一个库的时候，应该提供易用的、最小化的接口。用户不需要了解库的实现细节就可以使用。`ghtrending`使用函数式选项就是一个例子，有需要才传递，无需要可不提供。

自己通过爬取网页的方式来获取 Trending 列表比较容易受限制，例如过段时间 GitHub 网页结构变了，代码就不得不做适配。在官方没有提供 API 的情况下，目前也只能这么做了。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. ghtrending GitHub：[github.com/darjun/ghtrending](https://github.com/darjun/ghtrending)
2. Go 每日一库之 goquery：[https://darjun.github.io/2020/10/11/godailylib/goquery](https://darjun.github.io/2020/10/11/godailylib/goquery)
3. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~