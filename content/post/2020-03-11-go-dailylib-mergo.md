---
layout:    post
title:    "Go 每日一库之 mergo"
subtitle: 	"每天学习一个 Go 库"
date:		2020-03-11T12:06:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/03/11/godailylib/mergo"
categories: [
	"Go"
]
---

## 简介

今天我们介绍一个合并结构体字段的库[`mergo`](https://github.com/imdario/mergo)。`mergo`可以在相同的结构体或`map`之间赋值，可以将结构体的字段赋值到`map`中，可以将`map`的值赋值给结构体的字段。感谢[@thinkgos](https://github.com/thinkgos)推荐。

## 快速使用

先安装：

```golang
$ go get github.com/imdario/mergo
```

后使用：

```golang
package main

import (
  "fmt"
  "log"

  "github.com/imdario/mergo"
)

type redisConfig struct {
  Address string
  Port    int
  DB      int
}

var defaultConfig = redisConfig{
  Address: "127.0.0.1",
  Port:    6381,
  DB:      1,
}

func main() {
  var config redisConfig

  if err := mergo.Merge(&config, defaultConfig); err != nil {
    log.Fatal(err)
  }

  fmt.Println("redis address: ", config.Address)
  fmt.Println("redis port: ", config.Port)
  fmt.Println("redis db: ", config.DB)

  var m = make(map[string]interface{})
  if err := mergo.Map(&m, defaultConfig); err != nil {
    log.Fatal(err)
  }

  fmt.Println(m)
}
```

使用非常简单。`mergo`提供了两组接口（其实就是两个，`*WithOverwrite`已经废弃了，可使用`WithOverride`选项代替）:

* `Merge`：合并两个相同类型的结构或`map`；
* `Map`：在结构和`map`之间赋值。

参数 1 是目标对象，参数 2 是源对象，这两个函数的功能就是将源对象中的字段复制到目标对象的对应字段上。

## 高级选项

如果仅仅只是复制结构体，为啥不直接写`redisConfig = defaultConfig`呢？`mergo`提供了很多选项。

### 覆盖

默认情况下，如果目标对象的字段已经设置了，那么`Merge/Map`不会用源对象中的字段替换它。我们在上面程序的`var config redisConfig`定义下添加一行：

```golang
config.DB = 2
```

再看看运行结果，发现输出的`db`是 2，而非 1。

可以通过选项来改变这个行为，调用`Merge/Map`时，传入`WithOverride`参数，那么目标对象中已经设置的字段也会被覆盖：

```golang
if err := mergo.Merge(&config, defaultConfig, mergo.WithOverride); err != nil {
  log.Fatal(err)
}
```

只需要修改这一行调用。结果输出`db`是 1，覆盖了！

这里用到了 Go 中的**选项模式**。在参数比较多，且大部分有默认值的情况下，我们可以在函数最后添加一个可变的选项参数，通过传入选项来改变函数的行为，不传入的选项就使用默认值。选项模式在 Go 语言中使用非常广泛，能大大提高代码的可扩展性，使用可变参数也能使函数更易用。`mergo`中的选项都是这种形式。想要深入了解一下？看这里[https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)。

`mergo`老的接口`MergeWithOverride`和`MapWithOverride`都使用选项模式重构了。

### 切片

如果某个字段是一个切片，不覆盖就保留目标对象的值，或者用源对象的值覆盖都不合适。我们可能想将源对象中切片的值对添加到目标对象的字段中，这时可以使用`WithAppendSlice`选项。

```golang
package main

import (
  "fmt"
  "log"

  "github.com/imdario/mergo"
)

type redisConfig struct {
  Address string
  Port    int
  DBs     []int
}

var defaultConfig = redisConfig{
  Address: "127.0.0.1",
  Port:    6381,
  DBs:     []int{1},
}

func main() {
  var config redisConfig
  config.DBs = []int{2, 3}

  if err := mergo.Merge(&config, defaultConfig, mergo.WithAppendSlice); err != nil {
    log.Fatal(err)
  }

  fmt.Println("redis address: ", config.Address)
  fmt.Println("redis port: ", config.Port)
  fmt.Println("redis dbs: ", config.DBs)
}
```

我们将`DB`字段改为`[]int`类型的`DBs`，使用`WithAppendSliec`选项，最后输出的`DBs`为`[2 3 1]`。

### 空值覆盖

默认情况下，如果源对象中的字段为空值（数组、切片长度为 0 ，指针为`nil`，数字为 0，字符串为""等），即使我们使用了`WithOverride`选项也是不会覆盖的。下面两个选项就是强制这种情况下也覆盖：

* `WithOverrideEmptySlice`：源对象的空切片覆盖目标对象的对应字段；
* `WithOverwriteWithEmptyValue`：源对象中的空值覆盖目标对象的对应字段，其实这个对切片也有效。

文档中这两个选项的介绍比较混乱，我通过看源码和自己试验下来发现：

* 这两个选项都必须和`WithOverride`一起使用；
* `WithOverwriteWithEmptyValue`这个选项也可以处理切片类型的值。

看下面代码：

```golang
type redisConfig struct {
  Address string
  Port    int
  DBs     []int
}

var defaultConfig = redisConfig{
  Address: "127.0.0.1",
  Port:    6381,
}

func main() {
  var config redisConfig
  config.DBs = []int{2, 3}

  if err := mergo.Merge(&config, defaultConfig, mergo.WithOverride, mergo.WithOverrideEmptySlice); err != nil {
    log.Fatal(err)
  }

  fmt.Println("redis address: ", config.Address)
  fmt.Println("redis port: ", config.Port)
  fmt.Println("redis dbs: ", config.DBs)
}
```

最终会输出空的`DBs`。

### 类型检查

这个主要用在`map`之间的切片字段的赋值，因为使用`mergo`在两个结构体之间赋值必须保证两个结构体类型相同，没有类型检查的必要。因为`map`类型为`map[string]interface{}`，所以默认情况下，`map`切片类型不一致也是可以赋值的：

```golang
func main() {
  m1 := make(map[string]interface{})
  m1["dbs"] = []uint32{2, 3}

  m2 := make(map[string]interface{})
  m2["dbs"] = []int{1}

  if err := mergo.Map(&m1, &m2, mergo.WithOverride); err != nil {
    log.Fatal(err)
  }

  fmt.Println(m1)
}
```

如果添加`mergo.WithTypeCheck`选项，则切片类型不一致会抛出错误：

```golang
if err := mergo.Map(&m1, &m2, mergo.WithOverride, mergo.WithTypeCheck); err != nil {
    log.Fatal(err)
}
```

输出：

```golang
cannot override two slices with different type ([]int, []uint32)
exit status 1
```

## 注意事项

1. `mergo`不会赋值非导出字段；
2. `map`中对应的键名首字母会转为小写；
3. `mergo`可嵌套赋值，我们演示的只有一层结构。

## 总结

`mergo`其实在很多知名项目中都有应用，如`moby/kubernetes`等。本文介绍了`mergo`的基本用法，感兴趣可以去 GitHub 上深入学习。关于选项模式，这里多说一句，我在实际项目中多次应用，能极大地提高可扩展性，方便今后添加新的功能。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. mergo GitHub：[https://github.com/imdario/mergo](https://github.com/imdario/mergo)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)