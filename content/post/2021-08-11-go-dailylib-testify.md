---
layout:    post
title:    "Go 每日一库之 testify"
subtitle: "每天学习一个 Go 库"
date:     2021-08-11T07:16:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/08/11/godailylib/testify"
categories: [
  "Go"
]
---

## 简介

[`testify`](https://github.com/stretchr/testify)可以说是最流行的（从 GitHub star 数来看）Go 语言测试库了。`testify`提供了很多方便的函数帮助我们做`assert`和错误信息输出。使用标准库`testing`，我们需要自己编写各种条件判断，根据判断结果决定输出对应的信息。

`testify`核心有三部分内容：

* `assert`：断言；
* `mock`：测试替身；
* `suite`：测试套件。

## 准备工作

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir -p testify && cd testify
$ go mod init github.com/darjun/go-daily-lib/testify
```

安装`testify`库：

```cmd
$ go get -u github.com/stretchr/testify
```

## `assert`

`assert`子库提供了便捷的**断言**函数，可以大大简化测试代码的编写。总的来说，它将之前需要`判断 + 信息输出的模式`：

```golang
if got != expected {
  t.Errorf("Xxx failed expect:%d got:%d", got, expected)
}
```

简化为一行**断言**代码：

```golang
assert.Equal(t, got, expected, "they should be equal")
```

结构更清晰，更可读。熟悉其他语言测试框架的开发者对`assert`的相关用法应该不会陌生。此外，`assert`中的函数会自动生成比较清晰的错误描述信息：

```golang
func TestEqual(t *testing.T) {
  var a = 100
  var b = 200
  assert.Equal(t, a, b, "")
}
```

使用`testify`编写测试代码与`testing`一样，测试文件为`_test.go`，测试函数为`TestXxx`。使用`go test`命令运行测试：

```golang
$ go test
--- FAIL: TestEqual (0.00s)
    assert_test.go:12:
                Error Trace:
                Error:          Not equal:
                                expected: 100
                                actual  : 200
                Test:           TestEqual
FAIL
exit status 1
FAIL    github.com/darjun/go-daily-lib/testify/assert   0.107s
```

我们看到信息更易读。

`testify`提供的`assert`类函数众多，每种函数都有两个版本，一个版本是函数名不带`f`的，一个版本是带`f`的，区别就在于带`f`的函数，我们需要指定至少两个参数，一个格式化字符串`format`，若干个参数`args`：

```golang
func Equal(t TestingT, expected, actual interface{}, msgAndArgs ...interface{})
func Equalf(t TestingT, expected, actual interface{}, msg string, args ...interface{})
```

实际上，在`Equalf()`函数内部调用了`Equal()`：

```golang
func Equalf(t TestingT, expected interface{}, actual interface{}, msg string, args ...interface{}) bool {
  if h, ok := t.(tHelper); ok {
    h.Helper()
  }
  return Equal(t, expected, actual, append([]interface{}{msg}, args...)...)
}
```

所以，我们只需要关注不带`f`的版本即可。

### `Contains`

函数类型：

```golang
func Contains(t TestingT, s, contains interface{}, msgAndArgs ...interface{}) bool
```

`Contains`断言`s`包含`contains`。其中`s`可以是字符串，数组/切片，map。相应地，`contains`为子串，数组/切片元素，map 的键。

### `DirExists`

函数类型：

```golang
func DirExists(t TestingT, path string, msgAndArgs ...interface{}) bool
```

`DirExists`断言路径`path`是一个目录，如果`path`不存在或者是一个文件，断言失败。

### `ElementsMatch`

函数类型：

```golang
func ElementsMatch(t TestingT, listA, listB interface{}, msgAndArgs ...interface{}) bool
```

`ElementsMatch`断言`listA`和`listB`包含相同的元素，忽略元素出现的顺序。`listA/listB`必须是数组或切片。如果有重复元素，重复元素出现的次数也必须相等。

### `Empty`

函数类型：

```golang
func Empty(t TestingT, object interface{}, msgAndArgs ...interface{}) bool
```

`Empty`断言`object`是空，根据`object`中存储的实际类型，空的含义不同：

* 指针：`nil`；
* 整数：0；
* 浮点数：0.0；
* 字符串：空串`""`；
* 布尔：false；
* 切片或 channel：长度为 0。

### `EqualError`

函数类型：

```golang
func EqualError(t TestingT, theError error, errString string, msgAndArgs ...interface{}) bool
```

`EqualError`断言`theError.Error()`的返回值与`errString`相等。

### `EqualValues`

函数类型：

```golang
func EqualValues(t TestingT, expected, actual interface{}, msgAndArgs ...interface{}) bool
```

`EqualValues`断言`expected`与`actual`相等，或者可以转换为相同的类型，并且相等。这个条件比`Equal`更宽，`Equal()`返回`true`则`EqualValues()`肯定也返回`true`，反之则不然。实现的核心是下面两个函数，使用了`reflect.DeapEqual()`：

```golang
func ObjectsAreEqual(expected, actual interface{}) bool {
  if expected == nil || actual == nil {
    return expected == actual
  }

  exp, ok := expected.([]byte)
  if !ok {
    return reflect.DeepEqual(expected, actual)
  }

  act, ok := actual.([]byte)
  if !ok {
    return false
  }
  if exp == nil || act == nil {
    return exp == nil && act == nil
  }
  return bytes.Equal(exp, act)
}

func ObjectsAreEqualValues(expected, actual interface{}) bool {
    // 如果`ObjectsAreEqual`返回 true，直接返回
  if ObjectsAreEqual(expected, actual) {
    return true
  }

  actualType := reflect.TypeOf(actual)
  if actualType == nil {
    return false
  }
  expectedValue := reflect.ValueOf(expected)
  if expectedValue.IsValid() && expectedValue.Type().ConvertibleTo(actualType) {
    // 尝试类型转换
    return reflect.DeepEqual(expectedValue.Convert(actualType).Interface(), actual)
  }

  return false
}
```

例如我基于`int`定义了一个新类型`MyInt`，它们的值都是 100，`Equal()`调用将返回 false，`EqualValues()`会返回 true：

```golang
type MyInt int

func TestEqual(t *testing.T) {
  var a = 100
  var b MyInt = 100
  assert.Equal(t, a, b, "")
  assert.EqualValues(t, a, b, "")
}
```

### `Error`

函数类型：

```golang
func Error(t TestingT, err error, msgAndArgs ...interface{}) bool
```

`Error`断言`err`不为`nil`。

### `ErrorAs`

函数类型：

```golang
func ErrorAs(t TestingT, err error, target interface{}, msgAndArgs ...interface{}) bool
```

`ErrorAs`断言`err`表示的 error 链中至少有一个和`target`匹配。这个函数是对标准库中`errors.As`的包装。

### `ErrorIs`

函数类型：

```golang
func ErrorIs(t TestingT, err, target error, msgAndArgs ...interface{}) bool
```

`ErrorIs`断言`err`的 error 链中有`target`。

### 逆断言

上面的断言都是它们的逆断言，例如`NotEqual/NotEqualValues`等。

### Assertions 对象

观察到上面的断言都是以`TestingT`为第一个参数，需要大量使用时比较麻烦。`testify`提供了一种方便的方式。先以`*testing.T`创建一个`*Assertions`对象，`Assertions`定义了前面所有的断言方法，只是不需要再传入`TestingT`参数了。

```golang
func TestEqual(t *testing.T) {
  assertions := assert.New(t)
  assertion.Equal(a, b, "")
  // ...
}
```

顺带提一句`TestingT`是一个接口，对`*testing.T`做了一个简单的包装：

```golang
type TestingT interface{
  Errorf(format string, args ...interface{})
}
```

## require

`require`提供了和`assert`同样的接口，但是遇到错误时，`require`直接终止测试，而`assert`返回`false`。

## mock

`testify`提供了对 Mock 的简单支持。Mock 简单来说就是构造一个**仿对象**，仿对象提供和原对象一样的接口，在测试中用仿对象来替换原对象。这样我们可以在原对象很难构造，特别是涉及外部资源（数据库，访问网络等）。例如，我们现在要编写一个从一个站点拉取用户列表信息的程序，拉取完成之后程序显示和分析。如果每次都去访问网络会带来极大的不确定性，甚至每次返回不同的列表，这就给测试带来了极大的困难。我们可以使用 Mock 技术。

```golang
package main

import (
  "encoding/json"
  "fmt"
  "io/ioutil"
  "net/http"
)

type User struct {
  Name string
  Age  int
}

type ICrawler interface {
  GetUserList() ([]*User, error)
}

type MyCrawler struct {
  url string
}

func (c *MyCrawler) GetUserList() ([]*User, error) {
  resp, err := http.Get(c.url)
  if err != nil {
    return nil, err
  }

  defer resp.Body.Close()
  data, err := ioutil.ReadAll(resp.Body)
  if err != nil {
    return nil, err
  }

  var userList []*User
  err = json.Unmarshal(data, &userList)
  if err != nil {
    return nil, err
  }

  return userList, nil
}

func GetAndPrintUsers(crawler ICrawler) {
  users, err := crawler.GetUserList()
  if err != nil {
    return
  }

  for _, u := range users {
    fmt.Println(u)
  }
}
```

`Crawler.GetUserList()`方法完成爬取和解析操作，返回用户列表。为了方便 Mock，`GetAndPrintUsers()`函数接受一个`ICrawler`接口。现在来定义我们的 Mock 对象，实现`ICrawler`接口：

```golang
package main

import (
  "github.com/stretchr/testify/mock"
  "testing"
)

type MockCrawler struct {
  mock.Mock
}

func (m *MockCrawler) GetUserList() ([]*User, error) {
  args := m.Called()
  return args.Get(0).([]*User), args.Error(1)
}

var (
  MockUsers []*User
)

func init() {
  MockUsers = append(MockUsers, &User{"dj", 18})
  MockUsers = append(MockUsers, &User{"zhangsan", 20})
}

func TestGetUserList(t *testing.T) {
  crawler := new(MockCrawler)
  crawler.On("GetUserList").Return(MockUsers, nil)

  GetAndPrintUsers(crawler)

  crawler.AssertExpectations(t)
}
```

实现`GetUserList()`方法时，需要调用`Mock.Called()`方法，传入参数（示例中无参数）。`Called()`会返回一个`mock.Arguments`对象，该对象中保存着返回的值。它提供了对基本类型和`error`的获取方法`Int()/String()/Bool()/Error()`，和通用的获取方法`Get()`，通用方法返回`interface{}`，需要类型断言为具体类型，它们都接受一个表示索引的参数。

`crawler.On("GetUserList").Return(MockUsers, nil)`是 Mock 发挥魔法的地方，这里指示调用`GetUserList()`方法的返回值分别为`MockUsers`和`nil`，返回值在上面的`GetUserList()`方法中被`Arguments.Get(0)`和`Arguments.Error(1)`获取。

最后`crawler.AssertExpectations(t)`对 Mock 对象做断言。

运行：

```golang
$ go test
&{dj 18}
&{zhangsan 20}
PASS
ok      github.com/darjun/testify       0.258s
```

`GetAndPrintUsers()`函数功能正常执行，并且我们通过 Mock 提供的用户列表也能正确获取。

使用 Mock，我们可以精确断言某方法以特定参数的调用次数，`Times(n int)`，它有两个便捷函数`Once()/Twice()`。下面我们要求函数`Hello(n int)`要以参数 1 调用 1次，参数 2 调用两次，参数 3 调用 3 次：

```golang
type IExample interface {
  Hello(n int) int
}

type Example struct {
}

func (e *Example) Hello(n int) int {
  fmt.Printf("Hello with %d\n", n)
  return n
}

func ExampleFunc(e IExample) {
  for n := 1; n <= 3; n++ {
    for i := 0; i <= n; i++ {
      e.Hello(n)
    }
  }
}
```

编写 Mock 对象：

```golang
type MockExample struct {
  mock.Mock
}

func (e *MockExample) Hello(n int) int {
  args := e.Mock.Called(n)
  return args.Int(0)
}

func TestExample(t *testing.T) {
  e := new(MockExample)

  e.On("Hello", 1).Return(1).Times(1)
  e.On("Hello", 2).Return(2).Times(2)
  e.On("Hello", 3).Return(3).Times(3)

  ExampleFunc(e)

  e.AssertExpectations(t)
}
```

运行：

```golang
$ go test
--- FAIL: TestExample (0.00s)
panic:
assert: mock: The method has been called over 1 times.
        Either do one more Mock.On("Hello").Return(...), or remove extra call.
        This call was unexpected:
                Hello(int)
                0: 1
        at: [equal_test.go:13 main.go:22] [recovered]
```

原来`ExampleFunc()`函数中`<=`应该是`<`导致多调用了一次，修改过来继续运行：

```golang
$ go test
PASS
ok      github.com/darjun/testify       0.236s
```

我们还可以设置以指定参数调用会导致 panic，测试程序的健壮性：

```golang
e.On("Hello", 100).Panic("out of range")
```

## suite

`testify`提供了测试套件的功能（`TestSuite`），`testify`测试套件只是一个结构体，内嵌一个匿名的`suite.Suite`结构。测试套件中可以包含多个测试，它们可以共享状态，还可以定义钩子方法执行初始化和清理操作。钩子都是通过接口来定义的，实现了这些接口的测试套件结构在运行到指定节点时会调用对应的方法。

```golang
type SetupAllSuite interface {
  SetupSuite()
}
```

如果定义了`SetupSuite()`方法（即实现了`SetupAllSuite`接口），在套件中所有测试开始运行前调用这个方法。对应的是`TearDownAllSuite`：

```golang
type TearDownAllSuite interface {
  TearDownSuite()
}
```

如果定义了`TearDonwSuite()`方法（即实现了`TearDownSuite`接口），在套件中所有测试运行完成后调用这个方法。

```golang
type SetupTestSuite interface {
  SetupTest()
}
```

如果定义了`SetupTest()`方法（即实现了`SetupTestSuite`接口），在套件中每个测试执行前都会调用这个方法。对应的是`TearDownTestSuite`：

```golang
type TearDownTestSuite interface {
  TearDownTest()
}
```

如果定义了`TearDownTest()`方法（即实现了`TearDownTest`接口），在套件中每个测试执行后都会调用这个方法。

还有一对接口`BeforeTest/AfterTest`，它们分别在每个测试运行前/后调用，接受套件名和测试名作为参数。

我们来编写一个测试套件结构作为演示：

```golang
type MyTestSuit struct {
  suite.Suite
  testCount uint32
}

func (s *MyTestSuit) SetupSuite() {
  fmt.Println("SetupSuite")
}

func (s *MyTestSuit) TearDownSuite() {
  fmt.Println("TearDownSuite")
}

func (s *MyTestSuit) SetupTest() {
  fmt.Printf("SetupTest test count:%d\n", s.testCount)
}

func (s *MyTestSuit) TearDownTest() {
  s.testCount++
  fmt.Printf("TearDownTest test count:%d\n", s.testCount)
}

func (s *MyTestSuit) BeforeTest(suiteName, testName string) {
  fmt.Printf("BeforeTest suite:%s test:%s\n", suiteName, testName)
}

func (s *MyTestSuit) AfterTest(suiteName, testName string) {
  fmt.Printf("AfterTest suite:%s test:%s\n", suiteName, testName)
}

func (s *MyTestSuit) TestExample() {
  fmt.Println("TestExample")
}
```

这里只是简单在各个钩子函数中打印信息，统计执行完成的测试数量。由于要借助`go test`运行，所以需要编写一个`TestXxx`函数，在该函数中调用`suite.Run()`运行测试套件：

```golang
func TestExample(t *testing.T) {
  suite.Run(t, new(MyTestSuit))
}
```

`suite.Run(t, new(MyTestSuit))`会将运行`MyTestSuit`中所有名为`TestXxx`的方法。运行：

```golang
$ go test
SetupSuite
SetupTest test count:0
BeforeTest suite:MyTestSuit test:TestExample
TestExample
AfterTest suite:MyTestSuit test:TestExample
TearDownTest test count:1
TearDownSuite
PASS
ok      github.com/darjun/testify       0.375s
```

### 测试 HTTP 服务器

Go 标准库提供了一个`httptest`用于测试 HTTP 服务器。现在编写一个简单的 HTTP 服务器：

```golang
func index(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, "Hello World")
}

