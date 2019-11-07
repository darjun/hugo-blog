---
layout:     post
title:      "深入理解Javascript之Module"
subtitle:   "\"Javascript in depth\""
date:       2018-12-20T12:10:08
author:     "darjun"
image: "img/post-bg-2015.jpg"
tags:
    - 深入理解JavaScript
URL: "/2018/12/20/javascript-module/"
categories: [ 
    "JavaScript"
]
---

## <span id="模块">什么是模块</span>

模块（module）是什么呢？
模块是为了软件封装，复用。当今开源运动盛行，我们可以很方便地使用别人编写好的模块，而不用自己从头开始编写。在程序设计中，我们一直强调避免重复造轮子（Don't Repeat Yourself，DRY）。

想象一下，没有模块的日子，第三库基本都是导出一个全局变量供开发者使用。例如`jQuery`的`$`，`lodash`的`_`。这些库已经尽量避免了全局变量冲突，只使用几个全局变量。但是还是不能避免有冲突，`jQuery`还提供了`noConflict`。更遑论我们自己编写的代码。

最初，Javascript 中是没有模块的概念的。这可能与一开始 Javascript 的定位有关。Javascript 最初只是希望给网页增加动态元素，定位是简单易用的脚本。
但是，随着网页端功能越来越丰富，程序越来越庞大，软件变得越来越难以维护。特别是随着 NodeJs 的兴起，Javascript 语言进入服务端编程领域。在编写大型复杂的程序，模块更是必须品。

模块只是一个抽象概念，要想在实际编程中使用还需要规范。如果没有规范，我用这种写法，你用那种写法，岂不是乱了套。

