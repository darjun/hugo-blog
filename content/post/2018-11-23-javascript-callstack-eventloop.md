---
layout:     post
title:      "深入理解Javascript之CallStack&EventLoop"
subtitle:   "\"Javascript in depth\""
date:       2018-11-23T20:57:00 
author:     "darjun"
image: "/img/post-bg-2015.jpg"
tags:
    - 深入理解JavaScript
URL: "/2018/11/23/javascript-callstack-eventloop/"
categories: [ 
  "JavaScript"
]
---

## <span id="概述">1.概述</span>
众所周知，Javascript是一个单线程的语言。这意味着，在Javascript中，同一时间只能做一件事情。

这样的设计有一些优点，例如简单，避免了多线程中复杂的状态同步，写程序时不用考虑并发访问。但同时也带来了一些其他问题，其中比较突出的一个问题是：代码逻辑不直观。由于Javascript是单线程的，其中只有一个执行序列。所以，在执行异步操作（例如定时，网络请求这些不能立即完成的操作）时，Javascript运行时不可能在那里等着操作完成。否则整个运行时都被阻塞在那里了，导致其他所有的操作都无法进行，例如网页渲染，用户点击、滚动页面等操作。这样的用户体验是非常糟糕的。

正因为如此，Javascript使用回调函数处理异步操作结果。进行异步操作时传入一个回调，操作完成之后由Javascript引擎执行这个回调，将结果传入。慢慢地，Javascript中充斥着大量的回调。过多的使用回调让一段完整的逻辑被拆分成了很多片段，非常不利于阅读与维护。回调过多的问题在NodeJS中更为突出，故而出现了`Promise`（见我的前一篇[博客][1]）和`async/await`。

那么异步操作完成时，Javascript运行时是怎样感知到并调用对应的回调函数的呢？

答案是EventLoop（事件循环）。

要了解EventLoop是怎样运作的，我们首先需要了解Javascript是怎样处理一个个任务，调用一个个函数的。这就是CallStack（调用栈）所做的事。

## <span id="CallStack">2.调用栈</span>
相信有过其他语言编程经验的读者都听说过CallStack的概念。Javascript中的CallStack类似。

CallStack是一个栈结构，栈的特点是LIFO（后入先出），出栈入栈只会在一端（也就是栈顶）进行。

CallStack是用来处理函数调用与返回的。每次调用一个函数，Javascript运行时会生成一个新的调用结构压入CallStack。而函数调用结束返回时，JavaScript运行时会将栈顶的调用结构弹出。由于栈的LIFO特性，每次弹出的必然是最新调用的那个函数的结构。

Javascript启动时，从文件或标准输入加载程序。加载完成时，Javascript运行时会生成一个匿名的函数，函数体就是输入的代码。这个函数就有点类似于C/C++中的`main()`函数，是我们的入口函数。我们姑且称之为`<main>`函数。Javascript启动时，首先调用就是`<main>`函数。看下面代码：

```
function func1() {
    console.log('in function1');
}

function func2() {
    func1();
    console.log('in function2');
}

function func3() {
    func2();
    console.log('in function3');
}

func3();
```

上面代码很好理解，我们来看看Javascript是如何运行这段代码的。

Javascript首先加载代码，创建一个匿名`<main>`包裹这段代码并调用该函数。`<main>`函数执行，依次定义函数`func1`、`func2`、`func3`，然后调用函数`func3`。为`func3`创建调用结构并压栈。函数`func3`中调用`func2`，为`func2`创建调用结构并压栈。函数`func2`中调用`func1`，为`func1`创建调用结构并压栈。这个过程中，CallStack的变化如下。

![Call Stack Push](/img/in-post/callstack-eventloop/callstack-push.png)

然后，函数`func1`执行完成，从栈顶弹出调用结构。然后`func2`继续执行，`func2`执行完成后从栈顶弹出其调用结构。然后`func3`继续执行，`func3`执行完成后从栈顶弹出其调用结构。这个过程中，CallStack的变化如下。

![Call Stack Pop](/img/in-post/callstack-eventloop/callstack-pop.png)

