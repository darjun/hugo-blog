---
layout:    post
title:    "Go 每日一库之 jobrunner"
subtitle: 	"每天学习一个 Go 库"
date:		2020-09-01T20:07:09
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/09/01/godailylib/jobrunner"
categories: [
	"Go"
]
---

## 简介

我们在 Web 开发中时常会遇到这样的需求，执行一个操作之后，需要给用户一定形式的通知。例如，用户下单之后通过邮件发送电子发票，网上购票支付后通过短信发送车次信息。但是这类需求并不需要非常及时，如果放在请求流程中处理，会影响请求的响应时间。这类任务我们一般使用异步的方式来执行。[`jobrunner`](https://github.com/bamzi/jobrunner)就是其中一个用来执行异步任务的 Go 语言库。得益于强大的[`cron`](https://github.com/robfig/cron)库，再搭配`jobrunner`的任务状态监控，`jobrunner`非常易于使用。

## 快速使用

本文使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir jobrunner && cd jobrunner
$ go mod init github.com/darjun/go-daily-lib/jobrunner
```

安装`jobrunner`：

```cmd
$ go get -u github.com/bamzi/jobrunner
```

使用：

```golang
package main

import (
  "fmt"
  "time"

  "github.com/bamzi/jobrunner"
)

type GreetingJob struct {
  Name string
}

func (g GreetingJob) Run() {
  fmt.Println("Hello, ", g.Name)
}

func main() {
  jobrunner.Start()
  jobrunner.Schedule("@every 5s", GreetingJob{Name: "dj"})

  time.Sleep(10 * time.Second)
}
```

我们创建一个任务，每隔 5s 打印一条欢迎信息。任务的创建和执行与`cron`完全相同，详细使用见我前面的[一篇博文](https://darjun.github.io/2020/06/25/godailylib/cron/)。

注意，`jobrunner`需要先`Start()`，然后再添加任务。因为在`Start()`中创建`MainCron`对象，先添加任务会`panic`！！！

注意`main`函数尾的`time.Sleep(10 * time.Second)`，因为主 goroutine 结束之后整个程序就退出了，`jobrunner`中的任务就没有机会被执行了。加上`time.Sleep`是为了让大家能看到输出，实际使用中不会这样做。

## 与 web 框架整合

`jobrunner`能很方便地与当前常见的 Web 框架整合，如`Gin/Echo/Martini/Beego/Revel`等。下面通过一个简单的例子演示如何在 Gin 中使用`jobrunner`：用户登录时给他的邮箱发送一封邮件。

首先需要安装相应的库：

```cmd
$ go get -u github.com/gin-gonic/gin
$ github.com/jordan-wright/email
```

编写代码：

```golang
package main

import (
  "fmt"
  "net/smtp"
  "time"

  "github.com/bamzi/jobrunner"
  "github.com/gin-gonic/gin"
  "github.com/jordan-wright/email"
)

type EmailJob struct {
  Name  string
  Email string
}

type User struct {
  Name  string `form:"name"`
  Email string `form:"email"`
}

func (j EmailJob) Run() {
  e := email.NewEmail()
  e.From = "leedarjun@126.com"
  e.To = []string{j.Email}
  e.Cc = []string{"leedarjun@126.com"}
  e.Subject = "Welcome To Awesome-Web"
  e.Text = []byte(fmt.Sprintf(`
  Hello, %s
  Welcome Back
  `, j.Name))

  err := e.Send("smtp.126.com:25", smtp.PlainAuth("", "leedarjun@126.com", "yyyyyy", "smtp.126.com"))
  if err != nil {
    fmt.Printf("failed to send email to %s, err:%v", j.Name, err)
  }
}

func login(c *gin.Context) {
  var u User
  if c.ShouldBind(&u) == nil {
    c.String(200, "login success")

    jobrunner.In(5*time.Second, EmailJob{Name: u.Name, Email: u.Email})
  } else {
    c.String(404, "login failed")
  }
}

func main() {
  r := gin.Default()
  r.GET("/login", login)
  r.Run(":8888")
}
```

这里只是为了简单演示，我们编写了一个简陋的`login`函数处理登录，传入`name`和`email`，然后给该`email`发送邮件。`email`库的详细使用可以查看我之前的[博文](https://darjun.github.io/2020/02/16/godailylib/email)了解。

只需要在浏览器中输入`http://localhost:8888/login?name=dj&email=935653229@qq.com`，我的 QQ 邮箱就能收到邮件：

![](/img/in-post/godailylib/jobrunner1.png#center)

## 监控

`jobrunner`内置了一个监控模块，可以很方便地通过网页或者 API 获取当前的任务状态数据：

```golang
package main

import (
  "fmt"
  "html/template"
  "os"
  "time"

  "github.com/bamzi/jobrunner"
  "github.com/gin-gonic/gin"
)

type GreetingJob struct {
  Name string
}

func (g GreetingJob) Run() {
  fmt.Println("Hello,", g.Name)
}

type EmailJob struct {
  Email string
}

func (e EmailJob) Run() {
  fmt.Println("Send,", e.Email)
}

func main() {
  r := gin.Default()

  jobrunner.Start()
  jobrunner.Every(5*time.Second, GreetingJob{Name: "dj"})
  jobrunner.Every(10*time.Second, EmailJob{Email: "935653229@qq.com"})

  r.GET("/jobrunner/json", JobJson)
  r.GET("/jobrunner/html", JobHtml)

  r.Run(":8888")
}

func JobJson(c *gin.Context) {
  c.JSON(200, jobrunner.StatusJson())
}

func JobHtml(c *gin.Context) {
  t, err := template.ParseFiles(os.Getenv("GOPATH") + "/src/github.com/bamzi/jobrunner/views/Status.html")
  if err != nil {
    c.JSON(400, "error")
  }
  t.Execute(c.Writer, jobrunner.StatusPage())
}
```

运行之后，在浏览器中输入`http://localhost:8888/jobrunner/html`查看任务状态：

![](/img/in-post/godailylib/jobrunner2.png#center)

这里显示任务名、任务 ID、状态、上次运行时间、下次运行时间以及处理延迟。

我们还可以通过`http://localhost:8888/jobrunner/json`获取原始 JSON 格式的数据自己处理：

![](/img/in-post/godailylib/jobrunner3.png#center)

## 总结

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. jobrunner GitHub：[https://github.com/bamzi/jobrunner](https://github.com/bamzi/jobrunner)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)