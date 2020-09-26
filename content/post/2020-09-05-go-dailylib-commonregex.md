---
layout:    post
title:    "Go 每日一库之 commonregex"
subtitle: 	"每天学习一个 Go 库"
date:		2020-09-06T11:38:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/09/05/godailylib/commonregex"
categories: [
	"Go"
]
---

## 简介

有时，我们会遇到一些需要使用字符串的匹配和查找的任务。并且我们知道这种情况下，使用正则表达式是最简洁和优雅的。为了完成某个任务特地去系统地学习正则表达式费时费力，而且一段时间不用又很容易遗忘。下次遇到问题还要再重复这个过程。[`commonregex`](https://github.com/mingrammer/commonregex)库来了，它内置很多常用的正则表达式，开箱即用。当然，我并不是说没必要去学习正则表达式，熟练掌握正则表达式需要时间和练习，对于时长和文本处理打交道的开发人员，正则表达式决定是提升工作效率的一把利器。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir commonregex && cd commonregex
$ go mod init github.com/darjun/go-daily-lib/commonregex
```

安装`commonregex`库：

```cmd
$ go get -u github.com/mingrammer/commonregex
```

简单使用：

```golang
package main

import (
  "fmt"

  cregex "github.com/mingrammer/commonregex"
)

func main() {
  text := `John, please get that article on www.linkedin.com to me by 5:00PM on Jan 9th 2012. 4:00 would be ideal, actually. If you have any questions, You can reach me at (519)-236-2723x341 or get in touch with my associate at harold.smith@gmail.com`

  dateList := cregex.Date(text)
  timeList := cregex.Time(text)
  linkList := cregex.Links(text)
  phoneList := cregex.PhonesWithExts(text)
  emailList := cregex.Emails(text)

  fmt.Println("date list:", dateList)
  fmt.Println("time list:", timeList)
  fmt.Println("link list:", linkList)
  fmt.Println("phone list:", phoneList)
  fmt.Println("email list:", emailList)
}
```

运行结果：

```cmd
$ go run main.go
date list: [Jan 9th 2012]
time list: [5:00PM 4:00 ]
link list: [www.linkedin.com harold.smith@gmail.com]
phone list: [(519)-236-2723x341]
email list: [harold.smith@gmail.com]
```

`commonregex`提供的 API 非常易于使用，调用相应的类别方法返回一段文本中符合这些格式的字符串列表。上面依次从`text`获取**日期列表**，**时间列表**，**超链接列表**，**电话号码列表**和**电子邮件列表**。

## 内置的正则

`commonregex`支持很多常用的正则表达式：

* 日期；
* 时间；
* 电话号码；
* 超链接；
* 邮件地址；
* IPv4/IPv6/IP 地址；
* 价格；
* 十六进制颜色值；
* 信用卡卡号；
* 10/13 位 ISBN；
* 邮政编码；
* MD5；
* SHA1；
* SHA256；
* GUID，全局唯一标识；
* Git 仓库地址。

每种类型又支持多种格式，例如日期支持`09.11.2020`/`Sep 11th 2020`。

下面挑选几种类型来介绍。

### 日期

```golang
func main() {
  text := `commonregex support many date formats, like 09.11.2020, Sep 11th 2020 and so on.`
  dateList := commonregex.Date(text)

  fmt.Println(dateList)
}
```

匹配出来的日期（注意 Go 中 slice 的输出）：

```cmd
[09.11.2020 Sep 11th 2020]
```

### 时间

时间相对来说格式单一一些，有 24 小时制的时间如：`08:30`/`14:35`，有 12 小时制的时间：`08:30am`/`02:35pm`。

看示例：

```golang
func main() {
  text := `I wake up at 08:30 (aka 08:30am) in the morning, take a snap at 13:00(aka 01:00pm).`
  timeList := commonregex.Time(text)

  fmt.Println(timeList)
}
```

匹配出来的时间列表：

```cmd
[08:30  08:30am 13:00 01:00pm]
```

### IP/MAC/MD5

使用方法都是类似的，这几个放在一起举例。

IPv4 地址是 4 个以`.`分隔的数字，每个数字都在[0-255]范围内。

MAC 是计算机的物理地址（又叫以太网地址，局域网地址等），是 6 组以`:`分隔的十六进制数字，每组两个。

MD5 是一种哈希算法，将一段数据转为长度为 32 的字符串。

```golang
func main() {
  text := `mac address: ac:de:48:00:11:22, ip: 192.168.3.20, md5: fdbf72fdabb67ea6ef7ff5155a44def4`

  macList := commonregex.MACAddresses(text)
  ipList := commonregex.IPs(text)
  md5List := commonregex.MD5Hexes(text)

  fmt.Println("mac list:", macList)
  fmt.Println("ip list:", ipList)
  fmt.Println("md5 list:", md5List)
}
```

输出：

```cmd
mac list: [ac:de:48:00:11:22]
ip list: [192.168.3.20]
md5 list: [fdbf72fdabb67ea6ef7ff5155a44def4]
```

## 总结

`commonregex`足够我们去应付一般的使用场景了。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. commonregex GitHub：[https://github.com/mingrammer/commonregex](https://github.com/mingrammer/commonregex)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)