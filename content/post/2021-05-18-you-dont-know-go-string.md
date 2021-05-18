---
layout:    post
title:    "你不知道的 Go 之 string"
subtitle: 	"深入理解背后的知识和误区"
date:		2021-05-18T06:35:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/05/18/youdontknowgo/string"
categories: [
	"Go"
]
---

## 简介

字符串（string）是 Go 语言提供的一种基础数据类型。在编程开发中几乎随时都会使用。本文介绍字符串相关的知识，帮助你更好地理解和使用它。

## 底层结构

字符串底层结构定义在源码`runtime`包下的 string.go 文件中：

```golang
// src/runtime/string.go
type stringStruct struct {
  str unsafe.Pointer
  len int
}
```

* `str`：一个指针，指向存储实际字符串的内存地址。
* `len`：字符串的长度。与切片类似，在代码中我们可以使用`len()`函数获取这个值。注意，**`len`存储实际的字节数，而非字符数。所以对于非单字节编码的字符，结果可能让人疑惑**。后面会详细介绍多字节字符。

对于字符串`Hello`，实际底层结构如下：

![](/img/in-post/youdontknowgo/string1.png#center)

`str`中存储的是字符对应的编码，`H`对应编码`72`，`e`对应`101`等等。

我们可以使用下面的代码输出字符串的底层结构和存储的每个字节：

```golang
package main

import (
  "fmt"
  "unsafe"
)

type stringStruct struct {
  str unsafe.Pointer
  len int
}

func main() {
  s := "Hello World!"
  fmt.Println(*(*stringStruct)(unsafe.Pointer(&s)))

  for _, b := range s {
    fmt.Println(b)
  }
}
```

运行输出：

```
{0x8edaff 5}
```

由于`runtime.stringStruct`结构是非导出的，我们不能直接使用。所以我在代码中手动定义了一个`stringStruct`结构体，字段与`runtime.stringStruct`完全相同。

## 基本操作

### 创建

创建字符串有两种基本方式，使用`var`定义和字符串字面量：

```golang
var s1 string
s2 := "Hello World!"
```

注意`var s string`定义了一个字符串的空值，字符串的空值是空字符串，即`""`。字符串不可能为`nil`。

字符串字面量可以使用**双引号**或**反引号**定义。在双引号中出现的特殊字符需要进行转义，而在单引号中不需要：

```golang
s1 := "Hello \nWorld"
s2 := `Hello
World`
```

上面代码中，`s1`中出现的换行符需要使用转义字符`\n`，`s2`中直接键入换行。由于单引号定义的字面量与我们在代码中看到的完全相同，在包含大段文本（通常有换行）或比较多的特殊字符时经常使用。另外使用单引号时，注意首行后面其他行的空格问题：

```golang
package main

import "fmt"

func main() {
  s := `hello
  world`

  fmt.Println(s)
}
```

可能只是为了缩进和美观，在第二行的 "world" 前面加上了两个空格。实际上这些空格也是字符串的一部分。如果这不是有意为之，可能会造成一些困惑。上面代码输出：

```
hello
  world
```

### 索引和切片

可以使用索引获取字符串对应位置上存储的字节值，使用切片操作符获取字符串的一个子串：

```golang
package main

import "fmt"

func main() {
  s := "Hello World!"
  fmt.Println(s[0])

  fmt.Println(s[:5])
}
```

输出：

```
72
Hello
```

上篇文章[你不知道的 Go 之 slice](https://darjun.github.io/2021/05/09/youdontknowgo/slice)中也介绍过了，字符串的切片操作返回的不是切片，而是字符串。

### 字符串拼接

字符串拼接最简单直白的方式就是使用`+`符号，`+`可以拼接任意多个字符串。但是`+`的缺点是待拼接的字符串必须是已知的。另一种方式就是使用标准库`strings`包中的`Join()`函数，这个函数接受一个字符串切片和一个分隔符，将切片中的元素拼接成以分隔符分隔的单个字符串：

```golang
func main() {
  s1 := "Hello" + " " + "World"
  fmt.Println(s1)

  ss := []string{"Hello", "World"}
  fmt.Println(strings.Join(ss, " "))
}
```

上面代码首先使用`+`拼接字符串，然后将各个字符串存放在一个切片中，使用`strings.Join()`函数拼接。结果是一样的。需要注意的是，**将待拼接的字符串放在一行中，使用`+`拼接，在 Go 语言内部会先计算需要的空间，预先分配这个空间，最后将各个字符串拷贝过去**。这个行为与其他很多语言是不同的，所以在 Go 语言中使用`+`拼接字符串不会有性能损失，甚至由于内部优化比其他方式性能还要更好一些。当然前提拼接是一次完成的。下面代码多次使用`+`拼接，会产生大量临时字符串对象，影响性能：

```golang
s := "hello"
var result string
for i := 1; i < 100; i++ {
  result += s
}
```

我们来测试一下各种方式的性能差异。首先定义 3 个函数，分别用 1 次`+`拼接，多次`+`拼接和`Join()`拼接：

```golang
func ConcatWithMultiPlus() {
  var s string
  for i := 0; i < 10; i++ {
    s += "hello"
  }
}

func ConcatWithOnePlus() {
  s1 := "hello"
  s2 := "hello"
  s3 := "hello"
  s4 := "hello"
  s5 := "hello"
  s6 := "hello"
  s7 := "hello"
  s8 := "hello"
  s9 := "hello"
  s10 := "hello"
  s := s1 + s2 + s3 + s4 + s5 + s6 + s7 + s8 + s9 + s10
  _ = s
}

func ConcatWithJoin() {
  s := []string{"hello", "hello", "hello", "hello", "hello", "hello", "hello", "hello", "hello", "hello"}
  _ = strings.Join(s, "")
}
```

然后在文件`benchmark_test.go`中定义基准测试：

```golang
func BenchmarkConcatWithOnePlus(b *testing.B) {
  for i := 0; i < b.N; i++ {
    ConcatWithOnePlus()
  }
}

func BenchmarkConcatWithMultiPlus(b *testing.B) {
  for i := 0; i < b.N; i++ {
    ConcatWithMultiPlus()
  }
}

func BenchmarkConcatWithJoin(b *testing.B) {
  for i := 0; i < b.N; i++ {
    ConcatWithJoin()
  }
}
```

运行测试：

```cmd
$ go test -bench .
BenchmarkConcatWithOnePlus-8            11884388               170.5 ns/op
BenchmarkConcatWithMultiPlus-8           1227411              1006 ns/op
BenchmarkConcatWithJoin-8                6718507               157.5 ns/op
```

可以看到，使用`+`一次拼接和`Join()`函数性能差不多，而多次`+`拼接性能是其他两种方式的近 1/9。另外需要注意我在`ConcatWithOnePlus()`函数中先定义 10 个字符串变量，然后再使用`+`拼接。如果直接使用`+`拼接字符串字面量，编译器会直接优化为一个字符串字面量，结果就没有可比较性了。

在`runtime`包中，使用`concatstrings()`函数来处理使用`+`拼接字符串的操作：

```golang
// src/runtime/string.go
func concatstrings(buf *tmpBuf, a []string) string {
  idx := 0
  l := 0
  count := 0
  for i, x := range a {
    n := len(x)
    if n == 0 {
      continue
    }
    if l+n < l {
      throw("string concatenation too long")
    }
    l += n
    count++
    idx = i
  }
  if count == 0 {
    return ""
  }

  // If there is just one string and either it is not on the stack
  // or our result does not escape the calling frame (buf != nil),
  // then we can return that string directly.
  if count == 1 && (buf != nil || !stringDataOnStack(a[idx])) {
    return a[idx]
  }
  s, b := rawstringtmp(buf, l)
  for _, x := range a {
    copy(b, x)
    b = b[len(x):]
  }
  return s
}
```

### 类型转换

我们经常需要将 string 转为 []byte，或者从 []byte 转换回 string。这中间都会涉及一次内存拷贝，所以要注意转换频次不宜过高。string 转换为 []byte，转换语法为`[]byte(str)`。首先创建一个`[]byte`并分配足够的空间，然后将 string 内容拷贝过去。

![](/img/in-post/youdontknowgo/string2.png#center)

```golang
func main() {
  s := "Hello"

  b := []byte(s)
  fmt.Println(len(b), cap(b))
}
```

注意，输出的`cap`可能与`len`不同，多出的容量处于对后续追加的性能考虑。

`[]byte`转换为 string 转换语法为`string(bs)`，过程也是类似。

## 你不知道的 string

### 1 编码

在计算机发展早期，只有单字节编码，最知名的是 ASCII（American Standard Code for Information Interchange，美国信息交换标准代码）。单字节编码最多只能编码 256 个字符，这对英语国家可能够用了。但是随着计算机在全世界的普及，要编码其他国家的语言（典型的就是汉字），单字节显然是不够的。为此提出了 Unicode 编码方案。Unicode 编码为全世界所有国家的语言符号规定了统一的编码方案。Unicode 相关的知识请查看参考链接[每个程序员都必须知道的 Unicode 知识](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)。

有很多人不知道 Unicode 与 UTF8、UTF16、UTF32 这些有什么关系。实际上可以理解为 Unicode 只是规定了每个字符对应的编码值，实际很少直接存储和传输这个值。UTF8/UTF16/UTF32 则定义这些编码值如何在内存或文件中存储以及在网络上传输的格式。例如，汉字“中”，Unicode 编码值为`00004E2D`，其他编码如下：

```
UTF8编码：E4B8AD
UTF16BE编码：FEFF4E2D
UTF16LE编码：FFFE2D4E
UTF32BE编码：0000FEFF00004E2D
UTF32LE编码：FFFE00002D4E0000
```

Go 语言中的字符串存储是 UTF-8 编码。UTF8 是可变长编码，优点是兼容 ASCII。对非英语国家的字符采用多字节编码方案，而且对使用比较频繁的字符采用较短的编码，提升编码效率。缺点是 UTF8 的可变长编码让我们不能直接、直观地确定字符串的**字符长度**。一般的中文字符使用 3 个字节来编码，例如上面的“中”。对于生僻字，可能采用更多的字节来编码，例如“魋”的 UTF-8 编码为`E9AD8B20`。

我们使用`len()`函数获取到的都是编码后的**字节长度**，而**非字符长度**，这一点在使用非 ASCII 字符时很重要：

```golang
func main() {
  s1 := "Hello World!"
  s2 := "你好，中国"

  fmt.Println(len(s1))
  fmt.Println(len(s2))
}
```

输出：

```
12
15
```

`Hello World!`有 12 个字符很好理解，`你好，中国`有 5 个中文字符，每个中文字符占 3 个字节，所以输出 15。

对于使用非 ASCII 字符的字符串，我们可以使用标准库的 unicode/utf8 包中的`RuneCountInString()`方法获取实际字符数：

```golang
func main() {
  s1 := "Hello World!"
  s2 := "你好，中国"

  fmt.Println(utf8.RuneCountInString(s1)) // 12
  fmt.Println(utf8.RuneCountInString(s2)) // 5
}
```

为了便于理解，下面给出字符串“中国”的底层结构图：

![](/img/in-post/youdontknowgo/string4.png#center)

### 2 索引和遍历

使用索引操作字符串，获取的是对应位置上的字节值，如果该位置是某个多字节编码的中间位置，可能返回的字节值不是一个合法的编码值：

```golang
s := "中国"
fmt.Println(s[0])
```

前面介绍过“中”的 UTF8 编码为`E4B8AD`，故`s[0]`取第一个字节值，结果为 228（十六进制 E4 的值）。

为了方便地遍历字符串，Go 语言中`for-range`循环对多字符编码有特殊的支持。每次遍历返回的索引是每个字符开始的**字节位置**，值为该字符的编码值：

```golang
func main() {
  s := "Go 语言"

  for index, c := range s {
    fmt.Println(index, c)
  }
}
```

所以遇到多字节字符，索引就不是连续的。上面“语”占用 3 个字节，所以“言”的索引就是“中”的索引 3 加上它的字节数 3，结果就是 6。上面的代码输出如下：

```
0 71
1 111
2 32
3 35821
6 35328
```

我们也可以以字符形式输出：

```golang
func main() {
  s := "Go 语言"

  for index, c := range s {
    fmt.Printf("%d %c\n", index, c)
  }
}
```

输出：

```
0 G
1 o
2 
3 语
6 言
```

按照这个方法，我们可以编写一个简单的`RuneCountInString()`函数，就叫做`Utf8Count`吧：

```golang
func Utf8Count(s string) int {
  var count int
  for range s {
    count++
  }
  return count
}

fmt.Println(Utf8Count("中国")) // 2
```

### 3 乱码和不可打印字符

如果 string 中出现不合法的 utf8 编码，打印时对于每个不合法的编码字节都会输出一个特定的符号`�`：

```golang
func main() {
  s := "中国"
  fmt.Println(s[:5])

  b := []byte{129, 130, 131}
  fmt.Println(string(b))
}
```

上面输出：

```
中��
���
```

因为“国”编码有 3 个字节，`s[:5]`只取了前两个，这两个字节无法组成一个合法的 UTF8 字符，故输出两个`�`。

另外需要警惕不可打印字符，之前有个同事请教我一个问题，两个字符串输出的内容相同，但是它们就是不相等：

```golang
func main() {
  b1 := []byte{0xEF, 0xBB, 0xBF, 72, 101, 108, 108, 111}
  b2 := []byte{72, 101, 108, 108, 111}

  s1 := string(b1)
  s2 := string(b2)

  fmt.Println(s1)
  fmt.Println(s2)
  fmt.Println(s1 == s2)
}
```

输出：

```
hello
hello
false
```

我直接把字符串内部字节写出来了，可能一眼就看出来了。但是我们当时遇到这个问题还是稍微费了一番功夫来调试的。因为当时字符串是从文件中读取的，而文件采用的是带 BOM 的 UTF8 编码格式。我们都知道 BOM 格式会自动在文件头部加上 3 个字节`0xEFBBBF`。而字符串比较是会比较长度和每个字节的。让问题更难调试的是，在文件中 BOM 头也是不显示的。

### 4 编译优化

`[]byte`转换为 string 的场景很多，处于性能上的考虑。如果转换后的 string 只是临时使用，这时转换并不会进行内存拷贝。返回的 `string`会指向切片的内存。编译器会识别如下场景：

* map 查找：`m[string(b)]`；
* 字符串拼接：`"<" + string(b) + ">"`；
* 字符串比较：`string(b) == "foo"`。

因为 string 只是临时使用，期间切片不会发生变化。故这样使用没有问题。

## 总结

字符串是使用频率最高的基本类型之一，熟悉掌握它可以帮助我们更好地编码和解决问题。

## 参考

1. 《Go 专家编程》
2. 每个程序员都必须知道的 Unicode 知识，[https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)
3. 你不知道的Go GitHub：[https://github.com/darjun/you-dont-know-go](https://github.com/darjun/you-dont-know-go)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~