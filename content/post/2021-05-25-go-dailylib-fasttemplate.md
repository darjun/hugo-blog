---
layout:    post
title:    "Go 每日一库之 fasttemplate"
subtitle: 	"每天学习一个 Go 库"
date:		2021-05-24T23:30:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/05/24/godailylib/fasttemplate"
categories: [
	"Go"
]
---

## 简介

[`fasttemplate`](https://github.com/valyala/fasttemplate)是一个比较简单、易用的小型模板库。`fasttemplate`的作者[valyala](https://github.com/valyala)另外还开源了不少优秀的库，如大名鼎鼎的`fasthttp`，前面介绍的`bytebufferpool`，还有一个重量级的模板库`quicktemplate`。`quicktemplate`比标准库中的`text/template`和`html/template`要灵活和易用很多，后面会专门介绍它。今天要介绍的`fasttemlate`只专注于一块很小的领域——字符串替换。它的目标是为了替代`strings.Replace`、`fmt.Sprintf`等方法，提供一个简单，易用，高性能的字符串替换方法。

本文首先介绍`fasttemplate`的用法，然后去看看源码实现的一些细节。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir fasttemplate && cd fasttemplate
$ go mod init github.com/darjun/go-daily-lib/fasttemplate
```

安装`fasttemplate`库：

```cmd
$ go get -u github.com/valyala/fasttemplate
```

编写代码：

```golang
package main

import (
  "fmt"

  "github.com/valyala/fasttemplate"
)

func main() {
  template := `name: {{name}}
age: {{age}}`
  t := fasttemplate.New(template, "{{", "}}")
  s1 := t.ExecuteString(map[string]interface{}{
    "name": "dj",
    "age":  "18",
  })
  s2 := t.ExecuteString(map[string]interface{}{
    "name": "hjw",
    "age":  "20",
  })
  fmt.Println(s1)
  fmt.Println(s2)
}
```

* 定义模板字符串，使用`{{`和`}}`表示占位符，占位符可以在创建模板的时候指定；
* 调用`fasttemplate.New()`创建一个模板对象`t`，传入开始和结束占位符；
* 调用模板对象的`t.ExecuteString()`方法，传入参数。参数中有各个占位符对应的值。生成最终的字符串。

运行结果：

```cmd
name: dj
age: 18
```

我们可以自定义占位符，上面分别使用`{{`和`}}`作为开始和结束占位符。我们可以换成`[[`和`]]`，只需要简单修改一下代码即可：

```golang
template := `name: [[name]]
age: [[age]]`
t := fasttemplate.New(template, "[[", "]]")
```

另外，需要注意的是，传入参数的类型为`map[string]interface{}`，但是`fasttemplate`只接受类型为`[]byte`、`string`和`TagFunc`类型的值。这也是为什么上面的`18`要用双引号括起来的原因。

另一个需要注意的点，`fasttemplate.New()`返回一个模板对象，如果模板解析失败了，就会直接`panic`。如果想要自己处理错误，可以调用`fasttemplate.NewTemplate()`方法，该方法返回一个模板对象和一个错误。实际上，`fasttemplate.New()`内部就是调用`fasttemplate.NewTemplate()`，如果返回了错误，就`panic`：

```golang
// src/github.com/valyala/fasttemplate/template.go
func New(template, startTag, endTag string) *Template {
  t, err := NewTemplate(template, startTag, endTag)
  if err != nil {
    panic(err)
  }
  return t
}

func NewTemplate(template, startTag, endTag string) (*Template, error) {
  var t Template
  err := t.Reset(template, startTag, endTag)
  if err != nil {
    return nil, err
  }
  return &t, nil
}
```

**这其实也是一种惯用法，对于不想处理错误的示例程序，直接`panic`有时也是一种选择**。例如`html.template`标准库也提供了`Must()`方法，一般这样用，遇到解析失败就`panic`：

```golang
t := template.Must(template.New("name").Parse("html"))
```

**占位符中间内部不要加空格！！！**

**占位符中间内部不要加空格！！！**

**占位符中间内部不要加空格！！！**

## 快捷方式

使用`fasttemplate.New()`定义模板对象的方式，我们可以多次使用不同的参数去做替换。但是，有时候我们要做大量一次性的替换，每次都定义模板对象显得比较繁琐。`fasttemplate`也提供了一次性替换的方法：

```golang
func main() {
  template := `name: [name]
age: [age]`
  s := fasttemplate.ExecuteString(template, "[", "]", map[string]interface{}{
    "name": "dj",
    "age":  "18",
  })
  fmt.Println(s)
}
```

使用这种方式，我们需要同时传入模板字符串、开始占位符、结束占位符和替换参数。

## `TagFunc`

`fasttemplate`提供了一个`TagFunc`，可以给替换增加一些逻辑。`TagFunc`是一个函数：

```golang
type TagFunc func(w io.Writer, tag string) (int, error)
```

在执行替换的时候，`fasttemplate`针对每个占位符都会调用一次`TagFunc`函数，`tag`即占位符的名称。看下面程序：

```golang
func main() {
  template := `name: {{name}}
age: {{age}}`
  t := fasttemplate.New(template, "{{", "}}")
  s := t.ExecuteFuncString(func(w io.Writer, tag string) (int, error) {
    switch tag {
    case "name":
      return w.Write([]byte("dj"))
    case "age":
      return w.Write([]byte("18"))
    default:
      return 0, nil
    }
  })

  fmt.Println(s)
}
```

这其实就是`get-started`示例程序的`TagFunc`版本，根据传入的`tag`写入不同的值。如果我们去查看源码就会发现，实际上`ExecuteString()`最终还是会调用`ExecuteFuncString()`。`fasttemplate`提供了一个标准的`TagFunc`：

```golang
func (t *Template) ExecuteString(m map[string]interface{}) string {
  return t.ExecuteFuncString(func(w io.Writer, tag string) (int, error) { return stdTagFunc(w, tag, m) })
}

func stdTagFunc(w io.Writer, tag string, m map[string]interface{}) (int, error) {
  v := m[tag]
  if v == nil {
    return 0, nil
  }
  switch value := v.(type) {
  case []byte:
    return w.Write(value)
  case string:
    return w.Write([]byte(value))
  case TagFunc:
    return value(w, tag)
  default:
    panic(fmt.Sprintf("tag=%q contains unexpected value type=%#v. Expected []byte, string or TagFunc", tag, v))
  }
}
```

标准的`TagFunc`实现也非常简单，就是从参数`map[string]interface{}`中取出对应的值做相应处理，如果是`[]byte`和`string`类型，直接调用`io.Writer`的写入方法。如果是`TagFunc`类型则直接调用该方法，将`io.Writer`和`tag`传入。其他类型直接`panic`抛出错误。

如果模板中的`tag`在参数`map[string]interface{}`中不存在，有两种处理方式：

* 直接忽略，相当于替换成了空字符串`""`。标准的`stdTagFunc`就是这样处理的；
* 保留原始`tag`。`keepUnknownTagFunc`就是做这个事情的。

`keepUnknownTagFunc`代码如下：

```golang
func keepUnknownTagFunc(w io.Writer, startTag, endTag, tag string, m map[string]interface{}) (int, error) {
  v, ok := m[tag]
  if !ok {
    if _, err := w.Write(unsafeString2Bytes(startTag)); err != nil {
      return 0, err
    }
    if _, err := w.Write(unsafeString2Bytes(tag)); err != nil {
      return 0, err
    }
    if _, err := w.Write(unsafeString2Bytes(endTag)); err != nil {
      return 0, err
    }
    return len(startTag) + len(tag) + len(endTag), nil
  }
  if v == nil {
    return 0, nil
  }
  switch value := v.(type) {
  case []byte:
    return w.Write(value)
  case string:
    return w.Write([]byte(value))
  case TagFunc:
    return value(w, tag)
  default:
    panic(fmt.Sprintf("tag=%q contains unexpected value type=%#v. Expected []byte, string or TagFunc", tag, v))
  }
}
```

后半段处理与`stdTagFunc`一样，函数前半部分如果`tag`未找到。直接写入`startTag` + `tag` + `endTag`作为替换的值。

我们前面调用的`ExecuteString()`方法使用`stdTagFunc`，即直接将未识别的`tag`替换成空字符串。如果想保留未识别的`tag`，改为调用`ExecuteStringStd()`方法即可。该方法遇到未识别的`tag`会保留：

```golang
func main() {
  template := `name: {{name}}
age: {{age}}`
  t := fasttemplate.New(template, "{{", "}}")
  m := map[string]interface{}{"name": "dj"}
  s1 := t.ExecuteString(m)
  fmt.Println(s1)

  s2 := t.ExecuteStringStd(m)
  fmt.Println(s2)
}
```

参数中缺少`age`，运行结果：

```cmd
name: dj
age:
name: dj
age: {{age}}
```

## 带`io.Writer`参数的方法

前面介绍的方法最后都是返回一个字符串。方法名中都有`String`：`ExecuteString()/ExecuteFuncString()`。

我们可以直接传入一个`io.Writer`参数，将结果字符串调用这个参数的`Write()`方法直接写入。这类方法名中没有`String`：`Execute()/ExecuteFunc()`：

```golang
func main() {
  template := `name: {{name}}
age: {{age}}`
  t := fasttemplate.New(template, "{{", "}}")
  t.Execute(os.Stdout, map[string]interface{}{
    "name": "dj",
    "age":  "18",
  })

  fmt.Println()

  t.ExecuteFunc(os.Stdout, func(w io.Writer, tag string) (int, error) {
    switch tag {
    case "name":
      return w.Write([]byte("hjw"))
    case "age":
      return w.Write([]byte("20"))
    }

    return 0, nil
  })
}
```

由于`os.Stdout`实现了`io.Writer`接口，可以直接传入。结果直接写到`os.Stdout`中。运行：

```cmd
name: dj
age: 18
name: hjw
age: 20
```

## 源码分析

首先看模板对象的结构和创建：

```golang
// src/github.com/valyala/fasttemplate/template.go
type Template struct {
  template string
  startTag string
  endTag   string

  texts          [][]byte
  tags           []string
  byteBufferPool bytebufferpool.Pool
}

func NewTemplate(template, startTag, endTag string) (*Template, error) {
  var t Template
  err := t.Reset(template, startTag, endTag)
  if err != nil {
    return nil, err
  }
  return &t, nil
}
```

模板创建之后会调用`Reset()`方法初始化：

```golang
func (t *Template) Reset(template, startTag, endTag string) error {
  t.template = template
  t.startTag = startTag
  t.endTag = endTag
  t.texts = t.texts[:0]
  t.tags = t.tags[:0]

  if len(startTag) == 0 {
    panic("startTag cannot be empty")
  }
  if len(endTag) == 0 {
    panic("endTag cannot be empty")
  }

  s := unsafeString2Bytes(template)
  a := unsafeString2Bytes(startTag)
  b := unsafeString2Bytes(endTag)

  tagsCount := bytes.Count(s, a)
  if tagsCount == 0 {
    return nil
  }

  if tagsCount+1 > cap(t.texts) {
    t.texts = make([][]byte, 0, tagsCount+1)
  }
  if tagsCount > cap(t.tags) {
    t.tags = make([]string, 0, tagsCount)
  }

  for {
    n := bytes.Index(s, a)
    if n < 0 {
      t.texts = append(t.texts, s)
      break
    }
    t.texts = append(t.texts, s[:n])

    s = s[n+len(a):]
    n = bytes.Index(s, b)
    if n < 0 {
      return fmt.Errorf("Cannot find end tag=%q in the template=%q starting from %q", endTag, template, s)
    }

    t.tags = append(t.tags, unsafeBytes2String(s[:n]))
    s = s[n+len(b):]
  }

  return nil
}
```

初始化做了下面这些事情：

* 记录开始和结束占位符；
* 解析模板，将文本和`tag`切分开，分别存放在`texts`和`tags`切片中。后半段的`for`循环就是做的这个事情。

代码细节点：

* 先统计占位符一共多少个，一次构造对应大小的文本和`tag`切片，注意构造正确的模板字符串文本切片一定比`tag`切片大 1。像这样`| text | tag | text | ... | tag | text |`；
* 为了避免内存拷贝，使用`unsafeString2Bytes`让返回的字节切片直接指向`string`内部地址。

看上面的介绍，貌似有很多方法。实际上核心的方法就一个`ExecuteFunc()`。其他的方法都是直接或间接地调用它：

```golang
// src/github.com/valyala/fasttemplate/template.go
func (t *Template) Execute(w io.Writer, m map[string]interface{}) (int64, error) {
  return t.ExecuteFunc(w, func(w io.Writer, tag string) (int, error) { return stdTagFunc(w, tag, m) })
}

func (t *Template) ExecuteStd(w io.Writer, m map[string]interface{}) (int64, error) {
  return t.ExecuteFunc(w, func(w io.Writer, tag string) (int, error) { return keepUnknownTagFunc(w, t.startTag, t.endTag, tag, m) })
}

func (t *Template) ExecuteFuncString(f TagFunc) string {
  s, err := t.ExecuteFuncStringWithErr(f)
  if err != nil {
    panic(fmt.Sprintf("unexpected error: %s", err))
  }
  return s
}

func (t *Template) ExecuteFuncStringWithErr(f TagFunc) (string, error) {
  bb := t.byteBufferPool.Get()
  if _, err := t.ExecuteFunc(bb, f); err != nil {
    bb.Reset()
    t.byteBufferPool.Put(bb)
    return "", err
  }
  s := string(bb.Bytes())
  bb.Reset()
  t.byteBufferPool.Put(bb)
  return s, nil
}

func (t *Template) ExecuteString(m map[string]interface{}) string {
  return t.ExecuteFuncString(func(w io.Writer, tag string) (int, error) { return stdTagFunc(w, tag, m) })
}

func (t *Template) ExecuteStringStd(m map[string]interface{}) string {
  return t.ExecuteFuncString(func(w io.Writer, tag string) (int, error) { return keepUnknownTagFunc(w, t.startTag, t.endTag, tag, m) })
}
```

`Execute()`方法构造一个`TagFunc`调用`ExecuteFunc()`，内部使用`stdTagFunc`：

```golang
func(w io.Writer, tag string) (int, error) {
  return stdTagFunc(w, tag, m)
}
```

`ExecuteStd()`方法构造一个`TagFunc`调用`ExecuteFunc()`，内部使用`keepUnknownTagFunc`：

```golang
func(w io.Writer, tag string) (int, error) {
  return keepUnknownTagFunc(w, t.startTag, t.endTag, tag, m)
}
```

`ExecuteString()`和`ExecuteStringStd()`方法调用`ExecuteFuncString()`方法，而`ExecuteFuncString()`方法又调用了`ExecuteFuncStringWithErr()`方法，`ExecuteFuncStringWithErr()`方法内部使用`bytebufferpool.Get()`获得一个`bytebufferpoo.Buffer`对象去调用`ExecuteFunc()`方法。所以核心就是`ExecuteFunc()`方法：

```golang
func (t *Template) ExecuteFunc(w io.Writer, f TagFunc) (int64, error) {
  var nn int64

  n := len(t.texts) - 1
  if n == -1 {
    ni, err := w.Write(unsafeString2Bytes(t.template))
    return int64(ni), err
  }

  for i := 0; i < n; i++ {
    ni, err := w.Write(t.texts[i])
    nn += int64(ni)
    if err != nil {
      return nn, err
    }

    ni, err = f(w, t.tags[i])
    nn += int64(ni)
    if err != nil {
      return nn, err
    }
  }
  ni, err := w.Write(t.texts[n])
  nn += int64(ni)
  return nn, err
}
```

整个逻辑也很清晰，`for`循环就是`Write`一个`texts`元素，以当前的`tag`执行`TagFunc`，索引 +1。最后写入最后一个`texts`元素，完成。大概是这样：

```golang
| text | tag | text | tag | text | ... | tag | text |
```

注：`ExecuteFuncStringWithErr()`方法使用到了前面文章介绍的`bytebufferpool`，感兴趣可以回去翻看。

## 总结

可以使用`fasttemplate`完成`strings.Replace`和`fmt.Sprintf`的任务，而且`fasttemplate`灵活性更高。代码清晰易懂，值得一看。

吐槽：关于命名，`Execute()`方法里面使用`stdTagFunc`，`ExecuteStd()`方法里面使用`keepUnknownTagFunc`方法。我想是不是把`stdTagFunc`改名为`defaultTagFunc`好一点？

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. fasttemplate GitHub：[github.com/valyala/fasttemplate](https://github.com/valyala/fasttemplate)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~