---
layout:    post
title:    "Go 每日一库之 testing"
subtitle: "每天学习一个 Go 库"
date:     2021-08-03T23:30:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/08/03/godailylib/testing"
categories: [
  "Go"
]
---

## 简介

[`testing`](https://golang.google.cn/pkg/testing/)是 Go 语言标准库自带的测试库。在 Go 语言中编写测试很简单，只需要遵循 Go 测试的**几个约定**，与编写正常的 Go 代码没有什么区别。Go 语言中有 3 种类型的测试：单元测试，性能测试，示例测试。下面依次来介绍。

## 单元测试

单元测试又称为功能性测试，是为了测试函数、模块等代码的逻辑是否正确。接下来我们编写一个库，用于将表示罗马数字的字符串和整数互转。罗马数字是由`M/D/C/L/X/V/I`这几个字符根据一定的规则组合起来表示一个正整数：

* M=1000，D=500，C=100，L=50，X=10，V=5，I=1；
* 只能表示 1-3999 范围内的整数，不能表示 0 和负数，不能表示 4000 及以上的整数，不能表示分数和小数（当然有其他复杂的规则来表示这些数字，这里暂不考虑）；
* 每个整数只有一种表示方式，一般情况下，连写的字符表示对应整数相加，例如`I=1`，`II=2`，`III=3`。但是，**十位字符**（`I/X/C/M`）最多出现 3 次，所以不能用`IIII`表示 4，需要在`V`左边添加一个`I`（即`IV`）来表示，不能用`VIIII`表示 9，需要使用`IX`代替。另外**五位字符**（`V/L/D`）不能连续出现 2 次，所以不能出现`VV`，需要用`X`代替。

```golang
// roman.go
package roman

import (
  "bytes"
  "errors"
  "regexp"
)

type romanNumPair struct {
  Roman string
  Num   int
}

var (
  romanNumParis []romanNumPair
  romanRegex    *regexp.Regexp
)

var (
  ErrOutOfRange   = errors.New("out of range")
  ErrInvalidRoman = errors.New("invalid roman")
)

func init() {
  romanNumParis = []romanNumPair{
    {"M", 1000},
    {"CM", 900},
    {"D", 500},
    {"CD", 400},
    {"C", 100},
    {"XC", 90},
    {"L", 50},
    {"XL", 40},
    {"X", 10},
    {"IX", 9},
    {"V", 5},
    {"IV", 4},
    {"I", 1},
  }

  romanRegex = regexp.MustCompile(`^M{0,3}(CM|CD|D?C{0,3})(XC|XL|L?X{0,3})(IX|IV|V?I{0,3})$`)
}

func ToRoman(n int) (string, error) {
  if n <= 0 || n >= 4000 {
    return "", ErrOutOfRange
  }
  var buf bytes.Buffer
  for _, pair := range romanNumParis {
    for n > pair.Num {
      buf.WriteString(pair.Roman)
      n -= pair.Num
    }
  }

  return buf.String(), nil
}

func FromRoman(roman string) (int, error) {
  if !romanRegex.MatchString(roman) {
    return 0, ErrInvalidRoman
  }

  var result int
  var index int
  for _, pair := range romanNumParis {
    for roman[index:index+len(pair.Roman)] == pair.Roman {
      result += pair.Num
      index += len(pair.Roman)
    }
  }

  return result, nil
}
```

在 Go 中编写测试很简单，只需要在待测试功能所在文件的同级目录中创建一个以`_test.go`结尾的文件。在该文件中，我们可以编写一个个测试函数。测试函数名必须是`TestXxxx`这个形式，而且`Xxxx`必须以大写字母开头，另外函数带有一个`*testing.T`类型的参数：

```golang
// roman_test.go
package roman

import (
  "testing"
)

func TestToRoman(t *testing.T) {
  _, err1 := ToRoman(0)
  if err1 != ErrOutOfRange {
    t.Errorf("ToRoman(0) expect error:%v got:%v", ErrOutOfRange, err1)
  }

  roman2, err2 := ToRoman(1)
  if err2 != nil {
    t.Errorf("ToRoman(1) expect nil error, got:%v", err2)
  }
  if roman2 != "I" {
    t.Errorf("ToRoman(1) expect:%s got:%s", "I", roman2)
  }
}
```

在测试函数中编写的代码与正常的代码没有什么不同，调用相应的函数，返回结果，判断结果与预期是否一致，如果不一致则调用`testing.T`的`Errorf()`输出错误信息。运行测试时，这些错误信息会被收集起来，运行结束后统一输出。

测试编写完成之后，使用`go test`命令运行测试，输出结果：

```golang
$ go test
--- FAIL: TestToRoman (0.00s)
    roman_test.go:18: ToRoman(1) expect:I got:
FAIL
exit status 1
FAIL    github.com/darjun/go-daily-lib/testing  0.172s
```

我故意将`ToRoman()`函数中写错了一行代码，`n > pair.Num`中`>`应该为`>=`，单元测试成功找出了错误。修改之后重新运行测试：

```golang
$ go test
PASS
ok      github.com/darjun/go-daily-lib/testing  0.178s
```

这次测试都通过了！

我们还可以给`go test`命令传入`-v`选项，输出详细的测试信息：

```golang
$ go test -v
=== RUN   TestToRoman
--- PASS: TestToRoman (0.00s)
PASS
ok      github.com/darjun/go-daily-lib/testing  0.174s
```

在运行每个测试函数前，都输出一行`=== RUN`，运行结束之后输出`--- PASS`或`--- FAIL`信息。

### 表格驱动测试

在上面的例子中，我们实际上只测试了两种情况，0 和 1。按照这种方式将每种情况都写出来就太繁琐了，Go 中流行使用**表格**的方式将各个测试数据和结果列举出来：

```golang
func TestToRoman(t *testing.T) {
  testCases := []struct {
    num    int
    expect string
    err    error
  }{
    {0, "", ErrOutOfRange},
    {1, "I", nil},
    {2, "II", nil},
    {3, "III", nil},
    {4, "IV", nil},
    {5, "V", nil},
    {6, "VI", nil},
    {7, "VII", nil},
    {8, "VIII", nil},
    {9, "IX", nil},
    {10, "X", nil},
    {50, "L", nil},
    {100, "C", nil},
    {500, "D", nil},
    {1000, "M", nil},
    {31, "XXXI", nil},
    {148, "CXLVIII", nil},
    {294, "CCXCIV", nil},
    {312, "CCCXII", nil},
    {421, "CDXXI", nil},
    {528, "DXXVIII", nil},
    {621, "DCXXI", nil},
    {782, "DCCLXXXII", nil},
    {870, "DCCCLXX", nil},
    {941, "CMXLI", nil},
    {1043, "MXLIII", nil},
    {1110, "MCX", nil},
    {1226, "MCCXXVI", nil},
    {1301, "MCCCI", nil},
    {1485, "MCDLXXXV", nil},
    {1509, "MDIX", nil},
    {1607, "MDCVII", nil},
    {1754, "MDCCLIV", nil},
    {1832, "MDCCCXXXII", nil},
    {1993, "MCMXCIII", nil},
    {2074, "MMLXXIV", nil},
    {2152, "MMCLII", nil},
    {2212, "MMCCXII", nil},
    {2343, "MMCCCXLIII", nil},
    {2499, "MMCDXCIX", nil},
    {2574, "MMDLXXIV", nil},
    {2646, "MMDCXLVI", nil},
    {2723, "MMDCCXXIII", nil},
    {2892, "MMDCCCXCII", nil},
    {2975, "MMCMLXXV", nil},
    {3051, "MMMLI", nil},
    {3185, "MMMCLXXXV", nil},
    {3250, "MMMCCL", nil},
    {3313, "MMMCCCXIII", nil},
    {3408, "MMMCDVIII", nil},
    {3501, "MMMDI", nil},
    {3610, "MMMDCX", nil},
    {3743, "MMMDCCXLIII", nil},
    {3844, "MMMDCCCXLIV", nil},
    {3888, "MMMDCCCLXXXVIII", nil},
    {3940, "MMMCMXL", nil},
    {3999, "MMMCMXCIX", nil},
    {4000, "", ErrOutOfRange},
  }

  for _, testCase := range testCases {
    got, err := ToRoman(testCase.num)
    if got != testCase.expect {
      t.Errorf("ToRoman(%d) expect:%s got:%s", testCase.num, testCase.expect, got)
    }

    if err != testCase.err {
      t.Errorf("ToRoman(%d) expect error:%v got:%v", testCase.num, testCase.err, err)
    }
  }
}
```

上面将要测试的每种情况列举出来，然后针对每个整数调用`ToRoman()`函数，比较返回的罗马数字字符串和错误值是否与预期的相符。后续要添加新的测试用例也很方便。

### 分组和并行

有时候对同一个函数有不同维度的测试，将这些组合在一起有利于维护。例如上面对`ToRoman()`函数的测试可以分为非法值，单个罗马字符和普通 3 种情况。


为了分组，我对代码做了一定程度的重构，首先抽象一个`toRomanCase`结构：

```golang
type toRomanCase struct {
  num    int
  expect string
  err    error
}
```

将所有的测试数据划分到 3 个组中：

```golang
var (
  toRomanInvalidCases []toRomanCase
  toRomanSingleCases  []toRomanCase
  toRomanNormalCases  []toRomanCase
)

func init() {
  toRomanInvalidCases = []toRomanCase{
    {0, "", roman.ErrOutOfRange},
    {4000, "", roman.ErrOutOfRange},
  }

  toRomanSingleCases = []toRomanCase{
    {1, "I", nil},
    {5, "V", nil},
    // ...
  }

  toRomanNormalCases = []toRomanCase{
    {2, "II", nil},
    {3, "III", nil},
    // ...
  }
}
```

然后为了避免代码重复，抽象一个运行多个`toRomanCase`的函数：

```golang
func testToRomanCases(cases []toRomanCase, t *testing.T) {
  for _, testCase := range cases {
    got, err := roman.ToRoman(testCase.num)
    if got != testCase.expect {
      t.Errorf("ToRoman(%d) expect:%s got:%s", testCase.num, testCase.expect, got)
    }

    if err != testCase.err {
      t.Errorf("ToRoman(%d) expect error:%v got:%v", testCase.num, testCase.err, err)
    }
  }
}
```

为每个分组定义一个测试函数：

```golang
func testToRomanInvalid(t *testing.T) {
  testToRomanCases(toRomanInvalidCases, t)
}

func testToRomanSingle(t *testing.T) {
  testToRomanCases(toRomanSingleCases, t)
}

func testToRomanNormal(t *testing.T) {
  testToRomanCases(toRomanNormalCases, t)
}
```

在原来的测试函数中，调用`t.Run()`运行不同分组的测试函数，`t.Run()`第一个参数为子测试名，第二个参数为子测试函数：

```golang
func TestToRoman(t *testing.T) {
  t.Run("Invalid", testToRomanInvalid)
  t.Run("Single", testToRomanSingle)
  t.Run("Normal", testToRomanNormal)
}
```

运行：

```golang
$ go test -v
=== RUN   TestToRoman
=== RUN   TestToRoman/Invalid
=== RUN   TestToRoman/Single
=== RUN   TestToRoman/Normal
--- PASS: TestToRoman (0.00s)
    --- PASS: TestToRoman/Invalid (0.00s)
    --- PASS: TestToRoman/Single (0.00s)
    --- PASS: TestToRoman/Normal (0.00s)
PASS
ok      github.com/darjun/go-daily-lib/testing  0.188s
```

可以看到，依次运行 3 个子测试，子测试名是父测试名和`t.Run()`指定的名字组合而成的，如`TestToRoman/Invalid`。

默认情况下，这些测试都是依次顺序执行的。如果各个测试之间没有联系，我们可以让他们并行以加快测试速度。方法也很简单，在`testToRomanInvalid/testToRomanSingle/testToRomanNormal`这 3 个函数开始处调用`t.Parallel()`，由于这 3 个函数直接调用了`testToRomanCases`，也可以只在`testToRomanCases`函数开头出添加：

```golang
func testToRomanCases(cases []toRomanCase, t *testing.T) {
  t.Parallel()
  // ...
}
```

运行：

```golang
$ go test -v
...
--- PASS: TestToRoman (0.00s)
    --- PASS: TestToRoman/Invalid (0.00s)
    --- PASS: TestToRoman/Normal (0.00s)
    --- PASS: TestToRoman/Single (0.00s)
PASS
ok      github.com/darjun/go-daily-lib/testing  0.182s
```

我们发现测试完成的顺序并不是我们指定的顺序。

另外，这个示例中我将`roman_test.go`文件移到了`roman_test`包中，所以需要`import "github.com/darjun/go-daily-lib/testing/roman"`。这种方式在测试包有循环依赖的情况下非常有用，例如标准库中`net/http`依赖`net/url`，`url`的测试函数依赖`net/http`，如果把测试放在`net/url`包中，那么就会导致循环依赖`url_test(net/url)`->`net/http`->`net/url`。这时可以将`url_test`放在一个独立的包中。

### 主测试函数

有一种特殊的测试函数，函数名为`TestMain()`，接受一个`*testing.M`类型的参数。这个函数一般用于在运行**所有测试**前执行一些初始化逻辑（如创建数据库链接），或**所有测试**都运行结束之后执行一些清理逻辑（释放数据库链接）。如果测试文件中定义了这个函数，则`go test`命令会直接运行这个函数，否者`go test`会创建一个默认的`TestMain()`函数。这个函数的默认行为就是运行文件中定义的测试。**我们自定义`TestMain()`函数时，也需要手动调用`m.Run()`方法运行测试函数，否则测试函数不会运行**。默认的`TestMain()`类似下面代码：

```golang
func TestMain(m *testing.M) {
  os.Exit(m.Run())
}
```

下面自定义一个`TestMain()`函数，打印`go test`支持的选项：

```golang
func TestMain(m *testing.M) {
  flag.Parse()
  flag.VisitAll(func(f *flag.Flag) {
    fmt.Printf("name:%s usage:%s value:%v\n", f.Name, f.Usage, f.Value)
  })
  os.Exit(m.Run())
}
```

运行：

```golang
$ go test -v
name:test.bench usage:run only benchmarks matching `regexp` value:
name:test.benchmem usage:print memory allocations for benchmarks value:false
name:test.benchtime usage:run each benchmark for duration `d` value:1s
name:test.blockprofile usage:write a goroutine blocking profile to `file` value:
name:test.blockprofilerate usage:set blocking profile `rate` (see runtime.SetBlockProfileRate) value:1
name:test.count usage:run tests and benchmarks `n` times value:1
name:test.coverprofile usage:write a coverage profile to `file` value:
name:test.cpu usage:comma-separated `list` of cpu counts to run each test with value:
name:test.cpuprofile usage:write a cpu profile to `file` value:
name:test.failfast usage:do not start new tests after the first test failure value:false
name:test.list usage:list tests, examples, and benchmarks matching `regexp` then exit value:
name:test.memprofile usage:write an allocation profile to `file` value:
name:test.memprofilerate usage:set memory allocation profiling `rate` (see runtime.MemProfileRate) value:0
name:test.mutexprofile usage:write a mutex contention profile to the named file after execution value:
name:test.mutexprofilefraction usage:if >= 0, calls runtime.SetMutexProfileFraction() value:1
name:test.outputdir usage:write profiles to `dir` value:
name:test.paniconexit0 usage:panic on call to os.Exit(0) value:true
name:test.parallel usage:run at most `n` tests in parallel value:8
name:test.run usage:run only tests and examples matching `regexp` value:
name:test.short usage:run smaller test suite to save time value:false
name:test.testlogfile usage:write test action log to `file` (for use only by cmd/go) value:
name:test.timeout usage:panic test binary after duration `d` (default 0, timeout disabled) value:10m0s
name:test.trace usage:write an execution trace to `file` value:
name:test.v usage:verbose: print additional output value:tru
```

这些选项也可以通过`go help testflag`查看。

### 其他

**另一个函数`FromRoman()`我没有写任何测试，就交给大家了😀**

## 性能测试

性能测试是为了对函数的运行性能进行评测。性能测试也必须在`_test.go`文件中编写，且函数名必须是`BenchmarkXxxx`开头。性能测试函数接受一个`*testing.B`的参数。下面我们编写 3 个计算第 n 个斐波那契数的函数。

第一种方式：递归

```golang
func Fib1(n int) int {
  if n <= 1 {
    return n
  }
  
  return Fib1(n-1) + Fib1(n-2)
}
```

第二种方式：备忘录

```golang
func fibHelper(n int, m map[int]int) int {
  if n <= 1 {
    return n
  }

  if v, ok := m[n]; ok {
    return v
  }
  
  v := fibHelper(n-2, m) + fibHelper(n-1, m)
  m[n] = v
  return v
}

func Fib2(n int) int {
  m := make(map[int]int)
  return fibHelper(n, m)
}
```

第三种方式：迭代

```golang
func Fib3(n int) int {
  if n <= 1 {
    return n
  }
  
  f1, f2 := 0, 1
  for i := 2; i <= n; i++ {
    f1, f2 = f2, f1+f2
  }
  
  return f2
}
```

下面我们来测试这 3 个函数的执行效率：

```golang
// fib_test.go
func BenchmarkFib1(b *testing.B) {
  for i := 0; i < b.N; i++ {
    Fib1(20)
  }
}

func BenchmarkFib2(b *testing.B) {
  for i := 0; i < b.N; i++ {
    Fib2(20)
  }
}

func BenchmarkFib3(b *testing.B) {
  for i := 0; i < b.N; i++ {
    Fib3(20)
  }
}
```

需要特别注意的是`N`，`go test`会一直调整这个数值，直到测试时间能得出可靠的性能数据为止。运行：

```golang
$ go test -bench=.
goos: windows
goarch: amd64
pkg: github.com/darjun/go-daily-lib/testing/fib
cpu: Intel(R) Core(TM) i7-7700 CPU @ 3.60GHz
BenchmarkFib1-8            31110             39144 ns/op
BenchmarkFib2-8           582637              3127 ns/op
BenchmarkFib3-8         191600582            5.588 ns/op
PASS
ok      github.com/darjun/go-daily-lib/testing/fib      5.225s
```

性能测试默认不会执行，需要通过`-bench=.`指定运行。`-bench`选项的值是一个简单的模式，`.`表示匹配所有的，`Fib`表示运行名字中有`Fib`的。

上面的测试结果表示`Fib1`在指定时间内执行了 31110 次，平均每次 39144 ns，`Fib2`在指定时间内运行了 582637 次，平均每次耗时 3127 ns，`Fib3`在指定时间内运行了 191600582 次，平均每次耗时 5.588 ns。

### 其他选项

有一些选项可以控制性能测试的执行。

`-benchtime`：设置每个测试的运行时间。

```golang
$ go test -bench=. -benchtime=30s
```

运行了更长的时间：

```golang
$ go test -bench=. -benchtime=30s
goos: windows
goarch: amd64
pkg: github.com/darjun/go-daily-lib/testing/fib
cpu: Intel(R) Core(TM) i7-7700 CPU @ 3.60GHz
BenchmarkFib1-8           956464             38756 ns/op
BenchmarkFib2-8         17862495              2306 ns/op
BenchmarkFib3-8       1000000000             5.591 ns/op
PASS
ok      github.com/darjun/go-daily-lib/testing/fib      113.498s
```

`-benchmem`：输出性能测试函数的内存分配情况。

`-memprofile file`：将内存分配数据写入文件。

`-cpuprofile file`：将 CPU 采样数据写入文件，方便使用`go tool pprof`工具分析，详见我的另一篇文章[《你不知道的 Go 之 pprof》](https://darjun.github.io/2021/05/18/youdontknowgo/string/)

运行：

```golang
$ go test -bench=. -benchtime=10s -cpuprofile=./cpu.prof -memprofile=./mem.prof
goos: windows
goarch: amd64
pkg: github.com/darjun/fib
BenchmarkFib1-16          356006             33423 ns/op
BenchmarkFib2-16         8958194              1340 ns/op
BenchmarkFib3-16        1000000000               6.60 ns/op
PASS
ok      github.com/darjun/fib   33.321s
```

同时生成了 CPU 采样数据和内存分配数据，通过`go tool pprof`分析：

```golang
$ go tool pprof ./cpu.prof
Type: cpu
Time: Aug 4, 2021 at 10:21am (CST)
Duration: 32.48s, Total samples = 36.64s (112.81%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top10
Showing nodes accounting for 29640ms, 80.90% of 36640ms total
Dropped 153 nodes (cum <= 183.20ms)
Showing top 10 nodes out of 74
      flat  flat%   sum%        cum   cum%
   11610ms 31.69% 31.69%    11620ms 31.71%  github.com/darjun/fib.Fib1
    6490ms 17.71% 49.40%     6680ms 18.23%  github.com/darjun/fib.Fib3
    2550ms  6.96% 56.36%     8740ms 23.85%  runtime.mapassign_fast64
    2050ms  5.59% 61.95%     2060ms  5.62%  runtime.stdcall2
    1620ms  4.42% 66.38%     2140ms  5.84%  runtime.mapaccess2_fast64
    1480ms  4.04% 70.41%    12350ms 33.71%  github.com/darjun/fib.fibHelper
    1480ms  4.04% 74.45%     2960ms  8.08%  runtime.evacuate_fast64
    1050ms  2.87% 77.32%     1050ms  2.87%  runtime.memhash64
     760ms  2.07% 79.39%      760ms  2.07%  runtime.stdcall7
     550ms  1.50% 80.90%     7230ms 19.73%  github.com/darjun/fib.BenchmarkFib3
(pprof)
```

内存：

```golang
$ go tool pprof ./mem.prof
Type: alloc_space
Time: Aug 4, 2021 at 10:30am (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top10
Showing nodes accounting for 8.69GB, 100% of 8.69GB total
Dropped 12 nodes (cum <= 0.04GB)
      flat  flat%   sum%        cum   cum%
    8.69GB   100%   100%     8.69GB   100%  github.com/darjun/fib.fibHelper
         0     0%   100%     8.69GB   100%  github.com/darjun/fib.BenchmarkFib2
         0     0%   100%     8.69GB   100%  github.com/darjun/fib.Fib2 (inline)
         0     0%   100%     8.69GB   100%  testing.(*B).launch
         0     0%   100%     8.69GB   100%  testing.(*B).runN
(pprof)
```

## 示例测试

示例测试用于演示模块或函数的使用。同样地，示例测试也在文件`_test.go`中编写，并且示例测试函数名必须是`ExampleXxx`的形式。在`Example*`函数中编写代码，然后在注释中编写期望的输出，`go test`会运行该函数，然后将实际输出与期望的做比较。下面摘取自 Go 源码`net/url/example_test.go`文件中的代码演示了`url.Values`的用法：

```golang
func ExampleValuesGet() {
  v := url.Values{}
  v.Set("name", "Ava")
  v.Add("friend", "Jess")
  v.Add("friend", "Sarah")
  v.Add("friend", "Zoe")
  fmt.Println(v.Get("name"))
  fmt.Println(v.Get("friend"))
  fmt.Println(v["friend"])
  // Output:
  // Ava
  // Jess
  // [Jess Sarah Zoe]
}
```

注释中`Output:`后是期望的输出结果，`go test`会运行这些函数并与期望的结果做比较，比较会忽略空格。

有时候我们输出的顺序是不确定的，这时就需要使用`Unordered Output`。我们知道`url.Values`底层类型为`map[string][]string`，所以可以遍历输出所有的键值，但是输出顺序不确定：

```golang
func ExampleValuesAll() {
  v := url.Values{}
  v.Set("name", "Ava")
  v.Add("friend", "Jess")
  v.Add("friend", "Sarah")
  v.Add("friend", "Zoe")
  for key, values := range v {
    fmt.Println(key, values)
  }
  // Unordered Output:
  // name [Ava]
  // friend [Jess Sarah Zoe]
}
```

运行：

```golang
$ go test -v
$ go test -v
=== RUN   ExampleValuesGet
--- PASS: ExampleValuesGet (0.00s)
=== RUN   ExampleValuesAll
--- PASS: ExampleValuesAll (0.00s)
PASS
ok      github.com/darjun/url   0.172s
```

没有注释，或注释中无`Output/Unordered Output`的函数会被忽略。

## 总结

本文介绍了 Go 中的 3 种测试：单元测试，性能测试和示例测试。为了让程序更可靠，让以后的重构更安全、更放心，单元测试必不可少。排查程序中的性能问题，性能测试能派上大用场。示例测试主要是为了演示如何使用某个功能。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. testing 官方文档: [https://golang.google.cn/pkg/testing/](https://golang.google.cn/pkg/testing/)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)