---
layout:    post
title:    "Go 每日一库之 mapstructure"
subtitle: 	"每天学习一个 Go 库"
date:		2020-07-29T05:40:24
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/07/29/godailylib/mapstructure"
categories: [
	"Go"
]
---

## 简介

[`mapstructure`](https://github.com/mitchellh/mapstructure)用于将通用的`map[string]interface{}`解码到对应的 Go 结构体中，或者执行相反的操作。很多时候，解析来自多种源头的数据流时，我们一般事先并不知道他们对应的具体类型。只有读取到一些字段之后才能做出判断。这时，我们可以先使用标准的`encoding/json`库将数据解码为`map[string]interface{}`类型，然后根据标识字段利用`mapstructure`库转为相应的 Go 结构体以便使用。

## 快速使用

本文代码采用 Go Modules。

首先创建目录并初始化：

```cmd
$ mkdir mapstructure && cd mapstructure

$ go mod init github.com/darjun/go-daily-lib/mapstructure
```

下载`mapstructure`库：

```cmd
$ go get github.com/mitchellh/mapstructure
```

使用：

```golang
package main

import (
  "encoding/json"
  "fmt"
  "log"

  "github.com/mitchellh/mapstructure"
)

type Person struct {
  Name string
  Age  int
  Job  string
}

type Cat struct {
  Name  string
  Age   int
  Breed string
}

func main() {
  datas := []string{`
    { 
      "type": "person",
      "name":"dj",
      "age":18,
      "job": "programmer"
    }
  `,
    `
    {
      "type": "cat",
      "name": "kitty",
      "age": 1,
      "breed": "Ragdoll"
    }
  `,
  }

  for _, data := range datas {
    var m map[string]interface{}
    err := json.Unmarshal([]byte(data), &m)
    if err != nil {
      log.Fatal(err)
    }

    switch m["type"].(string) {
    case "person":
      var p Person
      mapstructure.Decode(m, &p)
      fmt.Println("person", p)

    case "cat":
      var cat Cat
      mapstructure.Decode(m, &cat)
      fmt.Println("cat", cat)
    }
  }
}
```

运行结果：

```cmd
$ go run main.go
person {dj 18 programmer}
cat {kitty 1 Ragdoll}
```

我们定义了两个结构体`Person`和`Cat`，他们的字段有些许不同。现在，我们约定通信的 JSON 串中有一个`type`字段。当`type`的值为`person`时，该 JSON 串表示的是`Person`类型的数据。当`type`的值为`cat`时，该 JSON 串表示的是`Cat`类型的数据。

上面代码中，我们先用`json.Unmarshal`将字节流解码为`map[string]interface{}`类型。然后读取里面的`type`字段。根据`type`字段的值，再使用`mapstructure.Decode`将该 JSON 串分别解码为`Person`和`Cat`类型的值，并输出。

实际上，Google Protobuf 通常也使用这种方式。在协议中添加消息 ID 或**全限定消息名**。接收方收到数据后，先读取协议 ID 或**全限定消息名**。然后调用 Protobuf 的解码方法将其解码为对应的`Message`结构。从这个角度来看，`mapstructure`也可以用于网络消息解码，**如果你不考虑性能的话**😄。

## 字段标签

默认情况下，`mapstructure`使用结构体中字段的名称做这个映射，例如我们的结构体有一个`Name`字段，`mapstructure`解码时会在`map[string]interface{}`中查找键名`name`。注意，这里的`name`是大小写不敏感的！

```golang
type Person struct {
  Name string
}
```

当然，我们也可以指定映射的字段名。为了做到这一点，我们需要为字段设置`mapstructure`标签。例如下面使用`username`代替上例中的`name`：

```golang
type Person struct {
  Name string `mapstructure:"username"`
}
```

看示例：

```golang
type Person struct {
  Name string `mapstructure:"username"`
  Age  int
  Job  string
}

type Cat struct {
  Name  string
  Age   int
  Breed string
}

func main() {
  datas := []string{`
    { 
      "type": "person",
      "username":"dj",
      "age":18,
      "job": "programmer"
    }
  `,
    `
    {
      "type": "cat",
      "name": "kitty",
      "Age": 1,
      "breed": "Ragdoll"
    }
  `,
    `
    {
      "type": "cat",
      "Name": "rooooose",
      "age": 2,
      "breed": "shorthair"
    }
  `,
  }

  for _, data := range datas {
    var m map[string]interface{}
    err := json.Unmarshal([]byte(data), &m)
    if err != nil {
      log.Fatal(err)
    }

    switch m["type"].(string) {
    case "person":
      var p Person
      mapstructure.Decode(m, &p)
      fmt.Println("person", p)

    case "cat":
      var cat Cat
      mapstructure.Decode(m, &cat)
      fmt.Println("cat", cat)
    }
  }
}
```

上面代码中，我们使用标签`mapstructure:"username"`将`Person`的`Name`字段映射为`username`，在 JSON 串中我们需要设置`username`才能正确解析。另外，注意到，我们将第二个 JSON 串中的`Age`和第三个 JSON 串中的`Name`首字母大写了，但是并没有影响解码结果。`mapstructure`处理字段映射是大小写不敏感的。

## 内嵌结构

结构体可以任意嵌套，嵌套的结构被认为是拥有该结构体名字的另一个字段。例如，下面两种`Friend`的定义方式对于`mapstructure`是一样的：

```golang
type Person struct {
  Name string
}

// 方式一
type Friend struct {
  Person
}

// 方式二
type Friend struct {
  Person Person
}
```

为了正确解码，`Person`结构的数据要在`person`键下：

```golang
map[string]interface{} {
  "person": map[string]interface{}{"name": "dj"},
}
```

我们也可以设置`mapstructure:",squash"`将该结构体的字段提到父结构中：

```golang
type Friend struct {
  Person `mapstructure:",squash"`
}
```

这样只需要这样的 JSON 串，无效嵌套`person`键：

```golang
map[string]interface{}{
  "name": "dj",
}
```

看示例：

```golang
type Person struct {
  Name string
}

type Friend1 struct {
  Person
}

type Friend2 struct {
  Person `mapstructure:",squash"`
}

func main() {
  datas := []string{`
    { 
      "type": "friend1",
      "person": {
        "name":"dj"
      }
    }
  `,
    `
    {
      "type": "friend2",
      "name": "dj2"
    }
  `,
  }

  for _, data := range datas {
    var m map[string]interface{}
    err := json.Unmarshal([]byte(data), &m)
    if err != nil {
      log.Fatal(err)
    }

    switch m["type"].(string) {
    case "friend1":
      var f1 Friend1
      mapstructure.Decode(m, &f1)
      fmt.Println("friend1", f1)

    case "friend2":
      var f2 Friend2
      mapstructure.Decode(m, &f2)
      fmt.Println("friend2", f2)
    }
  }
}
```

注意对比`Friend1`和`Friend2`使用的 JSON 串的不同。

另外需要注意一点，如果父结构体中有同名的字段，那么`mapstructure`会将JSON 中对应的值**同时设置到这两个字段中**，即这两个字段有相同的值。

## 未映射的值

如果源数据中有未映射的值（即结构体中无对应的字段），`mapstructure`默认会忽略它。

我们可以在结构体中定义一个字段，为其设置`mapstructure:",remain"`标签。这样未映射的值就会添加到这个字段中。注意，这个字段的类型只能为`map[string]interface{}`或`map[interface{}]interface{}`。

看示例：

```golang
type Person struct {
  Name  string
  Age   int
  Job   string
  Other map[string]interface{} `mapstructure:",remain"`
}

func main() {
  data := `
    { 
      "name": "dj",
      "age":18,
      "job":"programmer",
      "height":"1.8m",
      "handsome": true
    }
  `

  var m map[string]interface{}
  err := json.Unmarshal([]byte(data), &m)
  if err != nil {
    log.Fatal(err)
  }

  var p Person
  mapstructure.Decode(m, &p)
  fmt.Println("other", p.Other)
}
```

上面代码中，我们为结构体定义了一个`Other`字段，用于保存未映射的键值。输出结果：

```golang
other map[handsome:true height:1.8m]
```

## 逆向转换

前面我们都是将`map[string]interface{}`解码到 Go 结构体中。`mapstructure`当然也可以将 Go 结构体反向解码为`map[string]interface{}`。在反向解码时，我们可以为某些字段设置`mapstructure:",omitempty"`。这样当这些字段为默认值时，就不会出现在结构的`map[string]interface{}`中：

```golang
type Person struct {
  Name string
  Age  int
  Job  string `mapstructure:",omitempty"`
}

func main() {
  p := &Person{
    Name: "dj",
    Age:  18,
  }

  var m map[string]interface{}
  mapstructure.Decode(p, &m)

  data, _ := json.Marshal(m)
  fmt.Println(string(data))
}
```

上面代码中，我们为`Job`字段设置了`mapstructure:",omitempty"`，且对象`p`的`Job`字段未设置。运行结果：

```cmd
$ go run main.go 
{"Age":18,"Name":"dj"}
```

## `Metadata`

解码时会产生一些有用的信息，`mapstructure`可以使用`Metadata`收集这些信息。`Metadata`结构如下：

```golang
// mapstructure.go
type Metadata struct {
  Keys   []string
  Unused []string
}
```

`Metadata`只有两个导出字段：

* `Keys`：解码成功的键名；
* `Unused`：在源数据中存在，但是目标结构中不存在的键名。

为了收集这些数据，我们需要使用`DecodeMetadata`来代替`Decode`方法：

```golang
type Person struct {
  Name string
  Age  int
}

func main() {
  m := map[string]interface{}{
    "name": "dj",
    "age":  18,
    "job":  "programmer",
  }

  var p Person
  var metadata mapstructure.Metadata
  mapstructure.DecodeMetadata(m, &p, &metadata)

  fmt.Printf("keys:%#v unused:%#v\n", metadata.Keys, metadata.Unused)
}
```

先定义一个`Metadata`结构，传入`DecodeMetadata`收集解码的信息。运行结果：

```cmd
$ go run main.go 
keys:[]string{"Name", "Age"} unused:[]string{"job"}
```

## 错误处理

`mapstructure`执行转换的过程中不可避免地会产生错误，例如 JSON 中某个键的类型与对应 Go 结构体中的字段类型不一致。`Decode/DecodeMetadata`会返回这些错误：

```golang
type Person struct {
  Name   string
  Age    int
  Emails []string
}

func main() {
  m := map[string]interface{}{
    "name":   123,
    "age":    "bad value",
    "emails": []int{1, 2, 3},
  }

  var p Person
  err := mapstructure.Decode(m, &p)
  if err != nil {
    fmt.Println(err.Error())
  }
}
```

上面代码中，结构体中`Person`中字段`Name`为`string`类型，但输入中`name`为`int`类型；字段`Age`为`int`类型，但输入中`age`为`string`类型；字段`Emails`为`[]string`类型，但输入中`emails`为`[]int`类型。故`Decode`返回错误。运行结果：

```cmd
$ go run main.go 
5 error(s) decoding:

* 'Age' expected type 'int', got unconvertible type 'string'
* 'Emails[0]' expected type 'string', got unconvertible type 'int'
* 'Emails[1]' expected type 'string', got unconvertible type 'int'
* 'Emails[2]' expected type 'string', got unconvertible type 'int'
* 'Name' expected type 'string', got unconvertible type 'int'
```

从错误信息中很容易看出哪里出错了。

### 弱类型输入

有时候，我们并不想对结构体字段类型和`map[string]interface{}`的对应键值做强类型一致的校验。这时可以使用`WeakDecode/WeakDecodeMetadata`方法，它们会尝试做类型转换：

```golang
type Person struct {
  Name   string
  Age    int
  Emails []string
}

func main() {
  m := map[string]interface{}{
    "name":   123,
    "age":    "18",
    "emails": []int{1, 2, 3},
  }

  var p Person
  err := mapstructure.WeakDecode(m, &p)
  if err == nil {
    fmt.Println("person:", p)
  } else {
    fmt.Println(err.Error())
  }
}
```

虽然键`name`对应的值`123`是`int`类型，但是在`WeakDecode`中会将其转换为`string`类型以匹配`Person.Name`字段的类型。同样的，`age`的值`"18"`是`string`类型，在`WeakDecode`中会将其转换为`int`类型以匹配`Person.Age`字段的类型。
需要注意一点，如果类型转换失败了，`WeakDecode`同样会返回错误。例如将上例中的`age`设置为`"bad value"`，它就不能转为`int`类型，故而返回错误。

## 解码器

除了上面介绍的方法外，`mapstructure`还提供了更灵活的解码器（`Decoder`）。可以通过配置`DecoderConfig`实现上面介绍的任何功能：

```golang
// mapstructure.go
type DecoderConfig struct {
	ErrorUnused       bool
	ZeroFields        bool
	WeaklyTypedInput  bool
	Metadata          *Metadata
	Result            interface{}
	TagName           string
}
```

各个字段含义如下：

* `ErrorUnused`：为`true`时，如果输入中的键值没有与之对应的字段就返回错误；
* `ZeroFields`：为`true`时，在`Decode`前清空目标`map`。为`false`时，则执行的是`map`的合并。用在`struct`到`map`的转换中；
* `WeaklyTypedInput`：实现`WeakDecode/WeakDecodeMetadata`的功能；
* `Metadata`：不为`nil`时，收集`Metadata`数据；
* `Result`：为结果对象，在`map`到`struct`的转换中，`Result`为`struct`类型。在`struct`到`map`的转换中，`Result`为`map`类型；
* `TagName`：默认使用`mapstructure`作为结构体的标签名，可以通过该字段设置。

看示例：

```golang
type Person struct {
  Name string
  Age  int
}

func main() {
  m := map[string]interface{}{
    "name": 123,
    "age":  "18",
    "job":  "programmer",
  }

  var p Person
  var metadata mapstructure.Metadata

  decoder, err := mapstructure.NewDecoder(&mapstructure.DecoderConfig{
    WeaklyTypedInput: true,
    Result:           &p,
    Metadata:         &metadata,
  })

  if err != nil {
    log.Fatal(err)
  }

  err = decoder.Decode(m)
  if err == nil {
    fmt.Println("person:", p)
    fmt.Printf("keys:%#v, unused:%#v\n", metadata.Keys, metadata.Unused)
  } else {
    fmt.Println(err.Error())
  }
}
```

这里用`Decoder`的方式实现了前面弱类型输入小节中的示例代码。实际上`WeakDecode`内部就是通过这种方式实现的，下面是`WeakDecode`的源码：

```golang
// mapstructure.go
func WeakDecode(input, output interface{}) error {
  config := &DecoderConfig{
    Metadata:         nil,
    Result:           output,
    WeaklyTypedInput: true,
  }

  decoder, err := NewDecoder(config)
  if err != nil {
    return err
  }

  return decoder.Decode(input)
}
```

再实际上，`Decode/DecodeMetadata/WeakDecodeMetadata`内部都是先设置`DecoderConfig`的对应字段，然后创建`Decoder`对象，最后调用其`Decode`方法实现的。

## 总结

`mapstructure`实现优雅，功能丰富，代码结构清晰，非常推荐一看！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. mapstructure GitHub：[https://github.com/mitchellh/mapstructure](https://github.com/mitchellh/mapstructure)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)