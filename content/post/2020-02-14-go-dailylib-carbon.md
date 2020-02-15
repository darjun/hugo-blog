---
layout:    post
title:    "Go 每日一库之 carbon"
subtitle: 	"每天学习一个 Go 库"
date:		2020-02-14T21:12:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/02/14/godailylib/carbon"
categories: [
	"Go"
]
---

## 简介

一线开发人员每天都要使用日期和时间相关的功能，各种定时器，活动时间处理等。标准库[time](https://golang.org/pkg/time)使用起来不太灵活，特别是日期时间的创建和运算。[carbon](https://github.com/uniplaces/carbon)库是一个时间扩展库，基于 PHP 的[carbon](https://github.com/briannesbitt/carbon)库编写。提供易于使用的接口。本文就来介绍一下这个库。

## 快速使用

第三方库需要先安装：

```
$ go get github.com/uniplaces/carbon
```

后使用：

```golang
package main

import (
  "fmt"
  "time"

  "github.com/uniplaces/carbon"
)

func main() {
  fmt.Printf("Right now is %s\n", carbon.Now().DateTimeString())

  today, _ := carbon.NowInLocation("Japan")
  fmt.Printf("Right now in Japan is %s\n", today)

  fmt.Printf("Tomorrow is %s\n", carbon.Now().AddDay())
  fmt.Printf("Last week is %s\n", carbon.Now().SubWeek())

  nextOlympics, _ := carbon.CreateFromDate(2016, time.August, 5, "Europe/London")
  nextOlympics = nextOlympics.AddYears(4)
  fmt.Printf("Next olympics are in %d\n", nextOlympics.Year())

  if carbon.Now().IsWeekend() {
    fmt.Printf("Happy time!")
  }
}
```

`carbon`库的使用很便捷，首先它完全兼容标准库的`time.Time`类型，实际上该库的日期时间类型`Carbon`直接将`time.Time`内嵌到结构中，所以`time.Time`的方法可直接调用：

```golang
// src/github.com/uniplaces/carbon/carbon.go
type Carbon struct {
  time.Time
  weekStartsAt time.Weekday
  weekEndsAt   time.Weekday
  weekendDays  []time.Weekday
  stringFormat string
  Translator   *Translator
}
```

其次，简化了创建操作。标准库`time`创建一个`Time`对象，如果不是本地或 UTC 时区，需要自己先调用`LoadLocation`加载对应时区。然后将该时区对象传给`time.Date`方法创建。`carbon`可以直接传时区名字。

`carbon`还提供了很多方法做日期运算，如例子中的`AddDay`，`SubWeek`等，都是见名知义的。

## 时区

在介绍其它内容之前，我们先说一说这个时区的问题。以下引用维基百科的描述：

> 时区是地球上的区域使用同一个时间定义。以前，人们通过观察太阳的位置（时角）决定时间，这就使得不同经度的地方的时间有所不同（地方时）。1863年，首次使用时区的概念。时区通过设立一个区域的标准时间部分地解决了这个问题。
世界各国位于地球不同位置上，因此不同国家，特别是东西跨度大的国家日出、日落时间必定有所偏差。这些偏差就是所谓的时差。

例如，日本东京位于东九区，北京位于东八区，所以日本比中国快一个小时，日本14:00的时候中国13:00。

在 Linux 中，时区一般存放在类似`/usr/share/zoneinfo`这样的目录。这个目录中有很多文件，每个时区一个文件。时区文件是二进制文件，可以执行`info tzfile`查看具体格式。

时区名称的一般格式为`city`，或`country/city`，或`continent/city`。即要么就是一个城市名，要么是国家名+/+城市名，要么是洲名+/+城市名。例如上海时区为`Asia/Shanghai`，香港时区为`Asia/Hong_Kong`。也有一些特殊的，如 UTC，Local等。

Go 语言为了可移植性，在安装包中提供了时区文件，在安装目录下（我的为`C:\Go`）的`lib/time/zoneinfo.zip`文件，大家可以执行解压看看😀。

使用 Go 标准库`time`创建某个时区的时间，需要先加载时区：

```golang
package main

import (
  "fmt"
  "log"
  "time"
)

func main() {
  loc, err := time.LoadLocation("Japan")
  if err != nil {
    log.Fatal("failed to load location: ", err)
  }

  d := time.Date(2020, time.July, 24, 20, 0, 0, 0, loc)
  fmt.Printf("The opening ceremony of next olympics will start at %s in Japan\n", d)
}
```

使用`carbon`就不用这么麻烦：

```golang
package main

import (
  "fmt"
  "log"
  "time"

  "github.com/uniplaces/carbon"
)

func main() {
  c, err := carbon.Create(2020, time.July, 24, 20, 0, 0, 0, "Japan")
  if err != nil {
    log.Fatal(err)
  }

  fmt.Printf("The opening ceremony of next olympics will start at %s in Japan\n", c)
}
```

## 时间运算

使用标准库`time`的时间运算需要先定义一个`time.Duration`对象，`time`库预定义的只有纳秒到小时的精度：

```golang
const (
  Nanosecond  Duration = 1
  Microsecond = 1000 * Nanosecond
  Millisecond = 1000 * Microsecond
  Second      = 1000 * Millisecond
  Minute      = 60 * Second
  Hour        = 60 * Minute
)
```

其它的时长就需要自己使用`time.ParseDuration`构造了，而且`time.ParseDuration`不能构造其它精度的时间。
如果想要增加/减少年月日，就需要使用`time.Time`的`AddDate`方法：

```golang
package main

import (
  "fmt"
  "log"
  "time"
)

func main() {
  now := time.Now()

  fmt.Println("now is:", now)

  fmt.Println("one second later is:", now.Add(time.Second))
  fmt.Println("one minute later is:", now.Add(time.Minute))
  fmt.Println("one hour later is:", now.Add(time.Hour))

  d, err := time.ParseDuration("3m20s")
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println("3 minutes and 20 seconds later is:", now.Add(d))

  d, err = time.ParseDuration("2h30m")
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println("2 hours and 30 minutes later is:", now.Add(d))

  fmt.Println("3 days and 2 hours later is:", now.AddDate(0, 0, 3).Add(time.Hour*2))
}
```

需要注意的是，时间操作都是返回一个新的对象，原对象不会修改。`carbon`库也是如此。**Go 的标准库也建议我们不要使用`time.Time`的指针**。
当然`carbon`库也能使用上面的方法，它还提供了多种粒度的方法：

```golang
package main

import (
  "fmt"

  "github.com/uniplaces/carbon"
)

func main() {
  now := carbon.Now()

  fmt.Println("now is:", now)

  fmt.Println("one second later is:", now.AddSecond())
  fmt.Println("one minute later is:", now.AddMinute())
  fmt.Println("one hour later is:", now.AddHour())
  fmt.Println("3 minutes and 20 seconds later is:", now.AddMinutes(3).AddSeconds(20))
  fmt.Println("2 hours and 30 minutes later is:", now.AddHours(2).AddMinutes(30))
  fmt.Println("3 days and 2 hours later is:", now.AddDays(3).AddHours(2))
}
```

`carbon`还提供了：

* 增加**季度**的方法：`AddQuarters/AddQuarter`，复数形式介绍一个表示倍数的参数，单数形式倍数为1；
* 增加**世纪**的方法：`AddCenturies/AddCentury`；
* 增加**工作日**的方法：`AddWeekdays/AddWeekday`，这个方法会跳过非工作日；
* 增加**周**的方法：`AddWeeks/AddWeek`。

其实给上面方法传入负数就表示减少，另外`carbon`也提供了对应的`Sub*`方法。

## 时间比较

标准库`time`可以使用`time.Time`对象的`Before/After/Equal`判断是否在另一个时间对象前，后，或相等。`carbon`库也可以使用上面的方法比较时间。除此之外，它还提供了多组方法，每个方法提供一个简短名，一个详细名：

* `Eq/EqualTo`：是否相等；
* `Ne/NotEqualTo`：是否不等；
* `Gt/GreaterThan`：是否在之后；
* `Lt/LessThan`：是否在之前；
* `Lte/LessThanOrEqual`：是否相同或在之前；
* `Between`：是否在两个时间之间。

另外`carbon`提供了：

* 判断当前时间是周几的方法：`IsMonday/IsTuesday/.../IsSunday`；
* 是否是工作日，周末，闰年，过去时间还是未来时间：`IsWeekday/IsWeekend/IsLeapYear/IsPast/IsFuture`。

```golang
package main

import (
  "fmt"

  "github.com/uniplaces/carbon"
)

func main() {
  t1, _ := carbon.CreateFromDate(2010, 10, 1, "Asia/Shanghai")
  t2, _ := carbon.CreateFromDate(2011, 10, 20, "Asia/Shanghai")

  fmt.Printf("t1 equal to t2: %t\n", t1.Eq(t2))
  fmt.Printf("t1 not equal to t2: %t\n", t1.Ne(t2))

  fmt.Printf("t1 greater than t2: %t\n", t1.Gt(t2))
  fmt.Printf("t1 less than t2: %t\n", t1.Lt(t2))

  t3, _ := carbon.CreateFromDate(2011, 1, 20, "Asia/Shanghai")
  fmt.Printf("t3 between t1 and t2: %t\n", t3.Between(t1, t2, true))

  now := carbon.Now()
  fmt.Printf("Weekday? %t\n", now.IsWeekday())
  fmt.Printf("Weekend? %t\n", now.IsWeekend())
  fmt.Printf("LeapYear? %t\n", now.IsLeapYear())
  fmt.Printf("Past? %t\n", now.IsPast())
  fmt.Printf("Future? %t\n", now.IsFuture())
}
```

我们还可以使用`carbon`计算两个日期之间相差多少秒、分、小时、天：

```golang
package main

import (
  "fmt"

  "github.com/uniplaces/carbon"
)

func main() {
  vancouver, _ := carbon.Today("Asia/Shanghai")
  london, _ := carbon.Today("Asia/Hong_Kong")
  fmt.Println(vancouver.DiffInSeconds(london, true)) // 0

  ottawa, _ := carbon.CreateFromDate(2000, 1, 1, "America/Toronto")
  vancouver, _ = carbon.CreateFromDate(2000, 1, 1, "America/Vancouver")
  fmt.Println(ottawa.DiffInHours(vancouver, true)) // 3

  fmt.Println(ottawa.DiffInHours(vancouver, false)) // 3
  fmt.Println(vancouver.DiffInHours(ottawa, false)) // -3

  t, _ := carbon.CreateFromDate(2012, 1, 31, "UTC")
  fmt.Println(t.DiffInDays(t.AddMonth(), true))  // 31
  fmt.Println(t.DiffInDays(t.SubMonth(), false)) // -31

  t, _ = carbon.CreateFromDate(2012, 4, 30, "UTC")
  fmt.Println(t.DiffInDays(t.AddMonth(), true)) // 30
  fmt.Println(t.DiffInDays(t.AddWeek(), true))  // 7

  t, _ = carbon.CreateFromTime(10, 1, 1, 0, "UTC")
  fmt.Println(t.DiffInMinutes(t.AddSeconds(59), true))  // 0
  fmt.Println(t.DiffInMinutes(t.AddSeconds(60), true))  // 1
  fmt.Println(t.DiffInMinutes(t.AddSeconds(119), true)) // 1
  fmt.Println(t.DiffInMinutes(t.AddSeconds(120), true)) // 2
}
```

## 格式化

我们知道`time.Time`提供了一个`Format`方法，相比于其他编程语言使用格式化符来描述格式（需要记忆`%d/%m/%h`等的含义），Go 提供了一个一种更简单、直观的方式——使用 layout。即我们传入一个日期字符串，表示我们想要格式化成什么样子。Go 会用当前的时间替换字符串中的对应部分：

```golang
package main

import (
  "fmt"
  "time"
)

func main() {
  t := time.Now()
  fmt.Println(t.Format("2006-01-02 10:00:00"))
}
```

上面我们只需要传入一个`2006-01-02 10:00:00`表示我们想要的格式为`yyyy-mm-dd hh:mm:ss`，省去了我们需要记忆的麻烦。

为了使用方便，Go 内置了一些标准的时间格式：

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

除了上面这些格式，`carbon`还提供了其他一些格式：

```golang
// src/github.com/uniplaces/carbon
const (
  DefaultFormat       = "2006-01-02 15:04:05"
  DateFormat          = "2006-01-02"
  FormattedDateFormat = "Jan 2, 2006"
  TimeFormat          = "15:04:05"
  HourMinuteFormat    = "15:04"
  HourFormat          = "15"
  DayDateTimeFormat   = "Mon, Aug 2, 2006 3:04 PM"
  CookieFormat        = "Monday, 02-Jan-2006 15:04:05 MST"
  RFC822Format        = "Mon, 02 Jan 06 15:04:05 -0700"
  RFC1036Format       = "Mon, 02 Jan 06 15:04:05 -0700"
  RFC2822Format       = "Mon, 02 Jan 2006 15:04:05 -0700"
  RSSFormat           = "Mon, 02 Jan 2006 15:04:05 -0700"
)
```

注意一点，`time`库默认使用`2006-01-02 15:04:05.999999999 -0700 MST`格式，有点复杂了，`carbon`库默认使用更简洁的`2006-01-02 15:04:05`。

## 高级特性

### 修饰器

所谓修饰器（modifier）就是对一些特定的时间操作，获取开始和结束时间。如当天、月、季度、年、十年、世纪、周的开始和结束时间，还能获得上一个周二、下一个周一、下一个工作日的时间等等：

```golang
package main

import (
  "fmt"
  "time"

  "github.com/uniplaces/carbon"
)

func main() {
  t := carbon.Now()
  fmt.Printf("Start of day:%s\n", t.StartOfDay())
  fmt.Printf("End of day:%s\n", t.EndOfDay())
  fmt.Printf("Start of month:%s\n", t.StartOfMonth())
  fmt.Printf("End of month:%s\n", t.EndOfMonth())
  fmt.Printf("Start of year:%s\n", t.StartOfYear())
  fmt.Printf("End of year:%s\n", t.EndOfYear())
  fmt.Printf("Start of decade:%s\n", t.StartOfDecade())
  fmt.Printf("End of decade:%s\n", t.EndOfDecade())
  fmt.Printf("Start of century:%s\n", t.StartOfCentury())
  fmt.Printf("End of century:%s\n", t.EndOfCentury())
  fmt.Printf("Start of week:%s\n", t.StartOfWeek())
  fmt.Printf("End of week:%s\n", t.EndOfWeek())
  fmt.Printf("Next:%s\n", t.Next(time.Wednesday))
  fmt.Printf("Previous:%s\n", t.Previous(time.Wednesday))
}
```

### 自定义工作日和周末

有些地区每周的开始、周末和我们的不一样。例如，在美国周日是新的一周开始。没关系，`carbon`可以自定义每周的开始和周末：

```golang
package main

import (
  "fmt"
  "log"
  "time"

  "github.com/uniplaces/carbon"
)

func main() {
  t, err := carbon.Create(2020, 02, 11, 0, 0, 0, 0, "Asia/Shanghai")
  if err != nil {
    log.Fatal(err)
  }

  t.SetWeekStartsAt(time.Sunday)
  t.SetWeekEndsAt(time.Saturday)
  t.SetWeekendDays([]time.Weekday{time.Monday, time.Tuesday, time.Wednesday})

  fmt.Printf("Today is %s, weekend? %t\n", t.Weekday(), t.IsWeekend())
}
```

## 总结

`carbon`提供了很多的实用方法，另外`time`的方法它也能使用，使得它的功能非常强大。时间其实是一个非常复杂的问题，考虑到时区、闰秒、各地的夏令时等，自己处理起来简直是火葬场。幸好有这些库(┬＿┬)

## 参考

1. carbon GitHub 仓库：[https://github.com/uniplaces/carbon](https://github.com/uniplaces/carbon)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)