当然，我这里有一个地方不太严谨。不知道读者有没有注意到，`console.log`也是函数函数哦。所以在`func1`中调用`console.log`时，CallStack上也会有对应的调用堆栈。`func2`，`func3`中的`console.log`调用同样如此。有兴趣的话，可以自己画一画完整的调用流程，这样可以加深理解😀。

这里我推荐大家使用Google Chrome的开发者工具来帮助我们理解CallStack。下图是上面代码在开发者工具中的一步步执行的结果：

![Call Stack Demo](/img/in-post/callstack-eventloop/callstack-demo.gif)

* Chrome中`<main>`称为`<anonymous>`。

* 单步执行时，重点观察右侧工具栏中**Call Stack**一栏的变化。

在CallStack中执行的函数，我们称之为一个task（任务）。

接下来，我们来思考这样一个问题：`setTimeout`、`setInterval`、`AJAX`请求这些功能是怎么实现的？

一位牛人[Philip Roberts][2]曾经将V8引擎（Chrome内置的Javascript引擎）源码下载下来，然后用`grep`查找，发现源码中并没有实现这些函数的代码😂。

那么这些函数到底是如何实现的，又是如何与Javascript引擎交互的呢？

答案是：宿主（网页中指的是浏览器，Node中指的是Node引擎）提供实现，并在操作完成时将结果（异步的，会有延迟）放入Javascript引擎的task队列，由Javascript引擎处理。

## <span id="EventLoop">3.事件循环</span>

EventLoop顾名思义，其实就是Javascript引擎中的一个循环，它就是一个不停地从任务队列（task queue）中取出任务执行的过程。

我们前面详细了解了CallStack以及Javascript启动时是如何处理的。但是`<main>`退出后，Javascript引擎就没事做了吗？当然不是，有很多任务会不定时的触发需要Javascript引擎去处理。例如，用户点击按钮，定时器，页面渲染等。

其实，Javascript引擎中维护着一个任务队列。当CallStack中没有任务在执行时，引擎会从任务队列中取出任务压入CallStack处理。我们通过代码来具体看看（引用[jesstelford][4]）：

```
setTimeout(() => {
    console.log('hi');
}, 1000);
```

我们的Js代码，call stack，task queue和Web APIs（浏览器中实现）关系如下：

```
        [code]        |   [call stack]    | [task queue] | |    [Web APIs] |
  --------------------|-------------------|--------------| |---------------|
  setTimeout(() => {  |                   |              | |               |
    console.log('hi') |                   |              | |               |
  }, 1000)            |                   |              | |               |
                      |                   |              | |               |
```

开始时，代码未执行，所有都是空的。

```
        [code]        |   [call stack]    | [task queue] | |   [Web APIs]  |
  --------------------|-------------------|--------------| |---------------|
  setTimeout(() => {  | <main>            |              | |               |
    console.log('hi') |                   |              | |               |
  }, 1000)            |                   |              | |               |
                      |                   |              | |               |
```

开始执行代码，压入我们的`<main>`函数。

```
        [code]        |   [call stack]    | [task queue] | |   [Web APIs]  |
  --------------------|-------------------|--------------| |---------------|
> setTimeout(() => {  | <main>            |              | |               |
    console.log('hi') | setTimeout        |              | |               |
  }, 1000)            |                   |              | |               |
                      |                   |              | |               |
```

执行第一行代码，调用函数`setTimeout`。我们前面说过，每个函数调用都会创建一个新的调用记录压到栈上。

```
        [code]        |   [call stack]    | [task queue] | |   [Web APIs]  |
  --------------------|-------------------|--------------| |---------------|
  setTimeout(() => {  | <main>            |              | | timeout, 1000 |
    console.log('hi') |                   |              | |               |
  }, 1000)            |                   |              | |               |
                      |                   |              | |               |
```

`setTimeout`执行完成，从栈中移除对应调用记录。`Web APIs`记录超时和回调，超时机制由浏览器实现。

```
        [code]        |   [call stack]    | [task queue] | |   [Web APIs]  |
  --------------------|-------------------|--------------| |---------------|
  setTimeout(() => {  |                   |              | | timeout, 1000 |
    console.log('hi') |                   |              | |               |
  }, 1000)            |                   |              | |               |
                      |                   |              | |               |
```

