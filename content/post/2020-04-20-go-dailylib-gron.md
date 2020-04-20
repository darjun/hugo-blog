---
layout:    post
title:    "Go 每日一库之 gron"
subtitle: 	"每天学习一个 Go 库"
date:		2020-04-19T22:13:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/04/20/godailylib/gron"
categories: [
	"Go"
]
---

## 简介

[`gron`](https://github.com/roylee0704/gron)是一个比较小巧、灵活的定时任务库，可以执行定时的、周期性的任务。`gron`提供简洁的、并发安全的接口。我们先介绍`gron`库的使用，然后简单分析一下源码。

## 快速使用

先安装：

```golang
$ go get github.com/roylee0704/gron
```

后使用：

```golang
package main

import (
  "fmt"
  "sync"
  "time"

  "github.com/roylee0704/gron"
)

func main() {
  var wg sync.WaitGroup
  wg.Add(1)

  c := gron.New()
  c.AddFunc(gron.Every(5*time.Second), func() {
    fmt.Println("runs every 5 seconds.")
  })
  c.Start()

  wg.Wait()
}
```

`gron`的使用比较简单：

* 首先调用`gron.New()`创建一个**管理器**，这是一个定时任务的管理器；
* 然后调用管理器的`AddFunc()`或`Add()`方法向它添加任务，在启动时添加也是可以的，见下文分析；
* 最后调用管理器的`Start()`方法启动它。

`gron`支持两种添加任务的方式，一种是使用**无参数**的函数，另一种是实现任务接口。上面例子中使用的是前一种方式，实现接口的方式我们后面会介绍。添加任务时通过`gron.Every()`指定周期任务的间隔，上面添加了一个 5s 的周期任务，每隔 5s 输出一行文字。

需要注意的是，我们使用`sync.WaitGroup`保证主 goroutine 不退出。因为`c.Start()`中只是启动了一个 goroutine，如果主 goroutine 退出了，整个程序就停止了。

运行程序，每隔 5s 输出：

```golang
runs every 5 seconds.
runs every 5 seconds.
runs every 5 seconds.
```

该程序需要按下`ctrl + c`停止！

## 时间格式

`gron`接受`time.Duration`类型的时间间隔，除了`time`包中定义的基础`Second/Minute/Hour`，`gron`中的`xtime`子包还提供了`Day/Week`单位的时间。有一点需要注意，`gron`支持的时间精度为 1s，小于 1s 的间隔是不支持的。除了单位时间间隔，我们还可以使用`4m10s`这样的时间：

```golang
func main() {
  var wg sync.WaitGroup
  wg.Add(1)

  c := gron.New()
  c.AddFunc(gron.Every(1*time.Second), func() {
    fmt.Println("runs every second.")
  })
  c.AddFunc(gron.Every(1*time.Minute), func() {
    fmt.Println("runs every minute.")
  })
  c.AddFunc(gron.Every(1*time.Hour), func() {
    fmt.Println("runs every hour.")
  })
  c.AddFunc(gron.Every(1*xtime.Day), func() {
    fmt.Println("runs every day.")
  })
  c.AddFunc(gron.Every(1*xtime.Week), func() {
    fmt.Println("runs every week.")
  })
  t, _ := time.ParseDuration("4m10s")
  c.AddFunc(gron.Every(t), func() {
    fmt.Println("runs every 4 minutes 10 seconds.")
  })
  c.Start()

  wg.Wait()
}
```

通过`gron.Every()`设置每隔多长时间执行一次任务。对于大于 1 天的时间间隔，我们还可以使用`gron.Every().At()`指定其在某个时间点执行。例如下面的程序，从第二天的`22:00`开始，每隔一天触发一次，即每天的`22:00`触发：

```golang
func main() {
  var wg sync.WaitGroup
  wg.Add(1)

  c := gron.New()
  c.AddFunc(gron.Every(1*xtime.Day).At("22:00"), func() {
    fmt.Println("runs every second.")
  })
  c.Start()

  wg.Wait()
}
```

## 自定义任务

实现自定义任务也很简单，只需要实现`gron.Job`接口即可：

```golang
// src/github.com/roylee0704/gron/cron.go
type Job interface {
  Run()
}
```

我们需要调用调度器的`Add()`方法向管理器添加自定义任务：

```golang
type GreetingJob struct {
  Name string
}

func (g GreetingJob) Run() {
  fmt.Println("Hello ", g.Name)
}

func main() {
  var wg sync.WaitGroup
  wg.Add(1)

  g1 := GreetingJob{Name: "dj"}
  g2 := GreetingJob{Name: "dajun"}

  c := gron.New()
  c.Add(gron.Every(5*time.Second), g1)
  c.Add(gron.Every(10*time.Second), g2)
  c.Start()

  wg.Wait()
}
```

上面我们编写了一个`GreetingJob`结构，实现`gron.Job`接口，然后创建两个对象`g1/g2`，一个 5s 触发一次，一个 10s 触发一次。使用自定义任务的方式可以比较好地处理携带状态的任务，如上面的`Name`字段。

实际上，`AddFunc()`方法内部也是通过`Add()`实现的：

```golang
// src/github.com/roylee0704/gron/cron.go
func (c *Cron) AddFunc(s Schedule, j func()) {
  c.Add(s, JobFunc(j))
}

type JobFunc func()

func (j JobFunc) Run() {
  j()
}
```

在`AddFunc()`内部，将传入的函数转为`JobFunc`类型，而`gron`为`JobFunc`实现了`gron.Job`接口。是不是与`net/http`包中的`HandleFunc`和`Handle`很像。如果注意观察的话，在很多 Go 语言的代码中都有此类模式。

## 一点源码

`gron`的源码只有两个文件`cron.go`和`schedule.go`，`cron.go`中实现添加任务和调度的方法，`schedule.go`中是时间策略相关的代码。两个文件算上注释一共才 260 行！我们添加的任务在`gron`内部都是以`Entry`结构表示的：

```golang
type Entry struct {
  Schedule Schedule
  Job      Job
  Next time.Time
  Prev time.Time
}
```

`Next`为下次执行时间，`Prev`为上次执行时间，`Job`是要执行的任务，`Schedule`为`gron.Schedule`接口类型，调用其`Next()`可计算出下次执行的时间点。

管理器使用`gron.Cron`结构表示：

```golang
type Cron struct {
  entries []*Entry
  running bool
  add     chan *Entry
  stop    chan struct{}
}
```

任务的调度在另外一个 goroutine 中。如果调度未开始，添加任务可直接`append`到`entries`切片中；如果调度已开始（`Start()`方法已调用），需要向通道`add`发送待添加的任务。任务调度的核心逻辑在`Run()`方法中：

```golang
func (c *Cron) run() {
  var effective time.Time
  now := time.Now().Local()

  // to figure next trig time for entries, referenced from now
  for _, e := range c.entries {
    e.Next = e.Schedule.Next(now)
  }

  for {
    sort.Sort(byTime(c.entries))
    if len(c.entries) > 0 {
      effective = c.entries[0].Next
    } else {
      effective = now.AddDate(15, 0, 0) // to prevent phantom jobs.
    }

    select {
    case now = <-after(effective.Sub(now)):
      // entries with same time gets run.
      for _, entry := range c.entries {
        if entry.Next != effective {
          break
        }
        entry.Prev = now
        entry.Next = entry.Schedule.Next(now)
        go entry.Job.Run()
      }
    case e := <-c.add:
      e.Next = e.Schedule.Next(time.Now())
      c.entries = append(c.entries, e)
    case <-c.stop:
      return // terminate go-routine.
    }
  }
}
```

执行流程如下：

1. 调度器刚启动时，先计算所有任务的下次执行时间；
2. 然后在一个`for`循环中，按照执行时间从早到晚排序，取出最近需要执行任务的时间点；
3. 在`select`语句中等待到这个时间点，启动新的 goroutine 执行到期的任务，每个任务一个新的 goroutine；
4. 如果在等待的过程中，又添加了新的任务（通过通道`c.add`），计算这个新任务的首次执行时间。跳到步骤 2，因为新添加的任务可能最早执行。

有几个细节需要注意一下：

1. 任务到期判断使用的是本地时间：`time.Now().Local()`；
2. 如果没有任务，等待时间设置为`now.AddDate(15, 0, 0)`，即 15 年，防止 CPU 空转；
3. 任务都是在独立的 goroutine 中执行的；
4. 通过实现`sort.Interface`接口可以实现自定义排序（代码中的`byTime`）。

最后，我们来看一下时间策略的代码。我们知道在`Entry`结构中存储了一个`gron.Schedule`类型的对象，调用该对象的`Next()`方法返回下次执行的时间点：

```golang
// src/github.com/roylee0704/gron/schedule.go
type Schedule interface {
  Next(t time.Time) time.Time
}
```

`gron`内置实现了两种`Schedule`，一种是`periodicSchedule`，即**周期触发**，`gron.Every()`函数返回的就是这个对象：

```golang
// src/github.com/roylee0704/gron/schedule.go
type periodicSchedule struct {
  period time.Duration
}
```

一种是**固定时刻的周期触发**，它实际上也是周期触发，只是固定了时间点：

```golang
type atSchedule struct {
  period time.Duration
  hh     int
  mm     int
}
```

他们的核心逻辑在`Next()`方法中，`periodicSchedule`只需要用当前时间加上周期即可得到下次触发时间。这里`Truncate()`方法截掉了当前时间中小于 1s 的部分：

```golang
func (ps periodicSchedule) Next(t time.Time) time.Time {
  return t.Truncate(time.Second).Add(ps.period)
}
```

`atSchedule`的`Next()`方法先计算当天该时间点，再加上周期就是下次触发的时间：

```golang
func (as atSchedule) reset(t time.Time) time.Time {
  return time.Date(t.Year(), t.Month(), t.Day(), as.hh, as.mm, 0, 0, time.UTC)
}

func (as atSchedule) Next(t time.Time) time.Time {
  next := as.reset(t)
  if t.After(next) {
    return next.Add(as.period)
  }
  return next
}
```

`periodicSchedule`提供了`At()`方法可以转为`atSchedule`：

```golang
func (ps periodicSchedule) At(t string) Schedule {
  if ps.period < xtime.Day {
    panic("period must be at least in days")
  }

  // parse t naively
  h, m, err := parse(t)

  if err != nil {
    panic(err.Error())
  }

  return &atSchedule{
    period: ps.period,
    hh:     h,
    mm:     m,
  }
}
```

## 自定义时间策略

我们可以很轻松的实现一个自定义的时间策略。例如，我们要实现一个“指数退避”的时间序列，先等待 1s，然后 2s、4s...

```golang
type ExponentialBackOffSchedule struct {
	last int
}

func (e *ExponentialBackOffSchedule) Next(t time.Time) time.Time {
	interval := time.Duration(math.Pow(2.0, float64(e.last))) * time.Second
	e.last += 1
	return t.Truncate(time.Second).Add(interval)
}

func main() {
	var wg sync.WaitGroup
	wg.Add(1)

	c := gron.New()
	c.AddFunc(&ExponentialBackOffSchedule{}, func() {
		fmt.Println(time.Now().Local().Format("2006-01-02 15:04:05"), "hello")
	})
	c.Start()

	wg.Wait()
}
```

运行结果如下：

```golang
2020-04-20 23:47:11 hello
2020-04-20 23:47:13 hello
2020-04-20 23:47:17 hello
2020-04-20 23:47:25 hello
```

第二次输出与第一次相差 2s，第三次与第二次相差 4s，第4次与第三次相差 8s，完美！


## 总结

本文介绍了`gron`这个小巧的定时任务库，如何使用，如何自定义任务和时间策略，顺带分析了一下源码。`gron`源码实现非常简洁，非常推荐阅读！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gron GitHub：[https://github.com/roylee0704/gron](https://github.com/roylee0704/gron)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)