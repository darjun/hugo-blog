---
layout:    post
title:    "Go 每日一库之 nutsdb"
subtitle: 	"每天学习一个 Go 库"
date:		2020-04-25T23:40:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/04/25/godailylib/nutsdb"
categories: [
	"Go"
]
---

## 简介

[`nutsdb`](https://github.com/xujiajun/nutsdb)是一个完全由 Go 编写的简单、快速、可嵌入的持久化存储。`nutsdb`与我们之前介绍过的[`buntdb`](https://darjun.github.io/2020/03/21/godailylib/buntdb/)有些类似，但是支持`List`、`Set`、`Sorted Set`这些数据结构。

## 快速使用

先安装：

```golang
$ go get github.com/xujiajun/nutsdb
```

后使用：

```golang
package main

import (
  "fmt"
  "log"

  "github.com/xujiajun/nutsdb"
)

func main() {
  opt := nutsdb.DefaultOptions
  opt.Dir = "./nutsdb"
  db, err := nutsdb.Open(opt)
  if err != nil {
    log.Fatal(err)
  }
  defer db.Close()

  err = db.Update(func(tx *nutsdb.Tx) error {
    key := []byte("name")
    val := []byte("dj")
    if err := tx.Put("", key, val, 0); err != nil {
      return err
    }
    return nil
  })
  if err != nil {
    log.Fatal(err)
  }

  err = db.View(func(tx *nutsdb.Tx) error {
    key := []byte("name")
    if e, err := tx.Get("", key); err != nil {
      return err
    } else {
      fmt.Println(string(e.Value))
    }
    return nil
  })
  if err != nil {
    log.Fatal(err)
  }
}
```

看过前面介绍`buntdb`文章的小伙伴会发现，`nutsdb`的简单使用与`buntdb`非常相似。首先打开数据库`nutsdb.Open()`，通过选项指定数据库文件存放目录。数据的插入、修改和查找都是包装在一个事务方法中执行的。`nutsdb`允许同时存在多个读事务。但是有写事务存在时，其他事务不能并发执行。需要修改数据的操作在`db.Update()`的回调中执行，无副作用的操作在`db.View()`的回调中执行。上面代码先插入一个键值对，然后读取这个键。

从代码我们可以看出，由于涉及数据库操作，需要大量的错误处理。为了简洁起见，本文后面的代码省略了错误处理，在实际使用中必须加上！

## 特性

### 桶

**桶（`bucket`）**有点像命名空间的概念。在同一个桶中的键不能重复，不同的桶可以包含相同的键。`nutsdb`提供的更新和查询接口都需要传入桶名，只是我们在最开始的例子中将桶名设置为空字符串了。

```golang
func main() {
  opt := nutsdb.DefaultOptions
  opt.Dir = "./nutsdb"
  db, _ := nutsdb.Open(opt)
  defer db.Close()

  key := []byte("name")
  val := []byte("dj")

  db.Update(func(tx *nutsdb.Tx) error {
    tx.Put("bucket1", key, val, 0)
    return nil
  })

  db.Update(func(tx *nutsdb.Tx) error {
    tx.Put("bucket2", key, val, 0)
    return nil
  })

  db.View(func(tx *nutsdb.Tx) error {
    e, _ := tx.Get("bucket1", key)
    fmt.Println("val1:", string(e.Value))

    e, _ = tx.Get("bucket2", key)
    fmt.Println("val2:", string(e.Value))
    return nil
  })
}
```

运行：

```golang
val1: dj
val2: dj
```

我们可以将桶类比于 redis 中的 db 的概念，redis 可以在不同的 db 中存储相同的键，但是同一个 db 的键是唯一的。通过 redis 客户端连接服务器后，使用命令`select db`切换不同的 db。

### 更新和删除

上面我们看到使用`tx.Put()`插入字段，其实`tx.Put()`也用来更新（如果键已存在）。`tx.Delete()`用来删除一个字段。

```golang
func main() {
  opt := nutsdb.DefaultOptions
  opt.Dir = "./nutsdb"
  db, _ := nutsdb.Open(opt)
  defer db.Close()

  key := []byte("name")
  val := []byte("dj")
  db.Update(func(tx *nutsdb.Tx) error {
    tx.Put("", key, val, 0)
    return nil
  })

  db.View(func(tx *nutsdb.Tx) error {
    e, _ := tx.Get("", key)
    fmt.Println(string(e.Value))
    return nil
  })

  db.Update(func(tx *nutsdb.Tx) error {
    tx.Delete("", key)
    return nil
  })

  db.View(func(tx *nutsdb.Tx) error {
    e, err := tx.Get("", key)
    if err != nil {
      log.Fatal(err)
    } else {
      fmt.Println(string(e.Value))
    }
    return nil
  })
}
```

删除后再次`Get()`，返回`err`：

```golang
dj
2020/04/27 22:28:19 key not found in the bucket
exit status 1
```

### 过期

`nutsdb`支持在插入或更新键值对时设置一个过期时间。`Put()`的第四个参数即为过期时间，单位 s。传 0 表示不过期：

```golang
func main() {
  opt := nutsdb.DefaultOptions
  opt.Dir = "./nutsdb"
  db, _ := nutsdb.Open(opt)
  defer db.Close()

  key := []byte("name")
  val := []byte("dj")
  db.Update(func(tx *nutsdb.Tx) error {
    tx.Put("", key, val, 10)
    return nil
  })

  db.View(func(tx *nutsdb.Tx) error {
    e, _ := tx.Get("", key)
    fmt.Println(string(e.Value))
    return nil
  })

  time.Sleep(10 * time.Second)

  db.View(func(tx *nutsdb.Tx) error {
    e, err := tx.Get("", key)
    if err != nil {
      log.Fatal(err)
    } else {
      fmt.Println(string(e.Value))
    }
    return nil
  })
}
```

插入一个数据，设置过期时间为 10s。等待 10s 之后返回`err`：

```golang
dj
2020/04/27 22:31:16 key not found in the bucket
exit status 1
```

### 遍历

在`nutsdb`的每个桶中，键是以**字节**顺序保存的。这使得顺序遍历异常迅速。

#### 前缀遍历

我们可以使用`PrefixScan()`遍历具有特定前缀的键值对。它可以指定从第几个数据开始，返回多少条满足条件的数据。例如，每个玩家在`nutsdb`中保存在**`user_` + 玩家id**的键中：

```golang
func main() {
  opt := nutsdb.DefaultOptions
  opt.Dir = "./nutsdb"
  db, _ := nutsdb.Open(opt)
  defer db.Close()

  bucket := "user_list"
  prefix := "user_"
  db.Update(func(tx *nutsdb.Tx) error {
    for i := 1; i <= 300; i++ {
      key := []byte(prefix + strconv.FormatInt(int64(i), 10))
      val := []byte("dj" + strconv.FormatInt(int64(i), 10))
      tx.Put(bucket, key, val, 0)
    }
    return nil
  })

  db.View(func(tx *nutsdb.Tx) error {
    entries, _, _ := tx.PrefixScan(bucket, []byte(prefix), 25, 100)
    for _, entry := range entries {
      fmt.Println(string(entry.Key), string(entry.Value))
    }
    return nil
  })
}
```

先插入 300 条数据，然后使用`PrefixScan()`从第 25 条数据开始，一共返回 100 条数据。需要注意的是，键是以**字节**顺序排列，所以`user_21`在`user_209`之后。观察输出：

```golang
...
user_208 dj208
user_209 dj209
user_21 dj21
user_210 dj210
```

#### 范围遍历

可以使用`tx.RangeScan()`只返回键在指定范围内的数据：

```golang
func main() {
  opt := nutsdb.DefaultOptions
  opt.Dir = "./nutsdb"
  db, _ := nutsdb.Open(opt)
  defer db.Close()

  bucket := "user_list"
  prefix := "user_"
  db.Update(func(tx *nutsdb.Tx) error {
    for i := 1; i <= 300; i++ {
      key := []byte(prefix + strconv.FormatInt(int64(i), 10))
      val := []byte("dj" + strconv.FormatInt(int64(i), 10))
      tx.Put(bucket, key, val, 0)
    }
    return nil
  })

  db.View(func(tx *nutsdb.Tx) error {
    lbound := []byte("user_100")
    ubound := []byte("user_199")
    entries, _ := tx.RangeScan(bucket, lbound, ubound)
    for _, entry := range entries {
      fmt.Println(string(entry.Key), string(entry.Value))
    }
    return nil
  })
}
```

#### 获取全部

调用`tx.GetAll()`返回某个桶中所有的数据：

```golang
func main() {
  opt := nutsdb.DefaultOptions
  opt.Dir = "./nutsdb"
  db, _ := nutsdb.Open(opt)
  defer db.Close()

  bucket := "user_list"
  prefix := "user_"
  db.Update(func(tx *nutsdb.Tx) error {
    for i := 1; i <= 300; i++ {
      key := []byte(prefix + strconv.FormatInt(int64(i), 10))
      val := []byte("dj" + strconv.FormatInt(int64(i), 10))
      tx.Put(bucket, key, val, 0)
    }
    return nil
  })

  db.View(func(tx *nutsdb.Tx) error {
    entries, _ := tx.GetAll(bucket)
    for _, entry := range entries {
      fmt.Println(string(entry.Key), string(entry.Value))
    }
    return nil
  })
}
```

## 数据结构

相比其他数据库，`nutsdb`比较强大的地方在于它支持多种数据结构：`list/set/sorted set`。命令主要仿造`redis`命令编写。这三种结构的操作与`redis`中对应的命令非常相似，本文简单介绍一下`list`相关方法，`set/sorted set`可自行探索。

`nutsdb`中支持的`list`方法如下：

* `LPush`：从头部插入一个元素；
* `RPush`：从尾部插入一个元素；
* `LPop`：从头部删除一个元素；
* `RPop`：从尾部删除一个元素；
* `LPeek`：返回头部第一个元素；
* `RPeek`：返回尾部第一个元素；
* `LRange`：返回指定**索引**范围内的元素；
* `LRem`：删除指定数量的值等于特定值的项；
* `LSet`：设置某个索引的值；
* `Ltrim`：只保留指定索引范围内的元素，其它都移除；
* `LSize`：返回`list`长度。

下面简单演示一下如何使用这些方法，每一步的操作结果都以注释写在了命令下方：

```golang
func main() {
  opt := nutsdb.DefaultOptions
  opt.Dir = "./nutsdb"
  db, _ := nutsdb.Open(opt)
  defer db.Close()

  bucket := "list"
  key := []byte("userList")

  db.Update(func(tx *nutsdb.Tx) error {
    // 从头部依次插入多个值，注意顺序
    tx.LPush(bucket, key, []byte("user1"), []byte("user3"), []byte("user5"))
    // 当前list：user5, user3, user1

    // 从尾部依次插入多个值
    tx.RPush(bucket, key, []byte("user7"), []byte("user9"), []byte("user11"))
    // 当前list：user5, user3, user1, user7, user9, user11
    return nil
  })

  db.Update(func(tx *nutsdb.Tx) error {
    // 从头部删除一个值
    tx.LPop(bucket, key)
    // 当前list：user3, user1, user7, user9, user11

    // 从尾部删除一个值
    tx.RPop(bucket, key)
    // 当前list：user3, user1, user7, user9

    // 从头部删除两个值
    tx.LRem(bucket, key, 2)
    // 当前list：user7, user9
    return nil
  })

  db.View(func(tx *nutsdb.Tx) error {
    // 头部第一个值，user7
    b, _ := tx.LPeek(bucket, key)
    fmt.Println(string(b))

    // 长度
    l, _ := tx.LSize(bucket, key)
    fmt.Println(l)
    return nil
  })
}
```

**注意不要在同一个`Update`中执行插入和删除**。

## 数据库备份

`nutsdb`可以很方便地进行数据库备份，只需要调用`db.Backup()`，传入备份存放目录即可：

```golang
func main() {
  opt := nutsdb.DefaultOptions
  opt.Dir = "./nutsdb"
  db, _ := nutsdb.Open(opt)

  key := []byte("name")
  val := []byte("dj")
  db.Update(func(tx *nutsdb.Tx) error {
    tx.Put("", key, val, 0)
    return nil
  })

  db.Backup("./backup")
  db.Close()

  opt.Dir = "./backup"
  backupDB, _ := nutsdb.Open(opt)
  backupDB.View(func(tx *nutsdb.Tx) error {
    e, _ := tx.Get("", key)
    fmt.Println(string(e.Value))
    return nil
  })
}
```

上面先备份，再从备份中加载数据库，读取键。

## 总结

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. nutsdb GitHub：[https://github.com/xujiajun/nutsdb](https://github.com/xujiajun/nutsdb)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)