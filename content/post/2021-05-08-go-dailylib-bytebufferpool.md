---
layout:    post
title:    "Go 每日一库之 bytebufferpool"
subtitle: 	"每天学习一个 Go 库"
date:		2021-05-08T20:31:00
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/05/08/godailylib/bytebufferpool"
categories: [
	"Go"
]
---

## 简介

在编程开发中，我们经常会需要**频繁**创建和销毁同类对象的情形。这样的操作很可能会对性能造成影响。这时，常用的优化手段就是使用**对象池**（object pool）。需要创建对象时，我们先从对象池中查找。如果有空闲对象，则从池中移除这个对象并将其返回给调用者使用。只有在池中无空闲对象时，才会真正创建一个新对象。另一方面，对象使用完之后，我们并不进行销毁。而是将它放回到对象池以供后续使用。使用对象池在频繁创建和销毁对象的情形下，能大幅度提升性能。同时，为了避免对象池中的对象占用过多的内存。对象池一般还配有特定的清理策略。Go 标准库`sync.Pool`就是这样一个例子。`sync.Pool`中的对象会被垃圾回收清理掉。

在这类对象中，比较特殊的一类是字节缓冲（底层一般是字节切片）。在做字符串拼接时，为了拼接的高效，我们通常将中间结果存放在一个字节缓冲。在拼接完成之后，再从字节缓冲中生成结果字符串。在收发网络包时，也需要将不完整的包暂时存放在字节缓冲中。

