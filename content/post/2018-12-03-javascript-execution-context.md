---
layout:     post
title:      "深入理解Javascript之Execution Context"
subtitle:   "\"Javascript in depth\""
date:       2018-12-03 22:10:08
author:     "darjun"
image: "/img/post-bg-2015.jpg"
tags:
    - javascript
URL: "/2018/12/03/javascript-execution-context/"
categories: [ Tech ]
---

## <span id="概述">1.概述</span>
执行上下文（Execution Context）是执行 Javascript 代码的环境。可以毫不夸张地说，执行上下文是 Javascript 中最重要的概念。它是其他很多重要概念的基础。一旦搞清楚了执行上下文是什么，我们就能很轻松地掌握下面这些概念：

* **顶置（Hoisting）**
* **作用域链（Scope Chaining）**
* **闭包（Closure）**
* **`this`以及`arguments`是如何赋值的**
* 等等等等

## <span id="执行上下文">2.执行上下文</span>
在[深入理解 Javascript 之 CallStack&EventLoop][1]一文中，我们已经简单了解了 Javascript 程序是如何执行以及函数调用的过程。我们知道每次调用一个函数时，都会创建一个“调用信息”结构压入调用栈。其实这个调用结构就是执行上下文。因此调用栈（Call Stack）也被称为执行栈（Execution Stack）。

执行上下文有两种类型：

* 一种为全局执行上下文（Global Execution Context），程序开始时创建，有且只有一个。

* 另一种为局部执行上下文（Local Execution Context），调用函数时创建。局部执行上下文又称为函数执行上下文（Function Execution Context）。

看下面的代码：

```js
var name = "darjun";
var email = "leedarjun@gmail.com";

function greeting() {
    console.log(`Hi, I'm ${name}, my email is ${email}!`);
}

greeting();
```

**Javascript 引擎在执行代码时会创建一个全局对象（global object）。在浏览器中全局对象为`window`对象，在 Node 环境中为`global`对象。**

1） 在所有函数外层定义的变量都会保存在全局对象中。

2） 在函数内，未使用`var`，`let`或`const`修饰的变量定义也会将变量存储在全局对象中。

接下来引擎开始解析代码，创建`<main>`函数包裹代码。
然后，`<main>`函数执行。此时，Javascript 引擎首先会创建一个全局执行上下文。

执行上下文的创建分为两个阶段：

1）创建阶段（Creation Phase）

2）执行阶段（Execution Phase）

在全局执行上下文的创建阶段，引擎将进行如下处理：

1）绑定`this`到全局对象。

2）创建一个全局环境对象（Global Environment）。为`<main>`中定义的变量和函数分配内存。`var`定义的变量初始值为`undefined`。

此时，全局执行上下文如下所示：
```
GlobalExecutionContext = {
    Phase: Creation, // 创建阶段
    this: GlobalObject,
    GlobalEnvironment: {
        name: undefined,
        email: undefined,
        greeting: fn,
    }
}
```

**注意：此时代码还未执行。**

接下来，引擎开始**从上到下**，一行一行地执行`<main>`函数。

首先，引擎将全局执行上下文压入调用栈。这时全局执行上下文切换为执行阶段（Phase: Creation -> Execution）。然后，跳过函数定义。因为`greeting`函数在创建阶段就已经被解析完成并且放入全局环境对象中了。然后执行到代码`greeting();`调用`greeting`函数。

引擎首先为函数`greeting`创建一个局部执行上下文。局部执行上下文的创建也将经历创建和执行两个阶段。创建阶段时，引擎执行如下处理：

1）根据调用方式绑定`this`变量。在这个例子中，函数`greeting`是全局函数，没有对象限定。`this`被绑定到全局对象。

2）创建一个局部环境对象（Local Environment）。该对象与全局环境对象作用类似，只不过是为函数中定义的变量和函数分配内存。**该对象中有一个指向外层环境对象的指针`outer`**

这时的局部执行上下文如下所示：
```
Greeting ExecutionContext = {
    Phase: Creation, // 创建阶段
    this: GlobalObject,
    LocalEnvironment: {
        // 没有变量或函数定义
        outer: <GlobalEnvironment>
    },
}
```

引擎将该局部执行上下文压入调用栈开始执行。`greeting`执行完成之后，从调用栈上弹出其局部执行上下文。此时栈顶只有一个全局执行上下文，继续执行`<main>`。

`<main>`执行完成，将全局执行上下文从调用栈中弹出，程序结束。

## <span id="应用执行上下文理解其他概念">4.应用执行上下文理解其他概念</span>

上面我们了解了什么是执行上下文，并且深入到程序执行内部观察到引擎是怎么处理函数调用的。接下来，我们将运用执行上下文来了解 Javascript 的几个核心概念。

### 顶置

顶置其实是由于 Javascript 特殊的执行逻辑而出现的。我们先修改一下前面的示例代码：

```js
console.log(name);
console.log(email);

