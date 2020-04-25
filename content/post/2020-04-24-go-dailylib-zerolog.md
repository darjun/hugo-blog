---
layout:    post
title:    "Go 每日一库之 zerolog"
subtitle: 	"每天学习一个 Go 库"
date:		2020-04-24T23:57:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/04/24/godailylib/zerolog"
categories: [
	"Go"
]
---

## 简介

每个编程语言都有很多日志库，因为记录日志在每个项目中都是必须的。前面我们介绍了标准日志库[`log`](https://darjun.github.io/2020/02/07/godailylib/log/)、好用的[`logrus`](https://darjun.github.io/2020/02/07/godailylib/logrus/)和上一篇文章中介绍的由 uber 开源的高性能日志库[`zap`](https://darjun.github.io/2020/04/23/godailylib/zap)。`zerolog`相比`zap`更进了一步，它的 API 设计非常注重开发体验和性能。`zerolog`只专注于记录 JSON 格式的日志，号称 0 内存分配！

## 快速使用

先安装：

```golang
$ go get github.com/rs/zerolog/log
```

后使用：

```golang
package main

import "github.com/rs/zerolog/log"

func main() {
  log.Print("hello world")
}
```

常规使用与标准库`log`非常相似，只不过输出的是 JSON 格式的日志：

```golang
{"level":"debug","time":"2020-04-25T13:43:08+08:00","message":"hello world"}
```

## 字段

我们可以在日志中添加额外的字段信息，有助于调试和问题追踪。与`zap`一样，`zerolog`也区分字段类型，不同的是`zerolog`采用**链式调用**的方式：

```golang
func main() {
  log.Debug().
    Str("Scale", "833 cents").
    Float64("Interval", 833.09).
    Msg("Fibonacci is everywhere")

  log.Debug().
    Str("Name", "Tom").
    Send()
}
```

调用`Msg()`或`Send()`之后，日志会被输出：

```golang
{"level":"debug","Scale":"833 cents","Interval":833.09,"time":"2020-04-25T13:55:44+08:00","message":"Fibonacci is everywhere"}
{"level":"debug","Name":"Tom","time":"2020-04-25T13:55:44+08:00"}
```

### 嵌套

记录的字段可以任意嵌套，这通过`Dict()`来实现：

```golang
func main() {
  log.Info().
    Dict("dict", zerolog.Dict().
      Str("bar", "baz").
      Int("n", 1),
    ).Msg("hello world")
}
```

输出中`dict`字段为嵌套结构：

```golang
{"level":"info","dict":{"bar":"baz","n":1},"time":"2020-04-25T14:34:51+08:00","message":"hello world"}
```

## 全局`Logger`

上面我们使用`log.Debug()`、`log.Info()`调用的是全局的`Logger`。全局的`Logger`使用比较简单，不需要额外创建。

### 设置日志级别

每个日志库都有日志级别的概念，而且划分基本上都差不多。`zerolog`有`panic/fatal/error/warn/info/debug/trace`这几种级别。我们可以调用`SetGlobalLevel()`设置全局`Logger`的日志级别。

```golang
func main() {
  debug := flag.Bool("debug", false, "sets log level to debug")
  flag.Parse()

  if *debug {
    zerolog.SetGlobalLevel(zerolog.DebugLevel)
  } else {
    zerolog.SetGlobalLevel(zerolog.InfoLevel)
  }

  log.Debug().Msg("This message appears only when log level set to debug")
  log.Info().Msg("This message appears when log level set to debug or info")

  if e := log.Debug(); e.Enabled() {
    e.Str("foo", "bar").Msg("some debug message")
  }
}
```

在上面代码中，我们根据传入的命令行选项设置日志级别是`Debug`还是`Info`。如果日志级别为`Info`，`Debug`的日志是不会输出的。也可以调用`Enabled()`方法来判断日志是否需要输出，需要时再调用相应方法输出，节省了添加字段和日志信息的开销：

```golang
if e := log.Debug(); e.Enabled() {
  e.Str("foo", "bar").Msg("some debug message")
}
```

先不加命令行参数运行，默认为`Info`级别，`Debug`日志不会输出：

```golang
$ go run main.go
{"level":"info","time":"2020-04-25T14:13:34+08:00","message":"This message appears when log level set to debug or info"}
```

加上`-debug`选项，`Debug`和`Info`日志都输出了：

```golang
$ go run main.go -debug
{"level":"debug","time":"2020-04-25T14:18:19+08:00","message":"This message appears only when log level set to debug"}
{"level":"info","time":"2020-04-25T14:18:19+08:00","message":"This message appears when log level set to debug or info"}
{"level":"debug","foo":"bar","time":"2020-04-25T14:18:19+08:00","message":"some debug 
message"}
```

### 不输出级别和信息

有时候我们不想输出日志级别（即`level`字段），这时可以使用`log.Log()`方法。有时，我们没有日志信息可输出，这时传一个空字符串给`Msg()`方法：

```golang
func main() {
  log.Log().
    Str("foo", "bar").
    Msg("")
}
```

运行：

```golang
{"foo":"bar","time":"2020-04-25T14:19:48+08:00"}
```

## 创建`Logger`

上面我们使用的都是全局的`Logger`，这种方式有一个明显的缺点：如果在某个地方修改了设置，将影响全局的日志记录。为了消除这种影响，我们需要创建新的`Logger`：

```golang
func main() {
  logger := zerolog.New(os.Stderr)
  logger.Info().Str("foo", "bar").Msg("hello world")
}
```

调用`zerlog.New()`传入一个`io.Writer`作为日志写入器即可。

### 子`Logger`

基于当前的`Logger`可以创建一个子`Logger`，子`Logger`可以在父`Logger`上附加一些额外的字段。调用`logger.With()`创建一个上下文，然后为它添加字段，最后调用`Logger()`返回一个新的`Logger`：

```golang
func main() {
  logger := zerolog.New(os.Stderr)
  sublogger := logger.With().
    Str("foo", "bar").
    Logger()
  sublogger.Info().Msg("hello world")
}
```

`sublogger`会额外输出`"foo": "bar"`这个字段。

## 设置

`zerolog`提供了多种选项定制输入日志的行为。

### 美化输出

`zerolog`提供了一个`ConsoleWriter`可输出便于我们阅读的，带颜色的日志。调用`zerolog.Output()`来启用`ConsoleWriter`：

```golang
func main() {
  logger := log.Output(zerolog.ConsoleWriter{Out: os.Stderr})
  logger.Info().Str("foo", "bar").Msg("hello world")
}
```

输出：

![](/img/in-post/godailylib/zerolog1.png#center)

我们还能进一步对`ConsoleWriter`进行配置，定制输出的级别、信息、字段名、字段值的格式：

```golang
func main() {
  output := zerolog.ConsoleWriter{Out: os.Stderr, TimeFormat: time.RFC3339}
  output.FormatLevel = func(i interface{}) string {
    return strings.ToUpper(fmt.Sprintf("| %-6s|", i))
  }
  output.FormatMessage = func(i interface{}) string {
    return fmt.Sprintf("***%s****", i)
  }
  output.FormatFieldName = func(i interface{}) string {
    return fmt.Sprintf("%s:", i)
  }
  output.FormatFieldValue = func(i interface{}) string {
    return strings.ToUpper(fmt.Sprintf("%s", i))
  }

  logger := log.Output(output).With().Timestamp().Logger()
  logger.Info().Str("foo", "bar").Msg("hello world")
}
```

实际上就是对级别、信息、字段名和字段值设置钩子，输出前经过钩子函数转换一次：

![](/img/in-post/godailylib/zerolog2.png#center)

**`ConsoleWriter`的性能不够理想，建议只在开发环境中使用！**

### 设置自动添加的字段名

输出的日志中级别默认的字段名为`level`，信息默认为`message`，时间默认为`time`。可以通过`zerolog`中`LevelFieldName/MessageFieldName/TimestampFieldName`来设置：

```golang
func main() {
  zerolog.TimestampFieldName = "t"
  zerolog.LevelFieldName = "l"
  zerolog.MessageFieldName = "m"

  logger := zerolog.New(os.Stderr).With().Timestamp().Logger()
  logger.Info().Msg("hello world")
}
```

输出：

```golang
{"l":"info","t":"2020-04-25T14:53:08+08:00","m":"hello world"}
```

**注意，这个设置是全局的！！！**

## 输出文件名和行号

有时我们需要输出文件名和行号，以便能很快定位代码位置，方便找出问题。这可以通过在创建子`Logger`时带入`Caller()`选项完成：

```golang
func main() {
  logger := zerolog.New(os.Stderr).With().Caller().Logger()
  logger.Info().Msg("hello world")
}
```

输出：

```golang
{"level":"info","caller":"d:/code/golang/src/github.com/darjun/go-daily-lib/zerolog/setting/file-line/main.go:11","message":"hello world"}
```

## 日志采样

有时候日志太多了反而对我们排查问题造成干扰，`zerolog`支持日志采样的功能，可以每隔多少条日志输出一次，其他日志丢弃：

```golang
func main() {
  sampled := log.Sample(&zerolog.BasicSampler{N: 10})

  for i := 0; i < 20; i++ {
    sampled.Info().Msg("will be logged every 10 message")
  }
}
```

结果只输出两条：

```golang
{"level":"info","time":"2020-04-25T15:01:02+08:00","message":"will be logged every 10 message"}
{"level":"info","time":"2020-04-25T15:01:02+08:00","message":"will be logged every 10 message"}
```

还有更高级的设置：

```golang
func main() {
  sampled := log.Sample(&zerolog.LevelSampler{
    DebugSampler: &zerolog.BurstSampler{
      Burst:       5,
      Period:      time.Second,
      NextSampler: &zerolog.BasicSampler{N: 100},
    },
  })

  sampled.Debug().Msg("hello world")
}
```

上面代码只采样`Debug`日志，在 1s 内最多输出 5 条日志，超过 5条 时，每隔 100 条输出一条。

## 钩子

`zerolog`支持钩子，我们可以针对不同的日志级别添加一些额外的字段或进行其他的操作：

```golang
type AddFieldHook struct {
}

func (AddFieldHook) Run(e *zerolog.Event, level zerolog.Level, msg string) {
  if level == zerolog.DebugLevel {
    e.Str("name", "dj")
  }
}

func main() {
  hooked := log.Hook(AddFieldHook{})
  hooked.Debug().Msg("")
}
```

如果是`Debug`级别，额外输出`"name":"dj"`字段：

```golang
{"level":"debug","time":"2020-04-25T15:09:04+08:00","name":"dj"}
```

## 性能

关于性能，GitHub 上有一份详细的性能测试，与`logrus/zap`等日志库的比较。感兴趣可以去看看：[https://github.com/rs/zerolog#benchmarks](https://github.com/rs/zerolog#benchmarks)。`zerolog`的性能比`zap`还要优秀！

## 总结

正是因为有很多人不满足于现状，才带来了技术的进步和丰富多彩的开源世界！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. zerolog GitHub：[https://github.com/rs/zerolog](https://github.com/rs/zerolog)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)