Go 标准库中的类型`bytes.Buffer`封装字节切片，提供一些使用接口。我们知道切片的容量是有限的，容量不足时需要进行扩容。而频繁的扩容容易造成性能抖动。[`bytebufferpool`](https://github.com/valyala/bytebufferpool)实现了自己的`Buffer`类型，并使用一个简单的算法降低扩容带来的性能损失。`bytebufferpool`已经在大名鼎鼎的 Web 框架[fasthttp](https://github.com/valyala/fasthttp)和灵活的 Go 模块库[quicktemplate](https://github.com/valyala/quicktemplate)得到了应用。实际上，这 3 个库是同一个作者：valyala😀。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir bytebufferpool && cd bytebufferpool
$ go mod init github.com/darjun/go-daily-lib/bytebufferpool
```

安装`bytebufferpool`库：

```cmd
$ go get -u github.com/PuerkitoBio/bytebufferpool
```

典型的使用方式先通过`bytebufferpool`提供的`Get()`方法获取一个`bytebufferpool.Buffer`对象，然后调用这个对象的方法写入数据，使用完成之后再调用`bytebufferpool.Put()`将对象放回对象池中。例：

```golang
package main

import (
  "fmt"

  "github.com/valyala/bytebufferpool"
)

func main() {
  b := bytebufferpool.Get()
  b.WriteString("hello")
  b.WriteByte(',')
  b.WriteString(" world!")

  fmt.Println(b.String())

  bytebufferpool.Put(b)
}
```

直接调用`bytebufferpool`包的`Get()`和`Put()`方法，底层操作的是包中默认的对象池：

```golang
// bytebufferpool/pool.go
var defaultPool Pool

func Get() *ByteBuffer { return defaultPool.Get() }
func Put(b *ByteBuffer) { defaultPool.Put(b) }
```

我们当然可以根据实际需要创建新的对象池，将相同用处的对象放在一起（比如我们可以创建一个对象池用于辅助接收网络包，一个用于辅助拼接字符串）：

```golang
func main() {
  joinPool := new(bytebufferpool.Pool)
  b := joinPool.Get()
  b.WriteString("hello")
  b.WriteByte(',')
  b.WriteString(" world!")

  fmt.Println(b.String())

  joinPool.Put(b)
}
```

`bytebufferpool`没有提供具体的创建函数，不过可以使用`new`创建。

## 优化细节

在将对象放回池中时，会根据当前切片的容量进行相应的处理。`bytebufferpool`将大小分为 20 个区间：

| < 2^6 | 2^6 ~ 2^7-1 | ... | > 2^25 |

如果容量小于 2^6，则属于第一个区间。如果处于 2^6 和 2^7-1 之间，则落在第二个区间。依次类推。执行足够多的放回次数后，`bytebufferpool`会重新校准，计算处于哪个区间容量的对象最多。将`defaultSize`设置为该区间的上限容量，第一个区间的上限容量为 2^6，第二区间为 2^7，最后一个区间为 2^26。后续通过`Get()`请求对象时，若池中无空闲对象，创建一个新对象时，直接将容量设置为`defaultSize`。这样基本可以避免在使用过程中的切片扩容，从而提升性能。下面结合代码来理解：

```golang
// bytebufferpool/pool.go
const (
  minBitSize = 6 // 2**6=64 is a CPU cache line size
  steps      = 20

  minSize = 1 << minBitSize
  maxSize = 1 << (minBitSize + steps - 1)

  calibrateCallsThreshold = 42000
  maxPercentile           = 0.95
)

type Pool struct {
  calls       [steps]uint64
  calibrating uint64

  defaultSize uint64
  maxSize     uint64

  pool sync.Pool
}
```

我们可以看到，`bytebufferpool`内部使用了标准库中的对象`sync.Pool`。

这里的`steps`就是上面所说的区间，一共 20 份。`calls`数组记录放回对象容量落在各个区间的次数。

调用`Pool.Get()`将对象放回时，首先计算切片容量落在哪个区间，增加`calls`数组中相应元素的值：

```golang
// bytebufferpool/pool.go
func (p *Pool) Put(b *ByteBuffer) {
  idx := index(len(b.B))

  if atomic.AddUint64(&p.calls[idx], 1) > calibrateCallsThreshold {
    p.calibrate()
  }

  maxSize := int(atomic.LoadUint64(&p.maxSize))
  if maxSize == 0 || cap(b.B) <= maxSize {
    b.Reset()
    p.pool.Put(b)
  }
}
```

如果`calls`数组该元素超过指定值`calibrateCallsThreshold=42000`（说明距离上次校准，放回对象到该区间的次数已经达到阈值了，42000 应该就是个经验数字），则调用`Pool.calibrate()`执行校准操作：

```golang
// bytebufferpool/pool.go
func (p *Pool) calibrate() {
  // 避免并发放回对象触发 `calibrate`
  if !atomic.CompareAndSwapUint64(&p.calibrating, 0, 1) {
    return
  }

  // step 1.统计并排序
  a := make(callSizes, 0, steps)
  var callsSum uint64
  for i := uint64(0); i < steps; i++ {
    calls := atomic.SwapUint64(&p.calls[i], 0)
    callsSum += calls
    a = append(a, callSize{
      calls: calls,
      size:  minSize << i,
    })
  }
  sort.Sort(a)

  // step 2.计算 defaultSize 和 maxSize
  defaultSize := a[0].size
  maxSize := defaultSize

  maxSum := uint64(float64(callsSum) * maxPercentile)
  callsSum = 0
  for i := 0; i < steps; i++ {
    if callsSum > maxSum {
      break
    }
    callsSum += a[i].calls
    size := a[i].size
    if size > maxSize {
      maxSize = size
    }
  }

  // step 3.保存对应值
  atomic.StoreUint64(&p.defaultSize, defaultSize)
  atomic.StoreUint64(&p.maxSize, maxSize)

  atomic.StoreUint64(&p.calibrating, 0)
}
```

step 1.统计并排序

`calls`数组记录了放回对象到对应区间的次数。按照这个次数从大到小排序。注意：`minSize << i`表示区间`i`的上限容量。

step 2.计算`defaultSize`和`maxSize`

`defaultSize`很好理解，取排序后的第一个`size`即可。`maxSize`值记录放回次数超过 95% 的多个对象容量的最大值。它的作用是防止将使用较少的大容量对象放回对象池，从而占用太多内存。这里就可以理解`Pool.Put()`方法后半部分的逻辑了：

```golang
// 如果要放回的对象容量大于 maxSize，则不放回
maxSize := int(atomic.LoadUint64(&p.maxSize))
if maxSize == 0 || cap(b.B) <= maxSize {
  b.Reset()
  p.pool.Put(b)
}
```

step 3.保存对应值

后续通过`Pool.Get()`获取对象时，若池中无空闲对象，新创建的对象默认容量为`defaultSize`。这样的容量能满足绝大多数情况下的使用，避免使用过程中的切片扩容。

```golang
// bytebufferpool/pool.go
func (p *Pool) Get() *ByteBuffer {
  v := p.pool.Get()
  if v != nil {
    return v.(*ByteBuffer)
  }
  return &ByteBuffer{
    B: make([]byte, 0, atomic.LoadUint64(&p.defaultSize)),
  }
}
```

其他一些细节：

* 容量最小值取 2^6 = 64，因为这就是 64 位计算机上 CPU 缓存行的大小。这个大小的数据可以一次性被加载到 CPU 缓存行中，再小就无意义了。
* 代码中多次使用`atomic`原子操作，避免加锁导致性能损失。

当然这个库缺点也很明显，由于大部分使用的容量都小于`defaultSize`，会有部分内存浪费。

## 总结

去掉注释，空行，`bytebufferpool`只用了 150 行左右的代码就实现了一个高性能的`Buffer`对象池。其中细节值得细细品味。阅读高质量的代码有助于提升自己的编码能力，学习编码细节。强烈建议抽空细读！！！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. bytebufferpool GitHub：[https://github.com/valyala/bytebufferpool](https://github.com/valyala/bytebufferpool)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~