var name = "darjun";
var email = "leedarjun@gmail.com";

function greeting() {
    console.log(`Hi, I'm ${name}, my email is ${email}!`);
}

greeting();
```

代码前两行的输出是什么？

我们知道一个执行上下文会经历创建和执行两个阶段。在创建阶段时，引擎首先为函数中定义的变量和函数分配内存空间并存入环境对象中。`var`定义的变量初始化为`undefined`，函数直接解析完成。
然后，引擎压入该执行上下文，一行一行执行代码。

那么很清楚了，前两行的输出都是`undefined`。因为在执行上下文的创建阶段，`name`和`email`会被初始化为`undefined`。这就造成变量或函数还未定义就能直接使用的假象，看起来好像`var`变量和函数定义被“提升”或“顶置”到代码的最前面一样。同样的道理，在代码最上面也可以打印函数`greeting`，将打印出具体的函数对象。因为顶层函数在创建阶段就已经存在环境对象中了。快试试🤩。

`var`的这种特性经常会造成意想不到的结果，所以 ES6 引入了另一种变量定义方式`let`。`let`定义的变量在定义之前引用会抛出异常。这是怎么做到的呢？

其实很简单。在执行上下文的创建阶段，`let`定义的变量也会存入环境对象中。不过，它的初始值为`UnInitialized`（未初始化）。在执行时，如果引用一个值为`UnInitialized`的变量，引擎直接抛出一个错误🥴。

### 闭包

是指函数中能访问在函数外层定义的变量，这个函数加上外层的环境就构成了一个闭包。我们还是通过案例来分析：

```js
function makeAdder(num) {
    return function (x) {
        return x + num;
    }
}

var adder2 = makeAdder(2);
console.log(adder2(10)); // 12

var adder5 = makeAdder(5);
console.log(adder5(10)); // 15
```

第一次调用函数`makeAdder`时，传入参数`2`，返回一个匿名函数赋值给变量`adder2`。这时，`makeAdder`函数已返回。但是`adder2`调用时能正确返回`12`。说明`adder2`能访问到之前传入的参数`num`。

第二次调用函数`makeAdder`时，传入参数`5`，返回一个匿名函数赋值给变量`adder5`。此时，`makeAdder`函数已返回。但是`adder5`调用时能正确返回`15`。说明`adder5`能访问到之前传入的参数`num`。**并且，`adder2`与`adder5`访问到的`num`变量相互独立（一个为2，一个为5）**。

运用执行上下文模拟一次程序执行过程，能很清楚的看到闭包的工作原理。

参数`num`相当于是在函数内定义的变量。
首先，第一次调用`makeAdder`时。引擎为此次调用创建一个新的局部环境对象，`num`被保存在此对象中：

```
makeAdder LocalEnvironment2 = {
    num: 2,
}
```

`adder2`被调用时，引擎会创建一个新的局部环境对象。该对象中保存着`x = 10`，并且其`outer`指针指向上面的`LocalEnvironment2`：

```
adder2 LocalEnvironment = {
    x: 10,
    outer: <makeAdder LocalEnvironment2>
}
```

`adder2`执行过程中，访问变量`num`。引擎首先在`adder2`的局部环境对象中查找`num`，没有找到。然后引擎会到其外层的环境对象中继续查找，直到找到该变量。或者直到全局环境对象中也未能找到，抛出引用错误。
在该示例中，外层环境对象中查找到`num`为`2`。`adder2(10)`执行完成，输出`12`。

第二次调用`makeAdder`时。引擎为此次调用创建一个新的局部环境对象，`num`被保存在此对象中：

```
makeAdder LocalEnvironment5 = {
    num: 5,
}
```

`adder5`被调用时，引擎会创建一个新的局部环境对象。该对象中保存`x = 10`，并且其`outer`指针指向上面的`LocalEnvironment5`：

```
adder5 LocalEnvironment = {
    x: 10,
    outer: <makeAdder LocalEnvironment5>
}
```

执行代码`return x + num`时，按照上面的变量查找流程，在外层环境对象`LocalEnvironment5`中找到的`num`值为`5`。`adder5(10)`执行完成，输出`15`。

### `arguments`

我们知道，在函数调用中，`arguments`对象中包含传入的所有参数、参数的长度以及其他一些信息。例如：

```
function f(a, b, c) {
    console.log(arguments);
}