代码中没有其他逻辑，`<main>`函数结束，从栈移除调用信息。

```
        [code]        |   [call stack]    | [task queue] | |   [Web APIs]  |
  --------------------|-------------------|--------------| |---------------|
  setTimeout(() => {  |                   | function   <-----timeout, 1000 |
    console.log('hi') |                   |              | |               |
  }, 1000)            |                   |              | |               |
                      |                   |              | |               |
```

超时时间到了，`Web APIs`将回调放入task queue中。

```
        [code]        |   [call stack]    | [task queue] | |   [Web APIs]  |
  --------------------|-------------------|--------------| |---------------|
  setTimeout(() => {  | function        <---function     | |               |
    console.log('hi') |                   |              | |               |
  }, 1000)            |                   |              | |               |
                      |                   |              | |               |
```

EventLoop检测到Javascript没有任务处理，从task queue中取出任务执行。

```
        [code]        |   [call stack]    | [task queue] | |   [Web APIs]  |
  --------------------|-------------------|--------------| |---------------|
  setTimeout(() => {  | function          |              | |               |
>   console.log('hi') | console.log       |              | |               |
  }, 1000)            |                   |              | |               |
                      |                   |              | |               |
```

执行该回调函数，回调函数中又调用了`console.log`函数。

```
        [code]        |   [call stack]    | [task queue] | |   [Web APIs]  |
  --------------------|-------------------|--------------| |---------------|
  setTimeout(() => {  | function          |              | |               |
    console.log('hi') |                   |              | |               |
  }, 1000)            |                   |              | |               |
                      |                   |              | |               |
> hi
```

`console.log`执行完成，输出"hi"。

```
        [code]        |   [call stack]    | [task queue] | |   [Web APIs]  |
  --------------------|-------------------|--------------| |---------------|
  setTimeout(() => {  |                   |              | |               |
    console.log('hi') |                   |              | |               |
  }, 1000)            |                   |              | |               |
                      |                   |              | |               |
> hi
```

回调函数执行完成，CallStack再次为空。


上面就是`setTimeout`的执行过程，从中开始看出EventLoop在幕后做的工作。

下面我们再来看一段代码：
```
console.log("start");

setTimeout(() => {
    console.log("timeout");
}, 0);

Promise.resolve()
  .then(() => {
      console.log("promise1");
  })
  .then(() => {
      console.log("promise2");
  });

console.log("end");
```

这段程序的输出是什么？建议大家先思考一下，最好能动笔画一画图😎。

## <span id="MicroTaskQueue">4.微任务队列</span>

接着上一节的代码，**符合标准**（很多旧版本的浏览器实现都是不符合标准的，具体参见参考链接）的输出应该是：
```
start
end
promise1
promise2
timeout
```

但是，为？什？么？

实际上，Javascript中有另外一种队列。`Promise`的回调是被放入这个队列的。这个队列叫做**microtask queue**（微任务队列），ES6标准中叫[Job Queue][5]。microTask的优先级是比task高的，也就是说microtask队列中的任务要先处理。

* EventLoop检测到当前没有任务在执行，首先检查microtask队列中有没有需要处理的任务。如果有那么一个个执行，直到microtask队列为空。

* microtask队列中没有任务了，执行task队列中的任务。这里需要注意，**每执行一个task队列中的任务，就检查一下microtask队列状态。将microtask队列中所有任务都执行完成之后，再从task队列中取出任务执行。

下面我们看看上面那段代码是怎么一步步执行的：

（**为方便起见我们称`setTimeout`回调为`timeoutcb`，称第一个`then`成功回调为`promisecb1`，第二个`then`回调为`promisecb2`**）

