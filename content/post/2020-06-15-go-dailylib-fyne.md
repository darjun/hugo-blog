---
layout:    post
title:    "Go 每日一库之 fyne"
subtitle: 	"每天学习一个 Go 库"
date:		2020-06-15T23:15:55
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/06/15/godailylib/fyne"
categories: [
	"Go"
]
---

## 简介

Go 语言生态中，GUI 一直是短板，更别说跨平台的 GUI 了。[`fyne`](https://fyne.io/)向前迈了一大步。`fyne` 是 Go 语言编写的**跨平台的** UI 库，它可以很方便地移植到手机设备上。`fyne`使用上非常简单，同时它还提供`fyne`命令打包静态资源和应用程序。我们先简单介绍基本控件和布局，然后介绍如何发布一个`fyne`应用程序。

## 快速使用

本文代码使用 Go Modules。

先初始化：

```cmd
$ mkdir fyne && cd fyne
$ go mod init github.com/darjun/go-daily-lib/fyne
```

由于`fyne`包含一些 C/C++ 的代码，所以需要`gcc`编译工具。在 Linux/Mac OSX 上，`gcc`基本是标配，在 windows 上我们有 3 种方式安装`gcc`工具链：

* MSYS2 + MingW-w64：[https://www.msys2.org/](https://www.msys2.org/)；
* TDM-GCC：[https://jmeubank.github.io/tdm-gcc/download/](https://jmeubank.github.io/tdm-gcc/download/)；
* Cygwin：[https://www.cygwin.com/](https://www.cygwin.com/)。

本文选择`TDM-GCC`的方式安装。到[https://jmeubank.github.io/tdm-gcc/download/](https://jmeubank.github.io/tdm-gcc/download/)下载安装程序并安装。正常情况下安装程序会自动设置`PATH`路径。打开命令行，键入`gcc -v`。如果正常输出版本信息，说明安装成功且环境变量设置正确。

安装`fyne`：

```cmd
$ go get -u fyne.io/fyne
```

到此准备工作已经完成，我们开始编码。按照惯例，先以**Hello, World**程序开始：

```golang
package main

import (
  "fyne.io/fyne"
  "fyne.io/fyne/app"
  "fyne.io/fyne/widget"
)

func main() {
  myApp := app.New()

  myWin := myApp.NewWindow("Hello")
  myWin.SetContent(widget.NewLabel("Hello Fyne!"))
  myWin.Resize(fyne.NewSize(200, 200))
  myWin.ShowAndRun()
}
```

运行结果如下：

![](/img/in-post/godailylib/fyne1.png#center)

`fyne`的使用很简单。每个`fyne`程序都包括两个部分，一个是应用程序对象`myApp`，通过`app.New()`创建。另一个是窗口对象，通过应用程序对象`myApp`来创建`myApp.NewWindow("Hello")`。`myApp.NewWindow()`方法中传入的字符串就是窗口标题。

`fyne`提供了很多常用的组件，通过`widget.NewXXX()`创建（`XXX`为组件名）。上面示例中，我们创建了一个`Label`控件，然后设置到窗口中。最后，调用`myWin.ShowAndRun()`开始运行程序。实际上`myWin.ShowAndRun()`等价于

```golang
myWin.Show()
myApp.Run()
```

`myWin.Show()`显示窗口，`myApp.Run()`开启事件循环。

注意一点，`fyne`默认窗口大小是根据内容的宽高来设置的。上面我们调用`myWin.Resize()`手动设置了大小。否则窗口只能放下字符串`Hello Fyne!`。


## `fyne`包结构划分

`fyne`将功能划分到多个子包中：

* `fyne.io/fyne`：提供所有`fyne`应用程序代码共用的基础定义，包括数据类型和接口；
* `fyne.io/fyne/app`：提供创建应用程序的 API；
* `fyne.io/fyne/canvas`：提供`Fyne`使用的绘制 API；
* `fyne.io/fyne/dialog`：提供对话框组件；
* `fyne.io/fyne/layout`：提供多种界面布局；
* `fyne.io/fyne/widget`：提供多种组件，`fyne`所有的窗体控件和交互元素都在这个子包中。

## Canvas

在`fyne`应用程序中，所有显示元素都是绘制在画布（`Canvas`）上的。这些元素都是画布对象（`CanvasObject`）。调用`Canvas.SetContent()`方法可设置画布内容。`Canvas`一般和布局（`Layout`）容器（`Container`）一起使用。`canvas`子包中提供了一些基础的画布对象：

```golang
package main

import (
  "image/color"
  "math/rand"

  "fyne.io/fyne"
  "fyne.io/fyne/app"
  "fyne.io/fyne/canvas"
  "fyne.io/fyne/layout"
  "fyne.io/fyne/theme"
)

func main() {
  a := app.New()
  w := a.NewWindow("Canvas")

  rect := canvas.NewRectangle(color.White)

  text := canvas.NewText("Hello Text", color.White)
  text.Alignment = fyne.TextAlignTrailing
  text.TextStyle = fyne.TextStyle{Italic: true}

  line := canvas.NewLine(color.White)
  line.StrokeWidth = 5

  circle := canvas.NewCircle(color.White)
  circle.StrokeColor = color.Gray{0x99}
  circle.StrokeWidth = 5

  image := canvas.NewImageFromResource(theme.FyneLogo())
  image.FillMode = canvas.ImageFillOriginal

  raster := canvas.NewRasterWithPixels(
    func(_, _, w, h int) color.Color {
      return color.RGBA{uint8(rand.Intn(255)),
        uint8(rand.Intn(255)),
        uint8(rand.Intn(255)), 0xff}
    },
  )

  gradient := canvas.NewHorizontalGradient(color.White, color.Transparent)

  container := fyne.NewContainerWithLayout(
    layout.NewGridWrapLayout(fyne.NewSize(150, 150)),
    rect, text, line, circle, image, raster, gradient))
  w.SetContent(container)
  w.ShowAndRun()
}
```

程序运行结果如下：

![](/img/in-post/godailylib/fyne2.png#center)

`canvas.Rectangle`是最简单的画布对象了，通过`canvas.NewRectangle()`创建，传入填充颜色。

`canvas.Text`是显示文本的画布对象，通过`canvas.NewText()`创建，传入文本字符串和颜色。该对象可设置对齐方式和字体样式。对齐方式通过设置`Text`对象的`Alignment`字段值，取值有：

* `TextAlignLeading`：左对齐；
* `TextAlignCenter`：中间对齐；
* `TextAlignTrailing`：右对齐。

字体样式通过设置`Text`对象的`TextStyle`字段值，`TextStyle`是一个结构体：

```golang
type TextStyle struct {
  Bold      bool
  Italic    bool
  Monospace bool
}
```

对应字段设置为`true`将显示对应的样式：

* `Bold`：粗体；
* `Italic`：斜体；
* `Monospace`：系统等宽字体。

我们还可以通过设置环境变量`FYNE_FONT`为一个`.ttf`文件从而使用外部字体。

`canvas.Line`是线段，通过`canvas.NewLine()`创建，传入颜色。可以通过`line.StrokeWidth`设置线段宽度。默认情况下，线段是从父控件或画布的**左上角**到**右下角**的。可通过`line.Move()`和`line.Resize()`修改位置。

`canvas.Circle`是圆形，通过`canvas.NewCircle()`创建，传入颜色。另外通过`StrokeColor`和`StrokeWidth`设置圆形边框的颜色和宽度。

`canvas.Image`是图像，可以通过已加载的程序资源创建（`canvas.NewImageFromResource()`），传入资源对象。或通过文件路径创建（`canvas.NewImageFromFile()`），传入文件路径。或通过已构造的`image.Image`对象创建（`canvas.NewImageFromImage()`）。可以通过`FillMode`设置图像的填充模式：

* `ImageFillStretch`：拉伸，填满空间；
* `ImageFillContain`：保持宽高比；
* `ImageFillOriginal`：保持原始大小，不缩放。

下面程序演示了这 3 种创建图像的方式：

```golang
package main

import (
  "image"
  "image/color"

  "fyne.io/fyne"
  "fyne.io/fyne/app"
  "fyne.io/fyne/canvas"
  "fyne.io/fyne/layout"
  "fyne.io/fyne/theme"
)

func main() {
  a := app.New()
  w := a.NewWindow("Hello")

  img1 := canvas.NewImageFromResource(theme.FyneLogo())
  img1.FillMode = canvas.ImageFillOriginal

  img2 := canvas.NewImageFromFile("./luffy.jpg")
  img2.FillMode = canvas.ImageFillOriginal

  image := image.NewAlpha(image.Rectangle{image.Point{0, 0}, image.Point{100, 100}})
  for i := 0; i < 100; i++ {
    for j := 0; j < 100; j++ {
      image.Set(i, j, color.Alpha{uint8(i % 256)})
    }
  }
  img3 := canvas.NewImageFromImage(image)
  img3.FillMode = canvas.ImageFillOriginal

  container := fyne.NewContainerWithLayout(
    layout.NewGridWrapLayout(fyne.NewSize(150, 150)),
    img1, img2, img3)
  w.SetContent(container)
  w.ShowAndRun()
}
```

`theme.FyneLogo()`是 Fyne 图标资源，`luffy.jpg`是磁盘中的文件，最后创建一个`image.Image`，从中生成`canvas.Image`。

![](/img/in-post/godailylib/fyne3.png#center)

最后一种是梯度渐变效果，有两种类型`canvas.LinearGradient`（线性渐变）和`canvas.RadialGradient`（放射渐变），指从一种颜色渐变到另一种颜色。线性渐变又分为两种**水平线性渐变**和**垂直线性渐变**，分别通过`canvas.NewHorizontalGradient()`和`canvas.NewVerticalGradient()`创建。放射渐变通过`canvas.NewRadialGradient()`创建。我们在上面的示例中已经看到了水平线性渐变的效果，接下来一起看看放射渐变的效果：

```golang
func main() {
  a := app.New()
  w := a.NewWindow("Canvas")

  gradient := canvas.NewRadialGradient(color.White, color.Transparent)
  w.SetContent(gradient)
  w.Resize(fyne.NewSize(200, 200))
  w.ShowAndRun()
}
```

运行效果如下：

![](/img/in-post/godailylib/fyne4.png#center)

放射效果就是从中心向周围渐变。

## Widget

窗体控件是一个`Fyne`应用程序的主要组成部分。它们能适配当前的主题，并且处理与用户的交互。

### Label

标签（`Label`）是最简单的一个控件了，用于显示字符串。它有点类似于`canvas.Text`，不同之处在于`Label`可以处理简单的格式化，例如`\n`：

```golang
func main() {
  myApp := app.New()
  myWin := myApp.NewWindow("Label")

  l1 := widget.NewLabel("Name")
  l2 := widget.NewLabel("da\njun")

  container := fyne.NewContainerWithLayout(layout.NewVBoxLayout(), l1, l2)
  myWin.SetContent(container)
  myWin.Resize(fyne.NewSize(150, 150))
  myWin.ShowAndRun()
}
```

第二个`widget.Label`中`\n`后面的内容会在下一行渲染：

![](/img/in-post/godailylib/fyne5.png#center)

### Button

按钮（`Button`）控件让用户点击，给用户反馈。`Button`可以包含文本，图标或两者皆有。调用`widget.NewButton()`创建一个默认的文本按钮，传入文本和一个无参的回调函数。带图标的按钮需要调用`widget.NewButtonWithIcon()`，传入文本和回调参数，还需要一个`fyne.Resource`类型的图标资源：

```golang
func main() {
  myApp := app.New()
  myWin := myApp.NewWindow("Button")

  btn1 := widget.NewButton("text button", func() {
    fmt.Println("text button clicked")
  })

  btn2 := widget.NewButtonWithIcon("icon", theme.HomeIcon(), func() {
    fmt.Println("icon button clicked")
  })

  container := fyne.NewContainerWithLayout(layout.NewVBoxLayout(), btn1, btn2)
  myWin.SetContent(container)
  myWin.Resize(fyne.NewSize(150, 50))
  myWin.ShowAndRun()
}
```

上面创建了一个文本按钮和一个图标按钮，`theme`子包中包含一些默认的图标资源，也可以加载外部的图标。运行：

![](/img/in-post/godailylib/fyne6.png#center)

点击按钮，对应的回调就会被调用，试试看！

### Box

盒子控件（`Box`）就是一个简单的水平或垂直的容器。在内部，`Box`对子控件采用盒状布局（`Box Layout`），详见后文布局。我们可以通过传入控件对象给`widget.NewHBox()`或`widget.NewVBox()`创建盒子。或者调用已经创建好的`widget.Box`对象的`Append()`或`Prepend()`向盒子中添加控件。前者在尾部追加，后者在头部添加。

```golang
func main() {
  myApp := app.New()
  myWin := myApp.NewWindow("Box")

  content := widget.NewVBox(
    widget.NewLabel("The top row of VBox"),
    widget.NewHBox(
      widget.NewLabel("Label 1"),
      widget.NewLabel("Label 2"),
    ),
  )
  content.Append(widget.NewButton("Append", func() {
    content.Append(widget.NewLabel("Appended"))
  }))
  content.Append(widget.NewButton("Prepend", func() {
    content.Prepend(widget.NewLabel("Prepended"))
  }))

  myWin.SetContent(content)
  myWin.Resize(fyne.NewSize(150, 150))
  myWin.ShowAndRun()
}
```

我们甚至可以嵌套`widget.Box`控件，这样就可以实现比较灵活的布局。上面的代码中添加了两个按钮，点击时分别在尾部和头部添加一个`Label`：

![](/img/in-post/godailylib/fyne7.gif#center)

### Entry

输入框（`Entry`）控件用于给用户输入简单的文本内容。调用`widget.NewEntry()`即可创建一个输入框控件。我们一般保存输入框控件的引用，以便访问其`Text`字段来获取内容。注册`OnChanged`回调函数。每当内容有修改时，`OnChanged`就会被调用。我们可以调用`SetReadOnly(true)`设置输入框的只读属性。方法`SetPlaceHolder()`用来设置占位字符串，设置字段`Multiline`让输入框接受多行文本。另外，我们可以使用`NewPasswordEntry()`创建一个密码输入框，输入的文本不会以明文显示。

```golang
func main() {
  myApp := app.New()
  myWin := myApp.NewWindow("Entry")

  nameEntry := widget.NewEntry()
  nameEntry.SetPlaceHolder("input name")
  nameEntry.OnChanged = func(content string) {
    fmt.Println("name:", nameEntry.Text, "entered")
  }

  passEntry := widget.NewPasswordEntry()
  passEntry.SetPlaceHolder("input password")

  nameBox := widget.NewHBox(widget.NewLabel("Name"), layout.NewSpacer(), nameEntry)
  passwordBox := widget.NewHBox(widget.NewLabel("Password"), layout.NewSpacer(), passEntry)

  loginBtn := widget.NewButton("Login", func() {
    fmt.Println("name:", nameEntry.Text, "password:", passEntry.Text, "login in")
  })

  multiEntry := widget.NewEntry()
  multiEntry.SetPlaceHolder("please enter\nyour description")
  multiEntry.MultiLine = true

  content := widget.NewVBox(nameBox, passwordBox, loginBtn, multiEntry)
  myWin.SetContent(content)
  myWin.ShowAndRun()
}
```

这里我们实现了一个简单的登录界面：

![](/img/in-post/godailylib/fyne7.png#center)

### `Checkbox/Radio/Select`

`CheckBox`是简单的选择框，每个选择是独立的，例如爱好可以是足球、篮球，也可以都是。创建方法`widget.NewCheck()`，传入选项字符串（足球，篮球）和回调函数。回调函数接受一个`bool`类型的参数，表示该选项是否选中。

`Radio`是单选框，每个组内只能选择一个，例如性别，只能是男或女（？）。创建方法`widget.NewRadio()`，传入字符串切片和回调函数作为参数。回调函数接受一个字符串参数，表示选中的选项。也可以使用`Selected`字段读取选中的选项。

`Select`是下拉选择框，点击时显示一个下拉菜单，点击选择。选项非常多的时候，比较适合用`Select`。创建方法`widget.NewSelect()`，参数与`NewRadio()`完全相同。

```golang
func main() {
  myApp := app.New()
  myWin := myApp.NewWindow("Choices")

  nameEntry := widget.NewEntry()
  nameEntry.SetPlaceHolder("input name")

  passEntry := widget.NewPasswordEntry()
  passEntry.SetPlaceHolder("input password")

  repeatPassEntry := widget.NewPasswordEntry()
  repeatPassEntry.SetPlaceHolder("repeat password")

  nameBox := widget.NewHBox(widget.NewLabel("Name"), layout.NewSpacer(), nameEntry)
  passwordBox := widget.NewHBox(widget.NewLabel("Password"), layout.NewSpacer(), passEntry)
  repeatPasswordBox := widget.NewHBox(widget.NewLabel("Repeat Password"), layout.NewSpacer(), repeatPassEntry)

  sexRadio := widget.NewRadio([]string{"male", "female", "unknown"}, func(value string) {
    fmt.Println("sex:", value)
  })
  sexBox := widget.NewHBox(widget.NewLabel("Sex"), sexRadio)

  football := widget.NewCheck("football", func(value bool) {
    fmt.Println("football:", value)
  })
  basketball := widget.NewCheck("basketball", func(value bool) {
    fmt.Println("basketball:", value)
  })
  pingpong := widget.NewCheck("pingpong", func(value bool) {
    fmt.Println("pingpong:", value)
  })
  hobbyBox := widget.NewHBox(widget.NewLabel("Hobby"), football, basketball, pingpong)

  provinceSelect := widget.NewSelect([]string{"anhui", "zhejiang", "shanghai"}, func(value string) {
    fmt.Println("province:", value)
  })
  provinceBox := widget.NewHBox(widget.NewLabel("Province"), layout.NewSpacer(), provinceSelect)

  registerBtn := widget.NewButton("Register", func() {
    fmt.Println("name:", nameEntry.Text, "password:", passEntry.Text, "register")
  })

  content := widget.NewVBox(nameBox, passwordBox, repeatPasswordBox,
    sexBox, hobbyBox, provinceBox, registerBtn)
  myWin.SetContent(content)
  myWin.ShowAndRun()
}
```

这里我们实现了一个简单的注册界面：

![](/img/in-post/godailylib/fyne8.png#center)

### Form

表单控件（`Form`）用于对很多`Label`和输入控件进行布局。如果指定了`OnSubmit`或`OnCancel`函数，表单控件会自动添加对应的`Button`按钮。我们调用`widget.NewForm()`传入一个`widget.FormItem`切片创建`Form`控件。每一项中一个字符串作为`Label`的文本，一个控件对象。创建好`Form`对象之后还能调用其`Append(label, widget)`方法添加控件。

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Form")

  nameEntry := widget.NewEntry()
  passEntry := widget.NewPasswordEntry()

  form := widget.NewForm(
    &widget.FormItem{"Name", nameEntry},
    &widget.FormItem{"Pass", passEntry},
  )
  form.OnSubmit = func() {
    fmt.Println("name:", nameEntry.Text, "pass:", passEntry.Text, "login in")
  }
  form.OnCancel = func() {
    fmt.Println("login canceled")
  }

  myWindow.SetContent(form)
  myWindow.Resize(fyne.NewSize(150, 150))
  myWindow.ShowAndRun()
}
```

使用`Form`能大大简化表单的构建，我们使用`Form`重新编写了上面的登录界面：

![](/img/in-post/godailylib/fyne9.png#center)

注意`Submit`和`Cancel`按钮是自动生成的！

### ProgressBar

进度条控件（`ProgressBar`）用来表示任务的进度，例如文件下载的进度。创建方法`widget.NewProgressBar()`，默认最小值为`0.0`，最大值为`1.1`，可通过`Min/Max`字段设置。调用`SetValue()`方法来控制进度。还有一种进度条是循环动画，它表示有任务在进行中，并不能表示具体的完成情况。

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("ProgressBar")

  bar1 := widget.NewProgressBar()
  bar1.Min = 0
  bar1.Max = 100
  bar2 := widget.NewProgressBarInfinite()

  go func() {
    for i := 0; i <= 100; i ++ {
      time.Sleep(time.Millisecond * 500)
      bar1.SetValue(float64(i))
    }
  }()

  content := widget.NewVBox(bar1, bar2)
  myWindow.SetContent(content)
  myWindow.Resize(fyne.NewSize(150, 150))
  myWindow.ShowAndRun()
}
```

在另一个 goroutine 中更新进度。效果如下：

![](/img/in-post/godailylib/fyne10.png#center)

### TabContainer

标签容器（`TabContainer`）允许用户在不同的内容面板之间切换。标签可以是文本或图标。创建方法`widget.NewTabContainer()`，传入`widget.TabItem`作为参数。`widget.TabItem`可通过`widget.NewTabItem(label, widget)`创建。标签还可以设置位置：

* `TabLocationBottom`：显示在底部；
* `TabLocationLeading`：显示在顶部左边；
* `TabLocationTrailing`：显示在顶部右边。

看示例：

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("TabContainer")

  nameLabel := widget.NewLabel("Name: dajun")
  sexLabel := widget.NewLabel("Sex: male")
  ageLabel := widget.NewLabel("Age: 18")
  addressLabel := widget.NewLabel("Province: shanghai")
  addressLabel.Hide()
  profile := widget.NewVBox(nameLabel, sexLabel, ageLabel, addressLabel)

  musicRadio := widget.NewRadio([]string{"on", "off"}, func(string) {})
  showAddressCheck := widget.NewCheck("show address?", func(value bool) {
    if !value {
      addressLabel.Hide()
    } else {
      addressLabel.Show()
    }
  })
  memberTypeSelect := widget.NewSelect([]string{"junior", "senior", "admin"}, func(string) {})

  setting := widget.NewForm(
    &widget.FormItem{"music", musicRadio},
    &widget.FormItem{"check", showAddressCheck},
    &widget.FormItem{"member type", memberTypeSelect},
  )

  tabs := widget.NewTabContainer(
    widget.NewTabItem("Profile", profile),
    widget.NewTabItem("Setting", setting),
  )

  myWindow.SetContent(tabs)
  myWindow.Resize(fyne.NewSize(200, 200))
  myWindow.ShowAndRun()
}
```

上面代码编写了一个简单的个人信息面板和设置面板，点击`show address？`可切换地址信息是否显示：

![](/img/in-post/godailylib/fyne11.png#center)
![](/img/in-post/godailylib/fyne12.png#center)

### Toolbar

工具栏（`Toolbar`）是很多 GUI 应用程序必备的部分。工具栏将常用命令用图标的方式很形象地展示出来，方便使用。创建方法`widget.NewToolbar()`，传入多个`widget.ToolbarItem`作为参数。最常使用的`ToolbarItem`有命令（`Action`）、分隔符（`Separator`）和空白（`Spacer`），分别通过`widget.NewToolbarItemAction(resource, callback)`/`widget.NewToolbarSeparator()`/`widget.NewToolbarSpacer()`创建。命令需要指定回调，点击时触发。

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Toolbar")

  toolbar := widget.NewToolbar(
    widget.NewToolbarAction(theme.DocumentCreateIcon(), func() {
      fmt.Println("New document")
    }),
    widget.NewToolbarSeparator(),
    widget.NewToolbarAction(theme.ContentCutIcon(), func() {
      fmt.Println("Cut")
    }),
    widget.NewToolbarAction(theme.ContentCopyIcon(), func() {
      fmt.Println("Copy")
    }),
    widget.NewToolbarAction(theme.ContentPasteIcon(), func() {
      fmt.Println("Paste")
    }),
    widget.NewToolbarSpacer(),
    widget.NewToolbarAction(theme.HelpIcon(), func() {
      log.Println("Display help")
    }),
  )

  content := fyne.NewContainerWithLayout(
    layout.NewBorderLayout(toolbar, nil, nil, nil),
    toolbar, widget.NewLabel(`Lorem ipsum dolor, 
    sit amet consectetur adipisicing elit.
    Quidem consectetur ipsam nesciunt,
    quasi sint expedita minus aut,
    porro iusto magnam ducimus voluptates cum vitae.
    Vero adipisci earum iure consequatur quidem.`),
  )
  myWindow.SetContent(content)
  myWindow.ShowAndRun()
}
```

工具栏一般使用`BorderLayout`，将工具栏放在其他任何控件上面，布局后文会详述。运行：

![](/img/in-post/godailylib/fyne13.png#center)

### 扩展控件

标准的 Fyne 控件提供了最小的功能集和定制化以适应大部分的应用场景。有些时候，我们需要更高级的功能。除了自己编写控件外，我们还可以扩展现有的控件。例如，我们希望图标控件`widget.Icon`能响应鼠标左键、右键和双击。首先编写一个**构造函数**，调用`ExtendBaseWidget()`方法获得基础的控件功能：

```golang
type tappableIcon struct {
  widget.Icon
}

func newTappableIcon(res fyne.Resource) *tappableIcon {
  icon := &tappableIcon{}
  icon.ExtendBaseWidget(icon)
  icon.SetResource(res)

  return icon
}
```

然后实现相关的接口：

```golang
// src/fyne.io/fyne/canvasobject.go
// 鼠标左键
type Tappable interface {
  Tapped(*PointEvent)
}

// 鼠标右键或长按
type SecondaryTappable interface {
  TappedSecondary(*PointEvent)
}

// 双击
type DoubleTappable interface {
  DoubleTapped(*PointEvent)
}
```

接口实现：

```golang
func (t *tappableIcon) Tapped(e *fyne.PointEvent) {
  log.Println("I have been left tapped at", e)
}

func (t *tappableIcon) TappedSecondary(e *fyne.PointEvent) {
  log.Println("I have been right tapped at", e)
}

func (t *tappableIcon) DoubleTapped(e *fyne.PointEvent) {
  log.Println("I have been double tapped at", e)
}
```

最后使用：

```golang
func main() {
  a := app.New()
  w := a.NewWindow("Tappable")
  w.SetContent(newTappableIcon(theme.FyneLogo()))
  w.Resize(fyne.NewSize(200, 200))
  w.ShowAndRun()
}
```

运行，点击图标控制台有相应输出：

![](/img/in-post/godailylib/fyne23.png#center)

```cmd
2020/06/18 06:44:02 I have been left tapped at &{{110 97} {106 93}}
2020/06/18 06:44:03 I have been left tapped at &{{110 97} {106 93}}
2020/06/18 06:44:05 I have been right tapped at &{{88 102} {84 98}}
2020/06/18 06:44:06 I have been right tapped at &{{88 102} {84 98}}
2020/06/18 06:44:06 I have been left tapped at &{{88 101} {84 97}}
2020/06/18 06:44:07 I have been double tapped at &{{88 101} {84 97}}
```

输出的`fyne.PointEvent`中有绝对位置（对于窗口左上角）和相对位置（对于容器左上角）。

## Layout

布局（`Layout`）就是控件如何在界面上显示，如何排列的。要想界面好看，布局是必须要掌握的。几乎所有的 GUI 框架都提供了布局或类似的接口。实际上，在前面的示例中我们已经在`fyne.NewContainerWithLayout()`函数中使用了布局。

### BoxLayout

盒状布局（`BoxLayout`）是最常使用的一个布局。它将控件都排在一行或一列。在`fyne`中，我们可以通过`layout.NewHBoxLayout()`创建一个水平盒装布局，通过`layout.NewVBoxLayout()`创建一个垂直盒装布局。水平布局中的控件都排列在一行中，每个控件的宽度等于其内容的最小宽度（`MinSize().Width`），它们都拥有相同的高度，即所有控件的最大高度（`MinSize().Height`）。

垂直布局中的控件都排列在一列中，每个控件的高度等于其内容的最小高度，它们都拥有相同的宽度，即所有控件的最大宽度。

一般地，在`BoxLayout`中使用`layout.NewSpacer()`辅助布局，它会占满剩余的空间。对于水平盒状布局来说，第一个控件前添加一个`layout.NewSpacer()`，所有控件右对齐。最后一个控件后添加一个`layout.NewSpacer()`，所有控件左对齐。前后都有，那么控件中间对齐。如果在中间有添加一个`layout.NewSpacer()`，那么其它控件两边对齐。

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Box Layout")

  hcontainer1 := fyne.NewContainerWithLayout(layout.NewHBoxLayout(),
    canvas.NewText("left", color.White),
    canvas.NewText("right", color.White))

  // 左对齐
  hcontainer2 := fyne.NewContainerWithLayout(layout.NewHBoxLayout(),
    layout.NewSpacer(),
    canvas.NewText("left", color.White),
    canvas.NewText("right", color.White))

  // 右对齐
  hcontainer3 := fyne.NewContainerWithLayout(layout.NewHBoxLayout(),
    canvas.NewText("left", color.White),
    canvas.NewText("right", color.White),
    layout.NewSpacer())

  // 中间对齐
  hcontainer4 := fyne.NewContainerWithLayout(layout.NewHBoxLayout(),
    layout.NewSpacer(),
    canvas.NewText("left", color.White),
    canvas.NewText("right", color.White),
    layout.NewSpacer())

  // 两边对齐
  hcontainer5 := fyne.NewContainerWithLayout(layout.NewHBoxLayout(),
    canvas.NewText("left", color.White),
    layout.NewSpacer(),
    canvas.NewText("right", color.White))

  myWindow.SetContent(fyne.NewContainerWithLayout(layout.NewVBoxLayout(),
    hcontainer1, hcontainer2, hcontainer3, hcontainer4, hcontainer5))
  myWindow.Resize(fyne.NewSize(200, 200))
  myWindow.ShowAndRun()
}
```

运行效果：

![](/img/in-post/godailylib/fyne14.png#center)

### GridLayout

格子布局（`GridLayout`）每一行有固定的列，添加的控件数量超过这个值时，后面的控件将会在新的行显示。创建方法`layout.NewGridLayout(cols)`，传入每行的列数。

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Grid Layout")

  img1 := canvas.NewImageFromResource(theme.FyneLogo())
  img2 := canvas.NewImageFromResource(theme.FyneLogo())
  img3 := canvas.NewImageFromResource(theme.FyneLogo())
  myWindow.SetContent(fyne.NewContainerWithLayout(layout.NewGridLayout(2),
    img1, img2, img3))
  myWindow.Resize(fyne.NewSize(300, 300))
  myWindow.ShowAndRun()
}
```

运行效果：

![](/img/in-post/godailylib/fyne15.png#center)

该布局有个优势，我们缩放界面时，控件会自动调整大小。试试看~

### GridWrapLayout

`GridWrapLayout`是`GridLayout`的扩展。`GridWrapLayout`创建时会指定一个初始`size`，这个`size`会应用到所有的子控件上，每个子控件都保持这个`size`。初始，每行一个控件。如果界面大小变化了，这些子控件会重新排列。例如宽度翻倍了，那么一行就可以排两个控件了。有点像流动布局：

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Grid Wrap Layout")

  img1 := canvas.NewImageFromResource(theme.FyneLogo())
  img2 := canvas.NewImageFromResource(theme.FyneLogo())
  img3 := canvas.NewImageFromResource(theme.FyneLogo())
  myWindow.SetContent(
    fyne.NewContainerWithLayout(
      layout.NewGridWrapLayout(fyne.NewSize(150, 150)),
      img1, img2, img3))
  myWindow.ShowAndRun()
}
```

初始：

![](/img/in-post/godailylib/fyne16.png#center)

加大宽度：

![](/img/in-post/godailylib/fyne17.png#center)

再加大宽度：

![](/img/in-post/godailylib/fyne18.png#center)

### BorderLayout

边框布局（`BorderLayout`）比较常用于构建用户界面，上面例子中的`Toolbar`一般都和`BorderLayout`搭配使用。创建方法`layout.NewBorderLayout(top, bottom, left, right)`，分别传入顶部、底部、左侧、右侧的控件对象。添加到容器中的控件如果是这些**边界**对象，则显示在对应位置，其他都显示在中心：

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Border Layout")

  left := canvas.NewText("left", color.White)
  right := canvas.NewText("right", color.White)
  top := canvas.NewText("top", color.White)
  bottom := canvas.NewText("bottom", color.White)
  content := widget.NewLabel(`Lorem ipsum dolor, 
  sit amet consectetur adipisicing elit.
  Quidem consectetur ipsam nesciunt,
  quasi sint expedita minus aut,
  porro iusto magnam ducimus voluptates cum vitae.
  Vero adipisci earum iure consequatur quidem.`)

  container := fyne.NewContainerWithLayout(
    layout.NewBorderLayout(top, bottom, left, right),
    top, bottom, left, right, content,
  )
  myWindow.SetContent(container)
  myWindow.ShowAndRun()
}
```

效果：

![](/img/in-post/godailylib/fyne19.png#center)

### FormLayout

表单布局（`FormLayout`）其实就是一个 2 列的`GridLayout`，但是针对表单做了一些微调。

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Border Layout")

  nameLabel := canvas.NewText("Name", color.Black)
  nameValue := canvas.NewText("dajun", color.White)
  ageLabel := canvas.NewText("Age", color.Black)
  ageValue := canvas.NewText("18", color.White)

  container := fyne.NewContainerWithLayout(
    layout.NewFormLayout(),
    nameLabel, nameValue, ageLabel, ageValue,
  )
  myWindow.SetContent(container)
  myWindow.Resize(fyne.NewSize(150, 150))
  myWindow.ShowAndRun()
}
```

运行效果：

![](/img/in-post/godailylib/fyne20.png#center)

### CenterLayout

`CenterLayout`将容器内的所有控件显示在中心位置，按传入的顺序显示。最后传入的控件显示最上层。`CenterLayout`中所有控件将保持它们的最小尺寸（大小能容纳其内容）。

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Center Layout")

  image := canvas.NewImageFromResource(theme.FyneLogo())
  image.FillMode = canvas.ImageFillOriginal
  text := canvas.NewText("Fyne Logo", color.Black)

  container := fyne.NewContainerWithLayout(
    layout.NewCenterLayout(),
    image, text,
  )
  myWindow.SetContent(container)
  myWindow.ShowAndRun()
}
```

运行结果：

![](/img/in-post/godailylib/fyne21.png#center)

字符串`Fyne Logo`显示在图片上层。如果我们把`text`和`image`顺序对调，字符串将会被图片挡住，无法看到。动手试一下~

### MaxLayout

`MaxLayout`与`CenterLayout`类似，不同之处在于`MaxLayout`会让容器内的元素都显示为最大尺寸（等于容器的大小）。细心的朋友可能发现了，在`CenterLayout`的示例中。我们设置了图片的填充模式为`ImageFillOriginal`。如果不设置填充模式，图片的默认`MinSize`为`(1, 1)`。可以`fmt.Println(image.MinSize())`验证一下。这样图片就不会显示在界面中。

在`MaxLayout`的容器中，我们不需要这样处理：

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Max Layout")

  image := canvas.NewImageFromResource(theme.FyneLogo())
  text := canvas.NewText("Fyne Logo", color.Black)

  container := fyne.NewContainerWithLayout(
    layout.NewMaxLayout(),
    image, text,
  )
  myWindow.SetContent(container)
  myWindow.Resize(fyne.Size(200, 200))
  myWindow.ShowAndRun()
}
```

运行结果：

![](/img/in-post/godailylib/fyne22.png#center)

注意，`canvas.Text`显示为左对齐了。如果要居中对齐，设置其`Alignment`属性为`fyne.TextAlignCenter`。

### 自定义 Layout

内置布局在子包`layout`中。它们都实现了`fyne.Layout`接口：

```golang
// src/fyne.io/fyne/layout.go
type Layout interface {
  Layout([]CanvasObject, Size)
  MinSize(objects []CanvasObject) Size
}
```

要实现自定义的布局，只需要实现这个接口。下面我们实现一个台阶（对角）的布局，好似一个矩阵的对角线，从左上到右下。首先定义一个新的类型。然后实现接口`fyne.Layout`的两个方法：

```golang
type diagonal struct {
}

func (d *diagonal) MinSize(objects []fyne.CanvasObject) fyne.Size {
  w, h := 0, 0
  for _, o := range objects {
    childSize := o.MinSize()

    w += childSize.Width
    h += childSize.Height
  }

  return fyne.NewSize(w, h)
}

func (d *diagonal) Layout(objects []fyne.CanvasObject, containerSize fyne.Size) {
  pos := fyne.NewPos(0, 0)
  for _, o := range objects {
    size := o.MinSize()
    o.Resize(size)
    o.Move(pos)

    pos = pos.Add(fyne.NewPos(size.Width, size.Height))
  }
}
```

`MinSize()`返回所有子控件的`MinSize`之和。`Layout()`从左上到右下排列控件。然后是使用：

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Diagonal Layout")

  img1 := canvas.NewImageFromResource(theme.FyneLogo())
  img1.FillMode = canvas.ImageFillOriginal
  img2 := canvas.NewImageFromResource(theme.FyneLogo())
  img2.FillMode = canvas.ImageFillOriginal
  img3 := canvas.NewImageFromResource(theme.FyneLogo())
  img3.FillMode = canvas.ImageFillOriginal

  container := fyne.NewContainerWithLayout(
    &diagonal{},
    img1, img2, img3,
  )
  myWindow.SetContent(container)
  myWindow.ShowAndRun()
}
```

运行结果：

![](/img/in-post/godailylib/fyne24.png#center)

## fyne demo

`fyne`提供了一个 Demo，演示了大部分控件和布局的使用。可使用下面命令安装，执行：

```cmd
$ go get fyne.io/fyne/cmd/fyne_demo
$ fyne_demo
```

效果图：

![](/img/in-post/godailylib/fyne27.png#center)

## `fyne`命令

`fyne`库为了方便开发者提供了`fyne`命令。`fyne`可以用来将静态资源打包进可执行程序，还能将整个应用程序打包成可发布的形式。`fyne`命令通过下面命令安装：

```cmd
$ go get fyne.io/fyne/cmd/fyne
```

安装完成之后`fyne`就在`$GOPATH/bin`目录中，将`$GOPATH/bin`添加到系统`$PATH`中就可以直接运行`fyne`命令了。

### 静态资源

其实在前面的示例中我们已经多次使用了`fyne`内置的静态资源，使用最多的要属`fyne.FyneLogo()`了。下面我们有两个图片`image1.png/image2.jpg`。我们使用`fyne bundle`命令将这两个图片打包进代码：

```cmd
$ fyne bundle image1.png >> bundled.go
$ fyne bundle -append image2.jpg >> bundled.go
```

第二个命令指定`-append`选项表示添加到现有文件中，生成的文件如下：

```golang
// bundled.go
package main

import "fyne.io/fyne"

var resourceImage1Png = &fyne.StaticResource{
	StaticName: "image1.png",
	StaticContent: []byte{...}}

var resourceImage2Jpg = &fyne.StaticResource{
	StaticName: "image2.jpg",
	StaticContent: []byte{...}}
```

实际上就是将图片内容存入一个字节切片中，我们在代码中就可以调用`canvas.NewImageFromResource()`，传入`resourceImage1Png`或`resourceImage2Jpg`来创建`canvas.Image`对象了。

```golang
func main() {
  myApp := app.New()
  myWindow := myApp.NewWindow("Bundle Resource")

  img1 := canvas.NewImageFromResource(resourceImage1Png)
  img1.FillMode = canvas.ImageFillOriginal
  img2 := canvas.NewImageFromResource(resourceImage2Jpg)
  img2.FillMode = canvas.ImageFillOriginal
  img3 := canvas.NewImageFromResource(theme.FyneLogo())
  img3.FillMode = canvas.ImageFillOriginal

  container := fyne.NewContainerWithLayout(
    layout.NewGridLayout(1),
    img1, img2, img3,
  )
  myWindow.SetContent(container)
  myWindow.ShowAndRun()
}
```

运行结果：

![](/img/in-post/godailylib/fyne25.png#center)

注意，由于现在是两个文件，不能使用`go run main.go`，应该用`go run .`。

`theme.FyneLogo()`实际上是也是提前打包进代码的，代码文件是`bundled-icons.go`：

```golang
// src/fyne.io/fyne/theme/icons.go
func FyneLogo() fyne.Resource {
  return fynelogo
}

// src/fyne.io/fyne/theme/bundled-icons.go
var fynelogo = &fyne.StaticResource{
	StaticName: "fyne.png",
	StaticContent: []byte{}}
```

### 发布应用程序

发布图像应用程序到多个操作系统是非常复杂的任务。图形界面应用程序通常有图标和一些元数据。`fyne`命令提供了将应用程序发布到多个平台的支持。使用`fyne package`命令将创建一个可在其它计算机上安装/运行的应用程序。在 Windows 上，`fyne package`会创建一个`.exe`文件。在 macOS 上，会创建一个`.app`文件。在 Linux 上，会生成一个`.tar.xz`文件，可手动安装。

我们将上面的应用程序打包成一个`exe`文件：

```cmd
$ fyne package -os windows -icon icon.jpg
```

上面命令会在同目录下生成两个文件`bundle.exe`和`fyne.syso`，将这两个文件拷贝到任何目录或其他 Windows 计算机都可以通过直接双击`bundle.exe`运行了。没有其他的依赖。

![](/img/in-post/godailylib/fyne26.png#center)

`fyne`还支持交叉编译，能在 windows 上编译 mac 的应用程序，不过需要安装额外的工具，感兴趣可自行探索。

## 总结

`fyne`提供了丰富的组件和功能，我们介绍的只是很基础的一部分，还有剪切板、快捷键、滚动条、菜单等等等等内容。`fyne`命令实现打包静态资源和应用程序，非常方便。`fyne`还有其他高级功能留待大家探索、挖掘~

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