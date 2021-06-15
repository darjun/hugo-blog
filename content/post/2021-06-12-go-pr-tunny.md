---
layout:    post
title:    "为 tunny 提交的一次 PR"
subtitle: "每天进步一点点"
date:		2021-06-12T11:48:30
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/06/12/pr/tunny"
categories: [
	"Go"
]
---

## 背景

上周我写了一篇文章[Go 每日一库之 ants](https://darjun.github.io/2021/06/03/godailylib/ants)，深入剖析了`ants`这个 goroutine 池的实现。在反复阅读了多遍[panjf2000](https://github.com/panjf2000)关于`ants`的起源的文章——[GMP 并发调度器深度解析之手撸一个高性能 goroutine pool](https://strikefreedom.top/high-performance-implementation-of-goroutine-pool)，我感觉收获满满。这篇文章对于理解 Go 的 goroutine 并发机制有很大的参考价值，强烈建议一读。然后我花了几个小时时间详细阅读了`ants`的源码，代码写的很棒，非常优美。而后我写了一遍文章分析了`ants`的源码，见[`ants`源码赏析](https://darjun.github.io/2021/06/04/godailylib/ants-src/)。在写介绍`ants`的文章和深入阅读源码过程中，网上的资料提及 Go 语言中 goroutine 池的实现，时常会带上`tunny`这个库。于是，我又去研究了`tunny`的源码，产出一篇文章[Go 每日一库之 tunny](https://darjun.github.io/2021/06/10/godailylib/tunny/)。

在阅读`tunny`源码时，我发现有个方法的实现有些问题。我也在[Go 每日一库之 tunny](https://darjun.github.io/2021/06/10/godailylib/tunny/)中也指出了：

![](/img/in-post/pr/tunny1.png#center)

## 原理

我们知道 slice 结构中有一个指向数组的指针。假设一开始`tunny`中有 5 个 worker，示意图如下：

![](/img/in-post/pr/tunny2.png#center)

数组中每个元素都是一个指针，指向一个`worker`结构。然后我们使用`s := s[:4]`缩容了，进变成了下面这样：

![](/img/in-post/pr/tunny3.png#center)

现在最后一个元素无法通过切片访问了，但是又被底层数组引用着，无法被 Go 运行时的 gc 清理掉，360 软件管家都不行。这就有内存泄漏了。虽然这个泄漏并不严重，一是因为量不可能很大，因为 worker 数量有限，二是下次扩容后位置被覆盖就可以释放原 worker 了（因为没有引用它的指针了）。

`ants`源码中也有类似的操作，但是会将缩容掉的元素置为`nil`。例如`worker_stack.go`中的`reset()`方法：

```golang
func (wq *workerStack) reset() {
  for i := 0; i < wq.len(); i++ {
    wq.items[i].task <- nil
    wq.items[i] = nil
  }
  wq.items = wq.items[:0]
}
```

## PR

确定了问题。我先 fork 了`tunny`的仓库。点击 fork 按钮：

![](/img/in-post/pr/tunny4.png#center)

而后将 fork 后的我自己的仓库下载：

```cmd
$ git clone git@github.com:darjun/tunny.git
```

修改代码，commit，push 到我的远程仓库。下面是我所做的改动：

![](/img/in-post/pr/tunny5.png#center)

然后去`tunny`源仓库，点击`Pull Request`按钮。然后点击右侧的`New pull request`按钮：

![](/img/in-post/pr/tunny6.png#center)

新建一个 PR，然后点击`compare across forks`，选择我的 fork：

![](/img/in-post/pr/tunny7.png#center)

选择我做修改的那次提交，然后填写信息，提交，由于我已经提交了 PR。这次比较就没有差异了。

然后等待作者合并。一天后，作者合并了这次修改：

![](/img/in-post/pr/tunny8.png#center)

## 总结

GitHub 提交 PR 并不难，大到新增特性，小到一个拼写错误，都可以提 PR。相信不少人都听说过，Linus Torvalds 亲自合并了一个来自 11 岁小孩提交的关于 Linux 源码注释中拼写错误的 PR。

GitHub 上 star 很多的项目并非完美，肯定也有很多值得改进的地方。所以阅读源码时需要带有一定的怀疑精神，不要把什么奉为圭臬。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. tunny GitHub：[github.com/Jeffail/tunny](https://github.com/Jeffail/tunny)
2. ants GitHub：[github.com/panjf2000/ants](https://github.com/panjf2000/ants)
3. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~