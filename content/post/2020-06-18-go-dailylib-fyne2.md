---
layout:    post
title:    "使用 fyne 编写一个计算器"
subtitle: 	"学以致用"
date:		2020-06-18T22:41:55
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/06/18/godailylib/fyne2"
categories: [
	"Go"
]
---

## 简介

在上一篇文章中，我们介绍了一个 Go 的高颜值 GUI 库`fyne`。本文接着上一篇，介绍如何使用`fyne`编写一个简单的计算器程序。程序效果如下：

![](/img/in-post/godailylib/calc1.png#center)

## 控件布局

我们使用`widget.Entry`来显示输入的数字、运算符和运算结果。先创建一个`widget.Entry`对象，设置可显示多行：

```golang
display := widget.NewEntry()
display.MultiLine = true
```

其它数字和符号控件都用`widget.Button`来表示。按钮也分为两种，一种是没有特殊效果的，点击后直接在显示框中添加对应的字符即可。一种是有特殊效果的，例如清空显示框（`AC`）、进行计算（`=`）。中间三行按钮都是前一种，我们使用`GridLayout`来布局，每行显示 4 个：

```golang
digits := []string{
  "7", "8", "9", "×",
  "4", "5", "6", "-",
  "1", "2", "3", "+",
}
var digitBtns []fyne.CanvasObject
for _, val := range digits {
  digitBtns = append(digitBtns, widget.NewButton(val, input(display, val)))
}
digitContainer := fyne.NewContainerWithLayout(
  layout.NewGridLayout(4),
  digitBtns...)
```

`input`回调后面介绍。

第一行有 3 个具有特殊效果的按钮：

* `AC`：清空显示框；
* `+/-`：切换正负号；
* `%`将数字变为百分数，即除以 100。

外加一个除法按钮。这一行同样使用`GridLayout`布局：

```golang
clearBtn := widget.NewButton("AC", clear(display))
signBtn := widget.NewButton("+/-", sign(display))
percentBtn := widget.NewButton("%", percent(display, "%"))
divideBtn := widget.NewButton("÷", input(display, "÷"))
clearContainer := fyne.NewContainerWithLayout(
  layout.NewGridLayout(4),
  clearBtn,
  signBtn,
  percentBtn,
  divideBtn,
)
```

几个回调处理后面统一介绍。

最后一行由于`0`这个按钮宽度是其它按钮的 2 倍。我们先使用`GridLayout`布局，将这一行平均分成两`Grid`（即每行 2 个控件）。按钮`0`独占一个`Grid`，由于`GridLayout`布局中每个`Grid`大小相同，故按钮`0`有整个行一半的宽度。后一个`Grid`，按钮`.`和`=`平分，同样使用一个`GridLayout`达到这个效果：

```golang
zeroBtn := widget.NewButton("0", input(display, "0"))
dotBtn := widget.NewButton(".", input(display, "."))
equalBtn := widget.NewButton("=", equals(display))
zeroContainer := fyne.NewContainerWithLayout(
  layout.NewGridLayout(2),
  zeroBtn,
  fyne.NewContainerWithLayout(
    layout.NewGridLayout(2),
    dotBtn,
    equalBtn,
  ),
)
```

最后我们将所有部分用垂直的`BoxLayout`组合到一起：

```golang
container := fyne.NewContainerWithLayout(
  layout.NewVBoxLayout(),
  display,
  clearContainer,
  digitContainer,
  zeroContainer,
  copyright,
)
```

在实际开发中，一般会组合使用多种布局实现界面效果。

## 按钮响应

清空按钮响应比较简单，直接将显示框的`Text`设置为空即可：

```golang
func clear(display *widget.Entry) func() {
  return func() {
    display.Text = ""
    display.Refresh()
  }
}
```

注意，要调用`Entry.Refresh()`方法刷新界面。由于按钮响应都是对应显示框进行操作，所以都需要传入该对象。

我们设计在显示框中显示两行，第一行是上次计算的表达式，第二行是本次的。切换正负号在本次只输入一个数字时将该数字的正负号进行切换：

```golang
func sign(display *widget.Entry) func() {
  return func() {
    lines := strings.Split(display.Text, "\n")
    if len(lines) == 0 {
      return
    }

    line := lines[len(lines)-1]
    value, err := strconv.ParseInt(line, 10, 64)
    if err != nil {
      return
    }
    lines[len(lines)-1] = strconv.FormatInt(-value, 10)
    display.Text = strings.Join(lines, "\n")
  }
}
```

输入回调我们只做了简单的处理——将对应字符串拼接到显示框中：

```golang
func input(display *widget.Entry, value string) func() {
  return func() {
    display.Text += value
    display.Refresh()
  }
}
```

计算表达式的函数也很简单，我这里使用了`govaluate`库（可以看我之前的文章）：

```golang
func equals(display *widget.Entry) func() {
  return func() {
    lines := strings.Split(display.Text, "\n")
    if len(lines) == 0 {
      return
    }

    line := lines[len(lines)-1]
    line = strings.Trim(line, "+÷×")
    exprLine := strings.Replace(line, "÷", "/", -1)
    exprLine = strings.Replace(exprLine, "×", "*", -1)
    expr, _ := govaluate.NewEvaluableExpression(exprLine)
    result, _ := expr.Evaluate(nil)
    line += "=\n"
    line += fmt.Sprint(result)
    display.Text = line
    display.Refresh()
  }
}
```

注意，这里做了一点容错，把前后多余的运算符去掉了。另外，我们前面为了显示，使用了`÷`表示除法符号，`×`表示乘法符号。要使用`govaluate`，必须将它们分别替换为`/`和`*`。

至此计算器就编写完成了，下面我们介绍如何打包。

## 打包

准备好一张图片资源作为图标，放到项目目录下：

![](/img/in-post/godailylib/calc2.jpg#center)

打包：

```cmd
$ fyne package -os windows -icon icon.jpg
```

在同目录下生成了一个`.exe`文件和一个`.syso`文件。现在可直接点击`calculator.exe`运行程序，没有其他依赖。

## 总结

本文介绍如何使用`fyne`编写一个简单的计算器程序，主要介绍如何组合使用多种布局。当然计算器功能和错误处理还不完善，而且实现偏过程式编程，感兴趣的可自行完善。完整代码在`fyne/calculator`中。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. fyne GitHub：[https://github.com/fyne-io/fyne](https://github.com/fyne-io/fyne)
2. fyne 官网：[https://fyne.io/](https://fyne.io/)
3. fyne 官方入门教程：[https://developer.fyne.io/tour/introduction/hello.html](https://developer.fyne.io/tour/introduction/hello.html)
4. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)