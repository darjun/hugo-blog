---
layout:    post
title:    "Go 每日一库之 ants"
subtitle: 	"每天学习一个 Go 库"
date:		2021-06-03T07:16:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/06/03/godailylib/ants"
categories: [
	"Go"
]
---

## 简介

处理大量并发是 Go 语言的一大优势。语言内置了方便的并发语法，可以非常方便的创建很多个轻量级的 goroutine 并发处理任务。相比于创建多个线程，goroutine 更轻量、资源占用更少、切换速度更快、无线程上下文切换开销更少。但是受限于资源总量，系统中能够创建的 goroutine 数量也是受限的。默认每个 goroutine 占用 8KB 内存，一台 8GB 内存的机器满打满算也只能创建 8GB/8KB = 1000000 个 goroutine，更何况系统还需要保留一部分内存运行日常管理任务，go 运行时需要内存运行 gc、处理 goroutine 切换等。使用的内存超过机器内存容量，系统会使用交换区（swap），导致性能急速下降。我们可以简单验证一下创建过多 goroutine 会发生什么：

```golang
func main() {
  var wg sync.WaitGroup
  wg.Add(10000000)
  for i := 0; i < 10000000; i++ {
    go func() {
      time.Sleep(1 * time.Minute)
    }()
  }
  wg.Wait()
}
```

在我的机器上（8G内存）运行上面的程序会报`errno 1455`，即`Out of Memory`错误，这很好理解。**谨慎运行**。

另一方面，goroutine 的管理也是一个问题。goroutine 只能自己运行结束，外部没有任何手段可以强制j结束一个 goroutine。如果一个 goroutine 因为某种原因没有自行结束，就会出现 goroutine 泄露。此外，频繁创建 goroutine 也是一个开销。

鉴于上述原因，自然出现了与线程池一样的需求，即 goroutine 池。一般的 goroutine 池自动管理 goroutine 的生命周期，可以按需创建，动态缩容。向 goroutine 池提交一个任务，goroutine 池会自动安排某个 goroutine 来处理。

