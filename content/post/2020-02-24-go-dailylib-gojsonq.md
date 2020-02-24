---
layout:    post
title:    "Go 每日一库之 gojsonq"
subtitle: 	"每天学习一个 Go 库"
date:		2020-02-24T14:31:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/02/24/godailylib/gojsonq"
categories: [
	"Go"
]
---

## 简介

在日常工作中，每一名开发者，不管是前端还是后端，都经常使用 JSON。JSON 是一个很简单的数据交换格式。相比于 XML，它灵活、轻巧、使用方便。JSON 也是[RESTful API](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)推荐的格式。有时，我们只想读取 JSON 中的某一些字段。如果自己手动解析、一层一层读取，这就变得异常繁琐了。特别是在嵌套层次很深的情况下。今天我们介绍[`gojsonq`](https://github.com/thedevsaddam/gojsonq)。它可以帮助我们很方便的操作 JSON。

## 快速使用

先安装：

```cmd
$ go get github.com/thedevsaddam/gojsonq
```

后使用：

```golang
package main

import (
  "fmt"

  "github.com/thedevsaddam/gojsonq"
)

func main() {
  content := `{
  "user": {
    "name": "dj",
    "age": 18,
    "address": {
      "provice": "shanghai",
      "district": "xuhui"
    },
    "hobbies":["chess", "programming", "game"]
  }
}`

  gq := gojsonq.New().FromString(content)
  district := gq.Find("user.address.district")
  fmt.Println(district)

  gq.Reset()

  hobby := gq.Find("user.hobbies.[0]")
  fmt.Println(hobby)
}
```

操作非常简单：

* 首先调用`gojsonq.New()`创建一个`JSONQ`的对象；
* 然后就可以使用该类型的方法来查询属性了。

上面代码我们直接读取位于最内层的`district`值和`hobbies`数组的第一个元素！层与层之间用`.`隔开，如果是数组，则在属性字段后通过`.[index]`读取下标为`index`的元素。这种方式可以实现很灵活的读取。

注意到一个细节：在查询之后，我们手动调用了一次`Reset()`方法。因为`JSONQ`对象在调用`Find`方法时，内部会记录当前的节点，下一个查询会从上次查找的节点开始。也就是说如果我们注释掉`jq.Reset()`，第二个`Find()`方法实际上查找的是`user.address.district.user.hobbies.[0]`，自然就返回`nil`了。除此之外，`gojsonq`也提供了另外一种方式。如果你想要保存当前查询的一些状态信息，可以调用`JSONQ`的`Copy`方法返回一个初始状态下的对象，它们会共用底层的 JSON 字符串和解析后的对象。上面的`gq.Reset()`可以由下面这行代码代替：

```golang
gpCopy := gp.Copy()
```

后面就可以使用`gpCopy`查询`hobbies`了。

这个算是`gojsonq`库的一个特点，但也是初学者带来了很多困扰，需要特别注意。实际上，`JSONQ`提供的很多方法会改变当前节点，稍后部分我们会更清楚的看到。

## 数据源

除了从字符串中加载，`jsonq`还允许从文件和`io.Reader`中读取内容。分别使用`JSONQ`对象的`File`和`Reader`方法：

```golang
func main() {
  gq := gojsonq.New().File("./data.json")

  fmt.Println(gq.Find("items.[1].price"))
}
```

和下面程序的效果是一样的：

```golang
func main() {
  file, err := os.OpenFile("./data.json", os.O_RDONLY, 0666)
  if err != nil {
    log.Fatal(err)
  }

  gq := gojsonq.New().Reader(file)

  fmt.Println(gq.Find("items.[1].price"))
}
```

为了后面演示方便，我构造了一个`data.json`文件：

```json
{
  "name": "shopping cart",
  "description": "List of items in your cart",
  "prices": ["2400", "2100", "1200", "400.87", "89.90", "150.10"],
  "items": [
    {
      "id": 1,
      "name": "Apple",
      "count": 2,
      "price": 12
    },
    {
      "id": 2,
      "name": "Notebook",
      "count": 10,
      "price": 3
    },
    {
      "id": 3,
      "name": "Pencil",
      "count": 5,
      "price": 1
    },
    {
      "id": 4,
      "name": "Camera",
      "count": 1,
      "price": 1750
    },
    {
      "id": null,
      "name": "Invalid Item",
      "count": 1,
      "price": 12000
    }
  ]
}
```

## 高级查询

`gojsonq`的独特之处在于，它可以像 SQL 一样进行条件查询，可以选择返回哪些字段，可以做一些聚合统计。

### 字段映射

有时候，我们只关心对象中的几个字段，这时候就可以使用`Select`指定返回哪些字段，其余字段不返回：

```golang
func main() {
  r := gojsonq.New().File("./data.json").From("items").Select("id", "name").Get()
  data, _ := json.MarshalIndent(r, "", "  ")
  fmt.Println(string(data))
}
```

只会输出`id`和`name`字段：

```cmd
$ go run main.go
[
  {
    "id": 1,
    "name": "Apple"
  },
  {
    "id": 2,
    "name": "Notebook"
  },
  {
    "id": 3,
    "name": "Pencil"
  },
  {
    "id": 4,
    "name": "Camera"
  },
  {
    "id": null,
    "name": "Invalid Item"
  }
]
```

为了显示更直观一点，我这里用`json.MarshalIndent()`对输出做了一些美化。

是不是和 SQL 有点像`Select id,name From items`...

这里介绍一下`From`方法，这个方法的作用是将当前节点移动到指定位置。上面也说过当前节点的位置是记下来的。例如，上面的代码中我们先将当前节点移动到`items`，后面的查询和聚合操作都是针对这个数组。实际上`Find`方法内部就调用了`From`：

```golang
// src/github.com/thedevsaddam/gojsonq/jsonq.go
func (j *JSONQ) Find(path string) interface{} {
  return j.From(path).Get()
}

func (j *JSONQ) From(node string) *JSONQ {
  j.node = node
  v, err := getNestedValue(j.jsonContent, node, j.option.separator)
  if err != nil {
    j.addError(err)
  }
  // ============= 注意这一行，记住当前节点位置
  j.jsonContent = v
  return j
}
```

最后必须要调用`Get()`，它组合所有条件后执行这个查询，返回结果。

### 条件查询

有了`Select`和`From`，怎么能没有`Where`呢？`gojsonq`提供的`Where`方法非常多，我们大概看几个就行了。

首先是，`Where(key, op, val)`，这个是通用的`Where`条件，表示`key`和`val`是否满足`op`关系。`op`内置的就有将近 20 种，还支持自定义。例如`=`表示相等，`!=`表示不等，`startsWith`表示`val`是否是`key`字段的前缀等等等等；

其他很多条件都是`Where`的特例，例如`WhereIn(key, val)`就等价于`Where(key, "in", val)`，`WhereStartsWith(key, val)`就等价于`Where(key, "startsWith", val)`。

默认情况下，`Where`的条件都是`And`连接的，我们可以通过`OrWhere`让其以`Or`连接：

```golang
func main() {
  gq := gojsonq.New().File("./data.json")

  r := gq.From("items").Select("id", "name").
    Where("id", "=", 1).OrWhere("id", "=", 2).Get()
  fmt.Println(r)

  gq.Reset()

  r = gq.From("items").Select("id", "name", "count").
    Where("count", ">", 1).Where("price", "<", 100).Get()
  fmt.Println(r)
}
```

上面第一个查询，查找`id`为 1 **或** 2 的记录。第二个查询，查找`count`大于 1 **且** `price`小于 100 的记录。

### 指定偏移和返回条目数

有时我们想要分页显示，第一次查询时返回前 3 条内容，第二次查询时返回接下来的 3 条记录。我们可以使用`JSONQ`对象的`Offset`和`Limit`方法来指定偏移和返回的条目数：

```golang
func main() {
  gq := gojsonq.New().File("./data.json")

  r1 := gq.From("items").Select("id", "name").Offset(0).Limit(3).Get()
  fmt.Println("First Page:", r1)

  gq.Reset()

  r2 := gq.From("items").Select("id", "name").Offset(3).Limit(3).Get()
  fmt.Println("Second Page:", r2)
}
```

来看看运行结果：

```cmd
$ go run main.go
First Page: [map[id:1 name:Apple] map[id:2 name:Notebook] map[id:3 name:Pencil]]
Second Page: [map[id:4 name:Camera] map[id:<nil> name:Invalid Item]]
```

### 聚合统计

我们还能可以对一些字段做简单的统计，计算和、平均数、最大、最小值等：

```golang
func main() {
  gq := gojsonq.New().File("./data.json").From("items")

  fmt.Println("Total Count:", gq.Sum("count"))
  fmt.Println("Min Price:", gq.Min("price"))
  fmt.Println("Max Price:", gq.Max("price"))
  fmt.Println("Avg Price:", gq.Avg("price"))
}
```

上面统计商品的总数量、最低价格、最高价格和平均价格。

**聚合统计类的方法都不会修改当前节点的指向，所以`JSONQ`对象可以重复使用！**

还可以对数据进行分组和排序：

```golang
func main() {
  gq := gojsonq.New().File("./data.json")

  fmt.Println(gq.From("items").GroupBy("price").Get())
  gq.Reset()
  fmt.Println(gq.From("items").SortBy("price", "desc").Get())
}
```

## 其他格式

默认情况下，`gojsonq`使用 JSON 格式解析数据。我们也可以设置其他格式解析器让`gojsonq`可以处理其他格式的数据：

```golang
func main() {
  jq := gojsonq.New(gojsonq.SetDecoder(&yamlDecoder{})).File("./data.yaml")
  jq.From("items").Where("price", "<=", 500)
  fmt.Printf("%v\n", jq.First())
}

type yamlDecoder struct {
}

func (i *yamlDecoder) Decode(data []byte, v interface{}) error {
  bb, err := yaml.YAMLToJSON(data)
  if err != nil {
    return err
  }
  return json.Unmarshal(bb, &v)
}
```

上面代码用到了`yaml`库，需要额外安装：

```cmd
$ go get github.com/ghodss/yaml
```

解析器只要实现`gojsonq.Decoder`接口，都可以作为设置到`gojsonq`中，这样就可以实现任何格式的处理：

```golang
// src/github.com/thedevsaddam/gojsonq/decoder.go
type Decoder interface {
  Decode(data []byte, v interface{}) error
}
```

## 总结

`gojsonq`还有一些高级特性，例如自定义`Where`的操作类型，取第一个、最后一个、第 N 个值等。感兴趣可自行研究~

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gojsonq GitHub：[https://github.com/thedevsaddam/gojsonq](https://github.com/thedevsaddam/gojsonq)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)