func greeting(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "welcome, %s", r.URL.Query().Get("name"))
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", index)
  mux.HandleFunc("/greeting", greeting)

  server := &http.Server{
    Addr:    ":8080",
    Handler: mux,
  }

  if err := server.ListenAndServe(); err != nil {
    log.Fatal(err)
  }
}
```

很简单。`httptest`提供了一个`ResponseRecorder`类型，它实现了`http.ResponseWriter`接口，但是它只是记录写入的状态码和响应内容，不会发送响应给客户端。这样我们可以将该类型的对象传给处理器函数。然后构造服务器，传入该对象来驱动请求处理流程，最后测试该对象中记录的信息是否正确：

```golang
func TestIndex(t *testing.T) {
  recorder := httptest.NewRecorder()
  request, _ := http.NewRequest("GET", "/", nil)
  mux := http.NewServeMux()
  mux.HandleFunc("/", index)
  mux.HandleFunc("/greeting", greeting)

  mux.ServeHTTP(recorder, request)

  assert.Equal(t, recorder.Code, 200, "get index error")
  assert.Contains(t, recorder.Body.String(), "Hello World", "body error")
}

func TestGreeting(t *testing.T) {
  recorder := httptest.NewRecorder()
  request, _ := http.NewRequest("GET", "/greeting", nil)
  request.URL.RawQuery = "name=dj"
  mux := http.NewServeMux()
  mux.HandleFunc("/", index)
  mux.HandleFunc("/greeting", greeting)

  mux.ServeHTTP(recorder, request)

  assert.Equal(t, recorder.Code, 200, "greeting error")
  assert.Contains(t, recorder.Body.String(), "welcome, dj", "body error")
}
```

运行：

```golang
$ go test
PASS
ok      github.com/darjun/go-daily-lib/testify/httptest 0.093s
```

很简单，没有问题。

但是我们发现一个问题，上面的很多代码有重复，`recorder/mux`等对象的创建，处理器函数的注册。使用`suite`我们可以集中创建，省略这些重复的代码：

```golang
type MySuite struct {
  suite.Suite
  recorder *httptest.ResponseRecorder
  mux      *http.ServeMux
}

