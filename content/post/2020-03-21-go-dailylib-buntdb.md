---
layout:    post
title:    "Go 每日一库之 buntdb"
subtitle: 	"每天学习一个 Go 库"
date:		2020-03-21T20:27:24
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/03/21/godailylib/buntdb"
categories: [
	"Go"
]
---

## 简介

[`buntdb`](https://github.com/tidwall/buntdb)是一个完全用 Go 语言编写的内存键值数据库。它支持 ACID、并发读、自定义索引和空间信息数据。`buntdb`只用一个源码文件就实现了这些功能，对于想要学习数据库底层知识的童鞋更是不容错过。

感谢[@kiyonlin](https://github.com/kiyonlin)推荐！

## 快速使用

先安装：

```golang
$ go get github.com/tidwall/buntdb
```

后使用：

```golang
package main

import (
  "fmt"
  "log"

  "github.com/tidwall/buntdb"
)

func main() {
  db, err := buntdb.Open(":memory:")
  if err != nil {
    log.Fatal(err)
  }
  defer db.Close()

  db.Update(func(tx *buntdb.Tx) error {
    oldValue, replaced, err := tx.Set("testkey", "testvalue", nil)
    if err != nil {
      return err
    }

    fmt.Printf("old value:%q replaced:%t\n", oldValue, replaced)
    return nil
  })

  db.View(func(tx *buntdb.Tx) error {
    value, err := tx.Get("testkey")
    if err != nil {
      return err
    }

    fmt.Println("value is:", value)
    return nil
  })
}
```

`buntdb`在使用方式上与我们熟知的`sqlite`有些类似，只是前者支持的是键值对，后者支持的关系型数据。首先，我们要打开一个数据库，`buntdb`支持将数据存储到文件和内存，将数据保存在磁盘上的文件中，断电不会丢失。直接存放在内存中，程序退出后数据就丢失了。调用`buntdb.Open()`方法需要传入一个文件名的参数，指定数据保存的文件路径。如果传入特殊字符串`:memory:`，则`buntdb`不会将数据保存到磁盘。

在`buntdb`中，所有的读写操作都必须在一个事务中执行。同一时间只能存在一个写事务，但是可以同时存在多个并发的读事务。如果只需要读取数据，那么调用`db.View()`方法。方法接收一个类型为`func (tx *buntdb.Tx) error`的函数作为参数，`db.View()`方法内部会生成一个事务对象`tx`，然后将这个`tx`作为参数传给该函数。在此函数中使用事务对象`tx`的`Get()`方法执行读取的逻辑：

```golang
db.View(func(tx *buntdb.Tx) error {
  value, err := tx.Get("testkey")
  if err != nil {
    return err
  }

  fmt.Println("value is:", value)
  return nil
})
```

如果需要读写数据，那么使用`db.Update()`方法。同样地，也需要传入一个类型为`func (tx *buntdb.Tx) error`的函数，在此函数中使用事务对象`tx`的`Set`方法执行写入逻辑。`tx.Set()`方法返回 3 个值。如果`Set()`替换了当前值，则返回替换之前的值和`true`。如果此函数返回非空错误，`db.Update()`会回退此前所做的修改，反之会提交此次修改。

如果运行两次上面的程序，我们会看到下面的输出：

```golang
// 第一次运行
$ go run main.go 
old value:"" replaced:false
value is: testvalue

// 第二次运行
$ go run main.go 
old value:"testvalue" replaced:true
value is: testvalue
```

注意：

* 数据库操作很容易出错，所以基本上所有的方法都会返回错误，在实际中需要处理每个可能的错误。示例中为了代码简洁，有点地方忽略了；
* 在传入`db.View()`和`db.Update()`的函数中不要直接使用`db`对象，否则可能会导致程序死锁；
* 默认情况下，若键对应的值不存在，则返回`ErrNotFound`错误。

## 遍历

`buntdb`中存储的数据是根据键排序的，我们可以按顺序依次遍历这些数据。由于遍历是读取操作，我们用`db.View()`方法。`buntdb`提供了很多遍历的方法，基本形式都差不多，这里只介绍一个基本的`Ascend()`方法：

```golang
func (tx *Tx) Ascend(index string, iterator func(key, value string) bool) error
```

`Ascend()`方法接收一个索引名，然后以该索引定义的顺序遍历所有键值对，将遍历到的键值对传给`iterator`函数处理，如果`iterator`返回`false`，终止遍历。另外，如果未指定索引名，则根据键升序遍历：

```golang
func main() {
  db, err := buntdb.Open(":memory:")
  if err != nil {
    log.Fatal(err)
  }
  defer db.Close()

  db.Update(func(tx *buntdb.Tx) error {
    data := map[string]string{
      "a": "apple",
      "b": "banana",
      "p": "pear",
      "o": "orange",
    }
    for key, value := range data {
      tx.Set(key, value, nil)
    }
    return nil
  })

  db.View(func(tx *buntdb.Tx) error {
    var count int
    tx.Ascend("", func(key, value string) bool {
      fmt.Printf("key:%s value:%s\n", key, value)
      count++
      if count >= 3 {
        return false
      }
      return true
    })
    return nil
  })
}
```

上面代码中，我们按键升序遍历（因为传入索引名为`""`），在处理完第三个键值对后，`iterator`函数返回`false`，停止遍历。最终输出：

```golang
key:a value:apple
key:b value:banana
key:o value:orange
```

## 索引

`buntdb`将所有数据都存储在一个**B-tree**中，每组数据都有一个键和值。所有数据是根据键来排序的。我们也可以创建自定义索引，这样就可以对值进行排序了。创建索引需要调用`db.CreateIndex()`方法，该方法签名如下：

```golang
func (db *DB) CreateIndex(name, pattern string, less ...func(a, b string) bool) error
```

`name`为索引名，在上一节介绍遍历的时候，我们说过遍历时需要传入索引名，以便按照该索引所定义的顺序遍历。`pattern`为模式，指定索引对哪些键生效，可以只对某些特定模式的键创建索引。`*`表示所有键，`user:*:name`表示键名是`user:`和`:name`之间有任意字符的键。通过`less`函数，我们可以自定义排序规则。`buntdb`内置了一些排序规则，如`IndexString`对值进行大小写不敏感的排序，`IndexInt/IndexUint/IndexFloat`执行数值类型的排序。

```golang
func main() {
  db, err := buntdb.Open(":memory:")
  if err != nil {
    log.Fatal(err)
  }
  defer db.Close()

  db.CreateIndex("names", "user:*:name", buntdb.IndexString)
  db.Update(func(tx *buntdb.Tx) error {
    tx.Set("user:1:name", "tom", nil)
    tx.Set("user:2:name", "Randi", nil)
    tx.Set("user:3:name", "jane", nil)
    tx.Set("user:4:name", "Janet", nil)
    tx.Set("user:5:name", "Paula", nil)
    tx.Set("user:6:name", "peter", nil)
    tx.Set("user:7:name", "Terri", nil)
    return nil
  })

  db.View(func(tx *buntdb.Tx) error {
    tx.Ascend("names", func(key, value string) bool {
      fmt.Printf("%s: %s\n", key, value)
      return true
    })
    return nil
  })
}
```

我们先为键名满足模式`user:*:name`的数据创建一个名为`names`的索引，执行大小写不敏感的排序（`buntdb.IndexString`）。然后向`buntdb`中写入几组数据。最后，我们使用`Ascend()`方法，传入索引名`names`按该索引指定次序遍历键值对（这里只是遍历满足模式`user:*:name`的键值对）。

如果我们的键只有`user:*:name`这种模式的，也可以直接使用模式`*`或`user:*`

对于整数等非字符串类型的排序，我们需要注意一点：因为`buntdb`存储的键值都是字符串，所以自定义的排序函数需要执行相应的类型转换。一般需求的数值排序，内置函数就可以满足要求了：

```golang
func main() {
  db, err := buntdb.Open(":memory:")
  if err != nil {
    log.Fatal(err)
  }
  defer db.Close()

  db.CreateIndex("ages", "user:*:age", buntdb.IndexInt)
  db.Update(func(tx *buntdb.Tx) error {
    tx.Set("user:1:age", "16", nil)
    tx.Set("user:2:age", "35", nil)
    tx.Set("user:3:age", "24", nil)
    tx.Set("user:4:age", "32", nil)
    tx.Set("user:5:age", "25", nil)
    tx.Set("user:6:age", "28", nil)
    tx.Set("user:7:age", "31", nil)
    return nil
  })

  db.View(func(tx *buntdb.Tx) error {
    tx.Ascend("ages", func(key, value string) bool {
      fmt.Printf("%s: %s\n", key, value)
      return true
    })
    return nil
  })
}
```

首先，为键名满足`user:*:age`的键创建索引`ages`，因为在这些键对应的值中，我们存储的都是年龄（整数），故使用排序规则`IndexInt`。

### JSON 索引

`buntdb`提供了强大的 JSON 索引功能。如果存储的值是一个 JSON 字符串，`buntdb`可以对 JSON 串内部的键创建索引。`buntdb.IndexJSON()`实现了 JSON 索引的排序规则，我们需要传入键在 JSON 内部的路径，如`name.first`，`contact.email`等：

```golang
func main() {
  db, _ := buntdb.Open(":memory:")
  defer db.Close()

  db.CreateIndex("first_name", "user:*", buntdb.IndexJSON("name.first"))
  db.CreateIndex("age", "user:*", buntdb.IndexJSON("age"))
  db.Update(func(tx *buntdb.Tx) error {
    tx.Set("user:1", `{"name":{"first":"zhang","last":"san"},"age":18}`, nil)
    tx.Set("user:2", `{"name":{"first":"li","last":"si"},"age":27`, nil)
    tx.Set("user:3", `{"name":{"first":"wang","last":"wu"},"age":32}`, nil)
    tx.Set("user:4", `{"name":{"first":"sun","last":"qi"},"age":8}`, nil)
    return nil
  })

  db.View(func(tx *buntdb.Tx) error {
    fmt.Println("Order by first name")
    tx.Ascend("first_name", func(key, value string) bool {
      fmt.Printf("%s: %s\n", key, value)
      return true
    })

    fmt.Println("Order by age")
    tx.Ascend("age", func(key, value string) bool {
      fmt.Printf("%s: %s\n", key, value)
      return true
    })

    fmt.Println("Order by age range 18-30")
    tx.AscendRange("age", `{"age":18}`, `{"age":30}`, func(key, value string) bool {
      fmt.Printf("%s: %s\n", key, value)
      return true
    })
    return nil
  })
}
```

JSON 给我们提供了一种很好的存储用户数据的格式。以`user:`后加上用户 ID 作为键名，用户数据以 JSON 格式存储在值中，如上所示。

我们分别为 JSON 内部的键`name.first`和`age`创建索引。然后分别以`name.first`和`age`定义的顺序遍历输出。值得一提的是最后一个遍历使用了`AscendRange`，可以只遍历指定范围内的数据，例子中为年龄在 18~30 之间。范围遍历并非 JSON 索引独有的，与普通的`Ascend`相比，`AscendRange`需要传入区间上下限`min`和`max`，所有处于`[min, max)`之间的数据都会被遍历到（注意不包含`max`）。

### 多重索引

细节的盆友应该发现了，创建索引的方法`CreateIndex()`接受可变数量的排序规则函数，如果第一个函数无法判断两个值的大小，则继续使用后一个函数，直到可以判断或没有其他函数了。这个就是多重索引。在上面的示例中，我们可以将`first_name`和`age`两个索引放在一起，先对`name.first`比较，如果相等，再比较`age`：

```golang
func main() {
  db, _ := buntdb.Open(":memory:")
  defer db.Close()

  db.CreateIndex("first_name_age", "user:*", buntdb.IndexJSON("name.first"), buntdb.IndexJSON("age"))
  db.Update(func(tx *buntdb.Tx) error {
    tx.Set("user:1", `{"name":{"first":"zhang","last":"san"},"age":18}`, nil)
    tx.Set("user:2", `{"name":{"first":"li","last":"si"},"age":27`, nil)
    tx.Set("user:3", `{"name":{"first":"wang","last":"wu"},"age":30}`, nil)
    tx.Set("user:4", `{"name":{"first":"sun","last":"qi"},"age":8}`, nil)
    tx.Set("user:5", `{"name":{"first":"li", "name":"dajun"},"age":20}`, nil)
    return nil
  })

  db.View(func(tx *buntdb.Tx) error {
    tx.Ascend("first_name_age", func(key, value string) bool {
      fmt.Printf("%s: %s\n", key, value)
      return true
    })
    return nil
  })
}
```

由于`user:2`和`user:5`的`name.first`都是`li`，相等。故使用`age`的值排序，所以输出中`user:5`在`user:2`前面。

### 降序

我们使用的内置函数都是升序规则。可以使用`buntdb.Desc()`将升序规则变为降序，拿前面整数排序的例子来说，只需要将`buntdb.IndexInt`变为`buntdb.Desc(buntdb.IndexInt)`即可：

```golang
func main() {
  db, _ := buntdb.Open(":memory:")
  defer db.Close()

  db.CreateIndex("ages", "user:*:age", buntdb.Desc(buntdb.IndexInt))
  db.Update(func(tx *buntdb.Tx) error {
    tx.Set("user:1:age", "16", nil)
    tx.Set("user:2:age", "35", nil)
    tx.Set("user:3:age", "24", nil)
    tx.Set("user:4:age", "32", nil)
    tx.Set("user:5:age", "25", nil)
    tx.Set("user:6:age", "28", nil)
    tx.Set("user:7:age", "31", nil)
    return nil
  })

  db.View(func(tx *buntdb.Tx) error {
    tx.Ascend("ages", func(key, value string) bool {
      fmt.Printf("%s: %s\n", key, value)
      return true
    })
    return nil
  })
}
```

## 过期

在向`buntdb`中设置键值时，我们可以通过选项`buntdb.SetOptions`指定过期时间，超过这个时间数据会自动从`buntdb`中移除。如果想要移除过期时间，重新使用`nil`选项设置该键值即可：

```golang
func main() {
  db, _ := buntdb.Open(":memory:")
  defer db.Close()

  db.Update(func(tx *buntdb.Tx) error {
    tx.Set("testkey", "testvalue", &buntdb.SetOptions{Expires: true, TTL: time.Second})
    return nil
  })

  db.View(func(tx *buntdb.Tx) error {
    value, _ := tx.Get("testkey")
    fmt.Println("value is:", value)
    return nil
  })

  time.Sleep(time.Second)

  db.View(func(tx *buntdb.Tx) error {
    value, _ := tx.Get("testkey")
    fmt.Println("value is:", value)
    return nil
  })
}
```

上面例子中，我们先写入数据，并设置过期时间为`1s`。然后立刻读取，这时可以读到刚刚设置的值。然后`Sleep` 1s 之后再次读取，读到空值，说明已被删除：

```golang
value is: testvalue
value is: 
```

## 杂项

### 遍历时删除

`buntdb`不支持遍历时删除数据，一般迂回的做法是先记录需要删除的键，遍历结束后统一删除。下面将年龄 >= 30 的用户删掉（嗯，程序员年龄大了，干不动了）：

```golang
func main() {
  db, _ := buntdb.Open(":memory:")
  defer db.Close()

  db.Update(func(tx *buntdb.Tx) error {
    tx.Set("user:1:age", "16", nil)
    tx.Set("user:2:age", "35", nil)
    tx.Set("user:3:age", "24", nil)
    tx.Set("user:4:age", "32", nil)
    tx.Set("user:5:age", "25", nil)
    tx.Set("user:6:age", "28", nil)
    tx.Set("user:7:age", "31", nil)
    return nil
  })

  db.Update(func(tx *buntdb.Tx) error {
    // 先汇总
    deleteKeys := make([]string, 0)
    tx.Ascend("", func(key, value string) bool {
      age, _ := strconv.ParseUint(value, 10, 64)
      if age >= 30 {
        deleteKeys = append(deleteKeys, key)
      }
      return true
    })

    // 再删除
    for _, key := range deleteKeys {
      tx.Delete(key)
    }
    return nil
  })

  db.View(func(tx *buntdb.Tx) error {
    tx.Ascend("", func(key, value string) bool {
      fmt.Printf("%s: %s\n", key, value)
      return true
    })
    return nil
  })
}
```

## Web 服务

`buntdb`只能在本地程序中操作，我们简单为它编写一个 Web 服务，可以通过 HTTP 请求操作远程的`buntdb`。代码如下：

```golang
package main

import (
  "encoding/json"
  "fmt"
  "log"
  "net/http"
  "strconv"
  "time"

  "github.com/tidwall/buntdb"
)

var db *buntdb.DB

func init() {
  var err error
  db, err = buntdb.Open("data.db")
  if err != nil {
    log.Fatal(err)
  }
}

func response(w http.ResponseWriter, err error, data interface{}) {
  bytes, _ := json.Marshal(map[string]interface{}{
    "error": err,
    "data":  data,
  })
  w.Write(bytes)
}

func set(w http.ResponseWriter, r *http.Request) {
  key := r.FormValue("key")
  value := r.FormValue("value")
  expire, _ := strconv.ParseBool(r.FormValue("expire"))
  ttl, _ := time.ParseDuration(r.FormValue("ttl"))

  var setOption *buntdb.SetOptions
  if expire && ttl > 0 {
    setOption = &buntdb.SetOptions{Expires: true, TTL: ttl}
  }

  err := db.Update(func(tx *buntdb.Tx) error {
    _, _, err := tx.Set(key, value, setOption)
    return err
  })

  response(w, err, nil)
}

func get(w http.ResponseWriter, r *http.Request) {
  key := r.FormValue("key")

  var value string
  err := db.View(func(tx *buntdb.Tx) error {
    var err error
    value, err = tx.Get(key)
    return err
  })

  response(w, err, value)
}

type Pair struct {
  Key   string
  Value string
}

func iterate(w http.ResponseWriter, r *http.Request) {
  index := r.FormValue("index")
  fmt.Println(index)

  var items []Pair
  err := db.View(func(tx *buntdb.Tx) error {
    err := tx.Ascend(index, func(key, value string) bool {
      fmt.Println(key, value)
      items = append(items, Pair{key, value})
      return true
    })
    return err
  })

  response(w, err, items)
}

func createIndex(w http.ResponseWriter, r *http.Request) {
  name := r.FormValue("name")
  pattern := r.FormValue("pattern")
  less := buntdb.IndexString

  err := db.CreateIndex(name, pattern, less)
  response(w, err, nil)
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/get", get)
  mux.HandleFunc("/set", set)
  mux.HandleFunc("/iterate", iterate)
  mux.HandleFunc("/create_index", createIndex)

  server := &http.Server{
    Addr:    ":8000",
    Handler: mux,
  }

  if err := server.ListenAndServe(); err != nil {
    log.Fatal(err)
  }
}
```

我只编写了基本读取、设置、创建索引和遍历的功能，代码并不难理解。下面我们先运行程序，然后用浏览器请求：

请求`localhost:8000/set?key=name&value=dj`，返回：

```golang
{"error":null, "data":null}
```

`error`为`null`表示无错误。

请求`localhost:8000/set?key=dj&value=18`，返回：

```golang
{"error":null, "data":null}
```

请求`localhost:8000/iterate`，返回：

```golang
{
  "data": [
    {
      "Key": "age",
      "Value": "18"
    },
    {
      "Key": "name",
      "Value": "dj"
    }
  ],
  "error": null
}
```

感兴趣可以试着添加更多的功能。如果对 Go Web 编程不太了解，可以去看看我的[Go Web 编程](https://darjun.github.io/tags/go-web-%E7%BC%96%E7%A8%8B/)系列文章。

## 总结

本文介绍`buntdb`的读取、写入、创建索引等基本操作，最后编写一个简单的 web 服务可以在远程运行，其他程序通过 HTTP 与之交互。`buntdb`还支持空间索引等高级特性，感兴趣可自行研究。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. buntdb GitHub：[https://github.com/tidwall/buntdb](https://github.com/tidwall/buntdb)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)