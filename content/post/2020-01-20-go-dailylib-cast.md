---
layout:    post
title:    "Go 每日一库之 cast"
subtitle: 	"每天学习一个 Go 库"
date:		2020-01-18T21:08:05
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/01/20/godailylib/cast"
categories: [
	"Go"
]
---

## 简介

今天我们再来介绍 spf13 大神的另一个库[cast](https://github.com/spf13/cast)。`cast`是一个小巧、实用的类型转换库，用于将一个类型转为另一个类型。
最初开发`cast`是用在[hugo](https://gohugo.io/)中的。

## 快速使用

先安装：

```
$ go get github.com/spf13/cast
```

后使用：

```golang
package main

import (
  "fmt"

  "github.com/spf13/cast"
)

func main() {
  // ToString
  fmt.Println(cast.ToString("leedarjun"))        // leedarjun
  fmt.Println(cast.ToString(8))                  // 8
  fmt.Println(cast.ToString(8.31))               // 8.31
  fmt.Println(cast.ToString([]byte("one time"))) // one time
  fmt.Println(cast.ToString(nil))                // ""

  var foo interface{} = "one more time"
  fmt.Println(cast.ToString(foo))                // one more time

  // ToInt
  fmt.Println(cast.ToInt(8))      // 8
  fmt.Println(cast.ToInt(8.31))   // 8
  fmt.Println(cast.ToInt("8"))    // 8
  fmt.Println(cast.ToInt(true))   // 1
  fmt.Println(cast.ToInt(false))  // 0
  
  var eight interface{} = 8
  fmt.Println(cast.ToInt(eight))  // 8
  fmt.Println(cast.ToInt(nil))    // 0
}
```

实际上，`cast`实现了多种常见类型之间的相互转换，返回最符合直觉的结果。例如：

* `nil`转为`string`的结果为`""`，而不是`"nil"`；
* `true`转为`string`的结果为`"true"`，而`true`转为`int`的结果为`1`；
* `interface{}`转为其他类型，要看它里面存储的值类型。

这些类型包括所有的基本类型（整形、浮点型、布尔值和字符串）、空接口、`nil`，时间（`time.Time`）、时长（`time.Duration`）以及它们的切片类型，
还有`map[string]Type`（其中`Type`为前面提到的类型）：

```golang
byte     bool      float32    float64    string  
int8     int16     int32      int64      int
uint8    uint16    uint32     uint64     uint
interface{}   time.Time  time.Duration   nil
```

## 高级转换

`cast`提供了两组函数：

* `ToType`（其中`Type`可以为任何支持的类型），将参数转换为`Type`类型。如果无法转换，返回`Type`类型的零值或`nil`；
* `ToTypeE`以 E 结尾，返回转换后的值和一个`error`。这组函数可以区分参数中实际存储了零值，还是转换失败了。

实现上大部分代码都类似，`ToType`在内部调用`ToTypeE`函数，返回结果并忽略错误。`ToType`函数的实现在文件`cast.go`中，
而`ToTypeE`函数的实现在文件`caste.go`中。

```golang
// cast/cast.go
func ToBool(i interface{}) bool {
  v, _ := ToBoolE(i)
  return v
}

// ToDuration casts an interface to a time.Duration type.
func ToDuration(i interface{}) time.Duration {
  v, _ := ToDurationE(i)
  return v
}
```

`ToTypeE`函数都接受任意类型的参数（`interface{}`），然后使用类型断言根据具体的类型来执行不同的转换。如果无法转换，返回错误。

```golang
// cast/caste.go
func ToBoolE(i interface{}) (bool, error) {
  i = indirect(i)

  switch b := i.(type) {
  case bool:
    return b, nil
  case nil:
    return false, nil
  case int:
    if i.(int) != 0 {
      return true, nil
    }
    return false, nil
  case string:
    return strconv.ParseBool(i.(string))
  default:
    return false, fmt.Errorf("unable to cast %#v of type %T to bool", i, i)
  }
}
```

首先调用`indirect`函数将参数中可能的指针去掉。如果类型本身不是指针，那么直接返回。否则返回指针指向的值。
循环直到返回一个非指针的值：

```golang
// cast/caste.go
func indirect(a interface{}) interface{} {
  if a == nil {
    return nil
  }
  if t := reflect.TypeOf(a); t.Kind() != reflect.Ptr {
    // Avoid creating a reflect.Value if it's not a pointer.
    return a
  }
  v := reflect.ValueOf(a)
  for v.Kind() == reflect.Ptr && !v.IsNil() {
    v = v.Elem()
  }
  return v.Interface()
}
```

所以，下面代码输出都是 8：

```golang
package main

import (
	"fmt"

	"github.com/spf13/cast"
)

func main() {
  p := new(int)
  *p = 8
  fmt.Println(cast.ToInt(p))   // 8

  pp := &p
  fmt.Println(cast.ToInt(pp))  // 8
}
```

### 时间和时长转换

时间类型的转换代码如下：

```golang
func ToTimeE(i interface{}) (tim time.Time, err error) {
  i = indirect(i)

  switch v := i.(type) {
  case time.Time:
    return v, nil
  case string:
    return StringToDate(v)
  case int:
    return time.Unix(int64(v), 0), nil
  case int64:
    return time.Unix(v, 0), nil
  case int32:
    return time.Unix(int64(v), 0), nil
  case uint:
    return time.Unix(int64(v), 0), nil
  case uint64:
    return time.Unix(int64(v), 0), nil
  case uint32:
    return time.Unix(int64(v), 0), nil
  default:
    return time.Time{}, fmt.Errorf("unable to cast %#v of type %T to Time", i, i)
  }
}
```

根据传入的类型执行不同的处理：

* 如果是`time.Time`，直接返回；
* 如果是整型，将参数作为时间戳（自 UTC 时间`1970.01.01 00:00:00`到现在的秒数）调用`time.Unix`生成时间。`Unix`接受两个参数，第一个参数指定秒，第二个参数指定纳秒；
* 如果是字符串，调用`StringToDate`函数依次尝试以下面这些时间格式调用`time.Parse`解析该字符串。如果某个格式解析成功，则返回获得的`time.Time`。否则解析失败，返回错误；
* 其他任何类型都无法转换为`time.Time`。

字符串转换为时间：

```golang
// cast/caste.go
func StringToDate(s string) (time.Time, error) {
  return parseDateWith(s, []string{
    time.RFC3339,
    "2006-01-02T15:04:05", // iso8601 without timezone
    time.RFC1123Z,
    time.RFC1123,
    time.RFC822Z,
    time.RFC822,
    time.RFC850,
    time.ANSIC,
    time.UnixDate,
    time.RubyDate,
    "2006-01-02 15:04:05.999999999 -0700 MST", // Time.String()
    "2006-01-02",
    "02 Jan 2006",
    "2006-01-02T15:04:05-0700", // RFC3339 without timezone hh:mm colon
    "2006-01-02 15:04:05 -07:00",
    "2006-01-02 15:04:05 -0700",
    "2006-01-02 15:04:05Z07:00", // RFC3339 without T
    "2006-01-02 15:04:05Z0700",  // RFC3339 without T or timezone hh:mm colon
    "2006-01-02 15:04:05",
    time.Kitchen,
    time.Stamp,
    time.StampMilli,
    time.StampMicro,
    time.StampNano,
  })
}

func parseDateWith(s string, dates []string) (d time.Time, e error) {
  for _, dateType := range dates {
    if d, e = time.Parse(dateType, s); e == nil {
      return
    }
  }
  return d, fmt.Errorf("unable to parse date: %s", s)
}
```

时长类型的转换代码如下：

```golang
// cast/caste.go
func ToDurationE(i interface{}) (d time.Duration, err error) {
  i = indirect(i)

  switch s := i.(type) {
  case time.Duration:
    return s, nil
  case int, int64, int32, int16, int8, uint, uint64, uint32, uint16, uint8:
    d = time.Duration(ToInt64(s))
    return
  case float32, float64:
    d = time.Duration(ToFloat64(s))
    return
  case string:
    if strings.ContainsAny(s, "nsuµmh") {
      d, err = time.ParseDuration(s)
    } else {
      d, err = time.ParseDuration(s + "ns")
    }
    return
  default:
    err = fmt.Errorf("unable to cast %#v of type %T to Duration", i, i)
    return
  }
}
```

根据传入的类型进行不同的处理：

* 如果是`time.Duration`类型，直接返回；
* 如果是整型或浮点型，将其数值强制转换为`time.Duration`类型，单位默认为`ns`；
* 如果是字符串，分为两种情况：如果字符串中有时间单位符号`nsuµmh`，直接调用`time.ParseDuration`解析；否则在字符串后拼接`ns`再调用`time.ParseDuration`解析；
* 其他类型解析失败。

示例：

```golang
package main

import (
  "fmt"
  "time"

  "github.com/spf13/cast"
)

func main() {
  now := time.Now()
  timestamp := 1579615973
  timeStr := "2020-01-21 22:13:48"

  fmt.Println(cast.ToTime(now))       // 2020-01-22 06:31:50.5068465 +0800 CST m=+0.000997701
  fmt.Println(cast.ToTime(timestamp)) // 2020-01-21 22:12:53 +0800 CST
  fmt.Println(cast.ToTime(timeStr))   // 2020-01-21 22:13:48 +0000 UTC

  d, _ := time.ParseDuration("1m30s")
  ns := 30000
  strWithUnit := "130s"
  strWithoutUnit := "130"

  fmt.Println(cast.ToDuration(d))               // 1m30s
  fmt.Println(cast.ToDuration(ns))              // 30µs
  fmt.Println(cast.ToDuration(strWithUnit))     // 2m10s
  fmt.Println(cast.ToDuration(strWithoutUnit))  // 130ns
}
```

### 转换为切片

实际上，这些函数的实现基本类似。使用类型断言判断类型。如果就是要返回的类型，直接返回。否则根据类型进行相应的转换。

我们主要分析两个实现：`ToIntSliceE`和`ToStringSliceE`。`ToBoolSliceE/ToDurationSliceE`与`ToIntSliceE`基本相同。

首先是`ToIntSliceE`：

```golang
func ToIntSliceE(i interface{}) ([]int, error) {
  if i == nil {
    return []int{}, fmt.Errorf("unable to cast %#v of type %T to []int", i, i)
  }

  switch v := i.(type) {
  case []int:
    return v, nil
  }

  kind := reflect.TypeOf(i).Kind()
  switch kind {
  case reflect.Slice, reflect.Array:
    s := reflect.ValueOf(i)
    a := make([]int, s.Len())
    for j := 0; j < s.Len(); j++ {
      val, err := ToIntE(s.Index(j).Interface())
      if err != nil {
        return []int{}, fmt.Errorf("unable to cast %#v of type %T to []int", i, i)
      }
      a[j] = val
    }
    return a, nil
  default:
    return []int{}, fmt.Errorf("unable to cast %#v of type %T to []int", i, i)
  }
}
```

根据传入参数的类型：

* 如果是`nil`，直接返回错误；
* 如果是`[]int`，不用转换，直接返回；
* 如果传入类型为**切片**或**数组**，新建一个`[]int`，将切片或数组中的每个元素转为`int`放到该`[]int`中。最后返回这个`[]int`；
* 其他情况，不能转换。

`ToStringSliceE`：

```golang
func ToStringSliceE(i interface{}) ([]string, error) {
  var a []string

  switch v := i.(type) {
  case []interface{}:
    for _, u := range v {
      a = append(a, ToString(u))
    }
    return a, nil
  case []string:
    return v, nil
  case string:
    return strings.Fields(v), nil
  case interface{}:
    str, err := ToStringE(v)
    if err != nil {
      return a, fmt.Errorf("unable to cast %#v of type %T to []string", i, i)
    }
    return []string{str}, nil
  default:
    return a, fmt.Errorf("unable to cast %#v of type %T to []string", i, i)
  }
}
```

根据传入的参数类型：

* 如果是`[]interface{}`，将该参数中每个元素转为`string`，返回结果切片；
* 如果是`[]string`，不需要转换，直接返回；
* 如果是`interface{}`，将参数转为`string`，返回只包含这个值的切片；
* 如果是`string`，调用`strings.Fields`函数按空白符将参数拆分，返回拆分后的字符串切片；
* 其他情况，不能转换。

示例：

```golang
package main

import (
  "fmt"

  "github.com/spf13/cast"
)

func main() {
  sliceOfInt := []int{1, 3, 7}
  arrayOfInt := [3]int{8, 12}
  // ToIntSlice
  fmt.Println(cast.ToIntSlice(sliceOfInt))  // [1 3 7]
  fmt.Println(cast.ToIntSlice(arrayOfInt))  // [8 12 0]

  sliceOfInterface := []interface{}{1, 2.0, "darjun"}
  sliceOfString := []string{"abc", "dj", "pipi"}
  stringFields := " abc  def hij   "
  any := interface{}(37)
  // ToStringSliceE
  fmt.Println(cast.ToStringSlice(sliceOfInterface))  // [1 2 darjun]
  fmt.Println(cast.ToStringSlice(sliceOfString))     // [abc dj pipi]
  fmt.Println(cast.ToStringSlice(stringFields))      // [abc def hij]
  fmt.Println(cast.ToStringSlice(any))               // [37]
}
```

### 转为`map[string]Type`类型

`cast`库能将传入的参数转为`map[string]Type`类型，`Type`为上面支持的类型。

其实只需要分析一个`ToStringMapStringE`函数就可以了，其他的实现基本一样。`ToStringMapStringE`返回`map[string]string`类型的值。

```golang
func ToStringMapStringE(i interface{}) (map[string]string, error) {
  var m = map[string]string{}

  switch v := i.(type) {
  case map[string]string:
    return v, nil
  case map[string]interface{}:
    for k, val := range v {
      m[ToString(k)] = ToString(val)
    }
    return m, nil
  case map[interface{}]string:
    for k, val := range v {
      m[ToString(k)] = ToString(val)
    }
    return m, nil
  case map[interface{}]interface{}:
    for k, val := range v {
      m[ToString(k)] = ToString(val)
    }
    return m, nil
  case string:
    err := jsonStringToObject(v, &m)
    return m, err
  default:
    return m, fmt.Errorf("unable to cast %#v of type %T to map[string]string", i, i)
  }
}
```

根据传入的参数类型：

* 如果是`map[string]string`，不用转换，直接返回；
* 如果是`map[string]interface{}`，将每个值转为`string`存入新的 map，最后返回新的 map；
* 如果是`map[interface{}]string`，将每个键转为`string`存入新的 map，最后返回新的 map；
* 如果是`map[interface{}]interface{}`，将每个键和值都转为`string`存入新的 map，最后返回新的 map；
* **如果是`string`类型，`cast`将它看成一个 JSON 串，解析这个 JSON 到`map[string]string`，然后返回结果**；
* 其他情况，返回错误。

示例：

```golang
package main

import (
  "fmt"

  "github.com/spf13/cast"
)

func main() {
  m1 := map[string]string {
    "name": "darjun",
    "job": "developer",
  }

  m2 := map[string]interface{} {
    "name": "jingwen",
    "age": 18,
  }

  m3 := map[interface{}]string {
    "name": "pipi",
    "job": "designer",
  }

  m4 := map[interface{}]interface{} {
    "name": "did",
    "age": 29,
  }

  jsonStr := `{"name":"bibi", "job":"manager"}`

  fmt.Println(cast.ToStringMapString(m1))      // map[job:developer name:darjun]
  fmt.Println(cast.ToStringMapString(m2))      // map[age:18 name:jingwen] 
  fmt.Println(cast.ToStringMapString(m3))      // map[job:designer name:pipi]
  fmt.Println(cast.ToStringMapString(m4))      // map[job:designer name:pipi]
  fmt.Println(cast.ToStringMapString(jsonStr)) // map[job:manager name:bibi]
}
```

## 总结

`cast`库能在几乎所有常见类型之间转换，使用非常方便。代码量也很小，有时间建议读读源码。

完整示例代码在[GitHub](https://github.com/darjun/go-daily-lib/tree/master/cast)上。

ps: 本以为春节就这么几天，偷个懒，没想到。。。

## 参考

1. [cast](https://github.com/spf13/cast) GitHub 仓库

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)