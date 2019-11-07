---
layout:		post
title:		"高效生成JSON串——json-gen"
subtitle: 	"精准控制内存分配"
date:		2019-10-08T12:30:00
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - 开源项目
URL: "2019/10/08/golang-json-gen/"
categories: [ 
	"Go"
]
---

## 概述

游戏服务端的很多操作（包括玩家的和非玩家的）需要传给公司中台收集汇总，根据运营的需求分析数据。中台那边要求传过去的数据为 JSON 格式。一开始我们使用 golang 标准库中的`encoding/json`，发现性能不够理想（因为序列化使用了反射，涉及多次内存分配）。由于数据原始格式都是`map[string]interface{}`，且需要自己一个字段一个字段构造，于是我想可以在构造过程中就计算出最终 JSON 串的长度，那么就只需要一次内存分配了。

## 使用

下载：

```golang
$ go get github.com/darjun/json-gen
```

导入：

```golang
import (
  jsongen "github.com/darjun/json-gen"
)
```

使用起来还是比较方便的：

```golang
m := jsongen.NewMap()
m.PutUint("key1", 123)
m.PutInt("key2", -456)
m.PutUintArray("key3", []uint64{78, 90})
data := m.Serialize(nil)
```

`data`即为最终序列化完成的 JSON 串。当然，类型可以任意嵌套。代码参见[github](https://github.com/darjun/json-gen)。

`github`上有 Benchmark，是标准 JSON 库的性能的 10 倍！

| Library | Time/op(ns) |   B/op   | allocs/op |
|---------|---------|----------|-----------|
| encoding/json | 22209 | 6673 | 127 |
| darjun/json-gen | 3300 | 1152 | 1 |

## 实现

首先定义一个接口`Value`，所有可以序列化为 JSON 的值都实现这个接口：

```golang
type Value interface {
  Serialize(buf []byte) []byte
  Size() int
}
```

* `Serialize`可以传入一个分配好的内存，该方法会将值序列化后的 JSON 串追加到`buf`后面。
* `Size`返回该值最终在 JSON 串中占用的字节数。

### 分类

我将可序列化为 JSON 串的值分为了 4 类：

* `QuotedValue`：在最终的串中需要用`"`包裹起来的值，例如 golang 中的字符串。
* `UnquotedValue`：在最终的串中不需要用`"`包裹起来的值，例如`uint/int/bool/float32`等。
* `Array`：对应 JSON 中的数组。
* `Map`：对应 JSON 中的映射。

目前这 4 种类型已经可以满足我的需求了，后续扩展也很方便，只需要实现`Value`接口即可。下面根据`Value`的两个接口讨论这 4 种类型的实现。

### `QuotedValue`

底层基于`string`类型定义`QuotedValue`：

```golang
type QuotedValue string
```

由于`QuotedValue`最终在 JSON 串中会有 2 个`"`，故其大小为：长度 + 2。我们来看`Serialize`和`Size`方法的实现：

```golang
func (q QuotedValue) Serialize(buf []byte) []byte {
  buf = append(buf, '"')
  buf = append(buf, []byte(q)...)
  return append(buf, '"')
}

func (q QuotedValue) Size() int {
  return len(q) + 2
}
```

### `UnquotedValue`

同样基于`string`类型定义`UnquotedValue`：

```golang
type UnquotedValue string
```

与`QuotedValue`不同的是，`UnquotedValue`不需要`"`包裹，`Serialize`和`Size`方法的实现可以参见上面，比较简单！

### `Array`

`Array`表示一个 JSON 的数组。因为 JSON 数组可以包含任意类型的数据，我们可以基于`[]Value`为底层类型定义`Array`：

```golang
type Array []Value
```

这样`Array`在最终 JSON 串中占用的字节包括所有元素大小、元素之间的`,`和数组前后的`[]`，`Size`方法实现如下：

```golang
func (a Array) Size() int {
  size := 0
  for _, e := range a {
    // 递归求元素的大小
    size += e.Size()
  }

  // for []
  size += 2
  if len(a) > 1 {
    // for ,
    size += len(a) - 1
  }

  return size
}
```

`Serialize`方法递归调用元素的`Serialize`方法，在元素之间添加`,`，整个数组用`[]`包裹。

```golang
func (a Array) Serialize(buf []byte) []byte {
  if len(buf) == 0 {
    // 如果未传入分配好的空间，根据 Size 分配空间
    buf = make([]byte, 0, a.Size())
  }

  buf = append(buf, '[')
  count := len(a)
  for i, e := range a {
    buf = e.Serialize(buf)
    if i != count-1 {
      // 除了最后一个元素，每个元素后添加,
      buf = append(buf, ',')
    }
  }

  return append(buf, ']')
}
```

为了方便操作数组，我给数组添加很多方法，常用的基本类型和`Array/Map`都有对应的操作方法。操作方法命名为`AppendType`和`AppendTypeArray`（其中`Type`为`uint/int/bool/float/Array/Map`等类型名）。

除了`string/Array/Map`，其它的基本类型都使用`strconv`转为字符串，且强制转换为`UnquotedValue`，因为它不需要`"`包裹。

```golang
func (a *Array) AppendUint(u uint64) {
  value := strconv.FormatUint(u, 10)

	*a = append(*a, UnquotedValue(value))
}

func (a *Array) AppendString(value string) {
	*a = append(*a, QuotedValue(escapeString(value)))
}

func (a *Array) AppendUintArray(u []uint64) {
	value := make([]Value, 0, len(u))
	for _, v := range u {
		value = append(value, UnquotedValue(strconv.FormatUint(v, 10)))
	}

	*a = append(*a, Array(value))
}

func (a *Array) AppendStringArray(s []string) {
	value := make([]Value, 0, len(s))
	for _, v := range s {
		value = append(value, QuotedValue(escapeString(v)))
	}

	*a = append(*a, Array(value))
}
```

这里有点需要注意，由于`Append*`方法会修改`Array`（即切片），所以接收者需要使用指针！

### `Map`

实现`Map`时，有两种选择。第一种定义为`map[string]Value`，这样结构简单，但是由于`map`遍历的随机性会导致同一个`Map`生成的 JSON 串不一样。最终我选择了第二种方案，即键和值分开存放，这样可以保证在最终的 JSON 串中，键的顺序与插入的顺序相同：

```golang
type Map struct {
  keys []string
  values []Value
}
```

`Map`的大小包含多个部分：

* 键和值的大小。
* 前后需要`{}`包裹。
* 每个键需要用`"`包裹。
* 键和值之间需要有一个`:`。
* 每个键值对之间需要用`,`分隔。

搞清楚了这些组成部分，`Size`方法的实现就简单了：

```golang
func (m Map) Size() int {
  size := 0
	for i, key := range m.keys {
		// +2 for ", +1 for :
		size += len(key) + 2 + 1
		size += m.values[i].Size()
	}

	// +2 for {}
	size += 2

	if len(m.keys) > 1 {
		// for ,
		size += len(m.keys) - 1
	}

	return size
}
```

`Serialize`将多个键值对组装：

```golang
func (m Map) Serialize(buf []byte) []byte {
	if len(buf) == 0 {
		buf = make([]byte, 0, m.Size())
	}

	buf = append(buf, '{')
	count := len(m.keys)
	for i, key := range m.keys {
		buf = append(buf, '"')
		buf = append(buf, []byte(key)...)
		buf = append(buf, '"')
		buf = append(buf, ':')
		buf = m.values[i].Serialize(buf)
		if i != count-1 {
			buf = append(buf, ',')
		}
	}
	return append(buf, '}')
}
```

与`Array`类似，为了方便操作`Map`，我给`Map`添加了很多方法，常见的基本数据类型和`Array/Map`都有对应的操作方法。操作方法命名为`PutType`和`PutTypeArray`（其中`Type`为`uint/int/bool/float/Array/Map`等）。

```golang
func (m *Map) put(key string, value Value) {
	m.keys = append(m.keys, key)
	m.values = append(m.values, value)
}

func (m *Map) PutUint(key string, u uint64) {
	value := strconv.FormatUint(u, 10)

	m.put(key, UnquotedValue(value))
}

func (m *Map) PutUintArray(key string, u []uint64) {
	value := make([]Value, 0, len(u))
	for _, v := range u {
		value = append(value, UnquotedValue(strconv.FormatUint(v, 10)))
	}

	m.put(key, Array(value))
}
```

## 结语

我根据自身需求实现了一个生成 JSON 串的库，性能大为提升，尽管还不完善，但是后续扩展也非常简单。希望能给有相同需求的朋友带来启发。