目前，模块的规范主要有3中，[CommonJS模块](https://requirejs.org/docs/commonjs.html)、[AMD模块](https://github.com/amdjs/amdjs-api/wiki/AMD)和ES6模块。本文着重讲解 CommonJS 模块（以 Node 实现为代表）和ES6模块。

## <span id="CommonJS">2.CommonJS模块</span>

CommonJS 其实是一个通用的 Javascript 语言规范，并不仅仅是模块的规范。Node 中的模块遵循 CommonJS 规范。

### 基本用法

Node 中提供了一个`require`方法用来加载模块。例如：

```js
var fs = require('fs');

fs.readFile('file1.txt', 'utf8', function (err, data) {
    if (err) {
        console.error(err);
    } else {
        console.log(data);
    }
});
```

导入模块之后就可以使用模块中定义的接口了，如上例中的`readFile`。

### 模块类别

在 Node 中大体上有3种模块，普通模块、核心模块和第三方模块。
普通模块是我们自己编写的模块，核心模块是 Node 提供的模块。上面我们使用的`fs`就是核心模块。普通模块与核心模块的导入方式稍微有些区别。导入普通模块时，需要在`require`的参数中指定相对路径。例如：
```js
var myModule = require('./myModule');
myModule.func1();
```

模块`myModule`的后缀`.js`后缀可以省略。

Node 将核心模块编译进了引擎。导入核心模块只需要指定模块名，Node 引擎直接查找核心模块字典。

第三方模块的导入也是指定模块名，但是模块的查找方式有所不同。

* 首先，在项目目录下的`node_modules`目录中查找。

* 如果没有找到，接着去项目目录的父目录中查找。

* 直到找到加载该模块，或者到根目录还未找到返回失败。

### 定义模块

在我们日常的编程中，经常需要将一些功能封装在一个模块中，方便自己或他人使用。在 Node 中定义模块的语法很简单。模块单独在一个文件中，文件中可以使用`exports`导出接口或变量。例如：
```js
function addTwoNumber(a, b) {
    return a + b;
}

exports.addTwoNumber = addTwoNumber;
```

假设该模块在文件`myMath.js`中。在同一目录下，我们可以这样来使用：
```js
var myMath = require('./myMath');

console.log(myMath.addTwoNumber(10, 20)); // 30
```

### 模块导出详解

函数具体是怎么导出的呢？除了`exports`，我们经常看到的`module.exports`，`__dirname`，`__filename`是从哪里来的？
在执行`require`函数的时候，其实 Node 额外做了一些处理。

* 首先，将模块所在文件内容读出来。然后将这些内容包裹在一个函数中：
```js
function _doRequire(module, exports, __filename, __dirname) {
    // 模块文件内容
}
```

* 接下来，Node 引擎构造一个空的模块对象，给这个对象一个空的`exports`属性，然后推算出`__filename`（当前导入的这个模块的全路径文件名）和`__dirname`（模块文件所在路径）：
```js
var module = {};
module.exports = {}
// __filename = ...
// __dirname = ...
```

* 然后，调用第一步构造的那个函数，传入参数：
```js
_doRequire(module, module.exports, __filename, __dirname);
```

* 最后`require`返回的是`module.exports`的值。

按照上面的过程，我们可以很清楚地理解模块的导出过程。并且也能很快地判断一些写法是否有问题：

**错误写法：**
```js
function addTwoNumber(a, b) {
    return a + b;
}

exports = {
    addTwoNumber: addTwoNumber;
}
```

这种写法为什么不对？`exports`实际上初始时是`module.exports`的一个引用。给`exports`赋一个新值后，`module.exports`并没有改变，还是指向空对象。最后返回的对象是`module.exports`，没有`addTwoNumber`接口。

**正确写法：**
```js
function addTwoNumber(a, b) {
    return a + b;
}

// 正确写法一
exports.addTwoNumber = addTwoNumber;

// 正确写法二
module.exports.addTwoNumber = addTwoNumber;

// 正确写法三
module.exports = {
    addTwoNumber: addTwoNumber
};
```

`exports`和`module.exports`开始指向的是同一个对象。写法一通过`exports`设置属性，同样对`module.exports`也可见。写法二通过`module.exports`设置属性也可以导出。
写法三直接设置`module.exports`就更不用说了。

建议在程序开发中，坚持一种写法。**个人觉得写法三显示设置相对较容易理解。**

**有一点需要注意：不是只有对象可以导出，函数、类等值也可以。**例如下面就导出了一个函数：
```js
function addTwoNumber(a, b) {
    return a + b;
}

module.exports = addTwoNumber;
```

## <span id="ES6">3.ES模块</span>

ES6 在标准层面为 Javascript 引入了一套简单的模块系统。ES6 模块完全可以取代 CommonJS 和 AMD 规范。当前热门的开源框架 React 和 Vue 都已经使用了 ES6 模块来开发。

### 基本使用

ES6 模块使用`export`导出接口，`import from`导入需要使用的接口：

```js
// myMath.js
export var pi = 3.14;

export function addTwoNumber(a, b) {
    return a + b;
}

// 或
var pi = 3.14;

function addTwoNumber(a, b) {
    return a + b;
}

export { pi, addTwoNumber };
```

```js
// main.js
import { addTwoNumber } from './myMath';

console.log(addTwoNumber(10, 20));
```

在`myMath.js`中通过`export`导出一个变量`pi`和一个函数`addTwoNumber`。上例中演示了两种导出方式。一种是一个个导出，对每一个需要导出的接口都应用一次`export`。第二种是在文件中某处集中导出。当然，也可以混合使用这两种方式。**推荐使用第二种导出方式，因为能在一处比较清楚的看出模块导出了哪些接口。**

### ES6 模块特性

ES6 模块有一些需要了解和注意的特性。

#### 静态加载

ES6 模块一个非常重要的特性是“静态加载”，导入的接口是只读的，不能修改。NodeJS 中的模块，是动态加载的。

**静态加载就是“编译”时就已经确定了模块导出，可以做到高效率，并且便于做静态代码分析。同时，静态加载也限制了模块的加载只能在文件中所有语句之前，并且导入语法中不能含有动态的语法结构（例如变量、if语句等）。**

例如：

```js
// 可以调用，因为模块加载是“编译”时进行的。
funcA();
import { funcA, funcB } from './myModule';

// 错误，导入语法中含有变量
var foo = './myModule';
import { funcA, funcB } from './myModule';

// 错误，在if语句中
if (foo == "myModule") {
    import { funcA, funcB } from './myModule';
} else {
    import { funcA, funcB } from './hisModule';
}


// 错误，导出的接口是只读的，不能修改
import { funcA, funcB } from './myModule';
funcA = function () {};
```

**导出的接口与模块中定义的变量或函数必须是一一对应的**。而且模块内相应的值修改了，外部也能感知到。看下面代码：

```js
// 错误，导出值1，模块中没有对应
export 1;

// 错误，实际上也是导出1，模块中没有对应
var m = 1;
export m;

// 可以这样来导出，导出的m与模块中的变量m对应
export var m = 1;

// 可以这样导出
var m = 1;
export {m};
```

```js
var foo = "bar";
setTimeout(2000, () => { foo = "baz"});

// 2s后foo变为"baz"，外部能感知到
```

#### 别名

在导出模块时，可以为接口指定一个别名。这样，后续可以修改内部接口而保持导出接口不变。例如：

```js
// myModule.js
var funcA = function () {
}

var funcB = function () {
}

export {
    funcA as func1,
    funcB as func2,
    funcB as myFunc,
}
```

上面我们导出以别名`func1`导出函数`funcA`，以别名`func2`和`myFunc`导出函数`funcB`。`func2`和`myFunc`都是指向同一个函数`funcB`的。下面看看使用这个模块：

```js
// main.js
import { func1, func2, myFunc } from './myModule';
```

**同样的，导入模块时也可以指定别名：**
```js
// main.js
import { func1 as func } from './myModule';
```

#### default导出

上面介绍的模块导入必须知道接口名字。有时候，用户学习一个模块时希望能够快速上手，不想去看文档（怎么会有这个懒的人🤣）。ES6 提供了default导出。例如：

```js
// myModule.js
export default function () {
    console.log('hi');
}

// default导出方式可以看做是导出了一个别名为default的接口
var f = function () {
    console.log('hi');
}
export { f as default };
```

在外部导入的时候，不能有花括号：

```js
// main.js
import func from './myModule';
func();
```

也可以两种方式，同时使用：
```js
// myModule.js
function foo() {
    console.log('foo');
}

export default foo;

function bar() {
    console.log('bar');
}

export { bar };
```

```js
// main.js
import foo, { bar } from './myModule';
```

#### 整体加载

ES6 还允许一种整体加载的方式导入模块。通过使用`import *`可以导入模块中导出的所有接口：

```js
// myModule.js
export function funcA() {
    console.log('funcA');
}

export function funcB() {
    console.log('funcB');
}
```

```js
// main.js
import * as m from './myModule';

m.funcA();
m.funcB();
```

整体加载所在的那个对象（`m`），应该是可以静态分析的，所以不允许运行时改变。所以，下面的写法都是不允许的：

```js
// main.js
import * as m from './myModule';

// 错误
m.name = 'darjun';
m.func = function () {};
```

#### Node 中使用 ES6 模块

Node 由于已经有 CommonJS 的模块规范了，与 ES6 模块不兼容。为了使用 ES6 模块，Node 要求 ES6 模块采用`.mjs`后缀名，而且文件中只能使用`import`和`export`，不能使用`require`。而且该功能还在试验阶段，Node v8.5.0以上版本，指定`--experimental-modules`参数才能使用：

```js
// myModule.mjs
var counter = 1;

export function incCounter() {
    console.log('counter:', counter);
    counter++;
}
```

```js
// main.mjs
import { incCounter } from './myModule';

incCounter();
```

使用下面命令行运行程序：
```js
$ node --experimental-modules main.mjs
```

## <span id="总结">4.总结</span>

随着 Javascript 在大型项目中占用举足轻重的位置，模块的使用称为必然。Node 中使用 CommonJS 规范。ES6 中定义了简单易用高效的模块规范。ES6 规范化是个必然的趋势，所以在掌握当前 CommonJS 规范的前提下，学习 ES6 模块势在必行。

## <span id="参考链接">5.参考链接</span>

1. [Javascript模块化编程（一）](http://www.ruanyifeng.com/blog/2012/10/javascript_module.html)
2. [Javascript模块化编程（二）](http://www.ruanyifeng.com/blog/2012/10/asynchronous_module_definition.html)
3. [Javascript模块化编程（三）](http://www.ruanyifeng.com/blog/2012/11/require_js.html)
4. [ES6 Module](http://es6.ruanyifeng.com/#docs/module)

关于我：
[个人主页](https://darjun.github.io) [简书](https://www.jianshu.com/u/0aca4aa367c8) [掘金](https://juejin.im/user/5be15200e51d4505466ce2f4)