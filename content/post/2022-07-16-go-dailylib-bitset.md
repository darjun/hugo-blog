---
layout:    post
title:    "Go 每日一库之 bitset"
subtitle: "每天学习一个 Go 库"
date:     2022-07-16T12:30:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2022/07/16/godailylib/bitset"
categories: [
  "Go"
]
---

## 简介

我们都知道计算机是基于二进制的，位运算是计算机的基础运算。位运算的优势很明显，CPU 指令原生支持、速度快。基于位运算的位集合在有限的场景中替换集合数据结构可以收到意想不到的效果。[`bitset`](https://github.com/bits-and-blooms/bitset)库实现了位集合及相关操作，不妨拿来即用。

## 安装

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir -p bitset && cd bitset
$ go mod init github.com/darjun/go-daily-lib/bitset
```

安装`bitset`库：

```cmd
$ go get -u github.com/bits-and-blooms/bitset
```

## 使用

位集合的基本操作有：

* 检查位（Test）：检查某个索引是否为 1。类比检查元素是否在集合中
* 设置位（Set）：将某个索引设置为 1。类比向集合添加元素
* 清除位（Clear）：将某个索引清除，设置为 0。类比从集合中删除元素
* 翻转位（Flip）：如果某个索引为 1，则设置为 0，反之设置为 1
* 并（Union）：两个位集合执行并操作。类比集合的并
* 交（Intersection）：两个位集合执行交操作。类比集合的交

位集合一般用于小数值的非负整数的场景中。就拿游戏中简单的签到举例吧，很多游戏都有签到活动，短的有 7 天的，长的有 30 天。这种就很适合使用位集合。每个位的值表示其索引位置对应的那天有没有签到。

```golang
type Player struct {
  sign *bitset.BitSet
}

func NewPlayer(sign uint) *Player {
  return &Player{
    sign: bitset.From([]uint64{uint64(sign)}),
  }
}

func (this *Player) Sign(day uint) {
  this.sign.Set(day)
}

func (this *Player) IsSigned(day uint) bool {
  return this.sign.Test(day)
}

func main() {
  player := NewPlayer(1) // 第一天签到
  for day := uint(2); day <= 7; day++ {
    if rand.Intn(100)&1 == 0 {
      player.Sign(day - 1)
    }
  }

  for day := uint(1); day <= 7; day++ {
    if player.IsSigned(day - 1) {
      fmt.Printf("day:%d signed\n", day)
    }
  }
}
```

bitset 提供了多种创建 BitSet 对象的方法。

首先 bitset.BitSet 零值可用，如果一开始不知道有多少元素，可以使用这种方式创建：

```golang
var b bitset.BitSet
```

BitSet 在设置时自动调整大小。如果事先知道长度，创建 BitSet 时可传入此值，能有效避免自动调整的开销：

```golang
b := bitset.New(100)
```

bitset 结构支持链式调用，大部分方法返回自身的指针，所以可以这样写：

```golang
b.Set(10).Set(11).Clear(12).Flip(13);
```

注意，bitset 的索引是从 0 开始的。

记得之前在网上看过一道题：

一个农夫带着一只狼、一头羊和一颗白菜来到河边。他需要用船把他们带到对岸。然而，这艘船只能容下农夫本人和另外一种东西（要么是狼，要么是羊，要么是白菜）。如果农夫不在场的话，狼就会吃掉羊，羊会吃掉白菜。请为农夫解决这个问题。

这其实是一个状态搜索的问题，用回溯法就能解决。农夫、狼、羊、白菜都有两个状态，即在河左岸（假设刚开始农夫所处的是左岸）还是河右岸。这里实际上还有个船的状态，由于船一定和农夫的状态是一致的，就不用额外考虑了。这些状态我们很容易用位集合来表示：

```golang
const (
  FARMER = iota
  WOLF
  SHEEP
  CABBAGE
)
```

编写一个函数来判断状态是否合法。有两种状态不合法：

* 狼和羊在同一边，并且不和农夫在同一边。此时狼会吃掉羊
* 羊和白菜在同一边，并且不和农夫在同一边。此时羊会吃掉白菜

```golang
func IsStateValid(state *bitset.BitSet) bool {
  if state.Test(WOLF) == state.Test(SHEEP) &&
    state.Test(WOLF) != state.Test(FARMER) {
    // 狼和羊在同一边，并且不和农夫在同一边
    // 狼会吃掉羊，非法
    return false
  }

  if state.Test(SHEEP) == state.Test(CABBAGE) &&
    state.Test(SHEEP) != state.Test(FARMER) {
    // 羊和白菜在同一边，并且不和农夫在同一边
    // 羊会吃掉白菜，非法
    return false
  }

  return true
}
```

接下来编写搜索函数：

```golang
func search(b *bitset.BitSet, visited map[string]struct{}) bool {
  if !IsStateValid(b) {
    return false
  }

  if _, exist := visited[b.String()]; exist {
    // 状态已遍历
    return false
  }

  if b.Count() == 4 {
    return true
  }

  visited[b.String()] = struct{}{}
  for index := uint(FARMER); index <= CABBAGE; index++ {
    if b.Test(index) != b.Test(FARMER) {
      // 与农夫不在一边，不能带上船
      continue
    }

    // 带到对岸去
    b.Flip(index)
    if index != FARMER {
      // 如果 index 为 FARMER，表示不带任何东西
      b.Flip(FARMER)
    }

    if search(b, visited) {
      return true
    }

    // 状态恢复
    b.Flip(index)
    if index != FARMER {
      b.Flip(FARMER)
    }
  }

  return false
}
```

主函数：

```golang
func main() {
  b := bitset.New(4)

  visited := make(map[string]struct{})
  fmt.Println(search(b, visited))
}
```

初始时，所有状态为 0，都到对岸之后所有状态为 1，故`b.Count() == 4`表示都已到达对岸了。由于搜索是盲目的，可能会无限循环：这次农夫将羊带到对岸，下次又将其从对岸带回来了。所以我们需要做状态去重。`bitset.String()`返回当前位集合的字符串表示，我们以此来判断状态是否重复。

for 循环依次尝试带各种物品，或什么也不带。驱动查找过程。

如果想得到农夫正确的动作序列，可以给 search 加一个参数，记录每一步的操作：

```golang
func search(b *bitset.BitSet, visited map[string]struct{}, path *[]*bitset.BitSet) bool {
  // 记录路径
  *path = append(*path, b.Clone())
  if b.Count() == 4 {
    return true
  }

  // ...
  *path = (*path)[:len(*path)-1]

  return false
}

func main() {
  b := bitset.New(4)

  visited := make(map[string]struct{})
  var path []*bitset.BitSet
  if search(b, visited, &path) {
    PrintPath(path)
  }
}
```

如果找到解，打印之：

```golang
var names = []string{"农夫", "狼", "羊", "白菜"}

func PrintState(b *bitset.BitSet) {
  fmt.Println("=======================")
  fmt.Println("河左岸：")
  for index := uint(FARMER); index <= CABBAGE; index++ {
    if !b.Test(index) {
      fmt.Println(names[index])
    }
  }

  fmt.Println("河右岸：")
  for index := uint(FARMER); index <= CABBAGE; index++ {
    if b.Test(index) {
      fmt.Println(names[index])
    }
  }
  fmt.Println("=======================")
}

func PrintMove(cur, next *bitset.BitSet) {
  for index := uint(WOLF); index <= CABBAGE; index++ {
    if cur.Test(index) != next.Test(index) {
      if !cur.Test(FARMER) {
        fmt.Printf("农夫将【%s】从河左岸带到河右岸\n", names[index])
      } else {
        fmt.Printf("农夫将【%s】从河右岸带到河左岸\n", names[index])

      }
      return
    }
  }

  if !cur.Test(FARMER) {
    fmt.Println("农夫独自从河左岸到河右岸")
  } else {
    fmt.Println("农夫独自从河右岸到河左岸")
  }
}

func PrintPath(path []*bitset.BitSet) {
  cur := path[0]
  PrintState(cur)

  for i := 1; i < len(path); i++ {
    next := path[i]
    PrintMove(cur, next)
    PrintState(next)
    cur = next
  }
}
```

运行：

```
=======================
河左岸：
农夫
狼
羊
白菜

河右岸：
=======================
农夫将【羊】从河左岸带到河右岸
=======================
河左岸：
狼
白菜

河右岸：
农夫
羊
=======================
农夫独自从河右岸到河左岸
=======================
河左岸：
农夫
狼
白菜

河右岸：
羊
=======================
农夫将【狼】从河左岸带到河右岸
=======================
河左岸：
白菜

河右岸：
农夫
狼
羊
=======================
农夫将【羊】从河右岸带到河左岸
=======================
河左岸：
农夫
羊
白菜

河右岸：
狼
=======================
农夫将【白菜】从河左岸带到河右岸
=======================
河左岸：
羊

河右岸：
农夫
狼
白菜
=======================
农夫独自从河右岸到河左岸
=======================
河左岸：
农夫
羊

河右岸：
狼
白菜
=======================
农夫将【羊】从河左岸带到河右岸
=======================
河左岸：

河右岸：
农夫
狼
羊
白菜
=======================
```

即农夫操作过程为：将羊带到右岸->独自返回->将白菜带到右岸->再将羊带回左岸->带上狼到右岸->独自返回->最后将羊带到右岸->完成。

## 为什么要使用它？

似乎直接使用位运算就可以了，为什么要使用 bitset 库呢？

因为通用，上面列举的两个例子都是很小的整数值，如果整数值超过 64 位，我们必然要通过切片来存储。此时手写操作就非常不便了，而且容易出错。

库的优势体现在：

* 足够通用
* 持续优化
* 大规模使用

只要对外提供的接口保持不变，它可以一直优化内部实现。虽然我们也可以做，但是费时费力。而且有些优化涉及到比较复杂的算法，自己实现的难度较高且易错。

有一个很典型的例子，求一个 uint64 的二进制表示中 1 的数量（popcnt，或 population count）。实现的方法有很多。

最直接的，我们一位一位地统计：

```golang
func popcnt1(n uint64) uint64 {
  var count uint64

  for n > 0 {
    if n&1 == 1 {
      count++
    }

    n >>= 1
  }

  return count
}
```

考虑空间换时间，我们可以预先计算 0-255 这 256 个数字的二进制表示中 1 的个数，然后每 8 位计算一次，可能将计算次数减少到之前的 1/8。这也是标准库中的做法：

```golang
const pop8tab = "" +
  "\x00\x01\x01\x02\x01\x02\x02\x03\x01\x02\x02\x03\x02\x03\x03\x04" +
  "\x01\x02\x02\x03\x02\x03\x03\x04\x02\x03\x03\x04\x03\x04\x04\x05" +
  "\x01\x02\x02\x03\x02\x03\x03\x04\x02\x03\x03\x04\x03\x04\x04\x05" +
  "\x02\x03\x03\x04\x03\x04\x04\x05\x03\x04\x04\x05\x04\x05\x05\x06" +
  "\x01\x02\x02\x03\x02\x03\x03\x04\x02\x03\x03\x04\x03\x04\x04\x05" +
  "\x02\x03\x03\x04\x03\x04\x04\x05\x03\x04\x04\x05\x04\x05\x05\x06" +
  "\x02\x03\x03\x04\x03\x04\x04\x05\x03\x04\x04\x05\x04\x05\x05\x06" +
  "\x03\x04\x04\x05\x04\x05\x05\x06\x04\x05\x05\x06\x05\x06\x06\x07" +
  "\x01\x02\x02\x03\x02\x03\x03\x04\x02\x03\x03\x04\x03\x04\x04\x05" +
  "\x02\x03\x03\x04\x03\x04\x04\x05\x03\x04\x04\x05\x04\x05\x05\x06" +
  "\x02\x03\x03\x04\x03\x04\x04\x05\x03\x04\x04\x05\x04\x05\x05\x06" +
  "\x03\x04\x04\x05\x04\x05\x05\x06\x04\x05\x05\x06\x05\x06\x06\x07" +
  "\x02\x03\x03\x04\x03\x04\x04\x05\x03\x04\x04\x05\x04\x05\x05\x06" +
  "\x03\x04\x04\x05\x04\x05\x05\x06\x04\x05\x05\x06\x05\x06\x06\x07" +
  "\x03\x04\x04\x05\x04\x05\x05\x06\x04\x05\x05\x06\x05\x06\x06\x07" +
  "\x04\x05\x05\x06\x05\x06\x06\x07\x05\x06\x06\x07\x06\x07\x07\x08"

func popcnt2(n uint64) uint64 {
  var count uint64

  for n > 0 {
    count += uint64(pop8tab[n&0xff])
    n >>= 8
  }

  return count
}
```

最后是 bitset 库中的算法：

```golang
func popcnt3(x uint64) (n uint64) {
  x -= (x >> 1) & 0x5555555555555555
  x = (x>>2)&0x3333333333333333 + x&0x3333333333333333
  x += x >> 4
  x &= 0x0f0f0f0f0f0f0f0f
  x *= 0x0101010101010101
  return x >> 56
}
```

对以上三种实现进行性能测试：

```golang
goos: windows
goarch: amd64
pkg: github.com/darjun/go-daily-lib/bitset/popcnt
cpu: Intel(R) Core(TM) i7-7700 CPU @ 3.60GHz
BenchmarkPopcnt1-8         52405             24409 ns/op
BenchmarkPopcnt2-8        207452              5443 ns/op
BenchmarkPopcnt3-8       1777320               602 ns/op
PASS
ok      github.com/darjun/go-daily-lib/bitset/popcnt    4.697s
```

popcnt3 相对 popcnt1 有 40 倍的性能提升。在学习上我们可以自己尝试造轮子，以此来加深自己对技术的理解。但是在工程上，通常更倾向于使用稳定的、高效的库。

## 总结

本文借着 bitset 库介绍了位集合的使用。并且应用 bitset 解决了农夫过河问题。最后我们讨论了为什么要使用库？

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 一点闲话

我发现人的惰性实在是太可怕了。虽然这半年来没写文章一开始是因为工作上的原因，后来单纯是因为惯性，因为懒。而且总是“装着很忙”来逃避需要花时间、花精力的事情。在这种想动又不想动的角逐之间，时间就这么一晃而过。

我们总是在抱怨没有时间，没有时间。但仔细想想，仔细算算，我们花在刷短视频，玩游戏上的时间其实并不少。

上周我在阮一峰老师的周刊上看到一篇文章《人生不短》，看了之后深有触动。人总该有所坚持，生活才有意义。

## 参考

1. bitset GitHub：[github.com/bits-and-blooms/bitset](https://github.com/bits-and-blooms/bitset)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)