f(1, 2); // Arguments(2) [1, 2, callee: ƒ, Symbol(Symbol.iterator): ƒ]
```

参数列表在调用时会依次被赋予传入的实参。调用时的局部对象会包含所有参数变量，`arguments`等：

```
LocalEnvironment = {
    a: 1,
    b: 2,
    arguments: [1, 2] // ...
}
```

### `this`绑定

先看一段代码：

```js
var person = {
    name: "darjun",
    age: 29,
    greeting: function () {
        console.log(`Hi, I'm ${this.name}, ${this.age} years old`);
    }
}

person.greeting(); // 输出 Hi, I'm darjun, 29 years old

var g = person.greeting;

g(); // 输出 Hi, I'm undefined, undefined years old
```

前面我们知道 Javascript 引擎在执行一个函数前会进行`this`绑定。具体为`this`绑定什么值，视调用形式而定。

在上面的代码中，第一次调用`greeting`函数时，通过对象`person`限定，引擎会将`person`绑定为`this`。
第二次调用前，将`person.greeting`赋值给变量`g`。然后直接调用函数`g`，引擎看到此次调用没有`.`限定符，故而将`this`绑定为全局对象。
所以输出为"Hi, I'm undefined, undefined years old"（注意：输出视全局对象中是否有`name`和`age`属性而有所不同）。

## <span id="可视化">4.可视化</span>

这里我给大家推荐一个可视化查看程序执行的工具：[javascript-visualizer][2]。

顶置：

![](/img/in-post/execution-context/execution-stack-demo.gif)

闭包：

![](/img/in-post/execution-context/closure-demo.gif)

工具并不完善，但是非常有助于我们理解执行上下文。非常值得一试🤩。


## <span id="总结">5.总结</span>

我认为执行上下文是 Javascript 中最最重要的概念。掌握了执行上下文，我们能很深刻地洞悉 Javascript 程序的运行机理，能很轻松地理解其他的一些重要概念：顶置（Hoisting）、闭包（Closure）、`this`和`arguments`等。

掌握执行上下文，真的能称霸 Javascript 世界哦🤩。

## <span id="参考链接">6.参考链接</span>

1. [Javascript: What Is The Execution Context? What Is The Call Stack][3]
2. [Understand Execution Context and Execution Stack in Javascript][4]
3. [The Ultimate Guide to Hoisting, Scopes, and Closures in JavaScript][5]

关于我：
[个人主页][8] [简书][9] [掘金][10]

[1]: https://darjun.github.io/2018/11/23/javascript-callstack-eventloop/
[2]: https://tylermcginnis.com/javascript-visualizer/?code=console.log%28name%29%3B%0Aconsole.log%28email%29%3B%0A%0Avar%20name%20%3D%20%22darjun%22%3B%0Avar%20email%20%3D%20%22leedarjun%40gmail.com%22%3B%0A%0Afunction%20greeting%28%29%20%7B%0A%20%20console.log%28%22Hello%20World%22%29%3B%0A%7D%0A%0Agreeting%28%29%3B
[3]: https://www.valentinog.com/blog/js-execution-context-call-stack/
[4]: https://blog.bitsrc.io/understanding-execution-context-and-execution-stack-in-javascript-1c9ea8642dd0
[5]: https://tylermcginnis.com/ultimate-guide-to-execution-contexts-hoisting-scopes-and-closures-in-javascript/
[8]: https://darjun.github.io
[9]: https://www.jianshu.com/u/0aca4aa367c8
[10]: https://juejin.im/user/5be15200e51d4505466ce2f4