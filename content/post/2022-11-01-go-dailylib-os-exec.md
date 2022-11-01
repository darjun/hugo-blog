---
layout:    post
title:    "Go中调用外部命令的几种姿势"
subtitle: "每天学习一个 Go 库"
date:     2022-11-01T13:45:30
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2022/11/01/godailylib/osexec"
categories: [
  "Go"
]
---

## 引子

在工作中，我时不时地会需要在Go中调用外部命令。前段时间我做了一个工具，在钉钉群中添加了一个机器人，@这个机器人可以让它执行一些写好的脚本程序完成指定的任务。机器人倒是不难，照着钉钉开发者文档添加好机器人，然后@这个机器人就会向一个你指定的服务器发送一个POST请求，请求中会附带文本消息。所以我要做的就是搭一个Web服务器，可以用go原生的net/http包，也可以用gin/fasthttp/fiber这些Web框架。收到请求之后，检查附带文本中的关键字去调用对应的程序，然后返回结果。

go标准库中的os/exec包对调用外部程序提供了支持，本文详细介绍os/exec的使用姿势。

## 运行命令

Linux中有个`cal`命令，它可以显示指定年、月的日历，如果不指定年、月，默认为当前时间对应的年月。如果使用的是Windows，推荐安装msys2，这个软件包含了绝大多数的Linux常用命令。

