---
layout:    post
title:    "Go 每日一库之 gabs"
subtitle: 	"每天学习一个 Go 库"
date:		2020-09-03T21:10:09
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/09/03/godailylib/gabs"
categories: [
	"Go"
]
---

## 简介

JSON 是一种非常流行的数据交换格式。每种编程语言都有很多操作 JSON 的库，标准库、第三方库都有。Go 语言中标准库内置了 JSON 操作库`encoding/json`。我们之前也介绍过专门用于**查询** JSON 串的库[gjson](https://darjun.github.io/2020/03/22/godailylib/gjson/)和专门用于**修改** JSON 串的库[sjson](https://darjun.github.io/2020/03/24/godailylib/sjson)，还有一个非常方便的操作 JSON 数据的命令行工具[jj](https://darjun.github.io/2020/03/25/godailylib/jj/)。今天我们再介绍一个 JSON 工具库——[`gabs`](https://github.com/Jeffail/gabs)。`gabs`是一个用来查询和修改 JSON 串的库。它使用`encoding/json`将一般的 JSON 串转为`map[string]interface{}`，并提供便利的方法操作`map[string]struct{}`。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir gabs && cd gabs
$ go mod init github.com/darjun/go-daily-lib/gabs
```

安装`gabs`，目前最新版本为`v2`，推荐使用`v2`：

```cmd
$ go get -u github.com/Jeffail/gabs/v2
```

使用：

```golang
package main

import (
  "github.com/Jeffail/gabs/v2"
  "fmt"
)

func main() {
  jObj, _ := gabs.ParseJSON([]byte(`{
    "info": {
      "name": {
        "first": "lee",
        "last": "darjun"
      },
      "age": 18,
      "hobbies": [
        "game",
        "programming"
      ]
    }
    }`))

  fmt.Println("first name: ", jObj.Search("info", "name", "first").Data().(string))
  fmt.Println("second name: ", jObj.Path("info.name.last").Data().(string))
  gObj, _ := jObj.JSONPointer("/info/age")
  fmt.Println("age: ", gObj.Data().(float64))
  fmt.Println("one hobby: ", jObj.Path("info.hobbies.1").Data().(string))
}
```

首先，我们调用`gabs.ParseJSON()`方法解析传入的 JSON 串，得到一个`gabs.Container`对象。后续通过该`gabs.Container`对象来查询和修改解析出来的数据。

`gabs`提供 3 中查询方式：

* 以`.`分隔的路径调用`Path()`方法；
* 将路径各个部分作为可变参数传入`Search()`方法；
* 使用`/`分隔的路径调用`JSONPointer()`方法。

上述方法内部实现最终都是调用相同的方法，只是使用上稍微有些区别。注意：

* 3 个方法最终都返回一个`gabs.Container`对象，我们需要调用其`Data()`获取内部的数据，然后做一次类型转换得到实际的数据；
* 如果传入的路径有误或路径下无数据，则`Search/Path`方法返回的`gabs.Container`对象内部数据为`nil`，即`Data()`方法返回`nil`，而`JSONPointer`方法返回`err`。实际使用时注意进行空指针和错误判断；
* 如果路径某个部分对应的数据类型为数组，则可以在后面追加索引，读取对应索引下的数据，如`info.hobbies.1`；
* `JSONPointer()`参数必须以`/`开头。

运行结果：

```cmd
$ go run main.go
first name:  lee
second name:  darjun
age:  18
one hobby:  programming
```

## 查询 JSON 串

上一节中我们介绍过，在`gabs`中，路径有 3 种表示方式。这 3 种方式对应 3 个基础的查询方法：

* `Search(hierarchy ...string)`：也有一个简写形式`S`；
* `Path(path string)`：`path`以`.`分隔；
* `JSONPointer(path string)`：`path`以`/`分隔。

它们的基本用法上面已经介绍过了，对于数组我们还能对每个数组元素做递归查询。在下面例子中，我们依次返回数组`members`中每个元素的`name`、`age`和`relation`字段：

```golang
func main() {
  jObj, _ := gabs.ParseJSON([]byte(`{
    "user": {
      "name": "dj",
      "age": 18,
      "members": [
        {
          "name": "hjw",
          "age": 20,
          "relation": "spouse"
        },
        {
          "name": "lizi",
          "age": 3,
          "relation": "son"
        }
      ]
    }
  }`))

  fmt.Println("member names: ", jObj.S("user", "members", "*", "name").Data())
  fmt.Println("member ages: ", jObj.S("user", "members", "*", "age").Data())
  fmt.Println("member relations: ", jObj.S("user", "members", "*", "relation").Data())

  fmt.Println("spouse name: ", jObj.S("user", "members", "0", "name").Data().(string))
}
```

运行程序，输出：

```cmd
$ go run main.go
member names:  [hjw lizi]
member ages:  [20 3]
member relations:  [spouse son]
spouse name:  hjw
```

容易看出，在路径中遇到数组分下面两种情况处理：

* 下一个部分路径是`*`，则对所有的数组元素应用剩余的路径查询，结果放在一个数组中返回；
* 否则，下一个路径部分必须是数组索引，对该索引所在元素应用剩余的路径查询。

查看源码我们可以知道，实际上，`Path/JSONPointer`内部都是先将`path`解析为`hierarchy ...string`的形式，最终都会调用`searchStrict`方法：

```golang
func (g *Container) Search(hierarchy ...string) *Container {
  c, _ := g.searchStrict(true, hierarchy...)
  return c
}

func (g *Container) Path(path string) *Container {
  return g.Search(DotPathToSlice(path)...)
}

func (g *Container) JSONPointer(path string) (*Container, error) {
  hierarchy, err := JSONPointerToSlice(path)
  if err != nil {
    return nil, err
  }
  return g.searchStrict(false, hierarchy...)
}

func (g *Container) S(hierarchy ...string) *Container {
  return g.Search(hierarchy...)
}
```

`searchStrict`方法也不复杂，我们简单看一下：

```golang
func (g *Container) searchStrict(allowWildcard bool, hierarchy ...string) (*Container, error) {
  object := g.Data()
  for target := 0; target < len(hierarchy); target++ {
    pathSeg := hierarchy[target]
    if mmap, ok := object.(map[string]interface{}); ok {
      object, ok = mmap[pathSeg]
      if !ok {
        return nil, fmt.Errorf("failed to resolve path segment '%v': key '%v' was not found", target, pathSeg)
      }
    } else if marray, ok := object.([]interface{}); ok {
      if allowWildcard && pathSeg == "*" {
        tmpArray := []interface{}{}
        for _, val := range marray {
          if (target + 1) >= len(hierarchy) {
            tmpArray = append(tmpArray, val)
          } else if res := Wrap(val).Search(hierarchy[target+1:]...); res != nil {
            tmpArray = append(tmpArray, res.Data())
          }
        }
        if len(tmpArray) == 0 {
          return nil, nil
        }
        return &Container{tmpArray}, nil
      }
      index, err := strconv.Atoi(pathSeg)
      if err != nil {
        return nil, fmt.Errorf("failed to resolve path segment '%v': found array but segment value '%v' could not be parsed into array index: %v", target, pathSeg, err)
      }
      if index < 0 {
        return nil, fmt.Errorf("failed to resolve path segment '%v': found array but index '%v' is invalid", target, pathSeg)
      }
      if len(marray) <= index {
        return nil, fmt.Errorf("failed to resolve path segment '%v': found array but index '%v' exceeded target array size of '%v'", target, pathSeg, len(marray))
      }
      object = marray[index]
    } else {
      return nil, fmt.Errorf("failed to resolve path segment '%v': field '%v' was not found", target, pathSeg)
    }
  }
  return &Container{object}, nil
}
```

实际上就是顺着路径一层层往下走，遇到数组。如果下一个部分是通配符`*`，下面是处理代码：

```golang
tmpArray := []interface{}{}
for _, val := range marray {
  if (target + 1) >= len(hierarchy) {
    tmpArray = append(tmpArray, val)
  } else if res := Wrap(val).Search(hierarchy[target+1:]...); res != nil {
    tmpArray = append(tmpArray, res.Data())
  }
}
if len(tmpArray) == 0 {
  return nil, nil
}
return &Container{tmpArray}, nil
```

如果`*`是路径最后一个部分，返回所有数组元素：

```golang
if (target + 1) >= len(hierarchy) {
  tmpArray = append(tmpArray, val)
}
```

否则，应用剩余的路径查询每个元素，查询结果`append`到待返回切片中：

```golang
else if res := Wrap(val).Search(hierarchy[target+1:]...); res != nil {
  tmpArray = append(tmpArray, res.Data())
}
```

另一方面，如果不是通配符，那么下一个路径部分必须是索引，取这个索引的元素，继续往下查询：

```golang
index, err := strconv.Atoi(pathSeg)
```

### 遍历

`gabs`提供了两个方法可以方便地遍历数组和对象：

* `Children()`：返回所有数组元素的切片，如果在对象上调用该方法，`Children()`将以不确定顺序返回对象所有的值的切片；
* `ChildrenMap()`：返回对象的键和值。

看示例：

```golang
func main() {
  jObj, _ := gabs.ParseJSON([]byte(`{
    "user": {
      "name": "dj",
      "age": 18,
      "members": [
        {
          "name": "hjw",
          "age": 20,
          "relation": "spouse"
        },
        {
          "name": "lizi",
          "age": 3,
          "relation": "son"
        }
      ]
    }
  }`))

  for k, v := range jObj.S("user").ChildrenMap() {
    fmt.Printf("key: %v, value: %v\n", k, v)
  }

  fmt.Println()

  for i, v := range jObj.S("user", "members", "*").Children() {
    fmt.Printf("member %d: %v\n", i+1, v)
  }
}
```

运行结果：

```cmd
$ go run main.go
key: name, value: "dj"
key: age, value: 18
key: members, value: [{"age":20,"name":"hjw","relation":"spouse"},{"age":3,"name":"lizi","relation":"son"}]

member 1: {"age":20,"name":"hjw","relation":"spouse"}
member 2: {"age":3,"name":"lizi","relation":"son"}
```

这两个方法的源码很简单，建议去看看~

### 存在性判断

`gabs`提供了两个方法检查对应的路径上是否存在数据：

* `Exists(hierarchy ...string)`；
* `ExistsP(path string)`：方法名以`P`结尾，表示接受以`.`分隔的路径。

看示例：

```golang
func main() {
  jObj, _ := gabs.ParseJSON([]byte(`{"user":{"name": "dj","age": 18}}`))
  fmt.Printf("has name? %t\n", jObj.Exists("user", "name"))
  fmt.Printf("has age? %t\n", jObj.ExistsP("user.age"))
  fmt.Printf("has job? %t\n", jObj.Exists("user", "job"))
}
```

运行：

```cmd
$ go run main.go
has name? true
has age? true
has job? false
```

### 获取数组信息

对于类型为数组的值，`gabs`提供了几组便捷的查询方法。

* 获取数组大小：`ArrayCount/ArrayCountP`，不加后缀的方法接受可变参数作为路径，以`P`为后缀的方法需要传入`.`分隔的路径；
* 获取数组某个索引的元素：`ArrayElement/ArrayElementP`。

示例：

```golang
func main() {
  jObj, _ := gabs.ParseJSON([]byte(`{
    "user": {
      "name": "dj",
      "age": 18,
      "members": [
        {
          "name": "hjw",
          "age": 20,
          "relation": "spouse"
        },
        {
          "name": "lizi",
          "age": 3,
          "relation": "son"
        }
      ],
      "hobbies": ["game", "programming"]
    }
  }`))

  cnt, _ := jObj.ArrayCount("user", "members")
  fmt.Println("member count:", cnt)
  cnt, _ = jObj.ArrayCount("user", "hobbies")
  fmt.Println("hobby count:", cnt)

  ele, _ := jObj.ArrayElement(0, "user", "members")
  fmt.Println("first member:", ele)
  ele, _ = jObj.ArrayElement(1, "user", "hobbies")
  fmt.Println("second hobby:", ele)
}
```

输出：

```cmd
member count: 2
hobby count: 2
first member: {"age":20,"name":"hjw","relation":"spouse"}
second hobby: "programming"
```

## 修改和删除

我们可以使用`gabs`构造一个 JSON 串。根据要设置的值的类型，`gabs`将修改的方法又分为了两类：原始值、数组和对象。基本操作流程是相同的：

* 调用`gabs.New()`创建`gabs.Container`对象，或者`ParseJSON()`从现有 JSON 串中解析出`gabs.Container`对象；
* 调用方法设置或修改键值，也可以删除一些键；
* 生成最终的 JSON 串。

### 原始值

我们前面说过，`gabs`使用三种方式来表达路径。在设置时也可以通过这三种方式指定在什么位置设置值。对应方法为：

* `Set(value interface{}, hierarchy ...string)`：将路径各个部分作为可变参数传入即可；
* `SetP(value interface{}, path string)`：路径各个部分以点`.`分隔；
* `SetJSONPointer(value interface{}, path string)`：路径各个部分以`/`分隔，且必须以`/`开头。

示例：

```golang
func main() {
  gObj := gabs.New()

  gObj.Set("lee", "info", "name", "first")
  gObj.SetP("darjun", "info.name.last")
  gObj.SetJSONPointer(18, "/info/age")

  fmt.Println(gObj.String())
}
```

最终生成 JSON 串：

```cmd
$ go run main.go
{"info":{"age":18,"name":{"first":"lee","last":"darjun"}}}
```

我们也可以调用`gabs.Container`的`StringIndent`方法增加前缀和缩进，让输出更美观些：

```golang
fmt.Println(gObj.StringIndent("", "  "))
```

观察输出变化：

```cmd
$ go run main.go
{
  "info": {
    "age": 18,
    "name": {
      "first": "lee",
      "last": "darjun"
    }
  }
}
```

### 数组

相比原始值，数组的操作复杂不少。我们可以创建新的数组，也可以在原有的数组中添加、删除元素。

```golang
func main() {
  gObj := gabs.New()

  arrObj1, _ := gObj.Array("user", "hobbies")
  fmt.Println(arrObj1.String())

  arrObj2, _ := gObj.ArrayP("user.bugs")
  fmt.Println(arrObj2.String())

  gObj.ArrayAppend("game", "user", "hobbies")
  gObj.ArrayAppend("programming", "user", "hobbies")

  gObj.ArrayAppendP("crash", "user.bugs")
  gObj.ArrayAppendP("panic", "user.bugs")
  fmt.Println(gObj.String())
}
```

我们先通过`Array/ArrayP`分别在路径`user.hobbies`和`user.bugs`下创建数组，然后调用`ArrayAppend/ArrayAppendP`向这两个数组中添加元素。现在我们应该可以根据方法有无后缀，后缀是什么来区分它接受什么格式的路径了！

运行结果：

```cmd
{"user":{"bugs":["crash","panic"],"hobbies":["game","programming"]}}
```

实际上，我们甚至可以省略上面的数组创建过程，因为`ArrayAppend/ArrayAppendP`如果检测到中间路径上没有值，会自动创建对象。

当然我们也可以删除某个索引的数组元素，使用`ArrayRemove/ArrayRemoveP`方法：

```golang
func main() {
  jObj, _ := gabs.ParseJSON([]byte(`{"user":{"bugs":["crash","panic"],"hobbies":["game","programming"]}}`))

  jObj.ArrayRemove(0, "user", "bugs")
  jObj.ArrayRemoveP(1, "user.hobbies")
  fmt.Println(jObj.String())
}
```

删除完成之后还剩下：

```
{"user":{"bugs":["panic"],"hobbies":["game"]}}
```

### 对象

在指定路径下创建对象使用`Object/ObjectI/ObjectP`这组方法，其中`ObjectI`是指在数组的特定索引下创建。一般地我们使用`Set`类方法就足够了，中间路径不存在会自动创建。

对象删除使用`Delete/DeleteP`这组方法：

```golang
func main() {
  jObj, _ := gabs.ParseJSON([]byte(`{"info":{"age":18,"name":{"first":"lee","last":"darjun"}}}`))

  jObj.Delete("info", "name")
  fmt.Println(jObj.String())

  jObj.Delete("info")
  fmt.Println(jObj.String())
}
```

输出：

```cmd
{"info":{"age":18}}
{}
```

## `Flatten`

`Flatten`操作即将嵌套很深的字段提到最外层，`gabs.Flatten`返回一个新的`map[string]interface{}`，`interface{}`为 JSON 中叶子节点的值，键为该叶子的路径。例如：`{"foo":[{"bar":"1"},{"bar":"2"}]}`执行 flatten 操作之后返回`{"foo.0.bar":"1","foo.1.bar":"2"}`。

```golang
func main() {
  jObj, _ := gabs.ParseJSON([]byte(`{
    "user": {
      "name": "dj",
      "age": 18,
      "members": [
        {
          "name": "hjw",
          "age": 20,
          "relation": "spouse"
        },
        {
          "name": "lizi",
          "age": 3,
          "relation": "son"
        }
      ],
      "hobbies": ["game", "programming"]
    }
  }`))

  obj, _ := jObj.Flatten()
  fmt.Println(obj)
}
```

输出：

```cmd
map[user.age:18 user.hobbies.0:game user.hobbies.1:programming user.members.0.age:20 user.members.0.name:hjw user.members.0.relation:spouse user.members.1.age:3 user.members.1.name:lizi user.members.1.relation:son user.name:dj]
```

## 合并

我们可以将两个`gabs.Container`合并成一个。如果同一个路径下有相同的键：

* 如果两者都是对象类型，则对二者进行合并操作；
* 如果两者都是数组类型，则将后者中所有元素追加到前一个数组中；
* 其中一个为数组，合并之后另一个同名键的值将会作为元素添加到数组中。

例如：

```golang
func main() {
  obj1, _ := gabs.ParseJSON([]byte(`{"user":{"name":"dj"}}`))
  obj2, _ := gabs.ParseJSON([]byte(`{"user":{"age":18}}`))
  obj1.Merge(obj2)
  fmt.Println(obj1)

  arr1, _ := gabs.ParseJSON([]byte(`{"user": {"hobbies": ["game"]}}`))
  arr2, _ := gabs.ParseJSON([]byte(`{"user": {"hobbies": ["programming"]}}`))
  arr1.Merge(arr2)
  fmt.Println(arr1)

  obj3, _ := gabs.ParseJSON([]byte(`{"user":{"name":"dj", "hobbies": "game"}}`))
  arr3, _ := gabs.ParseJSON([]byte(`{"user": {"hobbies": ["programming"]}}`))
  obj3.Merge(arr3)
  fmt.Println(obj3)

  obj4, _ := gabs.ParseJSON([]byte(`{"user":{"name":"dj", "hobbies": "game"}}`))
  arr4, _ := gabs.ParseJSON([]byte(`{"user": {"hobbies": ["programming"]}}`))
  arr4.Merge(obj4)
  fmt.Println(arr4)

  obj5, _ := gabs.ParseJSON([]byte(`{"user":{"name":"dj", "hobbies": {"first": "game"}}}`))
  arr5, _ := gabs.ParseJSON([]byte(`{"user": {"hobbies": ["programming"]}}`))
  obj5.Merge(arr5)
  fmt.Println(obj5)
}
```

看结果：

```cmd
{"user":{"age":18,"name":"dj"}}
{"user":{"hobbies":["game","programming"]}}
{"user":{"hobbies":["game","programming"],"name":"dj"}}
{"user":{"hobbies":["programming","game"],"name":"dj"}}
{"user":{"hobbies":[{"first":"game"},"programming"],"name":"dj"}}
```

## 总结

`gabs`是一个十分方便的操作 JSON 的库，非常易于使用，而且代码实现比较简洁，值得一看。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gabs GitHub：[https://github.com/Jeffail/gabs](https://github.com/Jeffail/gabs)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)