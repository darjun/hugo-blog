---
layout:     post
title:      "深入理解Go之BCE优化"
subtitle:   "\"More Effecient Go\""
date:       2018-05-07T11:00:00 
author:     "darjun"
image: "/img/post-bg-2015.jpg"
tags:
    - 深入理解Go
URL: "/2018/05/07/bounds-check-elimination/"
categories: [
    "Go"
]
---

## 概述
自[Go 1.7][1]以后，标准Go编译器采用了一个新的编译器后端。该后端基于[静态单赋值形式][2]（简称SSA）。SSA利用BCE（Bounds Check Elimination）和CSE（Common Subexpression Elimination)优化可以让Go编译器生成更高效的代码。

BCE顾名思义，叫做边界检测消除。通常，Go程序中对一个slice或string的越界访问会抛出异常。有两种形式的边界检测：索引（a[i]）和切片（a[i:j]）。Go编译器会在每个访问处插入边界检测的代码。不可否认，边界检测是非常重要的。它能帮助我们在早期捕获常见的编程错误。但是在大多数情况下，这都是没有必要的，完全可以根据上下文进行推断。例：
```
package main

func f(s []int) {
    _ = s[2]
    _ = s[1]
    _ = s[0]
}

func main() {}
```

在Go的早期版本（1.7之前），上面的方法中每处切片索引都会做边界检测。但是我们知道，只需要给`_ = s[2]`添加边界检测。因为如果索引2未越界，1和0不可能越界。

在Go 1.7之后的版本中，后两行代码的边界检测就被消除了。可以使用`-gcflags="-d=ssa/check_bce/debug=1"`查看是否引入了边界检测：
```
$ go build -gcflags="-d=ssa/check_bce/debug=1" main.go
```
输出：
```
.\main.go:4:7: Found IsInBounds
```

那么什么情况下的边界检测会消除呢？

## 边界检测消除的情形

### 1. 重复检测
相同的索引被重复访问时，后面的访问无需边界检测。例：
```
var s[]int

_ = s[i] // 插入边界检测
_ = s[i] // 相同索引，不需要
```
```
_ = s[:i] // 插入边界检测
_ = s[:i] // 不需要
```
```
_ = s[i:] // 插入边界检测
_ = s[i:] // 不需要
```

### 2. 常量大小
访问数组（大小固定），并且索引使用某些位操作计算得到。例：
```
var a[17]int
_ = a[i&5] // 因为i&5 <= 5 < 17，边界检测消除
_ = a[i%5] // 不能消除，因为有可能为负数
```

### 3. 常量索引
使用常量索引访问。例：
```
var s[]int
if 5 < len(s) {
    _ = s[5] // 5 < len(s)，边界检测消除
}
```

### 4. 常量索引常量大小
使用常量索引访问数组。例：
```
var s[10]int
_ = s[5]   // 5 < 10，边界检测消除
```

### 5. 迭代中索引
当使用迭代中索引时，一些边界检测可以消除。例：
```
var s[]int
for i := range s {
    _ = s[i] // 消除
    _ = s[i:] // 消除
    _ = s[:i] // 消除
}
```

```
var s[]int
for i := 0; i < len(s); i++ {
    _ = s[i] // 消除
    _ = s[i:] // 消除
    _ = s[:i] // 消除
}
```

```
var a[]int
for i := range s {
    _ = s[:i+1] // 消除
    _ = s[i+1:] // 消除
}
```

### 6. 逐渐减小的常量索引
正如概述中例子：
```
var s[]int
_ = s[3] // 插入边界检测
_, _, _ = s[1], s[2], s[3] // 消除

// 或
_, _, _ = s[3], s[2], s[1] // 一个边界检测

// 或
if len(s) < 3 {
    _, _, _ = s[0], s[1], s[2] // 没有边界检测
}
```

### 7. 特殊情况
```
var s[]int
var index int
_ = s[:index] // 插入边界检测
_ = s[index:] // 插入边界检测
```
为什么上面两条赋值语句都需要边界检测？因为一个子切片可以比原来的切片大。
在Go中，`s[a:b]`有两个限制：
```
0 <= a <= len(s)
a <= b <= cap(s)
```

例如：
```
s0 := make([]int, 5, 10) // len(s0) == 5, cap(s0) == 10

index := 8

_ = s0[:index] // 这行没问题

// 因为index > len(s0)，所以异常了
_ = s0[index:] // panic: runtime error: slice bounds out of range
```

## 程序优化
我们可以利用边界检测消除优化程序性能。为了能够利用BCE，需要对程序做一点细小的调整。下面有几个例子：

例一：
```
func f(is []int, bs []byte) {
    if len(is) >= 256 {
        for _, n := range bs {
            _ = is[n] // for循环内边界检测会执行多次
        }
    }
}
```
可以调整为：
```
func f(is []int, bs []byte) {
    if len(is) >= 256 {
        is = is[:256] // 避免下面for循环中的边界检测
        for _, n := range bs {
            _ = is[n] // 边界检测消除！
        }
    }
}
```

例二：
```
func f(isa []int, isb []int) {
    if len(isa) > 0xFFF {
        for _, n := range isb {
            _ = isa[n & 0xFFF] // for循环内边界检测会执行多次
        }
    }
}
```
可以调整为：
```
func f(isa []int, isb []int) {
    if len(isa) > 0xFFF {
        isa = isa[:0xFFF+1] // 避免下面for循环中的边界检测
        for _, n := range isb {
            _ = isa[n & 0xFFF] // 边界检测消除！
        }
    }
}
```

例三：
```
func f(s []int) []int {
    s2 := make([]int, len(s))
    for i := range s {
        s2[i] = -s[i] // 插入边界检测
    }
    return s2
}
```
可以调整为：
```
func f(s []int) []int {
    s2 := make([]int, len(s))
    s2 = s2[:len(s)] // 避免下面的边界检测
    for i := range s {
        s2[i] = -s[i] // 边界检测消除！
    }
    return s2
}
```

## 参考链接：
1. [gBounds Checking Elimination][3]
2. [https://go101.org/article/bounds-check-elimination.html][4]


[1]: https://blog.golang.org/go1.7
[2]: https://zh.wikipedia.org/wiki/%E9%9D%99%E6%80%81%E5%8D%95%E8%B5%8B%E5%80%BC%E5%BD%A2%E5%BC%8F
[3]: https://docs.google.com/document/d/1vdAEAjYdzjnPA9WDOQ1e4e05cYVMpqSxJYZT33Cqw2g/edit
[4]: https://go101.org/article/bounds-check-elimination.html