---
layout:    post
title:    "Go 每日一库之 bubbletea"
subtitle: "每天学习一个 Go 库"
date:     2021-06-11T07:47:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/06/11/godailylib/bubbletea"
categories: [
  "Go"
]
---

## 简介

[`bubbletea`](https://github.com/charmbracelet/bubbletea)非常简单的、小巧的，非常方便地用来编写 TUI（terminal User Interface，控制台界面程序）程序。内置简单的事件处理机制，可以对外部事件做出响应，如键盘按键。一起来看下吧。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir bubbletea && cd bubbletea
$ go mod init github.com/darjun/go-daily-lib/bubbletea
```

安装`bubbletea`库：

```cmd
$ go get -u github.com/charmbracelet/bubbletea
```

`bubbletea`程序都需要有一个实现`bubbletea.Model`接口的类型：

```golang
type Model interface {
  Init() Cmd
  Update(Msg) (Model, Cmd)
  View() string
}
```

* `Init()`方法在程序启动时会立刻调用，它会做一些初始化工作，并返回一个`Cmd`告诉`bubbletea`要执行什么命令；
* `Update()`方法用来响应外部事件，返回一个修改后的模型，和需要`bubbletea`执行的命令；
* `View()`方法用于返回在控制台上显示的字符串。

下面我们来实现一个 Todo List。首先定义模型：

```golang
type model struct {
  todos    []string
  cursor   int
  selected map[int]struct{}
}
```

* `todos`：所有待完成事项；
* `cursor`：界面上光标位置；
* `selected`：已完成标识。

不需要任何初始化工作，实现一个空的`Init()`方法，并返回`nil`：

```golang
import (
  tea "github.com/charmbracelet/bubbletea"
)
func (m model) Init() tea.Cmd {
  return nil
}
```

我们需要响应按键事件，实现`Update()`方法。这个方法会在按键事件发生时调用，通过对参数`tea.Msg`类型断言，我们可以对按键事件进行相应的处理。

```golang
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
  switch msg := msg.(type) {
  case tea.KeyMsg:
    switch msg.String() {
    case "ctrl+c", "q":
      return m, tea.Quit

    case "up", "k":
      if m.cursor > 0 {
        m.cursor--
      }

    case "down", "j":
      if m.cursor < len(m.todos)-1 {
        m.cursor++
      }

    case "enter", " ":
      _, ok := m.selected[m.cursor]
      if ok {
        delete(m.selected, m.cursor)
      } else {
        m.selected[m.cursor] = struct{}{}
      }
    }
  }

  return m, nil
}
```

规定：
* `ctrl+c`或`q`：退出程序；
* `up`或`k`：向上移动光标；
* `down`或`j`：向下移动光标；
* `enter`或` `：切换光标处事项的完成状态。

处理`ctrl+c`或`q`按键时，返回一个特殊的`tea.Quit`，通知`bubbletea`需要退出程序。

最后实现`View()`方法，这个方法返回的字符串就是最终显示在控制台上的文本。我们可以按照自己想要的形式，根据模型数据拼装：

```golang
func (m model) View() string {
  s := "todo list:\n\n"

  for i, choice := range m.todos {
    cursor := " "
    if m.cursor == i {
      cursor = ">"
    }

    checked := " "
    if _, ok := m.selected[i]; ok {
      checked = "x"
    }

    s += fmt.Sprintf("%s [%s] %s\n", cursor, checked, choice)
  }

  s += "\nPress q to quit.\n"
  return s
}
```

光标所在位置用`>`标识，已完成的事项增加`x`标识。

模型类型定义好了之后，需要创建一个该模型的对象；

```golang
var initModel = model{
  todos:    []string{"cleanning", "wash clothes", "write a blog"},
  selected: make(map[int]struct{}),
}
```

为了让程序工作，我们还有创建一个`bubbletea`的运行程序，通过`bubbletea.NewProgram()`完成，然后调用这个对象的`Start()`方法执行：

```golang
func main() {
  cmd := tea.NewProgram(initModel)
  if err := cmd.Start(); err != nil {
    fmt.Println("start failed:", err)
    os.Exit(1)
  }
}
```

运行：

![](/img/in-post/godailylib/bubbletea1.wmv#center)

## 总结

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. bubbletea GitHub：[https://github.com/charmbracelet/bubbletea](https://https://github.com/charmbracelet/bubbletea)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~