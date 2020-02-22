---
layout:    post
title:    "Go 每日一库之 dig"
subtitle: 	"每天学习一个 Go 库"
date:		2020-02-22T12:40:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/02/22/godailylib/dig"
categories: [
	"Go"
]
---

## 简介

今天我们来介绍 Go 语言的一个依赖注入（DI）库——[dig](https://github.com/uber-go/dig)。dig 是 uber 开源的库。Java 依赖注入的库有很多，相信即使不是做 Java 开发的童鞋也听过大名鼎鼎的 Spring。相比庞大的 Spring，dig 很小巧，实现和使用都比较简洁。

## 快速使用

第三方库需要先安装，由于我们的示例中使用了前面介绍的[go-ini](https://darjun.github.io/2020/01/15/godailylib/go-ini/)和[go-flags](https://darjun.github.io/2020/01/10/godailylib/go-flags/)，这两个库也需要安装：

```cmd
$ go get go.uber.org/dig
$ go get gopkg.in/ini.v1
$ go get github.com/jessevdk/go-flags
```

下面看看如何使用：

```golang
package main

import (
  "fmt"

  "github.com/jessevdk/go-flags"
  "go.uber.org/dig"
  "gopkg.in/ini.v1"
)

type Option struct {
  ConfigFile string `short:"c" long:"config" description:"Name of config file."`
}

func InitOption() (*Option, error) {
  var opt Option
  _, err := flags.Parse(&opt)

  return &opt, err
}

func InitConf(opt *Option) (*ini.File, error) {
  cfg, err := ini.Load(opt.ConfigFile)
  return cfg, err
}

func PrintInfo(cfg *ini.File) {
  fmt.Println("App Name:", cfg.Section("").Key("app_name").String())
  fmt.Println("Log Level:", cfg.Section("").Key("log_level").String())
}

func main() {
  container := dig.New()

  container.Provide(InitOption)
  container.Provide(InitConf)

  container.Invoke(PrintInfo)
}
```

在同一目录下创建配置文件`my.ini`：

```ini
app_name = awesome web
log_level = DEBUG

[mysql]
ip = 127.0.0.1
port = 3306
user = dj
password = 123456
database = awesome

[redis]
ip = 127.0.0.1
port = 6381
```

运行程序，输出：

```cmd
$ go run main.go -c=my.ini
App Name: awesome web
Log Level: DEBUG
```

`dig`库帮助开发者管理这些对象的创建和维护，每种类型的对象会**创建且只创建一次**。`dig`库使用的一般流程：
* 创建一个容器：`dig.New`；
* 为想要让`dig`容器管理的类型创建构造函数，构造函数可以返回多个值，这些值都会被容器管理；
* 使用这些类型的时候直接编写一个函数，将这些类型作为参数，然后使用`container.Invoke`执行我们编写的函数。

## 参数对象

有时候，创建对象有很多依赖，或者编写函数时有多个参数依赖。如果将这些依赖都作为参数传入，那么代码将变得非常难以阅读：

```golang
container.Provide(func (arg1 *Arg1, arg2 *Arg2, arg3 *Arg3, ....) {
  // ...
})
```

`dig`支持将所有参数打包进一个对象中，唯一需要的就是将`dig.In`内嵌到该类型中：

```golang
type Params {
  dig.In

  Arg1 *Arg1
  Arg2 *Arg2
  Arg3 *Arg3
  Arg4 *Arg4
}

container.Provide(func (params Params) *Object {
  // ...
})
```

内嵌了`dig.In`之后，`dig`会将该类型中的其它字段看成`Object`的依赖，创建`Object`类型的对象时，会先将依赖的`Arg1/Arg2/Arg3/Arg4`创建好。

```golang
package main

import (
  "fmt"
  "log"

  "github.com/jessevdk/go-flags"
  "go.uber.org/dig"
  "gopkg.in/ini.v1"
)

type Option struct {
  ConfigFile string `short:"c" long:"config" description:"Name of config file."`
}

type RedisConfig struct {
  IP   string
  Port int
  DB   int
}

type MySQLConfig struct {
  IP       string
  Port     int
  User     string
  Password string
  Database string
}

type Config struct {
  dig.In

  Redis *RedisConfig
  MySQL *MySQLConfig
}

func InitOption() (*Option, error) {
  var opt Option
  _, err := flags.Parse(&opt)

  return &opt, err
}

func InitConfig(opt *Option) (*ini.File, error) {
  cfg, err := ini.Load(opt.ConfigFile)
  return cfg, err
}

func InitRedisConfig(cfg *ini.File) (*RedisConfig, error) {
  port, err := cfg.Section("redis").Key("port").Int()
  if err != nil {
    log.Fatal(err)
    return nil, err
  }

  db, err := cfg.Section("redis").Key("db").Int()
  if err != nil {
    log.Fatal(err)
    return nil, err
  }

  return &RedisConfig{
    IP:   cfg.Section("redis").Key("ip").String(),
    Port: port,
    DB:   db,
  }, nil
}

func InitMySQLConfig(cfg *ini.File) (*MySQLConfig, error) {
  port, err := cfg.Section("mysql").Key("port").Int()
  if err != nil {
    return nil, err
  }

  return &MySQLConfig{
    IP:       cfg.Section("mysql").Key("ip").String(),
    Port:     port,
    User:     cfg.Section("mysql").Key("user").String(),
    Password: cfg.Section("mysql").Key("password").String(),
    Database: cfg.Section("mysql").Key("database").String(),
  }, nil
}

func PrintInfo(config Config) {
  fmt.Println("=========== redis section ===========")
  fmt.Println("redis ip:", config.Redis.IP)
  fmt.Println("redis port:", config.Redis.Port)
  fmt.Println("redis db:", config.Redis.DB)

  fmt.Println("=========== mysql section ===========")
  fmt.Println("mysql ip:", config.MySQL.IP)
  fmt.Println("mysql port:", config.MySQL.Port)
  fmt.Println("mysql user:", config.MySQL.User)
  fmt.Println("mysql password:", config.MySQL.Password)
  fmt.Println("mysql db:", config.MySQL.Database)
}

func main() {
  container := dig.New()

  container.Provide(InitOption)
  container.Provide(InitConfig)
  container.Provide(InitRedisConfig)
  container.Provide(InitMySQLConfig)

  err := container.Invoke(PrintInfo)
  if err != nil {
    log.Fatal(err)
  }
}
```

上面代码中，类型`Config`内嵌了`dig.In`，`PrintInfo`接受一个`Config`类型的参数。调用`Invoke`时，`dig`自动调用`InitRedisConfig`和`InitMySQLConfig`，并将生成的`*RedisConfig`和`*MySQLConfig`“打包”成一个`Config`对象传给`PrintInfo`。

运行结果：

```cmd
$ go run main.go -c=my.ini
=========== redis section ===========
redis ip: 127.0.0.1
redis port: 6381
redis db: 1
=========== mysql section ===========
mysql ip: 127.0.0.1
mysql port: 3306
mysql user: dj
mysql password: 123456
mysql db: awesome
```

## 结果对象

前面说过，如果构造函数返回多个值，这些不同类型的值都会存储到`dig`容器中。参数过多会影响代码的可读性和可维护性，返回值过多同样也是如此。为此，`dig`提供了返回值对象，返回一个包含多个类型对象的对象。返回的类型，必须内嵌`dig.Out`：

```golang
type Results struct {
  dig.Out

  Result1 *Result1
  Result2 *Result2
  Result3 *Result3
  Result4 *Result4
}
```

```golang
dig.Provide(func () (Results, error) {
  // ...
})
```

我们把上面的例子稍作修改。将`Config`内嵌的`dig.In`变为`dig.Out`：

```golang
type Config struct {
  dig.Out

  Redis *RedisConfig
  MySQL *MySQLConfig
}
```

提供构造函数`InitRedisAndMySQLConfig`同时创建`RedisConfig`和`MySQLConfig`，通过`Config`返回。这样就不需要将`InitRedisConfig`和`InitMySQLConfig`加入`dig`容器了：

```golang
func InitRedisAndMySQLConfig(cfg *ini.File) (Config, error) {
  var config Config

  redis, err := InitRedisConfig(cfg)
  if err != nil {
    return config, err
  }

  mysql, err := InitMySQLConfig(cfg)
  if err != nil {
    return config, err
  }

  config.Redis = redis
  config.MySQL = mysql
  return config, nil
}

func main() {
  container := dig.New()

  container.Provide(InitOption)
  container.Provide(InitConfig)
  container.Provide(InitRedisAndMySQLConfig)

  err := container.Invoke(PrintInfo)
  if err != nil {
    log.Fatal(err)
  }
}
```

`PrintInfo`直接依赖`RedisConfig`和`MySQLConfig`：

```golang
func PrintInfo(redis *RedisConfig, mysql *MySQLConfig) {
  fmt.Println("=========== redis section ===========")
  fmt.Println("redis ip:", redis.IP)
  fmt.Println("redis port:", redis.Port)
  fmt.Println("redis db:", redis.DB)

  fmt.Println("=========== mysql section ===========")
  fmt.Println("mysql ip:", mysql.IP)
  fmt.Println("mysql port:", mysql.Port)
  fmt.Println("mysql user:", mysql.User)
  fmt.Println("mysql password:", mysql.Password)
  fmt.Println("mysql db:", mysql.Database)
}
```

可以看到`InitRedisAndMySQLConfig`返回`Config`类型的对象，该类型中的`RedisConfig`和`MySQLConfig`都被添加到了容器中，`PrintInfo`函数可直接使用。

运行结果与之前的例子完全一样。

## 可选依赖

默认情况下，容器如果找不到对应的依赖，那么相应的对象无法创建成功，调用`Invoke`时也会返回错误。有些依赖不是必须的，`dig`也提供了一种方式将依赖设置为可选的：

```golang
type Config struct {
  dig.In

  Redis *RedisConfig `optional:"true"`
  MySQL *MySQLConfig
}
```

通过在字段后添加结构标签`optional:"true"`，我们将`RedisConfig`这个依赖设置为可选的，容器中`RedisConfig`对象也不要紧，这时传入的`Config`中`redis`为 nil，方法可以正常调用。显然可选依赖只能在参数对象中使用。

我们直接注释掉`InitRedisConfig`，然后运行程序：

```golang
// 省略部分代码
func PrintInfo(config Config) {
  if config.Redis == nil {
    fmt.Println("no redis config")
  }
}

func main() {
  container := dig.New()

  container.Provide(InitOption)
  container.Provide(InitConfig)
  container.Provide(InitMySQLConfig)

  container.Invoke(PrintInfo)
}
```

输出：

```cmd
$ go run main.go -c=my.ini
no redis config
```

注意，创建失败和没有提供构造函数是两个概念。如果`InitRedisConfig`调用失败了，使用`Invoke`执行`PrintInfo`还是会报错的。

## 命名

前面我们说过，`dig`默认只会为每种类型创建一个对象。如果要创建某个类型的多个对象怎么办呢？可以为对象命名！

调用容器的`Provide`方法时，可以为构造函数的返回对象命名，这样同一个类型就可以有多个对象了。

```golang
type User struct {
  Name string
  Age  int
}

func NewUser(name string, age int) func() *User{} {
  return func() *User {
    return &User{name, age}
  }
}
container.Provide(NewUser("dj", 18), dig.Name("dj"))
container.Provide(NewUser("dj2", 18), dig.Name("dj2"))
```

也可以在结果对象中通过结构标签指定：

```golang
type UserResults struct {
  dig.Out

  User1 *User `name:"dj"`
  User2 *User `name:"dj2"`
}
```

然后在参数对象中通过名字指定使用哪个对象：

```golang
type UserParams struct {
  dig.In

  User1 *User `name:"dj"`
  User2 *User `name:"dj2"`
}
```

完整代码：

```golang
package main

import (
  "fmt"

  "go.uber.org/dig"
)

type User struct {
  Name string
  Age  int
}

func NewUser(name string, age int) func() *User {
  return func() *User {
    return &User{name, age}
  }
}

type UserParams struct {
  dig.In

  User1 *User `name:"dj"`
  User2 *User `name:"dj2"`
}

func PrintInfo(params UserParams) error {
  fmt.Println("User 1 ===========")
  fmt.Println("Name:", params.User1.Name)
  fmt.Println("Age:", params.User1.Age)

  fmt.Println("User 2 ===========")
  fmt.Println("Name:", params.User2.Name)
  fmt.Println("Age:", params.User2.Age)
  return nil
}

func main() {
  container := dig.New()

  container.Provide(NewUser("dj", 18), dig.Name("dj"))
  container.Provide(NewUser("dj2", 18), dig.Name("dj2"))

  container.Invoke(PrintInfo)
}
```

程序运行结果：

```cmd
$ go run main.go
User 1 ===========
Name: dj
Age: 18
User 2 ===========
Name: dj2
Age: 18
```

需要注意的时候，`NewUser`返回的是一个函数，由`dig`在需要的时候调用。

## 组

组可以将相同类型的对象放到一个切片中，可以直接使用这个切片。组的定义与上面名字定义类似。可以通过为`Provide`提供额外的参数：

```golang
container.Provide(NewUser("dj", 18), dig.Group("user"))
container.Provide(NewUser("dj2", 18), dig.Group("user"))
```

也可以在结果对象中添加结构标签`group:"user"`。

然后我们定义一个参数对象，通过指定同样的结构标签来使用这个切片：

```golang
type UserParams struct {
  dig.In

  Users []User `group:"user"`
}

func Info(params UserParams) error {
  for _, u := range params.Users {
    fmt.Println(u.Name, u.Age)
  }

  return nil
}

container.Invoke(Info)
```

最后我们通过一个完整的例子演示组的使用，我们将创建一个 HTTP 服务器：

```golang
package main

import (
  "fmt"
  "net/http"

  "go.uber.org/dig"
)

type Handler struct {
  Greeting string
  Path     string
}

func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "%s from %s", h.Greeting, h.Path)
}

func NewHello1Handler() HandlerResult {
  return HandlerResult{
    Handler: Handler{
      Path:     "/hello1",
      Greeting: "welcome",
    },
  }
}

func NewHello2Handler() HandlerResult {
  return HandlerResult{
    Handler: Handler{
      Path:     "/hello2",
      Greeting: "😄",
    },
  }
}

type HandlerResult struct {
  dig.Out

  Handler Handler `group:"server"`
}

type HandlerParams struct {
  dig.In

  Handlers []Handler `group:"server"`
}

func RunServer(params HandlerParams) error {
  mux := http.NewServeMux()
  for _, h := range params.Handlers {
    mux.Handle(h.Path, h)
  }

  server := &http.Server{
    Addr:    ":8080",
    Handler: mux,
  }
  if err := server.ListenAndServe(); err != nil {
    return err
  }

  return nil
}

func main() {
  container := dig.New()

  container.Provide(NewHello1Handler)
  container.Provide(NewHello2Handler)

  container.Invoke(RunServer)
}
```

我们创建了两个处理器，添加到`server`组中，在`RunServer`函数中创建 HTTP 服务器，将这些处理器注册到服务器中。

运行程序，在浏览器中输入`localhost:8080/hello1`和`localhost:8080/hello2`看看。关于 Go Web 编程相关的知识，可以看看我写的 Go Web 编程系列文章：

* [Go Web 编程之 Hello World](https://darjun.github.io/2019/11/25/goweb/hello-world/)
* [Go Web 编程之 程序结构](https://darjun.github.io/2019/12/05/goweb/structure/)
* [Go Web 编程之 请求](https://darjun.github.io/2019/12/09/goweb/request/)
* [Go Web 编程之 响应](https://darjun.github.io/2019/12/18/goweb/response/)
* [Go Web 编程之 模板（一）](https://darjun.github.io/2019/12/31/goweb/template1/)
* [Go Web 编程之 模板（二）](https://darjun.github.io/2019/12/31/goweb/template2/)
* [Go Web 编程之 静态文件](https://darjun.github.io/2020/01/13/goweb/fileserver/)
* [Go Web 编程之 数据库](https://darjun.github.io/2020/01/16/goweb/db/)

## 常见错误

使用`dig`过程中会遇到一些错误，我们来看看常见的错误。

`Invoke`方法在以下几种情况下会返回一个`error`：

* 无法找到依赖，或依赖创建失败；
* `Invoke`执行的函数返回`error`，该错误也会被传给调用者。

这两种情况，我们都可以判断`Invoke`的返回值来查找原因。

## 总结

本文介绍了`dig`库，它适用于解决循环依赖的对象创建问题。同时也有利于将关注点分离，我们不需要将各种对象传来传去，只需要将构造函数交给`dig`容器，然后通过`Invoke`直接使用依赖即可，连判空逻辑都可以省略了！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. dig GitHub：[https://github.com/uber-go/dig](https://github.com/uber-go/dig)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)