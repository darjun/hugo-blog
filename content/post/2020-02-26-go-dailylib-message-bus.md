---
layout:    post
title:    "Go 每日一库之 message-bus"
subtitle: 	"每天学习一个 Go 库"
date:		2020-02-26T14:40:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/02/26/godailylib/message-bus"
categories: [
	"Go"
]
---

## 简介

在一个涉及多模块交互的系统中，如果模块的交互需要手动去调用对方的方法，那么代码的耦合度就太高了。所以产生了异步消息通信。实际上，各种各样的消息队列都是基于异步消息的。不过它们大部分都有着非常复杂的设计，很多被设计成一个独立的软件来使用。今天我们介绍一个非常小巧的异步消息通信库`[message-bus]`(https://github.com/vardius/message-bus)，它只能在一个进程中使用。源代码只有一个文件，我们也简单看一下实现。

## 快速使用

安装：

```cmd
$ go get github.com/vardius/message-bus
```

使用：

```golang
package main

import (
  "fmt"
  "sync"

  messagebus "github.com/vardius/message-bus"
)

func main() {
  queueSize := 100
  bus := messagebus.New(queueSize)

  var wg sync.WaitGroup
  wg.Add(2)

  _ = bus.Subscribe("topic", func(v bool) {
    defer wg.Done()
    fmt.Println(v)
  })

  _ = bus.Subscribe("topic", func(v bool) {
    defer wg.Done()
    fmt.Println(v)
  })

  bus.Publish("topic", true)
  wg.Wait()
}
```

这是官网提供的例子，`message-bus`承担了模块间消息分发的角色。模块 A 和 模块 B 先向`message-bus`订阅主题（**topic**），即告诉`message-bus`对什么样的消息感兴趣。其他模块 C 产生某个主题的消息，通知`message-bus`，由`message-bus`分发到对此感兴趣的模块。这样就实现了模块之间的解耦，模块 A、B 和 C 之间不需要知道彼此。

上面的例子中：

* 首先，调用`messagebuss.New()`创建一个消息管理器；
* 其次调用`Subscribe()`方法向管理器订阅主题；
* 调用`Publish()`向管理器发布主题消息，这样订阅该主题的模块就会收到通知。

## 更复杂的例子

其实很多人会对何时使用这个库产生疑问，`message-bus` GitHub 仓库中 issue 中至今还躺着这个问题，[https://github.com/vardius/message-bus/issues/4](https://github.com/vardius/message-bus/issues/4)。我是做游戏后端开发的，在一个游戏中会涉及各种各样的功能模块，它们需要了解其他模块产生的事件。例如每日任务有玩家升多少级的任务、成就系统有等级的成就、其他系统还可能根据玩家等级执行其他操作...如果硬写的话，最后可能是这样：

```golang
func (p *Player) LevelUp() {
  // ...
  p.DailyMission.OnPlayerLevelUp(oldLevel, newLevel)
  p.Achievement.OnPlayerLevelUp(oldLevel, newLevel)
  p.OtherSystem.OnPlayerLevelUp(oldLevel, newLevel)
}
```

需求一直在新增和迭代，如果新增一个模块，也需要在玩家升级时进行一些处理，除了实现模块自身的`OnPlayerLevelUp`方法，还必须在玩家的`LevelUp()`方法调用。这样玩家模块必须清楚地知道其他模块的情况。如果功能模块再多一点，而且由不同的人开发的，那么情况会更复杂。使用异步消息可有效解决这个问题：在升级时我们只需要向消息管理器发布这个升级“消息”，由消息管理器通知订阅该消息的模块。

我们设计的目录结构如下：

```tree
game
├── achievement.go
├── daily_mission.go
├── main.go
├── manager.go
└── player.go
```

其中`manager.go`负责`message-bus`的创建：

```golang
package main

import (
  messagebus "github.com/vardius/message-bus"
)

var bus = messagebus.New(10)
```

`player.go`对应玩家结构（为了简便起见，很多字段省略了）：

```golang
package main

type Player struct {
  level uint32
}

func NewPlayer() *Player {
  return &Player{}
}

func (p *Player) LevelUp() {
  oldLevel := p.level
  newLevel := p.level+1
  p.level++

  bus.Publish("UserLevelUp", oldLevel, newLevel)
}
```

`achievement.go`和`daily_mission.go`分别是成就和每日任务（也是省略了很多无关细节）：

```golang
// achievement.go
package main

import "fmt"

type Achievement struct {
  // ...
}

func NewAchievement() *Achievement {
  a := &Achievement{}
  bus.Subscribe("UserLevelUp", a.OnUserLevelUp)
  return a
}

func (a *Achievement) OnUserLevelUp(oldLevel, newLevel uint32) {
  fmt.Printf("daily mission old level:%d new level:%d\n", oldLevel, newLevel)
}
```

```golang
// daily_mission.go
package main

import "fmt"

type DailyMission struct {
  // ...
}

func NewDailyMission() *DailyMission {
  d := &DailyMission{}
  bus.Subscribe("UserLevelUp", d.OnUserLevelUp)
  return d
}

func (d *DailyMission) OnUserLevelUp(oldLevel, newLevel uint32) {
  fmt.Printf("daily mission old level:%d new level:%d\n", oldLevel, newLevel)
}
```

在创建这两个功能的对象时，我们订阅了`UserLevelUp`主题。玩家在升级时会发布这个主题。

最后`main.go`驱动整个程序：

```golang
package main

import "time"

func main() {
  p := NewPlayer()
  NewDailyMission()
  NewAchievement()

  p.LevelUp()
  p.LevelUp()
  p.LevelUp()

  time.Sleep(1000)
}
```

注意，由于`message-bus`是异步通信，为了能看到结果我特意加了`time.Sleep`，实际开发中不太可能使用`Sleep`。

最后我们运行整个程序：

```golang
$ go run .
```

因为要运行的是一个多文件程序，不能使用`go run main.go`！

实际上，当年我因为苦于模块之间调来调去太麻烦了，自己用 C++ 撸了一个`event-manager`，[https://github.com/darjun/event-manager](https://github.com/darjun/event-manager)。思路是一样的。

## 缺点

`message-bus`订阅主题时传入一个函数，函数的参数可任意设置，发布时必须使用相同数量的参数，这个限制感觉有点勉强。如果我们传入的参数个数不一致，程序就`panic`了。我认为可以只用一个参数`interface{}`，传入对象即可。例如，上面的升级事件可以使用`EventUserLevelUp`的对象：

```golang
type EventUserLevelUp struct {
  oldLevel uint32
  newLevel uint32
}
```

对应地修改一下`Player`的`LevelUp`方法：

```golang
func (p *Player) LevelUp() {
  event := &EventUserLevelUp {
    oldLevel: p.level,
    newLevel: p.level+1,
  }
  p.level++

  bus.Publish("UserLevelUp", event)
}
```

和处理方法：

```golang
func (d *DailyMission) OnUserLevelUp(arg interface{}) {
  event := arg.(*EventUserLevelUp)
  fmt.Printf("daily mission old level:%d new level:%d\n", event.oldLevel, event.newLevel)
}
```

这样一来，我们似乎用不上反射了，订阅者都是`func (interface{})`类型的函数或方法。感兴趣的可自己实现一下，我 fork 了`message-bus`，做了这个修改。改动在这里：[https://github.com/darjun/message-bus](https://github.com/darjun/message-bus/commit/045ad9bd50acb7ef709dac35babe3e7d8ecf9749)，`message-bus`有测试和性能用例，改完跑一下😄。

## 源码分析

`message-bus`的源码只有一个文件，加上注释还不到 130 行，我们简单来看一下。

`MessageBus`就是一个简单的接口：

```golang
type MessageBus interface {
  Publish(topic string, args ...interface{})
  Close(topic string)
  Subscribe(topic string, fn interface{}) error
  Unsubscribe(topic string, fn interface{}) error
}
```

`Publish`和`Subscribe`都讲过了，`Unsubscribe`表示对某个主题不感兴趣了，取消订阅，`Close`直接关闭某个主题的队列，删除所有订阅者。

在`message-bus`内部，每个主题对应一组订阅者。每个订阅者使用`handler`结构存储回调和参数通道：

```golang
type handler struct {
  callback reflect.Value
  queue    chan []reflect.Value
}
```

所有订阅者都存储在一个 map 中：

```golang
type handlersMap map[string][]*handler

type messageBus struct {
  handlerQueueSize int
  mtx              sync.RWMutex
  handlers         handlersMap
}
```

`messageBus`是`MessageBus`接口的实现。我们来看看各个方法是如何实现的。

```golang
func (b *messageBus) Subscribe(topic string, fn interface{}) error {
  h := &handler{
    callback: reflect.ValueOf(fn),
    queue:    make(chan []reflect.Value, b.handlerQueueSize),
  }

  go func() {
    for args := range h.queue {
      h.callback.Call(args)
    }
  }()

  b.handlers[topic] = append(b.handlers[topic], h)
  return nil
}
```

调用`Subscribe`时传入一个函数，`message-bus`为每个订阅者创建一个`handler`对象，在该对象中创建一个带缓冲的参数通道，缓冲大小由`message-bus`创建时的参数指定。
同时启动一个`goroutine`，监听通道，每当有参数到来时就执行注册的回调。

```golang
func (b *messageBus) Publish(topic string, args ...interface{}) {
  rArgs := buildHandlerArgs(args)
  if hs, ok := b.handlers[topic]; ok {
    for _, h := range hs {
      h.queue <- rArgs
    }
  }
}
```

`Publish`发布主题，`buildHandlerArgs`将传入的参数转为`[]reflect.Value`，以便反射调用回调时传入。发送参数到该主题下所有`handler`的通道中。由`Subscribe`时创建的`goroutine`读取并触发回调。

```golang
func (b *messageBus) Unsubscribe(topic string, fn interface{}) error {
  rv := reflect.ValueOf(fn)

  if _, ok := b.handlers[topic]; ok {
    for i, h := range b.handlers[topic] {
      if h.callback == rv {
        close(h.queue)

        b.handlers[topic] = append(b.handlers[topic][:i], b.handlers[topic][i+1:]...)
      }
    }

    return nil
  }

  return fmt.Errorf("topic %s doesn't exist", topic)
}
```

`Unsubscribe`将某个订阅者从`message-bus`中移除，移除时需要关闭通道，否则会造成订阅者的 goroutine 泄露。

```golang
func (b *messageBus) Close(topic string) {
  if _, ok := b.handlers[topic]; ok {
    for _, h := range b.handlers[topic] {
      close(h.queue)
    }

    delete(b.handlers, topic)

    return
  }
}
```

`Close`关闭某主题下所有的订阅者参数通道，并删除该主题。

注意，为了保证并发安全，每个方法都加了锁，分析实现时先忽略锁和错误处理。

为了更直观的理解，我画了一个`message-bus`内部结构图：

![](/img/in-post/godailylib/message-bus1.png#center)

## 总结

`message-bus`是一个小巧的异步通信库，实际使用可能不多，但却是学习源码的好资源。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. message-bus GitHub：[https://github.com/vardius/message-bus](https://github.com/vardius/message-bus)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)