---
layout:    post
title:    "Go 每日一库之 dateparse"
subtitle: "每天学习一个 Go 库"
date:     2021-06-24T06:37:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/06/24/godailylib/dateparse"
categories: [
  "Go"
]
---

## 简介

不管什么时候，处理时间总是让人头疼的一件事情。因为时间格式太多样化了，再加上时区，夏令时，闰秒这些细枝末节处理起来更是困难。所以在程序中，涉及时间的处理我们一般借助于标准库或第三方提供的时间库。今天要介绍的[`dateparse`](https://github.com/araddon/dateparse)专注于一个很小的时间处理领域——解析日期时间格式的字符串。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir dateparse && cd dateparse
$ go mod init github.com/darjun/go-daily-lib/dateparse
```

安装`dateparse`库：

```cmd
$ go get -u github.com/araddon/dateparse
```

使用：

```golang
package main

import (
  "fmt"
  "log"
  "github.com/araddon/dateparse"
)

func main() {
  t1, err := dateparse.ParseAny("3/1/2014")
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println(t1.Format("2006-01-02 15:04:05"))

  t2, err := dateparse.ParseAny("mm/dd/yyyy")
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println(t2.Format("2006-01-02 15:04:05"))
}
```

`ParseAny()`方法接受一个日期时间字符串，解析该字符串，返回`time.Time`类型的值。如果传入的字符串`dateparse`库无法识别，则返回一个错误。上面程序运行输出：

```golang
$ go run main.go
2014-03-01 00:00:00
2021/06/24 14:52:39 Could not find format for "mm/dd/yyyy"
exit status 1
```

需要注意，当我们写出"3/1/2014"这个时间的时候，可以解释为**2014年3月1日**，也可以解释为**2014年1月3日**。这就存在二义性，`dateparse`默认采用`mm/dd/yyyy`这种格式，也就是**2014年3月1日**。我们也可以使用`ParseStrict()`函数让这种具有二义性的字符串解析失败：

```golang
func main() {
  t, err := dateparse.ParseStrict("3/1/2014")
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println(t.Format("2006-01-02 15:04:05"))
}
```

运行：

```golang
$ go run main.go
2021/06/24 14:57:18 This date has ambiguous mm/dd vs dd/mm type format
exit status 1
```

## 格式

`dateparse`支持丰富的日期时间格式，基本囊括了所有常用的格式。它支持标准库`time`中预定义的所有格式：

```golang
// src/time/format.go
const (
  ANSIC       = "Mon Jan _2 15:04:05 2006"
  UnixDate    = "Mon Jan _2 15:04:05 MST 2006"
  RubyDate    = "Mon Jan 02 15:04:05 -0700 2006"
  RFC822      = "02 Jan 06 15:04 MST"
  RFC822Z     = "02 Jan 06 15:04 -0700" // RFC822 with numeric zone
  RFC850      = "Monday, 02-Jan-06 15:04:05 MST"
  RFC1123     = "Mon, 02 Jan 2006 15:04:05 MST"
  RFC1123Z    = "Mon, 02 Jan 2006 15:04:05 -0700" // RFC1123 with numeric zone
  RFC3339     = "2006-01-02T15:04:05Z07:00"
  RFC3339Nano = "2006-01-02T15:04:05.999999999Z07:00"
  Kitchen     = "3:04PM"
  // Handy time stamps.
  Stamp      = "Jan _2 15:04:05"
  StampMilli = "Jan _2 15:04:05.000"
  StampMicro = "Jan _2 15:04:05.000000"
  StampNano  = "Jan _2 15:04:05.000000000"
)
```

支持的完整格式查看[dateparse README](https://github.com/araddon/dateparse)。

## 时区

`dateparse`支持在特定时区解析日期时间字符串。我们可以通过调用标准库的`time.LoadLocation()`方法，传入时区标识字符串来获得时区对象。时区标识字符串是类似`Asia/Shanghai`，`America/Chicago`这样的格式，它表示一个具体的时区，前者上海，后者洛杉矶。调用`dateparse.ParseIn()`方法传入时区对象，在指定时区中解析。`time`包中还预定义了两个时区对象，`time.Local`表示本地时区，`time.UTC`表示 UTC 时区。时区的权威数据请看[IANA](https://www.iana.org/time-zones)。

```golang
func main() {
  tz1, _ := time.LoadLocation("America/Chicago")
  t1, _ := dateparse.ParseIn("2021-06-24 15:50:30", tz1)
  fmt.Println(t1.Local().Format("2006-01-02 15:04:05"))

  t2, _ := dateparse.ParseIn("2021-06-24 15:50:30", time.Local)
  fmt.Println(t2.Local().Format("2006-01-02 15:04:05"))
}
```

运行：

```golang
$ go run main.go
2021-06-25 04:50:30
2021-06-24 15:50:30
```

美国洛杉矶时区的"2021年6月24日 15时30分30秒"等于本地时区（北京时间）的"2021年6月25日 04时50分30秒"。

## cli

`dateparse`还提供了一个命令行工具，用于极快地查看日期时间格式。安装：

```golang
$ go install github.com/araddon/dateparse/dateparse
```

默认会安装在`$GOPATH`路径下，我习惯上把`$GOPATH/bin`放到`$PATH`中。所以`dateparse`命令可以直接使用。

`dateparse`命令接收一个字符串，和一个可选的时区选项：

```golang
$ dateparse --timezone="Asia/Shanghai" "2021-06-24 06:46:08"

Your Current time.Local zone is CST

Layout String: dateparse.ParseFormat() => 2006-01-02 15:04:05

Your Using time.Local set to location=Asia/Shanghai CST

+-------------+---------------------------+-------------------------------+-------------------------------------+
| method      | Zone Source               | Parsed                        | Parsed: t.In(time.UTC)              |
+-------------+---------------------------+-------------------------------+-------------------------------------+
| ParseAny    | time.Local = nil          | 2021-06-24 06:46:08 +0000 UTC | 2021-06-24 06:46:08 +0000 UTC day=4 |
| ParseAny    | time.Local = timezone arg | 2021-06-24 06:46:08 +0000 UTC | 2021-06-24 06:46:08 +0000 UTC day=4 |
| ParseAny    | time.Local = time.UTC     | 2021-06-24 06:46:08 +0000 UTC | 2021-06-24 06:46:08 +0000 UTC day=4 |
| ParseIn     | time.Local = nil          | 2021-06-24 06:46:08 +0000 UTC | 2021-06-24 06:46:08 +0000 UTC       |
| ParseIn     | time.Local = timezone arg | 2021-06-24 06:46:08 +0800 CST | 2021-06-23 22:46:08 +0000 UTC       |
| ParseIn     | time.Local = time.UTC     | 2021-06-24 06:46:08 +0000 UTC | 2021-06-24 06:46:08 +0000 UTC       |
| ParseLocal  | time.Local = nil          | 2021-06-24 06:46:08 +0000 UTC | 2021-06-24 06:46:08 +0000 UTC       |
| ParseLocal  | time.Local = timezone arg | 2021-06-24 06:46:08 +0800 CST | 2021-06-23 22:46:08 +0000 UTC       |
| ParseLocal  | time.Local = time.UTC     | 2021-06-24 06:46:08 +0000 UTC | 2021-06-24 06:46:08 +0000 UTC       |
| ParseStrict | time.Local = nil          | 2021-06-24 06:46:08 +0000 UTC | 2021-06-24 06:46:08 +0000 UTC       |
| ParseStrict | time.Local = timezone arg | 2021-06-24 06:46:08 +0000 UTC | 2021-06-24 06:46:08 +0000 UTC       |
| ParseStrict | time.Local = time.UTC     | 2021-06-24 06:46:08 +0000 UTC | 2021-06-24 06:46:08 +0000 UTC       |
+-------------+---------------------------+-------------------------------+-------------------------------------+
```

输出当前本地时区，格式字符串（可用于生成同样格式的日期时间字符串）和一个表格。表格里面的数据是分别对`ParseAny/ParseIn/ParseLocal/ParseStrict`在不同的时区下调用的结果。

`method`列表示调用的方法，`Zone Source`列表示将本地时区设置的值，`Parsed`列是以日期时间字符串调用`ParseAny()`返回的`time.Time`对象的`Format()`方法调用结果，`Parsed: t.In(time.UTC)`列在返回的`time.Time`对象调用`Format()`方法前将其转为 UTC 时间。

由于`ParseAny/ParseStrict`不会考虑本地时区，都是在 UTC 下解析字符串，所以这 6 行的最后两列结果都一样。

`ParseIn`的第二行，将`time.Local`设置为我们通过命令行选项设置的时区，上面我设置为`Asia/Shanghai`，对应的 UTC 时间相差 8 小时。`ParseLocal`也是如此。

下面是`dateparse`命令行的部分源码，可以对照查看：

```golang
func main() {
  parsers := map[string]parser{
    "ParseAny":    parseAny,
    "ParseIn":     parseIn,
    "ParseLocal":  parseLocal,
    "ParseStrict": parseStrict,
  }

  for name, parser := range parsers {
    time.Local = nil
    table.AddRow(name, "time.Local = nil", parser(datestr, nil, false), parser(datestr, nil, true))
    if timezone != "" {
      time.Local = loc
      table.AddRow(name, "time.Local = timezone arg", parser(datestr, loc, false), parser(datestr, loc, true))
    }
    time.Local = time.UTC
    table.AddRow(name, "time.Local = time.UTC", parser(datestr, time.UTC, false), parser(datestr, time.UTC, true))
  }
}

func parseIn(datestr string, loc *time.Location, utc bool) string {
  t, err := dateparse.ParseIn(datestr, loc)
  if err != nil {
    return err.Error()
  }
  if utc {
    return t.In(time.UTC).String()
  }
  return t.String()
}
```

注意输出的本地时区为 CST，它可以代表不同的时区：

```golang
Central Standard Time (USA) UT-6:00
Central Standard Time (Australia) UT+9:30
China Standard Time UT+8:00
Cuba Standard Time UT-4:00
```

CST 可以同时表示美国、澳大利亚、中国和古巴四个国家的标准时间。

## 总结

使用`dateparse`可以很方便地从日期时间字符串中解析出时间对象和格式（layout）。同时`dateparse`命令行可以快速的查看和转换相应时区的时间，是一个非常不错的小工具。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. dateparse GitHub：[github.com/araddon/dateparse](https://github.com/araddon/dateparse)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~