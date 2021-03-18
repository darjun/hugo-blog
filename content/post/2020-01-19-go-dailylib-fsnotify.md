---
layout:    post
title:    "Go 每日一库之 fsnotify"
subtitle: 	"每天学习一个 Go 库"
date:		2020-01-18T21:08:05
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/01/19/godailylib/fsnotify"
categories: [
	"Go"
]
---

## 简介

上一篇文章[Go 每日一库之 viper](http://darjun.github.io/2020/01/18/godailylib/viper/)中，我们介绍了 viper 可以监听文件修改进而自动重新加载。
其内部使用的就是`fsnotify`这个库，它是跨平台的。今天我们就来介绍一下它。

## 快速使用

先安装：

```
$ go get github.com/fsnotify/fsnotify
```

后使用：

```golang
package main

import (
  "log"

  "github.com/fsnotify/fsnotify"
)

func main() {
  watcher, err := fsnotify.NewWatcher()
  if err != nil {
    log.Fatal("NewWatcher failed: ", err)
  }
  defer watcher.Close()

  done := make(chan bool)
  go func() {
    defer close(done)

    for {
      select {
      case event, ok := <-watcher.Events:
        if !ok {
          return
        }
        log.Printf("%s %s\n", event.Name, event.Op)
      case err, ok := <-watcher.Errors:
        if !ok {
          return
        }
        log.Println("error:", err)
      }
    }
  }()

  err = watcher.Add("./")
  if err != nil {
    log.Fatal("Add failed:", err)
  }
  <-done
}
```

`fsnotify`的使用比较简单：

* 先调用`NewWatcher`创建一个监听器；
* 然后调用监听器的`Add`增加监听的文件或目录；
* 如果目录或文件有事件产生，监听器中的通道`Events`可以取出事件。如果出现错误，监听器中的通道`Errors`可以取出错误信息。

上面示例中，我们在另一个 goroutine 中循环读取发生的事件及错误，然后输出它们。

编译、运行程序。在当前目录创建一个`新建文本文档.txt`，然后重命名为`file1.txt`文件，输入内容`some test text`，然后删除它。观察控制台输出：

```
2020/01/20 08:41:17 新建文本文档.txt CREATE
2020/01/20 08:41:25 新建文本文档.txt RENAME
2020/01/20 08:41:25 file1.txt CREATE
2020/01/20 08:42:28 file1.txt REMOVE
```

**其实，重命名时会产生两个事件，一个是原文件的`RENAME`事件，一个是新文件的`CREATE`事件。**

注意，`fsnotify`使用了操作系统接口，监听器中保存了系统资源的句柄，所以使用后需要关闭。

## 事件

上面示例中的事件是`fsnotify.Event`类型：

```golang
// fsnotify/fsnotify.go
type Event struct {
  Name string
  Op   Op
}
```

事件只有两个字段，`Name`表示发生变化的文件或目录名，`Op`表示具体的变化。`Op`有 5 种取值：

```golang
// fsnotify/fsnotify.go
type Op uint32

const (
  Create Op = 1 << iota
  Write
  Remove
  Rename
  Chmod
)
```

在[快速使用](#快速使用)中，我们已经演示了前 4 种事件。`Chmod`事件在文件或目录的属性发生变化时触发，在 Linux 系统中可以通过`chmod`命令改变文件或目录属性。

事件中的`Op`是按照位来存储的，可以存储多个，可以通过`&`操作判断对应事件是不是发生了。

```golang
if event.Op & fsnotify.Write != 0 {
  fmt.Println("Op has Write")
}
```

我们在代码中不需要这样判断，因为`Op`的`String()`方法已经帮我们处理了这种情况了：

```golang
// fsnotify.go
func (op Op) String() string {
  // Use a buffer for efficient string concatenation
  var buffer bytes.Buffer

  if op&Create == Create {
    buffer.WriteString("|CREATE")
  }
  if op&Remove == Remove {
    buffer.WriteString("|REMOVE")
  }
  if op&Write == Write {
    buffer.WriteString("|WRITE")
  }
  if op&Rename == Rename {
    buffer.WriteString("|RENAME")
  }
  if op&Chmod == Chmod {
    buffer.WriteString("|CHMOD")
  }
  if buffer.Len() == 0 {
    return ""
  }
  return buffer.String()[1:] // Strip leading pipe
}
```

## 应用

`fsnotify`的应用非常广泛，在 godoc 上，我们可以看到哪些库导入了`fsnotify`。只需要在`fsnotify`文档的 URL 后加上`?imports`即可：

[https://godoc.org/github.com/fsnotify/fsnotify?importers](https://godoc.org/github.com/fsnotify/fsnotify?importers)。有兴趣打开看看，要 fq。

上一篇文章中，我们介绍了调用`viper.WatchConfig`就可以监听配置修改，自动重新加载。下面我们就来看看`WatchConfig`是怎么实现的：

```golang
// viper/viper.go
func WatchConfig() { v.WatchConfig() }

func (v *Viper) WatchConfig() {
  initWG := sync.WaitGroup{}
  initWG.Add(1)
  go func() {
    watcher, err := fsnotify.NewWatcher()
    if err != nil {
      log.Fatal(err)
    }
    defer watcher.Close()
    // we have to watch the entire directory to pick up renames/atomic saves in a cross-platform way
    filename, err := v.getConfigFile()
    if err != nil {
      log.Printf("error: %v\n", err)
      initWG.Done()
      return
    }

    configFile := filepath.Clean(filename)
    configDir, _ := filepath.Split(configFile)
    realConfigFile, _ := filepath.EvalSymlinks(filename)

    eventsWG := sync.WaitGroup{}
    eventsWG.Add(1)
    go func() {
      for {
        select {
        case event, ok := <-watcher.Events:
          if !ok { // 'Events' channel is closed
            eventsWG.Done()
            return
          }
          currentConfigFile, _ := filepath.EvalSymlinks(filename)
          // we only care about the config file with the following cases:
          // 1 - if the config file was modified or created
          // 2 - if the real path to the config file changed (eg: k8s ConfigMap replacement)
          const writeOrCreateMask = fsnotify.Write | fsnotify.Create
          if (filepath.Clean(event.Name) == configFile &&
            event.Op&writeOrCreateMask != 0) ||
            (currentConfigFile != "" && currentConfigFile != realConfigFile) {
            realConfigFile = currentConfigFile
            err := v.ReadInConfig()
            if err != nil {
              log.Printf("error reading config file: %v\n", err)
            }
            if v.onConfigChange != nil {
              v.onConfigChange(event)
            }
          } else if filepath.Clean(event.Name) == configFile &&
            event.Op&fsnotify.Remove&fsnotify.Remove != 0 {
            eventsWG.Done()
            return
          }

        case err, ok := <-watcher.Errors:
          if ok { // 'Errors' channel is not closed
            log.Printf("watcher error: %v\n", err)
          }
          eventsWG.Done()
          return
        }
      }
    }()
    watcher.Add(configDir)
    initWG.Done()   // done initializing the watch in this go routine, so the parent routine can move on...
    eventsWG.Wait() // now, wait for event loop to end in this go-routine...
  }()
  initWG.Wait() // make sure that the go routine above fully ended before returning
}
```

其实流程是相似的：

* 首先，调用`NewWatcher`创建一个监听器；
* 调用`v.getConfigFile()`获取配置文件路径，抽出文件名、目录，配置文件如果是一个符号链接，获得链接指向的路径；
* 调用`watcher.Add(configDir)`监听配置文件所在目录，另起一个 goroutine 处理事件。

`WatchConfig`不能阻塞主 goroutine，所以创建监听器也是新起 goroutine 进行的。代码中有两个`sync.WaitGroup`变量，`initWG`是为了保证监听器初始化，
`eventsWG`是在事件通道关闭，或配置被删除了，或遇到错误时退出事件处理循环。

然后就是核心事件循环：

* 有事件发生时，判断变化的文件是否是在 viper 中设置的配置文件，发生的是否是创建或修改事件（只处理这两个事件）；
* 如果配置文件为符号链接，若符合链接的指向修改了，也需要重新加载配置；
* 如果需要重新加载配置，调用`v.ReadInConfig()`读取新的配置；
* 如果注册了事件回调，以发生的事件为参数执行回调。

## 总结

`fsnotify`的接口非常简单直接，所有系统相关的复杂性都被封装起来了。这也是我们平时设计模块和接口时可以参考的案例。

## 参考

1. [fsnotify API 设计](https://docs.google.com/document/d/1-GQrFdDVrA57-ce0kbzSth4lQqfOMMRKpih3hPJmvoU/edit)
2. [fsnotify GitHub 仓库](https://github.com/fsnotify/fsnotify)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)