---
layout:    post
title:    "Go 每日一库之 jj"
subtitle: 	"每天学习一个 Go 库"
date:		2020-03-25T22:52:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/03/25/godailylib/jj"
categories: [
	"Go"
]
---

## 简介

在前面两篇文章中，我们分别介绍了快速读取 JSON 值的库`gjson`和快速设置 JSON 值的库`sjson`。今天我们介绍它们的作者[tidwall](https://github.com/tidwall)的一个基于`gjson`和`sjson`的非常实用的命令行工具[`jj`](https://github.com/tidwall/jj)。它是使用 Go 编写的快速读取和设置 JSON 值的命令行程序。

## 快速使用

Mac 上可以直接使用`brew install tidwall/jj/jj`安装。其他系统可以通过下载编译好的可执行程序，下载地址为[https://github.com/tidwall/jj/releases](https://github.com/tidwall/jj/releases)。

我选择使用`go get`安装：

```golang
$ go get github.com/tidwall/jj/cmd/jj
```

上面命令执行完成之后，编译生成的`jj`程序会放在`$GOPATH/bin`目录中，我习惯把`$GOPATH/bin`加入系统可执行目录`$PATH`中，故可以直接使用。

简单的读取和设置（我的环境为 Win10 + Git Bash）：

```golang
$ echo '{"name":{"first":"li","last":"dj"}}' | jj name.last
dj

$ echo '{"name":{"first":"li","last":"dj"}}' | jj -v dajun name.last
{"name":{"first":"li","last":"dajun"}}
```

通过**键路径**来指定读取/设置的位置，上面第一个命令读取字段`name.last`，返回`dj`。

`-v`选项指定设置的值。第二个命令将字段`name.last`设置为`dajun`，输出设置之后的 JSON 串。键路径在前两篇文章中有详细的介绍，不熟悉的可以回去看一下。

## 读取和设置

实际上读取和设置的语法和形式与我们前面介绍`gjson`和`sjson`提到的基本一样，只不过是在命令行上完成的而已。

读取不存在的字段，返回`null`：

```golang
$ echo '{"name":{"first":"li","last":"dj"}}' | jj name.middle
null
```

读取一个对象类型的字段，返回该对象的 JSON 表示：

```golang
$ echo '{"name":{"first":"li","last":"dj"}}' | jj name
{"first":"li","last":"dj"}
```

使用索引（从 0 开始）读取数组的元素，非法的索引将返回空：

```golang
$ echo '{"fruits":["apple","orange","banana"]}' | jj fruits.1
orange

$ echo '{"fruits":["apple","orange","banana"]}' | jj fruits.3

```

使用索引设置数组的元素，下面命令将数组`fruits`的第二个元素设置为`pear`：

```golang
$ echo '{"fruits":["apple","orange","banana"]}' | jj -v pear fruits.1
{"fruits":["apple","pear","banana"]}
```

使用`-1`或数组长度作为索引，可以在数组后添加一个元素。如果索引超过了数组长度，则会多一定数量的`null`：

```golang
$ echo '{"fruits":["apple","orange","banana"]}' | jj -v strawberry fruits.-1
{"fruits":["apple","orange","banana","strawberry"]}

$ echo '{"fruits":["apple","orange","banana"]}' | jj -v grape fruits.3
{"fruits":["apple","orange","banana","grape"]}

$ echo '{"fruits":["apple","orange","banana"]}' | jj -v watermelon fruits.5
{"fruits":["apple","orange","banana",null,null,"watermelon"]}
```

使用选项`-D`删除指定键路径上的元素，如果对应元素不存在，则无效果：

```golang
$ echo '{"name":"dj","age":18}' | jj -D age
{"name":"dj"}

$ echo '{"fruits":["apple","orange","banana"]}' | jj -D fruits.2
{"fruits":["apple","orange"]}

$ echo '{"fruits":["apple","orange","banana"]}' | jj -D fruits.5
{"fruits":["apple","orange","banana"]}
```

第 1 个命令删除字段`age`；第 2 个命令删除数组`fruits`的第 2 个元素；第 3 个命令删除数组`fruits`的第 5 个元素，由于数组长度只有 3，故无效果。

## 文件

`jj`支持从文件中读取 JSON 串和将结果写到文件中。使用选项`-i`指定输入文件，选项`-o`指定输出文件。下面将从文件`fruits.txt`中读取 JSON 串，取数组的第 2 个元素，写到`out.txt`中：

```golang
$ jj -i fruits.txt -o out.txt fruits.1
```

`fruits.txt`的文件内容如下：

```golang
{"fruits":["apple","orange","banana"]}
```

执行命令，输出文件的内容为：

```golang
orange
```

## 格式化

`jj`支持将输出的 JSON 串进行一定的格式化。选项`-u`移除所有的空白符，节省存储空间。选项`-p`美化格式，便于阅读。

```golang
$ echo '{"name":{"first": "li", "last":"dj"}, "age":18}' | jj -u name
{"first":"li","last":"dj"}

$ echo '{"name":{"first": "li", "last":"dj"}, "age":18}' | jj -p name
{
  "first": "li",
  "last": "dj"
}
```

## 性能

与另一个 JSON 的命令行工具[`jq`](https://stedolan.github.io/jq/)相比，`jj`是其性能的 10 倍以上。因为`jj`不会验证 JSON 串的有效性，并且它只关心键路径指定的值，一旦该值处理完成就停止。这里有性能对比：[https://github.com/tidwall/jj#performance](https://github.com/tidwall/jj#performance)

## 用途

`jj`一个很方便的用途在于日志处理，当前很多日志库都支持 JSON 的格式，例如前面我们介绍的`logrus`。我们可以使用`jj`在这些日志中找到相应的信息。我们先用`logrus`生成 20 条玩家登陆和下线的日志：

```golang
package main

import "github.com/sirupsen/logrus"

func main() {
  logrus.SetFormatter(&logrus.JSONFormatter{})

  for i := 1; i <= 10; i++ {
    logrus.WithFields(logrus.Fields{
      "userid": i,
    }).Info("login")
    logrus.WithFields(logrus.Fields{
      "userid": i,
    }).Info("logoff")
  }
}
```

生成日志存储在`log.txt`文件中：

```golang
{"level":"info","msg":"login","time":"2020-03-26T23:36:04+08:00","userid":1}
{"level":"info","msg":"logoff","time":"2020-03-26T23:36:04+08:00","userid":1}
{"level":"info","msg":"login","time":"2020-03-26T23:36:04+08:00","userid":2}
{"level":"info","msg":"logoff","time":"2020-03-26T23:36:04+08:00","userid":2}
{"level":"info","msg":"login","time":"2020-03-26T23:36:04+08:00","userid":3}
{"level":"info","msg":"logoff","time":"2020-03-26T23:36:04+08:00","userid":3}
{"level":"info","msg":"login","time":"2020-03-26T23:36:04+08:00","userid":4}
{"level":"info","msg":"logoff","time":"2020-03-26T23:36:04+08:00","userid":4}
{"level":"info","msg":"login","time":"2020-03-26T23:36:04+08:00","userid":5}
{"level":"info","msg":"logoff","time":"2020-03-26T23:36:04+08:00","userid":5}
{"level":"info","msg":"login","time":"2020-03-26T23:36:04+08:00","userid":6}
{"level":"info","msg":"logoff","time":"2020-03-26T23:36:04+08:00","userid":6}
{"level":"info","msg":"login","time":"2020-03-26T23:36:04+08:00","userid":7}
{"level":"info","msg":"logoff","time":"2020-03-26T23:36:04+08:00","userid":7}
{"level":"info","msg":"login","time":"2020-03-26T23:36:04+08:00","userid":8}
{"level":"info","msg":"logoff","time":"2020-03-26T23:36:04+08:00","userid":8}
{"level":"info","msg":"login","time":"2020-03-26T23:36:04+08:00","userid":9}
{"level":"info","msg":"logoff","time":"2020-03-26T23:36:04+08:00","userid":9}
{"level":"info","msg":"login","time":"2020-03-26T23:36:04+08:00","userid":10}
{"level":"info","msg":"logoff","time":"2020-03-26T23:36:04+08:00","userid":10}
```

由于每一行都是一个单独的 JSON 串，我们可以使用`jj`支持的 JSON 行特性，使用`..`路径标识这些行。`..`使得`jj`将这些行看成数组的元素。我们可以读取这些数组元素。

获取数组长度，返回 20：

```golang
$ jj -i log.txt ..#
20
```

只读取每一行中的`userid`信息：

```golang
$ jj -i log.txt ..#.userid
[1,1,2,2,3,3,4,4,5,5,6,6,7,7,8,8,9,9,10,10]
```

只读取每一行中的`msg`信息：

```golang
$ jj -i log.txt ..#.msg
["login","logoff","login","logoff","login","logoff","login","logoff","login","logoff","login","logoff","login","logoff","login","logoff","login","logoff","login","logoff"]
```

更复杂一点的，如果我们要查看所有`userid=1`的日志：

```golang
$ jj -i log.txt ..#\(userid=1\)# -p
[  
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 1
  },
  {
    "level": "info",
    "msg": "logoff",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 1
  }
]
```

上面的命令注意两点，`(`和`)`是 shell 中的特殊字符，需要`\`转义。命令中我们使用`-p`选项使结果更易读。

如果我们只需要查找第一条符合条件的日志，则可以去掉最右侧的`#`：

```golang
$ jj -i log.txt ..#\(userid=1\) -p
{
  "level": "info",
  "msg": "login",
  "time": "2020-03-26T23:36:04+08:00",
  "userid": 1
}
```

如果要查看所有的登录信息：

```golang
$ jj -i log.txt ..#\(msg="login"\)# -p
[
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 1
  },
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 2
  },
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 3
  },
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 4
  },
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 5
  },
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 6
  },
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 7
  },
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 8
  },
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 9
  },
  {
    "level": "info",
    "msg": "login",
    "time": "2020-03-26T23:36:04+08:00",
    "userid": 10
  }
]
```

## 总结

`jj`是一个非常使用的 JSON 命令行工具，性能超赞。执行`jj -h`去看看其他选项吧。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. jj GitHub：[https://github.com/tidwall/jj](https://github.com/tidwall/jj)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)