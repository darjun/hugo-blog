---
layout:    post
title:    "一起用Go来做一个游戏(下)"
subtitle: "每天学习一个 Go 库"
date:     2022-11-23T22:39:30
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2022/11/23/godailylib/ebiten2"
categories: [
  "Go"
]
---

## 打包资源

使用一个file2byteslice包我们可以将图片和config.json文件打包进二进制程序中，之后编译生成一个二进制程序。然后拷贝这一个文件即可，不用再拷贝图片和其他配置文件了。

golang有很多第三方包可以将打包资源，原理其实很简单——读取资源文件的内容，然后生成一个go文件，在这个文件中创建一个变量保存这个文件的二进制内容。

我们将使用ebiten作者编写的file2byteslice包。首先使用`go install`命令安装它：

```cmd
$ go install github.com/hajimehoshi/file2byteslice
```

file2byteslice的命令格式如下：

```cmd
$ file2byteslice -input INPUT_FILE -output OUTPUT_FILE -package PACKAGE_NAME -var VARIABLE_NAME
```

故我们可以这样来打包文件：

```cmd
$ file2byteslice -input ../images/ship.png -output resources/ship.go -package resources -var ShipPng
$ file2byteslice -input ../images/alien.png -output resources/alien.go -package resources -var AlienPng
$ file2byteslice -input config.json -output resources/config.go -package resources -var ConfigJson
```

生成文件如下：

![](/img/in-post/godailylib/ebiten22.png#center)
![](/img/in-post/godailylib/ebiten23.png#center)
![](/img/in-post/godailylib/ebiten24.png#center)

相应的加载这些文件的代码需要相应的修改：

```golang
// alien.go
func NewAlien(cfg *Config) *Alien {
	img, _, err := ebitenutil.NewImageFromReader(bytes.NewReader(resources.AlienPng))
	if err != nil {
		log.Fatal(err)
	}
  // ...
}
```

```golang
// ship.go
func NewShip(screenWidth, screenHeight int) *Ship {
	img, _, err := ebitenutil.NewImageFromReader(bytes.NewReader(resources.ShipPng))
	if err != nil {
		log.Fatal(err)
	}
  // ...
}
```

```golang
// config.go
func loadConfig() *Config {
	var cfg Config
	if err := json.NewDecoder(bytes.NewReader(resources.ConfigJson)).Decode(&cfg); err != nil {
		log.Fatalf("json.Decode failed: %v\n", err)
	}

	return &cfg
}
```

然后，我们就可以编译成一个游戏二进制程序随意拷贝到其他电脑上运行了：

```cmd
$ go build -o alien_invasion
```

### go generate

前面先安装file2byteslice程序，然后一个命令一个命令地执行打包，操作起来很不方便。如果有文件修改，这个过程又需要来一次。

实际上，我们可以使用`go generate`让上面的过程更智能一点。在main.go文件中添加如下几行注释：

```golang
//go:generate go install github.com/hajimehoshi/file2byteslice
//go:generate mkdir resources
//go:generate file2byteslice -input ../images/ship.png -output resources/ship.go -package resources -var ShipPng
//go:generate file2byteslice -input ../images/alien.png -output resources/alien.go -package resources -var AlienPng
//go:generate file2byteslice -input config.json -output resources/config.go -package resources -var ConfigJson
```

注意，`//`和`go:generate`之间一定不能有空格，一定不能有空格，一定不能有空格，重要的事情说3遍！然后执行下面的命令即可完成安装file2byteslice和打包资源的工作：

```cmd
$ go generate
```

## 让游戏在网页上运行

借助于wasm的强大功能，我们的游戏可以很好地在web上运行！为了让程序能够在网页上运行，我们需要将其编译成wasm。Go内置对wasm的支持。编译方式如下：

```cmd
$ GOOS=js GOARCH=wasm go build -o alien_invasion.wasm
```

Go提供的胶水代码，将位于`$GOROOT/misc/wasm`目录下的wasm_exec.html和wasm_exec.js文件拷贝到我们的项目目录下。注意wasm_exec.html文件中默认是加载名为test.wasm的文件，我们需要将加载文件改为alien_invasion.wasm，或者将生成的文件改名为test.wasm。

然后编写一个简单的web服务器：

```golang
package main

import (
	"log"
	"net/http"
)

func main() {
	if err := http.ListenAndServe(":8080", http.FileServer(http.Dir("."))); err != nil {
		log.Fatal(err)
	}
}
```

运行：

```cmd
$ go run main.go
```

打开浏览器输入地址：localhost:8080/wasm_exec.html。

![](/img/in-post/godailylib/ebiten25.png#center)

点击run按钮即可愉快地玩耍啦！

![](/img/in-post/godailylib/ebiten26.mp4#center)

## 项目的不足

到目前为止，我们的游戏基本上可玩，但是还有很多的不足：
* 没有声音！
* 外星人没有横向的运动！
* 分数都没有！

这些有兴趣的童鞋可以自己去实现了😀。

## 总结

接着上文，本文介绍了如何将资源文件打包进一个二进制程序中，方便相互之间的传播。然后我们不费吹灰之力就将这个游戏移至到了网页之中。

总的来说ebiten是一款简单、易上手的2D游戏开发引擎。对游戏开发感兴趣的童鞋可以使用它来快速开发，引起自己的兴趣。用它来开发一些小游戏也是得心应手，而且自带跨平台功能，十分方便。但是，大型、复杂游戏的开发还是要借助专业的引擎。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)
2. ebitengine 官网：[https://ebitengine.org/](https://ebitengine.org/)
3. Python 编程（从入门到实践）：[https://book.douban.com/subject/35196328/](https://book.douban.com/subject/35196328/)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)