```
        [code]                    |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  --------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");        |                   |              | |                   | |               |
  2.                              |                   |              | |                   | |               |
  3. setTimeout(() => {           |                   |              | |                   | |               |
  4.     console.log("timeout");  |                   |              | |                   | |               |
  5. }, 0);                       |                   |              | |                   | |               |
  6.                              |                   |              | |                   | |               |
  7. Promise.resolve()            |                   |              | |                   | |               |
  8. .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1"); |                   |              | |                   | |               |
  10.})                           |                   |              | |                   | |               |
  11..then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2"); |                   |              | |                   | |               |
  13.});                          |                   |              | |                   | |               |
  14.                             |                   |              | |                   | |               |
  15.console.log("end");          |                   |              | |                   | |               |
```

开始时，call stack、task queue、microtask queue都为空。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
> 1. console.log("start");         |     <main>        |              | |                   | |               |
  2.                               |     console.log   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
  15. console.log("end");          |                   |              | |                   | |               |
```

程序开始运行，压入`<main>`函数。首先执行第一行代码，`console.log("start")`，将`console.log`压栈。


```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
> 1. console.log("start");         |     <main>        |              | |                   | |               |
  2.                               |                   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
  15. console.log("end");          |                   |              | |                   | |               |
> start
```

`console.log`执行完成，输出"start"。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |    <main>         |              | |                   | |               |
  2.                               |    setTimeout     |              | |                   | |               |
> 3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
  15. console.log("end");          |                   |              | |                   | |               |
> start
```

第二行为空跳过，开始执行第三行代码，`setTimeout`压栈。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |    <main>         |   timeoutcb <------------------------~~timeoutcb, 0~~|
  2.                               |                   |              | |                   | |               |
> 3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
  15. console.log("end");          |                   |              | |                   | |               |
> start
```

`setTimeout`执行完成，由于超时是0，所以立即回调立即进入task queue中。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |    <main>         |  timeoutcb   | |     promisecb1    | |               |
  2.                               |                   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
> 7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
  15. console.log("end");          |                   |              | |                   | |               |
> start
```

代码执行到第7行：

* 首先`Promise.resolve`压栈，执行完成后返回一个`Promise`对象。

* 然后调用该对象的`then`方法，该方法压栈，执行完成后返回一个全新的`Promise`对象，我们称该对象为**promise1**。由于`Promise.resolve`返回对象的状态为`resolved`，所以`promise1`回调直接**进入microtask队列**。

* 接着又执行新对象的`then`方法，我们称该对象为**promise2**。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |    <main>         |    timeout   | |     promisecb1    | |               |
  2.                               |    console.log    |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
> 15. console.log("end");          |                   |              | |                   | |               |
> start
```

代码执行到第15行，`console.log`压栈。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |    <main>         |    timeout   | |     promisecb1    | |               |
  2.                               |                   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
> 15. console.log("end");          |                   |              | |                   | |               |
> start
> end
```

`console.log("end")`执行完成，输出"end"，出栈。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |                   |    timeout   | |     promisecb1    | |               |
  2.                               |                   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
> 15. console.log("end");          |                   |              | |                   | |               |
> start
> end
```

`<main>`函数没有逻辑需要执行了，出栈。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |    promisecb1     |    timeout   | |                   | |               |
  2.                               |                   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
> 15. console.log("end");          |                   |              | |                   | |               |
> start
> end
```

EventLoop检查到microtask队列中有任务需要执行，将`promisecb1`取出压入call stack。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |                   |    timeout   | |      promisecb2   | |               |
  2.                               |                   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
  15. console.log("end");          |                   |              | |                   | |               |
> start
> end
> promise1
```

`promisecb1`执行完成，输出"promise1"，并且返回`undefined`。（其实在这里还有一个`console.log`压栈出栈的过程，我就不画了，下同）

在[深入理解Javascript之Promise][1]中看到，如果返回一个值，那么对象立刻变为`resolved`。**所以第二个`then`的回调需要安排执行，进入microtask队列**。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |     promisecb2    |    timeout   | |                   | |               |
  2.                               |                   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
  15. console.log("end");          |                   |              | |                   | |               |
> start
> end
> promise1
```

EventLoop检测到call stack中没有正在执行的任务，同时microtask队列不为空。从microtask队列取出任务压入call stack。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |                   |    timeout   | |                   | |               |
  2.                               |                   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
  15. console.log("end");          |                   |              | |                   | |               |
