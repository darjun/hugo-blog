---
layout:    post
title:    "Go 每日一库之 email"
subtitle: 	"每天学习一个 Go 库"
date:		2020-02-16T07:56:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/02/16/godailylib/email"
categories: [
	"Go"
]
---

## 简介

程序中时常有发送邮件的需求。有异常情况了需要通知管理员和负责人，用户下单后可能需要通知订单信息，电商平台、中国移动和联通都有每月账单，这些都可以通过邮件来推送。还有我们平时收到的垃圾邮件大都也是通过这种方式发送的😭。那么如何在 Go 语言发送邮件？本文我们介绍一下[`email`](https://github.com/jordan-wright/email)库的使用。

## 快速使用

这个库的使用快不了，为什么呢？

先安装库，这个自不必说：

```cmd
$ go get github.com/jordan-wright/email
```

我们需要额外一些工作。我们知道邮箱使用`SMTP/POP3/IMAP`等协议从邮件服务器上拉取邮件。邮件并不是直接发送到邮箱的，而是邮箱请求拉取的。
所以，我们需要配置`SMTP/POP3/IMAP`服务器。从头搭建固然可行，而且也有现成的开源库，但是比较麻烦。现在一般的邮箱服务商都开放了`SMTP/POP3/IMAP`服务器。
我这里拿 126 邮箱来举例，使用`SMTP`服务器。当然，用 QQ 邮箱也可以。

* 首先，登录邮箱；
* 点开顶部的设置，选择`POP3/SMTP/IMAP`；
* 点击开启`IMAP/SMTP`服务，按照步骤开启即可，有个密码设置，记住这个密码，后面有用。

然后就可以编码了：

```golang
package main

import (
  "log"
  "net/smtp"

  "github.com/jordan-wright/email"
)

func main() {
  e := email.NewEmail()
  e.From = "dj <xxx@126.com>"
  e.To = []string{"935653229@qq.com"}
  e.Subject = "Awesome web"
  e.Text = []byte("Text Body is, of course, supported!")
  err := e.Send("smtp.126.com:25", smtp.PlainAuth("", "xxx@126.com", "yyy", "smtp.126.com"))
  if err != nil {
    log.Fatal(err)
  }
}
```

**这里为了我的信息安全，我把真实信息都隐藏了**。代码中`xxx`替换成你的邮箱账号，`yyy`替换成上面设置的密码。

代码步骤比较简单清晰：

* 先调用`NewEmail`创建一封邮件；
* 设置`From`发送方，`To`接收者，`Subject`邮件主题（标题），`Text`设置邮件内容；
* 然后调用`Send`发送，参数1是 SMTP 服务器的地址，参数2为验证信息。

运行程序将会向我的 QQ 邮箱发送一封邮件：

![](/img/in-post/godailylib/email1.png#center)

有的邮箱会把这种邮件放在垃圾箱中，例如 QQ😭。如果收件箱找不到，记得到垃圾箱瞅瞅。

## 抄送

平常我们发邮件的时候可能会抄送给一些人，还有一些人要**秘密抄送**😄，即 CC（Carbon Copy）和 BCC （Blind Carbon Copy）。
`email`我们也可以设置这两个参数：

```golang
package main

import (
  "log"
  "net/smtp"

  "github.com/jordan-wright/email"
)

func main() {
  e := email.NewEmail()
  e.From = "dj <xxx@126.com>"
  e.To = []string{"935653229@qq.com"}
  e.Cc = []string{"test1@126.com", "test2@126.com"}
  e.Bcc = []string{"secret@126.com"}
  e.Subject = "Awesome web"
  e.Text = []byte("Text Body is, of course, supported!")
  err := e.Send("smtp.126.com:25", smtp.PlainAuth("", "xxx@126.com", "yyy", "smtp.126.com"))
  if err != nil {
    log.Fatal(err)
  }
}
```

还是一样的，抄送的邮箱自己替换`test1/test2/secret`用自己的。

运行程序将会向我的 QQ 邮件发送一封邮件，同时抄送一封到我另一个 126 邮箱：

![](/img/in-post/godailylib/email2.png)

![](/img/in-post/godailylib/email3.png)

## HTML 格式

发送纯文本，邮件不太美观。`email`支持发送 HTML 格式的内容。与发送纯文本类似，直接设置对象的`HTML`字段：

```golang
package main

import (
  "log"
  "net/smtp"

  "github.com/jordan-wright/email"
)

func main() {
  e := email.NewEmail()
  e.From = "dj <xxx@126.com>"
  e.To = []string{"935653229@qq.com"}
  e.Cc = []string{"xxx@126.com"}
  e.Subject = "Go 每日一库"
  e.HTML = []byte(`
  <ul>
<li><a "https://darjun.github.io/2020/01/10/godailylib/flag/">Go 每日一库之 flag</a></li>
<li><a "https://darjun.github.io/2020/01/10/godailylib/go-flags/">Go 每日一库之 go-flags</a></li>
<li><a "https://darjun.github.io/2020/01/14/godailylib/go-homedir/">Go 每日一库之 go-homedir</a></li>
<li><a "https://darjun.github.io/2020/01/15/godailylib/go-ini/">Go 每日一库之 go-ini</a></li>
<li><a "https://darjun.github.io/2020/01/17/godailylib/cobra/">Go 每日一库之 cobra</a></li>
<li><a "https://darjun.github.io/2020/01/18/godailylib/viper/">Go 每日一库之 viper</a></li>
<li><a "https://darjun.github.io/2020/01/19/godailylib/fsnotify/">Go 每日一库之 fsnotify</a></li>
<li><a "https://darjun.github.io/2020/01/20/godailylib/cast/">Go 每日一库之 cast</a></li>
</ul>
  `)
  err := e.Send("smtp.126.com:25", smtp.PlainAuth("", "xxx@126.com", "yyy", "smtp.126.com"))
  if err != nil {
    log.Fatal("failed to send email:", err)
  }
}
```

发送结果：

![](/img/in-post/godailylib/email4.png#center)

**注意，126 的 SMTP 服务器检测比较严格，加上 HTML 之后，很容易被识别为垃圾邮件不让发送，这时 CC 自己就 OK 了。**

## 附件

添加附件也很容易，直接调用`AttachFile`即可：

```golang
package main

import (
  "log"
  "net/smtp"

  "github.com/jordan-wright/email"
)

func main() {
  e := email.NewEmail()
  e.From = "dj <xxx@126.com>"
  e.To = []string{"935653229@qq.com"}
  e.Subject = "Go 每日一库"
  e.Text = []byte("请看附件")
  e.AttachFile("test.txt")
  err := e.Send("smtp.126.com:25", smtp.PlainAuth("", "xxx@126.com", "yyy", "smtp.126.com"))
  if err != nil {
    log.Fatal("failed to send email:", err)
  }
}
```

收到的邮件：

![](/img/in-post/godailylib/email5.png#center)

## 连接池

实际上每次调用`Send`时都会和 SMTP 服务器建立一次连接，如果发送邮件很多很频繁的话可能会有性能问题。`email`提供了连接池，可以复用网络连接：

```golang
package main

import (
  "fmt"
  "log"
  "net/smtp"
  "os"
  "sync"
  "time"

  "github.com/jordan-wright/email"
)

func main() {
  ch := make(chan *email.Email, 10)
  p, err := email.NewPool(
    "smtp.126.com:25",
    4,
    smtp.PlainAuth("", "leedarjun@126.com", "358942617ldj", "smtp.126.com"),
  )

  if err != nil {
    log.Fatal("failed to create pool:", err)
  }

  var wg sync.WaitGroup
  wg.Add(4)
  for i := 0; i < 4; i++ {
    go func() {
      defer wg.Done()
      for e := range ch {
        err := p.Send(e, 10*time.Second)
        if err != nil {
          fmt.Fprintf(os.Stderr, "email:%v sent error:%v\n", e, err)
        }
      }
    }()
  }

  for i := 0; i < 10; i++ {
    e := email.NewEmail()
    e.From = "dj <leedarjun@126.com>"
    e.To = []string{"935653229@qq.com"}
    e.Subject = "Awesome web"
    e.Text = []byte(fmt.Sprintf("Awesome Web %d", i+1))
    ch <- e
  }

  close(ch)
  wg.Wait()
}
```

上面程序中，我们创建 4 goroutine 共用一个连接池发送邮件，发送 10 封邮件后程序退出。为了等邮件都发送完成或失败，程序才退出，我们使用了`sync.WaitGroup`。

邮箱被轰炸了：

![](/img/in-post/godailylib/email6.png#center)

由于使用了 goroutine，邮件顺序不能保证。

## 总结

本文介绍了如何使用 Go 程序发送邮件，程序代码都已经放在 GitHub 上[https://github.com/darjun/go-daily-lib/tree/master/email](https://github.com/darjun/go-daily-lib/tree/master/email)。所有代码都通过测试，大家请放心食用~

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. email GitHub：[https://github.com/jordan-wright/email](https://github.com/jordan-wright/email)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)