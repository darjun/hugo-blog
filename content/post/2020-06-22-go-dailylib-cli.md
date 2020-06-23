---
layout:    post
title:    "Go 每日一库之 cli"
subtitle: 	"每天学习一个 Go 库"
date:		2020-06-22T08:46:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/06/22/godailylib/cli"
categories: [
	"Go"
]
---

## 简介

[`cli`](https://github.com/urfave/cli)是一个用于构建命令行程序的库。我们之前也介绍过一个用于构建命令行程序的库[`cobra`](https://darjun.github.io/2020/01/17/godailylib/cobra/)。在功能上来说两者差不多，`cobra`的优势是提供了一个脚手架，方便开发。`cli`非常简洁，所有的初始化操作就是创建一个`cli.App`结构的对象。通过为对象的字段赋值来添加相应的功能。

`cli`与我们上一篇文章介绍的`negroni`是同一个作者[urfave](https://github.com/urfave)。

## 快速使用

`cli`需要搭配 Go Modules 使用。创建目录并初始化：

```cmd
$ mkdir cli && cd cli
$ go mod init github.com/darjun/go-daily-lib/cli
```

安装`cli`库，有`v1`和`v2`两个版本。如果没有特殊需求，一般安装`v2`版本：

```cmd
$ go get -u github.com/urfave/cli/v2
```

使用：

```golang
package main

import (
  "fmt"
  "log"
  "os"

  "github.com/urfave/cli/v2"
)

func main() {
  app := &cli.App{
    Name:  "hello",
    Usage: "hello world example",
    Action: func(c *cli.Context) error {
      fmt.Println("hello world")
      return nil
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

使用非常简单，理论上创建一个`cli.App`结构的对象，然后调用其`Run()`方法，传入命令行的参数即可。一个空白的`cli`应用程序如下：

```golang
func main() {
  (&cli.App{}).Run(os.Args)
}
```

但是这个空白程序没有什么用处。我们的`hello world`程序，设置了`Name/Usage/Action`。`Name`和`Usage`都显示在帮助中，`Action`是调用该命令行程序时实际执行的函数，需要的信息可以从参数`cli.Context`获取。

编译、运行（环境：Win10 + Git Bash）：

```cmd
$ go build -o hello
$ ./hello
hello world
```

除了这些，`cli`为我们额外生成了帮助信息：

```cmd
$ ./hello --help
NAME:
   hello - hello world example

USAGE:
   hello [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h  show help (default: false)
```

## 参数

通过`cli.Context`的相关方法我们可以获取传给命令行的参数信息：

* `NArg()`：返回参数个数；
* `Args()`：返回`cli.Args`对象，调用其`Get(i)`获取位置`i`上的参数。

示例：

```golang
func main() {
  app := &cli.App{
    Name:  "arguments",
    Usage: "arguments example",
    Action: func(c *cli.Context) error {
      for i := 0; i < c.NArg(); i++ {
        fmt.Printf("%d: %s\n", i+1, c.Args().Get(i))
      }
      return nil
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

这里只是简单输出：

```cmd
$ go run main.go hello world
1: hello
2: world
```

## 选项

一个好用的命令行程序怎么会少了选项呢？`cli`设置和获取选项非常简单。在`cli.App{}`结构初始化时，设置字段`Flags`即可添加选项。`Flags`字段是`[]cli.Flag`类型，`cli.Flag`实际上是接口类型。`cli`为常见类型都实现了对应的`XxxFlag`，如`BoolFlag/DurationFlag/StringFlag`等。它们有一些共用的字段，`Name/Value/Usage`（名称/默认值/释义）。看示例：

```golang
func main() {
  app := &cli.App{
    Flags: []cli.Flag{
      &cli.StringFlag{
        Name:  "lang",
        Value: "english",
        Usage: "language for the greeting",
      },
    },
    Action: func(c *cli.Context) error {
      name := "world"
      if c.NArg() > 0 {
        name = c.Args().Get(0)
      }

      if c.String("lang") == "english" {
        fmt.Println("hello", name)
      } else {
        fmt.Println("你好", name)
      }
      return nil
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

上面是一个打招呼的命令行程序，可通过选项`lang`指定语言，默认为英语。设置选项为非`english`的值，使用汉语。如果有参数，使用第一个参数作为人名，否则使用`world`。注意选项是通过`c.Type(name)`来获取的，`Type`为选项类型，`name`为选项名。编译、运行：

```cmd
$ go build -o flags

# 默认调用
$ ./flags
hello world

# 设置非英语
$ ./flags --lang chinese
你好 world

# 传入参数作为人名
$ ./flags --lang chinese dj
你好 dj
```

我们可以通过`./flags --help`来查看选项：

```cmd
$ ./flags --help
NAME:
   flags - A new cli application

USAGE:
   flags [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --lang value  language for the greeting (default: "english")
   --help, -h    show help (default: false)
```

### 存入变量

除了通过`c.Type(name)`来获取选项的值，我们还可以将选项存到某个预先定义好的变量中。只需要设置`Destination`字段为变量的地址即可：

```golang
func main() {
  var language string

  app := &cli.App{
    Flags: []cli.Flag{
      &cli.StringFlag{
        Name:        "lang",
        Value:       "english",
        Usage:       "language for the greeting",
        Destination: &language,
      },
    },
    Action: func(c *cli.Context) error {
      name := "world"
      if c.NArg() > 0 {
        name = c.Args().Get(0)
      }

      if language == "english" {
        fmt.Println("hello", name)
      } else {
        fmt.Println("你好", name)
      }
      return nil
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

与上面的程序效果是一样的。

### 占位值

`cli`可以在`Usage`字段中为选项设置占位值，占位值通过反引号 **\`** 包围。只有第一个生效，其他的维持不变。占位值有助于生成易于理解的帮助信息：

```golang
func main() {
  app := & cli.App{
    Flags : []cli.Flag {
      &cli.StringFlag{
        Name:"config",
        Usage: "Load configuration from `FILE`",
      },
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

设置占位值之后，帮助信息中，该占位值会显示在对应的选项后面，对短选项也是有效的：

```cmd
$ go build -o placeholder
$ ./placeholder --help
NAME:
   placeholder - A new cli application

USAGE:
   placeholder [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --config FILE  Load configuration from FILE
   --help, -h     show help (default: false)
```

### 别名

选项可以设置多个别名，设置对应选项的`Aliases`字段即可：

```golang
func main() {
  app := &cli.App{
    Flags: []cli.Flag{
      &cli.StringFlag{
        Name:    "lang",
        Aliases: []string{"language", "l"},
        Value:   "english",
        Usage:   "language for the greeting",
      },
    },
    Action: func(c *cli.Context) error {
      name := "world"
      if c.NArg() > 0 {
        name = c.Args().Get(0)
      }

      if c.String("lang") == "english" {
        fmt.Println("hello", name)
      } else {
        fmt.Println("你好", name)
      }
      return nil
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

使用`--lang chinese`、`--language chinese`和`-l chinese`效果是一样的。如果通过不同的名称指定同一个选项，会报错：

```golang
$ go build -o aliase
$ ./aliase --lang chinese
你好 world
$ ./aliase --language chinese
你好 world
$ ./aliase -l chinese
你好 world
$ ./aliase -l chinese --lang chinese
Cannot use two forms of the same flag: l lang
```

### 环境变量

除了通过执行程序时手动指定命令行选项，我们还可以读取指定的环境变量作为选项的值。只需要将环境变量的名字设置到选项对象的`EnvVars`字段即可。可以指定多个环境变量名字，`cli`会依次查找，第一个有值的环境变量会被使用。

```golang
func main() {
  app := &cli.App{
    Flags: []cli.Flag{
      &cli.StringFlag{
        Name:    "lang",
        Value:   "english",
        Usage:   "language for the greeting",
        EnvVars: []string{"APP_LANG", "SYSTEM_LANG"},
      },
    },
    Action: func(c *cli.Context) error {
      if c.String("lang") == "english" {
        fmt.Println("hello")
      } else {
        fmt.Println("你好")
      }
      return nil
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

编译、运行：

```cmd
$ go build -o env
$ APP_LANG=chinese ./env
你好
```

### 文件

`cli`还支持从文件中读取选项的值，设置选项对象的`FilePath`字段为文件路径：

```golang
func main() {
  app := &cli.App{
    Flags: []cli.Flag{
      &cli.StringFlag{
        Name:     "lang",
        Value:    "english",
        Usage:    "language for the greeting",
        FilePath: "./lang.txt",
      },
    },
    Action: func(c *cli.Context) error {
      if c.String("lang") == "english" {
        fmt.Println("hello")
      } else {
        fmt.Println("你好")
      }
      return nil
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

在`main.go`同级目录创建一个`lang.txt`，输入内容`chinese`。然后编译运行程序：

```cmd
$ go build -o file
$ ./file
你好
```

`cli`还支持从`YAML/JSON/TOML`等配置文件中读取选项值，这里就不一一介绍了。

### 选项优先级

上面我们介绍了几种设置选项值的方式，如果同时有多个方式生效，按照下面的优先级从高到低设置：

* 用户指定的命令行选项值；
* 环境变量；
* 配置文件；
* 选项的默认值。

### 组合短选项

我们时常会遇到有多个短选项的情况。例如 linux 命令`ls -a -l`，可以简写为`ls -al`。`cli`也支持短选项合写，只需要设置`cli.App`的`UseShortOptionHandling`字段为`true`即可：

```golang
func main() {
  app := &cli.App{
    UseShortOptionHandling: true,
    Commands: []*cli.Command{
      {
        Name:  "short",
        Usage: "complete a task on the list",
        Flags: []cli.Flag{
          &cli.BoolFlag{Name: "serve", Aliases: []string{"s"}},
          &cli.BoolFlag{Name: "option", Aliases: []string{"o"}},
          &cli.BoolFlag{Name: "message", Aliases: []string{"m"}},
        },
        Action: func(c *cli.Context) error {
          fmt.Println("serve:", c.Bool("serve"))
          fmt.Println("option:", c.Bool("option"))
          fmt.Println("message:", c.Bool("message"))
          return nil
        },
      },
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

编译运行：

```cmd
$ go build -o short
$ ./short short -som "some message"
serve: true
option: true
message: true
```

需要特别注意一点，设置`UseShortOptionHandling`为`true`之后，我们不能再通过`-`指定选项了，这样会产生歧义。例如`-lang`，`cli`不知道应该解释为`l/a/n/g` 4 个选项还是`lang` 1 个。`--`还是有效的。

### 必要选项

如果将选项的`Required`字段设置为`true`，那么该选项就是必要选项。必要选项必须指定，否则会报错：

```golang
func main() {
  app := &cli.App{
    Flags: []cli.Flag{
      &cli.StringFlag{
        Name:     "lang",
        Value:    "english",
        Usage:    "language for the greeting",
        Required: true,
      },
    },
    Action: func(c *cli.Context) error {
      if c.String("lang") == "english" {
        fmt.Println("hello")
      } else {
        fmt.Println("你好")
      }
      return nil
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

不指定选项`lang`运行：

```cmd
$ ./required
2020/06/23 22:11:32 Required flag "lang" not set
```

### 帮助文本中的默认值

默认情况下，帮助文本中选项的默认值显示为`Value`字段值。有些时候，`Value`并不是实际的默认值。这时，我们可以通过`DefaultText`设置：

```golang
func main() {
  app := &cli.App{
    Flags: []cli.Flag{
      &cli.IntFlag{
        Name:     "port",
        Value:    0,
        Usage:    "Use a randomized port",
        DefaultText :"random",
      },
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

上面代码逻辑中，如果`Value`设置为 0 就随机一个端口，这时帮助信息中`default: 0`就容易产生误解了。通过`DefaultText`可以避免这种情况：

```cmd
$ go build -o default-text
$ ./default-text --help
NAME:
   default-text - A new cli application

USAGE:
   default-text [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --port value  Use a randomized port (default: random)
   --help, -h    show help (default: false)
```

## 子命令

子命令使命令行程序有更好的组织性。`git`有大量的命令，很多以某个命令下的子命令存在。例如`git remote`命令下有`add/rename/remove`等子命令，`git submodule`下有`add/status/init/update`等子命令。

`cli`通过设置`cli.App`的`Commands`字段添加命令，设置各个命令的`SubCommands`字段，即可添加子命令。非常方便！

```golang
func main() {
  app := &cli.App{
    Commands: []*cli.Command{
      {
        Name:    "add",
        Aliases: []string{"a"},
        Usage:   "add a task to the list",
        Action: func(c *cli.Context) error {
          fmt.Println("added task: ", c.Args().First())
          return nil
        },
      },
      {
        Name:    "complete",
        Aliases: []string{"c"},
        Usage:   "complete a task on the list",
        Action: func(c *cli.Context) error {
          fmt.Println("completed task: ", c.Args().First())
          return nil
        },
      },
      {
        Name:    "template",
        Aliases: []string{"t"},
        Usage:   "options for task templates",
        Subcommands: []*cli.Command{
          {
            Name:  "add",
            Usage: "add a new template",
            Action: func(c *cli.Context) error {
              fmt.Println("new task template: ", c.Args().First())
              return nil
            },
          },
          {
            Name:  "remove",
            Usage: "remove an existing template",
            Action: func(c *cli.Context) error {
              fmt.Println("removed task template: ", c.Args().First())
              return nil
            },
          },
        },
      },
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

上面定义了 3 个命令`add/complete/template`，`template`命令定义了 2 个子命令`add/remove`。编译、运行：

```cmd
$ go build -o subcommand
$ ./subcommand add dating
added task:  dating
$ ./subcommand complete dating
completed task:  dating
$ ./subcommand template add alarm
new task template:  alarm
$ ./subcommand template remove alarm
removed task template:  alarm
```

注意一点，子命令默认不显示在帮助信息中，需要显式调用子命令所属命令的帮助（`./subcommand template --help`）：

```cmd
$ ./subcommand --help
NAME:
   subcommand - A new cli application

USAGE:
   subcommand [global options] command [command options] [arguments...]

COMMANDS:
   add, a       add a task to the list
   complete, c  complete a task on the list
   template, t  options for task templates
   help, h      Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h  show help (default: false)

$ ./subcommand template --help
NAME:
   subcommand template - options for task templates

USAGE:
   subcommand template command [command options] [arguments...]

COMMANDS:
   add      add a new template
   remove   remove an existing template
   help, h  Shows a list of commands or help for one command

OPTIONS:
   --help, -h  show help (default: false)
```

### 分类

在子命令数量很多的时候，可以设置`Category`字段为它们分类，在帮助信息中会将相同分类的命令放在一起展示：

```golang
func main() {
  app := &cli.App{
    Commands: []*cli.Command{
      {
        Name:  "noop",
        Usage: "Usage for noop",
      },
      {
        Name:     "add",
        Category: "template",
        Usage:    "Usage for add",
      },
      {
        Name:     "remove",
        Category: "template",
        Usage:    "Usage for remove",
      },
    },
  }

  err := app.Run(os.Args)
  if err != nil {
    log.Fatal(err)
  }
}
```

编译、运行：

```cmd
$ go build -o categories
$ ./categories --help
NAME:
   categories - A new cli application

USAGE:
   categories [global options] command [command options] [arguments...]

COMMANDS:
   noop     Usage for noop
   help, h  Shows a list of commands or help for one command
   template:
     add     Usage for add
     remove  Usage for remove

GLOBAL OPTIONS:
   --help, -h  show help (default: false)
```

看上面的`COMMANDS`部分。

## 自定义帮助信息

在`cli`中所有的帮助信息文本都可以自定义，整个应用的帮助信息模板通过`AppHelpTemplate`指定。命令的帮助信息模板通过`CommandHelpTemplate`设置，子命令的帮助信息模板通过`SubcommandHelpTemplate`设置。甚至可以通过覆盖`cli.HelpPrinter`这个函数自己实现帮助信息输出。下面程序在默认的帮助信息后添加个人网站和微信信息：

```golang
func main() {
  cli.AppHelpTemplate = fmt.Sprintf(`%s

WEBSITE: http://darjun.github.io

WECHAT: GoUpUp`, cli.AppHelpTemplate)

  (&cli.App{}).Run(os.Args)
}
```

编译运行：

```cmd
$ go build -o help
$ ./help --help
NAME:
   help - A new cli application

USAGE:
   help [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h  show help (default: false)


WEBSITE: http://darjun.github.io

WECHAT: GoUpUp
```

我们还可以改写整个模板：

```golang
func main() {
  cli.AppHelpTemplate = `NAME:
  {{.Name}} - {{.Usage}}
USAGE:
  {{.HelpName}} {{if .VisibleFlags}}[global options]{{end}}{{if .Commands}} command [command options]{{end}} {{if .ArgsUsage}}{{.ArgsUsage}}{{else}}[arguments...]{{end}}
  {{if len .Authors}}
AUTHOR:
  {{range .Authors}}{{ . }}{{end}}
  {{end}}{{if .Commands}}
COMMANDS:
 {{range .Commands}}{{if not .HideHelp}}   {{join .Names ", "}}{{ "\t"}}{{.Usage}}{{ "\n" }}{{end}}{{end}}{{end}}{{if .VisibleFlags}}
GLOBAL OPTIONS:
  {{range .VisibleFlags}}{{.}}
  {{end}}{{end}}{{if .Copyright }}
COPYRIGHT:
  {{.Copyright}}
  {{end}}{{if .Version}}
VERSION:
  {{.Version}}
{{end}}
 `

  app := &cli.App{
    Authors: []*cli.Author{
      {
        Name:  "dj",
        Email: "darjun@126.com",
      },
    },
  }
  app.Run(os.Args)
}
```

`{{.XXX}}`其中`XXX`对应`cli.App{}`结构中设置的字段，例如上面`Authors`：

```cmd
$ ./help --help
NAME:
  help - A new cli application
USAGE:
  help [global options] command [command options] [arguments...]

AUTHOR:
  dj <darjun@126.com>

COMMANDS:
    help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
  --help, -h  show help (default: false)
```

注意观察`AUTHOR`部分。

通过覆盖`HelpPrinter`，我们能自己输出帮助信息：

```golang
func main() {
  cli.HelpPrinter = func(w io.Writer, templ string, data interface{}) {
    fmt.Println("Simple help!")
  }

  (&cli.App{}).Run(os.Args)
}
```

编译、运行：

```cmd
$ ./help --help
Simple help!
```

## 内置选项

### 帮助选项

默认情况下，帮助选项为`--help/-h`。我们可以通过`cli.HelpFlag`字段设置：

```golang
func main() {
  cli.HelpFlag = &cli.BoolFlag{
    Name:    "haaaaalp",
    Aliases: []string{"halp"},
    Usage:   "HALP",
  }

  (&cli.App{}).Run(os.Args)
}
```

查看帮助：

```cmd
$ go run main.go --halp
NAME:
   main.exe - A new cli application

USAGE:
   main.exe [global options] command [command options] [arguments...]

COMMANDS:
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --haaaaalp, --halp  HALP (default: false)
```

### 版本选项

默认版本选项`-v/--version`输出应用的版本信息。我们可以通过`cli.VersionFlag`设置版本选项 ：

```golang
func main() {
  cli.VersionFlag = &cli.BoolFlag{
    Name:    "print-version",
    Aliases: []string{"V"},
    Usage:   "print only the version",
  }

  app := &cli.App{
    Name:    "version",
    Version: "v1.0.0",
  }
  app.Run(os.Args)
}
```

这样就可以通过指定`--print-version/-V`输出版本信息了。运行：

```cmd
$ go run main.go --print-version
version version v1.0.0

$ go run main.go -V
version version v1.0.0
```

我们还可以通过设置`cli.VersionPrinter`字段控制版本信息的输出内容：

```golang
const (
  Revision = "0cebd6e32a4e7094bbdbf150a1c2ffa56c34e91b"
)

func main() {
  cli.VersionPrinter = func(c *cli.Context) {
    fmt.Printf("version=%s revision=%s\n", c.App.Version, Revision)
  }

  app := &cli.App{
    Name:    "version",
    Version: "v1.0.0",
  }
  app.Run(os.Args)
}
```

上面程序同时输出版本号和`git`提交的 SHA 值：

```cmd
$ go run main.go -v
version=v1.0.0 revision=0cebd6e32a4e7094bbdbf150a1c2ffa56c34e91b
```

## 总结

`cli`非常灵活，只需要设置`cli.App`的字段值即可实现相应的功能，不需要额外记忆函数、方法。另外`cli`还支持 Bash 自动补全的功能，对 zsh 的支持也比较好，感兴趣可自行探索。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. cli GitHub：[https://github.com/urfave/cli](https://github.com/urfave/cli)
2. Go 每日一库之 cobra：[https://darjun.github.io/2020/01/17/godailylib/cobra/](https://darjun.github.io/2020/01/17/godailylib/cobra/)
3. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)