> start
> end
> promise1
> promise2
```

`promisecb2`执行完成，输出"promise2"。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |    timeoutcb      |              | |                   | |               |
  2.                               |                   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
  15. console.log("end");          |                   |              | |                   | |               |
> start
> end
> promise1
> promise2
```

接着,EventLoop检测到call stack和microtask队列都为空，从task队列中取出`timeoutcb`压入栈。

```
        [code]                     |   [call stack]    | [task queue] | | [microtask queue] | |   [Web APIs]  |
  ---------------------------------|-------------------|--------------| |-------------------| |---------------|
  1. console.log("start");         |                   |              | |                   | |               |
  2.                               |                   |              | |                   | |               |
  3. setTimeout(() => {            |                   |              | |                   | |               |
  4.     console.log("timeout");   |                   |              | |                   | |               |
  5. }, 0);                        |                   |              | |                   | |               |
  6.                               |                   |              | |                   | |               |
  7. Promise.resolve()             |                   |              | |                   | |               |
  8.  .then(() => {                |                   |              | |                   | |               |
  9.     console.log("promise1");  |                   |              | |                   | |               |
  10. })                           |                   |              | |                   | |               |
  11. .then(() => {                |                   |              | |                   | |               |
  12.    console.log("promise2");  |                   |              | |                   | |               |
  13. });                          |                   |              | |                   | |               |
  14.                              |                   |              | |                   | |               |
  15. console.log("end");          |                   |              | |                   | |               |
> start
> end
> promise1
> promise2
> timeout
```

`timeoutcb`执行完成，输出"timeout"。

## <span id="总结">5.总结</span>

通过这篇文章，我们了解到Javascript时如何通过call stack来处理函数的调用与返回的。`setTimeout`等异步机制其实是宿主提供实现，并在异步操作完成负责将回调放入任务队列，最后由EventLoop在适合的时机取出压入call stack实际执行。

我们还看到了另外一种队列——microtask队列。该队列中存放的一般是优先级较高的任务，例如`Promise`的回调处理函数。

每当call stack中没有正在执行的任务时，EventLoop会优先从microtask队列中取出任务执行，当该队列为空时才会从task队列取。

在使用一门框架或语言时，对于是否需要了解底层运作机制和原理，往往会有比较大的争论。有人说，我不了解内部原理同样可以写出好程序，那为什么还需要花时间去研究呢？
对此，我觉得了解底层原理还是非常有必要的。有下面几个好处：

* 可以让我们看到全貌，了解整个系统是如何运作的。

* 底层原理大多是相通的，例如几乎所有语言的函数调用底层都是利用CallStack来实现的。学会了Javascript的CallStack运作机制，在学习其他语言的相关概念时往往能事半功倍。

* 了解底层可以让我们心中有数，明白什么事情能做，什么事情不能做。例如Javascript是单线程的，我们写代码时一定不能让线程阻塞了😀。

* 了解底层可以让我们更好的优化代码。当程序性能出现瓶颈时，可以更快地定位问题。

## <span id="参考链接">6.参考链接</span>

1. [Philip Roberts在欧洲JSConf上的演讲（必看）][3]
2. [What is the JS Event Loop and Call Stack? — Jess Telford][4]
3. [Tasks, microtasks, queues and schedules][6]
4. [Understanding the JavaScript call stack][7]

关于我：
[个人主页][8] [简书][9] [掘金][10]

[1]: https://darjun.github.io/2018/11/08/javascript-promise-intro
[2]: https://twitter.com/philip_roberts
[3]: https://www.youtube.com/watch?v=8aGhZQkoFbQ
[4]: https://gist.github.com/jesstelford/9a35d20a2aa044df8bf241e00d7bc2d0
[5]: http://www.ecma-international.org/ecma-262/6.0/#sec-jobs-and-job-queues
[6]: https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/
[7]: https://medium.freecodecamp.org/understanding-the-javascript-call-stack-861e41ae61d4
[8]: https://darjun.github.io
[9]: https://www.jianshu.com/u/0aca4aa367c8
[10]: https://juejin.im/user/5be15200e51d4505466ce2f4