[`ants`](https://github.com/panjf2000/ants)就是其中一个实现 goroutine 池的库。

## 快速使用

本文代码使用 Go Modules。

创建目录并初始化：

```cmd
$ mkdir ants && cd ants
$ go mod init github.com/darjun/go-daily-lib/ants
```

安装`ants`库，使用`v2`版本：

```cmd
$ go get -u github.com/panjf2000/ants/v2
```

我们接下来要实现一个计算大量整数和的程序。首先创建基础的任务结构，并实现其执行任务方法：

```golang
type Task struct {
  index int
  nums  []int
  sum   int
  wg    *sync.WaitGroup
}

func (t *Task) Do() {
  for _, num := range t.nums {
    t.sum += num
  }

  t.wg.Done()
}
```

很简单，就是将一个切片中的所有整数相加。

然后我们创建 goroutine 池，注意池使用完后需要手动关闭，这里使用`defer`关闭：

```golang
p, _ := ants.NewPoolWithFunc(10, taskFunc)
defer p.Release()

func taskFunc(data interface{}) {
  task := data.(*Task)
  task.Do()
  fmt.Printf("task:%d sum:%d\n", task.index, task.sum)
}
```

上面调用了`ants.NewPoolWithFunc()`创建了一个 goroutine 池。第一个参数是池容量，即池中最多有 10 个 goroutine。第二个参数为每次执行任务的函数。当我们调用`p.Invoke(data)`的时候，`ants`池会在其管理的 goroutine 中找出一个空闲的，让它执行函数`taskFunc`，并将`data`作为参数。

接着，我们模拟数据，做数据切分，生成任务，交给 `ants` 处理：

```golang
const (
  DataSize    = 10000
  DataPerTask = 100
)

nums := make([]int, DataSize, DataSize)
for i := range nums {
  nums[i] = rand.Intn(1000)
}

var wg sync.WaitGroup
wg.Add(DataSize / DataPerTask)
tasks := make([]*Task, 0, DataSize/DataPerTask)
for i := 0; i < DataSize/DataPerTask; i++ {
  task := &Task{
    index: i + 1,
    nums:  nums[i*DataPerTask : (i+1)*DataPerTask],
    wg:    &wg,
  }

  tasks = append(tasks, task)
  p.Invoke(task)
}

wg.Wait()
fmt.Printf("running goroutines: %d\n", ants.Running())
```

随机生成 10000 个整数，将这些整数分为 100 份，每份 100 个，生成`Task`结构，调用`p.Invoke(task)`处理。`wg.Wait()`等待处理完成，然后输出`ants`正在运行的 goroutine 数量，这时应该是 0。

最后我们将结果汇总，并验证一下结果，与直接相加得到的结果做一个比较：

```golang
var sum int
for _, task := range tasks {
  sum += task.sum
}

var expect int
for _, num := range nums {
  expect += num
}

fmt.Printf("finish all tasks, result is %d expect:%d\n", sum, expect)
```

运行：

```cmd
$ go run main.go
...
task:96 sum:53275
task:88 sum:50090
task:62 sum:57114
task:45 sum:48041
task:82 sum:45269
running goroutines: 0
finish all tasks, result is 5010172 expect:5010172
```

确实，任务完成之后，正在运行的 goroutine 数量变为 0。而且我们验证了，结果没有偏差。另外需要注意，**goroutine 池中任务的执行顺序是随机的，与提交任务的先后没有关系**。由上面运行打印的任务标识我们也能发现这一点。

## 函数作为任务

`ants`支持将一个不接受任何参数的函数作为任务提交给 goroutine 运行。由于不接受参数，我们提交的函数要么不需要外部数据，只需要处理自身逻辑，否则就必须用某种方式将需要的数据传递进去，例如闭包。

提交函数作为任务的 goroutine 池使用`ants.NewPool()`创建，它只接受一个参数表示池子的容量。调用池子对象的`Submit()`方法来提交任务，将一个不接受任何参数的函数传入。

最开始的例子可以改写一下。增加一个任务包装函数，将任务需要的参数作为包装函数的参数。包装函数返回实际的任务函数，该任务函数就可以通过闭包访问它需要的数据了：

```golang
type taskFunc func()

func taskFuncWrapper(nums []int, i int, sum *int, wg *sync.WaitGroup) taskFunc {
  return func() {
    for _, num := range nums[i*DataPerTask : (i+1)*DataPerTask] {
      *sum += num
    }

    fmt.Printf("task:%d sum:%d\n", i+1, *sum)
    wg.Done()
  }
}
```

调用`ants.NewPool(10)`创建 goroutine 池，同样池子用完需要释放，这里使用`defer`：

```golang
p, _ := ants.NewPool(10)
defer p.Release()
```

生成模拟数据，切分任务。提交任务给`ants`池执行，这里使用`taskFuncWrapper()`包装函数生成具体的任务，然后调用`p.Submit()`提交：

```golang
nums := make([]int, DataSize, DataSize)
for i := range nums {
  nums[i] = rand.Intn(1000)
}

var wg sync.WaitGroup
wg.Add(DataSize / DataPerTask)
partSums := make([]int, DataSize/DataPerTask, DataSize/DataPerTask)
for i := 0; i < DataSize/DataPerTask; i++ {
  p.Submit(taskFuncWrapper(nums, i, &partSums[i], &wg))
}
wg.Wait()
```

汇总结果，验证：

```golang
var sum int
for _, partSum := range partSums {
  sum += partSum
}

var expect int
for _, num := range nums {
  expect += num
}
fmt.Printf("running goroutines: %d\n", ants.Running())
fmt.Printf("finish all tasks, result is %d expect is %d\n", sum, expect)
```

这个程序的功能与最开始的完全相同。

## 执行流程

GitHub 仓库中有个执行流程图，我重新绘制了一下：

![](/img/in-post/godailylib/ants1.png#center)

执行流程如下：

* 初始化 goroutine 池；
* 提交任务给 goroutine 池，检查是否有空闲的 goroutine：
    * 有，获取空闲 goroutine
    * 无，检查池中的 goroutine 数量是否已到池容量上限：
        * 已到上限，检查 goroutine 池是否是非阻塞的：
            * 非阻塞，直接返回`nil`表示执行失败
            * 阻塞，等待 goroutine 空闲
        * 未到上限，创建一个新的 goroutine 处理任务
* 任务处理完成，将 goroutine 交还给池，以待处理下一个任务

## 选项

`ants`提供了一些选项可以定制 goroutine 池的行为。选项使用`Options`结构定义：

```golang
// src/github.com/panjf2000/ants/options.go
type Options struct {
  ExpiryDuration time.Duration
  PreAlloc bool
  MaxBlockingTasks int
  Nonblocking bool
  PanicHandler func(interface{})
  Logger Logger
}
```

各个选项含义如下：

* `ExpiryDuration`：过期时间。表示 goroutine 空闲多长时间之后会被`ants`池回收
* `PreAlloc`：预分配。调用`NewPool()/NewPoolWithFunc()`之后预分配`worker`（管理一个工作 goroutine 的结构体）切片。而且使用预分配与否会直接影响池中管理`worker`的结构。见下面源码
* `MaxBlockingTasks`：最大阻塞任务数量。即池中 goroutine 数量已到池容量，且所有 goroutine 都处理繁忙状态，这时到来的任务会在阻塞列表等待。这个选项设置的是列表的最大长度。阻塞的任务数量达到这个值后，后续任务提交直接返回失败
* `Nonblocking`：池是否阻塞，默认阻塞。提交任务时，如果`ants`池中 goroutine 已到上限且全部繁忙，阻塞的池会将任务添加的阻塞列表等待（当然受限于阻塞列表长度，见上一个选项）。非阻塞的池直接返回失败
* `PanicHandler`：panic 处理。遇到 panic 会调用这里设置的处理函数
* `Logger`：指定日志记录器

`NewPool()`部分源码：

```golang
if p.options.PreAlloc {
  if size == -1 {
    return nil, ErrInvalidPreAllocSize
  }
  p.workers = newWorkerArray(loopQueueType, size)
} else {
  p.workers = newWorkerArray(stackType, 0)
}
```

使用预分配时，创建`loopQueueType`类型的结构，反之创建`stackType`类型。这是`ants`定义的两种管理`worker`的数据结构。

`ants`定义了一些`With*`函数来设置这些选项：

```golang
func WithOptions(options Options) Option {
  return func(opts *Options) {
    *opts = options
  }
}

func WithExpiryDuration(expiryDuration time.Duration) Option {
  return func(opts *Options) {
    opts.ExpiryDuration = expiryDuration
  }
}

func WithPreAlloc(preAlloc bool) Option {
  return func(opts *Options) {
    opts.PreAlloc = preAlloc
  }
}

func WithMaxBlockingTasks(maxBlockingTasks int) Option {
  return func(opts *Options) {
    opts.MaxBlockingTasks = maxBlockingTasks
  }
}

func WithNonblocking(nonblocking bool) Option {
  return func(opts *Options) {
    opts.Nonblocking = nonblocking
  }
}

func WithPanicHandler(panicHandler func(interface{})) Option {
  return func(opts *Options) {
    opts.PanicHandler = panicHandler
  }
}

func WithLogger(logger Logger) Option {
  return func(opts *Options) {
    opts.Logger = logger
  }
}
```

这里使用了 Go 语言中非常常见的一种模式，我称之为选项模式，非常方便地构造有大量参数，且大部分有默认值或一般不需要显式设置的对象。

我们来验证几个选项。

### 最大等待队列长度

`ants`池设置容量之后，如果所有的 goroutine 都在处理任务。这时提交的任务默认会进入等待队列，`WithMaxBlockingTasks(maxBlockingTasks int)`可以设置等待队列的最大长度。超过这个长度，提交任务直接返回错误：

```golang
func wrapper(i int, wg *sync.WaitGroup) func() {
  return func() {
    fmt.Printf("hello from task:%d\n", i)
    time.Sleep(1 * time.Second)
    wg.Done()
  }
}

func main() {
  p, _ := ants.NewPool(4, ants.WithMaxBlockingTasks(2))
  defer p.Release()

  var wg sync.WaitGroup
  wg.Add(8)
  for i := 1; i <= 8; i++ {
    go func(i int) {
      err := p.Submit(wrapper(i, &wg))
      if err != nil {
        fmt.Printf("task:%d err:%v\n", i, err)
        wg.Done()
      }
    }(i)
  }

  wg.Wait()
}
```

上面代码中，我们设置 goroutine 池的容量为 4，最大阻塞队列长度为 2。然后一个 for 提交 8 个任务，期望结果是：4 个任务在执行，2 个任务在等待，2 个任务提交失败。运行结果：

```cmd
hello from task:8
hello from task:5
hello from task:4
hello from task:6
task:7 err:too many goroutines blocked on submit or Nonblocking is set
task:3 err:too many goroutines blocked on submit or Nonblocking is set
hello from task:1
hello from task:2
```

我们看到提交任务失败，打印`too many goroutines blocked ...`。

代码中有 4 点需要注意：

* **提交任务必须并行进行。如果是串行提交，第 5 个任务提交时由于池中没有空闲的 goroutine 处理该任务，`Submit()`方法会被阻塞，后续任务就都不能提交了。也就达不到验证的目的了**
* **由于任务可能提交失败，失败的任务不会实际执行，所以实际上`wg.Done()`次数会小于 8。因而在`err != nil`分支中我们需要调用一次`wg.Done()`。否则`wg.Wait()`会永远阻塞**
* **为了避免任务执行过快，空出了 goroutine，观察不到现象，每个任务中我使用`time.Sleep(1 * time.Second)`休眠 1s**
* **由于 goroutine 之间的执行顺序未显式同步，故每次执行的顺序不确定**

由于简单起见，前面的例子中`Submit()`方法的返回值都被我们忽略了。实际开发中一定不要忽略。

### 非阻塞

`ants`池默认是阻塞的，我们可以使用`WithNonblocking(nonblocking bool)`设置其为非阻塞。非阻塞的`ants`池中，在所有 goroutine 都在处理任务时，提交新任务会直接返回错误：

```golang
func main() {
  p, _ := ants.NewPool(2, ants.WithNonblocking(true))
  defer p.Release()

  var wg sync.WaitGroup
  wg.Add(3)
  for i := 1; i <= 3; i++ {
    err := p.Submit(wrapper(i, &wg))
    if err != nil {
      fmt.Printf("task:%d err:%v\n", i, err)
      wg.Done()
    }
  }

  wg.Wait()
}
```

使用上个例子中的`wrapper()`函数，`ants`池容量设置为 2。连续提交 3 个任务，期望结果前两个任务正常执行，第 3 个任务提交时返回错误：

```cmd
hello from task:2
task:3 err:too many goroutines blocked on submit or Nonblocking is set
hello from task:1
```

### panic 处理器

一个鲁棒性强的库一定不会忽视错误的处理，特别是宕机相关的错误。在 Go 语言中就是 panic，也被称为运行时恐慌，在程序运行的过程中产生的严重性错误，例如索引越界，空指针解引用等，都会触发 panic。如果不处理 panic，程序会直接意外退出，可能造成数据丢失的严重后果。

`ants`中如果 goroutine 在执行任务时发生`panic`，会终止当前任务的执行，将发生错误的堆栈输出到`os.Stderr`。**注意，该 goroutine 还是会被放回池中，下次可以取出执行新的任务**。

```golang
func wrapper(i int, wg *sync.WaitGroup) func() {
  return func() {
    fmt.Printf("hello from task:%d\n", i)
    if i%2 == 0 {
      panic(fmt.Sprintf("panic from task:%d", i))
    }
    wg.Done()
  }
}

func main() {
  p, _ := ants.NewPool(2)
  defer p.Release()

  var wg sync.WaitGroup
  wg.Add(3)
  for i := 1; i <= 2; i++ {
    p.Submit(wrapper(i, &wg))
  }

  time.Sleep(1 * time.Second)
  p.Submit(wrapper(3, &wg))
  p.Submit(wrapper(5, &wg))
  wg.Wait()
}
```

我们让偶数个任务触发`panic`。提交两个任务，第二个任务一定会触发`panic`。触发`panic`之后，我们还可以继续提交任务 3、5。注意这里没有 4，提交任务 4 还是会触发`panic`。

上面的程序需要注意 2 点：

* **任务函数中`wg.Done()`是在`panic`方法之后，如果触发了`panic`，函数中的其他正常逻辑就不会再继续执行了。所以我们虽然`wg.Add(3)`，但是一共提交了 4 个任务，其中一个任务触发了`panic`，`wg.Done()`没有正确执行**。实际开发中，我们一般使用`defer`语句来确保`wg.Done()`一定会执行
* **在 for 循环之后，我添加了一行代码`time.Sleep(1 * time.Second)`。如果没有这一行，后续的两条`Submit()`方法可以直接执行，可能会导致任务很快就完成了，`wg.Wait()`直接返回了，这时`panic`的堆栈还没有输出**。你可以尝试注释掉这行代码运行看看结果

除了`ants`提供的默认 panic 处理器，我们还可以使用`WithPanicHandler(paincHandler func(interface{}))`指定我们自己编写的 panic 处理器。处理器的参数就是传给`panic`的值：

```golang
func panicHandler(err interface{}) {
  fmt.Fprintln(os.Stderr, err)
}

p, _ := ants.NewPool(2, ants.WithPanicHandler(panicHandler))
defer p.Release()
```

其余代码与上面的完全相同，指定了`panicHandler`后触发`panic`就会执行它。运行：

```cmd
hello from task:2
panic from task:2
hello from task:1
hello from task:5
hello from task:3
```

看到输出了传给`panic`函数的字符串（第二行输出）。

## 默认池

为了方便使用，很多 Go 库都喜欢提供其核心功能类型的一个默认实现。可以直接通过库提供的接口调用。例如`net/http`，例如`ants`。`ants`库中定义了一个默认的池，默认容量为`MaxInt32`。goroutine 池的各个方法都可以直接通过`ants`包直接访问：

```golang
// src/github.com/panjf2000/ants/ants.go
defaultAntsPool, _ = NewPool(DefaultAntsPoolSize)

func Submit(task func()) error {
  return defaultAntsPool.Submit(task)
}

func Running() int {
  return defaultAntsPool.Running()
}

func Cap() int {
  return defaultAntsPool.Cap()
}

func Free() int {
  return defaultAntsPool.Free()
}

func Release() {
  defaultAntsPool.Release()
}

func Reboot() {
  defaultAntsPool.Reboot()
}
```

直接使用：

```golang
func main() {
  defer ants.Release()

  var wg sync.WaitGroup
  wg.Add(2)
  for i := 1; i <= 2; i++ {
    ants.Submit(wrapper(i, &wg))
  }
  wg.Wait()
}
```

默认池也需要`Release()`。

## 总结

本文介绍了 goroutine 池的由来，并借由`ants`库介绍了基本的使用方法，和一些细节。`ants`源码不多，去掉测试的核心代码只有 1k 行左右，建议有时间、感兴趣的童鞋深入阅读。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. ants GitHub：[github.com/valyala/ants](https://github.com/panjf2000/ants)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~