![](/img/in-post/godailylib/exec1.png#center)

![](/img/in-post/godailylib/exec2.png#center)

那么，在Go代码中怎么调用这个命令呢？其实也很简单：

```go
func main() {
  cmd := exec.Command("cal")
  err := cmd.Run()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }
}
```

首先，我们调用`exec.Command`传入命令名，创建一个命令对象`exec.Cmd`。接着调用该命令对象的`Run()`方法运行它。

如果你实际运行了，你会发现什么也没有发生，哈哈。事实上，使用os/exec执行命令，标准输出和标准错误默认会被丢弃。

## 显示输出

`exec.Cmd`对象有两个字段`Stdout`和`Stderr`，类型皆为`io.Writer`。我们可以将任意实现了`io.Writer`接口的类型实例赋给这两个字段，继而实现标准输出和标准错误的重定向。`io.Writer`接口在 Go 标准库和第三方库中随处可见，例如`*os.File`、`*bytes.Buffer`、`net.Conn`。所以我们可以将命令的输出重定向到文件、内存缓存甚至发送到网络中。

### 显示到标准输出

将`exec.Cmd`对象的`Stdout`和`Stderr`这两个字段都设置为`os.Stdout`，那么输出内容都将显示到标准输出：

```go
func main() {
  cmd := exec.Command("cal")
  cmd.Stdout = os.Stdout
  cmd.Stderr = os.Stderr
  err := cmd.Run()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }
}
```

运行程序。我在git bash运行，得到如下结果：

![](/img/in-post/godailylib/exec3.png#center)

输出了中文，检查一下环境变量LANG的值，果然是`zh_CN.UTF-8`。如果想输出英文，可以将环境变量LANG设置为`en_US.UTF-8`：

```bash
$ echo $LANG
zh_CN.UTF-8
$ LANG=en_US.UTF-8 go run main.go
```

得到输出：

![](/img/in-post/godailylib/exec4.png#center)

### 输出到文件

打开或创建文件，然后将文件句柄赋给`exec.Cmd`对象的`Stdout`和`Stderr`这两个字段即可实现输出到文件的功能。

```go
func main() {
  f, err := os.OpenFile("out.txt", os.O_WRONLY|os.O_CREATE, os.ModePerm)
  if err != nil {
    log.Fatalf("os.OpenFile() failed: %v\n", err)
  }

  cmd := exec.Command("cal")
  cmd.Stdout = f
  cmd.Stderr = f
  err = cmd.Run()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }
}
```

`os.OpenFile`打开一个文件，指定`os.O_CREATE`标志让操作系统在文件不存在时自动创建一个，返回该文件对象`*os.File`。`*os.File`实现了`io.Writer`接口。

运行程序：
```bash
$ go run main.go
$ cat out.txt
    November 2022   
Su Mo Tu We Th Fr Sa
       1  2  3  4  5
 6  7  8  9 10 11 12
13 14 15 16 17 18 19
20 21 22 23 24 25 26
27 28 29 30
```

### 发送到网络

现在我们来编写一个日历服务，接收年、月信息，返回该月的日历。

```go
func cal(w http.ResponseWriter, r *http.Request) {
  year := r.URL.Query().Get("year")
  month := r.URL.Query().Get("month")

  cmd := exec.Command("cal", month, year)
  cmd.Stdout = w
  cmd.Stderr = w

  err := cmd.Run()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }
}

func main() {
  http.HandleFunc("/cal", cal)
  http.ListenAndServe(":8080", nil)
}
```

这里为了简单，错误处理都省略了。正常情况下，year和month参数都需要做合法性校验。`exec.Command`函数接收一个字符串类型的可变参数作为命令的参数：

```go
func Command(name string, arg ...string) *Cmd
```

运行程序，使用浏览器请求`localhost:8080/cal?year=2021&month=2`得到：

![](/img/in-post/godailylib/exec5.png#center)

### 保存到内存对象中

`*bytes.Buffer`同样也实现了`io.Writer`接口，故如果我们创建一个`*bytes.Buffer`对象，并将其赋给`exec.Cmd`的`Stdout`和`Stderr`这两个字段，那么命令执行之后，该`*bytes.Buffer`对象中保存的就是命令的输出。

```go
func main() {
  buf := bytes.NewBuffer(nil)
  cmd := exec.Command("cal")
  cmd.Stdout = buf
  cmd.Stderr = buf
  err := cmd.Run()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }

  fmt.Println(buf.String())
}
```

运行：

```bash
$ go run main.go
    November 2022   
Su Mo Tu We Th Fr Sa
       1  2  3  4  5
 6  7  8  9 10 11 12
13 14 15 16 17 18 19
20 21 22 23 24 25 26
27 28 29 30
```

运行命令，然后得到输出的字符串或字节切片这种模式是如此的普遍，并且使用便利，`os/exec`包提供了一个便捷方法：`CombinedOutput`。

### 输出到多个目的地

有时，我们希望能输出到文件和网络，同时保存到内存对象。使用go提供的`io.MultiWriter`可以很容易实现这个需求。`io.MultiWriter`很方便地将多个`io.Writer`转为一个`io.Writer`。

我们稍微修改上面的web程序：

```go
func cal(w http.ResponseWriter, r *http.Request) {
  year := r.URL.Query().Get("year")
  month := r.URL.Query().Get("month")

  f, _ := os.OpenFile("out.txt", os.O_CREATE|os.O_WRONLY, os.ModePerm)
  buf := bytes.NewBuffer(nil)
  mw := io.MultiWriter(w, f, buf)

  cmd := exec.Command("cal", month, year)
  cmd.Stdout = mw
  cmd.Stderr = mw

  err := cmd.Run()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }

  fmt.Println(buf.String())
}
```

调用`io.MultiWriter`将多个`io.Writer`整合成一个`io.Writer`，然后将cmd对象的`Stdout`和`Stderr`都赋值为这个`io.Writer`。这样，命令运行时产出的输出会分别送往`http.ResponseWriter`、`*os.File`以及`*bytes.Buffer`。

## 运行命令，获取输出

前面提到，我们常常需要运行命令，返回输出。`exec.Cmd`对象提供了一个便捷方法：`CombinedOutput()`。该方法运行命令，将输出内容以一个字节切片返回便于后续处理。所以，上面获取输出的程序可以简化为：

```go
func main() {
  cmd := exec.Command("cal")
  output, err := cmd.CombinedOutput()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }

  fmt.Println(string(output))
}
```

So easy!

`CombinedOutput()`方法的实现很简单，先将标准输出和标准错误重定向到`*bytes.Buffer`对象，然后运行程序，最后返回该对象中的字节切片：

```go
func (c *Cmd) CombinedOutput() ([]byte, error) {
  if c.Stdout != nil {
    return nil, errors.New("exec: Stdout already set")
  }
  if c.Stderr != nil {
    return nil, errors.New("exec: Stderr already set")
  }
  var b bytes.Buffer
  c.Stdout = &b
  c.Stderr = &b
  err := c.Run()
  return b.Bytes(), err
}
```

`CombinedOutput`方法前几行判断表明，`Stdout`和`Stderr`必须是未设置状态。这其实很好理解，一般情况下，如果已经打算使用`CombinedOutput`方法获取输出内容，不会再自找麻烦地再去设置`Stdout`和`Stderr`字段了。

与`CombinedOutput`类似的还有`Output`方法，区别是`Output`只会返回运行命令产出的标准输出内容。

## 分别获取标准输出和标准错误

创建两个`*bytes.Buffer`对象，分别赋给`exec.Cmd`对象的`Stdout`和`Stderr`这两个字段，然后运行命令即可分别获取标准输出和标准错误。

```go
func main() {
  cmd := exec.Command("cal", "15", "2012")
  var stdout, stderr bytes.Buffer
  cmd.Stdout = &stdout
  cmd.Stderr = &stderr
  err := cmd.Run()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }

  fmt.Printf("output:\n%s\nerror:\n%s\n", stdout.String(), stderr.String())
}
```

## 标准输入

`exec.Cmd`对象有一个类型为`io.Reader`的字段`Stdin`。命令运行时会从这个`io.Reader`读取输入。先来看一个最简单的例子：

```go
func main() {
  cmd := exec.Command("cat")
  cmd.Stdin = bytes.NewBufferString("hello\nworld")
  cmd.Stdout = os.Stdout
  err := cmd.Run()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }
}
```

如果不带参数运行`cat`命令，则进入交互模式，`cat`按行读取输入，并且原样发送到输出。

再来看一个复杂点的例子。Go标准库中`compress/bzip2`包只提供解压方法，并没有压缩方法。我们可以利用Linux命令`bzip2`实现压缩。`bzip2`从标准输入中读取数据，将其压缩，并发送到标准输出。

```go
func bzipCompress(d []byte) ([]byte, error) {
  var out bytes.Buffer
  cmd := exec.Command("bzip2", "-c", "-9")
  cmd.Stdin = bytes.NewBuffer(d)
  cmd.Stdout = &out
  err := cmd.Run()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }

  return out.Bytes(), nil
}
```

参数`-c`表示压缩，`-9`表示压缩等级，9为最高。为了验证函数的正确性，写个简单的程序，先压缩"hello world"字符串，然后解压，看看是否能得到原来的字符串：

```go
func main() {
  data := []byte("hello world")
  compressed, _ := bzipCompress(data)
  r := bzip2.NewReader(bytes.NewBuffer(compressed))
  decompressed, _ := ioutil.ReadAll(r)
  fmt.Println(string(decompressed))
}
```

运行程序，输出"hello world"。

## 环境变量

环境变量可以在一定程度上微调程序的行为，当然这需要程序的支持。例如，设置`ENV=production`会抑制调试日志的输出。每个环境变量都是一个键值对。`exec.Cmd`对象中有一个类型为`[]string`的字段`Env`。我们可以通过修改它来达到控制命令运行时的环境变量的目的。

```go
package main

import (
  "fmt"
  "log"
  "os"
  "os/exec"
)

func main() {
  cmd := exec.Command("bash", "-c", "./test.sh")

  nameEnv := "NAME=darjun"
  ageEnv := "AGE=18"

  newEnv := append(os.Environ(), nameEnv, ageEnv)
  cmd.Env = newEnv

  out, err := cmd.CombinedOutput()
  if err != nil {
    log.Fatalf("cmd.Run() failed: %v\n", err)
  }

  fmt.Println(string(out))
}
```

上面代码获取系统的环境变量，然后又添加了两个环境变量`NAME`和`AGE`。最后使用`bash`运行脚本`test.sh`：

```shell
#!/bin/bash

echo $NAME
echo $AGE
echo $GOPATH
```

程序运行结果：

```bash
$ go run main.go 
darjun
18
D:\workspace\code\go
```

## 检查命令是否存在

一般在运行命令之前，我们通过希望能检查要运行的命令是否存在，如果存在则直接运行，否则提示用户安装此命令。`os/exec`包提供了函数`LookPath`可以获取命令所在目录，如果命令不存在，则返回一个error。

```go
func main() {
  path, err := exec.LookPath("ls")
  if err != nil {
    fmt.Printf("no cmd ls: %v\n", err)
  } else {
    fmt.Printf("find ls in path:%s\n", path)
  }

  path, err = exec.LookPath("not-exist")
  if err != nil {
    fmt.Printf("no cmd not-exist: %v\n", err)
  } else {
    fmt.Printf("find not-exist in path:%s\n", path)
  }
}
```

运行：

```bash
$ go run main.go 
find ls in path:C:\Program Files\Git\usr\bin\ls.exe
no cmd not-exist: exec: "not-exist": executable file not found in %PATH%
```

## 封装

执行外部命令的流程比较固定：

* 调用`exec.Command()`创建命令对象；
* 调用`Cmd.Run()`执行命令

如果要获取输出，需要调用`CombinedOutput/Output`之类的方法，或者手动创建`bytes.Buffer`对象并赋值给`exec.Cmd`的`Stdout`和`Stderr`字段。为了使用方便，我编写了一个包`goexec`。

接口如下：

```go
// 执行命令，丢弃标准输出和标准错误
func RunCommand(cmd string, arg []string, opts ...Option) error
// 执行命令，以[]byte类型返回输出
func CombinedOutput(cmd string, arg []string, opts ...Option) ([]byte, error)
// 执行命令，以string类型返回输出
func CombinedOutputString(cmd string, arg []string, opts ...Option) (string, error)
// 执行命令，以[]byte类型返回标准输出
func Output(cmd string, arg []string, opts ...Option) ([]byte, error)
// 执行命令，以string类型返回标准输出
func OutputString(cmd string, arg []string, opts ...Option) (string, error)
// 执行命令，以[]byte类型分别返回标准输出和标准错误
func SeparateOutput(cmd string, arg []string, opts ...Option) ([]byte, []byte, error)
// 执行命令，以string类型分别返回标准输出和标准错误
func SeparateOutputString(cmd string, arg []string, opts ...Option) (string, string, error)
```

相较于直接使用`os/exec`包，我倾向于一次函数调用就能获得结果。对输入、设置环境变量这些功能，我通过**Option模式**来提供支持。

```go
type Option func(*exec.Cmd)

func WithStdin(stdin io.Reader) Option {
  return func(c *exec.Cmd) {
    c.Stdin = stdin
  }
}

func Without(stdout io.Writer) Option {
  return func(c *exec.Cmd) {
    c.Stdout = stdout
  }
}

func WithStderr(stderr io.Writer) Option {
  return func(c *exec.Cmd) {
    c.Stderr = stderr
  }
}

func WithOutWriter(out io.Writer) Option {
  return func(c *exec.Cmd) {
    c.Stdout = out
    c.Stderr = out
  }
}

func WithEnv(key, value string) Option {
  return func(c *exec.Cmd) {
    c.Env = append(os.Environ(), fmt.Sprintf("%s=%s", key, value))
  }
}

func applyOptions(cmd *exec.Cmd, opts []Option) {
  for _, opt := range opts {
    opt(cmd)
  }
}
```

使用非常简单：

```go
func main() {
  fmt.Println(goexec.CombinedOutputString("cal", nil, goexec.WithEnv("LANG", "en_US.UTF-8")))
}
```

有一点我不太满意，为了使用Option模式，本来可以用可变参数来传递命令参数，现在只能用切片了，即使不需要指定参数，也必须要传入一个`nil`。暂时还没有想到比较优雅的解决方法。

## 总结

本文介绍了使用`os/exec`这个标准库调用外部命令的各种姿势。同时为了便于使用，我编写了一个goexec包封装对`os/exec`的调用。这个包目前for我自己使用是没有问题的，大家有其他需求可以提issue或者自己魔改😄。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. Advanced command execution in go with os/exec: [https://blog.kowalczyk.info/article/wOYk/advanced-command-execution-in-go-with-osexec.html](https://https://blog.kowalczyk.info/article/wOYk/advanced-command-execution-in-go-with-osexec.html)
2. goexec: [https://github.com/darjun/goexec](https://github.com/darjun/goexec)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)