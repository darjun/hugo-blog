---
layout:    post
title:    "Go 每日一库之 roaring"
subtitle: "每天学习一个 Go 库"
date:     2022-07-17T10:40:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2022/07/17/godailylib/roaring"
categories: [
  "Go"
]
---

## 简介

集合是软件中的基本抽象。实现集合的方法有很多，例如 hash set、tree等。要实现一个整数集合，位图（bitmap，也称为 bitset 位集合，bitvector 位向量）是个不错的方法。使用 n 个位（bit），我们可以表示整数范围`[0, n)`。如果整数 i 在集合中，第 i 位设置为 1。这样集合的交集（intersection）、并集（unions）和差集（difference）可以利用整数的按位与、按位或和按位与非来实现。而计算机执行位运算是非常迅速的。

上一篇文章我介绍了[bitset](https://darjun.github.io/2022/07/16/godailylib/bitset/)这个库。

bitset 在某些场景中会消耗大量的内存。例如，设置第 1,000,000 位，需要占用超过 100kb 的内存。为此 bitset 库的作者又开发了压缩位图库：[`roaring`](https://github.com/RoaringBitmap/roaring)。

本文首先介绍了 roaring 的使用。最后分析 roaring 的文件存储格式。

## 安装

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir -p roaring && cd roaring
$ go mod init github.com/darjun/go-daily-lib/roaring
```

安装`roaring`库：

```cmd
$ go get -u github.com/RoaringBitmap/roaring
```

## 使用

### 基本操作

```golang
func main() {
  bm1 := roaring.BitmapOf(1, 2, 3, 4, 5, 100, 1000)
  fmt.Println(bm1.String())         // {1,2,3,4,5,100,1000}
  fmt.Println(bm1.GetCardinality()) // 7
  fmt.Println(bm1.Contains(3))      // true

  bm2 := roaring.BitmapOf(1, 100, 500)
  fmt.Println(bm2.String())         // {1,100,500}
  fmt.Println(bm2.GetCardinality()) // 3
  fmt.Println(bm2.Contains(300))    // false

  bm3 := roaring.New()
  bm3.Add(1)
  bm3.Add(11)
  bm3.Add(111)
  fmt.Println(bm3.String())         // {1,11,111}
  fmt.Println(bm3.GetCardinality()) // 3
  fmt.Println(bm3.Contains(11))     // true

  bm1.Or(bm2)                       // 执行并集
  fmt.Println(bm1.String())         // {1,2,3,4,5,100,500,1000}
  fmt.Println(bm1.GetCardinality()) // 8
  fmt.Println(bm1.Contains(500))    // true

  bm2.And(bm3)                      // 执行交集
  fmt.Println(bm2.String())         // {1}
  fmt.Println(bm2.GetCardinality()) // 1
  fmt.Println(bm2.Contains(1))      // true
}
```

上面演示了两种创建 roaring bitmap 的方式：

* `roaring.BitmapOf()`：传入集合元素，创建位图并添加这些元素
* `roaring.New()`：创建一个空位图

首先，我们创建了一个位图 bm1:{1,2,3,4,5,100,1000}。输出它的字符串表示，集合大小，检查 3 是否在集合中。

然后又创建了一个位图 bm2:{1,100,500}。输出检查三连。

接着创建了一个空位图 bm3，依次添加元素 1，11，111。输出检查三连。

然后我们对 bm1 和 bm2 执行并集，结果直接存放在 bm1 中。由于集合中的元素各不相同，此时 bm1 中的元素为{1,2,3,4,5,100,500,1000}，大小为 8。

再然后我们对 bm2 和 bm3 执行交集，结果直接存放在 bm2 中。此时 bm2 中的元素为{1}，大小为 1。

可以看出 roaring 提供的基本操作与 bitset 大体相同。只是命名完全不一样，在使用时需要特别注意。

* `bm.String()`：返回 bitmap 的字符串表示
* `bm.Add(n)`：添加元素 n
* `bm.GetCardinality()`：返回集合的基数（Cardinality），即元素个数
* `bm1.And(bm2)`：执行集合交集，会修改 bm1
* `bm1.Or(bm2)`：执行集合并集，会修改 bm1

### 迭代

roaring 位图支持迭代。

```golang
func main() {
  bm := roaring.BitmapOf(1, 2, 3, 4, 5, 100, 1000)

  i := bm.Iterator()
  for i.HasNext() {
    fmt.Println(i.Next())
  }
}
```

与很多编程语言支持的迭代器一样，先调用对象的`Iterator()`返回一个迭代器，然后循环调用`HasNext()`检查是否有下一个元素，调用`i.Next()`返回下一个元素。

上面代码依次输出 1，2，3，4，5，100，1000。

### 并行操作

roaring 支持位图集合运算的并行执行。可以指定使用多少个 goroutine 对集合执行交集、并集等。同时可以传入可变数量的位图集合：

```golang
func main() {
  bm1 := roaring.BitmapOf(1, 2, 3, 4, 5, 100, 1000)
  bm2 := roaring.BitmapOf(1, 100, 500)
  bm3 := roaring.BitmapOf(1, 10, 1000)

  bmAnd := roaring.ParAnd(4, bm1, bm2, bm3)
  fmt.Println(bmAnd.String())         // {1}
  fmt.Println(bmAnd.GetCardinality()) // 1
  fmt.Println(bmAnd.Contains(1))      // true
  fmt.Println(bmAnd.Contains(100))    // false

  bmOr := roaring.ParOr(4, bm1, bm2, bm3)
  fmt.Println(bmOr.String())         // {1,2,3,4,5,10,100,500,1000}
  fmt.Println(bmOr.GetCardinality()) // 9
  fmt.Println(bmOr.Contains(10))     // true
}
```

并行操作使用相应接口的`Par*`版本，第一个参数指定 worker 数量，接着传入任意多个 bitmap。

### 写入与读取

roaring 可以将压缩的位图写入到文件中，并且格式与其他语言的实现保持兼容。也就是说，我们可以用 Go 将 roaring 位图写入文件，然后通过网络发送给另一台机器，在这台机器上使用 C++ 或 Java 的实现读取这个文件。

```golang
func main() {
  bm := roaring.BitmapOf(1, 3, 5, 7, 100, 300, 500, 700)

  buf := &bytes.Buffer{}
  bm.WriteTo(buf)

  newBm := roaring.New()
  newBm.ReadFrom(buf)
  if bm.Equals(newBm) {
    fmt.Println("write and read back ok.")
  }
}
```

* `WriteTo(w io.Writer)`：写入一个 io.Writer，可以是内存（byte.Buffer），可以是文件（os.File），甚至可以是网络（net.Conn)
* `ReadFrom(r io.Reader)`：从一个 io.Reader 中读取，来源同样可以是内存、文件或网络等

注意`WriteTo`的返回值为`size`和`err`，使用时需要处理错误情况。`ReadFrom`也是返回`size`和`err`，同样需要处理处理。

### 64 位版本

默认情况下，roaring 位图只能用来存储 32 位整数。所以 roaring 位图最多能包含 4294967296（`2^32`） 个整数。

roaring 也提供了存储 64 位整数的扩展，即`github.com/RoaringBitmap/roaring/roaring64`。提供的接口基本相同。然而，64 位版本不保证与 Java/C++ 等格式兼容。

## 存储格式

roaring 可以写入文件中，也可以从文件中读取。并且提供多种语言兼容的格式。下面我们一起来看看存储的格式。

roaring 位图默认只能存储 32 位的整数。在序列化时，将这些整数分容器（container）存储。每个容器有一个 16 位表示的基数（Cardinality，即元素个数，范围`[1,2^16]`）和一个键（key）。键取元素的最高有效 16 位（most significant），所以键的范围为`[0, 65536)`。这样如果两个整数的最高 16 位有效位相同，那么它们将被保存在同一个容器中。这样做还有一个好处：可以减少占用的空间。

**所有整数均采用小端存储。**

### 概览

roaring 采用的存储格式布局如下：

![](/img/in-post/godailylib/roaring1.png#center)

从上到下依次介绍。

开始部分是一个 Cookie Header。它用来识别一个二进制流是不是一个 roaring 位图，并且存储一些少量信息。

cookie 这个词有点意思，本意是饼干。我的理解是指小物件，所以 http 中的 cookie 只是用来存储少量信息。这里的 Cookie Header 也是如此。

接下来是 Descriptive Header。见名知义，它用来描述容器的信息。后面会详细介绍容器。

接下来有一个可选的 Offset Header。它记录了每个容器相对于首位的偏移，这让我们可以随机访问任意容器。

最后一部分是存储实际数据的容器。roaring 中一共有 3 种类型的容器：

* array（数组型）：16bit 整数数组
* bitset（位集型）：使用上一篇文章介绍的 bitset 存储数据
* run：这个有点不好翻译。有些人可能听说过 run-length 编码，有翻译成游程编码的。即使用长度+数据来编码，比如"0000000000"可以编码成"10,0"，表示有 10 个 0。run 容器也是类似的，后文详述

设计这种的布局，是为了不用将存储的位图全部载入内存就可以随机读取它的数据。并且每个容器的范围相互独立，这使得并行计算变得容易。

### Cookie Header

Cookier Header 有两种类型，分别占用 32bit 和 64bit 的空间。

第一种类型，前 32bit 的值为 12346，此时紧接着的 32bit 表示容器数量（记为 n）。同时这意味着，后面没有 run 类型的容器。12346 这魔术数字被定义为常量`SERIAL_COOKIE_NO_RUNCONTAINER`，含义不言自明。

![](/img/in-post/godailylib/roaring2.png#center)

第二种类型，前 32bit 的最低有效 16 位的值为 12347。此时，最高有效 16 位存储的值等于容器数量-1。将 cookie 右移 16 位再加 1 即可得到容器数量。由于这种类型的容器数量不会为 0，采用这种编码我们能的容器数量会多上 1 个。这种方法在很多地方都有应用，例如 redis。后面紧接着会使用 `(n+7)/8` 字节（作为一个 bitset）表示后面的容器是否 run 容器。每位对应一个容器，1 表示对应的容器是 run 容器，0 表示不是 run 容器。

![](/img/in-post/godailylib/roaring3.png#center)

由于是小端存储，所以流的前 16bit 一定是 12346 或 12347。如果读取到了其它的值，说明文件损坏，直接退出程序即可。

### Descriptive Header

Cookie Header 之后就是 Descriptive Header。它使用一对 16bit 数据描述每个容器。一个 16bit 存储键（即整数的最高有效 16bit），另一个 16bit 存储对应容器的基数（Cardinality）-1（又见到了），即容器存储的整数数量）。如果有 n 个容器，则 Descriptive Header 需要 32n 位 或 4n 字节。

![](/img/in-post/godailylib/roaring4.png#center)

扫描 Descriptive Header 之后，我们就能知道每个容器的类型。如果 cookie 值为 12347，cookie 后有一个 bitset 表示每个容器是否是 run 类型。对于非 run 类型的容器，如果容器的基数（Cardinality）小于等于 4096，它是一个 array 容器。反之，这是一个 bitset 容器

### Offset Header

满足以下任一条件，Offset Header 就会存在：

* cookie 的值为 SERIAL_COOKIE_NO_RUNCONTAINER（即 12346）
* cookie 的值为 SERIAL_COOKIE（即 12347），并且至少有 4 个容器。也有一个常量`NO_OFFSET_THRESHOLD = 4`

Offset Header 为每个容器使用 32bit 值存储对应容器距离流开始处的偏移，单位字节。

![](/img/in-post/godailylib/roaring5.png#center)

### Container

接下来就是实际存储数据的容器了。前面简单提到过，容器有三种类型。

#### array

存储**有序的** 16bit 无符号整数值，有序便于使用二分查找提高效率。16bit 值只是数据的最低有效 16bit，还记得 Descriptive Header 中每个容器都有一个 16bit 的 key 吧。将它们拼接起来才是实际的数据。

如果容器有 x 个值，占用空间 2x 字节。

![](/img/in-post/godailylib/roaring6.png#center)

#### bitmap/bitset

bitset 容器固定使用 8KB 的空间，以 64bit 为单位（称为字，word）序列化。因此，如果值 j 存在，则第 j/64 个字（从 0 开始）的 j%64 位会被设置为 1（从 0 开始）。

![](/img/in-post/godailylib/roaring7.png#center)

#### run

以一个表示 run 数量的 16bit 整数开始。后续每个 run 用一对 16bit 整数表示，前一个 16bit 表示开始的值，后一个 16bit 表示长度-1（又双见到了）。例如，11,4 表示数据 11,12,13,14,15。

![](/img/in-post/godailylib/roaring8.png#center)

### 手撸解析代码

验证我们是否真的理解了 roaring 布局最有效的方法就是手撸一个解析。使用标准库`encoding/binary`可以很容易地处理大小端问题。

定义常量:

```golang
const (
  SERIAL_COOKIE_NO_RUNCONTAINER = 12346
  SERIAL_COOKIE                 = 12347
  NO_OFFSET_THRESHOLD           = 4
)
```

读取 Cookie Header:

```golang
func readCookieHeader(r io.Reader) (cookie uint16, containerNum uint32, runFlagBitset []byte) {
  binary.Read(r, binary.LittleEndian, &cookie)
  switch cookie {
  case SERIAL_COOKIE_NO_RUNCONTAINER:
    var dummy uint16
    binary.Read(r, binary.LittleEndian, &dummy)
    binary.Read(r, binary.LittleEndian, &containerNum)

  case SERIAL_COOKIE:
    var u16 uint16
    binary.Read(r, binary.LittleEndian, &u16)
    containerNum = uint32(u16)
    buf := make([]uint8, (containerNum+7)/8)
    r.Read(buf)
    runFlagBitset = buf[:]

  default:
    log.Fatal("unknown cookie")
  }

  fmt.Println(cookie, containerNum, runFlagBitset)
  return
}
```

读取 Descriptive Header:

```golang
func readDescriptiveHeader(r io.Reader, containerNum uint32) []KeyCard {
  var keycards []KeyCard
  var key uint16
  var card uint16
  for i := 0; i < int(containerNum); i++ {
    binary.Read(r, binary.LittleEndian, &key)
    binary.Read(r, binary.LittleEndian, &card)
    card += 1
    fmt.Println("container", i, "key", key, "card", card)

    keycards = append(keycards, KeyCard{key, card})
  }

  return keycards
}
```

读取 Offset Header:

```golang
func readOffsetHeader(r io.Reader, cookie uint16, containerNum uint32) {
  if cookie == SERIAL_COOKIE_NO_RUNCONTAINER ||
    (cookie == SERIAL_COOKIE && containerNum >= NO_OFFSET_THRESHOLD) {
    // have offset header
    var offset uint32
    for i := 0; i < int(containerNum); i++ {
      binary.Read(r, binary.LittleEndian, &offset)
      fmt.Println("offset", i, offset)
    }
  }
}
```

读取容器，根据类型调用不同的函数:

```golang
// array
func readArrayContainer(r io.Reader, key, card uint16, bm *roaring.Bitmap) {
  var value uint16
  for i := 0; i < int(card); i++ {
    binary.Read(r, binary.LittleEndian, &value)
    bm.Add(uint32(key)<<16 | uint32(value))
  }
}

// bitmap
func readBitmapContainer(r io.Reader, key, card uint16, bm *roaring.Bitmap) {
  var u64s [1024]uint64
  for i := 0; i < 1024; i++ {
    binary.Read(r, binary.LittleEndian, &u64s[i])
  }

  bs := bitset.From(u64s[:])
  for i := uint32(0); i < 8192; i++ {
    if bs.Test(uint(i)) {
      bm.Add(uint32(key)<<16 | i)
    }
  }
}

// run
func readRunContainer(r io.Reader, key uint16, bm *roaring.Bitmap) {
  var runNum uint16
  binary.Read(r, binary.LittleEndian, &runNum)

  var startNum uint16
  var length uint16
  for i := 0; i < int(runNum); i++ {
    binary.Read(r, binary.LittleEndian, &startNum)
    binary.Read(r, binary.LittleEndian, &length)
    length += 1
    for j := uint16(0); j < length; j++ {
      bm.Add(uint32(key)<<16 | uint32(startNum+j))
    }
  }
}
```

整合:

```golang
func main() {
  data, err := ioutil.ReadFile("../roaring.bin")
  if err != nil {
    log.Fatal(err)
  }

  r := bytes.NewReader(data)
  cookie, containerNum, runFlagBitset := readCookieHeader(r)

  keycards := readDescriptiveHeader(r, containerNum)
  readOffsetHeader(r, cookie, containerNum)

  bm := roaring.New()
  for i := uint32(0); i < uint32(containerNum); i++ {
    if runFlagBitset != nil && runFlagBitset[i/8]&(1<<(i%8)) != 0 {
      // run
      readRunContainer(r, keycards[i].key, bm)
    } else if keycards[i].card <= 4096 {
      // array
      readArrayContainer(r, keycards[i].key, keycards[i].card, bm)
    } else {
      // bitmap
      readBitmapContainer(r, keycards[i].key, keycards[i].card, bm)
    }
  }

  fmt.Println(bm.String())
}
```

我将**写入读取**那个示例中的 byte.Buffer 保存到文件`roaring.bin`中。上面的程序就可以解析这个文件:

```
12346 1 []
container 0 key 0 card 8
offset 0 16
{1,3,5,7,100,300,500,700}
```

成功还原了位图😀

## 总结

本文我们首先介绍了 roaring 压缩位图的使用。如果不考虑内部实现，压缩位图和普通的位图在使用上并没有多少区别。

然后我通过 8 张原理图详细分析了存储的格式。

最后通过手撸一个解析来加深对原理的理解。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. roaring GitHub：[github.com/RoaringBitmap/roaring](https://github.com/RoaringBitmap/roaring)
2. roaring 文件格式：[https://github.com/RoaringBitmap/RoaringFormatSpec](https://github.com/RoaringBitmap/RoaringFormatSpec)
3. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)