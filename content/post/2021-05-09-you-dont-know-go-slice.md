---
layout:    post
title:    "你不知道的 Go 之 slice"
subtitle: 	"深入理解背后的结构和原理"
date:		2021-05-09T12:15:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/05/09/youdontknowgo/slice"
categories: [
	"Go"
]
---

## 简介

切片（slice）是 Go 语言提供的一种数据结构，使用非常简单、便捷。但是由于实现层面的原因，切片也经常会产生让人疑惑的结果。掌握切片的底层结构和原理，可以避免很多常见的使用误区。

## 底层结构

切片结构定义在源码`runtime`包下的 slice.go 文件中：

```golang
// src/runtime/slice.go
type slice struct {
  array unsafe.Pointer
  len int
  cap int
}
```

* `array`：一个指针，指向底层存储数据的数组
* `len`：切片的长度，在代码中我们可以使用`len()`函数获取这个值
* `cap`：切片的容量，**即在不扩容的情况下，最多能容纳多少元素**。在代码中我们可以使用`cap()`函数获取这个值

![](/img/in-post/youdontknowgo/slice1.png#center)

我们可以通过下面的代码输出切片的底层结构：

```golang
type slice struct {
  array unsafe.Pointer
  len   int
  cap   int
}

func printSlice() {
  s := make([]uint32, 1, 10)
  fmt.Printf("%#v\n", *(*slice)(unsafe.Pointer(&s)))
}

func main() {
  printSlice()
}
```

运行输出：

```
main.slice{array:(unsafe.Pointer)(0xc0000d6030), len:1, cap:10}
```

这里注意一个细节，由于`runtime.slice`结构是非导出的，我们不能直接使用。所以我在代码中手动定义了一个`slice`结构体，字段与`runtime.slice`结构相同。

我们结合切片的底层结构，先回顾一下切片的基础知识，然后再逐一看看切片的常见问题。

## 基础知识

### 创建切片

创建切片有 4 种方式：

1. `var`

`var`声明切片类型的变量，这时切片值为`nil`。

```golang
var s []uint32
```

这种方式创建的切片，`array`字段为空指针，`len`和`cap`字段都等于 0。

2. 切片字面量 

使用切片字面量将所有元素都列举出来，这时切片长度和容量都等于指定元素的个数。

```golang
s := []uint32{1, 2, 3}
```

创建之后`s`的底层结构如下：

![](/img/in-post/youdontknowgo/slice2.png#center)

`len`和`cap`字段都等于 3。

3. `make`

使用`make`创建，可以指定长度和容量。格式为`make([]type, len[, cap])`，可以只指定长度，也可以长度容量同时指定：

```golang
s1 := make([]uint32)
s2 := make([]uint32, 1)
s3 := make([]uint32, 1, 10)
```

4. 切片操作符

使用切片操作符可以从现有的切片或数组中切取一部分，创建一个新的切片。切片操作符格式为`[low:high]`，例如：

```golang
var arr [10]uint32
s1 := arr[0:5]
s2 := arr[:5]
s3 := arr[5:]
s4 := arr[:]
```

区间是左闭右开的，即`[low, high)`，包括索引`low`，不包括`high`。切取生成的切片长度为`high-low`。

另外`low`和`high`都有默认值。`low`默认为 0，`high`默认为原切片或数组的长度。它们都可以省略，省略时，相当于取默认值。

使用这种方式创建的切片底层共享相同的数据空间，在进行切片操作时可能会造成数据覆盖，要格外小心。

### 添加元素

可以使用`append()`函数向切片中添加元素，可以一次添加 0 个或多个元素。如果剩余空间（即`cap-len`）足够存放元素则直接将元素添加到后面，然后增加字段`len`的值即可。反之，则需要扩容，分配一个更大的数组空间，将旧数组中的元素复制过去，再执行添加操作。

```golang
package main

import "fmt"

func main() {
  s := make([]uint32, 0, 4)

  s = append(s, 1, 2, 3)
  fmt.Println(len(s), cap(s)) // 3 4

  s = append(s, 4, 5, 6)
  fmt.Println(len(s), cap(s)) // 6 8
}
```

## 你不知道的 slice

1. 空切片等于`nil`吗？

下面代码的输出什么？

```golang
func main() {
  var s1 []uint32
  s2 := make([]uint32, 0)

  fmt.Println(s1 == nil)
  fmt.Println(s2 == nil)
  fmt.Println("nil slice:", len(s1), cap(s1))
  fmt.Println("cap slice:", len(s2), cap(s2))
}
```

分析：

首先`s1`和`s2`的长度和容量都为 0，这很好理解。比较切片与`nil`是否相等，实际上要检查`slice`结构中的`array`字段是否是空指针。显然`s1 == nil`返回`true`，`s2 == nil`返回`false`。尽管`s2`长度为 0，但是`make()`为它分配了空间。**所以，一般定义长度为 0 的切片使用`var`的形式**。

2. 传值还是传引用？

下面代码的输出什么？

```golang
func main() {
  s1 := []uint32{1, 2, 3}
  s2 := append(s1, 4)

  fmt.Println(s1)
  fmt.Println(s2)
}
```

分析：

为什么`append()`函数要有返回值？因为我们将切片传递给`append()`时，其实传入的是`runtime.slice`结构。这个结构是按值传递的，所以函数内部对`array/len/cap`这几个字段的修改都不影响外面的切片结构。上面代码中，执行`append()`之后`s1`的`len`和`cap`保持不变，故输出为：

```
[1 2 3]
[1 2 3 4]
```

所以我们调用`append()`要写成`s = append(s, elem)`这种形式，将返回值赋值给原切片，从而覆写`array/len/cap`这几个字段的值。

初学者还可能会犯忽略`append()`返回值的错误：

```golang
append(s, elem)
```

这就更加大错特错了。添加的元素将会丢失，以为函数外切片的内部字段都没有变化。

我们可以看到，虽说切片是按引用传递的，但是实际上传递的是结构`runtime.slice`的值。只是对现有元素的修改会反应到函数外，因为底层数组空间是共用的。

3. 切片的扩容策略

下面代码的输出是什么？

```golang
func main() {
  var s1 []uint32
  s1 = append(s1, 1, 2, 3)
  s2 := append(s1, 4)
  fmt.Println(&s1[0] == &s2[0])
}
```

这涉及到切片的扩容策略。扩容时，若：

* 当前容量小于 1024，则将容量扩大为原来的 2 倍；
* 当前容量大于等于 1024，则将容量逐次增加原来的 0.25 倍，直到满足所需容量。

我翻看了 Go1.16 版本`runtime/slice.go`中扩容相关的源码，在执行上面规则后还会根据切片元素的大小和计算机位数进行相应的调整。整个过程比较复杂，感兴趣可以自行去研究。

我们只需要知道一开始容量较小，扩大为 2 倍，降低后续因添加元素导致扩容的频次。容量扩张到一定程度时，再按照 2 倍来扩容会造成比较大的浪费。

上面例子中执行`s1 = append(s1, 1, 2, 3)`后，容量会扩大为 4。再执行`s2 := append(s1, 4)`由于有足够的空间，`s2`底层的数组不会改变。所以`s1`和`s2`第一个元素的地址相同。

4. 切片操作符可以切取字符串

切片操作符可以切取字符串，但是与切取切片和数组不同。切取字符串返回的是字符串，而非切片。因为字符串是不可变的，如果返回切片。而切片和字符串共享底层数据，就可以通过切片修改字符串了。

```golang
func main() {
  str := "hello, world"
  fmt.Println(str[:5])
}
```

输出 hello。

5. 切片底层数据共享

下面代码的输出是什么？

```golang
func main() {
  array := [10]uint32{1, 2, 3, 4, 5}
  s1 := array[:5]

  s2 := s1[5:10]
  fmt.Println(s2)

  s1 = append(s1, 6)
  fmt.Println(s1)
  fmt.Println(s2)
}
```

分析：

首先注意到`s2 := s1[5:10]`上界 10 已经大于切片`s1`的长度了。要记住，**使用切片操作符切取切片时，上界是切片的容量，而非长度**。这时两个切片的底层结构有重叠，如下图：

![](/img/in-post/youdontknowgo/slice3.png#center)

这时输出`s2`为：

```golang
[0, 0, 0, 0, 0]
```

然后向切片`s1`中添加元素 6，这时结构如下图，其中切片`s1`和`s2`共享元素 6：

![](/img/in-post/youdontknowgo/slice4.png#center)

这时输出的`s1`和`s2`为：

```
[1, 2, 3, 4, 5, 6]
[6, 0, 0, 0, 0]
```

可以看到由于切片底层数据共享可能造成修改一个切片会导致其他切片也跟着修改。这有时会造成难以调试的 BUG。为了一定程度上缓解这个问题，Go 1.2 版本中提供了一个扩展切片操作符：```[low:high:max]```，用来限制新切片的容量。使用这种方式产生的切片容量为`max-low`。

```golang
func main() {
  array := [10]uint32{1, 2, 3, 4, 5}
  s1 := array[:5:5]

  s2 := array[5:10:10]
  fmt.Println(s2)

  s1 = append(s1, 6)
  fmt.Println(s1)
  fmt.Println(s2)
}
```

执行`s1 := array[:5:5]`我们限定了`s1`的容量为 5，这时结构如下图所示：

![](/img/in-post/youdontknowgo/slice5.png#center)

执行`s1 = append(s1, 6)`时，发现没有空闲容量了（因为`len == cap == 5`），重新创建一个底层数组再执行添加。这时结构如下图，`s1`和`s2`互不干扰：

![](/img/in-post/youdontknowgo/slice6.png#center)

## 总结

了解了切片的底层数据结构，知道了切片传递的是结构`runtime.slice`的值，我们就能解决 90% 以上的切片问题。再结合图形可以很直观的看到切片底层数据是如何操作的。

这个系列的名字是我仿造[《你不知道的 JavaScript》](https://github.com/getify/You-Dont-Know-JS)起的😀。

## 参考

1. 《Go 专家编程》，豆瓣链接：[https://book.douban.com/subject/35144587/](https://book.douban.com/subject/35144587/)
2. 你不知道的Go GitHub：[https://github.com/darjun/you-dont-know-go](https://github.com/darjun/you-dont-know-go)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~