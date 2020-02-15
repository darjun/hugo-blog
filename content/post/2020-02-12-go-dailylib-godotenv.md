---
layout:    post
title:    "Go 每日一库之 godotenv"
subtitle: 	"每天学习一个 Go 库"
date:		2020-02-12T20:45:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/02/12/godailylib/godotenv"
categories: [
	"Go"
]
---

## 简介

[twelve-factor](https://12factor.net/)应用提倡将配置存储在环境变量中。任何从开发环境切换到生产环境时需要修改的东西都从代码抽取到环境变量里。
但是在实际开发中，如果同一台机器运行多个项目，设置环境变量容易冲突，不实用。[godotenv](https://github.com/joho/godotenv)库从`.env`文件中读取配置，
然后存储到程序的环境变量中。在代码中可以使用读取非常方便。`godotenv`源于一个 Ruby 的开源项目[dotenv](https://github.com/bkeepers/dotenv)。

## 快速使用

第三方库需要先安装：

```
$ go get github.com/joho/godotenv
```

后使用：

```golang
package main

import (
  "fmt"
  "log"
  "os"

  "github.com/joho/godotenv"
)

func main() {
  err := godotenv.Load()
  if err != nil {
    log.Fatal(err)
  }

  fmt.Println("name: ", os.Getenv("name"))
  fmt.Println("age: ", os.Getenv("age"))
}
```

然后在可执行程序相同目录下，添加一个`.env`文件：

```env
name = dj
age = 18
```

运行程序，输出：

```
name:  dj
age:  18
```

可见，使用非常方便。默认情况下，`godotenv`读取项目根目录下的`.env`文件，文件中使用`key = value`的格式，每行一个键值对。
调用`godotenv.Load()`即可加载，可直接调用`os.Getenv("key")`读取。`os.Getenv`是用来读取环境变量的：

```golang
package main

import (
  "fmt"
  "os"
)

func main() {
  fmt.Println(os.Getenv("GOPATH"))
}
```

## 高级特性

### 自动加载

如果你有程序员的优良传统——懒，你可能连`Load`方法都不想自己调用。没关系，`godotenv`给你懒的权力！

导入`github.com/joho/godotenv/autoload`，配置会自动读取：

```golang
package main

import (
  "fmt"
  "os"

  _ "github.com/joho/godotenv/autoload"
)

func main() {
  fmt.Println("name: ", os.Getenv("name"))
  fmt.Println("age: ", os.Getenv("age"))
}
```

注意，由于代码中没有显式用到`godotenv`库，需要使用空导入，即导入时包名前添加一个`_`。

看`autoload`包的源码，其实就是库帮你调用了`Load`方法：

```golang
// src/github.com/joho/godotenv/autoload/autoload.go
package autoload

/*
	You can just read the .env file on import just by doing

		import _ "github.com/joho/godotenv/autoload"

	And bob's your mother's brother
*/

import "github.com/joho/godotenv"

func init() {
  godotenv.Load()
}
```

仔细看注释，程序员的恶趣味😂！

### 加载自定义文件

默认情况下，加载的是项目根目录下的`.env`文件。当然我们可以加载任意名称的文件，文件也不必以`.env`为后缀：

```golang
package main

import (
  "fmt"
  "log"
  "os"

  "github.com/joho/godotenv"
)

func main() {
  err := godotenv.Load("common", "dev.env")
  if err != nil {
    log.Fatal(err)
  }

  fmt.Println("name: ", os.Getenv("name"))
  fmt.Println("version: ", os.Getenv("version"))
  fmt.Println("database: ", os.Getenv("database"))
}
```

`common`文件内容：

```
name = awesome web
version = 0.0.1
```

`dev.env`：
```
database = sqlite
```

`production.env`：
```
database = mysql
```

自己运行看看结果吧！

注意：`Load`接收多个文件名作为参数，如果不传入文件名，默认读取`.env`文件的内容。
如果多个文件中存在同一个键，那么先出现的优先，后出现的不生效。所以，上面输出的`database`是什么？

### 注释

`.env`文件中可以添加注释，注释以`#`开始，直到该行结束。

```
# app name
name = awesome web
# current version
version = 0.0.1
```

### YAML

`.env`文件还可以使用 YAML 格式：

```
name: awesome web
version: 0.0.1
```

```golang
package main

import (
  "fmt"
  "os"

  _ "github.com/joho/godotenv/autoload"
)

func main() {
  fmt.Println("name: ", os.Getenv("name"))
  fmt.Println("version: ", os.Getenv("version"))
}
```

### 不存入环境变量

`godotenv`允许不将`.env`文件内容存入环境变量，使用`godotenv.Read()`返回一个`map[string]string`，可直接使用：

```golang
package main

import (
  "fmt"
  "log"

  "github.com/joho/godotenv"
)

func main() {
  var myEnv map[string]string
  myEnv, err := godotenv.Read()
  if err != nil {
    log.Fatal(err)
  }

  fmt.Println("name: ", myEnv["name"])
  fmt.Println("version: ", myEnv["version"])
}
```

直接操作`map`，简单直接！

### 数据源

除了读取文件，还可以从`io.Reader`，从`string`中读取配置：

```golang
package main

import (
  "fmt"
  "log"

  "github.com/joho/godotenv"
)

func main() {
  content := `
name: awesome web
version: 0.0.1
  `
  myEnv, err := godotenv.Unmarshal(content)
  if err != nil {
    log.Fatal(err)
  }

  fmt.Println("name: ", myEnv["name"])
  fmt.Println("version: ", myEnv["version"])
}
```

只要实现了`io.Reader`接口，就能作为数据源。可以从文件（`os.File`），网络（`net.Conn`），`bytes.Buffer`等多种来源读取：

```golang
package main

import (
	"bytes"
	"fmt"
	"log"
	"os"

	"github.com/joho/godotenv"
)

func main() {
  file, _ := os.OpenFile(".env", os.O_RDONLY, 0666)
  myEnv, err := godotenv.Parse(file)
  if err != nil {
    log.Fatal(err)
  }

  fmt.Println("name: ", myEnv["name"])
  fmt.Println("version: ", myEnv["version"])

  buf := &bytes.Buffer{}
  buf.WriteString("name: awesome web @buffer")
  buf.Write([]byte{'\n'})
  buf.WriteString("version: 0.0.1")
  myEnv, err = godotenv.Parse(buf)
  if err != nil {
    log.Fatal(err)
  }

  fmt.Println("name: ", myEnv["name"])
  fmt.Println("version: ", myEnv["version"])
}
```

注意，从字符串读取和从`io.Reader`读取使用的方法是不同的。前者为`Unmarshal`，后者是`Parse`。

### 生成`.env`文件

可以通过程序生成一个`.env`文件的内容，可以直接写入到文件中：

```golang
package main

import (
  "bytes"
  "log"

  "github.com/joho/godotenv"
)

func main() {
  buf := &bytes.Buffer{}
  buf.WriteString("name = awesome web")
  buf.WriteByte('\n')
  buf.WriteString("version = 0.0.1")

  env, err := godotenv.Parse(buf)
  if err != nil {
    log.Fatal(err)
  }

  err = godotenv.Write(env, "./.env")
  if err != nil {
    log.Fatal(err)
  }
}
```

查看生成的`.env`文件：

```
name="awesome web"
version="0.0.1"
```

还可以返回一个字符串，怎么揉捏随你：

```golang
package main

import (
  "bytes"
  "fmt"
  "log"

  "github.com/joho/godotenv"
)

func main() {
  buf := &bytes.Buffer{}
  buf.WriteString("name = awesome web")
  buf.WriteByte('\n')
  buf.WriteString("version = 0.0.1")

  env, err := godotenv.Parse(buf)
  if err != nil {
    log.Fatal(err)
  }

  content, err := godotenv.Marshal(env)
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println(content)
}
```

### 命令行模式

`godotenv`还提供了一个命令行的模式：

```
$ godotenv -f ./.env command args
```

前面在`go get`安装`godotenv`时，`godotenv`就已经安装在`$GOPATH/bin`目录下了，我习惯把`$GOPATH/bin`加入系统`PATH`，所以`godotenv`命令可以直接使用。

命令行模式就是读取指定文件（如果不通过`-f`指定，则使用`.env`文件），设置环境变量，然后运行后面的程序。

我们简单写一个程序验证一下：

```golang
package main

import (
  "fmt"
  "os"
)

func main() {
  fmt.Println(os.Getenv("name"))
  fmt.Println(os.Getenv("version"))
}
```

使用`godotenv`运行一下：

```
$ godotenv -f ./.env go run main.go
```

输出：

```
awesome web
0.0.1
```

### 多个环境

实践中，一般会根据`APP_ENV`环境变量的值加载不同的文件：

```golang
package main

import (
	"fmt"
	"log"
	"os"

	"github.com/joho/godotenv"
)

func main() {
	env := os.Getenv("GODAILYLIB_ENV")
	if env == "" {
		env = "development"
	}

	err := godotenv.Load(".env." + env)
	if err != nil {
		log.Fatal(err)
	}

	err = godotenv.Load()
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("name: ", os.Getenv("name"))
	fmt.Println("version: ", os.Getenv("version"))
	fmt.Println("database: ", os.Getenv("database"))
}
```

我们先读取环境变量`GODAILYLIB_ENV`，然后读取对应的`.env.` + env，最后读取默认的`.env`文件。

前面也提到过，先读取到的优先。我们可以在默认的`.env`文件中配置基础信息和一些默认的值，
如果在开发/测试/生产环境需要修改，那么在对应的`.env.development/.env.test/.env.production`文件中再配置一次即可。

`.env`文件内容：

```env
name = awesome web
version = 0.0.1
database = file
```

`.env.development`：

```
database = sqlite3
```

`.env.production`：

```
database = mysql
```

运行程序：

```
# 默认是开发环境
$ go run main.go
name:  awesome web
version:  0.0.1
database:  sqlite3

# 设置为生成环境
$ GODAILYLIB_ENV=production go run main.go
name:  awesome web
version:  0.0.1
database:  mysql
```

## 一点源码

`godotenv`读取文件内容，为什么可以使用`os.Getenv`访问：

```golang
// src/github.com/joho/godotenv/godotenv.go
func loadFile(filename string, overload bool) error {
	envMap, err := readFile(filename)
	if err != nil {
		return err
	}

	currentEnv := map[string]bool{}
	rawEnv := os.Environ()
	for _, rawEnvLine := range rawEnv {
		key := strings.Split(rawEnvLine, "=")[0]
		currentEnv[key] = true
	}

	for key, value := range envMap {
		if !currentEnv[key] || overload {
			os.Setenv(key, value)
		}
	}

	return nil
}
```

因为`godotenv`调用`os.Setenv`将键值对设置到环境变量中了。

## 总结

本文介绍了`godotenv`库的基础和高级用法。`godotenv`的源码也比较好读，有时间，有兴趣的童鞋建议一看~

## 参考

1. godotenv GitHub 仓库：[https://github.com/joho/godotenv](https://github.com/joho/godotenv)

## 我

[我的博客](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)