func (s *MySuite) SetupSuite() {
  s.recorder = httptest.NewRecorder()
  s.mux = http.NewServeMux()
  s.mux.HandleFunc("/", index)
  s.mux.HandleFunc("/greeting", greeting)
}

func (s *MySuite) TestIndex() {
  request, _ := http.NewRequest("GET", "/", nil)
  s.mux.ServeHTTP(s.recorder, request)

  s.Assert().Equal(s.recorder.Code, 200, "get index error")
  s.Assert().Contains(s.recorder.Body.String(), "Hello World", "body error")
}

func (s *MySuite) TestGreeting() {
  request, _ := http.NewRequest("GET", "/greeting", nil)
  request.URL.RawQuery = "name=dj"

  s.mux.ServeHTTP(s.recorder, request)

  s.Assert().Equal(s.recorder.Code, 200, "greeting error")
  s.Assert().Contains(s.recorder.Body.String(), "welcome, dj", "body error")
}
```

最后编写一个`TestXxx`驱动测试：

```golang
func TestHTTP(t *testing.T) {
  suite.Run(t, new(MySuite))
}
```

## 总结

`testify`扩展了`testing`标准库，断言库`assert`，测试替身`mock`和测试套件`suite`，让我们编写测试代码更容易！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. testify GitHub：[github.com/stretchr/testify](https://github.com/stretchr/testify)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)