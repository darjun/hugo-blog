---
layout:    post
title:    "Go 每日一库之 gorilla/securecookie"
subtitle: "每天学习一个 Go 库"
date:     2021-07-23T00:13:32
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/07/23/godailylib/gorilla/securecookie"
categories: [
  "Go"
]
---

## 简介

cookie 是用于在 Web 客户端（一般是浏览器）和服务器之间传输少量数据的一种机制。由服务器生成，发送到客户端保存，客户端后续的每次请求都会将 cookie 带上。cookie 现在已经被多多少少地滥用了。很多公司使用 cookie 来收集用户信息、投放广告等。

cookie 有两大缺点：
* 每次请求都需要传输，故不能用来存放大量数据；
* 安全性较低，通过浏览器工具，很容易看到由网站服务器设置的 cookie。

[`gorilla/securecookie`](https://github.com/gorilla/securecookie)提供了一种安全的 cookie，通过在服务端给 cookie 加密，让其内容不可读，也不可伪造。当然，敏感信息还是强烈建议不要放在 cookie 中。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir -p gorilla/securecookie && cd gorilla/securecookie
$ go mod init github.com/darjun/go-daily-lib/gorilla/securecookie
```

安装`gorilla/securecookie`库：

```cmd
$ go get github.com/gorilla/securecookie
```

```golang
package main

import (
  "fmt"
  "github.com/gorilla/mux"
  "github.com/gorilla/securecookie"
  "log"
  "net/http"
)

type User struct {
  Name string
  Age int
}

var (
  hashKey = securecookie.GenerateRandomKey(16)
  blockKey = securecookie.GenerateRandomKey(16)
  s = securecookie.New(hashKey, blockKey)
)

func SetCookieHandler(w http.ResponseWriter, r *http.Request) {
  u := &User {
    Name: "dj",
    Age: 18,
  }
  if encoded, err := s.Encode("user", u); err == nil {
    cookie := &http.Cookie{
      Name: "user",
      Value: encoded,
      Path: "/",
      Secure: true,
      HttpOnly: true,
    }
    http.SetCookie(w, cookie)
  }
  fmt.Fprintln(w, "Hello World")
}

func ReadCookieHandler(w http.ResponseWriter, r *http.Request) {
  if cookie, err := r.Cookie("user"); err == nil {
    u := &User{}
    if err = s.Decode("user", cookie.Value, u); err == nil {
      fmt.Fprintf(w, "name:%s age:%d", u.Name, u.Age)
    }
  }
}

func main() {
  r := mux.NewRouter()
  r.HandleFunc("/set_cookie", SetCookieHandler)
  r.HandleFunc("/read_cookie", ReadCookieHandler)
  http.Handle("/", r)
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

首先需要创建一个`SecureCookie`对象：

```golang
var s = securecookie.New(hashKey, blockKey)
```

其中`hashKey`是必填的，它用来验证 cookie 是否是伪造的，底层使用 HMAC（Hash-based message authentication code）算法。推荐`hashKey`使用 32/64 字节的 Key。

`blockKey`是可选的，它用来加密 cookie，如不需要加密，可以传`nil`。如果设置了，它的长度必须与对应的加密算法的块大小（block size）一致。例如对于 AES 系列算法，AES-128/AES-192/AES-256 对应的块大小分别为 16/24/32 字节。

为了方便也可以使用`GenerateRandomKey()`函数生成一个安全性足够强的随机 key。每次调用该函数都会返回不同的 key。上面代码就是通过这种方式创建 key 的。

调用`s.Encode("user", u)`将对象`u`编码成字符串，内部实际上使用了标准库`encoding/gob`。所以`gob`支持的类型都可以编码。

调用`s.Decode("user", cookie.Value, u)`将 cookie 值解码到对应的`u`对象中。

运行：

```golang
$ go run main.go

```

首先使用浏览器访问`localhost:8080/set_cookie`，这时可以在 Chrome 开发者工具的 Application 页签中看到 cookie 内容：

![](/img/in-post/godailylib/securecookie1.png#center)

访问`localhost:8080/read_cookie`，页面显示`name: dj age: 18`。

## 使用 JSON

`securecookie`默认使用`encoding/gob`编码 cookie 值，我们也可以改用`encoding/json`。`securecookie`将编解码器封装成一个`Serializer`接口：

```golang
type Serializer interface {
  Serialize(src interface{}) ([]byte, error)
  Deserialize(src []byte, dst interface{}) error
}
```

`securecookie`提供了`GobEncoder`和`JSONEncoder`的实现：

```golang
func (e GobEncoder) Serialize(src interface{}) ([]byte, error) {
  buf := new(bytes.Buffer)
  enc := gob.NewEncoder(buf)
  if err := enc.Encode(src); err != nil {
    return nil, cookieError{cause: err, typ: usageError}
  }
  return buf.Bytes(), nil
}

func (e GobEncoder) Deserialize(src []byte, dst interface{}) error {
  dec := gob.NewDecoder(bytes.NewBuffer(src))
  if err := dec.Decode(dst); err != nil {
    return cookieError{cause: err, typ: decodeError}
  }
  return nil
}

func (e JSONEncoder) Serialize(src interface{}) ([]byte, error) {
  buf := new(bytes.Buffer)
  enc := json.NewEncoder(buf)
  if err := enc.Encode(src); err != nil {
    return nil, cookieError{cause: err, typ: usageError}
  }
  return buf.Bytes(), nil
}

func (e JSONEncoder) Deserialize(src []byte, dst interface{}) error {
  dec := json.NewDecoder(bytes.NewReader(src))
  if err := dec.Decode(dst); err != nil {
    return cookieError{cause: err, typ: decodeError}
  }
  return nil
}
```

我们可以调用`securecookie.SetSerializer(JSONEncoder{})`设置使用 JSON 编码：

```golang
var (
  hashKey = securecookie.GenerateRandomKey(16)
  blockKey = securecookie.GenerateRandomKey(16)
  s = securecookie.New(hashKey, blockKey)
)

func init() {
  s.SetSerializer(securecookie.JSONEncoder{})
}
```

## 自定义编解码

我们可以定义一个类型实现`Serializer`接口，那么该类型的对象可以用作`securecookie`的编解码器。我们实现一个简单的 XML 编解码器：

```golang
package main

type XMLEncoder struct{}

func (x XMLEncoder) Serialize(src interface{}) ([]byte, error) {
  buf := &bytes.Buffer{}
  encoder := xml.NewEncoder(buf)
  if err := encoder.Encode(buf); err != nil {
    return nil, err
  }
  return buf.Bytes(), nil
}

func (x XMLEncoder) Deserialize(src []byte, dst interface{}) error {
  dec := xml.NewDecoder(bytes.NewBuffer(src))
  if err := dec.Decode(dst); err != nil {
    return err
  }
  return nil
}

func init() {
  s.SetSerializer(XMLEncoder{})
}
```

由于`securecookie.cookieError`未导出，`XMLEncoder`与`GobEncoder/JSONEncoder`返回的错误有些不一致，不过不影响使用。

## Hash/Block 函数

`securecookie`默认使用`sha256.New`作为 Hash 函数（用于 HMAC 算法），使用`aes.NewCipher`作为 Block 函数（用于加解密）：

```golang
// securecookie.go
func New(hashKey, blockKey []byte) *SecureCookie {
  s := &SecureCookie{
    hashKey:   hashKey,
    blockKey:  blockKey,
    // 这里设置 Hash 函数
    hashFunc:  sha256.New,
    maxAge:    86400 * 30,
    maxLength: 4096,
    sz:        GobEncoder{},
  }
  if hashKey == nil {
    s.err = errHashKeyNotSet
  }
  if blockKey != nil {
    // 这里设置 Block 函数
    s.BlockFunc(aes.NewCipher)
  }
  return s
}
```

可以通过`securecookie.HashFunc()`修改 Hash 函数，传入一个`func () hash.Hash`类型：

```golang
func (s *SecureCookie) HashFunc(f func() hash.Hash) *SecureCookie {
  s.hashFunc = f
  return s
}
```

通过`securecookie.BlockFunc()`修改 Block 函数，传入一个`f func([]byte) (cipher.Block, error)`：

```golang
func (s *SecureCookie) BlockFunc(f func([]byte) (cipher.Block, error)) *SecureCookie {
  if s.blockKey == nil {
    s.err = errBlockKeyNotSet
  } else if block, err := f(s.blockKey); err == nil {
    s.block = block
  } else {
    s.err = cookieError{cause: err, typ: usageError}
  }
  return s
}
```

更换这两个函数更多的是处于安全性的考虑，例如选用更安全的`sha512`算法：

```golang
s.HashFunc(sha512.New512_256)
```

## 更换 Key

为了防止 cookie 泄露造成安全风险，有个常用的安全策略：定期更换 Key。更换 Key，让之前获得的 cookie 失效。对应`securecookie`库，就是更换`SecureCookie`对象：

```golang
var (
  prevCookie    unsafe.Pointer
  currentCookie unsafe.Pointer
)

func init() {
  prevCookie = unsafe.Pointer(securecookie.New(
    securecookie.GenerateRandomKey(64),
    securecookie.GenerateRandomKey(32),
  ))
  currentCookie = unsafe.Pointer(securecookie.New(
    securecookie.GenerateRandomKey(64),
    securecookie.GenerateRandomKey(32),
  ))
}
```

程序启动时，我们先生成两个`SecureCookie`对象，然后每隔一段时间就生成一个新的对象替换旧的。

由于每个请求都是在一个独立的 goroutine 中处理的（读），更换 key 也是在一个单独的 goroutine（写）。为了并发安全，我们必须增加同步措施。但是这种情况下使用锁又太重了，毕竟这里更新的频率很低。我这里将`securecookie.SecureCookie`对象存储为`unsafe.Pointer`类型，然后就可以使用`atomic`原子操作来同步读取和更新了：

```golang
func rotateKey() {
  newcookie := securecookie.New(
    securecookie.GenerateRandomKey(64),
    securecookie.GenerateRandomKey(32),
  )

  atomic.StorePointer(&prevCookie, currentCookie)
  atomic.StorePointer(&currentCookie, unsafe.Pointer(newcookie))
}
```

`rotateKey()`需要在一个新的 goroutine 中定期调用，我们在`main`函数中启动这个 goroutine

```golang
func main() {
  ctx, cancel := context.WithCancel(context.Background())
  defer cancel()
  go RotateKey(ctx)
}

func RotateKey(ctx context.Context) {
  ticker := time.NewTicker(30 * time.Second)
  defer ticker.Stop()

  for {
    select {
    case <-ctx.Done():
      break
    case <-ticker.C:
    }

    rotateKey()
  }
}
```

这里为了方便测试，我设置每隔 30s 就轮换一次。同时为了防止 goroutine 泄漏，我们传入了一个可取消的`Context`。还需要注意`time.NewTicker()`创建的`*time.Ticker`对象不使用时需要手动调用`Stop()`关闭，否则会造成资源泄漏。

使用两个`SecureCookie`对象之后，我们编解码可以调用`EncodeMulti/DecodeMulti`这组方法，它们可以接受多个`SecureCookie`对象：

```golang
func SetCookieHandler(w http.ResponseWriter, r *http.Request) {
  u := &User{
    Name: "dj",
    Age:  18,
  }

  if encoded, err := securecookie.EncodeMulti(
    "user", u,
    // 看这里 🐒
    (*securecookie.SecureCookie)(atomic.LoadPointer(&currentCookie)),
  ); err == nil {
    cookie := &http.Cookie{
      Name:     "user",
      Value:    encoded,
      Path:     "/",
      Secure:   true,
      HttpOnly: true,
    }
    http.SetCookie(w, cookie)
  }
  fmt.Fprintln(w, "Hello World")
}
```

使用`unsafe.Pointer`保存`SecureCookie`对象后，使用时需要类型转换。并且由于并发问题，需要使用`atomic.LoadPointer()`访问。

解码时调用`DecodeMulti`依次传入`currentCookie`和`prevCookie`，让`prevCookie`不会立刻失效：

```golang
func ReadCookieHandler(w http.ResponseWriter, r *http.Request) {
  if cookie, err := r.Cookie("user"); err == nil {
    u := &User{}
    if err = securecookie.DecodeMulti(
      "user", cookie.Value, u,
      // 看这里 🐒
      (*securecookie.SecureCookie)(atomic.LoadPointer(&currentCookie)),
      (*securecookie.SecureCookie)(atomic.LoadPointer(&prevCookie)),
    ); err == nil {
      fmt.Fprintf(w, "name:%s age:%d", u.Name, u.Age)
    } else {
      fmt.Fprintf(w, "read cookie error:%v", err)
    }
  }
}
```

运行程序：

```golang
$ go run main.go

```

先请求`localhost:8080/set_cookie`，然后请求`localhost:8080/read_cookie`读取 cookie。等待 1 分钟后，再次请求，发现之前的 cookie 失效了：

```golang
read cookie error:securecookie: the value is not valid (and 1 other error)
```

## 总结

`securecookie`为 cookie 添加了一层保护罩，让 cookie 不能轻易地被读取和伪造。还是需要强调一下：

敏感数据不要放在 cookie 中！敏感数据不要放在 cookie 中！敏感数据不要放在 cookie 中！敏感数据不要放在 cookie 中！

重要的事情说 4 遍。在使用 cookie 存放数据时需要仔细权衡。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gorilla/securecookie GitHub：[github.com/gorilla/securecookie](https://github.com/gorilla/securecookie)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)