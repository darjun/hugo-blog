---
layout:    post
title:    "Go 每日一库之 resty"
subtitle: "每天学习一个 Go 库"
date:     2021-06-25T10:56:23
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/06/26/godailylib/resty"
categories: [
  "Go"
]
---

## 简介

[`resty`](https://github.com/go-resty/resty)是 Go 语言的一个 HTTP client 库。`resty`功能强大，特性丰富。它支持几乎所有的 HTTP 方法（GET/POST/PUT/DELETE/OPTION/HEAD/PATCH等），并提供了简单易用的 API。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir resty && cd resty
$ go mod init github.com/darjun/go-daily-lib/resty
```

安装`resty`库：

```cmd
$ go get -u github.com/go-resty/resty/v2
```

下面我们来获取百度首页信息：

```golang
package main

import (
  "fmt"
  "log"

  "github.com/go-resty/resty/v2"
)

func main() {
  client := resty.New()

  resp, err := client.R().Get("https://baidu.com")

  if err != nil {
    log.Fatal(err)
  }

  fmt.Println("Response Info:")
  fmt.Println("Status Code:", resp.StatusCode())
  fmt.Println("Status:", resp.Status())
  fmt.Println("Proto:", resp.Proto())
  fmt.Println("Time:", resp.Time())
  fmt.Println("Received At:", resp.ReceivedAt())
  fmt.Println("Size:", resp.Size())
  fmt.Println("Headers:")
  for key, value := range resp.Header() {
    fmt.Println(key, "=", value)
  }
  fmt.Println("Cookies:")
  for i, cookie := range resp.Cookies() {
    fmt.Printf("cookie%d: name:%s value:%s\n", i, cookie.Name, cookie.Value)
  }
}
```

`resty`使用比较简单。
* 首先，调用一个`resty.New()`创建一个`client`对象；
* 调用`client`对象的`R()`方法创建一个请求对象；
* 调用请求对象的`Get()/Post()`等方法，传入参数 URL，就可以向对应的 URL 发送 HTTP 请求了。返回一个响应对象；
* 响应对象提供很多方法可以检查响应的状态，首部，Cookie 等信息。

上面程序中我们获取了：

* `StatusCode()`：状态码，如 200；
* `Status()`：状态码和状态信息，如 200 OK；
* `Proto()`：协议，如 HTTP/1.1；
* `Time()`：从发送请求到收到响应的时间；
* `ReceivedAt()`：接收到响应的时刻；
* `Size()`：响应大小；
* `Header()`：响应首部信息，以`http.Header`类型返回，即`map[string][]string`；
* `Cookies()`：服务器通过`Set-Cookie`首部设置的 cookie 信息。

运行程序输出的响应基本信息：

```golang
Response Info:
Status Code: 200
Status: 200 OK
Proto: HTTP/1.1
Time: 415.774352ms
Received At: 2021-06-26 11:42:45.307157 +0800 CST m=+0.416547795
Size: 302456
```

首部信息：

```golang
Headers:
Server = [BWS/1.1]
Date = [Sat, 26 Jun 2021 03:42:45 GMT]
Connection = [keep-alive]
Bdpagetype = [1]
Bdqid = [0xf5a61d240003b218]
Vary = [Accept-Encoding Accept-Encoding]
Content-Type = [text/html;charset=utf-8]
Set-Cookie = [BAIDUID=BF2EE47AAAF7A20C6971F1E897ABDD43:FG=1; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com BIDUPSID=BF2EE47AAAF7A20C6971F1E897ABDD43; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com PSTM=1624678965; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com BAIDUID=BF2EE47AAAF7A20C716E90B86906D6B0:FG=1; max-age=31536000; expires=Sun, 26-Jun-22 03:42:45 GMT; domain=.baidu.com; path=/; version=1; comment=bd BDSVRTM=0; path=/ BD_HOME=1; path=/ H_PS_PSSID=34099_31253_34133_34072_33607_34135_26350; path=/; domain=.baidu.com]
Traceid = [1624678965045126810617700867425882583576]
P3p = [CP=" OTI DSP COR IVA OUR IND COM " CP=" OTI DSP COR IVA OUR IND COM "]
X-Ua-Compatible = [IE=Edge,chrome=1]
```

注意其中有一个`Set-Cookie`首部，这部分内容会出现在 Cookie 部分：

```golang
Cookies:
cookie0: name:BAIDUID value:BF2EE47AAAF7A20C6971F1E897ABDD43:FG=1
cookie1: name:BIDUPSID value:BF2EE47AAAF7A20C6971F1E897ABDD43
cookie2: name:PSTM value:1624678965
cookie3: name:BAIDUID value:BF2EE47AAAF7A20C716E90B86906D6B0:FG=1
cookie4: name:BDSVRTM value:0
cookie5: name:BD_HOME value:1
cookie6: name:H_PS_PSSID value:34099_31253_34133_34072_33607_34135_26350
```

## 自动 Unmarshal

现在很多网站提供 API 接口，返回结构化的数据，如 JSON/XML 格式等。`resty`可以自动将响应数据 Unmarshal 到对应的结构体对象中。下面看一个例子，我们知道很多 js 文件都托管在 cdn 上，我们可以通过`api.cdnjs.com/libraries`获取这些库的基本信息，返回一个 JSON 数据，格式如下：

![](/img/in-post/godailylib/resty1.png#center)

接下来，我们定义结构，然后使用`resty`拉取信息，自动 Unmarshal：

```golang
type Library struct {
  Name   string
  Latest string
}

type Libraries struct {
  Results []*Library
}

func main() {
  client := resty.New()

  libraries := &Libraries{}
  client.R().SetResult(libraries).Get("https://api.cdnjs.com/libraries")
  fmt.Printf("%d libraries\n", len(libraries.Results))

  for _, lib := range libraries.Results {
    fmt.Println("first library:")
    fmt.Printf("name:%s latest:%s\n", lib.Name, lib.Latest)
    break
  }
}
```

可以看到，我们只需要创建一个结果类型的对象，然后调用请求对象的`SetResult()`方法，`resty`会自动将响应的数据 Unmarshal 到传入的对象中。**这里设置请求信息时使用链式调用的方式，即在一行中完成多个设置**。

运行：

```golang
$ go run main.go
4040 libraries
first library:
name:vue latest:https://cdnjs.cloudflare.com/ajax/libs/vue/3.1.2/vue.min.js
```

一共 4040 个库，第一个就是 Vue✌️。我们请求`https://api.cdnjs.com/libraries/vue`就能获取 Vue 的详细信息：

![](/img/in-post/godailylib/resty2.png#center)

感兴趣可自行用`resty`来拉取这些信息。

一般请求下，`resty`会根据响应中的`Content-Type`来推断数据格式。但是有时候响应中无`Content-Type`首部或与内容格式不一致，我们可以通过调用请求对象的`ForceContentType()`强制让`resty`按照特定的格式来解析响应：

```golang
client.R().
  SetResult(result).
  ForceContentType("application/json")
```

## 请求信息

`resty`提供了丰富的设置请求信息的方法。我们可以通过两种方式设置查询字符串。一种是调用请求对象的`SetQueryString()`设置我们拼接好的查询字符串：

```golang
client.R().
  SetQueryString("name=dj&age=18").
  Get(...)
```

另一种是调用请求对象的`SetQueryParams()`，传入`map[string]string`，由`resty`来帮我们拼接。显然这种更为方便：

```golang
client.R().
  SetQueryParams(map[string]string{
    "name": "dj",
    "age": "18",
  }).
  Get(...)
```

`resty`还提供一种非常实用的设置路径参数接口，我们调用`SetPathParams()`传入`map[string]string`参数，然后后面的 URL 路径中就可以使用这个`map`中的键了：

```golang
client.R().
  SetPathParams(map[string]string{
    "user": "dj",
  }).
  Get("/v1/users/{user}/details")
```

注意，路径中的键需要用`{}`包起来。

设置首部：

```golang
client.R().
  SetHeader("Content-Type", "application/json").
  Get(...)
```

设置请求消息体：

```golang
client.R().
  SetHeader("Content-Type", "application/json").
  SetBody(`{"name": "dj", "age":18}`).
  Get(...)
```

消息体可以是多种类型：字符串，`[]byte`，对象，`map[string]interface{}`等。

设置携带`Content-Length`首部，`resty`自动计算：

```golang
client.R().
  SetBody(User{Name:"dj", Age:18}).
  SetContentLength(true).
  Get(...)
```

有些网站需要先获取 token，然后才能访问它的 API。设置 token：

```golang
client.R().
  SetAuthToken("youdontknow").
  Get(...)
```

### 案例

最后，我们通过一个案例来将上面介绍的这些串起来。现在我们想通过 GitHub 提供的 API 获取组织的仓库信息，[API 文档](https://docs.github.com/en/rest/reference/projects#list-organization-projects)见文后链接。GitHub API 请求地址为`https://api.github.com`，获取仓库信息的请求格式如下：

```golang
GET /orgs/{org}/repos
```

我们还可以设置以下这些参数：

![](/img/in-post/godailylib/resty3.png#center)

* `accept`：**首部**，这个必填，需要设置为`application/vnd.github.v3+json`；
* `org`：组织名，**路径参数**；
* `type`：仓库类型，**查询参数**，例如`public/private/forks(fork的仓库)`等；
* `sort`：仓库的排序规则，**查询参数**，例如`created/updated/pushed/full_name`等。默认按创建时间排序；
* `direction`：升序`asc`或降序`dsc`，**查询参数**；
* `per_page`：每页多少条目，最大 100，默认 30，**查询参数**；
* `page`：当前请求第几页，与`per_page`一起做分页管理，默认 1，**查询参数**。

GitHub API 必须设置 token 才能访问。登录 GitHub 账号，点开右上角头像，选择`Settings`：

![](/img/in-post/godailylib/resty4.png#center)

然后，选择`Developer settings`：

![](/img/in-post/godailylib/resty5.png#center)

选择`Personal access tokens`，然后点击右上角的`Generate new token`：

![](/img/in-post/godailylib/resty6.png#center)

填写 Note，表示 token 的用途，这个根据自己情况填写即可。下面复选框用于选择该 token 有哪些权限，这里不需要勾选：

![](/img/in-post/godailylib/resty7.png#center)

点击下面的`Generate token`按钮即可生成 token：

![](/img/in-post/godailylib/resty8.png#center)

注意，这个 token 只有现在能看见，关掉页面下次再进入就无法看到了。所以要保存好，另外不要用我的 token，测试完程序后我会删除 token😭。

响应中的 JSON 格式数据如下所示：

![](/img/in-post/godailylib/resty9.png#center)

字段非常多，为了方便起见，我这里之处理几个字段：

```golang
type Repository struct {
  ID              int        `json:"id"`
  NodeID          string     `json:"node_id"`
  Name            string     `json:"name"`
  FullName        string     `json:"full_name"`
  Owner           *Developer `json:"owner"`
  Private         bool       `json:"private"`
  Description     string     `json:"description"`
  Fork            bool       `json:"fork"`
  Language        string     `json:"language"`
  ForksCount      int        `json:"forks_count"`
  StargazersCount int        `json:"stargazers_count"`
  WatchersCount   int        `json:"watchers_count"`
  OpenIssuesCount int        `json:"open_issues_count"`
}

type Developer struct {
  Login      string `json:"login"`
  ID         int    `json:"id"`
  NodeID     string `json:"node_id"`
  AvatarURL  string `json:"avatar_url"`
  GravatarID string `json:"gravatar_id"`
  Type       string `json:"type"`
  SiteAdmin  bool   `json:"site_admin"`
}
```

然后使用`resty`设置路径参数，查询参数，首部，Token 等信息，然后发起请求：

```golang
func main() {
  client := resty.New()

  var result []*Repository
  client.R().
    SetAuthToken("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").
    SetHeader("Accept", "application/vnd.github.v3+json").
    SetQueryParams(map[string]string{
      "per_page":  "3",
      "page":      "1",
      "sort":      "created",
      "direction": "asc",
    }).
    SetPathParams(map[string]string{
      "org": "golang",
    }).
    SetResult(&result).
    Get("https://api.github.com/orgs/{org}/repos")

  for i, repo := range result {
    fmt.Printf("repo%d: name:%s stars:%d forks:%d\n", i+1, repo.Name, repo.StargazersCount, repo.ForksCount)
  }
}
```

上面程序拉取以创建时间升序排列的 3 个仓库：

```golang
$ go run main.go
repo1: name:gddo stars:1097 forks:289
repo2: name:lint stars:3892 forks:518
repo3: name:glog stars:2738 forks:775
```

## Trace

介绍完`resty`的主要功能之后，我们再来看看`resty`提供的一个辅助功能：trace。我们在请求对象上调用`EnableTrace()`方法启用 trace。启用 trace 可以记录请求的每一步的耗时和其他信息。**`resty`支持链式调用，也就是说我们可以在一行中完成创建请求，启用 trace，发起请求**：

```golang
client.R().EnableTrace().Get("https://baidu.com")
```

在完成请求之后，我们通过调用请求对象的`TraceInfo()`方法获取信息：

```golang
ti := resp.Request.TraceInfo()
fmt.Println("Request Trace Info:")
fmt.Println("DNSLookup:", ti.DNSLookup)
fmt.Println("ConnTime:", ti.ConnTime)
fmt.Println("TCPConnTime:", ti.TCPConnTime)
fmt.Println("TLSHandshake:", ti.TLSHandshake)
fmt.Println("ServerTime:", ti.ServerTime)
fmt.Println("ResponseTime:", ti.ResponseTime)
fmt.Println("TotalTime:", ti.TotalTime)
fmt.Println("IsConnReused:", ti.IsConnReused)
fmt.Println("IsConnWasIdle:", ti.IsConnWasIdle)
fmt.Println("ConnIdleTime:", ti.ConnIdleTime)
fmt.Println("RequestAttempt:", ti.RequestAttempt)
fmt.Println("RemoteAddr:", ti.RemoteAddr.String())
```

我们可以获取以下信息：

* `DNSLookup`：DNS 查询时间，如果提供的是一个域名而非 IP，就需要向 DNS 系统查询对应 IP 才能进行后续操作；
* `ConnTime`：获取一个连接的耗时，可能从连接池获取，也可能新建；
* `TCPConnTime`：TCP 连接耗时，从 DNS 查询结束到 TCP 连接建立；
* `TLSHandshake`：TLS 握手耗时；
* `ServerTime`：服务器处理耗时，计算从连接建立到客户端收到第一个字节的时间间隔；
* `ResponseTime`：响应耗时，从接收到第一个响应字节，到接收到完整响应之间的时间间隔；
* `TotalTime`：整个流程的耗时；
* `IsConnReused`：TCP 连接是否复用了；
* `IsConnWasIdle`：连接是否是从空闲的连接池获取的；
* `ConnIdleTime`：连接空闲时间；
* `RequestAttempt`：请求执行流程中的请求次数，包括重试次数；
* `RemoteAddr`：远程的服务地址，`IP:PORT`格式。

`resty`对这些区分得很细。实际上`resty`也是使用标准库`net/http/httptrace`提供的功能，`httptrace`提供一个结构，我们可以设置各个阶段的回调函数：

```golang
// src/net/http/httptrace.go
type ClientTrace struct {
  GetConn func(hostPort string)
  GotConn func(GotConnInfo)
  PutIdleConn func(err error)
  GotFirstResponseByte func()
  Got100Continue func()
  Got1xxResponse func(code int, header textproto.MIMEHeader) error // Go 1.11
  DNSStart func(DNSStartInfo)
  DNSDone func(DNSDoneInfo)
  ConnectStart func(network, addr string)
  ConnectDone func(network, addr string, err error)
  TLSHandshakeStart func() // Go 1.8
  TLSHandshakeDone func(tls.ConnectionState, error) // Go 1.8
  WroteHeaderField func(key string, value []string) // Go 1.11
  WroteHeaders func()
  Wait100Continue func()
  WroteRequest func(WroteRequestInfo)
}
```

可以从字段名简单了解回调的含义。`resty`在启用 trace 后设置了如下回调：

```golang
// src/github.com/go-resty/resty/trace.go
func (t *clientTrace) createContext(ctx context.Context) context.Context {
  return httptrace.WithClientTrace(
    ctx,
    &httptrace.ClientTrace{
      DNSStart: func(_ httptrace.DNSStartInfo) {
        t.dnsStart = time.Now()
      },
      DNSDone: func(_ httptrace.DNSDoneInfo) {
        t.dnsDone = time.Now()
      },
      ConnectStart: func(_, _ string) {
        if t.dnsDone.IsZero() {
          t.dnsDone = time.Now()
        }
        if t.dnsStart.IsZero() {
          t.dnsStart = t.dnsDone
        }
      },
      ConnectDone: func(net, addr string, err error) {
        t.connectDone = time.Now()
      },
      GetConn: func(_ string) {
        t.getConn = time.Now()
      },
      GotConn: func(ci httptrace.GotConnInfo) {
        t.gotConn = time.Now()
        t.gotConnInfo = ci
      },
      GotFirstResponseByte: func() {
        t.gotFirstResponseByte = time.Now()
      },
      TLSHandshakeStart: func() {
        t.tlsHandshakeStart = time.Now()
      },
      TLSHandshakeDone: func(_ tls.ConnectionState, _ error) {
        t.tlsHandshakeDone = time.Now()
      },
    },
  )
}
```

然后在获取`TraceInfo`时，根据各个时间点计算耗时：

```golang
// src/github.com/go-resty/resty/request.go
func (r *Request) TraceInfo() TraceInfo {
  ct := r.clientTrace

  if ct == nil {
    return TraceInfo{}
  }

  ti := TraceInfo{
    DNSLookup:      ct.dnsDone.Sub(ct.dnsStart),
    TLSHandshake:   ct.tlsHandshakeDone.Sub(ct.tlsHandshakeStart),
    ServerTime:     ct.gotFirstResponseByte.Sub(ct.gotConn),
    IsConnReused:   ct.gotConnInfo.Reused,
    IsConnWasIdle:  ct.gotConnInfo.WasIdle,
    ConnIdleTime:   ct.gotConnInfo.IdleTime,
    RequestAttempt: r.Attempt,
  }

  if ct.gotConnInfo.Reused {
    ti.TotalTime = ct.endTime.Sub(ct.getConn)
  } else {
    ti.TotalTime = ct.endTime.Sub(ct.dnsStart)
  }

  if !ct.connectDone.IsZero() {
    ti.TCPConnTime = ct.connectDone.Sub(ct.dnsDone)
  }

  if !ct.gotConn.IsZero() {
    ti.ConnTime = ct.gotConn.Sub(ct.getConn)
  }

  if !ct.gotFirstResponseByte.IsZero() {
    ti.ResponseTime = ct.endTime.Sub(ct.gotFirstResponseByte)
  }

  if ct.gotConnInfo.Conn != nil {
    ti.RemoteAddr = ct.gotConnInfo.Conn.RemoteAddr()
  }

  return ti
}
```

运行输出：

```golang
$ go run main.go
Request Trace Info:
DNSLookup: 2.815171ms
ConnTime: 941.635171ms
TCPConnTime: 269.069692ms
TLSHandshake: 669.276011ms
ServerTime: 274.623991ms
ResponseTime: 112.216µs
TotalTime: 1.216276906s
IsConnReused: false
IsConnWasIdle: false
ConnIdleTime: 0s
RequestAttempt: 1
RemoteAddr: 18.235.124.214:443
```

我们看到 TLS 消耗了近一半的时间。

## 总结

本文我介绍了 Go 语言一款非常方便易用的 HTTP Client 库。 `resty`提供非常实用的，丰富的 API。链式调用，自动 Unmarshal，请求参数/路径设置这些功能非常方便好用，让我们的工作事半功倍。限于篇幅原因，很多高级特性未能一一介绍，如提交表单，上传文件等等等等。只能留待感兴趣的大家去探索了。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)
2. resty GitHub：[github.com/go-resty/resty](https://github.com/go-resty/resty)
3. GitHub API：[https://docs.github.com/en/rest/overview/resources-in-the-rest-api](https://docs.github.com/en/rest/overview/resources-in-the-rest-api)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~