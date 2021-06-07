---
layout:    post
title:    "Go 每日一库之 ants（源码赏析）"
subtitle: 	"每天学习一个 Go 库"
date:		2021-06-04T07:26:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2021/06/04/godailylib/ants-src"
categories: [
	"Go"
]
---

## 简介

继上一篇[Go 每日一库之 ants](https://darjun.github.io/2021/06/03/godailylib/ants)，这篇文章我们来一起看看`ants`的源码。

## `Pool`

通过上篇文章，我们知道`ants`池有两种创建方式：

* `p, _ := ants.NewPool(cap)`：这种方式创建的池子对象需要调用`p.Submit(task)`提交任务，任务是一个无参数无返回值的函数；
* `p, _ := ants.NewPoolWithFunc(cap, func(interface{}))`：这种方式创建的池子对象需要指定池函数，并且使用`p.Invoke(arg)`调用池函数。`arg`就是传给池函数`func(interface{})`的参数。

在`ants`中这两种池子使用不同的结构来表示：`ants.Pool`和`ants.PoolWithFunc`。我们先来介绍`Pool`。`PoolWithFunc`结构也是类似的，介绍完`Pool`之后，我们再简单比较一下它们。

`Pool`结构定义在文件`pool.go`中：

```golang
// src/github.com/panjf2000/ants/pool.go
type Pool struct {
  capacity int32
  running int32
  workers workerArray
  state int32
  lock sync.Locker
  cond *sync.Cond
  workerCache sync.Pool
  blockingNum int
  options *Options
}
```

各个字段含义如下：

* `capacity`：池容量，表示`ants`最多能创建的 goroutine 数量。如果为负数，表示容量无限制；
* `running`：已经创建的 worker goroutine 的数量；
* `workers`：存放一组 worker 对象，`workerArray`只是一个接口，表示一个 worker 容器，后面详述；
* `state`：记录池子当前的状态，是否已关闭（`CLOSED`）；
* `lock`：锁。`ants`自己实现了一个自旋锁。用于同步并发操作；
* `cond`：条件变量。处理任务等待和唤醒；
* `workerCache`：使用`sync.Pool`对象池管理和创建`worker`对象，提升性能；
* `blockingNum`：阻塞等待的任务数量；
* `options`：选项。上一篇文章已经详细介绍过了。

这里明确一个概念，`ants`中为每个任务都是由 worker 对象来处理的，每个 worker 对象会对应创建一个 goroutine 来处理任务。`ants`中使用`goWorker`表示 worker：

```golang
// src/github.com/panjf2000/ants/worker.go
type goWorker struct {
  pool *Pool
  task chan func()
  recycleTime time.Time
}
```

后文详细介绍这一块内容，现在我们只需要知道`Pool.workers`字段就是存放`goWorker`对象的容器。

### `Pool`创建

创建`Pool`对象需调用`ants.NewPool(size, options)`函数。省略了一些处理选项的代码，最终代码如下：

```golang
// src/github.com/panjf2000/ants/pool.go
func NewPool(size int, options ...Option) (*Pool, error) {
  // ...
  p := &Pool{
    capacity: int32(size),
    lock:     internal.NewSpinLock(),
    options:  opts,
  }
  p.workerCache.New = func() interface{} {
    return &goWorker{
      pool: p,
      task: make(chan func(), workerChanCap),
    }
  }
  if p.options.PreAlloc {
    if size == -1 {
      return nil, ErrInvalidPreAllocSize
    }
    p.workers = newWorkerArray(loopQueueType, size)
  } else {
    p.workers = newWorkerArray(stackType, 0)
  }

  p.cond = sync.NewCond(p.lock)

  go p.purgePeriodically()
  return p, nil
}
```

代码不难理解：

* 创建`Pool`对象，设置容量，创建一个自旋锁来初始化`lock`字段，设置选项；
* 设置`workerCache`这个`sync.Pool`对象的`New`方法，在调用`sync.Pool`对象的`Get()`方法时，如果它没有缓存的 worker 对象了，则调用这个方法创建一个；
* 根据是否设置了预分配选项，创建不同类型的 workers；
* 使用`p.lock`锁创建一个条件变量；
* 最后启动一个 goroutine 用于定期清理过期的 worker。

`Pool.workers`字段为`workerArray`类型，这实际上是一个接口，表示一个 worker 容器：

```golang
type workerArray interface {
  len() int
  isEmpty() bool
  insert(worker *goWorker) error
  detach() *goWorker
  retrieveExpiry(duration time.Duration) []*goWorker
  reset()
}
```

每个方法从名字上很好理解含义：

* `len() int`：worker 数量；
* `isEmpty() bool`：worker 数量是否为 0；
* `insert(worker *goWorker) error`：goroutine 任务执行结束后，将相应的 worker 放回`workerArray`中；
* `detach() *goWorker`：从`workerArray`中取出一个 worker；
* `retrieveExpiry(duration time.Duration) []*goWorker`：取出所有的过期 worker；
* `reset()`：重置容器。

`workerArray`在`ants`中有两种实现，即`workerStack`和`loopQueue`。

#### `workerStack`

我们先来介绍一下`workerStack`，它位于文件`worker_stack.go`中：

```golang
// src/github.com/panjf2000/ants/worker_stack.go
type workerStack struct {
  items  []*goWorker
  expiry []*goWorker
  size   int
}

func newWorkerStack(size int) *workerStack {
  return &workerStack{
    items: make([]*goWorker, 0, size),
    size:  size,
  }
}
```

* `items`：空闲的`worker`；
* `expiry`：过期的`worker`。

goroutine 完成任务之后，`Pool`池会将相应的 worker 放回`workerStack`，调用`workerStack.insert()`直接`append`到`items`中即可：

```golang
func (wq *workerStack) insert(worker *goWorker) error {
  wq.items = append(wq.items, worker)
  return nil
}
```

新任务到来时，会调用`workerStack.detach()`从容器中取出一个空闲的 worker：

```golang
func (wq *workerStack) detach() *goWorker {
  l := wq.len()
  if l == 0 {
    return nil
  }

  w := wq.items[l-1]
  wq.items[l-1] = nil // avoid memory leaks
  wq.items = wq.items[:l-1]

  return w
}
```

这里总是返回最后一个 worker，每次`insert()`也是`append`到最后，符合栈后进先出的特点，故称为`workerStack`。

这里有一个细节，由于切片的底层结构是数组，只要有引用数组的指针，数组中的元素就不会释放。这里取出切片最后一个元素后，将对应数组元素的指针设置为`nil`，主动释放这个引用。

上面说过新建`Pool`对象时会创建一个 goroutine 定期检查和清理过期的 worker。通过调用`workerArray.retrieveExpiry()`获取过期的 worker 列表。`workerStack`实现如下：

```golang
func (wq *workerStack) retrieveExpiry(duration time.Duration) []*goWorker {
  n := wq.len()
  if n == 0 {
    return nil
  }

  expiryTime := time.Now().Add(-duration)
  index := wq.binarySearch(0, n-1, expiryTime)

  wq.expiry = wq.expiry[:0]
  if index != -1 {
    wq.expiry = append(wq.expiry, wq.items[:index+1]...)
    m := copy(wq.items, wq.items[index+1:])
    for i := m; i < n; i++ {
      wq.items[i] = nil
    }
    wq.items = wq.items[:m]
  }
  return wq.expiry
}
```

实现使用二分查找法找到已过期的最近一个 worker。由于过期时间是按照 goroutine 执行任务后的空闲时间计算的，而`workerStack.insert()`入队顺序决定了，它们的过期时间是从早到晚的。所以可以使用二分查找：

```golang
func (wq *workerStack) binarySearch(l, r int, expiryTime time.Time) int {
  var mid int
  for l <= r {
    mid = (l + r) / 2
    if expiryTime.Before(wq.items[mid].recycleTime) {
      r = mid - 1
    } else {
      l = mid + 1
    }
  }
  return r
}
```

二分查找的是最近过期的 worker，即将过期的 worker 的前一个。它和在它之前的 worker 已经全部过期了。

如果找到索引`index`，将`items`从开头到`index`（包括）的所有 worker 复制到`expiry`字段中。然后将`index`之后的所有未过期 worker 复制到切片头部，这里使用了`copy`函数。`copy`返回实际复制的数量，即未过期的 worker 数量`m`。然后将切片`items`从`m`开始所有的元素置为`nil`，避免内存泄漏，因为它们已经被复制到头部了。最后裁剪`items`切片，返回过期 worker 切片。

### `loopQueue`

`loopQueue`实现基于循环队列，结构定义在文件`worker_loop_queue`中：

```golang
type loopQueue struct {
  items  []*goWorker
  expiry []*goWorker
  head   int
  tail   int
  size   int
  isFull bool
}

func newWorkerLoopQueue(size int) *loopQueue {
  return &loopQueue{
    items: make([]*goWorker, size),
    size:  size,
  }
}
```

由于是循环队列，这里先创建好了一个长度为`size`的切片。循环队列有一个队列头指针`head`，指向第一个有元素的位置，一个队列尾指针`tail`，指向下一个可以存放元素的位置。所以一开始状态如下：

![](/img/in-post/godailylib/ants-src1.png#center)

在`tail`处添加元素，添加后`tail`指针后移。在`head`处取出元素，取出后`head`指针也后移。进行一段时间操作后，队列状态如下：

![](/img/in-post/godailylib/ants-src2.png#center)

`head`或`tail`指针到队列尾了，需要回绕。所以可能出现这种情况：

![](/img/in-post/godailylib/ants-src3.png#center)

当`tail`指针赶上`head`指针了，说明队列就满了：

![](/img/in-post/godailylib/ants-src4.png#center)

当`head`指针赶上`tail`指针了，队列再次为空：

![](/img/in-post/godailylib/ants-src5.png#center)

根据示意图，我们再来看`loopQueue`的操作方法就很简单了。

由于`head`和`tail`相等的情况有可能是队列空，也有可能是队列满，所以`loopQueue`中增加一个`isFull`字段以示区分。goroutine 完成任务之后，会将对应的 worker 对象放回`loopQueue`，执行的是`insert()`方法：

```golang
func (wq *loopQueue) insert(worker *goWorker) error {
  if wq.size == 0 {
    return errQueueIsReleased
  }

  if wq.isFull {
    return errQueueIsFull
  }
  wq.items[wq.tail] = worker
  wq.tail++

  if wq.tail == wq.size {
    wq.tail = 0
  }
  if wq.tail == wq.head {
    wq.isFull = true
  }

  return nil
}
```

这个方法执行的就是循环队列的入队流程，注意如果插入后`tail==head`了，说明队列满了，设置`isFull`字段。

新任务到来调用`loopQueeue.detach()`方法获取一个空闲的 worker 结构：

```golang
func (wq *loopQueue) detach() *goWorker {
  if wq.isEmpty() {
    return nil
  }

  w := wq.items[wq.head]
  wq.items[wq.head] = nil
  wq.head++
  if wq.head == wq.size {
    wq.head = 0
  }
  wq.isFull = false

  return w
}
```

这个方法对应的是循环队列的出队流程，注意每次出队后，队列肯定不满了，`isFull`要重置为`false`。

与`workerStack`结构一样，先入的 worker 对象过期时间早，后入的晚，获取过期 worker 的方法与`workerStack`中类似，只是没有使用二分查找了。这里就不赘述了。

### 再看`Pool`创建

介绍完两种`workerArray`的实现之后，再来看`Pool`的创建函数中`workers`字段的设置：

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

`newWorkerArray()`定义在文件`worker_array.go`中：

```golang
type arrayType int

const (
  stackType arrayType = 1 << iota
  loopQueueType
)

func newWorkerArray(aType arrayType, size int) workerArray {
  switch aType {
  case stackType:
    return newWorkerStack(size)
  case loopQueueType:
    return newWorkerLoopQueue(size)
  default:
    return newWorkerStack(size)
  }
}
```

即如果设置了预分配选项，就采用`loopQueue`结构。否则就采用`stack`的结构。

## worker 结构

介绍完`Pool`的创建和结构，我们来看看 worker 的结构。在`ants`中 worker 用结构体`goWorker`表示，定义在文件`worker.go`中。它的结构非常简单：

```golang
// src/github.com/panjf2000/ants/worker.go
type goWorker struct {
  pool *Pool
  task chan func()
  recycleTime time.Time
}
```

具体字段含义很明显：

* `pool`：持有 goroutine 池的引用；
* `task`：任务通道，通过这个通道将类型为`func ()`的函数作为任务发送给`goWorker`；
* `recyleTime`：这个字段记录`goWorker`什么时候被放回池中（即什么时候开始空闲）。其完成任务后，在将其放回 goroutine 池的时候设置。

`goWorker`创建时会调用`run()`方法，`run()`方法中启动一个新 goroutine 处理任务。`run()`主体流程非常简单：

```golang
func (w *goWorker) run() {
  go func() {
    for f := range w.task {
      if f == nil {
        return
      }
      f()
      if ok := w.pool.revertWorker(w); !ok {
        return
      }
    }
  }()
}
```

这个方法启动一个新的 goroutine，然后不停地从`task`通道中接收任务，然后执行任务，任务执行完成之后调用池对象的`revertWorker()`方法将该`goWorker`对象放回池中，以便下次取出处理新的任务。`revertWorker()`方法后面会详细分析。

这里注意，实际上`for f := range w.task`这个循环直到通道`task`关闭或取出为`nil`的任务才会终止。所以这个 goroutine 一直在运行，这正是`ants`高性能的关键所在。每个`goWorker`只会启动一次 goroutine， 后续重复利用这个 goroutine。goroutine 每次只执行一个任务就会被放回池中。

还有一个细节，**如果放回操作失败，则会调用`return`，这会让 goroutine 运行结束，防止 goroutine 泄漏**。

这里`f == nil`为 true 时`return`，也是一个细节点，我们后面讲池关闭的时候会详细介绍。

下面我们看看`run()`方法的异常处理：

```golang
defer func() {
  w.pool.workerCache.Put(w)
  if p := recover(); p != nil {
    if ph := w.pool.options.PanicHandler; ph != nil {
      ph(p)
    } else {
      w.pool.options.Logger.Printf("worker exits from a panic: %v\n", p)
      var buf [4096]byte
      n := runtime.Stack(buf[:], false)
      w.pool.options.Logger.Printf("worker exits from panic: %s\n", string(buf[:n]))
    }
  }
  w.pool.cond.Signal()
}()
```

简单来说，就是在`defer`中通过`recover()`函数捕获任务执行过程中抛出的`panic`。这时任务执行失败，goroutine 也结束了。但是`goWorker`对象还是可以重复利用，所以`defer`函数一开始调用`w.pool.workerCache.Put(w)`将`goWorker`对象放回`sync.Pool`池中。

接着就是处理`panic`，如果选项中指定了`panic`处理器，直接调用这个处理器。否则，`ants`调用选项中设置的`Logger`记录一些日志，如堆栈，`panic`信息等。

最后需要调用`w.pool.cond.Signal()`通知现在有空闲的`goWorker`了。因为我们实际运行的`goWorker`数量由于`panic`少了一个，而池中可能有其他任务在等待处理。

## 提交任务

接下来，通过提交任务就可以串起整个流程。由上一篇文章我们知道，可以调用池对象的`Submit()`方法提交任务：

```golang
func (p *Pool) Submit(task func()) error {
  if p.IsClosed() {
    return ErrPoolClosed
  }
  var w *goWorker
  if w = p.retrieveWorker(); w == nil {
    return ErrPoolOverload
  }
  w.task <- task
  return nil
}
```

首先判断池是否已关闭，然后调用`retrieveWorker()`方法获取一个空闲的 worker，然后将任务`task`发送到 worker 的任务通道。下面是`retrieveWorker()`实现：

```golang
func (p *Pool) retrieveWorker() (w *goWorker) {
  p.lock.Lock()

  w = p.workers.detach()
  if w != nil {
    p.lock.Unlock()
  } else if capacity := p.Cap(); capacity == -1 || capacity > p.Running() {
    p.lock.Unlock()
    spawnWorker()
  } else {
    if p.options.Nonblocking {
      p.lock.Unlock()
      return
    }
  Reentry:
    if p.options.MaxBlockingTasks != 0 && p.blockingNum >= p.options.MaxBlockingTasks {
      p.lock.Unlock()
      return
    }
    p.blockingNum++
    p.cond.Wait()
    p.blockingNum--
    var nw int
    if nw = p.Running(); nw == 0 {
      p.lock.Unlock()
      if !p.IsClosed() {
        spawnWorker()
      }
      return
    }
    if w = p.workers.detach(); w == nil {
      if nw < capacity {
        p.lock.Unlock()
        spawnWorker()
        return
      }
      goto Reentry
    }

    p.lock.Unlock()
  }
  return
}
```

这个方法稍微有点复杂，我们一点点来看。首先调用`p.workers.detach()`获取`goWorker`对象。`p.workers`是`loopQueue`或者`workerStack`对象，它们都实现了`detach()`方法，前面已经介绍过了。

如果返回了一个`goWorker`对象，说明有空闲 goroutine，直接返回。

否则，池容量还没用完（即容量大于正在工作的`goWorker`数量），则调用`spawnWorker()`新建一个`goWorker`，执行其`run()`方法：

```golang
spawnWorker := func() {
  w = p.workerCache.Get().(*goWorker)
  w.run()
}
```

否则，池容量已用完。如果设置了**非阻塞**选项，则直接返回。否则，如果设置了最大阻塞队列长度上限，且当前阻塞等待的任务数量已经达到这个上限，直接返回。否则，阻塞等待数量 +1，调用`p.cond.Wait()`等待。

然后`goWorker.run()`完成一个任务后，调用池的`revertWorker()`方法放回`goWorker`：

```golang
func (p *Pool) revertWorker(worker *goWorker) bool {
  if capacity := p.Cap(); (capacity > 0 && p.Running() > capacity) || p.IsClosed() {
    return false
  }
  worker.recycleTime = time.Now()
  p.lock.Lock()

  if p.IsClosed() {
    p.lock.Unlock()
    return false
  }

  err := p.workers.insert(worker)
  if err != nil {
    p.lock.Unlock()
    return false
  }

  p.cond.Signal()
  p.lock.Unlock()
  return true
}
```

这里设置了`goWorker`的`recycleTime`字段，用于判定过期。然后将`goWorker`放回池。`workers`的`insert()`方法前面也已经分析过了。

接着调用`p.cond.Signal()`唤醒之前`retrieveWorker()`方法中的等待。`retrieveWorker()`方法继续执行，阻塞等待数量 -1，这里判断当前`goWorker`的数量（也即 goroutine 数量）。如果数量等于 0，很有可能池子刚刚执行了`Release()`关闭，这时需要判断池是否处于关闭状态，如果是则直接返回。否则，调用`spawnWorker()`创建一个新的`goWorker`并执行其`run()`方法。

如果当前`goWorker`数量不为 0，则调用`p.workers.detach()`取出一个空闲的`goWorker`返回。这个操作有可能失败，因为可能同时有多个 goroutine 在等待，唤醒的时候只有部分 goroutine 能获取到`goWorker`。如果失败了，其容量还未用完，直接创建新的`goWorker`，反之重新执行阻塞等待逻辑。

这里有很多加锁和解锁的逻辑，再加上和信号量混在一起很难看明白。其实只需要知道一点就很简单了，那就是`p.cond.Wait()`内部会将当前 goroutine 挂起，然后解开它持有的锁，即会调用`p.lock.Unlock()`。这也是为什么`revertWorker()`中`p.lock.Lock()`加锁能成功的原因。然后`p.cond.Signal()`或`p.cond.Broadcast()`会唤醒因为`p.cond.Wait()`而挂起的 goroutine，但是需要`Signal()/Broadcast()`所在 goroutine 调用解锁方法。

最后，放上整体流程图：

![](/img/in-post/godailylib/ants1.png#center)

## 清理过期`goWorker`

在`NewPool()`函数中会启动一个 goroutine 定期清理过期的`goWorker`：

```golang
func (p *Pool) purgePeriodically() {
  heartbeat := time.NewTicker(p.options.ExpiryDuration)
  defer heartbeat.Stop()

  for range heartbeat.C {
    if p.IsClosed() {
      break
    }

    p.lock.Lock()
    expiredWorkers := p.workers.retrieveExpiry(p.options.ExpiryDuration)
    p.lock.Unlock()

    for i := range expiredWorkers {
      expiredWorkers[i].task <- nil
      expiredWorkers[i] = nil
    }

    if p.Running() == 0 {
      p.cond.Broadcast()
    }
  }
}
```

如果池子已关闭，直接退出 goroutine。由选项`ExpiryDuration`来设置清理的间隔，如果没有设置该选项，采用默认值 1s：

```golang
// src/github.com/panjf2000/ants/pool.go
func NewPool(size int, options ...Option) (*Pool, error) {
  if expiry := opts.ExpiryDuration; expiry < 0 {
    return nil, ErrInvalidPoolExpiry
  } else if expiry == 0 {
    opts.ExpiryDuration = DefaultCleanIntervalTime
  }
}

// src/github.com/panjf2000/ants/pool.go
const (
  DefaultCleanIntervalTime = time.Second
)
```

然后就是每个清理周期，调用`p.workers.retrieveExpiry()`方法，取出过期的`goWorker`。**因为由这些`goWorker`启动的 goroutine 还阻塞在通道`task`上，所以要向该通道发送一个`nil`值，而`goWorker.run()`方法中接收到一个值为`nil`的任务会`return`，结束 goroutine，避免了 goroutine 泄漏**。

如果所有`goWorker`都被清理掉了，可能这时还有 goroutine 阻塞在`retrieveWorker()`方法中的`p.cond.Wait()`上，所以这里需要调用`p.cond.Broadcast()`唤醒这些 goroutine。

## 容量动态修改

在运行过程中，可以动态修改池的容量。调用`p.Tune(size int)`方法：

```golang
func (p *Pool) Tune(size int) {
  if capacity := p.Cap(); capacity == -1 || size <= 0 || size == capacity || p.options.PreAlloc {
    return
  }
  atomic.StoreInt32(&p.capacity, int32(size))
}
```

这里只是简单设置了一下新的容量，不影响当前正在执行的`goWorker`，而且如果设置了预分配选项，容量不能再次设置。

下次执行`revertWorker()`的时候就会以新的容量判断是否能放回，下次执行`retrieveWorker()`的时候也会以新容量判断是否能创建新`goWorker`。

## 关闭和重新启动`Pool`

使用完成之后，需要关闭`Pool`，避免 goroutine 泄漏。调用池对象的`Release()`方法关闭：

```golang
func (p *Pool) Release() {
  atomic.StoreInt32(&p.state, CLOSED)
  p.lock.Lock()
  p.workers.reset()
  p.lock.Unlock()
  p.cond.Broadcast()
}
```

调用`p.workers.reset()`结束`loopQueue`或`wokerStack`中的 goroutine，做一些清理工作，同时为了防止有 goroutine 阻塞在`p.cond.Wait()`上，执行一次`p.cond.Broadcast()`。

`workerStack`与`loopQueue`的`reset()`基本相同，即发送`nil`到`task`通道从而结束 goroutine，然后重置各个字段：

```golang
// loopQueue 版本
func (wq *loopQueue) reset() {
  if wq.isEmpty() {
    return
  }

Releasing:
  if w := wq.detach(); w != nil {
    w.task <- nil
    goto Releasing
  }
  wq.items = wq.items[:0]
  wq.size = 0
  wq.head = 0
  wq.tail = 0
}

// stack 版本
func (wq *workerStack) reset() {
  for i := 0; i < wq.len(); i++ {
    wq.items[i].task <- nil
    wq.items[i] = nil
  }
  wq.items = wq.items[:0]
}
```

池关闭后还可以调用`Reboot()`重启：

```golang
func (p *Pool) Reboot() {
  if atomic.CompareAndSwapInt32(&p.state, CLOSED, OPENED) {
    go p.purgePeriodically()
  }
}
```

由于`p.purgePeriodically()`在`p.Release()`之后检测到池关闭就直接退出了，这里需要重新开启一个 goroutine 定期清理。

## `PoolWithFunc`和`WorkWithFunc`

上一篇文章中我们还介绍了另一种方式创建`Pool`，即`NewPoolWithFunc()`，指定一个函数。后面提交任务时调用`p.Invoke()`提供参数就可以执行该函数了。这种方式创建的 Pool 和 Woker 结构如下：

```golang
type PoolWithFunc struct {
  workers []*goWorkerWithFunc
  poolFunc func(interface{})
}

type goWorkerWithFunc struct {
  pool *PoolWithFunc
  args chan interface{}
  recycleTime time.Time
}
```

与前面介绍的`Pool`和`goWorker`大体相似，只是`PoolWithFunc`保存了传入的函数对象，使用数组保存 worker。`goWorkerWithFunc`以`interface{}`为`args`通道的数据类型，其实也好理解，因为已经有函数了，只需要传入数据作为参数就可以运行了：

```golang
func (w *goWorkerWithFunc) run() {
  go func() {
    for args := range w.args {
      if args == nil {
        return
      }
      w.pool.poolFunc(args)
      if ok := w.pool.revertWorker(w); !ok {
        return
      }
    }
  }()
}
```

从通道接收函数参数，执行池中保存的函数对象。

## 其他细节

### `task`缓冲通道

还记得创建`p.workerCache`这个`sync.Pool`对象的代码么：

```golang
p.workerCache.New = func() interface{} {
  return &goWorker{
    pool: p,
    task: make(chan func(), workerChanCap),
  }
}
```

在`sync.Pool`中没有`goWorker`对象时，调用`New()`方法创建一个，注意到这里创建的`task`通道使用`workerChanCap`作为容量。这个变量定义在`ants.go`文件中：

```golang
var (
  // workerChanCap determines whether the channel of a worker should be a buffered channel
  // to get the best performance. Inspired by fasthttp at
  // https://github.com/valyala/fasthttp/blob/master/workerpool.go#L139
  workerChanCap = func() int {
    // Use blocking channel if GOMAXPROCS=1.
    // This switches context from sender to receiver immediately,
    // which results in higher performance (under go1.5 at least).
    if runtime.GOMAXPROCS(0) == 1 {
      return 0
    }

    // Use non-blocking workerChan if GOMAXPROCS>1,
    // since otherwise the sender might be dragged down if the receiver is CPU-bound.
    return 1
  }()
)
```

为了方便对照，我把注释也放上来了。`ants`参考了著名的 Web 框架`fasthttp`的实现。当`GOMAXPROCS`为 1 时（即操作系统线程数为 1），向通道`task`发送会挂起发送 goroutine，将执行流程转向接收 goroutine，这能提升接收处理性能。如果`GOMAXPROCS`大于 1，`ants`使用带缓冲的通道，为了防止接收 goroutine 是 CPU 密集的，导致发送 goroutine 被阻塞。下面是`fasthttp`中的相关代码：

```golang
// src/github.com/valyala/fasthttp/workerpool.go
var workerChanCap = func() int {
  // Use blocking workerChan if GOMAXPROCS=1.
  // This immediately switches Serve to WorkerFunc, which results
  // in higher performance (under go1.5 at least).
  if runtime.GOMAXPROCS(0) == 1 {
    return 0
  }

  // Use non-blocking workerChan if GOMAXPROCS>1,
  // since otherwise the Serve caller (Acceptor) may lag accepting
  // new connections if WorkerFunc is CPU-bound.
  return 1
}()
```

### 自旋锁

`ants`利用`atomic.CompareAndSwapUint32()`这个原子操作实现了一个自旋锁。与其他类型的锁不同，自旋锁在加锁失败之后不会立刻进入等待，而是会继续尝试。这对于很快就能获得锁的应用来说能极大提升性能，因为能避免加锁和解锁导致的线程切换：

```golang
type spinLock uint32

func (sl *spinLock) Lock() {
  backoff := 1
  for !atomic.CompareAndSwapUint32((*uint32)(sl), 0, 1) {
    for i := 0; i < backoff; i++ {
      runtime.Gosched()
    }
    backoff <<= 1
  }
}

func (sl *spinLock) Unlock() {
  atomic.StoreUint32((*uint32)(sl), 0)
}

// NewSpinLock instantiates a spin-lock.
func NewSpinLock() sync.Locker {
  return new(spinLock)
}
```

另外这里使用了**指数退避**，先等 1 个循环周期，通过`runtime.Gosched()`告诉运行时切换其他 goroutine 运行。如果还是获取不到锁，就再等 2 个周期。如果还是不行，再等 4，8，16...以此类推。这可以防止短时间内获取不到锁，导致 CPU 时间的浪费。

## 总结

`ants`源码短小精悍，没有引用其他任何第三方库。各种细节处理，各种性能优化的点都是值得我们细细品味的。强烈建议大家读一读源码。阅读优秀的源码，能极大地提高自身的编码素养。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. ants GitHub：[github.com/panjf2000/ants](https://github.com/panjf2000/ants)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~