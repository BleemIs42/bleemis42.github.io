---
title: "JavaScript 工作原理: 引擎, 运行时和调用栈概述"
date: 2020-03-28T23:05:52+08:00
cover: "https://cdn-images-1.medium.com/max/1600/1*OnH_DlbNAPvB9KLxUCyMsA.png"
categories: ["JavaScript"]
tags: ["Nodejs", "V8", "JavaScript"]
---


## 概述

几乎每个人都听说过 V8 引擎这个概念, 大多数人都知道 JavaScript 是单线程的或者它使用的是回调队列.

在本文中, 我们将详细介绍这些概念, 并解释 JavaScript 实际上是如何运行的. 通过理解这些详细信息, 你能够利用这些 API 编写更好地非阻塞的应用程序.


如果你对 JavaScript 还比较陌生, 本文将帮助你理解为什么 JavaScript 与其他语言比如此"怪异".

如果你是一名经验丰富的 JavaScript 开发人员, 希望它能为你每天在实际工作中使用 JavaScript 运行时工作提供新的见解.


## JavaScript 引擎

JavaScript 引擎最流行的是谷歌的 V8 引擎. 例如, Chrome 和 Node.js 使用的就是 V8 引擎. 下面是一个非常简单的视图:

![](https://cdn-images-1.medium.com/max/1600/1*OnH_DlbNAPvB9KLxUCyMsA.png)

引擎由两个主要部分组成:
1. 内存堆: 这是分配内存发生的地方
2. 调用栈: 这是代码执行是堆栈帧所在的位置

## 运行时(Runtime)

浏览器中有几乎所有开发人员都是用过的 API (如`setTimeout`). 然而, 这些 API 不是引擎提供的.

那么, 他们来自哪里呢?

事实证明, 真是的情况要复杂一些.

![](https://cdn-images-1.medium.com/max/1600/1*4lHHyfEhVB0LnQ3HlhSs8g.png)

所以, 我们不但有引擎, 还有其它更多的东西: 那些由浏览器提供的 API, 比如 `DOM`, `AJAX` 以及 `setTimeout` 等等.

然后, 还有我们熟知的事件循环和回调队列.

## 调用栈

JavaScript 是单线程语言, 意味着它只有一个调用栈, 因此, 它一次只能做一件事情.

**调用栈是一种数据结构, 它基本上记录了我们在程序中的位置.** 如果我们进入了一个函数, 那么我们会把它放在堆栈顶部. 如果我们从函数返回, 就会从堆栈顶部弹出. 这就是堆栈所做所有的事情.

让我看看下面的例子:

```js
function multiply(x, y) {
    return x * y;
}
function printSquare(x) {
    var s = multiply(x, x);
    console.log(s);
}
printSquare(5);
```

当引擎开始执行这段代码时, 调用栈是空的, 接下来会发生如下步骤:

![](https://cdn-images-1.medium.com/max/1600/1*Yp1KOt_UJ47HChmS9y7KXw.png)

调用栈中的每一项叫做堆栈帧.

这种构造被用来精确追踪抛出的异常----它基本上就是发生异常时调用栈的状态. 请看下面的代码:

```js
function foo() {
    throw new Error('SessionStack will help you resolve crashes :)');
}
function bar() {
    foo();
}
function start() {
    bar();
}
start();
```

如果这个是在 Chrome 中执行的(假设代码位于 foo.js 文件中), 将生成以下追踪栈:

![](https://cdn-images-1.medium.com/max/1600/1*T-W_ihvl-9rG4dn18kP3Qw.png)

**堆栈溢出**, 达到最大调用堆栈时会发生这种情况. 它很容易发生, 特别是在使用递归而没有全面测试的情况下. 看看下面的示例代码:

```js
function foo() {
    foo();
}
foo();
```

当引擎开始执行这段代码时, 它从调用函数 `foo` 开始. 然而这个函数是递归的, 并且没有任何终止条件. 因此, 在每次执行的过程中相同的函数每次都被添加到调用栈中. 结果如下:

![](https://cdn-images-1.medium.com/max/1600/1*AycFMDy9tlDmNoc5LXd9-g.png)

所以, 当调用栈红的函数数量超出了堆栈的大小时, 浏览器就会抛出错误, 像下面这样:

![](https://cdn-images-1.medium.com/max/1600/1*e0nEd59RPKz9coyY8FX-uw.png)

在单线程上运行代码非常容易, 你不必处理多线程环境出现的复杂场景, 比如死锁.

但是在一个线程上运行的资源是有限的, 因为 JavaScript 只有一个调用栈, 所以当事情变慢的时候会发生什么呢?

## 并发 & 事件循环

如果调用堆栈中的函数调用需要大量的时间才能处理, 会发生什么呢? 例如, 你需要在浏览器中使用 JavaScript 做一些图像转换的操作.

你可能会问----这有什么问题呢? 它的问题在于, 调用栈中有函数待执行, 但是实际上它却无法执行任何操作, 因为它被阻塞了. 这意味着浏览器不能去渲染页面, 也不能运行其它代码, 它卡住了. 它使你的应用的 UI 变的不再流畅.

还不仅仅是这个问题. 一旦浏览器开始处理调用栈中的大量任务, 它可能会在很长一段时间内停止响应. 大多数浏览器都会提示错误, 询问是否要终止网页.

![](https://cdn-images-1.medium.com/max/1600/1*WlMXK3rs_scqKTRV41au7g.jpeg)

这不是最好的用户体验, 是吧?

那么, 我们如何去执行大量的代码而不阻塞 UI 并使浏览器无法响应呢? 解决方案就是异步回调.

请听下回分解.

> 原文: [How JavaScript works: an overview of the engine, the runtime, and the call stack](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf)    
> 作者: [Alexander Zlatkov](https://blog.sessionstack.com/@zlatkov)
